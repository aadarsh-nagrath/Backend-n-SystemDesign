# Client-side caching

Client-side caching stores frequently accessed data in the memory of the client — a browser, mobile app, or even an application server acting as a client to another service — instead of repeatedly hitting a remote database or API. Because local memory access is orders of magnitude faster than a network round-trip, this is one of the cheapest wins available for latency and server load.

## TL;DR
- **Cache hit** = data found locally, no network call. **Cache miss** = not found, fetch and (usually) store it.
- Browsers cache via HTTP headers (`Cache-Control`, `ETag`, `Last-Modified`), `localStorage`/`sessionStorage` for app data, and Service Workers for offline-capable PWAs.
- Application servers acting as clients to Redis/another store can cache locally too — Redis's **client-side caching (Tracking)** feature (v6+) supports this with server-assisted invalidation.
- Main risk is **stale data**: the client's copy diverges from the source of truth. Mitigate with TTLs, server-assisted invalidation, or versioned keys.
- Best for data that's read often and changes rarely (user profiles, static assets) — not for live counters or anything requiring strict real-time accuracy.

## What is client-side caching?

Client-side caching means storing a copy of server responses — database query results, API responses, static assets — locally on the client so repeat requests don't need to leave the client at all.

**Key concepts:**
- **Local cache** — a portion of client memory (RAM, browser cache, `localStorage`) used to store data.
- **Cache hit** — requested data is found in the local cache, avoiding a server round-trip.
- **Cache miss** — data isn't cached; the client must fetch it from the server (and typically caches it for next time).
- **Cache invalidation** — removing or updating stale entries when the source data changes.
- **TTL (Time-to-Live)** — how long a cached entry stays valid before it's treated as stale.

**Without client-side caching:**
```plaintext
Application -> Database: GET user:1234
Database -> Application: username = Alice
```
Every request pays a full network round-trip.

**With client-side caching:**
```plaintext
Application (Local Cache: user:1234 = Alice) -> No database request
```
The application serves straight from memory.

### Client-side vs. server-side caching

| | Client-side caching | Server-side caching |
|---|---|---|
| Storage location | Client memory (browser, app server, mobile device) | A server (Redis, Memcached) |
| Latency | Lowest — nanoseconds, in-process | Low — still a network hop, but centralized |
| Scalability | Limited by each client's own resources | Centralized, scales independently |
| Consistency | Harder — every client needs its own invalidation | Easier — one place to invalidate |
| Best for | Offline access, absolute-lowest-latency reads | Shared cache across many clients |

See [`../server-side-caching/ss.md`](../server-side-caching/ss.md) for the server-side patterns (cache-aside, write-through, write-behind) and [`../redis/redis.md`](../redis/redis.md) for Redis specifics.

## A brief history

1. **1990s** — early browsers introduce caching for static assets via `Cache-Control` and `Expires` headers.
2. **2000s** — application servers start caching database query results in-process (PHP, Java).
3. **2010s** — Redis and Node.js popularize sophisticated client-side caching with real invalidation protocols.
4. **2020s** — Redis Tracking mode and browser Service Workers push client-side caching toward real-time, distributed use cases.

## How it works

### 1. Data retrieval
The client requests data, decides whether to cache the response (based on policy — TTL, access frequency), and stores it locally (in-memory map, browser cache, etc.).

### 2. Cache access
Subsequent requests check the local cache first: hit → serve locally; miss → fetch from server, cache, then serve.

### 3. Cache invalidation
- **TTL-based** — entry expires after a fixed duration.
- **Server-assisted** — the server actively notifies clients when data changes (e.g., Redis Tracking, WebSocket push).
- **Manual** — the client explicitly purges (e.g., on logout).
- **Versioning** — cache under a versioned key (`user:1234:v2`) so an update naturally produces a cache miss on the old key.

### 4. Cache management
- **Eviction policy** — LRU, FIFO, LFU, or random, to bound memory use.
- **TTL handling** — respect server-provided TTLs or apply client-side defaults.
- **Memory limits** — cap total cache size to avoid exhausting client memory.

## 🟢 Beginner: browser caching fundamentals

