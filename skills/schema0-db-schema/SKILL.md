---
name: schema0-db-schema
description: Database schema generation with Drizzle ORM — table definitions, derived schemas, migrations, and transform utilities
---

# Database Schema

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

---

## Schema Definition (Single Source of Truth)

**File:** `packages/db/src/schema/{entities}.ts`

ALL schemas — table, insert, select, update, form, editForm, routerOutputSchema — are defined in `packages/db/src/schema/{entity}.ts` using drizzle-zod. Routers, collections, and forms import from `@template/db/schema`. Never define `z.object()` schemas manually elsewhere.

Every entity file defines exactly 7 schemas: table, insertSchema, selectSchema, updateSchema, formSchema, editFormSchema, routerOutputSchema.

### Full Template

```typescript
// packages/db/src/schema/entities.ts
import {
  pgTable,
  text,
  boolean,
  timestamp,
  integer,
  decimal,
} from "drizzle-orm/pg-core";
import { createInsertSchema, createSelectSchema } from "drizzle-zod";
import { z } from "zod/v4";

// ──────────────────────────────────────────────
// 1. TABLE DEFINITION
// ──────────────────────────────────────────────
export const entities = pgTable("entities", {
  id: text("id").primaryKey(), // ALWAYS text -- enables optimistic updates
  name: text("name").notNull(),
  description: text("description"), // Nullable (string | null in DB)
  email: text("email").notNull(),
  price: decimal("price", { precision: 10, scale: 2 }),
  active: boolean("active").default(true),
  userId: text("user_id"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

// ──────────────────────────────────────────────
// 2. INSERT SCHEMA
// ──────────────────────────────────────────────
// ALWAYS use callback overrides (schema) => ... for nullable columns
// NEVER use direct z.string().optional()
export const insertEntitiesSchema = createInsertSchema(entities, {
  name: (schema) => schema.min(1).max(200),
  email: (schema) => schema.email(),
  description: (schema) => schema.optional(), // nullable -> optional for RHF
  price: (schema) => schema.optional(),
  active: (schema) => schema.optional(),
  userId: (schema) => schema.optional(),
});

// ──────────────────────────────────────────────
// 3. SELECT SCHEMA
// ──────────────────────────────────────────────
export const selectEntitiesSchema = createSelectSchema(entities, {
  description: (schema) => schema.optional(),
  price: (schema) => schema.optional(),
  active: (schema) => schema.optional(),
  userId: (schema) => schema.optional(),
});

// ──────────────────────────────────────────────
// 4. UPDATE SCHEMA
// ──────────────────────────────────────────────
export const updateEntitiesSchema = selectEntitiesSchema
  .partial()
  .required({ id: true });

// ──────────────────────────────────────────────
// 5. FORM SCHEMA (create mode -- omits system fields)
// ──────────────────────────────────────────────
export const entitiesFormSchema = insertEntitiesSchema.omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

// ──────────────────────────────────────────────
// 6. EDIT FORM SCHEMA (edit mode -- partial, NO id)
// ──────────────────────────────────────────────
// MUST NOT include id -- Dialog adds id after submission
// Optionally omit immutable fields (e.g. email)
export const entitiesEditFormSchema = entitiesFormSchema
  .omit({ email: true })
  .partial();

// ──────────────────────────────────────────────
// 7. ROUTER OUTPUT SCHEMA (for .output() validation)
// ──────────────────────────────────────────────
// timestamp() returns Date objects, NOT strings
// Every column WITHOUT .notNull() needs .nullable().optional()
export const entitiesRouterOutputSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string().nullable().optional(),
  email: z.string(),
  price: z.string().nullable().optional(), // decimal returns string
  active: z.boolean().nullable().optional(),
  userId: z.string().nullable().optional(),
  createdAt: z.date(), // Date, NOT z.string()
  updatedAt: z.date(), // Date, NOT z.string()
});
```

### Schema Key Rules

1. **Callback overrides for nullable columns:** Every column without `.notNull()` must get `colName: (schema) => schema.optional()` in both insertSchema and selectSchema. This converts `string | null` to `string | undefined` for React Hook Form compatibility.
2. **NEVER use `.transform()`:** Creates `ZodEffects` which breaks `.omit()` and `.partial()` chaining.
3. **NEVER use `z.unknown()` or `z.any()`:** Add callback overrides instead.
4. **NEVER use `serial()` or `bigint` for primary keys:** Only `text("id").primaryKey()` supports client-generated IDs for optimistic updates.
5. **Router output schema MUST mirror table nullability:** Every column WITHOUT `.notNull()` needs `.nullable().optional()`. Omitting `.nullable()` causes silent "Output validation failed".
6. **Form schemas belong in the entity file**, not in `index.ts`.

### Schema Naming Conventions

All names use PLURAL form:

| Schema                         | Derivation                                      | Purpose                       |
| ------------------------------ | ----------------------------------------------- | ----------------------------- |
| `{entities}FormSchema`         | `insertSchema.omit({id, createdAt, updatedAt})` | Create form validation        |
| `{entities}EditFormSchema`     | `formSchema.partial()` (NO `id`)                | Edit form validation          |
| `insert{Entities}Schema`       | `createInsertSchema(table, overrides)`          | Bulk insert validation        |
| `select{Entities}Schema`       | `createSelectSchema(table, overrides)`          | Query/collection validation   |
| `update{Entities}Schema`       | `selectSchema.partial().required({id})`         | Bulk update validation        |
| `{entities}RouterOutputSchema` | `z.object({...})` with DB return types          | Router `.output()` validation |

### Column Types Reference

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

### Export in index.ts

```typescript
// packages/db/src/schema/index.ts
export * from "./entities";
// Add new entities here as you create them
```

### Migration Workflow

After creating or modifying a schema file, generate and apply migrations:

```bash
# Generate migration files (packages/db)
schema0 sandbox exec "bun drizzle-kit generate"

# Apply migrations to database
schema0 sandbox exec "bun drizzle-kit migrate"

# Generate test migrations (packages/test) -- required before running tests
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test
```

### Transform Utilities

Form-to-Database pipeline utilities for collection onInsert:

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
