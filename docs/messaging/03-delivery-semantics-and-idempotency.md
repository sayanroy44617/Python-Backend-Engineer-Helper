# Delivery Semantics and Idempotent Consumers

## What

**Delivery semantics** describe the guarantee a messaging system makes
about how many times a message will be delivered to a consumer:
**at-most-once**, **at-least-once**, or **exactly-once**. An
**idempotent consumer** is one designed so that processing the same
message more than once produces the same end result as processing it
once.

## Why

Network partitions, consumer crashes, and broker failovers are
unavoidable in a distributed system — the delivery guarantee determines
what happens when one occurs mid-delivery. Understanding which guarantee
your broker/configuration actually provides (and designing consumers
accordingly) is what prevents duplicate side effects (double-charging a
customer) or silent data loss (a missed notification) in production.

## How

### The three delivery semantics

```
At-most-once:   message delivered 0 or 1 times -- never redelivered, even
                if the consumer crashes before finishing. Risk: lost
                messages.

At-least-once:  message delivered 1 or more times -- redelivered if the
                consumer doesn't acknowledge in time. Risk: duplicate
                processing.

Exactly-once:   message delivered and processed exactly 1 time, no loss,
                no duplicates. Very hard to achieve end-to-end; usually
                approximated, not truly guaranteed, across arbitrary
                systems.
```

Most real systems run at-least-once delivery (safer default — no silent
loss) and handle the resulting possibility of duplicates by making
consumers idempotent, rather than chasing true exactly-once semantics
end-to-end.

### Why at-least-once is the common default

```python
# Ack BEFORE processing -> at-most-once (message lost if a crash happens
# between ack and completing work)
channel.basic_ack(delivery_tag)
process(message)

# Ack AFTER processing -> at-least-once (message redelivered if a crash
# happens between receiving and acking, so it might be processed twice)
process(message)
channel.basic_ack(delivery_tag)
```

Acking after processing is the safer default because losing a message
silently is almost always worse than occasionally processing one twice —
but it pushes the duplicate-handling responsibility onto the consumer.

### Making a consumer idempotent: unique operation IDs

```python
def process_payment(event: dict, db: Session) -> None:
    if db.get(ProcessedEvent, event["event_id"]) is not None:
        return  # already processed -- safe to skip

    charge_customer(event["customer_id"], event["amount"])
    db.add(ProcessedEvent(id=event["event_id"]))
    db.commit()
```

Recording that a specific `event_id` has already been processed (in the
same transaction as the side effect itself) is the standard pattern —
this directly mirrors the
[idempotency key pattern for HTTP APIs](../rest-api/02-rest-principles-and-idempotency.md),
applied to message consumers instead of API requests.

### Idempotent operations by design

```python
# Naturally idempotent: setting an absolute value
def set_account_balance(account_id: int, balance: float) -> None:
    account = get_account(account_id)
    account.balance = balance  # re-running with the same input is a no-op

# NOT naturally idempotent: applying a relative change
def add_to_balance(account_id: int, amount: float) -> None:
    account = get_account(account_id)
    account.balance += amount  # re-running DOUBLES the effect
```

Some operations are idempotent by their very nature (`PUT`-style
"set this value") without needing an explicit dedup mechanism; others
(`PATCH`-style relative changes) require the explicit idempotency-key
tracking pattern above.

### Exactly-once within a single system (Kafka transactions)

```python
producer = KafkaProducer(transactional_id="order-processor-1")
producer.init_transactions()

with producer.transaction():
    producer.send("processed-orders", value=result)
    consumer.commit()  # offset commit as part of the same transaction
```

Kafka's transactional API can provide exactly-once semantics *within
Kafka itself* (consume-process-produce as one atomic unit) — but the
moment a consumer's side effect reaches outside Kafka (a database write,
an external API call, an email), true exactly-once end-to-end is no
longer guaranteed by the broker alone; the consumer's own idempotency
handling still matters.

### Deduplication windows

```python
# A time-bounded or size-bounded set of recently seen IDs, rather than
# an unbounded table -- trades perfect dedup for bounded memory/storage
recent_ids = TTLCache(maxsize=100_000, ttl=3600)

def process(event):
    if event["event_id"] in recent_ids:
        return
    recent_ids[event["event_id"]] = True
    handle(event)
```

An unbounded "have I seen this ID before" table grows forever; a
bounded, time-windowed cache is a common practical trade-off when
duplicates are expected to arrive within a known time window (e.g.
broker redelivery after a short consumer restart) rather than
arbitrarily far apart.

## When to use

- Design every consumer to be idempotent by default when running
  at-least-once delivery (the common case) — treat "this message might
  arrive more than once" as a given, not an edge case.
- Naturally idempotent operations ("set to X") wherever the domain
  allows it, avoiding the need for explicit dedup tracking entirely.
- Explicit idempotency-key/processed-event tracking for relative or
  side-effecting operations (charging a payment, sending a notification)
  that aren't naturally idempotent.

## When NOT to use

- Don't rely on at-most-once delivery for anything where losing a message
  has real consequences (a payment event, an order) — the risk of silent
  loss is rarely worth the simplicity.
- Don't assume a broker's "exactly-once" feature (e.g. Kafka
  transactions) extends automatically to side effects outside that
  broker — external calls still need their own idempotency handling.
- Don't build unbounded deduplication storage when a bounded, time-
  windowed cache would suffice for the actual expected duplicate window.

## Common mistakes

- Acknowledging a message before processing completes, trading
  at-least-once safety for at-most-once risk without realizing it.
- Assuming "my broker supports exactly-once" means duplicate processing
  is impossible anywhere in the pipeline, including in downstream systems
  the broker doesn't control.
- Implementing relative-change operations (`balance += amount`) in a
  consumer without any dedup mechanism, causing real financial/data
  discrepancies on redelivery.
- Storing processed-event IDs without any expiry, causing unbounded
  storage growth over the system's lifetime.

## Interview questions

1. Why do most production systems choose at-least-once delivery with
   idempotent consumers over chasing true exactly-once semantics?
2. Walk through why acking a message before vs. after processing changes
   the delivery guarantee from at-least-once to at-most-once.
3. Give an example of an operation that's naturally idempotent and one
   that isn't, and explain how you'd make the non-idempotent one safe to
   retry.
4. What does Kafka's transactional API actually guarantee exactly-once
   for, and where does that guarantee stop applying?
5. Why might you use a time-bounded deduplication cache instead of an
   unbounded "processed IDs" table?

## Senior-level considerations

- Delivery semantics decisions ripple through the entire system design —
  choosing at-least-once with idempotent consumers is usually the
  pragmatic choice, but it requires *every* consumer along the pipeline
  to actually implement idempotency correctly, not just the first one.
- "Exactly-once" is one of the most commonly misused terms in messaging
  system marketing — a senior engineer should be able to precisely state
  what guarantee a given configuration actually provides and where its
  boundary is (e.g. within-broker vs. end-to-end across external systems).
- Idempotency-key storage and TTL design (how long to retain "already
  processed" records) is itself a trade-off between correctness
  confidence and storage/operational cost — driven by the real expected
  redelivery/retry window of the specific system, not a fixed rule of
  thumb.
