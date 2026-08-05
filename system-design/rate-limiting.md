# Rate Limiting

Rate limiting caps how many requests a client (an IP, a user, an API key, a service) can make in a given window of time. It's the mechanism that keeps one noisy client from taking down a service for everyone else, keeps costs predictable, and enforces the terms of an API's usage tiers.

## TL;DR
- Rate limiting protects against abuse, enforces fair usage across clients, shields downstream services (databases, third-party APIs) from overload, and bounds infrastructure cost.
- Five core algorithms, each with a real trade-off: **Token Bucket** (allows bursts, industry default), **Leaky Bucket** (smooths output to a constant rate), **Fixed Window Counter** (simple, but bursts at window boundaries), **Sliding Window Log** (precise, memory-expensive), **Sliding Window Counter** (good approximation, cheap — the practical default).
- Enforce it at multiple layers: client-side (courtesy), API gateway (first real line of defense), application-level (business-logic-aware limits), and database-level (last resort, protects the data layer directly).
- The hard part is **distributed rate limiting** — coordinating limit state across many app server instances, usually via Redis with atomic operations (`INCR`+`EXPIRE`, or a Lua script for compound atomicity).
- HTTP has standard semantics for this: `429 Too Many Requests`, `Retry-After`, and the (unofficial but near-universal) `X-RateLimit-*` headers.
- Rate limit by an identity that's actually meaningful (user ID, API key) — IP-based limiting breaks down behind NAT and corporate proxies where thousands of legitimate users share one IP.

## What it solves

- **Abuse prevention**: stops brute-force login attempts, credential stuffing, scraping, and denial-of-service style traffic from a single bad actor.
- **Fair usage**: in a multi-tenant system, one client hammering the API shouldn't degrade service for everyone else sharing the same infrastructure.
- **Protecting downstream services**: your API might be a thin layer over a database, a third-party payment processor, or an ML inference endpoint — all of which have their own capacity limits that a traffic spike upstream will blow through if nothing caps it first.
- **Cost control**: every request often has a real dollar cost (compute, third-party API calls, LLM inference). Uncapped usage is an uncapped bill.
- **SLA/tier enforcement**: "free tier gets 100 requests/day, paid tier gets 100,000" is a product decision implemented as a rate limit.

## Algorithms

### Token bucket

A bucket holds up to `capacity` tokens. Tokens refill at a fixed rate (e.g., 10/second). Each request consumes one token; if the bucket is empty, the request is rejected (or queued, depending on implementation). Because tokens accumulate while idle, this naturally allows **bursts** up to the bucket's capacity, then throttles to the steady refill rate.

This is the most common algorithm in production systems (AWS API Gateway, Stripe, and many others use variants of it) because it matches real traffic patterns well — most clients are bursty, not perfectly smooth.

```python
import time
import threading

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity: max tokens the bucket can hold (max burst size)
        refill_rate: tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.monotonic()
        self.lock = threading.Lock()

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

    def allow_request(self, cost: int = 1) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= cost:
                self.tokens -= cost
                return True
            return False

# 100 requests/sec sustained, bursts up to 500
limiter = TokenBucket(capacity=500, refill_rate=100)

if limiter.allow_request():
    handle_request()
else:
    return_429()
```

- **Pros**: allows natural bursts, smooth to reason about, cheap (one float + one timestamp per client).
- **Cons**: a client that's been idle can suddenly fire a full burst — if that's undesirable, leaky bucket is the better fit.

### Leaky bucket

Conceptually a queue with a hole in the bottom: requests enter the bucket (queue) and leak out — get processed — at a strictly constant rate. If the bucket (queue) is full when a request arrives, it's dropped. Unlike token bucket, leaky bucket **smooths bursts into a constant output rate** rather than allowing them through.

```python
import time
import threading
from collections import deque

class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        """
        capacity: max queued requests
        leak_rate: requests processed per second (constant output rate)
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.monotonic()
        self.lock = threading.Lock()

    def _leak(self):
        now = time.monotonic()
        elapsed = now - self.last_leak
        leaked = int(elapsed * self.leak_rate)
        for _ in range(min(leaked, len(self.queue))):
            self.queue.popleft()
        if leaked > 0:
            self.last_leak = now

    def allow_request(self) -> bool:
        with self.lock:
            self._leak()
            if len(self.queue) < self.capacity:
                self.queue.append(time.monotonic())
                return True
            return False
```

- **Pros**: output rate to downstream systems is perfectly smooth — ideal when the thing you're protecting (a legacy database, a strict third-party API) truly cannot handle bursts at all.
- **Cons**: a legitimate burst of traffic gets queued/delayed or dropped even if the server *could* have handled it instantly — less friendly to bursty-but-fine traffic than token bucket.

