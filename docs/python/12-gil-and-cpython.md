# GIL and CPython

**Level:** Advanced

## What

The **GIL** (Global Interpreter Lock) is a mutex in CPython (the reference
Python implementation) that allows only one thread to execute Python
bytecode at a time. This topic covers what the GIL actually protects, why
it exists, and how CPython's execution model shapes real concurrency
choices for backend services.

## Why

Whether to use threads, `asyncio`, or multiprocessing for a given backend
workload depends entirely on understanding the GIL. Getting this wrong
leads to real production issues: using threads for CPU-bound work expecting
a speedup that never materializes, or over-engineering multiprocessing for
I/O-bound work that `asyncio`/threads already handle well.

## How

### What the GIL protects

The GIL exists primarily to make CPython's memory management (reference
counting, see [Memory Management](11-memory-management.md)) thread-safe
without requiring fine-grained locks on every object. Only one thread runs
Python bytecode at a time; the interpreter periodically releases the GIL
(by default, on a time-based check interval) so other threads get a turn.

```python
import threading
import time

def cpu_bound(n: int) -> int:
    total = 0
    for i in range(n):
        total += i * i
    return total

start = time.perf_counter()
threads = [threading.Thread(target=cpu_bound, args=(20_000_000,)) for _ in range(4)]
[t.start() for t in threads]
[t.join() for t in threads]
print(time.perf_counter() - start)
# Roughly the same (or worse) as running cpu_bound sequentially 4 times --
# the GIL prevents true parallel execution of Python bytecode.
```

### I/O-bound work releases the GIL

Blocking I/O operations (file/network reads, `time.sleep`, most C
extension calls doing I/O) release the GIL while waiting, allowing other
threads to run. This is why **threads help for I/O-bound work** even
though they don't help for CPU-bound work.

```python
import threading
import requests  # I/O-bound: releases the GIL while waiting on the network

def fetch(url: str) -> None:
    requests.get(url)

threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
[t.start() for t in threads]
[t.join() for t in threads]
# Threads genuinely overlap here because each blocks on network I/O,
# releasing the GIL for others to run.
```

### GIL vs `asyncio` vs `multiprocessing`

| Approach | Concurrency model | Helps CPU-bound? | Helps I/O-bound? |
|---|---|---|---|
| Threads (`threading`) | OS threads, GIL-limited | No | Yes |
| `asyncio` | Single-threaded event loop, cooperative | No | Yes (often more efficient than threads for high-volume I/O) |
| `multiprocessing` | Separate OS processes, separate GIL each | Yes | Yes (but heavier overhead) |

`asyncio` doesn't bypass the GIL — it avoids needing multiple OS threads
at all for I/O-bound concurrency, using a single thread that cooperatively
switches between coroutines while they're waiting on I/O.
`multiprocessing` sidesteps the GIL entirely because each process has its
own interpreter and its own GIL.

### CPython basics relevant to backend work

- Python source is compiled to **bytecode** (`.pyc` files), executed by the
  CPython **bytecode interpreter** — a stack-based virtual machine. You can
  inspect bytecode with `dis.dis(func)`.
- The reference implementation (CPython) is what almost all production
  backend deployments use; alternative implementations (PyPy, with a JIT
  and different GC/GIL characteristics) exist but are far less common for
  typical FastAPI/SQLAlchemy stacks — mentioned here because "Python's
  behavior" is sometimes implementation-specific, not language-specific.
