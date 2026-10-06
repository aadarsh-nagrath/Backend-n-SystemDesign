# Monoliths, Modular Monoliths, and Microservices

> [`system-desing-websocket-focused.md`](../system-desing-websocket-focused.md) §1.1 and [`system-design/notes/concepts.md`](../system-design/notes/concepts.md) §10 introduce this trade-off. Here it's covered as a full decision framework, with decomposition techniques, communication styles, data ownership, migration strategies, and the failure modes of each choice.

## Table of Contents
1. [The Spectrum of Architectures](#spectrum)
2. [The Monolith (and Why It's Underrated)](#monolith)
3. [The Modular Monolith](#modular)
4. [Microservices: What They Actually Buy You](#micro)
5. [What Microservices Cost](#cost)
6. [Decision Framework](#decision)
7. [Finding Service Boundaries](#boundaries)
8. [Data Ownership: Database per Service](#data)
9. [Communication: Sync vs Async, and Coupling](#communication)
10. [Cross-Cutting Concerns in Microservices](#crosscutting)
11. [Anti-Patterns: Distributed Monolith and Friends](#anti)
12. [Migrating: Strangler Fig and Other Patterns](#migration)
13. [Serverless and Other Shapes](#serverless)
14. [Team Topology and Conway's Law](#conway)
15. [Interview Questions](#qa)

---

## 1. The Spectrum {#spectrum}

```
Big ball of mud ── Layered monolith ── Modular monolith ── Service-based (a few coarse services) ── Microservices ── Nanoservices / function-per-endpoint
     (avoid)                               (sweet spot for most)                                       (needs org scale)        (usually too far)
```
Architecture is about managing **coupling** and **cohesion** at the level of code, data, deployment, and teams. "Microservices vs monolith" is mostly a **deployment and team-autonomy** decision. Good modularity is required in either case.

---

## 2. The Monolith {#monolith}

One deployable unit (one codebase, one process type, usually one database).

Strengths:
- **Simplicity**: one build, one deploy, one place to debug, and stack traces across the whole request.
- **Performance**: in-process function calls (nanoseconds) vs network calls (milliseconds + serialization + failure modes).
- **Transactions**: ACID across the whole domain with one DB transaction.
- **Refactoring**: IDEs can rename across the codebase, and you can change module boundaries freely.
- **Operational cost**: few moving parts, cheap infrastructure, small platform needs.
- Proven at scale: Shopify (a modular Rails monolith handling massive peaks), Stack Overflow (a handful of servers), Basecamp, GitHub (largely a monolith with extracted services), Instagram early on, Etsy.

Weaknesses (mostly emerge with **team size**, not traffic):
- Coordination overhead as dozens to hundreds of engineers commit to one codebase (merge conflicts, deploy queues, broken builds blocking everyone).
- Coupling creep without enforced module boundaries, leading to a "big ball of mud".
- Scaling is all-or-nothing (you scale the whole app, even if only one part is hot; usually fine, since horizontal scaling of a stateless monolith is easy).
- One technology stack.
- A failure such as a memory leak in one feature can take down everything (blast radius).
- Long build and test times as the codebase grows.

---

## 3. The Modular Monolith {#modular}

A single deployable whose code is divided into **well-defined modules with explicit boundaries**: each module owns its domain logic and data and exposes a public API (interfaces/facades), while its internals are inaccessible.

Rules:
- Modules communicate through **public interfaces or in-process events**, never by reaching into each other's internals or tables.
- **Each module owns its tables** (separate schema per module). No cross-module joins, or at most via read-only views/APIs. This makes later extraction possible.
- Enforce boundaries with tooling: **Packwerk** (Shopify, Ruby), **ArchUnit** (Java), Spring Modulith, Java modules (JPMS), .NET project references and analyzers, `import-linter` (Python), Nx module boundaries / dependency-cruiser (TS), Go `internal/` packages.
- Optionally use in-process async events (outbox + local dispatcher) between modules, to rehearse event-driven decoupling.

Benefits: most of the design benefits of microservices (cohesion, ownership, clear contracts) with none of the distributed-systems tax. When a module truly needs independent scaling or deployment, **extract it**, and the boundary already exists.

This is the recommended starting point for almost all new systems ("Monolith First", Martin Fowler).

---

## 4. Microservices {#micro}

Independently deployable services, each owning a business capability and its data, communicating over the network. They're typically owned by small autonomous teams (two-pizza teams).

What they buy you:
1. **Independent deployability**: a team ships without coordinating with 20 others. **This is the main benefit.**
2. **Team autonomy and scaling the organization**: clear ownership, so it scales to hundreds of engineers (Amazon, Netflix, Uber).
3. **Independent scaling** of hot components (search scaled separately from account settings).
4. **Fault isolation** (if done right: bulkheads, timeouts, degradation).
5. **Technology heterogeneity**: the right tool per service (use sparingly: every new stack costs operational knowledge).
6. Smaller codebases, easier for a team to understand.

---

## 5. What Microservices Cost {#cost}

- **Distributed systems complexity**: network failures, latency, partial failures, retries, idempotency, timeouts (see [`distributed-systems/`](../distributed-systems/)).
- **No cross-service transactions**: sagas, outbox, and eventual consistency (see [`messaging/04`](../messaging/04-event-driven-architecture-patterns.md)).
- **Data consistency and querying**: joins across services become API composition or replicated read models.
- **Operational overhead**: CI/CD per service, container orchestration, service discovery, config, secrets, observability (distributed tracing becomes mandatory), and on-call per service.
- **Testing**: integration and contract testing across services. It's hard to run "the whole system" locally.
- **Latency**: a request fanning out through 5 hops adds network time plus serialization.
- **Versioning**: API and event contract evolution, backward compatibility, deploy-order dependencies.
- **Platform team requirement**: someone builds the paved road (templates, CI, deploy, observability, service mesh).
- Cost: more infrastructure, more idle capacity, cross-AZ traffic charges.

Notable reversals: Amazon Prime Video's audio/video monitoring team moved a serverless/microservice pipeline to a monolith and cut costs 90% (2023). Segment went from microservices back to a monolith ("Goodbye Microservices", 2018). Many startups regret premature microservices.

---

## 6. Decision Framework {#decision}

Choose microservices **only** when most of these are true:
- [ ] Multiple teams (roughly > 3–5 teams / 30–50+ engineers) are blocked by coordinating on one deployable.
- [ ] Domain boundaries are well understood (you've been running the product long enough).
- [ ] Parts of the system have genuinely different scaling, availability, or compliance needs (e.g., a PCI-scoped payment service).
- [ ] You have (or will invest in) platform capabilities: CI/CD automation, containers/orchestration, observability, on-call maturity.
- [ ] The organization accepts eventual consistency in places.

Otherwise: **a modular monolith**, possibly plus a few extracted services for clear reasons (a CPU-heavy image pipeline, a PCI payment vault, a third-party integration worker).

Interview framing: "I'd start with a modular monolith with strict module boundaries and separate schemas. I'd extract services when team scaling or a specific non-functional requirement justifies it, beginning with modules that have clean boundaries and different scaling characteristics."

---

## 7. Finding Service Boundaries {#boundaries}

Bad boundaries are the most expensive mistake: chatty services, distributed transactions everywhere, and changes requiring coordinated deploys.

Techniques:
- **Domain-Driven Design bounded contexts** (see [`02-clean-hexagonal-ddd.md`](02-clean-hexagonal-ddd.md)): each context has its own ubiquitous language and model. "Product" in Catalog (descriptions, images) ≠ "Product" in Inventory (stock levels) ≠ "Product" in Pricing.
- **Business capabilities**: ordering, payments, shipping, catalog, identity, notifications.
- **Event storming**: a workshop that maps domain events on a timeline, then clusters them into aggregates and contexts.
- **Volatility / change frequency**: things that change together belong together (high cohesion). Things that change for different reasons should be split.
- **Data ownership**: who is the source of truth for this data?
- **Team ownership**: a service should be ownable by one team.
- **Transactional boundaries**: if two things must be strongly consistent together, keep them in one service.

Smells of wrong boundaries:
- Services that must always deploy together.
- A request that synchronously calls 6 services in a chain.
- Services sharing a database table.
- Frequent distributed transactions or sagas for basic operations.
- An "entity service" per noun (UserService, OrderService, ProductService as CRUD wrappers) with business logic scattered in orchestrators. Prefer capability-oriented services.

---

## 8. Data Ownership {#data}

**Database per service** (logical ownership; it can be separate schemas on a shared server early on):
- Only the owning service reads or writes its data directly. Others go through its API or consume its events.
- Lets services evolve schemas independently and choose fitting storage.

Patterns for cross-service data needs:
| Need | Pattern |
|---|---|
| Query combining data from several services | **API composition** (aggregator/BFF/GraphQL calls services and joins in memory) |
| Fast local reads of another service's data | **Replicated read model** via events (event-carried state transfer / CQRS) |
| Business transaction across services | **Saga** + outbox |
| Reporting across everything | CDC → warehouse (analytics shouldn't query services) |
| Reference data (countries, currencies) | Replicate or use a shared library/config |

**Shared database anti-pattern**: multiple services writing the same tables means hidden coupling (a schema change breaks other teams), no clear ownership, and contention. It's sometimes acceptable as a transitional step (strangler) with clear table ownership rules.

---

## 9. Communication {#communication}

| Style | Examples | Coupling | Use |
|---|---|---|---|
| Synchronous request/response | REST, gRPC, GraphQL | **Temporal coupling**: caller availability depends on callee | Queries needing immediate answers, commands needing an immediate result |
| Asynchronous messaging | Events (Kafka), commands (queues) | Loose in time | Notifications, workflows, fan-out, load leveling |
| Request/async-reply | Command queue + reply event or callback | Medium | Long-running operations |

Guidelines:
- **Minimize synchronous chains** (A → B → C → D): availability multiplies (0.999⁴ ≈ 0.996) and latencies add. Prefer one hop plus local data (replicated via events).
- **Choose the integration style per interaction**, not per system.
- gRPC internally (typed, fast, streaming) and REST/GraphQL at the edges is common.
- Use an API gateway/BFF for client-facing aggregation.
- Every sync call needs timeouts, retries with budgets, circuit breakers, and fallbacks (see [`distributed-systems/05-resilience-patterns.md`](../distributed-systems/05-resilience-patterns.md)).
- **Contracts**: OpenAPI / protobuf / AsyncAPI + schema registry, backward-compatible evolution, and consumer-driven contract tests (Pact).

---

## 10. Cross-Cutting Concerns {#crosscutting}

What every service needs (ideally provided by a **paved road**: service templates/chassis, platform libraries, or a mesh):
- Service discovery and load balancing (K8s Services, Consul, mesh).
- Configuration and secrets (see [`03-twelve-factor-config-and-deployment.md`](03-twelve-factor-config-and-deployment.md)).
- AuthN/AuthZ between services (mTLS identities, JWT propagation, token exchange), and authorization for user context passed downstream.
- Observability: structured logs with correlation IDs, RED metrics, distributed tracing (OpenTelemetry), and health endpoints (see [`observability/`](../observability/)).
- Resilience policies.
- API versioning and documentation.
- CI/CD pipelines, deployment strategies, feature flags.
- Ownership metadata (service catalog: **Backstage**, OpsLevel, Cortex) with owner, on-call, runbooks, dependencies, and SLOs.

---

## 11. Anti-Patterns {#anti}

- **Distributed monolith**: services that are tightly coupled (shared DB, synchronous chains, lockstep deploys, shared domain libraries with business logic). You get all the costs of microservices with none of the benefits. **The most common outcome of premature microservices.**
- **Nanoservices**: services so small that overhead dominates (one service per function with a chatty network).
- **Shared libraries with business logic**: coupling via code. A change requires redeploying everything. (Shared *infrastructure* libraries like logging and tracing are fine.)
- **Entity services + anemic orchestrators**: logic lives in a giant orchestrator calling CRUD services.
- **Synchronous everything**: no isolation, cascading failures.
- **Microservices for a 5-person team**.
- **No platform investment**: each team reinvents deploys, logging, and alerting.
- **"Microservices will fix our code quality"**: a ball of mud distributed over the network is worse.

---

## 12. Migrating from Monolith {#migration}

**Strangler fig pattern** (Fowler): gradually replace functionality by routing specific requests to new services while the monolith handles the rest, until the monolith "dies" (or shrinks to a core).
1. Put a **routing facade** (API gateway or proxy) in front of the monolith.
2. Pick a seam: a capability with clear boundaries, high change rate or scaling needs, and low coupling (notifications, search, auth are common first picks).
3. Build the new service, with its data synced from the monolith (CDC or dual write via events), then shift traffic (feature flags, % rollouts, shadow traffic to compare responses).
4. Move write ownership of the data to the new service, and the monolith now calls or consumes from the service.
5. Delete the old code path.

Supporting patterns:
- **Branch by abstraction**: introduce an interface inside the monolith, implement it with a client to the new service, and toggle.
- **Anti-corruption layer (ACL)**: translate between the legacy model and the new service's model so legacy concepts don't leak in.
- **Change data capture** from the legacy DB to feed new services.
- **Parallel run / shadowing**: run old and new implementations and compare outputs before switching (Scientist library by GitHub).
- **Decompose the database last** (or carefully alongside): first split code, then split schemas (separate tables per module), then separate databases.

Before extracting: modularize the monolith first. If you can't draw clean module boundaries in-process, you won't get them over the network.

---

## 13. Serverless and Other Shapes {#serverless}

- **Functions-as-a-Service** (AWS Lambda, Cloud Functions/Run, Azure Functions, Cloudflare Workers):
  - Pros: no servers, scale to zero, pay per use, built-in scaling, great for event-driven glue, spiky workloads, cron, webhooks, and file processing.
  - Cons: cold starts (mitigated: provisioned concurrency, SnapStart, lighter runtimes), execution limits (15 min Lambda), connection management to DBs (proxies), vendor lock-in, local testing difficulty, cost at high steady throughput (containers become cheaper), and distributed-tracing complexity.
- **Serverless containers** (Cloud Run, Fargate, Azure Container Apps): containers with autoscaling, a good middle ground.
- **Service-based architecture**: a few coarse-grained services (5–10) around major domains, often sharing infrastructure. It's a pragmatic step between a monolith and microservices.
- **Event-driven architecture** with a broker as the backbone.
- **Space-based / actor-based** (Akka, Orleans, Durable Objects): stateful actors partitioned across a cluster, for high-contention in-memory state (gaming, IoT, collaboration).
- **Micro-frontends**: the frontend counterpart (independent UI deployments), with similar trade-offs.

---

## 14. Team Topology and Conway's Law {#conway}

**Conway's Law** (1967): "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations."
- **Inverse Conway maneuver**: shape teams to match the desired architecture (team per bounded context).
- *Team Topologies* (Skelton & Pais): **stream-aligned teams** (own a business flow end to end), **platform teams** (provide self-service internal platforms), **enabling teams** (coach), **complicated-subsystem teams** (deep specialists, e.g., a video codec team). Optimize for team cognitive load and fast flow.
- Each service should have exactly one owning team. A team can own several services.
- "You build it, you run it" (Werner Vogels): owning teams carry the pager.

---

## 15. Interview Questions {#qa}

1. Monolith vs microservices: what are the real trade-offs? When would you choose each?
2. What is a modular monolith and how do you enforce boundaries?
3. How do you decide service boundaries?
4. What is a distributed monolith and how do you recognize one?
5. How do microservices handle transactions and queries that span services?
6. Walk through migrating a feature out of a monolith with the strangler fig pattern.
7. What platform capabilities must exist before adopting microservices?
8. What is Conway's Law and how does it influence architecture?
9. When is serverless a good or bad fit?
10. Why might a company move from microservices back to a monolith?
