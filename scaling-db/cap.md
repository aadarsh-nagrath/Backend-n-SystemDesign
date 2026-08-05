# CAP Theorem

A fundamental result about distributed data stores: when the network breaks, you can't have perfectly consistent data and a system that's always available at the same time. It's the reason NoSQL databases split into families (MongoDB vs Cassandra, etc.) and why your bank app sometimes refuses a transfer instead of showing you a possibly-wrong balance.

## TL;DR
- **Consistency (C)**: every read gets the latest write, or an error.
- **Availability (A)**: every request gets a response, even if it's not the latest data.
- **Partition Tolerance (P)**: the system keeps working despite dropped/delayed messages between nodes.
- During a network partition, you must pick **C or A** — you can't have both. P itself isn't optional in a real distributed system, because networks fail.
- "Cheap, Fast, Good — pick two" for distributed systems: "Consistent, Available, Partition-tolerant — pick two," except P is forced on you, so really you're picking C or A when a partition happens.
- Proposed by Eric Brewer (2000), formally proven by Seth Gilbert and Nancy Lynch (2002).

## Origins

Eric Brewer, then a UC Berkeley professor and co-founder of Inktomi, introduced CAP as a conjecture in his 2000 PODC keynote. Before this, databases were mostly single-machine (Oracle, early SQL) — scaling meant buying bigger hardware. As the web grew, horizontal scaling (many cheap machines) became necessary, and that's what exposed the trade-off: once data lives on multiple nodes connected by an unreliable network, you can't dodge it.

In 2002, Gilbert and Lynch formalized and proved the conjecture in *"Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"* (ACM SIGACT News), turning it into a real theorem.

Brewer revisited it in 2012, clarifying it's not a strict binary — real systems tune the trade-off (e.g., "mostly consistent" with "high availability") rather than picking an absolute extreme. This nuance is what later work like PACELC tries to capture. CAP's popularization also fed directly into the NoSQL movement's pitch against traditional SQL for scalable, distributed workloads.

## The three properties

### Consistency (C)
Every node sees the same data at the same time. After a write succeeds, every subsequent read returns that value (or newer), or an error if it can't guarantee that.

**Analogy**: a shared Google Doc — when you edit it, everyone else should see the change immediately, never an old version.

Consistency isn't one flavor:
- **Strong consistency** — immediate sync, like ACID transactions in SQL.
- **Eventual consistency** — data may be briefly stale but converges eventually (typical of AP systems).
- **Causal consistency** — preserves cause-effect ordering (if A happens before B, reads reflect that order even if not perfectly in sync otherwise).

Fails during partitions when some nodes don't get the update in time, producing stale reads. Trade-off: accurate data builds trust (no double-spending in finance) but waiting for sync costs availability/latency.

### Availability (A)
Every non-failed node responds to every request — even if the answer isn't the latest data.

**Analogy**: a 24/7 convenience store — always open, even if the inventory list is stale (you might hear "out of stock" for something actually on the shelf).

Availability doesn't mean *fast* — just that you get *some* response. "High availability" is often expressed as uptime percentage (99.99% = "four nines"). Fails when the system instead chooses to wait for consistency during a partition, causing requests to hang or error. Trade-off: seamless UX vs. risk of stale/incorrect data.

### Partition Tolerance (P)
The system keeps functioning despite arbitrary message loss or delay between nodes.

**Analogy**: a remote team — if email goes down, everyone keeps working individually and syncs later.

Mandatory in practice because real networks are unreliable — latency spikes, packet loss, AWS outages, even undersea cable cuts. Partitions come in flavors:
- **Partial** — some nodes can talk, others can't.
- **Total** — complete isolation.
- **Byzantine** — nodes behave maliciously/arbitrarily (a harder problem than classic CAP, but related).

| Property | What it means | Analogy | Trade-off |
|---|---|---|---|
| **Consistency** | All reads get the latest write, or an error | Shared doc: instant updates everywhere | Slows down / blocks during failures |
| **Availability** | Every request gets a response | 24/7 store: always open | May serve stale data |
| **Partition tolerance** | Works despite network splits | Remote team: keeps working offline | Forces the C-vs-A choice; unavoidable in distributed systems |

## Why you can't have all three

