# DNS: How Names Become Addresses (and Why It Breaks Things)

> "It's always DNS." DNS is a globally distributed, hierarchical, heavily cached database. Its caching and TTL behavior decides how fast you can fail over, migrate, or roll back.

## Table of Contents
1. [What DNS Does](#what)
2. [The Hierarchy: Root, TLD, Authoritative](#hierarchy)
3. [Resolution Walkthrough: Recursive vs Iterative](#resolution)
4. [Record Types](#records)
5. [TTLs and Caching Layers](#ttl)
6. [DNS for Load Balancing, Failover, and Geo-Routing](#lb)
7. [DNS in Kubernetes and Service Discovery](#k8s)
8. [Security: Spoofing, DNSSEC, DoH/DoT, Takeovers](#security)
9. [Operational Gotchas](#gotchas)
10. [Debugging DNS](#debug)
11. [Interview Questions](#qa)

---

## 1. What DNS Does {#what}

It maps names to data: `api.example.com → 203.0.113.10`, plus mail servers, service locations, verification tokens, and more. Queries usually go over **UDP port 53** (TCP for large responses, zone transfers, or when truncated), and increasingly over DoH/DoT.

---

## 2. The Hierarchy {#hierarchy}

```
.                         root (13 named root server identities, hundreds of anycast instances)
├── com.                  TLD servers (Verisign for .com)
│   └── example.com.      authoritative nameservers (Route 53, Cloudflare, NS1...)
│       └── api.example.com.  → A 203.0.113.10
├── in.
└── io.
```
- A **zone** is the part of the namespace administered together (example.com and its subdomains, unless delegated).
- **Delegation**: the parent zone has **NS records** pointing to the child's nameservers, plus **glue records** (A records of the nameservers) when the nameservers are inside the child zone.
- Registrar (where you buy the domain) ≠ DNS host (where the zone lives). The registrar sets the NS records at the TLD.

---

## 3. Resolution Walkthrough {#resolution}

User types `api.example.com`:
1. **Application / OS stub resolver** checks the local cache, `/etc/hosts`, and nsswitch config.
2. Asks the configured **recursive resolver** (ISP, corporate, 8.8.8.8, 1.1.1.1, VPC resolver at `169.254.169.253` / `.2` address in AWS) with a recursive query.
3. The recursive resolver (if not cached) iterates:
   - Asks a **root server**: "who handles `.com`?" → referral to the `.com` TLD servers.
   - Asks a **.com TLD server**: "who handles `example.com`?" → referral to `ns1.example-dns.com`, …
   - Asks the **authoritative server**: "`api.example.com` A?" → `203.0.113.10`, TTL 60.
4. The resolver caches every answer (including referrals) per TTL and returns the result to the client.
5. The client connects.

Cold resolution can take 100+ ms. Cached, it's ~1 ms from a local resolver. **Negative answers (NXDOMAIN) are cached too**, governed by the SOA minimum / negative TTL. Querying a name before creating it can make it "not exist" for minutes afterward.

---

## 4. Record Types {#records}

| Type | Purpose | Example |
|---|---|---|
| **A** | IPv4 address | `api 60 IN A 203.0.113.10` |
| **AAAA** | IPv6 address | |
| **CNAME** | Alias to another name (resolver follows it) | `www CNAME example.com.` . Can't coexist with other records at the same name, and **can't be at the zone apex** (`example.com`) per RFC |
| ALIAS / ANAME / CNAME flattening | Provider-specific apex alias (Route 53 Alias, Cloudflare flattening) | `example.com ALIAS my-lb.elb.amazonaws.com` |
| **NS** | Delegation to nameservers | |
| **SOA** | Zone metadata: primary NS, admin, serial, refresh, retry, expire, negative TTL | |
| **MX** | Mail servers with priority | `MX 10 mail1.example.com.` |
| **TXT** | Arbitrary text: SPF, DKIM, DMARC, domain verification | `"v=spf1 include:_spf.google.com ~all"` |
| **SRV** | Service location: priority, weight, port, target | `_sip._tcp.example.com SRV 10 60 5060 sip1.example.com.` (used by Kubernetes, XMPP, some DB drivers: `mongodb+srv://`) |
| **CAA** | Which CAs may issue certificates | `CAA 0 issue "letsencrypt.org"` |
| **PTR** | Reverse lookup (IP → name) | Email deliverability |
| **HTTPS / SVCB** | Service binding: ALPN (h3), ports, ECH keys, alternative endpoints | Lets browsers use HTTP/3 immediately, apex aliasing |
| DS, DNSKEY, RRSIG, NSEC | DNSSEC | |

Email authentication (backend engineers sending email must set these): **SPF** (which servers may send for the domain), **DKIM** (signature public key), **DMARC** (policy and reporting). Since 2024, Gmail and Yahoo require them for bulk senders.

---

## 5. TTLs and Caching Layers {#ttl}

Caching happens at: browser (Chrome ~1 min), OS (systemd-resolved, nscd, mDNSResponder), **runtime/application** (the **JVM caches positive lookups forever** when a SecurityManager is installed, otherwise 30 s; set `networkaddress.cache.ttl`. Some HTTP clients and connection pools resolve once and never again), container node-local caches (NodeLocal DNSCache), the recursive resolver, and sometimes resolvers that **ignore TTLs** (clamping to minimums).

TTL strategy:
- Long TTLs (hours): fewer queries, faster resolution, cheaper. Slow to change.
- Short TTLs (30–60 s): fast failover and migrations, more query load and cost, slightly more latency on misses.
- **Before a migration, lower the TTL well in advance** (at least one old-TTL period earlier), migrate, verify, then raise it again.
- DNS-based failover is never instant: expect stragglers for minutes or hours due to misbehaving caches and long-lived connections that never re-resolve. **Existing TCP connections don't care about DNS changes.** Pooled connections keep talking to the old IP until they're recycled (one reason for pool `maxLifetime`).

---

## 6. DNS for Load Balancing and Failover {#lb}

- **Round-robin DNS**: multiple A records. Clients pick one, typically the first, which some resolvers rotate. There's no health awareness and uneven distribution due to caching. It's a crude approach.
- **Health-checked DNS failover** (Route 53 health checks, Cloudflare LB, NS1): unhealthy endpoints are removed from answers.
- **Weighted**: canary releases at the DNS level (10% to new stack).
- **Latency-based / geolocation / geoproximity routing**: answer with the closest region (GSLB, Global Server Load Balancing).
- **Anycast**: the same IP is announced from many locations via BGP, and the network routes to the nearest one. Used by root servers, public resolvers, CDNs, and Cloudflare. It's faster to shift than DNS, and it's network-level.
- EDNS Client Subnet (ECS): the resolver passes a truncated client subnet so geo-DNS answers for the user's location, not the resolver's (privacy trade-off).

---

## 7. DNS in Kubernetes and Service Discovery {#k8s}

- **CoreDNS** serves cluster DNS: `my-svc.my-namespace.svc.cluster.local` → ClusterIP. **Headless services** (`clusterIP: None`) return the pod IPs directly (StatefulSets: `pod-0.my-svc.ns.svc.cluster.local`).
- **`ndots:5` gotcha**: pods' `/etc/resolv.conf` has `search ns.svc.cluster.local svc.cluster.local cluster.local …` with `ndots:5`, so a lookup of `api.stripe.com` (2 dots < 5) first tries `api.stripe.com.ns.svc.cluster.local`, `…svc.cluster.local`, etc. That's 4–5 extra failed queries (×2 for A + AAAA) per external lookup, which overloads CoreDNS and adds latency. Fixes: use FQDNs with a trailing dot (`api.stripe.com.`), lower ndots via `dnsConfig`, and run NodeLocal DNSCache.
- Conntrack races with UDP DNS on Linux caused famous 5-second DNS delays in K8s (`single-request-reopen`, NodeLocal DNSCache fixed it).
- Other discovery: Consul DNS interface, AWS Cloud Map, Eureka (client-side registry, no DNS), service meshes.

---

## 8. Security {#security}

- **Cache poisoning / spoofing**: forged responses inserted into resolver caches (Kaminsky attack, 2008). Mitigations: source port randomization, 0x20 case randomization, and **DNSSEC**.
- **DNSSEC**: signs records (RRSIG) with zone keys (DNSKEY), with a chain of trust via DS records from the parent up to the root. It provides **authenticity and integrity, not confidentiality**. Adoption is partial, and misconfigured DNSSEC causes total outages (validation failures = SERVFAIL).
- **DoT (DNS over TLS, port 853)** and **DoH (DNS over HTTPS)**: encrypt the stub-to-resolver hop for privacy. Browsers use DoH, which can bypass corporate DNS policies.
- **Subdomain takeover**: `blog.example.com CNAME example.ghost.io` remains after you delete the Ghost/Heroku/S3/GitHub Pages resource, and an attacker claims that resource name, so they now serve content on your subdomain (cookie theft, phishing). **Delete DNS records when decommissioning resources.** Scan for dangling CNAMEs.
- **DNS rebinding**: an attacker's domain first resolves to their IP, then to `127.0.0.1` or internal IPs, bypassing same-origin assumptions and SSRF filters that validated the first resolution. Fix: validate the resolved IP at connection time, pin the resolved IP, and check Host headers on internal services.
- **DNS amplification DDoS**: open resolvers reflect large responses to spoofed victims. Don't run open resolvers.
- **Registrar account security**: domain hijacking via a compromised registrar account. Use MFA and registry lock for critical domains.
- DNS exfiltration (data encoded in queries) is used by malware. Monitor outbound DNS.

---

## 9. Operational Gotchas {#gotchas}

1. **Forgot the trailing dot** in zone files: `www CNAME example.com` becomes `example.com.example.com.`.
2. **CNAME at the apex** not allowed. Use provider ALIAS/flattening.
3. **Negative caching** after querying a name before creating it.
4. **Apps that resolve once** (at startup) and never again, which breaks failover and blue/green.
5. **TTL lowering done too late** before a migration.
6. **Split-horizon DNS** (different answers inside vs outside the VPC) causing "works on my laptop, not in prod".
7. **Resolver rate limits**: the AWS VPC resolver limit is 1024 packets/s per ENI. High-QPS services doing a lookup per request hit it. Cache DNS in the app or locally.
8. **IPv6 AAAA issues**: a service advertises AAAA but IPv6 is broken, so clients try IPv6 first and hang (Happy Eyeballs mitigates this in browsers, not always in backend clients).
9. **Long-lived connections** ignoring DNS changes (gRPC clients need DNS re-resolution or a proper LB policy).
10. **DNS outages are global outages**: the 2016 Dyn DDoS (Mirai botnet) took down Twitter, GitHub, and Netflix. Use multiple DNS providers for critical domains (secondary DNS with zone transfers or API sync), and keep long-enough TTLs on NS records.

---

## 10. Debugging {#debug}

```bash
dig api.example.com                    # answer, TTL, server used
dig +short api.example.com A
dig api.example.com AAAA
dig +trace api.example.com             # iterate from root yourself (bypass caches)
dig @8.8.8.8 api.example.com           # ask a specific resolver
dig @ns1.example-dns.com api.example.com +norecurse   # ask the authoritative directly
dig example.com NS / MX / TXT / CAA / SOA
dig -x 203.0.113.10                    # reverse
nslookup / host                        # simpler tools
resolvectl query api.example.com       # systemd-resolved view
getent hosts api.example.com           # what the OS/libc resolver returns (respects /etc/hosts, nsswitch)
kubectl exec -it pod -- cat /etc/resolv.conf
```
Online: DNSViz (DNSSEC), whatsmydns.net (global propagation), MXToolbox (mail records).

---

## 11. Interview Questions {#qa}

1. Walk through what happens when you type a URL. Start with DNS.
2. Recursive vs authoritative DNS servers?
3. A vs CNAME vs ALIAS records. Why can't a CNAME be at the apex?
4. How do TTLs affect failover? How would you plan a DNS cutover for a migration?
5. Why might your service keep talking to an old IP after a DNS change?
6. How does geo/latency-based DNS routing work? What's anycast?
7. What is a subdomain takeover and how do you prevent it?
8. Explain the `ndots:5` problem in Kubernetes.
9. What do SPF, DKIM, and DMARC do?
