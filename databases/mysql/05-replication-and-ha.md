# MySQL Replication and High Availability

## Table of Contents
1. [How Asynchronous Replication Works](#how)
2. [Binlog Formats](#formats)
3. [GTIDs](#gtid)
4. [Setting Up a Replica (8.0/8.4 syntax)](#setup)
5. [Semi-Synchronous Replication](#semisync)
6. [Parallel Replication and Lag](#parallel)
7. [Crash-Safe Replication](#crashsafe)
8. [Topologies](#topologies)
9. [Group Replication and InnoDB Cluster](#gr)
10. [Galera / Percona XtraDB Cluster](#galera)
11. [Orchestrator, ProxySQL, and Failover](#failover)
12. [Vitess: Sharding MySQL at Scale](#vitess)
13. [Managed Options](#managed)
14. [Common Replication Problems and Fixes](#problems)
15. [Interview Questions](#qa)

---

## 1. Asynchronous Replication {#how}

```
Source (primary)                                   Replica
──────────────────                                 ──────────────────────────────
Client commits → binlog (binlog.000123)
                  │
   Binlog dump thread  ── streams events ──►  Receiver (I/O) thread → relay log
                                                         │
                                               Applier (SQL) thread(s) → apply to data
```
- The source writes committed transactions to the binlog. A **binlog dump thread** per replica sends events.
- The replica's **receiver thread** (`Replica_IO_Running`) writes them to the local **relay log**.
- **Applier threads** (`Replica_SQL_Running`) replay relay log events.
- Commit on the source doesn't wait for replicas (**async**), so a failover can lose recently committed transactions.
- Terminology: since 8.0.22/8.0.23 `SOURCE`/`REPLICA` (`START REPLICA`, `SHOW REPLICA STATUS`, `CHANGE REPLICATION SOURCE TO`). 8.4 **removed** the old MASTER/SLAVE statements.

---

## 2. Binlog Formats {#formats}

| `binlog_format` | Logs | Pros | Cons |
|---|---|---|---|
| STATEMENT | SQL text | Small for bulk updates | **Unsafe with non-determinism**: `NOW()` is fine (timestamp is logged), but `UUID()`, `RAND()`, `LIMIT` without ORDER BY, `UPDATE … ORDER BY`, user-defined functions, and triggers can diverge |
| **ROW** (default) | Before/after images of each changed row | Deterministic; required for RC isolation, Group Replication, CDC (Debezium) | Big for mass updates (1 M-row UPDATE = 1 M row events) |
| MIXED | Statement unless unsafe → row | | Complexity |

Related settings:
- `binlog_row_image = FULL | MINIMAL | NOBLOB`: MINIMAL logs only the PK + changed columns, which is smaller, but CDC consumers may need FULL.
- `binlog_row_metadata = FULL` for column names in events (CDC friendliness).
- `binlog_transaction_compression = ON` (8.0.20+, zstd).
- `binlog_expire_logs_seconds` (default 30 days); `binlog_cache_size`; `max_binlog_size` (1 GB rotation).
- Read events: `mysqlbinlog --base64-output=DECODE-ROWS -vv binlog.000123`.

The binlog doubles as a **CDC source**: Debezium, Maxwell, Canal (Alibaba), and AWS DMS read it as a replica would.

---

## 3. GTIDs {#gtid}

**Global Transaction Identifier**: `source_server_uuid:transaction_number`, e.g. `3E11FA47-71CA-11E1-9E33-C80AA9429562:1-58423`.
- Each transaction committed on a server gets a unique GTID, which travels with it through replication.
- `gtid_executed` = the set of all GTIDs applied on a server.
- **Auto-positioning** (`SOURCE_AUTO_POSITION=1`): a replica tells the new source "I have these GTIDs", and the source sends what's missing. No more file/position bookkeeping, and failover/re-pointing becomes trivial.
- Enable: `gtid_mode=ON`, `enforce_gtid_consistency=ON` (disallows some unsafe statements, e.g., `CREATE TABLE … SELECT` before 8.0.21, and mixing transactional and non-transactional tables in one transaction).
- **Errant transactions**: GTIDs present on a replica but not on the source (someone wrote directly to a replica). If that replica is promoted, other replicas try to fetch those GTIDs, which may have been purged from the binlog, and replication breaks. Prevent with `super_read_only=ON` on replicas. Detect with `GTID_SUBTRACT(replica_executed, source_executed)`.
- Read-your-writes across replicas: after writing, read `@@gtid_executed` (or session-tracked last GTID), then on a replica `SELECT WAIT_FOR_EXECUTED_GTID_SET('uuid:1-1234', 1);`.

---

## 4. Setting Up a Replica {#setup}

```sql
-- On source (my.cnf): server_id=1, log_bin=ON (default 8.0), gtid_mode=ON, enforce_gtid_consistency=ON
CREATE USER 'repl'@'10.0.%' IDENTIFIED BY '…' REQUIRE SSL;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.%';   -- privilege name unchanged

-- Seed the replica's data: CLONE plugin (8.0.17+), XtraBackup, or MySQL Shell dump/load
-- CLONE (simplest, on the replica):
INSTALL PLUGIN clone SONAME 'mysql_clone.so';
SET GLOBAL clone_valid_donor_list = 'source-host:3306';
CLONE INSTANCE FROM 'clone_user'@'source-host':3306 IDENTIFIED BY '…';

-- On replica: server_id=2 (unique!), read_only=ON, super_read_only=ON, relay_log_recovery=ON
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'source-host', SOURCE_USER = 'repl', SOURCE_PASSWORD = '…',
  SOURCE_AUTO_POSITION = 1, SOURCE_SSL = 1;
START REPLICA;
SHOW REPLICA STATUS\G
-- Check: Replica_IO_Running: Yes, Replica_SQL_Running: Yes, Seconds_Behind_Source, Last_IO_Error, Last_SQL_Error,
--        Retrieved_Gtid_Set, Executed_Gtid_Set
```

Delayed replica (protection against `DROP TABLE` mistakes): `CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 3600;`

Filters: `replicate_do_db`, `replicate_ignore_table`, `replicate_wild_do_table` (careful: with STATEMENT format, filters apply by *default database*, not the table referenced, which is a classic bug).

---

## 5. Semi-Synchronous Replication {#semisync}

- The source waits, before acknowledging commit to the client, until at least `rpl_semi_sync_source_wait_for_replica_count` replicas **acknowledge receipt** (written to the relay log, not necessarily applied).
- `rpl_semi_sync_source_wait_point = AFTER_SYNC` (default, "lossless"): wait happens after the binlog sync but **before** the InnoDB commit is visible to other sessions. So no client can see a transaction that could be lost on failover. (`AFTER_COMMIT` = older behavior, where others might see data that then disappears after failover.)
- `rpl_semi_sync_source_timeout` (default 10 s): if no replica acks in time, the source **silently degrades to async**. Monitor `Rpl_semi_sync_source_status` and the `Rpl_semi_sync_source_no_tx` counter. Some teams set a huge timeout to prefer blocking over silent data-loss risk.
- Plugin names in 8.0.26+: `rpl_semi_sync_source` / `rpl_semi_sync_replica`.
- Cost: +1 network RTT per commit (group commit amortizes it).

---

## 6. Parallel Replication and Lag {#parallel}

Historically, a single applier thread was the bottleneck: the source commits with 64 concurrent threads, and the replica replays serially, so lag accumulates.

Multi-threaded replica (MTR):
- `replica_parallel_workers` (default **4** since 8.0.27; set to ~8–32).
- `replica_parallel_type = LOGICAL_CLOCK`: transactions that committed in the same group (or don't overlap) can be applied in parallel.
- **WRITESET dependency tracking** (`binlog_transaction_dependency_tracking = WRITESET` in 8.0; it's the only mode in 8.4): the source records hashes of the rows each transaction modified. Transactions with disjoint writesets can run in parallel even if they committed sequentially, which greatly improves parallelism.
- `replica_preserve_commit_order = ON` (default 8.0.27+): commits in source order, so replicas never expose states the source never had.

Lag causes and fixes:
| Cause | Fix |
|---|---|
| Single-threaded apply | MTR + WRITESET |
| Huge transactions (bulk UPDATE/DELETE of millions of rows) | Chunk them |
| DDL (an ALTER taking 1 h on the source takes 1 h on the replica *after*) | gh-ost / online DDL with throttling; instant DDL |
| Tables without PK under ROW format (each row event = full table scan on replica) | **Add PKs**; `sql_require_primary_key=ON` |
| Replica under-provisioned / serving heavy reads | Scale it; separate read pools |
| `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1` on replicas | Replicas can often use relaxed durability (rebuildable), if not failover candidates |

Measuring lag: `Seconds_Behind_Source` is computed from event timestamps and can show 0 while the receiver is stalled, or jump around. Use **pt-heartbeat** (writes a timestamp row on the source every second and compares on the replica) or `performance_schema.replication_applier_status_by_worker` timestamps (8.0: `LAST_APPLIED_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP`).

---

## 7. Crash-Safe Replication {#crashsafe}

Before 5.6, replica position was stored in files (`relay-log.info`) that could get out of sync with the data after a crash, leading to duplicate or skipped transactions. Now:
- Replication metadata lives in InnoDB tables (`mysql.slave_relay_log_info`, `slave_master_info`; `relay_log_info_repository=TABLE` is the only option in 8.0+), updated atomically with applied transactions.
- `relay_log_recovery = ON`: on restart, discard relay logs and re-fetch from the source from the last applied position.
- With GTID auto-positioning, recovery is even simpler.

---

## 8. Topologies {#topologies}

- **Source → N replicas** (star): most common.
- **Chained** (source → intermediate → replicas): reduces source load and cross-region bandwidth (requires `log_replica_updates=ON`, default 8.0). Lag compounds.
- **Multi-source replication**: one replica pulls from several sources (channels). Used for consolidation and analytics.
- **Circular / master-master** (two sources replicating each other): writes to both risk conflicts and auto-increment collisions (`auto_increment_increment`/`auto_increment_offset` hacks). Use it as **active-passive** only (writes go to one side at a time).
- **Cross-region**: async replicas in other regions for DR and local reads. InnoDB ClusterSet automates this.

---

## 9. Group Replication and InnoDB Cluster {#gr}

**Group Replication (GR)**: a plugin implementing a replicated state machine via a **Paxos-based group communication system (XCom)**.
- A transaction executes locally, and at commit its writeset is broadcast to the group. **Certification**: each member checks whether the writeset conflicts with concurrent transactions already ordered. If not, it commits everywhere; otherwise it rolls back on the originator.
- Requires: InnoDB, PK on every table, ROW binlog, GTIDs.
- **Single-primary mode** (default, recommended): one writable member, automatic primary election on failure.
- **Multi-primary mode**: all members writable. Conflicts cause rollbacks (optimistic), and there are DDL and isolation caveats (SERIALIZABLE not supported, FK cascades restricted).
- Needs a majority (3, 5, … up to 9 members). A minority partition blocks writes (prevents split-brain).
- **Flow control** throttles writers when members lag behind.
- **Consistency levels** (`group_replication_consistency`): `EVENTUAL` (default), `BEFORE_ON_PRIMARY_FAILOVER`, `BEFORE` (read waits for preceding writes: read-your-writes cluster-wide), `AFTER` (write waits until applied everywhere), `BEFORE_AND_AFTER`.

**InnoDB Cluster** = Group Replication + **MySQL Shell** (AdminAPI: `dba.createCluster()`, `cluster.addInstance()`, `cluster.status()`) + **MySQL Router** (routes app connections: R/W port 6446, R/O port 6447, follows the primary automatically).
**InnoDB ReplicaSet**: the same tooling for classic async replication (manual failover).
**InnoDB ClusterSet**: a primary cluster plus replica clusters in other regions (async), with controlled switchover/failover.

---

## 10. Galera / Percona XtraDB Cluster {#galera}

- **Virtually synchronous multi-primary**: writesets are replicated to all nodes at commit and **certified** (conflict check) in total order. Commit returns after all nodes have *certified* (not necessarily applied).
- Any node accepts writes. Conflicting concurrent writes on different nodes cause one to fail at commit with a deadlock error (**optimistic**), so the app must retry.
- **Every node holds full data** (no sharding). Write throughput is bounded by the slowest node and WAN latency.
- Flow control pauses the cluster when a node falls behind.
- Node joining: **SST** (State Snapshot Transfer, full copy via XtraBackup) or **IST** (incremental, from the gcache).
- Large transactions are problematic (writeset size limits). DDL uses TOI (Total Order Isolation: blocks the cluster) or RSU (rolling).
- Common practice: write to a single node (via ProxySQL/HAProxy) anyway, to avoid certification conflicts, and use the others as hot standbys plus readers.

---

## 11. Orchestrator, ProxySQL, Failover {#failover}

Classic async/semi-sync HA stack (GitHub, Booking.com, many others):
- **Orchestrator** (GitHub open source; now maintained by Percona; VTOrc is its Vitess descendant): discovers the topology by crawling replicas, detects failures *holistically* (it asks replicas whether they can also see the source, which reduces false positives), promotes the best candidate (most up-to-date, by promotion rules), re-points the other replicas using GTIDs, and runs hooks to update routing (Consul KV, DNS, ProxySQL).
- **ProxySQL**: SQL-aware proxy with connection multiplexing, read/write splitting by query rules, query routing by user/schema/digest, caching, mirroring, firewalling, and failover awareness via `read_only` monitoring (hostgroups switch automatically when a node's `read_only` flips) or native Group Replication / Galera / Aurora support.
- **MHA** (legacy), **MySQL Router** (for InnoDB Cluster), **HAProxy** + health checks, keepalived VIPs.

Failover with async replication **can lose transactions**. With semi-sync AFTER_SYNC and a sane timeout, data loss is effectively bounded to zero as long as at least one replica acked.

---

## 12. Vitess {#vitess}

Vitess (originated at YouTube, CNCF graduated; the basis of PlanetScale) is a sharding middleware for MySQL:
- **VTGate**: stateless proxy speaking the MySQL protocol. It parses queries, routes them to shards, and does scatter-gather for cross-shard queries.
- **VTTablet**: a sidecar per MySQL instance (query rules, connection pooling, consolidation of identical queries, health).
- **Topology service** (etcd/ZooKeeper/Consul): keyspace and shard metadata.
- **VSchema**: defines how tables are sharded via **vindexes** (e.g., hash on `customer_id`) and lookup vindexes for secondary keys.
- **Resharding online** (split/merge shards via VReplication), **MoveTables**, **online schema changes**, **VTOrc** for failover.
- Users: YouTube, Slack, GitHub, Shopify, Square/Cash App, HubSpot.

Alternatives for scaling MySQL horizontally: ShardingSphere (Apache, Java), ProxySQL sharding rules, application-level sharding, TiDB (MySQL-compatible distributed SQL), PlanetScale.

---

## 13. Managed Options {#managed}

| Service | Notes |
|---|---|
| Amazon RDS for MySQL | Multi-AZ (sync block replication) or Multi-AZ DB cluster (semi-sync, 2 readable standbys), read replicas, automated backups/PITR, Blue/Green deployments |
| **Amazon Aurora MySQL** | Storage replicated 6 ways across 3 AZs; replicas share storage (lag typically ~10–20 ms); fast failover; Global Database; backtrack (rewind in time) |
| Google Cloud SQL for MySQL | HA with regional persistent disks; Enterprise Plus near-zero downtime maintenance |
| Azure Database for MySQL – Flexible Server | Zone-redundant HA |
| PlanetScale | Vitess-based, branching, non-blocking schema changes |
| TiDB Cloud | MySQL-compatible distributed SQL |
| MySQL HeatWave (Oracle) | In-memory analytics/ML/vector engine attached to MySQL |

---

## 14. Common Problems {#problems}

| Symptom | Cause | Fix |
|---|---|---|
| `Last_SQL_Error: Duplicate entry` (1062) | Writes on replica, or a non-deterministic statement | Find the root cause; `super_read_only`; resync. Skipping transactions (inject an empty GTID transaction) is a last resort that hides divergence |
| `Could not find first log file name in binary log index file` / `Cannot replicate because the source purged required binary logs` | Replica down longer than binlog retention | Re-clone; raise retention |
| Errant GTIDs | Direct writes on a replica | Inject empty transactions on the source for those GTIDs, or rebuild |
| Data drift between source and replica | Statement-based non-determinism, replica writes, `sql_log_bin=0` misuse | **pt-table-checksum** + **pt-table-sync** to detect and fix |
| Lag spikes | Big transactions, DDL, missing PKs | See §6 |
| `server_uuid` duplicate | VM cloned with `auto.cnf` | Delete `auto.cnf` before first start |

---

## 15. Interview Questions {#qa}

1. Explain MySQL's async replication threads (dump, receiver, applier).
2. Statement vs row-based binlog: why is ROW the default?
3. What are GTIDs and what problems do they solve? What's an errant transaction?
4. Semi-sync AFTER_SYNC vs AFTER_COMMIT? What happens when the semi-sync timeout is hit?
5. Why does replication lag happen, and how does WRITESET-based parallel replication help?
6. Group Replication vs Galera vs async + Orchestrator: trade-offs?
7. How does Vitess shard MySQL? What's a vindex?
8. How do you detect data drift between source and replicas?
9. Design MySQL HA for a payments service with RPO ≈ 0.
