# Web Testing

Skip if `apps/web/` does not exist.

## Quick Start

```bash
# Step 1: Generate migrations (if schema changed)
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test

# Step 2: Run tests
schema0 sandbox exec "bun test web/{entity}.test.tsx" --cwd packages/test
```

## Pre-Test Checklist

Before running a test, verify the following or the test will fail in ways that are hard to diagnose.

### 1. Router MUST use `createDb()` -- NOT `fetchCustomResources`

Verify the router for your entity uses `createDb()` from `@template/db`. If the router imports `fetchCustomResources`, the test WILL fail with ECONNREFUSED because no HTTP backend runs during tests.

```typescript
// CORRECT -- router uses createDb(), works with real PGlite in tests
import { createDb } from "@template/db";
const db = createDb();
return await db.select().from(entities);

// WRONG -- router uses fetchCustomResources (HTTP); gets ECONNREFUSED in tests
import { fetchCustomResources } from "./utils";
const res = await fetchCustomResources("entities", { method: "GET" });
```

`fetchCustomResources` is ONLY allowed in `files.ts` and `users.ts`. If you see it in any other router, fix the router first -- do NOT work around it by mocking the client.

### 2. Collection `queryFn` must use direct server calls -- NOT HTTP requests

The collection's `queryFn` must call the server client directly. Custom entity collections must use `client.entity.selectAll()`. The built-in `files` and `users` collections use `fetchCustomResources` (an external HTTP service) and will get `ECONNREFUSED`.

```typescript
// CORRECT -- direct server call via mocked oRPC client, works in tests
queryFn: async (ctx) => {
  return await client.entity.selectAll({});
},

// WRONG -- client.files/users.selectAll calls fetchCustomResources (HTTP)
queryFn: async (ctx) => {
  return await client.files.selectAll({});
},
```

### 3. Database migrations must be generated in `packages/test`

```bash
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test
```

### 4. Collection `id` and `queryKey` must match the test's `invalidateQueries` call

```typescript
// collection
queryCollectionOptions({ id: "entity", queryKey: ["entity"], ... })

// test
void testQueryClient.invalidateQueries({ queryKey: ["entity"] });
```

### 5. NEVER use `.toBeInTheDocument()` or any jest-dom matchers

`@testing-library/jest-dom` is NOT installed. Tests run with bun:test + HappyDOM.

```typescript
// WRONG
await waitFor(() => {
  expect(screen.getByText("Test Company")).toBeInTheDocument();
});

// CORRECT -- getByText/getByRole throw if not found; use inside waitFor
await waitFor(() => screen.getByText("Test Company"), { timeout: 5000 });

// CORRECT -- if you need an assertion, use toBeDefined()
const el = screen.getByText("Test Company");
expect(el).toBeDefined();
```

## Test Infrastructure

| File              | Purpose                                                                          |
| ----------------- | -------------------------------------------------------------------------------- |
| `db.ts`           | Exports `db` (PGlite), `client`, `initializeTestDatabase`. Reads from `drizzle/` |
| `bunfig.toml`     | Tells Bun to preload `./web/happydom.ts` and `./web/setup.ts` for every web test |
| `web/happydom.ts` | Preload: registers HappyDOM globals, mocks `@template/auth` before any import    |
| `web/setup.ts`    | Configures `@testing-library/dom` error messages for cleaner test output         |

## Module Mock Ordering (Critical)

Bun's module mocking is hoisted -- `mock.module()` calls must come before the imports they intercept. The required order is:

```typescript
// 1. Mock the database -- BEFORE importing the router
void mock.module("@template/db", () => ({ createDb: () => db }));

// 2. Mock auth -- BEFORE importing the router
void mock.module("@template/auth", () => ({ auth: {}, env: {} }));

// 3. Import the router AFTER mocks -- it now uses real PGlite
import { appRouter } from "@template/api/routers/index";

// 4. Create the server client and QueryClient
const serverClient = createRouterClient(appRouter, { context: mockContext as any });
const testQueryClient = new QueryClient({ defaultOptions: { queries: { retry: false, staleTime: 0 } } });

// 5. Mock @/utils/orpc -- the collection captures `client` and `queryClient` at import time
void mock.module("@/utils/orpc", () => ({
  client: serverClient,
  queryClient: testQueryClient,
  orpc: {},
}));

// 6. Import collections and page components LAST
import { {entity}Collection } from "@/query-collections/custom/{entity}";
import EntityPage from "@/routes/_auth.entities";
import { {entity}Router } from "@template/api/routers/{entity}";
```

`mock.module()` returns a Promise -- always prefix with `void` to suppress floating promise warnings.

## 3-Layer Validation Model

Tests verify data correctness at three boundaries:

