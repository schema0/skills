# Project Architecture

## Structure

```
packages/db/      — Database schemas (Drizzle ORM)
packages/api/     — ORPC API routers
packages/auth/    — Server-side auth (uses @schema0/auth-web)
packages/test/    — Integrated tests (PGlite + UI)
packages/config/  — Shared config
apps/web/         — React Router v7 + TanStack DB         → web platform (if exists)
apps/native/      — React Native / Expo                   → mobile platform (if exists)
```

## Platform Detection

Before executing platform-specific tasks, check which platforms are installed:

- **Web**: `apps/web/` exists
- **Mobile**: `apps/native/` exists
- **Template-only**: neither exists (only `packages/`)

## Global Rules

1. **Use PLURAL entity names everywhere** — `themes`, `customers`, `activities` (not singular)
2. **Use `import { z } from "zod/v4"`** — NEVER `import z from "zod"` (v3-compat breaks drizzle-zod at runtime)
3. **No `any` types, no typecheck suppression** — NEVER use `any`, `// @ts-ignore`, `// @ts-expect-error`, `// @ts-nocheck`, or `// eslint-disable`. Fix the type error instead.
4. **Never hand-write migration files** — `packages/db/drizzle/` is managed by `drizzle-kit generate` and `drizzle-kit migrate`
5. **Use createDb() per request** — never use singleton db instances. Cloudflare Workers isolate requests — a singleton causes "Cannot perform I/O on behalf of a different request" errors.
6. **Testing is mandatory** — every feature MUST have tests. A feature is NOT complete without a passing test file.

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

Without web: only files 1–2.

## DB Package (`@template/db`)

```
packages/db/src/
├── index.ts          # Database client (createDb) and exports
├── schema/
│   ├── index.ts      # Barrel re-exports (export * from "./{entity}")
│   └── [entity].ts   # Table + all derived schemas (insert, select, update, form)
└── migrations/       # Auto-generated — never hand-write
```

### Database Client

Always use `createDb()` to create a fresh instance per request. Cloudflare Workers isolate requests — a singleton `db` causes "Cannot perform I/O on behalf of a different request" errors.

```typescript
import { createDb } from "@template/db";

// Fresh instance inside handler
const db = createDb();
return await db.select().from(users);

// WRONG: Module-level singleton — breaks in Workers
const db = createDb(); // at top of file
```

### Schema Definition (Single Source of Truth)

ALL schemas — table, insert, select, update, form, editForm — are defined in `packages/db/src/schema/{entity}.ts` using drizzle-zod. Routers, collections, and forms import from `@template/db/schema`. Never define `z.object()` schemas manually elsewhere.

#### Definitive Schema Example

```typescript
// packages/db/src/schema/entities.ts
import { pgTable, text, boolean, timestamp } from "drizzle-orm/pg-core";
import { createInsertSchema, createSelectSchema } from "drizzle-zod";

// 1. TABLE DEFINITION
export const entities = pgTable("entities", {
  id: text("id").primaryKey(), // ALWAYS text — enables optimistic updates
  name: text("name").notNull(),
  description: text("description"), // Nullable column (string | null in DB)
  email: text("email").notNull(),
  active: boolean("active").default(true),
  userId: text("user_id"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

// 2. INSERT SCHEMA — use CALLBACK overrides for nullable columns
// ALWAYS use callbacks (schema) => ... — NEVER direct z.string().optional()
export const insertEntitiesSchema = createInsertSchema(entities, {
  name: (schema) => schema.min(1).max(200),
  email: (schema) => schema.email(),
  description: (schema) => schema.optional(),
});

// 3. SELECT SCHEMA
export const selectEntitiesSchema = createSelectSchema(entities, {
  description: (schema) => schema.optional(),
});

// 4. UPDATE SCHEMA
export const updateEntitiesSchema = selectEntitiesSchema
  .partial()
  .required({ id: true });

// 5. FORM SCHEMA — for react-hook-form create mode (omits system fields)
export const entitiesFormSchema = insertEntitiesSchema.omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

// 6. EDIT FORM SCHEMA — MUST NOT include id (Dialog adds it after submission)
export const entitiesEditFormSchema = entitiesFormSchema
  .omit({ email: true })
  .partial();

// 7. ROUTER OUTPUT SCHEMA — for .output() on selectAll/selectById
// timestamp() returns Date objects, NOT strings
export const entitiesRouterOutputSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string().nullable().optional(),
  email: z.string(),
  active: z.boolean().nullable().optional(),
  userId: z.string().nullable().optional(),
  createdAt: z.date(), // Date, NOT z.string()
  updatedAt: z.date(), // Date, NOT z.string()
});
```

