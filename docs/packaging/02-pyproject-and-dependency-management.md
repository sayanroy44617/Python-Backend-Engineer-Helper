# pyproject.toml and Dependency Management

## What

`pyproject.toml` is the standardized (PEP 518/621) configuration file for
Python projects: project metadata, dependencies, build system, and tool
configuration in one place. This topic covers its structure, how
dependencies are declared and resolved, and how semantic versioning governs
what "compatible" means when specifying version ranges.

## Why

Before `pyproject.toml` was standardized, projects scattered configuration
across `setup.py`, `setup.cfg`, `requirements.txt`, and tool-specific files.
A single, declarative `pyproject.toml` makes a project's dependencies,
build process, and tool configuration inspectable and reproducible —
essential for CI, packaging, and onboarding.

## How

### Anatomy of `pyproject.toml`

```toml
[project]
name = "python-backend-engineer-helper"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
authors = [
    { name = "Sayan Roy", email = "sayan.roy@thoughtworks.com" }
]
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.115,<0.116",
    "sqlalchemy>=2.0,<3.0",
]

[project.scripts]
python-backend-engineer-helper = "python_backend_engineer_helper:main"

[build-system]
requires = ["uv_build>=0.12.10,<0.13.0"]
build-backend = "uv_build"

[dependency-groups]
dev = [
    "pytest>=8.0",
    "mkdocs-material>=9.7.7",
]
```

- `[project]` — PEP 621 standard metadata: name, version, dependencies,
  Python version constraint.
- `[project.scripts]` — declares CLI entry points (`python -m` style
  commands installed alongside the package).
- `[build-system]` — declares which backend builds the package (`uv_build`,
  `setuptools`, `hatchling`, `poetry-core`, etc.) — this is what lets `pip`/
  `uv`/`build` know how to turn source into an installable distribution.
- `[dependency-groups]` (PEP 735) — dependencies only needed for
  development/testing (`dev`), not installed for end users of the package.

### Dependency version specifiers

```toml
dependencies = [
    "fastapi==0.115.0",     # exact pin
    "fastapi>=0.115.0",     # minimum version
    "fastapi>=0.115,<0.116",# range
    "fastapi~=0.115.0",     # compatible release: >=0.115.0, <0.116.0
]
```

`~=` ("compatible release") is the most common choice for application
dependencies — it allows patch-level updates but blocks minor/major
changes that might break compatibility.

### Semantic versioning (SemVer)

Given `MAJOR.MINOR.PATCH` (e.g. `2.1.4`):

| Segment | Bumped when | Compatibility |
|---|---|---|
| `MAJOR` | Breaking changes | Not backward compatible |
| `MINOR` | New backward-compatible features | Backward compatible |
| `PATCH` | Backward-compatible bug fixes | Backward compatible |

SemVer is a *convention*, not enforced by the language or `pip` — a
library can technically break compatibility in a "patch" release. This is
why pinning ranges (`~=`, `<next-major>`) is a risk-mitigation strategy,
not a guarantee.

### Lockfiles vs version ranges

`pyproject.toml` declares *acceptable* version ranges; a lockfile
(`uv.lock`, or `poetry.lock`) records the **exact** versions actually
resolved and installed, including transitive dependencies.

```bash
uv lock        # resolve dependencies, write uv.lock
uv sync        # install exactly what's in uv.lock
```

This two-file model (loose ranges in `pyproject.toml`, exact pins in the
lockfile) lets you intentionally widen a range (to allow upgrades) while
still reproducing a known-good exact set of versions until you choose to
re-lock.

### Dependency groups vs optional extras

```toml
[project.optional-dependencies]
postgres = ["psycopg2-binary>=2.9"]
redis = ["redis>=5.0"]
```

```bash
pip install "my-package[postgres]"
```

Optional extras (`[project.optional-dependencies]`) are for *end users*
who may need optional functionality (e.g. a specific DB driver);
`[dependency-groups]` (dev, test) are for *contributors* working on the
project itself and are never installed for end users.

## When to use

- Declare all runtime dependencies with a bounded range (`~=` or explicit
  `<upper>`) rather than unbounded (`fastapi` with no version at all).
- Use `[dependency-groups]`/`extras` to separate what's needed to *run*
  the project from what's needed to *develop* or *test* it.
