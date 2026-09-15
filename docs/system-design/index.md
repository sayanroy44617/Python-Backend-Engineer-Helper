# System Design

How individual backend services and their surrounding infrastructure
(covered in depth throughout this handbook) combine into systems that
scale, stay available, and degrade gracefully under real-world load and
failure. This section is deliberately synthesis-focused — most of the
underlying mechanisms (caching, queues, databases, rate limiting) are
already covered in depth elsewhere; here the focus is on the
system-level reasoning: trade-offs, failure modes, and how these pieces
fit together when designing a system end to end.

## Topics

1. [Scalability, Statelessness, and Scaling Strategies](01-scalability-statelessness-and-scaling-strategies.md)
2. [Load Balancing and Caching at Scale](02-load-balancing-and-caching-at-scale.md)
3. [Databases and Queues at Scale](03-databases-and-queues-at-scale.md)
4. [CAP Theorem, Consistency, and Availability](04-cap-theorem-consistency-and-availability.md)
5. [Failure Handling and Rate Limiting](05-failure-handling-and-rate-limiting.md)