- The **GIL is a CPython implementation detail**, not part of the Python
  language specification — this is why alternative implementations (and
  ongoing CPython work like PEP 703's optional "free-threaded" build) can
  have different concurrency characteristics. This is actively
  version-dependent — check your target Python version/build if this
  matters for your deployment.

## When to use

- Threads: I/O-bound concurrency with existing blocking libraries, or
  interfacing with C extensions that release the GIL.
- `asyncio`: high-volume I/O-bound concurrency (many concurrent
  requests/connections) where the overhead of OS threads would be
  prohibitive — the default choice for modern async web frameworks
  (FastAPI's async endpoints).
- `multiprocessing`: genuinely CPU-bound work (data processing, image/ML
  inference, heavy computation) that needs to use multiple cores.

## When NOT to use

- Don't use `threading` expecting a speedup for CPU-bound work — the GIL
  prevents true parallel bytecode execution; use `multiprocessing` or
  push the work to a C extension/external service instead.
- Don't reach for `multiprocessing` for I/O-bound work — the process
  startup/IPC overhead is usually far worse than threads or asyncio for
  that workload.
- Don't assume `asyncio` gives you parallelism — it gives you
  concurrency on a single thread; a CPU-bound `async def` handler still
  blocks the entire event loop until it returns.

## Common mistakes

- Writing CPU-bound code inside an `async def` FastAPI handler without
  offloading it (e.g. to a thread pool or process pool), which blocks the
  entire event loop and stalls all other concurrent requests.
- Assuming multithreading in Python behaves like Java/Go, where CPU-bound
  work actually parallelizes across cores.
- Not knowing that `numpy`/many C extensions release the GIL internally
  during heavy computation — so **some** "CPU-bound" work in threads does
  benefit, depending on the library.

## Interview questions

- What does the GIL actually protect, and why does CPython have one?

    **Answer:** The GIL makes CPython's interpreter state and reference
    counting safe by allowing only one thread to run Python bytecode at a
    time. It exists mostly to keep the runtime simpler and object memory
    management thread-safe.

- Why do threads help I/O-bound Python code but not CPU-bound code?

    **Answer:** I/O-bound threads spend most of their time waiting on the
    network, disk, or sleep calls, and those waits release the GIL so other
    threads can run. CPU-bound Python code keeps wanting the GIL to execute
    bytecode, so threads mostly take turns instead of using multiple cores.

- How does `asyncio` achieve concurrency without multiple OS threads?

    **Answer:** `asyncio` runs one event loop that switches between
    coroutines when they hit an `await` on I/O. The concurrency comes from
    cooperative scheduling, not parallel CPU execution.

    ```python
    import asyncio

    async def main() -> None:
       await asyncio.gather(asyncio.sleep(1), asyncio.sleep(1))

    asyncio.run(main())
    ```

- When would you choose `multiprocessing` over `asyncio` or threads?

    **Answer:** Use `multiprocessing` when the bottleneck is real CPU work
    and you need multiple cores, like image processing or large batch
    computations. Each process gets its own interpreter and GIL, so the
    work can run in parallel.

- Is the GIL part of the Python language spec, or a CPython
    implementation detail? Why does that distinction matter?

    **Answer:** The GIL is a CPython runtime detail, not a Python language
    guarantee. That matters because different implementations or newer
    CPython builds can have different threading behavior.

- What happens if you run CPU-bound work inside an `async def` FastAPI
    route without offloading it?

    **Answer:** You block the event loop, so other requests stop making
    progress until that CPU-heavy code finishes. The route is `async` in
    syntax only — operationally it behaves like a single-threaded stall.

## Senior-level considerations

- Diagnosing "my async FastAPI service feels slow under load" often comes
  down to a blocking/CPU-bound call inside an `async def` handler starving
  the event loop — recognizing this pattern (and offloading via
  `run_in_executor`/a background worker) is a common senior-level
  production debugging skill; for example, a PDF render or Pandas
  transform inside the request path can make every other request wait.
- CPython's ongoing "free-threaded" (no-GIL) build effort (PEP 703) may
  change these trade-offs in future Python versions — treat GIL-related
  guidance as tied to the CPython version in use, not a permanent language
  property; for example, "threads won't help CPU work" is true for normal
  CPython today, but it is not a forever-language rule.
- Choosing between threads/asyncio/multiprocessing is a system design
  decision that should be driven by profiling the actual workload
  (I/O-bound vs CPU-bound), not by default habit — see
  [Performance and Profiling](14-performance-and-profiling.md) for how to
  measure before deciding; for example, an API gateway fits `asyncio`,
  while image thumbnail generation usually fits a process pool.
