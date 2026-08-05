# Database Sharding

Sharding is horizontal partitioning of data across *multiple database servers* — each shard holds a subset of rows but shares the same schema. It's how systems like Google, Facebook, and Netflix handle data volumes that no single machine could hold or query fast enough.

## TL;DR
- **Sharding** = splitting rows across multiple separate database instances/servers (horizontal, distributed).
- Different from **partitioning** (splitting within a single DB instance/server) and **replication** (copying the *entire* dataset to other servers for redundancy/read scaling).
- A **shard key** decides which shard a row lands on; a bad key choice creates **hotspots** (one shard overloaded, others idle).
- Common strategies: range, hash, modulo, tag/directory, geo, composite, consistent hashing.
- Enables horizontal scaling (add cheap servers) instead of vertical scaling (buy a bigger single server), which has a hard ceiling.
- Big costs: cross-shard joins/transactions are hard, rebalancing is operationally painful, and it adds real complexity — don't shard prematurely.
- Often combined with replication: each shard gets its own replica set for high availability.

## What is database sharding

**Analogy**: a single library with millions of books means every search means fighting for space on the same shelves. Sharding splits the library into smaller branches spread across town, each with a subset of books; you route each reader to the right branch so no single room is overloaded.

Formally: a **shard** is a horizontal partition of data in a database, stored on a separate server instance. Each shard is the "single source of truth" for its slice of the data, while small shared/lookup tables may be replicated across all shards.

Key components:
- **Logical shard**: the data chunk itself (e.g. rows 1-1000).
- **Physical shard**: the server/node storing it (one machine can host multiple logical shards).
- **Shard key**: the column that decides which shard a row goes to (e.g. `user_id`).
- **Sharding algorithm**: the rule that maps a key to a shard (e.g. hash the key, mod by shard count).

**Shared-nothing architecture**: shards don't share CPU/memory/disk with each other — they're fully independent, which is what makes them scale linearly and fail independently.

| Aspect | Unsharded database | Sharded database |
|---|---|---|
| Storage | All data on one server | Data split across many servers |
| Scaling | Vertical (bigger server) | Horizontal (add servers) |
| Query speed | Slows as data grows (scans everything) | Faster (touches only the relevant shard) |
| Fault tolerance | Single point of failure | One shard down ≠ total outage |
| Complexity | Simple | High (routing, rebalancing) |

**Horizontal vs. vertical sharding**:
- **Horizontal** (the common kind): splits *rows* — e.g. users A-M on Shard 1, N-Z on Shard 2.
- **Vertical**: splits *columns* — e.g. user profiles on Shard 1, transaction history on Shard 2. Less common since it fragments data that belongs together.

## Etymology

"Shard" isn't originally database jargon. Two likely roots:
1. **Computer Corporation of America's 1970s "Highly Available Replicated Data"** system used redundant hardware, fragmenting data like shattered glass — "shards."
2. **Ultima Online** (1997 MMORPG): creator Richard Garriott coined "shards" for parallel game worlds/servers, introduced to fix in-game ecology being over-hunted across a single shared world. The term stuck and later got adopted for parallel/fragmented distributed instances generally.

