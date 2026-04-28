---
name: schema0-rls
description: Row-level security setup — RLS policies, authenticated database connections, and user-scoped data access
---

# Row-Level Security (RLS) Setup

Database tables with Row-Level Security policies for user-scoped data access.

## When to Use

- Set up database tables with Row-Level Security policies
- Configure authenticated database connections
- Implement user-scoped data access
- Secure database operations with user-based access control

## Workflow

1. Define the entity in `packages/db/src/schema/{entities}.ts` with `crudPolicy`. See `references/patterns.md` for the full schema example (table + 7 zod schemas).
2. Generate and apply migrations with `bun run db:generate && bun run db:migrate`.
3. Create the router in `packages/api/src/routers/{entities}.ts` using `createRLSTransaction`. See `references/implementation.md` for the full router template.
4. Register the router in `packages/api/src/routers/index.ts`.

## 3 Policy Patterns

- **User-Scoped** -- `read: authUid(table.userId), modify: authUid(table.userId)` -- user sees only their own data
- **Public Read, User Write** -- `read: sql\`true\`, modify: authUid(table.userId)` -- all can read, only owner modifies
- **Admin-Only** -- `read: sql\`false\`, modify: sql\`false\`` -- no regular user access

## Key Rules

- ALL RLS operations MUST use `createRLSTransaction(context.request)` -- never raw `createDb()`
- ALWAYS use `protectedProcedure` -- never `publicProcedure` for RLS tables
- ALWAYS set `userId` from `context.session.user.id` on insert
- NEVER perform database operations in route loaders
- Import `crudPolicy`, `authUid` from `drizzle-orm/neon`
- Define `authenticatedUserRole` once in `packages/db/src/schema/index.ts`

## References

- `references/patterns.md` -- Full policy patterns with code (user-scoped, public-read, admin-only), table example, relations
- `references/implementation.md` -- createRLSTransaction, CRUD router example, filtering, migration workflow, debugging
