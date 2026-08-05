# Software design patterns

Design patterns, popularized by the Gang of Four (GoF) in *Design Patterns: Elements of Reusable Object-Oriented Software* (1994), are proven, reusable solutions to recurring software design problems. They promote maintainability, scalability, and reusability by giving engineers a shared vocabulary for structures that keep coming up.

## TL;DR
- **Creational** patterns (Singleton, Factory Method, Abstract Factory, Builder, Prototype) — control how objects get created.
- **Structural** patterns (Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy) — control how classes/objects compose into larger structures.
- **Behavioral** patterns (Chain of Responsibility, Command, Interpreter, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor) — control communication and responsibility between objects.
- Backend/distributed-systems work leans on a few patterns not in the original GoF catalog: **Circuit Breaker**, **Retry with backoff**, and the GoF **Adapter** shows up constantly for wrapping external APIs.
- Overuse of any pattern is itself an anti-pattern ("patternitis") — a pattern should solve a real problem you have, not be applied because it's available.

## Creational patterns

Focus on flexible object creation, decoupling instantiation from usage.

### Singleton
**Intent**: ensure a class has only one instance and provide global access to it.
**Motivation**: for resources like loggers or caches where multiple instances would waste resources or cause inconsistency.

```
+Singleton
-instance: Singleton
+getInstance(): Singleton  // Thread-safe
+operation()
```

| Pros | Cons |
|---|---|
| Controlled access | Global state is hard to test |
| Lazy initialization | Violates Single Responsibility Principle |

```python
class Singleton:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
s = Singleton()  # Same instance always
```

**Known uses**: Java's `Runtime`, database connection pools. **Relations**: similar to Monostate (shared state, but multiple instances allowed). **Advanced**: the Bill Pugh singleton (inner static/enum class) is the standard thread-safe idiom on the JVM without needing explicit locking; overused, Singleton becomes an anti-pattern — a disguised global variable that makes testing and reasoning about state harder.

### Factory Method
**Intent**: define an interface for creating an object but let subclasses decide which class to instantiate.
**Motivation**: a framework defines the creation logic without binding to concrete classes (e.g., GUI toolkits).

```
Creator --(defines)-- abstract factoryMethod(): Product
          |
ConcreteCreator --(implements)-- factoryMethod(): ConcreteProduct
Product --(uses)--
```

| Pros | Cons |
|---|---|
| Promotes loose coupling | Subclasses proliferate |
| Supports extension | Code duplication if variants are few |

```python
class Shape: def draw(self): pass
class Circle(Shape): def draw(self): print("Circle")
class ShapeFactory:
    def create_shape(self, type):
        if type == "circle": return Circle()
factory = ShapeFactory(); shape = factory.create_shape("circle"); shape.draw()
```

**Known uses**: Java's `Calendar.getInstance()`, document creators in apps. **Relations**: a variant of Abstract Factory; often paired with Template Method for algorithm skeletons. **Advanced**: parametric factory methods (pass the type as an argument) and functional factories (a lambda instead of a class) are common lightweight variants.

### Abstract Factory
**Intent**: provide an interface for creating families of related objects without specifying their concrete classes.
**Motivation**: UI themes (Windows vs. Mac buttons/menus need to match consistently).

```
AbstractFactory --(creates)-- AbstractProductA, AbstractProductB
|
ConcreteFactory1 --(creates)-- ProductA1, ProductB1
ConcreteFactory2 --(creates)-- ProductA2, ProductB2
```

| Pros | Cons |
|---|---|
| Ensures product compatibility within a family | Hard to add new product types (requires touching every factory) |
| Isolates concrete classes from the client | Risk of factory explosion |

```python
class Button: def paint(self): pass
class WinButton(Button): def paint(self): print("Win Button")
class GUIFactory: def create_button(self): pass
class WinFactory(GUIFactory): def create_button(self): return WinButton()
factory = WinFactory(); button = factory.create_button(); button.paint()
```

**Known uses**: Java's `LookAndFeel`, game asset factories. **Relations**: composed of multiple Factory Methods; often used with Bridge. **Advanced**: pluggable factories selected via config are common in service-factory setups in microservices.

