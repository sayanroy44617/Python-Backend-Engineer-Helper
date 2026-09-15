# Volumes, Networks, and Environment Variables

## What

**Volumes** provide persistent storage that survives beyond a
container's lifecycle. **Networks** let containers communicate with each
other (and the outside world) in isolated, configurable ways.
**Environment variables** are the standard mechanism for passing
configuration into a container without baking it into the image.

## Why

A container's own filesystem is ephemeral by design — deleting a
container destroys its filesystem changes, which is desirable for
application code (a fresh container should behave identically every
time) but wrong for genuinely persistent data (a database's actual data
files). Networking and environment variables solve the equally essential
problems of "how do containers talk to each other" and "how does the
same image behave differently across dev/staging/production without
being rebuilt."

## How

### Why a container's filesystem is ephemeral

```bash
docker run --name db1 postgres:16
# ... data gets written into the container ...
docker rm db1   # all that data is GONE -- the container's filesystem is deleted
```

Without a volume, any data written inside a container disappears the
moment the container is removed — fine for stateless application code,
unacceptable for a database or any service that must retain data across
restarts/redeployments.

### Named volumes

```bash
docker volume create pgdata
docker run -d --name db1 -v pgdata:/var/lib/postgresql/data postgres:16
docker rm db1  # the container is gone, but pgdata (the volume) persists
docker run -d --name db2 -v pgdata:/var/lib/postgresql/data postgres:16  # same data
```

A **named volume** is storage managed by Docker, independent of any
single container's lifecycle — mounting it at the same path in a new
container gives that container access to the exact same persisted data.

### Bind mounts (for local development)

```bash
docker run -v $(pwd)/app:/app myapp:1.0
```

A **bind mount** maps a specific host directory directly into the
container — commonly used in local development so code edits on the host
are immediately visible inside the running container (e.g. for hot
reload), without rebuilding the image for every change. Not typically
used in production, where the image itself should already contain the
correct, immutable code.

### Docker networks

```bash
docker network create app-network
docker run -d --name db --network app-network postgres:16
docker run -d --name api --network app-network myapp:1.0
```

Containers on the same user-defined network can reach each other by
container name as a hostname (`postgresql://db:5432/...` from within the
`api` container) — Docker's embedded DNS resolves container names within
a shared network automatically, without needing to know each other's IP
addresses (which can change on restart).

### Default bridge vs user-defined networks

```
Default bridge network: containers can only reach each other by IP,
                         not by name -- fragile and rarely used directly
User-defined network:   automatic DNS resolution by container name --
                         the standard choice for multi-container setups
```

Always create (or let Compose create) a user-defined network for
multi-container applications rather than relying on the default bridge
network's more limited, IP-only connectivity.

### Environment variables for configuration

```bash
docker run -e DATABASE_URL=postgresql://db:5432/app -e LOG_LEVEL=info myapp:1.0
```

```python
import os
DATABASE_URL = os.environ["DATABASE_URL"]
```

The same image runs correctly in dev, staging, and production purely by
varying the environment variables passed at `docker run` time — this is
what makes an image genuinely portable across environments without
rebuilding it per environment (see
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md)
for why actual secrets shouldn't be passed as plain environment variables
in production without a secrets manager backing them).

### Env files

```bash
# .env (not committed to version control)
DATABASE_URL=postgresql://db:5432/app
LOG_LEVEL=debug
```

```bash
docker run --env-file .env myapp:1.0
```

An `--env-file` is a convenience for local development (avoiding a long
list of `-e` flags) — the same durability/security caveats as any
`.env` file apply (see
[Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md));
it's not a production secrets mechanism on its own.

### Exposing ports

```dockerfile
EXPOSE 8000   # documents the port the app listens on -- doesn't publish it
```

```bash
docker run -p 8000:8000 myapp:1.0   # host_port:container_port -- this actually publishes it
```

`EXPOSE` in a Dockerfile is documentation/metadata — it doesn't actually
make the port reachable from the host. Only `-p`/`--publish` at `docker
run` time actually maps a host port to the container, which is why an
image can declare `EXPOSE 8000` and still be unreachable if run without
`-p`.

## When to use

- Named volumes for any container holding genuinely persistent data
  (databases, uploaded files) that must survive container recreation.
- Bind mounts for local development workflows needing live code reload
  without rebuilding the image.
- A user-defined Docker network for any multi-container setup, to get
  reliable container-name-based DNS resolution.
- Environment variables for all environment-specific configuration,
  keeping the image itself identical across environments.

## When NOT to use

- Don't rely on a container's own filesystem for data that must survive
  beyond that container's lifecycle.
- Don't use bind mounts in production — production images should be
  immutable, self-contained artifacts, not dependent on host filesystem
  state at runtime.
- Don't pass real production secrets as plain environment variables
  without a secrets manager backing them — see
  [Injection Attacks and Secrets Management](../../security/05-injection-attacks-and-secrets-management.md).

## Common mistakes

- Storing a database's data directory inside the container's own
  filesystem without a volume, then losing all data on the next
  `docker rm`/redeploy.
- Assuming containers on the default bridge network can reach each other
  by name, then being confused when DNS resolution fails.
- Forgetting that `EXPOSE` alone doesn't publish a port — only `-p` at
  `docker run` actually does.
- Baking environment-specific configuration directly into the image
  instead of injecting it via environment variables, requiring a
  separate image build per environment.

## Interview questions

1. Why is a container's own filesystem considered ephemeral, and what
   problem do volumes solve?
2. What's the difference between a named volume and a bind mount, and
   when would you use each?
3. Why do containers need a user-defined network (rather than the
   default bridge) to reliably resolve each other by name?
4. What does `EXPOSE` in a Dockerfile actually do, and what does it *not*
   do?
5. Why are environment variables the standard way to configure the same
   image differently across environments?

## Senior-level considerations

- Data persistence strategy (which services need volumes, and how those
  volumes are backed up) is a production-readiness concern that's easy to
  overlook until a container restart unexpectedly wipes data that was
  assumed to be durable.
- Container networking design (which services can reach which others,
  over which network) is effectively a lightweight network segmentation
  decision — worth deliberate design in multi-service systems, not just
  "put everything on one network."
- Treating the container image as immutable and injecting all
  environment-specific behavior via environment variables (the
  "build once, deploy everywhere" principle) is foundational to reliable,
  repeatable deployments across dev/staging/production.
