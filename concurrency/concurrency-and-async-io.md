# Concurrency, Parallelism, and Async I/O for Backend Engineers

> OS fundamentals and IPC are in [`os-sysdesign-ipc.md`](../os-sysdesign-ipc.md) §1. Interview Q&A: [`interview-prep/backend-engineer/05-concurrency-multithreading-async.md`](../interview-prep/backend-engineer/05-concurrency-multithreading-async.md). This note explains **how backend runtimes handle many concurrent requests**, from kernel I/O mechanisms up to language-level models, and the bugs that come with them.

## Table of Contents
1. [Concurrency vs Parallelism](#vs)
2. [Processes, Threads, Green Threads, Coroutines](#units)
3. [CPU-Bound vs I/O-Bound Work](#bound)
4. [I/O Models: Blocking, Non-Blocking, Multiplexing, Async](#io)
5. [epoll, kqueue, IOCP, io_uring](#kernel)
6. [The Event Loop (Node.js, Python asyncio, Netty)](#eventloop)
7. [Server Concurrency Architectures](#servers)
8. [Language Runtimes Compared](#runtimes)
9. [Synchronization Primitives](#sync)
10. [Concurrency Bugs: Races, Deadlocks, Livelocks, Starvation](#bugs)
11. [Memory Models, Visibility, Atomics](#memory)
12. [Lock-Free and Concurrent Data Structures](#lockfree)
13. [Concurrency Patterns](#patterns)
14. [Thread Pool Sizing](#pools)
15. [Async Pitfalls in Real Services](#pitfalls)
16. [Interview Questions](#qa)

---

## 1. Concurrency vs Parallelism {#vs}

- **Concurrency**: *dealing with* many things at once. Tasks make progress in overlapping time periods (possibly interleaved on one core). It's a property of program **structure**.
- **Parallelism**: *doing* many things at once. Tasks execute simultaneously on multiple cores. It's a property of **execution**.
- (Rob Pike: "Concurrency is about structure, parallelism is about execution.")

A Node.js server handling 10k connections on one thread is highly concurrent but not parallel. A matrix multiplication across 16 cores is parallel.

---

## 2. Units of Execution {#units}

| Unit | Scheduled by | Memory | Switch cost | Notes |
|---|---|---|---|---|
| **Process** | OS | Own address space | High (TLB flush, page tables) | Isolation; IPC needed; fork/prefork servers (Postgres, Gunicorn sync workers, Apache prefork) |
| **OS thread** (kernel thread) | OS (preemptive) | Shared address space; stack ~1–8 MB reserved (lazily committed) | ~1–5 µs context switch | True parallelism; Java platform threads, pthreads, C# threads |
| **Green threads / virtual threads / goroutines** | Language runtime (M:N onto OS threads) | Small growable stacks (Go: starts at 2–8 KB) | ~100–200 ns | Millions possible; Go goroutines, Java 21 virtual threads, Erlang processes |
| **Coroutines / async tasks** (stackless) | Event loop / executor (cooperative) | State machine object (bytes–KB) | Very cheap | `async/await` in JS, Python, C#, Rust, Kotlin coroutines; yield only at `await` points |

**Preemptive** (the OS or runtime can interrupt anytime: threads, goroutines, which are preemptible since Go 1.14) vs **cooperative** (a task runs until it yields: async/await coroutines, so a CPU-heavy coroutine blocks the whole event loop).

---

## 3. CPU-Bound vs I/O-Bound {#bound}

Most backend request handling is **I/O-bound**: waiting on DB queries, HTTP calls, caches, and disks. The CPU is idle during waits, so the goal is to **handle many waiting requests cheaply** (concurrency without a thread per waiting request).

CPU-bound work (image processing, JSON serialization of huge payloads, encryption, compression, ML inference, report generation) needs **parallelism** across cores. It shouldn't run on event loop threads, so offload it to worker pools, processes, or separate services.

Little's Law for servers: concurrent requests in flight = throughput × latency. 10,000 rps × 200 ms average latency (mostly waiting on I/O) = **2,000 concurrent requests**. With one OS thread per request, that's 2,000 threads (feasible but heavy). With async or virtual threads, it's trivial.

---

## 4. I/O Models {#io}

Stevens' five Unix I/O models:
1. **Blocking I/O**: `read()` blocks the thread until data arrives. Simple, and it needs one thread per concurrent connection.
2. **Non-blocking I/O**: `read()` returns `EAGAIN` immediately if there's no data, so the app polls (busy-waiting is wasteful on its own).
3. **I/O multiplexing**: `select`/`poll`/`epoll`/`kqueue` block on **many** file descriptors at once and report which are ready. One thread handles thousands of connections. This is the foundation of event loops (the **Reactor pattern**).
4. **Signal-driven I/O**: rarely used.
5. **Asynchronous I/O**: submit an operation, and the kernel notifies when it's **completed** (the **Proactor pattern**). Windows IOCP, Linux io_uring (and the old POSIX AIO, which was poor for sockets).

Readiness-based (epoll: "socket is readable, now you read") vs completion-based (io_uring/IOCP: "here's the data you asked for").

---

## 5. Kernel Mechanisms {#kernel}

| Mechanism | OS | Complexity | Notes |
|---|---|---|---|
| `select` | POSIX | O(n) per call, FD_SETSIZE limit (1024) | Legacy |
| `poll` | POSIX | O(n) per call, no hard limit | Legacy |
| **`epoll`** | Linux | O(1) per ready event; register FDs once (`epoll_ctl`), wait (`epoll_wait`) | Powers Nginx, Node (libuv), Netty, Go netpoller, Redis, HAProxy. Edge-triggered vs level-triggered modes |
| **`kqueue`** | BSD/macOS | Similar to epoll, more general (files, signals, timers) | |
| **IOCP** | Windows | Completion-based | .NET, libuv on Windows |
| **`io_uring`** | Linux 5.1+ | Shared ring buffers between user and kernel (submission + completion queues); batches syscalls, true async for files and sockets, zero/few syscalls under load | Used by newer runtimes (tokio-uring, Glommio, Monoio), TigerBeetle, PG 18 AIO option, high-performance proxies. Security concerns led some environments (e.g., Google production, ChromeOS, Docker's default seccomp) to restrict it |

Disk I/O caveat: regular file reads are always "ready" for epoll (buffered I/O blocks inside the kernel), so runtimes use **thread pools** for file I/O (libuv's threadpool, default 4 threads via `UV_THREADPOOL_SIZE`; it also handles DNS `getaddrinfo`, crypto, and zlib in Node!) or io_uring.

---

## 6. The Event Loop {#eventloop}

A single thread runs a loop:
```
while true:
    run expired timers
    run ready callbacks / resume coroutines whose awaited I/O completed
    poll the OS (epoll_wait) for I/O readiness, with a timeout until the next timer
```
**Node.js (libuv) phases**: timers → pending callbacks → idle/prepare → **poll** (I/O) → check (`setImmediate`) → close callbacks. Between each callback, the **microtask queue** runs (`process.nextTick` first, then Promise reactions). Starving the loop with recursive microtasks or `nextTick` blocks I/O.

**The golden rule: never block the event loop.** While one callback runs, no other request makes progress. Blockers include:
- CPU-heavy work (big `JSON.parse`/`stringify`, sorting large arrays, crypto with sync APIs, regexes with catastrophic backtracking (**ReDoS**), image processing).
- Synchronous APIs (`fs.readFileSync`, `bcrypt.hashSync`, sync DB drivers in Python asyncio code, `time.sleep()` in async Python, `requests` inside `async def`).
- Long loops over large datasets.

Mitigations: worker threads (`worker_threads`, Python `asyncio.to_thread`/`run_in_executor`, process pools), chunking work with `setImmediate` yields, async versions of libraries, moving CPU work to separate services or queues.

Monitor **event loop lag/delay** (`perf_hooks.monitorEventLoopDelay` in Node; asyncio debug mode warns about slow callbacks > 100 ms).

Python asyncio specifics: `async def` functions run only when awaited or scheduled (`asyncio.create_task`). Use `asyncio.gather` for concurrent awaits, `asyncio.TaskGroup` (3.11+) for structured concurrency, async drivers (asyncpg, httpx/aiohttp, aioredis/redis.asyncio), and uvloop for a faster loop. The GIL is irrelevant for I/O concurrency but limits CPU parallelism (see below).

---

## 7. Server Concurrency Architectures {#servers}

| Architecture | How | Examples | Trade-offs |
|---|---|---|---|
| **Process per request** (CGI, fork) | Fork a process per connection | Old CGI, Postgres backend-per-connection | Heavy, strong isolation |
| **Prefork process pool** | N worker processes, each handles one request at a time | Apache prefork + mod_php, Gunicorn sync workers, PHP-FPM, Unicorn | Simple, crash isolation; concurrency = number of processes (memory-heavy) |
| **Thread per request (pool)** | Pool of N threads, each blocks on I/O | Tomcat/Spring MVC (200 threads default), Puma (threads), Django/Flask under threaded servers, ASP.NET classic | Simple blocking code; concurrency capped by threads; context switching & memory at high counts |
| **Event loop (single-threaded)** | One thread multiplexes all connections | Node.js, Redis, Nginx worker (one loop per worker process), Python asyncio/uvicorn | Very efficient for I/O; CPU work blocks everything; one core per loop (run N processes for N cores: Node cluster/PM2, Gunicorn+Uvicorn workers) |
| **Multi-reactor / event loop per core** | One loop per core, connections distributed | Nginx (worker per core), Netty (boss + worker event loop groups), Vert.x, Envoy (worker threads), Seastar/ScyllaDB (shard-per-core, share-nothing) | Max efficiency; complex programming model |
| **M:N scheduler with blocking-style code** | Lightweight threads multiplexed over OS threads; runtime parks them on I/O transparently | **Go** (goroutines + netpoller), **Java 21 virtual threads** (Loom), Erlang/Elixir BEAM, Kotlin coroutines (structured, suspend-based) | Write simple sequential code, get async efficiency; runtime complexity hidden |
| **Async/await on multi-threaded executor** | Work-stealing thread pool running futures | Rust **tokio**, .NET async (thread pool), Java CompletableFuture/Reactor (WebFlux) | High performance; "function coloring" (async spreads through call graph) |

Netflix's Zuul 1 (blocking) → Zuul 2 (Netty async) migration found async gave better connection scalability but much harder debugging. That's one reason virtual threads were welcomed: blocking-style code with async-like scalability.

---

## 8. Language Runtimes Compared {#runtimes}

### Go
- Goroutines (cheap, growable stacks), with the **GMP scheduler**: G = goroutine, M = OS thread, P = logical processor (`GOMAXPROCS` = number of cores, container-aware since Go 1.25). Work stealing between Ps.
- The **netpoller** integrates epoll/kqueue: a goroutine blocking on a socket is parked, and the thread runs other goroutines.
- Communication: **channels** ("share memory by communicating") plus `sync` (Mutex, RWMutex, WaitGroup, Once, Cond, `sync/atomic`), `context.Context` for cancellation and deadlines, `errgroup` for structured fan-out with error propagation.
- Pitfalls: **goroutine leaks** (blocked forever on a channel nobody writes to, or forgotten without cancellation), data races (run tests with `-race`), unbuffered channel deadlocks ("all goroutines are asleep"), a shared map without a lock (a fatal concurrent map write), and closure capture of loop variables (fixed in Go 1.22).

### Java / JVM
- Platform threads (1:1 with OS threads) with thread pools (`ExecutorService`), `CompletableFuture`, and reactive (Project Reactor/WebFlux, RxJava): a non-blocking, callback-style programming model.
- **Virtual threads (Java 21, Project Loom)**: millions of cheap threads, blocking calls unmount the virtual thread from its carrier, and you write plain blocking code (JDBC, HTTP clients). Spring Boot 3.2+: `spring.threads.virtual.enabled=true`.
  - Caveats: **pinning** (blocking inside `synchronized` blocks pinned the carrier thread until **JDK 24 fixed it** (JEP 491); native calls still pin), ThreadLocal-heavy code (many virtual threads × big thread locals = memory; consider **ScopedValues**), and don't pool virtual threads (create one per task). Downstream limits (DB pool size) still apply: virtual threads let you *wait* cheaply, they don't make the DB faster.
  - Structured concurrency API (`StructuredTaskScope`, preview).
- `java.util.concurrent`: ConcurrentHashMap, locks (ReentrantLock, ReadWriteLock, StampedLock), atomics, LongAdder (contended counters), CountDownLatch, Semaphore, CyclicBarrier, Phaser, BlockingQueues, ForkJoinPool (work-stealing, used by parallel streams).

### Python
- **GIL (Global Interpreter Lock)** in CPython: only one thread executes Python bytecode at a time. Threads still help for **I/O-bound** work (the GIL is released during blocking I/O), but **not for CPU-bound** work. Use `multiprocessing`/`ProcessPoolExecutor`, or C extensions (NumPy releases the GIL).
- **Free-threaded CPython** (PEP 703): an experimental no-GIL build in 3.13, officially supported (no longer experimental) in 3.14. Ecosystem compatibility is still maturing.
- asyncio for high-concurrency I/O (FastAPI/Starlette, aiohttp), threads (`ThreadPoolExecutor`) for blocking libraries, processes for CPU.
- WSGI servers (Gunicorn sync/gthread workers) vs ASGI (Uvicorn, Hypercorn, Granian) for async frameworks.

### Node.js
- A single-threaded event loop (V8) + libuv thread pool for fs, DNS, crypto, and zlib.
- `worker_threads` for CPU parallelism (separate isolates, communication via messages and SharedArrayBuffer), `cluster` / PM2 / multiple containers to use all cores.
- Promises/async-await, plus streams with backpressure (`pipeline`).

### Rust
- Fearless concurrency: the ownership system and the `Send`/`Sync` traits prevent data races **at compile time**.
- Async with **tokio** (multi-threaded work-stealing runtime) or async-std/smol. `async fn` returns a Future that does nothing until polled.
- Pitfalls: blocking inside async tasks (use `spawn_blocking`), holding a `std::sync::Mutex` guard across `.await`, and cancellation semantics (dropping a future cancels it at the next await point).

### Erlang / Elixir (BEAM)
- Millions of isolated lightweight processes with **no shared memory**, communicating by message passing (the actor model), and preemptive scheduling (reduction counting), so one busy process can't block others.
- "Let it crash" + **supervisors** restart failed processes. Hot code upgrades. WhatsApp and Discord (Elixir) used it for massive concurrent connection counts.

### C# / .NET
- async/await over the thread pool (Task-based), `ConfigureAwait`, `ValueTask`, channels (`System.Threading.Channels`), `Parallel.ForEachAsync`. Pitfall: **sync-over-async** (`.Result`/`.Wait()` on tasks) causes deadlocks (with a synchronization context) and thread pool starvation.

---

## 9. Synchronization Primitives {#sync}

| Primitive | Purpose |
|---|---|
| **Mutex / lock** | Mutual exclusion: one holder at a time for a critical section |
| **Read-write lock** | Many readers or one writer; helps read-heavy shared data (writer starvation risk) |
| **Spinlock** | Busy-wait; only for very short critical sections in low-level code |
| **Semaphore** | Counter permitting N concurrent holders (limit concurrent DB calls to 10, bound parallelism) |
| **Condition variable** | Wait until a condition becomes true (always in a loop: spurious wakeups) |
| **Barrier / latch / WaitGroup** | Wait for N tasks to reach a point / complete |
| **Atomic operations** | Lock-free single-variable updates: CAS (compare-and-swap), fetch-add |
| **Channels / queues** | Pass ownership of data between tasks (CSP style); bounded queues provide backpressure |
| **Once / lazy init** | Initialize exactly once safely (`sync.Once`, double-checked locking with volatile) |
| **Thread-local / context-local storage** | Per-thread/task state (request context, MDC) — be careful with thread pools and async (values leak between requests if not cleared; use contextvars in Python, AsyncLocalStorage in Node) |

---

## 10. Concurrency Bugs {#bugs}

- **Race condition**: the outcome depends on timing. Classic **check-then-act** (`if not exists: create`) and **read-modify-write** (`count = count + 1` is three operations: read, add, write). These happen both in memory and in distributed form against databases (see [`databases/fundamentals/03`](../databases/fundamentals/03-transactions-isolation-concurrency.md)).
- **Data race**: concurrent unsynchronized access to the same memory with at least one write. It's undefined behavior in C/C++/Go (memory model). Use race detectors (Go `-race`, ThreadSanitizer, Java tools like jcstress for testing primitives).
- **Deadlock**: a cycle of tasks each waiting for a lock another holds. **Coffman conditions** (all four required): mutual exclusion, hold-and-wait, no preemption, circular wait. Prevent with **global lock ordering**, lock timeouts (`tryLock`), fewer locks or coarser locks, and avoiding calls to unknown code (callbacks) while holding locks.
- **Livelock**: tasks keep reacting to each other without progress (both stepping aside in a corridor). Fix with randomized backoff.
- **Starvation**: a task never gets the resource (unfair locks, priority inversion, writer starvation in RW locks). Use fair locks and aging.
- **Priority inversion**: a low-priority task holds a lock needed by a high-priority task while a medium-priority task runs (Mars Pathfinder, 1997). Fix with priority inheritance.
- **Thundering herd**: many waiters wake for one event (accept on many processes; mitigated by `EPOLLEXCLUSIVE`/`SO_REUSEPORT`; cache stampedes at the application level).
- **Lost wakeup**: signaling a condition before the waiter waits. Always check the condition under the lock in a loop.
- **ABA problem** (lock-free code): a CAS succeeds because the value changed A→B→A. Fix with tagged pointers or version counters, or hazard pointers / epoch-based reclamation.
- **Async-specific**: forgotten `await` (a coroutine never runs, or a promise rejection goes unhandled), interleaving at every `await` (state may change between awaits even single-threaded, so "async code has race conditions too"), and unbounded `Promise.all` over 100k items (overwhelms downstream; use a concurrency limit like p-limit or a semaphore).

---

## 11. Memory Models, Visibility, Atomics {#memory}

Modern CPUs and compilers **reorder** instructions and cache values in registers and per-core caches. Without synchronization, one thread's writes may not be visible to another, or may appear in a different order.
- A **memory model** (Java JMM, C++11, Go memory model) defines **happens-before** relationships: a write is guaranteed visible to a read only if they're ordered by synchronization (unlocking a mutex happens-before a subsequent lock; a volatile/atomic write happens-before a subsequent read of it; channel send happens-before receive; thread start/join).
- Java: `volatile` gives visibility and ordering (not atomicity for compound operations), `final` fields have safe publication guarantees, and `AtomicInteger`/`VarHandle` give atomic ops with memory ordering.
- C++/Rust atomics have explicit orderings: `Relaxed`, `Acquire`/`Release`, `SeqCst`.
- Classic bug: **double-checked locking** without volatile in Java can expose a partially constructed object.
- **False sharing**: two threads writing different variables on the same CPU cache line (64 bytes) cause cache-line ping-pong, which destroys performance. Fix with padding (`@Contended` in Java, alignment). LongAdder avoids contention by striping counters.

---

## 12. Lock-Free and Concurrent Data Structures {#lockfree}

- **Lock-free**: system-wide progress is guaranteed (some thread always makes progress). **Wait-free**: every thread makes progress in bounded steps. Built on CAS loops.
- Examples: `ConcurrentHashMap` (Java: fine-grained locking + CAS), `ConcurrentLinkedQueue` (Michael–Scott queue), LMAX **Disruptor** (a ring buffer for ultra-low-latency messaging), Go `sync.Map` (optimized for append-mostly or disjoint-keys patterns), crossbeam (Rust), RCU (read-copy-update in the Linux kernel: readers never block).
- **Copy-on-write** structures for read-mostly data (config snapshots swapped atomically via an atomic reference).
- Rule: prefer well-tested library structures. Writing your own lock-free code is extremely error-prone.

---

## 13. Concurrency Patterns {#patterns}

- **Worker pool**: a bounded number of workers consuming from a queue. It bounds resource use (DB connections, CPU).
- **Producer–consumer** with a bounded queue (backpressure).
- **Fan-out/fan-in**: launch parallel subtasks and gather results (Go `errgroup`, `Promise.all`, `asyncio.gather`, `CompletableFuture.allOf`, `StructuredTaskScope`). Always bound concurrency and propagate cancellation on first error (or decide on partial results).
- **Pipeline**: stages connected by channels or queues, each stage concurrent.
- **Scatter-gather with timeout**: query N backends, return what's ready by the deadline (search, price comparison).
- **Single-flight / request coalescing**: concurrent requests for the same key share one in-flight fetch (Go `singleflight`, cache stampede protection).
- **Actor model**: each actor owns its state and processes messages sequentially (no locks inside). Akka, Orleans, Erlang, Cloudflare Durable Objects.
- **Futures/promises** and **reactive streams** (backpressure-aware async sequences).
- **Structured concurrency**: child tasks can't outlive their parent scope, and errors and cancellation propagate (Trio/AnyIO nurseries, Python TaskGroup, Kotlin coroutineScope, Java StructuredTaskScope, Swift task groups). It prevents leaked background tasks.
- **Rate limiting / semaphores** around downstream calls.
- **Optimistic concurrency** with versioned compare-and-swap instead of locks (in memory with atomics, or in DBs with version columns).
- **Sharding state by key**: route all operations for a key to the same thread/partition (single-writer principle), which avoids locks entirely (Kafka partitions, actor per entity, Redis being single-threaded per shard).

---

## 14. Thread Pool Sizing {#pools}

Brian Goetz's formula (*Java Concurrency in Practice*):
```
threads = cores × target_utilization × (1 + wait_time / compute_time)
```
- CPU-bound: threads ≈ cores (+1).
- I/O-bound: much larger. If requests wait 90 ms on I/O per 10 ms of CPU on 8 cores: 8 × 1 × (1 + 9) = 80 threads.
- Then validate under load (see Little's Law in the DB pooling note).

Separate pools by workload type (bulkheads): HTTP request threads, outbound HTTP client threads, DB connection pool, CPU work pool, scheduled tasks. A blocking call on the wrong pool (e.g., blocking in Netty's event loop, or in the ForkJoin common pool) starves everything.

Size **downstream pools** consistently: if 200 request threads each need a DB connection and the pool has 20, 180 threads wait. That's fine if bounded and timed out, and catastrophic if not.

---

## 15. Async Pitfalls in Real Services {#pitfalls}

1. **Blocking calls inside async code** (sync DB driver in FastAPI `async def`, `requests` in asyncio, JDBC in a WebFlux handler, `.Result` in C#). It stalls the loop or executor, and the symptom is latency spikes across *all* endpoints under modest load.
2. **Unbounded concurrency**: `await asyncio.gather(*[fetch(u) for u in 100_000_urls])` opens 100k sockets. Use a semaphore or bounded worker pool.
3. **Lost errors**: fire-and-forget tasks whose exceptions are never observed (unhandled promise rejections; Python "Task exception was never retrieved"). Keep references, await them, or attach error handlers. Use structured concurrency.
4. **Missing timeouts/cancellation**: awaiting forever. Wrap with `asyncio.wait_for`, `AbortSignal.timeout`, `context.WithTimeout`, and propagate deadlines.
5. **Context propagation** across async boundaries: request IDs and trace context must flow (AsyncLocalStorage in Node, contextvars in Python, OpenTelemetry context). Thread-locals break with async and thread pools.
6. **Shared mutable state** across awaits: an `await` is a yield point, so re-validate state after awaiting.
7. **Connection pool exhaustion** when concurrency (cheap tasks/virtual threads) far exceeds pool size, causing long waits. Size concurrency to downstream capacity, not to how many tasks you *can* create.
8. **CPU work on the event loop**: JSON serialization of a 50 MB payload, bcrypt hashing synchronously, regex on user input (ReDoS). Use streaming, worker threads, async/native implementations, and safe regex engines (RE2).
9. **Memory growth** from buffering everything (reading whole request or response bodies into memory). Use streams with backpressure.
10. **Mixing sync and async libraries** incorrectly, leading to deadlocks (sync-over-async) or nested event loop errors.

---

## 16. Interview Questions {#qa}

1. Concurrency vs parallelism? Give an example of each.
2. How can Node.js handle 10,000 concurrent connections with one thread? What happens if a request does heavy CPU work?
3. Explain epoll and why it scales better than select/poll. What does io_uring add?
4. Thread-per-request vs event loop vs virtual threads: trade-offs?
5. What is the GIL? How do you achieve CPU parallelism in Python?
6. What are Java virtual threads, and what are their pitfalls (pinning, ThreadLocals, downstream limits)?
7. Explain how a deadlock occurs (Coffman conditions) and how to prevent it.
8. What's the difference between a race condition and a data race?
9. Why is `count++` not thread-safe? Fix it in three ways.
10. How do you size a thread pool for I/O-bound work?
11. Can single-threaded async code have race conditions? Example?
12. What is structured concurrency and what problem does it solve?
