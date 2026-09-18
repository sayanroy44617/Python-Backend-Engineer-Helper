# Dependency Injection

## What

FastAPI's `Depends()` mechanism lets a route (or another dependency)
declare what it needs — a DB session, the current user, a config object —
and have FastAPI construct and inject it automatically, resolving nested
dependencies and managing their lifecycle.

## Why

Without dependency injection, every route re-implements setup logic (open
a DB session, decode a token, check permissions) inline, making it
duplicated and hard to test. `Depends()` centralizes that logic once,
composes it across routes, and — critically for testing — lets you swap
real dependencies for fakes without touching route code.

## How

### A basic dependency

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{user_id}")
def get_user(user_id: int, db=Depends(get_db)):
    return db.query(User).get(user_id)
```

A dependency that uses `yield` behaves like a context manager: code before
`yield` runs before the route handler, code after runs after the response
is sent (or after an exception) — the standard pattern for resources that
need guaranteed cleanup (see
[Context Managers and Descriptors](../python/10-context-managers-and-descriptors.md)
for the underlying mechanism).

### Dependencies with parameters

```python
def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    payload = decode_token(token)
    return get_user_by_id(payload["sub"])

@app.get("/me")
def read_current_user(user: User = Depends(get_current_user)):
    return user
```

Dependencies can themselves declare dependencies (`get_current_user`
depends on `oauth2_scheme`) — FastAPI resolves the whole chain, calling
each once per request even if used by multiple downstream dependencies.

### Class-based dependencies

```python
class Pagination:
    def __init__(self, skip: int = 0, limit: int = 20) -> None:
        self.skip = skip
        self.limit = limit

@app.get("/items")
def list_items(pagination: Pagination = Depends()):
    return get_items(pagination.skip, pagination.limit)
```

A callable class works as a dependency — `Depends()` without an explicit
argument uses the parameter's type annotation, calling `Pagination(...)`
with query parameters matched to `__init__`'s signature.

### Sharing dependencies across routes

```python
router = APIRouter(dependencies=[Depends(verify_api_key)])

@router.get("/reports")
def get_reports():
    ...
```

Dependencies declared at the router (or app) level apply to every route
under it without repeating `Depends(...)` on each handler — useful for
cross-cutting checks like API key verification that apply to a whole group
of endpoints.

### Overriding dependencies (testing)

```python
app.dependency_overrides[get_db] = get_test_db

def get_test_db():
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.close()
```

`app.dependency_overrides` swaps a dependency for a test double without
changing any route code — the core mechanism that makes FastAPI apps
testable in isolation from real infrastructure (see
[OpenAPI and Testing](07-openapi-and-testing.md)).

### Caching within a request

```python
def get_settings():
    return Settings()

@app.get("/config")
def show_config(
    settings_a: Settings = Depends(get_settings),
    settings_b: Settings = Depends(get_settings),
):
    assert settings_a is settings_b  # same instance -- cached per request
