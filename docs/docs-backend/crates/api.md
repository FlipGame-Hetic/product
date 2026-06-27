# Crate: api

## Role

The `api` crate is the **integration layer** of the project. It is the only binary that the end user
talks to directly. It:

- Exposes the REST API (`/api/v1/*`)
- Manages WebSocket connections for the MQTT bridge and for physical screens
- Orchestrates the game session lifecycle
- Handles admin authentication
- Persists scores and config to SQLite

It does **not** contain game logic it delegates everything to `game-logic` and `screen-hub`.

---

## Source tree

```
crates/api/src/
├── main.rs               ← binary entry: tracing init, config load, DB migrate, server start
├── lib.rs                ← re-exports for integration tests
├── app.rs                ← builds the Axum router with CORS + tracing middleware
├── config.rs             ← reads env vars into AppConfig struct (unit-tested)
├── router.rs             ← merges all sub-routers, wires /docs, /ws endpoints
├── state.rs              ← AppState definition + constructor
├── errors.rs             ← ApiError enum → HTTP status + JSON body
└── modules/
    ├── game/
    │   ├── routes.rs     ← route handlers (thin: validate → call service → return)
    │   ├── service.rs    ← GameService: start/state/end + rail ticker spawning
    │   └── dto.rs        ← request/response types with Schemars derives
    ├── admin/
    │   ├── auth.rs       ← JWT sign/verify, AdminUser Axum extractor
    │   ├── routes.rs
    │   └── service.rs    ← config patching, persists to DB
    ├── realtime/
    │   ├── hub.rs        ← BroadcastHub (tokio broadcast channel, outbound only)
    │   ├── ws_handler.rs ← /ws/bridge WebSocket handler
    │   └── bridge_sync.rs← wires inbound WS messages to game processing
    ├── screen/
    │   ├── routes.rs     ← debug registry endpoints
    │   ├── ws_handler.rs ← /ws/screen/{screen_id} handler
    │   └── auth.rs       ← screen JWT validation
    ├── scores/
    │   ├── routes.rs
    │   ├── service.rs    ← leaderboard insert/query with cap-at-10 logic
    │   └── dto.rs
    ├── health/
    │   └── routes.rs     ← GET /health → 200 OK
    └── mqtt.rs           ← dispatches InboundMessage variants to game events
```

---

## Route map

```
GET  /health                         → 200 OK  (liveness probe)

POST /api/v1/game/start              → start a game session
GET  /api/v1/game/state              → current game snapshot
POST /api/v1/game/end                → force-end the current game

GET  /api/v1/characters              → roster + live config per character

POST /api/v1/scores                  → push a score to the leaderboard
GET  /api/v1/scores                  → get top-10 leaderboard

PATCH /api/v1/admin/config           → hot-patch GameConfig (requires admin JWT)
GET  /api/v1/admin/token             → generate a short-lived admin token (dev only)

GET  /api/v1/screens                 → list registered screens (debug)
GET  /api/v1/screens/{id}            → screen connection status (debug)

GET  /ws/bridge                      → MQTT bridge WebSocket
GET  /ws/screen/{screen_id}          → per-screen WebSocket (JWT auth)

GET  /docs                           → Lucyd OpenAPI UI (auto-generated)
```

---

## AppState

`AppState` is an `Arc`-wrapped struct cloned into every request handler by Axum.

```
AppState
├── game_engine:    Arc<Mutex<GameEngine>>
│     └── owns all game-logic state
├── active_session: Arc<Mutex<Option<GameSession>>>
│     └── tracks the running session ID + metadata
├── rail_sessions:  Arc<Mutex<HashMap<RailId, JoinHandle>>>
│     └── cancellable ticker tasks per rail
├── screen_hub:     Arc<ScreenRegistry>         (from screen-hub crate)
│     └── maps ScreenId → mpsc::Sender<ScreenEnvelope>
├── broadcast_hub:  BroadcastHub
│     └── tokio::broadcast sender for /ws/bridge clients
└── db:             SqlitePool
      └── r/w connection pool (sqlx)
```

> **Lock ordering**: always lock `game_engine` before `active_session` if you need both.
> Never invert this order — it causes deadlocks.

---

## Module anatomy

Every module under `modules/` follows the same three-file pattern:

