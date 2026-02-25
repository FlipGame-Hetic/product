# Plan de Logs — Flipper

> Quelles traces on produit pour observer et valider le bon fonctionnement du système.
> v1.0.0 — 25/02/26

---

## 1. Objectifs

- **Observer** le comportement du système en temps réel pendant le développement et les playtests
- **Diagnostiquer** rapidement les bugs (latence, désynchronisation, physique)
- **Valider** que les flux fonctionnent de bout en bout (capteur → écran)
- **Mesurer** les performances (latence réseau, FPS, fréquence physique)

---

## 2. Couches de logs

### 2.1 IoT / ESP32

| Signal                  | Quand                          | Format                              | Destination       |
| ----------------------- | ------------------------------ | ----------------------------------- | ------------------ |
| `input.flipper`         | Bouton flipper appuyé/relâché  | `{ side, state, timestamp_us }`     | Serial + MQTT      |
| `input.plunger`         | Valeur plunger (ADC)           | `{ value, timestamp_us }`           | Serial + MQTT      |
| `input.gyroscope`       | Mesure gyroscope               | `{ x, y, z, timestamp_us }`        | Serial + MQTT      |
| `input.coin`            | Pièce détectée                 | `{ timestamp_us }`                  | Serial + MQTT      |
| `output.solenoid`       | Solénoïde activé               | `{ id, duration_ms }`              | Serial              |
| `output.vibration`      | Moteur vibration activé        | `{ intensity, duration_ms }`       | Serial              |
| `health.wifi`           | Statut connexion WiFi          | `{ rssi, status }`                 | Serial              |
| `health.loop_time`      | Temps de boucle principale     | `{ duration_us }`                  | Serial (si > seuil) |

**Stockage :** Serial monitor pendant le dev. Pas de persistance long terme côté ESP32.

### 2.2 Broker MQTT (Mosquitto)

| Signal                  | Quand                          | Destination       |
| ----------------------- | ------------------------------ | ------------------ |
| `$SYS/broker/messages/received` | Compteur messages reçus | Logs Mosquitto    |
| `$SYS/broker/clients/connected` | Clients connectés       | Logs Mosquitto    |
| Messages sur topics de jeu      | Chaque interaction      | Bridge Rust        |

**Stockage :** Logs Mosquitto par défaut (rotation fichier).

### 2.3 Bridge Rust (MQTT ↔ WebSocket)

| Signal                  | Quand                          | Format                              | Niveau   |
| ----------------------- | ------------------------------ | ----------------------------------- | -------- |
| `bridge.mqtt.received`  | Message MQTT reçu              | `{ topic, payload_size, ts }`       | DEBUG    |
| `bridge.mqtt.invalid`   | Message MQTT invalide          | `{ topic, error, raw_payload }`     | WARN     |
| `bridge.ws.sent`        | Message WebSocket envoyé       | `{ type, payload_size, ts }`        | DEBUG    |
| `bridge.ws.connected`   | Connexion WS établie           | `{ endpoint }`                      | INFO     |
| `bridge.ws.disconnected`| Connexion WS perdue            | `{ reason, duration_ms }`           | WARN     |
| `bridge.latency`        | Latence MQTT → WS              | `{ duration_ms }`                   | DEBUG    |
| `bridge.throughput`     | Messages/seconde               | `{ count, window_s }`              | INFO     |

**Stockage :** stdout structuré (JSON), collecté par le système de logs GCP.

### 2.4 Serveur Rust (logique de jeu)

| Signal                     | Quand                           | Format                                   | Niveau   |
| -------------------------- | ------------------------------- | ---------------------------------------- | -------- |
| `game.room.created`        | Nouvelle room                   | `{ room_code, mode }`                    | INFO     |
| `game.room.joined`         | Joueur rejoint                  | `{ room_code, player_id }`              | INFO     |
| `game.round.started`       | Partie démarre                  | `{ room_code, character, lives }`        | INFO     |
| `game.round.ended`         | Partie terminée                 | `{ room_code, score, duration_s }`       | INFO     |
| `game.score.updated`       | Score mis à jour                | `{ player_id, delta, total }`            | DEBUG    |
| `game.ball.lost`           | Bille perdue                    | `{ player_id, lives_remaining }`         | INFO     |
| `game.ability.used`        | Capacité activée                | `{ player_id, ability_id }`              | INFO     |
| `game.ultimate.triggered`  | Ultime déclenché                | `{ player_id, damage }`                  | INFO     |
| `game.boss.hit`            | Boss touché                     | `{ boss_id, hp_remaining, damage }`      | DEBUG    |
| `game.boss.defeated`       | Boss vaincu                     | `{ boss_id, total_time_s }`             | INFO     |
| `game.pvp.damage`          | Dégâts PvP infligés             | `{ attacker, target, damage, hp_left }`  | DEBUG    |
| `game.pvp.disconnect`      | Déconnexion en PvP              | `{ player_id, grace_period_s }`          | WARN     |
| `game.pvp.forfeit`         | Forfait par déconnexion         | `{ player_id, winner_id }`              | INFO     |
| `game.tilt.detected`       | Tilt détecté                    | `{ player_id, gyro_values }`             | INFO     |
| `game.tick.slow`           | Tick serveur trop lent          | `{ duration_ms, expected_ms }`           | WARN     |
| `game.state.broadcast`     | Diffusion état de jeu           | `{ room_code, connected_clients }`       | DEBUG    |
| `server.ws.connection`     | Nouvelle connexion WS           | `{ client_id, ip }`                      | INFO     |
| `server.ws.disconnection`  | Déconnexion WS                  | `{ client_id, reason }`                  | INFO     |
| `server.error`             | Erreur non gérée                | `{ error, context }`                     | ERROR    |

