# front-screen — Contributing

This guide explains how to add new code to `front-screen`: new components, stores, hooks, sounds, and physics objects. Follow the patterns here to keep the codebase consistent.

---

## Prerequisites

```bash
# From the repo root
pnpm install

# Copy and fill environment file
cp apps/front-screen/.env.example apps/front-screen/.env

# Start the dev server
pnpm dev:front      # http://localhost:3000
```

Make sure your local backend is running. Without it, the WebSocket connection will fail (the app still renders, but no game events will arrive).

---

## Adding a New React Component

### Where to put it

All components live under `src/components/`. Group by domain, not by type:

```
src/components/
  bumpers/          # All bumper-related components
    Bumper.tsx
    BumperLight.tsx
    BumperCollision.tsx
  flippers/
    Flipper.tsx
    FlipperArm.tsx
```

One file per component. Do not put multiple exported components in the same file unless they are tightly coupled (e.g., a parent and its internal sub-parts that are never used elsewhere).

### Component template

```tsx
// src/components/ballSavers/BallSaverGate.tsx
import { useRef } from "react"
import { useFrame } from "@react-three/fiber"
import { RigidBody } from "@react-three/rapier"
import { useGameStore } from "@/stores/gameStore"

interface BallSaverGateProps {
  position: [number, number, number]
}

export function BallSaverGate({ position }: BallSaverGateProps) {
  const ref = useRef<THREE.Mesh>(null)
  const isActive = useGameStore((s) => s.ballSaverActive)

  useFrame(() => {
    if (!ref.current) return
    ref.current.visible = isActive
  })

  return (
    <RigidBody type="fixed" position={position}>
      <mesh ref={ref}>
        <boxGeometry args={[1, 0.1, 0.5]} />
        <meshStandardMaterial color="cyan" />
      </mesh>
    </RigidBody>
  )
}
```

### Naming rules

- Component file: `PascalCase.tsx`
- Component function: same `PascalCase`
- Props interface: `PascalCase + Props` (e.g., `BallSaverGateProps`)
- Export: named export, not default export

---

## Adding a Zustand Store

Create a new file in `src/stores/`:

```ts
// src/stores/plungerStore.ts
import { create } from "zustand"

interface PlungerStore {
  chargeLevel: number         // 0..1
  isCharging: boolean
  setChargeLevel: (level: number) => void
  startCharging: () => void
  release: () => void
}

export const usePlungerStore = create<PlungerStore>((set) => ({
  chargeLevel: 0,
  isCharging: false,
  setChargeLevel: (level) => set({ chargeLevel: level }),
  startCharging: () => set({ isCharging: true }),
  release: () => set({ isCharging: false, chargeLevel: 0 }),
}))
```

Rules:
- Keep stores focused on one domain.
- Actions are defined inside `create()`, not in separate helper functions.
- Initial state and actions go in the same object.
- Reset to initial state on `game_over` events in the `WebSocketSync` component.

---

## Adding a Custom Hook

Hooks live in `src/hooks/`. A hook should encapsulate one behavior:

```ts
// src/hooks/useFlipperInput.ts
import { useEffect } from "react"
import { usePlungerStore } from "@/stores/plungerStore"

export function useFlipperInput() {
  const startCharging = usePlungerStore((s) => s.startCharging)
  const release = usePlungerStore((s) => s.release)

  useEffect(() => {
    const onKeyDown = (e: KeyboardEvent) => {
      if (e.code === "Space") startCharging()
    }
    const onKeyUp = (e: KeyboardEvent) => {
      if (e.code === "Space") release()
    }

    window.addEventListener("keydown", onKeyDown)
    window.addEventListener("keyup", onKeyUp)
    return () => {
      window.removeEventListener("keydown", onKeyDown)
      window.removeEventListener("keyup", onKeyUp)
    }
  }, [startCharging, release])
}
```

Rules:
- Prefix with `use`.
- Clean up event listeners and subscriptions in the returned cleanup function.
- Do not call hooks conditionally inside the hook body.

---

## Adding Physics Objects

Physics objects use `@react-three/rapier`. The pattern:

```tsx
import { RigidBody, CuboidCollider } from "@react-three/rapier"

export function Wall({ position }: { position: [number, number, number] }) {
  return (
    <RigidBody type="fixed" position={position} colliders={false}>
      <CuboidCollider args={[0.1, 2, 0.5]} />
      <mesh>
        <boxGeometry args={[0.2, 4, 1]} />
        <meshStandardMaterial color="gray" />
      </mesh>
    </RigidBody>
  )
}
```

Collision groups are defined in `src/physics/`. Use them to prevent balls from colliding with specific objects (e.g., ball saver sensor zones).

---

## Adding a Sound

1. Add the audio file to `public/sounds/`.
2. Register it in `src/audio/soundConfig.ts`:

```ts
export const soundConfig = {
  bumper_hit:       { src: "/sounds/bumper_hit.mp3",  volume: 0.9 },
  ball_launch:      { src: "/sounds/ball_launch.mp3", volume: 1.0 },
  my_new_sound:     { src: "/sounds/new_sound.mp3",   volume: 0.7 },  // <-- add here
}
```

3. Call it where needed:

```ts
import { playSound } from "@/audio/soundEngine"
playSound("my_new_sound")
```

---

## Adding a WebSocket Event Handler

Game events arrive in `src/components/WebSocketSync.tsx` (or the equivalent top-level sync component). To handle a new event type:

```ts
// In WebSocketSync.tsx
const event = JSON.parse(lastMessage.data) as ScreenEvent

switch (event.type) {
  case "score_updated":
    useGameStore.getState().setScore(event.payload.score)
    break
  case "my_new_event":                              // <-- add case
    useMyStore.getState().handleMyEvent(event.payload)
    break
}
```

The event type must also be declared in `@frontend/types` so all apps share the same contract.

---

## Naming Conventions Summary

| Thing | Convention | Example |
|---|---|---|
| Component file | `PascalCase.tsx` | `BallSaverGate.tsx` |
| Hook file | `useCamelCase.ts` | `useFlipperInput.ts` |
| Store file | `camelCaseStore.ts` | `plungerStore.ts` |
| Props interface | `ComponentNameProps` | `BallSaverGateProps` |
| Store hook export | `usePascalCaseStore` | `usePlungerStore` |
| Sound key | `snake_case` | `"bumper_hit"` |
| WebSocket event type | `snake_case` | `"score_updated"` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_BALLS = 3` |

---

## Before Opening a Pull Request

```bash
# Type check
pnpm typecheck

# Lint
pnpm lint

# All unit tests
pnpm test

# Format code
pnpm format
```

The pre-commit hook runs Prettier and ESLint automatically on staged files. The pre-push hook runs a type check. Both must pass before you can push.
