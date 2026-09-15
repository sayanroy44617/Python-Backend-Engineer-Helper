# Locks, Deadlocks, and Connection Pooling

## What

**Locks** prevent concurrent transactions from conflicting on the same
data. A **deadlock** occurs when two transactions each hold a lock the
other needs, so neither can proceed. **Connection pooling** reuses a fixed
set of database connections across many application requests instead of
opening a new one per request.

## Why

Concurrent writes to the same rows are inevitable in any real backend
service — understanding locking explains both correctness (why a
transaction sometimes waits) and failure modes (deadlocks, which the
database must detect and resolve by force). Connection pooling matters
because opening a new PostgreSQL connection is relatively expensive
(process/memory overhead per connection) — without pooling, a
moderately-loaded service can exhaust the database's connection limit.

## How

### Row-level locks

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- locks this row
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- releases the lock
```

`FOR UPDATE` explicitly locks selected rows against concurrent
modification until the transaction ends — a second transaction attempting
the same `SELECT ... FOR UPDATE` on row `id = 1` blocks until the first
commits or rolls back. This is how you safely implement "read, compute,
write back" patterns without a race condition.

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` skips already-locked rows instead of waiting — a common
pattern for implementing a job queue where multiple workers pull rows
without blocking on each other.

### How a deadlock happens

```
Transaction A: locks row 1, then tries to lock row 2
Transaction B: locks row 2, then tries to lock row 1
-- neither can proceed: each waits on a lock the other holds
```

```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- waits on B's lock on row 2

-- Transaction B (concurrently)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- waits on A's lock on row 1
```

PostgreSQL detects this cycle and forcibly aborts one transaction with a
`deadlock detected` error, letting the other proceed — the aborted
transaction's application code must catch this and retry.

### Avoiding deadlocks: consistent lock ordering

```sql
-- Always lock accounts in a consistent order (e.g. by id ascending),
-- regardless of which "direction" the transfer is logically going
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

The most reliable deadlock prevention strategy: always acquire locks on
multiple rows in the same, consistent order across every code path that
might lock them together — eliminating the circular-wait condition
entirely.

### Table-level locks

```sql
LOCK TABLE orders IN SHARE MODE;
```

Table-level locks are coarser and rarer in application code — mostly
relevant during schema migrations (`ALTER TABLE` can briefly lock a whole
table) or explicit maintenance operations, not everyday query logic.

### Connection pooling

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql+psycopg://user:pass@host/db",
    pool_size=10,
    max_overflow=5,
    pool_timeout=30,
)
```

A connection pool maintains a fixed number of open connections
(`pool_size`), optionally allowing temporary extra connections under load
(`max_overflow`), and hands them out to requests as needed instead of
opening a fresh connection each time — created once at application
startup (see
[Async Endpoints, Background Tasks, and Lifespan](../../fastapi/06-async-endpoints-and-background-tasks.md)
for why this must happen in `lifespan`, not per-request).

### External connection poolers (PgBouncer)

For high-concurrency deployments (many application instances, each with
their own pool), a dedicated external pooler like **PgBouncer** sits
between the application and PostgreSQL, multiplexing many client
connections onto fewer actual database connections — necessary because
PostgreSQL's per-connection cost makes thousands of direct application-
level connections impractical at scale.

## When to use

- `SELECT ... FOR UPDATE` when a transaction needs to read a row, make a
  decision based on it, and write it back without a concurrent transaction
  changing it in between.
- Consistent lock ordering (e.g. always by primary key ascending) whenever
  a transaction locks more than one row that another transaction might
  also lock — the standard deadlock prevention technique.
- Connection pooling for every production service — a pool sized and
  configured once at startup, not a new connection per request.
- `SKIP LOCKED` for queue-like workloads where multiple workers pull tasks
  without blocking each other on the same rows.

## When NOT to use

- Don't hold row locks longer than necessary — a transaction with `FOR
  UPDATE` open while doing slow, unrelated work (external API calls, user
  interaction) blocks other transactions needlessly.
- Don't set connection pool size arbitrarily high "to be safe" — each
  connection consumes real database-side memory; an oversized pool across
  many application instances can itself overwhelm the database's
  connection limit.
- Don't rely on the database to sort out inconsistent lock ordering across
  your codebase — deadlock avoidance through consistent ordering is an
  application-level discipline, not something the database enforces for
  you automatically.

## Common mistakes

- Locking multiple rows in different orders across different code paths
  (e.g. one function locks accounts in payer-then-payee order, another in
  reverse), creating the exact conditions for a deadlock under concurrent
  execution.
- Not catching and retrying `deadlock detected` errors in application
  code — the database resolves deadlocks by aborting one transaction; the
  application must handle that gracefully.
- Creating a new database connection (or engine) per request instead of
  reusing a pool, exhausting the database's connection limit under load.
- Sizing the connection pool without considering the number of concurrent
  application instances — total connections across all instances, not
  just one pool's size, is what matters against the database's actual
  limit.

## Interview questions

1. What is a deadlock, and how does PostgreSQL resolve one when it
   detects it?
2. What's the standard technique for preventing deadlocks when multiple
   rows need to be locked together?
3. What does `SELECT ... FOR UPDATE` do, and what problem does it solve
   that a plain `SELECT` followed by `UPDATE` doesn't?
4. Why is connection pooling necessary, and what goes wrong without it?
5. What's the purpose of an external pooler like PgBouncer, given that
   SQLAlchemy already has its own connection pool?

## Senior-level considerations

- Deadlock frequency and lock contention are operational signals worth
  monitoring in production — a rising rate of `deadlock detected` errors
  or long lock wait times often indicates a code path that needs
  consistent lock ordering or shorter transactions, not just a one-off bug.
- Connection pool sizing is a capacity planning problem across the whole
  system: total connections = (application instances) × (pool size per
  instance), which must stay under the database's `max_connections` — this
  is why external poolers (PgBouncer) become necessary once an application
  scales horizontally to many instances.
- Transaction scope and lock duration directly affect how well a service
  scales horizontally — designing for short, narrowly-scoped transactions
  is as much a scalability decision as an indexing or query optimization
  one.
