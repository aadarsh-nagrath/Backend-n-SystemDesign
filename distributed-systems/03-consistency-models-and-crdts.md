# Consistency Models, CAP/PACELC in Practice, and CRDTs

> CAP basics: [`scaling-db/cap.md`](../scaling-db/cap.md). Database isolation (a related but *different* concept) is in [`databases/fundamentals/03-transactions-isolation-concurrency.md`](../databases/fundamentals/03-transactions-isolation-concurrency.md).

## Table of Contents
1. [Consistency vs Isolation: Two Different Axes](#axes)
2. [The Consistency Spectrum](#spectrum)
3. [Linearizability](#linearizability)
4. [Sequential Consistency](#sequential)
5. [Causal Consistency](#causal)
6. [Session Guarantees](#session)
7. [Eventual Consistency and Convergence](#eventual)
8. [Strict Serializability and How It Combines Both Axes](#strictser)
9. [CAP and PACELC, Used Correctly](#cap)
10. [Quorums in Depth](#quorums)
11. [Conflict Resolution Strategies](#conflicts)
12. [CRDTs](#crdts)
13. [Local-First Software and Collaborative Editing (OT vs CRDT)](#localfirst)
14. [Testing Consistency: Jepsen](#jepsen)
15. [Interview Questions](#qa)

---

## 1. Consistency vs Isolation {#axes}

- **Isolation** (databases, the I in ACID): how **concurrent transactions** (multi-object operations) interleave. Serializability is the gold standard.
- **Consistency models** (distributed systems): what values **reads of a single object** may return given **replication and real time**. Linearizability is the gold standard.
- They're orthogonal. **Strict serializability** = serializability + linearizability (transactions appear to execute atomically in an order consistent with real time). Spanner and FoundationDB provide it.
- The "C" in ACID (invariants hold) is unrelated to the "C" in CAP (linearizability). The same word means three different things.

---

## 2. The Spectrum {#spectrum}

From strongest to weakest (single-object models):
```
Linearizable (atomic, strong)
   │
Sequential
   │
Causal  ←── strongest model available under partition (with sticky clients)
   │
PRAM / FIFO, session guarantees (read-your-writes, monotonic reads/writes, writes-follow-reads)
   │
Eventual
```
Stronger models are easier to program against but cost more latency and availability. Jepsen's consistency map (jepsen.io/consistency) shows the full hierarchy, including which models are "totally available" (available on every non-failing node during partitions): at most causal consistency (with sticky availability).

---

## 3. Linearizability {#linearizability}

Also called atomic consistency or strong consistency. **Every operation appears to take effect instantaneously at some point between its invocation and its response, and all clients observe that same order, consistent with real time.**

It behaves as if there were a single copy of the data:
- Once a write completes, every subsequent read (by anyone, anywhere) returns that value or a newer one.
- Once any read returns a new value, all later reads return it too (no flip-flopping).

Needed for:
- Leader election and locks (exactly one holder).
- Uniqueness constraints (one username).
- Cross-channel timing dependencies (upload image to storage, then send a message saying "image ready": the reader must see the image).
- Financial balances where reads gate writes.

Implementations: single-leader replication with reads from the leader (and the leader confirming it's still leader), consensus (Raft/Paxos with ReadIndex/lease reads), Spanner/TrueTime. **Not** linearizable: async replicas, Dynamo-style quorums with sloppy quorums or LWW, most caches.

Cost: every operation needs coordination with a quorum or the leader, so latency is ≥ RTT to a majority, and it's unavailable on the minority side of a partition (CAP).

---

## 4. Sequential Consistency {#sequential}

All operations appear in **some total order** consistent with **each client's program order**, but not necessarily real-time order across clients. A client might read a stale value as long as everyone agrees on the global order. Used in CPU memory models (as an ideal) and ZooKeeper-like guarantees (ZooKeeper writes are linearizable, and reads are sequentially consistent per client unless you `sync`).

---

## 5. Causal Consistency {#causal}

Operations that are **causally related** must be seen by everyone in the same order. Concurrent operations may be seen in different orders by different nodes.

Example:
1. Alice posts "I lost my keys" (A).
2. Bob sees A and replies "found them in the kitchen" (B). B causally depends on A.
3. Under causal consistency, nobody sees B without A. Under eventual consistency, Carol could see the reply before the question.

It's implemented with dependency tracking (vector clocks, explicit dependencies, HLC timestamps). MongoDB causal consistency sessions, COPS/Eiger research systems, and AntidoteDB provide it.

Causal consistency is the **strongest model that remains available during network partitions**, which makes it attractive for geo-distributed systems.

---

## 6. Session Guarantees {#session}

Weaker, per-client guarantees (Terry et al., Bayou, 1994). These are very practical for app design:
| Guarantee | Meaning | Violation example | Fix |
|---|---|---|---|
| **Read-your-writes** | A client sees its own writes | Update profile, refresh, see old name (read hit a lagging replica) | Read from the leader after writes, or wait for the replica to reach the client's write position (LSN/GTID token) |
| **Monotonic reads** | Never see older data after seeing newer | Refresh twice: comment appears, then disappears | Sticky replica per session |
| **Monotonic writes** | A client's writes applied in order | Set name "A" then "B", replica applies B then A | Route writes for a client to one leader; sequence numbers |
| **Writes-follow-reads** | A write is ordered after the writes the client has read | Reply visible before original post | Causal tracking |

Many "eventually consistent" systems feel fine to users once these session guarantees are provided.

---

## 7. Eventual Consistency {#eventual}

"If no new updates are made, eventually all replicas converge to the same value." That says nothing about *when*, or about what you read in the meantime. It's weak, but highly available and low-latency.

**Strong eventual consistency (SEC)**: replicas that have received the same set of updates are in the same state, with no conflicts requiring rollback. CRDTs guarantee SEC.

Measure **staleness** in practice: replication lag percentiles, Probabilistically Bounded Staleness (PBS) analysis for quorum systems.

---

## 8. Strict Serializability {#strictser}

| | Single object | Multi-object transactions |
|---|---|---|
| No real-time constraint | Sequential | **Serializable** |
| Real-time order respected | **Linearizable** | **Strict serializable** |

Examples:
- PostgreSQL single node with SERIALIZABLE: strict serializable (one node, so real-time order holds).
- Postgres async replica reads: serializable on the primary, but replica reads are stale, so the overall system isn't linearizable.
- CockroachDB: serializable, plus "no stale reads" for single-key operations (linearizable per key), but not strictly serializable for all transactions (causal reverse anomalies are possible without TrueTime).
- Spanner: strict serializable ("external consistency").
- FoundationDB: strict serializable.

---

## 9. CAP and PACELC, Used Correctly {#cap}

Common misunderstandings:
- "Pick 2 of 3" is misleading. **Partitions aren't optional** in distributed systems. The real choice is: **when a partition happens**, do you refuse some requests (CP) or serve possibly inconsistent data (AP)?
- CAP's C is **linearizability** specifically, and its A is **every non-failing node responds** (a strong definition). Many systems are neither CAP-C nor CAP-A (e.g., a single-leader DB with async replicas: not linearizable for replica reads, not available on the minority side for writes).
- Systems often let you choose **per operation** (Cassandra CL, DynamoDB ConsistentRead, MongoDB read/write concerns).

**PACELC** (Daniel Abadi): if **P**artition → **A** or **C**; **E**lse → **L**atency or **C**onsistency. The "else" branch is what you live with every day: even without failures, strong consistency costs coordination latency.
| System | PACELC |
|---|---|
| DynamoDB (default), Cassandra, Riak | PA/EL |
| MongoDB (majority) | PC/EC (roughly) |
| Spanner, CockroachDB, etcd, ZooKeeper | PC/EC |
| PNUTS (Yahoo) | PC/EL |
| MySQL/PG async replication with replica reads | PA/EL-ish for reads, PC for writes (single leader) |

Practical framing for interviews: "For payments I need linearizable writes (no double-spend), so I accept higher latency and possible unavailability during partitions. For the like counter I choose availability and low latency and accept eventual consistency."

---

## 10. Quorums in Depth {#quorums}

N replicas, write to W, read from R:
- **W + R > N** means read and write sets overlap, so a read sees at least one copy of the latest successful write.
- W = N, R = 1: fast reads, writes fail if any replica is down.
- W = 1, R = N: fast writes, slow and fragile reads.
- W = R = ⌈(N+1)/2⌉: balanced (QUORUM).

**Quorums aren't automatically linearizable**:
1. **Concurrent writes** with LWW timestamps can lose updates.
2. **A write that failed** (succeeded on fewer than W replicas) isn't rolled back on the replicas where it succeeded, so later reads may or may not see it.
3. **Read-repair races**: a reader sees the new value from one replica while another reader concurrently sees the old one (non-monotonic). Linearizable quorum reads require **read repair before returning** (ABD algorithm: write back the latest value to a quorum before responding).
4. **Sloppy quorums + hinted handoff**: during failures, writes go to any N healthy nodes (not the designated replicas), so W + R > N no longer guarantees overlap.
5. Clock skew in LWW.

---

## 11. Conflict Resolution {#conflicts}

When concurrent writes to the same item happen on different replicas (multi-leader, leaderless, offline clients):
| Strategy | How | Trade-off |
|---|---|---|
| **Last-Write-Wins (LWW)** | Highest timestamp wins | Simple; **silently drops** concurrent writes; clock skew issues. OK for caches and immutable or idempotent data |
| **Conflict avoidance** | Route all writes for a key to one home replica/region | Simplest correct option; failover must reassign carefully |
| **Keep siblings + app merge** | Store all concurrent versions (version vectors); app merges on read | Correct but complex; merge logic per data type |
| **Custom merge functions** | Domain-specific (max, union, sum) | |
| **CRDTs** | Data types designed to merge automatically and deterministically | Limited to supported types; metadata overhead |
| **Operational transformation (OT)** | Transform concurrent edits against each other (central server) | Google Docs-style text editing |
| **Human resolution** | Show the conflict to users (git merge, Dropbox "conflicted copy") | |

---

## 12. CRDTs {#crdts}

**Conflict-free Replicated Data Types** (Shapiro et al., 2011): data structures where replicas can be updated independently and concurrently without coordination, and are **guaranteed to converge** when they've seen the same updates.

Two families:
- **State-based (CvRDT)**: replicas send their full state, and merge = **join** in a semilattice (commutative, associative, idempotent). Tolerates duplicate and out-of-order delivery (gossip-friendly).
- **Operation-based (CmRDT)**: replicas send operations, which must be commutative for concurrent ops. Needs reliable causal delivery (each op exactly once, in causal order). Smaller messages.
- **Delta-state CRDTs**: ship only state deltas, which combines the advantages of both.

Classic CRDTs:
| CRDT | Semantics |
|---|---|
| **G-Counter** (grow-only) | Vector of per-replica counts; value = sum; merge = element-wise max |
| **PN-Counter** | Two G-Counters (increments P, decrements N); value = P − N |
| **G-Set** | Add-only set; merge = union |
| **2P-Set** | Add-set + remove-set (tombstones); removed elements can't be re-added |
| **LWW-Register / LWW-Element-Set** | Timestamps decide (deterministic tie-break) |
| **OR-Set (Observed-Remove Set)** | Each add gets a unique tag; remove deletes observed tags. Concurrent add + remove → **add wins**. The most practical set |
| **MV-Register** | Keeps all concurrent values (like Dynamo siblings) |
| **Maps / JSON CRDTs** | Nested CRDTs (Automerge, Yjs) |
| **Sequence CRDTs** (RGA, Logoot, LSEQ, YATA, Fugue) | Collaborative text/lists: each character gets a unique, ordered ID |

G-Counter example (3 replicas):
```
A: [A:3, B:0, C:0]   B: [A:1, B:2, C:0]   C: [A:0, B:0, C:5]
merge(A, B) = [A:3, B:2, C:0]; merge with C = [3, 2, 5] → value 10 (order of merges doesn't matter)
```

Where CRDTs are used:
- **Redis Enterprise Active-Active** (CRDB: counters, sets, strings with LWW, etc.), **Riak** data types, Azure Cosmos DB conflict policies, **AntidoteDB**.
- Collaborative and local-first apps: **Automerge**, **Yjs** (used by many editors: Tiptap, BlockNote, JupyterLab collab), Figma (CRDT-inspired, server-authoritative), Apple Notes (CRDT-based), Linear sync.
- Distributed counters/presence in edge platforms.

Limitations: metadata growth (tombstones, per-replica vectors) needs garbage collection, some invariants can't be expressed without coordination (e.g., "balance never below zero": two replicas can each approve a withdrawal), and semantics can surprise users (add-wins vs remove-wins).

**Invariant confluence** (Bailis et al.): an invariant can be preserved without coordination only if merging valid states always yields a valid state. Uniqueness and non-negativity aren't confluent, so they need coordination. Grow-only properties are confluent.

---

## 13. Local-First and Collaborative Editing {#localfirst}

- **OT (Operational Transformation)**: Google Docs, Etherpad. Edits are operations transformed against concurrent ops, typically through a **central server** that orders operations. It's proven in production, but correct transformation functions are notoriously tricky.
- **CRDTs for text**: peer-to-peer capable, offline-first, server optional (or a relay). Yjs and Automerge have made them practical with compact encodings.
- **Local-first software** (Ink & Switch essay, 2019): data lives on the device first and syncs in the background (fast, offline, user-owned), with CRDT-based sync engines (Automerge, Yjs, ElectricSQL, PowerSync, Replicache/Zero, Triplit, Jazz, LiveStore).
- Server-authoritative sync (Figma, Linear, Replicache): clients apply optimistic mutations, and the server is the source of truth that rebases or replays them.

---

## 14. Testing Consistency: Jepsen {#jepsen}

**Jepsen** (Kyle Kingsbury) tests distributed databases under faults (partitions, clock skew, crashes, pauses) and checks histories against consistency models (Knossos/Elle checkers). It has found violations in nearly every system tested: lost writes, stale reads, and dirty reads in MongoDB, Elasticsearch, Redis, Cassandra, etcd, CockroachDB, PostgreSQL (a 2020 serializability bug), MySQL, TiDB, Kafka, and others. Many were fixed.

Lessons:
- Read the Jepsen report for any datastore you depend on.
- Vendor docs often overstate guarantees, so verify the default settings.
- Your own systems need fault-injection testing too (chaos engineering, deterministic simulation testing like FoundationDB, TigerBeetle, and Antithesis).

---

## 15. Interview Questions {#qa}

1. What's the difference between linearizability and serializability? What's strict serializability?
2. Give an example where read-your-writes is violated and how to fix it.
3. Why is causal consistency attractive for geo-distributed systems?
4. Explain PACELC. Classify DynamoDB, Spanner, and Cassandra.
5. Does W + R > N guarantee linearizability? Why not?
6. Compare LWW, siblings, and CRDTs for conflict resolution.
7. Explain how a G-Counter and an OR-Set work.
8. Why can't a CRDT enforce "account balance ≥ 0"?
9. How do collaborative editors like Google Docs and Figma handle concurrent edits?
10. What does Jepsen test, and what has it taught the industry?
