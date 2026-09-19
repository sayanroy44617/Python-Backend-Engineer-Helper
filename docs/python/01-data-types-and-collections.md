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

### Lists — creation, indexing, and slicing

```python
nums = [10, 20, 30, 40, 50]

nums[0]        # 10        -- first element
nums[-1]       # 50        -- last element
nums[1:3]      # [20, 30]  -- slice: start inclusive, stop exclusive
nums[::2]      # [10, 30, 50] -- every second element
nums[::-1]     # [50, 40, 30, 20, 10] -- reversed copy
```

### List methods

| Method | Effect | Example | Result |
|---|---|---|---|
| `append(x)` | Add `x` to the end | `nums.append(60)` | `[10, 20, 30, 40, 50, 60]` |
| `extend(iterable)` | Append every item from `iterable` | `nums.extend([60, 70])` | `[..., 60, 70]` |
| `insert(i, x)` | Insert `x` before index `i` | `nums.insert(0, 5)` | `[5, 10, 20, ...]` |
| `remove(x)` | Remove the **first** matching value (raises `ValueError` if absent) | `nums.remove(20)` | `[10, 30, 40, 50]` |
| `pop(i=-1)` | Remove and return item at index `i` (default: last) | `nums.pop()` | returns `50`, list shrinks |
| `clear()` | Remove all items | `nums.clear()` | `[]` |
| `index(x)` | Index of first match (raises `ValueError` if absent) | `nums.index(30)` | `2` |
| `count(x)` | Number of occurrences of `x` | `nums.count(20)` | `1` |
| `sort(key=None, reverse=False)` | Sort **in place** | `nums.sort(reverse=True)` | descending order |
| `reverse()` | Reverse **in place** | `nums.reverse()` | order flipped |
| `copy()` | Shallow copy | `nums.copy()` | new outer list, same nested refs |

```python
scores = [88, 95, 72, 61]
scores.sort()                      # [61, 72, 88, 95] -- mutates in place
top_two = sorted(scores, reverse=True)[:2]  # sorted() returns a NEW list

# sort() vs sorted(): sort() mutates and returns None; sorted() returns a
# new list and leaves the original untouched -- a common source of bugs
# when someone writes `scores = scores.sort()` and gets None.
```

```python
# key= for custom sort order -- extremely common in interviews
users = [{"name": "Ana", "age": 34}, {"name": "Bo", "age": 22}]
users.sort(key=lambda u: u["age"])
# [{'name': 'Bo', 'age': 22}, {'name': 'Ana', 'age': 34}]
```

### Tuples — packing, unpacking, and methods

```python
point = (10, 20)              # packing
x, y = point                   # unpacking
first, *rest = (1, 2, 3, 4)    # star-unpacking: first=1, rest=[2, 3, 4]

# Single-element tuples need a trailing comma -- (10) is just an int
single = (10,)
```

A tuple only has **two** methods, precisely because it's immutable — there's
nothing to mutate, so there's no `append`/`remove`/`sort`.

| Method | Effect | Example | Result |
|---|---|---|---|
| `count(x)` | Number of occurrences of `x` | `(1, 2, 2, 3).count(2)` | `2` |
| `index(x)` | Index of first match | `(1, 2, 2, 3).index(2)` | `1` |

```python
# Named tuples give tuple immutability with attribute-style access --
# a lightweight alternative to a class for simple records.
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(10, 20)
p.x, p.y   # (10, 20)
```

### Sets — creation and methods

```python
a = {1, 2, 3}
b = {3, 4, 5}
empty = set()   # NOT {} -- {} creates an empty dict, not an empty set
```

**Mutating methods** (change the set in place):

