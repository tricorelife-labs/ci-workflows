# CI Workflows

Reusable CI/CD workflows for all Rust projects in `tricorelife-labs`.

## Architecture

```
Your Repo                          ci-workflows (this repo)
.github/workflows/ci.yml          .github/workflows/
  jobs:                              ├── rust-ci.yml       ← fmt + clippy + build (parallel)
    ci:    ───calls──▶               ├── docker-build.yml  ← image → GHCR
    docker:───calls──▶               └── deploy.yml        ← SSH → target server
    deploy:───calls──▶

Self-hosted Runners (IDC server)
  ├── sccache   (shared compilation cache, all runners)
  ├── mold      (fast linker, 10-20x vs ld)
  └── cargo registry (persistent volume, no re-download)
```

## Integration Guide

### Step 1: Choose your level

| Level | What you get | Time |
|-------|-------------|------|
| **A. CI only** | fmt + clippy + build & test | 2 min |
| **B. CI + Docker** | A + build image → push to GHCR | 5 min |
| **C. CI + Docker + Deploy** | B + auto deploy to server | 10 min |

---

### Level A: CI Only

> For repos that only need code quality checks. No Docker, no deploy.

Create `.github/workflows/ci.yml` in your repo:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: tricorelife-labs/ci-workflows/.github/workflows/rust-ci.yml@main
```

Done. Push to trigger. fmt / clippy / build will run in parallel on self-hosted runners.

---

### Level B: CI + Docker Image

> For repos that need a Docker image published to GHCR.

**Prerequisites:**
- Your repo has a `Dockerfile` in the root (or specify path via `dockerfile` input)

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
```

On push to main: CI passes → Docker image built → pushed to `ghcr.io/tricorelife-labs/<repo>:latest`.

---

### Level C: CI + Docker + Deploy

> Full pipeline: code check → image build → deploy to target server.

**Prerequisites:**
1. Your repo has a `Dockerfile`
2. Target server has Docker + Docker Compose installed
3. Target server has a `docker-compose.yml` that references your image

**Step 1 — Set GitHub Secrets**

Go to your repo → Settings → Secrets and variables → Actions → New repository secret:

| Secret | Example | Description |
|--------|---------|-------------|
| `DEPLOY_SSH_KEY` | `-----BEGIN OPENSSH...` | SSH private key (ed25519 or RSA) |
| `DEPLOY_HOST` | `10.0.1.50` | Target server IP or hostname |
| `DEPLOY_USER` | `ubuntu` | SSH user on target server |

**Step 2 — Target server setup**

```bash
# On the target server:
mkdir -p /opt/your-app
cd /opt/your-app

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  app:
    image: ${IMAGE:-ghcr.io/tricorelife-labs/your-repo:latest}
    restart: unless-stopped
    env_file: .env
    ports:
      - "8080:8080"
EOF

# Create .env with your app's config
echo "DATABASE_URL=postgres://..." > .env
```

**Step 3 — Add workflow**

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

Push to main → CI → Docker build → Deploy. Full pipeline.

---

## Customization

All inputs have sensible defaults. Override only what you need:

```yaml
jobs:
  ci:
    uses: tricorelife-labs/ci-workflows/.github/workflows/rust-ci.yml@main
    with:
      toolchain: "nightly"                # default: stable
      cargo_test_args: "--workspace -j4"  # default: --all
      skip_test: true                     # default: false

  docker:
    uses: tricorelife-labs/ci-workflows/.github/workflows/docker-build.yml@main
    with:
      dockerfile: "deploy/Dockerfile"     # default: Dockerfile
      build_args: |                       # multi-line build args
        FEATURES=gpu
        PROFILE=release

  deploy:
    uses: tricorelife-labs/ci-workflows/.github/workflows/deploy.yml@main
    secrets: inherit
    with:
      deploy_path: /opt/my-app
      compose_file: prod.yml              # default: docker-compose.yml
      environment: staging                # default: production
```

## All Inputs Reference

### rust-ci.yml

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `toolchain` | string | `stable` | Rust toolchain version |
| `cargo_test_args` | string | `--all` | Args for `cargo test` |
| `cargo_clippy_args` | string | `--all-targets -- -D warnings` | Args for `cargo clippy` |
| `runner_labels` | string | `["self-hosted","linux","x64","liwei"]` | Runner labels (JSON array) |
| `skip_test` | boolean | `false` | Skip test step |

### docker-build.yml

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `dockerfile` | string | `Dockerfile` | Path to Dockerfile |
| `context` | string | `.` | Docker build context |
| `image_name` | string | `ghcr.io/<owner>/<repo>` | Override image name |
| `build_args` | string | `""` | Docker build args (KEY=VALUE per line) |
| `runner_labels` | string | `["self-hosted","linux","x64","liwei"]` | Runner labels |

**Outputs:** `image` (pushed tag), `digest` (image digest)

### deploy.yml

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `deploy_path` | string | *required* | Remote docker-compose directory |
| `compose_file` | string | `docker-compose.yml` | Compose file name |
| `environment` | string | `production` | GitHub environment name |
| `runner_labels` | string | `["self-hosted","linux","x64","liwei"]` | Runner labels |

**Required secrets:** `DEPLOY_SSH_KEY`, `DEPLOY_HOST`, `DEPLOY_USER`

## Dockerfile Template

If your repo doesn't have a Dockerfile yet, here's a template for Rust projects:

```dockerfile
# Build stage
FROM rust:1.82-slim AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    pkg-config libssl-dev ca-certificates && rm -rf /var/lib/apt/lists/*
WORKDIR /build
COPY Cargo.toml Cargo.lock ./
# Copy all member Cargo.toml files (adjust for your workspace)
# COPY crate-a/Cargo.toml crate-a/
# COPY crate-b/Cargo.toml crate-b/
COPY . .
RUN --mount=type=cache,target=/root/.cargo/registry \
    --mount=type=cache,target=/root/.cargo/git \
    --mount=type=cache,target=/build/target \
    cargo build --release --bin your-binary && cp target/release/your-binary /app

# Runtime stage
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates libssl3 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /app ./app
ENTRYPOINT ["./app"]
```

## FAQ

**Q: My repo is private. Can it call these workflows?**
A: Yes. Org settings allow all repos to use reusable workflows from any repo in the org.

**Q: How fast is CI?**
A: First run ~5-10 min (cold sccache). Subsequent runs ~1-3 min (sccache hits + mold linker).

**Q: Can I use GitHub-hosted runners instead?**
A: Yes, override `runner_labels`:
```yaml
with:
  runner_labels: '["ubuntu-latest"]'
```
Note: sccache/mold optimizations only apply to our self-hosted runners.

**Q: How do I pin to a specific version instead of `@main`?**
A: Use a tag or commit SHA:
```yaml
uses: tricorelife-labs/ci-workflows/.github/workflows/rust-ci.yml@v1.0.0
uses: tricorelife-labs/ci-workflows/.github/workflows/rust-ci.yml@abc1234
```
