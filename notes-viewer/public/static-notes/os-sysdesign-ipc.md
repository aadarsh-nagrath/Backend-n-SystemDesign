# Comprehensive Guide to OS Fundamentals and Design Patterns

Welcome to this **extremely comprehensive and super lengthy guide** on two cornerstone pillars of computer science and software engineering: **OS Fundamentals** (focusing on low-level concepts and Interprocess Communication, or IPC) and **Design Patterns** (covering fundamental software design patterns). This guide is designed to be exhaustive, drawing from foundational principles, historical context, practical examples, code snippets (in pseudocode or Python where relevant), real-world applications, pros/cons, and advanced considerations. Whether you're a student, developer, or architect, this will serve as a deep-dive reference.

I'll structure it into two major sections for clarity. Within each, we'll break down subtopics with subsections, bullet points, tables, and diagrams (described in text for readability). Expect this to be thorough—think of it as a mini-textbook. Let's dive in.

---

## Section 1: OS Fundamentals – Low-Level Concepts and Interprocess Communication (IPC)

Operating Systems (OS) are the backbone of modern computing, acting as intermediaries between hardware and software. This section explores **low-level concepts** (the gritty details of how OS manages resources at a hardware-software boundary) and **IPC** (how processes communicate in a multi-process environment). We'll cover theory, implementation nuances, and edge cases.

### 1.1 Low-Level OS Concepts

Low-level concepts refer to the kernel-level mechanisms that handle hardware abstraction, resource allocation, and system stability. These are "low-level" because they operate close to the hardware, often involving assembly-like instructions or direct memory access.

#### 1.1.1 Processes and Threads
- **Processes**: A process is the basic unit of execution in an OS. It's an instance of a program in execution, encompassing code, data, stack, heap, and program counter (PC). Processes are isolated via virtual memory to prevent interference.
  - **Process States**: Processes cycle through states like New (created), Ready (waiting for CPU), Running (executing), Waiting (I/O or event), and Terminated (finished).
    - Transition Diagram (text-based):
      ```
      New → Ready → Running → Terminated
                ↑        ↓
              Waiting ← (I/O or Block)
      ```
  - **Process Control Block (PCB)**: The kernel's data structure for each process, storing PID (Process ID), state, PC, registers, memory limits, and open files. Example PCB fields:
    | Field          | Description                          | Example Value |
    |----------------|--------------------------------------|---------------|
    | PID            | Unique identifier                    | 1234         |
    | State          | Current state                        | Running      |
    | PC             | Instruction pointer                  | 0x7FFF       |
    | CPU Registers  | Saved context                        | RAX=0xABCD   |
    | Memory Info    | Base/Limit registers                 | Base=0x1000  |
    | I/O Status     | Open files/devices                   | FD=3 (stdin) |

  - **Process Creation**: Via system calls like `fork()` (Unix) or `CreateProcess()` (Windows). `fork()` creates a child process as a duplicate of the parent, sharing code but with copy-on-write (COW) for efficiency.
    - Pseudocode Example:
      ```
      pid = fork();
      if (pid == 0) {
          // Child process
          execve("/bin/ls", args, env);
      } else {
          // Parent process
          waitpid(pid, status, 0);
      }
      ```
    - Pros: Isolation enhances security. Cons: High overhead (context switching ~1-10μs).

- **Threads**: Lightweight processes within a process, sharing the same address space (code, data, files) but with private stacks and PCs. Ideal for concurrency without full isolation.
  - **User vs. Kernel Threads**: User threads (managed by libraries like pthreads) are lightweight but can't be scheduled independently; kernel threads (OS-managed) allow true parallelism.
  - **Thread Lifecycle**: Similar to processes but faster creation (~1μs).
  - Example in Python (using threading module):
    ```python
    import threading
    def worker(num):
        print(f"Thread {num} started")
    threads = [threading.Thread(target=worker, args=(i,)) for i in range(5)]
    for t in threads: t.start()
    for t in threads: t.join()  # Wait for completion
    ```
  - **Multithreading Models**: Many-to-One (user-space multiplexing), One-to-One (kernel handles each), Many-to-Many (hybrid for scalability).
  - Advanced: Thread affinity (pinning to cores) via `sched_setaffinity()` to optimize cache locality.

#### 1.1.2 Memory Management
- **Virtual Memory**: Abstraction allowing processes to use more memory than physically available via paging/swapping.
  - **Paging**: Memory divided into fixed-size pages (4KB typical). Virtual Address → Physical Address via Page Table.
    - Page Table Entry (PTE): Includes frame number, valid bit, dirty bit, protection (R/W/X).
    - Translation Lookaside Buffer (TLB): Hardware cache for fast VA→PA lookup (hit rate >95%).
  - **Segmentation**: Variable-sized segments (code/data/stack) for logical division, but prone to fragmentation.
  - **Demand Paging**: Pages loaded on fault (lazy loading). Thrashing occurs if working set > physical RAM.
    - Replacement Algorithms:
      | Algorithm     | Description                          | Pros/Cons |
      |---------------|--------------------------------------|-----------|
      | FIFO         | First In, First Out                  | Simple / Suffers Belady's anomaly |
      | LRU          | Least Recently Used (stack-based)    | Optimal approx. / High overhead |
      | Optimal      | Replace page not used longest future | Ideal / Unrealizable in practice |
      | Clock (Second Chance) | Circular list with reference bit | Efficient / Approximate |

  - **Memory Allocation**: Heap via `malloc()` (best-fit, first-fit, buddy system). Stack grows downward automatically.
  - Security: Address Space Layout Randomization (ASLR) randomizes base addresses to thwart exploits.
  - Advanced: Huge Pages (2MB/1GB) reduce TLB misses; NUMA (Non-Uniform Memory Access) for multi-socket systems.

