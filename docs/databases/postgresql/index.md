# PostgreSQL

Notes on PostgreSQL as a production database engine: schema design,
transaction guarantees, concurrency control, and operational performance
concerns. Builds on [SQL](../sql/index.md) — this section focuses on
engine behavior and guarantees, not query syntax.

- [Database Design and Constraints](01-database-design-and-constraints.md) — schema design, keys, constraints, normalization
- [Transactions and ACID](02-transactions-and-acid.md) — transactions, ACID, isolation levels
- [Locks, Deadlocks, and Connection Pooling](03-locks-deadlocks-and-connection-pooling.md) — concurrency control and performance
