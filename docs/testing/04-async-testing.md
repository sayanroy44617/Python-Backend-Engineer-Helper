# Async Testing

## What

Testing `async def` code — coroutines, async generators, and async
context managers — requires an async-aware test runner, since a plain
`def test_...()` function has no event loop to run a coroutine on.
`pytest-asyncio` is the standard plugin providing this.

## Why

`async def foo(): ...` returns a coroutine object, not a result — calling
it without awaiting it inside a running event loop does nothing useful
(see [Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md)).
Testing async code correctly means giving each test its own event loop to
actually run the coroutine to completion.

## How

### A basic async test

```python
import pytest

async def fetch_user(user_id: int) -> dict:
    await asyncio.sleep(0.01)  # simulate I/O
    return {"id": user_id, "name": "Sayan"}

@pytest.mark.asyncio
async def test_fetch_user():
    result = await fetch_user(1)
    assert result == {"id": 1, "name": "Sayan"}
```

`@pytest.mark.asyncio` tells the plugin to run this `async def` test
function inside an event loop it manages — without it, pytest would treat
the function as a normal (never-awaited) test and it would pass trivially
without actually running any of its assertions.

### Enabling it project-wide (`asyncio_mode`)

```ini
# pytest.ini or pyproject.toml [tool.pytest.ini_options]
asyncio_mode = "auto"
```

With `asyncio_mode = "auto"`, every `async def test_...` is automatically
treated as an async test without needing `@pytest.mark.asyncio` on each
one — commonly preferred in projects where most tests are async (e.g. a
FastAPI codebase using `AsyncSession` throughout).

### Async fixtures

```python
@pytest.fixture
async def async_db_session():
    engine = create_async_engine("postgresql+asyncpg://localhost/test_db")
    async with AsyncSession(engine) as session:
        yield session
    await engine.dispose()

@pytest.mark.asyncio
async def test_create_user(async_db_session):
    user = await create_user_async(async_db_session, name="Sayan")
    assert user.id is not None
```

Async fixtures work the same way as sync fixtures (setup before `yield`,
teardown after), just declared with `async def` — pytest-asyncio handles
awaiting them correctly when a test requests one.

### Testing FastAPI async endpoints with `httpx.AsyncClient`

```python
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_async_endpoint():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        response = await ac.get("/users/1")
    assert response.status_code == 200
```

This mirrors the example in
[OpenAPI and Testing](../fastapi/07-openapi-and-testing.md) — an async
test client is needed when the test itself must `await` other async
setup, such as an async fixture providing a real `AsyncSession`.

### Testing concurrency behavior itself

```python
@pytest.mark.asyncio
async def test_runs_concurrently():
    start = time.monotonic()
    await asyncio.gather(slow_io_task(), slow_io_task(), slow_io_task())
    elapsed = time.monotonic() - start
    assert elapsed < 0.5  # would be ~3x longer if run sequentially
```

Occasionally you want to verify concurrency actually happened (not just
that individual coroutines return the right value) — timing-based
assertions like this are inherently a bit fragile, so keep the margin
generous and reserve them for cases where concurrency is the behavior
under test.

### Testing timeouts and cancellation

```python
@pytest.mark.asyncio
async def test_times_out():
    with pytest.raises(asyncio.TimeoutError):
        async with asyncio.timeout(0.01):
            await asyncio.sleep(1)
```

Verifying that slow operations are correctly bounded by a timeout is a
common and valuable async-specific test — it exercises cancellation
behavior that's easy to get subtly wrong (e.g. a `finally` block that
doesn't clean up correctly on cancellation).

### Mocking async dependencies

```python
from unittest.mock import AsyncMock

async def test_calls_async_dependency():
    mock_client = AsyncMock()
    mock_client.fetch.return_value = {"status": "ok"}

    result = await process(mock_client)

    mock_client.fetch.assert_awaited_once()
```

`AsyncMock` (not plain `Mock`) is required for mocking something that
will be `await`ed — it makes the mock itself awaitable and provides
`assert_awaited_*` assertions; using a plain `Mock` for an async
dependency causes a `TypeError` when the code under test tries to
`await` it.

## When to use

- `pytest-asyncio` (with `asyncio_mode = "auto"` for convenience) for any
  project with `async def` code paths to test.
