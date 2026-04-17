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

## File Operations

For file reads, writes, listing, and searching, use the dedicated commands in `references/sandbox-files.md` — do NOT use `sandbox exec "cat ..."`.

## Shell Commands

Use `sandbox exec` for commands that aren't file operations:

```bash
# Typechecking
schema0 sandbox exec "bunx oxlint --type-check --type-aware --quiet <your-files>"

# Running tests
schema0 sandbox exec "NODE_ENV=test bun test web/users.test.tsx" --cwd packages/test

# Installing packages
schema0 sandbox exec "bun add some-package"

# Generating migrations
schema0 sandbox exec "bun drizzle-kit generate" --cwd packages/db

# Git operations
schema0 sandbox exec "git add -A && git commit -m 'message'"
```

## Notes

- Always quote the command string to prevent local shell expansion
- Use `--cwd` instead of `cd` inside the command when possible — relative paths resolve against the project root
- For long-running commands (builds, tests), increase `--timeout`
- The sandbox filesystem persists between commands