#### 1.1.3 CPU Scheduling
- **Scheduler Types**:
  - **Preemptive**: OS interrupts (e.g., timer) to switch processes.
  - **Non-Preemptive**: Process yields voluntarily.
- **Algorithms**:
  | Algorithm          | Goal                                 | Formula/Metric | Pros/Cons |
  |--------------------|--------------------------------------|----------------|-----------|
  | FCFS (First Come First Served) | Simple queue | Wait Time = Arrival to Start | Fair / Convoy effect (long jobs block) |
  | SJF (Shortest Job First) | Minimize wait time | Select min burst time | Optimal for non-preemptive / Starvation for long jobs |
  | Priority Scheduling | Higher priority first | Priority queue | Flexible / Starvation (aging mitigates) |
  | Round Robin (RR)  | Time-sharing (quantum ~10-100ms) | CPU utilization = (Busy Time / Total Time) * 100 | Responsive / Overhead if quantum too small |
  | Multilevel Queue | Separate queues (foreground/background) | Varies by queue | Handles mixed workloads / Rigid assignment |
  | Multilevel Feedback Queue | Dynamic priority (demote on CPU use) | Aging formula: New_Pri = Old_Pri + (Wait_Time / Quantum) | Adaptive / Complex tuning |

- **Metrics**: Throughput (jobs/sec), Turnaround Time (submit to complete), Waiting Time, Response Time.
- Advanced: Real-Time Scheduling (RMS: Rate Monotonic for periodic tasks; EDF: Earliest Deadline First). Linux CFS (Completely Fair Scheduler) uses red-black trees for O(log n) fairness.

#### 1.1.4 File Systems and I/O
- **File Abstraction**: Files as byte streams or records. Inodes (Unix) store metadata (permissions, timestamps, pointers to blocks).
- **I/O Operations**: Synchronous (block until done) vs. Asynchronous (non-blocking, e.g., `aio_read()`).
- **Buffering**: Kernel buffers data to reduce disk seeks. Direct I/O bypasses for databases.
- **Disk Scheduling**: SSTF (Shortest Seek Time First), SCAN (elevator algorithm).
- Advanced: RAID levels (0: striping, 1: mirroring, 5: parity) for redundancy/performance.

#### 1.1.5 Synchronization Primitives
- **Race Conditions and Critical Sections**: Occur when shared resources are accessed concurrently without coordination.
- **Atomic Operations**: Hardware instructions like Test-And-Set (TAS) or Compare-And-Swap (CAS) for lock-free programming.
- **Deadlock Prevention**: Banker's Algorithm (resource allocation graph). Conditions: Mutual Exclusion, Hold-Wait, No Preemption, Circular Wait.
  - Detection: Wait-For Graph cycle detection (O(E+V) via DFS).

These low-level concepts form the OS kernel's core, implemented in microkernels (modular, e.g., Minix) vs. monolithic (integrated, e.g., Linux).

### 1.2 Interprocess Communication (IPC)

IPC enables data exchange and synchronization between processes, crucial for distributed systems and parallelism. Processes can't directly access each other's memory (isolation), so OS provides mechanisms.

#### 1.2.1 Shared Memory
- **Concept**: Multiple processes map the same physical memory region into their virtual address spaces.
- **Implementation**: `shmget()` (allocate), `shmat()` (attach) in POSIX.
- **Pros/Cons**:
  | Pros                  | Cons                          |
  |-----------------------|-------------------------------|
  | Fastest IPC (no copy) | Requires synchronization (e.g., semaphores) to avoid races |
  | Bidirectional         | Security risks if not isolated |

- Example (C-like pseudocode):
  ```
  int shmid = shmget(IPC_KEY, SIZE, IPC_CREAT | 0666);
  char *shm = shmat(shmid, NULL, 0);
  strcpy(shm, "Hello from Process 1");
  // Process 2 reads shm
  shmdt(shm); shmctl(shmid, IPC_RMID, NULL);
  ```
- Advanced: Anonymous shared memory (`mmap()` with MAP_SHARED) for threads; huge pages for performance.

#### 1.2.2 Message Passing
- **Concept**: Processes send/receive messages via OS-mediated queues. Synchronous (blocking) or Asynchronous (non-blocking).
- **Types**: Direct (to specific PID) vs. Indirect (via ports/mailboxes).
- **Primitives**: `send()`/`recv()` in models like MPI (Message Passing Interface) for HPC.
- **Pros/Cons**:
  | Pros                  | Cons                          |
  |-----------------------|-------------------------------|
  | No shared state (safer) | Slower due to copying/routing |
  | Scalable for distributed | Message loss if not buffered |

- Example: Unix `msgget()` for System V message queues.
  ```c
  // Sender
  msgsnd(msqid, &msgbuf, sizeof(msgbuf.mtext), 0);
  // Receiver
  msgrcv(msqid, &msgbuf, sizeof(msgbuf.mtext), type, 0);
  ```
- Advanced: ZeroMQ or RabbitMQ for pub-sub patterns in distributed IPC.

#### 1.2.3 Pipes and Named Pipes (FIFOs)
- **Unnamed Pipes**: Half-duplex, unidirectional channel between related processes (parent-child). Created via `pipe(fd[2])`.
  - Example: `ls | grep txt` (shell uses pipe).
    ```python
    import subprocess
    p1 = subprocess.Popen(['ls'], stdout=subprocess.PIPE)
    p2 = subprocess.Popen(['grep', 'txt'], stdin=p1.stdout)
    p1.stdout.close(); output = p2.communicate()[0]
    ```
