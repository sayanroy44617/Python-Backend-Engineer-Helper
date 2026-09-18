# Scalability, Statelessness, and Scaling Strategies

## What

**Scalability** is a system's ability to handle increased load by adding
resources. A **stateless service** holds no client-specific state
between requests, which is the property that makes **horizontal
scaling** (adding more instances) actually work, as opposed to
**vertical scaling** (making one instance bigger).

## Why

Every other technique in this section — load balancing, caching,
database scaling, queues — exists to support scaling a system beyond
what a single machine or process can handle. Understanding *why*
statelessness specifically is the prerequisite for horizontal scaling
(not just a nice property) is the foundation the rest of system design
builds on.

## How

### Statelessness: what it actually means

```python
# Stateful (bad for horizontal scaling): in-memory session tied to
# whichever process handled the login request.
sessions: dict[str, User] = {}

@app.post("/login")
def login(credentials: Credentials) -> str:
    session_id = create_session()
    sessions[session_id] = authenticate(credentials)  # lives only here
    return session_id
```

```python
# Stateless: session state lives externally (e.g. Redis, a signed JWT),
# not in this process's memory.
@app.post("/login")
def login(credentials: Credentials) -> TokenResponse:
    user = authenticate(credentials)
    token = issue_jwt(user)   # see FastAPI/Security JWT coverage
    return TokenResponse(access_token=token)
```

A stateless service can have *any* request routed to *any* instance,
because no instance holds information another instance lacks — this is
exactly what makes adding more instances behind a
[load balancer](02-load-balancing-and-caching-at-scale.md) an effective
way to add capacity. A stateful service (in-memory sessions, sticky
in-process caches assumed consistent) breaks this: a client's second
request must reach the same instance as its first, defeating the point
of having multiple interchangeable instances.

### Horizontal vs. vertical scaling

```
Vertical scaling:   one instance -> a bigger instance
                     (more CPU/RAM on the same machine)
Horizontal scaling: one instance -> many instances
                     (same-sized machines, more of them)
```

Vertical scaling is simpler (no architectural changes needed) but has a
hard ceiling (the largest available machine) and creates a single point
of failure. Horizontal scaling has no practical ceiling and improves
fault tolerance (one instance failing doesn't take down the whole
service) — but requires the service to actually be stateless (or to
externalize its state) to work correctly, and requires a
[load balancer](02-load-balancing-and-caching-at-scale.md) to distribute
requests across instances.

### The Horizontal Pod Autoscaler as a concrete example

```yaml
# see Kubernetes: Probes, Resources, and Scaling for the full mechanics
minReplicas: 2
maxReplicas: 10
metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Kubernetes's HPA (covered in depth in
[Probes, Resources, and Scaling](../devops/kubernetes/04-probes-resources-and-scaling.md))
is horizontal scaling automated — but it only works correctly if the
underlying application is actually stateless; the HPA has no way to
migrate in-memory state when scaling from 2 replicas to 10.

### Stateless doesn't mean no state — it means state lives externally

```
Application process:  stateless (no client-specific data held between
                       requests)
State lives in:        the database, a distributed cache (Redis), or
                       the client itself (a JWT, a cookie)
```

"Stateless" describes the *application process*, not the *system* — a
real system obviously has state (user accounts, orders, sessions); the
architectural choice is putting that state in a shared, external store
every instance can access identically
([databases](03-databases-and-queues-at-scale.md), a distributed cache
like [Redis](../caching/02-redis-in-practice.md)) rather than in any
one process's memory.

### Sharding as a scaling technique for state itself

```
Shard by user_id % N  -- each shard holds a subset of users' data,
                         handled by a separate database instance
```

Once the *external* state store itself becomes the bottleneck (a single
database can't handle the load), sharding partitions the data itself
across multiple database instances — a horizontal scaling technique
applied to storage rather than to compute, with its own trade-offs
around cross-shard queries and rebalancing, covered in
[Databases and Queues at Scale](03-databases-and-queues-at-scale.md).

### Scaling isn't free: added operational complexity

```
1 instance:   simple to reason about, but a single point of failure and
              a hard capacity ceiling
N instances:  no single point of failure, near-unlimited capacity, but
              requires load balancing, distributed state, and
              consistency reasoning that a single instance never needed
