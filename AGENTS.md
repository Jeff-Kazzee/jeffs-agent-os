# AGENTS.md — Agent Onboarding & Task Delegation

**Project:** Jeff's Agent OS Dashboard
**Owner:** Jeff Kazzee (@jeffkazzee, The Little AI Company)
**Stack:** Next.js 15 (App Router), TypeScript (strict), Tailwind CSS v4, SQLite (Drizzle ORM), Zustand, Bun, xterm.js, node-pty, WebSockets
**What:** A local-first web dashboard for orchestrating multiple AI coding agents from one interface. Single user, no auth, localhost:4000.
**Phase:** Building Phase 1 MVP — 6 features, ~40 files, ~3000 lines of code.

---

## 1. Required Reading

Read these files **in order** before writing any code:

| Priority | File | What You'll Learn |
|----------|------|-------------------|
| 1 | `AGENTS.md` | This file — task board, coordination, quality gates |
| 2 | `MISSION.md` | Phase 1 MVP scope, execution order, acceptance criteria, data models, design system values |
| 3 | `CLAUDE.md` | Code style, conventions, architecture philosophy (applies to ALL agents) |
| 4 | `project-status.md` | Current state — what's done, what's in progress, what's next |
| 5 | `docs/PRD.md` | Full product requirements, user stories, success metrics |
| 6 | `docs/DESIGN.md` | UI/UX design system, screen specs, component inventory, color palette |
| 7 | `docs/ARCHITECTURE.md` | Tech stack details, data models, API design, directory structure |

**Minimum reading before coding:** Files 1-4. Files 5-7 are reference — consult them when working on specific features.

---

## 2. Architecture Rules (Non-Negotiable)

These are hard constraints. Do not deviate.

- **Runtime:** Bun. Never use npm, pnpm, or yarn.
- **Framework:** Next.js 15 with App Router. Never use Pages Router.
- **Language:** TypeScript in strict mode. No `any` types unless absolutely unavoidable.
- **Styling:** Tailwind CSS v4. No CSS modules, no styled-components, no CSS-in-JS.
- **Database:** SQLite via `better-sqlite3` + Drizzle ORM. WAL mode enabled.
- **State:** Zustand for client-side state management.
- **Terminal:** xterm.js (browser) + node-pty (server). If node-pty fails to compile, fall back to `Bun.spawn`.
- **WebSocket:** `ws` library for real-time session output streaming.
- **Validation:** Zod for all input validation.
- **Port:** 4000 (not 3000 — avoids conflicts with other dev servers).
- **No authentication.** Single user, local only.
- **No cloud dependencies.** Everything runs on localhost.
- **No flashy animations.** Jeff has epilepsy. Smooth 150ms transitions only. No flashing, no rapid high-contrast movement.
- **Dark mode default.** Use the color palette defined in MISSION.md.
- **Four-layer architecture:** Core Types → Services → Actions → UI. Each layer only imports from the layer below it.

---

## 3. Code Conventions

| Convention | Rule | Example |
|-----------|------|---------|
| Files | kebab-case | `agent-service.ts` |
| Components | PascalCase | `SessionCard.tsx` |
| Hooks | camelCase with `use` prefix | `useAgents.ts` |
| Constants | SCREAMING_SNAKE_CASE | `KNOWN_AGENTS` |
| Types | PascalCase, exported from `src/lib/core/` | `Agent`, `Session` |
| Commits | Conventional format | `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:` |
| All code | Lives inside `dashboard/src/` | Never put app code in the workspace root |

---

## 4. Task Board — Phase 1 MVP

Each task is independently completable. Pick one that has all dependencies satisfied and no other agent has claimed.

### Dependency Graph

```
T1 (Scaffold)
 ├──► T2 (DB + Types) ──► T3 (Agent Service) ───┐
 │         │              ► T4 (Session Service) ─┤──► T6 (API Routes) ──► T7 (WebSocket)
 │         │              ► T5 (Project Service) ─┘        │
 │         └──► T9 (Stores + Hooks)                        │
 │                                                         │
 └──► T8 (Layout + Design System)                          │
       ├──► T10 (Dashboard Home) ◄──── T6, T9             │
       ├──► T11 (Agent Registry Page) ◄── T6, T9          │
       ├──► T12 (Session Monitor Page) ◄── T6, T7, T9     │
       │      └──► T13 (Live Terminal) ◄── T7              │
       ├──► T14 (Project Registry Page) ◄── T6, T9        │
       └──► T15 (Workspace Tree Page) ◄── T6              │
```

