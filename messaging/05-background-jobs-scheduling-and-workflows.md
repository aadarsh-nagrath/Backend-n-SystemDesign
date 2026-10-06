# Background Jobs, Scheduling, and Durable Workflows

## Table of Contents
1. [What Belongs in the Background](#what)
2. [Job Queue Architecture](#architecture)
3. [Popular Job Frameworks by Language](#frameworks)
4. [Designing Robust Jobs](#robust)
5. [Retries, Timeouts, and Failure Handling](#retries)
6. [Priorities, Fairness, and Rate Limits](#fairness)
7. [Scheduled and Recurring Jobs (Cron) in a Distributed World](#cron)
8. [Delayed Jobs and Timers](#delayed)
9. [Durable Execution / Workflow Engines (Temporal, Step Functions)](#workflows)
10. [Batch Processing Patterns](#batch)
11. [Observability and Operations](#ops)
12. [Interview Questions](#qa)

---

## 1. What Belongs in the Background {#what}

Move work out of the request/response path when it:
- Is **slow** (emails, PDFs, image/video processing, ML inference, report generation, exports).
- Calls **unreliable third parties** (payment webhooks, CRM syncs) and needs retries.
- Is **not needed for the response** (analytics, notifications, search indexing, cache warming).
- Is **bursty** and needs smoothing (bulk imports).
- Must run **on a schedule** (nightly billing, cleanup, digests).

Rule: the HTTP request should do the minimum to accept the work durably (persist + enqueue, ideally via outbox), return quickly (`202 Accepted` or the created resource), and let workers do the rest.

---

## 2. Job Queue Architecture {#architecture}

```
API server ──enqueue(job)──► Broker/store (Redis / Postgres / SQS / RabbitMQ)
                                   │
                    ┌──────────────┼──────────────┐
                Worker 1       Worker 2        Worker N   (concurrency per worker)
                    │
                Side effects (DB, email provider, S3, 3rd-party APIs)
```
Components:
- **Job payload**: type + arguments (IDs, not whole objects; fetch fresh data in the worker), job ID, attempt count, scheduled time, priority, queue name.
- **Queue store**: Redis lists/sorted sets/streams (fast, memory-bound), the DB (transactional enqueue, simpler infra), or SQS/RabbitMQ (managed, durable).
- **Workers**: processes or threads pulling jobs, with concurrency limits.
- **Scheduler**: moves delayed and retry jobs into ready queues and handles cron.
- **Dashboard**: queue depths, failures, retries, manual retry/delete (Sidekiq Web, Bull Board, Flower, Hangfire dashboard).

Enqueue **after commit** (or via outbox), never before the DB transaction commits. Otherwise the worker may run before the data exists (a race), or run for data that was rolled back. Frameworks offer hooks: Django `transaction.on_commit`, Rails `after_commit`, or DB-backed queues where the enqueue is in the same transaction.

---

## 3. Frameworks {#frameworks}

| Language | Frameworks |
|---|---|
| Ruby | **Sidekiq** (Redis; very fast, threads), GoodJob / Solid Queue (Postgres; Rails 8 default), Resque, Delayed Job |
| Python | **Celery** (Redis/RabbitMQ), **RQ**, Dramatiq, Huey, arq (asyncio), Procrastinate (PG), Taskiq |
| Node.js | **BullMQ** (Redis), Agenda (Mongo), Graphile Worker / pg-boss (Postgres), Bee-Queue |
| Java/Kotlin | Spring `@Async`/Batch, **JobRunr**, Quartz (scheduling), db-scheduler, Kafka/RabbitMQ consumers |
| Go | **River** (Postgres), Asynq (Redis), Machinery, Temporal SDK |
| .NET | **Hangfire**, Quartz.NET, MassTransit, Wolverine |
| Elixir | **Oban** (Postgres), built-in OTP supervision |
| Cloud | SQS + Lambda, Cloud Tasks (GCP), Azure Queue Storage + Functions, Cloudflare Queues |

---

## 4. Designing Robust Jobs {#robust}

1. **Idempotent**: jobs *will* run more than once (worker crash after the side effect, before the ack; visibility timeouts; manual retries). Use an idempotency key per side effect, check state before acting (`if invoice.sent_at: return`), and use upserts.
2. **Small, ID-based arguments**: `SendInvoiceEmail(invoice_id=42)`, not the serialized invoice (it'd be stale and large, and might contain PII sitting in Redis).
3. **Re-read state** at execution time and handle "entity no longer exists or no longer relevant" gracefully (a deleted user means skip, don't fail forever).
4. **Short and chunked**: split big work into many small jobs (fan-out: one job per 1,000 records), so retries redo little work and deploys don't kill hour-long jobs. Use batch/parent-child features (Sidekiq Batches, Celery chords/groups, BullMQ flows) to know when all children finish.
5. **Checkpointing** for long jobs: persist progress (cursor/last processed ID) so a restart resumes.
6. **Deterministic, explicit failures**: raise clear exceptions, classify retryable vs non-retryable.
7. **Timeouts on every external call** inside jobs.
8. **Graceful shutdown**: on SIGTERM, stop fetching new jobs, finish or requeue in-flight jobs within the grace period. Long jobs should check a cancellation flag.
9. **Versioning**: deploys may run new code on old jobs or vice versa (jobs enqueued by old code, processed by new code). Keep job argument schemas backward compatible, and handle old job names during transitions.
10. **Security**: don't put secrets in job payloads, and validate payloads (a queue is an input surface).

---

## 5. Retries and Failure Handling {#retries}

- **Exponential backoff with jitter**: e.g., Sidekiq's default is ~25 retries over ~21 days (`(count^4) + 15 + rand(10)*(count+1)` seconds).
- Cap attempts. After exhaustion, move to a **dead set / DLQ**, alert, and keep tools to inspect and retry.
- Distinguish:
  - **Retryable**: network timeouts, 5xx, 429 (respect `Retry-After`), deadlocks, lock timeouts.
  - **Non-retryable**: validation errors, 4xx (except 408/429), missing records, and bugs (retrying won't help until a fix is deployed; then bulk-retry the dead set).
- **Circuit breakers** around fragile dependencies: when a provider is down, stop hammering it (pause the queue or fail fast) and resume later.
- **Poison pill protection**: a job that crashes the worker process (segfault, OOM) must not loop forever. Track attempts in the store, not just in memory.
- Visibility/lock timeout > job max runtime, or the job runs concurrently twice. Use heartbeats to extend leases for long jobs.

---

## 6. Priorities, Fairness, Rate Limits {#fairness}

- **Separate queues by priority/latency class** (`critical`, `default`, `low`, `bulk`) with dedicated worker pools, so a 1M-row export doesn't delay password reset emails. Weighted polling across queues (Sidekiq weights) or strict priority (risk of starving low priority).
- **Multi-tenant fairness**: one tenant enqueues 1M jobs and everyone else waits. Mitigations: per-tenant queues or partitioning, fair scheduling (round-robin across tenants), per-tenant concurrency limits (BullMQ groups, Sidekiq Enterprise limiters, Oban Pro partitioned limits), and rate limiting at enqueue.
- **Downstream rate limits**: a global limiter (Redis token bucket) shared by all workers for the third-party API's quota, with concurrency limits per job type to protect the DB.
- **Uniqueness / dedup**: prevent enqueueing the same job twice while it's pending (`unique for 10 min` by args hash: Sidekiq unique jobs, BullMQ job IDs, Oban unique). Also **debounce** (only the last update in a 5 s window triggers reindexing) and **throttle**.

---

## 7. Scheduled / Cron Jobs in Distributed Systems {#cron}

Problem: you deploy 10 app instances, each with a cron entry for `0 2 * * * nightly_billing`, so billing runs 10 times.

Solutions:
1. **Single scheduler** that only *enqueues* jobs (Sidekiq-Cron/Sidekiq Enterprise periodic jobs, Celery Beat (must be a singleton process!), BullMQ repeatable jobs (dedup by key in Redis), Oban Cron (leader election via PG), Quartz clustered mode (DB locks), db-scheduler).
2. **Leader election / distributed lock**: run only on the instance holding the lock (PG advisory lock `pg_try_advisory_lock`, Redis lock with TTL, Kubernetes Lease, etcd/Consul sessions). Beware lock expiry while still running (use fencing tokens or idempotent jobs).
3. **Platform schedulers**: Kubernetes **CronJob** (`concurrencyPolicy: Forbid` to avoid overlap, `startingDeadlineSeconds`, history limits), AWS EventBridge Scheduler → Lambda/SQS/ECS, Cloud Scheduler (GCP), GitHub Actions schedule (not for production-critical work).
4. **Workflow engines** with cron schedules (Temporal schedules, Airflow for data pipelines).

Cron job design:
- **Idempotent and re-runnable**: running twice must be harmless ("bill all subscriptions due ≤ today that haven't been billed for this period").
- **Catch-up semantics**: if the scheduler was down at 02:00, should the job run when it comes back? Decide (missed-run policies: skip, run once, run all).
- **Overlap**: prevent a new run while the previous one is still going (lock or `concurrencyPolicy: Forbid`).
- **Time zones and DST**: "2:30 AM daily" doesn't exist on spring-forward days in DST zones, and runs twice on fall-back. Schedule in UTC, or use schedulers with explicit TZ handling.
- **Thundering herd at :00**: thousands of tenants' jobs all at midnight. Add jitter or spread schedules.
- **Monitoring**: alert if a job **didn't run** (dead man's switch: Healthchecks.io, Cronitor, a Prometheus metric with last success timestamp), not only if it failed.

---

## 8. Delayed Jobs and Timers {#delayed}

"Send a reminder 24 h after signup if not activated", "expire reservation after 15 minutes", "retry webhook in 1 h".
- Redis sorted set with score = due timestamp: a poller moves due jobs to the ready queue (Sidekiq scheduled set, BullMQ delayed).
- DB table with `run_at` + index + `SKIP LOCKED` polling.
- SQS delay (max 15 min) / message timers. EventBridge Scheduler one-time schedules (any future time, at scale). RabbitMQ delayed exchange / TTL+DLX. Kafka has no native delay (use retry topics with pause or a separate scheduler).
- Workflow engines: `workflow.sleep(Duration.ofDays(30))` in Temporal is durable across restarts.
- At execution, **re-check the condition** ("is the user still unactivated?"), because the world changed since scheduling. Prefer this over cancelling scheduled jobs.
- For many timers per entity, use a **timer wheel** / bucketed approach (process "due this minute" buckets).

---

## 9. Durable Execution / Workflow Engines {#workflows}

When business processes span minutes to months with multiple steps, retries, timers, human approvals, and compensations, hand-rolled job chains plus status columns become fragile spaghetti. **Workflow engines** persist the execution state for you.

**Temporal** (fork of Uber's Cadence):
```go
func OrderWorkflow(ctx workflow.Context, order Order) error {
    ao := workflow.ActivityOptions{StartToCloseTimeout: time.Minute,
        RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 5}}
    ctx = workflow.WithActivityOptions(ctx, ao)

    if err := workflow.ExecuteActivity(ctx, ReserveInventory, order).Get(ctx, nil); err != nil { return err }
    if err := workflow.ExecuteActivity(ctx, ChargePayment, order).Get(ctx, nil); err != nil {
        _ = workflow.ExecuteActivity(ctx, ReleaseInventory, order).Get(ctx, nil)   // compensation
        return err
    }
    // wait up to 7 days for a "shipped" signal
    sel := workflow.NewSelector(ctx)
    shipped := workflow.GetSignalChannel(ctx, "shipped")
    sel.AddReceive(shipped, func(c workflow.ReceiveChannel, _ bool) { c.Receive(ctx, nil) })
    sel.AddFuture(workflow.NewTimer(ctx, 7*24*time.Hour), func(f workflow.Future) { /* escalate */ })
    sel.Select(ctx)
    return workflow.ExecuteActivity(ctx, SendDeliveredEmail, order).Get(ctx, nil)
}
```
How it works:
- **Workflow code** must be **deterministic**: no direct I/O, random numbers, or system time; use workflow APIs. The engine records an **event history** (activity scheduled, completed, timer fired, signal received). On worker crash, another worker **replays** the history to rebuild state and continues where it left off.
- **Activities** are the side-effecting steps (call APIs, DB). They're retried per policy and must be idempotent.
- Durable **timers** (sleep for months), **signals** (external events into a running workflow), **queries** (read workflow state), child workflows, versioning APIs for changing workflow code safely.
- Use cases: order fulfillment, payments and refunds, user onboarding sequences, subscription billing lifecycles, infrastructure provisioning, ML pipelines, sagas.

Alternatives: **AWS Step Functions** (JSON/ASL state machines, managed, visual; Standard for long-running, Express for high-volume short ones), Azure Durable Functions, Google Workflows, **Restate**, **Inngest**, **Trigger.dev**, **DBOS** (durable execution in Postgres), Camunda/Zeebe (BPMN), Netflix Conductor. For data pipelines (DAGs of batch tasks): **Airflow**, Dagster, Prefect, Argo Workflows.

When to use: multi-step, long-running processes where reliability and visibility matter. Overkill for "send an email".

---

## 10. Batch Processing Patterns {#batch}

- **Chunked iteration** with keyset pagination (`WHERE id > :last ORDER BY id LIMIT 1000`), committing progress per chunk.
- **Fan-out/fan-in**: a parent job enqueues N child jobs (one per chunk/tenant), and a callback runs when all complete (Celery chord, Sidekiq batch callbacks, BullMQ flow, Temporal child workflows).
- **Throttle to protect the primary DB**: sleep between chunks, run on replicas for reads, stop when replica lag exceeds a threshold.
- **Exports**: stream query results to S3 (multipart upload), then deliver a presigned URL via email or notification.
- **Imports**: upload to S3, validate in a staging table, apply in batches, and produce an error report per row.
- **Spring Batch** (Java) concepts: Job → Steps → chunk-oriented (read → process → write), restartability via JobRepository, skip/retry policies, partitioning.

---

## 11. Observability and Operations {#ops}

Metrics per queue and job type:
- Enqueue rate, processing rate, **queue depth**, **latency (time from enqueue to start)**, which is the key user-facing metric, runtime p50/p95/p99, failure and retry rate, and dead set size.
- Worker utilization (busy/total), memory per worker (leaks in long-running processes, so recycle workers after N jobs).

Practices:
- SLOs per queue ("95% of `critical` jobs start within 10 s").
- Autoscale workers on queue latency or depth (KEDA, ECS target tracking on SQS `ApproximateNumberOfMessagesVisible` / backlog per task).
- Structured logs with job ID, attempt, and correlation ID. Traces linking the API request that enqueued the job to its execution.
- Runbooks for "queue backing up": check for a downstream outage, a poison message, an under-scaled pool, or a stuck lock.
- Separate Redis instances for queues vs cache (cache eviction policies must never evict jobs! Use `maxmemory-policy noeviction` for queue Redis).

---

## 12. Interview Questions {#qa}

1. What kinds of work should be moved to background jobs, and how should the API respond?
2. Why should jobs be idempotent? Give concrete techniques.
3. Why should you enqueue after the DB transaction commits?
4. How do you prevent a cron job from running on every instance?
5. How do you prevent one tenant's bulk job from starving others?
6. How would you implement "send a reminder 3 days after signup if the user hasn't completed onboarding"?
7. What problems do workflow engines like Temporal solve? Why must workflow code be deterministic?
8. Which metrics tell you a job system is unhealthy?
9. How do you process 50 million rows safely in a background job?
