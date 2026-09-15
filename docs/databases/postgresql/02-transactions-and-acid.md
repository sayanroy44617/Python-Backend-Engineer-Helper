# Transactions and ACID

## What

A **transaction** groups multiple statements so they either all succeed
(`COMMIT`) or all fail together (`ROLLBACK`). **ACID** — Atomicity,
Consistency, Isolation, Durability — is the set of guarantees a
transactional database provides. **Isolation levels** control how much
transactions running concurrently can see of each other's uncommitted or
concurrently-committed changes.

## Why

Real operations often require multiple writes that must succeed or fail
together (e.g. debit one account, credit another). Without transactions, a
crash or error partway through leaves data in an inconsistent state.
Understanding isolation levels specifically matters because the default
level in most databases still permits certain anomalies that can cause
subtle, hard-to-reproduce bugs under concurrent load.

## How

### Basic transaction

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- something goes wrong
ROLLBACK;  -- undoes the UPDATE above entirely
```

If the connection drops or the application crashes before `COMMIT`,
PostgreSQL automatically rolls back the transaction — partial writes never
persist.

### ACID guarantees

| Property | Guarantee |
|---|---|
| **Atomicity** | All statements in a transaction succeed, or none do |
| **Consistency** | A transaction moves the database from one valid state to another (constraints/triggers still hold) |
| **Isolation** | Concurrent transactions don't see each other's intermediate, uncommitted state (to a degree controlled by isolation level) |
| **Durability** | Once committed, data survives a crash (written to disk/WAL) |

### Isolation levels and the anomalies they prevent

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed (PostgreSQL default) | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Prevented (PostgreSQL implements this via snapshot isolation, stricter than the SQL standard requires) |
| Serializable | Prevented | Prevented | Prevented |

- **Dirty read**: reading another transaction's uncommitted changes.
- **Non-repeatable read**: re-reading the same row within a transaction
  gives a different result because another transaction committed a change
  in between.
- **Phantom read**: re-running the same query within a transaction returns
  a different *set* of rows because another transaction inserted/deleted
  matching rows in between.

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- e.g. 500
-- (another transaction commits a change to this row here)
SELECT balance FROM accounts WHERE id = 1;  -- still 500 -- consistent within this transaction
COMMIT;
```

### Read Committed (the default) in practice

```sql
BEGIN;  -- Read Committed by default
SELECT balance FROM accounts WHERE id = 1;  -- e.g. 500
-- another transaction commits, changing balance to 400
SELECT balance FROM accounts WHERE id = 1;  -- now reads 400 -- a non-repeatable read
COMMIT;
```

PostgreSQL's default (`Read Committed`) is sufficient for most application
code, but it does **not** prevent non-repeatable or phantom reads — worth
knowing explicitly, since it's easy to assume more consistency than the
default actually provides.

### Serializable and retryable conflicts

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... reads and writes ...
COMMIT;
-- may fail with: ERROR: could not serialize access due to concurrent update
-- application must catch this and retry the whole transaction
```

`SERIALIZABLE` gives the strongest guarantee (transactions behave as if
run one at a time), but requires application code to handle and retry
serialization failures — it's a real cost, so reserve it for operations
where correctness under concurrency genuinely requires it (e.g. transfers
between the same pair of accounts happening concurrently).

### Transactions in SQLAlchemy

```python
with Session(engine) as session:
    with session.begin():  # commits on success, rolls back on exception
        account_a.balance -= 100
        account_b.balance += 100
```

SQLAlchemy's session wraps this same transactional model — see
[Transactions and Connection Pools](../sqlalchemy/03-transactions-and-connection-pools.md)
for how it maps onto the ORM session lifecycle.

## When to use

- Wrap any set of related writes that must succeed or fail together in a
  single transaction — never issue them as independent statements when
  partial completion would leave inconsistent data.
- Use `SERIALIZABLE` (with retry logic) for operations where concurrent
  transactions could otherwise produce a subtly incorrect result that
  weaker isolation wouldn't catch (e.g. two concurrent transfers that
  together would overdraw an account).
- Keep transactions as short as possible — long-running transactions hold
  locks and resources longer than necessary (see
  [Locks, Deadlocks, and Connection Pooling](03-locks-deadlocks-and-connection-pooling.md)).

## When NOT to use

- Don't default to `SERIALIZABLE` everywhere "to be safe" — the retry
  requirement and performance cost aren't worth it for operations that
  don't actually need it; `Read Committed` (the default) is fine for most
  application code.
- Don't hold a transaction open across slow, unrelated work (e.g. an
  external API call in the middle of a DB transaction) — this extends lock
  hold time and connection usage unnecessarily.
- Don't assume a transaction protects you from application-level logic
  bugs (e.g. wrong business calculation) — ACID guarantees consistency of
  the *data operations*, not correctness of the *business logic* driving
  them.

## Common mistakes

- Assuming PostgreSQL's default isolation level (`Read Committed`)
  prevents non-repeatable reads — it doesn't; only `Repeatable Read` and
  `Serializable` do.
- Not handling serialization failures under `SERIALIZABLE` isolation —
  these are *expected*, retryable errors, not bugs.
- Leaving transactions open for long periods (e.g. waiting on user input
  or a slow external call mid-transaction), unnecessarily holding locks and
  connections.
- Forgetting that a `SELECT` alone inside a transaction still participates
  in isolation semantics — reads aren't "free" of transactional concerns
  just because they don't write.

## Interview questions

1. What do the four ACID properties each guarantee?
2. What's the difference between a dirty read, a non-repeatable read, and
   a phantom read?
3. What is PostgreSQL's default isolation level, and which anomalies does
   it still allow?
4. Why must application code be prepared to retry a transaction under
   `SERIALIZABLE` isolation?
5. Why should transactions be kept as short as possible?

## Senior-level considerations

- Isolation level choice is a deliberate trade-off between correctness
  guarantees and throughput/complexity — defaulting to the strongest level
  everywhere hurts concurrency; defaulting to the weakest everywhere risks
  subtle bugs under real concurrent load. Choose per-operation based on
  actual risk.
- Long-running transactions are a common root cause of production
  incidents (lock contention, connection pool exhaustion, replication lag)
  — a senior engineer reviewing a slow endpoint should check transaction
  duration and scope as a first step.
- Understanding isolation anomalies concretely (not just by name) is
  essential for diagnosing "impossible" bugs that only appear under
  concurrent load — these are exactly the kind of intermittent, hard-to-
  reproduce issues that isolation-level gaps produce in production.
