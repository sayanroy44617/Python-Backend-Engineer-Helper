# Failure Handling and Rate Limiting

## What

**Failure handling** is a system's ability to detect, contain, and
recover from partial failures without cascading into a total outage.
**Rate limiting** protects a system (and its dependencies) from being
overwhelmed by more requests than it — or a specific client — should be
allowed to send.

## Why

In a distributed system built from the pieces covered throughout this
handbook (services, databases, queues, caches), any individual component
*will* fail eventually — the system-design question isn't whether to
prevent all failure (impossible), but how to make sure one component's
failure doesn't take down everything else with it. Rate limiting is one
specific, high-leverage failure-handling technique: preventing excessive
load — from a bug, a traffic spike, or a malicious client — from being
the cause of that failure in the first place.

## How

### Cascading failure: the problem

```
Service A calls Service B calls Service C.
C becomes slow -> B's requests to C pile up, exhausting B's connection
pool/threads -> B becomes slow/unresponsive to A -> A's requests to B
pile up -> A becomes unresponsive too.

One slow service takes down the entire call chain.
```

This is the core failure-handling problem: without safeguards, a
localized slowdown in one component propagates and amplifies through
every service that depends on it, often turning a partial degradation
into a total outage.

### Timeouts as the first line of defense

```python
import httpx

response = httpx.get("https://downstream-service/api", timeout=2.0)
```

A call without a timeout can block indefinitely if the downstream
service hangs — this alone is often what turns a downstream slowdown
into an upstream outage. Every network call to another service should
have an explicit, deliberately chosen timeout, not rely on the
library/OS default.

### Circuit breakers

```
Closed:    requests flow normally
Open:      after enough recent failures, stop calling the downstream
           service entirely for a cooldown period, fail fast instead
Half-open: after the cooldown, allow a small number of test requests
           through to check if the downstream service has recovered
```

A circuit breaker stops a struggling downstream service from being
hammered with requests it can't handle (giving it room to recover) and
stops the calling service from wasting its own resources (threads,
connections) waiting on calls likely to fail anyway — directly
preventing the cascading-failure scenario above.

### Retries — but bounded and with backoff

```
Attempt 1: fails -> wait 100ms
Attempt 2: fails -> wait 200ms
Attempt 3: fails -> wait 400ms
Give up, surface the failure
```

The mechanics of retries, backoff, and avoiding a "retry storm" that
makes an already-struggling downstream service worse are covered in
depth in
[Retry Strategies and Dead Letter Queues](../messaging/04-retry-strategies-and-dead-letter-queues.md)
— the same principles apply whether the call being retried is to a
queue consumer or a synchronous downstream API.

### Bulkheads: isolating failure domains

```
Thread pool A: dedicated to calls to Service B
Thread pool B: dedicated to calls to Service C

If Service C hangs, only pool B's threads are exhausted -- calls to
Service B (using pool A) are unaffected.
```

Named after ship bulkheads that contain flooding to one compartment,
this pattern isolates resources per dependency so one dependency's
failure can't exhaust resources shared with calls to a different,
healthy dependency.

### Graceful degradation

```
Primary path:   fetch personalized recommendations from Service X
Degraded path:  if Service X is down/slow, fall back to a generic,
                cached "popular items" list instead of failing the
                whole page load
```

Not every failure needs to become a user-facing error — designing
explicit fallback behavior for non-critical dependencies (personalized
recommendations degrading to generic ones, rather than the entire page
failing) keeps the overall system more available even when one
dependency isn't.

### Health checks and load balancer removal

```
See Load Balancing and Caching at Scale, and Kubernetes: Probes,
Resources, and Scaling -- an unhealthy instance is automatically
removed from a load balancer's rotation, containing its failure to
just that instance rather than routing users to it.
```

This is the same mechanism covered from the load-balancing angle in
[Load Balancing and Caching at Scale](02-load-balancing-and-caching-at-scale.md)
— worth restating here as a failure-handling technique specifically:
automatic removal of unhealthy instances is what turns "one instance
crashed" into a contained, mostly invisible event rather than a
user-facing outage.

### Rate limiting: protecting a system from excessive load

```
Token bucket: each client has a bucket of tokens, refilled at a fixed
              rate; each request consumes a token; no tokens left ->
              request rejected (429) until the bucket refills.
```

The mechanics of rate limiting (algorithms, per-client vs. global
limits, HTTP status codes and headers) are covered in depth in
[Rate Limiting and API Security](../rest-api/05-rate-limiting-and-api-security.md)
— from a system-design perspective, rate limiting is a failure-handling
technique aimed specifically at *load*, whether from a misbehaving
client, a traffic spike, or a retry storm from an upstream caller that
itself lacks proper backoff.

### Rate limiting protects downstream dependencies too

