# back-screen — Testing

This document explains how the test suite for `back-screen` works and how to write tests for scenes, menu logic, boss config, and stores.

---

## Test Setup

| Tool | Purpose |
|---|---|
| Vitest | Unit and component tests |
| jsdom | DOM environment |
| @testing-library/react | Component rendering and interaction |
| Playwright | End-to-end browser tests |

The DOM environment for back-screen is `jsdom` (not `happy-dom`) because the app needs closer browser API compatibility for scene transitions and audio mocking.

---

## Test Structure

```
apps/back-screen/
└── tests/
    ├── unit/
    │   ├── setup.ts                     # Global mocks and test setup
    │   ├── bossConfig.test.ts           # Boss definition integrity
    │   ├── Leaderboard.test.tsx         # Leaderboard component
    │   ├── menuActions.test.ts          # Menu navigation pure functions
    │   ├── menuSound.test.ts            # Menu sound trigger logic
    │   ├── useBackScreenStore.test.ts   # Main store actions
    │   └── useScreenHubClient.test.ts   # WebSocket client hook
    └── e2e/
        └── *.spec.ts
```

---

## Running Tests

```bash
# All unit tests for back-screen
pnpm --filter @frontend/back-screen test

# Watch mode
pnpm --filter @frontend/back-screen test:watch

# Specific test file
pnpm --filter @frontend/back-screen test -- tests/unit/menuActions.test.ts

# E2E tests
pnpm --filter @frontend/back-screen test:e2e

# From root (all projects)
pnpm test
```

---

## Writing Unit Tests

### Menu actions (pure functions — easiest to test)

Menu navigation functions take state and return new state. No mocks needed:

```ts
// tests/unit/menuActions.test.ts
import { describe, it, expect } from "vitest"
import { navigateDown, navigateUp, getSelectedItem } from "@/menu/menuActions"

const baseState = {
  items: [
    { id: "start",   label: "Start",   action: "start_game" },
    { id: "options", label: "Options", action: "open_options" },
    { id: "credits", label: "Credits", action: "show_credits" },
  ],
  selectedIndex: 0,
}

describe("navigateDown", () => {
  it("moves selection to next item", () => {
    const next = navigateDown(baseState)
    expect(next.selectedIndex).toBe(1)
  })

  it("wraps around to first item at the end", () => {
    const next = navigateDown({ ...baseState, selectedIndex: 2 })
    expect(next.selectedIndex).toBe(0)
  })
})

describe("navigateUp", () => {
  it("wraps around to last item at the beginning", () => {
    const next = navigateUp(baseState)
    expect(next.selectedIndex).toBe(2)
  })
})
```

### Boss config integrity

Verify that every boss has all required fields:

```ts
// tests/unit/bossConfig.test.ts
import { describe, it, expect } from "vitest"
import { bossConfig } from "@/boss/bossConfig"

describe("bossConfig", () => {
  const bosses = Object.values(bossConfig)

  it("all bosses have an id, name, and warningDuration", () => {
    bosses.forEach((boss) => {
      expect(boss.id).toBeTruthy()
      expect(boss.name).toBeTruthy()
      expect(boss.warningDuration).toBeGreaterThan(0)
    })
  })

  it("all bosses have valid asset paths", () => {
    bosses.forEach((boss) => {
      expect(boss.assets.music).toMatch(/^\//)   // starts with /
    })
  })
})
```

### Zustand store

```ts
// tests/unit/useBackScreenStore.test.ts
import { describe, it, expect, beforeEach } from "vitest"
import { useBackScreenStore } from "@/stores/backScreenStore"

describe("backScreenStore", () => {
  beforeEach(() => {
    useBackScreenStore.setState(useBackScreenStore.getInitialState())
  })

  it("starts on the idle scene", () => {
    expect(useBackScreenStore.getState().currentScene).toBe("idle")
  })

  it("transitions to menu scene", () => {
    useBackScreenStore.getState().setScene("menu")
    expect(useBackScreenStore.getState().currentScene).toBe("menu")
  })

  it("updates players list", () => {
    const players = [{ id: "1", name: "Alice", score: 5000 }]
    useBackScreenStore.getState().setPlayers(players)
    expect(useBackScreenStore.getState().players).toEqual(players)
  })
})
```

### React component

```tsx
// tests/unit/Leaderboard.test.tsx
import { describe, it, expect } from "vitest"
import { render, screen } from "@testing-library/react"
import { Leaderboard } from "@/scenes/Leaderboard"

describe("Leaderboard", () => {
  it("renders all player names", () => {
    const players = [
      { id: "1", name: "Alice", score: 9000 },
      { id: "2", name: "Bob",   score: 4500 },
    ]
    render(<Leaderboard players={players} />)
    expect(screen.getByText("Alice")).toBeInTheDocument()
    expect(screen.getByText("Bob")).toBeInTheDocument()
  })

  it("displays players in descending score order", () => {
    const players = [
      { id: "1", name: "Bob",   score: 4500 },
      { id: "2", name: "Alice", score: 9000 },
    ]
    render(<Leaderboard players={players} />)
    const names = screen.getAllByTestId("player-name").map((el) => el.textContent)
    expect(names).toEqual(["Alice", "Bob"])
  })
})
```

### Screen-hub hook

Mock the WebSocket connection:

```ts
// tests/unit/useScreenHubClient.test.ts
import { describe, it, expect, vi } from "vitest"
import { renderHook, act } from "@testing-library/react"
import { useBackScreenStore } from "@/stores/backScreenStore"

vi.mock("@frontend/ws", () => ({
  useScreenHub: vi.fn(() => ({ lastEvent: null, send: vi.fn() })),
}))

import { useScreenHubClient } from "@/hooks/useScreenHubClient"
import { useScreenHub } from "@frontend/ws"

describe("useScreenHubClient", () => {
  it("transitions scene on scene_change event", () => {
    const mockUseScreenHub = vi.mocked(useScreenHub)
    mockUseScreenHub.mockReturnValue({
      lastEvent: { type: "scene_change", payload: { scene: "menu" } },
      send: vi.fn(),
    })

    renderHook(() => useScreenHubClient())
    expect(useBackScreenStore.getState().currentScene).toBe("menu")
  })
})
```

---

## Writing E2E Tests

```ts
// tests/e2e/leaderboard.spec.ts
import { test, expect } from "@playwright/test"

test("shows leaderboard on load", async ({ page }) => {
  await page.goto("/")
  await expect(page.locator("[data-testid='leaderboard']")).toBeVisible()
})

test("shows boss warning overlay", async ({ page }) => {
  await page.goto("/")
  // Trigger via API or mock event
  // ...
  await expect(page.locator("[data-testid='boss-warning']")).toBeVisible()
})
```

Use `data-testid` attributes in components to make E2E selectors stable:

```tsx
<div data-testid="leaderboard">...</div>
<div data-testid="boss-warning">...</div>
```

---

## Adding Tests for a New Feature

When you add a new scene, boss, or store action:

1. For a new pure function (menu action, score formatter): add a test file with all edge cases.
2. For a new store action: test it via `store.getState().action()` and assert the resulting state.
3. For a new component: render it with `@testing-library/react` and assert visible text or `data-testid` elements.
4. For a new event handler (screen-hub): mock `useScreenHub` and verify the store state after the event.

---

## Test Naming

```
describe("<SceneOrModule>")
  it("<verb> <what> when <condition>")
```

Good names:
- `"renders player names in score order"`
- `"transitions to boss_warning scene on boss_incoming event"`
- `"navigateDown wraps around when at last item"`

Bad names:
- `"works"`, `"test 1"`, `"should do stuff"`
