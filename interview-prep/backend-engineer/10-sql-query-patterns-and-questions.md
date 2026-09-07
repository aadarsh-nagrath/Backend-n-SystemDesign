# SQL Query Patterns & Practical Questions

> Sourced/topic-checked against GeeksforGeeks' 88-question SQL interview list. Complements `03-databases-sql-nosql.md` (which covers conceptual internals — MVCC, isolation levels, indexing internals) with the practical, "write a query" style questions that actually come up in SQL screening rounds.

---

### 1. What's the difference between `CHAR` and `VARCHAR`/`VARCHAR2`?
`CHAR(n)` is fixed-length — a stored value shorter than `n` is padded with spaces to always occupy exactly `n` characters of storage, regardless of actual content length. `VARCHAR`/`VARCHAR2(n)` is variable-length — it only stores the actual characters used (plus a small length overhead), up to a maximum of `n`. Use `CHAR` for genuinely fixed-length data (a 2-letter country code, a fixed-format ID) where the padding is harmless and the fixed size can occasionally offer marginal storage/performance benefits; use `VARCHAR` for anything variable in length (names, descriptions) to avoid wasting storage on padding.

### 2. What is a SQL view, and what's the difference between a regular view and a materialized view?
A view is a saved, named `SELECT` query that behaves like a virtual table — querying it re-runs the underlying query live against current data every time, with no data actually duplicated/stored. A materialized view *does* physically store the query's result set, refreshed periodically or on demand (not live) — trading data freshness for read speed, since querying it reads pre-computed stored data instead of re-executing a potentially expensive query every time. Use a regular view for convenience/abstraction over a query you don't want to keep rewriting; use a materialized view when the underlying query is expensive and slight staleness is an acceptable tradeoff for much faster reads.

### 3. Write a query to find the second-highest salary in an `employees` table.
```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```
This works by first finding the overall maximum, then finding the maximum salary strictly less than that. It naturally handles ties correctly (if multiple employees share the highest salary, this still correctly finds the *next distinct* value down) — a common follow-up is generalizing this to the Nth-highest using `DENSE_RANK()` (see Q7) instead, which scales more cleanly to "Nth highest" than nesting more subqueries.

### 4. What's the difference between `UNION` and `UNION ALL`?
`UNION` combines the result sets of two queries and removes duplicate rows (requiring an implicit sort/dedup pass, which has a real performance cost). `UNION ALL` combines them without removing duplicates — faster, since it skips the deduplication work entirely. Use `UNION ALL` whenever you know the result sets are already disjoint or duplicates are acceptable/expected, reserving `UNION` for when you specifically need deduplication.

### 5. Explain a correlated subquery, and give a use case.
A correlated subquery references a column from the *outer* query, meaning it can't be evaluated independently — it's conceptually re-evaluated once per row of the outer query. Example: finding employees who earn more than their own department's average salary:
```sql
SELECT e.name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
  SELECT AVG(salary) FROM employees e2 WHERE e2.department_id = e.department_id
);
```
The inner query's `WHERE e2.department_id = e.department_id` correlates it to the current outer row — this is different from a plain (uncorrelated) subquery, which is computed once and reused for every outer row.

### 6. What's the difference between `EXISTS`/`NOT EXISTS` and `IN`/`NOT IN`, and when does `NOT IN` have a dangerous gotcha?
`EXISTS` checks only whether a correlated subquery returns *any* row (often faster for large subquery result sets, since the database can short-circuit on the first match), while `IN` materializes the full subquery result set to compare against. The dangerous gotcha: `NOT IN` behaves unexpectedly if the subquery's result set contains even a single `NULL` — in standard SQL, comparing anything to `NULL` yields `UNKNOWN` rather than `TRUE`/`FALSE`, which can cause `NOT IN` to unexpectedly return *zero rows* for the entire outer query, silently. `NOT EXISTS` doesn't have this problem, which is why `NOT EXISTS` is generally the safer default recommendation over `NOT IN` whenever the subquery's column could contain NULLs.

### 7. Explain the difference between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()`.
All three are window functions assigning a sequential position to rows within an ordered partition. `ROW_NUMBER()` always assigns strictly sequential unique numbers (1, 2, 3, 4...) even for tied values — ties are broken arbitrarily (or by additional `ORDER BY` tiebreakers). `RANK()` gives tied rows the same rank, then *skips* the next rank number(s) accordingly (1, 2, 2, 4 — skipping 3, since two rows tied for 2nd). `DENSE_RANK()` also gives tied rows the same rank but does *not* skip subsequent numbers (1, 2, 2, 3). Use `DENSE_RANK()` for "find the Nth highest distinct value" style questions (generalizing Q3): `SELECT * FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk FROM employees) t WHERE rnk = 2;`.