| Layer                   | What it validates                             | Spy target                                                                                          | Pass means                | Fail means                                 |
| ----------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------- | ------------------------------------------ |
| **Layer 1: Form**       | Form data conforms to zodResolver schema      | If no `"Form validation errors:"` in console, it passed                                             | Data matches form schema  | zodResolver rejected, onSubmit never fires |
| **Layer 2: Collection** | Inserted object matches collection's `schema` | `spyOn((collection as any)._mutations, "validateData")` -- assert no issues                          | Data valid for collection | onInsert/onUpdate/onDelete never called    |
| **Layer 3: ORPC**       | Input matches router's `.input()` schema      | `spyOn((router.procedure as any)["~orpc"].inputSchema["~standard"], "validate")` -- assert no issues | Data valid for endpoint   | Handler never runs                         |

## Why `insert` Requires `optimistic: false`

When `collection.insert()` is called with `optimistic: true` (the default), TanStack DB synchronously notifies `useLiveQuery` which triggers React's `setState()` via `MessageChannel.postMessage()`. In HappyDOM, the `MessageChannel` message delivery is tied to HappyDOM's own async task queue. When `fireEvent.click` triggers an async form submit handler, the event loop enters a microtask chain and the `MessageChannel` message is never delivered -- deadlock.

The fix: intercept `collection.insert` and force `optimistic: false`:

```typescript
const originalInsert = {entity}Collection.insert.bind({entity}Collection);
const insertSpy = spyOn({entity}Collection, "insert").mockImplementation(
  (...args: any[]) => originalInsert(args[0], { ...args[1], optimistic: false }),
);
```

This is non-negotiable for insert tests. Do NOT apply it to update or delete -- they don't hang.

## Spying on oRPC Internals via `["~orpc"]`

`createRouterClient` returns a Proxy that creates a new object on every property access. You cannot spy on `serverClient.entity.insertMany` -- the spy wraps a different object than the one called. Instead, access `["~orpc"]` on the stable router object:

```typescript
const inputValidationSpy = spyOn(
  ({entity}Router.insertMany as any)["~orpc"].inputSchema["~standard"],
  "validate",
);

// Assert: no validation issues
const inputResult = await inputValidationSpy.mock.results[0]?.value;
expect(inputResult?.issues).toBeUndefined();
inputValidationSpy.mockRestore();
```

## `spyOn` Method Names Must Match the Collection

The method name in `spyOn` must exactly match the method the component calls:

```typescript
// Collection has `insert` -> component calls `collection.insert([...])` -> spy matches
const insertSpy = spyOn(entityCollection, "insert");

// Collection has `update` -> component calls `collection.update(id, changes)` -> spy matches
const updateSpy = spyOn(entityCollection, "update");
```

## Required: `debugTexts()` at the Start of Every Test

Every test MUST call `debugTexts()` as its first action. This logs all visible DOM text for debugging failures.

```typescript
function debugTexts(separator = " | ") {
  const elements = screen.getAllByText(/./i);
  const texts = elements.map((el) => el.textContent?.trim()).filter(Boolean);
  console.log(texts.join(separator));
}
```

## Required Test Flow -- No Shortcuts

Every test MUST exercise the full chain:

```
fireEvent (UI) -> collection.insert/update/delete -> router endpoint -> real PGlite
```

Do NOT call the router or server client directly as a substitute for the UI test. Do NOT remove a failing UI test and replace it with a direct DB/router test.

### Forbidden shortcuts

```typescript
// WRONG -- skips the UI and collection entirely
const insertResult = await serverClient.phonebook.insertMany({ ... });

// WRONG -- skips form submission
await phonebookCollection.insert([{ id: "x", name: "y", ... }]);
```

### Escape hatch

If CRUD tests fail after 3 fix attempts and you cannot identify the root cause, STOP and ask the user for help. Do NOT continue looping. Do NOT gut the tests.

## Full Web Test Template

