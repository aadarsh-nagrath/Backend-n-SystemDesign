# Apache Kafka Deep Dive

## Table of Contents
1. [What Kafka Is](#what)
2. [Core Concepts: Topics, Partitions, Offsets, Brokers](#core)
3. [Storage Internals: Segments, Indexes, Page Cache, Zero-Copy](#storage)
4. [Replication: Leaders, ISR, High Watermark](#replication)
5. [KRaft (ZooKeeper Removal)](#kraft)
6. [Producers: Partitioning, Batching, acks, Idempotence](#producers)
7. [Consumers and Consumer Groups: Rebalancing, Offsets](#consumers)
8. [Delivery Semantics and Transactions (EOS)](#eos)
9. [Retention and Log Compaction](#retention)
10. [Kafka Connect, Schema Registry, Kafka Streams, ksqlDB, Flink](#ecosystem)
11. [Topic and Partition Design](#design)
12. [Performance Tuning](#tuning)
13. [Operations and Monitoring](#ops)
14. [Kafka Alternatives and Managed Services](#alternatives)
15. [Common Patterns and Anti-Patterns](#patterns)
16. [Interview Questions](#qa)

---

## 1. What Kafka Is {#what}

A **distributed, partitioned, replicated commit log**, built at LinkedIn (2011, Jay Kreps, Neha Narkhede, Jun Rao) and now an Apache project, with Confluent as the main commercial company. It's used for:
- **Event streaming** between microservices (event-driven architecture backbone).
- **Data pipelines / CDC**: databases → Kafka → warehouses, search, caches.
- **Log and metrics aggregation**, activity tracking (its original LinkedIn use).
- **Stream processing** (Kafka Streams, Flink).
- **Event sourcing / durable event store** (with compaction or infinite retention + tiered storage).

Why it's fast: sequential disk I/O, the OS page cache, batching + compression end-to-end, zero-copy transfer, and partitioned parallelism. Clusters handle millions of messages per second.

---

## 2. Core Concepts {#core}

```
Topic "orders" (3 partitions, RF=3)
 Partition 0: [0][1][2][3][4][5][6] →  leader on broker 1, followers on 2,3
 Partition 1: [0][1][2][3]           →  leader on broker 2
 Partition 2: [0][1][2][3][4]        →  leader on broker 3
              ↑ offsets (per partition, monotonically increasing)
```
- **Record**: key (optional), value, headers, timestamp. Typically KB-sized (default max ~1 MB, `message.max.bytes`). For big payloads, store them in S3 and send a reference (the "claim check" pattern).
- **Topic**: a named stream, split into **partitions**.
- **Partition**: an ordered, immutable, append-only log. **Ordering is guaranteed only within a partition.** It's the unit of parallelism and of replication.
- **Offset**: position of a record within a partition.
- **Broker**: a Kafka server hosting partition replicas.
- **Producer** writes, **consumer** reads (pull model), and a **consumer group** shares partitions among its members.

---

## 3. Storage Internals {#storage}

Each partition replica is a directory of **segments**:
```
orders-0/
  00000000000000000000.log        # record batches
  00000000000000000000.index      # sparse offset → file position
  00000000000000000000.timeindex  # timestamp → offset
  00000000000005367851.log        # next segment (named by base offset)
  ...
  leader-epoch-checkpoint
```
- Writes append to the **active segment**. Segments roll by size (`segment.bytes`, 1 GB) or time (`segment.ms`, 7 days).
- Retention deletes **whole old segments** (cheap: unlink a file).
- Records are written in **batches** (the producer's batch is stored as-is, compressed), and brokers don't recompress if the codec matches.
- **Page cache**: Kafka relies on the OS cache rather than its own heap cache. Consumers reading the tail are served from memory. Give brokers lots of RAM for page cache and a modest JVM heap (~6 GB).
- **Zero-copy** (`sendfile`): data goes from the page cache to the socket without passing through user space (not possible with TLS, which needs encryption in user space; kTLS can help).
- Durability comes from **replication**, not fsync per message. By default Kafka doesn't fsync on every write (it leaves flushing to the OS), and it relies on replicas on other machines.
- **Tiered storage** (KIP-405, GA in Kafka 3.9): older segments are offloaded to object storage (S3). This enables long or infinite retention cheaply and faster broker rebuilds.

---

## 4. Replication {#replication}

- Each partition has a **replication factor** (RF, typically 3) with one **leader** replica (handles all reads and writes by default) and **followers** that fetch from the leader.
- **ISR (In-Sync Replicas)**: replicas caught up within `replica.lag.time.max.ms` (30 s). Lagging followers drop out of the ISR.
- **High watermark (HW)**: the highest offset replicated to all ISR members. **Consumers only see records up to the HW** (committed records).
- **`acks` + `min.insync.replicas`** decide durability:
  - `acks=all` + `min.insync.replicas=2` (with RF=3): a write succeeds only when at least 2 replicas have it. You tolerate one broker loss without data loss, and if the ISR shrinks below 2, producers get `NotEnoughReplicas` errors (availability traded for safety).
- **Leader election**: if the leader dies, the controller picks a new leader from the ISR. **`unclean.leader.election.enable=false`** (the default) prevents electing an out-of-sync replica, which would lose data.
- **Leader epochs** prevent log divergence after failovers (followers truncate to the right point).
- **Rack awareness** (`broker.rack`) spreads replicas across AZs. **Follower fetching** (KIP-392) lets consumers read from the nearest replica in their AZ, which cuts cross-AZ transfer costs.

---

## 5. KRaft {#kraft}

Historically Kafka used **ZooKeeper** for metadata (brokers, topics, partition leaders, controller election). **KRaft** (Kafka Raft, KIP-500) replaces it with an internal Raft quorum of **controller** nodes storing metadata in an internal log (`__cluster_metadata`).
- Production-ready since 3.3. ZooKeeper mode is deprecated in 3.x and **removed in Kafka 4.0 (2025)**.
- Benefits: one system to operate, faster controller failover, and support for millions of partitions per cluster.
- Deploy 3 or 5 controllers (dedicated or combined with brokers in small clusters).

---

## 6. Producers {#producers}

```java
Properties p = new Properties();
p.put("bootstrap.servers", "b1:9092,b2:9092");
p.put("acks", "all");                         // durability
p.put("enable.idempotence", "true");          // default true since 3.0: no duplicates on retry, ordering preserved
p.put("compression.type", "zstd");            // or lz4 / snappy
p.put("linger.ms", "10");                     // wait up to 10 ms to fill batches (5 ms default since 4.0)
p.put("batch.size", "65536");                 // bytes per partition batch
p.put("delivery.timeout.ms", "120000");       // total time incl. retries
KafkaProducer<String, String> producer = new KafkaProducer<>(p, new StringSerializer(), new StringSerializer());
producer.send(new ProducerRecord<>("orders", order.customerId(), json), (md, ex) -> { if (ex != null) log.error(...); });
```
- **Partitioning**: with a key, `murmur2(key) % numPartitions`, so the same key always goes to the same partition (ordering per key). With no key: the **sticky partitioner** (fills a batch for one partition, then switches), which improves batching.
- **Batching + compression** are the main throughput levers. `linger.ms` trades a few ms of latency for much bigger batches.
- `send()` is asynchronous. Records are buffered in `buffer.memory`, and a background thread sends batches. Calling `flush()` or `get()` per record kills throughput.
- **Idempotent producer**: the broker deduplicates retries using producer ID + sequence numbers per partition, which also preserves ordering with up to 5 in-flight requests.
- Hot keys mean hot partitions. One enormous customer can overload one partition.

---

## 7. Consumers and Consumer Groups {#consumers}

```java
props.put("group.id", "billing-service");
props.put("enable.auto.commit", "false");
props.put("auto.offset.reset", "earliest");      // where to start when no committed offset: earliest | latest
props.put("max.poll.records", "500");
props.put("max.poll.interval.ms", "300000");     // max time between polls before considered dead
props.put("partition.assignment.strategy", "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
consumer.subscribe(List.of("orders"));
while (running) {
  ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
  for (var r : records) process(r);              // idempotent!
  consumer.commitSync();                         // commit after processing → at-least-once
}
```
- Within a **consumer group**, each partition is assigned to **exactly one** member. Extra consumers beyond the partition count sit idle. **Different groups** each get all messages (pub/sub).
- **Offsets** are stored in the internal compacted topic `__consumer_offsets`. Commit after processing for at-least-once.
- **Rebalancing** happens when members join or leave, crash, or exceed `max.poll.interval.ms` (slow processing!), or when subscribed topics change:
  - Old **eager** protocol: stop-the-world. All partitions are revoked from everyone, then reassigned.
  - **Cooperative incremental** (CooperativeStickyAssignor): only moved partitions are revoked.
  - **Static membership** (`group.instance.id`): restarts within `session.timeout.ms` don't trigger rebalances, which helps rolling deploys.
  - **Next-gen consumer rebalance protocol** (KIP-848, GA in Kafka 4.0): the broker-side coordinator drives incremental assignment, which is much faster and less disruptive.
- **Consumer lag** = log end offset − committed offset. It's *the* key metric (Burrow, Kafka Lag Exporter, Confluent/MSK metrics).
- Pitfall: long processing per batch exceeds `max.poll.interval.ms`, the consumer gets kicked, and the result is a rebalance storm plus duplicate processing. Fixes: smaller `max.poll.records`, async processing with pause/resume, or a longer interval.
- **Share groups / "Queues for Kafka"** (KIP-932, early access in 4.0, preview in 4.1): queue semantics with per-message acks and more consumers than partitions.

---

## 8. Delivery Semantics and Transactions {#eos}

- **At-most-once**: commit offsets before processing.
- **At-least-once**: process then commit (the default practical choice) + idempotent consumers.
- **Exactly-once (EOS)** for **read-process-write within Kafka**:
  ```java
  producer.initTransactions();                       // transactional.id configured
  while (true) {
    var records = consumer.poll(...);
    producer.beginTransaction();
    for (var r : records) producer.send(transform(r));
    producer.sendOffsetsToTransaction(offsets(records), consumer.groupMetadata());
    producer.commitTransaction();                    // atomically: outputs visible + input offsets committed
  }
  ```
  Downstream consumers must use `isolation.level=read_committed` to skip aborted records. **Kafka Streams** gives EOS with one config (`processing.guarantee=exactly_once_v2`).
- EOS doesn't extend to external systems. For DB sinks, use idempotent upserts keyed by (topic, partition, offset) or a business ID, or store offsets in the same DB transaction as the results (then seek on startup).

---

## 9. Retention and Compaction {#retention}

- **Delete policy** (`cleanup.policy=delete`): remove segments older than `retention.ms` (7 days default) or beyond `retention.bytes`.
- **Compaction** (`cleanup.policy=compact`): keep at least the **latest record per key** (older versions removed by the log cleaner in the background). It's like a changelog/table snapshot.
  - Use cases: `__consumer_offsets`, Kafka Streams state changelogs, CDC topics representing current state, configuration topics, building materialized views by replaying.
  - **Tombstones** (key with null value) delete a key after `delete.retention.ms`.
  - `compact,delete` combines both.
- The **stream-table duality**: a table is the latest value per key of a stream, and a stream is the changelog of a table.

---

## 10. Ecosystem {#ecosystem}

- **Kafka Connect**: a framework for scalable, fault-tolerant **source** (DB/CDC, files, SaaS → Kafka) and **sink** (Kafka → S3, Elasticsearch, JDBC, Snowflake, BigQuery) connectors, configured as JSON with no code. **Single Message Transforms (SMTs)** for light changes. Distributed mode stores configs, offsets, and status in Kafka topics. **Debezium** is the CDC source connector family (see `databases/analytics`).
- **Schema Registry**: Avro/Protobuf/JSON Schema with compatibility checks. Producers register schemas, and messages carry a 5-byte header (magic + schema ID).
- **Kafka Streams**: a Java library (not a cluster) for stream processing. KStream/KTable/GlobalKTable, joins, windowed aggregations, state stores (RocksDB) backed by changelog topics, interactive queries, and EOS. Apps scale by running more instances (partition-based).
- **ksqlDB**: SQL over Kafka Streams (Confluent).
- **Apache Flink**: the leading general stream processor (event time, watermarks, large state, exactly-once sinks). Flink SQL. Often paired with Kafka.
- **MirrorMaker 2** / Confluent Cluster Linking / Replicator: cross-cluster and cross-region replication (DR, aggregation, migration). Offsets differ between clusters, so consumers failing over need offset translation.
- **REST Proxy**, **Cruise Control** (LinkedIn; automated partition rebalancing), **AKHQ / Kafka UI / Conduktor** (UIs), **Strimzi** (Kubernetes operator).

---

## 11. Topic and Partition Design {#design}

- **One topic per event type or per entity stream** (e.g., `orders.v1` containing OrderCreated, OrderPaid, … for ordering per order). Separating every event type into its own topic loses cross-type ordering for the same entity.
- Naming: `<domain>.<entity>.<event|state>.v<n>` (e.g., `payments.charge.events.v1`).
- **Partition key** = the entity whose events must be ordered (order_id, account_id). Check for skew (one tenant = 40% of traffic → hot partition).
- **Partition count**: consider target throughput ÷ per-partition throughput (a partition handles roughly 10+ MB/s, depending), the max consumer parallelism you'll need, and the cost of too many partitions (more open files, longer leader elections historically, more memory; KRaft handles far more). Common: 6–48 per topic, more for very high throughput. Over-provision moderately, because adding partitions later remaps keys.
- RF=3, `min.insync.replicas=2`, `acks=all` for important data.
- Retention by requirement: replay window for consumers (days) vs event store (compacted/infinite + tiered storage).
- Message size: keep it small. Use the claim-check pattern for big blobs.
- Avoid using Kafka as a request/response RPC mechanism (possible, but awkward).

---

## 12. Performance Tuning {#tuning}

Producer: `linger.ms` 5–50, `batch.size` 64–256 KB, `compression.type` zstd/lz4, `acks` per durability need, enough `buffer.memory`, and avoid synchronous `get()`.

Consumer: `fetch.min.bytes` / `fetch.max.wait.ms` (batching), `max.poll.records`, and parallel processing per key. Commit asynchronously with a periodic sync.

Broker: plenty of page cache RAM, fast disks (throughput matters more than IOPS; NVMe or gp3/st1 depending), `num.network.threads`, `num.io.threads`, `num.replica.fetchers`, socket buffers for cross-AZ/region, JBOD or RAID10 trade-offs, a separate disk from the OS, and XFS.

OS: `vm.swappiness=1`, high file descriptor limits, `vm.max_map_count` (lots of segments → mmap'd indexes).

Benchmark: `kafka-producer-perf-test.sh`, `kafka-consumer-perf-test.sh`, OpenMessaging Benchmark.

---

## 13. Operations and Monitoring {#ops}

Key metrics:
| Metric | Why |
|---|---|
| **Under-replicated partitions** (`UnderReplicatedPartitions`) | > 0 means a replica is behind or a broker is down |
| **Offline partitions** | > 0 means unavailable data. Page someone |
| **Under-min-ISR partitions** | Producers with acks=all failing |
| Active controller count | Exactly 1 |
| Request latency (produce/fetch p99), request queue time | Broker saturation |
| Network/request handler idle % | Thread pool saturation |
| Bytes in/out per broker and topic | Load balance |
| **Consumer lag** per group/partition (and lag in time) | Consumers falling behind |
| Disk usage, ISR shrink/expand rate | |
| Rebalance rate per group | Unstable consumers |

Operations:
- **Partition reassignment** when adding brokers (new brokers get no partitions automatically!). Use `kafka-reassign-partitions.sh`, Cruise Control, or managed auto-balancing. Throttle replication during moves.
- Rolling restarts with controlled shutdown (leadership moves first). Preferred leader election afterward.
- Upgrades: rolling, with `inter.broker.protocol.version` / `metadata.version` bumps afterward.
- Security: **TLS** encryption, **SASL** authentication (SCRAM, OAUTHBEARER, mTLS, AWS IAM for MSK), **ACLs** per principal, topic, and group, plus quotas per client to protect against noisy neighbors.
- Disaster recovery: multi-AZ cluster (RF=3 across AZs) for AZ failure, and MirrorMaker 2 or Cluster Linking to another region for region failure (async, RPO > 0).

---

## 14. Alternatives and Managed Services {#alternatives}

| Product | Notes |
|---|---|
| **Confluent Cloud** | Fully managed Kafka by its creators; serverless clusters, connectors, Flink, governance |
| **Amazon MSK** (provisioned / serverless), MSK Connect | AWS-managed Apache Kafka |
| Aiven, Instaclustr, Azure Event Hubs (Kafka API), Upstash | Managed / Kafka-compatible |
| **Redpanda** | Kafka-API-compatible C++ implementation, no JVM/ZooKeeper, thread-per-core, Raft per partition; lower latency, simpler ops |
| **WarpStream** (acquired by Confluent), AutoMQ, Bufstream, Confluent Freight | "Diskless" Kafka on object storage: cheaper (no cross-AZ replication traffic), higher latency |
| **Apache Pulsar** | Brokers (stateless) + BookKeeper (storage), multi-tenancy, geo-replication, tiered storage, queue + stream semantics |
| **AWS Kinesis Data Streams** | Shards (1 MB/s in, 2 MB/s out each), 24 h–365 d retention, simpler, AWS-only |
| **NATS JetStream** | Lightweight, persistence, streams + KV + object store |
| Redis Streams | Small-scale streams with consumer groups in Redis |

---

## 15. Patterns and Anti-Patterns {#patterns}

Patterns:
- **Transactional outbox + CDC** to publish domain events reliably (see the event-driven patterns note).
- **Event-carried state transfer**: consumers build local read models (CQRS).
- **Compacted topics as a distributed cache / configuration source**.
- **Retry topics + DLQ** for non-blocking retries.
- **Fan-out to many consumer groups** (analytics, search indexing, notifications) from one event stream.
- **Claim check** for large payloads.

Anti-patterns:
- Dual writes to DB and Kafka without an outbox.
- Treating Kafka as a database for random-access queries (it's a log; build materialized views elsewhere, e.g., Kafka Streams state stores, a DB, or a cache).
- Thousands of tiny topics, one per customer (use keys/partitions instead), or a single giant partition (no parallelism).
- Auto-commit with slow, side-effecting processing (message loss on crashes).
- Huge messages (> 1 MB) without the claim check.
- Ignoring consumer lag until users complain.
- Using unclean leader election to "fix" availability.

---

## 16. Interview Questions {#qa}

1. Why is Kafka so fast?
2. Explain partitions, offsets, and consumer groups. How many consumers can usefully read a 12-partition topic in one group?
3. What do `acks=all` and `min.insync.replicas` do together?
4. What is the ISR and the high watermark?
5. How does Kafka guarantee ordering, and when can ordering break?
6. Explain Kafka's exactly-once semantics. What doesn't it cover?
7. What triggers a consumer group rebalance, and how do you minimize its impact?
8. What is log compaction and when would you use it?
9. Why did Kafka replace ZooKeeper with KRaft?
10. How would you design topics and keys for an e-commerce order event stream?
11. A consumer group's lag keeps growing. How do you investigate and fix it?
