# Project Map — Flipper

> Cartographie du système : découpage, flux de données, interfaces et responsabilités.
> v1.0.0 — 25/02/26

---

## 1. Vue d'ensemble

Flipper est un jeu de pinball multijoueur cyberpunk sur borne physique. Le système relie des capteurs physiques (ESP32) à un rendu 3D multi-écrans via une chaîne temps réel :

```
Capteurs → ESP32 → MQTT → Bridge Rust → Serveur Rust → WebSocket → 3 Écrans
                                                          ↕
                                                     Cloud (GCP)
```

---

## 2. Blocs du système

### 2.1 IoT / Hardware

| Composant      | Techno       | Responsabilité                                                  |
| -------------- | ------------ | --------------------------------------------------------------- |
| ESP32-S3       | C++          | Lecture capteurs (boutons, plunger, gyroscope, monnayeur), commande actionneurs (solénoïdes, moteurs vibration, DMD) |
| Capteurs       | GPIO/ADC/I2C | Entrées physiques à 200 Hz                                      |
| Actionneurs    | PWM/GPIO/DAC | Feedback haptique, DMD, solénoïdes                              |

**Où trouver :** pas encore dans le repo (développement hardware séparé)

### 2.2 Communication

| Composant       | Techno         | Responsabilité                                                 |
| --------------- | -------------- | -------------------------------------------------------------- |
| Broker MQTT     | Mosquitto      | Pub/sub entre ESP32 et Bridge, transport léger à 200 Hz        |
| Bridge Rust     | Rust           | Désérialisation JSON, validation, filtrage, pont MQTT ↔ WebSocket |

**Où trouver :** pas encore dans le repo (backend)

### 2.3 Backend / Serveur

| Composant       | Techno       | Responsabilité                                                     |
| --------------- | ------------ | ------------------------------------------------------------------ |
| Serveur Rust    | Rust         | Serveur autoritatif : logique de jeu, gestion des ticks, rooms PvP |
| Redis           | Redis        | Cache état de jeu, pub-sub entre instances pour le PvP             |
| PostgreSQL      | PostgreSQL   | Persistance : scores, parties, stats                               |

**Où trouver :** pas encore dans le repo (backend)

### 2.4 Frontend (monorepo pnpm)

| Composant        | Techno                          | Responsabilité                               |
| ---------------- | ------------------------------- | -------------------------------------------- |
| `front-screen`   | React 19 + R3F + Rapier.js      | Écran joueur : vue 3D playfield, physique    |
| `back-screen`    | React 19 + Tailwind v4          | Écran scoreboard : score, HP boss, animations |
| DMD (à venir)    | —                               | Écran matrice : vies, score, alertes         |

**Où trouver :** `frontend/apps/front-screen/` et `frontend/apps/back-screen/`

### 2.5 Packages partagés

| Package              | Chemin                              | Responsabilité                            |
| -------------------- | ----------------------------------- | ----------------------------------------- |
| `@frontend/types`    | `frontend/packages/types/`          | Types TS partagés (GameState, WebSocket)  |
| `@frontend/utils`    | `frontend/packages/utils/`          | Utilitaires (clamp, lerp, formatScore)    |
| `@frontend/ui`       | `frontend/packages/ui/`             | Composants React partagés (ScoreDisplay)  |
| `@frontend/tailwind-config` | `frontend/packages/tailwind-config/` | Thème Tailwind v4 cyberpunk         |
| `@frontend/eslint-config`   | `frontend/packages/eslint-config/`   | Configuration ESLint partagée       |
| `@frontend/tsconfig`        | `frontend/packages/tsconfig/`        | Configs TypeScript de base          |

### 2.6 Cloud / Infra

| Composant | Techno    | Responsabilité                                    |
| --------- | --------- | ------------------------------------------------- |
| GCP       | VPS       | Hébergement serveur Rust, Redis, PostgreSQL       |
| S3 / CDN  | Storage   | Assets statiques (builds, textures, sons)         |
| CI/CD     | GitHub Actions | Tests automatisés, déploiements                |

---

## 3. Flux de données

### 3.1 Input joueur → Rendu (PvE)

```
1. Joueur appuie sur flipper
2. ESP32 capte GPIO → pub MQTT "flipper_appuyé"
3. Bridge Rust reçoit → déclenche feedback haptique local
4. Bridge → Serveur Rust via WebSocket "flipper_appuyé"
5. Serveur → front-screen via WebSocket "activer_flipper"
6. Rapier.js applique la force sur le flipper 3D
```

### 3.2 Collision → Score (PvE)

```
1. Rapier.js détecte collision bille/bumper sur front-screen
2. front-screen → Serveur WebSocket "bumper_touché 100pts"
3. Serveur met à jour score + calcule dégâts boss
4. Serveur → back-screen WebSocket "maj_score + hp_boss"
5. Serveur → DMD WebSocket "maj_score"
6. Serveur → Bridge → MQTT → ESP32 "déclencher_solénoïde"
```

### 3.3 Ultime (PvE)

