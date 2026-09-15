# Authentication and Password Hashing

## What

**Authentication** is the process of verifying an identity claim — proving
a request really comes from the user/client it claims to. **Password
hashing** is how a system stores a credential such that it can verify a
password without ever storing (or being able to recover) the original.

## Why

If credentials are compromised (a database leak, a log file, a stolen
token), the blast radius depends entirely on how they were stored/issued.
A leaked password hash (with a strong algorithm) is not directly usable
by an attacker; a leaked plaintext password is catastrophic and often
reused across other services by the same user.

## How

### Never store plaintext passwords

```python
# NEVER do this
user.password = "hunter2"

# Store a hash instead
import bcrypt

hashed = bcrypt.hashpw(b"hunter2", bcrypt.gensalt())
user.password_hash = hashed
```

Even internal/admin-only systems should never store plaintext passwords —
database backups, logs, and internal access all become a source of
credential leakage otherwise.

### Verifying a password

```python
def verify_password(plain_password: str, password_hash: bytes) -> bool:
    return bcrypt.checkpw(plain_password.encode(), password_hash)
```

Verification re-hashes the *input* with the same salt (embedded in the
stored hash) and compares — the plaintext password is never stored or
needed again after the initial hash is computed.

### Why hashing algorithms matter: bcrypt/argon2 vs SHA-256

```python
# WRONG: a general-purpose fast hash is not suitable for passwords
import hashlib
password_hash = hashlib.sha256(b"hunter2").hexdigest()  # fast = bad here

# RIGHT: a deliberately slow, salted password-hashing algorithm
import bcrypt
password_hash = bcrypt.hashpw(b"hunter2", bcrypt.gensalt())
```

General-purpose hashes (`SHA-256`, `MD5`) are designed to be *fast* —
exactly the wrong property for password storage, since it makes brute-
forcing billions of guesses (using GPUs/ASICs) cheap. Password-hashing
algorithms (`bcrypt`, `scrypt`, `argon2`) are deliberately slow and
tunable (a "cost factor"), and automatically salt each hash.

### Salting (built into bcrypt/argon2 automatically)

```python
bcrypt.hashpw(b"hunter2", bcrypt.gensalt())  # different output every time
bcrypt.hashpw(b"hunter2", bcrypt.gensalt())  # even for the same password
```

A **salt** is random data mixed into the hash so that identical passwords
produce different hashes — this defeats precomputed "rainbow table"
attacks and stops an attacker from spotting that two users share a
password just by comparing hashes.

### `passlib` for algorithm-agnostic hashing

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

hashed = pwd_context.hash("hunter2")
pwd_context.verify("hunter2", hashed)  # True
```

`passlib` wraps multiple algorithms behind one interface and supports
graceful migration (`deprecated="auto"` re-hashes with the current scheme
on next successful login if a user's hash uses an older/weaker one) —
useful when a project needs to upgrade its hashing algorithm over time.

### Session-based vs token-based authentication

```
Session-based:
  Login -> server creates a session record (in DB/Redis) -> session ID
  set in a cookie -> every request looks up the session server-side.

Token-based (e.g. JWT):
  Login -> server issues a signed token containing claims -> client sends
  it in the Authorization header -> server verifies the signature,
  no server-side lookup needed.
```

Session-based auth is simple to revoke (delete the session record) but
requires server-side state (or a shared store across instances).
Token-based auth (see
[JWT, OAuth2, and OIDC](02-jwt-oauth2-and-oidc.md)) is stateless and scales
horizontally without shared session storage, but revoking a single token
before its expiry is inherently harder.

### Multi-factor authentication (MFA)

```
Something you know  -- password
Something you have  -- TOTP app, hardware key, SMS code
Something you are   -- biometric
```

MFA requires two or more of these categories — a leaked password alone is
no longer sufficient to authenticate, which is why it's one of the
highest-value security controls a system can add for user accounts.

### Rate limiting and lockout on login attempts

```python
if get_failed_login_count(username) >= 5:
    raise HTTPException(status_code=429, detail="Too many attempts, try again later")
```

Without this, an attacker can brute-force passwords by simply trying many
combinations — see
[Rate Limiting and API Security](../rest-api/05-rate-limiting-and-api-security.md)
for the general rate-limiting mechanics; login endpoints are exactly the
"sensitive unauthenticated endpoint" case called out there.

## When to use

- A dedicated password-hashing algorithm (`bcrypt`, `argon2`) — never a
  general-purpose fast hash — for any stored credential.
- Token-based auth for stateless APIs consumed by multiple clients/
  services; session-based auth when simple, immediate revocation matters
  more than statelessness (e.g. a traditional server-rendered web app).
- MFA for any account with access to sensitive data or privileged actions.

## When NOT to use

- Don't design or implement a custom cryptographic hashing/encryption
  scheme — use well-reviewed, standard libraries and algorithms.
- Don't use fast general-purpose hashes (`MD5`, `SHA-256` alone) for
  passwords, even "just for an internal tool" — the habit and the risk
  both persist.
- Don't rely on session-based auth without a shared session store (Redis,
  a database) once the application runs on more than one instance.

## Common mistakes

- Hashing passwords with a fast, general-purpose algorithm instead of a
  purpose-built slow one.
- Rolling a custom salt/hash scheme instead of using `bcrypt`/`argon2`/
  `passlib`, which handle salting and algorithm parameters correctly by
  default.
- Comparing password hashes with a non-constant-time comparison
  (`==` on raw bytes can leak timing information) — always use the
  library's own `verify`/`checkpw` function, which is constant-time.
- No rate limiting/lockout on login endpoints, leaving them open to
  brute-force attacks.

## Interview questions

1. Why is a fast hash like SHA-256 a poor choice for storing passwords,
   even though it's cryptographically secure in other contexts?
2. What does a salt protect against, and how is it typically stored
   alongside the hash?
3. Compare session-based and token-based authentication: what are the
   trade-offs for revocation and horizontal scaling?
4. Why is `passlib`'s `deprecated="auto"` useful for a system that's been
   running for years?
5. What are the three categories of authentication factors, and why does
   combining two matter more than either alone?

## Senior-level considerations

- Authentication design decisions (session vs token, MFA requirements,
  password policy) have direct incident-response implications — knowing
  how to revoke access quickly across an entire system during a
  suspected breach is a core operational concern, not just a design-time
  one.
- Migrating a password-hashing algorithm across a live user base (e.g.
  moving from an older bcrypt cost factor to argon2) has to happen
  gradually, re-hashing on next successful login rather than forcing a
  mass password reset — planning for this migration path from day one
  avoids a painful later transition.
- Credential storage and authentication are consistently among the
  highest-value targets in a real system; treating them with
  disproportionate rigor (dedicated libraries, security review, no
  custom crypto) relative to other application code is expected of a
  senior engineer.
