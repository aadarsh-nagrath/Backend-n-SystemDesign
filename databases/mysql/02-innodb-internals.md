# InnoDB Internals: Clustered Index, Buffer Pool, Redo, Undo, Purge

## Table of Contents
1. [Tablespaces, Segments, Extents, Pages](#space)
2. [Row Formats and Hidden Columns](#rows)
3. [The Clustered Index (Index-Organized Tables)](#clustered)
4. [Secondary Indexes](#secondary)
5. [Primary Key Choice: Why It Matters So Much in InnoDB](#pk)
6. [The Buffer Pool](#bufferpool)
7. [Change Buffer and Adaptive Hash Index](#cbahi)
8. [Redo Log](#redo)
9. [Undo Log, MVCC, and Read Views](#undo)
10. [Purge and History List Length](#purge)
11. [Doublewrite Buffer and Flushing](#flush)
12. [Crash Recovery](#recovery)
13. [Monitoring InnoDB](#monitor)
14. [Interview Questions](#qa)

---

## 1. Tablespaces, Segments, Extents, Pages {#space}

```
Tablespace (.ibd file, or system/general/undo/temp tablespace)
 └── Segments (each index has a leaf segment + a non-leaf segment)
      └── Extents (1 MB = 64 × 16 KB pages)
           └── Pages (16 KB default: index pages, undo pages, BLOB pages, ...)
                └── Rows (records)
```
- `innodb_file_per_table=ON` (default) gives each table its own `.ibd`, so `DROP/TRUNCATE` returns space to the OS and tables can be moved individually.
- **General tablespaces** (`CREATE TABLESPACE`) hold several tables.
- **System tablespace** (`ibdata1`) holds the change buffer, plus legacy undo and doublewrite if not separated. It never shrinks.
- **Undo tablespaces** (separate since 8.0; can be truncated automatically with `innodb_undo_log_truncate=ON`).
- Space isn't returned to the OS after deletes inside a `.ibd`. Rebuild with `OPTIMIZE TABLE` (= `ALTER TABLE … FORCE`, done online) to reclaim it.

---

## 2. Row Formats and Hidden Columns {#rows}

Row formats: `REDUNDANT` (ancient), `COMPACT`, **`DYNAMIC`** (default), `COMPRESSED`.
- DYNAMIC stores long variable-length columns (TEXT/BLOB/long VARCHAR/JSON) fully off-page when the row doesn't fit, with a 20-byte pointer kept in the row.
- Max row size is ~half a page (~8 KB) for in-page data. Index key prefix limit: 3072 bytes (DYNAMIC). With utf8mb4 (4 bytes/char), `VARCHAR(768)` is the max fully indexable column.

Every clustered index record has hidden system columns:
| Column | Size | Purpose |
|---|---|---|
| `DB_TRX_ID` | 6 bytes | ID of the last transaction that inserted or updated the row |
| `DB_ROLL_PTR` | 7 bytes | Pointer to the undo log record with the previous version |
| `DB_ROW_ID` | 6 bytes | Only if no PK or unique NOT NULL key exists (hidden clustered key, global counter, contention point) |

Plus a record header (5 bytes in COMPACT/DYNAMIC) with a delete-mark flag, the next-record pointer (records are a singly linked list in key order within the page), and so on.

---

## 3. The Clustered Index {#clustered}

**InnoDB tables are B+trees ordered by primary key; the leaf pages contain the full rows.**

```
                [ PK 1..1000 | 1001..2000 | ... ]          non-leaf pages: PK separators
                 /            |
     [rows PK 1..120] ⇄ [rows PK 121..240] ⇄ ...          leaf pages: complete rows, doubly linked
```
Consequences:
- PK lookups go straight to the data: one B+tree traversal (3–4 levels typically, internal levels cached).
- PK range scans (`WHERE id BETWEEN …`, `ORDER BY id`) are sequential and very fast.
- Rows are physically ordered by PK, so **inserting in PK order appends to the rightmost page** (efficient). Random PK order inserts into random pages.
- Updating the PK = delete + insert (row physically moves). Never use a mutable PK.

PK selection order (InnoDB): the declared PRIMARY KEY, else the first `UNIQUE NOT NULL` index, else a hidden 6-byte `DB_ROW_ID`. **Always declare an explicit PK.** Tables without one also break or slow down row-based replication (replicas do full scans to find rows) and Group Replication requires a PK (`sql_require_primary_key=ON` enforces it).

---

## 4. Secondary Indexes {#secondary}

A secondary index leaf entry = **(indexed columns, primary key columns)**.

Lookup `WHERE email = ?` on `idx_email`:
1. Traverse `idx_email` → find `(email, id=42)`.
2. Traverse the **clustered index** with `id=42` → full row. (The "bookmark lookup" or "back to table" cost: `回表` in Chinese MySQL literature.)

Implications:
- **Covering indexes** skip step 2. A secondary index on `(email)` already covers `SELECT id FROM users WHERE email=?`, because the PK is included for free. EXPLAIN shows `Using index`.
- The **PK width multiplies across all secondary indexes**. A `CHAR(36)` UUID PK (36+ bytes) makes every secondary index entry ~36 bytes heavier than a BIGINT PK (8 bytes).
- Secondary index entries aren't updated in place on change. They're **delete-marked** and a new entry is inserted, and purge later removes the delete-marked ones.
- Secondary indexes have no MVCC version info of their own (only a page-level max trx id). Reading a secondary index under MVCC sometimes requires checking the clustered record, which is why covering index reads can still visit the clustered index when the page has recent changes.

---

## 5. PK Choice {#pk}

| PK type | Insert pattern | Page fill | Secondary index overhead | Verdict |
|---|---|---|---|---|
| `BIGINT AUTO_INCREMENT` | Append to right edge | ~94% (15/16) | +8 B per entry | ✅ Default |
| UUIDv4 `CHAR(36)` | Random | ~50–70% (splits) | +36 B (+ collation compare cost) | ❌ Worst |
| UUIDv4 `BINARY(16)` | Random | ~50–70% | +16 B | ⚠️ Still random |
| UUIDv1 `BINARY(16)` with `UUID_TO_BIN(u, 1)` (time-swapped) | ~Sequential | Good | +16 B | ✅ OK |
| UUIDv7 / ULID `BINARY(16)` | ~Sequential | Good | +16 B | ✅ Good for distributed IDs |
| Natural composite `(tenant_id, id)` | Sequential per tenant | Good | Wider | ✅ Great for multi-tenant locality: a tenant's rows are physically clustered together |

Benchmarks commonly show random UUID PKs causing several-fold insert slowdowns once the table exceeds the buffer pool, because every insert touches a random leaf page that has to be read from disk.

**Auto-increment lock modes** (`innodb_autoinc_lock_mode`):
- `0` traditional: table-level AUTO-INC lock held until the end of the statement.
- `1` consecutive: lightweight for simple inserts; table lock for bulk inserts with unknown row counts.
- `2` **interleaved** (default in 8.0): no table lock, max concurrency. IDs from concurrent bulk inserts interleave (needs row-based binlog, which is the default).
- Auto-increment values are **not** reused after rollback, so gaps are normal. Before 8.0, the counter was recomputed as `MAX(id)+1` on restart (deleted top IDs could be reused). It's persisted since 8.0.

---

## 6. The Buffer Pool {#bufferpool}

- `innodb_buffer_pool_size`: **the most important setting**. 50–75% of RAM on a dedicated server (`innodb_dedicated_server=ON` auto-sizes it). Resizable online.
- `innodb_buffer_pool_instances`: splits the pool into instances to reduce mutex contention (auto: 8 if ≥ 1 GB).
- Contents: data and index pages, plus the change buffer, adaptive hash index, lock info, and so on.
- **LRU with midpoint insertion**: newly read pages enter the "old" sublist at 3/8 from the tail (`innodb_old_blocks_pct=37`). A page moves to the "young" (hot) sublist only if accessed again after `innodb_old_blocks_time` (1000 ms). This protects the hot set from one-off full table scans and mysqldump.
- **Read-ahead**: linear (sequential access in an extent prefetches the next extent; `innodb_read_ahead_threshold`) and random (off by default).
- **Free list, LRU list, flush list** (dirty pages ordered by oldest modification LSN).
- **Warmup**: `innodb_buffer_pool_dump_at_shutdown` / `load_at_startup` (on by default) saves the page IDs of the hottest 25%, so restarts don't begin cold.
- Hit ratio: `1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests` (should be > 99.9% for OLTP).

---

## 7. Change Buffer and Adaptive Hash Index {#cbahi}

**Change buffer**: when a change (insert, delete-mark, purge) targets a **non-unique secondary index** page that isn't in the buffer pool, InnoDB records the change in the change buffer instead of reading the page from disk. It's merged later when the page is read or by a background merge. This saves random reads for write-heavy tables with many secondary indexes on slow disks.
- It doesn't apply to unique indexes (uniqueness checks need the page anyway) or to the clustered index.
- Settings: `innodb_change_buffering` (`all` in 8.0, **`none` by default in 8.4** since SSDs make it less valuable and it complicates crash recovery and slow shutdown), `innodb_change_buffer_max_size` (25% of the pool).

**Adaptive Hash Index (AHI)**: InnoDB watches B-tree search patterns and builds in-memory hash indexes on hot index page prefixes, turning B-tree descents into O(1) lookups. It helps read-mostly workloads with repeated equality lookups. It **hurts** with high concurrency DML (latch contention: you'll see `btr_search_latch` / AHI partitions in contention) and with `DROP TABLE` of large tables (hash entries must be cleaned). `innodb_adaptive_hash_index` is **OFF by default in 8.4**; partitioned via `innodb_adaptive_hash_index_parts`.

---

## 8. Redo Log {#redo}

- Physical/logical records ("in page X of space Y at offset Z, write these bytes") with LSNs.
- Write path: mini-transactions (mtr) produce records → log buffer (`innodb_log_buffer_size`, 64 MB default in 8.4) → written and fsynced to redo files at commit (per `innodb_flush_log_at_trx_commit`) and periodically.
- MySQL 8.0 redesigned redo writing to be lock-free with dedicated log writer and flusher threads, which scales better with many concurrent commits.
- **Capacity**: `innodb_redo_log_capacity` (8.0.30+, dynamic; replaces `innodb_log_file_size × innodb_log_files_in_group`). The redo log is circular, and a **checkpoint** advances as dirty pages are flushed.
  - Too small → the checkpoint age approaches capacity → **"async/sync flushing"** stalls where user threads must wait for page flushes. You see throughput drop in sawtooth patterns.
  - Rule of thumb: hold ~1 hour of peak redo generation (measure the LSN delta per hour: `SHOW ENGINE INNODB STATUS` "Log sequence number", or `performance_schema.log_status`).
- Can be disabled temporarily for bulk loads: `ALTER INSTANCE DISABLE INNODB REDO_LOG;` (8.0.21+, **unsafe**: a crash loses the instance).

`innodb_flush_log_at_trx_commit`:
| Value | Behavior | Loss window |
|---|---|---|
| 1 (default) | write + fsync at each commit | none (with `sync_binlog=1`) |
| 2 | write to OS cache at commit; fsync ~1/s | ~1 s on OS crash/power loss; safe on mysqld crash |
| 0 | write + fsync ~1/s | ~1 s even on mysqld crash |

---

## 9. Undo Log, MVCC, Read Views {#undo}

- Every modification writes an **undo record** (insert undo: just the PK for rollback; update undo: the old values of changed columns).
- Rows form a **version chain**: current row in the clustered index → `DB_ROLL_PTR` → undo record with the previous version → its roll pointer → older…
- A **read view** (InnoDB's snapshot) records:
  - `m_low_limit_id`: next trx id to be assigned at view creation (ids ≥ this are invisible),
  - `m_up_limit_id`: smallest active trx id (ids < this are visible),
  - `m_ids`: list of active (uncommitted) trx ids at creation (invisible),
  - `m_creator_trx_id`: own changes are visible.
- Visibility: walk the version chain from newest to oldest until finding a version whose `DB_TRX_ID` is visible to the read view.
- **When read views are created**:
  - REPEATABLE READ: at the **first consistent read** in the transaction (or at `START TRANSACTION WITH CONSISTENT SNAPSHOT`) and reused for the whole transaction.
  - READ COMMITTED: a new read view for **each consistent read statement**.
- Locking reads (`FOR UPDATE`/`FOR SHARE`) and DML **don't use the read view**. They read the latest committed version and lock it ("current read" vs "snapshot read").

Cost of long transactions: a reader with an old read view has to walk long undo chains for hot rows, so its reads get slower and slower. Meanwhile undo can't be purged.

---

## 10. Purge and History List Length {#purge}

- **Purge threads** (`innodb_purge_threads`, default 4) remove undo records no read view can need, and physically delete delete-marked records from clustered and secondary indexes.
- **History List Length (HLL)**: the number of unpurged transactions' undo logs. Shown in `SHOW ENGINE INNODB STATUS` under TRANSACTIONS ("History list length 1234"), or `information_schema.innodb_metrics` `trx_rseg_history_len`.
- Normal: hundreds to low thousands. Growing into millions means a long-running transaction (or a replica-like long read) is holding back purge, or purge can't keep up with the DML rate.
- Effects of high HLL: undo tablespaces grow, reads slow down (longer version chains), and secondary index scans hit many delete-marked records.
- Find the culprit: `SELECT trx_id, trx_started, trx_mysql_thread_id, trx_query FROM information_schema.innodb_trx ORDER BY trx_started LIMIT 5;`
- `innodb_max_purge_lag` can throttle DML when HLL gets too high (off by default).

---

## 11. Doublewrite and Flushing {#flush}

**Doublewrite buffer**: before writing dirty pages to their real location, InnoDB writes them to the sequential doublewrite area and fsyncs, then writes them in place. If a crash tears a page mid-write (16 KB InnoDB page vs 4 KB atomic disk writes), recovery restores the intact copy from doublewrite, then applies redo. Since 8.0.20 it lives in separate `#ib_*.dblwr` files. `innodb_doublewrite=OFF` is safe only on storage with atomic 16 KB writes (ZFS, some cloud block stores claim it; Aurora doesn't use it at all because its storage layer differs). 8.0.30 adds `DETECT_ONLY`.

**Flushing**:
- **Page cleaner threads** (`innodb_page_cleaners`) flush dirty pages from the flush list (oldest LSN first, to advance the checkpoint) and from LRU tails (to keep free pages available).
- `innodb_io_capacity` (background flush rate in IOPS; set to what the storage sustains: 2,000–20,000+ for SSDs; 8.4 raised the default to 10,000) and `innodb_io_capacity_max`.
- **Adaptive flushing** increases the flush rate as redo fills up (`innodb_adaptive_flushing_lwm`, `innodb_max_dirty_pages_pct`/`_lwm`).
- `innodb_flush_method=O_DIRECT` (Linux; default in 8.4 where supported) bypasses the OS page cache and avoids double buffering. `O_DIRECT_NO_FSYNC` is possible on some filesystems.
- `innodb_flush_neighbors=0` on SSDs (default 0 in 8.0). Flushing contiguous neighbor pages only helps HDDs.

---

## 12. Crash Recovery {#recovery}

1. Find the last checkpoint LSN.
2. Restore any torn pages from the doublewrite buffer.
3. **Redo**: apply redo records from the checkpoint forward (parallelized, using a hash of pages).
4. Determine transaction states: active transactions get rolled back using undo (in the background after startup); prepared XA transactions get resolved against the **binlog** (commit if the transaction is in the binlog, else roll back).
5. Purge continues.

Recovery time is roughly proportional to the redo between the checkpoint and the crash, so a larger redo capacity means longer worst-case recovery (modern versions handle tens of GB in minutes).

`innodb_force_recovery` (1–6) is a **last-resort** setting to start a corrupted instance just enough to dump data. Values ≥ 4 can permanently corrupt data. Never run with it normally.

---

## 13. Monitoring InnoDB {#monitor}

```sql
SHOW ENGINE INNODB STATUS\G
-- Sections: SEMAPHORES (latch waits), LATEST DETECTED DEADLOCK, TRANSACTIONS (HLL, active trx),
-- FILE I/O, INSERT BUFFER AND ADAPTIVE HASH INDEX, LOG (LSN, flushed up to, last checkpoint),
-- BUFFER POOL AND MEMORY (hit rate, young/not young), ROW OPERATIONS

SHOW GLOBAL STATUS LIKE 'Innodb_%';
-- Innodb_buffer_pool_reads vs read_requests (hit ratio), Innodb_buffer_pool_pages_dirty,
-- Innodb_row_lock_waits / Innodb_row_lock_time_avg, Innodb_log_waits (log buffer too small),
-- Innodb_os_log_written (redo bytes), Innodb_data_fsyncs

SELECT name, count FROM information_schema.innodb_metrics WHERE status = 'enabled';

-- Checkpoint age
SELECT * FROM performance_schema.log_status\G

-- Locks
SELECT * FROM performance_schema.data_locks;
SELECT * FROM sys.innodb_lock_waits\G

-- Buffer pool contents by table
SELECT object_schema, object_name, allocated, data, pages FROM sys.innodb_buffer_stats_by_table ORDER BY allocated DESC LIMIT 20;  -- expensive on big pools
```

---

## 14. Interview Questions {#qa}

1. What is a clustered index? How does a secondary index lookup work in InnoDB?
2. Why are random UUID primary keys bad in InnoDB specifically? What would you use instead?
3. Explain the buffer pool's midpoint insertion strategy. What problem does it solve?
4. What are the redo log and the undo log each used for?
5. How does InnoDB decide which version of a row a transaction sees?
6. What is the history list length and what makes it grow?
7. What does the doublewrite buffer protect against?
8. What happens when the redo log is too small?
9. Why did MySQL 8.4 disable the change buffer and adaptive hash index by default?
