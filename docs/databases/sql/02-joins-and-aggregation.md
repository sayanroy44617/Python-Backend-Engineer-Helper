# Joins and Aggregation

## What

**Joins** combine rows from multiple tables based on a related column.
**Aggregation** (`GROUP BY`/`HAVING` with aggregate functions like
`COUNT`/`SUM`/`AVG`) collapses multiple rows into summary values per group.

## Why

Relational data is normalized across multiple tables (see
[Database Design and Constraints](../postgresql/01-database-design-and-constraints.md));
joins are how you reassemble related data for a query. Aggregation is how
you answer "how many/how much per group" questions — both are used
constantly in reporting endpoints, dashboards, and everyday backend
queries.

## How

### Join types

```sql
-- INNER JOIN: only rows with a match in both tables
SELECT o.id, u.name
FROM orders o
INNER JOIN users u ON o.user_id = u.id;

-- LEFT JOIN: all rows from the left table, matched columns NULL if no match
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
-- users with no orders still appear, with o.id = NULL

-- RIGHT JOIN: mirror of LEFT JOIN (rare in practice; usually rewritten as LEFT JOIN)
-- FULL OUTER JOIN: all rows from both sides, NULLs where no match
SELECT u.name, o.id
FROM users u
FULL OUTER JOIN orders o ON o.user_id = u.id;
```

| Join | Keeps unmatched left rows? | Keeps unmatched right rows? |
|---|---|---|
| `INNER JOIN` | No | No |
| `LEFT JOIN` | Yes | No |
| `RIGHT JOIN` | No | Yes |
| `FULL OUTER JOIN` | Yes | Yes |

### Filtering matched vs unmatched rows correctly

```sql
-- Bug: WHERE filters out the NULL rows LEFT JOIN was meant to preserve,
-- silently turning this back into an INNER JOIN
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'shipped';

-- Fix: move the condition into the JOIN clause so unmatched users remain
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'shipped';
```

This is one of the most common real-world join bugs — a `WHERE` clause on
the right-hand table silently negates the "keep unmatched rows" intent of
a `LEFT JOIN`.

### Self-joins and many-to-many joins

```sql
-- Self-join: employees and their managers, both in the `employees` table
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Many-to-many via a join/association table
SELECT p.name, t.name
FROM products p
JOIN product_tags pt ON pt.product_id = p.id
JOIN tags t ON t.id = pt.tag_id;
```

### Aggregation with `GROUP BY`

```sql
SELECT status, COUNT(*) AS order_count, SUM(total) AS total_revenue
FROM orders
GROUP BY status;
```

Every non-aggregated column in `SELECT` must appear in `GROUP BY` (standard
SQL) — `status` here is the grouping key; `COUNT`/`SUM` collapse all rows
within each group into one summary row.

### Filtering groups with `HAVING`

```sql
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 10;
```

`WHERE` filters rows **before** grouping; `HAVING` filters groups **after**
aggregation — you can't write `WHERE COUNT(*) > 10` because `COUNT(*)`
doesn't exist yet at the `WHERE` stage (see the clause evaluation order in
[Querying Fundamentals](01-querying-fundamentals.md)).

```sql
SELECT status, COUNT(*)
FROM orders
WHERE created_at > now() - interval '30 days'  -- filters rows first
GROUP BY status
HAVING COUNT(*) > 5;                             -- then filters groups
```

### Common aggregate functions

```sql
COUNT(*), COUNT(DISTINCT user_id), SUM(total), AVG(total), MIN(total), MAX(total)
```

## When to use

- `INNER JOIN` when you only want rows that have a match on both sides
  (e.g. orders that definitely belong to an existing user).
- `LEFT JOIN` when you want to preserve all rows from one side even
  without a match (e.g. all users, including those with zero orders).
- `GROUP BY`/`HAVING` for per-category summaries (counts, sums, averages)
  rather than fetching all rows and aggregating in application code.

## When NOT to use

- Don't put a filter on the "preserved" side of a `LEFT JOIN` in `WHERE`
  when you mean to filter only the joined side — move it into the `ON`
  clause instead, or you'll silently lose the outer-join behavior.
