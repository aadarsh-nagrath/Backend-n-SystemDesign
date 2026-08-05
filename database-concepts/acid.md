# ACID properties

**ACID** — **A**tomicity, **C**onsistency, **I**solation, **D**urability — are four properties that guarantee reliable, predictable transaction processing in a database. They matter whenever multiple operations need to succeed or fail together, and whenever multiple transactions run concurrently without stepping on each other. ACID is the backbone of relational databases (MySQL, PostgreSQL, Oracle) and increasingly shows up in NoSQL databases too (MongoDB 4.0+ supports multi-document ACID transactions).

## TL;DR
- **Atomicity** — a transaction is all-or-nothing: every operation commits, or none do.
- **Consistency** — a transaction moves the database from one valid state to another, never violating constraints.
- **Isolation** — concurrent transactions don't see each other's uncommitted changes.
- **Durability** — once committed, changes survive a crash.
- **Isolation levels** (Read Uncommitted → Serializable) trade correctness for performance — memorize the anomaly table below, it's a very common interview question.
- ACID databases are typically **CP** in CAP-theorem terms; **BASE** (Basically Available, Soft state, Eventual consistency) is the relaxed alternative NoSQL databases often choose instead.

## What is a database transaction?

A **transaction** is a logical unit of work made of one or more operations (`INSERT`, `UPDATE`, `DELETE`) executed as a single, indivisible action. Transactions exist to maintain data integrity — either all steps succeed, or none are applied. Classic example: transferring money between accounts.

1. Check the source account has sufficient funds.
2. Deduct the amount from the source account.
3. Add the amount to the destination account.

If any step fails, the whole transaction rolls back — you never want money deducted from one account without appearing in the other.

## The four properties

### 1. Atomicity

A transaction is treated as one indivisible unit. Either **all** operations commit, or **none** are applied — the database is never left in a partially-applied state.

**Mechanics:**
- **Commit** — all operations succeed, changes become permanent.
- **Rollback** — any operation fails, all changes are undone, restoring the prior state.
- **Implementation** — DBMSs use **transaction logs** recording pre/post state for every operation, enabling rollback and crash recovery.

**Example**: transfer $100 from Account X ($500) to Account Y ($200).
```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'X';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'Y';
COMMIT;
-- if an error occurs partway through:
ROLLBACK;
```
Success: X → $400, Y → $300, both committed together. Failure (e.g., a crash between the two updates): the DBMS rolls back, restoring X to $500 and Y to $200 — never a state where money vanished or was duplicated.

⚠️ **Challenge**: atomicity is straightforward on one machine, but ensuring rollback across a distributed system (where nodes can fail independently) needs coordination protocols like two-phase commit (see Advanced, below).

### 2. Consistency

A transaction moves the database from one **valid** state to another, respecting every defined constraint — data types, foreign keys, unique constraints, check constraints.

**Mechanics**: the DBMS enforces constraints (e.g., `balance >= 0`) at commit time; a transaction that would violate one is aborted and rolled back, leaving the database unchanged.

**Example**: same transfer, with a constraint `CHECK (balance >= 0)`.
```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'X';
-- if this makes balance negative, the DBMS raises an error and the whole transaction is rolled back
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'Y';
COMMIT;
```
If X only had $50, deducting $100 would violate the constraint — the transaction aborts before either update takes effect, preserving the invariant that total balance ($700) never becomes invalid.

⚠️ Note this is **not** the same "consistency" as in the CAP theorem — see the comparison table below.

### 3. Isolation

Concurrent transactions execute as if they were running one at a time — partial (uncommitted) changes from one transaction are invisible to others until commit. This is what prevents **dirty reads**, **non-repeatable reads**, and **phantom reads** (defined below).

**Mechanics**: implemented via **locking**, **multiversion concurrency control (MVCC)**, or **optimistic concurrency control**, and tunable via **isolation levels** that trade strictness for throughput.

```sql
-- Transaction 1
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'X';
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'X';
COMMIT;

-- Transaction 2, running concurrently
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE account_id = 'X';  -- sees $500 until T1 commits, then $400
COMMIT;
```

### 4. Durability

