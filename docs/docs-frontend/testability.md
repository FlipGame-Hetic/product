# Testability

This document explains the testing strategy for the frontend monorepo: which tools are used, how tests are structured, and how to write new tests that are maintainable and deterministic.

---

## Test Stack

| Layer | Tool | Scope |
|---|---|---|
| Unit | Vitest | Functions, hooks, stores, pure logic |
| Component | Vitest + happy-dom / jsdom | React components in isolation |
| End-to-end | Playwright | Full browser flows per app |

---

## How Tests Are Organized

Every app and relevant package follows the same layout:

```
apps/<app>/
└── tests/
    ├── unit/
    │   ├── setup.ts          # Vitest setup (mocks, globals)
    │   └── *.test.ts(x)      # Test files
    └── e2e/
        ├── README.md
        └── *.spec.ts         # Playwright specs

packages/<pkg>/
└── tests/
    └── *.test.ts
```

Test files mirror the source path they cover:

```
src/stores/gameStore.ts           -->  tests/unit/gameStore.test.ts
src/components/balls/Ball.tsx     -->  tests/unit/Ball.test.tsx
src/hooks/useFlipperInput.ts      -->  tests/unit/useFlipperInput.test.ts
```

---

## Vitest — Unit and Component Tests

### Root configuration

The root `vitest.config.ts` registers all test projects in a single runner:

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    exclude: ["**/e2e/**", "**/node_modules/**", "**/dist/**"],
    projects: [
      { extends: "apps/front-screen/vite.config.ts",  test: { root: "apps/front-screen",  include: ["apps/front-screen/tests/**/*.test.{ts,tsx}"] } },
      { extends: "apps/back-screen/vite.config.ts",   test: { root: "apps/back-screen",   include: ["apps/back-screen/tests/**/*.test.{ts,tsx}"] } },
      { extends: "apps/dmd-screen/vite.config.ts",    test: { root: "apps/dmd-screen",    include: ["apps/dmd-screen/tests/**/*.test.{ts,tsx}"] } },
      { extends: "packages/utils/vitest.config.ts",   test: { root: "packages/utils",     include: ["packages/utils/tests/**/*.test.ts"] } },
      { extends: "packages/ws/vitest.config.ts",      test: { root: "packages/ws",        include: ["packages/ws/tests/**/*.test.ts"] } },
      { extends: "packages/ui/vitest.config.ts",      test: { root: "packages/ui",        include: ["packages/ui/tests/**/*.test.{ts,tsx}"] } },
    ],
  },
})
```

Each project inherits the Vite config from its app or package, so aliases, plugins, and environment settings stay consistent between dev and test.

### DOM environments

| Project | Environment | Reason |
|---|---|---|
| front-screen | `happy-dom` | Lighter; physics tests don't need full DOM |
| back-screen | `jsdom` | Needs closer browser API compatibility |
| dmd-screen | `jsdom` | Same as back-screen |

### Running tests

```bash
# All unit tests (all projects)
pnpm test

# Watch mode (re-runs on file changes)
pnpm test:watch

# Single project
pnpm --filter @frontend/front-screen test

# Single file
pnpm test -- tests/unit/gameStore.test.ts

# With coverage
pnpm test -- --coverage
```

---

## Playwright — End-to-End Tests

E2E tests run against a fully built app served locally. Each app has its own `playwright.config.ts`.

```bash
# All apps
pnpm test:e2e

# Single app
pnpm --filter @frontend/front-screen test:e2e

# With UI (headed browser)
pnpm --filter @frontend/front-screen test:e2e -- --headed
```

Reports are saved to `apps/<app>/playwright-report/` and uploaded as CI artifacts (7-day retention).

### Writing a Playwright test

```ts
// apps/front-screen/tests/e2e/home.spec.ts
import { test, expect } from "@playwright/test"

test("canvas is rendered on load", async ({ page }) => {
  await page.goto("/")
  const canvas = page.locator("canvas")
  await expect(canvas).toBeVisible()
})
```

---

## Writing Good Unit Tests

### Pure functions — simplest case

Test inputs and outputs without any setup:

```ts
import { describe, it, expect } from "vitest"
import { formatScore } from "@frontend/utils"

describe("formatScore", () => {
  it("pads with leading zeros to 8 digits", () => {
    expect(formatScore(1234)).toBe("00001234")
  })

  it("returns all zeros for zero", () => {
    expect(formatScore(0)).toBe("00000000")
  })
})
```

### Zustand stores

Reset store state between tests to prevent leakage:

```ts
import { describe, it, expect, beforeEach } from "vitest"
import { useGameStore } from "@/stores/gameStore"

describe("gameStore", () => {
  beforeEach(() => {
    useGameStore.setState(useGameStore.getInitialState())
  })

  it("updates score", () => {
    useGameStore.getState().setScore(500)
    expect(useGameStore.getState().score).toBe(500)
  })
})
```

### React components

Use `@testing-library/react` with Vitest:

```ts
import { describe, it, expect } from "vitest"
import { render, screen } from "@testing-library/react"
import { ScoreDisplay } from "@frontend/ui"

describe("ScoreDisplay", () => {
  it("renders the formatted score", () => {
    render(<ScoreDisplay score={1234} />)
    expect(screen.getByText("00001234")).toBeInTheDocument()
  })
})
```

### Hooks

Use `renderHook` from `@testing-library/react`:

```ts
import { renderHook, act } from "@testing-library/react"
import { useTimer } from "@/hooks/useTimer"

it("increments every second", () => {
  const { result } = renderHook(() => useTimer())
  act(() => result.current.start())
  expect(result.current.elapsed).toBeGreaterThan(0)
})
```

### Mocking WebSocket / external dependencies

Use `vi.mock` to replace modules that communicate with the outside world:

```ts
import { vi, describe, it, expect } from "vitest"

vi.mock("@frontend/ws", () => ({
  useGameSocket: () => ({ send: vi.fn(), connected: false }),
}))

it("shows offline indicator when disconnected", () => {
  render(<ConnectionStatus />)
  expect(screen.getByText("Offline")).toBeInTheDocument()
})
```

Never let tests open real network connections or touch the file system.

---

## What to Test — and What Not To

### Test these

- Pure utility functions (all paths, edge cases)
- Zustand store actions and selectors
- React component rendering with given props
- Custom hooks: state transitions and returned values
- WebSocket message parsing and validation logic

### Skip these

- Implementation details: internal variable names, private methods
- Three.js object positions or physics frame values — they are non-deterministic
- Styles and exact CSS class names
- Snapshot tests for complex 3D scenes

---

## CI Integration

On every pull request, GitHub Actions runs:

```
1. pnpm test           (all Vitest projects)
2. pnpm test:e2e       (Playwright, per app, Chromium)
```

Both jobs must pass before a PR can be merged. Playwright reports are uploaded as artifacts and linked in the CI summary.

```
ci.yml
  ├── build (matrix: front-screen, back-screen, dmd-screen)
  ├── unit-tests (depends on: build)
  └── e2e-tests  (depends on: build)
```

---

## Test Naming Conventions

```
describe("<ComponentOrModule>", () => {
  it("<verb> <what> when <condition>", () => { ... })
})
```

Examples:
- `it("returns zero when input is negative")`
- `it("renders score after receiving game_update event")`
- `it("disconnects on unmount")`
- `it("does not render overlay when game state is idle")`

Avoid vague names like `it("works")` or `it("test 1")`. A failing test name should tell you exactly what broke without reading the test body.
