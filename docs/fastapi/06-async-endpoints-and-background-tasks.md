# Async Endpoints, Background Tasks, and Lifespan

## What

FastAPI route handlers can be defined `async def` or plain `def`, with
different execution models under the hood. **Background tasks** run code
after a response is sent, without making the client wait. **Lifespan**
hooks run setup/teardown code once at application startup/shutdown (e.g.
opening/closing a connection pool).

## Why

Choosing `async def` vs `def` incorrectly is one of the most common
FastAPI production mistakes — it directly affects whether a slow operation
stalls every other concurrent request. Background tasks and lifespan hooks
are the standard tools for "do this without blocking the response" and
"set this up once, not per-request," respectively.

## How

### `async def` vs `def` route handlers

```python
@app.get("/fast-io")
async def fast_io_endpoint():
    result = await async_db_call()  # non-blocking, cooperates with the event loop
    return result

@app.get("/sync-cpu")
def sync_cpu_endpoint():
    result = cpu_heavy_function()  # FastAPI runs this in a thread pool automatically
    return result
```

FastAPI runs `async def` handlers directly on the event loop; it runs plain
`def` handlers in a separate thread pool automatically, so a blocking call
inside a `def` handler doesn't stall the event loop the way it would inside
`async def` (see
[Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md) for
exactly why blocking the loop is dangerous). The practical rule: use
`async def` only when everything inside it is truly non-blocking
(async DB drivers, `httpx.AsyncClient`, `await asyncio.sleep`); use plain
`def` for handlers that call blocking/synchronous code you can't easily
make async.

```python
# Anti-pattern: blocking call inside async def stalls the WHOLE event loop
@app.get("/bad")
async def bad_endpoint():
    time.sleep(2)  # blocks every other concurrent request too
    return {"ok": True}
```

### Background tasks

```python
from fastapi import BackgroundTasks

def send_welcome_email(email: str) -> None:
    ...  # slow: talks to an external email provider

@app.post("/users")
def create_user(payload: UserCreate, background_tasks: BackgroundTasks):
    user = save_user(payload)
    background_tasks.add_task(send_welcome_email, user.email)
    return user  # response is returned immediately; email sends after
```

`BackgroundTasks` runs the given function **after** the response has been
sent to the client, in the same process. It's for lightweight, best-effort
work — not a substitute for a real task queue (see the Messaging section)
when you need retries, persistence across restarts, or distributed
workers.

### Lifespan (startup/shutdown)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    app.state.db_pool = await create_db_pool()
    yield
    # shutdown
    await app.state.db_pool.close()

app = FastAPI(lifespan=lifespan)
```

`lifespan` is an async context manager (see
[Context Managers and Descriptors](../python/10-context-managers-and-descriptors.md))
— code before `yield` runs once at startup, code after runs once at
shutdown. This is the modern replacement for the older `@app.on_event
("startup")`/`@app.on_event("shutdown")` decorators, which are deprecated.

### Why lifespan matters for connection pools

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.engine = create_async_engine(DATABASE_URL, pool_size=10)
    yield
    await app.state.engine.dispose()
```

Creating a DB connection pool **once** at startup (not per-request) is
essential — creating a new pool per request would exhaust database
connections under load and defeats the purpose of pooling entirely.

## When to use

- `async def` when every I/O call inside the handler is genuinely
  async-native (async DB driver, async HTTP client).
- Plain `def` for handlers wrapping synchronous/blocking libraries — let
  FastAPI's thread pool isolate them from the event loop.
- `BackgroundTasks` for quick, best-effort, non-critical follow-up work
  (sending a notification, writing an audit log entry) that shouldn't
  delay the response.
- `lifespan` for anything that should be created once and reused across
  requests: DB connection pools, HTTP client sessions, ML model loading.

## When NOT to use

- Don't use `BackgroundTasks` for work that must survive a process
  restart, needs retries, or needs to run on a different machine — use a
  real task queue (Celery, RQ, or a message broker — see the Messaging
  section) instead.
