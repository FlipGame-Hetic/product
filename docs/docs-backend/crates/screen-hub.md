# Crate: screen-hub

## Role

`screen-hub` manages the **WebSocket connections of physical screens**. When a screen device
connects, it registers itself. When the game engine emits events, the hub routes them to the right
screen(s).

It is a pure library no HTTP, no MQTT, no game logic. The `api` crate imports it.

---

## Source tree

```
crates/screen-hub/src/
├── lib.rs          ← public exports
├── registry.rs     ← ScreenRegistry, ScreenHandle, ScreenGuard
├── router.rs       ← ScreenRouter: interceptor pipeline before dispatch
└── error.rs        ← ScreenHubError enum
```

---

## Key types

### ScreenId

Identifies which physical (or virtual) screen a message targets:

```
FrontScreen   ← the screen facing the player
BackScreen    ← the screen on the back panel (leaderboard, boss health…)
DmdScreen     ← the dot-matrix display
GameEngine    ← virtual: messages emitted by the engine internally
```

### ScreenRegistry

The central map of live connections:

```
ScreenRegistry
└── inner: RwLock<HashMap<ScreenId, mpsc::Sender<ScreenEnvelope>>>
```

Each connected screen holds an `mpsc::Receiver`. The registry only holds the `Sender` side.
Sending a message is lock-free after the initial lookup.

### ScreenHandle

Returned when a screen successfully registers:

```
ScreenHandle {
    receiver: mpsc::Receiver<ScreenEnvelope>,
    guard:    ScreenGuard,              ← RAII: unregisters on drop
}
```

The WebSocket handler for a screen owns the `ScreenHandle` for the lifetime of the connection.
When the WebSocket closes, the handle is dropped, the guard runs, and the screen is unregistered.

### ScreenGuard (RAII pattern)

```rust
impl Drop for ScreenGuard {
    fn drop(&mut self) {
        // spawns a short async task to remove self.screen_id from the registry
        tokio::spawn(async move {
            registry.unregister(screen_id).await;
        });
    }
}
```

This guarantees cleanup even if the handler panics or the connection is abruptly closed.
No explicit `unregister()` call is needed anywhere in the code.

---

## Connection lifecycle

```
Screen device connects to /ws/screen/front
         │
         ▼
ws_handler.rs: validate JWT, extract screen_id
         │
         ├── screen already registered?
         │    YES → reject with 409 Conflict (screens are exclusive)
         │    NO  ──────────────────────────────────────────────────►
         │                                                          │
         ▼                                                          │
registry.register(screen_id) → ScreenHandle { receiver, guard }     │
         │                                                          │
         ▼                                                         │
ws_handler loop:                                                    │
   recv from receiver → forward as WS text frame to device ◄────────┘

Screen device disconnects
         │
         ▼
ScreenHandle dropped → ScreenGuard::drop() → registry.unregister()
```

---

## ScreenRouter and interceptors

`ScreenRouter` wraps the registry and lets you attach **interceptors** — middleware-like functions
that run before a message is dispatched:

```
ScreenRouter
└── interceptors: Vec<Box<dyn Interceptor>>
         │
         ▼ (called in order)
   validate → mutate → gate (drop if returns false)
         │
         ▼
   ScreenRegistry::send(screen_id, envelope)
```

Example use case: log every message, or drop `ScoreUpdate` events when no game is active.

---

## Channel back-pressure

Each screen's channel is bounded at **128 messages**. If a screen's WebSocket is too slow to
consume messages, the channel fills and new sends return `Err(TrySendError::Full)`. The registry
logs a warning and drops the message — a slow screen does not block the engine.

---

## How to add a new screen type

1. Add a variant to `ScreenId` in `shared/src/screen.rs`:

```rust
pub enum ScreenId {
    FrontScreen,
    BackScreen,
    DmdScreen,
    GameEngine,
    MyNewScreen,   // ← add here
}
```

2. Add a WebSocket route in `api/src/modules/screen/routes.rs`:

```rust
.route("/ws/screen/my_new_screen", get(ws_handler::handle_screen))
```

3. Parse the screen ID from the path in the handler.

4. Emit `ScreenEnvelope { to: ScreenId::MyNewScreen, … }` from the game engine where appropriate.

No changes needed in `screen-hub` itself — the registry handles any `ScreenId` variant generically.

---

## How to add a new screen event type

1. Add a variant to `ScreenEventType` in `shared/src/screen.rs`:

```rust
pub enum ScreenEventType {
    // … existing variants
    MyNewEvent,
}
```

2. Build the envelope in `game-logic/src/engine/core/emit.rs`:

```rust
pub fn my_new_event(from: ScreenId, to: ScreenId, payload: serde_json::Value) -> ScreenEnvelope {
    ScreenEnvelope {
        from,
        to,
        event_type: ScreenEventType::MyNewEvent,
        payload,
    }
}
```

3. Call `emit::my_new_event(...)` from the relevant location in the engine and include the result
   in the returned `Vec<ScreenEnvelope>`.

4. The `api` crate forwards the envelope to the registry — no changes needed there.