- Commit the lockfile (`uv.lock`) to version control so CI and every
  contributor resolve the exact same dependency versions.

## When NOT to use

- Don't pin every dependency to an exact version in `pyproject.toml`
  itself (`fastapi==0.115.0`) for a library meant to be consumed by
  others — overly strict pins cause "dependency hell" for downstream
  consumers; use ranges in `pyproject.toml` and let the lockfile pin exact
  versions for your own reproducible dev/CI environment.
- Don't leave dependencies fully unbounded (no version at all) in
  anything beyond a quick prototype — an unannounced breaking change
  upstream can silently break your build.
- Don't manually edit a lockfile — regenerate it with the tool that owns
  it (`uv lock`) to keep its internal hashes/resolution consistent.

## Common mistakes

- Confusing "a library's `pyproject.toml` dependency range" with "the
  exact versions actually installed" — only the lockfile (or `pip freeze`)
  tells you what's really running.
- Assuming SemVer is enforced by tooling — it's a convention some
  maintainers don't follow strictly; a "minor" release can still contain
  breaking changes in practice.
- Forgetting to re-run `uv lock`/commit the updated lockfile after adding
  or changing a dependency, causing CI and local environments to drift
  apart.
- Mixing `[project.dependencies]` (runtime) and `[dependency-groups].dev`
  (development-only) incorrectly — shipping test/dev tools as runtime
  dependencies bloats production installs.

## Interview questions

1. What's the difference between a version range in `pyproject.toml` and
   what's recorded in a lockfile?

   **Answer:** `pyproject.toml` declares what's *acceptable* (e.g.
   `fastapi~=0.115.0`); the lockfile records the *exact* version (and all
   transitive versions) actually resolved and installed. The range can
   stay loose while the lockfile pins a precise, reproducible snapshot.

2. Explain semantic versioning. Is it enforced by tooling, or a
   convention?

   **Answer:** `MAJOR.MINOR.PATCH` — major = breaking change, minor = new
   backward-compatible feature, patch = backward-compatible bug fix. It's
   purely a convention; nothing stops a maintainer from shipping a
   breaking change in a "patch" release, which is why lockfiles/CI tests
   matter more than trusting the version number alone.

3. What's the difference between `[project.optional-dependencies]` and
   `[dependency-groups]`?

   **Answer:** Optional extras are for *end users* who opt into extra
   functionality at install time (`pip install "pkg[postgres]"`).
   Dependency groups (`dev`, `test`) are for *contributors* working on the
   project itself and are never installed for someone just using the
   package.

4. Why would a library maintainer use loose version ranges while an
   application team uses a lockfile with exact pins?

   **Answer:** A library has to stay compatible with whatever versions
   its many downstream consumers already have installed, so tight pins
   would cause conflicts everywhere. An application has one deployment
   target, so pinning exact versions via a lockfile removes ambiguity
   about what's actually running in production.

5. What does `[build-system]` in `pyproject.toml` actually control?

   **Answer:** It tells tools like `pip`/`uv`/`build` which backend
   (`setuptools`, `hatchling`, `uv_build`, etc.) knows how to turn your
   source tree into an installable wheel/sdist — without it, tools
   wouldn't know how to build the package at all.

## Senior-level considerations

- Dependency management strategy differs for **libraries** (loose ranges,
  broad compatibility, no lockfile shipped) vs **applications** (exact
  lockfile, reproducible deploys) — conflating the two leads to either
  overly rigid libraries or fragile, unreproducible applications. For
  example, a shared internal auth library pinning `httpx==0.27.0` exactly
  can break every service that depends on a newer `httpx` for something
  else.
- Regularly updating dependencies (and re-locking) is a deliberate
  practice, not a one-time setup step — stale dependencies accumulate
  security vulnerabilities; tools like Dependabot/Renovate automate this
  in CI. For example, a bot opens a weekly PR bumping `uv.lock`, and CI
  running the test suite against it catches breakage before a human even
  looks.
- `pyproject.toml`'s standardization (PEP 621) was a deliberate move away
  from fragmented tool-specific configs (`setup.py`, `setup.cfg`) —
  understanding this history explains why older codebases you may
  encounter still use `setup.py`, and how to reason about migrating them.
  For example, a legacy repo with `setup.py` + `setup.cfg` +
  `requirements.txt` can usually be consolidated into one
  `pyproject.toml` without changing what actually gets installed.
