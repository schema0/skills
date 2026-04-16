# CRUD Orchestration

## Execution Order Diagram

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

## Files Generated Per Entity (10 files)

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

## Post-Creation Steps

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

## Completion Verification

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
