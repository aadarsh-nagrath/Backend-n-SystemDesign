# System Design Round — Payments, Cross-Service Consistency, JVM Troubleshooting & High Concurrency

> These are the Round 3 questions from a real backend interview loop (Rounds 1–2 are in `../backend-engineer/14-java-spring-microservices-round-questions.md`). Each one is written out the way you'd actually answer it in a 45–60 minute round. *"When would you choose Kafka over synchronous REST?"* was in the same round, but it's already answered in `../backend-engineer/06-message-queues-event-driven.md` Q1, Q3 and Q7. Its tradeoffs also come up below in §1 and §2.
>
> `05-more-popular-case-studies.md` §4 has a short summary of a Stripe-like payment system. §1 here is the full-length version, focused on what this question actually asks: **high availability + no duplicate charges**.

---

## How system design questions are answered (the method)

Interviewers don't score the boxes in your diagram. They score whether you **drive the conversation**, **base decisions on numbers and requirements**, and **name tradeoffs without being asked**. Use the same structure every time (full details in `01-fundamentals-and-building-blocks.md` Q1–Q3):

| Step | What you do | Time (45 min) |
|---|---|---|
| 1. Clarify requirements | Functional (what it does) and non-functional (scale, latency, availability, consistency, compliance). Say what's **out of scope** | 4–6 min |
| 2. Estimate | QPS (average and peak), storage, bandwidth, and concurrency (Little's Law). Use them to justify later choices: "~1k TPS fits on one Postgres primary, so no sharding yet" | 3–5 min |
| 3. API | The 3–5 main endpoints/messages, including the *idempotency* and *error* behaviour | 3–5 min |
| 4. Data model | Tables/collections, keys, indexes, and **which constraints are enforced by the database itself** | 3–5 min |
| 5. High-level design | Draw the request path from start to finish. Keep it simple first | 8–10 min |
| 6. Deep dives | 2–3 hardest parts, ideally the ones the interviewer picks. Ask: *"Which part would you like me to go deeper on?"* | 15–20 min |
| 7. Failures & bottlenecks | Go through each component: "what happens when this is slow, down, or duplicated?" | 3–5 min |
| 8. Tradeoffs & wrap-up | What you'd change at 10× scale, what you gave up and why | 2–3 min |

**For "how would you troubleshoot X" questions** (like §3 here), the structure is different, but it's still a structure: **symptoms → triage/mitigation → how to tell the possible causes apart → tools → root cause → fix → prevention**. Interviewers want to hear that you **stop the bleeding first** (restart, roll back, scale out) and *then* investigate, and that you collect evidence (heap dumps, thread dumps) *before* restarting destroys it.

**Habits that score points**: say your assumptions out loud; use real numbers; for every component, say what happens if it fails; prefer boring, proven technology and explain why; and don't design for 100× scale when the requirements say 1×. Just say how you'd get there.

---

## 1. Design a highly available payment system that prevents duplicate transactions

### Step 1 — Clarify requirements

Ask first: *"Are we a merchant integrating with a payment service provider (PSP) like Stripe/Adyen/Razorpay, or are we the PSP talking to card networks?"* This answer assumes the more common case: **we're a company's internal payment service**, which takes payment requests from checkout, calls external PSPs, and records money movements.

**Functional**
- Create a payment (charge a customer for an order). Get payment status. Refund (full or partial).
- Support multiple PSPs/payment methods (card, UPI, wallet) behind one internal API.
- Notify other services (orders, notifications, accounting) when a payment changes state.
- Out of scope unless asked: storing raw card data (we tokenize: the card goes straight from the browser to the PSP, which keeps us out of most PCI DSS scope), payouts to merchants, FX, fraud scoring internals.

**Non-functional**
- **Correctness above everything**: never charge twice for one intent, never lose a payment, and every cent can be accounted for (an auditable ledger).
- **High availability**: 99.99% for creating payments (~52 min of downtime a year). Payments are revenue, so downtime costs money directly.
- **Durability**: RPO = 0 for committed payments (no acknowledged payment may be lost). RTO in minutes.
- **Latency**: p99 < 2 s end to end. Most of that is the PSP. Our own overhead should be < 100 ms.
- **Consistency**: strong (linearizable) for a single payment's state. Eventual consistency is fine for downstream views (order status, analytics).

### Step 2 — Back-of-envelope estimation

- 10 M payments/day ÷ 86,400 s ≈ **115 TPS average**. Sales events and festivals peak at ~10×, so design for **~1,500 TPS**.
- Each payment: ~1 KB payment row + ~2 ledger entries (~0.5 KB each) + idempotency record + outbox event, roughly **3 KB**. That's 10 M × 3 KB ≈ **30 GB/day, ~11 TB/year**, kept for 7–10 years for compliance.
- **Conclusion**: the write volume fits comfortably on **one well-sized relational primary** (Postgres/MySQL handle thousands of simple transactional writes per second). We pick a relational database because we *need* ACID transactions and unique constraints, not because of scale. Partition by time for retention, and only shard (by merchant or account id) at much larger scale.
- PSP calls take 300 ms–2 s. At 1,500 TPS × ~1 s that's ~1,500 calls in flight at once (Little's Law). That number drives thread/connection sizing; see §4.

### Step 3 — API

