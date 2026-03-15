# AGENTS.md

## Project Overview

Jeff's Agent OS is a local-first web dashboard for orchestrating multiple AI coding agents (Claude Code, Gemini CLI, Codex CLI, and others) from a single interface. It provides agent registration, session management with live terminal output, project scanning, and workspace navigation.

**Architecture:** Next.js 15 App Router with a four-layer architecture — Core Types → Services → Actions → UI. Each layer only imports from the layer below it. The app is a monolith inside the `dashboard/` directory, with the workspace root containing docs, knowledge base, and project directories.

**Key technologies:** Next.js 15 (App Router), TypeScript (strict), Tailwind CSS v4, SQLite (better-sqlite3 + Drizzle ORM), Zustand, Bun, xterm.js, node-pty, WebSockets (ws library), Zod.

**Workspace structure:**
```
Jeffs Agent OS/              ← workspace root
├── .agent-os/               ← runtime state (registry, sessions, logs, SQLite DB)
├── dashboard/               ← the Next.js app (ALL code lives here)
│   └── src/
│       ├── app/             ← pages + API routes (App Router)
│       ├── components/      ← UI primitives, layout, feature components
│       └── lib/             ← core types, services, db, stores, hooks, validations
├── docs/                    ← PRD, DESIGN, ARCHITECTURE docs
├── business/                ← business ops (not code)
├── projects/                ← coding projects (scanned by project registry)
├── knowledge/               ← Obsidian-compatible knowledge base
└── planning/                ← project planning hub
```

## Setup Commands

```bash
# Install dependencies (Bun ONLY — never use npm, pnpm, or yarn)
cd dashboard && bun install

# Initialize the database (creates .agent-os/agent-os.db with WAL mode)
cd dashboard && bun run db:init

# Start development server on port 4000
cd dashboard && bun dev

# Build for production
cd dashboard && bun run build

# Start production server on port 4000
cd dashboard && bun run start
```

**node-pty note:** This package requires native compilation. If `bun add node-pty` fails on your platform, the session service falls back to `Bun.spawn` with piped stdio. The rest of the app doesn't need to know which backend is used.

## Development Workflow

- Development server runs on **port 4000** (not 3000 — avoids conflicts with other dev servers).
- All application code lives inside `dashboard/src/`. Never put app code in the workspace root.
- The SQLite database lives at `.agent-os/agent-os.db` relative to the workspace root (one level up from `dashboard/`). It is created automatically on first run.
- Environment variables are in `dashboard/.env`:
  ```
  DATABASE_URL=../.agent-os/agent-os.db
  PORT=4000
  ```