### Builder
**Intent**: separate the construction of a complex object from its representation, so the same construction process can create different representations.
**Motivation**: avoids telescoping constructors for objects with many optional parameters (e.g., HTTP requests).

```
Director --(uses)-- Builder --(builds parts)-- Product
                     |
                ConcreteBuilder --(assembles)--
```

| Pros | Cons |
|---|---|
| Enables immutable objects | More code than a simple constructor |
| Step-by-step construction | A separate Director is overkill for simple cases |

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

**Known uses**: Java's `StringBuilder`, Lombok's `@Builder`. **Relations**: like Factory but for assembling something complex piece by piece. **Advanced**: fluent interfaces (method chaining returning `self`) are the modern default; the separate Director object is often skipped entirely in practice.

### Prototype
**Intent**: create new objects by cloning an existing instance rather than instantiating from scratch.
**Motivation**: useful when creation is expensive (e.g., large objects like game characters with lots of precomputed state).

```
Prototype --(clones)-- clone(): Prototype
|
ConcretePrototype --(implements)--
Client --(uses)--
```

| Pros | Cons |
|---|---|
| Speeds up creation | Shallow-copy pitfalls (shared references) |
| Hides creation complexity from the client | Language support for cloning varies |

```python
import copy
class Sheep:
    def __init__(self, name): self.name = name
    def clone(self): return copy.deepcopy(self)
original = Sheep("Dolly"); clone = original.clone(); clone.name = "Clone"
```

**Known uses**: Java's `clone()`, document templates. **Relations**: often used inside a Factory for instance reuse. **Advanced**: a prototype manager/registry keeps a set of pre-configured prototypes to clone from; always be deliberate about deep vs. shallow copy — shallow copies of mutable nested objects are a classic bug source.

## Structural patterns

Concerned with how classes and objects compose into larger structures.

### Adapter
**Intent**: convert the interface of a class into another interface clients expect.
**Motivation**: integrating legacy code (e.g., an old XML parser behind a new JSON-shaped API).

```
Target --(expects)-- request()
|
Adapter --(implements)-- Target, --(uses)-- Adaptee --(provides)-- specificRequest()
```

| Pros | Cons |
|---|---|
| Reuses existing code without modification | Adds a layer of indirection |
| Transparent to the client | Many adapters stacked together gets messy |

```python
class MediaPlayer: def play(self, audio): pass
class VlcPlayer: def play_vlc(self, file): print(f"Playing VLC {file}")
class MediaAdapter(MediaPlayer):
    def __init__(self): self.vlc = VlcPlayer()
    def play(self, audio):
        if "vlc" in audio: self.vlc.play_vlc(audio)
player = MediaAdapter(); player.play("vlc_file.mkv")
```

**Known uses**: Java's `InputStreamReader` (byte stream to char stream adapter). **Relations**: similar to Proxy, but Proxy preserves the same interface — Adapter exists specifically to bridge a mismatch. **Advanced**: two-way adapters that implement both interfaces; often combined with Strategy to make the adaptation logic itself pluggable.

**This is the single most common pattern in backend integration work** — wrapping a third-party SDK, a legacy internal service, or a different cloud provider's API behind your own consistent interface. See the "Adapter for backend integrations" example below.

### Bridge
**Intent**: decouple an abstraction from its implementation so the two can vary independently.
**Motivation**: e.g., shape-drawing code that needs to work with multiple rendering backends (OpenGL, DirectX) without an explosion of subclasses for every combination.

```
Abstraction --(has)-- Implementor --(implements)-- operationImp()
|                                   |
RefinedAbstraction --(uses)-- ConcreteImplementorA/B
```

| Pros | Cons |
|---|---|
| Extensibility along two independent dimensions | More classes than plain subclassing |
| Hides implementation details from the abstraction | Added indirection/complexity |

```python
class DrawAPI: def draw_circle(self, x, y, r): pass
class RedCircle(DrawAPI): def draw_circle(self, x, y, r): print(f"Red circle {r}")
class Shape: def __init__(self, api): self.api = api
class Circle(Shape): def draw(self): self.api.draw_circle(1, 2, 3)
circle = Circle(RedCircle()); circle.draw()
```

