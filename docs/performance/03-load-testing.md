# Load Testing

## What

**Load testing** simulates realistic (or intentionally excessive) traffic
against a system to measure how it behaves under load — throughput,
latency, error rate — before that load happens for real in production.

## Why

Code that works correctly for one request at a time can behave very
differently under concurrent load: connection pools exhaust, caches
thrash, previously-invisible N+1 queries multiply, and latency can
degrade nonlinearly as a system approaches its capacity limits. Load
testing surfaces these failure modes deliberately, on your own schedule,
instead of discovering them during a real traffic spike.

## How

### Types of load tests

| Type | Purpose |
|---|---|
| Load test | Verify behavior at expected/typical peak traffic |
| Stress test | Push beyond expected limits to find the actual breaking point |
| Soak test | Sustain moderate load over a long duration to catch slow leaks/degradation |
| Spike test | Sudden, sharp traffic increase to test how quickly the system adapts |

Each answers a different question — a load test confirms you can handle
expected traffic; a stress test tells you what happens beyond that; a
soak test catches issues (memory growth, connection leaks) that only
appear after sustained runtime, not in a short burst.

### A basic load test with Locust

```python
from locust import HttpUser, task, between

class ApiUser(HttpUser):
    wait_time = between(1, 3)

    @task(3)
    def get_products(self):
        self.client.get("/products")

    @task(1)
    def create_order(self):
        self.client.post("/orders", json={"product_id": 1, "quantity": 2})
```

```bash
locust -f locustfile.py --host https://staging.example.com --users 500 --spawn-rate 10
```

`@task(3)` vs `@task(1)` weights how often each action runs relative to
others — modeling realistic traffic composition (many more reads than
writes, typically) matters more for meaningful results than just
hammering a single endpoint.

### A basic load test with k6

```javascript
import http from "k6/http";
import { sleep, check } from "k6";

export const options = {
  stages: [
    { duration: "1m", target: 100 },  // ramp up to 100 concurrent users
    { duration: "3m", target: 100 },  // hold steady
    { duration: "1m", target: 0 },    // ramp down
  ],
};

export default function () {
  const res = http.get("https://staging.example.com/products");
  check(res, { "status is 200": (r) => r.status === 200 });
  sleep(1);
}
```

Staged ramp-up/hold/ramp-down (rather than an instant jump to full load)
better reflects real traffic growth and lets you observe *how* the
system degrades as load increases, not just its behavior at one fixed
load level.

### What to measure

```
Throughput:    requests handled per second
Latency:       p50/p95/p99 response time (not just average -- see below)
Error rate:    % of requests failing (non-2xx, timeouts)
Saturation:    CPU/memory/connection pool usage as load increases
```

These map directly onto the RED method covered in
[Metrics and Prometheus](../observability/02-metrics-and-prometheus.md) —
load testing is, in effect, deliberately generating the traffic needed to
observe those same metrics under controlled conditions.

### Why percentiles matter more than averages

```
Average latency: 120ms  -- looks fine
p99 latency:      2.4s  -- 1% of users have a genuinely bad experience
```

An average can look perfectly healthy while a meaningful fraction of
requests (the tail) are significantly slower — for a system serving many
users, p95/p99 latency reflects the experience of your worst-served
users, which matters disproportionately for user-perceived reliability.

### Finding the breaking point

```
Ramp load up in stages: 100 -> 500 -> 1000 -> 2000 concurrent users
Watch for: latency climbing nonlinearly, error rate rising, saturation
           metrics (CPU, connection pool, memory) hitting their limits
```

The goal of a stress test isn't just "does it survive" — it's finding
*where* and *why* it starts to fail (which resource saturates first: CPU,
database connections, memory) so that specific bottleneck can be
addressed or capacity-planned for ahead of time.

### Load testing against a realistic environment

```
Testing against a tiny dev database with 10 rows tells you almost
nothing about production behavior under real data volume and query plans.
```

A load test's usefulness depends heavily on how representative the test
environment is — data volume, index presence, network topology, and
downstream dependency behavior should approximate production as closely
as practical, or the results risk being misleading.

### Load testing and connection pools

```python
# A load test at 500 concurrent users against a pool_size=10 database
# connection pool will reveal exactly how pool_timeout errors manifest
# under real concurrent pressure
```

Load testing is exactly what validates connection pool sizing decisions
(see
[Transactions and Connection Pools](../databases/sqlalchemy/03-transactions-and-connection-pools.md))
empirically, rather than guessing at `pool_size`/`max_overflow` values
and hoping they hold up in production.

## When to use

- Before any significant traffic-driving event (launch, marketing
  campaign, seasonal peak) to validate the system handles expected load.
- After any significant architecture change (new caching layer, database
  migration, new service) to confirm performance assumptions still hold.
- Regularly (e.g. in CI or on a schedule) as a soak/regression check,
  not just as a one-off pre-launch activity.

## When NOT to use

- Don't run a load test against production without careful planning
  (rate limits, safety cutoffs, off-peak timing) — an uncontrolled load
  test can cause the very outage you're trying to prevent.
- Don't rely solely on average latency to judge results — always examine
  p95/p99 and error rate together.
- Don't load test against a non-representative environment (tiny dataset,
  no realistic index/data volume) and treat the results as predictive of
  production behavior.

## Common mistakes

- Judging results by average latency alone, missing a significant tail
  latency problem affecting a meaningful fraction of real users.
- Load testing against a dev/staging environment with unrealistic data
  volume, producing results that don't transfer to production behavior.
- Testing only a single endpoint in isolation rather than a realistic mix
  of traffic (reads, writes, varied endpoints) matching actual usage
  patterns.
- Treating load testing as a one-time pre-launch checkbox instead of an
  ongoing practice as the system and its traffic patterns evolve.

## Interview questions

1. What's the difference between a load test, a stress test, and a soak
   test, and what does each uniquely reveal?
2. Why is p99 latency often more important than average latency for
   understanding real user experience?
3. How would you use a load test to validate a connection pool's
   `pool_size` configuration?
4. What risks come with running a load test directly against a
   production environment, and how would you mitigate them?
5. Why does the realism of the test environment (data volume, indexes,
   network topology) matter for how much you can trust load test results?

## Senior-level considerations

- Load testing should be tied to specific, falsifiable capacity questions
  ("can we handle 2x current peak traffic with p99 latency under 500ms")
  rather than run as an unfocused, generic exercise — clear success
  criteria make results actionable.
- Finding the actual bottleneck during a stress test (which resource
  saturates first) directly informs capacity planning and prioritizes
  which performance fix (see
  [Application Performance Fundamentals](01-application-performance-fundamentals.md)
  and
  [Database Performance in Practice](02-database-performance-in-practice.md))
  to invest in next.
- Load testing is most valuable when integrated into the regular
  development lifecycle (e.g. run against every significant release
  candidate) rather than treated as a one-off pre-launch ritual — traffic
  patterns and system architecture both evolve, and yesterday's load test
  results don't necessarily hold today.
