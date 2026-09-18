# Rate Limiting and API Security

## What

**Rate limiting** restricts how many requests a client can make in a given
time window, protecting the API from overload and abuse. **API security**
here covers the API-design-level defenses (transport security, input
validation at the boundary, least-privilege access) that apply regardless
of framework — deeper cryptographic/auth protocol details (JWT, OAuth2,
password hashing) belong in the dedicated
[Security](../security/index.md) section.

## Why

A public or even internal API without rate limiting is vulnerable to a
single misbehaving client (buggy retry loop, scraper, or malicious actor)
degrading service for everyone. API-level security decisions — what's
exposed, how input is validated, what's logged — are the first line of
defense before deeper auth/crypto concerns even come into play.

## How

### Rate limiting strategies

| Algorithm | How it works | Trade-off |
|---|---|---|
| Fixed window | Count requests per fixed time bucket (e.g. per minute) | Simple; allows bursts at window boundaries (2x limit possible right at the edge) |
| Sliding window | Weighted count across the current and previous window | Smooths out boundary bursts; slightly more complex |
| Token bucket | Tokens refill at a steady rate; each request consumes one | Allows controlled bursts up to bucket size, then throttles |
| Leaky bucket | Requests processed at a constant rate regardless of arrival | Smooths bursts into a steady output rate |

Token bucket is the most common choice for APIs because it naturally
allows short bursts (a client making several quick requests) while still
enforcing a steady-state average rate.

### Signaling rate limits to clients

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1710000000
```

`429` is the standard status code for rate limiting (see
[HTTP Fundamentals](01-http-fundamentals.md)). `X-RateLimit-*` headers
(not officially standardized, but a widely adopted convention) let
well-behaved clients back off proactively before hitting the limit, not
just react after being rejected.

### What to key the rate limit on

```
Per API key / authenticated user  -- fairest for authenticated APIs
Per IP address                     -- fallback for unauthenticated endpoints
Per endpoint                       -- expensive endpoints get tighter limits
```

Rate limiting purely by IP is unreliable behind shared NATs/corporate
proxies (many users share one IP) and is trivially bypassed by an attacker
rotating IPs — prefer per-API-key/per-user limits whenever the caller is
authenticated, reserving IP-based limits for unauthenticated endpoints
(e.g. login, signup) where there's no better identity to key on.

### Distributed rate limiting

A single-process in-memory counter doesn't work once an API runs behind a
load balancer with multiple instances — each instance would enforce its
own separate limit, effectively multiplying the real limit by the instance
count. Production rate limiting typically uses a shared store (Redis,
commonly with an atomic increment + TTL) so all instances share one
counter (see the Caching section for Redis patterns).

### API security at the design level

```
- Validate and constrain all input at the boundary (see Pydantic and
  Validation) -- never trust client-supplied data, including headers and
  query parameters.
- Enforce HTTPS everywhere; never accept credentials over plain HTTP.
- Apply the principle of least privilege: each API key/token should carry
  only the scopes/permissions actually needed.
- Avoid leaking internal details in error responses (stack traces, DB
  errors, internal hostnames) -- log them internally, return a generic
  message externally (see Middleware and Exception Handling).
- Set security-related response headers (e.g. disabling caching of
  sensitive responses, restricting cross-origin access via CORS
  configuration).
```

### Input validation as a security boundary

Strict, explicit input validation (see
[Pydantic and Validation](../fastapi/02-pydantic-and-validation.md)) is
itself a security control — it's the first defense against injection-style
attacks (malformed IDs, oversized payloads, unexpected types) reaching
business logic or a database layer. Precise SQL-injection/authentication
depth belongs in the
[Injection Attacks and Secrets Management](../security/05-injection-attacks-and-secrets-management.md)
section, but the discipline of validating
everything at the API boundary starts here.

### Abuse protection beyond simple rate limits

```
- CAPTCHA or proof-of-work challenges on sensitive unauthenticated
  endpoints (signup, login) to slow down automated abuse.
- Request size limits (reject oversized bodies before they're fully
  parsed) to prevent resource-exhaustion attacks.
- Timeouts on all outbound calls the API makes, so a slow downstream
  dependency can't exhaust the API's own connection pool/thread capacity.
