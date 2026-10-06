# Storage Engine Internals: How Databases Actually Store and Find Bytes

> If you understand pages, B+trees, the buffer pool, the WAL, and LSM trees, almost every performance behavior of a database becomes predictable: why random UUIDs hurt, why `SELECT COUNT(*)` is slow, why writes amplify, why `fsync` matters, why Cassandra writes fast and reads slower.

## Table of Contents
1. [The Memory/Storage Hierarchy (numbers to know)](#hierarchy)
2. [Pages: The Unit of Everything](#pages)
3. [Heap Files vs Clustered (Index-Organized) Tables](#heap)
4. [B-Trees and B+Trees in Depth](#btree)
5. [The Buffer Pool](#buffer)
6. [Write-Ahead Logging (WAL) and Crash Recovery (ARIES)](#wal)
7. [Checkpoints, fsync, Torn Pages, Double-Write](#checkpoint)
8. [LSM Trees: The Write-Optimized Alternative](#lsm)
9. [B-Tree vs LSM: Amplification Trade-offs](#compare)
10. [Row Stores vs Column Stores](#columnar)
11. [Other Index Structures (Hash, Bitmap, Inverted, Spatial, Vector)](#other)
12. [Compression, TOAST and Large Values](#toast)
13. [Interview Questions](#qa)

---

## 1. Hierarchy & Latency Numbers {#hierarchy}

| Operation | Approx latency |
|---|---|
| L1 cache reference | ~1 ns |
| Main memory reference | ~100 ns |
| Read 4 KB randomly from NVMe SSD | ~20–100 µs |
| Read 1 MB sequentially from memory | ~3–10 µs |
| Read 1 MB sequentially from NVMe | ~100–200 µs |
| Network round trip in same datacenter | ~0.5 ms |
| HDD seek | ~5–10 ms |
| Cross-continent round trip | ~150 ms |

Consequences:
- RAM is ~1000× faster than SSD random reads → **the working set fitting in memory is the single biggest performance factor**. Buffer pool hit ratio ≥ 99% is normal for healthy OLTP.
- Sequential I/O is much cheaper than random → logs are append-only; LSM trees turn random writes into sequential ones.
- Disks transfer in blocks; DBs read whole **pages**, never single rows.
- `fsync` (forcing data to durable media) costs from ~50 µs (NVMe with power-loss protection) to milliseconds (cloud network disks like EBS). This bounds commit latency → group commit.

---

## 2. Pages {#pages}

A **page** (block) is the fixed-size unit of I/O and caching:
- PostgreSQL: **8 KB** (compile-time).
- MySQL InnoDB: **16 KB** default (`innodb_page_size`, 4–64 KB).
- SQL Server: 8 KB; Oracle: configurable, usually 8 KB.

Slotted page layout (PG heap page, simplified):
```
+----------------------------------------------------------+
| Page header (LSN, checksum, flags, lower/upper pointers)  |
| Item pointers (line pointers) → [1][2][3][4] ...  grows → |
|                                                           |
|                 free space                                |
|                                                           |
|       ← grows   ... tuple 4 | tuple 3 | tuple 2 | tuple 1 |
| Special space (used by index pages)                       |
+----------------------------------------------------------+
```
- Line pointers give each tuple a stable address `(page, slot)` = PG's **ctid** / TID — indexes point to this.
- Tuples can move within the page (compaction) without changing their TID.
- Every page header stores the **LSN** of the last WAL record that modified it — crucial for recovery.
- **Fill factor**: leave free space in pages for future updates (PG `fillfactor` on tables enables HOT updates; InnoDB leaves 1/16 free in index pages).

Row width matters: a 2 KB row means 4 rows per 8 KB page; a 100-byte row → ~70 rows. Narrow rows = more rows per page = fewer I/Os = more data cached.

---

## 3. Heap vs Clustered Tables {#heap}

### Heap-organized (PostgreSQL, Oracle default, SQL Server heaps)
- Rows stored in no particular order in the table file.
- **All indexes are secondary**: index entry = (key → TID). Primary key index is just another index with a uniqueness rule.
- Lookup via index = index traversal + **heap fetch** (random I/O).
- Updates create new tuple versions with new TIDs → indexes must be updated too (unless HOT — see PG notes).

### Index-organized / clustered (MySQL InnoDB, SQL Server clustered index, SQLite `WITHOUT ROWID`, Oracle IOT)
- **The table *is* a B+tree keyed by the primary key**; leaf pages contain the full rows.
- Secondary indexes store (secondary key → **primary key value**), not a physical address.
- Secondary lookup = traverse secondary index → get PK → traverse clustered index again ("double lookup" / bookmark lookup).
- Range scans on PK are very fast (rows physically ordered).
- Wide PKs bloat **every** secondary index. Random PKs (UUIDv4) scatter inserts across the whole table → page splits, ~50% page fill, buffer pool churn.
- No PK defined? InnoDB uses the first NOT NULL UNIQUE index, else a hidden 6-byte `DB_ROW_ID`.

| | Heap (PG) | Clustered (InnoDB) |
|---|---|---|
| PK range scan | Random heap access (unless CLUSTERed recently / correlated) | Sequential |
| Secondary index lookup | 1 index traversal + 1 heap fetch | 2 B+tree traversals |
| Row moves on update | Yes (new version) | No (in-place + undo) |
| Secondary index size | key + 6-byte TID | key + full PK |
| Insert pattern | Append anywhere with free space (FSM) | Must go to PK position |

---

## 4. B-Trees and B+Trees {#btree}

Almost every relational index is a **B+tree** (Bayer & McCreight, 1970; "B-tree" in docs usually means B+tree).

Properties:
- Balanced: all leaves at the same depth.
- High **fan-out**: each internal node is a page holding hundreds of keys/child pointers (8 KB page with 16-byte entries ≈ 400+ children).
- **Internal nodes** hold only separator keys + child pointers; **leaves** hold the keys + values (row pointers or rows), and are linked in a **doubly linked list** for range scans.
- Height is tiny: fan-out 500 → height 3 indexes 500³ = 125 million keys; height 4 → 62 billion. The root and internal levels are almost always cached, so a lookup costs ~1 leaf I/O.

```
                    [ 40 | 80 ]                        ← root
           /             |              \
   [10|20|30]        [50|60|70]        [90|100]        ← internal
   /  |  |  \         ...                 ...
 leaves: [1..9] ⇄ [10..19] ⇄ [20..29] ⇄ ... ⇄ [100..]   ← linked leaves
```

Operations:
- **Search**: binary search inside each page, descend → O(log_F N) page reads.
- **Range scan**: find start leaf, follow sibling pointers. This is why indexes support `<, >, BETWEEN, ORDER BY, LIKE 'prefix%'`.
- **Insert**: into the leaf; if full → **page split** (half the entries move to a new page, separator pushed to parent; splits may cascade to root → tree grows by one level at the top).
  - Monotonic keys: engines optimize "rightmost split" (90/10 or 100/0 split) → pages stay full.
  - Random keys: 50/50 splits everywhere → pages ~69% full on average, many more pages, more I/O.
- **Delete**: remove entry; pages may merge when under-filled (PG only reclaims fully empty pages; InnoDB merges at `MERGE_THRESHOLD` 50%).

Concurrency: B-trees use **latch crabbing** (lock child, release parent if safe) or **B-link trees** (Lehman & Yao — PG's nbtree) with right-links so readers can tolerate concurrent splits.

Why not binary trees / hash tables? Binary trees have fan-out 2 → height 27 for 100M keys = 27 random I/Os. Hash tables can't do range or ordered scans and resize expensively.

### Composite index ordering
An index on `(a, b, c)` is sorted by a, then b within a, then c within b. Like a phone book sorted by (last name, first name). Usable for:
- `a = ?` ✅, `a = ? AND b = ?` ✅, `a = ? AND b > ?` ✅, `ORDER BY a, b` ✅
- `b = ?` alone ❌ (except skip-scan: MySQL 8.0.13+ "skip scan", PG 18 B-tree skip scan, Oracle)
- `a > ? AND b = ?` → b can't narrow the range (only a's range is used for seeking; b filtered within).

Rule: **equality columns first, then range/sort columns.**

Deep dive on index design: [`scaling-db/db-indexing.md`](../../scaling-db/db-indexing.md) and `05-query-planning-and-optimization.md`.

---

## 5. The Buffer Pool {#buffer}

The DB caches pages in RAM in its own buffer pool rather than trusting the OS:
- PG: `shared_buffers` (typ. 25% of RAM) **plus** the OS page cache (PG uses buffered I/O → effectively double caching; `effective_cache_size` tells the planner how much total cache exists).
- InnoDB: `innodb_buffer_pool_size` (typ. 50–75% of RAM), uses `O_DIRECT` to bypass the OS cache.

Mechanics:
- Page table maps (file, block) → buffer frame.
- **Pin count** while a page is in use; **dirty flag** when modified (not yet written).
- Eviction policy:
  - PG: **clock-sweep** (approximate LRU with usage counts 0–5).
  - InnoDB: **midpoint-insertion LRU** — new pages enter at 3/8 from the tail ("old" sublist) and only move to the "young" sublist if accessed again after `innodb_old_blocks_time` (1 s). This protects the hot set from being flushed by a one-time full table scan.
  - PG uses small **ring buffers** for large sequential scans/VACUUM for the same reason.
- Dirty pages are written back by background writers / page cleaners and at checkpoints — **not at commit**. Commit only needs the WAL to be durable.

Why the DB manages its own cache: it knows access patterns (scans vs point lookups), must control write ordering for WAL correctness (a dirty page must not hit disk before its WAL record — the **WAL rule**), and can prefetch intelligently.

---

## 6. Write-Ahead Logging and Recovery {#wal}

**WAL rule**: before a modified page is written to the data file, the log record describing the change must be durably on disk. And at commit, all of the transaction's log records (up to the commit record) must be durable.

Why: writing a few sequential log bytes is far cheaper than random-writing every dirty page at commit. Data pages are written lazily.

| | PostgreSQL | InnoDB |
|---|---|---|
| Log name | WAL (`pg_wal/`, 16 MB segments) | **Redo log** (`#innodb_redo/` files in 8.0.30+, formerly `ib_logfile0/1`) |
| Also used for | Replication (physical streaming), PITR, logical decoding | Crash recovery only; replication uses the separate **binlog** |
| Undo | Not needed (old versions are in the heap) | **Undo log** (undo tablespaces) for rollback & MVCC |
| Commit coordination | Single log | **Two logs** (redo + binlog) kept consistent via internal 2PC (XA) |

Log Sequence Number (**LSN**): monotonically increasing byte position in the log. Every page records the LSN of its last change; recovery redoes only records with LSN > page LSN.

### ARIES (the classic recovery algorithm, Mohan et al. 1992)
1. **Analysis**: scan from last checkpoint, figure out which transactions were active and which pages were dirty.
2. **Redo**: replay history — reapply all logged changes (even for transactions that will be rolled back) to bring pages to their crash-time state. "Repeating history."
3. **Undo**: roll back transactions that never committed (InnoDB uses undo logs; logs compensation records so undo is itself idempotent).

Policies: **STEAL** (dirty pages of uncommitted txns may be flushed — needs undo) and **NO-FORCE** (commit doesn't force pages to disk — needs redo). This combo gives the best performance and is what ARIES/InnoDB use. PG's MVCC means it doesn't need undo for data pages: uncommitted tuples are simply invisible (commit status in `pg_xact`).

### Group commit
Each commit needs an fsync. With 5,000 commits/s and 1 ms fsync you'd be stuck. **Group commit** batches many transactions' commit records into one fsync. Both engines do it (PG `commit_delay`/`commit_siblings` tune it; InnoDB+binlog has a three-stage group commit pipeline: flush → sync → commit).

### Durability knobs (know what you trade)
| Setting | Meaning | Risk |
|---|---|---|
| PG `synchronous_commit = off` | Commit returns before WAL flush | Lose last ~3×`wal_writer_delay` (≈600 ms) of commits on crash; **no corruption** |
| PG `fsync = off` | Never fsync | **Corruption** on OS crash. Only for throwaway data |
| InnoDB `innodb_flush_log_at_trx_commit = 1` | fsync redo at every commit (default) | Safe |
| `= 2` | Write to OS at commit, fsync every second | Lose ≤1 s on OS crash/power loss (mysqld crash alone is safe) |
| `= 0` | Write + fsync every second | Lose ≤1 s even on mysqld crash |
| MySQL `sync_binlog = 1` | fsync binlog at commit (default) | Needed for replica/PITR consistency |

---

## 7. Checkpoints, fsync, Torn Pages {#checkpoint}

**Checkpoint**: flush all dirty pages up to some LSN, then record "recovery can start here". Bounds recovery time and lets old WAL be recycled.
- PG: `checkpoint_timeout` (5 min default) and `max_wal_size` (1 GB default) trigger checkpoints; `checkpoint_completion_target = 0.9` spreads writes. Too-frequent checkpoints → I/O spikes and **full-page-write** amplification.
- InnoDB: **fuzzy checkpointing** continuously via page cleaners; if redo log fills up (`innodb_redo_log_capacity`) it does aggressive "sync flushing" → throughput stalls. Size redo to hold ~1 hour of writes as a rule of thumb.

**Torn pages**: an 8/16 KB page write isn't atomic at the hardware level (disk sectors are 512 B/4 KB). A crash mid-write leaves half old/half new — WAL redo can't fix it since redo records assume a consistent base page.
- PG: **full_page_writes** — the first modification of a page after each checkpoint logs the *entire page image* into WAL. This is why WAL volume spikes right after checkpoints, and why fewer checkpoints = less WAL.
- InnoDB: **doublewrite buffer** — pages are first written to a sequential doublewrite area, fsynced, then to their real location. On recovery, a torn page is restored from the doublewrite copy. Can be disabled on filesystems with atomic writes (ZFS, some FusionIO/cloud setups).

**fsync semantics gotcha** ("fsyncgate", 2018): on Linux, if fsync fails, the kernel may mark dirty pages clean and drop the error; a retried fsync "succeeds" without the data. PG now PANICs on fsync failure (and crash recovery replays WAL) instead of retrying.

---

## 8. LSM Trees {#lsm}

**Log-Structured Merge-tree** (O'Neil et al., 1996) — used by RocksDB, LevelDB, Cassandra, ScyllaDB, HBase, Bigtable, InfluxDB, CockroachDB (Pebble), TiKV, MyRocks, YugabyteDB, ClickHouse's MergeTree is a cousin.

Write path:
1. Append to a **commit log / WAL** (durability).
2. Insert into the **memtable** (in-memory sorted structure: skip list or balanced tree).
3. When the memtable fills (e.g., 64 MB), freeze it and flush it to disk as an immutable, sorted file: an **SSTable** (Sorted String Table).
4. Background **compaction** merges SSTables, discarding overwritten values and deleted keys.

```
writes → [WAL] + [memtable] --flush--> L0: [sst][sst][sst]   (overlapping)
                                          ↓ compaction
                                   L1: [sst|sst|sst|sst]       (non-overlapping, 10× bigger)
                                          ↓
                                   L2: [....................]  (10× bigger)
```

Read path:
1. Check memtable, then immutable memtables.
2. Check SSTables newest → oldest; stop at first hit.
3. **Bloom filters** per SSTable answer "definitely not here" to skip files (false-positive rate ~1% at 10 bits/key).
4. **Sparse index / block index** inside each SSTable finds the block; block cache caches hot blocks.

Deletes write a **tombstone** (a marker) — the data isn't gone until compaction drops both the old value and the tombstone after it reaches the bottom level (Cassandra: after `gc_grace_seconds`, default 10 days, to avoid resurrecting data on replicas that missed the delete). Tombstone-heavy workloads (queues on Cassandra!) make reads scan many tombstones → slow → `TombstoneOverwhelmingException`.

### Compaction strategies
| Strategy | How | Good for | Cost |
|---|---|---|---|
| **Size-tiered (STCS)** | Merge SSTables of similar size | Write-heavy | High space amplification (needs up to 2× free disk), reads may check many files |
| **Leveled (LCS)** | Each level non-overlapping, 10× bigger than previous | Read-heavy, updates | High write amplification (rewrites data ~10× per level) |
| **Time-window (TWCS)** | Compact by time bucket; drop whole buckets on TTL expiry | Time-series with TTL | Bad if out-of-order writes/updates |
| Universal (RocksDB) / tiered+leveled hybrids | | Tunable | |

---

## 9. B-Tree vs LSM: Amplification {#compare}

Three amplifications (the RUM conjecture: you can optimize two of **R**ead, **U**pdate, **M**emory/space — not all three):
- **Write amplification**: bytes written to disk / bytes written by app.
- **Read amplification**: disk reads per logical read.
- **Space amplification**: disk used / logical data size.

| | B+tree (InnoDB, PG) | LSM (RocksDB, Cassandra) |
|---|---|---|
| Write pattern | Random in-place page writes | Sequential appends + background merges |
| Write amp | Page-sized write for a small change (+ WAL + full-page images/doublewrite) | Rewrites during compaction (leveled ~10–30×) |
| Read amp | ~1 random read for point lookup (internal nodes cached) | Possibly several (levels), mitigated by bloom filters & cache |
| Range scans | Excellent | Good but must merge iterators across levels |
| Space amp | Fragmentation (~30% for random inserts) | Low with leveled; high with size-tiered |
| Compression | Weaker (pages must be updatable) | Excellent (immutable files) |
| Latency predictability | Good | Compaction can cause latency spikes / write stalls |
| SSD wear | Higher (random small writes) | Lower per logical write in many workloads |
| Typical home | OLTP relational | High write ingest, time series, KV at scale |

MyRocks (RocksDB under MySQL) at Facebook cut storage ~50% vs InnoDB for the UDB (user DB) because of compression and lower space amplification.

---

## 10. Row Stores vs Column Stores {#columnar}

**Row store** (PG, MySQL): all columns of a row stored together. Great for OLTP: fetch/update a whole row by key.

**Column store** (ClickHouse, Redshift, BigQuery, Snowflake, DuckDB, Parquet files, Vertica): each column stored separately.
- Analytics scan few columns of many rows (`SUM(amount) WHERE date …`) → read only needed columns.
- Same-typed values compress extremely well (run-length, dictionary, delta, bit-packing) → 5–20× compression.
- **Vectorized execution**: process batches of column values with SIMD.
- Writes are expensive per row → batch inserts; updates/deletes are often async (ClickHouse mutations) or via merge-on-read.
- **Zone maps / min-max indexes** per block skip data without traditional indexes.

HTAP attempts: TiDB (TiFlash), SingleStore, PG with columnar extensions (Citus columnar, Hydra), MySQL HeatWave.

Rule: don't run heavy analytics on your OLTP primary — replicate (CDC) to a warehouse/OLAP store.

---

## 11. Other Index Structures {#other}

| Structure | Supports | Used by |
|---|---|---|
| **Hash index** | Equality only, O(1) | PG `USING hash` (WAL-logged since PG 10), MEMORY engine, InnoDB *adaptive hash index* (automatic, in-memory, for hot B-tree pages) |
| **Bitmap index** | Low-cardinality columns, fast AND/OR | Oracle, warehouses. PG builds *in-memory* bitmaps at query time (Bitmap Heap Scan) but has no persistent bitmap index |
| **Inverted index** | "Which docs contain term X" | Full-text: Elasticsearch/Lucene, PG GIN (also for arrays, JSONB), MySQL FULLTEXT |
| **GiST / R-tree** | Spatial, ranges, nearest-neighbor, overlaps | PostGIS, PG range types, exclusion constraints; MySQL SPATIAL (R-tree) |
| **SP-GiST** | Space-partitioned (quad-trees, k-d trees, radix tries) | PG: IPs, phone prefixes, points |
| **BRIN** | Block min/max summaries; tiny | PG for huge, naturally ordered tables (time-series append) |
| **Bloom filter index** | Multi-column equality, probabilistic | PG `bloom` extension; LSM SSTables |
| **Trie / radix** | Prefix search | In-memory engines, routing tables |
| **Skip list** | Ordered in-memory | Redis sorted sets, LSM memtables |
| **Vector indexes (HNSW, IVF)** | Approximate nearest neighbor on embeddings | pgvector, Milvus, Pinecone, Elasticsearch, MySQL HeatWave / MariaDB 11.7 vectors |

---

## 12. Large Values {#toast}

- **PG TOAST** (The Oversized-Attribute Storage Technique): values larger than ~2 KB are compressed (pglz or lz4 in PG 14+) and/or moved out-of-line into a TOAST table in ~2 KB chunks. The main tuple holds a pointer. `SELECT *` on rows with big JSONB/TEXT = extra reads + decompression; select only needed columns. Updating any column of a row rewrites the tuple, but unchanged TOASTed values are reused (pointer copied).
- **InnoDB** (DYNAMIC row format): large `VARCHAR/BLOB/TEXT/JSON` values that don't fit store a 20-byte pointer in the row and the value in overflow pages.
- Page-level compression: InnoDB `ROW_FORMAT=COMPRESSED` or transparent page compression; PG relies on TOAST + filesystem (ZFS/btrfs) compression.
- Store big blobs (images, PDFs) in **object storage** (S3) and keep only the key/URL + metadata in the DB. Blobs in the DB bloat backups, replication, and buffer pool.

---

## 13. Interview Questions {#qa}

1. Why do databases use B+trees instead of binary search trees or hash tables?
2. What happens on a page split? Why do random UUID primary keys hurt InnoDB more than PG?
3. Explain the WAL rule. Why is commit fast even though data pages aren't written?
4. What's a torn page and how do PG and InnoDB defend against it?
5. Walk through an LSM write and read. What are tombstones and why can they hurt reads?
6. Compare write/read/space amplification of B-tree vs LSM. When would you choose RocksDB/Cassandra?
7. Why is a column store faster for `SUM(amount) GROUP BY day` over a billion rows?
8. What's the difference between a clustered and a non-clustered index? What's a covering index?
9. `innodb_flush_log_at_trx_commit=2` — what exactly can you lose?
10. Why must the working set fit in memory? How would you measure it?
