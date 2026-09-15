# Interview Prep: Python

## How to approach Python interviews at the 2-3+ year level

Interviewers at this level rarely ask pure trivia ("what does `__init__`
do"). Expect questions that probe *why* Python behaves a certain way
(mutability, the GIL, generator laziness) and how that shapes real
design decisions (when `asyncio` actually helps, why a mutable default
argument is a footgun) — the "why" and "senior-level considerations"
framing used throughout the [Python](../python/index.md) section is
exactly what's being tested, not syntax recall.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | Why is `def f(x, cache={})` a bug waiting to happen? | Default argument objects are created once, at function definition time, and reused across every call — mutating `cache` mutates it for all future calls. | [Comprehensions and Functions](../python/02-comprehensions-and-functions.md) |
| 2 | What's the difference between a list comprehension and a generator expression? | A comprehension builds the entire result eagerly in memory; a generator expression yields values lazily, one at a time, and can only be iterated once. | [Iterators and Generators](../python/08-iterators-and-generators.md) |
| 3 | What does the GIL actually prevent, and what does it not prevent? | The GIL prevents two Python bytecode instructions from executing simultaneously in different threads in one process — it doesn't prevent concurrency from I/O-bound work, and it doesn't apply across separate processes. | [GIL and CPython](../python/12-gil-and-cpython.md) |
| 4 | When would you choose `multiprocessing` over `threading` in Python? | For CPU-bound work, since the GIL prevents threads from getting real parallelism on CPU-bound Python bytecode; `multiprocessing` sidesteps the GIL by using separate processes, each with its own interpreter. | [GIL and CPython](../python/12-gil-and-cpython.md) |
| 5 | Why doesn't `async def` alone make code run in parallel? | `asyncio` provides concurrency via a single-threaded event loop cooperatively switching between tasks at `await` points — it helps I/O-bound workloads overlap waiting time, not CPU-bound work needing true parallelism. | [Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md) |
| 6 | What's the difference between `@staticmethod`, `@classmethod`, and an instance method? | An instance method receives `self` and operates on a specific instance; a `classmethod` receives `cls` and typically builds/operates at the class level (e.g. alternate constructors); a `staticmethod` receives neither, just grouped under the class namespace. | [OOP Fundamentals](../python/06-oop-fundamentals.md) |
| 7 | Why use `@dataclass` instead of a plain class with `__init__`? | It generates `__init__`, `__repr__`, and `__eq__` automatically from declared fields, reducing boilerplate — but understand its defaults (e.g. it doesn't generate `__hash__` if the class is mutable) rather than treating it as a black box. | [Dataclasses, Properties, and Dunder Methods](../python/07-dataclasses-and-dunder-methods.md) |
| 8 | What problem do context managers (`with`) solve that a `try`/`finally` doesn't already solve? | They don't solve a fundamentally different problem — a context manager is exactly `try`/`finally` encapsulated and made reusable/composable, ensuring cleanup runs even on exception without repeating boilerplate at every call site. | [Context Managers and Descriptors](../python/10-context-managers-and-descriptors.md) |
| 9 | How does Python's reference counting garbage collector handle a reference cycle? | Pure reference counting alone can never free a cycle (two objects referencing each other, both otherwise unreachable) since each has count ≥ 1 forever — Python's supplementary cyclic garbage collector specifically detects and collects unreachable cycles. | [Memory Management](../python/11-memory-management.md) |
| 10 | Why prefer `contextlib.suppress` or specific exception types over a bare `except:`? | A bare `except:` (or overly broad `except Exception:`) can silently swallow bugs unrelated to the error you intended to handle (e.g. a `KeyboardInterrupt` or a typo causing a `NameError`) — catch the narrowest exception type that's actually expected. | [Exceptions](../python/03-exceptions.md) |
| 11 | What's the practical benefit of type hints if Python doesn't enforce them at runtime? | Static analysis (mypy), editor autocomplete/refactoring support, and documentation of intent — the enforcement happens via tooling (CI-run type checkers) rather than the interpreter, which is a deliberate trade-off, not an oversight. | [Type Hints](../python/05-type-hints.md) |
| 12 | How would you profile a slow Python function before optimizing it? | Measure first (`cProfile`, `py-spy`, or targeted timing) to find the actual bottleneck rather than guessing — optimizing a function that isn't the bottleneck wastes effort and adds complexity for no benefit. | [Performance and Profiling](../python/14-performance-and-profiling.md) |

## Live-coding / whiteboard tips

- State your **assumptions about input** out loud before coding (can the
  list be empty? can values repeat? is it sorted?) — interviewers
  usually want to see this reasoning, not just a working solution.
- For anything involving iteration, default to a generator/streaming
  approach and explicitly justify materializing a full list only when
  needed — this signals the same eager-vs-lazy reasoning covered in
  [Iterators and Generators](../python/08-iterators-and-generators.md).
- If asked to reason about concurrency, explicitly name whether the
  workload is I/O-bound or CPU-bound before choosing `asyncio`,
  `threading`, or `multiprocessing` — that classification is usually the
  crux of the question.

## Common red flags interviewers watch for

- Reaching for `threading` to speed up CPU-bound work without
  acknowledging the GIL.
- Using mutable default arguments without recognizing the bug.
- Treating `async`/`await` as "automatically faster" rather than
  "enables overlap of I/O wait time."
- Catching exceptions too broadly, masking real bugs.

## Related deep-dive material

- [Python section overview](../python/index.md) — all 14 topic pages,
  each with its own dedicated interview questions and senior-level
  considerations.
