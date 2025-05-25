---
layout: post
title: Dockerfile with intermediate layers for dependencies using uv
date: 2025-05-22 00:21 +0200
---

# Introduction

[uv](https://docs.astral.sh/uv/) is a modern, fast, Python package and project manager. It is arguably the best thing happened to the Python ecosystem in recent years.

Before its arrival, I had to use a combination of multiple tools to manage a project's dependencies and environments, such as `pip`, `venv`, `pyenv`, `poetry`, etc. And it required multiple steps to setup dev environment for a new project, which is quite error-prone. `uv` is a single tool to replace all of them, and I don't even need Python installed on my machine to use it.

There has been a lot of hype around uv since its release, and plenty of resources explaining its benefits and how to use it. So I won't delve into the details here, since it is not the focus of this post.

Switching to uv is quite straight-forward for most projects. However, there exists some edge cases that prevent teams from adopting it, such as adapting an existing `Dockerfile`.

For that purpose, the uv documentation already provides a very comprehensive [guide to use uv in Docker](https://docs.astral.sh/uv/guides/integration/docker/#using-uv-in-docker), and a repository with [multiple examples](https://github.com/astral-sh/uv-docker-example) that covers most use cases, including multi-stage builds.

This post is dedicated to an edge case that is not detailed in the documentation, which is **generating multiple dependency layers to optimize the `build` AND `pull` processes.**

First, let's define an example project and go through some minimal steps to migrate it to `uv`.

# Migrate existing project to uv

Let's consider a simple `FastAPI` project with the following requirements files:

-   `requirements-heavy.txt` with heavy packages that take a long time to install but rarely need to be updated:

```
tensorflow==2.19.0
torch==2.7.0
```

-   `requirements.txt` with light packages that are frequently updated:

```
fastapi==0.115.12
uvicorn==0.34.2
```

-   `requirements-dev.txt` for development dependencies:

```
pytest==8.3.5
ruff==0.11.11
```

To migrate the project to `uv`, we can simply create a `pyproject.toml` file to declare the dependencies.

```toml
[project]
name = "test-project"
version = "0.0.0"
requires-python = "==3.12.*"
dependencies = []

[dependency-groups]
dev = ["pytest==8.3.5", "ruff==0.11.11"]
heavy-rarely-updated = ["tensorflow==2.19.0", "torch==2.7.0"]
light-frequently-updated = ["fastapi==0.115.12", "uvicorn==0.34.2"]

[tool.uv]
default-groups = ["dev", "heavy-rarely-updated", "light-frequently-updated"]
```

This is a minimal `pyproject.toml` file that declares the main dependencies of the project in separate groups, and the development dependencies in the `dev` group.

> The `requires-python` field specifies the Python version that the project should run on, which will be picked up automatically by `uv` to generate the virtual environment.
> If the specific version does not exist on the machine, `uv` will automatically download it for immediate and future use without any input from the user. Very handy!
> {: .prompt-info }

To generate the virtual environment, simply run the following command:

```shell
uv sync
```

The command will also create an auto-generated `uv.lock` file that locks the dependencies of the project. This is super useful to ensure that the dependencies and sub-dependencies are the same across different machines.

# Legacy Dockerfile with pip install

A typical `Dockerfile` that uses `pip` to install the dependencies would be as follows:

```dockerfile
FROM python:3.12.10-slim-bookworm

# System dependencies: gcc for dependencies building
RUN apt-get update && apt-get install -y --no-install-recommends gcc

WORKDIR /app

# LAYER 1: heavy dependencies, rarely updated
COPY requirements-heavy.txt .
RUN pip install -r requirements-heavy.txt

# LAYER 2: light dependencies, frequently updated
COPY requirements.txt .
RUN pip install -r requirements.txt

# LAYER 3: source code
COPY src src

CMD ["uvicorn", "main:app"]
```

This is quite a common structure where the dependencies are installed in 3 separate [cache layers](https://docs.docker.com/build/cache/) to speed up the `build` process. It allows us to modify the light dependencies and the source code without invalidating the heavy dependencies layer, which takes a long time to install.

The separate layers also speed up the `pull` process on the client side, as it only needs to download the heavy dependencies layer once and reuse it for future deployments.

> This is particularly useful for client machines with limited bandwidth or unstable Internet connection. It is a specific use case that I encountered at work, which inspired the writing of this post.
> {: .prompt-warning }

# Adapt Dockerfile with uv

At first sight, the obvious questions that comes to mind are:

1. If we make unrelated changes to `pyproject.toml`, such as updating the project name or version, would the whole layer be invalidated?
2. How can we adapt a multi-layer structure with one single `pyproject.toml` file instead of multiple requirements files?
3. How to define a multi-stage build with multi-layer dependencies?

Let's try multiple approaches to adapt the `Dockerfile` to use `uv`, from the simplest to the most complex.

## Approach 1: single dependency layer

The first and simplest approach is covered in the uv [documentation](https://docs.astral.sh/uv/guides/integration/docker/#intermediate-layers) and [example](https://github.com/astral-sh/uv-docker-example/blob/main/Dockerfile). It should answer the first question:

```dockerfile
FROM python:3.12.10-slim-bookworm

# System dependencies should be the first layer before uv since they might be heavier and less likely to change.
RUN apt-get update && apt-get install -y --no-install-recommends gcc

# Copy uv binary from the official image instead of using base image with uv pre-installed.
# It allows to use the same image as before migration, and only install uv when needed.
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Force uv to use system Python instead of downloading it
ENV UV_PYTHON_DOWNLOADS=0

# Enable bytecode compilation to speed up the startup time
ENV UV_COMPILE_BYTECODE=1

# Copy from the cache instead of linking since it's a mounted volume
ENV UV_LINK_MODE=copy

# Force the build to fail if the auto-generated lock is not up to date
ENV UV_LOCKED=1

# LAYER 1 + 2: install HEAVY and LIGHT dependencies
# 1. The cache is mounted to avoid re-downloading the dependencies if the lock file is not updated.
# 2. The pyproject.toml and uv.lock files are bind-mounted which prevents the build from being invalidated by unrelated changes.
# 3. The --no-install-project flag is used to avoid installing the project as a package, since the source code has not been copied yet.
# 4. The --no-dev flag is used to avoid installing the development dependencies.
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --no-install-project --no-dev

# LAYER 3: source code
COPY src src

# Place executables in the environment at the front of the path
ENV PATH="/app/.venv/bin:$PATH"

CMD ["uvicorn", "main:app"]
```

Most of the comments are self-explanatory. In particular, to answer the first question, we combine 2 Docker techniques:

-   [cache mount](https://docs.docker.com/build/cache/optimize/#use-cache-mounts): persists the uv cache for packages installation across builds, so even if the layer is rebuilt, only new or changed packages are downloaded.
-   [bind mount](https://docs.docker.com/build/cache/optimize/#use-bind-mounts): links `pyproject.toml` and `uv.lock` from the host machine to the container for temporary use without generating any layer. If `COPY` is used here, it would generate a new layer that would be invalidated by any changes to the files.

Thus, this layer will only be invalidated if changes are made to the dependencies. Even so, it won't have to redownload everything, since the cache is mounted.

## Approach 2: multi-layer dependencies

The second approach is to split the dependencies into multiple layers, which should answer the second question. We will reuse the same `Dockerfile` and `RUN` command in the first approach, with only changes to the `RUN` command to separate layers 1 and 2 installation.

```dockerfile
# ... ENV variables ...

# LAYER 1: HEAVY dependencies
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --no-install-project --no-dev --only-group heavy-rarely-updated

# LAYER 2: LIGHT dependencies
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --no-install-project --no-dev --only-group light-frequently-updated

# ... LAYER 3 ...
```

The `--only-group` flag is used to install only the dependencies of the specified group, which replicates nicely the use of multiple requirements files.

This approach will not only speed up the `build` process in dev environment, but also the `pull` process for client machines, as they can cache the heavy dependencies layer once and reuse it for subsequent application updates.

## Approach 3: multi-layer dependencies

As you must have noticed, both approaches above are rather wasteful in terms of image size, as they must install `gcc` in system packages, and `uv` itself is not needed either during runtime. A minimal image size is important to speed up push and pull operations, and to minimize long-term storage cost.

To address this issue, we can use a multi-stage build to install the dependencies in a separate stage, then copy the virtual environment to the final image.

```dockerfile
# Build stage
FROM python:3.12.10-slim-bookworm AS builder

# Install build dependencies: gcc and uv
RUN apt-get update && apt-get install -y --no-install-recommends gcc
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Configure uv settings
ENV UV_PYTHON_DOWNLOADS=0 \
	UV_COMPILE_BYTECODE=1 \
	UV_LINK_MODE=copy \
	UV_LOCKED=1

WORKDIR /packages

# Install dependencies in separate layers
RUN --mount=type=cache,target=/root/.cache/uv \
	--mount=type=bind,source=uv.lock,target=uv.lock \
	--mount=type=bind,source=pyproject.toml,target=pyproject.toml \
	env UV_PROJECT_ENVIRONMENT=./heavy uv sync --no-install-project --no-dev --only-group heavy-rarely-updated && \
	env UV_PROJECT_ENVIRONMENT=./light uv sync --no-install-project --no-dev --only-group light-frequently-updated

# Final image without gcc and uv
FROM python:3.12.10-slim-bookworm

WORKDIR /app

ENV VENV_PATH=/app/.venv/

# Copy dependencies in separate layers
COPY --from=builder /packages/heavy $VENV_PATH
COPY --from=builder /packages/light $VENV_PATH

# Source code
COPY src src

ENV PATH="$VENV_PATH/bin:$PATH"

CMD ["uvicorn", "main:app"]
```

The biggest difference here is the use of environment variable `UV_PROJECT_ENVIRONMENT`. It is an official [uv configuration](https://docs.astral.sh/uv/concepts/projects/config/#project-environment-path), which is somewhat equivalent to the [`pip --prefix` option](https://pip.pypa.io/en/stable/cli/pip_install/#cmdoption-prefix). By customize this value, we can install each dependencies group into a separate directory, which is then copied to the final image, each copy being a separate layer.

With this approach, we can install all dependencies groups with a single `RUN` command during the build process, since they don't affect the final image layers at all. However, if it is necessary to optimize the build time, we can always separate the RUN command into multiple steps as in the second approach, with a small trade-off of a more bloated `Dockerfile`.

# Comparison with actual data

Now let's compare the difference between approaches with actual data. For each approach (including the legacy one), I will excecute the following steps and measure the execution time:

1. Cold build: build the image from scratch
2. Cold push: push the cold built image to the registry
3. Hot build: build the image again with a dependency update in `light-frequently-updated` group
4. Hot push: push the hot built image to the registry
5. Cold pull: pull the cold built image from the registry
6. Hot pull: pull the hot built image from the registry

| Approach   | Legacy   | Single layer | Multi-layer | Multi-stage |
| ---------- | -------- | ------------ | ----------- | ----------- |
| Cold build | 429.14s  | 302.21s      | 1m 30s      | 1m 30s      |
| Hot build  | 5.10s    | 207.43s      | 1m 30s      | 1m 30s      |
| Cold push  | 448.75s  | 250.55s      | 1m 30s      | 1m 30s      |
| Hot push   | 4.3s     | 229.12s      | 1m 30s      | 1m 30s      |
| Image Size | 19.48 GB | 12 GB        | 1.3 GB      | 1.3 GB      |

# Conclusion
