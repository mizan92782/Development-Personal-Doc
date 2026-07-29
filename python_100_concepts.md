# 100 Python Concepts for Interview Preparation

A topic-by-topic reference. Each concept has a short explanation and a code snippet where useful. Use this alongside hands-on practice (LeetCode/HackerRank) for best results.

---

## Section 1: Core Language Fundamentals

**1. Dynamic typing** — Variables aren't bound to a type; the type lives with the object, not the name.
```python
x = 5
x = "now a string"  # totally legal
```

**2. Everything is an object** — Functions, classes, modules, even integers are objects with attributes.
```python
print((5).bit_length())  # 3
```

**3. Variable scope (LEGB rule)** — Local → Enclosing → Global → Built-in, the order Python searches for names.

**4. `global` and `nonlocal` keywords**
```python
count = 0
def increment():
    global count
    count += 1
```

**5. Pass by object reference** — Python passes references to objects, not the objects themselves. Behavior differs for mutable vs immutable args (see mutable defaults below).

**6. Truthy and falsy values** — `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `None`, `False` are all falsy.

**7. Chained comparisons**
```python
1 < x < 10  # equivalent to 1 < x and x < 10
```

**8. Ternary (conditional) expressions**
```python
result = "even" if x % 2 == 0 else "odd"
```

**9. Walrus operator `:=` (assignment expressions, Python 3.8+)**
```python
if (n := len(data)) > 10:
    print(f"Too long: {n}")
```

**10. Type hints / annotations**
```python
def add(a: int, b: int) -> int:
    return a + b
```

---

## Section 2: Data Types & Structures

**11. Mutable vs immutable types** — `list`/`dict`/`set` are mutable; `int`/`str`/`tuple`/`frozenset` are immutable.

**12. Mutable default arguments trap**
```python
def f(items=[]):  # BAD — shared across calls
    items.append(1)
    return items
```

**13. Tuples vs lists** — Tuples are immutable, hashable (if contents are), and slightly faster; used for fixed-size records.

**14. Named tuples**
```python
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)
```

**15. `dataclasses` (Python 3.7+)**
```python
from dataclasses import dataclass
@dataclass
class Point:
    x: int
    y: int
```

**16. Sets and set operations** — union `|`, intersection `&`, difference `-`, symmetric difference `^`.

**17. `frozenset`** — an immutable, hashable version of a set (can be used as a dict key).

**18. Dictionary ordering** — since Python 3.7, dicts preserve insertion order (implementation detail turned language guarantee).

**19. `dict.get()` vs `dict[]`** — `.get()` returns `None` (or a default) instead of raising `KeyError`.

**20. `collections.defaultdict`**
```python
from collections import defaultdict
d = defaultdict(list)
d["missing"].append(1)  # no KeyError
```

**21. `collections.Counter`**
```python
from collections import Counter
Counter("mississippi")  # {'i': 4, 's': 4, 'p': 2, 'm': 1}
```

**22. `collections.deque`** — double-ended queue, O(1) appends/pops from both ends (unlike lists, which are O(n) from the front).

**23. `collections.OrderedDict`** — legacy ordered dict; mostly redundant now but still supports `move_to_end()`.

**24. String immutability & interning** — Small/short identifier-like strings may be cached (interned) by CPython for memory efficiency.

**25. Bytes vs strings** — `bytes` are raw binary data; `str` is Unicode text. Convert with `.encode()`/`.decode()`.

---

## Section 3: Control Flow

**26. `for...else` and `while...else`** — The `else` block runs only if the loop completes without hitting `break`.
```python
for x in range(5):
    if x == 10:
        break
else:
    print("completed without break")
```

**27. `break`, `continue`, `pass`** — `pass` is a no-op placeholder; doesn't affect loop control.

**28. `zip()`** — pairs elements from multiple iterables; stops at the shortest one.

**29. `enumerate()`** — yields `(index, value)` pairs; avoids manual counter variables.

**30. `range()` is lazy** — it's a sequence type that generates values on demand, not a pre-built list.

---

## Section 4: Functions

**31. First-class functions** — functions can be passed as arguments, returned, and assigned to variables.

**32. Lambda functions** — anonymous, single-expression functions.
```python
square = lambda x: x * x
```

**33. `*args` and `**kwargs`** — collect extra positional/keyword arguments.

**34. Argument unpacking** — `func(*list_args, **dict_kwargs)`.

**35. Keyword-only arguments**
```python
def f(a, *, b):  # b MUST be passed as keyword
    pass
