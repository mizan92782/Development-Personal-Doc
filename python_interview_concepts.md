# Python Critical Concepts — Interview Study Guide

---

## 1. Mutable vs Immutable Types

**Immutable**: `int`, `float`, `str`, `tuple`, `bool`, `frozenset` — cannot be changed after creation. Any "modification" creates a new object.

**Mutable**: `list`, `dict`, `set`, custom objects (by default) — can be changed in place.

```python
# Immutable example
x = "hello"
y = x
x += " world"
print(y)  # "hello" — unaffected, because x now points to a NEW string

# Mutable example
a = [1, 2, 3]
b = a
a.append(4)
print(b)  # [1, 2, 3, 4] — affected, because a and b point to the SAME list
```

### The classic trap: Mutable Default Arguments

```python
def add_item(item, items=[]):   # BAD — default list created ONCE at function definition
    items.append(item)
    return items

print(add_item(1))  # [1]
print(add_item(2))  # [1, 2]  <-- Surprise! Not [2]
```

**Fix:**
```python
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

**Why it happens:** default argument values are evaluated **once**, at function definition time, not on every call. Mutable defaults persist across calls.

---

## 2. GIL & Multithreading vs Multiprocessing

**GIL (Global Interpreter Lock)**: a mutex in CPython that allows only **one thread** to execute Python bytecode at a time, even on multi-core machines.

- **Threading** → good for **I/O-bound** tasks (network calls, file I/O, DB queries) because threads release the GIL while waiting on I/O.
- **Multiprocessing** → good for **CPU-bound** tasks (heavy computation) because each process has its own Python interpreter and memory space — bypasses the GIL entirely.

```python
import threading
import multiprocessing
import time

def cpu_task():
    return sum(i * i for i in range(10**7))

# Threading won't speed up CPU-bound work due to GIL
start = time.time()
threads = [threading.Thread(target=cpu_task) for _ in range(4)]
[t.start() for t in threads]
[t.join() for t in threads]
print("Threading:", time.time() - start)

# Multiprocessing achieves true parallelism
start = time.time()
processes = [multiprocessing.Process(target=cpu_task) for _ in range(4)]
[p.start() for p in processes]
[p.join() for p in processes]
print("Multiprocessing:", time.time() - start)
```

**Interview tip:** Mention `asyncio` too — a single-threaded, event-loop-based approach for I/O concurrency without the overhead of threads.

---

## 3. Generators/Iterators vs Lists (Memory Efficiency)

**List**: stores all items in memory at once.
**Generator**: produces items lazily, one at a time, using `yield` — much more memory-efficient for large datasets.

```python
# List — computes and stores all values immediately
def squares_list(n):
    return [i * i for i in range(n)]

# Generator — computes one value at a time, on demand
def squares_gen(n):
    for i in range(n):
        yield i * i

import sys
print(sys.getsizeof(squares_list(10000)))   # large — full list in memory
print(sys.getsizeof(squares_gen(10000)))    # tiny — just a generator object
```

**Iterator protocol**: an object is an iterator if it implements `__iter__()` and `__next__()`.

```python
class CountUp:
    def __init__(self, max_val):
        self.max_val = max_val
        self.current = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.max_val:
            raise StopIteration
        self.current += 1
        return self.current

for num in CountUp(5):
    print(num)  # 1 2 3 4 5
```

**Key point:** generators can only be iterated once; lists can be reused/iterated multiple times.

---

## 4. Decorators and Closures

**Closure**: a function that "remembers" variables from its enclosing scope even after that scope has finished executing.

```python
def make_multiplier(factor):
    def multiplier(x):
        return x * factor   # 'factor' is remembered via closure
    return multiplier

double = make_multiplier(2)
print(double(5))  # 10
```

**Decorator**: a function that wraps another function to extend/modify its behavior, without changing its source code.

```python
import functools
import time

def timer(func):
    @functools.wraps(func)  # preserves original function's name/docstring
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)

