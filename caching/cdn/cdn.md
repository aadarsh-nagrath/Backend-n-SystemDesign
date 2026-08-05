# Content delivery networks (CDNs)

A **CDN** is a geographically distributed network of servers that caches and delivers web content — HTML, JS, CSS, images, video, even API responses — from locations physically close to end users instead of a single origin server. CDNs are why a site hosted in Virginia can still load fast in Tokyo, and they carry the majority of web traffic for platforms like Netflix, Amazon, Google, and Facebook.

## TL;DR
- CDNs cache content at **edge servers** located at **Points of Presence (PoPs)** near users, cutting latency and origin load.
- **Routing** (Anycast or DNS-based) sends each request to the nearest/least-loaded edge.
- **Cache-Control**, `ETag`, and TTLs govern what's cached and for how long; invalidation (purge, versioning) handles updates.
- Beyond caching, modern CDNs bundle security (DDoS mitigation, WAF, TLS), compression, image optimization, and edge compute (Cloudflare Workers, Lambda@Edge).
- Trade-offs: cache invalidation is genuinely hard, dynamic/personalized content resists caching, and pricing/vendor lock-in need active management.

## What is a CDN?

A CDN is a system of interconnected **edge servers**, placed at PoPs across the globe, that cache content from an **origin server** and serve it to users based on geographic proximity. It sits between clients and the origin, optimizing delivery of both static and dynamic content.

**Key components:**
- **Edge servers** — machines at PoPs that cache and serve content. A user in Tokyo hits a Tokyo edge server instead of a US-based origin.
- **Points of Presence (PoPs)** — physical locations hosting edge servers, often at Internet Exchange Points (IXPs). Cloudflare runs 300+ PoPs; Akamai runs 4,100+.
- **Origin servers** — where the real content lives (e.g., an EC2 instance running the actual app).
- **Caching** — storing copies of content at the edge for fast repeat delivery.
- **Routing** — Anycast or DNS-based mechanisms that direct a request to the nearest PoP.

**CDN vs. web hosting**: hosting stores and serves the original content but has no geographic distribution; a CDN doesn't originate content, it caches and accelerates delivery of what hosting already serves. They're complementary, not competing.

## A brief history

1. **Late 1990s–early 2000s (first generation)** — Akamai (1998) pioneers intelligent traffic management and IXP-based caching for static content (HTML, images). Limited PoPs, high cost.
2. **Mid-2000s–2010s (second generation)** — streaming media and mobile drive P2P networks, cloud computing, dynamic content acceleration. Limelight, Level 3, Cloudflare (2009) emerge.
3. **2010s–present (third generation)** — edge computing, security, and real-time apps take over. Serverless edge functions, AI-driven routing, DDoS mitigation. Amazon CloudFront, Google Cloud CDN, Fastly, Vercel.
4. **Emerging** — autonomous edge networks with AI optimization, 5G integration, decentralized web (Web3) support.

## How CDNs work

### 1. Content distribution
The origin hosts the original content. It's pushed to edge servers proactively or pulled on-demand; edge servers store copies of cacheable content.

### 2. Request routing
- **DNS-based routing** — resolves the domain to the IP of the nearest PoP based on the user's DNS resolver location. Simple, widely supported, but coarser (depends on DNS propagation and resolver location, not the actual client).
- **Anycast routing** — a single IP address is announced from every PoP; BGP routes each request to the topologically closest one. Faster, more DDoS-resilient, requires solid network infrastructure. Used by Cloudflare and Google Cloud CDN.
- **Geo-routing** — routes based on user geolocation (IP-derived), e.g., serving EU users from a Frankfurt PoP.
- **QUIC/HTTP-3** — UDP-based transport that reduces handshake latency and improves reliability on lossy (e.g., mobile) networks.

### 3. Content delivery
- **Cache hit** — content is already cached at the edge; served immediately.
- **Cache miss** — not cached; edge server fetches from origin, caches it, then delivers.
- **Cache-to-cache fill** — a nearby PoP that already has the content shares it with the requesting PoP, avoiding an origin trip entirely.