```

**36. Positional-only arguments (Python 3.8+)**
```python
def f(a, b, /, c):  # a, b must be positional
    pass
```

**37. Closures** — inner functions capturing variables from an enclosing scope.

**38. Decorators** — functions wrapping other functions to add behavior.

**39. `functools.wraps`** — preserves `__name__`/`__doc__` of the original function when writing decorators.

**40. `functools.lru_cache`** — memoization decorator for caching function results.
```python
from functools import lru_cache
@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
```

**41. `functools.partial`** — pre-fills some arguments of a function.
```python
from functools import partial
double = partial(lambda x, y: x * y, 2)
```

**42. Recursion & recursion limits** — Python's default recursion limit is ~1000 (`sys.getrecursionlimit()`); no tail-call optimization exists.

**43. Docstrings & `__doc__`** — conventionally triple-quoted strings right after a `def`/`class`.

**44. Variable-length function signatures with defaults + `*args` + `**kwargs` ordering** — must follow: positional, `*args`, keyword-only, `**kwargs`.

**45. Function annotations aren't enforced** — they're metadata only; use `mypy`/`pyright` for actual static checking.

---

## Section 5: Object-Oriented Programming

**46. Classes and instances** — blueprint vs concrete object.

**47. `__init__` vs `__new__`** — `__new__` creates the instance (rarely overridden); `__init__` initializes it.

**48. Instance vs class attributes** — class attributes are shared across all instances unless shadowed.

**49. `self` — explicit instance reference** — Python doesn't implicitly bind methods to instances; `self` is passed explicitly.

**50. `classmethod` vs `staticmethod` vs instance method** — `classmethod` gets `cls`; `staticmethod` gets neither; instance methods get `self`.

**51. Property decorators (`@property`)** — expose methods as attributes, enabling computed properties and validation.
```python
class Circle:
    def __init__(self, r):
        self._r = r
    @property
    def area(self):
        return 3.14159 * self._r ** 2
```

**52. Getters/setters via `@property.setter`**
```python
@area.setter
def area(self, value):
    ...
```

**53. Encapsulation conventions** — `_protected`, `__private` (name-mangled, not truly private).

**54. Name mangling** — `self.__x` becomes `self._ClassName__x` internally.

**55. Inheritance** — `class Child(Parent):` reuses/extends parent behavior.

**56. Multiple inheritance & MRO** — Python uses C3 linearization to determine method lookup order (`ClassName.__mro__`).

**57. `super()`** — calls the next method in the MRO chain, enabling cooperative multiple inheritance.

**58. Abstract Base Classes (`abc` module)**
```python
from abc import ABC, abstractmethod
class Shape(ABC):
    @abstractmethod
    def area(self): ...
```

**59. Duck typing** — "If it walks like a duck and quacks like a duck…" — Python cares about behavior, not declared type.

**60. Magic/dunder methods** — `__str__`, `__repr__`, `__eq__`, `__len__`, `__getitem__`, `__contains__`, etc., let custom objects integrate with built-in syntax.

**61. `__str__` vs `__repr__`** — `__str__` is for readable user-facing output; `__repr__` should be unambiguous, ideally reproducible via `eval()`.

**62. Operator overloading** — implement `__add__`, `__eq__`, `__lt__`, etc. to support `+`, `==`, `<` on custom objects.

**63. `__slots__`** — restricts instance attributes to a fixed set, saving memory by skipping the per-instance `__dict__`.

**64. Composition vs inheritance** — "has-a" vs "is-a"; composition is often favored for flexibility.

**65. Metaclasses** — classes whose instances are themselves classes; `type` is the default metaclass. Used for advanced class customization (e.g., ORMs).

---

## Section 6: Functional Programming

**66. `map()`, `filter()`, `reduce()`**
```python
from functools import reduce
list(map(lambda x: x*2, [1,2,3]))
list(filter(lambda x: x % 2 == 0, [1,2,3,4]))
reduce(lambda a, b: a + b, [1,2,3,4])
```

**67. List/dict/set comprehensions** — concise, often faster syntax for building collections.

**68. Generator expressions** — like list comprehensions but lazy, using `()`.

**69. `itertools` module basics** — `chain`, `combinations`, `permutations`, `product`, `groupby`, `islice`.

**70. Pure functions & side effects** — functions that don't mutate external state are easier to test and reason about.

**71. Higher-order functions** — functions that take/return other functions (e.g., decorators, `map`).

**72. Currying (manual, via closures or `functools.partial`)** — transforming a multi-arg function into a chain of single-arg functions.

**73. Immutability as a design choice** — favoring immutable data structures to avoid accidental mutation bugs.

---

## Section 7: Memory & Performance

**74. Reference counting & garbage collection** — CPython primarily uses reference counting; a cyclic garbage collector handles reference cycles.

**75. `sys.getsizeof()`** — inspect the memory footprint of an object.

**76. Shallow copy vs deep copy** — `copy.copy()` vs `copy.deepcopy()`.

**77. Generators for memory efficiency** — process large datasets without loading everything into memory.

**78. String concatenation performance** — repeated `+=` on strings in a loop is O(n²); prefer `"".join(list)`.

**79. List vs generator trade-off** — lists allow repeated iteration/indexing; generators are memory-light but single-use.

**80. `__slots__` for memory savings** — avoids per-instance `__dict__` overhead in classes with many instances.

**81. Interning of small integers and strings** — CPython caches small ints (-5 to 256) and some string literals for speed.

---

## Section 8: Concurrency & Parallelism

**82. The GIL (Global Interpreter Lock)** — only one thread executes Python bytecode at a time in CPython.

**83. Threading for I/O-bound work** — `threading` module; threads release the GIL during I/O waits.

**84. Multiprocessing for CPU-bound work** — `multiprocessing` module; separate processes, separate memory, true parallelism.

**85. `asyncio` and `async`/`await`** — single-threaded cooperative concurrency, ideal for many concurrent I/O operations.
```python
import asyncio
async def fetch():
    await asyncio.sleep(1)
    return "done"