slow_function()  # prints: slow_function took 1.0001s
```

**Decorators with arguments** (a decorator factory):
```python
def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                func(*args, **kwargs)
        return wrapper
    return decorator

@repeat(3)
def greet():
    print("Hello!")

greet()  # prints "Hello!" 3 times
```

---

## 5. `*args`, `**kwargs`, and Argument Unpacking

- `*args` → collects extra **positional** arguments into a tuple
- `**kwargs` → collects extra **keyword** arguments into a dict

```python
def demo(*args, **kwargs):
    print(args)    # tuple
    print(kwargs)  # dict

demo(1, 2, 3, name="Alice", age=30)
# (1, 2, 3)
# {'name': 'Alice', 'age': 30}
```

**Unpacking when calling functions:**
```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))  # unpacks list into positional args → 6

data = {"a": 1, "b": 2, "c": 3}
print(add(**data))  # unpacks dict into keyword args → 6
```

**Combining with normal params (order matters):**
```python
def func(a, b, *args, c, **kwargs):
    print(a, b, args, c, kwargs)

func(1, 2, 3, 4, c=5, d=6)
# 1 2 (3, 4) 5 {'d': 6}
```

---

## 6. Deep Copy vs Shallow Copy

```python
import copy

original = [[1, 2, 3], [4, 5, 6]]

shallow = copy.copy(original)      # or original.copy() / original[:]
deep = copy.deepcopy(original)

original[0][0] = 999

print(shallow)  # [[999, 2, 3], [4, 5, 6]] — inner list is SHARED, so it's affected
print(deep)     # [[1, 2, 3], [4, 5, 6]]   — completely independent copy
```

- **Shallow copy**: creates a new outer object, but nested objects are still references to the originals.
- **Deep copy**: recursively copies everything, including nested objects — fully independent.

**Rule of thumb:** use shallow copy when your data is flat (no nested mutables); use deep copy when you have nested lists/dicts/objects and need true independence.

---

## 7. `is` vs `==`

- `==` → checks **value equality** (calls `__eq__`)
- `is` → checks **identity** (are they the *same object* in memory?)

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True  — same values
print(a is b)  # False — different objects in memory
print(a is c)  # True  — c is literally the same object as a
```

**Common gotcha with small integers/strings (CPython caching/interning):**
```python
x = 5
y = 5
print(x is y)  # True — small ints (-5 to 256) are cached/interned by CPython

x = 1000
y = 1000
print(x is y)  # False (usually) — outside the cached range, separate objects
```

**Best practice:** always use `is` only for singleton comparisons like `None`, `True`, `False` (e.g. `if x is None:`), and `==` for everything else.

---

## 8. Context Managers (`with` statement)

A context manager handles setup/teardown automatically (e.g., opening/closing files, acquiring/releasing locks) — guarantees cleanup even if an exception occurs.

```python
with open("file.txt", "r") as f:
    data = f.read()
# file is automatically closed here, even if an exception occurred above
```

**Writing your own with a class** (`__enter__` / `__exit__`):
```python
class ManagedFile:
    def __init__(self, filename):
        self.filename = filename

    def __enter__(self):
        self.file = open(self.filename, "r")
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        return False  # False = don't suppress exceptions

with ManagedFile("file.txt") as f:
    print(f.read())
```

**Writing one with `contextlib` (simpler, function-based):**
```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename):
    f = open(filename, "r")
    try:
        yield f
    finally:
        f.close()

with managed_file("file.txt") as f:
    print(f.read())
```

---

## 9. List / Dict / Set Comprehensions

```python
# List comprehension
squares = [x * x for x in range(10)]

# With condition (filter)
evens = [x for x in range(10) if x % 2 == 0]

# Dict comprehension
square_map = {x: x * x for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Set comprehension
unique_lengths = {len(word) for word in ["cat", "dog", "bird", "ox"]}
# {2, 3, 4}

# Nested comprehension (flatten a 2D list)
matrix = [[1, 2], [3, 4], [5, 6]]
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6]

# Generator expression (lazy, memory-efficient — use () instead of [])
gen = (x * x for x in range(10))
```

