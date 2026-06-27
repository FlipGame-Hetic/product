# Testing Guide

## Philosophy

Tests in this project serve two purposes:

1. **Correctness** — prove that a unit of code produces the right output for a given input.
2. **Safety net** — catch regressions when refactoring or adding features.

A test that can pass with a wrong implementation is worse than no test. Focus on observable
behaviour, not on internal implementation details.

---

## Test types and where they live

```
crates/
├── api/
│   ├── src/
│   │   ├── config.rs          ← unit tests (inline, #[cfg(test)] module)
│   │   └── modules/
│   │       └── *.rs           ← unit tests inline in each file
│   └── tests/
│       └── game_integration.rs ← integration test (uses a real DB + server)
├── game-logic/
│   └── src/
│       └── **/*.rs            ← unit tests inline, ~173 tests total
├── shared/
│   └── src/
│       └── **/*.rs            ← serialization roundtrip tests
└── screen-hub/
    └── src/
        └── **/*.rs            ← registry unit tests
```

### Rule of thumb

| What you're testing | Where to put it |
|---|---|
| A pure function or method | Inline `#[cfg(test)]` module in the same `.rs` file |
| A module that uses channels or `Arc<Mutex<>>` | Inline `#[cfg(test)]` module |
| A full HTTP request/response cycle | `crates/api/tests/` integration test |
| JSON serialization / deserialization | Inline in the type's file |

---

## Running tests

```bash
# Run every test in every crate
cargo test --workspace

# Run tests for one crate only
cargo test -p game-logic

# Run a specific test by name (substring match)
cargo test scoring::fibonacci

# Run with output visible (useful for debugging)
cargo test -- --nocapture

# Generate a coverage report (requires cargo-llvm-cov)
cargo llvm-cov --html
open target/llvm-cov/html/index.html
```

Current coverage target: **≥ 70%**.

---

## Why unit tests are inline, not in `tests/`

Rust offers two ways to organise tests: a separate `tests/` folder, or a `#[cfg(test)]` block directly in the source file. This project uses the second approach for unit tests, for two concrete reasons.

**1. Access to private code.**
In Rust everything is private by default. A function is only visible inside the file where it is written unless explicitly marked `pub`. A `tests/` folder is treated by the compiler as an *external* crate — it can only reach what is declared public. That means every private helper would have to be made public just to be testable, which weakens the module's encapsulation for no good reason. A `#[cfg(test)]` module placed inside the same file is still *inside* the module, so it inherits access to all private items without any exposure to the outside world.

**2. Compilation speed.**
Each file inside `tests/` is compiled as an independent binary. Ten test files mean ten separate link steps every time `cargo test` runs. Inline `#[cfg(test)]` blocks are compiled once together with the rest of the crate meaningfully faster as the project grows.