- **Named Pipes (FIFOs)**: Bidirectional, persistent, for unrelated processes. `mkfifo mypipe`.
- **Pros/Cons**:
  | Pros                  | Cons                          |
  |-----------------------|-------------------------------|
  | Simple for streams    | Unidirectional (unnamed); blocking by default |
  | Kernel-buffered       | Limited to line-buffered I/O |

- Advanced: Socket pairs (`socketpair()`) for bidirectional pipes.

#### 1.2.4 Semaphores
- **Concept**: Counter-based synchronization for mutual exclusion or signaling. Invented by Dijkstra.
  - Binary (0/1 for locks), Counting (for resource pools).
- **Operations**: Wait (P: decrement if >0, else block), Signal (V: increment).
- **Implementation**: `sem_init()`, `sem_wait()`, `sem_post()`.
- Example (Producer-Consumer):
  ```c
  sem_t mutex, full, empty;
  sem_init(&empty, 0, BUFFER_SIZE);
  // Producer
  sem_wait(&empty); sem_wait(&mutex);
  buffer[in] = item; in++;
  sem_post(&mutex); sem_post(&full);
  ```
- **Pros/Cons**:
  | Pros                  | Cons                          |
  |-----------------------|-------------------------------|
  | Prevents busy-waiting | Priority inversion (low-pri process holds, blocks high-pri) |
  | Atomic operations     | Deadlock if misused |

- Advanced: Named semaphores for IPC; spinlocks for short critical sections.

#### 1.2.5 Signals
- **Concept**: Asynchronous notifications (e.g., SIGINT for Ctrl+C). Process reacts via handler or ignores/blocks.
- **Standard Signals**:
  | Signal   | Meaning                  | Default Action |
  |----------|--------------------------|----------------|
  | SIGKILL | Force terminate          | Terminate     |
  | SIGSEGV | Segmentation fault       | Terminate     |
  | SIGTERM | Graceful termination     | Terminate     |
  | SIGALRM | Timer expire             | Terminate     |

- **Handling**: `signal(SIGINT, handler)` or `sigaction()` for modern use.
  ```c
  void handler(int sig) { printf("Caught SIGINT\n"); exit(0); }
  signal(SIGINT, handler);
  pause();  // Wait for signal
  ```
- **Pros/Cons**:
  | Pros                  | Cons                          |
  |-----------------------|-------------------------------|
  | Lightweight, async    | Unreliable delivery; no data  |
  | Unix standard         | Race conditions in handlers  |

- Advanced: Real-time signals (queued, SIGRTMIN+); sigsuspend() for atomic wait.

#### 1.2.6 Other IPC Mechanisms
- **Sockets**: Network IPC (TCP/UDP) or local (Unix domain). `socket()`, `bind()`, `listen()`, `accept()`.
  - Example: Client-server chat.
- **Remote Procedure Calls (RPC)**: Transparent calls across machines (e.g., gRPC).
- **D-Bus or Android Binder**: For system-level IPC in desktops/mobile.
- **Comparison Table**:
  | Mechanism     | Speed | Complexity | Use Case                  |
  |---------------|-------|------------|---------------------------|
  | Shared Mem   | Highest | High      | High-perf (e.g., databases)|
  | Pipes        | High  | Low       | Stream data (shell cmds)  |
  | Messages     | Med   | Med       | Distributed apps          |
  | Semaphores   | N/A   | Med       | Sync only                 |
  | Signals      | Highest| Low       | Events/notifications      |
  | Sockets      | Med   | High      | Networking                |

- **Challenges in IPC**: Portability (POSIX vs. Win32), Security (e.g., capabilities in Linux), Scalability (e.g., zero-copy via `sendfile()`).
- **Real-World**: Docker uses namespaces/cgroups for isolated IPC; Kubernetes pods leverage message queues for orchestration.

This wraps OS Fundamentals—low-level concepts ensure efficiency and isolation, while IPC enables collaboration. Next, design patterns.

---

## Section 2: Design Patterns – Fundamental Software Design Patterns

Design patterns, popularized by the Gang of Four (GoF) in *Design Patterns: Elements of Reusable Object-Oriented Software* (1994), are proven solutions to common software design problems. They promote maintainability, scalability, and reusability. We'll cover the 23 classic GoF patterns, categorized into **Creational** (object creation), **Structural** (class/object composition), and **Behavioral** (interaction/responsibility). For each: intent, motivation, structure (UML-like text diagram), participants, consequences, implementation (pseudocode/Python), known uses, and relations.

This section is exhaustive—expect deep dives, variants, anti-patterns, and modern adaptations (e.g., for functional programming or microservices).

### 2.1 Creational Patterns
Focus on flexible object creation, decoupling instantiation from usage.

#### 2.1.1 Singleton
- **Intent**: Ensure a class has only one instance and provide global access.
- **Motivation**: For resources like loggers or caches where multiples waste resources.
- **Structure**:
  ```
  +Singleton
  -instance: Singleton
  +getInstance(): Singleton  // Thread-safe
  +operation()
  ```
- **Participants**: Singleton class.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Controlled access             | Global state hard to test     |
  | Lazy initialization           | Violates Single Responsibility|
- **Implementation** (Thread-safe Python):
  ```python
  class Singleton:
      _instance = None
      def __new__(cls):
          if cls._instance is None:
              cls._instance = super().__new__(cls)
          return cls._instance
  s = Singleton()  # Same instance always
  ```
- **Known Uses**: Java's Runtime, database connections.
- **Relations**: Like Monostate (shared state, multiple instances).
- **Advanced**: Bill Pugh singleton (inner enum class) for JVM; anti-pattern if overused (leads to god objects).

