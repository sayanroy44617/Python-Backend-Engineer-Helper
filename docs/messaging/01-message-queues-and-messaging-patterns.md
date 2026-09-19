# Message Queues and Messaging Patterns

## What

A **message queue** is an intermediary that stores messages produced by
one service until a **consumer** processes them, decoupling the producer
from having to know who (or how many) will handle its messages, or when.

## Why

Direct, synchronous service-to-service calls couple the caller's
availability and latency to the callee's — if the callee is slow or down,
the caller is blocked or fails too. A message queue breaks this coupling:
the producer can succeed and move on the instant the message is
durably queued, regardless of whether (or how quickly) a consumer
processes it.

## How

### Point-to-point vs publish/subscribe

```
Point-to-point (queue):
  Producer -> [Queue] -> exactly one consumer processes each message
  (many consumers can share a queue for load-balancing, but each
  message goes to only one of them)

Publish/subscribe (topic/exchange):
  Producer -> [Topic] -> every subscriber gets its own copy of each
  message
```

Point-to-point is the right model for work distribution (e.g. "process
this image" — only one worker should do it). Pub/sub is right for
broadcasting an event to multiple independent interested parties (e.g.
"order placed" triggers both an email service and an inventory service,
each unaware of the other).

### A minimal producer/consumer with a broker (RabbitMQ example)

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
channel = connection.channel()
channel.queue_declare(queue="emails", durable=True)

channel.basic_publish(
    exchange="",
    routing_key="emails",
    body=json.dumps({"to": "user@example.com", "template": "welcome"}),
    properties=pika.BasicProperties(delivery_mode=2),  # persist to disk
)
```

```python
def callback(ch, method, properties, body):
    send_email(json.loads(body))
    ch.basic_ack(delivery_tag=method.delivery_tag)  # only ack after success

channel.basic_consume(queue="emails", on_message_callback=callback)
channel.start_consuming()
```

`delivery_mode=2` persists the message to disk so a broker restart doesn't
lose it; acknowledging (`basic_ack`) only *after* successfully processing
is what makes redelivery-on-failure work — acking too early (before
processing) risks losing a message if the consumer crashes mid-work.

### Decoupling in time, not just in call structure

```
Synchronous call:  caller blocked until callee responds (or times out)
Queue-based call:  caller returns immediately after the message is
                    durably queued; the consumer may process it a
                    millisecond or an hour later
```

This time decoupling is what makes queues valuable for smoothing traffic
spikes: a sudden burst of producer activity queues up and drains at the
consumer's own sustainable pace, rather than overwhelming a downstream
service directly (compare to
[Rate Limiting and API Security](../rest-api/05-rate-limiting-and-api-security.md),
which protects a *synchronous* API from the same kind of burst).

### Fan-out and event-driven architecture

```
"OrderPlaced" event published once
  -> Inventory service consumes it, reserves stock
  -> Email service consumes it, sends confirmation
  -> Analytics service consumes it, records the event
```

Each consumer is independent — adding a new consumer of "OrderPlaced"
(e.g. a fraud-detection service) requires no change to the producer or
any existing consumer, a core benefit of event-driven, pub/sub-based
architectures over direct service-to-service calls.

### Queues vs background task systems

```python
# A task queue framework (e.g. Celery) sits on top of a broker,
# adding scheduling, retries, and result tracking as a Python API
@app.task
def send_welcome_email(user_id: int):
    ...

send_welcome_email.delay(user_id=123)
```

Task queue frameworks (Celery, RQ, Dramatiq) use a message broker
underneath but add developer-facing conveniences (task definitions,
retries, scheduling) — useful for offloading work from a request/response
cycle (see
[Async Endpoints, Background Tasks, and Lifespan](../fastapi/06-async-endpoints-and-background-tasks.md)
for FastAPI's own `BackgroundTasks`, which is suitable only for very
short, best-effort, in-process work — not a substitute for a real queue
when durability or retries matter).

### Message ordering

```
Single queue, single consumer:  strict order preserved
Single queue, multiple consumers (competing): order NOT guaranteed
  across consumers -- message 2 might finish before message 1
```

Ordering guarantees are a real design constraint — if your consumers
process messages concurrently for throughput, you generally give up
strict global ordering unless you partition work so related messages
always land on the same consumer (see
[Kafka Fundamentals](02-kafka-fundamentals.md) for how partitioning
addresses this directly).

## When to use

- Decoupling a slow, non-critical-path operation (sending an email,
  generating a report, updating a search index) from the request that
  triggers it.
- Fan-out to multiple independent consumers reacting to the same event,
  without coupling them to each other or to the producer.
- Smoothing bursty traffic so a downstream consumer processes at a
  sustainable rate rather than being overwhelmed by a spike.

## When NOT to use

- Don't use a queue for something that genuinely needs a synchronous
  response before the caller can proceed (e.g. "is this payment
  authorized" needed before showing a confirmation page).
- Don't reach for a full message broker for very short, best-effort,
  in-process work where a framework's own background-task mechanism
  (e.g. FastAPI's `BackgroundTasks`) is sufficient and durability doesn't
  matter.
- Don't assume strict message ordering across concurrent consumers
  without deliberately partitioning work to preserve it where required.

## Common mistakes

- Acknowledging a message before processing succeeds, losing it silently
  if the consumer crashes mid-processing.
- Treating a task queue framework (Celery/RQ) as a magic reliability
  layer without understanding the broker underneath and its own delivery
  guarantees.
- Assuming message order is preserved across multiple competing
  consumers on the same queue.
- Using a queue where a direct synchronous call was actually the right
  fit, adding unnecessary complexity and latency for something the
  caller genuinely needed to wait on.

## Interview questions

- What's the practical difference between a point-to-point queue and a
    publish/subscribe topic?

    **Answer:** A point-to-point queue delivers each message to exactly
    one consumer (competing consumers share the work). A pub/sub topic
    delivers each message to every subscriber independently — one message
    fans out to many, not to just one.

- Why does acknowledging a message only after successful processing
    matter for reliability?

    **Answer:** If you ack (mark as done) before processing finishes and
    the consumer crashes mid-processing, the message is lost forever —
    the broker already thinks it was handled. Acking after success means a
    crash leaves the message unacked, so it gets redelivered.

- How does a message queue help absorb a sudden traffic spike that would
    otherwise overwhelm a downstream service?

    **Answer:** The queue holds incoming messages as a buffer, and the
    downstream service pulls from it at its own sustainable pace — so a
    burst of 10,000 requests doesn't have to be processed all at once, it
    just makes the queue temporarily longer.

- What are the benefits of an event-driven fan-out architecture over
    direct service-to-service calls?

    **Answer:** The producer doesn't need to know or call every consumer
    directly — it just publishes an event, and any number of services can
    subscribe independently. Adding a new consumer later requires zero
    changes to the producer.

- Why might strict message ordering be lost when multiple consumers
    compete for messages on the same queue, and how would you preserve it
    when needed?

    **Answer:** With multiple competing consumers, message B can finish
    processing before message A if A happens to take longer — there's no
    guarantee of who finishes first. To preserve order for related
    messages, route them to the same consumer (e.g. by a partition/routing
    key so all of one entity's events go to one worker).

## Senior-level considerations

- Choosing between direct synchronous calls, background tasks, and a full
  message queue is an architectural decision driven by durability,
  ordering, and coupling requirements — not just "make it async." For
  example, a "send welcome email" can be a fire-and-forget background
  task, but "charge the customer" needs the durability guarantees of a
  real queue.
- Event-driven fan-out architectures trade coupling for eventual
  consistency and operational complexity (more moving parts, harder
  end-to-end tracing) — a deliberate trade-off, not a free upgrade over
  synchronous calls. For example, debugging "why didn't the invoice get
  generated" now means tracing through a queue and multiple consumers
  instead of one linear call stack.
- Message queue reliability is only as good as the consumer's
  acknowledgment discipline and the broker's own durability guarantees —
  understanding exactly when a message is considered "safely delivered"
  by a given broker is essential before depending on it for critical
  data. For example, a broker configured to only persist to disk
  asynchronously can lose "acked" messages on a crash, which matters a lot
  for payment events.
