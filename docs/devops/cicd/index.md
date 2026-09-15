# CI/CD

How code goes from a developer's commit to a running, versioned
deployment: source control workflows, automated pipeline stages
(build/test/package), building and publishing container images, and
deploying/rolling back changes safely.

This section assumes familiarity with
[Testing](../../testing/index.md),
[Docker](../docker/index.md), and
[Kubernetes](../kubernetes/index.md)/[Helm](../helm/index.md) — CI/CD
is the automation layer that ties those together into a repeatable
pipeline, rather than a replacement for any of them.

## Topics

1. [Git Workflows and Branching](01-git-workflows-and-branching.md)
2. [Pipeline Stages: Build, Test, and Package](02-pipeline-stages-build-test-and-package.md)

3. [Docker Builds and Image Registries](03-docker-builds-and-image-registries.md)
4. [Deployment and Rollback Strategies](04-deployment-and-rollback-strategies.md)
