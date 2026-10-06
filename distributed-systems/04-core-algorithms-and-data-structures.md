# Core Algorithms and Data Structures of Large-Scale Systems

> Code implementations for consistent hashing, rate limiting, and load balancing are in [`system-design/awesome-notes/implementations/`](../system-design/awesome-notes/implementations/). This note explains the theory and the trade-offs behind those and other structures that appear in system design interviews.

## Table of Contents
1. [Consistent Hashing (Ring, Virtual Nodes, Jump, Rendezvous, Maglev)](#consistent)
2. [Bloom Filters and Variants](#bloom)
3. [HyperLogLog (Cardinality Estimation)](#hll)
4. [Count-Min Sketch (Frequency Estimation / Heavy Hitters)](#cms)
5. [Other Sketches: t-digest, Reservoir Sampling, MinHash](#sketches)
6. [Merkle Trees (Anti-Entropy, Sync, Integrity)](#merkle)
7. [Gossip Protocols](#gossip)
8. [Unique ID Generation (Snowflake, UUIDv7, ULID, Tickets)](#ids)
9. [Rate Limiting Algorithms](#ratelimit)
10. [Geospatial Indexing (Geohash, Quadtree, S2, H3)](#geo)
11. [Skip Lists, LSM, Tries, Inverted Indexes (pointers)](#others)
12. [Cheat Sheet: Which Structure for Which Problem](#cheatsheet)
13. [Interview Questions](#qa)

---

## 1. Consistent Hashing {#consistent}

**Problem**: distribute keys across N servers (cache nodes, shards). Naive `hash(key) % N` remaps **almost all keys** when N changes, which causes a mass cache miss and data movement storm.

**Ring hashing** (Karger et al., 1997; used in Dynamo, Cassandra, Riak, Memcached clients like ketama):
- Hash servers onto a ring (0 … 2³²−1). Hash each key onto the ring and walk **clockwise** to the first server.
- Adding or removing a server moves only the keys between it and its predecessor: **~K/N keys move** (minimal disruption).
- **Virtual nodes**: each physical server gets many positions (100–256 tokens) to smooth out uneven arcs and spread a failed node's load across many others. Weight a server by giving it more vnodes.
- Replication: the next R distinct physical nodes clockwise hold replicas.
- Lookup: binary search over sorted positions, O(log V).

**Jump consistent hash** (Google, 2014): `jump(key, num_buckets)`. No memory, very fast, perfectly even, but buckets can only be added or removed **at the end** (good for numbered shards, not arbitrary node removal).

**Rendezvous / Highest Random Weight (HRW) hashing**: for each key, compute `score = hash(key, server)` for every server and pick the highest. Removal only moves keys owned by the removed server. It's O(N) per lookup (fine for small N), simple, and supports weights. Used in some CDNs and load balancers.

**Maglev hashing** (Google's L4 LB): builds a lookup table (permutation-based) with near-perfect balance and minimal disruption, O(1) lookup. Used in Envoy's Maglev LB policy.

**Consistent hashing with bounded loads** (Google/Vimeo, 2017): caps each server at (1+ε) × average load, and overflow keys move to the next server. This handles hot keys and skew.

Hot keys remain a problem with any scheme: one celebrity key → one node. Mitigations: replicate hot keys to multiple nodes and read from any, use a local in-process cache, or add key salting/sharding.

---

## 2. Bloom Filters {#bloom}

A probabilistic set answering "**possibly in set**" or "**definitely not in set**".
- A bit array of m bits plus k hash functions. **Add**: set bits at the k hash positions. **Query**: if any of the k bits is 0, the item is definitely absent. If all are 1, it's probably present.
- **No false negatives, tunable false positives.**
- Optimal k = (m/n) ln 2. False positive rate ≈ (1 − e^(−kn/m))^k.
- Rule of thumb: **~10 bits per element gives ~1% FPR** (k ≈ 7). 1 billion items at 1% ≈ 1.2 GB.
- Can't delete (clearing bits may affect other items) → **counting Bloom filters** (counters instead of bits; 4× memory) or **cuckoo filters** (support deletion, often better space at low FPR). **Quotient filters** and **XOR/binary fuse filters** (static sets, even smaller).

Uses:
- **LSM trees**: skip SSTables that don't contain a key (RocksDB, Cassandra, HBase). This is the classic use.
- **Cache penetration protection**: reject lookups for keys that don't exist before hitting the DB.
- Avoid recommending already-seen articles or videos (Medium, feed systems).
- Web crawlers: "already visited this URL?" (approximately).
- Malicious URL checks (Chrome Safe Browsing historically), username-taken pre-check.
- Distributed joins (send a Bloom filter of keys to reduce shuffled data).
- Redis has a `BF.*` module (RedisBloom), and Postgres has a `bloom` index.

---

## 3. HyperLogLog {#hll}

Estimates the **number of distinct elements** (cardinality) using tiny fixed memory.
- Idea: hash each item. The probability of seeing a hash with ≥ k leading zeros is 2^−k, so the max leading-zero run observed tells you roughly log₂(cardinality). HLL uses many registers (buckets, by the first bits of the hash) and a **harmonic mean** to reduce variance.
- Error ≈ 1.04 / √m (m registers). Redis HLL uses 16,384 registers in **12 KB** for **0.81% standard error**, regardless of whether there are 1 thousand or 1 billion distinct items.
- **Mergeable**: union of HLLs = register-wise max, so you can compute daily uniques and merge them for weekly uniques without double counting.
- Can't list elements, and can't do exact intersections (estimate via inclusion-exclusion, poorly).

Uses: unique visitors per page per day (`PFADD page:home:2025-06-01 user123`, `PFCOUNT`), distinct search queries, `COUNT(DISTINCT)` approximations in BigQuery (`APPROX_COUNT_DISTINCT`), Druid, ClickHouse `uniq`, Presto `approx_distinct`, and PG extensions (postgresql-hll).

---

## 4. Count-Min Sketch {#cms}

Estimates **frequencies** of items in a stream.
- d rows × w counters. Each row has its own hash. **Add**: increment one counter per row. **Query**: take the **minimum** across rows (collisions only inflate counts, never deflate them).
- Overestimates by at most ε·N with probability 1−δ, where w = ⌈e/ε⌉ and d = ⌈ln(1/δ)⌉.
- **Heavy hitters / top-K**: combine with a min-heap of candidates. Find trending hashtags, top URLs, top API clients, abusive IPs.
- Conservative update and count-mean-min variants reduce error.
- Redis `CMS.*` and `TOPK.*` (RedisBloom, HeavyKeeper algorithm for top-K).

Uses: "Top 10 trending topics in the last hour", rate-limiting high-cardinality keys approximately, network flow monitoring, and query frequency for caching decisions.

---

## 5. Other Sketches and Sampling {#sketches}

- **t-digest / DDSketch / HDR Histogram / KLL**: approximate **quantiles** (p50, p99, p999) in streams with bounded memory and mergeability. This is how monitoring systems compute latency percentiles across many servers (merge sketches; you can't average p99s!). Prometheus native histograms and DataDog's DDSketch (relative-error guarantees) use this.
- **Reservoir sampling**: keep a uniform random sample of k items from a stream of unknown length (replace with probability k/i). Used for log or trace sampling and A/B analysis.
- **MinHash + LSH (Locality-Sensitive Hashing)**: estimate Jaccard similarity between sets, and find near-duplicates (duplicate web pages, plagiarism, similar items) at scale without pairwise comparisons.
- **SimHash**: near-duplicate detection for documents (Google's crawler).
- **Theta sketches** (Apache DataSketches): set operations (union/intersection/difference) on cardinality estimates.

---

## 6. Merkle Trees {#merkle}

A tree of hashes: leaves = hashes of data blocks, each parent = hash of its children's hashes. The root hash summarizes the entire dataset.
- **Efficient comparison**: two replicas compare root hashes. If they're equal, they're identical. If not, recurse into the children whose hashes differ. This finds the differing ranges in **O(log n)** comparisons and transfers only the diffs.
- **Integrity proofs**: prove a block belongs to the dataset with O(log n) sibling hashes (a Merkle proof).

Uses:
- **Anti-entropy repair**: Cassandra (`nodetool repair` builds Merkle trees per token range), Dynamo, Riak.
- **Git** (commits/trees/blobs form a Merkle DAG), **IPFS**, **Docker/OCI image layers** (content addressing).
- **Blockchains** (transactions in blocks; Bitcoin, Ethereum's Merkle-Patricia tries).
- **Certificate Transparency logs** (append-only Merkle logs with consistency proofs).
- **ZFS/Btrfs** data integrity, **rsync**-like sync and backup deduplication, and DynamoDB/Cosmos replication checks.

---

## 7. Gossip Protocols {#gossip}

Epidemic dissemination: periodically, each node picks a random peer (or a few) and exchanges state. Information spreads to all N nodes in **O(log N)** rounds with high probability, like an epidemic.
- Robust (no single point of failure), scalable (constant load per node), eventually consistent.
- Variants: push, pull, push-pull. **Anti-entropy** (full state comparison, often with Merkle trees) vs **rumor mongering** (spread new updates until they're "old").
- Uses: membership and failure detection (SWIM/memberlist in Consul and Serf, Cassandra gossip, Redis Cluster bus), propagating cluster metadata and load info, CRDT state sync, and Bitcoin transaction propagation.
- Trade-offs: propagation delay, bandwidth overhead with large states (use digests/deltas), and convergence that's probabilistic, not instant.

---

## 8. Unique ID Generation {#ids}

Requirements to clarify: globally unique? sortable by time? size (64-bit vs 128-bit)? generated without coordination? unguessable? throughput?

| Approach | Size | Sortable | Coordination | Notes |
|---|---|---|---|---|
| DB auto-increment | 64-bit | Yes | Single DB (bottleneck/SPOF) | Simple; leaks volume; doesn't scale across shards |
| **Multi-master auto-increment with offset** | 64-bit | Roughly | Static config | Server k generates k, k+N, k+2N… (Flickr ticket servers; MySQL `auto_increment_increment/offset`). Hard to add servers |
| **Ticket server / range allocation** | 64-bit | Roughly | Central, but batched | Each app instance grabs a block of 1,000 IDs from a central counter (Flickr, Hi/Lo algorithm). Gaps on restart |
| **UUIDv4** | 128-bit | No | None | 122 random bits; collision probability negligible; bad B-tree locality |
| **UUIDv7** (RFC 9562, 2024) | 128-bit | **Yes** (ms timestamp prefix) | None | Best modern default for 128-bit IDs |
| **ULID** | 128-bit | Yes | None | Crockford base32 26-char string; similar to UUIDv7 |
| **Snowflake** (Twitter) | **64-bit** | Yes (roughly, per ms) | Worker ID assignment only | `1 bit sign | 41 bits ms since custom epoch (~69 years) | 10 bits machine (1024 workers) | 12 bits sequence (4096/ms/worker)` → ~4M IDs/s per worker |
| Variants | 64-bit | Yes | | Instagram (PG function: 41-bit time, 13-bit shard ID, 10-bit sequence), Sonyflake (10 ms units, longer lifetime), Discord/Mastodon snowflakes, Baidu UidGenerator, Leaf (Meituan) |
| **KSUID** | 160-bit | Yes (seconds) | None | Segment.io |
| **NanoID** | Configurable | No | None | Short URL-safe random IDs |

Snowflake concerns:
- **Clock moving backwards** (NTP): refuse to generate (wait) or use the last timestamp + sequence, otherwise you get duplicates.
- **Worker ID assignment**: static config, ZooKeeper/etcd sequential nodes, a DB, or derived from pod ordinal (StatefulSet). Duplicate worker IDs mean duplicate IDs.
- Sequence exhaustion within one ms: wait for the next ms.
- IDs leak creation time and approximate volume.
- JavaScript numbers are only safe to 2⁵³, so **serialize 64-bit IDs as strings in JSON APIs** (Twitter's `id_str`).

Short public IDs (URL shorteners): base62-encode a numeric ID (62⁷ ≈ 3.5 trillion), or random base62 with a collision check. Don't expose sequential IDs if enumeration is a concern.

---

## 9. Rate Limiting Algorithms {#ratelimit}

Implementations: [`system-design/awesome-notes/implementations/*/rate_limiting/`](../system-design/awesome-notes/implementations/).

| Algorithm | How | Pros | Cons |
|---|---|---|---|
| **Fixed window counter** | Count requests per key per window (`key:minute`), reject over limit | Simple, 1 counter | **Boundary burst**: 2× limit across a window edge (100 at 0:59 + 100 at 1:00) |
| **Sliding window log** | Store a timestamp per request (sorted set), count those within the last window | Exact | Memory O(requests) per key |
| **Sliding window counter** | Weighted: current window count + previous window count × overlap fraction | Approximate, O(1) memory, smooths the boundary | Approximation (assumes uniform distribution in previous window) |
| **Token bucket** | Bucket holds up to B tokens, refilled at rate r/s; each request takes 1 token | **Allows bursts up to B**, then a steady rate; O(1) (store tokens + last refill time) | Two parameters to tune |
| **Leaky bucket** | Requests enter a queue that drains at a constant rate (or a meter that rejects when full) | Smooths output to a constant rate (traffic shaping) | Bursty traffic gets delayed or dropped |
| **GCRA** (Generic Cell Rate Algorithm) | Tracks a "theoretical arrival time"; equivalent to a leaky bucket as a meter | O(1), a single timestamp per key | Less intuitive (used by redis-cell, Stripe-like limiters) |
| **Concurrency limiter** | Cap in-flight requests, not rate | Protects slow resources | |
| **Adaptive (AIMD / gradient)** | Adjust limits based on observed latency or errors (Netflix concurrency-limits) | Self-tuning load shedding | Complexity |

Distributed rate limiting:
- Central store (Redis) with **atomic Lua scripts** or `INCR` + `EXPIRE`. That's an extra network hop per request, and Redis becomes critical (decide whether to **fail open or fail closed** when it's down).
- Local limiters per instance (limit/N each) are approximate but have no hop. Or a hybrid: local token buckets periodically synced with the global count (Envoy global rate limit service + local rate limit filter).
- Keys: user ID, API key, IP (beware NAT and IPv6 /64), tenant, endpoint, and combinations of them. Use multiple tiers (per second burst + per day quota).
- Return `429` + `Retry-After` + rate limit headers (see the API design note).
- Stripe's layered approach: request rate limiter, concurrent request limiter, fleet usage load shedder, and worker utilization load shedder.

---

## 10. Geospatial Indexing {#geo}

Problem: "find drivers within 2 km", "restaurants near me". 2D proximity can't use a 1D B-tree directly.

| Technique | Idea | Notes |
|---|---|---|
| **Geohash** | Interleave latitude/longitude bits, then base32-encode. A shared prefix means nearby (mostly) | Simple; prefix queries in any KV/B-tree. **Edge problem**: close points across a cell boundary have different prefixes, so query the cell + 8 neighbors. Precision 6 ≈ 1.2 km × 0.6 km cells. Redis GEO uses 52-bit geohash in sorted sets |
| **Quadtree** | Recursively split 2D space into 4 quadrants until each cell has ≤ k points | Adapts to density (dense cities get small cells). In-memory; rebuilding/rebalancing is a cost |
| **R-tree** | Hierarchy of bounding rectangles | Good for shapes (polygons); used by PostGIS (via GiST), MySQL SPATIAL, SQLite R*Tree |
| **Google S2** | Projects the sphere onto a cube, then Hilbert curve cell IDs (64-bit) with hierarchical levels | Excellent locality, covering arbitrary regions with cell unions. Used by Google Maps, Foursquare, MongoDB 2dsphere, CockroachDB |
| **Uber H3** | Hexagonal hierarchical grid (16 resolutions) | Hexagons have uniform neighbor distances (6 equidistant neighbors), great for aggregation (surge pricing zones, demand heatmaps), k-ring queries |
| **k-d tree** | Binary space partition alternating dimensions | In-memory nearest neighbor (lower dimensions) |

Proximity service design (Yelp/Uber): static places use geohash/quadtree in a DB plus a cache. Moving drivers update locations every few seconds into an in-memory index (Redis GEO, or H3 cell → set of drivers) sharded by region. To query, search the cells covering the radius, then filter by exact distance (haversine).

---

## 11. Other Structures (pointers) {#others}

- **Skip lists**: probabilistic balanced ordered structure with O(log n) operations and simpler concurrency than balanced trees. Used by Redis sorted sets (skip list + hash) and LSM memtables (LevelDB/RocksDB).
- **LSM trees and B+trees**: see [`databases/fundamentals/04-storage-engine-internals.md`](../databases/fundamentals/04-storage-engine-internals.md).
- **Tries / radix trees / FSTs**: prefix lookups (autocomplete, IP routing tables, Lucene's term dictionary as an FST, Redis Streams/rax).
- **Inverted indexes**: search ([`databases/nosql/elasticsearch.md`](../databases/nosql/elasticsearch.md)).
- **Ring buffers**: lock-free producer/consumer queues (LMAX Disruptor), log buffers.
- **Timing wheels**: efficient large numbers of timers (Kafka purgatory, Netty HashedWheelTimer, Linux kernel timers).
- **HNSW graphs**: approximate nearest-neighbor vector search.
- **Segment trees / Fenwick trees**: range aggregates (leaderboard rank queries in some designs).
- **LRU/LFU/ARC/TinyLFU**: cache eviction (Caffeine uses W-TinyLFU, which combines a count-min sketch for frequency with a small LRU window).

---

## 12. Cheat Sheet {#cheatsheet}

| Problem | Structure |
|---|---|
| Distribute keys over changing set of nodes | Consistent hashing (vnodes), rendezvous, jump hash |
| "Have I seen this before?" with tiny memory | Bloom / cuckoo filter |
| Count unique users | HyperLogLog |
| Top-K / heavy hitters / trending | Count-Min Sketch + heap, HeavyKeeper |
| Latency percentiles across fleet | t-digest / DDSketch / HDR histograms |
| Find differences between replicas | Merkle trees |
| Spread membership/state across a big cluster | Gossip (SWIM) |
| Sortable unique IDs without coordination | Snowflake (64-bit), UUIDv7/ULID (128-bit) |
| Throttle clients | Token bucket / sliding window counter / GCRA in Redis |
| Nearby search | Geohash, quadtree, S2, H3, R-tree (PostGIS) |
| Leaderboard | Redis sorted set (skip list) |
| Autocomplete | Trie / FST / edge n-grams |
| Near-duplicate detection | MinHash + LSH, SimHash |
| Many timers | Timing wheel |
| Cache eviction | LRU / W-TinyLFU |

---

## 13. Interview Questions {#qa}

1. Why is `hash(key) % N` bad for distributing cache keys? How does consistent hashing fix it? What do virtual nodes add?
2. Explain how a Bloom filter works. Can it have false negatives? How much memory for 100M items at 1% FPR?
3. How does HyperLogLog estimate cardinality in 12 KB? Why is mergeability important?
4. Design trending hashtags for the last hour.
5. How do Merkle trees speed up replica synchronization?
6. Design a unique ID generator producing 64-bit sortable IDs at 1M/s across data centers.
7. Compare token bucket, leaky bucket, and sliding window rate limiters. Implement one in Redis.
8. Design "find nearby drivers". Geohash vs quadtree vs H3?
9. Why can't you average p99 latencies across servers? What do you do instead?