#### 2.1.2 Factory Method
- **Intent**: Define an interface for creating objects but let subclasses decide instantiation class.
- **Motivation**: Framework defines creation without binding to concrete classes (e.g., GUI toolkits).
- **Structure**:
  ```
  Creator --(defines)-- abstract factoryMethod(): Product
            |
  ConcreteCreator --(implements)-- factoryMethod(): ConcreteProduct
  Product --(uses)--
  ```
- **Participants**: Creator, ConcreteCreator, Product, ConcreteProduct.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Promotes loose coupling       | Subclasses proliferate        |
  | Supports extension            | Code duplication if variants few |
- **Implementation** (Python for shapes):
  ```python
  class Shape: def draw(self): pass
  class Circle(Shape): def draw(self): print("Circle")
  class ShapeFactory:
      def create_shape(self, type):
          if type == "circle": return Circle()
  factory = ShapeFactory(); shape = factory.create_shape("circle"); shape.draw()
  ```
- **Known Uses**: Java's Calendar.getInstance(), document creators in apps.
- **Relations**: Variant of Abstract Factory; Template Method for algorithm skeleton.
- **Advanced**: Parametric Factory Method (pass type as arg); functional factories in lambdas.

#### 2.1.3 Abstract Factory
- **Intent**: Provide interface for creating families of related objects without specifying concrete classes.
- **Motivation**: UI themes (Windows/Mac buttons/menus).
- **Structure**:
  ```
  AbstractFactory --(creates)-- AbstractProductA, AbstractProductB
  |
  ConcreteFactory1 --(creates)-- ProductA1, ProductB1
  ConcreteFactory2 --(creates)-- ProductA2, ProductB2
  ```
- **Participants**: AbstractFactory, ConcreteFactory, AbstractProduct.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Ensures product compatibility | Hard to add new products      |
  | Isolates concrete classes     | Factory explosion             |
- **Implementation** (Python GUI):
  ```python
  class Button: def paint(self): pass
  class WinButton(Button): def paint(self): print("Win Button")
  class GUIFactory: def create_button(self): pass
  class WinFactory(GUIFactory): def create_button(self): return WinButton()
  factory = WinFactory(); button = factory.create_button(); button.paint()
  ```
- **Known Uses**: Java's LookAndFeel, game asset factories.
- **Relations**: Composes Factory Method; used with Bridge.
- **Advanced**: Pluggable factories via config; in microservices for service factories.

#### 2.1.4 Builder
- **Intent**: Separate construction of complex object from its representation.
- **Motivation**: Telescoping constructors for objects with many optional params (e.g., HTTP requests).
- **Structure**:
  ```
  Director --(uses)-- Builder --(builds parts)-- Product
                       |
                  ConcreteBuilder --(assembles)--
  ```
- **Participants**: Builder, ConcreteBuilder, Director, Product.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Immutable objects             | More code than simple ctor    |
  | Step-by-step construction     | Director overkill for simple  |
- **Implementation** (Python Pizza Builder):
  ```python
  class Pizza:
      def __init__(self): self.parts = []
      def add(self, part): self.parts.append(part)
  class PizzaBuilder:
      def __init__(self): self.pizza = Pizza()
      def build_dough(self): self.pizza.add("dough"); return self
      def build_topping(self): self.pizza.add("cheese"); return self
      def get_pizza(self): return self.pizza
  pizza = PizzaBuilder().build_dough().build_topping().get_pizza()
  ```
- **Known Uses**: Java's StringBuilder, Lombok @Builder.
- **Relations**: Like Factory but for complex assembly.
- **Advanced**: Fluent interfaces; director optional in modern use.

#### 2.1.5 Prototype
- **Intent**: Create new objects by cloning existing ones.
- **Motivation**: Expensive creation (e.g., large objects like game characters).
- **Structure**:
  ```
  Prototype --(clones)-- clone(): Prototype
  |
  ConcretePrototype --(implements)--
  Client --(uses)--
  ```
- **Participants**: Prototype (with clone()), ConcretePrototype.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Speeds creation               | Shallow copy issues (references) |
  | Hides creation complexity     | Language support varies       |
- **Implementation** (Python):
  ```python
  import copy
  class Sheep:
      def __init__(self, name): self.name = name
      def clone(self): return copy.deepcopy(self)
  original = Sheep("Dolly"); clone = original.clone(); clone.name = "Clone"
  ```
- **Known Uses**: Java's clone(), document templates.
- **Relations**: Used in Factory for instance reuse.
- **Advanced**: Prototype managers (registry of prototypes); deep vs. shallow copy.

### 2.2 Structural Patterns
Concerned with class/object composition for larger structures.

#### 2.2.1 Adapter
- **Intent**: Convert interface of a class into another expected by clients.
- **Motivation**: Integrate legacy code (e.g., old XML parser with new JSON API).
- **Structure**:
  ```
  Target --(expects)-- request()
  |
  Adapter --(implements)-- Target, --(uses)-- Adaptee --(provides)-- specificRequest()
  ```
- **Participants**: Target, Client, Adaptee, Adapter (class or object).
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Reuse existing code           | Increases indirection         |
  | Transparent to client         | Multiple adapters messy       |
- **Implementation** (Python media player):
  ```python
  class MediaPlayer: def play(self, audio): pass
  class VlcPlayer: def play_vlc(self, file): print(f"Playing VLC {file}")
  class MediaAdapter(MediaPlayer):
      def __init__(self): self.vlc = VlcPlayer()
      def play(self, audio): if "vlc" in audio: self.vlc.play_vlc(audio)
  player = MediaAdapter(); player.play("vlc_file.mkv")
  ```
- **Known Uses**: Java's InputStreamReader (byte to char adapter).
- **Relations**: Like Proxy but for interface mismatch.
- **Advanced**: Two-way adapters; pluggable via strategy.

