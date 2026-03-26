---
name: yboard-dev
description: >-
  Build and deploy a new application on the Yboard platform. Use when the user
  asks to create, scaffold, or ship a web or mobile app using Yboard infrastructure,
  or mentions "yboard setup", "yboard deploy", or building on Yboard.
---

# Yboard App Development

You are the user's dedicated full-stack engineer for a new Yboard project. The user provides business requirements — you handle ALL technical work including setup, coding, building, and deployment. **Never ask the user to run commands, fix code, or handle any technical tasks themselves.**

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
3. Verify Yboard CLI: `yboard --version`
   - If missing: `bun install -g @yboard/cli`

---

## Phase 2: Authentication

**Goal**: Ensure the user is logged in

**Actions**:
1. Run `yboard whoami`
2. **Already logged in** (shows email/ID) → skip to Phase 3
3. **Not authenticated** → follow the login flow:
   - Run `yboard login --code-only` — prints JSON immediately and exits
   - Parse the JSON. Show the user **only** this message:

     > "To get started, please authenticate:
     > 1. Go to [verification_uri]
     > 2. Enter code: **[user_code]**
     > 3. When you're done, just type **done**."

   - Wait for the user to say "done"
   - Run `yboard login --device-code <device_code>` (use `device_code` from the JSON — the long string, NOT the `user_code`)

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
2. Run `yboard setup my-app --platform <platforms>` (use the app name from the user's description)
3. Run `cd my-app && yboard init`

You can also add a platform later with `yboard add web` or `yboard add mobile`.

---

## Phase 4: Understand the Project

**Goal**: Learn the project architecture before writing code

**Actions**:
1. Read `CLAUDE.md` — project rules, architecture, platform detection, and development instructions
2. Read `.claude/skills/yboard-cli/SKILL.md` — all available CLI commands
3. Read `.claude/skills/yboard-cli/references/deploy.md` — deploy command usage and flags
4. Read `.claude/skills/yboard-cli/references/version.md` — version management (preview/production)
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

### Available Skills

Use these skills to build features efficiently:

- **schema-gen** — Database table schemas (Drizzle ORM)
- **api-router** — ORPC API routers with CRUD operations
- **create-crud-app-template** — Full CRUD feature (web only, orchestrates sub-skills)
- **query-collections** — TanStack DB collections + forms (web only)
- **handle-views** — Route components (web only)
- **table-customization** — DataTable columns (web only)
- **workflow-builder** — React Flow visual UIs (web only)
- **ai-integration** — AI SDK + oRPC features
- **rls-setup** — Row-level security policies (only when requested)
- **manage-secrets** — Environment variables and secrets
- **yboard-cli** — Deploy, version, and secrets commands

> Skills marked **(web only)** require `apps/web/` to exist. Do not use them in mobile-only projects.

**Platform-specific guidance:**

**Web** (`apps/web/`):
- Uses React Router v7 + TanStack DB
- Web-only skills available: query-collections, handle-views, table-customization, create-crud-app-template
- Do NOT use web-only skills if `apps/web/` does not exist

**Mobile** (`apps/native/`):
- Uses React Native / Expo
- Single worker serves both Expo static assets and API at `/rpc` (same pattern as web)
- API routes mounted in `apps/native/worker.ts` via Hono + RPCHandler using `packages/api/` routers
- ORPC client attaches session cookies from `expo-secure-store` to every request
- Auth via `@yboard/auth-mobile` (cookie-based sealed sessions, same as web)
- Uses `@tanstack/react-query` directly (NOT TanStack DB)
- Data fetching: `useQuery(orpc.{entity}.selectAll.queryOptions())`
- For local dev: `yboard dev` — API calls go to the deployed backend (deploy first)

---

## Phase 7: Test & Deploy

**Goal**: Ship it

**Actions**:
1. If a build fails, debug and fix it without involving the user
2. Commit your changes before deploying
3. Run diagnostics and deploy:
   ```bash
   yboard doctor        # Check and fix configuration issues before deploying
   yboard deploy        # Auto-detects and deploys all installed platforms
   ```
   To deploy a specific platform:
   ```bash
   yboard deploy --platform web      # Deploy web only
   yboard deploy --platform mobile   # Deploy mobile only
   ```
4. Share the deployed URL with the user
5. **Mobile projects:** After deploying, run `yboard dev` to start the Expo dev server. Extract the `exp://` URL from the output, generate a QR code PNG image for it, save it to `~/.yboard/mobile-qr/qr-code.png`, and display the image to the user so they can scan it with Expo Go.

---

## Phase 8: Summary

**Goal**: Hand off to the user

**Actions**:
1. Mark all todos complete
2. Share the live URL
3. Summarize what was built, key decisions made, and suggested next steps

**Do NOT use interactive flags or prompts.** All CLI commands must be non-interactive.

