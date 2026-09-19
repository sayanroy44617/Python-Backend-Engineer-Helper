# Authentication and Authorization

## What

**Authentication** verifies *who* is calling the API (login, token
validation). **Authorization** verifies *what* an authenticated caller is
allowed to do. FastAPI provides building blocks (`OAuth2PasswordBearer`,
security schemes) but leaves the actual auth logic and storage to you —
this topic covers the patterns used to wire them together.

This is the FastAPI-specific plumbing; broader auth concepts (JWT, OAuth2,
password hashing, OIDC) belong in the dedicated **Security** section — link
there for the deeper "why"/protocol details.

## Why

Almost every backend service needs to restrict access to some or all
endpoints. FastAPI's dependency injection system (see
[Dependency Injection](03-dependency-injection.md)) is the natural
mechanism for this: auth becomes "just another dependency" that routes
declare, rather than bespoke per-route logic.

## How

### Token extraction with `OAuth2PasswordBearer`

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.get("/me")
def read_current_user(token: str = Depends(oauth2_scheme)):
    return {"token": token}
```

`OAuth2PasswordBearer` is a dependency that extracts the bearer token from
the `Authorization` header and also tells FastAPI's OpenAPI schema that
this endpoint requires auth (shows an "Authorize" button in the docs UI).
It does **not** validate the token itself — that's your responsibility.

### A login endpoint issuing a JWT

```python
from fastapi.security import OAuth2PasswordRequestForm
from datetime import datetime, timedelta, timezone
import jwt

SECRET_KEY = "..."  # load from settings/secrets, never hardcode

@app.post("/token")
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if user is None:
        raise HTTPException(status_code=401, detail="Incorrect username or password")

    expire = datetime.now(timezone.utc) + timedelta(minutes=30)
    token = jwt.encode({"sub": user.id, "exp": expire}, SECRET_KEY, algorithm="HS256")
    return {"access_token": token, "token_type": "bearer"}
```

`OAuth2PasswordRequestForm` parses the standard `username`/`password` form
fields OAuth2's password flow expects.

### Validating the token and resolving the current user

```python
def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="Invalid or expired token")

    user = get_user_by_id(payload["sub"])
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    return user

@app.get("/me")
def read_current_user(user: User = Depends(get_current_user)):
    return user
```

`get_current_user` is a dependency that every protected route can require
— it centralizes token validation once instead of repeating it.

### Authorization: role/permission checks

```python
def require_role(role: str):
    def dependency(user: User = Depends(get_current_user)) -> User:
        if role not in user.roles:
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return dependency

@app.delete("/users/{user_id}")
def delete_user(user_id: int, admin: User = Depends(require_role("admin"))):
    ...
```

`require_role` is a dependency **factory** (compare to the decorator
factory pattern in
[Decorators and Closures](../python/09-decorators-and-closures.md)) — it
builds a parameterized dependency, letting you express "any admin" vs "any
authenticated user" declaratively per route.

### Distinguishing 401 vs 403

| Status | Meaning | Example |
|---|---|---|
| `401 Unauthorized` | Not authenticated (missing/invalid credentials) | Missing or expired token |
| `403 Forbidden` | Authenticated, but not permitted | Valid user token, but not an admin |

Conflating these confuses API clients about whether to re-authenticate
(401) or that access is simply denied regardless of re-login (403).

### Scopes (fine-grained permissions)

```python
from fastapi.security import SecurityScopes

def get_current_user(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme),
) -> User:
    payload = decode_token(token)
    user = get_user_by_id(payload["sub"])
    for scope in security_scopes.scopes:
        if scope not in payload.get("scopes", []):
            raise HTTPException(status_code=403, detail="Not enough permissions")
    return user
