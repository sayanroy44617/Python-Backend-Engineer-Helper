# Interview Prep: System Design

## How to approach a system design interview

System design interviews are deliberately open-ended and evaluate your
*process* as much as your final answer — interviewers are watching
whether you clarify requirements, reason about trade-offs explicitly,
and can go deep on at least one component when pressed, not whether you
produce one single "correct" architecture. The mechanics referenced
below are covered in depth in
[System Design](../system-design/index.md); this page is the
interview-specific *framework* for structuring your 30-45 minutes.

## A repeatable framework

1. **Clarify functional requirements (2-5 minutes).** What must the
   system actually do? Ask about the core use cases explicitly — don't
   assume. For "design a URL shortener," confirm: custom aliases?
   analytics? expiration?
2. **Clarify non-functional requirements and scale (2-5 minutes).**
   Rough read/write volume, latency expectations, consistency
   requirements. A back-of-envelope estimate ("100M requests/day ≈
   ~1,200 req/s average, plan for peak well above that") grounds every
   later decision and signals quantitative reasoning.
3. **Sketch a high-level architecture (5-10 minutes).** Client → load
   balancer → stateless service layer → cache → database, adding queues
   or additional services only as the requirements actually demand them.
   Keep it simple first; add complexity when you can justify it.
4. **Deep-dive into 1-2 components** the interviewer steers you toward
   (usually the database schema/scaling, or a specific tricky
   interaction) — this is where most of the signal is generated, so
   don't rush past the high-level sketch to get here, but don't linger
   on the high-level sketch either.
5. **Explicitly discuss trade-offs**, using the vocabulary from
   [CAP Theorem, Consistency, and Availability](../system-design/04-cap-theorem-consistency-and-availability.md):
   which operations need strong consistency vs. which can tolerate
   eventual consistency, and why.
6. **Address failure modes and scaling** explicitly if time allows: what
   happens when a component fails, and how does the design scale further
   from here — see
   [Failure Handling and Rate Limiting](../system-design/05-failure-handling-and-rate-limiting.md).
7. **Summarize and state what you'd do differently with more
   time/information** — this closing signal (recognizing your own
   design's limitations) is a strong senior-level indicator.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | Why is a stateless service layer usually the first architectural decision in a system design answer? | It's the prerequisite for horizontal scaling — any request can be routed to any instance, which is what lets a load balancer add capacity by adding instances. | [Scalability, Statelessness, and Scaling Strategies](../system-design/01-scalability-statelessness-and-scaling-strategies.md) |
| 2 | How would you scale a read-heavy database without changing the application layer? | Add read replicas and route reads to them, keeping writes on the primary — cheaper and simpler than sharding, appropriate as long as write volume/data size don't themselves require partitioning. | [Databases and Queues at Scale](../system-design/03-databases-and-queues-at-scale.md) |
| 3 | When would you introduce a message queue into a design, and what does it actually buy you? | When a producer and consumer have different or unpredictable processing rates, or when the producer shouldn't block waiting on the consumer — it decouples their rates and smooths load spikes, at the cost of eventual consistency between produce and consume. | [Databases and Queues at Scale](../system-design/03-databases-and-queues-at-scale.md) |
| 4 | How would you decide between strong and eventual consistency for a specific feature within your design? | Ask what happens if a user sees stale data for that feature — a "like count" tolerates staleness fine; a bank balance or inventory count generally does not — and make that determination per feature, not for the whole system uniformly. | [CAP Theorem, Consistency, and Availability](../system-design/04-cap-theorem-consistency-and-availability.md) |
| 5 | How would you prevent one failing downstream dependency from taking down your whole system? | Explicit timeouts on every call, a circuit breaker to fail fast once a dependency is clearly unhealthy, and bulkhead isolation so one dependency's failure doesn't exhaust resources shared with calls to healthy dependencies. | [Failure Handling and Rate Limiting](../system-design/05-failure-handling-and-rate-limiting.md) |
| 6 | Where would you place caching in a design, and how would you justify it? | At the layer(s) closest to where load is highest relative to data change frequency — a CDN for static/cacheable responses, an application-level cache (Redis) in front of expensive database reads — justified by the read/write ratio of the specific data being cached. | [Load Balancing and Caching at Scale](../system-design/02-load-balancing-and-caching-at-scale.md) |
| 7 | How would you rate-limit a public API in your design, and why does it matter at the system level? | Per-client rate limiting protects the system from a single misbehaving or malicious client overwhelming shared capacity — this is a failure-handling technique aimed at load, not just an API-contract detail. | [Failure Handling and Rate Limiting](../system-design/05-failure-handling-and-rate-limiting.md) |
| 8 | When would sharding a database actually be necessary in your design? | Once a single primary (even with read replicas and caching) can't handle the write volume or data size — sharding is usually the last lever pulled, not the first, given its operational complexity. | [Databases and Queues at Scale](../system-design/03-databases-and-queues-at-scale.md) |
| 9 | How would you estimate whether your design needs to worry about a specific bottleneck at all? | Do a rough back-of-envelope calculation (requests/sec, data volume, expected growth) — many designs don't actually need sharding, multi-region, or a queue at the scale the interviewer described, and saying so explicitly (with the math) is a stronger answer than defaulting to maximal complexity. | [Scalability, Statelessness, and Scaling Strategies](../system-design/01-scalability-statelessness-and-scaling-strategies.md) |
| 10 | How do health checks and load balancing work together to contain a single instance's failure? | The load balancer only routes to instances passing health checks — an unhealthy instance is automatically removed from rotation, containing its failure to itself rather than surfacing it to users. | [Load Balancing and Caching at Scale](../system-design/02-load-balancing-and-caching-at-scale.md) |

## Common red flags interviewers watch for

- Jumping straight to a complex architecture (sharding, multiple
  regions, a dozen microservices) without first estimating whether the
  stated scale actually requires it.
- Never stating an explicit trade-off — presenting one design as if it
  were the only correct answer.
- Ignoring failure modes entirely until explicitly prompted.
- Confusing operational availability (uptime) with the CAP-theorem sense
  of availability when discussing consistency trade-offs.
- Not being able to go deep on any single component when pressed,
  suggesting shallow, memorized knowledge rather than real understanding.

## Related deep-dive material

- [System Design section overview](../system-design/index.md) — 5 topic
  pages covering the underlying mechanics this framework draws on.
- [Failure Handling and Rate Limiting](../system-design/05-failure-handling-and-rate-limiting.md)
  and
  [CAP Theorem, Consistency, and Availability](../system-design/04-cap-theorem-consistency-and-availability.md)
  are the two pages most commonly probed in a deep-dive.
