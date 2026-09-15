# Load Balancing and Caching at Scale

## What

**Load balancing** distributes incoming requests across multiple service
instances. **Caching at scale** means placing caches at multiple layers
of a system (CDN, application-level, database) to absorb load before it
ever reaches the most expensive resource.

## Why

Horizontal scaling (see
[Scalability, Statelessness, and Scaling Strategies](01-scalability-statelessness-and-scaling-strategies.md))
only works if requests are actually distributed across instances — a
load balancer is the piece that makes that distribution happen.
Caching, independently, reduces how much load reaches a system's
scarcest resource (typically the database) in the first place — the two
techniques are complementary: load balancing spreads unavoidable load,
caching reduces how much load is unavoidable at all.

## How

### Load balancing algorithms

```
Round robin:        requests distributed evenly, in order, across
                     instances -- simple, works well when instances are
                     roughly equal capacity and requests are similar cost
Least connections:  routes to the instance with the fewest active
                     connections -- better when request cost varies
IP hash:             routes based on a hash of the client IP -- gives
                     the same client the same instance consistently
                     (useful for a stateful legacy service that hasn't
                     been made stateless yet)
```

Algorithm choice matters most when request cost or instance capacity is
uneven — round robin's assumption of "roughly equal cost" breaks down
for services with a mix of cheap reads and expensive writes/reports.

### Layer 4 vs. Layer 7 load balancing

```
Layer 4 (transport): routes based on IP/port only, doesn't inspect
                      HTTP content -- fast, protocol-agnostic
Layer 7 (application): routes based on HTTP content (path, host,
                        headers) -- what Kubernetes Ingress and most
                        API gateways use
```

This is the same distinction covered in
[Ingress and External Access](../devops/kubernetes/03-ingress-and-external-access.md)
— an L7 load balancer/Ingress can route `/users` and `/orders` to
different backend services, while an L4 load balancer only sees TCP
connections and can't make that distinction.

### Health checks feeding load balancer routing

```
Load balancer only routes to instances currently passing health checks
-- an unhealthy instance is automatically removed from rotation.
```

This is exactly the
[readinessProbe mechanism](../devops/kubernetes/04-probes-resources-and-scaling.md)
covered in Kubernetes — a load balancer that doesn't check instance
health will keep sending traffic to a broken instance, actively making
an outage worse instead of routing around it.

### Caching layers, from closest to the client to farthest

```
CDN / edge cache        -- static assets, sometimes full API responses,
                            served from a location near the client
Application-level cache -- Redis/Memcached, holding computed results or
                            frequently read database rows
Database query cache    -- the database's own internal caching
                            (buffer pool, prepared statement cache)
```

Each layer intercepts requests before they reach the next, more
expensive layer — a cache hit at the CDN never reaches the application
at all; a cache hit in Redis never reaches the database. The mechanics
of application-level caching (TTLs, invalidation strategies, Redis data
structures) are covered in depth in the
[Caching](../caching/index.md) section — this file focuses on *where*
caching fits among the other scaling techniques, not the caching
mechanics themselves.

### Caching read-heavy vs. write-heavy workloads

```
Read-heavy (e.g. a product catalog):  caching is highly effective --
    most requests can be served from cache, dramatically reducing load
    on the database
Write-heavy (e.g. an order-processing pipeline): caching is less
    effective and riskier -- cached data goes stale quickly, and
    invalidation overhead can outweigh the benefit
```

Deciding *what* to cache is a workload-shape decision — see
[Cache Invalidation and Distributed Caching](../caching/03-cache-invalidation-and-distributed-caching.md)
for the mechanics of keeping a cache correct once writes are involved at
all.

### Combining load balancing and caching: reducing load before it's distributed

```
Client -> CDN (cache hit? return immediately)
       -> Load balancer (cache miss traffic only)
       -> Service instances (behind the load balancer)
       -> Application cache (Redis) (cache hit? skip the database)
       -> Database (only what neither cache layer could serve)
```

The database — usually a system's least horizontally-scalable component
(see [Databases and Queues at Scale](03-databases-and-queues-at-scale.md))
— ends up seeing only the traffic that survives every caching layer
above it; this is why aggressive, correctly-invalidated caching is often
a bigger lever for overall system capacity than adding more service
instances.

### Sticky sessions: a load-balancing workaround for stateful services

```
sticky session: load balancer routes a given client to the same
                instance every time (via a cookie or IP hash)
```

Sticky sessions let a stateful service scale horizontally without first
becoming stateless — but this is a workaround, not a fix: it breaks the
moment that specific instance goes down (its in-memory state is gone),
and it can cause uneven load distribution if some clients are much more
active than others. Making the service properly stateless (see
[Scalability, Statelessness, and Scaling Strategies](01-scalability-statelessness-and-scaling-strategies.md))
removes the need for this workaround entirely.

## When to use

- Health-check-aware load balancing for any service running more than
  one instance, so unhealthy instances are automatically routed around.
- CDN/edge caching for static assets and any API responses that are
  genuinely cacheable across users.
- Application-level caching (Redis) for expensive, frequently-repeated
  reads with a read-heavy access pattern.

## When NOT to use

- Don't rely on sticky sessions as a long-term substitute for making a
  service properly stateless — it's a workaround with real failure
  modes, not equivalent to statelessness.
- Don't cache write-heavy or rapidly-changing data aggressively without
  a clear invalidation strategy — stale cached data can cause more harm
  than the latency it saves.
- Don't default to round-robin load balancing for workloads with highly
  uneven request cost — least-connections or a cost-aware strategy will
  distribute load more evenly.

## Common mistakes

- Load balancing without health checks, continuing to route traffic to
  an unhealthy instance and worsening an outage instead of mitigating it.
- Relying on sticky sessions indefinitely instead of addressing the
  underlying statefulness that necessitates them.
- Caching aggressively without a real invalidation strategy, serving
  stale data indefinitely or until a TTL happens to expire.
- Assuming a load balancer alone solves scaling, without also addressing
  the database — usually the actual bottleneck once compute is
  horizontally scaled.

## Interview questions

1. What's the difference between Layer 4 and Layer 7 load balancing, and
   when does the distinction matter?
2. How do health checks change a load balancer's routing behavior, and
   why does this matter during a partial outage?
3. Why are sticky sessions considered a workaround rather than a real
   fix for a stateful service?
4. Why is caching often a bigger lever for system capacity than adding
   more service instances?
5. How would you decide what to cache in a system with a mix of
   read-heavy and write-heavy workloads?

## Senior-level considerations

- Load balancing and caching are complementary, not interchangeable —
  a system needs both a way to distribute unavoidable load and a way to
  reduce how much load is unavoidable; over-indexing on one without the
  other leaves capacity on the table.
- Multi-layer caching (CDN, application, database) means a cache
  invalidation bug can manifest very differently depending on which
  layer is stale — debugging "why is the user seeing old data" requires
  reasoning about the whole cache hierarchy, not just one layer.
- Sticky sessions and similar workarounds are sometimes a pragmatic,
  short-term choice (e.g. migrating a legacy stateful service
  incrementally) — recognizing them explicitly as technical debt, with
  a plan to remove them, is better than treating them as a permanent
  architecture.