#### 2.2.2 Bridge
- **Intent**: Decouple abstraction from implementation so both vary independently.
- **Motivation**: Drivers (e.g., shape drawing with multiple renderers: OpenGL/DirectX).
- **Structure**:
  ```
  Abstraction --(has)-- Implementor --(implements)-- operationImp()
  |                                   |
  RefinedAbstraction --(uses)-- ConcreteImplementorA/B
  ```
- **Participants**: Abstraction, RefinedAbstraction, Implementor, ConcreteImplementor.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Extensibility in two dimensions| More classes than subclassing |
  | Hides implementation details  | Complexity overhead           |
- **Implementation** (Python shapes):
  ```python
  class DrawAPI: def draw_circle(self, x, y, r): pass
  class RedCircle(DrawAPI): def draw_circle(self, x, y, r): print(f"Red circle {r}")
  class Shape: def __init__(self, api): self.api = api
  class Circle(Shape): def draw(self): self.api.draw_circle(1,2,3)
  circle = Circle(RedCircle()); circle.draw()
  ```
- **Known Uses**: JDBC drivers, GUI toolkits.
- **Relations**: Avoids class explosion vs. subclassing.
- **Advanced**: Partial implementation; runtime switching.

#### 2.2.3 Composite
- **Intent**: Compose objects into tree structures to represent part-whole hierarchies uniformly.
- **Motivation**: File systems (files and directories).
- **Structure**:
  ```
  Component --(defines)-- operation(), add(child), remove(child)
  |
  Leaf --(implements)-- operation()
  Composite --(implements)-- operation() --(aggregates)-- children: Component*
  ```
- **Participants**: Component, Leaf, Composite.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Uniform treatment             | Over-generalization (leaves get add/remove) |
  | Simplifies client code        | Can make design overly general|
- **Implementation** (Python menu):
  ```python
  class MenuComponent:
      def add(self, c): pass; def print(self): pass
  class MenuItem(MenuComponent):
      def __init__(self, name): self.name = name
      def print(self): print(self.name)
  class Menu(MenuComponent):
      def __init__(self, name): self.name = name; self.children = []
      def add(self, c): self.children.append(c)
      def print(self): print(self.name); [c.print() for c in self.children]
  menu = Menu("Main"); menu.add(MenuItem("Burger")); menu.print()
  ```
- **Known Uses**: Java AWT Components, XML DOM.
- **Relations**: Used with Visitor for traversal.
- **Advanced**: Safe vs. Transparent composites; flyweight for shared leaves.

#### 2.2.4 Decorator
- **Intent**: Attach responsibilities to objects dynamically without subclassing.
- **Motivation**: Add features to classes (e.g., window borders, scrolls).
- **Structure**:
  ```
  Component --(defines)-- operation()
  |
  Decorator --(implements)-- Component --(delegates)-- component: Component
  |
  ConcreteDecoratorA/B --(adds)-- extra behavior
  ```
- **Participants**: Component, ConcreteComponent, Decorator, ConcreteDecorator.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Flexible additions            | Many small objects            |
  | Avoids feature explosion      | Debugging harder (no names)   |
- **Implementation** (Python coffee):
  ```python
  class Beverage: def cost(self): return 0
  class Espresso(Beverage): def cost(self): return 1.99
  class MilkDecorator(Beverage):
      def __init__(self, bev): self.bev = bev
      def cost(self): return self.bev.cost() + 0.5
  coffee = MilkDecorator(Espresso()); print(coffee.cost())  # 2.49
  ```
- **Known Uses**: Java I/O streams (BufferedReader).
- **Relations**: Like Adapter but for enhancement.
- **Advanced**: Stateful decorators; stackable via chains.

#### 2.2.5 Facade
- **Intent**: Provide simplified interface to a complex subsystem.
- **Motivation**: Home theater control (one button for movie mode).
- **Structure**:
  ```
  Subsystem Classes --(collaborate)--
  |
  Facade --(knows)-- subsystem ops, --(provides)-- simplified ops
  Client --(uses)--
  ```
- **Participants**: Facade, Subsystem classes.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Shields complexity            | Limits flexibility if overused|
  | Promotes loose coupling       | Facade may become god class   |
- **Implementation** (Python compiler):
  ```python
  class Scanner: def scan(self): return "tokens"
  class Parser: def parse(self, tokens): return "AST"
  class CodeGenerator: def generate(self, ast): return "code"
  class CompilerFacade:
      def __init__(self): self.scanner = Scanner(); self.parser = Parser(); self.cg = CodeGenerator()
      def compile(self, source): tokens = self.scanner.scan(); ast = self.parser.parse(tokens); return self.cg.generate(ast)
  compiler = CompilerFacade(); code = compiler.compile("int x=5;")
  ```
- **Known Uses**: Java's J2EE Facade, SLF4J logging.
- **Relations**: Like Abstract Factory for subsystems.
- **Advanced**: Nested facades; service layers in web apps.

#### 2.2.6 Flyweight
- **Intent**: Use sharing to support large numbers of fine-grained objects efficiently.
- **Motivation**: Document editors with thousands of characters sharing font/glyph data.
- **Structure**:
  ```
  Flyweight --(intrinsic state)-- operation(extrinsic)
  |
  ConcreteFlyweight --(shares)--
  FlyweightFactory --(manages)-- flyweights: HashMap
  ```
- **Participants**: Flyweight, ConcreteFlyweight, FlyweightFactory, Client (manages extrinsic state).
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Saves memory                  | Extrinsic state complicates   |
  | Runtime efficiency            | Factory centralizes control   |
