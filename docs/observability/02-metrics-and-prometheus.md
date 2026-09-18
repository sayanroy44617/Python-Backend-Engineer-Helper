# Metrics and Prometheus

## What

**Metrics** are numeric measurements of a system's behavior over time
(request count, latency, error rate, queue depth). **Prometheus** is the
de facto standard tool for collecting, storing, and querying
time-series metrics in modern backend systems, using a pull-based model.

## Why

Logs answer "what happened in this specific request/event"; metrics
answer "how is the system behaving in aggregate, right now and over
time." You can't reasonably alert on "is error rate elevated" by scanning
individual log lines — metrics aggregate that into a queryable number a
dashboard or alert rule can act on directly.

## How

### The four metric types

```python
from prometheus_client import Counter, Gauge, Histogram, Summary

requests_total = Counter("http_requests_total", "Total HTTP requests", ["method", "status"])
active_connections = Gauge("active_connections", "Current open connections")
request_duration = Histogram("http_request_duration_seconds", "Request duration")
```

| Type | Behavior | Example |
|---|---|---|
| Counter | Only increases (resets on restart) | Total requests served |
| Gauge | Goes up or down | Current queue depth, active connections |
| Histogram | Buckets observations, enables percentile queries | Request latency distribution |
| Summary | Similar to histogram, computes quantiles client-side | Request latency (less common than histogram in practice) |

Choosing the right type matters for what you can query later — a counter
can never answer "what's the current value," and a gauge can't answer
"how many total requests have there been since deployment."

### Instrumenting a FastAPI app

```python
from prometheus_client import Counter, Histogram, make_asgi_app
import time

requests_total = Counter("http_requests_total", "Total requests", ["method", "path", "status"])
request_duration = Histogram("http_request_duration_seconds", "Request duration", ["path"])

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.monotonic()
    response = await call_next(request)
    duration = time.monotonic() - start
    requests_total.labels(request.method, request.url.path, response.status_code).inc()
    request_duration.labels(request.url.path).observe(duration)
    return response

app.mount("/metrics", make_asgi_app())  # Prometheus scrapes this endpoint
```

This reuses the middleware pattern from
[Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md)
— metrics collection is exactly the kind of cross-cutting, per-request
concern middleware is designed for. `/metrics` exposes the current
values in Prometheus's text format for scraping.

### The pull model

```
Prometheus server periodically GETs http://your-service/metrics
(rather than your service pushing metrics out)
```

Prometheus's pull-based model means your application just exposes current
metric values on an endpoint; Prometheus itself is responsible for
scraping on a schedule and storing the time series — this simplifies the
application side (no metrics-shipping client to configure/fail) at the
cost of Prometheus needing network access to reach every instance.

### Labels and cardinality

```python
# LOW cardinality -- a small, bounded set of possible values
requests_total.labels(method="GET", path="/users/{id}", status=200)

# DANGEROUS -- high cardinality; a new time series per unique user ID
requests_total.labels(method="GET", path=f"/users/{user_id}", status=200)
```

Labels let you slice a metric along dimensions (method, status code,
endpoint) — but each unique label combination creates a separate time
series in Prometheus's storage. Using an unbounded value (a raw user ID,
a full URL with query params) as a label causes **cardinality
explosion** — potentially millions of time series, which can degrade or
crash a Prometheus instance.

### Querying with PromQL

```promql
# Requests per second over the last 5 minutes, by status code
rate(http_requests_total[5m])

# 95th percentile request latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate as a percentage of total requests
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

`rate()` computes the per-second average increase of a counter over a
time window — counters are cumulative, so you almost always query their
*rate of change*, not their raw value, to get a meaningful "requests per
second" style metric.

### The RED and USE methods

```
RED (for services):
  Rate     -- requests per second
  Errors   -- failed requests per second
  Duration -- latency distribution

USE (for resources):
  Utilization -- % time the resource was busy
  Saturation  -- how much extra work is queued
  Errors      -- error count
