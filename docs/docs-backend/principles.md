# DRY & SOLID Principles

These aren't abstract rules imposed from outside. They are solutions to real problems that appear
when a codebase grows. This document explains each principle, shows where it already applies in
this project, and explains what goes wrong when it's violated.

---

## DRY — Don't Repeat Yourself

> Every piece of knowledge must have a single, unambiguous, authoritative representation in a system.

### What it means

If the same logic appears in two places, they will eventually diverge. One gets fixed, one doesn't.
Both callers believe they're correct. You get silent bugs.

### Where it applies here

**Shared types in the `shared` crate**

`ButtonId`, `GamePhase`, `ScreenEventType` are defined once and imported everywhere. If they were
copy-pasted into each crate, a new variant added to `api` would be missing from `mqtt-bridge` and
cause a silent deserialization failure.

```
WRONG: each crate defines its own ButtonId
  api/src/model.rs      → pub enum ButtonId { L1, R1, L2, R2, Start }
  game-logic/src/...    → pub enum ButtonId { L1, R1, L2, R2 }   ← Start missing!
  mqtt-bridge/src/...   → pub enum ButtonId { L1, R1 }            ← even more missing!

RIGHT: shared/src/model.rs → pub enum ButtonId { L1, R1, L2, R2, Start, UnderPlunger, ... }
  All crates import the same definition. Change it once, the compiler enforces it everywhere.
```

**`GameConfig` with a single `RwLock`**

All game parameters (base points, streak thresholds, charge rates) live in one struct. The admin
API patches it in one place. Every engine function reads from that single source.

If each tuning parameter lived in a different place, patching the admin API would require touching
five different files, and you'd always forget one.

**`emit.rs` for building `ScreenEnvelope`s**

The pattern "build an envelope, set from/to, set event type, serialize payload" is used dozens of
times. Centralizing it in `emit.rs` means if the `ScreenEnvelope` struct gains a new required field,
there's one file to update, not thirty.

### How to spot a DRY violation

- You copy-paste a block of code and think "I'll just change the one variable".
- You search for a bug and find the same logic in three files, only one of which has the fix.
- A type is defined in one crate and duplicated with a different name in another.

### How to fix it

Extract the repeated logic into a function, method, or shared module. Apply it immediately — do
not leave duplicates and plan to "clean it up later".

---

## SOLID

### S — Single Responsibility Principle

> A module, class, or function should have one reason to change.

Each file in this project changes for exactly one reason:

```
routes.rs    → changes when the HTTP contract changes (new path, new status code)
service.rs   → changes when the business rule changes (new game logic, new DB query)
dto.rs       → changes when the data shape changes (new field in request/response)
```

If `routes.rs` contained business logic, it would have two reasons to change: the HTTP contract
AND the business rule. That makes it harder to test (you can't test the logic without an HTTP
server) and harder to review (a PR diff mixes routing changes with logic changes).

**Violation to avoid:**

```rust
// BAD: handler contains business logic
async fn start_game(State(state): State<AppState>, Json(body): Json<StartRequest>) -> Response {
    let mut engine = state.game_engine.lock().await;
    if engine.phase() != GamePhase::Idle {
        return (StatusCode::CONFLICT, "Game already running").into_response();
    }
    let character: Box<dyn Character> = match body.character_slug.as_str() {
        "KEENU" => Box::new(Enforcer),
        "VIPER" => Box::new(Viper),
        // 20 more lines of routing logic…
        _ => return (StatusCode::BAD_REQUEST, "Unknown character").into_response(),
    };
    engine.start_game(character);
    // …
}

// GOOD: handler delegates entirely
async fn start_game(State(state): State<AppState>, Json(body): Json<StartRequest>) -> Result<Json<GameSnapshot>, ApiError> {
    let snapshot = game_service::start_game(&state, body.character_slug).await?;
    Ok(Json(snapshot))
}
```

---

### O — Open/Closed Principle

> A module should be open for extension but closed for modification.

Adding a new character should not require modifying the game engine's core logic.

**How it works here:**

```
Character trait (closed — the interface is stable)
         ▲
         │ implement
    ┌────┼──────┬──────┬──────┐
  Enforcer  Viper  Ghost  Oracle   ← open: add new characters without touching core
```

`GameEngine` calls `self.character.ult_shape()`, `self.character.stats()`, etc. Adding `MyCharacter`
that implements `Character` requires zero changes to `GameEngine`. The engine is closed to
modification, but open to extension via the trait.

