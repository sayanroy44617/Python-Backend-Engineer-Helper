# Exceptions

**Level:** Foundation

## What

Python's mechanism for signaling and handling errors: `try`/`except`/`else`/
`finally`, the built-in exception hierarchy, raising exceptions, and defining
custom exception types.

## Why

Backend services must fail predictably and communicate *why* they failed —
to callers (HTTP error responses), to logs, and to on-call engineers.
Exception design is part of your API contract: a well-designed exception
hierarchy lets a FastAPI exception handler map domain errors to the correct
HTTP status codes without scattering `if` checks everywhere.

## How

### Basic control flow

```python
def parse_amount(raw: str) -> float:
    try:
        value = float(raw)
    except ValueError as exc:
        raise ValueError(f"invalid amount: {raw!r}") from exc
    else:
        # runs only if no exception was raised
        print("parsed successfully")
        return value
    finally:
        # always runs, even if an exception propagates
        print("parse_amount finished")
```

- `else`: runs only when the `try` block succeeds — useful to separate the
  "risky" code from code that should only run on success.
- `finally`: always runs — cleanup (closing files/connections) that must
  happen regardless of outcome.
- `raise ... from exc`: preserves the original traceback as
  `__cause__`, giving a clear chain instead of hiding the root cause.

### Exception hierarchy (partial)

```
BaseException
 ├── SystemExit
 ├── KeyboardInterrupt
 └── Exception
      ├── ValueError
      ├── TypeError
      ├── KeyError
      ├── LookupError
      │    └── IndexError
      ├── OSError
      │    └── FileNotFoundError
      └── ... 
```

Always catch `Exception`, not `BaseException` — the latter also catches
`SystemExit`/`KeyboardInterrupt`, which should normally propagate.

### Common built-in exceptions and when they're raised

| Exception | Raised when | Example |
|---|---|---|
| `ValueError` | Right type, invalid value | `int("abc")` |
| `TypeError` | Wrong type entirely | `"a" + 1` |
| `KeyError` | Missing dict key via `[]` | `{}["missing"]` |
| `IndexError` | Sequence index out of range | `[1, 2][5]` |
| `AttributeError` | Attribute/method doesn't exist | `"abc".push()` |
| `FileNotFoundError` | Path doesn't exist (subclass of `OSError`) | `open("nope.txt")` |
| `ZeroDivisionError` | Division/modulo by zero | `1 / 0` |
| `StopIteration` | Iterator exhausted (usually handled internally by `for`) | `next(iter([]))` |
| `RuntimeError` | Generic error not covered by a more specific type | raised explicitly by libraries |
| `NotImplementedError` | Abstract method not overridden | called on an ABC's placeholder method |

```python
try:
    config["missing_key"]
except KeyError as exc:
    print(f"missing config key: {exc}")   # str(exc) is just the key repr

try:
    [1, 2, 3][10]
except IndexError as exc:
    print(f"index error: {exc}")
```

### Catching multiple exception types

```python
try:
    process(payload)
except (KeyError, ValueError) as exc:
    # handle either the same way
    raise HTTPException(status_code=400, detail=str(exc)) from exc
except TypeError:
    # handle differently
    raise HTTPException(status_code=500, detail="internal error")
```

Order matters: Python checks `except` clauses top-to-bottom and uses the
first match — a broad `except Exception` placed *before* a specific
`except ValueError` would shadow it and never let the specific clause run.

### `raise` variants

```python
raise ValueError("bad input")                 # new exception
raise ValueError("bad input") from exc          # explicit chaining -- sets __cause__
raise                                           # re-raise the exception currently being handled, unchanged
```

```python
def validate(value: int) -> None:
    try:
        assert value > 0
    except AssertionError:
        raise   # preserves the original traceback exactly as-is
```

`raise ... from None` explicitly suppresses the chained context (useful
when the original low-level exception would be confusing/irrelevant to the
caller):

```python
try:
    int(raw)
except ValueError:
    raise ValueError(f"invalid amount: {raw!r}") from None
```

### Catching specific vs broad exceptions

```python
# Good: specific, informative
try:
    user = repository.get_by_id(user_id)
except UserNotFoundError:
    raise HTTPException(status_code=404, detail="User not found")

# Bad: swallows everything, including bugs
try:
    user = repository.get_by_id(user_id)
except Exception:
    return None
```

### Custom exceptions

```python
class DomainError(Exception):
    """Base class for all domain-level errors in this service."""


class UserNotFoundError(DomainError):
    def __init__(self, user_id: int) -> None:
        self.user_id = user_id
        super().__init__(f"user {user_id} not found")


class InsufficientBalanceError(DomainError):
    pass
```