**Parallelism:** After T1, two independent tracks can run simultaneously:
- **Backend track:** T2 → T3/T4/T5 (parallel) → T6 → T7
- **Frontend track:** T8 → T9

After T6 + T8 + T9 are done, up to 6 page tasks (T10-T15) can run in parallel.

---

### Task 1: Project Scaffold

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | None |
| **Scope** | `dashboard/` root files only |
| **Branch** | `feat/task-01-scaffold` |

**What to do:**
1. `cd dashboard && bun init`
2. Install all dependencies from MISSION.md Step 1 (next, react, react-dom, typescript, tailwindcss, drizzle-orm, better-sqlite3, zod, zustand, nanoid, xterm, ws, node-pty, etc.)
3. Create `next.config.ts`, `tsconfig.json`, `tailwind.config.ts`, `.env`, `drizzle.config.ts`
4. Configure `package.json` scripts: `dev` (port 4000), `build`, `start` (port 4000), `db:generate`, `db:migrate`, `test`
5. Create `src/app/layout.tsx` with a minimal "Hello Agent OS" page to confirm it works

**Acceptance criteria:**
- `cd dashboard && bun install` succeeds
- `cd dashboard && bun dev` starts Next.js on port 4000
- Navigating to `http://localhost:4000` shows a page

**Verification:** `cd dashboard && bun dev` → opens without errors on port 4000

---

### Task 2: Database Schema + Core Types (Layer 1)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Task 1 |
| **Scope** | `dashboard/src/lib/db/`, `dashboard/src/lib/core/`, `dashboard/src/lib/validations/` |
| **Branch** | `feat/task-02-db-types` |

**What to do:**
1. Create `src/lib/db/schema.ts` — Drizzle schema with `agents`, `sessions`, `projects`, `sessionLogs` tables (exact schema in MISSION.md)
2. Create `src/lib/db/client.ts` — SQLite connection using `better-sqlite3` with WAL mode
3. Create `src/lib/core/agent.types.ts` — TypeScript interfaces for Agent
4. Create `src/lib/core/session.types.ts` — TypeScript interfaces for Session
5. Create `src/lib/core/project.types.ts` — TypeScript interfaces for Project
6. Create `src/lib/validations/agent-schema.ts` — Zod schemas for agent input validation
7. Create `src/lib/validations/session-schema.ts` — Zod schemas for session input validation
8. Create `drizzle.config.ts` if not already done in Task 1
9. Run `bun run db:generate` to generate migrations
10. Write a test: `tests/services/db-setup.test.ts` — creates DB, inserts an agent, queries it back

**Acceptance criteria:**
- All type files compile with `bunx tsc --noEmit`
- Database file is created at `.agent-os/agent-os.db`
- Test passes: insert + query roundtrip works
- Zod schemas validate correctly

**Verification:** `cd dashboard && bun test tests/services/db-setup.test.ts`

---

### Task 3: Agent Service (Layer 2)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Task 2 |
| **Scope** | `dashboard/src/lib/services/agent-service.ts`, `dashboard/src/constants/` |
| **Branch** | `feat/task-03-agent-service` |

**What to do:**
1. Create `src/constants/agents.ts` — `KNOWN_AGENTS` array (exact data in MISSION.md)
2. Create `src/constants/paths.ts` — workspace path constants (exact paths in MISSION.md)
3. Create `src/lib/services/agent-service.ts` — `AgentRegistry` class with:
   - `autoDetect()` — scan PATH for known binaries using `where` (Windows) or `which` (Unix)
   - `register(agent)` — add agent to SQLite + write JSON to `.agent-os/registry/`
   - `getAll()` — list all registered agents
   - `getById(id)` — get single agent
   - `update(id, data)` — update agent
   - `delete(id)` — remove agent
   - `checkAvailability(binaryPath)` — verify binary exists on PATH
4. Write test: `tests/services/agent-service.test.ts`

**Acceptance criteria:**
- `autoDetect()` finds at least one binary installed on the machine
- CRUD operations work against SQLite
- Test passes

**Verification:** `cd dashboard && bun test tests/services/agent-service.test.ts`

---

