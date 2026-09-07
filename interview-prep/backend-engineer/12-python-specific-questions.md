# Python — Language-Specific Interview Questions

> Sourced/topic-checked against InterviewBit's Python interview question list. Complements `13-scripting-automation.md` in the devops section (which covers Python for automation scripts) with core language/CPython-internals questions relevant to backend roles.

---

### 1. What is the GIL (Global Interpreter Lock), and how does it affect multithreaded Python programs?
(See also the concurrency file's Q15 for the general framing.) The GIL is a mutex in CPython (the standard Python implementation) that allows only one thread to execute Python bytecode at a time, even on a multi-core machine. This means pure-Python CPU-bound code doesn't get true parallelism from `threading` — but I/O-bound code still benefits significantly, since the GIL is released during blocking I/O calls (file/network operations), letting other threads run in the meantime. For genuine CPU-bound parallelism, `multiprocessing` (separate processes, each with its own GIL and memory space) or offloading to a C-extension library that releases the GIL internally (like NumPy for numerical work) is required instead.

### 2. What's the difference between a list and a tuple, and why can a tuple be used as a dictionary key while a list cannot?
Lists are mutable (can be modified after creation); tuples are immutable. Dictionary keys must be hashable (their hash value must never change over their lifetime, since the hash determines where they're stored in the dict's internal hash table) — a mutable object like a list can't provide this guarantee (its contents, and thus its hash, could change after insertion, corrupting the dict's internal structure), so Python simply doesn't define `__hash__` for lists at all, making them unusable as dict keys, while tuples (being immutable, and hashable as long as all their elements are also hashable) work fine.

### 3. Explain Python decorators, and write a simple example.
A decorator is a function that takes another function as input and returns a modified/wrapped version of it, without changing the original function's own source code — used to add cross-cutting behavior (logging, timing, access control, memoization) declaratively via the `@decorator_name` syntax above a function definition.
```python
import time
def timed(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.3f}s")
        return result
    return wrapper

@timed
def slow_operation():
    time.sleep(1)
```
Calling `slow_operation()` now transparently times and logs its execution, without `slow_operation`'s own body needing any timing code — the same pattern underlies much of Flask/Django's routing decorators (`@app.route(...)`) and Python's built-in `@property`/`@staticmethod`/`@classmethod`.

### 4. What are Python generators, and how do they differ from regular functions returning a list?
A generator function (using `yield` instead of `return`) produces a sequence of values *lazily*, one at a time, pausing its execution state between each `yield` and resuming exactly where it left off when the next value is requested — rather than computing and returning an entire list upfront. This is far more memory-efficient for large or even infinite sequences (a generator producing a billion values never holds more than one value in memory at a time, whereas a function returning a list of a billion values must materialize the entire list) — directly analogous to the streaming/lazy-evaluation benefit discussed for Node.js streams and the "iterate a file line by line" pattern in the scripting-automation file.

