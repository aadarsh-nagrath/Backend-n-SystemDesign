# Load Balancers, Reverse Proxies, API Gateways, and Service Meshes

> Code for LB algorithms lives in [`system-design/awesome-notes/implementations/`](../system-design/awesome-notes/implementations/). Nginx/Apache details: [`web-servers/`](../web-servers/). This note covers the architecture and the trade-offs.

## Table of Contents
1. [Forward Proxy vs Reverse Proxy](#proxies)
2. [Why Load Balance](#why)
3. [L4 vs L7 Load Balancing](#layers)
4. [Load Balancing Algorithms](#algorithms)
5. [Health Checks](#health)
6. [Session Affinity (Sticky Sessions)](#sticky)
7. [TLS Termination, Passthrough, Re-encryption](#tls)
8. [Connection Draining and Zero-Downtime Deploys](#draining)
9. [Global Load Balancing (DNS, Anycast, GSLB)](#global)
10. [Client-Side Load Balancing](#clientside)
11. [High Availability of the Load Balancer Itself](#ha)
12. [API Gateways](#gateway)
13. [Service Mesh and Sidecars](#mesh)
14. [Product Landscape](#products)
15. [Interview Questions](#qa)

---

## 1. Forward vs Reverse Proxy {#proxies}

- **Forward proxy**: acts **for clients** (corporate egress proxy, Squid, VPN-ish anonymizers). The server sees the proxy. Uses: egress control, caching, filtering, fixed egress IPs for webhooks/allowlists.
- **Reverse proxy**: acts **for servers** (Nginx, Envoy, HAProxy, Traefik, Caddy, cloud LBs). Clients think they're talking to the origin. Uses: load balancing, TLS termination, caching, compression, routing, rate limiting, WAF, hiding topology, request buffering (protecting slow app servers from slow clients).

---

## 2. Why Load Balance {#why}

- **Scalability**: spread load across N instances (horizontal scaling).
- **Availability**: route around failed instances (health checks).
- **Maintainability**: drain instances for deploys and patches.
- **Flexibility**: canary/blue-green routing, A/B tests, multi-version.
- **Security**: single entry point, TLS policy, WAF, DDoS absorption.

Places LBs appear: edge (CDN/global LB) → regional L4/L7 LB → ingress controller → service-to-service (mesh/client-side) → database proxies (PgBouncer/ProxySQL).

---

## 3. L4 vs L7 {#layers}

| | L4 (transport) | L7 (application) |
|---|---|---|
| Sees | IPs, ports, TCP/UDP | HTTP method, path, headers, cookies, gRPC service/method, SNI |
| Routing | By connection (5-tuple hash) | **Per request**: path-based (`/api` → svc A), host-based, header/canary routing |
| Performance | Very high (millions of conns), low latency; can be done in kernel/eBPF/hardware | More CPU (parse HTTP, TLS) |
| TLS | Passthrough (or terminate in some) | Terminates TLS (needs certs) |
| Features | Simple balancing, preserve client IP (with proxy protocol/DSR) | Retries, timeouts, header manipulation, compression, caching, auth, rate limiting, WAF, observability per request |
| Long-lived conns (HTTP/2, gRPC, WebSocket) | One connection → one backend for its lifetime (imbalance) | Balances individual requests/streams |
| Examples | AWS NLB, GCP Network LB, LVS/IPVS, Maglev, Katran (eBPF), HAProxy (TCP mode) | AWS ALB, GCP HTTP(S) LB, Nginx, Envoy, HAProxy (HTTP mode), Traefik, Cloudflare |

**Preserving client IP**: L7 proxies add `X-Forwarded-For`. L4 proxies either preserve the source IP (transparent, DSR) or use the **PROXY protocol** (HAProxy-defined header prepended to the TCP stream, supported by NLB, Nginx, Postgres via pgbouncer…).

**Direct Server Return (DSR)**: the LB forwards inbound packets, and backends reply **directly** to the client, bypassing the LB on the (much larger) response path. Used by Facebook Katran, GitHub GLB, and Google Maglev for massive scale.

---

## 4. Algorithms {#algorithms}

| Algorithm | How | Good for | Pitfalls |
|---|---|---|---|
| **Round robin** | Rotate | Homogeneous servers, similar request costs | Ignores load differences |
| **Weighted round robin** | Proportional to weights | Mixed instance sizes, canaries (5% weight) | Static |
| **Least connections** | Fewest active connections | Varying request durations, long-lived conns | Needs state; new servers get flooded ("thundering herd" onto empty node) |
| **Weighted least connections** | Normalized by weight | Mixed capacity | |
| **Least response time / latency (EWMA)** | Fastest recent responses (+ fewest active) | Heterogeneous, latency-sensitive | Feedback loops; a fast-failing server looks "fast" (must combine with error-awareness) |
| **Power of two choices (P2C)** | Pick 2 random backends, choose the less loaded | Large fleets, distributed LBs with stale info; Envoy/Linkerd/Finagle default-ish | Near-optimal with little coordination |
| **Random** | Random pick | Simple, stateless | Variance at small scale |
| **IP hash / source hash** | hash(client IP) % N | Crude affinity | Rebalances everything when N changes; NAT'd clients all land on one server |
| **Consistent hashing** (ring, **Maglev**, **rendezvous/HRW**) | Hash key onto ring; minimal remapping when servers change | Cache affinity (route same key to same cache node), sticky without cookies | Hot keys; use bounded-load consistent hashing (Google/Vimeo) |
| **Ring hash by header/cookie** | Hash user ID header | Per-user affinity for local caches | |

Slow start / warm-up: gradually ramp traffic to new instances (JIT warm-up, cold caches, connection pool growth). Envoy `slow_start_config`, ALB slow start mode.

**Outlier detection** (Envoy): eject backends with consecutive 5xx or high latency for a period. This is passive health checking.

---

## 5. Health Checks {#health}

- **Active**: the LB probes `GET /healthz` every N seconds; unhealthy after X failures, healthy after Y successes.
- **Passive**: observe real traffic errors/timeouts (outlier detection).
- Liveness vs readiness (Kubernetes vocabulary, applies broadly):
  - **Liveness**: "is the process alive / not deadlocked?" Failure → restart. Keep it **cheap and dependency-free**.
  - **Readiness**: "can I serve traffic now?" (warmed up, DB pool ready, not draining). Failure → remove from LB, don't restart.
  - **Startup** probes for slow starters.
- **Don't make health checks depend on shared dependencies** (e.g., the DB). If the DB blips, *every* instance fails its health check simultaneously, the LB removes them all, and that turns a partial outage into a total one. Some LBs "fail open" when all targets are unhealthy (ALB, Envoy panic threshold) for exactly this reason.
- Health endpoints should be fast, unauthenticated (but not public, or not revealing internals), and excluded from access-log noise.

---

## 6. Sticky Sessions {#sticky}

Route a client to the same backend via a cookie (`AWSALB`, app cookie) or a hash.
- Needed for **stateful servers** (in-memory sessions, WebSocket servers with local state, legacy apps).
- Problems: uneven load, lost sessions when the instance dies, harder scaling and deploys.
- Better: **stateless services** + shared session store (Redis) or self-contained tokens (JWT), so any instance can serve any request.
- WebSockets/SSE are inherently connection-sticky (the connection itself), so use a pub/sub backplane to fan out messages across instances.

---

## 7. TLS Handling {#tls}

| Mode | Where TLS ends | Pros | Cons |
|---|---|---|---|
| **Termination at LB** | LB decrypts; plain HTTP to backends | L7 features, offloads crypto, central cert management | Plaintext inside network (acceptable only in trusted networks; compliance may forbid) |
| **Re-encryption (TLS bridging)** | LB terminates, opens new TLS to backend | L7 features + encryption in transit | More CPU, cert management on backends |
| **Passthrough** | Backend terminates (LB routes by SNI at L4) | End-to-end encryption, backend holds keys | No L7 features at LB |
| **mTLS** | Both sides present certs | Strong service identity (zero trust) | Cert issuance/rotation (meshes automate via SPIFFE/SPIRE) |

Certs: ACME/Let's Encrypt automation (cert-manager in K8s), cloud certificate managers (ACM), short-lived certs, OCSP stapling, TLS 1.2+ only (prefer 1.3), HSTS.

---

## 8. Draining and Zero-Downtime Deploys {#draining}

1. Mark the instance as **draining**: the LB stops sending new requests/connections.
2. In-flight requests complete (deregistration delay, ALB default 300 s; tune it to your longest request).
3. The app handles `SIGTERM`: stop accepting, finish in-flight work, close keep-alive connections gracefully (`Connection: close`, h2 GOAWAY), flush logs/metrics, exit.
4. Kubernetes nuance: endpoint removal propagates asynchronously to kube-proxy, ingress controllers, and service meshes. A **preStop sleep (~5–15 s)** prevents traffic from arriving after the app starts shutting down.
5. Long-lived connections (WebSockets) need app-level reconnect logic with jittered backoff, or else everyone reconnects at once (a thundering herd) to remaining instances.

Deployment strategies via LBs: rolling, **blue-green** (switch target group), **canary** (weighted routing 1% → 10% → 100% with automated metric analysis, e.g., Argo Rollouts, Flagger), **shadow/mirroring** (duplicate traffic to the new version and discard responses).

---

## 9. Global Load Balancing {#global}

- **GeoDNS / latency-based DNS** (see the DNS note): route users to the nearest healthy region. Failover is limited by TTL caching.
- **Anycast**: one IP announced from many PoPs. BGP routes users to the nearest one, and failover happens at the network level in seconds. Used by Cloudflare, Google Cloud's global HTTP LB, AWS Global Accelerator, and CloudFront.
- **Global L7 LBs**: GCP global external HTTP(S) LB (single anycast IP, routes to the closest healthy backend region), Cloudflare LB, Azure Front Door, AWS Global Accelerator (anycast L4 with regional endpoints) + CloudFront.
- Multi-region concerns: data locality and replication lag (see the DB notes), session and state handling, consistent failover drills.

---

## 10. Client-Side Load Balancing {#clientside}

The client gets the list of instances (from service discovery: Consul, Eureka, K8s endpoints, DNS SRV, xDS) and balances itself:
- No extra hop and no central bottleneck. Per-request balancing for HTTP/2 and gRPC.
- Examples: gRPC built-in LB policies (`round_robin`, `pick_first`, xDS-based), Netflix Ribbon (legacy) / Spring Cloud LoadBalancer, Finagle, and service-mesh sidecars (which are "client-side LB moved into a proxy").
- Cost: logic in every client library and language, so meshes or proxyless gRPC with xDS centralize the configuration.

---

## 11. HA of the LB Itself {#ha}

- Cloud LBs are managed and distributed (multi-AZ by design). You enable multiple AZs and subnets.
- Self-managed: an active-passive pair with a **floating VIP** (keepalived/VRRP), or active-active with ECMP routing + anycast/BGP (multiple LBs announce the same IP; routers spread flows). Consistent hashing (Maglev) ensures flows stick to the same backend even when the LB instance changes.
- Avoid single points of failure at every layer: DNS (multiple providers), LB, app, DB.

---

## 12. API Gateways {#gateway}

An **API gateway** is an L7 reverse proxy specialized for APIs, a single entry point that handles cross-cutting concerns:
- Routing to services (path/host/header), protocol translation (REST ↔ gRPC, gRPC-Web).
- **Authentication** (JWT validation, OAuth2 introspection, API keys, mTLS) and coarse authorization.
- **Rate limiting** and quotas per consumer/plan (with Redis for distributed counters).
- Request/response transformation, validation against OpenAPI, payload size limits.
- Caching, compression, CORS.
- Observability: access logs, metrics, tracing headers.
- Developer portal, API keys lifecycle, monetization (Apigee, Kong Enterprise, Azure APIM).
- Canary routing, circuit breaking, retries.

Products: **Kong** (Nginx/OpenResty-based, plugins), **AWS API Gateway** (REST/HTTP/WebSocket APIs, Lambda integration), Apigee (Google), Azure API Management, **Envoy Gateway**, Tyk, KrakenD, Traefik, Spring Cloud Gateway, Zuul (legacy), APISIX, Gloo, and the Kubernetes **Gateway API** (successor to Ingress).

**Backend for Frontend (BFF)**: a separate gateway/aggregation layer per client type (web, iOS, Android, partner), tailored responses, owned by the frontend team. GraphQL often plays this role.

Anti-patterns: putting business logic in the gateway (becomes a distributed monolith bottleneck owned by no one), a single gateway team as a bottleneck for every change, chatty multi-hop gateways adding latency.

---

## 13. Service Mesh {#mesh}

A **service mesh** handles service-to-service networking via a **data plane** of proxies (usually sidecars next to each service instance) configured by a **control plane**.
- Features: **mTLS by default** (service identity via SPIFFE IDs), L7 load balancing for gRPC/HTTP2, retries, timeouts, circuit breaking/outlier detection, traffic splitting (canaries), fault injection, rich telemetry (golden metrics per service pair, tracing spans), authorization policies (service A may call service B's `GET /orders`).
- Implementations: **Istio** (Envoy sidecars, or **ambient mode**: per-node ztunnel for L4 + optional waypoint proxies for L7, no sidecars), **Linkerd** (Rust micro-proxy, simple), Consul Connect, Cilium Service Mesh (eBPF, sidecar-less), Kuma, AWS App Mesh (deprecated in favor of ECS Service Connect/VPC Lattice).
- Costs: extra hops and latency (~ sub-ms to a few ms per hop), memory per sidecar, operational complexity (control plane upgrades, debugging through proxies), and a learning curve.
- When it's worth it: many services, polyglot stacks, strict zero-trust/compliance requirements, a platform team to own it. For a handful of services, libraries + an API gateway are usually enough.

**Envoy** in one paragraph: a high-performance C++ L4/L7 proxy built at Lyft. It's dynamically configured via **xDS APIs** (listeners, routes, clusters, endpoints, secrets), has first-class HTTP/2/gRPC support, advanced LB, outlier detection, retries with budgets, rate limiting, WASM/Lua filters, and excellent observability. It's the data plane of Istio, Envoy Gateway, AWS App Mesh, Consul, Contour, and Gloo.

---

## 14. Product Landscape {#products}

| Category | Options |
|---|---|
| Cloud L4 | AWS NLB, GCP Network LB, Azure Load Balancer |
| Cloud L7 | AWS ALB, GCP Application LB, Azure Application Gateway / Front Door |
| Software proxies | Nginx, HAProxy, Envoy, Traefik, Caddy, Pingora (Cloudflare, Rust) |
| Kernel/eBPF L4 | LVS/IPVS, Katran, Cilium |
| Kubernetes ingress | ingress-nginx (retiring in favor of Gateway API implementations), Traefik, Contour, Envoy Gateway, Istio Gateway, cloud controllers |
| API gateways | Kong, AWS API Gateway, Apigee, Azure APIM, Tyk, KrakenD, APISIX |
| Meshes | Istio, Linkerd, Cilium, Consul |
| CDN/edge | Cloudflare, Fastly, Akamai, CloudFront |

---

## 15. Interview Questions {#qa}

1. L4 vs L7 load balancing: what can each see and do?
2. Compare round robin, least connections, and power-of-two-choices.
3. When would you use consistent hashing in a load balancer?
4. Why should health checks not depend on the database?
5. Why are sticky sessions problematic, and what's the alternative?
6. TLS termination vs passthrough vs re-encryption?
7. How do you deploy with zero downtime behind a load balancer (including Kubernetes nuances)?
8. Why do long-lived gRPC connections cause uneven load, and how do you fix it?
9. What does an API gateway do, and what shouldn't it do?
10. What's a service mesh? When is it worth the complexity?
11. How does anycast work for global load balancing?
