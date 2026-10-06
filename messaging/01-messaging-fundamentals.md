# Messaging Fundamentals: Queues, Pub/Sub, Logs, and Delivery Semantics

## Table of Contents
1. [Why Asynchronous Messaging](#why)
2. [Models: Point-to-Point Queue, Pub/Sub, Log/Stream](#models)
3. [Message Anatomy](#anatomy)
4. [Delivery Semantics: At-Most-Once, At-Least-Once, Exactly-Once](#semantics)
5. [Idempotent Consumers (the real answer)](#idempotency)
6. [Ordering](#ordering)
7. [Acknowledgments, Visibility Timeouts, Redelivery](#acks)
8. [Retries, Backoff, and Dead-Letter Queues](#dlq)
9. [Backpressure and Flow Control](#backpressure)
10. [Consumer Scaling and Competing Consumers](#scaling)
11. [Poison Messages and Error Classification](#poison)
12. [Schemas and Evolution](#schemas)
13. [Choosing a Broker](#choosing)
14. [Interview Questions](#qa)

---

## 1. Why Asynchronous Messaging {#why}

Synchronous call chains (A → B → C over HTTP) couple availability and latency: if C is slow or down, A is slow or down. Messaging decouples them:

| Benefit | Example |
|---|---|
| **Temporal decoupling** | Producer doesn't wait for the consumer; the consumer can be down temporarily |
| **Load leveling** | A burst of 50k orders/min is buffered and consumed at a steady rate |
| **Scalability** | Add consumers to increase throughput |
| **Fan-out** | One `OrderPlaced` event → email, analytics, inventory, fraud services |
| **Resilience** | Retries without blocking the user request |
| **Long-running work off the request path** | Video transcoding, PDF generation, ML inference |
| **Integration** | Services evolve independently via events |

Costs: eventual consistency, harder debugging (distributed tracing needed), duplicate and out-of-order messages, operational burden of brokers, and harder end-to-end testing.

---

## 2. Models {#models}

### Point-to-point queue (work queue)
Each message is processed by **one** consumer among many competing consumers. The message is deleted after acknowledgment.
Examples: RabbitMQ queues, SQS, ActiveMQ, Sidekiq/Celery/BullMQ.

### Publish/Subscribe
Each message goes to **every subscriber** (each subscriber has its own copy or queue).
Examples: SNS → SQS fan-out, RabbitMQ fanout/topic exchanges, Google Pub/Sub subscriptions, Redis Pub/Sub (fire-and-forget, no persistence), NATS.

### Log / stream (append-only, replayable)
Messages are appended to an ordered, durable, partitioned log and **retained** (by time or size) regardless of consumption. Each consumer group tracks its own **offset**. Consumers can **replay** from any offset.
Examples: **Kafka**, Redpanda, Pulsar, Kinesis, Azure Event Hubs, Redis Streams, NATS JetStream.

| | Queue (RabbitMQ/SQS) | Log (Kafka) |
|---|---|---|
| After consumption | Deleted | Retained (days/forever) |
| Replay | No (DLQ redrive only) | Yes: reset offsets |
| Multiple independent consumers | Need separate queues | Built in (consumer groups) |
| Ordering | Per queue (weakens with competing consumers) | **Per partition** |
| Per-message ack/redelivery | Yes, fine-grained | Offset commit (position), not per message |
| Routing flexibility | Rich (exchanges, bindings, filters) | Topic + partition key |
| Throughput | High | Very high (sequential I/O, batching) |
| Typical use | Task queues, RPC, workflows | Event streaming, CDC, analytics pipelines, event sourcing |

---

## 3. Message Anatomy {#anatomy}

```json
{
  "id": "01J1Z7K6Y3...",                 // unique (UUIDv7/ULID) for dedup
  "type": "order.placed",                  // event name, past tense for events
  "version": 2,                            // schema version
  "source": "orders-service",
  "time": "2025-06-01T10:00:00Z",
  "subject": "ord_123",                    // entity / partition key
  "correlation_id": "req_abc",             // ties to originating request
  "causation_id": "evt_prev",              // which message caused this one
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "data": { "order_id": "ord_123", "customer_id": "cus_9", "total_minor": 49900, "currency": "INR" }
}
```
**CloudEvents** (CNCF) standardizes these attributes across brokers and protocols.

**Commands vs events**:
- **Command**: "do this" (`SendEmail`, `ChargeCard`). One intended handler. May be rejected.
- **Event**: "this happened" (`OrderPlaced`). It's a fact, immutable, with zero or more interested subscribers. The publisher doesn't know or care who listens.
- **Event notification** (thin: "order 123 changed", consumers call back for details) vs **event-carried state transfer** (fat: the full state in the event, so consumers keep local copies and don't need callbacks).

---

## 4. Delivery Semantics {#semantics}

| Semantics | Meaning | How it arises | Risk |
|---|---|---|---|
| **At-most-once** | 0 or 1 deliveries | Ack/commit **before** processing; no retries | Message loss |
| **At-least-once** | ≥1 deliveries | Ack/commit **after** processing; retry on failure | **Duplicates** |
| **Exactly-once** | Effect happens once | Only within closed systems (Kafka transactions read-process-write), or via idempotency/dedup | Expensive or impossible end-to-end |

Why exactly-once delivery is impossible in general: the consumer processes the message, then crashes before the ack reaches the broker. The broker must either redeliver (duplicate) or not (risking loss). This is the Two Generals problem.

**Practical stance: use at-least-once delivery + idempotent processing = effectively-once processing.**

Kafka's "exactly-once semantics" (EOS) covers: idempotent producers (no duplicate appends on retry) + transactions spanning consume-offset commit and produce (Kafka → Kafka processing). Side effects outside Kafka (DB writes, emails, HTTP calls) still need idempotency.

---

## 5. Idempotent Consumers {#idempotency}

Techniques:
1. **Natural idempotency**: `SET status = 'shipped'` is idempotent, while `balance = balance - 10` isn't.
2. **Dedup table (inbox pattern)**: in the same DB transaction as the side effect:
   ```sql
   BEGIN;
   INSERT INTO processed_messages (message_id, consumer) VALUES ($1, 'billing');  -- PK conflict → duplicate → skip
   UPDATE accounts SET balance = balance - 10 WHERE id = $2;
   COMMIT;
   ```
3. **Upserts keyed by business ID** (`INSERT … ON CONFLICT (order_id) DO NOTHING`).
4. **Version/sequence checks**: apply only if `event.version > entity.version` (also handles out-of-order delivery).
5. **Idempotency keys for external calls**: pass the message ID as the `Idempotency-Key` to payment providers.
6. Dedup windows in brokers: SQS FIFO (5-minute dedup window by `MessageDeduplicationId`), Azure Service Bus duplicate detection, NATS JetStream `Nats-Msg-Id`. These are bounded windows, not a complete solution.

---

## 6. Ordering {#ordering}

- Global total order doesn't scale (it forces a single partition/consumer).
- Most systems need **order per entity** (all events for `order_123` in sequence). Achieve it by **partitioning by entity key**: Kafka partition key, SQS FIFO `MessageGroupId`, RabbitMQ consistent-hash exchange or single active consumer, Pub/Sub ordering keys.
- Ordering breaks with: multiple competing consumers on one queue, retries (a failed message retried later lands after newer ones), DLQ redrives, producers with multiple in-flight requests (Kafka without idempotence: `max.in.flight.requests.per.connection > 1` + retries could reorder; idempotent producers fix this).
- When a message fails and must block the key (strict ordering), you need per-key retry handling: pause that key or partition, or park subsequent messages for the key. This is a trade-off against throughput (head-of-line blocking per partition).
- Consumers can also tolerate disorder: use version numbers and ignore stale events, or make operations commutative.

---

## 7. Acks, Visibility Timeouts, Redelivery {#acks}

- **RabbitMQ**: the consumer gets a message with a delivery tag and `ack`s (remove), `nack`/`reject` (requeue or dead-letter), or the connection drops (requeued). **Prefetch** (`basic.qos`) limits unacked messages per consumer.
- **SQS**: `ReceiveMessage` hides the message for the **visibility timeout**. The consumer must `DeleteMessage` before it expires, or the message becomes visible again (redelivery). Extend it for long jobs (`ChangeMessageVisibility`, a heartbeat). Set the visibility timeout above p99 processing time.
- **Kafka**: the consumer commits **offsets** (auto every 5 s by default, which risks loss or duplicates depending on timing, or manually after processing). Committing offset N means "everything before N is done". There's no per-message ack, so a single failed message needs app-level handling (retry topic/DLQ).
- **Lease/heartbeat** semantics in job systems (Sidekiq, BullMQ locks, Temporal heartbeats).

---

## 8. Retries, Backoff, DLQs {#dlq}

- **Immediate retry** for transient blips (once or twice), then **delayed retries with exponential backoff + jitter** (1 s, 2 s, 4 s … up to minutes or hours).
- Implementations:
  - RabbitMQ: DLX + per-message TTL "retry queues" (wait queues with TTL that dead-letter back to the main queue), or the delayed message plugin.
  - SQS: visibility timeout as backoff, `maxReceiveCount` → **redrive policy** to a DLQ.
  - Kafka: **retry topics** (`orders.retry.1m`, `orders.retry.10m`) + DLQ topic (Spring Kafka `@RetryableTopic`, Uber's design). The main partition isn't blocked.
- **Dead-letter queue (DLQ)**: messages that exhausted retries or are unprocessable. **Alert on DLQ depth**, inspect, fix, and **redrive** (replay) them. A DLQ nobody monitors is a black hole.
- Retry budgets and circuit breakers: if a downstream is down, retrying every message hammers it. Pause consumption instead (the circuit breaker opens, the consumer stops polling, and the queue absorbs the backlog).

---

## 9. Backpressure {#backpressure}

When producers outpace consumers:
- The queue grows, which is fine short-term (that's the point), but watch **queue depth / consumer lag** and the **age of the oldest message** (the best SLO metric for async pipelines).
- Bounded in-memory buffers in consumers (prefetch limits, `max.poll.records`).
- Producers may need to slow down: RabbitMQ **flow control/memory alarms** block publishers, Kafka producers block when `buffer.memory` is full (`max.block.ms`).
- Autoscale consumers on lag (KEDA scales Kubernetes deployments on Kafka lag, SQS depth, RabbitMQ queue length).
- Shed or degrade non-critical work under overload.

---

## 10. Consumer Scaling {#scaling}

- **Competing consumers**: N consumers on one queue share the work. RabbitMQ/SQS scale nearly linearly until the broker or downstream limits.
- **Kafka**: parallelism is capped by the **number of partitions** (max one consumer per partition per group). Choose partitions for future peak consumer parallelism (e.g., 12–48 for moderate topics). Increasing partitions later **changes key → partition mapping**, which breaks per-key ordering during the transition.
- Within a consumer: process a partition's messages concurrently **by key** (e.g., Confluent Parallel Consumer) to scale beyond the partition count while preserving per-key order.
- Downstream limits (DB connections, API rate limits) are usually the real bottleneck. Size consumer concurrency to them.

---

## 11. Poison Messages {#poison}

A **poison message** always fails (malformed payload, bug, unexpected schema). Without handling, it's retried forever: it blocks a Kafka partition, or consumes resources in a queue.
- Classify errors:
  - **Transient** (timeout, 503, deadlock, rate limit): retry with backoff.
  - **Permanent** (validation failure, missing referenced entity that will never exist, deserialization error): send to the DLQ immediately with error metadata (exception, stack, attempt count, original topic/partition/offset).
- Log message IDs and keys, never full sensitive payloads.
- Make the DLQ tooling good: search, inspect, edit and replay, bulk redrive.

---

## 12. Schemas and Evolution {#schemas}

Events are **contracts between teams**, often consumed by services you don't know about.
- Serialization formats: JSON (human-readable, schema optional, larger), **Avro** (compact, schema required, great evolution rules, common with Kafka), **Protobuf** (compact, field numbers, gRPC ecosystem), JSON Schema.
- **Schema Registry** (Confluent, Apicurio, AWS Glue, Redpanda) stores versions and enforces **compatibility** modes:
  - **Backward**: new consumers can read old data (you can delete fields, add optional fields with defaults). Upgrade consumers first.
  - **Forward**: old consumers can read new data (add fields; delete optional fields). Upgrade producers first.
  - **Full**: both.
  - Transitive variants check against all previous versions.
- Rules of thumb: only add optional fields with defaults, never reuse field names or numbers with new meaning, never change types, and introduce a **new event type/version** for breaking changes (publish both during migration).
- Consumers must **ignore unknown fields** (tolerant reader).
- Document events with **AsyncAPI**.

---

## 13. Choosing a Broker {#choosing}

| Need | Good choice |
|---|---|
| Simple background jobs in a monolith | DB-backed queue (Postgres `SKIP LOCKED`), Redis-based (Sidekiq, BullMQ, Celery w/ Redis) |
| Managed, zero-ops work queue on AWS | **SQS** (+ SNS for fan-out, EventBridge for routing) |
| Complex routing, priorities, RPC, per-message acks, moderate throughput | **RabbitMQ** |
| High-throughput event streaming, replay, multiple consumer groups, CDC, stream processing | **Kafka** (or Redpanda / MSK / Confluent Cloud) |
| Multi-tenant geo-replicated streaming with tiered storage + queues | Pulsar |
| Lightweight, very low latency pub/sub, edge/IoT, request-reply | **NATS** (JetStream for persistence) |
| GCP / Azure native | Google Pub/Sub; Azure Service Bus (queues/topics) and Event Hubs (Kafka-compatible stream) |
| Durable long-running workflows with state and timers | **Temporal** / AWS Step Functions (workflow engines, not brokers) |
| IoT devices | MQTT brokers (Mosquitto, EMQX, HiveMQ, AWS IoT Core) |

---

## 14. Interview Questions {#qa}

1. Queue vs pub/sub vs log: differences and when to use each.
2. Explain at-most-once, at-least-once, exactly-once. Why is exactly-once delivery impossible end-to-end?
3. How do you make a consumer idempotent?
4. How do you guarantee per-customer ordering while scaling consumers?
5. What's a visibility timeout? What happens if processing exceeds it?
6. Design retry handling for a Kafka consumer without blocking the partition.
7. What is a poison message and how do you deal with it?
8. How do you evolve event schemas without breaking consumers?
9. Which metric best indicates an async pipeline is unhealthy? (Age of oldest message / consumer lag.)
