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

## Complete Table Example

```typescript
// packages/db/src/schema/posts.ts
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";
import { crudPolicy, authUid } from "drizzle-orm/neon";
import { authenticatedUserRole } from "./index";

export const posts = pgTable(
  "posts",
  {
    id: text("id").primaryKey(),
    userId: text("user_id").notNull(),
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
```

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
