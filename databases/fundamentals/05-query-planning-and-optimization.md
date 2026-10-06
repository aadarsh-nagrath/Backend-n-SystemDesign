# Query Planning, Indexing Strategy, and Optimization

> Engine-agnostic method for making slow queries fast. Engine specifics: [`postgresql/04-explain-and-query-tuning.md`](../postgresql/04-explain-and-query-tuning.md), [`mysql/03-indexes-and-explain.md`](../mysql/03-indexes-and-explain.md). Index basics: [`scaling-db/db-indexing.md`](../../scaling-db/db-indexing.md).

## Table of Contents
1. [Life of a Query](#life)
2. [The Cost-Based Optimizer](#cbo)
3. [Statistics, Selectivity, Cardinality Estimation](#stats)
4. [Access Methods](#access)
5. [Index Design Methodology](#design)
6. [Covering Indexes & Index-Only Scans](#covering)
7. [Partial, Expression, and Multi-Column Indexes](#special)
8. [When Indexes Hurt](#hurt)
9. [A Systematic Slow-Query Workflow](#workflow)
10. [Common Anti-Patterns from Application Code](#anti)
11. [Query Rewrites That Help](#rewrites)
12. [Plan Instability and Prepared Statements](#instability)
13. [Interview Questions](#qa)

---

## 1. Life of a Query {#life}

```
SQL text
  → Parser           (syntax → parse tree)
  → Analyzer/Binder  (resolve tables/columns/types, permissions)
  → Rewriter         (views expanded, rules applied, RLS policies injected)
  → Planner/Optimizer (generate candidate plans, estimate cost, pick cheapest)
  → Executor         (iterator / "Volcano" model: each node pulls rows from children via next())
  → Results streamed to client
```
Prepared statements skip parse/analyze (and possibly planning) on re-execution.

---

## 2. The Cost-Based Optimizer {#cbo}

The optimizer enumerates equivalent plans and estimates each one's **cost** (abstract units ≈ I/O + CPU):
- **Join order**: for N tables there are N! orders (and many tree shapes). Dynamic programming (System R style) for small N; heuristics/genetic search for large N.
- **Join algorithm** per pair: nested loop / hash / merge.
- **Access path** per table: seq scan, index scan, index-only scan, bitmap scan.
- **Aggregation strategy**: hash aggregate vs sort + group aggregate.
- **Parallelism** (PG parallel workers; MySQL 8 has limited parallel read for `COUNT(*)`/CHECK TABLE).

PG cost model parameters (good to know):
| Param | Default | Meaning |
|---|---|---|
| `seq_page_cost` | 1.0 | Sequential page read |
| `random_page_cost` | 4.0 | Random page read — **set to ~1.1 on SSDs**; the default assumes spinning disks and makes the planner avoid index scans too often |
| `cpu_tuple_cost` | 0.01 | Processing a row |
| `cpu_index_tuple_cost` | 0.005 | |
| `cpu_operator_cost` | 0.0025 | Per operator/function |
| `effective_cache_size` | 4 GB | Planner's belief about total cache (shared_buffers + OS cache) |

MySQL has a similar cost model in `mysql.server_cost` / `mysql.engine_cost` tables.

**The optimizer is only as good as its row-count estimates.** Most bad plans come from wrong cardinality estimates, not from bad cost constants.

---

## 3. Statistics and Cardinality {#stats}

Stats collected by `ANALYZE` (PG, auto via autovacuum) / `ANALYZE TABLE` + persistent stats (InnoDB samples pages, `innodb_stats_persistent_sample_pages` = 20 by default — often too low for skewed big tables):
- Row count, page count.
- Per column: null fraction, number of distinct values (n_distinct), **most common values (MCV) + frequencies**, **histogram** of the rest, correlation (physical order vs logical order — affects index scan cost in PG).
- MySQL 8: **histograms** (`ANALYZE TABLE t UPDATE HISTOGRAM ON col WITH 100 BUCKETS`) for non-indexed columns; indexed columns use index dives / cardinality estimates.

**Selectivity** = fraction of rows that satisfy a predicate. `status = 'active'` on a column where 98% are active → selectivity 0.98 → seq scan is correct; `status = 'failed'` (0.1%) → index scan.

Estimation failure modes:
1. **Correlated columns**: `WHERE city = 'Mumbai' AND state = 'Maharashtra'` — optimizer multiplies selectivities assuming independence → massive underestimate. PG fix: `CREATE STATISTICS s (dependencies, ndistinct, mcv) ON city, state FROM addresses;`
2. **Stale stats** after bulk loads → run ANALYZE after large imports.
3. **Skew** not captured in small samples → raise `default_statistics_target` (PG, default 100) per column: `ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000`.
4. **Functions/expressions**: no stats on `lower(email)` unless there's an expression index (PG gathers stats for expression indexes).
5. **Join estimates** compound errors: a 10× error at each of 4 joins → 10,000× error at the top → nested loops over millions of rows.
6. **Parameter-dependent plans** (generic plans for prepared statements).

Reading estimates: in `EXPLAIN ANALYZE`, compare `rows=` (estimated) vs `actual rows=`. **Off by >10× is the thing to fix first.**

---

## 4. Access Methods {#access}

| Method | What happens | When chosen |
|---|---|---|
| **Sequential / full table scan** | Read every page in order | Low selectivity (>~5–20% of rows), small tables, no usable index |
| **Index scan** | Walk index, fetch each matching row from table | High selectivity; or need order matching the index |
| **Index-only scan / covering** | All needed columns in the index; skip table | Covering index exists (PG additionally needs the visibility map to say pages are all-visible) |
| **Bitmap index scan + bitmap heap scan** (PG) | Collect matching TIDs from one or more indexes into a bitmap, sort by page, then read pages in physical order | Medium selectivity; combining indexes (`a = 1 OR b = 2`) |
| **Index merge** (MySQL) | Union/intersection of several index range scans | `OR` across different indexed columns |
| **Loose index scan / skip scan** | Jump between distinct prefix values | `SELECT DISTINCT a` / `GROUP BY a` with index on (a, …); MySQL does it natively; PG needs a recursive CTE trick (until PG 18 skip scan) |
| **Index range scan** (MySQL "range") | | `BETWEEN`, `<`, `IN (…)` |
| **ref / eq_ref / const** (MySQL) | Equality lookup on non-unique / unique / single row | |

Why seq scan sometimes beats index scan: index scans do **random I/O** per row (heap fetch). Fetching 30% of a table via index = way more page reads than reading the whole table sequentially once. The tipping point depends on correlation and caching.

---

## 5. Index Design Methodology {#design}

Design indexes **from queries**, not from tables. For each important query (top N by total time in `pg_stat_statements` / `performance_schema.events_statements_summary_by_digest`):

1. **Equality predicates first** in the index (`WHERE tenant_id = ? AND status = ?`).
2. Then the **range or sort column** (`AND created_at > ?` / `ORDER BY created_at`) — only one range column benefits from seeking.
3. Then extra columns to make it **covering** (optional).

Example:
```sql
SELECT id, total FROM orders
WHERE tenant_id = $1 AND status = 'paid' AND created_at >= $2
ORDER BY created_at DESC LIMIT 50;
```
Ideal: `(tenant_id, status, created_at DESC) INCLUDE (total)` (PG) / `(tenant_id, status, created_at, total)` (MySQL; id is implicitly included as the PK in InnoDB secondaries). The DB seeks to `(tenant, 'paid', now)`, reads 50 entries backward, done — no sort, no table access.

**Column order and selectivity**: the "most selective column first" rule is a myth for equality-only predicates — any order works for equality on all columns. Order matters for: which *subsets* of queries the index can serve (leftmost prefix), and putting range columns last.

**Sort + LIMIT** is where indexes shine most: `ORDER BY x LIMIT 10` with an index on x = read 10 entries. Without it = sort the whole result.

**Index consolidation**: `(a)` is redundant if `(a, b)` exists (mostly — the narrower one is slightly faster/smaller). Find duplicates/unused indexes periodically:
- PG: `pg_stat_user_indexes.idx_scan = 0` (since stats reset, check replicas too!).
- MySQL: `sys.schema_unused_indexes`, `sys.schema_redundant_indexes`.

**Low cardinality columns** (boolean, status with 3 values): a standalone index is rarely useful, except when the queried value is rare (partial index!) or as a leading equality column in a composite.

---

## 6. Covering Indexes {#covering}

A **covering index** contains every column the query needs → no table lookup.
- PG: `CREATE INDEX ON orders (customer_id) INCLUDE (total, status);` — INCLUDE columns are stored only in leaves, not used for searching. Index-only scans in PG need the **visibility map** bits set (VACUUM sets them); on a heavily updated table you'll see `Heap Fetches: N` in EXPLAIN — the scan had to check the heap anyway.
- InnoDB: secondary indexes implicitly contain the PK, so `(customer_id)` already covers `SELECT id FROM orders WHERE customer_id=?`. EXPLAIN shows `Using index`.

Cost: bigger index, more write overhead. Use for the hottest read paths.

---

## 7. Partial, Expression, Multi-Column {#special}

**Partial index** (PG; SQLite; SQL Server "filtered"; MySQL lacks it):
```sql
CREATE INDEX ON jobs (created_at) WHERE status = 'queued';   -- tiny index of only pending jobs
CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL; -- unique among non-deleted (soft delete!)
```
Query must include a predicate implying the index's WHERE.

**Expression / functional index**:
```sql
CREATE INDEX ON users (lower(email));                 -- PG
CREATE INDEX idx ON users ((lower(email)));           -- MySQL 8.0.13+ (double parens)
-- MySQL alternative: generated column + index
ALTER TABLE users ADD email_lc VARCHAR(255) GENERATED ALWAYS AS (lower(email)) STORED, ADD INDEX (email_lc);
```
The query must use the exact same expression.

**Multi-column vs multiple single-column indexes**: two single-column indexes can be combined (bitmap AND / index merge intersect), but a composite is almost always faster for the combined predicate.

**JSON**: PG GIN on `jsonb` (`jsonb_path_ops` for containment `@>`), or B-tree on `(data->>'field')`. MySQL: generated column index or **multi-valued index** for JSON arrays (`CREATE INDEX ON t ((CAST(data->'$.tags' AS CHAR(32) ARRAY)))`).

---

## 8. When Indexes Hurt {#hurt}

Every index:
- Slows every INSERT and every UPDATE that touches its columns (and in PG, non-HOT updates touch *all* indexes).
- Costs disk, buffer pool memory, backup size, replication volume.
- Adds planning time (more candidate plans).
- Can lead the optimizer astray (picks an index scan with a wrong estimate).

Write-heavy tables (event logs, queues) should have the minimum indexes. 10+ indexes on a hot OLTP table is a smell.

---

## 9. Slow-Query Workflow {#workflow}

1. **Find what matters** — by *total time* (calls × mean), not just slowest single query:
   - PG: `pg_stat_statements` (`ORDER BY total_exec_time DESC`), `auto_explain` for plans of slow queries, `log_min_duration_statement`.
   - MySQL: slow query log (`long_query_time`), `pt-query-digest`, `performance_schema` digests, `sys.statements_with_full_table_scans`.
   - APM traces (Datadog, New Relic, OpenTelemetry) to tie queries to endpoints.
2. **Get the real plan with runtime stats**: `EXPLAIN (ANALYZE, BUFFERS)` / `EXPLAIN ANALYZE` (MySQL 8.0.18+) / `EXPLAIN FORMAT=TREE`. Use realistic parameters and production-like data (plans on a 1,000-row dev DB are meaningless).
3. **Read bottom-up / inside-out**; find the node where the time goes (actual time × loops).
4. **Compare estimated vs actual rows**. Big mismatch → stats problem (ANALYZE, extended stats, rewrite).
5. **Look for**: seq scans on big tables with selective filters; nested loops with large outer sides; sorts spilling to disk (`external merge`, `Using filesort` with temp files); `Rows Removed by Filter` huge (index doesn't cover the filter); `Heap Fetches` high on index-only scans; hash joins batching to disk.
6. **Fix in order of cheapness**: rewrite query (sargability, remove needless joins/columns) → add/adjust index → update stats → tune memory (`work_mem`, `sort_buffer_size`) → schema change (denormalize, partition) → caching → bigger hardware.
7. **Verify** with EXPLAIN ANALYZE again and in production metrics (p95/p99 latency), and check write overhead of new indexes.

Create indexes online: PG `CREATE INDEX CONCURRENTLY` (can't run in a transaction; failure leaves an INVALID index to drop); MySQL InnoDB online DDL `ALGORITHM=INPLACE, LOCK=NONE` (default for adding secondary indexes).

---

## 10. Anti-Patterns from Application Code {#anti}

### The N+1 query problem
```python
posts = Post.objects.all()[:50]          # 1 query
for p in posts:
    print(p.author.name)                 # 50 queries
```
Fixes: eager loading (`select_related` / `prefetch_related` in Django, `includes` in Rails, `JOIN FETCH` / `@EntityGraph` in JPA, `include` in Prisma), batching (`WHERE id IN (...)`), DataLoader (GraphQL). Detect with query counters in tests, APM, `nplusone`/`bullet` gems.

### Others
| Anti-pattern | Why bad | Fix |
|---|---|---|
| `SELECT *` | Prevents covering indexes; pulls TOASTed blobs; breaks on schema change | Select needed columns |
| `COUNT(*)` on huge tables for UI | MVCC means exact counts scan (PG) | Estimates (`pg_class.reltuples`), cached counters, "1000+" |
| `OFFSET 100000` | O(offset) | Keyset pagination |
| `ORDER BY RANDOM()` | Sorts the whole table | Random id range, `TABLESAMPLE`, precomputed random column |
| Queries in loops | Round trips dominate (0.5 ms × 1000) | Batch, `IN`, `unnest`/`VALUES` joins, bulk `COPY`/multi-row INSERT |
| Huge IN lists (10k+ items) | Parse/plan cost, plan cache pollution | Temp table / `= ANY($1::bigint[])` (PG) |
| Leading wildcard LIKE | Full scan | Trigram GIN, full-text search, search engine |
| `OR` across columns | May disable indexes | `UNION ALL` of two indexed queries |
| Implicit casts / collation mismatch | Index unused | Match types |
| One giant transaction for batch jobs | Locks, bloat, replication lag | Chunk by PK range (1k–10k rows), commit per chunk |
| Using DB as a queue with polling `SELECT … ORDER BY id LIMIT 1` without SKIP LOCKED | Contention | `FOR UPDATE SKIP LOCKED` or a real queue |
| Fetching then filtering in app | Transfers huge result sets | Push filters/aggregations into SQL |

---

## 11. Useful Rewrites {#rewrites}

- `NOT IN (subquery)` → `NOT EXISTS`.
- Correlated scalar subquery in SELECT list over many rows → `LEFT JOIN` with pre-aggregated derived table, or LATERAL.
- `DISTINCT` used to hide join fan-out → fix the join (EXISTS semi-join).
- `OR` → `UNION ALL` (with care for duplicates).
- Aggregate *before* joining (reduce rows early).
- Top-N per group with small N and an index → LATERAL + LIMIT instead of window function over everything.
- Replace `COUNT(*) > 0` with `EXISTS`.
- Push `LIMIT` into subqueries where semantics allow.
- Big `UPDATE … WHERE` → batched updates by PK range.

---

## 12. Plan Instability and Prepared Statements {#instability}

- Plans can flip after: ANALYZE (new stats), data growth crossing a cost threshold, a version upgrade, a config change, or different parameter values.
- **Prepared statements in PG**: first 5 executions get *custom plans* (planned with actual values); after that PG may switch to a *generic plan* if it's not more expensive on average. For skewed data (one tenant with 90% of rows) the generic plan can be terrible. Control with `plan_cache_mode = force_custom_plan`.
- Tools to pin plans: MySQL optimizer hints (`/*+ INDEX(t idx) */`, `FORCE INDEX`), PG `pg_hint_plan` extension (not core). Prefer fixing stats/indexes over hints — hints rot as data changes.
- Track plan regressions: `auto_explain`, `pg_stat_statements` mean time per queryid over time, MySQL `performance_schema`.

---

## 13. Interview Questions {#qa}

1. A query that was fast yesterday is slow today with no code change. What do you check?
2. Why would the optimizer choose a full table scan when an index exists? Is it always wrong?
3. Design the index for `WHERE a = ? AND b > ? ORDER BY c LIMIT 10`. What are the trade-offs?
4. What is a covering index? Why might a PG index-only scan still touch the heap?
5. Explain cardinality estimation errors with correlated columns and how to fix them.
6. What's the N+1 problem? How do ORMs solve it?
7. Why is OFFSET pagination slow, and how does keyset pagination fix it?
8. What's the cost of having too many indexes?
9. How do you add an index to a 500 GB table in production without downtime?
10. Why does `random_page_cost` matter on SSDs?
