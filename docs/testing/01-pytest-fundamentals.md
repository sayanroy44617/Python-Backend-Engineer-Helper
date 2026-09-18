# Pytest Fundamentals

## What

**pytest** is Python's de facto standard testing framework: plain
`assert` statements, automatic test discovery, and a powerful **fixture**
system for setup/teardown and dependency injection into tests.

## Why

Compared to the standard library's `unittest`, pytest removes boilerplate
(`self.assertEqual(a, b)` becomes `assert a == b`), gives much more
readable failure output (it introspects the failing expression), and its
fixture system composes far better than `setUp`/`tearDown` for sharing
setup logic across many tests.

## How

### A minimal test

```python
# test_math.py
def add(a: int, b: int) -> int:
    return a + b

def test_add():
    assert add(2, 3) == 5
```

```bash
pytest test_math.py -v
```

pytest discovers any file matching `test_*.py`/`*_test.py`, and within it
any function named `test_*` — no test class or special base class
required (though `unittest.TestCase`-style classes still work if you
have them).

### Fixtures for setup/teardown

```python
import pytest

@pytest.fixture
def db_session():
    session = create_session()
    yield session          # provided to the test
    session.rollback()     # teardown, runs after the test
    session.close()

def test_create_user(db_session):
    user = create_user(db_session, name="Sayan")
    assert user.id is not None
```

Everything before `yield` is setup, everything after is teardown —
pytest guarantees the teardown runs even if the test itself fails,
similar to a
[context manager's `__exit__`](../python/10-context-managers-and-descriptors.md).

### Fixture scope

```python
@pytest.fixture(scope="function")  # default: new instance per test
def db_session(): ...

@pytest.fixture(scope="module")  # shared across all tests in one file
def db_engine(): ...

@pytest.fixture(scope="session")  # shared across the entire test run
def app_config(): ...
```

Scope controls how often a fixture is recreated — `function` scope
(the default) gives full isolation at the cost of setup overhead;
`session`/`module` scope amortizes expensive setup (e.g. spinning up a
test database container) across many tests, at the cost of tests
potentially affecting each other if not careful.

### Fixture composition

```python
@pytest.fixture
def db_engine():
    return create_engine("sqlite:///:memory:")

@pytest.fixture
def db_session(db_engine):       # fixtures can depend on other fixtures
    with Session(db_engine) as session:
        yield session

def test_something(db_session):  # pytest resolves the whole chain
    ...
```

Fixtures requesting other fixtures is how pytest builds up complex test
setup from small, independently reusable pieces — analogous to how
FastAPI's own
[dependency injection](../fastapi/03-dependency-injection.md) composes
dependencies (pytest's fixture system was one of the inspirations for
FastAPI's `Depends`).

### Parametrization

```python
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

Runs the same test body once per parameter set, reported as separate
test results (`test_add[2-3-5]`, etc.) — avoids copy-pasting near-
identical test functions for each input/output pair.

### `conftest.py`: sharing fixtures across files

```python
# conftest.py (no import needed -- pytest finds it automatically)
@pytest.fixture
def db_session():
    ...
```

Fixtures defined in `conftest.py` are automatically available to every
test file in that directory (and subdirectories) without an explicit
import — this is how a project shares common fixtures (database session,
test client, sample data) across its whole test suite.

### Markers

```python
@pytest.mark.slow
def test_expensive_computation():
    ...
```

```bash
pytest -m "not slow"  # skip tests marked "slow"
```

Custom markers let you tag and selectively run/skip subsets of the test
suite (e.g. skipping slow integration tests during rapid local
iteration, running them only in CI).

## When to use

- Plain `assert` + fixtures for nearly all Python testing — pytest is the
  standard choice in the ecosystem.
- `conftest.py` for fixtures shared across multiple test files in a
  package/directory.
- Parametrization whenever you find yourself copy-pasting a test with
  only the input values changed.

## When NOT to use

- Don't reach for `session`-scoped fixtures by default — the isolation
  cost of getting shared state wrong (one test's leftover state
  affecting another) usually outweighs the setup-time savings, unless the
  setup is genuinely expensive (e.g. a real database container).
- Don't use fixtures for values that don't need setup/teardown at all —
  a plain constant or a helper function is simpler than a fixture that
  just returns a static value.

## Common mistakes

- Using function-scoped fixtures that hold genuinely expensive
  resources (spinning up a fresh database schema per test) instead of
  scoping appropriately and resetting state between tests more cheaply.
- Forgetting that fixture teardown code (after `yield`) doesn't run if
  the fixture itself raises during setup (before `yield`) — resource
  leaks can occur here if not handled carefully.
- Writing tests that depend on execution order (test B assumes test A's
  side effects) — pytest doesn't guarantee a specific order, and this
  breaks in parallel test runners.
- Not using `conftest.py`, instead importing fixtures directly between
  test files — this works but loses pytest's automatic fixture discovery
  and creates awkward cross-file coupling.

## Interview questions

1. What's the practical difference between pytest's fixtures and
   `unittest`'s `setUp`/`tearDown`?

   **Answer:** `setUp`/`tearDown` run for every test in a class,
   unconditionally. Fixtures are opt-in per test (only tests that request
   them get them), composable (a fixture can depend on other fixtures),
   and reusable across files via `conftest.py` — much more flexible for
   sharing setup selectively.

2. What does fixture `scope` control, and what's the trade-off between
   `function` and `session` scope?

   **Answer:** `scope` controls how often the fixture is recreated —
   `function` (default) creates it fresh per test (safest, fully
   isolated, slower); `session` creates it once for the whole test run
   (fast, but tests can leak state into each other if the fixture is
   mutable).

3. How does `conftest.py` make fixtures available without an import?

   **Answer:** pytest automatically discovers `conftest.py` files in the
   test directory tree and registers any fixtures defined there for every
   test file in that directory (and subdirectories) — no explicit import
   needed, it's just pytest's plugin/collection mechanism.

4. What does `@pytest.mark.parametrize` do, and why is it preferable to
   writing separate near-duplicate test functions?

   **Answer:** It runs the same test body once per set of input values you
   provide, showing each as a separate test result. It's preferable
   because you write the assertion logic once instead of copy-pasting
   near-identical test functions for each input case.

   ```python
   @pytest.mark.parametrize("value,expected", [(1, 2), (2, 4), (3, 6)])
   def test_double(value: int, expected: int) -> None:
       assert double(value) == expected
   ```

5. What happens to a fixture's teardown code if the test itself raises
   an exception?

   **Answer:** Teardown (code after `yield` in a fixture) still runs —
   pytest guarantees cleanup happens whether the test passes, fails, or
   errors, similar to a `finally` block.

## Senior-level considerations

- Fixture design is itself an architectural decision for a test suite —
  a well-designed fixture hierarchy (small, composable, correctly scoped)
  makes writing new tests fast; a poorly designed one makes every new
  test require understanding a tangle of shared state. For example, a
  `db_session` fixture that composes a smaller `engine` fixture is easier
  to reuse and reason about than one giant fixture doing everything.
- Test suite execution time matters at scale — knowing when to trade
  isolation for shared, more expensive fixtures (and how to do so safely)
  is a real engineering trade-off, not just a testing detail. For example,
  sharing one `session`-scoped Postgres container across a test file but
  rolling back a transaction per test keeps speed without sacrificing
  isolation.
- A senior engineer treats test code with the same standards as
  production code: DRY setup via fixtures, clear naming, and readable
  failure output, since test code is read (and debugged) at least as
  often as it's written. For example, a fixture named `authenticated_client`
  communicates intent far better than a generic `client2`.
