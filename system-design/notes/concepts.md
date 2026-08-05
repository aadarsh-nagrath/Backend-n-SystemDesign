# System Design Concepts: Reference Glossary

40 core backend/system design concepts, each with a plain-language explanation, how it works, a concrete example, trade-offs/anti-patterns, and a checklist. Meant as a dense reference — skim the TL;DR-style opener per concept, drop into "How it works" and "Example" for depth, use the checklist when actually building the thing.

## Table of contents
- [1) Circuit breakers, timeouts, retries](#1-circuit-breakers-timeouts-retries)
- [2) Distributed transactions (sagas)](#2-distributed-transactions-sagas)
- [3) Serialization & schema evolution](#3-serialization--schema-evolution)
- [4) Database choice (SQL vs NoSQL)](#4-database-choice-sql-vs-nosql)
- [5) API design (REST vs RPC/gRPC)](#5-api-design-rest-vs-rpcgrpc)
- [6) Normalization vs denormalization](#6-normalization-vs-denormalization)
- [7) Consensus & leader election](#7-consensus--leader-election)
- [8) Health checks & heartbeats](#8-health-checks--heartbeats)
- [9) Service discovery & config](#9-service-discovery--config)
- [10) Microservices vs monolith](#10-microservices-vs-monolith)
- [11) Rate limiting & throttling](#11-rate-limiting--throttling)
- [12) Data privacy & retention](#12-data-privacy--retention)
- [13) Data modeling & schema](#13-data-modeling--schema)
- [14) Event sourcing & CQRS](#14-event-sourcing--cqrs)
- [15) Redundancy & failover](#15-redundancy--failover)
- [16) Deployment strategies](#16-deployment-strategies)
- [17) Sharding / partitioning](#17-sharding--partitioning)
- [18) Latency & throughput](#18-latency--throughput)
- [19) Concurrency control](#19-concurrency-control)
- [20) Consistency models](#20-consistency-models)
- [21) Delivery semantics](#21-delivery-semantics)
- [22) Capacity estimation](#22-capacity-estimation)
- [23) Real-time delivery](#23-real-time-delivery)
- [24) Disaster recovery](#24-disaster-recovery)
- [25) Queues & streams](#25-queues--streams)
- [26) Cache invalidation](#26-cache-invalidation)
- [27) Caching strategies](#27-caching-strategies)
- [28) Networking basics](#28-networking-basics)
- [29) AuthN & AuthZ](#29-authn--authz)
- [30) Load balancing](#30-load-balancing)
- [31) API versioning](#31-api-versioning)
- [32) Multithreading](#32-multithreading)
- [33) Backpressure](#33-backpressure)
- [34) CAP theorem](#34-cap-theorem)
- [35) Observability](#35-observability)
- [36) Idempotency](#36-idempotency)
- [37) CDN & edge](#37-cdn--edge)
- [38) Replication](#38-replication)
- [39) Scalability](#39-scalability)
- [40) Indexing](#40-indexing)

---

### 1) Circuit breakers, timeouts, retries

Protects services from cascading failures when a dependency is slow or failing, by bounding how long you wait, retrying only what's safe to retry, and stopping calls to a dependency that's clearly down.

**When to use**: any network I/O — databases over TCP, caches, message brokers, internal/external HTTP/gRPC calls.

**How it works**:
- **Timeout** — bounds how long you wait for a dependency per attempt, plus an overall deadline for the whole request. Fail fast beats hanging.
- **Retry** — re-attempts transient failures with exponential backoff and jitter, within a total time budget, only for idempotent operations.
- **Circuit breaker** — tracks failure rate/latency; states are closed → open → half-open. Trips open past a threshold (stops calling the dependency entirely), then half-open sends a few probe requests to test recovery.

Concrete example (pseudocode):
```pseudo
deadline = 2500ms
attempt = 0
while now < start+deadline and attempt < 3:
  timeout = min(100ms * 2^attempt + jitter(±30%), 800ms)
  try call dep with timeout
    return ok
  on timeout/5xx: sleep(backoff) and retry if idempotent
if breaker_open: fail fast 503
```

Related patterns: hedged requests for tail latency (send a backup request after the p95 latency mark, cancel whichever loses); overall retry budget per user request rather than per-call.

**Anti-patterns**: retrying non-idempotent operations; synchronized retries across many clients (thundering herd); infinite timeouts; ignoring cancellation signals.

**Metrics**: error rate, p95/p99 latency, retry rate, breaker open duration, thread/connection pool saturation.

**Checklist**: sane defaults per dependency; a total request time budget; idempotency keys; cancellation propagated through the call chain; dashboards and alerts on breaker state.

---

### 2) Distributed transactions (sagas)

Multi-service data updates need atomicity across boundaries that a single database transaction can't span. Two answers: two-phase commit (strong but low-availability, coordinator is a single point of failure) or sagas (a sequence of local transactions with compensating actions if something fails downstream).

**When to use**: cross-service workflows (order → pay → ship) where 2PC is impractical or would hurt availability too much.

**How it works**:
- **Orchestration** — a central saga orchestrator issues commands to each service and awaits replies/timeouts, driving the state machine explicitly.
- **Choreography** — services publish events and react to each other's events; the workflow emerges implicitly, with no central coordinator.

Example — order saga:
1. `CreateOrder(pending)`
2. `ReserveInventory` → on failure: `CancelOrder`
3. `AuthorizePayment` → on failure: `ReleaseInventory` + `CancelOrder`
4. `CreateShipment` → on failure: `RefundPayment` + `ReleaseInventory` + `CancelOrder`
5. `MarkOrder(complete)`

Supporting patterns: a state machine per saga instance; the outbox pattern for reliably publishing events alongside a local commit; per-step timeouts and retries; a dead-letter queue for poison messages.

**Anti-patterns**: hidden coupling via events with no explicit contracts; missing compensations for some failure paths; non-idempotent handlers; no correlation IDs to trace a saga across services.

**Metrics**: saga completion time, failure rate per step, compensation rate, stuck/long-running saga counts.

**Checklist**: compensations defined for every step; idempotent handlers; correlation and causation IDs; persistent saga state; end-to-end monitoring.

---

### 3) Serialization & schema evolution

Serialization converts in-memory data to a wire/storage format; schema evolution lets that format's structure change over time without breaking readers or writers on different versions.

**When to use**: any inter-service communication, event logs, or persisted blobs that must be read by different versions or languages over time.

**How it works**: formats (JSON, Protobuf, Avro, Thrift, MessagePack, CBOR) define field identifiers and types — binary formats are compact and fast, text formats are human-friendly and easier to debug. Compatible evolution means new readers can read old data and old readers can read new data (ignoring unknown fields, falling back to defaults for missing ones).

Key techniques: stable field tags (Protobuf field numbers), a schema registry (Avro), reserving removed field numbers instead of reusing them, additive-only changes when compatibility matters.

Example: `Person{1:name, 2:age}` evolves to `Person{1:name, 2:age, 3:middle_name(optional)}`. Old readers ignore field 3; new readers default it when absent. Field 5 must never be repurposed to mean something else later.

**Anti-patterns**: reusing field numbers; changing an existing field's meaning; removing fields without a migration path; mixing null and empty-string/zero-value semantics inconsistently.

**Metrics**: consumer decode error rates, schema registry evolution audit trail, payload sizes over time.

**Checklist**: contract tests between producers and consumers; documented compatibility rules; deploy readers before writers; keep golden test vectors for each schema version.

---

### 4) Database choice (SQL vs NoSQL)

SQL databases enforce relational schemas and ACID transactions; NoSQL systems trade some of that for horizontal scale, flexible schema, or a specific access pattern. Neither is universally "better" — the choice follows from access patterns.

**When to use**: SQL for relational integrity, multi-row transactions, and complex queries. NoSQL for massive scale, flexible/nested documents, or very high write throughput on a simple access pattern.

**How it works** — NoSQL families and what they're for:
- **Key-value** (Redis, DynamoDB) — extreme throughput, simple key-based access.
- **Document** (MongoDB, Couch) — flexible schema, nested data.
- **Columnar/wide-column** (Cassandra, Bigtable) — high write throughput, time-series and large datasets.
- **Graph** (Neo4j) — relationship-heavy queries and traversals.

Decision guide:
| Need | Pick |
|---|---|
| OLTP with constraints | Postgres/MySQL |
| Analytics / time-series | ClickHouse/BigQuery |
| Write-heavy at large scale | Cassandra/Bigtable |
| Sessions / cache | Redis/Memcached |
| Relationships / traversals | Neo4j |

Example — a typical e-commerce stack: SQL for orders/payments (need transactions), Redis for session/cart (need speed, tolerate loss), Elasticsearch for search, Kafka for the event backbone. Multiple databases in one system, each earning its place.

**Anti-patterns**: picking a database for hype rather than access patterns; forcing relational joins into a NoSQL store without remodeling for it; ignoring backup/migration paths until it's too late.

**Metrics**: query latency/throughput, p95 write latency, replication lag, index hit ratio.

**Checklist**: model to actual access patterns first; plan indexing; test backup/restore; project growth before you're forced to.

---

### 5) API design (REST vs RPC/gRPC)

REST exposes resources over HTTP with standard verbs, caching, and wide client compatibility. gRPC defines RPC methods in a Protobuf contract over HTTP/2, with lower latency and native streaming. Public APIs lean REST; internal high-performance service-to-service calls lean gRPC.

**When to use**: REST for public APIs, anything cache-friendly, broad client reach. gRPC for low-latency internal calls, bi-directional streaming, or where a strict typed contract is valuable.

**How it works**: REST relies on URL paths, query params, and status codes as the interface; clients are typically hand-written or generated from OpenAPI. gRPC generates client/server stubs directly from `.proto` contracts and multiplexes many calls over one HTTP/2 connection.

Examples:
- REST: `GET /users?limit=50`, `PATCH /orders/{id}` with ETags and cache headers.
- gRPC: `rpc CreateOrder(CreateOrderRequest) returns (Order) {}` with explicit deadlines and interceptors for cross-cutting concerns (auth, logging, retries).

**Anti-patterns**: overloading POST for every action regardless of semantics; chatty APIs with no pagination; omitting deadlines on gRPC calls (they don't time out on their own); leaking internal types straight into the public contract.

**Metrics**: success/error rates, latency percentiles, payload sizes, cache hit ratio, gRPC deadline-exceeded rate.

**Checklist**: consistent resource design; explicit contracts (OpenAPI/Protobuf); pagination and filtering built in; auth on every endpoint; idempotency support for unsafe-verb retries.

---

### 6) Normalization vs denormalization

Normalization removes redundancy and enforces integrity via foreign keys and joins. Denormalization duplicates data to make reads fast at the cost of update complexity. Most real systems do both: normalize the transactional core, denormalize the read-heavy aggregates.

**When to use**: normalize wherever data is written and must stay correct; denormalize for read-heavy dashboards, listings, and aggregates where join cost dominates.

**How it works**: materialized views or projection pipelines build denormalized read models from a normalized source of truth, kept in sync on write or on a schedule.

Example: orders stay normalized (orders, items, customers as separate tables); a denormalized `order_summary` table aggregates order + items + customer + totals for fast listing pages, rebuilt or updated whenever the source rows change.

**Anti-patterns**: denormalizing without a clear owner/source of truth; duplicating mutable fields widely with no reconciliation process; treating denormalized data as authoritative.

**Checklist**: define the source of truth explicitly; decide the update strategy (sync write, async projection, scheduled rebuild); run reconciliation jobs; monitor for drift between source and denormalized copy.

---

### 7) Consensus & leader election

Consensus lets a set of nodes agree on a sequence of values despite failures and network issues; leader election is the common special case of picking a single coordinator node.

**When to use**: consistent configuration stores, distributed locks, metadata, and orchestrating any stateful system that needs a single writer.

**How it works (Raft, the common modern choice over classic Paxos)**: nodes hold a majority-vote election for leader; the leader replicates log entries to followers; an entry commits once a majority of nodes have it. ZooKeeper's Zab protocol and Multi-Paxos solve the same problem with different mechanics.

Example: etcd underpins Kubernetes' cluster state; databases elect a primary and use fencing tokens (monotonically increasing IDs attached to lock grants) to prevent two nodes from both believing they're the writer after a failover.

**Anti-patterns**: hand-rolling your own consensus algorithm instead of using a proven implementation; ignoring network partition behavior in design; long GC pauses that trigger false leader-failure elections.

**Checklist**: use proven systems (Raft/ZooKeeper/etcd) rather than building your own; set election timeouts based on realistic network latency; monitor leadership churn as a health signal.

---

### 8) Health checks & heartbeats

Three distinct signals get conflated as "health checks": liveness (is the process alive at all), readiness (can it currently serve traffic), and startup (has initialization finished). Heartbeats are periodic signals used for cluster membership.

**When to use**: always in orchestrated environments (Kubernetes, Nomad) and behind any load balancer.

**How it works**:
- **Liveness** — failing this restarts the container. Should only check "is this process still functioning," not downstream dependencies.
- **Readiness** — only route traffic when dependencies are OK and the app is warmed up. Failing this removes the instance from rotation without restarting it.
- **Startup** — delays liveness checks until a slow-starting app finishes initializing, so it isn't killed mid-boot.

Example: `/healthz` for liveness returns 200 unconditionally once the process is up; `/readyz` checks DB/cache connectivity and only returns 200 when those are actually reachable.

**Anti-patterns**: readiness that always returns true (defeats the purpose); checking expensive downstream dependencies on the liveness path (causes restart storms when a dependency is merely slow); no startup probe for apps with slow boot times.

**Checklist**: separate the three probe types; set sensible failure thresholds and backoff; combine with outlier detection at the load balancer.

---

### 9) Service discovery & config

Discovery maps a service name to its current network location(s), which matters once instances come and go dynamically (autoscaling, rolling deploys). Config delivery gets dynamic settings to running services without a redeploy.

**When to use**: microservices, autoscaling clusters, multi-zone deployments — anywhere instance addresses aren't static.

**How it works**:
- **Client-side discovery** — the client queries a registry (Consul, Eureka) directly and picks an instance itself.
- **Server-side discovery** — a load balancer or ingress does the lookup; the client just calls a stable endpoint.
- DNS-based discovery is the simplest option; a service mesh (Istio, Linkerd) adds retries, mTLS, and observability on top.
- Config typically lives in a key-value store (etcd, Consul, ZooKeeper), with feature flags or a config provider layered on for dynamic reload.

**Anti-patterns**: hardcoding service addresses; pushing config changes with no validation step; no fallback/default values if the config store is briefly unreachable.

**Checklist**: schema and validation on config; staged rollout for config changes; dynamic reload with a rollback path; seed nodes for cluster bootstrap.

---

### 10) Microservices vs monolith

A monolith is one deployable unit — simple to develop and operate, strongly consistent by default, easy to refactor early on. Microservices are many small services communicating over the network — independent deploys, polyglot tech choices, per-service scaling, at the cost of real operational complexity. The common path: start as a modular monolith, split out services only where a specific pain justifies it.

**When to use**: monolith early, for speed. Microservices once specific services need independent deploy cadence or independent scaling that a monolith can't give them.

**How it works**: define service boundaries with domain-driven design, give each service its own data store, agree on contracts between them, and invest in the platform capabilities microservices need (discovery, tracing, CI/CD per service).

**Walkthrough — extracting one service from a monolith**:
1. **Day 0** — Users, Catalog, Cart, Checkout, Orders, Payments all live in one codebase and one database. Fast to build, easy to deploy.
2. **Pain shows up** — Checkout changes weekly while other teams deploy monthly, and Checkout also absorbs the biggest traffic spikes (sales), slowing everyone's deploys down.
3. **Draw a boundary** — decide Checkout (cart → payment → order confirmation) should move behind a clear API. Write down the calls needed: `CreateCheckoutSession`, `AddItem`, `ApplyCoupon`, `PlaceOrder`.
4. **Prepare the monolith** — wrap Checkout logic behind an internal module that already uses those API-shaped functions, in-process. Add metrics/logs to observe real behavior first.
5. **Carve the data** — give Checkout its own tables or database (`checkouts`, `checkout_items`). Other services read via read-only views or events; nothing else writes Checkout's tables directly anymore.
6. **Put an API in front** — build the actual Checkout service exposing the API designed in step 3. Initially the monolith calls it over loopback so behavior stays identical.
7. **Cut traffic over safely** — behind a feature flag, route 10% of real traffic to the new service, watch errors/latency, ramp 10% → 50% → 100%. Flip the flag back if anything breaks.
8. **Finish the move** — once stable, remove the old in-process call path. Checkout now deploys and scales independently.
9. **Stop** — don't extract further services until there's a concrete pain (different release cadence, different scale, a real team boundary) justifying it.

**Anti-patterns**: a distributed monolith (services split by name but still tightly coupled and deployed together); premature decomposition before any real pain exists; a shared database across services (defeats the point of splitting).

**Checklist**: strong module boundaries even inside a monolith; platform capabilities (discovery, tracing, CI/CD) in place before splitting; observability per service; clear per-service ownership.

---

### 11) Rate limiting & throttling

Rate limiting caps how many requests are allowed over a time window; throttling shapes/slows traffic to protect downstream systems. Goals: protect resources, keep usage fair across clients, block abuse.

**When to use**: public APIs, login endpoints, expensive operations, bursty producers, and anywhere multi-tenant fairness matters.

**How it works** — algorithm choice depends on whether you want to allow bursts or smooth them out:
- **Token bucket** — allows bursts up to the bucket size, refills at a steady rate.
- **Leaky bucket** — smooths output regardless of input burstiness.
- **Fixed/sliding window** — counts events per time window; fixed windows have edge effects at window boundaries, sliding windows avoid that.
- **Sliding log** — exact per-event tracking, most precise, most storage-heavy.

Enforce at the gateway, in-service, or via a shared store (Redis) with atomic increments and expiry:
```text
KEY = user:123:rl:60
INCR if new then EXPIRE 60
if value > limit then 429 with Retry-After
```

**Anti-patterns**: a single global counter causing contention under load; no dimensioning (per-user/IP/tenant); accidentally rate-limiting health checks.

**Metrics**: allowed vs. limited request counts, p95 latency added by the limiter, hot-key skew, per-tenant distribution.

**Checklist**: pick the algorithm to match burst vs. smooth needs; use atomic operations; return informative headers (`X-RateLimit-*`, `Retry-After`); always allow safety/health routes through.

---

### 12) Data privacy & retention

Privacy governs lawful, minimal, purpose-bound processing of personal data. Retention governs how long that data is kept before deletion. Both are legally required in most jurisdictions handling user data (GDPR, CCPA, PCI, etc.), not optional engineering nice-to-haves.

**When to use**: always, whenever handling user data.

**How it works**: maintain a data inventory and data-flow map, bind data collection to a stated purpose, record consent, enforce access controls, encrypt at rest and in transit (plus field-level encryption/tokenization for the most sensitive fields), and run retention schedules with automated deletion pipelines.

Examples: logs auto-expire after 30 days via TTL; a user deletion request triggers erasure jobs across every service and backup, within a documented time window.

**Anti-patterns**: storing PII in logs; indefinite retention with no policy; copying PII into analytics systems without minimizing it first.

**Metrics**: deletion SLA adherence, percentage of data encrypted at rest, access audit anomalies, data minimization ratio.

**Checklist**: inventory what you hold; minimize collection; encrypt; enforce access control; define retention policies; build deletion workflows; audit regularly.

---

### 13) Data modeling & schema

Data modeling shapes entities, relationships, and constraints to fit how the application actually queries and mutates data — start from access patterns, not from what feels "clean" in the abstract.

**When to use**: continuously — at initial design and again every time a feature changes how data is accessed.

**How it works**: start from the queries you need to serve, choose a normalization level that fits, add indexes for the hot paths, and encode invariants as real constraints (not just application-layer checks) wherever the database supports it.

Example: embed comments directly in a post document if reads always fetch them together; break comments into a separate collection once they're queried independently or grow unbounded.

**Anti-patterns**: overloading a single column with multiple meanings; storing lists as comma-separated strings instead of a proper structure; skipping constraints and relying on application code alone for integrity.

**Metrics**: query plan quality, index hit ratio, migration run times, constraint violation counts.

**Checklist**: clear primary keys; consistent naming conventions; real constraints; indexes matched to hot queries; a defined migration strategy.

---

### 14) Event sourcing & CQRS

CQRS (Command Query Responsibility Segregation) separates the write model from the read model, letting each be optimized independently. Event sourcing stores state as an append-only, immutable log of domain events rather than as current-state rows — the two are often paired but are separate ideas.

**When to use**: when you need auditability, temporal queries ("what did this look like last Tuesday"), the ability to rebuild projections after a bug, complex write-side invariants, or a large read fan-out with several tailored projections of the same data.

**How it works**: writes append events to the log; readers build projections (materialized views) from that log. Snapshots of current state speed up recovery so you don't replay the entire history every time.

Example: a bank ledger stores debit/credit events; the current balance is derived by replaying them (or from a snapshot + recent events); a read model exposes account statements built from the same log.

**Anti-patterns**: using "events" as generic change-data-capture without real domain meaning; rebuilding projections without versioning the event schema first.

**Metrics**: event append latency, projection lag behind the log, replay time, snapshot frequency.

**Checklist**: version events explicitly; make handlers idempotent; have a snapshotting plan; monitor projection lag; invest in migration tooling for event schema changes.

---

### 15) Redundancy & failover

Redundancy duplicates components so there's no single point of failure; failover is the mechanism that switches traffic to a healthy replica when one fails. Applies at multiple levels — zonal, regional, multi-cloud — with the level chosen matching how much downtime/data loss the business can tolerate.

**When to use**: any production system with a real uptime requirement.

**How it works**:
- **Active-active** — multiple instances serve traffic simultaneously, load-balanced; failure of one just reduces capacity.
- **Active-passive** — a standby waits, promoted on health-check failure of the primary.
- **RTO** (recovery time objective) and **RPO** (recovery point objective / acceptable data loss) drive which topology and replication mode (sync vs. async) makes sense.
- Quorum-based reads/writes tolerate a minority of nodes being down without losing correctness.

Example: a multi-AZ database with automatic failover; a regional failover strategy driven by DNS or a traffic manager for a full-region outage.

**Anti-patterns**: failover paths that have never actually been tested (game days/chaos exercises exist specifically to catch this); data divergence after failover with no reconciliation plan; hidden regional dependencies that defeat a supposedly regional failover.

**Metrics**: RTO and RPO actually achieved during drills, failover success rate, replication lag.

**Checklist**: document RTO/RPO targets; test failover procedures for real, not just on paper; monitor continuously; automate the failover path; assign clear ownership.

---

### 16) Deployment strategies

Ways to release a new version while minimizing downtime and blast radius if something's wrong. Pick the strategy based on how critical the system is and how fast you need to detect a bad release.

**When to use**: always — the question is which strategy, not whether to use one.

**How it works**:
- **Blue/green** — two identical environments; traffic switches from one to the other atomically. Fast rollback (switch back), but needs 2x capacity during the switch.
- **Rolling** — instances are replaced gradually with the new version. No capacity doubling, but a bad release affects some traffic before it's caught.
- **Canary** — a small subset of traffic (e.g. 5%) gets the new version first; expand gradually (5% → 20% → 50% → 100%) while watching metrics.
- **Feature flags** — decouple deploy from release; toggle behavior at runtime without a new deploy, useful for risky changes independent of the deployment mechanism above.

Example: a 5%/20%/50%/100% canary ramp with automated rollback on error-rate regression; database migrations done as expand/contract (add new column, dual-write, backfill, cut over, remove old column) rather than a single breaking change.

**Anti-patterns**: one-shot big-bang deploys with no gradual rollout; schema-breaking migrations shipped without dual-write/dual-read support during the transition.

**Metrics**: error budget burn rate, latency and error rate during rollout specifically, rollback frequency.

**Checklist**: a real rollback plan; immutable build artifacts; metrics/alerts gating progression between rollout stages; migration safety verified before the code that depends on it ships.

---

### 17) Sharding / partitioning

> Full deep-dive with strategies (hash, range, directory, consistent hashing), rebalancing, hotspot handling, and worked examples: [`scaling-db/sharding.md`](../../scaling-db/sharding.md).

Splitting data or work across multiple nodes to scale horizontally once a single node can't hold or serve it fast enough. Distinct from replication (copying the *whole* dataset elsewhere) — sharding splits it.

**How it works, briefly**: hash-based keys distribute load evenly but kill ordered range scans; range-based keys support ordered scans but risk hotspots on sequential keys (e.g. time-ordered IDs); directory-based sharding keeps an explicit key→shard mapping for flexibility at the cost of an extra lookup. Consistent hashing minimizes data movement on rebalance.

**Anti-patterns**: skewed shard keys creating hot shards; cross-shard transactions (usually a sign the shard key is wrong); ad-hoc resharding with no plan.

**Checklist**: choose the key deliberately; have rebalance tooling ready; dual-write during migration; monitor for skew continuously.

---

### 18) Latency & throughput

Latency is time per request; throughput is requests processed per unit time. They're related but not interchangeable — you can have low latency and low throughput (single-threaded, fast per call) or high throughput with high latency (heavily batched).

**When to optimize**: SLO misses on p95/p99, visible queue buildup, or direct user experience complaints — not preemptively without a measured problem.

**How it works**: **Little's Law** (L = λW — items in system equals arrival rate times wait time) explains why reducing wait time increases achievable throughput at the same concurrency level. Tail latency (p99) matters disproportionately once you fan out to multiple downstream calls per request, since the slowest one dominates. Techniques: caching, batching, pipelining, parallelism, and network/kernel tuning.

Examples: batch database writes instead of one-row-at-a-time; pipeline CPU-bound stages; compress over the network only where the CPU cost is worth the bandwidth saved.

**Anti-patterns**: optimizing before measuring; looking only at averages and ignoring tail latency; unbounded concurrency that looks fast until it collapses under load.

**Metrics**: latency histograms (never just an average), queue depth, utilization, CPU/memory/IO.

**Checklist**: measure before touching anything; fix the biggest contributor first; protect calls with timeouts and budgets; validate improvements with real load tests.

---

### 19) Concurrency control

Methods for keeping data correct when multiple actors read/write it concurrently — the choice is essentially about whether you prevent conflicts up front or detect them after the fact.

**When to use**: shared resources, counters, inventory, financial operations — anywhere two writers could race.

**How it works**:
- **Pessimistic** — locks (row/table-level) block other writers until the lock releases; simple to reason about but can deadlock and limits throughput under contention.
- **Optimistic** — version numbers or ETags detect conflicts after the fact; the losing writer retries. Scales better under low contention, wastes work under high contention.
- **Distributed** — a lease/lock via a consensus store (etcd, ZooKeeper), always paired with a fencing token so a writer that lost its lock (e.g. due to a GC pause) can't still commit stale writes.

Examples: `SELECT ... FOR UPDATE` for pessimistic locking; `ETag`/`If-Match` headers for optimistic concurrency over HTTP; an etcd lock with a monotonically increasing fencing token.

**Anti-patterns**: holding locks too long; no deadlock avoidance strategy; a distributed lock used without a fencing token (the classic way "the lock holder" and "who actually committed" diverge).

**Metrics**: conflict rate, lock wait times, deadlock counts, abort/retry counts.

**Checklist**: prefer optimistic under low contention; always use timeouts on locks; instrument conflict and retry rates.

---

### 20) Consistency models

Guarantees about what order and staleness of reads/writes a distributed system promises. Distinct from — but related to — the CAP theorem, which is about the availability trade-off *during* a partition.

**When to choose**: based on product needs — correctness requirements vs. availability vs. latency, decided per data type rather than once globally.

**How it works**:
- **Strong/linearizable** — every read reflects the latest write, system-wide, as if there were only one copy of the data.
- **Sequential/PRAM** — each client's own operations stay in order, though different clients may see different overall orderings.
- **Causal** — respects cause-and-effect ordering (if A happened-before B, every reader sees A before B), weaker than strong, stronger than eventual.
- **Eventual** — converges to the same value eventually, with no guarantee on how stale a read can be in the meantime.
- **Tunable** — quorum-based systems (Cassandra) let you dial the read/write consistency level per operation.

Examples: a shopping cart wants read-your-writes within a session; analytics dashboards are fine with eventual; a financial ledger needs strong consistency.

**Anti-patterns**: assuming strong consistency is the global default without checking; mixing strong and eventual reads on the same data with no clear boundary for which is which.

**Metrics**: observed staleness, read/write latencies, quorum failure rate.

**Checklist**: define the required consistency level per domain/data type explicitly; document and enforce it; test actual behavior under simulated partitions.

---

### 21) Delivery semantics

The guarantee a messaging system makes about how many times a message is delivered — the trade-off is between risking loss and risking duplicates, and "exactly-once" is mostly a useful fiction achieved by combining at-least-once delivery with idempotent processing.

**When to use**: pick based on tolerance for loss vs. duplicates and the cost of processing.

**How it works**:
- **At-most-once** — no retries; a message can be lost but is never duplicated.
- **At-least-once** — retries on any doubt; duplicates are possible, so consumers must be idempotent.
- **Exactly-once** — genuinely hard end-to-end across independent systems; usually simulated via idempotency keys + deduplication + transactional writes, not a native guarantee you can just turn on.

Examples: payment processing uses idempotency keys so a retried charge doesn't double-charge; email sending tolerates occasional duplicates backed by a dedup cache instead.

**Anti-patterns**: relying on the broker alone for exactly-once semantics across multiple independent systems; consumers with no deduplication despite at-least-once delivery.

**Metrics**: redelivery rate, dedup cache hit rate, dead-letter-queue volume, processing latency.

**Checklist**: idempotency keys on writes; a dedup store; a DLQ with alerting; tooling to replay failed messages.

---

### 22) Capacity estimation

Forecasting the resources needed to meet SLOs under expected and peak load, done before launches, periodically as the system grows, and after any major architecture change.

**When to do it**: before launches; periodically as traffic grows; after major changes to the architecture.

**How it works**: forecast QPS, payload sizes, and concurrency; measure actual per-instance capacity via load testing; add headroom (commonly 30-50%) above forecast; build auto-scaling policies from the resulting curve. Load testing itself should establish a baseline, find the saturation point, and map the scaling curve between them.

Examples: provision for 2x normal daily peak and 3x for a known flash-sale event; size database IOPS for p99 write load, not average.

**Anti-patterns**: planning around averages only and ignoring peak/p99; skipping the back-of-the-envelope math entirely; ignoring hard external limits like third-party API quotas.

**Metrics**: saturation curves, utilization targets, cost per request, error rate under load.

**Checklist**: document every assumption; load-test before trusting the numbers; alert as usage approaches capacity; define concrete scaling triggers.

---

### 23) Real-time delivery

Pushing updates to clients as they happen, rather than clients polling for them.

**When to use**: chat, live collaboration, streaming dashboards, multiplayer, IoT telemetry.

**How it works**: transport options are WebSocket (full duplex), Server-Sent Events (simpler, server-to-client only), WebRTC data channels (peer-to-peer), or MQTT (lightweight, IoT-oriented). On top of the transport: pub/sub for fanout, presence tracking (who's currently connected), and per-connection backpressure so one slow client doesn't back up the whole system. Infra typically needs either sticky sessions (route a client back to the same server) or a shared broker so any server instance can deliver to any client.

Examples: chat rooms partitioned by room ID for horizontal scale; an offline message queue with a TTL for reconnecting clients; per-connection backpressure that sheds load gracefully instead of buffering unboundedly.

**Anti-patterns**: broadcasting every update to every connected client regardless of relevance; unbounded per-connection buffers; no heartbeat to detect dead connections.

**Metrics**: connected client count, fanout latency, dropped message count, reconnection rate.

**Checklist**: authenticate on connect; heartbeats; backpressure per connection; a deliberate sharding strategy for rooms/channels; graceful reconnect handling.

---

### 24) Disaster recovery

Architecture and procedures to restore service and data after a catastrophic failure — distinct from routine failover in scope (DR covers losing an entire region/datacenter, not just one node).

**When to use**: always, in some form; the specific RTO/RPO targets vary by business criticality.

**How it works**: tested backups (untested backups are not a DR plan), replication to a separate failure domain, warm standby capacity, DNS-based failover, and runbooks that are actually rehearsed rather than just written.

Examples: point-in-time recovery for a database; cross-region restore drills; automated failover tests run on a schedule, not just when something breaks.

**Anti-patterns**: backups that have never been test-restored; restore procedures that in practice exceed the stated RTO; a DR region that's missing dependencies the primary region has (so failover "succeeds" but the app still doesn't work).

**Metrics**: RTO/RPO actually attained in drills, restore success rate, drill frequency.

**Checklist**: backups tested regularly, not just taken; documented and rehearsed playbooks; automation over manual steps; clear ownership of the DR process.

---

### 25) Queues & streams

Queues (SQS, RabbitMQ) distribute discrete tasks to workers — point-to-point, pull-based, a message is typically consumed once. Streams (Kafka, Kinesis, Pulsar) are append-only logs supporting replay and multiple independent consumer groups reading the same data, with ordering guaranteed per partition.

**When to use**: queues for distributing work across a worker pool. Streams for event-driven architectures, fan-out to multiple independent consumers, or anywhere replay/audit matters.

**How it works**: queues use ack/nack plus a visibility timeout (message reappears if not acked in time). Streams use partitions, offsets per consumer group, and retention windows independent of whether a message has been "consumed."

Examples: an image-processing queue where each job goes to exactly one worker; business events on Kafka consumed independently by both a billing service and an analytics pipeline from the same topic.

**Anti-patterns**: using a plain queue when you actually need replay or multi-consumer fan-out; over-partitioning a stream with messages too small to justify the per-partition overhead.

**Metrics**: queue depth, consumer processing lag, redrive/DLQ counts, throughput per partition.

**Checklist**: pick queue vs. stream deliberately; define dead-letter handling; monitor consumer lag; capacity-plan partition and worker counts together.

---

### 26) Cache invalidation

The genuinely hard part of caching: knowing what to invalidate and when, so the cache never serves data that's meaningfully wrong.

**When to use**: any cached data that can change — i.e., almost all cached data eventually.

**How it works**: TTLs bound staleness automatically without any explicit invalidation logic; explicit key invalidation on write is more precise but requires the writer to know every cache key touched; cache-aside (read-through on miss, populate on read) is the most common pattern; versioned keys (embedding a version or content hash in the key) sidestep invalidation entirely by making stale keys simply unused rather than wrong.

Example: `user:123:v5` where `v5` increments on every profile update — old cached values under `v4` just age out naturally instead of needing active invalidation.

**Anti-patterns**: no invalidation strategy at all; long TTLs on data that changes often; a cache stampede (many clients simultaneously miss and hit the origin at once when a hot key expires).

**Metrics**: hit/miss ratio, rate of stale data actually served, stampede occurrences, resulting origin load.

**Checklist**: choose a strategy per data type, not one blanket policy; coordinate invalidation across services that share a cache; protect against stampedes (locking, early refresh); monitor continuously.

---

### 27) Caching strategies

Storing data closer to where it's consumed — client, CDN/edge, reverse proxy, application layer, or database cache — to cut latency and reduce load on the origin.

**When to use**: expensive reads, static content, computed results that are costly to regenerate.

**How it works**: each layer has a different lifetime and scope. Eviction policy matters once the cache is full: LRU (evict least recently used) is the default; LFU (least frequently used) suits skewed access patterns better; ARC adapts between the two. Negative caching (caching the fact that something doesn't exist) avoids repeated expensive misses. Stale-while-revalidate serves a slightly stale value immediately while refreshing in the background.

Examples: a CDN for images; Redis for a product catalog; local in-process memoization for config that rarely changes.

**Anti-patterns**: caching everything indiscriminately; caching stale-sensitive data with no invalidation plan; running a cache cluster with default eviction settings never tuned to actual access patterns.

**Metrics**: hit ratio and byte-hit ratio, origin egress, TTL effectiveness, eviction churn rate.

**Checklist**: place caches deliberately at each layer that earns it; set TTLs per data type; protect the origin from stampedes; instrument hit/miss continuously.

---

### 28) Networking basics

The layered foundations everything else sits on: OSI/TCP-IP model, TCP vs. UDP, TLS, and the HTTP/1.1 → 2 → 3 evolution — each of which changes multiplexing and head-of-line blocking behavior.

**Key points**: DNS lookup, TCP handshake, and TLS setup each add real latency before the first byte of a response even starts — this is why connection reuse (keep-alive) and HTTP/2+ multiplexing matter so much for real-world performance. Congestion control algorithms (CUBIC, BBR) shape how TCP throughput responds to loss; Nagle's algorithm can add latency for small, frequent writes if left on inappropriately.

Examples: enabling HTTP/2 for request multiplexing over one connection instead of many; using keep-alives to amortize handshake cost; tuning connection backlog and file descriptor limits for high concurrency.

**Checklist**: measure actual network timings (not just app-level latency) when debugging slowness; prefer HTTP/2/3 where the client and server both support it; secure every hop with TLS.

---

### 29) AuthN & AuthZ

Authentication (AuthN) verifies who's making a request; authorization (AuthZ) decides what that identity is allowed to do. Conflating the two is a common source of bugs — a request can be authenticated and still not authorized.

**When to use**: user logins, API access, and service-to-service trust — every request that isn't intentionally public.

**How it works**: OAuth2/OIDC handle delegated authentication (letting a user prove identity via a third party without sharing credentials with your app). Authorization models — RBAC (role-based), ABAC (attribute-based), ReBAC (relationship-based) — decide what's permitted, often centralized in a policy engine like OPA. Tokens carry the result: JWTs are self-contained and verifiable without a lookup (short TTL + rotation to limit blast radius from leakage); opaque tokens require an introspection call but can be revoked instantly. mTLS handles service-to-service trust without user-facing tokens at all.

Examples: an access token scoped to `orders:read`; a policy check delegated to OPA; mTLS between internal services so identity doesn't rely on a bearer token at all.

**Anti-patterns**: long-lived tokens with no rotation; putting sensitive data inside a JWT payload (it's base64, not encrypted, by default); no token revocation path at all.

**Metrics**: login success/failure rate, token issuance/refresh rate, policy decision latency.

**Checklist**: short-lived tokens; regular key rotation; least-privilege by default; audit logs on auth decisions; secrets stored and rotated properly.

---

### 30) Load balancing

Distributing traffic across multiple instances to improve both availability and performance — covered in depth for nginx/Apache specifically in [`web-servers/nginx-vs-apache.md`](../../web-servers/nginx-vs-apache.md).

**When to use**: any horizontally scaled service.

**How it works**: **L4** load balancing operates on IP/port (fast, protocol-agnostic); **L7** operates on HTTP semantics (can route by path/header, terminate TLS, inspect content). Algorithms: round robin (simplest), least connections (accounts for uneven request duration), weighted (accounts for uneven instance capacity), consistent hashing (stable client-to-instance mapping, useful for caching locality). Health checks remove bad endpoints from rotation automatically; connection draining lets in-flight requests finish before an instance is removed.

Examples: nginx/Envoy/ELB with outlier detection (automatically de-prioritizing an instance that's erroring); sticky sessions for legacy stateful apps, versus token-based auth for genuinely stateless ones (avoids needing stickiness at all).

**Anti-patterns**: no health checks at all; hard session stickiness causing hot-spotting on a few instances; ignoring slow-start (sending full traffic to a freshly started instance before it's warmed up).

**Metrics**: per-endpoint load, error rate, ejection count, request distribution evenness.

**Checklist**: health checks in place; slow-start configured; algorithm chosen to match the actual traffic pattern; full observability into per-instance load.

---

### 31) API versioning

Managing how a public API contract changes over time without breaking existing clients.

**When to use**: whenever a public/external contract changes in a way that could break a consumer.

**How it works**: version via URL (`/v1/`), a custom header (`Accept: application/vnd.company.v2+json`), or field-level/soft versioning (adding optional fields rather than bumping a whole version). The underlying discipline matters more than the mechanism: default to backward-compatible, additive changes; deprecate old versions with a clear timeline; publish changelogs so clients know what changed and when support ends.

Examples: `/v2/orders`; `Accept: application/vnd.company.v2+json`; a soft-versioned optional field added without any version bump at all.

**Anti-patterns**: shipping implicit breaking changes without a version bump; supporting legacy versions indefinitely with no sunset plan (each one is ongoing maintenance cost).

**Metrics**: client adoption by version, error rate per version, adherence to the stated deprecation timeline.

**Checklist**: a clear versioning policy written down; documentation kept current per version; telemetry on which versions are actually in use; a concrete rollout/deprecation plan.

---

### 32) Multithreading

Executing multiple threads concurrently to use available CPU cores and to hide I/O latency behind other work.

**When to use**: CPU-bound parallelism, or I/O-bound workloads built around blocking calls.

**How it works**: concurrency models range from raw threads, to an event loop (single-threaded, non-blocking I/O), to the actor model (isolated units communicating only via messages, sidestepping shared mutable state entirely). The classic hazards are data races (unsynchronized shared writes), deadlocks (circular lock waits), and false sharing (unrelated data on the same CPU cache line causing invalidation traffic). Mitigations: prefer immutable data, message passing over shared state, bounded thread pools, and structured concurrency (a task's child threads can't outlive the task itself).

Examples: separate thread pools for DB I/O vs. CPU-heavy work, so one doesn't starve the other; bounded queues in front of a pool to avoid unbounded memory growth under overload.

**Anti-patterns**: unbounded thread creation under load; shared mutable state with no synchronization; nested lock acquisition that can deadlock.

**Metrics**: context switch rate, lock wait time, queue sizes, achieved throughput.

**Checklist**: bound concurrency explicitly; avoid hot locks in the critical path; profile actual contention rather than guessing; load-test under realistic concurrency.

---

### 33) Backpressure

Mechanisms that keep a fast producer from overwhelming a slower consumer — controlling the producer's rate to match what the consumer can actually handle, rather than letting unbounded buffering paper over the mismatch.

**When to use**: streaming systems, message queues, any real-time delivery path.

**How it works**: bounded queues force a producer to slow down or drop once full (rather than growing memory unboundedly); credits/tokens let a consumer explicitly grant a producer permission to send more; feedback signals communicate consumer state upstream. TCP itself implements backpressure at the transport layer via flow control windows. Reactive Streams (as a spec/pattern) formalizes this for application-level stream processing, with consumer lag as the key health metric.

Examples: dropping the oldest frame in a live video pipeline under load rather than buffering indefinitely; applying per-connection send quotas so one slow client doesn't consume unbounded server memory.

**Anti-patterns**: unbounded buffers anywhere in the pipeline (the single most common backpressure bug); ignoring consumer lag until it's already critical.

**Metrics**: queue depth, consumer lag, drop count, recovery time after a backlog clears.

**Checklist**: define an explicit backpressure policy per stream; enforce hard bounds everywhere; expose backpressure state as a metric; make producers actually adapt to it rather than just failing.

---

### 34) CAP theorem

> Full deep-dive — the proof, PACELC, CP/AP/CA database classification, real-world examples, and common misconceptions: [`scaling-db/cap.md`](../../scaling-db/cap.md).

Under a network partition, a distributed system must choose between consistency (every read gets the latest write or an error) and availability (every request gets *some* response) — it cannot have perfect versions of both at once. Partition tolerance itself isn't really optional in a genuinely distributed system, since real networks do fail; the practical choice is CP or AP per partition event.

**How it works, briefly**: CP systems (Zookeeper, etcd, MongoDB) reject or delay requests during a partition rather than risk serving stale data. AP systems (DynamoDB, Cassandra) keep serving but may return stale reads or accept conflicting writes to reconcile later.

**Anti-patterns**: claiming a system is "CA" for a genuinely distributed deployment (not achievable — partitions aren't optional); designing without ever considering partition behavior at all.

**Checklist**: pick a CP/AP stance per domain, not once for the whole system; document what clients should expect from your API during a failure.

---

### 35) Observability

The ability to understand a system's internal state from its external outputs, built on three pillars — logs, metrics, traces — plus profiles, business events, real-user-monitoring, and SLIs/SLOs on top.

**When to use**: always — it's foundational to debugging, performance work, and reliability, not an add-on for later.

**How it works**: structured logs carry context (not just a message string); metrics carry labels for slicing by dimension; distributed tracing propagates a trace ID across service boundaries via headers, letting you follow one request through the whole system. The "golden signals" (latency, traffic, errors, saturation) are the standard starting dashboard for any service.

Examples: trace IDs propagated across every service a request touches; RED (rate/errors/duration) and USE (utilization/saturation/errors) dashboards; burn-rate alerts tied to SLO budgets rather than raw thresholds.

**Anti-patterns**: unstructured log lines that can't be queried; no sampling strategy on high-volume traces (cost explodes); trace context that isn't propagated across a service boundary, breaking the chain.

**Metrics**: trace coverage, log volume relative to actual signal extracted, SLO attainment over time.

**Checklist**: a unified logging schema; correlation IDs everywhere; sane retention policies; error budgets with real alerts attached.

---

### 36) Idempotency

An operation is idempotent if repeating it produces the same result as doing it once — critical wherever retries and at-least-once delivery are in play, which is most distributed systems.

**When to use**: payments, order creation, resource provisioning, webhook handlers — anywhere a duplicate side effect (double charge, duplicate resource) is a real problem.

**How it works**: assign an idempotency key per logical operation (client-generated, stable across retries); the server stores the result keyed by it and returns the same result on a repeat instead of re-executing; conditional updates (`ETag`/`If-Match`) achieve a similar effect for updates specifically.

Example: `POST /payments` with an `Idempotency-Key` header — the server returns the original result unchanged if the same key is submitted again, rather than charging twice.

**Anti-patterns**: using a timestamp or other non-stable value as the idempotency key (defeats the purpose — every retry gets a "new" key); duplicate side effects like emails or charges from not deduplicating at all.

**Metrics**: duplicate request rate, dedup cache hit rate, storage size consumed by idempotency keys.

**Checklist**: define what the key is and how long it's retained (TTL); persist it reliably; require it in clients for unsafe operations; actually test retry behavior, not just the happy path.

---

### 37) CDN & edge

A CDN caches and serves content from points of presence (POPs) close to users, cutting latency and offloading the origin. Edge compute runs actual logic (not just cached bytes) at those same locations for latency-sensitive personalization.

**When to use**: static assets always; APIs with cacheable responses; personalization that needs to happen with very low latency.

**How it works**: POPs cache by a cache key (which can vary by header, query param, or device); signed URLs/cookies control access to otherwise-cached content; edge functions can modify requests or responses in flight (redirects, A/B assignment, header injection) before they ever reach the origin.

Examples: image resizing performed at the edge instead of the origin; A/B test bucket assignment done at the edge; bot mitigation applied before a request ever reaches the origin.

**Anti-patterns**: cache-busting on every single deploy (defeats the purpose of the CDN for that window); leaking PII into edge logs, which are often less tightly controlled than origin logs.

**Metrics**: hit ratio and byte-hit ratio, origin offload percentage, edge error rate, time-to-first-byte by region.

**Checklist**: define cache keys and TTLs deliberately; protect sensitive content with signed URLs; monitor performance by region, not just in aggregate.

---

### 38) Replication

Copying data across multiple nodes for redundancy, read scaling, and failover support.

**When to use**: to scale reads, ensure durability beyond a single node, and support failover or geographic distribution.

**How it works**:
- **Synchronous** — a write isn't acknowledged until it's confirmed on replicas too; stronger durability, higher write latency.
- **Asynchronous** — a write is acknowledged immediately and replicated after the fact; lower latency, but a crash can lose the most recent writes.
- **Leader-follower** (single-leader) — all writes go through one node; simple, but the leader is a bottleneck and a single point of failure until failover completes.
- **Leaderless** (quorum-based, e.g. Cassandra/Dynamo-style) — any replica can accept writes; a quorum of reads/writes provides the consistency guarantee instead of a designated leader.
- **Multi-leader** — multiple nodes accept writes, requiring conflict resolution (last-write-wins, CRDTs, or application-level merge logic) when the same data is written in two places concurrently.

Examples: Postgres streaming replicas for read scaling and failover; Cassandra quorum reads/writes spread across replicas for tunable consistency.

**Anti-patterns**: assuming replication is synchronous when it's actually async (a common source of "but I just wrote that!" bugs after failover); no conflict resolution policy defined for multi-leader setups; ignoring replication lag as a real operational metric.

**Metrics**: replication lag, conflict rate (multi-leader), read/write latency by topology.

**Checklist**: choose the topology deliberately; monitor lag continuously; define a conflict resolution policy up front; actually test failover, don't just assume it works.

---

### 39) Scalability

The ability to handle load growth by adding resources efficiently — vertical (bigger machine) vs. horizontal (more machines), with horizontal generally preferred past a certain point since vertical scaling has a hard ceiling and a single point of failure.

**When to use**: whenever load is growing or spiky enough that current capacity is a real risk.

**How it works**: scale-out design requires statelessness in the app tier (session state pushed to a shared store) and shared-nothing architecture (see [Sharding](#17-sharding--partitioning)) at the data tier. **Amdahl's Law** bounds how much parallelism actually helps — the portion of the workload that's inherently serial caps the maximum speedup, no matter how much parallel capacity you add. Auto-scaling can be metrics-based (react to current load) or predictive (scale ahead of a known pattern); warm pools avoid cold-start latency when scaling out suddenly.

Examples: a stateless app tier behind a load balancer; database sharding for write scale; CDN offload for read scale; worker pools for background processing.

**Anti-patterns**: scaling one bottleneck while a different one caps throughput anyway (classic wasted effort); shared state that couples instances together and defeats horizontal scaling.

**Metrics**: throughput-vs-latency curves as load increases, cost per request, time to scale up/down.

**Checklist**: identify the actual bottleneck before scaling anything; design for horizontal scale by default; load-test at target scale, not just current scale; monitor cost alongside performance.

---

### 40) Indexing

> Full deep-dive — B-tree vs. LSM-tree internals, clustered vs. non-clustered, composite index ordering, covering indexes, and when the optimizer ignores an index: [`scaling-db/db-indexing.md`](../../scaling-db/db-indexing.md).

Data structures that accelerate queries by avoiding full table scans — the central trade-off is faster reads against slower writes and extra storage, since every index has to be maintained on every `INSERT`/`UPDATE`/`DELETE`.

**How it works, briefly**: B-tree indexes handle both equality and range queries and are the default in most relational databases; hash indexes are equality-only but O(1); LSM-trees trade read complexity for much better write throughput, which is why they show up in write-heavy stores (Cassandra, RocksDB-backed systems). Composite index column order matters — put the most selective/most-filtered column first. A covering index includes every column a query needs, avoiding the extra round-trip back to the table.

**Anti-patterns**: too many indexes (each one taxes every write); indexing low-selectivity columns that don't actually narrow the scan; composite index column order that doesn't match how queries actually filter.

**Metrics**: index hit ratio, scan vs. index-scan counts, index bloat/fragmentation, write amplification.

**Checklist**: align indexes with actual query patterns, not guesses; monitor usage; maintain (rebuild/reindex as needed); revisit as query patterns change over time.

---

## Practical patterns & checklists

### Timeout/retry/circuit breaker defaults
- Client timeout: slightly above dependency p95; enforce an overall request budget too.
- Retries: 2-3 attempts with exponential backoff + jitter, only for idempotent operations.
- Circuit breaker: trip on consecutive failures or error-rate threshold; use a half-open probe ratio to test recovery.

### Production readiness
- Health checks: liveness, readiness, and startup, kept distinct.
- Config: externalized, versioned, dynamically reloadable.
- Observability: dashboards for latency/errors/saturation; alerts on SLO burn rate.
- Security: TLS 1.2/1.3 minimum, rotated secrets, least privilege throughout.

### Data evolution
- Schema changes: additive and defaulted wherever possible.
- Breaking changes: dual-write/dual-read during the transition, never a single atomic cutover.
- Migrations: forwards-compatible, with backfill jobs and feature flags to control the switch.

### Disaster recovery
- Backups tested monthly (not just taken monthly).
- RTO/RPO documented and actually achievable, not aspirational.
- Failover drills run quarterly.

### Capacity & cost
- Track QPS, p95 latency, CPU/memory, egress continuously.
- Adopt a per-service budget; right-size instances; cache where it measurably reduces load.

---

## Extended examples

### Resilient payment workflow (saga)
Steps: create order → reserve inventory → pre-authorize payment → confirm order → capture payment → ship.

Failure handling:
- Payment declined → compensate by releasing inventory and cancelling the order.
- Shipment fails → refund or reattempt, and notify the customer either way.

Guarantees: idempotency keys on every step; saga state persisted so it survives a crash mid-workflow; retry with backoff; circuit-break calls to external providers (payment processors, carriers) specifically.

### Rate limiting in an API gateway
Token bucket per API key (N tokens/sec, burst size B), counters stored in Redis with TTL. Include `X-RateLimit-*` response headers on every request (not just when limited) and return 429 with `Retry-After` on excess; expose a status endpoint so clients can check their current limit state proactively.

### Real-time chat delivery
WebSocket gateway → Kafka (partitioned per room) → chat service → fanout to connected users. Handle backpressure by pausing reads per connection under load, capping buffers, and dropping the oldest non-essential events (typing indicators) before ever dropping messages themselves.

---

## References
- *Designing Data-Intensive Applications* — Martin Kleppmann
- *Site Reliability Engineering* — Google SRE
- *The Datacenter as a Computer* — Barroso, Hölzle, Ranganathan
- Raft paper; *Paxos Made Simple*
- *CAP Twelve Years Later* — Eric Brewer
- Kafka, Redis, Cassandra, Postgres official docs
