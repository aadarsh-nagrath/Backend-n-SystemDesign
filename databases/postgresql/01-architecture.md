# PostgreSQL Architecture: Processes, Memory, Storage

> How a Postgres server is put together. Knowing this explains connection limits, memory settings, why VACUUM exists, and what every background process in `ps` output is doing.

## Table of Contents
1. [History & Philosophy](#history)
2. [Process Model](#process)
3. [Memory Architecture](#memory)
4. [On-Disk Layout (PGDATA)](#disk)
5. [Logical Hierarchy: Cluster → Database → Schema → Object](#hierarchy)
6. [The System Catalog](#catalog)
7. [Life of a Query Inside Postgres](#query)
8. [Life of a Write (INSERT/UPDATE) Inside Postgres](#write)
9. [Transaction IDs and Commit Log](#xid)
10. [Version Landmarks](#versions)
11. [Interview Questions](#qa)

---

## 1. History & Philosophy {#history}

- Descends from **Ingres** → **POSTGRES** (Michael Stonebraker, UC Berkeley, 1986) → Postgres95 (added SQL) → **PostgreSQL** (1996).
- Community-developed, BSD-like license, no single vendor owns it.
- Design values: correctness and standards compliance first, **extensibility** (custom types, operators, index methods, procedural languages, extensions), and a conservative approach to data safety.
- Yearly major release (~September/October): PG 16 (2023), 17 (2024), 18 (2025). Each major is supported for 5 years. Minor releases (bug/security fixes) come quarterly. Always run the latest minor.

---

## 2. Process Model {#process}

PostgreSQL uses **processes, not threads**:

```
postmaster (the main "postgres" process; listens on 5432)
 ├── backend process (one per client connection)  ← forked on connect
 ├── backend process ...
 ├── background writer         – trickles dirty buffers to disk
 ├── checkpointer              – performs checkpoints
 ├── walwriter                 – flushes WAL buffers periodically
 ├── autovacuum launcher       – spawns autovacuum workers
 │     └── autovacuum worker(s)
 ├── stats / cumulative statistics (in shared memory since PG 15)
 ├── archiver                  – runs archive_command for WAL segments
 ├── logical replication launcher → apply workers
 ├── walsender (per replica / logical slot consumer)
 ├── walreceiver (on standbys)
 ├── startup process (recovery/replay on standbys)
 ├── parallel workers (spawned per parallel query)
 ├── io workers (PG 18 async I/O, io_method=worker)
 └── background workers (extensions: pg_cron, TimescaleDB, etc.)
```

Implications:
- **Each connection = a process** with its own memory → connections are expensive → **use a pooler** (see fundamentals/07).
- A crashing backend (segfault in an extension) makes the postmaster restart all backends and run crash recovery to protect shared memory. Every connection drops.
- `pg_stat_activity` shows one row per backend, including `backend_type` (client backend, autovacuum worker, walsender…).
- Kill a query: `SELECT pg_cancel_backend(pid);` (graceful) / `pg_terminate_backend(pid)` (kills the connection). Never `kill -9` a backend: it forces a cluster-wide crash recovery.

---

## 3. Memory Architecture {#memory}

### Shared memory (all processes)
| Area | Setting | Purpose |
|---|---|---|
| **Shared buffers** | `shared_buffers` (default 128 MB; set ~25% of RAM) | Page cache for tables/indexes |
| **WAL buffers** | `wal_buffers` (auto: 1/32 of shared_buffers, max 16 MB) | WAL records before flush |
| Lock table | `max_locks_per_transaction` × connections | Heavyweight locks |
| CLOG/commit status buffers, subtransaction buffers, multixact | SLRU caches (configurable in PG 17) | Transaction status lookups |
| Proc array | | Running transactions (snapshots built from it) |

### Per-backend (local) memory
| Area | Setting | Notes |
|---|---|---|
| **work_mem** | default 4 MB | Per **sort/hash operation**, not per query! A query with 5 hash joins in 4 parallel workers can use 5 × 4 × work_mem (and hash ops get `hash_mem_multiplier` = 2×). Too low → spills to disk (`external merge`); too high × many connections → OOM |
| **maintenance_work_mem** | default 64 MB | VACUUM, CREATE INDEX, ALTER TABLE ADD FK. Raise to 1–2 GB for faster index builds |
| `autovacuum_work_mem` | -1 (uses maintenance_work_mem) | Per autovacuum worker |
| `temp_buffers` | 8 MB | Temp tables |
| Catalog/relcache/plan cache | | Grows with number of tables touched → schema-per-tenant with 10k tables bloats every backend |

### OS page cache
Postgres uses **buffered I/O** (not O_DIRECT, until PG 18's async I/O work began moving in that direction). Data is often cached twice: shared_buffers and the kernel page cache. `effective_cache_size` (~50–75% of RAM) is only a *planner hint* about this combined cache.

**Huge pages** (`huge_pages = try/on`) reduce TLB misses and page-table memory for large shared_buffers. Recommended on Linux for big instances.

---

## 4. On-Disk Layout (PGDATA) {#disk}

```
$PGDATA/
├── PG_VERSION
├── postgresql.conf, postgresql.auto.conf (ALTER SYSTEM writes here), pg_hba.conf, pg_ident.conf
├── base/                 ← one directory per database (named by OID)
│   └── 16384/
│       ├── 16385         ← a table's main fork (1 GB segments: 16385, 16385.1, ...)
│       ├── 16385_fsm     ← free space map
│       ├── 16385_vm      ← visibility map (all-visible / all-frozen bits per page)
│       └── 16385_init    ← init fork (unlogged tables)
├── global/               ← cluster-wide catalogs (pg_database, pg_authid)
├── pg_wal/               ← WAL segments (16 MB each by default)
├── pg_xact/              ← commit status of transactions (formerly pg_clog)
├── pg_multixact/         ← multi-transaction lock info (row locked by several txns)
├── pg_subtrans/          ← subtransaction parent mapping
├── pg_logical/, pg_replslot/  ← logical decoding & replication slot state
├── pg_tblspc/            ← symlinks to tablespaces
└── pg_stat/, pg_stat_tmp/ ...
```

- Find a table's file: `SELECT pg_relation_filepath('orders');`
- Sizes: `pg_relation_size` (main fork), `pg_table_size` (+TOAST, FSM, VM), `pg_indexes_size`, `pg_total_relation_size` (everything).
- **Forks**: main, FSM (where to find free space for inserts), VM (used by index-only scans and VACUUM to skip pages), init.
- **TOAST table** per table with large columns: `pg_toast.pg_toast_<oid>`.

---

## 5. Logical Hierarchy {#hierarchy}

```
Cluster (one postmaster, one PGDATA, one port)
 ├── Roles (cluster-wide users/groups)
 ├── Tablespaces (cluster-wide)
 └── Databases (isolated: you can't join across databases without dblink/postgres_fdw)
      └── Schemas (namespaces: public, app, audit, pg_catalog, information_schema)
           └── Objects: tables, views, materialized views, indexes, sequences, functions, types, ...
```
- `search_path` decides which schema an unqualified name resolves to (`"$user", public`). A security footgun: set it explicitly in `SECURITY DEFINER` functions. Since PG 15, `public` schema no longer grants CREATE to everyone.
- Use schemas to organize (per module, per tenant) and to scope permissions.
- `template1` is cloned for `CREATE DATABASE`; `template0` is pristine.

---

## 6. The System Catalog {#catalog}

Everything about the database is stored in tables you can query:
| Catalog | Content |
|---|---|
| `pg_class` | Every relation (table, index, sequence, view, matview, TOAST). `relkind`, `reltuples` (row estimate), `relpages` |
| `pg_attribute` | Columns |
| `pg_index` | Index metadata |
| `pg_namespace` | Schemas |
| `pg_constraint` | Constraints |
| `pg_proc` | Functions |
| `pg_type` | Types |
| `pg_stats` (view) | Planner statistics per column (readable version of `pg_statistic`) |
| `pg_roles`, `pg_authid` | Roles |
| `pg_settings` | All config parameters with source & context |
| `pg_locks` | Current locks |
| `pg_stat_activity` | Sessions |
| `pg_stat_user_tables` / `_indexes` | Scan counts, tuples ins/upd/del, dead tuples, last (auto)vacuum/analyze |
| `pg_statio_user_tables` | Buffer hits vs reads |
| `pg_stat_statements` (extension) | Query performance aggregates |
| `pg_stat_io` (PG 16+) | I/O by backend type and context |

`information_schema` is the SQL-standard portable view; `pg_catalog` is richer and faster. `psql` meta-commands (`\d`, `\dt+`, `\di+`, `\df`, `\dn`, `\du`, `\l`, `\x`, `\timing`, `\watch`, `\gexec`) query these for you. Add `-E` to psql to see the underlying SQL.

**Fast row estimate** for a huge table: `SELECT reltuples::bigint FROM pg_class WHERE oid = 'orders'::regclass;` (vs `COUNT(*)` which scans).

---

## 7. Life of a Query {#query}

1. Client sends query (simple or extended protocol) to its backend.
2. **Parser** → raw parse tree. **Analyzer** → query tree (names resolved via catalog, privileges checked).
3. **Rewriter** → expands views, applies rules and **RLS policies**.
4. **Planner** → generates paths (seq scan, index scans, joins, parallel plans), costs them using `pg_statistic`, picks the cheapest. GEQO kicks in at ≥ 12 FROM items.
5. **Executor** → pulls tuples through the plan tree (Volcano iterator model; JIT compilation via LLVM for expensive expression evaluation if `jit = on` and cost > `jit_above_cost` — often better disabled for OLTP because compile time exceeds savings).
6. Pages requested go through **shared buffers**; misses read from the OS (maybe cached there) or disk.
7. Visibility of each tuple checked against the **snapshot** (MVCC).
8. Rows stream back to the client.

---

## 8. Life of a Write {#write}

`UPDATE accounts SET balance = balance - 10 WHERE id = 7;`

1. Find the row via index → heap tuple (page P, slot S).
2. Lock the tuple (row lock in the tuple header: `xmax` + infomask bits — **no lock table entry for row locks**, which is why PG can lock millions of rows without memory blowup).
3. Create a **new tuple version** with new `xmin` = current XID; set old tuple's `xmax` = current XID. If space on the same page and no indexed column changed → **HOT update** (no index updates). Otherwise insert new index entries in **every** index.
4. Write WAL records describing the changes into WAL buffers (full-page image if first change to the page since the last checkpoint).
5. Mark buffer dirty (stays in shared buffers).
6. On `COMMIT`: write commit record to WAL, **flush WAL to disk (fsync)** up to that LSN (group commit), set transaction status to committed in `pg_xact`. Return success.
7. Later: background writer / checkpointer write the dirty page to the data file.
8. Later still: once no snapshot can see the old version, **VACUUM** reclaims it.

---

## 9. Transaction IDs and Commit Log {#xid}

- Every writing transaction gets a 32-bit **XID** (read-only transactions only get a *virtual* XID — cheap).
- `pg_xact` stores 2 bits per XID: in progress / committed / aborted / sub-committed.
- **Hint bits**: the first reader that checks a tuple's xmin status in `pg_xact` sets a bit on the tuple ("xmin committed") so later readers skip the lookup. This is why a **SELECT can dirty pages and generate writes** right after a bulk load.
- 32-bit XIDs **wrap around** after ~4 billion. Comparison is modulo 2³², so each XID sees 2 billion in the "past" and 2 billion in the "future". Old tuples must be **frozen** (marked as visible-to-all, independent of XID) by VACUUM before they'd appear to be "in the future". If not, PG stops accepting writes to prevent data loss (**XID wraparound shutdown**). Details in `02-mvcc-and-vacuum.md`.
- 64-bit XIDs have been a long-discussed project; FullTransactionId exists internally (epoch + xid) but tuples still store 32-bit XIDs.

---

## 10. Version Landmarks {#versions}

| Version | Notable for backend engineers |
|---|---|
| 9.0 | Streaming replication, hot standby |
| 9.1 | Serializable SSI, synchronous replication, unlogged tables, extensions (`CREATE EXTENSION`) |
| 9.2 | Index-only scans, JSON type |
| 9.4 | **JSONB**, logical decoding |
| 9.5 | `ON CONFLICT` upsert, RLS, `SKIP LOCKED`, BRIN |
| 9.6 | Parallel query |
| 10 | Declarative partitioning, logical replication (pub/sub), identity columns |
| 11 | Instant `ADD COLUMN … DEFAULT`, covering indexes (`INCLUDE`), procedures with transaction control, JIT |
| 12 | CTE inlining, generated columns, `REINDEX CONCURRENTLY`, pluggable table AM |
| 13 | B-tree deduplication, incremental sort, parallel vacuum of indexes |
| 14 | Connection scalability improvements, multirange types, `pipeline mode` in libpq |
| 15 | `MERGE`, `UNIQUE NULLS NOT DISTINCT`, public schema CREATE revoked, JSON logs |
| 16 | Logical replication from standbys, `pg_stat_io`, SQL/JSON constructors, parallel FULL/RIGHT joins |
| 17 | Incremental backup, `JSON_TABLE`, VACUUM memory improvements (radix tree TID store), `MERGE … RETURNING`, failover slots, `transaction_timeout` |
| 18 | **Asynchronous I/O** subsystem (`io_method`), `uuidv7()`, virtual generated columns (default), B-tree skip scan, OAuth authentication, `RETURNING OLD/NEW`, faster major upgrades preserving planner stats |

---

## 11. Interview Questions {#qa}

1. Why does Postgres need a connection pooler more than MySQL?
2. What's `work_mem` and why is setting it to 1 GB dangerous?
3. What processes run in a Postgres cluster and what does each do?
4. Walk through what happens on `UPDATE` until the data is on disk.
5. What is a HOT update?
6. What are hint bits and why can a SELECT generate disk writes?
7. What is transaction ID wraparound?
8. How would you get a quick row count for a 2-billion-row table?
