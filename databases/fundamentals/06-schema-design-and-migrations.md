# Practical Schema Design and Zero-Downtime Migrations

> Normalization theory is in `01-relational-model-and-normalization.md`. This note is about the day-to-day decisions backend engineers make: data types, soft deletes, audit trails, multi-tenancy, enums, money, and changing a live schema without an outage.

## Table of Contents
1. [Choosing Data Types](#types)
2. [Standard Columns Every Table Should (Maybe) Have](#standard)
3. [Soft Deletes vs Hard Deletes vs Archiving](#softdelete)
4. [Audit Trails and History Tables](#audit)
5. [Enums: Native, Lookup Table, or CHECK?](#enums)
6. [Money, Quantities, and Precision](#money)
7. [Multi-Tenancy Models](#tenancy)
8. [JSON Columns: When They're Right](#json)
9. [Designing for Scale from Day One (without over-engineering)](#scale)
10. [Migrations: Tools and Principles](#migrations)
11. [Zero-Downtime Migration Playbook (Expand/Contract)](#zdt)
12. [Dangerous Operations Table (PG & MySQL)](#danger)
13. [Online Schema Change Tools](#tools)
14. [Backfills](#backfill)
15. [Checklist](#checklist)

---

## 1. Data Types {#types}

| Data | PostgreSQL | MySQL | Notes |
|---|---|---|---|
| Surrogate ID | `BIGINT GENERATED ALWAYS AS IDENTITY` | `BIGINT UNSIGNED AUTO_INCREMENT` | Don't use `INT`: 2.1B limit is reached more often than people think (Basecamp, Parse incidents). Migrating INT→BIGINT on a huge table is painful |
| Public/global ID | `UUID` (16 B) | `BINARY(16)` (+ `UUID_TO_BIN(uuid, 1)` swap flag for time-ordering v1) | Prefer UUIDv7 |
| Short string | `TEXT` + CHECK, or `VARCHAR(n)` | `VARCHAR(n)` | |
| Email | `citext` or `TEXT` + lower() unique index | `VARCHAR(320)` with `_ci` collation | |
| Boolean | `BOOLEAN` | `TINYINT(1)` / `BOOL` | |
| Money | `NUMERIC(19,4)` or `BIGINT` minor units | `DECIMAL(19,4)` or `BIGINT` | Never FLOAT/DOUBLE. PG `money` type is locale-dependent; avoid |
| Timestamp | `TIMESTAMPTZ` | `DATETIME(6)` in UTC (or `TIMESTAMP` with 2038 caveat) | Microsecond precision |
| Date only | `DATE` | `DATE` | Birthdays, business dates |
| Duration | `INTERVAL` | `BIGINT` seconds/ms | |
| IP address | `INET` / `CIDR` | `VARBINARY(16)` + `INET6_ATON` | |
| JSON | `JSONB` (not `JSON` — that keeps text, no indexing) | `JSON` | |
| Enum-ish | TEXT + CHECK, or lookup table, or `ENUM` type | `ENUM(...)`, or lookup table | See §5 |
| Large binary | Don't (object storage). If you must: `BYTEA` | `BLOB` variants | |
| Arrays | `BIGINT[]`, `TEXT[]` | JSON | |
| Ranges | `tstzrange`, `int4range`, multiranges | — | Bookings, validity periods |
| Geospatial | PostGIS `geography(Point, 4326)` | `POINT SRID 4326` + SPATIAL index | |
| Vectors | `vector(1536)` (pgvector) | `VECTOR` (MySQL 9 / HeatWave) | Embeddings |

Smaller types → more rows per page → more cached. But don't micro-optimize at the cost of future overflow (`SMALLINT` for a counter that will exceed 32,767).

Column ordering in PG affects padding/alignment: put fixed-width 8-byte columns first, then 4, 2, 1, then variable-length — can save 10–20% on narrow tables.

---

## 2. Standard Columns {#standard}

```sql
CREATE TABLE orders (
  id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  public_id     UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,   -- or uuidv7() on PG 18+
  tenant_id     BIGINT NOT NULL REFERENCES tenants(id),
  status        TEXT NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled')),
  total_minor   BIGINT NOT NULL CHECK (total_minor >= 0),
  currency      CHAR(3) NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  version       INT NOT NULL DEFAULT 0,          -- optimistic locking
  deleted_at    TIMESTAMPTZ                      -- only if soft delete is truly needed
);
CREATE INDEX ON orders (tenant_id, created_at DESC);
```
- `updated_at`: MySQL has `ON UPDATE CURRENT_TIMESTAMP`; PG needs a trigger (or set it in the app/ORM).
- `created_by`/`updated_by` if you need attribution (or put it in the audit log).
- Idempotency key column with UNIQUE for externally triggered creations (payments, webhooks).

---

## 3. Soft Deletes {#softdelete}

`deleted_at IS NULL` filters everywhere. Pros: undo, audit, referential "history". Cons — often underestimated:
- Every query, join, unique constraint, and index must account for it; one missed filter = deleted data shown (security/privacy bug).
- UNIQUE(email) blocks re-registering a deleted email → need partial unique index `WHERE deleted_at IS NULL` (PG) or a generated column trick in MySQL (`UNIQUE(email, deleted_marker)`).
- FKs to soft-deleted rows stay "valid".
- Tables grow forever; **GDPR/DPDP "right to erasure" requires actual deletion** of personal data.

Alternatives:
- **Hard delete + archive table / audit log** (move row to `orders_deleted` in the same transaction, or capture via CDC).
- **Status column** when "deleted" is a real business state (`cancelled`, `closed`) — model the domain instead of a generic flag.
- Views (`CREATE VIEW active_users AS … WHERE deleted_at IS NULL`) or PG RLS to make the filter hard to forget.

---

## 4. Audit Trails {#audit}

Options from simplest to most powerful:
1. `created_at/updated_at/updated_by` — no history.
2. **Trigger-based audit table**: generic `audit_log(table_name, row_pk, op, old JSONB, new JSONB, actor, at)`. Actor passed via `SET LOCAL app.user_id = '…'` (PG) and read with `current_setting('app.user_id', true)`. Extensions: `pgaudit` (statement auditing for compliance), `supa_audit`, `temporal_tables`.
3. **System-versioned (temporal) tables**: SQL:2011 `PERIOD FOR SYSTEM_TIME` — MariaDB, SQL Server, Db2 support natively; PG via extension/triggers; MySQL no.
4. **CDC to a log store** (Debezium → Kafka → warehouse): complete history without touching the OLTP schema.
5. **Event sourcing**: the event log *is* the source of truth (see messaging notes).

Keep audit data append-only and ideally in a separate schema with restricted grants.

---

## 5. Enums {#enums}

| Approach | Add value | Remove/rename | Ordering | Notes |
|---|---|---|---|---|
| PG `CREATE TYPE … AS ENUM` | `ALTER TYPE ADD VALUE` (cheap) | Painful (recreate type) | By declaration order | 4 bytes |
| MySQL `ENUM` | `ALTER TABLE` — instant only if appended at the end (8.0) | Table rebuild | By index | Sorting by internal number surprises people |
| `TEXT` + `CHECK (x IN (...))` | Drop & re-add CHECK (`NOT VALID` then `VALIDATE` in PG) | Same | Alphabetical | Flexible, explicit |
| Lookup table + FK | INSERT row | UPDATE row | Add `sort_order` column | Can hold metadata (display name, is_active); joins |
| `SMALLINT` codes mapped in app | Free | Free | — | Unreadable in SQL; avoid unless extremely hot |

Default recommendation: TEXT + CHECK or a lookup table. Use native enums when values are truly stable.

---

## 6. Money and Precision {#money}

- **Floats are binary** — `0.1 + 0.2 = 0.30000000000000004`. Never for money.
- Use `NUMERIC/DECIMAL(19,4)` (exact decimal) or **integer minor units** (paise/cents) in `BIGINT` (what Stripe does). Store the **currency** next to the amount; minor-unit exponent varies (JPY 0, INR 2, KWD 3).
- Decide rounding mode explicitly (banker's rounding / half-even for accounting) and where it happens.
- Ledgers: **double-entry, append-only**: never UPDATE a balance in place as the only record; insert entries (debit/credit pairs summing to zero) and derive/cache balances. Corrections are new reversing entries.
- Exchange rates: store the rate used with the transaction.

---

## 7. Multi-Tenancy Models {#tenancy}

| Model | Isolation | Ops cost | Scale | Notes |
|---|---|---|---|---|
| **Shared tables + `tenant_id` column** | Logical (app or RLS) | Lowest | Best (shard by tenant_id later — Citus, Vitess) | Every query/index must lead with tenant_id. Enforce with PG **Row-Level Security** as defense in depth. Noisy neighbors |
| **Schema per tenant** (PG schemas) | Better | Migrations × N schemas; catalog bloat beyond ~thousands | Medium | `search_path` per connection — pooling complications |
| **Database per tenant** | Strong (separate backups, encryption keys, regions) | High: N migrations, N connection pools | Limited by fleet automation | Common for enterprise/regulated customers |
| **Hybrid** | | | | Small tenants shared, big/regulated tenants dedicated |

Always: include `tenant_id` in every composite index and unique constraint (`UNIQUE (tenant_id, email)`), and in FKs (`FOREIGN KEY (tenant_id, customer_id) REFERENCES customers (tenant_id, id)` prevents cross-tenant references).

---

## 8. JSON Columns {#json}

Good uses:
- Truly variable attributes (product specs varying per category), user preferences, third-party webhook payloads, feature flag config.
- Data always read/written as a whole document.

Bad uses:
- Fields you filter/join/aggregate on constantly → make them columns.
- Relationships (IDs inside JSON can't have FKs).
- Things needing constraints (types, NOT NULL) — although PG CHECK with `jsonb_typeof`/JSON Schema extensions help.

Hybrid: core fields as columns + `attributes JSONB`. Promote fields to columns when they become important (generated columns make this easy).

---

## 9. Designing for Scale Early {#scale}

Cheap now, expensive later:
- `BIGINT` IDs.
- `tenant_id`/`user_id` on child tables you may shard by (even if derivable via join).
- Avoid cross-entity transactions where not essential (future shard boundaries).
- Time-ordered keys for append-heavy tables (and plan partitioning by time for logs/events).
- UTC timestamps, `utf8mb4`.
- Don't put blobs in the DB.
- Idempotency keys for external-facing writes.

Not worth it on day one: sharding, microservice-per-table splits, CQRS everywhere.

---

## 10. Migrations: Tools and Principles {#migrations}

Tools: Flyway, Liquibase (JVM); Alembic (Python/SQLAlchemy); Django migrations; Rails ActiveRecord; Prisma Migrate, Knex, TypeORM, Drizzle (Node); golang-migrate, Atlas, goose (Go); sqitch.

Principles:
1. **Versioned, ordered, immutable** migration files in git; never edit an applied migration.
2. Each migration runs in a transaction where possible (PG supports transactional DDL; **MySQL doesn't** — a failed multi-statement migration leaves a half-applied state; keep one DDL per migration in MySQL).
3. **Schema changes and code deploys are decoupled**: during a rolling deploy, old and new app versions run simultaneously against the same schema → every migration must be compatible with both N and N+1 code.
4. Migrations run by a single process (CI step or init job), guarded by a lock (Flyway lock table, PG advisory lock).
5. Down migrations: nice in dev, rarely safe in prod for destructive changes; prefer **roll-forward** fixes.
6. Test migrations against production-sized data (timing!) — staging restore from snapshot.
7. Set `lock_timeout` (PG) / `lock_wait_timeout` (MySQL) short in migrations and retry, so a migration never sits in the lock queue blocking all traffic.

---

## 11. Zero-Downtime Playbook: Expand → Migrate → Contract {#zdt}

### Renaming a column (`name` → `full_name`)
Never `ALTER TABLE RENAME COLUMN` with live traffic — old code instantly breaks.
1. **Expand**: add `full_name` (nullable). Deploy code that **writes both** columns, reads `name`.
2. **Backfill** `full_name = name` in batches.
3. Deploy code that **reads `full_name`** (still writes both).
4. Deploy code that stops writing `name`.
5. **Contract**: drop `name` (after confirming nothing reads it; in Rails/Django mark column ignored first so ORM caches don't reference it).

### Adding a NOT NULL column
- PG 11+: `ADD COLUMN x INT NOT NULL DEFAULT 0` is **instant** (default stored in catalog) for non-volatile defaults. Volatile default (`DEFAULT random()` / `clock_timestamp()`) rewrites the table.
- MySQL 8.0.12+: `ALGORITHM=INSTANT` for adding columns (at the end; any position since 8.0.29).
- Without a default: add nullable → backfill → add constraint.
  - PG: `ALTER TABLE t ADD CONSTRAINT x_nn CHECK (x IS NOT NULL) NOT VALID;` → `VALIDATE CONSTRAINT x_nn;` (only SHARE UPDATE EXCLUSIVE lock) → `ALTER COLUMN x SET NOT NULL` (PG 12+ uses the validated check to skip the scan) → drop the check.

### Adding a foreign key (PG)
```sql
ALTER TABLE orders ADD CONSTRAINT fk_cust FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;  -- fast
ALTER TABLE orders VALIDATE CONSTRAINT fk_cust;   -- scans, but doesn't block writes
```

### Adding an index
PG: `CREATE INDEX CONCURRENTLY` (two table scans, waits for existing transactions; if it fails, drop the INVALID index and retry). MySQL: online by default (`ALGORITHM=INPLACE, LOCK=NONE`), but beware replication lag: the replica applies the DDL single-threaded *after* the primary finishes → lag equal to build time.

### Changing a column type (e.g., INT → BIGINT on a big table)
Usually requires a rewrite. Strategy: add new column, dual-write via trigger, backfill in batches, swap in a quick transaction (rename both), drop old. Or use gh-ost/pt-osc (MySQL), or `pgroll`/`pg-osc`/logical replication to a new table (PG). Plan this years before you hit 2^31.

### Splitting a table / moving data to a new service
Dual-write or CDC to the new store → backfill → shadow-read and compare → switch reads → stop old writes → delete. (Same expand/contract, at system level.)

### Dropping a column
Deploy code that no longer references it first (including ORM `SELECT *` model caches), then drop. PG drop column is instant (marks dropped; space reclaimed on rewrite).

---

## 12. Dangerous Operations {#danger}

| Operation | PostgreSQL | MySQL 8 InnoDB |
|---|---|---|
| Add nullable column | Instant | Instant |
| Add column with constant default | Instant (11+) | Instant |
| Add column with volatile default | **Rewrite** | — |
| Drop column | Instant (metadata) | Instant (8.0.29+) else rebuild |
| Rename column | Instant (but breaks old code) | Instant |
| Change type (widening varchar) | Instant if just increasing VARCHAR length / removing limit | In-place if same length-byte count (≤255 → ≤255) else **copy** |
| Change type (int→bigint) | **Rewrite + ACCESS EXCLUSIVE** | **Copy table** |
| Add index | Blocks writes — use CONCURRENTLY | Online (INPLACE) |
| Add FK | Use NOT VALID + VALIDATE | Online with `foreign_key_checks=0` trick; otherwise copy |
| Add CHECK | NOT VALID + VALIDATE | Validates by scan (copy) |
| SET NOT NULL | Scan under exclusive lock (use CHECK trick) | Rebuild |
| Add PK | Builds unique index (do CONCURRENTLY first, then `ADD PRIMARY KEY USING INDEX`) | Rebuild |
| `VACUUM FULL` / `CLUSTER` | **Rewrite + ACCESS EXCLUSIVE** — use `pg_repack` | `OPTIMIZE TABLE` = online rebuild |
| Any DDL | Waits for ACCESS EXCLUSIVE → **queues every later query** | Waits for **metadata lock** → same queueing problem |

Always check the exact version's docs; MySQL "INSTANT"/"INPLACE" support changes per minor release.

---

## 13. Online Schema Change Tools {#tools}

- **gh-ost** (GitHub, MySQL): triggerless; creates a ghost table, copies rows in chunks, tails the **binlog** to apply ongoing changes, then atomic cut-over. Throttles on replica lag. Pausable.
- **pt-online-schema-change** (Percona, MySQL): trigger-based copy. Simpler, but triggers add write load and lock contention.
- **Vitess/PlanetScale online DDL** — gh-ost-like with revertible migrations.
- **pg_repack**: rebuild bloated tables/indexes online in PG.
- **pgroll** (Xata), **Reshape**: expand/contract automation for PG with versioned views.
- **pg-osc** (Shopify): gh-ost-style for PG using triggers + logical replication-ish copy.

---

## 14. Backfills {#backfill}

```sql
-- Batch by primary key range; never one giant UPDATE
UPDATE users SET full_name = name
WHERE id > :last_id AND id <= :last_id + 5000 AND full_name IS NULL;
```
- Loop in app/script with small sleeps; monitor replication lag and abort/throttle above threshold.
- Batch by PK range (not OFFSET).
- Idempotent (`AND full_name IS NULL`) so it can resume after interruption.
- Prefer commits per batch (short transactions → less bloat/undo, fewer lock waits).
- For PG, consider running `VACUUM (ANALYZE)` during/after big backfills.

---

## 15. Checklist {#checklist}

- [ ] Migration is backward compatible with currently deployed code.
- [ ] Lock timeout set; retry plan exists.
- [ ] Timing measured on production-size copy.
- [ ] No table rewrite under exclusive lock on large tables.
- [ ] Indexes created concurrently/online.
- [ ] Backfill batched, idempotent, throttled on replica lag.
- [ ] Contract step scheduled (don't leave dead columns forever).
- [ ] Rollback/roll-forward plan written down.
