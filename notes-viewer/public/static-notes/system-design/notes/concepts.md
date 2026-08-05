## System Design Concepts: Deep-Dive Guide with Examples

This guide explains 40 core backend/system design concepts in depth. Each section includes a plain-language explanation, practical examples, trade-offs, and best practices.

### Table of Contents
- [1) Circuit Breakers, Timeouts, Retries](#1-circuit-breakers-timeouts-retries)
- [2) Distributed Transactions (Sagas)](#2-distributed-transactions-sagas)
- [3) Serialization & Schema Evolution](#3-serialization--schema-evolution)
- [4) Database Choice (SQL vs NoSQL)](#4-database-choice-sql-vs-nosql)
- [5) API Design (REST vs RPC/gRPC)](#5-api-design-rest-vs-rpcgrpc)
- [6) Normalization vs Denormalization](#6-normalization-vs-denormalization)
- [7) Consensus & Leader Election](#7-consensus--leader-election)
- [8) Health Checks & Heartbeats](#8-health-checks--heartbeats)
- [9) Service Discovery & Config](#9-service-discovery--config)
- [10) Microservices vs Monolith](#10-microservices-vs-monolith)
- [11) Rate Limiting & Throttling](#11-rate-limiting--throttling)
- [12) Data Privacy & Retention](#12-data-privacy--retention)
- [13) Data Modeling & Schema](#13-data-modeling--schema)
- [14) Event Sourcing & CQRS](#14-event-sourcing--cqrs)
- [15) Redundancy & Failover](#15-redundancy--failover)
- [16) Deployment Strategies](#16-deployment-strategies)
- [17) Sharding / Partitioning](#17-sharding--partitioning)
- [18) Latency & Throughput](#18-latency--throughput)
- [19) Concurrency Control](#19-concurrency-control)
- [20) Consistency Models](#20-consistency-models)
- [21) Delivery Semantics](#21-delivery-semantics)
- [22) Capacity Estimation](#22-capacity-estimation)
- [23) Real-time Delivery](#23-real-time-delivery)
- [24) Disaster Recovery](#24-disaster-recovery)
- [25) Queues & Streams](#25-queues--streams)
- [26) Cache Invalidation](#26-cache-invalidation)
- [27) Caching Strategies](#27-caching-strategies)
- [28) Networking Basics](#28-networking-basics)
- [29) AuthN & AuthZ](#29-authn--authz)
- [30) Load Balancing](#30-load-balancing)
- [31) API Versioning](#31-api-versioning)
- [32) Multithreading](#32-multithreading)
- [33) Backpressure](#33-backpressure)
- [34) CAP Theorem](#34-cap-theorem)
- [35) Observability](#35-observability)
- [36) Idempotency](#36-idempotency)
- [37) CDN & Edge](#37-cdn--edge)
- [38) Replication](#38-replication)
- [39) Scalability](#39-scalability)
- [40) Indexing](#40-indexing)

---

### 1) Circuit Breakers, Timeouts, Retries
- What/Why: Protect services from cascading failures when a dependency is slow or failing.
- Components:
  - Timeouts: Bound how long you wait for a dependency (fail fast > hang).
  - Retries: Retry transient failures with jittered exponential backoff; avoid retry storms.
  - Circuit Breaker: States: closed → open → half-open. Trips open on error-rate or latency thresholds; half-open tests recovery.
- Example:
  - API calls DB via service A; DB is overloaded. Without timeouts/backoff, threads pile up; with breaker, A fails fast and sheds load.
- Patterns:
  - Use hedged requests for tail latency (send a backup after T p95). Cancel loser.
  - Budget retries: overall timeout budget per user request.
- Pitfalls: Retrying non-idempotent ops; synchronized retries (thundering herd).
- Best Practices:
  - Set timeouts near p95 of dependency + margin.
  - Exponential backoff with jitter.
  - Track breaker metrics; expose to dashboards.

Definition:
- Timeout ends slow calls predictably; Retry re-attempts transient failures; Circuit breaker stops calls to a failing dependency to let it recover and to protect upstream resources.

When to use:
- Any network I/O: databases over TCP, caches, message brokers, internal/external HTTP/gRPC.

How it works:
- Timeout per attempt and total deadline. Retries use exponential backoff with jitter within a total budget. Circuit breaker monitors failure rate/latency; opens when thresholds exceeded; half-open probes a few requests to determine recovery.

Algorithms:
- Exponential backoff with full jitter; rolling window error-rate calculation; token-bucket for hedged requests.

Concrete example (pseudo):
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

Anti-patterns:
- Retrying non-idempotent operations; synchronized retries; infinite timeouts; ignoring cancellation signals.

Metrics to track:
- Error rate, p95/p99 latency, retry rate, breaker open duration, thread/connection pool saturation.

Checklist:
- Defaults per dependency; total request budget; idempotency keys; cancellation propagation; dashboards and alerts.

---

### 2) Distributed Transactions (Sagas)
- Problem: Multi-service data updates need atomicity across boundaries.
- Two-Phase Commit (2PC): Coordinator prepares then commits. Strong consistency; low availability; coordinator single point.
- Saga Pattern: Sequence of local transactions with compensating actions for failures.
  - Orchestration: Central saga orchestrator drives steps.
  - Choreography: Steps emit events; downstream services react.
- Example: Order → Reserve inventory → Charge payment → Create shipment. If payment fails, release inventory.
- Tips: Design compensations upfront, ensure idempotency and deduplication.

Definition:
- A saga is a sequence of local transactions in different services with compensating actions to undo prior steps when later steps fail.

When to use:
- Cross-service workflows (order → pay → ship), where 2PC is impractical or harms availability.

How it works:
- Orchestrated: a central coordinator issues commands and awaits replies/timeouts.
- Choreographed: services publish events; others react, forming the workflow implicitly.

Algorithms/Patterns:
- State machine per saga instance; outbox pattern for reliable messaging; timeouts and retries per step; dead letter for poison messages.

Example (order saga):
1) CreateOrder(pending)
2) ReserveInventory → on failure CancelOrder
3) AuthorizePayment → on failure ReleaseInventory + CancelOrder
4) CreateShipment → on failure RefundPayment + ReleaseInventory + CancelOrder
5) MarkOrder(complete)