```
1. ESP32 détecte flippers maintenus → pub MQTT
2. Bridge → Serveur : barreUltime = 100% → prêt
3. ESP32 détecte flippers relâchés → pub MQTT
4. Serveur applique burst dégâts sur boss
5. Serveur diffuse animation ultime sur les 3 écrans
```

### 3.4 PvP — Dégâts croisés

```
1. Bridge J1 → Serveur : "bumper_touché 100pts"
2. Serveur convertit score J1 → dégâts HP J2
3. Serveur → les 4 écrans (PF1, BG1, PF2, BG2) : mise à jour
```

### 3.5 Bille perdue → Fin de partie

```
1. ESP32 → MQTT → Bridge → Serveur : "bille_perdue"
2. Serveur décrémente vies
3. Si vies > 0 : replacer bille
4. Si vies = 0 : game_over diffusé sur tous les écrans
```

---

## 4. Interfaces / Protocoles

### 4.1 MQTT (ESP32 ↔ Bridge)

- **Fréquence :** 200 Hz
- **Format :** JSON
- **Topics principaux :** `flipper_appuyé`, `bille_perdue`, `flippers_maintenus`, `flippers_relachés`, `vibration`, `déclencher_solénoïde`

### 4.2 WebSocket (Bridge ↔ Serveur ↔ Écrans)

- **Fréquence :** 200 Hz
- **Format :** JSON typé

**Messages Serveur → Client (`ServerMessage`) :**

| Type            | Payload              | Description                  |
| --------------- | -------------------- | ---------------------------- |
| `GAME_STATE`    | `GameState`          | État complet du jeu          |
| `PLAYER_JOINED` | `{ playerId }`       | Joueur rejoint la room       |
| `GAME_OVER`     | `{ winnerId }`       | Fin de partie                |

**Messages Client → Serveur (`ClientMessage`) :**

| Type           | Payload              | Description                  |
| -------------- | -------------------- | ---------------------------- |
| `FLIP_LEFT`    | —                    | Activation flipper gauche    |
| `FLIP_RIGHT`   | —                    | Activation flipper droit     |
| `USE_ABILITY`  | `{ abilityId }`      | Utilisation capacité         |
| `JOIN_ROOM`    | `{ roomCode }`       | Rejoindre une room PvP      |

> Types définis dans `frontend/packages/types/src/websocket.ts`

### 4.3 API REST (Frontend ↔ Cloud)

- Scores, XP, stats de parties
- Non implémenté pour le MVP

---

## 5. États du jeu

```
[*] → Attente → SélectionPersonnage → EnJeu → BillePerdue → FinDePartie → [*]
                                         ↺ Buff activé       ↺ Balles restantes
```

> Diagramme complet : `product/uml/state-diagram.mmd`

---

## 6. Où trouver quoi

| Je cherche...                        | Aller à                                         |
| ------------------------------------ | ------------------------------------------------ |
| Types partagés (GameState, WS)       | `frontend/packages/types/src/`                   |
| Utilitaires (math, format)           | `frontend/packages/utils/src/`                   |
| Composants UI partagés               | `frontend/packages/ui/src/`                      |
| Thème / Design tokens                | `frontend/packages/tailwind-config/`             |
| App écran principal (3D)             | `frontend/apps/front-screen/src/`                |
| App scoreboard                       | `frontend/apps/back-screen/src/`                 |
| Config ESLint / TS                   | `frontend/packages/eslint-config/`, `tsconfig/`  |
| Cas d'usage                          | `product/docs/use-cases/`                        |
| Diagrammes UML                       | `product/uml/`                                   |
| Game Design Document                 | `product/docs/GameDesignDocument.md`             |
| Cahier des charges technique         | `product/docs/cdc-technique.md`                  |
| Spécifications MVP                   | `product/docs/mvp.md`                            |
| Backlog frontend                     | `product/docs/BACKLOG.md`                        |
| Architecture interne                 | `product/docs/InternalArhitecture.md`            |
| Schéma architecture (Mermaid)        | `product/docs/SchemaInternalArchitecture.mmd`    |
| Templates GitHub (PR, issues)        | `.github/`                                       |
| Config monorepo pnpm                 | `frontend/pnpm-workspace.yaml`                  |
| Roadmap                              | `product/docs/cdc-technique.md` § 9              |
| Ce document                          | `product/docs/PROJECT_MAP.md`                    |
| Plan de logs                         | `product/docs/LOGS_PLAN.md`                      |

---

## 7. Responsabilités par équipe

| Domaine            | Périmètre                                              |
| ------------------ | ------------------------------------------------------ |
| Frontend           | Monorepo, apps écrans, physique Rapier.js, rendu R3F   |
| Backend            | Serveur Rust, Bridge MQTT↔WS, Redis, PostgreSQL        |
| IoT                | ESP32 firmware C++, capteurs, actionneurs               |
| DevOps             | GCP, Docker, CI/CD GitHub Actions, CDN                 |
| Design / Produit   | Game design, UX borne, assets, documentation produit   |

---

Maxime Bidan - Arnaud Fischer - Louis Dondey - Arthur Jenck - Alexis Gontier
