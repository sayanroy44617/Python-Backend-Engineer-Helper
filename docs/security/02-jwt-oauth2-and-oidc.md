# JWT, OAuth2, and OIDC

## What

**JWT** (JSON Web Token) is a compact, signed token format for carrying
claims (identity, permissions, expiry). **OAuth2** is an authorization
*framework* — a set of flows for granting a client limited access to a
resource without sharing the resource owner's credentials. **OIDC**
(OpenID Connect) is an identity layer built on top of OAuth2, adding
standardized authentication (proving *who* the user is, not just what
they're authorized to access).

## Why

Before these standards, every API invented its own token format and
"login with X" integration. JWT gives a standard, verifiable token shape;
OAuth2 gives a standard way for third-party apps (or your own frontend) to
get limited, revocable access on a user's behalf without ever handling
that user's password; OIDC layers standardized "who is this user"
identity on top, which OAuth2 alone doesn't define.

## How

### JWT structure

```
header.payload.signature

# header (base64):    {"alg": "HS256", "typ": "JWT"}
# payload (base64):   {"sub": "123", "role": "admin", "exp": 1710000000}
# signature:          HMACSHA256(base64(header) + "." + base64(payload), secret)
```

```python
import jwt

token = jwt.encode({"sub": "123", "exp": expire_time}, SECRET_KEY, algorithm="HS256")
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
```

The payload is **base64-encoded, not encrypted** — anyone can decode and
read it without the secret; the signature only proves it wasn't
*tampered with*. Never put sensitive data (passwords, full card numbers)
in a JWT payload.

### Symmetric vs asymmetric signing

```python
# Symmetric (HS256): one shared secret signs and verifies
jwt.encode(payload, SECRET_KEY, algorithm="HS256")

# Asymmetric (RS256): a private key signs, a public key verifies
jwt.encode(payload, private_key, algorithm="RS256")
jwt.decode(token, public_key, algorithms=["RS256"])
```

Asymmetric signing (RS256) lets you distribute the *public* key freely to
any service that needs to verify tokens, while only the issuing
authentication server holds the private key — important in a
microservices architecture where many services verify tokens but only
one should be able to issue them.

### Access tokens and refresh tokens

```
Access token   -- short-lived (minutes), sent with every request
Refresh token  -- long-lived (days/weeks), used only to obtain a new
                  access token, stored more carefully (httpOnly cookie,
                  not local storage)
```

Short-lived access tokens limit how long a stolen token remains useful;
the refresh token (which grants the ability to mint new access tokens)
is the more sensitive credential and deserves stricter storage/transport
protection.

### JWT algorithm confusion attack

```python
# VULNERABLE: accepting whatever algorithm the token header claims
jwt.decode(token, key, algorithms=jwt.get_unverified_header(token)["alg"])

# SAFE: explicitly pin the expected algorithm(s)
jwt.decode(token, key, algorithms=["RS256"])
```

A known real-world JWT vulnerability: if a server trusts the algorithm
named in the token's own header, an attacker can craft a token claiming
`"alg": "none"` or switch from RS256 to HS256 (using the public key, which
is not secret, as the HMAC secret) to forge a valid-looking signature.
Always explicitly specify the accepted algorithm(s) when verifying.

### OAuth2 grant types (flows)

| Flow | Used for | Notes |
|---|---|---|
| Authorization Code | Web apps with a backend | Most common/secure; code exchanged server-side for a token |
| Authorization Code + PKCE | Mobile/SPA apps (no secure backend) | Adds a proof-of-possession step since the client can't hold a secret |
| Client Credentials | Service-to-service (no user involved) | The "client" itself is the resource owner |
| Password (Resource Owner Password Credentials) | Legacy/first-party only | Deprecated in OAuth 2.1 — the app itself sees the user's password |

The password grant (used in the FastAPI `OAuth2PasswordBearer` example)
is convenient for learning/first-party APIs, but the Authorization Code
flow (with PKCE for public clients) is the modern, recommended approach
for anything involving a third-party client or browser-based app.

### Authorization Code flow (simplified)

```

1. App redirects user to the authorization server's /authorize endpoint.

2. User logs in and approves access; server redirects back with a code.

3. App's backend exchanges the code (+ client secret) for an access token
   at the /token endpoint -- this step happens server-to-server, so the
   code alone (briefly visible in a browser redirect) isn't enough to get
   a token without the client secret.

4. App uses the access token to call the API.
```

The code is short-lived and single-use specifically so that even if it
leaks (e.g. via browser history, a referrer header), it can't be
exchanged for a token without the client secret.

### OIDC: identity on top of OAuth2

```python
# An OIDC ID token (also a JWT) carries identity claims
{
  "sub": "123",
  "email": "sayan@example.com",
  "email_verified": true,
  "iss": "https://accounts.example.com",
  "aud": "my-client-id"
}
```

OAuth2 alone only tells you a client has *access* to a resource — it
doesn't standardize how to learn *who the user is*. OIDC adds the ID
token (identity claims about the authenticated user) and a standardized
`/userinfo` endpoint — this is what powers "Sign in with Google/GitHub"
style login.

### Verifying tokens issued by a third party (OIDC/JWKS)

```python
import jwt
from jwt import PyJWKClient

jwks_client = PyJWKClient("https://accounts.example.com/.well-known/jwks.json")
signing_key = jwks_client.get_signing_key_from_jwt(token)
payload = jwt.decode(token, signing_key.key, algorithms=["RS256"], audience="my-client-id")
```

The identity provider publishes its public keys at a well-known JWKS
endpoint; your service fetches (and caches) them to verify tokens it
didn't issue itself — always also verify `aud` (audience) and `iss`
(issuer) claims, not just the signature, to ensure the token was meant
for your service.

## When to use

- JWT for stateless, horizontally-scalable API authentication, especially
  across microservices verifying tokens independently.
- Asymmetric signing (RS256) when multiple services need to verify tokens
  but only one authority should issue them.
- OAuth2 Authorization Code (+ PKCE) for any third-party or browser-based
  client obtaining delegated access.
- OIDC whenever you need to know the authenticated user's identity, not
  just that they have valid access (e.g. "Sign in with X" integrations).

## When NOT to use

- Don't put sensitive data in a JWT payload — it's readable by anyone
  holding the token, signature or not.
- Don't use the Resource Owner Password Credentials grant for anything
  beyond a first-party app you fully control — it requires the app to
  handle the user's actual password.
- Don't accept the algorithm from the token's own header when verifying —
  always pin the expected algorithm(s) explicitly.

## Common mistakes

- Trusting the `alg` field in a JWT header instead of explicitly
  specifying accepted algorithms during verification (the algorithm
  confusion vulnerability).
- Treating a JWT's payload as confidential just because it's not
  human-readable at a glance — base64 is trivially decodable, not
  encryption.
- Using long-lived access tokens instead of a short-lived access token +
  longer-lived refresh token pair, increasing the damage window if a
  token is stolen.
- Not verifying `aud`/`iss` claims when accepting tokens issued by a
  third-party identity provider, potentially accepting a token meant for
  a different application.

## Interview questions

1. What does a JWT's signature actually protect against, and what does it
   *not* protect against?

   **Answer:** The signature proves the token was issued by someone holding the signing key and that the payload was not changed afterward. It does not hide the payload, so anyone holding the token can still read its claims.

2. Explain the JWT algorithm confusion vulnerability and how to prevent
   it.

   **Answer:** The bug happens when the server trusts the token header to tell it how to verify the token, which lets an attacker switch algorithms and trick validation. The fix is simple: hard-code the allowed algorithms and key type on the server side.

   ```python
   import jwt
   
   payload: dict[str, str] = jwt.decode(token, public_key, algorithms=["RS256"])
   ```

3. What's the difference between an access token and a refresh token, and
   why are they typically stored/transported differently?

   **Answer:** An access token is short-lived and sent on API calls; a refresh token is longer-lived and only used to get a new access token. Because a refresh token is effectively a session-renewal credential, teams usually keep it in a more protected place like an httpOnly secure cookie.

4. What problem does OIDC solve that plain OAuth2 doesn't?

   **Answer:** OAuth2 tells you a client got delegated access; it does not standardize user identity. OIDC adds identity claims and standard endpoints so you can answer "who signed in?" instead of only "what can this token access?"

   ```python
   id_token_claims: dict[str, str | bool] = {
       "sub": "123", "email": "dev@example.com", "email_verified": True
   }
   ```

5. Why is the Authorization Code flow (with PKCE) preferred over the
   Password grant for third-party/browser-based clients?

   **Answer:** It keeps the user's password with the identity provider instead of handing it to the client app, which is the big security win. PKCE also protects public clients like SPAs and mobile apps that cannot safely hold a client secret.

   Flow in plain English: browser goes to the identity provider, user logs in there, the app gets a short-lived code, then swaps that code for tokens.

## Senior-level considerations

- Choosing symmetric vs asymmetric JWT signing is an architectural
  decision tied to trust boundaries — asymmetric signing is the right
  default the moment more than one service needs to verify tokens it
  didn't issue. For example, an auth service signs with a private key,
  while the API gateway and three backend services verify with the public
  key only.
- Token revocation strategy (short-lived access tokens, a refresh-token
  blocklist, or a short-TTL token cache check) needs to be decided
  explicitly, since JWTs are not inherently revocable before expiry the
  way a server-side session is. For example, a 10-minute access token +
  refresh-token rotation gives you a bounded compromise window without
  forcing a DB lookup on every request.
- Real-world JWT vulnerabilities (algorithm confusion, missing audience
  validation, weak/shared secrets) come from implementation details, not
  the JWT concept itself — understanding these specific pitfalls is a
  strong signal of practical, not just theoretical, security knowledge.
  For example, accepting a valid Google-issued token without checking
  `aud == "your-api-client-id"` can let your API trust a token meant for
  some other app.
