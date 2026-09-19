# Application Performance Fundamentals

## What

A system-level framework for reasoning about backend performance: where
to look first, how CPU-bound and I/O-bound bottlenecks require different
fixes, and how the individual techniques covered elsewhere in this
handbook (profiling, async, caching, connection pooling) fit together
into one coherent approach.

## Why

Performance work without a framework tends to become guessing —
optimizing whatever looks slow at a glance, or applying a fix (like
"just make it async") to a problem it doesn't actually address. Measuring
first, correctly classifying the bottleneck, and choosing the fix that
matches the classification is what separates effective performance work
from wasted effort.

## How

### Measure before optimizing

```python
# Don't guess where the time goes -- profile it (see Performance and
# Profiling for cProfile/tracemalloc/line-profiler mechanics)
python -m cProfile -o profile.stats app.py
```

The single most common performance-work mistake is optimizing code that
isn't actually the bottleneck — see
[Performance and Profiling](../python/14-performance-and-profiling.md)
for the concrete tools; the discipline that matters here is: profile
first, always, before changing anything.

### The CPU-bound vs I/O-bound decision tree

```
Where is time actually going (per the profiler)?

  Mostly in socket/select/ssl/db-driver internals
    -> I/O-bound: use async/await or more workers; the CPU is mostly idle
       waiting on the network/disk

  Mostly in your own computation (loops, parsing, serialization)
    -> CPU-bound: async/await won't help (see the GIL); needs algorithmic
       improvement or multiprocessing to actually parallelize
```

