# Queries, Relationships, and Loading

## What

SQLAlchemy's query API (`select()`, 2.x style) builds SQL from Python
expressions. **Relationships** (`relationship()`) map foreign keys to
navigable Python attributes (`order.user`, `user.orders`). **Lazy** and
**eager loading** control *when* related objects are actually fetched from
the database.

## Why

How you query and how relationships load directly determines whether your
application issues one efficient query or hundreds of redundant ones (the
N+1 problem, see
[Indexes and Query Optimization](../sql/04-indexes-and-query-optimization.md)).
This is the single most common source of unexpected slowness in
ORM-backed applications.

## How

### Querying (2.x style)

```python
from sqlalchemy import select

stmt = select(User).where(User.email == "sayan@example.com")
user = session.execute(stmt).scalar_one_or_none()

stmt = select(Order).where(Order.total > 100).order_by(Order.created_at.desc())
orders = session.execute(stmt).scalars().all()
```

`scalar_one_or_none()` expects zero or one result and unwraps it directly;
`scalars().all()` returns a list of ORM objects rather than row tuples.

### Defining relationships

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    orders: Mapped[list["Order"]] = relationship(back_populates="user")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    user: Mapped["User"] = relationship(back_populates="orders")
```

`back_populates` keeps both sides of the relationship in sync in Python
memory (setting `order.user = some_user` also updates
`some_user.orders`), without an extra query.

### Lazy loading (the default)

```python
user = session.get(User, 1)
user.orders  # triggers a SEPARATE query here, the first time it's accessed
```

By default, accessing `user.orders` issues a new query at the moment
you access the attribute — convenient, but the source of the N+1 problem
when done in a loop:

```python
users = session.execute(select(User)).scalars().all()
for user in users:
    print(user.orders)  # one query PER user -- N+1
```

### Eager loading strategies

```python
from sqlalchemy.orm import selectinload, joinedload

# selectinload: one extra query fetching ALL related rows for ALL parents
# in a single IN (...) query -- generally the best default for one-to-many
stmt = select(User).options(selectinload(User.orders))
users = session.execute(stmt).scalars().all()
for user in users:
    print(user.orders)  # no additional query -- already loaded

# joinedload: a single JOIN query -- can be more efficient for
# many-to-one/one-to-one, but risks a "fan-out" row multiplication for
# one-to-many relationships (see Joins and Aggregation)
stmt = select(Order).options(joinedload(Order.user))
```

| Strategy | Extra queries | Best for |
|---|---|---|
| `lazy` (default) | 1 per access, per object | Rarely accessed relationships |
| `selectinload` | 1 extra query total (batched via `IN`) | One-to-many, avoiding row fan-out |
| `joinedload` | 0 (single JOIN) | Many-to-one/one-to-one, small result sets |
| `subqueryload` | 1 extra query (correlated subquery) | Legacy alternative to `selectinload`, rarely preferred now |

### Setting a default loading strategy on the relationship

```python
class User(Base):
    orders: Mapped[list["Order"]] = relationship(
        back_populates="user",
        lazy="selectin",  # applies by default unless overridden per-query
    )
```

Setting `lazy="selectin"` on the relationship itself changes the default
behavior everywhere it's accessed, rather than requiring `.options(...)`
on every query — useful for a relationship that's almost always needed
together with its parent.

### Detecting N+1s in development

```python
import logging
logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

Enabling SQL echo logging (or `create_engine(..., echo=True)`) during
development/tests surfaces exactly how many queries a given code path
issues — the most direct way to catch an N+1 before it reaches production.

## When to use

- `selectinload` as the default choice for one-to-many/many-to-many
  relationships you know you'll need — it avoids both N+1 queries and
  join-induced row fan-out.
- `joinedload` for many-to-one/one-to-one relationships where a single
  extra `JOIN` is cheap and doesn't multiply rows.
- Lazy loading (the default) only for relationships accessed rarely/
  conditionally, where paying for eager loading on every query would waste
  work.
- SQL echo logging during development to verify a new code path doesn't
  introduce an N+1.

## When NOT to use

