# MySQL Performance Tuning

## Table of Contents
1. [Tuning Philosophy](#philosophy)
2. [InnoDB Memory and I/O Settings](#innodb)
3. [Durability vs Throughput](#durability)
4. [Connections, Threads and Per-Session Buffers](#sessions)
5. [Temporary Tables and Sorting](#temp)
6. [Optimizer-Related Settings](#optimizer)
7. [Baseline my.cnf for 64 GB / 16 vCPU OLTP](#baseline)
8. [OS and Hardware](#os)
9. [Monitoring: What to Graph](#monitoring)
10. [Benchmarking](#bench)
11. [Troubleshooting Playbook](#playbook)
12. [Interview Questions](#qa)

---

## 1. Tuning Philosophy {#philosophy}

1. **Queries and indexes first.** No server setting rescues a full table scan run 1000×/s.
2. Then the buffer pool size (fit the working set), redo log capacity, and I/O capacity.
3. Change one thing at a time, measure with realistic load, keep a changelog.
4. Don't copy "magic" configs. Many legacy tips (query cache, `innodb_thread_concurrency`, huge `sort_buffer_size`) are obsolete or harmful.
5. `innodb_dedicated_server = ON` (8.0+) auto-sizes the buffer pool, redo capacity, and flush method for a dedicated host. It's a good starting point.

---

## 2. InnoDB Memory and I/O {#innodb}

| Setting | Guidance |
|---|---|
| `innodb_buffer_pool_size` | 60–75% of RAM on a dedicated host. The #1 setting |
| `innodb_buffer_pool_instances` | Default (8 when ≥ 1 GB) is fine |
| `innodb_redo_log_capacity` | Enough for ~1 h of peak redo; commonly 4–32 GB |
| `innodb_log_buffer_size` | 64 MB (8.4 default); larger for huge transactions. Watch `Innodb_log_waits` |
| `innodb_flush_method` | `O_DIRECT` on Linux (avoid double buffering) |
| `innodb_io_capacity` | What the storage sustains for background flushing: 2,000–20,000 on SSD (8.4 default 10,000) |
| `innodb_io_capacity_max` | ~2× io_capacity |
| `innodb_flush_neighbors` | 0 on SSD |
| `innodb_read_io_threads` / `innodb_write_io_threads` | 4–16; check pending I/O in INNODB STATUS |
| `innodb_purge_threads` | 4 (more for heavy DML/high HLL) |
| `innodb_page_cleaners` | = buffer pool instances |
| `innodb_adaptive_hash_index` | OFF (8.4 default) for write-heavy/high-concurrency; test ON for read-heavy point lookups |
| `innodb_change_buffering` | `none` (8.4 default) on SSD |
| `innodb_file_per_table` | ON |
| `innodb_stats_persistent_sample_pages` | 20 default; raise for big skewed tables (or per table) |
| `innodb_numa_interleave` | ON on multi-socket NUMA hosts |

Buffer pool sizing check: if `Innodb_buffer_pool_reads` (disk reads) per second is high and the data is bigger than the pool, add RAM or reduce the working set (archive old data, smaller rows and indexes).

---

## 3. Durability vs Throughput {#durability}

| Config | Durability | Use |
|---|---|---|
| `innodb_flush_log_at_trx_commit=1`, `sync_binlog=1` | Full ACID; no committed transaction lost on power failure | Primaries holding important data (**default**) |
| `=2`, `sync_binlog=0/1000` | Lose ≤ ~1 s on OS crash | Replicas that can be rebuilt, bulk-load windows, non-critical data |
| `=0` | Lose ≤ ~1 s even on mysqld crash | Rarely justified |

With a fast NVMe with power-loss protection, `1/1` costs little thanks to group commit (`binlog_group_commit_sync_delay` can deliberately batch more at the cost of latency). On cloud network disks, fsync latency (~0.5–2 ms) directly caps single-thread commit rate.

---

## 4. Connections and Session Buffers {#sessions}

| Setting | Guidance |
|---|---|
| `max_connections` | Size for the real peak (+ headroom), e.g. 500–2000. Each idle connection is cheap-ish, but active ones compete for CPU |
| `thread_cache_size` | Auto-sized; check `Threads_created` rate (should be near 0) |
| `table_open_cache` / `table_definition_cache` | Enough for (tables × concurrent sessions). Watch `Opened_tables` rate |
| `sort_buffer_size` | Keep default (256 KB). Allocated per sort, per session. Raise per session for specific reports |
| `join_buffer_size` | Default 256 KB; per join without index (and per hash join chunk) |
| `read_buffer_size`, `read_rnd_buffer_size` | Defaults |
| `max_allowed_packet` | 64 MB (8.0 default) or more for big BLOB/JSON rows and bulk inserts |
| `wait_timeout` / `interactive_timeout` | Close abandoned connections (e.g., 600 s), but keep it longer than the app pool's idle timeout, or pooled connections die and the app sees "MySQL server has gone away" |

Thread pool (Percona/MariaDB/Enterprise) for thousands of active connections. Otherwise, use ProxySQL multiplexing.

---

## 5. Temporary Tables and Sorting {#temp}

- Internal temp tables (GROUP BY, DISTINCT, UNION, derived tables, window functions) use the **TempTable engine** in memory up to `temptable_max_ram` (1 GB default; 8.4 makes it 3% of RAM bounded 1–4 GB), then spill to mmap files or InnoDB on-disk temp tables (`temptable_max_mmap`).
- `tmp_table_size` limits individual in-memory temp tables. `max_heap_table_size` applies to MEMORY tables.
- Monitor `Created_tmp_disk_tables` vs `Created_tmp_tables`. A high disk ratio means rewriting queries or adding indexes (so GROUP BY/ORDER BY can use index order).
- Filesort uses `sort_buffer_size`. `Sort_merge_passes` > 0 means multi-pass sorts on disk.

---

## 6. Optimizer Settings {#optimizer}

- `optimizer_switch`: leave defaults unless you've diagnosed a specific problem (e.g., `prefer_ordering_index=off` in 8.0.21+ when the optimizer picks an ORDER BY index with LIMIT badly, which is a known trap).
- `eq_range_index_dive_limit`: 200. Raise it if large IN lists get bad estimates, at a planning-time cost.
- `optimizer_search_depth`: 62 (auto). Lower it if joins of many tables take long to plan.
- Histograms on skewed non-indexed filter columns.
- `range_optimizer_max_mem_size`: exceeding it on huge IN/OR lists → the optimizer abandons range access → full scan. A real gotcha with thousands of IN items.

---

## 7. Baseline my.cnf: 64 GB / 16 vCPU / NVMe OLTP (MySQL 8.4) {#baseline}

```ini
[mysqld]
# Basics
server_id                         = 1
datadir                           = /var/lib/mysql
character_set_server              = utf8mb4
collation_server                  = utf8mb4_0900_ai_ci
default_time_zone                 = '+00:00'
sql_mode                          = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'
sql_require_primary_key           = ON
skip_name_resolve                 = ON

# Connections
max_connections                   = 1000
wait_timeout                      = 600
max_allowed_packet                = 256M
table_open_cache                  = 8000
table_definition_cache            = 4000

# InnoDB memory & I/O
innodb_buffer_pool_size           = 44G
innodb_redo_log_capacity          = 16G
innodb_log_buffer_size            = 64M
innodb_flush_method               = O_DIRECT
innodb_io_capacity                = 10000
innodb_io_capacity_max            = 20000
innodb_flush_neighbors            = 0
innodb_purge_threads              = 4
innodb_read_io_threads            = 8
innodb_write_io_threads           = 8
innodb_lock_wait_timeout          = 10
innodb_print_all_deadlocks        = ON
innodb_autoinc_lock_mode          = 2

# Durability
innodb_flush_log_at_trx_commit    = 1
sync_binlog                       = 1

# Binlog / replication
log_bin                           = binlog
binlog_format                     = ROW
binlog_row_image                  = FULL
binlog_expire_logs_seconds        = 604800
binlog_transaction_compression    = ON
gtid_mode                         = ON
enforce_gtid_consistency          = ON
replica_parallel_workers          = 16
replica_preserve_commit_order     = ON
relay_log_recovery                = ON

# Temp tables
temptable_max_ram                 = 2G
tmp_table_size                    = 64M
max_heap_table_size               = 64M

# Observability
slow_query_log                    = ON
long_query_time                   = 0.2
log_slow_extra                    = ON
performance_schema                = ON
log_error_verbosity               = 2
```
Validate against your workload and version docs. Defaults and variable names change between releases.

---

## 8. OS and Hardware {#os}

- Linux: XFS or ext4 with `noatime`. `vm.swappiness=1`. Disable Transparent Huge Pages (they cause latency spikes with jemalloc/glibc allocations). Consider **jemalloc** to reduce fragmentation.
- NUMA: `innodb_numa_interleave=ON` or `numactl --interleave=all`.
- I/O scheduler `none`/`mq-deadline` for NVMe.
- File descriptor limits (`open_files_limit`, systemd `LimitNOFILE`).
- Cloud: pick storage IOPS/throughput to match `innodb_io_capacity` and the checkpoint rate. Local NVMe instances (i4i, etc.) are much faster, but you need replication or backups for durability.

---

## 9. What to Graph {#monitoring}

| Metric | Source | Why |
|---|---|---|
| QPS / TPS (`Questions`, `Com_select/insert/update/delete`, `Com_commit`) | STATUS | Load |
| Threads_running (not Threads_connected) | STATUS | **Concurrency saturation**. Should stay well below core count × 2 most of the time |
| Query latency p95/p99 per digest | performance_schema / PMM | User impact |
| Buffer pool hit ratio, `Innodb_buffer_pool_reads`/s, dirty pages % | STATUS | Memory sizing |
| Checkpoint age vs redo capacity | `log_status` / INNODB STATUS | Flush stalls |
| History list length | innodb_metrics | Long transactions / purge lag |
| Row lock waits & time, deadlocks | STATUS | Contention |
| `Created_tmp_disk_tables`, `Sort_merge_passes`, `Select_full_join`, `Select_scan` | STATUS | Bad queries |
| Replication lag (heartbeat), applier worker status | pt-heartbeat / P_S | Read staleness, failover readiness |
| Aborted_connects / Aborted_clients, Connection_errors_* | STATUS | Auth/network problems |
| Disk latency, IOPS, CPU iowait | OS | Storage saturation |

Tools: **PMM**, mysqld_exporter + Grafana, Datadog, `innotop`, `pt-mysql-summary`, `pt-stalk` (captures diagnostics when a trigger condition happens), `mysqladmin ext -ri1` (status deltas per second).

---

## 10. Benchmarking {#bench}

- **sysbench**: `sysbench oltp_read_write --tables=16 --table-size=10000000 --threads=64 --time=600 run`.
- **mysqlslap** (simple), HammerDB (TPC-C-like), **tpcc-mysql**.
- Replay production traffic: Percona's `pt-upgrade` (compare query results/perf between versions), ProxySQL mirroring, or `query-playback`.
- Warm up the buffer pool before measuring, run long enough to hit steady-state flushing, and watch p99 not just averages.

---

## 11. Troubleshooting Playbook {#playbook}

**Sudden spike in Threads_running / latency**
1. `SELECT * FROM sys.processlist WHERE command <> 'Sleep' ORDER BY time DESC;`, then group by state (Sending data, Waiting for table metadata lock, statistics, updating, executing).
2. Lock waits → `sys.innodb_lock_waits`, MDL waits → `sys.schema_table_lock_waits`.
3. New query digest? `events_statements_summary_by_digest` sorted by `FIRST_SEEN`.
4. Plan flip after stats recalculation → `ANALYZE TABLE`, histograms, hint as a temporary fix via the query rewrite plugin or ProxySQL.

**Periodic stalls every few minutes**
- Checkpoint/flush storms: checkpoint age hitting the limit → raise `innodb_redo_log_capacity` and `innodb_io_capacity`.
- Purge spikes, or cron jobs (backups, analytics).

**Replica lag**: see the replication note.

**Disk full**: binlogs (retention), relay logs, undo tablespaces (long transaction), temp files, slow log, `ibdata1` growth (change buffer).

**"Too many connections"**: connection leaks, pool misconfiguration, slow queries causing pile-up. Check `max_used_connections`. Keep an admin connection available: 8.0.14+ `admin_address`/`admin_port`.

---

## 12. Interview Questions {#qa}

1. What are the most impactful MySQL settings and why?
2. How would you size the buffer pool and the redo log?
3. What does `innodb_io_capacity` control? What happens if it's too low?
4. Explain the durability trade-offs of `innodb_flush_log_at_trx_commit` and `sync_binlog`.
5. Why shouldn't you raise `sort_buffer_size` globally to 64 MB?
6. Threads_connected vs Threads_running: which one indicates trouble?
7. You see periodic throughput drops every 5 minutes. What could it be?
