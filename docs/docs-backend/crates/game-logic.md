# Crate: game-logic

## Role

`game-logic` is a **pure library** no HTTP, no async I/O, no database. It is the heart of the
project: it owns every rule about how the game works.

The `api` crate calls into this library and feeds it events (button presses, ball hits, etc).
The library computes the new state and returns `ScreenEnvelope` events to broadcast.

Keeping game logic completely isolated from infrastructure means:
- It can be unit-tested without spinning up a server.
- Rules can change without touching HTTP or MQTT code.
- The engine can theoretically be swapped or embedded in a different binary.

---

## Source tree

```
crates/game-logic/src/
├── lib.rs                          ← public exports (GameEngine, GameSnapshot, traits)
├── engine/
│   ├── mod.rs                      ← re-exports GameEngine
│   ├── config.rs                   ← GameConfig: lazy static RwLock, hot-patching
│   ├── events.rs                   ← GameEvent enum (all input events)
│   ├── states.rs                   ← GameState, GamePhase, TiltState
│   └── core/
│       ├── mod.rs                  ← GameEngine struct definition + take_snapshot()
│       ├── input.rs                ← process_button_press(): main event dispatcher
│       ├── charge.rs               ← ultimate charge accumulation logic
│       ├── ulti.rs                 ← ultimate activation / cancellation
│       ├── process.rs              ← routes GameEvent variants to handlers
│       └── emit.rs                 ← builds ScreenEnvelope events for screens
├── engine/
│   ├── components/
│   │   └── health.rs               ← Shield + damage tracking
│   ├── services/
│   │   ├── charge.rs               ← charge weight calculation per input type
│   │   └── ulti.rs                 ← character-specific ult behavior dispatch
│   ├── scoring.rs                  ← pure math: fibonacci, multiplier application
│   └── pve/
│       ├── engine.rs               ← PveEngine struct
│       ├── states.rs               ← PvePhase enum
│       ├── difficulty.rs           ← difficulty scaling with score
│       ├── events.rs               ← PVE-specific events
│       └── ennemy/
│           ├── mod.rs
│           ├── boss.rs             ← Boss struct (name, max_hp, current_hp)
│           └── kind.rs             ← BossKind enum (GLaDOS, Wheatley, …)
├── combo/
│   ├── mod.rs                      ← ComboDetector, ComboEffect exports
│   ├── detector.rs                 ← combo table + sequence matching logic
│   ├── model.rs                    ← ComboResult struct
│   ├── multiplier.rs               ← time-limited score multiplier (activated by ults)
│   ├── streak.rs                   ← streak tier system (hit count → multiplier tier)
│   └── error.rs
└── player/
    └── personnages/
        ├── mod.rs
        ├── character.rs            ← Character trait + 4 implementations
        └── character_stats.rs      ← CharacterStats: charge thresholds, ult shapes
```

---

## Game state machine

```
                      ┌─────────┐
                      │  Idle   │ ← server starts here
                      └────┬────┘
                           │ start_game(character)
                           ▼
                      ┌─────────┐
                      │ InGame  │ ◄───────────────────┐
                      └────┬────┘                     │
                           │                          │
           ┌───────────────┼───────────────┐          │
           │               │               │          │
       Drain #1         Drain #2        Drain #3      │
       (ball lost)      (ball lost)     (game over)   │
           │               │               │          │
           ▼               ▼               ▼         │
       respawn         respawn        ┌──────────┐    │
      (stay InGame)   (stay InGame)   │ GameOver │    │
                                      └──────────┘    │
                                           │          │
                              end_game() or POST /end │
                                           └──────────┘
                                       (returns to Idle)
```

---

## GameEngine struct

```
GameEngine
├── state: GameState           ← mutable game data
│   ├── phase: GamePhase       ← Idle | InGame | GameOver
│   ├── score: u64
│   ├── lives: u8              ← starts at 3
│   ├── ulti_charge: f32       ← 0.0 → 1.0
│   ├── tilt_state: TiltState  ← count of tilts, lock flag
│   ├── shield_hp: u8
│   └── multiball_active: bool
├── character: Box<dyn Character>
├── combo: ComboDetector
├── multiplier: MultiplierState ← time-limited ult override multiplier
├── streak: StreakState         ← hit-count based tier
└── pve: PveEngine
```

**Taking a snapshot** (for the `/api/v1/game/state` endpoint):

```rust
let snapshot: GameSnapshot = engine.take_snapshot();
```

`take_snapshot()` produces an immutable clone of the current state — it never fails and never
locks anything beyond the engine mutex that the caller already holds.

---

## Scoring formula

```
effective_multiplier = if ulti_override.is_some() {
    ulti_override              ← ult multiplier takes full precedence
} else {
    streak_multiplier × combo_multiplier
}

final_score_delta = base_points × effective_multiplier
```

Base points per event type:

| Event        | Base points | Notes                          |
|--------------|-------------|--------------------------------|
| Bumper hit   | 100         |                                |
| Rail tick    | fibonacci   | F(0)=1, F(1)=1, F(n)=F(n-2)+F(n-1), grows with sustained hits |
| Slingshot    | 50          |                                |
| Spinner      | 30          | per revolution                 |
| Target hit   | 200         |                                |

Streak tiers:

```
 0 –  4 hits  → ×1.0   (no streak)
 5 –  9 hits  → ×1.5
10 – 19 hits  → ×2.5
20 – 39 hits  → ×3.5
   40+ hits   → ×4.5
```

A drain resets the streak to 0.

