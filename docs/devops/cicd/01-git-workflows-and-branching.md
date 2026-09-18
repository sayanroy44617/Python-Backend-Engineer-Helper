# Git Workflows and Branching

## What

A **branching strategy** is a team's convention for how code moves from
in-progress work to a mergeable, releasable state in Git — which
branches exist, what triggers a merge, and how that maps to what CI/CD
actually builds, tests, and deploys.

## Why

CI/CD pipelines are triggered by Git events (a push, a pull request, a
tag) — the branching strategy a team uses directly determines what a
pipeline needs to do at each stage: run tests on every PR, build and push
an image on merge to `main`, deploy to production only on a tagged
release. Without a clear convention, it's ambiguous what a given
pipeline run should actually do, and deployments become inconsistent and
error-prone.

## How

### Trunk-based development

```
main ─●──●──●──●──●──●──●──●──●──●──▶
       \    /    \    /    \    /
        PR        PR        PR
       (short-lived feature branches, merged within a day or two)
```

Every change is a short-lived branch off `main`, merged back quickly
after passing CI — `main` is always close to production-ready. This is
the standard modern approach for teams practicing continuous deployment,
since long-lived branches accumulate drift and make merges (and CI
feedback) progressively harder the longer they live.

### GitFlow (longer-lived branches)

```
main     ──────●────────────●──────▶  (production releases only)
develop  ──●──●──●──●──●──●──●──●──▶  (integration branch)
            \        /
          feature/x  (longer-lived feature branch)
```

`develop` accumulates work between releases; `main` only receives
merges at release time. This adds process overhead (an extra
integration branch, release branches) that trunk-based development
avoids — appropriate for teams needing more release-cadence control
(e.g. scheduled releases with a longer QA cycle), but usually
unnecessary overhead for a team that can deploy continuously with good
test coverage.

### Pull requests as the CI trigger point

```yaml
# .github/workflows/ci.yml (conceptual)
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
```

Running the full test suite on every pull request (before merge, not
after) is what makes `main` trustworthy — catching a regression after
it's already merged is strictly more expensive than catching it in the
PR, since by then other work may already depend on the broken change.

### Commit and PR conventions that pipelines rely on

```
feat: add pagination to /users endpoint
fix: correct off-by-one in cursor pagination
chore: bump SQLAlchemy to 2.0.30
```

Conventional, structured commit messages (or PR titles) aren't just
style — pipelines commonly parse them to automate changelog generation
and semantic version bumps (`feat` → minor, `fix` → patch,
`BREAKING CHANGE` → major), removing manual versioning decisions from
the release process entirely.

### Tags as release/deployment triggers

```bash
git tag v1.4.0
git push origin v1.4.0
```

```yaml
on:
  push:
    tags:
      - 'v*'
```

Tagging a specific commit as a release version — rather than deploying
whatever happens to be on `main` at a given moment — gives an
unambiguous, reproducible reference for "exactly what code is running in
production," which matters directly for
[rollback](04-deployment-and-rollback-strategies.md): rolling back means
redeploying a specific prior tag, not guessing which commit was
previously live.

### Protecting `main`

```
Branch protection on main:
- require passing CI checks before merge
- require at least one approving review
- disallow force-push
```

Branch protection rules are what actually enforce that "every change
was tested and reviewed before reaching `main`" is true, rather than
being just a convention people can bypass under time pressure — without
enforcement, the guarantee a pipeline's later stages depend on (that
`main` is always in a deployable state) silently erodes.

## When to use

- Trunk-based development with short-lived feature branches and branch
  protection for teams practicing continuous deployment.
- GitFlow (or a similar longer-lived-branch model) specifically when a
  scheduled/controlled release cadence is a real business requirement,
  not by default.
- Git tags as the deployment trigger for production releases, keeping
  "what's deployed" traceable to an exact, immutable reference.

## When NOT to use

- Don't adopt GitFlow's extra branches/process by default — it adds
  overhead that most teams practicing frequent, incremental delivery
  don't need.
- Don't let feature branches live for weeks — the longer a branch
  diverges from `main`, the more expensive the eventual merge and the
  less useful CI feedback becomes during that time.
- Don't deploy directly from whatever commit happens to be on `main` at
  deploy time without a tag/release marker — it makes rollback and
  auditing which code is actually running much harder.

## Common mistakes

- Long-lived feature branches that drift far from `main`, causing large,
  risky merges and delayed CI feedback.
- No branch protection on `main`, allowing untested or unreviewed changes
  to merge directly, undermining every later pipeline stage's
  assumptions.
- Deploying an arbitrary `main` commit rather than a tagged release,
  making "what's actually running in production" ambiguous after the
  fact.
- Inconsistent or absent commit message conventions, blocking automated
  changelog/version generation the rest of the pipeline may depend on.

## Interview questions

1. What's the practical trade-off between trunk-based development and
   GitFlow?

   **Answer:** Trunk-based dev keeps everyone merging small changes into
   `main` frequently, which gives fast feedback but requires strong test/CI
   discipline. GitFlow uses long-lived branches (`develop`, release
   branches) that give more structure for scheduled releases, at the cost of
   slower merges and more complex conflict resolution.

2. Why should CI run on pull requests rather than only after merging to
   `main`?

   **Answer:** Catching a broken test on a PR is cheap — you fix it before
   it ever touches `main`. Catching it after merge means `main` is broken
   for everyone until someone reverts or fixes it, which blocks other
   people's work.

3. Why use Git tags (rather than arbitrary commits) as the trigger for a
   production deployment?

   **Answer:** A tag is an explicit, immutable, human-chosen "this is release
   1.4.0" marker, whereas any commit on `main` could be deployed accidentally
   by an automated trigger. Tags make "what's in production" traceable and
   intentional.

   ```bash
   git tag v1.4.0 && git push origin v1.4.0
   ```

4. What do branch protection rules actually guarantee, and why does the
   rest of a CI/CD pipeline depend on that guarantee holding?

   **Answer:** They guarantee that code can't reach `main` (or get deployed)
   without passing required checks and reviews — no direct pushes, no merging
   with a red CI run. The whole "main is always deployable" assumption that
   later pipeline stages rely on falls apart if this isn't actually enforced.

5. How can commit message conventions feed into automated versioning or
   changelog generation?

   **Answer:** Conventions like Conventional Commits (`feat:`, `fix:`,
   `BREAKING CHANGE:`) let tooling automatically decide the next semantic
   version and generate a changelog, instead of a human manually figuring out
   if a release is a major/minor/patch bump.

## Senior-level considerations

- Branching strategy is a team-process decision with real engineering
  consequences — the wrong choice for a team's actual release cadence
  and size creates friction (either too much process overhead or too
  little safety) that compounds over time. For example, a 3-person
  startup adopting full GitFlow with release branches often just adds
  merge overhead with no matching benefit.
- The reliability of "main is always deployable" is only as strong as
  the branch protection and CI enforcement behind it — treating it as a
  cultural norm alone, without enforcement, tends to erode under
  deadline pressure. For example, allowing admin override on required
  checks "just this once" during an incident often becomes a recurring
  habit.
- Tag-based, immutable release references are foundational to reliable
  rollback and auditing — without them, incident response has to first
  reconstruct what was actually deployed before it can even begin fixing
  it. For example, if deployments reference a moving branch instead of a
  tag, redeploying "the same version" during an incident might silently
  pick up new, untested commits.
