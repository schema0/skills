# Sandbox File Operations

Use these commands for all file operations on the remote sandbox. Prefer them over `sandbox exec "cat ..."` — they are faster, more reliable, and have structured output.

## sandbox read

Read a single file from the sandbox.

```bash
schema0 sandbox read <path>
```

**Important:**
- Accepts exactly ONE path. Unlike the local Read tool, it does NOT support multiple paths. Call it once per file.
- Path is relative to the project root (absolute paths also work).
- File content is written to stdout verbatim (no truncation, no line numbers).
- Fails with exit code 1 if the file does not exist.

```bash
# correct
schema0 sandbox read packages/db/src/schema/users.ts

# wrong — only the first arg is used, others are ignored or cause an error
schema0 sandbox read file1.ts file2.ts

# correct way to read multiple files — separate calls
schema0 sandbox read packages/db/src/schema/users.ts
schema0 sandbox read packages/api/src/routers/users.ts
```

## sandbox write

Write a file to the sandbox.

```bash
schema0 sandbox write <path> --content "inline content"   # inline
schema0 sandbox write <path>                               # from stdin (heredoc or pipe)
```

- Parent directories are created automatically.
- Path is relative to the project root.
- Overwrites existing files.

```bash
# Inline
schema0 sandbox write packages/db/src/schema/tasks.ts --content "import ..."

# Heredoc for multi-line
schema0 sandbox write packages/db/src/schema/tasks.ts <<'EOF'
import { pgTable, text } from "drizzle-orm/pg-core";
export const tasks = pgTable("tasks", { id: text("id").primaryKey() });
EOF
```

## sandbox ls

List directory tree. Respects `.gitignore`.

```bash
schema0 sandbox ls [path] [-L <depth>]
```

- `path` defaults to the project root.
- `-L <depth>` limits tree depth (1-10).
- Output format: `tree`-style (uses the `tree` command with `--gitignore`).

```bash
schema0 sandbox ls                    # full tree from project root
schema0 sandbox ls -L 2               # depth 2
schema0 sandbox ls packages/db/src    # subtree
```

## sandbox grep

Search file contents on the sandbox.

```bash
schema0 sandbox grep <pattern> [--path <dir>] [--include <glob>]
```

- `pattern` is a grep regex.
- `--path` limits search to a directory (defaults to project root).
- `--include` filters by file glob (e.g., `*.ts`).
- Output: `path:line:match` lines, max 100 results.

```bash
schema0 sandbox grep "createDb" --path packages --include "*.ts"
schema0 sandbox grep "protectedProcedure" --include "*.ts"
```

## When to use sandbox exec instead

Use `sandbox exec` only for shell commands that aren't file reads/writes/listing/searching: builds, tests, installs, git, migrations, typechecking.
