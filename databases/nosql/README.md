# NoSQL: Concepts, Data Modeling, and When to Use It

> "NoSQL" is an umbrella term for non-relational stores. They differ from each other as much as they differ from SQL. Each one gives up something (joins, ad-hoc queries, strong consistency, schemas) to get something else (horizontal scale, write throughput, flexible documents, low latency).

## Files in this folder
| File | Store | Model |
|---|---|---|
| [mongodb.md](mongodb.md) | MongoDB | Document |
| [cassandra.md](cassandra.md) | Apache Cassandra / ScyllaDB | Wide-column, leaderless |
| [dynamodb.md](dynamodb.md) | Amazon DynamoDB | Key-value / document, managed |
| [elasticsearch.md](elasticsearch.md) | Elasticsearch / OpenSearch | Search engine (inverted index) |
| Redis | see [`caching/redis/redis.md`](../../caching/redis/redis.md) | In-memory data structures |

## Table of Contents
1. [Why NoSQL Happened](#why)
2. [The Families at a Glance](#families)
3. [Consistency Models You'll Meet](#consistency)
4. [Partitioning and Replication Patterns](#partitioning)
5. [Query-First Data Modeling](#modeling)
6. [Common Misconceptions](#myths)
7. [Choosing](#choosing)

---

## 1. Why NoSQL Happened {#why}

Mid-2000s web giants hit limits of single-node relational databases:
- **Google Bigtable** (2006 paper) → HBase, Cassandra's data model.
- **Amazon Dynamo** (2007 paper): highly available, leaderless key-value store for the shopping cart. It was "always writeable", with eventual consistency, consistent hashing, vector clocks, sloppy quorums, hinted handoff, and Merkle-tree anti-entropy → Riak, Cassandra's distribution model, Voldemort.
- **Document stores** (CouchDB 2005, MongoDB 2009) matched JSON-centric web development and schema agility.

Drivers: horizontal scale on commodity hardware, availability under partitions (CAP), flexible schemas, and specialized access patterns.

Since then, relational databases absorbed many NoSQL strengths (JSON, horizontal scaling via Citus/Vitess/distributed SQL), and NoSQL databases added SQL-like features (MongoDB transactions, Cassandra lightweight transactions, DynamoDB PartiQL and transactions). The lines blurred, but the **core trade-offs remain**.

---

## 2. Families at a Glance {#families}

| Family | Examples | Data model | Strengths | Weaknesses |
|---|---|---|---|---|
| Key-Value | Redis, DynamoDB, Riak, etcd, Memcached | Opaque value by key | O(1) speed, simple scaling | No queries beyond key (Redis has structures) |
| Document | MongoDB, Couchbase, Firestore, CosmosDB | JSON/BSON docs, nested | Natural mapping to objects, flexible schema, secondary indexes | Joins limited; denormalization → update anomalies |
| Wide-column | Cassandra, ScyllaDB, HBase, Bigtable | Partition key → sorted rows of columns | Massive write throughput, multi-DC, predictable latency | Query-first rigidity; no joins; limited secondary indexes |
| Search | Elasticsearch, OpenSearch, Solr | Documents in inverted indexes | Full-text relevance, aggregations, log analytics | Not a system of record; eventual (near-real-time) |
| Graph | Neo4j, Neptune, JanusGraph | Nodes + edges + properties | Multi-hop traversals | Scaling writes/sharding graphs is hard |
| Time-series | InfluxDB, TimescaleDB, Prometheus, QuestDB | Timestamped points/series | Compression, downsampling, retention | Narrow use case |

---

## 3. Consistency Models {#consistency}

From strongest to weakest:
- **Linearizable / strong**: every read sees the latest committed write, as if there were a single copy. (DynamoDB strongly consistent reads; MongoDB majority read/write concern + linearizable read concern; Cassandra QUORUM/QUORUM *approximately*, not truly linearizable without LWT.)
- **Sequential**: all clients see operations in the same order, though not necessarily real-time order.
- **Causal**: causally related operations are seen in order by everyone (MongoDB causal consistency sessions).
- **Read-your-writes, monotonic reads, monotonic writes, writes-follow-reads**: session guarantees.
- **Eventual**: replicas converge if writes stop. No ordering promises in the meantime.

**Tunable consistency** (Cassandra, DynamoDB, CosmosDB): choose per operation. CosmosDB offers five named levels: Strong, Bounded Staleness, Session, Consistent Prefix, Eventual.

See [`scaling-db/cap.md`](../../scaling-db/cap.md) for CAP and PACELC: during a **P**artition, choose **A**vailability or **C**onsistency; **E**lse choose **L**atency or **C**onsistency. Dynamo-style systems are PA/EL; Spanner-like systems are PC/EC.

---

## 4. Partitioning and Replication {#partitioning}

| Technique | Used by |
|---|---|
| **Consistent hashing with virtual nodes** (token ring) | Cassandra, Dynamo, Riak, ScyllaDB |
| **Hash partitioning** into a fixed or dynamic number of partitions | DynamoDB (partitions split automatically), MongoDB hashed sharding |
| **Range partitioning** (key ranges split as they grow) | MongoDB ranged sharding, HBase/Bigtable regions, CockroachDB/TiKV ranges |
| **Leader–follower per partition** | MongoDB replica sets (per shard), DynamoDB (Paxos leader per partition), Kafka |
| **Leaderless quorum** | Cassandra, Riak, Dynamo |

Hot partitions are the universal enemy: a celebrity user, a popular product, "today's date" as a key. Mitigations are high-cardinality keys, **write sharding** (key suffix 0..N), caching hot reads, and adaptive capacity features.

---

## 5. Query-First Data Modeling {#modeling}

Relational: model the **entities**, normalize, then write any query you like (the optimizer figures it out).
NoSQL (especially wide-column/KV): **list the queries first**, then design tables/documents so each query is a single-partition lookup.

Process:
1. Enumerate access patterns with frequency, latency, and consistency needs ("get user's last 20 orders, sorted by date; 2k req/s; p99 < 20 ms").
2. Choose partition keys giving **even distribution** and **query locality** (all data for one query in one partition).
3. Choose sort/clustering keys to support ordering and range queries within the partition.
4. **Denormalize/duplicate** data per query pattern. Writes fan out to multiple tables or items.
5. Plan how duplicates stay in sync (application writes, batches, CDC streams, materialized views).
6. Bound partition sizes (time bucketing: `user_id + month`).

Document modeling (MongoDB): **embed** what's read together and bounded in size; **reference** what's unbounded, shared, or updated independently.

---

## 6. Misconceptions {#myths}

- "NoSQL is schemaless." The schema just moves into application code (implicit schema). You still need migrations: versioned documents, lazy migration on read, backfills. Use schema validation (MongoDB JSON Schema validator) where possible.
- "NoSQL doesn't support transactions." MongoDB (multi-document ACID since 4.0), DynamoDB (TransactWriteItems), Cassandra (lightweight transactions via Paxos; general ACID transactions via the Accord protocol, CEP-15, are being added in releases after 5.0), FoundationDB (fully ACID).
- "NoSQL is faster." It's faster *for the access patterns it was modeled for*. Ad-hoc queries can be far slower or impossible.
- "NoSQL scales, SQL doesn't." A well-tuned Postgres/MySQL handles enormous load, and Vitess/Citus/Spanner scale SQL horizontally.
- "MongoDB loses data." Old defaults (unacknowledged writes before 2012, `w:1`) gave it that reputation. Since 5.0 the default write concern is `majority`.

---

## 7. Choosing {#choosing}

See [`../fundamentals/09-choosing-a-database.md`](../fundamentals/09-choosing-a-database.md). Quick heuristics:
- **Need joins, ad-hoc queries, and strong integrity?** Relational.
- **Self-contained aggregates, evolving shape, moderate scale?** MongoDB (or PG jsonb).
- **Huge write volume, time-ordered per key, multi-region always-on?** Cassandra/ScyllaDB.
- **Serverless, predictable single-digit-ms key access at any scale, AWS-native?** DynamoDB.
- **Search, relevance, log analytics?** Elasticsearch/OpenSearch (as a derived store).
- **Sub-ms ephemeral data, counters, leaderboards, rate limits?** Redis.
