# Database Indexing

An index is a separate data structure that points to actual rows in a table, letting the database find data without scanning everything. It speeds up reads but costs on writes — the central trade-off of this whole topic.

## TL;DR
- An index is a sorted structure (usually a B-tree) mapping column values to row locations, stored alongside the table and auto-updated on every INSERT/UPDATE/DELETE.
- Turns query cost from O(n) full scan into O(log n) lookup.
- Cost: slower writes (every DML updates the index too) and extra storage (composite indexes can be 50%+ of table size).
- B-tree is the default and handles both equality and range queries; hash indexes are equality-only but O(1).
- Composite index column order matters — most selective / most-filtered column first.
- A covering index includes all columns a query needs, avoiding the extra "heap fetch" back to the table.
- The optimizer can ignore an index it has (low selectivity, leading wildcard `LIKE '%x%'`, function wrapping the column, stale statistics).

## What is a database index?

**Analogy**: a library without a catalog means checking every shelf sequentially. With a catalog (index), you look up the title and go straight to the shelf. Same idea as a book's index, or a phone book's A-Z tabs — skip straight to "Z" instead of reading the whole book.

Technically: tables store rows/columns, each row has an identifier. An index stores a sorted copy of one or more columns' values plus pointers back to the full rows (the row storage is often called the "heap"). Indexes live separately from table data, on disk or in memory, and are automatically maintained on every write — which is exactly where the cost comes from.

**Clustered vs. non-clustered** (terminology common in SQL Server/MySQL):
- **Clustered**: the table's actual data is physically sorted by the index key. Only one per table (usually the primary key) — since data can only be physically sorted one way. Lookups are fast because the data is right there.
- **Non-clustered**: a separate structure pointing back to the data. You can have many, but each lookup may need an extra "bookmark lookup" to fetch the full row.

Example: a `users` table with `id` as primary key (clustered) has data physically sorted by `id`. A non-clustered index on `email` sorts emails separately and points back to the matching `id`.

## Why indexes matter

Without an index, a query is a **full table scan** — every row gets checked. At scale this is disastrous: a Library-of-Congress-sized table (170M items) is unsearchable in any reasonable time without one.

Concrete numbers: on an 11-million-row `employees` table, querying by an indexed `id` takes ~0.1ms; querying by an unindexed `name` takes ~3 seconds. After indexing `name`, the same query drops to ~47ms (bitmap index scan).

Benefits:
- **Speed**: O(n) linear scan → O(log n) or better with B-trees.
- **Joins, sorts, filters**: indexes accelerate `WHERE`, `ORDER BY`, `JOIN`.
- **Scalability**: without indexes, databases grind to a halt as data grows.
- **Constraints**: indexes also back unique constraints (prevent duplicates) and full-text search.

## How indexes work: data structures

| Structure | Best for | Notes |
|---|---|---|
| **B-tree** | Range queries (`BETWEEN`, `>`, `<`) and equality | Default in Postgres/MySQL. Self-balancing tree; root narrows down logarithmically to a leaf. Search for ID=2000: "Is it <1000? No. <4000? Yes." — down to the leaf. Slower for pure exact-match than a hash index. |
| **Hash** | Exact match (`=`) only | O(1) lookup via hash function, but useless for ranges. Analogy: a dictionary where words hash straight to a page number. |
| **Bitmap** | Low-cardinality columns (few unique values, e.g. gender) | Uses 0/1 bitmaps per value; strong for analytics/COUNT queries. |
| **LSM tree** (Log-Structured Merge-Tree) | Write-heavy workloads | Used in Cassandra and other NoSQL stores. Writes go to memory, flush to disk as sorted segments, merge later. High write throughput; reads may need to merge across segments. |
| **GiST / GIN** | Geometric data / arrays & JSON | Postgres-specific specialized index types. |
| **Full-text** | Search engines | Elasticsearch-style tokenized text search. |

Postgres uses B-tree by default for primary keys.

## Types of indexes