```http
POST /v1/payments
Idempotency-Key: 7f9c2e1a-...            # required; one key per logical payment attempt
{
  "order_id": "ord_123",
  "amount": 49900,                        # minor units (paise/cents); never floats
  "currency": "INR",
  "payment_method_token": "pm_tok_abc",   # token from the PSP's client SDK, not a card number
  "customer_id": "cus_42"
}
→ 201 { "payment_id": "pay_789", "status": "SUCCEEDED" | "PENDING" | "FAILED" }
→ same key + same body again       → original response replayed (same status code + body)
→ same key + DIFFERENT body        → 422 (key reused for a different request)
→ same key while first is in flight → 409, client retries later

GET  /v1/payments/{payment_id}
POST /v1/payments/{payment_id}/refunds   (also requires Idempotency-Key)

# Inbound from PSP
POST /webhooks/psp/{provider}   (signature verified; idempotent on psp_event_id)
```

The client (checkout service or mobile app) generates the key **once per checkout attempt** and reuses it on every retry of that attempt. A double click, a network timeout, or a retry by the load balancer all send the same key, so they all map to one payment.

### Step 4 — Data model (relational)

```sql
CREATE TABLE idempotency_keys (
  merchant_id     BIGINT NOT NULL,
  idem_key        TEXT   NOT NULL,
  request_hash    TEXT   NOT NULL,           -- hash of the request body, to detect key misuse
  status          TEXT   NOT NULL,           -- IN_PROGRESS | COMPLETED
  response_code   INT,
  response_body   JSONB,
  payment_id      BIGINT,
  locked_until    TIMESTAMPTZ,               -- lease, so a crashed request can be retried
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (merchant_id, idem_key)        -- the database guarantees uniqueness
);

CREATE TABLE payments (
  id              BIGINT PRIMARY KEY,
  order_id        TEXT   NOT NULL,
  amount          BIGINT NOT NULL CHECK (amount > 0),
  currency        CHAR(3) NOT NULL,
  status          TEXT   NOT NULL,           -- CREATED|PROCESSING|SUCCEEDED|FAILED|UNKNOWN|REFUNDED
  psp             TEXT,
  psp_reference   TEXT,
  version         INT    NOT NULL DEFAULT 0, -- optimistic locking
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (psp, psp_reference)
);
-- Second safety net: at most one successful payment per order.
CREATE UNIQUE INDEX one_success_per_order ON payments(order_id) WHERE status = 'SUCCEEDED';

CREATE TABLE ledger_entries (                -- append-only, double-entry
  id          BIGSERIAL PRIMARY KEY,
  payment_id  BIGINT NOT NULL REFERENCES payments(id),
  account     TEXT   NOT NULL,               -- e.g. customer_receivable, psp_clearing, revenue
  direction   TEXT   NOT NULL CHECK (direction IN ('DEBIT','CREDIT')),
  amount      BIGINT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);  -- every transaction's debits = its credits; never UPDATE/DELETE rows

CREATE TABLE outbox (                        -- events to publish, written in the same transaction
  id BIGSERIAL PRIMARY KEY, aggregate_id BIGINT, event_type TEXT,
  payload JSONB, created_at TIMESTAMPTZ DEFAULT now(), published_at TIMESTAMPTZ
);

CREATE TABLE processed_webhooks (psp TEXT, psp_event_id TEXT, PRIMARY KEY (psp, psp_event_id));
```

The important point: **duplicate prevention is enforced by database constraints** (primary keys and unique indexes), not by application `if` checks. Two concurrent requests can both pass an `if (!exists)` check. Only one of them can insert a primary key.

### Step 5 — High-level design

```
 Client/Checkout ──HTTPS──► API Gateway / LB (auth, rate limit, TLS)
                                   │
                                   ▼
                       ┌─────────────────────────┐   stateless, N instances across 3 AZs
                       │     Payment Service     │
                       │  1. idempotency check   │
                       │  2. create payment row  │──────► Postgres primary (AZ-a)
                       │  3. call PSP adapter    │          │ synchronous replica (AZ-b)  ← RPO 0
                       │  4. update state +      │          │ async replica (AZ-c / other region)
                       │     ledger + outbox     │
                       └───────────┬─────────────┘
                                   │ PSP adapter (timeouts, breaker per PSP, PSP idempotency key)
                     ┌─────────────┼──────────────┐
                     ▼             ▼              ▼
                  PSP A         PSP B          PSP C   ──webhooks──► Webhook Handler ──► same DB
                                   
 Outbox relay (or Debezium CDC) ──► Kafka "payment-events" ──► Order svc, Notifications, Accounting
 Reconciliation job (hourly/daily): our ledger vs PSP settlement reports ──► mismatch queue/alerts
 Status poller: payments in PROCESSING/UNKNOWN past N seconds ──► query the PSP for the real status
```

**Happy path:**
1. Insert into `idempotency_keys` as `IN_PROGRESS`, *or* find the existing row and replay its response.
2. Insert the `payments` row as `PROCESSING`. Commit. (A short transaction: **never hold a DB transaction open during the PSP call**.)
3. Call the PSP with **our own payment id as the PSP's idempotency key**, with timeouts.
4. In one transaction: update the payment to `SUCCEEDED` with a conditional update, insert the ledger entries, insert an outbox event, and mark the idempotency key `COMPLETED` with the response. Commit.
5. Return the response. The outbox relay then publishes `PaymentSucceeded` to Kafka. The order service marks the order paid. Notifications sends a receipt.

