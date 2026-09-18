# Transport and Web Security: HTTPS, CORS, CSRF

## What

**HTTPS** encrypts traffic between client and server (TLS over HTTP).
**CORS** (Cross-Origin Resource Sharing) is a browser-enforced mechanism
controlling which origins may call your API from client-side JavaScript.
**CSRF** (Cross-Site Request Forgery) is an attack tricking a logged-in
user's browser into making an unwanted request to your API, and the
defenses against it.

## Why

These three sit at the boundary between browsers and your API and are
frequently confused with each other, but solve different problems: HTTPS
protects data *in transit* from eavesdropping/tampering; CORS protects
*your users* from a malicious site reading responses from your API using
their credentials; CSRF defenses protect *your API* from a malicious site
triggering unwanted state-changing requests using the victim's ambient
credentials (cookies).

## How

### HTTPS: what TLS actually protects

```
Without TLS: any network intermediary (public wifi, ISP, a compromised
             router) can read AND modify traffic in transit.
With TLS:    traffic is encrypted (confidentiality) and tamper-evident
             (integrity); the certificate also proves server identity.
```

HTTPS should be enforced everywhere, not just on login pages — any
endpoint carrying a session cookie or bearer token in plaintext HTTP is
one network intermediary away from a stolen credential.

### Redirecting/enforcing HTTPS

```python
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app.add_middleware(HTTPSRedirectMiddleware)
```

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

`HSTS` (`Strict-Transport-Security`) tells browsers to *never* attempt a
plain HTTP connection to this domain again for the given duration, closing
the window where a single accidental HTTP request could be intercepted
before a redirect to HTTPS even happens.

### CORS: same-origin policy and why it exists

```
Origin = scheme + host + port
https://app.example.com  and  https://api.example.com  ARE different
origins (different host) -- a browser blocks JS on one from reading
responses from the other unless explicitly allowed.
```

Browsers enforce the *same-origin policy* by default: JavaScript running
on one origin cannot read responses from a different origin. CORS is the
mechanism a server uses to explicitly relax this for specific origins.

### Configuring CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

This builds on the middleware pattern from
[Middleware and Exception Handling](../fastapi/04-middleware-and-exception-handling.md).
The server's CORS headers (`Access-Control-Allow-Origin`, etc.) tell the
*browser* whether to let the calling page's JavaScript read the
response — the request itself often still reaches your server either
way; CORS is a client-side (browser) enforcement mechanism, not a
server-side access control.

### The CORS preflight request

```http
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization
```

For "non-simple" requests (custom headers, methods beyond GET/POST/HEAD,
certain content types), the browser sends an `OPTIONS` preflight first to
ask permission before sending the real request — this is why a
misconfigured CORS setup often manifests as an `OPTIONS` request failing
in the browser console, not the actual `GET`/`POST`.

### `allow_origins=["*"]` and credentials don't mix

```python
# INVALID combination -- browsers reject this outright
allow_origins=["*"]
allow_credentials=True
```

