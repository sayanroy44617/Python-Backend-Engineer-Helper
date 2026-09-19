# Templates, values.yaml, and Helpers

## What

Helm **templates** are Kubernetes manifests written with Go template
syntax, parameterized by values from **`values.yaml`** (and overrides).
**Helpers** (`_helpers.tpl`) are reusable, named template snippets shared
across multiple manifest templates within a chart.

## Why

A chart intended for reuse across environments (dev/staging/prod) or
multiple deployments of the same application can't hard-code
environment-specific values (replica count, image tag, resource limits)
directly into its manifests — templating lets one set of manifest files
produce different rendered output per environment, driven entirely by
`values.yaml` overrides rather than by editing the manifests themselves.

## How

### A templated Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-api
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
```

Compare this to the raw manifest in
[Pods, Deployments, and Services](../kubernetes/01-pods-deployments-and-services.md)
— the structure is identical; every hard-coded value there is replaced
here with a `{{ .Values.* }}` reference resolved at render/install time.

### `values.yaml`

```yaml
replicaCount: 2

image:
  repository: myregistry/myapp
  tag: "1.4.0"

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

`values.yaml` is the chart's default configuration — installing with no
overrides uses these values exactly.

### Overriding values per environment

```bash
helm install myapp mychart/ -f values-prod.yaml
helm install myapp mychart/ --set replicaCount=5 --set image.tag=1.5.0
```

```yaml
# values-prod.yaml -- only overrides what differs from values.yaml
replicaCount: 5
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

An environment-specific values file only needs to specify the keys that
differ from the chart's defaults — Helm merges it over `values.yaml`,
not replaces it. This is the standard way one chart serves dev, staging,
and production with meaningfully different resource and scaling
configuration, without maintaining separate copies of the manifests.

### Built-in objects: `.Release`, `.Chart`, `.Files`

```yaml
metadata:
  name: {{ .Release.Name }}-api
  labels:
    chart-version: {{ .Chart.Version }}
    release: {{ .Release.Name }}
    revision: "{{ .Release.Revision }}"
```

`.Release` carries information about the specific install/upgrade
(name, revision number), `.Chart` carries `Chart.yaml` metadata, and
`.Files` gives access to non-template files bundled in the chart —
these are populated by Helm at render time, distinct from `.Values`
which comes from `values.yaml`/overrides.

### Control flow: conditionals and loops

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ . }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ $.Release.Name }}-api
                port: { number: 80 }
    {{- end }}
{{- end }}
```

`if`/`range` let a single template conditionally render a resource
entirely (e.g. skip the
[Ingress](../kubernetes/03-ingress-and-external-access.md) if
`ingress.enabled` is `false`) or repeat a block over a list of values
(multiple Ingress hosts) — this is what allows one chart to serve
configurations that need structurally different output, not just
different scalar values.

### Helpers (`_helpers.tpl`)

```yaml
{{/* templates/_helpers.tpl */}}
{{- define "myapp.fullname" -}}
{{- .Release.Name }}-{{ .Chart.Name }}
{{- end -}}

{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}
```

