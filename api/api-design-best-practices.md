# API Design Best Practices: The Details That Separate Good APIs from Painful Ones

> [`restapi.md`](restapi.md) covers REST basics. This note covers the production-grade details: resource modeling, error format, idempotency, pagination, filtering, concurrency control, versioning, long-running operations, webhooks, rate-limit headers, and documentation. Stripe, GitHub, Google AIP, and Microsoft's REST guidelines are the reference APIs.

## Table of Contents
1. [Resource Modeling and URLs](#resources)
2. [HTTP Methods: Safety and Idempotency](#methods)
3. [Status Codes That Matter](#status)
4. [Error Responses (RFC 9457 Problem Details)](#errors)
5. [Idempotency Keys](#idempotency)
6. [Pagination](#pagination)
7. [Filtering, Sorting, Sparse Fields, Expansion](#filtering)
8. [Concurrency Control: ETags and If-Match](#etag)
9. [HTTP Caching for APIs](#caching)
10. [Versioning and Evolution](#versioning)
11. [Long-Running Operations](#lro)
12. [Bulk and Batch Operations](#bulk)
13. [Rate Limiting Headers](#ratelimit)
14. [Webhooks (Outbound Events)](#webhooks)
15. [Request/Response Conventions](#conventions)
16. [OpenAPI and Contract-First Design](#openapi)
17. [Security Checklist for APIs](#security)
18. [API Review Checklist](#checklist)

---

## 1. Resource Modeling {#resources}

- **Nouns, plural, hierarchical where ownership is real**: `/customers/{id}/orders`. Keep nesting to one level, then use top-level resources with filters: `/orders?customer_id=…`.
- **IDs are opaque strings**, often prefixed by type (Stripe: `cus_…`, `pi_…`). Prefixes make logs debuggable and prevent mixing IDs across types.
- **Actions that aren't CRUD**: model them as sub-resources or state transitions:
  - `POST /orders/{id}/cancel` (pragmatic "custom method", Google AIP-136 uses `:cancel`).
  - Or create a resource representing the action: `POST /refunds { payment_id }`.
- Use **consistent casing** (snake_case like Stripe/GitHub, or camelCase like Google/Microsoft; pick one).
- Avoid leaking DB structure (join tables, internal flags).
- Model **states explicitly** (`status: "requires_payment" | "processing" | "succeeded" | "canceled"`) and document the transitions.

---

## 2. Methods: Safety and Idempotency {#methods}

| Method | Safe (no side effects) | Idempotent | Typical use |
|---|---|---|---|
| GET | ✅ | ✅ | Read |
| HEAD | ✅ | ✅ | Headers only |
| OPTIONS | ✅ | ✅ | CORS preflight, capabilities |
| PUT | ❌ | ✅ | Full replace (client knows the URL/ID) |
| DELETE | ❌ | ✅ | Delete (2nd call → 404 or 204; state is the same) |
| POST | ❌ | ❌ | Create (server assigns ID), actions |
| PATCH | ❌ | ❌ by spec (can be made idempotent) | Partial update: JSON Merge Patch (RFC 7386) or JSON Patch (RFC 6902) |
| QUERY (draft) | ✅ | ✅ | Safe query with a body (for complex searches); emerging standard |

Idempotency matters because **networks fail after the server acted**: the client times out and retries. Idempotent methods can be retried safely, while POST needs idempotency keys (§5).

---

## 3. Status Codes That Matter {#status}

| Code | Use |
|---|---|
| 200 OK | Success with body |
| 201 Created | Resource created; include `Location` header |
| 202 Accepted | Async processing started (see LRO) |
| 204 No Content | Success, no body (DELETE, some PUTs) |
| 301/308 | Permanent redirect (308 preserves method) |
| 304 Not Modified | Conditional GET cache hit |
| 400 Bad Request | Malformed syntax / generic validation failure |
| 401 Unauthorized | **Not authenticated** (missing/invalid credentials); include `WWW-Authenticate` |
| 403 Forbidden | Authenticated but **not allowed** |
| 404 Not Found | Doesn't exist **or** you may not know it exists (use 404 instead of 403 to avoid leaking existence) |
| 405 Method Not Allowed | Include `Allow` header |
| 406 / 415 | Unacceptable `Accept` / unsupported `Content-Type` |
| 409 Conflict | State conflict: duplicate, version mismatch (or 412), invalid state transition |
| 410 Gone | Permanently removed (deprecated endpoint after sunset) |
| 412 Precondition Failed | `If-Match` failed (optimistic concurrency) |
| 413 Content Too Large | Payload too big |
| 422 Unprocessable Content | Syntactically valid but semantically invalid (validation errors) |
| 428 Precondition Required | Require `If-Match` to prevent lost updates |
| 429 Too Many Requests | Rate limited; include `Retry-After` |
| 500 | Unexpected server error (never leak stack traces) |
| 502 / 503 / 504 | Bad gateway / unavailable (with `Retry-After` during maintenance or overload) / upstream timeout |

Rules: be consistent. **Don't return 200 with `{"error": …}`**, because it breaks clients, caches, monitoring, and retries.

---

## 4. Errors: RFC 9457 Problem Details {#errors}

Standard format (`Content-Type: application/problem+json`), which obsoletes RFC 7807:
```json
{
  "type": "https://api.example.com/problems/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Account acc_123 has balance 30.00 INR, but 50.00 INR is required.",
  "instance": "/payments/req_7f3a",
  "code": "insufficient_funds",
  "request_id": "req_7f3a9c",
  "errors": [
    { "field": "amount", "code": "exceeds_balance", "message": "Amount exceeds available balance" }
  ]
}
```
Guidelines:
- A **stable machine-readable `code`** clients can switch on. Messages are for humans and may change.
- **Field-level validation errors**, all at once (not one per round trip).
- A **request ID** (also in the `X-Request-Id` response header) that support and logs can correlate.
- Never leak internals (SQL errors, stack traces, hostnames).
- Document every error code.

---

## 5. Idempotency Keys {#idempotency}

Problem: the client POSTs `/payments`, the connection drops, and the client doesn't know if the charge happened. Retrying may double-charge.

Solution (Stripe's approach, now the IETF draft `Idempotency-Key` header):
```
POST /payments
Idempotency-Key: 6f1c8e2a-...-unique-per-logical-operation
```
Server algorithm:
1. Look up the key (scoped to the authenticated client + endpoint) in an idempotency store with a **unique constraint**.
2. **Not found**: insert `{key, request_hash, status: in_progress}` atomically, process the request, store the **response status and body**, then mark it complete. Ideally the business write and the key record are in the **same DB transaction**.
3. **Found & completed**: return the stored response (same status and body) without re-executing.
4. **Found & in progress**: return `409 Conflict` (or wait), since a concurrent duplicate is in flight.
5. **Found with a different request body hash**: return `422`, because the key was reused for a different request.
6. Expire keys after, e.g., 24 h.

Also make consumers idempotent internally: a unique constraint on `(merchant_id, external_reference)` and upserts. **Retries plus idempotency give effectively-once semantics.**

---

## 6. Pagination {#pagination}

| Style | Request | Pros | Cons |
|---|---|---|---|
| Offset | `?limit=20&offset=40` / `?page=3` | Jump to page, simple | Slow deep pages; duplicates/skips when data changes |
| **Cursor (keyset)** | `?limit=20&starting_after=ord_123` or opaque `?cursor=eyJ…` | Stable, O(1) per page | No random page access |
| Time-based | `?since=2025-06-01T00:00:00Z` | Sync use cases | Ties need a tiebreaker |

Response envelope:
```json
{
  "data": [ ... ],
  "has_more": true,
  "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNS0wNi0wMVQxMDowMDowMFoiLCJpZCI6Im9yZF8xMjMifQ"
}
```
Or the `Link` header (RFC 8288): `Link: <https://api.x.com/orders?cursor=abc>; rel="next"` (GitHub style).

Rules: a default and max `limit` (e.g., 20 / 100), deterministic sort with a unique tiebreaker, and opaque cursors (base64 of the sort keys; optionally signed or encrypted). Avoid `total_count` on big collections (it's expensive). Offer it as opt-in or an estimate.

---

## 7. Filtering, Sorting, Sparse Fields, Expansion {#filtering}

```
GET /orders?status=paid&created[gte]=2025-06-01&created[lt]=2025-07-01&sort=-created_at,id
GET /orders?fields=id,status,total                 (sparse fieldsets)
GET /orders/ord_1?expand[]=customer&expand[]=items.product   (Stripe-style expansion, avoid N round trips)
GET /products?q=running+shoes                       (free-text search)
```
- Allowlist filterable and sortable fields (they must be indexed!). Validate operators.
- Complex search: `POST /orders/search` with a JSON body (or the new `QUERY` method).
- Limit expansion depth (cost control, like GraphQL).
- Standard filter syntaxes exist (OData `$filter`, JSON:API `filter[...]`, Google AIP-160 filtering, RSQL/FIQL). Choose one consistently. See [`json-api.md`](json-api.md).

---

## 8. Concurrency Control: ETags {#etag}

Prevent **lost updates** when two clients edit the same resource:
```
GET /documents/doc_1
→ 200, ETag: "v7"

PUT /documents/doc_1
If-Match: "v7"
→ 200, ETag: "v8"                       (success)
→ 412 Precondition Failed               (someone else updated it → client must re-fetch and merge)
```
Backed by a version column (optimistic locking, see `databases/fundamentals/03`). Optionally require it with `428 Precondition Required`.

`If-None-Match: *` on PUT means "create only if it doesn't exist".

---

## 9. HTTP Caching for APIs {#caching}

- `Cache-Control: public, max-age=60, stale-while-revalidate=30` for shared, cacheable reads; `private` for user-specific responses; `no-store` for sensitive data.
- **Validators**: `ETag` / `Last-Modified` → conditional GET (`If-None-Match` / `If-Modified-Since`) returns `304` with no body, which saves bandwidth.
- `Vary: Authorization, Accept-Encoding` to avoid serving one user's cached response to another. Better: don't let CDNs cache authenticated responses unless you're certain.
- See [`caching/clientside/client-side-caching.md`](../caching/clientside/client-side-caching.md) and [`caching/cdn/cdn.md`](../caching/cdn/cdn.md).

---

## 10. Versioning and Evolution {#versioning}

**Backward-compatible (non-breaking) changes**: adding optional request fields, adding response fields, adding endpoints, adding enum values (only if clients are told to tolerate unknown values!), relaxing validation.

**Breaking changes**: removing or renaming fields, changing types and formats, making optional inputs required, changing error codes or status codes, changing default behavior or pagination, tightening validation, changing URLs.

Strategies:
| Strategy | Example | Notes |
|---|---|---|
| URL path | `/v1/orders` | Simple and visible; coarse |
| Header | `Accept: application/vnd.example.v2+json` or `API-Version: 2025-06-01` | Clean URLs |
| **Date-based versions pinned per account** | Stripe: `Stripe-Version: 2025-06-30` | Server keeps transformation layers (version change modules) that transform responses back to older shapes. Clients upgrade explicitly |
| No versioning, evolution only | GraphQL-style | Requires discipline |

Deprecation process: announce, add `Deprecation` and `Sunset` headers (RFC 8594/9745), log usage per client, contact heavy users, run brownouts (temporarily fail deprecated calls to surface remaining dependents), then remove.

**Robustness principle for clients**: ignore unknown fields, and handle unknown enum values gracefully.

---

## 11. Long-Running Operations {#lro}

For work that takes longer than an HTTP request should (video transcoding, reports, bulk imports):
```
POST /exports        → 202 Accepted
                       Location: /operations/op_123
                       { "id": "op_123", "status": "pending" }
GET /operations/op_123 → { "status": "running", "progress": 0.42 }
GET /operations/op_123 → { "status": "succeeded", "result": { "download_url": "https://…signed…" } }
```
- Clients poll (with `Retry-After` hints), or get a **webhook** on completion, or use SSE/WebSocket.
- Support cancellation (`POST /operations/op_123/cancel`).
- Make submission idempotent (idempotency key) so retries don't start duplicate jobs.

---

## 12. Bulk and Batch {#bulk}

- Bulk create: `POST /orders/batch` with up to N items, returning per-item results (`207 Multi-Status`-like body with status per item). Decide between **atomic** (all-or-nothing) and **partial success**, and document it.
- Large imports: upload a file to object storage, then start an LRO.
- Batching multiple different requests in one HTTP call (Google batch, Microsoft Graph `$batch`) is less necessary with HTTP/2 multiplexing.

---

## 13. Rate Limiting Headers {#ratelimit}

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Policy: "default";q=100;w=60
RateLimit: "default";r=0;t=30
```
- IETF `RateLimit` / `RateLimit-Policy` header fields (draft), or the widespread legacy trio `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
- Limit per API key/user/IP/tenant, and per endpoint class (writes vs reads, expensive search).
- Algorithms (token bucket, sliding window) are implemented in [`system-design/awesome-notes/implementations/`](../system-design/awesome-notes/implementations/). Distributed limits typically use Redis + Lua scripts.
- Return quota info on **every** response so clients can self-throttle.

---

## 14. Webhooks {#webhooks}

Webhooks push events to customer endpoints (`POST https://customer.com/hooks`):
```json
{ "id": "evt_123", "type": "payment.succeeded", "created": 1717236000, "api_version": "2025-06-01",
  "data": { "object": { "id": "pay_9", "amount": 5000, "currency": "inr" } } }
```
Sender responsibilities:
1. **Sign** payloads: `Webhook-Signature: t=1717236000,v1=HMAC_SHA256(secret, t + "." + body)`. Include a timestamp to prevent replay, and support secret rotation (multiple valid signatures). The **Standard Webhooks** spec standardizes this.
2. **At-least-once delivery** with retries and exponential backoff over hours or days, and a dead-letter state plus a dashboard for manual replay.
3. **Unique event IDs** so receivers can deduplicate.
4. Events may arrive **out of order**: include timestamps and versions, or send "thin" events (just IDs), so receivers fetch the latest state via the API.
5. Short timeouts (e.g., 5–10 s). Treat 2xx as success. Disable endpoints that fail persistently, and notify the owner.
6. Prevent **SSRF**: validate customer URLs (block private IP ranges, cloud metadata `169.254.169.254`), resolve DNS carefully (DNS rebinding), and send from egress proxies with fixed IPs that customers can allowlist.
7. Deliver from an outbox/queue, never inline in the request that caused the event.

Receiver best practices: verify the signature **on the raw body** (before JSON parsing), respond 2xx quickly and process asynchronously (queue), make processing idempotent by event ID, and reconcile periodically via the API in case events are missed.

---

## 15. Conventions {#conventions}

- **JSON**: UTF-8, `Content-Type: application/json`. Consistent naming. Dates as **RFC 3339** UTC strings (`2025-06-01T10:00:00Z`). Money as integer minor units + currency (Stripe) or decimal strings. Never use floats for money.
- **Null vs absent**: define semantics (especially for PATCH: absent = unchanged, null = clear).
- **Enums** as lowercase strings, and document that new values may be added.
- **Envelope**: consistent list envelope (`data`, pagination fields). Avoid wrapping single resources unnecessarily.
- **Request IDs** (`X-Request-Id` in/out) and **W3C `traceparent`** propagation.
- **Compression**: gzip/br for large responses.
- **Timeouts**: document server-side limits. Clients should set their own.
- **Locale/time zone** handling explicit (`Accept-Language`).
- **CORS** configured narrowly (see [`api-security/cors.md`](../api-security/cors.md)).
- Health endpoints (`/healthz` for liveness, `/readyz` for readiness) are separate from public APIs.

---

## 16. OpenAPI and Contract-First {#openapi}

```yaml
openapi: 3.1.0
info: { title: Orders API, version: "2025-06-01" }
paths:
  /orders/{id}:
    get:
      operationId: getOrder
      parameters:
        - { name: id, in: path, required: true, schema: { type: string, pattern: "^ord_" } }
      responses:
        "200": { description: OK, content: { application/json: { schema: { $ref: "#/components/schemas/Order" } } } }
        "404": { $ref: "#/components/responses/NotFound" }
components:
  schemas:
    Order:
      type: object
      required: [id, status, total, currency]
      properties:
        id: { type: string }
        status: { type: string, enum: [pending, paid, shipped, cancelled] }
        total: { type: integer, description: "Minor units" }
        currency: { type: string, example: "INR" }
```
- **Contract-first**: write the spec, review it as an API design doc, generate server stubs and client SDKs (openapi-generator, oapi-codegen, Kiota, Speakeasy, Stainless), mock servers (Prism), and docs (Redoc, Scalar, Swagger UI).
- **Code-first** generates the spec from annotations (FastAPI, springdoc, NestJS Swagger, tsoa). Fast, but design reviews happen later.
- Lint specs with **Spectral** (style rules), detect breaking changes in CI (oasdiff, openapi-diff), and run contract tests (Schemathesis fuzzing, Dredd, Pact for consumer-driven contracts).
- AsyncAPI is the equivalent for event-driven APIs (Kafka topics, webhooks).

---

## 17. Security Checklist {#security}

Expanded in [`api-security/api-security.md`](../api-security/api-security.md). OWASP API Security Top 10 (2023):
1. **Broken Object Level Authorization (BOLA/IDOR)**: check ownership on every object access.
2. Broken Authentication.
3. **Broken Object Property Level Authorization**: mass assignment (client sets `is_admin: true`) and excessive data exposure. Use explicit allowlists of writable and readable fields.
4. Unrestricted Resource Consumption: rate limits, pagination caps, payload size limits, timeouts.
5. Broken Function Level Authorization (admin endpoints reachable by users).
6. Unrestricted Access to Sensitive Business Flows (bots buying all tickets): business-level throttling and bot detection.
7. **SSRF** (webhooks, URL fetch features).
8. Security Misconfiguration (verbose errors, permissive CORS, missing TLS).
9. Improper Inventory Management (forgotten old versions and debug endpoints).
10. Unsafe Consumption of third-party APIs (validate their responses too).

---

## 18. API Review Checklist {#checklist}

- [ ] Resources and names consistent; IDs opaque and prefixed.
- [ ] Correct methods and status codes; no 200-with-error.
- [ ] Errors in Problem Details format with stable codes and request IDs.
- [ ] POST creates/actions accept `Idempotency-Key`.
- [ ] Lists are cursor-paginated with max limits.
- [ ] Filters and sorts are allowlisted and backed by indexes.
- [ ] Optimistic concurrency (ETag/If-Match) where concurrent edits are possible.
- [ ] Caching headers set intentionally.
- [ ] Authn on every endpoint; authz on every object; field-level allowlists.
- [ ] Rate limits plus headers; payload size limits; timeouts.
- [ ] Backward compatibility verified (oasdiff in CI); deprecation plan.
- [ ] Long operations are async (202 + operation resource or webhooks).
- [ ] Webhooks signed, retried, deduplicable, SSRF-safe.
- [ ] OpenAPI spec complete with examples; SDKs and docs generated.
- [ ] Observability: per-endpoint latency/error metrics, traces, structured logs with request IDs.
