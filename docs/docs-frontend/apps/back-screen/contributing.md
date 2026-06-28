# back-screen — Contributing

This guide explains how to add new code to `back-screen`: new scenes, menu items, boss configurations, components, and stores.

---

## Prerequisites

```bash
# From the repo root
pnpm install

cp apps/back-screen/.env.example apps/back-screen/.env
# Edit .env with your local backend URLs

pnpm dev:back      # http://localhost:3001
```

---

## Adding a New Scene

A scene is a full-screen React component that replaces the current view when activated.

### 1. Create the scene component

```tsx
// src/scenes/victory/Victory.tsx
import { useEffect } from "react"
import { useBackScreenStore } from "@/stores/backScreenStore"

export function Victory() {
  const players = useBackScreenStore((s) => s.players)

  return (
    <div className="scene-victory">
      <h1>Game Over</h1>
      {players.map((p) => (
        <p key={p.id}>{p.name}: {p.score}</p>
      ))}
    </div>
  )
}
```

### 2. Register it in the scene type

Add the scene name to `@frontend/types`:

```ts
// packages/types/src/screen.ts
export type SceneName =
  | "idle"
  | "menu"
  | "character_select"
  | "boss_warning"
  | "credits"
  | "victory"    // <-- add here
```

### 3. Add it to the scene router in `App.tsx`

```tsx
// src/App.tsx
import { Victory } from "@/scenes/victory/Victory"

// In the render logic:
if (currentScene === "victory") return <Victory />
```

### 4. Handle the incoming event in the screen-hub listener

```ts
case "scene_change":
  if (event.payload.scene === "victory") {
    setScene("victory", event.payload)
  }
  break
```

---

## Adding a Boss

Each boss needs a config entry and a renderer component.

### 1. Add the config entry

```ts
// src/boss/bossConfig.ts
export const bossConfig: Record<BossId, BossDefinition> = {
  // existing bosses...
  void_lord: {                                // <-- new boss
    id: "void_lord",
    name: "Void Lord",
    warningDuration: 5000,
    assets: {
      model: "/models/void_lord.glb",
      music: "/audio/void_lord_theme.mp3",
    },
  },
}
```

Also add `"void_lord"` to the `BossId` union type in `@frontend/types`.

### 2. Create the renderer

```tsx
// src/boss/renderers/VoidLordRenderer.tsx
import type { BossDefinition } from "@frontend/types"

interface VoidLordRendererProps {
  boss: BossDefinition
}

export function VoidLordRenderer({ boss }: VoidLordRendererProps) {
  return (
    <div className="boss-scene">
      <h2 className="warning-text">WARNING: {boss.name}</h2>
      {/* Three.js canvas, animations, etc. */}
    </div>
  )
}
```

### 3. Register the renderer in `BossScene`

```tsx
// src/boss/BossScene.tsx
import { VoidLordRenderer } from "./renderers/VoidLordRenderer"

const renderers: Record<BossId, ComponentType<{ boss: BossDefinition }>> = {
  spike:      SpikeRenderer,
  void_lord:  VoidLordRenderer,   // <-- add here
}
```

---

## Adding a Menu Item

Menu items are defined in `src/menu/`. Navigation logic is a pure function:

```ts
// src/menu/menuConfig.ts
export const mainMenuItems: MenuItem[] = [
  { id: "start",    label: "Start Game",   action: "start_game" },
  { id: "options",  label: "Options",      action: "open_options" },
  { id: "credits",  label: "Credits",      action: "show_credits" },
  { id: "my_item",  label: "My New Item",  action: "my_action" },  // <-- add here
]
```

Handle the action in the dispatch logic:

```ts
// src/menu/menuActions.ts
export function handleMenuAction(action: MenuAction, store: BackScreenStore) {
  switch (action) {
    case "my_action":
      store.setScene("my_scene")
      break
  }
}
```

---

## Adding a Store

```ts
// src/stores/victoryStore.ts
import { create } from "zustand"

interface VictoryStore {
  winner: Player | null
  setWinner: (player: Player) => void
  reset: () => void
}

export const useVictoryStore = create<VictoryStore>((set) => ({
  winner: null,
  setWinner: (player) => set({ winner: player }),
  reset: () => set({ winner: null }),
}))
```

Always add a `reset()` action and call it when the `game_over` or `idle` event arrives in the screen-hub listener.

---

## Adding a Component

Components live in `src/components/`. Use PascalCase for file and export names:

```tsx
// src/components/PlayerCard.tsx
interface PlayerCardProps {
  player: Player
  rank: number
}

export function PlayerCard({ player, rank }: PlayerCardProps) {
  return (
    <div className="player-card">
      <span className="rank">#{rank}</span>
      <span className="name">{player.name}</span>
      <span className="score">{formatScore(player.score)}</span>
    </div>
  )
}
```

Use `@frontend/utils` functions (`cn`, `formatScore`) instead of reimplementing them locally.

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Scene component | `PascalCase.tsx` in `scenes/<name>/` | `Victory.tsx` |
| Boss renderer | `PascalCaseRenderer.tsx` in `boss/renderers/` | `VoidLordRenderer.tsx` |
| Store file | `camelCaseStore.ts` | `victoryStore.ts` |
| Menu config | items as array of `{ id, label, action }` objects | |
| Scene name (string) | `snake_case` | `"character_select"` |
| Boss id (string) | `snake_case` | `"void_lord"` |
| Event action (string) | `snake_case` | `"start_game"` |

---

## Before Opening a Pull Request

```bash
pnpm typecheck
pnpm lint
pnpm test
pnpm format
```