The CORS spec disallows combining a wildcard origin with
`allow_credentials=True` (browsers won't honor it) — if cookies/
credentials need to be sent cross-origin, the server must echo back a
specific, validated origin, not a wildcard.

### CSRF: the attack

```html
<!-- Hosted on a malicious site the victim happens to visit -->
<form action="https://bank.example.com/transfer" method="POST">
  <input type="hidden" name="amount" value="1000">
  <input type="hidden" name="to_account" value="attacker">
</form>
<script>document.forms[0].submit()</script>
```

If the victim is already logged into `bank.example.com` (has a valid
session cookie), the browser automatically attaches that cookie to this
cross-site form submission — the request looks legitimate to the server
even though the victim never intended to make it.

### CSRF defenses

```python
# 1. CSRF tokens: a per-session/per-form token the server verifies
@app.post("/transfer")
def transfer(csrf_token: str = Form(...), session_csrf_token: str = Depends(get_session_csrf_token)):
    if not secrets.compare_digest(csrf_token, session_csrf_token):
        raise HTTPException(status_code=403, detail="Invalid CSRF token")
```

```http
# 2. SameSite cookies -- the modern, simpler-to-adopt defense
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
```

`SameSite=Strict`/`Lax` tells the browser not to send this cookie on
cross-site requests at all, neutralizing the CSRF attack at the browser
level without needing a token in every form. `SameSite=Lax` (the modern
browser default) still allows the cookie on top-level navigation (e.g.
following a link), which covers most CSRF vectors while remaining
convenient; `Strict` is more restrictive still.

### Why token-based (Authorization header) APIs are naturally less exposed to CSRF

```
Cookie-based auth:  browser attaches the cookie automatically -- CSRF-
                     exploitable unless SameSite/CSRF tokens are used.
Header-based auth:  a malicious site's cross-origin form/script can't
                     set a custom Authorization header on the victim's
                     behalf -- CSRF isn't applicable the same way.
```

This is one reason bearer-token APIs (see
[JWT, OAuth2, and OIDC](02-jwt-oauth2-and-oidc.md)) sidestep classic CSRF —
the attack fundamentally relies on the browser *automatically* attaching
credentials (cookies), which doesn't happen for an `Authorization` header
a script would have to set explicitly (and cross-origin scripts can't
read the victim's token to do so).

## When to use

- HTTPS (with HSTS) unconditionally, for every environment beyond local
  development.
- Explicit, narrow `allow_origins` lists for CORS — never a wildcard when
  credentials are involved.
- `SameSite=Lax`/`Strict` cookies as the default CSRF defense for any
  cookie-based session; add explicit CSRF tokens for extra-sensitive
  state-changing operations if needed.

## When NOT to use

- Don't use `allow_origins=["*"]` for any API that relies on cookies/
  credentials — it's rejected by browsers anyway and signals a
  misunderstanding of what CORS protects.
- Don't assume a bearer-token API is CSRF-proof by default if it also
  accepts the token via a cookie as a fallback — that reintroduces the
  cookie-based CSRF surface.
- Don't treat CORS as a server-side access control mechanism — it's
  enforced by the browser, so it does nothing to stop non-browser clients
  (curl, another server) from calling your API directly.

## Common mistakes

- Treating CORS errors in the browser console as "the request failed"
  when the request often reached the server fine — CORS just blocks the
  *browser* from letting JavaScript read the response.
- Setting `allow_origins=["*"]` broadly "to fix a CORS error" without
  understanding the credentials implication, then being confused when
  browsers still reject the combination with `allow_credentials=True`.
- Relying only on cookies for authentication without `SameSite` or CSRF
  tokens, leaving classic CSRF vulnerabilities open.
- Confusing CORS (a browser-side, client protection mechanism) with
  general API security/authorization (a server-side concern) — they
  solve different problems and neither substitutes for the other.

## Interview questions

1. What does HTTPS actually protect against, and why should HSTS be
   configured in addition to a plain HTTP→HTTPS redirect?

   **Answer:** HTTPS encrypts traffic and makes tampering visible, so people on the network cannot quietly read or change requests in transit. HSTS matters because it tells the browser to skip HTTP entirely on later visits, which closes the "first request was plain HTTP" gap.

2. Explain what CORS does and doesn't protect — who is it protecting, the
   server or the browser's user?

   **Answer:** CORS protects the browser user by controlling whether JavaScript on one origin may read responses from another origin. It does not stop curl, backend services, or other non-browser clients from calling your API directly.

3. Why can't you combine `allow_origins=["*"]` with
   `allow_credentials=True`?

   **Answer:** Because once cookies or other credentials are involved, the server has to name the exact allowed origin instead of saying "everyone." Browsers reject the wildcard-plus-credentials combination because it's too broad to be safe.

4. Walk through a CSRF attack scenario and explain how `SameSite` cookies
   prevent it.

   **Answer:** In a CSRF attack, the victim is logged in, visits a malicious page, and that page causes the browser to send a state-changing request to your app with the victim's session cookie attached. `SameSite=Lax` or `Strict` blocks that cookie on cross-site requests, so the forged request arrives unauthenticated.

5. Why are token-in-header APIs generally less exposed to classic CSRF
   than cookie-based session APIs?

   **Answer:** Classic CSRF depends on the browser automatically attaching credentials. An `Authorization` header is normally added by trusted client code, not by the browser on a random cross-site form post, so the attack does not map cleanly the same way.

## Senior-level considerations

- CORS misconfiguration is a common, easy-to-overlook production issue
  precisely because it's a browser-enforced mechanism — server logs alone
  won't show a CORS failure; it only surfaces in the browser console,
  making it a debugging blind spot without frontend visibility; for
  example, the API may return `200 OK`, while the frontend still sees a
  blocked response because `Access-Control-Allow-Origin` is missing.
- Defense-in-depth applies here: `SameSite` cookies substantially reduce
  CSRF risk, but explicit CSRF tokens remain valuable for the most
  sensitive operations (fund transfers, permission changes) as a second
  layer, given `SameSite`'s behavior still has edge cases (older browsers,
  certain cross-site navigation patterns); for example, a banking app may
  require both a valid CSRF token and a `SameSite=Strict` session cookie
  before allowing a transfer.
- HTTPS, CORS, and CSRF protections all need to be considered together
  when designing an authentication scheme (cookie vs bearer token) — the
  choice has cascading security implications across all three, not just
  a convenience trade-off; for example, choosing cookie auth means you
  must design `SameSite` and CSRF defenses up front, while a bearer-token
  SPA usually shifts the main focus toward token storage and CORS.
