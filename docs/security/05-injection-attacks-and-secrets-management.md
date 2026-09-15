# Injection Attacks and Secrets Management

## What

**SQL injection** is an attack where untrusted input is interpreted as
part of a SQL query rather than as data, letting an attacker read,
modify, or delete data outside their intended access. **Secrets
management** is how an application stores and accesses sensitive
configuration (database passwords, API keys, signing keys) without
exposing them in source code or logs.

## Why

Injection attacks remain one of the most damaging and most preventable
classes of vulnerability — the fix (parameterized queries) is
well-established and essentially free, yet string-concatenated SQL still
appears in real codebases. Secrets leaking into source control or logs is
an equally common, equally preventable failure mode with severe
consequences (a leaked database credential or signing key can compromise
an entire system).

## How

### SQL injection: the vulnerability

```python
# VULNERABLE: user input concatenated directly into SQL
username = request.query_params["username"]
query = f"SELECT * FROM users WHERE username = '{username}'"
cursor.execute(query)
```

```
# Attacker supplies: ' OR '1'='1
# Resulting query:
SELECT * FROM users WHERE username = '' OR '1'='1'
# Returns ALL users, bypassing the intended filter entirely
```

A more damaging payload (`'; DROP TABLE users; --`) can destroy data
outright, since the database has no way to distinguish "data that looks
like SQL" from actual SQL once it's concatenated into the query string.

### The fix: parameterized queries

```python
# SAFE: the driver treats the parameter as data, never as SQL syntax
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
```

```python
# SAFE with SQLAlchemy's query builder (see SQL Querying Fundamentals)
from sqlalchemy import select
stmt = select(User).where(User.username == username)
session.execute(stmt)
```

Parameterized queries send the query structure and the data separately to
the database — the driver ensures a parameter is always treated as a
literal value, never as executable SQL syntax, no matter what characters
it contains. This is the single fix that eliminates SQL injection almost
entirely; see
[Querying Fundamentals](../databases/sql/01-querying-fundamentals.md) for
the underlying SQL basics this protects.

### ORMs don't automatically guarantee safety

```python
# STILL VULNERABLE: raw SQL string built with an ORM's execute() escape hatch
session.execute(f"SELECT * FROM users WHERE username = '{username}'")

# SAFE: use the ORM's query-building API, or parameterize raw SQL explicitly
session.execute(text("SELECT * FROM users WHERE username = :username"), {"username": username})
```

Using an ORM (see
[ORM Models and Sessions](../databases/sqlalchemy/01-orm-models-and-sessions.md))
substantially reduces risk because its query-building API parameterizes
by default, but any raw-SQL escape hatch (`text()`, `.execute()` with a
manually built string) reintroduces the exact same risk if input is
concatenated rather than bound as a parameter.

### Other injection-adjacent risks

```python
# Command injection -- similarly dangerous, same root cause
os.system(f"convert {user_supplied_filename} output.png")  # NEVER do this
subprocess.run(["convert", user_supplied_filename, "output.png"])  # safer: no shell parsing
```

The same principle (never let untrusted input be interpreted as syntax/
commands rather than pure data) applies beyond SQL — command injection via
`os.system`/shell=True, and template injection in server-side templating
engines, share the identical root cause and identical fix pattern
(separate code/structure from data).

### Input validation as defense in depth

```python
class UsernameQuery(BaseModel):
    username: str = Field(max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
```

Strict input validation (see
[Pydantic and Validation](../fastapi/02-pydantic-and-validation.md)) is a
complementary, not a substitute, defense — it narrows what values reach
your code at all, but parameterized queries remain the actual fix for SQL
injection; validation alone (e.g. blocklisting quote characters) is
fragile and easy to bypass.

### Secrets: never in source code

```python
# NEVER do this
SECRET_KEY = "sk_live_abc123..."
DATABASE_URL = "postgresql://user:realpassword@prod-host/db"
```

```python
# Load from the environment instead
import os
SECRET_KEY = os.environ["SECRET_KEY"]
```

A secret committed to git remains in the repository's history forever,
even if later removed in a subsequent commit — rotating the leaked
secret (not just deleting the line) is the only real remediation once
committed.

### Environment variables and `.env` files (local development)

