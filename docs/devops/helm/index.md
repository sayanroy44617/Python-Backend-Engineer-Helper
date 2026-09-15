# Helm

Packaging, templating, and releasing Kubernetes manifests: how a
non-trivial application's YAML (Deployments, Services, ConfigMaps,
Secrets, and more) is parameterized, versioned, and installed/upgraded
repeatably instead of managed as one-off `kubectl apply` commands.

This section assumes familiarity with
[Kubernetes](../kubernetes/index.md) — Helm doesn't replace the
underlying objects (Pods, Deployments, Services, ConfigMaps, Secrets)
covered there; it's a packaging and templating layer on top of them.

## Topics

1. [Charts and Chart Structure](01-charts-and-chart-structure.md)
2. [Templates, values.yaml, and Helpers](02-templates-values-and-helpers.md)
3. [Releases and Release Management](03-releases-and-release-management.md)
4. [Secrets and Configuration Management](04-secrets-and-configuration-management.md)
