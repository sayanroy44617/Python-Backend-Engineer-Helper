# Interview Prep: FastAPI

## How to approach FastAPI interviews

FastAPI questions tend to test whether you understand *why* it's
structured the way it is (dependency injection, Pydantic validation,
async-first design) rather than just its decorator syntax — expect
follow-ups like "what happens if this dependency raises" or "why would
this endpoint block the whole event loop," which probe the same
mechanics covered throughout the [FastAPI](../fastapi/index.md) section.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | What does FastAPI's dependency injection (`Depends`) actually solve? | It lets shared logic (DB session creation, auth checks, pagination params) be declared once and reused/composed across endpoints, with FastAPI handling resolution, caching per-request, and cleanup automatically. | [Dependency Injection](../fastapi/03-dependency-injection.md) |
| 2 | How does Pydantic validation differ from manually checking request data in the endpoint body? | Pydantic validates and coerces the request against a declared schema *before* the endpoint function runs, returning a structured 422 error automatically on failure — manual checks are ad hoc, easy to miss a field, and mix validation logic with business logic. | [Pydantic and Validation](../fastapi/02-pydantic-and-validation.md) |
| 3 | What happens if you define an endpoint with `def` instead of `async def` in FastAPI? | FastAPI runs synchronous (`def`) endpoints in a thread pool, off the main event loop — so a blocking call inside it won't block other requests, unlike a blocking call inside an `async def` endpoint. | [Async Endpoints, Background Tasks, and Lifespan](../fastapi/06-async-endpoints-and-background-tasks.md) |
| 4 | Why is calling a blocking library (e.g. a sync DB driver) inside an `async def` endpoint dangerous? | It blocks the single-threaded event loop, stalling every other concurrent request being served by that worker until the blocking call returns — the opposite of what `async def` is meant to provide. | [Async Endpoints, Background Tasks, and Lifespan](../fastapi/06-async-endpoints-and-background-tasks.md) |
| 5 | How would you structure global exception handling in a FastAPI app? | Register exception handlers (`@app.exception_handler`) for specific exception types to convert them into consistent, structured error responses in one place, rather than duplicating try/except blocks across every endpoint. | [Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md) |
| 6 | What's the difference between middleware and a dependency in FastAPI? | Middleware wraps every request/response globally (logging, CORS, timing) regardless of route; a dependency is declared per-route (or per-router) and can access route-specific parameters and participate in FastAPI's DI resolution/caching. | [Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md) |
| 7 | How does FastAPI generate OpenAPI docs automatically, and why does that matter? | It introspects your route signatures, Pydantic models, and type hints to build the OpenAPI schema — meaning accurate types/models directly produce accurate, always-in-sync API documentation without manually maintaining a separate spec. | [OpenAPI and Testing](../fastapi/07-openapi-and-testing.md) |
| 8 | How would you implement JWT-based authentication as a reusable dependency? | A dependency function extracts and validates the token (signature, expiry, claims), raising an `HTTPException` on failure, and returns the authenticated user/claims — reused via `Depends` across every protected endpoint. | [Authentication and Authorization](../fastapi/05-authentication-and-authorization.md) |
| 9 | What's the purpose of FastAPI's `lifespan` context manager? | It runs setup/teardown logic once for the whole application's lifetime (e.g. creating a DB connection pool on startup, closing it on shutdown) rather than per-request. | [Async Endpoints, Background Tasks, and Lifespan](../fastapi/06-async-endpoints-and-background-tasks.md) |
| 10 | How would you structure a FastAPI project as it grows beyond a single `main.py`? | Split by domain/feature into routers, with clear layers for routes, schemas (Pydantic models), services (business logic), and data access — keeping route handlers thin and delegating logic elsewhere for testability. | [Application Architecture](../fastapi/08-application-architecture.md) |
| 11 | Why use `BackgroundTasks` instead of just `await`-ing a slow operation inline? | `BackgroundTasks` lets the response return to the client immediately while the task runs afterward — appropriate for fire-and-forget work (e.g. sending a notification) that shouldn't make the client wait, though it isn't a substitute for a real task queue for anything requiring retries/durability. | [Async Endpoints, Background Tasks, and Lifespan](../fastapi/06-async-endpoints-and-background-tasks.md) |
| 12 | How do you test a FastAPI endpoint that depends on a database? | Override the dependency (via `app.dependency_overrides`) with a test double or a test database session, using `TestClient`/`httpx.AsyncClient` to call the endpoint without needing the real production database. | [OpenAPI and Testing](../fastapi/07-openapi-and-testing.md) |

## Live-coding / whiteboard tips

- If asked to sketch an endpoint, default to declaring a Pydantic
  request/response model rather than accepting a raw `dict` — this is
  usually what the interviewer is checking for.
- When discussing a dependency that needs cleanup (e.g. a DB session),
  mention `yield`-based dependencies specifically, and explain that code
  after `yield` runs as teardown.
- If asked "how would you make this endpoint faster," first ask whether
  the bottleneck is I/O-bound (async helps) or CPU-bound (async alone
  won't help) before proposing a fix.

## Common red flags interviewers watch for

- Marking every endpoint `async def` without knowing whether the code
  inside actually awaits anything, or calling blocking code inside one.
- Reaching straight for raw `dict` request bodies instead of Pydantic
  models, losing validation and OpenAPI accuracy.
- Not knowing the difference between a dependency and middleware.
- Treating `BackgroundTasks` as equivalent to a durable task queue.

## Related deep-dive material

- [FastAPI section overview](../fastapi/index.md) — all 8 topic pages,
  each with its own dedicated interview questions and senior-level
  considerations.