This diagnosis step, covered mechanically in
[Performance and Profiling](../python/14-performance-and-profiling.md#cpu-bound-vs-io-bound-diagnosis),
is the fork in the road that determines whether the right fix is
`asyncio`, `multiprocessing`, or an algorithmic change — applying the
wrong one (e.g. going async on genuinely CPU-bound code) yields no
improvement, since the GIL prevents true parallel execution within one
process (see [GIL and CPython](../python/12-gil-and-cpython.md)).

### Async performance: where it actually helps

```python
# I/O-bound: async concurrency lets many waiting requests overlap
async def handle_request():
    result = await call_downstream_api()  # the event loop serves other
    return result                          # requests while this awaits
```

Async concurrency's entire value proposition is letting the event loop
do other work while one coroutine is waiting on I/O (see
[Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md)) — it
doesn't make any individual operation faster; it improves *throughput*
under concurrent load by not leaving the CPU idle during waits.

### The most common backend performance bottlenecks, ranked

```
1. N+1 database queries        -- see Queries, Relationships, and Loading
2. Missing/wrong indexes       -- see Indexes and Query Optimization
3. Synchronous I/O blocking     -- see Asyncio and Concurrency
   the event loop in async code
4. No caching for repeated,     -- see the Caching section
   expensive reads
5. Undersized/oversized          -- see Transactions and Connection Pools
   connection pools
```

In practice, database-related issues (N+1 queries, missing indexes) are
the most common source of real backend latency — before reaching for
more exotic fixes, verifying these fundamentals first has the highest
hit rate.

### Latency budgets

```
Total acceptable request latency: 200ms
  - Auth/validation:        5ms
  - Database query:        50ms   <- budget, not just "as fast as possible"
  - Downstream API call:  100ms
  - Serialization:          5ms
  - Buffer/margin:          40ms
```

Thinking in terms of a latency budget per request — how much time each
stage is *allowed* to take — turns "make it faster" into a concrete,
prioritized target: the stage furthest over its budget is where
optimization effort should go first.

### Vertical vs horizontal scaling as a performance lever

```
Vertical:   bigger/faster single instance (more CPU, more RAM)
Horizontal: more instances behind a load balancer
```

Scaling is sometimes the right "fix" when a system is already
reasonably efficient and simply needs more capacity — but scaling
inefficient code (e.g. one with an unresolved N+1 query pattern) just
multiplies database load across more instances rather than fixing the
underlying problem; algorithmic/query fixes should generally come before
reaching for more hardware.

### Premature optimization vs deliberate performance work

```python
# Premature: micro-optimizing a rarely-called, already-fast function
# based on a guess, without profiling data

# Deliberate: profiling shows this specific function actually accounts
# for 40% of request latency under realistic load -- worth optimizing
```

"Premature optimization is the root of all evil" doesn't mean never
optimize — it means don't optimize *without evidence* that the code in
question is actually a meaningful contributor to real-world latency or
cost.

## When to use

- Profile first, on realistic workloads, before any performance
  optimization effort.
- Classify a bottleneck as CPU-bound or I/O-bound before choosing between
  async concurrency and multiprocessing/algorithmic fixes.
- Check the most common bottlenecks (N+1 queries, missing indexes,
  blocking I/O in async code, missing caching) before assuming a more
  exotic cause.

## When NOT to use

- Don't optimize code based on intuition alone without profiling data
  confirming it's a real bottleneck under realistic load.
- Don't reach for `asyncio` to fix a CPU-bound bottleneck — it won't help,
  since the GIL prevents true parallel CPU work within one process.
- Don't scale horizontally to work around an unresolved N+1 query or
  missing index — it multiplies the underlying inefficiency across more
  instances rather than fixing it.

## Common mistakes

- Optimizing based on assumption rather than profiler output, often
  improving something that wasn't actually the bottleneck.
- Applying `asyncio` to CPU-bound code expecting a speedup, then being
  confused when there isn't one.
- Treating "add more instances" as a universal fix, masking (and scaling
  the cost of) an underlying inefficient query or algorithm.
- No latency budget or target — "make it faster" without a concrete goal
  makes it hard to know when performance work is actually done.

## Interview questions

- Why does adding `async`/`await` not help a CPU-bound bottleneck, even
    though it helps I/O-bound code significantly?

    **Answer:** `async`/`await` lets other work run while one task is
    *waiting* (I/O) — it doesn't make CPU instructions execute faster or
    in parallel. A CPU-bound task keeps the single event loop thread busy
    the whole time, so there's no waiting for anything else to fill.

- Walk through how you'd diagnose whether a slow endpoint is CPU-bound
    or I/O-bound.

    **Answer:** Profile it (`cProfile`/py-spy) and look at where time is
    actually spent: high time inside your own Python code (loops,
    computation) points to CPU-bound; high time spent waiting on
    DB/HTTP/file calls points to I/O-bound. A quick sanity check: CPU-bound
    work pegs a CPU core near 100%; I/O-bound work leaves the CPU mostly
    idle while waiting.

- Why should horizontal scaling generally come *after*, not instead of,
    fixing an N+1 query or missing index?

    **Answer:** Scaling adds more instances to handle the same
    inefficient work, multiplying infrastructure cost without fixing the
    root cause — an N+1 query fixed once benefits every request forever,
    for free, versus paying for more servers indefinitely to paper over it.

- What is a latency budget, and how does it help prioritize performance
    work?

    **Answer:** A latency budget is the total time allowed for a request
    (e.g. 200ms), broken down across each step (DB query, external call,
    serialization). It helps prioritize by showing which piece is eating
    the most of the budget — that's where optimization actually moves the
    needle.

- What are the most common sources of backend performance problems in
    practice, and why do database-related issues usually top the list?

    **Answer:** N+1 queries, missing indexes, and unbounded result sets
    are the usual suspects. The database tends to dominate because it's
    the one component doing disk I/O and lock coordination — CPU-bound
    Python code is comparatively rare in typical CRUD-style backend
    services.

## Senior-level considerations

- Performance work should be data-driven and prioritized by actual
  impact (profiler output, latency budgets, cost) — not by intuition or
  by whichever piece of code is most interesting to optimize. For
  example, spending a week optimizing a rarely-called function while a
  hot endpoint's N+1 query goes unnoticed is effort spent in the wrong
  place.
- Correctly diagnosing CPU-bound vs I/O-bound bottlenecks — and knowing
  which fix (async, multiprocessing, algorithmic change, caching, more
  hardware) matches which diagnosis — is a foundational skill that
  prevents wasted engineering effort on the wrong solution. For example,
  adding more `asyncio` concurrency to a CPU-bound image-resizing endpoint
  won't help; offloading it to a process pool will.
- Performance and cost are often the same conversation at scale — an
  inefficient query or algorithm that "just" needs more hardware to keep
  up is really a recurring infrastructure cost that a fix would
  eliminate; framing performance work this way helps prioritize it
  against feature work. For example, "this N+1 fix saves $2k/month in
  database instance costs" is a more compelling prioritization argument
  than "this code is slow."
