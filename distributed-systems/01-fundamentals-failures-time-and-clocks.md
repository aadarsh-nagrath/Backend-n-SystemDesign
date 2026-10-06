# Distributed Systems Fundamentals: Failures, Time, and Clocks

> A distributed system is "one in which the failure of a computer you didn't even know existed can render your own computer unusable" (Leslie Lamport). Every microservice architecture is a distributed system, so these fundamentals apply to everyday backend work.

## Table of Contents
1. [What Makes Distributed Systems Hard](#hard)
2. [The Eight Fallacies of Distributed Computing](#fallacies)
3. [System Models: Network, Timing, Failures](#models)
4. [Partial Failures and the Uncertainty of Timeouts](#partial)
5. [Detecting Failures: Heartbeats, Phi Accrual, SWIM](#detection)
6. [Physical Time: Clocks, NTP, Skew, Leap Seconds](#physical)
7. [Logical Time: Lamport Clocks](#lamport)
8. [Vector Clocks and Version Vectors](#vector)
9. [Hybrid Logical Clocks and TrueTime](#hlc)
10. [Ordering Guarantees: FIFO, Causal, Total](#ordering)
11. [Impossibility Results: FLP, Two Generals, CAP](#impossibility)
12. [Interview Questions](#qa)

---

## 1. What Makes Distributed Systems Hard {#hard}

Three fundamental problems:
1. **Partial failure**: some components fail while others keep working, and you often can't tell which (a slow node and a dead node look the same).
2. **Unreliable networks**: messages can be lost, delayed arbitrarily, duplicated, or reordered.
3. **No global clock**: each node has its own clock, which drifts. You can't use timestamps alone to order events across nodes.

On top of that: **concurrency** across nodes, **scale** (failures become constant: with 10,000 disks, several fail every day), and **heterogeneity** (versions, configs).

---

## 2. The Eight Fallacies (Peter Deutsch et al., Sun, 1994) {#fallacies}

1. **The network is reliable**: packets drop, switches fail, cables get cut, cloud networks have brownouts.
2. **Latency is zero**: cross-AZ ~1 ms, cross-region 50–150 ms. Chatty APIs multiply it.
3. **Bandwidth is infinite**: cross-AZ and egress traffic costs money and saturates.
4. **The network is secure**: assume a hostile network (zero trust, mTLS).
5. **Topology doesn't change**: autoscaling, pods rescheduled, IPs reassigned constantly.
6. **There is one administrator**: many teams, providers, and policies.
7. **Transport cost is zero**: serialization CPU, egress fees.
8. **The network is homogeneous**: mixed hardware, protocols, versions.

Every design review question ("what if this call times out?") traces back to these.

---

## 3. System Models {#models}

**Network model**:
- *Reliable links*: messages eventually delivered (with retries) but may be reordered.
- *Fair-loss links*: messages may be lost, but retrying enough eventually succeeds.
- *Arbitrary links*: an adversary can tamper (Byzantine), which needs cryptography.

**Timing model**:
- *Synchronous*: known bounds on message delay and clock drift. Unrealistic for real networks.
- *Asynchronous*: no timing bounds at all. Very pessimistic (FLP impossibility applies).
- **Partially synchronous**: the system behaves synchronously *most of the time* but sometimes exceeds bounds. **This is the realistic model** that practical consensus algorithms (Raft, Paxos) assume: safety always, liveness when the system is "synchronous enough".

**Failure model**:
- **Crash-stop**: a node halts and never returns.
- **Crash-recovery**: a node crashes and may restart, losing in-memory state but keeping durable storage. This is the realistic model for servers.
- **Omission**: a node fails to send or receive some messages.
- **Byzantine**: nodes behave arbitrarily or maliciously (lying, inconsistent messages). Handled by BFT protocols (PBFT, Tendermint, HotStuff), needed in blockchains and some aerospace systems. **Most backend systems assume non-Byzantine (crash-recovery) nodes** within a trusted datacenter.

---

## 4. Partial Failures and Timeouts {#partial}

You send a request and get no response. Possible causes:
1. The request was lost.
2. The remote node is down.
3. The remote node is slow (GC pause, overload, disk stall).
4. The remote node processed it, but the response was lost.
5. The response is delayed and will arrive later.

**You cannot distinguish these.** Case 4 is why retries need idempotency.

Timeouts are the only failure detector you have, and they're a trade-off:
- **Short timeout**: fast failure detection, but false positives (declaring a slow node dead), which can cause cascading failovers, duplicate work, and load on the remaining nodes.
- **Long timeout**: fewer false positives, but long waits during real failures.
- Set timeouts from **measured latency distributions** (e.g., slightly above p99.9), and adapt them (TCP's RTO, Phi accrual).

**Process pauses** are a hidden danger: a JVM stop-the-world GC pause (seconds), VM live migration, CPU steal, swapping, or a `SIGSTOP`. A node can be paused, declared dead by others, have its lease taken over, then wake up and keep acting as if it's still the leader. That's why distributed locks need **fencing tokens** (see the consensus note).

**Gray failures**: a component is partially broken (slow disk, packet loss on one link, one bad host behind a VIP), so health checks pass but users suffer. Detect these with end-to-end probes and outlier detection.

---

## 5. Failure Detection {#detection}

- **Heartbeats**: nodes periodically send "I'm alive". Missing N heartbeats means suspected dead.
- **Phi accrual failure detector** (Hayashibara et al., used by Cassandra and Akka): instead of a binary alive/dead, it outputs a suspicion level φ based on the statistical distribution of heartbeat inter-arrival times. The app picks a threshold. It adapts to network conditions.
- **SWIM protocol** (Scalable Weakly-consistent Infection-style Membership; HashiCorp memberlist/Serf/Consul): each node periodically pings a random member. If there's no ack, it asks k other members to ping it indirectly (to avoid false positives from a single bad link). Suspicion and confirmation are disseminated via **gossip** (piggybacked on pings). Load is O(1) per node, and detection time is O(log n) to spread. Lifeguard extensions reduce false positives.
- **Leases** (time-bounded authority) combine failure detection and safety: "you're leader until T", renewed periodically.

Perfect failure detection is impossible in asynchronous systems. All detectors trade completeness against accuracy.

---

## 6. Physical Time {#physical}

Each machine has a **quartz oscillator** that drifts (~10–100 ppm, i.e. roughly 1–10 seconds per day uncorrected).

Two kinds of clocks:
| | Time-of-day (wall clock) | Monotonic clock |
|---|---|---|
| API | `System.currentTimeMillis()`, `time.time()`, `Date.now()`, `CLOCK_REALTIME` | `System.nanoTime()`, `time.monotonic()`, `performance.now()`, `CLOCK_MONOTONIC` |
| Meaning | Seconds since epoch; comparable across machines (approximately) | Arbitrary origin; only differences on the same machine are meaningful |
| Can jump backward | **Yes** (NTP corrections, manual changes, leap seconds) | No |
| Use for | Timestamps shown to humans, cross-machine approximate times | **Measuring durations, timeouts, rate limiting** |

**Bug pattern**: measuring elapsed time with wall clock time → negative durations or huge timeouts when NTP steps the clock.

**NTP** (Network Time Protocol) synchronizes clocks via stratum servers. Typical accuracy is ~1–10 ms on a LAN, and tens of ms over the internet. It's worse when there's network congestion or misconfiguration. **chrony** is the modern implementation, and cloud providers offer precise time services (AWS Time Sync Service with microsecond accuracy on supported instances via PTP, Google public NTP with leap smearing).

**Leap seconds**: an extra second (23:59:60) inserted occasionally. They have caused outages (the 2012 Linux kernel livelock bug affected Reddit, LinkedIn, and Mozilla; Cloudflare's 2017 DNS bug from negative time). Google, AWS, and others **smear** the leap second over 24 h. The CGPM decided to abolish leap seconds by 2035.

**Consequences for system design**:
- Don't rely on synchronized clocks for **correctness** (ordering writes, LWW conflict resolution, lock expiry) unless you understand the error bounds.
- **Last-Write-Wins by timestamp** silently loses data when clocks skew: node A with a clock 50 ms fast "wins" over a later write from node B.
- Certificates, JWT `exp`/`nbf`, and TOTP codes need clocks within tolerance (allow small leeway, e.g., 30–60 s).
- Monitor clock offset (`chronyc tracking`, node_exporter `node_timex_offset_seconds`) and alert on drift.

---

## 7. Lamport Clocks {#lamport}

Leslie Lamport, *"Time, Clocks, and the Ordering of Events in a Distributed System"* (1978), one of the most important papers in the field.

**Happens-before (→)**: a → b if
1. a and b are on the same process and a comes before b, or
2. a is sending a message and b is receiving that message, or
3. transitively: a → c and c → b.

If neither a → b nor b → a, the events are **concurrent** (a ∥ b).

**Lamport clock algorithm**: each process keeps a counter `L`.
- Before each local event: `L = L + 1`.
- When sending: attach `L`.
- When receiving a message with timestamp `t`: `L = max(L, t) + 1`.

Property: if a → b then L(a) < L(b). **The converse doesn't hold**: L(a) < L(b) doesn't imply a → b (they might be concurrent). Lamport clocks can't detect concurrency.

A **total order** is obtained by breaking ties with the node ID: `(L, node_id)`. It's consistent with causality and useful for things like total-order mutual exclusion. But it's an arbitrary order for concurrent events, and it's not real-time order.

---

## 8. Vector Clocks and Version Vectors {#vector}

A **vector clock** is a vector of counters, one per process: `V = [A:2, B:1, C:0]`.
- Local event at process i: `V[i] += 1`.
- Send: attach V.
- Receive V': `V[j] = max(V[j], V'[j])` for all j, then `V[i] += 1`.

Comparison:
- `V1 ≤ V2` if every component V1[k] ≤ V2[k]. Then V1 happened before (or equals) V2.
- If neither `V1 ≤ V2` nor `V2 ≤ V1`, they're **concurrent**, i.e., **a conflict**.

So vector clocks **characterize causality exactly**: they detect concurrent updates.

**Version vectors** (the replica-level variant) are used in Dynamo, Riak, and Voldemort: each replica stores the version vector with the value. Concurrent versions are kept as **siblings**, and the client or application merges them (Amazon's shopping cart merged siblings by union, which famously caused deleted items to reappear). **Dotted version vectors** fix sibling explosion issues in Riak.

Cost: the size grows with the number of writers (actors). Systems prune or bound it.

Cassandra deliberately chose **LWW with timestamps** instead (simpler, no siblings, but data loss under concurrent writes).

---

## 9. Hybrid Logical Clocks and TrueTime {#hlc}

**Hybrid Logical Clocks (HLC)** (Kulkarni et al., 2014): a timestamp = (physical time component, logical counter). It stays close to wall-clock time (useful for humans and TTLs) while preserving the Lamport happens-before property, and it fits in 64 bits. Used by **CockroachDB**, **YugabyteDB**, MongoDB (cluster time), and others for MVCC timestamps and causal consistency.

**Google Spanner's TrueTime**: GPS receivers + atomic clocks in each datacenter give a time API returning an **interval** `[earliest, latest]` with a bounded uncertainty ε (typically < 7 ms, often ~1–4 ms).
- **Commit wait**: after choosing commit timestamp `s`, Spanner waits until `TT.now().earliest > s` before making the commit visible. This guarantees that if transaction T2 starts after T1 commits (in real time), T2's timestamp > T1's. The result is **external consistency** (linearizability for transactions) across the globe.
- The cost is a few ms of latency per commit, bounded by ε. That's why investing in precise clocks pays off.
- AWS offers microsecond-accurate time (ClockBound library exposes error bounds, similar idea), which Aurora DSQL uses.

CockroachDB without atomic clocks uses HLC + a max clock offset setting (500 ms default) and **uncertainty intervals**: reads that encounter values with timestamps within the uncertainty window restart at a higher timestamp. Nodes whose clocks drift beyond the max offset shut themselves down to protect correctness.

---

## 10. Ordering Guarantees {#ordering}

| Guarantee | Meaning | Mechanism |
|---|---|---|
| **FIFO (per sender)** | Messages from one sender delivered in send order | Sequence numbers per sender (TCP, Kafka per partition with idempotent producer) |
| **Causal** | If m1 → m2 causally, everyone delivers m1 before m2 | Vector clocks, causal broadcast |
| **Total order** | All nodes deliver all messages in the **same** order | Consensus / atomic broadcast (a single leader sequencing, Raft log, Kafka single partition, ZooKeeper's Zab) |
| Total + causal | | |

**Total order broadcast ≡ consensus** (equivalent problems): if you can agree on the order of messages, you can implement consensus and vice versa. This is why replicated state machines (every node applies the same log in the same order: Raft, ZooKeeper, etcd) are the foundation of strongly consistent systems.

---

## 11. Impossibility Results {#impossibility}

- **Two Generals' Problem**: two parties communicating over an unreliable channel can never be *certain* they agree (every ack might be lost, requiring an ack of the ack…). Practical consequence: **exactly-once delivery over unreliable networks is impossible**, so use idempotency instead.
- **FLP impossibility** (Fischer, Lynch, Paterson, 1985): in a fully **asynchronous** system where even **one** process may crash, no deterministic consensus algorithm can guarantee termination. Practical consequence: real consensus protocols guarantee **safety always** and **liveness only under partial synchrony** (timeouts, randomization). Raft can theoretically livelock in elections, and randomized timeouts make that improbable.
- **CAP theorem** (Brewer, proven by Gilbert & Lynch 2002): with a network **P**artition, a system must choose between **C**onsistency (linearizability) and **A**vailability (every non-failed node responds). See [`scaling-db/cap.md`](../scaling-db/cap.md). **PACELC** extends it: **E**lse (no partition), trade **L**atency vs **C**onsistency, which matters more day to day.
- **Byzantine generals**: tolerating f Byzantine nodes requires n ≥ 3f + 1 nodes.
- Crash-fault-tolerant consensus needs **n ≥ 2f + 1** (a majority quorum) to tolerate f failures: 3 nodes tolerate 1, and 5 tolerate 2.

---

## 12. Interview Questions {#qa}

1. Name the fallacies of distributed computing, and give a production example of one biting you.
2. A request to another service times out. What could have happened? What are the implications for retries?
3. Why should you use a monotonic clock for measuring timeouts?
4. Why is last-write-wins by timestamp dangerous?
5. Explain Lamport clocks. What can't they tell you that vector clocks can?
6. How does Spanner achieve external consistency using TrueTime?
7. What does the FLP result mean for practical systems like Raft?
8. Why do consensus clusters have an odd number of nodes?
9. What's a gray failure, and how would you detect one?
10. How does the SWIM protocol detect failures scalably?
