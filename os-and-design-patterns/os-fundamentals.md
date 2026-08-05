# OS Fundamentals: Processes, Threads, Memory, and Scheduling

The kernel-level mechanisms that manage hardware, isolate programs from each other, and decide who gets the CPU and when. These are "low-level" because they sit right at the hardware-software boundary — every language runtime, container, and distributed system is ultimately built on top of this.

## TL;DR
- A **process** is an isolated running instance of a program (code, data, stack, heap, PC); a **thread** is a lightweight unit of execution inside a process, sharing its address space.
- Processes move through states: New → Ready → Running → Waiting → Terminated, managed via the kernel's Process Control Block (PCB).
- **Virtual memory** (paging + page tables + TLB) lets processes use more memory than physically exists and keeps them isolated from each other.
- **CPU scheduling algorithms** (FCFS, SJF, Priority, Round Robin, Multilevel Feedback Queue) decide which ready process runs next — each trades off fairness, throughput, and responsiveness differently.
- Synchronization primitives (locks, atomics, deadlock detection) exist because concurrent access to shared state is unsafe without coordination.

## Processes and threads

### Processes
A process is the basic unit of execution in an OS — an instance of a program in execution, encompassing code, data, stack, heap, and program counter (PC). Processes are isolated via virtual memory to prevent interference between them.

**Process states**: New (created), Ready (waiting for CPU), Running (executing), Waiting (blocked on I/O or an event), Terminated (finished).

```
New → Ready → Running → Terminated
          ↑        ↓
        Waiting ← (I/O or Block)
```

**Process Control Block (PCB)**: the kernel's data structure for each process, storing PID, state, PC, registers, memory limits, and open files.

| Field | Description | Example value |
|---|---|---|
| PID | Unique identifier | 1234 |
| State | Current state | Running |
| PC | Instruction pointer | 0x7FFF |
| CPU registers | Saved context | RAX=0xABCD |
| Memory info | Base/limit registers | Base=0x1000 |
| I/O status | Open files/devices | FD=3 (stdin) |

**Process creation**: via system calls like `fork()` (Unix) or `CreateProcess()` (Windows). `fork()` creates a child process as a duplicate of the parent, sharing code but using copy-on-write (COW) for efficiency — pages aren't actually duplicated until one side writes to them.

```c
pid = fork();
if (pid == 0) {
    // Child process
    execve("/bin/ls", args, env);
} else {
    // Parent process
    waitpid(pid, status, 0);
}
```

Pros: isolation enhances security. Cons: higher overhead than threads (context switching costs roughly 1-10μs).

### Threads
Lightweight processes within a process, sharing the same address space (code, data, files) but with private stacks and program counters. Ideal for concurrency without full isolation.

- **User vs. kernel threads**: user threads (managed by libraries like pthreads) are lightweight but can't be scheduled independently across cores; kernel threads (OS-managed) allow true parallelism.
- **Thread lifecycle**: similar to processes but much faster to create (~1μs vs ~1-10μs for a process).

```python
import threading
def worker(num):
    print(f"Thread {num} started")
threads = [threading.Thread(target=worker, args=(i,)) for i in range(5)]
for t in threads: t.start()
for t in threads: t.join()  # Wait for completion
```

**Multithreading models**:
- **Many-to-One**: many user threads multiplexed onto one kernel thread — lightweight but no true parallelism, and one blocking call blocks all threads.
- **One-to-One**: each user thread maps to a kernel thread — true parallelism, but creation is more expensive (this is what Linux/Windows use today).
- **Many-to-Many**: hybrid, multiplexing many user threads onto a smaller/equal pool of kernel threads — scalable but complex to implement.

**Advanced**: thread affinity (pinning a thread to a specific core) via `sched_setaffinity()` improves cache locality by avoiding cross-core cache invalidation.

## Memory management

### Virtual memory
An abstraction allowing processes to use more memory than is physically available, via paging and swapping, while also isolating each process's address space from every other's.

**Paging**: memory is divided into fixed-size pages (4KB is typical). Virtual addresses translate to physical addresses via a page table.
- **Page Table Entry (PTE)**: includes the physical frame number, a valid bit, a dirty bit, and protection bits (read/write/execute).
- **Translation Lookaside Buffer (TLB)**: a hardware cache for fast virtual-to-physical lookups, typically with a hit rate above 95% — without it, every memory access would require a page table walk.

**Segmentation**: variable-sized segments (code/data/stack) for logical division of memory. More flexible than fixed pages conceptually, but prone to external fragmentation.

**Demand paging**: pages are loaded lazily, only on a page fault. If the working set exceeds physical RAM, **thrashing** occurs — the system spends more time swapping pages in and out than doing real work.

**Page replacement algorithms** (which page to evict when RAM is full):

| Algorithm | Description | Pros/cons |
|---|---|---|
| FIFO | First in, first out | Simple / suffers Belady's anomaly (more frames can mean *more* faults) |
| LRU | Least Recently Used (stack-based) | Good real-world approximation of optimal / high bookkeeping overhead |
| Optimal | Replace the page not needed for the longest time in the future | Theoretically ideal / unrealizable — requires knowing the future |
| Clock (Second Chance) | Circular list with a reference bit | Efficient approximation of LRU / not exact |

**Memory allocation**: heap allocation via `malloc()` (best-fit, first-fit, or buddy-system strategies). Stack grows downward automatically as function calls nest.