### 4. Optimizations applied at the edge
- **Compression** — Gzip or Brotli to shrink payloads.
- **Minification** — stripping unnecessary characters from JS/CSS.
- **Image optimization** — converting to modern formats (WebP, AVIF), resizing on the fly.
- **TLS termination** — handling the SSL/TLS handshake at the edge instead of the origin, cutting round-trip latency.
- **Connection reuse** — persistent connections back to origin to avoid repeated handshake overhead.

### 5. Security at the edge
- **DDoS mitigation** — absorbing attack traffic across many distributed PoPs instead of one origin.
- **TLS/SSL** — encryption with managed certificates.
- **WAF (Web Application Firewall)** — blocking malicious requests (SQLi, XSS) before they reach origin.

### Example request flow
1. A user in Sydney visits `www.example.com`.
2. Anycast routes the request to a Sydney PoP.
3. Edge server checks its cache: a CSS file is a **hit**, served instantly; an uncached video is a **miss** — fetched from the US origin, cached, then delivered.
4. The edge server compresses the response and terminates TLS.
5. The user gets content with minimal latency, without a round-trip to the US.

## CDN architecture

| Component | Role |
|---|---|
| **PoPs** | Physical locations (often IXPs) housing edge servers, placed to minimize distance to users |
| **Edge servers** | Cache content, terminate TLS, run edge compute; typically SSD-backed, running reverse proxy software (Nginx, Varnish) plus CDN logic |
| **Origin servers** | Host the real content, serve edge servers on cache misses |
| **Backbones** | High-speed private networks connecting PoPs and origins, reducing reliance on public internet for cache fills |
| **Load balancers** | Distribute traffic across edge servers within a PoP |

## 🟢 Beginner: caching fundamentals

### What gets cached

- **Static content** — images, CSS, JS, fonts: unchanging, ideal for long cache lifetimes (e.g., a logo cached for a year).
- **Dynamic content** — API responses, personalized/user-specific data: changes often, harder to cache safely, usually needs short TTLs or edge compute for personalization.

### Cache operations

- **Cache hit / miss** — as above.
- **Cache eviction** — removing expired or least-used content once TTL expires or cache size limits are hit.
- **Cache invalidation** — proactively purging outdated content, e.g., right after a deploy.

### Cache control via HTTP headers

```http
# Origin response
HTTP/1.1 200 OK
Cache-Control: max-age=86400
Content-Type: image/jpeg

# Edge server caches this for 24 hours, serves it directly on repeat requests
```
- **`Cache-Control`** — the primary directive (`max-age=3600` = 1-hour freshness).
- **`Expires`** — an older, absolute-date alternative to `max-age`.
- **`ETag`** — a content fingerprint used to validate freshness without re-sending the full body.

CDN configuration can also override origin headers — e.g., Cloudflare's "Cache Everything" page rule ignores what the origin says and caches regardless.

## 🟡 Intermediate: routing, invalidation, and advanced caching

### Cache modes
- **Default** — cache static content per headers.
- **Force cache** — ignore `no-cache` directives, cache everything anyway (aggressive, use carefully).
- **Bypass cache** — always hit origin directly (for private/sensitive data that must never be cached).

