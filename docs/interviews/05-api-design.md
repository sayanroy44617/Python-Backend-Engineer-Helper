# Interview Prep: API Design

## How to approach API design interview questions

"Design a REST API for X" is a distinct interview format from a
system-design question — the focus is the *contract* (resources,
methods, request/response shapes, error handling, versioning) rather
than infrastructure/scaling. The mechanics referenced throughout this
page are covered in depth in
[REST API Engineering](../rest-api/index.md); this page is the
interview-specific *framework* for structuring an answer under time
pressure, plus the questions most likely to probe gaps in that
framework.

## A repeatable framework for an API design question

1. **Clarify the resource model.** What are the core nouns (resources)?
   What are their relationships (one-to-many, many-to-many)? State this
   explicitly before writing any endpoint — most API design mistakes
   stem from skipping this step.
2. **Define the endpoints per resource**, following REST conventions
   (`GET /orders`, `POST /orders`, `GET /orders/{id}`, `PATCH
   /orders/{id}`, `DELETE /orders/{id}`) — see
   [REST Principles and Idempotency](../rest-api/02-rest-principles-and-idempotency.md)
   for the reasoning behind resource-oriented URLs and HTTP method
   semantics.
3. **Design the request/response payloads.** What does a client send to
   create a resource? What does the server return? Be explicit about
   required vs. optional fields and what's server-generated (IDs,
   timestamps).
4. **Handle collections properly**: pagination, filtering, sorting — see
   [Pagination, Filtering, and Sorting](../rest-api/03-pagination-filtering-sorting.md).
   Never propose returning an unbounded collection.
5. **Define the error model.** What does a validation error, a not-found,
   and a conflict look like? Use standard HTTP status codes consistently
   — see
   [API Versioning and Error Handling](../rest-api/04-versioning-and-error-handling.md).
6. **Address versioning strategy** if the interviewer asks about
   evolution — how would a breaking change to this API be rolled out
   without breaking existing clients?
7. **Address cross-cutting concerns**: authentication/authorization,
   rate limiting, idempotency for unsafe operations that might be
   retried — see
   [Rate Limiting and API Security](../rest-api/05-rate-limiting-and-api-security.md).
8. **State explicit trade-offs** you made along the way (e.g. "I chose
   cursor-based pagination over offset because this collection can grow
   large") — this is usually worth more credit than the specific design
   choice itself.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | How would you design endpoints for a resource with a one-to-many child relationship (e.g. orders and order items)? | Nest the child under the parent for creation/listing in context (`GET /orders/{id}/items`), but also consider whether the child needs to be addressable independently — the right nesting depth depends on whether the child has meaning outside its parent. | [REST Principles and Idempotency](../rest-api/02-rest-principles-and-idempotency.md) |
| 2 | Why should `POST /orders` support idempotency for retried requests? | A client retrying after a timeout (not knowing if the original request succeeded) risks creating duplicate orders — an `Idempotency-Key` header lets the server recognize and safely no-op a retried request instead of creating a duplicate. | [REST Principles and Idempotency](../rest-api/02-rest-principles-and-idempotency.md) |
| 3 | What's the difference between `PUT` and `PATCH`, and when would you use each? | `PUT` replaces the entire resource with the provided representation; `PATCH` applies a partial update — using `PUT` for a partial update risks silently clearing fields the client didn't include. | [REST Principles and Idempotency](../rest-api/02-rest-principles-and-idempotency.md) |
| 4 | How would you version an API that needs a breaking change? | Prefer a URL or header-based version marker (`/v2/orders` or an `Accept`/custom header) that lets old and new clients coexist during a migration window, rather than breaking existing clients outright. | [API Versioning and Error Handling](../rest-api/04-versioning-and-error-handling.md) |
| 5 | How should an API represent validation errors to the client? | A consistent, structured error body (e.g. a list of field-level errors alongside a top-level message) with an appropriate 4xx status code — not a generic 500 or an unstructured string message. | [API Versioning and Error Handling](../rest-api/04-versioning-and-error-handling.md) |
| 6 | Cursor-based vs. offset-based pagination — how would you decide, and why does it matter for API design specifically? | Cursor-based pagination is stable under concurrent inserts/deletes and scales better for large collections; offset-based is simpler to implement and sufficient for small, rarely-changing collections — choose based on collection size/volatility, and state that reasoning explicitly. | [Pagination, Filtering, and Sorting](../rest-api/03-pagination-filtering-sorting.md) |
| 7 | How would you design rate limiting into a public API from the start? | Communicate limits via response headers (remaining quota, reset time), return `429` with a `Retry-After` header when exceeded, and consider different limits for authenticated vs. unauthenticated clients. | [Rate Limiting and API Security](../rest-api/05-rate-limiting-and-api-security.md) |
| 8 | How would you design an endpoint that triggers a long-running operation (e.g. generating a report)? | Return `202 Accepted` immediately with a reference to check status later (a job ID and a `GET /jobs/{id}` endpoint), rather than holding the client's connection open until the operation completes. | [HTTP Fundamentals](../rest-api/01-http-fundamentals.md) |
| 9 | What's a common mistake when designing an API's URL structure? | Modeling actions as verbs in the URL (`/createOrder`) instead of resources with HTTP methods (`POST /orders`) — breaking the resource-oriented convention that makes REST APIs predictable and cacheable. | [REST Principles and Idempotency](../rest-api/02-rest-principles-and-idempotency.md) |
| 10 | How would you design an API to support filtering a large collection efficiently? | Expose filter parameters as query parameters mapped to indexed columns, document allowed filters explicitly, and ensure the underlying query uses appropriate indexes — an unindexed filter on a large table degrades badly regardless of how clean the API surface looks. | [Pagination, Filtering, and Sorting](../rest-api/03-pagination-filtering-sorting.md) |

## Common red flags interviewers watch for

- Jumping straight to endpoint URLs without first clarifying the
  resource model and relationships.
- Designing endpoints as RPC-style actions instead of resource-oriented
  nouns and HTTP verbs.
- Returning unbounded collections with no pagination story.
- No plan for versioning or backward compatibility when asked "how would
  this evolve."
- Ignoring idempotency for unsafe operations that a client might
  plausibly retry.

## Related deep-dive material

- [REST API Engineering section overview](../rest-api/index.md) — 5
  topic pages covering the underlying mechanics this framework draws on.
