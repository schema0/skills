# Web Feature Development

Building CRUD features for the web platform (`apps/web/`). Skip this entirely if `apps/web/` does not exist.

## Web Stack

- React Router v7 + TanStack DB + TanStack React Table
- oRPC for API (NOT tRPC)
- Drizzle ORM + drizzle-zod for schema
- shadcn/ui components (Dialog, AlertDialog, DataTable, Form)
- react-hook-form + zodResolver for forms
- `import { z } from "zod/v4"` everywhere -- NEVER `import z from "zod"`

---

## Implementation Order

For every new entity, create files in this exact sequence:

1. Database Schema (`packages/db/src/schema/{entities}.ts`)
2. API Router (`packages/api/src/routers/{entities}.ts`)
3. Query Collection (`apps/web/src/query-collections/custom/{entities}.ts`)
4. Dialog (`apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Dialog.tsx`)
5. Form (`apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Form.tsx`)
6. Table Columns (`apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Column.tsx`)
7. Component Index (`apps/web/src/components/ui/data-table/custom/{entities}/index.ts`)
8. List Route (`apps/web/src/routes/_auth.{entities}.tsx`)
9. Detail Route (`apps/web/src/routes/_auth.{entities}_.$id.tsx`)
10. Sidebar entry (`apps/web/src/components/app-sidebar.tsx`)

---

## 1. Database Schema

**File:** `packages/db/src/schema/{entities}.ts`

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

---

## 2. API Router

**File:** `packages/api/src/routers/{entities}.ts`

Every router provides 5 bulk CRUD operations: `selectAll`, `selectById`, `insertMany`, `updateMany`, `deleteMany`.

### Full Template

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

---

## 3. Query Collections

**File:** `apps/web/src/query-collections/custom/{entities}.ts`

Collections bridge the API router to TanStack DB for optimistic UI updates. Each collection defines `onInsert`, `onUpdate`, `onDelete` mutation handlers and a `queryFn` for fetching data.

### Full Collection Template

```typescript
// apps/web/src/query-collections/custom/entities.ts
import { queryCollectionOptions } from "@tanstack/react-db";
import { client, queryClient } from "@/utils/orpc";
import {
  selectEntitiesSchema,
  insertEntitiesSchema,
  updateEntitiesSchema,
} from "@template/db/schema";

export const entitiesCollection = queryCollectionOptions({
  id: "entities",
  queryKey: ["entities"],
  queryFn: async (ctx) => {
    return await client.entities.selectAll({});
  },
  getId: (item) => item.id,
  schema: selectEntitiesSchema,
  onInsert: async (tx) => {
    const records = tx.insertions.map((item) => ({
      id: item.id,
      name: item.name,
      description: item.description ?? null,
      email: item.email,
      price: item.price ?? null,
      active: item.active ?? true,
      userId: item.userId ?? null,
      createdAt: item.createdAt ?? new Date(),
      updatedAt: item.updatedAt ?? new Date(),
    }));
    await client.entities.insertMany(records);
    await queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
  onUpdate: async (tx) => {
    const updates = tx.updates.map((u) => ({
      id: u.id,
      ...u.changes,
    }));
    await client.entities.updateMany(updates);
    await queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
  onDelete: async (tx) => {
    const ids = tx.deletions.map((item) => ({ id: item.id }));
    await client.entities.deleteMany(ids);
    await queryClient.invalidateQueries({ queryKey: ["entities"] });
  },
  queryClient,
});
```

### Collection Key Rules

- `id` and `queryKey` must match each other and match the `invalidateQueries` call in tests.
- `queryFn` MUST use `client.{entity}.selectAll({})` directly -- never `fetchCustomResources`.
- `schema` uses `selectEntitiesSchema` (the select schema from the DB package).
- `onInsert` maps form data to the full record shape including system fields.
- `onUpdate` maps update data and always includes `id`.
- `onDelete` maps items to `{ id }` objects.
- Always `invalidateQueries` after mutations to refresh the cache.

### Dialog Template

**File:** `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Dialog.tsx`

The Dialog wraps the Form component, handles create vs edit mode, and calls the collection's insert/update.

