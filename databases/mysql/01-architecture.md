# MySQL Architecture: Server Layers, Storage Engines, and Versions

## Table of Contents
1. [History, Forks, and Versions](#history)
2. [The Layered Architecture](#layers)
3. [Connection Handling and Threads](#threads)
4. [The SQL Layer: Parser, Optimizer, Executor](#sql)
5. [Pluggable Storage Engines](#engines)
6. [Data Dictionary and Files on Disk](#files)
7. [The Two Logs: Binlog and Redo Log (and Why Both Exist)](#logs)
8. [Life of a Query](#query)
9. [Life of a Write (and Internal 2PC)](#write)
10. [Important System Schemas](#schemas)
11. [Interview Questions](#qa)

---

## 1. History, Forks, Versions {#history}

- Created in 1995 by MySQL AB (Monty Widenius, David Axmark). Sun bought it in 2008, then Oracle (via Sun) in 2010.
- Licensing: GPL Community Edition plus a commercial Enterprise Edition (thread pool, audit, backup, TDE key management, firewall).
- **Forks**:
  - **MariaDB** (Monty, 2009): diverged substantially. It has its own optimizer features, Aria engine, Galera built in, sequences, `RETURNING`, system-versioned tables, and different JSON (alias of LONGTEXT) and GTID implementations. *Not* a drop-in replacement for MySQL 8 anymore.
  - **Percona Server for MySQL**: drop-in compatible with enterprise-like features (thread pool, audit, more instrumentation, MyRocks), plus Percona XtraDB Cluster (Galera).
- **Versions**:
  - 5.7: EOL October 2023.
  - **8.0**: huge release (2018): transactional data dictionary, window functions, CTEs, roles, JSON improvements, `caching_sha2_password` default, instant ADD COLUMN, invisible/descending indexes, histograms, hash join (8.0.18), `EXPLAIN ANALYZE`, CHECK constraints (8.0.16). Reached EOL in **April 2026**.
  - **8.4 LTS** (April 2024): the current long-term-support line. It removed deprecated `MASTER/SLAVE` syntax, disabled `mysql_native_password` by default, and changed several InnoDB defaults (change buffering off, adaptive hash index off, higher `innodb_io_capacity`, and others).
  - **9.x "Innovation" releases**: quarterly feature releases (e.g., the `VECTOR` type in 9.0, JavaScript stored programs in Enterprise). Each is supported only until the next. Production should use LTS.
- Big MySQL users: Meta (MyRocks), Uber, Shopify, GitHub, Booking.com, YouTube (Vitess), Slack (Vitess), Airbnb, Twitter (historically), Wikipedia (MariaDB).

---

## 2. Layered Architecture {#layers}

```
┌──────────────────────────────────────────────────────────────┐
│ Clients (JDBC, mysql2, PyMySQL, Go driver, mysql CLI)        │
├──────────────────────────────────────────────────────────────┤
│ CONNECTION LAYER: listener, auth (caching_sha2_password),     │
│ TLS, thread-per-connection (or thread pool), session state   │
├──────────────────────────────────────────────────────────────┤
│ SQL LAYER ("server")                                         │
│  Parser → Resolver/Preparer → Optimizer (cost-based) →       │
│  Executor (iterator model since 8.0) ; binlog ; query rewrite│
│  Data dictionary (stored in InnoDB since 8.0)                │
├───────────────────── Handler API ────────────────────────────┤
│ STORAGE ENGINES: InnoDB (default) | MyRocks | MEMORY |       │
│                  MyISAM (legacy) | CSV | ARCHIVE | NDB       │
├──────────────────────────────────────────────────────────────┤
│ Files: .ibd tablespaces, redo log, undo tablespaces, binlogs │
└──────────────────────────────────────────────────────────────┘
```
The split between the **SQL layer** and the **storage engine** is MySQL's defining trait. Some consequences:
- The optimizer asks the engine for statistics (row estimates, index dives) through the handler API.
- Filtering can happen in the engine (**Index Condition Pushdown**) or in the server layer after rows come back.
- Replication (binlog) belongs to the server layer and is independent of the engine. Crash recovery (redo) belongs to InnoDB. Two logs, kept consistent by internal two-phase commit.

---

## 3. Connections and Threads {#threads}

- Default: **one OS thread per connection** (`thread_handling=one-thread-per-connection`). The thread cache (`thread_cache_size`) reuses threads.
- Thread pool (Enterprise, Percona, MariaDB) multiplexes many connections over a few worker threads. It helps when there are thousands of connections.
- `max_connections` (default 151). Each connection has session buffers: `sort_buffer_size`, `join_buffer_size`, `read_buffer_size`, `read_rnd_buffer_size`, `tmp_table_size`. Raising these globally multiplies with connections. Raise them per session instead.
- InnoDB limits concurrently executing threads only if `innodb_thread_concurrency` > 0 (default 0 = unlimited).
- `SHOW PROCESSLIST` / `performance_schema.processlist` / `sys.session` show sessions. `KILL QUERY <id>` stops the statement; `KILL <id>` closes the connection.

---

## 4. The SQL Layer {#sql}

- **Parser** builds a parse tree (Bison grammar).
- **Resolver/preparer** resolves names and transforms the query (subquery → semi-join, derived table merging, view merging).
- **Optimizer** (cost-based):
  - Chooses join order with a greedy/exhaustive search bounded by `optimizer_search_depth`.
  - Picks the access method per table: `const`, `eq_ref`, `ref`, `range`, `index`, `ALL`, `index_merge`, skip scan, and so on.
  - Gets row estimates from **index dives** (it actually probes the index for range boundaries; for long IN lists above `eq_range_index_dive_limit` = 200 it falls back to cardinality statistics) or from **persistent statistics** plus histograms.
  - Has optimizations like ICP (index condition pushdown), MRR (multi-range read), BKA (batched key access), semi-join strategies (FirstMatch, LooseScan, Materialize, DuplicateWeedout), derived condition pushdown (8.0.22), and hash join (8.0.18).
  - `optimizer_switch` toggles individual strategies. Optimizer hints: `/*+ JOIN_ORDER(a,b) INDEX(t idx) NO_MERGE() SET_VAR(sort_buffer_size=16M) MAX_EXECUTION_TIME(1000) */`.
- **Executor**: since 8.0.18 uses an iterator-based design (like PG's Volcano model). `EXPLAIN FORMAT=TREE` and `EXPLAIN ANALYZE` show this tree.
- **No query cache** since 8.0 (it was a global-mutex scalability killer, invalidated on any write to a table). Cache in the application or Redis, or use ProxySQL's query cache.

---

## 5. Storage Engines {#engines}

| Engine | Transactions | Locking | Crash-safe | Notes |
|---|---|---|---|---|
| **InnoDB** | Yes (ACID, MVCC) | Row-level | Yes | Default since 5.5; the only sane choice for OLTP |
| MyISAM | No | **Table-level** | **No** (needs REPAIR after crash) | Legacy. System tables moved off it in 8.0 |
| MEMORY | No | Table | Data lost on restart | Hash/B-tree indexes; temp use |
| **MyRocks** (Percona/MariaDB/Meta) | Yes | Row | Yes | RocksDB LSM: better compression and write efficiency; Meta's UDB |
| NDB (MySQL Cluster) | Yes | Row | Yes | Shared-nothing in-memory distributed engine; telecom use |
| ARCHIVE / CSV / BLACKHOLE / FEDERATED | — | — | — | Niche (BLACKHOLE was used for binlog relays) |

`SHOW ENGINES;` and `SELECT table_name, engine FROM information_schema.tables WHERE table_schema = 'app';` show what's in use. Convert legacy tables with `ALTER TABLE t ENGINE=InnoDB;`.

---

## 6. Data Dictionary and Files {#files}

MySQL 8.0 moved metadata into a **transactional data dictionary** stored in InnoDB (`mysql.ibd`). `.frm`, `.par`, and `.TRG` files are gone, which made DDL **atomic** (crash-safe: a crashed `DROP TABLE` doesn't leave orphan files). DDL is still **not transactional** in the PG sense: each DDL statement implicitly commits, and you can't roll back a group of DDLs.

`datadir` layout (8.0+):
```
/var/lib/mysql/
├── mysql.ibd                 data dictionary + system tables
├── ibdata1                   system tablespace (change buffer; legacy undo/doublewrite location)
├── undo_001, undo_002        undo tablespaces
├── #innodb_redo/             redo log files (8.0.30+; formerly ib_logfile0/1)
├── #innodb_temp/             session temporary tablespaces
├── ibtmp1                    global temp tablespace
├── #ib_16384_0.dblwr ...     doublewrite files (8.0.20+)
├── binlog.000123, binlog.index   binary logs
├── relay logs (on replicas)
├── auto.cnf                  server_uuid (must be unique per server!)
├── app/                      one directory per database (schema)
│   ├── orders.ibd            file-per-table tablespace (innodb_file_per_table=ON default)
│   └── customers.ibd
└── performance_schema/, sys/ ...
```

In MySQL, **database = schema** (they're synonyms). There's no extra namespace level like PG's database→schema.

`lower_case_table_names` (0 on Linux = case-sensitive table names, 1 on Windows, 2 on macOS) can **only be set at initialization** in 8.0. Mismatches between dev (macOS) and prod (Linux) cause "table doesn't exist" bugs.

---

## 7. The Two Logs {#logs}

| | Binary log (binlog) | InnoDB redo log |
|---|---|---|
| Layer | Server (SQL) | Storage engine |
| Content | Logical changes (row images or statements) | Physical page changes |
| Purpose | **Replication, PITR, CDC** (Debezium, Maxwell, Canal) | **Crash recovery** |
| Lifetime | Kept until `binlog_expire_logs_seconds` (default 30 days) | Circular; reused after checkpoint |
| Format setting | `binlog_format` = ROW (default) / STATEMENT / MIXED | — |

Why two? Historically, the binlog came first (engine-independent replication), and InnoDB brought its own redo log. Keeping them consistent requires **internal XA two-phase commit** (next section). Setting `sync_binlog=1` and `innodb_flush_log_at_trx_commit=1` gives full durability (the "double 1" setting).

Also: **undo logs** (InnoDB MVCC and rollback), the **relay log** (replica's local copy of the received binlog), the **general log** (every statement; never leave it on in production), the **slow query log**, and the **error log**.

---

## 8. Life of a Query {#query}

`SELECT name FROM users WHERE email = 'a@b.com';`
1. The connection thread receives the packet (`COM_QUERY`) or `COM_STMT_EXECUTE` for a server-side prepared statement.
2. Parse → resolve → optimize. The optimizer sees a unique index on email → `const` access.
3. Executor calls the InnoDB handler `index_read` on the `email` secondary index.
4. InnoDB looks up the page in the **buffer pool** (hash of space_id + page_no). On a miss it reads 16 KB from the `.ibd` file.
5. The secondary index leaf yields `(email → id)`. Since `name` isn't in the secondary index, InnoDB looks up the **clustered index** by `id` to get the full row (unless the index is covering).
6. MVCC: if the row's `DB_TRX_ID` isn't visible in the transaction's **read view**, InnoDB reconstructs the older version from the **undo log** via `DB_ROLL_PTR`.
7. The row returns to the server layer, which sends it to the client.

---

## 9. Life of a Write and Internal 2PC {#write}

`UPDATE accounts SET balance = balance - 10 WHERE id = 7;` with autocommit:
1. Locate the row in the clustered index, take an **exclusive record lock** (next-key lock semantics vary; see locking note).
2. Write an **undo record** (old value) into the undo log, which is itself protected by redo.
3. Modify the row **in place** in the buffer pool page and set `DB_TRX_ID` = this transaction and `DB_ROLL_PTR` → undo record. Mark the page dirty.
4. Generate **redo log records** into the log buffer.
5. Update secondary indexes if indexed columns changed (delete-mark old entry + insert new one).
6. Write the row event to the session's **binlog cache**.
7. COMMIT, via internal two-phase commit between the binlog and InnoDB:
   - **Prepare**: InnoDB writes a prepare record to redo and flushes it (with `innodb_flush_log_at_trx_commit=1`).
   - **Binlog write + fsync** (`sync_binlog=1`), group-committed in stages: flush, sync, commit.
   - **Commit**: InnoDB marks the transaction committed in redo (this doesn't need its own fsync, because recovery can use the binlog as the source of truth).
   - Crash recovery: a transaction that's prepared in InnoDB **and** present in the binlog gets committed; prepared but not in the binlog gets rolled back. That keeps replicas (which follow the binlog) and the primary consistent.
8. Later: page cleaner threads flush dirty pages (through the **doublewrite buffer**), and the purge thread removes undo records no read view needs.

---

## 10. System Schemas {#schemas}

| Schema | Purpose |
|---|---|
| `mysql` | Users, privileges, time zones, replication metadata, data dictionary (hidden) |
| `information_schema` | Standard metadata views (TABLES, COLUMNS, STATISTICS, KEY_COLUMN_USAGE, INNODB_TRX, …) |
| `performance_schema` | Low-level instrumentation: statement digests, waits, locks (`data_locks`, `data_lock_waits`), memory, I/O, replication status |
| `sys` | Friendly views over performance_schema: `sys.statement_analysis`, `sys.schema_unused_indexes`, `sys.schema_redundant_indexes`, `sys.innodb_lock_waits`, `sys.host_summary`, `sys.io_global_by_file_by_bytes` |

---

## 11. Interview Questions {#qa}

1. Describe MySQL's architecture. What does the storage engine API separate?
2. Why does MySQL have both a binlog and a redo log? How are they kept consistent?
3. Why was the query cache removed in 8.0?
4. InnoDB vs MyISAM: why is MyISAM unsuitable for OLTP?
5. What changed in MySQL 8.0's data dictionary, and what does "atomic DDL" mean (vs transactional DDL)?
6. Walk through what happens during an UPDATE until commit returns.
7. MySQL vs MariaDB vs Percona Server: what's the difference?