Anti-patterns:
- Hidden coupling via events without contracts; missing compensations; non-idempotent handlers; no correlation IDs.

Metrics:
- Saga completion time, failure rate per step, compensation rate, stuck/running counts.

Checklist:
- Define compensations; idempotent handlers; correlation and causation IDs; persistent state; monitoring.

---

### 3) Serialization & Schema Evolution
- Formats: JSON, Protobuf, Avro, Thrift, MessagePack, CBOR.
- Binary vs Text: Binary is compact/fast; text is human-friendly.
- Compatibility:
  - Backward compatible: New readers can read old data; old readers can read new data if unknown fields are ignored and defaults exist.
  - Techniques: Field tags (Protobuf), optional fields, default values, never reuse field numbers.
- Versioning: Use explicit schema versions in topics/endpoints; deploy readers before writers.
- Example: Add `middleName` optional field with default; do not change meaning of field 5.

Definition:
- Serialization converts in-memory data to a wire/storage format; schema evolution changes structures without breaking compatibility.

When to use:
- Any inter-service communication, event logs, or persisted blobs that must be read by different versions/languages.

How it works:
- Formats define field identifiers and types. Readers should ignore unknown fields; writers add fields with defaults.

Algorithms/Practices:
- Stable field tags (Protobuf), schema registry (Avro), reserving removed fields, additive-only changes for compatibility.

Example:
- V1: `Person{1:name,2:age}` → V2 adds `3:middle_name(optional)`. Old readers ignore field 3; new readers default when missing.

Anti-patterns:
- Reusing field numbers; changing field meaning; removing fields without migration; mixing null/empty semantics.

Metrics:
- Consumer error rates on decode, schema registry evolution audit, payload sizes.

Checklist:
- Contract tests; document compatibility rules; deploy readers first; keep golden test vectors.

---

### 4) Database Choice (SQL vs NoSQL)
- SQL (RDBMS): Strong consistency, joins, ACID transactions, mature tooling.
- NoSQL:
  - Key-Value (Redis/Dynamo): Extreme throughput, simple access.
  - Document (Mongo/Couch): Flexible schema, nested data.
  - Columnar/Wide-Column (Cassandra/Bigtable): High write throughput, time-series/large datasets.
  - Graph (Neo4j): Relationship-heavy queries.
- Choose by access patterns: If you need multi-row transactions & complex queries → SQL; if at scale with simple key-based access → KV/Columnar.

Definition:
- SQL databases enforce relational schemas and ACID transactions; NoSQL systems optimize for scale, flexible schema, or specific access patterns.

When to use:
- SQL for relational integrity and complex queries. NoSQL for massive scale, flexible documents, or high write throughput.

How it works:
- SQL uses normalized tables, indexes, and joins. NoSQL may use key-value access, document stores, column families, or graphs.

Decision guide:
- OLTP with constraints → Postgres/MySQL.
- Analytics/time-series → ClickHouse/BigQuery.
- Write-heavy, large scale → Cassandra/Bigtable.
- Sessions/cache → Redis/Memcached.
- Relationships/traversals → Neo4j.

Example:
- E-commerce: SQL for orders/payments; Redis for session/cart; Elasticsearch for search; Kafka for events.

Anti-patterns:
- Picking tech for hype; forcing joins in NoSQL without modeling; ignoring backup and migration paths.

Metrics:
- Query latency/throughput, p95 write latency, replication lag, index hit ratio.

Checklist:
- Model to access patterns; plan indexing; backup/restore tested; growth projections.

---

### 5) API Design (REST vs RPC/gRPC)
- REST: Resource-oriented, HTTP verbs, caching friendly, easy to debug, ubiquitous.
- RPC/gRPC: Contract-first (Protobuf), HTTP/2, bi-directional streaming, low latency.
- Guidance: Public APIs → REST; internal high-performance microservice calls → gRPC.

Definition:
- REST exposes resources over HTTP with standard verbs; gRPC defines RPC methods with Protobuf over HTTP/2, supporting streaming.

When to use:
- REST for public APIs, caching, and broad client compatibility. gRPC for low-latency, internal service-to-service calls and bi-directional streams.

How it works:
- REST relies on URL paths, query params, and status codes. gRPC generates client/server stubs from .proto contracts and multiplexes streams over HTTP/2.

Examples:
- REST: `GET /users?limit=50`, `PATCH /orders/{id}` with ETags and caching headers.
- gRPC: `rpc CreateOrder(CreateOrderRequest) returns (Order) {}` with deadlines and interceptors.

Anti-patterns:
- Overloading POST for all actions; chattiness without pagination; not setting deadlines in gRPC; leaking internal types.

Metrics:
- Success/error rates, latency percentiles, payload sizes, cache hit ratio, deadline exceeded in gRPC.

Checklist:
- Consistent resource design; OpenAPI/Protobuf contracts; pagination and filtering; auth; idempotency for unsafe retries.

---

### 6) Normalization vs Denormalization
- Normalization: Reduce redundancy; strong integrity; JOINs common.
- Denormalization: Duplicate data for read performance; update complexity.
- Hybrid: Normalize core data; denormalize hot aggregates/caches.
- Example: Store product info normalized; materialize “product + price + inventory” view for reads.

