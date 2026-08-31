# LEVEL 8 — Python Internals (Backend Developer Deep Dive)

---

## PART 1 — FUNDAMENTALS

### 1. Variables

In Python, a variable is **not a box that holds a value** — it's a **name bound to an object in memory**. Assignment creates a reference (a pointer), not a copy.

```python
a = [1, 2, 3]
b = a          # b is another name pointing to the SAME object
b.append(4)
print(a)       # [1, 2, 3, 4]  <-- a changed too!
print(id(a) == id(b))  # True, same memory address
```

### 2. Objects

Everything in Python is an object — functions, classes, modules, even integers. Every object has:
- **Identity** — `id(obj)` (memory address in CPython)
- **Type** — `type(obj)`
- **Value** — the actual data

```python
x = 10
print(id(x), type(x), x)
```

### 3. References

Python uses **reference counting** (plus a cyclic garbage collector) for memory management. When you assign a variable, you increase the reference count of the object.

```python
import sys
a = [1, 2, 3]
print(sys.getrefcount(a))  # counts references (includes temp ref from getrefcount itself)
b = a
print(sys.getrefcount(a))  # increased
del b
print(sys.getrefcount(a))  # decreased
```

When refcount hits 0, the object is destroyed immediately (in CPython). Circular references are cleaned by the **garbage collector** (`gc` module).

### 4. Mutable vs Immutable

| Mutable | Immutable |
|---|---|
| list, dict, set, bytearray | int, float, str, tuple, frozenset, bool |

```python
# Immutable — operations create NEW objects
s = "hello"
s2 = s.upper()
print(s is s2)  # False, new string object

# Mutable — operations modify IN PLACE
lst = [1, 2, 3]
lst.append(4)   # same object, modified

# Classic gotcha: mutable default arguments
def add_item(item, bucket=[]):   # DANGER: default list is created ONCE
    bucket.append(item)
    return bucket

print(add_item(1))  # [1]
print(add_item(2))  # [1, 2]  <-- unexpected! same list reused across calls

# Fix:
def add_item_safe(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

### 5. List, Tuple, Set, Dictionary

```python
# LIST — ordered, mutable, allows duplicates. Backed by a dynamic array.
lst = [3, 1, 2]
lst.sort()                 # O(n log n)
lst.append(5)               # O(1) amortized

# TUPLE — ordered, immutable, allows duplicates. Slightly faster & hashable if contents are hashable.
t = (1, 2, 3)
# t[0] = 99  -> TypeError, tuples can't be modified
d = {t: "coordinates"}      # tuples can be dict keys, lists can't

# SET — unordered, mutable, unique elements, backed by a hash table.
s = {1, 2, 2, 3}
print(s)                    # {1, 2, 3}
print(2 in s)                # O(1) average lookup, vs O(n) for list

# DICTIONARY — key-value store, backed by a hash table. Insertion-ordered since Python 3.7+.
d = {"name": "Alice", "age": 30}
d["age"] += 1
print(d.get("missing", "default"))
```

**Why it matters for backend work:** choosing a `set`/`dict` over a `list` for membership checks turns O(n) lookups into O(1) — critical when filtering large querysets or deduping IDs.

### 6. Functions

Functions are **first-class objects** — they can be assigned, passed, and returned.

```python
def greet(name):
    return f"Hello, {name}"

say_hi = greet          # assign function to a variable
print(say_hi("Bob"))

def apply(func, value):  # pass function as argument
    return func(value)

print(apply(greet, "Carol"))

def make_multiplier(n):  # return a function (closure)
    def multiplier(x):
        return x * n
    return multiplier

times3 = make_multiplier(3)
print(times3(10))  # 30, "n" is captured in the closure
```

### 7. Decorators

A decorator is a function that **wraps another function** to extend its behavior without modifying its source code.

```python
import time
import functools

def timer(func):
    @functools.wraps(func)   # preserves original function's name/docstring
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper

@timer
def slow_query():
    time.sleep(1)
    return "data"

slow_query()
# Equivalent to: slow_query = timer(slow_query)
```

Real backend example — a simple auth decorator:

```python
def require_login(view_func):
    @functools.wraps(view_func)
    def wrapper(request, *args, **kwargs):
        if not request.user.is_authenticated:
            raise PermissionError("Login required")
        return view_func(request, *args, **kwargs)
    return wrapper
