# Advanced Python Interview Questions & Answers

> A comprehensive guide of 100+ advanced Python interview questions with detailed theoretical answers, production-ready code examples, and design rationale. Aimed at engineers with 3+ years of professional experience.

## Table of Contents

1. [Advanced Python Mechanics](#1-advanced-python-mechanics) — Decorators, Generators, Context Managers, Metaclasses, Descriptors
2. [Concurrency and Parallelism](#2-concurrency-and-parallelism) — asyncio, threading, multiprocessing
3. [Memory, the GIL, and Garbage Collection](#3-memory-the-gil-and-garbage-collection)
4. [Data Structures and Design Patterns](#4-data-structures-and-design-patterns)
5. [Testing and Profiling](#5-testing-and-profiling)

---

## 1. Advanced Python Mechanics

### Q1. What is a decorator, and how does it work under the hood?

A decorator is a callable that takes another callable (function or class) and returns a replacement callable, usually wrapping the original to extend its behavior without modifying its source. The `@decorator` syntax is pure syntactic sugar: `@d\ndef f(): ...` is exactly equivalent to `f = d(f)`. Decorators rely on the fact that functions are first-class objects and that closures capture the enclosing scope.

```python
import functools
import time
from typing import Callable, TypeVar, ParamSpec

P = ParamSpec("P")
R = TypeVar("R")


def timed(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed = time.perf_counter() - start
            print(f"{func.__name__} took {elapsed:.6f}s")
    return wrapper


@timed
def compute(n: int) -> int:
    return sum(i * i for i in range(n))
```

`functools.wraps` copies `__name__`, `__doc__`, `__wrapped__`, and other metadata from the original to the wrapper, so introspection and debuggers still see the real function. `ParamSpec` preserves the exact signature for type checkers. The `try/finally` ensures timing is recorded even if the wrapped call raises.

### Q2. How do you write a decorator that accepts arguments?

A parameterized decorator needs three nested levels: the outer factory accepts the arguments and returns the actual decorator, which accepts the function and returns the wrapper. The extra layer exists because `@deco(arg)` first *calls* `deco(arg)` and then applies its return value to the function.

```python
import functools
from typing import Callable


def retry(times: int = 3, exceptions: tuple = (Exception,)):
    def decorator(func: Callable):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    last_exc = exc
                    if attempt == times:
                        raise
            raise last_exc  # pragma: no cover
        return wrapper
    return decorator


@retry(times=5, exceptions=(ConnectionError,))
def fetch(url: str) -> bytes:
    ...
```

The three-layer structure cleanly separates configuration (`times`, `exceptions`) from the call-time logic. Re-raising on the final attempt preserves the original traceback rather than swallowing failures, which is critical for debugging in production.

### Q3. What is the difference between a function decorator and a class decorator?

A function decorator returns a callable (often a function or instance). A class decorator can either *be* a class (using `__call__` to make instances callable) or *decorate* a class (receiving and returning a class object). Class-based decorators are useful when the wrapper must hold mutable state across calls, because instance attributes are a natural store, avoiding `nonlocal` gymnastics.

```python
import functools


class CountCalls:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        return self.func(*args, **kwargs)


@CountCalls
def greet(name: str) -> str:
    return f"Hello, {name}"
```

State (`self.count`) lives on the instance, which is cleaner than a closure with `nonlocal`. `functools.update_wrapper` is the function-form of `@wraps`, applied manually because we are not nesting a `def`. A caveat: decorating *methods* this way breaks the descriptor protocol unless you also implement `__get__`.

### Q4. Explain generators and how they differ from regular functions.

A generator is a function containing `yield`; calling it returns a generator object without executing the body. Each `next()` runs until the next `yield`, suspending the frame and preserving local state. This makes generators lazy and memory-efficient: they produce values on demand instead of materializing an entire collection. Under the hood, a generator implements the iterator protocol (`__iter__` and `__next__`) and raises `StopIteration` when the body returns.

```python
from typing import Iterator
from pathlib import Path


def read_large_file(path: Path, chunk_size: int = 8192) -> Iterator[str]:
    with open(path, "r", encoding="utf-8") as fh:
        while True:
            chunk = fh.read(chunk_size)
            if not chunk:
                break
            yield chunk
```

This streams a file of arbitrary size with bounded memory. The `with` block guarantees the file closes even if the consumer stops iterating early and the generator is garbage-collected (which triggers a `GeneratorExit` inside the suspended frame). Returning an `Iterator` rather than a `list` is the idiomatic way to express a stream.

### Q5. What does `yield from` do, and when is it useful?

`yield from <iterable>` delegates to a sub-iterator: it yields every value the sub-iterator produces and, crucially, establishes a transparent two-way channel so that values sent via `.send()`, exceptions thrown via `.throw()`, and the sub-generator's return value all propagate correctly. It replaces a manual `for x in sub: yield x` loop and is the foundation that `asyncio` was originally built on before `async`/`await`.

```python
from typing import Iterator


def flatten(nested) -> Iterator:
    for item in nested:
        if isinstance(item, (list, tuple)):
            yield from flatten(item)
        else:
            yield item


list(flatten([1, [2, [3, 4], 5], [6]]))  # [1, 2, 3, 4, 5, 6]
```

`yield from` makes recursive delegation concise and correct. A plain loop would also work for simple yielding, but `yield from` additionally forwards `send`/`throw` semantics and captures the sub-generator's `return` value as the expression result — behavior a manual loop cannot replicate without significant boilerplate.

### Q6. How do you send values into a generator? Explain coroutines via `send`.

Generators are bidirectional: `gen.send(value)` resumes the generator, and the `value` becomes the result of the suspended `yield` expression. This turns a generator into a coroutine that can receive data over time. You must "prime" the generator with `next()` (or `send(None)`) once before sending real values, to advance to the first `yield`.

```python
from typing import Generator


def running_average() -> Generator[float, float, None]:
    total = 0.0
    count = 0
    average = 0.0
    while True:
        value = yield average      # emit current avg, receive next value
        total += value
        count += 1
        average = total / count


avg = running_average()
next(avg)                # prime
avg.send(10)             # 10.0
avg.send(20)             # 15.0
```

The generator maintains accumulator state across `send` calls without any external object. Priming is mandatory because before the first `yield` there is no suspended expression to receive a value. This pattern predates `asyncio` and illustrates the coroutine mechanics that `async def` now formalizes.

### Q7. What is a context manager and how does the protocol work?

A context manager defines runtime setup/teardown around a block via the `with` statement. It implements `__enter__` (called on entry, its return value bound by `as`) and `__exit__(exc_type, exc_val, exc_tb)` (called on exit, guaranteed even on exceptions). If `__exit__` returns a truthy value, the exception is suppressed. Context managers are the Pythonic way to manage resources deterministically.

```python
import threading
from types import TracebackType
from typing import Optional, Type


class LockTimer:
    def __init__(self, lock: threading.Lock, name: str):
        self.lock = lock
        self.name = name

    def __enter__(self) -> "LockTimer":
        self.lock.acquire()
        return self

    def __exit__(
        self,
        exc_type: Optional[Type[BaseException]],
        exc_val: Optional[BaseException],
        exc_tb: Optional[TracebackType],
    ) -> bool:
        self.lock.release()
        return False  # propagate any exception
```

Returning `False` from `__exit__` ensures exceptions are never silently swallowed — a deliberate choice, since hiding errors in resource cleanup is a common source of subtle bugs. The lock is released even if the guarded block raises, guaranteeing no deadlock.

### Q8. How does `contextlib.contextmanager` simplify writing context managers?

The `@contextmanager` decorator turns a single generator into a context manager. Code before `yield` runs on entry, the yielded value is bound by `as`, and code after `yield` runs on exit. To handle exceptions, you wrap the `yield` in `try/finally` (for cleanup) or `try/except` (to suppress). It eliminates the boilerplate of a class with two dunder methods.

```python
import time
from contextlib import contextmanager
from typing import Iterator


@contextmanager
def timer(label: str) -> Iterator[None]:
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.4f}s")


with timer("db-query"):
    run_query()
```

The `finally` block is essential: it guarantees the timing print happens even if `run_query()` raises. An exception propagating through the `yield` is re-raised at the `yield` point inside the generator, so a bare generator without `try/finally` would skip its teardown on error.

### Q9. What is a metaclass and when would you legitimately use one?

A metaclass is the class of a class: just as an object is an instance of a class, a class is an instance of its metaclass (`type` by default). Metaclasses control class *creation* — you override `__new__`/`__init__` on the metaclass to inspect or mutate the class body, attributes, and bases before the class object exists. Legitimate uses are rare: registering subclasses automatically, enforcing API constraints across a hierarchy, or building ORMs/serialization frameworks. The maxim is: "if you wonder whether you need a metaclass, you don't."

```python
class RegistryMeta(type):
    registry: dict = {}

    def __new__(mcs, name, bases, namespace, **kwargs):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:  # skip the base class itself
            RegistryMeta.registry[name.lower()] = cls
        return cls


class Plugin(metaclass=RegistryMeta):
    pass


class JsonPlugin(Plugin):
    pass

# RegistryMeta.registry == {"jsonplugin": JsonPlugin}
```

The metaclass auto-registers every subclass at definition time, eliminating manual registration calls that are easy to forget. Guarding on `bases` skips the abstract root. In modern Python, `__init_subclass__` covers most of these cases more simply — reach for a metaclass only when you must alter the namespace or bases themselves.

### Q10. How does `__init_subclass__` compare to a metaclass?

`__init_subclass__` is a classmethod hook (implicitly) called on the *parent* whenever a subclass is defined, receiving the new subclass. It covers the most common metaclass use case — reacting to subclass creation — without the conceptual weight and composition problems of metaclasses (two classes with different metaclasses cannot be combined). Prefer it for validation, registration, and default-setting.

```python
class Serializable:
    _formats: dict = {}

    def __init_subclass__(cls, *, format_name: str, **kwargs):
        super().__init_subclass__(**kwargs)
        if format_name in Serializable._formats:
            raise ValueError(f"Duplicate format: {format_name}")
        Serializable._formats[format_name] = cls


class XmlSerializer(Serializable, format_name="xml"):
    pass
```

The keyword argument `format_name` is passed in the class definition's bracket list and consumed by `__init_subclass__`, enabling declarative configuration. Calling `super().__init_subclass__(**kwargs)` is mandatory for cooperative multiple inheritance. This is dramatically simpler than a metaclass and composes cleanly.

### Q11. Explain the descriptor protocol and its three method types.

A descriptor is an object defining any of `__get__`, `__set__`, or `__delete__`; placed as a class attribute, it intercepts attribute access on instances. A *data descriptor* defines `__set__` or `__delete__` and takes precedence over the instance `__dict__`; a *non-data descriptor* defines only `__get__` and is overridden by instance attributes. Descriptors power `property`, `classmethod`, `staticmethod`, and `functions-as-methods` (binding `self`).

```python
class Positive:
    def __set_name__(self, owner, name):
        self._name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self._name)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self._name[1:]} must be positive, got {value}")
        setattr(obj, self._name, value)


class Account:
    balance = Positive()

    def __init__(self, balance: float):
        self.balance = balance   # triggers Positive.__set__ → validation
```

`__set_name__` (Python 3.6+) automatically captures the attribute name, removing the need to pass it explicitly. Because `Positive` defines `__set__`, it is a data descriptor and reliably intercepts every assignment, enforcing the invariant centrally — far cleaner than scattering validation across setters.

### Q12. What is the difference between `__new__` and `__init__`?

`__new__` is the static constructor that allocates and returns the new instance; `__init__` is the initializer that configures an already-created instance and must return `None`. `__new__` runs first and is where you control *what* object is produced — essential for immutable types (whose value must be set at creation) and for patterns like singletons or returning cached/subclass instances. `__init__` is for ordinary attribute setup.

```python
class Cached:
    _instances: dict = {}

    def __new__(cls, key: str):
        if key in cls._instances:
            return cls._instances[key]
        instance = super().__new__(cls)
        cls._instances[key] = instance
        return instance

    def __init__(self, key: str):
        self.key = key
```

Overriding `__new__` lets us return an existing instance for a repeated key (flyweight/cache). Note a subtlety: `__init__` still runs on the cached instance each time, so idempotent init or a guard flag may be needed. For immutable subclasses like `int` or `tuple`, `__new__` is the *only* place to set the value.

### Q13. How does Python resolve method lookups (MRO)? Explain C3 linearization.

The Method Resolution Order is the deterministic sequence Python searches for an attribute across a class and its bases. Python uses the C3 linearization algorithm, which guarantees three properties: a class precedes its parents, the order of parents in the definition is preserved, and the result is monotonic (consistent across the hierarchy). C3 is what makes cooperative multiple inheritance with `super()` work predictably; if no consistent linearization exists, the class definition raises `TypeError`.

```python
class A:
    def who(self): return "A"

class B(A):
    def who(self): return "B -> " + super().who()

class C(A):
    def who(self): return "C -> " + super().who()

class D(B, C):
    def who(self): return "D -> " + super().who()


D().who()           # 'D -> B -> C -> A'
D.__mro__           # (D, B, C, A, object)
```

`super()` follows the MRO of the *instance's* type, not the static parent, so `B.who` dispatches to `C`, not directly to `A` — this is how the diamond is traversed exactly once. Inspecting `__mro__` is the reliable way to reason about which implementation wins.

### Q14. What are `__slots__` and what are the trade-offs?

`__slots__` declares a fixed set of instance attributes, instructing Python to store them in a compact array-like structure instead of a per-instance `__dict__`. This saves significant memory (no dict overhead) and slightly speeds attribute access. Trade-offs: you lose dynamic attribute assignment, `__dict__` and `__weakref__` are gone unless re-added, and inheritance requires care (subclasses without `__slots__` regain a `__dict__`).

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y


p = Point(1.0, 2.0)
# p.z = 3  -> AttributeError: 'Point' object has no attribute 'z'
```

For a program creating millions of small objects (e.g., points, graph nodes), `__slots__` can cut memory use by 40–50%. The restriction against arbitrary attributes is usually acceptable — and even desirable — for such value-like classes. Avoid it on classes needing dynamic attributes or multiple inheritance with conflicting slots.

### Q15. Explain the difference between `@staticmethod`, `@classmethod`, and instance methods.

An instance method receives the instance as `self` and operates on instance state. A `@classmethod` receives the class as `cls` and is the idiomatic vehicle for alternative constructors and operations on class-level state; it respects inheritance, since `cls` is the actual subclass. A `@staticmethod` receives neither — it is a plain function namespaced under the class, used to group logically related utilities. All three are implemented as descriptors.

```python
from __future__ import annotations
from datetime import date


class Employee:
    def __init__(self, name: str, birth_year: int):
        self.name = name
        self.birth_year = birth_year

    @classmethod
    def from_csv_row(cls, row: str) -> "Employee":
        name, year = row.split(",")
        return cls(name.strip(), int(year))   # cls => correct subclass

    @staticmethod
    def is_valid_year(year: int) -> bool:
        return 1900 <= year <= date.today().year
```

`from_csv_row` uses `cls`, so a subclass calling it produces a subclass instance — the key advantage over hardcoding `Employee(...)`. `is_valid_year` needs no instance or class state, so `@staticmethod` documents that intent and avoids an unused `self` parameter.

### Q16. What is monkey patching and when is it justified?

Monkey patching is replacing or adding attributes/methods on a module, class, or instance at runtime. It is justified narrowly: patching third-party code in tests (often via `unittest.mock.patch`), applying urgent hotfixes to libraries you cannot modify, or feature-flagging. In production application code it is dangerous — it creates action-at-a-distance, breaks on library upgrades, and confuses static analysis. Prefer subclassing, composition, or dependency injection.

```python
from unittest.mock import patch
import requests


def get_status(url: str) -> int:
    return requests.get(url).status_code


def test_get_status():
    with patch("requests.get") as mock_get:
        mock_get.return_value.status_code = 503
        assert get_status("http://x") == 503
```

Here patching is appropriate: it isolates the unit under test from the network, is scoped to a `with` block (auto-restored), and targets the *namespace where the function is looked up*. Outside tests, the same technique is a code smell that should prompt a redesign toward injectable dependencies.

### Q17. Explain closures and the late-binding gotcha.

A closure is a nested function that captures variables from its enclosing scope; Python stores these in cell objects referenced by the inner function's `__closure__`. The classic pitfall is *late binding*: closures capture variables by reference, not by value, so a loop creating closures all share the final value of the loop variable. The fix is to bind the current value via a default argument (evaluated at definition time).

```python
# Buggy: all functions return 2
funcs_bad = [lambda: i for i in range(3)]
print([f() for f in funcs_bad])    # [2, 2, 2]

# Correct: default arg captures current value
funcs_good = [lambda i=i: i for i in range(3)]
print([f() for f in funcs_good])   # [0, 1, 2]
```

Default arguments are evaluated once, at function-definition time, freezing the loop variable's current value into the parameter. This is the canonical idiom for snapshotting loop state into closures and a frequent interview discriminator because the late-binding behavior surprises even experienced developers.

### Q18. What is the difference between shallow copy and deep copy?

A shallow copy (`copy.copy` or `obj[:]`) creates a new container but inserts references to the *same* nested objects; mutating a nested object affects both copies. A deep copy (`copy.deepcopy`) recursively copies nested objects, producing a fully independent structure, and it tracks already-copied objects via a memo dict to handle shared references and cycles correctly. Deep copy is slower and can be expensive on large graphs.

```python
import copy

original = {"tags": ["a", "b"], "meta": {"count": 1}}

shallow = copy.copy(original)
shallow["tags"].append("c")
# original["tags"] is now ["a", "b", "c"]  -> shared!

deep = copy.deepcopy(original)
deep["meta"]["count"] = 99
# original["meta"]["count"] still 1  -> independent
```

The shallow copy shares the inner list, so the append leaks into the original — a frequent source of bugs when copying configs or default mutable arguments. `deepcopy` isolates every level; customize its behavior with `__deepcopy__` or `__copy__` if your objects hold un-copyable resources like file handles.

### Q19. How do `*args`, `**kwargs`, and keyword-only / positional-only parameters work?

`*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. A bare `*` in a signature marks all following parameters as keyword-only (must be passed by name), and `/` (3.8+) marks preceding parameters as positional-only (cannot be passed by name). These give precise control over an API's calling convention, improving clarity and forward-compatibility.

```python
def configure(host, /, port=8080, *, timeout, **options):
    # host: positional-only; port: positional-or-keyword;
    # timeout: keyword-only; options: extra keywords
    return {"host": host, "port": port, "timeout": timeout, **options}


configure("db", 5432, timeout=30, retries=3)
# configure(host="db", ...) -> TypeError: host is positional-only
```

Positional-only `host` lets you rename the parameter later without breaking callers (they never used the name). Keyword-only `timeout` forces an explicit, self-documenting call site, preventing accidental positional misuse. This level of signature control is a hallmark of well-designed library APIs.

### Q20. What are `functools.lru_cache` and `functools.cache`, and what are their caveats?

`lru_cache(maxsize=N)` memoizes a function's return values keyed by its arguments, evicting least-recently-used entries when full; `cache` (3.9+) is `lru_cache(maxsize=None)` — an unbounded memoizer. They dramatically speed up pure functions with repeated inputs. Caveats: arguments must be hashable, the cache holds strong references (a memory-leak risk for object arguments), it is not safe to assume thread-isolation of *computation* (the function may run concurrently), and caching impure functions yields stale results.

```python
import functools


@functools.lru_cache(maxsize=1024)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)


fibonacci(100)                 # fast: O(n) instead of exponential
fibonacci.cache_info()         # CacheInfo(hits=..., misses=..., ...)
fibonacci.cache_clear()        # reset
```

Memoization collapses the exponential recursion into linear time by reusing subproblem results. `cache_info()` exposes hit/miss telemetry for tuning `maxsize`. In long-running services, prefer a bounded `maxsize` over `cache` to cap memory, and never decorate methods of objects you need garbage-collected, since the cache pins `self`.


---

## 2. Concurrency and Parallelism

### Q21. What is the difference between concurrency and parallelism in Python?

Concurrency is *dealing with* many tasks by interleaving their progress (structuring work so tasks can make progress without blocking each other); parallelism is *literally executing* multiple tasks simultaneously on multiple cores. In CPython, threads and `asyncio` give concurrency but, due to the GIL, not CPU parallelism — only `multiprocessing` (or C extensions/`subinterpreters`) achieves true parallel computation. The practical rule: I/O-bound → `asyncio` or threads; CPU-bound → processes.

```python
# Concurrency (I/O-bound) via asyncio: overlapping waits
import asyncio


async def fetch(client, url):
    return await client.get(url)


async def main(urls):
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(*(fetch(client, u) for u in urls))
```

`gather` schedules all fetches concurrently on one thread; while one awaits network I/O, others proceed, so total time approaches the slowest single request rather than the sum. This is concurrency without parallelism — ideal for I/O, useless for crunching numbers, which is the distinction interviewers probe.

### Q22. Explain the Global Interpreter Lock (GIL) and its consequences.

The GIL is a mutex in CPython that allows only one thread to execute Python bytecode at a time, protecting interpreter internals (notably reference counts) from data races. Its consequence: multithreaded pure-Python CPU-bound code does not speed up — threads contend for the single lock and effectively serialize. The GIL is released during blocking I/O and inside many C extensions (NumPy, compression, hashing), so threads *do* help I/O-bound and C-heavy workloads.

```python
import threading

def cpu_task(n):
    total = 0
    for _ in range(n):
        total += 1
    return total

# Two threads do NOT halve the time for CPU work — the GIL serializes them.
threads = [threading.Thread(target=cpu_task, args=(10_000_000,)) for _ in range(2)]
for t in threads: t.start()
for t in threads: t.join()
```

This code runs no faster than sequential execution because both threads compete for the GIL on every bytecode batch. The takeaway for design: use `multiprocessing` or release the GIL via a C extension for CPU parallelism. (Note: Python 3.13 introduced an experimental free-threaded build that removes the GIL.)

### Q23. When should you use `threading` versus `multiprocessing`?

Use `threading` for I/O-bound concurrency — network calls, disk reads, database queries — where threads spend most time blocked (with the GIL released) and the low memory/startup cost of threads matters. Use `multiprocessing` for CPU-bound parallelism — each process has its own interpreter and GIL, so they truly run in parallel across cores, at the cost of higher memory use and inter-process communication overhead (data must be pickled).

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# I/O-bound: threads
with ThreadPoolExecutor(max_workers=32) as ex:
    pages = list(ex.map(download, urls))

# CPU-bound: processes
with ProcessPoolExecutor(max_workers=8) as ex:
    results = list(ex.map(heavy_compute, datasets))
```

The two executors share an identical API, so switching execution models is a one-line change — a strong argument for coding against `concurrent.futures` rather than raw threads/processes. Thread pools can use many workers (threads are cheap and mostly idle on I/O); process pools should roughly match core count to avoid context-switching overhead.

### Q24. Explain `asyncio`'s event loop, coroutines, and tasks.

`asyncio` runs an event loop on a single thread that schedules coroutines cooperatively. An `async def` function is a coroutine; `await` suspends it, yielding control back to the loop, which runs other ready coroutines until the awaited operation completes. A `Task` wraps a coroutine and schedules it to run concurrently on the loop. Cooperative scheduling means a coroutine that never awaits (e.g., a CPU loop) blocks the entire loop.

```python
import asyncio


async def worker(name: str, delay: float) -> str:
    await asyncio.sleep(delay)         # yields control to the loop
    return f"{name} done"


async def main():
    task1 = asyncio.create_task(worker("A", 2))
    task2 = asyncio.create_task(worker("B", 1))
    # Both run concurrently; total ~2s, not 3s
    return await asyncio.gather(task1, task2)


asyncio.run(main())
```

`create_task` schedules the coroutines immediately and concurrently, so the two `sleep`s overlap. `asyncio.run` creates and manages the loop lifecycle. The critical discipline: never call blocking functions (e.g., `time.sleep`, synchronous DB drivers) inside a coroutine — offload them with `loop.run_in_executor` to avoid stalling the loop.

### Q25. What is the difference between `asyncio.gather` and `asyncio.wait`?

`asyncio.gather` runs awaitables concurrently and returns their results as an ordered list matching the input; by default the first exception propagates (cancelling is opt-in via `return_exceptions`). `asyncio.wait` is lower-level: it accepts tasks, returns two sets `(done, pending)`, never raises on task failure (you inspect each task), and supports `return_when` conditions like `FIRST_COMPLETED`. Use `gather` for the common "run all, collect results" case; `wait` when you need fine-grained control over completion.

```python
import asyncio

async def main(coros):
    # gather: ordered results, fail-fast or collect exceptions
    results = await asyncio.gather(*coros, return_exceptions=True)

    # wait: react as soon as the first finishes
    tasks = [asyncio.create_task(c) for c in coros]
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)
    for p in pending:
        p.cancel()
    return results, done
```

`return_exceptions=True` makes `gather` return exceptions as values instead of raising, so one failure does not lose the other results. `wait` with `FIRST_COMPLETED` plus cancelling the rest implements a "race" pattern (e.g., querying replicas and taking the fastest). In modern code, `asyncio.TaskGroup` (3.11+) is often preferable to both.

### Q26. What is `asyncio.TaskGroup` and why is it preferred over raw `gather`?

`TaskGroup` (Python 3.11+) provides *structured concurrency*: tasks are created inside an `async with` block and the block does not exit until all complete. If any task raises, the rest are automatically cancelled and the errors are surfaced as an `ExceptionGroup`. This eliminates orphaned tasks and the manual cleanup that `gather` requires, making concurrency scopes lexically explicit and leak-free.

```python
import asyncio


async def main(urls):
    results = []
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(u)) for u in urls]
    # On exit, all tasks are guaranteed finished or cancelled
    return [t.result() for t in tasks]
```

If `fetch` raises for one URL, the `TaskGroup` cancels the siblings and re-raises inside an `ExceptionGroup`, so no task is left running detached — a common bug with bare `create_task`. The lexical scope mirrors how `with` manages resources, which is why structured concurrency is considered the modern best practice.

### Q27. How do you safely share state between threads? Explain locks, RLocks, and race conditions.

A race condition occurs when the correctness of a result depends on the unpredictable interleaving of threads accessing shared mutable state. The fix is mutual exclusion via a `Lock` around the critical section. An `RLock` (reentrant lock) can be acquired multiple times by the same thread — necessary when a locked method calls another locked method. Always acquire locks via `with` so they release on exceptions, and keep critical sections minimal to reduce contention.

```python
import threading


class Counter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self) -> None:
        with self._lock:          # critical section
            self._value += 1      # read-modify-write is NOT atomic

    @property
    def value(self) -> int:
        with self._lock:
            return self._value
```

`self._value += 1` is three bytecode operations (load, add, store); without the lock, two threads can interleave and lose an update. The `with self._lock` block makes it atomic. Even reads are guarded here to ensure visibility and consistency. Note that even a single GIL does not make compound operations atomic — a frequent misconception.

### Q28. What is a deadlock and how do you prevent it?

A deadlock occurs when two or more threads each hold a lock the other needs, so all wait forever. The four Coffman conditions (mutual exclusion, hold-and-wait, no preemption, circular wait) must all hold; breaking any one prevents deadlock. Practical preventions: acquire locks in a consistent global order, use timeouts on acquisition, hold as few locks as possible, and prefer higher-level constructs (queues, `concurrent.futures`) that avoid manual locking.

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()

# Deadlock-prone if another thread locks in B->A order.
# Fix: always acquire in a fixed order (e.g., by id).
def transfer():
    first, second = sorted((lock_a, lock_b), key=id)
    with first:
        with second:
            ...   # safe: every thread uses the same ordering
```

Sorting locks by `id()` imposes a global acquisition order, eliminating the circular-wait condition. Alternatively, `lock.acquire(timeout=...)` lets a thread back off and retry instead of blocking forever. The most robust strategy is architectural: pass data through thread-safe queues so threads never hold multiple locks simultaneously.

### Q29. Explain the producer-consumer pattern with `queue.Queue`.

The producer-consumer pattern decouples threads that generate work from those that process it, communicating through a thread-safe `queue.Queue` that handles all the locking internally. Producers `put` items; consumers `get` them. The queue's blocking semantics and `task_done`/`join` bookkeeping provide natural backpressure (a bounded queue blocks producers when full) and clean shutdown coordination.

```python
import queue
import threading

q: queue.Queue = queue.Queue(maxsize=100)
SENTINEL = object()


def producer():
    for item in range(1000):
        q.put(item)            # blocks if queue is full -> backpressure
    q.put(SENTINEL)


def consumer():
    while True:
        item = q.get()
        if item is SENTINEL:
            q.task_done()
            break
        process(item)
        q.task_done()
```

`Queue` encapsulates the locking, so application code never touches a mutex — eliminating an entire class of race bugs. The `maxsize` bound prevents a fast producer from exhausting memory. A sentinel object signals end-of-stream cleanly; for multiple consumers, send one sentinel per consumer or use `q.join()` to await completion.

### Q30. How do processes communicate in `multiprocessing`?

Because processes have separate memory, they communicate via mechanisms that serialize and transfer data: `multiprocessing.Queue` and `Pipe` (pickle objects across an OS pipe), shared-memory primitives (`Value`, `Array`, and `shared_memory.SharedMemory` for zero-copy buffers), and `Manager` objects (proxied shared containers, slower but flexible). The cost of pickling and IPC means you should transfer coarse-grained chunks of work, not chatty small messages.

```python
from multiprocessing import Process, Queue


def worker(in_q: Queue, out_q: Queue):
    while (item := in_q.get()) is not None:
        out_q.put(item * item)


def main():
    in_q, out_q = Queue(), Queue()
    procs = [Process(target=worker, args=(in_q, out_q)) for _ in range(4)]
    for p in procs: p.start()
    for n in range(100): in_q.put(n)
    for _ in procs: in_q.put(None)         # one sentinel per worker
    results = [out_q.get() for _ in range(100)]
    for p in procs: p.join()
```

Each worker pulls tasks and pushes results through pickled queues; sending one `None` sentinel per worker guarantees every process terminates. For large numeric arrays, prefer `shared_memory` to avoid the pickle copy entirely — a key optimization when IPC volume dominates runtime.

### Q31. What is the difference between `concurrent.futures` and the lower-level `threading`/`multiprocessing` APIs?

`concurrent.futures` is a high-level abstraction providing `Executor` pools and `Future` objects — a uniform, declarative interface over both threads and processes. It handles worker lifecycle, result collection, exception propagation (re-raised when you call `future.result()`), and timeouts. The lower-level modules give finer control (custom synchronization, daemon threads, process start methods) but require manual management. Prefer `concurrent.futures` unless you need that control.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def main(urls):
    with ThreadPoolExecutor(max_workers=16) as ex:
        future_to_url = {ex.submit(fetch, u): u for u in urls}
        for future in as_completed(future_to_url):
            url = future_to_url[future]
            try:
                data = future.result()    # re-raises worker exceptions here
            except Exception as exc:
                log.error("%s failed: %s", url, exc)
```

`as_completed` yields futures in completion order, enabling responsive processing of results as they finish rather than waiting for the slowest. Exceptions raised in workers are captured and re-raised at `result()`, so error handling stays at the call site. The `with` block guarantees the pool is shut down and threads joined.

### Q32. How do you cancel or time out async operations?

Use `asyncio.wait_for(coro, timeout)` to cap a single operation; on expiry it cancels the coroutine and raises `TimeoutError`. For cooperative cancellation, calling `task.cancel()` injects a `CancelledError` at the task's next `await`. Coroutines should let `CancelledError` propagate (catch it only for cleanup, then re-raise) — swallowing it breaks cancellation. Python 3.11 added `asyncio.timeout()` as a context-manager form.

```python
import asyncio


async def guarded():
    try:
        async with asyncio.timeout(5):        # 3.11+
            return await slow_operation()
    except TimeoutError:
        log.warning("operation timed out")
        raise


async def cleanup_aware():
    try:
        await long_running()
    except asyncio.CancelledError:
        await release_resources()              # cleanup
        raise                                  # MUST re-raise
```

`asyncio.timeout()` cancels everything inside the block on expiry, propagating `TimeoutError` cleanly. Re-raising `CancelledError` after cleanup is mandatory: it informs the event loop the task actually stopped, preventing "task was destroyed but it is pending" warnings and hangs during shutdown.

### Q33. What is the `async with` and `async for` protocol?

`async with` works with asynchronous context managers implementing `__aenter__` and `__aexit__` coroutines — for resources whose setup/teardown involves I/O (opening a connection pool, acquiring an async lock). `async for` works with asynchronous iterators implementing `__aiter__` and an `__anext__` coroutine, allowing each iteration step to `await` — ideal for streaming results from a network source without buffering them all.

```python
from typing import AsyncIterator


class AsyncCursor:
    def __init__(self, conn):
        self.conn = conn

    async def __aiter__(self) -> "AsyncCursor":
        return self

    async def __anext__(self):
        row = await self.conn.fetch_one()
        if row is None:
            raise StopAsyncIteration
        return row


async def stream_rows(conn):
    async for row in AsyncCursor(conn):       # awaits per row
        await process(row)
```

`StopAsyncIteration` (not `StopIteration`) terminates an `async for`. The pattern lets each fetch overlap with other coroutines on the loop, so a long result set streams with bounded memory and without blocking — the async analogue of the lazy generator-based file reader from Q4.

### Q34. Explain the GIL's interaction with C extensions and how libraries achieve parallelism.

CPython's C-API exposes `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS` macros that let a C extension release the GIL during long computations or blocking calls that do not touch Python objects. This is why NumPy matrix operations, `hashlib`, `zlib`, and most I/O actually run in parallel across threads despite the GIL — the heavy work happens in GIL-free C code. Understanding this guides when threading helps: it helps precisely when the hot path drops the GIL.

```python
import threading
import numpy as np

# These NumPy ops release the GIL internally, so threads DO parallelize.
def heavy(matrix):
    return matrix @ matrix.T          # BLAS call runs without the GIL

mats = [np.random.rand(1000, 1000) for _ in range(4)]
threads = [threading.Thread(target=heavy, args=(m,)) for m in mats]
for t in threads: t.start()
for t in threads: t.join()
```

Because the matrix multiply dispatches to a BLAS routine that releases the GIL, these four threads genuinely run on multiple cores — unlike the pure-Python loop in Q22. This is the practical reason data-science workloads often use threads effectively while pure-Python CPU loops do not.


### Q35. What is `loop.run_in_executor` and when do you need it?

`run_in_executor` offloads a blocking, synchronous call to a thread or process pool and returns an awaitable, letting an async program use blocking libraries without freezing the event loop. It is the bridge between sync and async worlds: any function that would block (legacy DB drivers, `requests`, heavy CPU work) should be wrapped so the loop stays responsive. With no executor argument it uses the default thread pool; pass a `ProcessPoolExecutor` for CPU-bound work.

```python
import asyncio
import functools
from concurrent.futures import ProcessPoolExecutor


def cpu_bound(data):
    return sum(x * x for x in data)


async def main(datasets):
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        tasks = [
            loop.run_in_executor(pool, functools.partial(cpu_bound, d))
            for d in datasets
        ]
        return await asyncio.gather(*tasks)
```

Wrapping the CPU work in a process pool gives true parallelism (escaping the GIL) while keeping the async interface, so the event loop continues serving I/O. `functools.partial` binds arguments since `run_in_executor` passes none. This sync-async bridge is essential in real services that mix async frameworks with blocking dependencies.

### Q36. Explain thread-local storage and when to use it.

`threading.local()` provides storage whose attribute values are distinct per thread — each thread sees its own version of the data through the same object. It is useful for per-thread context that should not be shared or locked: database connections, request-scoped state, or non-thread-safe library handles. It sidesteps both the need for locks (no sharing) and the awkwardness of threading state through every function call.

```python
import threading

_local = threading.local()


def get_connection():
    if not hasattr(_local, "conn"):
        _local.conn = create_db_connection()   # one per thread
    return _local.conn


def handle_request():
    conn = get_connection()      # same thread reuses its own connection
    return conn.execute(...)
```

Each thread lazily creates and reuses its own connection, avoiding both contention and the cost of per-call setup, without any locking. The caveat: in `asyncio`, many coroutines share one thread, so thread-locals do *not* isolate per-coroutine — use `contextvars.ContextVar` instead, which is async-aware.


---

## 3. Memory, the GIL, and Garbage Collection

### Q37. How does CPython manage memory? Explain reference counting.

CPython's primary memory management is reference counting: every object carries a counter incremented when a new reference is created and decremented when one is removed; at zero, the object is immediately deallocated. This gives deterministic, prompt cleanup and predictable behavior. Its weakness is reference cycles — objects referencing each other never reach zero — which is why CPython supplements it with a cyclic garbage collector.

```python
import sys

a = []
print(sys.getrefcount(a))   # e.g. 2: 'a' + the temporary arg to getrefcount

b = a
print(sys.getrefcount(a))   # 3: 'a', 'b', and the temporary

del b
print(sys.getrefcount(a))   # back to 2
```

`sys.getrefcount` reveals the live count (always one higher than expected due to the function's own argument reference). Because deallocation is immediate at zero, resources like file handles tied to object lifetime are released promptly — though relying on this for cleanup is discouraged in favor of explicit `with` blocks, since other implementations (PyPy) lack refcounting.

### Q38. Explain Python's cyclic garbage collector and the generational hypothesis.

The cyclic GC detects and collects groups of objects that reference each other but are unreachable from the program — cases reference counting alone cannot free. It is generational, based on the empirical observation that most objects die young: new objects start in generation 0, and survivors are promoted to older generations collected less frequently. This concentrates collection effort where garbage is most likely, minimizing overhead.

```python
import gc

class Node:
    def __init__(self):
        self.ref = None

# Create a cycle that refcounting can't free
a, b = Node(), Node()
a.ref = b
b.ref = a
del a, b                  # refcounts are 1, not 0 -> leaked without GC

collected = gc.collect()  # cyclic GC reclaims the pair
print(f"collected {collected} objects")
print(gc.get_threshold())  # (700, 10, 10): gen0/1/2 trigger counts
```

The cycle keeps each node's refcount at 1, so only the cyclic collector can reclaim them. `gc.get_threshold()` shows the allocation counts that trigger each generation. In performance-critical sections that allocate many short-lived non-cyclic objects, you can `gc.disable()` to avoid collection pauses, re-enabling afterward.

### Q39. What are weak references and when are they useful?

A weak reference (`weakref`) points to an object without incrementing its reference count, so it does not keep the object alive. When the referent is collected, the weak reference becomes dead (returns `None`). This is essential for caches, observer registries, and back-references that must not create cycles or prevent garbage collection — letting you observe objects without owning their lifetime.

```python
import weakref


class ExpensiveResource:
    def __init__(self, key):
        self.key = key


# Cache that doesn't prevent values from being collected
cache: "weakref.WeakValueDictionary[str, ExpensiveResource]" = (
    weakref.WeakValueDictionary()
)


def get_resource(key: str) -> ExpensiveResource:
    obj = cache.get(key)
    if obj is None:
        obj = ExpensiveResource(key)
        cache[key] = obj          # weak: GC'd when no strong refs remain
    return obj
```

A `WeakValueDictionary` caches objects only as long as something else holds a strong reference, so the cache never causes leaks by pinning otherwise-dead objects — unlike `lru_cache`, which holds strong references. This is the idiomatic way to build memory-friendly caches and to register listeners without lifetime entanglement.

### Q40. How can you diagnose and fix memory leaks in a Python service?

"Leaks" in Python are usually unbounded growth from lingering references — caches without eviction, ever-growing lists/dicts, registered callbacks, or cycles holding objects with `__del__`. Diagnose with `tracemalloc` (snapshots and diffs allocation by line), `gc` introspection (`gc.get_objects`, `gc.garbage`), and `objgraph` for reference chains. The fix is almost always finding and removing the unintended strong reference or adding eviction.

```python
import tracemalloc

tracemalloc.start()
snapshot1 = tracemalloc.take_snapshot()

run_workload()                          # suspected leaky operation

snapshot2 = tracemalloc.take_snapshot()
top = snapshot2.compare_to(snapshot1, "lineno")
for stat in top[:10]:
    print(stat)                         # biggest allocation growth by line
```

`compare_to` ranks the lines responsible for the largest memory growth between snapshots, pinpointing the leak's origin precisely rather than guessing. Common culprits surfaced this way include module-level mutable defaults, unbounded memoization, and closures capturing large objects. The remedy is bounding caches (`lru_cache(maxsize=...)`, `WeakValueDictionary`) or clearing registries on teardown.

### Q41. What is interning and how do small-integer and string caches affect identity comparisons?

CPython pre-allocates and caches (interns) small integers (-5 to 256) and many short strings, so multiple references to the same literal point to one shared object. This is an optimization, but it makes `is` (identity) accidentally appear to work like `==` (equality) for these values — a trap. The lesson: use `==` for value comparison and reserve `is` for singletons (`None`, `True`, `False`) and intentional identity checks.

```python
a = 256
b = 256
print(a is b)        # True: cached small int

x = 257
y = 257
print(x is y)        # often False: outside the cached range

import sys
s = sys.intern("a_long_repeated_key")   # explicit interning for fast compares
```

Relying on `is` for integers or strings produces code that works for small values and silently breaks for large ones — a classic bug. `sys.intern` is a legitimate optimization for dictionaries with many repeated string keys, since interned strings compare by pointer first, speeding up hashing and equality.

### Q42. Explain how `__del__` works and why finalizers are problematic.

`__del__` is an object's finalizer, called when its refcount hits zero (or during GC). It is unreliable for cleanup: timing is non-deterministic, it may never run at interpreter shutdown, exceptions inside it are ignored, and prior to Python 3.4 it could prevent collection of cycles. Use context managers or `weakref.finalize` for deterministic resource release instead of `__del__`.

```python
import weakref


class FileWrapper:
    def __init__(self, path):
        self.file = open(path)
        # Robust finalizer: runs at most once, no reference to self
        self._finalizer = weakref.finalize(self, self.file.close)

    def close(self):
        self._finalizer()        # idempotent explicit close


# Best practice: deterministic cleanup via context manager
with open("data.txt") as f:
    data = f.read()
```

`weakref.finalize` is safer than `__del__`: it does not hold a strong reference to the object (so it never prevents collection), runs at most once, and can be invoked explicitly and idempotently. Still, the `with` statement remains the gold standard for resource cleanup because it is deterministic and visible at the call site.

### Q43. What is the difference between mutable and immutable types' memory behavior?

Immutable objects (`int`, `str`, `tuple`, `frozenset`) cannot change after creation, so "modifying" them creates new objects; this enables safe sharing, caching/interning, and use as dict keys (stable hashes). Mutable objects (`list`, `dict`, `set`) can be modified in place, sharing one identity across references — which is efficient but causes aliasing bugs when a mutation is unexpectedly visible elsewhere, most infamously the mutable default argument trap.

```python
# Mutable default argument: created ONCE, shared across all calls
def append_bad(item, target=[]):       # BUG
    target.append(item)
    return target

append_bad(1)      # [1]
append_bad(2)      # [1, 2]  <- leaked state!

# Correct: sentinel default
def append_good(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

The default list is evaluated once at function-definition time and persists across calls, accumulating state — a direct consequence of mutability plus default-argument semantics. The `None` sentinel pattern creates a fresh list per call. Understanding this distinction underpins reasoning about aliasing, hashability, and thread safety.

### Q44. How does Python's memory allocator (pymalloc) work at a high level?

CPython uses a tiered allocator. Small objects (≤512 bytes, the vast majority) are served by `pymalloc`, which manages a hierarchy of arenas → pools → blocks, grouping same-sized allocations to reduce fragmentation and avoid frequent `malloc`/`free` syscalls. Larger requests fall through to the system allocator. Freed small-object memory often stays reserved in pools for reuse rather than returning to the OS, which is why a process's RSS may not shrink after objects are freed.

```python
import sys

# Small objects share preallocated pools; observe per-object overhead
print(sys.getsizeof(0))        # ~28 bytes: int object header + value
print(sys.getsizeof([]))       # ~56 bytes: empty list overhead
print(sys.getsizeof([1, 2, 3]))# grows with capacity, not just length
```

`getsizeof` reveals the substantial per-object overhead (headers, type pointers, refcount) that motivates `__slots__` and compact data structures for large collections. The key operational insight: because pymalloc retains freed memory in pools, steady-state memory of a long-running service is governed by peak usage, so bounding peaks matters more than freeing promptly.


### Q45. What are `contextvars` and how do they differ from thread-locals?

`contextvars.ContextVar` provides context-local state that works correctly with `asyncio`: each task gets its own logical copy, and the value follows the task across `await` boundaries. Thread-locals fail here because many coroutines share one thread. `contextvars` is the standard mechanism for request-scoped data (request IDs, auth context, trace spans) in async frameworks, propagating automatically when tasks are created.

```python
import contextvars
import asyncio

request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id")


async def handle(rid: str):
    request_id.set(rid)              # set in this task's context
    await process()                  # value persists across awaits

async def process():
    print(f"handling {request_id.get()}")   # reads this task's value


async def main():
    await asyncio.gather(handle("req-1"), handle("req-2"))
    # Each task sees its own request_id, no cross-contamination
```

Each task carries an isolated copy of `request_id`, so concurrent requests never see each other's values despite running on the same thread — something thread-locals cannot guarantee under `asyncio`. New tasks copy the current context at creation, giving clean, automatic propagation of cross-cutting request state.

---

## 4. Data Structures and Design Patterns

### Q46. When would you choose a `deque` over a `list`?

`collections.deque` is a doubly-linked-list-backed sequence offering O(1) appends and pops at *both* ends, whereas a `list` has O(1) operations only at the right end — `list.pop(0)` and `list.insert(0, x)` are O(n) because they shift every element. Use `deque` for queues, sliding windows, and any FIFO/double-ended workload. It also supports a `maxlen` for fixed-size ring buffers.

```python
from collections import deque

# Sliding window / bounded history with automatic eviction
recent = deque(maxlen=3)
for x in [1, 2, 3, 4, 5]:
    recent.append(x)
print(recent)            # deque([3, 4, 5]) - oldest dropped

# Efficient FIFO queue
queue = deque()
queue.append("task")     # enqueue, O(1)
queue.popleft()          # dequeue, O(1) - a list would be O(n)
```

The `maxlen` deque automatically discards the oldest element on overflow, giving a fixed-memory ring buffer for free — ideal for "last N events" tracking. For queue workloads, `deque.popleft()` is O(1) versus a list's O(n) `pop(0)`, a difference that dominates at scale.

### Q47. Explain `defaultdict`, `Counter`, and `OrderedDict` use cases.

These `collections` specializations encode common patterns. `defaultdict` supplies a default for missing keys via a factory, eliminating `setdefault`/`if-in` boilerplate when grouping or accumulating. `Counter` is a dict subclass for tallying with helpers like `most_common`. `OrderedDict` preserves and exposes insertion-order operations (`move_to_end`, order-sensitive equality); since 3.7 plain dicts keep insertion order, so `OrderedDict` is now reserved for those extra ordering methods (e.g., building an LRU cache).

```python
from collections import defaultdict, Counter

# Grouping without key-existence checks
groups: dict = defaultdict(list)
for word in ["apple", "ant", "bee", "bear"]:
    groups[word[0]].append(word)
# {'a': ['apple', 'ant'], 'b': ['bee', 'bear']}

# Frequency analysis in one line
counts = Counter("mississippi")
print(counts.most_common(2))     # [('s', 4), ('i', 4)]
```

`defaultdict(list)` auto-creates an empty list on first access to a key, turning a four-line grouping idiom into one. `Counter.most_common` ranks by frequency efficiently. Choosing the right specialized container makes intent obvious and code shorter while matching the optimal algorithm for the task.

### Q48. What is the time complexity of common operations on Python's core data structures?

Knowing complexities guides structure choice. `list`: index/append O(1), insert/delete/`pop(0)`/membership O(n). `dict` and `set`: average O(1) insert/lookup/delete (amortized, hash-based), O(n) worst case under pathological collisions. `deque`: O(1) at both ends, O(n) middle access. Sorting is O(n log n). Membership testing is the classic trap — `x in large_list` is O(n) while `x in large_set` is O(1).

```python
# O(n^2): membership test against a list inside a loop
def dedupe_slow(items, seen_list):
    return [x for x in items if x not in seen_list]   # in-list is O(n)

# O(n): use a set for membership
def dedupe_fast(items):
    seen = set()
    result = []
    for x in items:
        if x not in seen:        # O(1) average
            seen.add(x)
            result.append(x)
    return result
```

Swapping the `list` membership check for a `set` collapses an O(n²) algorithm to O(n) — often the single highest-impact optimization in real code. Interviewers probe this because choosing the wrong container is the most common cause of accidental quadratic behavior in otherwise reasonable Python.

### Q49. Explain the Singleton pattern in Python and its idiomatic alternatives.

A Singleton ensures one instance globally. Classic implementations override `__new__` or use a metaclass, but Python's idiom is usually simpler: a *module* is already a singleton (imported once, cached in `sys.modules`), so module-level objects/functions serve the purpose without ceremony. When you do need a class-based singleton (e.g., a connection pool), `__new__` caching or a decorator is clean; beware thread-safety during first creation.

```python
import threading


class Singleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:                  # double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

The double-checked locking pattern avoids paying the lock cost on every instantiation while remaining thread-safe for the one-time creation. Still, prefer a module-level instance or dependency injection where possible — global singletons complicate testing (shared state across tests) and hide dependencies, so reach for the pattern only when truly one instance must exist.

### Q50. Describe the Strategy pattern and its Pythonic form.

The Strategy pattern encapsulates interchangeable algorithms behind a common interface so behavior can be selected at runtime. In classic OOP this means a family of classes; in Python, because functions are first-class objects, a strategy is often just a callable passed as an argument — no class hierarchy needed. This keeps the pattern lightweight while retaining its core benefit: decoupling the client from the concrete algorithm.

```python
from typing import Callable, Sequence

# Strategy as a first-class function
def by_length(s: str) -> int:
    return len(s)

def process(items: Sequence[str], key: Callable[[str], int]) -> list:
    return sorted(items, key=key)        # 'key' IS the strategy

process(["bb", "a", "ccc"], key=by_length)   # ['a', 'bb', 'ccc']
process(["bb", "a", "ccc"], key=str.lower)   # different strategy, no new class
```

Passing `key` as a callable is the Strategy pattern stripped to its essence — `sorted`'s own `key` parameter is a built-in example. This avoids the boilerplate of strategy classes while preserving runtime interchangeability, illustrating how Python's first-class functions subsume many classic GoF patterns.

### Q51. Explain the Factory pattern and `@classmethod` factories.

A Factory centralizes object creation, decoupling clients from concrete classes and allowing the creation logic to choose the right type or perform setup. In Python, alternative constructors implemented as `@classmethod` are the idiomatic factory: they give descriptive named constructors (`from_json`, `from_file`) and, because they use `cls`, naturally produce the correct subclass. A standalone factory function or a registry-based dispatch handles the case of selecting among unrelated classes.

```python
from __future__ import annotations
import json


class Config:
    def __init__(self, settings: dict):
        self.settings = settings

    @classmethod
    def from_json(cls, text: str) -> "Config":
        return cls(json.loads(text))

    @classmethod
    def from_env(cls, prefix: str) -> "Config":
        import os
        return cls({k: v for k, v in os.environ.items() if k.startswith(prefix)})
```

Multiple `@classmethod` constructors give readable, intention-revealing creation paths (`Config.from_json(...)` vs. `Config.from_env(...)`) that a single overloaded `__init__` could not express clearly. Using `cls` rather than `Config` means subclasses inherit working factories that return their own type.

### Q52. What is the Observer pattern and how do you implement it safely?

The Observer pattern lets a subject notify a set of dependents (observers) of state changes without tight coupling. The safety concern is lifetime: if the subject holds strong references to observers, it prevents their garbage collection, leaking memory. Using `weakref` for the observer registry lets observers be collected naturally when no longer needed elsewhere, avoiding the leak while keeping the decoupling.

```python
import weakref
from typing import Callable


class Subject:
    def __init__(self):
        self._observers: weakref.WeakSet = weakref.WeakSet()

    def subscribe(self, observer) -> None:
        self._observers.add(observer)

    def notify(self, event) -> None:
        for obs in self._observers:
            obs.update(event)


class Observer:
    def update(self, event) -> None:
        print(f"received {event}")
```

The `WeakSet` registry holds observers weakly, so when an observer is otherwise unreferenced it is automatically dropped from notifications and collected — eliminating the dangling-listener leak that plagues naive implementations. This is the standard fix for event systems, callbacks, and signal/slot mechanisms in long-lived applications.


### Q53. When should you use `dataclasses` versus `NamedTuple` versus `attrs` versus `pydantic`?

`dataclasses` (stdlib) auto-generate `__init__`, `__repr__`, `__eq__` for mutable or frozen records — the default choice for plain data containers. `NamedTuple` gives immutable, tuple-compatible, memory-light records ideal for fixed return values. `attrs` is a more powerful third-party predecessor with validators and converters. `pydantic` adds runtime type validation and (de)serialization, making it the choice for parsing untrusted external data (API payloads, config). Pick by need: structure only → dataclass; validation → pydantic.

```python
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float
    tags: list = field(default_factory=list)

    def distance_to(self, other: "Point") -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5
```

`frozen=True` makes instances immutable and hashable; `slots=True` (3.10+) removes the per-instance `__dict__` for memory savings. `field(default_factory=list)` correctly gives each instance its own list, avoiding the shared-mutable-default trap. This single decorator replaces dozens of lines of boilerplate while remaining pure stdlib.

### Q54. Explain the difference between `__eq__`, `__hash__`, and their contract.

`__eq__` defines value equality; `__hash__` returns an integer used to place objects in hash tables (`dict`, `set`). The contract: equal objects *must* have equal hashes (the converse need not hold), and an object's hash must remain constant for its lifetime — hence only immutable objects should be hashable. Defining `__eq__` without `__hash__` makes the class unhashable (Python sets `__hash__ = None`) to prevent inconsistent table behavior.

```python
class Money:
    def __init__(self, amount: int, currency: str):
        self.amount = amount
        self.currency = currency

    def __eq__(self, other) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __hash__(self) -> int:
        return hash((self.amount, self.currency))   # consistent with __eq__
```

`__hash__` hashes the same fields `__eq__` compares, satisfying the contract so equal `Money` objects collide into the same bucket and behave correctly in sets/dicts. Returning `NotImplemented` (not `False`) for foreign types lets Python try the reflected operation. Violating the equal-implies-equal-hash rule causes silent, baffling lookup failures.

### Q55. What are abstract base classes (ABCs) and how do they enable structural typing?

ABCs (`abc.ABC`, `@abstractmethod`) define interfaces that subclasses must implement, raising `TypeError` on instantiation if abstract methods are unimplemented — enforcing a contract at construction time. They also support *virtual subclassing* (`register`) and `__subclasshook__`, enabling `isinstance` checks based on structure rather than inheritance. The `collections.abc` module provides ready-made ABCs (`Iterable`, `Mapping`, `Sequence`) for duck-typing checks.

```python
from abc import ABC, abstractmethod


class Repository(ABC):
    @abstractmethod
    def get(self, id: int) -> dict:
        ...

    @abstractmethod
    def save(self, entity: dict) -> None:
        ...


class InMemoryRepo(Repository):
    def __init__(self):
        self._data: dict = {}

    def get(self, id: int) -> dict:
        return self._data[id]

    def save(self, entity: dict) -> None:
        self._data[entity["id"]] = entity
```

Because `Repository` declares both methods abstract, forgetting to implement one makes `InMemoryRepo()` fail at instantiation rather than at first use — moving errors to the earliest possible point. This formalizes the dependency-inversion principle: high-level code depends on the `Repository` abstraction, enabling swappable implementations and easy test doubles.

### Q56. Explain `typing.Protocol` and structural subtyping.

`Protocol` (PEP 544) enables static duck typing: a class satisfies a protocol if it has the required methods/attributes, *without* explicitly inheriting from it. This matches Python's runtime duck-typing philosophy while giving static type checkers something to verify. Mark a protocol `@runtime_checkable` to also allow `isinstance` checks (structural, method-presence only). Protocols decouple interface from implementation more loosely than ABCs.

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None:
        ...


def cleanup(resource: Closeable) -> None:
    resource.close()


class Connection:           # does NOT inherit Closeable
    def close(self) -> None:
        print("closed")


cleanup(Connection())       # type-checks: structural match
```

`Connection` satisfies `Closeable` purely by having a matching `close` method — no inheritance, no registration. This lets you type-annotate functions against the behavior they actually need (the interface-segregation principle) and apply interfaces retroactively to third-party classes you cannot modify, which ABCs cannot do without explicit registration.

### Q57. What is the difference between composition and inheritance, and when to prefer each?

Inheritance models an "is-a" relationship and reuses behavior by subclassing, but it tightly couples subclass to superclass internals and can produce fragile, deep hierarchies. Composition models "has-a" by holding other objects and delegating to them, yielding looser coupling and runtime flexibility. The guideline "prefer composition over inheritance" reflects that composition avoids the fragile-base-class problem and combinatorial subclass explosion; reserve inheritance for genuine subtype polymorphism.

```python
# Composition: behavior assembled from injected collaborators
class Logger:
    def log(self, msg: str) -> None:
        print(f"[LOG] {msg}")


class OrderService:
    def __init__(self, logger: Logger, repo):
        self._logger = logger      # has-a, not is-a
        self._repo = repo

    def place(self, order: dict) -> None:
        self._repo.save(order)
        self._logger.log(f"placed {order['id']}")
```

`OrderService` composes a logger and repository rather than inheriting from them, so each collaborator can be swapped (e.g., a silent logger in tests) without touching the service. This dependency injection makes the design testable and flexible, illustrating why composition is the default and inheritance the exception in modern Python design.

### Q58. Explain the `__getattr__`, `__getattribute__`, and `__setattr__` hooks.

`__getattribute__` is called for *every* attribute access and is the low-level interceptor (override with care — easy to cause infinite recursion). `__getattr__` is called only as a *fallback* when normal lookup fails, making it the safe hook for dynamic/proxy attributes. `__setattr__` intercepts all attribute assignment. These enable proxies, lazy attributes, and attribute-style access to dynamic data.

```python
class LazyConfig:
    def __init__(self, loader):
        # bypass __setattr__ recursion via the base implementation
        object.__setattr__(self, "_loader", loader)
        object.__setattr__(self, "_cache", {})

    def __getattr__(self, name: str):           # only on miss
        if name not in self._cache:
            self._cache[name] = self._loader(name)
        return self._cache[name]
```

`__getattr__` fires only when an attribute is not found normally, so cached values (stored in `_cache`) are returned through the fast path and only uncached names trigger the loader — implementing lazy, on-demand loading. Using `object.__setattr__` in `__init__` avoids recursion that a naive `self._cache = {}` would cause if `__setattr__` were also overridden.


### Q59. How do you implement an LRU cache from scratch?

An LRU (Least Recently Used) cache evicts the entry unused for the longest time when at capacity. The classic implementation pairs a hash map (O(1) lookup) with a doubly linked list (O(1) reordering) — but in Python, `OrderedDict` provides both: `move_to_end` reorders in O(1) and `popitem(last=False)` evicts the oldest. This gives an interview-friendly, correct LRU in a few lines.

```python
from collections import OrderedDict
from typing import Any, Optional


class LRUCache:
    def __init__(self, capacity: int):
        self._cap = capacity
        self._data: OrderedDict = OrderedDict()

    def get(self, key: Any) -> Optional[Any]:
        if key not in self._data:
            return None
        self._data.move_to_end(key)        # mark as most-recently used
        return self._data[key]

    def put(self, key: Any, value: Any) -> None:
        if key in self._data:
            self._data.move_to_end(key)
        self._data[key] = value
        if len(self._data) > self._cap:
            self._data.popitem(last=False)  # evict least-recently used
```

`move_to_end` on every access keeps the ordering as a recency list, and `popitem(last=False)` removes the front (oldest) entry in O(1). This leverages `OrderedDict`'s linked-list backing to achieve the textbook O(1) LRU without manually maintaining pointers — a frequent whiteboard exercise.

### Q60. What is the difference between `sorted` with a `key` versus `cmp`-style comparison, and `functools.cmp_to_key`?

Python 3 removed the `cmp` parameter in favor of `key` functions, because a key (computing a comparison proxy once per element) is O(n) calls versus a comparator's O(n log n) calls — faster and simpler. When you genuinely need pairwise comparison logic (e.g., custom ordering not reducible to a single key), `functools.cmp_to_key` adapts an old-style comparator into a key function. Prefer `key` whenever the ordering can be expressed as a derived value.

```python
import functools

# Preferred: key function, called once per element
people = [{"name": "Ann", "age": 30}, {"name": "Bob", "age": 25}]
people.sort(key=lambda p: (p["age"], p["name"]))   # multi-level sort

# When only a comparator makes sense:
def compare(a, b) -> int:
    return (a % 10) - (b % 10)      # by last digit

nums = sorted([23, 11, 35, 42], key=functools.cmp_to_key(compare))
```

The tuple key `(age, name)` expresses a multi-level sort declaratively and runs each key computation once. `cmp_to_key` is the escape hatch for irreducibly relational comparisons, but it is slower, so it should be a last resort after confirming the order cannot be captured by a key. This trade-off is a common deep-dive in interviews.

### Q61. Explain bisect and how to maintain a sorted collection efficiently.

The `bisect` module performs binary search (O(log n)) on a sorted list to find insertion points, and `insort` inserts while keeping order. It is the right tool for maintaining a sorted sequence with frequent lookups and occasional inserts, or for range queries and "find nearest" operations — far better than re-sorting after each insert. Note the insert itself is still O(n) due to element shifting, so for insert-heavy workloads a tree/heap may be better.

```python
import bisect

scores = [60, 70, 75, 85, 90]

# O(log n) lookup: how many scores are below 80?
idx = bisect.bisect_left(scores, 80)        # 3

# Maintain sorted order on insert
bisect.insort(scores, 78)                    # [60, 70, 75, 78, 85, 90]

# Map a value to a grade band via boundaries
def grade(score: int, breakpoints=(60, 70, 80, 90), grades="FDCBA") -> str:
    return grades[bisect.bisect(breakpoints, score)]
```

`bisect_left` locates the boundary in logarithmic time without scanning, and the grade-band trick maps a continuous value to a category with one binary search — a clean, fast pattern for bucketing. Choosing `bisect` over linear scans or repeated sorting is a hallmark of complexity-aware Python.

### Q62. What is a heap, and when do you use `heapq`?

A heap is a binary tree maintaining the heap invariant (parent ≤ children for a min-heap), giving O(1) access to the smallest element and O(log n) push/pop. `heapq` implements a min-heap over a plain list. Use it for priority queues, scheduling, merging sorted streams, and "top-K" problems where you need repeated access to extremes without fully sorting. For top-K, a bounded heap is O(n log k) versus O(n log n) for a full sort.

```python
import heapq
from typing import Iterable

# Top-K largest using a bounded min-heap: O(n log k)
def top_k(items: Iterable[int], k: int) -> list:
    heap: list = []
    for x in items:
        if len(heap) < k:
            heapq.heappush(heap, x)
        elif x > heap[0]:                 # compare to smallest in heap
            heapq.heapreplace(heap, x)    # pop min, push x: one operation
    return sorted(heap, reverse=True)


# Priority queue with (priority, item) tuples
pq: list = []
heapq.heappush(pq, (2, "low"))
heapq.heappush(pq, (0, "urgent"))
heapq.heappop(pq)                          # (0, 'urgent')
```

The bounded heap keeps only the k largest seen so far, so memory is O(k) and time O(n log k) — superior to sorting everything when k ≪ n. `heapreplace` fuses a pop and push into one sift, halving the work versus separate calls. For max-heaps, negate values or use tuples, since `heapq` is min-only.

### Q63. Explain the Adapter and Facade patterns with Python examples.

An Adapter wraps an incompatible interface to match the one a client expects (interface translation); a Facade provides a single simplified interface over a complex subsystem (interface unification). Both reduce coupling, but Adapter targets *compatibility* between two specific interfaces, while Facade targets *simplicity* for many. In Python, both are typically lightweight wrapper classes that delegate.

```python
# Adapter: make a third-party logger fit our expected interface
class ThirdPartyLogger:
    def write_log(self, level: str, text: str) -> None: ...

class LoggerAdapter:
    def __init__(self, adaptee: ThirdPartyLogger):
        self._adaptee = adaptee

    def info(self, msg: str) -> None:                 # our interface
        self._adaptee.write_log("INFO", msg)          # translate


# Facade: one method hiding a multi-step subsystem
class OrderFacade:
    def __init__(self, inventory, payment, shipping):
        self._inv, self._pay, self._ship = inventory, payment, shipping

    def checkout(self, cart) -> str:
        self._inv.reserve(cart)
        self._pay.charge(cart)
        return self._ship.dispatch(cart)
```

The `LoggerAdapter` exposes our `info()` method while delegating to the third party's differently-named `write_log`, letting us swap logging backends without changing callers. The `OrderFacade.checkout` collapses a three-subsystem dance into one call, shielding clients from orchestration details. Both keep client code stable as implementations evolve.


### Q64. How do generators enable memory-efficient data pipelines?

Chaining generators builds a pull-based pipeline where each stage processes one item at a time and passes it downstream, so the entire dataset is never held in memory at once. Because generators are lazy, no work happens until the final consumer iterates, and only as much as the consumer requests. This composes complex transformations over arbitrarily large (even infinite) data streams with constant memory.

```python
from typing import Iterator
import csv


def read_rows(path: str) -> Iterator[dict]:
    with open(path) as fh:
        yield from csv.DictReader(fh)

def filter_active(rows: Iterator[dict]) -> Iterator[dict]:
    return (r for r in rows if r["status"] == "active")

def to_summary(rows: Iterator[dict]) -> Iterator[str]:
    return (f"{r['name']}: {r['score']}" for r in rows)


# Pipeline: each item flows through all stages lazily
pipeline = to_summary(filter_active(read_rows("data.csv")))
for line in pipeline:           # one row in memory at a time
    print(line)
```

Each stage is a generator that yields transformed items on demand, so processing a multi-gigabyte CSV uses memory proportional to a single row, not the file. The composition reads top-to-bottom yet executes lazily from the consumer, embodying the Unix-pipe philosophy in Python — a powerful, scalable alternative to loading everything into lists.

### Q65. What is the difference between `itertools.chain`, `islice`, `groupby`, and `tee`?

`itertools` provides lazy iterator building blocks. `chain` concatenates iterables into one stream; `islice` takes a lazy slice (start/stop/step) without indexing; `groupby` groups *consecutive* equal-key elements (so input must be pre-sorted by that key); `tee` splits one iterator into several independent ones (buffering as needed). They enable expressive, memory-light stream processing without intermediate lists.

```python
import itertools

# chain: flatten multiple sources lazily
combined = itertools.chain([1, 2], [3, 4], [5])      # 1,2,3,4,5

# islice: take first 3 of an infinite generator
first_three = list(itertools.islice(itertools.count(10), 3))   # [10,11,12]

# groupby: requires sorted input on the key
data = [("a", 1), ("a", 2), ("b", 3)]
for key, group in itertools.groupby(data, key=lambda t: t[0]):
    print(key, [t[1] for t in group])     # a [1,2]  /  b [3]
```

`groupby`'s consecutive-only semantics is the classic gotcha — unsorted input yields fragmented groups, so sorting by the same key first is mandatory. `islice` lets you safely consume part of an infinite generator like `count`. These tools replace manual buffering loops with declarative, lazy composition, keeping memory flat over large streams.

### Q66. Explain the Decorator design pattern (structural) versus Python's `@decorator` syntax.

These share a name but differ. The structural Decorator *pattern* wraps an object to add responsibilities dynamically while preserving its interface, enabling layered behavior (e.g., stacking compression over encryption over a file stream). Python's `@decorator` *syntax* is a language feature for wrapping functions/classes at definition time. The pattern is about runtime object composition; the syntax is about call-time function transformation — they solve related but distinct problems.

```python
from typing import Protocol


class DataSource(Protocol):
    def read(self) -> str: ...
    def write(self, data: str) -> None: ...


class FileSource:
    def read(self) -> str: return "raw"
    def write(self, data: str) -> None: ...


class EncryptedDecorator:                 # structural Decorator pattern
    def __init__(self, wrapped: DataSource):
        self._wrapped = wrapped

    def read(self) -> str:
        return self._decrypt(self._wrapped.read())

    def write(self, data: str) -> None:
        self._wrapped.write(self._encrypt(data))

    def _encrypt(self, s): return s[::-1]
    def _decrypt(self, s): return s[::-1]
```

`EncryptedDecorator` wraps any `DataSource` and adds encryption transparently — and because it implements the same interface, decorators can be stacked (compression around encryption around the file) at runtime. This composition-based extension is distinct from the `@` syntax, though both express "wrap to augment." Distinguishing them signals depth in both design patterns and language features.

### Q67. How do you design an immutable value object in Python?

An immutable value object has no observable state changes after construction, equality based on value, and is hashable. Achieve it with a frozen dataclass or `NamedTuple`, ensuring any contained collections are themselves immutable (tuples, `frozenset`) — a frozen dataclass holding a mutable list is not truly immutable. Immutability brings thread-safety, safe sharing/caching, and usability as dict keys.

```python
from dataclasses import dataclass
from typing import Tuple


@dataclass(frozen=True)
class Color:
    r: int
    g: int
    b: int

    def lighten(self, amount: int) -> "Color":
        # return a NEW object instead of mutating
        clamp = lambda v: min(255, v + amount)
        return Color(clamp(self.r), clamp(self.g), clamp(self.b))


c = Color(10, 20, 30)
c2 = c.lighten(50)        # c unchanged; c2 is a new value
# c.r = 99   -> FrozenInstanceError
```

`frozen=True` blocks attribute assignment and auto-generates `__hash__`, so `Color` is safely shareable and hashable. Mutating operations like `lighten` return new instances rather than modifying in place — the functional style that makes reasoning about state trivial and eliminates a whole class of aliasing bugs in concurrent code.

---

## 5. Testing and Profiling

### Q68. What are pytest fixtures and how does dependency injection work in pytest?

Fixtures are functions decorated with `@pytest.fixture` that produce reusable test dependencies (data, connections, configured objects). A test requests a fixture by naming it as a parameter; pytest resolves and injects it — a form of dependency injection. Fixtures can depend on other fixtures, and a `yield` fixture provides setup before the yield and teardown after, giving clean, composable resource management.

```python
import pytest


@pytest.fixture
def db_connection():
    conn = create_connection(":memory:")
    conn.execute("CREATE TABLE users (id INT, name TEXT)")
    yield conn                      # provide to the test
    conn.close()                    # teardown runs after the test


def test_insert_user(db_connection):       # injected by name
    db_connection.execute("INSERT INTO users VALUES (1, 'Ann')")
    row = db_connection.execute("SELECT name FROM users WHERE id=1").fetchone()
    assert row[0] == "Ann"
```

The test declares `db_connection` as a parameter and pytest supplies a freshly set-up connection, then runs the post-`yield` teardown afterward — no manual setup/teardown methods. This injection model keeps tests focused on behavior, makes dependencies explicit, and composes: a fixture can build on others to assemble complex test environments declaratively.

### Q69. Explain fixture scopes and the `conftest.py` mechanism.

A fixture's `scope` controls how often it is created and torn down: `function` (default, per test), `class`, `module`, `package`, or `session` (once per test run). Wider scopes amortize expensive setup (a database, a server) across many tests. `conftest.py` is a special file pytest auto-discovers; fixtures defined there are available to all tests in that directory tree without imports, enabling shared test infrastructure.

```python
# conftest.py - fixtures here are visible to the whole test tree
import pytest


@pytest.fixture(scope="session")
def app_server():
    server = start_test_server()        # expensive: do once per run
    yield server
    server.shutdown()


@pytest.fixture(scope="function")
def clean_state(app_server):            # fresh per test, reuses the server
    app_server.reset()
    return app_server
```

The `session`-scoped `app_server` starts one server for the entire run (avoiding per-test startup cost), while the `function`-scoped `clean_state` resets it before each test for isolation — combining efficiency with independence. Placing these in `conftest.py` makes them globally available, the idiomatic way to share heavy fixtures across a suite.

### Q70. What is the difference between `Mock`, `MagicMock`, and `patch`?

`Mock` is a flexible stand-in that records calls and returns configurable values; `MagicMock` is a `Mock` that additionally implements magic/dunder methods (so it supports `len()`, iteration, context managers, etc.). `patch` temporarily replaces a target object with a mock for the duration of a test, restoring it afterward. The cardinal rule: patch where the name is *looked up*, not where it is defined.

```python
from unittest.mock import MagicMock, patch


# Patch where 'requests' is used, not where it's defined
@patch("myapp.service.requests")
def test_fetch(mock_requests):
    mock_requests.get.return_value.json.return_value = {"ok": True}
    from myapp.service import fetch_data
    assert fetch_data("http://x") == {"ok": True}
    mock_requests.get.assert_called_once_with("http://x")
```

The patch target `myapp.service.requests` is the *usage site* — patching `requests.get` directly would miss the reference already imported into the service module. `MagicMock`'s chained `return_value` mocks the fluent `requests.get(...).json()` call. `assert_called_once_with` verifies the interaction, testing behavior at the boundary without real network access.


### Q71. What is the difference between a mock, a stub, a fake, and a spy?

These are test double types. A *stub* returns canned answers to calls. A *mock* additionally has expectations and verifies how it was called (interaction testing). A *fake* is a working but simplified implementation (e.g., an in-memory database). A *spy* wraps a real object, delegating to it while recording the interactions. Choosing the right double matters: over-mocking couples tests to implementation; fakes test more realistically.

```python
from unittest.mock import Mock

# Stub: just returns a value
stub_clock = Mock()
stub_clock.now.return_value = "2025-01-01"

# Mock: verifies interaction
notifier = Mock()
service.process(order, notifier)
notifier.send.assert_called_once_with(order.id)   # expectation check

# Fake: real behavior, simplified
class FakeRepo:
    def __init__(self): self._store = {}
    def save(self, e): self._store[e["id"]] = e
    def get(self, i): return self._store[i]
```

The stub supplies a fixed time without verification; the mock asserts the notifier was called correctly; the `FakeRepo` provides genuine save/get behavior in memory. Preferring fakes for collaborators with rich behavior (repositories, caches) yields tests that survive refactoring, while reserving strict mocks for verifying important side effects keeps tests meaningful rather than brittle.

### Q72. How does `pytest.mark.parametrize` improve test design?

`parametrize` runs the same test function across multiple input/expected pairs, each reported as a separate test case. This eliminates duplicated test bodies, makes edge cases explicit as data, and pinpoints exactly which input failed. It embodies table-driven testing — adding a case is adding a row, not copying a function — improving coverage breadth with minimal code.

```python
import pytest


@pytest.mark.parametrize("value, expected", [
    (0, "zero"),
    (1, "positive"),
    (-1, "negative"),
    (1_000_000, "positive"),     # edge: large
])
def test_classify(value, expected):
    assert classify(value) == expected
```

Each tuple becomes an independently reported test, so a failure for `-1` is identified by name without obscuring the passing cases. Expanding coverage is just appending rows, which keeps edge cases visible and the test body single-sourced. This data-driven structure scales far better than writing one function per scenario and is a sign of mature test design.

### Q73. What is property-based testing and how does Hypothesis work?

Property-based testing asserts *properties* that should hold for all inputs, then a framework (Hypothesis) generates many varied inputs to try to falsify them — including edge cases humans forget (empty, huge, unicode, boundary numbers). On failure it *shrinks* the input to a minimal reproducing example. It complements example-based tests by exploring the input space systematically rather than relying on hand-picked cases.

```python
from hypothesis import given, strategies as st


@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)        # property

@given(st.lists(st.integers()))
def test_sort_preserves_length(xs):
    assert len(sorted(xs)) == len(xs)
```

Instead of asserting a single example, these tests state invariants — sorting twice equals sorting once, and length is preserved — that Hypothesis checks against hundreds of generated lists, surfacing inputs you would never enumerate manually. When a property fails, Hypothesis reports the smallest counterexample (e.g., `[0]`), making the bug trivial to reproduce. This finds edge-case defects that example tests miss.

### Q74. How do you test asynchronous code?

Async tests need an event loop and a runner that awaits coroutine test functions. Use `pytest-asyncio` (mark tests `@pytest.mark.asyncio`) or `anyio`. Mock async dependencies with `AsyncMock` (whose calls return awaitables). Be careful to actually `await` the code under test and to avoid mixing blocking calls that would stall the test loop.

```python
import pytest
from unittest.mock import AsyncMock


@pytest.mark.asyncio
async def test_fetch_user():
    client = AsyncMock()
    client.get.return_value = {"id": 1, "name": "Ann"}

    result = await fetch_user(client, user_id=1)    # await the coroutine

    assert result["name"] == "Ann"
    client.get.assert_awaited_once_with("/users/1")  # async-aware assertion
```

`AsyncMock` makes `client.get(...)` awaitable so it can stand in for a real async client, and `assert_awaited_once_with` verifies the coroutine was actually awaited (not merely called). Marking the test `asyncio` lets pytest drive it on an event loop. The key disciplines are awaiting the subject and using async-aware mocks/assertions rather than their synchronous counterparts.

### Q75. Explain code coverage metrics and their limitations.

Coverage measures which code executes during tests: *line* coverage (statements run), *branch* coverage (both sides of each conditional taken), and stricter forms. It identifies untested code, but high coverage does not imply correctness — it shows code *ran*, not that meaningful assertions verified its behavior. Treat coverage as a floor and a gap-finder, never as a quality goal to maximize blindly.

```bash
# Measure branch coverage and fail under a threshold
pytest --cov=myapp --cov-branch --cov-report=term-missing --cov-fail-under=85
```

```python
# 100% line coverage but a missing branch test:
def divide(a, b):
    if b == 0:
        return None        # is this branch actually tested?
    return a / b
# A test calling divide(4, 2) gives line coverage but never exercises b == 0.
```

`--cov-branch` catches exactly the gap above — the `b == 0` path may be unexecuted even at full line coverage, so branch coverage and `--cov-report=term-missing` reveal it. Enforcing a threshold in CI prevents regressions, but the deeper point is that coverage guides where to add *assertive* tests; it cannot certify that the assertions present are adequate.


### Q76. How do you profile CPU-bound Python code? Compare `cProfile`, `line_profiler`, and `py-spy`.

`cProfile` (stdlib) gives deterministic function-level timing — call counts and cumulative/total time per function — with low setup but some overhead. `line_profiler` (third-party `@profile`) drills down to per-line timing within chosen functions, ideal once you've localized a hotspot. `py-spy` is a sampling profiler that attaches to a running process *without* code changes or restarts, with negligible overhead — perfect for production. The workflow: `cProfile` to find the hot function, `line_profiler` to find the hot line.

```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

result = run_workload()

profiler.disable()
stats = pstats.Stats(profiler).sort_stats("cumulative")
stats.print_stats(10)        # top 10 functions by cumulative time
```

Sorting by `cumulative` time surfaces the functions where the program spends the most wall time including callees, directing optimization effort to what matters. For a live service you cannot instrument, `py-spy dump --pid <PID>` or `py-spy record` samples the stack externally. Matching the profiler to the situation — deterministic vs. sampling, function vs. line — is the practical skill interviewers look for.

### Q77. How do you profile memory usage in Python?

Use `tracemalloc` (stdlib) for allocation tracking and snapshot diffing by line; `memory_profiler` for per-line memory deltas of a function; and `objgraph`/`guppy` for inspecting object counts and reference chains. For a growing-RSS investigation, snapshot before and after the suspect operation and compare — this localizes which lines allocate the most retained memory, distinguishing genuine growth from transient spikes.

```python
import tracemalloc

tracemalloc.start(25)        # keep 25 frames of traceback per allocation

baseline = tracemalloc.take_snapshot()
build_large_structure()
current = tracemalloc.take_snapshot()

for stat in current.compare_to(baseline, "traceback")[:5]:
    print(stat)
    for line in stat.traceback.format():
        print("  ", line)    # full allocation stack
```

Capturing 25 frames lets `compare_to` attribute growth to a full call stack, not just one line — invaluable when allocations happen deep inside library calls. Diffing snapshots isolates the operation's net retained memory. This pinpoints leaks and over-allocation precisely, which guesswork or `top` alone cannot, and pairs naturally with the leak-diagnosis approach from Q40.

### Q78. What is the difference between wall-clock time, CPU time, and `timeit` for micro-benchmarks?

Wall-clock time (`time.perf_counter`) measures elapsed real time including I/O and scheduling; CPU time (`time.process_time`) measures only time the CPU spent on your process, excluding sleeps/waits. For micro-benchmarks, `timeit` is the right tool: it runs the snippet many times, disables the GC during timing by default, and reports the best run — controlling for noise that a single `perf_counter` measurement would include. Never benchmark with a single timed run.

```python
import timeit

# Compare two approaches fairly over many iterations
list_comp = timeit.timeit("[x*x for x in range(1000)]", number=10_000)
map_call = timeit.timeit(
    "list(map(lambda x: x*x, range(1000)))", number=10_000
)
print(f"comprehension: {list_comp:.4f}s   map: {map_call:.4f}s")
```

`timeit` repeats each snippet 10,000 times and isolates timing from GC pauses, so the comparison reflects the code's intrinsic cost rather than one-off scheduling noise. Choosing `perf_counter` vs. `process_time` depends on whether you care about total latency (includes waiting) or pure computation. Reporting the minimum (not mean) is standard, since the fastest run best approximates the noise-free cost.

### Q79. How do you optimize a CPU-bound hotspot once profiling identifies it?

After profiling localizes the hotspot, the optimization ladder is: pick a better algorithm/data structure first (largest wins), then reduce work (caching, avoiding recomputation), then leverage vectorized libraries (NumPy) that run in C, then move to compiled extensions (Cython, `numba`, Rust via PyO3) or true parallelism (`multiprocessing`). Micro-optimizing Python syntax is the last resort with the smallest payoff. Always re-profile to confirm the change helped.

```python
import numpy as np

# Pure-Python hotspot: element-wise math in a loop (slow)
def scale_slow(data, factor):
    return [x * factor + 1 for x in data]

# Vectorized: the loop runs in C, releasing the GIL
def scale_fast(data: np.ndarray, factor: float) -> np.ndarray:
    return data * factor + 1        # one vectorized op over the whole array
```

Replacing the Python-level loop with a NumPy vectorized expression pushes the iteration into optimized C (often SIMD-accelerated), typically yielding 10–100× speedups for numeric work — far more than any syntactic tweak. The broader lesson is to climb the ladder from algorithm to vectorization to compilation, measuring at each step, rather than prematurely reaching for micro-optimizations.

### Q80. What is `__debug__` and how do `assert` statements behave in production?

`__debug__` is a built-in constant that is `True` normally and `False` when Python runs with `-O` (optimize). Crucially, `assert` statements are *removed entirely* under `-O`. Therefore `assert` must only express internal invariants and debugging checks — never validate external input, enforce security, or perform side effects, because all of that vanishes in optimized runs. Use explicit `if ... raise` for production validation.

```python
def withdraw(balance: float, amount: float) -> float:
    # WRONG: this check disappears under python -O
    # assert amount > 0, "amount must be positive"

    # CORRECT: real validation always runs
    if amount <= 0:
        raise ValueError("amount must be positive")
    if amount > balance:
        raise ValueError("insufficient funds")
    return balance - amount
```

Because `-O` strips asserts, relying on them for input validation creates a silent security/correctness hole in optimized deployments. The explicit `if ... raise` always executes, making it the correct tool for validating untrusted input and enforcing business rules. Reserve `assert` for "this should be impossible" internal sanity checks during development.


---

## 6. Cross-Cutting Advanced Topics

### Q81. Explain exception chaining with `raise ... from ...`.

Exception chaining records the relationship between exceptions. When you catch one error and raise another, `raise New() from original` sets `__cause__`, producing a traceback that explicitly shows "the above exception was the direct cause." Implicit chaining (`__context__`) happens automatically when an exception is raised during handling of another. Use explicit `from` to translate low-level errors into domain errors while preserving the root cause for debugging; use `from None` to deliberately suppress an irrelevant context.

```python
class ConfigError(Exception):
    pass


def load_config(path: str) -> dict:
    try:
        with open(path) as fh:
            return parse(fh.read())
    except FileNotFoundError as exc:
        raise ConfigError(f"config missing: {path}") from exc   # preserve cause
    except ValueError:
        raise ConfigError("config is malformed") from None      # hide noise
```

`from exc` keeps the original `FileNotFoundError` visible in the traceback, so a caller sees both the domain-level `ConfigError` and its underlying cause — invaluable for diagnosis. `from None` suppresses the chained context when the low-level error (a parse `ValueError`) would only add noise. This gives clean, meaningful error hierarchies that translate implementation details into a stable API of exceptions.

### Q82. What are `ExceptionGroup` and `except*`?

`ExceptionGroup` (Python 3.11) bundles multiple exceptions raised concurrently — notably from `asyncio.TaskGroup` where several tasks may fail at once. The `except*` syntax matches and handles exceptions *by type within a group*, processing all matching ones and re-raising the rest as a residual group. This solves the long-standing problem that a single `except` could only surface one of several simultaneous failures.

```python
import asyncio


async def main():
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(may_raise_value())
            tg.create_task(may_raise_io())
    except* ValueError as eg:
        for exc in eg.exceptions:
            log.error("value error: %s", exc)
    except* OSError as eg:
        log.error("io errors: %d", len(eg.exceptions))
```

When both tasks fail, the `TaskGroup` raises an `ExceptionGroup` containing both; `except* ValueError` handles the value errors and `except* OSError` handles the I/O errors independently, rather than losing one to the other. This is essential for structured concurrency, where partial failure across many tasks is the norm and each failure category may need distinct handling.

### Q83. Explain the walrus operator `:=` and good use cases.

The assignment expression `:=` assigns and returns a value within an expression, letting you bind a name where a statement was previously needed. Good uses: capturing a value used in a loop/comprehension condition, avoiding a redundant call, or simplifying read-and-test loops. Overuse harms readability, so reserve it for cases where it genuinely removes duplication or an extra line without obscuring intent.

```python
# Read-process loop without a duplicated read or a while-True/break dance
def read_chunks(stream, size: int):
    while (chunk := stream.read(size)):
        process(chunk)

# Reuse an expensive computation inside a comprehension
results = [y for x in data if (y := expensive(x)) is not None]
```

In the loop, `:=` reads into `chunk` and tests it in one place, eliminating the classic "read before loop, read again at the end" duplication. In the comprehension, it computes `expensive(x)` once and both filters on and collects the result — without walrus you'd compute it twice or fall back to a loop. The win is removing redundancy; the risk is cramming too much into one expression.

### Q84. How does Python's import system work, and what are the dangers of circular imports?

Importing executes a module's top-level code once and caches the module object in `sys.modules`; subsequent imports return the cache. A circular import (A imports B, B imports A) is dangerous because when the cycle re-enters a partially-initialized module, names defined later in that module don't yet exist, causing `ImportError` or `AttributeError`. Fixes: restructure to break the cycle, defer the import inside a function, or import the module (not its names) so binding is late.

```python
# module a.py
import b                     # safe: import the module object

def use_b():
    return b.helper()        # 'b.helper' resolved at call time, not import time

# module b.py
import a

def helper():
    return a.something()
```

Importing the *module* (`import b`) rather than a name (`from b import helper`) defers attribute lookup to call time, by which point both modules are fully initialized — sidestepping the partial-initialization trap. The deeper remedy is architectural: a circular import usually signals that two modules share a responsibility that belongs in a third, lower-level module.

### Q85. What is the difference between `__repr__` and `__str__`?

`__repr__` is the unambiguous, developer-facing representation — ideally valid Python that recreates the object — and is the fallback shown in the REPL, debuggers, and containers. `__str__` is the readable, user-facing representation used by `print` and `str()`. If only `__repr__` is defined, `str()` falls back to it; the reverse is not true. Always define `__repr__`; define `__str__` only when a distinct human-friendly form adds value.

```python
class Temperature:
    def __init__(self, celsius: float):
        self.celsius = celsius

    def __repr__(self) -> str:
        return f"Temperature(celsius={self.celsius!r})"   # recreatable

    def __str__(self) -> str:
        return f"{self.celsius}°C"                         # for humans


t = Temperature(21.5)
repr(t)   # 'Temperature(celsius=21.5)'  -> shown in lists, debugger
str(t)    # '21.5°C'                      -> shown by print
```

The `__repr__` produces a string that, pasted into a REPL, would reconstruct the object — exactly what you want when inspecting a list of these in a debugger. `__str__` gives the clean display for end users. Defining `__repr__` first is a strong habit because it improves every debugging session, logging line, and collection display by default.

### Q86. Explain `functools.singledispatch` and generic functions.

`@singledispatch` creates a generic function that dispatches to different implementations based on the type of the *first* argument — a clean, extensible alternative to a chain of `isinstance` checks. New types register their own implementation with `@func.register`, even from other modules, achieving open extensibility (the open/closed principle) without modifying the original function. `singledispatchmethod` provides the same for methods.

```python
from functools import singledispatch


@singledispatch
def serialize(value) -> str:
    raise TypeError(f"cannot serialize {type(value).__name__}")


@serialize.register
def _(value: int) -> str:
    return f"int:{value}"


@serialize.register
def _(value: list) -> str:
    return "[" + ",".join(serialize(v) for v in value) + "]"


serialize(42)         # 'int:42'
serialize([1, 2])     # '[int:1,int:2]'
```

Each type's handling lives in its own registered function, so adding support for a new type is adding a registration — never editing a growing `if/elif isinstance` ladder. Dispatch is by the runtime type of the first argument, keeping each implementation focused and the system extensible from outside. This is Python's idiomatic answer to type-based polymorphism for functions.

### Q87. What are type variables, generics, and variance in Python typing?

`TypeVar` parameterizes generic functions and classes so they preserve type relationships (a `Stack[int]` yields `int`s). Bounds (`TypeVar("T", bound=Base)`) constrain the type; variance controls subtyping of the generic itself: covariant (`covariant=True`) for producers/read-only, contravariant for consumers, invariant (default) for mutable containers. Correct variance prevents unsound assignments that type checkers must reject. Python 3.12 introduced cleaner `class Stack[T]:` syntax.

```python
from typing import TypeVar, Generic

T = TypeVar("T")


class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()


s: Stack[int] = Stack()
s.push(1)
value: int = s.pop()        # checker knows this is int
```

The `TypeVar` `T` links `push`'s input to `pop`'s output, so a `Stack[int]` is statically known to return `int`s — catching type errors before runtime. `Stack` is invariant because it is mutable: allowing `Stack[int]` where `Stack[object]` is expected would let a caller push a non-int, breaking the original holder. Understanding variance is what separates correct generic APIs from subtly unsound ones.

### Q88. How do you implement a custom iterable versus a custom iterator?

An *iterable* implements `__iter__` returning a fresh *iterator*; an *iterator* implements `__iter__` (returning self) and `__next__` (producing values, raising `StopIteration` when done). Separating them is important: making a class its own iterator means it can only be iterated once and not concurrently, whereas returning a new iterator from `__iter__` allows independent, repeatable iteration. The cleanest implementation makes `__iter__` a generator.

```python
class NumberRange:
    def __init__(self, start: int, stop: int):
        self.start = start
        self.stop = stop

    def __iter__(self):
        # generator => a fresh, independent iterator each time
        current = self.start
        while current < self.stop:
            yield current
            current += 1


r = NumberRange(0, 3)
list(r)        # [0, 1, 2]
list(r)        # [0, 1, 2] again — repeatable, independent
```

Because `__iter__` is a generator, every call produces a new iterator, so the range can be iterated multiple times and even in nested loops without interference — unlike a self-iterator that would exhaust after one pass. This generator-as-`__iter__` pattern is the idiomatic, bug-resistant way to make a class iterable, neatly separating the collection from the traversal state.

### Q89. Explain the difference between `list.sort()` and `sorted()`, and what "stable sort" means.

`list.sort()` sorts in place and returns `None` (mutating the original); `sorted()` returns a new sorted list and accepts any iterable, leaving the input untouched. Both use Timsort, which is *stable*: elements comparing equal retain their original relative order. Stability is what enables correct multi-key sorting by sorting on the secondary key first, then the primary — a widely used technique.

```python
records = [
    {"name": "Ann", "dept": "eng"},
    {"name": "Bob", "dept": "eng"},
    {"name": "Cara", "dept": "ops"},
]

# Stable multi-key sort: sort by secondary key first, then primary
records.sort(key=lambda r: r["name"])    # secondary
records.sort(key=lambda r: r["dept"])    # primary; ties keep name order
# Within each dept, names remain alphabetically ordered
```

Because Timsort is stable, the first sort by name is preserved within groups produced by the second sort by department — giving a correct two-level ordering without a compound key. Choosing `sorted()` versus `sort()` comes down to whether you must preserve the original (and whether the source is a non-list iterable). Knowing the sort is stable unlocks this clean layered-sorting idiom.

### Q90. What is duck typing, and how does it relate to EAFP versus LBYL?

Duck typing judges an object by its behavior (methods/attributes) rather than its type — "if it walks like a duck." This pairs with the EAFP style ("Easier to Ask Forgiveness than Permission"): attempt the operation and catch the exception if it fails, rather than LBYL ("Look Before You Leap") which checks preconditions first. EAFP is idiomatic in Python because it avoids race conditions between check and use and embraces duck typing — you try the behavior instead of inspecting the type.

```python
# EAFP (idiomatic): try the operation, handle failure
def get_value(obj, key):
    try:
        return obj[key]
    except (KeyError, TypeError):
        return None

# LBYL (less Pythonic; also racy for files/shared state)
def get_value_lbyl(obj, key):
    if hasattr(obj, "__getitem__") and key in obj:
        return obj[key]
    return None
```

The EAFP version simply attempts the subscript and handles the failure, working for any object that supports indexing regardless of its concrete type — pure duck typing. The LBYL version's pre-checks are more verbose, can miss edge cases, and for external resources (files, sockets) introduce a time-of-check/time-of-use race. EAFP's "try it and catch" aligns with how Python is designed to be written.


### Q91. How do you write a thread-safe singleton or lazy initializer correctly?

Lazy initialization that must run exactly once under concurrency requires guarding the first-time creation. The double-checked locking pattern checks the sentinel without the lock (fast path), then acquires the lock and rechecks before creating, ensuring only one thread initializes. Alternatively, module-level initialization is inherently thread-safe because imports are serialized, and `functools.cache` on a no-arg function provides a clean lazy singleton in modern code.

```python
import functools


@functools.cache                # no args: caches the single result, thread-safe init in CPython
def get_settings() -> "Settings":
    return Settings(load_from_disk())   # runs once, reused thereafter
```

Using `@cache` on a parameterless factory yields a lazily-initialized, reused singleton with no explicit locks, leaning on the cache's first-call semantics — far simpler than hand-rolled double-checked locking and free of the subtle ordering bugs that pattern invites. For cases needing eager control or non-cache semantics, the module-level instance remains the most robust and readable approach.

### Q92. What is the difference between `==` and `is`, and when is each correct?

`==` invokes `__eq__` to compare *values*; `is` compares *identity* (whether two names reference the same object in memory). Use `==` for value comparison (numbers, strings, containers) and `is` only for singletons (`None`, `True`, `False`) and intentional identity checks. The frequent bug is using `is` for value comparison, which works by accident for interned small ints/strings and fails unpredictably otherwise (see Q41).

```python
# Correct: identity check against the None singleton
def process(value=None):
    if value is None:           # canonical, fast, unambiguous
        value = []
    return value

# Correct: value comparison
assert [1, 2] == [1, 2]         # True: equal values
assert ([1, 2] is [1, 2]) is False   # different objects
```

`value is None` is the canonical idiom because `None` is a guaranteed singleton, making identity both correct and faster than equality. The two equal lists compare `==` True but `is` False — they hold the same values in distinct objects, illustrating why `is` must never be used for value comparison. Reserving `is` for singletons keeps these semantics unambiguous.

### Q93. Explain how `@property` works and when to use it over plain attributes.

`@property` turns method calls into attribute-access syntax, letting you add validation, computation, or laziness behind `obj.attr` without changing the public interface. It is built on the descriptor protocol. The guidance: start with plain attributes (Python has no cost to exposing them) and introduce a property *only when* you need to intercept access — validation, computed/derived values, or backward-compatible refactoring. Avoid heavy work in a getter, since callers expect attribute access to be cheap.

```python
class Circle:
    def __init__(self, radius: float):
        self.radius = radius        # uses the setter -> validation

    @property
    def radius(self) -> float:
        return self._radius

    @radius.setter
    def radius(self, value: float) -> None:
        if value < 0:
            raise ValueError("radius cannot be negative")
        self._radius = value

    @property
    def area(self) -> float:
        return 3.141592653589793 * self._radius ** 2   # computed, read-only
```

The `radius` property enforces non-negativity on every assignment — including in `__init__` — centralizing the invariant, while `area` exposes a derived value as if it were stored. Because the property preserves `circle.radius` syntax, you can introduce validation later without breaking any caller, which is precisely why Python favors public attributes upfront and properties only when interception becomes necessary.

### Q94. How does `super()` work without arguments, and what does it actually return?

In Python 3, the zero-argument `super()` inside a method uses compiler-injected `__class__` and the instance to return a *proxy* that dispatches attribute lookups to the next class in the instance's MRO (not necessarily the literal parent). It enables cooperative multiple inheritance: each class calls `super()` and the chain walks the MRO exactly once. The proxy is bound, so it forwards `self` automatically.

```python
class Base:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)      # eventually reaches object
        print("Base init")

class Mixin:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        print("Mixin init")

class Concrete(Mixin, Base):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        print("Concrete init")

Concrete()      # Base init -> Mixin init -> Concrete init (MRO unwinds)
```

Each `__init__` calls `super().__init__(**kwargs)`, so the chain follows `Concrete`'s MRO (`Concrete → Mixin → Base → object`) and every class initializes exactly once — the cooperative pattern that makes mixins work. Passing `**kwargs` through lets each level consume its own arguments and forward the rest. The proxy resolves "next in MRO" dynamically, which is why `super()` is not simply "the parent class."

### Q95. What are coroutine-based vs. thread-based vs. process-based concurrency trade-offs in one framework?

The choice hinges on workload and overhead. Coroutines (`asyncio`): thousands of cheap tasks on one thread, best for high-concurrency I/O, but require fully async libraries and one blocking call stalls everything. Threads: preemptive, work with blocking libraries, moderate overhead, limited by the GIL for CPU work, risk of races. Processes: true parallelism for CPU work, full isolation, but heavy memory and IPC/pickling costs. Match the model to whether the bottleneck is I/O concurrency, blocking-library compatibility, or CPU.

```python
# Decision guide encoded as a routing function
def choose_executor(workload: str):
    return {
        "high_concurrency_io": "asyncio",      # 10k+ network calls
        "blocking_io_libs":    "threads",      # legacy/blocking I/O clients
        "cpu_bound":           "processes",    # number crunching
    }[workload]
```

The mapping captures the practical heuristic: `asyncio` when you have massive I/O concurrency and async-native libraries; threads when you need concurrency but depend on blocking libraries (they release the GIL during I/O); processes when CPU is the bottleneck and you need real parallelism. Naming the deciding factor — I/O vs. CPU vs. library compatibility — rather than reciting features is what distinguishes a senior answer.

### Q96. How do you handle backpressure in an async system?

Backpressure prevents a fast producer from overwhelming a slow consumer and exhausting memory. In `asyncio`, a bounded `asyncio.Queue` provides it naturally: `put` awaits (suspends the producer) when the queue is full, and `get` awaits when empty, throttling production to match consumption. Semaphores cap concurrent operations (e.g., limit simultaneous outbound requests). The principle is to bound buffers and concurrency rather than letting unbounded work accumulate.

```python
import asyncio


async def producer(queue: asyncio.Queue):
    for item in range(1000):
        await queue.put(item)        # suspends when queue is full -> backpressure


async def consumer(queue: asyncio.Queue):
    while True:
        item = await queue.get()
        await slow_process(item)
        queue.task_done()


async def main():
    queue: asyncio.Queue = asyncio.Queue(maxsize=50)   # bounded buffer
    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue))
        tg.create_task(consumer(queue))
        await queue.join()
```

The `maxsize=50` queue forces the producer to await once the buffer fills, so it cannot outrun the consumer or balloon memory — backpressure with no manual rate logic. To bound *outbound* concurrency instead (e.g., API calls), wrap operations in `async with semaphore`. Designing for bounded buffers and limited concurrency from the start is what keeps async services stable under load.

### Q97. What is the difference between `pickle`, `json`, and other serialization formats, including security concerns?

`json` is text-based, language-agnostic, and safe to parse, but handles only basic types. `pickle` serializes almost any Python object including custom classes, but is Python-specific and *insecure*: unpickling untrusted data can execute arbitrary code, so never unpickle data from an untrusted source. For cross-language or untrusted contexts use JSON, MessagePack, or Protocol Buffers; reserve `pickle` for trusted, internal, Python-to-Python transfer (e.g., `multiprocessing`).

```python
import json
import pickle

obj = {"name": "Ann", "scores": [90, 85]}

# JSON: safe, portable, human-readable; basic types only
text = json.dumps(obj)
restored = json.loads(text)            # safe even on untrusted input

# pickle: powerful but DANGEROUS on untrusted data
blob = pickle.dumps(obj)
# pickle.loads(untrusted_blob)  -> can execute arbitrary code: NEVER do this
```

`json.loads` cannot execute code, making it safe for data crossing trust boundaries (web APIs, user files), whereas `pickle.loads` can invoke `__reduce__` to run arbitrary code during deserialization — a remote-code-execution vector if the data is attacker-controlled. The senior takeaway is to treat serialization format choice as a security decision: portability and safety (JSON/protobuf) for boundaries, `pickle` only within a trusted Python boundary.

### Q98. Explain `enum.Enum` and its advanced uses (`auto`, `Flag`, `IntEnum`).

`Enum` defines a set of named, singleton constant members, giving type safety and readability over bare strings/ints. `auto()` assigns values automatically; `IntEnum` members compare equal to ints (useful for interop at the cost of type strictness); `Flag`/`IntFlag` support bitwise combination for sets of options. Enums are iterable, hashable, and their members are singletons (compare with `is`), preventing the bugs that magic constants cause.

```python
from enum import Enum, Flag, auto


class Status(Enum):
    PENDING = auto()
    ACTIVE = auto()
    CLOSED = auto()


class Permission(Flag):
    READ = auto()
    WRITE = auto()
    EXECUTE = auto()


access = Permission.READ | Permission.WRITE     # combine flags
print(Permission.READ in access)                 # True
print(Status.ACTIVE is Status.ACTIVE)            # True: singleton members
```

`Status` replaces error-prone string literals with type-checked singletons that an IDE can autocomplete and a checker can verify exhaustively. `Permission(Flag)` models combinable options with bitwise operators, so `READ | WRITE` is a single value testable with `in` — far cleaner than juggling integer bitmasks by hand. Enums turn implicit "magic" constants into a self-documenting, mistake-resistant type.

### Q99. How do you design for testability (dependency injection, ports and adapters)?

Testable design isolates business logic from external concerns (databases, network, time, randomness) by depending on abstractions and injecting concrete implementations. The hexagonal/ports-and-adapters architecture formalizes this: the core defines *ports* (interfaces) and the outside world provides *adapters*. In tests you inject fakes, so logic is verified without real I/O — fast, deterministic, and independent of the environment. The enabling techniques are dependency injection and programming to interfaces.

```python
from typing import Protocol


class Clock(Protocol):
    def now(self) -> float: ...


class TokenService:
    def __init__(self, clock: Clock):
        self._clock = clock          # injected dependency

    def is_expired(self, issued_at: float, ttl: float) -> bool:
        return self._clock.now() - issued_at > ttl


# Test with a controllable fake clock
class FakeClock:
    def __init__(self, t): self._t = t
    def now(self): return self._t

assert TokenService(FakeClock(100)).is_expired(issued_at=0, ttl=50) is True
```

Injecting a `Clock` lets the test supply a `FakeClock` with a fixed time, making the expiry logic deterministic — no `time.sleep`, no flakiness. The `TokenService` depends only on the `Clock` protocol, so production wiring passes a real clock and tests pass a fake, with identical core logic. Designing seams like this upfront is the single biggest lever for a fast, reliable test suite.

### Q100. What are the most impactful Pythonic principles for writing maintainable production code?

The durable principles: favor readability and explicitness (the Zen of Python), prefer composition and dependency injection over inheritance and globals, use the right data structure for the algorithmic complexity, manage resources with context managers, validate at boundaries (not with `assert`), make illegal states unrepresentable (enums, frozen dataclasses, types), and write tests against behavior through injected seams. Lean on the standard library and idioms before reaching for cleverness; clarity compounds across a codebase's lifetime.

```python
from dataclasses import dataclass
from enum import Enum


class OrderState(Enum):
    DRAFT = "draft"
    PLACED = "placed"
    SHIPPED = "shipped"


@dataclass(frozen=True, slots=True)
class Order:
    id: str
    state: OrderState
    total: float

    def ship(self) -> "Order":
        if self.state is not OrderState.PLACED:
            raise ValueError(f"cannot ship from {self.state.value}")
        return Order(self.id, OrderState.SHIPPED, self.total)
```

This small example concentrates several principles: a frozen dataclass makes `Order` immutable and hashable, an enum makes the state space explicit and illegal values unrepresentable, and `ship` returns a new validated instance rather than mutating — so invalid transitions fail loudly and state is always consistent. Code shaped this way is easy to test, reason about, and safely evolve, which is the real measure of senior-level Python.

### Q101. How does `asyncio.Semaphore` differ from a `Lock`, and how do you rate-limit concurrent tasks?

A `Lock` permits exactly one holder at a time (mutual exclusion); a `Semaphore` permits up to *N* concurrent holders, making it the tool for bounding concurrency rather than enforcing exclusivity. To rate-limit outbound work — say, capping simultaneous API calls so you don't overwhelm a downstream service or hit connection limits — wrap each operation in `async with semaphore`, which blocks new entrants once N are in flight.

```python
import asyncio


async def fetch_all(urls: list[str], client, limit: int = 10):
    sem = asyncio.Semaphore(limit)              # at most `limit` concurrent

    async def bounded_fetch(url: str):
        async with sem:                          # acquire a slot
            return await client.get(url)

    return await asyncio.gather(*(bounded_fetch(u) for u in urls))
```

Even though `gather` schedules every fetch at once, the semaphore ensures only `limit` are actually executing at any moment — the rest await a free slot. This decouples the *number of tasks* (could be thousands) from the *concurrency level* (bounded), protecting both your process and the downstream service. A `Lock` would serialize everything; the semaphore tunes the parallelism precisely.

### Q102. What is the difference between `os.fork`, `spawn`, and `forkserver` start methods in multiprocessing?

`multiprocessing` supports three ways to start workers. `fork` (Unix default historically) copies the parent's memory via copy-on-write — fast, but unsafe with threads and can duplicate problematic state (locks, open connections). `spawn` (default on macOS/Windows, and now safer everywhere) starts a fresh interpreter and re-imports the module — slower but clean and isolated. `forkserver` forks from a minimal pristine server process, combining speed with safety. The choice affects correctness, especially in threaded or library-heavy programs.

```python
import multiprocessing as mp


def worker(x):
    return x * x


if __name__ == "__main__":                  # REQUIRED for spawn/forkserver
    ctx = mp.get_context("spawn")           # explicit, portable start method
    with ctx.Pool(4) as pool:
        print(pool.map(worker, range(10)))
```

Selecting `spawn` explicitly avoids the subtle bugs `fork` causes when the parent holds threads or locks (the child inherits a possibly-inconsistent copy), at the cost of re-importing the module — which is exactly why the `if __name__ == "__main__"` guard is mandatory, preventing infinite process spawning on re-import. For threaded applications, preferring `spawn` or `forkserver` is the safe default.

### Q103. How do you avoid blocking the event loop, and how do you detect when you have?

The event loop is single-threaded, so any synchronous call that takes appreciable time — CPU loops, blocking I/O, `time.sleep` — freezes *all* coroutines. Avoid it by using async-native libraries, offloading blocking work via `run_in_executor`, and breaking long CPU work into chunks or moving it to processes. Detect blocking with `asyncio`'s debug mode, which logs callbacks/coroutines that run longer than a threshold, and with `loop.slow_callback_duration` tuning.

```python
import asyncio


async def main():
    loop = asyncio.get_running_loop()
    # Debug mode warns about slow callbacks that block the loop
    loop.slow_callback_duration = 0.1        # warn if a callback exceeds 100ms

    # WRONG: time.sleep(2) would freeze the whole loop
    # RIGHT: yield control
    await asyncio.sleep(2)


asyncio.run(main(), debug=True)              # enables loop monitoring
```

Running with `debug=True` and a low `slow_callback_duration` makes the loop log a warning whenever a coroutine or callback hogs the thread, surfacing accidental blocking that is otherwise invisible until latency spikes in production. The fix is always the same: replace the blocking call with its async equivalent or push it to an executor, keeping every coroutine's synchronous run between awaits short.

### Q104. Explain `functools.partial` and `functools.reduce` with practical uses.

`partial` pre-binds some arguments of a callable, returning a new callable with a smaller signature — useful for adapting functions to callback/strategy interfaces, configuring handlers, or supplying fixed configuration. `reduce` folds a binary function across an iterable to a single accumulated value; it is powerful but often less readable than a loop or a specialized built-in (`sum`, `math.prod`, `any`), so reserve it for genuine custom accumulations.

```python
import functools
import operator

# partial: pre-configure a logger or callback
def log(level: str, msg: str) -> None:
    print(f"[{level}] {msg}")

error = functools.partial(log, "ERROR")     # fixes the first argument
error("disk full")                           # [ERROR] disk full

# reduce: custom fold where no built-in fits
product = functools.reduce(operator.mul, [1, 2, 3, 4], 1)   # 24
deep_merge = functools.reduce(merge_dicts, list_of_configs, {})
```

`partial(log, "ERROR")` creates a specialized `error` function without a wrapper `def`, ideal for handing pre-configured callables to frameworks expecting a simple signature. `reduce` shines for accumulations like merging a sequence of config dicts where no built-in applies; for sums and products, prefer `sum`/`math.prod`, since explicit names read better than a generic fold. Both reflect the first-class-function ethos of idiomatic Python.

### Q105. How do you profile and optimize an `asyncio` application specifically?

Async profiling differs because wall-time includes intentional awaits. Use `asyncio` debug mode to catch blocking callbacks, `py-spy` (which handles async stacks) to sample where real CPU time goes, and `yappi` (async-aware) to attribute time per coroutine including correct handling of context switches. The optimization priorities: eliminate accidental blocking first, then reduce per-task overhead and excessive task creation, then bound concurrency to avoid resource thrashing.

```python
import yappi
import asyncio


async def workload():
    await asyncio.gather(*(do_work(i) for i in range(100)))


yappi.set_clock_type("wall")        # wall time matters for async
yappi.start()
asyncio.run(workload())
yappi.stop()

yappi.get_func_stats().sort("ttot").print_all()   # total time per coroutine
```

`yappi` with wall-clock timing correctly attributes time across coroutine context switches — something `cProfile` mishandles for async code — so the report reflects where the application actually spends time, including awaits. Sorting by total time reveals whether the bottleneck is a blocking call, excessive task churn, or a genuinely slow downstream, directing the fix. Pairing this with debug-mode blocking detection (Q103) covers the two failure modes unique to async systems.

---

## Closing Notes

These 105 questions span the depth expected of a senior Python engineer: language internals, the concurrency model and its GIL-imposed trade-offs, memory and garbage collection, data-structure complexity, design patterns rendered Pythonically, and a rigorous approach to testing and profiling. The strongest interview answers do three things consistently — state the underlying mechanism precisely, show idiomatic production code, and articulate the *trade-off* that motivates the choice. Mechanism, idiom, and trade-off together are what distinguish recall from mastery.