### Step 6 — Deep dives

#### 6a. Preventing duplicates. Duplicates can enter at five places, so there's a defense at each one.

| Where a duplicate comes from | Defense |
|---|---|
| User double-clicks, mobile app retries, gateway/LB retries a timed-out request | **Client idempotency key** + `PRIMARY KEY (merchant_id, idem_key)` |
| Two concurrent requests with the same key arrive at once | The insert race is settled by the database: exactly one `INSERT … ON CONFLICT DO NOTHING` wins. The loser sees `IN_PROGRESS` and gets 409 |
| Our service times out talking to the PSP and retries | Send the **PSP's own idempotency key** (= our `payment_id`). The PSP returns the original charge instead of charging again |
| A new idempotency key for the same order (buggy client, a second browser tab) | `UNIQUE (order_id) WHERE status='SUCCEEDED'`, plus an "active payment for this order?" check that returns the existing one |
| Kafka / webhook redelivery (at-least-once) | Consumers are **idempotent**: `processed_webhooks` / an inbox table with a unique event id, and state transitions that are conditional |

**The idempotency-key algorithm:**
```sql
-- Step A: claim the key (atomic; settles concurrent requests)
INSERT INTO idempotency_keys (merchant_id, idem_key, request_hash, status, locked_until)
VALUES ($m, $k, $hash, 'IN_PROGRESS', now() + interval '30 seconds')
ON CONFLICT (merchant_id, idem_key) DO NOTHING
RETURNING *;
```
- **Row returned** → we own the key. Do the work.
- **No row returned** → the key exists. `SELECT` it:
  - `request_hash` differs → **422** (the key was reused for a different request).
  - `COMPLETED` → **replay** the stored `response_code` and `response_body`. Same answer, no new charge.
  - `IN_PROGRESS` and the lease (`locked_until`) hasn't expired → **409 Conflict** ("in progress, retry later").
  - `IN_PROGRESS` and the lease has expired (the owner crashed) → take over the lease with a conditional `UPDATE … WHERE locked_until < now()`. Then **resume from the payment's recorded state**: if a PSP call may already have been made, *query the PSP* before charging.
- Keep keys for at least as long as clients may retry (24 h–7 days), then delete them in a TTL job. Redis is fine as a fast-path cache in front, but **the database is the source of truth** for money.

**Conditional state transitions** (a state machine enforced in SQL):
```sql
UPDATE payments SET status = 'SUCCEEDED', psp_reference = $ref, version = version + 1, updated_at = now()
WHERE id = $id AND status IN ('PROCESSING', 'UNKNOWN');   -- 0 rows updated → already handled, do nothing
```
Allowed transitions: `CREATED → PROCESSING → SUCCEEDED | FAILED | UNKNOWN`, `UNKNOWN → SUCCEEDED | FAILED`, `SUCCEEDED → REFUNDED/PARTIALLY_REFUNDED`. A late webhook can never move `SUCCEEDED` back to `FAILED`.

#### 6b. The hardest case: the PSP call times out (the outcome is unknown)

The request reached the PSP, but our connection died before the response came back. The customer **may or may not have been charged**. This is the core problem in payments. Interviewers ask about it every time.

- **Never treat a timeout as a failure** ("it failed, let the user pay again"). That's how double charges happen.
- Mark the payment **`UNKNOWN`** and tell the client `PENDING` ("we're confirming your payment").
- Find out the real outcome from three independent sources, all writing through the same conditional update:
  1. **Retry the PSP call with the same PSP idempotency key**. It's safe because the PSP returns the original result.
  2. **The PSP's webhook** (`charge.succeeded`), verified by signature and deduplicated by event id.
  3. **A status poller** that queries `GET /charges?idempotency_key=…` for payments stuck in `UNKNOWN` for more than N seconds, with backoff, escalating to manual review after a few hours.
- **Last line of defense: daily reconciliation** against the PSP's settlement file. A charge that exists at the PSP but not in our ledger (or the reverse) creates an alert and an automatic refund/void or a manual review.

#### 6c. High availability

- **Stateless Payment Service**, ≥3 instances spread across 3 availability zones behind the LB, with autoscaling, rolling/canary deploys, and readiness probes. Losing an instance loses nothing, because all state is in the database, and in-flight requests are retried by the client with the same key.
- **Database**: Postgres with a **synchronous standby in another AZ** (`synchronous_commit = on`, quorum of 1). A commit is acknowledged only after it's on two machines, so **RPO = 0**. Automated failover (Patroni, or a managed service like RDS Multi-AZ / Aurora) in about 30–60 s. Clients reconnect through a stable endpoint, and the idempotency keys make retries during failover safe.
- **Multi-region**: for payments, run **active-passive** for the source of truth (the primary region writes; the second region holds an async replica and can be promoted in a regional disaster, with a small accepted RPO and a reconciliation pass afterwards). Active-active writes to money records means conflict resolution on balances, which you should avoid unless the data is partitioned by region/merchant so each record has exactly one home region. Read paths (status pages) can be served from any region.
- **PSP redundancy**: a circuit breaker per PSP. When PSP A's error rate spikes, route *new* payments to PSP B. **Never fail over a payment that's already in flight**: its outcome at PSP A is unknown and must be resolved there first (6b).
- **Load spikes**: rate limiting per merchant at the gateway; bulkheads so refunds, reporting, and webhooks can't starve payment creation; and a pool sized for PSP latency (§4).
- **Graceful degradation**: if Kafka is down, payments still succeed, because events wait in the outbox table and are published later. If the reconciliation job or analytics are down, nothing customer-facing is affected.

