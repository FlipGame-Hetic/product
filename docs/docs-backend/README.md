# Documentation Flipper Backend

Welcome to the backend documentation for the connected pinball machine.
This folder is your entry point for understanding the architecture, conventions, and best practices of the project.

---

## Table of Contents

| Document | Description |
|---|---|
| [Global Architecture](./architecture.md) | System overview, data flow diagrams |
| **Crates** | |
| [api](./crates/api.md) | HTTP/WebSocket server, routes, middlewares |
| [game-logic](./crates/game-logic.md) | Game engine, state machine, characters |
| [mqtt-bridge](./crates/mqtt-bridge.md) | MQTT ↔ WebSocket bridge |
| [screen-hub](./crates/screen-hub.md) | Screen connection registry |
| [shared](./crates/shared.md) | Shared types, DTOs, protocols |
| **Guides** | |
| [Testing](./testing.md) | How to write and organize tests |
| [DRY & SOLID Principles](./principles.md) | Why and how to apply them here |
| [Conventions](./conventions.md) | Code, branches, commits, PRs |

---

## Quick Overview

```
┌─────────────────────────────────────────────────────────┐
│                    FLIPPER BACKEND                      │
│                                                         │
│  ESP32 ──MQTT──► mqtt-bridge ──WS──► api               │
│                                       │                 │
│                               game-logic  screen-hub    │
│                                       │                 │
│                              shared (common types)      │
└─────────────────────────────────────────────────────────┘
```

Start with **[Global Architecture](./architecture.md)** before reading the individual crate docs.
