# Secrets and Configuration Management

## What

Managing **Secrets** and **Configuration** through Helm means deciding
how sensitive and non-sensitive values flow from `values.yaml`/overrides
into the [ConfigMap and Secret](../kubernetes/02-configmaps-and-secrets.md)
objects a chart renders — and, for real secret values, how to avoid
putting them in plaintext anywhere in the chart or its version history.

## Why

A chart's templates make it trivial to render a
[ConfigMap or Secret](../kubernetes/02-configmaps-and-secrets.md) from
`values.yaml` — but that convenience creates a real risk: it's just as
easy to accidentally commit a real credential into a `values.yaml`
override file tracked in Git as it is to commit any other secret,
covered generally in
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md).
Helm-specific tooling and conventions exist specifically to avoid that
outcome while still keeping secret values templated and environment
aware.

## How

### Templating a ConfigMap from values

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  FEATURE_FLAG_NEW_CHECKOUT: {{ .Values.config.newCheckoutEnabled | quote }}
```

```yaml
# values.yaml
config:
  logLevel: info
  newCheckoutEnabled: "false"
```

Non-sensitive configuration flows through `values.yaml` exactly like any
other value — this is a direct application of the general ConfigMap
mechanics in
[ConfigMaps and Secrets](../kubernetes/02-configmaps-and-secrets.md),
just templated instead of hard-coded.

### Templating a Secret — and why not to put real values in `values.yaml`

```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-db-credentials
type: Opaque
data:
  password: {{ .Values.db.password | b64enc }}
```

Rendering the *structure* of a Secret from a template is fine — the
problem is where `.Values.db.password`'s actual value comes from. If
it's set in a `values.yaml`/`values-prod.yaml` file committed to Git,
that's the same "committed a real secret to version control" mistake
covered in
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md)
— Helm makes the Secret *rendering* trivial without automatically
solving *where the real value safely lives*.

### `--set` for one-off, non-committed secret values

```bash
helm upgrade --install myapp mychart/ \
  --set db.password="$DB_PASSWORD"
```

Passing a secret value at install/upgrade time via `--set` (sourced from
an environment variable or a CI/CD secret store, never written to disk
in the repo) avoids ever committing the real value — but it does mean
the value must be supplied correctly on every install/upgrade, and it
can still appear in shell history or process listings if not handled
carefully.

### Encrypted values with Helm Secrets / SOPS

```bash
helm secrets encrypt values-prod-secrets.yaml > values-prod-secrets.enc.yaml
helm secrets upgrade myapp mychart/ -f values-prod-secrets.enc.yaml
```

The `helm-secrets` plugin (backed by SOPS) lets an encrypted values file
be safely committed to Git — decrypting only at install/upgrade time,
using keys the plugin is configured to access (e.g. a cloud KMS key).
This mirrors the same Sealed Secrets/SOPS pattern mentioned in
[ConfigMaps and Secrets](../kubernetes/02-configmaps-and-secrets.md) for
raw Kubernetes Secret manifests, applied at the Helm values layer
instead.

### Referencing an externally managed Secret instead of templating one

```yaml
# templates/deployment.yaml
envFrom:
  - secretRef:
      name: {{ .Values.db.existingSecret | default (printf "%s-db-credentials" (include "myapp.fullname" .)) }}
```

```yaml
# values.yaml
db:
  existingSecret: ""   # if set, use this pre-existing Secret instead of templating one
```

Many production charts support an `existingSecret` override, letting the
Secret itself be created and managed *outside* the chart entirely — for
example, synced from a real secrets manager via an operator, as
described in
[ConfigMaps and Secrets](../kubernetes/02-configmaps-and-secrets.md) —
while the chart's Deployment still just references it by name. This
decouples "how the chart consumes credentials" from "where credentials
actually come from," which is usually the right separation of concerns
in production.

### Environment-specific configuration layering

```bash
helm upgrade --install myapp mychart/ \
  -f values.yaml \
  -f values-prod.yaml \
  -f values-prod-secrets.enc.yaml