| Method | Effect | Example | Result |
|---|---|---|---|
| `add(x)` | Add a single element | `a.add(4)` | `{1, 2, 3, 4}` |
| `remove(x)` | Remove `x` (raises `KeyError` if absent) | `a.remove(1)` | `{2, 3, 4}` |
| `discard(x)` | Remove `x` if present (no error if absent) | `a.discard(99)` | unchanged, no error |
| `pop()` | Remove and return an arbitrary element | `a.pop()` | removes some element |
| `clear()` | Remove all elements | `a.clear()` | `set()` |
| `update(iterable)` | Add all elements from `iterable` | `a.update([5, 6])` | union, in place |

**Set algebra** (each has an operator form and a `*_update` in-place form):

| Method | Operator | Effect | Example |
|---|---|---|---|
| `union(b)` | `a \| b` | Elements in either set | `{1,2,3} \| {3,4}` → `{1,2,3,4}` |
| `intersection(b)` | `a & b` | Elements in both sets | `{1,2,3} & {2,3,4}` → `{2,3}` |
| `difference(b)` | `a - b` | Elements in `a` but not `b` | `{1,2,3} - {2,3}` → `{1}` |
| `symmetric_difference(b)` | `a ^ b` | Elements in exactly one set | `{1,2,3} ^ {2,3,4}` → `{1,4}` |
| `issubset(b)` | `a <= b` | Is `a` entirely contained in `b`? | `{1,2} <= {1,2,3}` → `True` |
| `issuperset(b)` | `a >= b` | Does `a` contain all of `b`? | `{1,2,3} >= {1,2}` → `True` |
| `isdisjoint(b)` | — | No overlap at all? | `{1,2}.isdisjoint({3,4})` → `True` |

```python
active_users = {"alice", "bob", "carol"}
premium_users = {"bob", "dave"}

active_users & premium_users   # {'bob'} -- active AND premium
active_users - premium_users   # {'alice', 'carol'} -- active, not premium
active_users | premium_users   # union of both groups
```

`frozenset` supports every **non-mutating** set method above (`union`,
`intersection`, `issubset`, etc.) but none of the mutating ones (`add`,
`remove`, `update`) — it trades mutability for hashability, which is why a
`frozenset` (unlike a `set`) can itself be an element of another set or a
dict key.

### Dictionaries — creation and methods

```python
user = {"name": "Ana", "age": 30}
user2 = dict(name="Ana", age=30)          # equivalent, kwargs form
user3 = dict.fromkeys(["a", "b"], 0)      # {'a': 0, 'b': 0}
```

| Method | Effect | Example | Result |
|---|---|---|---|
| `get(key, default=None)` | Look up `key`, return `default` instead of raising if missing | `user.get("email", "n/a")` | `"n/a"` |
| `setdefault(key, default)` | Return `user[key]` if present, else insert `default` and return it | `user.setdefault("age", 0)` | `30` (already present) |
| `update(other)` | Merge another dict/iterable of pairs in place | `user.update({"age": 31})` | `age` becomes `31` |
| `pop(key, default)` | Remove `key` and return its value (or `default` if absent) | `user.pop("age")` | returns `30`, key removed |
| `popitem()` | Remove and return the **last inserted** `(key, value)` pair | `user.popitem()` | LIFO order (3.7+) |
| `keys()` | View of all keys (live, reflects later changes) | `user.keys()` | `dict_keys([...])` |
| `values()` | View of all values | `user.values()` | `dict_values([...])` |
| `items()` | View of `(key, value)` pairs | `user.items()` | `dict_items([...])` |
| `clear()` | Remove all entries | `user.clear()` | `{}` |
| `copy()` | Shallow copy | `user.copy()` | new outer dict, same nested refs |

```python
# get() vs [] -- the single most common dict interview question
user = {"name": "Ana"}
user["email"]                 # KeyError: 'email'
user.get("email")             # None -- no exception
user.get("email", "unknown")  # "unknown" -- explicit default

# setdefault() for grouping/counting patterns
groups: dict[str, list[str]] = {}
for name in ["ana", "bo", "aria"]:
    groups.setdefault(name[0], []).append(name)
# {'a': ['ana', 'aria'], 'b': ['bo']}
```

