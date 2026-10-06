# MySQL Data Types, Character Sets, SQL Modes, and Gotchas

> MySQL historically chose leniency over strictness: it silently truncated data, accepted invalid dates, and its "utf8" wasn't real UTF-8. Modern defaults are much better, but legacy apps, old configs, and managed-service parameter groups still carry these traps.

## Table of Contents
1. [SQL Mode: Strict vs Permissive](#sqlmode)
2. [Character Sets and Collations](#charset)
3. [Numeric Types](#numeric)
4. [String and Binary Types](#strings)
5. [Date and Time Types](#time)
6. [JSON Type](#json)
7. [ENUM and SET](#enum)
8. [Generated Columns](#generated)
9. [Gotcha Catalog](#gotchas)
10. [Interview Questions](#qa)

---

## 1. SQL Mode {#sqlmode}

```sql
SELECT @@GLOBAL.sql_mode;
-- 8.0 default: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```
| Mode | Effect |
|---|---|
| `STRICT_TRANS_TABLES` / `STRICT_ALL_TABLES` | Invalid or out-of-range values → **error** instead of silent truncation/adjustment with a warning |
| `ONLY_FULL_GROUP_BY` | Reject selecting non-aggregated columns not functionally dependent on GROUP BY |
| `NO_ZERO_DATE`, `NO_ZERO_IN_DATE` | Reject `'0000-00-00'` and `'2025-00-10'` |
| `ERROR_FOR_DIVISION_BY_ZERO` | Error instead of NULL on /0 in writes |
| `NO_ENGINE_SUBSTITUTION` | Error if the requested engine is unavailable (rather than silently using the default) |
| `ANSI_QUOTES` | `"` quotes identifiers |
| `PIPES_AS_CONCAT` | `||` concatenates |
| `NO_BACKSLASH_ESCAPES` | `\` is literal in strings |

Without strict mode:
```sql
CREATE TABLE t (name VARCHAR(5), age TINYINT UNSIGNED);
INSERT INTO t VALUES ('Aadarsh', -5);
-- Non-strict: inserts ('Aadar', 0) with warnings nobody reads. Silent data corruption.
-- Strict: ERROR 1406 Data too long for column 'name'
```
**Always run strict.** Check legacy apps that set `sql_mode=''` in their connection init.

---

## 2. Character Sets and Collations {#charset}

- **`utf8` in MySQL = `utf8mb3`**: max 3 bytes per character, so no emoji (😀 is 4 bytes) and no some rare CJK. Inserting emoji fails (strict) or truncates the string at the emoji (non-strict). `utf8` is deprecated as an alias and may become utf8mb4 in the future.
- **Use `utf8mb4` everywhere**: server, database, tables, columns, **and the connection** (`SET NAMES utf8mb4`, or driver `characterEncoding`/`charset=utf8mb4`). A mismatched connection charset produces **mojibake** (double-encoded garbage like `Ã©`).
- 8.0 default: `utf8mb4` with collation **`utf8mb4_0900_ai_ci`** (Unicode 9.0 UCA, **a**ccent-**i**nsensitive, **c**ase-**i**nsensitive).
  - `'a' = 'A'` → true; `'e' = 'é'` → true. UNIQUE on username treats `jose` and `José` as duplicates (maybe desired, maybe not).
  - `_as_cs` collations: accent and case sensitive. `_bin` / `utf8mb4_0900_bin`: binary comparison.
  - Trailing spaces: `_0900_` collations are NO PAD (`'a' <> 'a '`), while older `utf8mb4_general_ci`/`unicode_ci` are PAD SPACE (`'a' = 'a '`!).
- Mixing collations in comparisons/joins → `Illegal mix of collations` errors or index-unusable implicit conversions. Keep them uniform; migrate legacy `latin1`/`utf8mb3` tables carefully (`ALTER TABLE … CONVERT TO CHARACTER SET utf8mb4`, which is a COPY rebuild).
- Index length: utf8mb4 counts 4 bytes per character in key length limits (3072 bytes → `VARCHAR(768)`).
- Case-sensitive comparisons per query: `WHERE name = 'X' COLLATE utf8mb4_0900_as_cs` (this may prevent index use).

---

## 3. Numeric Types {#numeric}

| Type | Bytes | Signed range |
|---|---|---|
| TINYINT | 1 | −128…127 (UNSIGNED 0…255) |
| SMALLINT | 2 | ±32K |
| MEDIUMINT | 3 | ±8.3M |
| INT | 4 | ±2.1B (UNSIGNED 4.29B) |
| BIGINT | 8 | ±9.2×10¹⁸ |
| DECIMAL(p,s) | variable | exact; p ≤ 65 |
| FLOAT / DOUBLE | 4 / 8 | approximate |

Gotchas:
- **`INT(11)` display width means nothing about storage or range** (it's deprecated since 8.0.17; `ZEROFILL` too). `INT(1)` still holds 2 billion.
- `TINYINT(1)` is how `BOOL`/`BOOLEAN` is implemented. Drivers map `TINYINT(1)` to boolean, so storing 2 in it surprises ORMs.
- UNSIGNED arithmetic: `SELECT unsigned_col - 1` where the value is 0 gives **an error** (BIGINT UNSIGNED value out of range) in strict mode. Subtraction of unsigned values needs `CAST(... AS SIGNED)`. Many teams avoid UNSIGNED except for IDs.
- `FLOAT` columns compared with literals: `WHERE price = 19.99` may not match the stored float. Use DECIMAL.
- AUTO_INCREMENT overflow: an INT PK reaching 2,147,483,647 → `Duplicate entry '2147483647' for key 'PRIMARY'` errors. Monitor. The migration to BIGINT is a COPY ALTER (gh-ost).
- Integer division: `SELECT 5/2` gives 2.5000 (DECIMAL). `DIV` gives integer division.

---

## 4. String and Binary Types {#strings}

| Type | Max | Notes |
|---|---|---|
| CHAR(n) | 255 chars | Fixed-width, right-padded; trailing spaces stripped on read |
| VARCHAR(n) | 65,535 bytes **per row total** | 1–2 byte length prefix; n in characters |
| TINYTEXT/TEXT/MEDIUMTEXT/LONGTEXT | 255 B / 64 KB / 16 MB / 4 GB | Stored off-page when large; can't have DEFAULT before 8.0.13 (expression defaults); index requires prefix length; internal temp tables with TEXT historically went to disk |
| BINARY/VARBINARY | | Bytes, binary comparison (UUIDs as BINARY(16)) |
| BLOB variants | as TEXT | |

- Row size limit: 65,535 bytes for all columns combined (excluding off-page TEXT/BLOB pointers). Several `VARCHAR(5000)` utf8mb4 columns can hit it (`Row size too large`).
- **Choose VARCHAR lengths realistically**: the declared max affects memory for in-memory temp tables and sorts (8.0 TempTable handles variable length better, but sort buffers still use max lengths in some paths).
- `TEXT` vs `VARCHAR(10000)`: similar on disk with DYNAMIC rows. `VARCHAR` can have defaults and full (non-prefix) indexes up to the key limit.

---

## 5. Date and Time {#time}

| Type | Range | Time zone behavior | Bytes |
|---|---|---|---|
| DATE | 1000-01-01 … 9999-12-31 | none | 3 |
| DATETIME(fsp) | 1000 … 9999 | **None**: stores what you give it | 5 (+fsp) |
| TIMESTAMP(fsp) | 1970-01-01 00:00:01 UTC … **2038-01-19 03:14:07 UTC** | Converted from the session `time_zone` to UTC on write, back on read | 4 (+fsp) |
| TIME | ±838:59:59 | | 3 |
| YEAR | 1901–2155 | | 1 |

- **Year 2038**: TIMESTAMP columns break for dates after 2038-01-19. Contracts, subscriptions, and loans that expire after 2038 already hit this. Prefer `DATETIME(6)` storing UTC by convention.
- Session `time_zone` matters for TIMESTAMP and for `NOW()`. Set the server `default_time_zone='+00:00'` and have the app send UTC. Load tz tables (`mysql_tzinfo_to_sql`) if you need named zones (`CONVERT_TZ(ts, 'UTC', 'Asia/Kolkata')`; it returns NULL if the tables aren't loaded!).
- `NOW()` = statement start time (constant within a statement, also in triggers/functions). `SYSDATE()` = actual time at execution (non-deterministic, unsafe for statement replication).
- Fractional seconds: `DATETIME(6)` for microseconds. Without it, values are **rounded** (not truncated) by default (`TIME_TRUNCATE_FRACTIONAL` mode changes this), which can produce times in the future or next second.
- `ON UPDATE CURRENT_TIMESTAMP` auto-updates the column on any row change. Before 8.0 (`explicit_defaults_for_timestamp=OFF`), the first TIMESTAMP column in a table **implicitly** got `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP`, so "created_at" silently changed on every update. Legacy schemas still have this bug.

---

## 6. JSON Type {#json}

- Native binary JSON (5.7+), validated on insert, and fast path access. Max size limited by `max_allowed_packet`.
- Functions: `JSON_EXTRACT(doc, '$.a.b')` / `doc->'$.a'` (JSON) / `doc->>'$.a'` (unquoted text), `JSON_SET`, `JSON_INSERT`, `JSON_REPLACE`, `JSON_REMOVE`, `JSON_ARRAY_APPEND`, `JSON_CONTAINS`, `JSON_OVERLAPS`, `MEMBER OF`, `JSON_TABLE` (8.0: JSON → relational rows), `JSON_ARRAYAGG`, `JSON_OBJECTAGG`, `JSON_SCHEMA_VALID` (8.0.17).
- **Indexing**: no direct index on JSON. Use generated columns + index, functional indexes on `CAST(doc->>'$.x' AS …)`, or multi-valued indexes for arrays.
- **Partial in-place updates** (8.0): `JSON_SET`/`JSON_REPLACE`/`JSON_REMOVE` can update in place and log only the diff to the binlog (`binlog_row_value_options=PARTIAL_JSON`), which reduces write amplification.
- Comparison and ordering of JSON values follow JSON type rules. Extract and cast for numeric comparisons.

---

## 7. ENUM and SET {#enum}

```sql
status ENUM('pending','paid','shipped') NOT NULL DEFAULT 'pending'
```
- Stored as 1–2 byte index. Compact.
- `ORDER BY status` sorts by **declaration index**, not alphabetically.
- Inserting an invalid value: error in strict mode; in non-strict mode it stores `''` (index 0)!
- `WHERE status = 1` compares with the index number. Confusing.
- Adding a value at the **end** is INSTANT. Inserting in the middle or removing a value requires a rebuild.
- `SET('a','b','c')` stores a bitmap of multiple values, which is rarely a good idea (it's not 1NF). Use a junction table.

---

## 8. Generated Columns {#generated}

```sql
ALTER TABLE orders
  ADD COLUMN total DECIMAL(12,2) AS (qty * unit_price) STORED,
  ADD COLUMN customer_email VARCHAR(255) AS (payload->>'$.customer.email') VIRTUAL,
  ADD INDEX idx_customer_email (customer_email);
```
- `VIRTUAL` (default): computed on read, not stored, but **can be indexed** (the index stores the values). Adding one is INSTANT.
- `STORED`: computed on write and stored. Adding one requires a rebuild.
- Expressions must be deterministic.

---

## 9. Gotcha Catalog {#gotchas}

| # | Gotcha | Avoid it by |
|---|---|---|
| 1 | `utf8` isn't full UTF-8 | `utf8mb4` everywhere incl. connection |
| 2 | Case- and accent-insensitive default collation affects uniqueness & equality | Choose collation deliberately per column |
| 3 | Silent truncation in non-strict mode | Strict `sql_mode` |
| 4 | Implicit string↔number conversion: `WHERE varchar_col = 123` is a full scan; `'123abc' = 123` is TRUE | Compare like types |
| 5 | `TIMESTAMP` 2038 limit | `DATETIME(6)` in UTC |
| 6 | DDL causes implicit commit, so no transactional migrations | One DDL per migration; idempotent migrations |
| 7 | `REPLACE INTO` = delete + insert (new auto-inc ID, cascades, triggers) | `INSERT … ON DUPLICATE KEY UPDATE` |
| 8 | `ON DUPLICATE KEY UPDATE` with multiple unique keys picks an unpredictable row | Single unique key for upserts |
| 9 | `UPDATE … LIMIT` / `DELETE … LIMIT` without ORDER BY is nondeterministic (unsafe for statement replication) | ORDER BY PK |
| 10 | `GROUP_CONCAT` truncated at 1024 bytes silently (warning) | Raise `group_concat_max_len` per session |
| 11 | `lower_case_table_names` differs across OSes | Lowercase names only; same setting everywhere |
| 12 | `'0000-00-00'` zero dates in legacy data break drivers (`zeroDateTimeBehavior` in JDBC) | Clean data; strict modes |
| 13 | `NULL` sorts first in ASC | `ORDER BY col IS NULL, col` |
| 14 | `CHECK` constraints ignored before 8.0.16 | Verify version |
| 15 | MyISAM tables lurking (no transactions; table locks; crash-unsafe) | Convert to InnoDB |
| 16 | `innodb_lock_wait_timeout` rolls back only the statement | Roll back the whole transaction in app |
| 17 | `SELECT … FOR UPDATE` on missing rows takes gap locks → deadlocks | Upserts / RC isolation |
| 18 | Auto-increment gaps & non-reuse | Never rely on contiguous IDs |
| 19 | `LIMIT` with huge `OFFSET` | Keyset pagination |
| 20 | Large `IN()` exceeding `range_optimizer_max_mem_size` silently falls back to full scan | Batch, temp table join |
| 21 | Replication filters with STATEMENT format depend on `USE db` | ROW format; avoid filters |
| 22 | `LOAD DATA LOCAL INFILE` security risk | `local_infile=OFF` unless needed |
| 23 | Connection charset/collation (`character_set_client/connection/results`) mismatch → mojibake | Driver config `utf8mb4` |
| 24 | `wait_timeout` shorter than pool idle → "MySQL server has gone away" | Pool max lifetime < wait_timeout |
| 25 | Prepared statement leaks (`Prepared_stmt_count` hitting `max_prepared_stmt_count`) | Close statements; driver caching |

---

## 10. Interview Questions {#qa}

1. Why is `utf8` in MySQL a problem? What should you use and where must you set it?
2. What does `utf8mb4_0900_ai_ci` mean and how can it affect a UNIQUE constraint?
3. What happens when you insert a too-long string in strict vs non-strict mode?
4. DATETIME vs TIMESTAMP: differences and the 2038 problem?
5. What does `INT(11)` mean?
6. How do you index a field inside a JSON column in MySQL?
7. Why is `REPLACE INTO` dangerous?
