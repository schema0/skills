# Column Types Reference

## Available Column Types

```typescript
import { pgTable, text, varchar, integer, boolean, timestamp, jsonb, decimal, real } from "drizzle-orm/pg-core";

id: text("id").primaryKey(),
name: text("name").notNull(),
email: varchar("email", { length: 255 }).unique(),
age: integer("age"),
price: decimal("price", { precision: 10, scale: 2 }),
rating: real("rating"),
active: boolean("active").default(true),
metadata: jsonb("metadata").$type<{ key: string }>(),
createdAt: timestamp("created_at").defaultNow().notNull(),
updatedAt: timestamp("updated_at").defaultNow().notNull(),
```

## Schema Key Rules

1. **Callback overrides for nullable columns:** Every column without `.notNull()` must get `colName: (schema) => schema.optional()` in both insertSchema and selectSchema. This converts `string | null` to `string | undefined` for React Hook Form compatibility.
2. **NEVER use `.transform()`:** Creates `ZodEffects` which breaks `.omit()` and `.partial()` chaining.
3. **NEVER use `z.unknown()` or `z.any()`:** Add callback overrides instead.
4. **NEVER use `serial()` or `bigint` for primary keys:** Only `text("id").primaryKey()` supports client-generated IDs for optimistic updates.
5. **Router output schema MUST mirror table nullability:** Every column WITHOUT `.notNull()` needs `.nullable().optional()`. Omitting `.nullable()` causes silent "Output validation failed".
6. **Form schemas belong in the entity file**, not in `index.ts`.

## Database Client

Always use `createDb()` to create a fresh instance per request. Cloudflare Workers isolate requests -- a singleton `db` causes "Cannot perform I/O on behalf of a different request" errors.

```typescript
import { createDb } from "@template/db";

// Fresh instance inside handler
const db = createDb();
return await db.select().from(users);

// WRONG: Module-level singleton -- breaks in Workers
const db = createDb(); // at top of file
```
