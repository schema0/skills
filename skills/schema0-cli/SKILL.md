---
name: schema0-cli
description: CLI commands for sandbox execution, deployment, version management, secrets, and third-party integrations
---

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

---

## Version Management

Manage preview and production deployments. A **preview** runs your new application code against a database branched from production, letting you test changes with the production schema before going live.

### Commands

```bash
schema0 version list                                    # List all versions
schema0 version preview <commitHash>                    # Create preview from dev deployment
schema0 version deploy <commitHash>                     # Promote preview to production
schema0 version remove <commitHash>                     # Remove a preview
schema0 version confirm-migration <commitHash> --statements "SQL..."  # Apply migration
```

### Typical Flow

```bash
# 1. Check current state
schema0 version list

# 2. Create preview from a dev deployment
schema0 version preview abc1234

# 3. If migration required, review the schema diff and apply SQL
schema0 version confirm-migration abc1234 --statements "ALTER TABLE ...;"

# 4. Deploy preview to production
schema0 version deploy abc1234

# 5. If production migration required
schema0 version confirm-migration abc1234 --production --statements "ALTER TABLE ...;"
```

### Notes

- Schema diffs are returned when migrations are needed — you write the SQL statements
- Preview must exist before deploying to production
- Migration must be completed before a preview can be promoted
- Use `--production` flag on `confirm-migration` when targeting production

---

## Integrations

Discover and execute third-party integrations via connected platforms (Gmail, Slack, Stripe, etc.).

### Commands

```bash
schema0 integrations connections                           # List connected platforms
schema0 integrations connections --platform gmail          # Filter by platform
schema0 integrations search <platform> "<query>"           # Search actions (natural language)
schema0 integrations details <systemId>                    # Get action parameters and docs
schema0 integrations execute <connectionKey> <path> [opts] # Execute an action
```

### Typical Workflow

```bash
# 1. See what platforms are connected
schema0 integrations connections

# 2. Search for actions using natural language
schema0 integrations search gmail "send email"
schema0 integrations search slack "send message"

# 3. Get details about a specific action
schema0 integrations details <systemId>

# 4. Execute the action
schema0 integrations execute <connectionKey> /v0/endpoint --method GET --action-id <systemId>
```

### Key Concepts

- **Connection key**: Identifies which authenticated platform account to use (from `connections` output)
- **System ID**: Unique identifier for an action (from `search` output)
- **Search query**: Natural language description of what you want to do
- **Action path**: The API path for the passthrough request (from `search` or `details` output)

### Notes

- The `search` query is natural language — describe the action in plain English
- Always run `details` before `execute` to understand required parameters
- Path parameters use `{{param}}` or `:param` format and are substituted automatically via `--path-params`