Definition:
- Normalization removes redundancy and enforces integrity; denormalization duplicates data to optimize reads.

When to use:
- Normalize transactional core; denormalize for read-heavy aggregates and dashboards.

How it works:
- Use materialized views or projection pipelines to build read models from normalized sources.

Example:
- Orders normalized; a denormalized `order_summary` table aggregates order, items, customer, totals for fast listing.

Anti-patterns:
- Denormalizing without ownership; duplicating mutable fields widely; no reconciliation.

Checklist:
- Define source of truth; update strategies; reconciliation jobs; monitoring for drift.

---

### 7) Consensus & Leader Election
- Leader election: Choose a single writer/coordination node (Raft, ZooKeeper/Zab, etcd).
- Consensus protocols: Paxos, Raft (easier to implement/understand), Multi-Paxos.
- Use cases: Config store, locks, metadata, cluster membership.
- Pitfalls: Split-brain; clock drift assumptions.

Definition:
- Consensus ensures nodes agree on a sequence of values; leader election picks a coordinator.

When to use:
- For consistent configuration, locks, metadata, and orchestrating stateful systems.

How it works (Raft):
- Nodes elect a leader via majority vote; leader replicates log entries to followers; entries commit when replicated to a majority.

Example:
- etcd for Kubernetes state; databases elect a primary with fencing tokens to avoid dual-writes.

Anti-patterns:
- DIY consensus; ignoring network partitions; long GC pauses leading to false elections.

Checklist:
- Use proven systems (Raft/ZK/etcd); set timeouts based on latency; monitor leadership churn.

---

### 8) Health Checks & Heartbeats
- Liveness: Is process alive? Readiness: Can it serve traffic? Startup: Is it done initializing?
- Heartbeats: Periodic signals to indicate health/membership.
- Use: Load balancers/K8s use readiness to route; auto-restart on failing liveness.

Definition:
- Health checks indicate if a process is alive, ready to serve, or still starting; heartbeats signal membership.

When to use:
- Always in orchestrated environments (K8s, Nomad) and behind load balancers.

How it works:
- Liveness: restarts the container if unresponsive.
- Readiness: only route when dependencies OK and warmed.
- Startup: delay liveness until init completes.

Example:
- `/healthz` for liveness returns 200; `/readyz` checks DB/cache and returns 200 only when OK.

Anti-patterns:
- Readiness always true; checking expensive dependencies on liveness; no startup probes for slow apps.

Checklist:
- Separate probes; sensible thresholds; backoff; outlier detection in LB.

---

### 9) Service Discovery & Config
- Patterns: Client-side discovery (Eureka/Consul), server-side discovery (LB/Ingress).
- Config: Centralized store (etcd/Consul/ZK), dynamic reload, versioned, audit.
- DNS-based discovery for simplicity; service mesh for advanced features.

Definition:
- Discovery maps service names to network locations; config delivers dynamic settings to services.

When to use:
- Microservices, autoscaling clusters, multi-zone deployments.

How it works:
- Client-side discovery queries a registry (Consul/Eureka). Server-side discovery uses a load balancer/ingress.
- Config via key-value stores (etcd/Consul), feature flags, or providers.

Anti-patterns:
- Hardcoding addresses; unsafe config pushes without validation; no fallback values.

Checklist:
- Schema and validation; staged rollout; dynamic reload with rollback; seed nodes for bootstrap.

---

### 10) Microservices vs Monolith
- Monolith: Simple dev/ops; strong consistency; easier refactoring early.
- Microservices: Independent deploys, polyglot, scaling by service; operational complexity.
- Path: Start modular monolith → split hot boundaries when needed.

Definition:
- Monolith: one deployable unit. Microservices: many small services communicating over the network.

When to use:
- Monolith early for speed and simplicity. Microservices when teams/services need independent deploys and scaling.

How it works:
- Define clear service boundaries (domain-driven design), separate data stores, shared contracts, and platform support (discovery, tracing, CI/CD).

Example (walkthrough: from monolith to one service extraction):
1) Day 0 — Everything in one app: Users, Catalog, Cart, Checkout, Orders, Payments live in a single codebase and database. This is fast to build and easy to deploy.
2) Pain shows up — Checkout changes weekly, but other teams deploy monthly. Checkout also gets most traffic spikes (sales), slowing everyone’s deploys.
3) Draw a boundary — We decide that Checkout (cart → payment → order confirmation) should move behind a clear API. We write down the request/response we need, like `CreateCheckoutSession`, `AddItem`, `ApplyCoupon`, `PlaceOrder`.
4) Prepare the monolith — Inside the monolith, we wrap all Checkout logic behind an internal module (same process) that already uses those API-shaped functions. We add metrics and logs to see how it behaves.
5) Carve the data — We give Checkout its own tables (e.g., `checkouts`, `checkout_items`) or a separate database. Other services read what they need through read-only views or events. No other module writes Checkout tables directly anymore.
6) Put an API in front — We build a small HTTP/gRPC service for Checkout that exposes the same API we designed. For a while, the monolith calls this API locally (loopback) so behavior stays identical.
7) Cut traffic over safely — Behind a feature flag, we route 10% of real traffic from the monolith to the new Checkout service, watch errors/latency, then ramp to 50% → 100%. If anything breaks, flip the flag off and all calls go back to the monolith.
8) Finish the move — Once stable, we remove old in-process calls. Now Checkout can deploy on its own schedule, and we can scale it separately during sales without touching the rest of the app.
9) Keep it simple — We do not extract more services until there’s a clear pain (different release cadence, different scale, or a hard team boundary).

Anti-patterns:
- Distributed monolith (tight coupling across services); premature decomposition; shared DB across services.

