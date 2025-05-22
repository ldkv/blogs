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

For that purpose, the uv documentation already provides a very comprehensive [guide to use uv in Docker](https://docs.astral.sh/uv/guides/integration/docker/#using-uv-in-docker).

This post will cover an edge case that is not mentioned in the documentation, which is using intermediate layers to speed up the `build` AND `pull` processes.

# Example project

Let's consider a simple `FastAPI` project with the following requirements files:

-   `requirements-big.txt` with heavy packages that take a long time to install but rarely need to be updated:

```
tensorflow==2.19.0
torch==2.7.0
```

-   `requirements.txt` with light packages:

```
fastapi==0.115.12
uvicorn==0.34.2
```

-   `requirements-dev.txt` for development dependencies:

```
pytest==8.3.5
ruff==0.11.11
```

And a `Dockerfile` that uses `pip` to install the dependencies:

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements-big.txt .

RUN pip install -r requirements-big.txt

COPY requirements.txt .

RUN pip install -r requirements.txt
```

The migration is as simple as creating a `pyproject.toml` file to declare the dependencies.

```toml
[project]
name = "test-project"
version = "0.0.0"
requires-python = "==3.13.*"
dependencies = ["fastapi==0.115.12", "torch==2.7.0", "uvicorn==0.34.2"]

[dependency-groups]
dev = ["pytest==8.3.5", "ruff==0.11.11"]
```

This is a minimal `pyproject.toml` file that declares the main dependencies of the project, and the development dependencies in the `dev` group. The `requires-python` field allows to specify the Python version that the project runs on.

To generate the virtual environment, simply run the following command:

```bash
uv sync
```

The command will also create an auto-generated `uv.lock` file that locks the dependencies of the project. This is super useful to ensure that the dependencies and sub-dependencies are the same across different machines.

As we can see, switching to uv is quite straight-forward for most projects. However, there exists some edge cases that prevent teams from adopting it, such as adapting an existing `Dockerfile`.

This post is dedicated to demonstrate some methods to adapt the `Dockerfile` to use `uv` for different scenarios. The same example project will be used throughout this post.

# Covered scenarios

Let's say the current `Dockerfile` is as follows:

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .
```

The uv documentation provides a very comprehensive [guide to use uv in Docker](https://docs.astral.sh/uv/guides/integration/docker/#using-uv-in-docker).

Let's consider a simple `Dockerfile` that uses `pip` to install the dependencies:

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

CMD ["uv", "run", "fastapi", "dev"]
```
