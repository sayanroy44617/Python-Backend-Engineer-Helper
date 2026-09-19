# OpenAPI and Testing

## What

FastAPI automatically generates an **OpenAPI** schema (and interactive
docs at `/docs`/`/redoc`) from your route signatures and Pydantic models.
Its `TestClient` (built on `httpx`) lets you write tests that call your API
in-process, without running a real server.

## Why

Auto-generated, always-in-sync API documentation removes an entire class
of "docs drifted from the code" problems. In-process testing makes API
tests fast and reliable, avoiding the flakiness of spinning up a real HTTP
server for every test run.

## How

### OpenAPI generation

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="My API", version="1.0.0")

class UserOut(BaseModel):
    id: int
    name: str

@app.get("/users/{user_id}", response_model=UserOut, tags=["users"])
def get_user(user_id: int):
    ...
```

Visiting `/openapi.json` returns the generated schema; `/docs` (Swagger UI)
and `/redoc` render it interactively. The schema is built entirely from
type hints, Pydantic models, and route metadata (`tags`, `summary`,
`description`) — there's no separate spec to keep in sync manually.

### Customizing documentation

```python
@app.get(
    "/users/{user_id}",
    response_model=UserOut,
    summary="Get a user by ID",
    description="Returns a single user, or 404 if not found.",
    responses={404: {"description": "User not found"}},
)
def get_user(user_id: int):
    ...
```

Documenting non-2xx responses (`responses={...}`) makes the generated docs
show what error responses clients should expect — valuable when your
[exception handlers](04-middleware-and-exception-handling.md) produce
specific status codes for specific conditions.

### Testing with `TestClient`

```python
from fastapi.testclient import TestClient

client = TestClient(app)

def test_get_user():
    response = client.get("/users/1")
    assert response.status_code == 200
    assert response.json() == {"id": 1, "name": "Sayan"}

def test_create_user_validation_error():
    response = client.post("/users", json={"name": "", "email": "bad"})
    assert response.status_code == 422
```

`TestClient` sends requests directly into the ASGI app in-process — no
real network socket or running server needed, making tests fast and
deterministic.

### Overriding dependencies in tests

```python
def override_get_db():
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db

def test_create_user():
    response = client.post("/users", json={"name": "Sayan", "email": "s@example.com"})
    assert response.status_code == 200
```

Reuses the override mechanism from
[Dependency Injection](03-dependency-injection.md) — tests run against a
real (test) database or a fake, without touching production
infrastructure, while exercising the actual route/dependency wiring.

### Testing async endpoints

```python
import pytest
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_async_endpoint():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        response = await ac.get("/fast-io")
    assert response.status_code == 200
```

`TestClient` works fine for `async def` endpoints too (it runs its own
event loop internally), but an async test client (`httpx.AsyncClient` +
`ASGITransport`) is useful when the test itself needs to `await` other
async setup/teardown code.

### Testing authentication

```python
def test_requires_auth():
    response = client.get("/me")
    assert response.status_code == 401

def test_authenticated_request():
    app.dependency_overrides[get_current_user] = lambda: User(id=1, name="Sayan")
    response = client.get("/me")
    assert response.status_code == 200
```

Overriding `get_current_user` (rather than generating and sending a real
token) keeps auth-dependent tests fast and focused on the route's own
behavior, not the auth mechanism itself (which should have its own
dedicated tests).

## When to use

- Rely on FastAPI's generated OpenAPI schema as your API documentation —
  add `summary`/`description`/`responses` metadata rather than maintaining
  a separate spec.
- Use `TestClient`/`AsyncClient` for all route-level tests — they exercise
  the real routing, validation, and dependency-injection wiring.
- Use `dependency_overrides` to substitute test doubles for DB sessions,
  external clients, and authenticated users.

## When NOT to use

- Don't hand-write or maintain a separate OpenAPI/Swagger spec file
  alongside FastAPI's auto-generated one — they will drift; customize via
  code (metadata, models) instead.
- Don't spin up a real running server (a separate process bound to a port)
  for route-level unit/integration tests — `TestClient` in-process testing
  is faster and just as effective for most cases; reserve real servers for
  true end-to-end tests.
- Don't test authentication mechanics (token signing/expiry) through every
  route test — override `get_current_user` for route tests, and test the
  auth dependency itself separately.

## Common mistakes

- Letting hand-maintained API documentation drift from the actual code —
  a symptom of not fully relying on FastAPI's auto-generated schema.
- Forgetting to reset `app.dependency_overrides` between tests, leaking
  overrides from one test into another (typically handled via a pytest
  fixture that clears overrides after each test).
- Writing tests that hit a real external service or database instead of an
  overridden test double, making the suite slow and flaky.
- Not testing error paths (validation errors, 404s, permission denials) —
  only testing the happy path leaves the exception handling layer
  unverified.

## Interview questions

1. How does FastAPI generate its OpenAPI schema without a separate spec
   file?

   **Answer:** It builds the schema from your route signatures, type hints, Pydantic models, and route metadata like `response_model` and `summary`. In practice, the code is the source of truth and the docs are generated from that.

   ```python
   from pydantic import BaseModel

   class UserOut(BaseModel):
       id: int
   ```

2. Why is `TestClient` generally faster and more reliable than spinning up
   a real server for tests?

   **Answer:** It calls the ASGI app in-process, so there is no real socket, no process management, and less test flakiness. You still exercise routing, validation, dependencies, and exception handling, which is what most route tests actually need.

3. How would you test a route that requires authentication without
   generating a real token in every test?

   **Answer:** Override `get_current_user` so the test injects a known user directly. That keeps the test focused on the route behavior instead of dragging token creation into every case.

   ```python
   app.dependency_overrides[get_current_user] = lambda: User(id=1, name="Sayan")
   response = client.get("/me")
   assert response.status_code == 200
   ```

4. What's the risk of not resetting `app.dependency_overrides` between
   tests?

   **Answer:** Overrides leak across test cases, so one test can accidentally change the behavior of another. That gives you false positives, confusing failures, and order-dependent tests.

5. When would you reach for `httpx.AsyncClient` over the standard
   `TestClient`?

   **Answer:** Use it when the test itself needs to `await` async setup, async DB calls, or other async helpers. If the whole test is synchronous, `TestClient` is usually simpler.

   ```python
   from httpx import ASGITransport, AsyncClient

   transport = ASGITransport(app=app)
   ```

## Senior-level considerations

- Treating the OpenAPI schema as a generated artifact (not hand-authored)
  is what keeps documentation trustworthy at scale — consider validating
  it in CI (e.g. diffing against a committed snapshot) to catch
  unintentional breaking changes to the public API contract; for example, CI
  can fail if a response field disappears from `/openapi.json` unexpectedly.
- A test suite built around `dependency_overrides` scales well because it
  tests the real route/validation/dependency wiring while still isolating
  external systems — this is usually a better default than heavy mocking
  of internal functions; for example, override `get_db` with a test session
  instead of mocking `UserService.create_user` line by line.
- Distinguish test levels deliberately: fast in-process route tests
  (`TestClient` + overrides) for the majority of coverage, a smaller number
  of true end-to-end tests (real server, real DB) for confidence that the
  whole stack wires together correctly — see the Testing section for this
  test pyramid in depth; for example, keep hundreds of route tests in-process
  and only a handful of full dockerized smoke tests in CI.
