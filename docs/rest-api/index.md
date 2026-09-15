# REST API Engineering

Framework-agnostic notes on designing HTTP APIs: the protocol semantics
REST relies on, and the design decisions (versioning, pagination, rate
limiting, security) that apply regardless of which web framework
implements them.

For FastAPI-specific implementation of these ideas, see the
[FastAPI](../fastapi/index.md) section — particularly
[Routing and Request Handling](../fastapi/01-routing-and-request-handling.md)
and
[Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md).

## Topics

- [HTTP Fundamentals](01-http-fundamentals.md) — HTTP, methods, status codes, headers
- [REST Principles and Idempotency](02-rest-principles-and-idempotency.md) — REST principles, idempotency, PUT vs PATCH
- [Pagination, Filtering, and Sorting](03-pagination-filtering-sorting.md) — designing list endpoints
- [API Versioning and Error Handling](04-versioning-and-error-handling.md) — versioning strategies, error response design
- [Rate Limiting and API Security](05-rate-limiting-and-api-security.md) — throttling, defense-in-depth for public APIs