- **Primary index**: on the primary key (unique, non-null), auto-created, often clustered.
- **Unique index**: enforces uniqueness like a primary key, but allows nulls.
- **Composite/multi-column index**: on multiple columns, e.g. `CREATE INDEX ON users (last_name, first_name)`. Good for queries filtering on both columns together. **Column order matters — put the most selective/most-filtered column first.**
- **Partial index**: only indexes rows meeting a condition (e.g. `WHERE status='active'`) — saves space.
- **Functional index**: indexes an expression, e.g. `LOWER(email)`, for case-insensitive search.
- **Covering index**: includes every column a query needs directly in the index, so no heap lookup is required. Example: if the index on `name` also includes `id`, then `SELECT id, name WHERE name='Zs'` is answered entirely from the index.
- **Spatial index**: for geo-data (e.g. R-tree in Postgres).
- **Full-text index**: tokenizes words for text search, handles things like synonyms.

## Creating indexes

```sql
CREATE INDEX <index_name>
ON <table_name> (column1, column2, ...);

-- Example
CREATE INDEX customers_by_phone ON customers (phone_number);
```

Variations across databases:
```sql
-- MySQL: unique index
CREATE UNIQUE INDEX idx_email ON users(email);

-- SQL Server
CREATE NONCLUSTERED INDEX idx_age ON people(age);
```
```js
// MongoDB (NoSQL)
db.collection.createIndex({ field: 1 })  // 1 = ascending
```

Full example (Postgres):
```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,  -- Auto-indexed
    name VARCHAR(255)
);

INSERT INTO employees (name) VALUES ('Alice'), ('Bob'), ('Zs');

CREATE INDEX idx_name ON employees (name);

EXPLAIN ANALYZE SELECT * FROM employees WHERE name = 'Zs';
-- Output should show "Index Scan" instead of "Seq Scan"
```

Indexes auto-update on every INSERT/UPDATE/DELETE — invisible to users, handled by the engine. Only index columns that are actually searched frequently; creating an index on a large table (e.g. 11M rows) itself takes time since it has to scan the whole table once to build it.

## 🟢 Beginner: with vs. without an index

Without an index, `SELECT id FROM employees WHERE name = 'Zs'` on 11M rows takes ~3 seconds (full scan). With `LIKE '%Zs%'` it's even slower (~1.1s) since it pattern-matches row by row.

After indexing `name`: the same equality query drops to ~47ms (bitmap index scan). If the query only needs indexed columns, there's no heap fetch at all — as low as 0.1ms. But `WHERE name LIKE '%Zs%'` still forces a full scan even with an index — a **leading wildcard** prevents the index from being used, because a B-tree can't start a search from an unknown prefix.

Other things a query plan reveals:
- **Planning time**: the query planner/optimizer deciding whether to use the index.
- **Execution time**: the actual work.
- **Caching**: repeat queries are faster due to OS/DB caching.
- **Heap fetch**: an extra step when the query needs columns not in the index (e.g. 2.5ms vs. 0.1ms).

## 🟡 Intermediate: when the optimizer skips your index

Having an index doesn't guarantee the database uses it. Common reasons it gets ignored:
- **Low selectivity**: if the filter would return more than roughly 10-20% of rows, a full scan is often faster than an index scan.
- **Functions/expressions on the column**: `WHERE UPPER(name) = 'ZS'` won't use a plain index on `name` — needs a functional index.
- **Leading wildcards in LIKE**: `%Zs%` still scans the full table.
- **OR conditions**: can defeat index use if the conditions are complex.
- **Small tables**: the overhead of an index lookup isn't worth it.
- **Outdated statistics**: the optimizer needs accurate row-count/selectivity stats — run `ANALYZE`.

Check with `EXPLAIN`; force it if needed (`FORCE INDEX` in MySQL).

**Selectivity** is the concept tying this together: indexes shine on high-selectivity columns (many unique values). Low-selectivity columns (e.g. booleans) are often skipped by the planner even when indexed.

### Index maintenance
- **Statistics**: run `ANALYZE TABLE` (MySQL) or `ANALYZE` (Postgres) to keep the optimizer's row-count info current.
- **Rebuild/reorganize**: indexes fragment over time; rebuild periodically (e.g. `ALTER INDEX REBUILD` in SQL Server).
- **Monitoring**: use `EXPLAIN` or tools like pgAdmin / `pg_stat_user_indexes` to spot slow queries and unused indexes.
- **Drop unused indexes**: `DROP INDEX idx_name;` — every unused index is pure write overhead.

