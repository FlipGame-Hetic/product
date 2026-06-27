# Deploying the Demo App

> **Who this is for:** students or developers deploying this demo application
> onto the shared Flipper cabinet. It covers pushing your code, understanding
> what runs inside the cabinet, and how to ship updates.
>
> **You never need to touch the cabinet's configuration.** Deploying only
> involves your own repository and two buttons in the dashboard.

## Overview

When the cabinet loads this app, it:

1. Pulls pre-built Docker images from GHCR (GitHub Container Registry).
2. Starts all services defined in `deploy/docker-compose.yml`.
3. Flashes the ESP32 firmware from `firmware/build/firmware.bin`.
4. Points all three cabinet screens at your running services.

Everything is automated — there is no manual file copying.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     Flipper Cabinet                      │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │ Playfield   │  │  Backglass  │  │      DMD        │   │
│  │  (screen 1) │  │  (screen 2) │  │   (screen 3)    │   │
│  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘   │
│         │                │                  │            │
│  ┌──────▼──────────────────────────────────▼────────┐   │
│  │                    nginx :80                      │   │
│  │           (reverse proxy + health checks)         │   │
│  └───────────────────────┬───────────────────────────┘   │
│                          │                               │
│              ┌───────────▼────────────┐                  │
│              │      api (backend)     │                  │
│              │      :8080             │                  │
│              └──────┬──────────┬──────┘                  │
│                     │          │                         │
│          ┌──────────▼──┐  ┌────▼──────────┐             │
│          │ mosquitto   │  │  mqtt-bridge   │             │
│          │ (MQTT :1883 │◄─┤  (WS ↔ MQTT)   │             │
│          │  WS :9001)  │  │                │             │
│          └─────────────┘  └────────────────┘             │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │                   ESP32 chip                       │  │
│  │        (flashed with firmware/build/firmware.bin)  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Services at a glance

| Service       | Image                                  | Role                                    |
|---------------|----------------------------------------|-----------------------------------------|
| `front-screen`| `ghcr.io/flipgame-hetic/front-screen`  | Playfield UI (Vite app, port 80)        |
| `back-screen` | `ghcr.io/flipgame-hetic/back-screen`   | Backglass UI (Vite app, port 80)        |
| `dmd-screen`  | `ghcr.io/flipgame-hetic/dmd-screen`    | DMD display UI (Vite app, port 80)      |
| `api`         | `ghcr.io/flipgame-hetic/backend`       | REST API + WebSocket hub (port 8080)    |
| `mqtt-bridge` | `ghcr.io/flipgame-hetic/mqtt-bridge`   | Bridges MQTT ↔ backend WebSocket        |
| `nginx`       | `nginx:alpine`                         | Reverse proxy, exposes port 80          |
| `mosquitto`   | `eclipse-mosquitto:2`                  | MQTT broker (TCP 1883, WS 9001)         |

---

## Step-by-step deployment

### 1. Join the cabinet's network

The cabinet is not on the public internet it lives on a private
[Tailscale](https://tailscale.com) network.

1. Accept the invite link your instructor gave you and sign in.
2. Install the Tailscale client on your machine and sign in with the same account.
3. Leave Tailscale running while you work.

> Do this once per machine.

### 2. Open the dashboard
For the IP please just check on Discord. 
With Tailscale connected, navigate to:

```
http://xxx.xxx.xxx:8080
```

Log in (or register if your instructor has opened registration).

### 3. Register and load your app

```
Dashboard
  └── Apps
        ├── [+ Register]  ← paste your GitHub repo URL here
        │       Cabinet clones the repo and validates fliphetic.toml.
        │       Errors in the manifest are shown inline.
        │
        └── [Load]        ← switches the cabinet to your app
                Cabinet stops the current app, flashes the ESP32,
                pulls updated Docker images, and starts your services.
```

### 4. Ship an update

```
You                    GitHub              Cabinet
 │                        │                   │
 ├─ git commit & push ───►│                   │
 │                        │                   │
 │                        │  (update badge    │
 │                        │   appears after   │
 │                        │   ~1 minute)      │
 │                        │                  ◄│── polls your repo
 │                        │                   │
 ├─── click [Load] ──────────────────────────►│
 │                        │                   │
 │                        │         pulls images + restarts services
```

> The cabinet **never** auto-deploys. You always trigger the update manually
> with **Load**.

---

## SSH access (rarely needed)

Deploying does **not** require SSH. If your instructor specifically asks you
to open a terminal on the cabinet:

```sh
ssh flipper@xxx.xxx.xxx.xxx
```

Use this only when asked. A terminal gives you access to the whole cabinet —
a mistake here can break the setup for the entire class.

---

## On-site Wi-Fi

When you are physically next to the cabinet it broadcasts its own network:

- **SSID:** check discord
- **Password:** check discord

Your normal deploy workflow uses Tailscale, not this Wi-Fi.

---

## Do NOT change the cabinet configuration

The cabinet is shared. Its system settings screens, ESP32 devices, users,
network config are configured once and used by everyone.

- You never need to touch them.
- Changing a setting to fix *your* app can silently break *everyone else's*.
- If the cabinet looks misconfigured, **tell your instructor**.

Stay inside your own repository and the **Apps** page and you cannot break
anything for anyone.
