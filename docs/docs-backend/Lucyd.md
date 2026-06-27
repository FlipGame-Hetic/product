# Lucyd

**Repository:** https://github.com/Jeck0v/Lucyd  
**Crate:** https://crates.io/crates/lucyd  
**Version:** `lucyd = "0.1.9"`

---

## What it is

Lucyd is a Rust library we (Backend TEAM from S.P.A.M.M.E.R) built to auto-generate an interactive documentation and testing interface for our Axum backend covering HTTP REST, WebSocket, and MQTT all at once, served directly at `/docs`.

The problem it solves: no existing solution for Axum / Tokio handles all three protocols in a single place. Documentation tools are either HTTP-only, or split across multiple apps and tools, which creates friction and maintenance overhead. Lucyd brings all three under one roof, with zero external tooling required.

---

## Why we built it ourselves

We could not find a crate that unified HTTP, WebSocket, and MQTT documentation and live testing for Axum. The existing ecosystem treats them as separate concerns, which forces you to maintain separate interfaces or skip documenting non-HTTP endpoints altogether.

Lucyd was extracted from this project into its own crate because we see it living beyond this backend. The plan is to maintain it as a proper open-source product something genuinely reusable by any team building Axum services that mix protocols.

It lives outside the project's GitHub organisation for that reason: it is not a project-internal utility, it is its own thing.

---

## How it works

Annotate your handlers with the appropriate macro. Everything else is automatic.

```rust
use lucyd::{docs_router, lucy_http, lucy_ws, lucy_mqtt};
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

#[derive(Deserialize, JsonSchema)]
pub struct Ping { pub message: String }

#[derive(Serialize, JsonSchema)]
pub struct Pong { pub echo: String }

#[lucy_http(
    method      = "POST",
    path        = "/api/ping",
    tags        = "system",
    description = "Echo back the message",
    request     = Ping,
    response    = Pong,
)]
async fn ping(axum::Json(body): axum::Json<Ping>) -> axum::Json<Pong> {
    axum::Json(Pong { echo: body.message })
}

#[lucy_ws(path = "/ws/events", tags = "realtime", description = "Live event stream")]
async fn events(ws: axum::extract::ws::WebSocketUpgrade) -> impl axum::response::IntoResponse {
    ws.on_upgrade(|_| async {})
}

#[lucy_mqtt(topic = "sensors/temperature", tags = "iot", description = "Temperature readings")]
async fn on_temp(_payload: bytes::Bytes) {}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/api/ping", post(ping))
        .merge(docs_router()); // serves /docs and /docs/spec.json

    axum::serve(
        tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap(),
        app,
    )
    .await
    .unwrap();
}
```

---

## Macros

| Macro | Protocol | Use for |
|---|---|---|
| `#[lucy_http]` | HTTP REST | Standard CRUD routes |
| `#[lucy_ws]` | WebSocket | Real-time bidirectional streams |
| `#[lucy_mqtt]` | MQTT | IoT device messaging topics |

---

## Features

- **HTTP REST** — collapsible endpoint cards grouped by tag, editable request body pre-filled from JSON Schema, Execute button, live cURL preview, response display with status and latency
- **WebSocket** — Connect/Disconnect per endpoint, message textarea, real-time message log (in/out), RFC 6455 close code descriptions
- **MQTT** — shared broker WebSocket connection, Subscribe/Unsubscribe per topic, Publish, per-topic message log
- **JSON Schema** — derive `JsonSchema` on your types and pass them to `request =` / `response =`; typed examples and schema viewers are generated automatically
- **Authentication** — global Authorize modal (Bearer / API Key / Basic), persisted in localStorage, applied to all HTTP requests
- **Models tab** — lists all unique request/response schemas collected from registered endpoints
- **Zero runtime overhead** — registration happens at link time via the `inventory` crate; no reflection, no startup cost

---

## Crate structure

| Crate | Role |
|---|---|
| `lucyd` | Public facade the only crate you import |
| `lucy-macro` | Proc-macros: parse `#[lucy_*]` attributes, emit `inventory::submit!` |
| `lucy-core` | Runtime: global registry, spec generation, Axum router, asset serving |
| `lucy-types` | Shared types: `Protocol`, `EndpointMeta`, `EndpointMetaStatic` |
| `xtask` | Build tooling: `cargo xtask build-ui` for local, for contributor |

---

## Dependencies

```toml
[dependencies]
lucyd = ">= 0.1.9"
schemars = "0.8"
serde    = { version = "1", features = ["derive"] }
axum     = "0.8"
tokio    = { version = "1", features = ["full"] }
```
