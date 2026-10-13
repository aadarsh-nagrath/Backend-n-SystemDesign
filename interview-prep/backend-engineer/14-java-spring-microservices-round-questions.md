# Core Java, DSA, Spring Boot & Microservices — Real Interview Round Questions

> These come from an actual multi-round backend interview loop (Round 1: Core Java/DSA, Round 2: Spring Boot & microservices). Questions that were already answered elsewhere in this repo are not repeated here:
> - *"How would you make a REST API idempotent?"* → see `02-api-design-rest-graphql-grpc.md` Q4 and Q13, and `01-backend-fundamentals.md` Q8. The payment-system design in `../system-design/07-payments-consistency-and-concurrency-at-scale.md` walks through the full idempotency-key implementation.
> - *"When would you choose Kafka over synchronous REST?"* → see `06-message-queues-event-driven.md` Q1, Q3 and Q7.
>
> The Round 3 (system design) questions are answered in full interview format in `../system-design/07-payments-consistency-and-concurrency-at-scale.md`.

---

## Round 1 — Core Java, DSA & Problem Solving

### 1. How does `HashMap` work internally? Explain collisions and resizing.

**Structure (Java 8+).** A `HashMap` is an array (`table`) of *buckets*. Each bucket holds a linked list of `Node<K,V>` entries (`hash`, `key`, `value`, `next`), or a red-black tree of `TreeNode`s once the bucket gets large. The table length is always a power of two.

**`put(key, value)` step by step:**
1. **Hash the key.** `hash = key.hashCode() ^ (key.hashCode() >>> 16)`. XOR-ing in the high 16 bits mixes them into the low bits. This matters because the index only uses the low bits, so keys whose hash codes differ only in the high bits would otherwise all collide.
2. **Find the bucket.** `index = (n - 1) & hash`, where `n` is the table length. Because `n` is a power of two, this is a fast replacement for `hash % n`.
3. **Empty bucket**: put a new node there.
4. **Non-empty bucket (a collision)**: walk the chain. For each node, check `node.hash == hash && (node.key == key || key.equals(node.key))`. If a node matches, **replace its value** and return the old one. If none matches, **append** a new node at the tail (Java 8 appends at the tail; Java 7 inserted at the head).
5. `++size`; if `size > threshold`, **resize**.

`get(key)` follows the same steps: hash, find the bucket, then compare `hash` first (a cheap int check) and `equals()` only on a hash match.

`null` key: allowed, exactly one. Its hash is 0, so it always goes in bucket 0.

**Collisions.** Two keys collide when they land in the same bucket. That happens either because their `hashCode()` values are equal, or because different hashes have the same low bits. Collisions are resolved by **separate chaining** (the linked list in the bucket).
- With a good hash function and load factor 0.75, chains stay very short, so `get`/`put` are O(1) on average.
- Worst case, every key lands in one bucket. With a linked list that is O(n) per operation, and attackers have used this for "hash flooding" DoS attacks on web frameworks.
- **Treeification (Java 8)**: when a single bucket reaches **8 nodes** (`TREEIFY_THRESHOLD`) *and* the table has at least **64 buckets** (`MIN_TREEIFY_CAPACITY`), that bucket becomes a red-black tree, so the worst case is O(log n). If the table is smaller than 64, the map resizes instead, since a small table is the more likely cause. A tree turns back into a list when it shrinks to **6** (`UNTREEIFY_THRESHOLD`). The gap between 8 and 6 stops it flip-flopping. The tree orders nodes by hash, then by `compareTo` when the keys are `Comparable`, then by a tie-breaker.

**Resizing.**
- Defaults: capacity **16**, load factor **0.75**, so `threshold = 16 × 0.75 = 12`. The table is allocated lazily on the first `put`.
- When `size > threshold`, the table **doubles** (16 → 32 → 64…), and every entry moves to the new table.
- Java 8 does this cleverly. Since the capacity doubled, an entry's new index is either **the same index** or **old index + oldCapacity**, depending on one bit: `(hash & oldCap) == 0`. Each chain is split into a "lo" list and a "hi" list, keeping relative order, and no hash is recomputed.
- Resizing costs O(n), but it happens geometrically less often, so `put` is still **amortized O(1)**. If you know the size in advance, pre-size the map: `new HashMap<>(expected / 0.75f + 1)` (or `HashMap.newHashMap(expected)` in Java 19+) to avoid repeated resizes.

