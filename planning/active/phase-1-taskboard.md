# Phase 1 MVP — Task Board

**Phase:** Building
**Features:** 6 (Agent Registry, Agent Launcher, Session Monitor, Dashboard Home, Project Registry, Workspace Tree)
**Estimated Scope:** ~50 files, ~3500 LOC

---

## Dependency Graph

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

**Parallelism:** After T1, two independent tracks:
- **Backend track:** T2 → T3/T4/T5 (parallel) → T6 → T7
- **Frontend track:** T8 → T9

After T6 + T8 + T9 complete, page tasks T10-T15 can run in parallel.

---

## Sprint Assignment

| Sprint | Tasks | Goal |
|--------|-------|------|
| S1: "Hello Agent OS" | T1, T2, T8 | `bun dev` on :4000, dark UI shell, DB roundtrip |
| S2: "The Engine" | T3, T4, T5, T6 | All services + API endpoints returning JSON |
| S3: "The Face" | T9, T10, T11, T12, T14, T15 | All 6 pages render with real API data |
| S4: "The Terminal" | T7, T13, T16 | Live xterm.js + WebSocket + general CLI shell |
| S5: "Ship It" | DevOps tasks | CI green, lint clean, prod build works |

---

## Task Details

### Task 1: Project Scaffold

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Sprint** | S1 |
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
| **Sprint** | S1 |
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
| **Sprint** | S2 |
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
| **Sprint** | S2 |
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
| **Sprint** | S2 |
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
| **Sprint** | S2 |
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
| **Sprint** | S4 |
| **Depends on** | Tasks 4, 6 |
| **Scope** | `dashboard/src/app/api/sessions/ws/`, `dashboard/server.ts` |
| **Branch** | `feat/task-07-websocket` |

