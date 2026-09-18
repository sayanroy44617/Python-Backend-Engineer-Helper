# Performance and Profiling

**Level:** Advanced

## What

Techniques and tools for measuring where a Python program actually spends
time and memory, rather than guessing — `timeit`, `cProfile`,
`tracemalloc`, and the discipline of profiling before optimizing.

## Why

Optimization without measurement usually wastes effort on code that isn't
the actual bottleneck. In backend systems, the real bottleneck is often
outside your Python code entirely (a slow DB query, an N+1 query pattern, a
downstream API call) — profiling tells you where to actually spend
optimization effort, and whether the fix belongs in code, in the database,
or in architecture (caching, batching).

## How

### Micro-benchmarking with `timeit`

```python
import timeit

timeit.timeit("[x**2 for x in range(1000)]", number=1000)
timeit.timeit("list(map(lambda x: x**2, range(1000)))", number=1000)
```

`timeit` runs a snippet many times and reports total/average time,
controlling for common measurement noise (e.g. disabling the GC during
runs). Use it for small, isolated comparisons — not for profiling a whole
application.

### Profiling a whole call with `cProfile`

```python
import cProfile
import pstats

def slow_function() -> None:
    total = 0
    for i in range(1_000_000):
        total += i * i

profiler = cProfile.Profile()
profiler.enable()
slow_function()
profiler.disable()

stats = pstats.Stats(profiler).sort_stats("cumulative")
stats.print_stats(10)  # top 10 functions by cumulative time
```

```bash
python -m cProfile -s cumulative my_script.py
```

`cProfile` reports per-function call counts and time (own time vs
cumulative time including callees) — the standard first step when you know
*a* function is slow but not which part.

### Memory profiling with `tracemalloc`

```python
import tracemalloc

tracemalloc.start()

# ... code that might be leaking or using too much memory ...
data = [str(i) for i in range(1_000_000)]

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")
for stat in top_stats[:5]:
    print(stat)
```

`tracemalloc` tracks memory allocations by source location, useful for
finding unexpected memory growth (e.g. an accumulating cache, a large
object retained longer than expected — see
[Memory Management](11-memory-management.md)).

### Line-level profiling

```bash
pip install line_profiler
kernprof -l -v my_script.py
```

`cProfile` only reports per-function timings; a line profiler (e.g.
`line_profiler`, decorate target functions with `@profile`) shows time
spent per line — useful once `cProfile` has narrowed down which function
to look at more closely.

### Common backend-specific bottlenecks

```python
# N+1 query pattern -- one query per order to fetch its items
for order in orders:
    items = get_items_for_order(order.id)  # N additional queries

# Fix: batch-fetch in one query
items_by_order = get_items_for_orders([o.id for o in orders])
```

Profiling application code alone won't reveal this — you also need to
inspect actual SQL query counts/timing (e.g. via SQLAlchemy's echo mode or
an APM tool), since the "slow" time is spent waiting on the database, not
executing Python.

### CPU-bound vs I/O-bound diagnosis

```python
import cProfile
# If cProfile shows most time in `socket`/`select`/`ssl` internals,
# the code is I/O-bound -- async/concurrency helps.
# If it shows most time in your own computation functions,
# it's CPU-bound -- algorithmic improvements or multiprocessing help.
```

Distinguishing CPU-bound from I/O-bound is the deciding factor for whether
`asyncio`, threads, or `multiprocessing` is the right fix (see
[GIL and CPython](12-gil-and-cpython.md)).

## When to use

- Profile (`cProfile`) before optimizing anything beyond obviously bad
  algorithmic complexity — measure first, always.
- `tracemalloc` when investigating unexplained memory growth in a
  long-running process.
- `timeit` for quick, isolated comparisons between two implementations of a
  small operation.
- Load testing (e.g. with `locust`/`k6`, covered under System Design/
  Performance more broadly) to find bottlenecks under realistic concurrent
  traffic, not just single-request timing.

## When NOT to use

- Don't optimize code based on intuition alone ("this loop looks slow") —
  verify with a profiler; intuition about performance is frequently wrong,
  especially across different data sizes.
- Don't micro-optimize Python-level code when the actual bottleneck is a
  database query, network call, or lock contention — profile the whole
  request path, not just CPU time in isolated functions.
- Don't leave profiling instrumentation (`cProfile.Profile()`, `print`
  timing statements) in production code paths — profiling has overhead and
  belongs in dedicated investigation/benchmarking, or lightweight ongoing
  APM/tracing instead.

## Common mistakes

- Assuming an operation is slow without measuring it (premature
  optimization on the wrong target).
- Profiling in an environment that doesn't match production (e.g. a
  different data volume, no simulated network latency) and drawing the
  wrong conclusion.
- Ignoring the N+1 query problem because it doesn't show up as "slow
  Python code" in a naive profile — it shows up as many small DB round
  trips, which application-level profiling alone won't surface.
- Confusing wall-clock time with CPU time when profiling I/O-bound
  code — high wall-clock time with low CPU time usually means the fix is
  concurrency (async/threads), not algorithmic optimization.

## Interview questions

1. Why should you profile before optimizing? Give an example of an
   intuition-driven optimization that turned out to be wrong.

   **Answer:** Because the obvious-looking slow code is often not the real
   bottleneck. A common miss is rewriting a Python loop for speed when the
   endpoint is actually spending most of its time waiting on the database.
2. What's the difference between what `cProfile` and `tracemalloc` each
   measure?

   **Answer:** `cProfile` shows where execution time goes, function by
   function. `tracemalloc` shows where memory allocations come from, which
   is what you want when memory keeps climbing.
3. How would you diagnose whether a slow endpoint is CPU-bound or
   I/O-bound?

   **Answer:** Profile the request and look at where time is spent: if it
   is mostly your computation functions, it is CPU-bound; if it is mostly
   socket, DB, HTTP, or waiting time, it is I/O-bound. Pair Python
   profiling with SQL/query timing or tracing so you do not miss external
   waits.
4. What is the N+1 query problem, and why might it not show up clearly in
   a Python-level profiler?

   **Answer:** It's when code does one query to load a set of rows and then
   one extra query per row to load related data. Python profiling just sees
   "waiting on DB" repeated many times, so you need query logs or APM to
   spot the pattern cleanly.

   ```python
   for user_id in user_ids:
       load_orders(user_id)  # one extra DB call per user
   ```
5. What's the difference between micro-benchmarking (`timeit`) and
   whole-program profiling (`cProfile`)?

   **Answer:** `timeit` is for tiny isolated comparisons, like two ways to
   build the same list. `cProfile` is for understanding where a real call
   path spends time across multiple functions.

## Senior-level considerations

- In production systems, ongoing observability (tracing, APM, structured
  logs with timing) matters more than one-off profiling sessions — you
  need to know *when* a regression happens, not just be able to
  investigate it after a complaint; for example, a trace can show that p99
  latency jumped right after a new downstream API integration shipped.
- Performance work should be driven by SLOs/latency budgets (e.g. p95/p99
  targets), not by chasing every possible micro-optimization — know when
  "fast enough" has been reached; for example, shaving 2 ms off a helper
  function is noise if the endpoint already meets its 150 ms p95 target.
- The biggest wins in backend performance are usually architectural
  (caching, batching, reducing round trips, connection pooling, async I/O)
  rather than Python-level micro-optimization — profiling helps you find
  which of those applies, rather than optimizing the wrong layer; for
  example, collapsing 50 SQL round trips into 2 usually beats tuning a
  list comprehension.
