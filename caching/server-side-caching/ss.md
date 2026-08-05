# Server-side caching

Caching stores a copy of data somewhere faster to access than the original source, so repeat requests skip the expensive path (a database query, a slow computation, a network call). **Server-side caching** specifically means that copy lives on the server (or a dedicated cache server) rather than on the client — see [`../clientside/client-side-caching.md`](../clientside/client-side-caching.md) for the browser/client-memory side of this, and [`../redis/redis.md`](../redis/redis.md) for the mechanics of the most common tool used to implement it.

## TL;DR
- Server-side caching stores data or rendered output on the origin server (or a dedicated cache layer) so future requests don't repeat expensive work.
- The three core update patterns are **cache-aside**, **write-through**, and **write-behind** — each with a different consistency/latency trade-off.
- Invalidation is the hard part: TTL expiry, explicit invalidation, and event-based invalidation are the three main strategies, often combined.
- At scale, caching becomes **distributed** — data partitioned across multiple cache nodes via consistent hashing, with its own replication and fault-tolerance concerns.
- This file covers strategy in tool-agnostic terms; for Redis-specific commands and configuration, see [`../redis/redis.md`](../redis/redis.md).

## Server-side vs. client-side caching

| | Server-side caching | Client-side caching |
|---|---|---|
| Storage location | Origin server or a dedicated cache server (Redis, Memcached) | Browser, mobile app, or client-side app memory |
| Managed by | Backend/infra team, centrally | Each individual client |
| Consistency | Easier — one place to invalidate | Harder — every client needs its own invalidation logic |
| Typical use | Shared cache across many clients/requests | Per-user local speed, offline access |

## How server-side caching works

1. A request comes in for a resource (a webpage, an API response, a query result).
2. On the **first** request, the server does the real work (renders the page, runs the query) and stores a copy in the cache.
3. On **subsequent** requests, the server serves directly from the cache instead of redoing the work.

### Types of server-side caching
- **Object caching** — caches database query results or application-level objects (the pattern this file focuses on).
- **Opcode caching** — compiles scripts (e.g., PHP) into bytecode once, skipping recompilation on every request (e.g., OPcache).
- **CDN caching** — geographically distributed edge servers caching content close to users; see [`../cdn/cdn.md`](../cdn/cdn.md) for the full picture.

### Drawbacks to keep in mind
- **Latency still exists** depending on where the cache server sits relative to the requester.
- **Stale data** is the central risk — anything cached needs an invalidation strategy for when the underlying data changes.

## 🟢 Beginner: the three core caching patterns

These patterns answer one question: **when data changes, how does the cache find out?** Get this wrong and you either serve stale data (bad) or pay a full round-trip on every read (defeats the purpose of caching).

### Cache-aside (lazy loading)

The application is responsible for checking the cache, and populates it only on a miss. This is by far the most common pattern — it's usually what people mean by "we cache with Redis" without further qualification.

**How it works:**
1. App checks the cache for the key.
2. **Hit** → return the cached value directly.
3. **Miss** → app queries the database, then writes the result into the cache (usually with a TTL) before returning it.

```python
def get_product(product_id):
    cached = cache.get(f"product:{product_id}")
    if cached is not None:
        return cached                      # cache hit

    product = db.query("SELECT * FROM products WHERE id = %s", product_id)  # cache miss
    cache.set(f"product:{product_id}", product, ttl=600)
    return product
```

- **Pros**: only actually-requested data ever gets cached (no wasted memory); if the cache is down, the app degrades to hitting the database directly rather than failing outright.
- **Cons**: the very first request for any key pays the full miss cost; data can drift stale between writes and the next natural cache refresh.

### Write-through

Every write goes to the cache and the database together, as one synchronous operation. Reads always see current data, because the cache is never allowed to fall behind.

```python
def update_product(product_id, data):
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    cache.set(f"product:{product_id}", data, ttl=600)   # updated immediately, same request
```

- **Pros**: cache and database stay in sync — no read ever sees stale data (assuming this is the *only* write path).
- **Cons**: every write now costs two operations instead of one, adding latency to writes to save latency on reads. Only worth it when reads vastly outnumber writes.

### Write-behind (write-back)

Writes land in the cache immediately and are pushed to the database **asynchronously** — batched, queued, or flushed on a timer — rather than as part of the same request.

```python
def update_product_fast(product_id, data):
    cache.set(f"product:{product_id}", data)
    write_queue.enqueue({"table": "products", "id": product_id, "data": data})  # flushed later
```

- **Pros**: writes are very fast (the database round-trip is off the critical path); multiple writes to the same key can be coalesced into fewer database operations.
- **Cons**: the most dangerous pattern — if the cache crashes before a queued write reaches the database, that write is lost. Needs a durable queue and reconciliation logic. Reserve this for data where losing the last few milliseconds of writes is tolerable (view counters, activity logs), never for anything like financial state.

| Pattern | Read path | Write path | Staleness risk | When to use |
|---|---|---|---|---|
| Cache-aside | Check cache, fall back to DB on miss | App writes DB; cache updated separately/lazily | Between write and next cache refresh | Default choice for most read-heavy caching |
| Write-through | Always hits a current cache | App writes both cache and DB synchronously | Low, if this is the only write path | Read-heavy, consistency-sensitive data |
| Write-behind | Always hits cache | App writes cache only; DB updated async | Real risk of loss if cache crashes before flush | High write throughput, tolerant of rare data loss |

