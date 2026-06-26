# Conventions

Conventions exist to reduce cognitive load. When everyone follows the same patterns, you spend
less time decoding structure and more time solving the actual problem.

---

## Branch naming

```
<type>/<short-description-in-kebab-case>

Types:
  feat/     → new feature
  fix/      → bug fix
  refacto/  → internal restructure, no new feature
  docs/     → documentation only
  test/     → adding or fixing tests
  chore/    → tooling, CI, dependency update
  hotfix/   → urgent production fix (branched from main directly)

Examples:
  feat/add-ghost-character
  fix/screen-guard-memory-leak
  refacto/split-game-service-module
  docs/add-mqtt-bridge-doc
  test/cover-scoring-edge-cases
  chore/update-axum-to-0-8-9
```

**Rules:**
- Always branch from `main` (never from another feature branch, unless explicitly coordinating).
- Keep the description lowercase, words separated by hyphens.
- No personal names in branch names (`feat/kae-screen-fix` → `dev/screen-no-content-on-missing`).

---

## Commit messages

```
<type>(<scope>): <short imperative description>

[Optional body: explains WHY, not WHAT]
```

The description is written in the **imperative mood**: "add","HOTFIX", "fix", "remove", not "added",
"fixes", "removed".

```
Examples:
  feat(game-logic): add time_slow ultimate for Oracle character
  fix(api): return 409 when game session already active
  refacto(screen-hub): replace Mutex with RwLock for read-heavy registry
  docs(shared): document WsMessage discriminated union format
  test(game-logic): cover fibonacci scoring edge cases
  chore: update sqlx to 0.8.3

With body (for non-obvious changes):
  fix(mqtt-bridge): cap inbound channel at 256 to prevent OOM

  During load testing, the channel was unbounded and grew without limit
  when the WebSocket write loop was slower than MQTT ingest.
  A bound of 256 adds back-pressure without dropping messages in normal use.
```

**Rules:**
- First line ≤ 72 characters.
- No period at the end of the first line.
- Body separated from the first line by a blank line.
- Body explains the motivation and context, not the code change (the diff already shows the code).

---

## Pull request process

```
1. Branch from main
       │
       ▼
2. Implement the change
       │
       ▼
3. Run locally:
   cargo fmt --all
   cargo clippy --workspace --all-targets -- -D warnings
   cargo test --workspace
       │
       ▼
4. Open PR against main
       │
       ▼
5. CI runs automatically (fmt → clippy → build → test)
       │
   CI fails? → fix and push, CI reruns
       │
       ▼
6. At least one reviewer approves
       │
       ▼
7. Squash and merge into main
```

**PR title** follows the same convention as commit messages.

**PR description** must include:
- What changed and why (not just a list of files).
- How to test it manually if it involves new behaviour.
- Screenshots or request/response examples for API changes.

---

## Code style

### Formatting

Use `rustfmt` with the project defaults. Never manually format code — run `cargo fmt --all` before
committing. CI enforces this.

### Naming

| What | Convention | Example |
|---|---|---|
| Types, traits, enums, variants | `PascalCase` | `GameEngine`, `ButtonId`, `InGame` |
| Functions, methods, variables | `snake_case` | `start_game`, `ulti_charge`, `screen_id` |
| Constants, statics | `SCREAMING_SNAKE_CASE` | `MAX_LIVES`, `DEFAULT_PORT` |
| Module files | `snake_case` | `game_service.rs`, `ws_handler.rs` |
| Crate names | `kebab-case` in `Cargo.toml`, `snake_case` in `use` | `game-logic` / `game_logic` |

### Comments

Write comments only when the **why** is non-obvious. The code already shows the **what**.

```rust
// BAD: describes what the code does (already obvious)
// Lock the engine mutex
let engine = state.game_engine.lock().await;

// GOOD: explains a non-obvious constraint
// Always lock game_engine before active_session — reversing the order causes deadlock.
let engine = state.game_engine.lock().await;
let session = state.active_session.lock().await;
```

No multi-line docstring blocks on private functions. Reserve `///` doc comments for public API
items in the `shared` and `screen-hub` crates.

### Error handling

