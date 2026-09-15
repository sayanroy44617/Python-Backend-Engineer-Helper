# Ingress and External Access

## What

**Ingress** is a Kubernetes object that routes external HTTP(S) traffic
to internal Services based on hostname/path rules, typically also
handling TLS termination — a single entry point for many services,
rather than a separate cloud load balancer per Service.

## Why

Giving every Service its own `LoadBalancer` (see
[Pods, Deployments, and Services](01-pods-deployments-and-services.md))
means provisioning a separate cloud load balancer per service —
expensive and unwieldy once a cluster hosts more than a handful of
services. Ingress centralizes routing and TLS termination behind one (or
a few) load balancer, with the routing rules themselves managed as
Kubernetes configuration.

## How

### A basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

This routes all traffic for `api.example.com` to the `api` Service. The
Ingress object itself is just a routing *specification* — an
**Ingress controller** (a running component like NGINX Ingress
Controller or Traefik) actually implements it, watching Ingress objects
and configuring a real load balancer/proxy accordingly.

### Path-based and host-based routing

```yaml
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service: { name: users-service, port: { number: 80 } }
          - path: /orders
            pathType: Prefix
            backend:
              service: { name: orders-service, port: { number: 80 } }
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: admin-service, port: { number: 80 } }
```

One Ingress can route multiple hostnames and multiple paths to different
backend Services — a common pattern for exposing several microservices
(or a versioned/split API) through one shared external entry point.

### TLS termination

```yaml
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert
  rules:
    - host: api.example.com
      # ...
```

The Ingress controller terminates HTTPS at the edge using a certificate
stored in a [Secret](02-configmaps-and-secrets.md) — see
[Transport and Web Security](../../security/04-transport-and-web-security.md)
for why HTTPS matters at all; the Ingress is simply *where* that
termination commonly happens in a Kubernetes cluster, often paired with
`cert-manager` to automate certificate issuance/renewal (e.g. via Let's
Encrypt) rather than manually rotating certificates.

### Ingress vs. a Service's `LoadBalancer` type

```
LoadBalancer Service: one dedicated cloud load balancer per Service --
                      simple, but doesn't scale cost/operationally
                      beyond a small number of services
Ingress:              one (or few) load balancer(s) in front of the
                      Ingress controller, which then does L7 routing to
                      many Services based on host/path
```

Ingress operates at Layer 7 (HTTP), enabling host/path-based routing a
plain `LoadBalancer` Service (Layer 4) cannot do — this is the core
reason Ingress is the standard choice for HTTP-based services in any
cluster hosting more than a trivial number of them.

### Rate limiting and other cross-cutting concerns at the Ingress layer

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

Many Ingress controllers support annotations for cross-cutting HTTP
concerns (rate limiting, request size limits, redirect rules) applied at
the edge — conceptually similar to the
[rate limiting](../../rest-api/05-rate-limiting-and-api-security.md)
covered in REST API Engineering, just enforced at the cluster's entry
point rather than in application code.

### Ingress controller choice matters

```
NGINX Ingress Controller  -- most widely used, broad annotation support
Traefik                   -- built-in dynamic configuration, popular
                             with Docker/Kubernetes hybrid setups
Cloud-provider-specific   -- e.g. AWS ALB Ingress Controller, tightly
  controllers                integrated with that cloud's load balancer
```

The Ingress *object* is a Kubernetes-standard API, but its actual
behavior (which annotations are supported, performance characteristics)
depends entirely on which controller is installed in the cluster —
Ingress YAML written for one controller's annotations isn't necessarily
portable to another without adjustment.

## When to use

- Ingress for any cluster hosting more than a small handful of
  HTTP-based services needing external access.
- TLS termination at the Ingress layer (backed by `cert-manager` for
  automated certificate management) rather than handling TLS in each
  individual application.
- Host/path-based routing rules to consolidate multiple services behind
  one external entry point.

## When NOT to use

- Don't provision a separate `LoadBalancer` Service per application once
  you have more than a couple of HTTP services — it doesn't scale
  operationally or cost-wise the way a shared Ingress does.
- Don't assume Ingress annotations are portable across different Ingress
  controllers without checking — they're controller-specific extensions,
  not part of the core Ingress API.
- Don't use Ingress for non-HTTP traffic (raw TCP/UDP) — it's an L7,
  HTTP(S)-specific routing mechanism.

## Common mistakes

- Assuming the Ingress object alone does anything without an Ingress
  controller actually installed and running in the cluster.
- Writing Ingress annotations for one controller (e.g. NGINX) and
  expecting them to work unchanged after switching to a different
  controller.
- Manually managing TLS certificates instead of using `cert-manager` for
  automated issuance and renewal, risking an expired certificate outage.
- Provisioning a `LoadBalancer` Service per microservice instead of
  consolidating external access through a shared Ingress.

## Interview questions

1. What's the relationship between the Ingress object and an Ingress
   controller — why doesn't Ingress work without one installed?
2. Why is Ingress (Layer 7) able to do host/path-based routing that a
   plain `LoadBalancer` Service (Layer 4) cannot?
3. Where does TLS termination typically happen in a Kubernetes cluster,
   and what tool commonly automates certificate management there?
4. Why might Ingress annotations written for one controller not work
   after switching to a different one?
5. When would you still use a `LoadBalancer` Service directly instead of
   routing through an Ingress?

## Senior-level considerations

- Choosing an Ingress controller is itself a real architectural decision
  — annotation support, performance characteristics, and cloud
  integration differ meaningfully between NGINX, Traefik, and
  cloud-native controllers, and switching later has real migration cost.
- Centralizing TLS termination, rate limiting, and routing at the Ingress
  layer is both an operational simplification and a security control
  point — it's often where organization-wide policies (WAF rules, global
  rate limits) are enforced consistently across many services.
- Ingress is the typical seam where a cluster's internal service mesh (if
  any) meets the outside world — understanding this boundary clearly is
  important for reasoning about where security controls, observability,
  and traffic management responsibilities actually live in a given
  architecture.