- **Implementation** (Python glyphs):
  ```python
  class Glyph:
      def __init__(self, char): self.char = char  # Intrinsic
      def display(self, x, y): print(f"{self.char} at ({x},{y})")  # Extrinsic x,y
  class GlyphFactory:
      def __init__(self): self.glyphs = {}
      def get_glyph(self, char): return self.glyphs.setdefault(char, Glyph(char))
  factory = GlyphFactory(); g = factory.get_glyph('A'); g.display(10,20)
  ```
- **Known Uses**: Java String intern(), game sprites.
- **Relations**: Complements Composite for leaves.
- **Advanced**: Thread-safe factories; purging unused flyweights.

#### 2.2.7 Proxy
- **Intent**: Provide surrogate or placeholder for another object to control access.
- **Motivation**: Lazy loading (e.g., image proxy loads on demand), remote access.
- **Structure**:
  ```
  Subject --(interface)-- request()
  |
  Proxy --(implements)-- Subject --(controls access to)-- RealSubject --(provides)-- request()
  ```
- **Types**: Virtual (lazy), Protection (access control), Remote (RMI), Smart (caching).
- **Participants**: Subject, Proxy, RealSubject.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Adds functionality transparently | Indirection overhead          |
  | Centralizes control           | Remote proxies need marshalling|
- **Implementation** (Python image proxy):
  ```python
  class Image: def display(self): pass
  class RealImage(Image):
      def __init__(self, file): self.file = file; self.load_from_disk()
      def load_from_disk(self): print(f"Loading {self.file}")
      def display(self): print(f"Displaying {self.file}")
  class ProxyImage(Image):
      def __init__(self, file): self.file = file; self.real = None
      def display(self):
          if self.real is None: self.real = RealImage(self.file)
          self.real.display()
  image = ProxyImage("test.jpg"); image.display()  # Lazy load
  ```
- **Known Uses**: Java dynamic proxies (AOP), Spring transactions.
- **Relations**: Like Decorator but for control, not extension.
- **Advanced**: CGLIB proxies; firewall proxies.

### 2.3 Behavioral Patterns
Focus on algorithms, responsibilities, and communication.

#### 2.3.1 Chain of Responsibility
- **Intent**: Avoid coupling sender/receiver by passing request along chain of handlers.
- **Motivation**: Event bubbling in GUIs (mouse click handled by window/panel/button).
- **Structure**:
  ```
  Handler --(has)-- successor: Handler
  |           --(defines)-- handleRequest()
  ConcreteHandler1/2 --(implements)-- if condition, handle else pass to successor
  ```
- **Participants**: Handler, ConcreteHandler, Client.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Decouples sender/receiver     | Request may go unhandled      |
  | Dynamic chain ordering        | Debugging chains hard         |
- **Implementation** (Python logger):
  ```python
  class Logger:
      def __init__(self, level): self.level = level; self.next = None
      def set_next(self, logger): self.next = logger; return logger
      def log(self, msg, lvl):
          if lvl >= self.level: print(f"{self.level}: {msg}")
          if self.next: self.next.log(msg, lvl)
  ERROR = Logger(1); DEBUG = Logger(2); ERROR.set_next(DEBUG)
  ERROR.log("Error msg", 1)  # ERROR: Error msg; DEBUG: Error msg
  ```
- **Known Uses**: Servlet filters, middleware in Express.js.
- **Relations**: Like Composite for linear hierarchy.
- **Advanced**: Async chains; priority-based.

#### 2.3.2 Command
- **Intent**: Encapsulate request as object, parameterizing clients with queues, logs, etc.
- **Motivation**: Undo/redo (e.g., text editor commands).
- **Structure**:
  ```
  Command --(interface)-- execute()
  |
  ConcreteCommand --(implements)-- execute() --(binds)-- receiver: Receiver
  Invoker --(stores/queues)-- commands
  Receiver --(knows)-- action()
  ```
- **Participants**: Command, ConcreteCommand, Invoker, Receiver, Client.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Undoable, queueable           | Many classes                  |
  | Extensible operations         | Decouples but adds objects    |
- **Implementation** (Python remote control):
  ```python
  class Light: def on(self): print("Light on")
  class LightOnCommand:
      def __init__(self, light): self.light = light
      def execute(self): self.light.on()
  class RemoteControl:
      def __init__(self): self.commands = {}
      def set_command(self, slot, cmd): self.commands[slot] = cmd
      def press_button(self, slot): self.commands[slot].execute()
  rc = RemoteControl(); light = Light(); rc.set_command(0, LightOnCommand(light)); rc.press_button(0)
  ```
- **Known Uses**: Java's Runnable, GUI actions.
- **Relations**: With Memento for undo.
- **Advanced**: Macro commands (composite); transactions.

#### 2.3.3 Interpreter
- **Intent**: Define grammar for simple language and interpreter for sentences.
- **Motivation**: SQL parsers or expression evaluators.
- **Structure**:
  ```
  AbstractExpression --(defines)-- interpret(context)
  |
  Terminal/NonTerminal --(implements)-- interpret() --(uses)-- expressions
  Client --(builds AST)--
  ```
- **Participants**: AbstractExpression, TerminalExpression, NonTerminalExpression, Context.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Easy for simple grammars      | Inefficient for complex (use parser gen) |
  | Extensible grammar            | Class explosion for rules     |
- **Implementation** (Python expression):
  ```python
  class Number: def __init__(self, val): self.val = val
  class Plus: def __init__(self, left, right): self.left = left; self.right = right
  class Expression:
      def interpret(self): return self.val if isinstance(self, Number) else self.left.interpret() + self.right.interpret()
  expr = Plus(Number(5), Number(3)); print(expr.interpret())  # 8
  ```
- **Known Uses**: Regular expressions, rule engines.
- **Relations**: With Composite for AST.
- **Advanced**: LL/Recursive Descent parsers; bytecode for efficiency.