### 8. What do the `LAG()` and `LEAD()` window functions do, and give a use case?
`LAG(column, n)` retrieves a value from `n` rows *before* the current row (within the window's partition/order); `LEAD(column, n)` retrieves a value from `n` rows *after*. Use case: computing month-over-month change — `SELECT month, revenue, revenue - LAG(revenue) OVER (ORDER BY month) AS change FROM monthly_revenue;` — without these, you'd need a self-join against a shifted version of the same table, which is far more verbose and typically slower.

### 9. Write a query to compute a running total of sales per product, ordered by date.
```sql
SELECT product_id, sale_date, amount,
       SUM(amount) OVER (
         PARTITION BY product_id
         ORDER BY sale_date
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM sales;
```
The `PARTITION BY` resets the running total per product; the `ORDER BY` combined with the frame clause (`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which is actually the default frame when `ORDER BY` is present in the window, but is good practice to state explicitly) accumulates the sum from the start of that partition up through the current row.

### 10. What is a Common Table Expression (CTE), and how does it differ from a subquery or a temporary table?
A CTE (`WITH cte_name AS (SELECT ...) SELECT * FROM cte_name`) is a named, temporary result set scoped to a single query — improving readability for complex queries (breaking a large query into named, logical steps) and allowing the same named result to be referenced multiple times within that one query without repeating the subquery text. Unlike a subquery, a CTE can be self-referencing (recursive CTEs, see Q17) and generally reads top-to-bottom more clearly. Unlike a temporary table, a CTE has no persistent storage and exists only for the duration of the single statement it's attached to — it isn't indexed or reusable across separate queries the way a temp table can be.

### 11. What's the difference between a temporary table and a table variable/CTE in terms of performance for a large intermediate result set?
A temporary table is materialized (physically written to disk/tempdb) and *can* have its own indexes and statistics, making it a better choice than a CTE when the intermediate result set is large and will be queried/joined against multiple times within a complex multi-step process — the database's query optimizer can make better decisions with real statistics on a materialized temp table than it often can when repeatedly inlining a CTE's definition. A CTE (especially a non-recursive one) is often, though not always guaranteed by every database engine, treated more like an inlined subquery expanded at each reference point rather than materialized once — meaning a CTE referenced multiple times in one query can sometimes cause the underlying logic to be recomputed multiple times, unlike a genuinely materialized temp table computed once upfront.

### 12. Explain `PIVOT`, and what it's used for.
`PIVOT` transforms row-based data into a column-based (cross-tab) layout — e.g., turning rows of `(month, product, sales)` into a single row per product with one column per month showing that month's sales, useful for reporting/dashboard-style output where a spreadsheet-like cross-tab view is more readable than long-format rows. Syntax and support vary meaningfully across database engines (SQL Server has a native `PIVOT` operator; Postgres/MySQL typically achieve the same result via conditional aggregation, e.g., `SUM(CASE WHEN month = 'Jan' THEN sales END)` for each desired column) — worth mentioning this portability difference if asked, since it's a common "gotcha" when someone learned `PIVOT` syntax on one engine and expects it to transfer directly.

### 13. What is a bitmap index, and how does it differ from a standard B-tree index?
A bitmap index stores, for each distinct value of a column, a bitmap (one bit per row) indicating whether that row has that value — extremely space-efficient and fast for columns with *low cardinality* (few distinct values, e.g., a "gender" or "status" column) since bitwise operations (AND/OR) across bitmaps for combined filter conditions are very cheap. A B-tree index is far better suited to high-cardinality columns (like a primary key or a unique email) where a bitmap-per-value approach would need an enormous number of near-empty bitmaps. Bitmap indexes are also generally poorly suited to tables with frequent, highly concurrent writes (updating a bitmap index can require locking a large swath of the bitmap, unlike a B-tree's more localized update) — making them more common in read-heavy analytical/data-warehouse workloads than in high-write OLTP systems.

### 14. What's the difference between OLTP and OLAP, and why do they typically use different database designs?
OLTP (Online Transaction Processing) systems handle many short, frequent read/write transactions (a typical application database — placing an order, updating a user profile) — optimized for fast, highly-concurrent point lookups/writes, typically normalized to minimize update anomalies (see the normalization question in the main databases file). OLAP (Online Analytical Processing) systems handle complex, large-scan analytical queries (aggregations across millions of rows, e.g., "total revenue by region by quarter") — typically denormalized (star/snowflake schemas) and column-oriented, optimized for scanning and aggregating large volumes of data efficiently rather than for fast individual-row updates. Trying to run heavy OLAP-style reporting queries directly against a production OLTP database is a classic anti-pattern (it competes for the same resources/locks the transactional workload needs), which is why data warehouses/ETL pipelines exist to separate the two concerns (see the data storage design file's CDC question).

### 15. How would you find and delete duplicate rows in a table, keeping only one copy of each?
```sql
DELETE FROM employees
WHERE id NOT IN (
  SELECT MIN(id)
  FROM employees
  GROUP BY name, email, department  -- the columns that define a "duplicate"
);
```
Or, more robustly (avoiding the `NOT IN`/NULL gotcha from Q6) using window functions:
```sql
DELETE FROM employees
WHERE id IN (
  SELECT id FROM (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY name, email, department ORDER BY id) AS rn
    FROM employees
  ) t
  WHERE rn > 1
);
```
The window-function version is generally preferred in practice — it's explicit about keeping exactly one row per duplicate group (the first by whatever `ORDER BY` you choose) and avoids `NOT IN`'s NULL-handling trap entirely.

### 16. How would you find employees hired within the last N months?
```sql
SELECT * FROM employees
WHERE hire_date >= CURRENT_DATE - INTERVAL '6 months';  -- Postgres syntax
```
Or, using an explicit difference function (MySQL's `TIMESTAMPDIFF`):
```sql
SELECT * FROM employees
WHERE TIMESTAMPDIFF(MONTH, hire_date, CURDATE()) <= 6;
```
Note the tradeoff: the first form (a direct date-range comparison) is generally preferable for performance, since it can use an index on `hire_date` directly (a "sargable" predicate); the second form wraps the indexed column in a function call, which — depending on the database engine — can prevent the query planner from using an index on `hire_date` efficiently, forcing a full scan even on a large, well-indexed table. This distinction (sargable vs. non-sargable predicates) is a genuinely important practical performance point worth raising proactively in an interview.

### 17. What is a recursive CTE, and give a use case?
A recursive CTE references itself, repeatedly building up a result by combining a base case (the "anchor" query) with a recursive step that references the CTE's own previous iteration, until the recursive step produces no more new rows. Classic use case: traversing a hierarchical structure like an org chart (finding all employees under a given manager, at any depth):
```sql
WITH RECURSIVE subordinates AS (
  SELECT id, name, manager_id FROM employees WHERE id = 1  -- anchor: the starting manager
  UNION ALL
  SELECT e.id, e.name, e.manager_id
  FROM employees e
  JOIN subordinates s ON e.manager_id = s.id  -- recursive step: find their direct reports
)
SELECT * FROM subordinates;
```
This is the standard SQL approach to representing tree/graph traversal that a plain (non-recursive) query fundamentally can't express, since a normal query can't know in advance how many levels deep the hierarchy goes.

### 18. What is the difference between horizontal and vertical partitioning, and how does partitioning differ from sharding (see also the data storage design file)?
Horizontal partitioning splits a table's *rows* into multiple physical partitions based on some criteria (e.g., by date range, by region) while keeping the same columns in each partition — commonly used to make queries against a specific range (and old-data archival/deletion) far cheaper, since the database can skip scanning partitions that don't match a query's filter ("partition pruning"). Vertical partitioning splits a table's *columns* across multiple physical tables (e.g., separating rarely-accessed large text/blob columns from frequently-accessed small columns) to improve cache efficiency for the hot columns. The key distinction from sharding: partitioning typically happens *within a single database instance* (the database engine manages routing to the right partition transparently), while sharding distributes data across *multiple separate database instances/servers* — partitioning solves a single-server query-efficiency problem, sharding solves a horizontal-scale-beyond-one-server problem.

### 19. What's the purpose of the `COALESCE` function, and how does it differ from `NVL`/`NVL2` (Oracle-specific)?
`COALESCE(a, b, c, ...)` returns the first non-NULL value among its arguments — standard ANSI SQL, supported broadly across database engines, and can take any number of arguments. `NVL(a, b)` (Oracle-specific) is functionally similar but limited to exactly two arguments (returns `a` if not null, else `b`). `NVL2(a, b, c)` (also Oracle-specific) returns `b` if `a` is not null, or `c` if `a` is null — a three-way conditional in one function call, unlike `COALESCE`'s simple "first non-null" semantics. `COALESCE` is generally the more portable choice to reach for by default given its wide cross-engine support.

### 20. What happens if you run `COUNT()` on a column containing NULL values, and how does this differ from `COUNT(*)`?
`COUNT(column_name)` counts only the rows where that specific column is *not* NULL — NULL values are excluded from the count entirely. `COUNT(*)` counts all rows regardless of any column's NULL-ness (it's counting rows, not evaluating a specific column's values at all). This distinction is a very common source of subtly wrong query results when someone assumes `COUNT(some_column)` gives the total row count and doesn't realize it silently excludes NULLs in that specific column.
