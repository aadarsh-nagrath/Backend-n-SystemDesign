# RabbitMQ, Amazon SQS/SNS/EventBridge, NATS, and Redis as a Broker

## Table of Contents
1. [RabbitMQ: AMQP 0-9-1 Model](#rabbit)
2. [Exchanges and Routing](#exchanges)
3. [Queues: Classic, Quorum, Streams](#queues)
4. [Reliability: Publisher Confirms, Consumer Acks, Prefetch](#reliability)
5. [Dead-Lettering, TTL, Delayed Retries, Priorities](#dlx)
6. [RabbitMQ Clustering, Operations, Pitfalls](#rabbitops)
7. [Amazon SQS](#sqs)
8. [SNS and the Fan-Out Pattern](#sns)
9. [EventBridge](#eventbridge)
10. [Google Pub/Sub and Azure Service Bus (brief)](#others)
11. [NATS and JetStream](#nats)
12. [Redis: Pub/Sub, Lists, Streams](#redis)
13. [Comparison Table](#compare)
14. [Interview Questions](#qa)

---

## 1. RabbitMQ: The AMQP Model {#rabbit}

RabbitMQ (Erlang/OTP, now Broadcom) is a general-purpose message broker. It's the classic choice for **task queues, flexible routing, and per-message acknowledgment**. It speaks AMQP 0-9-1 natively, plus AMQP 1.0 (native since 4.0), MQTT, STOMP, and its own Streams protocol.

```
Producer ──publish(exchange, routing_key, msg)──► Exchange ──bindings──► Queue(s) ──► Consumer(s)
```
- Producers **never publish directly to queues**. They publish to an **exchange** (the default exchange `""` routes by queue name, which looks like direct-to-queue).
- **Bindings** connect exchanges to queues (with a binding key or arguments).
- **Connections** are TCP (expensive; long-lived), and **channels** are lightweight virtual connections multiplexed on one connection (one per thread is the usual pattern).
- **Virtual hosts** (vhosts) give multi-tenant isolation within a broker.

---

## 2. Exchanges and Routing {#exchanges}

| Exchange type | Routing rule | Use |
|---|---|---|
| **direct** | routing key == binding key | Task routing by type (`email`, `sms`) |
| **fanout** | All bound queues (ignore key) | Broadcast/pub-sub |
| **topic** | Pattern match on dot-separated keys: `*` = one word, `#` = zero or more | `orders.*.created`, `logs.#.error` |
| **headers** | Match on message headers (`x-match: all/any`) | Multi-attribute routing |
| consistent-hash (plugin) | Hash of routing key → one of N queues | Partitioning with ordering per key |
| x-local-random (4.x) / others | | |

Example topology: `orders` topic exchange →
- `billing.q` bound with `order.created`
- `analytics.q` bound with `order.#`
- `eu-fulfillment.q` bound with `order.*.eu`

**Alternate exchange**: catches unroutable messages (otherwise they're silently dropped, unless `mandatory=true` makes the broker return them).

---

## 3. Queue Types {#queues}

| Type | Durability/replication | Notes |
|---|---|---|
| **Classic** | Single node (mirrored classic queues **removed in 4.0**) | Fast, non-replicated; fine for transient or non-critical data |
| **Quorum queues** (3.8+) | **Raft-replicated** across 3/5 nodes | The default choice for durable data. Poison-message handling (delivery limit, default 20 in 4.0), at-least-once, leader election |
| **Streams** (3.9+) | Replicated append-only log | Kafka-like: non-destructive consumption, replay from offset, large fan-out, very high throughput. Super streams = partitioned streams |

Queue properties: `durable` (survives broker restart), `exclusive` (one connection, deleted on close), `auto-delete`, arguments (`x-queue-type`, `x-max-length`, `x-overflow` reject-publish/drop-head, `x-message-ttl`, `x-dead-letter-exchange`, `x-single-active-consumer`).

Messages: `delivery_mode=2` (persistent) to survive restarts on durable queues (quorum queues always persist).

**Lazy mode** is now the default behavior: messages go to disk early, which avoids memory alarms with long queues.

---

## 4. Reliability {#reliability}

End-to-end reliability needs **both sides** confirmed:

**Publisher confirms** (`confirm.select`): the broker acks the publish after the message is safely enqueued (and, for quorum queues, replicated to a majority). Unconfirmed publishes must be retried by the producer, which can create duplicates, so consumers must be idempotent. Without confirms, a broker crash can silently lose "sent" messages.

**Consumer acknowledgments** (manual ack mode):
- `basic.ack` after successful processing, which removes the message.
- `basic.nack(requeue=false)` / `basic.reject` sends it to the DLX if configured. `requeue=true` puts it back (beware infinite hot loops with poison messages).
- If a consumer's channel or connection closes with unacked messages, they're **redelivered** (with the `redelivered` flag).
- **Consumer delivery acknowledgement timeout** (default 30 min): unacked beyond this closes the channel.

**Prefetch (QoS)**: `basic.qos(prefetch_count=N)` caps unacked messages per consumer.
- Too high: one consumer hoards messages (unfair distribution; memory). Too low (1): throughput suffers from round trips.
- Typical: 10–300 depending on processing time. Use 1 for long tasks needing fair distribution.

Ordering: a single queue with a single consumer is FIFO. With multiple consumers, or with requeues/redeliveries, order is not guaranteed. `x-single-active-consumer` keeps exactly one consumer active (failover), which preserves order.

---

## 5. Dead-Lettering, TTL, Delayed Retries, Priorities {#dlx}

A message is **dead-lettered** to the queue's DLX when it's rejected/nacked with requeue=false, its TTL expires, the queue length limit is exceeded (drop-head), or the quorum queue delivery limit is exceeded.

Retry-with-delay pattern (no plugin):
```
work.q  ──(nack, requeue=false)──►  DLX "retry"  ──► retry.30s.q (x-message-ttl=30000, x-dead-letter-exchange="work")
                                                          │ (TTL expires)
                                                          └──► back to work exchange → work.q
After N attempts (count via x-death header) → publish to parking-lot / DLQ for humans
```
Or use the **delayed message exchange plugin** (`x-delayed-message`).

**Priority queues**: `x-max-priority` (classic queues; use a small range like 1–5). Quorum queues support two priorities (normal/high) since 4.0.

**Per-message TTL / queue TTL / queue expiry** for ephemeral data.

---

## 6. RabbitMQ Clustering and Operations {#rabbitops}

- A cluster of nodes shares metadata (users, vhosts, exchanges, bindings, queue definitions; **Khepri** (Raft-based metadata store) replaces Mnesia in 4.x). Queue **contents** live on their leader/replicas (quorum queues: Raft groups).
- Use an odd number of nodes (3, 5). Handle network partitions with `pause_minority` for classic queues (quorum queues handle partitions via Raft).
- **Federation** and **Shovel** for cross-datacenter or cross-cluster message movement.
- **Memory and disk alarms**: when memory exceeds the watermark (40% of RAM default), or free disk drops below the limit, the broker **blocks all publishers** (flow control). The app sees publishes hang. Monitor them.
- **Management UI/API** (port 15672), Prometheus plugin, `rabbitmq-diagnostics`, `rabbitmqctl list_queues name messages consumers`.
- Pitfalls:
  - **Long queues**: millions of messages backing up mean a memory and disk crunch and slow recovery. RabbitMQ is happiest when queues are near-empty (consumers keep up). Use streams or Kafka for big backlogs.
  - **Connection churn**: opening a connection per publish (common in PHP/serverless) kills performance. Use long-lived connections and pooled channels.
  - Unbounded queues without `x-max-length` or TTL.
  - Requeue loops with poison messages (use a quorum queue delivery limit plus a DLX).
  - Too many queues (100k+) or too many channels.
- Kubernetes: the RabbitMQ Cluster Operator. Managed: Amazon MQ, CloudAMQP.

---

## 7. Amazon SQS {#sqs}

Fully managed queues with no capacity planning.

| | **Standard** | **FIFO** |
|---|---|---|
| Throughput | Nearly unlimited | 300 msg/s per API action (3,000 with batching); **high-throughput mode** up to tens of thousands/s per queue across message groups |
| Ordering | Best-effort | **Strict per `MessageGroupId`** |
| Delivery | **At-least-once** (occasional duplicates) | **Exactly-once processing** within a 5-minute dedup window (`MessageDeduplicationId` or content-based dedup) |
| Use | Most workloads | Order-sensitive per entity, dedup needed |

Mechanics:
- **Visibility timeout** (default 30 s, max 12 h): a received message is hidden from other consumers. Delete it before the timeout, or it reappears. Extend it with `ChangeMessageVisibility` for long jobs.
- **Long polling** (`WaitTimeSeconds` up to 20): reduces empty receives and cost. Always use it.
- Batch APIs (`SendMessageBatch`, `DeleteMessageBatch`: 10 messages) for cost and throughput.
- **Message retention** 1 min to 14 days (default 4 days). **Max message size 256 KB** (raised to 1 MiB in 2025). Use the Extended Client Library with S3 for larger payloads.
- **Delay queues / message timers** (0–15 min) for delayed delivery.
- **DLQ via redrive policy** (`maxReceiveCount`). **DLQ redrive** API/console to move messages back. Monitor `ApproximateAgeOfOldestMessage` and DLQ `ApproximateNumberOfMessagesVisible`.
- **Lambda event source mapping**: Lambda polls SQS, invokes with batches, and deletes on success. Use **partial batch responses** (`ReportBatchItemFailures`) so one failure doesn't retry the whole batch. Set the visibility timeout to at least 6× the function timeout. Use maximum concurrency settings to protect downstreams.
- FIFO pitfall: a failing message blocks its **message group** until it succeeds or goes to the DLQ (per-group HOL blocking, by design). Use fine-grained group IDs (per order, not per tenant).
- Pricing per request (64 KB chunks), so batching saves money.
- Security: IAM policies, SSE-KMS or SSE-SQS encryption, VPC endpoints.

---

## 8. SNS and Fan-Out {#sns}

**Amazon SNS**: managed pub/sub. A topic pushes to subscribers: SQS queues, Lambda, HTTP(S) endpoints, email, SMS, mobile push, Firehose.
- **SNS → multiple SQS queues** is the canonical AWS fan-out pattern: each consuming service gets its own durable queue, retries, and DLQ. Producers stay unaware of consumers.
- **Message filtering policies** on subscriptions (by message attributes or body) so each queue receives only relevant events.
- SNS FIFO topics → SQS FIFO queues for ordered fan-out.
- Delivery retries per protocol, and a DLQ for failed deliveries to subscribers.
- **Raw message delivery** avoids wrapping the payload in SNS JSON.

---

## 9. EventBridge {#eventbridge}

Serverless event bus for AWS-native event-driven architectures:
- Events (JSON) are put on a **bus** (default bus receives AWS service events, custom buses for apps, partner buses for SaaS like Stripe/Datadog/Shopify).
- **Rules** with content-based **event patterns** route to 20+ target types (Lambda, SQS, Step Functions, API destinations (any HTTP API with auth), other buses, cross-account and cross-region).
- **Schema registry/discovery**, **archive and replay**, **EventBridge Pipes** (point-to-point source → filter → enrich → target, e.g., DynamoDB Stream → Step Functions), **EventBridge Scheduler** (cron and one-time schedules at scale).
- Trade-offs vs SNS/SQS: richer routing and integrations, but higher latency (~hundreds of ms typical) and lower default throughput quotas than SNS. Choose SNS+SQS for high-volume fan-out, EventBridge for integration and routing flexibility.

---

## 10. Google Pub/Sub and Azure Service Bus {#others}

**Google Cloud Pub/Sub**: global, serverless, topics + subscriptions (pull, push to HTTPS, BigQuery/Cloud Storage subscriptions), at-least-once by default with an exactly-once delivery option (per subscription, regional), ordering keys, DLQ topics, seek and replay to a timestamp or snapshot, retention up to 31 days. **Pub/Sub Lite** was deprecated (2024).

**Azure Service Bus**: enterprise broker with queues, topics, and subscriptions (with SQL-like filters). Sessions (ordered processing per session ID, like FIFO groups), duplicate detection, scheduled messages, deferral, DLQ, transactions across entities, and AMQP 1.0. **Azure Event Hubs** is the Kafka-like streaming service (with a Kafka-compatible endpoint).

---

## 11. NATS and JetStream {#nats}

**NATS** (CNCF): a very lightweight, high-performance messaging system written in Go.
- **Core NATS**: subject-based pub/sub (`orders.created.eu`, wildcards `*` and `>`), **request-reply** built in, **queue groups** (load-balanced subscribers). **At-most-once**, with no persistence. If no subscriber is listening, the message is gone. Sub-millisecond latency, millions of msgs/s.
- **JetStream**: persistence layer with streams (retention by limits, interest, or work-queue), consumers (push/pull, durable, ack policies, redelivery, max deliver, backoff), exactly-once publish via `Nats-Msg-Id` dedup + double-ack, replication via Raft, a **KV store** and an **object store** on top.
- Strengths: simplicity, a single small binary, multi-tenancy (accounts), leaf nodes and superclusters for edge/hybrid/multi-cloud topologies, great for IoT and edge.
- Users: microservices control planes, IoT, edge computing, Synadia Cloud.

---

## 12. Redis as a Broker {#redis}

| Feature | Semantics | Use |
|---|---|---|
| **Pub/Sub** (`PUBLISH`/`SUBSCRIBE`) | Fire-and-forget, at-most-once, no persistence, disconnected subscribers miss messages | WebSocket backplanes, cache invalidation broadcasts, live notifications where loss is OK. **Sharded pub/sub** (`SSUBSCRIBE`, Redis 7) scales in cluster mode |
| **Lists** (`LPUSH` + `BRPOP` / `BLMOVE`) | Simple queue; `BLMOVE` into a processing list enables reliable-queue patterns | Simple job queues (Sidekiq, RQ, Resque, older Celery broker) |
| **Streams** (`XADD`, `XREADGROUP`, `XACK`, `XAUTOCLAIM`) | Append-only log with IDs, **consumer groups**, pending entries list (PEL), acks, claiming stuck messages, `MAXLEN`/`MINID` trimming | Lightweight event streams, job queues with acks, activity feeds |
| Sorted sets | Score = timestamp | Delayed/scheduled jobs (poll `ZRANGEBYSCORE ≤ now`) |

Caveats: memory-bound (all data in RAM), persistence is async (RDB/AOF; `appendfsync everysec` can lose ~1 s), and failover with async replication can lose acknowledged writes. Fine for jobs that can be re-derived or tolerate rare loss. Not a Kafka replacement for critical high-volume streams. (Valkey is the Linux Foundation fork after Redis's 2024 license change. Redis 8 returned to an OSI license, AGPLv3, in 2025.)

---

## 13. Comparison {#compare}

| | RabbitMQ | Kafka | SQS | SNS | NATS JetStream | Redis Streams |
|---|---|---|---|---|---|---|
| Model | Queue + routing (and streams) | Partitioned log | Queue | Pub/sub push | Streams + core pub/sub | Log + consumer groups |
| Retention after consume | No (streams: yes) | Yes | No | No | Configurable | Until trimmed |
| Replay | Streams only | Yes | No | No | Yes | Yes |
| Ordering | Per queue (single consumer) | Per partition | FIFO per group | FIFO topics | Per stream/subject | Per stream |
| Throughput | 10k–100k+/s per queue | Millions/s cluster | Very high (standard) | Very high | Very high | High (memory) |
| Latency | Low (ms) | Low–mid (batching) | ~10s of ms | ~10s–100s ms | Sub-ms–ms | Sub-ms |
| Ops | Medium | High (self-managed) | None | None | Low | Low |
| Best for | Task queues, complex routing, RPC | Event streaming, CDC, analytics | AWS work queues | AWS fan-out | Lightweight, edge, request-reply | Small-scale streams/jobs |

---

## 14. Interview Questions {#qa}

1. Explain RabbitMQ exchanges, queues, and bindings. When would you use a topic exchange?
2. How do publisher confirms and consumer acks together provide at-least-once delivery?
3. What does prefetch control, and what goes wrong if it's too high?
4. How do you implement delayed retries in RabbitMQ?
5. Classic vs quorum queues vs streams?
6. SQS Standard vs FIFO? What's a visibility timeout and how do you choose it?
7. Explain the SNS → SQS fan-out pattern and its advantages.
8. When would you choose RabbitMQ over Kafka, and vice versa?
9. Why is Redis Pub/Sub unsuitable for critical events?
10. A FIFO queue's processing is stuck for one customer but not others. Why?