```tsx
// apps/web/src/components/ui/data-table/custom/entities/EntitiesDialog.tsx
import { useState } from "react";
import { useCollection } from "@tanstack/react-db";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { entitiesCollection } from "@/query-collections/custom/entities";
import { EntitiesForm } from "./EntitiesForm";
import type { z } from "zod/v4";
import type {
  entitiesFormSchema,
  entitiesEditFormSchema,
} from "@template/db/schema";

interface EntitiesDialogProps {
  mode: "create" | "edit";
  entityData?: Record<string, unknown>; // existing data for edit mode
  trigger?: React.ReactNode;
  userId: string;
}

export function EntitiesDialog({
  mode,
  entityData,
  trigger,
  userId,
}: EntitiesDialogProps) {
  const [open, setOpen] = useState(false);
  const collection = useCollection(entitiesCollection);

  const handleCreate = (data: z.infer<typeof entitiesFormSchema>) => {
    collection.insert([
      {
        ...data,
        id: crypto.randomUUID(),
        userId,
        createdAt: new Date(),
        updatedAt: new Date(),
      },
    ]);
    setOpen(false);
  };

  const handleEdit = (data: z.infer<typeof entitiesEditFormSchema>) => {
    if (!entityData?.id) return;
    collection.update(entityData.id as string, {
      ...data,
      updatedAt: new Date(),
    });
    setOpen(false);
  };

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        {trigger ?? (
          <Button>{mode === "create" ? "Add Entity" : "Edit"}</Button>
        )}
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>
            {mode === "create" ? "Create Entity" : "Edit Entity"}
          </DialogTitle>
        </DialogHeader>
        <EntitiesForm
          mode={mode}
          defaultValues={mode === "edit" ? entityData : undefined}
          onSubmit={mode === "create" ? handleCreate : handleEdit}
        />
      </DialogContent>
    </Dialog>
  );
}
```

### Form Template

**File:** `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Form.tsx`

The Form uses react-hook-form with zodResolver. It handles both create and edit modes via schema switching.

```tsx
// apps/web/src/components/ui/data-table/custom/entities/EntitiesForm.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import type { z } from "zod/v4";
import {
  entitiesFormSchema,
  entitiesEditFormSchema,
} from "@template/db/schema";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Textarea } from "@/components/ui/textarea";
import { Switch } from "@/components/ui/switch";

type CreateFormData = z.infer<typeof entitiesFormSchema>;
type EditFormData = z.infer<typeof entitiesEditFormSchema>;

interface EntitiesFormProps {
  mode: "create" | "edit";
  defaultValues?: Record<string, unknown>;
  onSubmit: (data: CreateFormData | EditFormData) => void;
}

export function EntitiesForm({
  mode,
  defaultValues,
  onSubmit,
}: EntitiesFormProps) {
  const schema =
    mode === "create" ? entitiesFormSchema : entitiesEditFormSchema;

  const form = useForm({
    resolver: zodResolver(schema),
    defaultValues: (defaultValues ?? {}) as Record<string, unknown>,
  });

  const handleSubmit = form.handleSubmit(
    (data) => {
      onSubmit(data as CreateFormData | EditFormData);
    },
    (errors) => {
      // onInvalid callback -- logs form validation errors for debugging
      console.error("Form validation errors:", errors);
    },
  );

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {/* Name field -- required in create, optional in edit */}
      <div className="space-y-2">
        <Label htmlFor="name">Name</Label>
        <Input id="name" {...form.register("name")} placeholder="Enter name" />
        {form.formState.errors.name && (
          <p className="text-sm text-destructive">
            {form.formState.errors.name.message as string}
          </p>
        )}
      </div>

      {/* Email field -- only shown in create mode (omitted from edit schema) */}
      {mode === "create" && (
        <div className="space-y-2">
          <Label htmlFor="email">Email</Label>
          <Input
            id="email"
            type="email"
            {...form.register("email")}
            placeholder="Enter email"
          />
          {form.formState.errors.email && (
            <p className="text-sm text-destructive">
              {form.formState.errors.email.message as string}
            </p>
          )}
        </div>
      )}

      {/* Description field -- optional textarea */}
      <div className="space-y-2">
        <Label htmlFor="description">Description</Label>
        <Textarea
          id="description"
          {...form.register("description")}
          placeholder="Enter description"
        />
      </div>

      {/* Active toggle -- boolean with Switch */}
      <div className="flex items-center space-x-2">
        <Switch
          id="active"
          checked={form.watch("active") as boolean}
          onCheckedChange={(checked) => form.setValue("active", checked)}
        />
        <Label htmlFor="active">Active</Label>
      </div>

      <Button type="submit">{mode === "create" ? "Create" : "Save"}</Button>
    </form>
  );
}
```