```

These are widely used frameworks for deciding *what* to measure: RED for
request-driven services (APIs), USE for underlying resources (CPU,
database connections, queue depth) — a useful starting checklist rather
than measuring arbitrarily.

### Application-level vs. business metrics

```python
orders_placed = Counter("orders_placed_total", "Total orders placed", ["payment_method"])
order_value = Histogram("order_value_dollars", "Order value distribution")
```

Beyond infrastructure metrics (latency, error rate), tracking business-
relevant metrics (orders placed, signups, revenue) in the same system lets
engineering and product/business stakeholders share the same
observability tooling for very different questions.

## When to use

- Counters for cumulative totals (requests served, errors), gauges for
  current state (queue depth, active connections), histograms for latency
  distributions.
- Low-cardinality labels only — bounded sets of known values (status
  code, method, a small fixed set of endpoint patterns).
- The RED method for instrumenting a new service's request path from
  scratch, as a sane default checklist.

## When NOT to use

- Don't use a raw, high-cardinality value (user ID, full request path
  with dynamic segments) as a label — it risks cardinality explosion.
- Don't reach for metrics to answer "what exactly happened in this one
  request" — that's what structured logs (and traces) are for; metrics
  are aggregate by design and lose per-event detail.
- Don't instrument everything indiscriminately — start with the RED/USE
  basics and add metrics deliberately as specific questions arise, rather
  than measuring everything "just in case."

## Common mistakes

- Using a Counter to represent something that can decrease (should be a
  Gauge instead), producing a metric that can't be queried meaningfully.
- High-cardinality labels (raw user IDs, unbounded query parameters)
  quietly degrading Prometheus performance over time.
- Querying a Counter's raw value instead of `rate(...)`, which is rarely
  the meaningful question for a cumulative counter.
- No metrics at all until an incident, then trying to retrofit
  instrumentation while already firefighting — metrics need to exist
  *before* they're needed.

## Interview questions

1. What's the difference between a Counter and a Gauge, and why can't a
   Counter represent something like "current active connections"?

   **Answer:** A Counter only goes up, apart from process restarts, so it's for totals like requests served or jobs completed. Active connections go up and down, so modeling them with a Counter gives you the wrong shape of data.

   ```python
   from prometheus_client import Gauge

   active_connections = Gauge("active_connections", "Current open connections")
   active_connections.inc()
   active_connections.dec()
   ```

2. Why does Prometheus use a pull model, and what's the practical
   implication for how your application needs to be instrumented?

   **Answer:** With pull, your app just exposes current metric values and Prometheus comes and scrapes them on a schedule. In practice, that means your service needs a reachable `/metrics` endpoint instead of custom push logic for normal collection.

3. What is cardinality explosion, and how would a poorly chosen label
   cause it?

   **Answer:** Cardinality explosion happens when one metric turns into too many unique time series because label values are unbounded. A label like `user_id` or raw URL can create a new series for every request and overwhelm Prometheus storage and query performance.

4. Why do you almost always query `rate(counter[5m])` rather than a raw
   counter value in PromQL?

   **Answer:** A raw counter mostly tells you that time has passed and requests accumulated; it keeps increasing for the life of the process. `rate(...)` converts that cumulative number into something operationally useful, like requests per second over a recent window.

5. What do the RED and USE methods each focus on, and when would you use
   each?

   **Answer:** RED is for request-driven services and focuses on rate, errors, and duration, so it's a good default for APIs. USE is for resources like CPU, DB pools, or queues, where you care about utilization, saturation, and errors.

## Senior-level considerations

- Metric design (what to measure, what labels to use) should anticipate
  the questions you'll need to answer during an incident — "can I filter
  by endpoint and status code to isolate this problem" needs to be
  designed in ahead of time, not discovered as a gap mid-incident — for
  example, `method`, route template, and status code are usually worth
  capturing on HTTP request metrics from day one.
- Cardinality is a genuine operational risk at scale, not just a
  theoretical concern — a single poorly chosen label in a
  high-traffic service can degrade an entire monitoring stack shared
  across a whole organization — for example, labeling a request metric by
  raw `user_id` in a busy consumer app can create millions of series.
- Metrics, logs, and traces are complementary, not redundant — a mature
  observability strategy uses metrics for aggregate trend/alerting,
  logs for detailed per-event context, and traces (see
  [Tracing and OpenTelemetry](03-tracing-and-opentelemetry.md)) for
  understanding cross-service request flow, each answering a different
  class of question — for example, metrics tell you checkout latency is
  up, logs show which orders failed, and traces show the slowdown is in
  the payment service hop.
