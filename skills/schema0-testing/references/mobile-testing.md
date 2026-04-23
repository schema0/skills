# Mobile Testing

Skip if `apps/mobile/` does not exist.

## Quick Start

```bash
# Step 1: Build compiled @orpc packages (required once, and after @orpc upgrades)
schema0 sandbox exec "node mobile/build-orpc.js" --cwd packages/test

# Step 2: Generate migrations (if schema changed)
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test

# Step 3: Run tests
schema0 sandbox exec "NODE_ENV=test NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.js --forceExit mobile/{entity}.test.tsx" --cwd packages/test --timeout 120000
```

## Pre-Test Checklist

### 1. Router MUST use `createDb()` -- NOT `fetchCustomResources`

Same rule as web: verify the router uses `createDb()` from `@template/db`. `fetchCustomResources` will cause ECONNREFUSED.

### 2. Database migrations must be generated in `packages/test`

```bash
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test
```

## Test Infrastructure

| File                      | Purpose                                                                                                            |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `jest.config.js`          | Jest config: merges `jest-expo` preset, maps `@/` to `apps/mobile/`, redirects `@orpc/*` to compiled CJS           |
| `babel.config.js`         | `babel-preset-expo` with `unstable_transformImportMeta` + `@babel/plugin-transform-dynamic-import`                 |
| `mobile/setup-globals.js` | Polyfills `window.location` for PGlite's Emscripten, patches `ArrayBuffer[Symbol.hasInstance]` for cross-VM compat |
| `mobile/setup.ts`         | Jest mocks: `@template/auth`, `@template/db`, `expo-router`, `@schema0/auth-mobile`, expo modules, RN safe area    |
| `mobile/build-orpc.js`    | Pre-compiles `@orpc/*` from ESM (.mjs) to CJS (.js) via esbuild                                                    |
| `mobile/compiled-orpc/`   | Pre-compiled CJS bundles of `@orpc/server`, `@orpc/client`, `@orpc/tanstack-query`                                 |

## Key Differences from Web Tests

- **Jest + jest-expo** instead of bun:test + HappyDOM
- **`@testing-library/react-native`** instead of `@testing-library/react` -- uses `fireEvent.press`/`fireEvent.changeText` instead of `fireEvent.click`/`fireEvent.change`
- **`--experimental-vm-modules`** required for PGlite's `import()` calls
- **`mockQueryClient` must be created inside `jest.mock` factory** -- ESM mode hoists `jest.mock` above `const` declarations
- **No collections** -- mobile uses `useQuery`/`useMutation` directly
- **No `optimistic: false` workaround** -- no TanStack DB collections
- **Each test renders fresh** -- `@testing-library/react-native` doesn't support shared renders across tests
- **NEVER use `e.stopPropagation()` in `onPress` handlers** -- `fireEvent.press` in `@testing-library/react-native` doesn't pass a real event object, causing `TypeError: e.stopPropagation is not a function`

## `jest.mock` Factory Hoisting in ESM Mode

With `--experimental-vm-modules`, `jest.mock` factories are hoisted above `const` declarations. Variables defined outside the factory are `undefined` when the factory runs:

```typescript
// BROKEN -- mockQueryClient is undefined inside factory
const mockQueryClient = new QueryClient({...});
jest.mock("@/src/utils/query-client", () => ({
  queryClient: mockQueryClient, // <- undefined!
}));

// WORKING -- create inside factory, assign to outer let
let mockQueryClient;
jest.mock("@/src/utils/query-client", () => {
  const { QueryClient: QC } = require("@tanstack/react-query");
  const qc = new QC({...});
  mockQueryClient = qc;
  return { queryClient: qc };
});
```

## App-Specific Mocks

`mobile/setup.ts` only mocks modules that every mobile test needs. Your screen will likely import app-specific modules not covered by setup.ts (e.g., custom contexts, additional expo packages, `@gorhom/bottom-sheet`, `react-native-gesture-handler`). You MUST add `jest.mock()` for these in your test file, not in setup.ts.

## `Alert.alert` in Delete Flows