### Form Key Rules

- `zodResolver(schema)` uses the correct schema per mode: `formSchema` for create, `editFormSchema` for edit.
- `handleSubmit` MUST include the `onInvalid` callback for debugging: `form.handleSubmit(onValid, onInvalid)`.
- The edit form schema MUST NOT include `id` -- the Dialog adds it after submission.
- Default values for edit mode come from the existing row data.
- The `"Form validation errors:"` console message is used by tests (Layer 1 validation).

### Index Export

```typescript
// apps/web/src/components/ui/data-table/custom/entities/index.ts
export { EntitiesDialog } from "./EntitiesDialog";
export { EntitiesForm } from "./EntitiesForm";
export { entitiesColumns, entitiesBaseColumns } from "./EntitiesColumn";
```

---

## 4. Table Columns

**File:** `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Column.tsx`

Columns define how data appears in the DataTable and include the actions column with edit/delete buttons.

### Full Columns Template

```tsx
// apps/web/src/components/ui/data-table/custom/entities/EntitiesColumn.tsx
import type { ColumnDef } from "@tanstack/react-table";
import type { z } from "zod/v4";
import type { selectEntitiesSchema } from "@template/db/schema";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Pencil, Trash2 } from "lucide-react";

type Entity = z.infer<typeof selectEntitiesSchema>;

interface EntitiesColumnMeta {
  onEdit?: (entity: Entity) => void;
  onDelete?: (entity: Entity) => void;
}

// Base columns -- data display without actions
export const entitiesBaseColumns: ColumnDef<Entity>[] = [
  {
    accessorKey: "name",
    header: "Name",
  },
  {
    accessorKey: "email",
    header: "Email",
  },
  {
    accessorKey: "description",
    header: "Description",
    cell: ({ row }) => {
      const value = row.getValue("description") as string | null;
      return value ?? "—";
    },
  },
  {
    accessorKey: "active",
    header: "Status",
    cell: ({ row }) => {
      const active = row.getValue("active") as boolean | null;
      return (
        <Badge variant={active ? "default" : "secondary"}>
          {active ? "Active" : "Inactive"}
        </Badge>
      );
    },
  },
  {
    accessorKey: "createdAt",
    header: "Created",
    cell: ({ row }) => {
      const date = row.getValue("createdAt") as Date;
      return date ? new Date(date).toLocaleDateString() : "—";
    },
  },
];

// Full columns with actions -- used in the list view
export function entitiesColumns(meta: EntitiesColumnMeta): ColumnDef<Entity>[] {
  return [
    ...entitiesBaseColumns,
    {
      id: "actions",
      header: "Actions",
      cell: ({ row }) => (
        <div className="flex items-center gap-2">
          <Button
            variant="ghost"
            size="icon"
            aria-label="Edit"
            onClick={() => meta.onEdit?.(row.original)}
          >
            <Pencil className="h-4 w-4" />
          </Button>
          <Button
            variant="ghost"
            size="icon"
            aria-label="Delete"
            onClick={() => meta.onDelete?.(row.original)}
          >
            <Trash2 className="h-4 w-4" />
          </Button>
        </div>
      ),
    },
  ];
}
```

### Column Customization Examples

Common cell renderers for different data types:

```tsx
// Date column
{
  accessorKey: "createdAt",
  header: "Created",
  cell: ({ row }) => {
    const date = row.getValue("createdAt") as Date;
    return date ? new Date(date).toLocaleDateString() : "—";
  },
}

// Status badge (boolean)
{
  accessorKey: "active",
  header: "Status",
  cell: ({ row }) => {
    const active = row.getValue("active") as boolean;
    return (
      <Badge variant={active ? "default" : "secondary"}>
        {active ? "Active" : "Inactive"}
      </Badge>
    );
  },
}

// Price / decimal
{
  accessorKey: "price",
  header: "Price",
  cell: ({ row }) => {
    const price = row.getValue("price") as string | null;
    return price ? `$${parseFloat(price).toFixed(2)}` : "—";
  },
}

// Email link
{
  accessorKey: "email",
  header: "Email",
  cell: ({ row }) => {
    const email = row.getValue("email") as string;
    return <a href={`mailto:${email}`} className="text-primary underline">{email}</a>;
  },
}

// URL link
{
  accessorKey: "website",
  header: "Website",
  cell: ({ row }) => {
    const url = row.getValue("website") as string | null;
    return url ? (
      <a href={url} target="_blank" rel="noopener noreferrer" className="text-primary underline">
        {new URL(url).hostname}
      </a>
    ) : "—";
  },
}

// Avatar with fallback
{
  accessorKey: "avatarUrl",
  header: "Avatar",
  cell: ({ row }) => {
    const url = row.getValue("avatarUrl") as string | null;
    const name = row.getValue("name") as string;
    return url ? (
      <img src={url} alt={name} className="h-8 w-8 rounded-full" />
    ) : (
      <div className="flex h-8 w-8 items-center justify-center rounded-full bg-muted text-xs">
        {name?.charAt(0)?.toUpperCase()}
      </div>
    );
  },
}

// Truncated text
{
  accessorKey: "description",
  header: "Description",
  cell: ({ row }) => {
    const text = row.getValue("description") as string | null;
    if (!text) return "—";
    return text.length > 50 ? `${text.slice(0, 50)}...` : text;
  },
}
```

### Column Key Rules

- Actions column uses `aria-label="Edit"` and `aria-label="Delete"` -- these labels are used by tests to find buttons.
- Delete button calls `onDelete?.(row.original)` -- passes the full object, NOT just `row.original.id`.
- NEVER use `confirm()` for delete -- use `onDelete` callback which triggers an AlertDialog in the view.
- The `meta` pattern keeps columns pure; the view provides the callbacks.

---

## 5. Views (Route Components)

### List Route

**File:** `apps/web/src/routes/_auth.{entities}.tsx`

The list route is the main page for an entity. It uses `useLiveQuery` for real-time data, DataTable for display, Dialog for create/edit, and AlertDialog for delete confirmation.

**CRITICAL:** The `loader` function MUST be exported as a named export. React Router v7 requires it.

