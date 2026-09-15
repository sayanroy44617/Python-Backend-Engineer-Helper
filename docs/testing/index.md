# Testing

Testing turns "it works on my machine" into a repeatable, automated
guarantee. This section covers pytest fundamentals, mocking, test
strategy, and testing async code — the general-purpose testing skills
that apply whether you're testing a pure function, a database layer, or
an API.

API-specific testing (FastAPI's `TestClient`, dependency overrides for
testing routes) is covered in
[OpenAPI and Testing](../fastapi/07-openapi-and-testing.md) — this section
focuses on the underlying pytest mechanics that make that possible.

## Topics

1. [Pytest Fundamentals](01-pytest-fundamentals.md)
2. [Mocking and Monkeypatch](02-mocking-and-monkeypatch.md)
3. [Test Strategy and Isolation](03-test-strategy-and-isolation.md)
4. [Async Testing](04-async-testing.md)
