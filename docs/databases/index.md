# Databases

Notes covering SQL fundamentals, PostgreSQL as a production database
engine, and SQLAlchemy as the Python ORM layer on top of it.

- [SQL](sql/index.md) — query language fundamentals, joins, subqueries, indexes
- [PostgreSQL](postgresql/index.md) — schema design, transactions, ACID, locking, performance
- [SQLAlchemy](sqlalchemy/index.md) — ORM models, sessions, relationships, migrations

These build on each other: SQL is the language, PostgreSQL is the engine
executing it with specific guarantees and behavior, and SQLAlchemy is the
Python abstraction layer most FastAPI services use to talk to it (see
[FastAPI Dependency Injection](../fastapi/03-dependency-injection.md) for
how a DB session is wired into a request).