```typescript
import {
  describe,
  test,
  expect,
  mock,
  beforeAll,
  beforeEach,
  afterAll,
  spyOn,
} from "bun:test";
import { createRouterClient } from "@orpc/server";
import "@testing-library/react/pure";
import {
  render,
  screen,
  fireEvent,
  waitFor,
  cleanup,
  act,
} from "@testing-library/react/pure";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { db, initializeTestDatabase } from "./index";
import React from "react";

// --- Mock ordering (see "Module Mock Ordering" above) ---
void mock.module("@template/db", () => ({ createDb: () => db }));
void mock.module("@template/auth", () => ({ auth: {}, env: {} }));

import { appRouter } from "@template/api/routers/index";

const mockContext = {
  session: { user: { id: "user_123456", email: "test@example.com" } },
  request: new Request("https://example.com"),
};
const serverClient = createRouterClient(appRouter, { context: mockContext as any });

const testQueryClient = new QueryClient({
  defaultOptions: { queries: { retry: false, staleTime: 0 } },
});

void mock.module("@/utils/orpc", () => ({
  client: serverClient,
  queryClient: testQueryClient,
  orpc: {},
}));

import { {entity}Collection } from "@/query-collections/custom/{entity}";
import { BrowserRouter } from "react-router";
// DEFAULT import for page -- no curly braces (route uses `export default function`)
import EntityPage, { loader as entitiesLoader } from "@/routes/_auth.entities";
import { {entity}Router } from "@template/api/routers/{entity}";

const Providers = ({ children }: { children: React.ReactNode }) => (
  <QueryClientProvider client={testQueryClient}>
    <BrowserRouter>{children}</BrowserRouter>
  </QueryClientProvider>
);

function debugTexts(separator = " | ") {
  const elements = screen.getAllByText(/./i);
  const texts = elements.map((el) => el.textContent?.trim()).filter(Boolean);
  console.log(texts.join(separator));
}

beforeAll(async () => {
  await initializeTestDatabase();
});

beforeEach(async () => {
  void testQueryClient.invalidateQueries({ queryKey: ["{entity}"] });
});

describe("{Entity}Page - full CRUD via UI", () => {
  // Render once and share across all tests in this describe block.
  // Tests run sequentially: create -> update -> delete.
  beforeAll(async () => {
    await act(async () => {
      render(
        <Providers>
          <{Entity}Page loaderData={{ user: { id: "user_123456" } } as any} />
        </Providers>,
      );
    });
  });

  afterAll(async () => {
    cleanup();
  });

  test("{entity} route exports a loader", () => {
    expect(typeof {entity}Loader).toBe("function");
  });

  test("create via UI triggers insertMany endpoint", async () => {
    debugTexts();
    // REQUIRED: wrap insert with optimistic: false to prevent HappyDOM hang
    const originalInsert = {entity}Collection.insert.bind({entity}Collection);
    const insertSpy = spyOn({entity}Collection, "insert").mockImplementation(
      (...args: any[]) => originalInsert(args[0], { ...args[1], optimistic: false }),
    );

    // LAYER 2: Collection schema validation
    const dataValidationSpy = spyOn(
      ({entity}Collection as any)._mutations,
      "validateData",
    );

    // LAYER 3: ORPC input schema validation
    const inputValidationSpy = spyOn(
      ({entity}Router.insertMany as any)["~orpc"].inputSchema["~standard"],
      "validate",
    );

    // Open the create dialog
    await waitFor(() => screen.getByRole("button", { name: /add {entity}/i }));
    fireEvent.click(screen.getByRole("button", { name: /add {entity}/i }));
    await waitFor(() => screen.getByLabelText(/name/i));

    // Fill required fields and submit
    fireEvent.change(screen.getByLabelText(/name/i), { target: { value: "Test Item" } });
    fireEvent.click(screen.getByRole("button", { name: /^create$/i }));

    // Assert: collection.insert was called
    await waitFor(() => expect(insertSpy).toHaveBeenCalled(), { timeout: 3000 });

    // Assert no insert error
    expect(insertSpy.mock.results[0]?.value?.error).toBeUndefined();

    // LAYER 2: assert collection schema validation passed
    expect(dataValidationSpy.mock.results[0]?.value?.issues).toBeUndefined();
    dataValidationSpy.mockRestore();

    const insertedItem = insertSpy.mock.calls[0]?.[0]?.[0];
    insertSpy.mockRestore();

    expect(insertedItem.name).toBe("Test Item");
    expect(insertedItem.userId).toBe("user_123456");

    // LAYER 3: assert ORPC input validation passed
    const inputResult = await inputValidationSpy.mock.results[0]?.value;
    if (inputResult?.issues) {
      console.error("ORPC input validation failed:", inputResult.issues);
    }
    expect(inputResult?.issues).toBeUndefined();
    inputValidationSpy.mockRestore();
  });

  test("update via UI triggers updateMany endpoint", async () => {
    debugTexts();
    // Plain spyOn -- no mockImplementation needed. Update does not hang.
    const updateSpy = spyOn({entity}Collection, "update");

    // LAYER 3: ORPC input schema validation for updateMany
    const updateInputSpy = spyOn(
      ({entity}Router.updateMany as any)["~orpc"].inputSchema["~standard"],
      "validate",
    );

    fireEvent.click(screen.getByRole("button", { name: /edit/i }));
    await waitFor(() => screen.getByLabelText(/name/i));
    fireEvent.change(screen.getByLabelText(/name/i), { target: { value: "Updated Item" } });
    fireEvent.click(screen.getByRole("button", { name: /^save$/i }));

    // Assert: collection.update was called
    await waitFor(() => expect(updateSpy).toHaveBeenCalled(), { timeout: 3000 });
    const [updatedId] = updateSpy.mock.calls[0] as any[];
    expect(typeof updatedId).toBe("string");
    updateSpy.mockRestore();

    // LAYER 3: assert ORPC input validation passed
    const updateInputResult = await updateInputSpy.mock.results[0]?.value;
    if (updateInputResult?.issues) {
      console.error("ORPC updateMany input validation failed:", updateInputResult.issues);
    }
    expect(updateInputResult?.issues).toBeUndefined();
    updateInputSpy.mockRestore();
  });

  test("delete via UI triggers deleteMany endpoint", async () => {
    debugTexts();

    // LAYER 3: ORPC input schema validation for deleteMany
    const deleteInputSpy = spyOn(
      ({entity}Router.deleteMany as any)["~orpc"].inputSchema["~standard"],
      "validate",
    );

    // Delete uses 2-step flow: row button -> AlertDialog -> confirm button
    fireEvent.click(screen.getByRole("button", { name: /delete/i }));
    await waitFor(() => screen.getByRole("button", { name: /^delete$/i }));
    fireEvent.click(screen.getByRole("button", { name: /^delete$/i }));

    // LAYER 3: assert ORPC input validation passed
    const deleteInputResult = await deleteInputSpy.mock.results[0]?.value;
    if (deleteInputResult?.issues) {
      console.error("ORPC deleteMany input validation failed:", deleteInputResult.issues);
    }
    expect(deleteInputResult?.issues).toBeUndefined();
    deleteInputSpy.mockRestore();
  });
});
```