A custom hierarchy lets calling code catch broadly (`except DomainError`) or
narrowly (`except UserNotFoundError`) as needed, and lets a single FastAPI
exception handler translate `DomainError` subclasses into HTTP responses.

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(UserNotFoundError)
async def handle_user_not_found(request: Request, exc: UserNotFoundError):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

### Exception groups (Python 3.11+)

Python 3.11 introduced `ExceptionGroup`/`except*` for handling multiple
unrelated exceptions raised concurrently (e.g. from `asyncio.TaskGroup`).
This is version-dependent — check your target Python version before relying
on it.

### Context managers for guaranteed cleanup

`with` blocks are the preferred alternative to `try`/`finally` for cleanup —
`__exit__` runs even if an exception propagates:

```python
class DatabaseTransaction:
    def __enter__(self):
        self.conn = get_connection()
        return self.conn

    def __exit__(self, exc_type, exc_value, traceback) -> bool:
        if exc_type is None:
            self.conn.commit()
        else:
            self.conn.rollback()   # runs automatically on any exception
        self.conn.close()
        return False   # False = don't suppress the exception; re-raises it

with DatabaseTransaction() as conn:
    conn.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    conn.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
```

Returning `True` from `__exit__` **suppresses** the exception — useful for
narrow cases (e.g. `contextlib.suppress`) but easy to misuse if it silently
swallows errors the caller expected to see.

## When to use

- Raise exceptions for **exceptional, unexpected** conditions — not for
  routine control flow (e.g. don't use exceptions to check "does this key
  exist" when `dict.get()` works).
- Define custom exception types at domain boundaries (services,
  repositories) so callers can react to *meaning*, not just a generic
  `ValueError`.
- Use `finally` for guaranteed cleanup when a context manager isn't
  available.

## When NOT to use

- Don't use bare `except:` (catches everything, including
  `KeyboardInterrupt`) — always specify `Exception` at minimum.
- Don't use exceptions for expected, frequent flow control (e.g. validating
  every form field with try/except is slower and less readable than
  explicit checks or a validation library like Pydantic).
- Don't swallow exceptions silently (`except Exception: pass`) — always log
  or re-raise; silent failures are extremely hard to debug in production.

## Common mistakes

- Catching `Exception` too broadly and hiding real bugs.
- Losing the original traceback by re-raising without `from exc`.
- Using mutable default state inside custom exceptions incorrectly (rare,
  but same mutable-default pitfall as functions applies to `__init__`).
- Forgetting that `finally` runs even when a function `return`s from inside
  `try` — and that a `return` inside `finally` silently swallows any
  exception in flight.

```python
def risky() -> str:
    try:
        raise ValueError("boom")
    finally:
        return "never see the ValueError"  # anti-pattern: swallows the error
```

## Interview questions

1. What's the difference between `Exception` and `BaseException`? Why does
   it matter which one you catch?
2. What does `raise ... from exc` do, and why is it useful?
3. When does `else` in a try/except block execute, versus putting that code
   directly after the `try`?
4. Why is `except Exception: pass` considered a serious anti-pattern in
   production code?
5. How would you design an exception hierarchy for a service with multiple
   failure modes (not found, validation, permission denied)?
6. What happens if a `finally` block contains a `return` statement while an
   exception is propagating?
7. What's the difference between `raise` (bare) and `raise exc` inside an
   `except` block?
8. What does returning `True` from `__exit__` do, and why is it risky?
9. Give an example of a built-in exception that is a subclass of another
   built-in exception (e.g. `IndexError` vs `LookupError`) — why does that
   hierarchy matter when writing a broad `except` clause?
10. What does `raise ... from None` do, and when would you use it over
    `raise ... from exc`?

## Senior-level considerations

- A well-structured exception hierarchy is part of your service's public
  contract — changing it (renaming, removing a subclass) can be a breaking
  change for consumers who catch specific types.
- In distributed systems, distinguish **retryable** errors (timeouts,
  connection resets) from **non-retryable** ones (validation errors,
  permission denied) — often via a marker base class or attribute — so
  retry/backoff logic can decide correctly.
- Logging exceptions with full context (structured fields, not just the
  message) is critical for observability — see
  [Observability: Logging](../observability/01-logging-and-structured-logging.md).
- Avoid leaking internal exception details (stack traces, DB errors) in API
  responses — map to a safe, generic message while logging the full detail
  internally.
