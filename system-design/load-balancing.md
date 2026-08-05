# Load Balancing

A load balancer sits in front of a pool of servers and distributes incoming traffic across them. It's the piece of infrastructure that turns "one server that falls over under load" into "a fleet that can grow, shrink, and tolerate failures without clients noticing."

## TL;DR
- A single server has a ceiling on throughput and is a single point of failure. Load balancing spreads requests across multiple servers to raise both capacity and availability.
- **Layer 4** balances at the TCP/UDP level (fast, protocol-agnostic, no visibility into HTTP). **Layer 7** balances at the HTTP level (can route on path/header/cookie, terminate TLS, but costs more CPU per request).
- Algorithms trade off simplicity, fairness, and state: round robin, weighted round robin, least connections, least response time, IP hash, consistent hashing.
- Health checks (active probes + passive failure detection) are what let a load balancer actually route around dead nodes instead of blindly trusting its server list.
- Sticky sessions solve the "user state lives on one server" problem but reintroduce the coupling load balancing was meant to remove — prefer external session storage when you can.
- The load balancer itself must be redundant (active-passive or active-active pair, or DNS-based failover) — otherwise you've just moved the single point of failure up one layer.

## The problem it solves

A single server has hard limits: CPU, memory, network bandwidth, open file descriptors, disk I/O. Past a certain request rate it either queues requests (latency climbs), starts dropping connections, or crashes. And even if it never hits capacity, it's a single point of failure — if that box dies, the entire service is down.