**Known uses**: JDBC drivers, GUI toolkits. **Relations**: exists specifically to avoid the class explosion that comes from subclassing across two independent dimensions. **Advanced**: implementations can be partial (some default behavior in the base) and switched at runtime.

### Composite
**Intent**: compose objects into tree structures to represent part-whole hierarchies, so clients treat individual objects and compositions of objects uniformly.
**Motivation**: file systems (files and directories handled through the same interface).

```
Component --(defines)-- operation(), add(child), remove(child)
|
Leaf --(implements)-- operation()
Composite --(implements)-- operation() --(aggregates)-- children: Component*
```

| Pros | Cons |
|---|---|
| Uniform treatment of leaves and composites | Over-generalization — leaves inherit `add`/`remove` they can't meaningfully support |
| Simplifies client code | Can make the design overly general |

```python
class MenuComponent:
    def add(self, c): pass
    def print(self): pass
class MenuItem(MenuComponent):
    def __init__(self, name): self.name = name
    def print(self): print(self.name)
class Menu(MenuComponent):
    def __init__(self, name): self.name = name; self.children = []
    def add(self, c): self.children.append(c)
    def print(self): print(self.name); [c.print() for c in self.children]
menu = Menu("Main"); menu.add(MenuItem("Burger")); menu.print()
```

**Known uses**: Java AWT components, XML DOM. **Relations**: often paired with Visitor for traversal operations. **Advanced**: "safe" composites (leaf and composite have different interfaces) vs. "transparent" composites (same interface, less type-safe but simpler client code); Flyweight can be layered in for shared leaves.

### Decorator
**Intent**: attach additional responsibilities to an object dynamically, without subclassing.
**Motivation**: adding features to objects at runtime (e.g., window borders, scrollbars).

```
Component --(defines)-- operation()
|
Decorator --(implements)-- Component --(delegates)-- component: Component
|
ConcreteDecoratorA/B --(adds)-- extra behavior
```

| Pros | Cons |
|---|---|
| Flexible, composable additions | Can result in many small wrapper objects |
| Avoids a combinatorial explosion of subclasses | Debugging is harder — the actual class name doesn't tell you the full behavior |

```python
class Beverage: def cost(self): return 0
class Espresso(Beverage): def cost(self): return 1.99
class MilkDecorator(Beverage):
    def __init__(self, bev): self.bev = bev
    def cost(self): return self.bev.cost() + 0.5
coffee = MilkDecorator(Espresso()); print(coffee.cost())  # 2.49
```

**Known uses**: Java I/O streams (`BufferedReader` wrapping a `Reader`). **Relations**: similar structure to Adapter, but Decorator preserves the interface and *adds* behavior rather than converting it. **Advanced**: stateful decorators; stacking multiple decorators to compose behavior (common in middleware chains).

### Facade
**Intent**: provide a simplified, unified interface to a complex subsystem.
**Motivation**: a home theater "movie mode" button that internally controls the projector, lights, and sound system.

```
Subsystem Classes --(collaborate)--
|
Facade --(knows)-- subsystem ops, --(provides)-- simplified ops
Client --(uses)--
```

| Pros | Cons |
|---|---|
| Shields clients from subsystem complexity | Limits flexibility if the facade is the only entry point and later needs bypassing |
| Promotes loose coupling | Risk of becoming a god class if it accumulates too much logic |

```python
class Scanner: def scan(self): return "tokens"
class Parser: def parse(self, tokens): return "AST"
class CodeGenerator: def generate(self, ast): return "code"
class CompilerFacade:
    def __init__(self):
        self.scanner = Scanner(); self.parser = Parser(); self.cg = CodeGenerator()
    def compile(self, source):
        tokens = self.scanner.scan(); ast = self.parser.parse(tokens); return self.cg.generate(ast)
compiler = CompilerFacade(); code = compiler.compile("int x=5;")
```

**Known uses**: Java's J2EE Facade pattern, SLF4J's logging facade. **Relations**: similar in spirit to Abstract Factory but focused on simplifying access to a subsystem rather than object creation. **Advanced**: nested facades; "service layers" in web apps are essentially Facade applied to a group of lower-level services/repositories.

