# Health Checks and Alerting

## What

A **health check** is an endpoint (or process) that reports whether a
service is running correctly, used by orchestrators (Kubernetes, load
balancers) to decide whether to route traffic to an instance or restart
it. **Alerting** turns observability data (metrics, logs, traces) into
actionable notifications when something needs human attention.

## Why

Without health checks, an orchestrator has no reliable way to know an
instance has silently wedged (still running, but unable to serve
requests) versus genuinely healthy — it would keep routing traffic to a
broken instance. Without deliberate alerting, an on-call engineer only
learns about an incident from a user complaint, rather than from the
system itself surfacing the problem as (or before) it happens.

### Liveness vs. readiness

```python
@app.get("/health/live")
def liveness():
    return {"status": "ok"}  # is the process itself still running?

@app.get("/health/ready")
def readiness(db: Session = Depends(get_db)):
    try:
        db.execute(text("SELECT 1"))
    except Exception:
        raise HTTPException(status_code=503, detail="database unavailable")
    return {"status": "ready"}  # can this instance actually serve traffic?
```

**Liveness** answers "is the process alive at all" — a failing liveness
check tells an orchestrator to restart the instance. **Readiness**
answers "can this instance currently serve traffic" — a failing readiness
check tells a load balancer to stop routing to it *without* restarting
it (e.g. during a slow startup, or a temporary dependency outage that
will resolve on its own).

### Why conflating liveness and readiness is a common mistake

```python
# WRONG: a single /health check restarts the whole instance every time
# a downstream dependency (e.g. the database) is briefly unavailable
@app.get("/health")
def health(db: Session = Depends(get_db)):
    db.execute(text("SELECT 1"))  # if this fails, /health fails
    return {"status": "ok"}
```

If a single combined check backs *both* liveness and readiness, a
transient database blip causes the orchestrator to restart every
instance (liveness failure) instead of simply pausing traffic to them
(readiness failure) — restarting doesn't fix a database outage and adds
unnecessary churn (lost in-flight requests, cold-start latency) on top of
an already-degraded situation.

### What a readiness check should (and shouldn't) verify

```python
@app.get("/health/ready")
def readiness(db: Session = Depends(get_db), redis_client: Redis = Depends(get_redis)):
    checks = {"database": check_db(db), "cache": check_redis(redis_client)}
    if not all(checks.values()):
        raise HTTPException(status_code=503, detail=checks)
    return {"status": "ready", "checks": checks}
```

Readiness checks should verify dependencies the service genuinely can't
function without (its primary database) — but avoid cascading failure by
checking *every* downstream dependency, including ones the service can
degrade gracefully without (e.g. an optional recommendations service);
otherwise one unrelated dependency's outage takes down instances that
could otherwise still serve most traffic.

### Startup probes for slow-starting services

```yaml
startupProbe:
  httpGet:
    path: /health/live
    port: 8000
  failureThreshold: 30
  periodSeconds: 2
```

A separate startup probe (common in Kubernetes) gives a slow-starting
service (large cache warm-up, migration check) extra time before liveness
checks begin, preventing an orchestrator from repeatedly killing an
instance that simply hasn't finished starting yet.

### Alerting on symptoms, not just causes

```
Symptom-based (preferred): "error rate > 5% for 5 minutes",
                            "p99 latency > 2s for 10 minutes"
Cause-based (secondary):   "CPU > 90%", "disk usage > 80%"
```

Symptom-based alerts (derived from user-facing metrics — see
[Metrics and Prometheus](02-metrics-and-prometheus.md) and the RED
method) directly reflect whether users are actually affected. Cause-based
alerts (resource utilization) are useful for capacity planning and early
warning, but high CPU alone doesn't necessarily mean anything is actually
broken — alerting primarily on symptoms avoids paging someone for a
condition that isn't yet (and might never become) user-impacting.

### Alert fatigue

