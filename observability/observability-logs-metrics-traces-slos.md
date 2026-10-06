# Observability: Logs, Metrics, Traces, SLOs, Alerting, and Incident Response

> DevOps-flavored Q&A: [`interview-prep/devops/10-monitoring-logging-observability.md`](../interview-prep/devops/10-monitoring-logging-observability.md) and [`12-sre-incident-management.md`](../interview-prep/devops/12-sre-incident-management.md). This note is the backend engineer's guide to **instrumenting** services and **using** telemetry to run them.

## Table of Contents
1. [Monitoring vs Observability](#what)
2. [The Signals: Logs, Metrics, Traces, Profiles, Events](#signals)
3. [Structured Logging Done Right](#logging)
4. [Metrics: Types, Labels, Cardinality](#metrics)
5. [What to Measure: RED, USE, Golden Signals](#methods)
6. [Histograms and Percentiles](#percentiles)
7. [Distributed Tracing](#tracing)
8. [OpenTelemetry](#otel)
9. [Correlating Signals](#correlation)
10. [SLIs, SLOs, Error Budgets](#slo)
11. [Alerting That Doesn't Burn Out the Team](#alerting)
12. [Dashboards](#dashboards)
13. [Continuous Profiling](#profiling)
14. [Incident Response and Postmortems](#incidents)
15. [Tooling Landscape and Cost Control](#tools)
16. [Interview Questions](#qa)

---

## 1. Monitoring vs Observability {#what}

- **Monitoring**: watching known failure modes with predefined dashboards and alerts ("is CPU > 90%?", "is the error rate > 1%?"). It answers known questions.
- **Observability**: the ability to understand **any** internal state from external outputs, so you can ask **new questions** about unknown-unknowns without shipping new code ("why are checkout requests from Android users in Mumbai with coupon X slow since 14:05?"). This requires rich, high-cardinality, correlated telemetry.

Distributed systems fail in novel ways, so you need both.

---

## 2. The Signals {#signals}

| Signal | What | Strengths | Weaknesses |
|---|---|---|---|
| **Logs** | Timestamped records of discrete events | Detailed context, debugging specific cases | Expensive at volume; hard to aggregate unless structured |
| **Metrics** | Numeric time series (counters, gauges, histograms) with labels | Cheap, fast queries, alerting, long retention, trends | Pre-aggregated, so they lose per-request detail; cardinality limits |
| **Traces** | The causal path of one request across services (spans with timings) | Where time goes, dependency graphs, cross-service debugging | Sampling needed at volume; instrumentation effort |
| **Profiles** | Where CPU/memory is spent in code (stack samples) | Optimization, finding hot code paths and leaks | Not request-scoped (unless linked) |
| **Events / wide events** | One richly attributed record per unit of work | Arbitrary slicing (Honeycomb's model) | Storage cost; needs columnar backends |

Modern view (Charity Majors and others, "Observability 2.0"): store **wide, structured events** (one per request per service, with dozens to hundreds of fields) as the source of truth, and derive metrics from them.

---

## 3. Structured Logging {#logging}

```json
{"ts":"2025-06-01T10:00:00.123Z","level":"error","service":"orders-api","env":"prod","version":"1.42.0",
 "trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7","request_id":"req_7f3a",
 "http.method":"POST","http.route":"/orders","http.status_code":502,"duration_ms":2034,
 "user_id":"u_42","tenant_id":"t_9","order_id":"ord_1",
 "msg":"payment provider call failed","error.type":"TimeoutError","provider":"stripe","attempt":3}
```
Rules:
- **Structured (JSON)**, not free text. Machine-parseable and queryable by field.
- **Consistent field names** across services (adopt OpenTelemetry semantic conventions: `http.request.method`, `http.response.status_code`, `db.system`, `error.type`, `service.name`).
- Always include: timestamp (UTC, ISO-8601), level, service, version, environment, **trace_id/span_id**, request ID, and relevant business IDs (tenant, user, order).
- **Levels**: ERROR (needs attention, something failed), WARN (unexpected but handled), INFO (significant business/lifecycle events), DEBUG (diagnostic detail, usually off in prod or sampled). Don't log at ERROR for client mistakes (4xx). That's noise.
- **Log events, not narration**: one log line per meaningful event with all context, rather than ten lines telling a story.
- **Never log secrets or sensitive PII** (passwords, tokens, card numbers, Aadhaar/SSN, full request bodies by default). Use redaction middleware and allowlists, and keep regulatory constraints in mind (GDPR, DPDP, PCI, HIPAA).
- Log to **stdout**, and let the platform ship logs (Fluent Bit, Vector, OTel Collector) to storage (Loki, Elasticsearch/OpenSearch, ClickHouse, CloudWatch, Datadog, Splunk).
- Use **async/buffered logging**, so logging I/O doesn't block request threads. Be careful with log volume in hot loops.
- **Sampling** for high-volume, low-value logs (e.g., 1% of successful health checks), and keep all errors.
- Libraries: Go `slog`/zap/zerolog, Java SLF4J + Logback/Log4j2 with a JSON encoder (+ MDC for context), Python `structlog`/`logging` JSON formatter, Node `pino`, .NET Serilog.
- **Contextual logging**: attach the request context once (middleware puts trace_id, user_id into MDC/context) so every log line inherits it.

---

## 4. Metrics {#metrics}

Types (Prometheus/OpenMetrics vocabulary):
| Type | Semantics | Examples |
|---|---|---|
| **Counter** | Monotonically increasing (resets on restart) | `http_requests_total`, `orders_created_total`, `errors_total`. Query with `rate()` / `increase()` |
| **Gauge** | Value that goes up and down | `queue_depth`, `db_pool_active_connections`, `memory_bytes`, `temperature` |
| **Histogram** | Observations counted into buckets (+ sum, count) | `http_request_duration_seconds`; aggregatable percentiles across instances |
| **Summary** | Client-side quantiles | Precomputed p99 per instance (**can't be aggregated across instances**, so prefer histograms) |

**Labels/dimensions** (`method="POST", route="/orders", status="502"`) give powerful slicing, but each unique label combination is a separate time series.
**Cardinality** = number of unique series. **Never use unbounded values as metric labels**: user IDs, order IDs, raw URLs with IDs (`/orders/123`), email, IP, error message text. That causes **cardinality explosion**: memory blow-ups in Prometheus and huge bills in SaaS. Use the **route template** (`/orders/{id}`), status class, and bounded enums. Put high-cardinality data in logs, traces, and wide events instead.

Naming conventions (Prometheus): `<namespace>_<name>_<unit>_<suffix>` like `http_server_request_duration_seconds`, `_total` for counters, base units (seconds, bytes).

Pull vs push:
- **Prometheus pulls** (scrapes `/metrics` endpoints). Service discovery finds targets, and it's easy to see if a target is down.
- Push: StatsD, OTLP push to a collector, Prometheus Pushgateway (only for batch jobs).
- Scaling Prometheus: federation, or remote write to **Thanos**, **Cortex/Grafana Mimir**, or **VictoriaMetrics** for long-term storage and global queries.

Business metrics matter as much as technical ones: orders/min, payment success rate, signups, cart abandonment. They're often the first signal of user-facing problems ("orders dropped to zero" catches bugs that return 200 OK).

---

## 5. What to Measure {#methods}

**RED method** (Tom Wilkie), for request-driven **services**:
- **R**ate: requests per second.
- **E**rrors: failed requests per second (or %).
- **D**uration: latency distribution (histogram → p50/p90/p99).

**USE method** (Brendan Gregg), for **resources** (CPU, memory, disk, network, connection pools, thread pools, queues):
- **U**tilization: % time busy.
- **S**aturation: queued work (run queue length, pool wait time, swap).
- **E**rrors: error counts.

**Four Golden Signals** (Google SRE): **latency** (separate successful vs failed request latency, since fast errors skew averages), **traffic**, **errors**, **saturation**.

Instrument at every boundary: inbound HTTP/gRPC (server middleware), outbound calls (client middleware: per dependency rate, errors, latency), DB queries (pool stats + query latency), caches (hit ratio), queues (lag, oldest message age, processing time), background jobs, and runtime (GC pauses, heap, threads, event loop lag in Node, goroutines in Go).

---

## 6. Histograms and Percentiles {#percentiles}

- **Averages lie**: an average of 100 ms can hide 5% of requests at 3 s. Use percentiles (p50, p95, p99, p99.9) and look at max/outliers.
- **Tail latency matters at scale**: a page making 50 backend calls is affected by the p99 of each (see "The Tail at Scale" in [`distributed-systems/05`](../distributed-systems/05-resilience-patterns.md#hedging)).
- **Never average percentiles** across instances (the average of per-host p99s isn't the fleet p99). Aggregate histograms (bucket counts sum correctly) or mergeable sketches, then compute percentiles.
- Bucket choice affects accuracy. Use buckets around your SLO threshold (e.g., 0.05, 0.1, 0.25, **0.3**, 0.5, 1, 2.5 s if the SLO is 300 ms). **Native/exponential histograms** (Prometheus native histograms, OTel exponential histograms) give automatic high resolution.
- Measure latency **from the client's perspective** too (LB logs, RUM for frontends). Server-side timers miss queueing before the handler.
- **Coordinated omission** (Gil Tene): load generators that wait for responses before sending the next request under-report latency during stalls. Use tools that account for it (wrk2, k6 with arrival-rate executors, HdrHistogram).

---

## 7. Distributed Tracing {#tracing}

A **trace** is the end-to-end journey of one request, a tree (DAG) of **spans**:
```
Trace 4bf92f…  (POST /checkout, 840 ms)
├─ span: api-gateway  (840 ms)
│  └─ span: orders-api POST /orders  (820 ms)
│     ├─ span: SELECT customers…  (12 ms)        db.system=postgresql
│     ├─ span: inventory-svc ReserveStock (gRPC)  (95 ms)
│     │  └─ span: UPDATE stock…  (40 ms)
│     ├─ span: payments-svc Charge  (650 ms)  ← the bottleneck
│     │  └─ span: HTTP POST api.stripe.com  (630 ms)   http.response.status_code=200
│     └─ span: publish OrderPlaced (kafka)  (5 ms)
```
- Span = name, start/end time, trace ID, span ID, parent span ID, attributes (key-value), events (timestamped logs within the span), status, links (to other traces, e.g., batch consumers linking to many producers).
- **Context propagation**: trace context travels between services in headers (W3C **`traceparent`**: `00-<trace-id>-<parent-span-id>-<flags>`, plus `tracestate`, and **baggage** for custom key-values). Messages carry it in headers too (Kafka record headers, SQS message attributes).
- **Sampling**:
  - **Head-based**: decide at the start (e.g., 1% of traces). Cheap, but it misses rare errors.
  - **Tail-based**: buffer complete traces in a collector, then keep all errors and slow traces plus a sample of normal ones. Better signal, but more infrastructure (OTel Collector tail sampling processor, Honeycomb Refinery).
  - Always sample consistently across services for the same trace (respect the parent's sampled flag).
- Uses: find the bottleneck service or query, see fan-out and N+1 patterns (100 DB spans per request!), build service dependency maps, attribute latency to dependencies, and debug errors across services.
- Backends: Jaeger, Grafana Tempo, Zipkin, Honeycomb, Datadog APM, New Relic, Lightstep/ServiceNow, AWS X-Ray, Google Cloud Trace, Elastic APM, SigNoz.

---

## 8. OpenTelemetry {#otel}

**OpenTelemetry (OTel)** (CNCF, a merger of OpenTracing + OpenCensus) is the vendor-neutral standard for generating and collecting telemetry:
- **API + SDKs** per language for traces, metrics, and logs (plus profiles emerging).
- **Auto-instrumentation** (Java agent, .NET, Python, Node, and Go via eBPF-based instrumentation and compile-time options) for common libraries: HTTP servers and clients, gRPC, DB drivers, Kafka, Redis.
- **Semantic conventions**: standard attribute names, so dashboards and queries work across languages and vendors.
- **OTLP**: the wire protocol (gRPC/HTTP).
- **OpenTelemetry Collector**: a pipeline (receivers → processors → exporters) deployed as an agent or gateway. It batches, samples (tail sampling), redacts PII, enriches with K8s metadata, transforms, and routes to one or many backends. This decouples your code from vendors.
- **Resource attributes** identify the source: `service.name`, `service.version`, `deployment.environment.name`, `k8s.pod.name`, `cloud.region`.

```python
from opentelemetry import trace
tracer = trace.get_tracer("orders")

def place_order(cmd):
    with tracer.start_as_current_span("PlaceOrder") as span:
        span.set_attribute("order.items_count", len(cmd.items))
        span.set_attribute("tenant.id", cmd.tenant_id)
        try:
            ...
        except PaymentDeclined as e:
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, "payment declined"))
            raise
```
Recommendation: instrument with OTel APIs everywhere, run the Collector, and choose backends freely.

---

## 9. Correlating Signals {#correlation}

The power comes from **jumping between signals**:
- Logs include `trace_id`, so from a trace you can see all its logs, and from an error log you can open its trace.
- **Exemplars**: a metric histogram bucket links to an example trace ID ("this p99 spike → click → see a slow trace").
- Traces link to profiles (span → CPU profile for that span: Pyroscope/Grafana, Datadog).
- Deploy markers and feature flag changes annotated on dashboards ("latency rose right after deploy 1.42.0").
- Consistent resource attributes (service, version, env, region, pod) across all signals enable filtering everywhere.

---

## 10. SLIs, SLOs, Error Budgets {#slo}

- **SLI (Service Level Indicator)**: a quantitative measure of user-perceived reliability, usually a ratio of good events to total events:
  - Availability SLI: `successful requests / valid requests` (exclude client errors; count 5xx and timeouts as bad).
  - Latency SLI: `requests faster than 300 ms / valid requests`.
  - Freshness (data pipelines): `% of records processed within 5 min`.
  - Correctness, durability, coverage for other system types.
- **SLO (Service Level Objective)**: the target for an SLI over a window: "99.9% of checkout requests succeed over a rolling 28 days", "99% under 300 ms".
- **SLA (Service Level Agreement)**: a contractual promise with penalties, set **looser** than the internal SLO.
- **Error budget** = 1 − SLO. At 99.9% over 30 days that's **43.2 minutes** of full downtime (or equivalent partial failure):
  | SLO | Downtime / 30 days | / year |
  |---|---|---|
  | 99% | 7.2 h | 3.65 days |
  | 99.9% | 43.2 min | 8.76 h |
  | 99.95% | 21.6 min | 4.38 h |
  | 99.99% | 4.32 min | 52.6 min |
  | 99.999% | 26 s | 5.26 min |
- **Error budget policy**: if the budget is exhausted, freeze risky launches and prioritize reliability work. If there's budget left, ship faster. This aligns product and reliability incentives.
- Choose SLOs from **user expectations**, not from current performance. 100% is the wrong target (it's infinitely expensive, and users can't tell 99.99% from 100% through their flaky mobile networks).
- **Availability of dependencies multiplies**: a service with 5 hard dependencies each at 99.9% can be at most ~99.5% unless you add resilience.
- Tools: Sloth, Pyrra (generate Prometheus SLO rules), OpenSLO spec, Nobl9, Datadog/Grafana/Google Cloud SLO features.

---

## 11. Alerting {#alerting}

Principles (Google SRE, Rob Ewaschuk's *My Philosophy on Alerting*):
- **Alert on symptoms (user impact), not causes**: page on "checkout error rate burning the SLO budget", not "CPU 85%" (high CPU without user impact isn't an emergency). Cause-based signals belong on dashboards and in tickets.
- Every page must be **urgent, actionable, and require human intelligence**. Otherwise make it a ticket or automate it.
- **SLO burn-rate alerts** (multi-window, multi-burn-rate):
  - Page if the budget burns **14.4× faster** than sustainable over 1 h (and also over 5 m, to confirm it's still happening). That would consume 2% of a 30-day budget in an hour.
  - Page if it burns 6× over 6 h (5% of budget).
  - Ticket if it burns 1× over 3 days (10% of budget).
  This catches both fast outages and slow degradations with few false alarms.
- Include in every alert: what's wrong, user impact, a dashboard link, a **runbook link**, and recent changes.
- Route by severity: page (immediate), ticket (business hours), log/notification (FYI).
- Review alerts regularly: delete noisy ones, and track pages per on-call shift (Google aims for ≤ 2 incidents per 12-hour shift).
- **Alert fatigue** causes missed real incidents. Fewer, better alerts.
- Dead-man's-switch alerts for things that should happen (backups ran, cron jobs completed, data arriving).

---

## 12. Dashboards {#dashboards}

- **Service overview dashboard** per service: RED metrics per endpoint, SLO status and budget remaining, saturation (CPU, memory, pools, queues), dependency health (outbound RED per dependency), deploy annotations.
- Top-down drill-down: business KPIs → service map → service → endpoint → instance → traces and logs.
- Keep dashboards few and maintained, and treat them as code (Grafana provisioning, Grafonnet, Terraform).
- Show percentiles, rates, and ratios, not raw counters.
- Use consistent time ranges, UTC, and units.

---

## 13. Continuous Profiling {#profiling}

Always-on, low-overhead sampling profilers in production show **which functions consume CPU and memory** across the fleet over time:
- Tools: Grafana **Pyroscope**, Parca, Datadog Continuous Profiler, Google Cloud Profiler, async-profiler (JVM), `pprof` (Go, built in: `net/http/pprof`), py-spy, eBPF-based whole-system profilers.
- **Flame graphs** (Brendan Gregg): x-axis = proportion of samples, y-axis = stack depth. Wide boxes are where time goes.
- Uses: reduce cloud cost (find the 20% of code burning 80% of CPU), find regressions by diffing profiles between versions, and find memory leaks and allocation hotspots (GC pressure).

---

## 14. Incident Response and Postmortems {#incidents}

**During an incident**:
1. **Detect** (alert or user report) → **acknowledge** → declare an incident with severity.
2. Roles: **Incident Commander** (coordinates and decides; doesn't debug), Operations/Tech lead (hands on keyboard), Communications lead (status page, stakeholders), Scribe (timeline).
3. **Mitigate first, root-cause later**: roll back the deploy, flip the feature flag off, fail over, shed load, scale up, block the abusive client. Restore service, then investigate.
4. Communicate regularly (internal channel + status page) with what's known, impact, next update time.
5. Keep a timeline as you go.
6. Resolve → monitor → close.

**Postmortem (blameless)**:
- What happened (timeline), impact (users, duration, SLO budget consumed, revenue), detection (how and how fast), response, **contributing factors** (systemic causes, not "human error": why was the error possible and not caught?), what went well, where we got lucky, and **action items** with owners and dates (prevent, detect faster, mitigate faster).
- Blameless culture: people acted reasonably with the information they had. Fix the system (guardrails, automation, tests), not the person.
- Track MTTD (mean time to detect), MTTA (acknowledge), and MTTR (restore/resolve), plus the recurrence of similar incidents.
- Share postmortems widely. Famous public ones (GitLab 2017, Cloudflare 2019 regex, AWS us-east-1 2021, Meta BGP 2021, CrowdStrike 2024) are excellent learning material.

Tools: PagerDuty, Opsgenie, incident.io, FireHydrant, Rootly, Grafana OnCall, Statuspage/Instatus.

---

## 15. Tooling Landscape and Cost {#tools}

| Category | Open source | SaaS |
|---|---|---|
| Metrics | Prometheus, VictoriaMetrics, Mimir, Thanos | Datadog, New Relic, Grafana Cloud, Chronosphere, CloudWatch |
| Logs | Loki, Elasticsearch/OpenSearch, ClickHouse-based (SigNoz, Uptrace), Quickwit | Datadog, Splunk, Sumo Logic, Better Stack, Axiom |
| Traces | Jaeger, Tempo, Zipkin, SigNoz | Honeycomb, Datadog APM, Lightstep, New Relic |
| Collection | OTel Collector, Fluent Bit, Vector, Grafana Alloy | |
| Visualization | Grafana, Kibana/OpenSearch Dashboards | |
| Profiling | Pyroscope, Parca | Datadog, Polar Signals |
| Error tracking | Sentry (self-host), GlitchTip | Sentry, Rollbar, Bugsnag |
| Synthetic/uptime | Blackbox exporter, Uptime Kuma | Checkly, Pingdom, Datadog Synthetics |
| RUM (frontend) | OpenTelemetry JS, Grafana Faro | Datadog RUM, Sentry, New Relic Browser |

Cost control (observability bills can rival compute bills):
- Control metric cardinality (drop unused labels and series via relabeling, and aggregate at the collector).
- Sample traces (tail-based), sample or drop verbose logs, use log levels sensibly, and set retention tiers (hot 7 days, cold archive in S3).
- Route debug-level data to cheap storage, and keep high-value signals in expensive tools.
- Measure telemetry cost per team or service.

---

## 16. Interview Questions {#qa}

1. What's the difference between monitoring and observability?
2. Explain the RED and USE methods. What would you put on a service dashboard?
3. Why shouldn't you use user IDs as metric labels? Where should that data go?
4. Why are averages misleading for latency? Why can't you average p99s?
5. How does distributed tracing work across services and message queues? What is `traceparent`?
6. Head-based vs tail-based sampling?
7. Define SLI, SLO, SLA, and error budget. Compute the monthly error budget for 99.95%.
8. What is burn-rate alerting and why is it better than threshold alerts?
9. What makes a good alert? What causes alert fatigue?
10. Walk through how you'd run an incident and write a postmortem.
11. What is OpenTelemetry and why use it?
