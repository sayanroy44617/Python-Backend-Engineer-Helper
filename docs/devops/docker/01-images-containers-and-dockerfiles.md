# Images, Containers, and Dockerfiles

## What

A Docker **image** is a read-only, layered filesystem snapshot containing
an application and everything it needs to run (dependencies, runtime,
config). A **container** is a running instance of an image — an isolated
process with its own filesystem view, network namespace, and resource
limits. A **Dockerfile** is the declarative recipe used to build an
image.

## Why

"Works on my machine" is a symptom of environment drift — different OS,
different library versions, different configuration between a
developer's laptop, CI, and production. A container packages the exact
runtime environment alongside the application itself, so the same image
runs identically everywhere it's deployed.

## How

### Image vs container

```
Image:      docker build -t myapp:1.0 .        -- a static, immutable artifact
Container:  docker run myapp:1.0                -- a running instance of it

# Many containers can run from the same image simultaneously,
# each with its own isolated process/filesystem/network namespace
```

An image is analogous to a class; a container is analogous to an
instance of it — the same image can be started as a container many times
(e.g. multiple replicas of the same service), each independent of the
others.

### A basic Dockerfile for a Python service

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

COPY . .

EXPOSE 8000
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`FROM` picks a base image (here, an official slim Python image);
`WORKDIR` sets the working directory for subsequent instructions;
`COPY` brings files from the build context into the image; `RUN`
executes a command at build time; `CMD` defines the default command run
when a container starts (not at build time).

### Layers

```dockerfile
FROM python:3.12-slim      # layer 1
WORKDIR /app                 # layer 2
COPY pyproject.toml uv.lock ./  # layer 3
RUN uv sync --frozen         # layer 4
COPY . .                     # layer 5
```

Each instruction creates a new, cached **layer**. Docker rebuilds only
the layers whose inputs changed (and every layer after them) — this is
why the order of instructions matters: putting rarely-changing steps
(installing dependencies) *before* frequently-changing ones (copying
application source) means code changes don't invalidate the (slow)
dependency-installation layer's cache.

### Why dependency installation comes before copying source code

```dockerfile
# GOOD: dependencies installed before app code is copied
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .

# BAD: any source change invalidates the dependency-install cache too,
# forcing a full reinstall on every build
COPY . .
RUN uv sync --frozen --no-dev
```

Because Docker invalidates a layer's cache whenever its inputs change,
copying only the dependency manifest files first (and installing from
them) means editing application code doesn't force Docker to reinstall
every dependency on the next build — a significant, easy build-time win.

### `.dockerignore`

```
# .dockerignore
.git
__pycache__
*.pyc
.venv
.env
tests/
```

Excludes files from the build context sent to the Docker daemon —
smaller build context means faster builds, and prevents accidentally
copying secrets (`.env`), local virtual environments, or version control
metadata into the image.

### Running a container

```bash
docker run -d -p 8000:8000 --name myapp-container myapp:1.0
docker logs -f myapp-container
docker exec -it myapp-container bash   # shell into a running container
docker stop myapp-container
```

`-p 8000:8000` maps a host port to the container's exposed port;
`-d` runs detached (in the background); `exec -it ... bash` is the
standard way to inspect/debug a running container interactively.

### Choosing a base image

```
python:3.12          -- full Debian-based image, largest, most compatible
python:3.12-slim      -- smaller Debian-based image, missing some build tools
python:3.12-alpine    -- smallest, but musl libc can cause subtle
                          incompatibilities with some Python packages
                          that ship compiled C extensions
```

`slim` is a common, reasonable default — smaller than the full image
without Alpine's occasional binary-compatibility surprises (some
packages with C extensions expect glibc, not musl, and either fail to
install or behave differently on Alpine).

### Running as a non-root user

```dockerfile
RUN useradd --create-home appuser
USER appuser
```

By default, a container's process runs as root inside the container —
if an attacker exploits a vulnerability in the application, running as a
dedicated non-root user limits what they can do within the container
(least privilege, same principle as
[Authorization Models and Least Privilege](../../security/03-authorization-and-least-privilege.md)
applied at the container level).

## When to use

- Order Dockerfile instructions from least-frequently-changing (base
  image, dependency install) to most-frequently-changing (application
  source) to maximize layer cache reuse.
- A `.dockerignore` file in every project, excluding secrets, VCS
  metadata, and local-only artifacts.
- A non-root user for the running process in any production image.

## When NOT to use

- Don't copy the entire project directory before installing dependencies
  — it defeats layer caching for the (usually slow) dependency install
  step.
- Don't default to Alpine images without verifying your dependencies
  (especially any with C extensions) actually work correctly against
  musl libc.
- Don't run application processes as root in a production container
  without a specific reason — it's an avoidable privilege escalation
  risk.

## Common mistakes

- `COPY . .` before installing dependencies, causing a full dependency
  reinstall on every single code change during development.
- No `.dockerignore`, accidentally including `.git`, local `.venv`
  directories, or `.env` secrets in the built image.
- Using the largest base image "just to be safe" without considering the
  image size/attack-surface trade-off `slim` usually offers with no real
  downside.
- Running as root by default and only discovering the security
  implication during a security review, rather than designing for
  least privilege from the start.

## Interview questions

1. What's the difference between a Docker image and a container?
2. Why does instruction order in a Dockerfile matter for build speed?
3. Why should dependency installation happen before copying application
   source code in a typical Python Dockerfile?
4. What's the trade-off between `python:3.12-slim` and
   `python:3.12-alpine` as a base image?
5. Why would you run a containerized process as a non-root user?

## Senior-level considerations

- Dockerfile structure directly affects both build speed (via layer
  caching) and image size/security posture (via base image choice,
  non-root user, minimal copied context) — these are engineering
  decisions with real operational impact, not just boilerplate to copy
  from a template.
- Reproducibility depends on pinning both the base image tag (avoiding
  a floating `latest` tag) and the application's own dependencies (see
  [pyproject.toml and Dependency Management](../../packaging/02-pyproject-and-dependency-management.md))
  — an unpinned Dockerfile can silently produce a different image on
  every build as upstream images/packages are updated.
- Image size and layer count affect not just build time but deployment
  speed (pulling a large image onto a new node) and attack surface (more
  installed packages/tools mean more potential vulnerabilities) — both
  are worth actively managing, not incidental side effects of "however
  the Dockerfile happened to be written."