## AlertDialog Delete Flow Pitfalls

| Mistake                                           | Symptom                                                    | Fix                                                               |
| ------------------------------------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------- |
| Column uses `confirm()` instead of `meta?.delete` | `confirm` returns `false` in HappyDOM, delete never fires  | Use `onDelete?.(row.original)` which opens AlertDialog            |
| Column passes `row.original.id` to delete handler | AlertDialog text is wrong                                  | Pass `row.original` (full object)                                 |
| Test clicks wrong "Delete" button                 | Clicks the row button again instead of AlertDialog confirm | Use `/^delete$/i` (exact) for confirm, `/delete/i` for row button |

## Common Failure Symptoms

| Symptom                        | Root Cause                                   | Fix                                            |
| ------------------------------ | -------------------------------------------- | ---------------------------------------------- |
| ECONNREFUSED in router         | Router uses `fetchCustomResources`           | Rewrite router to use `createDb()`             |
| Schema validation rejects data | Missing fields or nullable/optional mismatch | Fix the schema overrides                       |
| `onInsert` never called        | Collection schema validation gate            | Add system fields before `collection.insert()` |
| HappyDOM deadlock on insert    | Missing `optimistic: false`                  | Wrap insert spy with `optimistic: false`       |

## Web Key Rules Summary

| Rule                                                                     | Reason                                                                                      |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| Run `bun test` from `packages/test/`                                     | `bunfig.toml` preload path resolves relative to CWD                                         |
| `queryFn` must use `client.entity.selectAll()` directly                  | `files`/`users` collections use HTTP; gets ECONNREFUSED                                     |
| Collection `id`/`queryKey` must match `invalidateQueries` key            | Mismatch means stale data leaks between tests                                               |
| `void mock.module(...)`                                                  | `mock.module()` returns a Promise; `void` suppresses the warning                            |
| Mock `@template/db` before importing router                              | Router captures `createDb` at module load                                                   |
| Mock `@template/auth` before importing router                            | Auth validates env vars at module load                                                      |
| Mock `@/utils/orpc` before importing collections                         | Collection captures `client` and `queryClient` at module load                               |
| Import collections and page components last                              | They must receive the mocked orpc client                                                    |
| Use `@testing-library/react/pure`                                        | Disables auto-cleanup; required when sharing one render across tests                        |
| Use `optimistic: false` on insert spy                                    | Prevents HappyDOM deadlock from React's MessageChannel scheduler                            |
| Assert `insertSpy.mock.results[0]?.value?.error` is undefined            | Catches silent schema validation failures on insert                                         |
| Spy on `collection._mutations.validateData` and assert no issues         | Surfaces collection schema validation failures; never remove this check                     |
| Spy on `router.procedure["~orpc"].inputSchema["~standard"].validate`     | ORPC uses ~standard.validate (NOT safeParse); spy on stable router object, NOT serverClient |
| Always call `spy.mockRestore()` after assertions                         | Prevents spy state from leaking into subsequent tests                                       |
| `spyOn(collection, "method")` name must match collection's actual method | Wrong name = spy never fires                                                                |
| NEVER use `.toBeInTheDocument()`                                         | jest-dom is not installed; use `toBeDefined()` or bare `getByText` inside `waitFor`         |
| NEVER use `any` type or typecheck suppression comments                   | Use proper types, generics, or `unknown` with type narrowing                                |