**What to do:**
1. Create custom `server.ts` that wraps Next.js HTTP handler and adds a `ws` WebSocket server (Next.js App Router doesn't support WebSocket upgrade natively)
2. Intercept upgrade requests to `/api/sessions/ws`
3. Clients connect and send `{ "subscribe": "<session-id>" }` to receive live output
4. Server streams PTY output chunks as `{ "sessionId": "...", "data": "...", "streamType": "stdout" }`
5. Support bidirectional: client sends `{ "type": "input", "sessionId": "...", "data": "<keystroke>" }` for stdin
6. Handle client disconnect gracefully
7. Support multiple clients watching the same session
8. Update `package.json` scripts to use `server.ts`

**Acceptance criteria:**
- WebSocket connection establishes successfully
- Subscribing to a session ID streams live output
- Keyboard input is forwarded to PTY stdin
- Multiple clients can watch the same session
- Disconnects are handled without crashes

**Verification:** Use `wscat` or browser console to connect and subscribe

---

### Task 8: Root Layout + Design System (Layer 4)

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Sprint** | S1 |
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
   - `Sidebar.tsx` — collapsible nav sidebar with links: Home, Agents, Sessions, Projects, Workspace, Terminal
   - `Header.tsx` — top bar with page title
5. Create `src/app/(main)/layout.tsx` — wraps pages with Sidebar + Header

**Acceptance criteria:**
- Dark background renders (#0a0e27)
- Sidebar shows 6 navigation links
- UI components render with correct colors and styles
- No animations faster than 150ms

**Verification:** `bun dev` → navigate to localhost:4000, visually confirm dark theme + sidebar

---

### Task 9: Zustand Stores + Custom Hooks

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Sprint** | S3 |
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
| **Sprint** | S3 |
| **Depends on** | Tasks 6, 8, 9 |
| **Scope** | `dashboard/src/app/(main)/page.tsx` |
| **Branch** | `feat/task-10-dashboard-home` |

**What to do:**
1. Create the `/` route page (inside `(main)` layout group)
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
| **Sprint** | S3 |
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
| **Sprint** | S3 |
| **Depends on** | Tasks 6, 8, 9 |
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
| **Sprint** | S4 |
| **Depends on** | Tasks 7, 12 |
| **Scope** | `dashboard/src/components/features/session-monitor/TerminalView.tsx` |
| **Branch** | `feat/task-13-live-terminal` |

**What to do:**
1. Create `TerminalView.tsx` — xterm.js terminal component
2. Use `@xterm/addon-fit` for responsive sizing
3. Use `@xterm/addon-web-links` for clickable URLs
4. Connect to WebSocket via `useWebSocket` hook
5. Subscribe to session output by session ID
6. Send keyboard input back through WebSocket to PTY stdin
7. Render ANSI-colored output in the terminal
8. Handle terminal resize events
9. Configure scrollback buffer (10,000 lines)
10. Dynamic import with `ssr: false` (xterm.js requires DOM)
11. Integrate into `/sessions/[id]` page

**Acceptance criteria:**
- Terminal renders with dark background matching design system
- Live output streams in real-time from WebSocket
- ANSI colors render correctly
- Keyboard input works (bidirectional)
- Terminal resizes when window resizes
- Scrollback works

**Verification:** Launch an agent session, navigate to its detail page, see live terminal output

---

### Task 14: Project Registry Page

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Sprint** | S3 |
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
| **Sprint** | S3 |
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

### Task 16: General-Purpose CLI Terminal

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Sprint** | S4 |
| **Depends on** | Tasks 7, 13 |
| **Scope** | `dashboard/src/app/api/terminal/route.ts`, `dashboard/src/app/(main)/terminal/page.tsx` |
| **Branch** | `feat/task-16-general-cli` |

**What to do:**
1. Create `POST /api/terminal` endpoint — spawns a raw shell (bash on Unix, cmd/powershell on Windows) via PTY
2. Returns a session ID for WebSocket subscription
3. Create `/terminal` page with embedded `TerminalView` component
4. Full bidirectional terminal — user types commands, sees output
5. Add "Terminal" link to sidebar navigation

**Acceptance criteria:**
- Can open a terminal from the dashboard
- Can type commands and see output
- Shell persists until explicitly closed
- Terminal link appears in sidebar

**Verification:** Navigate to `/terminal`, type `ls` or `dir`, see directory listing

---

## DevOps Tasks (Sprint 5)

### DevOps 1: ESLint + Prettier

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Branch** | `feat/devops-linting` |

Install and configure ESLint (flat config, Next.js + TypeScript) and Prettier (single quotes, semis, trailing commas, 2-space, 100 width). Add `lint`, `format`, `format:check` scripts to `package.json`.

### DevOps 2: GitHub Actions CI

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Branch** | `feat/devops-ci` |

Create `.github/workflows/ci.yml`: checkout → setup-bun → install → lint → format:check → typecheck → test → build. Trigger on push + PR.

### DevOps 3: DB Init Script

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Branch** | `feat/devops-db-init` |

Create `dashboard/scripts/db-init.ts`: ensures `.agent-os/` dirs exist, runs Drizzle migrations, verifies DB access. Add `db:init` script.

### DevOps 4: Production Startup

| Field | Value |
|-------|-------|
| **Status** | `NOT STARTED` |
| **Branch** | `feat/devops-production` |

Create `dashboard/scripts/start.sh`: `bun run db:init && bun run build && bun run start`. Verify `bun build && bun start` works on port 4000.

---

## Coordination Protocol

### Claiming a Task

1. Read `project-status.md` — check "Currently In Progress" table
2. If task is unclaimed, add your claim:
   ```
   | Task 3: Agent Service | Claude Code | 2026-03-14 15:30 | feat/task-03-agent-service |
   ```
3. Create branch: `git checkout -b feat/task-{id}-{short-name}`

### Scope Isolation

Each task specifies its file scope. Only modify files within your task's scope. If you need a file outside scope, a dependency hasn't been completed yet.

### Completing a Task

1. Run all quality gates
2. Update `project-status.md` (move from In Progress → Completion Log)
3. Commit and merge to main

### Quality Gates

```bash
cd dashboard
bunx tsc --noEmit       # Zero TS errors
bun test                # All tests pass
bun run build           # Next.js build succeeds
bun dev                 # Starts on :4000
```

### Hard Rules

- Do NOT build Tier 2 features (workflow builder, task router, knowledge base UI, auto-context injection)
- Do NOT modify files outside your task scope
- Do NOT skip quality gates
- Do NOT use npm/pnpm/yarn — Bun only