```python
# Merging dicts (3.9+): the | and |= operators
defaults = {"timeout": 30, "retries": 3}
overrides = {"retries": 5}
config = defaults | overrides      # {'timeout': 30, 'retries': 5}
defaults |= overrides              # in-place merge, same result mutated in place

# Dict comprehension
squares = {n: n * n for n in range(5)}   # {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

```python
# Iterating a dict -- keys by default, or explicitly via items()/values()
for key in user:                 # iterates keys
    ...
for key, value in user.items():  # iterates (key, value) pairs -- most common
    ...
```

### `collections` module: purpose-built alternatives worth knowing

| Type | What it adds over the built-in | Typical use |
|---|---|---|
| `defaultdict(factory)` | Auto-creates a default value for a missing key instead of raising `KeyError` | Grouping/counting without `setdefault` boilerplate |
| `Counter(iterable)` | A dict subclass specialized for counting hashable items | Word frequency, histogram-style tallies |
| `deque` | A double-ended queue with O(1) appends/pops from **both** ends | Queues, sliding windows (a `list`'s `pop(0)`/`insert(0, x)` are O(n)) |

```python
from collections import defaultdict, Counter, deque

groups = defaultdict(list)
groups["a"].append("ana")   # no KeyError, no setdefault needed

word_counts = Counter("mississippi")
word_counts.most_common(2)  # [('i', 4), ('s', 4)]

queue = deque([1, 2, 3])
queue.appendleft(0)   # O(1), unlike list.insert(0, 0) which is O(n)
queue.popleft()        # O(1), unlike list.pop(0) which is O(n)
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
- `collections.defaultdict`/`Counter`: grouping or counting patterns that
  would otherwise need repetitive `setdefault`/manual counting logic.
- `collections.deque`: a queue or sliding-window buffer needing fast
  insertion/removal from both ends, where a `list` would be O(n).

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
- Confusing `sort()` (mutates in place, returns `None`) with `sorted()`
  (returns a new list) — writing `scores = scores.sort()` silently sets
  `scores` to `None`.
- Using `set()` where insertion order matters — regular dicts preserve
  insertion order since 3.7, but plain sets never guarantee any order.
- Writing `empty = {}` intending an empty set — that's an empty `dict`;
  the correct spelling is `set()`.

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

- What's the difference between `is` and `==`? When would `is` give a
    surprising result?

    **Answer:** `==` compares values; `is` compares object identity. `is`
    gets surprising when CPython reuses objects like small integers or interned
    strings, so two equal values may sometimes be the same object and sometimes not.

    ```python
    a: list[int] = [1, 2]
    b: list[int] = [1, 2]
    print(a == b)  # True
    print(a is b)  # False
    ```

- Why is a mutable default argument a bug? How do you fix it?

    **Answer:** The default object is created once at function definition time,
    so every call shares the same list or dict. Use `None` as the default and
    create a fresh object inside the function.

    ```python
    def add_tag(tag: str, tags: list[str] | None = None) -> list[str]:
       tags = [] if tags is None else tags
       tags.append(tag)
       return tags
    ```

- When would you choose a `tuple` over a `list`?

    **Answer:** Use a `tuple` when the shape should not change and immutability
    is part of the contract, like coordinates or composite dict keys. It also
    signals to other engineers that "this is a fixed record, not a work queue."

    ```python
    location: tuple[float, float] = (12.97, 77.59)
    cache_key: tuple[str, int] = ("user", 42)
    ```

- What's the time complexity of membership testing (`in`) for a `list` vs a
    `set`? Why?

    **Answer:** `x in list` is O(n) because Python may need to scan each item.
    `x in set` is O(1) on average because sets are hash tables.

    ```python
    allowed_ids: set[int] = {1, 2, 3}
    print(3 in allowed_ids)  # True
    ```