```

### 8. Generators

A generator produces values **lazily**, one at a time, instead of building a whole list in memory. Uses `yield`.

```python
def read_large_file(path):
    with open(path) as f:
        for line in f:
            yield line.strip()   # pauses here, resumes on next()

for line in read_large_file("huge_log.txt"):
    process(line)   # memory-efficient — never loads whole file at once
```

Generators are **iterators created via functions**. Calling a generator function doesn't run the body — it returns a generator object; execution happens on each `next()` call, resuming right after the last `yield`.

```python
def counter():
    print("start")
    yield 1
    print("middle")
    yield 2
    print("end")

g = counter()
next(g)   # prints "start", returns 1
next(g)   # prints "middle", returns 2
```

### 9. Iterators

An iterator is any object implementing `__iter__()` and `__next__()`. `for` loops work on anything iterable by internally calling these.

```python
class CountUp:
    def __init__(self, limit):
        self.limit = limit
        self.n = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.n >= self.limit:
            raise StopIteration
        self.n += 1
        return self.n

for num in CountUp(3):
    print(num)   # 1, 2, 3
```

**Iterable vs Iterator:** an *iterable* has `__iter__()` (e.g. a list); an *iterator* is what `__iter__()` returns, and it remembers state via `__next__()`.

### 10. Context Managers

Used with `with` to guarantee setup/teardown (e.g., closing files, releasing DB connections) even if an exception occurs.

```python
# Class-based
class DBConnection:
    def __enter__(self):
        print("opening connection")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("closing connection")
        return False   # False = don't suppress exceptions

with DBConnection() as conn:
    print("using connection")
# automatically closed, even if an error was raised inside

# Function-based, using contextlib
from contextlib import contextmanager

@contextmanager
def db_connection():
    print("opening connection")
    try:
        yield "conn_object"
    finally:
        print("closing connection")

with db_connection() as conn:
    print("using", conn)
```

### 11. Exceptions

Python uses exceptions for error handling instead of return codes.

```python
class InsufficientFundsError(Exception):
    pass

def withdraw(balance, amount):
    if amount > balance:
        raise InsufficientFundsError(f"Cannot withdraw {amount}, balance is {balance}")
    return balance - amount

try:
    withdraw(100, 150)
except InsufficientFundsError as e:
    print(f"Error: {e}")
except Exception as e:
    print(f"Unexpected: {e}")
else:
    print("Withdrawal successful")   # runs only if no exception
finally:
    print("Transaction attempt finished")  # always runs
```

### 12. Modules

A module is simply a `.py` file. Importing runs the file once and caches it in `sys.modules`.

```python
# math_utils.py
def add(a, b):
    return a + b

# main.py
import math_utils
print(math_utils.add(2, 3))

from math_utils import add
print(add(2, 3))
```

### 13. Packages

A package is a directory of modules with an `__init__.py` (optional since Python 3.3+ via namespace packages, but still common).

```
myapp/
    __init__.py
    models.py
    views.py
    utils/
        __init__.py
        helpers.py
```

```python
from myapp.utils.helpers import format_date
```

### 14. Virtual Environments

Isolates project dependencies so different projects can use different package versions without conflict.

```bash
python -m venv venv
source venv/bin/activate      # Linux/Mac
venv\Scripts\activate         # Windows

pip install django fastapi
pip freeze > requirements.txt

deactivate
```

Under the hood, activating a venv just changes `PATH` and `sys.prefix` so `python`/`pip` point to the isolated interpreter and site-packages folder.

---

## PART 2 — ADVANCED PYTHON (Concurrency & Parallelism)

### 1. GIL (Global Interpreter Lock)

CPython's GIL is a mutex that allows **only one thread to execute Python bytecode at a time**, even on multi-core machines. It exists because CPython's memory management (reference counting) isn't thread-safe.

**Consequence:** pure-Python CPU-bound code doesn't get faster with threads. I/O-bound code still benefits, because the GIL is released during I/O waits (disk, network, sleep).

```python
import threading
import time

def cpu_task():
    count = 0
    for _ in range(50_000_000):
        count += 1

start = time.time()
t1 = threading.Thread(target=cpu_task)
t2 = threading.Thread(target=cpu_task)
t1.start(); t2.start()
t1.join(); t2.join()
print("Threaded:", time.time() - start)   # NOT ~2x faster, due to GIL
```

### 2. Threading

Best for **I/O-bound** work: network calls, file I/O, DB queries — tasks that spend time *waiting*, not computing.

```python
import threading
import requests

