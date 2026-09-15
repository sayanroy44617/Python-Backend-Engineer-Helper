# Performance

System-level performance engineering: how to reason about where time
actually goes in a backend service, and how to validate performance
under realistic load before it becomes a production incident.

Much of the deep, topic-specific material already lives elsewhere in
this handbook — this section synthesizes those topics into a
system-level performance mindset and adds what isn't covered yet (load
testing), rather than repeating existing content:

- **Profiling** (`cProfile`, `tracemalloc`, line profiling) —
  [Performance and Profiling](../python/14-performance-and-profiling.md)
- **CPU-bound vs I/O-bound diagnosis, async concurrency** —
  [Performance and Profiling](../python/14-performance-and-profiling.md) and
  [Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md)
- **N+1 queries** —
  [Queries, Relationships, and Loading](../databases/sqlalchemy/02-queries-relationships-and-loading.md)
  and [Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md)
- **Connection pools** —
  [Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md)
- **Caching** — the entire [Caching](../caching/index.md) section

## Topics

1. [Application Performance Fundamentals](01-application-performance-fundamentals.md)
2. [Database Performance in Practice](02-database-performance-in-practice.md)
3. [Load Testing](03-load-testing.md)
