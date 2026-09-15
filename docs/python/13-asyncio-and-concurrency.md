# Asyncio and Concurrency

**Level:** Advanced

## What

`asyncio` is Python's standard library framework for writing concurrent
code using `async`/`await` coroutines scheduled on a single-threaded event
loop. This topic covers coroutines, tasks, common concurrency primitives,
and how `asyncio` relates to threading and multiprocessing (see
[GIL and CPython](12-gil-and-cpython.md) for *why* these models behave
differently).

## Why

Modern Python backends (FastAPI, async DB drivers, async HTTP clients) are
built around `asyncio`. Understanding it correctly is essential to avoid
the most common production issue in async services: a single blocking call
silently stalling every other concurrent request on the same event loop.

## How

### Coroutines and `await`

```python
import asyncio

async def fetch_user(user_id: int) -> dict:
    await asyncio.sleep(0.1)  # simulates an I/O-bound call
    return {"id": user_id}

async def main() -> None:
    user = await fetch_user(1)
    print(user)

asyncio.run(main())
```

Calling `fetch_user(1)` doesn't execute the function — it returns a
coroutine object. `await` drives the coroutine, yielding control back to
the event loop whenever it hits an `await` on something that isn't ready
yet (e.g. a socket read), letting other coroutines run in the meantime.

### Running things concurrently: `asyncio.gather` and `TaskGroup`

```python
async def main() -> None:
    results = await asyncio.gather(
        fetch_user(1),
        fetch_user(2),
        fetch_user(3),
    )
    print(results)  # runs all three concurrently, not sequentially
```

```python
# Python 3.11+ -- structured concurrency, propagates exceptions as an
# ExceptionGroup and cancels sibling tasks on failure
async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch_user(1))
        tg.create_task(fetch_user(2))
```

`TaskGroup` is version-dependent (3.11+) and is generally preferred over
manually tracking tasks — it guarantees all child tasks are awaited and
cleans up properly on error, unlike unmanaged `asyncio.create_task()`
calls.

### Tasks vs coroutines

```python
async def main() -> None:
    task = asyncio.create_task(fetch_user(1))  # scheduled immediately
    # ... do other work while it runs ...
    result = await task
```

A coroutine object alone does nothing until awaited or wrapped in a task.
`asyncio.create_task()` schedules it to run concurrently on the event
loop *now*, rather than only when awaited directly.

### Common pitfall: blocking the event loop

```python
import time

async def bad_handler() -> None:
    time.sleep(2)  # BLOCKS the entire event loop -- nothing else runs

async def good_handler() -> None:
    await asyncio.sleep(2)  # yields control back to the event loop
```

Any synchronous, blocking call (`time.sleep`, a synchronous DB driver, CPU-
heavy computation) inside an `async def` function blocks the **entire**
event loop, stalling every other concurrent coroutine — not just the
current request. This is the single most common async production bug.

### Offloading blocking work

```python
async def handler() -> dict:
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, cpu_heavy_function, arg)
    return result
```

`run_in_executor` (or a library-provided equivalent, e.g. FastAPI running
sync `def` route handlers in a thread pool automatically) moves blocking
work off the event loop into a thread pool, so the loop stays responsive.

### Synchronization primitives

```python
lock = asyncio.Lock()

async def update_shared_state() -> None:
    async with lock:
        # only one coroutine at a time executes this block
        ...
```

`asyncio.Lock`, `asyncio.Semaphore`, and `asyncio.Queue` mirror their
`threading` counterparts but are designed for cooperative, single-threaded
concurrency — they coordinate coroutines, not OS threads.

### Concurrency vs parallelism, and where `multiprocessing` fits

- **Concurrency** (`asyncio`, threads): multiple tasks make progress by
  interleaving, often on one thread — ideal for I/O-bound work.
- **Parallelism** (`multiprocessing`): multiple tasks run *simultaneously*
  on multiple CPU cores — needed for CPU-bound work, since the GIL
  prevents true parallel bytecode execution within one process (see
  [GIL and CPython](12-gil-and-cpython.md)).

```python
from concurrent.futures import ProcessPoolExecutor

async def handler() -> int:
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        return await loop.run_in_executor(pool, cpu_bound_function, data)
```

Combining `asyncio` (for I/O concurrency) with `ProcessPoolExecutor` (for
CPU-bound parallelism) is a common pattern in real services that need
both.

## When to use

- `asyncio` for I/O-bound backend workloads: HTTP calls, async DB drivers,
  many concurrent connections — the default for modern FastAPI services.
- `TaskGroup`/`gather` when you need multiple independent I/O operations to
  run concurrently instead of sequentially (e.g. fanning out to several
  downstream services).
- `run_in_executor`/`ProcessPoolExecutor` when a handler must do
  CPU-bound or unavoidably blocking work without stalling the event loop.

## When NOT to use

- Don't use `asyncio` for CPU-bound work expecting a speedup — it doesn't
  provide parallelism; use `multiprocessing` instead.
- Don't mix blocking synchronous libraries directly inside `async def`
  functions (e.g. a sync DB driver, `requests` instead of `httpx`'s async
  client) — always use async-native libraries or offload to an executor.
- Don't manually manage bare `asyncio.create_task()` calls without keeping
  a reference or handling cancellation/errors — prefer `TaskGroup` for
  structured concurrency where available.

## Common mistakes

- Blocking the event loop with a synchronous call inside `async def` (see
  above) — the most common and impactful async bug in production FastAPI
  services.
- Forgetting to `await` a coroutine, silently creating a "coroutine was
  never awaited" warning and skipping the work entirely.
- Losing a reference to a task created with `asyncio.create_task()`,
  letting it be garbage collected before completion (the "fire and forget"
  task trap) — store a reference or use a `TaskGroup`.
- Assuming `asyncio.gather`/`TaskGroup` execution is thread-safe shared-
  memory parallelism — it's still single-threaded cooperative concurrency;
  shared mutable state can still cause bugs if a coroutine is interrupted
  mid-operation by another at an `await` point.

## Interview questions

1. What's the difference between a coroutine object and a task in
   `asyncio`?
2. Why does calling a blocking function like `time.sleep()` inside an
   `async def` function affect the entire application, not just the
   current request?
3. When would you use `multiprocessing` instead of (or alongside)
   `asyncio`?
4. What does `asyncio.TaskGroup` provide over manually tracking tasks with
   `create_task`?
5. How would you offload CPU-bound work from an async FastAPI handler
   without blocking the event loop?
6. What's the "fire and forget" task bug, and how do you avoid it?

## Senior-level considerations

- Diagnosing event-loop starvation in production (p99 latency spikes
  across *unrelated* endpoints) is a key async debugging skill — it often
  traces back to one handler doing blocking work; instrumentation (event
  loop lag monitoring) helps catch this before it becomes a customer-facing
  incident.
- Structured concurrency (`TaskGroup`) is a deliberate design choice to
  avoid orphaned tasks and inconsistent error handling — prefer it over ad
  hoc `create_task` usage in new code where your Python version supports
  it (3.11+).
- Async code composes with the iterator protocol via async generators
  (`async def` + `yield`, consumed with `async for`) — the same lazy,
  memory-efficient principles from
  [Iterators and Generators](08-iterators-and-generators.md) apply when
  streaming large async result sets (e.g. paginated downstream API calls).
