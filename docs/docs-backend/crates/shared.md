# Crate: shared

## Role

`shared` is the **common vocabulary** of the project. It contains only types: enums, structs, and
their serialization. No logic, no I/O, no async.

Every other crate imports it. Nothing in `shared` imports any internal crate.

```
shared
  ▲   ▲     ▲     ▲
  │   │      │     │
 api  game  mqtt  screen
           bridge  hub
```

Keeping types here avoids circular dependencies and ensures that the wire format between crates
has a single source of truth.

---

## Source tree

```
crates/shared/src/
├── lib.rs          ← module exports
├── dto.rs          ← MQTT topic parsing (Topic, Subtopic)
├── events.rs       ← WsMessage, InboundMessage, OutboundMessage
├── model.rs        ← ButtonId, HitType, GamePhase, EventKind, CommandKind
└── screen.rs       ← ScreenId, ScreenEventType, ScreenEnvelope
```

---

## dto.rs MQTT topic format

MQTT topics follow the pattern `pinball/<device_id>/<subtopic>`.

```rust
pub struct Topic {
    pub device_id: String,
    pub subtopic:  Subtopic,
}
```

`Topic::parse("pinball/esp32-01/input/button")` returns `Some(Topic { device_id: "esp32-01", subtopic: Subtopic::InputButton })`.

### Subtopic variants

```
InputButton    → player pressed a physical button
InputPlunger   → plunger was pulled/released
InputGyro      → accelerometer reading (tilt detection)
BallHit        → ball hit a bumper / rail / slingshot / …
GameState      → device reporting its own state
Telemetry      → sensor diagnostics (temperature, uptime…)
Events         → arbitrary device events
Cmd            → command the device should execute
Status         → device online/offline heartbeat
```

---

## events.rs WebSocket message envelope

All messages flowing over `/ws/bridge` are JSON-serialized `WsMessage`:

```rust
pub enum WsMessage {
    Inbound  { payload: InboundMessage  },
    Outbound { payload: OutboundMessage },
}
```

The `dir` tag on the wire:

```json
{ "dir": "inbound",  "payload": { ... } }
{ "dir": "outbound", "payload": { ... } }
```

### InboundMessage (ESP32 → API)

```
Button     { device_id, button_id: ButtonId, pressed: bool }
Plunger    { device_id, force: f32 }
Gyro       { device_id, x: f32, y: f32, z: f32 }
Telemetry  { device_id, uptime_secs: u64, temp_celsius: f32 }
Event      { device_id, kind: EventKind }
Status     { device_id, online: bool }
```

### OutboundMessage (API → ESP32)

```
BallHit   { device_id, hit_type: HitType }
GameState { device_id, phase: GamePhase }
Command   { device_id, command: CommandKind }
```

---

## model.rs primitive game types

### ButtonId

```
L1   R1   L2   R2         ← left/right flipper buttons (top and bottom)
Start                      ← activates ultimate
UnderPlunger               ← plunger sensor
Top   Middle   Bottom      ← three target bank buttons
```

### HitType

```
Bumper      ← round pop bumper
Rail        ← ball on a timed rail/ramp
Slingshot   ← triangular slingshot
Drain       ← ball lost to drain hole
Target      ← stationary target
Spinner     ← spinning target
```

### GamePhase

```
Idle        ← no game running
Attract     ← demo mode (no game, attract animation)
Start       ← game just started, animation playing
Playing     ← normal play
BallLost    ← drain animation
Bonus       ← end-of-ball bonus calculation
Tilt        ← tilt penalty sequence
GameOver    ← game ended, score display
HighScore   ← new high score entry
```

### EventKind / CommandKind

`EventKind` covers arbitrary device events (e.g. `BallDetected`, `BallMissing`).
`CommandKind` covers commands sent to devices (e.g. `Rumble`, `LedPattern`, `Reset`).

---

## screen.rs screen routing types

### ScreenId

```
FrontScreen   ← player-facing LCD
BackScreen    ← back panel (leaderboard, boss health)
DmdScreen     ← dot-matrix display (score, small animations)
GameEngine    ← virtual: events emitted internally by the engine
```

### ScreenEnvelope

Every message sent through `screen-hub` is wrapped in an envelope:

```rust
pub struct ScreenEnvelope {
    pub from:       ScreenId,
    pub to:         ScreenId,
    pub event_type: ScreenEventType,
    pub payload:    serde_json::Value,
}
```

The `payload` field is a freeform JSON value each event type defines its own shape.
This allows adding new payload fields without a breaking change to the envelope structure.

### ScreenEventType 50+ variants (selection)

```
StartGame            ← new game started
GameOver             ← session ended
ScoreUpdate          ← score changed (with delta)
LivesUpdate          ← life count changed
UltimateCharged      ← charge reached 100%
UltimateTriggered    ← ult activated
UltimateCancelled    ← sustained ult cancelled
ComboDetected        ← combo sequence matched
StreakUpdate         ← streak tier changed
BossSpawned          ← new PVE boss appeared
BossHit              ← boss took damage
BossDefeated         ← boss HP reached 0
TiltWarning          ← tilt count increased
TiltLocked           ← third tilt, score frozen
ShieldHit            ← shield absorbed damage
ShieldBroken         ← shield destroyed
MultiballStart       ← multiball mode activated
LeaderboardUpdate    ← new score on leaderboard
```

---

## Adding new shared types

The rule: **if two or more crates need the same type, it belongs in `shared`**.

Steps:

1. Add the type to the appropriate file in `crates/shared/src/`.
2. Derive `Serialize, Deserialize` (required for wire transmission).
3. Add `JsonSchema` if it appears in API request/response bodies.
4. Re-export it from `lib.rs` if it needs to be public:

```rust
pub use screen::{ScreenId, ScreenEventType, ScreenEnvelope};
```

5. Run `cargo check --workspace` to verify nothing broke.

**Do not** put types in `shared` that only one crate uses. Keep those local.