## 🔴 Advanced

- **Covering indexes in practice**: put as much of the query's column list into the index as possible so it never needs the heap. Postgres 11+: `CREATE INDEX idx_cover ON employees (name) INCLUDE (id);`
- **Composite index ordering**: for compound `WHERE` clauses, put the most-filtered column first — a composite index on `(last_name, first_name)` serves `WHERE last_name = 'Smith' AND first_name = 'John'` well, but doesn't help a query that filters on `first_name` alone.
- **Partitioning + indexing**: for huge tables, combine partitioning (splitting by date/range) with local indexes per partition.
- **NoSQL indexing**: MongoDB supports secondary indexes; Cassandra allows indexing non-primary-key columns but with caveats around distributed query cost.
- **Index hints**: force the optimizer's hand when it's making the wrong call, but treat this as a last resort — usually a symptom of stale statistics or a bad index design.

## Pros and cons

**Pros**: faster reads/joins, enforces uniqueness constraints, enables efficient sorting/grouping.

**Cons**:
- **Slower writes**: every INSERT/UPDATE/DELETE has to update every index on the table too — more time, more disk I/O.
- **Storage overhead**: indexes take real space — composite indexes can add 50%+ of the table's own size.
- **Maintenance burden**: fragmentation accumulates over time and needs rebuilding.
- **Over-indexing**: too many indexes slow down writes without proportionate read benefit.

Rule of thumb: index read-heavy columns, avoid indexing write-heavy/high-churn ones.

## Real-world examples

- **E-commerce** (Amazon-like): index on `product_name` and `category`. `SELECT * FROM products WHERE category='Electronics' AND price < 100` stays fast across millions of items. Without it, the site lags and users leave.
- **Social media** (Twitter/X): index on `user_id` and `timestamp` for timelines — fetching recent posts stays fast at scale.
- **Finance** (banking app): index on `account_number` and `transaction_date` — range scans on audits avoid full scans across billions of records.
- **Healthcare**: index on `patient_id` and `diagnosis_code` for quick record lookups; without it, emergency-care queries are delayed.
- **Edge case — logging systems**: write-heavy by nature, so minimize indexes there to avoid slowing ingestion.

## Common pitfalls

- Assuming an index always gets used — the optimizer can and does skip them (see intermediate section).
- Leading wildcard `LIKE '%text%'` searches never use a standard B-tree index.
- Wrapping an indexed column in a function (`WHERE UPPER(name) = ...`) silently disables the index unless there's a matching functional index.
- Over-indexing a write-heavy table — each additional index is one more structure to update on every write.
- Forgetting to run `ANALYZE`/update statistics — a stale optimizer makes bad index-vs-scan decisions.
- Using `SELECT *` when a covering index could have satisfied a narrower column list without a heap fetch.

## Best practices

- Analyze real query patterns (slow `EXPLAIN` output) before adding indexes — don't index speculatively.
- Keep indexes limited — a rough ceiling of 5-10 per table is a reasonable sanity check, not a hard rule.
- Monitor usage (`pg_stat_user_indexes` in Postgres) and drop indexes that never get hit.
- Benchmark with and without an index before committing to it.
- Avoid `SELECT *` — it pulls unnecessary data and can defeat covering-index optimizations.
- Update statistics regularly.
- Consider alternatives for some problems: materialized views, or an external cache (Redis) instead of another index.

## Quick reference

- Index = sorted structure + pointers back to rows; speeds reads, costs writes.
- B-tree: default, handles equality + ranges. Hash: equality only, O(1). Bitmap: low-cardinality columns. LSM tree: write-heavy NoSQL.
- Composite index column order: most selective/most-filtered column first.
- Covering index: includes all needed columns, skips the heap fetch entirely.
- Optimizer skips indexes on: low selectivity, functions on the column, leading wildcards, stale stats, small tables.
- `EXPLAIN ANALYZE` to see whether an index is actually used.
- Maintenance: `ANALYZE` for stats, rebuild for fragmentation, drop unused indexes.