#### Schema Key Rules

1. **Callback overrides for nullable columns:** For every column without `.notNull()`, add: `colName: (schema) => schema.optional()` to convert `string | null` to `string | undefined` for React Hook Form compatibility.
2. **NEVER use `.transform()` for null-to-undefined:** Creates `ZodEffects` which breaks `.omit()` and `.partial()` chaining.
3. **NEVER use `z.unknown()` or `z.any()`:** Add callback overrides instead.
4. **NEVER use `serial()` or `bigint` for primary keys:** Only `text("id").primaryKey()` supports client-generated IDs for optimistic updates.
5. **Router output schema MUST mirror table nullability:** Every column WITHOUT `.notNull()` needs `.nullable().optional()` in the output schema. Omitting `.nullable()` causes silent "Output validation failed".
6. **Form schemas belong in the entity file**, not in `index.ts`. The barrel file only contains `export * from "./{entity}"`.

#### Schema Naming Conventions

All names use PLURAL form:

| Schema                         | Derivation                                      | Purpose                       |
| ------------------------------ | ----------------------------------------------- | ----------------------------- |
| `{entities}FormSchema`         | `insertSchema.omit({id, createdAt, updatedAt})` | Create form validation        |
| `{entities}EditFormSchema`     | `formSchema.partial()` — NO `id`                | Edit form validation          |
| `insert{Entities}Schema`       | `createInsertSchema(table, overrides)`          | Bulk insert validation        |
| `select{Entities}Schema`       | `createSelectSchema(table, overrides)`          | Query/collection validation   |
| `update{Entities}Schema`       | `selectSchema.partial().required({id})`         | Bulk update validation        |
| `{entities}RouterOutputSchema` | `z.object({...})` with DB return types          | Router `.output()` validation |

### Transform Utilities

Form-to-Database pipeline utilities:

```typescript
import {
  nullToUndefined,
  prepareFormForQueryCollection,
} from "@template/db/schema";

// Convert null to undefined
const queryData = nullToUndefined(formData);

// Complete pipeline: validate, transform, add system fields
const queryData = prepareFormForQueryCollection(formData, entityFormSchema, {
  id: generateId(),
  userId: currentUser.id,
  createdAt: new Date(),
  updatedAt: new Date(),
});
```

### Column Types Reference

```typescript
import { pgTable, text, varchar, integer, boolean, timestamp, jsonb, decimal, real } from "drizzle-orm/pg-core";

id: text("id").primaryKey(),
name: text("name").notNull(),
email: varchar("email", { length: 255 }).unique(),
age: integer("age"),
price: decimal("price", { precision: 10, scale: 2 }),
active: boolean("active").default(true),
metadata: jsonb("metadata").$type<{ key: string }>(),
createdAt: timestamp("created_at").defaultNow().notNull(),
updatedAt: timestamp("updated_at").defaultNow().notNull(),
```

## API Package (`@template/api`)

```
packages/api/src/
├── index.ts           # Base procedures (publicProcedure, protectedProcedure)
├── context.ts         # Request context type
└── routers/
    ├── index.ts       # App router (register all routers here)
    └── [entity].ts    # Entity-specific CRUD routers
```

### Base Procedures

