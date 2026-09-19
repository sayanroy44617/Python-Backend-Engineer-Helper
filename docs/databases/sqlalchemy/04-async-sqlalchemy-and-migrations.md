# Async SQLAlchemy and Migrations

## What

**Async SQLAlchemy** provides `AsyncSession`/`AsyncEngine` for use with
`async def` FastAPI routes and async database drivers. **Alembic** is
SQLAlchemy's companion tool for versioned, reviewable schema
**migrations** — tracking how the database schema changes over time
alongside model definitions.

## Why

A synchronous SQLAlchemy call inside an `async def` route blocks the
entire event loop (see
[Asyncio and Concurrency](../../python/13-asyncio-and-concurrency.md)) —
async SQLAlchemy avoids that by using a non-blocking driver end to end.
Alembic solves a different but equally critical problem: schema changes
need to be applied consistently and in order across development, CI,
staging, and production — manually running `ALTER TABLE` statements
doesn't scale and isn't reviewable or reversible.

## How

### Async engine and session

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine("postgresql+asyncpg://user:pass@host/db", pool_size=10)

async def get_db():
    async with AsyncSession(engine) as session:
        yield session
```

Note the driver: `postgresql+asyncpg` (an async driver), not
`postgresql+psycopg` — async SQLAlchemy requires an async-capable DBAPI
underneath; you can't get real async behavior by wrapping a synchronous
driver.

### Async queries

```python
from sqlalchemy import select

async def get_user(session: AsyncSession, user_id: int) -> User | None:
    result = await session.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()

async def create_user(session: AsyncSession, name: str, email: str) -> User:
    user = User(name=name, email=email)
    session.add(user)
    await session.commit()
    await session.refresh(user)  # reload any DB-generated defaults
    return user
```

Every I/O-triggering call (`execute`, `commit`, `refresh`, `flush`) is
`await`ed — this is what keeps the event loop free to handle other
requests while waiting on the database.

### Async relationship loading

```python
from sqlalchemy.orm import selectinload

stmt = select(User).options(selectinload(User.orders))
result = await session.execute(stmt)
users = result.scalars().all()
for user in users:
    print(user.orders)  # already loaded -- no implicit lazy query here
```

Implicit lazy loading (accessing an unloaded relationship attribute,
triggering an on-the-spot query) doesn't work the same way in async
code — it would require a synchronous, blocking query at attribute-access
time. Always eager-load (`selectinload`/`joinedload`) what you'll need
before leaving the session's async context (see
[Queries, Relationships, and Loading](02-queries-relationships-and-loading.md)).

### Wiring into FastAPI

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.engine = create_async_engine(DATABASE_URL, pool_size=10)
    yield
    await app.state.engine.dispose()

async def get_db():
    async with AsyncSession(app.state.engine) as session:
        yield session

@app.get("/users/{user_id}")
async def get_user_route(user_id: int, db: AsyncSession = Depends(get_db)):
    return await get_user(db, user_id)
```

Same pattern as synchronous SQLAlchemy — engine created once in
`lifespan`, session created per request via a dependency — just with
`async`/`await` throughout (see
[Async Endpoints, Background Tasks, and Lifespan](../../fastapi/06-async-endpoints-and-background-tasks.md)).

### Alembic: initializing and generating migrations

```bash
alembic init alembic
alembic revision --autogenerate -m "add users table"
alembic upgrade head
```

`--autogenerate` compares your current SQLAlchemy models against the
database's actual schema and generates a migration script approximating
the diff — it's a starting point requiring review, not a guarantee of
correctness (it can miss certain changes, like some constraint renames or
data migrations).

### A migration script

```python
# alembic/versions/xxxx_add_users_table.py
def upgrade() -> None:
    op.create_table(
        "users",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("name", sa.String(100), nullable=False),
        sa.Column("email", sa.String(255), unique=True, nullable=False),
    )

def downgrade() -> None:
    op.drop_table("users")
```

Every migration defines both `upgrade()` and `downgrade()` — the ability
to roll back a migration is part of the contract, not optional, even
though `downgrade` is used far less often in practice.

### Data migrations

```python
def upgrade() -> None:
    op.add_column("users", sa.Column("is_active", sa.Boolean, server_default="true"))
    # backfill existing rows if a plain column default isn't sufficient
    op.execute("UPDATE users SET is_active = true WHERE is_active IS NULL")
```

