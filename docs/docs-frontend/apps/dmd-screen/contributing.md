# dmd-screen — Contributing

This guide explains how to add new DMD scenes, components, event handlers, and store actions to `dmd-screen`.

---

## Prerequisites

```bash
pnpm install

cp apps/dmd-screen/.env.example apps/dmd-screen/.env
# VITE_SCREEN_HUB_URL=ws://localhost
# VITE_SCREEN_TOKEN=local-token

pnpm dev:dmd      # http://localhost:3002
```

---

## Adding a New DMD Scene

### 1. Create the scene component

Scenes live in `src/dmd/scenes/`. Keep them simple: display state, animate CSS, done.

```tsx
// src/dmd/scenes/Jackpot.tsx
import { formatScore } from "@frontend/utils"

interface JackpotProps {
  amount: number
}

export function Jackpot({ amount }: JackpotProps) {
  return (
    <div className="dmd-scene jackpot">
      <p className="dmd-text pulse">JACKPOT!</p>
      <p className="dmd-score">{formatScore(amount)}</p>
    </div>
  )
}
```

### 2. Add the scene type to `@frontend/types`

```ts
// packages/types/src/screen.ts
export type DmdScene =
  | "idle"
  | "playing"
  | "combo"
  | "game_over"
  | "bonus"
  | "jackpot"     // <-- add here
```

### 3. Register in App.tsx

```tsx
// src/App.tsx
import { Jackpot } from "@/dmd/scenes/Jackpot"

// In the scene router:
if (currentScene === "jackpot") return <Jackpot amount={payload.amount} />
```

### 4. Handle the screen-hub event

In the screen-hub listener (usually `src/App.tsx` or a dedicated hook):

```ts
case "jackpot":
  setScene("jackpot", event.payload)
  break
```

---

## Adding a New Component

Shared display pieces live in `src/components/`. Follow the same PascalCase naming:

```tsx
// src/components/DmdText.tsx
interface DmdTextProps {
  children: React.ReactNode
  blink?: boolean
  size?: "sm" | "md" | "lg"
}

export function DmdText({ children, blink = false, size = "md" }: DmdTextProps) {
  return (
    <span className={`dmd-text dmd-text--${size} ${blink ? "blink" : ""}`}>
      {children}
    </span>
  )
}
```

---

## Adding a Store Action

The DMD store is small. Keep it that way — only add what scenes need to render:

```ts
// src/stores/dmdStore.ts
import { create } from "zustand"

interface DmdStore {
  currentScene: DmdScene
  score: number
  comboLabel: string | null
  setScene: (scene: DmdScene, payload?: unknown) => void
  setScore: (score: number) => void
  showCombo: (label: string) => void
  reset: () => void
}

export const useDmdStore = create<DmdStore>((set) => ({
  currentScene: "idle",
  score: 0,
  comboLabel: null,
  setScene: (scene, payload) => set({ currentScene: scene }),
  setScore: (score) => set({ score }),
  showCombo: (label) => set({ comboLabel: label }),
  reset: () => set({ currentScene: "idle", score: 0, comboLabel: null }),
}))
```

Always expose a `reset()` action and call it on `game_over` or `idle` events.

---

## Adding a Combo Payload Handler

Combo events carry a payload that must be validated before use. Add the new combo type in `src/dmd/comboPayload.ts`:

```ts
// src/dmd/comboPayload.ts
export type ComboType = "multiplier" | "chain" | "jackpot" | "my_new_combo"

export interface ComboPayload {
  type: ComboType
  label: string
  value: number
}

export function parseComboPayload(raw: unknown): ComboPayload {
  if (!raw || typeof raw !== "object") throw new Error("Invalid combo payload")
  const p = raw as Record<string, unknown>
  if (typeof p.type !== "string")  throw new Error("Missing combo type")
  if (typeof p.label !== "string") throw new Error("Missing combo label")
  if (typeof p.value !== "number") throw new Error("Missing combo value")
  return { type: p.type as ComboType, label: p.label, value: p.value }
}
```

---

## DMD Visual Guidelines

dmd-screen has a strict visual style. Keep these rules when adding scenes:

- Text must use the pixel font loaded from `@frontend/assets`. Do not use system fonts.
- Colors: amber/orange for primary content (`neon-yellow` token), dim gray for secondary text.
- Animations: CSS keyframes only. No JS-driven animation libraries.
- No images or gradients — the DMD aesthetic is monochromatic and flat.
- Blink animations should use a toggle class (`blink`) not opacity tweening.

```css
/* Example blink animation in index.css */
@keyframes dmd-blink {
  0%, 49% { opacity: 1; }
  50%, 100% { opacity: 0; }
}
.blink { animation: dmd-blink 0.8s step-end infinite; }
```

---

## Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Scene component | `PascalCase.tsx` in `dmd/scenes/` | `Jackpot.tsx` |
| UI component | `PascalCase.tsx` in `components/` | `DmdText.tsx` |
| Scene name (string) | `snake_case` | `"jackpot"` |
| Event type (string) | `snake_case` | `"combo"`, `"jackpot"` |
| Store file | `camelCaseStore.ts` | `dmdStore.ts` |
| Combo type (string) | `snake_case` | `"multiplier"`, `"chain"` |

---

## Before Opening a Pull Request

```bash
pnpm typecheck
pnpm lint
pnpm test
pnpm format
```
