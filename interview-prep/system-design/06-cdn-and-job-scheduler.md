# System Design Case Studies — CDN & Distributed Job Scheduler

> Two more commonly asked designs not covered in `04-case-studies-walkthroughs.md` or `05-more-popular-case-studies.md`. Each one is written out the way you would actually run it in a 45–60 minute interview: **clarify requirements → estimate → API → data model → high-level design → deep dives → bottlenecks & failure modes → tradeoffs → likely follow-ups**. The step-by-step method is explained in `01-fundamentals-and-building-blocks.md` Q1.
>
> **Time budget for a 45-min round** (a rough guide): requirements + estimation ~5–8 min, API + data model ~5 min, high-level diagram ~10 min, deep dives ~15–20 min, bottlenecks/tradeoffs/wrap-up ~5 min. Say your assumptions out loud, and ask the interviewer which part they want to go deeper on before you start the deep dives.

---

## 1. Design a Content Delivery Network (CDN)

### Step 1 — Clarify requirements

Start by asking: *"Are we building the CDN itself, as a provider like Cloudflare/Akamai/CloudFront, or just using one in our app?"* This answer assumes we're building the CDN, which is the harder version of the question.

**Functional**
- Serve customer ("origin") content from edge locations close to end users: static assets (images, JS/CSS, video segments) and, optionally, cacheable dynamic responses.
- Customers configure caching per path: TTLs, cache keys, which headers/cookies vary the response, and the origin address.
- **Purge/invalidate** content by URL, by prefix, or by tag ("surrogate key") within seconds.
- HTTPS everywhere, including customer custom domains (TLS cert management).
- Logs and analytics for customers: hit ratio, bandwidth, status codes.

**Non-functional**
- **Low latency**: edge time-to-first-byte (TTFB) under ~50 ms for cache hits, for most users worldwide.
- **High availability**: 99.99%+. A dead PoP (point of presence) must not take customers offline.
- **High hit ratio** (>90–95% for static content), because origin offload is the whole point.
- Absorb traffic spikes and DDoS attacks without passing them to the origin.
- Out of scope unless asked: edge compute (Workers/Lambda@Edge), WAF rule authoring, live video ingest.

### Step 2 — Back-of-envelope estimation

Assumptions to state out loud:
- Peak egress of 100 Tbps worldwide across ~300 PoPs, so about 330 Gbps per PoP on average (big-city PoPs are much higher, small ones much lower).
- Average object served is ~100 KB (a mix of images, scripts and video segments). 100 Tbps ÷ 800 Kbit ≈ **125M requests/sec worldwide**, or ~400K req/s per PoP.
- A single edge server with 2×100 GbE NICs serving from RAM/NVMe can push roughly 100–200 Gbps, so a mid-size PoP needs a few dozen servers. A large PoP needs hundreds.
- With a 95% hit ratio, origin fetches are 5% of 125M ≈ **6M req/s spread across all customers' origins**. Request collapsing and tiered caching (below) cut that further.
- Storage: the working set per PoP is what matters, not the whole internet. 100 servers × 30 TB NVMe ≈ 3 PB per large PoP, plus ~0.5–1 TB of RAM per server for the hottest objects.

**What the numbers tell you**: this is a **bandwidth and hit-ratio problem**, not a compute problem. Every design decision should be judged by "does it raise the hit ratio or cut latency/bandwidth cost?"

### Step 3 — API / interfaces

There are two surfaces: the data plane (end users) and the control plane (customers).

```
# Data plane: plain HTTP(S); the CDN is transparent to end users
GET https://cdn.customer.com/img/logo.png

# Control plane: customer-facing REST API
PUT  /v1/zones/{zoneId}/config        { origin, cacheRules[], tlsCert, ... }
POST /v1/zones/{zoneId}/purge         { urls[] | prefixes[] | tags[] }   -> 202 { purgeId }
GET  /v1/zones/{zoneId}/purge/{id}    -> { status: pending|completed, popsDone, popsTotal }
GET  /v1/zones/{zoneId}/analytics?from=&to=&groupBy=
```

The origin controls caching through standard headers: `Cache-Control: max-age / s-maxage / stale-while-revalidate / stale-if-error`, `ETag` / `Last-Modified` for revalidation, `Vary`, and `Surrogate-Key` for tag-based purge.