**Same pattern in `ScreenRouter`:**

Interceptors let you add new message-processing behaviour (logging, filtering, mutation) without
modifying the router itself.

**Violation to avoid:**

```rust
// BAD: adding a new character requires editing the engine
fn activate_ult(&mut self) {
    match self.character_slug {
        "KEENU" => self.split_multiball(),
        "VIPER" => self.activate_rampage(),
        "MY_NEW_CHAR" => self.do_new_thing(),  // ← must edit this file every time
    }
}

// GOOD: the trait handles dispatch
fn activate_ult(&mut self) {
    self.character.activate_ult(self);   // ← no change needed for new characters
}
```

---

### L — Liskov Substitution Principle

> Any implementation of a trait must be substitutable for the trait itself.

Every `Box<dyn Character>` must work correctly wherever a `Character` is expected. That means:

- `slug()` must always return a non-empty `&'static str`.
- `stats()` must always return a valid `&CharacterStats` (no panics, no None).
- `ult_shape()` must return a shape that the engine can handle without special-casing.

A character that returns `CharacterStats { charge_per_hit: -1.0, … }` violates LSP because it
breaks the engine's assumption that charge only ever increases. The engine cannot safely substitute
this character for any other.

**Practical rule:** if you find yourself writing `if character.slug() == "GHOST" { ... }` inside
the engine, you have an LSP violation. The special case belongs in `Ghost`'s `Character` impl.

---

### I — Interface Segregation Principle

> Don't force a type to implement methods it doesn't need.

The `Character` trait only exposes what the engine actually needs: `slug`, `stats`, `ult_shape`,
`ulti_id`. It does not expose `display_name`, `lore_description`, or any UI concern.

If those UI fields were added to the `Character` trait, every `Character` implementor would have
to provide them — even the engine, which never uses them. The UI data belongs in a separate type
(a `CharacterInfoDto` in the `api` crate) that the game-logic crate never imports.

**Violation to avoid:**

```rust
// BAD: trait has methods only some callers need
pub trait Character {
    fn slug(&self) -> &'static str;
    fn stats(&self) -> &CharacterStats;
    fn ult_shape(&self) -> UltShape;
    fn display_name(&self) -> &str;      // ← only needed by the UI, not by the engine
    fn lore_text(&self) -> &str;         // ← same
    fn icon_url(&self) -> &str;          // ← same
}
```

---

### D — Dependency Inversion Principle

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

`GameEngine` (high-level) does not depend on `Enforcer` or `Viper` (low-level concrete types).
It depends on `Box<dyn Character>` (the abstraction).

```
GameEngine ──depends on──► Character (trait / abstraction)
                                  ▲
                        ┌─────────┼──────────┐
                     Enforcer   Viper   Ghost   Oracle
                     (concrete implementations)
```

The `api` crate creates the concrete character (`Box::new(Enforcer)`) and injects it into the
engine at game start. The engine never imports `Enforcer` directly.

This means:
- `game-logic` does not need to know about `api`.
- You can test the engine with a mock character (`struct TestCharacter`).
- Swapping character implementations at runtime is trivial.

**Violation to avoid:**

```rust
// BAD: engine depends on concrete type
use crate::player::personnages::enforcer::Enforcer;

impl GameEngine {
    pub fn start_with_enforcer(&mut self) {
        self.character = Box::new(Enforcer);  // ← hardcoded, not injectable
    }
}
```

---

## Summary table

| Principle | Applied in this project | Benefit |
|-----------|------------------------|---------|
| **DRY** | `shared` crate, `GameConfig`, `emit.rs` | One place to change, no divergence |
| **SRP** | routes/service/dto split | Small files, single reason to change |
| **OCP** | `Character` trait, `ScreenRouter` interceptors | Add features without modifying core |
| **LSP** | All `Character` impls are valid substitutes | Engine works with any character |
| **ISP** | `Character` trait only has engine-needed methods | No forced stubs, no dead interface surface |
| **DIP** | `GameEngine` depends on `dyn Character`, not concrete types | Testable, swappable, decoupled |

---

## When rules conflict

These principles are guides, not laws. Occasionally they conflict:

- Extracting a function for DRY can create an abstraction that violates ISP (the function does too
  much for each caller).
- Following OCP strictly leads to deep trait hierarchies that are hard to follow.

When in doubt: **optimize for readability**. A slightly duplicated piece of code that's easy to
understand beats an over-abstracted one that requires five files to follow.