Two servers behind a load balancer already fix both problems:
- **Capacity**: traffic splits across machines, so aggregate throughput scales roughly with server count (not perfectly — see Amdahl's law and shared-resource contention, but close enough to matter).
- **Availability**: if one server dies, the load balancer stops sending it traffic and the others absorb the load. Users see degraded capacity, not an outage.

This is the foundation almost every horizontally-scaled system is built on — web tiers, API gateways, database read replicas (via a proxy), message consumers, and so on.

## Layer 4 vs Layer 7 load balancing

The "layer" refers to the OSI model layer the load balancer operates at, and it fundamentally changes what the LB can see and do.

### Layer 4 (transport layer)

Operates on TCP/UDP — IP addresses and ports. The load balancer looks at the packet headers, picks a backend based on a simple algorithm (or a hash of the connection tuple), and then just forwards packets. It doesn't parse HTTP, doesn't know about cookies, paths, or headers — as far as it's concerned it's moving bytes.

- **Fast**: minimal per-packet processing, often handled in kernel space or dedicated hardware.
- **Protocol-agnostic**: works for HTTP, gRPC, raw TCP, database connections, SMTP — anything over TCP/UDP.
- **Can't route on content**: no path-based routing, no host-based routing, no cookie-based stickiness.
- Typically operates as either a pure packet forwarder (NAT-based, rewriting destination IP) or a **direct server return (DSR)** setup where the response bypasses the LB entirely on the way back — used when the LB shouldn't become a bandwidth bottleneck for large response payloads.
- Examples: AWS Network Load Balancer (NLB), IPVS (Linux Virtual Server), raw TCP mode in HAProxy/Nginx (`stream` block).

### Layer 7 (application layer)

Operates on HTTP/HTTPS (or other application protocols). The load balancer terminates the connection, actually parses the request — method, path, headers, cookies, body if needed — and makes routing decisions based on that content.

- **Content-aware routing**: `/api/*` goes to the API fleet, `/static/*` goes to a CDN origin, `Host: admin.example.com` goes to the admin cluster.
- **Can terminate TLS**: decrypts HTTPS once at the LB, then talks plain HTTP (or re-encrypted HTTPS) to backends — see the SSL termination section below.
- **More CPU-intensive per request**: has to fully parse HTTP, which costs more than L4's "look at 4-tuple and forward."
- Can do things L4 fundamentally cannot: A/B testing by header, canary releases by percentage, request retries, response caching, WAF (web application firewall) rules.
- Examples: Nginx, HAProxy (HTTP mode), AWS Application Load Balancer (ALB), Envoy, Traefik.

| | Layer 4 | Layer 7 |
|---|---|---|
| Sees | IP + port (TCP/UDP header) | Full HTTP request (path, headers, cookies, body) |
| Routing granularity | Per-connection | Per-request |
| Protocol support | Any TCP/UDP protocol | HTTP/HTTPS (and similar app protocols) |
| TLS termination | No (passthrough only) | Yes |
| Performance | Very high, low overhead | Lower throughput per core, more features |
| Content-based routing | No | Yes (path, host, header, cookie) |
| Typical use | Databases, raw TCP services, ultra-high-throughput edge | Web APIs, microservices, anything needing smart routing |

A common production pattern is **both, layered**: an L4 load balancer (e.g., NLB) at the very edge for raw throughput and DDoS absorption, forwarding to a tier of L7 load balancers/API gateways (e.g., ALB, Envoy) that do the smart routing to services.

## Load balancing algorithms

The algorithm decides, for each new connection or request, *which* backend gets it.

| Algorithm | How it picks | Good for | Weakness |
|---|---|---|---|
| Round robin | Cycles through servers in order: 1, 2, 3, 1, 2, 3... | Simple, stateless, roughly fair when servers are equal and requests are uniform | Ignores server load and request cost — a slow server gets the same share as a fast one |
| Weighted round robin | Round robin but servers with higher weight get proportionally more requests | Heterogeneous hardware (a bigger box should get more traffic) | Weights are static — doesn't adapt to real-time load |
| Least connections | Sends the new request to whichever backend currently has the fewest active connections | Long-lived or variable-duration connections (WebSockets, streaming) | Doesn't account for how *expensive* each connection is — 100 idle connections vs 100 heavy ones look the same |
| Weighted least connections | Least connections, weighted by server capacity | Heterogeneous fleets with variable request duration | More moving parts to tune correctly |
| Least response time | Picks the server with the lowest (active connections × average response time), or similar composite score | Latency-sensitive workloads where server speed varies | Needs continuous latency measurement; more overhead |
| IP hash | Hashes the client IP to deterministically map it to a backend | Simple session affinity without cookies | Uneven distribution if client IPs aren't well distributed (e.g., many users behind one corporate NAT); rehashing on backend add/remove reshuffles most mappings |
| Random | Picks a backend at random (optionally "power of two choices": pick 2 at random, choose the less loaded) | Very cheap, works surprisingly well at scale | Pure random can be uneven at low request volume |
| Consistent hashing | Maps both servers and keys onto a hash ring; a key routes to the next server clockwise on the ring | Minimizes remapping when servers are added/removed — critical for caches and sharded/stateful backends | More complex to implement correctly (virtual nodes needed for even distribution) |

Consistent hashing deserves a full note of its own — see **[consistent-hashing.md](./consistent-hashing.md)** for the ring mechanics, virtual nodes, and why it's the standard answer for cache/shard routing where minimizing remapping on scale events matters. The short version: naive hashing (`hash(key) % N`) remaps almost every key when `N` changes; consistent hashing only remaps `~1/N` of keys.

### Picking an algorithm in practice

- **Round robin / weighted round robin**: default choice for stateless, roughly-uniform-cost HTTP requests. This is what most people mean when they say "load balancing" with no qualifiers.
- **Least connections**: default for anything with long-lived connections or highly variable request duration (this is Nginx's and HAProxy's recommended default over round robin for most real HTTP workloads).
- **IP hash / consistent hashing**: when you need the *same client* to consistently land on the *same backend* without cookies — often for session affinity or to maximize cache hit rate on that backend.

## Health checks

A load balancer is only useful if it stops sending traffic to dead or degraded servers. That requires continuously knowing which servers are actually healthy.

### Active health checks

The load balancer proactively probes each backend on an interval — typically an HTTP GET to a dedicated `/health` or `/healthz` endpoint, or a TCP connect for L4.

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:8080 max_fails=3 fail_timeout=30s;
}
```

```
# HAProxy active health check
backend web_servers
    option httpchk GET /healthz
    http-check expect status 200
    server web1 10.0.0.1:8080 check inter 5s fall 3 rise 2
    server web2 10.0.0.2:8080 check inter 5s fall 3 rise 2
    server web3 10.0.0.3:8080 check inter 5s fall 3 rise 2
