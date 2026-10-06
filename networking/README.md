# Networking for Backend Engineers

Every request your service handles crosses DNS, TCP/QUIC, TLS, HTTP, and several proxies. These notes cover each layer in enough depth to debug production issues and to reason about latency in system design.

| # | Note | Topics |
|---|---|---|
| 01 | [TCP/IP, UDP & Sockets](01-tcp-ip-udp-and-sockets.md) | OSI/TCP-IP, IP/NAT/MTU, handshakes, TIME_WAIT/CLOSE_WAIT, retransmission, flow & congestion control (CUBIC/BBR), Nagle, UDP, socket API, accept queues, keepalives, Linux tuning, debugging tools |
| 02 | [DNS](02-dns.md) | Hierarchy, resolution, record types (incl. SVCB/HTTPS, CAA, SPF/DKIM/DMARC), TTL strategy, GeoDNS/anycast, Kubernetes ndots, DNSSEC/DoH, subdomain takeover, rebinding |
| 03 | [HTTP/1.1, HTTP/2, HTTP/3 & QUIC](03-http-evolution-1-1-2-3-and-quic.md) | Semantics, framing, multiplexing, HPACK/QPACK, HOL blocking, QUIC handshakes and migration, ALPN, headers, cookies, compression, request lifecycle, smuggling |
| 04 | [Load Balancers, Proxies, Gateways & Meshes](04-load-balancers-proxies-and-gateways.md) | L4 vs L7, algorithms (P2C, consistent hashing, Maglev), health checks, stickiness, TLS modes, draining, global LB, client-side LB, API gateways, BFF, Envoy, service mesh |
| 05 | [Real-Time Communication](05-real-time-communication.md) | Polling, long polling, SSE, WebSockets, WebTransport, WebRTC (STUN/TURN/SFU), gRPC streaming, push, scaling with backplanes, delivery guarantees |

Related notes elsewhere:
- TLS & HTTPS: [`api-security/ssl-tls.md`](../api-security/ssl-tls.md), [`api-security/https.md`](../api-security/https.md), [`Roadmap/Connection-networking.md`](../Roadmap/Connection-networking.md)
- CORS: [`api-security/cors.md`](../api-security/cors.md)
- CDNs: [`caching/cdn/cdn.md`](../caching/cdn/cdn.md)
- Nginx/Apache: [`web-servers/`](../web-servers/)
- WebSocket architectures: [`system-desing-websocket-focused.md`](../system-desing-websocket-focused.md)
- Interview drills: [`interview-prep/devops/02-networking-dns-http.md`](../interview-prep/devops/02-networking-dns-http.md)

## Hands-on exercises
1. `curl -w` timing breakdown: write a `timing.txt` format file with `time_namelookup`, `time_connect`, `time_appconnect`, `time_starttransfer`, and `time_total`, then compare a cold request with a warm keep-alive one.
2. Capture a TLS 1.3 handshake plus an HTTP/2 request with `tcpdump` and open it in Wireshark (set `SSLKEYLOGFILE` to decrypt).
3. Write a tiny TCP echo server and client. Send 3 messages quickly and observe how they coalesce into one `recv()`. Then add length-prefix framing.
4. Generate TIME_WAIT exhaustion by opening and closing connections in a loop to one port, and watch `ss -s`. Fix it with connection reuse.
5. Run `dig +trace` on your domain and identify every delegation step.
6. Build an SSE endpoint and a WebSocket endpoint in your language. Kill the server mid-stream and implement resume with `Last-Event-ID` / sequence numbers.
7. Put Nginx in front of two app instances. Try round robin vs least_conn, add a health check, and do a zero-downtime restart.
