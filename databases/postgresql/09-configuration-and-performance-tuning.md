# PostgreSQL Configuration and Performance Tuning

> Defaults are designed to start on a tiny machine, not to perform. This is a practical baseline, with the reasoning behind each value. Always measure before and after.

## Table of Contents
1. [How Configuration Works](#how)
2. [Memory](#memory)
3. [WAL and Checkpoints](#wal)
4. [Planner Cost Settings](#planner)
5. [Parallelism and Async I/O](#parallel)
6. [Connections](#connections)
7. [Autovacuum](#autovacuum)
8. [Logging and Observability](#logging)
9. [Timeouts and Safety](#safety)
10. [OS and Hardware](#os)
11. [A Baseline for a 64 GB / 16 vCPU OLTP Server](#baseline)
12. [Performance Investigation Method](#method)
13. [Interview Questions](#qa)

---

## 1. How Configuration Works {#how}

- Files: `postgresql.conf`, `postgresql.auto.conf` (written by `ALTER SYSTEM SET …`, which overrides the former), plus `include` directives.
- Levels, from lowest to highest precedence: config file → `ALTER DATABASE … SET` → `ALTER ROLE … SET` → `ALTER ROLE … IN DATABASE … SET` → session `SET` → transaction `SET LOCAL`.
- **Context** column in `pg_settings` says what's needed to change it: `postmaster` (restart), `sighup` (reload: `SELECT pg_reload_conf();`), `superuser`/`user` (session).
```sql
SELECT name, setting, unit, context, source, pending_restart FROM pg_settings WHERE name LIKE '%work_mem%';
SHOW shared_buffers;
```
- Starting-point tools: **PGTune** (pgtune.leopard.in.ua), `timescaledb-tune`. Managed services pre-tune many of these.

---

## 2. Memory {#memory}

| Setting | Guidance | Why |
|---|---|---|
| `shared_buffers` | ~25% of RAM (8–16 GB for 64 GB). Sometimes up to 40% | Rest of RAM used by OS cache; too large → double caching waste and longer checkpoints |
| `effective_cache_size` | ~50–75% of RAM | Planner hint only (no allocation); higher → more index scans considered |
| `work_mem` | 16–64 MB for OLTP; raise per-session for reports | **Per sort/hash node per process**. Worst case ≈ connections × nodes × work_mem × (1 + parallel workers) × hash_mem_multiplier. Find spills via `log_temp_files = 0` |
| `hash_mem_multiplier` | 2.0 (default since PG 15) | Hash ops get more memory than sorts |
| `maintenance_work_mem` | 1–2 GB | Faster CREATE INDEX, VACUUM |
| `autovacuum_work_mem` | 512 MB–1 GB | Per autovacuum worker |
| `huge_pages` | `try` (configure `vm.nr_hugepages`) | Lower TLB overhead for big shared_buffers |
| `temp_file_limit` | e.g. 20 GB | Stops a runaway query from filling the disk |

Check buffer cache hit ratio (aim for > 99% for OLTP):
```sql
SELECT sum(blks_hit)*100.0/nullif(sum(blks_hit)+sum(blks_read),0) AS hit_pct FROM pg_stat_database;
```
A "read" here can still be served from the OS page cache, so a low hit ratio with low disk I/O is OK-ish. Use `pg_stat_io` (PG 16+) and OS metrics too. `pg_buffercache` shows what's currently in shared buffers; `pg_prewarm` warms the cache after restarts.

---

## 3. WAL and Checkpoints {#wal}

| Setting | Guidance |
|---|---|
| `wal_level` | `replica` (or `logical` if CDC needed) |
| `max_wal_size` | 4–64 GB depending on write volume. Higher means fewer checkpoints, less full-page-image WAL, and longer crash recovery |
| `min_wal_size` | 1–4 GB |
| `checkpoint_timeout` | 15 min (default 5 min) |
| `checkpoint_completion_target` | 0.9 (default since PG 14) — spread checkpoint writes |
| `wal_compression` | `lz4` or `zstd` (PG 15+) — fewer WAL bytes from full-page images |
| `wal_buffers` | -1 (auto, 16 MB) is fine; 64 MB for very write-heavy |
| `synchronous_commit` | `on` by default; consider `off` per transaction for low-value writes |
| `commit_delay` / `commit_siblings` | Group commit tuning on slow-fsync storage |
| `wal_writer_delay` | 200 ms default |

Check `log_checkpoints = on` (default on in PG 15+). Lines like *"checkpoints are occurring too frequently"* or many "requested" (vs "timed") checkpoints in `pg_stat_checkpointer` (PG 17; earlier `pg_stat_bgwriter`) mean `max_wal_size` is too small.

---

## 4. Planner Cost Settings {#planner}

| Setting | SSD/cloud value | Why |
|---|---|---|
| `random_page_cost` | 1.1–1.5 | Default 4.0 assumes spinning disks; with SSDs random ≈ sequential |
| `effective_io_concurrency` | 200 (SSD) / 16 default in PG 18 | Prefetching for bitmap heap scans (and more with AIO) |
| `maintenance_io_concurrency` | 100+ | |
| `default_statistics_target` | 100 (raise per column for skewed data) | Histogram/MCV sizes |
| `jit` | `off` for OLTP is common | JIT compile time often exceeds savings on short queries; sometimes causes surprising latency |
| `plan_cache_mode` | `auto` | `force_custom_plan` if generic plans misbehave on skewed data |

---

## 5. Parallelism and Async I/O {#parallel}

| Setting | Guidance |
|---|---|
| `max_worker_processes` | ≥ cores (restart) |
| `max_parallel_workers` | ≈ cores |
| `max_parallel_workers_per_gather` | 2–4 for mixed workloads; 0 to disable for pure OLTP latency predictability |
| `max_parallel_maintenance_workers` | 2–4 (index builds) |
| `io_method` (PG 18) | `worker` (default) or `io_uring` (Linux): asynchronous reads for seq scans, bitmap heap scans, vacuum |
| `io_workers` (PG 18) | 3 default; increase on fast storage |

---

## 6. Connections {#connections}

- `max_connections`: keep it modest (100–500) and put **PgBouncer** in front for thousands of clients. Each connection reserves shared memory slots (lock table, proc array) even when idle.
- `superuser_reserved_connections` (3) and `reserved_connections` (PG 16, for `pg_use_reserved_connections` role) keep admin access during connection storms.
- `tcp_keepalives_idle/interval/count` to detect dead clients.
- `client_connection_check_interval` (PG 14) cancels queries whose client disconnected.

---

## 7. Autovacuum {#autovacuum}

See [`02-mvcc-and-vacuum.md`](02-mvcc-and-vacuum.md). Baseline:
```
autovacuum_max_workers = 5
autovacuum_naptime = 15s
autovacuum_vacuum_cost_limit = 2000
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_scale_factor = 0.05     # per-table overrides for big tables (0.005–0.01)
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_insert_scale_factor = 0.05
log_autovacuum_min_duration = 1s
```

---

## 8. Logging and Observability {#logging}

```
logging_collector = on
log_destination = 'stderr'                 # or jsonlog (PG 15+) for structured logs
log_line_prefix = '%m [%p] %q%u@%d app=%a client=%h '   # time, pid, user@db, application, host
log_min_duration_statement = 250ms         # or use auto_explain + sampling
log_lock_waits = on
log_temp_files = 0
log_checkpoints = on
log_connections = on                       # can be noisy; PG 18 makes it more granular
log_disconnections = on
log_autovacuum_min_duration = 1s
log_statement = 'ddl'                      # audit schema changes
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.track = top
track_io_timing = on                       # I/O time in EXPLAIN BUFFERS and pg_stat_statements (cheap on modern clocks; verify with pg_test_timing)
track_wal_io_timing = on
compute_query_id = on
```
Tools: **pgBadger** (log analysis reports), **pganalyze**, **PMM** (Percona Monitoring and Management), Datadog DBM, **postgres_exporter** + Grafana, pg_activity / pgcenter (top-like TUI), pg_stat_kcache.

Key metrics to graph: TPS (xact_commit/rollback), connections by state, cache hit ratio, rows fetched/returned/inserted/updated/deleted, deadlocks, temp bytes, replication lag, WAL rate, checkpoint stats, dead tuples and autovacuum activity, transaction ID age, slot retained WAL, longest transaction age, lock waits, disk usage and growth.

---

## 9. Timeouts and Safety {#safety}

```sql
ALTER ROLE app_user SET statement_timeout = '15s';
ALTER ROLE app_user SET idle_in_transaction_session_timeout = '60s';
ALTER ROLE app_user SET lock_timeout = '5s';
ALTER ROLE reporting SET statement_timeout = '10min';
ALTER ROLE app_user SET transaction_timeout = '60s';      -- PG 17
-- server-wide
idle_session_timeout = 0           # careful: poolers keep idle sessions intentionally
max_slot_wal_keep_size = 100GB
```

---

## 10. OS and Hardware {#os}

- **Storage**: NVMe/SSD; on cloud, provisioned IOPS (io2/gp3 with tuned IOPS/throughput). Latency of fsync bounds commit rate. Separate WAL volume can help on network storage.
- **Filesystem**: ext4 or XFS, `noatime`. ZFS (compression, snapshots) is viable with tuning (recordsize, `full_page_writes` trade-offs).
- **Linux**:
  - `vm.swappiness = 1`. Avoid swapping shared memory.
  - `vm.overcommit_memory = 2` (with appropriate `overcommit_ratio`) so the OOM killer doesn't shoot the postmaster. Or at least protect the postmaster with `oom_score_adj`.
  - Disable **Transparent Huge Pages** (`never`) and use explicit huge pages.
  - `vm.dirty_background_bytes` / `vm.dirty_bytes` lowered so the kernel flushes steadily rather than in giant bursts.
  - I/O scheduler `none`/`mq-deadline` for NVMe.
- **CPU**: high single-thread performance matters (each query mostly runs on one core).
- **RAM**: enough to hold the working set (hot tables + indexes).
- Benchmark with `pgbench` (built-in TPC-B-like or custom scripts: `pgbench -c 32 -j 8 -T 300 -f custom.sql`) and `fio` for raw disk performance.

---

## 11. Baseline: 64 GB RAM / 16 vCPU / SSD, OLTP {#baseline}

```ini
# Memory
shared_buffers = 16GB
effective_cache_size = 48GB
work_mem = 32MB
maintenance_work_mem = 2GB
autovacuum_work_mem = 1GB
huge_pages = try

# Connections (PgBouncer in front)
max_connections = 300

# WAL / checkpoints
wal_compression = lz4
max_wal_size = 16GB
min_wal_size = 2GB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

# Planner
random_page_cost = 1.1
effective_io_concurrency = 200
default_statistics_target = 200
jit = off

# Parallelism
max_worker_processes = 16
max_parallel_workers = 16
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 4

# Autovacuum
autovacuum_max_workers = 5
autovacuum_naptime = 15s
autovacuum_vacuum_cost_limit = 2000
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02

# Safety
idle_in_transaction_session_timeout = 60s
max_slot_wal_keep_size = 100GB
temp_file_limit = 50GB

# Observability
shared_preload_libraries = 'pg_stat_statements,auto_explain'
track_io_timing = on
log_min_duration_statement = 250ms
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 1s
auto_explain.log_min_duration = 1s
```
This is a starting point, not gospel. Validate with your workload.

---

## 12. Investigation Method {#method}

When "the database is slow":
1. **Is it the DB?** Check app traces. Maybe it's pool wait time, the network, or N+1 query counts.
2. **Resource saturation** (USE method: Utilization, Saturation, Errors): CPU, memory (swap?), disk IOPS/throughput/latency/queue depth, network.
3. **What's running now**: `pg_stat_activity` grouped by state and wait_event. Many `Lock` waits → blocking chain. `IO` waits → disk. `LWLock` → internal contention. `Client` → app slow to read results.
4. **What changed**: deploys, data growth, new query, stats/plan flip, autovacuum or vacuum not running, a replica or slot issue, a checkpoint storm.
5. **Top queries**: `pg_stat_statements` delta over the incident window.
6. **Plans**: auto_explain output for the slow ones.
7. **Fix → verify → write it down** (postmortem, add the alert that would have caught it).

---

## 13. Interview Questions {#qa}

1. How would you size `shared_buffers` and `work_mem`? Why can a large `work_mem` crash the server?
2. What does `random_page_cost` do and why change it on SSD?
3. What happens if `max_wal_size` is too small?
4. Why put PgBouncer in front instead of raising `max_connections` to 5000?
5. Which settings protect a production database from runaway sessions?
6. Your cache hit ratio dropped from 99.9% to 95%. What might have happened?
7. Why disable transparent huge pages for Postgres?