urls = ["https://example.com"] * 5
results = []

def fetch(url):
    r = requests.get(url)
    results.append(r.status_code)

threads = [threading.Thread(target=fetch, args=(u,)) for u in urls]
for t in threads: t.start()
for t in threads: t.join()

print(results)
```

### 3. Multiprocessing

Bypasses the GIL entirely by running separate **OS processes**, each with its own Python interpreter and memory space. Best for **CPU-bound** work.

```python
from multiprocessing import Pool

def square(n):
    return n * n

if __name__ == "__main__":
    with Pool(processes=4) as pool:
        results = pool.map(square, range(10))
    print(results)   # true parallel execution across cores
```

Trade-off: processes have overhead (spawning, no shared memory by default — need `multiprocessing.Queue`/`Manager` to communicate), so they're heavier than threads.

### 4. AsyncIO

A **single-threaded, single-process** concurrency model using an event loop and cooperative multitasking. Instead of OS-level context switching, coroutines voluntarily yield control at `await` points.

```python
import asyncio

async def fetch_data(id):
    print(f"Start fetching {id}")
    await asyncio.sleep(1)          # simulates non-blocking I/O wait
    print(f"Done fetching {id}")
    return f"data-{id}"

async def main():
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3),
    )
    print(results)

asyncio.run(main())
# All 3 "fetches" run concurrently — total time ~1s, not 3s
```

### 5. Coroutines

A coroutine is a special function defined with `async def`. Calling it doesn't run it immediately — it returns a **coroutine object** that must be awaited or scheduled on the event loop.

```python
async def say_hello():
    print("hello")

coro = say_hello()     # nothing printed yet — just creates the coroutine object
asyncio.run(coro)      # NOW it runs, prints "hello"
```

### 6. Event Loop

The event loop is the scheduler that manages and executes coroutines, callbacks, and I/O events. It runs one task until it hits an `await`, then switches to another ready task — all on a single thread.

```python
import asyncio

async def task(name, delay):
    print(f"{name} started")
    await asyncio.sleep(delay)   # yields control back to the event loop
    print(f"{name} finished")

async def main():
    loop = asyncio.get_event_loop()
    print("Loop running:", loop.is_running())
    await asyncio.gather(task("A", 2), task("B", 1))

asyncio.run(main())
# Output order: A started, B started, B finished (after 1s), A finished (after 2s)
```

### 7. `async` / `await`

- `async def` marks a function as a coroutine function.
- `await` pauses execution of the current coroutine until the awaited coroutine/future completes, **without blocking the thread** — the event loop runs other tasks in the meantime.

```python
async def get_user(user_id):
    await asyncio.sleep(0.5)   # imagine this is: await db.fetch_one(...)
    return {"id": user_id, "name": "Alice"}

async def get_orders(user_id):
    await asyncio.sleep(0.3)   # imagine this is: await db.fetch_all(...)
    return ["order1", "order2"]

async def get_user_dashboard(user_id):
    # run both independent I/O calls concurrently instead of sequentially
    user, orders = await asyncio.gather(
        get_user(user_id),
        get_orders(user_id),
    )
    return {"user": user, "orders": orders}

asyncio.run(get_user_dashboard(1))
# Sequential (await one after another): 0.5 + 0.3 = 0.8s
# Concurrent (gather): max(0.5, 0.3) = 0.5s
```

### 8. Concurrency vs Parallelism

| | Concurrency | Parallelism |
|---|---|---|
| Definition | Dealing with many things at once (interleaving) | Doing many things at the exact same instant |
| Example | asyncio, threading | multiprocessing, multi-core CPUs |
| Analogy | One chef switching between 3 pots on the stove | 3 chefs, each with their own pot |
| Best for | I/O-bound tasks | CPU-bound tasks |
| Python tool | `asyncio`, `threading` (limited by GIL) | `multiprocessing` |

```python
# Concurrency (asyncio) — one thread juggling multiple waits
async def concurrent_demo():
    await asyncio.gather(fetch_data(1), fetch_data(2))

# Parallelism (multiprocessing) — literally running on different cores at once
from multiprocessing import Process

def cpu_bound(n):
    sum(i * i for i in range(n))

