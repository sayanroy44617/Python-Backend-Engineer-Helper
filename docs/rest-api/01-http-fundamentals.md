# HTTP Fundamentals

## What

HTTP is the request/response protocol REST APIs are built on. This topic
covers HTTP methods, status codes, and headers — the vocabulary every API
design decision is expressed in, independent of any specific framework.

## Why

Every REST API decision (which method to use, what status code to return,
what header conveys what) is a convention built on top of HTTP semantics.
Misusing them (e.g. `200` for an error, `GET` with side effects) breaks
client expectations, caching behavior, and tooling that assumes standard
HTTP semantics.

## How

### HTTP methods and their semantics

| Method | Purpose | Safe | Idempotent | Has a body |
|---|---|---|---|---|
| `GET` | Retrieve a resource | Yes | Yes | No (by convention) |
| `POST` | Create a resource / trigger an action | No | No | Yes |
| `PUT` | Replace a resource entirely | No | Yes | Yes |
| `PATCH` | Partially update a resource | No | No (usually) | Yes |
| `DELETE` | Remove a resource | No | Yes | No (by convention) |
| `HEAD` | Like `GET`, headers only, no body | Yes | Yes | No |
| `OPTIONS` | Discover allowed methods/CORS preflight | Yes | Yes | No |

- **Safe** methods don't change server state — a client (or a crawler,
  cache, or retry mechanism) can call them freely without side effects.
- **Idempotent** methods produce the same end state no matter how many
  times they're repeated with the same input — covered in depth in
  [REST Principles and Idempotency](02-rest-principles-and-idempotency.md).

### Status code categories

| Range | Category | Example |
|---|---|---|
| `1xx` | Informational | `100 Continue` |
| `2xx` | Success | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirection | `301 Moved Permanently`, `304 Not Modified` |
| `4xx` | Client error | `400 Bad Request`, `401 Unauthorized`, `404 Not Found` |
| `5xx` | Server error | `500 Internal Server Error`, `503 Service Unavailable` |

### Commonly confused status codes

| Code | Meaning | Common mistake |
|---|---|---|
| `400` | Malformed request (bad syntax/structure) | Used for validation errors — often more precisely `422` |
| `401` | Not authenticated | Used when the real issue is a permissions problem (`403`) |
| `403` | Authenticated but not permitted | Used when the resource simply doesn't exist (leaks its existence) |
| `404` | Resource not found | Used for "route not implemented" (`501`) or auth failures |
| `409` | Conflict (e.g. duplicate resource, concurrent edit conflict) | Often skipped in favor of a generic `400` |
| `422` | Semantically invalid request body | Confused with `400` |
| `429` | Rate limited | Confused with `503` |
| `500` | Unhandled server error | Used to mask what's actually a client error |
| `503` | Server temporarily unavailable (overload, maintenance) | Confused with `500` |

Choosing `403` vs `404` for an unauthorized resource access is a deliberate
security trade-off: `404` avoids confirming a resource exists to an
unauthorized caller, at the cost of a less informative error for
legitimate callers who mistyped an ID.

### Important headers

```
Content-Type: application/json
Authorization: Bearer <token>
Accept: application/json
Cache-Control: no-cache
ETag: "33a64df551"
If-None-Match: "33a64df551"
Location: /users/42
Retry-After: 30
```

| Header | Purpose |
|---|---|
| `Content-Type` | Format of the request/response body |
| `Accept` | Formats the client can handle (content negotiation) |
| `Authorization` | Credentials (bearer token, basic auth) |
| `ETag` / `If-None-Match` | Conditional requests / caching validation |
| `Location` | Where to find a newly created resource (with `201`) |
| `Retry-After` | How long to wait before retrying (with `429`/`503`) |

### A well-formed response example

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /users/42

{"id": 42, "name": "Sayan"}
```

## When to use

- `GET` for retrieval only — never for anything with side effects, since
  caches, browsers, and crawlers assume `GET` is safe to call repeatedly
  without consequence.
- `201 Created` with a `Location` header when a `POST` creates a new
  resource, so clients know where to find it.
- `204 No Content` for successful operations with nothing meaningful to
  return (e.g. a successful `DELETE`).
- Precise 4xx codes (`401` vs `403` vs `404` vs `409` vs `422`) rather than
  defaulting to `400` for everything client-related.

## When NOT to use

- Don't use `200 OK` with an error message in the body — status codes
  exist precisely so clients, proxies, and monitoring can distinguish
  success from failure without parsing the body.
- Don't use `GET` with a request body to smuggle in "query" data that's
  really more like a filter payload — many HTTP clients/proxies don't
  reliably support `GET` bodies; use query parameters or switch to `POST`
  for complex search endpoints.
- Don't return `500` for expected client errors (bad input, missing
  resource) — reserve `5xx` for genuine server-side failures.

## Common mistakes

- Returning `200` for every response regardless of outcome, forcing
  clients to inspect the body to detect failure.
- Using `POST` for retrieval operations that have no side effects, losing
  cacheability and idempotency guarantees `GET` provides.
- Conflating `401` and `403` — clients can't tell whether re-authenticating
  would help.
- Ignoring `Content-Type`/`Accept` negotiation, silently assuming every
  client wants JSON without checking.

## Interview questions

1. What's the difference between a "safe" and an "idempotent" HTTP method?
   Give an example of each that's one but not the other.
2. When would you use `401` vs `403`? Why does the distinction matter to
   API clients?
3. Why does a `201 Created` response typically include a `Location`
   header?
4. What's the difference between `422` and `400`? Which is more precise
   for a validation error?
5. Why is `GET` expected to have no side effects, and what breaks if you
   violate that expectation?

## Senior-level considerations

- Correct method/status code usage isn't just semantics — it determines
  real behavior in HTTP infrastructure: browsers/proxies retry safe
  methods automatically, cache `GET` responses, and treat `3xx`/`4xx`/`5xx`
  differently for monitoring and alerting.
- Distinguishing `4xx` (client's fault — don't page on-call) from `5xx`
  (server's fault — does page on-call) is often wired directly into
  production alerting/SLO dashboards — getting status codes wrong pollutes
  those signals with false positives or masks real incidents.
- API design at scale should treat headers like `ETag`/`If-None-Match` and
  `Retry-After` as first-class tools for caching and backpressure, not
  optional extras — they materially reduce load on the server when used
  correctly.
