# Tracing and OpenTelemetry

## What

**Distributed tracing** records the full path a single request takes as
it flows through multiple services/functions, as a tree of timed
**spans**. **OpenTelemetry** (OTel) is the vendor-neutral standard API/SDK
for producing traces (and metrics/logs) that can be exported to any
compatible backend (Jaeger, Tempo, Datadog, etc.).

## Why

A correlation ID in logs (see
[Logging and Structured Logging](01-logging-and-structured-logging.md))
lets you find all log lines for one request, but reconstructing *timing
and causality* — which call took how long, which downstream service was
the bottleneck — from scattered log lines across services is slow and
error-prone. Tracing captures this structure directly: one request, one
trace, with each unit of work as a span showing exactly where time was
spent.

## How

### Traces and spans

```
Trace: "GET /checkout" (total: 340ms)
├── span: validate_cart (12ms)
├── span: call payment-service (180ms)
│   └── span: charge_card (170ms)   <- happens inside payment-service
├── span: call inventory-service (90ms)
└── span: send_confirmation_email (8ms, async, doesn't block response)
```

A **trace** represents one end-to-end request; each **span** is a named,
timed unit of work within it, possibly nested (a span calling into
another service creates a child span there). This structure immediately
shows *where* time went — here, the payment service call is clearly the
dominant cost.

### Instrumenting with OpenTelemetry (Python)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
tracer = trace.get_tracer(__name__)

def process_order(order_id: int) -> None:
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order_id", order_id)
        validate_order(order_id)
        charge_payment(order_id)
```

`start_as_current_span` both creates a span and makes it the "current"
one — any span started inside this `with` block is automatically nested
as its child, building the trace tree without manually threading a
parent reference through every function call.

### Auto-instrumentation for FastAPI

```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

FastAPIInstrumentor.instrument_app(app)
```

OTel provides auto-instrumentation packages for common frameworks
(FastAPI, `requests`/`httpx`, SQLAlchemy) that automatically create spans
for incoming requests, outgoing HTTP calls, and database queries without
manual `start_as_current_span` calls everywhere — manual spans are then
reserved for business-logic-specific units of work you want visible in a
trace beyond what auto-instrumentation already covers.

### Context propagation across service boundaries

```http
GET /orders/42 HTTP/1.1
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

For a trace to span multiple services, the trace context (trace ID,
parent span ID) must travel with the request — typically via the
`traceparent` HTTP header (the W3C Trace Context standard OTel
implements by default). Without propagating this header on every
outgoing call, each service would start its own disconnected trace
instead of contributing to one unified one.

```python
import httpx
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor

HTTPXClientInstrumentor().instrument()  # automatically injects traceparent
```

Auto-instrumenting the HTTP client library used for outgoing calls is
what makes this propagation automatic rather than something every call
site needs to handle manually.

### Sampling

```python
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased

sampler = TraceIdRatioBased(0.1)  # trace ~10% of requests
```

Tracing every single request in a very high-traffic system generates
substantial data volume and export overhead — sampling (tracing only a
percentage of requests, or always tracing errors/slow requests
specifically) keeps volume manageable while preserving enough
representative data for diagnosing typical behavior.

### Correlating traces with logs

```python
span = trace.get_current_span()
trace_id = format(span.get_span_context().trace_id, "032x")
logger.info("processing order", extra={"trace_id": trace_id, "order_id": order_id})
```

Including the trace ID in structured log lines lets you jump from "I see
an elevated error rate in this metric/dashboard" to "here's the exact
trace for one affected request" to "here are the detailed log lines for
that same request" — tracing, metrics, and logging working together
rather than as three disconnected tools.

### What a trace reveals that logs and metrics don't

```
Metrics tell you: "p99 latency for /checkout went up"
Logs tell you:    "here's what happened in one specific request"
Traces tell you:  "here's EXACTLY which downstream call in that request
                    took 280ms out of the total 340ms"
```

This is the core value proposition of tracing in a distributed
system: pinpointing *where in the call graph* time was spent or an error
originated, which becomes genuinely difficult to determine from logs and
metrics alone once a request spans several services.

