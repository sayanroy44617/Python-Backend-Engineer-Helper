# Querying Fundamentals

## What

The four core SQL statements for reading and mutating data: `SELECT`,
`INSERT`, `UPDATE`, and `DELETE` (together, "CRUD" at the database level).

## Why

Every backend service ultimately reduces to these four operations against
a database, however many layers of ORM/service abstraction sit on top.
Understanding their exact semantics — what gets matched, what gets
returned, what happens with no matching rows — is a prerequisite for
reasoning about SQLAlchemy behavior and query performance later.

## How

### `SELECT`

```sql
SELECT id, name, email
FROM users
WHERE is_active = true
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;
```

Clause execution order (conceptually, not written order):
`FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`.
This is why you can't reference a `SELECT`-aliased column in `WHERE` (it
doesn't exist yet at that logical stage) but you can in `ORDER BY`.

```sql
SELECT DISTINCT status FROM orders;          -- unique values only
SELECT COUNT(*) FROM orders WHERE status = 'shipped';
```

### `INSERT`

```sql
INSERT INTO users (name, email)
VALUES ('Sayan', 'sayan@example.com')
RETURNING id;

INSERT INTO users (name, email)
VALUES ('A', 'a@example.com'), ('B', 'b@example.com');  -- multi-row insert
```

`RETURNING` (a PostgreSQL extension to standard SQL) returns generated
values (like an auto-increment `id`) from the same statement, avoiding a
separate round-trip query.

```sql
INSERT INTO users (email, name) VALUES ('a@example.com', 'A')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;  -- "upsert"
```

`ON CONFLICT` handles the common "insert or update" pattern atomically,
avoiding a race condition between a separate `SELECT` check and `INSERT`.

### `UPDATE`

```sql
UPDATE orders
SET status = 'shipped', shipped_at = now()
WHERE id = 42;
```

**Always include a `WHERE` clause** unless you genuinely intend to update
every row in the table — an `UPDATE` without `WHERE` is one of the most
common catastrophic mistakes in production SQL.

```sql
UPDATE orders SET status = 'shipped' WHERE id = 42 RETURNING *;
```

### `DELETE`

```sql
DELETE FROM orders WHERE status = 'cancelled' AND created_at < now() - interval '90 days';
```

Same warning applies: a bare `DELETE FROM orders;` with no `WHERE` deletes
every row. Many teams enforce a linter/review rule requiring an explicit
`WHERE` (or a deliberate comment confirming intent) on any `UPDATE`/
`DELETE` statement.

### Transactions around multiple statements

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Wrapping related `INSERT`/`UPDATE`/`DELETE` statements in a transaction
ensures they all succeed or all roll back together — covered in depth in
[Transactions and ACID](../postgresql/02-transactions-and-acid.md).

## When to use

- `SELECT ... RETURNING` (INSERT/UPDATE with RETURNING) to avoid an extra
  round-trip when you need the row you just wrote.
- `ON CONFLICT` (upsert) for "insert if new, update if exists" logic
  instead of a check-then-act pattern prone to race conditions.
- Explicit column lists in `INSERT`/`SELECT` rather than `*` — resilient to
  schema changes (new columns) and clearer about intent.

## When NOT to use

- Don't write `UPDATE`/`DELETE` without a `WHERE` clause unless that's
  genuinely the intent — always double-check before running in production.
- Don't use `SELECT *` in application code that will be maintained
  long-term — it silently changes shape when the schema changes and
  fetches unnecessary columns/bandwidth.
- Don't rely on separate `SELECT` + `INSERT`/`UPDATE` calls for "check
  then act" logic when a single atomic statement (`ON CONFLICT`, or a
  proper transaction with the right isolation level) avoids a race
  condition.

## Common mistakes

- Running `UPDATE`/`DELETE` without `WHERE` in production — always test
  destructive statements against a `SELECT` with the same `WHERE` clause
  first to confirm which rows would be affected.
- Assuming clause execution follows written (`SELECT` → `FROM` → `WHERE`)
  order rather than logical execution order (`FROM` → `WHERE` → ... →
  `SELECT`), leading to confusion about what's referenceable where.
- Forgetting `RETURNING` and issuing a second `SELECT` to fetch a row that
  was already available from the write statement.
- Not handling `NULL` correctly in `WHERE` clauses — `column = NULL` never
  matches (use `IS NULL`), a common source of "why isn't this row showing
  up" bugs.

## Interview questions

- What is the logical order of SQL clause evaluation, and why does it
    matter for what you can reference where?

    **Answer:** Logically: `FROM` → `WHERE` → `GROUP BY` → `HAVING` →
    `SELECT` → `ORDER BY`. That's why you can't reference a `SELECT`
    column alias in `WHERE` (it doesn't exist yet at that stage) but you
    can in `ORDER BY` (it runs after `SELECT`).

- Why should every `UPDATE`/`DELETE` statement include a `WHERE` clause
    (barring deliberate exceptions)?

    **Answer:** Without `WHERE`, the statement applies to every row in the
    table — a classic "forgot the WHERE" incident that wipes/overwrites an
    entire table instead of one row.

    ```sql
    DELETE FROM orders WHERE id = 42;   -- one row
    DELETE FROM orders;                  -- every row, silently
    ```

- What does `RETURNING` do, and what problem does it solve?

    **Answer:** `RETURNING` gives back the row(s) affected by an
    `INSERT`/`UPDATE`/`DELETE` in the same round trip — so you don't need a
    separate `SELECT` afterward just to get the generated `id` or updated
    values.

    ```sql
    INSERT INTO users (name) VALUES ('ana') RETURNING id;
    ```

- How does `INSERT ... ON CONFLICT DO UPDATE` avoid a race condition that
    a separate `SELECT`-then-`INSERT` wouldn't?

    **Answer:** `SELECT` then `INSERT` has a gap where two concurrent
    requests can both see "no row exists" and both try to insert, causing
    a duplicate-key error. `ON CONFLICT` makes the check-and-write atomic
    at the database level, so only one wins cleanly.

- Why does `WHERE column = NULL` never match any rows?

    **Answer:** `NULL` means "unknown," and comparing anything to
    "unknown" with `=` also yields unknown (not true), so no rows ever
    match. You need `WHERE column IS NULL` instead.

## Senior-level considerations

- Preferring atomic statements (`ON CONFLICT`, single `UPDATE ... WHERE`)
  over "check then act" application logic eliminates a whole class of
  race conditions under concurrent load — relevant to correctness at
  scale, not just style. For example, two requests incrementing a
  `stock_count` with `UPDATE products SET stock = stock - 1 WHERE id = ?`
  is safe under concurrency; reading the value in Python then writing it
  back is not.
- Destructive statement safety (mandatory `WHERE`, dry-run via `SELECT`
  first, code review for raw SQL migrations) is often enforced via team
  process/tooling, not just individual discipline — worth establishing as
  a convention on any team writing raw SQL or migrations. For example,
  requiring every migration PR to include the `SELECT` version of a
  `DELETE`'s `WHERE` clause in the description, reviewed before merge.
- Understanding exact `SELECT`/`UPDATE`/`DELETE` semantics is the
  foundation for reasoning correctly about what SQLAlchemy generates and
  executes underneath the ORM abstraction (see
  [SQLAlchemy](../sqlalchemy/index.md)). For example, knowing that
  `session.query(User).filter(User.id == 1).delete()` compiles to a plain
  `DELETE ... WHERE` (not per-row `__del__` calls) explains why bypassing
  the ORM's in-memory objects also bypasses any Python-side cascade logic.