```yaml
# templates/deployment.yaml
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

Helpers avoid repeating the same naming/labeling logic across every
template file in a chart (Deployment, Service, ConfigMap all need
consistent labels) — defined once via `define`, then reused via
`include`, so a naming convention change only needs to happen in one
place.

### Rendering locally before installing

```bash
helm template mychart/ -f values-prod.yaml
```

Rendering locally (no cluster interaction) is the fastest way to verify
that a template change or values override produces the expected YAML —
the same technique mentioned in
[Charts and Chart Structure](01-charts-and-chart-structure.md) for
validating a chart before install.

## When to use

- Templating any value that legitimately differs across environments or
  deployments (replica count, image tag, resource sizing, feature flags).
- Helpers for any naming/labeling logic repeated across more than one
  template file in the same chart.
- `values.yaml` structured to mirror how the application is actually
  configured, rather than mirroring raw Kubernetes YAML structure
  one-to-one.

## When NOT to use

- Don't templatize values that never actually change — unnecessary
  `{{ .Values.* }}` indirection for a constant adds complexity without
  benefit.
- Don't duplicate the same labeling/naming logic across multiple
  templates when a helper would consolidate it into one definition.
- Don't nest `if`/`range` control flow so deeply that the rendered output
  becomes hard to reason about — favor `helm template` output review over
  guessing what a complex template produces.

## Common mistakes

- Hard-coding a value directly in a template that should have been
  parameterized through `values.yaml`, breaking reuse across
  environments.
- Forgetting `{{- ... -}}` whitespace control, producing malformed YAML
  from stray blank lines/indentation in the rendered output.
- Duplicating naming/labeling logic across every template file instead
  of factoring it into a shared helper in `_helpers.tpl`.
- Not rendering (`helm template`) and reviewing output before installing,
  discovering a templating bug only after a failed install against a
  live cluster.

## Interview questions

- What's the difference between `.Values`, `.Release`, and `.Chart` in
    a Helm template?

    **Answer:** `.Values` holds your configurable settings from `values.yaml`
    (or `--set`), `.Release` holds info about this specific install/upgrade
    (name, namespace, revision), and `.Chart` holds metadata about the chart
    itself (name, version). They're just different built-in objects Helm
    injects into every template.

    ```yaml
    name: {{ .Release.Name }}-{{ .Chart.Name }}
    image: {{ .Values.image.repository }}
    ```

- Why would you define a named template in `_helpers.tpl` instead of
    repeating the same logic in each manifest template?

    **Answer:** It's the DRY principle applied to Helm — write the logic
    (like a standard label block) once, then call it from every manifest with
    `{{ include "mychart.labels" . }}`. If the convention changes, you edit it
    in one place instead of every file.

- How do environment-specific values files (e.g. `values-prod.yaml`)
    interact with a chart's default `values.yaml`?

    **Answer:** Helm merges them, with later files overriding earlier ones —
    `values.yaml` provides the defaults and `values-prod.yaml` only needs to
    override what's different for production.

    ```bash
    helm upgrade my-app ./chart -f values.yaml -f values-prod.yaml
    ```

- What's `helm template` useful for, and when would you use it instead
    of `helm install --dry-run`?

    **Answer:** `helm template` renders manifests fully offline with no
    cluster connection at all, which makes it great for CI linting, diffing,
    or just reading output quickly. `--dry-run` actually talks to the API
    server (for validation/admission checks) so it's closer to a real install.

- How would you conditionally render an entire resource (like an
    Ingress) based on a value?

    **Answer:** Wrap the whole resource in an `{{- if .Values.ingress.enabled
    }}` / `{{- end }}` block so the file only produces output when the flag
    is true.

    ```yaml
    {{- if .Values.ingress.enabled }}
    kind: Ingress
    {{- end }}
    ```

## Senior-level considerations

- Designing a chart's `values.yaml` schema thoughtfully (grouping related
  configuration logically, providing sane defaults) matters as much as
  the templates themselves — a poorly structured values schema makes a
  chart hard to configure correctly even if the underlying templates are
  fine. For example, flat unrelated keys like `dbHost`, `enableCache`,
  `logLevel` at the top level are harder to reason about than nested
  `database:`, `cache:`, `logging:` blocks.
- Helm's Go templating has real limits (no first-class validation of
  `values.yaml` structure without an additional `values.schema.json`) —
  mature charts add a JSON Schema to catch invalid values overrides at
  install time rather than failing obscurely during template rendering.
  For example, a typo'd `replicaCont: 3` would otherwise be silently
  ignored instead of failing fast.
- Overusing control flow (deeply nested `if`/`range`) inside templates
  trades initial flexibility for long-term maintainability — many teams
  prefer several simpler charts over one highly conditional
  "one chart fits all environments" chart. For example, a chart with 5
  nested `if`s to support dev/staging/prod/canary/DR becomes very hard
  to debug when a rendering bug only shows up in one combination.
