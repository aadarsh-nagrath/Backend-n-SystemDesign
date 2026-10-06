# MongoDB Deep Dive

## Table of Contents
1. [Core Concepts](#core)
2. [Data Modeling: Embed vs Reference, and Patterns](#modeling)
3. [Queries and the Aggregation Pipeline](#queries)
4. [Indexes](#indexes)
5. [WiredTiger Storage Engine](#wiredtiger)
6. [Replica Sets: Elections, Write Concern, Read Concern, Read Preference](#replsets)
7. [Transactions](#transactions)
8. [Sharding](#sharding)
9. [Change Streams](#changestreams)
10. [Schema Validation and Migrations](#schema)
11. [Performance and Operations](#ops)
12. [When (Not) to Use MongoDB](#when)
13. [Interview Questions](#qa)

---

## 1. Core Concepts {#core}

| MongoDB | Relational analogy |
|---|---|
| Database | Database |
| Collection | Table |
| Document (BSON, max **16 MB**) | Row (but nested, flexible) |
| Field | Column |
| `_id` (unique, default ObjectId) | Primary key |
| Embedded document / array | Joined child rows |
| `$lookup` | LEFT OUTER JOIN |
| Index | Index |
| Replica set | Primary + replicas with automatic failover |
| Shard | Horizontal partition |

**BSON**: binary JSON with extra types (ObjectId, Date, Decimal128, Int32/Int64, Binary, Timestamp).

**ObjectId** (12 bytes): 4-byte timestamp + 5-byte random per process + 3-byte counter. It's roughly time-ordered, and the creation time can be extracted.

---

## 2. Data Modeling {#modeling}

**Rule: data that is accessed together should be stored together.**

### Embed when
- One-to-few relationships (a user's 3 addresses).
- Data is read together with the parent most of the time.
- The child doesn't need to be queried or updated independently at high rates.
- The embedded array is **bounded** (documents max out at 16 MB, and huge arrays make updates slow).

### Reference when
- One-to-many with many or unbounded children (a product's millions of reviews).
- Many-to-many relationships.
- The child is shared by many parents and updated independently (an author's profile shown on every post).
- Large subdocuments are rarely needed.

```js
// Embedded: order with line items (bounded, always read together)
{ _id: ObjectId(...), customerId: 42, status: "paid",
  items: [ { sku: "A1", qty: 2, price: NumberDecimal("199.00") }, ... ],
  shipping: { city: "Pune", pincode: "411001" },
  createdAt: ISODate("2025-06-01T10:00:00Z") }

// Referenced: reviews in their own collection
{ _id: ..., productId: ObjectId("..."), userId: ..., rating: 5, text: "...", createdAt: ... }
```

### Named schema design patterns (MongoDB University)
| Pattern | Problem it solves | Example |
|---|---|---|
| **Extended reference** | Avoid `$lookup` for commonly needed fields | Embed `{customerId, name}` in each order |
| **Subset** | Huge array, only recent items needed | Product doc holds last 10 reviews; rest in reviews collection |
| **Computed** | Expensive aggregates read often | `reviewCount`, `avgRating` updated on write |
| **Bucket** | Many small time-series points | One doc per sensor per hour with an array of readings (native time-series collections now do this) |
| **Outlier** | A few docs exceed normal bounds | Flag `hasOverflow: true` and spill to extra docs (celebrity with 10M followers) |
| **Attribute** | Many similar sparse fields to index | `attrs: [{k: "color", v: "red"}, {k: "size", v: "M"}]` + one index on `attrs.k, attrs.v` |
| **Polymorphic** | Different shapes in one collection | `type` field + shape per type |
| **Schema versioning** | Gradual migrations | `schemaVersion: 2`; code handles both |
| **Tree** patterns | Hierarchies | Parent refs, child refs, array of ancestors, materialized paths |
| **Approximation** | Write-heavy counters | Increment by 10 every 10th event |
| **Archive** | Old data | Move to cheaper cluster or Atlas Online Archive |

Anti-patterns: massive or unbounded arrays, too many collections (each collection plus its indexes has overhead), unnecessary indexes, bloated documents (reading 2 MB to show a title), case-insensitive queries without a collation index, and separating data that is always accessed together.

---

## 3. Queries and Aggregation {#queries}

```js
db.orders.find(
  { customerId: 42, status: { $in: ["paid", "shipped"] }, createdAt: { $gte: ISODate("2025-01-01") } },
  { items: 0 }                                     // projection: exclude items
).sort({ createdAt: -1 }).limit(20)

db.orders.updateOne({ _id: id, status: "pending" },        // conditional (atomic per document)
  { $set: { status: "paid" }, $inc: { version: 1 }, $push: { history: { at: new Date(), to: "paid" } } })

db.counters.findOneAndUpdate({ _id: "orderNo" }, { $inc: { seq: 1 } }, { upsert: true, returnDocument: "after" })
```
**Single-document operations are atomic**, including updates to nested arrays. Designing so that one business operation touches one document avoids the need for transactions.

Array update operators: `$push` (with `$each`, `$slice` to cap size, `$sort`), `$addToSet`, `$pull`, positional `$`, `$[]`, `$[<identifier>]` with arrayFilters.

### Aggregation pipeline
```js
db.orders.aggregate([
  { $match: { status: "paid", createdAt: { $gte: ISODate("2025-06-01") } } },   // use indexes: put $match/$sort first
  { $unwind: "$items" },
  { $group: { _id: "$items.sku", revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } }, orders: { $sum: 1 } } },
  { $sort: { revenue: -1 } },
  { $limit: 10 },
  { $lookup: { from: "products", localField: "_id", foreignField: "sku", as: "product" } },
  { $project: { revenue: 1, orders: 1, name: { $first: "$product.name" } } }
], { allowDiskUse: true })
```
Stages: `$match`, `$project`/`$set`/`$unset`, `$group`, `$sort`, `$limit`/`$skip`, `$unwind`, `$lookup` (joins; correlated pipelines; works with sharded collections since 5.1), `$facet` (multiple pipelines), `$bucket`, `$graphLookup` (recursive), `$setWindowFields` (window functions, 5.0), `$merge`/`$out` (materialize results), `$unionWith`, `$densify`, `$fill`, `$search` (Atlas Search, Lucene), `$vectorSearch` (Atlas).

Memory: each stage is limited to 100 MB unless `allowDiskUse` (default true since 6.0).

---

## 4. Indexes {#indexes}

B-tree indexes (in WiredTiger), on any field including nested and arrays:
| Type | Notes |
|---|---|
| Single field, compound | Compound order matters: **ESR rule = Equality, Sort, Range** |
| Multikey | Automatically when a field is an array; one entry per element. A compound index can include at most one array field |
| Text | Basic full-text (Atlas Search is far better) |
| 2dsphere / 2d | Geospatial |
| Hashed | For hashed sharding; equality only |
| Wildcard (`$**`) | Index unknown/dynamic fields |
| Partial | `partialFilterExpression` |
| Sparse | Skip docs missing the field (partial supersedes it) |
| TTL | Auto-delete docs after `expireAfterSeconds` (background task ~every 60 s) |
| Unique | Including compound unique |
| Clustered collections (5.3+) | Docs stored in `_id` order |
| Hidden indexes | Like MySQL invisible indexes |

**ESR rule** example: query `{status: "paid", createdAt: {$gte: X}}` sorted by `{amount: -1}`. Index: `{status: 1, amount: -1, createdAt: 1}`: equality (status), then sort (amount), then range (createdAt). This avoids an in-memory sort.

Explain:
```js
db.orders.find({...}).sort({...}).explain("executionStats")
// winningPlan stages: IXSCAN (good), FETCH, COLLSCAN (full scan), SORT (in-memory sort, bad on big sets), SORT_MERGE
// executionStats: nReturned, totalKeysExamined, totalDocsExamined, executionTimeMillis
// Ideal: keysExamined ≈ docsExamined ≈ nReturned
```
Covered queries: projection includes only indexed fields and excludes `_id` (unless it's indexed in that index), so `totalDocsExamined: 0`.

Index builds (4.2+) use an optimized process holding exclusive locks only at the start and end. On replica sets, builds run simultaneously on all data-bearing members (4.4+). Rolling builds are an alternative.

---

## 5. WiredTiger {#wiredtiger}

The default storage engine since 3.2:
- **Document-level concurrency control** (optimistic; write conflicts are retried internally).
- **MVCC with snapshots**. Checkpoints every 60 s write a consistent snapshot to disk. A **journal** (WAL, group-committed every 100 ms by default or on `j:true`) covers writes between checkpoints.
- **Compression**: snappy by default for collections (zstd/zlib optional), prefix compression for indexes.
- **Cache**: `max(50% of (RAM − 1 GB), 256 MB)` by default. The OS file cache holds compressed data too. Watch cache dirty % and eviction pressure. Application threads doing eviction is a sign of saturation.
- History store (4.4+) keeps old versions for snapshot reads; long-running transactions and snapshot reads pressure the cache.

---

## 6. Replica Sets {#replsets}

- 1 **primary** + N **secondaries** (typically 3 members total; max 50, max 7 voting).
- Replication via the **oplog**: a capped collection (`local.oplog.rs`) of idempotent operations that secondaries tail and apply. **Oplog window** = time span the oplog covers. A secondary down longer than the window must do a full resync. Size it for hours to days.
- **Elections**: Raft-like protocol (pv1). If the primary is unreachable for `electionTimeoutMillis` (10 s), an eligible secondary calls an election. A majority of voting members is needed. Priorities and tags influence placement. Typical failover is ~10–12 s.
- **Arbiter**: voting member with no data. Avoid it in production with majority write concern (PSA topology problems).
- Hidden and delayed members for backups/analytics or a "time machine".

### Write concern
`{ w: <n | "majority">, j: <bool>, wtimeout: ms }`
- `w:1`: acknowledged by the primary only. It **can be rolled back** if the primary fails before replicating (rolled-back writes are saved to rollback files).
- `w:"majority"` (default since 5.0): acknowledged once a majority has it, so it survives failover.
- `j:true`: journaled to disk before acknowledgement.

### Read concern
- `local` (default for reads on primary): latest data, might be rolled back.
- `available`: for sharded secondaries, may return orphans.
- `majority`: data acknowledged by a majority (won't be rolled back).
- `linearizable`: majority plus confirms the primary is still the primary (single-document reads, slow).
- `snapshot`: for transactions; point-in-time across documents.

### Read preference
`primary` (default), `primaryPreferred`, `secondary`, `secondaryPreferred`, `nearest`, plus tag sets and `maxStalenessSeconds`. Reads from secondaries are **eventually consistent**. Use **causally consistent sessions** (`startSession({causalConsistency: true})` with majority read/write concerns) for read-your-writes when reading from secondaries.

---

## 7. Transactions {#transactions}

Multi-document ACID transactions: replica sets since 4.0, sharded clusters since 4.2.
```js
const session = client.startSession();
await session.withTransaction(async () => {
  await accounts.updateOne({ _id: "A", balance: { $gte: 100 } }, { $inc: { balance: -100 } }, { session });
  await accounts.updateOne({ _id: "B" }, { $inc: { balance: 100 } }, { session });
}, { readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } });
```
- Snapshot isolation. Write conflicts abort with `TransientTransactionError`, and `withTransaction` retries automatically. `UnknownTransactionCommitResult` → retry the commit.
- Default max lifetime is 60 s (`transactionLifetimeLimitSeconds`). Keep transactions short and small (each is limited by the 16 MB oplog entry size, though large transactions span multiple oplog entries since 4.2).
- Performance cost is significant. **Prefer modeling for single-document atomicity**, and use transactions for the genuinely cross-document cases.

---

## 8. Sharding {#sharding}

Components:
- **Shards**: each is a replica set holding a subset of data.
- **Config servers** (a replica set): metadata such as chunk/range → shard mapping.
- **mongos**: query routers; apps connect to them.

Shard key choice (the most important and, before 5.0, irreversible decision; now `reshardCollection` exists in 5.0+, and `refineCollectionShardKey` 4.4+):
| Property | Why |
|---|---|
| High cardinality | Many distinct values → many chunks to spread |
| Low frequency | No single value dominating (hot chunk) |
| Non-monotonic (for ranged) | Monotonic keys (ObjectId, timestamp) send all inserts to the last chunk → one hot shard |
| Matches query patterns | Queries including the shard key are **targeted** to one shard; others are **scatter-gather** to all shards |

Strategies:
- **Ranged**: efficient range queries on the key, but hotspot risk with monotonic keys.
- **Hashed**: even distribution, but range queries scatter.
- **Compound**: `{ tenantId: 1, createdAt: 1 }` is targeted per tenant, spread across tenants.
- **Zones**: pin ranges to shards (data residency: EU tenants on EU shards).

Data is split into **chunks/ranges** (default 128 MB in 6.0+), and the **balancer** migrates them between shards. Jumbo chunks (can't split because all docs share a key value) are a classic problem.

Sharded cluster constraints: unique indexes must be prefixed by the shard key, and transactions across shards are slower (2PC).

---

## 9. Change Streams {#changestreams}

```js
const stream = db.collection("orders").watch(
  [{ $match: { operationType: { $in: ["insert", "update"] }, "fullDocument.status": "paid" } }],
  { fullDocument: "updateLookup" }
);
for await (const change of stream) { /* publish to Kafka, update cache, ... */ saveResumeToken(change._id); }
```
- Built on the oplog. Ordered, resumable via **resume tokens** (as long as the token is still within the oplog window), majority-committed changes only.
- Uses: CDC to Kafka (MongoDB Kafka Connector, Debezium), cache invalidation, real-time features, triggers (Atlas Triggers).
- Pre- and post-images (6.0+) for full before/after documents.

---

## 10. Schema Validation and Migrations {#schema}

```js
db.createCollection("users", { validator: { $jsonSchema: {
  bsonType: "object", required: ["email", "createdAt"],
  properties: {
    email: { bsonType: "string", pattern: "^.+@.+$" },
    age: { bsonType: "int", minimum: 0 },
    createdAt: { bsonType: "date" }
  }}}, validationLevel: "strict", validationAction: "error" });
```
Migration strategies:
- **Lazy (on read/write)**: code reads `schemaVersion`, upgrades the document in memory, and writes the new version when saving.
- **Eager backfill**: batch update in the background, throttled.
- Both together: deploy code that handles v1 and v2, backfill, then remove the v1 handling.
- ODMs: Mongoose (Node), Spring Data MongoDB, Beanie/ODMantic (Python), Mongoid (Ruby).

---

## 11. Performance and Operations {#ops}

- Keep the **working set (frequently accessed docs + indexes) in the WiredTiger cache**.
- Profile: `db.setProfilingLevel(1, { slowms: 100 })`, `db.system.profile`, `$currentOp`, `mongotop`, `mongostat`, Atlas Performance Advisor / Query Profiler.
- Avoid unindexed queries (COLLSCAN), in-memory SORT stages, large `$skip` pagination (use range-based), and unbounded `$lookup` fan-out.
- Use projections, `bulkWrite` for batches, and `insertMany` with `ordered: false` for throughput.
- Connection pooling (`maxPoolSize`, default 100 per mongos/host in drivers).
- **Retryable writes** (default on in modern drivers) safely retry once on failover. Retryable reads too.
- Backups: Atlas continuous backups/PITR, `mongodump`/`mongorestore` (logical, small DBs), filesystem snapshots with journaling, Percona Backup for MongoDB, Ops Manager.
- Time-series collections (5.0+) for metrics/IoT: automatic bucketing and compression.
- Licensing: **SSPL** since 2018 (not OSI open source), which triggered AWS DocumentDB (API-compatible, different engine), FerretDB (MongoDB protocol on Postgres), and Azure Cosmos DB for MongoDB.

---

## 12. When (Not) to Use MongoDB {#when}

Good fit:
- Aggregates with nested, variable structure read as a whole (catalogs, CMS content, user profiles, configuration, event payloads).
- Rapidly evolving schemas in early product stages.
- Horizontal scaling with sharding and built-in HA.
- Geo queries, mobile/edge sync (Atlas Device Sync), developer velocity with JSON-native stacks.

Poor fit:
- Highly relational data with many-to-many relationships and complex joins/reporting.
- Strong multi-entity invariants everywhere (heavy transaction use defeats the purpose).
- Analytics over the whole dataset (use a warehouse; Atlas Data Federation / SQL interface exist).

---

## 13. Interview Questions {#qa}

1. Embed vs reference: how do you decide? Give examples of each.
2. Why is a monotonically increasing shard key bad with ranged sharding?
3. Explain write concern `w:1` vs `majority`. What can be lost with `w:1`?
4. How do you get read-your-writes when reading from secondaries?
5. What is the ESR rule for compound indexes?
6. When would you use multi-document transactions, and why try to avoid them?
7. What is the oplog, and what happens if a secondary falls behind the oplog window?
8. How do change streams work and what are they used for?
9. How do you handle schema migrations in a "schemaless" database?
