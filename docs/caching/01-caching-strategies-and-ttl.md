# Caching Strategies and TTL

## What

Caching stores a copy of expensive-to-compute or expensive-to-fetch data
somewhere faster to access than its source of truth. The main **caching
strategies** (cache-aside, read-through, write-through, write-behind)
differ in exactly when the cache is populated and how it stays in sync
with the underlying data store. **TTL** (time-to-live) and **eviction**
policies control how long cached data lives before it's removed.

## Why

A database or an external API call is almost always slower than an
in-memory or Redis lookup, and often more expensive (connection pool
pressure, rate limits, compute cost). Caching trades a small amount of
staleness risk for a large latency and load reduction — the right
strategy depends on how tolerant a given piece of data is of being
briefly out of date.

## How

### Cache-aside (lazy loading) — the most common pattern

```python
def get_user(user_id: int) -> User:
    cached = redis_client.get(f"user:{user_id}")
    if cached is not None:
        return User.model_validate_json(cached)

    user = db.get(User, user_id)  # cache miss -- read from the source of truth
    redis_client.set(f"user:{user_id}", user.model_dump_json(), ex=300)
    return user
```

The application checks the cache first; on a miss, it reads from the
database and populates the cache for next time. Simple and widely used —
the cache only ever contains data that's actually been requested, and a
cache outage just means every request falls back to the database (slower,
but not broken).

### Read-through

```python
# The cache layer itself owns the "miss -> load from source" logic,
# rather than the application code doing it explicitly
cache = ReadThroughCache(loader=lambda uid: db.get(User, uid), ttl=300)
user = cache.get(user_id)
```

Functionally similar to cache-aside, but the loading logic lives in the
caching layer/library itself rather than being written explicitly in
application code each time — a design/ownership distinction more than a
different guarantee.

### Write-through

```python
def update_user(user_id: int, **changes) -> User:
    user = db.get(User, user_id)
    for key, value in changes.items():
        setattr(user, key, value)
    db.commit()
    redis_client.set(f"user:{user_id}", user.model_dump_json(), ex=300)  # update cache immediately
    return user
```

Every write updates both the database and the cache together — reads are
always fresh (no stale cache window), at the cost of every write paying
the latency of updating both stores.

### Write-behind (write-back)

```python
def update_user(user_id: int, **changes) -> None:
    redis_client.set(f"user:{user_id}", json.dumps(changes), ex=300)
    queue.enqueue("persist_user_changes", user_id, changes)  # DB write happens later, async
```

The cache is updated immediately and the durable write is deferred
(often via a queue — see
[Message Queues and Messaging Patterns](../messaging/01-message-queues-and-messaging-patterns.md)).
Fastest write path, but risks losing the update if the system crashes
before the deferred write completes — appropriate only when some data
loss risk is acceptable in exchange for write latency.

### TTL (time-to-live)

```python
redis_client.set("session:abc123", session_data, ex=1800)  # expires in 30 minutes
```

A TTL bounds how long stale data can persist in the cache without any
explicit invalidation — even if you forget to invalidate a cache entry on
update, it self-heals within the TTL window. Choosing a TTL is a direct
trade-off between staleness tolerance and cache hit rate (a very short
TTL keeps data fresh but caches almost nothing usefully; a very long TTL
maximizes hit rate but risks serving stale data longer).

### Eviction policies

| Policy | Behavior |
|---|---|
| LRU (Least Recently Used) | Evicts the entry not accessed for the longest time |
| LFU (Least Frequently Used) | Evicts the entry accessed the fewest times |
| TTL-based | Evicts entries once their TTL expires, regardless of usage |
| Random | Evicts an arbitrary entry (rare, used in some simple caches) |

Eviction policies matter once the cache reaches its configured memory
limit — Redis, for example, supports several (`allkeys-lru`,
`volatile-ttl`, etc.) configurable per deployment; picking one that
matches actual access patterns (e.g. LRU for "recently viewed" data)
meaningfully affects hit rate under memory pressure.

### What belongs in a cache

```
Good fit:  data that's expensive to compute/fetch, read far more often
           than written, and tolerant of brief staleness (user profile
           data, product catalog, computed aggregates)

Poor fit:  data that must always be perfectly consistent (real-time
           account balance right before a transaction), or data that
           changes on nearly every read (defeats the purpose of caching)
```

## When to use

- Cache-aside as the default strategy — simplest to reason about, and
  degrades gracefully (cache outage → slower, not broken).
- Write-through when reads must never see stale data and the extra write
  latency is acceptable.
- Write-behind only when write throughput is critical and some risk of
  losing the most recent update on a crash is genuinely acceptable.
- A TTL on every cached entry, even ones you also invalidate explicitly —
  it's a safety net against invalidation bugs.

## When NOT to use

- Don't cache data that changes on nearly every access — the cache hit
  rate will be near zero and you've added complexity for no benefit.
- Don't use write-behind for data where losing the most recent update
  (on a crash before the deferred write persists) is unacceptable.
- Don't set an infinite TTL "to maximize hit rate" — it removes the
  self-healing safety net that bounds how stale data can become if
  invalidation is missed or buggy.

## Common mistakes

- No TTL at all, so a missed or buggy invalidation leaves stale data
  cached indefinitely.
- Caching data that's read once and never again, wasting cache memory
  without any hit-rate benefit.
- Choosing write-behind for data where durability matters more than write
  latency, without weighing the crash-loss risk explicitly.
- Ignoring eviction policy configuration and being surprised when
  frequently-needed entries get evicted under memory pressure because the
  default policy didn't match the actual access pattern.

## Interview questions

1. Walk through cache-aside end to end: what happens on a cache hit, a
   cache miss, and a cache outage?
2. Compare write-through and write-behind — what does each trade off?
3. Why does a TTL matter even on data you also invalidate explicitly on
   writes?
4. What's the difference between LRU and LFU eviction, and when would you
   prefer one over the other?
5. What characteristics make a piece of data a good vs. poor candidate
   for caching?

## Senior-level considerations

- Cache strategy choice should be driven by the actual read/write ratio
  and staleness tolerance of specific data, not applied uniformly across
  an entire system — different data in the same application often
  warrants different strategies.
- A cache should almost always be treated as an optimization layer the
  system can operate (more slowly) without — designing for graceful
  degradation on a cache outage (cache-aside's natural fallback to the
  source of truth) is safer than architectures where the cache becomes a
  hidden source of truth.
- TTL and eviction policy tuning is an ongoing operational concern, not a
  one-time setting — real access patterns shift over time, and cache hit
  rate/memory pressure should be monitored, not assumed static from
  initial configuration.
