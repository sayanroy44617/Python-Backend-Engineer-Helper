# CAP Theorem, Consistency, and Availability

## What

The **CAP theorem** states that a distributed system can provide at most
two of three guarantees simultaneously during a network partition:
**Consistency** (every read sees the most recent write), **Availability**
(every request gets a response), and **Partition tolerance** (the system
keeps working despite network failures between nodes).

## Why

Every distributed system design decision — read replicas, sharding,
caching, multi-region deployment — involves an implicit or explicit
choice about what happens when nodes can't communicate. CAP theorem
gives a precise vocabulary for reasoning about that choice, rather than
discovering the trade-off only when a real partition happens in
production.

## How

### The theorem, precisely

```
Partition tolerance is not optional for any real distributed system --
networks fail. The actual choice CAP theorem describes is:

  CP: during a partition, remain Consistent by refusing to answer
      some requests (sacrificing Availability)
  AP: during a partition, remain Available by answering with
      potentially stale/conflicting data (sacrificing Consistency)
```

"Choose 2 of 3" is a common but slightly misleading simplification —
partition tolerance isn't really optional (any system spanning more than
one node will experience partitions eventually); the real choice is
between CP and AP *specifically during* a partition.

### A concrete example: a replicated database during a network split

```
Primary (writes) --X-- Replica (reads)   -- network partition between them

CP choice: reject reads from the replica until it can confirm it's
           caught up with the primary (or reject writes to the primary
           if it can't confirm the replica received them)
AP choice: serve reads from the replica anyway, accepting they may be
           stale until the partition heals
```

This directly extends the read-replica discussion in
[Databases and Queues at Scale](03-databases-and-queues-at-scale.md) —
replication lag is a *routine*, non-partition version of this same
trade-off: even without a network partition, a replica read can be
stale simply because replication hasn't caught up yet.

### Strong consistency vs. eventual consistency

```
Strong consistency:   a read always reflects the most recent
                       acknowledged write -- typically requires
                       coordination (consensus, synchronous replication)
Eventual consistency: a read may return stale data temporarily, but all
                       replicas converge to the same value given enough
                       time with no new writes
```

Strong consistency is more intuitive to reason about but costs latency
and availability (coordination between nodes takes time and can fail).
Eventual consistency scales better and stays available during
partitions, but pushes the burden of handling temporary inconsistency
onto the application (or the user).

### Where this shows up in a typical backend stack

```
PostgreSQL (single primary, synchronous replicas): tends toward CP --
    a write isn't acknowledged until required replicas confirm it
Most caches (Redis used as a cache, CDNs):            tend toward AP --
    serve whatever's cached, refresh asynchronously
DNS:                                                   AP -- keeps
    resolving with possibly-stale records rather than failing outright
```

Recognizing which CAP trade-off a component your system depends on has
already made lets you reason correctly about what guarantees your own
system can actually provide — a system built on an AP cache cannot
itself promise strong consistency to its own users without additional
work.

### Availability as its own, distinct concern (SLAs and uptime)

```
99.9%  ("three nines")  ≈ 8.7 hours of downtime/year
99.99% ("four nines")   ≈ 52 minutes of downtime/year
```

Separately from the CAP-theorem sense, "availability" in an operational
context means uptime — this is what
[health checks, probes](../devops/kubernetes/04-probes-resources-and-scaling.md),
and redundant instances behind a
[load balancer](02-load-balancing-and-caching-at-scale.md) are built to
maximize: removing single points of failure so one instance's failure
doesn't take down the whole service.

### Consistency models beyond a binary strong/eventual split

```
Read-your-writes:  a user always sees their own writes immediately,
                    even if other users might see them slightly later
Causal consistency: writes that are causally related are seen in the
                    same order by everyone, unrelated writes may not be
```

Real systems often don't need full strong consistency everywhere, nor
can they tolerate fully unconstrained eventual consistency for every
operation — intermediate models like read-your-writes are frequently
the pragmatic middle ground (e.g. a user should always see their own
just-placed order, even if other users' view of aggregate order counts
can lag briefly).

### Designing around CAP rather than fighting it

```
Ask, per operation: "if this fails to reach consensus/replicate right
now, is it more important that this specific request succeeds
(possibly with stale data), or that it fails rather than risk
inconsistency?"
```

The trade-off isn't necessarily uniform across an entire system — a
"like" count can tolerate eventual consistency and staying available;
a balance transfer generally cannot. Making this choice deliberately,
per operation/data type, produces a more robust design than applying one
blanket consistency model everywhere.

## When to use

- CP behavior (favoring consistency over availability during a
  partition) for operations where stale or conflicting data causes real
  harm — financial transactions, inventory decrement, anything requiring
  a single source of truth at the moment of the operation.
- AP behavior (favoring availability, tolerating temporary staleness)
  for operations where staleness is acceptable and availability matters
  more — content delivery, social feeds, most read-heavy caching.
- Explicit per-operation reasoning about which trade-off a given piece
  of functionality actually needs, rather than one global choice.

## When NOT to use

- Don't assume a single consistency model applies uniformly across an
  entire system — different operations within the same system often
  have genuinely different requirements.
- Don't treat "eventual consistency" as an excuse to skip reasoning about
  how stale data actually affects users — the acceptable staleness
  window still needs to be bounded and understood.
- Don't ignore partition tolerance as "unlikely to matter" — any system
  spanning multiple nodes or availability zones will experience network
  partitions eventually.

## Common mistakes

- Assuming a system is strongly consistent by default without checking
  what consistency guarantees its actual components (cache, replicas,
  cross-region replication) provide.
- Applying one blanket consistency requirement to an entire system
  instead of reasoning about it per operation, over-engineering some
  paths and under-protecting others.
- Confusing CAP-theorem "availability" (does every request get a
  response) with operational "availability"/uptime — they're related
  but distinct concepts.
- Not accounting for replication lag as a routine (non-partition)
  source of the same staleness CAP theorem describes for partition
  scenarios.

## Interview questions

1. Why is "choose 2 of 3" in CAP theorem a slightly misleading
   simplification, and what's the more precise framing?
2. Give an example of a real backend component that leans CP, and one
   that leans AP.
3. What's the difference between strong consistency and eventual
   consistency, and what does each cost?
4. Why might different operations within the same system need different
   consistency guarantees?
5. How does replication lag relate to the CAP theorem trade-off, even
   without an actual network partition occurring?

## Senior-level considerations

- CAP theorem is a useful mental model but an oversimplification of
  real distributed systems — most production systems mix consistency
  models across different data/operations rather than picking one
  extreme for everything, and recognizing where that mixing is
  appropriate is a mark of mature system design.
- Choosing CP for an operation has a real, measurable availability
  cost (rejected requests during a partition) — this trade-off should be
  made deliberately and revisited as a system's actual failure patterns
  and business requirements become clearer, not decided once and
  forgotten.
- Consistency requirements interact directly with user experience design
  — sometimes the right fix for an eventual-consistency staleness issue
  isn't a stronger consistency guarantee at the data layer, but a UI/UX
  choice that makes the staleness window a non-issue for the user (e.g.
  optimistic UI updates).
