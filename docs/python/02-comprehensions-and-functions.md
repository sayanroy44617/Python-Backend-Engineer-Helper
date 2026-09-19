# Comprehensions and Functions

**Level:** Foundation

## What

Comprehensions are concise syntax for building lists, dicts, sets, and
generators from an iterable. Functions are the core unit of reusable
behavior in Python, with flexible argument passing via positional,
keyword, `*args`, and `**kwargs`. Scope resolution follows the **LEGB**
rule: Local, Enclosing, Global, Built-in.

## Why

Comprehensions are idiomatic Python — more readable and often faster than an
equivalent `for` loop with `.append()`. Understanding argument passing and
scope is essential for writing correct APIs (e.g. FastAPI route handlers,
decorators, dependency-injected functions) and for avoiding subtle bugs
around variable capture and default arguments.

## How

### Comprehensions

```python
# List comprehension
squares = [n * n for n in range(10)]

# Dict comprehension
by_id = {user.id: user for user in users}

# Set comprehension
unique_domains = {email.split("@")[1] for email in emails}

# Generator expression -- lazy, doesn't build the full list in memory
total = sum(n * n for n in range(10))

# With filtering and nesting
evens = [n for n in range(20) if n % 2 == 0]
pairs = [(x, y) for x in range(3) for y in range(3) if x != y]
```

Prefer a generator expression over a list comprehension when you only need
to iterate once (e.g. passed to `sum()`, `any()`, `all()`) — it avoids
materializing the full sequence in memory.

### Common built-in functions used alongside comprehensions

| Function | Effect | Example | Result |
|---|---|---|---|
| `map(fn, iterable)` | Apply `fn` to every item, lazily | `list(map(str, [1, 2, 3]))` | `['1', '2', '3']` |
| `filter(fn, iterable)` | Keep items where `fn` is truthy, lazily | `list(filter(lambda n: n > 1, [1, 2, 3]))` | `[2, 3]` |
| `zip(*iterables)` | Pair up items positionally, stops at the shortest | `list(zip([1, 2], ["a", "b"]))` | `[(1, 'a'), (2, 'b')]` |
| `enumerate(iterable, start=0)` | Pair each item with its index | `list(enumerate(["a", "b"]))` | `[(0, 'a'), (1, 'b')]` |
| `sorted(iterable, key=, reverse=)` | New sorted list (doesn't mutate input) | `sorted([3, 1, 2])` | `[1, 2, 3]` |
| `any(iterable)` | `True` if at least one item is truthy | `any(n > 5 for n in [1, 6])` | `True` |
| `all(iterable)` | `True` if every item is truthy | `all(n > 0 for n in [1, 2])` | `True` |

```python
# map/filter vs. an equivalent comprehension -- comprehensions are usually
# considered more Pythonic/readable for anything beyond a single function call
doubled = list(map(lambda n: n * 2, [1, 2, 3]))
doubled = [n * 2 for n in [1, 2, 3]]           # equivalent, more idiomatic

# zip() is the standard way to iterate two sequences in lockstep
names = ["ana", "bo"]
ages = [34, 22]
for name, age in zip(names, ages):
    print(f"{name} is {age}")

# enumerate() instead of manual index tracking
for i, name in enumerate(names, start=1):
    print(f"{i}. {name}")
```

```python
from functools import reduce, partial

# reduce() folds an iterable down to a single value -- less common than
# map/filter in idiomatic Python, but shows up in interviews
total = reduce(lambda acc, n: acc + n, [1, 2, 3, 4], 0)  # 10

# partial() pre-fills some arguments, returning a new callable
add = lambda a, b: a + b
add_five = partial(add, 5)
add_five(10)  # 15
```

### Lambda functions

```python
square = lambda n: n * n   # equivalent to: def square(n): return n * n
sorted(users, key=lambda u: u["age"])   # most common real use: a sort key
```

A `lambda` is restricted to a single expression (no statements, no
annotations) — anything more complex should be a regular `def` function for
readability and debuggability (lambdas show up as `<lambda>` in
tracebacks).

### Functions and arguments

```python
def create_user(
    name: str,
    email: str,
    *,
    is_admin: bool = False,
) -> dict[str, str | bool]:
    return {"name": name, "email": email, "is_admin": is_admin}
```

- Positional and keyword arguments.
- `*` forces subsequent arguments to be keyword-only (`is_admin` above) —
  common in production APIs to prevent ambiguous positional calls.
- `/` (less common) forces preceding arguments to be positional-only.

```python
def move(x, y, /, *, label: str = "point") -> str:
    return f"{label}: ({x}, {y})"

move(1, 2)                  # OK -- x, y positional
move(1, 2, label="origin")  # OK -- label keyword-only
move(x=1, y=2)               # TypeError -- x, y are positional-only
```

Positional-only parameters (`/`) are common in built-in functions
(`len(obj, /)`) and are useful when you want the freedom to rename a
parameter later without breaking callers who might otherwise pass it by
keyword.

### Default argument evaluation timing

```python
import datetime

def log_event(message: str, timestamp=datetime.datetime.now()) -> None:
    print(timestamp, message)

# Bug: timestamp is computed ONCE, at function definition time, not per call
log_event("first")   # both calls print the SAME timestamp
log_event("second")

# Fix: use None as a sentinel and compute the real default inside the body
def log_event(message: str, timestamp: datetime.datetime | None = None) -> None:
    timestamp = timestamp or datetime.datetime.now()
    print(timestamp, message)
```

This is the same root cause as the mutable-default-argument bug covered in
[Data Types and Collections](01-data-types-and-collections.md) — default
argument expressions are evaluated exactly once, whether they're a mutable
literal (`[]`) or a function call (`datetime.now()`).

### Unpacking arguments at the call site

```python
def create_user(name: str, email: str, is_admin: bool = False) -> dict:
    return {"name": name, "email": email, "is_admin": is_admin}

args = ("ana", "ana@example.com")
kwargs = {"is_admin": True}

create_user(*args, **kwargs)   # unpacks the tuple and dict into the call
# equivalent to: create_user("ana", "ana@example.com", is_admin=True)
```

This is the mirror image of `*args`/`**kwargs` in a function *definition*
(which **collect** arguments into a tuple/dict) — at the *call site*, `*`/`**`
**spread** an existing iterable/mapping out into individual arguments.

### `*args` and `**kwargs`

```python
def log_call(*args: object, **kwargs: object) -> None:
    print("positional:", args)   # tuple
    print("keyword:", kwargs)    # dict

log_call(1, 2, user="sayan")
# positional: (1, 2)
# keyword: {'user': 'sayan'}
```

Common real use: wrapping/forwarding calls in decorators or adapter
functions without knowing the wrapped function's exact signature.

```python
def wrapper(*args, **kwargs):
    return original_function(*args, **kwargs)
```

### Scope and LEGB

Name resolution order: **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in.

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)  # "local"

    inner()
    print(x)  # "enclosing"

