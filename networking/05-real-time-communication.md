# Real-Time Communication: Polling, Long Polling, SSE, WebSockets, WebRTC, Streaming RPC

> WebSocket-focused architectures (Twitter, LinkedIn, and others) are covered in [`system-desing-websocket-focused.md`](../system-desing-websocket-focused.md). This note compares **all** the real-time options and covers how to scale them.

## Table of Contents
1. [The Options at a Glance](#options)
2. [Short Polling](#polling)
3. [Long Polling](#longpolling)
4. [Server-Sent Events (SSE)](#sse)
5. [WebSockets](#websockets)
6. [WebTransport](#webtransport)
7. [WebRTC](#webrtc)
8. [gRPC Streaming](#grpc)
9. [Mobile Push and Webhooks](#push)
10. [Scaling Real-Time Systems](#scaling)
11. [Delivery Guarantees, Ordering, Reconnection](#guarantees)
12. [Decision Guide](#decision)
13. [Interview Questions](#qa)

---

## 1. Options at a Glance {#options}

| Technique | Direction | Transport | Latency | Complexity | Typical use |
|---|---|---|---|---|---|
| Short polling | Client pulls | HTTP | Poll interval | Lowest | Dashboards that refresh every 30 s, job status |
| Long polling | Server → client (emulated) | HTTP | Near real-time | Low–medium | Legacy chat, fallback transport, simple notifications |
| **SSE** | Server → client | HTTP (text/event-stream) | Real-time | Low | Live feeds, notifications, **LLM token streaming**, progress updates |
| **WebSocket** | Bidirectional | TCP (upgraded HTTP) | Real-time | Medium–high | Chat, multiplayer games, collaborative editing, trading |
| WebTransport | Bidirectional, multi-stream, unreliable datagrams | HTTP/3 (QUIC) | Real-time | High (newer) | Games, media, low-latency streaming |
| **WebRTC** | Peer-to-peer (or via SFU) media + data | UDP (SRTP/SCTP over DTLS) | Lowest (<500 ms media) | Highest | Video/voice calls, screen sharing, P2P data |
| gRPC streaming | Uni/bidirectional | HTTP/2 | Real-time | Medium | Service-to-service streams, mobile clients |
| Push notifications (APNs/FCM) | Server → device (via OS vendor) | Vendor channel | Seconds | Medium | Wake up mobile apps |
| Webhooks | Server → server | HTTP POST | Seconds | Medium | Integrations |

---

## 2. Short Polling {#polling}

```js
setInterval(() => fetch('/api/jobs/123').then(r => r.json()).then(render), 5000);
```
- Simple, stateless, cache-friendly, works everywhere.
- Wastes requests when nothing changes. Latency ≤ interval. Load = clients ÷ interval (100k clients at 5 s = 20k rps).
- Improve with conditional requests (`ETag`/`304`), adaptive intervals (back off when idle), and `Retry-After` hints.
- Often the **right** answer for low-frequency updates. Don't over-engineer.

---

## 3. Long Polling {#longpolling}

The client sends a request, and the server **holds it open** until data is available or a timeout (~25–60 s) expires, then responds. The client immediately re-requests.
```
Client ── GET /events?since=1042 ──► Server (waits…)
       ◄── 200 [event 1043] ──────── (data arrived)
Client ── GET /events?since=1043 ──► (waits…)
       ◄── 204 (timeout, nothing new)
Client ── GET /events?since=1043 ──► …
```
- Near real-time over plain HTTP, through any proxy/firewall.
- Overhead: a full HTTP request per message batch. The server holds many idle requests (needs async I/O).
- Cursor (`since=`) gives at-least-once and gap recovery.
- Used by early Facebook chat, Comet-era apps, and still as a Socket.IO fallback and in some message queue APIs (SQS `WaitTimeSeconds` long polling, Kafka fetch with `fetch.max.wait.ms`, Telegram bot `getUpdates`).

---

## 4. Server-Sent Events (SSE) {#sse}

A standard (HTML Living Standard) for a **one-way stream** from server to client over a long-lived HTTP response.
```
GET /stream HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

id: 1043
event: order.updated
data: {"id":"ord_1","status":"shipped"}

: heartbeat comment line (keeps proxies from timing out)

id: 1044
data: {"id":"ord_2","status":"paid"}
retry: 5000
```
Browser:
```js
const es = new EventSource('/stream', { withCredentials: true });
es.addEventListener('order.updated', (e) => update(JSON.parse(e.data)));
es.onerror = () => { /* EventSource auto-reconnects */ };
```
Key features:
- **Automatic reconnection** built into the browser, with a **`Last-Event-ID`** header so the server resumes from the last event. Gap-free delivery is easy if the server keeps a replay buffer.
- Plain HTTP: works with HTTP/2 multiplexing (many streams on one connection, so the old 6-connections-per-domain limit of HTTP/1.1 no longer bites), standard auth via cookies, CORS, compression, and normal LBs.
- Text only (UTF-8). Binary needs base64.
- One-directional: the client sends data via normal HTTP requests (often fine: "commands up via POST, events down via SSE").
- Native `EventSource` can't set custom headers (use cookies, or fetch-based SSE libraries for `Authorization` headers).

Why SSE is underrated: most "real-time" features are server → client notifications. SSE gives you that with HTTP semantics and auto-reconnect, without WebSocket's operational complexity. **LLM APIs (OpenAI, Anthropic, and others) stream tokens with SSE.**

Server gotchas: disable proxy buffering (`X-Accel-Buffering: no` for Nginx, `proxy_buffering off`), send heartbeats every ~15–30 s, set long proxy read timeouts, flush after each event, and use async servers (no thread per connection).

---

## 5. WebSockets {#websockets}

RFC 6455: a **full-duplex, message-oriented** channel over a single TCP connection, established through an HTTP Upgrade.
```
GET /ws HTTP/1.1
Host: chat.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat.v2
Origin: https://app.example.com

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
- After the upgrade: **frames** (text, binary, ping, pong, close), with 2–14 byte headers. Client → server frames are **masked** (XOR, to protect against cache poisoning of intermediaries).
- `wss://` = WebSocket over TLS (always use it; plain `ws://` gets mangled by middleboxes).
- No built-in: reconnection, message acknowledgments, ordering across reconnects, rooms/channels, or auth refresh. **You build a protocol on top** (or use Socket.IO, SignalR, Phoenix Channels, Centrifugo, Ably/Pusher, ActionCable, STOMP, MQTT over WS, GraphQL `graphql-ws`).
- Compression extension `permessage-deflate` (memory cost per connection; watch out).
- Heartbeats: ping/pong to detect dead connections and keep NAT/LB idle timers alive.

Security:
- **Validate the `Origin` header** on upgrade. Browsers send cookies with WS handshakes, so without an Origin check you're open to **Cross-Site WebSocket Hijacking** (CSRF for WebSockets).
- Authenticate on connect (cookie session, or a short-lived token in the first message or query param; avoid long-lived tokens in URLs because they get logged). **Re-validate** periodically (token expiry, permission revocation) and close connections of logged-out or banned users.
- Rate-limit messages per connection, cap message sizes, validate every message like an API request.

---

## 6. WebTransport {#webtransport}

A newer W3C/IETF API over **HTTP/3 (QUIC)**: multiple bidirectional/unidirectional streams plus **unreliable datagrams**, no HOL blocking between streams, and connection migration. It's suitable for games, live media, and anything that prefers fresh data over retransmitted stale data. Supported in Chromium-based browsers and Firefox, with Safari support emerging. Server support is still maturing. It's a WebSocket successor candidate for demanding use cases.

---

## 7. WebRTC {#webrtc}

Real-time **audio/video/data between browsers/devices**, ideally peer-to-peer:
- **Signaling** (not specified by WebRTC): peers exchange **SDP** offers/answers (codecs, media parameters) and **ICE candidates** (possible network addresses) via your server, usually over WebSocket.
- **NAT traversal via ICE**:
  - **STUN** server: tells a peer its public IP:port (server-reflexive candidate). Works for most NATs.
  - **TURN** server: relays media when direct P2P fails (symmetric NATs, strict firewalls). Roughly 10–20% of calls need it, and it costs bandwidth (coturn, Twilio/Cloudflare TURN).
- Media: SRTP over UDP, encrypted with **DTLS-SRTP** (mandatory encryption). Congestion control (GCC), jitter buffers, simulcast, SVC.
- **Data channels**: SCTP over DTLS. Reliable/ordered or unreliable/unordered, which is useful for games and file transfer.
- Multi-party architectures:
  - **Mesh**: everyone connects to everyone. O(n²) streams, OK for ≤4 participants.
  - **SFU (Selective Forwarding Unit)**: each participant uploads once, and the SFU forwards streams to the others (with simulcast layers per receiver bandwidth). This is the industry standard (Zoom, Meet, Discord, LiveKit, mediasoup, Janus, Jitsi).
  - **MCU**: the server mixes streams into one. CPU-heavy, legacy.
- Live streaming at scale (one → millions): WebRTC for sub-second "real-time" streams (Cloudflare Stream, LiveKit, Millicast via WHIP/WHEP), or HLS/DASH (CDN-friendly, 2–30 s latency; LL-HLS for ~2–5 s).

---

## 8. gRPC Streaming {#grpc}

Server-streaming, client-streaming, and bidirectional RPCs over HTTP/2 streams. They're typed (protobuf) with flow control and deadlines. Used for service-to-service event streams, mobile apps (gRPC native), and IoT. Browser support needs **gRPC-Web** or **Connect** (no client streaming in gRPC-Web). See [`api/grpc.md`](../api/grpc.md).

---

## 9. Mobile Push and Webhooks {#push}

- Mobile apps can't keep sockets open in the background (OS kills them for battery). Use **APNs** (Apple) / **FCM** (Google): your server sends to the vendor, which delivers to the device. Payload limits are ~4 KB, and delivery isn't guaranteed (best effort, collapsible). Use them to "wake up and sync". Treat push tokens as rotating and handle invalid-token feedback.
- Web Push (Push API + service workers, VAPID keys) for browsers.
- Server-to-server: **webhooks** (see [`api/api-design-best-practices.md`](../api/api-design-best-practices.md#webhooks)).

---

## 10. Scaling Real-Time Systems {#scaling}

Persistent connections change the scaling model:

**1. Connection capacity per node**
- Use event-driven servers (epoll/kqueue-based: Node.js, Go, Netty, Elixir/Phoenix, Rust tokio, uWebSockets). A well-tuned node holds **hundreds of thousands to millions** of idle connections (WhatsApp famously ran ~2M connections per Erlang server; Phoenix demonstrated 2M channels on one box).
- Memory per connection (buffers, app state, TLS state) is the main cost. Tune file descriptors, ephemeral ports at proxies, and kernel buffers.

**2. Fan-out across nodes: the backplane**
User A is connected to node 1, and user B to node 7. A message from A to B must reach node 7:
- **Pub/Sub backplane**: Redis Pub/Sub or Streams, NATS, Kafka, RabbitMQ. Each node subscribes to channels for the users and rooms it hosts.
- **Connection registry / presence**: `user_id → node_id` in Redis (with TTL heartbeats), so messages are routed directly to the owning node.
- Room/channel sharding for large groups. Massive channels (a live event with 1M viewers) need hierarchical fan-out (edge nodes subscribe once and fan out locally).

**3. Load balancing**
- L4 or L7 with WebSocket support (upgrade-aware), long idle timeouts (≥ heartbeat interval), and connection-count-based balancing. Ensure LB connection limits suffice.
- New nodes start empty and old ones stay full. Rebalance slowly (close a fraction of connections with a "reconnect" hint) rather than all at once.

**4. Deploys and failures**
- Draining a node disconnects all its clients, so they reconnect elsewhere. Use **jittered exponential backoff** on clients, otherwise a deploy or crash causes a **reconnect storm** (thundering herd) that can take down the auth service and backplane.
- Send a "reconnect soon" control message before shutdown, and spread reconnects over a window.

**5. State and presence**
- Online/offline presence via heartbeats with TTL. Presence fan-out to friends is expensive at scale, so batch and throttle it, or compute it lazily.
- Typing indicators and read receipts are ephemeral. Don't persist each one.

**6. Managed options**: AWS API Gateway WebSocket APIs (connection IDs, `@connections` API to push), AWS AppSync/IoT Core, Azure Web PubSub/SignalR Service, Google Firebase Realtime/Firestore listeners, Ably, Pusher, PubNub, Cloudflare Durable Objects (one object per room, holding WebSockets with hibernation), Centrifugo (self-hosted), Supabase Realtime.

---

## 11. Delivery Guarantees, Ordering, Reconnection {#guarantees}

The real-time channel is **not a durable log**. Design for loss:
- Persist messages to a durable store first (DB / Kafka), then push notifications over the socket. The socket is a fast path, not the source of truth.
- Give each message a **monotonic sequence number per conversation/stream**. On reconnect, the client sends its last seen sequence (`Last-Event-ID` / `since=`) and the server replays the gap from storage.
- **Client-side deduplication** by message ID (at-least-once delivery means duplicates happen).
- **Acks** from the client for important messages (delivered/read receipts). Retransmit if not acked.
- Ordering: guarantee it per conversation (partition by conversation ID in the backplane/Kafka), not globally.
- Optimistic UI: show a sent message immediately with a client-generated ID, then reconcile with the server's ack.
- Backpressure: slow consumers (bad mobile network) must not exhaust server memory. Use bounded per-connection send queues, and drop or close when they're full. For non-critical streams, send the latest state only (conflation: skip intermediate price ticks).

---

## 12. Decision Guide {#decision}

```
Need audio/video or P2P?                           → WebRTC (SFU for groups)
Server → client only (notifications, feeds, LLM tokens, progress)?
                                                   → SSE (or polling if updates are rare)
Frequent bidirectional low-latency messages (chat, games, collaborative editing, trading)?
                                                   → WebSocket (WebTransport for advanced cases)
Updates every ≥30 s, or simplicity matters most?   → Short polling with ETags
Mobile app in background?                          → Push notifications (+ sync on open)
Service-to-service streaming?                      → gRPC streaming or Kafka
Server-to-server event notifications?              → Webhooks
```

---

## 13. Interview Questions {#qa}

1. Compare polling, long polling, SSE, and WebSockets. When would you choose each?
2. How does the WebSocket handshake work? What security check is essential on upgrade?
3. Why is SSE a good fit for streaming LLM responses?
4. Design the real-time layer for a chat app with 10M concurrent users.
5. How do you route a message to a user connected to a different server?
6. What happens during a deploy of WebSocket servers, and how do you avoid a reconnect storm?
7. How do you guarantee no messages are lost when a client's connection drops?
8. What are STUN and TURN in WebRTC? What's an SFU?
9. Why can't mobile apps rely on WebSockets in the background?
