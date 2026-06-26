# Global Architecture

## What this project is

A real-time backend for a connected pinball machine. Physical buttons and sensors on the machine
communicate via MQTT. The backend processes game logic and pushes updates to physical display
screens and a web frontend over WebSocket.

---

## Workspace layout

```
Backend/
├── Cargo.toml              ← workspace root (resolver = "3")
├── Cargo.lock
├── Dockerfile              ← multi-stage cargo-chef build
├── docker-compose.yml      ← api + mqtt-bridge + mosquitto + nginx
├── .github/
│   └── workflows/
│       ├── ci.yml          ← lint → build → test on every PR
│       ├── security.yml    ← cargo-audit / dependency scan
│       └── docker-publish.yml
├── contracts/              ← MQTT topics + WebSocket message specs (reference)
├── nginx/                  ← reverse proxy config
├── migrations/             ← SQLx SQL migration files
└── crates/
    ├── api/                ← HTTP + WebSocket server  (binary)
    ├── game-logic/         ← game engine              (library)
    ├── mqtt-bridge/        ← MQTT ↔ WebSocket relay   (binary)
    ├── screen-hub/         ← screen registry          (library)
    └── shared/             ← cross-crate types        (library)
```

---

## Crate dependency graph

```
              ┌─────────┐
              │  shared │  ← no internal deps
              └────┬────┘
       ┌───────────┼────────────┐
       ▼           ▼           ▼
  ┌──────────┐ ┌──────────┐ ┌────────────┐
  │   api    │ │game-logic│ │screen-hub  │
  └────┬─────┘ └──────────┘ └────────────┘
       │ uses game-logic + screen-hub
       │
  ┌────┴──────────┐
  │  mqtt-bridge  │  ← only depends on shared
  └───────────────┘
```

- **shared** is a pure types crate — zero business logic, imported by everyone.
- **game-logic** is a pure library — no I/O, no HTTP, no async runtime dependency.
- **screen-hub** is a pure library — screen connection management only.
- **api** is the integration layer — it wires all libraries together and exposes HTTP + WebSocket.
- **mqtt-bridge** is an independent binary — bridges MQTT broker to the api WebSocket.

---

## Full system data flow

```
  Physical Machine (ESP32)
         │
         │  MQTT publish  (topic: pinball/<device_id>/input/button)
         ▼
  ┌──────────────┐
  │   Mosquitto  │  MQTT broker (port 1883)
  │   broker     │
  └──────┬───────┘
         │ subscribe to pinball/#
         ▼
  ┌──────────────┐         WebSocket (/ws/bridge)
  │ mqtt-bridge  │ ──────────────────────────────────►┐
  └──────────────┘  WsMessage { dir: inbound, ... }    │
                                                       │
  ┌────────────────────────────────────────────────────▼─┐
  │                         api                           │
  │                                                       │
  │  ws_handler ──► mqtt.rs ──► GameService              │
  │                                 │                     │
  │                          GameEngine (game-logic)      │
  │                                 │                     │
  │                        ScreenEnvelope events          │
  │                                 │                     │
  │                          screen-hub registry          │
  │                         /│\             /│\           │
  └──────────────────────────┼──────────────┼────────────-┘
                             │              │
                    WebSocket│              │ WebSocket
                 /ws/screen/front    /ws/screen/back
                             │              │
                    ┌────────┴──┐    ┌──────┴──────┐
                    │FrontScreen│    │ BackScreen  │
                    └───────────┘    └─────────────┘
                    (physical LCD)   (physical LCD)

  Web browser / admin panel
         │
         │  HTTP REST
         ▼
  ┌──────────────┐
  │     api      │
  │  /api/v1/*   │
  └──────────────┘
```

---

## Runtime concurrency model

The entire server runs on a single multi-threaded **Tokio** runtime.

```
  Tokio runtime
  ├── axum HTTP listener          (main task)
  ├── per-connection WS tasks     (spawned on accept)
  │   ├── /ws/bridge handler
  │   └── /ws/screen/{id} handler
  ├── rail ticker tasks           (spawned on game start)
  ├── PVE cooldown ticker         (spawned on boss kill)
  └── screen guard cleanup tasks  (spawned on WS disconnect)
```

Shared mutable state lives in `AppState` behind `Arc<Mutex<>>`:

```
AppState {
    game_engine:    Arc<Mutex<GameEngine>>    ← lock first
    active_session: Arc<Mutex<Option<...>>>  ← lock second (never reverse)
    rail_sessions:  Arc<Mutex<HashMap<...>>>
    screen_hub:     Arc<ScreenRegistry>       ← no Mutex, uses channels
    db:             SqlitePool                ← connection pool, no mutex needed
    broadcast_hub:  BroadcastHub             ← unbounded sender, clone freely
}
```

> **Lock ordering rule**: always acquire `game_engine` before `active_session`.
> Reversing the order risks a deadlock.

---

## Environment variables

| Variable          | Required | Default                      | Used by     |
|-------------------|----------|------------------------------|-------------|
| `SCREEN_JWT_SECRET` | **Yes** | —                            | api         |
| `DATABASE_URL`    | No       | `sqlite:///data/flipper.db`  | api         |
| `API_PORT`        | No       | `8080`                       | api         |
| `ALLOWED_ORIGINS` | No       | `http://localhost:3000`      | api         |
| `MQTT_HOST`       | No       | `localhost`                  | mqtt-bridge |
| `MQTT_PORT`       | No       | `1883`                       | mqtt-bridge |
| `WS_URL`          | No       | `ws://localhost:8080/ws/bridge` | mqtt-bridge |

---

## Docker compose topology

```
  ┌──────────┐    ┌─────────────┐    ┌──────────────┐
  │  nginx   │◄──│     api      │◄──│ mqtt-bridge  │
  │ :80      │    │ :8080       │    │              │
  └──────────┘    └──────┬──────┘    └──────┬───────┘
                         │ SQLite            │ MQTT
                         ▼                  ▼
                    ┌───────────┐      ┌──────────────┐
                    │ /data/    │      │  mosquitto   │
                    │flipper.db │      │ :1883 / :9001│
                    └───────────┘      └──────────────┘
```

---

## Database schema

```
scores
┌────┬───────────┬───────┬───────────────────────┐
│ id │ character │ score │ created_at            │
├────┼───────────┼───────┼───────────────────────┤
│ 1  │ KEENU     │ 9800  │ 2026-06-27T10:00:00Z  │
└────┴───────────┴───────┴───────────────────────┘
  INDEX on (score DESC)
  Cap: top-10 only (evict min on insert when full)

game_config
┌────┬─────┬───────┐
│ id │ key │ value │
├────┼─────┼───────┤
│ 1  │ ... │ ...   │   ← serialized JSON blob of GameConfig
└────┴─────┴───────┘
  Hot-patched via PATCH /api/v1/admin/config (no restart needed)
```
