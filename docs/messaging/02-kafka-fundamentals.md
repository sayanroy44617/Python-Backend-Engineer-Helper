# Kafka Fundamentals

## What

**Kafka** is a distributed, log-based messaging platform: producers
append messages to durable, ordered **topics**, and consumers read them
independently at their own pace by tracking their position (**offset**)
in the log. Unlike a traditional queue, a Kafka message isn't removed
once consumed — it stays in the log until it expires (or a retention
policy removes it).

## Why

Traditional message queues (like the one in
[Message Queues and Messaging Patterns](01-message-queues-and-messaging-patterns.md))
typically remove a message once it's consumed and acked. Kafka's
append-only log model instead supports replaying history (a new consumer
can start from the beginning), independent consumer groups reading the
same data for different purposes, and much higher sustained throughput —
the trade-offs that make Kafka the standard choice for event streaming
and log-based architectures at scale.

## How

### Topics and partitions

```
Topic "orders"
├── Partition 0: [msg1, msg2, msg5, msg8, ...]
├── Partition 1: [msg3, msg6, msg9, ...]
└── Partition 2: [msg4, msg7, msg10, ...]
```

A **topic** is a named stream of messages, split into **partitions** for
parallelism — each partition is an ordered, append-only log. Order is
only guaranteed *within* a partition, not across the whole topic; this is
the key trade-off that lets Kafka scale (more partitions = more
consumers can read in parallel).

### Choosing a partition key

```python
producer.send("orders", key=str(order.customer_id).encode(), value=order_bytes)
```

Kafka hashes the message key to deterministically pick a partition — all
messages with the same key always land on the same partition, which is
how you get ordering guarantees *for a given entity* (e.g. all events for
one customer processed in order) while still parallelizing across
different entities.

### Producing messages

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=["localhost:9092"],
    value_serializer=lambda v: json.dumps(v).encode(),
)

producer.send("orders", key=str(order.id).encode(), value={"id": order.id, "total": order.total})
producer.flush()  # ensure the message is actually sent before continuing
```

### Consuming messages

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "orders",
    bootstrap_servers=["localhost:9092"],
    group_id="inventory-service",
    auto_offset_reset="earliest",
    enable_auto_commit=False,
)

for message in consumer:
    process_order(json.loads(message.value))
    consumer.commit()  # manually commit only after successful processing
```

`enable_auto_commit=False` + a manual `commit()` after processing gives
the same "ack only after success" discipline covered in
[Message Queues and Messaging Patterns](01-message-queues-and-messaging-patterns.md)
— auto-committing before processing risks losing a message if the
consumer crashes mid-work.

### Consumer groups

```
Topic "orders" (3 partitions)

Consumer group "inventory-service" (2 consumer instances):
  Instance A -> partitions 0, 1
  Instance B -> partition 2

Consumer group "analytics-service" (1 consumer instance):
  Instance C -> partitions 0, 1, 2  (reads the SAME messages independently)
```

A **consumer group** is a set of consumer instances sharing the work of
one topic — each partition is assigned to exactly one instance *within*
a group (for load-balanced, parallel processing), but a *different*
consumer group reads the entire topic independently from its own offset.
This is what lets multiple services consume the same event stream for
different purposes without interfering with each other.

### Partition count and consumer parallelism

```
3 partitions, 2 consumer instances in a group -> one instance handles 2
  partitions, the other handles 1 (uneven, but both busy)
3 partitions, 5 consumer instances in a group -> only 3 instances get a
  partition; the other 2 sit idle
```

The partition count is the hard upper bound on how many consumer
instances in a single group can process a topic in parallel — a common
capacity-planning mistake is under-provisioning partitions and later
discovering you can't scale consumers further without a (nontrivial)
repartitioning operation.

### Offset management and replay

```python
consumer.seek(partition, offset=0)  # rewind and reprocess from the beginning
```

Because Kafka retains messages (based on a configured retention period,
not "until consumed"), a consumer can rewind and reprocess history — this
underlies use cases like rebuilding a derived data store, backfilling a
new consumer, or recovering from a bug in earlier processing logic by
reprocessing the affected time range.

### Retention vs traditional queue semantics

```
Traditional queue: message deleted once acknowledged
Kafka:             message retained for a configured period (or size
                    limit) regardless of consumption, independent of
                    whether every consumer group has read it yet
```

This is the fundamental architectural difference from a queue like
RabbitMQ/SQS — Kafka is a durable, replayable log first, a
messaging system second; a traditional queue is built around
"deliver once, then discard."

## When to use

- High-throughput event streaming where multiple independent consumers
  need to read the same data (analytics, audit, multiple downstream
  services).
- Use cases benefiting from replay (reprocessing history, backfilling a
  new consumer, rebuilding derived state from an event log).
- Scenarios needing ordering guarantees *per entity* (e.g. per customer,
  per account) rather than strictly global ordering — achieved via
  partition key selection.

## When NOT to use

- Don't use Kafka for simple task distribution where a lightweight queue
  (RabbitMQ, SQS, a task queue framework) is a better operational fit —
  Kafka's operational complexity (partition planning, retention tuning,
  broker cluster management) isn't justified for basic "process this job
  once" use cases.
- Don't expect strict ordering across an entire topic — only per-
  partition ordering is guaranteed; design your partition key around
  what actually needs to stay in order.
- Don't under-provision partitions expecting to easily scale consumer
  parallelism later — increasing partitions on an existing topic is
  possible but doesn't retroactively rebalance existing keyed data
  cleanly.

## Common mistakes

- Choosing a partition key that doesn't distribute evenly (e.g. a key
  with very few distinct values), creating "hot" partitions that bottleneck
  throughput regardless of overall partition count.
- Assuming messages are deleted after consumption the way they are in a
  traditional queue, leading to confusion about storage growth and
  retention configuration.
- Auto-committing offsets before processing completes, losing messages on
  a consumer crash mid-processing.
- Under-provisioning partitions relative to the eventual expected number
  of parallel consumer instances.

## Interview questions

1. What's the difference between a Kafka partition and a Kafka topic, and
   why does ordering only apply within a partition?
2. How does a Kafka consumer group provide both load balancing (within a
   group) and independent parallel consumption (across groups)?
3. Why does the partition count set a hard limit on consumer parallelism
   within one consumer group?
4. How does Kafka's retention-based storage model differ fundamentally
   from a traditional message queue's delete-on-consume model?
5. How would you achieve ordering guarantees for a specific entity (e.g.
   one customer's events) while still parallelizing across entities?

## Senior-level considerations

- Partition key selection is one of the most consequential early design
  decisions in a Kafka-based system — it determines both ordering
  guarantees and load distribution, and is expensive to change once a
  topic has significant data and downstream consumers depending on
  current partitioning.
- Kafka's replay capability enables architectural patterns like event
  sourcing and rebuilding derived read models from scratch — a
  significant advantage over traditional queues for systems that need to
  evolve their processing logic over time without losing historical data.
- Capacity planning for Kafka (partition count, retention period, broker
  count) needs to account for expected future consumer parallelism and
  data volume upfront, since some of these dimensions are difficult or
  operationally risky to change after a topic is in heavy production use.
