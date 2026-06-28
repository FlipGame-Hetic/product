# dmd-screen — Overview

`@frontend/dmd-screen` is the Dot Matrix Display (DMD) screen of the pinball cabinet. It is the small, high-contrast display traditionally found at the top of a physical pinball machine. It shows scores, combo messages, game events, and custom animated scenes in a retro pixel aesthetic.

---

## Responsibilities

- Render a dot-matrix-style score display
- Show combo messages and multiplier popups
- Animate DMD scenes: game over, bonus, attract mode
- React to screen-hub events (score updates, game phase changes, combo triggers)
- Display icons and short character animations in pixel style

---

## Port

| Mode | Port |
|---|---|
| Dev server (`pnpm dev:dmd`) | `3002` |
| Docker container (host) | `3001` |
| Docker container (internal) | `80` |

---

## Source Map

```
apps/dmd-screen/
├── src/
│   ├── main.tsx            # React entry point
│   ├── App.tsx             # Root component, DMD scene router
│   │
│   ├── assets/             # Static images, sprites, pixel fonts
│   │
│   ├── components/         # Reusable DMD UI components
│   │
│   ├── dmd/
│   │   ├── scenes/         # Individual DMD scene components
│   │   │   ├── GameOver.tsx
│   │   │   ├── Bonus.tsx
│   │   │   └── Attract.tsx
│   │   └── dmdConfig.ts    # Scene transition rules, timing config
│   │
│   └── main.tsx
│
├── tests/
│   ├── unit/
│   │   ├── setup.ts
│   │   ├── appGameOver.test.ts
│   │   ├── comboPayload.test.ts
│   │   └── icons.test.tsx
│   └── e2e/
│
├── Dockerfile
├── entrypoint.sh
├── vite.config.ts
├── tsconfig.json
├── package.json
└── .env.example
```

---

## Key Technologies

| Technology | Version | Role |
|---|---|---|
| React | 19 | UI component tree |
| Zustand | 5.x | Reactive state |
| Vite | 7.x | Dev server and bundler |
| TypeScript | 5.9 | Type safety |

dmd-screen is the lightest of the three apps. It has no Three.js, no physics, and no audio. It is a pure React + CSS application focused on rendering fast, animated text and pixel graphics.

---

## Application Bootstrap

```
index.html
  |
  +-- loads config.js (window.__ENV__ — runtime config)
  |
  v
src/main.tsx
  |
  v
src/App.tsx
  |
  +-- <ScreenHubListener />   (subscribes to events via @frontend/ws)
  |
  +-- renders current DMD scene based on store state:
       |
       +-- "idle"      -->  <Attract />
       +-- "playing"   -->  <ScoreDisplay />
       +-- "combo"     -->  <ComboFlash />
       +-- "game_over" -->  <GameOver />
       +-- "bonus"     -->  <BonusScene />
```

---

## DMD Aesthetic

The display mimics a physical dot-matrix display with:

- Monospaced, chunky pixel fonts loaded from `@frontend/assets`
- Orange/amber color palette (classic DMD look) via Tailwind tokens
- CSS animations for scan-line effects and brightness pulses
- No anti-aliasing on text — crisp pixel rendering is intentional

---

## Scene System

Scenes are defined in `src/dmd/scenes/`. Each scene is a React component that takes the current game payload as props:

```tsx
// src/dmd/scenes/GameOver.tsx
interface GameOverProps {
  players: Player[]
  finalScore: number
}

export function GameOver({ players, finalScore }: GameOverProps) {
  return (
    <div className="dmd-scene game-over">
      <p className="dmd-text blink">GAME OVER</p>
      <p className="dmd-score">{formatScore(finalScore)}</p>
    </div>
  )
}
```

---

## Combo Payload

Combos arrive as events from the screen-hub with a structured payload. The `comboPayload` module parses and validates these:

```ts
// src/dmd/comboPayload.ts
export function parseComboPayload(raw: unknown): ComboPayload {
  // validates shape, returns typed object or throws
}
```

This is one of the tested modules — tests cover valid payloads and malformed inputs.

---

## Screen-Hub Connection

dmd-screen only needs two environment variables: `VITE_SCREEN_HUB_URL` and `VITE_SCREEN_TOKEN`. It subscribes to events and updates its store:

```ts
const { lastEvent } = useScreenHub(getEnv("VITE_SCREEN_HUB_URL"))

useEffect(() => {
  if (!lastEvent) return
  switch (lastEvent.type) {
    case "score_updated":
      setScore(lastEvent.payload.score)
      break
    case "combo":
      showCombo(parseComboPayload(lastEvent.payload))
      break
    case "game_over":
      setScene("game_over", lastEvent.payload)
      break
  }
}, [lastEvent])
```

---

## Environment Variables

| Variable | Description |
|---|---|
| `VITE_SCREEN_HUB_URL` | Screen-hub WebSocket URL |
| `VITE_SCREEN_TOKEN` | Authentication token for this screen |

No `VITE_API_URL` or `VITE_WS_URL` — dmd-screen does not talk directly to the game backend.
