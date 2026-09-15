# Security

Application-level security: authentication, authorization, the JWT/OAuth2/
OIDC protocols behind modern auth, password hashing, transport/web
security (HTTPS, CORS, CSRF), injection attacks, and secrets management.

This section covers the underlying concepts and protocols. Framework-level
wiring is covered elsewhere and linked from each topic rather than
repeated here:

- [FastAPI: Authentication and Authorization](../fastapi/05-authentication-and-authorization.md) —
  `OAuth2PasswordBearer`, dependency-based auth checks, `401` vs `403`.
- [REST API Engineering: Rate Limiting and API Security](../rest-api/05-rate-limiting-and-api-security.md) —
  rate limiting, input validation as a security boundary, API-level
  hardening.

## Topics

1. [Authentication and Password Hashing](01-authentication-and-password-hashing.md)
2. [JWT, OAuth2, and OIDC](02-jwt-oauth2-and-oidc.md)
3. [Authorization Models and Least Privilege](03-authorization-and-least-privilege.md)
4. [Transport and Web Security: HTTPS, CORS, CSRF](04-transport-and-web-security.md)
5. [Injection Attacks and Secrets Management](05-injection-attacks-and-secrets-management.md)
