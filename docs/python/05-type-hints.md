# Type Hints

**Level:** Foundation

## What

Optional static type annotations added via the `typing` module (and native
syntax since Python 3.9/3.10). Type hints don't change runtime behavior by
default — they're checked by external tools (mypy, pyright) and used by
frameworks like FastAPI/Pydantic for validation and serialization.

## Why

Type hints turn implicit contracts into explicit, checkable ones. In backend
code they catch bugs before runtime, power editor autocomplete, and are
*load-bearing* in FastAPI — request/response models and dependency injection
are driven by type annotations, not just documentation.

## How

### Basic annotations

```python
def get_user(user_id: int) -> dict[str, str]:
    return {"id": str(user_id), "name": "Sayan"}

age: int = 30
name: str = "Sayan"
```

### Built-in generics (Python 3.9+)

```python
names: list[str] = ["a", "b"]
scores: dict[str, int] = {"a": 1}
coordinates: tuple[float, float] = (1.0, 2.0)
unique_ids: set[int] = {1, 2, 3}
```

Before Python 3.9, you needed `typing.List`, `typing.Dict`, etc. — modern
code should prefer the built-in generics.

### Optional and Union

```python
from typing import Optional

def find_user(user_id: int) -> Optional[dict]:
    ...

# Python 3.10+ union syntax (equivalent, preferred in modern code)
def find_user(user_id: int) -> dict | None:
    ...

def parse(value: int | str) -> str:
    return str(value)
```

`Optional[X]` is shorthand for `X | None` — it does **not** mean "this
argument has a default value"; it only describes that `None` is a valid
value.

### Generics with `TypeVar` / generic classes

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Repository(Generic[T]):
    def __init__(self, items: list[T]) -> None:
        self._items = items

    def get(self, index: int) -> T:
        return self._items[index]

user_repo: Repository[User] = Repository([User(id=1)])
```

### Protocols (structural typing)

```python
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...

def cleanup(resource: SupportsClose) -> None:
    resource.close()
```

Any object with a matching `close()` method satisfies `SupportsClose` —
no explicit inheritance required (duck typing, checked statically).

### `TypedDict` and `NewType`

```python
from typing import TypedDict, NewType

class UserPayload(TypedDict):
    id: int
    name: str

UserId = NewType("UserId", int)

def get_user(user_id: UserId) -> UserPayload:
    ...
```

`TypedDict` describes the shape of a plain dict (useful for JSON payloads
without creating a full class). `NewType` creates a distinct type for type
checkers without runtime cost — useful to avoid mixing up e.g. `UserId` and
`OrderId`, both plain `int`s.

### More typing constructs

| Construct | Purpose | Example |
|---|---|---|
| `Literal["a", "b"]` | Value must be one of a fixed set of literals | `def set_mode(mode: Literal["fast", "slow"]) -> None: ...` |
| `Final` | Marks a name as not meant to be reassigned/overridden | `MAX_RETRIES: Final = 3` |
| `Callable[[int, str], bool]` | A function taking `(int, str)` and returning `bool` | `handler: Callable[[Request], Response]` |
| `Sequence[T]` / `Mapping[K, V]` | Read-only, structural container types (broader than `list`/`dict`) | `def total(items: Sequence[int]) -> int: ...` |
| `Annotated[T, ...]` | Attach framework metadata to a type without changing it at runtime | `UserId = Annotated[int, "must be positive"]` |
| type alias (`type X = ...`, 3.12+) | Give a complex type a readable name | `type JSON = dict[str, "JSON"] \| list["JSON"] \| str \| int \| bool \| None` |
| `@overload` | Describe multiple valid call signatures for one function | see below |

```python
from typing import Literal, Final, Callable

MAX_RETRIES: Final = 3   # mypy flags any later reassignment as an error

def set_log_level(level: Literal["debug", "info", "warning", "error"]) -> None:
    ...