- Explain shallow vs deep copy with an example involving nested structures.

    **Answer:** A shallow copy creates a new outer container but keeps nested
    references shared; a deep copy clones nested objects too. That matters when
    mutating nested state like request payloads or ORM-ish dict trees.

    ```python
    import copy

    data: dict[str, list[int]] = {"ids": [1, 2]}
    shallow = data.copy()
    deep = copy.deepcopy(data)
    ```

- Why can't you use a `list` as a dictionary key?

    **Answer:** Dict keys must be hashable, and a `list` is mutable so its value
    can change after insertion. Python blocks that because changing the key would
    break the hash table's bookkeeping.

    ```python
    key: tuple[str, int] = ("user", 1)
    cache: dict[tuple[str, int], str] = {key: "hit"}
    ```

- What's the difference between `dict.get(key)` and `dict[key]` when the
    key is missing?

    **Answer:** `dict[key]` raises `KeyError`; `dict.get(key)` returns `None` or
    a default you provide. Use `get` when "missing" is expected, and `[]` when
    missing should be treated as a bug.

    ```python
    config: dict[str, str] = {"env": "prod"}
    print(config.get("region", "us-east-1"))
    ```

- What does `dict.setdefault()` do, and how is it useful for grouping
    items by a key?

    **Answer:** It returns the existing value for a key, or inserts a default
    and returns that. It's handy for grouping because you can create the bucket
    and append in one line.

    ```python
    grouped: dict[str, list[str]] = {}
    for name in ["ana", "bo", "aria"]:
       grouped.setdefault(name[0], []).append(name)
    ```

- What's the difference between `list.sort()` and the built-in
    `sorted()`?

    **Answer:** `list.sort()` mutates the list in place and returns `None`;
    `sorted()` returns a new list and works with any iterable. Use `sorted()`
    when you need the original order preserved.

    ```python
    scores: list[int] = [3, 1, 2]
    ordered = sorted(scores)
    print(scores, ordered)
    ```

- When would you reach for `collections.defaultdict` or
    `collections.Counter` instead of a plain `dict`?

    **Answer:** `defaultdict` is good when missing keys should auto-create a
    bucket; `Counter` is good when the job is counting occurrences. They remove
    repetitive guard code and make intent obvious.

    ```python
    from collections import Counter

    counts: Counter[str] = Counter(["ok", "ok", "fail"])
    print(counts["ok"])  # 2
    ```

- Why is `deque` preferred over `list` for a queue that needs to pop
    from the front frequently?

    **Answer:** `list.pop(0)` is O(n) because all remaining items shift left.
    `deque.popleft()` is O(1), so it stays fast under queue-like workloads.

    ```python
    from collections import deque

    queue: deque[int] = deque([1, 2, 3])
    print(queue.popleft())  # 1
    ```

## Senior-level considerations

- Mutability and identity semantics directly explain ORM identity maps
  (SQLAlchemy returns the *same* Python object for the same primary key
  within a session) and why `==` on ORM models is often overridden. Example:
  two queries for `User(id=1)` in the same session usually compare with
  `user1 is user2`.
- Choosing `frozenset`/`tuple` for immutable shared state avoids
  accidental mutation bugs in concurrent code — important once you start
  reasoning about asyncio tasks or multiprocessing sharing data. Example: keep
  feature flags as `frozenset({"beta-dashboard", "search-v2"})` so tasks can
  read them without one task mutating another's view.
- At scale, container choice affects memory footprint: a `dict` has higher
  per-entry overhead than a `list`; for large read-heavy datasets consider
  `array`, `__slots__`, or external stores instead of naive dict/list use.
  Example: storing 10 million small counters as dict entries is much heavier
  than a compact numeric array indexed by ID.
- Knowing the time complexity of each method matters as much as knowing it
  exists — `list.insert(0, x)` and `list.pop(0)` are O(n) because every
  remaining element shifts, which is exactly the gap `collections.deque`
  closes with O(1) operations at both ends. Example: a worker queue that
  processes 50k jobs per minute should use `deque`, not repeatedly `pop(0)`
  from a list.
