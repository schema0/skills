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

## Quick Start: Generate RLS Router

```bash
schema0 sandbox exec "bun run scaffold-scripts/generate.ts rls-service <name>"
```

Generated output: `packages/api/src/routers/[name].ts` -- ORPC router with `protectedProcedure`, `createRLSTransaction`, CRUD with RLS.

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