```

Horizontal scaling trades simplicity for capacity and resilience — a
system doesn't need to scale horizontally from day one, and doing so
prematurely adds real complexity (distributed state, network
partitions, harder debugging) before the load actually justifies it.

## When to use

- Stateless service design as a default for any backend service expected
  to scale beyond a single instance.
- Horizontal scaling once a service's load exceeds what vertical scaling
  can reasonably provide, or when fault tolerance (surviving a single
  instance failure) matters.
- Vertical scaling as a quick, simple mitigation for a service that
  isn't yet a bottleneck at scale, or genuinely can't be made stateless
  (e.g. certain legacy or inherently single-writer workloads).

## When NOT to use

- Don't design a service to hold client-specific in-memory state (a
  session, a request-scoped cache assumed consistent across calls)
  if it will ever need to run more than one instance.
- Don't prematurely introduce horizontal scaling's operational
  complexity (distributed state, sharding) for a service comfortably
  within a single instance's capacity.
- Don't assume "stateless" means "no state anywhere" — the state has to
  live somewhere; the goal is putting it in a shared store, not
  eliminating it.

## Common mistakes

- Storing session or request-scoped state in an application process's
  memory, then discovering it breaks the moment a second instance is
  added behind a load balancer.
- Scaling vertically indefinitely instead of addressing a design that
  prevents horizontal scaling, hitting a hard capacity ceiling later
  than necessary but eventually anyway.
- Introducing horizontal scaling and its associated complexity before a
  service is actually anywhere near a single instance's capacity limits.
- Conflating "stateless service" with "the system has no state" and
  under-designing the external store that state actually needs to live
  in.

## Interview questions

1. Why does statelessness specifically enable horizontal scaling, and
   what breaks in a stateful service when you add a second instance?

   **Answer:** If any instance can handle any request, you can safely add
   more instances behind a load balancer. In a stateful service, the
   second request may hit a different instance that does not have the
   user's in-memory session, cart, or workflow state.

2. What are the trade-offs between horizontal and vertical scaling?

   **Answer:** Vertical scaling is simpler and faster at the start, but it
   hits a hard machine limit and keeps you exposed to one-box failure.
   Horizontal scaling gives better resilience and more headroom, but you
   now need load balancing, shared state, and better operational
   discipline.

3. If an application is stateless, where does its state actually live?

   **Answer:** The app process is stateless, but the system still stores
   state in shared places like PostgreSQL, Redis, object storage, or in
   a signed token on the client. The key idea is that every app instance
   can read the same truth instead of depending on one process's memory.

   ```python
   @app.get("/users/{user_id}")
   def get_user(user_id: int, session: Session) -> UserOut:
       user = session.get(User, user_id)  # shared external state
       return UserOut.model_validate(user)
   ```

4. When would sharding a database be necessary even if the application
   layer itself is already stateless and horizontally scaled?

   **Answer:** You shard when the database itself becomes the bottleneck,
   especially on writes, storage size, or hot tables that one machine
   cannot handle well anymore. Stateless app servers remove compute
   pressure, but they do not magically make one database scale forever.

5. Why might a team choose vertical scaling over horizontal scaling, at
   least initially?

   **Answer:** Because it is usually the fastest low-risk move when the
   system is still small and the problem is just "we need more CPU/RAM
   this week." It buys time without forcing a bigger redesign before the
   team has evidence that distributed complexity is worth it.

   ```yaml
   resources:
     requests: { cpu: "500m", memory: "512Mi" }
     limits: { cpu: "2", memory: "2Gi" }
   ```

## Senior-level considerations

- Designing for statelessness from the start is far cheaper than
  retrofitting it onto a service already built around in-process state —
  this is one of the highest-leverage early architectural decisions in a
  system's lifetime — for example, moving login sessions to Redis in
  month one is much easier than untangling years of sticky-session
  assumptions across multiple services.
- Horizontal scaling shifts complexity from "can one machine handle
  this" to "can this distributed system remain correct and consistent" —
  a trade worth making only once the load genuinely requires it — for
  example, a checkout API with 3x traffic growth may justify multiple
  replicas and shared state, while an internal admin tool usually does
  not.
- Real systems scale different components independently and
  differently — a stateless API layer scales horizontally with ease,
  while its backing database may need sharding, read replicas, or
  vertical scaling depending on its specific bottleneck, discussed
  further in [Databases and Queues at Scale](03-databases-and-queues-at-scale.md)
  — for example, a read-heavy catalog API may jump from 4 to 20 app
  replicas while the database scales mainly through Redis plus read
  replicas instead of sharding.