Checklist:
- Strong module boundaries; platform capabilities; observability; per-service ownership.

---

### 11) Rate Limiting & Throttling
- Goals: Protect resources, ensure fairness, prevent abuse.
- Algorithms: Token Bucket, Leaky Bucket, Fixed/Sliding Window, Sliding Log.
- Dimensions: per user/IP/API key/tenant; global vs per-endpoint.
- Implementation: Redis counters, Envoy/Nginx, API gateway.

Definition:
- Rate limiting controls how many requests are allowed over time; throttling slows or shapes traffic to protect downstream systems.

When to use:
- Public APIs, login endpoints, expensive operations, bursty producers, and to enforce fair usage across tenants.

How it works:
- Algorithms: token bucket (allow bursts), leaky bucket (smooths), fixed/sliding windows (count events per window), sliding log (precise per event).
- Enforce at gateway, service, or shared store (e.g., Redis) with atomic increments and expirations.

Example (Redis + Lua):
```text
KEY = user:123:rl:60
INCR if new then EXPIRE 60
if value > limit then 429 with Retry-After
```

Anti-patterns:
- Global single counter causing contention; no dimensioning; rejecting health checks.

Metrics:
- Allowed vs limited counts, p95 latency at limiter, hot key skew, dimensional distribution per tenant.

Checklist:
- Choose algorithm by need (burst vs smooth), atomics, return informative headers, prioritize safety routes.

---

### 12) Data Privacy & Retention
- Principles: Data minimization, purpose limitation, consent, subject rights.
- Techniques: Pseudonymization, encryption at rest/in transit, field-level encryption, tokenization.
- Retention: TTL policies, deletion workflows, legal hold.

Definition:
- Privacy ensures lawful, minimal, and purpose-bound processing of personal data; retention governs how long data is stored.

When to use:
- Always when handling user data; legally required under GDPR/CCPA/PCI/etc.

How it works:
- Data inventory and flows, purpose binding, consent records, access controls, encryption, retention schedules, and deletion pipelines.

Examples:
- TTL for logs after 30 days; user deletion triggers erasure jobs across services and backups (with documented windows).

Anti-patterns:
- Storing PII in logs; indefinite retention; copying PII into analytics without minimization.

Metrics:
- Deletion SLA, encrypted-at-rest coverage, access audit anomalies, data minimization ratios.

Checklist:
- Inventory, minimize, encrypt, access control, retention policies, deletion workflows, audits.

---

### 13) Data Modeling & Schema
- Start from queries/access patterns; design for how data is used.
- Use canonical IDs, avoid overloading fields.
- Evolve with migrations; embrace backward/forward compatibility.

Definition:
- Data modeling shapes entities, relationships, and constraints to support application behaviors and queries.

When to use:
- Early in design and continuously as features evolve.

How it works:
- Start from access patterns; choose normalization level; add indexes; encode invariants as constraints.

Examples:
- Document embed comments for post reads; separate collection if comments are huge and queried independently.

Anti-patterns:
- Overloading columns, storing lists in comma-separated strings, lack of constraints.

Metrics:
- Query plans, index hit ratio, migration times, constraint violation counts.

Checklist:
- Clear primary keys, consistent naming, constraints, indexes for hot queries, migration strategy.

---

### 14) Event Sourcing & CQRS
- CQRS: Separate read/write models; read side optimized for queries.
- Event Sourcing: State derived from append-only event log; immutable history.
- Pros: Auditability, temporal queries, rebuild projections.
- Cons: Complexity, eventual consistency, migration cost.

Definition:
- Event sourcing stores state as an ordered log of domain events; CQRS splits write and read models.

When to use:
- Auditability, temporal queries, rollback by replay, complex write invariants; large read fan-out with tailored projections.

How it works:
- Writes append events; readers build projections (materialized views). Snapshots speed recovery.

Examples:
- Bank ledger events; rebuild balance by replay; read model exposes account statements.

Anti-patterns:
- Using events as generic change data without domain meaning; rebuilding projections without versioning.

Metrics:
- Event append latency, projection lag, replay times, snapshot frequency.

Checklist:
- Versioned events, idempotency, snapshotting plan, projection monitoring, migration tooling.

---

### 15) Redundancy & Failover
- Levels: Zonal, regional, multi-cloud.
- Active-active vs active-passive; RTO/RPO objectives.
- Quorum writes/reads; failover testing (game days/chaos).

Definition:
- Redundancy duplicates components to eliminate single points of failure; failover switches traffic to healthy replicas.

When to use:
- Any production system with uptime requirements.

How it works:
- Active-active with load balancing; active-passive with health checks and promotion.

Examples:
- Multi-AZ database with automatic failover; regional failover with DNS and traffic managers.

Anti-patterns:
- Unpracticed failover; data divergence without reconciliation; hidden regional dependencies.

Metrics:
- RTO (recovery time), RPO (data loss), failover success rate, replication lag.

Checklist:
- Documented RTO/RPO, tested procedures, monitoring, automation, and clear ownership.

---

### 16) Deployment Strategies
- Blue/Green: Two identical environments; switch traffic.
- Rolling: Gradual rollout across instances.
- Canary: Small subset gets new version; monitor; expand if healthy.
- Feature flags for runtime control.

Definition:
- Strategies to release new versions safely while minimizing downtime and risk.

When to use:
- Always; pick the strategy based on system criticality.

How it works:
- Blue/green swaps traffic; rolling updates replace instances gradually; canaries shift a small percentage and analyze.

Examples:
- 5%/20%/50%/100% canary ramp; feature flag toggles for risky changes; DB migrations with expand/contract.

Anti-patterns:
- One-shot big-bang deploys; schema-breaking migrations without dual writes/reads.

