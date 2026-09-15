# Transactions and Connection Pools

## What

How SQLAlchemy's `Session` maps onto the database transaction model
covered in
[Transactions and ACID](../postgresql/02-transactions-and-acid.md), and
how the `Engine`'s connection pool manages the underlying database
connections a session borrows.

## Why

The session/transaction boundary determines exactly when changes become
visible to other transactions and when they'd be lost on an error. Getting
pool configuration wrong causes two very different production failures:
too few connections exhausts the pool under load; too many overwhelms the
database (see
[Locks, Deadlocks, and Connection Pooling](../postgresql/03-locks-deadlocks-and-connection-pooling.md)).

## How

### The session as a transaction boundary

```python
with Session(engine) as session:
    with session.begin():  # starts a transaction
        user = User(name="Sayan", email="sayan@example.com")
        session.add(user)
        # commits automatically on successful exit,
        # rolls back automatically if an exception propagates
```

`session.begin()` as a context manager is the idiomatic 2.x pattern —
it ties the transaction's lifetime to the `with` block, guaranteeing
commit-or-rollback without manually calling `.commit()`/`.rollback()`
(the same context-manager discipline as
[Context Managers and Descriptors](../../python/10-context-managers-and-descriptors.md)).

### Manual transaction control

```python
session = Session(engine)
try:
    session.add(user)
    session.commit()
except Exception:
    session.rollback()
    raise
finally:
    session.close()
```

Equivalent to the context-manager form, but explicit — useful to
understand what the context manager is doing under the hood, though the
`with session.begin():` form is preferred in new code.

### Multiple operations in one transaction

```python
with Session(engine) as session:
    with session.begin():
        payer.balance -= 100
        payee.balance += 100
        # both updates commit together, or neither does
```

This directly mirrors the raw-SQL transaction example in
[Transactions and ACID](../postgresql/02-transactions-and-acid.md) — the
session just tracks the pending object changes and flushes them as SQL
`UPDATE` statements when needed.

### Nested transactions (savepoints)

```python
with session.begin():
    session.add(user)
    try:
        with session.begin_nested():  # SAVEPOINT
            session.add(risky_record)
            raise ValueError("something went wrong")
    except ValueError:
        pass  # only the nested savepoint rolls back; `user` is still staged
    session.commit()
```

`begin_nested()` uses a SQL `SAVEPOINT`, letting you roll back part of a
transaction without losing the rest — useful for "try this, and if it
fails, continue without it" logic within a larger unit of work.

### Engine and connection pool configuration

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pass@host/db",
    pool_size=10,       # baseline number of persistent connections
    max_overflow=5,     # extra connections allowed temporarily under load
    pool_timeout=30,    # seconds to wait for a connection before erroring
    pool_recycle=1800,  # recycle connections older than this (avoids stale connections)
)
```

The `Engine` owns the connection pool; a `Session` borrows a connection
from it only when it actually needs to execute SQL, and returns it to the
pool when the transaction ends — sessions themselves are cheap to create,
but they depend on the pool having available connections.

### One engine, many sessions

```python
# Created ONCE at application startup (see FastAPI lifespan)
engine = create_engine(DATABASE_URL, pool_size=10)
SessionLocal = sessionmaker(bind=engine)

# Created PER REQUEST
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

This is the same principle from
[Async Endpoints, Background Tasks, and Lifespan](../../fastapi/06-async-endpoints-and-background-tasks.md):
one long-lived engine/pool created at startup, many short-lived sessions
created and closed per request, each borrowing a connection from the
shared pool as needed.

## When to use

- `with session.begin():` for every logical unit of work — it guarantees
  commit-or-rollback semantics without manual bookkeeping.
- `begin_nested()` (savepoints) when part of a larger transaction should
  be allowed to fail and be discarded without aborting the whole unit of
  work.
- Configure `pool_size`/`max_overflow` based on expected concurrent
  request volume and the database's actual connection limit — not
  arbitrary defaults.

## When NOT to use

- Don't create a new `Engine` per request — it recreates the connection
  pool every time, defeating its purpose entirely and quickly exhausting
  database connections under load.
- Don't leave a session's transaction open across slow, unrelated
  operations (external API calls, waiting on user input) — this holds
  database locks and pool connections longer than necessary.
- Don't set `pool_size` without considering the total across all
  application instances — see
  [Locks, Deadlocks, and Connection Pooling](../postgresql/03-locks-deadlocks-and-connection-pooling.md)
  for the capacity math.

## Common mistakes

- Creating an `Engine`/`sessionmaker` inside a request handler instead of
  once at application startup.
- Forgetting that `session.add()` alone doesn't commit — changes are only
  durable after `commit()` (or lost on `rollback()`/an unhandled
  exception without a `with session.begin():` wrapper).
- Not using `begin_nested()` when partial failure recovery is genuinely
  needed, instead wrapping large chunks of logic in fragile manual
  try/except/rollback blocks.
- Undersizing the connection pool for actual concurrent load, causing
  `pool_timeout` errors under traffic spikes; or oversizing it across many
  instances, exhausting the database's `max_connections`.

## Interview questions

1. What does `with session.begin():` guarantee compared to manually
   calling `commit()`/`rollback()`?
2. What is a savepoint (`begin_nested()`), and when would you use one?
3. Why should the `Engine` (and its connection pool) be created once at
   application startup rather than per request?
4. What's the relationship between a `Session` and a connection from the
   pool — when does a session actually borrow one?
5. What happens if `pool_size` + `max_overflow` across all your
   application instances exceeds the database's `max_connections`?

## Senior-level considerations

- Pool sizing is a capacity planning exercise spanning the whole
  deployment (all instances × pool size vs. the database's connection
  limit), not a per-service tuning knob considered in isolation — this
  connects directly to
  [Locks, Deadlocks, and Connection Pooling](../postgresql/03-locks-deadlocks-and-connection-pooling.md).
- Keeping the transactional scope of a session as narrow as possible (no
  slow I/O inside `session.begin():`) is both a performance and a
  correctness concern — it minimizes lock hold time and reduces the
  chance of holding a connection open when the pool is under pressure.
- Understanding exactly when SQL is actually sent (flush) versus staged in
  memory is essential for reasoning about performance and correctness in
  code that batches many changes before a single commit — a common
  interview probe for real SQLAlchemy experience versus surface-level
  familiarity.
