---
name: schema0-api-router
description: ORPC API router generation — CRUD procedures, error handling, context, and router registration
---

# API Router

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

---

## Full Router Template

**File:** `packages/api/src/routers/{entities}.ts`

Every router provides 5 bulk CRUD operations: `selectAll`, `selectById`, `insertMany`, `updateMany`, `deleteMany`.

```typescript
// packages/api/src/routers/entities.ts
import { z } from "zod/v4";
import { eq, inArray } from "drizzle-orm";
import { createDb } from "@template/db";
import { protectedProcedure } from "../index";
import {
  entities,
  insertEntitiesSchema,
  updateEntitiesSchema,
  entitiesRouterOutputSchema,
} from "@template/db/schema";

export const entitiesRouter = {
  // ──────────────────────────────────────────────
  // SELECT ALL
  // ──────────────────────────────────────────────
  selectAll: protectedProcedure
    .input(z.object({}).optional())
    .output(z.array(entitiesRouterOutputSchema))
    .handler(async ({ context }) => {
      const db = createDb();
      return await db
        .select()
        .from(entities)
        .where(eq(entities.userId, context.session.user.id));
    }),

  // ──────────────────────────────────────────────
  // SELECT BY ID
  // ──────────────────────────────────────────────
  selectById: protectedProcedure
    .input(z.object({ id: z.string() }))
    .output(entitiesRouterOutputSchema.nullable())
    .handler(async ({ input }) => {
      const db = createDb();
      const result = await db
        .select()
        .from(entities)
        .where(eq(entities.id, input.id))
        .limit(1);
      return result[0] ?? null;
    }),

  // ──────────────────────────────────────────────
  // INSERT MANY
  // ──────────────────────────────────────────────
  insertMany: protectedProcedure
    .input(z.array(insertEntitiesSchema))
    .output(z.array(entitiesRouterOutputSchema))
    .handler(async ({ input }) => {
      const db = createDb();
      return await db.insert(entities).values(input).returning();
    }),

  // ──────────────────────────────────────────────
  // UPDATE MANY
  // ──────────────────────────────────────────────
  updateMany: protectedProcedure
    .input(z.array(updateEntitiesSchema))
    .output(z.array(entitiesRouterOutputSchema))
    .handler(async ({ input }) => {
      const db = createDb();
      const results = [];
      for (const item of input) {
        const { id, ...data } = item;
        const updated = await db
          .update(entities)
          .set({ ...data, updatedAt: new Date() })
          .where(eq(entities.id, id))
          .returning();
        results.push(...updated);
      }
      return results;
    }),

  // ──────────────────────────────────────────────
  // DELETE MANY
  // ──────────────────────────────────────────────
  deleteMany: protectedProcedure
    .input(z.array(z.object({ id: z.string() })))
    .output(z.array(entitiesRouterOutputSchema))
    .handler(async ({ input }) => {
      const db = createDb();
      const ids = input.map((i) => i.id);
      return await db
        .delete(entities)
        .where(inArray(entities.id, ids))
        .returning();
    }),
};
```

### Register in index.ts

```typescript
// packages/api/src/routers/index.ts
import { publicProcedure } from "../index";
import { usersRouter } from "./users";
import { entitiesRouter } from "./entities";

export const appRouter = {
  healthCheck: publicProcedure.handler(() => "OK"),
  users: usersRouter,
  entities: entitiesRouter, // <-- add new router here
};

export type AppRouter = typeof appRouter;
```

### Router Critical Rules

1. **Use `.handler()` for ALL procedures** -- NEVER `.query()` or `.mutation()` (those are tRPC patterns).
2. **Use `createDb()` inside each handler** -- never module-level singleton. Cloudflare Workers isolate requests.
3. **NEVER use `fetchCustomResources`** for new routers -- only for built-in `files.ts` and `users.ts`.
4. **Use `import { z } from "zod/v4"`** -- NEVER `import z from "zod"`.
5. **Import schemas from `@template/db/schema`** -- NEVER define `z.object()` schemas inline.
6. **Use `{entity}RouterOutputSchema` for `.output()`** -- must match DB return types.
7. **Bulk operations only** -- `insertMany`, `updateMany`, `deleteMany` for TanStack DB optimistic updates.
8. **Data MUST be stored in database** -- every mutation must hit DB and return persisted result.

### API Anti-Patterns

- NEVER use tRPC patterns (`initTRPC.create()`, `t.router`, `t.procedure`)
- NEVER use `.query()` or `.mutation()` — only `.handler()`
- NEVER perform database operations in route loaders
- NEVER use singleton db at module level
- NEVER use `fetchCustomResources` for new entities
- NEVER define schemas inline — import from `@template/db/schema`
- NEVER use `any` type or typecheck suppression comments