**Why the load factor is 0.75**: it trades memory against collision probability. A higher value packs the table tighter and gives longer chains. A lower value wastes space.

**Common follow-ups:**
- **Is it thread-safe?** No. Concurrent `put`s can lose updates or corrupt size. In Java 7, a concurrent resize with head insertion could create a **cycle in a bucket's chain**, so `get()` spun at 100% CPU forever. Java 8's tail insertion fixed that particular cycle, but the map is still unsafe. Use `ConcurrentHashMap`, which uses CAS to fill an empty bin, locks (`synchronized`) only the head node of a non-empty bin, and resizes cooperatively across threads. It does not allow `null` keys or values.
- **Mutable keys**: if you change a field that `hashCode()` uses *after* inserting the key, the entry sits in the wrong bucket and `get` can never find it. Use immutable keys (`String`, `Integer`, records).
- `LinkedHashMap` adds a doubly-linked list over the entries for insertion or access order (the basis of a simple LRU cache via `removeEldestEntry`). `TreeMap` is a red-black tree with O(log n) operations and sorted keys.

---

### 2. Find the longest substring without repeating characters in O(n) time.

**Approach — sliding window with a "last seen index" map.** Keep a window `[left, right]` that has no repeated characters. Move `right` forward one character at a time. If the new character was already seen *inside the current window*, jump `left` to just past its previous position. Record the best window length as you go.

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int best = 0, left = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        Integer prev = lastSeen.get(c);
        if (prev != null && prev >= left) {   // repeat inside the current window
            left = prev + 1;                  // jump past it; never move left backwards
        }
        lastSeen.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Walkthrough** for `"abcabcbb"`: window grows to `abc` (3). At index 3 (`a`, last seen at 0), `left` jumps to 1, giving `bca`. Then `cab`, then `abc`. At index 6 (`b`, last seen at 4) `left` becomes 5. At index 7 (`b`, last seen at 6) `left` becomes 7. Answer: **3**.

**Why `prev >= left` matters**: in `"abba"`, when you reach the final `a` its last index is 0, but `left` is already at 2. Without the check, `left` would move *backwards* to 1 and the answer would be wrong (3 instead of 2). `left = Math.max(left, prev + 1)` is an equivalent way to write it.

**Complexity**: O(n) time, since each index enters the window once and `left` only moves forward. Space is O(min(n, alphabet)). For ASCII you can replace the map with `int[128]` (store `index + 1` so that 0 means "not seen"), which is faster and uses constant space.

**Variant to mention**: the "shrink one step at a time" version uses a `HashSet` and `while (set.contains(c)) set.remove(s.charAt(left++))`. It is also O(n), because each character is added and removed at most once, but it does up to 2n steps.

**How to present it in the interview**: say the brute force first (check every substring, O(n³), or O(n²) with a set). Point out the wasted work: when you find a repeat, you already know everything up to the earlier occurrence is useless. That observation leads to the sliding window. Then state the complexity and test the edge cases: empty string, all the same character (`"bbbb"` → 1), all unique, and `"abba"`.

---

### 3. Detect a cycle in a linked list without using extra space.

**Floyd's cycle detection ("tortoise and hare")**: two pointers start at the head. `slow` moves 1 step and `fast` moves 2 steps. If there is no cycle, `fast` reaches `null`. If there is a cycle, `fast` enters it first and closes the gap on `slow` by one node per step, so they **must meet**. `fast` can't skip over `slow`, because the gap shrinks by exactly 1 each step.

```java
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;   // compare references, not values
    }
    return false;
}
```

O(n) time, O(1) space. A `HashSet<ListNode>` of visited nodes also works, but uses O(n) space, which the question rules out.

**Follow-up 1: find the node where the cycle starts.** Once they meet, move one pointer back to `head`. Then advance **both by 1** until they meet again. That meeting node is the start of the cycle.