```

## When to use

- Apply rate limiting to every publicly reachable endpoint, with stricter
  limits on expensive or sensitive operations (login, search, report
  generation).
- Key rate limits on API key/authenticated user identity whenever
  possible; fall back to IP-based limiting only for unauthenticated
  endpoints.
- Use a shared store (Redis) for rate limit state as soon as the API runs
  on more than one instance.
- Treat input validation, HTTPS enforcement, and least-privilege access as
  non-negotiable baseline API security, regardless of what deeper auth
  mechanism is layered on top.

## When NOT to use

- Don't rely solely on IP-based rate limiting for authenticated endpoints
  — it's both unfair (shared IPs) and easily bypassed.
- Don't implement rate limiting purely in-process/in-memory once the API
  runs across multiple instances — it won't enforce a real shared limit.
- Don't treat rate limiting as a substitute for proper authentication/
  authorization — it protects against volume/abuse, not against a
  legitimately authenticated but malicious or over-privileged caller.

## Common mistakes

- Fixed-window rate limiting that allows a burst of 2x the intended limit
  right at a window boundary (e.g. 100 requests in the last second of one
  minute, another 100 in the first second of the next).
- Forgetting `Retry-After`/`X-RateLimit-*` headers, leaving clients to
  guess how long to back off after a `429`.
- Leaking internal error details (stack traces, DB connection strings) in
  API error responses, handing attackers useful reconnaissance information.
- Assuming HTTPS termination at a load balancer means internal
  service-to-service calls don't need to be secured too — internal traffic
  deserves the same scrutiny in a zero-trust-oriented architecture.

## Interview questions

1. Compare fixed window, sliding window, and token bucket rate limiting.
   What's the practical weakness of fixed window?

   **Answer:** Fixed window counts requests in a hard-boundary window
   (e.g. 100/minute) and resets at the boundary — its weakness is a burst
   right at the edge (99 requests at 0:59, 99 more at 1:00) can double the
   effective rate. Sliding window smooths that by looking at a rolling
   time range. Token bucket adds tokens at a steady rate and lets requests
   spend them, allowing controlled bursts without the edge problem.

2. Why is IP-based rate limiting often insufficient for authenticated
   APIs?

   **Answer:** Many real users can share one IP (corporate NAT, mobile
   carrier), so IP limiting either blocks innocent users together or is
   trivially bypassed by an attacker rotating IPs. Limiting by
   authenticated user/API key ties the limit to the actual identity.

3. Why doesn't an in-memory rate limit counter work correctly once an API
   runs on multiple instances behind a load balancer?

   **Answer:** Each instance keeps its own separate counter in memory, so
   a client hitting 3 instances round-robin effectively gets 3x the
   intended limit. You need a shared store (Redis) all instances check
   against.

4. What status code and headers should a rate-limited response include,
   and why?

   **Answer:** `429 Too Many Requests`, plus a `Retry-After` header
   telling the client how long to wait before trying again — so
   well-behaved clients back off instead of hammering the API immediately.

5. Why is strict input validation itself considered a security control,
   not just a correctness one?

   **Answer:** Most injection/overflow/DoS-style attacks start with
   unexpected input (oversized payloads, malformed types, unexpected
   characters) — rejecting anything that doesn't match the expected shape
   closes off a huge class of attacks before your business logic even
   runs.

## Senior-level considerations

- Rate limiting design should be informed by real traffic patterns and
  business priorities — e.g. protecting a payment endpoint more strictly
  than a read-only catalog endpoint — rather than applying one blanket
  limit everywhere. For example, `POST /payments` might get 5/minute per
  user while `GET /products` gets 1000/minute.
- Distributed rate limiting (shared Redis-backed counters) introduces its
  own failure mode: if the shared store is unavailable, decide explicitly
  whether to fail open (allow all requests, risking overload) or fail
  closed (reject all requests, risking unnecessary downtime) — this is a
  deliberate architectural trade-off, not an accident. For example, a
  payment API might fail closed (safer to reject than risk unlimited
  abuse), while a public catalog API might fail open (availability over
  strict limiting).
- API-level security (input validation, least privilege, no detail
  leakage) is the foundation that deeper mechanisms (JWT validation,
  OAuth2 scopes, OIDC) build on top of — weaknesses at this level
  (accepting unvalidated input, verbose errors) can undermine even a
  well-implemented auth layer. For example, a stack trace leaked in a
  500 response can hand an attacker internal file paths or library
  versions, regardless of how solid your JWT validation is.
