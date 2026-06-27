# Crate: mqtt-bridge

## Role

`mqtt-bridge` is an **independent binary** that acts as a relay between the local MQTT broker
(where ESP32 devices publish hardware events) and the central `api` WebSocket endpoint.

It has no business logic. It translates bytes from one protocol to the other, in both directions.

Running as a separate binary means:
- The bridge can restart independently of the API (hardware connectivity doesn't block the game server).
- MQTT and HTTP scaling concerns are separated.

---

## Source tree

```
crates/mqtt-bridge/src/
├── main.rs       ← entry point: load config, init tracing, run bridge loop
├── bridge.rs     ← Bridge struct: orchestrates three concurrent tasks
├── client.rs     ← MqttClient wrapper around rumqttc
├── config.rs     ← BridgeConfig: reads env vars
├── handler.rs    ← message parsing and serialization
└── errors.rs     ← BridgeError enum
```

---

## Data flow

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                         mqtt-bridge                            │
  │                                                                 │
  │  Mosquitto ──publish──► mqtt_inbound_loop                       │
  │                                │                               │
  │                         mpsc::channel (cap=256)                 │
  │                                │                               │
  │                         ws_write_loop ──text frame──► /ws/bridge│
  │                                                                 │
  │  Mosquitto ◄──publish── ws_outbound_loop ◄──text frame── /ws/bridge│
  └─────────────────────────────────────────────────────────────────┘
```

Three tasks run concurrently inside a `tokio::select!`:

```
tokio::select! {
    _ = mqtt_inbound_loop(mqtt_client, inbound_tx)  => { ... }
    _ = ws_write_loop(inbound_rx, ws_sink)          => { ... }
    _ = ws_outbound_loop(ws_stream, mqtt_client)    => { ... }
}
```

If **any** of the three tasks exits (error or clean shutdown), `select!` cancels the other two.
The outer loop then reconnects everything with exponential backoff.

---

## Reconnection logic

```
loop {
    connect to MQTT broker
    connect to API WebSocket

    tokio::select! { ... }   ← runs until any task fails

    error logged
    sleep(backoff_delay)     ← starts at config.reconnect_delay_ms
    backoff_delay = min(backoff_delay * 2, max_delay)
}
```

This means a dropped MQTT connection or a restarted API automatically triggers a reconnect.
No manual intervention needed.

---

## Message format

All messages sent over the WebSocket are JSON-serialized `WsMessage` (defined in `shared`):

```json
{
  "dir": "inbound",
  "payload": {
    "type": "Button",
    "device_id": "esp32-01",
    "button_id": "L1",
    "pressed": true
  }
}
```

Outbound (API → MQTT):

```json
{
  "dir": "outbound",
  "payload": {
    "type": "Command",
    "target": "esp32-01",
    "command": "rumble"
  }
}
```

`handler.rs` handles the parse/serialize and logs malformed messages without crashing.

---

## Config

All configuration is via environment variables (no config file):

| Variable            | Default                          |
|---------------------|----------------------------------|
| `MQTT_HOST`         | `localhost`                      |
| `MQTT_PORT`         | `1883`                           |
| `MQTT_CLIENT_ID`    | `mqtt-bridge`                    |
| `WS_URL`            | `ws://localhost:8080/ws/bridge`  |
| `RECONNECT_DELAY_MS`| `1000`                           |
| `MAX_RECONNECT_MS`  | `30000`                          |

---

## Channel back-pressure

The inbound MQTT → WebSocket channel is bounded at **256 messages**. If the WebSocket write loop
falls behind (API is slow), the channel fills up and `mqtt_inbound_loop` applies back-pressure on
the MQTT subscription — it stops polling new messages until space frees up.

This prevents unbounded memory growth during traffic bursts.

---

## How to add handling for a new MQTT topic

1. Add the new subtopic variant to `Subtopic` in `shared/src/dto.rs`:

```rust
pub enum Subtopic {
    InputButton,
    InputPlunger,
    MyNewTopic,   // ← add here
    // …
}
```

2. Add the parsing arm in `shared/src/dto.rs` `Topic::parse()`:

```rust
"my_new_topic" => Some(Subtopic::MyNewTopic),
```

3. Add the corresponding `InboundMessage` variant in `shared/src/events.rs` if the message carries
   a new payload shape.

4. Handle it in `api/src/modules/mqtt.rs` in the dispatch function.

The bridge itself (`mqtt-bridge`) does not need to change — it relays all topics blindly.