Metrics:
- Error budget burn, latency, error rate during rollout, rollback frequency.

Checklist:
- Rollback plan, artifact immutability, metrics/alerts gating, database migration safety.

---

### 17) Sharding / Partitioning
- Keys: Hash-based (uniform), range (ordered scans), directory-based.
- Rebalancing: Consistent hashing; move only affected shards.
- Hotspots: Avoid skew; choose keys with entropy; time-bucket rotation.

Definition:
- Partitioning splits data/work across nodes to scale horizontally.

When to use:
- Dataset or traffic exceeds a single node’s capacity.

How it works:
- Hash-based distributes evenly; range-based supports ordered queries; directory-based maintains a mapping.

Examples:
- Kafka partitions; DB sharding by user ID; time-range partitions for logs.

Anti-patterns:
- Skewed keys; cross-shard transactions; ad-hoc resharding without plan.

Metrics:
- Per-shard load, partition skew, hot partitions, rebalancing time.

Checklist:
- Key selection, rebalance tooling, dual-write during migration, monitoring skew.

---

### 18) Latency & Throughput
- Little’s Law: L = λW; optimize wait time to increase throughput.
- Tail Latency: p99 matters for user experience and fan-out calls.
- Techniques: caching, batching, pipelining, parallelism, kernel/network tuning.

Definition:
- Latency is time per request; throughput is requests processed per unit time.

When to optimize:
- SLO misses on p95/p99, queue buildup, user experience issues.

How it works:
- Queueing effects (Little’s Law), parallelism, batching, backpressure to control inflow.

Examples:
- Batch writes to DB; pipeline CPU-bound tasks; compress over network with proper CPU trade-offs.

Anti-patterns:
- Premature optimization; ignoring tail latency; unlimited concurrency.

Metrics:
- Latency histograms (not just averages), queue depth, utilization, CPU/memory/IO.

Checklist:
- Measure first; address biggest contributors; protect with timeouts and budgets; validate with load tests.

---

### 19) Concurrency Control
- Pessimistic: Locks (row/table); serialization; deadlocks possible.
- Optimistic: Version numbers/ETags; retry on conflict.
- Distributed: Lease/lock via consensus store; fencing tokens to prevent stale writers.

Definition:
- Methods to ensure correctness when multiple actors access/modify shared data.

When to use:
- Shared resources, counters, inventory, financial operations.

How it works:
- Pessimistic locks block others; optimistic detects conflicts via versions; distributed locks need fencing tokens.

Examples:
- Use `SELECT ... FOR UPDATE`; ETag/If-Match; etcd lock with increasing fencing.

Anti-patterns:
- Long-held locks; no deadlock avoidance; lock lost without fencing.

Metrics:
- Conflict rate, lock wait times, deadlocks, abort/retry counts.

Checklist:
- Prefer optimistic for low contention; use timeouts; instrument conflicts and retries.

---

### 20) Consistency Models
- Strong: Linearizable; reads reflect latest writes.
- Sequential/PRAM: Per-client order preserved.
- Causal: Respects cause-effect; stronger than eventual, weaker than strong.
- Eventual: Converges without guarantees on staleness.
- Tunable: QUORUM (e.g., Cassandra).

Definition:
- Guarantees about visibility/order of reads and writes in distributed systems.

When to choose:
- Based on product needs: correctness vs availability vs latency trade-offs.

How it works:
- Strong/linearizable vs causal vs eventual; quorum-based systems tune consistency per operation.

Examples:
- Shopping cart read-your-writes per session; analytics eventual; financial ledger strong.

Anti-patterns:
- Assuming global strong consistency by default; mixing strong and eventual without clear boundaries.

Metrics:
- Staleness observed, read/write latencies, quorum failures.

Checklist:
- Define per-domain consistency needs; document and enforce; test under partitions.

---

### 21) Delivery Semantics
- At-most-once: No retries; possible loss; no duplicates.
- At-least-once: Retries; duplicates possible; need idempotency.
- Exactly-once: Very hard end-to-end; simulate with idempotency + dedup + transactions.

Definition:
- Semantics for message delivery: at-most-once (no retries), at-least-once (retries/duplicates), exactly-once (logical effect once).

When to use:
- Pick based on tolerance for loss vs duplicates and processing cost.

How it works:
- At-least-once with idempotent consumers and deduplication keys; outbox pattern for producer consistency.

Examples:
- Payment processing uses idempotency keys; email sending tolerates duplicates with dedup cache.

Anti-patterns:
- Relying on broker for exactly-once across multiple systems; no dedup on consumers.

Metrics:
- Redelivery rate, dedup hits, DLQ volume, processing latency.

Checklist:
- Idempotency keys, dedup store, DLQ with alerts, replay tooling.

---

### 22) Capacity Estimation
- Process: Forecast QPS, payload sizes, growth; define SLOs; derive CPU/mem/network/storage.
- Headroom: 30–50% safety margin; plan for p95/p99.
- Load testing: baseline, saturation point, scaling curves.

Definition:
- Estimating resources required to meet SLOs under expected and peak loads.

When to do it:
- Before launches; periodically with growth; after major changes.

How it works:
- Forecast QPS, payloads, and concurrency; measure service capacity; add headroom; build scaling policies.

Examples:
- 2x daily peak, 3x flash-sale peak headroom; provision DB IOPS for p99 writes.

Anti-patterns:
- Average-only planning; no back-of-the-envelope; ignoring external limits (e.g., API quotas).

Metrics:
- Saturation curves, utilization targets, cost per request, error rates under load.

Checklist:
- Document assumptions; test; alert on approaching capacity; plan scaling triggers.

---