### HTTP cache headers

The browser's HTTP cache is controlled almost entirely by response headers from the server.

**`Cache-Control`** — the primary directive, replacing the older `Expires` header:
```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600
Content-Type: image/jpeg
```
Common directives:

| Directive | Meaning |
|---|---|
| `max-age=<seconds>` | How long the response is fresh, from time of request |
| `no-cache` | Can be cached, but must revalidate with the server before each use |
| `no-store` | Never cache this response at all (e.g., sensitive data) |
| `private` | Only the browser may cache it (not shared/proxy caches) |
| `public` | May be cached by any cache, including CDNs and proxies |
| `immutable` | Content will never change during its freshness window — skip revalidation entirely |
| `must-revalidate` | Once stale, must be revalidated before use — no serving stale-while-erroring |

**`ETag` and `Last-Modified`** — validators used for *revalidation* once a cached response goes stale:
```http
# First response
HTTP/1.1 200 OK
ETag: "33a64df551"
Cache-Control: max-age=3600

# After max-age expires, browser revalidates:
GET /style.css
If-None-Match: "33a64df551"

# If unchanged, server replies with no body:
HTTP/1.1 304 Not Modified
```
`ETag` is a content hash/fingerprint (more precise); `Last-Modified` + `If-Modified-Since` is date-based (cheaper to compute, coarser granularity). A `304 Not Modified` response saves the bandwidth of re-sending the body while still confirming freshness.

**Practical pattern**: fingerprint static assets in the filename (`app.a3f9c2.js`) and set `Cache-Control: max-age=31536000, immutable` — since the URL changes whenever content changes, you get both aggressive caching and instant invalidation on deploy.

### `localStorage` and `sessionStorage`

Browser APIs for storing small amounts of string data client-side, distinct from the HTTP cache.

| | `localStorage` | `sessionStorage` |
|---|---|---|
| Lifetime | Persists until explicitly cleared | Cleared when the tab/window closes |
| Scope | Shared across all tabs for the same origin | Isolated per tab |
| Size limit | ~5-10MB (browser-dependent) | ~5-10MB |
| Typical use | User preferences, theme, cached API responses, auth tokens (with caveats) | Multi-step form state, per-tab session data |

```javascript
// Storing and reading
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');
localStorage.removeItem('theme');

// Only strings — objects need serialization
localStorage.setItem('user', JSON.stringify({ id: 1, name: 'Alice' }));
const user = JSON.parse(localStorage.getItem('user'));

sessionStorage.setItem('formStep', '2'); // gone when the tab closes
```

⚠️ Both are synchronous and block the main thread — avoid storing large payloads. Neither is encrypted; don't store secrets (session tokens in `localStorage` are readable by any injected script, i.e. vulnerable to XSS — an `HttpOnly` cookie is safer for auth tokens).

## 🟡 Intermediate: caching strategies and structures

### Cache types
- **In-memory cache** — plain data structure in RAM (Node.js `Map`, Java `HashMap`). Fast, but gone on process restart.
- **Browser HTTP cache** — governed by `Cache-Control`/`ETag`, used automatically by the browser for requests it recognizes as cacheable.
- **Service Worker cache** — the `Cache` API, giving JS full control over what's cached and when, enabling offline-first PWAs.
- **`localStorage`/`sessionStorage`** — small structured data, not subject to HTTP caching rules.

### Eviction policies
- **LRU (Least Recently Used)** — evict the entry that hasn't been accessed in the longest time. Most common default.
- **FIFO (First-In-First-Out)** — evict the oldest entry regardless of access pattern. Simple, sometimes suboptimal.
- **LFU (Least Frequently Used)** — evict the entry accessed least often.
- **Random eviction** — cheapest to implement, no bookkeeping.

### Service Worker offline caching

```javascript
// service-worker.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('my-cache').then((cache) => {
      return cache.addAll(['/index.html', '/styles.css', '/script.js']);
    })
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      return response || fetch(event.request).then((networkResponse) => {
        caches.open('my-cache').then((cache) => {
          cache.put(event.request, networkResponse.clone());
        });
        return networkResponse;
      });
    })
  );
});
```
This is a **cache-first** strategy: serve from cache if present, otherwise hit the network and populate the cache for next time. Other common Service Worker strategies: **network-first** (try network, fall back to cache — good for content that should be fresh when possible) and **stale-while-revalidate** (serve cached immediately, refresh in the background for next time).

