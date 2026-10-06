# MySQL Backup, Recovery, Online Schema Changes, Security and Operations

## Table of Contents
1. [Backup Methods Compared](#methods)
2. [mysqldump Done Right](#mysqldump)
3. [mydumper / MySQL Shell Dump Utilities](#mydumper)
4. [Percona XtraBackup (physical, hot)](#xtrabackup)
5. [Clone Plugin](#clone)
6. [Point-in-Time Recovery with Binlogs](#pitr)
7. [Online Schema Changes (Instant/Inplace/Copy, gh-ost, pt-osc)](#osc)
8. [Users, Roles, Authentication, TLS](#security)
9. [Upgrades](#upgrades)
10. [Day-2 Operations Cheat Sheet](#cheatsheet)
11. [Interview Questions](#qa)

---

## 1. Backup Methods {#methods}

| Method | Type | Hot? | Speed (large DB) | PITR base | Notes |
|---|---|---|---|---|---|
| `mysqldump` | Logical | Yes (`--single-transaction`, InnoDB only) | Slow dump, very slow restore | Yes (with `--source-data`) | Single-threaded; portable |
| **mydumper/myloader** | Logical | Yes | Parallel; much faster | Yes | Per-table files, compression |
| **MySQL Shell `util.dumpInstance()` / `loadDump()`** | Logical | Yes | Parallel, chunked, zstd | Yes | Can load directly to/from OCI/S3-compatible storage; used for HeatWave/migrations |
| **Percona XtraBackup** | Physical | Yes (copies files + redo) | Fast; restore = copy back | Yes | Incremental; the standard for self-managed |
| MySQL Enterprise Backup | Physical | Yes | Fast | Yes | Commercial |
| **Clone plugin** | Physical | Yes | Fast (network) | Provisioning | Great for building replicas |
| Filesystem/LVM/EBS snapshots | Physical | With `LOCK INSTANCE FOR BACKUP` / `FLUSH TABLES WITH READ LOCK` briefly, or crash-consistent | Very fast | Yes | Must coordinate binlog position |
| Delayed replica | Live copy | — | Instant "undo" for recent mistakes | — | Not a backup substitute |

Strategy commonly used: nightly XtraBackup full (or weekly full + daily incremental) to object storage, continuous binlog streaming (`mysqlbinlog --read-from-remote-server --raw --stop-never` or binlog-to-S3 tools), logical dumps for portability and partial restores, plus **automated restore tests**.

---

## 2. mysqldump Done Right {#mysqldump}

```bash
mysqldump --single-transaction --quick --routines --triggers --events \
  --source-data=2 --set-gtid-purged=ON --hex-blob --default-character-set=utf8mb4 \
  --databases app | zstd > app_$(date +%F).sql.zst
```
- `--single-transaction`: dumps inside a REPEATABLE READ snapshot (consistent for InnoDB without locking). Non-InnoDB tables aren't consistent.
- `--quick`: streams rows instead of buffering whole tables in memory.
- `--source-data=2` (formerly `--master-data`): writes the binlog coordinates as a comment, which you need for PITR or seeding a replica. It briefly takes `FLUSH TABLES WITH READ LOCK`, which can stall if long queries are running.
- `--set-gtid-purged`: GTID info for replica seeding.
- DDL during the dump can break consistency or fail it (DDL invalidates the snapshot for altered tables → "Table definition has changed" errors).
- Restore: `zstd -d < app.sql.zst | mysql`. Speed it up by disabling `unique_checks`/`foreign_key_checks` in the session, using larger `innodb_buffer_pool_size`, and temporarily relaxing durability (`innodb_flush_log_at_trx_commit=2`, or even disable redo with `ALTER INSTANCE DISABLE INNODB REDO_LOG` on a throwaway instance).

---

## 3. mydumper and MySQL Shell {#mydumper}

```bash
mydumper -h db -u backup --threads 8 --compress --trx-tables --outputdir /backup/app_2025_06_01 --database app
myloader -h newdb --threads 8 --directory /backup/app_2025_06_01 --overwrite-tables
```

```js
// mysqlsh
util.dumpInstance("/backup/full", {threads: 8, compression: "zstd"})
util.dumpSchemas(["app"], "s3://bucket/app", {s3BucketName: "bucket"})
util.loadDump("/backup/full", {threads: 16, deferTableIndexes: "all"})   // build secondary indexes after data load
```

---

## 4. Percona XtraBackup {#xtrabackup}

How it works: it copies InnoDB data files while the server runs, simultaneously **tails the redo log** to capture changes made during the copy, then briefly locks (backup lock: `LOCK INSTANCE FOR BACKUP`, which is non-blocking for DML in 8.0) to grab non-InnoDB files and the binlog position. **Prepare** applies the captured redo so the files become consistent (like crash recovery).

```bash
# Full backup
xtrabackup --backup --target-dir=/backup/full --user=backup --password=... --parallel=8 --compress
# Incremental (only pages with LSN > last backup's)
xtrabackup --backup --target-dir=/backup/inc1 --incremental-basedir=/backup/full
# Stream to S3
xtrabackup --backup --stream=xbstream --parallel=8 | xbcloud put --storage=s3 --s3-bucket=backups full_2025_06_01

# Restore
xtrabackup --decompress --target-dir=/backup/full
xtrabackup --prepare --apply-log-only --target-dir=/backup/full
xtrabackup --prepare --apply-log-only --target-dir=/backup/full --incremental-dir=/backup/inc1
xtrabackup --prepare --target-dir=/backup/full          # final prepare (rollback uncommitted)
systemctl stop mysql && rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/full && chown -R mysql:mysql /var/lib/mysql
cat /backup/full/xtrabackup_binlog_info                  # binlog file/pos + GTID for PITR or replication
```
XtraBackup versions track MySQL versions (8.0.x ↔ XtraBackup 8.0.x, 8.4 ↔ 8.4). Use the matching one.

---

## 5. Clone Plugin {#clone}

MySQL 8.0.17+ can **clone an instance over the network** (or locally) natively: page-level copy plus redo, consistent, then the recipient restarts with the data. It's the easiest way to provision a replica or a Group Replication member (InnoDB Cluster uses it automatically). It's not a backup tool (there's no storage format or incrementals).

---

## 6. PITR with Binlogs {#pitr}

Scenario: a bad `DELETE` at 2025-06-01 14:03:17.
```bash
# 1. Restore the latest full backup (taken at 02:00) onto a NEW instance; note its binlog position / GTID set
cat xtrabackup_binlog_info     # binlog.000210  154  uuid:1-998877

# 2. Find the bad event
mysqlbinlog --base64-output=DECODE-ROWS -vv --start-datetime="2025-06-01 14:00:00" binlog.000231 | less
#   → locate "DELETE FROM `app`.`orders`" at position 88123 (GTID uuid:1012345)

# 3. Replay from backup position up to just before the bad transaction
mysqlbinlog --start-position=154 binlog.000210 binlog.000211 ... binlog.000231 --stop-position=88000 | mysql
#   With GTIDs, exclude just the bad transaction and continue past it:
mysqlbinlog --exclude-gtids='uuid:1012345' binlog.000210 ... binlog.000240 | mysql

# 4. Verify, then either switch traffic or copy the affected rows back into production
```
Faster replay alternative: make the restored instance a replica of a binlog server, and use `START REPLICA UNTIL SQL_BEFORE_GTIDS = 'uuid:1012345'` (parallel applier).

Aurora: **Backtrack** rewinds the cluster in place to a point in time within a window (seconds to minutes, no restore needed).

---

## 7. Online Schema Changes {#osc}

MySQL's own online DDL has three algorithms:
| Algorithm | What | Examples (8.0/8.4) |
|---|---|---|
| **INSTANT** | Metadata only, no data touched | ADD COLUMN (any position 8.0.29+), DROP COLUMN (8.0.29+), RENAME COLUMN, set/drop default, modify ENUM/SET by appending values, change index visibility, rename table |
| **INPLACE** | Rebuild or modify in the engine without copying through the server layer; concurrent DML allowed (`LOCK=NONE`) mostly | ADD INDEX, DROP INDEX, RENAME INDEX, add FK (with `foreign_key_checks=0`), change row format, OPTIMIZE TABLE, extend VARCHAR within the same length-byte tier |
| **COPY** | Creates a new table and copies rows; **blocks writes** | Change column data type (INT→BIGINT), change charset, drop PK & add another… |

```sql
ALTER TABLE orders ADD COLUMN note TEXT, ALGORITHM=INSTANT;           -- errors if not possible (good: no surprises)
ALTER TABLE orders ADD INDEX idx_status (status), ALGORITHM=INPLACE, LOCK=NONE;
```
Always specify `ALGORITHM`/`LOCK` so MySQL errors out instead of silently falling back to a blocking COPY.

Caveats of native online DDL:
- Concurrent DML changes during INPLACE are buffered in an online log (`innodb_online_alter_log_max_size`). If it overflows, the DDL fails at the end.
- **Replication**: the replica starts the DDL only after the source finishes, and applies it single-threaded, so lag equals the DDL duration.
- Not throttleable or pausable.
- INSTANT ADD COLUMN has a limit on the number of instant row versions (64) before a rebuild is needed.

**gh-ost** (GitHub), for big tables in production:
```bash
gh-ost --host=replica1 --database=app --table=orders \
  --alter="MODIFY id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT" \
  --chunk-size=1000 --max-lag-millis=1500 --throttle-control-replicas=replica1,replica2 \
  --cut-over=default --exact-rowcount --concurrent-rowcount --postpone-cut-over-flag-file=/tmp/ghost.postpone \
  --execute
```
- Creates `_orders_gho`, copies rows in chunks, applies ongoing changes by **reading the binlog** (no triggers), and throttles on replica lag or load. It's interactive via a socket (pause, change chunk size), and the cut-over is an atomic rename under a brief lock.
- Requires ROW binlog with FULL row image. It has caveats with FKs (unsupported by default) and with triggers on the table.

**pt-online-schema-change** (Percona): trigger-based (INSERT/UPDATE/DELETE triggers copy changes to the new table). It supports FKs (with options), but triggers add write overhead and contention.

Also: Vitess/PlanetScale online DDL (revertible), Spirit (Cash App's gh-ost successor).

---

## 8. Users, Roles, Authentication, TLS {#security}

```sql
CREATE USER 'orders_svc'@'10.0.%' IDENTIFIED BY '…' REQUIRE SSL
  PASSWORD EXPIRE INTERVAL 90 DAY FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;
CREATE ROLE app_rw, app_ro;
GRANT SELECT, INSERT, UPDATE, DELETE ON app.* TO app_rw;
GRANT SELECT ON app.* TO app_ro;
GRANT app_rw TO 'orders_svc'@'10.0.%';
SET DEFAULT ROLE app_rw TO 'orders_svc'@'10.0.%';     -- roles aren't active by default!
SHOW GRANTS FOR 'orders_svc'@'10.0.%' USING app_rw;
```
- A user is `'name'@'host'`. `'app'@'%'` and `'app'@'localhost'` are **different accounts**. Host matching precedence surprises people (anonymous users, most-specific host wins).
- Authentication plugins: **`caching_sha2_password`** (default since 8.0; old clients/drivers may fail with "authentication plugin cannot be loaded", so upgrade the drivers), `mysql_native_password` (SHA-1-based; **disabled by default in 8.4, removed in 9.0**), `auth_socket` (Unix socket), LDAP/Kerberos (Enterprise/Percona), AWS IAM auth (RDS).
- TLS: `require_secure_transport=ON` server-wide; clients use `ssl-mode=VERIFY_IDENTITY`.
- Least privilege: no `SUPER`/`ALL PRIVILEGES` for apps. Use dynamic privileges in 8.0 (`BINLOG_ADMIN`, `CONNECTION_ADMIN`, `REPLICATION_SLAVE_ADMIN`, `SYSTEM_VARIABLES_ADMIN`, …) for ops accounts. No `FILE` privilege (it allows `LOAD DATA INFILE`/`SELECT INTO OUTFILE` on the server filesystem); `secure_file_priv` restricts paths.
- `local_infile=OFF` unless needed (malicious server/client file-read attacks).
- `super_read_only=ON` on replicas.
- Encryption at rest: InnoDB tablespace encryption (`ENCRYPTION='Y'`) with a keyring component (file keyring in Community; KMS/Vault keyrings in Enterprise/Percona), plus redo/undo/binlog encryption (`binlog_encryption=ON`).
- Audit: Enterprise Audit, Percona Audit Log plugin, MariaDB server_audit.
- Password validation component (`validate_password`).

---

## 9. Upgrades {#upgrades}

- **In-place**: stop, swap binaries, start. 8.0+ upgrades the data dictionary and system tables automatically at startup (no separate `mysql_upgrade` since 8.0.16). Only upgrades between consecutive LTS lines (8.0 → 8.4 → next LTS) are supported. Downgrades are generally **not** supported (restore from backup).
- Run **MySQL Shell's upgrade checker** first: `util.checkForServerUpgrade()` catches reserved word conflicts, removed features, auth plugin issues, deprecated syntax, and changed defaults.
- **Replica-first / blue-green**: upgrade a replica, let it replicate from the old source (newer replica, older source is supported), test the app against it, then fail over to it. RDS Blue/Green deployments automate this.
- Test query plans: `pt-upgrade` replays queries against both versions and compares results and timing.
- Watch for changed defaults (e.g., 8.0's `caching_sha2_password` and utf8mb4 collation `0900_ai_ci`; 8.4's InnoDB default changes).

---

## 10. Day-2 Cheat Sheet {#cheatsheet}

```sql
-- Sessions
SHOW FULL PROCESSLIST;
SELECT * FROM sys.processlist WHERE command <> 'Sleep'\G
KILL QUERY 1234;  KILL 1234;

-- Sizes
SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024/1024, 2) AS gb
FROM information_schema.tables GROUP BY table_schema ORDER BY gb DESC;
SELECT table_name, ROUND(data_length/1024/1024) data_mb, ROUND(index_length/1024/1024) idx_mb,
       ROUND(data_free/1024/1024) free_mb, table_rows
FROM information_schema.tables WHERE table_schema = 'app' ORDER BY data_length DESC LIMIT 20;

-- Unused & redundant indexes
SELECT * FROM sys.schema_unused_indexes;
SELECT * FROM sys.schema_redundant_indexes;

-- Top statements
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;

-- Tables without primary keys
SELECT t.table_schema, t.table_name FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints c
  ON c.table_schema = t.table_schema AND c.table_name = t.table_name AND c.constraint_type = 'PRIMARY KEY'
WHERE t.table_type = 'BASE TABLE' AND c.constraint_name IS NULL
  AND t.table_schema NOT IN ('mysql','sys','performance_schema','information_schema');

-- Auto-increment exhaustion risk
SELECT table_schema, table_name, auto_increment FROM information_schema.tables
WHERE auto_increment IS NOT NULL ORDER BY auto_increment DESC LIMIT 10;   -- compare against column type max (INT signed 2,147,483,647)

-- Replication
SHOW REPLICA STATUS\G
SHOW BINARY LOGS;   PURGE BINARY LOGS BEFORE NOW() - INTERVAL 7 DAY;

-- Variables
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SET PERSIST innodb_io_capacity = 4000;          -- 8.0: persists to mysqld-auto.cnf
SELECT * FROM performance_schema.variables_info WHERE variable_source <> 'COMPILED';
```
Percona Toolkit essentials: `pt-query-digest`, `pt-online-schema-change`, `pt-table-checksum`/`pt-table-sync`, `pt-heartbeat`, `pt-kill` (kill queries matching rules), `pt-stalk`, `pt-summary`/`pt-mysql-summary`, `pt-archiver` (chunked archive/purge of old rows), `pt-duplicate-key-checker`, `pt-deadlock-logger`.

---

## 11. Interview Questions {#qa}

1. How do you take a consistent backup of a live MySQL database without blocking writes?
2. How does XtraBackup produce a consistent backup while the server is being written to?
3. Walk through point-in-time recovery to just before an accidental DELETE.
4. INSTANT vs INPLACE vs COPY DDL? How would you change an INT PK to BIGINT on a 2 TB table?
5. How does gh-ost differ from pt-online-schema-change?
6. Why might upgrading from 5.7/8.0 break application logins?
7. How do you safely upgrade MySQL major versions?
