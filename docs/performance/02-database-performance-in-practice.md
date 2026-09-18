# Database Performance in Practice

## What

A practical checklist for diagnosing and fixing database-related
performance problems in a backend service — pulling together indexing,
query patterns, connection pooling, and caching (each covered in depth
elsewhere) into the order you'd actually investigate them.

## Why

The database is the most common source of backend latency in practice —
but "the database is slow" isn't a diagnosis. Working through a
consistent checklist (query plan, N+1 pattern, indexing, pool
configuration) turns a vague complaint into a specific, fixable root
cause.

## How

### Step 1: is it actually one slow query, or many queries?

```python
# Enable SQL logging temporarily to see what's actually being executed
engine = create_engine(DATABASE_URL, echo=True)
```

The very first question: does the request issue one query that's
genuinely slow, or does it issue *far more queries than expected* (the
N+1 pattern)? These require different fixes — see
[Queries, Relationships, and Loading](../databases/sqlalchemy/02-queries-relationships-and-loading.md)
for exactly how N+1 arises from lazy-loaded relationships and how
`selectinload`/`joinedload` fix it.

### Step 2: for a genuinely slow single query, read the query plan

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

A sequential scan on a large, frequently-filtered table almost always
means a missing index — see
[Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md)
for how to read a query plan and decide what to index; adding the right
index is usually the single highest-leverage database performance fix
available.

### Step 3: check for lock contention, not just query speed

```sql
SELECT * FROM pg_stat_activity WHERE state = 'active';
```

A query that's fast in isolation can appear slow in production because
it's *waiting* on a lock held by another transaction, not because the
query itself is inefficient — see
[Locks, Deadlocks, and Connection Pooling](../databases/postgresql/03-locks-deadlocks-and-connection-pooling.md)
for diagnosing this distinct failure mode.

### Step 4: check connection pool health

```python
# Are requests timing out waiting for a pool connection, or is the
# actual query execution the slow part?
engine = create_engine(DATABASE_URL, pool_size=10, max_overflow=5, pool_timeout=30)
```

