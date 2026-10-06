# Event-Driven Architecture Patterns: Outbox, Inbox, Sagas, CQRS, Event Sourcing

> These patterns are how microservices stay consistent without distributed transactions. [`system-design/notes/concepts.md`](../system-design/notes/concepts.md) introduces sagas and CQRS briefly. This note goes deep, with code-level detail.

## Table of Contents
1. [The Core Problem: Dual Writes](#dualwrite)
2. [Transactional Outbox](#outbox)
3. [Inbox / Idempotent Receiver](#inbox)
4. [Sagas: Choreography vs Orchestration](#sagas)
5. [Designing Compensations and Semantic Locks](#compensation)
6. [CQRS](#cqrs)
7. [Event Sourcing](#es)
8. [Event Sourcing + CQRS: Projections and Rebuilds](#projections)
9. [Domain Events vs Integration Events](#domain)
10. [Other Patterns: Claim Check, Competing Consumers, Process Manager, Scatter-Gather, Strangler via Events](#other)
11. [Consistency and UX in Eventually Consistent Systems](#ux)
12. [Observability of Event-Driven Systems](#observability)
13. [When NOT to Go Event-Driven](#when)
14. [Interview Questions](#qa)

---

## 1. The Dual-Write Problem {#dualwrite}

A service must update its database **and** publish an event:
```python
def place_order(cmd):
    db.insert(order)            # 1
    kafka.publish(OrderPlaced)  # 2
```
- Crash between 1 and 2: the order exists, but no event goes out, so inventory is never reserved and no email is sent.
- Publish first, then the DB fails: the event announces an order that doesn't exist.
- Publishing inside the DB transaction doesn't help: the broker isn't part of the transaction (the event can go out and then the commit fails).
- 2PC/XA across DB and broker: rarely supported (Kafka doesn't support XA), slow, and fragile.

The solutions are the outbox (write the event into the same DB transaction) and CDC.

---

## 2. Transactional Outbox {#outbox}

```sql
CREATE TABLE outbox (
  id            UUID PRIMARY KEY,
  aggregate_type TEXT NOT NULL,       -- 'order'
  aggregate_id  TEXT NOT NULL,        -- partition key
  event_type    TEXT NOT NULL,        -- 'OrderPlaced'
  payload       JSONB NOT NULL,
  headers       JSONB,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at  TIMESTAMPTZ           -- for the polling variant
);
```
```python
with db.transaction():
    db.insert("orders", order)
    db.insert("outbox", {"id": uuid7(), "aggregate_type": "order", "aggregate_id": order.id,
                         "event_type": "OrderPlaced", "payload": to_json(order)})
# Commit is atomic: both rows or neither.
```
A **relay** then publishes outbox rows to the broker:

**Option A: polling publisher**
```sql
SELECT * FROM outbox WHERE published_at IS NULL ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED;
-- publish each (await broker ack), then
UPDATE outbox SET published_at = now() WHERE id = ANY($1);   -- or DELETE
```
It's simple, but polling adds latency and load. Ordering across relay instances needs care (one relay per partition key range, or a single active relay). Clean up published rows regularly (partition by day and drop).

**Option B: log-based CDC** (Debezium outbox event router): Debezium tails the WAL/binlog, sees inserts into `outbox`, and routes them to topics by `aggregate_type`, keyed by `aggregate_id`. Ordering follows commit order. There's no polling, and rows can be deleted right after insert (the WAL still has them).

Properties:
- **At-least-once publication**: the relay may crash after publishing but before marking the row, so it republishes. **Consumers must be idempotent** (use the outbox `id` as the message ID).
- Ordering per aggregate is preserved if the relay publishes in commit order per key.
- Events reflect committed state only.

Variants: the **"listen to yourself"** pattern (write only the event to Kafka, and the service consumes its own event to update its DB, which makes the log the source of truth), and event sourcing (the event store *is* the database).

---

## 3. Inbox / Idempotent Receiver {#inbox}

Consumers receive duplicates (at-least-once). The inbox records processed message IDs **in the same transaction** as the state change:
```python
def handle(msg):
    with db.transaction():
        inserted = db.execute(
            "INSERT INTO inbox (message_id, handler) VALUES (%s, %s) ON CONFLICT DO NOTHING",
            (msg.id, "reserve-inventory")).rowcount
        if inserted == 0:
            return                     # duplicate: already processed
        reserve_inventory(msg.data)    # state change in same tx
    ack(msg)                           # ack after commit
```
Retention: keep inbox rows longer than the max redelivery window, then purge them.

The **outbox + inbox** pair gives effectively-once processing across services.

---

## 4. Sagas {#sagas}

A **saga** is a sequence of **local transactions**, each in one service, each publishing an event or message that triggers the next step. If a step fails, **compensating transactions** undo the previous steps semantically. (The term comes from Garcia-Molina & Salem, 1987.)

Example: place order → reserve inventory → charge payment → schedule shipping.

### Choreography (decentralized)
Each service reacts to events and emits new ones:
```
Order svc:     OrderCreated(PENDING) ─────────────►
Inventory svc:      ◄── consumes OrderCreated → reserves → InventoryReserved ──►
Payment svc:                         ◄── consumes InventoryReserved → charges → PaymentSucceeded / PaymentFailed ──►
Order svc:          ◄── PaymentSucceeded → order CONFIRMED
                    ◄── PaymentFailed → order CANCELLED
Inventory svc:      ◄── PaymentFailed → release reservation (compensation)
```
- Pros: loose coupling, no central coordinator, simple for 2–4 steps.
- Cons: the workflow is implicit and spread across services (hard to see, debug, or change), risk of cyclic dependencies, and it's hard to answer "what state is order 123's saga in?"

### Orchestration (centralized)
An **orchestrator** (saga execution coordinator) tells each participant what to do and tracks state:
```
OrderSaga orchestrator (state machine persisted in DB):
  1. send ReserveInventory → wait InventoryReserved | InventoryFailed
  2. send ChargePayment    → wait PaymentSucceeded  | PaymentFailed
       on PaymentFailed: send ReleaseInventory (compensate), mark order CANCELLED
  3. send ScheduleShipment → ...
  4. mark order CONFIRMED
```
- Pros: explicit workflow, central visibility, easier to change, timeouts and retries in one place, no cyclic event dependencies.
- Cons: orchestrator logic can become a "god service" (keep business rules in participants), and the orchestrator itself must be reliable (persistent state, idempotent steps).
- Implementations: **Temporal** / Cadence (workflow as code with durable execution), **AWS Step Functions**, Camunda/Zeebe (BPMN), Netflix Conductor/Orkes, Axon (Java), MassTransit sagas (.NET), NServiceBus sagas, Eventuate Tram, or a hand-rolled state machine table + outbox.

Rule of thumb: choreography for simple flows with few participants, and orchestration once you have more than ~3 steps, complex branching, or timeouts.

---

## 5. Compensations and Semantic Locks {#compensation}

Sagas give up **isolation (the I in ACID)**: other transactions see intermediate states. Design for it:
- **Compensating transactions** are semantic undos, not rollbacks: refund rather than "un-charge", "release reservation", "cancel shipment", "send apology email". Some actions can't be compensated (an email already sent), so order steps so that **irreversible steps come last** ("pivot transaction": after it, the saga only moves forward with retriable steps).
- Step types: **compensatable** (before the pivot), the **pivot** (go/no-go point), and **retriable** (after the pivot; must eventually succeed: retry forever and alert).
- **Semantic lock**: mark records with a pending state (`order.status = PENDING_PAYMENT`, `reservation.status = HELD`) so other operations know they're in flux (e.g., disallow cancelling while payment is in progress, or treat it specially).
- **Commutative updates**: design operations that can apply in any order (increment/decrement rather than set).
- **Pessimistic view**: reorder steps to minimize the business risk of dirty reads (e.g., credit check before shipping).
- **Reread value / version file**: verify data hasn't changed before acting.
- **Timeouts**: every step needs a deadline. A reservation hold expires automatically after 15 min if payment never completes.
- Every step and compensation must be **idempotent** (they will be retried).
- Compensation can fail too, so retry it, alert, and provide manual resolution tooling.

---

## 6. CQRS {#cqrs}

**Command Query Responsibility Segregation**: separate the **write model** (commands that validate business rules and change state) from the **read model(s)** (queries optimized for display).

```
            Commands                         Queries
Client ──► Command handler ──► Write DB    Client ──► Read API ──► Read store(s)
                 │ (events via outbox/CDC)                          ▲
                 └────────────────────► Projectors / updaters ──────┘
                                    (denormalized views: Elasticsearch, Redis, a view table, DynamoDB)
```
Why:
- Read and write workloads have different shapes and scale (reads often 100× writes).
- Read models can be **denormalized per screen/use case** (no joins at read time), stored in the most suitable technology (search engine, cache, graph).
- The write model stays focused on invariants (DDD aggregates).

Costs:
- **Eventual consistency** between write and read sides: the user may not see their change immediately (see §11).
- More moving parts, projection code to maintain, rebuild procedures.

Levels:
1. Same DB, separate code paths/models (cheap, common: query services using raw SQL views, command services using ORM aggregates).
2. Same DB, separate read tables or materialized views updated in the same transaction (consistent).
3. Separate read stores updated asynchronously via events (full CQRS, eventual consistency).

**CQRS doesn't require event sourcing**, and event sourcing practically requires CQRS (event streams are bad for queries).

---

## 7. Event Sourcing {#es}

Instead of storing current state, store the **sequence of events** that led to it. Current state = fold over events.
```
Account acc_1 stream:
  v1 AccountOpened   {owner: "asha"}
  v2 MoneyDeposited  {amount: 100}
  v3 MoneyWithdrawn  {amount: 30}
  v4 MoneyDeposited  {amount: 50}
→ state: balance = 120 (computed by replaying v1..v4)
```
Event store requirements:
- Append events to a stream with **optimistic concurrency**: "append at expected version 4" fails if someone else appended v5. This guards the aggregate's invariants.
- Read a stream from version N. Subscribe to all events (global ordered feed) for projections.
- Implementations: **EventStoreDB/KurrentDB**, Marten (PG, .NET), Axon Server, Eventuous, DynamoDB/Postgres tables (`events(stream_id, version, type, data, PRIMARY KEY(stream_id, version))`), Kafka (good for distribution; weak as a per-aggregate store with optimistic concurrency).

Command handling flow:
```
load events for aggregate → rebuild state → validate command against state → produce new events
→ append with expectedVersion → (on conflict: reload and retry) → publish/subscribe to projections
```

Benefits:
- **Complete audit log** by construction (finance, healthcare, compliance).
- **Temporal queries**: "what was the state at 3 pm last Tuesday?"
- **Rebuild read models** any time, and create new projections from history (new reports over past data).
- Debugging: replay production streams to reproduce bugs.
- A natural fit with event-driven integration.

Costs and pitfalls:
- **Event schema evolution forever**: you can't migrate old events easily (they're immutable facts). Use **upcasters** (transform old event versions on read), versioned event types, or (rarely) copy-and-transform migrations of the store.
- **Long streams** slow down loading → **snapshots** (persist state at version N, replay only later events).
- **GDPR / right to erasure** vs immutable logs → **crypto-shredding** (encrypt personal data per user with a key, and delete the key), or store PII outside events with references.
- Querying requires projections (CQRS), plus eventual consistency.
- Steep learning curve. Easy to over-apply. Use it for core domains where history matters, not for every CRUD table.
- Set-based validation (unique usernames across aggregates) needs a separate mechanism (a reservation table/aggregate, or a uniqueness index in a read model with compensation).

---

## 8. Projections and Rebuilds {#projections}

A **projection** consumes events and maintains a read model:
```python
def on_event(evt):
    match evt.type:
        case "MoneyDeposited": db.execute("UPDATE balances SET amount = amount + %s, version = %s WHERE account_id = %s AND version < %s", ...)
        case "AccountOpened":  db.execute("INSERT INTO balances (account_id, amount, version) VALUES (%s, 0, %s) ON CONFLICT DO NOTHING", ...)
    checkpoint(evt.global_position)     # store position in same tx as the read model update
```
- Idempotent via the version or checkpoint (store the checkpoint in the same transaction as the update).
- **Rebuild**: create a new read model table (v2), replay from position 0 into it, catch up to live, then switch reads to v2 (blue/green projection). This is how you fix projection bugs or add new views.
- Monitor **projection lag** (event store head position − projection checkpoint).

---

## 9. Domain Events vs Integration Events {#domain}

- **Domain events**: internal to a bounded context/service. They're fine-grained and can change freely with the internal model (`OrderLineAdded`).
- **Integration (public) events**: published to other services. They're **contracts**: stable, versioned, coarse-grained, documented (`OrderPlaced` v2 with the fields consumers need).
- Translate domain events → integration events at the boundary (outbox writer or an anti-corruption layer). Don't leak internal DB schemas through CDC of raw tables to other teams, because every column rename becomes a breaking change. Use the outbox with explicit event payloads.

---

## 10. Other Patterns {#other}

- **Claim check**: store a big payload in S3 and send a reference in the message.
- **Competing consumers**: N workers on one queue to scale processing.
- **Process manager**: a stateful component routing messages based on the current state (general form of a saga orchestrator).
- **Scatter-gather**: send requests to many services in parallel, then aggregate responses with a timeout (price comparison, search federation).
- **Request-reply over messaging**: correlation ID + reply-to queue (RabbitMQ RPC, NATS request-reply). Useful for async RPC, but it couples services in time again.
- **Event-carried state transfer**: fat events so consumers keep local replicas (reduces runtime coupling, at the cost of data duplication).
- **Event notification + callback**: thin event, consumer fetches current state from the source API. Simpler schemas and always fresh, but runtime coupling and load on the source.
- **Strangler fig via events**: a legacy monolith emits events (CDC), and new services build on them gradually while traffic shifts.
- **Event collaboration / choreography**, and **event-driven sagas** (above).
- **Message translator, content-based router, aggregator, splitter, resequencer**: the classic *Enterprise Integration Patterns* (Hohpe & Woolf). Worth skimming.

---

## 11. Consistency and UX {#ux}

Eventual consistency becomes a UX problem when users don't see their own actions. Techniques:
- **Read-your-writes from the write side**: after a command, return the new state in the command response, and render it directly.
- **Optimistic UI**: show the expected outcome immediately, then reconcile.
- **Version tokens**: the command returns a version, and the client's subsequent read says "at least version N". The read API waits briefly for the projection to catch up, or falls back to the write model.
- **Status-driven UI**: "Order received, processing…" with push updates (SSE/WebSocket) when it completes.
- Accept that dashboards lag by seconds, and say so.

---

## 12. Observability {#observability}

- Propagate **correlation IDs** and **W3C trace context** in message headers. OpenTelemetry messaging semantic conventions link producer and consumer spans (span links for batch consumption).
- Metrics: publish rate, consumer lag / oldest message age, processing latency, retry counts, DLQ depth, saga states (count by state, stuck sagas older than X), projection lag.
- **Event catalog**: AsyncAPI docs plus schema registry, ownership, and consumers per event.
- Tools to trace a business transaction across events (search logs by order ID or correlation ID).
- Alert on stuck sagas and DLQs, not just errors.

---

## 13. When NOT to Go Event-Driven {#when}

- A small team, a monolith, and simple CRUD: a modular monolith with DB transactions is far simpler and more consistent.
- Workflows needing strong immediate consistency across steps.
- When you can't invest in observability, idempotency, schema governance, and DLQ handling.

Start with synchronous calls inside a well-structured monolith. Introduce events where decoupling, fan-out, or load leveling clearly pay off.

---

## 14. Interview Questions {#qa}

1. What is the dual-write problem? Explain the transactional outbox and two ways to relay it.
2. Why must consumers still be idempotent with an outbox?
3. Choreography vs orchestration sagas: trade-offs, and when to use each?
4. Sagas lack isolation. How do you deal with that? What's a pivot transaction?
5. Explain CQRS. Does it require event sourcing?
6. What are the benefits and drawbacks of event sourcing? How do you handle schema changes and GDPR?
7. How do you rebuild a read model?
8. Domain events vs integration events?
9. A user updates their profile and doesn't see the change in the list view. How do you fix this in a CQRS system?
10. Design an order checkout flow across Order, Inventory, Payment, and Shipping services.
