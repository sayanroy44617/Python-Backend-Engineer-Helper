# Authorization Models and Least Privilege

## What

**Authorization** determines what an authenticated identity is permitted
to do. Common models include **RBAC** (Role-Based Access Control) and
**ABAC** (Attribute-Based Access Control). **Least privilege** is the
principle that any identity (user, service, token) should hold only the
permissions it actually needs, no more.

## Why

Authentication alone ("who are you") says nothing about what that
identity should be allowed to do. Without a deliberate authorization
model, permission checks tend to sprawl as ad-hoc `if user.is_admin`
conditions scattered through the codebase — hard to audit and easy to get
wrong. Least privilege limits the blast radius when a credential is
compromised or a bug grants unintended access.

## How

### RBAC: role-based access control

```python
class Role(str, Enum):
    VIEWER = "viewer"
    EDITOR = "editor"
    ADMIN = "admin"

ROLE_PERMISSIONS = {
    Role.VIEWER: {"read"},
    Role.EDITOR: {"read", "write"},
    Role.ADMIN: {"read", "write", "delete"},
}

def has_permission(user: User, permission: str) -> bool:
    return permission in ROLE_PERMISSIONS.get(user.role, set())
```

RBAC groups permissions into named roles and assigns users to roles — the
model most systems start with because it's simple to reason about
("editors can write, admins can also delete").

### ABAC: attribute-based access control

```python
def can_edit_document(user: User, document: Document) -> bool:
    return (
        document.owner_id == user.id
        or user.department == document.department
        or user.role == Role.ADMIN
    )
```

ABAC evaluates a policy against *attributes* of the user, the resource,
and sometimes the environment (time of day, IP range) — more flexible
than RBAC for rules like "you can edit documents in your own department"
that a fixed role hierarchy can't express cleanly.

### RBAC vs ABAC

| | RBAC | ABAC |
|---|---|---|
| Model | Named roles → fixed permission sets | Policies evaluated against attributes |
| Simplicity | Easier to reason about and audit | More flexible, more complex to reason about |
| Fits well | "Admins can do X, editors can do Y" | "You can edit your own resources", "only during business hours" |
| Common combination | Use RBAC as the default, add ABAC-style resource-ownership checks where roles alone aren't precise enough | |

Most real systems use a hybrid: RBAC for coarse-grained access (what
screens/endpoints exist for this role) plus attribute checks
(resource ownership, tenant isolation) for fine-grained per-resource
decisions.

### Enforcing authorization as a dependency (FastAPI)

```python
def require_permission(permission: str):
    def dependency(user: User = Depends(get_current_user)) -> User:
        if not has_permission(user, permission):
            raise HTTPException(status_code=403, detail="Insufficient permissions")
        return user
    return dependency

@app.delete("/documents/{doc_id}")
def delete_document(doc_id: int, user: User = Depends(require_permission("delete"))):
    ...
```

Same dependency-factory pattern from
[FastAPI: Authentication and Authorization](../fastapi/05-authentication-and-authorization.md)
— authorization checks belong at this explicit, auditable layer rather
than scattered through business logic.

### Resource-level authorization (ownership checks)

```python
def get_document_for_user(session: Session, doc_id: int, user: User) -> Document:
    document = session.get(Document, doc_id)
    if document is None:
        raise HTTPException(status_code=404, detail="Not found")
    if document.owner_id != user.id and user.role != Role.ADMIN:
        raise HTTPException(status_code=403, detail="Not permitted")
    return document
```

A role check alone ("is this user an editor") isn't enough for
multi-tenant or per-resource systems — you also need to check that this
*specific* resource belongs to (or is shared with) the requesting user,
not just that their role generally permits the action.

### Least privilege for service accounts and API keys

```python
# BAD: one service account with full database admin rights, used everywhere
DATABASE_URL = "postgresql://admin:pass@host/db"

# GOOD: scoped credentials per service, matching what it actually needs
DATABASE_URL = "postgresql://reporting_readonly:pass@host/db"  # for a reporting service
```

A reporting service that only ever runs `SELECT` queries should connect
with a database role that can only `SELECT` — if that service is
compromised, the attacker inherits only the reporting service's limited
permissions, not full database control.

### Least privilege for OAuth2 scopes and API tokens

```python
token = create_token(user_id=123, scopes=["orders:read"])  # not "orders:write" or "admin"
```

A third-party integration that only needs to read a user's order history
should be issued a token scoped to exactly that — not a general-purpose
token carrying every permission the user has (see
[JWT, OAuth2, and OIDC](02-jwt-oauth2-and-oidc.md) for scope mechanics).

### Auditing authorization decisions

```python
logger.info("authz_check", user_id=user.id, permission="delete", resource_id=doc_id, allowed=allowed)
```

Logging authorization decisions (especially denials, and especially for
sensitive operations) gives you an audit trail for security reviews and
incident investigation — "who tried to access what, and were they
allowed" is a question you want answered from logs, not reconstructed
after the fact.

## When to use

- RBAC as the default model for most applications — it's simple, auditable,
  and matches how most organizations already think about access ("admins",
  "editors", "viewers").
- ABAC-style attribute/ownership checks layered on top of RBAC wherever a
  role alone isn't precise enough (per-resource ownership, multi-tenancy,
  department-scoped access).
- Least-privilege scoped credentials for every service account, API key,
  and OAuth2 token — never a single all-powerful credential reused
  everywhere.

## When NOT to use

- Don't build a full custom policy engine (a general ABAC system) for a
  simple application where RBAC alone covers every real requirement —
  added complexity should be justified by an actual need for it.
- Don't perform authorization checks only at the UI layer — the API
  itself must always enforce access control server-side, since any
  client-side check can be bypassed entirely.
- Don't grant broad "just in case" permissions to a service account or
  token to avoid future access-denied errors — expand scope deliberately
  when a real need arises, not preemptively.

## Common mistakes

- Checking only role, not resource ownership — an "editor" role check
  might wrongly let a user edit another user's private document.
- Authorization logic duplicated inconsistently across multiple endpoints
  instead of centralized in a shared dependency/policy function.
- Service accounts and database users provisioned with far broader
  permissions than the service actually needs, "to avoid future issues."
- No audit logging of authorization decisions, making it impossible to
  reconstruct what happened after a suspected access-control incident.

## Interview questions

1. What's the difference between RBAC and ABAC, and when would you reach
   for each?
2. Why is a role check alone often insufficient for multi-tenant or
   per-resource authorization?
3. What does the principle of least privilege mean in practice for a
   service account's database credentials?
4. Where should authorization checks be enforced, and why is a client-side
   (UI) check never sufficient on its own?
5. Why is auditing authorization decisions (especially denials) valuable
   even in a system that's otherwise working correctly?

## Senior-level considerations

- Authorization model choice (RBAC vs ABAC vs hybrid) should match the
  actual complexity of the domain's access rules — over-engineering a
  full policy engine for simple role-based needs adds unnecessary
  operational and cognitive overhead.
- Least privilege is an ongoing discipline, not a one-time setup — access
  reviews (periodically checking that granted permissions still match
  actual need) prevent permission creep as an organization and its
  systems evolve.
- Centralizing authorization logic (shared dependencies/policy functions,
  audit logging) pays off most clearly during incident response — being
  able to quickly answer "could this compromised credential have accessed
  X" depends entirely on how consistently and legibly authorization was
  implemented beforehand.
