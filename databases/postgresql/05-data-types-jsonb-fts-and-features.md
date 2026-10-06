# PostgreSQL Data Types, JSONB, Full-Text Search, and Power Features

> Postgres can cover jobs that would otherwise need extra systems. Used well, its feature set lets one database do the work of three.

## Table of Contents
1. [Numeric, Text, Temporal Types — Gotchas](#basic)
2. [Identity, Sequences, UUIDs](#ids)
3. [Arrays](#arrays)
4. [JSON vs JSONB, Operators, Indexing, SQL/JSON](#jsonb)
5. [Range and Multirange Types](#ranges)
6. [Enums, Domains, Composite Types](#custom)
7. [Full-Text Search](#fts)
8. [Generated Columns, Views, Materialized Views](#views)
9. [Functions, Procedures, Triggers, PL/pgSQL](#plpgsql)
10. [LISTEN/NOTIFY](#notify)
11. [Postgres as a Queue](#queue)
12. [Other Gems: COPY, FDW, Unlogged Tables, Advisory Locks, TABLESAMPLE](#gems)
13. [Interview Questions](#qa)

---

## 1. Basic Types and Gotchas {#basic}

**Numbers**
- `smallint` (2 B), `integer` (4 B), `bigint` (8 B).
- `numeric(p,s)`: exact decimal, arbitrary precision, slower than integers. Use it for money (or bigint minor units).
- `real`/`double precision`: IEEE floats. Use them for measurements, never money.
- `serial`/`bigserial`: legacy shorthand for an integer column plus a sequence default. Prefer `GENERATED … AS IDENTITY`.

**Text**
- `text` = `varchar` (no limit) = `varchar(n)` (limit check). They perform the same. `char(n)` pads with blanks; avoid it.
- `citext` extension: case-insensitive text.
- Strings are stored with a 1- or 4-byte header. Values over ~2 KB get TOASTed (compressed/out of line).

**Time**
- `timestamptz`: **always** use it for instants. It's stored as UTC microseconds and displayed in the session `TimeZone`.
- `timestamp` (without tz): wall-clock only. A common bug is storing `now()` into it: `now()` returns timestamptz, and the conversion uses the session timezone.
- `date`, `time` (rarely useful), `interval` (months, days, and microseconds are kept separately, so `'1 month'` ≠ `'30 days'`).
- `now()`/`current_timestamp` = transaction start; `statement_timestamp()`; `clock_timestamp()` = actual current time.
- Infinity values: `'infinity'::timestamptz`. They're useful for open-ended validity ranges.

**Boolean**: `true/false/null`; accepts `'t'`, `'yes'`, `'on'`, `'1'`.

**Binary**: `bytea` (hex output).

**Network**: `inet`, `cidr`, `macaddr` with operators (`<<` contained by, `&&` overlap).

**Money type**: avoid it, because its output depends on `lc_monetary`.

---

## 2. Identity, Sequences, UUIDs {#ids}

```sql
CREATE TABLE t (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,     -- ALWAYS: rejects manual inserts unless OVERRIDING SYSTEM VALUE
  ...
);
```
- Sequences are **non-transactional**. Rolled-back inserts consume values, so **gaps are normal**. Never use them for "invoice numbers must be gapless". For that you need a counter row updated in the same transaction (which serializes writers).
- Sequence caching (`CACHE 50`) per session improves concurrency, at the cost of more gaps and non-monotonic order across sessions.
- After bulk-loading explicit IDs: `SELECT setval(pg_get_serial_sequence('t','id'), max(id)) FROM t;`
- UUIDs: `gen_random_uuid()` (v4, built in since PG 13), `uuidv7()` (PG 18). The `uuid` type is 16 bytes. Never store UUIDs as text.

---

## 3. Arrays {#arrays}

```sql
CREATE TABLE posts (id bigint PRIMARY KEY, tags text[] NOT NULL DEFAULT '{}');
INSERT INTO posts VALUES (1, ARRAY['pg','sql']);
SELECT * FROM posts WHERE 'pg' = ANY(tags);          -- not GIN-indexable
SELECT * FROM posts WHERE tags @> ARRAY['pg'];       -- GIN-indexable
SELECT unnest(tags) FROM posts;                       -- array → rows
SELECT array_agg(name ORDER BY name) FROM users;      -- rows → array
SELECT * FROM users WHERE id = ANY($1::bigint[]);     -- pass list as one parameter
```
Use arrays for small, bounded, value-like lists (tags, flags). Use a child table when elements need FKs, attributes, or independent querying at scale.

---

## 4. JSON and JSONB {#jsonb}

| | `json` | `jsonb` |
|---|---|---|
| Storage | Exact text | Decomposed binary |
| Key order / duplicates / whitespace | Preserved | Not preserved; last duplicate key wins |
| Insert speed | Faster | Slightly slower (parse) |
| Query speed | Re-parse every access | Fast |
| Indexing | Expression only | GIN + expression |
| Use | Logging raw payloads byte-exact | **Almost always** |

### Operators
```sql
data -> 'user'              -- jsonb value
data ->> 'email'            -- text value
data #> '{user,address,city}'   -- path → jsonb
data #>> '{user,address,city}'  -- path → text
data @> '{"status":"active"}'   -- contains (GIN)
data ? 'email'              -- key exists (GIN jsonb_ops)
data ?| array['a','b']      -- any key exists
data ?& array['a','b']      -- all keys exist
data @? '$.items[*] ? (@.qty > 10)'   -- jsonpath exists (GIN)
data @@ '$.price > 100'               -- jsonpath predicate
data['user']['email']       -- subscripting (PG 14+), also for UPDATE SET data['k'] = '"v"'
data - 'key'                -- delete key
data || '{"k": 1}'          -- merge (shallow)
jsonb_set(data, '{user,city}', '"Pune"', true)
```

### Functions
`jsonb_build_object`, `jsonb_agg`, `jsonb_object_agg`, `jsonb_each`, `jsonb_array_elements`, `jsonb_to_recordset`, `jsonb_path_query`, `jsonb_strip_nulls`, `jsonb_typeof`, `jsonb_pretty`.

### SQL/JSON standard (PG 16/17)
```sql
SELECT JSON_VALUE(data, '$.user.email'), JSON_QUERY(data, '$.items'), JSON_EXISTS(data, '$.coupon');
SELECT jt.* FROM orders,
  JSON_TABLE(data, '$.items[*]' COLUMNS (sku text PATH '$.sku', qty int PATH '$.qty')) AS jt;   -- PG 17
SELECT 'abc' IS JSON;
```

### Building API responses in SQL
```sql
SELECT jsonb_build_object(
  'id', u.id, 'name', u.name,
  'orders', COALESCE((SELECT jsonb_agg(jsonb_build_object('id', o.id, 'total', o.total) ORDER BY o.created_at DESC)
                      FROM orders o WHERE o.user_id = u.id), '[]'::jsonb)
) FROM users u WHERE u.id = $1;
```
One round trip, no N+1. PostgREST and Hasura use this pattern.

### JSONB pitfalls
- No statistics on keys inside JSONB, so the planner guesses selectivity (often 0.1% for `@>`). Bad plans follow. Promote hot keys to real or generated columns.
- Updating one key rewrites the whole (TOASTed) document. Large documents with frequent small updates cause write amplification and bloat.
- Numbers in jsonb are `numeric`. Cast them when extracting: `(data->>'qty')::int`.
- Validation: CHECK constraints (`CHECK (jsonb_typeof(data->'qty') = 'number')`) or the `pg_jsonschema` extension.

---

## 5. Range and Multirange Types {#ranges}

`int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange`, and multiranges (PG 14+, e.g. `tstzmultirange`).
```sql
SELECT '[2025-01-01, 2025-02-01)'::daterange @> '2025-01-15'::date;   -- true
SELECT tstzrange(now(), now() + interval '1h') && tstzrange(...);     -- overlap
-- Bounds: [ inclusive, ( exclusive; canonical form for discrete types is [)
```
Use cases: bookings, prices valid during a period, versioned rows (`valid_during tstzrange`), shift schedules. Pair them with GiST indexes and EXCLUDE constraints. PG 18 adds temporal `PRIMARY KEY/UNIQUE … WITHOUT OVERLAPS` and `PERIOD` foreign keys.

---

## 6. Enums, Domains, Composite Types {#custom}

```sql
CREATE TYPE order_status AS ENUM ('pending','paid','shipped');
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'paid';

CREATE DOMAIN email AS citext CHECK (VALUE ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$');
CREATE DOMAIN positive_money AS numeric(19,4) CHECK (VALUE >= 0);

CREATE TYPE address AS (line1 text, city text, pincode text);
```
Domains centralize validation that would otherwise be repeated as CHECKs on many tables.

---

## 7. Full-Text Search {#fts}

Concepts:
- `tsvector`: a document normalized into **lexemes** with positions. `to_tsvector('english', 'The quick brown foxes jumped')` → `'brown':3 'fox':4 'jump':5 'quick':2` (stop words removed, words stemmed).
- `tsquery`: a search expression: `to_tsquery('english', 'fox & (jump | leap) & !cat')`, `plainto_tsquery`, `phraseto_tsquery`, `websearch_to_tsquery` (Google-like syntax: quotes, `-`, `or`). Use the last one for user input.
- Match: `tsv @@ query`.
- Ranking: `ts_rank`, `ts_rank_cd` (cover density). Weights A–D via `setweight(to_tsvector(title),'A') || setweight(to_tsvector(body),'B')`.
- Highlighting: `ts_headline` (expensive, so only run it on the final page of results).
- Dictionaries/configurations: `english`, `simple` (no stemming), plus custom synonyms and thesaurus. `unaccent` extension.

```sql
ALTER TABLE articles ADD COLUMN search tsvector
  GENERATED ALWAYS AS (setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
                       setweight(to_tsvector('english', coalesce(body,'')),  'B')) STORED;
CREATE INDEX articles_search_idx ON articles USING gin (search);

SELECT id, title, ts_rank(search, q) AS rank
FROM articles, websearch_to_tsquery('english', $1) q
WHERE search @@ q
ORDER BY rank DESC LIMIT 20;
```

PG FTS vs Elasticsearch:
| PG FTS is enough when | Use a search engine when |
|---|---|
| Up to millions of docs, simple relevance | Advanced relevance tuning (BM25 variants, boosting, learning-to-rank) |
| Transactional consistency with data matters | Typo tolerance, autocomplete-as-you-type, synonyms at scale |
| You want one less system | Facets/aggregations over huge corpora, multi-language analyzers, log search |

(ParadeDB's `pg_search` brings BM25 and Tantivy into Postgres. pg_trgm handles typo tolerance.)

---

## 8. Generated Columns, Views, Materialized Views {#views}

```sql
-- Generated columns (STORED computed on write; PG 18 adds VIRTUAL computed on read, and makes it the default)
ALTER TABLE orders ADD COLUMN total numeric GENERATED ALWAYS AS (qty * unit_price) STORED;

-- Views: stored queries; simple views are auto-updatable
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;
-- security_barrier / security_invoker (PG 15) options matter with RLS

-- Materialized views: stored results
CREATE MATERIALIZED VIEW daily_revenue AS
  SELECT date_trunc('day', created_at) d, sum(total) FROM orders GROUP BY 1;
CREATE UNIQUE INDEX ON daily_revenue (d);                   -- required for CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;       -- readers not blocked; diff-based
```
Materialized views refresh **fully** (there's no built-in incremental refresh). For incremental maintenance use the `pg_ivm` extension, summary tables maintained by triggers or jobs, or TimescaleDB continuous aggregates. Schedule refreshes with `pg_cron`.

---

## 9. Functions, Procedures, Triggers {#plpgsql}

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END $$;

CREATE TRIGGER trg_orders_updated BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```
- Volatility labels (`IMMUTABLE`/`STABLE`/`VOLATILE`) affect optimization and index eligibility. Mislabeling a function as IMMUTABLE causes wrong results.
- `SECURITY DEFINER` runs with the owner's privileges. Always `SET search_path = pg_catalog, public` (or similar) on such functions to prevent hijacking.
- **Procedures** (`CREATE PROCEDURE`, PG 11+) can `COMMIT` inside, which is useful for batch jobs run with `CALL`.
- Languages: SQL, PL/pgSQL, PL/Python (untrusted), PL/v8, PL/Rust.
- Triggers: BEFORE/AFTER/INSTEAD OF; ROW/STATEMENT; transition tables (`REFERENCING NEW TABLE AS new_rows`) for efficient statement-level triggers; constraint triggers (deferrable).

Trigger caution: hidden logic, harder debugging, per-row overhead, surprises during bulk loads. Good uses: `updated_at`, audit logging, enforcing invariants that can't be expressed as constraints, maintaining denormalized summaries. Avoid putting business workflows in triggers.

---

## 10. LISTEN/NOTIFY {#notify}

```sql
LISTEN order_events;                                     -- session A
NOTIFY order_events, '{"id": 42, "status": "paid"}';     -- session B (or pg_notify('order_events', payload))
```
- Notifications are delivered **at commit** and only to currently connected listeners. They're not persisted, so missed messages are gone. Payload limit is 8000 bytes.
- Use them to wake up workers or invalidate caches ("something changed, re-check the table"), with the table as the source of truth.
- A global lock during commit serializes NOTIFY-ing transactions, which limits throughput at high rates.
- Doesn't work through PgBouncer in transaction mode (it needs a dedicated session).

---

## 11. Postgres as a Queue {#queue}

```sql
CREATE TABLE jobs (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  queue text NOT NULL,
  payload jsonb NOT NULL,
  run_at timestamptz NOT NULL DEFAULT now(),
  attempts int NOT NULL DEFAULT 0,
  locked_until timestamptz,
  status text NOT NULL DEFAULT 'ready'
);
CREATE INDEX ON jobs (queue, run_at) WHERE status = 'ready';

-- Dequeue a batch without workers blocking each other
WITH next AS (
  SELECT id FROM jobs
  WHERE queue = 'email' AND status = 'ready' AND run_at <= now()
  ORDER BY run_at
  LIMIT 10
  FOR UPDATE SKIP LOCKED
)
UPDATE jobs j SET status = 'running', locked_until = now() + interval '5 min', attempts = attempts + 1
FROM next WHERE j.id = next.id
RETURNING j.*;
```
- Pros: transactional enqueue with your business data (no dual-write problem), no extra infrastructure. This fits most apps up to thousands of jobs per second.
- Cons: MVCC churn means you need aggressive autovacuum on the jobs table, polling latency (pair it with LISTEN/NOTIFY), and it's not built for fan-out/pub-sub or huge throughput.
- Libraries: **pgmq**, **River** (Go), **Oban** (Elixir), **Graphile Worker** (Node), **good_job / Solid Queue** (Rails), **Procrastinate** (Python), pg-boss (Node).

---

## 12. Other Gems {#gems}

- **COPY**: the fastest import/export path. `\copy` in psql runs it client-side.
- **Foreign Data Wrappers**: `postgres_fdw` (query remote PG with pushdown), `mysql_fdw`, `file_fdw`, `parquet_fdw`. Useful for migrations and federated queries.
- **Unlogged tables**: no WAL, so writes are ~2× faster, but they're **truncated after a crash** and not replicated. Use them for staging and cache-like data.
- **Temporary tables**: session-scoped. Heavy use causes catalog bloat.
- **Advisory locks**: `pg_try_advisory_xact_lock(hashtext('job:daily-report'))` ensures only one instance runs a cron job.
- **TABLESAMPLE**: `SELECT * FROM big TABLESAMPLE SYSTEM (1);` reads ~1% of pages, handy for fast approximate analytics.
- **Row constructors / row comparison**: `(a, b) > (1, 2)` for keyset pagination.
- **`DISTINCT ON`**, **`FILTER`**, **`LATERAL`**, **`GROUPING SETS`**, **`generate_series`**, **`WITH ORDINALITY`**.
- **Event triggers**: fire on DDL (audit schema changes, block dangerous DDL).
- **pg_cron**: cron inside the database.
- **`RETURNING OLD.*, NEW.*`** (PG 18) on UPDATE/DELETE/MERGE.

---

## 13. Interview Questions {#qa}

1. json vs jsonb: which and why? How do you index a JSONB field for equality vs containment?
2. Why do sequences have gaps? How would you implement gapless invoice numbers?
3. Why is `timestamp without time zone` dangerous for event times?
4. Implement a job queue in Postgres that multiple workers can consume safely.
5. When is Postgres full-text search enough, and when would you move to Elasticsearch?
6. How do materialized views refresh? What does CONCURRENTLY require?
7. What are the risks of LISTEN/NOTIFY as a messaging system?
8. What is a SECURITY DEFINER function and what's the search_path risk?
