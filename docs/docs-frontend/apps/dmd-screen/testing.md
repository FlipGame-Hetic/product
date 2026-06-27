# dmd-screen — Testing

This document explains how the test suite for `dmd-screen` works and how to write new tests for scenes, combo payload parsing, icon rendering, and store actions.

---

## Test Setup

| Tool | Purpose |
|---|---|
| Vitest | Unit and component tests |
| jsdom | DOM environment |
| @testing-library/react | Component rendering |
| Playwright | End-to-end browser tests |

---

## Test Structure

```
apps/dmd-screen/
└── tests/
    ├── unit/
    │   ├── setup.ts              # Global mocks and DOM setup
    │   ├── appGameOver.test.ts   # Game over scene logic
    │   ├── comboPayload.test.ts  # Payload validation functions
    │   └── icons.test.tsx        # Icon component rendering
    └── e2e/
        └── *.spec.ts
```

---

## Running Tests

```bash
# All unit tests for dmd-screen
pnpm --filter @frontend/dmd-screen test

# Watch mode
pnpm --filter @frontend/dmd-screen test:watch

# Single file
pnpm --filter @frontend/dmd-screen test -- tests/unit/comboPayload.test.ts

# E2E
pnpm --filter @frontend/dmd-screen test:e2e

# All projects from root
pnpm test
```

---

## Writing Unit Tests

### Combo payload parser (pure function — ideal for unit tests)

This is the most important function to test thoroughly because bad payloads from the server can crash the display:

```ts
// tests/unit/comboPayload.test.ts
import { describe, it, expect } from "vitest"
import { parseComboPayload } from "@/dmd/comboPayload"

describe("parseComboPayload", () => {
  it("parses a valid payload", () => {
    const raw = { type: "multiplier", label: "x3", value: 3 }
    const result = parseComboPayload(raw)
    expect(result).toEqual({ type: "multiplier", label: "x3", value: 3 })
  })

  it("throws when type is missing", () => {
    expect(() => parseComboPayload({ label: "x3", value: 3 })).toThrow()
  })

  it("throws when value is not a number", () => {
    expect(() => parseComboPayload({ type: "chain", label: "x2", value: "3" })).toThrow()
  })

  it("throws for null input", () => {
    expect(() => parseComboPayload(null)).toThrow()
  })

  it("throws for non-object input", () => {
    expect(() => parseComboPayload("invalid")).toThrow()
  })
})
```

### Game over scene

```ts
// tests/unit/appGameOver.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import { render, screen } from "@testing-library/react"
import { useDmdStore } from "@/stores/dmdStore"
import { GameOver } from "@/dmd/scenes/GameOver"

describe("GameOver scene", () => {
  it("renders GAME OVER text", () => {
    render(<GameOver players={[]} finalScore={0} />)
    expect(screen.getByText("GAME OVER")).toBeInTheDocument()
  })

  it("displays formatted final score", () => {
    render(<GameOver players={[]} finalScore={12345} />)
    expect(screen.getByText("00012345")).toBeInTheDocument()
  })

  it("lists all player names", () => {
    const players = [
      { id: "1", name: "Alice", score: 9000 },
      { id: "2", name: "Bob",   score: 4500 },
    ]
    render(<GameOver players={players} finalScore={9000} />)
    expect(screen.getByText("Alice")).toBeInTheDocument()
    expect(screen.getByText("Bob")).toBeInTheDocument()
  })
})
```

### Icon component

```tsx
// tests/unit/icons.test.tsx
import { describe, it, expect } from "vitest"
import { render, screen } from "@testing-library/react"
import { BallIcon } from "@/components/icons/BallIcon"

describe("BallIcon", () => {
  it("renders without crashing", () => {
    render(<BallIcon />)
    expect(screen.getByRole("img", { hidden: true })).toBeInTheDocument()
  })

  it("applies custom class", () => {
    const { container } = render(<BallIcon className="my-class" />)
    expect(container.firstChild).toHaveClass("my-class")
  })
})
```

### Store actions

```ts
import { describe, it, expect, beforeEach } from "vitest"
import { useDmdStore } from "@/stores/dmdStore"

describe("dmdStore", () => {
  beforeEach(() => {
    useDmdStore.setState(useDmdStore.getInitialState())
  })

  it("starts on the idle scene", () => {
    expect(useDmdStore.getState().currentScene).toBe("idle")
  })

  it("updates scene on setScene", () => {
    useDmdStore.getState().setScene("game_over")
    expect(useDmdStore.getState().currentScene).toBe("game_over")
  })

  it("resets all state on reset", () => {
    useDmdStore.getState().setScore(9999)
    useDmdStore.getState().setScene("game_over")
    useDmdStore.getState().reset()
    const state = useDmdStore.getState()
    expect(state.score).toBe(0)
    expect(state.currentScene).toBe("idle")
  })
})
```

---

## Writing E2E Tests

```ts
// tests/e2e/dmd.spec.ts
import { test, expect } from "@playwright/test"

test("shows attract scene on load", async ({ page }) => {
  await page.goto("/")
  await expect(page.locator("[data-testid='dmd-attract']")).toBeVisible()
})

test("shows game over after game_over event", async ({ page }) => {
  await page.goto("/")
  // Simulate event via test API or mocked WebSocket
  await expect(page.locator("[data-testid='dmd-game-over']")).toBeVisible()
})
```

Add `data-testid` attributes to top-level scene containers:

```tsx
export function GameOver(props: GameOverProps) {
  return (
    <div data-testid="dmd-game-over" className="dmd-scene game-over">
      ...
    </div>
  )
}
```

---

## Adding Tests for a New Scene

When you add a new DMD scene:

1. Create `tests/unit/<SceneName>.test.tsx`.
2. Test that it renders without crashing with minimal props.
3. Test each prop: correct output for different score values, different player lists, different labels.
4. If the scene has a timed animation (blink, flash), test that the CSS class is applied, not that the animation plays.

When you add a new combo type to `parseComboPayload`:

1. Add a test for the valid case.
2. Add a test for each missing required field.
3. Add a test for wrong types (e.g., string where number is expected).

---

## What Not to Test

- CSS animation timing (cannot be reliably measured in jsdom)
- Pixel-exact font rendering
- The `window.__ENV__` config injection (covered by Docker integration)

---

## Test Naming

```
describe("<ComponentOrModule>")
  it("<verb> <what> when <condition>")
```

Examples:
- `"throws when type is missing from payload"`
- `"renders JACKPOT text when scene is jackpot"`
- `"resets score to 0 on reset"`
