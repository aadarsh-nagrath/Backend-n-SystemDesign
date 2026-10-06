# Databases for Backend Engineers

> In most backend systems, the database is where correctness and performance are decided. This folder takes you from relational theory to running PostgreSQL and MySQL in production, then through NoSQL and analytics.

## How this folder is organized

```
databases/
├── fundamentals/        engine-agnostic theory & practice (start here)
├── postgresql/          PostgreSQL deep dive (11 notes + lab)
├── mysql/               MySQL / InnoDB deep dive (9 notes + lab)
├── nosql/               MongoDB, Cassandra/ScyllaDB, DynamoDB, Elasticsearch
└── analytics/           OLAP, warehousing, CDC, streaming, ClickHouse
```
Related notes elsewhere in the repo (written earlier, still canonical):
- [`database-concepts/acid.md`](../database-concepts/acid.md): ACID properties
- [`database-concepts/orms.md`](../database-concepts/orms.md): ORMs
- [`scaling-db/db-indexing.md`](../scaling-db/db-indexing.md): indexing basics
- [`scaling-db/sharding.md`](../scaling-db/sharding.md): sharding
- [`scaling-db/cap.md`](../scaling-db/cap.md): CAP theorem
- [`caching/redis/redis.md`](../caching/redis/redis.md): Redis
- [`interview-prep/backend-engineer/03-databases-sql-nosql.md`](../interview-prep/backend-engineer/03-databases-sql-nosql.md) and [`10-sql-query-patterns-and-questions.md`](../interview-prep/backend-engineer/10-sql-query-patterns-and-questions.md): interview drills

---

## 1. Fundamentals (read in order)

