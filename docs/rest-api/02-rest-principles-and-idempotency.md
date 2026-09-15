# REST Principles and Idempotency

## What

**REST** (Representational State Transfer) is an architectural style for
designing networked APIs around resources, identified by URLs and
manipulated via a uniform set of HTTP methods. **Idempotency** — an
operation producing the same end state no matter how many times it's
repeated with the same input — is a core property that shapes safe retry
behavior, and directly explains the practical difference between `PUT` and
`PATCH`.

## Why

REST conventions exist to make APIs predictable across teams and
organizations without bespoke documentation for every quirk. Idempotency
specifically matters for reliability: networks fail, clients retry, and an
API that isn't idempotent where it should be can silently create
duplicate resources or double-charge a customer on a simple retry.

## How

### Core REST constraints

| Constraint | Meaning |
|---|---|
| Client-server | Separation of concerns between UI/client and data/server |
| Stateless | Each request contains all information needed; no server-side session state between requests |
| Cacheable | Responses declare whether they can be cached (`Cache-Control`, `ETag`) |
| Uniform interface | Resources identified by URLs, manipulated via standard HTTP methods |
| Layered system | Client can't tell (and shouldn't need to know) if it's talking to the origin server or an intermediary (proxy, gateway, load balancer) |

Statelessness in particular is why REST APIs scale horizontally so well —
any server instance can handle any request since no session affinity is
required (relevant to load balancing, covered in the System Design
section).

### Resource-oriented URLs

```
GET    /orders           # list orders
POST   /orders           # create an order
GET    /orders/42        # retrieve order 42
PUT    /orders/42        # replace order 42 entirely
PATCH  /orders/42        # partially update order 42
DELETE /orders/42        # delete order 42

GET    /orders/42/items  # nested resource: items belonging to order 42
```

URLs identify **resources** (nouns), not actions (verbs) — `POST
/orders/42/cancel` is a common, pragmatic exception for actions that don't
map cleanly to CRUD, but the default should be resource-oriented paths.

### Idempotency

An operation is idempotent if calling it once has the same effect as
calling it N times with the same input.

```
PUT /orders/42 {"status": "shipped"}
```

Calling this once or five times leaves order 42 in exactly the same state
— idempotent. Compare:

```
POST /orders {"item": "widget", "quantity": 1}
```

Calling this five times (e.g. due to a client retry after a timeout)
creates five separate orders — **not** idempotent, unless the server
explicitly does something to prevent it (see idempotency keys below).

| Method | Idempotent? | Why |
|---|---|---|
| `GET` | Yes | Read-only, no state change |
| `PUT` | Yes | Replacing with the same data yields the same end state |
| `DELETE` | Yes | Deleting an already-deleted resource still ends with it gone (typically returns `204` or `404` on repeat, but state converges) |
| `POST` | No (by default) | Typically creates a new resource each call |
| `PATCH` | Usually no | Depends on the patch semantics (see below) |

### `PUT` vs `PATCH`

```http
PUT /users/42
Content-Type: application/json

{"name": "Sayan", "email": "sayan@example.com", "is_active": true}
```

`PUT` semantically **replaces the entire resource** — omitted fields are
implicitly cleared/reset to defaults, not left untouched.

```http
PATCH /users/42
Content-Type: application/json

{"email": "new-email@example.com"}
```

`PATCH` applies a **partial update** — only the specified fields change;
everything else on the resource stays as it was.

```
PATCH is not inherently idempotent: applying {"balance": "+10"} twice
changes the result differently than once. Applying {"balance": 110}
(an absolute value) twice has the same effect as once.
```

Whether a given `PATCH` operation is idempotent depends entirely on
whether the patch expresses an absolute value or a relative delta — this
is a design decision each API must make explicit (commonly documented per
endpoint).

### Idempotency keys (for non-idempotent operations like `POST`)

```http
POST /payments
Idempotency-Key: 8f14e45f-ceea-467e-bc48-ddc7a6b6b6f8
Content-Type: application/json

{"amount": 100, "currency": "USD"}
```

The server records the key alongside the result of the first request; if
the same key arrives again (e.g. a client retry after a network timeout),
the server returns the original result instead of creating a second
payment. This is the standard pattern for making an inherently
non-idempotent operation (like charging a card) safe to retry.

## When to use

- Use `PUT` when the client sends the complete representation of a
  resource and intends to fully replace it.
- Use `PATCH` when the client sends only the fields that changed —
  document explicitly whether your `PATCH` semantics are idempotent
  (absolute values) or not (relative deltas).
- Use idempotency keys for `POST` endpoints representing critical,
  side-effect-heavy operations (payments, order creation) where a client
  retry must not duplicate the effect.
- Design URLs around resources/nouns; use a small number of well-documented
  action-style endpoints (`/orders/42/cancel`) only when a REST-ful
  reinterpretation would be unnatural.

## When NOT to use

- Don't implement `PUT` as a partial update — that violates client
  expectations that omitted fields would be cleared, and clients might
  send true partial payloads assuming `PATCH`-like behavior.
- Don't assume `POST` is safe to retry blindly without an idempotency key
  when the operation has real-world side effects (money, inventory,
  notifications).
- Don't design deeply verb-based endpoints (`/getUserOrders`,
  `/createNewOrderForUser`) as the default style — that abandons the
  benefits of a uniform, resource-oriented interface.

## Common mistakes

- Treating `PATCH` as always idempotent — only true if the patch
  semantics are designed that way (absolute values, not deltas).
- Implementing `PUT` that silently ignores omitted fields instead of
  replacing the whole resource, causing confusing behavior when clients
  expect full replacement semantics.
- Relying on client-side retry logic being "safe" for `POST` without any
  server-side idempotency mechanism, leading to duplicate charges/orders
  under network instability.
- Storing session state on the server between REST calls, breaking
  statelessness and preventing simple horizontal scaling.

## Interview questions

1. What does it mean for an HTTP method to be idempotent? Which methods
   are idempotent by default, and which aren't?
2. What's the practical difference between `PUT` and `PATCH`?
3. Is `PATCH` always idempotent? Give an example where it isn't.
4. How would you make a `POST /payments` endpoint safe to retry without
   double-charging a customer?
5. Why does REST's statelessness constraint matter for horizontal
   scaling?

## Senior-level considerations

- Idempotency keys are a foundational pattern in any distributed system
  where retries are expected (client timeouts, load balancer retries,
  message queue redelivery) — understanding this pattern generalizes well
  beyond REST APIs into async messaging and distributed transactions.
- Whether `PATCH` is idempotent is a concrete example of a design decision
  that should be documented per-API, not assumed — inconsistency across
  endpoints in the same API creates subtle client bugs.
- REST purity is a design philosophy, not a hard requirement — pragmatic
  deviations (action-style endpoints, non-idempotent `PATCH`) are fine when
  documented clearly; the goal is a predictable, well-understood contract,
  not dogmatic adherence to a style guide.
