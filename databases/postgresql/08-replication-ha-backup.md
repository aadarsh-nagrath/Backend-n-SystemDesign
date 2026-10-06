# PostgreSQL Replication, High Availability, Backup and Upgrades

> Concepts (sync vs async, failover, RPO/RTO): [`fundamentals/08-replication-ha-backup-recovery.md`](../fundamentals/08-replication-ha-backup-recovery.md). This note covers Postgres mechanics and commands.

## Table of Contents
1. [WAL: The Foundation of Everything](#wal)
2. [Physical Streaming Replication](#streaming)
3. [Replication Slots](#slots)
4. [Synchronous Replication Settings](#sync)
5. [Hot Standby: Queries on Replicas and Conflicts](#hotstandby)
6. [Logical Replication and Logical Decoding (CDC)](#logical)
7. [Failover and HA with Patroni](#patroni)
8. [Backups: pg_dump, pg_basebackup, pgBackRest, WAL-G](#backup)
9. [Point-in-Time Recovery Walkthrough](#pitr)
10. [Major Version Upgrades](#upgrades)
11. [Monitoring Replication](#monitoring)
12. [Interview Questions](#qa)

---

## 1. WAL {#wal}

- WAL lives in `pg_wal/` in 16 MB segments (`--wal-segsize` at initdb). Records carry LSNs like `0/3A2B4C8`.
- `wal_level`:
  - `minimal`: crash recovery only. Some bulk operations skip WAL, so no replication is possible.
  - `replica` (default): enough for physical replication and archiving.
  - `logical`: adds the information needed for logical decoding (slightly more WAL).
- WAL volume drivers: write rate, **full-page images after checkpoints** (more frequent checkpoints → more FPIs), index count, non-HOT updates. Check `pg_stat_wal` and `EXPLAIN (ANALYZE, WAL)`. `wal_compression = lz4/zstd` (PG 15+) shrinks FPIs.
- Useful functions: `pg_current_wal_lsn()`, `pg_walfile_name(lsn)`, `pg_wal_lsn_diff(a, b)`, `pg_switch_wal()`. `pg_waldump` decodes WAL files.

---

## 2. Physical Streaming Replication {#streaming}

The standby connects to the primary, a **walsender** streams WAL records, and the standby's **walreceiver** writes them while the **startup process** replays them. The standby is a byte-identical copy, read-only (hot standby).

Setup (PG 12+):
```bash
# On primary: postgresql.conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
# pg_hba.conf
host replication replicator 10.0.0.0/24 scram-sha-256

# On standby: clone the primary (creates standby.signal and primary_conninfo with -R)
pg_basebackup -h primary -U replicator -D /var/lib/postgresql/data -X stream -C -S standby1_slot -R -P
# start postgres → it streams
```
Key standby settings: `primary_conninfo`, `primary_slot_name`, `hot_standby = on`, `recovery_min_apply_delay = '1h'` (**delayed replica**: protection against operator mistakes, since you can stop it before a bad `DROP` replays).

Cascading replication: a standby can feed other standbys.

Promotion: `pg_ctl promote` or `SELECT pg_promote();`. The standby exits recovery and starts a **new timeline** (timeline ID increments; history files track the branches).

Re-joining an old primary after failover: its WAL diverged, so run **`pg_rewind`** to rewind it to the branch point (requires `wal_log_hints = on` or data checksums), then start it as a standby. Otherwise, re-clone it.

---

## 3. Replication Slots {#slots}

A slot makes the primary **retain WAL** (and, for logical slots, catalog rows) until the consumer confirms it.
- Without slots, a lagging standby may find its needed WAL already recycled, so it breaks and must be re-cloned (unless WAL archiving lets it catch up).
- With slots, a dead or forgotten consumer means **unbounded WAL growth until the disk fills and the primary goes down**. This is a top cause of PG outages.

Protect yourself:
```sql
-- postgresql.conf
max_slot_wal_keep_size = '100GB'      -- invalidate slot beyond this instead of filling disk (PG 13+)

SELECT slot_name, slot_type, active, wal_status, safe_wal_size,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;

SELECT pg_drop_replication_slot('old_debezium_slot');
```
Alert on inactive slots and retained WAL size.

**Failover slots** (PG 17): logical slots can be synchronized to standbys (`sync_replication_slots`, `failover = true`), so CDC consumers (Debezium) survive a primary failover without data loss or a resnapshot. Before PG 17 this was a major gap (Patroni had a workaround for permanent slots).

---

## 4. Synchronous Replication {#sync}

```
synchronous_standby_names = 'ANY 1 (standby_a, standby_b)'   # quorum: any one of them
# or 'FIRST 1 (standby_a, standby_b)' — priority-based
synchronous_commit = on        # wait for standby to flush WAL to disk
```
`synchronous_commit` levels (per transaction too, via `SET LOCAL synchronous_commit = …`):
| Level | Commit waits for |
|---|---|
| `off` | Nothing (local WAL flushed asynchronously; up to ~3× wal_writer_delay of commits lost on crash, but **no corruption**) |
| `local` | Local flush only |
| `remote_write` | Standby received and wrote to OS (not fsynced) |
| `on` | Standby flushed to disk |
| `remote_apply` | Standby replayed it (readers on the standby will see it, giving read-your-writes on that standby) |

Per-transaction tuning example: payments use `on`; analytics events use `off`.

Danger: with `FIRST 1 (a)` and only one sync standby, if that standby dies, all commits on the primary **hang**. Always have ≥2 candidates with `ANY 1`, and have HA tooling (Patroni's `synchronous_mode`) manage it.

---

## 5. Hot Standby Conflicts {#hotstandby}

Replaying WAL on a standby can conflict with queries running there. For example, the primary VACUUMs away rows that a long standby query still needs, or takes an ACCESS EXCLUSIVE lock (DROP, TRUNCATE, vacuum truncation).

Options:
| Setting | Effect |
|---|---|
| `max_standby_streaming_delay = 30s` (default) | Replay waits up to 30 s, then **cancels** the conflicting query (`canceling statement due to conflict with recovery`) |
| `-1` | Wait forever, so replication lag grows unbounded |
| `hot_standby_feedback = on` | Standby tells the primary its oldest xmin, so the primary doesn't vacuum those rows. Fewer cancellations, but **bloat on the primary** from long standby queries |

Check `pg_stat_database_conflicts` on the standby.

Analytics on a replica: dedicate a replica with a long delay setting and accept lag there, or use logical replication to a separate reporting database, or move analytics to a warehouse.

---

## 6. Logical Replication and Decoding {#logical}

### Built-in publish/subscribe (PG 10+)
```sql
-- Publisher (wal_level = logical)
CREATE PUBLICATION app_pub FOR TABLE orders, customers;     -- or FOR ALL TABLES / FOR TABLES IN SCHEMA app (PG 15)
-- PG 15: row filters and column lists
CREATE PUBLICATION paid_orders FOR TABLE orders (id, total, status) WHERE (status = 'paid');

-- Subscriber (schema must already exist; DDL is NOT replicated)
CREATE SUBSCRIPTION app_sub CONNECTION 'host=pub dbname=app user=repl' PUBLICATION app_pub;
-- initial table sync copies existing data, then streams changes
```
Properties:
- Row-level changes (INSERT/UPDATE/DELETE/TRUNCATE) applied via SQL-like apply. The subscriber is writable and can have different indexes, extra columns, or a different major version.
- Requires a **replica identity** (PK by default) for UPDATE/DELETE. Tables without a PK need `REPLICA IDENTITY FULL` (slow).
- **Not replicated**: DDL, sequence values, large objects. Plan schema changes on both sides and sync sequences at cutover.
- Conflicts (e.g., unique violation on the subscriber) stop the apply worker until fixed. PG 17 has better conflict logging, and `ALTER SUBSCRIPTION … SKIP` exists.
- Large transactions: `streaming = on`/`parallel` (PG 14/16) applies in-progress transactions instead of waiting.
- Bidirectional replication of the same tables needs `origin = none` (PG 16) to avoid loops, plus conflict handling. Use it with care.

Uses: zero-downtime **major version upgrades**, consolidating or splitting databases, replicating a subset to a reporting DB, and migrating to or from managed services.

### Logical decoding (CDC)
Output plugins turn WAL into change streams: `pgoutput` (built in), `wal2json`, `test_decoding`, `decoderbufs`.
- **Debezium** connects via a logical slot, snapshots, then streams changes into Kafka. This is the standard for CDC and the outbox pattern.
- Consumer must advance the slot regularly; otherwise WAL is retained (see slots).

---

## 7. HA with Patroni {#patroni}

Patroni (Zalando) is the de facto standard for self-managed PG HA:
- Each node runs a Patroni agent next to Postgres.
- A **DCS** (etcd, Consul, ZooKeeper, or Kubernetes API) holds the **leader key with a TTL**. The leader renews it, and if it fails to (crash or partition), replicas race to acquire it. The winner (healthy and least lagged) promotes.
- The old leader, unable to renew, **demotes itself** (self-fencing), and optionally a watchdog (softdog) reboots the node to ensure it's dead.
- Routing: HAProxy health-checks Patroni's REST API (`/primary`, `/replica`), or a VIP (vip-manager), or Consul DNS, or Kubernetes Services.
- `synchronous_mode: true` for RPO=0 with managed standby lists.
- `patronictl list | switchover | failover | reinit | edit-config`.

Kubernetes operators: **CloudNativePG** (doesn't use Patroni; uses the K8s API directly), Zalando postgres-operator (Patroni), Crunchy PGO (Patroni + pgBackRest), StackGres, Percona Operator.

Managed: RDS (Multi-AZ instance/cluster), Aurora PostgreSQL, Cloud SQL, AlloyDB, Azure Flexible Server, Neon (separates storage from compute; branching), Supabase, Crunchy Bridge.

---

## 8. Backups {#backup}

### pg_dump / pg_dumpall / pg_restore (logical)
```bash
pg_dump -Fc -d app -f app.dump                  # custom format (compressed, selective restore)
pg_dump -Fd -j 8 -d app -f app_dir              # directory format, parallel dump
pg_restore -j 8 -d app_new app.dump             # parallel restore
pg_restore -l app.dump > toc.txt                # list contents; edit; restore with -L toc.txt
pg_dumpall --globals-only > globals.sql         # roles & tablespaces (pg_dump excludes them!)
pg_dump -t 'public.orders' --data-only ...
```
- Consistent via a repeatable-read snapshot; doesn't block writes. A long dump holds back the xmin horizon, which causes bloat on busy systems.
- Restoring a large database is slow (index rebuilds, FK validation). Fine up to tens of GB, painful beyond.
- Good for: migrations across versions or architectures, partial restores, dev copies (with anonymization).

### pg_basebackup (physical)
```bash
pg_basebackup -D /backups/base_2025_06_01 -Ft -z -X stream -P -c fast
# PG 17 incremental:
pg_basebackup --incremental=/backups/base/backup_manifest -D /backups/incr1 ...  # requires summarize_wal = on
pg_combinebackup /backups/base /backups/incr1 -o /restore/full
```

### pgBackRest / Barman / WAL-G (production-grade)
- **pgBackRest**: full, differential, and incremental (block-level) backups, parallel compression (zstd/lz4), encryption, S3/GCS/Azure repos, async WAL archiving, backup verification, and **parallel restore with delta** (only changed files). The most common choice.
- **WAL-G**: lightweight, cloud-native, delta backups, popular on Kubernetes.
- **Barman** (EDB): backup server pulling from many PG servers.

WAL archiving config (pgBackRest example):
```
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_timeout = 60                         # force segment switch at least every 60s (bounds RPO on idle systems)
```
Monitor `pg_stat_archiver` (`failed_count`, `last_failed_wal`). A failing archive_command also makes `pg_wal` grow.

Data checksums: enable at initdb (`--data-checksums`, **default on in PG 18**) or offline via `pg_checksums --enable`. They detect storage corruption. `pgBackRest` verifies checksums during backup.

---

## 9. PITR Walkthrough {#pitr}

Scenario: someone ran `DELETE FROM orders;` at 14:03:17 UTC.
```bash
# 1. Find the time (logs, pg_stat_statements, app logs) → target 14:03:00
# 2. Restore the latest base backup before that point into a NEW data directory / new server
pgbackrest --stanza=main --type=time --target="2025-06-01 14:03:00+00" \
           --target-action=pause --delta restore
# 3. Start Postgres → replays WAL up to target, pauses
# 4. Verify: SELECT count(*) FROM orders;  (read-only while paused)
# 5a. If good: SELECT pg_wal_replay_resume(); → promotes (with target_action=promote)
# 5b. Or: pg_dump the orders table from the recovered instance and load into production
```
Underlying settings: `restore_command`, `recovery_target_time | _lsn | _xid | _name` (named restore points via `pg_create_restore_point('before_migration')`), `recovery_target_inclusive`, `recovery_target_action`.

Restoring onto a *separate* instance and copying data back is usually safer than rewinding all of production, because writes after 14:03 are preserved.

---

## 10. Major Version Upgrades {#upgrades}

| Method | Downtime | Notes |
|---|---|---|
| `pg_dump` + `pg_restore` | Long (hours for big DBs) | Simple, cleans bloat |
| **`pg_upgrade --link`** (or `--clone`/`--copy`, PG 18 adds `--swap`) | Minutes | Rewrites catalogs and hard-links data files. Needs both versions installed. Run `--check` first. Afterwards run `vacuumdb --analyze-in-stages` (PG 18 preserves optimizer stats, reducing this need). With `--link` you can't go back to the old cluster once the new one has started |
| **Logical replication** (blue/green) | Seconds (switchover) | Replicate to new-version cluster, sync sequences, switch traffic. Handles DDL freeze during migration. RDS **Blue/Green Deployments** automate it |
| Managed in-place upgrade (RDS) | Minutes, runs pg_upgrade | Take a snapshot first |

Upgrade checklist: test the application on the new version (planner changes!), check extension compatibility, update replicas (re-clone them, or pg_upgrade + rsync per docs), and re-analyze.

Minor upgrades (17.4 → 17.5) are just a binary swap + restart, but **read the release notes**, because some require REINDEX for specific bugs.

---

## 11. Monitoring Replication {#monitoring}

On the primary:
```sql
SELECT application_name, client_addr, state, sync_state,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn))   AS send_lag,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```
On the standby:
```sql
SELECT pg_is_in_recovery(),
       pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp() AS replay_delay;   -- misleading when primary is idle
SELECT * FROM pg_stat_wal_receiver;
```
Alert on: replay lag (bytes and time), slot retained WAL, archive failures, standby not streaming, sync standby missing, and timeline changes.

---

## 12. Interview Questions {#qa}

1. How does streaming replication work in Postgres? What's a timeline?
2. What is a replication slot, and what happens if a consumer disappears?
3. Explain `synchronous_commit` levels. How would you make only payment transactions synchronous?
4. Why do queries on a standby get cancelled, and what are the trade-offs of `hot_standby_feedback`?
5. Physical vs logical replication: what can logical do that physical can't, and vice versa?
6. How does Patroni prevent split-brain?
7. Design a backup strategy with PITR for a 3 TB database.
8. Walk through recovering from an accidental `DELETE` at a known time.
9. How do you upgrade from PG 13 to PG 17 with under one minute of downtime?
