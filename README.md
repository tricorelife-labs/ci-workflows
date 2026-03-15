# CI Workflows

Reusable CI/CD workflows for all Rust projects in `tricorelife-labs`.

## Workflows

| Workflow | Description |
|----------|-------------|
| `rust-ci.yml` | fmt + clippy + build & test (parallel) |
| `docker-build.yml` | Docker image build, push to GHCR |
| `deploy.yml` | SSH deploy via docker compose |

## Quick Start

Create `.github/workflows/ci.yml` in your repo:

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: tricorelife-labs/ci-workflows/.github/workflows/rust-ci.yml@main

  docker:
    needs: ci
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: tricorelife-labs/ci-workflows/.github/workflows/docker-build.yml@main
    permissions:
      contents: read
      packages: write

  deploy:
    needs: docker
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: tricorelife-labs/ci-workflows/.github/workflows/deploy.yml@main
    secrets: inherit
    with:
      deploy_path: /opt/your-app
```

## CI Only (no build/deploy)

```yaml
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: tricorelife-labs/ci-workflows/.github/workflows/rust-ci.yml@main
```

## Inputs

### rust-ci.yml

| Input | Default | Description |
|-------|---------|-------------|
| `toolchain` | `stable` | Rust toolchain version |
| `cargo_test_args` | `--all` | Extra args for cargo test |
| `cargo_clippy_args` | `--all-targets -- -D warnings` | Extra args for cargo clippy |
| `runner_labels` | `["self-hosted","linux","x64","liwei"]` | Runner labels (JSON) |
| `skip_test` | `false` | Skip test step |

### docker-build.yml

| Input | Default | Description |
|-------|---------|-------------|
| `dockerfile` | `Dockerfile` | Path to Dockerfile |
| `context` | `.` | Docker build context |
| `image_name` | `ghcr.io/<owner>/<repo>` | Override image name |
| `build_args` | | Docker build args |

### deploy.yml

| Input | Default | Description |
|-------|---------|-------------|
| `deploy_path` | *required* | Remote compose directory |
| `compose_file` | `docker-compose.yml` | Compose file name |
| `environment` | `production` | GitHub environment name |

## Required Secrets (for deploy)

Set in each repo's Settings > Secrets:

| Secret | Description |
|--------|-------------|
| `DEPLOY_SSH_KEY` | SSH private key for target server |
| `DEPLOY_HOST` | Target server IP or hostname |
| `DEPLOY_USER` | SSH username |

## Runner Optimizations

Self-hosted runners come pre-configured with:

- **sccache** shared compilation cache across all runners
- **mold** fast linker (10-20x vs default ld)
- **Shared cargo registry** persistent volume mount