---

## Character system

### The `Character` trait

```rust
pub trait Character: Send + Sync {
    fn slug(&self) -> &'static str;        // "KEENU", "VIPER", …
    fn stats(&self) -> &CharacterStats;    // charge thresholds, ult params
    fn ult_shape(&self) -> UltShape;       // Instant | Sustained | Inherited
    fn ulti_id(&self) -> &'static str;    // "multiball_split", "rampage", …
}
```

### The four characters

```
┌─────────────────────────────────────────────────────────────────────┐
│ Slug    │ Ult name        │ Shape     │ Description                 │
├─────────┼─────────────────┼───────────┼─────────────────────────────┤
│ KEENU   │ multiball_split │ Instant   │ Splits ball, brief chaos    │
│ VIPER   │ rampage         │ Sustained │ Score multiplier override   │
│ GHOST   │ mimic           │ Inherited │ Cycles through other shapes │
│ ORACLE  │ time_slow       │ Sustained │ Time-based passive charge   │
└─────────┴─────────────────┴───────────┴─────────────────────────────┘
```

### UltShape variants

```
Instant   → triggers once, effect resolves immediately
Sustained → activates, persists for a duration, can be cancelled
Inherited → (Ghost only) copies the ult shape of another character at runtime
```

### How to add a new character

1. Create `crates/game-logic/src/player/personnages/my_character.rs`.

2. Define the struct and implement the `Character` trait:

```rust
use crate::player::personnages::{Character, CharacterStats};

pub struct MyCharacter;

impl Character for MyCharacter {
    fn slug(&self) -> &'static str { "MY_CHAR" }

    fn stats(&self) -> &CharacterStats {
        static STATS: std::sync::OnceLock<CharacterStats> = std::sync::OnceLock::new();
        STATS.get_or_init(|| CharacterStats {
            charge_per_hit: 0.05,
            ult_duration_secs: Some(8.0),
            // …
        })
    }

    fn ult_shape(&self) -> UltShape { UltShape::Instant }

    fn ulti_id(&self) -> &'static str { "my_ult_name" }
}
```

3. Export it in `player/personnages/mod.rs`:

```rust
pub mod my_character;
pub use my_character::MyCharacter;
```

4. Wire it in the API character factory (in `api/src/modules/game/service.rs`) where character
   slugs are resolved to `Box<dyn Character>`.

5. Add unit tests in the same file (see [Testing guide](../testing.md)).

---

## Ultimate logic flow

```
process_button_press(ButtonId::Start)   ← "Start" is the ult button
         │
         ▼
charge.rs: is ulti_charge >= 1.0?
         │
    NO ──┘   → nothing happens, charge already displayed
         │
    YES ──► ulti.rs::activate_ult(character, state)
                    │
                    ├── Instant   → apply effect immediately, reset charge
                    ├── Sustained → set ulti_active = true, spawn timeout task
                    │              (task cancels ult after duration)
                    └── Inherited → resolve current shape, apply accordingly
```

Charge accumulates on every ball hit, weighted by `services/charge.rs`:

```
charge_delta = base_charge_weight × character.stats().charge_per_hit
```

---

## PVE system

```
PvePhase state machine:

  ┌──────────────────┐
  │ WaitingForScore  │ ← boss not yet spawned
  └────────┬─────────┘
           │ score threshold crossed
           ▼
  ┌──────────────────┐
  │    Fighting      │ ← boss spawned, each hit reduces boss HP
  └────────┬─────────┘
           │ boss HP reaches 0
           ▼
  ┌──────────────────┐
  │    Cooldown      │ ← victory animation, timer
  └────────┬─────────┘
           │ timer expires → spawn next (harder) boss
           └──────────────────────────────────────────►  back to Fighting
```

Difficulty scaling: each boss has `max_hp` set by `difficulty.rs` based on how many bosses have
already been defeated in the current session. More defeats = more HP.

### BossKind enum

```
GLaDOS    ← first boss
Wheatley  ← second boss
…
```

To add a new boss: add a variant to `BossKind` in `pve/ennemy/kind.rs` and add an entry to the
difficulty table in `pve/difficulty.rs`.

---

## Tilt system

```
TiltState {
    count: u8,     ← increments on each tilt event
    locked: bool,  ← set when count reaches 3
}
```

When `locked = true`:
- The score is frozen (all score deltas are silently ignored).
- A `TiltLocked` event is emitted to screens.
- The lock clears when the ball drains (new ball = clean slate).

This models the real pinball anti-cheat: shake the machine three times and your score stops counting.

---

## Combo system

`ComboDetector` maintains a recent button-press window and matches it against a static combo table.

```
Button presses:   L1  R1  L1  R1
                   ↑
             within combo_window_ms
                         ↓
                  combo "PING-PONG" matched
                         ↓
                  ComboResult { multiplier: 2.0, name: "ping_pong" }
```

The multiplier from combos compounds with the streak multiplier (unless a ult override is active).

---

## GameConfig hot-patching

`GameConfig` is a `lazy_static! { static ref CONFIG: RwLock<GameConfig> }`.

The admin API can patch it at runtime:

```
PATCH /api/v1/admin/config
  { "bumper_base_points": 150, "streak_tier_2_threshold": 8 }
         │
         ▼
admin/service.rs: acquire CONFIG.write(), merge patch, persist to DB
         │
         ▼
All subsequent engine calls read the new values (CONFIG.read())
```

No server restart needed. On startup, the engine reads persisted config from the DB to restore
the last patch.