With no partition, all three are achievable simultaneously — CAP only bites *during* a partition. When nodes can't talk to each other, you must choose:
- **CP**: stay consistent, sacrifice availability (refuse/delay requests until you can guarantee correctness).
- **AP**: stay available, sacrifice consistency (answer anyway, possibly with stale data, and reconcile later).
- **CA**: theoretically "consistent and available," but only achievable by *not* tolerating partitions — meaning not really distributed (a single node, or a system that gives up the moment the network splits). Not a realistic choice for a genuinely distributed system, since partitions aren't optional — they happen whether you're ready or not.

Brewer's 2012 clarification: this isn't a strict binary switch. Systems can be "mostly" consistent or "mostly" available, tuning the trade-off along a spectrum depending on the operation and the moment.

## The proof, walked through

Take two nodes, G1 and G2, both tracking a variable `v` (initially `v0`), able to communicate over the network — but a partition can drop messages between them.

1. **Assume a CAP system exists.** For contradiction, assume a system that is simultaneously Consistent, Available, and Partition-tolerant.
2. **Introduce a partition.** The network between G1 and G2 breaks — no messages get through.
3. **Write to G1.** A client writes `v1` to G1. Because the system claims to be Available, G1 must acknowledge the write. But it can't propagate that write to G2 — the partition blocks it.
4. **Read from G2.** A client reads from G2. Because the system claims to be Available, G2 must respond — but the only value it has is the stale `v0`, since it never got the update.
5. **Contradiction.** The read after the write returned stale data, violating Consistency. So no system can be simultaneously C, A, and P during a partition — the assumption in step 1 fails.

This is an impossibility proof in the same family as Arrow's impossibility theorem in economics — it doesn't say "hard to build," it says "cannot exist." Edge case: if no writes happen during the partition, nothing looks broken — but the theorem is about the worst case, not the lucky case.

## CAP vs. ACID

Easy to confuse because both use the word "consistency," but they mean different things:

| Aspect | CAP consistency | ACID consistency |
|---|---|---|
| Focus | Same (latest) data visible across all nodes | Database moves from one valid state to another |
| Scope | Distributed system, across nodes | Single transaction, typically one machine |
| Example | Bank balance is identical on every replica | A transfer never leaves a negative balance mid-transaction |

They overlap on "data integrity matters" but operate at different layers — CAP is about distribution, ACID is about transactional correctness. NoSQL databases that relax CAP consistency often lean on **BASE** instead (Basically Available, Soft state, Eventual consistency) — explicitly prioritizing A and P over strict C.

## CP, AP, and CA databases

Databases get classified by which two of the three they prioritize:

- **CP** — Consistency + Partition tolerance, sacrifices Availability. During a partition, parts of the system shut down or block rather than risk serving stale data (e.g., waiting for a leader election).
- **AP** — Availability + Partition tolerance, sacrifices Consistency. Keeps answering requests during a partition, possibly with stale data, and reconciles afterward (eventual consistency).
- **CA** — Consistency + Availability, no real Partition tolerance. Works for non-distributed or tightly-coupled replicated systems, but isn't a genuine option once you're truly distributed, because partitions aren't something you get to opt out of.

| Database | CAP type | Key features | Typical use case |
|---|---|---|---|
| MongoDB | CP | Document store, replica sets, single primary for writes | Content management, apps needing accurate reads |
| HBase | CP | Strong consistency, big-data column store | Large-scale analytical workloads needing correctness |
| Redis (cluster mode) | CP | In-memory, prioritizes consistency | Session storage, caching |
| PostgreSQL (replicated w/ Patroni etc.) | CA-leaning | Traditional relational, add-on tooling for P | Systems that need strong consistency and can tolerate some downtime |
| Cassandra | AP | Masterless, peer-to-peer, eventual consistency via repair | High-traffic sites (e.g., Netflix-style recommendation data) |
| DynamoDB | AP (tunable) | Key-value, serverless, tunable strong/eventual reads | E-commerce shopping carts |
| Cosmos DB | AP (tunable) | Multi-model, dial-able consistency levels | Global apps needing flexible consistency |
| CouchDB | AP | JSON documents, multi-master sync | Mobile/offline-first sync |