### 23) Real-time Delivery
- Protocols: WebSocket, Server-Sent Events (SSE), WebRTC data, MQTT.
- Patterns: Pub/sub, fanout, presence, backpressure, connection lifecycles.
- Infra: Sticky sessions or shared brokers; scale with sharding and presence services.

Definition:
- Delivering updates to clients as they happen via push channels.

When to use:
- Chat, collaboration, streaming dashboards, multiplayer, IoT.

How it works:
- WebSockets/SSE/HTTP2 streams; pub/sub messaging; presence tracking; fanout per channel.

Examples:
- Chat rooms partitioned by ID; offline queue with TTL; backpressure per connection.

Anti-patterns:
- Broadcasting to all; unbounded buffers; lack of heartbeats.

Metrics:
- Connected clients, fanout latency, dropped messages, reconnection rates.

Checklist:
- AuthN on connect, heartbeats, backpressure, shard strategy, graceful reconnect.

---

### 24) Disaster Recovery
- RTO (time) and RPO (data loss) targets.
- Backups: Verified restores, immutable backups, cross-region.
- Runbooks, drills, automation (infrastructure as code).

Definition:
- Procedures and architecture to restore service/data after catastrophic failures.

When to use:
- Always; requirements vary by business criticality.

How it works:
- Backups (tested), replication, warm standbys, DNS failover, runbooks.

Examples:
- Point-in-time recovery for DB; cross-region restores; automated failover tests.

Anti-patterns:
- Backups not tested; restore times exceeding RTO; missing dependencies in DR region.

Metrics:
- RTO/RPO attainment, restore success rate, drill frequency.

Checklist:
- Regular tested backups, documented playbooks, automation, and ownership.

---

### 25) Queues & Streams
- Queues (SQS/RabbitMQ): Task distribution, point-to-point, work pulling.
- Streams (Kafka/Kinesis/Pulsar): Append-only logs, replay, consumer groups, ordering per partition.
- Picking: Retry semantics, ordering needs, throughput, retention.

Definition:
- Queues deliver tasks to workers; streams provide ordered logs for scalable consumption.

When to use:
- Queues for work distribution; streams for event-driven architectures and analytics.

How it works:
- Queues: ack/nack, visibility timeouts. Streams: partitions, offsets, consumer groups.

Examples:
- Image processing queue; business events on Kafka with multiple consumers (billing, analytics).

Anti-patterns:
- Using a queue when you need replay/ordering; over-partitioning with too small messages.

Metrics:
- Queue depth, processing lag, redrive counts, throughput per partition.

Checklist:
- Choose correctly; define DLQs; monitor lag; capacity plan partitions/workers.

---

### 26) Cache Invalidation
- Hard problem: What to invalidate and when.
- Strategies: Explicit key invalidation, TTLs, write-through/write-behind, cache-aside.
- Tools: Versioned keys (hash of payload), generational caches.

Definition:
- Ensuring caches reflect the correct data when the source changes.

When to use:
- Any cached mutable data.

How it works:
- TTLs, explicit key invalidation on writes, cache-aside patterns, versioned keys for automatic busting.

Examples:
- `user:123:v5` where v5 increments on profile update; cache-aside read-through on miss.

Anti-patterns:
- No invalidation strategy; long TTLs on hot mutable data; cache stampede.

Metrics:
- Hit/miss ratio, stale serves, stampede occurrences, origin load.

Checklist:
- Choose strategy per data; coordinate invalidations; protect against stampede; monitor.

---

### 27) Caching Strategies
- Layers: Client, CDN, edge, reverse proxy, app cache, DB cache.
- Policies: LRU/LFU/ARC; negative caching; stale-while-revalidate; prewarming.
- Metrics: Hit ratio, byte hit ratio, origin load.

Definition:
- Layered approaches to store data closer to compute/users to reduce latency and load.

When to use:
- Expensive reads, static content, computed results.

How it works:
- Client/CDN/edge/proxy/app/DB caches; eviction policies (LRU/LFU/ARC), negative caching, SWR.

Examples:
- CDN for images; Redis for product catalog; local in-process memoization for config.

Anti-patterns:
- Caching everything; stale-sensitive data without invalidation; cache clusters without eviction tune.

Metrics:
- Hit/byte-hit ratio, origin egress, TTL effectiveness, eviction churn.

Checklist:
- Place caches strategically; set TTLs; protect origin; instrument.

---

### 28) Networking Basics
- OSI vs TCP/IP; TCP vs UDP; TLS; HTTP/1.1 vs 2 vs 3.
- Latency sources: DNS, TCP handshake, TLS, queueing, server processing, network hops.
- Congestion control: CUBIC/BBR; Nagle’s algorithm; keep-alives.

Definition:
- Foundations of communication: layers, protocols, and performance characteristics.

Key points:
- DNS lookup, TCP handshake, TLS setup add latency; HTTP versions change multiplexing and head-of-line behavior.

Examples:
- Enable HTTP/2 for multiplexing; use keep-alives; tune backlog and file limits for high concurrency.

Checklist:
- Measure network timings; prefer HTTP/2/3 where appropriate; secure with TLS.

---

### 29) AuthN & AuthZ
- AuthN: Credentials → identity (passwords, MFA, OAuth2, OIDC, SAML).
- AuthZ: RBAC/ABAC/ReBAC; policy engines (OPA), scopes/claims.
- Patterns: JWTs (short TTL + rotation), opaque tokens (introspection), mTLS for service-to-service.

Definition:
- Authentication verifies identity; authorization decides what that identity can do.

When to use:
- For user logins, API access, and service-to-service trust.

How it works:
- OAuth2/OIDC for delegated auth; RBAC/ABAC/ReBAC for permissions; tokens (JWT/opaque) convey claims.

Examples:
- Access token with scope `orders:read`; policy checks with OPA; mTLS for internal services.