### Advanced caching techniques
- **Cache key customization** — control what's included in the cache key; e.g., strip `?utm_source=...` from the key so tracking params don't fragment the cache into near-duplicate entries.
- **Tiered caching** — a mid-tier cache sits between edge PoPs and the origin, absorbing cache misses from many edges before they all hit origin directly. Reduces origin load significantly at scale.
- **Smart/predictive caching** — AI-driven pre-caching based on observed traffic patterns (e.g., Fastly's predictive caching).

### Cache invalidation strategies

This is the hard part of CDN operations — "there are only two hard things in computer science: cache invalidation and naming things."

| Strategy | How it works | Trade-off |
|---|---|---|
| **TTL expiry** | Content simply expires after `max-age` | Simple, but stale content can be served for up to the full TTL window |
| **Instant purge API** | Explicitly tell the CDN to evict a specific URL/path now | Fast, but requires the deploy pipeline to call it |
| **Versioned/fingerprinted URLs** | Bake a content hash into the filename (`app.a3f9c2.js`) | Sidesteps invalidation entirely — a new version is a new URL, old URL can cache forever |
| **Cache tags/surrogate keys** | Tag cached objects (e.g., "product-123") and purge by tag | Lets you invalidate many related URLs in one call without knowing every exact path |

Fingerprinted URLs plus long TTLs (`Cache-Control: max-age=31536000, immutable`) is the standard pattern for static assets — you get maximum cache lifetime with zero invalidation risk, because a content change always produces a new URL.

### Routing mechanisms in depth

| Mechanism | Pros | Cons |
|---|---|---|
| DNS-based | Simple, universally supported | DNS propagation delay, less precise for mobile clients |
| Anycast (BGP) | Fast, resilient, great for DDoS absorption | Needs serious network infrastructure to run well |
| Geo-routing | Fine-grained control by region | Depends on IP geolocation accuracy |
| QUIC/HTTP-3 | Faster handshakes, better on lossy networks | Requires UDP support end-to-end |

## 🔴 Advanced: edge compute, security, and multi-CDN

### Edge computing
Running actual code at the edge, not just caching bytes:
- **Cloudflare Workers**, **AWS Lambda@Edge** — serverless functions executed per-request at the PoP.
- Use cases: personalizing content by geolocation, A/B testing without origin round-trips, auth checks before content is served.

### Advanced security
- **Bot management** — distinguishing legitimate crawlers/users from scraper/attack bots (e.g., Akamai Bot Manager).
- **Signed URLs / signed cookies** — restrict access to premium content (video segments, downloads) to authorized requests only.
- **Rate limiting** — block excessive requests from a single client/IP at the edge, before they reach origin.

### Multi-CDN strategies
Running two or more CDN providers simultaneously (e.g., Cloudflare + Akamai) for redundancy and negotiating leverage. Adds operational complexity (config has to be kept in sync across providers, DNS/routing logic to split traffic) but avoids single-vendor outages taking the whole site down.

### Performance optimizations at scale
- **Compression** — Brotli typically beats Gzip for text assets by 15-20%.
- **Minification** — automatic whitespace/dead-code stripping (Cloudflare Auto Minify).
- **Image optimization** — automatic WebP/AVIF conversion and responsive resizing (Fastly Image Optimizer, Cloudflare Polish).
- **Connection optimization** — persistent TCP connections to origin to avoid repeated handshakes (CloudFront).

## Benefits

| Category | Benefit | Example |
|---|---|---|
| Performance | Lower latency, faster loads, edge-terminated TLS | A 2s page load cut to ~500ms |
| Cost | Lower bandwidth/infra spend at origin | ~80% bandwidth savings with CloudFront on a 1TB/month site |
| Reliability | High availability, load balancing, automatic failover | Google Cloud CDN targets 99.99% uptime |
| Security | DDoS absorption, WAF, managed TLS, signed URLs | Akamai has mitigated multi-Tbps DDoS attacks |
| Scalability | Handles traffic spikes and viral events | Hulu streams tens of GBps via CloudFront |
| UX | Lower bounce rates, consistent global performance | Reuters delivers breaking news instantly worldwide |

## Challenges

| Challenge | Issue | Mitigation |
|---|---|---|
| Cache invalidation | Purging stale content is complex, may lag | Versioned URLs, instant purge APIs, cache tags |
| Dynamic content | Frequent changes resist caching | Short TTLs, edge compute for personalization |
| Cost complexity | Pricing varies by traffic/region/feature | Monitor usage, optimize cache hit ratio |
| Security misconfig | A misconfigured CDN can leak sensitive data | Strict access controls, config audits |
| Vendor lock-in | Proprietary features tie you to one provider | Standard protocols, multi-CDN strategy |

## Use cases

- **Web performance** — accelerating e-commerce, blogs, SaaS (Shopify uses Fastly for checkout).
- **Media streaming** — low-latency video/audio (Netflix runs its own CDN, Open Connect).
- **Gaming** — patch distribution, multiplayer server proximity (King uses CloudFront for billions of daily sessions).
- **IoT** — firmware delivery to distributed devices.
- **API acceleration** — caching API responses close to mobile clients.
- **Real-time apps** — live streaming, chat, WebSocket acceleration (Zoom).

## Popular CDN providers

| Provider | Key features | Strengths | Weaknesses | Typical use |
|---|---|---|---|---|
| **Cloudflare** | 300+ PoPs, free SSL, DDoS protection | Easy setup, security-focused | Limited advanced analytics | SMEs, security-critical apps |
| **Amazon CloudFront** | AWS integration, 450+ PoPs, Lambda@Edge | Scalable, feature-rich | Complex pricing | E-commerce, media streaming |
| **Akamai** | 4,100+ PoPs, bot management, Kona WAF | Enterprise-grade, robust | High cost | Large enterprises, gaming |
| **Google Cloud CDN** | Anycast, QUIC, global load balancing | High performance, Google backbone | Google Cloud dependency | Cloud-native apps |
| **Fastly** | Edge computing, instant purge, VCL | Developer-friendly, flexible | Smaller PoP network | Real-time apps, APIs |
| **Vercel Edge** | Next.js integration, serverless | Simple for frontend devs | Limited enterprise features | Static sites, JAMstack |

### Setting up Cloudflare (example)
1. Sign up and add a domain (e.g., `example.com`).
2. Update DNS to point to Cloudflare's nameservers.
3. Enable caching with a "Cache Everything" rule.
4. Configure SSL/TLS mode to "Full."
5. Monitor cache hit ratio via the dashboard.

### CloudFront distribution (Infrastructure as Code)
```yaml
# CloudFormation Template for a CloudFront Distribution
Resources:
  CloudFrontDistribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - DomainName: example.com
            Id: S3-origin
            S3OriginConfig:
              OriginAccessIdentity: ""
        Enabled: true
        DefaultCacheBehavior:
          TargetOriginId: S3-origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
        PriceClass: PriceClass_100
        HttpVersion: http2
```
Flow: create an S3 bucket for static content → deploy this distribution → point DNS at the CloudFront domain → configure caching policy and SSL → verify with a tool like WebPageTest.

## Metrics to track

| Metric | What it measures | Target |
|---|---|---|
| Cache hit ratio | % of requests served from cache, not origin | >90% for static content |
| Latency | Time to deliver content to the user | <100ms for edge delivery |
| Bandwidth savings | Reduction in origin traffic due to caching | ~80% with a well-tuned cache |
| Error rate | % of failed (5xx) requests | <0.1% |
| Time to First Byte (TTFB) | Time from request to first byte received | <200ms |

## Best practices

1. Maximize cache hit ratio: long TTLs for static content, short TTLs (or bypass) for dynamic/personalized content.
2. Use Anycast routing where available for speed and DDoS resilience.
3. Enforce modern TLS (1.3) at the edge.
4. Monitor cache hit ratio, latency, and error rate continuously.
5. Purge strategically — prefer versioned URLs over relying on manual purges.
6. Test from multiple geographic regions, not just your own location.
7. Combine with edge compute for personalization instead of disabling caching entirely.
8. Consider multi-CDN for redundancy on business-critical traffic.

## Further reading
- [Akamai: how a CDN works](https://www.akamai.com/glossary/what-is-a-cdn)
- [Cloudflare: what is a CDN](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [AWS CloudFront documentation](https://docs.aws.amazon.com/cloudfront/)
