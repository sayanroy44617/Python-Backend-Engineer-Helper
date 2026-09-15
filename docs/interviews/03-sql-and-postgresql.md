# Interview Prep: SQL and PostgreSQL

## How to approach SQL/PostgreSQL interviews

Expect a mix of hands-on query writing (joins, aggregation, window
functions) and conceptual questions about transactions, indexing, and
locking — interviewers are usually checking whether you can reason about
*why* a query is slow or *why* a transaction anomaly happens, not just
whether you can recite SQL syntax. The
[SQL](../databases/sql/index.md) and
[PostgreSQL](../databases/postgresql/index.md) sections cover the
mechanics in depth; this page focuses on the questions most likely to
come up and how to structure a confident answer quickly.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | What's the difference between `INNER JOIN`, `LEFT JOIN`, and `FULL OUTER JOIN`? | `INNER JOIN` returns only matching rows from both tables; `LEFT JOIN` returns all rows from the left table plus matches (NULLs where no match); `FULL OUTER JOIN` returns all rows from both sides, NULL-filling either side where unmatched. | [Joins and Aggregation](../databases/sql/02-joins-and-aggregation.md) |
| 2 | When would you use a window function instead of a `GROUP BY`? | When you need per-row results alongside an aggregate computed over a related set of rows (e.g. "each order's amount alongside that customer's running total") — `GROUP BY` collapses rows into one per group, losing row-level detail a window function preserves. | [Subqueries, CTEs, and Window Functions](../databases/sql/03-subqueries-ctes-and-window-functions.md) |
| 3 | What's the difference between a correlated and a non-correlated subquery, performance-wise? | A non-correlated subquery runs once, independent of the outer query; a correlated subquery re-evaluates once per outer row, which can be far slower without proper indexing — often rewritable as a join for better performance. | [Subqueries, CTEs, and Window Functions](../databases/sql/03-subqueries-ctes-and-window-functions.md) |
| 4 | How does an index actually speed up a query, and when can it hurt? | An index (commonly a B-tree) lets the database seek directly to matching rows instead of scanning the whole table — it speeds up reads but adds overhead to every write (the index must be updated too), so over-indexing hurts write-heavy tables. | [Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md) |
| 5 | How would you diagnose why a specific query is slow? | Run `EXPLAIN ANALYZE` to see the actual query plan (sequential scan vs. index scan, estimated vs. actual row counts) rather than guessing — the plan usually reveals a missing index, a bad join order, or stale statistics. | [Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md) |
| 6 | What do the ACID properties actually guarantee, one by one? | Atomicity: a transaction fully applies or not at all. Consistency: constraints/invariants hold before and after. Isolation: concurrent transactions don't see each other's uncommitted changes (to a degree set by isolation level). Durability: a committed transaction survives a crash. | [Transactions and ACID](../databases/postgresql/02-transactions-and-acid.md) |
| 7 | What's a non-repeatable read, and which isolation level prevents it? | Reading the same row twice within one transaction and getting different values because another transaction committed a change in between — `REPEATABLE READ` (and stricter) prevents it; `READ COMMITTED` (Postgres's default) does not. | [Transactions and ACID](../databases/postgresql/02-transactions-and-acid.md) |
| 8 | What causes a deadlock, and how does PostgreSQL handle it? | Two transactions each hold a lock the other is waiting for, forming a cycle — PostgreSQL detects the cycle and aborts one transaction (returning a deadlock error) to break it; the application must be prepared to retry. | [Locks, Deadlocks, and Connection Pooling](../databases/postgresql/03-locks-deadlocks-and-connection-pooling.md) |
| 9 | Why would a connection pool (e.g. PgBouncer) be necessary in front of PostgreSQL? | Each PostgreSQL connection has real memory/process overhead, and a horizontally scaled application can easily exceed PostgreSQL's practical connection limit — a pooler multiplexes many client connections onto fewer real database connections. | [Locks, Deadlocks, and Connection Pooling](../databases/postgresql/03-locks-deadlocks-and-connection-pooling.md) |
| 10 | How would you enforce that an email column is both unique and never null? | A column-level constraint: `email TEXT NOT NULL UNIQUE` (or an equivalent `UNIQUE` constraint plus `NOT NULL`) — enforced by the database itself, not just application-level validation, which can be bypassed by a second writer or a bug. | [Database Design and Constraints](../databases/postgresql/01-database-design-and-constraints.md) |
| 11 | What's the difference between a `CTE` (`WITH` clause) and a subquery? | Functionally similar (Postgres can inline or materialize either depending on the query planner), but a CTE is named and can be referenced multiple times in the same query, and — especially with `RECURSIVE` — supports patterns (hierarchical queries) a plain subquery can't express as cleanly. | [Subqueries, CTEs, and Window Functions](../databases/sql/03-subqueries-ctes-and-window-functions.md) |
| 12 | How would you paginate a large result set efficiently? | Prefer keyset/cursor-based pagination (`WHERE id > last_seen_id ORDER BY id LIMIT n`) over `OFFSET`-based pagination for large tables — `OFFSET` requires scanning and discarding all preceding rows, getting slower as the offset grows. | [Indexes and Query Optimization](../databases/sql/04-indexes-and-query-optimization.md) |

## Live-coding / whiteboard tips

- When asked to write a query, state your assumptions about the schema
  (nullable columns, one-to-many vs. many-to-many) before writing SQL.
- For any join, be ready to explain what happens to unmatched rows on
  each side — this is one of the most common follow-up questions.
- If asked to optimize a query, mention `EXPLAIN ANALYZE` as your first
  diagnostic step before proposing any specific fix.

## Common red flags interviewers watch for

- Confidently proposing an index without being able to explain what
  column(s) it should cover and why.
- Not knowing PostgreSQL's default isolation level (`READ COMMITTED`) or
  what anomalies it does/doesn't prevent.
- Suggesting `OFFSET`-based pagination as the default for large,
  performance-sensitive result sets.
- Treating application-level validation as equivalent to a database
  constraint.

## Related deep-dive material

- [SQL section overview](../databases/sql/index.md) — 4 topic pages.
- [PostgreSQL section overview](../databases/postgresql/index.md) — 3
  topic pages.
