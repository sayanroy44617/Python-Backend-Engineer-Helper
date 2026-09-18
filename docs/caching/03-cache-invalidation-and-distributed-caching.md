# Cache Invalidation and Distributed Caching

## What

**Cache invalidation** is the problem of removing or updating cached data
when the underlying source of truth changes, so the cache doesn't keep
serving stale data. **Distributed caching** extends caching across
multiple application instances/nodes, introducing consistency and
coordination challenges a single-process cache doesn't have.

## Why

"There are only two hard things in Computer Science: cache invalidation
and naming things" — the joke persists because it's true: a cache that's
never invalidated correctly serves subtly wrong data, often intermittently
and hard to reproduce. Distributed caching adds another dimension: with
multiple app instances, "the cache" isn't one thing anymore, and keeping
them consistent requires deliberate design.

## How

### Invalidate-on-write

```python
def update_user(user_id: int, **changes) -> User:
    user = db.get(User, user_id)
    for key, value in changes.items():
        setattr(user, key, value)
    db.commit()
    redis_client.delete(f"user:{user_id}")  # invalidate -- next read repopulates
    return user
```

The simplest and most common invalidation strategy: on every write,
delete (rather than update) the corresponding cache entry. The next read
after a write pays one cache-miss cost, but you avoid the risk of
writing an inconsistent value directly into the cache.

### Why delete instead of update on write

```python
# RISKY: two concurrent writes can race and leave the cache with the
# WRONG value even though both database writes succeeded correctly
redis_client.set(f"user:{user_id}", new_serialized_value, ex=300)

# SAFER: just invalidate; the next read repopulates from the
# now-consistent database
redis_client.delete(f"user:{user_id}")
```

If two concurrent updates both try to write their own version directly
into the cache, whichever write happens to land last "wins" in the cache
— even if it wasn't the last write to the database — leaving the cache
permanently inconsistent with the source of truth until the next
invalidation. Deleting sidesteps this: the cache simply doesn't have a
value until the next read fetches the current, correct one.

### TTL as a safety net (defense in depth)

```python
redis_client.set(f"user:{user_id}", value, ex=300)  # even with explicit invalidation
```

Explicit invalidation logic can have bugs (a code path that updates data
without remembering to invalidate the cache) — a TTL bounds the maximum
staleness even when invalidation is missed, which is why relying on TTL
*alone* without invalidation, or invalidation *alone* without a TTL, is
each individually more fragile than combining both.

### Cache key design for easier invalidation

```python
# Hard to invalidate precisely: one big blob covering many entities
r.set("all_users", json.dumps(all_users), ex=300)

# Easy to invalidate precisely: one key per entity
r.set(f"user:{user_id}", json.dumps(user), ex=300)
```

Fine-grained cache keys (one per entity, or per entity + query
parameters) let you invalidate exactly what changed — coarse-grained keys
force invalidating (and re-fetching) far more than necessary on any
single write.

### Invalidating derived/aggregate data

```python
# A cached "top 10 products" list depends on ANY product's price/stock
# changing -- there's no single natural cache key to invalidate precisely
def update_product_price(product_id: int, new_price: float) -> None:
    db.execute(update(Product).where(Product.id == product_id).values(price=new_price))
    redis_client.delete(f"product:{product_id}")
    redis_client.delete("top_10_products")  # must also invalidate the derived view
```

Cached aggregates/derived views (leaderboards, "top N" lists, computed
summaries) don't have one obvious owning write to hook invalidation into
— every write that could affect the aggregate needs to know to invalidate
it too, which is easy to miss as a codebase grows; a short, deliberate TTL
is often the pragmatic fallback for this category of cached data.

### Cache stampede ("thundering herd")

```python
# PROBLEM: cache entry expires; 1000 concurrent requests all miss at once
# and all hit the database simultaneously to repopulate it

# MITIGATION: only one request repopulates; others wait briefly or serve
# slightly-stale data
lock_acquired = r.set(f"lock:user:{user_id}", "1", nx=True, ex=5)
if lock_acquired:
    user = db.get(User, user_id)
    r.set(f"user:{user_id}", user.model_dump_json(), ex=300)
else:
    time.sleep(0.05)
    return get_user(user_id)  # retry, likely a cache hit now
```

A popular cache entry expiring under high concurrent load can cause a
"stampede" of simultaneous cache-miss requests all hitting the database
at once, potentially overwhelming it — using a short-lived lock (see
[Redis in Practice](02-redis-in-practice.md)) so only one request
repopulates the cache while others wait avoids this.

### Jittered TTLs to avoid synchronized expiry

```python
ttl = 300 + random.randint(-30, 30)  # spread expiry across a 1-minute window
redis_client.set(key, value, ex=ttl)
```

If many cache entries are all set with the exact same TTL at the same
time (e.g. from a bulk cache-warming job), they all expire simultaneously,
producing a stampede-like spike — a small random jitter on the TTL
spreads expiry out over time, smoothing the resulting load. Similar in
spirit to the retry jitter covered in
[Retry Strategies and Dead-Letter Queues](../messaging/04-retry-strategies-and-dead-letter-queues.md).

### Distributed caching consistency