- Use `?` to propagate errors. Never `unwrap()` or `expect()` in production code paths.
- `unwrap()` is acceptable only in tests and in `main()` during startup (where a failure should
  abort the process).
- Convert infrastructure errors to domain errors at the boundary:

```rust
// At the DB boundary in service.rs:
sqlx::query!(...).fetch_one(&pool).await
    .map_err(|e| ApiError::Internal(format!("DB query failed: {e}")))?;

// NOT propagated raw to the HTTP layer — caller gets a clean ApiError
```

### Async

- Default to `async fn` for functions that do I/O.
- Do not `block_in_place` or `spawn_blocking` unless you have a CPU-bound operation or a
  non-async third-party call that you cannot avoid.
- Do not hold a `MutexGuard` across an `.await` point — this blocks the executor thread.

```rust
// BAD: holds lock across await
let guard = state.game_engine.lock().await;
sqlx::query!(...).await?;   // ← guard held here, blocks other tasks
drop(guard);

// GOOD: do all work inside the lock, release before awaiting
let snapshot = {
    let mut engine = state.game_engine.lock().await;
    engine.process(event)
};   // ← lock released here
sqlx::query!(...).await?;   // ← now safe to await
```

---

## Module structure rules

Every new domain area under `modules/` follows this layout:

```
modules/
└── my_domain/
    ├── mod.rs        ← re-exports only: pub use routes::router; pub use service::*;
    ├── routes.rs     ← Axum handlers, no logic
    ├── service.rs    ← business logic, no HTTP types
    └── dto.rs        ← Serde + Schemars request/response types
```

If a module grows large, split it into sub-modules. If a module is tiny (one handler, one
service function), it's acceptable to collapse into a single file — but keep the same logical
separation as comments:

```rust
// routes.rs
// ─── handlers ─────────────────────────────────────────────

// ─── service ──────────────────────────────────────────────
// (only if the module is genuinely small)
```

---

## Crate dependency rules

```
shared      → no internal deps
game-logic  → shared only
screen-hub  → shared only
mqtt-bridge → shared only
api         → shared + game-logic + screen-hub
```

**Never** create a dependency that goes the other way (e.g., `game-logic` importing from `api`).
That would create a cycle. The compiler will reject it — but avoid designing towards it.

If `game-logic` needs something that currently lives in `api`, move it to `shared` first.

---

## CI/CD overview

```
On every PR:
┌─────────────────────────────────────────────────────┐
│  ci.yml                                             │
│  1. cargo fmt --all -- --check                      │
│  2. cargo clippy --workspace --all-targets          │
│  3. cargo build --workspace                         │
│  4. cargo test --workspace                          │
└─────────────────────────────────────────────────────┘

On merge to main:
┌─────────────────────────────────────────────────────┐
│  docker-publish.yml                                 │
│  1. Build Docker image (cargo-chef multi-stage)     │
│  2. Push to ghcr.io/flipgame-hetic/backend          │
│     Tags: branch, semver, short SHA, "latest"       │
└─────────────────────────────────────────────────────┘
```

### What Clippy enforces (examples)

- No unused imports or variables.
- No unnecessary `clone()` or `collect()`.
- No `match` where `if let` suffices.
- No `unwrap()` in library code (via `clippy::unwrap_used` if configured).

---

## Common mistakes to avoid

```
┌───────────────────────────────────────────────────────────────────┐
│ Mistake                   │ Why it's a problem                    │
├───────────────────────────┼───────────────────────────────────────┤
│ Logic in routes.rs        │ Can't unit-test without HTTP layer    │
│ Types duplicated across   │ Diverge silently, hard to find bugs   │
│ crates                    │                                       │
│ Lock held across .await   │ Blocks the Tokio executor thread      │
│ unwrap() in handlers      │ Panics crash the whole server         │
│ Branch named after a      │ Branches outlive people; use feature  │
│ person                    │ names                                 │
│ Commit "WIP" or "fix"     │ Tells future readers nothing          │
│ with no context           │                                       │
│ Skipping --no-verify on   │ CI exists for a reason; fix the root  │
│ commit hook               │ cause                                 │
│ Merging without CI green  │ Broken main blocks everyone           │
└───────────────────────────┴───────────────────────────────────────┘
```