#### 6d. Publishing events reliably (why the outbox exists)

"Commit the payment to the DB, then publish to Kafka" is a **dual write**. If the process crashes between the two, the order is never marked paid. Instead, write the event to the `outbox` table **in the same DB transaction** as the state change. A relay (polling `WHERE published_at IS NULL`, or Debezium reading the WAL) publishes it to Kafka with at-least-once delivery, partitioned by `payment_id` to keep order. Consumers deduplicate by event id. More detail in §2.

### Step 7 — Bottlenecks & failure modes

| Failure | What happens | Mitigation |
|---|---|---|
| Client retries after a timeout | Same idempotency key | Response replayed, no second charge |
| Service crashes after the PSP charged but before our DB update | Payment stuck in `PROCESSING` | Poller/webhook resolves it via the PSP; the key's lease expires and a retry resumes safely |
| PSP slow or down | Threads pile up, timeouts | Timeouts, breaker, route new payments to a backup PSP, return `PENDING` |
| DB primary fails | ~30–60 s of write unavailability | Synchronous standby promoted, no data lost, clients retry with the same keys |
| Kafka down | Events not delivered | Outbox buffers them; downstream catches up later |
| Duplicate/out-of-order webhooks | Possible double processing | Dedup table + conditional state machine updates |
| Hot merchant (flash sale) | Contention on that merchant's rows | Per-merchant rate limits, no shared "balance" row updated on every payment (append ledger entries instead) |
| Bug that miscalculates amounts | Silent money errors | Double-entry invariant checks (debits = credits), reconciliation, alerts |

### Step 8 — Tradeoffs to call out

- **Relational DB vs NoSQL**: we choose ACID and unique constraints over easy horizontal scaling. At our scale that's free. At much larger scale, shard by merchant/account, or use a distributed SQL database (Spanner/CockroachDB/Yugabyte) to keep transactions.
- **Synchronous standby**: adds a few ms to every commit in exchange for RPO 0. That's correct for money, and wrong for, say, click analytics.
- **Synchronous vs asynchronous API**: we authorize synchronously (the user is waiting) and do settlement, notifications, and accounting asynchronously. The honest third answer is `PENDING`, which needs UI support.
- **Exactly-once is not available end to end.** We get "effectively once" from at-least-once delivery + idempotency + reconciliation. Say this explicitly.

### Likely follow-ups

- *How long do you keep idempotency keys?* As long as clients retry, plus a margin (Stripe uses 24 h). Payment uniqueness per order is also protected by the partial unique index, so expired keys don't reopen the hole.
- *How do refunds avoid duplicates?* The same mechanism: an idempotency key per refund request, plus a check that the total refunded ≤ the captured amount, inside a transaction with `SELECT … FOR UPDATE` on the payment row.
- *Why not 2PC between the payment DB and the PSP?* The PSP is an external HTTP API and can't take part in a distributed transaction. That's why state machines, idempotency, and reconciliation are used instead.
- *How do you test this?* Fault injection (kill the service between the PSP call and the DB commit), PSP sandbox timeouts, chaos testing DB failover, and property tests for the ledger invariants.

---

## 2. How would you maintain data consistency across microservices without distributed transactions?

### Step 1 — Clarify the problem

Each service owns its own database (`../backend-engineer/07-microservices-architecture.md` Q3). One business operation, for example "place order", touches several of them: Order (create), Payment (charge), Inventory (reserve), Shipping (schedule). We want *all or nothing* behaviour **without** a distributed transaction.