**Security**: Address Space Layout Randomization (ASLR) randomizes the base addresses of the heap, stack, and libraries to make memory-corruption exploits harder to land reliably.

**Advanced**: Huge Pages (2MB or 1GB instead of 4KB) reduce TLB misses for memory-intensive workloads (databases, JVMs). NUMA (Non-Uniform Memory Access) matters on multi-socket systems, where memory local to a CPU socket is faster to access than memory attached to a different socket — schedulers and allocators that are NUMA-aware keep threads and their memory on the same node.

## CPU scheduling

**Scheduler types**:
- **Preemptive**: the OS interrupts a running process (e.g., via a timer) to switch to another.
- **Non-preemptive**: a process runs until it voluntarily yields or blocks.

**Algorithms**:

| Algorithm | Goal | Formula/metric | Pros/cons |
|---|---|---|---|
| FCFS (First Come First Served) | Simple queue | Wait time = arrival to start | Fair / convoy effect — one long job blocks everything behind it |
| SJF (Shortest Job First) | Minimize average wait time | Select minimum burst time | Provably optimal for non-preemptive average wait / can starve long jobs |
| Priority Scheduling | Higher priority runs first | Priority queue | Flexible / starvation for low-priority jobs (aging mitigates) |
| Round Robin (RR) | Time-sharing (quantum ~10-100ms) | CPU utilization = (busy time / total time) × 100 | Responsive, fair / overhead if quantum is too small (excessive context switches) |
| Multilevel Queue | Separate queues (e.g., foreground/background) | Varies by queue | Handles mixed workloads well / rigid — a process can't move between queues |
| Multilevel Feedback Queue | Dynamic priority, demotes CPU-bound processes | Aging: `New_Pri = Old_Pri + (Wait_Time / Quantum)` | Adaptive, avoids starvation / complex to tune |

**Metrics**: throughput (jobs/sec), turnaround time (submission to completion), waiting time, response time.

**Advanced**: Real-time scheduling uses different algorithms entirely — Rate Monotonic Scheduling (RMS) for periodic tasks with fixed priorities by period, and Earliest Deadline First (EDF) which dynamically prioritizes by deadline proximity. Linux's Completely Fair Scheduler (CFS) uses a red-black tree keyed on virtual runtime to achieve O(log n) scheduling decisions while approximating ideal fair-share CPU allocation across all runnable tasks.

## File systems and I/O

- **File abstraction**: files as byte streams or records. Inodes (Unix) store metadata — permissions, timestamps, and pointers to the data blocks.
- **I/O operations**: synchronous (blocks the caller until done) vs. asynchronous (non-blocking, e.g. `aio_read()`), which lets a process issue an I/O request and keep doing other work.
- **Buffering**: the kernel buffers data to reduce disk seeks; direct I/O bypasses the kernel buffer cache, which databases often want so they can manage their own caching.
- **Disk scheduling**: SSTF (Shortest Seek Time First) and SCAN (the "elevator algorithm," which sweeps across the disk in one direction before reversing) reduce seek overhead on spinning disks.
- **Advanced**: RAID levels — 0 (striping, performance no redundancy), 1 (mirroring, full redundancy at 2x cost), 5 (striping with distributed parity, tolerates one disk failure) — trade redundancy against usable capacity and performance.

## Synchronization primitives

**Race conditions and critical sections**: occur when shared resources are accessed concurrently without coordination — the classic bug class that all of the below exists to prevent.

**Atomic operations**: hardware instructions like Test-And-Set (TAS) or Compare-And-Swap (CAS) let you build lock-free data structures and synchronization primitives without needing OS-level blocking.

**Deadlock**: four conditions must all hold simultaneously for deadlock to occur — Mutual Exclusion, Hold-and-Wait, No Preemption, and Circular Wait. Breaking any one of them prevents deadlock.
- **Prevention**: the Banker's Algorithm uses a resource allocation graph to only grant requests that keep the system in a "safe state."
- **Detection**: Wait-For Graph cycle detection, run via DFS in O(E+V) — if a cycle exists, deadlock exists.

These low-level concepts form the OS kernel's core. They're implemented differently across kernel architectures: microkernels (modular, minimal kernel — e.g. Minix) keep most services in user space, while monolithic kernels (e.g. Linux) run most services in kernel space for performance at the cost of isolation.

## Quick reference
- Process = isolated execution unit with its own address space. Thread = lightweight execution unit sharing a process's address space.
- Process states: New → Ready → Running → Waiting → Terminated.
- Paging + TLB make virtual memory fast; page replacement algorithms (LRU, Clock) decide what to evict under pressure.
- Scheduling algorithms trade off fairness (RR), throughput (SJF), and responsiveness (MLFQ) — Linux uses CFS (red-black tree, O(log n)).
- Deadlock requires all four conditions (mutual exclusion, hold-and-wait, no preemption, circular wait); break one to prevent it.
- RAID 0 = speed, no redundancy. RAID 1 = mirroring. RAID 5 = striping + parity, survives one disk loss.

## Further reading
- Silberschatz, Galvin, Gagne — *Operating System Concepts* (the standard textbook covering all of the above in depth).
- Linux kernel documentation on CFS (`Documentation/scheduler/`).
- Related: see `ipc.md` in this folder for how processes communicate once isolated, and `design-patterns.md` for higher-level software design.