```

Key parameters, universal across implementations even if named differently:
- **Interval**: how often to check (e.g., every 5s).
- **Fall threshold**: consecutive failures before marking a server unhealthy (e.g., 3).
- **Rise threshold**: consecutive successes before marking a recovered server healthy again (e.g., 2).
- **Timeout**: how long to wait for a response before counting it as a failure.

A good `/health` endpoint checks that the process can actually serve traffic — not just "the process is running." A common mistake is a health check that always returns 200 regardless of whether the app can reach its database; that defeats the purpose. A better health check distinguishes **liveness** (is the process alive — restart if not) from **readiness** (can it serve traffic right now — pull from LB rotation if not, but don't necessarily restart), a distinction Kubernetes makes explicit with separate liveness and readiness probes.

### Passive health checks

Instead of (or in addition to) dedicated probes, the load balancer watches real traffic: if requests to a backend start timing out or returning 5xx at an elevated rate, it marks that backend unhealthy without needing a separate check cycle.

```
# HAProxy passive check via observe
backend web_servers
    server web1 10.0.0.1:8080 check observe layer7 error-limit 5 on-error mark-down
```

Passive checks react faster to real failures (no waiting for the next probe interval) but need live traffic to detect a problem — a backend that's broken but not yet receiving requests won't be caught until it does. Active and passive checks are complementary, not either/or; production setups typically run both.

### What happens after a server is marked unhealthy

The load balancer removes it from the rotation (new requests don't go to it) but usually doesn't kill in-flight connections immediately — that's a separate concern (draining, covered below). It keeps probing on the same interval so it can bring the server back once it passes the rise threshold.

## Session persistence / sticky sessions

Some applications store session state (shopping cart, login session, in-memory cache) on the specific server that first handled a user — if the next request lands on a *different* server, that state isn't there. Sticky sessions solve this by pinning a client to the same backend for the duration of their session.

### How it's implemented

- **Cookie-based** (L7 only): the load balancer inserts a cookie (e.g., `AWSALB`, or Nginx's `route` in a cookie) identifying which backend handled the first request; subsequent requests with that cookie go to the same backend.
  ```nginx
  upstream backend {
      ip_hash;   # simplest form of stickiness — no cookie needed
      server 10.0.0.1:8080;
      server 10.0.0.2:8080;
  }
  ```
  ```
  # Nginx cookie-based stickiness (requires nginx-plus or third-party module in OSS)
  upstream backend {
      server 10.0.0.1:8080;
      server 10.0.0.2:8080;
      sticky cookie srv_id expires=1h domain=.example.com path=/;
  }
  ```
- **IP hash** (works at L4 or L7): deterministically maps client IP to a backend — simple, no cookie, but breaks down when many clients share an IP (NAT, corporate proxy) or when a client's IP changes mid-session (mobile networks switching towers).

### The trade-off

Sticky sessions directly work against the reason load balancing is valuable in the first place:

- **Uneven load**: if some sessions are much heavier than others, stickiness can pin disproportionate load onto specific backends with no way to rebalance mid-session.
- **Breaks during scale-down**: if the backend a session is pinned to gets terminated (autoscaling scale-in, deploy, crash), that session's state is gone — the user gets logged out or loses their cart, depending on how gracefully the app handles it.
- **Complicates deploys**: rolling deploys need to drain sticky connections carefully rather than just cycling instances.
- **Reduces effective redundancy**: a "pool of N servers" behaves more like N independent single points of failure for the sessions pinned to each one.

**The better fix in most cases**: make the application actually stateless by moving session state out of the server's memory and into shared external storage — Redis, Memcached, a database — that every backend can read. Then any backend can serve any request, stickiness becomes unnecessary, and you get the full benefit of load balancing back. Sticky sessions are a pragmatic patch for legacy or hard-to-refactor apps, not a design goal.

## Hardware vs software load balancers

**Hardware load balancers** (F5 BIG-IP, Citrix ADC) are dedicated physical appliances with custom silicon tuned for packet processing. They handle very high throughput with predictable latency and often bundle extra features (WAF, DDoS protection, SSL acceleration chips). Downsides: expensive, harder to scale elastically (you buy/provision more boxes), typically not how cloud-native systems are built anymore.

**Software load balancers** run as regular processes on commodity hardware or VMs, and dominate modern infrastructure because they're cheap, scriptable, and fit cloud elasticity:

- **Nginx** — widely used as both a web server and L7 reverse proxy/load balancer. Simple config, huge ecosystem, good for HTTP.
- **HAProxy** — purpose-built load balancer, supports both L4 and L7, historically the go-to for high-performance TCP/HTTP load balancing, very mature health-check and stats tooling.
- **Envoy** — modern L7 proxy built for microservices/service mesh; first-class support for gRPC, HTTP/2, dynamic configuration via APIs (xDS), rich observability. The data plane most service meshes (Istio, etc.) are built on.
- **Traefik** — designed for dynamic environments (Docker, Kubernetes), auto-discovers backends from container/orchestrator metadata instead of static config files.
- **AWS ELB family**:
  - **CLB (Classic Load Balancer)** — legacy, mostly deprecated in favor of the two below.
  - **ALB (Application Load Balancer)** — L7, HTTP/HTTPS/gRPC, path/host-based routing, integrates with target groups, WAF, and Lambda.
  - **NLB (Network Load Balancer)** — L4, TCP/UDP/TLS, extremely high throughput and low latency, static IP support, preserves source IP.
- Cloud equivalents exist elsewhere: GCP Cloud Load Balancing, Azure Load Balancer / Application Gateway.

Managed cloud load balancers (ALB/NLB, GCP LB) are usually the right default for cloud-hosted systems — they're redundant by design, autoscale, and remove the "who patches/scales the LB itself" problem. Self-managed Nginx/HAProxy/Envoy earns its keep when you need behavior the managed product doesn't support, want to run on-prem, or are building a service mesh data plane.

## DNS-based load balancing vs a dedicated LB

**DNS round robin**: a domain name resolves to multiple IP addresses, and clients (or their resolvers) pick one — often just "the first one" or randomly, depending on the client/resolver. This is a form of load balancing that happens *before* any packet reaches your infrastructure.

- **Pros**: no extra infrastructure, works across regions/data centers, can distribute at a global scale that a single LB pair can't.
- **Cons**: no real health awareness (DNS doesn't know if a server is actually healthy — you rely on short TTLs and external health-check-driven DNS updates), no fine-grained algorithms, client-side and resolver caching can keep sending traffic to a dead IP well past the TTL, no visibility into per-request behavior.

**Dedicated load balancer**: a physical or virtual chokepoint that actually inspects and forwards each connection/request, with real-time health checks and algorithmic control.

In practice, systems combine both: DNS routes users to the *nearest healthy region* (global load balancing), and within that region a dedicated L7/L4 LB distributes traffic across the local server pool.

### Global server load balancing (GSLB)

GSLB is DNS-based load balancing made smarter for multi-region deployments — it answers "which data center/region should this client talk to?" rather than "which server in this data center?"

- **Geo-based routing**: route users to the nearest region by geography (reduces latency).
- **Latency-based routing**: route based on measured latency from resolver locations to each region (AWS Route 53 latency-based routing).
- **Health-aware failover**: if a region's health checks fail, GSLB stops returning that region's IPs and shifts traffic to healthy regions — this is how active-active or active-passive multi-region failover typically works at the DNS layer.
- **Weighted routing**: split traffic by percentage across regions, useful for canary rollouts or gradual migrations.

Real implementations: AWS Route 53 (latency-based, geo, weighted, failover routing policies), Cloudflare Load Balancing, Google Cloud's Global external Application Load Balancer (which, unlike pure DNS GSLB, uses anycast so it isn't limited by DNS TTL/caching problems).

⚠️ GSLB's biggest practical limitation is DNS TTL and caching: clients and intermediate resolvers cache the resolved IP for the TTL duration (and sometimes longer than they should), so a region failover isn't instant — it's bounded by how quickly caches expire across the internet. Low TTLs (30-60s) help but increase DNS query volume.

## 🟢 Beginner

- One server = one failure domain and one capacity ceiling. Load balancing exists to remove both limits by fanning traffic out across a pool.
- The load balancer becomes the single entry point clients talk to; it's the thing deciding "which backend handles this."
- The most basic mental model: round robin across a list of healthy servers. Everything else in this note is refinement on top of that idea.

## 🟡 Intermediate

- Know the L4 vs L7 distinction cold — it's one of the most common system design interview questions ("would you use an ALB or NLB here, and why").
- Understand why least connections tends to beat plain round robin for real HTTP workloads with variable request cost.
- Health checks are not optional in a real deployment — a load balancer with no health checking is just a traffic splitter, not a *reliability* mechanism, since it'll happily keep sending 1/N of traffic to a dead node.
- A worked example — Nginx as an L7 load balancer with health-aware routing and sticky sessions:

```nginx
http {
    upstream api_backend {
        least_conn;
        server 10.0.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
        server 10.0.1.11:8080 weight=2 max_fails=3 fail_timeout=30s;
        server 10.0.1.12:8080 weight=1 max_fails=3 fail_timeout=30s backup;
    }

    server {
        listen 443 ssl;
        server_name api.example.com;

        ssl_certificate     /etc/nginx/certs/api.example.com.crt;
        ssl_certificate_key /etc/nginx/certs/api.example.com.key;

        location / {
            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
        }
    }
}
```

Here `weight` implements weighted least-connections (bigger boxes get more traffic), `backup` marks a server as standby-only (only receives traffic if all primaries are down), and `X-Forwarded-*` headers preserve the original client's info since the backend now sees the LB's IP, not the client's.

- Equivalent HAProxy config, showing the stats page most teams also wire up:

```
frontend http_front
    bind *:443 ssl crt /etc/haproxy/certs/api.pem
    default_backend api_backend

