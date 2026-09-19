# Deployment and Rollback Strategies

## What

The final CI/CD stage: taking a built, tested, tagged image (see
[Docker Builds and Image Registries](03-docker-builds-and-image-registries.md))
and deploying it to a running environment — plus having a fast, reliable
way to reverse that deployment (**rollback**) if it turns out to be
broken.

## Why

Automating the build and test stages but still deploying manually
reintroduces exactly the inconsistency and human error CI/CD is meant to
remove. Automated, repeatable deployment (and an equally automated,
fast rollback path) is what turns "we have a pipeline" into "we can
ship changes to production confidently and frequently."

## How

### Continuous delivery vs. continuous deployment

```
Continuous delivery:  every passing build is deployable, but production
                       deployment requires a manual approval step
Continuous deployment: every passing build on main is deployed to
                       production automatically, no manual gate
```

The distinction matters for how much a team trusts its automated
checks — continuous deployment requires enough confidence in the test
suite, monitoring, and rollback speed that a manual gate before
production isn't adding meaningful safety, only latency.

### Deploying via Helm in a pipeline

```yaml
# .github/workflows/deploy.yml (conceptual)
- run: |
    helm upgrade --install myapp mychart/ \
      --set image.tag=${{ github.sha }} \
      --namespace production \
      --atomic --timeout 5m
```

This is the same
[`helm upgrade --install --atomic`](../helm/03-releases-and-release-management.md)
pattern covered in the Helm section, just invoked from a pipeline step
instead of a developer's terminal — `--atomic` matters even more here,
since an automated deployment has no human watching in real time to
notice and manually roll back a failed rollout.

### Rolling deployment (default Kubernetes behavior)

```
Old Pods gradually replaced by new Pods, respecting maxUnavailable/
maxSurge, traffic only shifts to new Pods once they pass readinessProbe.
```

This is the default
[Deployment rolling-update mechanism](../kubernetes/01-pods-deployments-and-services.md)
— no additional infrastructure required, and it's what most services
use by default unless they need finer-grained control over the
transition.

### Blue-green deployment

```
Blue (current, live)     ── receiving 100% of traffic
Green (new version)      ── fully deployed, receiving 0% of traffic
                             -> switch traffic all at once, once verified
```

Both versions run simultaneously at full scale; a router/load balancer
switch (not a gradual Pod-by-pod rollout) cuts traffic over instantly.
Rollback is just switching traffic back to blue — near-instant, since
the old version's Pods never stopped running. The cost is running double
the infrastructure during the transition.

### Canary deployment

```
100% traffic ─┬─▶ old version (95%)
              └─▶ new version (5%)  -- gradually increased if healthy
```

Only a small percentage of traffic reaches the new version initially,
increased gradually while monitoring error rates/latency — this limits
the blast radius of a bad deployment to a small fraction of real traffic
rather than either 100% (rolling update, eventually) or a full,
immediate cutover (blue-green). Canary requires more sophisticated
traffic-splitting infrastructure (a service mesh or an Ingress
controller supporting weighted routing) than a plain rolling update.

### Feature flags decoupling deploy from release

```python
if feature_flags.is_enabled("new_checkout_flow", user=current_user):
    return new_checkout_handler(request)
return legacy_checkout_handler(request)
```

Deploying new code paths behind a flag — turned on for real users
independently of the deployment itself — decouples "code is running in
production" from "users are experiencing the new behavior," letting a
team deploy frequently while controlling exposure/rollout of risky
changes separately and instantly (flipping a flag) rather than through a
full redeploy.

### Automated rollback triggers

```yaml
- run: helm upgrade --install myapp mychart/ --atomic --timeout 5m
# --atomic already rolls back automatically on failed rollout
```

```
Additional automated rollback signal: a post-deploy monitoring check
(error rate / latency spike) triggering an automatic `helm rollback`
even after a rollout technically "succeeded" but the new version is
behaving badly under real traffic.
```

`--atomic` catches rollouts that fail to *become healthy* (failing
readiness checks); it doesn't catch a rollout that succeeds but then
causes elevated error rates under real production load — some teams add
a post-deploy monitoring window (tied to
[Health Checks and Alerting](../../observability/04-health-checks-and-alerting.md))
that can trigger an automatic rollback based on real traffic behavior,
not just readiness.

### Manual rollback as the fallback

```bash
helm rollback myapp <previous-revision> --namespace production
```

