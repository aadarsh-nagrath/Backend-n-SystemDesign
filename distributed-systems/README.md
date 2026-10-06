# Distributed Systems for Backend Engineers

This folder covers the theory under system design: why distributed systems fail in strange ways, and which algorithms and patterns make them reliable anyway.

| # | Note | Topics |
|---|---|---|
| 01 | [Fundamentals: Failures, Time & Clocks](01-fundamentals-failures-time-and-clocks.md) | Fallacies, system models, partial failures, failure detectors (phi accrual, SWIM), NTP & clock skew, Lamport & vector clocks, HLC, TrueTime, ordering, FLP/Two Generals/CAP |
| 02 | [Consensus, Leader Election & Coordination](02-consensus-leader-election-and-coordination.md) | Majority quorums, replicated state machines, Raft in depth, Paxos/Multi-Paxos, ZooKeeper/etcd/Consul, leader election, distributed locks & fencing tokens, Redlock debate, 2PC/3PC |
| 03 | [Consistency Models & CRDTs](03-consistency-models-and-crdts.md) | Linearizability vs serializability, causal & session guarantees, PACELC, quorum caveats, conflict resolution, CRDTs, OT vs CRDT, local-first, Jepsen |
| 04 | [Core Algorithms & Data Structures](04-core-algorithms-and-data-structures.md) | Consistent hashing (ring, jump, rendezvous, Maglev, bounded load), Bloom/cuckoo, HyperLogLog, Count-Min, t-digest, Merkle trees, gossip, ID generation (Snowflake/UUIDv7), rate limiting, geospatial (geohash/quadtree/S2/H3) |
| 05 | [Resilience Patterns](05-resilience-patterns.md) | Cascading & metastable failures, timeouts & deadlines, retries/backoff/jitter/budgets, circuit breakers, bulkheads, load shedding, backpressure, degradation, hedging, cells & shuffle sharding, static stability, chaos engineering |

Related notes:
- CAP theorem: [`scaling-db/cap.md`](../scaling-db/cap.md) · Sharding: [`scaling-db/sharding.md`](../scaling-db/sharding.md)
- Replication & HA (database view): [`databases/fundamentals/08-replication-ha-backup-recovery.md`](../databases/fundamentals/08-replication-ha-backup-recovery.md)
- Event-driven consistency (outbox, sagas): [`messaging/04-event-driven-architecture-patterns.md`](../messaging/04-event-driven-architecture-patterns.md)
- Code: consistent hashing, rate limiting, LB algorithms in [`system-design/awesome-notes/implementations/`](../system-design/awesome-notes/implementations/)
- Papers list: [`system-design/awesome-notes/README.md`](../system-design/awesome-notes/README.md)

## Essential papers (read in this order)
1. Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System* (1978)
2. *Dynamo: Amazon's Highly Available Key-value Store* (2007)
3. *Bigtable* (2006), *GFS* (2003), *MapReduce* (2004)
4. Ongaro & Ousterhout, *In Search of an Understandable Consensus Algorithm (Raft)* (2014)
5. Lamport, *Paxos Made Simple* (2001)
6. *Spanner: Google's Globally-Distributed Database* (2012)
7. Dean & Barroso, *The Tail at Scale* (2013)
8. Shapiro et al., *Conflict-free Replicated Data Types* (2011)
9. Kreps, *The Log: What every software engineer should know about real-time data's unifying abstraction* (blog, 2013)
10. Bronson et al., *Metastable Failures in Distributed Systems* (2021)
11. *Kafka: a Distributed Messaging System for Log Processing* (2011)
12. *ZooKeeper: Wait-free coordination for Internet-scale systems* (2010)

## Courses and books
- MIT 6.5840 (formerly 6.824) Distributed Systems: lectures + Raft labs in Go (do the labs!)
- Martin Kleppmann's Cambridge *Distributed Systems* lecture series (free on YouTube)
- *Designing Data-Intensive Applications* (Kleppmann), Part II
- *Database Internals* (Petrov), Part II
- *Understanding Distributed Systems* (Roberto Vitillo): practical and concise
- Google SRE Book chapters on load balancing, overload, and cascading failures (free at sre.google)
- jepsen.io analyses; Marc Brooker's blog (brooker.co.za) on retries, backoff, and metastability
