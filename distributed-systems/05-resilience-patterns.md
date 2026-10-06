# Resilience Patterns: Building Systems That Survive Failure

> [`system-design/notes/concepts.md`](../system-design/notes/concepts.md) introduces circuit breakers, timeouts, retries, and backpressure. This note goes deeper into how failures cascade and how each pattern stops them, including the subtle ways the patterns themselves cause outages.

## Table of Contents
1. [How Distributed Systems Fail: Cascades and Metastability](#cascades)
2. [Timeouts and Deadlines](#timeouts)
3. [Retries Done Right](#retries)
4. [Circuit Breakers](#circuit)
5. [Bulkheads](#bulkheads)
6. [Load Shedding and Admission Control](#shedding)
7. [Backpressure](#backpressure)
8. [Graceful Degradation and Fallbacks](#degradation)
9. [Hedged and Tied Requests (Tail Latency)](#hedging)
10. [Caching for Resilience (and Its Traps)](#caching)
11. [Health Checks, Self-Healing, and Restarts](#health)
12. [Redundancy, Cells, and Blast Radius](#cells)
13. [Static Stability](#static)
14. [Chaos Engineering and Game Days](#chaos)
15. [Libraries and Tools](#tools)
16. [Interview Questions](#qa)

---

## 1. How Systems Fail {#cascades}

**Cascading failure**: a failure in one part increases load or latency elsewhere, which then fails too.
Typical chain:
```
DB slows (bad query / failover) → app threads wait on DB → thread pool exhausted
→ app stops responding (even health checks) → LB marks instances unhealthy → remaining instances get more load
→ they exhaust too → clients retry (3×) → load quadruples → total outage
→ DB recovers but the retry storm + cold caches keep it down
```
Contributing factors: no timeouts or overly long ones, unbounded retries, shared thread or connection pools, synchronous call chains, health checks coupled to dependencies, autoscaling lag, and cold caches after restarts.

**Metastable failures** (Bronson et al., HotOS 2021): the system enters a bad state that **persists even after the trigger is removed**, sustained by a feedback loop (usually retries or cache misses). Examples: retry amplification keeping a service overloaded, a cache that's empty after a flush making every request hit the DB (which is too slow to repopulate the cache), and GC death spirals. **Recovery requires breaking the loop** (shedding load, disabling retries, warming caches gradually), not just fixing the trigger.

**Work amplification**: one user request → 10 backend calls → each retried 3× at 3 layers = 3³ = 27× load per call path during failures.

---

## 2. Timeouts and Deadlines {#timeouts}

- **Every network call needs a timeout**: connect, read, and overall. Defaults are often infinite (many HTTP clients, JDBC socket reads).
- Choose them from data: slightly above the dependency's p99.9 under normal load, and well below the caller's own timeout.
- **Deadline propagation**: pass the *remaining* time budget downstream (gRPC deadlines propagate automatically; for HTTP, a header like `X-Request-Deadline`). If the client gives up at 2 s, downstream services shouldn't keep working for 30 s on a request nobody will read. Cancel work when the deadline passes (`context.Context` in Go, `CancellationToken` in .NET, `AbortSignal` in JS).
- **Timeout budgets nest**: client 3 s > API gateway 2.8 s > service A 2.5 s > its DB call 1 s.
- Too-short timeouts cause false failures and retries. Too-long ones cause resource exhaustion.

---

## 3. Retries Done Right {#retries}

Retries fix transient failures and **amplify** persistent ones. Rules:
1. **Retry only retryable errors**: timeouts, connection resets, 502/503/504, 429 (honoring `Retry-After`), deadlocks, serialization failures. **Never** retry 4xx validation errors.
2. **Only retry idempotent operations**, or use idempotency keys (otherwise you get double charges).
3. **Exponential backoff with jitter**:
   ```
   sleep = random_between(0, min(cap, base * 2 ** attempt))      # "full jitter" (AWS Architecture Blog)
   ```
   Jitter de-synchronizes clients so they don't retry in waves (a thundering herd).
4. **Cap attempts** (2–3) and total time.
5. **Retry at one layer only**, typically the outermost or the closest to the failure, not at every layer (avoids multiplicative amplification).
6. **Retry budgets**: allow retries only up to e.g. 10–20% of normal request volume (Envoy `retry_budget`, Finagle, gRPC retry throttling). When the dependency is broadly failing, retries stop automatically.
7. Respect server signals: `Retry-After`, gRPC `RetryInfo`, and "don't retry" pushback.
8. Log and monitor retry rates. A rising retry rate is an early warning.

---

## 4. Circuit Breakers {#circuit}

Like an electrical breaker: stop calling a failing dependency for a while so it can recover, and fail fast instead of waiting on timeouts.

States:
```
CLOSED ──(failure rate > threshold over window, e.g. 50% of last 100 calls, min 20 calls)──► OPEN
OPEN   ──(after wait duration, e.g. 30 s)──► HALF-OPEN
HALF-OPEN ──(N trial calls succeed)──► CLOSED
HALF-OPEN ──(trial fails)──► OPEN
```
- In OPEN state, calls fail immediately (or return a fallback) without touching the dependency.
- Count **slow calls** as failures too (latency-based tripping; Resilience4j `slowCallRateThreshold`).
- Scope: per dependency, and possibly per endpoint or per host (one bad host shouldn't open the breaker for the whole service; use outlier detection at the LB level).
- Combine with fallbacks (§8) and alerting on state changes.
- Pitfalls: a breaker opened on shared infrastructure errors causing broader denial, tuning thresholds too sensitive (flapping) or too lax, and in-process breakers on many instances each discovering failure separately (that's acceptable).

---

## 5. Bulkheads {#bulkheads}

Ships have watertight compartments so one breach doesn't sink the vessel. In software, **isolate resources** so one failing dependency or tenant can't exhaust everything:
- **Separate thread or connection pools per dependency**: if the recommendations service hangs, only its pool of 20 threads is blocked, and checkout calls keep working (Hystrix's original model; semaphore bulkheads in Resilience4j).
- Separate DB connection pools for critical vs background work.
- Separate worker queues per job class/priority.
- **Per-tenant limits** (rate limits, concurrency caps, quotas) against noisy neighbors.
- Separate deployments or clusters for critical paths (checkout cluster vs browsing cluster).
- **Cell-based architecture** (§12) is a bulkhead at the infrastructure level.

---

## 6. Load Shedding and Admission Control {#shedding}

When demand exceeds capacity, **reject some requests early and cheaply** rather than letting everything slow down (where every request fails because it times out). Serving 80% of requests well beats serving 100% badly.
- Reject at the edge with `503` + `Retry-After` (or `429` per client) **before** doing expensive work.
- Signals: in-flight request count, queue length or wait time, CPU, latency vs target (adaptive concurrency limits: Netflix `concurrency-limits`, Envoy adaptive concurrency).
- **Prioritize**: shed low-priority traffic first (analytics, prefetch, bots, batch) and keep critical traffic (checkout, login). Use request priority headers or criticality tags (Google's criticality levels: CRITICAL_PLUS, CRITICAL, SHEDDABLE_PLUS, SHEDDABLE).
- **LIFO queuing under overload** / **CoDel** (controlled delay): when queues are long, old requests are likely already abandoned by clients, so serve the newest or drop requests that waited too long (Facebook's adaptive LIFO).
- Drop requests whose deadline has already passed (don't do work nobody's waiting for).
- **Client-side throttling** (Google SRE book): clients track their own accept ratio and probabilistically drop requests locally when the backend rejects many, so rejected requests cost the backend nothing. `p(reject) = max(0, (requests − K × accepts) / (requests + 1))`.

---

## 7. Backpressure {#backpressure}

Signal upstream to **slow down** instead of buffering unboundedly:
- **Bounded queues** everywhere (thread pool queues, channels, in-memory buffers). An unbounded queue just converts overload into memory exhaustion and huge latency.
- Pull-based consumption (Kafka consumers, Reactive Streams `request(n)`, gRPC/HTTP2 flow control) gives natural backpressure.
- TCP flow control propagates backpressure across the network if the application stops reading.
- When you can't slow the producer (user traffic), backpressure becomes **load shedding** at the edge.
- Async systems: monitor queue depth and age, and autoscale consumers, or apply producer-side rate limits.

---

## 8. Graceful Degradation and Fallbacks {#degradation}

Design features so that **partial failure gives a reduced experience, not an error page**:
- Recommendations down → show popular items (static/cached list).
- Personalization down → generic homepage.
- Reviews service down → product page without reviews (lazy-loaded, independently failing component).
- Search down → category browsing.
- Payment provider A down → route to provider B.
- Read-only mode when the primary DB is unavailable (serve from replicas and caches, disable writes with a clear message).
- **Feature flags / kill switches** to disable expensive features under load.
- **Stale-while-revalidate / stale-if-error**: serve cached data past TTL when the origin fails.
- Fallbacks must be **simpler and more reliable** than the primary path, tested regularly, and must **not** call the same failing dependency (fallback storms).
- Decide per dependency: is it critical (fail the request) or optional (degrade)? Map your dependency graph by criticality.

---

## 9. Hedged and Tied Requests {#hedging}

From *"The Tail at Scale"* (Dean & Barroso, 2013): in large fan-out systems, tail latency dominates. If a request touches 100 servers and each has a 1% chance of a 1 s hiccup, then 63% of requests hit at least one slow server.
- **Hedged requests**: send the request to one replica; if there's no response within the p95 latency, send a second request to another replica, use whichever responds first, and cancel the other. This costs ~5% extra load and cuts tail latency dramatically.
- **Tied requests**: send to two replicas simultaneously, with each knowing about the other. The first to start processing cancels the other.
- Only for **idempotent reads** (or with dedup). gRPC supports hedging policies. Cassandra **speculative retry** is the same idea.
- Other tail-latency techniques: micro-partitioning (many small shards for fast rebalancing), selective replication of hot data, latency-induced probation (temporarily removing slow servers), and good-enough results (return partial results when some shards are slow, as in search).

---

## 10. Caching for Resilience {#caching}

Caches absorb load and can serve during origin outages, but they introduce failure modes (see [`caching/`](../caching/)):
- **Cache stampede / thundering herd**: a hot key expires and 1,000 requests miss simultaneously and hammer the DB. Fixes: request coalescing (single-flight: one fetch, others wait), probabilistic early expiration (XFetch), locks around regeneration, stale-while-revalidate.
- **Cold cache after restart/flush** creates a metastable failure risk. Warm caches before taking traffic, ramp traffic slowly, and never flush production caches casually.
- **Cache as a critical dependency**: if the DB can't handle the load without the cache, then the cache *is* critical. Make it highly available (replicas, cluster) and have a plan for when it's down (load shedding).
- **Cache penetration** (queries for non-existent keys bypass the cache): cache negative results, use Bloom filters.
- **TTL jitter**: avoid synchronized expiration of many keys.

---

## 11. Health Checks, Self-Healing, Restarts {#health}

- Liveness (restart if broken) vs readiness (stop routing if not ready). See [`networking/04-load-balancers-proxies-and-gateways.md`](../networking/04-load-balancers-proxies-and-gateways.md#health).
- **Don't put dependency checks in liveness probes**: a DB outage would restart every pod in a loop, adding cold starts to the outage.
- Crash-only design: make startup and recovery the same code path, so killing and restarting is always safe.
- Supervisors (Kubernetes, systemd, Erlang/OTP "let it crash") restart failed processes. Combine them with **backoff** (CrashLoopBackOff) to avoid hot restart loops.
- **Watchdogs** for deadlocks, plus memory limits that kill leaking processes before they starve the node.

---

## 12. Redundancy, Cells, Blast Radius {#cells}

- **Redundancy** at every layer (N+1 or N+2 capacity), multi-AZ by default. Survive the loss of one AZ with the remaining capacity (which means running at ≤ 66% utilization in 3 AZs).
- **Blast radius reduction**: limit how much fails when something fails.
  - **Cell-based architecture** (AWS, Slack, DoorDash): split the system into independent **cells** (complete stacks serving a subset of customers, routed by a thin cell router on customer ID). A bad deploy, poison request, or overload affects only one cell (e.g., 1/20 of customers).
  - **Shuffle sharding** (AWS Route 53): each customer is assigned a random combination of k out of N nodes. Two customers rarely share all their nodes, so a "poison" customer taking down its nodes affects very few others completely.
  - Progressive deployment: one canary instance → one cell/AZ → one region → everywhere, with automated rollback on metric regression and bake time between waves.
  - Regional isolation: no cross-region dependencies in the request path.
- **Avoid global single points of failure**: global config services, a single auth service, DNS, control planes. Design for "control plane down, data plane keeps working".

---

## 13. Static Stability {#static}

AWS's principle: **the system keeps working during an impairment without needing to make changes** (no need to launch new instances, call control-plane APIs, or update DNS during the failure).
- Pre-provision capacity across AZs so losing one AZ doesn't require scaling up during the incident (control planes may themselves be impaired).
- Data planes cache control-plane config and continue with the last-known-good state if the control plane is unreachable.
- Prefer push-based config with local caching over runtime lookups.
- EC2 example: running instances keep running even if the EC2 API is down.

---

## 14. Chaos Engineering and Game Days {#chaos}

**Chaos engineering** (Netflix, *Principles of Chaos*): experiment on production-like systems to build confidence in their resilience.
1. Define **steady state** (business metrics: orders/min, p99 latency).
2. Hypothesize that steady state continues during a fault.
3. Inject real-world faults: kill instances (Chaos Monkey), add latency, drop packets, fail a dependency, exhaust CPU or disk, AZ evacuation (Chaos Kong), clock skew, DNS failure.
4. Look for differences, and **minimize blast radius** (start small, with abort conditions).
5. Automate it continuously.

Tools: Chaos Monkey/Simian Army, Gremlin, AWS Fault Injection Service, Azure Chaos Studio, LitmusChaos, Chaos Mesh (Kubernetes), Toxiproxy (inject network faults in tests), Pumba.

**Game days**: scheduled exercises simulating incidents (region failover drills, restore from backup, "what if Redis dies?") with the on-call team, to validate runbooks, alerts, and dashboards. **DiRT** (Google's Disaster Recovery Testing) is the large-scale version.

Load testing to find limits before users do: k6, Gatling, Locust, JMeter, Vegeta. Know your breaking point and how the system behaves past it (it should shed load, not collapse).

---

## 15. Libraries and Tools {#tools}

| Ecosystem | Resilience library |
|---|---|
| Java | **Resilience4j** (circuit breaker, retry, bulkhead, rate limiter, time limiter), Failsafe; Hystrix (retired) |
| .NET | **Polly** (resilience pipelines, built into `Microsoft.Extensions.Http.Resilience`) |
| Go | sony/gobreaker, failsafe-go, hashicorp/go-retryablehttp, cenkalti/backoff, `x/time/rate`, singleflight |
| Node.js | opossum (circuit breaker), cockatiel, p-retry, Bottleneck (rate limiting) |
| Python | tenacity (retries), pybreaker, aiobreaker |
| Infra-level | Envoy/Istio/Linkerd (retries, timeouts, outlier detection, circuit breaking, retry budgets), API gateways |

Prefer **infrastructure-level** policies (the mesh/gateway) for uniformity, and **application-level** ones where business context matters (fallback content, idempotency).

---

## 16. Interview Questions {#qa}

1. Walk through how a slow database can take down an entire microservice system. Which patterns break the chain?
2. Why can retries make outages worse? How do you retry safely?
3. Explain the circuit breaker states. How do you choose thresholds?
4. What's the difference between a bulkhead and a circuit breaker?
5. What is load shedding, and how do you decide what to shed?
6. What is a metastable failure? Give an example and explain how to recover.
7. How do hedged requests reduce tail latency, and when are they unsafe?
8. What is cell-based architecture? How does shuffle sharding reduce blast radius?
9. What does static stability mean?
10. Design graceful degradation for an e-commerce product page.
11. How would you run your first chaos experiment safely?