backend api_backend
    balance leastconn
    option httpchk GET /healthz
    http-check expect status 200
    server web1 10.0.1.10:8080 check weight 3
    server web2 10.0.1.11:8080 check weight 2
    server web3 10.0.1.12:8080 check weight 1 backup

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

## 🔴 Advanced

### Multi-tier load balancing

Real production systems rarely have just one LB layer. A typical topology:

```
Internet
   │
   ▼
[DNS / GSLB]  ──── routes to nearest healthy region
   │
   ▼
[L4 LB]  (e.g., AWS NLB)  ──── absorbs raw connection volume, DDoS-resistant edge
   │
   ▼
[L7 LB / API Gateway]  (e.g., ALB, Envoy)  ──── path routing, TLS termination, auth
   │
   ▼
[Service mesh sidecar LB]  (e.g., Envoy as sidecar)  ──── per-service load balancing, retries, circuit breaking
   │
   ▼
Backend pods/instances
```

Each tier solves a different problem: DNS/GSLB picks a region, L4 handles raw throughput and provides a stable anycast/VIP entry point, L7 does content-aware routing and terminates TLS, and the mesh sidecar layer does fine-grained per-service traffic management (this is what Istio/Linkerd/Consul Connect are built on — every service instance gets a local Envoy proxy that load-balances outgoing calls to other services, so load balancing decisions happen right next to the caller instead of only at a central chokepoint).