- Don't aggregate large datasets in application code (fetching every row
  and summing in Python) when the database can do it far more efficiently
  in one query.
- Don't default to `FULL OUTER JOIN`/`RIGHT JOIN` out of habit — `LEFT
  JOIN` covers the vast majority of real use cases and is easier to reason
  about consistently.

## Common mistakes

- Filtering a `LEFT JOIN`'s right-hand table in `WHERE`, accidentally
  converting it into an `INNER JOIN`.
- Forgetting that `GROUP BY` requires every non-aggregated selected column
  to be part of the grouping key.
- Using `WHERE` where `HAVING` was needed (filtering on an aggregate
  result) and getting a syntax/semantic error or unexpected results.
- Producing a "fan-out" row multiplication bug: joining a one-to-many
  relationship (e.g. orders to order items) and then aggregating the
  "one" side's columns without `DISTINCT`, double-counting values.

## Interview questions

- What's the difference between `INNER JOIN` and `LEFT JOIN`? What
    happens to unmatched rows in each?

    **Answer:** `INNER JOIN` only keeps rows that match in both tables —
    unmatched rows on either side are dropped. `LEFT JOIN` keeps every row
    from the left table regardless of a match, filling unmatched right-side
    columns with `NULL`.

- Why does putting a condition on the right table in `WHERE` silently
    break a `LEFT JOIN`'s intent?

    **Answer:** `WHERE right.col = 'x'` filters out any row where
    `right.col` is `NULL` — which is exactly the "no match" rows a
    `LEFT JOIN` was meant to keep. That condition needs to move into the
    `ON` clause instead, so the filter runs during the join, not after.

    ```sql
    -- Wrong: turns LEFT JOIN back into an INNER JOIN
    SELECT * FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE o.status = 'shipped';

    -- Right: keeps users with no matching order
    SELECT * FROM users u
    LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'shipped';
    ```

- What's the difference between `WHERE` and `HAVING`? Why can't you
    filter on `COUNT(*)` in `WHERE`?

    **Answer:** `WHERE` filters individual rows before grouping happens;
    `HAVING` filters groups after `GROUP BY`/aggregation. `COUNT(*)` only
    exists after grouping, so it has to be filtered with `HAVING`.

- How would you find users with more than 10 orders using `GROUP BY`/
    `HAVING`?

    **Answer:** Group by `user_id`, aggregate with `COUNT(*)`, and filter
    the grouped result with `HAVING`, since the count only exists after
    grouping.

    ```sql
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
    HAVING COUNT(*) > 10;
    ```

- What is a "fan-out" bug in a join + aggregation query, and how do you
    avoid it?

    **Answer:** Joining a table to a one-to-many related table multiplies
    rows before you aggregate, so `SUM`/`COUNT` on the "one" side counts
    duplicates. Fix it by aggregating the "many" side in a subquery/CTE
    first, then joining the pre-aggregated result.

## Senior-level considerations

- Join type choice directly affects query plans and performance —
  understanding what indexes support a given join (see
  [Indexes and Query Optimization](04-indexes-and-query-optimization.md))
  is essential once queries run against real production data volumes. For
  example, an unindexed foreign key used as a join condition forces a
  sequential scan on the joined table for every row.
- The `LEFT JOIN` + `WHERE`-on-right-table bug is common enough in code
  review that many teams treat "does this WHERE clause accidentally negate
  the LEFT JOIN?" as a standard review checklist item for any query
  touching optional relationships. For example, a "users with zero
  orders" report silently returning zero rows is a classic symptom of this
  bug.
- Aggregation performed in the database (vs. in application code after
  fetching all rows) is almost always both faster and more correct under
  concurrent writes — pushing aggregation logic into SQL is a common,
  high-value performance optimization identified during profiling (see
  [Performance and Profiling](../../python/14-performance-and-profiling.md)).
  For example, fetching 100k order rows into Python just to `sum()` them
  is far slower than `SELECT SUM(amount) FROM orders` in the database.
