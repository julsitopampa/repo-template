# Project Name

Short description of the repository, its role in the homelab/infrastructure, and target environment.

## Architecture & Overview

- **Component**: ...
- **Target Host / Environment**: ...
- **Dependencies**: ...

## Getting Started

### 1. Local Toolchain (`mise` + `hk`)

This repository uses [mise](https://mise.jdx.dev/) for reproducible tool management and [hk](https://hk.jdx.dev/) for pre-commit and commit-msg hooks.

```bash
mise trust
mise install
```

### 2. Run Quality Checks

```bash
# Check all files (secrets, workflows, YAML, shell, formatting)
hk check --all

# Auto-fix formatting issues
hk fix --all
```

## Documentation

- [Git & GitHub Workflow Guide](docs/GIT_WORKFLOW.md)
