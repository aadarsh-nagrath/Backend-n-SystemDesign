# Redis

Redis (**RE**mote **DI**ctionary **S**erver) is an open-source, in-memory, multi-model database known for sub-millisecond latency. Created in 2009 by Salvatore Sanfilippo ("antirez") to solve a real problem: applications like early Twitter were growing fast enough that disk-bound relational databases couldn't keep up, and something needed to serve hot data out of RAM instead.

## TL;DR
- All data lives in RAM (fast), with optional persistence to disk (RDB snapshots, AOF log) so it survives restarts.
- Rich data structures beyond key-value: strings, lists, sets, sorted sets, hashes, streams, bitmaps, HyperLogLogs, geospatial — each with its own atomic commands.
- Single-threaded command execution (multi-threaded I/O since v6) — no locks needed for individual commands, but one slow command blocks everything behind it.
- Common as a **cache in front of a database**, but flexible enough to be a primary store for sessions, queues, leaderboards, and real-time analytics.
- Caching with Redis means picking a **pattern** (cache-aside, write-through, write-behind) and an **eviction policy** (LRU, LFU, TTL) — get these wrong and you get stale data or OOM crashes.
- Not a drop-in replacement for a relational DB when you need full ACID multi-row transactions across large datasets — it trades some of that for raw speed and simplicity.

## Why Redis exists

Historically, Redis has been used mainly as a **key-value cache** sitting in front of a relational database — cache frequently-read query results, cut load on the primary DB, cut response times. That's still its most common role. But Redis has grown into a genuine **multi-model database**: with its native structures and optional modules, teams sometimes use it as the primary store, which can *reduce* system complexity because high-scale performance is built in rather than bolted on via a separate caching layer.

Every value in Redis lives under a unique **key**, and the value can be one of several structures (strings, lists, hashes, etc.) rather than being forced into rows or JSON blobs. This mirrors how programming languages already model data, which is a big part of why it feels natural to use.

**Strengths:**
- Sub-millisecond latency for most operations (in-memory).
- Multiple data models via core features and optional modules (document, time-series, vector search for AI).
- Built-in replication, clustering, sharding for horizontal scale.
- Configurable persistence, trading speed for durability as needed.
- Atomic operations for safe concurrent access.
- Pub/Sub messaging and Lua scripting for custom server-side logic.

**Trade-offs:**
- Being in-memory means RAM cost scales with dataset size — large datasets get expensive.
- Full durability (`appendfsync always`) costs some of the speed that's the whole point of using Redis.
- Not the right tool for workloads needing complex multi-row ACID transactions at scale — that's still a relational database's job.

Rule of thumb: use Redis as primary storage when you need speed, simplicity, and flexible data models (real-time apps, caching, sessions). Reach for a relational database when you need strict ACID guarantees across complex multi-table transactions (financial ledgers, for instance) — or use both together, which is the far more common setup.

## A brief history

- **2009** — Salvatore Sanfilippo builds Redis 1.0 as a fast in-memory cache for an Italian startup's web app; released under BSD 3-clause.
- **2010–2012** — rapid adoption; Redis 2.0 adds Lua scripting and better persistence; Twitter adopts it for timelines and caching.
- **2013** — Redis Labs (now Redis Inc.) forms to provide enterprise support.
- **2015** — Redis 3.0 adds clustering for horizontal scale.
- **2018** — Redis 5.0 introduces Streams, positioning Redis as a lightweight alternative to Kafka for some workloads.
- **2020** — Redis 6.0 adds ACLs and client-side caching (Tracking), plus multi-threaded I/O.
- **2021–2023** — Redis Stack bundles JSON, Search, Graph, and TimeSeries modules; Redis 7.0 lands.
- **2024–2025** — heavy focus on AI/ML workloads: vector search via RediSearch for RAG (Retrieval-Augmented Generation) pipelines; deeper integration with AWS/Azure/GCP managed offerings.

