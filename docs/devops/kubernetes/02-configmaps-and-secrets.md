# ConfigMaps and Secrets

## What

A **ConfigMap** stores non-sensitive configuration data as key-value
pairs, injectable into Pods as environment variables or mounted files. A
**Secret** stores sensitive data the same way, with additional handling
conventions (though not automatic encryption by default) reflecting its
more sensitive nature.

## Why

Baking configuration into a container image (as covered generally in
[Volumes, Networks, and Environment Variables](../docker/02-volumes-networks-and-environment-variables.md))
means rebuilding the image for every environment. ConfigMaps and Secrets
let the same image run correctly across dev/staging/production by
injecting environment-specific configuration and credentials at
deployment time instead.

## How

### Creating a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: "info"
  FEATURE_FLAG_NEW_CHECKOUT: "true"
```

### Consuming a ConfigMap as environment variables

```yaml
spec:
  containers:
    - name: api
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: api-config
```

`envFrom` injects every key in the ConfigMap as an environment variable
in the container — the application reads them exactly as it would any
other environment variable (`os.environ["LOG_LEVEL"]`), unaware they came
from Kubernetes specifically.

### Consuming a ConfigMap as a mounted file

```yaml
spec:
  containers:
    - name: api
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: api-config
```

Mounting as a file (rather than environment variables) is useful for
larger configuration (a full config file, a set of feature flags as
JSON) or when the application already expects to read configuration from
a file path rather than environment variables.

### Creating a Secret

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=appuser \
  --from-literal=password=supersecret
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YXBwdXNlcg==   # base64-encoded, NOT encrypted
  password: c3VwZXJzZWNyZXQ=
```

### Secrets are base64-encoded, not encrypted, by default

```bash
echo "YXBwdXNlcg==" | base64 -d   # appuser -- trivially reversible
```

A raw Kubernetes Secret's `data` field is only base64-*encoded*, not
encrypted — anyone with API access to read the Secret object can decode
it instantly. This directly parallels the point made in
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md)
about JWTs: encoding is not encryption. Real protection requires
additional layers — RBAC restricting who/what can read Secret objects,
and encryption at rest (etcd encryption) configured at the cluster level.

### Consuming a Secret

```yaml
spec:
  containers:
    - name: api
      envFrom:
        - secretRef:
            name: db-credentials
```

Functionally identical to consuming a ConfigMap — the distinction between
ConfigMap and Secret is primarily about *intent and access control*
conventions (RBAC rules commonly restrict Secret access more tightly),
not a difference in the underlying mechanism.

### External secrets managers

```yaml
# Using an operator (e.g. External Secrets Operator) to sync a real
# secrets manager's values into a Kubernetes Secret automatically
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

For anything beyond casual/dev use, most production clusters sync
Secrets from a real secrets manager (AWS Secrets Manager, HashiCorp
Vault — see
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md))
rather than storing the source of truth directly as a Kubernetes Secret
— this centralizes rotation, auditing, and access control in the
dedicated secrets system rather than duplicating that logic in
Kubernetes-native tooling.

### RBAC restricting Secret access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["db-credentials"]
```

Restricting exactly which service accounts/users can read which specific
Secrets (rather than granting broad `secrets: get` cluster-wide) is the
Kubernetes-level application of
[least privilege](../../security/03-authorization-and-least-privilege.md)
— without it, any workload/user with general Secret-read access can read
every Secret in the namespace, not just the ones it actually needs.

### Never commit raw Secret manifests to version control

```
A Secret manifest with `data:` populated is functionally a plaintext
credential, one base64 decode away -- the same "never commit a secret to
git" rule from Injection Attacks and Secrets Management applies directly.
```

Tools like Sealed Secrets or SOPS encrypt Secret values so the encrypted
form is safe to commit to version control (a common GitOps requirement),
decrypting only inside the cluster — committing a raw, unencrypted
Secret manifest is exactly the "secret committed to git" mistake covered
in
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md).

## When to use

- ConfigMaps for non-sensitive, environment-specific configuration
  (feature flags, log levels, non-secret URLs).
- Secrets (ideally backed by a real secrets manager via an operator) for
  credentials, API keys, and any sensitive value.
- RBAC rules scoped to specific Secret names rather than broad
  namespace-wide access.

## When NOT to use

- Don't treat a raw Kubernetes Secret as sufficiently protected on its
  own — base64 encoding provides no real confidentiality without RBAC and
  etcd encryption at rest also in place.
- Don't commit populated Secret manifests to version control without
  encrypting them first (Sealed Secrets, SOPS, or equivalent).
- Don't use a ConfigMap for sensitive data just because it's simpler to
  set up — the distinction with Secret exists specifically to signal and
  enforce different access-control treatment.

## Common mistakes

- Assuming base64 encoding in a Secret provides real security, without
  additional RBAC restrictions or etcd encryption at rest.
- Committing raw Secret YAML manifests (with real values) to a Git
  repository, effectively leaking the credential into version history.
- Granting overly broad RBAC permissions (`secrets: get` cluster-wide)
  instead of scoping access to the specific Secrets a workload actually
  needs.
- Storing the source of truth for credentials directly as Kubernetes
  Secrets instead of syncing from a dedicated secrets manager, losing
  centralized rotation and audit capabilities.

## Interview questions

1. Why is a Kubernetes Secret's base64 encoding not equivalent to
   encryption, and what additional layers actually protect it?
2. What's the practical (not just naming) difference between a ConfigMap
   and a Secret?
3. How would you avoid committing real secret values to version control
   while still managing Kubernetes manifests via GitOps?
4. Why would a production cluster typically sync Secrets from an external
   secrets manager rather than storing the source of truth as native
   Kubernetes Secrets?
5. How does RBAC apply the principle of least privilege specifically to
   Secret access?

## Senior-level considerations

- Treating "it's a Kubernetes Secret" as sufficient protection is a
  common, dangerous misconception — real secret security in Kubernetes
  requires RBAC, etcd encryption at rest, and often an external secrets
  manager working together, not the Secret object type alone.
- GitOps workflows (declarative manifests as the source of truth in Git)
  create a direct tension with secret management — solving this properly
  (Sealed Secrets, SOPS, or external-secrets syncing) is a common
  real-world Kubernetes architecture decision, not an edge case.
- Least-privilege RBAC scoping for Secrets specifically (not just broad
  namespace access) is one of the highest-value, most commonly
  under-implemented security controls in real Kubernetes clusters.
