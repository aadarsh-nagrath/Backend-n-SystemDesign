# PostgreSQL Indexes: Every Type, When to Use It, and How to Maintain It

> General index theory: [`fundamentals/05-query-planning-and-optimization.md`](../fundamentals/05-query-planning-and-optimization.md) and [`scaling-db/db-indexing.md`](../../scaling-db/db-indexing.md). Here: Postgres's index access methods and features.

## Table of Contents
1. [Index Types Overview](#overview)
2. [B-tree (default)](#btree)
3. [Hash](#hash)
4. [GIN](#gin)
5. [GiST](#gist)
6. [SP-GiST](#spgist)
7. [BRIN](#brin)
8. [Extension Indexes: pg_trgm, bloom, pgvector (HNSW/IVFFlat)](#ext)
9. [Index Features: Partial, Expression, INCLUDE, Unique, NULLS NOT DISTINCT](#features)
10. [Operator Classes and Collations](#opclass)
11. [Building Indexes Safely](#building)
12. [Maintaining and Auditing Indexes](#maintenance)
13. [Decision Table](#decision)
14. [Interview Questions](#qa)

---

## 1. Overview {#overview}

| Type | Good for | Operators |
|---|---|---|
| **B-tree** | Equality, ranges, sorting, prefix LIKE, uniqueness | `< <= = >= >`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'abc%'` (with right opclass) |
| **Hash** | Equality only | `=` |
| **GIN** (Generalized Inverted) | Values containing many elements: arrays, JSONB, full-text, trigrams | `@>`, `<@`, `&&`, `?`, `?|`, `?&`, `@@`, `%` (trgm), `LIKE '%x%'` (trgm) |
| **GiST** (Generalized Search Tree) | Geometric, ranges, nearest-neighbor, exclusion constraints, full-text (lossy) | `&&`, `@>`, `<->` (distance), `<<`, `-|-` |
| **SP-GiST** (Space-Partitioned) | Non-balanced partitioning: quadtrees, k-d trees, radix tries | Points, IP ranges (`inet`), text prefixes |
| **BRIN** (Block Range) | Huge tables where column correlates with physical order | `< <= = >= >` (minmax), bloom, multi-minmax |

```sql
CREATE INDEX [CONCURRENTLY] [IF NOT EXISTS] name ON table USING method (columns/expressions)
  [INCLUDE (cols)] [WITH (storage_params)] [WHERE predicate];
```

---

## 2. B-tree {#btree}

The default and right answer ~90% of the time.
- Implementation: Lehman–Yao B-link tree with high-key and right-links (concurrent splits without blocking readers).
- **Deduplication** (PG 13+): duplicate keys stored once with a posting list of TIDs → indexes on low-cardinality columns shrink 2–3×.
- **Bottom-up deletion** (PG 14+): removes obsolete version entries from non-HOT updates before splitting.
- **Skip scan** (PG 18): can use an index on `(a, b)` for `WHERE b = ?` when `a` has few distinct values.
- Max entry size ~1/3 of a page (~2.7 KB) → indexing big text fails; index a hash (`md5(col)`) or prefix instead.
- Multi-column: up to 32 columns. Column order matters (leftmost prefix).
- **Sort direction**: `(a ASC, b DESC)` matters only for mixed-direction ORDER BY (`ORDER BY a, b DESC`); a single-direction index can be scanned backward.
- NULLs: indexed (unlike Oracle), so `IS NULL` can use the index. `NULLS FIRST/LAST` per column.

```sql
CREATE INDEX orders_customer_created_idx ON orders (customer_id, created_at DESC);
```

---

## 3. Hash {#hash}

- Equality only; stores 4-byte hash codes → smaller than B-tree for long keys (URLs, long strings).
- WAL-logged and crash-safe since PG 10 (before that: never use).
- No uniqueness, no multi-column, no sorting, no index-only scans.
- Rarely worth it over B-tree; consider for very long equality-only keys.

---

## 4. GIN {#gin}

Inverted index: maps each **element** (array item, JSONB key/value, lexeme, trigram) to a **posting list/tree** of TIDs.

```sql
-- Arrays
CREATE INDEX ON posts USING gin (tags);
SELECT * FROM posts WHERE tags @> ARRAY['postgres'];      -- contains
SELECT * FROM posts WHERE tags && ARRAY['pg','mysql'];    -- overlaps

-- JSONB (default jsonb_ops: supports ?, ?|, ?&, @>, @?, @@)
CREATE INDEX ON events USING gin (payload);
-- jsonb_path_ops: only @>, @?, @@ but smaller & faster
CREATE INDEX ON events USING gin (payload jsonb_path_ops);
SELECT * FROM events WHERE payload @> '{"type":"signup","plan":"pro"}';

-- Full-text
CREATE INDEX ON articles USING gin (to_tsvector('english', title || ' ' || body));
-- or a generated column:
ALTER TABLE articles ADD COLUMN tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) STORED;
CREATE INDEX ON articles USING gin (tsv);
SELECT * FROM articles WHERE tsv @@ websearch_to_tsquery('english', 'postgres -mysql "index types"');
```

Write cost: inserting one row may add many index entries. GIN's **fastupdate** (pending list) buffers new entries and merges them later (on VACUUM or when `gin_pending_list_limit` = 4 MB fills) → fast inserts but occasional slow merges and slower searches while the pending list is large. Disable (`WITH (fastupdate = off)`) for predictable latency.

**Common mistake**: `WHERE payload->>'type' = 'signup'` does **not** use a GIN index on `payload`. Either use containment (`payload @> '{"type":"signup"}'`) or a B-tree expression index `ON events ((payload->>'type'))`.

---

## 5. GiST {#gist}

A balanced tree framework where each internal node stores a "predicate" covering children (e.g., bounding boxes). Lossy → recheck on heap.

Uses:
- **PostGIS** geometry/geography (R-tree-like), `ST_DWithin`, `ST_Intersects`.
- **Range types**: `tstzrange`, `int4range` overlaps (`&&`), containment.
- **KNN / nearest neighbor**: `ORDER BY location <-> point(…) LIMIT 10` — index-assisted ordering by distance.
- **Exclusion constraints**:
  ```sql
  CREATE EXTENSION btree_gist;   -- lets GiST handle scalar = too
  CREATE TABLE bookings (
    room_id int,
    during tstzrange,
    EXCLUDE USING gist (room_id WITH =, during WITH &&)
  );
  -- Now overlapping bookings for the same room are impossible, race-free.
  ```
- `pg_trgm` similarity (`gist_trgm_ops`) — supports `<->` distance ordering (GIN can't).
- Full-text (smaller and faster to update than GIN but slower to search; lossy).

---

## 6. SP-GiST {#spgist}

Space-partitioned (non-overlapping) structures: quad-trees for points, k-d trees, radix trees for text, and `inet`/`cidr` (`inet_ops`). Good for data with natural clustering and many non-overlapping partitions; e.g., IP range lookups `WHERE ip << '10.0.0.0/8'`, phone prefix lookups.

---

## 7. BRIN {#brin}

Stores a **summary per block range** (default 128 pages): min/max of the column. A query checks which ranges *might* contain matches, then scans those pages.
- Tiny: a BRIN index on a 1 TB table can be a few MB (vs. tens of GB for B-tree).
- Only useful when the column's values **correlate with physical row order** — append-only timestamps, serial IDs. Check `pg_stats.correlation` (close to ±1 is good).
- Lossy: always re-checks rows in matching ranges. Great for "last 7 days" over years of logs; bad for point lookups.
- Summarization of new ranges: autovacuum or `autosummarize = on`, or `brin_summarize_new_values()`.
- PG 14+: `minmax_multi_ops` (handles outliers/some disorder) and `bloom_ops` (equality on non-correlated data).

```sql
CREATE INDEX ON logs USING brin (created_at) WITH (pages_per_range = 32);
```

---

## 8. Extension Indexes {#ext}

### pg_trgm (trigram) — fuzzy and infix search
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX ON users USING gin (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%ram%';           -- infix LIKE uses the index!
SELECT * FROM users WHERE name % 'Aadarsh';             -- similarity > pg_trgm.similarity_threshold (0.3)
SELECT name, similarity(name, 'Adarsh') FROM users ORDER BY name <-> 'Adarsh' LIMIT 5;  -- needs GiST for ordering
```
Patterns shorter than 3 chars can't use trigrams effectively.

### bloom
Multi-column equality on many columns where any subset might be queried: `CREATE INDEX ON t USING bloom (a, b, c, d) WITH (length=80, col1=2, …);`. Lossy, small.

### btree_gin / btree_gist
Let GIN/GiST index scalar types — for combining scalar equality with array/range conditions in one index or exclusion constraint.

### pgvector — embeddings
```sql
CREATE EXTENSION vector;
CREATE TABLE docs (id bigserial PRIMARY KEY, content text, embedding vector(1536));

-- HNSW: better recall/speed trade-off, slower build, more memory; no training needed
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
SET hnsw.ef_search = 100;   -- query-time recall knob

-- IVFFlat: faster build, needs data present to train lists; lower recall
CREATE INDEX ON docs USING ivfflat (embedding vector_l2_ops) WITH (lists = 1000);
SET ivfflat.probes = 10;

SELECT id FROM docs ORDER BY embedding <=> $1 LIMIT 10;   -- <-> L2, <=> cosine distance, <#> negative inner product
```
Approximate: may miss true neighbors; filtering (`WHERE tenant_id = …`) combined with ANN is tricky — post-filtering can return fewer than LIMIT rows (pgvector 0.8 added iterative index scans). Also `halfvec` (16-bit), `sparsevec`, binary quantization.

---

## 9. Index Features {#features}

### Partial indexes
```sql
CREATE INDEX ON orders (created_at) WHERE status = 'pending';      -- tiny if few pending
CREATE UNIQUE INDEX ON users (lower(email)) WHERE deleted_at IS NULL;
CREATE INDEX ON jobs (run_at) WHERE finished_at IS NULL;
```
The query's WHERE must logically imply the index predicate (planner does simple implication proofs; with parameters in prepared statements it can fail to prove — use literal values for the partial condition).

### Expression indexes
```sql
CREATE INDEX ON users (lower(email));
CREATE INDEX ON events ((payload->>'user_id'));
CREATE INDEX ON orders (date_trunc('day', created_at));   -- only IMMUTABLE functions allowed
```
`date_trunc` on `timestamptz` is only STABLE (depends on TimeZone) → not allowed; use `(created_at AT TIME ZONE 'UTC')::date` or a generated column.

### Covering (INCLUDE)
```sql
CREATE INDEX ON orders (customer_id, created_at) INCLUDE (status, total);
```
INCLUDE columns: not part of the search key, not ordered, no opclass needed, don't count toward uniqueness.

### Unique and constraints
```sql
CREATE UNIQUE INDEX CONCURRENTLY users_email_key ON users (email);
ALTER TABLE users ADD CONSTRAINT users_email_key UNIQUE USING INDEX users_email_key;  -- attach without rebuild
-- NULL handling (PG 15+)
CREATE UNIQUE INDEX ON t (a, b) NULLS NOT DISTINCT;
```
Deferrable unique constraints (`DEFERRABLE INITIALLY DEFERRED`) — checked at commit; needed for e.g. swapping positions `UPDATE t SET pos = pos + 1`. Can't be used as ON CONFLICT arbiters.

---

## 10. Operator Classes & Collations {#opclass}

An **operator class** tells an index method how to handle a type.
- `text_pattern_ops` / `varchar_pattern_ops`: B-tree for `LIKE 'prefix%'` under non-C collations (default collation index doesn't support LIKE prefix matching).
  ```sql
  CREATE INDEX ON products (sku text_pattern_ops);
  ```
- `jsonb_path_ops` vs `jsonb_ops` (GIN).
- `gin_trgm_ops`, `gist_trgm_ops`.
- `vector_l2_ops`, `vector_cosine_ops`, `vector_ip_ops`.

Collation: an index is built with a collation; a query with a different collation can't use it. Collation library upgrades (glibc 2.28!) can change sort order → **corrupted indexes** (wrong results, unique violations missed). Mitigate: ICU collations with versioning, or `C`/builtin `C.UTF-8` (PG 17) for identifiers; after OS upgrades check `pg_collation_actual_version()` and `REINDEX`. Use `amcheck` (`bt_index_check`) to detect corruption.

---

## 11. Building Indexes Safely {#building}

```sql
SET maintenance_work_mem = '2GB';
SET max_parallel_maintenance_workers = 4;          -- parallel B-tree builds (PG 11+)
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders (customer_id);
```
- Normal `CREATE INDEX` takes a **SHARE lock**: blocks INSERT/UPDATE/DELETE for the whole build. Fine for small tables or maintenance windows only.
- `CONCURRENTLY`: takes SHARE UPDATE EXCLUSIVE (writes continue), does two table scans and waits for all transactions that might see the old state to finish. Slower (2–3×), can't run inside a transaction block, and if it fails leaves an **INVALID** index (still maintained on writes, unused for reads!) → `DROP INDEX CONCURRENTLY` and retry.
  ```sql
  SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
  ```
- Progress: `SELECT * FROM pg_stat_progress_create_index;`
- Partitioned tables: `CREATE INDEX CONCURRENTLY` isn't supported on the parent directly — create `ON ONLY parent` (invalid), then concurrently on each partition, then `ALTER INDEX parent_idx ATTACH PARTITION child_idx`.
- Rebuild: `REINDEX INDEX CONCURRENTLY idx;` (PG 12+).

---

## 12. Maintaining and Auditing {#maintenance}

```sql
-- Unused indexes (check on primary AND replicas — replicas have their own stats)
SELECT s.relname, s.indexrelname, s.idx_scan, pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s JOIN pg_index i USING (indexrelid)
WHERE s.idx_scan = 0 AND NOT i.indisunique AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;

-- Duplicate indexes (same columns)
SELECT pg_size_pretty(sum(pg_relation_size(idx))::bigint), array_agg(idx)
FROM (SELECT indexrelid::regclass idx, (indrelid::text||E'\n'||indclass::text||E'\n'||indkey::text||E'\n'||coalesce(indexprs::text,'')||E'\n'||coalesce(indpred::text,'')) AS key FROM pg_index) sub
GROUP BY key HAVING count(*) > 1;

-- Index size vs table size
SELECT relname, pg_size_pretty(pg_table_size(relid)) table_sz, pg_size_pretty(pg_indexes_size(relid)) idx_sz
FROM pg_stat_user_tables ORDER BY pg_indexes_size(relid) DESC LIMIT 20;

-- Tables with lots of seq scans on large size (missing index candidates)
SELECT relname, seq_scan, seq_tup_read, idx_scan, n_live_tup
FROM pg_stat_user_tables WHERE n_live_tup > 100000 ORDER BY seq_tup_read DESC LIMIT 20;

-- Missing FK indexes
SELECT c.conrelid::regclass AS tbl, c.conname, a.attname
FROM pg_constraint c JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
AND NOT EXISTS (SELECT 1 FROM pg_index i WHERE i.indrelid = c.conrelid AND (i.indkey::int2[])[0] = c.conkey[1]);
```
Also: `hypopg` extension creates **hypothetical indexes** so you can see in EXPLAIN whether the planner *would* use an index without building it.

Stats reset: `pg_stat_reset()`; know when stats were last reset (`pg_stat_database.stats_reset`) before concluding an index is unused.

---

## 13. Decision Table {#decision}

| Query pattern | Index |
|---|---|
| `WHERE id = ?`, ranges, ORDER BY | B-tree |
| `WHERE lower(email) = ?` | B-tree expression |
| `WHERE status = 'pending'` (rare value) | Partial B-tree |
| `WHERE tags @> ARRAY[...]` | GIN |
| `WHERE payload @> '{"k":"v"}'` | GIN `jsonb_path_ops` |
| `WHERE payload->>'k' = ?` | B-tree expression |
| `WHERE name ILIKE '%foo%'` | GIN `gin_trgm_ops` |
| Full-text search | GIN on tsvector |
| Fuzzy "did you mean" ranking | GiST trgm `<->` |
| Geospatial within radius / nearest | GiST (PostGIS) |
| No overlapping ranges | EXCLUDE USING gist |
| Time-series range on append-only table (TB-scale) | BRIN |
| Embedding similarity | pgvector HNSW |
| IP containment | GiST/SP-GiST `inet_ops` |

---

## 14. Interview Questions {#qa}

1. B-tree vs GIN vs GiST vs BRIN — one use case each.
2. Why doesn't `WHERE data->>'type' = 'x'` use a GIN index on `data`?
3. How do you make `LIKE '%term%'` fast in Postgres?
4. What happens if `CREATE INDEX CONCURRENTLY` fails halfway?
5. How do you enforce "no two bookings for the same room overlap"?
6. When is BRIN a great choice and when is it useless?
7. Why can an OS upgrade corrupt Postgres indexes?
8. How would you find unused indexes safely?
