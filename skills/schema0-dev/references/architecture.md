# Project Architecture

## Structure

```
packages/db/      -- Database schemas (Drizzle ORM)
packages/api/     -- ORPC API routers
packages/auth/    -- Server-side auth (uses @schema0/auth-web)
packages/test/    -- Integrated tests (PGlite + UI)
packages/config/  -- Shared config
apps/web/         -- React Router v7 + TanStack DB         -> web platform (if exists)
apps/native/      -- React Native / Expo                   -> mobile platform (if exists)
```

## Platform Detection

Before executing platform-specific tasks, check which platforms are installed:

- **Web**: `apps/web/` exists
- **Mobile**: `apps/native/` exists
- **Template-only**: neither exists (only `packages/`)

## Global Rules

1. **Use PLURAL entity names everywhere** -- `themes`, `customers`, `activities` (not singular)
2. **Use `import { z } from "zod/v4"`** -- NEVER `import z from "zod"` (v3-compat breaks drizzle-zod at runtime)
3. **No `any` types, no typecheck suppression** -- NEVER use `any`, `// @ts-ignore`, `// @ts-expect-error`, `// @ts-nocheck`, or `// eslint-disable`. Fix the type error instead.
4. **Never hand-write migration files** -- `packages/db/drizzle/` is managed by `drizzle-kit generate` and `drizzle-kit migrate`
5. **Use createDb() per request** -- never use singleton db instances. Cloudflare Workers isolate requests -- a singleton causes "Cannot perform I/O on behalf of a different request" errors.
6. **Testing is mandatory** -- every feature MUST have tests. A feature is NOT complete without a passing test file.

## Catalog Dependencies

The root `package.json` defines a `catalog` with shared dependency versions. Use `catalog:` in app `package.json` files. All packages in `packages/` are workspace dependencies (`"@template/db": "workspace:*"`).

## Environment Variables

Access: `import { env } from "@template/auth"`

| Variable       | Description                |
| -------------- | -------------------------- |
| `YB_URL`       | App URL                    |
| `DATABASE_URL` | Database connection string |

## File Checklist (per entity)

| #   | File          | Location                                                                   |
| --- | ------------- | -------------------------------------------------------------------------- |
| 1   | Schema        | `packages/db/src/schema/{entity}.ts`                                       |
| 2   | Router        | `packages/api/src/routers/{entity}.ts`                                     |
| 3   | Collection    | `apps/web/src/query-collections/custom/{entity}.ts`                        |
| 4   | Dialog        | `apps/web/src/components/ui/data-table/custom/{entity}/{Entity}Dialog.tsx` |
| 5   | Form          | `apps/web/src/components/ui/data-table/custom/{entity}/{Entity}Form.tsx`   |
| 6   | Columns       | `apps/web/src/components/ui/data-table/custom/{entity}/{Entity}Column.tsx` |
| 7   | Index         | `apps/web/src/components/ui/data-table/custom/{entity}/index.ts`           |
| 8   | List Route    | `apps/web/src/routes/_auth.{entity}.tsx`                                   |
| 9   | Detail Route  | `apps/web/src/routes/_auth.{entity}_.$id.tsx`                              |
| 10  | Test (web)    | `packages/test/web/{entity}.test.tsx`                                      |
| 11  | Test (mobile) | `packages/test/mobile/{entity}.test.tsx`                                   |

Without web: only files 1-2.

## Auth Package (`@template/auth`)

Wraps `@schema0/auth-web` (npm package) with environment configuration and pre-built auth client.

### Key Exports

```typescript
import { auth } from "@template/auth"; // Auth client
import { env } from "@template/auth"; // Validated env vars
import type { User, Impersonator } from "@template/auth"; // Types
```

### Getting the User

```typescript
const user = await auth.getUser(request);
// user contains: id, email, name, organizationId, impersonator?
```

### Auth URLs

```typescript
// Sign-in URL
const signInUrl = await auth.getSignInUrl({
  state: { redirect: "/dashboard" },
});

// Logout URL
const logoutUrl = await auth.getLogoutUrl({
  returnTo: env.YB_URL,
  serverRequest: request,
});

// Access token
const accessToken = await auth.getAccessToken(request);
```

### Auth Middleware Pattern

```typescript
import { auth, env, type User } from "@template/auth";
import { redirect } from "react-router";
import { authContext } from "@/context";

export const authMiddleware: Route.MiddlewareFunction = async ({
  request,
  context,
}) => {
  try {
    const user = await auth.getUser(request);
    if (!user) {
      const url = new URL(request.url);
      throw redirect(
        `/login?redirect=${encodeURIComponent(url.pathname + url.search)}`,
      );
    }
    authContext.set(context, {
      user,
      organizationId: env.YB_ORGANIZATION_ID,
      appId: env.YB_APP_ID,
      apiHostname: env.YB_API_HOSTNAME,
    });
  } catch (error) {
    const url = new URL(request.url);
    throw redirect(
      `/login?redirect=${encodeURIComponent(url.pathname + url.search)}`,
    );
  }
};
```

### Authentication Flow

1. **Middleware** -- Apply `authMiddleware` to protected layout (`apps/web/src/routes/_auth.tsx`)
2. **Login** -- Redirect to sign-in URL via `auth.getSignInUrl()`
3. **Auth Context** -- Store auth data via `authContext.set()`
4. **Access Auth** -- Get auth data via `authContext.get()`
5. **Logout** -- Redirect to logout URL via `auth.getLogoutUrl()`

### Environment Variables

```typescript
import { env } from "@template/auth";

env.YB_URL; // App URL
env.YB_API_HOSTNAME; // Schema0 API hostname
env.YB_ORGANIZATION_ID; // Organization ID
env.YB_APP_ID; // App ID
env.DATABASE_URL; // Database connection string
```

### Auth Rules

- Use `@template/auth` for server-side auth -- NEVER import `@schema0/auth-web` directly
- Environment variables are validated at startup
- For mobile, use `@schema0/auth-mobile` instead