```

By default, FastAPI calls each dependency **once per request** and reuses
the result for every other dependency/route that needs it in that same
request (unless `use_cache=False` is passed to `Depends`).

## When to use

- Any resource with a request-scoped lifecycle: DB sessions, request-scoped
  caches, per-request tracing context.
- Authentication/authorization checks that many routes share — declare
  once, reuse via `Depends()` at the router or route level.
- Anything you want to substitute in tests (real DB vs test DB, real auth
  vs a fake authenticated user).

## When NOT to use

- Don't use `Depends()` for values that never vary per request and have no
  setup/teardown — a plain module-level constant or function call is
  simpler.
- Don't put heavy business logic inside a dependency just to "inject" it —
  dependencies are best for cross-cutting request-scoped concerns, not a
  substitute for a proper service layer.
- Don't rely on `use_cache=False` as a default habit — the per-request
  caching behavior is usually what you want (e.g. one DB session shared
  across nested dependencies in the same request).

## Common mistakes

- Forgetting the `yield`-based cleanup pattern, leaking DB
  connections/sessions across requests under load.
- Assuming a dependency runs once per *nested use* rather than once per
  *request* — leads to confusion about caching behavior.
- Not using `app.dependency_overrides` in tests, instead reaching for heavy
  mocking of internal functions — overrides are the idiomatic,
  FastAPI-native way to substitute dependencies.
- Circular dependencies between dependency functions, similar in spirit to
  circular imports (see
  [Modules and Packages](../python/04-modules-and-packages.md)).

## Interview questions

1. How does FastAPI resolve a chain of nested dependencies?

   **Answer:** It walks the dependency graph from the route signature,
   resolves upstream dependencies first, then injects their results into the
   downstream ones. FastAPI also caches each dependency result once per
   request unless you turn that off.

   ```python
   from fastapi import Depends

   def get_token() -> str:
       return "token"

   def get_current_user(token: str = Depends(get_token)) -> str:
       return token
   ```

2. What does a `yield`-based dependency give you that a plain `return`-based
   one doesn't?

   **Answer:** It gives you setup plus guaranteed teardown around the
   request. That's what you want for things like DB sessions where cleanup
   must happen even if the handler raises.

   ```python
   from collections.abc import Iterator

   def get_db() -> Iterator[str]:
       try:
           yield "db-session"
       finally:
           print("closed")
   ```

3. How would you swap a real database dependency for a test database in
   your test suite?

   **Answer:** Override the dependency through `app.dependency_overrides`
   so the routes keep the same contract but receive test infrastructure.
   That's cleaner than mocking route internals.

   ```python
   from fastapi import FastAPI

   app = FastAPI()

   def get_db() -> str:
       return "prod"

   def get_test_db() -> str:
       return "test"

   app.dependency_overrides[get_db] = get_test_db
   ```

4. Is a dependency's result shared across multiple parts of the same
   request that depend on it? Why does that matter?

   **Answer:** Yes, by default FastAPI caches it once per request. That
   matters because you usually want one shared DB session or one resolved
   current user, not duplicate work and inconsistent state.

   ```python
   from fastapi import Depends, FastAPI

   app = FastAPI()

   def get_settings() -> dict[str, str]:
       return {"env": "dev"}

   @app.get("/config")
   def show_config(
       a: dict[str, str] = Depends(get_settings),
       b: dict[str, str] = Depends(get_settings),
   ) -> dict[str, bool]:
       return {"same_object": a is b}
   ```

5. When would you attach a dependency at the router level instead of on
   each individual route?

   **Answer:** Use router-level dependencies when the same check or setup
   applies to every endpoint in that area. Good examples are auth, API key
   checks, tenant resolution, or audit context.

   ```python
   from fastapi import APIRouter, Depends

   def verify_api_key() -> None:
       return None

   router = APIRouter(dependencies=[Depends(verify_api_key)])
   ```

## Senior-level considerations

- `Depends()` is FastAPI's built-in inversion-of-control mechanism —
  understanding it deeply means you rarely need a separate DI framework;
  overreliance on ad hoc global state/singletons instead of dependency
  injection makes an app much harder to test in isolation. For example, a
  route that calls a global `current_db_session` is harder to replace in
  tests than a route that declares `db=Depends(get_db)`.
- Dependency chains that mix request-scoped and application-scoped
  concerns (e.g. a dependency that opens a new DB engine per request
  instead of reusing a pooled engine) are a common source of connection
  pool exhaustion under load — the dependency should typically request a
  session from an already-initialized pool, not create the pool itself.
  For example, initialize `engine = create_engine(...)` once at startup,
  then yield `SessionLocal()` per request.
- Structuring dependencies as small, composable, single-purpose functions
  (rather than a few large ones) makes them easier to override
  individually in tests and easier to reason about in code review. For
  example, keep `get_current_user`, `require_admin`, and `get_db` separate
  instead of hiding all three behaviors inside one large dependency.
