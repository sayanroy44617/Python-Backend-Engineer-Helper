# Interview Prep: SQLAlchemy

## How to approach SQLAlchemy interviews

SQLAlchemy questions at this level focus on the ORM's session/identity
lifecycle, relationship loading strategies (and the N+1 problem they
prevent or cause), and how the async variant differs from the sync one
— not just model definition syntax. The
[SQLAlchemy](../databases/sqlalchemy/index.md) section covers the
mechanics; this page distills the questions most likely to surface.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | What is a SQLAlchemy `Session`, and what's its relationship to a database transaction? | The `Session` is a unit-of-work object tracking loaded/modified objects (the identity map) and manages an underlying transaction — changes are staged in memory and only flushed/committed to the database when explicitly told to. | [ORM Models and Sessions](../databases/sqlalchemy/01-orm-models-and-sessions.md) |
| 2 | What's the identity map, and what problem does it solve? | Within one `Session`, querying the same row twice returns the *same Python object* rather than two separate copies — this avoids inconsistent in-memory state and redundant queries for objects already loaded. | [ORM Models and Sessions](../databases/sqlalchemy/01-orm-models-and-sessions.md) |
| 3 | What causes the N+1 query problem, and how do you fix it in SQLAlchemy? | Iterating over a collection and accessing a lazy-loaded relationship per item triggers one query per item (N queries) in addition to the original query — fixed by eager loading (`joinedload`/`selectinload`) the relationship upfront in the original query. | [Queries, Relationships, and Loading](../databases/sqlalchemy/02-queries-relationships-and-loading.md) |
| 4 | What's the difference between `joinedload` and `selectinload`? | `joinedload` fetches the related rows in the same query via a SQL `JOIN`; `selectinload` issues a second query using `IN (...)` for all parent IDs — `joinedload` can duplicate parent row data across a large one-to-many join, while `selectinload` avoids that at the cost of an extra round trip. | [Queries, Relationships, and Loading](../databases/sqlalchemy/02-queries-relationships-and-loading.md) |
| 5 | Why would a `session.commit()` fail with a `DetachedInstanceError` accessed later? | Accessing a lazy-loaded attribute on an object after its `Session` has been closed has no active connection to issue the query — the object is "detached," which is a common bug when returning ORM objects out of a request-scoped session and accessing relationships afterward. | [ORM Models and Sessions](../databases/sqlalchemy/01-orm-models-and-sessions.md) |
| 6 | How does SQLAlchemy's connection pool relate to your database's connection limit? | The engine maintains a pool of reusable connections sized by `pool_size`/`max_overflow` — sizing this too high across many application instances can exceed the database's actual connection limit, which is why pool sizing must be considered alongside horizontal scaling. | [Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md) |
| 7 | What's the difference between `session.flush()` and `session.commit()`? | `flush()` sends pending changes to the database (assigning generated PKs, etc.) within the current transaction, without ending it — `commit()` flushes and then actually commits the transaction, making the changes permanent and ending the transaction. | [Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md) |
| 8 | How does async SQLAlchemy (`AsyncSession`) differ from the sync ORM in practice? | Query-issuing calls (`execute`, `commit`, `refresh`) become coroutines you `await`, requiring an async driver (e.g. `asyncpg`) — but the underlying session/unit-of-work concepts (identity map, flush/commit) are the same; it's the I/O that becomes non-blocking. | [Async SQLAlchemy and Migrations](../databases/sqlalchemy/04-async-sqlalchemy-and-migrations.md) |
| 9 | Why use Alembic instead of just letting SQLAlchemy create tables from models? | `Base.metadata.create_all()` only creates tables that don't yet exist — it has no concept of altering an existing table's schema over time; Alembic tracks incremental, versioned, reversible migrations, which is what a production system with existing data actually needs. | [Async SQLAlchemy and Migrations](../databases/sqlalchemy/04-async-sqlalchemy-and-migrations.md) |
| 10 | What's the difference between `lazy="select"` (the default) and `lazy="joined"` on a relationship? | `lazy="select"` defers loading the relationship until first accessed (a separate query then); `lazy="joined"` always loads it eagerly via a JOIN in the original query — the right choice depends on whether the relationship is almost always needed alongside the parent. | [Queries, Relationships, and Loading](../databases/sqlalchemy/02-queries-relationships-and-loading.md) |
| 11 | How would you handle a transaction that needs to roll back partway through, in SQLAlchemy? | Use a context manager (`with Session() as session: ... session.commit()`) or explicit `try`/`except`/`session.rollback()` — an unhandled exception inside a `with` block using `session.begin()` triggers an automatic rollback. | [Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md) |
| 12 | Why might increasing `pool_size` not actually fix a "connection pool exhausted" error? | If connections are being held longer than necessary (e.g. a session left open across a slow external API call), the fix is releasing connections sooner, not just adding more — a bigger pool just delays hitting the same underlying problem at a higher connection count. | [Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md) |

## Live-coding / whiteboard tips

- If asked to fix an N+1 problem, name the specific eager-loading
  strategy (`joinedload` vs. `selectinload`) and justify the choice
  based on the relationship's cardinality, rather than just saying
  "eager load it."
- When discussing session lifecycle, be explicit about *when* the
  session is opened and closed relative to a request (e.g. FastAPI's
  per-request dependency pattern) — this is a common follow-up.
- If asked about migrations, mention Alembic by name and describe the
  autogenerate-then-review workflow, not just "run a migration tool."

## Common red flags interviewers watch for

- Not recognizing an N+1 query pattern when shown a loop accessing a
  relationship.
- Confusing `flush()` and `commit()`.
- Assuming increasing pool size is always the right fix for connection
  exhaustion without considering how long connections are held.
- Not knowing why Alembic (or an equivalent migration tool) is needed
  beyond `create_all()`.

## Related deep-dive material

- [SQLAlchemy section overview](../databases/sqlalchemy/index.md) — 4
  topic pages.
