# Reading EXPLAIN and Tuning Queries in PostgreSQL

## Table of Contents
1. [EXPLAIN Variants](#variants)
2. [Anatomy of a Plan Node](#anatomy)
3. [Scan Nodes](#scans)
4. [Join Nodes](#joins)
5. [Aggregation, Sort, Limit, and Others](#other)
6. [Parallel Query](#parallel)
7. [Worked Examples: Diagnose and Fix](#examples)
8. [pg_stat_statements: Finding What to Tune](#pgss)
9. [auto_explain and Logging](#autoexplain)
10. [Planner Knobs (for diagnosis, not production)](#knobs)
11. [Tuning Checklist](#checklist)

---

## 1. EXPLAIN Variants {#variants}

```sql
EXPLAIN SELECT ...;                         -- estimated plan only, doesn't run
EXPLAIN (ANALYZE) SELECT ...;               -- RUNS the query, shows actual time/rows
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;      -- + shared/local/temp block hits/reads/writes (default ON with ANALYZE in PG 18)
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, WAL, FORMAT TEXT) ...;
EXPLAIN (GENERIC_PLAN) SELECT ... WHERE id = $1;   -- PG 16+: plan with placeholders
EXPLAIN (ANALYZE, SERIALIZE) ...;           -- PG 17+: include cost of converting output (detoasting!)
EXPLAIN (MEMORY) ...;                       -- PG 17+: planner memory
```
**`ANALYZE` executes the statement** — wrap writes: `BEGIN; EXPLAIN ANALYZE UPDATE ...; ROLLBACK;`

Visualizers: explain.depesz.com, explain.dalibo.com (PEV2), pgMustard.

---

## 2. Anatomy of a Plan Node {#anatomy}

```
Index Scan using orders_customer_idx on orders o  (cost=0.43..8.45 rows=10 width=48) (actual time=0.020..0.035 rows=12 loops=1)
  Index Cond: (customer_id = 42)
  Filter: (status = 'paid'::text)
  Rows Removed by Filter: 3
  Buffers: shared hit=5 read=1
```
- `cost=startup..total` — abstract units; startup = before first row (high for Sort, Hash), total = all rows.
- `rows` — **estimated** rows output; `width` — avg bytes per row.
- `actual time=first..last` in ms **per loop**; `rows` actual **per loop**; `loops` — how many times executed. **Total time of a node ≈ actual time × loops.**
- `Index Cond` — used to navigate the index (good). `Filter` — applied after fetching (rows removed = wasted work). `Recheck Cond` — lossy bitmap rechecks.
- `Buffers: shared hit` (from shared_buffers), `read` (from OS/disk), `dirtied`, `written`; `temp read/written` = spill to disk.
- Times of a parent **include** children.

Read plans **inside-out / bottom-up**: leaf scans feed joins feed aggregates feed the top.

---

## 3. Scan Nodes {#scans}

| Node | Meaning | Watch for |
|---|---|---|
| **Seq Scan** | Read all pages | On big tables with selective filter → missing index or non-sargable predicate |
| **Index Scan** | Traverse index, fetch heap per match | Many rows → random I/O; consider bitmap or seq |
| **Index Only Scan** | Data from index only | `Heap Fetches: N` high → vacuum more |
| **Bitmap Index Scan → Bitmap Heap Scan** | Build TID bitmap, read heap in page order | `lossy` blocks when bitmap exceeds `work_mem` → recheck many rows; `BitmapAnd/Or` combine indexes |
| **TID Scan / TID Range Scan** | By ctid | |
| **Function Scan**, **Values Scan**, **CTE Scan**, **Subquery Scan** | | CTE Scan of materialized CTE can block pushdown |

---

## 4. Join Nodes {#joins}

| Node | Notes |
|---|---|
| **Nested Loop** | Outer rows × inner lookups. Great when outer is small and inner is an index scan. Disaster when estimates said 1 outer row but actual is 100,000 (`loops=100000`) |
| **Hash Join** | `Hash` child builds table from inner (smaller) input. `Batches: 1` = in memory; `Batches > 1` = spilled to disk → raise work_mem for that query |
| **Merge Join** | Both sides sorted; check for expensive Sort children |
| **Memoize** (PG 14+) | Caches inner side results for repeated outer keys in nested loops |
| Semi/Anti variants | From EXISTS / NOT EXISTS |

---

## 5. Aggregation, Sort, Limit, Others {#other}

- **Sort**: `Sort Method: quicksort Memory: 25kB` (in memory) vs `external merge Disk: 102400kB` (spilled). `top-N heapsort` with LIMIT. **Incremental Sort** (PG 13+) when input is presorted on a prefix.
- **HashAggregate** vs **GroupAggregate** (sorted input). HashAggregate can spill to disk (PG 13+): `Batches`, `Disk Usage`.
- **Limit**: stops children early — a plan with `Limit → Index Scan` might read only a few rows. But `ORDER BY x LIMIT 10` where the planner picks an index on x and the filter is selective on another column = reads millions of rows to find 10 matches (the "LIMIT trap").
- **Materialize**, **Unique**, **WindowAgg**, **Append** / **Merge Append** (partitions, UNION ALL), **Gather / Gather Merge** (parallel), **Result**, **ProjectSet**, **LockRows** (FOR UPDATE), **ModifyTable** (INSERT/UPDATE/DELETE — check `Trigger` time lines for FK triggers!).
- Partition pruning shows as `Subplans Removed: N` or fewer Append children.

---

## 6. Parallel Query {#parallel}

```
Gather (workers planned: 4, launched: 4)
  -> Parallel Seq Scan on events
```
- Settings: `max_parallel_workers_per_gather` (2), `max_parallel_workers` (8), `max_worker_processes` (8), `parallel_setup_cost`, `parallel_tuple_cost`, `min_parallel_table_scan_size` (8 MB).
- Helps large scans/aggregations/joins; generally irrelevant for OLTP point queries.
- `launched < planned` → worker pool exhausted.
- Not used for: queries that write (except some CREATE TABLE AS / CREATE INDEX), cursors, functions marked PARALLEL UNSAFE, serializable isolation (pre-12).

---

## 7. Worked Examples {#examples}

### Example 1 — missing index
```
Seq Scan on orders  (cost=0.00..180000.00 rows=50 width=64) (actual time=0.01..850.2 rows=47 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 4999953
```
Read 5M rows to return 47. Fix: `CREATE INDEX CONCURRENTLY ON orders (customer_id);` → Index Scan, ~0.05 ms.

### Example 2 — estimate mismatch → nested loop disaster
```
Nested Loop  (rows=1) (actual rows=250000 loops=1)
  -> Seq Scan on users u (rows=1) (actual rows=5000 loops=1)
       Filter: ((country = 'IN') AND (city = 'Mumbai'))
  -> Index Scan on events e (actual rows=50 loops=5000)
```
Planner estimated 1 user (assumed independence of country & city); actually 5000. Fix: `CREATE STATISTICS users_geo (dependencies, mcv) ON country, city FROM users; ANALYZE users;`

### Example 3 — sort spill
```
Sort  (actual time=4200..4900 rows=2000000)
  Sort Key: created_at
  Sort Method: external merge  Disk: 185000kB
```
Fixes: an index on `created_at` to avoid the sort entirely (especially with LIMIT); or `SET LOCAL work_mem = '256MB'` for this report query; or don't sort 2M rows (paginate).

### Example 4 — LIMIT trap
```sql
SELECT * FROM events WHERE user_id = 7 ORDER BY created_at DESC LIMIT 10;
```
```
Limit -> Index Scan Backward using events_created_at_idx on events
           Filter: (user_id = 7)
           Rows Removed by Filter: 3,400,000
```
Planner thought user 7's events are spread uniformly so it'd find 10 quickly walking the time index; user 7 is inactive → scans millions. Fix: composite index `(user_id, created_at DESC)`.

### Example 5 — function on column
```sql
WHERE date(created_at) = '2025-06-01'   -- Seq Scan
WHERE created_at >= '2025-06-01' AND created_at < '2025-06-02'  -- Index Scan
```

### Example 6 — implicit cast/collation
`WHERE external_id = 12345` where `external_id` is `text` → error in PG (no implicit cast int→text in comparisons) — good. But `WHERE uuid_col::text = $1` → no index. Compare native types.

### Example 7 — slow DELETE due to FK
```
Delete on customers (actual time=0.1..0.1)
Trigger for constraint orders_customer_id_fkey: time=4800.000 calls=1
```
The FK check scans `orders` because `orders.customer_id` isn't indexed. Index it.

### Example 8 — `IN` list vs `ANY(array)`
Large generated `IN (1,2,3,...10000)` → long parse/plan and plan cache misses. Use `WHERE id = ANY($1::bigint[])` with one array parameter.

### Example 9 — OR across columns
`WHERE email = $1 OR phone = $2` → BitmapOr of two indexes (fine) or Seq Scan if one column lacks an index.

---

## 8. pg_stat_statements {#pgss}

```sql
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION pg_stat_statements;

-- Top by total time
SELECT queryid, calls, round(total_exec_time::numeric,0) AS total_ms,
       round(mean_exec_time::numeric,2) AS mean_ms, round(stddev_exec_time::numeric,2) AS sd_ms,
       rows, round(100.0*shared_blks_hit/nullif(shared_blks_hit+shared_blks_read,0),1) AS hit_pct,
       left(query, 120)
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
```
- Normalizes queries (constants → `$1`) and aggregates.
- Also look at: highest `mean_exec_time`, highest `calls` (chatty N+1), highest `shared_blks_read` (I/O hogs), `temp_blks_written` (spills), `wal_bytes` (write-heavy).
- Reset periodically / snapshot deltas (`pg_stat_statements_reset()`), or use tools that snapshot (pganalyze, PMM, Datadog DBM, pg_stat_monitor).

---

## 9. auto_explain and Logging {#autoexplain}

```
shared_preload_libraries = 'pg_stat_statements,auto_explain'
auto_explain.log_min_duration = '500ms'
auto_explain.log_analyze = on            # overhead: timing per node — consider log_timing = off
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on  # inside functions
auto_explain.sample_rate = 0.1
log_min_duration_statement = '250ms'
log_lock_waits = on
log_temp_files = 0                        # log every temp file (spills)
log_checkpoints = on
log_autovacuum_min_duration = '1s'
```
Captures the plan **as it happened in production**, with real parameters — invaluable for intermittent slowness.

---

## 10. Planner Knobs for Diagnosis {#knobs}

Session-only experiments to understand why a plan was chosen:
```sql
SET enable_seqscan = off;    -- "what would the index plan cost?"
SET enable_nestloop = off;
SET enable_hashjoin = off;
SET random_page_cost = 1.1;
SET work_mem = '256MB';
SET plan_cache_mode = force_custom_plan;
SET jit = off;
```
If disabling seqscan yields a much faster plan, the planner's cost estimates are off (stats, correlation, `random_page_cost`, `effective_cache_size`). Fix the root cause rather than shipping `enable_*` settings. For persistent hints there's the `pg_hint_plan` extension.

---

## 11. Tuning Checklist {#checklist}

1. Is the query necessary? Can it be cached, batched, or moved to a replica/warehouse?
2. Run `EXPLAIN (ANALYZE, BUFFERS)` with real parameters.
3. Find the most expensive node (time × loops).
4. Estimated vs actual rows — off by 10×+? → ANALYZE, extended stats, statistics target, rewrite.
5. Seq scan with high "Rows Removed by Filter" → index (right column order) or make predicate sargable.
6. Nested loop with huge loops → fix estimates or add index on inner join key.
7. Sort/Hash spilling → index for order, or raise `work_mem` for that query/role.
8. Index-only scans with heap fetches → vacuum.
9. Long `Trigger for constraint` times → index FK columns.
10. Planning time high (`Planning Time:` line) → too many partitions, huge IN lists, complex views; use prepared statements.
11. Re-test and monitor p95/p99 after deploying.
