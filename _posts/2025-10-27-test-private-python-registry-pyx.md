---
layout: post
title: test-private-python-registry-pyx
date: 2025-10-27 22:46 +0100
---

## Introduction

This is a test post to verify the functionality of a private Python package registry using PyX.

## Configuration

The following configuration is used for the PyX registry:

```toml
[[tool.uv.index]]
name = "pyx"
url = "https://api.pyx.dev/v1/upload/skynopy/main/"
publish-url = "https://api.pyx.dev/v1/upload/skynopy/main"
explicit = true
```

## Dependency Groups

The project is organized into several dependency groups for better management:

```toml
[dependency-groups]
dev = ["pytest==8.4.2", "ruff==0.14.2"]
heavy-rarely-updated = ["torch==2.9.0"]
light-frequently-updated = ["fastapi==0.120.1", "uvicorn==0.38.0"]
```

## Test mirror download speed

-   With Pyx registry: 27.72s
-   Without Pyx registry (default PyPI index): 43.43s 45.83s
-   
