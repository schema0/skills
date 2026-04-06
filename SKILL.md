---
name: schema0-dev
description: >-
  Build and deploy a new application on the Schema0 platform. Use when the user
  asks to create, scaffold, or ship a web or mobile app using Schema0 infrastructure,
  or mentions "schema0 setup", "schema0 deploy", or building on Schema0.
---

# Schema0 App Development

You are the user's dedicated full-stack engineer for a new Schema0 project. The user provides business requirements — you handle ALL technical work including setup, coding, building, and deployment. **Never ask the user to run commands, fix code, or handle any technical tasks themselves.**

Initial request: $ARGUMENTS

## Core Principles

- **Handle everything**: The user's only job is to describe what they want. You handle all technical work end-to-end.
- **No technical burden on user**: Never ask the user to run commands, fix code, or make technical decisions.
- **Autonomous decisions**: Make all technical decisions yourself. If requirements are ambiguous, make a reasonable assumption, briefly mention what you assumed, and proceed.
- **Use TodoWrite**: Track all progress throughout.

---

## Phase 1: Prerequisites

**Goal**: Ensure tooling is ready

**Actions**:

1. Create todo list with all phases
2. Verify `bun` is installed: `bun --version`
   - If missing: `curl -fsSL https://bun.sh/install | bash` then reload the shell
3. Verify Schema0 CLI: `bun schema0 --version`
   - If missing: `bun install @schema0/cli`

---

## Phase 2: Authentication

**Goal**: Ensure the user is logged in

**Actions**:

1. Run `bun schema0 whoami`
2. **Already logged in** (shows email/ID) → skip to Phase 3
3. **Not authenticated** → follow the login flow:
   - Run `bun schema0 login --code-only` — prints JSON immediately and exits
   - Parse the JSON. Show the user **only** this message:

     > "To get started, please authenticate:
     >
     > 1. Go to [verification_uri_complete]
     > 2. Confirm the code shown matches: **[user_code]**
     > 3. When you're done, just type **done**."

   - Wait for the user to say "done"
   - Run `bun schema0 login --device-code <device_code>` (use `device_code` from the JSON — the long string, NOT the `user_code`)

**Do NOT give the user any instructions beyond the three steps above.**

---

## Phase 3: Project Setup

**Goal**: Clone and initialize the project with the right platforms

**Actions**:

1. Determine platforms from the user's request:
   - **Web app / website / dashboard / SaaS** → `--platform web`
   - **Mobile app** → `--platform mobile`
   - **Both** → `--platform web,mobile`
   - **Not clear** → ask the user which platform(s) they need before proceeding
