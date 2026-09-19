# Docker Compose

## What

**Docker Compose** defines and runs a multi-container application from a
single declarative YAML file — instead of manually running several
`docker run`/`docker network`/`docker volume` commands, `docker compose
up` starts the whole stack (application, database, cache, etc.)
together, wired up correctly.

## Why

A realistic backend service rarely runs in isolation — it needs a
database, likely a cache, maybe a message broker, all networked together
for local development. Manually recreating this with individual `docker
run` commands (and remembering the right network/volume/environment
flags every time) is tedious and error-prone; Compose captures the whole
setup as version-controlled, reproducible configuration.

## How

### A basic `docker-compose.yml`

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/app
      REDIS_URL: redis://cache:6379/0
    depends_on:
      - db
      - cache

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data

  cache:
    image: redis:7

volumes:
  pgdata:
```

```bash
docker compose up -d      # start everything in the background
docker compose logs -f api
docker compose down       # stop and remove containers (volumes persist by default)
```

Compose automatically creates a shared network for all services in the
file — `api` can reach the database at hostname `db` and the cache at
hostname `cache` exactly as described in
[Volumes, Networks, and Environment Variables](02-volumes-networks-and-environment-variables.md),
without any manual network setup.

### `depends_on` only controls start order, not readiness

```yaml
services:
  api:
    depends_on:
      - db  # starts db's container first, but doesn't wait for
            # Postgres to actually be ready to accept connections
```

`depends_on` guarantees container *start* order, not that the dependency
is actually ready to serve requests — Postgres's container can be
"started" while the database engine inside is still initializing. The
application itself needs its own retry/backoff logic on startup (or a
`healthcheck` + `condition: service_healthy`, below) to handle this gap.

### Health checks for real readiness

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    depends_on:
      db:
        condition: service_healthy
```

Combining a `healthcheck` with `condition: service_healthy` makes `api`
actually wait until the database passes its readiness check (not just
until its container starts) — the Compose-level analog of the
[readiness probes](../../observability/04-health-checks-and-alerting.md)
covered in Observability.

### Environment-specific overrides

```yaml
# docker-compose.override.yml (automatically merged with docker-compose.yml)
services:
  api:
    volumes:
      - ./app:/app   # bind mount for live code reload, local dev only
    environment:
      LOG_LEVEL: debug
```

Compose automatically merges `docker-compose.override.yml` on top of the
base file — a common pattern for adding local-development-only behavior
(bind mounts, debug logging) without touching the base file used for
CI/other environments; a separate `docker-compose.prod.yml` (loaded
explicitly with `-f`) is used similarly for production-specific overrides.

### `.env` file integration

```
# .env
POSTGRES_PASSWORD=devpassword
```

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

Compose automatically reads a `.env` file in the project directory and
substitutes `${VAR}` references in the Compose file — the same "fine for
local dev, not a production secrets mechanism" caveat from
[Volumes, Networks, and Environment Variables](02-volumes-networks-and-environment-variables.md)
applies here too.

### Scaling a service locally

```bash
docker compose up -d --scale api=3
```

Running multiple instances of one service locally (behind a manual load
balancer, or just to exercise multi-instance behavior like connection
pool sizing across instances) — a lightweight way to sanity-check
assumptions that matter more once you reach real orchestration (see
[Kubernetes](../kubernetes/index.md)).

### Compose for local development vs. production orchestration

```
Compose:     ideal for local development and simple single-host setups
Kubernetes:  the standard choice for production-grade orchestration
             (multi-host scheduling, self-healing, rolling deployments)
```

Compose is explicitly a local-development and simple-deployment tool —
it doesn't provide the self-healing, multi-host scheduling, or rolling
deployment capabilities a production system typically needs; those
belong to a real orchestrator (see [Kubernetes](../kubernetes/index.md)
for depth).

## When to use

- Local development environments needing multiple coordinated services
  (app + database + cache + broker) started/stopped together.
- CI pipelines that need a real database/cache running for integration
  tests (see
  [Test Strategy and Isolation](../../testing/03-test-strategy-and-isolation.md)).
