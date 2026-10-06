# Transactions, Isolation Levels, and Concurrency Control

> ACID basics live in [`database-concepts/acid.md`](../../database-concepts/acid.md). This note goes deeper: *what actually goes wrong under concurrency*, how each isolation level is implemented (MVCC vs locking), how PostgreSQL and MySQL differ, and the application patterns that keep data correct.

## Table of Contents
1. [What a Transaction Promises](#promise)
2. [The Anomaly Zoo](#anomalies)
3. [Isolation Levels: The Standard vs Reality](#levels)
4. [Implementation 1: Two-Phase Locking (2PL)](#2pl)
5. [Implementation 2: MVCC and Snapshot Isolation](#mvcc)
6. [Implementation 3: Serializable Snapshot Isolation (SSI)](#ssi)
7. [PostgreSQL vs MySQL InnoDB, Level by Level](#pgmysql)
8. [Lock Types and Granularity](#locks)
9. [Deadlocks](#deadlocks)
10. [Application Patterns: Pessimistic, Optimistic, Atomic Updates, Constraints](#patterns)
11. [Retrying Transactions Correctly](#retry)
12. [Transactions and the Outside World (dual writes)](#outside)
13. [Long Transactions: Why They Hurt](#long)
14. [Distributed Transactions: 2PC, Sagas](#distributed)
15. [Interview Questions](#qa)

---

## 1. What a Transaction Promises {#promise}

A transaction groups operations into one logical unit:
- **Atomicity** — all or nothing (implemented by undo/rollback: undo log in InnoDB, MVCC old versions + commit status (CLOG) in PG).
- **Consistency** — invariants (constraints) hold before and after. Mostly *your* job; the DB enforces declared constraints.
- **Isolation** — concurrent transactions don't see each other's half-done work *to the degree promised by the isolation level*. **This is the hard and leaky one.**
- **Durability** — once COMMIT returns, data survives a crash (WAL / redo log flushed with fsync). Tunable: `synchronous_commit=off` (PG) or `innodb_flush_log_at_trx_commit=2` trade durability of the last ~second for speed.

Autocommit: by default each statement is its own transaction in both PG and MySQL. Frameworks often open explicit transactions per request — know what yours does.

---

## 2. The Anomaly Zoo {#anomalies}

Notation: T1, T2 run concurrently.

### Dirty write
T1 writes x, T2 overwrites x before T1 commits; then T1 rolls back — whose value? Prevented by **every** real isolation level (row write locks).

### Dirty read
T2 reads a value T1 wrote but hasn't committed. T1 rolls back → T2 acted on data that never existed.

### Non-repeatable read (fuzzy read)
T1 reads row x = 100. T2 updates x = 50, commits. T1 reads x again → 50. Same query, different answer within one transaction.

### Phantom read
T1: `SELECT COUNT(*) FROM bookings WHERE room=1 AND day='Mon'` → 0. T2 inserts a booking for that room/day, commits. T1 re-runs → 1. A *new row* matching the predicate appears. Different from non-repeatable read: it's about the *set*, not one row.

### Lost update
Both read counter = 10, both write 11. One increment lost. The classic **read-modify-write** bug:
```python
balance = db.query("SELECT balance FROM accounts WHERE id=1")   # both read 100
db.execute("UPDATE accounts SET balance=%s WHERE id=1", balance - 30)  # both write 70; should be 40
```

### Read skew (inconsistent analysis)
Alice has two accounts, 500 each (total 1000). T2 transfers 100 from A to B. T1 reads A *before* the transfer (500) and B *after* (600) → sees 1100. Snapshot isolation prevents this.

### Write skew
Two transactions read the **same** data, make **disjoint** writes based on it, and together violate an invariant.
Classic: on-call doctors. Rule: at least one doctor on call. Alice and Bob are both on call.
```
T1 (Alice): SELECT count(*) FROM doctors WHERE on_call → 2 → OK to leave → UPDATE doctors SET on_call=false WHERE name='Alice'
T2 (Bob):   SELECT count(*) FROM doctors WHERE on_call → 2 → OK to leave → UPDATE doctors SET on_call=false WHERE name='Bob'
```
Both commit → zero doctors on call. No row was written by both, so row locks/snapshot "first-updater-wins" don't catch it. **Snapshot isolation (PG REPEATABLE READ, Oracle "SERIALIZABLE") allows write skew.** Only true serializability prevents it (or explicit locking / constraints).

Other write-skew examples: double-booking a meeting room; username uniqueness checked in app; spending more than balance across two "wallet" rows; claiming the last ticket.

### Phantom-induced write skew
Same as above but the check is "no row matching X exists" — there's no row to lock at all. Needs predicate/range locks (MySQL next-key locks under locking reads, SSI in PG), a UNIQUE/EXCLUDE constraint, or "materializing the conflict" (create a row to lock, e.g., a `room_days` table).

---

## 3. Isolation Levels: Standard vs Reality {#levels}

ANSI SQL-92 defines levels by which of three phenomena they forbid:

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | ✗ | possible | possible |
| REPEATABLE READ | ✗ | ✗ | possible |
| SERIALIZABLE | ✗ | ✗ | ✗ |

**This table is famously inadequate** (Berenson et al., *"A Critique of ANSI SQL Isolation Levels"*, 1995): it ignores lost update, read skew, write skew, and was written assuming lock-based implementations. Real databases behave like this:

| Anomaly | PG Read Committed | PG Repeatable Read (= Snapshot Isolation) | PG Serializable (SSI) | MySQL InnoDB Read Committed | MySQL InnoDB Repeatable Read (default) | MySQL Serializable |
|---|---|---|---|---|---|---|
| Dirty read | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Non-repeatable read | **possible** | ✗ | ✗ | **possible** | ✗ (plain SELECT)* | ✗ |
| Phantom (plain reads) | **possible** | ✗ | ✗ | **possible** | ✗ (plain SELECT)* | ✗ |
| Lost update | **possible** | ✗ (error, retry) | ✗ | **possible** | **possible**** | ✗ (deadlock/lock) |
| Read skew | **possible** | ✗ | ✗ | **possible** | ✗* | ✗ |
| Write skew | **possible** | **possible** | ✗ | **possible** | **possible** | ✗ |

\* InnoDB RR: plain `SELECT`s read a consistent snapshot, but **locking reads and UPDATE/DELETE read the latest committed version**, not the snapshot. So inside one RR transaction you can see a mix: `SELECT` shows old value, `UPDATE … WHERE` affects rows your snapshot can't see, then a subsequent `SELECT` suddenly shows them. This "semi-consistent" behavior surprises people.

\*\* InnoDB RR doesn't detect lost updates: T1 and T2 read 10 in their snapshots; T1 `UPDATE SET v=11`; T2's `UPDATE SET v=11` blocks, then proceeds after T1 commits and overwrites. PG RR would abort T2 with `could not serialize access due to concurrent update`.

Defaults: **PostgreSQL = READ COMMITTED. MySQL InnoDB = REPEATABLE READ.** Oracle = READ COMMITTED. SQL Server = READ COMMITTED (locking, unless RCSI enabled).

Setting it:
```sql
-- PG
BEGIN ISOLATION LEVEL SERIALIZABLE;   -- or SET TRANSACTION ISOLATION LEVEL ... as first statement
-- MySQL
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;  -- next transaction
SET SESSION transaction_isolation = 'READ-COMMITTED';
```

---

## 4. Two-Phase Locking (2PL) {#2pl}

Pure locking approach (MySQL SERIALIZABLE, SQL Server default, old systems):
- Readers take **shared (S)** locks, writers take **exclusive (X)** locks.
- S is compatible with S; X conflicts with everything.
- **Two phases**: a growing phase (acquire locks), then a shrinking phase (release). **Strict 2PL** holds all locks until commit/abort — needed for recoverability.
- Phantoms prevented by **predicate locks** or their practical approximation, **index-range locks** (InnoDB next-key locks).

Downsides: readers block writers and writers block readers → low concurrency, frequent deadlocks, latency spikes. This is why MVCC took over.

---

## 5. MVCC and Snapshot Isolation {#mvcc}

**Multi-Version Concurrency Control**: writes create a *new version* of a row; readers see the version that was committed as of their **snapshot**. **Readers never block writers, writers never block readers.** Writers still block writers on the same row.

How a snapshot is defined (PG terms): when a snapshot is taken it records
- `xmin`: oldest still-running transaction ID,
- `xmax`: next transaction ID to be assigned,
- `xip[]`: list of in-progress transaction IDs.

A row version is **visible** if it was created by a transaction that committed before the snapshot and not deleted by one that committed before the snapshot.

When snapshots are taken:
| Level | Snapshot |
|---|---|
| READ COMMITTED | **New snapshot per statement** → each statement sees all data committed before *it* started |
| REPEATABLE READ / SI | **One snapshot per transaction** (taken at first statement, not at BEGIN; MySQL: `START TRANSACTION WITH CONSISTENT SNAPSHOT` to take it immediately) |

Where old versions live:
| | PostgreSQL | MySQL InnoDB |
|---|---|---|
| New version placement | New tuple written into the heap (append); old tuple stays in place with `xmax` set | Row updated **in place** in the clustered index; previous version copied to the **undo log** |
| Reader of old version | Reads old tuple directly from heap | Reconstructs old version by walking undo chain (roll pointer) |
| Garbage | Dead tuples in the table → **VACUUM** | Old undo records → **purge thread** |
| Long-running txn effect | Prevents VACUUM from removing dead tuples → table/index bloat | Undo log grows (history list length ↑), reads of old snapshots slow down |

Details per engine: [`postgresql/02-mvcc-and-vacuum.md`](../postgresql/02-mvcc-and-vacuum.md), [`mysql/02-innodb-internals.md`](../mysql/02-innodb-internals.md).

### Write–write conflicts under SI
- **First-updater-wins** (PG RR): if T2 tries to update a row modified by T1 after T2's snapshot, T2 waits for T1; if T1 commits, T2 **errors** (SQLSTATE `40001`). Your app must retry.
- READ COMMITTED (PG): T2 waits; when T1 commits, T2 **re-evaluates the WHERE clause on the new version** ("EvalPlanQual") and proceeds. No error, but your read-modify-write in app code is still racy.

---

## 6. Serializable Snapshot Isolation (SSI) {#ssi}

PostgreSQL's SERIALIZABLE (since 9.1, Cahill/Fekete research) = SI + **detection of dangerous dependency structures**:
- Tracks **rw-antidependencies** (T1 read something T2 later wrote) using SIREAD "locks" (not real locks; they never block).
- If it finds a "dangerous structure" (two consecutive rw-edges in a cycle pivot: T1 →rw T2 →rw T3), it aborts one transaction with `40001 could not serialize access due to read/write dependencies among transactions`.
- **Optimistic**: no blocking, false positives possible (aborts that weren't strictly necessary, especially when granularity escalates from tuple → page → relation).

Practical guidance:
- Works great when transactions are short and the app has a **generic retry wrapper**.
- Declare read-only transactions `READ ONLY` (enables optimizations); `SERIALIZABLE READ ONLY DEFERRABLE` for long reports waits for a safe snapshot, then never aborts.
- Indexes matter: without an index, SIREAD locks on a seq scan cover the whole relation → many false aborts.

MySQL SERIALIZABLE is 2PL-flavored: plain SELECTs become `SELECT … FOR SHARE` (when autocommit is off) → blocking and deadlocks instead of aborts.

---

## 7. PostgreSQL vs MySQL, Level by Level {#pgmysql}

**The same code can be correct on one and broken on the other.** Example — "reserve seat if available":
```sql
BEGIN;
SELECT available FROM seats WHERE id = 7;           -- returns true
-- app checks, then:
UPDATE seats SET available = false, holder = 'alice' WHERE id = 7;
COMMIT;
```
- PG RC: two sessions both read true, both update → second update waits, then re-checks WHERE (`id=7` still matches) → both "succeed" → double booking.
- PG RR: second update errors with 40001 → retry → sees false → correct (if you retry).
- MySQL RR: second update waits, then overwrites → double booking.
- **Fix that works everywhere** — make the check part of the write:
  ```sql
  UPDATE seats SET available = false, holder = 'alice' WHERE id = 7 AND available = true;
  -- check affected rows == 1
  ```

Other MySQL InnoDB specifics:
- **Gap locks / next-key locks** at RR: a locking read `SELECT … WHERE x BETWEEN 10 AND 20 FOR UPDATE` locks the index records *and the gaps between them*, blocking inserts into the range → prevents phantoms for locking reads. Side effect: surprising lock waits and deadlocks on inserts. At READ COMMITTED gap locking is (mostly) disabled — many high-throughput MySQL shops run RC with row-based binlog for this reason.
- `UPDATE … WHERE non_indexed_col = …` at RR locks **every row scanned** (effectively the table). Always have an index for UPDATE/DELETE predicates.

---

## 8. Lock Types and Granularity {#locks}

**Row-level**
| Lock | Taken by |
|---|---|
| Exclusive row lock | UPDATE, DELETE, `SELECT … FOR UPDATE` |
| Shared row lock | `SELECT … FOR SHARE` (MySQL also `LOCK IN SHARE MODE`); FK checks on parent (InnoDB S lock; PG `FOR KEY SHARE`) |
| PG extras | `FOR NO KEY UPDATE` (taken by UPDATE not touching keys) and `FOR KEY SHARE` (FK checks) — these don't conflict with each other, which is why PG FK-heavy workloads deadlock less than old PG/MySQL |

Modifiers:
- `NOWAIT` — error immediately if locked.
- `SKIP LOCKED` — skip locked rows (job queues, ticket allocation).

**Table-level**: DDL takes heavy locks. PG `ALTER TABLE` mostly takes `ACCESS EXCLUSIVE` (blocks even SELECT). Lock queue gotcha: an ALTER waiting behind a long SELECT blocks *every subsequent query* behind it → outage. Always `SET lock_timeout = '3s'` for migrations and retry. MySQL has **metadata locks (MDL)** with the same queueing problem.

**Intention locks** (InnoDB IS/IX): table-level markers saying "some rows inside are locked", so a table lock request can detect conflicts without scanning rows.

**Advisory locks** (PG `pg_advisory_lock(key)`, `pg_try_advisory_xact_lock`; MySQL `GET_LOCK('name', timeout)`): application-defined mutexes. Uses: singleton cron jobs, migrations, serializing per-tenant work. Session-level ones survive across transactions — careful with connection poolers (PgBouncer transaction mode breaks them).

**Latches** ≠ locks: short internal mutexes protecting in-memory structures (buffer pages). You see them as LWLock waits in PG or mutex/rw-lock contention in InnoDB.

---

## 9. Deadlocks {#deadlocks}

T1 locks A then wants B; T2 locks B then wants A → cycle. Both engines run a **deadlock detector** (PG after `deadlock_timeout` = 1s; InnoDB immediately via wait-for graph, `innodb_deadlock_detect`) and abort a victim.

Prevention:
1. **Consistent lock ordering**: always touch rows in the same order (e.g., sort IDs: transfer between accounts locks lower id first).
2. Keep transactions short; no network calls inside.
3. Index the predicates of UPDATE/DELETE (fewer rows locked).
4. Avoid lock upgrades (S then X on same row: two readers both upgrading deadlock). Use `FOR UPDATE` upfront.
5. Batch large updates in deterministic order and small chunks.
6. Treat deadlock errors as **retryable** (PG `40P01`, MySQL `1213 ER_LOCK_DEADLOCK`).

Debug: PG logs details with `log_lock_waits = on`; view `pg_locks` joined with `pg_stat_activity` (`pg_blocking_pids(pid)`). MySQL: `SHOW ENGINE INNODB STATUS` (LATEST DETECTED DEADLOCK section), `innodb_print_all_deadlocks = ON`, `performance_schema.data_locks`, `sys.innodb_lock_waits`.

---

## 10. Application Patterns {#patterns}

### a) Atomic in-database update (best when possible)
```sql
UPDATE accounts SET balance = balance - 30 WHERE id = 1 AND balance >= 30;  -- check rowcount
UPDATE products SET stock = stock - 1 WHERE id = 9 AND stock > 0;
INSERT … ON CONFLICT DO UPDATE SET n = t.n + 1;
```
No read-modify-write gap. Works at any isolation level.

### b) Pessimistic locking
```sql
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;  -- lock in deterministic order
-- business logic in app
UPDATE accounts SET balance = balance - 30 WHERE id = 1;
UPDATE accounts SET balance = balance + 30 WHERE id = 2;
COMMIT;
```
Use when contention is high and logic is complex. Cost: blocking.

### c) Optimistic concurrency control (version column)
```sql
SELECT id, title, body, version FROM docs WHERE id = 5;         -- version = 7
-- user edits for 3 minutes (no transaction held!)
UPDATE docs SET title = $1, body = $2, version = version + 1
WHERE id = 5 AND version = 7;                                  -- 0 rows → someone else saved → 409 Conflict
```
Perfect for long "user think time" edits and HTTP (map to `ETag` / `If-Match`). JPA `@Version`, Rails `lock_version`, Django via `filter(version=v).update(...)`.

### d) Let constraints do it
UNIQUE for "username taken", `EXCLUDE` for overlaps, CHECK for non-negative balance. Catch the violation (PG `23505`, MySQL `1062`) and map to a 409.

### e) Materialize the conflict
For write skew with no row to lock (e.g., "max 3 bookings per day"), create a lockable row: `booking_days(room, day)` row; `SELECT … FOR UPDATE` on it first.

### f) SERIALIZABLE + retry
The most general — the DB finds conflicts for you. Requires a retry loop around the whole transaction.

### g) Hot rows
A single counter row updated by every request (likes on a viral post) serializes all writers. Options: shard the counter (`counter_shards(post_id, shard_no, n)` and SUM on read), buffer in Redis and flush, append-only events + periodic aggregation.

---

## 11. Retrying Transactions Correctly {#retry}

Retryable errors:
| | PostgreSQL SQLSTATE | MySQL error |
|---|---|---|
| Serialization failure | `40001` | — |
| Deadlock | `40P01` | `1213` |
| Lock wait timeout | `55P03` (lock_not_available) | `1205` (only the *statement* rolls back by default! `innodb_rollback_on_timeout`) |

```python
def run_in_tx(fn, retries=5):
    for attempt in range(retries):
        try:
            with db.transaction(isolation="serializable"):
                return fn()
        except (SerializationFailure, DeadlockDetected):
            sleep(random.uniform(0, 0.05 * 2 ** attempt))   # jittered backoff
    raise TooMuchContention()
```
Rules: retry the **whole** transaction (re-read everything), and the function must have **no external side effects** (emails, HTTP calls, Kafka publishes) — those would repeat.

---

## 12. Transactions and the Outside World {#outside}

The **dual-write problem**: 
```python
with tx: db.insert(order)
kafka.publish("order_created")   # crash here → DB has order, no event
```
Publishing inside the transaction doesn't help either (event sent, then tx rolls back).

Solutions:
- **Transactional outbox**: insert the event into an `outbox` table in the same transaction; a relay (poller or CDC like Debezium reading WAL/binlog) publishes it, at-least-once. Consumers are idempotent.
- **CDC directly** from the table.
- **Listen to yourself**: write only to Kafka, consume it to update the DB.

See [`messaging/`](../../messaging/) for full coverage.

---

## 13. Long Transactions {#long}

Never hold a transaction open across user think time or slow external calls. Damage:
- Holds row locks → blocks writers → connection pool exhaustion → cascading outage.
- PG: `xmin` horizon held back → VACUUM can't clean → bloat; also blocks DDL. `idle in transaction` sessions are a classic incident. Set `idle_in_transaction_session_timeout` and `statement_timeout`.
- MySQL: undo history grows (`History list length` in `SHOW ENGINE INNODB STATUS`), purge lags, everything slows.
- Replication: long transactions delay apply on replicas; PG `hot_standby_feedback` can push replica queries' horizon onto the primary.

Monitor: PG `SELECT pid, now()-xact_start, state, query FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY 2 DESC;` MySQL `information_schema.innodb_trx ORDER BY trx_started`.

---

## 14. Distributed Transactions {#distributed}

**Two-Phase Commit (2PC)**: coordinator asks participants to PREPARE (persist + vote), then COMMIT/ABORT.
- Blocking protocol: if coordinator dies after prepare, participants hold locks indefinitely ("in-doubt" transactions).
- PG `PREPARE TRANSACTION` / `COMMIT PREPARED` (`max_prepared_transactions`); MySQL `XA START/END/PREPARE/COMMIT`.
- Orphaned prepared transactions in PG hold back VACUUM forever — monitor `pg_prepared_xacts`.

**Sagas**: a sequence of local transactions with compensating actions on failure (orchestrated or choreographed). Gives up isolation (other transactions see intermediate states) → design with semantic locks ("PENDING" states), commutative updates, and idempotency. See [`system-design/notes/concepts.md`](../../system-design/notes/concepts.md) §2.

**Distributed SQL** (Spanner, CockroachDB, YugabyteDB, TiDB) provide serializable/SI distributed transactions using consensus (Raft/Paxos) per range + timestamps (TrueTime / hybrid logical clocks).

---

## 15. Interview Questions {#qa}

1. Explain write skew and give an example. Which isolation level prevents it in PG? In MySQL?
2. Default isolation in PG vs MySQL; what anomaly bites each by default?
3. How does MVCC let readers and writers not block each other? Where are old versions stored in PG vs InnoDB?
4. Two users click "buy" on the last item simultaneously. Show three correct implementations.
5. What is a gap lock and why does it cause deadlocks on INSERT?
6. Optimistic vs pessimistic locking — when each? How does it map to HTTP?
7. Why can't you just publish a Kafka message inside your DB transaction? What's the outbox pattern?
8. Why is a long-running transaction dangerous in Postgres even if it only reads?
9. What errors should be retried, and what must be true of the code you retry?
10. What does `SELECT … FOR UPDATE SKIP LOCKED` do and what is it used for?
