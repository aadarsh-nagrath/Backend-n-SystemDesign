# MySQL/InnoDB Transactions and Locking

> Theory is in [`fundamentals/03-transactions-isolation-concurrency.md`](../fundamentals/03-transactions-isolation-concurrency.md). InnoDB's locking is the most commonly misunderstood part of MySQL. Gap locks and next-key locks cause most "mysterious" deadlocks.

## Table of Contents
1. [Consistent Reads vs Locking Reads](#reads)
2. [Isolation Levels in InnoDB](#isolation)
3. [Lock Types: Record, Gap, Next-Key, Insert Intention, Auto-Inc](#types)
4. [How Many Rows Does a Statement Lock? (it depends on the index)](#howmany)
5. [Worked Locking Scenarios](#scenarios)
6. [Deadlocks: Reading the Report](#deadlocks)
7. [Metadata Locks (MDL)](#mdl)
8. [Lock Monitoring](#monitoring)
9. [Practical Guidance](#guidance)
10. [Interview Questions](#qa)

---

## 1. Consistent Reads vs Locking Reads {#reads}

InnoDB has **two kinds of reads in the same transaction**:

| | Consistent (snapshot) read | Locking (current) read |
|---|---|---|
| Statements | Plain `SELECT` | `SELECT … FOR UPDATE`, `SELECT … FOR SHARE` (`LOCK IN SHARE MODE`), and the read part of `UPDATE`, `DELETE`, `INSERT … SELECT` (source rows at RR get S locks), `INSERT … ON DUPLICATE KEY` |
| Sees | Snapshot (read view) | **Latest committed version** |
| Locks | None | Record/gap/next-key locks |

The "surprise" this creates at REPEATABLE READ:
```sql
-- T1 (RR)
START TRANSACTION;
SELECT COUNT(*) FROM tasks WHERE status = 'new';                  -- 5 (snapshot)
                                     -- T2: INSERT a 'new' task; COMMIT
SELECT COUNT(*) FROM tasks WHERE status = 'new';                  -- still 5 (snapshot)
UPDATE tasks SET status = 'picked' WHERE status = 'new';          -- affects 6 rows! (current read)
SELECT COUNT(*) FROM tasks WHERE status = 'picked';               -- 6 (own changes visible)
COMMIT;
```
So InnoDB's RR is snapshot isolation for reads, plus locking for writes, with a mixed view inside one transaction.

---

## 2. Isolation Levels {#isolation}

```sql
SELECT @@transaction_isolation;                 -- REPEATABLE-READ by default
SET SESSION transaction_isolation = 'READ-COMMITTED';
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;   -- next transaction only
```
| Level | Consistent reads | Locking reads / DML |
|---|---|---|
| READ UNCOMMITTED | Read latest, even uncommitted (dirty reads) | Like RC |
| **READ COMMITTED** | New read view per statement | **Record locks only** (gap locks disabled except for FK and duplicate-key checks). **Semi-consistent read**: UPDATE skips locked rows that don't match the WHERE using the last committed version. Locks on non-matching rows are released after WHERE evaluation |
| **REPEATABLE READ** (default) | One read view per transaction | **Next-key locks** on scanned ranges (prevents phantoms for locking reads). Locks on all scanned rows held to commit |
| SERIALIZABLE | Plain SELECTs become `FOR SHARE` when autocommit is off | Next-key locks; lots of blocking |

**Why many large MySQL shops run READ COMMITTED**: far fewer gap locks means fewer deadlocks and lock waits on insert-heavy workloads, and behavior closer to PG/Oracle. It requires `binlog_format=ROW` (statement-based replication isn't safe under RC). Trade-off: non-repeatable reads within a transaction.

---

## 3. Lock Types {#types}

**Shared (S) and Exclusive (X)** at the row level. **Intention locks (IS, IX)** at the table level signal "I'm going to lock rows inside" so that `LOCK TABLES` / DDL can detect conflicts cheaply. IS/IX don't conflict with each other.

InnoDB locks **index records**, not "rows". Every lock is on an entry in some index (clustered or secondary). If there's no usable index, it locks every record it scans in the clustered index.

| Lock | What it covers | Purpose |
|---|---|---|
| **Record lock** | One index record | Protect the row |
| **Gap lock** | The open interval *between* two index records (or before the first / after the last) | Prevent **inserts** into the gap (phantoms). Gap locks don't conflict with each other: two transactions can hold gap locks on the same gap. They only block inserts |
| **Next-key lock** | Record + gap before it: `(prev, rec]` | Default lock for scans at RR |
| **Insert intention lock** | A special gap lock taken by INSERT before inserting into a gap | Multiple inserters into the same gap at different positions don't block each other, but they **wait for any gap lock** held by others |
| **AUTO-INC lock** | Table-level during auto-increment allocation | Mostly avoided with `innodb_autoinc_lock_mode=2` |
| Predicate locks | Spatial indexes | |

Lock modes appear in `performance_schema.data_locks.LOCK_MODE`:
- `X` = next-key X lock
- `X,REC_NOT_GAP` = record-only lock
- `X,GAP` = gap-only lock
- `X,INSERT_INTENTION`
- `S`, `S,REC_NOT_GAP`, `IX`, `IS`

---

## 4. How Many Rows Get Locked {#howmany}

At REPEATABLE READ, InnoDB locks **every index record it scans** (next-key), plus the gap after the last scanned record for range conditions. The access path decides the scope:

| Query | Index used | Locks (RR) |
|---|---|---|
| `UPDATE t SET … WHERE id = 10` (PK, exists) | PRIMARY | Record lock on id=10 only (unique equality → no gap) |
| `… WHERE id = 10` (PK, **doesn't exist**) | PRIMARY | **Gap lock** on the gap where 10 would be (blocks inserts of nearby ids!) |
| `… WHERE email = 'x'` (unique secondary) | uk_email | Record lock on the secondary entry + the clustered record |
| `… WHERE status = 'new'` (non-unique secondary `idx_status`) | idx_status | Next-key locks on all matching entries + gap after the last one + record locks on matching clustered rows |
| `… WHERE id BETWEEN 10 AND 20` | PRIMARY | Next-key locks on 10..20 and the gap up to the next record (e.g., up to 25) |
| `… WHERE note = 'x'` (**no index**) | full clustered scan | **Next-key lock on every row in the table** (≈ table lock). At RC, locks on non-matching rows are released after the check |

**Most important practical rule: always have an index matching the WHERE of UPDATE/DELETE/locking SELECT.**

---

## 5. Worked Scenarios {#scenarios}

### Scenario A: "SELECT … FOR UPDATE on a non-existent row" deadlock (the classic upsert deadlock)
```sql
-- Table accounts(id PK), existing ids: 10, 20
-- T1                                         -- T2
START TRANSACTION;                            START TRANSACTION;
SELECT * FROM accounts WHERE id = 15 FOR UPDATE;   -- gap lock (10,20)
                                              SELECT * FROM accounts WHERE id = 16 FOR UPDATE;  -- gap lock (10,20) too (gap locks are compatible!)
INSERT INTO accounts (id) VALUES (15);        -- needs insert intention in (10,20) → waits for T2's gap lock
                                              INSERT INTO accounts (id) VALUES (16);  -- waits for T1's gap lock → DEADLOCK
```
"Check if exists, else insert" under RR with FOR UPDATE deadlocks under concurrency. Fixes: `INSERT … ON DUPLICATE KEY UPDATE` / `INSERT IGNORE` directly and catch duplicate-key errors, use READ COMMITTED, or accept a retry on deadlock.

### Scenario B: Insert blocked by a range scan
```sql
-- T1: UPDATE orders SET flagged = 1 WHERE created_at > '2025-06-01';   -- next-key locks on the range incl. supremum
-- T2: INSERT INTO orders (created_at, …) VALUES (NOW(), …);             -- blocks until T1 commits
```
Under RC, T2 wouldn't block (no gap locks).

### Scenario C: Unique-key duplicate check deadlock
Three transactions insert the same unique key: T1 inserts and holds the X lock; T2 and T3 get duplicate-key errors and request **S locks** on the record. T1 rolls back. Now T2 and T3 both hold S and both want X to insert → deadlock. This is a documented InnoDB behavior. Handle it by retrying.

### Scenario D: Lock order inversion on secondary vs primary index
T1 updates via a secondary index (locks secondary entry → then clustered row). T2 updates the same row via PK (locks clustered row → then needs the secondary entry to update the indexed column). Opposite order → deadlock. This is rare but real. Updating by PK consistently avoids it.

### Scenario E: Foreign key S locks
Inserting a child row takes an **S lock on the parent row** (to ensure it isn't deleted concurrently). Concurrent `UPDATE parent` (needs X) waits. Two transactions each inserting children for, then updating, the same two parents in opposite order → deadlock.

---

## 6. Deadlocks {#deadlocks}

- InnoDB detects deadlocks **immediately** via the wait-for graph (`innodb_deadlock_detect=ON`) and rolls back the transaction with the fewest undo records (the "cheapest"). The victim gets `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction`.
- At very high concurrency on hot rows, deadlock detection itself becomes expensive. Some shops disable it and rely on `innodb_lock_wait_timeout` (rarely advisable).
- **Lock wait timeout**: `innodb_lock_wait_timeout` (default **50 s**; lower to 5–10 s for OLTP). `ERROR 1205`. By default **only the statement is rolled back, not the transaction** (`innodb_rollback_on_timeout=OFF`). Your app must roll back explicitly or it may commit a partial transaction!

Reading `SHOW ENGINE INNODB STATUS` → `LATEST DETECTED DEADLOCK`:
```
*** (1) TRANSACTION: TRANSACTION 5001, ACTIVE 3 sec inserting
    ... INSERT INTO accounts (id) VALUES (15)
*** (1) HOLDS THE LOCK(S): RECORD LOCKS ... index PRIMARY ... lock_mode X locks gap before rec
*** (1) WAITING FOR THIS LOCK TO BE GRANTED: ... lock_mode X locks gap before rec insert intention waiting
*** (2) TRANSACTION: ...
*** WE ROLL BACK TRANSACTION (2)
```
Log all deadlocks: `innodb_print_all_deadlocks = ON` (to the error log). Monitor the deadlock rate with `Innodb_deadlocks` (8.0.x status var, or `innodb_metrics` `lock_deadlocks`).

Prevention checklist:
1. Access rows in a consistent order (sort by PK).
2. Keep transactions short. Don't do network calls inside them.
3. Index UPDATE/DELETE predicates.
4. Avoid "SELECT FOR UPDATE then INSERT" for non-existing rows; use upserts.
5. Consider READ COMMITTED.
6. Batch large updates into small chunks.
7. Always retry on 1213 (and on 1205 after rolling back).

---

## 7. Metadata Locks (MDL) {#mdl}

Every statement that touches a table takes a **metadata lock** (shared for DML/SELECT, exclusive for DDL) held until the **transaction ends**.

The outage pattern (same as PG's lock queue):
```
T1: START TRANSACTION; SELECT * FROM orders LIMIT 1;   -- holds shared MDL until commit… then the app forgets (idle in trx)
T2: ALTER TABLE orders ADD COLUMN x INT;                -- needs exclusive MDL → "Waiting for table metadata lock"
T3..Tn: SELECT * FROM orders …                          -- queued behind T2's pending exclusive request → all block
```
Diagnose:
```sql
SELECT * FROM performance_schema.metadata_locks WHERE object_name = 'orders';
SELECT * FROM sys.schema_table_lock_waits\G       -- shows the blocking pid & a ready KILL statement
SHOW PROCESSLIST;                                  -- "Waiting for table metadata lock"
```
Defense: `SET SESSION lock_wait_timeout = 5;` before DDL (default **1 year**!), kill long idle transactions first, and use gh-ost/pt-osc (they also need a brief MDL at cut-over; gh-ost retries).

Even "instant" DDL needs the exclusive MDL briefly.

---

## 8. Lock Monitoring {#monitoring}

```sql
-- Current locks (8.0+)
SELECT ENGINE_TRANSACTION_ID AS trx, OBJECT_NAME AS tbl, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks;

-- Who waits for whom
SELECT * FROM sys.innodb_lock_waits\G
-- columns: waiting_query, blocking_query, blocking_pid, sql_kill_blocking_query, wait_age

-- Active transactions with age
SELECT trx_id, trx_state, trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_s,
       trx_rows_locked, trx_rows_modified, trx_mysql_thread_id, LEFT(trx_query, 100)
FROM information_schema.innodb_trx ORDER BY trx_started;

-- Status counters
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock%';   -- waits, time, avg, max
```

---

## 9. Practical Guidance {#guidance}

- Use **atomic conditional updates** (`UPDATE stock SET qty = qty - 1 WHERE id = ? AND qty > 0`) and check affected rows. (Note: MySQL's "affected rows" counts *changed* rows by default; JDBC `useAffectedRows` / client flag `CLIENT_FOUND_ROWS` toggles between found and changed.)
- **Optimistic locking** with a version column for user-facing edits.
- `SELECT … FOR UPDATE SKIP LOCKED` (8.0+) for job queues and seat/ticket allocation.
- `NOWAIT` to fail fast.
- At RR, avoid range locking reads on hot tables. Prefer PK-based locking.
- `autocommit=1` by default. Frameworks often send `SET autocommit=0` and manage transactions themselves. Watch for forgotten open transactions in pooled connections (a sleeping connection holding locks shows `trx_state = RUNNING` in innodb_trx with no query).
- Hot rows (counters): spread over N rows, or batch increments in Redis and flush.

---

## 10. Interview Questions {#qa}

1. What's the difference between a consistent read and a locking read in InnoDB?
2. Explain record, gap, and next-key locks. Why do gap locks exist?
3. Why can two transactions both acquire a gap lock on the same gap, and how does that lead to deadlocks?
4. Why does an UPDATE without an index on its WHERE clause effectively lock the whole table?
5. Why do some companies run MySQL at READ COMMITTED instead of the default?
6. What's a metadata lock, and how can an idle transaction cause an outage during a migration?
7. After `ERROR 1205 Lock wait timeout`, what state is your transaction in?
8. How do you find which session is blocking others in MySQL 8?