By the 2010s sharding was standard in NoSQL (MongoDB 1.6, 2010) and later in blockchains (Ethereum's shard chains). Today it's core to NewSQL systems like CockroachDB, TiDB, and Vitess.

## Why shard

Traditional single-server RDBMS (Postgres, MySQL) chokes once you're past a certain scale — one server has finite CPU, RAM, disk I/O, and disk capacity.

**Key drivers**:
- **Performance bottlenecks**: a single server maxes out CPU/RAM/disk I/O.
- **Storage limits**: one disk caps total rows; sharding removes that ceiling by adding nodes.
- **Query latency**: scanning 1TB takes seconds on one machine; split across shards, each one scans a fraction in parallel.
- **High availability**: one shard failing is a partial outage, not a total one.
- **Cost efficiency**: scaling out with commodity/cloud instances is usually cheaper than scaling up a single powerful machine.

| Scaling type | Pros | Cons | Use case |
|---|---|---|---|
| **Vertical** | Simple, no app changes | Hardware limits, single failure point | Small apps (<1TB) |
| **Horizontal (sharding)** | Unlimited scale, fault-tolerant | Complex routing, rebalancing | High-traffic systems (e.g. e-commerce) |

**When to shard**: data > 1TB, queries > 10K/sec, persistent hotspots, or growth projections that will hit limits within 6-12 months.

**When NOT to shard**: DB under ~100GB, simple query patterns, no dedicated ops capacity — try caching, indexing, or read replicas first. Premature sharding adds real complexity for no real benefit; it should generally be the *last* scaling lever pulled, after indexing and caching.

## Sharding vs. partitioning vs. replication

These three get conflated constantly — they solve different problems:

| Concept | Scope | Goal | Example | Key difference |
|---|---|---|---|---|
| **Partitioning** | Single DB instance | Manage large tables, prune scans | Postgres `PARTITION BY RANGE (date)` | No distribution — stays on one machine |
| **Sharding** | Multiple DB instances | Horizontal scale, load balance | MongoDB: user IDs hashed to shards | Distributed across servers; adds routing overhead |
| **Replication** | Multiple full copies | High availability, read scaling | MySQL master-slave sync | Duplicates *all* data; doesn't split anything |

**Analogies**: partitioning = organizing one bookshelf into sections (A-M, N-Z). Sharding = multiple libraries, each with its own section. Replication = photocopying the entire library for backups.

**Hybrid**: shard *and* replicate — each shard gets its own replica set. This is the standard production pattern, pushing availability toward 99.99%+.

## Sharding architecture

Shards build on horizontal partitioning but distribute across separate instances in a shared-nothing setup. Using MongoDB's architecture as a concrete example:

- **Shards**: the actual data holders (usually replica sets for HA, not single nodes).
- **Config servers**: store cluster metadata — the shard map, which keys live where.
- **Query router** (`mongos`): the client-facing component that routes each query to the right shard(s).
- **Balancer**: background process that migrates chunks of data to keep shards evenly loaded.

```
Client Query --> Router (mongos)
                |
                |--> Config Server (Metadata: "User 123 -> Shard 2")
                |
                |--> Shard 1 (Users A-M) --> Replica (HA)
                |--> Shard 2 (Users N-Z) --> Replica
                |--> Shard 3 (Global lookups, replicated)
```

**Architecture variants**:
1. **Horizontal** (default): row splits. Even load if the key is well chosen; cross-shard joins are painful.
2. **Vertical**: column splits. Good for very wide tables; fragments related entities (e.g. splitting a user from their orders).
3. **Geo-sharding**: split by location (e.g. EU data kept in Frankfurt). Low latency and regulatory compliance (GDPR); can be uneven if user population skews to one region.
4. **Hybrid**: combine strategies, e.g. geo-partition first, then hash within each region.

## How sharding works

1. **Choose a shard key** — a column like `user_id`, ideally high-cardinality and evenly distributed.
2. **Apply an algorithm** — e.g. `hash(key) % num_shards` decides the destination shard.
3. **Chunking** — data is split into chunks (MongoDB defaults to 64MB units) that get assigned and moved as whole units.
4. **Routing** — the router/coordinator consults the config server or shard map and forwards each query to the correct shard(s).
5. **Balancing** — a background process migrates chunks between shards if load becomes uneven.
6. **Query execution** — single-shard queries go straight to one shard; multi-shard queries scatter-gather across shards and aggregate results at the router.

**Why it's faster**: parallelism — 4 shards can mean ~4x the scan throughput. Index sizes also shrink per shard (fewer rows each), which speeds up individual seeks. In OLTP workloads this can mean multiples (e.g. 3x) improvement in throughput.

**Shared vs. unshared data**: small, frequently-joined tables (e.g. a country lookup table) are often replicated in full across every shard rather than sharded themselves, so no cross-shard join is needed for common reference data.

## Sharding strategies

The shard key and algorithm are the most consequential decisions in a sharding design — a bad choice creates permanent hotspots that no amount of infrastructure fixes.

**Shard key criteria**:
- **High cardinality**: many unique values (avoid a boolean or other near-binary column — it caps you at ~2 useful shards).
- **Even distribution**: avoid columns that skew (e.g. `age` for a fitness app, where 30-45 dominates).
- **No monotonicity**: avoid always-increasing keys like auto-increment IDs or raw timestamps — all *new* data lands on one "hot" shard while older shards sit idle.
- **Query alignment**: pick a key that matches how data is actually queried (e.g. `timestamp` for a logging system that's queried by time range).

| Strategy | How it works | Pros | Cons | Example use |
|---|---|---|---|---|
| **Range** | Splits by value ranges (e.g. ID 1-1000 → Shard 1) | Efficient range queries | Hotspots — recent/sequential data piles onto one shard | Time-series logs, sharded by date |
| **Hash** | `hash(key) % num_shards` | Even distribution, no hotspots | Ranges become meaningless; resharding is harder | User data with random/opaque IDs |
| **Modulo (MOD)** | `key % num_shards` | Simple, even for sequential IDs | Changing shard count forces a full rehash of everything | Basic even/odd-style load balancing |
| **Tag/Directory** | Lookup table maps a tag/value to a shard (e.g. `color=blue → Shard B`) | Flexible, groupings are meaningful | The lookup table itself is a single point of failure/bottleneck | GDPR: EU tag → EU shard; e-commerce by category |
| **Geo** | Shard by location (city/country/region) | Low latency, regulatory compliance | Uneven if population skews to one region | Dating apps, region-local matching |
| **Composite** | Combine multiple keys (e.g. geo + hash, or tenant + region) | Fine-tuned, relevant balance | More complex to manage and reason about | Global multi-tenant SaaS |
| **Consistent hashing** | Ring-based; adding/removing a shard only remaps a small fraction of keys | Handles growth gracefully, minimal remapping | More setup complexity upfront | Systems that reshard often (e.g. blockchain-style networks) |

Tip: analyze real query patterns first — if most traffic hits recent data, a pure range/date key will create a hot shard; hashing (or a composite key) spreads that load instead.

## Real-world implementations

| Database/tool | Type | Sharding style | Key features |
|---|---|---|---|
| **MongoDB** | NoSQL | Auto, range/hash | Built-in balancer, `mongos` routers, config servers |
| **MySQL + Vitess** | Relational | Manual/auto | YouTube-scale; dynamic resharding; PlanetScale offers zero-downtime workflows on top |
| **PostgreSQL + Citus** | Relational (extension) | Hash/range/schema-based | Distributed tables, transparent to the app, multi-tenant schema sharding |
| **Apache ShardingSphere** | Middleware (proxy/driver) | All strategies | Data migration tooling, DistSQL, federates multiple underlying DBs |
| **Elasticsearch / Solr** | Search | Auto | Lucene-based, geo-sharding, search-optimized |
| **Cassandra / ScyllaDB** | NoSQL | Consistent hashing | Decentralized/masterless, tunable consistency |
| **Google Spanner** | Distributed SQL | Global, auto | ACID + sharding together; scales to trillions of rows across multiple datacenters |
| **CockroachDB** | NewSQL | Auto | Distributed SQL, automatic rebalancing |
| **TiDB** | NewSQL | Auto | HTAP (OLTP+OLAP combined), MySQL-compatible wire protocol |
| **Oracle DB** | Enterprise | Auto (since 12c) | Built-in sharding, multi-model |
| **Neo4j** | Graph | Property sharding | Graph-specific distribution model |

Cloud-managed options: AWS Aurora (auto), Azure SQL Elastic pools, Alibaba DRDS (built for large e-commerce scale). Open-source middleware: Citus (Postgres), Vitess (MySQL), ShardingSphere (cross-database proxy).

The term "shard" tracing back to Ultima Online's parallel worlds is a nice full-circle: both are about splitting one overloaded shared space into independent, load-balanced pieces.

## 🟢 Beginner: advantages

- **Performance**: smaller per-shard datasets mean faster queries and smaller indexes to search — a query that hits one shard instead of a full unsharded table can drop from ~500ms to ~50ms.
- **Scalability**: add shards to grow essentially without a hard ceiling, versus a single server's fixed maximum.
- **Availability**: one shard's failure is a partial outage, not a total one — especially when combined with replication.
- **Cost**: commodity/cloud hardware added incrementally is typically cheaper than repeatedly upgrading one large server.
- **Multi-tenancy**: a proxy layer can shard by tenant, isolating customers from each other's load in a SaaS product.

## 🟡 Intermediate: the hot shard problem, and disadvantages generally

A **hot shard** (hotspot) happens when the shard key doesn't distribute load evenly — one shard gets disproportionate traffic while others sit idle. Common causes:
- A **monotonically increasing** key (auto-increment ID, raw timestamp) — all new writes land on the newest/last shard.
- A **skewed value distribution** — e.g. sharding by `age` in a fitness app dumps the 30-45 bracket onto one shard.
- A **celebrity/power-user problem** — one entity (a viral post, a huge tenant) generates far more traffic than a typical shard was sized for.

Mitigations: switch to hash-based or consistent-hashing keys to break up monotonic sequences; use composite keys to add entropy; monitor per-shard load (Prometheus-style dashboards) and rebalance proactively rather than reactively.

**Other disadvantages**:

| Disadvantage | Description | Mitigation |
|---|---|---|
| **SQL/query complexity** | Joins and transactions that span shards are hard or impossible in the general case | Denormalize data; use a NewSQL system with built-in distributed ACID (e.g. Spanner, CockroachDB) |
| **Single failure point (per shard)** | One shard down can mean part of the data is unavailable | Replicas per shard + automatic failover |
| **Rebalancing overhead** | Moving data between shards can mean downtime or degraded performance mid-migration | Automated online balancers (MongoDB), consistent hashing to minimize data movement |
| **Backup/ops complexity** | Backups and schema changes now have to be coordinated across every shard | Dedicated tooling (Vitess, ShardingSphere), scripted/automated syncs |
| **Cross-shard queries** | Aggregations spanning shards are slow (scatter-gather) | Design queries to be shard-aligned where possible; use approximate results (e.g. sampling) when exact aggregation isn't required |

## 🔴 Advanced: resharding, cross-shard transactions, and combining with replication

### Resharding pain

Changing the number of shards (to add capacity or fix a hotspot) is one of the most operationally painful events in a sharded system:
- With **modulo-based** sharding, changing the shard count changes `key % N` for almost every key — a near-total rehash and data migration.
- With **consistent hashing**, adding/removing a shard only remaps a small fraction of keys (the ring only reassigns the portion adjacent to the change) — this is *why* consistent hashing is the standard choice for systems that expect to reshard over their lifetime.
- Real resharding workflows (Vitess `ReshardTable`, MongoDB's balancer) move data in the background while keeping the system live, but still require careful monitoring — a botched reshard can cause read/write errors or data loss.

### Cross-shard transactions

ACID guarantees don't naturally extend across shard boundaries. Two real approaches:
- **Two-phase commit (2PC)**: coordinator asks all shards to prepare, then commits only if all agree — correct but slow and blocks on the slowest participant.
- **Saga pattern**: break the transaction into a sequence of local transactions with compensating actions to undo earlier steps if a later one fails — used heavily in microservices architectures that already accept eventual consistency.

Distributed SQL systems (Spanner, CockroachDB) build these mechanics in so the application doesn't have to hand-roll them, at the cost of some latency (Spanner's TrueTime-based commits, for instance).

### Combining sharding with replication

The standard production topology: each shard has a primary plus one or more replicas (e.g. P1/R1, P2/R2).

```
App --> Proxy
      |
      |--> Shard1 Primary --> Replica1 (reads)
      |--> Shard2 Primary --> Replica2
```

Benefits: failover (promote a replica if the primary dies, sub-second outage in a well-tuned setup), read scaling (offload reads to replicas), and higher aggregate availability (three replicas can push a shard toward 99.999% availability). The complexity this adds is routing — the proxy/coordinator layer has to track which node is currently primary; sidecar proxies (Envoy-style, one per app instance) are one way to keep that logic decentralized instead of a single shared bottleneck.

### Accessing sharded data: routing, proxies, sidecars

The application generally shouldn't need to know shards exist — that's the point of the middleware layer:
- **Client-side driver**: the app embeds the routing logic itself (largely legacy/deprecated as a pattern, e.g. Hibernate Shards, but the concept persists in some driver libraries).
- **Proxy**: a central component (e.g. ShardingSphere) that looks like one database to the app. Transparent, but can become a bottleneck if not scaled itself.
- **Sidecar**: a per-instance proxy (Istio-style) — more decentralized, better observability per-app, but adds per-request overhead.

Typical query flow: app issues `SELECT * FROM users WHERE id=123` → router computes `hash(123) → Shard 2` → executes on Shard 2 → returns. Multi-shard aggregation (e.g. a global `SUM`) requires scatter-gather across all shards and is inherently slower — a strong argument for designing queries (and shard keys) to stay single-shard whenever possible.

## Migration workflow: sharding an existing database

Real step-by-step of moving a system from unsharded to sharded without extended downtime:

1. **Design**: analyze query patterns (`EXPLAIN ANALYZE` on hot queries) to pick the shard key and algorithm — e.g. hash on `user_id` if that's the dominant filter column in production traffic.
2. **Migrate data**:
   - *Historical* data: bulk-loaded via an ETL process that applies the sharding algorithm as it writes.
   - *Incremental* data: captured via change-data-capture (CDC) — parsing MySQL binlogs or Postgres WAL and replaying changes into the new sharded cluster (tools like Debezium).
   - *Verification*: checksums (e.g. CRC32) or row-by-row comparison, trading speed against certainty.
3. **Dual-write / test phase**: the application writes to both the old and new systems simultaneously while reads still come from the old one, to validate correctness before cutover (`SELECT COUNT(*) FROM old` vs `new` as a sanity check).
4. **Shift traffic**: during an off-peak window, block writes to the old system briefly, flip the proxy/DNS to route to the new sharded cluster, and monitor closely (a spike in 5xx errors is the signal to roll back).

For systems that can't tolerate any write freeze (24/7 operations), a longer "shadow mode" — running the new cluster in parallel for weeks while validating — is the safer path over a hard cutover.

## Hands-on: sharding in MongoDB

MongoDB auto-shards collections; the balancer handles rebalancing for you.

```yaml
# docker-compose.yml — simplified local cluster
version: '3'
services:
  mongos:
    image: mongo:7.0
    ports: ["27017:27017"]
    depends_on: [configsvr]
  configsvr:
    image: mongo:7.0
    command: --configsvr --replSet configReplSet
  shard1:
    image: mongo:7.0
    command: --shardsvr --replSet rs1
  shard2:
    image: mongo:7.0
    command: --shardsvr --replSet rs2
```

```js
// Connect via mongosh, then initialize replica sets for config server and each shard
// (rs.initiate({...}) on each), then from mongos:

sh.addShard("shard1/rs1")
sh.addShard("shard2/rs2")
sh.enableSharding("mydb")
sh.shardCollection("mydb.users", { user_id: "hashed" })  // hashed shard key

// Insert and query
db.users.insertMany([
  { user_id: 1, name: "Alice", region: "EU" },
  { user_id: 2, name: "Bob", region: "US" }
])
db.users.find({ region: "EU" })   // routed to the relevant shard(s)
sh.status()                        // shows chunk distribution across shards
```

`hash(user_id)` decides the destination shard; the balancer moves chunks automatically if distribution becomes uneven. The app talks to `mongos` and never sees the underlying shards — inserting 1000 sequentially-increasing IDs is a good way to observe a hotspot and watch the balancer react.

## Hands-on: sharding in PostgreSQL with Citus

Citus turns Postgres into a distributed database via extension rather than a different engine.

```sql
-- On the coordinator node
CREATE EXTENSION citus;
SELECT citus_set_coordinator_host('localhost', 5432);

-- Create and distribute a table
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INT,
    amount DECIMAL,
    created_at TIMESTAMP
);
SELECT create_distributed_table('orders', 'user_id');  -- hashes on user_id, 32 shards by default

-- Add data and query
INSERT INTO orders (user_id, amount, created_at)
VALUES (123, 99.99, NOW()), (456, 49.99, NOW());

SELECT * FROM orders WHERE user_id = 123;  -- routes to a single worker shard
SELECT COUNT(*) FROM orders;                -- aggregates across all shards

-- Add more capacity
SELECT citus_add_node('worker1', 5433);

-- Inspect shard layout
SELECT * FROM citus_shards;
```

Citus 12+ also supports **schema-based sharding** for multi-tenant SaaS: each tenant gets its own schema, and `citus_distribute_schema('tenant1')` shards at the schema level rather than per-row. The app connects to the coordinator and queries it like vanilla Postgres — the distribution is transparent.

## Hands-on: sharding in MySQL with Vitess

Vitess adds sharding to MySQL via a proxy layer (`VTGate`) — this is the engine behind YouTube's and PlanetScale's scale-out MySQL.

```sql
-- Define the table behind the proxy
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  region VARCHAR(20)
);
```
```bash
# Apply a sharding rule (shard key = id)
vtctlclient ApplyVSchema -vschema mydb.users 'table: {shard_key: id}'

# Insert and query through the proxy — transparent to the app
mysql -h 127.0.0.1 -P 15306 -u vtuser mydb -e "INSERT INTO users VALUES (1, 'Alice', 'EU');"
mysql -h 127.0.0.1 -P 15306 -u vtuser mydb -e "SELECT * FROM users WHERE id=1;"

# Add capacity later
vtctlclient ReshardTable mydb.users
```

Vitess hashes `id` to determine the target shard; the app only ever talks to the proxy.

## Real-world examples

| System | Sharding approach | Notes |
|---|---|---|
| **Facebook** (TAO + MySQL) | User posts sharded by ID hash | Handles billions of users; geo-shards to reduce latency; cross-shard feed aggregation solved with edge caches |
| **Netflix** (Cassandra) | Viewer/profile data by region, hash + range for recommendations | Geo-sharding cuts latency (~50ms improvement cited); shards auto-scale during traffic peaks (e.g. big show drops) |
| **Uber** (Schemaless + MySQL) | Rides sharded by city ID (geo) | Consistent hashing allows adding shards without full remaps; ML used to predict and rebalance hotspots |
| **Stack Overflow** (MySQL, later Vitess) | Sharded by tag/user_id | Started vertical, moved to hybrid horizontal sharding; resharded without downtime |
| **YouTube** (Vitess, Google) | Videos/comments sharded by ID | Dynamic balancing absorbs traffic spikes; shards span continents for CDN alignment |
| **Ethereum** (blockchain shard chains) | Transaction/state data split across shard chains (64 planned) | Reduces per-node verification load roughly proportional to shard count |
| **Alibaba** (DRDS) | Orders range-sharded by date/region | Built to survive Singles' Day-scale order volume (over a billion orders) |
| **AWS IoT**-style systems | Sensor data geo-sharded by device/location | Even distribution via hashing to handle very high event-per-second ingestion |

Small-scale example: a fitness app shards users by join date (range-based) — the most recent shard gets extra replicas to absorb high write volume from new signups, while older shards serve mostly read/analytics traffic.

## Common pitfalls / gotchas

- **Sharding too early.** If caching, indexing, or a read replica would solve the actual bottleneck, sharding just adds operational complexity for no benefit. Optimize those first.
- **Choosing a monotonic shard key** (auto-increment ID, raw timestamp) — creates a permanent hot shard for all new writes.
- **Low-cardinality shard keys** (booleans, small enums) — caps effective parallelism at the number of distinct values.
- **Treating cross-shard joins/transactions as free** — they require either denormalization, 2PC, or saga-style compensation; none of these are "just works."
- **Underestimating rebalancing cost** — resharding with a modulo scheme forces a near-total data reshuffle; plan for consistent hashing if reshard events are expected.
- **No plan for the router/proxy as a new single point of failure** — if it's not itself replicated, the proxy becomes the outage risk that sharding was supposed to avoid.
- **Ignoring per-shard monitoring** — without visibility into per-shard load, hotspots go unnoticed until they're an incident.

## Best practices

1. **Key selection**: high cardinality, evenly distributed, ideally immutable, aligned with actual query patterns. Sanity-check distribution with something like `SELECT COUNT(*) FROM table GROUP BY shard_key`.
2. **Start small**: prototype with 2-3 shards and monitor skew before committing to a production topology.
3. **Use existing balancers/tools** rather than hand-rolling rebalancing logic (MongoDB balancer, Vitess workflows, Citus rebalancer).
4. **Denormalize where needed** to avoid cross-shard joins — embedding related data (NoSQL-style) is often simpler than distributed joins.
5. **Monitor per-shard**: size, QPS, latency; alert on skew (e.g. >80% CPU on one shard while others are idle).
6. **Test failure modes**: chaos-engineering style — kill a shard, measure failover time and correctness.
7. **Plan backups and schema changes as cluster-wide operations**, coordinated and scripted, not manual per-shard steps.
8. **Prefer online/zero-downtime schema changes** (e.g. tools like gh-ost for MySQL) over blocking DDL across a live sharded cluster.
9. **Shard what actually grows** — leave small, stable config/lookup tables unsharded (replicate them instead).
10. **Golden rule**: shard after caching and indexing have been exhausted — it should be a late-stage lever, not a first response to scale problems.

## Quick reference

- **Sharding** = rows split across multiple servers. **Partitioning** = split within one server. **Replication** = full copy across servers.
- Shard key criteria: high cardinality, even distribution, non-monotonic, aligned to query patterns.
- Strategies: range (good for range queries, hotspot risk), hash (even, loses range queries), modulo (simple, expensive to reshard), tag/directory (flexible, lookup table is a SPOF), geo (low latency/compliance, can be uneven), consistent hashing (best for systems that reshard often).
- Hot shard = one shard overloaded from a bad key choice (monotonic ID, skewed values, celebrity/power-user traffic).
- Cross-shard transactions: 2PC (correct, slow) or sagas (eventual consistency, used in microservices).
- Standard production topology: shard + replicate together (primary + replicas per shard).
- Access patterns: client-side driver (legacy), central proxy (simple, can bottleneck), sidecar (decentralized, more overhead).
- Migration: analyze → migrate (bulk + CDC) → dual-write/verify → cutover off-peak, monitor for rollback triggers.
- Tools: MongoDB (native), Citus (Postgres), Vitess (MySQL), ShardingSphere (cross-DB proxy), Spanner/CockroachDB/TiDB (distributed SQL with sharding built in).

## Further reading
- "How Sharding a Database Can Make It Faster" — Juan Pan (SphereEx), Stack Overflow Blog.
- Wikipedia: Shard (database architecture).
- *Database Internals* by Alex Petrov.
- Vitess, Citus, and ShardingSphere documentation for implementation-level detail.
