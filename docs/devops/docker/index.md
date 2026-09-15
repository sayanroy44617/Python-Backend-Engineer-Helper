# Docker

Packaging a Python backend service and its dependencies into a portable,
reproducible container: images, layers, Dockerfiles, volumes, networking,
and Docker Compose for local multi-service development.

This section assumes familiarity with Python environment/dependency
management (see
[Environments and Installers](../../packaging/01-environments-and-installers.md))
— a Dockerfile largely automates the same `venv`/`pip`/`uv` steps inside
a reproducible, isolated image rather than introducing new concepts.

## Topics

1. [Images, Containers, and Dockerfiles](01-images-containers-and-dockerfiles.md)
2. [Volumes, Networks, and Environment Variables](02-volumes-networks-and-environment-variables.md)
3. [Docker Compose](03-docker-compose.md)
4. [Multi-Stage Builds for Python](04-multi-stage-builds-for-python.md)
