# Databases and Queues at Scale

## What

Scaling the components a stateless application layer depends on for
persistence and asynchronous work: read replicas and sharding for
**databases**, and **message queues** for decoupling and smoothing load
between services.

## Why

Making the application layer stateless and horizontally scalable (see
[Scalability, Statelessness, and Scaling Strategies](01-scalability-statelessness-and-scaling-strategies.md))
just moves the bottleneck downstream — to the database and to whatever
synchronous calls between services still exist. Scaling a system
end-to-end means addressing these too, not just the stateless compute
layer.

## How

### Read replicas

```
Primary (writes) ──replicates──▶ Replica 1 (reads)
                  ──replicates──▶ Replica 2 (reads)
```

Read-heavy workloads (far more common than write-heavy in most backend
systems) can route reads to replicas, leaving the primary free to handle
writes — this is a horizontal scaling technique applied specifically to
read capacity. The trade-off is replication lag: a replica may be
milliseconds (or more, under load) behind the primary, so a read
immediately following a write may not reflect it — see
[CAP Theorem, Consistency, and Availability](04-cap-theorem-consistency-and-availability.md)
for the consistency implications of this lag.

### Connection pooling under scale

```
N application instances × M connections each = total connections the
database must support -- this can exceed the database's connection
limit long before compute or query performance is actually the
bottleneck.
```

As the application layer scales horizontally, the total number of
database connections scales with it — this exact problem, and
`PgBouncer`/connection-pool sizing as the fix, is covered in depth in
[Locks, Deadlocks, and Connection Pooling](../databases/postgresql/03-locks-deadlocks-and-connection-pooling.md)
and
[Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md)
— worth revisiting specifically when reasoning about how many
application instances a given database can actually support.

### Sharding

```
Shard by customer_id hash:
  Shard 0: customers 0-999999
  Shard 1: customers 1000000-1999999
  ...
```

Once a single database instance (even with read replicas) can't handle
the write load or data volume, sharding partitions data across multiple
independent database instances — each shard is a complete, independent
database holding a subset of the data. This solves write scalability
(replicas only help reads) at the cost of real complexity: queries
spanning multiple shards (e.g. "all orders across all customers")
require querying every shard and aggregating results at the application
level, and resharding to rebalance data as it grows is a genuinely hard
operational problem.

### Caching in front of the database

```
Application -> Redis (cache hit? return) -> Database (cache miss only)
```

This is the same technique covered from the caching-layer perspective in
[Load Balancing and Caching at Scale](02-load-balancing-and-caching-at-scale.md)
— worth restating here because, for most systems, an effective cache is
a cheaper and simpler scaling lever than sharding, and should typically
be exhausted before reaching for sharding's added complexity.

### Message queues for decoupling and load smoothing

```
Producer -> Queue -> Consumer(s)
```

The mechanics of queues (delivery semantics, retries, dead-letter
queues) are covered in depth in
[Message Queues and Messaging Patterns](../messaging/01-message-queues-and-messaging-patterns.md)
and the rest of the Messaging section — from a system-design
perspective, a queue's value is decoupling a producer's request rate
from a consumer's processing rate: a traffic spike fills the queue
rather than overwhelming the consumer directly, letting the consumer
process at its own sustainable pace.

### Queues as a scaling technique specifically

```
Without a queue: producer's request rate directly determines the load
                 the consumer must handle in real time
With a queue:    consumer scales (more workers) independently of the
                 producer's request rate, processing a backlog at its
                 own pace
```

This decoupling is itself a scaling technique — it converts "the
consumer must scale in lockstep with the producer" into "the consumer
can scale independently, based on queue depth" — the same principle
behind autoscaling a worker pool based on queue length rather than
request rate directly.

### Choosing what to scale, in order of typical cost/benefit

```
1. Add a cache in front of the database  -- cheapest, highest leverage
2. Add read replicas for read-heavy load -- moderate complexity
3. Introduce a queue to decouple producer/consumer -- moderate complexity
4. Shard the database                    -- highest complexity, last resort
```

This isn't a rigid rule, but reflects a real pattern: teams that reach
for sharding before exhausting caching and read replicas often take on
sharding's operational complexity (cross-shard queries, resharding)
before it was actually necessary.

## When to use

- Read replicas for read-heavy workloads once a single database
  instance's read capacity becomes the bottleneck.
