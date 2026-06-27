# Docker images and GHCR

## What is GHCR?

**GHCR** stands for **GitHub Container Registry** GitHub's built-in service
for hosting Docker images, at the address `ghcr.io`.

It is the Docker equivalent of npm for packages or PyPI for Python: a place
where pre-built images are stored and can be pulled from anywhere with an
internet connection.

```
GitHub repository
  └── GitHub Actions CI/CD
        │
        │  docker build → docker push
        ▼
  ghcr.io/flipgame-hetic/<image>:<tag>
        │
        │  docker pull  (no login required for public images)
        ▼
  Flipper Cabinet
```

---

## Why GHCR and not Docker Hub?

| Feature                  | GHCR (public)            | Docker Hub (free tier)         |
|--------------------------|--------------------------|--------------------------------|
| Pull without login       | Yes                      | Rate-limited (100/6h per IP)   |
| Storage                  | Free for public repos    | Limited on free plan           |
| CI/CD integration        | Native with GitHub Actions | Requires extra secrets        |
| Namespace                | `ghcr.io/<org>/<image>`  | `<user>/<image>`               |
| Versioning by branch/tag | Built-in (`main`, `v1.0`)| Manual                         |

For a shared cabinet that pulls images multiple times a day across many
students, GHCR public avoids Docker Hub's pull-rate limits entirely.

---

## Why public images?

This project uses **public** GHCR images no authentication token is needed
to pull them.

```
Cabinet                     GHCR (public)
   │                             │
   ├─ docker pull ──────────────►│
   │  ghcr.io/flipgame-hetic/    │
   │  backend:main               │
   │                             │
   │◄── image layers ────────────┤
   │    (no login, no token)     │
```

Benefits:
- **Zero credential management** the cabinet does not need to store any
  secrets to pull images.
- **Simple CI push** GitHub Actions pushes to GHCR using the built-in
  `GITHUB_TOKEN`; no external secret is required.
- **Reproducible** any student, any machine, any moment can pull the exact
  same image.

---

## How images are built and published

Each service lives in its own repository under the `flipgame-hetic` GitHub
organisation. When a commit is merged into `main`, a GitHub Actions workflow
builds the Docker image and pushes it to GHCR automatically:

```
Developer pushes to main
        │
        ▼
  GitHub Actions workflow
  ┌─────────────────────────────────────────────┐
  │  1. docker/login-action                     │
  │       → logs in to ghcr.io with GITHUB_TOKEN│
  │                                             │
  │  2. docker/build-push-action                │
  │       → builds the image                    │
  │       → pushes ghcr.io/<org>/<image>:main   │
  └─────────────────────────────────────────────┘
        │
        ▼
  ghcr.io/flipgame-hetic/<image>:main
  (public, immediately pullable)
```

The `:main` tag always points to the latest build from the `main` branch.
When the cabinet runs `docker compose pull` (triggered by **Load**), it fetches
the freshest `:main` image which is why `pull_policy: always` is set in
`docker-compose.yml`.

---

## Where the images are used in this project

`deploy/docker-compose.yml` references five GHCR images:

```yaml
front-screen:
  image: ghcr.io/flipgame-hetic/front-screen:main

back-screen:
  image: ghcr.io/flipgame-hetic/back-screen:main

dmd-screen:
  image: ghcr.io/flipgame-hetic/dmd-screen:main

api:
  image: ghcr.io/flipgame-hetic/backend:main

mqtt-bridge:
  image: ghcr.io/flipgame-hetic/mqtt-bridge:main
```

`nginx` and `mosquitto` use official public images (`nginx:alpine`,
`eclipse-mosquitto:2`) from Docker Hub — these are standard, well-maintained
images with no rate-limit concerns at this scale.

---

## What happens at deploy time

```
Cabinet receives [Load]
        │
        ├─ 1. docker compose pull
        │       Pulls the latest :main for each GHCR image.
        │       Skips if the local digest already matches.
        │
        ├─ 2. docker compose up -d
        │       Starts (or restarts) all services.
        │       Health checks gate the startup order:
        │         mosquitto → api → nginx
        │                  └──► mqtt-bridge
        │
        └─ 3. ESP32 flash
                Flashes firmware/build/firmware.bin onto the chip.
```

Because images are public and `pull_policy: always` is set, every **Load**
guarantees the cabinet is running the most recent build — no stale cache.
