# back-screen — Overview

`@frontend/back-screen` is the secondary display of the pinball cabinet. It shows the scoreboard, menu system, character selection, boss warning scenes, credits, and all player-facing UI that is not on the playfield.

Internal project name: **S.P.A.M.E.R.**

---

## Responsibilities

- Display real-time scores and leaderboard
- Show the main menu, character select, and credits scenes
- Animate boss warning / intro sequences
- Play background music and menu audio
- Receive screen events from the backend screen-hub to transition between scenes
- Display player names alongside scores

---

## Port

| Mode | Port |
|---|---|
| Dev server (`pnpm dev:back`) | `3001` |
| Docker container (host) | `3000` |
| Docker container (internal) | `80` |

---

## Source Map

```
apps/back-screen/
├── src/
│   ├── main.tsx               # React entry point
│   ├── App.tsx                # Root component, scene router
│   │
│   ├── api/                   # REST API calls (fetch game state, etc.)
│   │
│   ├── audio/                 # Sound effects and background music
│   │
│   ├── boss/
│   │   ├── bossConfig.ts      # Boss definitions: name, assets, behavior
│   │   └── renderers/         # Per-boss scene rendering components
│   │
│   ├── components/            # Reusable UI components
│   │   └── controls/          # Control hint overlays
│   │
│   ├── hooks/                 # Custom React hooks
│   │
│   ├── menu/                  # Menu system (navigation, actions, sounds)
│   │
│   ├── scenes/
│   │   ├── characterSelect/   # Character selection scene
│   │   └── credits/           # End credits scene
│   │
│   └── stores/                # Zustand stores
│
├── tests/
│   ├── unit/
│   │   ├── setup.ts
│   │   ├── bossConfig.test.ts
│   │   ├── Leaderboard.test.tsx
│   │   ├── menuActions.test.ts
│   │   ├── menuSound.test.ts
│   │   ├── useBackScreenStore.test.ts
│   │   └── useScreenHubClient.test.ts
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
| Three.js | 0.183 | Boss scene 3D rendering (lightweight) |
| Zustand | 5.x | Reactive state management |
| Howler.js | 2.x | Background music and SFX |
| Vite | 7.x | Dev server and bundler |
| TypeScript | 5.9 | Type safety |

---

## Scene Architecture

The app is structured around a scene router. Each scene is a full-screen React component that renders when the corresponding screen-hub event is received.

```
App.tsx
  |
  +-- reads currentScene from store
  |
  +-- renders matching scene:
       |
       +-- "idle"             -->  <Leaderboard />
       +-- "menu"             -->  <Menu />
       +-- "character_select" -->  <CharacterSelect />
       +-- "boss_warning"     -->  <BossScene bossId={...} />
       +-- "credits"          -->  <Credits />
       +-- "playing"          -->  <HUDOverlay />
```

Scene transitions are triggered by events arriving from the screen-hub WebSocket. The store holds the current scene name and any scene-specific payload.

---

## State Management

```ts
// src/stores/backScreenStore.ts
interface BackScreenStore {
  currentScene: SceneName
  players: Player[]
  activeBoss: BossId | null
  setScene: (scene: SceneName, payload?: unknown) => void
  setPlayers: (players: Player[]) => void
}
```

Zustand is used for everything that needs to re-render. Menu navigation state and transient animations use local component state or refs.

---

## Boss System

Boss configurations are defined in `src/boss/bossConfig.ts`. Each boss is a typed object with its assets, name, and scene parameters.

```ts
// src/boss/bossConfig.ts
export const bossConfig: Record<BossId, BossDefinition> = {
  spike: {
    id: "spike",
    name: "Spike",
    warningDuration: 4000,
    assets: { model: "/models/spike.glb", music: "/audio/spike_theme.mp3" },
  },
  // ...
}
```

Each boss has a dedicated renderer in `src/boss/renderers/`. The renderer receives the `BossDefinition` as props and handles all scene logic for that boss.

---

## Menu System

The menu system lives in `src/menu/`. Navigation actions are pure functions that compute the next menu state:

```ts
// src/menu/menuActions.ts
export function navigateDown(state: MenuState): MenuState {
  const nextIndex = (state.selectedIndex + 1) % state.items.length
  return { ...state, selectedIndex: nextIndex }
}
```

Keeping menu logic as pure functions makes it easy to test without rendering.

---

## Screen-Hub Connection

back-screen subscribes to screen events via `@frontend/ws`:

```ts
const { lastEvent } = useScreenHub(getEnv("VITE_SCREEN_HUB_URL"))

useEffect(() => {
  if (!lastEvent) return
  switch (lastEvent.type) {
    case "scene_change":
      setScene(lastEvent.payload.scene)
      break
    case "players_updated":
      setPlayers(lastEvent.payload.players)
      break
  }
}, [lastEvent])
```

---

## Audio

Background music starts and stops on scene transitions. Menu navigation plays short click sounds. All audio is managed via Howler.js with the same `playSound` pattern as front-screen. Audio configuration lives in `src/audio/`.
