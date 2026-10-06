# TCP/IP, UDP, and Sockets: Networking Fundamentals for Backend Engineers

> When something is "slow" or "timing out" in production, the cause is often in this layer: handshakes, Nagle, TIME_WAIT, connection limits, MTU, retransmissions, keepalives. Understanding TCP makes you dramatically better at debugging distributed systems.

## Table of Contents
1. [Layered Models: OSI vs TCP/IP](#layers)
2. [IP: Addressing, Routing, Subnets, NAT, MTU](#ip)
3. [TCP: Reliable Byte Streams](#tcp)
4. [The Three-Way Handshake and Teardown](#handshake)
5. [TCP State Machine and TIME_WAIT](#states)
6. [Reliability: Sequence Numbers, ACKs, Retransmission](#reliability)
7. [Flow Control and Congestion Control](#congestion)
8. [Latency Killers: Nagle, Delayed ACK, Head-of-Line Blocking, Slow Start](#latency)
9. [UDP](#udp)
10. [Sockets API and How Servers Accept Connections](#sockets)
11. [Connection Limits: Ports, File Descriptors, Backlogs](#limits)
12. [Keepalives, Timeouts, and Dead Peers](#keepalive)
13. [Linux Tuning Knobs](#tuning)
14. [Debugging Toolkit](#tools)
15. [Interview Questions](#qa)

---

## 1. Layered Models {#layers}

| OSI layer | TCP/IP layer | Examples | Unit |
|---|---|---|---|
| 7 Application | Application | HTTP, gRPC, DNS, SMTP, SSH, TLS (often placed here) | Message |
| 6 Presentation | Application | Encoding, encryption (TLS), compression | |
| 5 Session | Application | Session management | |
| 4 Transport | Transport | **TCP, UDP, QUIC** (over UDP) | Segment / datagram |
| 3 Network | Internet | **IP** (v4/v6), ICMP, routing | Packet |
| 2 Data link | Link | Ethernet, Wi-Fi, ARP, VLANs | Frame |
| 1 Physical | Link | Cables, radio | Bits |

Encapsulation: HTTP data → TCP segment (src/dst port) → IP packet (src/dst IP) → Ethernet frame (src/dst MAC).

Load balancers are described by layer: **L4** (TCP/UDP: sees IPs and ports) vs **L7** (HTTP: sees paths, headers, cookies).

---

## 2. IP {#ip}

- **IPv4**: 32-bit addresses (`10.0.1.25`). **CIDR** notation `10.0.0.0/16` = first 16 bits are the network (65,536 addresses).
- **Private ranges** (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Plus `127.0.0.0/8` loopback, `169.254.0.0/16` link-local (cloud metadata service lives at `169.254.169.254`, a classic SSRF target).
- **IPv6**: 128-bit (`2001:db8::1`). No NAT needed, but dual-stack quirks exist (happy eyeballs, AAAA records).
- **Routing**: each hop forwards based on the longest-prefix match in its routing table. **TTL** (hop limit) decrements at each hop. `traceroute` exploits TTL expiry to map the path.
- **NAT** (Network Address Translation): many private IPs share one public IP, and the router rewrites source IP:port. Consequences: inbound connections need port forwarding, NAT tables have **idle timeouts** (AWS NAT Gateway drops idle TCP flows after 350 s, which silently kills idle DB connections without keepalives), and connection tracking tables (`nf_conntrack`) can fill up on busy hosts.
- **MTU**: max frame payload, usually 1500 bytes on Ethernet (9001 "jumbo frames" inside AWS VPCs). **MSS** = MTU − IP/TCP headers (1460). Packets larger than the path MTU get fragmented or dropped. **Path MTU Discovery** relies on ICMP "fragmentation needed" messages, and firewalls blocking ICMP cause **PMTU black holes**: small requests work, large responses hang. Classic in VPNs and tunnels (lower MTU due to encapsulation overhead).
- **ICMP**: ping (echo), unreachable, time exceeded. Don't block it blindly.
- **ARP**: IP → MAC resolution on the local network.

---

## 3. TCP {#tcp}

TCP provides a **reliable, ordered, byte-stream** abstraction over unreliable IP:
- **Connection-oriented** (handshake, per-connection state on both ends).
- **Reliable**: retransmits lost segments.
- **Ordered**: reassembles by sequence number.
- **Byte stream, not messages**: there are no message boundaries. One `send()` of 10 KB may arrive as several `recv()` calls, and several sends may coalesce. **Application protocols must frame messages** (length prefix, delimiter, HTTP Content-Length or chunked encoding). This is the source of many beginner bugs.
- **Flow control** (don't overwhelm the receiver) and **congestion control** (don't overwhelm the network).
- **Full duplex**.

TCP header essentials: source/dest port, sequence number, acknowledgment number, flags (SYN, ACK, FIN, RST, PSH, URG, ECE, CWR), window size, checksum, options (MSS, window scale, SACK permitted, timestamps).

A connection is identified by the **4-tuple**: (src IP, src port, dst IP, dst port).

---

## 4. Handshake and Teardown {#handshake}

```
Client                              Server
  │ ── SYN (seq=x) ───────────────►  │   SYN_SENT → (server: SYN_RECEIVED)
  │ ◄── SYN-ACK (seq=y, ack=x+1) ──  │
  │ ── ACK (ack=y+1) ─────────────►  │   ESTABLISHED (both)
  │ ══ data ════════════════════════ │
```
- Costs **1 RTT** before the client can send data. Then TLS 1.3 adds 1 more RTT (TLS 1.2: 2 RTTs). So a new HTTPS connection costs 2–3 RTTs before the first byte of the request. Cross-continent (150 ms RTT), that's 300–450 ms of pure setup. **This is why connection reuse (keep-alive, pooling) and CDNs/edge termination matter so much.**
- **TCP Fast Open** (TFO) lets data ride on the SYN for repeat connections (limited deployment).
- **SYN flood**: attackers send SYNs without completing handshakes, filling the server's SYN queue. Mitigated by **SYN cookies** (the server encodes state in the sequence number instead of allocating it).

Teardown (four-way):
```
Active closer                       Passive closer
  │ ── FIN ───────────────────────►  │  FIN_WAIT_1 → CLOSE_WAIT
  │ ◄── ACK ──────────────────────   │  FIN_WAIT_2
  │ ◄── FIN ──────────────────────   │  LAST_ACK   (after app calls close())
  │ ── ACK ───────────────────────►  │  CLOSED
  TIME_WAIT (2×MSL, Linux: 60 s) → CLOSED
```
- **Half-close**: one side can stop sending while still receiving.
- **RST** (reset): abrupt termination (connection to a closed port, app crash, `SO_LINGER` with 0, firewall/LB killing idle flows). "Connection reset by peer" = you received a RST.

---

## 5. States and TIME_WAIT {#states}

States you'll see in `ss -tan`: `LISTEN`, `SYN-SENT`, `SYN-RECV`, `ESTAB`, `FIN-WAIT-1/2`, `CLOSE-WAIT`, `LAST-ACK`, `TIME-WAIT`, `CLOSING`.

**TIME_WAIT** (on the side that closes first) lasts 2×MSL (Linux hardcodes 60 s). Purposes:
1. Ensure the final ACK can be retransmitted if lost.
2. Ensure delayed duplicate packets from the old connection don't corrupt a new connection with the same 4-tuple.

Problems: a client opening and closing many short connections to the **same destination IP:port** can exhaust **ephemeral ports** (default range `32768–60999` ≈ 28k ports), because each one sits in TIME_WAIT for 60 s. That gives ~470 new connections/s sustained per destination before `EADDRNOTAVAIL` / "cannot assign requested address".
Fixes: **reuse connections** (pooling, HTTP keep-alive). That's the real fix. Others: widen `net.ipv4.ip_local_port_range`, `net.ipv4.tcp_tw_reuse=1` (safe for outgoing connections with timestamps), and more source IPs. **Never use `tcp_tw_recycle`** (removed in Linux 4.12; it broke clients behind NAT).

**CLOSE_WAIT piling up** = your application received FIN but **never called close()**. That's a socket/connection leak in your code (unreturned pool connections, unclosed HTTP response bodies in Go: `defer resp.Body.Close()`).

---

## 6. Reliability {#reliability}

- Each byte has a **sequence number**. The receiver sends **cumulative ACKs** ("I've received everything up to N").
- **Retransmission timeout (RTO)**: computed from smoothed RTT and variance (min 200 ms on Linux, initial 1 s). It doubles on each timeout (exponential backoff). A SYN that gets no answer is retried at 1, 3, 7, 15, 31 s… That's why connecting to a black-holed IP hangs for about 2 minutes (`tcp_syn_retries=6`) unless you set a **connect timeout**.
- **Fast retransmit**: 3 duplicate ACKs → retransmit without waiting for the RTO.
- **SACK** (selective ACK): the receiver reports which out-of-order blocks it has, so the sender retransmits only the missing ones.
- Packet loss of even 1–2% significantly reduces TCP throughput and inflates tail latency (each loss can cost an RTO).

---

## 7. Flow Control and Congestion Control {#congestion}

**Flow control** (receiver-driven): the receiver advertises a **receive window** (free buffer space). The sender can't have more unacknowledged bytes in flight than that. **Window scaling** (option) allows windows beyond 64 KB.
**Bandwidth-delay product (BDP)** = bandwidth × RTT = bytes needed in flight to fill the pipe. 1 Gbps × 100 ms = 12.5 MB. If socket buffers are smaller, throughput is capped at window/RTT, no matter how fat the link is. Linux autotunes buffers (`tcp_rmem`/`tcp_wmem` max values matter for long fat networks).

**Congestion control** (sender-driven, protects the network): the sender keeps a **congestion window (cwnd)**. Bytes in flight ≤ min(cwnd, rwnd).
- **Slow start**: cwnd starts small (**initial window = 10 segments ≈ 14 KB**, RFC 6928) and **doubles every RTT** until loss or the slow start threshold. So a new connection can't immediately use full bandwidth, and a 1 MB response over a fresh connection needs several RTTs. (This is another reason connection reuse helps, and why web performance folks want critical resources within the first ~14 KB.)
- **Congestion avoidance**: linear growth (additive increase), cut on loss (multiplicative decrease): **AIMD**.
- Algorithms: Reno, NewReno, **CUBIC** (Linux default), **BBR** (Google: model-based, estimates bottleneck bandwidth and RTT rather than reacting to loss; better on lossy or long links, used by Google, YouTube, and many CDNs). Set with `net.ipv4.tcp_congestion_control=bbr` + `fq` qdisc.
- **ECN**: routers mark packets instead of dropping them.
- Idle connections may restart slow start after idle (`tcp_slow_start_after_idle=1` by default; disabling it helps long-lived connections that send in bursts, like HTTP/2 or gRPC).

---

## 8. Latency Killers {#latency}

- **Nagle's algorithm**: buffers small writes until previous data is ACKed, to avoid floods of tiny packets. Combined with the receiver's **delayed ACK** (waits up to ~40 ms (Linux) to piggyback ACKs), it causes the infamous **40 ms (or 200 ms) latency** for request/response protocols that do write-write-read (e.g., header then body in separate writes). Fix: `TCP_NODELAY` (most HTTP clients, DB drivers, and Redis clients set it), or write the whole message in one call.
- **Head-of-line (HOL) blocking**: TCP delivers bytes in order. One lost packet stalls everything behind it, even if those bytes belong to independent HTTP/2 streams. QUIC fixes this by moving streams into the transport (see the HTTP note).
- **Slow start** on new connections (above).
- **Too many round trips**: DNS + TCP + TLS + request. Each RTT matters more than bandwidth for typical API payloads. Latency is usually RTT-bound, not bandwidth-bound.
- **Bufferbloat**: oversized buffers in routers or the kernel cause queueing delay under load. fq_codel and BBR mitigate it.

---

## 9. UDP {#udp}

- Connectionless datagrams: no handshake, no reliability, no ordering, no congestion control (the app must be careful), and message boundaries are preserved. Only an 8-byte header.
- Use cases: **DNS** (queries over UDP, TCP for large responses and zone transfers), **QUIC/HTTP/3**, video/voice (RTP, WebRTC), gaming, metrics (StatsD), DHCP, NTP, VPNs (WireGuard), service discovery gossip (SWIM/memberlist).
- Apps that need some reliability over UDP implement their own (QUIC, KCP, custom ACKs).
- Datagrams larger than the MTU fragment at the IP level (bad), so keep them under ~1200–1400 bytes on the internet.
- UDP amplification DDoS (DNS, NTP, memcached reflection): small spoofed request, huge response to the victim.

---

## 10. Sockets API {#sockets}

Server:
```c
fd = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, ...);   // bind even if old connections are in TIME_WAIT
bind(fd, {0.0.0.0, 8080});
listen(fd, backlog);                              // kernel completes handshakes and queues them
while (1) { conn = accept(fd); handle(conn); }    // or non-blocking + epoll (see concurrency notes)
```
Client: `socket()` → `connect()` (kernel picks an ephemeral source port) → `send/recv` → `close()`.

Kernel queues:
- **SYN queue** (half-open connections, `tcp_max_syn_backlog`).
- **Accept queue** (fully established, waiting for `accept()`): its size is `min(backlog, net.core.somaxconn)` (somaxconn default 4096 on modern kernels, 128 on old ones). If the app accepts too slowly and the queue overflows, new SYNs/ACKs get dropped, clients see timeouts or retransmits, and `ListenOverflows` increments in `netstat -s` / `nstat`.

Options worth knowing: `SO_REUSEADDR`, `SO_REUSEPORT` (multiple processes/threads bind the same port, and the kernel load-balances: Nginx `reuseport`, Envoy), `TCP_NODELAY`, `SO_KEEPALIVE` + `TCP_KEEPIDLE/INTVL/CNT`, `SO_LINGER`, `SO_RCVBUF/SNDBUF`, `TCP_USER_TIMEOUT` (max time unacked data can remain before the connection is dropped; good for detecting dead peers fast), `TCP_DEFER_ACCEPT`, `TCP_FASTOPEN`.

Unix domain sockets: IPC on the same host (no TCP overhead). Used by Postgres local connections, Docker daemon, and sidecars.

---

## 11. Connection Limits {#limits}

How many connections can a server hold? The 4-tuple means a **server** listening on one port can hold connections from many client IP:port pairs. The limit is memory and file descriptors, not ports. **C10K** (1999) then **C10M** were challenges about efficient I/O multiplexing (epoll), not port counts.

Real limits:
- **File descriptors**: `ulimit -n` (often 1024 by default!), systemd `LimitNOFILE`, `fs.file-max`. "Too many open files" (EMFILE) is a very common production error.
- **Ephemeral ports** on the **client** side per destination (see TIME_WAIT).
- Memory per connection (kernel socket buffers + app state: Go goroutine ~2–8 KB, a thread ~1 MB stack by default).
- `nf_conntrack_max` on hosts with iptables/NAT (Kubernetes nodes!). When the table is full, packets drop: `nf_conntrack: table full, dropping packet`.
- Load balancer limits (connections per target, idle timeouts).

---

## 12. Keepalives, Timeouts, Dead Peers {#keepalive}

A TCP connection can stay "ESTABLISHED" forever on one side after the other side vanished (host crash, cable pull, NAT/firewall dropped state). No packets flow, so nobody notices until a write fails, possibly minutes later after retransmissions (~15 min by default with `tcp_retries2=15`).

Defenses:
- **TCP keepalive**: probes after idle (Linux default idle **7200 s**, which is way too long for production). Set per socket: idle 60 s, interval 10 s, count 5. This also keeps NAT/LB idle timers from expiring.
- **Application-level heartbeats** (WebSocket ping/pong, gRPC keepalive pings, Kafka heartbeats, DB pool validation).
- **Timeouts at every layer**: connect, read/idle, write, overall request deadline.
- **`TCP_USER_TIMEOUT`** to bound how long unacknowledged data can sit.
- Match idle timeouts across the chain: client keep-alive < LB idle timeout < server keep-alive. Otherwise the LB closes the connection while the client still thinks it's reusable, and the client gets intermittent "connection reset" / 502 errors. (Classic: Node.js `keepAliveTimeout` (5 s default) < AWS ALB idle timeout (60 s) produces sporadic 502s. Set the server keep-alive timeout above the LB's.)

---

## 13. Linux Tuning Knobs {#tuning}

```bash
# Accept queue / backlog
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=8192
# Ephemeral ports and TIME_WAIT
sysctl -w net.ipv4.ip_local_port_range="10240 65535"
sysctl -w net.ipv4.tcp_tw_reuse=1
# Buffers for high BDP links
sysctl -w net.core.rmem_max=67108864 net.core.wmem_max=67108864
sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864" net.ipv4.tcp_wmem="4096 65536 67108864"
# Congestion control
sysctl -w net.core.default_qdisc=fq net.ipv4.tcp_congestion_control=bbr
# Keepalive defaults (apps should still set per socket)
sysctl -w net.ipv4.tcp_keepalive_time=60 net.ipv4.tcp_keepalive_intvl=10 net.ipv4.tcp_keepalive_probes=6
# Don't restart slow start after idle (long-lived connections)
sysctl -w net.ipv4.tcp_slow_start_after_idle=0
# File descriptors
ulimit -n 1048576
# Conntrack (on NAT/K8s nodes)
sysctl -w net.netfilter.nf_conntrack_max=1048576
```
Measure before and after. Defaults are reasonable for most services.

---

## 14. Debugging Toolkit {#tools}

| Tool | Use |
|---|---|
| `ss -tanp` / `ss -s` | Sockets by state, owning process; summary counts (TIME-WAIT, CLOSE-WAIT) |
| `ss -ti` | Per-connection TCP internals: rtt, cwnd, retrans, delivery rate |
| `netstat -s` / `nstat -az` | Protocol counters: retransmits, listen overflows, resets |
| `tcpdump -i any -nn port 5432 -w cap.pcap` + **Wireshark** | Packet-level truth |
| `curl -v -w "@timing.txt"` | DNS/connect/TLS/TTFB timing breakdown (`time_namelookup`, `time_connect`, `time_appconnect`, `time_starttransfer`) |
| `dig`, `nslookup` | DNS |
| `ping`, `mtr`, `traceroute` | Reachability, path, per-hop loss/latency |
| `nc` (netcat) / `telnet` | Is the port open? Raw protocol poking |
| `openssl s_client -connect host:443 -servername host` | TLS handshake and certificate inspection |
| `iperf3` | Throughput between hosts |
| `lsof -i :8080` | Who has the port |
| `conntrack -S` | Conntrack table stats |
| eBPF tools (`tcplife`, `tcpretrans`, `tcpconnect`, bpftrace) | Low-overhead production tracing |

Mental model for a "timeout" ticket: is it **DNS**, **connect** (no SYN-ACK: firewall/security group, wrong IP, accept queue overflow), **TLS** (cert, SNI, protocol mismatch), **request/TTFB** (server slow), or **transfer** (bandwidth, loss, window)? `curl -w` timings separate these immediately.

---

## 15. Interview Questions {#qa}

1. Walk through the TCP three-way handshake. Why three steps and not two?
2. What is TIME_WAIT, why does it exist, and how can it cause port exhaustion?
3. Lots of sockets in CLOSE_WAIT. What does that tell you?
4. Explain flow control vs congestion control.
5. What is slow start, and why does it make connection reuse important?
6. What's Nagle's algorithm and when should you disable it?
7. TCP vs UDP: when would you choose UDP?
8. Why can idle database connections silently die in the cloud? How do you prevent it?
9. A service intermittently returns 502s behind an ALB. What's a likely cause related to keep-alive?
10. What happens when the accept queue overflows?
11. Small requests work, large responses hang over a VPN. What's the likely cause?