Anti-patterns:
- Long-lived tokens; putting sensitive data in JWT; no token revocation.

Metrics:
- Login success/failure, token issuance/refresh rates, policy decision latency.

Checklist:
- Short-lived tokens; rotate keys; least privilege; audit logs; secure secrets.

---

### 30) Load Balancing
- L4 vs L7; algorithms: round robin, least connections, weighted, consistent hashing.
- Health checks; connection draining; sticky sessions (beware of skew).
- Global LB: anycast DNS, geo routing.

Definition:
- Distributing traffic across multiple instances to improve availability and performance.

When to use:
- Any horizontally scaled service.

How it works:
- L4/L7 load balancers use algorithms (round-robin, least-conns, weighted). Health checks remove bad endpoints.

Examples:
- Nginx/Envoy/ELB with outlier detection; sticky sessions for legacy, tokens for stateless.

Anti-patterns:
- No health checks; hard stickiness causing hot-spotting; ignoring slow-start.

Metrics:
- Per-endpoint load, error rates, ejections, request distribution.

Checklist:
- Health checks, slow-start, proper algorithm selection, observability.

---

### 31) API Versioning
- URL (`/v1/`), header (`Accept: application/vnd.company.v2+json`), field-level/soft versioning.
- Guidelines: Backward compatibility by default; additive changes; deprecate with timelines; provide changelogs.

Definition:
- Managing API evolution over time without breaking clients.

When to use:
- Whenever public contracts change.

How it works:
- URI/header versioning; additive changes; deprecation policies; compatibility layers.

Examples:
- `/v2/orders`; `Accept: application/vnd.company.v2+json`; soft-version fields.

Anti-patterns:
- Implicit breaking changes; unbounded support for legacy versions.

Metrics:
- Client adoption by version, error rates per version, deprecation timeline adherence.

Checklist:
- Clear policy, documentation, telemetry, rollout plans.

---

### 32) Multithreading
- Concurrency models: threads, event loop, actor model.
- Hazards: data races, deadlocks, false sharing.
- Tools: immutable data, message passing, thread pools, structured concurrency.

Definition:
- Executing multiple threads concurrently to utilize CPU and hide IO latency.

When to use:
- CPU-bound parallelism or IO-bound workloads with blocking calls.

How it works:
- Thread pools, futures/promises, async/await; avoid shared mutable state or protect with locks/atomics.

Examples:
- Separate pools for DB IO and CPU-heavy tasks; use bounded queues to avoid overload.

Anti-patterns:
- Unbounded thread creation; shared state races; nested locks causing deadlocks.

Metrics:
- Context switches, lock wait times, queue sizes, throughput.

Checklist:
- Bound concurrency, avoid hot locks, profile contention, test under load.

---

### 33) Backpressure
- Definition: Controlling producer rate to match consumer capacity.
- Approaches: Bounded queues, rate-based signaling, credits/tokens, TCP flow control.
- In streams: Reactive Streams, consumer lag metrics.

Definition:
- Mechanisms to prevent fast producers from overwhelming slow consumers.

When to use:
- Streaming systems, message queues, realtime delivery.

How it works:
- Bounded queues, credits/tokens, feedback signals; TCP flow control at transport layer.

Examples:
- Drop oldest frame in video pipeline; apply per-connection send quotas.

Anti-patterns:
- Unbounded buffers; ignoring consumer lag.

Metrics:
- Queue depth, consumer lag, drop counts, recovery time.

Checklist:
- Define policies, enforce bounds, expose backpressure metrics, adapt producers.

---

### 34) CAP Theorem
- You can’t have perfect Consistency, Availability, and Partition tolerance simultaneously under network partitions.
- Classify systems by behavior under partition: CP (prefer consistency), AP (prefer availability).
- Reality: Design around business needs; use bounded staleness or compensation.

Definition:
- Under network partitions, a system must choose between consistency and availability while tolerating partitions.

When to use:
- To reason about trade-offs in distributed storage and services.

How it works:
- CP systems reject some requests during partitions; AP systems serve but risk stale reads or write conflicts.

Examples:
- CP: Zookeeper/etcd; AP: DynamoDB/Cassandra.

Anti-patterns:
- Claiming CA under partitions; ignoring partition behaviors in design.

Checklist:
- Pick per-domain stance; document client expectations under failure.

---

### 35) Observability
- Three pillars: Logs, Metrics, Traces. Plus: profiles, events, RUM, SLOs/SLIs.
- Correlation: Trace IDs across services; structured logs.
- Golden signals: latency, traffic, errors, saturation.

Definition:
- Ability to understand internal state from external outputs (logs, metrics, traces).

When to use:
- Always; essential for debugging, performance, and reliability.

How it works:
- Structured logs with context, metrics with labels, distributed tracing with propagation headers.

Examples:
- Trace IDs across services; RED/USE dashboards; burn-rate alerts.

Anti-patterns:
- Unstructured logs; no sampling; missing propagation.

Metrics:
- Coverage of traces, log volume vs signal, SLO attainment.

Checklist:
- Unified schema, correlation IDs, retention policies, budgets and alerts.

---

### 36) Idempotency
- Definition: Repeating an operation yields the same result.
- Techniques: Idempotency keys, upserts, conditional updates (ETags/If-Match), at-most-once side effects.
- Use with retries and at-least-once delivery.

Definition:
- An operation that can be safely repeated without additional side effects.

When to use:
- Payments, order creation, provisioning, webhooks.

How it works:
- Assign idempotency key per logical operation; store result or detect duplicates; conditionally apply changes.

Examples:
- `POST /payments` with `Idempotency-Key`; server returns same result on retry.

Anti-patterns:
- Using timestamps or non-stable keys; duplicate side effects (emails, charges).

