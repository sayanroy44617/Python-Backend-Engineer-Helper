# Indexes and Query Optimization

## What

An **index** is a separate data structure (typically a B-tree) that lets
the database find rows matching a condition without scanning every row.
**Query optimization** is the practice of understanding and improving how
the database executes a query — primarily by reading its execution plan.

## Why

As tables grow, unindexed queries degrade from milliseconds to seconds (or
worse), often the single biggest cause of slow backend endpoints. Indexes
trade write cost and storage for read speed; knowing when and what to
index — and how to verify a query actually uses one — is a core skill for
keeping an API fast as data grows.

## How

### How an index helps

```sql
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

Without an index, `WHERE user_id = 42` requires a **sequential scan**
(checking every row). With an index, the database can navigate a B-tree
to find matching rows directly — the difference between O(n) and roughly
O(log n) for lookups.

### Reading a query plan

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42;
```

```
Index Scan using idx_orders_user_id on orders  (cost=0.29..8.31 rows=1 width=72) (actual time=0.02..0.03 rows=3 loops=1)
  Index Cond: (user_id = 42)
Planning Time: 0.15 ms
Execution Time: 0.05 ms
```

`EXPLAIN ANALYZE` actually runs the query and reports real timing plus the
chosen plan — `Index Scan` confirms the index was used; `Seq Scan` would
indicate a full table scan (expected on tiny tables, a red flag on large
ones).

```
Seq Scan on orders  (cost=0.00..1834.00 rows=1 width=72) (actual time=12.4..12.5 rows=1 loops=1)
  Filter: (user_id = 42)
  Rows Removed by Filter: 99999
```

`Rows Removed by Filter` on a sequential scan shows exactly how much work
was wasted scanning non-matching rows — a strong signal an index is
missing.

### Composite (multi-column) indexes

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
```

A composite index supports queries filtering on the leading column alone
(`user_id`) or both columns together (`user_id AND status`), but **not**
efficiently on the trailing column alone (`status` without `user_id`) —
column order matters and should match the most common query patterns.

### Unique and partial indexes

```sql
CREATE UNIQUE INDEX idx_users_email ON users (email);  -- also enforces uniqueness

CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
```

A **partial index** only indexes rows matching a condition — smaller and
faster than a full index when queries consistently filter on that
condition (e.g. frequently querying only "pending" orders out of millions
of historical ones).

### The write-side cost of indexes

Every index must be updated on every `INSERT`/`UPDATE`/`DELETE` touching
its columns — indexes speed up reads but slow down writes and consume
storage. Adding an index to every column "just in case" is a common
anti-pattern that hurts write-heavy tables.

### N+1 queries

```python
# 1 query to fetch orders, then N queries -- one per order -- to fetch items
for order in orders:
    items = get_items_for_order(order.id)
```

The N+1 problem isn't fixed by indexing alone (each individual query might
be fast) — it's fixed by restructuring to fetch related data in one
query (a `JOIN`, or a single `WHERE id IN (...)` batch query). This
connects directly to SQLAlchemy's eager/lazy loading choices (see
[Queries, Relationships, and Loading](../sqlalchemy/02-queries-relationships-and-loading.md)).

### Other common optimization levers

```sql
-- Avoid SELECT * -- fetch only needed columns, reducing I/O
SELECT id, status FROM orders WHERE user_id = 42;

-- Avoid wrapping an indexed column in a function -- it prevents index use
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-01';        -- can't use a plain index on created_at
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';  -- can
```

## When to use

- Index columns frequently used in `WHERE`, `JOIN ON`, and `ORDER BY`
  clauses — especially foreign keys, which are **not** automatically
  indexed in PostgreSQL (unlike primary keys).
- Composite indexes when queries consistently filter on the same
  combination of columns together.
- Partial indexes when queries consistently target a small, well-defined
  subset of a large table.
- `EXPLAIN ANALYZE` before and after any indexing change to confirm it
  actually improved the plan — don't guess.

## When NOT to use

- Don't index every column "defensively" — each index adds write overhead
  and storage; index based on actual query patterns, not speculation.
- Don't wrap an indexed column in a function/expression in `WHERE` unless
  you've created a matching expression index — it silently defeats the
  index and forces a sequential scan.
- Don't assume a small development/staging dataset reflects production
  query performance — index-related issues often only appear at real data
  volume; test with representative data sizes where possible.

## Common mistakes

- Forgetting that PostgreSQL does **not** automatically index foreign key
  columns — a very common source of slow joins on unindexed `_id` columns.
- Not verifying index usage with `EXPLAIN ANALYZE`, assuming an index is
  used when the planner actually chose a sequential scan (which can
  happen even with an index present, if the planner estimates it's not
  worth it for a small/unselective result).
- Treating the N+1 query problem as an indexing problem when it's
  actually a query-structuring problem — no amount of indexing fixes 101
  separate round-trips instead of 1-2 batched queries.
- Over-indexing a write-heavy table, silently degrading `INSERT`/`UPDATE`
  throughput without realizing indexes were the cause.

## Interview questions

1. How does an index change a `WHERE user_id = 42` query from O(n) to
   roughly O(log n)?
2. What does `EXPLAIN ANALYZE` tell you that `EXPLAIN` alone doesn't?
3. Why does column order matter in a composite index?
4. Why doesn't PostgreSQL automatically index foreign key columns, and why
   does that matter in practice?
5. Why can't the N+1 query problem be solved by adding more indexes?
6. What's the trade-off of adding an index — what does it cost, not just
   what does it save?

## Senior-level considerations

- Index strategy should be driven by actual production query patterns
  (from slow query logs / `pg_stat_statements`), not guesses — adding
  indexes reactively without evidence is a common source of both missed
  opportunities and unnecessary write overhead.
- Query optimization work should start with `EXPLAIN ANALYZE` on the
  actual slow query, not assumptions — the planner's cost estimates and
  actual row counts (`rows=X` vs `rows=Y`) reveal whether statistics are
  stale (`ANALYZE` the table) or the query itself needs restructuring.
- At scale, the biggest performance wins are usually architectural (fixing
  N+1 patterns, adding the right composite/partial indexes, denormalizing
  a hot read path) rather than micro-tuning individual queries — this
  mirrors the general profiling discipline in
  [Performance and Profiling](../../python/14-performance-and-profiling.md):
  measure before optimizing.
