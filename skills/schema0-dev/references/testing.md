# Testing

Testing is mandatory — every feature MUST have tests. A feature is NOT complete without a passing test file. NEVER gut tests to make them pass — fix the source code instead.

## Test Infrastructure

| Type   | Location                   | Runner                               |
| ------ | -------------------------- | ------------------------------------ |
| Web    | `packages/test/web/{entity}.test.tsx`    | bun:test + HappyDOM    |
| Mobile | `packages/test/mobile/{entity}.test.tsx` | Jest + @testing-library/react-native |

All tests run against a real in-process PGlite database — no mocks for the DB layer.

## Before Writing Tests

1. Read `packages/test/CLAUDE.md` (or equivalent for your agent) in full
2. Generate test DB migrations if schema changed:
   ```bash
   schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test
   ```

## Web Tests

Skip if `apps/web/` does not exist.

Create `packages/test/web/{entity}.test.tsx` with create, update, and delete tests.

```bash
schema0 sandbox exec "bun test web/{entity}.test.tsx" --cwd packages/test
```

## Mobile Tests

Skip if `apps/native/` does not exist.

Create `packages/test/mobile/{entity}.test.tsx` with create, update, and delete tests.

```bash
schema0 sandbox exec "node mobile/build-orpc.js" --cwd packages/test
schema0 sandbox exec "NODE_OPTIONS='--experimental-vm-modules' npx jest --config jest.config.js --forceExit mobile/{entity}.test.tsx" --cwd packages/test --timeout 120000
```

## Key Rules

- Always run `bun drizzle-kit generate` from `packages/test/` when schema changes — test infra reads from `packages/test/drizzle/`
- Run `bun test` from `packages/test/` — `bunfig.toml` preload path resolves relative to CWD
- All tests must pass before proceeding to deployment