```
routes.rs   ← Axum handler functions only. No business logic.
              Extracts path/query/body, calls service, returns response.

service.rs  ← All business logic lives here. Takes &AppState or sub-parts.
              Returns domain types or ApiError.

dto.rs      ← Serde + Schemars structs for request bodies and responses.
              Kept separate so they can be inspected without reading logic.
```

**Why this separation?**

- `routes.rs` changes when the HTTP contract changes.
- `service.rs` changes when the business rule changes.
- `dto.rs` changes when the data shape changes.
Each file has a single reason to change → SOLID Single Responsibility.

---

## Error handling

All errors implement `IntoResponse` through `ApiError`:

```rust
pub enum ApiError {
    BadRequest(String),
    NotFound(String),
    Conflict(String),
    Unauthorized(String),
    Internal(String),
    Serialization(String),
}
```

Wire-format (always JSON):

```json
{
  "error": "NOT_FOUND",
  "message": "No active game session"
}
```

Mapping:

```
BadRequest    → 400
Unauthorized  → 401
NotFound      → 404
Conflict      → 409
Internal      → 500
Serialization → 500
```

**How to use it in a handler:**

```rust
pub async fn my_handler(State(state): State<AppState>) -> Result<Json<MyDto>, ApiError> {
    let result = state.db.fetch_one(...).await
        .map_err(|e| ApiError::Internal(e.to_string()))?;
    Ok(Json(result.into()))
}
```

Never `unwrap()` or `expect()` inside a handler. Every fallible operation must convert to `ApiError`.

---

## Authentication

Admin endpoints are protected by HS256 JWTs validated by the `AdminUser` extractor:

```
Request
  │  Authorization: Bearer <token>
  ▼
AdminUser::from_request_parts()
  │  decode + verify signature using SCREEN_JWT_SECRET
  ├─ valid   → handler receives AdminUser { claims }
  └─ invalid → 401 Unauthorized (short-circuit, handler never called)
```

Screen WebSocket connections use a separate per-screen JWT (same secret, different claims).

---

## How to add a new route

1. **Create or pick a module** under `modules/`. If it's a new domain, create a new folder with
   `routes.rs`, `service.rs`, `dto.rs`.

2. **Define the DTO** in `dto.rs`:

```rust
#[derive(Debug, Deserialize, JsonSchema)]
pub struct MyRequest {
    pub field: String,
}

#[derive(Debug, Serialize, JsonSchema)]
pub struct MyResponse {
    pub result: String,
}
```

3. **Write the service function** in `service.rs`:

```rust
pub async fn do_something(
    state: &AppState,
    req: MyRequest,
) -> Result<MyResponse, ApiError> {
    // business logic here
    Ok(MyResponse { result: req.field })
}
```

4. **Write the handler** in `routes.rs`:

```rust
pub async fn handle_something(
    State(state): State<AppState>,
    Json(body): Json<MyRequest>,
) -> Result<Json<MyResponse>, ApiError> {
    let res = service::do_something(&state, body).await?;
    Ok(Json(res))
}

pub fn router() -> Router<AppState> {
    Router::new().route("/something", post(handle_something))
}
```

5. **Register the router** in `router.rs`:

```rust
pub fn build_router(state: AppState) -> Router {
    Router::new()
        .merge(my_module::routes::router())  // ← add this line
        // ...existing routers
        .with_state(state)
}
```

6. **Write tests** (see [Testing guide](../testing.md)).

---

## Game session lifecycle

```
POST /api/v1/game/start
         │
         ▼
  GameService::start_game()
         │
         ├── lock game_engine
         ├── engine.start_game(character)  → GameSnapshot
         ├── lock active_session           → store SessionId + timestamp
         ├── spawn rail ticker tasks       → store JoinHandle in rail_sessions
         └── broadcast StartGame event to screens via screen_hub

  (game running)
         │ MQTT button press arrives via /ws/bridge
         ▼
  mqtt.rs::dispatch(InboundMessage::Button)
         │
         ├── lock game_engine
         ├── engine.process_button_press(button_id)  → Vec<ScreenEnvelope>
         └── send each envelope to screen_hub

POST /api/v1/game/end  (or natural game over from drain event)
         │
         ▼
  GameService::end_game()
         │
         ├── cancel rail ticker tasks
         ├── lock game_engine → engine.end_game() → final score
         ├── lock active_session → clear
         ├── persist score to DB (if score > 0)
         └── broadcast GameOver event to screens
```