Metrics:
- Duplicate request rate, dedup hits, storage size for keys.

Checklist:
- Define keys, TTLs, persistence; include in clients; test retries.

---

### 37) CDN & Edge
- CDN: Distribute static/dynamic content near users; TLS termination, WAF, bot mitigation.
- Edge compute: Functions/Workers close to users for latency-sensitive logic.
- Cache keys: Vary by headers, query, device; use signed URLs.

Definition:
- CDN caches and delivers content from locations close to users; edge compute runs logic near users.

When to use:
- Static assets, APIs with cacheable responses, personalization with low latency.

How it works:
- POPs cache by key; signed URLs/cookies control access; edge functions modify requests/responses.

Examples:
- Image resizing at edge; A/B testing logic; bot mitigation before origin.

Anti-patterns:
- Cache-busting every deploy; leaking PII to edge logs.

Metrics:
- Hit/byte-hit ratio, origin offload, edge error rate, TTFB by region.

Checklist:
- Define cache keys and TTLs; protect with signed URLs; monitor by region.

---

### 38) Replication
- Synchronous vs asynchronous; leader-follower, leaderless (quorum-based), multi-leader.
- Lag, conflict resolution (last-write-wins, CRDTs, app merges).
- Read replicas for scale; promote on failover.

Definition:
- Copying data across nodes for redundancy and performance.

When to use:
- To scale reads, ensure durability, and support failover and geo distribution.

How it works:
- Sync vs async; single-leader vs multi-leader vs leaderless quorums.

Examples:
- Postgres streaming replicas; Cassandra quorum reads/writes across replicas.

Anti-patterns:
- Assuming sync when async; no conflict resolution policy; ignoring lag.

Metrics:
- Replication lag, conflict rate, read/write latencies by topology.

Checklist:
- Choose topology; monitor lag; define conflicts; test failover.

---

### 39) Scalability
- Vertical vs horizontal; scale-out design: stateless services, shared-nothing.
- Bottleneck analysis: Amdahl’s law; parallelism vs contention.
- Auto-scaling: metrics-based, predictive; warm pools.

Definition:
- Ability to handle growth in load by increasing resources efficiently.

When to use:
- When load grows or is spiky.

How it works:
- Scale up/out; remove state; partition; replicate; offload; cache; async.

Examples:
- Stateless app tier behind LB; DB sharding; CDN offload; worker pools.

Anti-patterns:
- Scaling one bottleneck while others cap; shared state coupling.

Metrics:
- Throughput vs latency curves, cost per request, scaling time.

Checklist:
- Identify bottlenecks; design for horizontal scale; test at scale; monitor cost.

---

### 40) Indexing
- B-Tree vs LSM-Tree; covering indexes; composite/index selectivity; partial indexes.
- Write amplification vs read performance trade-offs.
- Query planning: use EXPLAIN; avoid N+1; denormalize when necessary.

Definition:
- Data structures that accelerate queries by avoiding full scans.

When to use:
- For frequent filters/sorts/joins where full scans are expensive.

How it works:
- B-Tree for range/point lookups; hash indexes for equality; LSM trees for write-heavy workloads.

Examples:
- Composite index `(user_id, created_at DESC)` for timelines; partial index for active=true.

Anti-patterns:
- Too many indexes; low-selectivity columns; mismatch with query order.

Metrics:
- Index hit ratio, scan vs index scan counts, bloat/fragmentation, write amplification.

Checklist:
- Align with queries; monitor; maintain; revisit as patterns change.

---

## Practical Patterns & Checklists

### Timeout/Retry/Circuit Breaker Defaults
- Client timeout: slightly above p95; overall request budget enforced.
- Retries: 2–3 with exponential backoff + jitter, only for idempotent operations.
- Circuit breaker: trip on consecutive failures or error-rate; half-open probe ratio.

### Production Readiness
- Health checks: liveness, readiness, startup.
- Config: externalized, versioned, dynamic reload.
- Observability: dashboards for latency, errors, saturation; alert SLO burn rate.
- Security: TLS 1.2/1.3, rotate secrets, principle of least privilege.

### Data Evolution
- Schema changes: additive, defaulted; dual-write/dual-read when breaking changes.
- Migrations: forwards-compatible; backfill jobs; feature flags to switch.

### Disaster Recovery
- Backups tested monthly; RTO/RPO documented; failover drills quarterly.

### Capacity & Cost
- Track QPS, p95, CPU/mem, egress; adopt budgets per service; right-size instances; cache where efficient.

---

## Extended Examples

### Example: Resilient Payment Workflow (Saga)
- Steps: Create order → Reserve inventory → Pre-authorize payment → Confirm order → Capture payment → Ship.
- Failures:
  - Payment decline: compensate by releasing inventory, cancel order.
  - Shipment failed: refund or reattempt; notify customer.
- Guarantees:
  - Use idempotency keys per step.
  - Store saga state; retry with backoff; circuit-break external providers.

### Example: Rate Limiting in API Gateway
- Implement token bucket per API key (N tokens/sec, burst B). Store counters in Redis with TTL.
- Include `X-RateLimit-*` headers; return 429 on excess; provide status endpoint.

### Example: Real-time Chat Delivery
- WebSocket gateway → Kafka (per-room partition) → Chat service → Fanout to connected users.
- Handle backpressure by pausing reads per connection; buffer caps; drop oldest non-essential events.

---

## References
- Designing Data-Intensive Applications (Kleppmann)
- Site Reliability Engineering (Google SRE)
- The Datacenter as a Computer (Hennessy/Patterson)
- Raft paper; Paxos Made Simple
- CAP Twelve Years Later
- Kafka, Redis, Cassandra, Postgres docs


