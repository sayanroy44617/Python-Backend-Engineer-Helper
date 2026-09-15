# Multi-Stage Builds for Python

## What

A **multi-stage build** uses multiple `FROM` instructions in one
Dockerfile, where each stage can selectively copy only the artifacts it
needs from a previous stage — letting you use heavyweight build tools in
an early stage without including them in the final image.

## Why

Building a Python application often needs tools the running application
doesn't (compilers for packages with C extensions, build-time
dependencies, `uv`/`pip` caches). Including all of that in the final
production image needlessly bloats it and expands its attack surface —
multi-stage builds let you build in one stage and ship only the final,
minimal runtime artifacts in another.

## How

### A single-stage Dockerfile (the naive approach)

```dockerfile
FROM python:3.12

WORKDIR /app
RUN apt-get update && apt-get install -y build-essential  # needed to compile some packages
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen
COPY . .
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

This works, but the final image permanently includes `build-essential`
(a full C compiler toolchain) even though it's only needed *during* the
dependency install step, not at runtime — larger image, more installed
packages that could carry vulnerabilities, for no runtime benefit.

### The same build, multi-stage

```dockerfile
# --- Stage 1: build ---
FROM python:3.12 AS builder

WORKDIR /app
RUN apt-get update && apt-get install -y build-essential
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

# --- Stage 2: runtime ---
FROM python:3.12-slim

WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .

ENV PATH="/app/.venv/bin:$PATH"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

The `builder` stage has the compiler toolchain and builds the virtual
environment; the final stage starts fresh from a smaller `slim` base and
copies over *only* the already-built `.venv` directory — `build-essential`
never exists in the final image at all.

### `COPY --from=<stage>`

```dockerfile
COPY --from=builder /app/.venv /app/.venv
```

This copies files from a *previous build stage's filesystem*, not from
the host machine — the mechanism that makes multi-stage builds work:
each stage is otherwise isolated, and only explicitly copied artifacts
carry over.

### Naming stages for clarity

```dockerfile
FROM python:3.12 AS builder
# ...
FROM python:3.12-slim AS runtime
COPY --from=builder /app/.venv /app/.venv
```

`AS builder`/`AS runtime` gives each stage a name, making
`COPY --from=builder` explicit and the Dockerfile self-documenting — more
readable than referencing stages by numeric index (`--from=0`), which
still works but obscures intent.

### Multi-stage builds for compiled dependencies

```dockerfile
FROM python:3.12 AS builder
RUN apt-get update && apt-get install -y build-essential libpq-dev
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM python:3.12-slim
RUN apt-get update && apt-get install -y libpq5  # runtime lib only, not the -dev headers
COPY --from=builder /app/.venv /app/.venv
```

Some packages (e.g. `psycopg2` compiling against PostgreSQL's client
library) need development headers (`libpq-dev`) *at build time* but only
the runtime shared library (`libpq5`) *at run time* — multi-stage builds
let each stage install exactly what it needs, keeping the final image as
minimal as possible.

### Testing inside a build stage

```dockerfile
FROM python:3.12 AS builder
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen
COPY . .
RUN uv run pytest   # fails the build here if tests fail

FROM python:3.12-slim AS runtime
COPY --from=builder /app /app
```

Running the test suite as part of an intermediate build stage means a
broken build simply fails to produce an image at all — a lightweight way
to enforce "never ship an image whose tests don't pass" directly in the
build process, complementary to running tests in CI (see
[Pipeline Stages: Build, Test, and Package](../cicd/02-pipeline-stages-build-test-and-package.md)).

### Image size comparison

```
Single-stage (with build-essential left in): ~450MB
Multi-stage (slim runtime, no build tools):  ~180MB
```

The size difference is a real, measurable outcome of applying
multi-stage builds correctly to a typical Python service with any
compiled dependencies — smaller images pull faster onto new nodes and
reduce the attack surface (fewer installed packages/tools that could
carry vulnerabilities).

## When to use

- Any Python application with dependencies requiring build-time-only
  tools (compilers, `-dev` header packages) that aren't needed at
  runtime.
- Running the test suite as a build stage, to fail the build (not just a
  separate CI step) if tests don't pass.
- Any production image, as a default habit — the size/security benefit
  rarely has a real downside.

## When NOT to use

- Don't bother with multi-stage builds for a trivial application with no
  compiled dependencies and negligible image size difference — the
  added Dockerfile complexity should be justified by an actual benefit.
- Don't skip copying only the necessary artifacts (`COPY --from=builder`
  targeting specific paths) in favor of copying the entire builder
  stage's filesystem, which reintroduces the bloat multi-stage builds are
  meant to eliminate.

## Common mistakes

- Installing build-time-only tools (compilers, `-dev` packages) in a
  single-stage Dockerfile, leaving them in the final production image
  unnecessarily.
- Copying more than necessary from the builder stage (e.g. the whole
  `/app` directory including build caches) instead of just the specific
  built artifacts needed at runtime.
- Not pinning the base image version consistently across stages,
  potentially introducing subtle inconsistencies between the build and
  runtime environment.
- Forgetting to also install genuinely necessary *runtime* shared
  libraries (e.g. `libpq5` for a compiled PostgreSQL driver) in the final
  stage, causing the application to fail at startup despite a successful
  build.

## Interview questions

1. What problem do multi-stage builds solve that a single-stage
   Dockerfile can't?
2. How does `COPY --from=builder` work, and why is naming stages with
   `AS` useful?
3. Why might a package need a `-dev` header package at build time but
   only a plain runtime library at container runtime?
4. What's the benefit of running your test suite as a build stage rather
   than only as a separate CI step?
5. What real-world impact does a smaller final image size have beyond
   just disk space?

## Senior-level considerations

- Multi-stage builds are one of the highest-leverage, lowest-risk
  Dockerfile optimizations available — meaningfully smaller, more secure
  images for a modest amount of added Dockerfile structure, making them
  close to a default best practice for production Python images.
- Image size and attack surface reduction compound at scale — across
  hundreds of deployments and image pulls, a smaller image measurably
  improves deployment speed and reduces the vulnerability surface an
  organization has to track and patch.
- Treating the Dockerfile itself as build automation (running tests,
  linting, or other CI-style checks as intermediate stages) blurs the
  line between "build" and "CI" in a useful way — a broken build failing
  to even produce an image is a strong, hard-to-bypass guarantee.
