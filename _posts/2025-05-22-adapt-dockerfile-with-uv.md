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

For that purpose, the uv documentation already provides a very comprehensive [guide to use uv in Docker](https://docs.astral.sh/uv/guides/integration/docker/#using-uv-in-docker), and a repository with [multiple examples](https://github.com/astral-sh/uv-docker-example) that covers most use cases.

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
{: .prompt-info }

To generate the virtual environment, simply run the following command:

```shell
uv sync
```

The command will also create an auto-generated `uv.lock` file that locks the dependencies of the project. This is super useful to ensure that the dependencies and sub-dependencies are the same across different machines.

# Legacy Dockerfile with pip install

A typical `Dockerfile` that uses `pip` to install the dependencies would be as follows:

```dockerfile
FROM python:3.12.10-slim-bookworm

WORKDIR /app

# LAYER 1: heavy dependencies, rarely updated
COPY requirements-heavy.txt .
RUN pip install -r requirements-heavy.txt

# LAYER 2: light dependencies, frequently updated
COPY requirements.txt .
RUN pip install -r requirements.txt

# LAYER 3: source code - install the project as a package
COPY . .
RUN pip install .

CMD ["uvicorn", "main:app"]
```

This is quite a common structure where the dependencies are installed in 3 separate [cache layers](https://docs.docker.com/build/cache/) to speed up the `build` process. It allows us to modify the light dependencies and the source code without invalidating the heavy dependencies layer, which takes a long time to install.

The separate layers also speed up the `pull` process on the client side, as it only needs to download the heavy dependencies layer once and reuse it for future deployments.

> This is particularly useful for client machines with limited bandwidth or unstable Internet connection. This is a specific use case that I encountered at work, which inspired the writing of this post.
{: .prompt-info }

# Adapt Dockerfile with uv

At first sight, the obvious questions that comes to mind are:

1. If we make unrelated changes to `pyproject.toml`, such as updating the project name or version, would the whole layer be invalidated?
2. How can we adapt a multi-layer structure with one single `pyproject.toml` file instead of multiple requirements files?

Let's try multiple approaches to adapt the `Dockerfile` to use `uv`, from the simplest to the most complex.

## Approach 1: single dependency layer

The first and simplest approach is covered in the uv [documentation](https://docs.astral.sh/uv/guides/integration/docker/#intermediate-layers) and [example](https://github.com/astral-sh/uv-docker-example/blob/main/Dockerfile). It should answer the first question:

```dockerfile
FROM python:3.12.10-slim-bookworm
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Enable bytecode compilation and copy mode
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_LOCKED=1

# LAYER 1 + 2: install HEAVY and LIGHT dependencies
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --no-install-project --no-dev

# LAYER 3: source code - install the project as a package
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --no-dev

CMD ["uvicorn", "main:app"]
```
