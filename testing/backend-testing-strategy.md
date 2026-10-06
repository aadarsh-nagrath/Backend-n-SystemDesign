# Backend Testing Strategy: From Unit Tests to Production Verification

> Interview Q&A on testing: [`interview-prep/backend-engineer/09-testing-quality.md`](../interview-prep/backend-engineer/09-testing-quality.md). This note covers a complete, practical testing strategy for backend services.

## Table of Contents
1. [Why and What We Test](#why)
2. [Test Shapes: Pyramid, Trophy, Honeycomb](#shapes)
3. [Unit Tests](#unit)
4. [Test Doubles: Dummies, Stubs, Fakes, Mocks, Spies](#doubles)
5. [Integration Tests with Real Dependencies (Testcontainers)](#integration)
6. [Testing the Database Layer](#db)
7. [API / Component Tests](#api)
8. [Contract Testing (Consumer-Driven Contracts)](#contract)
9. [End-to-End Tests](#e2e)
10. [Testing Asynchronous and Event-Driven Code](#async)
11. [Property-Based Testing and Fuzzing](#property)
12. [Mutation Testing](#mutation)
13. [Performance and Load Testing](#performance)
14. [Testing in Production: Canaries, Synthetic Monitoring, Chaos](#production)
15. [Flaky Tests](#flaky)
16. [Test Data Management](#data)
17. [Coverage, CI, and Test Suite Health](#ci)
18. [Interview Questions](#qa)

---

## 1. Why and What {#why}

Tests exist to give **fast, reliable feedback** so you can change code confidently. Good tests:
- **Fail when behavior breaks** and **don't fail otherwise** (they're resistant to refactoring and test behavior, not implementation details).
- **Are fast**, so they run constantly.
- **Are deterministic** (no flakiness).
- **Are readable**: they document intended behavior (Arrange–Act–Assert / Given–When–Then).

Kent Beck's test desiderata: isolated, composable, deterministic, fast, writable, readable, behavioral, structure-insensitive, automated, specific, predictive, inspiring.

---

## 2. Test Shapes {#shapes}

- **Test pyramid** (Mike Cohn): many unit tests, fewer integration tests, few E2E tests. Faster and cheaper at the bottom.
- **Testing trophy** (Kent C. Dodds): emphasizes **integration tests** as the best confidence per cost, with static analysis (types, linters) as the base.
- **Microservices honeycomb** (Spotify): mostly **integration tests of each service** (service with real DB, mocked external services), few implementation-detail unit tests, and few integrated E2E tests.

Practical blend for a backend service:
| Layer | Share | What |
|---|---|---|
| Static analysis | Always | Type checking (mypy/pyright, TS strict), linters, formatters, security linters (Semgrep, gosec, Bandit) |
| Unit | Many | Pure domain logic, calculations, validation, state machines: no I/O, milliseconds |
| Integration / component | Many | Service + real DB/cache/broker in containers, external HTTP stubbed; through the API boundary |
| Contract | Per integration point | Consumer-driven contracts for service APIs and events |
| E2E | Few, critical journeys | Deployed system, real flows (checkout, signup) |
| Production verification | Continuous | Canary analysis, synthetic checks, monitoring |

---

## 3. Unit Tests {#unit}

- Target **domain logic** with no I/O: business rules, pricing, validation, parsing, state transitions. Hexagonal architecture makes this easy (see [`architecture/02`](../architecture/02-clean-hexagonal-ddd.md)).
- A "unit" is a **unit of behavior**, not necessarily one class. Testing a cluster of collaborating domain objects together is fine.
- Test **public behavior**, not private methods or internal call sequences.
- Inject time, randomness, and IDs (`Clock`, `IdGenerator`) to make tests deterministic.
- Table-driven / parameterized tests for edge cases (Go table tests, pytest.mark.parametrize, JUnit `@ParameterizedTest`).
- Edge cases to always consider: empty, null/None, zero, negative, max values, boundaries (off-by-one), unicode, time zones and DST, leap years, concurrency (if applicable), very large inputs, duplicates.

```python
@pytest.mark.parametrize("subtotal, coupon, expected", [
    (Money("100.00","INR"), None,           Money("100.00","INR")),
    (Money("100.00","INR"), Percent(10),    Money("90.00","INR")),
    (Money("100.00","INR"), Flat("150.00"), Money("0.00","INR")),   # never negative
])
def test_apply_coupon(subtotal, coupon, expected):
    assert apply_coupon(subtotal, coupon) == expected
```

---

## 4. Test Doubles {#doubles}

Gerard Meszaros's taxonomy:
| Double | Purpose | Example |
|---|---|---|
| **Dummy** | Fills a parameter, never used | `None` logger |
| **Stub** | Returns canned answers | `PaymentGateway` stub always returns "approved" |
| **Fake** | Working lightweight implementation | `InMemoryOrderRepository`, fake clock, SQLite instead of PG (beware semantic differences!) |
| **Mock** | Pre-programmed expectations about calls; verifies interactions | "expect `send_email` called once with X" |
| **Spy** | Records calls for later assertions | Capture published events |

Guidelines:
- Prefer **fakes and stubs over mocks** for most collaborators. Excessive mocking couples tests to implementation (refactor → tests break while behavior is unchanged) and can make tests pass even when the real integration is broken.
- **Mock only what you own** at architectural boundaries (ports). Wrap third-party SDKs in your own adapter, and test the adapter separately against the real thing or a high-fidelity stub.
- Verify interactions (mocks) only when the interaction **is** the behavior (an email must be sent, an event must be published).
- **Don't mock the database** for data-access code. Test against a real database (§6).
- Classicist (Detroit) vs mockist (London) schools: classicists use real objects where possible, and mockists isolate every class. Most backend teams do best leaning classicist, with fakes at the I/O boundaries.

---

## 5. Integration Tests with Testcontainers {#integration}

**Testcontainers** (Java, Go, Python, Node, .NET, Rust…) starts real dependencies in Docker for tests: PostgreSQL, MySQL, Redis, Kafka, RabbitMQ, LocalStack (AWS), Elasticsearch, and more.

```java
@Testcontainers
class OrderRepositoryIT {
  @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17-alpine");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
  }

  @Test void savesAndLoadsOrderWithLines() { /* real SQL, real constraints, real migrations */ }
}
```
Why real dependencies:
- They catch SQL errors, migration bugs, constraint violations, transaction and isolation behavior, JSON operator differences, collation and time zone issues, and driver quirks, none of which an in-memory fake or H2/SQLite will show.
- Use the **same major version** as production.

Speed tips: start containers once per test suite (singleton container pattern), run each test in a transaction rolled back at the end (or truncate tables between tests), parallelize with separate schemas or databases per worker, use reusable containers locally, and template databases (`CREATE DATABASE test_x TEMPLATE migrated_template` in PG is fast).

External HTTP APIs (Stripe, Twilio): stub with **WireMock**, MockServer, `nock` (Node), `responses`/`respx` (Python), `httptest` (Go), or recorded fixtures (VCR.py, Polly.js). Also run a small set of tests against the provider's sandbox in a separate pipeline.

---

## 6. Testing the Database Layer {#db}

Test:
- Repository queries return the correct results, including edge cases (NULLs, ordering, pagination boundaries, soft-deleted rows, tenant isolation).
- **Constraints work**: unique violations map to domain errors, FKs, CHECKs.
- **Migrations**: apply cleanly from scratch *and* from the previous production version, and roll back if you support down migrations. Lint them for locking hazards.
- **Concurrency**: two transactions racing (optimistic locking conflicts, `SELECT … FOR UPDATE`, upsert races). Use real concurrent connections in the test (threads/goroutines plus barriers).
- **Query performance guards**: assert the query count per request to catch N+1 (Django `assertNumQueries`, `nplusone`, Bullet; or count via a driver hook), and optionally assert that `EXPLAIN` uses an index for critical queries.
- RLS policies (see [`databases/postgresql/10-security-roles-rls.md`](../databases/postgresql/10-security-roles-rls.md)): tenant A can't see tenant B.
- pgTAP for testing database functions and policies in SQL.

---

## 7. API / Component Tests {#api}

Test the service **through its public interface** (HTTP/gRPC) with real infrastructure in containers and external services stubbed:
```python
def test_create_order_returns_201_and_persists(client, db, stripe_stub):
    stripe_stub.charge_succeeds()
    resp = client.post("/orders", json={"items": [{"sku": "A1", "qty": 2}]},
                       headers={"Authorization": token_for("u_1"), "Idempotency-Key": "k1"})
    assert resp.status_code == 201
    order_id = resp.json()["id"]
    assert db.fetch_one("SELECT status FROM orders WHERE id = %s", order_id)["status"] == "paid"

    # idempotent retry returns the same response, no second charge
    resp2 = client.post("/orders", json={"items": [{"sku": "A1", "qty": 2}]},
                        headers={"Authorization": token_for("u_1"), "Idempotency-Key": "k1"})
    assert resp2.json()["id"] == order_id
    assert stripe_stub.charge_count() == 1
```
Cover: happy paths, validation errors (400/422 with error codes), auth (401 without a token, 403 for another user's resource, which is the IDOR test), not found, conflicts (409), idempotency, pagination, rate limiting, and error mapping when dependencies fail (stub returns 500 or times out → your API returns 502/503 with a proper body, retries happen, the circuit opens).

Schema conformance: validate responses against the OpenAPI spec in tests (Schemathesis can generate tests from the spec, including fuzzing).

---

## 8. Contract Testing {#contract}

Problem: service A (consumer) depends on service B's (provider's) API. Each team's tests pass in isolation, B changes a field, and production breaks. Full E2E environments to catch this are slow and flaky.

**Consumer-driven contract testing** (**Pact**):
1. The consumer's tests define expectations (the requests it makes and the minimal response fields it uses) against a Pact mock server, which generates a **contract file** (pact).
2. The contract is published to a **Pact Broker** (or PactFlow).
3. The provider's CI **verifies** the contract by replaying the requests against the real provider (with state setup hooks) and checking the responses satisfy it.
4. **`can-i-deploy`** checks the broker: is this version compatible with the versions of its consumers/providers in the target environment?

Benefits: fast, independent pipelines, and providers know exactly which fields consumers rely on. It also works for **message contracts** (Kafka/SQS events).

Alternatives: provider-driven schemas with compatibility checks (OpenAPI diff in CI, protobuf `buf breaking`, schema registry compatibility for Avro/Protobuf events), Spring Cloud Contract.

---

## 9. End-to-End Tests {#e2e}

- Test critical **user journeys** across the deployed system (signup → login → add to cart → checkout → confirmation email).
- Few and stable: they're slow and flaky, and failures are hard to diagnose.
- Run against staging or ephemeral environments, plus a subset as **production smoke tests** after deploys.
- Tools: Playwright/Cypress (with UI), k6/REST-assured/Postman-Newman/Karate (API-level E2E), plus test accounts and data isolation in shared environments.
- Make failures debuggable: capture trace IDs, logs, and screenshots.

---

## 10. Testing Async and Event-Driven Code {#async}

- **Test handlers as plain functions**: given an event, assert the resulting state changes and emitted events (no broker needed).
- **Idempotency tests**: deliver the same message twice and assert a single effect.
- **Out-of-order tests**: deliver v2 before v1, and assert the correct final state.
- **Integration with a real broker** (Kafka/RabbitMQ in Testcontainers): the produce → consume → DB state round trip.
- **Eventually consistent assertions**: poll with timeout instead of `sleep(5)` (Awaitility in Java, `eventually` helpers, Go `require.Eventually`):
  ```java
  await().atMost(Duration.ofSeconds(10)).untilAsserted(() ->
      assertThat(orderRepo.find(id).status()).isEqualTo(PAID));
  ```
- **Outbox tests**: the business row and the outbox row are written in the same transaction, and a rollback leaves neither.
- **Saga tests**: simulate each step's failure and assert the compensations run.
- **Workflow engines**: Temporal has a test environment with time skipping (test a 30-day timer in milliseconds).
- **Time-dependent logic**: inject the clock and advance it manually.

---

## 11. Property-Based Testing and Fuzzing {#property}

**Property-based testing**: instead of hand-picked examples, state **properties** that must hold for all inputs, and let the framework generate hundreds of random inputs and **shrink** failures to minimal counterexamples.
```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_idempotent_and_ordered(xs):
    once = my_sort(xs)
    assert my_sort(once) == once
    assert all(a <= b for a, b in zip(once, once[1:]))
    assert sorted(xs) == once

@given(st.text())
def test_encode_decode_roundtrip(s):
    assert decode(encode(s)) == s
```
Common properties: round-trip (serialize/deserialize, encode/decode), invariants (balance never negative after any sequence of operations), idempotency (f(f(x)) = f(x)), commutativity, equivalence with a simple reference implementation (an oracle), and metamorphic relations.

**Stateful/model-based testing**: generate random sequences of operations against your system and a simple model, and compare (Hypothesis stateful testing, jqwik, QuickCheck state machines). Great for caches, queues, and state machines.

Tools: Hypothesis (Python), jqwik (Java), fast-check (JS/TS), gopter/rapid (Go), FsCheck (.NET), proptest (Rust).

**Fuzzing**: feed random or mutated inputs to find crashes and security bugs, especially in parsers, decoders, and input handling. Go has native fuzzing (`go test -fuzz`), plus libFuzzer/AFL++, Jazzer (Java), Atheris (Python), cargo-fuzz, and OSS-Fuzz. API fuzzing: Schemathesis, RESTler.

**Deterministic simulation testing** (FoundationDB, TigerBeetle, Antithesis): run the whole distributed system in a single-threaded deterministic simulator with injected faults, which makes rare bugs reproducible.

---

## 12. Mutation Testing {#mutation}

Coverage tells you which lines executed, not whether tests would **catch bugs** there. Mutation testing introduces small bugs (**mutants**: `>` → `>=`, `+` → `-`, remove a call, return null) and checks whether some test fails (the mutant is "killed").
- **Mutation score** = killed / total mutants. Surviving mutants reveal weak assertions.
- Tools: PIT (Java), Stryker (JS/TS, .NET, Scala), mutmut/cosmic-ray (Python), go-mutesting.
- It's expensive, so run it on core domain modules or nightly, not on every PR for the whole codebase.

---

## 13. Performance and Load Testing {#performance}

Types:
| Type | Question |
|---|---|
| **Load test** | Does it meet latency/throughput targets at expected peak load? |
| **Stress test** | Where does it break, and how (gracefully shedding load, or collapsing)? |
| **Soak / endurance** | Does it degrade over hours (memory leaks, connection leaks, disk filling, GC)? |
| **Spike test** | Sudden 10× surge (flash sale)? Does autoscaling react in time? |
| **Capacity test** | How many instances are needed per X rps (for capacity planning)? |
| **Benchmark / microbenchmark** | Function-level speed (JMH, Go `testing.B`, pytest-benchmark, criterion) |

Tools: **k6** (JS scripts, great CI integration), **Gatling** (Scala/Java/Kotlin), **Locust** (Python), JMeter, wrk2/Vegeta (simple HTTP load with correct latency measurement), ghz (gRPC), plus cloud services (Grafana Cloud k6, Azure Load Testing, Distributed Load Testing on AWS).

```js
// k6: ramp to 500 virtual users, assert SLOs
import http from 'k6/http';
import { check } from 'k6';
export const options = {
  scenarios: { checkout: { executor: 'ramping-arrival-rate', startRate: 50, timeUnit: '1s',
    preAllocatedVUs: 200, stages: [{ target: 500, duration: '5m' }, { target: 500, duration: '10m' }] } },
  thresholds: { http_req_failed: ['rate<0.001'], http_req_duration: ['p(95)<300', 'p(99)<800'] },
};
export default function () {
  const r = http.post(`${__ENV.BASE}/orders`, JSON.stringify({ items: [{ sku: 'A1', qty: 1 }] }),
    { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${__ENV.TOKEN}` } });
  check(r, { 'status 201': (res) => res.status === 201 });
}
```
Practices:
- Use an **open model** (arrival rate) for internet traffic, not a closed model (fixed users waiting for responses), which hides queueing (coordinated omission).
- Production-like data volumes and distributions (a 1,000-row test DB is meaningless), realistic traffic mix, warmed caches (and also a cold-cache test).
- Observe the system under test (dashboards, traces, profiles) to find the bottleneck, not just client-side numbers.
- Run performance regression tests in CI for critical paths (with thresholds), and do bigger tests before major launches.

---

## 14. Testing in Production {#production}

Pre-production tests can't reproduce production fully. Complement them with:
- **Canary deployments + automated analysis** (compare canary vs baseline metrics).
- **Feature flags** with internal-users-first rollout ("dogfooding").
- **Synthetic monitoring**: scripted probes of critical journeys every minute from multiple regions (Checkly, Datadog Synthetics, Grafana synthetic monitoring), alerting on failure.
- **Shadow traffic / dark launches**: run new code on real traffic without exposing results (compare outputs: GitHub's Scientist).
- **Chaos engineering** (see [`distributed-systems/05-resilience-patterns.md`](../distributed-systems/05-resilience-patterns.md#chaos)).
- **Observability-driven verification** after every deploy.
- Use clearly marked **test tenants/accounts** in production, excluded from billing and analytics.

---

## 15. Flaky Tests {#flaky}

A flaky test passes and fails without code changes. It destroys trust ("just re-run it"), hides real failures, and wastes CI time.

Common causes and fixes:
| Cause | Fix |
|---|---|
| Timing/sleeps (`sleep(1)` then assert) | Poll with timeout (eventually); control time with fake clocks |
| Shared state between tests (DB rows, globals, singletons, caches) | Isolate: transactions per test, unique IDs per test, reset state |
| Test order dependence | Randomize order (pytest-randomly) to expose it; make tests independent |
| Real network / external services | Stub them; use containers |
| Concurrency races in code | Real bugs! Investigate (race detector: `go test -race`, TSan) |
| Non-deterministic data (random, UUIDs, unordered results) | Seeded randomness; `ORDER BY` in queries; compare sets not lists |
| Time zones, DST, "today" | Inject the clock; fixed TZ in CI |
| Resource limits in CI (ports, memory, slow containers) | Dynamic ports, generous but bounded timeouts |

Process: detect flakies (CI re-run analytics, BuildPulse, Datadog CI Visibility, Gradle/Bazel flaky detection), **quarantine** them immediately (move to a non-blocking suite with an owner and deadline), fix the root cause, and track the flake rate as a team metric.

---

## 16. Test Data Management {#data}

- **Builders / factories** (factory_boy, FactoryBot, Fishery, Test Data Builders, Instancio) create valid objects with sensible defaults, overriding only what the test cares about. This keeps tests readable and resilient to schema changes.
- Avoid huge shared fixture files that every test depends on (a fragile "mystery guest").
- Each test creates its own data, with unique identifiers to allow parallel runs.
- **Seed data** for local and dev environments via scripts (versioned).
- **Production data in lower environments**: only anonymized or masked (PII), and comply with regulations. Use subsetting tools (Tonic, Neosync, pg_anonymizer, Greenmask) or synthetic data generation.
- Golden files / snapshot tests (approval testing) for complex outputs (rendered emails, API responses), reviewed carefully on update.

---

## 17. Coverage, CI, and Suite Health {#ci}

- **Coverage** is a useful *negative* indicator (0% coverage on payment code is bad), but 100% coverage doesn't mean well tested. Track branch coverage on critical modules, and don't game the number with assertion-free tests.
- **Fast CI**: parallelize (sharding), cache dependencies and containers, run only affected tests in monorepos (Nx, Bazel, Turborepo, Gradle), and keep the PR suite under ~10 minutes.
- **Tests as code quality gates**: required checks before merge, plus pre-commit hooks for fast checks.
- **Test naming** describes behavior: `test_cancel_order_after_shipment_raises_not_cancellable`.
- **Each bug fix gets a regression test** reproducing the bug first (red → green).
- **TDD** (red-green-refactor) is valuable especially for domain logic and bug fixes: tests drive design toward testable, decoupled code.
- Watch for slow tests, flaky rates, and test-code duplication. Refactor test code as seriously as production code.

---

## 18. Interview Questions {#qa}

1. Describe your testing strategy for a microservice with a Postgres DB, a Kafka consumer, and a third-party payment API.
2. Mocks vs stubs vs fakes: when do you use each? What's wrong with mocking everything?
3. Why test against a real database instead of an in-memory substitute?
4. What is contract testing and what problem does it solve?
5. How do you test eventually consistent/asynchronous behavior without flaky sleeps?
6. What is property-based testing? Give properties you'd test for a money/currency library.
7. What is mutation testing and why is coverage alone insufficient?
8. Load vs stress vs soak testing? What is coordinated omission?
9. How do you deal with flaky tests?
10. How do you test that an API is safe against IDOR (accessing other users' resources)?
