# SQL Deep Dive: Beyond SELECT * FROM

> Portable SQL with notes on PostgreSQL vs MySQL 8 differences. Every pattern here shows up in real backend code and in interviews.

## Table of Contents
1. [Execution Order (Logical)](#order)
2. [Joins — All of Them, and How They Execute](#joins)
3. [Subqueries: Scalar, Correlated, EXISTS, IN, LATERAL](#subqueries)
4. [Aggregation, GROUP BY, HAVING, FILTER, ROLLUP/CUBE](#agg)
5. [Window Functions (the most underused SQL feature)](#window)
6. [CTEs and Recursive Queries](#cte)
7. [Set Operations](#sets)
8. [Upserts, RETURNING, MERGE](#upsert)
9. [Pagination: OFFSET vs Keyset](#pagination)
10. [Sargability: Writing Index-Friendly Predicates](#sargable)
11. [Dates, Times and Time Zones](#time)
12. [Strings, Collations and Case-Insensitivity](#strings)
13. [Classic Problem Patterns (with solutions)](#patterns)
14. [Postgres vs MySQL Syntax Cheat Sheet](#diff)

---

## 1. Logical Execution Order {#order}

```
1. FROM, JOIN (incl. LATERAL)      → build the row source
2. WHERE                           → filter rows (no aggregates allowed)
3. GROUP BY                        → collapse to groups
4. HAVING                          → filter groups
5. Window functions                → computed over the post-GROUP BY rows
6. SELECT (expressions, aliases)
7. DISTINCT
8. UNION / INTERSECT / EXCEPT
9. ORDER BY
10. OFFSET / LIMIT (FETCH FIRST)
```
Consequences:
- Can't filter on a window function in `WHERE` → wrap in subquery/CTE (or `QUALIFY` in Snowflake/DuckDB/BigQuery — not in PG/MySQL).
- `WHERE` vs `HAVING`: filter rows before grouping with `WHERE` (cheaper). `HAVING` only for aggregate conditions.
- The *physical* plan can be completely different — optimizer reorders freely as long as results match.

---

## 2. Joins {#joins}

```sql
-- INNER: only matching pairs
SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id;

-- LEFT: all left rows, NULLs where no match
SELECT c.id, o.id FROM customers c LEFT JOIN orders o ON o.customer_id = c.id;

-- RIGHT: mirror of LEFT (rarely used; rewrite as LEFT for readability)
-- FULL OUTER: all rows from both (MySQL lacks it — emulate with LEFT UNION RIGHT ... WHERE left IS NULL)
-- CROSS: cartesian product
-- SELF JOIN: table joined to itself (employees ↔ managers)
-- SEMI JOIN: EXISTS / IN — "rows in A that have a match in B", no duplication
-- ANTI JOIN: NOT EXISTS — "rows in A with no match in B"
```

### The ON vs WHERE trap with LEFT JOIN
```sql
-- Intent: all customers, plus their 2024 orders if any
SELECT c.id, o.id
FROM customers c LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= '2024-01-01';          -- WRONG: turns it into an INNER JOIN (NULL fails the predicate)

SELECT c.id, o.id
FROM customers c LEFT JOIN orders o
  ON o.customer_id = c.id AND o.created_at >= '2024-01-01';  -- RIGHT
```

### Join fan-out / double counting
```sql
-- Wrong: payments × shipments multiply each other
SELECT o.id, SUM(p.amount), COUNT(s.id)
FROM orders o
LEFT JOIN payments p ON p.order_id = o.id
LEFT JOIN shipments s ON s.order_id = o.id
GROUP BY o.id;
```
If an order has 2 payments and 3 shipments you get 6 rows → SUM triple-counted. Fix: pre-aggregate each child in a subquery/CTE or LATERAL, then join.

### How joins execute physically
| Algorithm | How | Best when | Complexity |
|---|---|---|---|
| **Nested loop** | For each outer row, look up inner (ideally via index) | Small outer side + index on inner join key | O(N × log M) with index, O(N×M) without |
| **Hash join** | Build hash table on smaller input, probe with the other | Large unsorted inputs, equality joins | O(N + M), needs memory (`work_mem` / `join_buffer_size`) — spills to disk if too big |
| **Merge join** | Both inputs sorted on key, walk in lockstep | Inputs already sorted (index) or huge | O(N + M) + sort cost |

MySQL had only nested-loop (with block nested loop) until **8.0.18 added hash joins**. Postgres has all three. Seeing "Nested Loop" with a big row estimate on the outer side and no index on the inner = the #1 slow-query pattern.

---

## 3. Subqueries {#subqueries}

```sql
-- Scalar subquery (must return ≤1 row)
SELECT name, (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count FROM customers c;

-- IN (semi-join)
SELECT * FROM customers WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);

-- EXISTS (semi-join, stops at first match)
SELECT * FROM customers c WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.total > 1000);

-- NOT EXISTS (anti-join) — prefer over NOT IN (NULL trap)
SELECT * FROM customers c WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

Modern optimizers usually treat `IN` and `EXISTS` identically. The real differences: `NOT IN` NULL semantics, and old MySQL (≤5.5) executing `IN (subquery)` as a dependent subquery per row (fixed in 5.6+ with semi-join strategies).

### LATERAL — "for each row, run this subquery" (top-N per group)
```sql
-- Latest 3 orders per customer (Postgres; MySQL 8.0.14+ supports LATERAL too)
SELECT c.id, o.*
FROM customers c
CROSS JOIN LATERAL (
  SELECT id, total, created_at FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC
  LIMIT 3
) o;
```
With an index on `orders(customer_id, created_at DESC)`, this does 3 index reads per customer — far better than a window function over the entire orders table when you only need a few customers.

---

## 4. Aggregation {#agg}

```sql
SELECT customer_id,
       COUNT(*)                         AS orders,
       COUNT(DISTINCT product_id)       AS distinct_products,
       SUM(total)                       AS revenue,
       AVG(total)                       AS aov,
       COUNT(*) FILTER (WHERE status = 'refunded') AS refunds      -- Postgres
       -- MySQL: SUM(status = 'refunded')  or  SUM(CASE WHEN status='refunded' THEN 1 ELSE 0 END)
FROM orders
WHERE created_at >= now() - interval '30 days'
GROUP BY customer_id
HAVING SUM(total) > 10000;
```

- **Functional dependence in GROUP BY**: Postgres lets you `SELECT c.name … GROUP BY c.id` when `c.id` is the PK. MySQL with `ONLY_FULL_GROUP_BY` (default since 5.7) does the same. Old MySQL allowed selecting arbitrary non-grouped columns and returned a random row's value — a famous bug source.
- `string_agg(name, ', ' ORDER BY name)` (PG) / `GROUP_CONCAT(name ORDER BY name SEPARATOR ', ')` (MySQL; truncated at `group_concat_max_len` = 1024 bytes by default!).
- `array_agg`, `json_agg`, `jsonb_object_agg` (PG); `JSON_ARRAYAGG`, `JSON_OBJECTAGG` (MySQL).
- `ROLLUP` / `CUBE` / `GROUPING SETS` for subtotals: `GROUP BY ROLLUP (region, city)` gives per city, per region, and grand total rows. MySQL supports `WITH ROLLUP` only.
- Percentiles: `percentile_cont(0.95) WITHIN GROUP (ORDER BY latency)` (PG). MySQL has no built-in; use window functions (`PERCENT_RANK`, `NTILE`) or compute in app.

---

## 5. Window Functions {#window}

A window function computes a value **per row** using a set of related rows, *without collapsing* them like GROUP BY.

```sql
function(...) OVER (
  PARTITION BY ...     -- groups (like GROUP BY but rows survive)
  ORDER BY ...         -- order within partition
  ROWS|RANGE|GROUPS BETWEEN ... AND ...  -- frame
)
```

### Ranking
```sql
SELECT name, dept, salary,
  ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn,   -- 1,2,3,4 (ties broken arbitrarily!)
  RANK()       OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk,  -- 1,2,2,4
  DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS drnk, -- 1,2,2,3
  NTILE(4)     OVER (ORDER BY salary) AS quartile,
  PERCENT_RANK() OVER (ORDER BY salary) AS pr
FROM employees;
```
Make `ROW_NUMBER` deterministic by adding a tiebreaker: `ORDER BY salary DESC, id`.

### Offsets
```sql
LAG(price)  OVER (PARTITION BY symbol ORDER BY ts)     -- previous row
LEAD(price, 1, 0) OVER (...)                            -- next row, default 0
FIRST_VALUE(x) OVER w, LAST_VALUE(x) OVER w, NTH_VALUE(x, 2) OVER w
```

### Running totals and moving averages — frames
```sql
SELECT day, revenue,
  SUM(revenue) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
  AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)         AS ma_7d
FROM daily_revenue;
```
**Frame default gotcha**: with `ORDER BY` and no explicit frame, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — rows with **equal ORDER BY values (peers) are all included**. That's why `LAST_VALUE(x) OVER (ORDER BY d)` returns the *current* row's peer group's last value, not the partition's last value. Use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

- `ROWS` = physical row count.
- `RANGE` = value-based (`RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` — correct for gaps in dates; PG 11+, MySQL 8).
- `GROUPS` = peer groups (PG only).

Named windows: `... OVER w ... WINDOW w AS (PARTITION BY dept ORDER BY salary)`.

---

## 6. CTEs and Recursion {#cte}

```sql
WITH recent AS (
  SELECT * FROM orders WHERE created_at > now() - interval '7 days'
), big AS (
  SELECT customer_id, SUM(total) s FROM recent GROUP BY customer_id HAVING SUM(total) > 1000
)
SELECT c.* FROM customers c JOIN big USING (customer_id);
```

**Materialization behavior (important!)**:
- Postgres < 12: CTEs were *always* materialized ("optimization fence") — filters outside weren't pushed in. Old advice "CTEs are slow" comes from this.
- Postgres ≥ 12: non-recursive, side-effect-free CTEs referenced once are **inlined**. Force with `AS MATERIALIZED` / `AS NOT MATERIALIZED`.
- MySQL 8: optimizer may merge or materialize.

### Recursive CTE
```sql
-- Generate a date series (MySQL; PG has generate_series)
WITH RECURSIVE days(d) AS (
  SELECT DATE '2025-01-01'
  UNION ALL
  SELECT d + INTERVAL 1 DAY FROM days WHERE d < '2025-01-31'
)
SELECT d FROM days;
```
MySQL caps depth with `cte_max_recursion_depth` (1000).

### Data-modifying CTEs (Postgres)
```sql
-- Move rows atomically in one statement
WITH moved AS (
  DELETE FROM jobs WHERE status = 'done' AND finished_at < now() - interval '30 days'
  RETURNING *
)
INSERT INTO jobs_archive SELECT * FROM moved;
```

---

## 7. Set Operations {#sets}

- `UNION` removes duplicates (sort/hash cost). **`UNION ALL` doesn't — use it unless you need dedup.**
- `INTERSECT`, `EXCEPT` (MySQL 8.0.31+). Oracle calls EXCEPT `MINUS`.
- Columns matched by position, not name.

---

## 8. Upserts, RETURNING, MERGE {#upsert}

```sql
-- PostgreSQL
INSERT INTO counters (key, value) VALUES ('page:home', 1)
ON CONFLICT (key) DO UPDATE SET value = counters.value + EXCLUDED.value
RETURNING value;

INSERT INTO users (email, name) VALUES ($1, $2)
ON CONFLICT (email) DO NOTHING;   -- idempotent insert

-- MySQL
INSERT INTO counters (`key`, value) VALUES ('page:home', 1) AS new
ON DUPLICATE KEY UPDATE value = counters.value + new.value;   -- 8.0.19+ alias syntax (VALUES() deprecated)
```

Gotchas:
- MySQL `ON DUPLICATE KEY` fires on *any* unique key conflict, not a specific one. With multiple unique keys it may update an unexpected row.
- MySQL `REPLACE INTO` = DELETE + INSERT → fires delete triggers, cascades FKs, changes auto-increment id. Almost never what you want.
- MySQL auto-increment values are consumed even when the upsert updates → gaps (harmless but surprising).
- Postgres `ON CONFLICT DO UPDATE` is atomic and race-free; the "SELECT then INSERT" pattern in app code is **not**.
- `RETURNING` (Postgres; MariaDB has it; MySQL doesn't — use `LAST_INSERT_ID()`).
- `MERGE` (SQL standard): Postgres 15+ (with `RETURNING` in 17). Prefer `ON CONFLICT` for simple upserts in PG — MERGE isn't guaranteed race-free against concurrent inserts.

---

## 9. Pagination {#pagination}

### OFFSET/LIMIT
```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
```
Problems:
1. DB still reads and discards 100,000 rows → page N costs O(N).
2. **Unstable**: if rows are inserted/deleted between page loads, users see duplicates or miss items.

### Keyset (seek / cursor) pagination
```sql
-- First page
SELECT id, created_at, title FROM posts ORDER BY created_at DESC, id DESC LIMIT 20;

-- Next page: pass last row's (created_at, id) as cursor
SELECT id, created_at, title FROM posts
WHERE (created_at, id) < ($last_created_at, $last_id)       -- row-value comparison (PG, MySQL)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
Index: `(created_at DESC, id DESC)`. Cost is O(page size) regardless of depth. Unique tiebreaker (`id`) is mandatory, otherwise rows with equal timestamps get skipped.

Note: MySQL historically didn't use indexes well for row-value comparisons; the expanded form is safer:
```sql
WHERE created_at < :ts OR (created_at = :ts AND id < :id)
```
Trade-off: no "jump to page 57". Encode the cursor opaquely (base64 JSON) in the API. This is what Slack, Stripe, GitHub, Twitter APIs use.

---

## 10. Sargability {#sargable}

**SARGable** = *Search ARGument able*: the predicate can use an index range scan. Wrapping the indexed column in a function or expression usually kills index use.

| Non-sargable ❌ | Sargable ✅ |
|---|---|
| `WHERE YEAR(created_at) = 2025` | `WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'` |
| `WHERE DATE(created_at) = '2025-06-01'` | `WHERE created_at >= '2025-06-01' AND created_at < '2025-06-02'` |
| `WHERE price * 1.18 > 1000` | `WHERE price > 1000 / 1.18` |
| `WHERE LOWER(email) = 'a@b.com'` | Expression index `ON users (LOWER(email))`, or citext / case-insensitive collation |
| `WHERE name LIKE '%son'` | Trigram index (PG `pg_trgm`), reverse-string index, or full-text search |
| `WHERE id::text = '42'` / implicit cast | Compare like with like: `WHERE id = 42` |
| `WHERE COALESCE(deleted_at, now()) > …` | Rewrite or partial index |
| `WHERE a = 1 OR b = 2` | Often fine (bitmap OR in PG; index_merge in MySQL), else `UNION ALL` |

**Implicit type conversion** is the silent killer in MySQL: `WHERE phone = 9876543210` when `phone` is `VARCHAR` → MySQL casts *every row's* column to number → full scan. Same with charset/collation mismatches in joins.

`LIKE 'abc%'` (prefix) is sargable with a B-tree (in PG need `text_pattern_ops` or `C` collation for non-C locales).

---

## 11. Dates, Times and Time Zones {#time}

- Store instants in UTC. PG: **`TIMESTAMPTZ`** (stores UTC, converts to session `TimeZone` on display). `TIMESTAMP` (without tz) is a wall-clock reading with no zone — use only for "local" concepts like "store opens at 09:00".
- MySQL: `TIMESTAMP` is stored as UTC and converted using session `time_zone`, but range ends **2038-01-19** (32-bit epoch) — MySQL 8.0.28+ extended it on 64-bit platforms for functions, but the column type range is still documented to 2038. `DATETIME` has no zone semantics; store UTC by convention.
- Never store "user's local time" for events; store UTC + the user's IANA zone (`Asia/Kolkata`) separately when needed.
- Date ranges: use **half-open intervals** `[start, end)` — `>= start AND < end`. Avoids `23:59:59.999` hacks and works across precision.
- Bucketing: `date_trunc('hour', ts)` (PG), `DATE_FORMAT(ts, '%Y-%m-%d %H:00:00')` (MySQL). TimescaleDB `time_bucket`.
- `now()` in PG returns the *transaction start* time (constant within a transaction). `clock_timestamp()` gives wall time.
- DST: "add 1 day" ≠ "add 24 hours" in a zone with DST. PG `interval '1 day'` on `timestamptz` respects the session zone.

---

## 12. Strings, Collations, Case {#strings}

- **Collation** decides ordering and equality (`'a' = 'A'`?). MySQL default `utf8mb4_0900_ai_ci` is **accent- and case-insensitive**: `'resume' = 'Résumé'` is TRUE, and a UNIQUE index treats them as duplicates. Postgres default collations are case-sensitive (deterministic); use `citext` or nondeterministic ICU collations for case-insensitive.
- **MySQL `utf8` is NOT UTF-8** — it's `utf8mb3` (max 3 bytes, no emoji, no some CJK). Always `utf8mb4`.
- `VARCHAR(n)`: PG counts characters; `TEXT` and `VARCHAR` are the same performance in PG — length limits are business rules (a CHECK is just as good). In MySQL, `VARCHAR` length affects index key size limits (3072 bytes for InnoDB DYNAMIC) and in-memory temp tables.
- `CHAR(n)` pads with spaces — basically never use it.
- Changing collation of an indexed column = index rebuild. glibc upgrades changing collation rules can **silently corrupt PG indexes** (glibc 2.28 incident) — prefer ICU or `C` collation for keys; Postgres 17 adds a builtin `C.UTF-8` provider.

---

## 13. Classic Problem Patterns {#patterns}

### Top-N per group
```sql
SELECT * FROM (
  SELECT e.*, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC, id) rn FROM employees e
) t WHERE rn <= 3;
-- PG: DISTINCT ON for top-1:
SELECT DISTINCT ON (dept) * FROM employees ORDER BY dept, salary DESC, id;
```

### Nth highest salary
```sql
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;  -- 2nd highest
-- or DENSE_RANK() = N
```

### Deduplicate keeping the newest row
```sql
DELETE FROM events e USING (
  SELECT id, ROW_NUMBER() OVER (PARTITION BY external_id ORDER BY received_at DESC, id DESC) rn FROM events
) d WHERE e.id = d.id AND d.rn > 1;        -- PG
```
Then add a UNIQUE constraint so it can't recur.

### Gaps and islands (consecutive streaks)
```sql
-- Longest streak of consecutive login days per user
WITH d AS (SELECT DISTINCT user_id, login_date FROM logins),
g AS (
  SELECT user_id, login_date,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
  FROM d
)
SELECT user_id, MIN(login_date) AS start, MAX(login_date) AS finish, COUNT(*) AS days
FROM g GROUP BY user_id, grp ORDER BY days DESC;
```
Trick: date minus row-number is constant within a run of consecutive dates.

### Running balance
```sql
SELECT account_id, ts, amount,
       SUM(amount) OVER (PARTITION BY account_id ORDER BY ts, id ROWS UNBOUNDED PRECEDING) AS balance
FROM ledger;
```

### Pivot (rows → columns)
```sql
SELECT product_id,
  SUM(qty) FILTER (WHERE month = 1) AS jan,
  SUM(qty) FILTER (WHERE month = 2) AS feb
FROM sales GROUP BY product_id;      -- MySQL: SUM(CASE WHEN month=1 THEN qty END)
```

### Month-over-month growth
```sql
SELECT month, revenue,
  ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month)) / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2) AS mom_pct
FROM monthly_revenue;
```
`NULLIF(x, 0)` prevents division-by-zero errors.

### Retention cohort
```sql
WITH first AS (SELECT user_id, date_trunc('month', MIN(created_at)) cohort FROM orders GROUP BY user_id),
act AS (SELECT DISTINCT user_id, date_trunc('month', created_at) m FROM orders)
SELECT f.cohort,
       EXTRACT(YEAR FROM age(a.m, f.cohort))*12 + EXTRACT(MONTH FROM age(a.m, f.cohort)) AS month_n,
       COUNT(*) users
FROM first f JOIN act a USING (user_id)
GROUP BY 1, 2 ORDER BY 1, 2;
```

### Find rows in A not in B (anti-join) — three ways, choose NOT EXISTS
```sql
SELECT a.* FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.a_id = a.id);
SELECT a.* FROM a LEFT JOIN b ON b.a_id = a.id WHERE b.a_id IS NULL;
SELECT id FROM a EXCEPT SELECT a_id FROM b;
```

### Job queue pop (concurrency-safe)
```sql
-- Many workers, no double-processing, no blocking each other (PG 9.5+, MySQL 8.0+)
UPDATE jobs SET status = 'running', locked_at = now()
WHERE id = (
  SELECT id FROM jobs WHERE status = 'queued'
  ORDER BY priority DESC, id
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

---

## 14. Postgres vs MySQL Syntax Cheat Sheet {#diff}

| Task | PostgreSQL | MySQL 8 |
|---|---|---|
| Identifier quoting | `"name"` | `` `name` `` (or `"` with ANSI_QUOTES) |
| Auto ID | `BIGINT GENERATED ALWAYS AS IDENTITY` (or `BIGSERIAL`) | `BIGINT AUTO_INCREMENT` |
| Upsert | `ON CONFLICT … DO UPDATE` | `ON DUPLICATE KEY UPDATE` |
| Return inserted row | `RETURNING *` | `LAST_INSERT_ID()` |
| String concat | `a \|\| b` | `CONCAT(a, b)` (`\|\|` is OR unless PIPES_AS_CONCAT) |
| Boolean | real `BOOLEAN` | `TINYINT(1)` alias |
| Case-insensitive like | `ILIKE` | `LIKE` under `_ci` collation |
| Regex | `~`, `~*` | `REGEXP` / `REGEXP_LIKE` |
| Limit | `LIMIT n OFFSET m` / `FETCH FIRST n ROWS ONLY` | `LIMIT m, n` or `LIMIT n OFFSET m` |
| Full outer join | ✅ | ❌ (emulate) |
| `FILTER (WHERE …)` | ✅ | ❌ (CASE) |
| `DISTINCT ON` | ✅ | ❌ (window fn) |
| Arrays | ✅ native | ❌ (JSON) |
| JSON | `jsonb` (binary, indexable with GIN) | `JSON` (binary), index via generated columns / multi-valued indexes |
| Transactional DDL | ✅ (`BEGIN; ALTER…; ROLLBACK;` works) | ❌ (DDL implicitly commits) |
| Partial index | ✅ `WHERE` | ❌ (functional index workaround) |
| Generate series | `generate_series()` | recursive CTE |
| Date diff | `age()`, subtraction → interval | `TIMESTAMPDIFF(unit, a, b)` |
| Explain with runtime | `EXPLAIN (ANALYZE, BUFFERS)` | `EXPLAIN ANALYZE` (8.0.18+) |

**Practice**: [`interview-prep/backend-engineer/10-sql-query-patterns-and-questions.md`](../../interview-prep/backend-engineer/10-sql-query-patterns-and-questions.md) has more drill questions.
