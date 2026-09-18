# Iterators and Generators

**Level:** Intermediate

## What

An **iterator** is any object implementing `__iter__` and `__next__` (the
iterator protocol). A **generator** is a function using `yield` (or a
generator expression) that produces an iterator automatically, pausing and
resuming execution instead of computing everything up front.

## Why

Backend code frequently processes streams of data that don't fit — or
shouldn't be forced — into memory at once: paginated API results, large
query result sets, file parsing, or event streams. Generators let you
express "process one item at a time" clearly and efficiently, and they
underpin `async for`/async generators used throughout async backend code.

## How

### The iterator protocol

```python
class CountUp:
    def __init__(self, limit: int) -> None:
        self.limit = limit
        self.current = 0

    def __iter__(self) -> "CountUp":
        return self

    def __next__(self) -> int:
        if self.current >= self.limit:
            raise StopIteration
        self.current += 1
        return self.current

for n in CountUp(3):
    print(n)  # 1, 2, 3
```

`for` loops work by calling `iter(obj)` to get an iterator, then repeatedly
calling `next()` until `StopIteration` is raised. `list`, `dict`, `str`,
etc. are **iterable** (define `__iter__`) but are not themselves iterators
— each call to `iter()` on them returns a fresh iterator, which is why you
can loop over the same list multiple times.

### Generators with `yield`

```python
def count_up(limit: int):
    current = 0
    while current < limit:
        current += 1
        yield current

for n in count_up(3):
    print(n)  # 1, 2, 3
```

Calling `count_up(3)` doesn't run the function body — it returns a
generator object immediately. Code runs up to each `yield`, pauses, and
resumes from there on the next `next()` call. This makes generators ideal
for lazy, memory-efficient iteration.

```python
def read_large_file(path: str):
    with open(path) as f:
        for line in f:
            yield line.strip()

# Only one line is in memory at a time, regardless of file size
for line in read_large_file("huge.csv"):
    process(line)
```

### Generator expressions

```python
total = sum(n * n for n in range(1_000_000))  # no intermediate list built
```

Covered briefly in
[Comprehensions and Functions](02-comprehensions-and-functions.md); the
key iterator-protocol detail is that a generator expression produces a
generator object, evaluated lazily one item at a time.

### `yield from` and delegation

```python
def inner():
    yield 1
    yield 2

def outer():
    yield from inner()
    yield 3

list(outer())  # [1, 2, 3]
```

`yield from` delegates iteration to a sub-generator/iterable, forwarding
values (and exceptions/return values) — useful for composing generators
without manual loops.

### Sending values into a generator

```python
def running_total():
    total = 0
    while True:
        value = yield total
        total += value

gen = running_total()
next(gen)          # prime the generator, get initial 0
gen.send(10)        # 10
gen.send(5)         # 15
```

`send()` resumes the generator and provides a value for the paused `yield`
expression — used in coroutine-style patterns, though `asyncio` (covered in
Advanced) largely replaces this pattern for real concurrency.

### Generators vs returning a list

```python
# Eager: builds the whole list in memory
def get_ids_list(n: int) -> list[int]:
    return [i for i in range(n)]

# Lazy: produces one value at a time
def get_ids_gen(n: int):
    for i in range(n):
        yield i
```

A generator can only be iterated **once** — once exhausted, it doesn't
reset. If you need to iterate multiple times, materialize a list or
re-create the generator.

## When to use

- Streaming/processing large or unbounded data (files, DB cursors,
  paginated API responses) without loading everything into memory.
- Producing infinite or lazily-computed sequences (e.g. an incrementing ID
  generator).
- Composing pipelines of transformations lazily (generator expressions
  chained together) for memory efficiency.

## When NOT to use

- Don't use a generator when you need to iterate multiple times or need
  `len()`/random access — use a `list` instead.
- Don't use a custom iterator class (`__iter__`/`__next__`) when a simple
  generator function does the same thing more concisely — reserve manual
  iterator classes for cases needing extra state/methods beyond iteration.
- Avoid generators for small, fixed-size collections where the eager list
  is simpler to reason about and debug.

## Common mistakes

- Trying to re-iterate an exhausted generator and getting no results,
  with no error raised.
- Forgetting that a generator function's body doesn't execute at all until
  you start iterating (a bug inside a generator won't surface until
  consumed).
- Returning a value from a generator (`return value` inside a generator)
  and expecting it in a `for` loop — it's only accessible via
  `StopIteration.value` when driving the generator manually, or as the
  result of `yield from` in the delegating generator.
- Mixing up `yield` and `return` — a function with any `yield` is always a
  generator function, even if `return` also appears (as a way to stop
  early).

## Interview questions

1. What's the difference between an iterable and an iterator?

   **Answer:** An iterable is something you can ask for an iterator from, like a `list` or `dict`. An iterator is the object that keeps the iteration state and knows what the next item is.

   ```python
   items: list[int] = [1, 2, 3]
   iterator = iter(items)
   print(next(iterator))  # 1
   ```
2. How does a `for` loop use `iter()` and `next()` under the hood?

   **Answer:** `for` first calls `iter(obj)`, then keeps calling `next()` until it gets `StopIteration`. That's why anything implementing the iterator protocol works in a normal loop.

   ```python
   values = iter([10, 20])
   print(next(values))  # 10
   print(next(values))  # 20
   ```
3. Why are generators more memory-efficient than returning a list? Give a
   backend example.

   **Answer:** A generator yields one item at a time instead of building the full result up front, so memory stays flat even for large inputs. Typical backend case: streaming rows from a big query or lines from a log file.

   ```python
   from collections.abc import Iterator

   def user_ids() -> Iterator[int]:
       for user_id in range(1_000_000):
           yield user_id
   ```
4. What does `yield from` do, and why is it useful when composing
   generators?

   **Answer:** `yield from` hands control to another iterable or generator so you don't have to write the loop manually. It's mainly about cleaner composition when one generator is just forwarding values from another.

   ```python
   from collections.abc import Iterator

   def child() -> Iterator[int]:
       yield from [1, 2]
   ```
5. Can you iterate a generator twice? What happens if you try?

   **Answer:** No — a generator is single-use. Once it's exhausted, iterating it again gives you nothing unless you create a fresh generator object.

   ```python
   gen = (n for n in range(2))
   print(list(gen))  # [0, 1]
   print(list(gen))  # []
   ```
6. What's the difference between `next(gen)` and `gen.send(value)`?

   **Answer:** `next(gen)` just resumes the generator and asks for the next yielded value. `send(value)` also resumes it, but injects a value back into the paused `yield` expression.

   ```python
   from collections.abc import Iterator

   def echo() -> Iterator[int]:
       received = yield 0
       yield received
   ```

## Senior-level considerations

- Generators are the foundation for `async for`/async generators
  (`async def` with `yield`), which are common in async database drivers
  and streaming HTTP responses — understanding sync generators first makes
  async generators much easier to reason about; for example, a FastAPI
  streaming response can yield chunks one at a time instead of buffering
  the whole payload.
- Lazy pipelines (chained generator expressions) can make debugging harder
  since errors surface only when consumed, not when constructed — weigh
  memory efficiency against debuggability for critical paths; for example,
  `(parse(row) for row in rows)` won't fail until some later consumer
  actually pulls the bad row.
- When streaming large query results (e.g. via SQLAlchemy's
  `yield_per`/server-side cursors), understanding the iterator protocol
  explains why the DB connection must stay open for the duration of
  iteration — a common source of "connection already closed" bugs if the
  generator is only partially consumed; for example, returning a generator
  from inside a function after the session context has already exited will
  usually break on the first real iteration.
