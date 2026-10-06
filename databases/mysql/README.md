# MySQL (InnoDB) — Deep Dive

Read in order. Cross-engine concepts are in [`../fundamentals/`](../fundamentals/).

| # | File | What you'll be able to do after |
|---|---|---|
| 01 | [Architecture](01-architecture.md) | Explain the server/engine split, the binlog vs redo log, and the lifecycle of a query or write |
| 02 | [InnoDB Internals](02-innodb-internals.md) | Reason about the clustered index, buffer pool, redo/undo, purge, and doublewrite |
| 03 | [Indexes & EXPLAIN](03-indexes-and-explain.md) | Design composite and covering indexes, and read every EXPLAIN column |
| 04 | [Transactions & Locking](04-transactions-and-locking.md) | Understand gap/next-key locks and MDL, and debug deadlocks |
| 05 | [Replication & HA](05-replication-and-ha.md) | Work with binlog formats, GTIDs, semi-sync, Group Replication, Orchestrator, Vitess |
| 06 | [Performance Tuning](06-performance-tuning.md) | Write a baseline my.cnf, know what to monitor, and follow a troubleshooting playbook |
| 07 | [Backup, Recovery & Ops](07-backup-recovery-operations.md) | Use XtraBackup, run PITR, choose between gh-ost and online DDL, set up users/TLS, plan upgrades |
| 08 | [Data Types, Charsets & Gotchas](08-data-types-charsets-and-gotchas.md) | Avoid utf8/collation/strict-mode/2038 traps |
| 09 | [MySQL vs PostgreSQL](09-mysql-vs-postgresql.md) | Make and defend a choice, and migrate between them |

## Hands-on lab

```bash
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=pw -p 3306:3306 mysql:8.4
docker exec -it mysql mysql -uroot -ppw
```
1. Create `orders(id BIGINT AUTO_INCREMENT PK, customer_id, status, created_at, total)` and load 5M rows (a recursive CTE or `sysbench` prepare).
2. Compare `EXPLAIN ANALYZE` for `WHERE customer_id = ? ORDER BY created_at DESC LIMIT 10` with indexes `(customer_id)` vs `(customer_id, created_at)`.
3. Create the same table with a `CHAR(36)` UUIDv4 PK and compare insert throughput and `.ibd` size once it exceeds the buffer pool (set `innodb_buffer_pool_size=256M` to make the effect visible).
4. In two sessions at REPEATABLE READ, reproduce the gap-lock deadlock (`SELECT … FOR UPDATE` on a missing id, then INSERT). Then switch to READ COMMITTED and retry.
5. Open a transaction, `SELECT` from a table, leave it idle, then run `ALTER TABLE … ADD COLUMN` in another session and a `SELECT` in a third. Inspect `sys.schema_table_lock_waits`.
6. Set up a GTID replica with the CLONE plugin; break it with a write on the replica and observe the errant GTID.
7. Run `pt-query-digest` on a slow log captured under `sysbench` load.

## Must-read external resources
- MySQL Reference Manual (8.4): InnoDB chapter, Optimization chapter, Replication chapter
- *High Performance MySQL*, 4th ed. (Botros & Tinley)
- *Efficient MySQL Performance* (Daniel Nichter)
- Percona blog, PlanetScale blog, Vitess docs, the gh-ost docs
- Jeremy Cole's InnoDB internals blog series and `innodb_ruby`
