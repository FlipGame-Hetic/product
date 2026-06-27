# Environments and Docker

This document covers how environment variables work in this project, how to run the stack with Docker, and the relationship between build-time and runtime configuration.

---

## Environment Variables

All environment variables used by the apps are prefixed with `VITE_`. This is a Vite convention — only `VITE_`-prefixed variables are exposed to the browser bundle.

### Variables per app

**front-screen**

| Variable | Example | Description |
|---|---|---|
| `VITE_WS_URL` | `ws://localhost/ws/bridge` | WebSocket endpoint for the game backend |
| `VITE_SCREEN_HUB_URL` | `ws://localhost` | Screen-hub WebSocket for inter-screen events |
| `VITE_API_URL` | `http://localhost:8080` | REST API base URL |
| `VITE_SCREEN_TOKEN` | `my-secret-token` | Authentication token for screen identity |
| `VITE_ENVIRONMENT` | `local` | Environment label (`local`, `staging`, `production`) |

**back-screen**

| Variable | Example | Description |
|---|---|---|
| `VITE_SCREEN_HUB_URL` | `ws://localhost` | Screen-hub WebSocket |
| `VITE_API_URL` | `http://localhost:8080` | REST API base URL |
| `VITE_SCREEN_TOKEN` | `my-secret-token` | Screen authentication token |
| `VITE_ENVIRONMENT` | `local` | Environment label |

**dmd-screen**

| Variable | Example | Description |
|---|---|---|
| `VITE_SCREEN_HUB_URL` | `ws://localhost` | Screen-hub WebSocket |
| `VITE_SCREEN_TOKEN` | `my-secret-token` | Screen authentication token |

---

## Setting Up Local Environment Files

Each app ships with a `.env.example` file. Copy it to `.env` before running the dev server:

```bash
cp apps/front-screen/.env.example  apps/front-screen/.env
cp apps/back-screen/.env.example   apps/back-screen/.env
cp apps/dmd-screen/.env.example    apps/dmd-screen/.env
```

Then edit the `.env` files with values matching your local backend setup. `.env` files are git-ignored and never committed.

---

## Build-time vs. Runtime Configuration

This is the most important concept to understand before deploying:

```
Development (pnpm dev)
  .env file is read by Vite at startup
  VITE_* vars are inlined into the JS bundle
  Changing .env requires restarting the dev server

Production (Docker container)
  .env is NOT used — the image is pre-built
  entrypoint.sh reads process.env at container start
  entrypoint.sh writes dist/config.js  -->  window.__ENV__
  App reads window.__ENV__ via getEnv() at runtime
  Changing values only requires restarting the container
```

This means the Docker image is environment-agnostic. You pass different `VITE_*` variables to `docker run` and the app reconfigures itself without a rebuild.

### How getEnv works

```ts
// packages/utils/src/env.ts
export function getEnv(key: string): string {
  // In Docker: window.__ENV__ is written by entrypoint.sh
  // In dev:    window.__ENV__ is undefined, falls back to import.meta.env
  return (window as any).__ENV__?.[key] ?? import.meta.env[key] ?? ""
}
```

Usage in any app:

```ts
const wsUrl = getEnv("VITE_WS_URL")
```

---

## Local Development (without Docker)

```bash
# Install all dependencies (run once)
pnpm install

# Start all three apps in parallel
pnpm dev

# Or start a single app
pnpm dev:front    # front-screen on http://localhost:3000
pnpm dev:back     # back-screen  on http://localhost:3001
pnpm dev:dmd      # dmd-screen   on http://localhost:3002
```

Hot Module Replacement (HMR) is enabled by default. File changes reflect immediately in the browser.

---

## Docker — Local Stack

The `docker-compose.yml` at the repo root builds and runs all three apps as containers.

### Port mapping

```
Host port   Container port   Service
3000     -->     80          back-screen
3001     -->     80          dmd-screen
3002     -->     80          front-screen
```

