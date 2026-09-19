# Package Structure and Editable Installs

## What

How a Python project's files are laid out so it can be installed as a
package (`src` layout vs flat layout), and **editable installs** — a way to
install a local project so code changes take effect immediately without
reinstalling.

This is about *distribution/installation* structure. For how imports
resolve at runtime once installed, see
[Modules and Packages](../python/04-modules-and-packages.md).

## Why

A project's on-disk layout determines whether it can be reliably packaged,
installed, and imported the same way in development, CI, and production.
Editable installs are what make local development workflows fast — you
edit source and immediately re-run without a rebuild/reinstall step.

## How

### `src` layout (recommended)

```
my-project/
├── pyproject.toml
├── README.md
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── api/
│       └── services/
└── tests/
    └── test_services.py
```

### Flat layout

```
my-project/
├── pyproject.toml
├── README.md
├── my_package/
│   ├── __init__.py
│   ├── api/
│   └── services/
└── tests/
    └── test_services.py
```

The `src` layout puts the importable package inside a `src/` directory,
separate from the project root. This forces the package to be **installed**
(even in editable mode) before it can be imported — you can't accidentally
import it via the current working directory, which is exactly what catches
missing dependencies or packaging misconfigurations early, before they
reach CI/production. The flat layout is simpler for small scripts but
makes it easier to accidentally rely on `cwd`-relative imports that break
once actually installed elsewhere.

### Declaring the package in `pyproject.toml`

```toml
[project]
name = "my-package"

[tool.setuptools.packages.find]
where = ["src"]
```

Different build backends (`setuptools`, `hatchling`, `uv_build`) have
their own equivalent configuration for where to find the package(s) to
include — the principle is the same: tell the build backend where the
importable code lives.

### Editable installs

```bash
pip install -e .
# or, with uv:
uv pip install -e .
```

An editable install registers the package so Python imports it directly
from your working source tree (via a path reference, historically an
`.egg-link`/now typically `.pth` files or PEP 660 metadata) instead of
copying files into `site-packages`. Editing `src/my_package/api.py` and
re-running immediately reflects the change — no reinstall needed.

```bash
# Without -e: a snapshot copy is installed; source edits have no effect
# until you reinstall
pip install .
```

### Building a distributable package

```bash
python -m build          # produces dist/*.whl and dist/*.tar.gz
uv build                  # uv's equivalent
```

A **wheel** (`.whl`) is a prebuilt binary distribution format (fast to
install, no build step needed at install time). An **sdist**
(`.tar.gz`) is a source distribution, built from source at install time if
no matching wheel is available. Modern tooling produces both by default.

## When to use

- Use the `src` layout for anything beyond a throwaway script — it
  enforces that imports only work through a proper install, catching
  packaging bugs before they reach production.
- Use editable installs (`pip install -e .` / `uv sync` with a local
  package) during development so changes are picked up immediately.
- Declare a proper `[build-system]` and package discovery configuration
  as soon as a project is meant to be installed elsewhere (another
  service, a Docker image, a shared internal package registry) — not just
  run from the checkout directory.

## When NOT to use

- Don't rely on flat-layout, cwd-relative imports for anything that will
  eventually be packaged/deployed — it works during ad hoc local
  development but can mask import bugs that only surface once actually
  installed as a package.
- Don't ship editable installs to production — they're a development
  convenience; production/CI builds should install a real wheel/sdist (or
  the exact pinned lockfile versions) for reproducibility.
- Don't manually copy source files into a Docker image without an install
  step if the project is structured as a package — you lose entry points,
  metadata, and dependency resolution guarantees.

## Common mistakes

- Flat layout accidentally "working" locally because the current
  directory happens to be importable, masking a missing/incorrect package
  configuration that only shows up once deployed elsewhere.
- Forgetting `-e` and being confused why code changes don't take effect
  until a manual reinstall.
- Not declaring `[tool.<backend>.packages.find]` (or backend equivalent)
  correctly, so `pip install .` silently installs nothing importable, or
  installs unintended files.
- Committing `dist/`, `build/`, or `*.egg-info/` build artifacts to version
  control instead of `.gitignore`-ing them.

## Interview questions

- What problem does the `src` layout solve compared to a flat layout?

    **Answer:** It stops your code from being accidentally importable just
    because it happens to sit in the current working directory. With
    `src/`, the package *must* be installed (even editable) before it can
    be imported, which catches missing packaging config early instead of
    in production.

- What does `pip install -e .` actually do differently from a normal
    install?

    **Answer:** A normal install copies files into `site-packages`; an
    editable install points Python at your working source tree instead, so
    edits to the source take effect immediately without reinstalling.

- What's the difference between a wheel and an sdist?

    **Answer:** A wheel (`.whl`) is a prebuilt binary distribution — fast
    to install, no build step needed. An sdist (`.tar.gz`) is raw source
    that gets built at install time if no matching wheel exists.

- Why might a project "work" locally with a flat layout but fail once
    actually packaged and installed elsewhere?

    **Answer:** Locally, Python can import the package straight from the
    current directory even without a real install, hiding a broken or
    missing `[build-system]`/package-discovery config. Once installed
    properly elsewhere (Docker image, another machine), that "free"
    cwd-relative import disappears and the app fails to find the package.

- When would you *not* want an editable install (e.g. in CI/production)?

    **Answer:** In CI/production you want to install a real, immutable
    wheel/sdist (or lockfile-pinned build) so what's tested is exactly
    what's deployed — an editable install ties the running code to a local
    source tree that shouldn't exist in that environment.

## Senior-level considerations

- The `src` layout is now the broadly recommended default for anything
  beyond a trivial script, precisely because it surfaces packaging
  mistakes early (in development) rather than late (in production) — a
  meaningful reliability win for teams shipping installable internal
  packages. For example, a missing `__init__.py` or bad package-discovery
  config throws an `ImportError` the moment you run `pytest` locally,
  instead of only failing after a Docker image ships to production.
- Editable installs and reproducible production installs are different
  concerns: use editable installs for local dev velocity, but production/
  CI images should install from a lockfile-pinned, non-editable build to
  guarantee what's running matches what was tested. For example, a
  Dockerfile should run `uv sync --frozen --no-editable`, not `pip install
  -e .`.
- Understanding the wheel/sdist distinction matters when your organization
  publishes internal packages to a private package index — wheel-only
  publishing speeds up install times across many services/CI jobs
  compared to sdist-only, which requires a build step per install. For
  example, publishing only an sdist for a package with a C extension
  forces every consuming service to compile it on every CI run instead of
  just downloading a prebuilt wheel.
