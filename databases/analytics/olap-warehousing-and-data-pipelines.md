# OLAP, Data Warehousing, and Data Pipelines for Backend Engineers

> Backend engineers increasingly own the path from the OLTP database to analytics. This note covers why analytics can't live on the production database, how warehouses work, dimensional modeling, ETL/ELT, CDC, the lakehouse, and ClickHouse in particular.

## Table of Contents
1. [Why Separate Analytics from OLTP](#why)
2. [Warehouse Architecture: Columnar, MPP, Separated Storage/Compute](#arch)
3. [Dimensional Modeling: Facts, Dimensions, Star Schema, SCDs](#modeling)
4. [ETL vs ELT and the Modern Data Stack](#elt)
5. [Change Data Capture (CDC)](#cdc)
6. [Batch vs Streaming Processing](#streaming)
7. [Data Lakes, Table Formats, and the Lakehouse](#lakehouse)
8. [ClickHouse Deep Dive](#clickhouse)
9. [Real-Time Analytics Stores (Druid, Pinot)](#realtime)
10. [Data Quality, Governance, and Privacy](#quality)
11. [Interview Questions](#qa)

---

## 1. Why Separate Analytics {#why}

Running `SELECT region, SUM(amount) FROM orders JOIN … GROUP BY …` over 500M rows on the primary:
- Saturates I/O and CPU, which slows customer-facing queries.
- Long-running reads hold back MVCC cleanup (PG bloat) or grow undo history (MySQL).
- Row-store layout means reading every column of every row when you need two.
- Schemas are normalized for writes, not for analysis.

Solution: **replicate data to an analytical system** designed for scans and aggregations: a warehouse (Snowflake, BigQuery, Redshift, Databricks SQL), a real-time OLAP store (ClickHouse, Druid, Pinot), or embedded OLAP (DuckDB).

---

## 2. Warehouse Architecture {#arch}

- **Columnar storage** + compression + vectorized execution (see `fundamentals/04-storage-engine-internals.md`).
- **MPP (massively parallel processing)**: a query is split across many nodes/slices, each scanning its portion. Data is distributed by key, and joins need **shuffles** (redistribution) unless co-located.
- **Separation of storage and compute** (Snowflake, BigQuery, Databricks, Redshift RA3/Serverless): data lives in object storage, and independent compute clusters scale up and down, so many teams can query the same data without contention. You pay for compute time.
- **Partition pruning / micro-partitions / clustering keys / zone maps**: skip data blocks whose min/max excludes the predicate.
- **Materialized views, result caches, search optimization indexes**.
- Cost model: BigQuery bills by bytes scanned (on-demand), so `SELECT *` costs money; use partitioned and clustered tables. Snowflake bills by warehouse-seconds.

---

## 3. Dimensional Modeling (Kimball) {#modeling}

- **Fact tables**: measurements of business events, at a declared **grain** (one row per order line, per page view). Numeric measures (amount, qty) + foreign keys to dimensions. Huge and append-mostly.
- **Dimension tables**: descriptive context (customer, product, date, store). Wide, relatively small, and denormalized (the product dimension contains category and brand names directly).
- **Star schema**: fact in the center, dimensions around it, one join hop. **Snowflake schema** normalizes dimensions further (more joins, rarely worth it in columnar warehouses).
- **Date dimension**: a precomputed calendar table (day, week, month, quarter, fiscal periods, holidays).
- **Surrogate keys** in dimensions, decoupled from source system IDs.
- **Slowly Changing Dimensions (SCD)**: how to handle attribute changes (a customer moves city):
  | Type | Behavior |
  |---|---|
  | 0 | Never change |
  | 1 | Overwrite (history lost) |
  | **2** | **New row with validity dates** (`valid_from`, `valid_to`, `is_current`). Facts link to the version current at event time. Most common for history |
  | 3 | Extra column for the previous value |
  | 4 | Separate history table |
  | 6 | Hybrid 1+2+3 |
- **One Big Table (OBT)**: fully denormalized wide tables are popular in columnar systems because joins are the expensive part.
- Other approaches: **Data Vault** (hubs, links, satellites; auditable, for enterprise integration layers), **Inmon** (normalized enterprise DW then marts).
- Layering: **bronze/raw → silver/staging (cleaned, conformed) → gold/marts (business-ready)** (the medallion architecture).

---

## 4. ETL vs ELT {#elt}

- **ETL** (Extract → Transform → Load): transform before loading (Informatica, SSIS, custom Spark jobs). Historically necessary when warehouse compute was expensive.
- **ELT** (Extract → Load raw → Transform inside the warehouse with SQL): the modern default since warehouses scale compute elastically.

Modern data stack:
| Stage | Tools |
|---|---|
| Ingestion (EL) | Fivetran, Airbyte, Stitch, Meltano, AWS DMS, Debezium/Kafka Connect, custom |
| Storage/compute | Snowflake, BigQuery, Redshift, Databricks, ClickHouse, DuckDB/MotherDuck |
| Transformation | **dbt** (SQL models, tests, docs, lineage, incremental models), SQLMesh |
| Orchestration | Airflow, Dagster, Prefect, Temporal, cloud schedulers |
| BI | Looker, Metabase, Superset, Tableau, Power BI, Hex |
| Reverse ETL | Hightouch, Census (warehouse → SaaS tools) |
| Quality/observability | dbt tests, Great Expectations, Soda, Monte Carlo |
| Catalog/governance | DataHub, Amundsen, Unity Catalog, OpenMetadata |

**dbt** in one paragraph: each model is a `SELECT` in a `.sql` file. dbt resolves `ref()` dependencies into a DAG, materializes models as views, tables, or **incremental** tables (merge only new or changed rows), and runs **tests** (not_null, unique, relationships, accepted_values, custom). It also generates docs and lineage. It brings software engineering (git, CI, code review) to analytics SQL.

---

## 5. Change Data Capture (CDC) {#cdc}

Ways to extract changes from an OLTP DB:
| Method | How | Issues |
|---|---|---|
| Full dump | Copy everything periodically | Heavy, slow, no deletes history |
| Query-based incremental | `WHERE updated_at > :last_run` | Misses hard deletes; relies on correct `updated_at`; clock/transaction-commit-order edge cases (a long transaction commits with an older updated_at → skipped) |
| Trigger-based | Triggers write to a change table | Write overhead |
| **Log-based CDC** | Read the WAL (PG logical decoding) / binlog (MySQL) / oplog (Mongo) | Captures every change incl. deletes in commit order with low overhead. **Industry standard** |

**Debezium** (Kafka Connect source connectors for PG, MySQL, MongoDB, SQL Server, Oracle, …):
1. Initial **snapshot** (consistent; incremental snapshots via a signaling table avoid long locks).
2. Streams change events to Kafka topics (`server.schema.table`), with `before`, `after`, `op` (c/u/d/r), `source` (LSN/binlog position, txId, ts).
3. Schema changes are tracked (MySQL: schema history topic). Use Avro/Protobuf with a Schema Registry.
4. Sinks: JDBC, Elasticsearch, S3, Snowflake, BigQuery, ClickHouse connectors, or Flink.

CDC concerns:
- **Replication slots** (PG) must be monitored (WAL retention!).
- Ordering is guaranteed per key/partition (topic partitioned by primary key).
- Exactly-once into the sink needs idempotent upserts by PK + source position/version.
- Toasted unchanged columns in PG updates aren't included unless `REPLICA IDENTITY FULL` (Debezium marks them with a placeholder value).
- The **outbox pattern** with CDC: publish domain events from an `outbox` table rather than raw table changes. This decouples the internal schema from the event contracts (Debezium's outbox event router).

Managed/alternative: AWS DMS, Google Datastream, Fivetran HVR, Airbyte CDC, PeerDB (PG → warehouses, acquired by ClickHouse), Estuary, Sequin.

---

## 6. Batch vs Streaming {#streaming}

| | Batch | Stream |
|---|---|---|
| Latency | Minutes–hours | Seconds–sub-second |
| Tools | Spark, dbt, SQL in warehouse, Hadoop MapReduce (legacy) | **Flink**, Kafka Streams, Spark Structured Streaming, Materialize, RisingWave, ksqlDB, Beam/Dataflow |
| Correctness | Easy (recompute) | Harder: late/out-of-order events, state, exactly-once |
| Cost | Cheaper | More expensive to operate |

Stream processing concepts:
- **Event time vs processing time**: use event time for correct windowed aggregations.
- **Windows**: tumbling (fixed, non-overlapping), hopping/sliding (overlapping), session (gap-based).
- **Watermarks**: "we believe all events up to time T have arrived". Late events beyond allowed lateness go to a side output or get dropped.
- **State**: keyed state with checkpoints (Flink: RocksDB state backend + periodic distributed snapshots, Chandy–Lamport-style).
- **Exactly-once**: Flink checkpoints + transactional sinks (two-phase commit to Kafka); Kafka Streams with `processing.guarantee=exactly_once_v2`.
- **Lambda architecture** (batch + speed layer, merged at query time; two codebases) vs **Kappa architecture** (one streaming pipeline; reprocess by replaying the log).

---

## 7. Data Lakes and the Lakehouse {#lakehouse}

- **Data lake**: raw files (JSON, CSV, **Parquet**, Avro, ORC) in object storage. Cheap and flexible, but historically a "swamp": no transactions, no schema enforcement, slow listing, painful updates and deletes.
- **Open table formats** add database features on top of Parquet files in S3:
  - **Apache Iceberg** (Netflix; now the de facto standard backed by AWS, Snowflake, Google, Databricks via Tabular acquisition), **Delta Lake** (Databricks), **Apache Hudi** (Uber).
  - Features: ACID commits via metadata/manifest files + atomic pointer swap in a catalog, **schema evolution**, **hidden partitioning** and partition evolution, **time travel** (query a snapshot), row-level updates and deletes (copy-on-write or merge-on-read), compaction.
- **Lakehouse**: warehouse-like SQL engines (Spark, Trino/Presto, Snowflake, BigQuery, Athena, DuckDB, StarRocks, Dremio) querying open table formats directly, so there's one copy of the data and many engines.
- Catalogs: Hive Metastore (legacy), AWS Glue, Iceberg REST catalogs (Polaris, Nessie, Unity Catalog).
- **Parquet** internals: row groups → column chunks → pages, with min/max statistics per row group/page (predicate pushdown), dictionary and RLE encoding, nested types (Dremel encoding).

---

## 8. ClickHouse Deep Dive {#clickhouse}

Open-source columnar OLAP DBMS (Yandex 2016; ClickHouse Inc.). It's extremely fast for aggregations over billions of rows on modest hardware. Users include Cloudflare (HTTP analytics), Uber (logging), eBay, and many observability products (SigNoz, PostHog, Langfuse, Highlight).

**MergeTree engine family**:
```sql
CREATE TABLE events (
  event_date  Date DEFAULT toDate(ts),
  ts          DateTime64(3),
  tenant_id   UInt32,
  user_id     UInt64,
  event       LowCardinality(String),
  properties  String,              -- or JSON type
  duration_ms UInt32
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (tenant_id, event, ts)          -- sorting key = primary index
TTL event_date + INTERVAL 180 DAY;
```
- Inserts create immutable **parts** (sorted by the ORDER BY key), and background **merges** combine parts (an LSM-like structure).
- **Sparse primary index**: one index entry per **granule** (8192 rows by default), not per row. It's tiny and always in memory. Queries filtering on a prefix of the ORDER BY key skip granules. **Choose ORDER BY by query filters**, with low-cardinality columns first.
- **Data-skipping indexes** (minmax, set, bloom_filter, ngrambf) for other columns.
- **Codecs**: LZ4 (default), ZSTD, Delta, DoubleDelta, Gorilla (time series), T64. Typical compression is 5–20×.
- `LowCardinality(String)`: dictionary encoding.
- **Engines**: `ReplacingMergeTree` (dedup by key at merge time, eventually; `FINAL` for query-time dedup), `SummingMergeTree`, `AggregatingMergeTree` (stores partial aggregate states: `uniqState`, `quantileState`), `CollapsingMergeTree`/`VersionedCollapsingMergeTree` (updates via sign rows), `ReplicatedMergeTree` (replication via ClickHouse Keeper/ZooKeeper), `Distributed` (sharding across nodes), `Kafka` engine (consume topics), `S3`/`Iceberg` table functions.
- **Materialized views** in ClickHouse are **insert triggers**: on each insert into the source, they transform the block and write into a target table (often AggregatingMergeTree), which builds real-time rollups cheaply.
- Caveats:
  - Prefer **big batched inserts** (thousands+ rows per insert). Many tiny inserts cause "too many parts" errors. **Async inserts** help.
  - Updates and deletes are **mutations** (heavy, asynchronous rewrites of parts). Lightweight `DELETE` (mask-based) and newer lightweight updates exist, but design for append-only.
  - Joins: improved a lot (hash, grace hash, parallel hash, full sorting merge), but denormalize where you can. Use dictionaries for dimension lookups.
  - No full ACID transactions. Eventual dedup with ReplacingMergeTree.
  - Uniqueness isn't enforced by the primary key (it's a sort key).

---

## 9. Real-Time OLAP: Druid, Pinot {#realtime}

- **Apache Druid** and **Apache Pinot** (LinkedIn; also Uber "Eats" analytics, Stripe) target **user-facing analytics**: thousands of concurrent queries with sub-second latency on fresh streaming data (Kafka ingestion), with pre-aggregation (rollups), inverted/star-tree indexes, and time partitioning.
- **StarRocks / Apache Doris**: MPP OLAP with MySQL protocol, good joins, real-time upserts.
- **Materialize / RisingWave**: streaming databases that maintain SQL materialized views incrementally (Postgres wire protocol).
- **Tinybird**: managed ClickHouse with API endpoints.

---

## 10. Data Quality, Governance, Privacy {#quality}

- **Data contracts**: producers (backend teams) define schemas and guarantees for the events and tables consumed downstream. Enforce them with a schema registry (backward-compatibility checks) and CI.
- Tests: freshness, volume anomalies, null rates, uniqueness, referential integrity, distribution drift.
- Lineage: know which dashboards break if a column changes.
- **PII**: classify columns, mask/tokenize/hash in analytical copies, apply access policies (row/column-level security in warehouses), honor deletion requests (GDPR/DPDP "right to erasure" must propagate to warehouses and lakes, so open table formats with row-level deletes help), and set retention policies.
- Cost governance: query budgets, partitioned/clustered tables, auto-suspend warehouses.

---

## 11. Interview Questions {#qa}

1. Why shouldn't you run analytics queries on your production OLTP database? What are the alternatives?
2. Explain star schema, fact vs dimension tables, and grain.
3. What's an SCD Type 2 and how do you query "revenue by customer city at the time of order"?
4. ETL vs ELT. What does dbt do?
5. How does log-based CDC work and why is it better than `updated_at` polling?
6. Explain event time, watermarks, and windowing in stream processing.
7. What problems do Iceberg/Delta solve on top of Parquet in S3?
8. Why is ClickHouse so fast? What is a granule and how should you choose the ORDER BY?
9. Design a real-time analytics dashboard for 1M events/s with sub-second queries.