### Starting the stack

```bash
# Build images and start all containers
docker compose up --build

# Start without rebuilding (uses cached images)
docker compose up

# Run in background
docker compose up -d

# Stop and remove containers
docker compose down
```

### Passing environment variables to containers

Add an `environment` block in `docker-compose.yml` for each service:

```yaml
services:
  front_screen:
    build: ...
    ports:
      - 3002:80
    environment:
      VITE_WS_URL: ws://localhost/ws/bridge
      VITE_SCREEN_HUB_URL: ws://localhost
      VITE_API_URL: http://localhost:8080
      VITE_SCREEN_TOKEN: local-token
      VITE_ENVIRONMENT: local
```

Or use an env file:

```yaml
services:
  front_screen:
    env_file:
      - apps/front-screen/.env
```

### Health checks

Each container exposes a health check via `wget`:

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://localhost:80/ || exit 1"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

Check container health:

```bash
docker inspect multi-react-apps-front_screen | jq '.[0].State.Health'
```

### Resource limits

Containers are constrained to avoid overloading the host machine:

```
front_screen:  0.2 CPU,  128 MB RAM
back_screen:   0.3 CPU,  256 MB RAM
dmd_screen:    0.3 CPU,  256 MB RAM
```

If you need more resources during local development, edit the `deploy.resources.limits` section in `docker-compose.yml`.

---

## Docker — Full Build Flow

```
docker compose up --build
       |
       v
Dockerfile (each app):
  1. FROM node:22-slim
  2. Install pnpm 10.30.1 via corepack
  3. Copy pnpm-workspace.yaml, lock file, root package.json
  4. Copy apps/<app>/package.json + packages/
  5. pnpm install --frozen-lockfile   (uses Docker layer cache)
  6. Copy apps/<app>/src + config files
  7. pnpm --filter @frontend/<app> build
  8. EXPOSE 80
  9. CMD: entrypoint.sh
       |
       v
entrypoint.sh:
  1. Read VITE_* from process.env
  2. Write dist/config.js with window.__ENV__
  3. exec pnpm preview --host 0.0.0.0 --port 80
```

### Layer caching tip

Dependencies are installed before source files are copied. This means `pnpm install` is only re-run when `pnpm-lock.yaml` or a `package.json` changes. Source-only changes skip the install step entirely, making subsequent builds fast.

---

## CI Environments

GitHub Actions uses named environments (configured in repository Settings > Environments) to inject secrets into CI builds. Each app has a matching environment:

| GitHub Environment | Secrets it provides |
|---|---|
| `front-screen` | `VITE_WS_URL`, `VITE_SCREEN_HUB_URL`, `VITE_API_URL`, `VITE_SCREEN_TOKEN`, `VITE_ENVIRONMENT` |
| `back-screen` | `VITE_SCREEN_HUB_URL`, `VITE_API_URL`, `VITE_SCREEN_TOKEN`, `VITE_ENVIRONMENT` |
| `dmd-screen` | `VITE_SCREEN_HUB_URL`, `VITE_SCREEN_TOKEN` |

These secrets are only available to workflows running in those environments and are never logged.

---

## Adding a New Environment Variable

1. Add it to `apps/<app>/.env.example` with a placeholder value.
2. Add it to `apps/<app>/entrypoint.sh` so it is written into `window.__ENV__`.
3. Add it to the matching GitHub Environment secrets in repository settings.
4. Read it in the app with `getEnv("VITE_NEW_VAR")`.

```sh
# entrypoint.sh (add the new variable here)
const env = {
  VITE_SCREEN_TOKEN:   process.env.VITE_SCREEN_TOKEN   || '',
  VITE_WS_URL:         process.env.VITE_WS_URL         || '',
  VITE_NEW_VAR:        process.env.VITE_NEW_VAR        || '',   # <-- add this
};
```

Do not hardcode any secret or URL directly in source files. All external addresses must go through environment variables.