```
Single-process in-memory cache:  trivially consistent (one copy)
Shared Redis cache:              all app instances share one cache --
                                  naturally consistent, but a single
                                  dependency/bottleneck
Per-instance local cache
  ("L1" cache in front of Redis): fastest, but each instance can have a
                                  different (stale) view until its own
                                  TTL expires
```

A common pattern layers a small per-instance in-memory cache ("L1") in
front of a shared Redis cache ("L2") for extremely hot keys — faster than
a network round-trip to Redis, but reintroduces the multi-copy
consistency problem at a smaller scale, so L1 TTLs are typically kept very
short.

### Invalidating a distributed local cache

```python
# Publish an invalidation event; every instance's local cache subscribes
# and evicts the key on receipt
redis_client.publish("cache_invalidation", json.dumps({"key": f"user:{user_id}"}))
```

Since a per-instance local cache isn't automatically aware of writes
happening via other instances, invalidating it requires an explicit
broadcast mechanism (commonly Redis pub/sub, given its low latency for
this exact "best-effort notification" use case — see
[Redis in Practice](02-redis-in-practice.md)) so every instance evicts the
stale entry from its own local cache.

## When to use

- Invalidate-on-write (delete, not update) as the default invalidation
  strategy for entity-level caches.
- Fine-grained, per-entity cache keys so invalidation can be precise
  rather than clearing broad swaths of the cache unnecessarily.
- A short TTL (with jitter) for cached aggregates/derived views that don't
  have one clear owning write to hook invalidation into.
- A repopulation lock for high-traffic cache keys where a stampede on
  expiry is a real risk.

## When NOT to use

- Don't update a cache entry's value directly on every write when
  concurrent writes are possible — prefer deleting and letting the next
  read repopulate consistently.
- Don't rely purely on TTL for entities you write to frequently — combine
  it with explicit invalidation for acceptable freshness without waiting
  out a full TTL window on every change.
- Don't add a per-instance local ("L1") cache layer without an explicit
  invalidation broadcast mechanism — otherwise different instances can
  silently serve different, inconsistent data.

## Common mistakes

- Updating (rather than deleting) a cache entry on write, creating a race
  where a slower write's cached value overwrites a faster, more recent
  write's value.
- No invalidation path for cached aggregates/derived views, leaving them
  stale until their TTL expires regardless of how much underlying data
  actually changed.
- Setting identical TTLs across a bulk-loaded batch of cache entries,
  causing a synchronized mass-expiry stampede later.
- Adding a local in-process cache layer for performance without realizing
  it introduces per-instance staleness that a shared cache didn't have.

## Interview questions

1. Why is deleting a cache entry on write generally safer than updating it
   directly, under concurrent writes?

   **Answer:** Updating the cache directly on write risks a race: two
   concurrent writes can update the DB in one order but the cache in the
   other, leaving the cache holding the *older* value. Deleting the entry
   just forces the next read to fetch fresh from the DB — simpler and
   avoids that ordering race.

2. What is a cache stampede, and how would you prevent one for a
   high-traffic cache key?

   **Answer:** A stampede happens when a hot cache key expires and many
   concurrent requests all miss at once, all hammering the DB
   simultaneously to refill it. Prevent it with a lock/single-flight
   pattern (only one request recomputes, others wait for that result) or
   by refreshing the value proactively before it expires.

3. Why is a TTL still valuable even when you also invalidate explicitly on
   every relevant write?

   **Answer:** Explicit invalidation depends on every write path
   correctly triggering it — a missed code path, bug, or out-of-band DB
   change leaves stale data with no automatic expiry. A TTL guarantees a
   worst-case staleness window regardless of whether invalidation logic
   has a gap.

4. How would you invalidate a cached aggregate (e.g. "top 10 products")
   that doesn't correspond to a single entity's write?

   **Answer:** Since no single write "owns" that key, rely on a short TTL
   to naturally refresh it periodically, or explicitly invalidate it from
   any write path that could plausibly affect the ranking (e.g. any order
   placed).

5. What extra consistency challenge does a per-instance local cache layer
   introduce compared to a single shared Redis cache?

   **Answer:** With a shared cache, one invalidation clears the value
   everywhere. With per-instance local caches, invalidating on one
   instance doesn't touch the copies held in other instances' memory —
   you need a way to broadcast the invalidation to every instance (e.g. a
   pub/sub message).

## Senior-level considerations

- Cache invalidation strategy should be designed alongside the data model
  itself — deciding cache keys and what triggers their invalidation is
  part of the initial design, not an afterthought bolted on once staleness
  bugs start appearing in production. For example, deciding upfront that
  `product:{id}` gets invalidated by any `products` table write to that
  ID avoids "why is this stale" debugging sessions later.
- Distributed caching consistency is fundamentally a trade-off between
  latency and freshness — a senior engineer should be able to articulate,
  for any given cached value, exactly how stale it's allowed to become and
  why that's acceptable for its specific use case. For example, "this
  product price can be up to 30 seconds stale, and that's fine because
  checkout re-validates the real price anyway."
- Cache-related incidents (stampedes, stale-data bugs, invalidation gaps)
  are common enough in practice that observability into cache hit rate,
  staleness, and invalidation events is worth investing in early, rather
  than debugging cache behavior blind after an incident. For example,
  logging every cache invalidation event makes it possible to answer "was
  this key invalidated recently?" during an incident instead of guessing.