```

Layering multiple `-f` files (base defaults, environment overrides,
encrypted secrets) lets non-sensitive and sensitive configuration be
managed with different tooling and access controls, while still
composing into one coherent install — non-secret values reviewed freely
in pull requests, secret values readable only through the decryption
path.

## When to use

- `values.yaml`/environment override files for all non-sensitive
  configuration — logging levels, feature flags, resource sizing.
- `existingSecret`-style overrides for any chart intended for real
  production use, so Secret lifecycle can be managed independently of
  the chart (e.g. by an external secrets operator).
- Encrypted values files (`helm-secrets`/SOPS) or CI/CD-injected `--set`
  values for any real secret that must flow through a chart install.

## When NOT to use

- Don't put real secret values directly in a `values.yaml`/override file
  committed to version control, even if it's Helm-templated — the
  encoding/templating doesn't change the fact that it's a plaintext
  credential in Git history.
- Don't hard-code secret values as chart defaults "for convenience" —
  defaults are exactly what get copied into other environments/charts
  unnoticed.
- Don't rely solely on `--set` for secrets in automated pipelines without
  ensuring the value itself comes from a proper secret store rather than
  a plaintext CI/CD variable.

## Common mistakes

- Committing a `values-prod.yaml` with real credentials in plaintext,
  assuming Helm's templating provides some inherent protection it
  doesn't.
- Not supporting an `existingSecret`-style override in an internal chart,
  forcing every consumer to either accept the chart's own Secret
  templating or fork the chart.
- Forgetting that `--set` values may be visible in shell history, CI
  logs, or process listings if not sourced and handled carefully.
- Treating base64 encoding inside a templated Secret as sufficient
  protection on its own, repeating the misconception already covered in
  [ConfigMaps and Secrets](../kubernetes/02-configmaps-and-secrets.md).

## Interview questions

1. Why doesn't templating a Secret through Helm solve the problem of
   safely storing its real value?

   **Answer:** Helm still needs the actual secret value from somewhere
   (a values file, `--set`, or an env var) to render the template, and that
   source itself — plain-text in Git, shell history, CI logs — is where the
   real risk lives. Templating just moves the value into YAML; it doesn't
   protect it.

2. What's the purpose of an `existingSecret`-style override in a
   production-grade chart?

   **Answer:** It lets the chart reference a Secret that already exists in
   the cluster (created by an external secrets operator or manually) instead
   of forcing users to pass raw secret values through Helm's own values.

   ```yaml
   database:
     existingSecret: "db-credentials"
   ```

3. How does `helm-secrets`/SOPS allow encrypted values to be safely
   committed to version control?

   **Answer:** SOPS encrypts specific values in a YAML file using a KMS key,
   so the committed file is unreadable without access to that key. `helm
   secrets` transparently decrypts it in memory right before Helm renders
   the chart.

   ```bash
   helm secrets upgrade my-app ./chart -f secrets.enc.yaml
   ```

4. What are the risks of passing a secret value via `--set` in a CI/CD
   pipeline, and how would you mitigate them?

   **Answer:** `--set` values often end up visible in shell history, process
   lists, and CI logs. Mitigate by using `--set-file` pointed at a file the
   CI system injects securely, or by using `existingSecret`/an operator so
   Helm never sees the raw value at all.

5. How would you layer base, environment-specific, and secret values
   files together for a single `helm upgrade` command?

   **Answer:** Pass multiple `-f` flags in order from least to most specific
   — Helm merges them left to right, so the last file wins on overlapping
   keys.

   ```bash
   helm upgrade my-app ./chart \
     -f values.yaml -f values-prod.yaml -f secrets.enc.yaml
   ```

## Senior-level considerations

- A well-designed internal chart cleanly separates "how the application
  consumes a Secret" (an `envFrom`/volume reference) from "where that
  Secret's real value comes from" (`existingSecret`, an external secrets
  operator, or an encrypted values file) — conflating the two is one of
  the most common causes of production secret-handling incidents. For
  example, hard-coding a chart to only accept raw values via `--set` makes
  it impossible to later switch to an external secrets operator without
  rewriting the templates.
- Encrypted-values tooling (`helm-secrets`/SOPS) versus dedicated
  external-secrets operators represent different points on the same
  trade-off: values-file encryption keeps everything in the chart/Git
  workflow, while an operator centralizes secret lifecycle management
  outside of Helm entirely — larger organizations often prefer the
  latter for auditability and rotation. For example, an operator can
  auto-rotate a database password and update the Secret without anyone
  touching Helm at all.
- Reviewing what actually ends up rendered (`helm template`) for any
  chart touching secrets is a cheap, high-value habit — it's the
  fastest way to confirm a templating change didn't inadvertently
  expose or misconfigure a Secret before it reaches a real cluster. For
  example, a misplaced `{{ .Values.dbPassword }}` in a ConfigMap instead
  of a Secret would show up immediately in the rendered output.