### Step 4 — Data model

- **Config store** (control plane, strongly consistent; Postgres or etcd): `zones(zone_id, customer_id, domains[], origin, cache_rules jsonb, version)`. Every change bumps `version` and is pushed to all PoPs.
- **Edge cache index** (per server, in memory): `cache_key → {disk_location, size, expires_at, etag, surrogate_keys[], last_access}`. The cache key defaults to `scheme + host + path + normalized query`, plus any `Vary` header values.
- **Purge log**: `purge_id, zone_id, matchers, created_at`. Tag purges also need a `surrogate_key → [cache_keys]` index on each node.
- **Logs**: edge request logs are streamed (Kafka) into an OLAP store (ClickHouse/Druid) for customer analytics. They are never written synchronously on the request path.

### Step 5 — High-level design

```
                    ┌───────────────── Control plane ─────────────────┐
 Customer ──API──▶  │ Config API → Config DB → Config distribution    │
                    │ Purge API  → Purge fan-out (pub/sub to all PoPs)│
                    │ Cert manager (ACME) → Key store (HSM/KMS)        │
                    └──────────────────────┬──────────────────────────┘
                                           │ push config/purges/certs
 End user                                  ▼
   │ 1. DNS / Anycast        ┌─────────── Edge PoP (×300) ───────────┐
   └────────────────────────▶│ L4 LB (ECMP) → L7 edge servers        │
                             │   TLS termination, WAF/DDoS filters   │
                             │   RAM cache → NVMe cache              │
                             │   consistent-hash to "owner" server   │
                             └───────────────┬───────────────────────┘
                                   miss      ▼
                             ┌──── Regional / shield tier (tier-2) ───┐
                             │ bigger caches, fewer nodes per origin  │
                             └───────────────┬────────────────────────┘
                                   miss      ▼
                                   Customer origin (S3 / app servers)

 Edge logs ──▶ Kafka ──▶ ClickHouse ──▶ customer analytics / billing
```