```

OAuth2 scopes let a single token carry a specific, narrower set of
permissions than "everything this user can do" — useful for third-party
integrations issued limited-scope tokens.

## When to use

- Extract and validate tokens via a dependency (`get_current_user`) shared
  across every protected route, rather than duplicating token-parsing
  logic.
- Use dependency factories (`require_role(...)`) for reusable, parameterized
  authorization checks.
- Return `401` for missing/invalid credentials, `403` for valid credentials
  lacking permission — keep these semantically distinct.

## When NOT to use

- Don't implement your own password hashing or token signing scheme from
  scratch — use established libraries (`passlib`/`bcrypt`, `PyJWT`) and
  see [Authentication and Password Hashing](../security/01-authentication-and-password-hashing.md)
  for the underlying cryptographic concerns.
- Don't perform authorization checks inline scattered across business
  logic — express them as dependencies so they're visible directly in a
  route's signature and easy to audit.
- Don't put long-lived secrets in source code or version control — load
  `SECRET_KEY`/similar from environment variables or a secrets manager.

## Common mistakes

- Returning `403` when a token is simply missing/expired (should be `401`),
  confusing clients about whether re-authentication would help.
- Trusting a token's claims without verifying its signature/expiry — always
  go through a proper `jwt.decode()` (or equivalent) with signature
  verification, never manually parse the payload.
- Forgetting that `OAuth2PasswordBearer` only *extracts* the token — it
  doesn't validate it; validation must happen in your own dependency.
- Applying authorization checks only in the API layer while a background
  job or internal script bypasses them entirely, allowing an unintended
  path to privileged operations.

## Interview questions

- What's the difference between authentication and authorization? Give a
    401 vs 403 example.

    **Answer:** Authentication is "who are you?"; authorization is "are you allowed to do this?". Return `401` when the token is missing or bad, and `403` when the token is valid but the user still lacks the required permission.

    ```python
    from fastapi import HTTPException

    def require_admin(is_authenticated: bool, is_admin: bool) -> None:
       if not is_authenticated:
           raise HTTPException(status_code=401, detail="Missing token")
       if not is_admin:
           raise HTTPException(status_code=403, detail="Admin role required")
    ```

- What does `OAuth2PasswordBearer` actually do, and what does it *not* do?

    **Answer:** It pulls the bearer token out of the `Authorization` header and marks the route as secured in OpenAPI. It does not verify signature, expiry, issuer, or load the user for you.

    ```python
    from fastapi import Depends
    from fastapi.security import OAuth2PasswordBearer

    oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

    def read_token(token: str = Depends(oauth2_scheme)) -> dict[str, str]:
       return {"token": token}
    ```

- How would you implement a reusable "require this role" check across
    many routes?

    **Answer:** Use a dependency factory so the route declares the required role in one place and the check stays centralized. That keeps the route signature readable and avoids copy-pasting the same `if role not in user.roles` block everywhere.

    ```python
    from fastapi import Depends, HTTPException

    def require_role(role: str):
       def dependency(user_roles: set[str] = Depends(lambda: {"admin"})) -> None:
           if role not in user_roles:
               raise HTTPException(status_code=403, detail="Forbidden")
       return dependency
    ```

- Why should token validation happen in a dependency rather than inline
    in every route?

    **Answer:** Because auth is cross-cutting plumbing, not route-specific business logic. A dependency gives you one place to validate tokens, load the user, and apply the same behavior consistently across every protected endpoint.

- What are OAuth2 scopes, and when would you use them over simple role
    checks?

    **Answer:** Scopes are narrower permissions carried by the token itself, like `users:read` or `reports:write`. Use them when one identity should get different limited tokens for different clients or integrations instead of one broad role.

    ```python
    def can_read_users(scopes: list[str]) -> bool:
       return "users:read" in scopes
    ```

## Senior-level considerations

- Centralizing auth as dependencies makes it straightforward to reason
  about which routes are protected and how — a security review can scan
  route signatures for `Depends(get_current_user)`/`Depends(require_role(...))`
  rather than auditing scattered inline checks; for example, `def read_me(user:
  User = Depends(get_current_user))` is much easier to audit than a route with
  ad hoc header parsing in the function body.
- Token validation, expiry, and revocation strategy (e.g. short-lived
  access tokens + refresh tokens, or a token blocklist) is a system design
  decision with real trade-offs between security and complexity — see the
  Security section for
  [JWT, OAuth2, and OIDC](../security/02-jwt-oauth2-and-oidc.md) depth; for
  example, a 15-minute access token plus a refresh token reduces blast radius
  but adds refresh and revocation flows.
- Authorization logic that only exists at the API boundary is a common gap
  — background jobs, admin scripts, and internal service-to-service calls
  need the same authorization guarantees, not just HTTP-layer checks; for
  example, a bulk-delete admin script should still call the same policy check
  before removing user accounts.