Redis is written primarily in C, has 60k+ GitHub stars, and powers high-traffic systems at Twitter/X, GitHub, Snapchat, Craigslist, and Stack Overflow.

## Architecture

Redis is a **client-server** system with a single-threaded event loop for command execution — this avoids locking overhead and keeps individual commands atomic almost for free. The server listens on TCP (default port `6379`).

**Core components:**
- **Memory storage** — everything lives in RAM, allocated via jemalloc by default.
- **Persistence layer** — optional RDB snapshots and/or AOF (append-only file) writes to disk.
- **Replication** — asynchronous master-replica replication for read scaling and failover.
- **Clustering** — sharding across nodes using 16,384 hash slots distributed among masters.
- **Event loop** — I/O multiplexing (epoll/libevent-style) handling many connections on one thread.
- **Protocol** — RESP (Redis Serialization Protocol), a simple, binary-safe TCP protocol.

### Single-threaded model
Commands execute sequentially on one thread, so no locks are needed and every command is inherently atomic relative to others. The trade-off: a single CPU-heavy command (a large `SORT`, a big `KEYS *` scan) blocks everything behind it. Redis 6.0 added multi-threading for I/O (accepting/parsing connections) but core command *execution* is still single-threaded.

### Memory management
- **Eviction policies** — when `maxmemory` is hit, Redis evicts keys per `maxmemory-policy`: LRU, LFU, or random (see the eviction section below for the full list).
- **Key expiration** — keys can carry a TTL for automatic removal.
- **Memory hygiene** — avoid very large individual keys (>10KB is a rough danger threshold; big values can block the single thread during access).

### Deployment modes
- **Standalone** — single instance, simplest setup.
- **Sentinel** — adds automatic failover and monitoring on top of standalone/replicated setups.
- **Cluster** — sharding and horizontal scale-out.
- **Redis Enterprise** — adds CRDT-based active-active replication (CRDB) across regions.

## Redis vs. Memcached

The other major in-memory cache, and the first comparison most teams make before choosing.

| | Redis | Memcached |
|---|---|---|
| Data structures | Strings, lists, sets, sorted sets, hashes, streams, bitmaps, HyperLogLog, geo | Strings only (values are opaque blobs) |
| Persistence | Optional (RDB snapshots, AOF log) | None — pure in-memory, data is gone on restart |
| Threading | Single-threaded execution (multi-threaded I/O since v6) | Multi-threaded from the start |
| Replication/clustering | Built-in (Sentinel, Cluster) | Not built-in — client-side sharding only |
| Atomic operations | Rich (INCR, transactions, Lua scripts) | Limited (INCR/DECR, CAS) |
| Pub/Sub, streams | Yes | No |
| Max value size | 512MB | 1MB by default |
| Memory efficiency for simple caching | Slightly higher overhead per key (extra metadata for structures) | Slightly leaner for pure key→string caching |
| Typical fit | Anything beyond trivial caching: sessions, queues, leaderboards, pub/sub, rate limiting | Extremely simple, high-throughput, ephemeral key-value caching with predictable eviction |

**Practical takeaway**: choose Memcached when your entire need is "cache flat strings, evict under memory pressure, nothing else" and you want its slightly simpler multi-threaded model. Choose Redis for almost everything else — richer data types, persistence, pub/sub, and the ability to serve as more than just a cache. Most new projects default to Redis because the extra capability costs little and is frequently needed later.

## Installation and setup

