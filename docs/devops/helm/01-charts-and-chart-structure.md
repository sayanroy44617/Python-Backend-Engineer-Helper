# Charts and Chart Structure

## What

A **Helm chart** is a packaged, versioned bundle of Kubernetes manifests
— templates plus default configuration — that can be installed,
upgraded, and uninstalled as a single unit via the `helm` CLI.

## Why

A real application's Kubernetes footprint is rarely one manifest — a
[Deployment, Service](../kubernetes/01-pods-deployments-and-services.md),
[ConfigMap, Secret](../kubernetes/02-configmaps-and-secrets.md), and
[Ingress](../kubernetes/03-ingress-and-external-access.md) together, at
minimum. Managing these as separate `kubectl apply -f` files makes
versioning, environment-specific configuration (dev vs. staging vs.
prod), and rollback all manual and error-prone. A chart bundles them
into one installable, versioned artifact with a single command
(`helm install`/`helm upgrade`) replacing many.

## How

### Standard chart directory layout

```
mychart/
├── Chart.yaml          # chart metadata: name, version, appVersion
├── values.yaml          # default configuration values
├── templates/           # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl     # reusable template snippets
│   └── NOTES.txt        # printed after install/upgrade
├── charts/              # bundled subchart dependencies
└── .helmignore
```

`helm create mychart` scaffolds this structure with a working example
Deployment/Service — most real charts start from this scaffold rather
than being built from scratch.

### `Chart.yaml`

```yaml
apiVersion: v2
name: myapp
description: A Python backend API
version: 1.4.0        # chart version -- bumped on any chart change
appVersion: "2.1.0"    # version of the application itself
```

`version` and `appVersion` are deliberately separate: `version` tracks
changes to the chart's templates/structure, while `appVersion` tracks
the application's own version (commonly matching the container image
tag) — a chart can bump its `version` for a templating fix without any
change to the application itself, and vice versa.

### Chart dependencies (subcharts)

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

```bash
helm dependency update   # fetches the dependency charts into charts/
```

Rather than writing Kubernetes manifests for a database or cache from
scratch, applications commonly depend on a well-maintained public chart
(e.g. Bitnami's PostgreSQL/Redis charts) as a subchart — the `condition`
key lets consumers of the parent chart toggle the dependency on/off via
their own `values.yaml`.

### Packaging and distributing a chart

```bash
helm package mychart/              # produces myapp-1.4.0.tgz
helm repo index . --url https://charts.example.com

helm repo add myrepo https://charts.example.com
helm install myapp myrepo/myapp --version 1.4.0
```

A packaged chart (`.tgz`) hosted in a chart repository (a plain static
file server serving an `index.yaml`, or an OCI registry) is how charts
are distributed and versioned across teams/environments, analogous to
how [container images are pushed to a registry](../docker/01-images-containers-and-dockerfiles.md).

### Linting and validating a chart

```bash
helm lint mychart/
helm template mychart/             # renders manifests locally, no cluster needed
helm install --dry-run --debug myapp mychart/
```

`helm template` and `--dry-run` render the final Kubernetes YAML without
touching a real cluster — the standard way to sanity-check a chart's
output (and catch templating mistakes) before actually installing or
upgrading anything.

## When to use

- Any application with more than one or two Kubernetes manifests that
  need to be deployed/versioned together.
- Reusing existing, well-maintained charts (databases, caches, ingress
  controllers) as dependencies instead of hand-writing their manifests.
- Distributing an internal application's deployment configuration across
  multiple teams/environments in a versioned, repeatable way.

## When NOT to use

- Don't reach for Helm for a single, static manifest that never varies
  across environments — plain `kubectl apply` or a lighter templating
  tool (e.g. Kustomize) may be simpler.
- Don't hand-maintain manifests for common infrastructure (databases,
  message brokers) that already have a well-maintained public chart
  available as a subchart dependency.

## Common mistakes

- Not separating chart `version` from application `appVersion`,
  conflating a templating change with an actual application release.
- Skipping `helm lint`/`helm template` before installing, discovering
  templating errors only after a failed (or worse, partially applied)
  install against a real cluster.
- Hand-writing manifests for common dependencies (Postgres, Redis)
  instead of using an existing, well-tested chart as a subchart.
- Committing an unpackaged chart directory without ever versioning
  releases, losing the ability to reliably roll back to a known-good
  chart version.

## Interview questions

1. What's the practical difference between a chart's `version` and
   `appVersion` fields?
2. Why would a team use an existing public chart as a subchart dependency
   rather than writing the manifests themselves?
3. How would you validate a chart's rendered output before actually
   installing it against a cluster?
4. What problem does bundling multiple related Kubernetes manifests into
   one chart solve that managing them as separate files doesn't?
5. How are charts typically packaged and distributed across teams?

## Senior-level considerations

- Chart versioning discipline (bumping `version` deliberately, keeping
  `appVersion` accurate) is what makes chart-based deployments reliably
  auditable and reversible — sloppy versioning erodes exactly the benefit
  Helm is meant to provide.
- Deciding when to depend on a public subchart vs. writing manifests
  in-house is a real trade-off between maintenance burden and control —
  public charts (e.g. Bitnami's) sometimes lag behind upstream releases
  or make configuration choices that don't fit a given deployment.
- Chart repository/registry choice (a static file server vs. an OCI
  registry) affects how charts integrate with existing artifact
  management and CI/CD tooling — increasingly, teams standardize on OCI
  registries to unify container image and chart distribution.