```tsx
// apps/web/src/routes/_auth.entities.tsx
import { useState } from "react";
import { useLiveQuery } from "@tanstack/react-db";
import { useCollection } from "@tanstack/react-db";
import { DataTable } from "@/components/ui/data-table";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { authContext } from "@/context";
import { entitiesCollection } from "@/query-collections/custom/entities";
import { EntitiesDialog } from "@/components/ui/data-table/custom/entities";
import { entitiesColumns } from "@/components/ui/data-table/custom/entities";
import type { Route } from "./+types/_auth.entities";
import type { z } from "zod/v4";
import type { selectEntitiesSchema } from "@template/db/schema";

type Entity = z.infer<typeof selectEntitiesSchema>;

// MUST be exported -- React Router v7 requires named export
export function loader({ context }: Route.LoaderArgs) {
  const auth = authContext.get(context);
  return { user: auth.user };
}

export default function EntitiesPage({ loaderData }: Route.ComponentProps) {
  const { user } = loaderData;

  // Live query -- reacts to optimistic updates in real time
  const { rows: entities } = useLiveQuery((query) =>
    query.from({ entities: entitiesCollection }),
  );

  const collection = useCollection(entitiesCollection);

  // Edit dialog state
  const [editEntity, setEditEntity] = useState<Entity | null>(null);
  const [editOpen, setEditOpen] = useState(false);

  // Delete confirmation state
  const [deleteEntity, setDeleteEntity] = useState<Entity | null>(null);
  const [deleteOpen, setDeleteOpen] = useState(false);

  const handleEdit = (entity: Entity) => {
    setEditEntity(entity);
    setEditOpen(true);
  };

  const handleDelete = (entity: Entity) => {
    setDeleteEntity(entity);
    setDeleteOpen(true);
  };

  const confirmDelete = () => {
    if (deleteEntity) {
      collection.delete(deleteEntity.id);
    }
    setDeleteOpen(false);
    setDeleteEntity(null);
  };

  const columns = entitiesColumns({
    onEdit: handleEdit,
    onDelete: handleDelete,
  });

  return (
    <div className="container mx-auto py-6 space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Entities</h1>
        <EntitiesDialog mode="create" userId={user.id} />
      </div>

      <DataTable columns={columns} data={entities} />

      {/* Edit Dialog -- controlled via state */}
      {editEntity && (
        <Dialog open={editOpen} onOpenChange={setEditOpen}>
          <DialogContent>
            <DialogHeader>
              <DialogTitle>Edit Entity</DialogTitle>
            </DialogHeader>
            <EntitiesForm
              mode="edit"
              defaultValues={editEntity as Record<string, unknown>}
              onSubmit={(data) => {
                collection.update(editEntity.id, {
                  ...data,
                  updatedAt: new Date(),
                });
                setEditOpen(false);
                setEditEntity(null);
              }}
            />
          </DialogContent>
        </Dialog>
      )}

      {/* Delete Confirmation */}
      <AlertDialog open={deleteOpen} onOpenChange={setDeleteOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Delete Entity</AlertDialogTitle>
            <AlertDialogDescription>
              Are you sure you want to delete "{deleteEntity?.name}"? This
              action cannot be undone.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction onClick={confirmDelete}>
              Delete
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  );
}
```

### Detail Route

**File:** `apps/web/src/routes/_auth.{entities}_.$id.tsx`

The detail route shows a single entity by ID. It loads the entity via the loader and displays its fields.

```tsx
// apps/web/src/routes/_auth.entities_.$id.tsx
import { authContext } from "@/context";
import { client } from "@/utils/orpc";
import type { Route } from "./+types/_auth.entities_.$id";

// MUST be exported
export async function loader({ context, params }: Route.LoaderArgs) {
  const auth = authContext.get(context);
  const entity = await client.entities.selectById({ id: params.id });
  if (!entity) {
    throw new Response("Not Found", { status: 404 });
  }
  return { user: auth.user, entity };
}

export default function EntityDetailPage({ loaderData }: Route.ComponentProps) {
  const { entity } = loaderData;

  return (
    <div className="container mx-auto py-6 space-y-4">
      <h1 className="text-2xl font-bold">{entity.name}</h1>
      <div className="grid gap-4">
        <div>
          <span className="text-muted-foreground">Email:</span> {entity.email}
        </div>
        {entity.description && (
          <div>
            <span className="text-muted-foreground">Description:</span>{" "}
            {entity.description}
          </div>
        )}
        <div>
          <span className="text-muted-foreground">Status:</span>{" "}
          {entity.active ? "Active" : "Inactive"}
        </div>
        <div>
          <span className="text-muted-foreground">Created:</span>{" "}
          {new Date(entity.createdAt).toLocaleDateString()}
        </div>
      </div>
    </div>
  );
}
```

### View Key Rules

1. **`loader` MUST be exported** as a named export -- React Router v7 requires this.
2. **Use `authContext.get(context)`** to access the authenticated user in loaders.
3. **Use `useLiveQuery`** from `@tanstack/react-db` for real-time data in list views.
4. **Use `useCollection`** from `@tanstack/react-db` for mutation operations (insert, update, delete).
5. **Delete uses AlertDialog** (2-step: row button opens AlertDialog, confirm button executes). NEVER use `window.confirm()`.
6. **Never perform database operations in loaders** -- loaders only call ORPC client methods.
7. **Ignore `./+types/` import errors** -- these types are generated at build time.

---

## 6. Sidebar Entry

After creating all files, add the entity to the sidebar:

```tsx
// apps/web/src/components/app-sidebar.tsx
// Add to the navigation items array:
{
  title: "Entities",
  url: "/entities",
  icon: SomeIcon,  // from lucide-react
}
```

---

## 7. CRUD Orchestration

### Execution Order Diagram

```
1. Schema    packages/db/src/schema/{entities}.ts
     |
     v
2. Router    packages/api/src/routers/{entities}.ts
     |       + register in packages/api/src/routers/index.ts
     v
3. Migrate   schema0 sandbox exec "bun drizzle-kit generate"
     |       schema0 sandbox exec "bun drizzle-kit migrate"
     v
4. Collection  apps/web/src/query-collections/custom/{entities}.ts
     |
     v
5. Dialog    apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Dialog.tsx
     |
     v
6. Form      apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Form.tsx
     |
     v
7. Columns   apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Column.tsx
     |
     v
8. Index     apps/web/src/components/ui/data-table/custom/{entities}/index.ts
     |
     v
9. List      apps/web/src/routes/_auth.{entities}.tsx
     |
     v
10. Detail   apps/web/src/routes/_auth.{entities}_.$id.tsx
     |
     v
11. Sidebar  apps/web/src/components/app-sidebar.tsx
     |
     v
12. Schema index  packages/db/src/schema/index.ts  (add export)
```

### Files Generated Per Entity (10 files)

| #   | File            | Location                                                                       |
| --- | --------------- | ------------------------------------------------------------------------------ |
| 1   | Schema          | `packages/db/src/schema/{entities}.ts`                                         |
| 2   | Router          | `packages/api/src/routers/{entities}.ts`                                       |
| 3   | Collection      | `apps/web/src/query-collections/custom/{entities}.ts`                          |
| 4   | Dialog          | `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Dialog.tsx` |
| 5   | Form            | `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Form.tsx`   |
| 6   | Columns         | `apps/web/src/components/ui/data-table/custom/{entities}/{Entities}Column.tsx` |
| 7   | Component Index | `apps/web/src/components/ui/data-table/custom/{entities}/index.ts`             |
| 8   | List Route      | `apps/web/src/routes/_auth.{entities}.tsx`                                     |
| 9   | Detail Route    | `apps/web/src/routes/_auth.{entities}_.$id.tsx`                                |
| 10  | Test            | `packages/test/web/{entities}.test.tsx`                                        |

Plus modifications to:

- `packages/db/src/schema/index.ts` (add export)
- `packages/api/src/routers/index.ts` (register router)
- `apps/web/src/components/app-sidebar.tsx` (add nav item)

### Post-Creation Steps

```bash
# 1. Generate database migrations
schema0 sandbox exec "bun drizzle-kit generate"
schema0 sandbox exec "bun drizzle-kit migrate"

# 2. Generate test migrations
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test

# 3. Typecheck (pass only your new files)
schema0 sandbox exec "bunx oxlint --type-check --type-aware --quiet packages/db/src/schema/entities.ts packages/api/src/routers/entities.ts"

# 4. Run tests
schema0 sandbox exec "bun test web/entities.test.tsx" --cwd packages/test

# 5. Deploy
schema0 sandbox deploy
```

### Completion Verification

A feature is complete when ALL of the following are true:

1. All 10 files exist and compile
2. Schema exports 7 schemas (table, insert, select, update, form, editForm, routerOutput)
3. Router has 5 operations (selectAll, selectById, insertMany, updateMany, deleteMany)
4. Router is registered in `packages/api/src/routers/index.ts`
5. Schema is exported from `packages/db/src/schema/index.ts`
6. Migrations generated and applied
7. Tests pass: create, update, delete via UI
8. Sidebar entry added
9. Deployed successfully

---

## Global Rules Checklist

- Use PLURAL entity names everywhere (`entities`, not `entity`)
- Use `import { z } from "zod/v4"` -- NEVER `import z from "zod"`
- No `any` types, no typecheck suppression (`@ts-ignore`, `@ts-expect-error`, `@ts-nocheck`, `eslint-disable`)
- Never hand-write migration files -- always `drizzle-kit generate`
- Use `createDb()` per request -- never singleton
- Testing is mandatory -- minimum 3 CRUD tests per entity
