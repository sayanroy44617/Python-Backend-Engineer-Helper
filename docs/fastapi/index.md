# FastAPI

Notes on building production backend services with FastAPI — routing,
validation, dependency injection, and the surrounding concerns needed to
ship a real API, not just a toy endpoint.

This section builds on:

- [Type Hints](../python/05-type-hints.md) — FastAPI/Pydantic use type
  annotations as their source of truth for validation and serialization.
- [Exceptions](../python/03-exceptions.md) — custom exception hierarchies
  and the FastAPI exception handler pattern.
- [Asyncio and Concurrency](../python/13-asyncio-and-concurrency.md) — what
  `async def` route handlers actually do to the event loop.

## Topics

- [Routing and Request Handling](01-routing-and-request-handling.md) — routing, request parameters, request bodies
- [Pydantic and Validation](02-pydantic-and-validation.md) — response models, Pydantic, validation
- [Dependency Injection](03-dependency-injection.md) — `Depends`, scoping, overrides
- [Middleware and Exception Handling](04-middleware-and-exception-handling.md) — middleware, exception handling
- [Authentication and Authorization](05-authentication-and-authorization.md) — auth patterns in FastAPI
- [Async Endpoints, Background Tasks, and Lifespan](06-async-endpoints-and-background-tasks.md) — async endpoints, background tasks, lifespan
- [OpenAPI and Testing](07-openapi-and-testing.md) — OpenAPI, testing
- [Application Architecture](08-application-architecture.md) — structuring a real FastAPI project
