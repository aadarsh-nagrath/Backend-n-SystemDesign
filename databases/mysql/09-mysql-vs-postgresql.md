# MySQL vs PostgreSQL: A Detailed, Honest Comparison

> Both are excellent, battle-tested databases. The differences that matter are architectural (clustered index vs heap, undo vs in-heap MVCC, threads vs processes) and in SQL features and ecosystem. "Which is faster?" depends entirely on the workload.

## Table of Contents
1. [Architecture Differences](#arch)
2. [MVCC and Its Consequences](#mvcc)
3. [Indexing and Storage](#indexing)
4. [SQL Feature Comparison](#sql)
5. [Transactions and Isolation](#tx)
6. [Replication, HA, and Scaling Out](#replication)
7. [Operations](#ops)
8. [Ecosystem and Community](#ecosystem)
9. [Performance Characteristics by Workload](#performance)
10. [Decision Guide](#decision)
11. [Migrating Between Them](#migrate)

---

## 1. Architecture {#arch}

| Aspect | PostgreSQL | MySQL (InnoDB) |
|---|---|---|
| Connection model | Process per connection | Thread per connection |
| Connection scalability | Needs pooler beyond a few hundred | Handles thousands of idle connections better; active concurrency still bounded |
| Engine | One integrated engine (pluggable table AMs exist but rarely used) | Pluggable engines; InnoDB in practice |
| Table organization | **Heap** + secondary indexes pointing to TIDs | **Clustered** by PK; secondary indexes store PK |
| Logs | WAL (recovery + replication + PITR) | Redo (recovery) + binlog (replication/PITR) + undo, with internal 2PC |
| Default page size | 8 KB | 16 KB |
| Extensibility | Very high (types, operators, index AMs, languages, hooks, extensions) | Plugins/components; less extensible at the type/index level |
| DDL | **Transactional** (roll back schema changes) | Atomic per statement but **implicitly commits** |

---

## 2. MVCC Consequences {#mvcc}

| | PostgreSQL | InnoDB |
|---|---|---|
| Update | Writes a new tuple version in the heap | Updates in place, old version in undo |
| Garbage | Dead tuples in tables/indexes → **VACUUM** | Undo → **purge** |
| Write amplification on update | Higher (whole new row + index entries unless HOT) | Lower for in-place updates; secondary index changes still delete-mark + insert |
| Bloat | Table & index bloat is a real operational concern | Less table bloat; fragmentation after deletes; undo growth with long transactions |
| Long transaction impact | Bloat everywhere (vacuum blocked) | History list grows; reads of old snapshots slow |
| XID wraparound | Must be prevented by freezing | No equivalent (64-bit-ish trx ids, 48-bit) |
| Rollback cost | Cheap (just mark aborted) | Expensive for big transactions (must apply undo) |
| Reading old versions | Direct | Reconstruct via undo chain |

Uber's famous 2016 post ("Why Uber Engineering Switched from Postgres to MySQL") centered on these differences: write amplification of index updates on a heavily indexed, update-heavy table, replication of physical WAL (bandwidth and cross-version issues), replica MVCC conflicts, and process-per-connection. Postgres has since improved in several areas (HOT improvements, bottom-up index deletion, logical replication, connection scalability), and the post is still worth reading critically to understand the trade-offs.

---

## 3. Indexing and Storage {#indexing}

| Feature | PostgreSQL | MySQL |
|---|---|---|
| B-tree | ✅ | ✅ |
| Hash | ✅ (persistent) | Adaptive (auto, in-memory) / MEMORY engine |
| GIN (inverted: JSONB, arrays, FTS, trigrams) | ✅ | FULLTEXT only; multi-valued for JSON arrays |
| GiST / SP-GiST (ranges, geo, KNN, exclusion) | ✅ | SPATIAL R-tree |
| BRIN | ✅ | ❌ |
| Partial indexes | ✅ | ❌ |
| Expression indexes | ✅ | ✅ (8.0.13+) |
| Covering (INCLUDE) | ✅ | Implicit PK inclusion; add columns to key |
| Invisible indexes | ❌ (hypopg for "virtual" ones) | ✅ |
| Descending indexes | ✅ | ✅ (8.0) |
| Concurrent index build | `CONCURRENTLY` | Online INPLACE by default |
| Index-only scan caveat | Needs visibility map | Always (with MVCC page check) |
| Vector search | pgvector (mature) | VECTOR type (9.x), HeatWave |
| PK lookups | Index + heap fetch | Single B-tree traversal |
| Range scans by PK | Random heap I/O unless correlated | Sequential (clustered) |

---

## 4. SQL Features {#sql}

| Feature | PostgreSQL | MySQL 8.x |
|---|---|---|
| Window functions | ✅ (incl. GROUPS frames, FILTER) | ✅ |
| CTEs / recursive | ✅ (+ data-modifying CTEs) | ✅ |
| LATERAL | ✅ | ✅ (8.0.14) |
| FULL OUTER JOIN | ✅ | ❌ |
| INTERSECT / EXCEPT | ✅ | ✅ (8.0.31) |
| RETURNING | ✅ | ❌ (MariaDB ✅) |
| Upsert | `ON CONFLICT` (targeted) | `ON DUPLICATE KEY` (any unique key) |
| MERGE | ✅ (15+) | ❌ |
| CHECK constraints | ✅ | ✅ (8.0.16) |
| Exclusion constraints | ✅ | ❌ |
| Deferrable constraints | ✅ | ❌ |
| Arrays | ✅ | ❌ |
| JSON | `jsonb` + GIN + jsonpath + SQL/JSON | `JSON` + generated-column indexes + JSON_TABLE |
| Range types | ✅ | ❌ |
| Custom types / domains | ✅ | ❌ |
| Full-text search | tsvector/tsquery (configurable dictionaries) | FULLTEXT (simpler) |
| Sequences | ✅ (+ identity) | AUTO_INCREMENT only (MariaDB has sequences) |
| Materialized views | ✅ (full refresh) | ❌ |
| Stored procedures | PL/pgSQL, PL/Python, etc.; procedures with COMMIT | SQL/PSM procedures, events (built-in scheduler) |
| Row-level security | ✅ | ❌ (views/app logic) |
| Table inheritance / partitioning | Declarative partitioning; partition-wise ops | Partitioning (no FKs on partitioned tables) |
| Foreign data wrappers | ✅ | FEDERATED engine (weak) |
| LISTEN/NOTIFY | ✅ | ❌ |
| Generated columns | STORED (+VIRTUAL in 18) | VIRTUAL & STORED |
| Strictness | Strict by default | Strict by default since 5.7 (configurable) |

---

## 5. Transactions and Isolation {#tx}

| | PostgreSQL | MySQL InnoDB |
|---|---|---|
| Default isolation | READ COMMITTED | REPEATABLE READ |
| RR semantics | True snapshot isolation; first-updater-wins errors (40001) | Snapshot for plain reads; **current reads for DML/locking**; lost updates possible |
| Phantom protection | Snapshot (RR), SSI (SERIALIZABLE) | Next-key/gap locks for locking reads |
| SERIALIZABLE | SSI: optimistic, non-blocking, aborts on conflict | 2PL-ish: SELECTs take shared locks; blocking |
| Gap locks | None | Yes at RR (deadlock source) |
| Row locks storage | In tuple header (unlimited) | In lock system memory (per page bitmap) |
| DDL in transactions | ✅ | ❌ |
| Savepoints | ✅ (heavy use of subtransactions has perf cliffs: >64 subxacts per txn overflow cache) | ✅ |

---

## 6. Replication, HA, Scaling {#replication}

| | PostgreSQL | MySQL |
|---|---|---|
| Built-in physical replication | Streaming WAL (byte-identical) | — |
| Built-in logical replication | Publications/subscriptions (no DDL) | **Binlog is logical by nature** (row-based); replicas apply SQL-level changes; cross-version easy |
| Parallel apply on replicas | Physical: single startup process (but fast); logical: parallel streaming (16+) | Multi-threaded applier with WRITESET |
| Semi-sync | Sync replication with quorum (`ANY n`) | Semi-sync (AFTER_SYNC), Group Replication |
| Built-in consensus HA | ❌ (Patroni external) | Group Replication / InnoDB Cluster |
| Multi-primary | Not built-in (BDR/pgEdge, logical bi-dir in 16) | Group Replication multi-primary, Galera |
| CDC | Logical decoding (pgoutput) + slots | Binlog (very mature: Debezium, Maxwell, Canal) |
| Sharding middleware | **Citus** (extension) | **Vitess** (proxy layer), ProxySQL |
| Distributed-SQL compatible | CockroachDB, YugabyteDB (PG wire/dialect) | TiDB (MySQL wire/dialect) |

---

## 7. Operations {#ops}

| Concern | PostgreSQL | MySQL |
|---|---|---|
| Biggest operational chore | VACUUM tuning, bloat, wraparound, connection pooling | Online schema changes on big tables, replication lag & topology management |
| Online schema change | Many changes instant/concurrent; `pg_repack` for rewrites | INSTANT/INPLACE + gh-ost/pt-osc for the rest |
| Major upgrades | pg_upgrade (minutes) or logical replication | In-place (DD upgrade) or replica-first |
| Backups | pgBackRest/WAL-G/Barman | XtraBackup/MySQL Shell/mydumper |
| Observability | pg_stat_* views, pg_stat_statements | performance_schema + sys schema (very detailed) |
| Config knobs | Fewer but subtle (work_mem, autovacuum) | Many InnoDB knobs |
| Managed availability | Everywhere (RDS, Aurora, Cloud SQL, AlloyDB, Azure, Neon, Supabase…) | Everywhere (RDS, Aurora, Cloud SQL, Azure, PlanetScale, HeatWave…) |

---

## 8. Ecosystem and Community {#ecosystem}

- **PostgreSQL**: independent community governance (no single company can change the license). Rapid feature growth, a huge extension ecosystem, and strong momentum. It's been the most used/admired DB in Stack Overflow surveys in recent years. Default choice for many new startups, GIS, analytics-heavy apps, and AI apps (pgvector).
- **MySQL**: owned by Oracle. Community concerns about pace and openness drove forks (MariaDB, Percona). There's a massive installed base (WordPress, Drupal, Magento, Laravel/PHP stacks), web-scale operational know-how (Meta, YouTube/Vitess, Shopify, GitHub, Uber), and strong hosted offerings (PlanetScale, Aurora).

---

## 9. Performance by Workload {#performance}

| Workload | Tends to favor | Why |
|---|---|---|
| Simple PK lookups / range scans by PK at very high QPS | MySQL | Clustered index, thread model, mature low-latency path |
| Update-heavy tables with many secondary indexes | MySQL (or PG with HOT-friendly design) | In-place updates vs new tuple versions |
| Complex queries, many joins, analytics on OLTP | PostgreSQL | Richer planner (merge/hash joins forever, better subquery handling, parallel query), more index types |
| JSON document querying | PostgreSQL | jsonb + GIN |
| Geospatial | PostgreSQL | PostGIS |
| Full-text (basic) | PostgreSQL slightly | Configurable dictionaries & ranking |
| Massive connection counts | MySQL (or PG + PgBouncer) | Threads vs processes |
| Write-heavy append with huge tables | Either (partitioning); MyRocks for compression | |
| Multi-tenant SaaS with sharding needs | PG + Citus or MySQL + Vitess | Both proven |

Benchmarks on the internet are almost always tuned for one side. Benchmark **your** workload.

---

## 10. Decision Guide {#decision}

Pick **PostgreSQL** if:
- You value rich SQL, data integrity features (partial unique indexes, exclusion constraints, transactional DDL, RLS).
- You need JSONB, GIS, full-text, vectors, time-series via extensions.
- The workload mixes OLTP with complex reporting.
- You're starting fresh and the team has no strong preference (the industry default today).

Pick **MySQL** if:
- The team, tooling, and ops experience are MySQL-centric.
- You expect to need **Vitess-style horizontal sharding** or want PlanetScale.
- The workload is high-QPS, simple, PK-driven OLTP.
- You depend on MySQL-specific ecosystem (WordPress/Magento, binlog-based CDC pipelines already built).

Either is a fine answer in a system design interview if you **justify it with the access patterns** and acknowledge the trade-offs.

---

## 11. Migrating Between Them {#migrate}

Tools: **pgloader** (MySQL → PG in one command, with type casting rules), AWS DMS, Debezium CDC (continuous replication for zero-downtime cutover), ora2pg-like scripts, MySQL Shell (for migration into MySQL).

Watch out for:
- Type mapping: `TINYINT(1)`→`boolean`, `DATETIME`→`timestamptz` (decide the TZ semantics!), `ENUM`→text+CHECK or enum type, `UNSIGNED`→larger signed type or CHECK, `JSON`→`jsonb`, zero dates → NULL.
- Case sensitivity: MySQL's default `_ci` collations make `WHERE email = 'A@B.com'` match lowercase, but PG doesn't (use citext or lower() indexes).
- Identifier quoting (`` ` `` vs `"`), unquoted identifiers folded to lowercase in PG.
- `AUTO_INCREMENT` → identity + `setval`.
- Upsert syntax, `LIMIT x, y`, `GROUP_CONCAT` → `string_agg`, `IFNULL` → `COALESCE`, `NOW()` semantics, boolean expressions in SELECT.
- Implicit casts MySQL allowed (string ↔ number) error in PG, so application queries need fixing.
- Transaction behavior differences (default isolation!). Re-test concurrency-sensitive code.
- ORMs hide many differences, but raw SQL won't port as-is.
