# Comprehensive Notes on Redis

## Introduction

Redis, which stands for **Remote Dictionary Server**, is an open-source, in-memory, multi-model database renowned for its exceptional performance and sub-millisecond latency. Created in 2009 by Salvatore Sanfilippo (also known as antirez), Redis was initially developed to address the need for a fast caching solution that could also serve as a durable data store. At the time, applications like Twitter were experiencing exponential growth, and traditional relational databases struggled to deliver data to end users quickly enough due to disk I/O bottlenecks. Redis revolutionized this by storing and manipulating data primarily in the computer's main memory (RAM), which is orders of magnitude faster than disk-based storage, while still providing durability through disk persistence mechanisms like snapshots and append-only files (AOF).

Unlike traditional databases that rely on slower disk reads and writes, Redis operates as an in-memory data store, meaning all data modifications and reads happen directly in RAM. However, to ensure durability and prevent data loss in case of crashes or restarts, Redis supports persistence options that allow data to be reconstructed from disk as needed. This makes Redis fully durable, balancing speed with reliability.

Historically, Redis has been primarily described and used as a **key-value store**, often serving as a caching layer to accelerate relational databases at scale. For example, it can cache frequently accessed query results from a primary SQL database, reducing load and improving response times. However, Redis has evolved far beyond a simple cache. It is now a **multi-model database**, supporting a variety of data structures and paradigms, making it suitable as a primary database. Using Redis as the primary store can dramatically reduce system complexity because high-scale performance is inherently built-in—no need for elaborate caching layers, sharding logic, or micro-optimizations that often complicate architectures.

Redis's flexibility allows developers to store data in a natural way, mirroring structures from programming languages (e.g., strings, lists, hashes) rather than forcing it into rigid tables or JSON blobs. Every piece of data in Redis is identified by a unique **key**, followed by one of several supported **data structures** (e.g., strings, lists, hashes, sets, sorted sets, streams, and more with modules). Interaction with Redis is straightforward via a simple command protocol, such as `SET key value` to store data and `GET key` to retrieve it.

Redis powers some of the world's most heavily trafficked sites, including Twitter (now X), GitHub, Snapchat, Craigslist, Stack Overflow, and many others. Its adoption stems from its ability to handle real-time workloads, such as leaderboards, session stores, real-time analytics, and message queuing. As of 2025, Redis continues to be actively developed, with the latest stable version being Redis 7.x (open-source) and advanced features in Redis Enterprise.

Key advantages of Redis include:
- **High Performance**: Sub-millisecond latency for most operations due to in-memory storage.
- **Versatility**: Supports multiple data models (key-value, document, graph, time-series, vector search for AI, etc.) via core features and optional modules.
- **Scalability**: Built-in replication, clustering, and sharding for horizontal scaling.
- **Durability Options**: Configurable persistence to balance speed and data safety.
- **Atomic Operations**: Ensures consistency for concurrent access.
- **Pub/Sub Messaging**: For real-time communication.
- **Lua Scripting**: For custom server-side logic.

Potential drawbacks include:
- **Memory Usage**: As an in-memory store, it requires sufficient RAM; large datasets can be expensive.
- **Persistence Trade-offs**: Full in-memory speed comes at the cost of potential data loss if not configured properly.
- **Not Ideal for All Workloads**: Best for read-heavy, low-latency needs; less suited for complex ACID transactions compared to relational DBs.

Would you use Redis as your primary database? It depends on your use case. Yes, if you need speed, simplicity, and flexible data models (e.g., real-time apps, caching, sessions). No, if you require full ACID compliance for financial transactions or massive write-heavy OLTP without careful tuning.

## History and Evolution

Redis was conceived in 2009 by Salvatore Sanfilippo while working on a project for an Italian startup. The initial goal was to create a fast in-memory cache to replace slower components in a web application. By December 2009, version 1.0 was released under the BSD 3-clause license, making it open-source. Early adopters included high-traffic sites needing low-latency data access.

