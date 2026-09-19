# Releases and Release Management

## What

A Helm **release** is a specific, named instance of a chart installed
into a cluster, with its own revision history — installing the same
chart twice (with different release names) produces two independent
releases, each independently upgradeable and rollback-able.

## Why

Kubernetes itself has no built-in concept of "this set of manifests, as
installed at this point in time, versioned and reversible as a unit" —
`kubectl apply` just reconciles individual objects to match a file, with
no memory of prior states as a cohesive whole. Releases give that
missing layer: every `helm upgrade` records a new revision, making
"what changed" and "roll back to before that change" first-class
operations instead of manual `kubectl` archaeology.

## How

### Installing and naming a release

```bash
helm install myapp-prod mychart/ -f values-prod.yaml
helm install myapp-staging mychart/ -f values-staging.yaml
```

`myapp-prod` and `myapp-staging` are independent releases of the same
chart — separate revision histories, separate values, typically
installed into separate namespaces to keep them fully isolated.

### Upgrading a release

```bash
helm upgrade myapp-prod mychart/ -f values-prod.yaml --set image.tag=1.5.0
```

`helm upgrade` diffs the newly rendered manifests against the
currently installed ones and applies the changes — conceptually similar
to the rolling update mechanics in
[Pods, Deployments, and Services](../kubernetes/01-pods-deployments-and-services.md),
but operating at the level of the whole chart's resources rather than a
single Deployment.

### Release history and rollback

```bash
helm history myapp-prod
# REVISION  UPDATED       STATUS       CHART          APP VERSION
# 1         ...           superseded   myapp-1.3.0    2.0.0
# 2         ...           superseded   myapp-1.4.0    2.1.0
# 3         ...           deployed     myapp-1.4.0    2.1.0

helm rollback myapp-prod 2
```

Every `helm install`/`helm upgrade` creates a new revision, retained in
history — `helm rollback` reverts to a previous revision's exact
rendered manifests. This is Helm's version of the `kubectl rollout undo`
mechanic covered in
[Pods, Deployments, and Services](../kubernetes/01-pods-deployments-and-services.md),
but applied to everything the chart manages (Deployment, ConfigMap,
Ingress, and so on together), not just one Deployment.

### `--atomic` and safe upgrades

```bash
helm upgrade myapp-prod mychart/ --atomic --timeout 5m
```

`--atomic` automatically rolls back to the previous revision if the
upgrade fails (e.g. a new Pod never becomes Ready within the timeout) —
without it, a failed upgrade can leave a release in a partially applied,
inconsistent state requiring manual intervention.

### Uninstalling a release

```bash
helm uninstall myapp-staging
helm uninstall myapp-staging --keep-history   # retain revision history for reference
```

Uninstalling removes all resources the release created — by default
also the revision history, unless `--keep-history` is passed to retain
it for later inspection (e.g. auditing what was previously deployed).

### Checking release status

```bash
helm status myapp-prod
helm list                    # all releases in the current namespace
helm list --all-namespaces
```

`helm list`/`helm status` are the equivalent of `kubectl get
deployments`/`kubectl describe` at the release level — the quickest way
to see what's currently installed, its chart version, and its status
across a cluster or namespace.

### Releases and CI/CD

```bash
helm upgrade --install myapp-prod mychart/ -f values-prod.yaml --atomic
```

`--install` makes the command idempotent — install if the release
doesn't exist yet, upgrade if it does — which is the standard pattern in
a CI/CD pipeline deploying to Kubernetes (see
[Deployment and Rollback Strategies](../cicd/04-deployment-and-rollback-strategies.md)
for the broader pipeline context this step fits into), since the
pipeline shouldn't need to know in advance whether this is the first
deployment or the hundredth.

## When to use

- Separate, named releases per environment (or per tenant, if
  multi-tenant) of the same chart, rather than one release reused across
  environments.
- `--atomic` for any upgrade in a production pipeline, to avoid leaving a
  release in a broken, half-applied state on failure.
- `helm rollback` as the fast path to recover from a bad deployment,
  before investigating root cause.

## When NOT to use

- Don't reuse a single release name across genuinely different
  environments/configurations — it conflates unrelated deployments'
  revision histories and makes rollback ambiguous.
- Don't skip `--atomic`/timeouts on production upgrades assuming they'll
  always succeed — a failed upgrade without atomic rollback can leave
  the cluster in a state requiring manual cleanup.
- Don't rely on `helm rollback` as a substitute for actually fixing the
  root cause of a bad release — it's a fast mitigation, not a resolution.

## Common mistakes

- Installing the same chart into the same namespace under the same
  release name for what are actually different logical environments,
  causing confusing, intertwined revision history.
- Running `helm upgrade` without `--atomic`/`--timeout` in production,
  leaving a failed upgrade partially applied.
- Forgetting `--install` in CI/CD scripts, causing the pipeline to fail
  on first deploy (`upgrade` alone fails if the release doesn't exist
  yet).
- Assuming `helm uninstall` retains history by default — it doesn't,
  unless `--keep-history` is explicitly passed.

## Interview questions

- What does a Helm "release" represent that a plain `kubectl apply`
    doesn't track?

    **Answer:** A release is a named, versioned history of everything Helm
    applied for that install — every upgrade creates a new revision Helm
    remembers, so you can see what changed and roll back. Plain `kubectl
    apply` has no built-in history at all.

- How does `helm rollback` work, and what's the relationship between
    revisions and rollback?

    **Answer:** Helm stores the rendered manifests for every past revision;
    `helm rollback <release> <revision>` just re-applies an older revision's
    manifests as a new revision. Nothing is "undone" magically — it's a
    forward apply of old state.

    ```bash
    helm rollback my-app 3
    ```

- Why is `--atomic` important for production upgrades, and what does it
    actually do on failure?

    **Answer:** `--atomic` tells Helm to automatically roll back to the
    previous working revision if the upgrade fails or times out, so you don't
    get stuck with half-applied, broken state in production.

    ```bash
    helm upgrade my-app ./chart --atomic --timeout 5m
    ```

- Why is `helm upgrade --install` the standard pattern in CI/CD
    pipelines rather than `helm install` alone?

    **Answer:** `--install` makes the command idempotent — it installs if the
    release doesn't exist yet, or upgrades it if it does. That means the same
    pipeline command works for both the first deploy and every deploy after.

- What's the difference between chart version, application version, and
    release revision?

    **Answer:** Chart version (`version`) tracks the templates/values
    changing, app version (`appVersion`) documents which app version is
    deployed, and release revision is simply an incrementing counter Helm
    keeps every time you install/upgrade/rollback that specific release.

## Senior-level considerations

- Release naming and namespace strategy (one release per environment,
  isolated namespaces) is a foundational decision that's expensive to
  change later — get this right early rather than retrofitting isolation
  onto an already-tangled set of releases. For example, migrating from one
  shared namespace with `my-app-dev`/`my-app-prod` releases to separate
  namespaces later means re-provisioning RBAC and network policies too.
- `--atomic` upgrades trade a slower failure path (waiting for the full
  timeout before rolling back) for much safer production behavior — this
  is almost always the right trade-off for anything user-facing. For
  example, a 5-minute timeout means a bad rollout stays broken for at most
  5 minutes before Helm reverts it automatically.
- Relying purely on `helm rollback` for incident response without also
  tracking *why* a release failed (via post-incident review) risks
  repeating the same failure on the next upgrade — rollback is
  mitigation, not root-cause resolution. For example, rolling back a bad
  config change without fixing the underlying CI validation gap means the
  same bad config can be pushed again next week.