### Flyweight
**Intent**: use sharing to support large numbers of fine-grained objects efficiently.
**Motivation**: a document editor with thousands of characters, where font/glyph rendering data is shared rather than duplicated per character.

```
Flyweight --(intrinsic state)-- operation(extrinsic)
|
ConcreteFlyweight --(shares)--
FlyweightFactory --(manages)-- flyweights: HashMap
```

| Pros | Cons |
|---|---|
| Saves memory by sharing common state | Extrinsic state (passed in per-call) complicates the API |
| Improves runtime efficiency | The factory centralizes control, becoming a coordination point |

```python
class Glyph:
    def __init__(self, char): self.char = char  # Intrinsic
    def display(self, x, y): print(f"{self.char} at ({x},{y})")  # Extrinsic x,y
class GlyphFactory:
    def __init__(self): self.glyphs = {}
    def get_glyph(self, char): return self.glyphs.setdefault(char, Glyph(char))
factory = GlyphFactory(); g = factory.get_glyph('A'); g.display(10, 20)
```

**Known uses**: Java's `String.intern()`, game sprite systems. **Relations**: complements Composite for shared leaf nodes. **Advanced**: thread-safe factories for concurrent access; purging unused flyweights to reclaim memory over time.

### Proxy
**Intent**: provide a surrogate or placeholder for another object to control access to it.
**Motivation**: lazy loading (e.g., an image proxy that loads from disk only on first display), or remote access.

```
Subject --(interface)-- request()
|
Proxy --(implements)-- Subject --(controls access to)-- RealSubject --(provides)-- request()
```

**Types**: Virtual (lazy loading), Protection (access control), Remote (RMI-style), Smart (adds caching or reference counting).

| Pros | Cons |
|---|---|
| Adds functionality transparently to the client | Indirection adds overhead |
| Centralizes access control | Remote proxies need marshalling/serialization logic |

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

**Known uses**: Java dynamic proxies (used for AOP), Spring's transaction proxies. **Relations**: structurally similar to Decorator, but Proxy is about controlling access, not adding behavior. **Advanced**: CGLIB-based proxies (subclassing instead of interface implementation); firewall proxies as a real-world Protection Proxy example.

## Behavioral patterns

Focus on algorithms, responsibilities, and communication between objects.

### Chain of Responsibility
**Intent**: avoid coupling the sender of a request to its receiver by giving multiple objects a chance to handle it, passed along a chain.
**Motivation**: event bubbling in GUIs (a mouse click handled by the button, then panel, then window if unhandled).

```
Handler --(has)-- successor: Handler
|           --(defines)-- handleRequest()
ConcreteHandler1/2 --(implements)-- if condition, handle else pass to successor
```

| Pros | Cons |
|---|---|
| Decouples sender from receiver | A request can go unhandled if the chain is misconfigured |
| Chain order can be built dynamically | Debugging a long chain is harder than a direct call |

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

**Known uses**: Servlet filters, Express.js middleware chains. **Relations**: structurally like Composite but arranged linearly instead of as a tree. **Advanced**: async chains; priority-based ordering of handlers.

