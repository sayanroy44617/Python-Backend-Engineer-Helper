# Docker Builds and Image Registries

## What

Building a container **image** as a pipeline stage, tagging it
immutably, and pushing it to an **image registry** — the standard way a
packaged application artifact (see
[Pipeline Stages](02-pipeline-stages-build-test-and-package.md)) becomes
something a cluster can actually pull and run.

## Why

A pipeline that runs tests but never produces a deployable artifact
isn't complete — for a containerized backend service, that artifact is a
container image. Building it automatically and consistently (not by a
developer manually running `docker build` on their laptop) and pushing
it to a registry the deployment environment can reach is what connects
"code passed CI" to "this exact, tested build is what gets deployed."

## How

### Building in CI

```yaml
# .github/workflows/ci.yml (conceptual, after tests pass)
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: myregistry/myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

This builds from the same
[multi-stage Dockerfile](../docker/04-multi-stage-builds-for-python.md)
used for local development — the pipeline just automates and enforces
running it consistently, on every relevant commit, rather than manually.

### Tagging strategy: immutable, traceable tags

```bash
docker build -t myregistry/myapp:${GITHUB_SHA} .
docker build -t myregistry/myapp:v1.4.0 .
```

```
BAD:  myregistry/myapp:latest       -- ambiguous, mutable, not traceable
GOOD: myregistry/myapp:a1b2c3d      -- exact commit, always reproducible
GOOD: myregistry/myapp:v1.4.0       -- exact release, human-readable
```

Tagging every build with an immutable reference (commit SHA and/or
semantic version) — never relying on `:latest` for anything beyond local
experimentation — is what makes
[rollback](04-deployment-and-rollback-strategies.md) possible at all:
rolling back means deploying a specific previous tag, which only works
if that tag unambiguously points to one specific, immutable build.

### Multi-architecture builds

```yaml
- uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: myregistry/myapp:${{ github.sha }}
```

Building for multiple CPU architectures in the pipeline (rather than
only the architecture a developer's laptop happens to use) matters once
production runs on a different architecture (e.g. ARM-based cloud
instances) than local development machines — `docker buildx` handles
this transparently via QEMU emulation or native multi-arch runners.

### Pushing to a registry

```bash
docker login myregistry.example.com
docker push myregistry/myapp:a1b2c3d
```

```
Options:
- Cloud-provider registry (ECR, GCR, ACR) -- tightly integrated with
  that cloud's IAM and deployment tooling
- Self-hosted registry (Harbor)          -- full control, more ops burden
- Public registry (Docker Hub, GHCR)     -- simplest, less access control
```

The registry choice affects authentication/authorization (who can push,
who can pull), vulnerability scanning integration, and how tightly it
integrates with the deployment platform — cloud-provider registries are
usually the path of least resistance when already deployed on that
cloud's Kubernetes offering.

### Image scanning as a pipeline gate

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myregistry/myapp:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: 1  # fail the pipeline on findings
```

Scanning the built image for known vulnerabilities (in the base image
and installed dependencies) before pushing/deploying it catches issues
that pinned, locked dependencies alone don't — see
[Environments and Installers](../../packaging/01-environments-and-installers.md)
for why pinned dependencies matter for reproducibility; scanning is the
complementary check for known *vulnerabilities* in those exact pinned
versions, enforced as an automated pipeline gate rather than a manual
review step.

### Least-privilege registry credentials

```yaml
permissions:
  contents: read
  packages: write   # only what this specific job needs
```

Scoping the pipeline's registry credentials to exactly the permissions
needed (push access to one specific image repository, not broad
registry-admin access) is the CI/CD-pipeline application of
[least privilege](../../security/03-authorization-and-least-privilege.md)
— a compromised pipeline credential should be able to do as little
damage as possible.

### Signing images for supply-chain integrity

```bash
cosign sign --key cosign.key myregistry/myapp:a1b2c3d
```

Cryptographically signing built images (e.g. with Sigstore/Cosign) lets
a deployment environment verify an image actually came from the expected
pipeline and hasn't been tampered with — an increasingly common
requirement for supply-chain security in regulated or security-sensitive
environments.

