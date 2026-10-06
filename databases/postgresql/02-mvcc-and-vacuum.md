# PostgreSQL MVCC, VACUUM, Bloat, and Wraparound

> The most Postgres-specific topic there is. Most "Postgres got slow over time" stories and some of its worst outages (wraparound shutdowns) come from misunderstanding this.

## Table of Contents
1. [Tuple Versioning: xmin, xmax, ctid](#tuples)
2. [Visibility Rules](#visibility)
3. [See It Yourself (hands-on)](#handson)
4. [Dead Tuples and Bloat](#bloat)
5. [HOT Updates and Fillfactor](#hot)
6. [What VACUUM Does (and Doesn't)](#vacuum)
7. [Autovacuum Tuning](#autovacuum)
8. [Freezing and XID Wraparound](#wraparound)
9. [Things That Block VACUUM (the xmin horizon)](#horizon)
10. [Measuring and Fixing Bloat](#fix)
11. [Visibility Map and Index-Only Scans](#vm)
12. [Queue Tables and Other MVCC-Hostile Patterns](#hostile)
13. [Interview Questions](#qa)

---

## 1. Tuple Versioning {#tuples}

Each heap tuple header (23 bytes + alignment) includes:
- **`xmin`**: XID of the transaction that inserted this version.
- **`xmax`**: XID of the transaction that deleted/updated it (or locked it). 0 if live.
- **`ctid`**: physical location `(page, slot)`. For an updated tuple, points to the newer version → a version chain.
- **infomask** bits: hint bits (xmin committed/aborted, xmax committed/aborted), HOT flags, lock flags.
- `cmin/cmax` (command IDs within a transaction, overlaid in one field).

Operations:
- **INSERT**: new tuple with `xmin = me`, `xmax = 0`.
- **DELETE**: set `xmax = me` on the existing tuple. Nothing is physically removed.
- **UPDATE** = DELETE + INSERT: old tuple gets `xmax = me` and its ctid points to the new tuple; new tuple gets `xmin = me`.

So **every UPDATE writes a whole new row**, even if you change one boolean. Wide rows plus frequent updates mean heavy write amplification.

---

## 2. Visibility Rules {#visibility}

Simplified: a tuple is visible to a snapshot if
1. `xmin` is committed **and** was committed before the snapshot (not in the snapshot's in-progress list and < snapshot xmax), **or** `xmin` is my own transaction (and inserted by an earlier command), **and**
2. `xmax` is 0, **or** `xmax` aborted, **or** `xmax` is not yet committed as of the snapshot (in progress, or committed after the snapshot), **or** `xmax` is just a row lock (not a delete).

Snapshot (`pg_current_snapshot()`): `xmin:xmax:xip_list`, e.g. `100:105:101,103`. XIDs < 100 are finished; ≥ 105 haven't started (invisible); 101 and 103 are in progress (invisible); 100, 102, 104 are finished (visible if committed).

---

## 3. Hands-On {#handson}

```sql
CREATE TABLE t (id int PRIMARY KEY, v text);
INSERT INTO t VALUES (1, 'a');
SELECT ctid, xmin, xmax, * FROM t;
--  ctid  | xmin | xmax | id | v
-- (0,1)  | 742  |   0  |  1 | a

UPDATE t SET v = 'b' WHERE id = 1;
SELECT ctid, xmin, xmax, * FROM t;
-- (0,2)  | 743  |   0  |  1 | b        ← new version in slot 2; slot 1 is now dead

-- Look at raw page contents (dead tuples included):
CREATE EXTENSION pageinspect;
SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('t', 0));
-- lp | t_xmin | t_xmax | t_ctid
--  1 |   742  |  743   | (0,2)     ← old version, deleted by 743, points to new
--  2 |   743  |    0   | (0,2)

VACUUM t;
SELECT lp, lp_flags, t_xmin, t_xmax FROM heap_page_items(get_raw_page('t', 0));
-- slot 1 now a redirect / unused
```

Two sessions:
```sql
-- Session A                          -- Session B
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT v FROM t WHERE id=1;  -- 'b'
                                      UPDATE t SET v='c' WHERE id=1;  -- autocommit
SELECT v FROM t WHERE id=1;  -- still 'b' (snapshot)
UPDATE t SET v='d' WHERE id=1;
-- ERROR: could not serialize access due to concurrent update
```

---

## 4. Dead Tuples and Bloat {#bloat}

A **dead tuple** is a version no running or future transaction can see. Until VACUUM removes it, it:
- Occupies space in heap pages → tables grow, fewer live rows per page, more I/O, less effective cache.
- Has index entries pointing to it → index scans fetch heap pages only to discover the tuple is dead.
- Slows sequential scans.

**Bloat** = space taken by dead tuples plus free space that won't be reused efficiently. Regular VACUUM marks the space reusable *within the table* (via the free space map). It does **not** return it to the OS, except for truncating empty pages at the very end of the table.

So: a table that had a 100M-row DELETE stays the same size on disk after VACUUM (but future inserts reuse the space). To shrink: `VACUUM FULL` (rewrites, ACCESS EXCLUSIVE lock — downtime), **`pg_repack`** (online), or `CLUSTER` (rewrite + order, also exclusive).

Index bloat: B-tree pages split but rarely merge. PG 13+ deduplication and PG 14+ **bottom-up index deletion** (removes duplicate version entries during page splits caused by non-HOT updates) greatly reduce it. Rebuild with **`REINDEX CONCURRENTLY`** (PG 12+).

---

## 5. HOT Updates and Fillfactor {#hot}

**Heap-Only Tuple (HOT)** update: if
1. no column referenced by **any** index is changed (including partial-index predicates and expression-index inputs), and
2. the new version fits on the **same heap page**,

then PG doesn't insert new index entries. The old tuple's line pointer forwards to the new tuple within the page (a HOT chain), and old versions can be pruned **on-page** without a full VACUUM ("HOT pruning", done opportunistically by any query touching the page).

Benefits: no index write amplification, less WAL, less bloat.

Maximize HOT:
- Don't index frequently updated columns unless needed (an index on `updated_at` kills HOT for every update!).
- Lower **fillfactor** on update-heavy tables so pages have room: `ALTER TABLE accounts SET (fillfactor = 80);` (applies to new pages; rewrite with pg_repack to apply to existing ones).
- Check: `SELECT relname, n_tup_upd, n_tup_hot_upd, round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct FROM pg_stat_user_tables ORDER BY n_tup_upd DESC;`

PG 16 made HOT possible when only BRIN-indexed columns change ("summarizing indexes").

---

## 6. What VACUUM Does {#vacuum}

`VACUUM` (plain, also what autovacuum runs):
1. Scans heap pages (skipping all-visible pages via the visibility map, unless freezing aggressively).
2. Collects dead tuple TIDs into memory (`maintenance_work_mem`/`autovacuum_work_mem`; PG 17's radix-tree TID store made this far more memory-efficient, so one index pass is usually enough).
3. Scans **each index** and removes entries pointing to dead TIDs (index vacuuming — often the most expensive part on tables with many indexes).
4. Marks the heap line pointers unused; updates FSM.
5. Updates the **visibility map** (all-visible / all-frozen bits).
6. **Freezes** old tuples (see §8).
7. Updates `pg_class.relpages/reltuples`, and with `ANALYZE` refreshes stats.
8. Truncates empty pages at the end of the table (needs a brief ACCESS EXCLUSIVE lock; can be disabled with `vacuum_truncate = off` if it causes lock issues on replicas).

Plain VACUUM runs concurrently with reads and writes (SHARE UPDATE EXCLUSIVE lock — conflicts only with DDL and other VACUUMs).

Variants:
- `VACUUM (VERBOSE, ANALYZE) t;`
- `VACUUM (FREEZE) t;` — aggressive freeze.
- `VACUUM (INDEX_CLEANUP OFF)` — emergency wraparound mode (skip indexes).
- `VACUUM (PARALLEL 4)` — parallel index vacuuming (manual VACUUM only).
- `VACUUM FULL` — rewrite (exclusive lock).
- `vacuumdb --all --analyze-in-stages` after major upgrades/restores.

---

## 7. Autovacuum Tuning {#autovacuum}

Autovacuum triggers on a table when:
```
dead tuples > autovacuum_vacuum_threshold (50) + autovacuum_vacuum_scale_factor (0.2) × reltuples
```
and for inserts (PG 13+): `autovacuum_vacuum_insert_threshold` (1000) + `autovacuum_vacuum_insert_scale_factor` (0.2) × reltuples. Analyze similarly with `autovacuum_analyze_scale_factor` (0.1).

**The default 20% is far too lax for big tables**: a 1-billion-row table accumulates 200 million dead tuples before vacuum starts, then takes hours.

Typical tuning:
```sql
-- Per hot table
ALTER TABLE events SET (
  autovacuum_vacuum_scale_factor = 0.01,     -- 1%
  autovacuum_vacuum_threshold = 10000,
  autovacuum_analyze_scale_factor = 0.005,
  autovacuum_vacuum_cost_limit = 2000
);
```
Global (postgresql.conf) for modern hardware:
```
autovacuum_max_workers = 5                  # default 3 (PG 18 allows changing without restart up to autovacuum_worker_slots)
autovacuum_naptime = 15s                    # default 1min
autovacuum_vacuum_cost_limit = 2000         # default 200 (-1 → vacuum_cost_limit) — the throttle
autovacuum_vacuum_cost_delay = 2ms          # default 2ms since PG 12 (was 20ms)
maintenance_work_mem / autovacuum_work_mem = 1GB
```
The cost limit is **shared among all workers** — more workers without raising the limit makes each slower.

Monitor:
```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0*n_dead_tup/nullif(n_live_tup+n_dead_tup,0),1) AS dead_pct,
       last_autovacuum, last_autoanalyze, autovacuum_count
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Running vacuums and progress
SELECT * FROM pg_stat_progress_vacuum;
```
Log slow autovacuums: `log_autovacuum_min_duration = '1s'` (default 10 min in PG 15+).

**Never disable autovacuum.** If it "uses too much I/O", that's a sign it's behind; tune it to run more often with smaller batches.

---

## 8. Freezing and XID Wraparound {#wraparound}

Problem: XIDs are 32-bit, compared modulo 2³². A tuple with xmin = 100 looks "in the past" today, but after ~2.1 billion more transactions it would look "in the future" and become **invisible** — data silently disappearing.

Solution: **freezing** — VACUUM marks old tuples as frozen (infomask bit `HEAP_XMIN_FROZEN`), meaning "visible to everyone forever, ignore xmin".

Key settings:
| Setting | Default | Meaning |
|---|---|---|
| `vacuum_freeze_min_age` | 50M | Tuples older than this get frozen when vacuum visits their page |
| `vacuum_freeze_table_age` | 150M | When table's `relfrozenxid` age exceeds this, VACUUM scans all non-frozen pages (aggressive) |
| `autovacuum_freeze_max_age` | 200M | Forces an **anti-wraparound autovacuum** even if autovacuum is disabled for the table. These can't be cancelled by lock conflicts — the classic "autovacuum (to prevent wraparound)" blocking your DDL |
| `vacuum_failsafe_age` | 1.6B | PG 14+: vacuum drops cost limits and skips index cleanup to finish freezing ASAP |
| Hard stop | ~2B − 3M | Server refuses new XIDs: `database is not accepting commands to avoid wraparound data loss`. Before PG 16-ish you had to go single-user mode; now you run VACUUM in normal mode |

Same mechanism exists for **MultiXact IDs** (used when multiple transactions lock the same row, e.g., FK checks): `autovacuum_multixact_freeze_max_age`. Real outages have come from multixact wraparound too.

Monitor (alert at ~ 500M–1B):
```sql
SELECT datname, age(datfrozenxid) AS xid_age, mxid_age(datminmxid) AS mxid_age FROM pg_database ORDER BY 2 DESC;

SELECT c.oid::regclass, age(c.relfrozenxid) AS xid_age, pg_size_pretty(pg_total_relation_size(c.oid))
FROM pg_class c WHERE c.relkind IN ('r','m','t') ORDER BY 2 DESC LIMIT 20;
```

Real incidents: Sentry (2015), Mailchimp/Mandrill (2019), Joyent — high-write systems where vacuum couldn't keep up or was blocked → hours of downtime.

---

## 9. The xmin Horizon: What Blocks VACUUM {#horizon}

VACUUM can only remove tuples dead to **everyone**. The oldest snapshot anywhere sets the horizon. Things that hold it back:

1. **Long-running transactions** (including `idle in transaction` sessions and long analytical queries at RR).
2. **Abandoned replication slots** — a logical/physical slot that no consumer reads keeps `xmin`/`catalog_xmin` *and* retains WAL forever → disk full. Check `pg_replication_slots` (`active`, `restart_lsn`, `wal_status`), set `max_slot_wal_keep_size`. PG 18 adds `idle_replication_slot_timeout`.
3. **`hot_standby_feedback = on`** on a replica with long queries → replica's oldest snapshot is reported to the primary.
4. **Orphaned prepared transactions** (`pg_prepared_xacts`).
5. Long-running `CREATE INDEX` (non-concurrent ones on other tables are fine; `CREATE INDEX CONCURRENTLY` holds a snapshot).

Find the culprit:
```sql
SELECT pid, datname, usename, state, backend_xmin, age(backend_xmin) AS xmin_age,
       now() - xact_start AS xact_duration, left(query, 80)
FROM pg_stat_activity WHERE backend_xmin IS NOT NULL ORDER BY age(backend_xmin) DESC LIMIT 10;

SELECT slot_name, slot_type, active, xmin, catalog_xmin,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;

SELECT gid, prepared, owner FROM pg_prepared_xacts;
```

Safety settings: `idle_in_transaction_session_timeout = '60s'`, `statement_timeout` per role, `transaction_timeout` (PG 17), `max_slot_wal_keep_size`.

---

## 10. Measuring and Fixing Bloat {#fix}

Measure:
- Estimation queries (ioguide/pgexperts bloat queries) — fast, approximate.
- `pgstattuple` extension: exact (`SELECT * FROM pgstattuple('orders');` → `dead_tuple_percent`, `free_percent`), `pgstattuple_approx` faster; `pgstatindex('idx')` → `avg_leaf_density`.

Fix:
| Situation | Fix |
|---|---|
| Table bloat moderate, steady state | Tune autovacuum; space gets reused |
| Severe table bloat, need space back | `pg_repack -t table` (online; needs PK/unique index & ~2× space) |
| Index bloat | `REINDEX INDEX CONCURRENTLY idx;` |
| Massive delete of old data | Use **partitioning** next time: `DROP`/`DETACH PARTITION` is instant and bloat-free |
| Update-heavy table | Lower fillfactor, avoid indexing hot columns (HOT) |

---

## 11. Visibility Map and Index-Only Scans {#vm}

The VM holds 2 bits per heap page: **all-visible** (every tuple visible to all transactions) and **all-frozen**.
- **Index-only scans** check the VM: if the page is all-visible, skip the heap. Otherwise fetch the heap page to check visibility (`Heap Fetches` in EXPLAIN ANALYZE). Frequent VACUUM keeps index-only scans truly index-only.
- VACUUM skips all-visible pages (normal) and all-frozen pages (even aggressive).
- Any modification clears the page's bits.
- Insert-only tables before PG 13 never triggered autovacuum (no dead tuples) → VM never set → index-only scans degraded, and freezing happened all at once at 200M XIDs. PG 13's insert-triggered autovacuum fixed this.

---

## 12. MVCC-Hostile Patterns {#hostile}

| Pattern | Problem | Better |
|---|---|---|
| **Queue table** with high-churn INSERT → UPDATE status → DELETE | Dead tuples pile up at the head; `SELECT … ORDER BY id LIMIT 1` scans thousands of dead index entries | `FOR UPDATE SKIP LOCKED` + aggressive autovacuum on the table; partition by time and drop; or pgmq / a real queue |
| **Counter row** updated thousands of times/sec | Each update creates a version; HOT chains; lock contention | Shard counters, batch increments, Redis |
| **Session/last-seen timestamp updates** on wide user rows | Rewrites the full wide row each time | Move hot columns into a narrow separate table |
| Large batch UPDATE of whole table | Doubles table size; bloat | Batch + vacuum between batches, or create new table + swap |
| `UPDATE` setting a column to its same value | Still creates a new version! | `WHERE col IS DISTINCT FROM new_value` |
| Long analytics queries on primary | Holds horizon → bloat everywhere | Run on replica (with `hot_standby_feedback` off + accept cancellations) or warehouse |

---

## 13. Interview Questions {#qa}

1. How does PG implement MVCC differently from InnoDB? Pros and cons of each.
2. Why does an UPDATE in Postgres create bloat? What is a HOT update and how do you encourage it?
3. What does VACUUM do vs VACUUM FULL? Why doesn't VACUUM shrink the file?
4. Explain transaction ID wraparound and how to monitor for it.
5. A table keeps growing even though row count is stable. Diagnose.
6. What can prevent VACUUM from removing dead tuples?
7. Why is an inactive replication slot dangerous?
8. Why might an index-only scan still read the heap?
9. How would you tune autovacuum for a 2-billion-row, write-heavy table?