A `pool_timeout` error means requests are queuing for a database
connection, not that the database itself is slow — this points to
undersized pooling (or too many application instances relative to the
database's connection limit) rather than a query problem; see
[Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md).

### Step 5: is this data a good candidate for caching?

```python
# If the same expensive read happens repeatedly with the same result,
# caching may eliminate the database round-trip entirely for most requests
cached = redis_client.get(f"product:{product_id}")
```

Once query-level issues are ruled out or fixed, ask whether the read even
needs to hit the database on every request — see the
[Caching](../caching/index.md) section, especially
[Caching Strategies and TTL](../caching/01-caching-strategies-and-ttl.md),
for when caching is (and isn't) the right lever.

### Batching to reduce round-trips

```python
# One query per ID -- N round-trips
for order_id in order_ids:
    order = session.get(Order, order_id)

# One query for all IDs -- 1 round-trip
orders = session.execute(select(Order).where(Order.id.in_(order_ids))).scalars().all()
```

Beyond the ORM-relationship N+1 pattern, the same round-trip-reduction
principle applies to any loop issuing one query per item — batching into
a single `IN (...)` query (or a bulk insert/update) is a broadly
applicable fix whenever a request pattern issues many small queries that
could be one larger one.

### Read replicas for read-heavy workloads

```python
# Route read-only queries to a replica, writes to the primary
read_engine = create_engine(READ_REPLICA_URL)
write_engine = create_engine(PRIMARY_URL)
```

For read-heavy systems where caching alone isn't sufficient (data that
must always be fresh, or too varied to cache effectively), routing reads
to one or more read replicas offloads load from the primary — at the
cost of replication lag, meaning a replica read might briefly reflect
slightly stale data compared to a write that just happened on the
primary.

### Putting it together: a diagnostic order

```
1. Log/inspect actual queries issued -- is it N+1?
2. EXPLAIN ANALYZE the slow query -- missing index?
3. Check pg_stat_activity -- lock contention?
4. Check pool metrics -- connection exhaustion, not query speed?
5. Consider caching -- does this read need to hit the DB every time?
6. Consider read replicas -- read-heavy workload beyond what caching covers?
```

## When to use

- This diagnostic order whenever "the database is slow" is reported,
  before jumping straight to any one fix.
- `EXPLAIN ANALYZE` as the standard first step for any specific slow
  query, before guessing at what index might help.
- Batching (single `IN (...)` query, bulk operations) whenever a code
  path issues one query per item in a loop.

## When NOT to use

- Don't add an index reactively without checking the query plan first —
  indexing the wrong column doesn't help and still carries the write-side
  cost of every index (see
  [Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md)).
- Don't assume a slow-looking query is actually the query's fault without
  ruling out lock contention or connection pool exhaustion first — the
  fix is completely different in each case.
- Don't reach for a read replica before exhausting caching and query-level
  fixes — replicas add real operational complexity (replication lag,
  routing logic) that's often unnecessary if the underlying query
  patterns are fixed first.

## Common mistakes

- Diagnosing "slow database" purely by query text/logic review without
  ever running `EXPLAIN ANALYZE` to see what the database is actually
  doing.
- Confusing connection pool exhaustion (queuing for a connection) with
  slow query execution — they look similar (elevated request latency) but
  have entirely different fixes.
- Fixing an N+1 pattern with caching instead of eager loading — caching
  can mask the symptom for a while, but the underlying query count problem
  remains and resurfaces for any uncached case.
- Reaching for a read replica or more hardware before checking for an
  obvious missing index or N+1 pattern, which is almost always cheaper to
  fix and higher-leverage.

## Interview questions

1. What's your diagnostic order when told "the database is slow" for a
   specific endpoint?

   **Answer:** Start broad and narrow down: check if it's actually the DB
   (vs. waiting on a connection pool slot), look at slow query logs /
   `EXPLAIN ANALYZE` for the specific query, check for lock contention,
   then consider caching or a read replica if the query itself is already
   optimal.

2. How do you distinguish a genuinely slow query from a request that's
   just waiting for a connection pool slot?

   **Answer:** Check the query's own execution time via `EXPLAIN ANALYZE`
   or query logs — if that's fast but the request as a whole is slow,
   time is being spent waiting to *acquire* a connection from the pool,
   not executing SQL. Pool wait-time metrics (if instrumented) confirm
   this directly.

3. Why might a query that runs fast in isolation be slow under real
   production load? What would you check?

   **Answer:** In isolation there's no contention — under load, the same
   query can be blocked by locks from other transactions, competing for
   CPU/disk I/O with everything else, or waiting for a pool connection.
   Check lock waits, concurrent transaction volume, and pool saturation,
   not just the query plan alone.

4. When would caching be the right fix for a database performance
   problem, and when would it just be masking an underlying N+1 or
   missing-index issue?

   **Answer:** Caching is right when the underlying query is already
   efficient but simply run too often for the same, rarely-changing data.
   It's masking a real problem if the query itself is inefficient (N+1,
   missing index) — caching just hides the slowness for cached requests
   while the first request (and any cache miss) still pays the full,
   unfixed cost.

5. What's the trade-off introduced by adding a read replica for a
   read-heavy workload?

   **Answer:** You gain read throughput by spreading reads across
   replicas, but you introduce replication lag — a replica can serve
   slightly stale data, so anything requiring strict read-your-writes
   consistency needs to explicitly read from the primary instead.

## Senior-level considerations

- A consistent, repeatable diagnostic process (query logging → query
  plan → lock contention → pool health → caching → replicas) is more
  valuable than knowing any single fix — it prevents jumping to the wrong
  solution under production incident pressure. For example, adding an
  index under pressure without checking `EXPLAIN ANALYZE` first can add
  write overhead without fixing the actual slow query.
- Database performance problems compound with scale — an N+1 pattern that
  was tolerable at low traffic can become a serious incident once traffic
  grows, making it worth fixing proactively (caught via profiling/query
  logging in development) rather than reactively under load. For example,
  an N+1 that added 50ms at 10 requests/sec can turn into a full outage
  once traffic hits 500 requests/sec and saturates the connection pool.
- Read replicas, caching, and query/index optimization address different
  scaling dimensions (read throughput, repeated-read latency, individual
  query efficiency respectively) — a mature system typically applies all
  three deliberately rather than over-relying on just one. For example,
  a read replica won't help if the query itself is still doing a full
  table scan on every replica too — indexing has to happen regardless.
