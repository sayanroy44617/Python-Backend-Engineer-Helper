# Mocking and Monkeypatch

## What

**Mocking** replaces a real dependency (an external API, a database, the
system clock) with a fake, controllable substitute during a test.
`unittest.mock` is the standard library's mocking toolkit; pytest's
`monkeypatch` fixture is a safer, auto-cleaning way to patch objects,
attributes, and environment variables for the duration of a single test.

## Why

Tests need to be fast, deterministic, and independent of external
systems. Calling a real third-party API or a real database in every unit
test makes the suite slow, flaky (network failures, rate limits), and
dependent on external state — mocking isolates the code under test from
everything it depends on but isn't itself testing.

## How

### `unittest.mock.Mock` basics

```python
from unittest.mock import Mock

mock_client = Mock()
mock_client.get_user.return_value = {"id": 1, "name": "Sayan"}

result = mock_client.get_user(1)
assert result == {"id": 1, "name": "Sayan"}
mock_client.get_user.assert_called_once_with(1)
```

A `Mock` accepts any attribute access or method call, recording how it
was used — `assert_called_once_with(...)` then verifies the code under
test called it correctly.

### `patch` to replace a dependency during a test

```python
from unittest.mock import patch

def send_welcome_email(user_email: str) -> bool:
    return email_service.send(user_email, "Welcome!")

def test_send_welcome_email():
    with patch("myapp.email_service.send") as mock_send:
        mock_send.return_value = True
        assert send_welcome_email("a@example.com") is True
        mock_send.assert_called_once_with("a@example.com", "Welcome!")
```

`patch` replaces the target for the duration of the `with` block (or a
decorated test function) and restores the original automatically
afterward, even if the test fails — critical for not leaking a patched
object into other tests.

### Patching where it's *used*, not where it's *defined*

```python
# myapp/notifications.py
from myapp import email_service

def send_welcome_email(user_email: str) -> bool:
    return email_service.send(user_email, "Welcome!")
```

```python
# CORRECT: patch the name as looked up in notifications.py
with patch("myapp.notifications.email_service") as mock_service:
    ...

# WRONG (often silently doesn't work): patching the original module
with patch("myapp.email_service") as mock_service:
    ...
```

This is the single most common mocking mistake — `patch` needs the
import path *as it's referenced by the code under test*, not where the
real object originally lives, because `from x import y` binds a new name
in the importing module's namespace.

### `MagicMock` and side effects

```python
from unittest.mock import MagicMock

mock_db = MagicMock()
mock_db.query.side_effect = ConnectionError("db down")

def test_handles_db_error():
    with pytest.raises(ConnectionError):
        mock_db.query("SELECT 1")
```

`side_effect` lets a mock raise an exception or compute a return value
dynamically (a callable), useful for testing error-handling paths that
are hard to trigger with a real dependency.

### pytest's `monkeypatch` fixture

```python
def test_reads_env_var(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key-123")
    assert get_api_key() == "test-key-123"

def test_patches_attribute(monkeypatch):
    monkeypatch.setattr("myapp.notifications.email_service.send", lambda *a: True)
    assert send_welcome_email("a@example.com") is True
```

`monkeypatch` is pytest's own patching mechanism: functionally similar to
`unittest.mock.patch`, but it automatically reverts *any* change (env
vars, attributes, dict items, `sys.path`) at the end of the test without
needing a `with` block — commonly preferred within pytest test suites for
its simpler syntax.

### Mocking time

```python
def test_expired_token(monkeypatch):
    fixed_now = datetime(2024, 1, 1, tzinfo=timezone.utc)
    monkeypatch.setattr("myapp.auth.datetime", Mock(now=lambda tz=None: fixed_now))
    assert is_token_expired(issued_at=datetime(2023, 1, 1, tzinfo=timezone.utc))
```

Code that depends on "now" is otherwise nondeterministic to test — mocking
the clock is the standard way to test time-dependent logic (token expiry,
scheduled jobs) deterministically.

### Spying (verify without replacing behavior)

```python
from unittest.mock import patch

def test_logs_are_called():
    with patch("myapp.notifications.logger.info", wraps=logger.info) as spy:
        send_welcome_email("a@example.com")
        spy.assert_called_once()
```

`wraps=` lets the mock still call through to the real implementation
while recording the call — useful when you want to verify a call
happened without changing the actual behavior.

## When to use

- Mock any dependency that is slow, external, non-deterministic (time,
  randomness), or has side effects you don't want in a test (sending a
  real email, charging a real payment).
- `monkeypatch` for most in-pytest-suite patching needs — simpler syntax,
  automatic cleanup, and idiomatic within pytest.
- `unittest.mock.patch`/`Mock` when writing framework-agnostic test
  utilities or when finer control (call history assertions, `side_effect`
  chains) is needed.

## When NOT to use

- Don't mock the thing you're actually trying to test — if a test mocks
  so much that it no longer exercises real logic, it stops providing
  meaningful coverage.
- Don't over-mock in integration tests — the point of an integration test
  is to verify components work together with real (or realistic)
  dependencies; see
  [Test Strategy and Isolation](03-test-strategy-and-isolation.md).
- Don't mock simple, fast, pure functions — there's no benefit to mocking
  something with no side effects and no external dependency.

## Common mistakes

- Patching the wrong import path (patching where an object is *defined*
  instead of where it's *imported and used*), causing the mock to
  silently have no effect.
- Over-specifying mock assertions (asserting exact call arguments for
  every internal detail), producing brittle tests that break on harmless
  refactors.
- Forgetting `side_effect` vs `return_value`: `side_effect` set to an
  exception *instance* raises it; set to a *callable* it's invoked;
  `return_value` is just returned as-is.
- Leaving a `unittest.mock.patch` unpatched by not using a `with` block
  or decorator correctly, leaking the mock into subsequent tests.

## Interview questions

1. What's the difference between `Mock` and `MagicMock`?
2. Why must you patch a dependency where it's *used*, not where it's
   *defined*? Walk through why the "wrong" patch target silently fails.
3. What's the practical difference between `unittest.mock.patch` and
   pytest's `monkeypatch`?
4. When would you use `side_effect` instead of `return_value`?
5. How would you deterministically test code that depends on the current
   time?

## Senior-level considerations

- Excessive mocking is a code smell as much as a testing technique — if
  a unit needs a dozen mocks to test, that's often a signal the unit has
  too many responsibilities or too tightly coupled dependencies.
- Mocking hides real integration bugs (a mocked API client can't tell you
  the real API changed its response shape) — a mature test suite balances
  mocked unit tests with a smaller number of real integration tests that
  catch those gaps (see
  [Test Strategy and Isolation](03-test-strategy-and-isolation.md)).
- Knowing the exact import-resolution mechanics behind "patch where it's
  used" (Python's module namespace binding) is a strong signal of real
  hands-on testing experience versus surface-level familiarity with
  mocking syntax.
