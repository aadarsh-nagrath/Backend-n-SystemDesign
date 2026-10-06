# MySQL Indexes and EXPLAIN

## Table of Contents
1. [Index Types in MySQL](#types)
2. [Leftmost Prefix Rule and Composite Index Design](#leftmost)
3. [Covering Indexes and Index Condition Pushdown](#covering)
4. [Functional, Descending, Invisible, Prefix, Multi-Valued Indexes](#special)
5. [EXPLAIN: Every Column Explained](#explain)
6. [The `type` Column: Access Methods Ranked](#type)
7. [The `Extra` Column: What to Celebrate and What to Fear](#extra)
8. [EXPLAIN ANALYZE and FORMAT=TREE/JSON](#analyze)
9. [Statistics, Histograms, and Index Dives](#stats)
10. [Optimizer Hints and FORCE INDEX](#hints)
11. [Finding Slow Queries: Slow Log, pt-query-digest, performance_schema](#slow)
12. [Worked Examples](#examples)
13. [Interview Questions](#qa)

---

## 1. Index Types {#types}

| Type | Engine | Use |
|---|---|---|
| B+tree (`BTREE`) | InnoDB (default; `USING HASH` is ignored and silently becomes B-tree) | Almost everything |
| Hash | MEMORY engine; InnoDB's *adaptive* hash index is automatic | Equality |
| FULLTEXT | InnoDB (5.6+), MyISAM | `MATCH … AGAINST` natural language / boolean mode; ngram parser for CJK |
| SPATIAL (R-tree) | InnoDB (5.7+) | Geometry with SRID |
| Multi-valued | InnoDB (8.0.17+) | JSON arrays |

Key kinds: PRIMARY (clustered), UNIQUE, plain INDEX/KEY, FULLTEXT, SPATIAL.

---

## 2. Leftmost Prefix Rule {#leftmost}

Index `idx(a, b, c)` can serve:
| Predicate | Uses index? |
|---|---|
| `a = 1` | ✅ (key_len covers a) |
| `a = 1 AND b = 2` | ✅ |
| `a = 1 AND b = 2 AND c = 3` | ✅ full |
| `a = 1 AND c = 3` | ✅ seeks on a; c can be filtered via ICP |
| `b = 2` | ❌ (or **skip scan** in 8.0.13+ if a has few distinct values and the query is covering-ish) |
| `a > 1 AND b = 2` | ✅ range on a; b not used for seeking (ICP may filter) |
| `a = 1 AND b > 2 AND c = 3` | ✅ seeks a, range b; c only filtered |
| `a IN (1,2) AND b = 3` | ✅ multiple ranges (IN on leading column is like several equalities) |
| `ORDER BY a, b` | ✅ no filesort |
| `WHERE a = 1 ORDER BY b` | ✅ no filesort |
| `WHERE a > 1 ORDER BY b` | ❌ filesort (range on a breaks b's order) |
| `ORDER BY a ASC, b DESC` | ✅ only with descending index `(a, b DESC)` (8.0+) |

**`key_len` in EXPLAIN tells you how many index columns were used** (sum of the byte lengths of the used parts; +1 for nullable, +2 for variable length). For example, `INT NOT NULL` = 4, `BIGINT` = 8, `VARCHAR(50) utf8mb4 NOT NULL` = 50×4+2 = 202.

Design rule: **equality columns first, then the one range/sort column, then covering columns.**

---

## 3. Covering Indexes and ICP {#covering}

**Covering index**: all selected and filtered columns are in the index (plus the implicit PK) → EXPLAIN `Extra: Using index` → no clustered index lookup.
```sql
-- idx(customer_id, status, created_at)
SELECT id, status, created_at FROM orders WHERE customer_id = 7;   -- covered (id is the PK)
```

**Index Condition Pushdown (ICP)** (`Using index condition`): when part of the WHERE references index columns that can't be used for seeking, InnoDB evaluates them **on the index entries** before fetching the full row from the clustered index. Fewer back-to-table lookups.
```sql
-- idx(last_name, first_name)
SELECT * FROM people WHERE last_name = 'Shah' AND first_name LIKE '%esh%';
-- Seeks on last_name; first_name LIKE filtered in the index (ICP), only matches fetched from clustered index
```

**Multi-Range Read (MRR)** (`Using MRR`): collects PKs from a secondary index scan, sorts them, then reads the clustered index in PK order, which makes the I/O more sequential.

---

## 4. Special Indexes {#special}

**Functional (expression) indexes** (8.0.13+):
```sql
CREATE INDEX idx_email_lower ON users ((LOWER(email)));
SELECT * FROM users WHERE LOWER(email) = 'a@b.com';     -- must match expression exactly
-- Pre-8.0.13 way: generated column
ALTER TABLE users ADD email_lower VARCHAR(255) AS (LOWER(email)) VIRTUAL, ADD INDEX (email_lower);
```

**Descending indexes** (8.0+; before that `DESC` was parsed and ignored): `INDEX (created_at DESC)`, useful for mixed-order sorts.

**Invisible indexes** (8.0): `ALTER TABLE t ALTER INDEX idx INVISIBLE;` The optimizer ignores it, but it's still maintained. This is the **safe way to test dropping an index**: make it invisible, watch for regressions, then drop it (or flip it back instantly). Test per session with `SET optimizer_switch='use_invisible_indexes=on'`.

**Prefix indexes**: `INDEX (url(100))` indexes the first 100 characters. Smaller, but it can't be covering and can't help ORDER BY. Choose the length by selectivity: `SELECT COUNT(DISTINCT LEFT(url,100))/COUNT(*) FROM t;`

**Multi-valued indexes** for JSON arrays:
```sql
CREATE TABLE products (id BIGINT PRIMARY KEY, attrs JSON,
  INDEX tags_idx ((CAST(attrs->'$.tags' AS CHAR(32) ARRAY))));
SELECT * FROM products WHERE 'eco' MEMBER OF (attrs->'$.tags');
SELECT * FROM products WHERE JSON_CONTAINS(attrs->'$.tags', '["eco","new"]');
SELECT * FROM products WHERE JSON_OVERLAPS(attrs->'$.tags', '["eco","sale"]');
```

**FULLTEXT**:
```sql
ALTER TABLE articles ADD FULLTEXT ft_title_body (title, body);
SELECT id, MATCH(title, body) AGAINST ('postgres replication' IN NATURAL LANGUAGE MODE) AS score
FROM articles WHERE MATCH(title, body) AGAINST ('+postgres -mysql' IN BOOLEAN MODE);
```
Caveats: minimum token size (`innodb_ft_min_token_size` = 3), stopwords, and no relevance tuning. Fine for basic search; use Elasticsearch/OpenSearch for serious search.

---

## 5. EXPLAIN Columns {#explain}

```sql
EXPLAIN SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'paid' AND o.created_at >= '2025-01-01' ORDER BY o.created_at DESC LIMIT 20;
```
| Column | Meaning |
|---|---|
| `id` | SELECT number. Rows with the same id are joined in **top-to-bottom order** (first row = driving table) |
| `select_type` | SIMPLE, PRIMARY, SUBQUERY, DEPENDENT SUBQUERY (⚠ re-executed per outer row), DERIVED (derived table materialized), UNION, MATERIALIZED |
| `table` | Table or alias (`<derived2>`, `<subquery3>`) |
| `partitions` | Partitions accessed (pruning check) |
| `type` | **Access method** (see §6) |
| `possible_keys` | Candidate indexes |
| `key` | Chosen index (NULL = none) |
| `key_len` | Bytes of the index key used (how many columns) |
| `ref` | What's compared to the index (const, column, func) |
| `rows` | **Estimated** rows examined for this table (per row of the previous table in nested loop) |
| `filtered` | Estimated % of rows remaining after the WHERE conditions on this table. Rows × filtered% = rows joined onward |
| `Extra` | Crucial details (see §7) |

Estimated rows examined for a nested-loop join ≈ product of `rows × filtered%` down the list.

---

## 6. `type`: Access Methods Ranked (best → worst) {#type}

| type | Meaning | Example |
|---|---|---|
| `system` | Table has one row | |
| `const` | At most one matching row via PK/unique with constants; read once, treated as a constant | `WHERE id = 5` |
| `eq_ref` | One row per previous-table row via PK/unique NOT NULL | Join on `c.id = o.customer_id` |
| `ref` | Non-unique index equality lookup | `WHERE customer_id = 7` |
| `fulltext` | FULLTEXT index | |
| `ref_or_null` | `ref` + `OR col IS NULL` | |
| `index_merge` | Combines several indexes (union/intersection/sort-union) | `WHERE a = 1 OR b = 2` |
| `unique_subquery` / `index_subquery` | IN-subquery via index | |
| `range` | Index range scan | `BETWEEN`, `>`, `IN (…)`, `LIKE 'x%'` |
| `index` | **Full index scan** (reads whole index; better than ALL only if covering) | `SELECT id FROM t` |
| `ALL` | **Full table scan** | Missing/unusable index |

Seeing `ALL` on a large table, or `index` without `Using index`, usually needs attention.

---

## 7. `Extra` Column {#extra}

Good:
- `Using index`: covering index, no table access.
- `Using index condition`: ICP active.
- `Using where; Using index`: filtering on a covering index.
- `Using index for group-by`: loose index scan (very efficient GROUP BY/DISTINCT).
- `Using index for skip scan`.
- `Select tables optimized away`: answered from index metadata (e.g., `MIN(id)`).

Warning signs:
- **`Using filesort`**: sort not satisfied by index order. It's not necessarily on disk (it can be in memory within `sort_buffer_size`), but on large result sets it's expensive. Combine with LIMIT to see if an index could avoid it.
- **`Using temporary`**: an internal temporary table (GROUP BY on non-indexed columns, DISTINCT + ORDER BY on different columns, UNION, derived tables). In-memory (TempTable engine, `temptable_max_ram`) until it spills to disk.
- `Using join buffer (hash join)` / `(Block Nested Loop)`: no index on the join column. Hash join is OK for big analytical joins but suspicious in OLTP.
- `Using where` with `type = ALL`: scanning everything and filtering.
- `Range checked for each record (index map: …)`: the optimizer re-evaluates per row; usually a bad sign.
- `Impossible WHERE` / `no matching row in const table`: informational.
- `Using MRR`, `Using sort_union(...)`, `Using intersect(...)`: index merge details.
- `FirstMatch`, `LooseScan`, `Start temporary / End temporary` (DuplicateWeedout), `MaterializeScan`: semi-join strategies.

---

## 8. EXPLAIN ANALYZE, TREE, JSON {#analyze}

```sql
EXPLAIN FORMAT=TREE SELECT ...;      -- iterator tree (8.0.16+), with cost and estimated rows
EXPLAIN ANALYZE SELECT ...;          -- 8.0.18+: EXECUTES and shows actual time/rows/loops per iterator
EXPLAIN FORMAT=JSON SELECT ...;      -- detailed costs ("query_cost"), used_columns, attached_condition
EXPLAIN FOR CONNECTION <id>;         -- plan of a running statement in another session
```
`EXPLAIN ANALYZE` output:
```
-> Limit: 20 row(s)  (cost=1520 rows=20) (actual time=0.212..0.640 rows=20 loops=1)
    -> Nested loop inner join  (cost=1520 rows=1340) (actual time=0.210..0.635 rows=20 loops=1)
        -> Filter: (o.status = 'paid')  (cost=650 rows=1340) (actual time=0.180..0.540 rows=20 loops=1)
            -> Index range scan on o using idx_created_at over ('2025-01-01' <= created_at) (reverse) ...
        -> Single-row index lookup on c using PRIMARY (id=o.customer_id)  (actual time=0.004..0.004 rows=1 loops=20)
```
Read it just like PG plans: inner nodes first, time per loop × loops, estimated vs actual rows.

`optimizer_trace` gives the deep "why":
```sql
SET optimizer_trace = 'enabled=on';
SELECT ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G     -- considered plans, costs, chosen path
SET optimizer_trace = 'enabled=off';
```

---

## 9. Statistics, Histograms, Index Dives {#stats}

- **Persistent statistics** (`innodb_stats_persistent=ON`): stored in `mysql.innodb_table_stats` / `innodb_index_stats`, computed by sampling `innodb_stats_persistent_sample_pages` (default 20) pages per index. Auto-recalculated when ~10% of rows change (`innodb_stats_auto_recalc`). For big skewed tables raise the sample: `ALTER TABLE t STATS_SAMPLE_PAGES = 200; ANALYZE TABLE t;`
- `SHOW INDEX FROM t` → `Cardinality` (estimated distinct values).
- **Index dives**: for `range`/`ref` with constants, the optimizer probes the actual B-tree to estimate the rows in the range. Accurate, but costly for huge `IN()` lists, so beyond `eq_range_index_dive_limit` (200) it uses cardinality statistics (less accurate → plan changes when the list size crosses 200!).
- **Histograms** (8.0) for non-indexed columns used in filters:
  ```sql
  ANALYZE TABLE orders UPDATE HISTOGRAM ON status, country WITH 64 BUCKETS;
  SELECT * FROM information_schema.column_statistics;
  ANALYZE TABLE orders DROP HISTOGRAM ON status;
  ```
  Histograms aren't auto-updated (8.4 adds `AUTO UPDATE` option). They help the optimizer estimate `filtered` and choose join order.
- `ANALYZE TABLE t;` refreshes index statistics (quick, sample-based).

---

## 10. Optimizer Hints and FORCE INDEX {#hints}

```sql
SELECT /*+ INDEX(o idx_customer_created) */ ... FROM orders o ...;
SELECT /*+ NO_INDEX(o idx_status) */ ...;
SELECT /*+ JOIN_ORDER(c, o) HASH_JOIN(o) */ ...;
SELECT /*+ MAX_EXECUTION_TIME(2000) */ ...;            -- ms, SELECT only
SELECT /*+ SET_VAR(sort_buffer_size = 16M) */ ...;
SELECT /*+ SKIP_SCAN(t idx) */ ...;
SELECT ... FROM orders FORCE INDEX (idx_customer_created) WHERE ...;   -- older syntax
SELECT ... FROM orders IGNORE INDEX (idx_status) WHERE ...;
```
Hints are a **last resort**: they don't adapt as data changes, and they break if an index is renamed or dropped. Prefer better indexes, histograms, or a query rewrite. **Query rewrite plugin** / ProxySQL rules can inject hints without code changes during an incident.

---

## 11. Finding Slow Queries {#slow}

```ini
slow_query_log = ON
long_query_time = 0.2                  # seconds; 0 logs everything (for short sampling windows)
log_queries_not_using_indexes = OFF    # noisy
log_slow_extra = ON                    # 8.0.14+: rows examined, bytes, temp tables, etc.
min_examined_row_limit = 1000
```
- **`pt-query-digest /var/log/mysql/slow.log`** (Percona Toolkit) aggregates by fingerprint and ranks by total time. It's the classic tool.
- performance_schema statement digests:
  ```sql
  SELECT digest_text, count_star, ROUND(sum_timer_wait/1e12, 2) AS total_s,
         ROUND(avg_timer_wait/1e9, 2) AS avg_ms, sum_rows_examined, sum_rows_sent,
         sum_no_index_used, sum_created_tmp_disk_tables
  FROM performance_schema.events_statements_summary_by_digest
  ORDER BY sum_timer_wait DESC LIMIT 20;
  ```
- `sys.statement_analysis`, `sys.statements_with_full_table_scans`, `sys.statements_with_temp_tables`, `sys.statements_with_sorting`, `sys.schema_tables_with_full_table_scans`.
- **rows_examined / rows_sent ratio** is a great signal. 1,000,000 examined to send 10 means a missing or bad index.
- Tools: PMM (Percona Monitoring and Management), MySQL Enterprise Monitor, Datadog DBM, SolarWinds DPM, mysqld_exporter.

---

## 12. Worked Examples {#examples}

**1. Implicit conversion kills the index**
```sql
-- phone VARCHAR(20) indexed
EXPLAIN SELECT * FROM users WHERE phone = 9876543210;   -- type: ALL (string column compared to number → cast every row)
EXPLAIN SELECT * FROM users WHERE phone = '9876543210'; -- type: ref
```

**2. Charset/collation mismatch in join**
```sql
-- orders.customer_code utf8mb3, customers.code utf8mb4 → index on customers.code unusable in join
-- Fix: ALTER TABLE orders MODIFY customer_code VARCHAR(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

**3. ORDER BY + LIMIT filesort**
```sql
SELECT * FROM orders WHERE customer_id = 7 ORDER BY created_at DESC LIMIT 10;
-- With idx(customer_id): ref + Using filesort (sorts all of customer 7's orders)
-- With idx(customer_id, created_at): ref, backward index scan, no filesort, reads 10 rows
```

**4. OR across columns**
```sql
SELECT * FROM users WHERE email = ? OR phone = ?;
-- index_merge union(idx_email, idx_phone) if both indexed; else ALL
-- Alternative: SELECT ... WHERE email=? UNION SELECT ... WHERE phone=?
```

**5. Dependent subquery (old patterns)**
```sql
SELECT * FROM customers WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);
-- 8.0 converts to a semi-join (FirstMatch/Materialization). If EXPLAIN shows DEPENDENT SUBQUERY, rewrite as JOIN/EXISTS.
```

**6. Deep pagination**
```sql
SELECT * FROM posts ORDER BY id DESC LIMIT 20 OFFSET 500000;  -- reads 500,020 rows
-- "Deferred join" trick: paginate on the covering index, then fetch rows
SELECT p.* FROM posts p JOIN (SELECT id FROM posts ORDER BY id DESC LIMIT 20 OFFSET 500000) x USING (id);
-- Best: keyset pagination WHERE id < :last_id ORDER BY id DESC LIMIT 20
```

**7. COUNT(*)**
InnoDB has no stored row count (MVCC makes it per-transaction), so `COUNT(*)` scans the **smallest secondary index** (parallel in 8.0.14+). For UI, use approximations (`information_schema.tables.table_rows` is a rough estimate) or maintained counters.

---

## 13. Interview Questions {#qa}

1. Explain the leftmost prefix rule with `(a, b, c)`. Which queries can use it?
2. What do `type: ref`, `range`, `index`, and `ALL` mean?
3. What does `Using filesort` mean? Is it always on disk?
4. What is Index Condition Pushdown?
5. How would you safely test dropping an index in production?
6. Why might a query's plan change when an `IN` list grows from 150 to 250 values?
7. How do you find the queries consuming the most total time?
8. Why might `WHERE phone = 9876543210` do a full table scan?
9. Why is `COUNT(*)` slow on large InnoDB tables?