## When to use

- Immutable tags (commit SHA and/or semantic version) for every image
  build — never `:latest` for anything deployed to a real environment.
- Automated vulnerability scanning as a pipeline gate before an image is
  pushed or deployed.
- Multi-architecture builds when production and development environments
  run on different CPU architectures.

## When NOT to use

- Don't use `:latest` as a deployment tag in any environment beyond local
  experimentation — it makes rollback and auditing which build is
  actually running effectively impossible.
- Don't skip image scanning "to save pipeline time" for anything
  deployed beyond a local/throwaway environment — the cost of a shipped
  known vulnerability far exceeds the scan's runtime.
- Don't grant pipeline credentials broader registry access than the
  specific push/pull the job actually needs.

## Common mistakes

- Deploying `:latest` and later being unable to determine exactly which
  code is running in production, or to roll back precisely.
- Skipping vulnerability scanning in the pipeline, only discovering a
  known-vulnerable base image or dependency after it's already deployed.
- Granting a CI/CD pipeline broad, long-lived registry credentials
  instead of scoped, short-lived tokens tied to the specific job.
- Not caching Docker build layers in CI, causing every pipeline run to
  rebuild from scratch and needlessly slowing down the pipeline.

## Interview questions

1. Why is tagging images with `:latest` for production deployments
   considered a mistake?

   **Answer:** `:latest` is mutable — it can point to a different image
   tomorrow than it does today, so "redeploy the same version" or "roll
   back" becomes ambiguous. You want an immutable tag (a commit SHA or
   semantic version) that always refers to exactly one image.

   ```bash
   docker build -t my-api:1.4.0 .
   docker build -t my-api:$(git rev-parse --short HEAD) .
   ```

2. What does image scanning in a pipeline catch that source-level
   dependency scanning might miss?

   **Answer:** Image scanning checks the full built artifact — including the
   base OS packages and anything installed at build time — not just your
   application's declared dependencies, so it catches OS-level CVEs that
   source scanning never sees.

3. Why would a team build multi-architecture images, and how is that
   typically automated?

   **Answer:** Multi-arch images (amd64 + arm64) let the same image run on
   different hardware — cheaper ARM cloud instances or Apple Silicon dev
   machines — without maintaining separate builds. `docker buildx` automates
   building and pushing a single manifest covering both architectures.

   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 -t my-api:1.4.0 --push .
   ```

4. How does least privilege apply specifically to CI/CD pipeline
   registry credentials?

   **Answer:** The pipeline's registry credential should only be able to
   push to the specific repositories it needs, not admin-level access to the
   whole registry — so a compromised pipeline can't overwrite or delete
   unrelated images.

5. What's the purpose of image signing (e.g. with Cosign) in a
   supply-chain security context?

   **Answer:** Signing lets a cluster or deployment tool cryptographically
   verify an image was actually built by your trusted pipeline and hasn't
   been tampered with or swapped for a malicious one before deployment.

## Senior-level considerations

- Registry and tagging strategy decisions made early (immutable tags,
  registry choice, scanning gates) are foundational to reliable
  rollback and incident response later — retrofitting immutable tagging
  onto a system built around `:latest` is disruptive and error-prone.
  For example, switching from `:latest` to SHA-based tags mid-project
  means auditing every place a deployment config still references
  `:latest`.
- Image scanning and signing are part of a broader supply-chain security
  posture, not standalone checkboxes — they matter most when combined
  with pinned, reproducible dependencies
  ([Environments and Installers](../../packaging/01-environments-and-installers.md))
  and least-privilege pipeline credentials working together. For
  example, a signed image built from unpinned dependencies can still
  silently include a newly-compromised transitive package.
- Build caching and multi-architecture support are largely solved
  problems with mature tooling (`docker buildx`, registry-native
  caching) — reinventing them with custom scripts is rarely worth the
  maintenance cost compared to using well-supported existing tooling.
  For example, a hand-rolled layer-caching script is one CI runner
  upgrade away from silently breaking, while `buildx`'s registry cache
  is maintained upstream.
