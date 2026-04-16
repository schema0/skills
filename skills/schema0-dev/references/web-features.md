# Web Feature Development

Building CRUD features for the web platform (`apps/web/`). Skip this entirely if `apps/web/` does not exist.

## Implementation Order

For every new feature, follow this sequence:

### 1. Database Schema

Create `packages/db/src/schema/{entity}.ts` with:
- Table definition (plural name, `text("id").primaryKey()`)
- Insert schema (`createInsertSchema` with callback overrides for nullable columns)
- Select schema (`createSelectSchema` with callback overrides)
- Update schema (`selectSchema.partial().required({ id: true })`)
- Form schema (`insertSchema.omit({ id, createdAt, updatedAt })`)
- Edit form schema (`formSchema.partial()` — must NOT include `id`)
- Router output schema (`z.object({...})` — timestamps use `z.date()`, not `z.string()`)

Export from `packages/db/src/schema/index.ts`.

Read the project's `create-db-schema` skill for the complete pattern.

### 2. API Router

Create `packages/api/src/routers/{entity}.ts` with `selectAll`, `selectById`, `insertMany`, `updateMany`, `deleteMany` handlers.

Key rules:
- Use `import { z } from "zod/v4"` — NEVER `import z from "zod"`
- Use `.handler()` for all procedures (NOT `.query()` or `.mutation()`)
- Import schemas from `@template/db/schema` — never define inline
- Use `createDb()` inside each handler — never singleton
- Use `{entity}RouterOutputSchema` for `.output()` validation

Register in `packages/api/src/routers/index.ts`.

Read the project's `api-router` skill for the complete template.

### 3. Frontend

Read and follow the project's skills:
- **query-collections** → Collection, Dialog, Form
- **customize-table** → DataTable columns
- **handle-views** → List Route and Detail Route

Add the route to the sidebar in `apps/web/src/components/app-sidebar.tsx`.

### 4. Typecheck

```bash
schema0 sandbox exec "bunx oxlint --type-check --type-aware --quiet <your-files>"
```

Pass only files you created or modified. Ignore `./+types/` import errors (generated at build time).

### Web Stack

- React Router v7 + TanStack DB
- Web-only skills: query-collections, handle-views, customize-table, create-crud-app-template
- Do NOT use web-only skills if `apps/web/` does not exist