Once a transaction commits, its changes are permanently saved to non-volatile storage and survive crashes, power loss, or restarts.

**Mechanics:**
- **Write-Ahead Logging (WAL)** — changes are logged to durable storage *before* being applied to the actual database.
- **Commit process** — the DBMS flushes logs to disk before acknowledging the commit to the caller.
- **Recovery** — after a crash, the DBMS replays the log: redo committed transactions, undo uncommitted ones.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'X';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'Y';
COMMIT; -- DBMS flushes logs to disk before returning success
```
Without WAL, a crash immediately after commit but before the data page is flushed could silently lose the change — WAL is precisely what closes that gap.

⚠️ **Challenge**: durability costs I/O — every commit either waits on a disk flush or risks data loss, and distributed durability (replicating the commit to multiple nodes before acknowledging) adds latency on top of that.

## Isolation levels and the anomalies they prevent

This is the part worth memorizing cold — it comes up constantly in interviews.

### The three anomalies

| Anomaly | Definition | Example |
|---|---|---|
| **Dirty read** | A transaction reads uncommitted changes from another transaction that may later roll back | T1 sets a token to `INVALID` but hasn't committed. T2 reads `INVALID`. T1 rolls back — T2 acted on data that was never really true. |
| **Non-repeatable read** | A transaction reads the same row twice within itself and gets different values, because another transaction committed a change in between | T1 reads a post's title, T2 updates and commits the title, T1 reads it again and sees the new value — inconsistent within one transaction. |
| **Phantom read** | A transaction re-runs the same query and finds *new rows* that weren't there before, because another transaction inserted matching rows | T1 counts all posts in a category, T2 inserts a new post in that category and commits, T1's second identical query now returns one more row. |

### The isolation levels

| Isolation level | Dirty reads | Non-repeatable reads | Phantom reads | Typical mechanism |
|---|---|---|---|---|
| **Read Uncommitted** | ✅ Possible | ✅ Possible | ✅ Possible | No locking on reads — reads whatever is currently in memory, committed or not |
| **Read Committed** | 🚫 Prevented | ✅ Possible | ✅ Possible | Only ever reads committed data, but re-reads within the same transaction can see newer commits |
| **Repeatable Read** | 🚫 Prevented | 🚫 Prevented | ✅ Possible (standard allows it; some engines, e.g. InnoDB, prevent it too) | Snapshot taken at transaction start; same rows re-read return identical values |
| **Serializable** | 🚫 Prevented | 🚫 Prevented | 🚫 Prevented | Full serial-equivalent execution — often via range locks or MVCC + conflict detection |

Reading the table: each level going down prevents strictly more anomalies, at the cost of more locking/coordination and lower concurrency. **Read Committed** is the most common default in production databases (PostgreSQL, SQL Server); **Repeatable Read** is InnoDB's (MySQL) default.

```sql
-- Dirty read example (only possible under Read Uncommitted)
-- Transaction 1
BEGIN TRANSACTION;
UPDATE tokens SET token_status = 'INVALID' WHERE token_id = 1;
-- not yet committed
-- Transaction 2 (under Read Uncommitted)
SELECT token_status FROM tokens WHERE token_id = 1; -- sees 'INVALID' — a dirty read
-- Transaction 1
ROLLBACK; -- token is valid again, but T2 already acted on bad data
```

**Trade-offs**: higher isolation reduces anomalies but increases locking overhead and the chance of contention/deadlocks. Use **Read Committed** for general-purpose applications; reserve **Serializable** for genuinely critical correctness paths (financial ledgers, inventory counts that must never oversell).

## A brief history

- **1970s** — transaction concepts emerge with early DBMSs like IBM's IMS and System R.
- **1983** — Andreas Reuter and Theo Härder formalize the term in *"Principles of Transaction-Oriented Database Recovery,"* coining the ACID acronym.
- **1990s** — relational databases (Oracle, SQL Server) adopt ACID as the standard for reliability.
- **2000s** — the NoSQL movement challenges ACID, prioritizing scalability and availability instead (see CAP theorem).
- **2010s–2020s** — hybrid models emerge; some NoSQL databases (MongoDB 4.0+) add ACID transaction support back in.

## How ACID is implemented

| Mechanism | What it does | Example |
|---|---|---|
| **Locking** (shared/exclusive, row/table/page-level) | Prevents conflicting concurrent access | InnoDB's row-level locking |
| **Write-Ahead Logging (WAL)** | Logs changes durably before applying them | PostgreSQL's WAL for crash recovery |
| **Multiversion concurrency control (MVCC)** | Keeps multiple versions of a row so readers don't block writers | PostgreSQL's snapshot isolation |
| **Optimistic concurrency control** | Assumes conflicts are rare; checks for them only at commit time | Common in NoSQL, e.g. Cassandra |
| **Pessimistic concurrency control** | Locks data up front, for the duration of the transaction | SQL Server's default locking strategy |
| **Transaction logs** | Record every operation, enabling rollback/recovery | Oracle's redo logs |

## ACID across databases

| Database | Concurrency mechanism | Default isolation |
|---|---|---|
| MySQL (InnoDB) | Row-level locking + MVCC | Repeatable Read |
| PostgreSQL | MVCC + WAL | Read Committed |
| Oracle | MVCC + redo logs | Read Committed (supports flashback queries) |
| SQL Server | Locking + MVCC | Read Committed |
| SQLite | WAL-based, lightweight | Serializable (single-writer model) |

```sql
-- PostgreSQL example
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 'X';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'Y';
COMMIT;
```

## ACID in NoSQL and distributed systems

NoSQL databases often relax ACID in favor of **BASE** (Basically Available, Soft state, Eventual consistency) to prioritize scale and availability. Some support ACID to varying degrees:

- **MongoDB (4.0+)** — multi-document ACID transactions in replica sets and sharded clusters, via optimistic concurrency and snapshots.
  ```javascript
  const session = db.getMongo().startSession();
  session.startTransaction();
  db.accounts.updateOne({ _id: "X" }, { $inc: { balance: -100 } }, { session });
  db.accounts.updateOne({ _id: "Y" }, { $inc: { balance: 100 } }, { session });
  session.commitTransaction();
  ```
- **Cassandra** — lightweight transactions (LWT) via Paxos give limited ACID support; generally prioritizes availability over strict consistency (per CAP).
- **DynamoDB** — supports ACID transactions (since 2018) for small-scale operations, using optimistic locking.

### CAP theorem and ACID

The **CAP theorem** says a distributed system can only guarantee two of **Consistency, Availability, Partition tolerance** during a network partition. ACID-style databases typically choose **CP** — consistent and partition-tolerant, sacrificing availability (rejecting writes) during a partition rather than risking incorrect data. BASE-style databases choose **AP** — available and partition-tolerant, accepting writes during a partition and reconciling afterward (eventual consistency).

| | ACID (CP) | BASE (AP) |
|---|---|---|
| Example | PostgreSQL rejects writes during a partition to stay correct | Cassandra accepts writes during a partition, syncs later |

Note again that "consistency" here (distributed data agreement across nodes) is a different concept from ACID's "consistency" (constraint enforcement within a transaction) — see the comparison table below.

## ACID vs. CAP consistency — don't confuse them

| Aspect | CAP consistency | ACID consistency |
|---|---|---|
| Focus | Same (latest) data visible across distributed nodes | Database moves from one valid state to another |
| Scope | Distributed system, across machines | A single transaction, typically one machine |
| Example | Every replica shows the same account balance | A transfer never leaves an account with a negative balance mid-transaction |

## Advantages

| Property | What it guarantees | Example |
|---|---|---|
| Data integrity | Accurate, reliable state | No partial money transfers |
| Consistency | Constraints hold across every transaction | Non-negative balances always enforced |
| Concurrency control | Simultaneous transactions don't conflict incorrectly | No double-spending |
| Crash recovery | Committed data survives failure | Restores committed transactions after a crash |
| Predictability | Deterministic all-or-nothing outcomes | No half-applied operations |

## Disadvantages

- **Performance overhead** — locking, logging, and isolation cost CPU and I/O; Serializable isolation can meaningfully slow high-concurrency systems.
- **Scalability challenges** — coordinating ACID guarantees across distributed nodes (sharded databases especially) is genuinely hard; cross-shard transactions are often disallowed or expensive.
- **Complexity** — implementing MVCC, WAL, and tunable isolation correctly requires real expertise.
- **Latency** — durability requires disk writes; WAL flushes can delay commit acknowledgment.

## Use cases

- **Financial systems** — accurate transfers and balances (banking apps on PostgreSQL/Oracle).
- **E-commerce** — preventing overselling or incorrect order state (Amazon's order system).
- **Healthcare** — consistent patient records across systems.
- **Inventory management** — avoiding phantom reads during stock updates.
- **Reservation systems** — atomic booking (airline ticketing, e.g. Sabre).

## 🔴 Advanced topics

### Concurrency control in depth
- **Pessimistic locking** — lock data during transaction execution; e.g. `SELECT ... FOR UPDATE`.
- **Optimistic locking** — assume no conflict, verify at commit time; e.g. MongoDB's transaction model.
- **MVCC** — maintain data snapshots to let readers and writers proceed without blocking each other; e.g. PostgreSQL's snapshot isolation.

### Distributed transactions
- **Two-Phase Commit (2PC)** — a coordinator asks every participating node to "prepare," then tells them all to commit only if every node agreed. Guarantees atomicity across nodes, but at real cost: high latency, and if the coordinator itself fails mid-protocol, participants can be left blocked holding locks.
- **Saga pattern** — an alternative for microservices: break a distributed transaction into a sequence of local transactions, each with a compensating action to undo it if a later step fails. Trades strict atomicity for eventual consistency and much better availability — the standard approach when 2PC's blocking behavior is unacceptable.

### Performance optimization
- **Batch commits** — group multiple operations to amortize disk I/O.
- **Reduced isolation** — drop to a lower isolation level for non-critical workloads where the anomalies are tolerable.
- **Asynchronous WAL** — offload log writes to improve throughput (at some durability risk).
- **Indexing** — well-chosen indexes reduce the duration rows/pages need to stay locked.

## Best practices

1. Choose isolation level deliberately — Read Committed for most apps, Serializable for financial-grade correctness.
2. Keep transactions short — long-running transactions hold locks longer and hurt concurrency.
3. Index frequently accessed columns to speed up queries and shorten lock duration.
4. Enable deadlock detection and retry logic (e.g., MySQL's `SHOW ENGINE INNODB STATUS` for diagnosis).
5. Configure synchronous disk writes for genuinely critical systems; understand what you're trading away if you don't.
6. Test recovery paths — simulate crashes to verify WAL and recovery actually work as expected.
7. For high-throughput systems, consider deliberately partial ACID compliance (e.g., MongoDB's tunable consistency) rather than defaulting to the strictest option everywhere.

## ACID vs. BASE

| | ACID | BASE |
|---|---|---|
| Focus | Consistency, reliability | Availability, scalability |
| Consistency | Strong, immediate | Eventual |
| Availability | May sacrifice during partitions | High, even during partitions |
| Use case | Banking, e-commerce | Social media, analytics |
| Examples | MySQL, PostgreSQL | Cassandra, DynamoDB |

**When to use which**: ACID for systems where wrong data is worse than no data (financial transactions); BASE for systems where availability matters more than momentary staleness (social feeds).

## Quick reference

- **A**tomicity: all-or-nothing.
- **C**onsistency: valid state → valid state, constraints always hold.
- **I**solation: concurrent transactions don't see each other's uncommitted work.
- **D**urability: committed means committed, even through a crash.
- Isolation level order (weakest → strongest): Read Uncommitted → Read Committed → Repeatable Read → Serializable.
- Each level up the list prevents one more anomaly: dirty read → non-repeatable read → phantom read.
- ACID ≈ CP in CAP terms; BASE ≈ AP.

## Further reading
- Reuter & Härder, *"Principles of Transaction-Oriented Database Recovery"* (1983) — the paper that coined ACID.
- [PostgreSQL documentation: transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [MySQL documentation: InnoDB transaction model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
- [`../scaling-db/cap.md`](../scaling-db/cap.md) — CAP theorem, for the distributed-systems consistency/availability trade-off referenced above