| # | Note | Key topics |
|---|---|---|
| 01 | [Relational Model & Normalization](fundamentals/01-relational-model-and-normalization.md) | Keys (natural/surrogate, int vs UUID), constraints, FDs, 1NF→5NF, denormalization, trees, NULL logic, relational algebra |
| 02 | [SQL Deep Dive](fundamentals/02-sql-deep-dive.md) | Joins (and how they execute), subqueries, LATERAL, window functions, CTEs, upserts, keyset pagination, sargability, classic problem patterns |
| 03 | [Transactions, Isolation & Concurrency](fundamentals/03-transactions-isolation-concurrency.md) | Anomaly zoo (write skew!), isolation levels in PG vs MySQL, 2PL vs MVCC vs SSI, locks, deadlocks, optimistic locking, retries, outbox |
| 04 | [Storage Engine Internals](fundamentals/04-storage-engine-internals.md) | Pages, heap vs clustered, B+trees, buffer pool, WAL/ARIES, checkpoints, torn pages, LSM trees, column stores |
| 05 | [Query Planning & Optimization](fundamentals/05-query-planning-and-optimization.md) | Cost-based optimizer, statistics, access methods, index design method, covering/partial indexes, N+1, slow-query workflow |
| 06 | [Schema Design & Migrations](fundamentals/06-schema-design-and-migrations.md) | Data types, soft deletes, audit, enums, money, multi-tenancy, JSON, zero-downtime expand/contract migrations, backfills |
| 07 | [Connections, Pooling & App Integration](fundamentals/07-connections-pooling-and-app-integration.md) | Pool sizing (Little's Law), PgBouncer/ProxySQL, timeouts, SQL injection, ORMs, read replicas & lag, serverless, bulk loads |
| 08 | [Replication, HA, Backup & DR](fundamentals/08-replication-ha-backup-recovery.md) | Physical vs logical replication, topologies, sync vs async, failover & split-brain, backups, PITR, RPO/RTO |
| 09 | [Choosing a Database](fundamentals/09-choosing-a-database.md) | OLTP vs OLAP, every DB family, decision framework, system-design mappings |

## 2. PostgreSQL → [postgresql/README.md](postgresql/README.md)
Architecture · MVCC & VACUUM · Indexes (B-tree/GIN/GiST/BRIN/HNSW) · EXPLAIN · JSONB/FTS/queues · Locking · Partitioning · Replication/Patroni/pgBackRest · Tuning · Security & RLS · Extensions & ops

## 3. MySQL → [mysql/README.md](mysql/README.md)
Architecture · InnoDB internals · Indexes & EXPLAIN · Gap/next-key locking · Replication/GTID/Group Replication/Vitess · Tuning · Backup/gh-ost/ops · Charset & gotchas · MySQL vs PostgreSQL

## 4. NoSQL → [nosql/README.md](nosql/README.md)
Families & consistency models · [MongoDB](nosql/mongodb.md) · [Cassandra/ScyllaDB](nosql/cassandra.md) · [DynamoDB](nosql/dynamodb.md) · [Elasticsearch/OpenSearch](nosql/elasticsearch.md)

## 5. Analytics → [analytics/olap-warehousing-and-data-pipelines.md](analytics/olap-warehousing-and-data-pipelines.md)
Columnar/MPP warehouses · star schema & SCDs · ETL/ELT & dbt · CDC with Debezium · stream processing · Iceberg/lakehouse · ClickHouse · Druid/Pinot

---

## Suggested learning path

**Weeks 1–2: foundations.** Fundamentals 01 → 02 → 03. Do every SQL pattern in 02 by hand on a real Postgres. Reproduce each anomaly from 03 in two psql sessions.

**Weeks 3–4: internals and performance.** Fundamentals 04 → 05, then postgresql 01–04. Load 10M rows and practice EXPLAIN until reading plans feels routine.

**Weeks 5–6: production engineering.** Fundamentals 06 → 07 → 08, then postgresql 06–10. Run a zero-downtime column rename and a PITR restore yourself.

**Week 7: MySQL.** All of mysql/. Compare every behavior against Postgres (isolation defaults, clustered index, gap locks, online DDL).

**Week 8: beyond relational.** nosql/ and analytics/. Model one application (e.g., a chat app) three ways: Postgres, Cassandra, DynamoDB single-table.

## Self-check: can you answer these without notes?

1. Why do random UUIDv4 primary keys hurt InnoDB more than Postgres? What do you use instead?
2. What is write skew, and which isolation level prevents it in Postgres? In MySQL?
3. Explain how a B+tree index makes `WHERE a = ? ORDER BY b LIMIT 10` fast, and which index you'd create.
4. Why does Postgres need VACUUM, and what can stop it from working?
5. What's the outbox pattern, and why can't you publish to Kafka inside a DB transaction?
6. How do you add a NOT NULL column with a foreign key to a 1 TB table with zero downtime (PG and MySQL)?
7. 50 pods × pool size 20 = ? What breaks, and what do you do?
8. Async replica, primary dies. What's lost? How do you prevent split-brain?
9. Design a Cassandra table for "latest 50 messages in a channel".
10. Why can't a DynamoDB Query with `Limit 10` + filter guarantee 10 results?
11. Why is ClickHouse fast for `GROUP BY` over billions of rows but bad at single-row updates?
12. When is Postgres full-text search enough, and when do you need Elasticsearch?

## Books & references
- *Designing Data-Intensive Applications*, Martin Kleppmann (get the 2nd edition if available). Essential.
- *Database Internals*, Alex Petrov
- *SQL Performance Explained* / use-the-index-luke.com, Markus Winand
- *SQL Antipatterns*, Bill Karwin
- *High Performance MySQL*, 4th ed.; *Efficient MySQL Performance*
- *The Art of PostgreSQL*, Dimitri Fontaine; *PostgreSQL 14 Internals*, Egor Rogov (free)
- *The DynamoDB Book*, Alex DeBrie
- CMU 15-445/645 Database Systems (Andy Pavlo) lectures on YouTube: the best free DB internals course
- Jepsen analyses (jepsen.io) for how distributed databases actually behave under faults
