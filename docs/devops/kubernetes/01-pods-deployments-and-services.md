# Pods, Deployments, and Services

## What

A **Pod** is the smallest deployable unit in Kubernetes — one or more
tightly coupled containers sharing a network namespace and storage. A
**Deployment** manages a set of identical Pod replicas, handling rolling
updates and self-healing. A **Service** provides a stable network
identity (a name and virtual IP) in front of a changing set of Pods.

## Why

Containers alone (as covered in [Docker](../docker/index.md)) don't
provide self-healing, scaling, or stable networking across many hosts.
Kubernetes' object model exists specifically to answer: "if a Pod dies,
who restarts it?", "how do I run N identical copies and roll out updates
safely?", and "how does one service reliably find another when Pods are
constantly being created and destroyed?"

## How

### A minimal Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
spec:
  containers:
    - name: api
      image: myapp:1.0
      ports:
        - containerPort: 8000
```

A bare Pod is rarely created directly in practice — if its node fails or
the Pod crashes, nothing recreates it. Pods are almost always created
*indirectly*, managed by a higher-level object like a Deployment.

### Deployments: managed, self-healing Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myapp:1.0
          ports:
            - containerPort: 8000
```

A Deployment continuously ensures `replicas` Pods matching `template` are
running — if one crashes or its node fails, the Deployment's controller
creates a replacement automatically. `selector`/`labels` is how the
Deployment identifies which Pods it owns; this label-based matching (not
Pod names) is the core mechanism nearly every Kubernetes object uses to
associate with Pods.

### Rolling updates

```bash
kubectl set image deployment/api api=myapp:1.1
kubectl rollout status deployment/api
kubectl rollout undo deployment/api   # roll back to the previous version
```

A Deployment updates Pods gradually by default (a **rolling update**):
new Pods (with the new image) are created and old ones terminated
incrementally, keeping the service available throughout — rather than
killing every old Pod before any replacement is ready, which would cause
an outage.

### Rolling update strategy configuration

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 Pod below `replicas` at a time
      maxSurge: 1         # at most 1 extra Pod above `replicas` during rollout
```

`maxUnavailable`/`maxSurge` control exactly how aggressive the rollout is
— a smaller `maxUnavailable` prioritizes availability during rollout
(more conservative, slower); a larger `maxSurge` allows the rollout to
happen faster at the cost of briefly running more Pods than usual.

### Services: stable networking for a changing set of Pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8000
```

A Service gives Pods matching `selector` a single stable DNS name (`api`)
and virtual IP, load-balancing traffic across whichever Pods currently
match — individual Pod IPs are ephemeral (a new Pod gets a new IP), so
other services should always talk to a Service's stable name, never a
Pod's IP directly.

### Service types

| Type | Use |
|---|---|
| `ClusterIP` (default) | Internal-only access within the cluster |
| `NodePort` | Exposes the service on a static port on every node — mostly used for simple/dev setups |
| `LoadBalancer` | Provisions an external cloud load balancer — the common choice for public-facing production traffic |

`ClusterIP` is the right default for internal service-to-service
communication (e.g. an API talking to an internal auth service);
external-facing traffic is more commonly routed through an
[Ingress](03-ingress-and-external-access.md) than a raw `LoadBalancer`
Service per application.

### How a Service finds its Pods

```yaml
# Deployment's Pod template
template:
  metadata:
    labels:
      app: api
      tier: backend

# Service selecting those same labels
spec:
  selector:
    app: api
```

The Service's `selector` and the Pod template's `labels` must match — this
label-matching mechanism (not any direct reference by name) is how a
Service continuously discovers the current, correct set of Pods to
route to, even as Pods are replaced during a rollout or after a crash.

### Multi-container Pods

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.0
    - name: log-shipper
      image: fluent-bit:latest  # a "sidecar" container
```

A Pod can hold multiple containers that share a network namespace and can
share volumes — the common **sidecar** pattern (e.g. a log-forwarding or
service-mesh proxy container alongside the main application container) —
but multi-container Pods should be reserved for containers genuinely
tightly coupled to the primary one's lifecycle, not a substitute for
running unrelated services in separate Pods.

## When to use

- A Deployment (not a bare Pod) for essentially every stateless
  application workload, to get self-healing and rolling updates.
- `ClusterIP` Services for internal service-to-service traffic;
  `LoadBalancer` (or Ingress) for external-facing traffic.
- Multi-container Pods only for genuinely tightly-coupled sidecar
  patterns (logging agents, service mesh proxies), not general
  multi-service composition.

## When NOT to use

- Don't create bare Pods directly for anything beyond quick debugging —
  production workloads need a Deployment (or another controller) for
  self-healing.
- Don't reference Pod IPs directly from other services — they change on
  every Pod recreation; always go through a Service.
- Don't bundle unrelated services into one multi-container Pod just to
  avoid creating separate Deployments — it couples their scaling and
  lifecycle unnecessarily.

## Common mistakes

- Creating bare Pods for real workloads, then being surprised when a
  crashed Pod isn't automatically replaced.
- Mismatched labels between a Deployment's Pod template and a Service's
  selector, causing the Service to route to zero Pods silently.
- Assuming a rolling update guarantees zero downtime without also
  configuring proper [readiness probes](04-probes-resources-and-scaling.md)
  — Kubernetes only stops routing to a Pod once it's marked *not ready*,
  so a Pod that's technically running but not yet able to serve requests
  can still receive traffic without one.
- Treating `NodePort`/direct Pod IPs as a stable production access
  pattern instead of a Service.

## Interview questions

1. Why does Kubernetes rarely have you create bare Pods directly in
   practice?
2. How does a rolling update keep a service available throughout a
   deployment, and what do `maxUnavailable`/`maxSurge` control?
3. How does a Service know which Pods to route traffic to, given that
   Pods are constantly being created and destroyed?
4. What's the difference between `ClusterIP`, `NodePort`, and
   `LoadBalancer` Service types?
5. When would a multi-container Pod (sidecar pattern) be appropriate, and
   when is it an anti-pattern?

## Senior-level considerations

- Label/selector design is a foundational Kubernetes skill — nearly every
  higher-level object (Services, NetworkPolicies, Deployments) relies on
  consistent, well-thought-out labeling; a haphazard labeling scheme
  causes subtle, hard-to-debug misrouting.
- Rolling update configuration (`maxUnavailable`/`maxSurge`) combined with
  correct readiness probes is what actually achieves zero-downtime
  deployments — either piece alone is insufficient, and this combination
  is a common gap in less mature Kubernetes setups.
- The Pod/Deployment/Service separation reflects a broader Kubernetes
  design principle: small, composable objects, each with one
  responsibility (identity/lifecycle vs. desired-state management vs.
  stable networking) — understanding this separation clarifies almost
  every other Kubernetes object built on top of it.
