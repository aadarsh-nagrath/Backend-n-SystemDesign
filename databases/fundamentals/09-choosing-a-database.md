# Choosing a Database: The Landscape and a Decision Framework

> "Which database should we use?" is a top system-design question. The honest default answer for most products is **PostgreSQL (or MySQL) until proven otherwise**, plus purpose-built stores for specific access patterns. This note maps the landscape and gives you a defensible way to decide.

## Table of Contents
1. [Workload Types: OLTP, OLAP, HTAP, Streaming](#workloads)
2. [The Database Families](#families)
3. [Decision Framework (questions to ask)](#framework)
4. [PostgreSQL vs MySQL — Short Version](#pgvsmysql)
5. [When SQL Is the Wrong Tool](#notsql)
6. [Polyglot Persistence — and Its Costs](#polyglot)
7. [Managed vs Self-Hosted](#managed)
8. [Common System-Design Mappings](#mappings)
9. [Interview Answer Template](#template)

---

## 1. Workload Types {#workloads}

| | OLTP | OLAP | HTAP |
|---|---|---|---|
| Purpose | Run the business (orders, payments, users) | Analyze the business (reports, dashboards, ML features) | Both on the same data |
| Queries | Short, many, point lookups & small ranges by key | Long, few, scan & aggregate billions of rows | Mixed |
| Writes | Many small transactional writes | Bulk loads / streaming appends | Both |
| Data | Current state, normalized | Historical, denormalized (star/snowflake), columnar | |
| Latency | ms | seconds–minutes acceptable | |
| Examples | PostgreSQL, MySQL, SQL Server, Oracle, Aurora, Spanner, CockroachDB | Snowflake, BigQuery, Redshift, ClickHouse, Databricks, DuckDB, Druid, Pinot | TiDB, SingleStore, MySQL HeatWave, AlloyDB |

**Streaming / event logs**: Kafka, Redpanda, Pulsar, Kinesis — not databases in the classic sense, but often the "source of truth" pipe between systems.

---

## 2. Database Families {#families}

### Relational (SQL)
PostgreSQL, MySQL/MariaDB, SQL Server, Oracle, SQLite.
- Strengths: ACID transactions, joins, constraints, ad-hoc queries, mature tooling, decades of operational knowledge.
- Limits: single-writer scale ceiling (vertical scaling + read replicas go very far — tens of TB, tens of thousands of TPS), sharding is manual (or via Citus/Vitess).
- Modern Postgres also covers JSON documents, full-text search, geospatial (PostGIS), vectors (pgvector), time-series (Timescale), queues (SKIP LOCKED / pgmq).

### Distributed SQL / NewSQL
Google Spanner, CockroachDB, YugabyteDB, TiDB, Aurora DSQL, Neon/AlloyDB (disaggregated storage rather than sharded).
- Horizontal scale *with* SQL and serializable/strong transactions; data auto-sharded into ranges replicated with Raft/Paxos.
- Cost: higher per-query latency (consensus round trips), cross-region writes pay WAN latency, operational complexity, some SQL feature gaps.
- Use when: you've genuinely outgrown a single primary, need multi-region strong consistency, or want HA without a failover story.

### Key-Value
Redis/Valkey, Memcached, DynamoDB (KV + document), etcd/Consul (config, consensus), RocksDB/LevelDB (embedded), FoundationDB.
- O(1) access by key; minimal query capabilities.
- Use: caches, sessions, rate limiters, leaderboards (Redis sorted sets), feature flags, distributed locks (carefully), service config.

### Document
MongoDB, Couchbase, Firestore, CosmosDB, DynamoDB (document model), PG `jsonb`.
- Schema-flexible JSON/BSON documents; good when data is naturally hierarchical and accessed as a unit (a product with nested variants).
- Weaker at cross-document relations/joins (MongoDB has `$lookup` and multi-document transactions since 4.0, but design favors embedding).

### Wide-Column
Apache Cassandra, ScyllaDB, HBase, Google Bigtable.
- Partitioned rows with sorted clustering columns; LSM storage; linear write scalability; multi-DC replication.
- Query-first modeling: one table per query pattern; no joins, limited secondary indexes.
- Use: massive write throughput (IoT, messaging inbox, activity logs, time series), always-on multi-region (AP).

### Search Engines
Elasticsearch, OpenSearch, Solr, Meilisearch, Typesense, Vespa.
- Inverted indexes, relevance scoring (BM25), faceting, fuzzy matching, aggregations on logs.
- Not a primary store: feed via CDC; near-real-time (refresh interval ~1 s).

### Time-Series
TimescaleDB (PG extension), InfluxDB, Prometheus (metrics TSDB), VictoriaMetrics, QuestDB, ClickHouse (often), Druid.
- Time-partitioned, compressed, downsampling, retention policies, append-optimized.

### Graph
Neo4j, Amazon Neptune, JanusGraph, TigerGraph, Memgraph, PG + Apache AGE.
- Traversals of many hops (friends-of-friends, fraud rings, recommendation, knowledge graphs) where SQL recursive joins get painful.
- Query languages: Cypher (and ISO GQL), Gremlin, SPARQL (RDF).

### Vector Databases
Pinecone, Weaviate, Milvus, Qdrant, Chroma; pgvector, Elasticsearch/OpenSearch kNN, Redis, MongoDB Atlas Vector Search.
- Approximate nearest neighbor (HNSW, IVF, DiskANN) over embeddings — semantic search, RAG for LLM apps, recommendations.
- Start with pgvector if you already run PG and have < tens of millions of vectors; dedicated stores for very large scale, filtering-heavy, or multi-tenant ANN.

### Object Storage
S3, GCS, Azure Blob, MinIO, R2.
- Blobs (images, video, backups, data lake files in Parquet/Iceberg/Delta). 11 nines durability, cheap, high latency (10s of ms), strong read-after-write consistency on S3 since 2020.

### Embedded
SQLite (most deployed DB in the world; also growing on the server: Litestream, LiteFS, Turso/libSQL, Cloudflare D1), DuckDB (embedded OLAP), RocksDB.

### Ledger / Immutable
Amazon QLDB (being retired), immudb, or just append-only tables with hash chaining in PG.

---

## 3. Decision Framework {#framework}

Ask, in order:

1. **What are the access patterns?** List the top 5–10 queries with expected QPS and latency. Key lookups? Ranges? Ad-hoc filters? Full-text? Aggregations? Graph traversals?
2. **What consistency is required?** Money/inventory/auth → strong + transactions. Feeds/likes/analytics → eventual is fine.
3. **Data shape and relationships?** Highly relational (orders ↔ customers ↔ products) → relational. Self-contained aggregates → document/KV. Many-hop relationships → graph.
4. **Scale — honestly estimated?** Data size in 1–3 years, write QPS, read QPS. A single PG/MySQL instance on modern hardware handles TBs and 10k+ writes/s. Most products never need more.
5. **Write vs read ratio** — write-heavy append (LSM/wide-column/time-series) vs read-heavy (relational + caches + replicas).
6. **Latency & geography** — single region vs global users; multi-region writes?
7. **Operational capacity** — does the team know how to run it? Is there a managed offering? How mature is backup/restore/upgrade tooling?
8. **Ecosystem** — drivers, ORMs, CDC connectors, monitoring, hiring.
9. **Cost** — licensing, managed pricing models (DynamoDB per-request vs provisioned), egress.
10. **Compliance** — data residency, encryption, audit.

**Default**: PostgreSQL as the system of record, Redis for cache/ephemeral, object storage for blobs, a search engine if search is a product feature, a warehouse for analytics. Add specialized stores only when a concrete requirement demands it.

---

## 4. PostgreSQL vs MySQL — Short Version {#pgvsmysql}

Detailed comparison: [`mysql/09-mysql-vs-postgresql.md`](../mysql/09-mysql-vs-postgresql.md).

| Choose PostgreSQL when | Choose MySQL when |
|---|---|
| Rich SQL (window functions, CTEs, partial/expression indexes, transactional DDL) | Simple, high-QPS OLTP with primary-key access patterns |
| Advanced types: JSONB with GIN, arrays, ranges, PostGIS, pgvector | You need Vitess-scale sharding or PlanetScale |
| Strong correctness defaults, constraints (EXCLUDE), RLS | Team/ecosystem is MySQL-heavy (PHP/WordPress/Laravel, legacy) |
| Extensions ecosystem (Timescale, Citus, pg_cron…) | Mature, simple replication & huge operational playbooks at web scale (Meta, Uber historically, Shopify, GitHub) |
| Analytical-ish queries alongside OLTP | Clustered PK layout benefits your access pattern |

Both are excellent. The bigger risk is using either badly.

---

## 5. When SQL Is the Wrong Tool {#notsql}

- **Extreme write ingest with simple access by key/time** (millions/s IoT telemetry) → Cassandra/Scylla, time-series DBs, ClickHouse.
- **Huge analytical scans** → columnar warehouse.
- **Relevance-ranked fuzzy text search** → search engine (PG FTS is fine for simple needs).
- **Sub-millisecond ephemeral state** → Redis.
- **Deep graph traversals** with variable depth over billions of edges → graph DB.
- **Global active-active writes with partition tolerance over consistency** → Dynamo-style stores.
- **Large binary objects** → object storage.

---

## 6. Polyglot Persistence and Its Costs {#polyglot}

Using multiple specialized stores is powerful but adds:
- Data synchronization (CDC pipelines, outbox, dual-write bugs).
- Consistency gaps between stores (search shows a product that was deleted).
- More things to back up, monitor, secure, upgrade, and be on call for.
- Team cognitive load.

Rule: **one system of record per piece of data**; other stores are derived views that can be rebuilt from it.

---

## 7. Managed vs Self-Hosted {#managed}

| | Managed (RDS, Aurora, Cloud SQL, AlloyDB, Azure Flexible Server, Neon, Supabase, PlanetScale, Crunchy Bridge, Atlas, DynamoDB) | Self-hosted (VMs / Kubernetes operators) |
|---|---|---|
| Ops burden | Backups, PITR, failover, patching handled | All yours |
| Control | Limited superuser, extensions allowlist, config subsets | Full |
| Cost | Higher per resource; cheaper in people | Cheaper hardware; expensive expertise |
| Performance | Network storage (EBS/io2) latency; Aurora/AlloyDB custom storage | Local NVMe possible (much faster, but you handle durability) |
| Lock-in | Aurora/DynamoDB features are proprietary | |

Most teams should start managed.

---

## 8. Common System-Design Mappings {#mappings}

| Component | Typical choice |
|---|---|
| Users, accounts, orders, payments | PostgreSQL / MySQL |
| Sessions, rate limits, short-lived tokens | Redis |
| Product catalog with variable attributes | PG with JSONB, or MongoDB |
| Chat messages at WhatsApp/Discord scale | Cassandra/ScyllaDB (Discord moved MongoDB → Cassandra → ScyllaDB) |
| News feed timelines | Redis (fan-out cache) + Cassandra/MySQL for persistence |
| URL shortener mapping | KV (DynamoDB/Redis) or sharded MySQL |
| Search box | Elasticsearch/OpenSearch fed by CDC |
| Metrics/monitoring | Prometheus/VictoriaMetrics/Mimir; logs in Loki/Elasticsearch/ClickHouse |
| Event analytics / product analytics | ClickHouse / BigQuery / Snowflake |
| Social graph / recommendations | Graph DB, or adjacency tables in MySQL (Facebook TAO on MySQL) |
| Location / nearby drivers | Redis GEO, PostGIS, geohash/S2/H3 cells in a KV store |
| Files, images, video | S3 + CDN; metadata in SQL |
| Leaderboards | Redis sorted sets |
| Distributed locks / leader election / config | etcd, ZooKeeper, Consul |
| Job queue (moderate scale) | PG `SKIP LOCKED` / Redis (Sidekiq, BullMQ) → SQS/RabbitMQ/Kafka as scale grows |
| Semantic search / RAG | pgvector → dedicated vector DB at scale |

---

## 9. Interview Answer Template {#template}

> "For the core transactional data — users, orders, payments — I'd use **PostgreSQL**, because we need multi-row ACID transactions and relational integrity; at our estimated 5k writes/s and 2 TB in two years a single primary with read replicas handles it, with a path to sharding by `tenant_id` via Citus if needed. Hot reads go through **Redis** with cache-aside and TTLs. Product search uses **OpenSearch** fed by **CDC (Debezium → Kafka)** from Postgres so the DB stays the source of truth. Images go to **S3** behind a CDN. Analytics events stream through Kafka into **ClickHouse**. The trade-off I'm accepting is eventual consistency between Postgres and search (~seconds), which is fine for this feature."

Name the access pattern → the property needed → the store that provides it → the trade-off you accept.