### Service mesh sidecars as client-side load balancers

In a mesh, each service doesn't call other services directly — it calls its local sidecar proxy, which does service discovery + load balancing + retries + circuit breaking on its behalf. This is effectively **client-side load balancing**: instead of every request going through a shared, centralized LB (a potential bottleneck and extra network hop), each caller's sidecar picks the destination instance directly using a locally-cached view of healthy endpoints (kept fresh via a control plane like Istio's Pilot or Envoy's xDS protocol).

Trade-off: removes the centralized-LB bottleneck and hop, but pushes complexity to every node (each sidecar needs an up-to-date, consistent view of the healthy endpoint set) and makes debugging routing decisions more distributed.

### SSL/TLS termination vs passthrough

- **Termination at the LB**: the LB decrypts HTTPS, inspects/routes the plaintext HTTP request, then either forwards it in plaintext to backends (common inside a trusted VPC) or re-encrypts for the hop to the backend ("SSL bridging"). This is what enables L7 features — you can't route on `Host` or path if you can't read the request. Centralizes certificate management at the LB.
- **Passthrough**: the LB forwards the encrypted TCP stream untouched; the backend server terminates TLS itself. Required when the LB must not (or contractually cannot) see plaintext traffic, or when doing L4-only load balancing where the LB has no HTTP visibility anyway. Each backend needs its own certificate and does its own TLS handshake, which costs more CPU per backend and complicates cert rotation (no single place to update).

| | Termination at LB | Passthrough |
|---|---|---|
| LB can route on HTTP content | Yes | No |
| Backend CPU cost | Lower (no TLS handshake) | Higher |
| Cert management | Centralized at LB | Distributed across backends |
| End-to-end encryption | Only if LB re-encrypts to backend | Yes, inherently |
| Typical use | Public-facing APIs, most web traffic | Compliance-mandated e2e encryption, pure L4 setups |

### Failure modes