set_log_level("info")     # OK
set_log_level("verbose")  # mypy error: not one of the allowed literals

Handler = Callable[[int], str]

def register(handler: Handler) -> None:
    ...
```

```python
from typing import overload

@overload
def parse(value: str) -> int: ...
@overload
def parse(value: bytes) -> int: ...
def parse(value):
    return int(value)
```

`@overload` lets a type checker verify call sites against multiple precise
signatures (e.g. "returns `str` when called with `mode='text'`, returns
`bytes` when called with `mode='binary'`") even though there's only one
actual runtime implementation.

Annotated is what FastAPI itself increasingly relies on for combining a type
with framework metadata in one place:

```python
from typing import Annotated
from fastapi import Depends, Query

def list_users(
    limit: Annotated[int, Query(le=100)] = 10,
    db: Annotated[Session, Depends(get_db)] = ...,
):
    ...
```

### Static checking

```bash
mypy src/
```

Type hints are not enforced at runtime by the interpreter itself — calling
`get_user("not an int")` will not raise a `TypeError` from the annotation
alone. Runtime validation requires a library (Pydantic) or explicit checks.

## When to use

- Public function signatures, especially at module/service boundaries.
- FastAPI route handlers and Pydantic models — required for the framework's
  validation/serialization to work.
- Complex data shapes where a type hint documents intent better than a
  comment (`dict[str, list[int]]` vs a vague comment).

## When NOT to use

- Don't over-annotate trivial local variables where inference is obvious
  (`x = 5` doesn't need `x: int = 5`).
- Don't use `Any` as an escape hatch throughout a codebase — it disables
  type checking silently; prefer a precise type or `object` plus narrowing.
- Don't assume type hints provide runtime safety — untrusted input (API
  request bodies) must still be validated with Pydantic or explicit checks.

## Common mistakes

- Believing type hints are enforced at runtime (they are not, without a
  validation library).
- Overusing `Optional`/`| None` without actually handling the `None` case,
  leading to `AttributeError` at runtime.
- Using mutable types as default values in a way that also violates the
  mutable-default-argument rule (see Comprehensions and Functions) — a
  type hint doesn't fix that bug.
- Mixing old-style `typing.List`/`Dict` with new-style `list`/`dict` for no
  reason — pick one convention (built-in generics) per your Python version.

## Interview questions

1. Are type hints enforced at runtime? What actually validates types in a
   FastAPI app?
2. What's the difference between `Optional[int]` and `int | None`? Are they
   equivalent?
3. What is a `Protocol`, and how does it differ from an abstract base
   class?
4. When would you use `TypedDict` instead of a full class or a Pydantic
   model?
5. Why might a large codebase enforce mypy in CI even though Python is
   dynamically typed?
6. What problem does `NewType` solve that a plain type alias doesn't?
7. What's the difference between `Literal["fast", "slow"]` and just using
   `str`? What does it buy you?
8. What is `@overload` for, given that the actual implementation is a single
   function?
9. How does `Annotated` let FastAPI combine a type hint with validation
   metadata (e.g. `Query(le=100)`) in one place?

## Senior-level considerations

- Type hints are a form of documentation that stays in sync with code (unlike
  comments) as long as CI enforces a type checker — treat mypy/pyright
  failures as build failures in serious projects.
- FastAPI + Pydantic essentially use type hints as the single source of
  truth for request validation, serialization, and OpenAPI schema
  generation — understanding this deeply avoids "fighting the framework."
- Gradual typing lets you introduce type hints incrementally in a legacy
  codebase; senior engineers use `# type: ignore` sparingly and track it
  (e.g. via mypy's `--strict` mode adopted module by module) rather than
  disabling checks broadly.
- Behavior here is version-dependent: built-in generic syntax
  (`list[str]`), the `X | Y` union syntax, and newer `typing` features
  require specific minimum Python versions — always check target runtime
  version compatibility.
