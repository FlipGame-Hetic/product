# front-screen — Testing

This document covers how the test suite for `front-screen` is structured and how to write new tests for components, stores, hooks, and physics logic.

---

## Test Setup

| Tool | Version | Purpose |
|---|---|---|
| Vitest | 4.x | Unit and component tests |
| happy-dom | latest | DOM environment (lighter than jsdom) |
| @testing-library/react | latest | Component rendering |
| Playwright | 1.58 | End-to-end browser tests |

Tests live in:

```
apps/front-screen/
└── tests/
    ├── unit/
    │   ├── setup.ts        # Vitest globals: vi.mock stubs, DOM setup
    │   └── *.test.ts(x)    # All unit tests
    └── e2e/
        ├── README.md
        └── *.spec.ts       # Playwright specs
```

---

## Running Tests

```bash
# Run all unit tests once
pnpm --filter @frontend/front-screen test

# Watch mode (re-runs on change)
pnpm --filter @frontend/front-screen test:watch

# Run a specific file
pnpm --filter @frontend/front-screen test -- tests/unit/gameStore.test.ts

# Run all E2E tests
pnpm --filter @frontend/front-screen test:e2e

# E2E in headed mode (visible browser)
pnpm --filter @frontend/front-screen test:e2e -- --headed

# From the repo root (all projects at once)
pnpm test
```

---

## Vitest Configuration

The Vite config at `apps/front-screen/vite.config.ts` includes the test block. The root `vitest.config.ts` uses `extends` to inherit it:

```ts
// apps/front-screen/vite.config.ts (test block excerpt)
test: {
  environment: "happy-dom",
  setupFiles: ["./tests/unit/setup.ts"],
  css: true,
}
```

`happy-dom` is used instead of `jsdom` because it is faster and sufficient for most front-screen tests. Physics calculations are done in Rapier (WebAssembly) which does not run in the test environment — physics-dependent behavior must be tested via mocking.

---

## Writing Unit Tests

### Zustand store

```ts
// tests/unit/gameStore.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import { useGameStore } from "@/stores/gameStore"

describe("gameStore", () => {
  beforeEach(() => {
    // Reset to initial state before each test to avoid leakage
    useGameStore.setState(useGameStore.getInitialState())
  })

  it("starts with score 0 and 3 balls", () => {
    const state = useGameStore.getState()
    expect(state.score).toBe(0)
    expect(state.ballsLeft).toBe(3)
  })

  it("updates score via setScore", () => {
    useGameStore.getState().setScore(12000)
    expect(useGameStore.getState().score).toBe(12000)
  })

  it("decrements ballsLeft on drainBall", () => {
    useGameStore.getState().drainBall()
    expect(useGameStore.getState().ballsLeft).toBe(2)
  })
})
```

### React component

```tsx
// tests/unit/BallSaverGate.test.tsx
import { describe, it, expect } from "vitest"
import { render } from "@testing-library/react"
import { BallSaverGate } from "@/components/ballSavers/BallSaverGate"
import { useGameStore } from "@/stores/gameStore"

describe("BallSaverGate", () => {
  it("renders without crashing", () => {
    // React Three Fiber components need Canvas — mock or wrap as needed
    expect(() => render(<BallSaverGate position={[0, 0, 0]} />)).not.toThrow()
  })
})
```

Three.js / R3F components often need a `<Canvas>` wrapper or mocked Three.js context. If the component is purely about state or UI logic, extract that into a hook and test the hook instead.

### Custom hook

```ts
// tests/unit/useFlipperInput.test.ts
import { describe, it, expect, vi } from "vitest"
import { renderHook, act } from "@testing-library/react"
import { usePlungerStore } from "@/stores/plungerStore"
import { useFlipperInput } from "@/hooks/useFlipperInput"

describe("useFlipperInput", () => {
  it("starts charging on Space keydown", () => {
    renderHook(() => useFlipperInput())

    act(() => {
      window.dispatchEvent(new KeyboardEvent("keydown", { code: "Space" }))
    })

    expect(usePlungerStore.getState().isCharging).toBe(true)
  })

  it("releases on Space keyup", () => {
    renderHook(() => useFlipperInput())

    act(() => {
      window.dispatchEvent(new KeyboardEvent("keydown", { code: "Space" }))
      window.dispatchEvent(new KeyboardEvent("keyup", { code: "Space" }))
    })

    expect(usePlungerStore.getState().isCharging).toBe(false)
  })
})
```

### Mocking Three.js and Rapier

Three.js and Rapier cannot run in happy-dom. Mock them at the top of the test file or in `setup.ts`:

```ts
// tests/unit/setup.ts
import { vi } from "vitest"

vi.mock("three", () => ({
  Mesh: class {},
  BoxGeometry: class {},
  MeshStandardMaterial: class {},
  // add what tests need
}))

vi.mock("@react-three/rapier", () => ({
  RigidBody: ({ children }: any) => children,
  CuboidCollider: () => null,
  useRapier: () => ({ world: {} }),
}))
```

### Mocking WebSocket

```ts
import { vi } from "vitest"

vi.mock("@frontend/ws", () => ({
  useGameSocket: () => ({
    lastMessage: null,
    send: vi.fn(),
    connected: false,
  }),
}))
```

---

## Writing E2E Tests (Playwright)

E2E tests run against a built and served app. They test the full browser experience, not individual components.

```ts
// tests/e2e/playfield.spec.ts
import { test, expect } from "@playwright/test"

test("canvas is visible on load", async ({ page }) => {
  await page.goto("/")
  await expect(page.locator("canvas")).toBeVisible()
})

test("score display shows 0 at start", async ({ page }) => {
  await page.goto("/")
  await expect(page.locator("[data-testid='score']")).toHaveText("00000000")
})
```

Use `data-testid` attributes to select elements in E2E tests. Do not select by CSS class names, which are auto-generated by Tailwind.

```tsx
// In your component:
<span data-testid="score">{formatScore(score)}</span>
```

---

## Adding a Test for a New Feature

When you add a new component, hook, or store action, add a test alongside it:

1. Create `tests/unit/<FeatureName>.test.ts(x)`.
2. Cover the happy path (expected behavior).
3. Cover the edge cases (zero values, empty arrays, error states).
4. For UI: assert what the user sees, not what CSS class is applied.

Example checklist for a new store action `addMultiplier`:
- It increases the multiplier by 1.
- It does not exceed `MAX_MULTIPLIER`.
- It resets to 1 on `resetMultiplier`.

---

## What Not to Test

- Three.js object world positions (non-deterministic, physics-frame-dependent)
- Exact pixel positions of 3D elements
- Rapier simulation steps (unit test the response logic, not the physics engine)
- Internal Zustand state structure (test via public actions and selectors)
- CSS class names generated by Tailwind

---

## Test File Naming

Mirror the source file:

```
src/stores/gameStore.ts            -->  tests/unit/gameStore.test.ts
src/hooks/useFlipperInput.ts       -->  tests/unit/useFlipperInput.test.ts
src/components/bumpers/Bumper.tsx  -->  tests/unit/Bumper.test.tsx
src/utils/formatScore.ts           -->  tests/unit/formatScore.test.ts
```

Test names follow the pattern: `"<verb> <what> when <condition>"`.
