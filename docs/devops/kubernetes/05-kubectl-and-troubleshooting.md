# kubectl and Troubleshooting

## What

`kubectl` is the command-line tool for interacting with a Kubernetes
cluster's API server — inspecting, creating, and modifying resources.
Troubleshooting in Kubernetes is largely a systematic process of using
`kubectl` to narrow down *where* in the Pod's lifecycle a failure is
occurring.

## Why

Kubernetes failures rarely surface as a single clear error message — a
Pod stuck `Pending`, crash-looping, or silently not receiving traffic
could stem from scheduling, the image, the application itself, networking,
or configuration. A systematic `kubectl`-driven workflow narrows down the
cause quickly instead of guessing.

## How

### Core inspection commands

```bash
kubectl get pods                          # list Pods and their status
kubectl get pods -o wide                  # + node, IP
kubectl get deployments
kubectl get services
kubectl describe pod <pod-name>           # detailed status + events
kubectl logs <pod-name>                   # container stdout/stderr
kubectl logs <pod-name> -c <container>    # specific container in a multi-container Pod
kubectl logs <pod-name> --previous        # logs from the last crashed instance
kubectl logs -f <pod-name>                # follow/tail logs
```

`describe` is usually the single most useful troubleshooting command —
its `Events` section at the bottom shows the actual sequence of
scheduling/pull/probe events, which is where most root causes are
visible directly.

### Executing into a running container

```bash
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- python -c "import os; print(os.environ)"
```

Useful for verifying environment variables, testing DNS/connectivity to
a dependency, or checking whether a mounted [ConfigMap/Secret](02-configmaps-and-secrets.md)
actually appears as expected inside the container — though for
production debugging, prefer structured logs/metrics
(see [Logging and Structured Logging](../../observability/01-logging-and-structured-logging.md))
over ad hoc `exec` sessions where possible.

### Applying and managing manifests

```bash
kubectl apply -f deployment.yaml     # create or update from a manifest
kubectl delete -f deployment.yaml    # delete the resources it defines
kubectl rollout status deployment/api
kubectl rollout undo deployment/api  # roll back to the previous revision
kubectl rollout history deployment/api
```

`kubectl apply` is declarative (it reconciles the cluster to match the
file, whether creating or updating), in contrast to `kubectl create`
which fails if the resource already exists — `apply` is the standard
choice for repeatable, GitOps-friendly deployments.

### `CrashLoopBackOff`

```bash
kubectl describe pod <pod-name>   # check Events for exit reason
kubectl logs <pod-name> --previous  # see why the last attempt crashed
```

`CrashLoopBackOff` means the container starts, exits (typically nonzero
exit code), and Kubernetes keeps restarting it with an increasing
back-off delay. Root causes are almost always in the application itself
or its immediate configuration — a missing environment variable, a
failing startup dependency check, or an unhandled exception on boot.
`--previous` is essential here since the current container instance may
have already restarted again by the time you look.

### `ImagePullBackOff` / `ErrImagePull`

```bash
kubectl describe pod <pod-name>   # Events show the exact pull error
```

Almost always one of: a typo in the image name/tag, the image doesn't
exist in the registry, or missing registry credentials (an
`imagePullSecrets` reference — see
[ConfigMaps and Secrets](02-configmaps-and-secrets.md) — that's missing
or incorrect for a private registry).

### Pods stuck `Pending`

```bash
kubectl describe pod <pod-name>   # Events show scheduling failure reason
kubectl describe nodes            # check node capacity/taints
```

A Pod stuck `Pending` means the scheduler cannot place it — common
causes are insufficient node capacity for the requested
[resources](04-probes-resources-and-scaling.md), no node matching a
`nodeSelector`/affinity rule, or a taint on all available nodes without a
matching toleration.

### Pods `Running` but not receiving traffic

```bash
kubectl get endpoints <service-name>   # are any Pod IPs listed?
kubectl describe service <service-name>
```

If a Service's `Endpoints` list is empty despite Pods being `Running`,
the label selector (see
[Pods, Deployments, and Services](01-pods-deployments-and-services.md))
doesn't match the Pods' labels, or the Pods are failing their
`readinessProbe` (see
[Probes, Resources, and Scaling](04-probes-resources-and-scaling.md)) and
so are excluded from the Service despite running.

### A systematic troubleshooting order

```
1. kubectl get pods            -- what state is it actually in?
2. kubectl describe pod        -- Events section: what happened, in order?
3. kubectl logs [--previous]   -- what did the application itself say?
4. kubectl exec                -- verify config/connectivity directly, if needed
```

Working top-down through this order (state → events → application logs →
direct inspection) resolves the large majority of Pod issues without
guessing, since each step narrows the search space concretely rather than
speculating about the cause.

## When to use

- `kubectl describe` as the first troubleshooting step for almost any
  Pod issue — its Events section is the highest-signal, lowest-effort
  source of information.
- `kubectl logs --previous` specifically for crash-looping containers,
  since the current instance's logs may not reflect the actual failure.
- `kubectl apply` (over `create`) for all manifest management, to keep
  deployments idempotent and repeatable.

## When NOT to use

- Don't rely on `kubectl exec` as a primary debugging tool for production
  issues — prefer structured logs and metrics that persist and don't
  require an interactive session into a live container.
- Don't skip `describe` and jump straight to `logs` — many failures (like
  `Pending` or `ImagePullBackOff`) never produce application logs at all
  since the container never started.

## Common mistakes

- Reading `kubectl logs` on a crash-looping Pod without `--previous` and
  seeing only the (potentially already-restarted) current instance,
  missing the actual crash cause.
- Assuming a `Running` Pod is receiving traffic without checking
  `kubectl get endpoints` — label mismatches or failing readiness checks
  can leave a Running Pod invisible to its Service.
- Using `kubectl create` in scripts/pipelines where `kubectl apply` would
  be idempotent and safer for repeated runs.
- Not checking `kubectl describe pod` Events first, missing a scheduling
  or image-pull failure that never reaches the application logs stage at
  all.

## Interview questions

1. What's the difference between `kubectl apply` and `kubectl create`,
   and why does that matter for CI/CD pipelines?
2. Walk through your troubleshooting steps for a Pod stuck in
   `CrashLoopBackOff`.
3. A Service has Pods that are `Running`, but no traffic reaches them —
   what would you check first?
4. Why is `kubectl logs --previous` important, and when would you need
   it specifically?
5. What are the common causes of a Pod stuck in `Pending` state?

## Senior-level considerations

- Effective Kubernetes troubleshooting is fundamentally about knowing
  *where in the Pod lifecycle* a failure occurs (scheduling → image pull
  → container start → readiness → traffic) — experienced engineers
  narrow down the failing stage almost immediately from symptoms alone,
  rather than working through every command exhaustively.
- Relying heavily on `kubectl exec` for debugging is itself a signal of
  an observability gap — mature production setups should make most
  issues diagnosable through logs/metrics/traces alone
  (see [Observability](../../observability/index.md)), without needing an
  interactive session into a running container.
- In larger organizations, `kubectl` access itself is often restricted by
  RBAC to specific namespaces/verbs — understanding this is important
  both for troubleshooting under real access constraints and for
  designing sane operational access policies for a team.
