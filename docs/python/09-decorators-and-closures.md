# Decorators and Closures

**Level:** Intermediate

## What

A **closure** is a function that captures variables from an enclosing
scope, remembering them even after the outer function has returned (basics
covered in
[Comprehensions and Functions](02-comprehensions-and-functions.md#scope-and-legb)).
A **decorator** is a function that takes a function (or class) and returns
a new one, typically adding behavior around the original — decorators are
one of the most common practical uses of closures.

## Why

Decorators are everywhere in backend Python: `@app.get(...)` in FastAPI,
`@pytest.fixture`, `@lru_cache`, `@retry`, `@login_required`. Understanding
how they're built on closures demystifies framework "magic" and lets you
write your own cross-cutting behavior (logging, timing, auth checks,
caching) without duplicating code across every function.

## How

### Closures recap

```python
def make_multiplier(factor: int):
    def multiply(value: int) -> int:
        return value * factor  # `factor` is captured from the enclosing scope
    return multiply

double = make_multiplier(2)
triple = make_multiplier(3)
double(5)  # 10
triple(5)  # 15
```

`double` and `triple` are closures — each remembers its own `factor` even
though `make_multiplier` has already returned. This is exactly the
mechanism decorators rely on.

### A basic decorator

```python
import functools
import time

def timed(func):
    @functools.wraps(func)  # preserves func's __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timed
def fetch_data(user_id: int) -> dict:
    return {"id": user_id}

fetch_data(1)
# fetch_data took 0.0001s
```

`@timed` is sugar for `fetch_data = timed(fetch_data)`. `wrapper` is a
closure over `func`. `functools.wraps` is important in production code —
without it, `fetch_data.__name__` would be `"wrapper"`, breaking
introspection, logging, and frameworks that rely on function metadata.

### Decorators with arguments

```python
def retry(times: int):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as exc:
                    last_exc = exc
            raise last_exc
        return wrapper
    return decorator

@retry(times=3)
def call_external_api():
    ...
```

This is a decorator **factory**: `retry(times=3)` returns the actual
decorator, which then wraps `call_external_api`. Three nested levels of
closures: `retry` → `decorator` → `wrapper`.

### Class-based decorators

```python
class CountCalls:
    def __init__(self, func) -> None:
        functools.update_wrapper(self, func)
        self.func = func
        self.calls = 0

    def __call__(self, *args, **kwargs):
        self.calls += 1
        return self.func(*args, **kwargs)

@CountCalls
def greet() -> str:
    return "hi"

greet()
greet.calls  # 1
```

Useful when the decorator needs to maintain more complex state than a
simple closure variable (here, `self.calls`).

### Decorating classes

```python
def add_repr(cls):
    def __repr__(self) -> str:
        return f"{cls.__name__}({self.__dict__})"
    cls.__repr__ = __repr__
    return cls

@add_repr
class Config:
    def __init__(self, debug: bool) -> None:
        self.debug = debug
```

Decorators aren't limited to functions — `@dataclass` itself is a class
decorator (see
[Dataclasses, Properties, and Dunder Methods](07-dataclasses-and-dunder-methods.md)).

### Stacking decorators

```python
@timed
@retry(times=3)
def call_external_api():
    ...
```

Applied bottom-up: `call_external_api = timed(retry(times=3)(call_external_api))`.
Order matters — here, `retry` retries the inner call, and `timed` measures
the total time across all retries.

## When to use

- Cross-cutting concerns that apply uniformly across many functions:
  logging, timing, retries, caching (`functools.lru_cache`), auth checks,
  input validation.
- Framework integration points (route registration, fixtures, CLI command
  registration) where a decorator gives a declarative API.

## When NOT to use

- Don't use a decorator to add behavior that's only relevant to one
  function — a plain helper call is simpler and easier to trace.
- Don't stack more than 2-3 decorators on a single function without strong
  justification — the effective behavior becomes hard to reason about.
- Avoid decorators that change a function's signature/return type in
  surprising ways without clear typing (use `ParamSpec`/`TypeVar` if you
  need decorators to preserve accurate type hints).

## Common mistakes

- Forgetting `functools.wraps`, breaking `__name__`/`__doc__` and anything
  relying on function introspection (some frameworks, debuggers, docs
  generators).
- Confusing a decorator factory (`@retry(times=3)`) with a plain decorator
  (`@timed`) — forgetting the extra call level and its parentheses.
- Mutating shared state in a decorator (e.g. a module-level counter)
  without considering thread/async safety.
- Applying decorators in the wrong order and getting confused about which
  wraps which — remember it's bottom-up.

## Interview questions

1. What is a closure? Give an example where it's used to configure
   behavior (like `make_multiplier`).
2. Walk through what `@timed` on a function desugars to.
3. Why is `functools.wraps` important in a decorator?
4. What's a decorator factory, and how does `@retry(times=3)` differ from
   `@retry`?
5. In what order do stacked decorators apply?
6. When would you write a class-based decorator instead of a function-based
   one?

## Senior-level considerations

- Decorators are a core extensibility mechanism in frameworks you'll build
  or maintain (e.g. a custom `@cache_response` for an internal API) —
  understanding closures lets you design decorators that compose cleanly
  with `functools.wraps` and correct typing (`ParamSpec` for accurate
  wrapped signatures).
- Be cautious with decorators that hold mutable state (counters, caches) in
  concurrent/async contexts — closures capture variables by reference, so
  shared state across concurrent requests needs the same thread/async
  safety considerations as any other shared mutable state.
- Overuse of decorators for "clever" implicit behavior can hurt
  readability/debuggability in large codebases — prefer explicit
  composition (plain function calls) when the indirection doesn't earn its
  complexity.
