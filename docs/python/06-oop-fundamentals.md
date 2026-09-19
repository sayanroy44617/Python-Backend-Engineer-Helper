# OOP Fundamentals

**Level:** Intermediate

## What

Object-oriented building blocks beyond basic classes: inheritance,
composition, abstract base classes (ABCs), and how they relate to
`Protocol`s (structural typing, already covered in
[Type Hints](05-type-hints.md)).

## Why

Backend services model domain concepts (users, orders, payments) as
classes. Choosing inheritance vs composition, and nominal (ABC) vs
structural (`Protocol`) typing, directly affects how easily a codebase can
be extended, tested, and mocked — a recurring theme in senior-level design
discussions and code reviews.

## How

### Inheritance

```python
class PaymentMethod:
    def charge(self, amount: float) -> None:
        raise NotImplementedError

class CreditCard(PaymentMethod):
    def charge(self, amount: float) -> None:
        print(f"Charging ${amount} to credit card")

class PayPal(PaymentMethod):
    def charge(self, amount: float) -> None:
        print(f"Charging ${amount} via PayPal")
```

Inheritance models an **is-a** relationship. Python supports multiple
inheritance; method resolution follows the **MRO** (Method Resolution
Order, C3 linearization), inspectable via `ClassName.__mro__`.

```python
class A:
    def greet(self) -> str:
        return "A"

class B(A):
    def greet(self) -> str:
        return "B -> " + super().greet()

class C(A):
    def greet(self) -> str:
        return "C -> " + super().greet()

class D(B, C):
    pass

D().greet()      # "B -> C -> A"
D.__mro__         # (D, B, C, A, object)
```

### Composition

```python
class EmailNotifier:
    def send(self, message: str) -> None:
        print(f"Emailing: {message}")

class OrderService:
    def __init__(self, notifier: EmailNotifier) -> None:
        self._notifier = notifier  # has-a relationship

    def place_order(self, order_id: int) -> None:
        self._notifier.send(f"Order {order_id} placed")
```

Composition models a **has-a** relationship: `OrderService` *uses* a
notifier rather than *being* one. Favor composition over inheritance when
you only need to reuse behavior, not share a type hierarchy — it avoids
deep, fragile class trees and makes dependencies swappable (e.g. for
testing, inject a fake notifier).

### Abstract base classes

```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def charge(self, amount: float) -> None: ...

    def receipt(self, amount: float) -> str:
        return f"Charged ${amount}"  # concrete shared behavior

class CreditCard(PaymentMethod):
    def charge(self, amount: float) -> None:
        print(f"Charging ${amount}")

PaymentMethod()   # TypeError: can't instantiate abstract class
CreditCard()      # OK -- implements charge()
```

`ABC` enforces (at instantiation time) that subclasses implement required
methods, and lets you share concrete behavior (`receipt`) alongside the
abstract contract.

### ABC vs Protocol

| | `ABC` | `Protocol` |
|---|---|---|
| Typing | Nominal (must inherit explicitly) | Structural (duck typing, no inheritance needed) |
| Enforcement | Runtime, at instantiation | Static, via type checker only (unless `@runtime_checkable`) |
| Shared implementation | Yes (concrete methods) | No (interface only) |
| Use case | A real base class with shared logic | A lightweight interface for functions/tests |

See [Type Hints](05-type-hints.md#protocols-structural-typing) for the
`Protocol` example (`SupportsClose`). Use `ABC` when you want a real base
class with enforced structure and shared code; use `Protocol` when you just
need to describe "any object with this method" without forcing inheritance.

## When to use

- Inheritance: a true **is-a** relationship with shared behavior across a
  small, stable hierarchy (e.g. `PaymentMethod` subclasses).
- Composition: reusing behavior across unrelated classes, or when the
  relationship might need to change at runtime (inject a different
  strategy).
- `ABC`: you control the base class and want to guarantee subclasses
  implement specific methods, with some shared logic.
- `Protocol`: you're describing a contract for external/third-party types
  you don't control, or want structural typing for tests/mocks.

## When NOT to use

- Don't use deep inheritance chains (more than 2-3 levels) — MRO becomes
  hard to reason about and small changes ripple unpredictably.
- Don't use multiple inheritance for anything beyond mixins with narrow,
  well-documented responsibility (e.g. `LoggingMixin`).
- Don't reach for an `ABC` when a `Protocol` or even a plain function
  would do — not every interface needs a class hierarchy.

## Common mistakes

- Forgetting `super().__init__()` in a subclass `__init__`, silently
  skipping base class setup.
- Using inheritance purely for code reuse when composition would be more
  flexible and testable ("inheritance for reuse" anti-pattern).
- Assuming `@abstractmethod` prevents *calling* an unimplemented method at
  runtime if the class was constructed through unusual means — it only
  blocks direct instantiation of a class with unimplemented abstract
  methods.

## Interview questions

- What's the difference between inheritance and composition? Give a
    backend example.

    **Answer:** Inheritance says one class is a specialized version of another; composition says one class uses another to get work done. In backend code, `RefundService` using a `PaymentGateway` dependency is usually composition, and it's easier to swap in tests.

    ```python
    class PaymentGateway:
       def charge(self, amount: int) -> None:
           print(amount)
    ```

- Explain Python's MRO with a diamond inheritance example.

    **Answer:** MRO is the order Python follows when resolving methods across parent classes. In a diamond shape, Python uses C3 linearization so each base appears once in a predictable order.

    ```python
    class A: ...
    class B(A): ...
    class C(A): ...
    class D(B, C): ...

    print(D.__mro__)
    ```

- When would you choose an `ABC` over a `Protocol`, and vice versa?

    **Answer:** Use an `ABC` when you want a real base class with shared logic and runtime enforcement. Use a `Protocol` when you only care about the method shape and don't want to force inheritance.

 4. Why is "favor composition over inheritance" a common guideline?

    **Answer:** Because composition keeps dependencies explicit and swappable, while inheritance tends to couple behavior into a hierarchy that's harder to change later. It's usually the safer default in service code where requirements move a lot.

- What does `@abstractmethod` actually enforce, and when?

    **Answer:** It prevents you from instantiating a class until all abstract methods are implemented. It does not validate business semantics; it only enforces that the required method names exist.

    ```python
    from abc import ABC, abstractmethod

    class Base(ABC):
       @abstractmethod
       def run(self) -> None: ...
    ```

## Senior-level considerations

- Composition-based designs (dependency injection of collaborators) are
  what make services testable without heavy mocking frameworks — this is
  the same principle behind FastAPI's `Depends()`; for example, injecting a
  fake email sender into `UserService` is simpler than subclassing the whole
  service for tests.
- Overusing inheritance in domain models is a common source of rigid,
  hard-to-refactor codebases; many senior engineers default to composition
  and reserve inheritance for genuinely stable, narrow hierarchies; for
  example, `CsvReportExporter` and `PdfReportExporter` are often better as
  separate strategies than siblings in a deep reporting tree.
- `Protocol` + composition together enable structural typing without
  coupling your code to a specific class hierarchy — useful when writing
  library code that must accept many caller-defined types; for example, a
  cache helper can accept any object exposing `get()` and `set()` methods,
  not just one Redis wrapper class.