### 5. What's the difference between an iterator and an iterable, and what protocol makes something iterable in Python?
An iterable is any object implementing `__iter__()` (returning an iterator) — something you can loop over with `for x in obj`. An iterator is the object actually doing the iterating, implementing both `__iter__()` (returning itself) and `__next__()` (returning the next value, or raising `StopIteration` when exhausted) — an iterator tracks its own current position/state, while an iterable might not (a list is iterable but isn't itself an iterator; calling `iter(my_list)` produces a fresh iterator object each time, which is why you can loop over the same list multiple times independently).

### 6. What do `*args` and `**kwargs` mean, and give a practical use case for each.
`*args` collects any number of extra positional arguments into a tuple inside the function; `**kwargs` collects any number of extra keyword arguments into a dict. Practical use case: a generic decorator/wrapper function (like Q3's example) that needs to forward whatever arguments the wrapped function was called with, without knowing in advance what those arguments will be — `def wrapper(*args, **kwargs): return func(*args, **kwargs)` works for wrapping *any* function signature, which is essential for writing generic, reusable decorators rather than one hand-written per specific function signature.

### 7. Explain the difference between deep copy and shallow copy in Python.
A shallow copy (`copy.copy()`, or a list's `.copy()`/`list(x)`) creates a new container object, but the elements *inside* it are still references to the exact same underlying objects as the original — mutating a nested mutable object (like a list inside a list) through the copy also affects the original, since both point to the same inner object. A deep copy (`copy.deepcopy()`) recursively copies every nested object too, producing a fully independent structure with no shared references at any level — necessary whenever you need to modify a copied nested structure without any risk of affecting the original, at the cost of more time/memory for complex nested data.

### 8. How is memory managed in Python — explain reference counting and the garbage collector's role alongside it.
CPython's primary memory management mechanism is reference counting — every object tracks how many references point to it, and is immediately deallocated the moment that count drops to zero. This alone can't handle reference cycles (object A references object B, which references back to A — neither's count ever reaches zero even if nothing external references either) — Python's separate generational garbage collector periodically scans for and cleans up these unreachable reference cycles specifically, running alongside (not replacing) reference counting as the primary mechanism.

### 9. What is the difference between `is` and `==` in Python, and why can `a is b` sometimes unexpectedly be `True` for small integers or short strings?
`==` compares value equality (can be customized via `__eq__`); `is` compares object identity (are these literally the same object in memory). CPython caches and reuses small integers (typically -5 to 256) and some short strings ("interning") as a memory optimization — so `a = 5; b = 5; a is b` can be `True` purely as an implementation detail of this caching, not a guaranteed language behavior; relying on `is` for value comparison of anything other than singletons like `None` (see the API design file's related JavaScript-adjacent discussion) is a common and risky mistake, since it can appear to work correctly in casual testing purely by accident of CPython's caching range.

### 10. What is a Python namespace, and how does variable scope resolution work (the LEGB rule)?
A namespace is a mapping from names to objects (essentially a dict) — Python has several nested namespaces active at any point, and resolves a name reference by checking them in order: **L**ocal (the current function's own scope) → **E**nclosing (any enclosing function's scope, relevant for nested/closure functions) → **G**lobal (the module's top-level scope) → **B**uilt-in (Python's built-in names like `len`, `print`). This LEGB order explains many scoping surprises — e.g., a variable assigned inside a function is local by default even if a same-named global variable exists, unless explicitly declared `global`/`nonlocal`.

### 11. What's a common gotcha with using a mutable default argument (like a list or dict) in a Python function definition?
Default argument values are evaluated *once*, at function definition time, not on every call — so `def append_item(item, target=[]): target.append(item); return target` reuses the *same* list object across every call that doesn't explicitly pass its own `target`, causing items to silently accumulate across unrelated calls rather than each call getting a fresh empty list as most people intuitively expect. The standard fix: use `None` as the default and create a fresh mutable object inside the function body if it wasn't provided (`def append_item(item, target=None): target = target if target is not None else []`).

### 12. What is pickling and unpickling, and what's a security concern with unpickling data from an untrusted source?
Pickling serializes a Python object into a byte stream (for storage or transmission); unpickling deserializes it back into a live Python object. The security concern: unpickling is not just "parsing data" — the pickle format can encode instructions that execute arbitrary code during deserialization (by design, to support reconstructing complex custom objects), meaning `pickle.loads()` on data from an untrusted/unauthenticated source is a direct remote code execution vulnerability, not merely a data-integrity risk — untrusted data should be deserialized with a safe, restricted format (JSON, or a schema-validated format) instead, never raw pickle.

### 13. What are Python context managers, and how does the `with` statement relate to them (see also the scripting-automation file's related question)?
A context manager is any object implementing `__enter__()` and `__exit__()`, used with the `with` statement to guarantee setup/teardown logic runs correctly regardless of how the block exits (normal completion or an exception) — `with open("file.txt") as f:` guarantees the file is closed even if an exception occurs while reading it, without needing explicit `try/finally` boilerplate. Custom context managers can be written either as a class (implementing both dunder methods) or more concisely via the `@contextlib.contextmanager` decorator wrapping a generator function that `yield`s exactly once (code before the yield is `__enter__`, code after is `__exit__`).

### 14. What's the difference between `@staticmethod`, `@classmethod`, and a regular instance method?
A regular instance method automatically receives the instance itself (`self`) as its first argument, and can access/modify instance state. `@classmethod` receives the class itself (`cls`, not an instance) as its first argument — used for alternative constructors or logic that operates on class-level state rather than any specific instance (e.g., `@classmethod def from_json(cls, data): return cls(**data)`). `@staticmethod` receives neither `self` nor `cls` — it's just a regular function namespaced inside the class for organizational purposes, with no automatic access to instance or class state at all, used when a method logically belongs with the class but doesn't need to interact with any instance/class data.

### 15. What is duck typing, and how does it relate to Python being a dynamically-typed language?
Duck typing means an object's suitability for an operation is determined by whether it has the required methods/behavior ("if it walks like a duck and quacks like a duck..."), not by its explicit declared type/class hierarchy — Python doesn't require an object to formally implement a specific interface/abstract base class to be used somewhere that expects "something iterable" or "something callable"; it just tries the operation and works as long as the object actually supports it at runtime. This is a direct consequence of Python's dynamic typing (types are checked at runtime, not compile time) and is why Python code often favors "ask forgiveness, not permission" (try the operation, catch the exception if it fails) over explicit upfront type-checking (`isinstance` checks), though the latter is still used where genuinely appropriate.
