---
name: schema0-api-router
description: ORPC API router generation — CRUD procedures, error handling, context, and router registration
---

# API Router

**STOP before writing any router.**

1. New routers MUST use `createDb()` against a Drizzle table.
2. `users.ts` and `files.ts` are NOT templates. They call platform endpoints purpose-built for the `users` and `files` platform features — there is no Drizzle table behind them. Your application's entities are application data and go through your own tables; do not copy `users.ts` / `files.ts` for any new entity.
3. If your entity has no Drizzle table yet, invoke `schema0-db-schema` first to define one.

## Does this entity need RLS?

Decide before writing the router. The entity needs Row-Level Security if any of the following apply, based on the user's request:

- Each user sees only their own rows (e.g. personal notes, tasks, journal entries).
- Data is scoped per organization or tenant.
- The entity has ownership or permission boundaries (creator vs reader vs editor).

If yes → invoke `Skill("schema0-rls")` and follow that skill's template. RLS routers wrap each handler in `createRLSTransaction(context.request)` instead of calling `createDb()` directly. The procedure shape (`selectAll` / `selectById` / `insertMany` / `updateMany` / `deleteMany`) is the same as below — only the database access pattern changes.

If no → continue with the `createDb()` template that follows.

## File Location

```
packages/api/src/
├── index.ts           # Base procedures (publicProcedure, protectedProcedure)
├── context.ts         # Request context type
└── routers/
    ├── index.ts       # App router (register all routers here)
    └── [entity].ts    # Entity-specific CRUD routers
```

## Prerequisites

Every router provides 5 bulk CRUD operations: `selectAll`, `selectById`, `insertMany`, `updateMany`, `deleteMany`.

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

### Error Handling (ORPCError)

```typescript
import { ORPCError } from "@orpc/server";

// CORRECT -- string code + options object
throw new ORPCError("NOT_FOUND", { status: 404, message: "Resource not found" });
throw new ORPCError("CONFLICT", { status: 409, message: "Already exists" });

// WRONG -- object with code property
throw new ORPCError({ code: "NOT_FOUND", message: "Not found" });
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

## Key Rules

1. **Use `.handler()` for ALL procedures** -- NEVER `.query()` or `.mutation()` (tRPC patterns).
2. **Use `createDb()` inside each handler** -- never module-level singleton.
3. **NEVER use `fetchCustomResources`** for new routers -- only for built-in `files.ts` and `users.ts`.
4. **Use `import { z } from "zod/v4"`** -- NEVER `import z from "zod"`.
5. **Import schemas from `@template/db/schema`** -- NEVER define `z.object()` schemas inline.
6. **Use `{entity}RouterOutputSchema` for `.output()`** -- must match DB return types.
7. **Bulk operations only** -- `insertMany`, `updateMany`, `deleteMany` for TanStack DB optimistic updates.
8. **Data MUST be stored in database** -- every mutation must hit DB and return persisted result.
9. **Register in `packages/api/src/routers/index.ts`** after creating.

## References

- `references/crud-template.md` -- Full CRUD router code template (selectAll, selectById, insertMany, updateMany, deleteMany) + registration example