*Why it works*: let `a` = the distance from the head to the cycle start, `b` = the distance from the cycle start to the meeting point, and `L` = the cycle length. When they meet, `slow` has walked `a + b` and `fast` has walked `2(a + b)`. Fast's extra distance is a whole number of laps: `a + b = kL`, so `a = kL − b`. Walking `a` steps from the meeting point therefore lands exactly on the cycle start, which is also `a` steps from the head.

```java
ListNode cycleStart(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next; fast = fast.next.next;
        if (slow == fast) {
            ListNode p = head;
            while (p != slow) { p = p.next; slow = slow.next; }
            return p;
        }
    }
    return null;
}
```

**Follow-up 2: cycle length.** From the meeting point, keep one pointer still and count steps around until you are back at it.

**Follow-up 3: remove the cycle.** Find the start, then walk around the cycle to the node whose `next` is the start and set its `next` to `null`.

**Other places this is used**: detecting a cycle in any "next" function, such as LeetCode 287 (find the duplicate number, treating values as pointers) and the "happy number" problem. Brent's algorithm is an alternative with fewer function evaluations.

---

### 4. What happens when multiple threads modify the same shared object?

Without synchronization, you get three separate kinds of problems. Naming all three is what separates a senior answer from "you get a race condition":

**1. Lost updates (atomicity).** `count++` is really three steps: read, add, write. Two threads can both read 5 and both write 6, so one increment disappears. The same thing happens with check-then-act code (`if (!map.containsKey(k)) map.put(k, v)`) and with read-modify-write on any field.

**2. Visibility (the Java Memory Model).** Each thread may keep a value in a CPU cache or register, and the JIT may hoist a read out of a loop. Without a *happens-before* relationship, one thread's write may **never** become visible to another thread. Classic bug: `while (!stopped) { }` spins forever even after another thread sets `stopped = true`, because nothing forces the loop to re-read the field.

**3. Reordering / partially constructed objects.** The compiler and CPU can reorder instructions, as long as the result looks the same from inside one thread. Another thread can then see an object reference before the object's fields are written. Broken double-checked locking without `volatile` is the textbook example.

**Symptoms you see in practice**: wrong counters or balances, invariants broken between related fields (for example `start > end`), `ConcurrentModificationException` when one thread iterates a collection while another modifies it, a corrupted `HashMap` (lost entries, and in Java 7 an infinite loop), and bugs that disappear when you add logging, because logging changes the timing.

**How to fix it, from simplest to most involved:**
| Technique | Fixes | Notes |
|---|---|---|
| **Don't share**: immutability (`final` fields, records), thread confinement, `ThreadLocal` | Everything | The best option. Immutable objects are thread-safe by construction |
| `synchronized` / `ReentrantLock` | Atomicity + visibility | Unlocking a monitor *happens-before* the next lock of the same monitor. Keep critical sections small and lock on a private final object |
| `volatile` | Visibility + ordering only | Good for flags and safe publication. **Does not** make `count++` atomic |
| `AtomicInteger`, `AtomicReference`, `LongAdder` | Atomicity of single variables | Lock-free CAS. `LongAdder` is better for heavily contended counters |
| Concurrent collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`) | Safe collection access | Use atomic compound methods (`computeIfAbsent`, `merge`) instead of check-then-act |
| Higher-level tools: `ExecutorService`, `CompletableFuture`, message passing | Avoid hand-written locking | |

**Happens-before rules to quote**: unlocking a monitor happens-before every later lock of it. A write to a `volatile` field happens-before every later read of it. `Thread.start()` happens-before anything the started thread does. Everything a thread does happens-before another thread's `join()` on it returns. Writes to `final` fields in a constructor are visible once the reference is safely published.

**Across processes it's the same problem.** Two service instances updating the same database row is the same race. The fixes there are optimistic locking (a `@Version` column), `SELECT … FOR UPDATE`, or an atomic `UPDATE … SET balance = balance - 10 WHERE balance >= 10`. See `05-concurrency-multithreading-async.md` Q3 and Q14.

---

### 5. What happens if you override `equals()` but not `hashCode()`?

You break the **`equals`/`hashCode` contract**: *if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must also be true*. (The reverse isn't required: unequal objects may share a hash code.) `Object.hashCode()` is based on object identity, so two objects that are logically equal will almost always get **different** hash codes.

**What breaks:** every hash-based collection (`HashMap`, `HashSet`, `ConcurrentHashMap`, `LinkedHashMap`), and any cache keyed by the object.

```java
class Emp {
    String id;
    Emp(String id) { this.id = id; }
    @Override public boolean equals(Object o) {
        return o instanceof Emp e && id.equals(e.id);
    }
    // no hashCode()
}