```
Too many low-signal alerts -> on-call engineers start ignoring/muting
  alerts -> a genuinely critical alert gets missed among the noise
```

An alert that fires frequently without requiring action trains
responders to ignore alerts generally — every alert should be actionable
(there's something a human can/should do in response) and should
represent a real, meaningful degradation; alerts that don't meet this bar
should be turned into a dashboard panel instead, not a page.

### Alert severity and routing

```
Page immediately (wakes someone up): user-facing outage, data loss risk
Notify during business hours:        degraded but non-critical, capacity
                                      trending toward a limit
Log/dashboard only:                  informational, no action needed
```

Not every anomaly deserves an immediate page — matching alert severity to
actual urgency (and routing accordingly) is what keeps a page a
meaningful, trusted signal rather than noise.

### Runbooks alongside alerts

```
Alert: "checkout error rate > 5%"
Runbook: 1. Check the payment-service dashboard for its own error rate.
         2. Check recent deploys (has anything shipped in the last hour?).
         3. If payment-service is degraded, see its incident runbook.
```

An alert without a runbook forces whoever's on-call to rediscover the
diagnostic steps from scratch, often during a stressful incident — a
linked runbook (or at minimum, the relevant dashboards) turns "something
is wrong" into "here's how to start investigating," directly speeding up
incident response.

## When to use

- Separate liveness and readiness checks for any service running under an
  orchestrator (Kubernetes, most cloud platforms) — never one combined
  check backing both behaviors.
- Readiness checks covering only hard dependencies the service truly
  cannot function without.
- Symptom-based alerting (RED-method style: error rate, latency) as the
  primary paging signal, with cause-based/resource alerts as
  secondary/informational.
- A runbook (or at least a linked dashboard) attached to every paging
  alert.

## When NOT to use

- Don't combine liveness and readiness into a single check — it causes
  unnecessary restarts for issues that only need traffic to pause, not a
  full restart.
- Don't page for every metric anomaly — reserve pages for genuinely
  actionable, user-impacting conditions; route everything else to a
  dashboard or a lower-urgency notification channel.
- Don't include optional/non-critical dependencies in a readiness check —
  it causes unrelated outages to cascade into your own service's
  unavailability.

## Common mistakes

- One `/health` endpoint used for both liveness and readiness, causing
  unnecessary restarts during transient dependency issues.
- Readiness checks that verify every downstream dependency (including
  optional ones), turning any unrelated outage into a full outage of this
  service too.
- Alerting on raw resource metrics (CPU, memory) as the primary signal
  instead of symptom-based metrics that actually reflect user impact.
- Alert fatigue from too many low-value alerts, causing genuinely
  critical alerts to be missed or dismissed out of habit.

## Interview questions

1. What's the difference between a liveness check and a readiness check,
   and why does conflating them cause unnecessary restarts?
2. What should — and shouldn't — a readiness check verify? Why is
   checking every downstream dependency a mistake?
3. Why is symptom-based alerting (error rate, latency) generally
   preferred over cause-based alerting (CPU, memory) as the primary
   paging mechanism?
4. What is alert fatigue, and what design practices help prevent it?
5. Why does attaching a runbook to an alert matter operationally, beyond
   just having the alert fire correctly?

## Senior-level considerations

- Health check design directly affects system resilience during partial
  outages — a well-designed readiness check lets an orchestrator route
  around a genuinely degraded instance without unnecessarily restarting
  healthy ones, minimizing blast radius.
- Alerting philosophy (symptom-based, actionable, appropriately routed) is
  a deliberate design discipline that directly affects on-call
  sustainability — poorly designed alerting is a common, underestimated
  cause of team burnout and, ironically, worse incident response over
  time as real alerts get lost in noise.
- Observability (logging, metrics, tracing, health checks, alerting) works
  best as one integrated system rather than five separate tools — a
  senior engineer designs these together so an alert can lead directly to
  the relevant dashboard, trace, and logs for the affected request,
  rather than requiring manual correlation during an incident.