**Token bucket vs leaky bucket, the intuition**: token bucket controls how much can go out in a burst (rate of *permission*); leaky bucket controls how fast things actually flow out (rate of *processing*). Token bucket is about admission control; leaky bucket is closer to traffic shaping.

### Fixed window counter

Divide time into fixed windows (e.g., 00:00-00:59, 01:00-01:59) and keep a counter per window. Increment on each request; reject once the counter exceeds the limit; reset the counter when the window rolls over.

```python
# Redis-backed fixed window
def is_allowed(redis_client, user_id: str, limit: int, window_seconds: int) -> bool:
    window = int(time.time() // window_seconds)
    key = f"rl:{user_id}:{window}"
    count = redis_client.incr(key)
    if count == 1:
        redis_client.expire(key, window_seconds)
    return count <= limit
```

- **Pros**: trivial to implement, minimal memory (one counter per client per window).
- **Cons**: the **boundary burst problem** — a client can send `limit` requests in the last second of one window and another `limit` requests in the first second of the next window, getting `2 × limit` requests through in a ~2-second span despite a per-window cap. This is the textbook flaw interviewers probe for.

### Sliding window log

Store a timestamp for every request in a sorted structure (e.g., a Redis sorted set). On each new request, drop timestamps older than `now - window`, then check if the remaining count is under the limit.

```python
def is_allowed(redis_client, user_id: str, limit: int, window_seconds: int) -> bool:
    now = time.time()
    key = f"rl:log:{user_id}"
    pipe = redis_client.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_seconds)  # drop expired entries
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)                                       # count remaining
    pipe.expire(key, window_seconds)
    _, _, count, _ = pipe.execute()
    return count <= limit
```

- **Pros**: perfectly precise — no boundary burst issue, exact sliding window semantics.
- **Cons**: memory scales with request volume within the window (storing every timestamp), which gets expensive at high request rates or with many clients.

### Sliding window counter

A practical approximation of the sliding window log that avoids storing every timestamp. Keep two fixed-window counters (current and previous), and compute a weighted estimate of the sliding window count by assuming requests in the previous window were evenly distributed:

```
estimated_count = current_window_count
                 + previous_window_count * (overlap_fraction_with_sliding_window)
```

```python
import time

def is_allowed(redis_client, user_id: str, limit: int, window_seconds: int) -> bool:
    now = time.time()
    current_window = int(now // window_seconds)
    previous_window = current_window - 1

    curr_key = f"rl:{user_id}:{current_window}"
    prev_key = f"rl:{user_id}:{previous_window}"

    pipe = redis_client.pipeline()
    pipe.get(prev_key)
    pipe.get(curr_key)
    prev_count, curr_count = pipe.execute()

    prev_count = int(prev_count or 0)
    curr_count = int(curr_count or 0)

    # Fraction of the current window that has NOT yet elapsed —
    # this is how much of the previous window's traffic still "counts"
    # against a true sliding window right now.
    elapsed_in_current = now % window_seconds
    weight = (window_seconds - elapsed_in_current) / window_seconds

    estimated = curr_count + prev_count * weight

    if estimated >= limit:
        return False

    pipe = redis_client.pipeline()
    pipe.incr(curr_key)
    pipe.expire(curr_key, window_seconds * 2)
    pipe.execute()
    return True
```

