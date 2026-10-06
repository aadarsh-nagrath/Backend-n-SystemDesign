# PostgreSQL Locking: Table Locks, Row Locks, Lock Queues, and Deadlocks

> Concurrency theory: [`fundamentals/03-transactions-isolation-concurrency.md`](../fundamentals/03-transactions-isolation-concurrency.md). This note covers exactly which locks Postgres takes and how to avoid lock-related outages.

## Table of Contents
1. [Lock Levels](#levels)
2. [Table-Level Lock Modes and the Conflict Matrix](#table)
3. [Which Statement Takes Which Lock](#statements)
4. [Row-Level Locks](#row)
5. [The Lock Queue Problem (how a migration takes down prod)](#queue)
6. [Isolation Levels in Postgres Specifically](#isolation)
7. [Deadlocks in Postgres](#deadlocks)
8. [Advisory Locks](#advisory)
9. [Diagnosing Lock Problems (queries)](#diagnose)
10. [Interview Questions](#qa)

---

## 1. Lock Levels {#levels}

| Level | Scope | Stored in |
|---|---|---|
| Table-level ("heavyweight") locks | Relations, also objects/transactions | Shared lock table (`max_locks_per_transaction`), visible in `pg_locks` |
| Row-level locks | Tuples | **Tuple header** (xmax + infomask); multiple lockers → MultiXact. Not in lock table (unlimited rows) |
| Page-level | Rare (hash indexes, GIN pending list) | |
| Predicate locks (SIREAD) | SERIALIZABLE only; never block | `pg_locks` mode SIReadLock |
| Advisory | App-defined | Lock table |
| LWLocks / spinlocks | Internal shared-memory latches | `pg_stat_activity.wait_event` |

Waiting on a row lock is actually waiting on the **transaction lock** of the holder (`transactionid` lock type in `pg_locks`).

---

## 2. Table-Level Lock Modes {#table}

From weakest to strongest:

| # | Mode | Typical acquirer |
|---|---|---|
| 1 | ACCESS SHARE | `SELECT` |
| 2 | ROW SHARE | `SELECT … FOR UPDATE/SHARE` |
| 3 | ROW EXCLUSIVE | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| 4 | SHARE UPDATE EXCLUSIVE | `VACUUM` (non-FULL), `ANALYZE`, `CREATE INDEX CONCURRENTLY`, `CREATE STATISTICS`, some `ALTER TABLE` (`VALIDATE CONSTRAINT`, `SET STATISTICS`, set storage params), `ALTER TABLE … DETACH PARTITION CONCURRENTLY` |
| 5 | SHARE | `CREATE INDEX` (non-concurrent) |
| 6 | SHARE ROW EXCLUSIVE | `CREATE TRIGGER`, some `ALTER TABLE` (ADD FOREIGN KEY on both tables) |
| 7 | EXCLUSIVE | `REFRESH MATERIALIZED VIEW CONCURRENTLY` |
| 8 | ACCESS EXCLUSIVE | `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`, `CLUSTER`, `REINDEX` (non-concurrent), `REFRESH MATERIALIZED VIEW`, **most `ALTER TABLE`** (ADD COLUMN, DROP COLUMN, ALTER TYPE, SET NOT NULL, RENAME), `LOCK TABLE` default |

Conflict matrix (✗ = conflicts):

| Requested ↓ / Held → | AS | RS | RE | SUE | S | SRE | E | AE |
|---|---|---|---|---|---|---|---|---|
| ACCESS SHARE | | | | | | | | ✗ |
| ROW SHARE | | | | | | | ✗ | ✗ |
| ROW EXCLUSIVE | | | | | ✗ | ✗ | ✗ | ✗ |
| SHARE UPDATE EXCL | | | | ✗ | ✗ | ✗ | ✗ | ✗ |
| SHARE | | | ✗ | ✗ | | ✗ | ✗ | ✗ |
| SHARE ROW EXCL | | | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| EXCLUSIVE | | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| ACCESS EXCLUSIVE | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

Key takeaways:
- Plain reads conflict **only** with ACCESS EXCLUSIVE.
- Writes conflict with SHARE (non-concurrent CREATE INDEX) and above.
- ACCESS EXCLUSIVE blocks everything, including SELECT.
- Locks are held **until the end of the transaction**. Running DDL inside a long transaction holds the lock for the whole duration.

---

## 3. Statement → Lock Cheat Sheet {#statements}

| Operation | Lock | Blocks reads? | Blocks writes? |
|---|---|---|---|
| SELECT | AS | – | – |
| INSERT/UPDATE/DELETE | RE + row locks | – | Only same rows |
| CREATE INDEX | S | No | **Yes** |
| CREATE INDEX CONCURRENTLY | SUE | No | No |
| ALTER TABLE ADD COLUMN (nullable/constant default) | AE (brief) | Briefly | Briefly |
| ALTER TABLE ADD CONSTRAINT … NOT VALID | AE (brief) | Briefly | Briefly |
| ALTER TABLE VALIDATE CONSTRAINT | SUE | No | No |
| ALTER TABLE ALTER COLUMN TYPE (rewrite) | AE (long) | **Yes** | **Yes** |
| ADD FOREIGN KEY | SRE on both tables | No | **Yes** (unless NOT VALID first) |
| VACUUM | SUE | No | No |
| VACUUM FULL / CLUSTER | AE (long) | **Yes** | **Yes** |
| TRUNCATE | AE | Yes | Yes |
| REFRESH MATVIEW | AE | Yes | – |
| REFRESH MATVIEW CONCURRENTLY | E | No | – |
| ATTACH PARTITION | SUE on parent (PG 12+), AE on the partition being attached | | |
| DETACH PARTITION CONCURRENTLY | SUE (PG 14+) | | |

---

## 4. Row-Level Locks {#row}

| Mode | Acquired by | Conflicts with |
|---|---|---|
| `FOR KEY SHARE` | FK checks on referenced row | FOR UPDATE |
| `FOR SHARE` | Explicit | FOR NO KEY UPDATE, FOR UPDATE |
| `FOR NO KEY UPDATE` | `UPDATE` not modifying key columns | FOR SHARE, NO KEY UPDATE, UPDATE |
| `FOR UPDATE` | `DELETE`, `UPDATE` of key columns, explicit | everything |

Why it matters: inserting a child row (FK check takes `FOR KEY SHARE` on the parent) doesn't conflict with a concurrent `UPDATE parent SET name=…` (`FOR NO KEY UPDATE`). Before PG 9.3 it did, causing lots of FK deadlocks.

Options:
```sql
SELECT * FROM seats WHERE event_id = 1 AND status = 'free' LIMIT 1 FOR UPDATE SKIP LOCKED;
SELECT * FROM accounts WHERE id = 5 FOR UPDATE NOWAIT;   -- ERROR 55P03 if locked
SELECT … FOR UPDATE OF a;                                 -- lock only table a's rows in a join
```

Row locks are written into tuples (a write: dirty page + WAL). Locking many rows with `FOR UPDATE` isn't free.

---

## 5. The Lock Queue Problem {#queue}

The most common way a "harmless" migration causes an outage:

```
t0: Long analytics SELECT on orders (holds ACCESS SHARE) — runs 10 minutes
t1: Migration: ALTER TABLE orders ADD COLUMN note text;   -- needs ACCESS EXCLUSIVE → waits for t0
t2: Every new SELECT/INSERT on orders → needs ACCESS SHARE/ROW EXCLUSIVE → conflicts with the *queued* ACCESS EXCLUSIVE request → waits behind it
→ All traffic to orders stops for up to 10 minutes. Connection pools fill. Site down.
```
Postgres lock queues are FIFO-ish. A waiting strong lock blocks later weaker requests that conflict with it.

**Defense**:
```sql
SET lock_timeout = '3s';          -- give up quickly if we can't get the lock
SET statement_timeout = '15s';
ALTER TABLE orders ADD COLUMN note text;
-- retry with backoff in your migration tool if it times out
```
Also: kill or avoid long transactions before migrations, run DDL off-peak, and split DDL into separate short transactions. Linters like `squawk` and `strong_migrations` (Rails) flag dangerous migrations in CI.

---

## 6. Isolation Levels in Postgres {#isolation}

- **READ UNCOMMITTED** behaves exactly like READ COMMITTED (PG never shows dirty reads).
- **READ COMMITTED** (default): new snapshot per statement. When an UPDATE finds a row updated by a concurrent committed transaction, it **re-evaluates the WHERE on the latest version** and updates it if it still matches. No serialization errors, but read-modify-write races remain.
- **REPEATABLE READ**: snapshot isolation. Prevents phantoms too (stronger than the SQL standard requires). Concurrent update of the same row → `ERROR: could not serialize access due to concurrent update` (40001). Allows write skew.
- **SERIALIZABLE**: SSI. Prevents write skew, and can raise 40001 on commit or even on reads. Requires retries.

Subtle READ COMMITTED behaviors:
```sql
-- Two sessions both run:
UPDATE counters SET n = n + 1 WHERE id = 1;
-- Correct: the second waits, then re-reads n from the latest version. Atomic increments are safe in RC.

-- But:
SELECT n FROM counters WHERE id = 1;  -- 5 in both sessions
UPDATE counters SET n = 6 WHERE id = 1;  -- lost update!
```

---

## 7. Deadlocks {#deadlocks}

- Detection runs after waiting `deadlock_timeout` (1 s). One transaction is aborted with `40P01 deadlock detected`, and the log shows both statements.
- Common PG deadlock sources:
  1. Two transactions updating the same rows in different orders (batch updates without ORDER BY).
  2. `UPDATE … FROM` / multi-row updates where the row order differs between runs.
  3. Upserts on multiple unique keys in different orders.
  4. Lock upgrades (`FOR SHARE` then `UPDATE`).
  5. Foreign keys plus parent updates in older versions.
- Fix: deterministic ordering (`SELECT … ORDER BY id FOR UPDATE` before updating), smaller transactions, and retries.

`deadlock_timeout` also controls when `log_lock_waits` logs a long wait. Keep `log_lock_waits = on`.

---

## 8. Advisory Locks {#advisory}

```sql
-- Session-level (held until unlock or disconnect)
SELECT pg_advisory_lock(42);   SELECT pg_advisory_unlock(42);
SELECT pg_try_advisory_lock(42);    -- non-blocking, returns bool

-- Transaction-level (released at commit/rollback) — preferred
SELECT pg_try_advisory_xact_lock(hashtext('cron:send-digest'));

-- Two-int keyspace
SELECT pg_advisory_xact_lock(tenant_id, 7);
```
Uses: single-runner cron across many app instances, serializing operations per entity without a row to lock, and migration tool locks.

Pitfalls: session-level advisory locks with PgBouncer transaction pooling get "leaked" onto other clients' sessions, so use xact-level ones. `hashtext` collisions are possible (bigint key space is large, but still).

---

## 9. Diagnosing Lock Problems {#diagnose}

```sql
-- Who is blocking whom
SELECT blocked.pid AS blocked_pid, blocked.usename, now() - blocked.query_start AS waiting_for,
       left(blocked.query, 60) AS blocked_query,
       blocking.pid AS blocking_pid, blocking.state AS blocking_state,
       now() - blocking.xact_start AS blocking_xact_age, left(blocking.query, 60) AS blocking_query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid) ON true
JOIN pg_stat_activity blocking ON blocking.pid = b.pid
ORDER BY waiting_for DESC;

-- Lock waits overview
SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
WHERE state <> 'idle' GROUP BY 1, 2 ORDER BY 3 DESC;

-- Locks held on a table
SELECT l.pid, l.mode, l.granted, a.state, left(a.query, 60)
FROM pg_locks l JOIN pg_stat_activity a USING (pid)
WHERE l.relation = 'orders'::regclass;

-- Kill the blocker (after confirming!)
SELECT pg_terminate_backend(<pid>);
```
`wait_event_type = 'Lock'` means a heavyweight lock wait, while `LWLock` means internal contention (e.g., `WALWrite`, `BufferMapping`, `LockManager` with too many partitions or relations per query).

**LockManager contention**: queries touching many relations (partitioned tables with hundreds of partitions, many indexes) exceed the 16 "fast-path" lock slots per backend and hit the shared lock manager, which hurts at high concurrency. PG 18 raised the fast-path capacity (now scales with `max_locks_per_transaction`). Partition pruning at plan time also reduces this.

---

## 10. Interview Questions {#qa}

1. Which lock does `ALTER TABLE ADD COLUMN` take, and why can it take down a busy site even though it's "instant"?
2. What's the difference between `CREATE INDEX` and `CREATE INDEX CONCURRENTLY` in locking terms?
3. Why does `UPDATE … SET n = n + 1` work correctly in READ COMMITTED while read-then-write in the app doesn't?
4. Explain FOR UPDATE vs FOR NO KEY UPDATE vs FOR KEY SHARE.
5. How do you find which session is blocking others?
6. How would you guarantee a cron job runs on only one of 10 app servers?
7. What does SKIP LOCKED do and when is it appropriate?
