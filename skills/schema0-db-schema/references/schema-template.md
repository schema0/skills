# Full 7-Schema Code Template

**File:** `packages/db/src/schema/{entities}.ts`

ALL schemas -- table, insert, select, update, form, editForm, routerOutputSchema -- are defined in `packages/db/src/schema/{entity}.ts` using drizzle-zod. Routers, collections, and forms import from `@template/db/schema`. Never define `z.object()` schemas manually elsewhere.

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
  userId: text("user_id").notNull(), // server-stamped (see "Server-Stamped Fields" below)
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
  userId: z.string(), // .notNull() column
  createdAt: z.date(), // Date, NOT z.string()
  updatedAt: z.date(), // Date, NOT z.string()
});
```

## Server-Stamped Fields

Some fields are stamped by the server from session context, never sent by the client — most commonly `userId`, `organizationId`, `createdBy`.

Pattern: keep them `.notNull()` in the table (data integrity), but mark them `.optional()` in the insert schema (client omits them).

```typescript
// TABLE: column is .notNull() for data integrity
export const tasks = pgTable("tasks", {
  // ...
  userId: text("user_id").notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// INSERT SCHEMA: mark as optional so clients can omit them
export const insertTasksSchema = createInsertSchema(tasks, {
  userId: (schema) => schema.optional(), // server stamps from session
});

// ROUTER: stamp the value server-side after validation
insertMany: protectedProcedure
  .input(z.object({ tasks: z.array(insertTasksSchema) }))
  .handler(async ({ input, context }) => {
    const userId = context.session.user.id;
    return await db.insert(tasks).values(
      input.tasks.map((t) => ({ ...t, userId })),
    );
  });
```

**Why:** `createInsertSchema` makes `.notNull()` columns required in Zod. If the client omits `userId`, ORPC throws `BAD_REQUEST: Input validation failed` before the handler runs. Marking the field `.optional()` in the insert schema lets the client omit it; the handler then fills it in from the session.

Apply this pattern to any column the server populates (not the client): `userId`, `organizationId`, `createdBy`, `tenantId`, etc.

## Schema Naming Conventions

All names use PLURAL form:

| Schema                         | Derivation                                      | Purpose                       |
| ------------------------------ | ----------------------------------------------- | ----------------------------- |
| `{entities}FormSchema`         | `insertSchema.omit({id, createdAt, updatedAt})` | Create form validation        |
| `{entities}EditFormSchema`     | `formSchema.partial()` (NO `id`)                | Edit form validation          |
| `insert{Entities}Schema`       | `createInsertSchema(table, overrides)`          | Bulk insert validation        |
| `select{Entities}Schema`       | `createSelectSchema(table, overrides)`          | Query/collection validation   |
| `update{Entities}Schema`       | `selectSchema.partial().required({id})`         | Bulk update validation        |
| `{entities}RouterOutputSchema` | `z.object({...})` with DB return types          | Router `.output()` validation |

## Export in index.ts

```typescript
// packages/db/src/schema/index.ts
export * from "./entities";
// Add new entities here as you create them
```

## Migration Workflow

After creating or modifying a schema file, generate and apply migrations:

```bash
# Generate migration files
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/db

# Apply migrations to database
schema0 sandbox exec "bun drizzle-kit migrate" --cwd packages/db

# Generate test migrations (packages/test) -- required before running tests
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test
```

## Transform Utilities

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
