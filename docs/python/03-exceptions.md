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

'''CustomException in python should end with error not exception
   UserNotFoundError not UserNotFoundException(java style)'''
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
```python
#gives error like during handling exception another exception is raised
def fetch_user_data(user_id: int):
    try:
        # Imagine this fails with a KeyError or DatabaseError
        user = {}["missing_key"]
    except KeyError as exc:
        # ❌ BAD: Raising a new exception without chaining
        raise ValueError("Failed to fetch user profile")

fetch_user_data(1)


# this gives the proper traceback
def fetch_user_data(user_id: int):
    try:
        user = {}["missing_key"]
    except KeyError as exc:
        # ✅ GOOD: Explicitly chain the original exception
        raise ValueError("Failed to fetch user profile") from exc

fetch_user_data(1)
```
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

- What's the difference between `Exception` and `BaseException`? Why does
    it matter which one you catch?

    **Answer:** `Exception` is the normal application-error branch; `BaseException`
    also includes things like `SystemExit` and `KeyboardInterrupt`. Catching
    `BaseException` is usually too broad because it can stop shutdown and interrupt handling.

    ```python
    try:
       raise ValueError("bad input")
    except Exception:
       print("handled")
    ```

- What does `raise ... from exc` do, and why is it useful?

    **Answer:** It explicitly chains a higher-level exception to the original
    one so the traceback keeps both layers. That's useful when translating a
    low-level failure into a domain or API-level error.

    ```python
    try:
       int("x")
    except ValueError as exc:
       raise ValueError("invalid user id") from exc
    ```

- When does `else` in a try/except block execute, versus putting that code
    directly after the `try`?

    **Answer:** `else` runs only if the `try` block completed without raising.
    It keeps success-only logic separate from the code that might fail.

    ```python
    try:
       value: int = int("12")
    except ValueError:
       print("bad input")
    else:
       print(value)
    ```

- Why is `except Exception: pass` considered a serious anti-pattern in
    production code?

    **Answer:** It hides real failures and removes the signal you need for
    debugging, alerting, and retries. In production, swallowed exceptions often
    turn into silent data loss or stuck workflows.

- How would you design an exception hierarchy for a service with multiple
    failure modes (not found, validation, permission denied)?

    **Answer:** Create a shared domain base class, then add specific subclasses
    per failure mode so callers can catch broadly or narrowly. That keeps error
    mapping centralized instead of scattering `if "not found"` checks around.

    ```python
    class DomainError(Exception): ...
    class NotFoundError(DomainError): ...
    class PermissionDeniedError(DomainError): ...
    ```

- What happens if a `finally` block contains a `return` statement while an
    exception is propagating?

    **Answer:** The `return` from `finally` wins and the original exception gets
    swallowed. That's why `return` inside `finally` is almost always a bug.

    ```python
    def broken() -> str:
       try:
           raise ValueError("boom")
       finally:
           return "hidden"
    ```

- What's the difference between `raise` (bare) and `raise exc` inside an
    `except` block?

    **Answer:** Bare `raise` re-raises the current exception and preserves the
    original traceback. `raise exc` raises that exception object again and can
    make the traceback noisier or less direct.

- What does returning `True` from `__exit__` do, and why is it risky?

    **Answer:** It tells the context manager to suppress the exception instead
    of letting it propagate. That's risky because callers may think the block
    succeeded when it actually failed.

    ```python
    class Swallow:
       def __exit__(self, exc_type, exc, tb) -> bool:
           return True
    ```

- Give an example of a built-in exception that is a subclass of another
    built-in exception (e.g. `IndexError` vs `LookupError`) — why does that
    hierarchy matter when writing a broad `except` clause?

    **Answer:** `IndexError` is a subclass of `LookupError`, so catching
    `LookupError` handles list indexing errors and dict key lookup errors in one
    place. The hierarchy matters because a broader base class may catch more than you intended.

    ```python
    try:
       [1, 2][5]
    except LookupError:
       print("lookup failed")
    ```

- What does `raise ... from None` do, and when would you use it over
    `raise ... from exc`?

    **Answer:** It suppresses the original exception context and shows only the
    new error. Use it when the low-level cause would just confuse the caller and
    adds no value to the API surface.

## Senior-level considerations

- A well-structured exception hierarchy is part of your service's public
  contract — changing it (renaming, removing a subclass) can be a breaking
  change for consumers who catch specific types. Example: if SDK users catch
  `PaymentDeclinedError`, replacing it with a generic `PaymentError` is a real
  compatibility break.
- In distributed systems, distinguish **retryable** errors (timeouts,
  connection resets) from **non-retryable** ones (validation errors,
  permission denied) — often via a marker base class or attribute — so
  retry/backoff logic can decide correctly. Example: `class RetryableError(Exception): ...`
  lets worker code retry `UpstreamTimeoutError` but fail fast on `ValidationError`.
- Logging exceptions with full context (structured fields, not just the
  message) is critical for observability — see
  [Observability: Logging](../observability/01-logging-and-structured-logging.md).
  Example: log `order_id`, `user_id`, and `payment_provider` with the exception,
  not just `"payment failed"`.
- Avoid leaking internal exception details (stack traces, DB errors) in API
  responses — map to a safe, generic message while logging the full detail
  internally. Example: return `"detail": "internal server error"` to the client,
  while logs keep the original `psycopg` or SQLAlchemy traceback.