- The custom `server.ts` wraps Next.js to add WebSocket support (Next.js App Router doesn't natively support WebSocket upgrade). Both `dev` and `start` scripts use this server.
- Hot reload works normally via Next.js. Changes to `src/` files trigger automatic refresh.

## Testing Instructions

```bash
# Run all tests
cd dashboard && bun test

# Run a specific test file
cd dashboard && bun test tests/services/agent-service.test.ts

# Run tests in watch mode
cd dashboard && bun test --watch

# Type checking (zero errors required)
cd dashboard && bunx tsc --noEmit
```

- **Test framework:** Bun's built-in `bun:test` — no additional test runner needed.
- **Test location:** `dashboard/tests/services/` for service-level tests.
- **Test database:** Tests should create an in-memory SQLite database or use a temporary file, not the production database.
- **Minimum coverage:** At least 5 service-level tests must pass before any build is considered complete.
- **What to test:** Services layer (agent-service, session-service, project-service) and database operations. UI components are verified visually.

### Test files:
```
dashboard/tests/services/
├── db-setup.test.ts          # DB creation, insert, query roundtrip
├── agent-service.test.ts     # Auto-detect, CRUD, checkAvailability
├── session-service.test.ts   # Spawn process, capture output, status transitions
└── project-service.test.ts   # Auto-scan, detectStack, categorize
```

## Code Style

### Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Files | kebab-case | `agent-service.ts` |
| React components | PascalCase | `SessionCard.tsx` |
| Hooks | camelCase with `use` prefix | `useAgents.ts` |
| Constants | SCREAMING_SNAKE_CASE | `KNOWN_AGENTS` |
| Types/Interfaces | PascalCase | `Agent`, `SessionStatus` |
| CSS variables | kebab-case with `--` prefix | `--bg-primary` |

### TypeScript Rules

- Strict mode enabled. No `any` types unless absolutely unavoidable.
- Types are defined in `src/lib/core/` and exported from there.
- Zod schemas for all external input validation live in `src/lib/validations/`.
- Use `nanoid` for generating unique IDs.

### Commit Messages

Conventional commit format required:
```
feat: add agent auto-detection
fix: handle missing binary path in agent registry
docs: update API endpoint documentation
refactor: extract PTY abstraction from session service
test: add session service integration tests
chore: update dependencies
```

### Linting and Formatting

```bash
# Run ESLint
cd dashboard && bun run lint

# Run Prettier check
cd dashboard && bun run format:check

# Auto-format with Prettier
cd dashboard && bun run format
```

- ESLint: extends Next.js recommended + TypeScript strict + React hooks rules.
- Prettier: single quotes, semicolons, trailing commas, 2-space indent, 100 char print width.

## Build and Deployment

This is a **self-hosted, single-user, local-only** application. There is no cloud deployment. It runs on Jeff's machine.

```bash
# Full production build and start
cd dashboard && bun run db:init && bun run build && bun run start

# Or use the startup script
./dashboard/scripts/start.sh
```

- **Build output:** `dashboard/.next/` (standard Next.js output)
- **Production server:** Runs on port 4000 via custom `server.ts` (includes WebSocket support)
- **Database:** SQLite with WAL mode at `.agent-os/agent-os.db` — persists between restarts
- **No Docker required.** Bun + Next.js run natively.
- **No environment secrets.** Single user, no auth, no API keys needed for core features.

### CI/CD (GitHub Actions)

```bash
# The CI pipeline runs on every push and PR:
# 1. bun install
# 2. bun run lint
# 3. bun run format:check
# 4. bunx tsc --noEmit
# 5. bun test
# 6. bun run build
```

CI config lives at `.github/workflows/ci.yml`. Uses `oven-sh/setup-bun@v2` for Bun installation.

## Architecture Rules

These are non-negotiable constraints. Do not deviate.

- **Runtime:** Bun. Never use npm, pnpm, or yarn for any operation.
- **Framework:** Next.js 15 with App Router. Never use Pages Router.
- **Styling:** Tailwind CSS v4. No CSS modules, no styled-components, no CSS-in-JS.
- **Database:** SQLite via `better-sqlite3` + Drizzle ORM. WAL mode. DB path: `../.agent-os/agent-os.db`.
- **State:** Zustand for client-side state. No Redux, no Context API for global state.
- **Terminal:** xterm.js (browser) + node-pty (server) with Bun.spawn fallback.
- **Validation:** Zod for all input validation at API boundaries.
- **Port:** 4000. Always.
- **No authentication.** Single user, local only, localhost.
- **No cloud dependencies.** Everything runs locally.
- **No flashy animations.** The owner has epilepsy. Smooth 150ms CSS transitions only. No flashing, no rapid high-contrast movement, no auto-playing animations.
- **Dark mode only.** Deep navy-black aesthetic (#0a0e27 base). Design system colors defined in `src/app/globals.css`.
- **Four-layer architecture:**
  1. **Core Types** (`src/lib/core/`) — TypeScript interfaces
  2. **Services** (`src/lib/services/`) — Business logic classes
  3. **Actions** (`src/app/api/`) — REST + WebSocket API routes
  4. **UI** (`src/components/`, `src/app/`) — React components and pages

  Each layer only imports from the layer below it. UI never imports services directly — it goes through API routes or hooks.

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | Health check — returns `{ status, version, uptime }` |
| GET | `/api/agents` | List all registered agents |
| POST | `/api/agents` | Register a new agent (Zod-validated) |
| POST | `/api/agents/auto-detect` | Scan PATH for known agent binaries |
| GET | `/api/sessions` | List all sessions (optional `?status=` filter) |
| POST | `/api/sessions` | Create + launch a new agent session |
| GET | `/api/sessions/[id]` | Get session details by ID |
| WS | `/api/sessions/ws` | WebSocket — subscribe to live session output |
| POST | `/api/terminal` | Spawn a general-purpose shell session |
| GET | `/api/projects` | List all registered projects |
| POST | `/api/projects/auto-scan` | Scan workspace for projects |
| GET | `/api/workspace` | List directory contents (`?path=` relative path) |

All REST endpoints return JSON. Errors return `{ error: string, details?: object }` with appropriate HTTP status codes (400, 404, 500).

## Platform Notes

- **OS:** Windows 11. Use `where` instead of `which` for binary detection. Write platform-aware code:
  ```typescript
  const cmd = process.platform === 'win32' ? 'where' : 'which';
  ```
- **Bun version:** 1.3.10
- **Path separators:** Use forward slashes in code. The `path` module handles cross-platform concerns.
- **node-pty native compilation:** May fail on some platforms. The session service abstracts this behind a PTY interface that falls back to `Bun.spawn`.

## Pull Request Guidelines

- **Branch naming:** `feat/task-{id}-{short-name}` (e.g., `feat/task-03-agent-service`)
- **Title format:** `feat: add agent auto-detection service`
- **Required checks before submission:**
  1. `bunx tsc --noEmit` — zero TypeScript errors
  2. `bun test` — all tests pass
  3. `bun run build` — Next.js build succeeds
  4. `bun dev` — dev server starts on port 4000 without errors
- **Commit messages:** conventional format (see Code Style section)
- **Scope isolation:** Each PR should only touch files within its feature scope. See `planning/active/phase-1-taskboard.md` for task scopes.

## Additional Context

- **Planning docs:** `docs/PRD.md` (requirements), `docs/DESIGN.md` (UI/UX), `docs/ARCHITECTURE.md` (technical architecture). Consult these for detailed specs.
- **Task board:** `planning/active/phase-1-taskboard.md` contains the Phase 1 MVP task breakdown with dependency graph, file scopes, acceptance criteria, and coordination protocol for multi-agent parallel execution.
- **Sprint plan:** See the plan file for epic/sprint/story breakdown.
- **Project status:** `project-status.md` tracks current state and agent coordination.
- **Prior art:** `Stochastic multi-agent Chat Room/` contains multi-agent orchestration patterns (chatrooms, prompt contracts, consensus, verification loops) from earlier research.

## Debugging and Troubleshooting

- **`bun dev` won't start:** Check that port 4000 is free. Kill any process using it: `lsof -i :4000` (Unix) or `netstat -ano | findstr :4000` (Windows).
- **Database errors:** Delete `.agent-os/agent-os.db` and re-run `bun run db:init` to recreate from scratch.
- **node-pty compilation fails:** The session service automatically falls back to `Bun.spawn`. Check `dashboard/src/lib/services/session-service.ts` for the abstraction.
- **WebSocket not connecting:** The WebSocket runs on the same port (4000) via custom `server.ts`. Ensure you're connecting to `ws://localhost:4000/api/sessions/ws`.
- **TypeScript errors after pulling:** Run `bunx tsc --noEmit` to see all errors. Fix them before proceeding.
- **Tests failing:** Run `bun test --verbose` for detailed output. Check that no test is hitting the production database.