```
Service A rate-limits its own calls to Service B, even if B doesn't
enforce a limit itself -- protecting B (and A's own resources) from A
overwhelming it during a traffic spike or bug.
```

Rate limiting isn't only something a public-facing API applies to its
clients — a service calling an internal dependency should often
self-limit its own outbound call rate too, as a defensive measure
independent of whether the dependency enforces its own limit.

## When to use

- Explicit, deliberately chosen timeouts on every network call to
  another service — never rely on defaults.
- Circuit breakers and bulkheads for calls to dependencies that could
  plausibly become slow or unavailable and whose failure shouldn't
  cascade.
- Rate limiting both at a system's public boundary (protecting it from
  external load) and internally (protecting downstream dependencies from
  a service's own excessive call rate).
- Graceful degradation for genuinely non-critical dependencies where a
  fallback provides real user value over an outright failure.

## When NOT to use

- Don't add circuit breakers/bulkheads to every single call
  indiscriminately — the added complexity is worth it for dependencies
  that can plausibly fail or slow down, not for trivial, always-fast
  in-process operations.
- Don't retry indefinitely or without backoff — an unbounded retry loop
  against a struggling dependency is itself a common cause of cascading
  failure.
- Don't treat graceful degradation as appropriate for every failure — a
  payment failure generally shouldn't "degrade gracefully" into silently
  succeeding with wrong data; some failures should surface clearly.

## Common mistakes

- Network calls without explicit timeouts, letting one slow dependency
  block the calling service indefinitely.
- Unbounded or overly aggressive retries without backoff, worsening load
  on an already-struggling dependency (a "retry storm").
- No circuit breaker or bulkhead isolation, letting one failing
  dependency exhaust resources shared with calls to unrelated, healthy
  dependencies.
- Rate limiting only at the public API boundary, not between internal
  services, leaving internal dependencies unprotected from a
  misbehaving upstream service.

## Interview questions

1. Walk through how a cascading failure propagates through a chain of
   services, and what specific techniques stop it at each stage.

   **Answer:** One downstream service gets slow, callers wait too long, their worker threads or connection pools fill up, and then that slowness spreads upstream. You contain it with short timeouts, bounded retries with backoff, circuit breakers to fail fast, and bulkheads so one bad dependency does not consume shared resources.

2. What's the difference between a circuit breaker's closed, open, and
   half-open states?

   **Answer:** Closed means calls are flowing normally. Open means the breaker has seen enough failures that it stops sending traffic for a while, and half-open means it lets a few test requests through to see whether the dependency has recovered.

   ```python
   if state == "open":
       raise DownstreamUnavailable()
   elif state == "half-open":
       allow_limited_probe_requests()
   ```

3. Why is a bulkhead pattern useful even when a circuit breaker is
   already in place?

   **Answer:** A circuit breaker reacts after failures are detected, but a bulkhead protects resource isolation all the time. If one dependency hangs before the breaker trips, its dedicated pool gets hurt, not the threads or connections needed for other healthy dependencies.

4. Why would a service rate-limit its own outbound calls to a downstream
   dependency, rather than relying solely on the dependency's own rate
   limiting?

   **Answer:** Because by the time the downstream starts rejecting traffic, you may have already flooded it and tied up your own workers. Self-limiting outbound calls protects both systems earlier and gives you a predictable ceiling during spikes or bugs.

   ```python
   if outbound_requests_this_second > 200:
       return fallback_response()
   ```

5. When is graceful degradation the right response to a failure, and
   when is it the wrong one?

   **Answer:** It is right when the failed dependency is useful but not critical, like recommendations, avatars, or analytics. It is wrong when returning partial or guessed behavior would break correctness, such as payments, auth decisions, or inventory confirmation.

## Senior-level considerations

- Failure handling is fundamentally about designing for the failure
  modes you can anticipate (timeouts, circuit breakers, bulkheads,
  graceful degradation) while accepting that some failures will still
  be novel — the goal is limiting blast radius and enabling fast
  recovery, not achieving an impossible zero-failure system — for
  example, a team may plan for database failover and cache loss but
  still rely on runbooks and feature flags when an unexpected DNS issue
  hits multiple services at once.
- Rate limiting, circuit breakers, and bulkheads all trade a small,
  controlled amount of rejected/degraded service now for avoiding a
  much larger, uncontrolled outage later — this trade-off is often
  counterintuitive to communicate to stakeholders who see "deliberately
  rejecting some requests" as itself a failure — for example, returning
  429s to a bursty client for two minutes can protect the checkout
  database from saturating and keep the rest of the site alive.
- Post-incident review after a cascading failure should identify not
  just the initial trigger, but which specific containment mechanism
  (or lack thereof) allowed it to propagate — that's usually where the
  most valuable, durable fix actually lives — for example, after a slow
  search cluster caused an API outage, the lasting fix may be adding a
  timeout and fallback path in the caller rather than only scaling the
  search nodes.
