# Apache Cassandra (and ScyllaDB) Deep Dive

## Table of Contents
1. [What Cassandra Is For](#what)
2. [Architecture: Ring, Tokens, Gossip, Snitches](#arch)
3. [Data Model: Keyspace, Table, Partition Key, Clustering Columns](#model)
4. [Query-First Modeling with Examples](#modeling)
5. [Write Path and Read Path](#paths)
6. [Consistency Levels and Quorums](#consistency)
7. [Repair, Hinted Handoff, Read Repair](#repair)
8. [Tombstones and Deletes](#tombstones)
9. [Compaction Strategies](#compaction)
10. [Lightweight Transactions, Counters, Batches, Secondary Indexes, MVs](#features)
11. [Multi-Datacenter Deployments](#multidc)
12. [Operations and Tuning](#ops)
13. [ScyllaDB and Alternatives](#scylla)
14. [Interview Questions](#qa)

---

## 1. What Cassandra Is For {#what}

Created at Facebook (2008) for inbox search, combining **Dynamo's distribution** (leaderless, consistent hashing, tunable consistency) with **Bigtable's data model** (LSM storage, column families). Apache top-level project.

Sweet spot:
- **Massive write throughput** that scales linearly by adding nodes.
- **Always-on availability**: no single point of failure, multi-datacenter active-active.
- Access patterns known upfront, mostly **by partition key**, with time-ordered data inside the partition.
- Examples: messaging (Discord, before moving to ScyllaDB), activity feeds, IoT/telemetry, event logs, user timelines, fraud signals, Netflix viewing history, Apple (one of the largest deployments).

Bad fit: ad-hoc queries, joins, aggregations across partitions, strong multi-row transactions, small datasets (operational overhead), queue-like delete-heavy workloads.

---

## 2. Architecture {#arch}

- **Peer-to-peer**: every node is equal. There's no primary. Any node can coordinate any request.
- **Token ring**: the partitioner (Murmur3) hashes the partition key to a 64-bit token. Each node owns token ranges. **Virtual nodes (vnodes)** give each node many small ranges (`num_tokens`, 16 recommended in 4.0+), which spreads load and eases rebalancing.
- **Replication factor (RF)** per keyspace per datacenter. `NetworkTopologyStrategy` places replicas on distinct racks/AZs.
- **Gossip**: nodes exchange state every second (liveness, schema, load). The **phi accrual failure detector** decides when a node is down.
- **Snitch**: knows the topology (DC/rack), e.g., `GossipingPropertyFileSnitch`, `Ec2Snitch`. It's used for replica placement and routing reads to the closest replicas (dynamic snitch scores latency).
- **Coordinator**: the node receiving a request forwards it to the replicas and gathers responses. Token-aware drivers send requests directly to a replica.

---

## 3. Data Model {#model}

```sql
CREATE KEYSPACE chat WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3, 'dc2': 3};

CREATE TABLE chat.messages_by_channel (
  channel_id  uuid,
  bucket      text,          -- e.g. '2025-06' to bound partition size
  message_id  timeuuid,
  author_id   uuid,
  body        text,
  PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy', 'compaction_window_unit': 'DAYS', 'compaction_window_size': 1};
```
- **Partition key** `(channel_id, bucket)`: decides which nodes store the data. All rows with the same partition key live together, sorted.
- **Clustering columns** `message_id`: sort order within the partition. Range queries are allowed on them (in order).
- **Primary key** = partition key + clustering columns, unique per row.
- **Static columns**: one value shared by all rows in a partition.
- Collections: `list`, `set`, `map` (keep them small), frozen UDTs, tuples.
- **Upserts by default**: `INSERT` and `UPDATE` are both upserts. There's no read-before-write. Writing the same primary key overwrites (last-write-wins by **write timestamp**, per cell).
- **TTL** per cell: `INSERT … USING TTL 86400`.

Query rules (CQL is SQL-like but restricted):
- Must specify the full partition key (otherwise `ALLOW FILTERING`, which is a full cluster scan; never use it in production).
- Clustering columns are restricted left to right; a range applies only on the last restricted one.
- No joins, no subqueries, and GROUP BY only on primary key columns within partitions.

---

## 4. Query-First Modeling {#modeling}

Example: a video platform.
| Query | Table | Primary key |
|---|---|---|
| Q1: Get video by id | `videos` | `((video_id))` |
| Q2: Latest videos uploaded by a user | `videos_by_user` | `((user_id), uploaded_at, video_id)` DESC |
| Q3: Latest videos with a tag | `videos_by_tag` | `((tag, bucket), uploaded_at, video_id)` |
| Q4: Comments on a video, newest first | `comments_by_video` | `((video_id), comment_id timeuuid)` DESC |
| Q5: Comments by a user | `comments_by_user` | `((user_id), comment_id)` DESC |

The same comment is written to two tables (Q4, Q5), so writes are duplicated, and the app (or a logged batch) keeps them in sync. **Storage is cheap; cross-partition queries are not.**

Partition sizing guidelines: keep partitions under ~100 MB (ideally tens of MB) and under ~100k rows. Unbounded partitions (all messages of a channel forever) cause hot spots, slow reads, compaction pain, and repair issues. **Bucket by time** (`channel_id + month`), and have the app walk buckets backward for pagination.

Hot partitions: a celebrity's timeline or a global counter. Shard the key (`(user_id, shard_no)` with shard_no 0..N) and merge on read.

---

## 5. Write and Read Paths {#paths}

**Write** (very fast, no read-before-write):
1. Coordinator sends the mutation to all RF replicas (in all DCs).
2. Each replica appends to the **commit log** (sequential) and writes to the **memtable**.
3. The replica acks. The coordinator responds once **CL** replicas have acked.
4. A memtable fills and flushes to an immutable **SSTable**. The commit log segment is then recyclable.
5. Background compaction merges SSTables.

**Read**:
1. Coordinator contacts CL replicas (the closest by snitch). It asks one for the full data and the others for digests.
2. Each replica checks the memtable, row cache (rarely enabled), then for each SSTable: **bloom filter** → partition key cache → partition summary/index → compression offsets → data. With 4.0+ BTI format (5.0 trie-indexed SSTables), lookups get faster.
3. Merge cells from all sources by timestamp (latest wins), and apply tombstones.
4. If digests mismatch → read repair (fetch full data, reconcile, write back).

Writes are cheaper than reads. Reads get slower as data is spread across more SSTables (compaction health matters).

---

## 6. Consistency Levels {#consistency}

Per-request CL. With RF=3:
| CL | Replicas that must respond |
|---|---|
| ANY (writes only) | Any node, even a hint. Weakest |
| ONE / TWO / THREE | That many |
| **QUORUM** | ⌊RF_total/2⌋+1 across all DCs (RF 3+3 → 4) |
| **LOCAL_QUORUM** | Quorum in the coordinator's DC (2 of 3). **The usual choice for multi-DC** |
| EACH_QUORUM (writes) | Quorum in every DC |
| LOCAL_ONE | One in the local DC |
| ALL | All replicas. Any node down → failure |
| SERIAL / LOCAL_SERIAL | For lightweight transactions (Paxos) |

**Strong consistency recipe**: `W + R > RF`, e.g., LOCAL_QUORUM writes + LOCAL_QUORUM reads (2 + 2 > 3). That gives read-your-writes within a DC. Caveat: it's still not linearizable under concurrent writes to the same cell (last-write-wins by client/coordinator timestamps, so **clock skew** can make an older write win). Use NTP or chrony, or LWT for compare-and-set semantics.

Availability: with RF=3 and LOCAL_QUORUM, the cluster tolerates one replica down per token range per DC.

---

## 7. Repair, Hinted Handoff, Read Repair {#repair}

Replicas diverge (a down node misses writes, dropped mutations under overload). Mechanisms that converge them:
1. **Hinted handoff**: if a replica is down, the coordinator stores a hint and replays it when the node returns (for up to `max_hint_window`, default 3 h). Beyond that, the node needs repair.
2. **Read repair**: on digest mismatch during reads at CL > ONE, the data gets fixed. (Probabilistic background read repair was removed in 4.0; blocking read repair remains.)
3. **Anti-entropy repair** (`nodetool repair`): builds **Merkle trees** per token range, compares them between replicas, and streams the differences. **Must run regularly, more often than `gc_grace_seconds`** (default 10 days), or deleted data can resurrect (see tombstones). Use incremental repair (4.0 fixed it), **Cassandra Reaper** to schedule subrange repairs, or the upcoming built-in auto-repair (CEP-37).

---

## 8. Tombstones and Deletes {#tombstones}

A delete writes a **tombstone** (a marker with a timestamp). It can't remove the data immediately, because other replicas may still have the old value. If the tombstone were dropped early, the old value would come back via repair ("zombie data").

- Tombstones are purged during compaction only after `gc_grace_seconds` (10 days default), and only when the compaction includes all SSTables holding the shadowed data.
- TTL expiry also creates tombstones.
- Kinds: cell, row, range (clustering range), partition tombstones.

**Tombstone problems**:
- Reads scan tombstones before finding live data. Warnings at `tombstone_warn_threshold` (1,000) and failures at `tombstone_failure_threshold` (100,000) → `TombstoneOverwhelmingException`.
- **Queue anti-pattern**: insert jobs, delete when done, read "the oldest job". Every read scans all the tombstones of processed jobs. Use a real queue, or time-bucketed partitions you stop reading.
- Inserting `null` values creates tombstones! Prepared statements binding null for unset columns: use "unset" (protocol v4+) instead.
- Collections overwritten with a new full value write a tombstone first.

---

## 9. Compaction Strategies {#compaction}

| Strategy | When | Notes |
|---|---|---|
| **STCS** (Size-Tiered, default before 5.0) | Write-heavy, few updates | Needs ~50% free disk headroom for big compactions; reads may touch many SSTables |
| **LCS** (Leveled) | Read-heavy, update-heavy | ~90% of reads hit one SSTable; high write amplification and I/O |
| **TWCS** (Time-Window) | Time-series with TTL, append-only | Whole windows expire and drop cheaply; avoid out-of-order writes and explicit deletes |
| **UCS** (Unified, Cassandra 5.0) | General: configurable between tiered and leveled behavior | Added in 5.0 (default in the `cassandra_latest.yaml` profile); sharded, parallel compactions |

Monitor pending compactions (`nodetool compactionstats`), SSTables per read (`nodetool tablehistograms`), and disk headroom.

---

## 10. LWT, Counters, Batches, Indexes, MVs {#features}

**Lightweight transactions (LWT)**: compare-and-set with Paxos (4 round trips in classic Paxos; Paxos v2 in 4.1 is faster).
```sql
INSERT INTO users (username, email) VALUES ('aadarsh', 'a@x.com') IF NOT EXISTS;
UPDATE accounts SET balance = 70 WHERE id = 1 IF balance = 100;
```
Use them sparingly (uniqueness of usernames, leader claims). They're much slower and can contend. Don't mix LWT and non-LWT writes on the same data.

**Counters**: a special `counter` column type with `UPDATE … SET c = c + 1`. They're not idempotent (a retry after a timeout can double count), so they suit approximate metrics.

**Batches**:
- **Logged batch**: atomic (eventually all or nothing) across partitions via the batchlog. Use it to keep denormalized tables in sync. It's not isolated, and it's not a performance feature.
- **Unlogged batch**: only beneficial when all statements target the **same partition**. Multi-partition unlogged batches overload the coordinator, so it's an anti-pattern for bulk loading. Use async parallel writes instead.

**Secondary indexes**: classic 2i are local per node, so a query hits all nodes, which only suits low-cardinality filters within a partition. **SASI** is deprecated. **SAI (Storage-Attached Indexes, Cassandra 5.0)** is much better: numeric ranges, multiple indexes per query, vector search (ANN). Still, query-first tables are preferred for hot paths.

**Materialized views**: server-maintained denormalized tables. They're **experimental and disabled by default** (consistency bugs). Prefer application-maintained tables.

---

## 11. Multi-DC {#multidc}

- RF per DC (`{'us_east': 3, 'eu_west': 3}`); writes go to all DCs asynchronously from the client's view at `LOCAL_*` CLs.
- LOCAL_QUORUM reads/writes avoid cross-region latency, so the cluster stays available if a whole DC fails.
- Use cases: global low-latency, DR, separating analytics (a Spark DC) from OLTP DCs.
- Conflicts resolve by last-write-wins timestamps. Design for idempotent and commutative writes where possible.

---

## 12. Operations and Tuning {#ops}

- `nodetool status | ring | info | tpstats | tablestats | tablehistograms | proxyhistograms | compactionstats | repair | cleanup | drain | decommission | removenode | snapshot | flush | gcstats`.
- Scaling: add nodes (bootstrap streams data), then run `nodetool cleanup` on old nodes to remove ranges they no longer own.
- **JVM**: G1GC (or ZGC with Java 17 in 5.0). Avoid huge heaps (8–31 GB typical). GC pauses cause timeouts.
- Disk: SSD/NVMe, JBOD acceptable, keep 30–50% free for compaction.
- Backups: `nodetool snapshot` (hard links), incremental backups, **Medusa** (Cassandra backup tool), managed services.
- Drivers: token-aware, DC-aware load balancing; prepared statements; idempotent queries marked for speculative execution and retries; paging (fetch size) for large partitions.
- Monitoring: read/write latency p99 per table, pending compactions, dropped mutations (overload!), hints, GC, disk usage, tombstones per read, partition size outliers (`nodetool tablehistograms` max partition size).
- Managed: DataStax Astra, Amazon Keyspaces (Cassandra-compatible API, different engine), Azure Managed Instance for Cassandra, Instaclustr.

---

## 13. ScyllaDB and Alternatives {#scylla}

- **ScyllaDB**: a C++ rewrite compatible with CQL and the Cassandra drivers. Built on the Seastar framework with a **shard-per-core** architecture (no locks, no JVM GC). It often achieves several times the throughput per node and better tail latency, and is moving to tablets (Raft-based dynamic sharding) instead of vnodes. **Discord** migrated trillions of messages from Cassandra to ScyllaDB (2022–2023), citing GC pauses, hot partitions, and operational toil.
- **Amazon Keyspaces**: serverless Cassandra API.
- **HBase/Bigtable**: wide-column with strong consistency per row (region-server leader), range partitioning, Hadoop ecosystem (HBase) / Google-managed (Bigtable).
- **DynamoDB**: a managed alternative for similar key-based access patterns (see dynamodb.md).

---

## 14. Interview Questions {#qa}

1. Explain partition key vs clustering columns with an example table design.
2. How would you model chat messages so partitions don't grow unbounded?
3. What does LOCAL_QUORUM mean? How do you get read-your-writes?
4. Why are tombstones a problem, and why is Cassandra a bad queue?
5. Why must repair run within `gc_grace_seconds`?
6. Compare STCS, LCS, and TWCS.
7. What are lightweight transactions and why should they be rare?
8. Why are multi-partition unlogged batches an anti-pattern?
9. Why did Discord move from Cassandra to ScyllaDB?
