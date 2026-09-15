# SQLAlchemy

Notes on SQLAlchemy as the Python ORM/toolkit layer over
[PostgreSQL](../postgresql/index.md), covering the 2.x-style API used in
current FastAPI applications.

- [ORM Models and Sessions](01-orm-models-and-sessions.md) — declarative models, sessions
- [Queries, Relationships, and Loading](02-queries-relationships-and-loading.md) — querying, relationships, lazy vs eager loading
- [Transactions and Connection Pools](03-transactions-and-connection-pools.md) — session transaction boundaries, pooling
- [Async SQLAlchemy and Migrations](04-async-sqlalchemy-and-migrations.md) — async sessions, Alembic
