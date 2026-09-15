# Context Managers and Descriptors

**Level:** Intermediate

## What

A **context manager** implements `__enter__`/`__exit__` to guarantee setup
and teardown around a block of code (the `with` statement) — the
canonical solution for "always release this resource, even on error." A
**descriptor** implements `__get__`/`__set__`/`__delete__` and controls
attribute access on other classes — the mechanism behind `@property`,
methods, and ORM-style declarative fields.

## Why

Backend code constantly manages resources with a strict acquire/release
lifecycle: DB connections, file handles, locks, transactions. Context
managers make that lifecycle correct and readable, replacing manual
try/finally. Descriptors matter because they explain *how* `@property`
works internally and how libraries like SQLAlchemy and Pydantic implement
declarative, validated attributes.

## How

### `with` and manual context managers

```python
class DatabaseConnection:
    def __enter__(self) -> "DatabaseConnection":
        print("opening connection")
        return self

    def __exit__(self, exc_type, exc_value, traceback) -> bool:
        print("closing connection")
        return False  # False -- don't suppress exceptions

with DatabaseConnection() as conn:
    print("using connection")
    # __exit__ runs even if an exception is raised here
```

`__exit__` receives exception info if one occurred inside the block.
Returning `True` **suppresses** the exception; returning `False`/`None`
lets it propagate — almost always what you want unless you're explicitly
handling a specific error type.

### `contextlib.contextmanager` (generator-based)

```python
from contextlib import contextmanager

@contextmanager
def database_connection():
    print("opening connection")
    conn = "connection-object"
    try:
        yield conn
    finally:
        print("closing connection")

with database_connection() as conn:
    print("using", conn)
```

Code before `yield` is `__enter__`; code after (in `finally`) is
`__exit__`. This is the concise, most common way to write a custom context
manager without a full class.

### Multiple and nested context managers

```python
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())
```

```python
from contextlib import ExitStack

def process_files(paths: list[str]):
    with ExitStack() as stack:
        files = [stack.enter_context(open(p)) for p in paths]
        # all files closed automatically, even if one open() fails midway
```

`ExitStack` is useful when the number of context managers isn't known
upfront (e.g. opening a dynamic list of resources).

### Async context managers

```python
class AsyncConnection:
    async def __aenter__(self) -> "AsyncConnection":
        ...
        return self

    async def __aexit__(self, exc_type, exc_value, traceback) -> bool:
        ...
        return False

async with AsyncConnection() as conn:
    ...
```

Common in async DB drivers/HTTP clients (`async with session.get(url)`).
Deeper async patterns are covered in the Advanced asyncio topic.

### Descriptors

```python
class PositiveNumber:
    def __set_name__(self, owner, name) -> None:
        self._name = "_" + name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, self._name)

    def __set__(self, instance, value) -> None:
        if value < 0:
            raise ValueError(f"{self._name} must be positive")
        setattr(instance, self._name, value)

class Order:
    quantity = PositiveNumber()

    def __init__(self, quantity: int) -> None:
        self.quantity = quantity  # goes through PositiveNumber.__set__

order = Order(5)
order.quantity = -1  # raises ValueError
```

A **data descriptor** defines both `__get__` and `__set__` (like above); a
**non-data descriptor** defines only `__get__` (this is how plain
functions become bound methods on instances). Data descriptors take
priority over instance `__dict__` entries; non-data descriptors do not —
this is why setting `instance.__dict__["quantity"]` directly can't bypass
a data descriptor's validation, but *can* shadow a non-data descriptor.

`@property` is itself implemented as a descriptor — understanding
descriptors explains why `@property` works the way it does at the class
level.

## When to use

- Context managers: any acquire/release pair — DB sessions/transactions,
  file handles, locks, temporary state changes (e.g. temporarily
  monkeypatching a setting in tests).
- `contextlib.contextmanager` for simple cases; a full class when you need
  additional methods or more complex state.
- Descriptors: building reusable, validated attribute behavior shared
  across many classes (rare to write directly in application code — more
  common when building a small framework/library, e.g. an ORM-like field
  system).

## When NOT to use

- Don't manage resources with manual try/finally when a context manager
  (existing or custom) already exists — it's less error-prone and more
  idiomatic.
- Don't write a custom descriptor when `@property` already solves the
  problem for a single class — reserve descriptors for genuinely reusable
  behavior across multiple classes.
- Don't suppress exceptions in `__exit__` (`return True`) unless you have a
  specific, documented reason — silently swallowing errors during cleanup
  is a common source of confusing bugs.

## Common mistakes

- Forgetting `__exit__` must return `False`/`None` to propagate exceptions
  — accidentally returning a truthy value swallows all exceptions.
- Using `@contextmanager` without wrapping the `yield` in `try/finally`,
  so cleanup code doesn't run if the block raises.
- Confusing data vs non-data descriptors and being surprised when instance
  `__dict__` does or doesn't override descriptor behavior.
- Writing a context manager that isn't reentrant/reusable when the code
  using it expects to enter it multiple times (e.g. in a loop) — generator-
  based context managers from `@contextmanager` are single-use per call.

## Interview questions

1. What do `__enter__`/`__exit__` need to return, and what does the return
   value of `__exit__` control?
2. Show how `contextlib.contextmanager` maps `yield` to
   `__enter__`/`__exit__`.
3. When would you use `ExitStack`?
4. What's the difference between a data descriptor and a non-data
   descriptor? Why does it matter?
5. How is `@property` implemented in terms of descriptors?
6. Why shouldn't `__exit__` suppress exceptions by default?

## Senior-level considerations

- Context managers are the standard way to guarantee DB transaction
  boundaries (commit/rollback) are correct even under exceptions — a
  common production bug is a transaction left open because cleanup wasn't
  wrapped in `try/finally`.
- Async context managers (`__aenter__`/`__aexit__`) are essential for
  correctly managing connection pools and sessions in async web services;
  misusing them (not awaiting `__aexit__`, or reusing a single-use context
  manager) causes subtle connection leaks under load.
- Descriptors are a "framework-level" tool — understanding them helps when
  reading (or building) libraries with declarative class-level fields
  (ORMs, serialization libraries) rather than something typically written
  in everyday application code.
