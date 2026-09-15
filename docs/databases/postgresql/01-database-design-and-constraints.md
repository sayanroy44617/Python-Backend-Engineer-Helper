# Database Design and Constraints

## What

**Database design** is the process of modeling entities and relationships
as tables. **Primary keys**, **foreign keys**, and **constraints** enforce
data integrity at the database level. **Normalization** is the discipline
of structuring tables to minimize redundancy and avoid update anomalies.

## Why

Application-level validation (Pydantic, service-layer checks) can be
bypassed by a bug, a direct SQL script, or a second application writing to
the same database. Database constraints are the last line of defense that
guarantees data integrity regardless of what wrote the data — a
foundational reliability property that's expensive to retrofit once bad
data has accumulated.

## How

### Primary keys

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL
);
```

A primary key uniquely identifies each row and is automatically indexed.
`GENERATED ALWAYS AS IDENTITY` (modern standard SQL, PostgreSQL 10+) is
preferred over the legacy `SERIAL` type for auto-incrementing IDs.

```sql
-- UUID primary key -- useful when IDs must be unguessable or generated
-- client-side/across distributed systems without a central sequence
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
);
```

| | Auto-increment integer | UUID |
|---|---|---|
| Size | Smaller (8 bytes) | Larger (16 bytes) |
| Guessability | Sequential, guessable | Effectively unguessable |
| Generation | Requires a round-trip to the DB (or `IDENTITY`) | Can be generated client-side before insert |
| Index locality | Good (sequential inserts) | Can fragment indexes (random inserts) unless using a time-ordered UUID variant |

### Foreign keys

```sql
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE
);
```

A foreign key guarantees `user_id` always references an existing `users`
row — the database rejects an `INSERT`/`UPDATE` that would violate this,
regardless of what application code intended.

| `ON DELETE` option | Behavior when the referenced row is deleted |
|---|---|
| `CASCADE` | Delete dependent rows too |
| `RESTRICT` / `NO ACTION` | Reject the delete if dependent rows exist |
| `SET NULL` | Set the foreign key column to `NULL` |
| `SET DEFAULT` | Set the foreign key column to its default value |

Choosing the wrong `ON DELETE` behavior is a common source of either
unexpected cascading data loss (`CASCADE` used too broadly) or confusing
"can't delete" errors (`RESTRICT` where cascading was actually intended).

### Constraints

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    sku TEXT UNIQUE NOT NULL
);
```

| Constraint | Enforces |
|---|---|
| `NOT NULL` | Column must have a value |
| `UNIQUE` | No duplicate values in the column (across rows) |
| `CHECK` | An arbitrary boolean condition per row |
| `PRIMARY KEY` | Unique + not null, identifies the row |
| `FOREIGN KEY` | Value must exist in the referenced table |

These are checked on every write — an application bug that skips
validation still can't insert a negative price or a duplicate SKU.

### Normalization

```sql
-- Unnormalized: repeats user_name/user_email on every order row
CREATE TABLE orders (id INT, user_name TEXT, user_email TEXT, total NUMERIC);

-- Normalized (3NF): user data lives once, referenced by foreign key
CREATE TABLE users (id INT PRIMARY KEY, name TEXT, email TEXT);
CREATE TABLE orders (id INT PRIMARY KEY, user_id INT REFERENCES users(id), total NUMERIC);
```

| Normal form | Rule |
|---|---|
| 1NF | Atomic column values (no repeating groups/arrays in a single column) |
| 2NF | 1NF + every non-key column depends on the *whole* primary key (relevant for composite keys) |
| 3NF | 2NF + no non-key column depends on another non-key column (no transitive dependency) |

Normalization avoids **update anomalies**: in the unnormalized example,
changing a user's email requires updating every order row that duplicated
it — miss one, and the data is now inconsistent.

### Deliberate denormalization

```sql
-- Storing a computed/duplicated total on the order for fast reads,
-- accepting the risk of it drifting from the source of truth
ALTER TABLE orders ADD COLUMN item_count INT NOT NULL DEFAULT 0;
```

Denormalization trades normalization's consistency guarantees for read
performance — a deliberate choice for specific hot paths (e.g. avoiding a
`COUNT`/`JOIN` on every read), not a default starting point.

## When to use

- Foreign keys and constraints on every relationship and business rule
  that must always hold, even if application code also validates it —
  defense in depth, not redundancy for its own sake.
- Normalize by default (aim for 3NF) when designing new schemas — it's
  easier to selectively denormalize a known hot path later than to
  untangle redundant, inconsistent data after the fact.
- UUID primary keys when IDs must be generated client-side, merged across
  distributed systems, or must not reveal sequence/volume information.

## When NOT to use

- Don't skip foreign keys/constraints "for simplicity" in early
  development — data integrity bugs discovered late (after bad data has
  accumulated) are far more expensive to fix than a constraint would have
  been to write upfront.
- Don't over-normalize to the point that every simple query needs many
  joins across many tiny tables — some deliberate denormalization for
  genuinely hot, read-heavy paths is a reasonable trade-off, made
  consciously.
- Don't use `ON DELETE CASCADE` reflexively on every foreign key — for
  relationships where the "child" data should survive or block deletion
  (e.g. financial records), `RESTRICT` or `SET NULL` is often the safer
  default.

## Common mistakes

- Relying solely on application-level validation and skipping database
  constraints, then discovering inconsistent data after a bug bypassed the
  application layer entirely.
- Choosing `ON DELETE CASCADE` without considering the full blast radius —
  deleting a user accidentally cascading through orders, payments, and
  audit logs.
- Denormalizing prematurely, before profiling shows it's actually needed,
  then having to maintain consistency between the duplicated and source
  data manually.
- Using a natural key (e.g. email, SSN) as a primary key when it might
  need to change later — prefer a surrogate key (auto-increment/UUID) with
  a separate `UNIQUE` constraint on the natural key.

## Interview questions

1. Why are database-level constraints important even if the application
   already validates the same rules?
2. What's the difference between `ON DELETE CASCADE`, `RESTRICT`, and `SET
   NULL`? When would you choose each?
3. What update anomaly does normalization prevent? Give a concrete
   example.
4. What are the trade-offs between an auto-increment integer primary key
   and a UUID primary key?
5. When would deliberate denormalization be the right call, despite
   normalization's benefits?

## Senior-level considerations

- Schema design decisions (key types, normalization level, cascade
  behavior) are expensive to change once a table has real production
  data and many dependent services — invest more design time upfront than
  feels necessary for a "simple" table.
- Constraints double as documentation: a `NOT NULL`/`CHECK`/foreign key
  makes an invariant explicit and enforced, rather than relying on
  scattered application code and tribal knowledge to maintain it.
- Denormalization decisions should be traceable to a specific, measured
  performance need (see
  [Indexes and Query Optimization](../sql/04-indexes-and-query-optimization.md))
  — treat it as a targeted trade-off with a maintenance cost, not a
  default schema style.
