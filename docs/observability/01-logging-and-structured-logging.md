# Logging and Structured Logging

## What

**Logging** records discrete events as a system runs. **Structured
logging** emits those events as machine-parseable data (typically JSON)
with consistent fields, rather than free-form text — enabling logs to be
searched, filtered, and aggregated reliably at scale.

## Why

Free-form log messages ("User 123 logged in from 1.2.3.4") are fine for a
human tailing a single file, but become nearly unusable once a system runs
across many instances and logs are aggregated centrally — you can't
reliably query "all failed logins for user 123 in the last hour" out of
unstructured text without fragile regex parsing. Structured logs make
that a simple field-based query.

## How

### Unstructured vs. structured logging

```python
# Unstructured -- readable, but not reliably queryable
logger.info(f"User {user_id} logged in from {ip_address}")

# Structured -- same information, but as queryable fields
logger.info("user_login", extra={"user_id": user_id, "ip_address": ip_address})
```

The structured version lets a log aggregation system (Elasticsearch,
Datadog, CloudWatch Logs Insights) filter/aggregate on `user_id` or
`ip_address` directly, rather than needing to parse them back out of a
sentence.

### Using Python's `logging` module correctly

```python
import logging

logger = logging.getLogger(__name__)  # module-level logger, not the root logger

def process_order(order_id: int) -> None:
    logger.info("processing order", extra={"order_id": order_id})
```

`logging.getLogger(__name__)` gives each module its own named logger
(reflecting the module hierarchy), letting you configure verbosity per
module/package rather than only globally — using the root logger
directly everywhere loses this granularity.

### Log levels and when to use each

| Level | Use for |
|---|---|
| `DEBUG` | Detailed diagnostic info, useful only during active debugging |
| `INFO` | Normal operational events worth recording (request handled, job started) |
| `WARNING` | Something unexpected but recoverable (a retry, a fallback path taken) |
| `ERROR` | An operation failed and needs attention, but the process continues |
| `CRITICAL` | The application itself is in a state where it likely can't continue correctly |

Consistent level usage across a codebase is what makes log-level-based
filtering (e.g. "show me only WARNING and above in production") actually
meaningful — logging routine operational info at `ERROR` (or genuine
errors at `INFO`) undermines this.

### JSON structured logging with `structlog`

```python
import structlog

structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ]
)
logger = structlog.get_logger()

logger.info("order_processed", order_id=42, total=99.50, duration_ms=120)
```

```json
{"event": "order_processed", "order_id": 42, "total": 99.5, "duration_ms": 120, "timestamp": "2024-01-01T12:00:00Z"}
```

`structlog` (or the standard library's `logging` with a JSON formatter)
outputs each log entry as a JSON object — the standard format log
aggregation systems expect, and directly queryable/filterable by any
field without text parsing.

### Correlation/request IDs

```python
import contextvars

request_id_var = contextvars.ContextVar("request_id", default=None)

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    request_id = str(uuid.uuid4())
    request_id_var.set(request_id)
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

logger.info("processing", extra={"request_id": request_id_var.get()})
```

A **correlation ID** (request ID) attached to every log line for a given
request lets you reconstruct the full sequence of events for one request
across every log statement it triggered — essential once a single
request touches multiple functions, services, or async tasks (see
[Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md)
for the middleware pattern this builds on). `contextvars` (rather than a
plain global) is what makes this safe under
[asyncio concurrency](../python/13-asyncio-and-concurrency.md) — each
concurrent request gets its own isolated value.

### What to log (and what not to)

```python
# GOOD -- operational context, no sensitive data
logger.info("payment_processed", order_id=42, amount=99.50, payment_method="card")

# BAD -- logs sensitive data that shouldn't end up in a log aggregation
# system (see Injection Attacks and Secrets Management)
logger.info("payment_processed", card_number=card_number, cvv=cvv)
```

Log enough context to debug and audit meaningfully (IDs, key state
transitions, timing), but never log secrets, full card numbers, passwords,
or tokens — see
[Injection Attacks and Secrets Management](../security/05-injection-attacks-and-secrets-management.md)
for why logs are a common, easy-to-overlook place secrets leak.

### Exception logging

```python
try:
    process_order(order_id)
except Exception:
    logger.exception("order_processing_failed", extra={"order_id": order_id})
    raise
```

`logger.exception(...)` (called from within an `except` block) automatically
includes the full traceback — always prefer it over `logger.error(str(e))`,
which discards the traceback and makes root-causing the failure much
harder.

### Log volume and sampling

```python
# High-frequency, low-value logs -- sample rather than log every occurrence
if random.random() < 0.01:  # log ~1% of these
    logger.debug("cache_hit", key=cache_key)
```

At high request volume, logging every single occurrence of a very
frequent, low-diagnostic-value event can overwhelm log storage/ingestion
cost — sampling (or simply lowering the level and adjusting production
verbosity) keeps volume manageable while retaining enough signal for
debugging.

## When to use

- Structured (JSON) logging as the default in any service whose logs are
  aggregated centrally — which is effectively every production service.
- A request/correlation ID propagated through every log line for a given
  request, especially once multiple services or async tasks are involved.
- `logger.exception(...)` specifically inside `except` blocks, to
  preserve the traceback.
- Consistent log levels across the codebase so level-based filtering in
  production is meaningful.

## When NOT to use

- Don't log sensitive data (passwords, tokens, full payment details) —
  treat logs as a system with weaker access control than your primary
  database by default.
- Don't log at `INFO` (or above) for extremely high-frequency, low-value
  events without sampling — it drowns out genuinely important log entries
  and inflates log storage cost.
- Don't rely on unstructured, free-text log messages in a production
  system whose logs are aggregated and searched at any meaningful scale.

## Common mistakes

- Using `logger.error(str(e))` instead of `logger.exception(...)`,
  silently discarding the traceback needed to actually debug the failure.
- Logging sensitive fields (passwords, card numbers, tokens) directly,
  creating a compliance and security exposure in the logging pipeline
  itself.
- Inconsistent log levels across a codebase (routine events at `ERROR`,
  genuine failures at `INFO`), making level-based filtering useless.
- No correlation/request ID, making it very difficult to reconstruct what
  happened for one specific request once logs from many concurrent
  requests are interleaved.

## Interview questions

1. Why does structured (JSON) logging matter more as a system scales,
   compared to free-form text logs?
2. What's the practical difference between `logger.error(str(e))` and
   `logger.exception(...)`, and why does it matter?
3. How would you correlate all log lines belonging to a single request
   across multiple functions or services?
4. Why is `contextvars` (rather than a plain global variable) the right
   tool for storing a per-request correlation ID under async concurrency?
5. What kinds of data should never appear in application logs, and why?

## Senior-level considerations

- Logging strategy (what to log, at what level, with what fields) should
  be a deliberate design decision made alongside the code it instruments,
  not an afterthought added during an incident when it's too late to
  capture the missing context retroactively.
- Log volume and cost scale with traffic — sampling, retention policy, and
  level tuning are ongoing operational concerns for any system at
  meaningful scale, not a one-time setup.
- Correlation IDs and structured fields are the foundation that
  [tracing](03-tracing-and-opentelemetry.md) builds on — designing
  consistent, propagated identifiers early makes adopting full distributed
  tracing later far less disruptive.
