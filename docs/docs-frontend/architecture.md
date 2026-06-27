# Architecture

This document describes the high-level structure of the frontend monorepo: how the workspace is organized, how applications relate to shared packages, and how data flows at runtime.

---

## Monorepo Overview

The frontend is a **pnpm workspace** monorepo. All apps share a single `node_modules` installation at the root. Internal packages are referenced with the `workspace:*` protocol so no publishing step is needed during development.

```
frontend/
├── apps/
│   ├── front-screen/     # 3D pinball playfield (React Three Fiber)
│   ├── back-screen/      # Scoreboard + menu (React)
│   └── dmd-screen/       # Dot Matrix Display (React)
├── packages/
│   ├── types/            # Shared TypeScript types
│   ├── utils/            # Pure utility functions
│   ├── ui/               # Shared React components
│   ├── ws/               # WebSocket / ScreenHub client
│   ├── assets/           # Fonts and static assets
│   ├── tailwind-config/  # Design tokens (Tailwind v4)
│   ├── eslint-config/    # Shared ESLint flat config
│   └── tsconfig/         # Shared TypeScript configs
├── .github/workflows/    # CI/CD pipelines
├── docker-compose.yml    # Local multi-container setup
├── vitest.config.ts      # Root test runner (multi-project)
└── package.json          # Root workspace scripts
```

---

## The Three Screens

The project simulates a physical pinball arcade cabinet with three independent screens. Each screen is a separate Vite + React application that runs in its own container.

```
+--------------------------------------------------+
|                  PINBALL CABINET                 |
|                                                  |
|  +------------------+    +--------------------+  |
|  |   back-screen    |    |    dmd-screen      |  |
|  |  (Scoreboard /   |    |  (Dot Matrix       |  |
|  |   Menu / Boss)   |    |   Display)         |  |
|  |                  |    |                    |  |
|  |  port 3000       |    |  port 3002         |  |
|  +------------------+    +--------------------+  |
|                                                  |
|  +--------------------------------------------+  |
|  |               front-screen                 |  |
|  |        (3D Playfield - Three.js +          |  |
|  |         Rapier physics + R3F)              |  |
|  |                                            |  |
|  |                 port 3001                  |  |
|  +--------------------------------------------+  |
+--------------------------------------------------+
```

All three apps connect to the same backend over WebSocket. They communicate with each other indirectly through the backend's screen-hub event bus.

---

## Package Dependency Graph

```
apps/front-screen ──┐
apps/back-screen  ──┼──> @frontend/ws      ──> @frontend/types
apps/dmd-screen   ──┘    @frontend/utils       @frontend/utils
                    │
                    ├──> @frontend/ui      ──> @frontend/types
                    │
                    ├──> @frontend/types
                    ├──> @frontend/utils
                    ├──> @frontend/assets
                    └──> @frontend/tailwind-config
```

Shared packages do not depend on any app. They export pure functions, types, or React components only.

---

## Shared Packages

### `@frontend/types`

Single source of truth for all game domain types. Every other package and app imports from here.

```ts
// What this package exports (examples)
export type GameState = "idle" | "playing" | "game_over"
export type Player = { id: string; name: string; score: number }
export type ScreenEvent = { type: string; payload: unknown }
```

### `@frontend/utils`

Pure, side-effect-free helper functions. No React, no browser APIs.

```ts
import { cn } from "@frontend/utils"   // clsx + tailwind-merge
import { formatScore } from "@frontend/utils"
import { getEnv } from "@frontend/utils"
```

### `@frontend/ws`

Wraps WebSocket connections and exposes React hooks. Two main transports:

- `useGameSocket` — direct WebSocket to the game backend
- `useScreenHub` — screen-to-screen event bus (screens subscribe to each other's events)

```
         +---------------+
         |    Backend    |
         |  (WebSocket)  |
         +-------+-------+
                 |
        +--------+--------+
        |                 |
+-------+------+  +-------+------+
| useGameSocket|  |useScreenHub  |
|   (raw WS)   |  | (event bus)  |
+--------------+  +--------------+
       |                  |
  front-screen      all 3 screens
```

### `@frontend/ui`

Presentational React components that are reused across screens (e.g., `ScoreDisplay`, `HUD`). No state management, no side effects — props in, JSX out.

### `@frontend/tailwind-config`

Tailwind v4 theme with the project's design tokens. Import once per app in the entry CSS:

```css
@import "@frontend/tailwind-config/theme.css";
```

Available color tokens: `neon-pink`, `neon-cyan`, `neon-purple`, `neon-yellow`, plus `surface-*` semantic tokens.

---

## Runtime Configuration

Vite builds static files at compile time. Environment variables (`VITE_*`) are baked in during `vite build`. For Docker deployments, variables are injected at container startup via `entrypoint.sh`, which writes `window.__ENV__` into `dist/config.js` before starting the preview server.

```
Docker container starts
        |
        v
entrypoint.sh reads environment variables
        |
        v
Writes dist/config.js:
  window.__ENV__ = { VITE_WS_URL: "...", ... }
        |
        v
vite preview --host 0.0.0.0 --port 80
        |
        v
index.html loads config.js (via <script> tag)
        |
        v
App reads window.__ENV__ via getEnv() util
```

This pattern allows a single Docker image to be configured differently per environment without rebuilding.

---

## State Management Strategy

Each app uses two complementary patterns depending on update frequency.

### Zustand (reactive state)

Used for data that React needs to render: scores, game phase, UI visibility, player data.

```ts
// stores/gameStore.ts
import { create } from "zustand"

interface GameStore {
  score: number
  setScore: (score: number) => void
}

export const useGameStore = create<GameStore>((set) => ({
  score: 0,
  setScore: (score) => set({ score }),
}))
```

### Module singletons / refs (non-reactive state)

Used for high-frequency game loop data (60 fps physics, audio, Three.js objects). Storing this in Zustand would cause unnecessary re-renders.

```ts
// registries/ballRegistry.ts
const balls = new Map<string, BallRef>()

export const ballRegistry = {
  add: (id: string, ref: BallRef) => balls.set(id, ref),
  get: (id: string) => balls.get(id),
  remove: (id: string) => balls.delete(id),
}
```

Rule of thumb: if React needs to re-render when the value changes, use Zustand. If not, use a module-level variable or ref.

---

## Build Pipeline

```
pnpm build
     |
     v (runs pnpm -r run build for each workspace package)
     |
     +---> packages/* (types, utils, ui, ws) compiled first
     |
     +---> apps/front-screen: tsc -b && vite build --> dist/
     +---> apps/back-screen:  tsc -b && vite build --> dist/
     +---> apps/dmd-screen:   tsc -b && vite build --> dist/
```

TypeScript compilation (`tsc -b`) runs before Vite to catch type errors. Vite then bundles and optimizes assets for production.

---

## Naming Conventions

### Files and directories

- React components: `PascalCase.tsx` — e.g., `BallsManager.tsx`, `ScoreDisplay.tsx`
- Hooks: `camelCase.ts` prefixed with `use` — e.g., `useGameSocket.ts`, `useFlipperInput.ts`
- Stores: `camelCase.ts` suffixed with `Store` — e.g., `gameStore.ts`, `uiStore.ts`
- Utilities: `camelCase.ts` — e.g., `formatScore.ts`, `env.ts`
- Types: `camelCase.ts` — e.g., `player.ts`, `screenEvents.ts`
- Test files: mirror the source file name with `.test.ts` / `.test.tsx`

### Identifiers

- React components: `PascalCase`
- Hooks: `useCamelCase`
- Constants: `SCREAMING_SNAKE_CASE`
- Zustand stores: `usePascalCaseStore` when exported as a hook
- Event types: string literals in `snake_case` (e.g., `"ball_drained"`, `"score_updated"`)

### Package names

All internal packages follow `@frontend/<name>` (e.g., `@frontend/types`, `@frontend/ws`).
