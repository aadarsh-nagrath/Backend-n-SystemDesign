# Consensus, Leader Election, and Coordination (Raft, Paxos, ZooKeeper, etcd, Distributed Locks)

## Table of Contents
1. [The Consensus Problem](#problem)
2. [Replicated State Machines](#rsm)
3. [Raft in Depth](#raft)
4. [Paxos and Multi-Paxos](#paxos)
5. [Other Protocols: Zab, Viewstamped Replication, EPaxos, BFT](#others)
6. [Coordination Services: ZooKeeper, etcd, Consul](#coord)
7. [Leader Election Patterns](#election)
8. [Distributed Locks and Fencing Tokens](#locks)
9. [The Redlock Debate](#redlock)
10. [Two-Phase and Three-Phase Commit](#2pc)
11. [Where Consensus Shows Up in Real Systems](#where)
12. [Interview Questions](#qa)

---

## 1. The Consensus Problem {#problem}

A set of nodes must **agree on a single value** (or a sequence of values) despite failures.

Properties:
- **Agreement**: all correct nodes decide the same value.
- **Validity/integrity**: the decided value was proposed by some node, and a node decides at most once.
- **Termination**: every correct node eventually decides (liveness; not guaranteed under full asynchrony, per FLP).

Used for: leader election, atomic commit of replicated logs, membership changes, distributed locks/leases, configuration storage, unique sequence assignment, and linearizable key-value stores.

Crash-fault tolerance needs a **majority quorum**: n = 2f + 1 nodes tolerate f failures. Any two majorities intersect in at least one node, which prevents two conflicting decisions.

| Cluster size | Quorum | Failures tolerated |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 (no gain over 3!) |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

More nodes mean more fault tolerance but **slower writes** (more acks needed) and more coordination traffic. 3 or 5 is typical.

---

## 2. Replicated State Machines {#rsm}

If every node starts in the same state and applies the **same deterministic commands in the same order**, all nodes end in the same state. Consensus is used to agree on the **log** of commands.

```
Client → Leader: "SET x=5"
Leader appends to its log, replicates to followers.
Once a majority has stored the entry → committed → leader applies to state machine → responds to client.
Followers apply committed entries in order.
```
This is how etcd, ZooKeeper, Consul, CockroachDB ranges, TiKV regions, Kafka's KRaft metadata, MongoDB replica sets (Raft-like), RabbitMQ quorum queues, and Spanner Paxos groups work.

---

## 3. Raft {#raft}

Raft (Ongaro & Ousterhout, 2014, *"In Search of an Understandable Consensus Algorithm"*) was designed for understandability. It decomposes consensus into **leader election**, **log replication**, and **safety**.

### Roles and terms
- Each node is a **follower**, **candidate**, or **leader**.
- Time is divided into **terms** (monotonically increasing integers). Each term has at most one leader. Terms act as a logical clock: a node seeing a higher term steps down to follower and updates its term. Messages with stale terms are rejected.

### Leader election
1. Followers expect periodic **heartbeats** (empty AppendEntries) from the leader.
2. If a follower hears nothing for its **election timeout** (randomized, e.g., 150–300 ms), it becomes a **candidate**: increments its term, votes for itself, and sends **RequestVote** to all.
3. Nodes grant **one vote per term**, first-come-first-served, **only if the candidate's log is at least as up-to-date** as theirs (higher last log term, or the same term and a longer log). This is the **election restriction**, and it ensures the leader has all committed entries.
4. A candidate with a majority of votes becomes leader and sends heartbeats immediately.
5. Split vote (no majority): timeout, then a new term with re-randomized timeouts make repeated splits unlikely.

### Log replication
- The leader receives client commands and appends `(term, index, command)` to its log.
- **AppendEntries** RPC to followers includes `prevLogIndex` and `prevLogTerm`. A follower rejects it if its log doesn't contain a matching entry at prevLogIndex (the **log matching property**: if two logs have an entry with the same index and term, all preceding entries are identical).
- On rejection, the leader decrements `nextIndex` for that follower and retries, eventually finding the point of agreement and overwriting the follower's conflicting suffix.
- An entry is **committed** once stored on a majority **and** it's from the leader's current term (entries from older terms are committed indirectly). Committed entries are durable and will be in every future leader's log (**leader completeness**).
- The leader tracks `commitIndex` and tells followers, which then apply entries up to it.

### Safety details
- **Never commit entries from previous terms by counting replicas** (Figure 8 in the paper): a leader commits a new entry from its own term, which implicitly commits earlier ones. New leaders often append a no-op entry at the start of their term for this reason.
- **Linearizable reads**: a leader might be deposed without knowing it (partitioned). Options:
  - Commit a read through the log (slow).
  - **ReadIndex**: the leader confirms it's still leader via a heartbeat round with a majority, then serves the read after applying up to the commit index.
  - **Lease reads**: the leader assumes leadership for a lease period shorter than the election timeout (relies on bounded clock drift).
  - Follower reads with ReadIndex from the leader.
- **Pre-vote** (optional extension): a node checks whether it could win before incrementing its term, so partitioned nodes rejoining don't disrupt a stable leader with higher terms.
- **Leadership transfer** for graceful maintenance.

### Membership changes
Adding or removing nodes safely: **joint consensus** (old+new configurations must both agree during the transition) or **single-server changes** (one node at a time). Learner/non-voting members catch up before becoming voters.

### Log compaction
Logs grow forever, so take periodic **snapshots** of the state machine and discard the log prefix. **InstallSnapshot** RPC lets lagging followers catch up.

### Performance notes
- Writes need one round trip to a majority (plus fsync on each).
- Batching and pipelining of AppendEntries are essential for throughput.
- Multi-Raft: systems like CockroachDB and TiKV run **thousands of Raft groups** (one per data range), with heartbeat coalescing.

Visualize it: **thesecretlivesofdata.com/raft** and raft.github.io (interactive).

---

## 4. Paxos {#paxos}

Leslie Lamport's Paxos (1989/1998 *"The Part-Time Parliament"*, 2001 *"Paxos Made Simple"*). It's the foundational consensus protocol, notoriously hard to understand and to implement fully.

**Single-decree (basic) Paxos** decides one value. Roles: proposers, acceptors, learners.

**Phase 1 (Prepare/Promise)**:
- A proposer picks a unique proposal number `n` and sends `Prepare(n)` to a majority of acceptors.
- An acceptor that hasn't promised a higher number replies `Promise(n, acceptedN, acceptedValue)` and promises to ignore proposals < n.

**Phase 2 (Accept/Accepted)**:
- If the proposer gets promises from a majority, it sends `Accept(n, v)`, where **v = the value of the highest-numbered accepted proposal among the promises**, or its own value if none were accepted.
- Acceptors accept unless they promised a higher n. Once a majority accepts `(n, v)`, v is **chosen**. Learners are notified.

The rule "adopt the highest accepted value" is what guarantees safety: once a value could have been chosen, all later proposals carry it.

**Multi-Paxos**: to decide a sequence of values (a log), elect a **stable leader** (distinguished proposer) that runs Phase 1 once for all future slots, then only Phase 2 per entry. That's one round trip per value, the same as Raft. Raft is essentially a specific, well-specified Multi-Paxos variant with a strong leader and log continuity constraints.

Real implementations: Google Chubby, Spanner (Paxos per split), Megastore, Azure Storage, Apache Cassandra LWT (Paxos per partition), Ceph monitors (Paxos), WANdisco.

---

## 5. Other Protocols {#others}

- **Zab** (ZooKeeper Atomic Broadcast): leader-based total order broadcast with recovery phases. Similar spirit to Raft.
- **Viewstamped Replication** (Oki & Liskov, 1988; revisited 2012): predates Paxos publication, with a design similar to Raft.
- **EPaxos** (Egalitarian Paxos): leaderless, so commands that don't conflict commit in one round trip from any replica. Good for WAN, but complex.
- **Flexible Paxos**: phase-1 and phase-2 quorums only need to intersect each other (not each be majorities), enabling smaller write quorums.
- **CASPaxos**: Paxos for a single register with compare-and-set semantics.
- **Accord** (Apache Cassandra general transactions): leaderless, timestamp-based consensus for multi-key transactions.
- **Byzantine fault tolerance**: PBFT (1999), Tendermint/CometBFT, HotStuff (used by Diem/Aptos), needing 3f+1 nodes.
- **Chain replication / CRAQ**: nodes arranged in a chain (head handles writes, tail handles reads) with high throughput. A separate configuration service (consensus) manages membership.

---

## 6. Coordination Services {#coord}

Rather than implementing consensus in every app, use a **coordination service** built on it:

### ZooKeeper (Apache, Zab)
- Hierarchical namespace of **znodes** (like a filesystem), each holding small data (≤ 1 MB, typically KB).
- **Ephemeral znodes**: deleted automatically when the creating session ends (client crash, session timeout). Used for liveness and membership.
- **Sequential znodes**: get a monotonically increasing suffix (`/locks/lock-0000000042`), used for locks, queues, and leader election.
- **Watches**: one-time notifications on change.
- Guarantees: linearizable writes, FIFO client order. Reads may be stale by default (served by followers); use `sync()` before a read for freshness.
- Recipes (Apache Curator): leader latch/election, distributed locks (InterProcessMutex), barriers, counters, service discovery.
- Users: Kafka (pre-KRaft), HBase, Hadoop HA, Solr, ClickHouse (now ClickHouse Keeper, a Raft-based compatible reimplementation).

### etcd (CNCF, Raft)
- A flat key-value store with **MVCC revisions** (every change gets a global revision number), **watches** on key ranges (streamed, resumable from a revision), **leases** (TTL; keys attached to a lease vanish when it expires, like ephemeral nodes), **transactions** (compare-and-swap: `If(version(key) == 0) Then(Put) Else(Get)`), and linearizable reads by default.
- It's **Kubernetes' brain**: all cluster state lives in etcd. It's sensitive to disk fsync latency (use SSDs) and has a recommended DB size limit (2 GB default quota, 8 GB suggested max).
- Concurrency library: `concurrency.NewElection`, `concurrency.NewMutex`.

### Consul (HashiCorp)
- Service discovery + health checking + KV + service mesh. Raft among servers, gossip (Serf/SWIM) among all agents.
- Sessions + KV for locks and leader election.

### Others
Kubernetes **Lease** objects (coordination.k8s.io; used for controller leader election via client-go's leaderelection package, backed by etcd), Google Chubby (lock service), DynamoDB conditional writes (lock tables), Redis (with caveats, see below).

---

## 7. Leader Election Patterns {#election}

Why have a leader? To serialize decisions (a single writer), run singleton tasks (cron, schedulers, partition assignment), and avoid conflicting actions.

Patterns:
1. **Consensus-based** (etcd/ZooKeeper/Consul/K8s Lease): the candidate acquires a key with a lease/TTL, renews it periodically, and others watch the key and campaign when it disappears.
2. **Database-based**: a row lock or advisory lock in the shared DB (`pg_try_advisory_lock`), or a "leader" table with `UPDATE … SET leader = me, expires_at = now() + 15s WHERE expires_at < now() OR leader = me` (conditional update = compare-and-swap).
3. **Bully algorithm / ring election** (textbook): the highest-ID live node wins. These aren't safe under partitions without quorums.

**The core danger: two leaders at once.** A leader may believe it still holds leadership after it has expired (GC pause, network partition, clock drift). Mitigations:
- Leaders must stop acting **before** their lease expires (use a safety margin with a monotonic clock).
- **Fencing tokens** on every action (next section).
- Make actions idempotent and checkable.

---

## 8. Distributed Locks and Fencing Tokens {#locks}

Two flavors of locks:
- **For efficiency**: avoid doing the same work twice (e.g., two workers regenerating the same cache entry). An occasional double-execution is harmless. A simple Redis `SET key value NX PX 30000` lock is fine.
- **For correctness**: two holders would corrupt data (e.g., two writers to the same file or account). This requires a consensus-backed lock **and fencing**.

**The pause problem** (Martin Kleppmann, *"How to do distributed locking"*):
```
Client 1 acquires lock (lease 30 s) → GC pause 40 s → lease expires
Client 2 acquires lock → writes to storage
Client 1 wakes up, still believes it holds the lock → writes to storage → CORRUPTION
```
**Fencing tokens**: the lock service returns a **monotonically increasing token** with each grant (etcd revision, ZooKeeper zxid or sequential znode number, DB sequence). Every write to the protected resource includes the token, and the **resource rejects tokens lower than the highest it has seen**:
```
Client 1: lock → token 33 → (pause)
Client 2: lock → token 34 → write(token=34) → storage records max_token = 34
Client 1: write(token=33) → storage rejects (33 < 34)
```
This requires the storage to participate (a conditional write `WHERE token < :t`, or an object storage precondition). If the resource can't check tokens, the lock alone can't guarantee safety.

Lock implementation checklist:
- Acquire with a TTL/lease (so crashes don't leave locks held forever).
- Unique owner ID per holder. Release only if you still own it (compare-and-delete, a Lua script in Redis: `if redis.call("get",KEYS[1]) == ARGV[1] then return redis.call("del",KEYS[1]) end`).
- Renew (heartbeat) long holds. Stop work if renewal fails.
- Fencing tokens for correctness.
- Prefer avoiding locks via design: single-writer partitioning (route all operations for an entity to one consumer/partition), optimistic concurrency (version checks), or DB transactions and constraints.

---

## 9. The Redlock Debate {#redlock}

**Redlock** (proposed by Redis's creator, antirez): acquire the same lock on N independent Redis masters (e.g., 5) within a time window, succeeding if a majority is acquired and the elapsed time is less than the validity.

**Kleppmann's critique** (2016):
1. No fencing tokens: Redlock doesn't generate monotonic tokens, so the pause problem remains.
2. It relies on **timing assumptions** (bounded clock drift, bounded pauses, bounded network delays). Clock jumps (NTP), GC pauses, and delays can let two clients both believe they hold the lock.
3. So it's too heavy for efficiency locks (a single Redis is enough) and not safe enough for correctness locks (use consensus + fencing).

**Antirez's rebuttal** argued that the timing assumptions are reasonable in practice and suggested using monotonic clocks. The community consensus is roughly: **for correctness, use a CP coordination service (etcd/ZooKeeper) with fencing tokens. For efficiency, a single-instance Redis lock is fine.**

---

## 10. Two-Phase and Three-Phase Commit {#2pc}

**2PC** gives atomic commitment across multiple resources (databases, partitions):
1. **Prepare**: the coordinator asks all participants "can you commit?" Each participant durably writes the transaction (locks held) and votes yes or no.
2. **Commit/Abort**: if all voted yes, the coordinator writes its commit decision durably, then tells everyone to commit. Otherwise it aborts.

Problems:
- **Blocking**: if the coordinator crashes after participants voted yes, they're **in doubt**. They can't commit or abort unilaterally and hold locks until the coordinator recovers. Manual intervention ("heuristic decisions") risks inconsistency.
- Latency: two round trips + multiple fsyncs, and locks held across them.
- Coordinator availability is critical.

**3PC** adds a pre-commit phase to avoid blocking under crash failures with bounded delays, but it isn't safe under network partitions. It's rarely used.

**Modern approach**: make the coordinator itself fault-tolerant by **running 2PC over consensus groups** (Spanner, CockroachDB: each participant is a Raft/Paxos group, so it never "disappears", and the transaction record lives in a replicated range). Or avoid cross-service atomic commit altogether with **sagas** (see [`messaging/04-event-driven-architecture-patterns.md`](../messaging/04-event-driven-architecture-patterns.md)).

XA in practice: Java JTA/XA with multiple DBs or message brokers, PG `PREPARE TRANSACTION`, MySQL XA. Use it sparingly.

---

## 11. Where Consensus Shows Up {#where}

| System | Consensus use |
|---|---|
| Kubernetes | etcd (Raft) stores all state; leader election for controllers via Leases |
| Kafka | KRaft controllers (Raft) for metadata; partition leadership from the controller |
| CockroachDB / TiDB(TiKV) / YugabyteDB | Raft per range/region; 2PC across ranges with replicated txn records |
| Spanner | Paxos per split + TrueTime + 2PC |
| MongoDB | Raft-based replica set election protocol (pv1) |
| Cassandra | Paxos for lightweight transactions (LWT) |
| Elasticsearch | Master election (custom, Raft-like in 7.x+) |
| Consul / Nomad / Vault (integrated storage) | Raft |
| RabbitMQ quorum queues / Khepri | Raft (Ra library) |
| Redis | Sentinel (quorum-based failover, not full consensus), Redis Cluster (gossip + epochs); Raft-based variants exist (RedisRaft, experimental) |
| ClickHouse Keeper | Raft (ZooKeeper replacement) |

---

## 12. Interview Questions {#qa}

1. Why does consensus require a majority? Why are clusters sized 3 or 5 rather than 4?
2. Walk through Raft leader election. Why are election timeouts randomized?
3. When is a Raft log entry committed? Why can't a new leader commit old-term entries by counting replicas?
4. How do you serve linearizable reads in Raft without writing to the log?
5. Explain Paxos's two phases. How does Multi-Paxos relate to Raft?
6. What is a fencing token and why is it needed for distributed locks?
7. What's the Redlock controversy about?
8. Why is 2PC called a blocking protocol? How do Spanner/CockroachDB mitigate it?
9. How would you implement leader election for a singleton job across 10 pods?
10. What happens to an etcd/ZooKeeper cluster that loses its majority?