### Application-server-side client cache (Python example)

An application server that itself talks to Redis can keep a local LRU cache in front of it to skip network calls for hot keys entirely:

```python
from collections import OrderedDict
import redis

class ClientCache:
    def __init__(self, max_size=1000):
        self.cache = OrderedDict()  # acts as an LRU cache
        self.max_size = max_size
        self.redis = redis.Redis(host='localhost', port=6379)

    def get(self, key):
        if key in self.cache:
            self.cache.move_to_end(key)  # refresh LRU order
            return self.cache[key]
        value = self.redis.get(key)
        if value:
            value = value.decode('utf-8')
            self._add_to_cache(key, value)
            return value
        return None

    def _add_to_cache(self, key, value):
        if len(self.cache) >= self.max_size:
            self.cache.popitem(last=False)  # evict oldest
        self.cache[key] = value

cache = ClientCache()
user_data = cache.get('user:1234')  # Redis on miss, memory on hit
```

## 🔴 Advanced: Redis client-side caching (Tracking)

Redis 6+ ships a **Tracking** feature purpose-built for client-side caching: the server tracks which keys each client has read and pushes invalidation messages when those keys change, so clients don't need to guess when to refresh.

### Tracking modes

- **Default mode** — the server maintains an **invalidation table** mapping tracked keys to the clients that read them, and sends targeted `INVALIDATE` messages when a key changes.
  - Pros: precise invalidation, minimal wasted messages.
  - Cons: consumes server memory proportional to the number of tracked keys.
- **Broadcasting mode** — clients subscribe to key *prefixes* (e.g., `user:`) and get invalidation messages for any matching key, without the server tracking individual client-key pairs.
  - Pros: zero server memory overhead for tracking.
  - Cons: more invalidation messages sent overall (any client watching the prefix gets notified, even for keys it never read), higher bandwidth.

```plaintext
# Default mode
CLIENT TRACKING ON
GET user:1234
# server now tracks user:1234 for this client

# Broadcasting mode
CLIENT TRACKING ON BCAST PREFIX user:
# client gets invalidations for any key starting with "user:"
```

### Invalidation in practice

```plaintext
SET user:1234 "Flora"
# Default mode  -> Server -> Client: INVALIDATE "user:1234"
# Broadcasting   -> Server -> Clients: INVALIDATE ["user:1234"]
```

### Connection models

Redis needs a channel to push invalidation messages, separate from the normal request/response flow:

- **Single connection (RESP3)** — multiplexes data and invalidation push messages over one connection.
  ```plaintext
  HELLO 3
  CLIENT TRACKING ON
  GET user:1234
  # server may push: INVALIDATE "user:1234"
  ```
- **Two connections (RESP2/RESP3)** — a dedicated Pub/Sub connection carries invalidations while a separate connection handles data:
  ```plaintext
  # Invalidation connection
  CLIENT ID
  :4
  SUBSCRIBE __redis__:invalidate
  # Data connection
  CLIENT TRACKING ON REDIRECT 4
  GET user:1234
  ```

### Advanced options

- **OPTIN** — clients explicitly opt individual keys into caching with `CLIENT CACHING YES` before the read, rather than caching everything by default.
  ```plaintext
  CLIENT TRACKING ON OPTIN
  CLIENT CACHING YES
  GET user:1234
  ```
- **OPTOUT** — the inverse: everything is cached unless explicitly excluded with `CLIENT UNTRACKING`.
- **NOLOOP** — suppresses invalidation messages for keys the client itself just modified (you already know your own write happened).
  ```plaintext
  CLIENT TRACKING ON NOLOOP
  SET user:1234 "Flora"  # no invalidation echoed back to this client
  ```
- **PREFIX** — declares which key prefixes to watch, used with broadcasting mode.

### Node.js implementation

