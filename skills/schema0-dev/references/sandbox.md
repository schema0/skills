# Sandbox Commands

The codebase lives on a remote sandbox. All file operations and shell commands go through `schema0 sandbox exec`. Deployment uses `schema0 sandbox deploy`.

## schema0 sandbox exec

Run a shell command on the remote sandbox.

```bash
schema0 sandbox exec "<command>" [--cwd <dir>] [--timeout <ms>]
```

| Option      | Description                    |
| ----------- | ------------------------------ |
| `--cwd`     | Working directory on sandbox   |
| `--timeout` | Timeout in ms (default: 30000) |

Maximum timeout is 300000 (5 minutes). Exit code is forwarded — non-zero causes CLI to exit with failure.

### Common Patterns

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

### Notes

- Always quote the command string to prevent local shell expansion
- Use `--cwd` instead of `cd` inside the command when possible
- For long-running commands (builds, tests), increase `--timeout`
- The sandbox filesystem persists between commands

---

## schema0 sandbox deploy

Build and deploy the application from the remote sandbox.

```bash
schema0 sandbox deploy [options]
```

| Option                 | Description                                 |
| ---------------------- | ------------------------------------------- |
| `--skip-db`            | Skip database migrations                    |
| `--platform`           | Deploy specific platform: `web` or `native` |
| `--include-source-map` | Include source maps in the deployment       |

### What It Does

**Web** (`--platform web`):
1. Generates and runs database migrations (unless `--skip-db`)
2. Builds the web application on the sandbox
3. Deploys the build output

**Native** (`--platform native`):
1. Generates and runs database migrations (unless `--skip-db`)
2. Exports the Expo app with `EXPO_PUBLIC_*` env vars injected
3. Runs `expo-adapter-workers` to prepare for deployment
4. Builds the worker (`build-worker.ts`)
5. Deploys the build output

### Notes

- **Do NOT run build commands manually** — `schema0 sandbox deploy` handles all builds internally
- **Do NOT run `schema0 sandbox exec "bun run build"` before deploying**
- Use `--skip-db` when only frontend code changed
- Use `--platform` to deploy a specific platform

---

## Other CLI Commands

These run directly (not through sandbox exec):

```bash
schema0 whoami                           # Verify authentication
schema0 secrets list                     # List secret names
schema0 secrets set 'KEY=value'          # Set a secret
schema0 secrets delete <name>            # Delete a secret
schema0 doctor                           # Check deployment readiness
schema0 logs                             # View deployment logs
schema0 delete                           # Delete the app and all resources
```
