# Full CRUD Router Template

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

## Register in index.ts

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