```python
# .env (in .gitignore, never committed)
DATABASE_URL=postgresql://user:pass@localhost/dev_db
SECRET_KEY=dev-only-not-a-real-secret
```

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    secret_key: str

    class Config:
        env_file = ".env"

settings = Settings()
```

`.env` files are convenient for local development but are still plaintext
on disk — fine for local dev secrets that don't matter in production, not
a substitute for a real secrets manager in deployed environments.

### Secrets managers for production

```
AWS Secrets Manager / Parameter Store
HashiCorp Vault
Google Secret Manager / Azure Key Vault
Kubernetes Secrets (base64-encoded, not encrypted at rest by default --
    typically paired with a KMS-backed encryption provider)
```

A dedicated secrets manager provides access control (who/what can read a
given secret), audit logging (who accessed it and when), and rotation
support — capabilities a `.env` file or a plain environment variable in a
deployment manifest doesn't give you.

### Secret rotation

```python
# Support two valid keys during a rotation window
def verify_token(token: str) -> dict:
    for key in [CURRENT_SECRET_KEY, PREVIOUS_SECRET_KEY]:
        try:
            return jwt.decode(token, key, algorithms=["HS256"])
        except jwt.InvalidSignatureError:
            continue
    raise HTTPException(status_code=401, detail="Invalid token")
```

Rotating a secret that's actively used to sign long-lived tokens requires
a transition period accepting both the old and new key, otherwise every
token issued before rotation becomes invalid the instant the old key is
retired.

### Avoiding secrets in logs

```python
# BAD: logs the full request, including an Authorization header or password field
logger.info(f"Request: {request.headers}, body: {body}")

# GOOD: redact sensitive fields before logging
logger.info("Request", extra={"path": request.url.path, "user_id": user.id})
```

Verbose request/response logging is a common, easy-to-miss way secrets
and credentials end up in log aggregation systems — logs often have wider
access and weaker access controls than the production database itself.

## When to use

- Parameterized queries (or an ORM's query-building API) for every query
  that includes any input not fully controlled by your own code.
- A dedicated secrets manager for any production credential — environment
  variables sourced from it at deploy/startup time, never hardcoded.
- Explicit secret rotation support (accepting old + new key during a
  transition window) for any long-lived signing key.

## When NOT to use

- Don't use string formatting/concatenation to build a SQL query from any
  input that isn't a fixed, hardcoded literal.
- Don't rely on `.env` files or plain environment variables as the sole
  secrets mechanism in a production deployment — use them for local
  development only.
- Don't log full request/response payloads indiscriminately in a
  production system without redacting sensitive fields first.

## Common mistakes

- Using an ORM's raw-SQL escape hatch (`text()`, `.execute()`) with
  string-concatenated input, silently reintroducing SQL injection despite
  "using an ORM."
- Committing a secret to git and considering it resolved after simply
  removing it in a later commit — the secret remains in history and must
  be rotated.
- Logging full request objects/headers, inadvertently capturing
  `Authorization` headers or password fields in a log aggregation system.
- No rotation plan for a signing key, making key rotation an emergency,
  all-or-nothing operation instead of a routine, gradual one.

## Interview questions

1. Walk through exactly why string-concatenated SQL is exploitable and
   why a parameterized query fixes it at the driver level, not just by
   convention.
2. Does using an ORM automatically prevent SQL injection? Explain the
   caveat.
3. Why is deleting a committed secret in a later git commit insufficient
   remediation?
4. What capabilities does a dedicated secrets manager provide over a
   plain environment variable or `.env` file?
5. Why does rotating a signing key require a transition window accepting
   both the old and new key?

## Senior-level considerations

- SQL injection prevention should be enforced structurally (a linter rule
  or code review checklist item flagging any raw SQL string
  concatenation) rather than relying on every individual engineer
  remembering to parameterize — process, not just knowledge, prevents
  this class of bug at scale.
- Secrets management maturity (dedicated secrets manager, automated
  rotation, least-privilege access to secrets themselves) is one of the
  clearest differentiators between an early-stage system and a
  production-hardened one — it's usually retrofitted under pressure after
  an incident rather than designed in from the start, which is itself a
  lesson worth internalizing.
- Defense in depth matters here too: parameterized queries plus input
  validation plus least-privilege database credentials (see
  [Authorization Models and Least Privilege](03-authorization-and-least-privilege.md))
  mean that even if one layer fails, the blast radius of an injection
  attempt is still contained.