print(x)  # "global"
```

To modify an enclosing/global variable from a nested scope, you need
`nonlocal`/`global` — otherwise Python treats the assignment as creating a
new local variable.

```python
def counter():
    count = 0

    def increment():
        nonlocal count
        count += 1
        return count

    return increment
```

## When to use

- Comprehensions: transforming/filtering an iterable into a new collection
  in one clear expression.
- Generator expressions: single-pass iteration, streaming, or large/unknown
  size data where memory matters.
- `*args`/`**kwargs`: generic decorators, adapters, or APIs that forward
  arguments to another callable.
- Keyword-only arguments (`*`): public functions/APIs where positional
  ambiguity would be a footgun (e.g. multiple boolean flags).

## When NOT to use

- Don't nest comprehensions more than 2 levels deep — readability drops
  fast; use a regular loop or extract a helper function instead.
- Don't overuse `*args`/`**kwargs` on functions with a fixed, known
  signature — it hides the real API and defeats type checking/IDE
  autocomplete.
- Avoid comprehensions with side effects (e.g. calling a function purely for
  its effect) — a plain `for` loop communicates intent better.

## Common mistakes

- Using a list comprehension when a generator would do, wasting memory on
  large datasets.
- Losing loop variables to the outer scope in Python 2 style thinking — in
  Python 3, comprehension variables are scoped to the comprehension itself
  (this is *not* a bug, but worth knowing it differs from a plain `for`
  loop).
- Late binding closures: capturing a loop variable inside a comprehension or
  lambda without binding its current value.

```python
# Bug: all lambdas capture the same variable `i`, evaluated at call time
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]  # [2, 2, 2] -- not [0, 1, 2]