**Tunable systems** (DynamoDB, Cosmos DB, Cassandra) let you dial consistency vs. availability per-operation rather than locking the whole database into one mode — e.g., a strongly-consistent read for a critical operation, eventually-consistent for everything else.

## 🟢 Beginner: picking the right side

Rule of thumb: does *wrong data* hurt more than *no data*?

- If yes (money, medical dosages, seat assignments) → lean **CP**.
- If no (social feeds, product recommendations, "likes" counts) → lean **AP**.

Simple mental model: a group chat app with messages stored on servers worldwide. If one server loses connection (a partition), you either (a) wait for it to resync before showing new messages — prioritizing C, sacrificing A — or (b) show possibly-outdated messages but keep the app responsive — prioritizing A, sacrificing C. You cannot do both at once without risking correctness or uptime.

## 🟡 Intermediate: it's not one global choice

Brewer's 2012 point in practice: real systems don't pick CP or AP once for the whole database — they pick per operation, per data type, or per criticality:

- A single e-commerce platform might use CP for payment processing and inventory-critical writes, but AP for product recommendations and browsing history.
- "Read-your-writes" consistency is a common partial fix in AP systems: a user always sees their own writes immediately (routed to the node they wrote to, or session-pinned), even if other users see it more slowly.
- **Quorums** (e.g., in Cassandra) require a majority of replicas to agree before confirming a read/write, tuning the C/A dial without going fully strong or fully eventual.
- **Polyglot persistence** — using multiple database types in one system, each chosen for the CAP profile that operation needs (CP store for orders, AP store for logs/analytics) — is standard in real architectures, not a compromise.

## 🔴 Advanced: PACELC and beyond

**PACELC** (Daniel Abadi, 2010) extends CAP to cover the case CAP doesn't address — normal operation with no partition. It says: *if there's a Partition (P), trade off Availability vs. Consistency (A vs C) — Else (E), trade off Latency vs. Consistency (L vs C)*. This matters because CAP is silent about behavior when the network is healthy, but real systems still make a consistency/latency trade-off then too (synchronous replication costs latency even without a partition).

Classified systems get a 4-letter code:
- DynamoDB: **PA/EL** — available during a partition, low-latency otherwise.
- MongoDB: **PC/EC** — consistent during a partition (at the cost of availability), consistent otherwise (at the cost of latency).

