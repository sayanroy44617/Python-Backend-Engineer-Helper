# Probes, Resources, and Scaling

## What

**Probes** let Kubernetes check container health (`livenessProbe`,
`readinessProbe`, `startupProbe`). **Resource requests/limits** tell the
scheduler how much CPU/memory a container needs and caps how much it can
use. The **Horizontal Pod Autoscaler (HPA)** automatically adjusts
replica count based on observed load.

## Why

Without probes, Kubernetes only knows whether a container's process is
running, not whether it's actually able to serve traffic correctly — the
*why* behind liveness vs. readiness checks is covered in depth in
[Health Checks and Alerting](../../observability/04-health-checks-and-alerting.md);
this file focuses on the Kubernetes-specific YAML mechanics. Resource
requests/limits and the HPA exist because a cluster running arbitrary
workloads needs a way to both schedule Pods sensibly (requests) and
adapt capacity automatically as load changes (scaling), rather than
requiring manual intervention for every traffic spike.

## How

### `livenessProbe`, `readinessProbe`, and `startupProbe`

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.0
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8000
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8000
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
      startupProbe:
        httpGet:
          path: /health/live
          port: 8000
        failureThreshold: 30
        periodSeconds: 2
```

A failing `livenessProbe` causes Kubernetes to **restart** the
container; a failing `readinessProbe` causes Kubernetes to **remove the
Pod from Service endpoints** (stop routing traffic to it) without
restarting it — this distinction, and why an app needs separate
`/health/live` and `/health/ready` endpoints, is exactly what
[Health Checks and Alerting](../../observability/04-health-checks-and-alerting.md)
covers; use that as the source of truth for the *rationale*.
`startupProbe` exists specifically for slow-starting containers — it
suppresses the liveness probe until the app has had time to fully start,
preventing Kubernetes from killing a container that just needs more time
to boot (e.g. warming a cache, running migrations) than the
`livenessProbe`'s normal thresholds would allow.

### Probe types beyond `httpGet`

```yaml
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
---
livenessProbe:
  tcpSocket:
    port: 5432
