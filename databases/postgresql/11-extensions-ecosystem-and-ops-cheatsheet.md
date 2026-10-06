# PostgreSQL Extensions, Ecosystem, and Ops Cheat Sheet

## Table of Contents
1. [How Extensions Work](#how)
2. [Must-Know Extensions](#must)
3. [Ecosystem: Poolers, HA, Backup, Monitoring, Tools](#ecosystem)
4. [Postgres-Compatible and Derived Systems](#derived)
5. [psql Power-User Cheat Sheet](#psql)
6. [Diagnostic Query Library](#queries)
7. [Incident Runbook Snippets](#runbook)

---

## 1. How Extensions Work {#how}

- An extension packages SQL objects plus an optional C library: `CREATE EXTENSION name [VERSION x] [SCHEMA s];`, `ALTER EXTENSION name UPDATE;`, `\dx` lists installed ones, `pg_available_extensions` lists available ones.
- Some need `shared_preload_libraries` (loaded at server start, followed by a restart): pg_stat_statements, auto_explain, pg_cron, timescaledb, pgaudit, pg_partman_bgw, citus.
- Managed services allow only a curated list. Check before you design around one.
- Trusted extensions (PG 13+) can be installed by non-superusers who have CREATE on the database.

---

## 2. Must-Know Extensions {#must}

| Extension | What | Use |
|---|---|---|
| **pg_stat_statements** | Query stats | Always on |
| **auto_explain** | Logs plans of slow queries | Always on (sampled) |
| **pgcrypto** | Hashing, encryption, `gen_random_bytes` | `gen_random_uuid()` is now core |
| **uuid-ossp** | UUID generators | Mostly obsolete (core has v4, PG 18 has v7) |
| **citext** | Case-insensitive text | Emails, usernames |
| **pg_trgm** | Trigram similarity/indexes | Fuzzy & infix search |
| **unaccent** | Remove diacritics | Search |
| **btree_gin / btree_gist** | Scalar ops in GIN/GiST | Exclusion constraints with `=` |
| **hstore** | Key/value type | Legacy; prefer jsonb |
| **ltree** | Hierarchical label paths | Category trees |
| **intarray** | Fast int array ops | |
| **PostGIS** | Geospatial types, functions, indexes | The gold standard for GIS |
| **pgvector** | Vector type + HNSW/IVFFlat | Embeddings, RAG |
| **TimescaleDB** | Hypertables, compression, continuous aggregates | Time-series |
| **Citus** | Distributed Postgres (sharding) | Multi-tenant SaaS, real-time analytics |
| **pg_partman** | Partition automation | Time partitions |
| **pg_cron** | Cron scheduler in DB | Matview refresh, cleanup jobs |
| **pgaudit** | Audit logging | Compliance |
| **postgres_fdw / mysql_fdw / oracle_fdw** | Query remote DBs | Migration, federation |
| **pg_repack** | Online table/index rebuild | Bloat removal |
| **pgstattuple** | Exact bloat stats | |
| **pageinspect** | Look at raw pages | Learning, debugging |
| **pg_buffercache** | What's in shared buffers | Cache analysis |
| **pg_prewarm** | Load relations into cache | After restart/failover |
| **hypopg** | Hypothetical indexes | "Would this index be used?" |
| **pg_hint_plan** | Planner hints | Last resort |
| **amcheck** | Verify B-tree/heap integrity | Corruption detection (`pg_amcheck` CLI) |
| **pg_squeeze** | Bloat removal using logical decoding | Alternative to pg_repack |
| **pgmq** | Message queue on Postgres | SQS-like queues |
| **pg_ivm** | Incremental materialized views | |
| **pgsodium / supabase vault** | Libsodium encryption | Secrets |
| **pg_search (ParadeDB)** | BM25 full-text search | Elastic-like search in PG |
| **pg_duckdb / pg_mooncake** | DuckDB analytics inside PG | Columnar analytics |
| **Apache AGE** | Graph queries (openCypher) | Graph on PG |
| **pg_jsonschema** | JSON Schema validation | |
| **plv8 / plpython3u / plrust** | Procedural languages | |

---

## 3. Ecosystem {#ecosystem}

| Category | Tools |
|---|---|
| Pooling | PgBouncer, PgCat, Supavisor, Odyssey, pgpool-II (also LB/HA, complex) |
| HA | Patroni, CloudNativePG, repmgr, pg_auto_failover, Stolon |
| Backup | pgBackRest, WAL-G, Barman, pg_probackup |
| Monitoring | postgres_exporter + Prometheus/Grafana, pganalyze, PMM, Datadog DBM, pgwatch, pgBadger, pg_activity, pgcenter |
| Migrations/schema | Flyway, Liquibase, Sqitch, Atlas, pgroll, Reshape, migra (diff) |
| Linting | squawk (migrations), sqlfluff |
| Admin GUIs | pgAdmin, DBeaver, DataGrip, TablePlus, Postico |
| API layers | PostgREST, Hasura, PostGraphile, Supabase |
| CDC | Debezium, pglogical, wal2json, Sequin, PeerDB (PG → warehouses) |
| Load testing | pgbench, HammerDB, sysbench (pg support) |
| Testing | pgTAP (unit tests in SQL), Testcontainers (real PG in integration tests) |
| Upgrades | pg_upgrade, logical replication, pglogical |
| Learning/visualization | explain.dalibo.com, explain.depesz.com, pgMustard |

---

## 4. Postgres-Compatible and Derived Systems {#derived}

| System | What it is |
|---|---|
| **Amazon Aurora PostgreSQL** | PG engine on distributed log-structured storage (6 copies / 3 AZs); fast replicas & failover |
| **Aurora DSQL** | Serverless distributed SQL, PG-compatible subset, optimistic concurrency |
| **Google AlloyDB** | PG with disaggregated storage + columnar engine for analytics |
| **Azure Cosmos DB for PostgreSQL** | Managed Citus |
| **Neon** | Serverless PG: separated storage (pageserver + safekeepers), branching, scale-to-zero |
| **Supabase** | PG + auth + storage + realtime + PostgREST |
| **CockroachDB** | Distributed SQL with PG wire protocol (not PG internals) |
| **YugabyteDB** | Distributed SQL reusing PG's query layer on DocDB (RocksDB + Raft) |
| **Greenplum / Redshift** (historically derived from PG 8.x) | MPP analytics |
| **TimescaleDB, Citus** | Extensions, not forks |
| **ParadeDB, Tembo stacks** | PG + extension bundles for search/analytics |

"PG-compatible" varies widely. Check transaction semantics, extensions, and isolation levels.

---

## 5. psql Cheat Sheet {#psql}

```
psql "postgresql://user@host:5432/db?sslmode=verify-full"
\?                help on meta-commands       \h ALTER TABLE   SQL syntax help
\l                list databases              \c dbname        connect
\dn               schemas                     \dt[+] [pattern] tables (+ sizes)
\d[+] table       describe table              \di[+]           indexes
\dv \dm \ds \df   views, matviews, sequences, functions
\du               roles                        \dp table        privileges
\dx               extensions                   \x [auto]        expanded display
\timing on        show query time              \watch 2         re-run last query every 2s
\e                edit query in $EDITOR        \i file.sql      run a file
\o out.txt        send output to file          \copy t TO 'f.csv' CSV HEADER   client-side copy
\gexec            execute each result cell as a query (generate DDL!)
\gset             store result columns into psql variables
\set ON_ERROR_STOP on        stop scripts at first error
\set ECHO_HIDDEN on          show SQL behind \d commands
\conninfo         current connection
\pset null '∅'    display NULLs visibly
```
`~/.psqlrc` tips: `\set QUIET 1`, `\pset null '∅'`, `\x auto`, `\set HISTSIZE 10000`, `\timing on`, plus a custom PROMPT1 showing user@host/db and transaction status.

Example `\gexec`: analyze every table in a schema:
```sql
SELECT format('ANALYZE %I.%I', schemaname, tablename) FROM pg_tables WHERE schemaname = 'app' \gexec
```

---

## 6. Diagnostic Query Library {#queries}

```sql
-- Database sizes
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;

-- Biggest tables (with indexes & toast)
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) total,
       pg_size_pretty(pg_relation_size(relid)) heap, pg_size_pretty(pg_indexes_size(relid)) idx
FROM pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;

-- Active sessions right now
SELECT pid, usename, application_name, client_addr, state, wait_event_type, wait_event,
       now() - query_start AS running_for, left(query, 100)
FROM pg_stat_activity WHERE state <> 'idle' AND pid <> pg_backend_pid() ORDER BY running_for DESC;

-- Connection count by state/app
SELECT application_name, state, count(*) FROM pg_stat_activity GROUP BY 1, 2 ORDER BY 3 DESC;

-- Long transactions
SELECT pid, now() - xact_start AS age, state, left(query, 80) FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY age DESC LIMIT 10;

-- Cache hit ratio per table
SELECT relname, heap_blks_read, heap_blks_hit,
       round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) AS hit_pct
FROM pg_statio_user_tables ORDER BY heap_blks_read DESC LIMIT 20;

-- Index usage ratio per table
SELECT relname, seq_scan, idx_scan, round(100.0*idx_scan/nullif(seq_scan+idx_scan,0),1) AS idx_pct, n_live_tup
FROM pg_stat_user_tables ORDER BY seq_scan DESC LIMIT 20;

-- Dead tuples / vacuum status
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum, last_autoanalyze FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- XID wraparound risk
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;

-- Replication & slots
SELECT * FROM pg_stat_replication;
SELECT slot_name, active, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) FROM pg_replication_slots;

-- Blocking tree
SELECT pid, pg_blocking_pids(pid) AS blocked_by, wait_event, left(query, 60)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Table I/O stats (PG 16+)
SELECT backend_type, object, context, reads, writes, extends, hits, evictions FROM pg_stat_io WHERE reads > 0 OR writes > 0;

-- Sequences close to exhaustion (int4 identity hitting 2^31)
SELECT sequencename, last_value, max_value, round(100.0*last_value/max_value, 2) AS pct_used
FROM pg_sequences WHERE last_value IS NOT NULL ORDER BY pct_used DESC LIMIT 10;

-- Settings changed from default
SELECT name, setting, source FROM pg_settings WHERE source NOT IN ('default', 'override') ORDER BY name;
```

---

## 7. Incident Runbook Snippets {#runbook}

**Too many connections**
1. `SELECT application_name, client_addr, state, count(*) FROM pg_stat_activity GROUP BY 1,2,3 ORDER BY 4 DESC;`
2. Kill idle-in-transaction sessions older than N minutes: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle in transaction' AND now() - state_change > interval '5 min';`
3. Fix the source (pool size, leak, missing pooler).

**Disk filling up**
1. Check `pg_wal` size → inactive replication slots? failing `archive_command`?
2. Biggest tables and growth, temp files (`log_temp_files`), bloat.
3. Server logs growing? (log rotation)

**CPU at 100%**
1. `pg_stat_activity` active queries; `pg_stat_statements` deltas.
2. A new query or plan flip? Missing index after a deploy? Stats out of date after a bulk load (`ANALYZE`)?
3. Autovacuum storm (anti-wraparound) or a parallel query explosion.

**Replication lag growing**
1. Replica I/O/CPU saturated? Long-running replica queries with `max_standby_streaming_delay = -1`?
2. Big transaction or bulk load on the primary? Network bandwidth?

**Sudden slowness after failover**
- Cold cache on the new primary (`pg_prewarm`, autoprewarm). Stats are fine (physical replica), but `pg_stat_statements` and cumulative stats reset.
