# Subqueries, CTEs, and Window Functions

## What

**Subqueries** are queries nested inside another query. **CTEs** (Common
Table Expressions, `WITH ... AS`) name a subquery for reuse and
readability. **Window functions** compute values across a set of rows
related to the current row (e.g. running totals, rankings) without
collapsing rows the way `GROUP BY` does.

## Why

These three tools let you express queries that would otherwise require
multiple round-trips or application-level post-processing. Window
functions in particular are frequently the difference between a single
efficient SQL query and fetching everything into Python to compute
rankings/running totals manually.

## How

### Subqueries

```sql
-- Scalar subquery -- returns a single value, usable like a column
SELECT name, (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;

-- Subquery in WHERE
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE status = 'shipped');

-- Correlated subquery -- references the outer query, re-evaluated per row
SELECT * FROM orders o
WHERE total > (SELECT AVG(total) FROM orders o2 WHERE o2.user_id = o.user_id);
```

A **correlated** subquery depends on the outer query's current row and
runs once per outer row (potentially expensive); a non-correlated
subquery runs once, independent of the outer query.

### CTEs

```sql
WITH high_value_orders AS (
    SELECT * FROM orders WHERE total > 1000
)
SELECT user_id, COUNT(*)
FROM high_value_orders
GROUP BY user_id;
```

CTEs name an intermediate result, improving readability for multi-step
queries — the equivalent subquery nested inline would be harder to read.
In PostgreSQL, CTEs are generally inlined by the planner like a subquery
(not an optimization fence by default, since PostgreSQL 12) — treat them
as a readability tool, not automatically a performance one.

### Recursive CTEs

```sql
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id IS NULL           -- base case: top-level

    UNION ALL

    SELECT e.id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id  -- recursive case
)
SELECT * FROM org_chart ORDER BY depth;
```

Recursive CTEs solve hierarchical/tree-structured queries (org charts,
category trees, graph traversal) that would otherwise require multiple
round-trips or application-level recursion.

### Window functions

```sql
SELECT
    user_id,
    total,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at) AS order_sequence,
    SUM(total) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total,
    RANK() OVER (ORDER BY total DESC) AS overall_rank
FROM orders;
```

Unlike `GROUP BY`, a window function doesn't collapse rows — every input
row still appears in the output, with an extra computed column. `PARTITION
BY` defines the "window" of rows to compute over (like a `GROUP BY` key,
but without collapsing); `ORDER BY` inside `OVER (...)` defines row order
within each partition (needed for running totals, row numbers, ranks).

### Common window functions

| Function | Purpose |
|---|---|
| `ROW_NUMBER()` | Sequential number per partition |
| `RANK()` / `DENSE_RANK()` | Ranking, with/without gaps for ties |
| `LAG()` / `LEAD()` | Value from a previous/next row in the partition |
| `SUM()`/`AVG()`/`COUNT() OVER (...)` | Running/moving aggregates without collapsing rows |
| `FIRST_VALUE()` / `LAST_VALUE()` | First/last value in the window frame |

```sql
-- Month-over-month comparison using LAG
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS previous_month_revenue
FROM monthly_revenue;
```

### Window functions vs `GROUP BY`

```sql
-- GROUP BY: one row per user, detail rows are gone
SELECT user_id, SUM(total) FROM orders GROUP BY user_id;

-- Window function: every order row preserved, with a running total column added
SELECT user_id, total, SUM(total) OVER (PARTITION BY user_id) AS user_total
FROM orders;
```

## When to use

- CTEs to break a complex query into named, readable steps — especially
  when the same intermediate result is referenced more than once.
- Recursive CTEs for hierarchical data (org charts, categories, nested
  comments) instead of recursive application code issuing repeated
  queries.
- Window functions for rankings, running totals, and row-to-row
  comparisons (e.g. "revenue vs. previous month") that need per-row detail
  alongside aggregate context — this is exactly the case `GROUP BY` can't
  handle since it collapses rows.

## When NOT to use

- Don't use a correlated subquery inside a large result set when a `JOIN`
  or window function would compute the same thing in one pass — a
  correlated subquery evaluated once per outer row can be significantly
  slower at scale.
- Don't reach for a recursive CTE for shallow, fixed-depth hierarchies
  (e.g. a known 2-level category structure) — a couple of simple joins are
  clearer.
- Don't assume a CTE is materialized/optimized separately in PostgreSQL by
  default — treat it as an inlined subquery for performance reasoning
  unless you explicitly force materialization.

## Common mistakes

- Using a correlated subquery where a join or window function would be
  both clearer and faster.
- Confusing `RANK()` and `ROW_NUMBER()` — `RANK()` leaves gaps after ties
  (1, 2, 2, 4), `ROW_NUMBER()` never does (1, 2, 3, 4), and `DENSE_RANK()`
  doesn't leave gaps either (1, 2, 2, 3).
- Forgetting `PARTITION BY` in a window function, causing the window to
  span the entire result set instead of per-group as intended.
- Assuming a recursive CTE has no depth limit — an unbounded/incorrect
  recursive case can loop far longer than intended (`RECURSIVE` queries
  can be limited or guarded explicitly).

## Interview questions

1. What's the difference between a `GROUP BY` aggregate and a window
   function that also aggregates?
2. What is a correlated subquery, and why can it be slower than a join?
3. How would you compute a running total per user using a window
   function?
4. What's the difference between `RANK()`, `DENSE_RANK()`, and
   `ROW_NUMBER()`?
5. How would you query a hierarchical structure (e.g. an org chart) using
   a recursive CTE?

## Senior-level considerations

- Window functions frequently replace what would otherwise require
  fetching all rows into application code and computing rankings/running
  totals in Python — pushing this into SQL is both faster and reduces
  data transferred over the network.
- Query readability matters at scale: CTEs make complex analytical queries
  reviewable and maintainable, which matters when queries live in
  migrations, reporting code, or are revisited months later during an
  incident investigation.
- Recognizing when a correlated subquery is a performance smell (and
  rewriting it as a join or window function) is a common, high-value query
  optimization pattern surfaced during query plan analysis (see
  [Indexes and Query Optimization](04-indexes-and-query-optimization.md)).
