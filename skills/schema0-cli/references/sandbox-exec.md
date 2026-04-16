# sandbox exec

Run a shell command on the remote sandbox.

```bash
schema0 sandbox exec "<command>" [--cwd <dir>] [--timeout <ms>]
```

| Option      | Description                    |
| ----------- | ------------------------------ |
| `--cwd`     | Working directory on sandbox   |
| `--timeout` | Timeout in ms (default: 30000) |

Maximum timeout is 300000 (5 minutes). Exit code is forwarded -- non-zero causes CLI to exit with failure.

## Common Patterns

```bash
# Reading files
schema0 sandbox exec "cat packages/db/src/schema/users.ts"

# Writing files
schema0 sandbox exec "cat > packages/db/src/schema/entity.ts << 'FILEEOF'
import { pgTable, text, timestamp } from \"drizzle-orm/pg-core\";
// ...
FILEEOF"

# Typechecking
schema0 sandbox exec "bunx oxlint --type-check --type-aware --quiet <your-files>"

# Running tests
schema0 sandbox exec "bun test web/users.test.tsx" --cwd packages/test

# Installing packages
schema0 sandbox exec "bun add some-package"

# Generating migrations
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/test
```

## Notes

- Always quote the command string to prevent local shell expansion
- Use `--cwd` instead of `cd` inside the command when possible
- For long-running commands (builds, tests), increase `--timeout`
- The sandbox filesystem persists between commands