- **Pros**: smooths out the fixed-window boundary burst problem to a good approximation, constant memory per client (two counters, not a log of timestamps) — this is why it's the practical default for high-traffic systems (Cloudflare and others document using this exact approach).
- **Cons**: still an approximation (assumes uniform distribution within the previous window, which isn't always true) — not as precise as sliding window log, but close enough for virtually all real use cases at a fraction of the cost.

### Comparison

| Algorithm | Precision | Memory cost | Allows bursts? | Complexity |
|---|---|---|---|---|
| Token Bucket | Good | Low (2 values/client) | Yes, up to bucket capacity | Low |
| Leaky Bucket | Good | Low-medium (queue size) | No — smooths to constant rate | Medium |
| Fixed Window Counter | Poor (boundary bug) | Very low (1 value/client) | Yes, unintentionally at boundaries | Very low |
| Sliding Window Log | Exact | High (1 entry per request) | No | Medium |
| Sliding Window Counter | Very good approximation | Low (2 values/client) | Slightly, bounded | Medium |

**Interview framing**: if asked "how would you rate limit an API," Token Bucket (simple, burst-friendly, matches most real traffic) and Sliding Window Counter (precise enough, cheap, avoids the fixed-window boundary bug) are the two answers that show you understand the trade-offs — which is why both got full code above.

## Where to enforce it

Rate limiting isn't a single chokepoint decision — different layers protect against different failure modes, and defense in depth is normal.

| Layer | Protects against | Notes |
|---|---|---|
| **Client-side** | Nothing malicious — this is courtesy/UX (e.g., debouncing a search box, respecting `Retry-After`) | Never trust it; a malicious client just won't implement it |
| **API gateway / edge** (Cloudflare, Kong, AWS API Gateway, Nginx) | Volumetric abuse, DDoS-scale traffic, before it reaches app servers | Cheapest place to reject — fails fast, saves app server CPU |
| **Application-level** | Business-logic-aware limits (e.g., "5 password reset emails per hour," "3 free trial signups per IP per day") — needs app context the gateway doesn't have | Where per-user-tier logic usually lives |
| **Database-level** | Last resort — connection limits, query rate caps, statement timeouts | Protects the data layer even if every layer above it fails; not really "rate limiting" in the request sense but the same principle |

A well-designed system layers these: the gateway drops obviously abusive volume cheaply at the edge, the application enforces nuanced per-user/per-endpoint business rules, and the database has hard limits as a backstop that should rarely if ever actually trigger.

### Distributed rate limiting — the hard part

The moment you have more than one application server instance, a naive in-memory counter (`self.count += 1` in each process) is wrong: each instance only sees its own slice of traffic, so a client can get `limit × num_instances` requests through by hitting different instances. Rate limiting state has to be **shared** across all instances enforcing the same limit.

**The standard solution: Redis as shared, atomic counter storage.**

The simplest version — `INCR` + `EXPIRE` — has a subtle race condition if done as two separate commands:

```python
# NAIVE — has a race condition
count = redis_client.incr(key)
if count == 1:
    redis_client.expire(key, window_seconds)  # <- if the process crashes/is slow
                                                #    between INCR and EXPIRE, key never expires
```

Two fixes, in increasing order of robustness:

**1. `SET key 0 EX window NX` first, then `INCR`** — ensures the expiry is always set atomically with key creation:

```python
def is_allowed(redis_client, key: str, limit: int, window_seconds: int) -> bool:
    redis_client.set(key, 0, ex=window_seconds, nx=True)  # no-op if key already exists
    count = redis_client.incr(key)
    return count <= limit
```

**2. A Lua script — fully atomic, the production-grade answer.** Redis executes Lua scripts atomically (single-threaded, no other command can interleave), so this eliminates every race condition in one round trip:

```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call("INCR", key)
if current == 1 then
    redis.call("EXPIRE", key, window)
end

if current > limit then
    return 0  -- rejected
else
    return 1  -- allowed
end
```

```python
# Load once, call cheaply many times (EVALSHA)
script = redis_client.register_script(open("rate_limit.lua").read())

def is_allowed(user_id: str, limit: int = 100, window_seconds: int = 60) -> bool:
    result = script(keys=[f"rl:{user_id}"], args=[limit, window_seconds])
    return result == 1
```

This pattern generalizes to token bucket and sliding window counter too — anywhere you need "read state, compute, conditionally write" to happen without another request interleaving, push it into a Lua script rather than doing multiple round trips from the app.

**Other distributed approaches**, for context:
- **Centralized rate-limiting service**: a dedicated microservice (backed by Redis or an in-memory store with consistent hashing to shard clients across nodes) that all app servers call before processing a request — used at very large scale (this is roughly what Envoy's `ratelimit` service and similar sidecar-adjacent services do).
- **Approximate/local + sync**: each instance keeps a local counter and periodically syncs with a central store, trading strict accuracy for lower latency (no synchronous call to Redis on every request) — acceptable when being off by a small margin occasionally is fine.
- **Sticky routing**: route all of a given client's requests to the same app server (e.g., via consistent hashing at the load balancer — see [load-balancing.md](./load-balancing.md)) so a local in-memory limiter is accurate without needing shared state at all. Works, but couples rate limiting to routing decisions and reintroduces the stickiness trade-offs discussed there.

## HTTP semantics

- **`429 Too Many Requests`** — the correct status code (RFC 6585) when a client has been rate limited. Not `403 Forbidden` (that implies a permissions issue, not a temporal one) and not `503 Service Unavailable` (that implies the server itself is overloaded, not that this specific client exceeded a quota).
- **`Retry-After`** — tells the client how long to wait before retrying, either as seconds or an HTTP date:
  ```http
  HTTP/1.1 429 Too Many Requests
  Retry-After: 30
  Content-Type: application/json

  {"error": "rate_limit_exceeded", "message": "Too many requests, retry after 30 seconds"}
  ```
- **`X-RateLimit-*` headers** — not standardized in an official RFC (there's a draft, `RateLimit-Limit`/`RateLimit-Remaining`/`RateLimit-Reset`, that's gaining adoption) but the `X-RateLimit-*` form is a de facto industry standard used by GitHub, Twitter/X, and many others:
  ```http
  HTTP/1.1 200 OK
  X-RateLimit-Limit: 5000
  X-RateLimit-Remaining: 4987
  X-RateLimit-Reset: 1735689600
  ```
  - `X-RateLimit-Limit`: total requests allowed in the current window.
  - `X-RateLimit-Remaining`: how many are left.
  - `X-RateLimit-Reset`: when the window resets (usually a Unix timestamp).

Well-behaved clients read these headers and back off proactively instead of hammering the API until they hit a 429 — good API design surfaces this information on *every* response, not just the one that finally gets rejected.

## 🟢 Beginner

- Rate limiting = "cap how many requests X can make in time period Y."
- The simplest possible version is a counter per client that resets every fixed window (e.g., every minute) — understand this first, then learn why it has a flaw (see Fixed Window Counter above) before reaching for anything fancier.
- Always return `429` with a `Retry-After` header, never silently drop the request or return a generic `403`/`500` — the client needs to know *why* and *when to try again*.

## 🟡 Intermediate

- Know all five algorithms (token bucket, leaky bucket, fixed window, sliding window log, sliding window counter) and can explain the specific flaw fixed window has and how sliding window counter approximates a fix cheaply.
- Understand *where* to enforce limits (gateway vs app vs DB) and why doing it only at one layer is insufficient — the gateway can't apply nuanced per-user business rules, and the application layer alone won't survive a genuine volumetric attack without an edge layer absorbing it first.
- Know that in-memory counters don't work once you have multiple app instances — this is the tell that separates "understands rate limiting" from "actually understands *distributed* rate limiting" in an interview.
- Nginx built-in rate limiting (token-bucket-like, using the "leaky bucket" terminology in its own docs but behaving with burst allowance):

```nginx
http {
    # 10 requests/sec per client IP, tracked in a 10MB shared memory zone
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_req_status 429;
            proxy_pass http://backend;
        }
    }
}
```
  `burst=20` allows a client to exceed the steady 10r/s rate by up to 20 queued requests; `nodelay` serves burst requests immediately instead of artificially delaying them to smooth the rate (i.e., closer to token-bucket-style burst admission than strict leaky-bucket smoothing).

- HAProxy equivalent using a stick table:

```
frontend http_front
    bind *:80
    stick-table type ip size 100k expire 60s store http_req_rate(60s)

    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }

    default_backend api_backend
```
  This tracks request rate per source IP over a 60-second window and rejects with 429 once a client exceeds 100 requests in that window.

## 🔴 Advanced

- **Per-endpoint, per-tier limits**: different endpoints have wildly different costs (a `GET /users/:id` vs a `POST /reports/generate` that kicks off a heavy job) — a single global limit per client is usually wrong. Most production systems key limits by `(client, endpoint or endpoint-class)` and set the limit based on actual backend cost, not a flat number.
- **Cost-weighted rate limiting**: instead of every request costing "1 unit," assign different costs per operation (GitHub's GraphQL API does this — a query's "points" cost depends on its complexity/breadth, not just that it's one HTTP request). Token bucket generalizes to this cleanly — pass `cost` as the number of tokens consumed.
- **Adaptive/dynamic limits**: adjust limits in real time based on overall system load rather than a fixed number — e.g., tighten limits automatically when backend latency or error rate crosses a threshold, loosen them when the system has headroom. This is closer to load-shedding than classic rate limiting but the two are often implemented together.
- **Multi-region distributed limiting**: if app servers span multiple regions and share a global limit (not per-region), a single Redis instance becomes a cross-region latency bottleneck and a single point of failure. Options: accept eventual consistency with regional Redis instances that async-replicate/aggregate counts (looser but low-latency), or accept the latency cost of a global data store for the sake of strict accuracy, or set per-region sub-limits that sum to the global budget (simple, but wastes budget if traffic is unevenly distributed across regions).
- **Rate limiting and idempotency interact**: a client that gets a `429` and retries must not accidentally double-submit a non-idempotent operation — pairs naturally with the idempotency key pattern (see [restapi.md](../api/restapi.md#idempotency-keys-safe-retries-for-non-idempotent-operations)) so a retried request that finally gets through doesn't create a duplicate side effect.
- **Distinguishing global vs local overload**: a 429 from the gateway (you've been rate limited) and a 503 from the backend (the whole service is overloaded) look similar to a naive client but call for different retry strategies — a well-designed client backs off much more aggressively on 503 (something is actually broken) than on a routine 429 (you're just over your personal quota).

## Common pitfalls

- ⚠️ **Race conditions in naive counter implementations.** `GET` then `SET` (read-modify-write without atomicity) across two Redis round trips lets concurrent requests both read the same "under limit" value and both proceed, silently allowing more traffic through than the limit — always use `INCR` (atomic in Redis) or a Lua script for compound operations, never a manual read-then-write.
- ⚠️ **Clock skew across distributed nodes.** Algorithms with time-window boundaries (fixed window, sliding window) depend on consistent clocks. If app servers' clocks drift relative to each other (or relative to Redis, if computing windows client-side instead of using Redis's own `TIME` command), window boundaries shift inconsistently across nodes — usually a minor effect at typical NTP-synced drift levels, but worth being aware of, and it's why letting Redis (a single source of truth) own the "current window" computation server-side (as in the Lua script above) is safer than trusting each app server's local clock.
- ⚠️ **Rate limiting by IP breaks behind NAT/corporate proxies.** Thousands of employees behind one corporate NAT, or many mobile users behind a carrier-grade NAT, all share one public IP — an IP-based limit punishes all of them collectively for one bad actor's traffic, or conversely lets one determined attacker rotate through IPs (via a botnet, proxy pool, or just switching networks) to evade the limit entirely. Prefer limiting by an authenticated identity (user ID, API key, OAuth client ID) whenever the client is authenticated; fall back to IP only for unauthenticated endpoints (login, signup) where there's no identity yet, and consider combining IP + other signals (device fingerprint, request pattern) there.
- ⚠️ **Forgetting the fixed-window boundary burst.** If precision actually matters (e.g., protecting a fragile downstream system with a hard capacity ceiling), fixed window's "2x limit at the boundary" flaw is a real production bug, not a theoretical one — use sliding window counter or token bucket instead.
- ⚠️ **No differentiation between "you're rate limited" and "the service is down."** Returning `503` for rate limiting (or `429` for a genuine outage) sends the wrong signal to clients about how to react and how aggressively to back off.
- ⚠️ **Limits that don't scale with legitimate growth.** A hardcoded global limit that made sense at launch can start rejecting legitimate traffic as usage grows — rate limits need the same capacity-planning attention as any other infrastructure number, not a "set once and forget."
- ⚠️ **Trusting client-supplied identifiers for rate limiting.** If a rate limit key is derived from something the client controls and can freely change (a client-generated session ID with no auth behind it, an easily-spoofed header), it's trivial to evade — always tie limits to something the server verifies (authenticated user ID, API key validated server-side, or IP as a last resort).

## Real-world examples

- **GitHub REST API**: 5,000 requests/hour for authenticated users (higher for GitHub Apps/Enterprise), exposed via `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers on every response; GraphQL API uses a cost-weighted "points" system instead of flat per-request counting.
- **Stripe API**: enforces both a sustained rate and burst limit (token-bucket-like) per API key, returns `429` with a `Retry-After` header, and explicitly documents exponential backoff as the expected client behavior.
- **Cloudflare**: offers rate limiting as an edge product (blocking at the CDN/WAF layer before traffic ever reaches origin servers) and has published engineering blog posts detailing their use of the sliding window counter approximation at massive scale for exactly the memory-cost reasons described above.
- **Twitter/X API**: historically used both per-endpoint and per-app rate limits with `X-Rate-Limit-*` headers, a widely copied convention.
- **AWS API Gateway**: supports both account-level throttling and per-client usage plans (token-bucket-based: steady-state rate + burst capacity), configurable per API key.

## Further reading

- RFC 6585 — [Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585) (defines 429)
- IETF draft — [RateLimit header fields for HTTP](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/) (the emerging standardized alternative to `X-RateLimit-*`)
- Cloudflare blog — [How we built rate limiting capable of scaling to millions of domains](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/) (sliding window counter in production)
- Stripe docs — [Rate limits](https://docs.stripe.com/rate-limits)
- GitHub docs — [Rate limits for the REST API](https://docs.github.com/en/rest/using-the-api/rate-limits-for-the-rest-api)
- Redis docs — [EVAL/Lua scripting](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/) and [rate limiting pattern](https://redis.io/glossary/rate-limiting/)
- [load-balancing.md](./load-balancing.md) — sticky routing as an alternative to shared distributed state
- [restapi.md](../api/restapi.md) — idempotency keys, relevant when clients retry after a 429