Redis runs on Linux, macOS, Windows (via WSL/Docker), and more.

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```
Verify: `redis-cli ping` → `PONG`.

**From source:**
```bash
wget https://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
sudo make install
# with modules:
make BUILD_WITH_MODULES=yes
```

**Docker:**
```bash
docker run --name my-redis -p 6379:6379 redis:latest
# Redis Stack (bundled modules):
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest
```

**Windows**: use WSL2 or Docker — native support is experimental.

### Key config parameters (`redis.conf`)
```conf
bind 127.0.0.1          # bind to localhost for security
port 6379
requirepass yourpassword # enable authentication
maxmemory 2gb            # cap RAM usage
maxmemory-policy allkeys-lru
save 900 1                # RDB snapshot every 900s if >=1 key changed
appendonly yes             # enable AOF
```
Reload without restart: `redis-cli CONFIG REWRITE`.

### Managed / cloud option
Redis Cloud's free tier (up to 30MB, one extra module) is the fastest way to get a hosted instance without local setup — sign up, create a database, get connection details, attach modules as needed.

### Client tools
- **redis-cli** — `redis-cli -h host -p 6379 -a password`; basic commands like `SET key value`, `GET key`, `DEL key`.
- **RedisInsight** — GUI for browsing data, monitoring, and CRUD, available for Windows/macOS/Linux/Docker.
- **VS Code extension** — query/manage from the editor.

### Connecting from application code
```python
# Python (redis-py): pip install redis
import redis
r = redis.Redis(host='localhost', port=6379, password='password')
r.set('key', 'value')
print(r.get('key'))  # b'value'
```
```javascript
// Node.js (node-redis): npm install redis
const redis = require('redis');
const client = redis.createClient({ url: 'redis://localhost:6379' });
await client.connect();
await client.set('key', 'value');
console.log(await client.get('key'));  // 'value'
```
```java
// Java (Jedis)
import redis.clients.jedis.Jedis;
Jedis jedis = new Jedis("localhost", 6379);
jedis.auth("password");
jedis.set("key", "value");
System.out.println(jedis.get("key"));  // "value"
```
Other languages: Go (go-redis), .NET (StackExchange.Redis), PHP (Predis). See [redis.io/docs/clients](https://redis.io/docs/clients/).

Connection best practices: use connection pooling, handle reconnects gracefully, set sane timeouts.

## Data types and structures

Every value is key-based with atomic operations per structure.

### Strings
Binary-safe, up to 512MB. Used for caching, counters, sessions.
```
SET user:1 "John Doe" EX 3600   # set with 1-hour expiry
GET user:1
DEL user:1
INCR visits                      # atomic increment
APPEND key value
STRLEN key
SETBIT key offset value          # bit ops
BITCOUNT key
```
Use cases: cache HTML fragments, rate limiting (`INCR ip:192.168.1.1` + expire).

### Lists
Ordered collections (doubly-linked list), ideal for queues/stacks.
```
LPUSH mylist "item1"
RPUSH mylist "item2"
LRANGE mylist 0 -1       # ["item1", "item2"]
BLPOP mylist 5           # blocking pop, for consumer queues
```
Keep under ~10k elements per list for good performance.

### Sets
Unordered, unique members, with set algebra.
```
SADD fruits "apple" "banana"
SADD veggies "carrot" "banana"
SINTER fruits veggies    # ["banana"]
SCARD fruits              # cardinality
```
Use cases: tags, unique visitor tracking, mutual-friends via intersection.

### Sorted sets (ZSets)
Sets ordered by a score — effectively a priority queue.
```
ZADD leaderboard 100 "player1" 200 "player2"
ZRANGE leaderboard 0 -1 WITHSCORES
ZREVRANK leaderboard "player1"
```
Use cases: leaderboards, rate limiting (score = timestamp).

### Hashes
Field-value maps, like an object/dict.
```
HSET user:1 name "John" age 30
HGET user:1 name
HGETALL user:1            # {name: "John", age: "30"}
HINCRBY user:1 age 1
```
More memory-efficient than storing a JSON string for flat objects.

### Bitmaps
Strings interpreted as bit arrays.
```
SETBIT users:week1 123 1   # user 123 logged in
BITCOUNT users:week1 0 1000
```
Use cases: compact login tracking, Bloom-filter-style checks.

### HyperLogLogs
Probabilistic cardinality estimation with ~0.81% error, using a tiny fixed memory footprint regardless of set size.
```
PFADD pageviews:page1 "user1" "user2"
PFCOUNT pageviews:page1   # ~2
```
Use cases: unique visitor counts at scale where exact counts aren't worth the memory.

### Streams
Append-only log, roughly "Kafka lite."
```
XADD mystream * sensor "temp:25"
XREAD STREAMS mystream 0
XGROUP CREATE mystream mygroup $
```
Use cases: event sourcing, chat logs, with consumer groups for pub/sub-style fan-out.

### Geospatial indexes
Sorted sets scored by lat/long.
```
GEOADD Sicily 15.0877 37.5025 "Palermo"
GEORADIUS Sicily 15 37 200 km
```
Use cases: location services, ride-sharing "nearby drivers" queries.

### JSON (RedisJSON module)
```
JSON.SET doc $ '{"name": "John"}'
JSON.GET doc $
JSON.ARRAPPEND doc $.tags "new"
```
Use cases: document store, hierarchical data without a separate document DB.

### TimeSeries (RedisTimeSeries module)
```
TS.ADD temp:1 1640995200 23.5
TS.RANGE temp:1 - +
```
Use cases: IoT sensor data, stock prices, metrics.

## Commands, transactions, and scripting

Redis has 200+ commands. `redis-cli --scan` lists keys safely (never use `KEYS *` in production — it's O(N) and blocks); `HELP @string` gives group help.

### Basic CRUD
```
EXISTS key
EXPIRE key seconds
TTL key
FLUSHDB / FLUSHALL   # clears current/all DBs — use with caution
```

### Transactions
`MULTI`/`EXEC` bundle commands atomically:
```
MULTI
SET a 1
INCR a
EXEC
```
`WATCH` enables optimistic locking (check-and-set): watch a key, and if it changes before `EXEC`, the transaction aborts.

### Lua scripting
Runs server-side, avoiding multiple round-trips for multi-step logic:
```lua
-- Increment if not present, else INCR
if redis.call('GET', KEYS[1]) == false then
  redis.call('SET', KEYS[1], 1)