- `AsyncMock` specifically whenever mocking something the code under test
  will `await`.
- `httpx.AsyncClient` + `ASGITransport` when a test needs to `await`
  other async setup alongside making requests to the app.

## When NOT to use

- Don't use `pytest-asyncio` markers on synchronous test functions — only
  `async def test_...` functions need it.
- Don't rely on timing-based assertions (`elapsed < X`) as your primary
  correctness check for concurrent code — use them sparingly, and prefer
  asserting on actual outcomes (call counts, results) wherever possible.
- Don't mock an async dependency with a plain `Mock` — it silently breaks
  when awaited (or requires extra manual wiring) where `AsyncMock` just
  works.

## Common mistakes

- Writing `async def test_...()` without `@pytest.mark.asyncio` (and
  without `asyncio_mode = "auto"`) — the test appears to pass because the
  coroutine is never actually awaited, silently skipping every assertion.
- Using `Mock` instead of `AsyncMock` for an async dependency, causing a
  `TypeError: object Mock can't be used in 'await' expression`.
- Forgetting that async fixtures need `async def` (not just returning a
  coroutine from a sync fixture function).
- Writing brittle, tightly-bounded timing assertions for concurrency
  tests that intermittently fail on a loaded CI runner.

## Interview questions

1. Why does a plain `async def test_...` function silently "pass" without
   `@pytest.mark.asyncio` or `asyncio_mode = "auto"` configured?

   **Answer:** Without pytest-asyncio driving it, pytest just calls the
   `async def` function, gets back a coroutine object, and — since
   nothing ever awaits it — the coroutine body never actually runs. The
   test "passes" because no assertion inside it ever executed, not
   because anything was verified.

2. Why do you need `AsyncMock` instead of `Mock` for mocking an async
   dependency? What error do you get if you use the wrong one?

   **Answer:** Calling a regular `Mock` returns a `Mock` object
   immediately, which isn't awaitable — awaiting it raises `TypeError:
   object Mock can't be used in 'await' expression`. `AsyncMock` returns
   a coroutine when called, so `await` works correctly.

3. How would you test that a piece of code times out correctly using
   `asyncio.timeout`?

   **Answer:** Run the code against something that intentionally takes
   longer than the timeout (e.g. `asyncio.sleep`) and assert that
   `TimeoutError` is raised.

   ```python
   import asyncio
   import pytest

   async def test_times_out() -> None:
       with pytest.raises(TimeoutError):
           async with asyncio.timeout(0.01):
               await asyncio.sleep(1)
   ```

4. What's a risk with asserting on wall-clock elapsed time to verify
   concurrent behavior in a test?

   **Answer:** Wall-clock timing is sensitive to CI machine load/jitter —
   a test asserting "this ran in under 100ms" can flake on a busy CI
   runner even though the concurrency logic is correct.

5. How does testing an async FastAPI endpoint differ from testing a
   synchronous one?

   **Answer:** You typically use an async HTTP test client (e.g.
   `httpx.AsyncClient`) and `await` its calls inside an `async def` test,
   instead of a plain synchronous `TestClient` call — the test itself
   needs to run inside the event loop pytest-asyncio manages.

## Senior-level considerations

- Reliable async testing requires understanding the event loop lifecycle
  pytest-asyncio manages per test — debugging "hangs forever" or "works
  locally, flakes in CI" async test failures often comes back to event
  loop or fixture scope misconfiguration. For example, mixing a
  `session`-scoped async fixture with `function`-scoped event loops can
  cause a "Future attached to a different loop" error.
- Testing cancellation and timeout behavior correctly is disproportionately
  valuable in async codebases — these are exactly the code paths (cleanup
  in `finally` under cancellation, propagating timeouts through nested
  `await`s) that are easy to get subtly wrong and hard to catch via manual
  testing. For example, a test that cancels a task mid-`await` and
  asserts its `finally` block still closed a file/connection catches bugs
  that would otherwise only show up as a leaked resource in production.
- Async test suites can be slower to run than an equivalent sync suite if
  fixtures aren't scoped well (e.g. recreating an async engine per test
  instead of per session) — balancing isolation against the overhead of
  async resource setup is a real design decision in larger test suites.
  For example, sharing one `session`-scoped async engine across a test
  file (with per-test transaction rollback) avoids reconnecting to the DB
  for every single test.