# Fix: bind the value as a default argument
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]  # [0, 1, 2]
```

- Forgetting `nonlocal` when mutating an enclosing variable, causing an
  `UnboundLocalError`.

## Interview questions

1. What is the LEGB rule? Walk through an example with nested functions.

   **Answer:** Python resolves names in this order: Local, Enclosing, Global,
   Built-in. In nested functions, an inner function sees its own variables
   first, then the outer function's variables, then module-level names.

   ```python
   x: str = "global"
   def outer() -> str:
       x: str = "enclosing"
       return x
   ```

2. What's the difference between a list comprehension and a generator
   expression? When does the difference matter?

   **Answer:** A list comprehension builds the whole list immediately; a
   generator expression produces values lazily as you iterate. The difference
   matters when the input is large or you only need one pass.

   ```python
   nums: list[int] = [n * n for n in range(3)]
   total: int = sum(n * n for n in range(3))
   ```

3. Explain the late-binding closure bug with loop variables and how to fix
   it.

   **Answer:** Closures capture the variable, not the value at each iteration,
   so every function can end up reading the loop variable's final value. Bind
   the current value with a default argument or extract a helper function.

   ```python
   funcs = [lambda i=i: i for i in range(3)]
   print([fn() for fn in funcs])  # [0, 1, 2]
   ```

4. What happens if you mutate a variable from an enclosing scope without
   `nonlocal`?

   **Answer:** Python treats that assignment as creating a new local variable,
   so reading it first usually triggers `UnboundLocalError`. `nonlocal` tells
   Python you mean the variable from the enclosing function.

   ```python
   from collections.abc import Callable

   def counter() -> Callable[[], int]:
       count: int = 0
       def inc() -> int:
           nonlocal count
           count += 1
           return count
       return inc
   ```

5. Why would you make an argument keyword-only?

   **Answer:** Keyword-only params make calls clearer and harder to misuse,
   especially for booleans or optional behavior flags. They're also nicer for
   API evolution because you can add them without breaking positional callers.

   ```python
   def fetch_user(user_id: int, *, include_deleted: bool = False) -> None:
       pass
   ```

6. How do `*args` and `**kwargs` work under the hood (tuple/dict packing)?

   **Answer:** Extra positional args are packed into a tuple, and extra keyword
   args are packed into a dict. That makes them useful for wrappers and
   decorators that need to forward arbitrary calls.

   ```python
   def log_call(*args: object, **kwargs: object) -> tuple[tuple[object, ...], dict[str, object]]:
       return args, kwargs
   ```

7. What's the difference between `*args` in a function *definition* versus
   `*some_list` at a *call site*?

   **Answer:** In a definition, `*args` collects positional arguments into a
   tuple. At a call site, `*some_list` unpacks an iterable into separate
   positional arguments.

   ```python
   def add(a: int, b: int) -> int:
       return a + b
   values: list[int] = [2, 3]
   print(add(*values))
   ```

8. Why is `def f(x, timestamp=datetime.now())` a bug? How do you fix it?

   **Answer:** `datetime.now()` runs once when the function is defined, not on
   every call, so later calls reuse the same timestamp. Use `None` and compute
   the real default inside the function body.

   ```python
   from datetime import datetime

   def stamp(ts: datetime | None = None) -> datetime:
       return ts or datetime.now()
   ```

9. What does `/` mean in a function signature, and where have you seen it
   used in the standard library?

   **Answer:** `/` marks parameters before it as positional-only, so callers
   cannot pass them by keyword. You see it in built-ins like `len(obj, /)` and
   `divmod(a, b, /)`.

   ```python
   def move(x: int, y: int, /) -> tuple[int, int]:
       return x, y
   ```

10. When would you reach for `functools.partial` instead of a `lambda`?

   **Answer:** Use `partial` when you just want to pre-fill some arguments of
   an existing callable and keep its behavior otherwise unchanged. It's clearer
   than a tiny wrapper lambda for simple argument binding.

   ```python
   from functools import partial

   def add(a: int, b: int) -> int:
       return a + b
   add_five = partial(add, 5)
   ```

## Senior-level considerations

- Keyword-only arguments and explicit signatures matter for API stability:
  adding a new keyword-only parameter is backward compatible; adding a new
  positional parameter can break callers. Example: changing
  `connect(url, timeout=5)` to `connect(url, *, timeout=5, retries=3)` is much
  safer than inserting another positional arg into existing call sites.
- Excessive `**kwargs` forwarding in layered codebases (e.g. repository →
  service → API) makes it hard to trace what parameters actually flow
  through — prefer explicit, typed parameters at public boundaries and
  reserve `**kwargs` for truly generic infrastructure code (middlewares,
  decorators). Example: `create_user(email: str, is_admin: bool)` is easier to
  reason about than passing `**kwargs` through three layers and hoping the
  final callee accepts the right keys.
- Generator expressions compose well with async generators
  (`async for`) — the same "don't materialize everything in memory"
  principle applies when streaming large query results or paginated API
  responses. Example: stream rows from the database and feed them straight
  into a CSV writer instead of first building a giant in-memory list.