```javascript
const redis = require('redis');
const clientData = redis.createClient({ url: 'redis://localhost:6379' });
const clientInv = redis.createClient({ url: 'redis://localhost:6379' });

(async () => {
  await clientInv.connect();
  await clientData.connect();

  const invId = await clientInv.clientId();
  const localCache = new Map();

  // Dedicated invalidation connection
  await clientInv.subscribe('__redis__:invalidate', (message) => {
    localCache.delete(JSON.parse(message)[0]);
  });

  // Data connection, redirected to push invalidations to the sub connection
  await clientData.clientTracking({ on: true, redirect: invId });

  const value = await clientData.get('user:1234');
  localCache.set('user:1234', value);
  // When another client runs SET user:1234 "Flora", localCache auto-evicts the key
})();
```

### Memory management

- **Server-side**: the invalidation table has a max size and evicts old entries under pressure; broadcasting mode uses no server memory for tracking.
- **Client-side**: cap cache size with an eviction policy (LRU is typical) to avoid unbounded growth.

## Advantages

- **Reduced latency** — local memory access (nanoseconds) vs. network round-trip (milliseconds). Browser caching alone can cut page load from ~500ms to ~10ms for repeat visits.
- **Lower server load** — fewer requests reach the database or API.
- **Scalability** — the same backend serves more clients since a chunk of demand never leaves the client.
- **Cost efficiency** — reduced bandwidth and compute for the origin.
- **Offline support** — Service Worker caching lets PWAs function without a network connection.

## Disadvantages / gotchas

⚠️ **Stale data risk** — without proper invalidation, clients keep serving outdated data (e.g., an old username after a profile update).

⚠️ **Memory pressure** — caching consumes client memory, which is genuinely limited on mobile devices.

⚠️ **Invalidation complexity** — server-assisted invalidation (Redis Tracking, WebSocket push) requires extra infrastructure and adds server-side CPU/memory cost.

⚠️ **Race conditions** — an invalidation message can arrive before the corresponding data does, or a stale value can be cached *after* an invalidation already fired for it. Mitigate with placeholders (a "caching-in-progress" flag) or by relying on the single-connection RESP3 model, which orders data and invalidation messages more predictably.

⚠️ **Implementation complexity** — correctly handling TTLs, evictions, and race conditions is real engineering work, not a drop-in.

## Use cases

- **Web apps** — caching user profiles, session data, or static assets in-process or in the browser.
- **Mobile apps** — caching API responses (often in SQLite or an in-memory store) for offline access.
- **PWAs** — Service Worker caching for offline-first experiences (e.g., a news app showing cached articles without a connection).
- **E-commerce** — caching product details/images to cut database load.
- **Real-time apps** — caching recent messages in a chat app for instant scrollback.

## Best practices

1. Pick an eviction policy that matches access patterns — LRU for most cases, FIFO if simplicity matters more than hit rate.
2. Always set a TTL, even a generous one, so nothing caches forever by accident.
3. Cap cache size explicitly (e.g., 1GB ceiling for an app server) to avoid memory exhaustion.
4. Prefer server-assisted invalidation (Redis Tracking, WebSocket) over short TTLs when precise freshness matters.
5. Monitor hit/miss ratios — a cache with a low hit rate is often not worth the complexity.
6. Cache what's frequently read and infrequently written — user profiles, not live counters.
7. Test cache consistency deliberately: simulate an update and confirm stale data isn't served.

## Client-side caching in different contexts

| Context | Technologies | Typical use |
|---|---|---|
| Web browsers | HTTP cache, Service Workers, `localStorage`/`sessionStorage` | Static assets, offline PWA content |
| Application servers | In-memory maps (`HashMap`, `Map`), Redis Tracking | Caching results from a shared backing store |
| Mobile apps | SQLite, in-memory caches | Offline API response access |
| Desktop apps | File-based caches, in-memory stores | Config data, user preferences, local game state |

## Further reading
- [Redis client-side caching (Tracking) docs](https://redis.io/docs/latest/develop/reference/client-side-caching/)
- MDN: [HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- MDN: [Using the Cache API (Service Workers)](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