**Stockage :** stdout structuré (JSON) → Cloud Logging GCP. Persistance PostgreSQL pour les événements métier (scores, parties).

### 2.5 Frontend (React / Three.js)

| Signal                     | Quand                           | Format                                   | Destination     |
| -------------------------- | ------------------------------- | ---------------------------------------- | --------------- |
| `ws.connected`             | Connexion WebSocket établie     | `{ url, timestamp }`                     | Console         |
| `ws.disconnected`          | Connexion WebSocket perdue      | `{ reason, reconnect_attempt }`          | Console         |
| `ws.message.received`      | Message WebSocket reçu          | `{ type, latency_ms }`                   | Console (DEBUG) |
| `ws.message.sent`          | Message WebSocket envoyé        | `{ type }`                               | Console (DEBUG) |
| `physics.fps`              | FPS moteur physique Rapier      | `{ fps, delta_ms }`                      | Overlay dev     |
| `render.fps`               | FPS rendu Three.js              | `{ fps }`                                | Overlay dev     |
| `game.state.sync`          | État reçu du serveur            | `{ seq, ball_pos, score }`               | Console (DEBUG) |
| `game.collision`           | Collision détectée (Rapier)     | `{ object_a, object_b, force }`          | Console (DEBUG) |
| `screen.role`              | Identification de l'écran       | `{ screen: "front" | "back" | "dmd" }`  | Console         |
| `error.render`             | Erreur de rendu                 | `{ error, component }`                   | Console + Sentry |
| `error.physics`            | Erreur moteur physique          | `{ error, context }`                     | Console + Sentry |

**Stockage :** Console navigateur en dev. Sentry (ou équivalent) pour les erreurs en production.

---

## 3. Événements métier à persister

Ces événements sont stockés en base (PostgreSQL) pour l'historique et les stats.

| Événement              | Table               | Données clés                                     |
| ---------------------- | ------------------- | ------------------------------------------------ |
| Partie démarrée        | `games`             | `id, mode, character, started_at`                |
| Partie terminée        | `games`             | `score, duration_s, lives_used, ended_at`        |
| Score final            | `leaderboard`       | `player_id, score, character, mode, created_at`  |
| Boss vaincu            | `boss_kills`        | `game_id, boss_id, time_to_kill_s`               |
| Match PvP              | `pvp_matches`       | `player1_id, player2_id, winner_id, scores`      |

---

## 4. Niveaux de log

| Niveau   | Usage                                           | Visible en prod |
| -------- | ------------------------------------------------ | --------------- |
| `ERROR`  | Erreur bloquante, crash, état corrompu           | Oui + alerte    |
| `WARN`   | Anomalie non bloquante (déconnexion, tick lent)  | Oui             |
| `INFO`   | Événement métier significatif                    | Oui             |
| `DEBUG`  | Détail technique pour le développement           | Non (dev only)  |

---

## 5. Métriques de performance

| Métrique                    | Seuil acceptable    | Seuil alerte      | Source          |
| --------------------------- | ------------------- | ------------------ | --------------- |
| Latence MQTT → WS           | < 5 ms              | > 20 ms            | Bridge          |
| Latence WS → Rendu          | < 16 ms (60 FPS)    | > 50 ms            | Frontend        |
| Latence totale input→écran  | < 100 ms             | > 150 ms           | End-to-end      |
| FPS rendu Three.js          | ≥ 60 FPS             | < 30 FPS           | Frontend        |
| FPS physique Rapier.js      | ≥ 60 FPS             | < 30 FPS           | Frontend        |
| Tick serveur                | 5 ms (200 Hz)        | > 10 ms            | Serveur         |
| Débit MQTT                  | 200 msg/s            | > 500 msg/s        | Broker          |
| Connexions WS simultanées   | ≤ 6 (2 bornes × 3)  | > 10               | Serveur         |

---

## 6. Outils

| Outil                 | Usage                                           |
| --------------------- | ------------------------------------------------ |
| Serial Monitor        | Debug ESP32 en direct                            |
| MQTT Explorer         | Visualiser les topics et messages MQTT           |
| Console navigateur    | Logs frontend en dev                             |
| Overlay dev (R3F)     | FPS physique/rendu en surimpression              |
| Cloud Logging (GCP)   | Agrégation logs serveur et bridge en production  |
| Sentry (ou équivalent)| Tracking erreurs frontend en production          |
| PostgreSQL            | Requêtes sur l'historique des parties            |

---

## 7. Convention de nommage

Tous les logs suivent le format `domaine.sous-domaine.action` :

```
game.room.created
game.ball.lost
bridge.mqtt.received
server.ws.connection
ws.message.received
input.flipper
health.wifi
```

**Format de sortie** (backend) :

```json
{
  "ts": "2026-02-25T14:30:00.123Z",
  "level": "INFO",
  "event": "game.round.ended",
  "data": {
    "room_code": "ABC123",
    "score": 42500,
    "duration_s": 187
  }
}
```

---

Maxime Bidan - Arnaud Fischer - Louis Dondey - Arthur Jenck - Alexis Gontier
