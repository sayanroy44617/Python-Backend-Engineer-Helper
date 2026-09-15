# Middleware and Exception Handling

## What

**Middleware** runs code around every request/response, regardless of
which route handled it (logging, timing, CORS, auth headers). **Exception
handlers** map exceptions raised anywhere in the app to well-formed HTTP
responses. Together they're how cross-cutting request concerns and error
responses are centralized instead of duplicated per route.

## Why

Every production API needs consistent error responses (a stable JSON error
shape, correct status codes) and cross-cutting behavior (request logging,
CORS, timing headers) applied uniformly. Handling these per-route is
repetitive and inconsistent; centralizing them in middleware and exception
handlers keeps the API contract predictable for clients.

## How

### Middleware

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = str(time.perf_counter() - start)
    return response
```

Middleware wraps `call_next(request)` — code before it runs on the way in,
code after runs on the way out. Because it's `async def`, a blocking call
here stalls the event loop for **every** request, not just one (see
[Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md)).

### Built-in middleware

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)
```

CORS, GZip, and trusted-host middleware ship with Starlette/FastAPI —
prefer these over hand-rolled equivalents.

### Exception handlers

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse

class UserNotFoundError(Exception):
    def __init__(self, user_id: int) -> None:
        self.user_id = user_id

@app.exception_handler(UserNotFoundError)
async def handle_user_not_found(request: Request, exc: UserNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"detail": f"user {exc.user_id} not found"},
    )
```

This builds on the custom exception hierarchy pattern from
[Exceptions](../python/03-exceptions.md) — define domain exceptions once,
map them to HTTP responses in one place, and let route handlers simply
`raise UserNotFoundError(user_id)` without knowing about HTTP status codes
at all.

### `HTTPException` — the direct route

```python
from fastapi import HTTPException

@app.get("/users/{user_id}")
def get_user(user_id: int):
    user = repository.get(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

`HTTPException` is FastAPI's built-in shortcut when you want to raise an
HTTP error directly from a route without a custom exception type — simpler
for one-off cases, but less reusable than a domain exception mapped by a
handler when the same error can originate from multiple places (services,
repositories).

### Overriding validation error responses

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={"detail": "invalid request", "errors": exc.errors()},
    )
```

Overriding FastAPI's default validation error handler lets you standardize
error response shape across both validation errors (from Pydantic) and
domain errors (from your own exception handlers).

### A catch-all handler

```python
@app.exception_handler(Exception)
async def handle_unexpected_error(request: Request, exc: Exception):
    logger.exception("unhandled error")
    return JSONResponse(status_code=500, content={"detail": "internal server error"})
```

A last-resort handler ensures unexpected exceptions never leak a raw stack
trace to clients — always log the full exception internally while
returning a generic message externally.

## When to use

- Middleware for concerns that apply to *every* request regardless of
  route: request logging, timing, CORS, request ID propagation.
- Custom exception handlers for domain errors that can originate from
  multiple layers (service, repository) and need consistent HTTP mapping.
- `HTTPException` for simple, route-local errors that don't warrant a
  dedicated exception type.
- A catch-all `Exception` handler in every production app, to guarantee no
  raw internal error ever reaches a client.

## When NOT to use

- Don't put business logic in middleware — it runs for every request
  regardless of route, so it should stay generic and lightweight.
- Don't scatter `try/except HTTPException` blocks in every route when a
  centralized exception handler for a domain exception type would remove
  the duplication.
- Don't return internal error details (stack traces, DB error messages) to
  clients even in a catch-all handler — log them, don't expose them.

## Common mistakes

- Blocking the event loop inside `async def` middleware, silently
  degrading every request's latency (see
  [Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md)).
- Registering middleware/exception handlers in an order that doesn't
  behave as expected — middleware wraps in the order added (last added
  wraps outermost for `add_middleware`), which can surprise you if you
  assumed the reverse.
- Forgetting a catch-all exception handler, so an unexpected bug leaks a
  raw traceback (and potentially sensitive info) directly to API clients.
- Using `HTTPException` for an error that actually originates deep in a
  service/repository layer, forcing that layer to know about HTTP status
  codes instead of raising a domain exception.

## Interview questions

1. What's the execution order of middleware around a request/response?
2. Why would you prefer a custom exception + handler over raising
   `HTTPException` directly from a route?
3. Why is a catch-all exception handler important in production, and what
   should (and shouldn't) it expose to the client?
4. What happens if middleware code blocks the event loop?
5. How would you standardize the JSON error response shape for both
   validation errors and domain errors?

## Senior-level considerations

- A consistent error response contract (status code + stable JSON shape)
  across the whole API is part of your service's contract with consumers —
  changing it later is a breaking change, so design it deliberately early.
- Centralizing domain-to-HTTP error mapping in exception handlers (rather
  than scattering `HTTPException` calls) keeps the service/business logic
  layer free of HTTP concerns — important for keeping that logic reusable
  outside the web layer (e.g. from a CLI or background worker).
- Middleware ordering and cost matter at scale: expensive middleware (e.g.
  synchronous logging to an external system) on every request can become a
  measurable latency tax across the whole API — profile it like any other
  hot path (see [Performance and Profiling](../python/14-performance-and-profiling.md)).
