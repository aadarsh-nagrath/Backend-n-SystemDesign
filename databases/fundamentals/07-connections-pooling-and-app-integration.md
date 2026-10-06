# Connections, Pooling, and How Applications Should Talk to Databases

> Many production database incidents are not slow queries. They are connection storms, pool exhaustion, timeouts that are missing or wrong, retry amplification, or SQL injection. This note covers the layer between your code and the DB.

## Table of Contents
1. [What a Connection Costs](#cost)
2. [Connection Pooling (client-side)](#pool)
3. [Pool Sizing — the Math](#sizing)
4. [Server-Side Poolers: PgBouncer, ProxySQL, RDS Proxy](#proxies)
5. [Timeouts: Every Layer Needs One](#timeouts)
6. [Prepared Statements and the Wire Protocol](#prepared)
7. [SQL Injection — and Why Parameterization Fixes It](#sqli)
8. [ORMs vs Query Builders vs Raw SQL](#orm)
9. [Read/Write Splitting and Replica Lag](#rw)
10. [Serverless and Connection Limits](#serverless)
11. [Bulk Operations](#bulk)
12. [Failure Handling: Retries, Failover, Circuit Breakers](#failure)
13. [Observability for the DB Client](#obs)
14. [Interview Questions](#qa)

---

## 1. What a Connection Costs {#cost}

Opening a connection involves a TCP handshake, a TLS handshake (1–2 RTT), authentication (SCRAM-SHA-256 = several round trips plus deliberately expensive hashing), and session setup. That is **several milliseconds to tens of ms**.

Server-side:
- **PostgreSQL: one OS process per connection** (fork from postmaster). Each one uses ~2–10 MB of private memory, more with large `work_mem` sorts, plus catalog caches. Thousands of connections mean context switching, lock-manager and snapshot contention (`GetSnapshotData` scales with the connection count; PG 14 improved this a lot), and memory pressure. Practical healthy range: a few hundred at most.
- **MySQL: one thread per connection** (thread pool plugin in Enterprise/Percona/MariaDB). Cheaper than a process, but each thread still has per-connection buffers (`sort_buffer_size`, `join_buffer_size`, `read_buffer_size`…), and thousands of concurrently *active* threads thrash InnoDB.
- `max_connections`: PG default 100, MySQL default 151.

Key insight: **a DB server can only do useful work for roughly (cores × small factor) queries at a time.** More concurrent active queries than that only adds queuing and contention.

---

## 2. Client-Side Connection Pooling {#pool}

A pool keeps N open connections and lends them out per unit of work.
- Java: **HikariCP** (Spring Boot default). Node: `pg.Pool`, `mysql2` pool, Prisma's pool, Knex. Python: SQLAlchemy `QueuePool`, psycopg_pool, asyncpg pool. Go: `database/sql` has a built-in pool (`SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime`).

Settings that matter:
| Setting | Purpose | Guidance |
|---|---|---|
| max pool size | Concurrency cap | Small. See §3 |
| min idle | Warm connections | = max for steady services (avoid connection churn) |
| connection timeout / acquire timeout | Max wait to *get* a connection from the pool | Short (~1–5 s). Fail fast rather than pile up requests |
| max lifetime | Recycle connections periodically | Slightly less than any LB/proxy/DB idle cutoff (e.g., 30 min); helps rebalance after failover/DNS change |
| idle timeout | Close excess idle connections | |
| validation / keepalive | Detect dead connections | Use cheap checks (`isValid()`/protocol ping), not `SELECT 1` before every borrow |
| leak detection | Warn if a connection is held too long | Enable in dev/staging |

**Always return connections**: `try-with-resources`, `with` blocks, `defer rows.Close()`. In Go, forgetting `rows.Close()` leaks connections. In Node, forgetting `client.release()` after `pool.connect()` leaks one.

**Don't hold a connection during slow non-DB work** (HTTP calls, file uploads). Acquire late, release early.

---

## 3. Pool Sizing — the Math {#sizing}

HikariCP's well-known guidance (from PG's wiki):
```
connections ≈ (core_count × 2) + effective_spindle_count
```
For a 16-core DB with SSD, that's ~32–40 **active** connections total across all app instances. The formula is a starting point; measure.

**Little's Law**: concurrency = throughput × latency. If you need 2,000 queries/s and the average query takes 5 ms, you need 2,000 × 0.005 = **10 connections busy on average**. Add headroom for bursts, maybe 20–30.

The fleet multiplication trap:
```
50 app pods × pool size 20 = 1,000 connections → over max_connections, or a thrashing DB
```
Fixes: smaller per-pod pools, a server-side pooler (PgBouncer), or fewer, larger app instances.

Symptoms of an undersized pool are high "time waiting for a connection" while the DB sits idle. Symptoms of an oversized one are DB CPU saturation, lock contention, and rising latency on every query. **Counter-intuitive but true: shrinking the pool often makes things faster under load.**

---

## 4. Server-Side Poolers {#proxies}

### PgBouncer (PostgreSQL)
A lightweight single-threaded proxy. Thousands of client connections multiplex onto a few server connections.

| Mode | Server connection is assigned for | Breaks |
|---|---|---|
| **session** | The whole client session | Nothing, but little multiplexing gain |
| **transaction** (most common) | One transaction | Session state: `SET` (use `SET LOCAL`), session advisory locks, `LISTEN/NOTIFY`, temp tables, WITH HOLD cursors, and protocol-level prepared statements *before* PgBouncer 1.21 (now supported with `max_prepared_statements`) |
| **statement** | One statement | Multi-statement transactions |

Alternatives: **PgCat** and **Supavisor** (multi-threaded, sharding-aware), **Odyssey** (Yandex), cloud ones like **RDS Proxy**, plus built-in poolers in Neon and Supabase.

### MySQL
- **ProxySQL**: connection multiplexing, query routing (read/write split by regex rules), query caching, query rewriting, and failover awareness (works with Orchestrator / group replication).
- MySQL Router (InnoDB Cluster), MaxScale (MariaDB), RDS Proxy.
- MySQL's threads are cheaper, so many shops run without a proxy until a large scale.

### RDS Proxy / Cloud SQL Auth Proxy
Managed pooling plus IAM auth and faster failover handling. Watch for **pinning**: session state forces a client to stay attached to one DB connection, which kills multiplexing.

---

## 5. Timeouts at Every Layer {#timeouts}

With no timeouts, one slow dependency makes every request thread wait forever. The pool exhausts, health checks fail, and the outage cascades.

| Layer | Setting |
|---|---|
| Pool acquire | `connectionTimeout` (Hikari), `acquireTimeoutMillis` |
| TCP connect | driver `connectTimeout` |
| Socket read | `socketTimeout` (JDBC) — the last safety net |
| Statement | PG `statement_timeout` (per role/db/session/`SET LOCAL`); MySQL `max_execution_time` (SELECT only) or `/*+ MAX_EXECUTION_TIME(1000) */`; JDBC `setQueryTimeout` |
| Lock wait | PG `lock_timeout`; MySQL `innodb_lock_wait_timeout` (50 s default — way too high for OLTP) |
| Idle in transaction | PG `idle_in_transaction_session_timeout` |
| Transaction | PG 17 `transaction_timeout` |
| HTTP request | Server/load balancer timeouts |

**Timeout budgets must nest**: DB statement timeout < app request timeout < LB timeout < client timeout. If the LB times out at 30 s but the query runs for 60 s, the DB keeps working for a client that has already left.

Also set TCP keepalives so dead peers are detected (`tcp_keepalives_idle` in PG; driver options). Cloud NAT and load balancers silently drop idle flows after ~350 s (AWS NLB/NAT).

---

## 6. Prepared Statements and the Wire Protocol {#prepared}

PG extended query protocol: **Parse** (SQL with `$1` placeholders → named or unnamed statement) → **Bind** (parameters) → **Execute** → **Sync**. Parameters travel **separately from the SQL text**. This is what makes them injection-proof.

Benefits:
- Security (no string interpolation).
- Skip parse/plan on reuse (named statements; plan caching rules covered in the query planning note).
- Binary transfer of parameters.

Caveats:
- Named prepared statements are per-connection. Poolers in transaction mode need explicit support.
- MySQL: client-side "emulated" prepares (PHP PDO default, some drivers) interpolate safely on the client; server-side prepares (`useServerPrepStmts=true` in Connector/J) use `COM_STMT_PREPARE`. Watch `max_prepared_stmt_count` leaks.
- **Pipelining** (PG 14+ libpq, asyncpg, pgx batch) sends many queries without waiting for each response, which cuts round trips dramatically for batches.

---

## 7. SQL Injection {#sqli}

```python
# VULNERABLE
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")
# email = "' OR '1'='1' --"  → returns every user
# email = "'; DROP TABLE users; --" → stacked query (if driver allows multi-statements)

# SAFE
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

Parameterization cannot bind **identifiers or keywords**: table names, column names, `ORDER BY` direction. For dynamic sort columns, use an **allowlist** mapping:
```python
SORTS = {"newest": "created_at DESC", "price": "price ASC"}
order = SORTS.get(user_sort, "created_at DESC")
```
Or use identifier quoting helpers (`psycopg.sql.Identifier`, `pg-format %I`).

Other vectors:
- `LIKE` patterns: user-supplied `%` or `_` cause slow scans or unexpected matches. Escape them.
- ORM "raw" escape hatches: `Model.objects.raw(f"...")`, `sequelize.query(\`...${x}\`)`, Rails `where("name = '#{x}'")`.
- Second-order injection: stored data is later concatenated into SQL.
- Defense in depth: least-privilege DB users (the app user can't `DROP`), no multi-statement support, WAF as a minor extra layer.

---

## 8. ORMs vs Query Builders vs Raw SQL {#orm}

Covered extensively in [`database-concepts/orms.md`](../../database-concepts/orms.md). Summary of the trade-offs:

| | ORM (Hibernate/JPA, Django ORM, ActiveRecord, Prisma, SQLAlchemy ORM, TypeORM) | Query builder (jOOQ, Knex, Kysely, SQLAlchemy Core, Drizzle, squirrel) | Raw SQL (+ codegen: sqlc, PgTyped) |
|---|---|---|---|
| Productivity for CRUD | Highest | Medium | Lowest |
| Control over SQL | Low | High | Total |
| Hidden performance traps | N+1, lazy loading, over-fetching, chatty flushes | Few | None hidden |
| Type safety | Varies | Good (jOOQ, Kysely) | sqlc generates typed code from SQL |
| Portability | High | Medium | Low |

Practical approach: ORM for simple CRUD, and hand-written SQL for complex reporting or hot paths. Always **log or inspect the generated SQL** in development.

ORM pitfalls to know:
- **Lazy loading inside loops** causes N+1.
- **Open Session in View** (Spring's `spring.jpa.open-in-view=true` default) keeps the connection during view rendering and permits lazy loads anywhere. Disable it.
- **Implicit transactions and flush ordering**: JPA flushes at commit or before queries, so writes happen later than the code suggests.
- **Save the whole entity** → `UPDATE` all columns → lost updates on concurrent edits of different fields. Use dynamic update or optimistic locking.
- **Huge identity maps / sessions** in batch jobs → memory blow-ups. Clear the session periodically.

---

## 9. Read/Write Splitting and Replica Lag {#rw}

Sending reads to replicas scales reads, but replicas are **asynchronously behind** (milliseconds normally, minutes during heavy writes or long DDL).

Bugs this causes:
- **Read-your-own-writes violation**: a user updates their profile, the page reloads from a replica, and they see the old data.
- **Monotonic reads violation**: successive requests hit different replicas and data goes back in time.

Mitigations:
1. Route reads that follow a user's write (for N seconds, or via a session flag/cookie) to the primary.
2. **Causal tokens**: after a write, record the primary's position (PG `pg_current_wal_lsn()`, MySQL GTID). Then read from a replica only once it has replayed that position (`pg_last_wal_replay_lsn()`; MySQL `WAIT_FOR_EXECUTED_GTID_SET()`). ProxySQL supports GTID-consistent reads.
3. Sticky replica per session (monotonic reads).
4. Keep critical flows such as checkout, auth, and balances on the primary.

Monitor lag: PG `pg_stat_replication` (`replay_lag`), MySQL `SHOW REPLICA STATUS` (`Seconds_Behind_Source` is unreliable; use heartbeat tables like `pt-heartbeat`).

---

## 10. Serverless and Connection Limits {#serverless}

Lambda or Cloud Functions scale to 1,000 concurrent instances, and each may open its own connection, which overwhelms `max_connections` immediately.

Options:
- RDS Proxy / PgBouncer in front.
- **HTTP/websocket-based drivers**: Neon serverless driver, PlanetScale database-js, Supabase/PostgREST, AWS RDS Data API (Aurora).
- Reuse the connection across invocations (declare the client outside the handler), and keep the pool size at 1 per instance.
- Cap concurrency (reserved concurrency) to protect the DB.

---

## 11. Bulk Operations {#bulk}

Row-by-row inserts cost one round trip plus one commit each. Faster options:
| Technique | Speedup |
|---|---|
| Wrap many inserts in one transaction | Avoids a commit/fsync per row |
| Multi-row `INSERT … VALUES (…),(…),(…)` | ~10–100× |
| PG `COPY FROM STDIN` (binary or CSV) | Fastest load path, ~millions of rows/min |
| MySQL `LOAD DATA [LOCAL] INFILE` | Fastest for MySQL |
| PG `INSERT … SELECT * FROM unnest($1::int[], $2::text[])` | Arrays as parameters, 1 statement |
| JDBC `addBatch` + `rewriteBatchedStatements=true` (MySQL) / `reWriteBatchedInserts=true` (PG) | Driver rewrites batches into multi-row inserts |
| Drop or delay secondary indexes during initial bulk load, then create them | Avoids index maintenance per row |

Large deletes: batch by PK range. For time-series data, drop whole **partitions** instead (instant, with no bloat).

---

## 12. Failure Handling {#failure}

- **Classify errors**: retryable (connection reset, failover in progress, serialization/deadlock, `too many connections` with backoff) vs non-retryable (constraint violation, syntax error, permission).
- Retries need **exponential backoff + jitter** and a **retry budget**. Otherwise every client retries 3× during an incident → 4× load on a DB that's already struggling (retry storm).
- **Idempotency**: a retried non-idempotent INSERT after a timeout may double-insert (the first attempt may have committed). Use idempotency keys and UNIQUE constraints, or upserts.
- **Failover**: managed DBs (RDS Multi-AZ, Aurora, Cloud SQL) flip DNS. Clients must reconnect, avoid long DNS caching (the JVM caches DNS forever with a security manager; set `networkaddress.cache.ttl`), and handle in-flight transaction failures. Multi-host connection strings (`host=a,b target_session_attrs=read-write` in libpq/pgjdbc) let the client find the new primary.
- **Circuit breaker** around the DB in the app: when the DB is down, fail fast instead of tying up threads.
- **Bulkheads**: separate pools for critical vs background work, so a reporting job can't starve checkout.

---

## 13. Observability {#obs}

Track per service:
- Pool: active, idle, pending (waiting threads), acquire time histogram, timeouts. HikariCP exposes these via Micrometer.
- Query latency histograms per query name/fingerprint; error rate by SQLSTATE class.
- Rows returned per query (catches accidental full-table fetches).
- Connections opened per second (churn means a pool is misconfigured).
- Distributed tracing spans per query (OpenTelemetry DB semantic conventions: `db.system`, `db.statement` (sanitized), `db.operation`).
- Tag queries with the app/endpoint: `application_name` (PG), `/* controller:orders,action:create */` comments (sqlcommenter / Marginalia) so DB-side views (`pg_stat_activity`, slow log) show where queries come from.

---

## 14. Interview Questions {#qa}

1. Why is opening a DB connection per request bad? What does a pool solve?
2. 40 pods, each with pool size 25, against a PG with `max_connections=500`. What happens, and how do you fix it?
3. Explain PgBouncer's transaction pooling and what features break under it.
4. How do you size a connection pool? Use Little's Law.
5. A user updates their profile and sees stale data on refresh. Why, and how do you fix it?
6. How does a prepared statement prevent SQL injection? What can't it parameterize?
7. Which timeouts should a backend set for DB access, and how should they relate?
8. How would you insert 10 million rows quickly?
9. Which DB errors are safe to retry? What makes retries dangerous?
10. How do serverless functions cause connection exhaustion, and what are the fixes?
