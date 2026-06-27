# front-screen — Overview

`@frontend/front-screen` is the main player-facing screen of the pinball cabinet. It renders the 3D playfield in real-time using React Three Fiber and Rapier physics, handles all input events, and streams game state from the backend over WebSocket.

---

## Responsibilities

- Render the 3D pinball playfield (Three.js scene via React Three Fiber)
- Simulate physics with Rapier (ball motion, collisions, gravity)
- Handle all player input: keyboard, gamepad (L1/R1 for flippers, plunger)
- Connect to the backend via WebSocket and apply incoming game events
- Play spatial audio (Howler.js)
- Display post-processing effects (bloom, depth of field)

---

## Port

| Mode | Port |
|---|---|
| Dev server (`pnpm dev:front`) | `3000` |
| Docker container (host) | `3002` |
| Docker container (internal) | `80` |

---

## Source Map

```
apps/front-screen/
├── src/
│   ├── main.tsx               # React entry point, mounts <App />
│   ├── App.tsx                # Root component, sets up Canvas + providers
│   ├── index.css              # Tailwind imports + theme
│   │
│   ├── audio/
│   │   ├── soundEngine.ts     # Howler wrapper, sound playback API
│   │   └── soundConfig.ts     # Sound definitions (file paths, volumes)
│   │
│   ├── components/            # All 3D and UI components
│   │   ├── balls/             # Ball mesh, physics body, manager
│   │   ├── ballSavers/        # Ball saver gate mechanics
│   │   ├── bumpers/           # Pop bumper logic + hit reactions
│   │   ├── flippers/          # Flipper physics + control binding
│   │   ├── plunger/           # Plunger spring + launch
│   │   ├── playfield/         # Static geometry, lights, decorations
│   │   ├── physics/           # Collision groups, walls, snap joints
│   │   ├── vfx/               # Particle effects, trail, flash
│   │   └── postprocessing/    # Three.js EffectComposer passes
│   │
│   ├── config/                # Static game configuration (constants)
│   ├── debug/                 # Leva debug panel integration
│   ├── gameplay/              # High-level game loop coordination
│   ├── hooks/                 # Custom React hooks
│   ├── input/                 # Keyboard router, gamepad polling
│   ├── stores/                # Zustand stores
│   ├── types/                 # App-local TypeScript types
│   └── utils/                 # App-local helpers
│
├── tests/
│   ├── unit/                  # Vitest tests
│   └── e2e/                   # Playwright specs
│
├── public/                    # Static assets served at /
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
| React Three Fiber | 9.x | Three.js declarative wrapper |
| @react-three/rapier | latest | Rapier physics bindings |
| Three.js | 0.183 | 3D rendering |
| Zustand | 5.x | Reactive state management |
| Howler.js | 2.x | Spatial audio |
| Leva | latest | In-browser debug controls |
| Vite | 7.x | Dev server and bundler |
| TypeScript | 5.9 | Type safety |

---

## Application Bootstrap

```
index.html
  |
  +-- loads config.js (window.__ENV__ — Docker runtime config)
  |
  v
src/main.tsx
  |
  +-- ReactDOM.createRoot(document.getElementById("root"))
  |
  v
src/App.tsx
  |
  +-- <Canvas>          (React Three Fiber scene root)
  |     |
  |     +-- <Physics>   (Rapier world)
  |     |     |
  |     |     +-- <Balls />, <Flippers />, <Bumpers />, <Plunger />, ...
  |     |
  |     +-- <EffectComposer> (post-processing)
  |
  +-- <HUD />           (2D overlay: score, debug panel)
  +-- <WebSocketSync /> (connects to backend, applies events to stores)
```

---

## State Management

front-screen uses two complementary patterns:

### Zustand — reactive UI state

```ts
// src/stores/gameStore.ts
import { create } from "zustand"

interface GameStore {
  score: number
  ballsLeft: number
  gamePhase: "idle" | "playing" | "game_over"
  setScore: (score: number) => void
}

export const useGameStore = create<GameStore>((set) => ({
  score: 0,
  ballsLeft: 3,
  gamePhase: "idle",
  setScore: (score) => set({ score }),
}))
```

### Module registries — non-reactive physics state

Three.js objects, Rapier rigid bodies, and audio instances update at 60 fps and must not trigger React re-renders:

```ts
// src/gameplay/ballRegistry.ts
const bodies = new Map<string, RapierRigidBody>()

export const ballRegistry = {
  register:   (id: string, body: RapierRigidBody) => bodies.set(id, body),
  unregister: (id: string) => bodies.delete(id),
  get:        (id: string) => bodies.get(id),
  getAll:     () => Array.from(bodies.values()),
}
```

Rule: if a change must re-render a React component, use Zustand. If it only updates a Three.js object or a physics body, use a registry or a ref.

---

## Input System

Input is centralized in `src/input/`. A keyboard router reads raw events and dispatches named game actions:

```
KeyboardEvent ("keydown", "keyup")
        |
        v
src/input/keyboardRouter.ts
        |
        +-- maps key codes to game actions
        |   e.g., "ShiftLeft" -> "flipper:left"
        |         "Space"     -> "plunger:charge"
        |
        v
Dispatched to relevant components via store or direct callback
```

Gamepad support follows the same pattern via `navigator.getGamepads()` polled inside the animation loop.

---

## WebSocket Sync

`@frontend/ws` provides `useGameSocket`, which is mounted in a top-level `<WebSocketSync />` component. Incoming events are routed to the appropriate Zustand store:

```ts
// src/components/WebSocketSync.tsx
const { lastMessage } = useGameSocket(getEnv("VITE_WS_URL"))

useEffect(() => {
  if (!lastMessage) return
  const event = JSON.parse(lastMessage.data)
  if (event.type === "score_updated") useGameStore.getState().setScore(event.payload.score)
  if (event.type === "ball_drained")  useGameStore.getState().drainBall()
}, [lastMessage])
```

---

## Audio

Sound is played through `soundEngine.ts`, a thin wrapper around Howler:

```ts
import { playSound } from "@/audio/soundEngine"

// Inside a collision callback:
playSound("bumper_hit", { volume: 0.8 })
```

Sound files are defined in `soundConfig.ts`. Add a new sound by registering it there and calling `playSound` with its key.
