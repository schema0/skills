# RLS Implementation

## Database Access Pattern

ALL RLS database operations MUST use `createRLSTransaction`:

```typescript
import { createRLSTransaction } from "@template/db";
import { protectedProcedure } from "../index";

export const usersRouter = {
  getAll: protectedProcedure.handler(async ({ context }) => {
    const RLSTransaction = await createRLSTransaction(context.request);
    return await RLSTransaction(async (tx) => {
      return await tx.select().from(users);
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

```typescript
// packages/api/src/routers/posts.ts
import { protectedProcedure } from "../index";
import { createRLSTransaction } from "@template/db";
import { posts } from "@template/db/schema";
import { eq } from "drizzle-orm";
import { z } from "zod/v4";

export const postsRouter = {
  getAll: protectedProcedure.handler(async ({ context }) => {
    const RLSTransaction = await createRLSTransaction(context.request);
    return await RLSTransaction(async (tx) => {
      return await tx.select().from(posts);
    });
  }),

  getById: protectedProcedure
    .input(z.object({ id: z.string() }))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        const result = await tx
          .select()
          .from(posts)
          .where(eq(posts.id, input.id));
        return result[0] || null;
      });
    }),

  create: protectedProcedure
    .input(
      z.object({
        title: z.string().min(1),
        content: z.string(),
      }),
    )
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      const userId = context.session.user.id;
      return await RLSTransaction(async (tx) => {
        return await tx
          .insert(posts)
          .values({ ...input, userId })
          .returning();
      });
    }),

  update: protectedProcedure
    .input(
      z.object({
        id: z.string(),
        title: z.string().optional(),
        content: z.string().optional(),
      }),
    )
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      const { id, ...data } = input;
      return await RLSTransaction(async (tx) => {
        return await tx
          .update(posts)
          .set(data)
          .where(eq(posts.id, id))
          .returning();
      });
    }),

  delete: protectedProcedure
    .input(z.object({ id: z.string() }))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        return await tx.delete(posts).where(eq(posts.id, input.id)).returning();
      });
    }),
};
```

## Router with Filtering

```typescript
export const tasksRouter = {
  getByStatus: protectedProcedure
    .input(z.object({ completed: z.boolean() }))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        return await tx
          .select()
          .from(tasks)
          .where(eq(tasks.completed, input.completed))
          .orderBy(desc(tasks.createdAt));
      });
    }),

  toggle: protectedProcedure
    .input(z.object({ id: z.string() }))
    .handler(async ({ input, context }) => {
      const RLSTransaction = await createRLSTransaction(context.request);
      return await RLSTransaction(async (tx) => {
        const [task] = await tx
          .select()
          .from(tasks)
          .where(eq(tasks.id, input.id))
          .limit(1);
        if (!task) throw new Error("Task not found");
        return await tx
          .update(tasks)
          .set({ completed: !task.completed })
          .where(eq(tasks.id, input.id))
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

## Post-Generation Steps

1. Export schema in `packages/db/src/schema/index.ts`:
   ```typescript
   export * from "./tasks";
   ```
2. Register router in `packages/api/src/routers/index.ts`:
   ```typescript
   import { tasksRouter } from "./tasks";
   export const appRouter = { tasks: tasksRouter, ... };
   ```
3. Complete TODOs in generated router (schema fields, table operations)
4. Generate and apply migrations:
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
