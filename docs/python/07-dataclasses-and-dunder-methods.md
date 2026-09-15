# Dataclasses, Properties, and Dunder Methods

**Level:** Intermediate

## What

`@dataclass` auto-generates boilerplate (`__init__`, `__repr__`, `__eq__`,
etc.) for classes that primarily hold data. `@property` turns method calls
into attribute-like access with validation/computed logic. Dunder
("double underscore") methods (`__eq__`, `__repr__`, `__hash__`, `__len__`,
...) let custom classes integrate with built-in operators and functions.

## Why

Most backend domain objects (DTOs, value objects, simple entities) are
mostly data with a bit of behavior — dataclasses remove repetitive
boilerplate. Properties let you validate/compute values while keeping a
clean attribute-access API. Dunder methods are how your classes plug into
`==`, `repr()`, `len()`, sorting, hashing, and more — essential for correct
behavior in sets, dicts, and tests (`assert obj == expected`).

## How

### Dataclasses

```python
from dataclasses import dataclass, field

@dataclass
class User:
    id: int
    name: str
    tags: list[str] = field(default_factory=list)  # avoids mutable default bug
    is_active: bool = True

u1 = User(id=1, name="Sayan")
u2 = User(id=1, name="Sayan")

u1 == u2          # True -- __eq__ generated, compares fields
repr(u1)          # "User(id=1, name='Sayan', tags=[], is_active=True)"
```

`field(default_factory=...)` is the dataclass-native fix for the mutable
default argument problem (see
[Data Types and Collections](01-data-types-and-collections.md)) — a plain
`tags: list[str] = []` raises `ValueError` at class definition time.

```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float
```

`frozen=True` makes instances immutable (raises on attribute
reassignment) and makes the generated `__hash__` usable, so frozen
dataclasses can be used as dict keys or set members — unlike regular
mutable dataclasses, which are unhashable by default once `eq=True` is set.

### Properties

```python
class Account:
    def __init__(self, balance: float) -> None:
        self._balance = balance

    @property
    def balance(self) -> float:
        return self._balance

    @balance.setter
    def balance(self, value: float) -> None:
        if value < 0:
            raise ValueError("balance cannot be negative")
        self._balance = value

account = Account(100)
account.balance = 50     # calls the setter, validates
account.balance = -10    # raises ValueError
```

Properties let you start with a plain attribute and later add
validation/computed logic without breaking the public API (callers still
write `account.balance`, not `account.get_balance()`).

### Dunder methods

```python
class Money:
    def __init__(self, cents: int) -> None:
        self.cents = cents

    def __repr__(self) -> str:
        return f"Money(cents={self.cents})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return self.cents == other.cents

    def __lt__(self, other: "Money") -> bool:
        return self.cents < other.cents

    def __add__(self, other: "Money") -> "Money":
        return Money(self.cents + other.cents)

    def __hash__(self) -> int:
        return hash(self.cents)
```

| Dunder | Triggered by |
|---|---|
| `__repr__` | `repr(obj)`, debugger/console display |
| `__str__` | `str(obj)`, `print(obj)` (falls back to `__repr__`) |
| `__eq__` | `==` |
| `__lt__`, `__le__`, ... | `<`, `<=`, sorting (`sorted()`) |
| `__hash__` | `hash(obj)`, use as dict key/set member |
| `__len__` | `len(obj)` |
| `__add__` | `+` |
| `__contains__` | `in` |

Defining `__eq__` without `__hash__` makes the class **unhashable** — Python
sets `__hash__` to `None` automatically in that case, because mutable
equality and identity-based hashing would otherwise be inconsistent. Only
define `__hash__` for objects you intend to treat as immutable value
objects.

## When to use

- `@dataclass` for DTOs, value objects, and simple entities where
  `__init__`/`__eq__`/`__repr__` would otherwise be repetitive boilerplate.
- `@dataclass(frozen=True)` for value objects you want to use as dict
  keys/set members or that should never mutate after creation (e.g. a
  `Money` or `Coordinates` type).
- `@property` when you need validation, computed values, or backward
  compatibility while migrating a plain attribute to logic-backed access.
- Custom `__eq__`/`__lt__`/`__hash__` when domain equality differs from
  default identity-based comparison (e.g. two `Money(100)` instances should
  be equal).

## When NOT to use

- Don't use `@dataclass` for classes with significant behavior/methods and
  little data — a plain class is clearer.
- Don't add `@property` for simple, no-validation attributes "just in
  case" — start with a plain attribute (per YAGNI); add a property later if
  validation becomes necessary (the public API stays the same).
- Don't implement `__eq__` without also handling `__hash__` deliberately —
  decide explicitly whether instances should be hashable.

## Common mistakes

- Using a mutable default (`tags: list[str] = []`) in a dataclass instead
  of `field(default_factory=list)` — dataclasses actually raise an error
  for this specific case, but it still trips people up.
- Forgetting `frozen=True` needed for hashability, then being surprised a
  dataclass instance can't be put in a `set`.
- Implementing `__eq__` but returning `False` instead of `NotImplemented`
  for unrelated types, which breaks reflected comparison (`other == self`).
- Confusing `__str__` and `__repr__` — `__repr__` should be unambiguous
  (ideally `eval`-able or at least developer-facing); `__str__` is for
  user-facing display.

## Interview questions

1. What does `@dataclass` generate for you by default, and what does it
   skip unless you ask for it (e.g. `__hash__`, ordering)?
2. Why does defining `__eq__` set `__hash__` to `None` unless you define it
   explicitly?
3. When would you use `@property` instead of a plain public attribute?
4. What's the difference between `__str__` and `__repr__`?
5. Why is `field(default_factory=list)` needed instead of `= []` in a
   dataclass?
6. What does returning `NotImplemented` from `__eq__` accomplish?

## Senior-level considerations

- Frozen dataclasses are a lightweight way to introduce immutable value
  objects, which simplifies reasoning in concurrent code (asyncio tasks,
  multiprocessing) since shared immutable state can't be mutated
  unexpectedly.
- Overusing properties for trivial validation scattered across many small
  classes can hide business rules that belong in a dedicated validation
  layer (e.g. Pydantic models at the API boundary) — be deliberate about
  where validation logic lives.
- Dunder methods are part of your type's contract with the rest of the
  language (sorting, hashing, arithmetic) — changing their semantics later
  is a breaking change for any code relying on them (e.g. code that puts
  your objects in a `set` or sorts them).