For Redis-specific implementations of all three (including the exact commands and a distributed-lock pattern for cache-aside), see the [caching patterns section in redis.md](../redis/redis.md#caching-patterns).

## 🟡 Intermediate: cache invalidation strategies

"There are only two hard things in computer science: cache invalidation and naming things." Three main strategies, often combined:

### 1. TTL-based (time-to-live) expiration
Every cached entry gets a lifespan; after it expires, the next request is treated as a miss and refetches fresh data.
```python
cache.set(f"product:{product_id}", product, ttl=600)   # stale after 10 minutes, automatically
```
- **Pros**: simple, self-healing — no explicit invalidation logic needed anywhere else in the codebase.
- **Cons**: data can be stale for up to the full TTL window; picking the right TTL is a genuine trade-off (too short = low hit rate, too long = stale data lingers).

### 2. Explicit invalidation
The application actively deletes or overwrites a cache entry the moment the underlying data changes, instead of waiting for a TTL.
```python
def update_product(product_id, data):
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    cache.delete(f"product:{product_id}")   # force the next read to be a miss and refresh
```
- **Pros**: cache is stale for the shortest possible window — only until the next read.
- **Cons**: requires every single write path to remember to invalidate; easy to miss a code path (a batch job, an admin panel, a different service) and end up with silently stale data.

### 3. Event-based invalidation
Instead of the write path invalidating the cache directly, a change publishes an event (via a message queue, database change stream, or pub/sub) and one or more subscribers invalidate the relevant cache entries.
```python
# Publisher (e.g., on a DB change-data-capture stream, or after a write)
event_bus.publish("product.updated", {"product_id": product_id})

# Subscriber (could be a different service than the one that wrote the data)
def on_product_updated(event):
    cache.delete(f"product:{event['product_id']}")
```
- **Pros**: decouples cache invalidation from every individual write path — works even when multiple services/processes can modify the same underlying data.
- **Cons**: more moving infrastructure (an event bus/queue), and invalidation now happens with some lag (however small) rather than in the same request as the write.

### Other invalidation techniques worth knowing
- **Versioned keys** — cache under a key that includes a version number (`product:123:v7`); an update bumps the version, so old cache entries are simply never looked up again (they age out via TTL rather than needing active deletion).
- **Cache tags** — group related cache entries under a tag so one invalidation call can clear all of them (e.g., invalidate every cached page referencing "category:electronics" after that category changes).

## 🟡 Intermediate: cache strategy in the backend

### Estimating cache size
A practical rule of thumb: the 80/20 rule — roughly 20% of data serves 80% of requests, so you often don't need to cache everything.

Example: 10M users, 500KB of cacheable data per user → caching the "hot" 20% needs roughly:
```
10,000,000 users × 500KB × 20% ≈ 1,000,000,000 KB ≈ ~100GB total cache size
```

### Cache replacement (eviction) policies
When the cache is full, something has to be evicted to make room:
- **LRU (Least Recently Used)** — evict whatever hasn't been touched in the longest time. Most common default; matches "recently accessed data is likely to be accessed again."
- **LFU (Least Frequently Used)** — evict whatever has been accessed the fewest times overall, regardless of recency.
- **FIFO (First In, First Out)** — evict the oldest entry by insertion time, ignoring access patterns entirely. Simple, sometimes suboptimal.

(Redis supports all of these plus TTL-aware variants — see the [eviction policies table in redis.md](../redis/redis.md#eviction-policies).)

## 🔴 Advanced: distributed caching

A single cache server eventually becomes a bottleneck or a single point of failure. **Distributed caching** spreads cached data across multiple cache nodes for scalability, availability, and lower latency to geographically spread users.

### Key concepts
- **Data partitioning** — deciding which node holds which keys:
  - **Consistent hashing** — maps keys to nodes such that adding/removing a node only reshuffles a small fraction of keys, instead of nearly all of them (as naive `hash(key) % N` would).
  - **Virtual nodes** — each physical node is represented by many points on the hash ring, smoothing out load distribution even when nodes have different capacities.
- **Replication strategies**:
  - **Master-slave** — one node accepts writes, others replicate from it and serve reads.
  - **Peer-to-peer** — nodes are equal, any node can accept reads/writes, with data synced between them.

### Benefits
- Reduced latency (data served from a nearby node).
- Horizontal scalability (add nodes as load grows).
- Fault tolerance (one node failing doesn't take down the whole cache).

### Example
An online store with a global user base runs Redis clusters across continents. A user in Asia gets served from a local Asia-region node instead of round-tripping to a US-based cache, cutting latency substantially.

### Popular distributed caching solutions

| Solution | Features |
|---|---|
| **Redis** | In-memory, rich data structures, built-in clustering — see [redis.md](../redis/redis.md) |
| **Memcached** | Lightweight, simple key-value only, multi-threaded — see the [Redis vs. Memcached comparison](../redis/redis.md#redis-vs-memcached) |
| **Hazelcast** | Distributed in-memory data grid with clustering support |
| **Apache Ignite** | Supports ACID transactions, SQL queries, and persistence on top of a distributed cache |

## Best practices

- Always set a **TTL** on cached entries — even a long one — so nothing lingers forever by accident.
- Monitor **cache hit/miss rates** — a low hit rate usually means the caching layer isn't earning its complexity.
- Avoid over-caching highly dynamic content — data that changes on every request gains little from caching and adds staleness risk for no benefit.
- Handle **cache invalidation** deliberately — pick a strategy (TTL, explicit, event-based, or a combination) rather than leaving it implicit.
- Choose the caching **pattern** (cache-aside, write-through, write-behind) based on your actual read/write ratio and tolerance for staleness — don't default to the most complex option.

## Further reading
- [`../redis/redis.md`](../redis/redis.md) — Redis-specific caching patterns, eviction policies, and cache stampede mitigations
- [`../clientside/client-side-caching.md`](../clientside/client-side-caching.md) — caching on the client instead of the server
- [`../cdn/cdn.md`](../cdn/cdn.md) — edge/CDN caching for static and semi-static content