**Other related concepts:**
- **Harvest and Yield** (Brewer's own follow-up framing) — instead of a hard availability cutoff, think of *yield* (fraction of requests answered) and *harvest* (fraction of the data reflected in the answer); systems can trade completeness for speed rather than an all-or-nothing switch.
- **CRDTs** (Conflict-free Replicated Data Types) — data structures designed to merge concurrent updates from different nodes without conflicts, letting AP systems get "eventual consistency" with well-defined, automatic resolution instead of ad hoc reconciliation.
- **Byzantine fault tolerance** — a harder problem than partitions: nodes that send incorrect or malicious data, not just silence. Relevant to blockchain-style systems, outside classic CAP's scope.

## Real-world examples

| Scenario | CAP choice | Why | Notes |
|---|---|---|---|
| **Banking app** | CP | Wrong balance risks overdrafts/double-spending; the system would rather say "temporarily unavailable" than show incorrect data | Most banks run CP systems (SQL clusters, or CP-tuned NoSQL like MongoDB) |
| **E-commerce cart** (e.g., Amazon) | AP | Availability drives sales; a slightly stale stock count is fixable later with a refund/backorder, but a down cart isn't | DynamoDB powers this at Amazon; tunable consistency for critical paths |
| **ATM withdrawal** | CP (typically) | Two ATMs in different cities must not both let you withdraw against the same balance; if the link to the bank breaks, better to deny the withdrawal than allow an overdraw | During the 2012 RBS (UK) outage, a partition caused days of unavailability specifically to preserve consistency and avoid double-withdrawals. Some modern ATMs relax this for non-critical features (e.g., balance inquiry stays available while withdrawal stays consistency-gated) |
| **Social media feed** (Twitter/X, Facebook) | AP | Users tolerate a slightly stale feed; downtime loses engagement immediately | Facebook's TAO graph store is AP-leaning |
| **Airline reservations** | CP | Double-booking the same seat is a hard failure mode | Systems like Sabre use strong consistency |
| **IoT / smart grid energy management** | CP | Inconsistent power-usage readings across nodes can cause real overloads/blackouts | Accuracy directly affects physical safety |
| **Health records** | CP | Stale data across hospitals could mean a wrong dosage — a genuinely dangerous failure mode | Consistency outweighs uptime here |
| **Messaging apps** (WhatsApp, etc.) | AP | Messages deliver eventually; the app should keep working offline and sync once reconnected | Classic eventual-consistency UX users already expect |

Microservices angle: CAP choice is typically made *per service*, not once for the whole system — e.g., a payment microservice goes CP while a recommendation microservice goes AP, each backed by whichever database type fits.

## Common pitfalls / misconceptions

- **"CAP means always picking two, permanently."** No — the trade-off only actually forces a choice *during* a partition. Outside of that, tuning is possible (see PACELC).
- **"SQL is CA, NoSQL is AP or CP."** No — SQL databases can be distributed too (e.g., Google Spanner is a CP system built on relational semantics). The CAP category is a property of the deployment/architecture, not the query language or data model.
- **"Eventual consistency is just bad/broken."** No — it's a legitimate, deliberate design choice, fine for plenty of real systems (DNS is a canonical example of a system everyone relies on that is only eventually consistent).
- **Ignoring partitions entirely during design.** Assuming "the network is reliable enough" is how outages happen (e.g., large-scale AWS outages have exposed exactly this assumption in production systems). Partitions aren't a rare edge case at scale — they're a certainty over a long enough timeline.
- **Treating CAP as the only axis that matters.** Latency, throughput, and operational complexity matter just as much in practice — PACELC exists precisely because CAP alone under-specifies system behavior.

## Design implications and mitigations

- **Choice depends on domain**: finance and health lean CP; social and most e-commerce lean AP. There's no universally "correct" side.
- **Quorums** — require majority-node agreement before confirming reads/writes, giving a tunable middle ground rather than an all-or-nothing choice.
- **Replication** — copying data across nodes is what makes both availability and durability possible in the first place, but it's also *why* the C/A trade-off exists (more copies = more to keep in sync).
- **Read-your-writes consistency** — a practical partial fix that gives users a strongly-consistent view of their own actions inside an otherwise AP system.
- **Hybrid / polyglot persistence** — use multiple databases with different CAP profiles for different data in the same product, rather than forcing one choice system-wide.
- **Cloud-managed tunability** — AWS, Azure, and others expose configurable consistency levels (e.g., DynamoDB's choice between eventually-consistent and strongly-consistent reads) so the CAP trade-off becomes a per-request setting instead of an architectural commitment.
- **Performance trade-offs**: CP systems tend to be slower under partition/contention (waiting for sync/quorum); AP systems scale and respond faster but need reconciliation logic.
- **Testing partition behavior**: tools like Jepsen exist specifically to simulate network partitions against real databases and verify whether they actually deliver the consistency guarantees they claim. Benchmark tools like YCSB (Yahoo! Cloud Serving Benchmark) help evaluate performance trade-offs across CAP-classified systems.
- **Operational practice**: monitor network health and use circuit breakers so a partition degrades gracefully instead of cascading into a full outage.

## Quick reference

- **C**onsistency: latest write or an error, every read.
- **A**vailability: a response, every request, always.
- **P**artition tolerance: keeps working despite network splits — not optional in real distributed systems.
- Partition happens → pick **CP** or **AP**. **CA** only exists without real partition tolerance.
- **PACELC**: no partition → trade Latency vs. Consistency instead.
- CAP consistency ≠ ACID consistency (distributed data sync vs. transactional validity).
- CP examples: MongoDB, HBase, Redis (cluster), Spanner.
- AP examples: Cassandra, DynamoDB, Cosmos DB, CouchDB.
- Mitigations: quorums, replication, read-your-writes, polyglot persistence, tunable consistency.

## Further reading
- Eric Brewer's original PODC 2000 keynote and his 2012 "CAP Twelve Years Later" retrospective.
- Gilbert & Lynch, *"Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"* (ACM SIGACT News, 2002).
- Daniel Abadi's PACELC paper (2010).
- Jepsen (jepsen.io) for real partition-testing case studies against production databases.