```

`httpGet` (an HTTP endpoint returning 2xx/3xx = healthy) is the most
common for web APIs. `exec` runs a command inside the container (exit
code 0 = healthy) for cases without an HTTP endpoint. `tcpSocket` just
checks that a TCP port accepts connections — useful for non-HTTP
services (databases, raw TCP servers) where no richer health signal is
available.

### Resource requests and limits

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.0
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

`requests` is what the scheduler guarantees is available and uses to
decide which node can fit the Pod; `limits` is the hard ceiling — a
container exceeding its memory limit is OOM-killed, while exceeding its
CPU limit is throttled rather than killed. Setting `requests` too low
lets the scheduler overcommit a node, causing real contention under
load; setting `limits` too low causes unnecessary restarts/throttling
even when the node has spare capacity.

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

The HPA watches a metric (commonly CPU utilization, but custom metrics
are supported) against the target Deployment and adjusts `replicas`
automatically within `minReplicas`/`maxReplicas` bounds — this requires
`resources.requests` to be set on the containers, since utilization
percentages are calculated relative to the request value.

### Scaling requires stateless, horizontally-scalable Pods

```
HPA increases replicas of a Deployment -- each replica must be able to
handle any request independently (no in-memory session state pinned to
one specific replica) for this to work correctly.
```

Horizontal scaling assumes Pods are interchangeable — an application
relying on local in-memory state that differs between replicas (sticky
sessions, in-process caches assumed consistent across all requests) will
misbehave once the HPA adds a second replica, regardless of how correct
the HPA/Deployment configuration itself is.

### Probes interact with rolling updates

```
Rolling update -- new Pod only receives traffic once its readinessProbe
passes, and the old Pod is only removed once the new one is Ready.
```

This ties back directly to the rolling-update mechanics in
[Pods, Deployments, and Services](01-pods-deployments-and-services.md) —
a correctly configured `readinessProbe` is what makes a rolling update
actually zero-downtime; without it, Kubernetes has no reliable signal
for when a new Pod is genuinely ready to receive traffic.

## When to use

- Always configure both `livenessProbe` and `readinessProbe` for any
  production Deployment serving traffic — they're what make health-based
  self-healing and zero-downtime rollouts actually work.
- `startupProbe` for containers with variable or slow startup time
  (large data loads, migrations, cache warming).
- HPA for workloads with variable load where automatic scaling saves
  meaningful operational overhead over static replica counts.
- Resource `requests`/`limits` on every container, sized from observed
  real usage rather than guessed.

## When NOT to use

- Don't reuse the same endpoint for both liveness and readiness if the
  app has external dependencies (a DB outage should typically fail
  readiness, not liveness — see
  [Health Checks and Alerting](../../observability/04-health-checks-and-alerting.md)
  for the reasoning).
- Don't configure HPA without first setting resource `requests` — CPU
  utilization percentage targets are meaningless without them.
- Don't rely on HPA for a stateful, non-horizontally-scalable workload
  without first ensuring the application can safely run multiple
  replicas.

## Common mistakes

- Using the same check for both liveness and readiness, causing a
  transient dependency outage to trigger unnecessary container restarts
  instead of just temporarily removing the Pod from load balancing.
- Omitting `startupProbe` for slow-starting containers, causing
  Kubernetes to kill and restart them repeatedly before they finish
  booting.
- Setting resource `limits` without `requests` (or leaving both unset),
  leading to poor scheduling decisions and unpredictable node contention.
- Configuring the HPA on an application with per-replica in-memory state
  that isn't safe to run at more than one replica.

## Interview questions

- What's the practical difference in Kubernetes behavior between a
    failing `livenessProbe` and a failing `readinessProbe`?

    **Answer:** A failing liveness probe tells Kubernetes to restart the container because it thinks the app is broken. A failing readiness probe keeps the container running but removes that Pod from Service traffic until it becomes ready again.

    ```yaml
    readinessProbe:
     httpGet:
       path: /health/ready
       port: 8000
    ```

- Why does `startupProbe` exist, and what problem does it solve that
    tuning `livenessProbe` thresholds alone cannot?

    **Answer:** `startupProbe` protects slow-starting apps from being killed during boot. It gives Kubernetes a separate startup window, so you do not have to make the regular liveness check overly slow for the whole life of the container.

    ```yaml
    startupProbe:
     httpGet:
       path: /health/live
       port: 8000
     failureThreshold: 30
    ```

- What's the difference between a container's resource `requests` and
    `limits`, and what happens when each is exceeded?

    **Answer:** Requests are used for scheduling, so they tell Kubernetes how much CPU and memory the Pod needs reserved. Limits are the cap at runtime: going over a memory limit usually gets the container OOM-killed, while going over a CPU limit usually means throttling.

- Why must resource `requests` be set for CPU-based HPA scaling to work
    correctly?

    **Answer:** CPU-based HPA usually works from utilization percentage, and that percentage is calculated against the CPU request. If requests are missing, the HPA does not have a solid baseline, so scaling decisions become invalid or unavailable.

    ```yaml
    resources:
     requests:
       cpu: "250m"
    ```

- Why does a readinessProbe matter for making a rolling update actually
    zero-downtime?

    **Answer:** During a rollout, Kubernetes should only send traffic to the new Pod after the app is actually ready. A readiness probe gives that signal, so users are less likely to hit a process that has started but still cannot serve real requests.

## Senior-level considerations

- Probe design should reflect real dependency health, not just process
  liveness — a `readinessProbe` that checks downstream dependencies
  (database connectivity, for instance) prevents a Pod from receiving
  traffic it can't actually serve, at the cost of needing careful
  threshold tuning to avoid flapping — for example, an API Pod stays out of rotation during a brief PostgreSQL failover instead of accepting requests that would all return 500s.
- Autoscaling based on CPU alone is often too crude for real-world
  traffic patterns — custom metrics (queue depth, request latency,
  in-flight request count) frequently produce better scaling decisions
  for I/O-bound backend services than CPU utilization does — for example, a worker service scales on RabbitMQ queue depth because jobs are piling up even though CPU usage is still low.
- Resource request/limit sizing is an ongoing tuning exercise, not a
  one-time setup — misconfigured values (too generous or too tight)
  directly cause either wasted cluster capacity or avoidable
  throttling/OOM-kills under real production load — for example, a FastAPI service with a 128Mi memory limit keeps restarting during peak traffic until production metrics justify raising it to 512Mi.