**Interview tip:** comprehensions are generally faster than equivalent `for` loops with `.append()` because the looping happens in optimized C code internally.

---

## 10. Exception Handling and Custom Exceptions

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"Error: {e}")
except (TypeError, ValueError) as e:
    print(f"Type or value error: {e}")
else:
    print("Runs only if no exception occurred")
finally:
    print("Always runs — cleanup code goes here")
```

**Custom exceptions:**
```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(f"Cannot withdraw {amount}, balance is only {balance}")

class BankAccount:
    def __init__(self, balance):
        self.balance = balance

    def withdraw(self, amount):
        if amount > self.balance:
            raise InsufficientFundsError(self.balance, amount)
        self.balance -= amount

account = BankAccount(100)
try:
    account.withdraw(150)
except InsufficientFundsError as e:
    print(e)  # Cannot withdraw 150, balance is only 100
```

**`raise ... from ...`** — chaining exceptions to preserve context:
```python
try:
    int("abc")
except ValueError as e:
    raise RuntimeError("Failed to process input") from e
```

---

## 11. OOP: classmethod vs staticmethod, MRO, Multiple Inheritance

**`classmethod`** — receives the class (`cls`) as first arg, often used for alternate constructors.
**`staticmethod`** — receives nothing special, behaves like a plain function namespaced inside the class.

```python
class Pizza:
    def __init__(self, ingredients):
        self.ingredients = ingredients

    @classmethod
    def margherita(cls):          # alternate constructor
        return cls(["mozzarella", "tomato"])

    @staticmethod
    def is_valid_ingredient(ing):  # utility function, doesn't need class or instance
        return ing in ["cheese", "tomato", "mozzarella", "basil"]

p = Pizza.margherita()
print(Pizza.is_valid_ingredient("cheese"))  # True
```

**MRO (Method Resolution Order)** — the order Python searches classes to resolve a method/attribute in multiple inheritance. Uses the **C3 linearization algorithm**.

```python
class A:
    def greet(self):
        print("A")

class B(A):
    def greet(self):
        print("B")

class C(A):
    def greet(self):
        print("C")

class D(B, C):
    pass

d = D()
d.greet()  # "B" — follows MRO

print(D.__mro__)
# (D, B, C, A, object) — this is the search order
```

**Multiple inheritance with `super()`** (cooperative multiple inheritance):
```python
class Base:
    def __init__(self):
        print("Base init")

class Left(Base):
    def __init__(self):
        super().__init__()
        print("Left init")

class Right(Base):
    def __init__(self):
        super().__init__()
        print("Right init")

class Child(Left, Right):
    def __init__(self):
        super().__init__()
        print("Child init")

Child()
# Base init
# Right init
# Left init
# Child init
```

**Why this matters in interviews:** `super()` doesn't just call the immediate parent — it follows the MRO chain, which is what makes cooperative multiple inheritance work correctly (each class in the chain runs exactly once).

---

## Quick Self-Test Checklist

Before your interview, make sure you can explain **out loud, without notes**:
- [ ] Why mutable default arguments are dangerous
- [ ] Why threading doesn't speed up CPU-bound Python code
- [ ] The memory difference between a list and a generator
- [ ] How a closure "remembers" outer variables
- [ ] The difference between `*args` and `**kwargs`
- [ ] When shallow copy breaks vs when it's fine
- [ ] Why `is` can behave unexpectedly with integers
- [ ] What `__enter__`/`__exit__` do
- [ ] Why comprehensions can be faster than loops
- [ ] How to design a custom exception hierarchy
- [ ] How Python resolves method calls in multiple inheritance (MRO)

**Recommended deepening resources for each topic:**
- Real Python (realpython.com) — search each topic name
- docs.python.org/3/reference/datamodel.html — for MRO, dunder methods, descriptors
- tenthousandmeters.com — "Python behind the scenes" series for GIL/CPython internals