else
  redis.call('INCR', KEYS[1])
end
```
Call via `EVAL script numkeys key [key ...] arg [arg ...]`.

### Pub/Sub
```
PUBLISH news "Breaking!"
SUBSCRIBE news
PSUBSCRIBE news.*   # pattern subscribe
```

### Sorting and pagination
```
SORT users BY score:* LIMIT 0 10
```

Full reference: [redis.io/commands](https://redis.io/commands/). Most commands are O(1); scans/sorts are O(N).

## Caching patterns

Redis is most often deployed as a cache in front of a slower system of record (usually a relational database). How you keep the two in sync is one of the most consequential decisions in a caching design. See [`../server-side-caching/ss.md`](../server-side-caching/ss.md) for the pattern-agnostic version of this discussion — this section covers the same patterns with Redis-specific mechanics.

### Cache-aside (lazy loading)

The application owns the caching logic: check Redis first, fall back to the database on a miss, then populate Redis for next time.

```python
def get_user(user_id):
    cached = redis_client.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)          # cache hit

    user = db.query("SELECT * FROM users WHERE id = %s", user_id)  # cache miss
    redis_client.set(f"user:{user_id}", json.dumps(user), ex=300)   # populate, 5 min TTL
    return user
```
- **Pros**: only requested data gets cached (no wasted memory on cold data); Redis being down degrades to "just hit the DB," not a hard failure.
- **Cons**: first request for any key always pays a full cache-miss penalty; data can go stale between writes and the next natural expiry/refresh.
- This is the default choice for most read-heavy caching — it's what most people mean when they say "we cache with Redis."

### Write-through

Writes go to the cache and the database together, synchronously, as part of the same operation. Reads always hit a cache that's guaranteed current.

```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    redis_client.set(f"user:{user_id}", json.dumps(data), ex=300)   # update cache immediately
```
- **Pros**: cache is never stale relative to the database (assuming no other write path bypasses this function).
- **Cons**: every write pays the cost of two operations; if the DB write succeeds but the Redis write fails (or vice versa), the two can diverge — needs care around ordering and error handling (write DB first, then cache, and treat a cache-write failure as non-fatal since cache-aside can recover it on next read).

### Write-behind (write-back)

Writes go to the cache immediately and are pushed to the database asynchronously (batched, queued, or on a timer).

```python
def update_user_fast(user_id, data):
    redis_client.set(f"user:{user_id}", json.dumps(data))
    write_queue.enqueue({"table": "users", "id": user_id, "data": data})  # async DB flush later
