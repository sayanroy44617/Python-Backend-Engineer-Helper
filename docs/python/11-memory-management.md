# Memory Management

**Level:** Advanced

## What

CPython manages memory automatically via **reference counting** plus a
supplementary **garbage collector** for cyclic references. This topic
covers how object lifetime is determined, why some objects leak memory
unexpectedly, and how to inspect memory behavior.

## Why

Backend services that run for a long time (workers, API servers) are
sensitive to memory leaks and unbounded growth. Understanding reference
counting explains *why* most Python objects are freed immediately when
unused, while understanding the cyclic garbage collector explains the
common exception — reference cycles (e.g. parent/child object graphs,
caches) that reference counting alone can't clean up.

## How

### Reference counting

Every object has a reference count, incremented when a new reference is
created and decremented when a reference goes out of scope, is reassigned,
or is deleted. When it hits zero, CPython frees the object **immediately**
— this is why simple objects are usually deallocated deterministically
rather than at some unpredictable future GC pass.

```python
import sys

a = []
print(sys.getrefcount(a))  # includes the temporary ref from getrefcount's own argument

b = a
print(sys.getrefcount(a))  # increased by one -- `b` also references it

del b
print(sys.getrefcount(a))  # back down
```

### Reference cycles

Reference counting alone cannot free objects that reference each other in
a cycle, since each still has a non-zero count even when unreachable from
the rest of the program.

```python
class Node:
    def __init__(self) -> None:
        self.parent = None
        self.children = []

root = Node()
child = Node()
root.children.append(child)
child.parent = root  # cycle: root -> child -> root

del root
del child
# neither refcount reaches zero due to the cycle -- the generational
# garbage collector is needed to reclaim them
```

### The generational garbage collector

CPython's `gc` module implements a generational, cycle-detecting collector
that runs periodically (based on allocation thresholds) to find and
collect reference cycles that pure refcounting misses.

```python
import gc

gc.collect()          # force a collection pass
gc.get_threshold()     # (700, 10, 10) by default -- generation thresholds
gc.get_stats()          # per-generation collection stats
```

Objects are grouped into 3 generations; young objects are collected more
frequently (most objects die young — the generational hypothesis),
promoted to older generations if they survive a collection.

### Breaking cycles: `weakref`

```python
import weakref

class Node:
    def __init__(self) -> None:
        self.parent: "weakref.ReferenceType[Node] | None" = None
        self.children: list["Node"] = []

root = Node()
child = Node()
root.children.append(child)
child.parent = weakref.ref(root)  # doesn't increase root's refcount
```

A `weakref` lets one side of a parent/child relationship reference the
other without contributing to its reference count, avoiding the cycle
entirely — a common pattern for caches and observer-style back-references.

### `__slots__`

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
```

By default, instances store attributes in a per-instance `__dict__`, which
has real memory overhead. `__slots__` replaces that dict with fixed,
fixed-size storage — meaningfully reducing memory for classes with many
short-lived instances (e.g. millions of small data objects), at the cost
of losing dynamic attribute assignment and some flexibility (no default
multiple inheritance with other `__slots__` classes, no `__dict__` unless
added back explicitly).

## When to use

- Reach for `gc` module functions (`gc.collect()`, `gc.get_stats()`,
  `gc.disable()`) only when actively investigating a memory issue — not as
  routine code.
- Use `weakref` for cache/observer/back-reference relationships where you
  don't want to keep an object alive artificially or create a cycle.
- Use `__slots__` for high-volume, attribute-fixed classes where memory
  footprint matters (e.g. millions of parsed records held in memory).

## When NOT to use

- Don't manually call `gc.collect()` in normal request-handling code — it's
  a diagnostic/rare-tuning tool, not something to sprinkle through business
  logic; doing so can hurt performance (forces a full scan).
- Don't disable the GC (`gc.disable()`) unless you've profiled and
  confirmed cycles aren't accumulating — disabling it blindly risks slow,
  unbounded memory growth in long-running processes.
- Don't add `__slots__` to every class by default — it removes the
  per-instance `__dict__`, which some libraries (ORMs, mocking frameworks)
  rely on; apply selectively where profiling shows it matters.

## Common mistakes

- Assuming Python "never leaks memory" because it's garbage collected —
  reference cycles combined with objects that define `__del__` used to be
  uncollectable in older Python versions (fixed since 3.4, but still worth
  knowing this was version-dependent history).
- Holding long-lived caches (e.g. a module-level `dict`) that grow forever
  because nothing ever evicts old entries — this is a "leak" even though
  the GC works correctly; it's an application logic problem, not a GC bug.
- Creating accidental reference cycles via closures or callbacks that
  capture `self`, delaying collection longer than expected in an
  otherwise-reference-counted object graph.

## Interview questions

1. How does CPython decide when to free an object with plain reference
   counting? What does that guarantee, and what does it not handle?
2. Why does a parent/child object graph with back-references need the
   cyclic garbage collector?
3. What's the purpose of `weakref`, and when would you use it?
4. What memory trade-off does `__slots__` make, and when is it worth it?
5. Why shouldn't you routinely call `gc.collect()` in application code?

## Senior-level considerations

- In long-running services (workers, API servers), gradual memory growth
  is often caused by application-level caches, not GC bugs — profile with
  `tracemalloc` or `objgraph` before assuming it's a garbage collector
  issue.
- Memory management interacts with the GIL (see
  [GIL and CPython](12-gil-and-cpython.md)): reference count updates
  themselves need to be thread-safe, which is one of the historical
  reasons CPython's GIL exists.
- For memory-sensitive, high-throughput services, `__slots__` and careful
  object lifetime management (avoiding accidental cycles/leaks) can
  meaningfully reduce container memory footprint and GC pause frequency —
  relevant when tuning Kubernetes memory limits/requests.