- `condition: service_healthy` whenever a service's startup genuinely
  depends on another being *ready*, not just started.

## When NOT to use

- Don't use Compose as a production orchestration solution for anything
  beyond a very simple, single-host deployment — it lacks the
  self-healing and scheduling capabilities of a real orchestrator.
- Don't rely on `depends_on` alone (without a health check) when a
  service genuinely needs its dependency to be *ready*, not just started.
- Don't put real production secrets into a Compose file or its `.env` —
  treat both the same way as any other local-development-only secrets
  mechanism.

## Common mistakes

- Assuming `depends_on` waits for a dependency to be fully ready,
  causing intermittent startup failures when the app connects before the
  database has finished initializing.
- Committing a `.env` file containing real secrets to version control.
- Using Compose in production without understanding its lack of
  multi-host scheduling, automatic failover, and rolling deployment
  support compared to Kubernetes.
- Not separating local-development-only configuration (bind mounts, debug
  logging) from the base Compose file, making it harder to reuse the same
  file across environments cleanly.

## Interview questions

- What problem does Docker Compose solve that plain `docker run`
    commands don't?

    **Answer:** It lets you describe the whole app stack in one file so everyone starts the same services, networks, ports, and volumes with one command instead of remembering a long sequence of `docker run` flags.

    ```bash
    docker compose up -d
    docker compose logs -f api
    ```

- Why doesn't `depends_on` alone guarantee a dependency is ready to
    accept connections, and how would you fix that?

    **Answer:** `depends_on` only means Docker starts the other container first; it does not mean Postgres or Redis inside that container is ready yet. Add a health check and make the app retry on startup so brief timing differences do not break the service.

    ```yaml
    depends_on:
     db:
       condition: service_healthy
    ```

- How does Compose enable inter-service communication by hostname
    without manual network configuration?

    **Answer:** Compose creates a shared network for the project and registers each service name in that network's DNS. That means `api` can call `db:5432` or `cache:6379` directly without you creating a bridge network by hand.

    ```yaml
    environment:
     DATABASE_URL: postgresql://user:pass@db:5432/app
    ```

- What's the practical difference between Docker Compose and
    Kubernetes, and when would you outgrow Compose?

    **Answer:** Compose is great for one-machine workflows like local dev, smoke testing, or simple single-host deployments. You outgrow it when you need production orchestration features like self-healing, rolling deploys, service discovery across nodes, or autoscaling.

    ```bash
    docker compose up -d        # local stack
    kubectl rollout restart deployment/api  # production orchestrator flow
    ```

- How would you structure Compose configuration to support different
    settings for local development vs. CI?

    **Answer:** Keep the base file close to the shared setup, then layer environment-specific overrides on top so local-only bind mounts or debug settings do not leak into CI. That keeps one mental model while still letting each environment add what it needs.

    ```bash
    docker compose -f docker-compose.yml -f docker-compose.ci.yml up --build
    ```

## Senior-level considerations

- Compose is genuinely valuable as a local-development and CI tool, but
  recognizing exactly where its capabilities stop (no multi-host
  scheduling, no automatic failover, no rolling deployments) prevents
  reaching for it inappropriately in a production context — for example,
  using Compose for `api + db + redis` in CI is fine, but using it as
  the failover story for a multi-node production platform is the wrong
  tool.
- Startup ordering and readiness (health checks, retry/backoff in
  application code) matter more than they first appear — a system that
  "usually works" locally because services happen to start in a
  convenient order can fail unpredictably once deployed to an environment
  with different timing characteristics. For example, an API that boots
  fine on a laptop might fail in CI if Postgres takes 8 seconds longer
  to become ready and the app exits after one connection attempt.
- Keeping local Compose configuration close to production configuration
  (same image, same environment variable names, same service topology
  where practical) reduces the classic "works in Compose, breaks in
  production" class of surprises — for example, using the same
  `DATABASE_URL` shape locally and in production avoids code paths that
  only exist in one environment.
