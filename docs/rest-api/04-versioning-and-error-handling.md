# API Versioning and Error Handling

## What

**API versioning** is how an API evolves without breaking existing
clients. **Error handling** design is how an API communicates failures in
a consistent, predictable shape. Both are contract-level concerns — how an
API behaves over time and under failure — as opposed to what a single
successful request/response looks like.

## Why

APIs change: fields get added/removed, behavior shifts, bugs get fixed in
ways that alter responses. Without a deliberate versioning strategy,
"internal" changes silently break external clients. Without a consistent
error contract, every client has to guess how to detect and parse failures
per-endpoint, which doesn't scale as an API grows.

## How

### What counts as a breaking change

| Change | Breaking? |
|---|---|
| Adding a new optional field to a response | No |
| Adding a new required field to a request | Yes |
| Removing a field from a response | Yes |
| Renaming a field | Yes |
| Changing a field's type (`string` → `int`) | Yes |
| Adding a new endpoint | No |
| Changing a status code for an existing scenario | Yes |
| Adding a new enum value a client doesn't expect | Often yes, in practice (clients that don't handle unknown values gracefully) |

A useful discipline: clients should be built to tolerate unknown fields and
unknown enum values (forward compatibility), but the API should still
version deliberately for anything a reasonable client couldn't be expected
to tolerate.

### Versioning strategies

```
# URL path versioning (most common, most explicit)
GET /v1/orders
GET /v2/orders

# Header versioning
GET /orders
Accept: application/vnd.myapi.v2+json

# Query parameter versioning
GET /orders?version=2
```

| Strategy | Pros | Cons |
|---|---|---|
| URL path (`/v1/...`) | Explicit, cacheable, easy to route | "Pollutes" the URL; implies the whole resource changed even for small changes |
| Header (`Accept`/custom) | Keeps URLs clean, resource identity stable | Less visible/discoverable; harder to test manually (e.g. in a browser) |
| Query parameter | Simple to add | Easy to forget/default incorrectly; less conventional |

