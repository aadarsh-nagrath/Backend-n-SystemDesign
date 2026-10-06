# PostgreSQL — Deep Dive

Read the files in order. Each one assumes the cross-engine concepts in [`../fundamentals/`](../fundamentals/).

| # | File | What you'll be able to do after |
|---|---|---|
| 01 | [Architecture](01-architecture.md) | Explain processes, memory, on-disk layout, and the life of a query or write |
| 02 | [MVCC & VACUUM](02-mvcc-and-vacuum.md) | Diagnose bloat, tune autovacuum, prevent XID wraparound |
| 03 | [Indexes](03-indexes.md) | Pick B-tree/GIN/GiST/BRIN/HNSW correctly; build indexes safely |
| 04 | [EXPLAIN & Query Tuning](04-explain-and-query-tuning.md) | Read plans, find the bottleneck node, fix it |
| 05 | [Types, JSONB, FTS & Features](05-data-types-jsonb-fts-and-features.md) | Use JSONB, ranges, FTS, matviews, LISTEN/NOTIFY, and PG as a queue |
| 06 | [Locking & Concurrency](06-locking-and-concurrency.md) | Avoid lock-queue outages; debug blocking and deadlocks |
| 07 | [Partitioning](07-partitioning.md) | Partition by time, automate retention, migrate big tables |
| 08 | [Replication, HA & Backup](08-replication-ha-backup.md) | Run streaming/logical replication, Patroni, pgBackRest, PITR, upgrades |
| 09 | [Configuration & Tuning](09-configuration-and-performance-tuning.md) | Set memory, WAL, planner, and autovacuum settings with reasons |
| 10 | [Security, Roles & RLS](10-security-roles-rls.md) | Lay out least-privilege roles, set up TLS/SCRAM, and isolate tenants with RLS |
| 11 | [Extensions, Ecosystem & Ops Cheat Sheet](11-extensions-ecosystem-and-ops-cheatsheet.md) | Know the toolbox; use a diagnostic query library and runbooks |

## Hands-on lab (do this, don't just read)

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=pg -p 5432:5432 postgres:17
docker exec -it pg psql -U postgres
```
1. Generate data: `CREATE TABLE t AS SELECT g AS id, md5(g::text) AS v, now() - g * interval '1 min' AS ts FROM generate_series(1, 5000000) g;`
2. Run `EXPLAIN (ANALYZE, BUFFERS)` for a lookup with and without an index; with `random_page_cost` at 4 vs 1.1.
3. `UPDATE t SET v = v WHERE id < 1000000;`, then check `pg_stat_user_tables.n_dead_tup` and the table size, run `VACUUM`, and check again.
4. Open two `psql` sessions and reproduce: lost update in READ COMMITTED, a serialization error in REPEATABLE READ, and a deadlock.
5. Start a long transaction in one session, then run `ALTER TABLE t ADD COLUMN x int;` in another, and a `SELECT` in a third. Watch the lock queue in `pg_stat_activity`.
6. Set up a streaming replica with a second container plus `pg_basebackup -R`.
7. Install `pg_stat_statements`, run `pgbench`, and find the top queries.

## Must-read external resources
- Official docs: https://www.postgresql.org/docs/current/ (especially chapters on MVCC, Indexes, Performance Tips, WAL, High Availability)
- *The Internals of PostgreSQL*, Hironobu Suzuki: https://www.interdb.jp/pg/
- *PostgreSQL 14 Internals*, Egor Rogov (free PDF from Postgres Professional)
- Use The Index, Luke: https://use-the-index-luke.com/
- depesz.com, Cybertec blog, pganalyze blog, Crunchy Data blog
- "Postgres FM" podcast
