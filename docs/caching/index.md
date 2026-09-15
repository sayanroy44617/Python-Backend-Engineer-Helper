# Caching

Application-level caching: strategies for keeping a fast, temporary copy
of data close to where it's needed, Redis as the standard tool for it,
and the invalidation/distributed-caching problems that make caching
harder than it first appears.

This section covers **application-level** caching (Redis, in-memory,
cache-aside). HTTP-level caching (`Cache-Control`, `ETag`, browser/CDN
caching) is a related but distinct mechanism, briefly noted in
[HTTP Fundamentals](../rest-api/01-http-fundamentals.md) — not duplicated
here.

## Topics

1. [Caching Strategies and TTL](01-caching-strategies-and-ttl.md)
2. [Redis in Practice](02-redis-in-practice.md)
3. [Cache Invalidation and Distributed Caching](03-cache-invalidation-and-distributed-caching.md)