Key milestones:
- **2010-2012**: Rapid adoption; Redis 2.0 introduced Lua scripting and better persistence. Companies like Twitter integrated it for timelines and caching.
- **2013**: Redis Labs (now Redis Inc.) formed to provide enterprise support, leading to Redis Enterprise.
- **2015**: Redis 3.0 added clustering for horizontal scaling.
- **2018**: Redis 5.0 introduced Streams for message queuing, competing with Kafka for some use cases.
- **2020**: Redis 6.0 enhanced security with ACLs (Access Control Lists) and improved performance.
- **2021-2023**: Redis Stack emerged, bundling modules like JSON, Search, and Graph. Redis 7.0 added client-side caching and better multi-threading.
- **2024-2025**: Focus on AI/ML with vector search in RediSearch; integration with cloud providers like AWS, Azure, GCP. Redis 7.2+ includes enhancements for GenAI (e.g., RAG - Retrieval Augmented Generation).

As of September 18, 2025, Redis is at version 7.2.x for open-source, with ongoing developments in vector databases for AI workloads. The project has over 60k GitHub stars, a vibrant community, and is written primarily in C for optimal performance.

## Architecture

Redis follows a **client-server architecture** with a single-threaded event loop for command execution, which minimizes context switching and maximizes throughput. The server listens on a TCP port (default 6379) for client connections. Data is stored in memory as key-value pairs, where values can be complex data structures.

### Core Components
- **Memory Storage**: All data resides in RAM. Redis uses its own memory allocator (jemalloc by default) for efficient allocation.
- **Persistence Layer**: Optional disk writes for RDB snapshots (point-in-time dumps) or AOF logs (append-only file for every write).
- **Replication**: Master-slave async replication for high availability.
- **Clustering**: Sharding across nodes using hash slots (16384 slots distributed among masters).
- **Event Loop**: Based on libevent or epoll for handling I/O multiplexing.
- **Command Protocol**: RESP (Redis Serialization Protocol), a simple TCP-based protocol supporting binary-safe strings.

### Single-Threaded Model
Redis processes commands sequentially in a single thread to avoid locks and ensure atomicity. This makes it fast but can bottleneck on CPU-bound operations (e.g., large sorts). Multi-threading was introduced in Redis 6.0 for I/O (e.g., accepting connections) but core execution remains single-threaded.

### Memory Management
- **Eviction Policies**: When memory is full (controlled by `maxmemory`), Redis evicts keys based on policies like LRU (Least Recently Used), LFU (Least Frequently Used), or random. Configurable via `maxmemory-policy`.
- **Key Expiration**: Keys can have TTL (Time To Live) for automatic eviction.
- **Memory Optimization**: Use compact structures; avoid large keys (>10KB recommended limit to prevent blocking).

### Deployment Modes
- **Standalone**: Single instance.
- **Sentinel**: For high availability with automatic failover.
- **Cluster**: For sharding and scaling out.
- **Redis Enterprise**: Adds CRDB (Conflict-free Replicated Data Types) for active-active replication.

Example architecture for a high-scale app: Clients connect via a pool, read from replicas, write to master; cluster shards data; persistence to disk; monitoring with Prometheus.

## Installation and Setup

### Installing Redis Open Source
Redis can be installed on Linux, macOS, Windows (via WSL or Docker), and other platforms.

#### On Ubuntu/Debian
```bash
sudo apt update
sudo apt install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```
Test: `redis-cli ping` (should return "PONG").

#### From Source (Linux/macOS)
```bash
wget https://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
sudo make install
```
Build with modules: `make BUILD_WITH_MODULES=yes`.

#### Using Docker
```bash
docker run --name my-redis -p 6379:6379 redis:latest
```
For Redis Stack (with modules): `docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest`.

#### On Windows
Use WSL2 or Docker; official support is experimental.

#### Configuration File (redis.conf)
Key parameters:
- `bind 127.0.0.1` (bind to localhost for security).
- `port 6379`.
- `requirepass yourpassword` (enable authentication).
- `maxmemory 2gb` (limit RAM).
- `maxmemory-policy allkeys-lru`.
- `save 900 1` (RDB persistence: snapshot every 900s if 1 key changed).
- `appendonly yes` (enable AOF).

Reload config: `redis-cli CONFIG REWRITE`.