### Task 4: Session Service (Layer 2)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Task 2 |
| **Scope** | `dashboard/src/lib/services/session-service.ts` |
| **Branch** | `feat/task-04-session-service` |

**What to do:**
1. Create `src/lib/services/session-service.ts` — `SessionManager` class with:
   - `create(agentId, workingDir, taskDescription?)` — spawn agent process via node-pty (or Bun.spawn fallback), create session record in SQLite, return session ID
   - `getAll()` — list all sessions
   - `getById(id)` — get session with metadata
   - `getActive()` — list running sessions
   - `stop(id)` — kill process, update status
   - `onOutput(sessionId, callback)` — subscribe to live output from PTY
   - `getOutput(sessionId)` — get buffered output from session_logs
2. Handle PTY lifecycle: start → stream output → stop → cleanup
3. Store output chunks in `session_logs` table
4. Write test: `tests/services/session-service.test.ts`

**Acceptance criteria:**
- Can spawn a simple process (e.g., `echo hello`) and capture output
- Session status transitions work (starting → running → stopped)
- Output is stored in session_logs
- Test passes

**Verification:** `cd dashboard && bun test tests/services/session-service.test.ts`

---

### Task 5: Project Service (Layer 2)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Task 2 |
| **Scope** | `dashboard/src/lib/services/project-service.ts` |
| **Branch** | `feat/task-05-project-service` |

**What to do:**
1. Create `src/lib/services/project-service.ts` — `ProjectScanner` class with:
   - `autoScan()` — scan `projects/personal/`, `projects/business/`, `projects/tools/` for directories containing `package.json` or `CLAUDE.md`
   - `register(project)` — add project to SQLite
   - `getAll()` — list all projects
   - `getById(id)` — get single project
   - `detectStack(projectPath)` — read `package.json` to determine stack (next.js, typescript, etc.)
   - `categorize(projectPath)` — determine category from parent dir (personal/business/tool)
2. Write test: `tests/services/project-service.test.ts`

**Acceptance criteria:**
- `autoScan()` correctly identifies project directories
- Stack detection reads `package.json` dependencies
- Category is inferred from parent directory
- Test passes

**Verification:** `cd dashboard && bun test tests/services/project-service.test.ts`

---

### Task 6: API Routes (Layer 3)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 3, 4, 5 |
| **Scope** | `dashboard/src/app/api/` |
| **Branch** | `feat/task-06-api-routes` |

**What to do:**
1. `GET /api/agents` — list all agents
2. `POST /api/agents` — register new agent (validate with Zod)
3. `POST /api/agents/auto-detect` — trigger auto-detection
4. `POST /api/sessions` — create + launch session (validate with Zod)
5. `GET /api/sessions` — list all sessions
6. `GET /api/sessions/[id]` — get session details
7. `GET /api/projects` — list all projects
8. `POST /api/projects/auto-scan` — trigger project scanning
9. `GET /api/workspace` — list directory contents (accepts `?path=` query param)
10. `GET /api/health` — return `{ status: 'ok', version: '0.1.0', uptime: process.uptime() }`

Each route: validate input with Zod, call the appropriate service, return JSON with proper status codes.

**Acceptance criteria:**
- `curl http://localhost:4000/api/health` returns `{"status":"ok","version":"0.1.0",...}`
- All routes return valid JSON
- Invalid input returns 400 with error details
- Missing resources return 404

**Verification:** Start dev server, `curl` each endpoint

---

### Task 7: WebSocket Session Streaming

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 4, 6 |
| **Scope** | `dashboard/src/app/api/sessions/ws/` |
| **Branch** | `feat/task-07-websocket` |

**What to do:**
1. Create WebSocket endpoint at `/api/sessions/ws`
2. Clients connect and send `{ "subscribe": "<session-id>" }` to receive live output
3. Server streams PTY output chunks as `{ "sessionId": "...", "data": "...", "streamType": "stdout" }`
4. Handle client disconnect gracefully
5. Support multiple clients watching the same session

**Acceptance criteria:**
- WebSocket connection establishes successfully
- Subscribing to a session ID streams live output
- Multiple clients can watch the same session
- Disconnects are handled without crashes

**Verification:** Use `wscat` or browser console to connect and subscribe

---

### Task 8: Root Layout + Design System (Layer 4)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Task 1 |
| **Scope** | `dashboard/src/app/layout.tsx`, `dashboard/src/app/globals.css`, `dashboard/src/components/ui/`, `dashboard/src/components/layout/` |
| **Branch** | `feat/task-08-layout-design` |