if __name__ == "__main__":
    p1 = Process(target=cpu_bound, args=(10_000_000,))
    p2 = Process(target=cpu_bound, args=(10_000_000,))
    p1.start(); p2.start()
    p1.join(); p2.join()
```

---

## Sync vs Async — Why This Matters for Backend Development

```python
async def get_user():
    user = await db.get(...)
    return user
```

### The theory

In a **synchronous** web server, each request occupies a worker thread/process for its *entire* duration — including time spent waiting on the database, an external API, or disk I/O. If your DB query takes 200ms, that worker sits **idle but blocked** for 200ms, unable to serve any other request. To handle more concurrent requests, you need more workers (threads/processes) — each costing memory and OS scheduling overhead.

In an **asynchronous** server, when a coroutine hits `await db.get(...)`, it doesn't block the thread. It tells the event loop "I'm waiting on I/O — go run something else," and the event loop immediately picks up another incoming request's coroutine. When the DB response arrives, the event loop resumes the paused coroutine exactly where it left off. A single thread can juggle thousands of these paused-and-resumed coroutines because none of them are burning CPU while waiting — they're just waiting on a "wake me up when data arrives" signal (typically via `epoll`/`kqueue` at the OS level).

### Concrete comparison

```python
# --- SYNCHRONOUS version ---
import time

def get_user_sync():
    time.sleep(0.2)       # simulates blocking DB call — thread frozen for 200ms
    return {"id": 1}

def handle_100_requests_sync():
    start = time.time()
    for _ in range(100):
        get_user_sync()
    print("Sync total:", time.time() - start)   # ~20 seconds if run one worker at a time

# --- ASYNCHRONOUS version ---
import asyncio

async def get_user_async():
    await asyncio.sleep(0.2)   # simulates non-blocking DB call — thread free to do other work
    return {"id": 1}

async def handle_100_requests_async():
    start = asyncio.get_event_loop().time()
    await asyncio.gather(*(get_user_async() for _ in range(100)))
    print("Async total:", asyncio.get_event_loop().time() - start)   # ~0.2 seconds total!

asyncio.run(handle_100_requests_async())
```

**Why the async version is ~100x faster here:** all 100 "DB calls" are waiting on I/O simultaneously. Since none of them need the CPU while waiting, a single thread can have all 100 in flight at once and just wait for whichever finishes first. The synchronous version, run one at a time on one thread, pays the 200ms cost 100 separate times.

### When async actually helps (and when it doesn't)

- **Helps:** I/O-bound workloads — DB queries, HTTP calls to other services, file/network reads, waiting on message queues. This is the bread and butter of backend APIs (FastAPI's biggest selling point).
- **Doesn't help:** CPU-bound work — image processing, heavy computation, ML inference. `await` doesn't make a slow calculation faster; it only helps while *waiting*. For CPU-bound work inside an async app, you'd offload to `multiprocessing` or a thread pool executor (`loop.run_in_executor`) so you don't block the event loop.
- **Danger:** if you call a *blocking* function (e.g., `time.sleep()` or a sync DB driver) inside an `async def` without `await`-ing something non-blocking, you **freeze the entire event loop** — every other "concurrent" request stalls too. This is the #1 async footgun.

```python
# BAD — blocks the entire event loop, defeats the purpose of async
async def bad_get_user():
    time.sleep(0.2)   # blocking call! freezes ALL other coroutines
    return {"id": 1}

# GOOD — use an async-compatible sleep/driver
async def good_get_user():
    await asyncio.sleep(0.2)   # non-blocking, yields control back to the loop
    return {"id": 1}
```

### Summary table

| Aspect | Sync | Async |
|---|---|---|
| Model | Blocking calls, one at a time per worker | Non-blocking, cooperative multitasking |
| Scaling strategy | More threads/processes | More coroutines on fewer threads |
| Best for | CPU-bound work | I/O-bound work |
| Memory cost per "unit of concurrency" | High (thread stack ~MBs) | Low (coroutine ~KBs) |
| Framework examples | Flask (default), Django (sync views) | FastAPI, Django (async views), aiohttp |
| Risk | Thread exhaustion under high I/O load | Blocking the event loop by accident |

This is exactly why FastAPI (built on Starlette + asyncio) can handle thousands of concurrent I/O-bound requests on a single process far more efficiently than a traditional sync WSGI app under similar load — as long as the entire I/O chain (DB driver, HTTP client, cache client) is truly async-compatible end to end.