### Getting Started with Redis Cloud (Free Tier)
As mentioned in the doc, the quickest way is Redis Enterprise Cloud's free tier:
1. Sign up at [redis.io/cloud](https://redis.io/cloud).
2. Create a free database (up to 30MB, with one extra module like JSON or Search).
3. Get connection details (host, port, password).
4. Attach modules as needed.

This provides a managed instance without local setup.

### Client Tools
- **Redis CLI**: Command-line interface.
  Example: `redis-cli -h host -p 6379 -a password`.
  Basic commands: `SET key value`, `GET key`, `DEL key`.
- **RedisInsight**: GUI tool for browsing data, monitoring, and CRUD operations. Download from [redis.io/redisinsight](https://redis.io/redisinsight). Supports interactive dashboards, query builders, and visualization.
  Installation: Available for Windows, macOS, Linux, Docker.
- **VS Code Extension**: Redis extension for querying and managing from IDE.

### Connecting from Applications
Use client libraries:
- **Python (redis-py)**: `pip install redis`
  ```python
  import redis
  r = redis.Redis(host='localhost', port=6379, password='password')
  r.set('key', 'value')
  print(r.get('key'))  # b'value'
  ```
- **Node.js (node-redis)**: `npm install redis`
  ```javascript
  const redis = require('redis');
  const client = redis.createClient({ url: 'redis://localhost:6379' });
  await client.connect();
  await client.set('key', 'value');
  console.log(await client.get('key'));  // 'value'
  ```
- **Java (Jedis)**: `Maven dependency: redis.clients:jedis`
  ```java
  import redis.clients.jedis.Jedis;
  Jedis jedis = new Jedis("localhost", 6379);
  jedis.auth("password");
  jedis.set("key", "value");
  System.out.println(jedis.get("key"));  // "value"
  ```
- Other languages: Go (go-redis), .NET (StackExchange.Redis), PHP (Predis), etc. See [redis.io/docs/clients](https://redis.io/docs/clients/) for more.

Connection best practices: Use connection pooling, handle reconnections, set timeouts.

## Data Types and Structures

Redis supports rich data types beyond simple strings, allowing natural data modeling. All are key-based, with atomic operations.

### 1. Strings
Binary-safe strings (up to 512MB). Used for caching, counters, sessions.
- **Commands**:
  - `SET key value [EX seconds] [PX milliseconds] [NX|XX]` : Set key-value, optional expiry or conditions.
    Example: `SET user:1 "John Doe" EX 3600` (expires in 1 hour).
  - `GET key` : Retrieve value.
  - `DEL key` : Delete.
  - `INCR key` : Atomic increment (for counters).
    Example: `INCR visits` (increments by 1).
  - `APPEND key value` : Append to string.
  - `STRLEN key` : Length.
  - Bit operations: `SETBIT key offset value`, `BITCOUNT key`.
- Use cases: Cache HTML fragments, rate limiting (e.g., `INCR ip:192.168.1.1` and expire).
- Example in Python:
  ```python
  r.set('counter', 0)
  r.incr('counter')  # Now 1
  ```

### 2. Lists
Ordered collections of strings (doubly-linked lists). Ideal for queues, stacks.
- **Commands**:
  - `LPUSH key value [value ...]` / `RPUSH` : Push to left/right.
  - `LPOP key` / `RPOP` : Pop from left/right.
  - `LRANGE key start stop` : Get range.
    Example: `LPUSH mylist "item1"`, `RPUSH mylist "item2"`, `LRANGE mylist 0 -1` → ["item1", "item2"].
  - `BLPOP key [key ...] timeout` : Blocking pop for queues.
- Use cases: Task queues (e.g., job processing), recent items list.
- Limits: Up to 2^32 elements, but keep <10k for performance.

### 3. Sets
Unordered collections of unique strings. Set operations like union/intersection.
- **Commands**:
  - `SADD key member [member ...]` : Add members.
  - `SMEMBERS key` : Get all.
  - `SINTER key1 key2` : Intersection.
    Example: `SADD fruits "apple" "banana"`, `SADD veggies "carrot" "banana"`, `SINTER fruits veggies` → ["banana"].
  - `SCARD key` : Cardinality (size).
- Use cases: Tags, unique visitors, friends lists (intersection for mutual friends).

### 4. Sorted Sets (ZSets)
Sets with scores for ordering. Like priority queues.
- **Commands**:
  - `ZADD key score member [score member ...]` : Add with score.
  - `ZRANGE key start stop [WITHSCORES]` : Get range by index.
  - `ZREVRANK key member` : Reverse rank.
    Example: `ZADD leaderboard 100 "player1" 200 "player2"`, `ZRANGE leaderboard 0 -1 WITHSCORES` → ["player1", "100", "player2", "200"].
- Use cases: Leaderboards, rate limiting (score as timestamp).

### 5. Hashes
Maps between string fields and values. Like objects/dicts.
- **Commands**:
  - `HSET key field value` : Set field.
  - `HGET key field` : Get field.
  - `HGETALL key` : All fields/values.
    Example: `HSET user:1 name "John" age 30`, `HGETALL user:1` → {name: "John", age: "30"}.
  - `HINCRBY key field increment` : Increment field.
- Use cases: Storing user profiles, shopping carts. More memory-efficient than JSON for flat objects.

### 6. Bitmaps
Strings interpreted as bit arrays. For probabilistic structures.
- **Commands**:
  - `SETBIT key offset value` : Set bit.
  - `BITCOUNT key [start end]` : Count set bits.
    Example: Track user logins: `SETBIT users:week1 123 1` (user 123 logged in), `BITCOUNT users:week1 0 1000` → active users.
- Use cases: Bloom filters alternative, analytics (unique visitors).

### 7. HyperLogLogs
Probabilistic data structure for estimating cardinality (unique count).
- **Commands**:
  - `PFADD key element [element ...]` : Add.
  - `PFCOUNT key [key ...]` : Estimate count.
    Example: `PFADD pageviews:page1 "user1" "user2"`, `PFCOUNT pageviews:page1` → ~2 (with 0.81% error).
- Use cases: Unique visitor counting, low-memory sets.

### 8. Streams
Append-only logs for message streaming. Like Kafka lite.
- **Commands**:
  - `XADD key ID field value` : Append entry (ID auto-generated if *).
  - `XREAD [COUNT count] [BLOCK ms] STREAMS key ID` : Read entries.
  - `XGROUP CREATE key groupname $` : Create consumer group.
    Example: `XADD mystream * sensor "temp:25"`, `XREAD STREAMS mystream 0` → Reads entry.
- Use cases: Event sourcing, logs, chat messages. Supports consumer groups for pub/sub-like patterns.

### 9. Geospatial Indexes
Sorted sets with lat/long scores. For location-based queries.
- **Commands**:
  - `GEOADD key longitude latitude member` : Add point.
  - `GEORADIUS key longitude latitude radius m|km|ft|mi` : Find nearby.
    Example: `GEOADD Sicily 15.0877 37.5025 "Palermo"`, `GEORADIUS Sicily 15 37 200 km` → Nearby cities.
- Use cases: Location services, ride-sharing.

### 10. JSON (via RedisJSON Module)
Store and query JSON documents.
- **Commands** (prefixed with JSON.):
  - `JSON.SET key path $.value` : Set JSON.
  - `JSON.GET key path` : Get.
  - `JSON.ARRAPPEND key path value` : Append to array.
    Example: `JSON.SET doc $ '{"name": "John"}'`, `JSON.GET doc $` → {"name": "John"}.
- Use cases: Document store, hierarchical data.

### 11. TimeSeries (via RedisTimeSeries Module)
For time-stamped data.
- **Commands** (TS. prefixed):
  - `TS.ADD key timestamp value [label ...]` : Add sample.
  - `TS.RANGE key from to` : Query range.
    Example: `TS.ADD temp:1 1640995200 23.5`, `TS.RANGE temp:1 - +` → Timestamp-value pairs.
- Use cases: IoT sensor data, stock prices, metrics.

Advanced: Use modules for more (see below). Always index large datasets for efficiency.

## Commands and Usage

Redis has over 200 commands, grouped by type. Use `redis-cli --scan` to list keys, `HELP @string` for group help.

### Basic CRUD
- `EXISTS key` : Check existence.
- `EXPIRE key seconds` : Set TTL.
- `TTL key` : Remaining time.
- `FLUSHDB` / `FLUSHALL` : Clear current/all DBs (use cautiously).

### Transactions
Atomic bundles with `MULTI` / `EXEC`.
Example:
```
MULTI
SET a 1
INCR a
EXEC
```
Supports `WATCH` for optimistic locking (CAS - Check And Set).

### Lua Scripting
Server-side scripts for complex logic, reducing round-trips.
- `EVAL script numkeys key [key ...] arg [arg ...]`
Example (SHA for reuse):
```lua
-- Increment if <10
if redis.call('GET', KEYS[1]) == false then
  redis.call('SET', KEYS[1], 1)
else
  redis.call('INCR', KEYS[1])
end
```
`EVAL` script 1 mykey.

### Pub/Sub
For messaging.
- `PUBLISH channel message` : Send.
- `SUBSCRIBE channel` : Listen.
- `PSUBSCRIBE pattern` : Pattern subscribe.
Example: Publisher: `PUBLISH news "Breaking!"`; Subscriber: `SUBSCRIBE news` (receives message).

### Sorting and Pagination
`SORT key [BY pattern] [LIMIT offset count] [GET pattern] [ASC|DESC] [ALPHA]`.
Example: Sort users by score: `SORT users BY score:* LIMIT 0 10`.

### Full Command Reference
Refer to [redis.io/commands](https://redis.io/commands/) for exhaustive list with syntax, complexity (e.g., O(1) for GET), and examples. Time complexities: Most are O(1), scans/sorts O(N).

## Persistence and Durability

Redis is in-memory but durable via:
### RDB (Snapshotting)
Point-in-time dumps to disk. Config: `save <seconds> <changes>` (e.g., `save 60 1000`).
- `BGSAVE` : Background save.
- Pros: Fast, compact. Cons: Potential data loss since last snapshot.
- Use for backups.

### AOF (Append-Only File)
Logs every write. Config: `appendonly yes`, `appendfsync everysec` (1s durability).
- `BGREWRITEAOF` : Rewrite for compactness.
- Pros: Durable. Cons: Larger files, slower.
- Best: Use both (RDB for backups, AOF for recovery).

Hybrid: Redis loads AOF on start, falls back to RDB. For max durability: `appendfsync always` (but slower).

Example config:
```
appendonly yes
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

## Replication and High Availability

### Replication
Async master-slave.
- Slave: `SLAVEOF master_host master_port`.
- `WAIT numreplicas timeout` : Sync wait.
- Use: Read scaling, failover.

### Sentinel
Monitors masters, auto-failover.
Setup: Multiple Sentinels, config with `sentinel monitor mymaster ip port quorum`.
Commands: `SENTINEL get-master-addr-by-name mymaster`.

### Clustering
Sharding via 16384 hash slots.
- `CLUSTER NODES` : View cluster.
- Enable: `cluster-enabled yes`.
- Client: Use hash tags `{tag}` for consistent sharding.
Pros: Auto-sharding, fault-tolerant. Cons: Multi-key ops limited (e.g., no transactions across slots).

For active-active: Redis Enterprise CRDB.

## Modules and Multi-Model Features

Redis is extensible via modules (dynamic libraries). Load with `loadmodule /path/to/module.so`.

### Redis Stack
Bundled distribution with core + modules. Install via Docker or packages. Includes JSON, Search, Graph, TimeSeries.

### Key Modules
1. **RedisJSON (ReJSON)**: JSON document store with path queries.
   - Commands: JSON.SET, JSON.GET, JSON.ARRINDEX, etc.
   - Example: Hierarchical data, querying `JSON.GET doc $.users[0].name`.
   - Use: Document-oriented DB.

2. **RediSearch**: Full-text search, vector similarity.
   - Create index: `FT.CREATE idx SCHEMA title TEXT body TEXT`.
   - Query: `FT.SEARCH idx "redis"`.
   - Vector: `FT.CREATE idx SCHEMA embedding VECTOR HNSW 6 DIM 128`.
   - Use: Search engines, semantic search for AI (e.g., KNN for vectors in RAG).
   - Example for bicycles (from doc):
     ```javascript
     // Schema
     const schema = {
       '$.brand': { type: 'TEXT', AS: 'brand' },
       // ...
     };
     await client.ft.create('idx:bicycle', schema, { ON: 'JSON', PREFIX: 'bicycle:' });
     // Search
     await client.ft.search('idx:bicycle', '@brand:"Noka Bikes"');
     ```

3. **RedisGraph**: Graph database with Cypher queries.
   - `GRAPH.QUERY db "CREATE (a:Person {name:'Alice'})"`.
   - Queries: MATCH, RETURN.
   - Use: Relationships, recommendations (e.g., social graphs).

4. **RedisTimeSeries**: As above, for time-series.

5. **RedisBloom**: Probabilistic filters (Bloom, Cuckoo).
   - `BF.ADD filter item` : Add to Bloom filter.
   - Use: Deduplication, existence checks without false negatives.

6. **RedisGears**: Server-side scripting with Python-like.
   - Execute functions on data changes.
   - Use: Real-time processing.

7. **Other**: RedisAI (ML models), Redis Raft (consensus).

Modules turn Redis into a vector DB for AI, search engine, etc. Attach one free in Redis Cloud.

## Performance Tuning and Best Practices

### Performance Optimization
- **Memory Tuning**:
  - Set `maxmemory` to 80-90% of available RAM.
  - Policy: `allkeys-lru` for caches, `volatile-lru` if using TTLs.
  - Use `MEMORY USAGE key` to check size.
  - Avoid large keys: Split >10KB values.
  - Enable `lazyfree-lazy-eviction yes` for non-blocking evictions.

- **Persistence Tuning**:
  - For speed: RDB only, no AOF.
  - For durability: AOF everysec + RDB.
  - RDB on slaves only to avoid master blocking.
  - Keep instance <10GB to speed fork() for snapshots.

- **Network/IO**:
  - Bind to private IP: `bind 127.0.0.1`.
  - Increase `tcp-keepalive 300`.
  - Use pipelining for batch commands (reduces RTT).
  - Client-side: Connection pooling (e.g., 10-50 connections).

- **CPU/Threading**:
  - Disable Transparent Huge Pages (THP): `echo never > /sys/kernel/mm/transparent_hugepage/enabled` (slows fork).
  - For multi-core: Use I/O threading (`io-threads 4` in Redis 6+).
  - Avoid slow ops like `KEYS *` (use `SCAN` instead).

- **Benchmarking**:
  - Use `redis-benchmark` : `redis-benchmark -t set,get -n 100000 -q`.
  - Monitor: `INFO` command (e.g., `used_memory`, `instantaneous_ops_per_sec`).
  - Tools: RedisInsight, Prometheus exporter.

Example tuning for high perf:
```
maxmemory 4gb
maxmemory-policy allkeys-lru
save ""
appendonly no  # If cache-only
tcp-backlog 511
timeout 0
```

### Best Practices
- **Data Modeling**: Choose right structure (e.g., hashes over strings for objects). Use prefixes for organization (e.g., `user:123:session`).
- **Key Naming**: Descriptive, consistent (e.g., `{user:123}:posts` for hashing).
- **Eviction and Expiry**: Always set TTL for caches to prevent memory bloat.
- **Batch Operations**: Pipeline commands; use Lua for multi-step.
- **Scaling**: Replicas for reads; cluster for >1 node.
- **Monitoring**: Track latency (`SLOWLOG GET`), memory, connections. Integrate Grafana.
- **Testing**: Load test with tools like LoadForge; simulate failures.
- **General**: Keep values small; use SCAN/HSCAN over KEYS/SMEMBERS.

From sources: For virtualized envs, ensure low-latency disks; avoid over-provisioning.

## Security

Redis defaults to no auth, so secure it!

### Best Practices
- **Authentication**: Set `requirepass strongpassword` (20+ chars, complex).
- **ACLs** (Redis 6+): User-based permissions.
  Example: `ACL SETUSER alice on >password ~keys:* +get +set`.
  - `ACL LIST`, `ACL WHOAMI`.
- **Encryption**: Use TLS (`tls-port 6380`, certs). Stunnel for older versions.
- **Network**: Bind to localhost (`bind 127.0.0.1`); use firewalls (e.g., ufw allow from specific IPs). Disable dangerous commands: `rename-command FLUSHALL ""`.
- **Protected Mode**: Enabled by default (rejects non-local non-auth connections).
- **Avoid Exposures**: Don't run as root; use least privilege. Disable THP as above.
- **Common Threats**:
  - Brute-force: Strong pass + firewall.
  - Injection: Sanitize inputs (RESP safe).
  - DoS: Limit connections (`maxclients 10000`), monitor slow ops.
  - Eavesdropping: TLS for prod.
- **Enterprise**: RBAC, audit logs.

Example config:
```
requirepass yourstrongpass123!
rename-command CONFIG ""
protected-mode yes
```
Monitor logs for unauthorized access.

## Use Cases and Examples

### As Cache
Accelerate DB: Cache user profiles.
Example: App queries Redis first; miss → DB → set with TTL.

### As Primary DB
Store sessions, configs. Reduce complexity—no separate cache.

### Real-Time Analytics
HyperLogLog for UVs; Streams for logs.
Example: `PFADD daily_uvs "user123"`, aggregate daily.

### Leaderboards
Sorted sets: `ZADD scores 100 "user1"`, `ZREVRANGE 0 9 WITHSCORES`.

### Session Store
Hashes: `HSET session:abc123 user_id 1 expires 3600`, `EXPIRE session:abc123 3600`.

### Message Broker
Pub/Sub for chat; Streams for reliable queuing.

### Document Store
With JSON: Store/query user docs.

### Graph DB
With RedisGraph: `GRAPH.QUERY friends "MATCH (a:Person)-[:FRIEND]->(b) RETURN a,b"`.

### Time-Series
IoT: `TS.ADD sensor:temp * 25.5 label device_id 123`.

### Vector Search for AI
RediSearch: Embeddings for similarity search in RAG.
Example: Index vectors, query `FT.SEARCH idx "@vector:[KNN 10 @embedding $vec]"`.

### Full Example: Simple Web App Cache (Node.js)
```javascript
const express = require('express');
const redis = require('redis');
const app = express();
const client = redis.createClient();

app.get('/user/:id', async (req, res) => {
  const { id } = req.params;
  let user = await client.get(`user:${id}`);
  if (!user) {
    // Simulate DB fetch
    user = JSON.stringify({ id, name: 'John' });
    await client.set(`user:${id}`, user, { EX: 300 });
  }
  res.json(JSON.parse(user));
});

app.listen(3000);
```

## Advanced Topics

### Lua Scripting in Depth
Custom functions: Register with `SCRIPT LOAD`, call by SHA. Reduces latency for complex ops.

### Client-Side Caching (Redis 6.2+)
`CLIENT CACHING yes` on connection; offloads caching to client.

### Module Development
Write in C; load dynamically. See GitHub examples.

### Integration with Other Tools
- **Prometheus/Grafana**: Exporter for metrics.
- **Kafka/Streams**: Use Streams as alternative.
- **ELK Stack**: Log to Redis, process with Logstash.

### Migration
Use RIOT tool: `riot-redis --read redis://source --write redis://target`.

## Redis Enterprise and Cloud

Redis Enterprise extends open-source with:
- **CRDB**: Multi-master replication.
- **Active-Active**: Geo-distribution.
- **Vector DB**: Advanced AI features.
- **Cloud**: Managed on AWS/Azure/GCP. Free tier: 30MB DB +1 module.
- Differences: Enterprise has support, more modules, compliance (GDPR, HIPAA).
- Pricing: See [x.ai/grok](https://x.ai/grok) wait, no—redirect to [redis.io/pricing](https://redis.io/pricing) for details.
- API: For xAI API, see [x.ai/api](https://x.ai/api), but Redis has its own.

Start with free tier as per doc.

## Monitoring and Troubleshooting

- **Commands**: `INFO` (sections: server, memory, stats), `SLOWLOG GET 10` (slow queries), `MONITOR` (live commands—careful, resource-heavy).
- **Tools**: RedisInsight (dashboards, slow log viewer), `redis-cli --latency` for history.
- **Common Issues**:
  - OOM: Tune eviction.
  - High Latency: Check fork time (RDB), THP, network.
  - Connection Issues: Pool properly, increase backlog.
- **Logs**: `loglevel notice`, check /var/log/redis/redis.log.

## Conclusion

Redis is a powerhouse for modern applications, from simple caching to full-fledged multi-model databases. By leveraging its in-memory speed, rich structures, and modules, you can build scalable, low-latency systems. Experiment with the free cloud tier, CLI, and Insight to get hands-on. For production, focus on security, tuning, and monitoring. As the doc asks: Yes, I'd use Redis as primary for speed-focused apps—its simplicity reduces complexity at scale. For complex relations, combine with modules like Graph or JSON. Dive deeper via official docs and community.