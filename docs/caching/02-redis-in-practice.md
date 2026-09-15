# Redis in Practice

## What

**Redis** is an in-memory data store commonly used as a cache, but also
supports rich data structures (strings, hashes, lists, sets, sorted sets)
and features (pub/sub, expiry, atomic operations) that make it useful
well beyond simple key-value caching.

## Why

An in-memory store is orders of magnitude faster than a disk-backed
database for the read-heavy, latency-sensitive access patterns caching
exists to serve. Redis specifically is the near-universal default choice
because of its speed, simplicity, and data structure support that fits
many caching and lightweight-state use cases without needing a
general-purpose database.

## How

### Basic key-value operations

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

r.set("user:123:name", "Sayan", ex=300)  # ex = TTL in seconds
r.get("user:123:name")  # "Sayan"
r.delete("user:123:name")
r.exists("user:123:name")  # 0
```

`ex=300` sets a TTL directly on the `SET` call — the standard way to
ensure every cache entry has a bounded lifetime (see
[Caching Strategies and TTL](01-caching-strategies-and-ttl.md)).

### Storing structured data

```python
import json

user = {"id": 123, "name": "Sayan", "email": "s@example.com"}
r.set(f"user:{user['id']}", json.dumps(user), ex=300)

cached = r.get(f"user:{user['id']}")
user = json.loads(cached) if cached else None
```

Redis strings are just bytes — JSON-serializing structured Python objects
before storing (and deserializing on read) is the standard pattern for
caching anything beyond a plain string/number.

### Hashes: caching an object's fields

```python
r.hset(f"user:{user_id}", mapping={"name": "Sayan", "email": "s@example.com"})
r.hget(f"user:{user_id}", "email")            # fetch one field
r.hgetall(f"user:{user_id}")                  # fetch all fields
r.expire(f"user:{user_id}", 300)              # TTL applies to the whole hash
```

A hash lets you cache an object's fields individually — useful when you
often need to read/update just one field (e.g. incrementing a view count)
without serializing/deserializing the entire object.

### Atomic increment (avoiding race conditions)

```python
r.incr("page:home:views")           # atomic, safe under concurrent access
r.incrby("user:123:credits", -10)   # atomic decrement
```

`INCR`/`INCRBY` are atomic at the Redis level — critical for counters
under concurrent access, where a naive "read, add one, write back" in
application code would race (two concurrent increments could both read
the same starting value and one update would be lost).

### Sets and sorted sets

```python
r.sadd("post:42:likes", "user:1", "user:2")     # unique membership
r.sismember("post:42:likes", "user:1")           # True

r.zadd("leaderboard", {"user:1": 150, "user:2": 320})  # sorted by score
r.zrevrange("leaderboard", 0, 9, withscores=True)       # top 10
```

Sorted sets (`ZADD`/`ZRANGE`) are a common way to implement a
leaderboard, a rate limiter's sliding window, or "most recent N items"
directly in Redis without extra application-level bookkeeping.

### Expiring keys and checking TTL

```python
r.ttl("user:123:name")     # seconds remaining, -1 if no TTL, -2 if key doesn't exist
r.persist("user:123:name") # remove the TTL, make the key permanent
```

Checking `TTL` explicitly is useful for debugging ("why did this cache
entry disappear") and for implementing cache-warming logic that refreshes
entries proactively before they expire.

### Distributed locks (`SET NX`)

```python
acquired = r.set("lock:process-order:42", "1", nx=True, ex=10)
if acquired:
    try:
        process_order(42)
    finally:
        r.delete("lock:process-order:42")
```

`SET key value NX EX ttl` atomically sets a key only if it doesn't
already exist, with an expiry as a safety net — the basis of a simple
distributed lock, ensuring only one process handles a given task at a
time even across multiple application instances. The TTL is essential:
without it, a crashed process holding the lock would block everyone else
forever.

### Pub/Sub

```python
# Publisher
r.publish("notifications", json.dumps({"event": "order_placed", "order_id": 42}))

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe("notifications")
for message in pubsub.listen():
    if message["type"] == "message":
        handle(json.loads(message["data"]))
```

Redis pub/sub is lightweight but **not durable** — a subscriber that's
offline when a message is published simply misses it (unlike Kafka's log
retention, see
[Kafka Fundamentals](../messaging/02-kafka-fundamentals.md)) — suitable
for ephemeral, best-effort notifications, not a substitute for a real
message queue where delivery guarantees matter.

### Connection pooling

```python
pool = redis.ConnectionPool(host="localhost", port=6379, max_connections=20)
r = redis.Redis(connection_pool=pool)
```

Same principle as database connection pooling (see
[Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md))
— create the pool once at application startup and reuse it, rather than
opening a new Redis connection per request.

## When to use

- `SET ... EX` for straightforward cache-aside caching with a bounded TTL.
- Atomic operations (`INCR`, `SET NX`) for counters and simple distributed
  locks where correctness under concurrent access matters.
- Sorted sets for ranking/leaderboard-style data structures that would
  otherwise require extra application-level sorting logic.
- Redis pub/sub only for ephemeral, best-effort notifications where
  missed messages during a subscriber outage are acceptable.

## When NOT to use

- Don't use Redis pub/sub where guaranteed delivery matters — use a real
  message queue (see [Messaging](../messaging/index.md)) instead.
- Don't implement a "read, modify, write" counter pattern in application
  code when `INCR`/`INCRBY` provides the same result atomically and
  race-free.
- Don't rely on a distributed lock built on `SET NX` without a TTL — a
  crashed lock holder would otherwise block all other processes
  indefinitely.
- Don't treat Redis as a durable primary data store by default — it's
  commonly configured for in-memory speed over durability (though
  persistence options like RDB/AOF exist and are configurable if truly
  needed).

## Common mistakes

- Storing structured data as a plain string without a consistent
  serialization format, leading to bugs deserializing older cached values
  after a schema change.
- Implementing counters with separate `GET` + application-side increment +
  `SET` instead of the atomic `INCR`, introducing race conditions under
  concurrent access.
- Building a distributed lock without a TTL, risking a permanently stuck
  lock if the holder crashes before releasing it.
- Creating a new Redis connection per request instead of reusing a
  connection pool, adding unnecessary connection overhead per operation.

## Interview questions

1. Why is `INCR` safe under concurrent access while a manual "get, add
   one, set" sequence in application code is not?
2. How would you implement a simple distributed lock with Redis, and why
   does the TTL matter?
3. What's the key durability difference between Redis pub/sub and a
   message queue like Kafka or RabbitMQ?
4. When would you choose a Redis hash over just storing a JSON-serialized
   string for an object?
5. Why should a Redis connection pool be created once at startup rather
   than per request?

## Senior-level considerations

- Choosing Redis data structures deliberately (hash vs. string, sorted
  set vs. list) based on actual access patterns (partial field updates,
  ranking, membership checks) avoids unnecessary application-level logic
  and takes advantage of Redis's atomic, purpose-built operations.
- Redis's default in-memory nature means a Redis outage or restart can
  mean losing cached data — application code should always be able to
  degrade gracefully (falling back to the source of truth) rather than
  assuming Redis as an always-available, always-durable store.
- Distributed locks built on Redis (`SET NX EX`) are simple but have known
  edge cases (clock drift, lock expiry racing with slow processing) —
  understanding these limits, and when a more rigorous distributed lock
  algorithm (e.g. Redlock) or a different coordination mechanism entirely
  is warranted, is a mark of real production experience with Redis-based
  locking.