```

**86. `concurrent.futures`** — high-level `ThreadPoolExecutor` and `ProcessPoolExecutor` for parallel task execution.

**87. Race conditions & locks** — `threading.Lock()` to guard shared mutable state across threads.

**88. Deadlocks** — occur when two or more threads wait on each other's locks indefinitely.

**89. Queue-based thread communication** — `queue.Queue` for safe data passing between threads.

---

## Section 9: Error Handling

**90. `try`/`except`/`else`/`finally`** — full exception-handling flow, with `else` running only on success and `finally` always running.

**91. Custom exception classes** — subclass `Exception` for domain-specific errors.

**92. Exception chaining (`raise ... from ...`)** — preserves the original traceback context.

**93. `assert` statements** — for debugging invariants; stripped out when Python runs with `-O` (don't use for production validation).

**94. Catching multiple exception types** — `except (TypeError, ValueError) as e:`.

**95. Custom exception hierarchies** — base exception class with specific subclasses for different failure modes.

---

## Section 10: Iterators, Generators & Context Managers

**96. Iterator protocol** — `__iter__()` and `__next__()`; `StopIteration` signals exhaustion.

**97. Generator functions (`yield`)** — pause/resume execution, maintaining state between calls.

**98. `yield from`** — delegates iteration to a sub-generator/iterable.

**99. Context managers (`with` statement, `__enter__`/`__exit__`)** — guarantee setup/teardown (e.g., closing files) even on exceptions.

**100. `contextlib.contextmanager`** — write context managers as generator functions instead of classes.
```python
from contextlib import contextmanager
@contextmanager
def open_resource():
    print("acquire")
    yield "resource"
    print("release")
```

---

## Bonus Section: Frequently-Tested Extras

**101. `is` vs `==`** — identity vs equality.

**102. Exception `finally` with `return`** — a `return` in `finally` overrides a `return`/exception from the `try` block (a common gotcha).

**103. `__eq__` and `__hash__` relationship** — if you override `__eq__`, you should override `__hash__` too, or the object becomes unhashable.

**104. Method Resolution Order edge cases** — diamond inheritance and how `super()` avoids calling shared ancestors twice.

**105. Type coercion vs type casting** — implicit (`1 + 1.0 == 2.0`) vs explicit (`int("5")`).

---

## How to Use This Guide

1. Go section by section — don't jump around.
2. For each concept, run the code snippet yourself and tweak it.
3. Try to explain the concept out loud in one or two sentences before moving on.
4. Revisit Sections 2, 4, 5, and 8 the most — they come up most often in interviews.
5. Pair this with real coding problems (LeetCode/HackerRank) so concepts stick through application, not just recall.

**Deepen further at:**
- realpython.com — long-form deep dives on almost every topic above
- docs.python.org/3/reference/datamodel.html — dunder methods, MRO, descriptors
- docs.python.org/3/library/ — official docs for `itertools`, `functools`, `collections`, `asyncio`, etc.
- tenthousandmeters.com — "Python behind the scenes" series on CPython internals