```
- **Pros**: very low write latency (the DB round-trip is off the critical path); can batch/coalesce many writes into fewer DB operations.
- **Cons**: real risk of data loss if Redis crashes before the queued write reaches the database; the most operationally complex of the three patterns — needs a durable queue and retry/reconciliation logic.
- Used when write throughput matters more than the small risk window (e.g., view counters, activity logs) — rarely used for anything where losing a write is unacceptable (financial data).

| Pattern | Read path | Write path | Staleness risk | Complexity |
|---|---|---|---|---|
| Cache-aside | App checks cache, falls back to DB | App writes DB, invalidates/updates cache separately | Possible between write and next cache refresh | Low |
| Write-through | Always hits cache (kept current) | App writes both cache and DB synchronously | Low, if the write path is the only writer | Medium |
| Write-behind | Always hits cache | App writes cache only; DB updated async | Higher — window where DB lags cache | High |

## Cache stampede (thundering herd)

A **cache stampede** happens when a popular cached key expires (or the cache goes cold after a restart) and a large burst of concurrent requests all miss the cache at once, all fall through to the database simultaneously, and overwhelm it — sometimes badly enough to cause a cascading outage. This is a classic, commonly-tested caching failure mode, distinct from ordinary cache misses because of the *simultaneity*.

**Why it happens**: a hot key with a fixed TTL expires at a single instant; if that key gets thousands of requests per second, all of them miss in the same window and hit the database at once, often triggering the *same* expensive query thousands of times redundantly.

### Mitigations

**1. Mutex / locking (only one request repopulates the cache)**
```python
def get_user_safe(user_id):
    cached = redis_client.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    lock_key = f"lock:user:{user_id}"
    if redis_client.set(lock_key, "1", nx=True, ex=10):  # only one caller wins the lock
        try:
            user = db.query("SELECT * FROM users WHERE id = %s", user_id)
            redis_client.set(f"user:{user_id}", json.dumps(user), ex=300)
            return user
        finally:
            redis_client.delete(lock_key)
    else:
        time.sleep(0.05)              # brief wait, then retry read
        return get_user_safe(user_id)
