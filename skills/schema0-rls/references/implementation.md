# RLS Implementation

## Database Access Pattern

ALL RLS database operations MUST use `createRLSTransaction`:

```typescript
import { createRLSTransaction } from "@template/db";
import { protectedProcedure } from "../index";

export const postsRouter = {
  selectAll: protectedProcedure.handler(async ({ context }) => {
    const RLSTransaction = await createRLSTransaction(context.request);
    return await RLSTransaction(async (tx) => {
      return await tx.select().from(posts);
    });
  }),
};
```

### How It Works

1. `createRLSTransaction(request)` takes the request object from ORPC context
2. Internally uses `env.YB_API_HOSTNAME`, `env.YB_ORGANIZATION_ID`
3. Fetches JWT claims from `https://${env.YB_API_HOSTNAME}/api/user_management/claims`
4. Validates claims against `env.YB_ORGANIZATION_ID` for authorization
5. Returns a function that wraps queries in a transaction with RLS context
6. `RLSTransaction(async (tx) => { ... })` sets JWT claims via `set_config('request.jwt.claims', ...)`
7. Database enforces RLS policies based on authenticated user's claims

## Complete CRUD Router Example

Procedure shape matches `schema0-api-router` (`selectAll` / `selectById` / `insertMany` / `updateMany` / `deleteMany`); only the database access pattern differs.

```typescript
// packages/api/src/routers/posts.ts
import { z } from "zod/v4";
import { eq, inArray } from "drizzle-orm";
import { protectedProcedure } from "../index";
import { createRLSTransaction } from "@template/db";
import {
  posts,
  insertPostsSchema,
  updatePostsSchema,
  postsRouterOutputSchema,
} from "@template/db/schema";

export const postsRouter = {
  selectAll: protectedProcedure
    .input(z.object({}).optional())
    .output(z.array(postsRouterOutputSchema))
    .handler(async ({ context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        return await tx.select().from(posts);
      });
    }),

  selectById: protectedProcedure
    .input(z.object({ id: z.string() }))
    .output(postsRouterOutputSchema.nullable())
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        const result = await tx
          .select()
          .from(posts)
          .where(eq(posts.id, input.id))
          .limit(1);
        return result[0] ?? null;
      });
    }),

  insertMany: protectedProcedure
    .input(z.array(insertPostsSchema))
    .output(z.array(postsRouterOutputSchema))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      const userId = context.session.user.id;
      return await RLSTransaction(async (tx) => {
        return await tx
          .insert(posts)
          .values(input.map((row) => ({ ...row, userId })))
          .returning();
      });
    }),

  updateMany: protectedProcedure
    .input(z.array(updatePostsSchema))
    .output(z.array(postsRouterOutputSchema))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        const results = [];
        for (const item of input) {
          const { id, ...data } = item;
          const updated = await tx
            .update(posts)
            .set({ ...data, updatedAt: new Date() })
            .where(eq(posts.id, id))
            .returning();
          results.push(...updated);
        }
        return results;
      });
    }),

  deleteMany: protectedProcedure
    .input(z.array(z.object({ id: z.string() })))
    .output(z.array(postsRouterOutputSchema))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      const ids = input.map((i) => i.id);
      return await RLSTransaction(async (tx) => {
        return await tx
          .delete(posts)
          .where(inArray(posts.id, ids))
          .returning();
      });
    }),
};
```

## When to Use RLS vs Regular db

### Use `createRLSTransaction` for:

- User-scoped data (posts, tasks, comments)
- Data filtered by ownership
- Tables with `crudPolicy` in schema
- Any table with a `userId` column

### Use regular `createDb()` for:

- Public/shared data (categories, tags, settings)
- Admin-only tables without user filtering
- Tables without RLS policies
- Lookup tables and reference data

## Common Mistakes

### NEVER: Direct database connections for RLS tables

```typescript
// WRONG -- Bypasses RLS
const db = createDb();
return await db.select().from(tasks); // Won't filter by user!

// CORRECT
const RLSTransaction = await createRLSTransaction(context.request);
return await RLSTransaction(async (tx) => {
  return await tx.select().from(tasks);
});
```

### NEVER: Database operations in route loaders

```typescript
// WRONG -- All database operations belong in ORPC procedures
export async function loader({ request }: Route.LoaderArgs) {
  const RLSTransaction = await createRLSTransaction(request);
  // Put in ORPC router instead!
}
```

### ALWAYS: Set userId when creating records

```typescript
const userId = context.session.user.id;
await tx.insert(tasks).values({ title: "Task", userId }); // MUST include userId
```

### ALWAYS: Use protectedProcedure

RLS requires authenticated requests. Never use `publicProcedure` for RLS tables.

## Migration Workflow

```bash
# 1. Edit packages/db/src/schema/[table].ts
# 2. Generate migration
schema0 sandbox exec "bun run db:generate"
# 3. Review generated SQL in packages/db/src/migrations/
# 4. Apply migration
schema0 sandbox exec "bun run db:migrate"
```

## Wiring Steps

1. Export schema in `packages/db/src/schema/index.ts`:
   ```typescript
   export * from "./posts";
   ```
2. Register router in `packages/api/src/routers/index.ts`:
   ```typescript
   import { postsRouter } from "./posts";
   export const appRouter = { posts: postsRouter, ... };
   ```
3. Generate and apply migrations:
   ```bash
   schema0 sandbox exec "bun run db:generate && bun run db:migrate"
   ```

## Debugging RLS Issues

| Problem                   | Cause                             | Fix                                             |
| ------------------------- | --------------------------------- | ----------------------------------------------- |
| Empty results             | Not using `createRLSTransaction`  | Wrap queries in `RLSTransaction`                |
| Can see other users' data | Using `createDb()` instead of RLS | Switch to `createRLSTransaction`                |
| Cannot create records     | Missing `userId` on insert        | Set `userId` from `context.session.user.id`     |
| Policies not applied      | Migrations not run                | Run `bun run db:generate && bun run db:migrate` |

## Deployment Checklist

1. Database schema has RLS policies with `crudPolicy()`
2. Schema exported in `packages/db/src/schema/index.ts`
3. Migrations generated and applied
4. Router created and registered in `packages/api/src/routers/index.ts`
5. All RLS procedures use `createRLSTransaction`
6. All creates set `userId` from `context.session.user.id`
7. Environment variables set: `DATABASE_URL`, `YB_API_HOSTNAME`, `YB_ORGANIZATION_ID`