#### 2.3.4 Iterator
- **Intent**: Provide sequential access to aggregate elements without exposing representation.
- **Motivation**: Traverse collections uniformly (list/tree/stack).
- **Structure**:
  ```
  Iterator --(interface)-- next(), hasNext()
  Aggregate --(creates)-- iterator(): Iterator
  |
  ConcreteIterator --(tracks)-- current, aggregate
  ```
- **Participants**: Iterator, ConcreteIterator, Aggregate, ConcreteAggregate.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Encapsulates traversal        | Language support (built-in) reduces need |
  | Multiple traversals           | Extra classes                 |
- **Implementation** (Python tree iterator):
  ```python
  class Node:
      def __init__(self, val): self.val = val; self.children = []
      def add(self, child): self.children.append(child)
  class TreeIterator:
      def __init__(self, root): self.stack = [root]
      def has_next(self): return bool(self.stack)
      def next(self): node = self.stack.pop(); self.stack.extend(reversed(node.children)); return node.val
  root = Node(1); root.add(Node(2)); it = TreeIterator(root); while it.has_next(): print(it.next())
  ```
- **Known Uses**: Java's Iterable, Python's iter().
- **Relations**: With Composite for traversal.
- **Advanced**: External vs. Internal iterators; bidirectional.

#### 2.3.5 Mediator
- **Intent**: Define object that encapsulates how set of objects interact, promoting loose coupling.
- **Motivation**: Air traffic control (planes communicate via tower).
- **Structure**:
  ```
  Mediator --(defines)-- notify(sender, event)
  Colleague --(knows)-- mediator --(communicates via)--
  |
  ConcreteMediator --(coordinates)-- colleagues
  ```
- **Participants**: Mediator, ConcreteMediator, Colleague classes.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Centralizes control           | Mediator becomes complex      |
  | Reduces dependencies          | Single point of failure       |
- **Implementation** (Python chat room):
  ```python
  class Mediator:
      def notify(self, sender, msg): pass
  class User:
      def __init__(self, name, mediator): self.name = name; self.mediator = mediator
      def send(self, msg): self.mediator.notify(self, msg)
  class ChatMediator(Mediator):
      def __init__(self): self.users = []
      def add_user(self, user): self.users.append(user)
      def notify(self, sender, msg):
          for user in self.users: if user != sender: print(f"{user.name}: {msg}")
  cm = ChatMediator(); u1 = User("Alice", cm); cm.add_user(u1); u1.send("Hi!")
  ```
- **Known Uses**: MVC controllers, event buses.
- **Relations**: Opposite of Observer (central vs. broadcast).
- **Advanced**: Mediated pub-sub; transparent mediators.

#### 2.3.6 Memento
- **Intent**: Capture/restore object's internal state without violating encapsulation.
- **Motivation**: Game saves or editor undo.
- **Structure**:
  ```
  Originator --(creates)-- Memento --(stores)-- state
               --(uses)-- setMemento(m), getState()
  Caretaker --(stores)-- mementos
  ```
- **Participants**: Originator, Memento, Caretaker.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Preserves encapsulation       | Memory overhead for states    |
  | Simple undo                  | Narrow/wide interfaces        |
- **Implementation** (Python text editor):
  ```python
  class Memento:
      def __init__(self, state): self.state = state
  class Editor:
      def __init__(self, text=""): self.text = text
      def save(self): return Memento(self.text)
      def restore(self, m): self.text = m.state
      def write(self, s): self.text += s
  class History:
      def __init__(self): self.mementos = []
      def push(self, m): self.mementos.append(m)
      def pop(self): return self.mementos.pop()
  editor = Editor(); history = History(); editor.write("Hello"); history.push(editor.save())
  editor.write(" World"); editor.restore(history.pop()); print(editor.text)  # Hello
  ```
- **Known Uses**: Java's javax.swing.undo, Git commits.
- **Relations**: With Command for undo stack.
- **Advanced**: Token mementos (externalize state); versioning.

#### 2.3.7 Observer
- **Intent**: Define one-to-many dependency where objects notify dependents of state changes.
- **Motivation**: MVC (model notifies views).
- **Structure**:
  ```
  Subject --(maintains)-- observers, --(methods)-- attach(o), notify()
  |
  ConcreteSubject --(state)-- notify() { for o in observers: o.update() }
  Observer --(interface)-- update()
  ConcreteObserver --(implements)-- update() --(uses)-- subject
  ```
- **Participants**: Subject, ConcreteSubject, Observer, ConcreteObserver.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Loose coupling                | Unexpected updates            |
  | Supports broadcast            | Memory leaks if not detached  |
- **Implementation** (Python stock):
  ```python
  class Observer: def update(self, data): pass
  class Stock:
      def __init__(self): self.observers = []; self.price = 0
      def attach(self, o): self.observers.append(o)
      def notify(self): [o.update(self.price) for o in self.observers]
      def set_price(self, p): self.price = p; self.notify()
  class Investor(Observer):
      def __init__(self, name): self.name = name
      def update(self, price): print(f"{self.name}: Price is {price}")
  stock = Stock(); inv = Investor("Alice"); stock.attach(inv); stock.set_price(100)
  ```
- **Known Uses**: Java's Observable, RxJS observables.
- **Relations**: Publish-Subscribe variant (with broker).
- **Advanced**: Pull vs. Push models; reactive extensions.

#### 2.3.8 State
- **Intent**: Allow object to alter behavior when internal state changes, appearing as class change.
- **Motivation**: TCP connection states (listen/syn/established).
- **Structure**:
  ```
  Context --(has)-- state: State --(delegates)-- handle()
  |
  ConcreteStateA/B --(implements)-- handle() --(transitions)-- context.state = newState
  ```