- **Thundering herd on failover**: when a backend (or an entire AZ/region) is marked unhealthy, all its in-flight and would-be traffic suddenly redirects to the remaining healthy nodes *at once*. If those nodes were already near capacity, the sudden spike can cascade — the survivors get overloaded and start failing too, marking themselves unhealthy, redirecting even more traffic onto whatever's left. Mitigations: keep meaningful headroom in capacity planning (don't run at 90%+ utilization steady-state), use gradual traffic shifting during failover rather than instant all-at-once cutover, and pair with circuit breakers/load shedding at the backend so an overloaded server fails fast and cheaply instead of falling over slowly and expensively.
- **Retry storms**: clients (or upstream LBs) retrying failed requests amplify load on already-struggling backends — the classic "retries made the outage worse" failure. Exponential backoff with jitter, and capping total retry budget, are the standard mitigations.
- **Connection draining**: when a backend is being removed (deploy, scale-down, marked unhealthy), the LB should stop sending *new* connections to it but let *existing* in-flight requests finish before terminating it (AWS calls this "deregistration delay," HAProxy has graceful shutdown support). Skipping this means every deploy drops in-flight requests.
- **Split-brain / stale endpoint lists**: in dynamic environments (autoscaling, container orchestration), if the LB's view of "which backends exist" lags reality, it can route to instances that no longer exist or miss newly launched ones. This is why service discovery integration (not static config) matters at scale.
- **LB itself as a single point of failure**: this is the meta-pitfall — see below.

## Common pitfalls

- ⚠️ **The load balancer becomes the new single point of failure.** Putting all traffic behind one Nginx/HAProxy box just relocates the problem it was meant to solve. Real deployments run LBs in redundant pairs (active-passive with a floating VIP via keepalived/VRRP, or active-active) or use a managed cloud LB (ALB/NLB) that's redundant by default across AZs.
- ⚠️ **Uneven load from bad hashing.** IP hash with a small/skewed client population (e.g., most traffic from behind a few corporate NATs) can pile disproportionate load onto one backend. Consistent hashing without enough virtual nodes has the same problem — see [consistent-hashing.md](./consistent-hashing.md) for the fix.
- ⚠️ **Sticky sessions silently breaking during scale-down or deploys.** A user's session gets orphaned when their pinned backend disappears — often surfaces as mysterious "random logouts" or lost cart contents that are hard to reproduce because they only happen during scaling events.
- ⚠️ **Health checks that don't check anything meaningful.** A `/health` endpoint that returns `200 OK` unconditionally (or just checks "is the HTTP server up") won't catch a backend that's alive but can't reach its database — it stays in rotation and serves errors until something else notices.
- ⚠️ **No connection draining on deploy.** Instantly killing instances during a rolling deploy drops in-flight requests; always drain first.
- ⚠️ **Forgetting `X-Forwarded-For`/`X-Real-IP`.** Once traffic goes through a proxy, backends see the LB's IP as the source unless the LB explicitly forwards the original client IP in a header — breaks IP-based logging, rate limiting, and geolocation if not handled.
- ⚠️ **Mismatched timeouts across tiers.** If the LB's timeout is shorter than the backend's expected processing time, the LB gives up and retries (or errors) while the backend is still working — wasting backend capacity on a request nobody's waiting for anymore. Timeouts should generally get *tighter* the further you are from the actual work, not looser.

## Real-world examples

- **Netflix**: uses AWS ELB at the edge combined with Eureka for service discovery and Ribbon/Zuul-style client-side load balancing internally (their microservices historically load-balanced calls to each other client-side, a precursor to today's service mesh pattern).
- **GitHub**: uses HAProxy extensively in front of its application tier, documented in several of their engineering blog posts on scaling MySQL and web traffic.
- **Google**: Maglev, Google's software network load balancer, handles massive L4 traffic at the edge of their network using consistent hashing for connection stability across a fleet of load balancers themselves (a good example of "the load balancer pool is itself load balanced").
- **Cloudflare**: operates GSLB via anycast — the same IP address is announced from many data centers, and BGP routing (not DNS) sends each client to the nearest one, sidestepping DNS TTL/caching issues entirely.

## Further reading

- [consistent-hashing.md](./consistent-hashing.md) — the ring-based hashing algorithm referenced above
- NGINX docs: [Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
- HAProxy documentation: [Configuration Manual](http://docs.haproxy.org/)
- AWS docs: [Elastic Load Balancing features](https://aws.amazon.com/elasticloadbalancing/features/) (ALB vs NLB vs CLB comparison)
- Envoy docs: [Load balancing](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers)
- Google SRE Book: [Chapter 20 — Load Balancing in the Datacenter](https://sre.google/sre-book/load-balancing-datacenter/)
- "Maglev: A Fast and Reliable Software Network Load Balancer" (Google, NSDI 2016)
