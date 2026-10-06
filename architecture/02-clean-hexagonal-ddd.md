# Code Architecture: Layers, Hexagonal/Clean Architecture, and Domain-Driven Design

> OOP design patterns (GoF) are covered in [`os-sysdesign-ipc.md`](../os-sysdesign-ipc.md) §2 and [`interview-prep/sde-general/02-oop-design-patterns.md`](../interview-prep/sde-general/02-oop-design-patterns.md). This note is about structuring a backend codebase so business logic stays testable, understandable, and independent of frameworks and infrastructure.

## Table of Contents
1. [Why Code Architecture Matters](#why)
2. [Classic Layered Architecture](#layered)
3. [Hexagonal Architecture (Ports and Adapters)](#hexagonal)
4. [Clean Architecture and Onion Architecture](#clean)
5. [Vertical Slice Architecture](#vertical)
6. [DDD Strategic Design: Bounded Contexts, Context Maps, Ubiquitous Language](#strategic)
7. [DDD Tactical Design: Entities, Value Objects, Aggregates, Repositories, Domain Services, Domain Events](#tactical)
8. [Aggregate Design Rules](#aggregates)
9. [Application Services and Use Cases](#application)
10. [Transactions, Persistence, and ORMs in a Clean Design](#persistence)
11. [Validation: Where Does It Go?](#validation)
12. [Error Handling Strategy](#errors)
13. [Dependency Injection](#di)
14. [SOLID, Briefly, with Backend Examples](#solid)
15. [A Reference Project Structure](#structure)
16. [When This Is Overkill](#overkill)
17. [Interview Questions](#qa)

---

## 1. Why {#why}

Frameworks, databases, and APIs change. Business rules live for years. Good code architecture:
- Keeps **business logic independent** of frameworks, DBs, and transport (HTTP/gRPC/queue), so it's testable in milliseconds without infrastructure.
- Makes **dependencies point inward** toward the stable domain.
- Makes the codebase's **structure reveal its intent** ("screaming architecture": the top-level folders say "orders, payments", not "controllers, models").
- Localizes change: swapping Postgres for DynamoDB, or REST for gRPC, touches adapters only.

---

## 2. Classic Layered Architecture {#layered}

```
Presentation (controllers, views, API handlers)
      ↓
Business / Service layer
      ↓
Data access layer (repositories, DAOs, ORM)
      ↓
Database
```
It's simple and familiar (typical Spring/Django/Rails/Express apps).

Problems as apps grow:
- Business logic depends on the data layer (and often on ORM entities), so the **domain is shaped by the database**.
- "Anemic domain model": entities are bags of getters and setters, and all logic lives in giant `*Service` classes (transaction scripts).
- Layers get skipped (controllers calling repositories directly).
- Organized by technical concern, so a feature change touches every layer's folder.

That's fine for CRUD-heavy apps. For complex domains, invert the dependencies.

---

## 3. Hexagonal Architecture (Ports and Adapters) {#hexagonal}

Alistair Cockburn (2005). The application core sits in the middle and interacts with the outside world only through **ports** (interfaces owned by the core). **Adapters** implement or call these ports for specific technologies.

```
          Driving (primary) side                                  Driven (secondary) side
   ┌─────────────────────────┐                             ┌──────────────────────────────┐
   │ HTTP controller (adapter)│──► [Inbound port:          │ [Outbound port: OrderRepository]◄── PostgresOrderRepository (adapter)
   │ gRPC handler   (adapter)│      PlaceOrderUseCase] ──► │ [Outbound port: PaymentGateway] ◄── StripePaymentGateway (adapter)
   │ Kafka consumer (adapter)│          APPLICATION CORE   │ [Outbound port: EventPublisher] ◄── OutboxEventPublisher (adapter)
   │ CLI / test     (adapter)│      (domain + use cases)   │ [Outbound port: Clock]          ◄── SystemClock / FixedClock (test)
   └─────────────────────────┘                             └──────────────────────────────┘
```
- **Inbound (driving) ports**: use-case interfaces the outside world calls (`PlaceOrder`, `CancelOrder`).
- **Outbound (driven) ports**: interfaces the core needs (`OrderRepository`, `PaymentGateway`, `Clock`, `IdGenerator`).
- **The core depends on nothing external.** Adapters depend on the core.
- Tests drive the core through inbound ports with in-memory fakes for the outbound ports, which makes them fast and deterministic.

```python
# core/ports.py
class OrderRepository(Protocol):
    def get(self, order_id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class PaymentGateway(Protocol):
    def charge(self, customer_id: str, amount: Money, idempotency_key: str) -> PaymentResult: ...

# core/use_cases.py
class PlaceOrder:
    def __init__(self, orders: OrderRepository, payments: PaymentGateway, events: EventPublisher, clock: Clock):
        self.orders, self.payments, self.events, self.clock = orders, payments, events, clock

    def execute(self, cmd: PlaceOrderCommand) -> OrderId:
        order = Order.place(cmd.customer_id, cmd.items, now=self.clock.now())   # domain logic & invariants
        result = self.payments.charge(cmd.customer_id, order.total, idempotency_key=cmd.request_id)
        order.mark_paid(result.payment_id)
        self.orders.save(order)
        self.events.publish(order.pull_events())
        return order.id

# adapters/http.py: FastAPI route builds PlaceOrderCommand from request, calls use case, maps result to HTTP
# adapters/postgres_repo.py: implements OrderRepository with SQL / ORM, mapping rows <-> domain objects
```

---

## 4. Clean and Onion Architecture {#clean}

Robert C. Martin's **Clean Architecture** (2012) and Jeffrey Palermo's **Onion Architecture** (2008) express the same idea as concentric circles:
```
        ┌──────────────── Frameworks & Drivers (web, DB, UI, devices, external interfaces)
        │  ┌───────────── Interface Adapters (controllers, presenters, gateways, repositories impl)
        │  │  ┌────────── Application Business Rules (use cases / interactors)
        │  │  │  ┌─────── Enterprise Business Rules (entities / domain model)
```
**The Dependency Rule**: source code dependencies point **only inward**. Inner circles know nothing about outer ones (no framework annotations, no SQL, no HTTP in the domain).

Data crossing boundaries uses simple structures (DTOs, commands, results), not ORM entities or HTTP request objects.

Hexagonal, Onion, and Clean are variations on one principle: **domain at the center, infrastructure at the edges, dependency inversion at the boundaries**.

---

## 5. Vertical Slice Architecture {#vertical}

Jimmy Bogard: organize by **feature/use case** (a slice through all layers), not by technical layer:
```
features/
  place_order/      handler.py, request.py, validator.py, tests.py
  cancel_order/
  get_order_history/  (may use raw SQL directly — reads don't need the domain model)
```
- Each slice can choose its own approach: rich domain model for complex commands, a simple SQL query for reads (pairs naturally with CQRS).
- Minimizes coupling between features. Changes are local.
- Shared domain model where rules are shared. Avoid premature abstraction.
- Common with mediator libraries (MediatR in .NET), and in Go and Node codebases.

Hexagonal boundaries (ports for infrastructure) combine well with vertical slices inside the core.

---

## 6. DDD Strategic Design {#strategic}

Eric Evans, *Domain-Driven Design* (2003). Vaughn Vernon, *Implementing DDD* (2013) and *DDD Distilled*.

- **Ubiquitous language**: developers and domain experts use the **same precise terms** in conversation, code, tests, and docs. If the business says "policy holder", the class isn't `User`.
- **Domain / subdomains**:
  - **Core domain**: your competitive advantage (pricing engine for an insurer, matching for a marketplace). Invest the best engineers and DDD here.
  - **Supporting subdomains**: necessary, specific to you, but not differentiating.
  - **Generic subdomains**: solved problems (auth, email, payments). Buy or use off-the-shelf.
- **Bounded context**: a boundary within which a model and its language are consistent. The same word can mean different things in different contexts ("Account" in Banking vs Identity). **Bounded contexts are the best candidates for module and service boundaries.**
- **Context map**: relationships between contexts:
  | Relationship | Meaning |
  |---|---|
  | Partnership | Two teams coordinate closely |
  | Shared Kernel | Small shared model (risky coupling) |
  | Customer–Supplier | Upstream serves downstream's needs |
  | Conformist | Downstream adopts upstream's model as-is |
  | **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model to protect its own (essential with legacy systems and third-party APIs) |
  | Open Host Service + Published Language | Upstream offers a well-documented protocol (public API/events) for many consumers |
  | Separate Ways | No integration |
- **Event storming** (Alberto Brandolini): a collaborative workshop to discover domain events, commands, aggregates, policies, and bounded contexts using sticky notes on a timeline.

---

## 7. DDD Tactical Patterns {#tactical}

| Building block | What | Example |
|---|---|---|
| **Entity** | Has identity that persists through state changes; equality by ID | `Order(id=ord_1)` |
| **Value object** | Defined by attributes, **immutable**, equality by value, self-validating | `Money(amount=499.00, currency=INR)`, `Email`, `Address`, `DateRange` |
| **Aggregate** | A cluster of entities and value objects treated as one consistency unit, with one **aggregate root** as the only entry point | `Order` (root) containing `OrderLine`s |
| **Repository** | Collection-like interface to load and save **whole aggregates** | `OrderRepository.get(id)`, `.save(order)` |
| **Domain service** | Domain logic that doesn't belong to a single entity | `TransferService.transfer(from, to, amount)` across two accounts, `PricingPolicy` |
| **Domain event** | Something meaningful that happened in the domain | `OrderPlaced`, `PaymentFailed` |
| **Factory** | Encapsulates complex creation | `Order.place(...)` static factory enforcing invariants |
| **Specification** | Reusable business rule/predicate | `EligibleForFreeShipping` |
| **Policy** | Reacts to events with commands ("whenever X, then Y") | "When payment fails 3 times, suspend subscription" |

**Value objects are hugely underused.** Replacing primitives (`str email`, `int amount_cents`, `str currency`) with value objects (`Email`, `Money`) eliminates whole classes of bugs (mixing currencies, invalid emails deep in the code, "primitive obsession").

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str
    def __post_init__(self):
        if self.amount.as_tuple().exponent < -2: raise ValueError("too many decimals")
    def __add__(self, other: "Money") -> "Money":
        if other.currency != self.currency: raise CurrencyMismatch()
        return Money(self.amount + other.amount, self.currency)
```

**Rich domain model**: behavior lives with data, and invariants are enforced by methods:
```python
class Order:
    def add_item(self, product_id, qty, unit_price: Money):
        if self.status != OrderStatus.DRAFT: raise OrderNotEditable()
        if qty <= 0: raise InvalidQuantity()
        self._lines.append(OrderLine(product_id, qty, unit_price))
        self._events.append(ItemAdded(self.id, product_id, qty))
    def submit(self):
        if not self._lines: raise EmptyOrder()
        self.status = OrderStatus.SUBMITTED
        self._events.append(OrderSubmitted(self.id, self.total()))
```
Contrast with the anemic `order.status = "SUBMITTED"` set from five different services, each with slightly different checks.

---

## 8. Aggregate Design Rules {#aggregates}

Vernon's rules:
1. **Protect true invariants inside aggregate boundaries**: an aggregate is a transactional consistency boundary. Everything inside is consistent after each transaction.
2. **Design small aggregates**: large aggregates (Customer with all orders ever) cause contention (concurrent edits conflict), slow loads, and memory bloat.
3. **Reference other aggregates by ID only**, not object references (`order.customer_id`, not `order.customer`).
4. **Use eventual consistency between aggregates**: one transaction modifies one aggregate. Cross-aggregate effects happen via domain events (possibly in-process, after commit).
5. Use optimistic concurrency (version) on aggregate roots.

Example: "Order total must not exceed the customer's credit limit" spans Order and Customer. Options: check at submission (with an accepted race window + compensating flow), model a `CreditReservation` aggregate, or reconsider the boundary if the invariant is truly critical.

---

## 9. Application Services and Use Cases {#application}

The application layer **orchestrates** a use case and contains no business rules:
1. Receive a command/DTO (already syntactically validated).
2. Authorize (or delegate to a policy).
3. Load aggregate(s) via repositories.
4. Call domain methods (the business rules live there).
5. Persist via repositories inside a transaction (unit of work).
6. Publish domain events (via outbox).
7. Return a result DTO.

Keep controllers thin: parse and validate input, call the use case, map the result or exception to transport (HTTP status, gRPC code).

**Command/query separation in code**: commands go through the domain model, and queries can bypass it entirely (direct SQL into DTOs), which is simpler and faster for reads.

---

## 10. Persistence and ORMs {#persistence}

- **Persistence ignorance** (ideal): domain objects don't know how they're stored. Repositories map between domain objects and storage models.
- Pragmatic options:
  1. Separate persistence models (ORM entities/rows) + mappers to domain objects: the cleanest, at the cost of boilerplate.
  2. Use ORM-mapped classes as domain entities, but keep ORM concerns minimal (JPA annotations on domain classes is a common compromise; SQLAlchemy imperative/classical mapping keeps domain classes plain).
- **Unit of Work**: tracks changes and commits them atomically (SQLAlchemy Session, EF Core DbContext, Hibernate Session). Transaction boundaries are set at the application service.
- Repositories return **aggregates**, not query results for screens (use query services or read models for those).
- Beware lazy loading leaking out of the transaction ("LazyInitializationException", N+1). Load what the aggregate needs.
- See [`database-concepts/orms.md`](../database-concepts/orms.md).

---

## 11. Validation {#validation}

Validation happens at multiple levels for different reasons:
| Level | Checks | Where |
|---|---|---|
| **Input/syntactic** | Required fields, types, formats, lengths, ranges | Edge: request DTO validation (Pydantic, Bean Validation, Zod, class-validator, JSON Schema/OpenAPI) → 400/422 |
| **Domain invariants** | Business rules always true for the entity ("order can't be shipped before paid", "quantity > 0") | Inside entities, value objects, and aggregates (constructor and methods). **Always enforced**, regardless of the entry point |
| **Contextual/application** | Rules depending on external state ("email not already registered", "user has permission") | Application service (with DB constraints as a backstop for uniqueness) |
| **Database constraints** | NOT NULL, UNIQUE, FK, CHECK | Last line of defense against races and bugs |

"Make illegal states unrepresentable": types and value objects, so invalid data can't even be constructed past the boundary ("parse, don't validate").

---

## 12. Error Handling Strategy {#errors}

- **Domain errors** (expected business outcomes): typed exceptions or result types (`InsufficientFunds`, `OrderNotCancellable`). Map them to 409/422 with stable error codes (see the API design note).
- **Infrastructure errors** (DB down, timeout): let them propagate to a global handler, which maps them to 5xx/503, logs with context, and doesn't leak internals.
- **Programming errors** (bugs): fail fast, alert.
- Result types (`Result<T, E>` in Rust, `Either` in functional styles, Go's `(value, error)`) make expected failures explicit in signatures. Exceptions suit truly exceptional cases.
- Never swallow exceptions silently. Never use exceptions for normal control flow in hot paths.
- Centralized mapping (an exception → HTTP middleware) keeps controllers clean.

---

## 13. Dependency Injection {#di}

**Dependency inversion**: high-level modules (use cases) depend on abstractions (ports), and low-level modules (adapters) implement them. **DI** is the mechanism that provides the concrete implementations from outside.
- **Constructor injection** (preferred): explicit and testable. Dependencies are visible in the signature.
- **Composition root**: wire everything up in one place at startup (`main`, a container module).
- DI containers: Spring, .NET built-in, Guice/Dagger (Java), NestJS (TS), InversifyJS, `dependency-injector` (Python), Wire/fx (Go). In Go and Python, manual wiring is often clearer than a container.
- Lifetimes: singleton (stateless services, clients, pools), scoped per request (unit of work, DB session, request context), transient.
- Anti-patterns: the service locator (hidden dependencies), injecting the container itself, and too many constructor parameters (a sign the class does too much: SRP).

---

## 14. SOLID with Backend Examples {#solid}

- **S**ingle Responsibility: a class has one reason to change. `InvoiceService` that calculates totals, renders PDFs, and sends emails should be split into `InvoiceCalculator`, `InvoiceRenderer`, and `InvoiceNotifier`.
- **O**pen/Closed: extend behavior without modifying existing code. A new payment provider means adding a `PaymentGateway` implementation, not editing an `if provider == ...` chain everywhere (strategy pattern + DI).
- **L**iskov Substitution: subtypes must honor the base contract. A `ReadOnlyRepository` subclass throwing on `save()` violates it, so separate the interfaces instead.
- **I**nterface Segregation: small, focused interfaces. Use cases depend on `OrderReader` rather than a 40-method `OrderRepository`.
- **D**ependency Inversion: depend on abstractions owned by the core (ports), not on concrete infrastructure.

Related principles: DRY (but prefer duplication over the wrong abstraction), KISS, YAGNI, composition over inheritance, the Law of Demeter, "tell, don't ask".

---

## 15. Reference Project Structure {#structure}

Modular monolith with hexagonal modules (Python-flavored; maps to any language):
```
src/
  shared/                     # cross-cutting: config, logging, tracing, db session, outbox, auth context
  modules/
    ordering/
      domain/                 # entities, value objects, aggregates, domain events, domain services (no imports from infra)
        order.py  money.py  events.py
      application/            # use cases (commands/queries), ports (interfaces), DTOs
        place_order.py  cancel_order.py  ports.py  queries/order_history.py
      infrastructure/         # adapters: SQL repositories, API clients, message publishers
        sql_order_repository.py  stripe_gateway.py
      api/                    # HTTP/gRPC handlers, request/response schemas
        routes.py  schemas.py
      tests/
        unit/ (domain + use cases with fakes)  integration/ (adapters with Testcontainers)
    billing/
    catalog/
  main.py                     # composition root: wire adapters into use cases, mount routes
```
Enforce: `domain` imports nothing from `application`/`infrastructure`/`api`, and modules import each other only via their `application` public API (checked with import-linter or ArchUnit).

---

## 16. When This Is Overkill {#overkill}

- Simple CRUD services, admin backoffices, prototypes: use the framework's conventions (Rails/Django/Laravel "fat models, thin controllers"), and keep it simple.
- Apply DDD tactical patterns to the **core domain** with complex rules, not to every table.
- Watch for over-abstraction: interfaces with one implementation that will never change (e.g., wrapping the ORM in five layers) add indirection without value. It's fine to depend directly on stable infrastructure in non-core code.
- Architecture should grow with complexity: start simple, and refactor toward ports and adapters as the domain logic grows.

---

## 17. Interview Questions {#qa}

1. Explain hexagonal architecture. What are ports and adapters? How does it help testing?
2. What's the Dependency Rule in Clean Architecture?
3. What is a bounded context? How does it relate to microservice boundaries?
4. Entity vs value object? Give examples of value objects you'd introduce in a payments system.
5. What is an aggregate, and why should aggregates be small? How do you handle invariants across aggregates?
6. What is an anemic domain model, and why is it considered an anti-pattern?
7. Where should validation live?
8. What's an anti-corruption layer and when do you need one?
9. Layered vs vertical slice architecture?
10. When is DDD not worth it?
