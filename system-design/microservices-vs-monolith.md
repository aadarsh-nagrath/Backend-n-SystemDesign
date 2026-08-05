# Microservices vs Monolith

An architectural decision about how you split (or don't split) a system into deployable
units. It gets treated as a religious debate online; in practice it's a trade-off with
real costs on both sides, and the "right" answer depends on team size, org structure,
and what's actually causing pain today.

## TL;DR
- **Monolith**: one deployable unit, one codebase, one database (usually). Simple to
  build, run, and debug. Scaling means scaling the whole thing.
- **Microservices**: the system is split into independently deployable services, each
  owning its own data, communicating over the network. Enables independent scaling and
  deployment — at the cost of network latency, distributed transactions, and real
  operational complexity.
- Splitting services along **team/deploy boundaries without decoupling the data and
  logic** gives you a **distributed monolith** — the worst of both worlds. This is the
  single most common microservices failure mode.
- Most companies should start with a monolith (ideally a *modular* one) and split out
  services only when there's a concrete reason — a scaling bottleneck, a team-ownership
  conflict, or a need for independent deploy cadence.
- Conway's Law is not a suggestion: your system's architecture will mirror your org
  chart whether you plan it or not. Plan it.

## The problem it solves / why it exists

A single codebase deployed as one unit is the default starting point for almost every
system. It works great until the organization or the system outgrows it:

- **Team scaling**: 50 engineers committing to one codebase means constant merge
  conflicts, a shared CI pipeline that's slow for everyone, and a release process where
  one team's bug blocks everyone else's deploy.
- **Resource scaling**: if your image-processing endpoint is CPU-heavy and your user
  login endpoint is I/O-light, a monolith forces you to scale both together — you're
  paying to over-provision the light parts to keep up with the heavy part.
- **Independent deployability**: a team wants to ship fixes to checkout ten times a day
  without waiting on the catalog team's release train.
- **Technology fit**: a recommendation engine might be better served by Python/ML
  tooling while the core transactional system is fine in Java — a monolith locks you
  into one stack.

Microservices exist to address these problems. They also introduce new ones — this note
is mostly about that trade-off.

## The same system, drawn two ways

Take a typical e-commerce app: users browse a catalog, add items to a cart, check out,
pay, and get order confirmations with inventory updates.

### As a monolith

```
┌─────────────────────────────────────────────┐
│              E-Commerce App                 │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ │
│  │  Catalog  │ │   Cart    │ │  Checkout  │ │
│  │  module   │ │  module   │ │   module   │ │
│  └───────────┘ └───────────┘ └────────────┘ │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ │
│  │  Payments │ │ Inventory │ │   Orders   │ │
│  │  module   │ │  module   │ │   module   │ │
│  └───────────┘ └───────────┘ └────────────┘ │
│         all in one process, one deploy       │
└───────────────────┬───────────────────────┘
                     │
              ┌──────┴──────┐
              │  One shared │
              │  database   │
              └─────────────┘
```

One Git repo, one build, one deployable artifact (a JAR, a Docker image, whatever), one
database. `checkout()` calling `reserveInventory()` is a normal in-process function
call. A single `BEGIN; ... COMMIT;` transaction can atomically deduct inventory, create
an order, and charge a saved payment method — real ACID guarantees, no extra effort.

### As microservices

```
   Client
     │
┌────▼─────┐
│   API    │
│ Gateway  │
└────┬─────┘
     │
 ┌───┼────────┬───────────┬────────────┬───────────┐
 │            │           │            │            │
┌▼───────┐ ┌──▼─────┐ ┌───▼──────┐ ┌───▼──────┐ ┌───▼──────┐
│Catalog │ │  Cart  │ │ Checkout │ │ Payments │ │Inventory │
│Service │ │Service │ │ Service  │ │ Service  │ │ Service  │
│  own   │ │  own   │ │   own    │ │   own    │ │   own    │
│  DB    │ │  DB    │ │   DB     │ │   DB     │ │   DB     │
└────────┘ └────────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
                            │            │            │
                            └───event bus / queue─────┘
                          (order.created, payment.captured...)
```

Each box is its own codebase, its own deploy pipeline, its own database, potentially
its own language/runtime. Checkout no longer *calls* inventory directly in-process — it
makes a network call (REST/gRPC) or publishes an event and waits (or doesn't wait) for
a response. "Atomically deduct inventory and create an order" is no longer a single DB
transaction — it's a distributed operation across two services with their own storage,
usually solved with a saga (see below) instead of a two-phase commit.

## Monolith: pros and cons

**Pros**
- **Simplicity** — one codebase to understand, one place to `grep`, one build.
- **Easy local dev** — `git clone && run` gets the whole system up. No need to stub out
  eight other services to test one feature end-to-end.
- **Real ACID transactions** — a `checkout` that touches inventory, orders, and
  payments can be one database transaction. No sagas, no eventual consistency to reason
  about.
- **No network overhead** — a function call is nanoseconds; a network call is
  milliseconds at best, and can fail in ways a function call can't (timeout, partial
  failure, retry storms).
- **Easier debugging** — one stack trace spans the whole request. No hopping between
  five services' logs (pre-distributed-tracing) to reconstruct what happened.
- **Simpler testing** — integration tests spin up one process, not a constellation of
  services and their fakes.

**Cons**
- **Scale the whole thing to scale one part** — if only the search feature is
  CPU-bound, you still replicate the entire app (including the parts that aren't
  bottlenecked) to add capacity.
- **Deployment risk** — one bad change anywhere can take down the whole app; a full
  redeploy is required even for a one-line fix in an unrelated module.
- **Tech stack lock-in** — the whole app is one language/runtime; adopting a better
  tool for one subproblem means rewriting or awkwardly embedding it.
- **Team contention** — as headcount grows, a shared codebase and shared release
  process becomes the bottleneck (merge conflicts, slow CI, coordinated release
  trains).
- **Blast radius of coupling** — over time, without discipline, module boundaries
  erode and everything ends up calling everything (the "big ball of mud").

## Microservices: pros and cons

**Pros**
- **Independent scaling** — replicate only the inventory service under load; leave
  catalog at 2 replicas.
- **Independent deployment** — the checkout team ships fixes without waiting on
  catalog's release train, and a bad deploy is contained to one service.
- **Technology diversity** — use Go for a latency-sensitive service, Python for an
  ML-heavy one, without forcing the whole org onto one stack.
- **Fault isolation** — (if done right) the recommendation service crashing doesn't
  take down checkout, especially with circuit breakers and graceful degradation.
- **Team autonomy** — each team owns a service end-to-end: code, deploy, on-call,
  database schema. Matches how larger orgs actually want to operate (see Conway's Law
  below).

**Cons**
- **Network latency and reliability** — every service-to-service call can be slow,
  time out, or fail outright, in ways an in-process call never does. A request that
  used to be one function call chain can now involve five network hops.
- **Distributed transactions are hard** — no more `BEGIN/COMMIT` across services. You
  need sagas, eventual consistency, or careful choreography — and you have to design
  for the case where step 3 of 5 fails and steps 1-2 already committed.
- **Operational complexity** — you now need service discovery, centralized logging,
  distributed tracing, per-service monitoring/alerting, a CI/CD pipeline per service,
  and (usually) container orchestration. This is real infrastructure investment before
  you ship your first feature.
- **Testing complexity** — end-to-end tests need multiple services running together
  (or convincing fakes); a bug can live purely in the interaction between two services
  that each pass their own unit tests.
- **Data consistency across services** — the catalog service's view of a product and
  the search service's cached copy of it *will* drift; you need to design for staleness
  and reconciliation, not assume a single source of truth is always in sync.
- **Debugging is harder** — one user request can now touch 6 services; without
  distributed tracing (e.g. OpenTelemetry + Jaeger) reconstructing "what happened" is
  brutal.

| | Monolith | Microservices |
|---|---|---|
| Deployment | One unit, all-or-nothing | Independent per service |
| Scaling | Whole app scales together | Scale only what's hot |
| Transactions | Real ACID | Sagas / eventual consistency |
| Local dev | `run` and you're done | Need N services or mocks |
| Debugging | One stack trace | Distributed tracing required |
| Team ownership | Shared codebase | One team per service (ideally) |
| Network calls | None (in-process) | Every cross-service call |
| Tech stack | One for everything | Can vary per service |
| Operational overhead | Low | High (discovery, mesh, observability) |

## The distributed monolith anti-pattern

This is the trap teams fall into most often, and it's worth understanding in detail
because it's not a strawman — it's the default outcome of splitting a codebase without
also decoupling the underlying design.

**What it looks like**: the system has been physically split into multiple deployable
services (so it "looks like microservices" on an architecture diagram), but:

- Services share a single database (or worse, read/write each other's tables directly).
- Services are released together, in lockstep, because their APIs/schemas are so
  tightly coupled that deploying one without the others breaks something.
- A single logical change (e.g. "add a field to Order") requires coordinated PRs and
  deploys across 4 services.
- Service A calls Service B synchronously, which calls Service C synchronously, in a
  deep chain — so a slow or down C takes down A even though they're "independent
  services."
- There's no clear owner per service — the same team touches all of them, so the
  organizational benefit of microservices (team autonomy) doesn't materialize either.

**Why it's worse than a monolith**: you now pay *all* the costs of microservices —
network latency, operational complexity (separate deploys, separate logging,
distributed debugging) — while getting *none* of the benefits — no independent
scaling (schema coupling forces lockstep changes), no independent deployment (release
coordination is still required), no fault isolation (synchronous call chains propagate
failures just as badly as in-process calls, but slower and with more failure modes).

**How it happens**: usually a monolith gets split along a *technical* or *convenient*
boundary — "let's move the auth code to its own service" — without first identifying a
true bounded context with its own data ownership. The split is done for
organizational optics ("we're doing microservices now") rather than because a real
domain boundary and scaling/ownership need existed.

**The fix**: before splitting, make sure each candidate service (a) owns its own data
exclusively — no other service touches its tables directly, only its API — and (b) can
be deployed independently without coordinating with other teams' release schedules.
If you can't say yes to both, you have a distributed monolith, not microservices.

## Service decomposition strategies

### By business capability / domain-driven design (recommended)

Split along **bounded contexts** — a DDD concept meaning a self-contained area of the
business with its own model, its own language ("ubiquitous language"), and its own data.
In the e-commerce example: Catalog, Cart, Checkout, Payments, Inventory, Shipping are
each bounded contexts — each has a coherent, independent reason to change, and each
naturally owns its own data.

Signals you've found a good boundary:
- The team that owns it can describe what it does without referencing another service's
  internals.
- It has its own persistent data that nothing else needs to write to directly.
- It changes for reasons unrelated to why neighboring services change (different rate of
  change, different scaling needs, different team).

### By technical layer (anti-pattern)

Splitting into a "presentation service," "business logic service," and "data access
service" sounds like separation of concerns but is a well-known anti-pattern for
microservices: almost every real user request needs to hit all three layers, so you get
a synchronous chain (presentation → logic → data) for nearly everything, maximizing
network calls without gaining any independent scalability or deployability — the layers
change together, deploy together, and are owned by the same people in practice. This is
a recipe for a distributed monolith. Layered architecture is a fine pattern *inside* a
single service — it's the wrong axis to split *between* services.

## Communication patterns between services

### Synchronous: REST / gRPC

The caller sends a request and blocks (or awaits) until a response comes back. Simple
mental model, but couples the caller's availability to the callee's — if the downstream
service is slow or down, the caller is stuck too (unless you add timeouts, retries, and
circuit breakers).

- **REST over HTTP** — ubiquitous, human-debuggable, language-agnostic. See
  [`api/restapi.md`](../api/restapi.md) for the full rundown of methods, status codes,
  and design practices. Good default for public-facing and simpler internal APIs.
- **gRPC** — binary protocol over HTTP/2, code-generated clients/servers from a
  `.proto` schema, supports streaming. Lower latency and stronger typing than REST/JSON,
  favored for internal service-to-service calls in latency-sensitive systems (this is
  what Netflix, Google, and most large-scale internal platforms use internally). See
  [`api/grpc.md`](../api/grpc.md) for the protocol details and code examples.

Use synchronous calls when the caller genuinely needs an immediate answer to proceed
(e.g. "is this payment method valid" during checkout).

### Asynchronous: message queues / events

The caller publishes a message/event and moves on; one or more consumers process it
independently, potentially much later. Decouples the caller's uptime from the
consumer's — if the inventory service is down, orders can still be placed and the
inventory update just catches up once it's back.

This is the backbone of how distributed transactions actually get handled in
microservices (see sagas below) and how services stay loosely coupled. See
[`system-design/message-queues.md`](./message-queues.md) for queue mechanics, delivery
guarantees, and broker comparisons (Kafka, RabbitMQ, SQS).

Use asynchronous messaging when:
- The caller doesn't need an immediate result (e.g. "send a confirmation email").
- Multiple services need to react to the same event (order placed → charge payment,
  reserve inventory, notify shipping — all independently, all triggered off one event).
- You want to decouple lifetimes — the consumer can be down without blocking the
  producer.

**Trade-off**: asynchronous flows are eventually consistent and harder to trace/debug
(no simple call stack; you need to follow a message through a broker across services).

### Handling distributed transactions: the saga pattern

Since a single ACID transaction across services isn't possible, sagas break a business
transaction into a sequence of local transactions, each in one service, coordinated via
events or an orchestrator, with **compensating transactions** to undo prior steps if a
later one fails.

Example — placing an order:
1. Order service creates order (status: `pending`).
2. Inventory service reserves stock. On success, emits `inventory.reserved`.
3. Payment service charges the customer. On success, emits `payment.captured`.
4. Order service marks order `confirmed`.

If step 3 fails (card declined), a compensating action fires: inventory service releases
the reservation (`inventory.released`), and the order is marked `failed` — undoing step
2 instead of relying on a rollback that doesn't exist across service boundaries.

Two coordination styles:
- **Choreography** — each service listens for the previous service's event and reacts;
  no central coordinator. Simple for a few steps, hard to trace as the chain grows.
- **Orchestration** — a dedicated orchestrator service calls each step and explicitly
  handles failure/compensation. More visible and debuggable, but the orchestrator
  becomes a critical, complex component itself.

## API Gateway pattern

A single entry point that sits in front of all the backend services and handles
cross-cutting concerns so individual services don't each have to reimplement them:

- **Routing** — maps `/products/*` to the Catalog service, `/orders/*` to the Order
  service, etc.
- **Authentication/authorization** — validates the JWT/session once, at the edge,
  instead of every service re-implementing auth.
- **Rate limiting** — enforced centrally, protecting all backend services uniformly.
- **Request aggregation** — sometimes combines calls to multiple services into one
  client-facing response (avoiding "chatty" client-to-backend traffic), though this can
  also live in a dedicated Backend-for-Frontend (BFF) layer.
- **TLS termination, request/response logging, response caching.**

Without a gateway, every client needs to know the address of every service, and every
service needs to implement its own auth/rate-limiting — duplicated effort and
inconsistent enforcement. Examples: Kong, AWS API Gateway, NGINX, Envoy-based gateways,
or a hand-rolled BFF.

⚠️ The gateway can become a single point of failure and a bottleneck if not scaled and
made highly available itself — and a "smart gateway" that starts containing business
logic slowly turns back into a monolith at the edge.

## Service mesh (concept)

At scale (dozens+ of services), cross-cutting service-to-service concerns — retries,
timeouts, mTLS encryption, load balancing, observability (who's calling whom, at what
latency, with what error rate) — get tedious and inconsistent to implement per service.

A **service mesh** (Istio, Linkerd) solves this by deploying a lightweight **sidecar
proxy** (Envoy, in Istio's case) alongside every service instance. All network traffic
in and out of the service goes through its sidecar instead of directly over the wire.
The sidecars are centrally configured, so you get uniform:

- **mTLS** between all services, without any application code changes.
- **Retries, timeouts, circuit breaking** applied consistently at the infrastructure
  layer instead of reimplemented in every service's code.
- **Observability** — the mesh sees every request between every pair of services, so
  you get consistent metrics/tracing "for free."
- **Traffic shaping** — canary releases, traffic splitting (e.g. 5% of traffic to a new
  version) without changing application code.

This is advanced, operationally heavy infrastructure — it's usually only worth adopting
once you have enough services that manually keeping their networking concerns
consistent has become its own problem. Most teams with under ~10-15 services don't need
one yet.

## Circuit breaker pattern

Protects a service from cascading failure when a downstream dependency is slow or down.
Without one, a struggling downstream service can drag its callers down too — threads
pile up waiting on a slow call, exhausting the caller's own resources and taking it down
as well (this is exactly how outages cascade across a service graph).

A circuit breaker wraps a call to a downstream service and tracks its failure rate,
moving through three states:

- **Closed** — normal operation. Requests pass through. Failures are counted; if the
  failure rate crosses a threshold (e.g. 50% of the last 20 calls failed), the breaker
  **trips** and moves to Open.
- **Open** — requests fail immediately (or fall back to a default/cached response)
  *without even attempting* the downstream call, for a cooldown period. This protects
  both the caller (no more wasted waiting) and the struggling downstream service (no
  more added load while it's already unhealthy).
- **Half-Open** — after the cooldown, the breaker allows a small number of trial
  requests through. If they succeed, it closes again (resume normal traffic); if they
  fail, it reopens and the cooldown restarts.

```
        failure rate > threshold
  CLOSED ───────────────────────► OPEN
    ▲                               │
    │                        cooldown timer expires
    │  trial requests succeed       │
    └──────────── HALF-OPEN ◄───────┘
                     │
              trial requests fail
                     │
                     ▼
                   OPEN
```

**Concrete example**: the Checkout service calls Payments over HTTP. Payments starts
timing out under load. Without a circuit breaker, every checkout request waits the full
timeout (say 30s) before failing — Checkout's thread pool fills up with stuck requests,
and Checkout itself becomes unresponsive to everything, not just payment-related
requests. With a circuit breaker (e.g. using Resilience4j, Polly, or Hystrix-style
libraries), after enough failures the breaker trips to Open — subsequent calls fail
instantly (or fall back to "payment processing delayed, we'll email you") instead of
hanging, keeping Checkout responsive for everything else while Payments recovers.

## The strangler fig pattern

A practical, low-risk strategy for migrating a monolith to microservices incrementally,
named after the strangler fig vine that grows around a host tree and gradually replaces
it — you never do a risky big-bang rewrite.

**How it works**:
1. Put a routing layer (API Gateway or reverse proxy) in front of the monolith.
2. Pick one small, well-bounded piece of functionality (e.g. "product search").
3. Build it as a new standalone service.
4. Update the routing layer to send *only* traffic for that functionality to the new
   service; everything else still goes to the monolith.
5. Once the new service is proven in production, remove that code from the monolith.
6. Repeat for the next piece, until the monolith is "strangled" down to nothing (or
   down to a manageable core that's fine to leave as-is).

```
Before:              During:                    After:
Client               Client                      Client
  │                    │                            │
  ▼                    ▼                            ▼
Monolith          ┌─Router─┐                    ┌─Router─┐
                   │        │                    │        │
              (most) ▼   ▼ (search)          ▼        ▼        ▼
              Monolith  Search Svc      Monolith*  Search Svc  Cart Svc
                                        (shrinking)
```

**Why it's the practical default** for migrating an existing system (as opposed to
greenfield microservices): it never requires a big-bang cutover, each step is
independently testable and revertible (route traffic back to the monolith if the new
service misbehaves), and the business keeps shipping features throughout instead of
freezing for a rewrite. This is how most real-world "monolith to microservices"
migrations (including Amazon's and Shopify's incremental extractions) actually happen —
not a rewrite from scratch.

⚠️ The hard part in practice isn't routing — it's data. If the strangled functionality
needs data still owned by the monolith's database, you either need to sync it
(dual-write, CDC) or give the new service temporary read access while you migrate
ownership — this data migration is usually the slow part, not the code split.

## When to actually choose which — a decision framework

Ignore the hype. Ask these questions:

**Team size and structure**
- Small team (< ~10-15 engineers)? A monolith is almost always right. You don't have
  enough people to separately own, deploy, and operate multiple services, and the
  coordination overhead of microservices will slow you down, not speed you up.
- Multiple independent teams that need to ship on their own schedule without
  coordinating releases? That's a real signal for splitting — but split along the team
  boundaries you actually have, not aspirational ones.

**Conway's Law**
> "Organizations which design systems ... are constrained to produce designs which are
> copies of the communication structures of these organizations." — Melvin Conway

Your system's architecture *will* mirror your org chart, whether you design it that way
or not. If you have one team, you'll end up with a de-facto monolith even if you draw
five boxes on an architecture diagram (see: distributed monolith). If you want
independent services, you need independent teams to own them — the reverse (org
structure following architecture, sometimes called the "inverse Conway maneuver") is
also a valid deliberate strategy: restructure teams around the service boundaries you
want first, and the architecture tends to follow.

**Actual scaling needs**
- Is there a real, measured bottleneck in one part of the system that requires scaling
  independently of the rest? That's a legitimate driver.
- Is the "we need to scale" reasoning hypothetical ("we might need to scale this
  someday")? That's premature — you can scale a well-written monolith (more replicas,
  a read replica, caching, a faster host) further than most teams expect before it
  becomes the actual constraint.

**Domain clarity**
- Do you understand the domain boundaries well enough to know where the seams should
  be? Splitting *before* the domain model has stabilized means you'll split it wrong
  and pay to re-merge or re-split later. It's often cheaper to build a modular monolith
  first, let the boundaries prove themselves under real usage, and extract services
  once they're clear (this is close to what Shopify did — see below).

**Rule of thumb**: start with a monolith. Extract a service when you have a concrete,
current (not hypothetical) reason — a genuine independent-scaling need, a genuine
independent-team-ownership need, or a genuine independent-deploy-cadence need — not
because "microservices are what real companies do."

## Real-world examples

- **Netflix** — one of the most cited microservices success stories, and for good
  reason: they moved from a monolith to hundreds of microservices starting around 2009,
  driven by a real need (massive, globally distributed scale, hundreds of engineering
  teams, extremely high availability requirements). They also built much of the
  supporting tooling the rest of the industry now uses or copies — Hystrix (circuit
  breakers), Eureka (service discovery), Zuul (API gateway). Netflix's story is often
  cited as "the reason to do microservices," but it's worth noting the scale (200M+
  subscribers, thousands of engineers) that justified the investment — most companies
  are nowhere near that scale.

- **Amazon's "two-pizza teams"** — Amazon's internal mandate that every team should be
  small enough to be fed with two pizzas (roughly 6-10 people), each owning a service
  end-to-end (code, deploy, on-call) via well-defined APIs, with no direct access to
  other teams' data stores. This is Conway's Law applied deliberately — the
  organizational structure (small autonomous teams) was designed specifically to
  produce a microservices-shaped architecture, not the other way around. This is also
  the origin of the internal culture that led to AWS itself.

- **Shopify's modular monolith** — a genuinely important counter-example. Shopify
  runs its core commerce platform as a single large Ruby on Rails monolith (still true
  as of their public engineering writeups), deliberately choosing *not* to split into
  microservices despite massive scale. Instead they invested heavily in enforcing
  **modularity within the monolith** — using tooling (their "Packwerk" gem) to enforce
  boundaries between internal modules, preventing the "everything calls everything"
  decay that kills most monoliths, while keeping the operational simplicity of a single
  deployable unit. It's a strong reminder that "monolith" and "unmaintainable big ball
  of mud" are not the same thing — a well-modularized monolith can scale to enormous
  size and traffic (Shopify handles some of the largest e-commerce traffic spikes in
  the world, e.g. Black Friday) without needing a microservices rewrite.

## Common pitfalls

⚠️ **Splitting before the domain model is understood** — you'll get the boundaries
wrong, and un-splitting (merging services back, or re-drawing boundaries) is more
painful than not splitting in the first place.

⚠️ **Shared databases between "independent" services** — the moment two services read
or write the same tables directly, you've lost independent deployability; a schema
change in one breaks the other. This is the core mechanic of the distributed monolith.

⚠️ **Synchronous call chains that are too deep** — service A calls B calls C calls D
synchronously; D being slow makes A slow, with no isolation. Favor async messaging or
add circuit breakers/timeouts at each hop.

⚠️ **No distributed tracing** — without it (OpenTelemetry, Jaeger, Zipkin), debugging a
multi-service request becomes archaeology. Invest in this *before* you need it in an
incident, not during one.

⚠️ **Underestimating operational cost** — CI/CD per service, monitoring/alerting per
service, on-call rotations, service discovery, secrets management — this is real,
ongoing engineering investment that doesn't show up in a "look how clean our
architecture diagram is" pitch.

⚠️ **Doing it for resume-driven development / hype** — "we should use microservices"
without a concrete driving problem is a common way engineering orgs inflict real,
lasting operational pain on themselves for no measurable benefit.

⚠️ **Ignoring data consistency** — assuming a search index, a cache, and a source-of-
truth service all agree at all times. They won't. Design explicitly for staleness
windows and reconciliation instead of hoping it doesn't happen.

## Quick reference

| Signal | Lean toward |
|---|---|
| Small team, early-stage product | Monolith |
| Domain boundaries still unclear | Monolith (modular) |
| Need ACID transactions across the whole flow | Monolith |
| Multiple teams need independent deploy cadence | Microservices |
| One component has genuinely different scaling needs | Microservices (extract that one) |
| No investment yet in observability/CI infra for N services | Monolith, or invest first |
| Migrating an existing large monolith | Strangler fig, incrementally |
| Downstream dependency might fail/slow down | Circuit breaker regardless of architecture |

## Further reading
- Martin Fowler — [MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
  and [Microservices](https://martinfowler.com/articles/microservices.html)
- Sam Newman — *Building Microservices* (O'Reilly)
- Eric Evans — *Domain-Driven Design* (bounded contexts, ubiquitous language)
- [`api/restapi.md`](../api/restapi.md) — synchronous REST communication between services
- [`api/grpc.md`](../api/grpc.md) — binary RPC for internal service-to-service calls
- [`system-design/message-queues.md`](./message-queues.md) — asynchronous messaging,
  delivery guarantees, broker comparisons
- Shopify Engineering blog — "Deconstructing the Monolith" and related modular monolith
  writeups
- Netflix Technology Blog — Hystrix, Eureka, and the microservices migration writeups