**What to do:**
1. Create `src/app/globals.css` — CSS custom properties for the design system (colors, fonts, radii from MISSION.md)
2. Update `src/app/layout.tsx` — dark mode (`<html class="dark">`), Inter + JetBrains Mono font imports, metadata
3. Create UI primitives in `src/components/ui/`:
   - `Button.tsx` — primary, secondary, ghost, danger variants
   - `Card.tsx` — container with border, background, optional header
   - `Input.tsx` — text input with label, error state
   - `Badge.tsx` — status badges (running, stopped, error, unknown)
   - `Modal.tsx` — overlay dialog with backdrop
   - `StatusDot.tsx` — colored dot indicator (green/amber/red/gray)
4. Create layout components in `src/components/layout/`:
   - `Sidebar.tsx` — collapsible nav sidebar with links: Home, Agents, Sessions, Projects, Workspace
   - `Header.tsx` — top bar with page title
5. Create `src/app/(main)/layout.tsx` — wraps pages with Sidebar + Header

**Acceptance criteria:**
- Dark background renders (#0a0e27)
- Sidebar shows 5 navigation links
- UI components render with correct colors and styles
- No animations faster than 150ms

**Verification:** `bun dev` → navigate to localhost:4000, visually confirm dark theme + sidebar

---

### Task 9: Zustand Stores + Custom Hooks

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Task 2 |
| **Scope** | `dashboard/src/lib/stores/`, `dashboard/src/lib/hooks/` |
| **Branch** | `feat/task-09-stores-hooks` |

**What to do:**
1. Create `src/lib/stores/ui-store.ts` — sidebar collapsed state, active modal, toast notifications
2. Create `src/lib/stores/session-store.ts` — active sessions list, live output buffers per session
3. Create `src/lib/hooks/useAgents.ts` — fetch/mutate agents via API
4. Create `src/lib/hooks/useSessions.ts` — fetch/mutate sessions via API
5. Create `src/lib/hooks/useWebSocket.ts` — WebSocket connection management, auto-reconnect

**Acceptance criteria:**
- Stores initialize with correct default values
- Hooks fetch data from API endpoints
- WebSocket hook connects and receives messages
- TypeScript compiles

**Verification:** `cd dashboard && bunx tsc --noEmit`

---

### Task 10: Dashboard Home Page

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 6, 8, 9 |
| **Scope** | `dashboard/src/app/page.tsx` or `dashboard/src/app/(main)/page.tsx` |
| **Branch** | `feat/task-10-dashboard-home` |

**What to do:**
1. Create the `/` route page
2. Show 4 sections:
   - **Active Sessions** — count + compact session cards (agent name, status dot, working dir, elapsed time)
   - **Registered Agents** — grid of agents with status indicators (green = available, gray = not found)
   - **Recent Activity** — last 10 session starts/stops with timestamps
   - **Quick Actions** — buttons: "New Session", "Scan Agents", "Scan Projects"
3. Fetch data from API endpoints via hooks

**Acceptance criteria:**
- Page renders all 4 sections
- Quick action buttons trigger API calls
- Status indicators use correct colors

**Verification:** `bun dev` → navigate to localhost:4000

---

### Task 11: Agent Registry Page

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 6, 8, 9 |
| **Scope** | `dashboard/src/app/(main)/agents/page.tsx`, `dashboard/src/components/features/agent-registry/` |
| **Branch** | `feat/task-11-agent-registry` |

**What to do:**
1. Create `/agents` page
2. Create components: `AgentList.tsx`, `AgentCard.tsx`, `RegisterAgentForm.tsx`
3. AgentCard shows: name, model, binary path, status dot, capabilities badges, [Launch] button
4. RegisterAgentForm: name, binary path, model, description, capabilities (multi-select)
5. Auto-detect button at top that calls `POST /api/agents/auto-detect`

**Acceptance criteria:**
- Lists all registered agents
- Auto-detect button finds installed binaries
- Can register a new agent via form
- Agent cards show correct status colors

**Verification:** `bun dev` → navigate to `/agents`, click "Auto Detect"

---

### Task 12: Session Monitor Page

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 6, 7, 8, 9 |
| **Scope** | `dashboard/src/app/(main)/sessions/`, `dashboard/src/components/features/session-monitor/` (except TerminalView) |
| **Branch** | `feat/task-12-session-monitor` |

**What to do:**
1. Create `/sessions` page — list all sessions
2. Create `/sessions/[id]` page — session detail view (placeholder for terminal)
3. Create components: `SessionList.tsx`, `SessionCard.tsx`
4. SessionCard shows: agent name, working dir, start time, status badge, elapsed time
5. Session detail shows: metadata header, placeholder for terminal output, stop button

**Acceptance criteria:**
- Session list renders with correct status badges
- Clicking a session navigates to detail page
- Detail page shows session metadata

**Verification:** `bun dev` → navigate to `/sessions`

---

### Task 13: Live Terminal (xterm.js)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 7, 12 |
| **Scope** | `dashboard/src/components/features/session-monitor/TerminalView.tsx` |
| **Branch** | `feat/task-13-live-terminal` |

**What to do:**
1. Create `TerminalView.tsx` — xterm.js terminal component
2. Use `@xterm/addon-fit` for responsive sizing
3. Use `@xterm/addon-web-links` for clickable URLs
4. Connect to WebSocket via `useWebSocket` hook
5. Subscribe to session output by session ID
6. Render ANSI-colored output in the terminal
7. Handle terminal resize events
8. Configure scrollback buffer (10,000 lines)
9. Integrate into `/sessions/[id]` page

**Acceptance criteria:**
- Terminal renders with dark background matching design system
- Live output streams in real-time from WebSocket
- ANSI colors render correctly
- Terminal resizes when window resizes
- Scrollback works

**Verification:** Launch an agent session, navigate to its detail page, see live terminal output

---

### Task 14: Project Registry Page

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 6, 8, 9 |
| **Scope** | `dashboard/src/app/(main)/projects/page.tsx`, `dashboard/src/components/features/project-registry/` |
| **Branch** | `feat/task-14-project-registry` |

**What to do:**
1. Create `/projects` page
2. Create components: `ProjectList.tsx`, `ProjectCard.tsx`
3. ProjectCard shows: name, path, stack, status, category badge
4. Auto-scan button that calls `POST /api/projects/auto-scan`
5. Quick-launch: click project → select agent → launches session in that directory

**Acceptance criteria:**
- Auto-scan finds project directories
- Projects display with correct metadata
- Quick-launch opens agent selection and creates session

**Verification:** `bun dev` → navigate to `/projects`, click "Scan Projects"

---

### Task 15: Workspace Tree Page

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Depends on** | Tasks 6, 8 |
| **Scope** | `dashboard/src/app/(main)/workspace/page.tsx`, `dashboard/src/components/features/workspace-tree/` |
| **Branch** | `feat/task-15-workspace-tree` |

**What to do:**
1. Create `/workspace` page
2. Create components: `WorkspaceTree.tsx`, `TreeNode.tsx`
3. Show workspace root directories: `business/`, `projects/`, `knowledge/`, `planning/`
4. Expandable nodes — click to load children via `GET /api/workspace?path=<relative>`
5. Icons for folders vs files
6. Click a project directory to navigate to its detail

**Acceptance criteria:**
- Tree renders top-level workspace directories
- Directories expand on click to show children
- File/folder icons display correctly

**Verification:** `bun dev` → navigate to `/workspace`, expand directories

---

## 5. Coordination Protocol

Rules for multi-agent parallel execution. Follow these strictly.

### 5.1 Claiming a Task

Before starting work on a task:

1. Read `project-status.md` — check the "Currently In Progress" table
2. If the task is already claimed by another agent, pick a different task
3. If unclaimed, update `project-status.md` to add your claim:
   ```
   | Task 3: Agent Service | Claude Code | 2026-03-14 15:30 | feat/task-03-agent-service |
   ```
4. Create your branch: `git checkout -b feat/task-{id}-{short-name}`

### 5.2 Scope Isolation

Each task specifies its file scope. You MUST only modify files within your task's scope. If you need to change a file outside your scope, that means:
- You're working on the wrong task, OR
- A dependency task hasn't been completed yet, OR
- The task breakdown needs adjustment (flag this in `project-status.md`)

### 5.3 Branching

- One branch per task: `feat/task-{id}-{short-name}`
- Commit frequently with conventional messages
- When done, merge to `main` (or create a PR if preferred)

### 5.4 Completing a Task

When your task is done:

1. Run all quality gates (Section 6)
2. Update `project-status.md`:
   - Remove your row from "Currently In Progress"
   - Add a row to "Task Completion Log"
3. Update THIS file: change the task's Status from `NOT STARTED` to `DONE`
4. Commit these status updates
5. Merge your branch to `main`

### 5.5 Hard Rules

- **Do NOT build Tier 2 features.** No workflow builder, no task router, no knowledge base UI, no auto-context injection.
- **Do NOT modify files outside your task scope.**
- **Do NOT skip quality gates.**
- **Do NOT use npm/pnpm/yarn.** Bun only.

---

## 6. Quality Gates

Run these checks before marking any task as DONE:

```bash
# 1. TypeScript compiles
cd dashboard && bunx tsc --noEmit

# 2. Tests pass (if task includes tests)
cd dashboard && bun test

# 3. Build succeeds
cd dashboard && bun run build

# 4. Dev server starts
cd dashboard && bun dev
# Verify it starts on port 4000 without errors
```

All four must pass. If any fails, fix it before marking the task complete.

---

## 7. Platform Notes

- **OS:** Windows 11 (10.0.26200)
- **Shell:** bash (Git Bash / WSL-compatible)
- **Binary detection:** Use `where <binary>` on Windows, not `which`. Write platform-aware code:
  ```typescript
  const cmd = process.platform === 'win32' ? 'where' : 'which';
  ```
- **Bun version:** 1.3.10
- **node-pty:** May require native compilation. If it fails, fall back to `Bun.spawn` with `{ stdin: 'pipe', stdout: 'pipe', stderr: 'pipe' }`. The session service should abstract this so the rest of the app doesn't care which backend is used.
- **Path separators:** Use forward slashes in code. Node.js `path` module handles cross-platform concerns.
- **Database location:** `.agent-os/agent-os.db` (relative to workspace root, NOT dashboard/)

---

## 8. File Map

Every file the Phase 1 MVP will create, which task creates it, and its purpose.

| File | Task | Purpose |
|------|------|---------|
| `dashboard/package.json` | T1 | Dependencies and scripts |
| `dashboard/next.config.ts` | T1 | Next.js configuration |
| `dashboard/tsconfig.json` | T1 | TypeScript configuration |
| `dashboard/tailwind.config.ts` | T1 | Tailwind CSS v4 configuration |
| `dashboard/.env` | T1 | Environment variables |
| `dashboard/drizzle.config.ts` | T1/T2 | Drizzle ORM configuration |
| `dashboard/src/app/layout.tsx` | T1/T8 | Root layout (dark mode, fonts) |
| `dashboard/src/app/globals.css` | T8 | Design system CSS custom properties |
| `dashboard/src/app/page.tsx` | T10 | Dashboard home page |
| `dashboard/src/app/(main)/layout.tsx` | T8 | Sidebar + header wrapper |
| `dashboard/src/app/(main)/agents/page.tsx` | T11 | Agent registry page |
| `dashboard/src/app/(main)/sessions/page.tsx` | T12 | Session list page |
| `dashboard/src/app/(main)/sessions/[id]/page.tsx` | T12/T13 | Session detail + terminal |
| `dashboard/src/app/(main)/projects/page.tsx` | T14 | Project registry page |
| `dashboard/src/app/(main)/workspace/page.tsx` | T15 | Workspace tree page |
| `dashboard/src/app/api/agents/route.ts` | T6 | GET/POST agents |
| `dashboard/src/app/api/agents/auto-detect/route.ts` | T6 | POST auto-detect |
| `dashboard/src/app/api/sessions/route.ts` | T6 | GET/POST sessions |
| `dashboard/src/app/api/sessions/[id]/route.ts` | T6 | GET session by ID |
| `dashboard/src/app/api/sessions/ws/route.ts` | T7 | WebSocket endpoint |
| `dashboard/src/app/api/projects/route.ts` | T6 | GET projects |
| `dashboard/src/app/api/projects/auto-scan/route.ts` | T6 | POST auto-scan |
| `dashboard/src/app/api/workspace/route.ts` | T6 | GET workspace tree |
| `dashboard/src/app/api/health/route.ts` | T6 | GET health check |
| `dashboard/src/lib/db/schema.ts` | T2 | Drizzle ORM schema |
| `dashboard/src/lib/db/client.ts` | T2 | SQLite connection |
| `dashboard/src/lib/core/agent.types.ts` | T2 | Agent TypeScript types |
| `dashboard/src/lib/core/session.types.ts` | T2 | Session TypeScript types |
| `dashboard/src/lib/core/project.types.ts` | T2 | Project TypeScript types |
| `dashboard/src/lib/validations/agent-schema.ts` | T2 | Zod agent schemas |
| `dashboard/src/lib/validations/session-schema.ts` | T2 | Zod session schemas |
| `dashboard/src/lib/services/agent-service.ts` | T3 | Agent registry business logic |
| `dashboard/src/lib/services/session-service.ts` | T4 | Session manager business logic |
| `dashboard/src/lib/services/project-service.ts` | T5 | Project scanner business logic |
| `dashboard/src/constants/agents.ts` | T3 | Known agent binaries |
| `dashboard/src/constants/paths.ts` | T3 | Workspace path constants |
| `dashboard/src/lib/stores/ui-store.ts` | T9 | UI state (sidebar, modals) |
| `dashboard/src/lib/stores/session-store.ts` | T9 | Session state (active, output) |
| `dashboard/src/lib/hooks/useAgents.ts` | T9 | Agent data fetching hook |
| `dashboard/src/lib/hooks/useSessions.ts` | T9 | Session data fetching hook |
| `dashboard/src/lib/hooks/useWebSocket.ts` | T9 | WebSocket connection hook |
| `dashboard/src/components/ui/Button.tsx` | T8 | Button primitive |
| `dashboard/src/components/ui/Card.tsx` | T8 | Card primitive |
| `dashboard/src/components/ui/Input.tsx` | T8 | Input primitive |
| `dashboard/src/components/ui/Badge.tsx` | T8 | Badge primitive |
| `dashboard/src/components/ui/Modal.tsx` | T8 | Modal primitive |
| `dashboard/src/components/ui/StatusDot.tsx` | T8 | Status indicator dot |
| `dashboard/src/components/layout/Sidebar.tsx` | T8 | Navigation sidebar |
| `dashboard/src/components/layout/Header.tsx` | T8 | Page header |
| `dashboard/src/components/features/agent-registry/AgentList.tsx` | T11 | Agent list view |
| `dashboard/src/components/features/agent-registry/AgentCard.tsx` | T11 | Agent card component |
| `dashboard/src/components/features/agent-registry/RegisterAgentForm.tsx` | T11 | Agent registration form |
| `dashboard/src/components/features/session-monitor/SessionList.tsx` | T12 | Session list view |
| `dashboard/src/components/features/session-monitor/SessionCard.tsx` | T12 | Session card component |
| `dashboard/src/components/features/session-monitor/TerminalView.tsx` | T13 | xterm.js terminal |
| `dashboard/src/components/features/project-registry/ProjectList.tsx` | T14 | Project list view |
| `dashboard/src/components/features/project-registry/ProjectCard.tsx` | T14 | Project card component |
| `dashboard/src/components/features/workspace-tree/WorkspaceTree.tsx` | T15 | Workspace tree view |
| `dashboard/src/components/features/workspace-tree/TreeNode.tsx` | T15 | Tree node component |
| `dashboard/tests/services/db-setup.test.ts` | T2 | Database setup test |
| `dashboard/tests/services/agent-service.test.ts` | T3 | Agent service test |
| `dashboard/tests/services/session-service.test.ts` | T4 | Session service test |
| `dashboard/tests/services/project-service.test.ts` | T5 | Project service test |

---

## 9. Final Checklist (Phase 1 MVP Complete When...)

- [ ] `bun dev` starts dashboard on port 4000 without errors
- [ ] Dashboard home shows registered agents and any active sessions
- [ ] Auto-detect finds at least `claude` binary (or whatever's installed)
- [ ] Can manually register a new agent from the UI
- [ ] Can launch a session (pick agent + working dir) and see it in active sessions
- [ ] Session detail page shows live terminal output via xterm.js
- [ ] Project auto-scan finds directories in `projects/` that contain `package.json`
- [ ] Workspace tree shows the folder hierarchy and is expandable
- [ ] All API endpoints return valid JSON
- [ ] SQLite database persists between restarts
- [ ] UI is dark mode, uses design system colors, feels like a power tool
- [ ] No TypeScript errors (`bun run build` succeeds)
- [ ] At least 5 service-level tests pass