The `tests/` folder is not abandoned: it is reserved for **integration tests** that simulate a real HTTP request hitting a real server (see [Writing an integration test](#writing-an-integration-test)). That is exactly the use-case it was designed for.

---

## Writing a unit test

Unit tests live at the bottom of the file they test, inside a `#[cfg(test)]` module.
This is idiomatic Rust the test module has access to private items of the parent module.

```rust
// crates/game-logic/src/engine/scoring.rs

pub fn fibonacci_score(tick: u32) -> u64 {
    match tick {
        0 | 1 => 1,
        n => fibonacci_score(n - 1) + fibonacci_score(n - 2),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fibonacci_base_cases() {
        assert_eq!(fibonacci_score(0), 1);
        assert_eq!(fibonacci_score(1), 1);
    }

    #[test]
    fn fibonacci_progression() {
        assert_eq!(fibonacci_score(5), 8);
        assert_eq!(fibonacci_score(10), 89);
    }
}
```

### Naming convention

```
<function_or_method>_<scenario>_<expected_outcome>

Examples:
  start_game_with_valid_character_returns_snapshot
  process_button_press_when_charge_full_activates_ult
  register_screen_twice_returns_conflict_error
  fibonacci_score_at_tick_5_returns_8
```

The name is the only documentation a test needs. If the name is not self-explanatory, the test
is probably testing the wrong thing.

---

## Writing an async unit test

Many types (channels, `Arc<Mutex<>>`) need a Tokio runtime even in unit tests. Use `#[tokio::test]`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::sync::mpsc;

    #[tokio::test]
    async fn registry_send_delivers_to_receiver() {
        let registry = ScreenRegistry::new();
        let handle = registry.register(ScreenId::FrontScreen).await.unwrap();

        registry.send(ScreenId::FrontScreen, make_test_envelope()).await.unwrap();

        let received = handle.receiver.try_recv().unwrap();
        assert_eq!(received.event_type, ScreenEventType::ScoreUpdate);
    }
}
```

---

## Writing an integration test

Integration tests live in `crates/api/tests/`. They start a real Axum server against an in-memory
SQLite database and send actual HTTP requests.

```rust
// crates/api/tests/game_integration.rs

use axum_test::TestServer;
use api::build_app;

async fn make_test_server() -> TestServer {
    let pool = sqlx::SqlitePool::connect(":memory:").await.unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();
    let app = build_app(pool).await;
    TestServer::new(app).unwrap()
}

#[tokio::test]
async fn start_game_returns_snapshot() {
    let server = make_test_server().await;

    let response = server
        .post("/api/v1/game/start")
        .json(&serde_json::json!({ "character_slug": "KEENU" }))
        .await;

    response.assert_status_ok();
    let body: serde_json::Value = response.json();
    assert_eq!(body["phase"], "InGame");
}
```

---

## Writing serialization tests

Every type that crosses a wire boundary (WsMessage, ScreenEnvelope, DTOs) must have a roundtrip
test:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ws_message_button_roundtrip() {
        let original = WsMessage::Inbound {
            payload: InboundMessage::Button {
                device_id: "esp32-01".into(),
                button_id: ButtonId::L1,
                pressed: true,
            },
        };

        let json = serde_json::to_string(&original).unwrap();
        let decoded: WsMessage = serde_json::from_str(&json).unwrap();

        assert_eq!(original, decoded);
    }
}
```

This catches typos in `#[serde(rename = "...")]` attributes and missing `Deserialize` derives
that wouldn't be caught by the type system alone.

---

## What to test — checklist

When you add a new function or module, ask yourself:

```
□ Happy path  → does it return the right value for valid input?
□ Edge cases  → zero, empty, max values, boundary conditions?
□ Error path  → does it return the right error for invalid input?
□ Roundtrip   → (for serializable types) does JSON encode/decode preserve the value?
□ Concurrency → (for Mutex/channel types) does it behave correctly under concurrent access?
```

You do **not** need to test:
- Third-party library behaviour (trust that `serde_json::to_string` works).
- Trivial getters that just return a field.
- Private helpers that are fully covered by tests of the public API that calls them.

---

## What NOT to do

```rust
// BAD: tests internal state, not observable behaviour
#[test]
fn charge_internal_counter_increments() {
    let mut state = GameState::default();
    state.ulti_charge += 0.1;      // ← reached into internal field
    assert_eq!(state.ulti_charge, 0.1);
}

// GOOD: tests via the public API
#[test]
fn bumper_hit_increases_charge() {
    let mut engine = GameEngine::new(Box::new(Enforcer));
    engine.start_game();
    engine.process_event(GameEvent::BallHit { hit_type: HitType::Bumper });
    let snap = engine.take_snapshot();
    assert!(snap.ulti_charge > 0.0);
}
```

```rust
// BAD: panic on failure loses the error message
fn get_game_state() -> GameState {
    state.lock().expect("lock poisoned")   // panic is not a test failure
}

// GOOD: return a Result, let the test harness report it
fn start_game_returns_error_when_already_running() {
    let result = service.start_game("KEENU");
    assert!(matches!(result, Err(ApiError::Conflict(_))));
}
```

---

## CI enforcement

The CI pipeline (`ci.yml`) runs:

```
cargo fmt --all -- --check      ← formatting (fails PR if code is not formatted)
cargo clippy --workspace        ← lints (warnings treated as errors)
cargo build --workspace         ← compilation
cargo test --workspace          ← all tests
```

A PR cannot be merged if any of these fail. Run them locally before pushing:

```bash
cargo fmt --all
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```