- Don't create a new DB connection/pool/HTTP client inside each request
  handler — initialize it once via `lifespan` and store it on `app.state`.
- Don't mark a handler `async def` just because it's the "modern" style if
  it internally calls blocking code — that's worse than plain `def`, which
  at least gets thread-pool isolation.

## Common mistakes

- Calling a blocking/synchronous function inside `async def`, stalling the
  entire event loop for all concurrent requests (the single most common
  FastAPI async bug).
- Creating a new database engine/connection pool per request instead of
  once at startup via `lifespan`.
- Relying on `BackgroundTasks` for critical operations (e.g. charging a
  payment) that need guaranteed delivery/retries — if the process crashes
  after the response is sent but before the background task runs, the work
  is lost.
- Using the deprecated `@app.on_event("startup")` style in new code instead
  of the `lifespan` context manager.

## Interview questions

1. What's the practical difference between `async def` and `def` route
   handlers in FastAPI?

   **Answer:** `async def` runs on the event loop, so it only helps when the work inside is actually non-blocking. Plain `def` runs in FastAPI's thread pool, which is usually the safer choice for sync libraries.

   ```python
   async def fetch_user() -> dict[str, int]:
       return {"id": 1}

   def parse_report() -> dict[str, bool]:
       return {"ok": True}
   ```
2. Why would a blocking call inside `async def` be worse than the same
   call inside plain `def`?

   **Answer:** In `async def`, a blocking call freezes the event loop, so unrelated requests get stuck too. In plain `def`, FastAPI isolates that blocking work in a worker thread instead of choking the loop.

   ```python
   import time

   async def bad_endpoint() -> dict[str, bool]:
       time.sleep(1)
       return {"ok": True}
   ```
3. What is `BackgroundTasks` appropriate for, and where does it fall short
   compared to a real task queue?

   **Answer:** It's fine for quick follow-up work that is okay to lose, like sending a non-critical email or writing a best-effort audit line. It falls short when you need retries, persistence, scheduling, or work to survive a process crash.

   ```python
   from fastapi import BackgroundTasks

   def enqueue_email(background_tasks: BackgroundTasks, email: str) -> None:
       background_tasks.add_task(print, f"send welcome email to {email}")
   ```
4. Why should a DB connection pool be created in `lifespan` rather than
   inside each request handler?

   **Answer:** A pool is meant to be long-lived and shared; creating one per request defeats pooling and can burn through database connections fast. `lifespan` gives you one setup point at startup and one cleanup point at shutdown.
5. What replaced `@app.on_event("startup")`, and what problem did that
   change solve?

   **Answer:** The `lifespan` async context manager replaced it. It puts startup and shutdown logic in one explicit place, which makes resource setup and cleanup easier to reason about together.

   ```python
   from contextlib import asynccontextmanager

   @asynccontextmanager
   async def lifespan(app):
       yield
   ```

## Senior-level considerations

- Recognizing the `async def` + blocking-call anti-pattern is one of the
  highest-value production debugging skills for FastAPI services — it
  manifests as latency spikes across *unrelated* endpoints, not just the
  offending one; for example, one `time.sleep()` in `/reports` can slow down
  `/health` and `/users` too because they share the same event loop.
- `BackgroundTasks` vs a real task queue is an architecture decision about
  durability guarantees: BackgroundTasks work runs in-process and is lost
  on crash/restart; a task queue persists work and can retry — choose
  based on whether the operation is genuinely best-effort or must not be
  lost; for example, "send marketing email" can be best-effort, but "charge
  card and issue invoice" usually cannot.
- `lifespan`-managed resources (pools, clients) tie the application's
  resource lifecycle to the process lifecycle — this matters for graceful
  shutdown in containerized/Kubernetes environments, where the process
  needs to drain in-flight requests and close connections cleanly before
  terminating; for example, on SIGTERM you want the app to stop taking new
  traffic, finish current requests, then close the DB pool cleanly.
