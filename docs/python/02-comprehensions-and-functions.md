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
2. What's the difference between a list comprehension and a generator
   expression? When does the difference matter?
3. Explain the late-binding closure bug with loop variables and how to fix
   it.
4. What happens if you mutate a variable from an enclosing scope without
   `nonlocal`?
5. Why would you make an argument keyword-only?
6. How do `*args` and `**kwargs` work under the hood (tuple/dict packing)?

## Senior-level considerations

- Keyword-only arguments and explicit signatures matter for API stability:
  adding a new keyword-only parameter is backward compatible; adding a new
  positional parameter can break callers.
- Excessive `**kwargs` forwarding in layered codebases (e.g. repository →
  service → API) makes it hard to trace what parameters actually flow
  through — prefer explicit, typed parameters at public boundaries and
  reserve `**kwargs` for truly generic infrastructure code (middlewares,
  decorators).
- Generator expressions compose well with async generators
  (`async for`) — the same "don't materialize everything in memory"
  principle applies when streaming large query results or paginated API
  responses.