URL path versioning is the most common in practice for public APIs because
it's explicit and simple to route/cache; header-based versioning is more
"RESTfully pure" (the resource URL doesn't change, only its representation)
but adds friction for API consumers.

### Versioning scope: whole API vs per-resource

Most real APIs version the whole API at once (`/v1`, `/v2`) rather than
each resource independently — independent per-resource versioning is
technically more flexible but usually adds more operational complexity
than it's worth for typical backend services.

### Deprecation policy

```http
GET /v1/orders
Deprecation: true
Sunset: Sat, 31 Dec 2025 23:59:59 GMT
Link: <https://api.example.com/v2/orders>; rel="successor-version"
```

Announce deprecations with a clear timeline (`Sunset` header, changelog,
advance notice) rather than removing an old version abruptly — breaking
existing integrations without warning damages trust with API consumers.

### A consistent error response shape

```json
{
  "error": {
    "code": "user_not_found",
    "message": "User 42 not found",
    "details": []
  }
}
```

A stable, machine-parseable `code` (not just a human-readable `message`)
lets clients handle specific error conditions programmatically without
string-matching a message that might change wording later. This builds
directly on the exception-handling pattern in
[Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md)
— the framework-level mechanism (a catch-all handler mapping domain
exceptions to responses) is how this consistent shape actually gets
produced.

### Validation error details

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request failed validation",
    "details": [
      {"field": "email", "message": "not a valid email address"},
      {"field": "age", "message": "must be >= 0"}
    ]
  }
}
```

Field-level detail in validation errors lets clients (especially UI forms)
show specific, actionable feedback rather than a single generic message.

### Idempotent error responses for retries

Combine with the idempotency-key pattern from
[REST Principles and Idempotency](02-rest-principles-and-idempotency.md):
if a client retries a request that already succeeded, the API should
return the original success response (or a clear "already processed"
response), not a confusing new error.

## When to use

- Version the whole API via URL path (`/v1/...`) for public/external APIs
  where simplicity and cacheability matter most.
- Use a stable, documented error `code` field, separate from the
  human-readable `message`, so clients can branch on error type reliably.
- Announce deprecations with a `Sunset` header/timeline well before
  removing an old API version.
- Version only when a change is genuinely breaking — additive, backward-
  compatible changes don't need a new version.

## When NOT to use

- Don't version an API for every trivial change — this fragments the API
  surface and burdens clients with unnecessary migrations.
- Don't rely on HTTP status code alone to convey error meaning when
  multiple distinct failure reasons map to the same status code (e.g.
  several different `400` scenarios) — a machine-readable error code adds
  the missing precision.
- Don't silently remove or repurpose an old API version without a
  deprecation window — this breaks integrations without warning.

## Common mistakes

- Treating "adding a field" the same as "removing/renaming a field" —
  only the latter is reliably breaking for well-behaved clients.
- Returning inconsistent error shapes across different endpoints (one
  returns `{"error": "..."}`, another `{"message": "..."}`, another a bare
  string) — forces clients to special-case error parsing per endpoint.
- Bumping the whole API's major version for a change that only affects one
  endpoint, forcing unrelated clients to migrate unnecessarily.
- Removing a deprecated API version without adequate notice, breaking
  integrations that had no reasonable way to know the deadline.

## Interview questions

- What counts as a breaking API change, and what doesn't?

    **Answer:** Breaking = anything an existing client's code could choke
    on: removing/renaming a field, changing a field's type, making an
    optional field required, changing a status code. Non-breaking =
    additive stuff, like adding a new optional field or a new endpoint,
    that old clients simply ignore.

- Compare URL path versioning, header versioning, and query parameter
    versioning. Which would you choose for a public API, and why?

    **Answer:** Path (`/v1/users`) is the most visible/discoverable and
    easiest to route/cache; header (`Accept: application/vnd.api.v1+json`)
    is "cleaner" REST-wise but harder to test/debug in a browser; query
    param (`?version=1`) is the least common, easy to forget to send. For
    a public API, path versioning is usually the pragmatic choice —
    obvious to clients and trivial to route.

- Why should an error response include a machine-readable `code` in
    addition to a human-readable `message`?

    **Answer:** Clients need to branch on errors programmatically (retry?
    show a specific UI message? log and alert?). A stable `code` like
    `"insufficient_funds"` is safe to `if`-check; a human `message` can
    change wording anytime without breaking client logic.

- How would you communicate and enforce a deprecation timeline for an old
    API version?

    **Answer:** Announce it in docs/changelog with a firm sunset date,
    return a `Deprecation`/`Sunset` header on old-version responses, and
    monitor usage so you know when it's actually safe to remove.

- Why is a consistent error response shape important across an entire
    API surface?

    **Answer:** If every endpoint's error looks different, client error
    handling can't be written generically — one team has to write bespoke
    parsing per endpoint instead of one shared error handler.

## Senior-level considerations

- Versioning strategy is a long-term commitment — once external clients
  depend on `/v1`, supporting it (or planning its deprecation) becomes an
  ongoing operational cost; factor this into the decision rather than
  treating versioning as a one-time technical choice. For example, a team
  that ships `/v1` and `/v2` may end up running both code paths in
  production for years while waiting for clients to migrate.
- A well-designed error contract (stable codes, field-level validation
  detail) reduces support burden significantly — client teams can build
  reliable, automated error handling instead of ad hoc message parsing.
  For example, `{"code": "validation_error", "fields": {"email": "invalid
  format"}}` lets a frontend highlight the exact field without regexing a
  message string.
- Coordinating API versioning across a large organization (multiple teams
  owning different services) benefits from a shared, written convention
  (which versioning strategy, which error shape) rather than each team
  inventing its own — inconsistency here compounds as the number of
  internal API consumers grows. For example, a company-wide "API
  standards" doc specifying path versioning + a shared error envelope
  saves every new service from re-litigating the same decisions.
