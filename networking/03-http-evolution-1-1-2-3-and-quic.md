# HTTP Deep Dive: HTTP/1.1, HTTP/2, HTTP/3 and QUIC

> TLS details live in [`api-security/ssl-tls.md`](../api-security/ssl-tls.md) and [`api-security/https.md`](../api-security/https.md). Headers reference: [`api-security/http_header.pdf`](../api-security/http_header.pdf). This note covers HTTP semantics and how each protocol version moves bytes.

## Table of Contents
1. [HTTP Semantics (Version-Independent)](#semantics)
2. [HTTP/1.0 → 1.1: Persistent Connections, Pipelining, Chunking](#h1)
3. [HTTP/1.1 Problems](#h1problems)
4. [HTTP/2: Binary Framing, Multiplexing, HPACK](#h2)
5. [HTTP/2 Problems: TCP Head-of-Line Blocking](#h2problems)
6. [QUIC and HTTP/3](#h3)
7. [Version Negotiation: ALPN, Alt-Svc, HTTPS Records](#negotiation)
8. [Important Headers for Backend Engineers](#headers)
9. [Cookies](#cookies)
10. [Content Negotiation, Compression, Range Requests](#content)
11. [Full Request Lifecycle and Latency Budget](#lifecycle)
12. [Backend Implications (proxies, gRPC, keep-alive)](#backend)
13. [Interview Questions](#qa)

---

## 1. HTTP Semantics {#semantics}

RFC 9110 (2022) defines semantics shared by all versions: **methods, status codes, headers/fields, content, caching (RFC 9111)**. RFC 9112 (HTTP/1.1), 9113 (HTTP/2), and 9114 (HTTP/3) define the wire formats.

Request = method + target + headers + optional body. Response = status + headers + optional body. HTTP is **stateless**: each request is independent, and state lives in cookies, tokens, or server-side sessions.

---

## 2. HTTP/1.0 → 1.1 {#h1}

```
GET /orders/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJ...
Connection: keep-alive

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 87
Cache-Control: private, max-age=0

{"id":42,...}
```
- Text-based, line-oriented.
- HTTP/1.0: a new TCP connection per request by default.
- **HTTP/1.1 (1997)**:
  - **Persistent connections** by default (keep-alive), which reuse TCP+TLS across requests.
  - `Host` header required, enabling **virtual hosting** (many sites per IP).
  - **Chunked transfer encoding**: stream a body of unknown length (`Transfer-Encoding: chunked`).
  - **Pipelining**: send multiple requests without waiting, but responses must come back in order. It was broken in practice (HOL blocking, buggy proxies) and is disabled in browsers.
  - Caching (`Cache-Control`, `ETag`), range requests, content negotiation, `100 Continue`.

---

## 3. HTTP/1.1 Problems {#h1problems}

- **One outstanding request per connection** → **application-level head-of-line blocking**: a slow response blocks everything behind it on that connection.
- Browsers open **~6 connections per origin** to parallelize, each paying TCP + TLS handshakes and slow start.
- Workarounds that became "best practices": domain sharding, spriting, concatenating JS/CSS, inlining. All of them are hacks.
- **Verbose, repetitive headers** (cookies, user-agent) sent uncompressed on every request.
- Parsing ambiguities between proxies and servers lead to **HTTP request smuggling** (`Content-Length` vs `Transfer-Encoding` disagreements).

---

## 4. HTTP/2 (2015, RFC 9113) {#h2}

Based on Google's SPDY. Same semantics, new framing:
- **Binary framing layer**: messages split into **frames** (HEADERS, DATA, SETTINGS, WINDOW_UPDATE, PING, GOAWAY, RST_STREAM, PRIORITY, PUSH_PROMISE) tagged with a **stream ID**.
- **Multiplexing**: many concurrent **streams** (request/response pairs) interleaved on **one TCP connection**. No more HTTP-level HOL blocking and no need for 6 connections.
- **HPACK header compression**: static table + dynamic table of previously sent headers + Huffman coding. Repeated headers cost a few bytes. (A CRIME-style compression attack drove HPACK's design: no cross-request DEFLATE.)
- **Stream prioritization**: dependency trees in the original spec (complex, poorly implemented), replaced by the simpler **Extensible Priorities** (RFC 9218: `Priority: u=3, i`).
- **Flow control** per stream and per connection (WINDOW_UPDATE).
- **Server push**: proactively send resources. It was rarely beneficial, **Chrome removed it** (2022), and it's effectively dead. Use `103 Early Hints` + preload instead.
- In practice h2 requires TLS (browsers only do h2 over TLS; **h2c** cleartext is used internally, e.g., gRPC without TLS).
- `GOAWAY` frame: graceful shutdown ("finish what you have, no new streams"). Important for zero-downtime deploys.

**gRPC is built on HTTP/2**: each RPC is a stream, streaming RPCs use long-lived streams, and metadata travels as headers/trailers. See [`api/grpc.md`](../api/grpc.md).

---

## 5. HTTP/2 Problems {#h2problems}

- **TCP-level head-of-line blocking**: all streams share one TCP byte stream. One lost packet stalls **all** streams until it's retransmitted. On lossy mobile networks, h2 can perform worse than h1 with 6 connections.
- One connection = one congestion window. Middleboxes and LBs see one flow.
- Security issues: **HTTP/2 Rapid Reset (CVE-2023-44487)**: clients open and immediately cancel streams (RST_STREAM) at enormous rates, producing record DDoS (hundreds of millions of rps). Servers now limit stream resets. There were also HPACK bombs and CONTINUATION flood (2024).
- **Load balancing gRPC/h2** with L4 LBs: since one long-lived connection carries all requests, an L4 balancer pins all traffic from a client to one backend, which causes imbalance. Use **L7 (h2-aware) load balancing** (Envoy, Linkerd, gRPC client-side LB) or periodic connection recycling (`MAX_CONNECTION_AGE`).

---

## 6. QUIC and HTTP/3 {#h3}

**QUIC** (RFC 9000, 2021; originally Google): a transport protocol over **UDP** implementing reliability, congestion control, and streams in user space, with **TLS 1.3 built in**.
- **Independent streams at the transport level**: a lost packet only blocks the stream it belongs to. This eliminates TCP HOL blocking.
- **Faster handshakes**: transport + crypto handshake combined: **1 RTT** for new connections (vs TCP 1 + TLS 1), **0-RTT** for resumption (with replay-attack caveats: only use it for idempotent requests).
- **Connection migration**: connections are identified by **connection IDs**, not the 4-tuple, so switching from Wi-Fi to 4G keeps the connection alive.
- Encrypted transport headers reduce middlebox ossification and leak less metadata.
- User-space implementations iterate faster (but cost more CPU than kernel TCP, though this is improving with GSO/GRO, kernel offloads, and sendmmsg).
- Pluggable congestion control (BBR common).

**HTTP/3** (RFC 9114, 2022) = HTTP semantics over QUIC. **QPACK** replaces HPACK (HPACK assumed in-order delivery, so QPACK avoids HOL blocking on the header tables).

Adoption: enabled by Google, Meta, Cloudflare, Akamai, Fastly, and all major browsers (~30%+ of web traffic). Server support: Nginx (1.25+ experimental/stable QUIC), Caddy (built in), HAProxy, Envoy, LiteSpeed, and cloud LBs/CDNs. Most backends terminate h3 at the edge/CDN and speak h1.1/h2 to origins.

Challenges: UDP blocked or rate-limited on some networks (clients fall back to TCP), higher CPU, and UDP load balancing needs connection-ID-aware routing (QUIC-LB).

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC (UDP) |
| Format | Text | Binary frames | Binary frames |
| Multiplexing | No (1 req at a time per conn) | Yes | Yes |
| HOL blocking | HTTP + TCP | TCP | None at transport (per stream only) |
| Header compression | None | HPACK | QPACK |
| Handshake (new, with TLS 1.3) | 2 RTT | 2 RTT | 1 RTT (0-RTT resume) |
| Encryption | Optional | Effectively required | Always |
| Connection migration | No | No | Yes |

---

## 7. Version Negotiation {#negotiation}

- **ALPN** (TLS extension): during the TLS handshake the client offers `h2, http/1.1` and the server picks. There's no extra round trip.
- **HTTP/3 discovery**: the first connection goes over TCP, and the server advertises `Alt-Svc: h3=":443"; ma=86400`. The client tries QUIC on later connections (racing with TCP). The **HTTPS DNS record** (SVCB) can advertise h3 support before the first connection.
- `Upgrade: websocket` (HTTP/1.1) / extended CONNECT (RFC 8441) for WebSockets over h2.

---

## 8. Headers Backend Engineers Must Know {#headers}

| Header | Purpose |
|---|---|
| `Host` / `:authority` | Target host (virtual hosting, routing) |
| `Content-Type` / `Accept` | Media type of body / acceptable response types |
| `Content-Length` / `Transfer-Encoding: chunked` | Body framing |
| `Content-Encoding` / `Accept-Encoding` | Compression (gzip, br, zstd) |
| `Authorization` / `WWW-Authenticate` | Credentials / challenge |
| `Cache-Control`, `ETag`, `Last-Modified`, `If-None-Match`, `If-Modified-Since`, `Vary`, `Age`, `Expires` | Caching (see the caching notes) |
| `If-Match`, `If-Unmodified-Since` | Optimistic concurrency |
| `Location` | Redirect target / created resource |
| `Retry-After` | When to retry (429/503) |
| `X-Forwarded-For` / `X-Forwarded-Proto` / `X-Forwarded-Host` / standardized **`Forwarded`** (RFC 7239) | Original client info through proxies. **Only trust them from known proxies**, because clients can spoof them (IP-based rate limiting/auth bypass) |
| `X-Request-Id`, `traceparent` / `tracestate` (W3C Trace Context) | Correlation and distributed tracing |
| `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`, `Permissions-Policy` | Security headers (see [`api-security/csp-owasp-server-security.md`](../api-security/csp-owasp-server-security.md)) |
| `Access-Control-*`, `Origin` | CORS (see [`api-security/cors.md`](../api-security/cors.md)) |
| `Connection`, `Keep-Alive`, `Upgrade` | Hop-by-hop (not forwarded by proxies; forbidden in h2) |
| `User-Agent`, `Referer` | Client info |
| `Idempotency-Key` | Safe retries of POST |
| `Server-Timing` | Backend timing breakdown visible in browser devtools |
| `Early-Data` / 425 Too Early | 0-RTT replay protection |
| `Alt-Svc` | Advertise HTTP/3 |

Header names are case-insensitive (lowercase on the wire in h2/h3). Size limits exist at proxies (Nginx default 8 KB per header line by `large_client_header_buffers`), and huge cookies or JWTs cause `431 Request Header Fields Too Large` or 400 errors.

---

## 9. Cookies {#cookies}

```
Set-Cookie: session=abc123; Path=/; Domain=example.com; Max-Age=3600; Secure; HttpOnly; SameSite=Lax
```
- `Secure`: only over HTTPS. `HttpOnly`: not readable by JS, which mitigates XSS token theft.
- `SameSite`: `Strict` (never cross-site), `Lax` (default in modern browsers; sent on top-level GET navigations), `None` (cross-site allowed; requires Secure). It's the primary CSRF defense.
- `Domain` widens scope to subdomains (omit it for host-only cookies, which is safer).
- Prefixes: `__Host-` (Secure, Path=/, no Domain → locked to the host) and `__Secure-`.
- Third-party cookies are being restricted (Safari/Firefox block them, and Chrome's plans have shifted repeatedly). Use first-party architectures.
- Size limit ~4 KB per cookie. They're sent with **every** request to matching paths, which costs bandwidth.
- **CHIPS** (`Partitioned` attribute) for embedded third-party contexts.

---

## 10. Content Negotiation, Compression, Ranges {#content}

- **Compression**: gzip (universal), **Brotli** (`br`, ~15–25% smaller for text, standard on CDNs), **zstd** (supported in Chrome 123+ and Firefox; fast). Compress JSON/HTML/JS/CSS. Don't compress already-compressed media. Watch CPU on dynamic responses (compression level 4–6 for dynamic content). Security: BREACH attack (compressing secrets alongside attacker-controlled input over TLS).
- **Range requests**: `Range: bytes=0-1023` → `206 Partial Content`. Used for video seeking, resumable downloads, and parallel downloads (`Accept-Ranges: bytes`).
- **Content negotiation**: `Accept`, `Accept-Language`, `Accept-Encoding` → server picks a representation and sets `Vary` accordingly.
- **Streaming responses**: chunked (h1) or DATA frames (h2), plus SSE (`text/event-stream`), NDJSON streaming (common for LLM token streaming APIs).

---

## 11. Request Lifecycle and Latency Budget {#lifecycle}

`https://api.example.com/orders` from a phone:
1. **DNS** lookup: 0–100+ ms (cached or not).
2. **TCP handshake**: 1 RTT.
3. **TLS 1.3 handshake**: 1 RTT (0 with resumption/0-RTT; QUIC merges steps 2+3).
4. Request sent. Possibly through a CDN/edge (TLS terminated near the user, with warm connections to the origin).
5. **Load balancer** → reverse proxy → app server.
6. App: auth, DB queries, downstream calls (each with its own network hops).
7. **TTFB** (time to first byte).
8. Content download (bandwidth, slow start, compression).

Mobile 4G RTT is ~50–100 ms, so steps 1–3 can cost 300 ms before your code even runs. Wins: connection reuse, edge termination (CDN), HTTP/2/3, fewer round trips (batch, BFF/GraphQL), DNS prefetch/preconnect, smaller payloads.

---

## 12. Backend Implications {#backend}

- **Keep-alive everywhere**: client → LB → app → downstream. Reuse connections in HTTP clients (a shared `http.Client` in Go, `requests.Session` in Python, a keep-alive agent in Node, a pooled `HttpClient` in Java/.NET).
- **Timeouts aligned** across the chain (see TCP notes): server keep-alive timeout > LB idle timeout.
- **Proxies normalize protocols**: browsers speak h3 to Cloudflare, Cloudflare speaks h2 or h1.1 to your ALB, the ALB speaks h1.1 to targets (or h2/gRPC if configured). Know where TLS terminates and which headers are added (`X-Forwarded-*`).
- **Request smuggling**: ensure front and back servers parse framing identically. Prefer h2 end-to-end or normalize at the edge, and reject ambiguous requests (both CL and TE).
- **Large uploads**: stream them, don't buffer. Use `Expect: 100-continue` or presigned direct-to-S3 uploads.
- **Graceful shutdown**: stop accepting, send `Connection: close` / h2 `GOAWAY`, drain in-flight requests, then exit (Kubernetes preStop + terminationGracePeriod).
- **Max header/body sizes and request timeouts** at the edge protect against slowloris-style attacks.

---

## 13. Interview Questions {#qa}

1. What are the key differences between HTTP/1.1, HTTP/2, and HTTP/3?
2. What is head-of-line blocking at the HTTP layer vs the TCP layer? Which versions solve which?
3. How does HTTP/2 multiplexing work? Why does it make L4 load balancing of gRPC uneven?
4. Why does QUIC run over UDP? What is connection migration?
5. What is 0-RTT and what's the security risk?
6. How does a client discover a server supports HTTP/3?
7. Which headers carry the real client IP through proxies, and why can't you trust them blindly?
8. Explain cookie attributes `HttpOnly`, `Secure`, `SameSite`.
9. Walk through the latency of an HTTPS request from DNS to the last byte.
10. What is HTTP request smuggling?
