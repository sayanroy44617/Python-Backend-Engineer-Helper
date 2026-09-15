# Interview Prep: DevOps

## How to approach DevOps interview questions

DevOps questions for a backend engineering role (as opposed to a
dedicated infrastructure/SRE role) usually focus on whether you
understand the deployment pipeline your own code travels through —
containerization, orchestration basics, and CI/CD — rather than deep
Kubernetes cluster administration. The
[DevOps](../devops/index.md) section (Docker, Kubernetes, Helm, CI/CD)
covers the mechanics; this page distills the questions most likely to
come up for an application-engineer-level interview.

## Rapid-fire questions

| # | Question | Key idea | Deep dive |
|---|----------|----------|-----------|
| 1 | Why use a multi-stage Dockerfile for a Python application? | It separates the build environment (compilers, dev dependencies) from the runtime image, so the final image ships only what's needed to run the app — smaller, faster to pull, and with a reduced attack surface. | [Multi-Stage Builds for Python](../devops/docker/04-multi-stage-builds-for-python.md) |
| 2 | What's the difference between a Docker volume and a bind mount? | A named volume is managed by Docker and lives outside any specific container's filesystem, portable across container recreation; a bind mount maps a specific host path directly into the container, useful for local development but tying the container to the host's filesystem layout. | [Volumes, Networks, and Environment Variables](../devops/docker/02-volumes-networks-and-environment-variables.md) |
| 3 | What's the difference between a Kubernetes Pod, Deployment, and Service? | A Pod is the smallest deployable unit (one or more tightly coupled containers); a Deployment manages a set of Pod replicas and handles rolling updates; a Service provides a stable network identity/load-balancing endpoint in front of a changing set of Pods. | [Pods, Deployments, and Services](../devops/kubernetes/01-pods-deployments-and-services.md) |
| 4 | What's the practical difference between a liveness probe and a readiness probe? | A failing liveness probe causes Kubernetes to restart the container; a failing readiness probe removes the Pod from Service traffic without restarting it — conflating the two can cause unnecessary restarts for a transient dependency outage that should only affect readiness. | [Probes, Resources, and Scaling](../devops/kubernetes/04-probes-resources-and-scaling.md) |
| 5 | Why is a Kubernetes Secret's base64 encoding not the same as encryption? | Base64 is trivially reversible by anyone who can read the Secret object — real protection requires RBAC restricting who can read it and etcd encryption at rest, not the encoding itself. | [ConfigMaps and Secrets](../devops/kubernetes/02-configmaps-and-secrets.md) |
| 6 | What problem does Helm solve that raw `kubectl apply` doesn't? | It packages, templates, and versions a set of related Kubernetes manifests as one installable unit with a release history — enabling parameterization across environments and reliable rollback that plain manifest files don't provide on their own. | [Charts and Chart Structure](../devops/helm/01-charts-and-chart-structure.md) |
| 7 | How does `helm rollback` work, and why does it matter for incident response? | Every `helm install`/`upgrade` creates a new release revision; `rollback` reverts to a specific prior revision's exact rendered manifests — giving a fast, reliable way to revert a bad deployment without manually reconstructing what was previously running. | [Releases and Release Management](../devops/helm/03-releases-and-release-management.md) |
| 8 | Why should CI run the full test suite on every pull request rather than only after merging to `main`? | Catching a regression before merge is strictly cheaper than after — once merged, other work may already depend on the broken change, and `main` should always remain deployable. | [Git Workflows and Branching](../devops/cicd/01-git-workflows-and-branching.md) |
| 9 | Why is tagging a container image `:latest` for production considered a mistake? | It's mutable and ambiguous — you can't tell exactly which build is running, and rollback (which requires redeploying a specific prior build) becomes unreliable without an immutable reference like a commit SHA or version tag. | [Docker Builds and Image Registries](../devops/cicd/03-docker-builds-and-image-registries.md) |
| 10 | What's the difference between a rolling deployment, blue-green deployment, and canary deployment? | Rolling gradually replaces old Pods with new ones (default Kubernetes behavior); blue-green runs both versions at full scale and switches traffic all at once (near-instant rollback); canary routes a small percentage of traffic to the new version first, increasing gradually while monitoring. | [Deployment and Rollback Strategies](../devops/cicd/04-deployment-and-rollback-strategies.md) |
| 11 | How would you troubleshoot a Pod stuck in `CrashLoopBackOff`? | `kubectl describe pod` for the Events section, then `kubectl logs --previous` to see why the last attempt crashed (the current instance may have already restarted again) — the root cause is almost always in the application itself or its immediate configuration. | [kubectl and Troubleshooting](../devops/kubernetes/05-kubectl-and-troubleshooting.md) |
| 12 | What does `helm upgrade --install --atomic` do, and why use it in a pipeline? | `--install` makes the command idempotent (install if the release doesn't exist, upgrade if it does); `--atomic` automatically rolls back to the previous revision if the upgrade fails — both matter specifically because a pipeline runs unattended, with no human watching to manually intervene on failure. | [Releases and Release Management](../devops/helm/03-releases-and-release-management.md) |

## Common red flags interviewers watch for

- Confusing liveness and readiness probes, or not knowing Kubernetes
  restarts the container on the former and just removes it from traffic
  on the latter.
- Treating a Kubernetes Secret's base64 encoding as sufficient
  protection on its own.
- Deploying with `:latest` and having no clear answer for "how would you
  roll back to exactly what was running yesterday."
- Not knowing why a multi-stage Docker build matters for a production
  image (assuming it's just about file organization, not image
  size/attack surface).

## Related deep-dive material

- [DevOps section overview](../devops/index.md) — Docker, Kubernetes,
  Helm, and CI/CD, 18 topic pages in total.
