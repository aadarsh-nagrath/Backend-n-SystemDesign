# PostgreSQL Security: Roles, Privileges, Authentication, RLS, Encryption

## Table of Contents
1. [Threat Model for a Database](#threats)
2. [Roles and Privileges](#roles)
3. [A Practical Role Layout for an Application](#layout)
4. [Authentication: pg_hba.conf, SCRAM, Certificates, IAM](#auth)
5. [Row-Level Security (RLS) for Multi-Tenancy](#rls)
6. [Column-Level Security and Views](#columns)
7. [Encryption: In Transit, At Rest, Column-Level](#encryption)
8. [Auditing](#audit)
9. [Common Vulnerabilities and Hardening Checklist](#hardening)
10. [Interview Questions](#qa)

---

## 1. Threat Model {#threats}

- SQL injection through the application (the biggest one, covered in `fundamentals/07`).
- Over-privileged app credentials: the app user can `DROP TABLE`, read every tenant's data, or create extensions.
- Leaked credentials in code, CI logs, or images.
- Network exposure: a DB reachable from the internet (Shodan finds thousands of open 5432/3306 ports, which ransomware bots wipe).
- Backups stored unencrypted or publicly.
- Insider access without auditing.
- Privilege escalation via `SECURITY DEFINER` functions, untrusted languages, or extensions.

---

## 2. Roles and Privileges {#roles}

In Postgres, **users and groups are both "roles"**. A role with `LOGIN` is a user.
```sql
CREATE ROLE app_rw NOLOGIN;                         -- group role
CREATE ROLE api_service LOGIN PASSWORD '...' IN ROLE app_rw;
GRANT app_rw TO api_service;                        -- membership
```
Role attributes: `SUPERUSER` (bypasses all checks; never for apps), `CREATEDB`, `CREATEROLE`, `REPLICATION`, `BYPASSRLS`, `LOGIN`, `CONNECTION LIMIT`, `VALID UNTIL`.

Privileges are granted on objects:
| Object | Privileges |
|---|---|
| Database | CONNECT, CREATE, TEMP |
| Schema | USAGE (needed to access anything inside), CREATE |
| Table | SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER, MAINTAIN (PG 17) — and column-level SELECT/INSERT/UPDATE |
| Sequence | USAGE, SELECT, UPDATE |
| Function | EXECUTE (granted to PUBLIC by default!) |

- **Ownership**: the owner of an object can do anything to it, including DROP and ALTER, and ownership can't be restricted. So **don't let the app role own tables**. Use a separate migration/owner role.
- `PUBLIC` pseudo-role = everyone. Revoke defaults: `REVOKE ALL ON DATABASE app FROM PUBLIC; REVOKE CREATE ON SCHEMA public FROM PUBLIC;` (the latter is default since PG 15).
- `ALTER DEFAULT PRIVILEGES` makes future tables get grants automatically:
  ```sql
  ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
  ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
    GRANT USAGE, SELECT ON SEQUENCES TO app_rw;
  ```
- Predefined roles: `pg_read_all_data`, `pg_write_all_data` (PG 14), `pg_monitor`, `pg_read_all_stats`, `pg_signal_backend`, `pg_checkpoint`, `pg_maintain` (PG 17), `pg_create_subscription` (PG 16).

---

## 3. A Practical Role Layout {#layout}

```sql
-- 1. Owner: owns schema & tables; used only by migrations
CREATE ROLE app_owner NOLOGIN;
CREATE SCHEMA app AUTHORIZATION app_owner;
CREATE ROLE migrator LOGIN PASSWORD '...' IN ROLE app_owner;

-- 2. Read-write group for services
CREATE ROLE app_rw NOLOGIN;
GRANT CONNECT ON DATABASE appdb TO app_rw;
GRANT USAGE ON SCHEMA app TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA app TO app_rw;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app GRANT USAGE, SELECT ON SEQUENCES TO app_rw;

-- 3. Read-only group for analysts/BI
CREATE ROLE app_ro NOLOGIN;
GRANT CONNECT ON DATABASE appdb TO app_ro;
GRANT USAGE ON SCHEMA app TO app_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app GRANT SELECT ON TABLES TO app_ro;

-- 4. Login roles per service (traceability, independent rotation)
CREATE ROLE orders_svc LOGIN PASSWORD '...' IN ROLE app_rw;
ALTER ROLE orders_svc SET statement_timeout = '10s';
CREATE ROLE metabase LOGIN PASSWORD '...' IN ROLE app_ro;
ALTER ROLE metabase SET statement_timeout = '5min';
ALTER ROLE metabase SET default_transaction_read_only = on;
```
One login role per service lets `pg_stat_activity` and the logs show *who* is running what, and lets you revoke one compromised service without touching the others.

---

## 4. Authentication {#auth}

`pg_hba.conf` (host-based authentication) is evaluated **top to bottom, first match wins**:
```
# TYPE   DATABASE  USER        ADDRESS          METHOD
local    all       postgres                      peer
hostssl  appdb     +app_rw     10.0.0.0/16      scram-sha-256
hostssl  replication replicator 10.0.1.0/24     scram-sha-256
hostssl  all       all         0.0.0.0/0        cert clientcert=verify-full
host     all       all         0.0.0.0/0        reject
```
- **`scram-sha-256`**: use it. `password_encryption = scram-sha-256` (default since PG 14). **`md5` is deprecated** (PG 18 warns), and `password`/`trust` are never acceptable over a network.
- `hostssl` forces TLS for that rule; `hostnossl` matches non-TLS connections.
- `peer` (Unix socket OS user = DB user) works for local admin.
- `cert`: client TLS certificates (mTLS).
- LDAP / RADIUS / GSSAPI (Kerberos) / **OAuth** (PG 18, via validator modules) for central identity.
- Cloud: **IAM database authentication** (RDS: short-lived tokens via `generate-db-auth-token`; GCP Cloud SQL IAM). No static passwords.
- Rotate secrets via a **secrets manager** (Vault dynamic database credentials generate a per-lease role with TTL, AWS Secrets Manager rotation).

---

## 5. Row-Level Security {#rls}

RLS adds a WHERE clause (a policy) automatically to every query on a table, per role. It's ideal as **defense in depth for multi-tenant** shared-table designs.

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;   -- apply to the table owner too

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::bigint)           -- rows visible (SELECT/UPDATE/DELETE)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);     -- rows allowed to be written

-- App, per request/transaction:
BEGIN;
SET LOCAL app.tenant_id = '42';      -- SET LOCAL: scoped to this transaction (pooler-safe)
SELECT * FROM orders;                -- only tenant 42's rows
COMMIT;
```
Details:
- Policies can be `PERMISSIVE` (OR-ed, default) or `RESTRICTIVE` (AND-ed), per command (`FOR SELECT | INSERT | UPDATE | DELETE | ALL`), and per role (`TO role`).
- With RLS enabled and **no policy**, the default is deny-all.
- Superusers and roles with `BYPASSRLS` skip RLS. So do owners unless FORCE is set.
- Performance: the policy predicate becomes part of the query, so **index `tenant_id`** (leading column). Use simple, inlinable predicates. Wrapping `current_setting(...)` in `(SELECT …)` helps it get evaluated once. Avoid volatile functions and subqueries per row.
- Leakage: functions marked `LEAKPROOF` can be pushed below the policy filter; non-leakproof user functions in WHERE are evaluated after it.
- Supabase builds its security model on RLS with JWT claims: `auth.uid()` reads `request.jwt.claims`.
- Test it: create tests that connect as the app role with tenant A and assert zero rows from tenant B.

Pitfall: if the app sets the tenant with session-level `SET` (not `SET LOCAL`) behind a transaction-mode pooler, the next client on that connection inherits the wrong tenant. Always use `SET LOCAL` or `set_config('app.tenant_id', $1, true)`.

---

## 6. Column-Level Security and Views {#columns}

```sql
REVOKE SELECT ON users FROM app_ro;
GRANT SELECT (id, name, created_at) ON users TO app_ro;    -- no email/phone
```
Or expose views that mask data:
```sql
CREATE VIEW users_masked WITH (security_barrier, security_invoker = true) AS   -- security_invoker: PG 15
SELECT id, name, left(email, 2) || '***@' || split_part(email, '@', 2) AS email FROM users;
```
`security_barrier` prevents leaky functions from seeing filtered rows. `security_invoker` makes the view check the caller's privileges and RLS (by default views run with the owner's rights).

The `postgresql_anonymizer` extension handles dynamic masking and anonymized dumps for dev/test.

---

## 7. Encryption {#encryption}

**In transit**: TLS.
```
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_min_protocol_version = 'TLSv1.2'      # default since PG 12
```
Clients: `sslmode=verify-full` (verifies the cert *and* hostname). `require` encrypts but **doesn't verify identity**, so it's MITM-able. PG 17 supports direct TLS negotiation (`sslnegotiation=direct`) to save a round trip.

**At rest**: Postgres community edition has **no built-in TDE** (transparent data encryption). Options:
- Disk/volume encryption (LUKS, cloud EBS/PD encryption with KMS). This is the standard, and managed services enable it by default.
- TDE in forks/distros: EDB, Percona's `pg_tde`, Cybertec, Fujitsu.
- Encrypt backups (pgBackRest `repo-cipher-type`, S3 SSE-KMS).

**Column-level / application-level**:
- `pgcrypto`: `pgp_sym_encrypt(data, key)` / `crypt()` for password hashes. The downside is the key passes through SQL, so it can appear in logs and `pg_stat_statements`.
- Better: **encrypt in the application** (envelope encryption with KMS: data key encrypts the field, KMS master key encrypts the data key) for highly sensitive fields (Aadhaar/PAN/SSN, health data). To search them, store a deterministic HMAC (blind index) alongside.
- Passwords: never encrypt, **hash** with bcrypt/scrypt/argon2id in the app (see [`web-security/hashing-algos`](../../web-security/hashing-algos/scrypt-and-bcrypt.md)).

---

## 8. Auditing {#audit}

- `log_statement = 'ddl'` (or `mod` for all writes, which is verbose), `log_connections`, `log_disconnections`.
- **pgaudit** extension: structured session and object audit logging (`pgaudit.log = 'write, ddl, role'`, object-level auditing via an audit role's grants). Needed for SOC 2 / PCI / HIPAA type controls.
- Trigger-based data change audit (see fundamentals/06).
- Ship logs to a SIEM with immutability. Alert on role changes, grants, logins from new networks, mass reads by unusual roles.

---

## 9. Hardening Checklist {#hardening}

- [ ] DB not publicly reachable; private subnet; security groups allow only app subnets / bastion / VPN.
- [ ] `listen_addresses` restricted; `pg_hba.conf` least privilege, ends with `reject`.
- [ ] SCRAM only; TLS required (`hostssl`), clients use `verify-full`.
- [ ] No app connects as a superuser or as the table owner.
- [ ] Separate roles per service; read-only roles for BI; default privileges set.
- [ ] `REVOKE CREATE ON SCHEMA public FROM PUBLIC` (pre-15); `REVOKE ALL ON DATABASE … FROM PUBLIC`.
- [ ] `SECURITY DEFINER` functions set `search_path` and are owned by minimal-privilege roles; revoke EXECUTE from PUBLIC on sensitive functions.
- [ ] Untrusted languages (plpython3u) and dangerous extensions (`adminpack`, `file_fdw` for non-superusers, `dblink` to arbitrary hosts) restricted.
- [ ] `COPY … TO/FROM PROGRAM` and server file access only for superusers / `pg_execute_server_program` (keep it that way).
- [ ] Secrets in a vault; rotation; no passwords in `~/.pgpass` on shared hosts or in Git.
- [ ] Encryption at rest + encrypted, access-controlled, immutable backups.
- [ ] pgaudit for regulated data; logs shipped off-host.
- [ ] Minor versions kept current (security CVEs are fixed in minors).
- [ ] RLS for multi-tenant tables as defense in depth, with tests.
- [ ] Statement and idle timeouts per role to reduce DoS impact.

---

## 10. Interview Questions {#qa}

1. Why shouldn't the application's DB user own the tables?
2. How do `ALTER DEFAULT PRIVILEGES` and grants on existing tables differ?
3. Implement tenant isolation with RLS. What's the pooler pitfall?
4. `sslmode=require` vs `verify-full`?
5. How do you encrypt a column so it's still searchable by exact value?
6. What's the risk of a SECURITY DEFINER function without a fixed search_path?
7. How would you give analysts access without exposing PII?
