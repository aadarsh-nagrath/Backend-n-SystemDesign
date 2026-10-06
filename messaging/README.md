# Messaging, Event-Driven Architecture, and Background Processing

How services communicate asynchronously and stay consistent without distributed transactions, and how to run work reliably outside the request path.

| # | Note | Topics |
|---|---|---|
| 01 | [Messaging Fundamentals](01-messaging-fundamentals.md) | Queue vs pub/sub vs log, commands vs events, delivery semantics, idempotent consumers, ordering, acks/visibility, retries & DLQs, backpressure, poison messages, schema evolution, choosing a broker |
| 02 | [Kafka](02-kafka.md) | Partitions/offsets, storage internals, replication/ISR/HW, KRaft, producers (batching, idempotence), consumer groups & rebalancing, EOS transactions, compaction, Connect/Streams/Flink, topic design, tuning, ops |
| 03 | [RabbitMQ, SQS/SNS/EventBridge, NATS, Redis](03-rabbitmq-sqs-and-other-brokers.md) | Exchanges & routing, quorum queues & streams, confirms/acks/prefetch, DLX retries, SQS Standard vs FIFO, visibility timeouts, SNS fan-out, EventBridge, Pub/Sub, Service Bus, NATS JetStream, Redis Streams |
| 04 | [Event-Driven Architecture Patterns](04-event-driven-architecture-patterns.md) | Dual-write problem, transactional outbox, inbox, sagas (choreography/orchestration, compensations, pivot), CQRS, event sourcing, projections, domain vs integration events, UX under eventual consistency |
| 05 | [Background Jobs, Scheduling & Workflows](05-background-jobs-scheduling-and-workflows.md) | Job queue architecture, frameworks per language, robust/idempotent jobs, retries, fairness & rate limits, distributed cron, delayed jobs, Temporal/Step Functions, batch patterns, job observability |

Related:
- Postgres as a queue (`SKIP LOCKED`, LISTEN/NOTIFY): [`databases/postgresql/05-data-types-jsonb-fts-and-features.md`](../databases/postgresql/05-data-types-jsonb-fts-and-features.md#queue)
- CDC with Debezium: [`databases/analytics/olap-warehousing-and-data-pipelines.md`](../databases/analytics/olap-warehousing-and-data-pipelines.md#cdc)
- Transactions and the outbox motivation: [`databases/fundamentals/03-transactions-isolation-concurrency.md`](../databases/fundamentals/03-transactions-isolation-concurrency.md#outside)
- Interview drills: [`interview-prep/backend-engineer/06-message-queues-event-driven.md`](../interview-prep/backend-engineer/06-message-queues-event-driven.md)

## Hands-on lab
1. Run Kafka (KRaft) via Docker: `docker run -p 9092:9092 apache/kafka:latest`. Create a 6-partition topic, produce keyed messages, and run 3 consumers in a group. Kill one and watch the rebalance.
2. Implement the transactional outbox with Postgres + a polling relay, then switch to Debezium.
3. Build an idempotent consumer with an inbox table, and prove it by redelivering the same message 10 times.
4. Implement a 3-step order saga twice: choreography (events) and orchestration (Temporal dev server).
5. RabbitMQ: build retry-with-delay via TTL queues + DLX, and send poison messages to a parking lot after 3 attempts.
6. Run a distributed cron with a Postgres advisory lock across 3 app instances, and verify single execution.
