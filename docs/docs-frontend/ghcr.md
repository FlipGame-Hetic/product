# GHCR — GitHub Container Registry

This document explains how Docker images are built and pushed to the GitHub Container Registry (GHCR), which tags are produced, how to pull images locally, and how to extend the pipeline.

---

## What is GHCR

GitHub Container Registry (`ghcr.io`) is the package registry attached to the GitHub organization. Each frontend app is published as a separate image under the `flipgame-hetic` organization:

```
ghcr.io/flipgame-hetic/front-screen
ghcr.io/flipgame-hetic/back-screen
ghcr.io/flipgame-hetic/dmd-screen
```

Images are public (readable without authentication) but can only be pushed by CI with a valid `GITHUB_TOKEN`.

---

## When Images Are Published

The workflow `.github/workflows/docker-publish.yml` triggers on two events:

```
Push to main branch   -->  builds and pushes all 3 images
Release published     -->  builds and pushes all 3 images with semver tag
```

Pull requests do not push images. They only run builds and tests (see `ci.yml`).

---

## Tag Strategy

Every push produces multiple tags for the same image:

```
Event: push to main
  ghcr.io/flipgame-hetic/front-screen:main
  ghcr.io/flipgame-hetic/front-screen:sha-a1b2c3d
  ghcr.io/flipgame-hetic/front-screen:latest

Event: release v1.2.0
  ghcr.io/flipgame-hetic/front-screen:1.2.0
  ghcr.io/flipgame-hetic/front-screen:sha-a1b2c3d
  ghcr.io/flipgame-hetic/front-screen:latest
```

| Tag | Description |
|---|---|
| `latest` | Always points to the most recent push on `main` |
| `main` | Branch name; same as `latest` for this repo |
| `sha-<short>` | Git commit SHA — immutable and safe for rollback |
| `<semver>` | Produced only when a GitHub Release is published |

For deployments, prefer the `sha-` tag or a semver tag. The `latest` tag is convenient but not pinned.

---

## Build Pipeline Detail

The workflow runs a matrix strategy: all three apps build in parallel.

```
docker-publish.yml
       |
       v
matrix: [front-screen, back-screen, dmd-screen]
       |
       +----> for each app:
              |
              1. checkout
              2. login to ghcr.io (GITHUB_TOKEN)
              3. setup Docker Buildx
              4. extract metadata (tags + labels)
              5. build image from apps/<app>/Dockerfile
                 context: . (repo root — needed for pnpm workspace)
              6. push to ghcr.io/flipgame-hetic/<app>
              7. write buildcache back to registry
```

### Registry-based cache

Each image uses a `buildcache` tag for layer caching between runs:

```yaml
cache-from: type=registry,ref=ghcr.io/flipgame-hetic/<app>:buildcache
cache-to:   type=registry,ref=ghcr.io/flipgame-hetic/<app>:buildcache,mode=max
```

This means the second build after a dependency-only change (no source changes) reuses all cached layers and completes in seconds.

---

## Pulling Images Locally

```bash
# Pull the latest stable image
docker pull ghcr.io/flipgame-hetic/front-screen:latest

# Pull a specific commit
docker pull ghcr.io/flipgame-hetic/front-screen:sha-a1b2c3d

# Run it locally (inject env vars at runtime)
docker run -p 3002:80 \
  -e VITE_WS_URL=ws://localhost/ws/bridge \
  -e VITE_SCREEN_HUB_URL=ws://localhost \
  -e VITE_API_URL=http://localhost:8080 \
  -e VITE_SCREEN_TOKEN=my-token \
  -e VITE_ENVIRONMENT=local \
  ghcr.io/flipgame-hetic/front-screen:latest
```

All `VITE_*` environment variables are injected at container start by `entrypoint.sh`, not at build time. The same image can be used in any environment.

---

## Dockerfile Anatomy

Each app has an identical single-stage Dockerfile (no multi-stage needed because the workspace dependencies require the full context):

```
FROM node:22-slim
  |
  v
Install pnpm 10.30.1 via corepack
  |
  v
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/<app>/package.json
COPY packages/ ./packages/
  |
  v
pnpm install --frozen-lockfile        (cached layer)
  |
  v
COPY apps/<app>/ ./apps/<app>/
  |
  v
pnpm --filter @frontend/<app> build   (tsc + vite)
  |
  v
EXPOSE 80
CMD entrypoint.sh                     (injects window.__ENV__, starts preview server)
```

The build context is the repo root (not the app directory) because pnpm workspaces need `pnpm-workspace.yaml` and the `packages/` directory.

---

## Adding a New App to GHCR

If a fourth screen is added, follow these steps:

1. Create `apps/<new-app>/Dockerfile` following the existing pattern.
2. Create `apps/<new-app>/entrypoint.sh` writing the appropriate `window.__ENV__` variables.
3. Add the app to the matrix in `docker-publish.yml`:

```yaml
strategy:
  matrix:
    app: [front-screen, back-screen, dmd-screen, new-app]   # <-- add here
```

4. Add the app to the matrix in `ci.yml` (build and e2e jobs).
5. Create a GitHub environment named `new-app` (in repository Settings > Environments) and add the required secrets for that environment.

---

## Permissions and Secrets

The workflow uses `GITHUB_TOKEN` (automatically provided by GitHub Actions) with these permissions:

```yaml
permissions:
  contents: read
  packages: write
```

No additional secrets are needed to push to GHCR. The token is scoped to the repository and expires after the workflow run.

Build-time secrets (e.g., `VITE_API_URL`) are pulled from GitHub Environments. Each app has a matching environment name in the repository settings where those secrets live.

---

## Checking Published Images

```bash
# List tags for an image (requires gh CLI + jq)
gh api /orgs/flipgame-hetic/packages/container/front-screen/versions \
  | jq '.[].metadata.container.tags'

# Or via Docker Hub CLI
docker manifest inspect ghcr.io/flipgame-hetic/front-screen:latest
```

You can also browse images directly on GitHub: `github.com/orgs/flipgame-hetic/packages`.