```typescript
import { ORPCError, os } from "@orpc/server";
import type { Context } from "./context";

export const o = os.$context<Context>();
export const publicProcedure = o;

const requireAuth = o.middleware(async ({ context, next }) => {
  if (!context.session?.user) {
    throw new ORPCError("UNAUTHORIZED");
  }
  return next({ context: { session: context.session } });
});

export const protectedProcedure = publicProcedure.use(requireAuth);
```

### Router Registration

```typescript
// packages/api/src/routers/index.ts
import { publicProcedure } from "../index";
import { usersRouter } from "./users";
import { entitiesRouter } from "./entities";

export const appRouter = {
  healthCheck: publicProcedure.handler(() => "OK"),
  users: usersRouter,
  entities: entitiesRouter,
};

export type AppRouter = typeof appRouter;
```

### Error Handling (ORPCError)

```typescript
import { ORPCError } from "@orpc/server";

// CORRECT — string code + options object
throw new ORPCError("NOT_FOUND", {
  status: 404,
  message: "Resource not found",
});
throw new ORPCError("CONFLICT", { status: 409, message: "Already exists" });
throw new ORPCError("BAD_REQUEST", { status: 400, message: "Invalid input" });
throw new ORPCError("UNAUTHORIZED", {
  status: 401,
  message: "Not authenticated",
});

// WRONG — object with code property
throw new ORPCError({ code: "NOT_FOUND", message: "Not found" });
```

### Using Context

```typescript
export const profileRouter = {
  getMyProfile: protectedProcedure.handler(async ({ context }) => {
    const userId = context.session.user.id;
    const db = createDb();
    const user = await db
      .select()
      .from(users)
      .where(eq(users.id, userId))
      .limit(1);
    return user[0];
  }),
};
```

### Nested Routers

```typescript
// packages/api/src/routers/admin/users.ts
export const adminUsersRouter = { list: protectedProcedure.handler(async () => { ... }) };

// packages/api/src/routers/admin/index.ts
export const adminRouter = { users: adminUsersRouter };

// packages/api/src/routers/index.ts
export const appRouter = { admin: adminRouter };
// Client: orpc.admin.users.list.queryOptions()
```

### API Critical Rules

1. **NEVER use `fetchCustomResources` for new routers** — only for built-in `files.ts` and `users.ts`. All other routers MUST use `createDb()` directly.
2. **Use `import { z } from "zod/v4"`** — NEVER `import z from "zod"`.
3. **Import schemas from `@template/db/schema`** — NEVER define `z.object()` schemas inline in router files.
4. **Use `{entity}RouterOutputSchema` for `.output()` validation** — must match DB return types (timestamps are `z.date()`, not `z.string()`).
5. **Data MUST be stored in database** — every mutation must hit DB and return persisted result.
6. **Fresh `createDb()` per request** — never module-level singleton.
7. **Use `.handler()` for ALL procedures** — NEVER `.query()` or `.mutation()` (those are tRPC patterns).
8. **Bulk operations only** — `insertMany`, `updateMany`, `deleteMany` for TanStack DB optimistic updates.

### API Anti-Patterns

- NEVER use tRPC patterns (`initTRPC.create()`, `t.router`, `t.procedure`)
- NEVER use `.query()` or `.mutation()` — only `.handler()`
- NEVER perform database operations in route loaders
- NEVER use singleton db at module level
- NEVER use `fetchCustomResources` for new entities
- NEVER define schemas inline — import from `@template/db/schema`
- NEVER use `any` type or typecheck suppression comments

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

1. **Middleware** — Apply `authMiddleware` to protected layout (`apps/web/src/routes/_auth.tsx`)
2. **Login** — Redirect to sign-in URL via `auth.getSignInUrl()`
3. **Auth Context** — Store auth data via `authContext.set()`
4. **Access Auth** — Get auth data via `authContext.get()`
5. **Logout** — Redirect to logout URL via `auth.getLogoutUrl()`

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

- Use `@template/auth` for server-side auth — NEVER import `@schema0/auth-web` directly
- Environment variables are validated at startup
- For mobile, use `@schema0/auth-mobile` instead
