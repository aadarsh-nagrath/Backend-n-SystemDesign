# PostgreSQL Partitioning

> Partitioning splits one logical table into many physical tables on **one server**. Sharding spreads data across many servers (see [`scaling-db/sharding.md`](../../scaling-db/sharding.md)). Citus combines both.

## Table of Contents
1. [Why Partition (and Why Not)](#why)
2. [Declarative Partitioning: Range, List, Hash](#types)
3. [Partition Pruning](#pruning)
4. [Indexes, Constraints, and Keys on Partitioned Tables](#indexes)
5. [Managing Partitions: Create, Attach, Detach, Drop](#manage)
6. [Automation: pg_partman, pg_cron](#automation)
7. [Migrating an Existing Big Table to Partitioned](#migrate)
8. [Limitations and Pitfalls](#pitfalls)
9. [Partitioning vs Sharding vs Citus](#citus)
10. [Interview Questions](#qa)

---

## 1. Why Partition {#why}

Good reasons:
1. **Data lifecycle / retention**: drop a month of data instantly with `DROP TABLE events_2024_01` instead of a multi-hour `DELETE` that creates massive bloat and WAL.
2. **Query performance via pruning**, when most queries filter on the partition key (recent data), so only relevant partitions are scanned.
3. **Maintenance at smaller granularity**: VACUUM, REINDEX, and backups per partition; old partitions become frozen and static.
4. **Hot/cold separation**: recent partitions on fast storage, old ones on a cheaper tablespace or compressed or archived to S3.
5. **Index sizes**: each partition's indexes are smaller, so the hot ones stay in cache.

Bad reasons / costs:
- Queries that **don't** filter on the partition key now scan *every* partition (slower than one big table).
- More planning overhead, and more locks (each partition and index is a relation).
- Unique constraints must include the partition key.
- Operational complexity.

Rule of thumb: consider partitioning when a table is approaching the size of RAM / hundreds of GB, **or** when you have time-based retention requirements.

---

## 2. Declarative Partitioning {#types}

### Range (most common: time)
```sql
CREATE TABLE events (
  id bigint GENERATED ALWAYS AS IDENTITY,
  tenant_id bigint NOT NULL,
  created_at timestamptz NOT NULL,
  payload jsonb,
  PRIMARY KEY (id, created_at)                 -- must include partition key
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_06 PARTITION OF events
  FOR VALUES FROM ('2025-06-01') TO ('2025-07-01');     -- upper bound exclusive
CREATE TABLE events_2025_07 PARTITION OF events
  FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
CREATE TABLE events_default PARTITION OF events DEFAULT; -- catches rows matching no partition
```

### List (discrete values: region, tenant tier, status)
```sql
CREATE TABLE customers (...) PARTITION BY LIST (region);
CREATE TABLE customers_in PARTITION OF customers FOR VALUES IN ('IN');
CREATE TABLE customers_eu PARTITION OF customers FOR VALUES IN ('DE','FR','NL');
```

### Hash (even spread, no natural range)
```sql
CREATE TABLE sessions (...) PARTITION BY HASH (user_id);
CREATE TABLE sessions_p0 PARTITION OF sessions FOR VALUES WITH (MODULUS 8, REMAINDER 0);
-- ... p1..p7
```
Hash partitioning helps spread write contention and VACUUM, but provides no retention benefit, and changing the modulus means repartitioning.

### Multi-level (sub-partitioning)
```sql
CREATE TABLE events_2025_06 PARTITION OF events FOR VALUES FROM ('2025-06-01') TO ('2025-07-01')
  PARTITION BY HASH (tenant_id);
```
Use sparingly, because partition count multiplies.

Old-style **inheritance partitioning** (pre-PG 10, with triggers and CHECK constraints) is legacy. Use declarative.

---

## 3. Partition Pruning {#pruning}

The planner/executor skips partitions that can't contain matching rows.
- **Plan-time pruning**: with constants in WHERE (`created_at >= '2025-06-01'`).
- **Execution-time pruning** (PG 11+): with parameters (prepared statements) or values from subqueries/joins (`Subplans Removed: N` in EXPLAIN).
- Requires `enable_partition_pruning = on` (default).
- The WHERE clause must constrain the **partition key** directly. A function on the key (`date(created_at) = …`) can defeat pruning.

```sql
EXPLAIN SELECT count(*) FROM events WHERE created_at >= '2025-07-01' AND created_at < '2025-07-08';
-- Aggregate -> Seq Scan on events_2025_07   (only one partition)
```

**Partition-wise join/aggregate** (`enable_partitionwise_join`, `enable_partitionwise_aggregate`, both off by default because of planning cost): when two tables are partitioned identically, they can be joined partition by partition.

---

## 4. Indexes, Constraints, Keys {#indexes}

- `CREATE INDEX ON events (tenant_id, created_at);` on the parent creates a matching index on every partition (existing and future). Each partition has its own physical index, so there's **no global index** in Postgres.
- **PRIMARY KEY / UNIQUE must include all partition key columns**, because uniqueness is only enforced per partition. "id unique across all partitions" isn't possible unless id is in the key. Workarounds: rely on identity/UUID uniqueness by construction, or keep a separate non-partitioned lookup table with a UNIQUE constraint.
- Foreign keys: partitioned tables can reference other tables, and since PG 12 can be referenced by FKs.
- Lookups by `id` alone (without `created_at`) must probe the index of every partition. If you need fast lookup by id, include the time in the ID (UUIDv7/Snowflake IDs let you derive the partition from the id) or keep a lookup table.

---

## 5. Managing Partitions {#manage}

```sql
-- Create ahead of time (never let inserts hit a missing range → error or DEFAULT partition)
CREATE TABLE events_2025_08 PARTITION OF events FOR VALUES FROM ('2025-08-01') TO ('2025-09-01');

-- Attach an existing table (bulk-load it separately first, then attach)
CREATE TABLE events_2025_09 (LIKE events INCLUDING DEFAULTS INCLUDING CONSTRAINTS);
-- add a CHECK matching the bound so ATTACH can skip the validation scan:
ALTER TABLE events_2025_09 ADD CONSTRAINT chk CHECK (created_at >= '2025-09-01' AND created_at < '2025-10-01');
ALTER TABLE events ATTACH PARTITION events_2025_09 FOR VALUES FROM ('2025-09-01') TO ('2025-10-01');

-- Retention: detach without blocking queries (PG 14+), then archive or drop
ALTER TABLE events DETACH PARTITION events_2024_01 CONCURRENTLY;
-- pg_dump -t events_2024_01 | gzip > s3://archive/...   (or COPY TO parquet via extension)
DROP TABLE events_2024_01;
```

**DEFAULT partition caveat**: when you attach or create a new partition, Postgres must scan the DEFAULT partition to verify no rows belong to the new range (holding a lock). A large default partition makes adding partitions slow. Keep it empty and alert if rows land there.

PG 17 added `ALTER TABLE … SPLIT PARTITION` / `MERGE PARTITIONS`, but they were **reverted before release**. Don't rely on them.

---

## 6. Automation {#automation}

**pg_partman** (extension) creates future partitions and drops/detaches old ones on a schedule:
```sql
CREATE EXTENSION pg_partman;
SELECT partman.create_parent(
  p_parent_table => 'public.events',
  p_control      => 'created_at',
  p_interval     => '1 month',
  p_premake      => 4           -- keep 4 future partitions ready
);
UPDATE partman.part_config SET retention = '12 months', retention_keep_table = false
WHERE parent_table = 'public.events';
-- run maintenance periodically (pg_partman_bgw background worker or pg_cron):
SELECT cron.schedule('partman', '0 * * * *', $$CALL partman.run_maintenance_proc()$$);
```
TimescaleDB **hypertables** automate time partitioning ("chunks") plus compression, retention, and continuous aggregates.

---

## 7. Migrating an Existing Big Table {#migrate}

You can't `ALTER TABLE … PARTITION BY` an existing table. Options:

1. **Attach the old table as the first partition**: create the new partitioned parent, rename the old table, and attach it as the "everything before now" partition (add a CHECK constraint first to avoid a long scan). New data goes to new partitions, and old data ages out. This is fast, with minimal downtime.
2. **Copy + dual write**: create the partitioned table, use a trigger or app dual-write for new rows, backfill old rows in batches, then swap names in a short transaction.
3. **Logical replication** into a new partitioned table on the same or another cluster, then switch over.

---

## 8. Limitations and Pitfalls {#pitfalls}

- Too many partitions (thousands) cause planning-time explosions, lock manager contention, memory use per backend (relcache), and slower DDL. Aim for **dozens to low hundreds** of partitions touched per query. Daily partitions × years × sub-partitions is a classic mistake.
- Too-small partitions add overhead without benefit. Size them so the hot partitions' indexes fit in memory.
- Rows that update their partition key move between partitions (DELETE + INSERT internally, supported since PG 11). Concurrent updates may error.
- `ON CONFLICT` on a partitioned table needs a unique index that includes the partition key.
- Some statements still lock all partitions when pruning can't happen at plan time (generic plans can lock everything at executor startup — improved in recent versions).
- Global ordering queries (`ORDER BY created_at DESC LIMIT 20` across partitions) use **Merge Append**, which works well with per-partition indexes.
- Logical replication of partitioned tables: `publish_via_partition_root = true` to replicate as the parent.

---

## 9. Partitioning vs Sharding vs Citus {#citus}

| | Partitioning | Sharding (app-level/Vitess-like) | Citus |
|---|---|---|---|
| Nodes | 1 | Many | Many (coordinator + workers) |
| Transparent to app | Yes | Usually needs shard-aware routing | Mostly (distributed tables via `create_distributed_table('events','tenant_id')`) |
| Scales writes beyond one machine | No | Yes | Yes |
| Cross-shard joins/transactions | N/A | Hard | Supported (co-located joins fast; distributed 2PC) |
| Typical key | Time | tenant/user id | tenant id (multi-tenant SaaS) or time (real-time analytics) |

Citus (open source, Microsoft) also offers reference tables (replicated to all nodes), columnar storage, and schema-based sharding (Citus 12+).

---

## 10. Interview Questions {#qa}

1. When would you partition a table, and on what key?
2. Why must a primary key on a partitioned table include the partition key?
3. How does partition pruning work with prepared statements?
4. How do you implement 90-day retention on a 5 TB events table without bloat?
5. What's the risk of a DEFAULT partition?
6. Partitioning vs sharding: what problems does each solve?
7. How do you convert an existing 1 TB table to a partitioned table with minimal downtime?
