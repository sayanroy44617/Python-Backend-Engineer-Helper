# Data Types and Collections

**Level:** Foundation

## What

Python's built-in scalar types (`int`, `float`, `bool`, `str`, `None`) and its
core collection types: `list`, `tuple`, `set`, `dict`. Also covers mutability,
identity vs equality, and copying semantics — the mechanics that cause the
most subtle backend bugs.

## Why

Every data structure decision affects correctness and performance: choosing a
`list` where a `set` gives O(1) membership checks, or sharing a mutable
default argument across requests, are common sources of production bugs.
Understanding identity vs equality also underpins caching, memoization, and
ORM identity maps (e.g. SQLAlchemy's identity map).

## How

### Scalars

```python
x: int = 42
y: float = 3.14
name: str = "sayan"
is_active: bool = True
nothing: None = None
```

### Collections at a glance

| Type | Ordered | Mutable | Duplicates | Typical use |
|---|---|---|---|---|
| `list` | Yes | Yes | Yes | Sequential data, order matters |
| `tuple` | Yes | No | Yes | Fixed-size records, dict keys, function returns |
| `set` | No | Yes | No | Membership tests, deduplication |
| `frozenset` | No | No | No | Hashable set (usable as dict key) |
| `dict` | Yes (insertion) | Yes | Keys unique | Lookups by key, JSON-like structures |

```python
items: list[str] = ["a", "b", "c"]
point: tuple[int, int] = (10, 20)
seen: set[int] = {1, 2, 3}
config: dict[str, str] = {"env": "prod"}
```

### Mutability

Mutable objects (`list`, `dict`, `set`) can be changed in place; their
identity (`id()`) stays the same. Immutable objects (`int`, `str`, `tuple`,
`frozenset`) cannot — any "modification" creates a new object.

```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)  # [1, 2, 3, 4] -- same underlying object
```

The classic pitfall: mutable default arguments are created **once**, at
function definition time, and shared across all calls.

```python
# Bug: the same list is reused across calls
def add_item(item, bucket=[]):
    bucket.append(item)
    return bucket

# Fix: use None as sentinel
def add_item(item, bucket: list | None = None) -> list:
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

### Identity vs equality

- `==` compares **value** equality (calls `__eq__`).
- `is` compares **identity** — whether two names point to the same object in
  memory (equivalent to `id(a) == id(b)`).

```python
a = [1, 2, 3]
b = [1, 2, 3]
a == b  # True  -- same contents
a is b  # False -- different objects

x = None
x is None  # correct way to check for None
```

Small integers (-5 to 256) and interned strings may share identity due to
CPython implementation details — never rely on `is` for value comparisons of
`int`/`str`; it is a CPython optimization detail, not a language guarantee.

### Copying

- **Assignment** (`b = a`) does not copy — both names reference the same
  object.
- **Shallow copy** (`list(a)`, `a.copy()`, `copy.copy(a)`) creates a new outer
  container, but nested objects are still shared.
- **Deep copy** (`copy.deepcopy(a)`) recursively copies nested objects too.

```python
import copy

original = {"nested": [1, 2, 3]}
shallow = original.copy()
shallow["nested"].append(4)
print(original["nested"])  # [1, 2, 3, 4] -- nested list is shared

deep = copy.deepcopy(original)
deep["nested"].append(5)
print(original["nested"])  # unaffected
```

## When to use

- `list`: default sequential container; order matters, you need indexing or
  append-heavy workloads.
- `tuple`: immutable records (e.g. a `(lat, lon)` pair), function returns
  with multiple values, dict/set keys when hashability is required.
- `set`: fast membership checks, deduplication, set algebra (union,
  intersection).
- `dict`: key-based lookups, structured records before reaching for a
  dataclass or Pydantic model.

## When NOT to use

- Don't use a `list` for membership testing on large data — `in` is O(n) vs
  O(1) average for `set`/`dict`.
- Don't use mutable objects as dict keys or set members — they aren't
  hashable (`TypeError: unhashable type`).
- Don't reach for `deepcopy` by default — it's expensive; only use it when
  nested mutable state truly must be isolated.

## Common mistakes

- Mutable default arguments (see above).
- Using `is` to compare values instead of identity.
- Assuming `list.copy()` deep-copies nested structures.
- Modifying a list while iterating over it (skips elements or raises
  `RuntimeError` for other containers like dicts/sets).

```python
# Bug: mutating a list while iterating
nums = [1, 2, 3, 4]
for n in nums:
    if n % 2 == 0:
        nums.remove(n)  # skips elements

# Fix: iterate over a copy, or build a new list
nums = [n for n in nums if n % 2 != 0]
```

## Interview questions

1. What's the difference between `is` and `==`? When would `is` give a
   surprising result?
2. Why is a mutable default argument a bug? How do you fix it?
3. When would you choose a `tuple` over a `list`?
4. What's the time complexity of membership testing (`in`) for a `list` vs a
   `set`? Why?
5. Explain shallow vs deep copy with an example involving nested structures.
6. Why can't you use a `list` as a dictionary key?

## Senior-level considerations

- Mutability and identity semantics directly explain ORM identity maps
  (SQLAlchemy returns the *same* Python object for the same primary key
  within a session) and why `==` on ORM models is often overridden.
- Choosing `frozenset`/`tuple` for immutable shared state avoids
  accidental mutation bugs in concurrent code — important once you start
  reasoning about asyncio tasks or multiprocessing sharing data.
- At scale, container choice affects memory footprint: a `dict` has higher
  per-entry overhead than a `list`; for large read-heavy datasets consider
  `array`, `__slots__`, or external stores instead of naive dict/list use.