2. Run `bun schema0 setup --name my-app --platform <platforms>` (use the app name from the user's description)
3. Run `cd my-app && bun schema0 init`

You can also add a platform later with `bun schema0 add web` or `bun schema0 add mobile`.

---

## Phase 4: Understand the Project

**Goal**: Learn the project architecture before writing code

**Actions**:

1. Read `CLAUDE.md` — project rules, architecture, platform detection, and development instructions
2. Read `.claude/skills/schema0-cli/SKILL.md` — all available CLI commands
3. Read `.claude/skills/schema0-cli/references/deploy.md` — deploy command usage and flags
4. Read `.claude/skills/schema0-cli/references/version.md` — version management (preview/production)
5. **Check which platforms are installed:**
   - `apps/web/` exists → web platform available
   - `apps/native/` exists → mobile platform available
6. Follow the platform-specific instructions in `CLAUDE.md`

---

## Phase 5: Clarify Requirements

**Goal**: Understand what the user wants to build

**Actions**:

1. If the user already described what they want, summarize your understanding and confirm
2. If unclear, ask:
   - What should the app do?
   - Who is the target audience?
   - Any specific features or pages?
3. Keep it brief — one round of questions max, then proceed

---

## Phase 6: Build

**Goal**: Implement the application

**Actions**:

1. Convert requirements into working code
2. Follow all project conventions from CLAUDE.md
3. Make technical decisions autonomously — don't ask the user to choose between frameworks, file structures, or implementation details
4. Update todos as you progress

### Building Web Features

For web CRUD features, follow the `create-crud-app-template` skill's execution order — it orchestrates the correct sequence of sub-skills (create-db-schema → api-router → query-collections → handle-views → customize-table) and includes testing as a required step.

### Other Available Skills

- **build-workflow** — React Flow visual UIs (web only)
- **ai-integration** — AI SDK + oRPC features
- **rls-setup** — Row-level security policies (only when requested)
- **manage-secrets** — Environment variables and secrets
- **schema0-cli** — Deploy, version, and secrets commands
- **integrations** — Discover and execute third-party integrations (Gmail, Slack, etc.)

> Skills marked **(web only)** require `apps/web/` to exist. Do not use them in mobile-only projects.

### Using Integrations

When the user wants to interact with external services through connected platforms:

```bash
bun schema0 integrations connections                           # List connected platforms
bun schema0 integrations search <platform> "what you want"     # Search actions (natural language)
bun schema0 integrations details <systemId>                    # Get parameters and docs
bun schema0 integrations execute <connKey> <path> [options]    # Execute an action
```

Always: list connections → search actions → get details → execute. The search query is natural language.

**Platform-specific guidance:**

**Web** (`apps/web/`):

- Uses React Router v7 + TanStack DB
- Web-only skills available: query-collections, handle-views, customize-table, create-crud-app-template
- Do NOT use web-only skills if `apps/web/` does not exist

**Mobile** (`apps/native/`):

- Uses React Native / Expo
- Single worker serves both Expo static assets and API at `/rpc` (same pattern as web)
- API routes mounted in `apps/native/worker.ts` via Hono + RPCHandler using `packages/api/` routers
- ORPC client attaches session cookies from `expo-secure-store` to every request
- Auth via `@schema0/auth-mobile` (cookie-based sealed sessions, same as web)
- Uses `@tanstack/react-query` directly (NOT TanStack DB)
- Data fetching: `useQuery(orpc.{entity}.selectAll.queryOptions())`
- For local dev: `bun schema0 dev` — starts Expo with a public tunnel URL (deploy first, then dev). The dev server must remain running — the app launched by scanning the QR code connects to it. If the server stops, the app will lose connection.

---

## Phase 7: Test

**Goal**: Verify each feature works before deploying

**Actions** (web platform):

1. Read `packages/test/CLAUDE.md` before writing any test code
2. Run `cd packages/test && bun drizzle-kit generate` to generate test DB migrations
3. For each entity, create `packages/test/src/{entity}.test.tsx` with create, update, and delete tests
4. Typecheck your files first, then run `cd packages/test && bun test src/{entity}.test.tsx`
5. Fix any failures — do not proceed to deployment until all tests pass

> If `apps/web/` does not exist (mobile-only project), skip steps 1–5 and verify the app builds without errors instead.

---

## Phase 8: Deploy

**Prerequisite**: All tests from Phase 7 pass.

**Actions**:

1. Commit your changes before deploying
2. Run diagnostics and deploy:
   ```bash
   bun schema0 doctor        # Check and fix configuration issues before deploying
   bun schema0 deploy        # Auto-detects and deploys all installed platforms
   ```
   To deploy a specific platform:
   ```bash
   bun schema0 deploy --platform web      # Deploy web only
   bun schema0 deploy --platform mobile   # Deploy mobile only
   ```
3. If a build fails, debug and fix it without involving the user
4. Share the deployed URL with the user
5. **Mobile projects:** After deploying, run `bun schema0 dev` to start the Expo dev server with a public tunnel URL. Display the QR code output to the user so they can scan it with Expo Go. The dev server must remain running — the app launched by scanning the QR code connects to it. If the server stops, the app will lose connection.

---

## Phase 9: Summary

**Goal**: Hand off to the user

**Actions**:

1. Mark all todos complete
2. Share the live URL
3. Summarize what was built, key decisions made, and suggested next steps

**Do NOT use interactive flags or prompts.** All CLI commands must be non-interactive.
