# Replication, High Availability, Backup and Disaster Recovery

> Engine-agnostic concepts. Engine specifics: [`postgresql/08-replication-ha-backup.md`](../postgresql/08-replication-ha-backup.md), [`mysql/05-replication-and-ha.md`](../mysql/05-replication-and-ha.md), [`mysql/07-backup-recovery-operations.md`](../mysql/07-backup-recovery-operations.md). Distributed consistency theory is in [`scaling-db/cap.md`](../../scaling-db/cap.md).

## Table of Contents
1. [Why Replicate](#why)
2. [What Gets Shipped: Physical vs Logical vs Statement](#what)
3. [Topologies: Single-Leader, Multi-Leader, Leaderless](#topologies)
4. [Synchronous vs Asynchronous vs Semi-Sync vs Quorum](#sync)
5. [Replication Lag and the Consistency Guarantees It Breaks](#lag)
6. [Failover: Detection, Promotion, Fencing, Split-Brain](#failover)
7. [HA Architectures in Practice](#ha)
8. [Backups: Logical, Physical, Snapshots, Incremental](#backup)
9. [Point-in-Time Recovery (PITR)](#pitr)
10. [RPO, RTO, and DR Strategy](#rpo)
11. [Testing Restores (the part everyone skips)](#test)
12. [Famous Incidents and Lessons](#incidents)
13. [Interview Questions](#qa)

---

## 1. Why Replicate {#why}

1. **High availability**: survive the loss of a node or zone.
2. **Read scaling**: serve reads from replicas.
3. **Latency**: put replicas near users (multi-region reads).
4. **Isolation of workloads**: analytics, backups, and ETL run on replicas.
5. **Zero-downtime upgrades and migrations**: logical replication to a new version or cluster, then switch.

**Replication is not backup.** A `DROP TABLE` or a bad `UPDATE` without `WHERE` replicates to every replica within milliseconds. (A *delayed replica*, e.g. one hour behind, is a partial exception.)

---

## 2. What Gets Shipped {#what}

| Method | What flows | Examples | Pros | Cons |
|---|---|---|---|---|
| **Physical / WAL shipping** | Byte-level page changes (redo) | PG streaming replication, Aurora storage replication, InnoDB redo (Aurora MySQL), Oracle Data Guard physical | Exact copy, low overhead, everything replicated (DDL, sequences) | Same major version & architecture; replica is the whole cluster (no per-table selection); read-only replica |
| **Logical (row-based)** | Row changes: "row with PK 5 changed col x from a to b" | PG logical replication / logical decoding (pgoutput, wal2json), MySQL binlog `ROW` format, Debezium CDC | Cross-version, selective tables, cross-engine (CDC to Kafka), replicas can be writable / have extra indexes | DDL not replicated (PG), sequence values not replicated (PG — sync them manually at cutover), tables need PKs or replica identity, more overhead |
| **Statement-based** | SQL statements | MySQL binlog `STATEMENT` (legacy) | Compact | **Non-deterministic** statements (`NOW()`, `UUID()`, `LIMIT` without ORDER BY, triggers) diverge replicas |
| **Trigger-based** | Triggers capture changes into queue tables | Slony, Londiste, Bucardo (legacy PG) | Flexible | Heavy write overhead |

MySQL's default `binlog_format=ROW` (since 5.7.7) is the safe choice.

---

## 3. Topologies {#topologies}

### Single-leader (primary/replica, master/slave)
All writes go to one leader; followers apply its log. Simple and consistent on the leader. This is by far the most common setup for relational DBs.
- Cascading replicas (replica of a replica) reduce load on the primary.
- Read replicas can lag.

### Multi-leader (multi-primary, active-active)
Several nodes accept writes and replicate to each other (MySQL group replication multi-primary mode, Galera, BDR / pgEdge for PG, CouchDB, multi-region DynamoDB Global Tables, Cassandra across DCs).
- **Write conflicts** are inevitable when two leaders modify the same row concurrently:
  - Last-writer-wins (LWW) by timestamp loses data silently and depends on clock sync.
  - Application-defined merge, CRDTs (counters, sets that merge mathematically), version vectors + sibling resolution.
  - Avoidance: route all writes for a given key/tenant/user to one "home" region.
- Use cases: multi-region writes with low latency, offline clients (each device is a leader), collaborative editing.

### Leaderless (Dynamo-style)
Clients (or a coordinator) write to **N** replicas and read from several (Cassandra, ScyllaDB, Riak, DynamoDB internals, Voldemort).
- **Quorums**: write to W, read from R; if **W + R > N**, the read set overlaps the write set, so you see the latest write (modulo edge cases: sloppy quorums, concurrent writes, partial failures).
- Typical: N=3, W=2, R=2 (`QUORUM`). `W=1, R=1` → fast and weak; `W=N` → slow and fragile.
- Repair: **read repair** (fix stale replicas found during reads), **anti-entropy** (Merkle-tree comparison in the background), **hinted handoff** (a node holds writes for a down peer and delivers them later).

---

## 4. Synchronous vs Asynchronous {#sync}

| Mode | Commit waits for | Data loss on primary failure | Latency | Availability risk |
|---|---|---|---|---|
| **Asynchronous** | Local durability only | Up to lag (seconds of commits lost on failover) | Lowest | Primary never blocked by replicas |
| **Semi-synchronous** (MySQL) | ≥1 replica *received* (not applied) the event | Near-zero (falls back to async after timeout!) | +1 RTT | Timeout fallback can silently hide that you're async |
| **Synchronous** (PG `synchronous_standby_names`) | Replica flushed (`remote_write` / `on` / `remote_apply` levels) | Zero for committed txns | +1 RTT + replica fsync | If sync replica dies, primary **blocks writes**. So use `ANY 1 (r1, r2)` quorum among 2+ standbys |
| **Quorum / consensus** (Raft/Paxos: Aurora, Spanner, CockroachDB, MySQL Group Replication, etcd) | Majority of replicas | Zero; tolerates minority failure | +RTT to majority | Needs odd number (3/5); loses availability without majority |

Aurora: 6 copies across 3 AZs, write quorum 4/6, read quorum 3/6 at the storage layer. It survives losing an AZ plus one more node for reads.

Cross-region sync replication adds 50–150 ms per commit. That's usually why cross-region replication is async, which means accepting a non-zero RPO for region-level disasters.

---

## 5. Replication Lag and Broken Guarantees {#lag}

Causes: replica slower hardware, single-threaded apply (old MySQL; PG's WAL apply is single-process), long-running queries on the replica conflicting with apply (PG `max_standby_streaming_delay` / hot standby conflicts), big transactions or DDL, network bandwidth, and replica I/O saturation.

Guarantees that async replicas break, and the fixes:
| Guarantee | Problem | Fix |
|---|---|---|
| **Read-your-writes** | User doesn't see own update | Read own data from leader; LSN/GTID-wait |
| **Monotonic reads** | Data "goes back in time" across requests | Sticky replica per user |
| **Consistent prefix reads** | See an answer before its question (different partitions, different lag) | Causally related writes to the same partition; causal tokens |

MySQL parallel apply: `replica_parallel_workers` with `LOGICAL_CLOCK` and `WRITESET` dependency tracking (8.0) lets replicas apply non-conflicting transactions concurrently, which hugely reduces lag.

---

## 6. Failover {#failover}

Steps:
1. **Detect** that the leader is dead (heartbeat timeouts). Too short → false positives; too long → longer outage. Network partitions make "dead" ambiguous.
2. **Choose** a new leader: the most up-to-date replica (least data loss), or via consensus.
3. **Reconfigure**: clients and other replicas point to the new leader (DNS/VIP/proxy/service discovery update).
4. **Fence** the old leader so it cannot accept writes if it comes back (STONITH: "shoot the other node in the head", revoke its storage access, or epoch/generation numbers that make stale leaders' writes rejectable).

Hazards:
- **Split-brain**: two nodes both believe they're leader and accept writes → divergent data that is very painful to merge. Prevent with consensus (a majority must agree) plus fencing.
- **Lost writes with async replication**: writes acknowledged by the old leader but not replicated are lost. If the old leader rejoins, its divergent transactions must be discarded (`pg_rewind` in PG; MySQL errant GTIDs).
- **Auto-increment reuse**: the new leader reissues IDs the old one already gave out, and those IDs may exist in other systems (cache, Redis, external). GitHub's 2012 incident involved this.
- **Cascading failover flapping**: aggressive automation fails over repeatedly. Add cooldowns.

---

## 7. HA Architectures {#ha}

| Stack | How it works |
|---|---|
| **AWS RDS Multi-AZ (instance)** | Synchronous block-level replication to a standby in another AZ; DNS flip in ~60–120 s; standby not readable |
| **RDS Multi-AZ DB cluster** | 1 writer + 2 readable standbys, semi-sync, faster failover (~35 s) |
| **Aurora** | Shared distributed storage; replicas read the same storage; failover ~30 s or less; Global Database for cross-region (<1 s lag typical) |
| **PG: Patroni** + etcd/Consul/ZooKeeper | Patroni uses the DCS as a leader lock; promotes replicas, fences via lease expiry; HAProxy/pgBouncer/VIP route to leader. Industry standard self-managed |
| **PG: CloudNativePG / Zalando / Crunchy operators** | Kubernetes operators wrapping similar logic |
| **PG: repmgr, pg_auto_failover** | Simpler alternatives |
| **MySQL: InnoDB Cluster** (Group Replication + MySQL Router + Shell) | Paxos-based group communication; single-primary by default |
| **MySQL: Orchestrator** + semi-sync + ProxySQL | GitHub's historic setup: topology discovery and automated promotion |
| **MySQL: Galera / Percona XtraDB Cluster** | Virtually synchronous multi-primary via certification |
| **Vitess** | Sharding + VTOrc failover for MySQL at massive scale (YouTube, Slack, PlanetScale) |
| **Distributed SQL** (CockroachDB, YugabyteDB, TiDB, Spanner) | Raft per range; HA built in; survive node/zone/region loss given replica placement |

---

## 8. Backups {#backup}

| Type | How | Pros | Cons |
|---|---|---|---|
| **Logical** | Export SQL/data: `pg_dump`, `mysqldump`, `mydumper`, MySQL Shell `util.dumpInstance` | Portable across versions, selective (one table), human-readable | Slow to restore large DBs (must rebuild indexes); `pg_dump` is consistent via snapshot; mysqldump needs `--single-transaction` for InnoDB consistency |
| **Physical (file-level)** | Copy data files + WAL: `pg_basebackup`, **pgBackRest**, **Barman**, WAL-G; **Percona XtraBackup**, MySQL Enterprise Backup, MySQL Clone plugin | Fast backup/restore for big DBs; enables PITR | Same major version/platform; whole cluster |
| **Storage snapshots** | EBS/disk snapshots, ZFS, LVM | Very fast, incremental at block level | Must be crash-consistent (or coordinate with the DB: `pg_backup_start`, `FLUSH TABLES WITH READ LOCK`/`LOCK INSTANCE FOR BACKUP`); same-region by default |
| **Managed automated backups** | RDS/Aurora/Cloud SQL daily snapshot + continuous log archiving | PITR to any second within retention (1–35 days) | Deleted with the instance unless retained! Copy cross-account/region |

Incremental/differential: pgBackRest (full/diff/incr, block-level incremental), PG 17 native incremental `pg_basebackup --incremental`, XtraBackup incremental via LSN.

**3-2-1 rule**: 3 copies, 2 different media, 1 offsite. Modern addition: **1 immutable/air-gapped copy** (S3 Object Lock, separate cloud account) for ransomware and for compromised credentials that could delete backups. Encrypt backups and protect the keys separately.

---

## 9. Point-in-Time Recovery {#pitr}

PITR = **base backup + continuous archive of the transaction log** (WAL in PG, binlog in MySQL). Restore the base backup, then replay logs up to a target timestamp, LSN, or transaction — e.g. "right before the bad `DELETE` at 14:03:17".

- PG: `archive_command`/`archive_library` (or pgBackRest/WAL-G push) plus restore settings `restore_command`, `recovery_target_time`, `recovery_target_action = promote|pause`.
- MySQL: restore the XtraBackup copy, then `mysqlbinlog --start-position … --stop-datetime '…' binlog.0000* | mysql`.

PITR granularity depends on log archiving frequency: `archive_timeout` in PG forces segment switches so idle systems still archive regularly.

---

## 10. RPO, RTO, and DR Strategy {#rpo}

- **RPO (Recovery Point Objective)**: how much data loss is acceptable, measured as time. Sync replication → ~0. Async → seconds. Nightly dump → up to 24 h.
- **RTO (Recovery Time Objective)**: how long until the service is back. Hot standby failover → seconds to minutes. Restore 5 TB from S3 → hours.

DR tiers (cheapest to most expensive):
| Strategy | RPO | RTO | Cost |
|---|---|---|---|
| Backup & restore (to another region) | Hours | Hours–day | $ |
| Pilot light (DB replica running in DR region, app infra off) | Seconds–minutes | Tens of minutes | $$ |
| Warm standby (scaled-down full stack in DR) | Seconds | Minutes | $$$ |
| Multi-site active-active | ~0 | ~0 | $$$$ + application complexity (conflicts) |

Design the RPO/RTO with the business. "Zero data loss and zero downtime across regions" costs synchronous cross-region consensus (latency) plus active-active app design.

---

## 11. Testing Restores {#test}

> "Nobody cares about backups. They care about restores."

- **Automate restore drills**: regularly (weekly/monthly) restore the latest backup into a scratch environment, run integrity checks (row counts, checksums, app smoke tests), and record the elapsed time (that's your real RTO).
- Verify PITR to a random timestamp.
- Monitor: backup job success, backup age, WAL/binlog archive lag (`pg_stat_archiver.failed_count`), and backup size trend (a sudden drop signals a problem).
- Practice failover in game days (planned switchover in production, chaos testing).
- Keep runbooks current, and make sure on-call engineers have actually executed them.

---

## 12. Incidents and Lessons {#incidents}

- **GitLab, Jan 2017**: an engineer ran `rm -rf` on the primary's data directory (intending the replica) during a replication-lag firefight. Of 5 backup mechanisms, none worked as expected: pg_dump was silently failing (version mismatch), and alerts went to an email that was being rejected. Recovery came from a 6-hour-old staging snapshot, and ~6 hours of data was lost. Lessons: test restores, monitor backup success, add guardrails on destructive commands.
- **GitHub, Oct 2018**: a 43-second network partition between US East and West triggered an Orchestrator failover to the West coast. Writes landed in both regions, and the result was 24 hours of degraded service to reconcile. Lesson: cross-region failover policies must account for latency and in-flight writes; prefer fencing and region-aware promotion constraints.
- **Many teams**: replicas promoted with lost async writes, and auto-increment IDs reused → mismatched references in caches and search indexes.

---

## 13. Interview Questions {#qa}

1. Physical vs logical replication. When would you choose each?
2. Explain W + R > N. Is it sufficient for linearizability? (No: sloppy quorums, concurrent writes, and LWW clocks all break it.)
3. Your primary dies with async replication. What can be lost, and how do you minimize it?
4. What is split-brain, and how do HA systems prevent it?
5. Replication is not backup. Explain and give an example.
6. Design a backup strategy with RPO 5 min and RTO 1 h for a 2 TB Postgres.
7. Why might semi-sync replication give you a false sense of safety?
8. How would you upgrade a major Postgres/MySQL version with minimal downtime? (Logical replication / blue-green; RDS Blue/Green deployments.)
9. How do you serve reads from replicas without users seeing stale data after their own writes?
10. What's the difference between RPO and RTO? Give examples of architectures for each tier.