Even with automated safeguards, a fast, well-rehearsed manual rollback
path (see
[Releases and Release Management](../helm/03-releases-and-release-management.md))
is essential — automated triggers won't catch every failure mode, and
during an incident, the ability to immediately revert without
reasoning through a complex automated system is valuable in itself.

## When to use

- Rolling deployments as the default for most services — no extra
  infrastructure, good enough safety when combined with `--atomic` and
  readiness probes.
- Blue-green or canary specifically for high-traffic, high-risk services
  where limiting blast radius or enabling instant rollback justifies the
  added infrastructure/complexity.
- Feature flags for risky behavioral changes that benefit from being
  decoupled from the deployment itself.

## When NOT to use

- Don't reach for blue-green/canary complexity for low-traffic or
  low-risk internal services where a plain rolling update's safety is
  already sufficient.
- Don't rely solely on automated rollback triggers without a rehearsed
  manual rollback path — automation won't catch every failure mode.
- Don't treat continuous deployment (fully automatic production
  deploys) as a default without first having the test coverage,
  monitoring, and rollback speed to justify removing the manual gate.

## Common mistakes

- Deploying without `--atomic` (or an equivalent safeguard), leaving a
  failed rollout partially applied and requiring manual cleanup.
- Adopting canary/blue-green deployment infrastructure before a service
  actually has the traffic volume or risk profile to justify the added
  complexity.
- No rehearsed manual rollback procedure, causing a slow, error-prone
  response during an actual incident.
- Conflating "deployed" with "released" — shipping risky behavioral
  changes directly rather than behind a feature flag, losing the ability
  to instantly disable them without a redeploy.

## Interview questions

- What's the difference between continuous delivery and continuous
    deployment, and what does that distinction depend on?

    **Answer:** Continuous delivery means every change is automatically
    built, tested, and made ready to deploy, but a human still clicks
    "deploy." Continuous deployment goes one step further and deploys to
    production automatically once tests pass — the distinction is really
    whether there's a manual approval gate before production.

- Compare rolling, blue-green, and canary deployment strategies — what
    problem does each solve, and at what cost?

    **Answer:** Rolling replaces instances gradually with no extra
    infrastructure but a slower, harder rollback. Blue-green runs two full
    environments and switches traffic instantly, giving fast rollback at
    double the infrastructure cost. Canary sends a small percentage of
    traffic to the new version first, limiting blast radius but requiring
    good metrics/monitoring to decide when to proceed.

- Why might `--atomic` not catch every kind of bad deployment, and what
    additional safeguard addresses that gap?

    **Answer:** `--atomic` only catches failures Kubernetes/Helm can detect
    (pods crashing, timeouts) — it won't catch a deployment that "succeeds"
    but has a subtle logic bug or elevated error rate. Health checks plus
    real application-level monitoring/alerting catch that gap.

- How do feature flags decouple deployment from release, and why is
    that useful?

    **Answer:** Deploying ships the code to production, but a feature flag
    controls whether it's actually active for users — so you can deploy
    risky code dark, then flip it on gradually or instantly roll it back
    without a new deployment at all.

    ```python
    if feature_flags.is_enabled("new_checkout"):
       return new_checkout_flow()
    ```

- Why is a rehearsed manual rollback procedure still necessary even with
    automated rollback triggers in place?

    **Answer:** Automated triggers only cover failure modes someone thought
    to detect in advance — a novel failure (e.g. a slow data-corruption bug)
    might not trip any automated trigger at all, so the team needs to know
    how to roll back by hand under pressure.

## Senior-level considerations

- Choosing a deployment strategy (rolling vs. blue-green vs. canary) is
  a real cost/safety trade-off, not a one-size-fits-all decision — the
  right choice depends on a service's traffic volume, blast-radius
  tolerance, and available infrastructure, and should be revisited as a
  service's risk profile changes. For example, a low-traffic internal
  tool may not justify blue-green's double infrastructure cost, while a
  payments API almost certainly does.
- Feature flags are a powerful tool for decoupling deploy from release,
  but they add their own complexity (flag proliferation, testing
  combinatorics of flag states) that needs active management, not just
  adoption. For example, a codebase with 40 long-lived flags can end up
  effectively testing exponentially many code paths nobody fully covers.
- The maturity of a team's CI/CD practice is often best measured by how
  fast and how confidently it can roll back a bad production deployment
  — a fast, well-tested rollback path is arguably more valuable day to
  day than any specific deployment strategy's sophistication. For
  example, a team that can roll back in under 2 minutes with one command
  is in a much stronger position during an incident than one debating
  canary percentages while the site is down.