**Request flow**
1. **Routing the user to a nearby PoP**: either **Anycast** (every PoP announces the same IP prefix over BGP and the internet routes to the topologically closest one; simple and DDoS-friendly) or **GeoDNS** (the DNS resolver returns the IP of the best PoP based on resolver location, latency measurements and PoP load, which gives finer control). Many CDNs use Anycast plus DNS-based steering for load.
2. Inside the PoP, an L4 load balancer (ECMP + something like Maglev/Katran) spreads connections across edge servers.
3. The edge server terminates TLS, applies WAF/rate-limit rules, computes the cache key and checks RAM → NVMe.
4. **Hit**: serve it. **Miss**: consistent-hash the key to the server in this PoP that "owns" it (so the PoP stores one copy, not N), then go to the regional shield tier, then to the origin.
5. The response is stored at each tier according to TTL rules and streamed back to the client *while* it is still downloading (don't wait for the full object).

### Step 6 — Deep dives

**Deep dive A: Maximizing hit ratio (tiered caching and consistent hashing)**
- Without coordination, each of 300 PoPs × 100 servers would fetch the same object from the origin separately. Two fixes:
  - **Within a PoP**: consistent hashing on the cache key means each object lives on one or two "owner" servers. Non-owner servers proxy to the owner, and may keep a small hot copy in RAM for very popular objects.
  - **Across PoPs**: a **tiered / origin-shield** layer. Edge PoPs miss to a handful of regional tier-2 caches, and only those talk to the origin. This turns 300 origin fetches into a few, and raises the effective hit ratio for long-tail content.
- **Request collapsing (coalescing)**: if 10,000 users request the same uncached object at the same moment, only *one* request goes upstream; the others wait on it. This is the CDN version of cache-stampede protection (see the caching file).
- **Cache key normalization**: strip irrelevant query params (`utm_*`), sort the rest, lowercase the host, and avoid `Vary: User-Agent` (it fragments the cache into thousands of variants; normalize to device class instead).
- **Eviction**: LRU is the baseline. Real CDNs use segmented LRU / TinyLFU-style admission ("only admit to disk after the second request") so one-hit wonders don't push hot content out. Use separate pools for small objects (RAM) and large objects (disk).

**Deep dive B: Invalidation / purge in seconds**
- TTL expiry is the default. Purge exists for "we shipped a bad file, remove it everywhere now".
- Flow: Purge API → write to purge log → publish on a global pub/sub fan-out (a tree of relays, or Kafka per region) → every edge server applies it → it ACKs → the API reports `completed` when all PoPs have acknowledged.
- **Cheap implementation**: don't scan disks. Either (a) mark the index entry invalid, or (b) for prefix/tag/"purge everything", keep a per-zone **generation number** in the cache key; bumping it makes all old entries unreachable at once, and LRU removes them later.
- **Soft purge**: mark entries stale rather than deleting them, so the edge can still serve stale content if the origin is down (`stale-if-error`) and revalidates with `If-None-Match`.
- A PoP that was partitioned during a purge must catch up from the purge log when it reconnects (track the last applied sequence number per PoP) before it serves from its cache again.

**Deep dive C: TLS at the edge**
- Terminating TLS at the edge saves the 1–2 long round trips of the handshake. Session resumption and TLS 1.3 / QUIC (HTTP/3) help further.
- Custom domains: issue certs automatically via ACME (Let's Encrypt), distribute them to PoPs, and select them by SNI. Private keys go to edge machines encrypted, or stay in an HSM/keyless setup where the edge calls a key server for the signing step only.

**Deep dive D: Dynamic content and protecting the origin**
- Uncacheable responses still benefit: the edge keeps **warm, persistent, multiplexed connections** to the origin and routes over the CDN's private backbone, which beats a cold TCP+TLS handshake across the public internet.
- Use `stale-while-revalidate` for semi-dynamic pages, and microcaching (1–5 s TTL) for high-QPS API responses that tolerate staleness.
- DDoS: Anycast spreads volumetric attacks across all PoPs. SYN cookies and XDP/eBPF filters drop junk at the NIC, and L7 rate limiting is applied per zone. The origin should only accept traffic from CDN IP ranges.

### Step 7 — Bottlenecks & failure modes
| Failure | Mitigation |
|---|---|
| Edge server dies | L4 LB health checks remove it. Consistent hashing remaps only its keys (~1/N). |
| Whole PoP down / overloaded | Anycast withdraws the BGP route, or DNS steers users to the next-nearest PoP. Keep capacity headroom (N+1 PoPs per region). |
| Origin down | Serve stale content (`stale-if-error`), and return customer error pages from the edge. |
| Thundering herd on a viral object | Request collapsing, the shield tier, and replicating the hot object to more servers within the PoP. |
| Bad config pushed globally | Versioned configs, staged rollout (canary PoPs first), and instant rollback. Many big CDN outages came from global config pushes. |
| Purge fan-out lag | Track per-PoP ACKs. A lagging PoP is drained or must catch up before it serves again. |

### Step 8 — Tradeoffs to call out
- **Anycast vs GeoDNS**: Anycast is simpler, fails over fast and absorbs DDoS well, but gives coarse control. GeoDNS gives fine-grained steering, but DNS TTLs slow failover and the resolver's location ≠ the user's (EDNS Client Subnet helps).
- **Push vs pull CDN**: *pull* (fetch from origin on first miss) suits most web content. *Push* (the customer uploads to the CDN ahead of time) suits large, predictable content like a game patch or a video release.
- **Long TTL vs freshness**: long TTLs plus versioned filenames (`app.3f9a.js`) give near-100% hit ratio with no purge needed. Short TTLs mean more origin load.
- **More PoPs vs bigger PoPs**: many small PoPs give lower latency but a lower hit ratio each (the cache is fragmented). Fewer, larger PoPs have better hit ratios. Tiered caching gets you some of both.

### Likely follow-ups
- *"How would you serve video?"* HLS/DASH segments are ideal CDN objects (small, immutable, cacheable). Pre-warm popular titles. See the video streaming case study in `04`.
- *"How do you bill customers?"* Aggregate bytes served per zone per PoP from the log pipeline, priced by region.
- *"How do you measure which PoP is best for a user?"* Real-user monitoring (RUM) beacons feed latency maps into the DNS steering layer.

---

## 2. Design a Distributed Job Scheduler

(Examples: cron at scale, Airflow, Quartz cluster, AWS EventBridge Scheduler, Uber Cadence/Temporal timers.)

### Step 1 — Clarify requirements

Ask: *"One-off delayed jobs, recurring cron jobs, or DAG workflows with dependencies? Is a job a payload we hand to a worker, or code we run?"* Assume: one-off and recurring jobs, with optional simple dependencies, executed by a worker fleet that pulls tasks.

**Functional**
- Submit a job: run `payload` at time T (one-off), or on a cron expression (recurring), optionally in a time zone.
- Cancel / pause / update a job; query job status and execution history.
- Retries with backoff, a per-job timeout, and a max-attempts limit, then a dead-letter queue (DLQ).
- Priorities and per-tenant quotas (one tenant can't starve the others).
- *(Stretch)* Job dependencies (run B after A succeeds), i.e. a DAG.

**Non-functional**
- **Never lose a job** once it has been accepted (durability).
- **Timeliness**: jobs start within ~1–2 s of their scheduled time at p99.
- **At-least-once execution**, made *effectively* exactly-once through idempotency. Be explicit that true exactly-once across a network is not achievable (see the messaging file).
- Horizontally scalable, with no single point of failure.

### Step 2 — Back-of-envelope estimation
- 100M scheduled executions/day ≈ 1,160/s average. Schedules cluster on round times (top of the hour, midnight UTC), so plan for **peaks of ~50–100K jobs due in the same second**. That spike is the core scaling problem.
- Stored jobs: 500M active definitions × ~1 KB ≈ 500 GB. Execution history: 100M/day × 500 B ≈ 50 GB/day, so it must be TTL'd or archived (keep 30 days hot, ≈ 1.5 TB).
- Workers: if the average job runs for 2 s, then 1,160 jobs/s × 2 s ≈ 2,300 concurrent slots on average, and far more at peak. Workers must autoscale on queue depth.

**Takeaway**: storing the jobs is easy. The hard parts are **finding due jobs efficiently** and **dispatching each one exactly once at peak**.

### Step 3 — API
```
POST   /v1/jobs   { name, payload, schedule: {runAt | cron, timezone},
                    retryPolicy: {maxAttempts, backoff}, timeoutSec,
                    priority, idempotencyKey, dependsOn[] }   -> 201 { jobId }
GET    /v1/jobs/{jobId}                     -> definition + next_run_at
GET    /v1/jobs/{jobId}/executions?limit=   -> history
PATCH  /v1/jobs/{jobId}  { paused | schedule | ... }
DELETE /v1/jobs/{jobId}

# Worker protocol (internal)
POST /internal/lease       { workerId, queues[], max }  -> [ {executionId, payload, leaseUntil} ]
POST /internal/heartbeat   { executionId }              -> extends lease
POST /internal/complete    { executionId, status, output|error }
```
The `idempotencyKey` on submit stops client retries from creating duplicate jobs.

### Step 4 — Data model

```sql
-- Job definitions (source of truth); sharded by job_id
jobs(job_id PK, tenant_id, payload, cron_expr, timezone, run_at,
     retry_policy, timeout_sec, priority, status /*active|paused|deleted*/,
     next_run_at, version)

-- The "due index": what the scheduler scans. Partitioned by time bucket
-- (and shard) so finding due work is a range read, not a full scan.
schedule_index(shard_id, bucket_minute, next_run_at, job_id)  -- PK in this order

-- One row per attempt-run; also the dedup guard
executions(execution_id PK, job_id, scheduled_time, attempt, status
           /*queued|running|succeeded|failed|timed_out*/, worker_id,
           lease_until, started_at, finished_at, error,
           UNIQUE(job_id, scheduled_time))
```

`UNIQUE(job_id, scheduled_time)` is the key correctness trick: if two scheduler nodes both decide to fire the same occurrence, only one insert succeeds.

Storage choice: Postgres/MySQL (sharded) is fine up to a large scale and gives transactions plus `SELECT … FOR UPDATE SKIP LOCKED`. At very large scale, use Cassandra/DynamoDB with time-bucketed partitions for the schedule index.

### Step 5 — High-level design

```
 Clients ─▶ API service ─▶ Jobs DB (definitions + schedule_index, sharded)
                                   ▲             │ scan due buckets
                                   │             ▼
                        update next_run_at   Scheduler nodes (each owns a set of
                                   │          shards via leader election / leases)
                                   │             │ insert execution (dedup) + enqueue
                                   │             ▼
                                   │      Task queue (Kafka / SQS / Redis streams),
                                   │      partitioned by priority/tenant
                                   │             │ pull / lease
                                   │             ▼
                                   └──────  Worker fleet (autoscaled on queue lag)
                                                 │ heartbeat / complete
                                                 ▼
                                   Executions DB ─▶ metrics, alerts, DLQ, UI
          Reaper/Watchdog: finds expired leases → requeue or fail
```

**Flow**
1. The API validates the cron expression, computes the first `next_run_at` (in UTC, from the job's time zone) and writes `jobs` plus `schedule_index` in one transaction.
2. **Scheduler nodes** each own a subset of shards. The ownership map lives in ZooKeeper/etcd, or uses lease rows in the DB. Every ~1 s, each node reads the due rows from its shards: `next_run_at <= now() + lookahead`.
3. For each due job: insert an `executions` row (the unique constraint blocks duplicates), publish to the task queue, then compute and write the job's **next** `next_run_at` (recurring jobs) or mark it done (one-off).
4. **Workers** lease a task, heartbeat while running, and report completion. Failures are retried with exponential backoff and jitter by re-scheduling a new attempt. After `maxAttempts` the task goes to the DLQ and an alert fires.
5. A **reaper** finds executions whose `lease_until` has passed (the worker crashed) and requeues them.

### Step 6 — Deep dives

**Deep dive A: Finding due jobs efficiently**
- A naive `SELECT * FROM jobs WHERE next_run_at <= now()` over 500M rows doesn't scale, and a single poller is a single point of failure.
- **Time-bucketed index**: partition by `(shard, minute bucket)`, so each scheduler reads only the current bucket of the shards it owns. That is a small, sequential range read.
- **Two-level approach (common in practice)**: a coarse DB scan every minute loads jobs due in the *next* minute into an in-memory **timing wheel** or min-heap on the scheduler node, which then fires each one at its exact second. This gives sub-second precision without hammering the DB.
- Alternatively, use Redis sorted sets (`ZADD due <timestamp> jobId`, then `ZRANGEBYSCORE` + `ZREM` atomically in Lua) as the hot index, with the DB as the source of truth for rebuilding it.

**Deep dive B: Exactly-once dispatch / avoiding double runs**
- Two schedulers can believe they own the same shard: a GC pause or network partition outlives the lease, and the "zombie" node wakes up still thinking it is leader. Defenses, in layers:
  1. **Leases with fencing tokens**: each shard-ownership lease carries a monotonically increasing epoch. Writes include the epoch and the DB rejects stale ones (see the consensus file in `distributed-systems/`).
  2. **The DB unique constraint** on `(job_id, scheduled_time)` is the final arbiter: a second dispatch attempt fails the insert and is dropped.
  3. **Worker-side idempotency**: the queue is at-least-once, so the job handler must be idempotent, e.g. use `execution_id` as an idempotency key when charging a card or sending an email.
- Alternative pattern: `SELECT … FROM schedule_index WHERE next_run_at <= now() ORDER BY next_run_at LIMIT 500 FOR UPDATE SKIP LOCKED`. This lets many schedulers poll the same table without coordination; each row is claimed by exactly one transaction. It is simple and works well up to roughly tens of thousands of jobs/s per DB cluster.

**Deep dive C: Handling the top-of-the-hour spike**
- Lookahead pre-loading (deep dive A) spreads the DB reads out ahead of time.
- Dispatch is just an enqueue, which is cheap. The queue absorbs the burst and workers drain it, so jobs may *start* a few seconds late under extreme spikes. Expose that as a metric (`start_delay_seconds` p99) and alert on it.
- Optionally add **jitter**: let tenants opt into "run any time within this 5-minute window", which flattens the peak a lot. Many internal cron systems do this by default.
- Priority queues keep critical jobs (payments, billing) from waiting behind bulk report jobs. Per-tenant rate limits and quotas keep one noisy tenant from flooding the queue.

**Deep dive D: Retries, timeouts and long-running jobs**
- Leases plus heartbeats: a worker must heartbeat every N seconds, or the reaper assumes it is dead and requeues the task. Requeued tasks may then run twice, which is why idempotency is required.
- Retry policy: exponential backoff with full jitter, a capped number of attempts, and separate handling of retryable errors (timeouts, 5xx) vs non-retryable ones (validation errors), which go straight to the DLQ.
- Missed runs after an outage, a.k.a. **misfire policy**: when the scheduler comes back after 10 minutes down, should a job that runs every minute fire 10 times, once, or not at all? Make it configurable (`fire_all | fire_once | skip`). This is a great point to raise unprompted.

**Deep dive E: Cron and time-zone correctness**
- Store and compare everything in UTC, and compute `next_run_at` from the cron expression *in the job's time zone*. DST transitions mean "02:30 daily" might not exist on one day and happen twice on another. Pick a rule (Quartz-style: skip the nonexistent time, run the ambiguous one once) and document it.
- Always compute the next run from the **scheduled** time, not the actual finish time, or the schedule drifts.

**Deep dive F (stretch): DAG dependencies**
- Store edges `(parent_job, child_job)`. When a parent execution succeeds, a completion handler decrements the child's pending-parents counter for that run, atomically, and enqueues the child when the count reaches zero. That is essentially Kahn's topological order, applied one event at a time.
- For complex workflows with state, branching and human steps, say you would use a workflow engine (Temporal/Cadence, Airflow, Step Functions) rather than build one. See `messaging/05-background-jobs-scheduling-and-workflows.md`.

### Step 7 — Bottlenecks & failure modes
| Failure | Mitigation |
|---|---|
| Scheduler node crashes | Its shard leases expire, other nodes take over (rebalance via etcd/ZK), and the new owner rescans from the last checkpointed bucket, so nothing is skipped. |
| Split brain (two schedulers own a shard) | Fencing tokens, plus the unique `(job_id, scheduled_time)` constraint, plus idempotent handlers. |
| Worker crashes mid-job | Lease expiry, then the reaper requeues the task (at-least-once). |
| Queue backlog at peak | Autoscale workers on queue lag; priority lanes; alert on start-delay SLO. |
| DB hot partition (everyone at 00:00) | Shard the schedule index by `hash(job_id)` *within* each time bucket, so one minute's jobs are spread over many partitions. |
| Poison job crashes workers repeatedly | Max attempts, then DLQ. Isolate execution (container/sandbox per job type) so one bad job can't kill a worker. |

### Step 8 — Tradeoffs to call out
- **Polling the DB vs an in-memory timing wheel**: polling is simple and durable but coarse and DB-heavy. A timing wheel is precise and cheap, but must be rebuilt from the DB after a crash. Use both: DB as the source of truth, wheel for precision.
- **Push (scheduler sends to workers) vs pull (workers lease from a queue)**: pull gives natural backpressure and simpler worker autoscaling, so it is usually preferred.
- **Central scheduler vs sharded schedulers**: a single leader (with hot standby) is simple and fine up to ~thousands of jobs/s. Shard ownership is needed beyond that.
- **Precision vs load**: second-level precision costs much more than "within a minute". Ask what the business actually needs.

### Likely follow-ups
- *"How do you guarantee a job never runs twice?"* You can't fully, so say exactly that. Explain the dedup layers, then idempotent handlers keyed on `execution_id`.
- *"How would users see what happened?"* An execution history table/UI, structured logs per `execution_id`, metrics (success rate, start delay, duration) and alerts on DLQ growth.
- *"How do you handle 1 billion one-off delayed jobs, like 'remind me in 30 days'?"* Store them cold in time-bucketed partitions and only load the next window into the hot index (Redis sorted set / timing wheel), so long-dated jobs cost nothing until they are close.

---

## Already covered elsewhere (not duplicated here)

| Question | Where |
|---|---|
| Chat (WhatsApp/Slack), URL shortener, notifications, video streaming, rate limiter, ride-hailing, autocomplete, distributed cache, social feed, distributed ID generator, e-commerce order/checkout flow | `04-case-studies-walkthroughs.md` |
| Payment system (Stripe/Razorpay), Google Docs, Dropbox/Google Drive | `05-more-popular-case-studies.md` |
