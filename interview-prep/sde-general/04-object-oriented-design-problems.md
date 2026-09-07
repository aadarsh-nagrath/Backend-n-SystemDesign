# Object-Oriented / Low-Level Design (LLD) Problems

> A distinct interview category from both "system design" (distributed systems, scale) and "DSA" (algorithms) — LLD questions ask you to design the *classes, interfaces, and relationships* for a real-world system, testing OOP fundamentals (see `02-oop-design-patterns.md`) applied to a concrete problem. Sourced/topic-checked as consistently among the most frequently asked LLD interview questions across major tech interview prep resources.

---

## 1. Design a Parking Lot System

**Requirements to clarify first**: Multiple levels, multiple spot sizes (motorcycle/compact/large); tracking availability in real time; calculating fees on exit; possibly multiple entry/exit points.

**Core classes**:
- `ParkingLot` (likely a Singleton — see the OOP file's discussion of when Singleton is/isn't appropriate; here it's a legitimate use since there's genuinely exactly one parking lot instance the whole system coordinates against) — holds a collection of `Level` objects and coordinates ticket issuance.
- `Level` — holds a collection of `ParkingSpot` objects and tracks available spots per spot type for fast availability lookups.
- `ParkingSpot` (abstract base, or an enum-typed field) with subtypes/sizes: `MotorcycleSpot`, `CompactSpot`, `LargeSpot` — each knows whether it's currently occupied and what vehicle types it can accommodate.
- `Vehicle` (abstract base) with subtypes `Motorcycle`, `Car`, `Bus` — each knows what spot size(s) it requires (a `Bus` might require multiple contiguous `LargeSpot`s, a `Motorcycle` can fit in any spot type).
- `Ticket` — created on entry, records the vehicle, assigned spot, and entry timestamp; used on exit to calculate duration/fee.
- `Payment`/`FeeCalculator` — computes the charge based on duration and vehicle type, decoupled from `Ticket` itself (Single Responsibility — see the OOP file's SOLID discussion) so pricing logic can change independently of ticket-tracking logic.

**Key design decisions to discuss out loud**:
- How does the system find an available spot efficiently? Maintain a count of free spots per type per level (updated on entry/exit) so you can quickly identify a candidate level *before* scanning for a specific free spot within it, rather than linearly scanning every spot in the entire lot on every entry.
- What happens when the lot is full? The entry gate should check availability *before* issuing a ticket at all, returning a clear "lot full" signal rather than issuing a ticket and only discovering the problem later.
- How do you avoid a race condition where two vehicles are simultaneously assigned the same spot (directly the same class of concurrency problem discussed throughout the backend/system-design files) — spot assignment must be an atomic operation (a lock per spot, or an atomic conditional update if backed by a database) so two concurrent entry requests can never both claim the same physical spot.

**What a strong answer demonstrates**: correctly modeling the vehicle-to-spot-size relationship as its own concern (not hardcoded if/else chains scattered everywhere), keeping fee calculation decoupled and extensible (strategy pattern — see the OOP file — for different rate schemes: hourly, flat, day-pass), and proactively raising the concurrency question without being prompted.

---

## 2. Design an Elevator System

**Requirements to clarify first**: Single or multiple elevators; how requests are dispatched (which elevator answers a call); does it need to optimize for wait time, or just correctness?

**Core classes**:
- `Elevator` — tracks current floor, direction (up/down/idle), and a queue of destination requests.
- `ElevatorController`/`Dispatcher` — receives external floor-call requests ("someone on floor 5 wants to go up") and internal cabin requests ("someone inside elevator 2 pressed floor 8"), and decides which elevator should service which request.
- `Request` (or separate `ExternalRequest`/`InternalRequest`) — represents a single call, with a floor and, for external requests, a direction.
- `Door` — a simple state machine (open/closed/opening/closing) the `Elevator` interacts with at each stop.

**Key design decisions to discuss out loud**:
- **Dispatch algorithm**: the classic, most commonly discussed approach is variations of the SCAN/elevator algorithm — an elevator continues moving in its current direction, picking up/dropping off requests along the way, only reversing direction once it has no more requests ahead of it in the current direction — this avoids the poor behavior of naively always sending the *nearest idle* elevator (which can lead to unnecessary direction reversals and worse overall wait times) or servicing requests purely in the order received (which can be wildly inefficient, e.g., going to floor 1 then floor 10 then floor 2).
- **How does the controller pick which elevator answers an external call** when there are multiple elevators? A common heuristic: prefer an elevator already moving toward the requested floor in the requested direction over an idle one, and prefer the closest idle elevator over a busy one moving away — this is a genuinely open-ended, discuss-the-tradeoffs question, not a single "correct" algorithm, and interviewers often want to see you reason about the tradeoff (optimizing average wait time vs. worst-case wait time vs. simplicity of implementation) rather than recite one memorized answer.
- **State management**: modeling the elevator's current state (idle, moving up, moving down, door open) as an explicit state machine (rather than scattered boolean flags) makes the transition logic much easier to reason about and extend — a natural opportunity to mention the State design pattern.

**What a strong answer demonstrates**: separating the *dispatch decision* (which elevator handles which request) from each individual elevator's own *local scheduling* (the order it services its own queued requests) as two distinct, independently-reasonable-about concerns, and being able to discuss the SCAN-algorithm tradeoff explicitly rather than treating dispatch as a trivial "send the nearest one" problem.

---

## 3. Design a Library Management System

**Requirements to clarify first**: Track books and their available copies; members can borrow/return/reserve books; handle overdue fines; search by title/author/ISBN.

**Core classes**:
- `Book` — represents a title's metadata (ISBN, title, author) — note this is distinct from...
- `BookCopy` (or `BookItem`) — represents one *physical* copy of a `Book`, with its own status (available, checked out, reserved, lost) — a crucial modeling distinction, since a `Book` (the title/work) is conceptually one thing but a library can hold multiple physical copies, each independently trackable.
- `Member` — a library patron, tracking their currently borrowed `BookCopy` objects and any outstanding fines.
- `Loan`/`Checkout` — represents one borrowing transaction (which member, which copy, checkout date, due date, return date if returned) — this is where overdue-fine calculation logic naturally lives, keyed off `due_date` vs. actual `return_date`.
- `Reservation` — represents a member's request to be notified/have priority when a currently-checked-out book becomes available, forming a queue per `Book` title.
- `Catalog`/`SearchService` — indexes books by title/author/ISBN/genre for lookup, decoupled from the `Book`/`BookCopy` data model itself (Single Responsibility, and directly parallel to the Repository pattern discussion in the OOP file).

**Key design decisions to discuss out loud**:
- Modeling `Book` (the abstract title) separately from `BookCopy` (a physical, trackable instance) is the single most important modeling decision in this problem — getting this distinction right (or wrong) is usually what separates a strong answer from a weak one, since conflating them makes correctly handling "3 copies exist, 2 are checked out" naturally awkward.
- How does the reservation queue interact with returns? When a `BookCopy` is returned, the system should check if there's a pending `Reservation` for that title and, if so, transition that copy to "reserved for member X" rather than simply making it generally available again — a real business-logic branch point worth narrating explicitly.
- Fine calculation as its own decoupled component (similar to the Parking Lot's `FeeCalculator`) so the fine policy (per-day rate, maximum cap, grace period) can change independently of the core loan-tracking logic.

**What a strong answer demonstrates**: the `Book`/`BookCopy` distinction made explicitly and early, and correctly reasoning through the reservation-queue-interacts-with-return-flow business logic rather than treating "return a book" as a trivial status flip.

---

## 4. Design a Vending Machine

**Requirements to clarify first**: Multiple product slots with limited inventory per slot; accepts multiple payment types (coins, cash, card); must handle insufficient payment, exact-vs-change scenarios, and out-of-stock items.

**Core classes**:
- `VendingMachine` — top-level coordinator holding the current `State` and delegating to it.
- `Inventory` — tracks each product slot's item and remaining quantity.
- `Product` — item metadata (name, price).
- Payment handling: a `PaymentStrategy` interface (directly the Strategy pattern from the OOP file) with implementations like `CoinPayment`, `CashPayment`, `CardPayment` — letting the machine accept new payment types later without modifying its core logic.
- **State** — this problem is the canonical textbook example for the State design pattern: `IdleState`, `HasMoneyState`, `DispensingState`, `OutOfStockState` — each state defines what actions are valid from it (e.g., you can't "select a product" from `DispensingState`) and what the valid transitions to other states are, making illegal sequences of operations structurally impossible to represent rather than needing to be defensively checked with scattered conditionals throughout the code.

**Key design decisions to discuss out loud**:
- Explicitly naming and using the State pattern here (rather than a tangle of boolean flags like `hasInsertedMoney`, `isDispensing`, etc.) is exactly the kind of "recognizing which pattern actually fits this problem" signal LLD interviews are designed to surface — see the OOP file's Q17 on recognizing genuine pattern fit versus over-engineering; this is a case where the pattern genuinely earns its place, since the problem *is* fundamentally a state machine.
- How is change calculated and dispensed, and what happens if the machine can't make exact change? This is a real edge case worth raising proactively: the machine should check whether it has sufficient change *before* completing a transaction that would require it, refusing (or refunding) rather than dispensing a product with no way to give correct change back.
- Concurrency: if the vending machine is a shared, networked/IoT-style machine rather than a single physical unit, could two people trigger a purchase for the last item in a slot simultaneously? Raising this — and that it's the same inventory-decrement race condition discussed throughout the backend fundamentals and system-design files — shows the connection between LLD and the broader concurrency concepts rather than treating LLD as an isolated category.

---

## General LLD interview strategy (applies to all of the above)

Start by clarifying the exact scope and functional requirements before naming a single class (mirroring the system design interview approach in that file's Q1 — LLD interviews reward the same "clarify before designing" discipline). Identify the core *entities* (nouns) and their *responsibilities* first, then work out relationships between them (composition vs. aggregation — see the OOP file's Q16 — and whether a relationship is genuinely inheritance/"is-a" or should be composition/"has-a"). Explicitly name any design pattern you're applying and *why* it fits this specific problem's actual variability/complexity, rather than forcing a pattern in for its own sake (the over-engineering warning from the OOP file's final question applies directly here — LLD interviewers are specifically listening for whether you can justify a pattern's use, not just whether you can recite pattern names). Finally, proactively raise at least one non-obvious edge case or concurrency concern for the specific problem (as modeled in each example above) — this is consistently what distinguishes a strong LLD answer from a merely competent one.
