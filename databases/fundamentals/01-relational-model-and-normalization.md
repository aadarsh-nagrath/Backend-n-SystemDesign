# The Relational Model, Keys, and Normalization

> Goal: understand *why* relational databases look the way they do, so that schema decisions stop being guesswork. Everything later (indexes, transactions, query planning) builds on this.

## Table of Contents
1. [Why the Relational Model Exists](#why)
2. [Core Vocabulary (and the Formal Names)](#vocab)
3. [Keys: Super, Candidate, Primary, Alternate, Foreign, Surrogate, Natural](#keys)
4. [Constraints: The Database as the Last Line of Defense](#constraints)
5. [Functional Dependencies — the Math Behind Normalization](#fd)
6. [Anomalies: What Bad Schemas Break](#anomalies)
7. [Normal Forms 1NF → 5NF with Worked Examples](#nf)
8. [Denormalization: When and How to Break the Rules](#denorm)
9. [Relationship Patterns (1:1, 1:N, M:N, Self-referencing, Polymorphic)](#relationships)
10. [Modeling Hierarchies and Trees in SQL](#trees)
11. [NULL: The Three-Valued Logic Trap](#null)
12. [Relational Algebra (what SQL compiles to)](#algebra)
13. [Checklist & Interview Questions](#checklist)

---

## 1. Why the Relational Model Exists {#why}

Before 1970, databases were **hierarchical** (IBM IMS) or **network** (CODASYL). To read data you navigated pointers: "start at customer record, follow the pointer to the first order, follow next-order pointer…". Your program encoded the *access path*. Change the physical layout → rewrite the program.

Edgar F. Codd (IBM, 1970, *"A Relational Model of Data for Large Shared Data Banks"*) proposed:

1. **Data independence** — applications describe *what* they want, not *how* to get it. The physical layout (indexes, files, order) can change without touching queries.
2. **Everything is a relation** (a set of tuples). No pointers visible to the user.
3. **A declarative, set-based language** (became SQL) with a formal mathematical basis (relational algebra / relational calculus).

This is the single most important idea: **SQL is declarative; the query optimizer chooses the access path.** That's why the same query can be fast one day and slow the next (statistics changed → plan changed), and why "understand the planner" is a senior skill.

---

## 2. Core Vocabulary {#vocab}

| Everyday term | Formal term | Meaning |
|---|---|---|
| Table | Relation | A *set* of tuples sharing a heading |
| Row | Tuple | One fact |
| Column | Attribute | A named, typed slot |
| Column type | Domain | Set of allowed values |
| Number of columns | Degree / arity | |
| Number of rows | Cardinality | (also used for "number of distinct values" in index talk — context matters) |
| Schema of table | Relation schema / heading | |

Important consequences of "a relation is a **set**":
- **No duplicate rows** in theory. SQL tables are actually *bags* (multisets) — they allow duplicates unless you add a key. This is why you always want a primary key.
- **No inherent row order.** `SELECT * FROM t` without `ORDER BY` has **no guaranteed order**, even if it "always comes back sorted" in dev. Parallel scans, synchronized scans (Postgres), and plan changes will break that assumption.
- **No inherent column order** in theory (SQL does keep one for `SELECT *`, which is why `SELECT *` in application code is fragile).

**Each row is a true proposition.** A row `(42, 'Asha', 'Bangalore')` in `users(id, name, city)` asserts "user 42 is named Asha and lives in Bangalore". This "rows are facts" framing helps hugely when deciding where a column belongs: *is this attribute a fact about the whole key?*

---

## 3. Keys {#keys}

- **Superkey**: any set of columns that uniquely identifies a row. `{id}`, `{id, name}`, `{email, name}` are all superkeys if `id` and `email` are unique.
- **Candidate key**: a *minimal* superkey (remove any column and it stops being unique). `{id}` and `{email}`.
- **Primary key (PK)**: the candidate key you choose as the main identifier. Implies `NOT NULL` + `UNIQUE`.
- **Alternate key**: the other candidate keys → enforce with `UNIQUE` constraints. *People forget this*: if email must be unique, declare it; don't rely on application checks (race conditions!).
- **Foreign key (FK)**: column(s) referencing a candidate key in another (or same) table. Enforces *referential integrity*.
- **Composite key**: a key of more than one column, e.g. `(order_id, line_no)`.
- **Natural key**: a key with business meaning (email, ISBN, country code).
- **Surrogate key**: a meaningless generated ID (`BIGINT` identity, UUID).

### Natural vs surrogate — the real trade-offs

| | Natural key | Surrogate key |
|---|---|---|
| Stability | Business values change (people change emails; "unique" SSNs get reissued) | Never changes |
| Size | Often wide (strings) → every FK and every secondary index (in InnoDB) carries it | 8 bytes (`BIGINT`) or 16 (UUID) |
| Meaning | Self-documenting, avoids a join sometimes | Requires join to show anything |
| Recommended | For tiny, truly immutable lookup codes (ISO country `IN`, currency `INR`) | Almost everything else — **plus** a `UNIQUE` constraint on the natural key |

### Integer vs UUID primary keys

| | `BIGINT` identity/auto-increment | UUIDv4 (random) | UUIDv7 / ULID (time-ordered) |
|---|---|---|---|
| Size | 8 bytes | 16 bytes (36 as text — never store as text!) | 16 bytes |
| Generated | By DB (needs a round trip or `RETURNING`) | Anywhere, offline | Anywhere |
| Index locality | Perfect: always appends to rightmost B-tree page | **Terrible**: random inserts all over the B-tree → page splits, cache misses, bloated index, WAL/redo amplification | Good: roughly monotonic |
| Leaks info | Yes — row count & growth rate guessable (`/orders/1005` → competitor knows your volume), enumeration attacks | No | Leaks creation timestamp |
| Merging/sharding | Collisions across shards | Globally unique | Globally unique |

**Rule of thumb (2025+)**: `BIGINT` internally is the default; if you need client-generated or globally unique IDs, use **UUIDv7** (Postgres 18 has `uuidv7()`; elsewhere generate in app). Avoid random UUIDv4 as a **clustered** key in MySQL/InnoDB in particular — the whole table *is* the PK B-tree there (see `mysql/02-innodb-internals.md`).

Never expose sequential IDs if enumeration matters; either use a separate public ID (UUID/slug) or authorize every access (you should anyway — IDOR is OWASP API #1).

---

## 4. Constraints {#constraints}

Constraints are *declarative invariants the database guarantees under concurrency*. Application-level checks cannot (two requests can both check "email not taken" and both insert).

| Constraint | Purpose | Notes |
|---|---|---|
| `NOT NULL` | Required value | Default to NOT NULL; allow NULL only when "unknown/not applicable" is a real state |
| `UNIQUE` | Alternate keys | In most DBs, multiple NULLs are allowed in a UNIQUE column (NULL ≠ NULL). Postgres 15+: `UNIQUE NULLS NOT DISTINCT` |
| `PRIMARY KEY` | Identity | |
| `FOREIGN KEY` | Referential integrity | Choose `ON DELETE` behavior deliberately: `RESTRICT`/`NO ACTION` (default, safe), `CASCADE` (dangerous at scale — one delete can lock/delete millions), `SET NULL`, `SET DEFAULT` |
| `CHECK` | Row-level predicate | `CHECK (price >= 0)`, `CHECK (start_at < end_at)`. MySQL only enforces since 8.0.16! |
| `EXCLUDE` (Postgres) | "No two rows overlap" | `EXCLUDE USING gist (room WITH =, during WITH &&)` — prevents double booking. Impossible to do race-free in app code without locks |
| `DEFAULT` | | |
| Generated columns | Derived values | `total GENERATED ALWAYS AS (qty*price) STORED` |

**FK and indexes**: Postgres does **not** automatically index the referencing (child) column. Deleting a parent row then scans the child table for each delete → slow deletes and lock pile-ups. Always index FK columns (MySQL InnoDB auto-creates one).

**Should you use FKs at all?** Some large shops (historically GitHub, many sharded MySQL setups via Vitess) drop FKs because: they don't work across shards, they add locking (parent row gets a shared lock when child inserts), and online schema change tools struggle with them. For a single-node or moderately sized system: **use them**. Data corruption is far more expensive than the overhead.

---

## 5. Functional Dependencies {#fd}

A **functional dependency (FD)** `X → Y` means: *if two rows agree on X, they must agree on Y.* "X determines Y."

Example table `orders_flat(order_id, customer_id, customer_name, customer_city, product_id, product_name, qty)`:
- `order_id → customer_id`
- `customer_id → customer_name, customer_city`
- `product_id → product_name`
- `(order_id, product_id) → qty`

Key ideas:
- **Full dependency**: Y depends on the *whole* composite key, not a part of it.
- **Partial dependency**: Y depends on *part* of a composite key (`product_id → product_name` where key is `(order_id, product_id)`). Violates 2NF.
- **Transitive dependency**: `A → B → C` where B isn't a key (`order_id → customer_id → customer_city`). Violates 3NF.
- **Closure (X⁺)**: all attributes determined by X. X is a superkey iff X⁺ = all attributes.
- **Armstrong's axioms**: reflexivity, augmentation, transitivity — used to derive all implied FDs.

FDs are **business rules**, not something you discover from sample data. "Does city depend on zip code?" is a question for the domain (in India, one PIN code can span multiple localities; in the US, ZIPs can cross city lines).

---

## 6. Anomalies {#anomalies}

Using `orders_flat` above (one wide table):

1. **Update anomaly**: Customer moves city → must update every order row. Miss one → two "truths".
2. **Insert anomaly**: Can't record a new customer until they place an order (no order_id to fill the key).
3. **Delete anomaly**: Delete a customer's only order → you lose the customer.
4. **Redundancy**: the name is repeated per order line → storage waste, cache waste.

Normalization = decomposing tables so that **each fact is stored exactly once**, with lossless joins to reconstruct the original.

---

## 7. Normal Forms {#nf}

### 1NF — atomic values, no repeating groups
- Each cell holds one value of its domain; no `phone1, phone2, phone3` columns; no comma-separated lists.
- Bad: `tags = 'db,postgres,sql'` → can't index, can't FK, `LIKE '%sql%'` matches `mysql`.
- Fix: child table `post_tags(post_id, tag_id)`.
- Nuance: Postgres `ARRAY`/`JSONB` columns technically break strict 1NF but are fine when the value is *treated as one unit* (you don't need to join/constrain on individual elements). If you find yourself querying inside them constantly or needing FKs on elements → normalize.

### 2NF — no partial dependencies (only matters with composite keys)
`order_items(order_id, product_id, qty, product_name)` — `product_name` depends only on `product_id`. Move it to `products`.

### 3NF — no transitive dependencies
`orders(order_id, customer_id, customer_city)` — city depends on customer, not order. Move to `customers`.

Mnemonic (Bill Kent): *every non-key attribute must provide a fact about **the key, the whole key, and nothing but the key** (so help me Codd).*

### BCNF (Boyce–Codd) — every determinant is a candidate key
3NF permits an FD `X → A` where A is part of a candidate key. BCNF doesn't.

Classic example: `teaching(student, course, instructor)` with rules: each instructor teaches one course; a student takes a course from one instructor.
- FDs: `(student, course) → instructor`, `instructor → course`.
- `instructor` is a determinant but not a key → not BCNF.
- Decompose: `instructor_course(instructor, course)` + `student_instructor(student, instructor)`.
- Trade-off: the rule "(student, course) → instructor" can no longer be enforced by a single UNIQUE constraint. **BCNF decomposition is lossless but not always dependency-preserving.** This is why 3NF is the usual practical target.

### 4NF — no multi-valued dependencies
`employee_skills_languages(emp, skill, language)` where skills and languages are independent. Storing them together forces a cross product (3 skills × 2 languages = 6 rows). Split into `emp_skill` and `emp_language`.

### 5NF (PJNF) — no join dependencies
Rare in practice: a table that can only be losslessly reconstructed by joining *three or more* projections (e.g., agent–company–product rules). If you hit it, you'll know.

### 6NF — every table is key + at most one attribute
Used in temporal databases / anchor modeling / some data-warehouse designs. Not for OLTP.

### Practical target
**OLTP: 3NF/BCNF by default, then denormalize deliberately with measurements.** OLAP/warehouse: dimensional modeling (star schema) which is intentionally denormalized.

---

## 8. Denormalization {#denorm}

Denormalization trades **write complexity + consistency risk** for **read speed**.

Common, legitimate forms:
| Technique | Example | How to keep it consistent |
|---|---|---|
| Cached aggregate / counter | `posts.comment_count` | Same transaction as insert; or trigger; or async recompute. Beware hot-row contention (see transactions note) |
| Copied attribute | `order_items.unit_price` | This is actually *not* denormalization — price at time of purchase is a different fact than current price! Snapshot data is correct modeling |
| Precomputed join | `feed_items(user_id, post_id, author_name)` | Event-driven fan-out; tolerate staleness |
| Materialized view | `CREATE MATERIALIZED VIEW daily_sales …` | `REFRESH MATERIALIZED VIEW CONCURRENTLY` (Postgres); scheduled |
| JSON document column | `orders.shipping_address JSONB` | Treated as atomic value |
| Search index / cache | Elasticsearch, Redis | CDC / outbox pipeline |

Rules:
1. Normalize first. Denormalize only for a measured bottleneck.
2. Have **one source of truth**; everything else is derived and rebuildable.
3. Document who updates the copy and how staleness is bounded.

---

## 9. Relationship Patterns {#relationships}

**One-to-many**: FK on the "many" side. `orders.customer_id → customers.id`.

**One-to-one**: FK + UNIQUE on one side. Use it for: splitting rarely-used wide columns (`user_profiles`), optional subtype data, or security (separate `user_credentials` table with stricter grants).

**Many-to-many**: junction table.
```sql
CREATE TABLE enrollments (
  student_id BIGINT NOT NULL REFERENCES students(id),
  course_id  BIGINT NOT NULL REFERENCES courses(id),
  enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  grade TEXT,
  PRIMARY KEY (student_id, course_id)        -- serves "courses of a student"
);
CREATE INDEX ON enrollments (course_id, student_id); -- serves "students in a course"
```
Composite PK order matters: it only accelerates lookups by the leading column. Add the reverse index.

**Self-referencing**: `employees.manager_id → employees.id`.

**Polymorphic associations** (`comments(commentable_type, commentable_id)` à la Rails) — convenient but **no FK possible**. Alternatives:
- Exclusive arcs: `comments(post_id NULL REFERENCES posts, photo_id NULL REFERENCES photos, CHECK (num_nonnulls(post_id, photo_id) = 1))`.
- Separate join tables per parent (`post_comments`, `photo_comments`).
- A shared supertype table (`commentables(id)`) that posts and photos reference.

**Inheritance / subtypes** (Vehicle → Car, Truck):
| Strategy | Layout | Pros | Cons |
|---|---|---|---|
| Single table (STI) | one table, nullable subtype columns + `type` | Simple, no joins | Many NULLs, weak constraints |
| Class table (CTI) | base table + one table per subtype sharing PK | Normalized, constraints work | Joins |
| Concrete table | one full table per subtype | No joins per type | Querying "all vehicles" needs UNION; IDs must be globally unique |

**EAV (Entity–Attribute–Value)** `(entity_id, attribute, value)` — the "infinitely flexible" anti-pattern: no types, no constraints, horrendous queries. Use `JSONB` instead for truly dynamic attributes.

---

## 10. Hierarchies and Trees {#trees}

| Model | Structure | Read subtree | Move subtree | Notes |
|---|---|---|---|---|
| **Adjacency list** | `parent_id` | Recursive CTE | Update 1 row | Default choice; recursive CTEs are fast with an index on `parent_id` |
| **Path enumeration / materialized path** | `path = '/1/4/9/'` (or Postgres `ltree`) | `WHERE path LIKE '/1/4/%'` (index-able prefix) | Rewrite paths of all descendants | Great for read-heavy categories |
| **Nested sets** | `lft, rgt` | `WHERE lft BETWEEN p.lft AND p.rgt` | Rewrites ~half the table | Read-fast, write-awful; mostly legacy |
| **Closure table** | `(ancestor, descendant, depth)` all pairs | Simple join | Insert/delete O(depth × subtree) rows | Flexible, supports multiple queries well; O(n·depth) storage |

Recursive CTE (works in both Postgres and MySQL 8):
```sql
WITH RECURSIVE subtree AS (
  SELECT id, parent_id, name, 0 AS depth FROM categories WHERE id = 42
  UNION ALL
  SELECT c.id, c.parent_id, c.name, s.depth + 1
  FROM categories c JOIN subtree s ON c.parent_id = s.id
)
SELECT * FROM subtree;
```
Guard against cycles (`CYCLE` clause in Postgres 14+, or carry an array path and check `NOT id = ANY(path)`).

---

## 11. NULL and Three-Valued Logic {#null}

NULL means "unknown / not applicable", **not** "empty" and not "zero". Comparisons with NULL produce `UNKNOWN`, and `WHERE` keeps only `TRUE`.

| Expression | Result |
|---|---|
| `NULL = NULL` | UNKNOWN (not TRUE!) |
| `NULL <> 1` | UNKNOWN |
| `NULL AND FALSE` | FALSE |
| `NULL AND TRUE` | UNKNOWN |
| `NULL OR TRUE` | TRUE |
| `NOT NULL` (the value) | UNKNOWN |

Traps:
1. `WHERE col = NULL` never matches. Use `IS NULL`.
2. `WHERE status <> 'deleted'` **excludes rows where status IS NULL.**
3. **`NOT IN` with a NULL in the subquery returns no rows at all**:
   ```sql
   SELECT * FROM users WHERE id NOT IN (SELECT manager_id FROM teams); -- if any manager_id IS NULL → empty result
   ```
   Use `NOT EXISTS` instead — it's NULL-safe and usually plans better (anti-join).
4. Aggregates ignore NULLs: `COUNT(col)` ≠ `COUNT(*)`; `AVG(col)` divides by non-null count.
5. `SUM` of zero rows is NULL, not 0 → `COALESCE(SUM(x), 0)`.
6. NULL-safe equality: Postgres `IS NOT DISTINCT FROM`; MySQL `<=>`.
7. Sorting: Postgres puts NULLs **last** in ASC by default; MySQL puts them **first**. Use `NULLS FIRST/LAST` (Postgres) explicitly.
8. UNIQUE allows many NULLs (see constraints).
9. `CONCAT` vs `||`: in Postgres `'a' || NULL` is NULL; `concat('a', NULL)` = `'a'`.

---

## 12. Relational Algebra {#algebra}

The optimizer turns SQL into a tree of these operators, then rewrites it.

| Operator | Symbol | SQL |
|---|---|---|
| Selection | σ | `WHERE` |
| Projection | π | `SELECT col list` |
| Cartesian product | × | `FROM a, b` / `CROSS JOIN` |
| Join | ⋈ | `JOIN … ON` |
| Union / Intersect / Difference | ∪ ∩ − | `UNION`, `INTERSECT`, `EXCEPT` |
| Rename | ρ | `AS` |
| Grouping/aggregation | γ | `GROUP BY` |
| Semi-join | ⋉ | `EXISTS`, `IN (subquery)` |
| Anti-join | ▷ | `NOT EXISTS` |

Classic rewrites the optimizer does:
- **Predicate pushdown**: apply `WHERE` filters as early as possible (before joins).
- **Projection pushdown**: carry only needed columns.
- **Join reordering**: joins are associative/commutative — choose the cheapest order (this is an NP-hard search; Postgres switches to a genetic optimizer, GEQO, beyond `geqo_threshold` = 12 tables).
- **Subquery unnesting / decorrelation**: turn `IN (SELECT …)` into a semi-join.

Logical order of SQL evaluation (*not* the written order):
```
FROM / JOIN → WHERE → GROUP BY → HAVING → window functions → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET
```
This explains why you can't use a `SELECT` alias in `WHERE` (it doesn't exist yet), but you can in `ORDER BY` (Postgres and MySQL allow it).

---

## 13. Checklist & Interview Questions {#checklist}

**Schema review checklist**
- [ ] Every table has a PK; natural uniqueness is enforced with UNIQUE.
- [ ] Columns NOT NULL unless absence is a meaningful state.
- [ ] Money as `NUMERIC(p,s)`/`DECIMAL` or integer minor units — **never FLOAT**.
- [ ] Timestamps with time zone (`TIMESTAMPTZ` in Postgres; store UTC in MySQL `DATETIME`, or use `TIMESTAMP` aware of its 2038 limit).
- [ ] FKs declared and indexed; `ON DELETE` chosen deliberately.
- [ ] CHECK constraints for invariants (non-negative, ranges, enums).
- [ ] No comma-separated lists, no EAV, no `phone1..phone3`.
- [ ] Every denormalized value has a documented owner and refresh path.

**Questions to be able to answer**
1. Explain 1NF/2NF/3NF/BCNF with an example. Why is 3NF the usual target rather than BCNF?
2. Natural vs surrogate keys; integer vs UUID (and why random UUIDs hurt B-trees).
3. Why does `NOT IN` with NULLs return nothing? How do you fix it?
4. Model an M:N relationship with attributes on the relationship.
5. How would you store a category tree that's read 1000× more than written?
6. How do you prevent double-booking a meeting room without race conditions? (EXCLUDE constraint / serializable / lock on room row)
7. When is denormalization justified? How do you keep a counter column correct?