If the screen uses `Alert.alert` for delete confirmation, `@testing-library/react-native` cannot interact with it (it's a mobile dialog). Override `Alert.alert` at runtime in the delete test to auto-confirm:

```typescript
import { Alert } from "react-native";

test("delete via UI", async () => {
  const originalAlert = Alert.alert;
  Alert.alert = ((
    _title: string,
    _msg?: string,
    buttons?: Array<{ onPress?: () => void }>,
  ) => {
    const confirmButton = buttons?.[buttons.length - 1];
    if (confirmButton?.onPress) confirmButton.onPress();
  }) as typeof Alert.alert;

  // ... render, interact, assert ...

  Alert.alert = originalAlert;
});
```

## Required Test Structure

Every mobile test MUST:

1. Render the screen with `render(<Screen />, { wrapper: Wrapper })`
2. Interact via UI -- `fireEvent.press`, `fireEvent.changeText`
3. Verify on screen -- `await waitFor(() => screen.getByText("..."))` or `expect(screen.queryByText("...")).toBeNull()`
4. Verify in PGlite -- `await db.select().from(entity)` assertions
5. Verify ORPC Layer 3 -- spy on `["~orpc"].inputSchema["~standard"].validate`

## Full Mobile Test Template

The mock factory (`jest.mock("@/src/utils/query-client", ...)`) and the `let mockQueryClient` declaration are **fixed infrastructure — copy them verbatim**. The only entity-specific parts are in the test body: `db.delete({entity})`, `render(<{Entity}Screen ...>)`, `fireEvent.*`, and `screen.get*` / `screen.queryBy*` calls.

```typescript
/// <reference types="@types/jest" />

import React from "react";
import {
  render,
  screen,
  fireEvent,
  waitFor,
  act,
} from "@testing-library/react-native";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { db, initializeTestDatabase } from "../db";
import { {entity} } from "@template/db/schema";
import { {entity}Router } from "@template/api/routers/{entity}";
import { jest } from "@jest/globals";

// CRITICAL: mockQueryClient MUST be created inside the jest.mock factory.
let mockQueryClient: QueryClient;

jest.mock("@/src/utils/query-client", () => {
  const { QueryClient: QC } = require("@tanstack/react-query");
  const { createRouterClient } = require("@orpc/server");
  const { createTanstackQueryUtils } = require("@orpc/tanstack-query");
  const { appRouter } = require("@template/api/routers/index"); // always appRouter — never the entity router

  const qc = new QC({
    defaultOptions: {
      queries: { retry: false, staleTime: 0 },
      mutations: { retry: false },
    },
  });

  const mockContext = {
    session: { user: { id: "user_mobile_123", email: "mobile@test.com" } },
    request: new Request("https://example.com"),
  };

  const mockServerClient = createRouterClient(appRouter, {
    context: mockContext, // required — auth middleware reads context.session
  });

  mockQueryClient = qc;

  return {
    client: mockServerClient,
    queryClient: qc,
    orpc: createTanstackQueryUtils(mockServerClient),
  };
});

// App-specific mocks: add jest.mock() for any modules your screen imports
// that aren't covered by mobile/setup.ts

// Import the screen AFTER all mocks are set up
import {Entity}Screen from "@/app/(tabs)/{entity}";

const Wrapper = ({ children }: { children: React.ReactNode }) => (
  <QueryClientProvider client={mockQueryClient}>{children}</QueryClientProvider>
);

beforeAll(async () => {
  await initializeTestDatabase();
});

beforeEach(async () => {
  await db.delete({entity});
  mockQueryClient.clear();
});

describe("{Entity}Screen - full CRUD via mobile UI", () => {
  test("create via UI triggers insertMany endpoint", async () => {
    const inputValidationSpy = jest.spyOn(
      (
        {entity}Router.insertMany as unknown as {
          "~orpc": {
            inputSchema: { "~standard": { validate: () => unknown } };
          };
        }
      )["~orpc"].inputSchema["~standard"],
      "validate",
    );

    render(<{Entity}Screen />, { wrapper: Wrapper });
    await waitFor(() => screen.getByLabelText("Title"));

    fireEvent.changeText(screen.getByLabelText("Title"), "Test Item");
    fireEvent.press(screen.getByLabelText("Create {Entity}"));

    await waitFor(() => screen.getByText("Test Item"), { timeout: 10000 });

    // Verify data persisted in PGlite
    const rows = await db.select().from({entity});
    expect(rows).toHaveLength(1);
    expect(rows[0]?.title).toBe("Test Item");

    // Layer 3: ORPC input validation passed
    const validationResult = await inputValidationSpy.mock.results[0]?.value;
    expect(
      (validationResult as { issues?: unknown[] } | undefined)?.issues,
    ).toBeUndefined();
    inputValidationSpy.mockRestore();
  });

  test("update via UI triggers updateMany endpoint", async () => {
    // Seed a record
    await db.insert({entity}).values({
      id: "{entity}_upd_1",
      title: "Original Title",
      userId: "user_mobile_123",
    });

    const updateInputSpy = jest.spyOn(
      (
        {entity}Router.updateMany as unknown as {
          "~orpc": {
            inputSchema: { "~standard": { validate: () => unknown } };
          };
        }
      )["~orpc"].inputSchema["~standard"],
      "validate",
    );

    render(<{Entity}Screen />, { wrapper: Wrapper });
    await waitFor(() => screen.getByText("Original Title"), { timeout: 10000 });

    fireEvent.press(screen.getByLabelText("Edit"));
    await waitFor(() => screen.getByLabelText("Edit Title"));

    fireEvent.changeText(screen.getByLabelText("Edit Title"), "Updated Title");
    fireEvent.press(screen.getByLabelText("Save"));

    await waitFor(() => screen.getByText("Updated Title"), { timeout: 10000 });

    // Verify in DB
    const rows = await db.select().from({entity});
    expect(rows[0]?.title).toBe("Updated Title");

    // Layer 3: ORPC input validation passed
    const result = await updateInputSpy.mock.results[0]?.value;
    expect(
      (result as { issues?: unknown[] } | undefined)?.issues,
    ).toBeUndefined();
    updateInputSpy.mockRestore();
  });

  test("delete via UI triggers deleteMany endpoint", async () => {
    // Seed a record
    await db.insert({entity}).values({
      id: "{entity}_del_1",
      title: "To Be Deleted",
      userId: "user_mobile_123",
    });

    const deleteInputSpy = jest.spyOn(
      (
        {entity}Router.deleteMany as unknown as {
          "~orpc": {
            inputSchema: { "~standard": { validate: () => unknown } };
          };
        }
      )["~orpc"].inputSchema["~standard"],
      "validate",
    );

    render(<{Entity}Screen />, { wrapper: Wrapper });
    await waitFor(() => screen.getByText("To Be Deleted"), { timeout: 5000 });

    fireEvent.press(screen.getByLabelText("Delete"));

    await waitFor(() => {
      expect(screen.queryByText("To Be Deleted")).toBeNull();
    }, { timeout: 10000 });

    // Verify data removed from PGlite
    const rows = await db.select().from({entity});
    expect(rows).toHaveLength(0);

    // Layer 3: ORPC input validation passed
    const result = await deleteInputSpy.mock.results[0]?.value;
    expect(
      (result as { issues?: unknown[] } | undefined)?.issues,
    ).toBeUndefined();
    deleteInputSpy.mockRestore();
  });
});
```