```
`SET key val NX EX 10` is Redis's atomic "set if not exists with expiry" — the classic building block for a distributed lock.

**2. Early/probabilistic expiration** — recompute the cache *before* it actually expires, with rising probability as the TTL nears zero, so a fraction of requests refresh it early and the rest keep getting a hit. Spreads the cost instead of concentrating it at one instant.

**3. Staggered/jittered TTLs** — instead of `ex=300` for every key, use `ex=300 + random(0, 30)`. Prevents many keys cached at the same moment (e.g., a cold-start warm-up) from all expiring simultaneously later.

**4. Background refresh** — a scheduled job refreshes hot keys before they expire, so user-facing requests never see a cold cache for popular data at all.

**5. Serve-stale-while-revalidating** — keep serving the expired value (past TTL) to incoming requests while exactly one background request refreshes it, rather than making everyone wait or fall through to the DB.

Which mitigation to use depends on how hot the key is and how expensive the fallback query is — a lock is simplest and covers most cases; jittered TTLs are nearly free and worth doing by default for anything cached in bulk.

## Persistence and durability

Redis is in-memory but not necessarily ephemeral — persistence options trade speed against durability.

### RDB (snapshotting)
Point-in-time dumps to disk.
```conf
save 60 1000   # snapshot if >=1000 keys changed within 60s
```
`BGSAVE` triggers a background save. Fast and compact, but you can lose everything written since the last snapshot. Good for backups.

### AOF (append-only file)
Logs every write operation.
```conf
appendonly yes
appendfsync everysec       # fsync every second — good durability/speed balance
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```
`BGREWRITEAOF` compacts the log. More durable than RDB alone, but larger files and slower.

**Best practice**: use both — RDB for fast full backups/restores, AOF for fine-grained recovery. Redis loads the AOF on startup if present, falling back to RDB otherwise. For maximum durability, `appendfsync always` fsyncs on every write (noticeably slower).

## Eviction policies

Once `maxmemory` is reached, Redis needs a policy for what to evict:

| Policy | Behavior |
|---|---|
| `noeviction` | Refuse new writes once memory is full (errors instead of evicting) |
| `allkeys-lru` | Evict least-recently-used key, from all keys |
| `volatile-lru` | Evict least-recently-used key, but only among keys with a TTL set |
| `allkeys-lfu` | Evict least-frequently-used key, from all keys |
| `volatile-lfu` | Evict least-frequently-used key, only among keys with a TTL |
| `allkeys-random` | Evict a random key from all keys |
| `volatile-random` | Evict a random key, only among keys with a TTL |
| `volatile-ttl` | Evict the key with the nearest expiry first |

`allkeys-lru` is the standard choice for a pure cache. `volatile-lru` (or `volatile-ttl`) suits a mixed workload where some keys are permanent (primary data) and only TTL'd keys should ever be evicted.

## Replication and high availability

### Replication
Asynchronous master-replica:
```
SLAVEOF master_host master_port
WAIT numreplicas timeout   # block until N replicas acknowledge
```
Use: read scaling, failover readiness.

### Sentinel
Monitors masters and automates failover.
```conf
sentinel monitor mymaster ip port quorum
```
`SENTINEL get-master-addr-by-name mymaster` to discover the current master.

### Clustering
Sharding via 16,384 hash slots.
```
cluster-enabled yes
CLUSTER NODES
```
Use hash tags (`{tag}`) to force related keys onto the same slot for multi-key operations. Trade-off: auto-sharding and fault tolerance, but multi-key operations (transactions, Lua scripts touching multiple keys) are restricted to keys in the same slot.

For active-active multi-region writes, Redis Enterprise's CRDB (Conflict-free Replicated Data Types) is the tool — open-source clustering doesn't support multi-master.

## Modules and multi-model features

Redis extends via loadable modules (`loadmodule /path/to/module.so`). **Redis Stack** bundles the common ones (JSON, Search, Graph, TimeSeries) into one distribution.

- **RedisJSON (ReJSON)** — JSON document store with path queries (`JSON.GET doc $.users[0].name`).
- **RediSearch** — full-text and vector similarity search.
  ```
  FT.CREATE idx SCHEMA title TEXT body TEXT
  FT.SEARCH idx "redis"
  FT.CREATE idx SCHEMA embedding VECTOR HNSW 6 DIM 128   # vector index for RAG/semantic search
  ```
  ```javascript
  const schema = { '$.brand': { type: 'TEXT', AS: 'brand' } };
  await client.ft.create('idx:bicycle', schema, { ON: 'JSON', PREFIX: 'bicycle:' });
  await client.ft.search('idx:bicycle', '@brand:"Noka Bikes"');
  ```
- **RedisGraph** — graph database with Cypher-style queries (`GRAPH.QUERY db "CREATE (a:Person {name:'Alice'})"`).
- **RedisTimeSeries** — as above.
- **RedisBloom** — probabilistic filters (`BF.ADD filter item`) for deduplication/existence checks without false negatives.
- **RedisGears** — server-side scripting triggered on data changes, for real-time processing.
- **RedisAI**, **Redis Raft** — ML model serving, consensus, respectively.

Modules effectively turn Redis into a vector DB, search engine, or graph DB as needed — one free module is available on the Redis Cloud free tier.

## Performance tuning

### Memory
- Set `maxmemory` to 80-90% of available RAM, not 100%.
- Policy: `allkeys-lru` for pure caches, `volatile-lru` when mixing permanent and cached data.
- Check per-key size with `MEMORY USAGE key`; split values over ~10KB.
- `lazyfree-lazy-eviction yes` for non-blocking evictions.

### Persistence
- Speed-first: RDB only, no AOF.
- Durability-first: AOF `everysec` + RDB.
- Run RDB snapshots on replicas, not the master, to avoid blocking the master's fork().
- Keep instances under ~10GB to keep `fork()` (needed for RDB/AOF rewrite) fast.

### Network/IO
- Bind to a private IP (`bind 127.0.0.1` for local-only).
- `tcp-keepalive 300`.
- Pipeline commands to cut round-trips.
- Client-side connection pooling (10-50 connections is a typical range).

### CPU/threading
- Disable Transparent Huge Pages (THP) — `echo never > /sys/kernel/mm/transparent_hugepage/enabled` — they slow `fork()`.
- Multi-core I/O: `io-threads 4` (Redis 6+).
- Never run `KEYS *` in production — use `SCAN` (cursor-based, non-blocking).

### Benchmarking and monitoring
```bash
redis-benchmark -t set,get -n 100000 -q
```
`INFO` (e.g. `used_memory`, `instantaneous_ops_per_sec`), RedisInsight, or a Prometheus exporter for ongoing monitoring.

Example high-perf tuning:
```conf
maxmemory 4gb
maxmemory-policy allkeys-lru
save ""
appendonly no   # cache-only workload, no durability needed
tcp-backlog 511
timeout 0
```

### Best practices
- Pick the right structure (hashes over strings for objects); use consistent key prefixes (`user:123:session`).
- Always set TTLs on cache entries to prevent unbounded memory growth.
- Pipeline or use Lua for multi-step operations.
- Replicas for read scaling; cluster for horizontal write scaling.
- Track `SLOWLOG GET`, memory, and connection count; wire into Grafana.
- Load test and simulate failures before trusting a config in production.
- Prefer `SCAN`/`HSCAN` over `KEYS`/`SMEMBERS` for anything that could be large.

## Security

Redis has **no authentication by default** — securing it is not optional for anything internet-reachable.

- **Authentication**: `requirepass strongpassword` (long, high-entropy).
- **ACLs (Redis 6+)**: per-user permissions.
  ```
  ACL SETUSER alice on >password ~keys:* +get +set
  ACL LIST
  ACL WHOAMI
  ```
- **Encryption**: TLS (`tls-port 6380` with certs); Stunnel as a workaround for older versions without native TLS.
- **Network**: bind to localhost or a private network, firewall to specific IPs, and disable dangerous commands (`rename-command FLUSHALL ""`).
- **Protected mode**: on by default, rejects non-local connections without auth.
- **Least privilege**: don't run as root; disable THP as noted above.
- **Threat checklist**: brute-force → strong password + firewall; injection → RESP is binary-safe, but still sanitize app-level inputs; DoS → `maxclients 10000` and watch slow ops; eavesdropping → TLS in production.
- **Enterprise**: adds RBAC and audit logging.

```conf
requirepass yourstrongpass123!
rename-command CONFIG ""
protected-mode yes
```

## Use cases

- **Cache in front of a DB** — the classic role: app queries Redis first, falls back to DB on miss, sets with a TTL.
- **Primary store for sessions/configs** — reduces architectural complexity by removing a separate cache layer.
- **Real-time analytics** — HyperLogLog for unique visitor counts, Streams for event logs.
- **Leaderboards** — sorted sets (`ZADD scores 100 "user1"`, `ZREVRANGE scores 0 9 WITHSCORES`).
- **Session store** — hashes with TTL (`HSET session:abc123 user_id 1`, `EXPIRE session:abc123 3600`).
- **Message broker** — Pub/Sub for chat/notifications, Streams for durable queuing with consumer groups.
- **Document store** — RedisJSON for storing/querying user documents.
- **Graph DB** — RedisGraph for relationship queries (social graphs, recommendations).
- **Time-series** — RedisTimeSeries for IoT/metrics data.
- **Vector search for AI** — RediSearch embeddings for similarity search in RAG pipelines (`FT.SEARCH idx "@vector:[KNN 10 @embedding $vec]"`).

### Full example: web app cache (Node.js)
```javascript
const express = require('express');
const redis = require('redis');
const app = express();
const client = redis.createClient();

