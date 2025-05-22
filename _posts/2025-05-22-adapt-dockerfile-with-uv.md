---
layout: post
title: Adapt Dockerfile with uv
date: 2025-05-22 00:21 +0200
---

# What is uv?

[uv](https://docs.astral.sh/uv/) is a modern, fast, Python package and project manager. It is arguably the best thing happened to the Python ecosystem in recent years.

Before its arrival, Python developers had to use a combination of multiple tools to manage their dependencies and environments, such as `pip`, `venv`, `pyenv`, `poetry`, etc. And it required multiple steps to setup dev environment for a new project, which is quite error-prone and time-consuming.

# How to use uv in Dockerfile?

uv provides a `uv.lock` file to lock the dependencies of a project. It is a simple YAML file that lists the dependencies of a project.

# Dockerfile

```dockerfile

```