## When to use

- Distributed tracing as soon as a request meaningfully spans more than
  one service (or several significant internal stages) — its value is
  proportional to how much a single request fans out.
- Auto-instrumentation for frameworks/libraries you're already using
  (FastAPI, database drivers, HTTP clients) before writing manual spans.
- Sampling (rather than tracing every request) once traffic volume makes
  100% tracing costly, while always tracing errors/slow outliers.

## When NOT to use

- Don't manually instrument every function with a span — start with
  auto-instrumentation and add manual spans only for business-logic
  boundaries you specifically want visible.
- Don't skip context propagation on outgoing calls — an un-propagated
  trace context breaks the trace into disconnected fragments per service,
  losing most of tracing's value.
- Don't trace 100% of requests in a very high-volume system without
  considering sampling — the storage/export cost can become substantial
  for limited additional insight beyond a representative sample.

## Common mistakes

- Forgetting to propagate trace context on outgoing HTTP calls, producing
  disconnected per-service traces instead of one unified end-to-end trace.
- Manually instrumenting spans for things auto-instrumentation already
  covers, duplicating effort and cluttering the trace with redundant spans.
- No correlation between trace IDs and log lines, losing the ability to
  pivot from a trace to detailed logs for the same request.
- Treating tracing as a replacement for metrics/logging rather than a
  complementary tool — each answers a different diagnostic question.

## Interview questions

- What's the difference between a trace and a span, and how do child
    spans relate to their parent?

    **Answer:** A trace is the full end-to-end story for one request, while a span is one timed step inside that story. Child spans represent work done inside a parent operation, so the trace shows both nesting and where the time actually went.

- How does trace context propagate across service boundaries, and what
    breaks if it doesn't?

    **Answer:** The caller sends trace metadata, usually with the `traceparent` header, and the downstream service continues the same trace instead of starting a new one. If that header is missing, you get disconnected traces and lose the end-to-end view.

    ```python
    headers: dict[str, str] = {"traceparent": "00-abc-123-01"}
    ```

- Why is sampling necessary in a high-traffic system, and what's a
    reasonable strategy beyond a flat percentage?

    **Answer:** Full tracing gets expensive fast in busy systems because you're storing and exporting a lot of span data. A practical strategy is to sample a small baseline of normal traffic but keep all erroring or unusually slow requests.

- What can a trace tell you about a slow request that a log correlation
    ID alone cannot?

    **Answer:** A correlation ID helps you collect the logs for one request, but you still have to infer timing and causality from scattered messages. A trace shows the call tree directly, so you can see that, for example, `payment-service` consumed 280ms of a 340ms request.

- Why does auto-instrumentation typically cover most of what's needed,
    with manual spans reserved for specific cases?

    **Answer:** Most of the useful plumbing is already at framework boundaries like inbound requests, outbound HTTP calls, and database queries, so auto-instrumentation captures a lot with low effort. Manual spans are better for business steps you care about, like `price_quote` or `fraud_check`.

## Senior-level considerations

- Tracing infrastructure (context propagation, exporters, a tracing
  backend) is most valuable exactly in the microservices/distributed
  architectures where it's also most complex to fully instrument
  correctly — planning for it as services are first split apart is far
  easier than retrofitting it once dozens of services exist; for
  example, standardizing on one HTTP client middleware for trace
  propagation is easy with 3 services and painful with 30.
- The combination of correlated logs, metrics, and traces (often called
  "the three pillars of observability") is what makes root-causing a
  production incident in a distributed system tractable — investing in
  this correlation (shared trace/request IDs across all three) pays off
  disproportionately during an actual incident; for example, an alert on
  checkout p99 can link to a trace, and that same trace ID can pull the
  exact payment-service logs.
- Sampling strategy is itself a design decision with trade-offs — biasing
  sampling toward errors and slow requests (rather than uniform random
  sampling) maximizes diagnostic value per unit of tracing overhead/cost;
  for example, you might sample 1% of healthy requests but 100% of 5xx
  responses and requests slower than 1 second.