- **Participants**: Context, State, ConcreteState.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Localizes state behavior      | Many states = many classes    |
  | Flat state space              | Initial setup                 |
- **Implementation** (Python vending machine):
  ```python
  class State: def insert_money(self, ctx): pass
  class NoQuarterState(State):
      def insert_money(self, ctx): print("Quarter inserted"); ctx.state = ctx.has_quarter_state
  class VendingMachine:
      def __init__(self): self.state = NoQuarterState()
      def insert_money(self): self.state.insert_money(self)
  vm = VendingMachine(); vm.insert_money()
  ```
- **Known Uses**: State machines in UML, workflow engines.
- **Relations**: Like Strategy but state-driven.
- **Advanced**: Hierarchical states; orthogonal regions.

#### 2.3.9 Strategy
- **Intent**: Define family of algorithms, encapsulate each, make interchangeable.
- **Motivation**: Sorting (quicksort/mergesort based on data).
- **Structure**:
  ```
  Context --(has)-- strategy: Strategy --(delegates)-- algorithmInterface()
  |
  ConcreteStrategyA/B --(implements)-- algorithmInterface()
  ```
- **Participants**: Strategy, ConcreteStrategy, Context.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Families of interchangeable   | Clients must know strategies  |
  | Avoids conditionals           | Overhead for simple cases     |
- **Implementation** (Python payment):
  ```python
  class PaymentStrategy: def pay(self, amount): pass
  class CreditCard(PaymentStrategy): def pay(self, amount): print(f"Paid {amount} by CC")
  class ShoppingCart:
      def __init__(self): self.strategy = None
      def set_strategy(self, s): self.strategy = s
      def checkout(self, amount): self.strategy.pay(amount)
  cart = ShoppingCart(); cart.set_strategy(CreditCard()); cart.checkout(100)
  ```
- **Known Uses**: Java's Comparator, layout managers.
- **Relations**: Like State but algorithm-focused.
- **Advanced**: Runtime selection via factory; lambda strategies in FP.

#### 2.3.10 Template Method
- **Intent**: Define algorithm skeleton in superclass, let subclasses override steps.
- **Motivation**: Framework hooks (e.g., test setup/teardown).
- **Structure**:
  ```
  AbstractClass --(defines)-- templateMethod() { step1(); step2(); step3(); }
               --(abstract)-- primitive ops
  |
  ConcreteClass --(implements)-- primitive ops
  ```
- **Participants**: AbstractClass, ConcreteClass.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Reuses invariant parts        | Hard to modify structure      |
  | Inversion of control          | Fragile base class            |
- **Implementation** (Python game):
  ```python
  class Game:
      def play(self): self.initialize(); self.start_play(); self.end_play()
      def initialize(self): raise NotImplementedError
      def start_play(self): raise NotImplementedError
      def end_play(self): raise NotImplementedError
  class Chess(Game):
      def initialize(self): print("Chess board")
      def start_play(self): print("Start chess")
      def end_play(self): print("End chess")
  chess = Chess(); chess.play()
  ```
- **Known Uses**: Java's AbstractList, JUnit @Before.
- **Relations**: Hollywood Principle ("Don't call us, we'll call you").
- **Advanced**: Hook methods (optional overrides); hooks in frameworks.

#### 2.3.11 Visitor
- **Intent**: Represent operation on elements of object structure without changing classes.
- **Motivation**: Add ops to composite without subclassing (e.g., report generators on AST).
- **Structure**:
  ```
  Element --(defines)-- accept(visitor: Visitor)
  |
  ConcreteElementA/B --(implements)-- accept(v) { v.visit(this) }
  Visitor --(defines)-- visitElementA/B()
  |
  ConcreteVisitor --(implements)-- visit() { do op on element }
  ObjectStructure --(iterates)-- elements, --(applies)-- visitor
  ```
- **Participants**: Visitor, ConcreteVisitor, Element, ConcreteElement, ObjectStructure.
- **Consequences**:
  | Pros                          | Cons                          |
  |-------------------------------|-------------------------------|
  | Adds ops without subclassing  | Structure changes break visitors |
  | Accumulates state             | Violates encapsulation        |
- **Implementation** (Python shapes):
  ```python
  class Shape: def accept(self, v): v.visit_shape(self)
  class Circle(Shape): def accept(self, v): v.visit_circle(self)
  class AreaVisitor:
      def visit_shape(self, s): print("Unknown")
      def visit_circle(self, c): print(f"Area: {3.14 * c.radius**2}")
  shapes = [Circle()]; for s in shapes: s.accept(AreaVisitor())
  ```
- **Known Uses**: Compilers (type checkers), Eclipse plugins.
- **Relations**: Double dispatch; with Composite.
- **Advanced**: Acyclic visitors (for graphs); functional folds.

### 2.4 Additional Considerations for Design Patterns
- **Anti-Patterns**: Overuse leads to "patternitis" (unnecessary complexity). E.g., Singleton as global variable abuse.
- **Modern Adaptations**: Functional patterns (e.g., monads as Strategy); microservices (Circuit Breaker as Proxy); concurrency (Thread-Safe Singleton).
- **Selection Guide**:
  | Category     | When to Use                          | Example Domain |
  |--------------|--------------------------------------|---------------|
  | Creational  | Flexible instantiation               | Frameworks   |
  | Structural  | Composition over inheritance         | UI/Libraries |
  | Behavioral  | Communication/Algorithms             | Apps/Systems |
- **Critiques**: GoF patterns are OO-centric; Enterprise patterns (POSA) for distributed; RESTful patterns for web.
- **Learning Tips**: Implement in your language; refactor code to patterns; study source (e.g., Spring uses many).

This guide clocks in at super lengthy—over 5,000 words! If you need expansions, code repos, or quizzes, let me know. Happy coding!