### Command
**Intent**: encapsulate a request as an object, letting you parameterize clients with queues, logs, and undoable operations.
**Motivation**: undo/redo functionality (e.g., a text editor's command history).

```
Command --(interface)-- execute()
|
ConcreteCommand --(implements)-- execute() --(binds)-- receiver: Receiver
Invoker --(stores/queues)-- commands
Receiver --(knows)-- action()
```

| Pros | Cons |
|---|---|
| Supports undo/redo, queueing, logging | Introduces many small classes |
| Decouples invoker from receiver | The extra indirection is unnecessary for trivial operations |

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

**Known uses**: Java's `Runnable`, GUI action objects. **Relations**: commonly paired with Memento to implement undo stacks. **Advanced**: macro commands (a composite of commands); using Command objects as the basis for transactional/queued execution (e.g., a job queue).

### Interpreter
**Intent**: given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.
**Motivation**: small DSLs, SQL-like query parsers, expression evaluators.

```
AbstractExpression --(defines)-- interpret(context)
|
Terminal/NonTerminal --(implements)-- interpret() --(uses)-- expressions
Client --(builds AST)--
```

| Pros | Cons |
|---|---|
| Easy to implement for simple grammars | Inefficient for complex grammars (use a real parser generator instead) |
| Grammar is extensible | Class explosion — one class per grammar rule |

```python
class Number: def __init__(self, val): self.val = val
class Plus: def __init__(self, left, right): self.left = left; self.right = right
class Expression:
    def interpret(self):
        return self.val if isinstance(self, Number) else self.left.interpret() + self.right.interpret()
expr = Plus(Number(5), Number(3)); print(expr.interpret())  # 8
```

**Known uses**: regular expression engines, business rule engines. **Relations**: builds on Composite to represent the abstract syntax tree. **Advanced**: for anything beyond toy grammars, LL/recursive-descent parsers or a bytecode VM outperform the naive tree-walking interpreter shown above.

### Iterator
**Intent**: provide a way to access elements of an aggregate object sequentially without exposing its underlying representation.
**Motivation**: traversing collections uniformly regardless of whether the backing structure is a list, tree, or stack.

```
Iterator --(interface)-- next(), hasNext()
Aggregate --(creates)-- iterator(): Iterator
|
ConcreteIterator --(tracks)-- current, aggregate
```

| Pros | Cons |
|---|---|
| Encapsulates traversal logic | Most modern languages have this built in, reducing the need to implement it manually |
| Supports multiple simultaneous traversals | Extra classes for a custom implementation |

```python
class Node:
    def __init__(self, val): self.val = val; self.children = []
    def add(self, child): self.children.append(child)
class TreeIterator:
    def __init__(self, root): self.stack = [root]
    def has_next(self): return bool(self.stack)
    def next(self):
        node = self.stack.pop(); self.stack.extend(reversed(node.children)); return node.val
root = Node(1); root.add(Node(2))
it = TreeIterator(root)
while it.has_next(): print(it.next())
```

**Known uses**: Java's `Iterable`, Python's `iter()`. **Relations**: often paired with Composite for tree traversal. **Advanced**: external iterators (client drives the loop, as above) vs. internal iterators (the aggregate drives, calling back into client code, e.g. `forEach`); bidirectional iterators for reverse traversal.

### Mediator
**Intent**: define an object that encapsulates how a set of objects interact, promoting loose coupling by keeping objects from referring to each other directly.
**Motivation**: air traffic control — planes don't talk to each other directly, they communicate through the tower.

```
Mediator --(defines)-- notify(sender, event)
Colleague --(knows)-- mediator --(communicates via)--
|
ConcreteMediator --(coordinates)-- colleagues
```

| Pros | Cons |
|---|---|
| Centralizes control logic | The mediator itself can become complex |
| Reduces pairwise dependencies between objects | Becomes a single point of failure/bottleneck |

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
        for user in self.users:
            if user != sender: print(f"{user.name}: {msg}")
cm = ChatMediator(); u1 = User("Alice", cm); cm.add_user(u1); u1.send("Hi!")
```

**Known uses**: MVC controllers, event buses. **Relations**: philosophically the opposite of Observer — Mediator centralizes coordination, Observer broadcasts. **Advanced**: mediated pub-sub hybrids; "transparent" mediators that colleagues aren't explicitly aware of.

### Memento
**Intent**: capture and externalize an object's internal state so it can be restored later, without violating encapsulation.
**Motivation**: game save states, editor undo functionality.

```
Originator --(creates)-- Memento --(stores)-- state
             --(uses)-- setMemento(m), getState()
Caretaker --(stores)-- mementos
```

| Pros | Cons |
|---|---|
| Preserves encapsulation of the originator's internals | Memory overhead from storing many states |
| Simple to implement undo with | Balancing narrow (caretaker) vs. wide (originator) interfaces takes care |

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
editor = Editor(); history = History()
editor.write("Hello"); history.push(editor.save())
editor.write(" World"); editor.restore(history.pop()); print(editor.text)  # Hello
```

**Known uses**: `javax.swing.undo`, Git commits (each commit is effectively a memento of repository state). **Relations**: often used together with Command to build an undo stack. **Advanced**: "token" mementos that externalize state to storage; versioning systems generalize this pattern.

### Observer
**Intent**: define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically.
**Motivation**: MVC — the model notifies views when data changes.

```
Subject --(maintains)-- observers, --(methods)-- attach(o), notify()
|
ConcreteSubject --(state)-- notify() { for o in observers: o.update() }
Observer --(interface)-- update()
ConcreteObserver --(implements)-- update() --(uses)-- subject
```

| Pros | Cons |
|---|---|
| Loose coupling between subject and observers | Update ordering/side effects can be unexpected |
| Naturally supports broadcast to many listeners | Memory leaks if observers aren't detached (dangling references) |

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

**Known uses**: Java's `Observable`, RxJS observables. **Relations**: Publish-Subscribe is essentially Observer with a broker/message bus in between (see `system-design/websockets-and-realtime.md` for pub-sub fanout in a real distributed system). **Advanced**: push model (subject sends the new state directly) vs. pull model (subject just notifies, observer queries for what it needs); reactive extensions build entire frameworks around this.

### State
**Intent**: allow an object to alter its behavior when its internal state changes, appearing as if the object changed class.
**Motivation**: TCP connection states (Listen, SynSent, Established, ...) — each state behaves completely differently for the same operations.

```
Context --(has)-- state: State --(delegates)-- handle()
|
ConcreteStateA/B --(implements)-- handle() --(transitions)-- context.state = newState
```

| Pros | Cons |
|---|---|
| Localizes state-specific behavior into its own class | Many states means many classes |
| Keeps the state space flat and explicit | Initial setup/wiring of transitions takes care |

```python
class State: def insert_money(self, ctx): pass
class NoQuarterState(State):
    def insert_money(self, ctx):
        print("Quarter inserted"); ctx.state = ctx.has_quarter_state
class VendingMachine:
    def __init__(self): self.state = NoQuarterState()
    def insert_money(self): self.state.insert_money(self)
vm = VendingMachine(); vm.insert_money()
```

**Known uses**: UML state machines, workflow engines. **Relations**: structurally similar to Strategy, but State is driven by internal transitions rather than external client choice. **Advanced**: hierarchical state machines (nested states); orthogonal regions for independent concurrent state.

### Strategy
**Intent**: define a family of algorithms, encapsulate each one, and make them interchangeable at runtime.
**Motivation**: choosing a sort algorithm (quicksort vs. mergesort) based on the data characteristics.

```
Context --(has)-- strategy: Strategy --(delegates)-- algorithmInterface()
|
ConcreteStrategyA/B --(implements)-- algorithmInterface()
```

| Pros | Cons |
|---|---|
| Interchangeable algorithm families | Clients need to know the available strategies to choose one |
| Eliminates large conditional blocks | Overhead/ceremony for very simple cases |

```python
class PaymentStrategy: def pay(self, amount): pass
class CreditCard(PaymentStrategy): def pay(self, amount): print(f"Paid {amount} by CC")
class ShoppingCart:
    def __init__(self): self.strategy = None
    def set_strategy(self, s): self.strategy = s
    def checkout(self, amount): self.strategy.pay(amount)
cart = ShoppingCart(); cart.set_strategy(CreditCard()); cart.checkout(100)
```

**Known uses**: Java's `Comparator`, GUI layout managers. **Relations**: like State, but focused on client-selected algorithms rather than internally driven transitions. **Advanced**: runtime strategy selection via a factory; in functional-programming style languages, a plain lambda often replaces the whole class hierarchy.

### Template Method
**Intent**: define the skeleton of an algorithm in a superclass, letting subclasses override specific steps without changing the algorithm's overall structure.
**Motivation**: framework lifecycle hooks (e.g., test setup/teardown around a test body).

```
AbstractClass --(defines)-- templateMethod() { step1(); step2(); step3(); }
             --(abstract)-- primitive ops
|
ConcreteClass --(implements)-- primitive ops
```

| Pros | Cons |
|---|---|
| Reuses the invariant parts of an algorithm | Hard to change the overall structure later |
| Inverts control to the framework (Hollywood Principle) | Fragile base class problem — changes to the base ripple to all subclasses |

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

**Known uses**: Java's `AbstractList`, JUnit's `@Before`/`@After`. **Relations**: the canonical example of the Hollywood Principle ("Don't call us, we'll call you"). **Advanced**: optional "hook" methods with default no-op implementations that subclasses can override selectively — the mechanism most application frameworks are built around.

### Visitor
**Intent**: represent an operation to be performed on elements of an object structure, without changing the classes of the elements it operates on.
**Motivation**: adding new operations to a Composite structure (e.g., a report generator walking an AST) without touching every element class.

```
Element --(defines)-- accept(visitor: Visitor)
|
ConcreteElementA/B --(implements)-- accept(v) { v.visit(this) }
Visitor --(defines)-- visitElementA/B()
|
ConcreteVisitor --(implements)-- visit() { do op on element }
ObjectStructure --(iterates)-- elements, --(applies)-- visitor
```

| Pros | Cons |
|---|---|
| Adds new operations without subclassing elements | Adding a *new element type* breaks every existing visitor |
| Can accumulate state across a traversal | Can violate encapsulation (visitor needs access to element internals) |

```python
class Shape: def accept(self, v): v.visit_shape(self)
class Circle(Shape): def accept(self, v): v.visit_circle(self)
class AreaVisitor:
    def visit_shape(self, s): print("Unknown")
    def visit_circle(self, c): print(f"Area: {3.14 * c.radius**2}")
shapes = [Circle()]
for s in shapes: s.accept(AreaVisitor())
```

**Known uses**: compiler type checkers, IDE plugin systems (e.g., Eclipse). **Relations**: relies on double dispatch; commonly paired with Composite. **Advanced**: acyclic visitors for graph structures where a plain visitor would create circular dependencies; functional-style folds are the FP equivalent.

## Patterns for backend and distributed systems

The GoF catalog predates distributed systems as we build them today. A few patterns aren't in the original 23 but are just as fundamental for backend work — they deal with *failure*, not just structure.

### Circuit breaker
**Intent**: stop calling a dependency that's already failing, instead of piling up latency and retries on top of an outage. Modeled after an electrical circuit breaker — trip it open, let the failing system recover, then cautiously let traffic back through.

**States**: Closed (normal, requests flow through) → Open (failing fast, no requests sent to the dependency) → Half-Open (a trial request is allowed through to check if the dependency has recovered) → back to Closed or Open depending on the result.

```python
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.state = "CLOSED"
        self.opened_at = None

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.opened_at > self.reset_timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit open — failing fast")
        try:
            result = func(*args, **kwargs)
            self.failure_count = 0
            self.state = "CLOSED"
            return result
        except Exception:
            self.failure_count += 1
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
                self.opened_at = time.time()
            raise
```

**Why it matters**: without a circuit breaker, a slow or dead downstream dependency causes calling threads/connections to pile up waiting on timeouts, which can exhaust the caller's own resources and cascade the outage upstream — this is how one failing service takes down an entire system. Netflix's Hystrix (now largely superseded by resilience4j) popularized this pattern for microservices. Structurally, a circuit breaker is a Proxy or Decorator around the real call, with State managing the Closed/Open/Half-Open transitions internally.

### Retry with backoff
**Intent**: transient failures (a dropped packet, a momentarily overloaded server) often succeed on retry — but naive immediate retries can amplify load on an already-struggling system. Exponential backoff with jitter spaces retries out and avoids synchronized retry storms across many clients.

```python
import random
import time

def retry_with_backoff(func, max_retries=5, base_delay=0.5, max_delay=30):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            delay = min(max_delay, base_delay * (2 ** attempt))
            jitter = random.uniform(0, delay * 0.5)
            time.sleep(delay + jitter)
```

**Why it matters**: without jitter, many clients that failed at the same moment (e.g., due to a brief network blip) all retry at exactly the same backed-off intervals, creating synchronized load spikes ("thundering herd") that can prevent the recovering service from ever stabilizing. AWS's own architecture blog is the standard reference for why "full jitter" backoff outperforms plain exponential backoff. Often combined with Circuit Breaker — stop retrying entirely once the breaker trips open.

### Adapter for backend integrations
The GoF Adapter pattern (above) is worth calling out specifically for backend work because it's the standard way to isolate your codebase from a third-party API's shape — so that swapping providers, or absorbing an upstream breaking change, touches one file instead of every call site.

```python
class PaymentGateway:
    """Our internal, stable interface."""
    def charge(self, amount_cents: int, currency: str) -> str:
        raise NotImplementedError

class StripeAdapter(PaymentGateway):
    def __init__(self, stripe_client):
        self.client = stripe_client
    def charge(self, amount_cents: int, currency: str) -> str:
        # Translate our interface into Stripe's actual SDK call shape
        intent = self.client.PaymentIntent.create(
            amount=amount_cents, currency=currency, confirm=True
        )
        return intent.id

class BraintreeAdapter(PaymentGateway):
    def __init__(self, braintree_gateway):
        self.gateway = braintree_gateway
    def charge(self, amount_cents: int, currency: str) -> str:
        result = self.gateway.transaction.sale({
            "amount": str(amount_cents / 100), "options": {"submit_for_settlement": True}
        })
        return result.transaction.id
```

Application code calls `PaymentGateway.charge()` and never touches Stripe- or Braintree-specific shapes directly — the adapter absorbs the difference. This is also exactly the pattern behind ORMs (adapting different SQL dialects to one query API) and cloud-provider abstraction layers (adapting S3/GCS/Azure Blob to one storage interface).

## Anti-patterns and selection guide

**Anti-patterns**: overuse of any pattern leads to "patternitis" — unnecessary complexity from applying a pattern where a plain function or object would do. Singleton used as a disguised global variable is the most common offender; Abstract Factory with only one concrete factory is another (you paid the complexity cost for flexibility you don't use).

**Modern adaptations**: functional languages express Strategy as a plain function/lambda parameter instead of a class hierarchy; monads are sometimes described as a functional analog to Strategy/Template Method. Circuit Breaker is essentially Proxy + State applied to failure handling. Thread-safe Singleton matters a lot more once concurrency is in play — the naive `__new__` implementation above is not thread-safe under concurrent first access.

| Category | When to use | Example domain |
|---|---|---|
| Creational | Flexible, decoupled instantiation | Frameworks, plugin systems |
| Structural | Composition over inheritance | UI toolkits, libraries, API integration layers |
| Behavioral | Communication and algorithm variation | Application logic, workflow engines |
| Resilience (Circuit Breaker, Retry) | Handling failure in distributed calls | Microservices, any network call to a dependency |

**Critiques**: the GoF patterns are heavily OO-centric and reflect Java/C++-era design constraints; Enterprise patterns (see *Patterns of Enterprise Application Architecture*, POSA) extend the same thinking to distributed/enterprise systems; REST-specific patterns exist for web API design specifically.

**Learning approach**: implement each pattern once by hand in a language you know well, then look for it (or its absence) in code you already use — Spring, for instance, leans on Proxy, Template Method, and Strategy extensively; most HTTP client libraries use Builder for request construction.

## Quick reference
- Need exactly one instance, globally accessible → Singleton (but watch for hidden global-state issues).
- Need to create objects without hardcoding the concrete class → Factory Method / Abstract Factory.
- Need to build a complex object step by step → Builder.
- Need to avoid an expensive constructor → Prototype (clone instead).
- Need to make two incompatible interfaces work together → Adapter.
- Need to add behavior at runtime without subclassing → Decorator.
- Need a simple entry point to a complex subsystem → Facade.
- Need to control/delay access to an object → Proxy.
- Need to notify many listeners of a state change → Observer.
- Need interchangeable algorithms → Strategy.
- Need to protect a call to a failing dependency → Circuit Breaker.
- Need to survive transient failures without hammering the dependency → Retry with backoff + jitter.

## Further reading
- Gamma, Helm, Johnson, Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (the original GoF book).
- Fowler, *Patterns of Enterprise Application Architecture* — patterns for the enterprise/distributed-systems layer GoF doesn't cover.
- Nygard, *Release It!* — the standard reference for Circuit Breaker and other stability patterns in production systems.
- AWS Architecture Blog, "Exponential Backoff and Jitter" — the canonical explanation of why jittered backoff beats plain exponential backoff.
