# RLS Policy Patterns

## RLS Imports

```typescript
import { pgTable, text, timestamp, bigint, boolean } from "drizzle-orm/pg-core";
import { crudPolicy, authUid } from "drizzle-orm/neon";
import { authenticatedUserRole } from "./index";
```

- `crudPolicy` -- Helper to define RLS policies
- `authUid` -- Checks if current user matches userId column
- `authenticatedUserRole` -- Role for authenticated users (defined in `packages/db/src/schema/index.ts`)

## Authenticated User Role

Define once in `packages/db/src/schema/index.ts`:

```typescript
import { pgRole } from "drizzle-orm/pg-core";

export const authenticatedUserRole = pgRole("authenticated_user").existing();
```

## Complete Entity File (table + 7 schemas)

This is the canonical RLS entity. Same 7-schema pattern as `schema0-db-schema`, with `crudPolicy` added to the table and `userId` as a server-stamped field.

```typescript
// packages/db/src/schema/posts.ts
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";
import { crudPolicy, authUid } from "drizzle-orm/neon";
import { createInsertSchema, createSelectSchema } from "drizzle-zod";
import { z } from "zod/v4";
import { authenticatedUserRole } from "./index";

// 1. TABLE DEFINITION (with RLS policy)
export const posts = pgTable(
  "posts",
  {
    id: text("id").primaryKey(),
    userId: text("user_id").notNull(), // server-stamped from session
    title: text("title").notNull(),
    content: text("content").notNull(),
    isPublic: boolean("is_public").default(false).notNull(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at").defaultNow().notNull(),
  },
  (table) => [
    crudPolicy({
      role: authenticatedUserRole,
      read: authUid(table.userId),
      modify: authUid(table.userId),
    }),
  ],
);

// 2. INSERT SCHEMA — userId is .optional() because the router stamps it from the session
export const insertPostsSchema = createInsertSchema(posts, {
  title: (schema) => schema.min(1).max(200),
  userId: (schema) => schema.optional(),
  isPublic: (schema) => schema.optional(),
});

// 3. SELECT SCHEMA
export const selectPostsSchema = createSelectSchema(posts, {
  isPublic: (schema) => schema.optional(),
});

// 4. UPDATE SCHEMA
export const updatePostsSchema = selectPostsSchema
  .partial()
  .required({ id: true });

// 5. FORM SCHEMA
export const postsFormSchema = insertPostsSchema.omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

// 6. EDIT FORM SCHEMA
export const postsEditFormSchema = postsFormSchema.partial();

// 7. ROUTER OUTPUT SCHEMA
export const postsRouterOutputSchema = z.object({
  id: z.string(),
  userId: z.string(),
  title: z.string(),
  content: z.string(),
  isPublic: z.boolean().nullable().optional(),
  createdAt: z.date(),
  updatedAt: z.date(),
});
```

The `userId` server-stamping pattern (notNull on the table, optional on insert schema, filled in by the router from `context.session.user.id`) is the same one documented in `schema0-db-schema/references/schema-template.md` under "Server-Stamped Fields". It is what lets the RLS policy enforce ownership while letting clients omit `userId` from their payloads.

## User-Scoped Data (Most Common)

```typescript
crudPolicy({
  role: authenticatedUserRole,
  read: authUid(table.userId),
  modify: authUid(table.userId),
});
```

User can only read and modify their own data.

## Public Read, User Write

```typescript
crudPolicy({
  role: authenticatedUserRole,
  read: sql`true`,
  modify: authUid(table.userId),
});
```

All authenticated users can read, only owner can modify.

## Admin-Only Access

```typescript
crudPolicy({
  role: authenticatedUserRole,
  read: sql`false`,
  modify: sql`false`,
});
```

No regular users can access (admin only via different role).

## Adding Relations

```typescript
import { relations } from "drizzle-orm";

export const postsRelations = relations(posts, ({ one, many }) => ({
  user: one(users, { fields: [posts.userId], references: [users.id] }),
  comments: many(comments),
}));
```