Set<Emp> set = new HashSet<>();
set.add(new Emp("42"));
set.add(new Emp("42"));
set.size();                       // 2: a "duplicate" was allowed
set.contains(new Emp("42"));      // false (almost always)

Map<Emp, String> m = new HashMap<>();
m.put(new Emp("42"), "Alice");
m.get(new Emp("42"));             // null: the entry is "lost"
```

**Why**: `HashMap` uses `hashCode()` to pick the bucket *before* it ever calls `equals()`. Different hash codes send the lookup to a different bucket, so `equals()` is never even called. Even when two keys land in the same bucket by luck, the map compares the stored `hash` field first, and that check fails. The behaviour is *nondeterministic*. It usually fails, occasionally works, and is very confusing to debug.

**Things that don't break**: `ArrayList.contains` / `indexOf` and `List.equals`, because they only use `equals()`. That's why the bug often gets past unit tests that only use lists.

**The reverse mistake** (overriding `hashCode()` but not `equals()`): equal-looking objects land in the same bucket, but identity `equals` treats them as different. You get duplicates in sets and failed lookups, just with more collisions.

**Doing it right:**
- Use the same fields in both methods: `Objects.equals(...)` in `equals`, and `Objects.hash(f1, f2)` in `hashCode`.
- Use only **immutable** fields, or never change them while the object is a key.
- Prefer a Java `record` (it generates both automatically), or Lombok `@EqualsAndHashCode`, or let the IDE generate them.
- `equals` must be reflexive, symmetric, transitive, consistent, and return `false` for `null`. Mixing subclasses and `instanceof` can break symmetry, which is one reason records are final.
- **JPA entity gotcha**: don't base `hashCode` on a database-generated `@Id`. The id is `null` before persist and set afterwards, so the entity "moves" buckets inside a `Set` it was already in. Use a natural/business key, or return a constant `hashCode` together with an id-based `equals`.

---

## Round 2 — Spring Boot & Microservices

### 6. What happens internally when a method annotated with `@Transactional` is called?

**Short answer**: Spring doesn't change your class. At startup it wraps the bean in a **proxy**. Callers get the proxy, and the proxy's `TransactionInterceptor` starts a transaction, calls your real method, then commits or rolls back.

**Step by step:**

1. **At startup (proxy creation).** `@EnableTransactionManagement` is auto-configured by Spring Boot. It registers a `BeanPostProcessor` (`InfrastructureAdvisorAutoProxyCreator`). For each bean with `@Transactional` methods, it creates an AOP proxy:
   - **CGLIB subclass proxy** by default in Spring Boot (`spring.aop.proxy-target-class=true`). This is why `final` classes/methods can't be advised.
   - **JDK dynamic proxy** when proxying through interfaces.
   - Whatever injects that bean receives the **proxy**, not the raw object.

2. **At call time**, the proxy delegates to `TransactionInterceptor.invoke()`, which calls `TransactionAspectSupport.invokeWithinTransaction()`:
   - **Read the attributes** (propagation, isolation, timeout, readOnly, rollbackFor) from `AnnotationTransactionAttributeSource`. The results are cached per method.
   - **Pick the `PlatformTransactionManager`**: `JpaTransactionManager` when you use JPA, `DataSourceTransactionManager` for plain JDBC, or the reactive equivalent.
   - **`getTransaction(definition)` applies the propagation rule**:
     - `REQUIRED` (the default): join the current transaction if there is one, otherwise start a new one.
     - `REQUIRES_NEW`: suspend the current transaction and start a new, independent one on a **different connection**.
     - `NESTED`: create a savepoint inside the current transaction.
     - `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER`: run with or without a transaction, or throw.
   - **Starting a new transaction**: borrow a `Connection` from the pool (HikariCP), call `setAutoCommit(false)`, apply the isolation level, read-only flag, and timeout, and **bind the connection (and the JPA `EntityManager`) to the current thread** using `TransactionSynchronizationManager`, which is backed by `ThreadLocal`. That binding is how your repositories, `JdbcTemplate`, and so on all reuse the same connection without you passing it around.

3. **Your method runs** (`invocation.proceed()`).

4. **Completion:**
   - **Returns normally**: `commitTransactionAfterReturning()`. For JPA, the persistence context is **flushed** first (dirty-checked `UPDATE`/`INSERT` statements are sent now, which is when constraint violations often show up), then `connection.commit()`.
   - **Throws**: `completeTransactionAfterThrowing()` checks the rollback rules. By default it **rolls back only on `RuntimeException` and `Error`**. A **checked exception still commits** unless you set `rollbackFor = Exception.class`.
   - **Synchronization callbacks** run: `beforeCommit`, `afterCommit`, `afterCompletion`. This is how `@TransactionalEventListener(phase = AFTER_COMMIT)` works.
   - **Cleanup**: unbind from the thread, restore auto-commit and isolation, return the connection to the pool, and resume any suspended outer transaction.

**Pitfalls interviewers look for:**
- **Self-invocation** skips the proxy (see the next question).
- **`private`, `final`, or `static` methods** aren't intercepted. Since Spring 6 `protected` and package-private methods work with class-based proxies, but `public` is still the safe choice.
- **Catching the exception inside the method** means the interceptor never sees it, so the transaction commits.
- **`UnexpectedRollbackException`**: an inner `REQUIRED` method throws, which marks the *shared* transaction rollback-only. The outer method catches the exception and carries on, and then the commit fails. If the inner work needs to fail independently, use `REQUIRES_NEW` or `NESTED`.
- **The transaction belongs to a thread**: work handed to `@Async`, `CompletableFuture.supplyAsync`, or a new thread is *not* part of the caller's transaction.
- **Long transactions hold a DB connection the whole time.** Never make HTTP calls or other slow I/O inside `@Transactional`. You'll run out of pool connections (see `../system-design/07-payments-consistency-and-concurrency-at-scale.md` §3).
- `readOnly = true` is a hint: Hibernate skips dirty checking and flushing, and some drivers route the query to a replica. It doesn't stop writes at the database level everywhere.
- **Calling a remote service and then committing isn't atomic** across the two. That's what the outbox pattern is for (system design file §2).

---

### 7. Why can `@Transactional` fail during self-invocation?

Because the transaction logic lives in the **proxy**, and a call from inside the same object uses `this`, which is the **raw target object**, not the proxy. The interceptor is skipped completely, so no transaction starts, and annotation settings like `REQUIRES_NEW` are silently ignored.

```java
@Service
public class OrderService {
    public void placeOrders(List<Order> orders) {
        for (Order o : orders) {
            this.saveOne(o);              // calls the target directly; no proxy, no transaction
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOne(Order o) { ... }  // annotation has no effect on the call above
}
```

```
caller ──► [ CGLIB proxy ── TransactionInterceptor ] ──► target.placeOrders()
                                                              │
                                                              └── this.saveOne()  ← stays inside target
```

The same is true for **every proxy-based annotation**: `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize`, and Resilience4j's `@CircuitBreaker`.

**How to fix it, best option first:**
1. **Move the method to a separate bean** (for example `OrderWriter.saveOne`) and inject it. This is the cleanest fix, and it usually makes the design better too.
2. **Use `TransactionTemplate`** for a programmatic transaction boundary around the work:
   ```java
   for (Order o : orders) {
       txTemplate.executeWithoutResult(status -> repo.save(o));
   }
   ```
   (Set `txTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW)` if you need it.)
3. **Inject the bean into itself** with `@Lazy` (or `ObjectProvider<OrderService>`) and call `self.saveOne(o)`. It works, but it's a code smell.
4. Call `AopContext.currentProxy()` with `@EnableAspectJAutoProxy(exposeProxy = true)`. This ties your code to Spring AOP.
5. **AspectJ weaving** (`mode = AdviceMode.ASPECTJ`, compile-time or load-time). The bytecode itself is changed, so self-calls are advised too. It's rarely worth the build complexity.

**Common follow-up**: *"If the outer `placeOrders` is `@Transactional`, does the self-call to `saveOne` run in a transaction?"* Yes, but in the **outer** one. The inner annotation is still ignored, so a `REQUIRES_NEW` you were relying on (for example "save each order independently so one failure doesn't roll back the rest") doesn't happen, and one failure rolls back everything.

---

### 8. How would you handle failures in downstream microservices?

Assume every downstream call **will** fail in each of these ways: slow, erroring, down, or returning garbage. The goal is to keep one failing dependency from spreading through your service and taking it down too. Use these layers, in this order:

**1. Timeouts on every call. This is non-negotiable.** Set a connect timeout (about 1 s) and a read/response timeout derived from the dependency's p99 latency and *your own* latency budget. The defaults for many HTTP clients are infinite or very long. With no timeout, a slow dependency ties up your request threads until you run out of them, so a slow dependency is often more dangerous than a dead one. Pass deadlines along the call chain: if the caller has 300 ms left, don't give the next hop 2 s.

**2. Retries, only when safe.** Retry only **transient** errors (connect failures, 503, 429, timeouts) and only **idempotent** operations, or non-idempotent ones that carry an idempotency key. Use **exponential backoff with jitter**, a small attempt limit (2–3), and ideally a *retry budget* (for example, retries may be at most 10% of traffic). Retry at **one layer only**: if each of 3 layers retries 3 times, one user request becomes 27 downstream calls, a "retry storm" that finishes off a struggling service. Never retry 4xx errors other than 408 and 429.

**3. Circuit breaker.** Track the failure rate and slow-call rate over a sliding window. When it crosses a threshold (for example 50% over 20 calls), go **OPEN**: fail immediately without calling the dependency, which protects both your threads and the struggling dependency. After a wait period, go **HALF-OPEN** and let a few trial calls through. If they succeed go **CLOSED**, otherwise back to OPEN.

**4. Bulkheads.** Give each dependency its own limited concurrency (a semaphore or separate thread pool) so that a slow payment provider can't use up the threads that serve product browsing. See `07-microservices-architecture.md` Q13.

**5. Fallbacks and graceful degradation.** Decide per call what failure should look like. Options: a cached or stale value (last known prices), a default (empty recommendations), a reduced feature (hide reviews), or a clear error with a fast fail. For **critical writes** (payments), don't make up a success. Return a clear "pending" or "try again" answer, or queue the work.

**6. Go asynchronous where the user doesn't need to wait.** If the user doesn't need the result right away (emails, analytics, inventory sync), publish an event to Kafka or a queue instead of calling the service. The message broker absorbs the outage, consumers catch up later, and failed messages go to a **DLQ** after N attempts.

**7. Protect yourself and the dependency**: rate-limit outbound calls, shed load early when you're saturated (return 503 with `Retry-After` instead of queueing without limit), and use health checks plus outlier detection in the load balancer or service mesh to take bad instances out of rotation.

**8. Consistency for multi-step flows**: if step 3 of 4 fails, you need sagas with compensating actions, idempotent consumers, and the outbox pattern, not hope (system design file §2).

**9. Observability**: distributed tracing to see which hop failed, metrics per dependency (latency, error rate, breaker state), and alerts on breaker state changes.

**Spring Boot implementation (Resilience4j):**
```java
@CircuitBreaker(name = "inventory", fallbackMethod = "stockFallback")
@Retry(name = "inventory")
@Bulkhead(name = "inventory")
public Stock getStock(String sku) {
    return inventoryClient.getStock(sku);     // client configured with connect/read timeouts
}

private Stock stockFallback(String sku, Throwable t) {
    return stockCache.getOrDefault(sku, Stock.unknown(sku));
}
```
```yaml
resilience4j:
  circuitbreaker.instances.inventory:
    sliding-window-size: 20
    failure-rate-threshold: 50
    slow-call-duration-threshold: 800ms
    slow-call-rate-threshold: 50
    wait-duration-in-open-state: 10s
    permitted-number-of-calls-in-half-open-state: 3
  retry.instances.inventory:
    max-attempts: 3
    wait-duration: 200ms
    enable-exponential-backoff: true
    enable-randomized-wait: true          # jitter
    retry-exceptions: [java.io.IOException, java.util.concurrent.TimeoutException]
  bulkhead.instances.inventory:
    max-concurrent-calls: 25
```
By default Resilience4j applies these in the order `Retry(CircuitBreaker(RateLimiter(TimeLimiter(Bulkhead(call)))))`. Retry is the outermost layer, so once the breaker opens, retries fail fast instead of hammering the dependency. A service mesh (Istio/Linkerd) can provide timeouts, retries, and outlier detection at the infrastructure level instead. Just don't configure retries in both places.

---

### 9. How would you troubleshoot a slow API in production?

A good answer is a **method**: narrow the scope, follow the evidence down the layers, fix the cause, and prove the fix worked. Don't start by guessing "add an index" or "add a cache".

**Step 1 — Measure exactly what "slow" means.**
- Which endpoint(s)? All of them, or one? All users, or one tenant, region, or request size?
- Is it p50, or only p99? If only the tail is slow, suspect GC pauses, lock contention, pool waits, or a slow shard. If everything is slow, suspect a shared dependency or saturation.
- **When did it start?** Line it up against deploys, config changes, traffic growth, data growth (a table crossing a size where the query plan changes), batch jobs, and infrastructure events. "It started at 14:05 and we deployed at 14:03" solves half of all incidents. If you can't diagnose quickly, **roll back first**.

**Step 2 — Find where the time goes (latency breakdown).**
- Pick a **distributed trace** for a slow request (OpenTelemetry, Jaeger, Zipkin, Datadog APM). The span waterfall shows whether the time is in your code, a DB query, a downstream call, or *gaps* between spans. Gaps usually mean queueing or waiting for a thread or connection.
- If you have no tracing, use structured logs with timings around each external call, plus the APM's per-endpoint breakdown.

**Step 3 — Look at the layer that's responsible.**

| Layer | What to check | Typical causes |
|---|---|---|
| **Database** | Slow query log / `pg_stat_statements`, `EXPLAIN ANALYZE`, lock waits, replication lag | Missing or unused index, plan change after data growth or stale statistics, **N+1 queries from an ORM** (100 queries for 100 rows; turn on SQL logging and count), `SELECT *` on wide rows, row lock contention, a long transaction blocking others |
| **Connection pool** | HikariCP `hikaricp.connections.pending`, `…acquire` time, active vs max | Pool too small, connections leaked or held during remote calls inside `@Transactional`, so requests *queue for a connection* |
| **Downstream calls** | Per-dependency latency in traces/metrics | A slow dependency, missing timeouts, calls made one after another that could run in parallel, a retry storm |
| **Application / JVM** | CPU profile (async-profiler, JFR), **several thread dumps a few seconds apart** (`jcmd <pid> Thread.print`), GC logs | Hot loops, heavy serialization (large JSON payloads), regexes, logging inside hot paths, `synchronized` contention, long GC pauses or a heap close to full (see the system design file §3) |
| **Server threads** | Tomcat busy threads vs `server.tomcat.threads.max`, request queue | Thread pool saturated, so requests wait before any code even runs |
| **Infrastructure** | CPU throttling in Kubernetes (`container_cpu_cfs_throttled_seconds_total`), node noisy neighbours, network retransmits, DNS latency, LB queue | CPU limit set too low (throttling causes latency spikes while average CPU looks fine), cross-AZ/region calls, TLS handshakes on every request (no keep-alive) |

**Step 4 — Form a hypothesis, test it, fix it.** Reproduce it in staging with production-like data if you can. Make **one** change at a time: add or fix the index, batch the N+1 with `JOIN FETCH`/`@EntityGraph`/batch size, parallelize independent calls, cache hot reads, add timeouts, resize the pool, paginate, or move work to async.

**Step 5 — Verify and prevent.** Compare p50/p95/p99 before and after on the same dashboard. Add an SLO alert on p99 latency, add a regression test or load test, and keep the slow-query log turned on. Write up the root cause.

**Tools worth naming for a Java/Spring service**: Spring Boot Actuator plus Micrometer → Prometheus/Grafana (`http.server.requests` percentiles, Hikari metrics, JVM metrics), OpenTelemetry tracing, `jcmd`/JFR/async-profiler, Hibernate statistics, the database's `EXPLAIN ANALYZE`. (`../devops/14-scenario-troubleshooting.md` Q5 and Q6 cover the DB-CPU and "latency doubled with no errors" variants.)