**Why not 2PC/XA?** It's a blocking protocol (if the coordinator crashes, participants hold locks indefinitely), it ties every service's availability together (the overall availability is the product of all participants' availability), it adds latency through lock holding, and most modern components don't support XA at all (Kafka, most NoSQL stores, third-party HTTP APIs). In CAP terms, 2PC chooses consistency over availability for the whole flow.

**What we accept instead**: **eventual consistency** with guaranteed convergence. Every step either completes or is compensated, nothing is lost, and nothing is applied twice. Clarify with the interviewer which invariants must *never* be violated, even briefly (e.g. "never ship an unpaid order"), and which can be temporarily inconsistent (e.g. "order shows PENDING for 2 seconds").

### Step 2 — The building blocks (use them together)

**1. Transactional outbox: fixes the dual-write problem inside one service.**
"Update my DB *and* publish an event" can't be atomic across a DB and a broker. So write the business change **and** an `outbox` row in **one local ACID transaction**. A separate relay publishes the outbox rows to Kafka.
```sql
BEGIN;
  INSERT INTO orders (id, status, ...) VALUES ('o1', 'PENDING', ...);
  INSERT INTO outbox (id, aggregate_id, type, payload) VALUES (gen_random_uuid(), 'o1', 'OrderCreated', '{...}');
COMMIT;
```
The relay is either (a) **a poller**: `SELECT … WHERE published_at IS NULL ORDER BY id LIMIT 500 FOR UPDATE SKIP LOCKED`, publish, then mark them published; or (b) **CDC** with Debezium reading the WAL/binlog and its outbox event router, which adds lower latency and no polling load. Either way delivery is **at-least-once**, which leads to the next block.

**2. Idempotent consumers (the inbox pattern).** Every event carries a unique `event_id`. The consumer, in **one local transaction**, inserts `event_id` into a `processed_events` table (primary key) *and* applies the change. A duplicate violates the key, so it's skipped. Also make the operations idempotent by nature where possible (`SET status='PAID'` rather than `balance = balance + x`).

**3. Saga: a multi-step business transaction made of local transactions, with compensations.** Each step is a local ACID transaction that publishes an event. If a later step fails, run **compensating transactions** for the completed steps, in reverse order. Compensations are *semantic* undos (refund, release the reservation), not database rollbacks.

```
Happy path:   Order: create(PENDING) → Payment: charge → Inventory: reserve → Order: CONFIRMED
Inventory fails: Inventory: OutOfStock → Payment: refund (compensate) → Order: CANCELLED (compensate)
```
- **Choreography**: services react to each other's events. There's no central coordinator, and the coupling is low, but the flow is hard to see and debug once you have more than about 4 steps.
- **Orchestration**: an orchestrator (a service with a persisted state machine, or a workflow engine such as **Temporal**/Camunda/AWS Step Functions) sends commands and tracks progress. The flow is explicit, timeouts and retries live in one place, and it's easier to monitor. **Recommended for flows that involve money.** (Details in `../backend-engineer/06-message-queues-event-driven.md` Q10–Q11.)
- **Order the steps carefully**: put steps that are *hard to compensate* (sending an email, shipping goods, calling an external party) **last**, after a "pivot" step (usually payment) after which the saga only moves forward and retries until it succeeds.

**4. Handle the lack of isolation.** Sagas are ACD without the I: other transactions can see the intermediate states. Mitigations:
- **Semantic locks / pending states**: the order is `PENDING` and inventory is `RESERVED` (not decremented), so other flows know the data isn't final.
- **Commutative updates** where possible.
- **Re-read and re-validate** before the pivot step (re-check the price).
- **Reservations with expiry** (hold seats for 10 minutes, as in the Ticketmaster design in `05` §3).

**5. Ordering.** Partition Kafka topics by aggregate id (`order_id`) so that all events for one order are consumed in order. Include a version/sequence number in the event, and have consumers ignore anything older than what they've already applied.

**6. Reconciliation (the safety net).** A periodic job compares sources of truth (orders marked PAID vs payments SUCCEEDED, reservations vs orders) and repairs or alerts on mismatches. Every real system has one, because bugs, bad deploys, and manual fixes happen.

**7. Reading across services without consistency surprises.** Query the owning service, or keep a **local read model** fed by events (CQRS). Accept that it lags slightly, and show `PENDING` honestly in the UI.

### Step 3 — Walkthrough to tell in the interview (orchestrated order saga)

1. Client `POST /orders` (idempotency key). The Order service writes order `PENDING` + outbox `OrderCreated` in one transaction.
2. The orchestrator (Temporal workflow or a saga table) receives it and sends `ChargePayment(order_id, amount, idem_key=order_id)`.
3. Payment charges (§1), and writes `PaymentSucceeded` to its outbox.
4. The orchestrator sends `ReserveStock`. Inventory checks stock, reserves it, and emits `StockReserved`, or `OutOfStock` if there isn't any.
5. **Success**: the orchestrator sends `ConfirmOrder`, and the order becomes `CONFIRMED`. Shipping is triggered (a forward-only step).
6. **Failure at step 4**: the orchestrator sends `RefundPayment` (idempotent, keyed on order_id) and then `CancelOrder`. Every command is retried with backoff until it's acknowledged. Compensations must eventually succeed, and if they keep failing, they're escalated to a human with alerts.
7. **Timeouts**: if no reply arrives within T, the orchestrator queries the participant's state (rather than assuming failure) and continues or compensates.

### Step 4 — Failure analysis (go through each crash point)

| Crash point | Why it's still consistent |
|---|---|
| Order service crashes after commit, before publishing | The outbox row is durable. The relay publishes it after restart |
| Relay publishes, crashes before marking `published_at` | The event is published again; the consumer's inbox deduplicates it |
| Payment consumer crashes after charging, before acknowledging | Redelivered; the payment idempotency key (order_id) prevents a second charge |
| Orchestrator crashes mid-saga | Its state is persisted; it resumes from the last recorded step |
| Compensation fails (the refund API is down) | Retried with backoff; alert + manual queue after N attempts; reconciliation catches anything left over |
| Events arrive out of order | Partitioned by order_id + version checks |

### Tradeoffs to call out
- Eventual consistency means intermediate states are visible. That's a product decision too (the UI shows "processing").
- Sagas add code: every step needs a compensation and idempotency. They're worth it for cross-service flows. **If two pieces of data always have to change together atomically, maybe they belong in the same service/DB.** Rethinking service boundaries is often the real fix.
- Outbox polling vs CDC: polling is simple but adds latency and DB load. CDC has lower latency but is another system to run (Kafka Connect/Debezium).

---

## 3. How would you troubleshoot memory leaks and thread pool exhaustion?

(A troubleshooting question: answer with *symptoms → mitigate → diagnose → fix → prevent*. JVM-focused, since the round was Java. The concepts are the same for other runtimes.)

### Part A — Memory leaks

**Symptoms**
- Heap usage *after each full GC* (the "floor" of the sawtooth graph) **keeps rising** over hours or days. That rising floor is the defining sign: a high but stable heap isn't a leak.
- GC runs more and more often, with longer pauses. CPU goes to GC (GC overhead > 10–20%). Latency gets worse over time, until `java.lang.OutOfMemoryError: Java heap space` / `GC overhead limit exceeded` happens, or Kubernetes **OOMKills** the pod (exit code 137).
- Restarting "fixes" it for a while. That pattern is itself a strong clue.

**Mitigate first**: restart or roll back to stop the impact, but **capture evidence first**. Run with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps` (on a persistent volume) and GC logging (`-Xlog:gc*:file=gc.log`) enabled ahead of time, so the evidence survives a crash.

**Diagnose:**
1. **Confirm it's a leak, and work out which kind of memory.** In Kubernetes, compare the container's RSS against the heap size:
   - **Heap** leak: Micrometer `jvm.memory.used{area=heap}` / `jstat -gcutil <pid> 5s` shows old gen climbing after each GC.
   - **Off-heap / native**: RSS grows while the heap is flat. Suspects are direct `ByteBuffer`s (Netty), thread stacks (too many threads!), Metaspace (classloader leaks from hot redeploys or dynamic proxies), JNI/native libraries, and glibc malloc arenas. Use **Native Memory Tracking** (`-XX:NativeMemoryTracking=summary`, `jcmd <pid> VM.native_memory summary`).
   - **Metaspace**: `OutOfMemoryError: Metaspace`, which means classes are being generated without limit.
2. **Look at what's on the heap.** `jcmd <pid> GC.class_histogram` taken twice, minutes apart. The classes whose instance counts keep growing are the suspects. Low cost and safe in production.
3. **Heap dump** (`jcmd <pid> GC.heap_dump /dumps/h.hprof`). It pauses the JVM, and the file is roughly the size of the heap, so take it on an instance that's out of rotation. Open it in **Eclipse MAT** (or VisualVM/IntelliJ). Use the **Leak Suspects** report and the **dominator tree** to find the object holding the most retained memory, then follow **"path to GC roots"** to see *who is keeping it alive*. That reference chain is the bug.
4. **Profile allocations** if it's not obvious: JFR (`jcmd <pid> JFR.start duration=120s filename=rec.jfr`) or async-profiler in allocation mode.

**Common causes (name a few):**
- Unbounded **caches/static maps** (`static Map<String, Object> cache` that never evicts). Fix: Caffeine with `maximumSize`/`expireAfterWrite`.
- **ThreadLocals in pooled threads** that are never `remove()`d. A request's data stays alive in Tomcat worker threads forever.
- **Listeners/callbacks** registered and never unregistered.
- Unclosed resources: streams, `ResultSet`s, HTTP responses (their connections leak too).
- A huge Hibernate first-level cache from loading millions of entities in one transaction (use paging, `clear()`, or `StatelessSession`).
- `String.intern()`/classloader leaks, and unbounded in-memory queues (a producer faster than its consumer, which is a backpressure bug).
- High-cardinality metrics tags (a user id used as a tag creates one time series per user).

**Fix and prevent**: fix the reference chain. Add a memory-growth alert (on old gen after GC, not on raw usage), run soak tests (a load test for 12–24 h while watching the post-GC floor), set container limits with headroom for non-heap memory (`-XX:MaxRAMPercentage=70–75`), and keep heap dump on OOM turned on in production.

### Part B — Thread pool exhaustion

**Symptoms**
- Latency shoots up for **all** endpoints, including trivial ones like `/health`, while **CPU is low**. Threads are *waiting*, not working. That combination is the signature.
- Tomcat busy threads = `server.tomcat.threads.max` (default **200**), the accept queue fills up, then connections are refused or the LB returns 502/504. Liveness probes time out, so Kubernetes restarts pods, which makes things worse.
- For executor pools: `RejectedExecutionException`, a growing `executor.queued` metric, or memory growth from a huge unbounded queue (`Executors.newFixedThreadPool` uses an **unbounded** `LinkedBlockingQueue`).
- HikariCP: `Connection is not available, request timed out after 30000ms`, and `hikaricp.connections.pending` > 0.

**Mitigate**: scale out, shed load (rate limit), open breakers to the slow dependency. **Take thread dumps first** (below), then restart.

**Diagnose with thread dumps.** This is the main tool. Take **3 dumps about 5–10 s apart** (`jcmd <pid> Thread.print > td1.txt`, or `jstack`, or `kill -3`), then group threads by stack trace (fastthread.io, or just `grep`). Threads stuck in the **same frame across all dumps** are the bottleneck. Read the top frames:

| Where most threads are stuck | Meaning | Fix |
|---|---|---|
| `SocketInputStream.socketRead0` / `NioSocketImpl.read` inside an HTTP client | Waiting on a slow downstream with **no or too long a read timeout** | Timeouts, circuit breaker, bulkhead per dependency |
| `HikariPool.getConnection` (WAITING/TIMED_WAITING) | **DB connection pool** is the bottleneck: too small, slow queries, or connections **leaked** / held across remote calls inside `@Transactional` | Find long transactions; `leakDetectionThreshold`; keep transactions short; fix slow queries |
| `BLOCKED` on `waiting to lock <0x…>`, all on one monitor | **Lock contention**: a single `synchronized` method or shared resource | Shrink the critical section, use concurrent structures |
| `Unsafe.park` in `CompletableFuture.get` / `Future.get` **inside the same pool** | **Pool-induced deadlock**: tasks in pool X block waiting on subtasks queued to pool X, so no thread is free to run the subtasks | Separate pools per stage, or non-blocking composition (`thenCompose`) |
| Deadlock section at the end of the dump ("Found one Java-level deadlock") | A real deadlock | Consistent lock ordering, `tryLock` with timeout |
| Many `RUNNABLE` threads in your own code, CPU high | A real CPU-bound overload or an infinite loop | Profile (async-profiler), scale out |

**Common root causes:**
- **No timeouts** on HTTP/DB calls. This is the most common one by far.
- Calling blocking code from a small pool: blocking JDBC/HTTP on a Netty/WebFlux event loop, or on `ForkJoinPool.commonPool()` (`parallelStream`, default `CompletableFuture.supplyAsync`).
- Thread pool sizes that don't fit together: 200 Tomcat threads but 10 DB connections, so 190 threads queue for connections.
- Retry storms that multiply the load on an already slow dependency.
- `@Async` with the default executor settings, or unbounded queues that hide the problem until memory runs out.

**Fix and prevent:**
- Timeouts everywhere; circuit breakers + bulkheads per dependency (`../backend-engineer/14-java-spring-microservices-round-questions.md` Q8).
- **Bounded** pools with **bounded** queues and an explicit rejection policy (`CallerRunsPolicy` for natural backpressure, or fail fast with 503).
- Size pools using Little's Law (§4). Separate pools for separate kinds of work.
- Java 21 **virtual threads** (`spring.threads.virtual.enabled=true`) remove the request thread limit for blocking I/O, but **not** downstream limits: you still have only N DB connections. Watch out for *pinning* (`synchronized` blocks around blocking I/O on JDK < 24).
- Monitor: `tomcat.threads.busy`, `executor.active` / `executor.queued`, `hikaricp.connections.pending`/`.usage`, and alert before you hit 100%.

---

## 4. How would you design a backend system to handle thousands of concurrent requests?

### Step 1 — Clarify requirements

"Thousands of concurrent requests" is ambiguous, so pin it down first:
- **Concurrent (in flight at once) or per second?** They're related by **Little's Law**: `concurrency = throughput × latency`. 10,000 requests in flight at 200 ms average latency = **50,000 RPS**. The same 50,000 RPS at 20 ms latency means only 1,000 in flight. **Reducing latency reduces the concurrency you have to handle.**
- What kind of work: read-heavy (product pages: very cacheable), write-heavy (orders, events), or long-lived connections (WebSockets, streaming)?
- Latency SLO (e.g. p99 < 300 ms), availability (99.9%+), and whether the traffic is spiky (ticket sales: 50× in 1 minute) or steady.
- Let's assume: **10k concurrent, ~50k RPS at peak, 80% reads, p99 < 300 ms, spiky traffic.**

### Step 2 — Estimate

- If one instance with a thread-per-request blocking model (200 Tomcat threads) takes 200 ms per request, it handles about 200 / 0.2 = **1,000 RPS**. So 50k RPS needs about 50 instances, or a model that doesn't block a thread per request. The bigger issue is **the DB**:
- If every request ran one 5 ms query, 50k RPS = 250 connections busy all the time. Postgres gets slower with many hundreds of active connections. 50 instances × 20 pool connections = 1,000 connections, which will hurt it. **So the design has to keep most reads away from the primary database.** That's the main point of this question.

### Step 3 — High-level design

```
 Clients ──► CDN (static + cacheable GETs) ──► DNS / Global LB
                                                   │
                                     L7 Load Balancer / API Gateway
                           (TLS termination, rate limiting, auth, keep-alive)
                                                   │
                    ┌──────────────────────────────┼─────────────────────────────┐
                    ▼                              ▼                             ▼
            App instance 1..N     (stateless, autoscaled on CPU/RPS/latency, 3 AZs)
                    │ local in-process cache (Caffeine, seconds TTL for hot keys)
                    ├────► Redis cluster (shared cache, sessions, rate-limit counters)
                    ├────► PgBouncer ──► Postgres primary (writes)
                    │                └─► read replicas (reads that can be slightly stale)
                    └────► Kafka/SQS (async work: emails, analytics, heavy processing)
                                       └─► worker pool (scaled on queue lag)
```

### Step 4 — Deep dives

**4a. Stateless horizontal scaling.** No session state in instance memory (use JWTs or Redis sessions), so any instance can serve any request. Autoscale on RPS per instance or latency (CPU is a poor signal for I/O-bound services). Spread instances across AZs. For predictable spikes, scale up ahead of time, because autoscaling reacts in minutes and spikes take seconds.

**4b. Concurrency model inside each instance.** This is where Java interviewers dig in:
| Model | How it handles 10k in flight | Tradeoffs |
|---|---|---|
| Thread-per-request (Tomcat, platform threads) | Needs ~10k threads (≈1 MB stack each, plus context switching) or many instances | Simple and debuggable; wasteful for I/O-heavy work |
| **Virtual threads** (Java 21+, `spring.threads.virtual.enabled=true`) | Millions of cheap threads; blocking I/O parks the virtual thread, not the OS thread | Same simple blocking code; watch pinning and **downstream** limits (still N DB connections, so use a semaphore/bulkhead) |
| Reactive / event loop (WebFlux + Netty, Node.js) | A few threads, non-blocking I/O | Very efficient; harder to write and debug; **one blocking call stalls the event loop** |

For a new Java service, the modern default is **blocking code on virtual threads** plus explicit concurrency limits on scarce resources.

**4c. Caching. This is what actually makes the high concurrency affordable.** With 80% reads:
- CDN for static and public GET responses (`Cache-Control`, `stale-while-revalidate`).
- Redis with cache-aside for hot entities. A 95% hit rate turns 40k read RPS into 2k DB reads.
- A small in-process cache with a short TTL for the hottest keys, to avoid a Redis *hot key* problem.
- Guard against a **cache stampede** when a hot key expires: request coalescing (single-flight), early probabilistic refresh, TTL jitter. (`../backend-engineer/04-caching-strategies.md`.)

**4d. Database protection.**
- **Connection pooling**, sized deliberately. A good starting point is `connections ≈ (cores × 2) + effective spindles` per DB node (the HikariCP guidance), *not* "as many as there are threads". More connections than the DB can actually run in parallel only adds contention. Put **PgBouncer** (transaction mode) between many app instances and Postgres.
- Read replicas for reads that can be slightly stale. Route reads that must see the user's own writes to the primary (read-your-writes).
- Query hygiene: indexes, no N+1 queries, pagination, short transactions.
- When write volume outgrows one primary, **shard** by tenant/user id (`03-data-storage-design.md`).

**4e. Async processing for anything slow.** Accept the request, validate, write the minimum synchronously, return `202 Accepted` + a job id, and put the rest on Kafka/SQS. The queue absorbs spikes (load levelling), and workers scale on queue lag. Use this for email, PDF generation, image processing, fan-out, and third-party calls.

**4f. Protect the system when demand exceeds capacity.** Design for overload, because it will happen:
- **Rate limiting** per user/API key at the gateway (token bucket in Redis) → 429 + `Retry-After`.
- **Bounded queues + load shedding**: if the in-flight count goes over the limit, return 503 immediately instead of queueing (a request that waits 30 s will be retried by the client anyway, which doubles the load). Prioritize: shed analytics before checkout.
- **Timeouts, circuit breakers, bulkheads** on every dependency (`../backend-engineer/14-java-spring-microservices-round-questions.md` Q8).
- **Backpressure** end to end: bounded executors, reactive streams' `request(n)`, Kafka consumers that pull at their own pace.
- For extreme spikes (ticket drops, flash sales): a **virtual waiting room / queue** in front, letting users in at the rate the backend can handle.

**4g. Efficiency per request** (lower latency also means lower concurrency, by Little's Law): HTTP keep-alive and connection reuse (HTTP/2 to backends), compact payloads (gzip, or gRPC between services), run independent downstream calls **in parallel** instead of one after another, avoid synchronous logging on hot paths, and tune GC (G1/ZGC) for low pauses.

### Step 5 — Verify it works
Load test before launch (k6, Gatling, JMeter) with realistic traffic mixes and a *spike* profile, not just a steady ramp. Find the first bottleneck (usually DB connections or a downstream), fix it, and repeat. Watch the four golden signals: latency (p99), traffic, errors, and saturation (threads, pool usage, queue depth). Run chaos tests: kill an AZ, slow down Redis.

### Step 6 — Bottlenecks & tradeoffs

| Bottleneck | Symptom | Fix |
|---|---|---|
| DB connections | `hikaricp.connections.pending`, p99 rises | Cache, replicas, PgBouncer, shorter transactions |
| Hot cache key | One Redis shard at 100% | Local cache, key replication/splitting |
| Downstream dependency | Threads blocked on socket reads | Timeouts, breaker, bulkhead, async |
| Request threads | Busy threads = max, low CPU | Virtual threads, or fix what they're waiting on |
| Spike faster than autoscaling | Errors during the first minutes | Pre-scale, waiting room, load shedding |

**Tradeoffs to say out loud**: caching and replicas trade **freshness** for scale; async processing trades **immediacy** for resilience (the UI must handle "processing"); reactive code trades **simplicity** for efficiency (virtual threads mostly remove this tradeoff for I/O-bound services); and load shedding means deliberately failing some requests so the rest succeed, which is the right call when the alternative is a total outage.

### Likely follow-ups
- *How many instances do you need?* Work it out with Little's Law from the measured per-instance throughput in the load test, plus N+1 headroom per AZ.
- *What happens when Redis goes down?* Fall back to the DB with a concurrency limit (otherwise the sudden DB load causes an outage), serve stale data from local caches, and degrade non-essential features.
- *WebSockets with 1M connections?* That's a different problem: event-loop servers (Netty), sticky routing, a pub/sub fan-out layer. See the chat design in `04-case-studies-walkthroughs.md`.
