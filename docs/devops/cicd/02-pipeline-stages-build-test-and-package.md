# Pipeline Stages: Build, Test, and Package

## What

A CI/CD **pipeline** is an automated sequence of stages — commonly
build, test, and package — run on every relevant Git event (a PR, a
merge to `main`, a tag), turning source code into a validated,
distributable artifact without manual intervention.

## Why

Manually running tests and building artifacts before every deployment is
slow, inconsistent, and easy to skip under time pressure. A pipeline
makes these steps automatic and mandatory — the same checks run
identically every time, for every change, removing "did someone remember
to run the tests" as a source of production incidents.

## How

### A typical pipeline shape

```yaml
# .github/workflows/ci.yml (conceptual, GitHub Actions)
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install uv && uv sync
      - run: uv run ruff check .
      - run: uv run mypy .
      - run: uv run pytest --cov
```

Each step fails fast — a linting failure stops the pipeline before
running the (slower) test suite, giving the fastest possible feedback
for the most common class of failure.

### Build stage

```bash
uv sync                 # install dependencies from a locked file
```

The build stage establishes a clean, reproducible environment before
anything else runs — see
[Environments and Installers](../../packaging/01-environments-and-installers.md)
for why a locked dependency file (not just `requirements.txt` with
loose versions) matters here specifically: a pipeline that resolves
different dependency versions on different runs produces
non-reproducible, hard-to-debug CI failures.

### Test stage

```bash
uv run pytest --cov=src --cov-report=xml --cov-fail-under=80
```

The test stage runs the same [pytest](../../testing/01-pytest-fundamentals.md)
suite a developer would run locally — see
[Test Strategy and Isolation](../../testing/03-test-strategy-and-isolation.md)
for how a suite should be structured (fast unit tests, isolated
integration tests) to keep this stage fast enough to run on every PR
without becoming a bottleneck. `--cov-fail-under` turns a coverage
regression into a hard pipeline failure rather than a number nobody
looks at.

### Parallelizing pipeline stages

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [ ... ]
  unit-tests:
    runs-on: ubuntu-latest
    steps: [ ... ]
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
    steps: [ ... ]
```

Independent stages (lint, unit tests, integration tests needing a real
database service container) can run in parallel jobs rather than
sequentially — reducing total pipeline wall-clock time meaningfully once
a test suite is large enough that sequential execution becomes the
slowest part of the developer feedback loop.

### Package stage

```bash
uv build                 # produces a wheel/sdist for a library
# or, for a deployable service:
docker build -t myapp:${{ github.sha }} .
```

Packaging produces the actual distributable artifact — a wheel for a
library, or (far more commonly for a backend service) a container image,
covered in depth in
[Docker Builds and Image Registries](03-docker-builds-and-image-registries.md).
The artifact produced here, tagged with an immutable reference (a commit
SHA or version tag), is exactly what later flows into
[deployment](04-deployment-and-rollback-strategies.md).

### Caching dependencies between runs

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"
```

Caching dependency installation between pipeline runs (keyed on the lock
file's hash) avoids re-downloading/re-resolving the same dependencies on
every single run — a meaningful speedup for pipelines run dozens of
times a day across a team, at the cost of occasionally needing to bust a
stale cache.

### Required checks gating merge

```
Branch protection on main requires:
- lint: passing
- unit-tests: passing
- integration-tests: passing
```

Tying pipeline stage results to branch protection (see
[Git Workflows and Branching](01-git-workflows-and-branching.md)) is what
turns "we have CI" into "CI actually prevents bad code from merging" —
a pipeline that runs but doesn't block merging on failure provides much
weaker guarantees.

## When to use

- Fail-fast stage ordering (lint → unit tests → integration tests →
  package) to give the quickest feedback for the most common failures.
- Parallel jobs once a test suite is large enough that sequential
  execution meaningfully slows down PR feedback.
- Coverage/quality gates (`--cov-fail-under`, required lint/type checks)
  enforced as pipeline failures, not just informational output.

## When NOT to use

- Don't run the full, slow integration/E2E suite on every single commit
  push within a PR if a lighter subset can give fast feedback, reserving
  the full suite for pre-merge or scheduled runs if it's genuinely too
  slow otherwise.
- Don't skip caching dependencies between runs once install time becomes
  a meaningful fraction of total pipeline time.
- Don't treat a pipeline stage as a real quality gate if it isn't wired
  into required branch protection checks — an ignorable failing check
  provides false confidence.

## Common mistakes

- Sequential-only pipelines where independent stages (lint, unit tests,
  integration tests) that could run in parallel are needlessly serialized,
  slowing down feedback.
- Not caching dependency installation, adding unnecessary minutes to
  every single pipeline run.
- Coverage or lint checks that run but don't fail the build on
  regression, becoming ignorable noise rather than an enforced gate.
- Pipeline stages not tied to required branch-protection checks, so a
  failing pipeline doesn't actually block a bad merge.

## Interview questions

1. Why should linting/fast checks run before the full test suite in a
   pipeline, rather than in parallel or after?

   **Answer:** Lint/type checks usually take seconds and catch trivial
   mistakes — failing fast on those saves the time and compute cost of
   running a slow full test suite on code that was already broken.

2. How would you speed up a CI pipeline whose test suite has grown too
   slow for fast PR feedback?

   **Answer:** Parallelize/shard tests across workers, cache dependency
   installs, and separate a fast "smoke" subset that runs on every push from
   a slower full suite that runs less often (e.g. pre-merge or nightly).

3. What's the difference between the build, test, and package stages,
   and what does each stage's output feed into?

   **Answer:** Build compiles/prepares the code, test verifies it behaves
   correctly, and package bundles the verified artifact (e.g. a Docker
   image or wheel) for deployment. Each stage should only run if the
   previous one succeeded, since a later stage's output is only trustworthy
   if earlier gates passed.

4. Why does caching dependency installation matter for pipeline
   performance, and what's it typically keyed on?

   **Answer:** Reinstalling every dependency from scratch on every run wastes
   minutes per pipeline run across hundreds of runs a week. It's usually
   keyed on a hash of the lockfile (e.g. `uv.lock`/`poetry.lock`), so the
   cache is reused unless dependencies actually changed.

   ```yaml
   key: deps-${{ hashFiles('uv.lock') }}
   ```

5. What makes a pipeline stage function as an actual quality gate rather
   than just informational output?

   **Answer:** It has to be a *required, blocking* check — if a stage can
   fail and the pipeline still proceeds (or a human can merge anyway), it's
   just a report, not a gate.

## Senior-level considerations

- Pipeline stage design is a direct trade-off between feedback speed and
  thoroughness — the right balance (what runs on every push vs. only
  pre-merge vs. only nightly) depends on team size, test suite size, and
  how expensive a slow feedback loop actually is for that team. For
  example, a team with a 40-minute suite might run only unit tests per
  push and reserve integration tests for pre-merge.
- A pipeline is only as strong as its enforcement — coverage gates, lint
  checks, and required status checks that can be silently bypassed
  (admin merge overrides, non-blocking checks) provide much weaker
  guarantees than they appear to on paper. For example, a "required"
  check marked non-blocking in the branch protection settings gives a
  false sense of safety.
- As pipelines grow, treating pipeline configuration itself as code
  (versioned, reviewed, tested where practical) becomes important — a
  broken or misconfigured pipeline is itself an incident, not just a
  minor inconvenience, once deployment depends on it. For example, an
  unreviewed change to a shared reusable workflow file can silently break
  deployments for every team using it.
