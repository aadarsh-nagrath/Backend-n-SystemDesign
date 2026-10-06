# Amazon DynamoDB Deep Dive

## Table of Contents
1. [What DynamoDB Is](#what)
2. [Core Concepts: Tables, Items, Keys](#core)
3. [Partitions, Throughput, and Hot Keys](#partitions)
4. [Capacity Modes and Cost](#capacity)
5. [Secondary Indexes: LSI vs GSI](#indexes)
6. [Single-Table Design](#singletable)
7. [Operations: Get, Query, Scan, Conditional Writes, Updates](#operations)
8. [Consistency and Transactions](#consistency)
9. [Streams, TTL, Global Tables, Backups](#streams)
10. [DAX and Caching](#dax)
11. [Limits and Gotchas](#limits)
12. [When to Use (and Not)](#when)
13. [Interview Questions](#qa)

---

## 1. What DynamoDB Is {#what}

A fully managed, serverless key-value and document database from AWS (2012), inspired by but **different from** the 2007 Dynamo paper. It uses leader-based replication per partition with Multi-Paxos, not leaderless quorums.
- **Single-digit-millisecond latency at any scale**: Amazon retail Prime Day peaks reach tens of millions of requests per second.
- No servers, no patching, no connection pools (HTTPS API with SigV4 auth), automatic partitioning, 3-AZ replication.
- Trade-offs: you must know access patterns upfront, there are no joins and limited ad-hoc queries, cost scales with access patterns, and you're locked into AWS.

---

## 2. Core Concepts {#core}

- **Table** → **items** (≤ **400 KB** each) → **attributes** (typed: S, N, B, BOOL, NULL, M (map), L (list), SS/NS/BS (sets)). Schemaless apart from the key.
- **Primary key**, one of:
  - **Simple**: partition key (PK) only. Item identity = PK.
  - **Composite**: partition key + **sort key** (SK). Items with the same PK form an **item collection**, sorted by SK, and can be range-queried (`begins_with`, `between`, `<`, `>`).
- The partition key is hashed to choose the physical partition.

---

## 3. Partitions, Throughput, Hot Keys {#partitions}

- Data is split into partitions of up to ~10 GB. Each partition supports up to **3,000 RCU and 1,000 WCU** per second.
- DynamoDB splits partitions automatically as data or throughput grows.
- **Adaptive capacity** shifts throughput toward hot partitions, and **split for heat** isolates hot items. But a **single partition key** is still capped around 1,000 writes/s and 3,000 strongly consistent reads/s.
- Hot key mitigation:
  - High-cardinality partition keys (userId, orderId, not status or date).
  - **Write sharding**: `PK = "LEADERBOARD#2025-06-01#" + random(0..9)`, then query all 10 shards and merge.
  - Cache hot reads (DAX/ElastiCache).
- Throttling shows up as `ProvisionedThroughputExceededException` / `ThrottlingException`. The SDKs retry with backoff.

---

## 4. Capacity Modes and Cost {#capacity}

Units:
- **RCU**: one strongly consistent read/s of up to 4 KB (or 2 eventually consistent reads). Transactional reads cost 2 RCU per 4 KB.
- **WCU**: one write/s of up to 1 KB. Transactional writes cost 2×.
- Item size matters: a 3.5 KB write = 4 WCU. Keep items small, and move large blobs to S3 with a pointer.

| Mode | How | When |
|---|---|---|
| **On-demand** | Pay per request; instant scaling (to 2× previous peak instantly, more over time) | Unpredictable, spiky, or new workloads; dev/test. AWS cut on-demand prices ~50% in late 2024, making it the default choice for many |
| **Provisioned** | Set RCU/WCU, optionally with auto scaling (target utilization) | Steady, predictable traffic; cheaper at high sustained utilization; reserved capacity for further savings |

Other cost drivers: storage (Standard vs Standard-IA table class), GSIs (each GSI write costs extra WCU), streams reads, global tables (replicated writes), backups, and data transfer.

**Scans are expensive**: they read (and bill for) every item.

---

## 5. Secondary Indexes {#indexes}

| | Local Secondary Index (LSI) | Global Secondary Index (GSI) |
|---|---|---|
| Key | Same PK, **different sort key** | **Any** PK and SK |
| Created | Only at table creation | Anytime |
| Consistency | Strong or eventual | **Eventual only** (async propagation) |
| Throughput | Shares the table's | Its own capacity; under-provisioned GSI throttles **table writes** (back-pressure) |
| Size limit | Item collection (PK) ≤ **10 GB** with LSIs | None |
| Limit | 5 per table | 20 per table (default quota) |
| Projection | KEYS_ONLY / INCLUDE / ALL | Same |

**Sparse indexes**: a GSI only contains items that have the GSI key attributes. That makes it an efficient filter, e.g., a GSI on `pendingSince` only contains pending orders.

GSI **overloading**: generic attribute names (`GSI1PK`, `GSI1SK`) hold different meanings per entity type, so one GSI serves several access patterns (single-table design).

---

## 6. Single-Table Design {#singletable}

Popularized by Rick Houlihan (AWS) and Alex DeBrie (*The DynamoDB Book*): store **multiple entity types in one table**, with generic key names, so related items share a partition and can be fetched in **one Query**. It's the NoSQL way to "pre-join".

Example: an e-commerce slice.
| PK | SK | Attributes | Purpose |
|---|---|---|---|
| `CUSTOMER#42` | `PROFILE` | name, email | Customer |
| `CUSTOMER#42` | `ORDER#2025-06-01#9001` | status, total | Customer's orders (sorted by date) |
| `CUSTOMER#42` | `ADDRESS#home` | line1, city | Addresses |
| `ORDER#9001` | `ORDER` | customerId, status, total | Order by id |
| `ORDER#9001` | `ITEM#A1` | qty, price | Order line items |
| GSI1PK = `STATUS#PENDING`, GSI1SK = `2025-06-01T10:00` | | | Pending orders by time (sparse) |
| GSI2PK = `EMAIL#a@b.com`, GSI2SK = `CUSTOMER#42` | | | Lookup customer by email |

Access patterns:
- Get customer + recent orders + addresses: `Query PK = CUSTOMER#42` (one request, one partition).
- Customer's orders in June: `Query PK = CUSTOMER#42 AND begins_with(SK, "ORDER#2025-06")`.
- Order with items: `Query PK = ORDER#9001`.
- Pending orders: `Query GSI1 PK = STATUS#PENDING` (watch out: low-cardinality GSI PK means a hot partition, so shard it if high volume).

Methodology:
1. Draw an ERD.
2. **List every access pattern** (with parameters, sort order, and volume).
3. Design PK/SK patterns to satisfy them, adding GSIs for the rest.
4. Use **composite sort keys** (`STATUS#DATE`) for hierarchical filtering.
5. Duplicate attributes where needed, and use **transactions** or **streams** to keep copies consistent.

Trade-offs: single-table design is efficient but hard to evolve, hard to read in the console, and hard to export for analytics. **Multi-table designs** are fine when access patterns don't need pre-joined collections. Many teams with GraphQL (AppSync) or evolving needs prefer them.

---

## 7. Operations {#operations}

```python
table.get_item(Key={"PK": "CUSTOMER#42", "SK": "PROFILE"}, ConsistentRead=True)

table.query(
  KeyConditionExpression=Key("PK").eq("CUSTOMER#42") & Key("SK").begins_with("ORDER#2025"),
  ScanIndexForward=False, Limit=20,
  FilterExpression=Attr("status").eq("SHIPPED"))   # filter applies AFTER read: you pay for filtered-out items

# Conditional write: create only if absent (idempotent create / uniqueness)
table.put_item(Item={...}, ConditionExpression="attribute_not_exists(PK)")

# Atomic counter + optimistic locking
table.update_item(
  Key={"PK": "PRODUCT#9", "SK": "STOCK"},
  UpdateExpression="SET qty = qty - :one, version = version + :one",
  ConditionExpression="qty >= :one AND version = :v",
  ExpressionAttributeValues={":one": 1, ":v": 7})
```
- **GetItem / BatchGetItem** (≤ 100 items, 16 MB).
- **Query**: one partition key, optional SK condition, results ≤ **1 MB per page**. Paginate with `LastEvaluatedKey` → `ExclusiveStartKey`.
- **Scan**: whole table; parallel scan with `Segment`/`TotalSegments`. Avoid it in request paths.
- **PutItem / UpdateItem / DeleteItem** with `ConditionExpression` (compare-and-set, uniqueness, optimistic locking).
- **BatchWriteItem** (≤ 25 puts/deletes). Not atomic; check `UnprocessedItems` and retry.
- `ReturnValues` (ALL_OLD, UPDATED_NEW, …), `ReturnValuesOnConditionCheckFailure`.
- PartiQL (`SELECT * FROM "table" WHERE PK = ?`): SQL-ish syntax over the same operations. A `SELECT` without a key condition **becomes a Scan**.

**Uniqueness on non-key attributes** (e.g., unique email): write a second item `PK = "EMAIL#a@b.com"` with `attribute_not_exists` in the same transaction as the user item.

---

## 8. Consistency and Transactions {#consistency}

- Writes are acknowledged after durable replication to 2 of 3 AZ replicas (leader + quorum).
- **Eventually consistent reads** (default, half price) may return stale data briefly (usually < 1 s).
- **Strongly consistent reads** (`ConsistentRead=True`) are served by the leader. They're not available on GSIs or across regions in global tables (except the newer multi-Region strong consistency mode for global tables).
- **Transactions**: `TransactWriteItems` / `TransactGetItems`, up to **100 items** (raised from 25 in 2022), ACID across items and tables in one region and account, with **serializable** isolation for the transaction items. They cost 2× capacity. They fail with `TransactionCanceledException` on condition failure or conflict. **Idempotency**: pass a `ClientRequestToken` (valid 10 min), so retries don't double-apply.

---

## 9. Streams, TTL, Global Tables, Backups {#streams}

**DynamoDB Streams**: an ordered (per item) change log for 24 hours, with view types KEYS_ONLY / NEW_IMAGE / OLD_IMAGE / NEW_AND_OLD_IMAGES. Consume with **Lambda** triggers or the KCL. Uses:
- Maintain denormalized copies and aggregates (counters, materialized views).
- Replicate to OpenSearch for search, S3/warehouse for analytics.
- Event-driven workflows (outbox-like: the item write *is* the event).
- Alternatively, **Kinesis Data Streams for DynamoDB** gives longer retention and more consumers.

**TTL**: set an epoch-seconds attribute, and items expire and get deleted in the background (typically within days of expiry, not exact) at no WCU cost. Deletions appear in streams (with a service principal identity), which lets you archive expired items. Filter expired-but-not-yet-deleted items in reads if exactness matters.

**Global Tables**: multi-region, **multi-active** replication (typically sub-second). Conflict resolution is **last-writer-wins**. Design to avoid concurrent writes to the same item from different regions (route each user to a home region). A newer mode offers multi-region strong consistency (MRSC) at higher write latency.

**Backups**: on-demand backups, **PITR** (continuous, any second in the last 35 days), export to S3 (no RCU consumption) for analytics with Athena/Glue, and import from S3.

---

## 10. DAX and Caching {#dax}

**DynamoDB Accelerator (DAX)**: an in-memory, write-through cache cluster with an API-compatible client, offering microsecond reads.
- Item cache (GetItem/BatchGetItem) and query cache (keyed by parameters).
- **Eventually consistent only**: strongly consistent reads pass through. Query cache entries aren't invalidated by item writes (they rely on TTL), so stale query results are possible.
- Use for read-heavy, repeated reads of hot items. Alternatively, ElastiCache/Redis with explicit cache-aside gives more control.

---

## 11. Limits and Gotchas {#limits}

- Item size 400 KB (including attribute names, so shorten names on huge-volume tables).
- Query/Scan page = 1 MB before filters, so a "Limit 20" with a filter can return 0 items plus a LastEvaluatedKey. Clients must keep paginating.
- **No ad-hoc queries**: new access patterns need new GSIs (backfilled automatically, at a cost) or a redesign.
- **No server-side aggregation** (`COUNT`, `SUM` across items). Maintain aggregates via streams or atomic counters, or export to analytics.
- LSIs must be created at table creation and cap item collections at 10 GB.
- GSIs are eventually consistent, and GSI throttling back-pressures table writes.
- Hot partitions cap per-key throughput.
- Number type is up to 38 digits of precision, sent as strings in the API (use Decimal in Python, not float).
- Empty strings are allowed for non-key attributes (since 2020), but not for key attributes.
- Reserved words in expressions need `ExpressionAttributeNames` (`#name`).
- Pagination tokens are opaque and tied to the query. There's no "jump to page N".
- Cost surprises: Scans, large items, many GSIs (each GSI with ALL projection duplicates data and writes), and global tables multiply writes per region.
- Local development: DynamoDB Local, LocalStack. Modeling tool: NoSQL Workbench.

---

## 12. When to Use (and Not) {#when}

Use when:
- Access patterns are known and key-based, at any scale, with minimal ops (serverless architectures with Lambda/API Gateway).
- You need predictable low latency with huge or spiky traffic (shopping carts, sessions, user profiles, game state, IoT device state, idempotency keys, rate-limit counters, metadata stores).
- Multi-region active-active with LWW is acceptable.

Avoid when:
- Requirements need flexible querying, joins, reporting, or aggregations (use RDS/Aurora or pair DynamoDB with analytics exports).
- Data is highly relational and evolving rapidly with unknown access patterns.
- You need strong multi-region consistency for every write at low latency.
- Cost modeling shows heavy scans or huge items.

---

## 13. Interview Questions {#qa}

1. Partition key vs sort key? How do you choose a partition key?
2. LSI vs GSI: differences in consistency, creation time, and limits?
3. Explain single-table design with an example. What are its trade-offs?
4. How do you enforce a unique email in DynamoDB?
5. How would you implement optimistic locking and idempotent writes?
6. Why might a Query with `Limit=10` and a FilterExpression return 0 items?
7. How do you handle a hot partition key such as a viral post's like counter?
8. What are DynamoDB Streams used for? How would you maintain a count of orders per customer?
9. On-demand vs provisioned capacity: how do you decide?
10. How do global tables resolve conflicts, and how would you design around it?