- Don't rely on lazy loading inside a loop over many parent objects — this
  is exactly the N+1 pattern; use `selectinload`/`joinedload` instead.
- Don't use `joinedload` for one-to-many relationships with many related
  rows per parent — the join multiplies parent columns across every child
  row (fan-out), wasting bandwidth and complicating deduplication.
- Don't access lazy-loaded relationships after the session that loaded the
  parent object has closed — this raises `DetachedInstanceError` (see
  [ORM Models and Sessions](01-orm-models-and-sessions.md)); eager-load
  what the caller will need before the session ends.

## Common mistakes

- Looping over a list of ORM objects and accessing a lazy relationship on
  each — the classic N+1 query bug, often invisible until profiling or SQL
  echo logging reveals it.
- Using `joinedload` on a one-to-many relationship with a large number of
  children per parent, causing far more data to be transferred (and
  potentially duplicated parent rows in the result set) than expected.
- Forgetting `back_populates` (or the older `backref`), leading to two
  independent, out-of-sync in-memory relationship attributes.
- Assuming setting a relationship's `lazy=` strategy is a one-time
  decision — different call sites often need different loading strategies
  for the same relationship, which `.options(...)` per-query supports.

## Interview questions

1. What causes the N+1 query problem in an ORM, and how would you detect
   it in development?

   **Answer:** Fetching a list of parent rows, then accessing a
   lazy-loaded relationship on each one in a loop, triggers a separate
   query per parent (1 + N total). You can spot it by turning on SQL echo
   logging (`echo=True`) or an APM query counter and watching the query
   count explode relative to the number of rows.

2. What's the difference between `selectinload` and `joinedload`? When
   would you choose one over the other?

   **Answer:** `selectinload` runs a second, separate `SELECT ... WHERE
   id IN (...)` to fetch related rows in bulk — no duplication, good
   default for one-to-many. `joinedload` fetches everything in a single
   `JOIN`ed query — one round trip, but duplicates parent columns per
   child row, so it's better for one-to-one/many-to-one.

3. Why can `joinedload` cause row "fan-out" on a one-to-many relationship?

   **Answer:** A `JOIN` produces one result row per matching child, so a
   user with 5 orders comes back as 5 rows, each repeating the user's
   columns — the same fan-out issue as a plain SQL join, just from the ORM.

4. What does `back_populates` do, and what happens if you forget it?

   **Answer:** It keeps both sides of a relationship in sync in Python
   memory — appending a child updates the parent's collection and vice
   versa. Forget it and the two sides can silently disagree until you
   re-query from the DB.

5. Why might you want a relationship's default loading strategy to differ
   from what a specific query actually needs?

   **Answer:** A relationship's default (set on the model) is a
   reasonable general-purpose choice, but a specific query might need more
   or less data — you override it per-query with `.options(selectinload
   (...))` rather than changing the model-wide default for every caller.

## Senior-level considerations

- N+1 query bugs are one of the most common, highest-impact performance
  issues in ORM-backed backend services — reviewing new endpoints for
  relationship access inside loops (and verifying with SQL echo logging or
  APM query counts) is a standard part of a thorough code review. For
  example, `for user in users: print(user.orders)` in a template or
  serializer is a textbook N+1 waiting to happen.
- Loading strategy choice interacts directly with the join/fan-out
  concerns from
  [Joins and Aggregation](../sql/02-joins-and-aggregation.md) — understanding
  the underlying SQL each strategy generates (not just the ORM API) is
  necessary to reason about performance correctly. For example, knowing
  `joinedload` produces a single wide `JOIN` explains why it can be slower
  than `selectinload` for a parent with hundreds of children.
- For very large or performance-critical read paths, sometimes bypassing
  the ORM entirely (raw `select()` with only the needed columns, or Core-
  level queries without hydrating full ORM objects) is the right trade-off
  — the ORM's convenience has a real cost in object construction and
  memory that matters at scale. For example, a dashboard endpoint reading
  a handful of columns from a huge table can be far faster as a Core
  query returning tuples than as full hydrated ORM model instances.
