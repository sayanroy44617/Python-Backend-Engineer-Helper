# ORM Models and Sessions

## What

SQLAlchemy's **ORM** maps Python classes to database tables (**models**),
letting you work with rows as objects. A **Session** is the ORM's unit of
work — it tracks loaded objects, stages changes, and manages the
transaction that persists them.

## Why

Writing raw SQL for every query works, but the ORM removes repetitive
boilerplate, gives you Python objects with relationships instead of manual
joins, and integrates with migrations (Alembic) that track schema changes
alongside model definitions. Understanding the Session's object lifecycle
is essential — most confusing SQLAlchemy bugs (stale data, unexpected
queries, "object not bound to a session" errors) come from misunderstanding
it.

## How

### Declarative models (SQLAlchemy 2.x style)

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, ForeignKey

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    total: Mapped[float]
```

`Mapped[...]` type annotations (2.x style) drive both the Python type and
the inferred SQL column type — this is the same "type hints as source of
truth" pattern seen in
[Type Hints](../../python/05-type-hints.md) and Pydantic, applied to the
ORM layer.

### Creating tables from models

```python
Base.metadata.create_all(engine)
```

Useful for tests/prototypes; in a real application, schema changes are
tracked via Alembic migrations instead (see
[Async SQLAlchemy and Migrations](04-async-sqlalchemy-and-migrations.md)),
not by calling `create_all` against a production database.

### The Session — unit of work

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("postgresql+psycopg://user:pass@host/db")

with Session(engine) as session:
    user = User(name="Sayan", email="sayan@example.com")
    session.add(user)
    session.commit()
    print(user.id)  # populated after commit (or after a flush)
```

The session doesn't write to the database on every `.add()` — it batches
pending changes and issues SQL when it **flushes** (automatically before a
query that needs current data, or explicitly via `.flush()`), and persists
them permanently on `.commit()`.

### Identity map

```python
user_a = session.get(User, 1)
user_b = session.get(User, 1)
user_a is user_b  # True -- same Python object for the same primary key,
                   # within the same session
```

Within a single session, fetching the same row twice returns the **same**
Python object (the identity map) — this is the direct application of
identity vs equality concepts from
[Data Types and Collections](../../python/01-data-types-and-collections.md)
to the ORM layer.

### Object states

```
transient  -- created, not yet added to a session
pending    -- added to a session, not yet flushed to the DB
persistent -- flushed/committed, associated with a session and a DB row
detached   -- was persistent, but its session has been closed
```

```python
user = User(name="Sayan", email="s@example.com")  # transient
session.add(user)                                   # pending
session.commit()                                    # persistent

session.close()
user.name  # accessing an unloaded attribute on a detached object raises
           # DetachedInstanceError if it wasn't already loaded
```

Understanding these states explains a very common bug: accessing a lazy-
loaded relationship attribute on an object after its session has closed
(e.g. after a FastAPI request-scoped session dependency has ended) raises
`DetachedInstanceError` — see
[Queries, Relationships, and Loading](02-queries-relationships-and-loading.md)
for the eager-loading fix.

### Session scope in a web application

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

One session per request is the standard pattern (see
[FastAPI Dependency Injection](../../fastapi/03-dependency-injection.md))
— the session, and the identity map/object state it tracks, is scoped to
a single request's lifetime, not shared globally across requests.

## When to use

- Declarative models with `Mapped[...]` type annotations for any table
  your application reads/writes regularly.
- One `Session` per logical unit of work (typically one per HTTP request
  in a web app) — never share a session across concurrent requests/
  threads.
- `session.get(Model, pk)` for primary-key lookups (uses the identity map,
  avoiding a redundant query if already loaded in this session).

## When NOT to use

- Don't share a single `Session` instance across multiple requests or
  threads — sessions are not thread-safe and are meant to be short-lived,
  scoped to one unit of work.
- Don't call `Base.metadata.create_all()` against a production database as
  your schema management strategy — use Alembic migrations so schema
  changes are versioned and reviewable.
- Don't access an object's unloaded attributes/relationships after its
  session has closed — expect a `DetachedInstanceError`, and design around
  it (eager load what you need before the session closes).

## Common mistakes

- Sharing one global `Session` across the whole application instead of
  one per request — leads to stale identity-map data and thread-safety
  bugs under concurrent requests.
- Forgetting the difference between `flush` (sends SQL, still
  rollback-able) and `commit` (permanently persists, ends the
  transaction) — assuming an object has a real database ID before either
  has happened.
- Accessing a relationship/attribute on a detached object (after the
  session closed) and being confused by `DetachedInstanceError`.
- Calling `create_all()`/`drop_all()` against a database that already has
  Alembic-managed migrations, causing schema drift between what
  migrations describe and what actually exists.

## Interview questions

- What does the SQLAlchemy Session actually do — what's the "unit of
    work" pattern?

    **Answer:** The Session tracks every object you add/modify/delete
    in-memory and batches all the resulting SQL into one flush at commit
    time, instead of sending a statement immediately for every change —
    that batching-and-committing-together approach is the "unit of work"
    pattern.

- What's the difference between `flush()` and `commit()`?

    **Answer:** `flush()` sends pending SQL to the database (so it's
    visible within the current transaction) but doesn't end the
    transaction. `commit()` flushes *and* commits the transaction, making
    changes permanent and visible to others.

- What is the identity map, and what guarantee does it provide within a
    single session?

    **Answer:** The identity map ensures that querying the same row twice
    in one session returns the *same Python object*, not two separate
    copies — so mutating it once is consistent everywhere you reference it
    in that session.

- What are the transient/pending/persistent/detached object states, and
    when does `DetachedInstanceError` occur?

    **Answer:** Transient = created but never added to a session.
    Pending = added, not flushed yet. Persistent = flushed/committed, has a
    DB row, tracked by a session. Detached = was persistent, but its
    session closed — accessing a lazy-loaded attribute on it then raises
    `DetachedInstanceError` because there's no session left to run the
    query.

- Why is "one session per request" the standard pattern in a web
    application, rather than one global session?

    **Answer:** A shared global session accumulates state across unrelated
    requests, risking stale identity-map data and objects leaking between
    users. A fresh session per request keeps each request's scope isolated
    and short-lived, matching how a transaction should be scoped.

## Senior-level considerations

- The identity map's "same object for the same row within a session"
  guarantee is what makes ORM-level equality/mutation tracking work
  correctly — but it also means a session growing very large (loading
  many thousands of objects) has real memory cost; understand when to use
  `session.expire_all()`/bulk operations that bypass the ORM object
  overhead for large batch operations. For example, iterating and
  updating 200k rows through full ORM objects can OOM a worker where a
  `session.execute(update(...))` bulk statement wouldn't.
- Session lifecycle bugs (detached instance errors, stale data from an
  overly long-lived session) are among the most common production
  SQLAlchemy issues — tracing them back to session scope and object state
  is a core debugging skill. For example, returning an ORM object from a
  FastAPI dependency after its session closed, then accessing a
  lazy-loaded relationship in the response serializer, is a classic
  `DetachedInstanceError`.
- Treating models (`Mapped[...]` declarative classes) as the source of
  truth for schema, paired with Alembic autogenerate for migrations,
  keeps schema evolution reviewable and versioned — critical for any team
  running the same schema across dev/staging/production environments. For
  example, a PR changing a `Mapped[str]` to `Mapped[str | None]` alongside
  its generated Alembic migration lets reviewers see the model and schema
  change together.