Schema changes (`add_column`) and data changes (backfilling values) can
live in the same migration, but consider running large data backfills
separately/in batches for tables with significant existing data — a
single giant `UPDATE` can lock a large table for an extended period (see
[Locks, Deadlocks, and Connection Pooling](../postgresql/03-locks-deadlocks-and-connection-pooling.md)).

## When to use

- Async SQLAlchemy (`AsyncSession` + an async driver) when your FastAPI
  routes are `async def` and you want the database call to genuinely not
  block the event loop.
- `alembic revision --autogenerate` as a starting point for every schema
  change, always followed by manual review of the generated script before
  running it.
- Eager loading (`selectinload`/`joinedload`) proactively in async code,
  since implicit lazy loading isn't a safe fallback the way it can be
  (if inefficient) in sync code.

## When NOT to use

- Don't mix a synchronous driver with `AsyncSession`/`create_async_engine`
  — you need an async-capable DBAPI (`asyncpg`, `psycopg` in async mode)
  end to end.
- Don't trust Alembic's autogenerate output blindly for anything beyond
  simple additive changes — always review the generated migration,
  especially for renames, data migrations, and constraint changes it may
  express in a way that causes data loss.
- Don't run large, unbatched data backfills in a single migration
  transaction against a live production table without considering lock
  duration and table size.

## Common mistakes

- Using a synchronous driver connection string (`postgresql+psycopg`)
  with `create_async_engine`, causing confusing errors or defeating the
  purpose of async entirely.
- Accessing an unloaded lazy relationship on an async-loaded ORM object,
  which doesn't work the way it does in sync code and typically raises an
  error rather than silently issuing a blocking query.
- Editing an already-applied migration file instead of writing a new one
  — once a migration has run in any shared environment (staging,
  production), it should be treated as immutable history.
- Forgetting to review/test `downgrade()`, discovering it doesn't actually
  work correctly only when a rollback is urgently needed.

## Interview questions

- Why does synchronous SQLAlchemy inside an `async def` FastAPI route
    defeat the purpose of using `async def` at all?

    **Answer:** Sync SQLAlchemy calls block the single event-loop thread
    while waiting on the database, so every other coroutine (other
    requests) also has to wait — you've paid for `async def` syntax but
    lost the concurrency benefit it's supposed to give you.

- What driver-level requirement does async SQLAlchemy have that sync
    SQLAlchemy doesn't?

    **Answer:** It needs an async-capable DBAPI driver (e.g. `asyncpg` for
    Postgres instead of `psycopg2`), because the driver itself has to
    support non-blocking I/O for `await` to actually yield control.

- Why should you always eager-load relationships in async code rather
    than relying on lazy loading?

    **Answer:** Lazy loading normally issues a fresh sync-style query the
    moment you touch the attribute — that doesn't work safely in an async
    context without extra plumbing, so you eager-load
    (`selectinload`/`joinedload`) upfront in the original async query
    instead.

- What does `alembic revision --autogenerate` actually do, and why does
    its output still need manual review?

    **Answer:** It diffs your current models against what Alembic thinks
    the DB schema looks like, and generates a migration script for the
    difference. It can miss things (renames look like drop+add, some type
    changes aren't detected) or capture unrelated diffs — you have to read
    and fix the generated file before trusting it.

- Why does every migration need both `upgrade()` and `downgrade()`?

    **Answer:** `upgrade()` applies the change; `downgrade()` is how you
    safely revert it if the deploy needs to be rolled back — without it,
    a bad migration can't be undone cleanly in production.

## Senior-level considerations

- Migration discipline (one migration per schema change, reviewed in the
  same PR as the model change, applied consistently across environments)
  is what keeps schema evolution safe across a team — schema drift between
  what migrations say and what's actually in production is a common,
  hard-to-debug source of incidents. For example, someone manually
  running `ALTER TABLE` directly against production "just this once"
  leaves the migration history lying about the real schema.
- Large-scale data migrations against production tables are an
  operational concern as much as a code concern — batching, running
  during low-traffic windows, and monitoring lock/replication impact are
  standard practices for any migration touching a large, actively-used
  table. For example, adding a `NOT NULL` column with a default to a
  100M-row table can lock it for a long rewrite unless done in batched
  steps.
- Choosing sync vs. async SQLAlchemy is a whole-application architectural
  decision (driver choice, session dependency wiring, relationship loading
  discipline) — mixing the two inconsistently across a codebase is a
  common source of confusing, hard-to-diagnose bugs. For example, one
  route using `asyncpg`/`AsyncSession` and another accidentally importing
  the sync `Session` against the same models creates two incompatible
  session-management patterns in one app.