app.get('/user/:id', async (req, res) => {
  const { id } = req.params;
  let user = await client.get(`user:${id}`);
  if (!user) {
    user = JSON.stringify({ id, name: 'John' }); // simulate DB fetch
    await client.set(`user:${id}`, user, { EX: 300 });
  }
  res.json(JSON.parse(user));
});

app.listen(3000);
```

## Advanced topics

- **Lua scripting in depth** — register reusable scripts with `SCRIPT LOAD`, invoke by SHA to skip re-sending the script text; reduces latency for multi-step server-side logic.
- **Client-side caching (Redis 6.2+)** — `CLIENT CACHING yes` on a connection offloads caching to the client itself; see [`../clientside/client-side-caching.md`](../clientside/client-side-caching.md) for the full mechanics (Tracking modes, invalidation, connection models).
- **Module development** — modules are written in C and loaded dynamically; see the Redis GitHub for examples.
- **Integrations** — Prometheus/Grafana for metrics, Kafka/Streams as complementary or alternative messaging, ELK stack for log processing.
- **Migration** — the RIOT tool moves data between Redis instances: `riot-redis --read redis://source --write redis://target`.

## Redis Enterprise and Cloud

Redis Enterprise extends the open-source core with:
- **CRDB** — multi-master, conflict-free replication.
- **Active-active** — geo-distributed writes across regions.
- **Advanced vector DB features** for AI workloads.
- **Managed cloud** on AWS/Azure/GCP, with a free tier (30MB DB + 1 module).
- **Compliance** — GDPR, HIPAA support, plus enterprise support contracts.

See [redis.io/pricing](https://redis.io/pricing) for current tiers.

## Monitoring and troubleshooting

- **Commands**: `INFO` (server/memory/stats sections), `SLOWLOG GET 10` (recent slow queries), `MONITOR` (live command stream — resource-heavy, use sparingly).
- **Tools**: RedisInsight (dashboards, slow log viewer), `redis-cli --latency` for latency history.
- **Common issues**:
  - OOM → tune eviction policy and `maxmemory`.
  - High latency → check RDB fork time, THP status, network.
  - Connection issues → proper pooling, increase `tcp-backlog`.
- **Logs**: `loglevel notice`, check `/var/log/redis/redis.log`.

## Further reading
- [roadmap.sh/redis](https://roadmap.sh/redis) — structured learning path for Redis
- [redis.io/commands](https://redis.io/commands/) — full command reference
- [redis.io/docs/clients](https://redis.io/docs/clients/) — client library list
- [redis.io/docs/latest/develop/reference/client-side-caching](https://redis.io/docs/latest/develop/reference/client-side-caching/) — Tracking feature docs
- [redis.io/pricing](https://redis.io/pricing) — Redis Cloud/Enterprise pricing
