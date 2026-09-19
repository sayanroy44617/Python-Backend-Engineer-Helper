# Retry Strategies and Dead-Letter Queues

## What

A **retry strategy** defines how a failed message is retried (how many
times, with what delay) before giving up. A **dead-letter queue (DLQ)**
is where messages that exhaust their retries — or fail in a way judged
unrecoverable — are routed instead of being retried forever or silently
dropped.

## Why

Some failures are transient (a downstream service briefly unavailable)
and resolve themselves on retry; others are permanent (malformed
message, a bug in the consumer) and will never succeed no matter how many
times they're retried. Without a deliberate retry strategy and a DLQ, a
consumer either gives up too early on a transient failure (losing valid
work), or retries a permanently broken message forever, blocking the
queue behind it ("poison message").

## How

### Fixed-delay retry

```python
def process_with_retry(message, max_attempts=3, delay_seconds=2):
    for attempt in range(1, max_attempts + 1):
        try:
            handle(message)
            return
        except TransientError:
            if attempt == max_attempts:
                raise
            time.sleep(delay_seconds)
```

Simple, but a fixed delay retries a struggling downstream service at the
same rate repeatedly — if the failure is caused by that service being
overloaded, hammering it at a constant interval doesn't give it room to
recover.

### Exponential backoff (with jitter)

```python
import random

def process_with_backoff(message, max_attempts=5, base_delay=1.0):
    for attempt in range(1, max_attempts + 1):
        try:
            handle(message)
            return
        except TransientError:
            if attempt == max_attempts:
                raise
            delay = base_delay * (2 ** (attempt - 1))
            jitter = random.uniform(0, delay * 0.1)
            time.sleep(delay + jitter)
```

Exponential backoff spaces out retries increasingly (1s, 2s, 4s, 8s...),
giving a struggling downstream more time to recover. **Jitter** (a small
random addition) prevents the "thundering herd" problem — many failed
consumers all retrying at the exact same scheduled moment, which would
recreate the very overload condition that caused the failures.

### Distinguishing retryable from non-retryable failures

```python
def process(message):
    try:
        handle(message)
    except (ConnectionError, TimeoutError) as e:
        raise RetryableError(str(e)) from e   # transient -- retry
    except ValidationError as e:
        raise NonRetryableError(str(e)) from e  # permanent -- straight to DLQ
```

Not every failure deserves a retry — a malformed message will fail
identically on every attempt, wasting retry budget and delaying the
message's eventual (inevitable) trip to the DLQ. Classifying exceptions
as retryable vs. not is what prevents this.

### Dead-letter queue routing

```python
def process_with_dlq(message, max_attempts=3):
    attempt = get_delivery_attempt_count(message)
    try:
        handle(message)
    except NonRetryableError:
        publish_to_dlq(message, reason="validation_failed")
    except RetryableError:
        if attempt >= max_attempts:
            publish_to_dlq(message, reason="max_retries_exceeded")
        else:
            requeue_with_backoff(message, attempt)
```

The DLQ isn't a dumping ground to ignore — it's a queue meant to be
actively monitored, with alerting on its depth and a process for
inspecting, fixing, and potentially replaying its messages once the root
cause is addressed.

### Broker-native retry/DLQ support

```python
# RabbitMQ: dead-letter-exchange configured on the original queue
channel.queue_declare(
    queue="orders",
    arguments={
        "x-dead-letter-exchange": "orders.dlx",
        "x-message-ttl": 30000,  # requeue to DLX after 30s if unacked
    },
)
```

```
# Kafka: no built-in DLQ concept -- convention is a separate topic
# ("orders.DLQ") that the consumer explicitly publishes to on failure
```

Some brokers (RabbitMQ, SQS) have first-class DLQ support built into
queue configuration; Kafka has no native concept and relies on the
consumer explicitly producing failed messages to a separate topic
convention — know which model your broker uses before assuming DLQ
behavior is automatic.

### Poison message protection

```python
if get_delivery_attempt_count(message) > MAX_ATTEMPTS:
    publish_to_dlq(message, reason="poison_message")
    return
```

A "poison message" is one that will *always* fail regardless of retries
(a bug in deserialization, an unexpected schema) — without a hard cap on
attempts and DLQ routing, a poison message can block a queue/partition
indefinitely, since a broker's redelivery mechanism has no way to know
the failure is permanent rather than transient.

### Circuit breakers alongside retries

```python
if circuit_breaker.is_open():
    requeue_with_backoff(message, delay=circuit_breaker.reset_timeout)
    return

try:
    handle(message)
    circuit_breaker.record_success()
except TransientError:
    circuit_breaker.record_failure()
    raise
```

When a downstream dependency is *known* to be down (many consecutive
failures), a circuit breaker stops even attempting calls for a cooldown
period — combined with retry/backoff, this avoids wasting retry attempts
(and adding load to an already-struggling dependency) during a known
outage.