- Sharding only once caching, read replicas, and vertical scaling of the
  database itself are genuinely insufficient — it's a significant,
  hard-to-reverse architectural commitment.
- Message queues wherever a producer and consumer have different or
  unpredictable relative processing rates, or where a producer shouldn't
  block on a consumer's availability.

## When NOT to use

- Don't shard prematurely — the cross-shard query complexity and
  resharding operational burden are substantial, and most systems never
  need it if caching and read replicas are used well first.
- Don't add a queue between every pair of services by default — a
  queue introduces asynchronicity and eventual consistency where a
  simple, reliable synchronous call might be entirely sufficient and
  simpler to reason about.
- Don't scale the application layer horizontally without checking
  whether the database's connection limit can actually support the
  resulting number of connections.

## Common mistakes

- Scaling the application layer horizontally without noticing the
  database connection count scaling right along with it, until hitting
  the database's connection limit under load.
- Reaching for sharding before exhausting simpler options (caching, read
  replicas), taking on significant complexity earlier than necessary.
- Introducing a queue between services without considering the added
  operational complexity (monitoring queue depth, handling consumer
  failures, eventual consistency) it brings along with the decoupling
  benefit.
- Routing reads to a replica without accounting for replication lag,
  causing a user to not see their own very-recent write.

## Interview questions

- What problem do read replicas solve, and what problem do they *not*
    solve (i.e., what still requires sharding)?

    **Answer:** Read replicas increase read capacity by offloading SELECT
    traffic from the primary. They do not fix write bottlenecks, hot write
    paths, or a dataset that no longer fits well on one primary, which is
    where sharding may be needed.

- Why does horizontally scaling the application layer put pressure on
    the database's connection limit, and how is that typically mitigated?

    **Answer:** More app instances usually means more total DB
    connections, and databases hit connection limits long before they hit
    raw CPU limits in many real systems. Teams usually mitigate this with
    connection pooling, smaller pool sizes per instance, and tools like
    PgBouncer.

    ```python
    engine = create_engine(
       DB_URL,
       pool_size=20,
       max_overflow=10,
    )
    ```

- What are the trade-offs of sharding a database, beyond just "it's
    more complex"?

    **Answer:** You trade one big database for many smaller ones, which
    changes query patterns, operational tooling, and failure handling.
    Cross-shard joins get harder, resharding is painful, and a bad shard
    key can create hotspots that cancel out the expected scaling benefit.

- How does a message queue act as a scaling technique, not just a
    decoupling mechanism?

    **Answer:** It absorbs spikes so producers do not force consumers to
    process everything immediately in real time. That lets you scale
    workers based on backlog depth and smooth out bursty traffic instead
    of overprovisioning for every peak.

    ```text
    API -> enqueue job
    workers x 5 -> drain backlog
    queue depth up -> scale workers
    ```

- In what order would you typically reach for caching, read replicas,
    queues, and sharding when scaling a system, and why?

    **Answer:** Usually caching first, then read replicas for read-heavy
    DB pressure, then queues where producer and consumer rates differ, and
    sharding last because it is the hardest to undo. That order gives the
    best cost-to-complexity trade-off for many backend systems.

## Senior-level considerations

- Database scaling decisions (read replicas, sharding) are usually far
  more expensive to reverse than application-layer scaling decisions —
  a sharding scheme chosen too early, or with a poor shard key, can be
  extremely costly to change once significant data has accumulated — for
  example, sharding by region may look fine early on, then become a
  major migration problem when one region grows 10x faster than the
  others.
- The "cache and replicate before you shard" ordering is a strong
  default, but not universal — a genuinely write-heavy, high-volume
  workload (e.g. high-frequency event ingestion) may need sharding (or a
  purpose-built store) from early on, and recognizing that pattern
  correctly matters as much as the default ordering itself — for
  example, an analytics pipeline ingesting millions of events per minute
  may need partitioned storage long before a normal CRUD product would.
- Queues introduce eventual consistency and failure modes (message
  loss, duplicate processing, ordering) that a synchronous call doesn't
  have — introducing a queue is a real architectural trade-off, not a
  free decoupling win, and should be justified by the actual load-
  smoothing or resilience benefit it provides — for example, an email
  service can safely tolerate delayed retries, but an inventory-reserve
  workflow may need much tighter correctness guarantees before going
  async.