### Monitoring and alerting on the DLQ

```
Alert when: DLQ depth > threshold, or DLQ growth rate spikes
Dashboard:  DLQ message age, count by failure reason, consumer lag
```

A DLQ that's silently accumulating messages with no alerting is
equivalent to silently dropping them — the DLQ only provides value if
someone (or some automated process) is actually watching it and acting
on what accumulates there.

## When to use

- Exponential backoff with jitter as the default retry strategy for
  transient failures (network blips, temporary downstream unavailability).
- A DLQ for any consumer where message loss and infinite-retry blocking
  are both unacceptable outcomes — which is most production consumers.
- Explicit retryable/non-retryable exception classification so
  permanently-failing messages reach the DLQ quickly instead of
  exhausting retry attempts pointlessly.

## When NOT to use

- Don't retry non-retryable failures (validation errors, malformed
  messages) — route them to the DLQ immediately; retrying wastes time and
  delays visibility into a real bug.
- Don't retry indefinitely without a maximum attempt cap — this is
  exactly the poison-message scenario that blocks queue processing.
- Don't set up a DLQ without monitoring it — an unmonitored DLQ is
  functionally the same as dropping the messages, just with extra steps.

## Common mistakes

- Fixed-delay retries without jitter, causing synchronized retry storms
  from many consumers that failed at the same moment.
- Retrying every failure uniformly instead of distinguishing transient
  from permanent failures, both wasting retry budget and delaying DLQ
  routing for messages that will never succeed.
- No maximum retry cap, letting a poison message retry forever and block
  a queue or partition.
- Treating the DLQ as a black hole — no alerting, no process for
  triaging and potentially replaying its contents once a root cause is
  fixed.

## Interview questions

- Why is exponential backoff with jitter preferable to a fixed retry
    delay?

    **Answer:** A fixed delay means every failed consumer retries at
    exactly the same intervals, creating synchronized retry storms that
    hit the downstream service all at once. Exponential backoff spaces
    retries further apart over time, and jitter (random variation) spreads
    them out so they don't all land at the same instant.

- What's a "poison message," and how does a maximum retry count with DLQ
    routing protect against it?

    **Answer:** A poison message is one that will *never* process
    successfully no matter how many times you retry (e.g. malformed data
    that always throws). Without a retry cap, it gets retried forever,
    blocking the queue behind it. A max retry count routes it to a dead
    letter queue after N attempts, unblocking everything else.

- How would you distinguish a retryable failure from a non-retryable one
    in a message consumer, and why does that distinction matter?

    **Answer:** Retryable = transient (network timeout, temporary 503,
    deadlock) — likely to succeed if tried again. Non-retryable =
    permanent (malformed payload, business rule violation) — retrying
    changes nothing. Retrying a non-retryable failure just wastes time and
    delays sending it to the DLQ where it can actually be investigated.

- Compare how RabbitMQ's dead-letter-exchange and Kafka's DLQ convention
    differ structurally.

    **Answer:** RabbitMQ has built-in dead-lettering — the broker itself
    automatically routes rejected/expired messages to a configured
    dead-letter exchange. Kafka has no built-in DLQ concept — it's just a
    convention where your consumer code explicitly publishes the failed
    message to a separate "dead letter" topic itself.

- Why does a DLQ need active monitoring to actually provide value?

    **Answer:** A DLQ that nobody watches just becomes a silent graveyard
    of failed messages — the failures still happened, they're just hidden
    instead of crashing loudly. Monitoring/alerting on DLQ depth is what
    turns it from "data loss you don't notice" into "a signal someone
    investigates."

## Senior-level considerations

- Retry and DLQ strategy should be informed by the actual cost of
  duplicate processing vs. message loss for that specific message
  type — a payment event and a "user viewed page" analytics event
  warrant very different retry/DLQ rigor. For example, a payment consumer
  might retry aggressively with strict idempotency, while an analytics
  event consumer might just drop failures after one retry since losing a
  page-view doesn't matter much.
- Circuit breakers and retry/backoff are complementary, not redundant —
  retries handle isolated transient failures; circuit breakers handle
  sustained downstream outages, avoiding wasted retry load during a known
  incident. For example, if a downstream API is fully down, a circuit
  breaker stops even attempting calls for a cooldown period instead of
  each consumer independently retrying into a dead service.
- A mature messaging system treats the DLQ as an operational surface with
  its own runbook (how to triage, how to safely replay messages after a
  fix, how to avoid reprocessing side effects twice on replay) — not just
  a technical safety net that's configured once and forgotten. For
  example, a runbook step like "confirm the bug is fixed, then replay DLQ
  messages in batches of 100 while watching error rate" prevents a replay
  from re-triggering the same incident.
