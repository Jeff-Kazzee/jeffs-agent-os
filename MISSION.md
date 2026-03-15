# MISSION: Build Jeff's Agent OS Dashboard — Phase 1 MVP

**Target Agent:** Claude Code, Codex CLI, or any agent with filesystem + terminal access
**Estimated Scope:** ~40 files, ~3000 lines of code
**Runtime:** Bun
**Framework:** Next.js 15 (App Router) + TypeScript + Tailwind v4

---

## CONTEXT

You are building the web dashboard for "Jeff's Agent OS" — a local-first command center for controlling multiple AI coding agents from one interface. The full planning docs exist at:

- `docs/PRD.md` — Product requirements, user stories, acceptance criteria
- `docs/DESIGN.md` — UI/UX, screens, component inventory, design system
- `docs/ARCHITECTURE.md` — Tech stack, data models, directory structure, API design

**Read all three docs before writing any code.** They are your source of truth.

This is a **single-user, local-only** app. No auth. No cloud. Runs on `localhost:4000`. The user is Jeff — a power user who runs 6+ AI coding agents daily and needs a unified dashboard.

---

## WHAT TO BUILD (Phase 1 MVP Only)

Build ONLY these 6 features. Nothing else. No Tier 2 features. No workflow builder. No task router. Just these:

### 1. Agent Registry
- Auto-detect installed agents by scanning PATH for known binaries: `claude`, `gemini`, `codex`, `opencode`, `qwen`, `kumi`, `quine`, `rootcode`
- Manual agent registration (name, binary path, model, capabilities)
- Store agent configs in SQLite + `.agent-os/registry/` JSON files
- API: `GET /api/agents`, `POST /api/agents`, `POST /api/agents/auto-detect`

### 2. Agent Launcher
- Launch any registered agent from the dashboard
- Specify working directory + optional task prompt
- Use `node-pty` to spawn agent process in a PTY
- Generate session ID, track in SQLite
- API: `POST /api/sessions` (creates session + launches agent)

### 3. Session Monitor
- Real-time view of all active agent sessions
- Live terminal output via xterm.js in the browser
- WebSocket connection for streaming PTY output to frontend
- Session list with: agent name, working dir, start time, status (running/stopped/error)
- API: `GET /api/sessions`, `GET /api/sessions/:id`, WebSocket at `/api/sessions/ws`

### 4. Dashboard Home
- Overview page showing:
  - Active sessions count + list (compact cards)
  - Registered agents with status indicators (green = found on PATH, gray = not found)
  - Recent activity feed (last 10 session starts/stops)
  - Quick action buttons: "New Session", "Scan Agents", "Scan Projects"
- This is the `/` route

### 5. Project Registry
- Auto-scan workspace dirs (`projects/personal/`, `projects/business/`, `projects/tools/`) for projects (directories containing `package.json` or `CLAUDE.md`)
- Register projects with: name, path, stack, status, category
- Quick-launch: click a project → pick an agent → launch session in that project's directory
- API: `GET /api/projects`, `POST /api/projects/auto-scan`

### 6. Workspace Tree
- File tree view of the Agent OS workspace root
- Show: `business/`, `projects/`, `knowledge/`, `planning/` with expandable nodes
- Click a directory to see contents
- Click a project to open its detail view
- API: `GET /api/workspace?path=<relative-path>`

---

## TECH STACK (Non-Negotiable)

```
Runtime:         Bun (NOT npm, NOT pnpm, NOT yarn)
Framework:       Next.js 15 with App Router
Language:        TypeScript (strict mode)
Styling:         Tailwind CSS v4
State:           Zustand
Database:        SQLite via better-sqlite3 + Drizzle ORM
Terminal:        xterm.js (browser) + node-pty (server)
WebSocket:       ws library
Validation:      Zod
Port:            4000
```

---

## DIRECTORY STRUCTURE

Create the dashboard app inside the existing `dashboard/` folder:

```
dashboard/
├── src/
│   ├── app/
│   │   ├── layout.tsx              # Root layout (dark mode, Inter + JetBrains Mono fonts)
│   │   ├── page.tsx                # Dashboard home
│   │   ├── globals.css             # Tailwind + custom properties
│   │   ├── (main)/
│   │   │   ├── layout.tsx          # Sidebar + header layout
│   │   │   ├── agents/page.tsx     # Agent registry
│   │   │   ├── sessions/page.tsx   # Session monitor
│   │   │   ├── sessions/[id]/page.tsx
│   │   │   ├── projects/page.tsx   # Project registry
│   │   │   └── workspace/page.tsx  # Workspace tree
│   │   └── api/
│   │       ├── agents/route.ts
│   │       ├── agents/auto-detect/route.ts
│   │       ├── sessions/route.ts
│   │       ├── sessions/[id]/route.ts
│   │       ├── sessions/ws/route.ts
│   │       ├── projects/route.ts
│   │       ├── projects/auto-scan/route.ts
│   │       ├── workspace/route.ts
│   │       └── health/route.ts
│   ├── components/
│   │   ├── ui/                     # Primitives (Button, Card, Input, Badge, Modal)
│   │   ├── features/
│   │   │   ├── agent-registry/     # AgentList, AgentCard, RegisterAgentForm
│   │   │   ├── session-monitor/    # SessionList, SessionCard, TerminalView
│   │   │   ├── project-registry/   # ProjectList, ProjectCard
│   │   │   └── workspace-tree/     # WorkspaceTree, TreeNode
│   │   └── layout/                 # Sidebar, Header, Navigation
│   ├── lib/
│   │   ├── core/                   # TypeScript types (Layer 1)
│   │   │   ├── agent.types.ts
│   │   │   ├── session.types.ts
│   │   │   └── project.types.ts
│   │   ├── services/               # Business logic (Layer 2)
│   │   │   ├── agent-service.ts
│   │   │   ├── session-service.ts
│   │   │   └── project-service.ts
│   │   ├── db/
│   │   │   ├── schema.ts           # Drizzle schema
│   │   │   ├── client.ts           # SQLite connection
│   │   │   └── migrations/
│   │   ├── stores/                 # Zustand stores
│   │   │   ├── ui-store.ts
│   │   │   └── session-store.ts
│   │   ├── hooks/
│   │   │   ├── useAgents.ts
│   │   │   ├── useSessions.ts
│   │   │   └── useWebSocket.ts
│   │   ├── utils/
│   │   │   └── path-utils.ts
│   │   └── validations/
│   │       ├── agent-schema.ts
│   │       └── session-schema.ts
│   └── constants/
│       ├── agents.ts               # Known agent binaries + defaults
│       └── paths.ts                # Workspace path constants
├── tests/
│   └── services/
│       ├── agent-service.test.ts
│       └── session-service.test.ts
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── package.json
├── drizzle.config.ts
└── .env
```

---

## DATA MODELS

### SQLite Schema (Drizzle ORM)

```typescript
// src/lib/db/schema.ts

import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const agents = sqliteTable('agents', {
  id: text('id').primaryKey(),                    // nanoid
  name: text('name').notNull(),                   // "Claude Code"
  slug: text('slug').notNull().unique(),          // "claude-code"
  binaryPath: text('binary_path').notNull(),      // "/usr/local/bin/claude"
  binaryName: text('binary_name').notNull(),      // "claude"
  model: text('model'),                           // "claude-sonnet-4"
  description: text('description'),
  capabilities: text('capabilities'),             // JSON array: ["architecture","refactoring"]
  status: text('status').notNull().default('unknown'), // "available" | "not_found" | "unknown"
  config: text('config'),                         // JSON blob for extra config
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull(),
});

export const sessions = sqliteTable('sessions', {
  id: text('id').primaryKey(),                    // nanoid
  agentId: text('agent_id').notNull().references(() => agents.id),
  projectId: text('project_id'),                  // nullable — not all sessions are project-scoped
  taskDescription: text('task_description'),
  workingDir: text('working_dir').notNull(),
  status: text('status').notNull().default('starting'), // "starting" | "running" | "stopped" | "error"
  pid: integer('pid'),                            // OS process ID
  startTime: integer('start_time', { mode: 'timestamp' }),
  endTime: integer('end_time', { mode: 'timestamp' }),
  exitCode: integer('exit_code'),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull(),
});

export const projects = sqliteTable('projects', {
  id: text('id').primaryKey(),                    // nanoid
  name: text('name').notNull(),
  path: text('path').notNull().unique(),
  stack: text('stack'),                           // "next.js, typescript, tailwind"
  status: text('status').notNull().default('active'), // "planning" | "active" | "paused" | "done"
  category: text('category').notNull().default('personal'), // "personal" | "business" | "tool"
  description: text('description'),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull(),
});

export const sessionLogs = sqliteTable('session_logs', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  sessionId: text('session_id').notNull().references(() => sessions.id),
  chunk: text('chunk').notNull(),                 // raw terminal output chunk
  streamType: text('stream_type').notNull().default('stdout'), // "stdout" | "stderr"
  timestamp: integer('timestamp', { mode: 'timestamp' }).notNull(),
});
```

---

## DESIGN SYSTEM (Key Values)

```css
/* Dark mode default — these go in globals.css */
--bg-primary: #0a0e27;        /* Deep navy-black */
--bg-secondary: #111631;      /* Slightly lighter */
--bg-tertiary: #1a1f3d;       /* Card backgrounds */
--bg-hover: #232850;           /* Hover states */

--text-primary: #e0e0e0;      /* Main text */
--text-secondary: #8b8fa3;    /* Muted text */
--text-tertiary: #5a5f7a;     /* Disabled/hint */

--accent-blue: #3b82f6;       /* Primary actions */
--accent-green: #22c55e;      /* Success / running */
--accent-red: #ef4444;        /* Error / stopped */
--accent-amber: #f59e0b;      /* Warning / starting */
--accent-purple: #a855f7;     /* Info / special */

--border-default: #1e2340;    /* Subtle borders */
--border-active: #3b82f6;     /* Focused/active borders */

--font-sans: 'Inter', system-ui, sans-serif;
--font-mono: 'JetBrains Mono', 'Fira Code', monospace;

--radius-sm: 4px;
--radius-md: 6px;
--radius-lg: 8px;
```

**Key aesthetic principles:**
- Terminal/hacker aesthetic, polished — think Linear meets Warp
- Dense information layout (not spacious marketing-site vibes)
- Monospace for all code/terminal/agent output
- Sidebar navigation, collapsible
- Keyboard-first (add shortcuts after core UI works)
- NO flashy animations (Jeff is epileptic) — smooth 150ms transitions only
- Status dots: green = running/available, amber = starting, red = error/stopped, gray = unknown

---

## KNOWN AGENT BINARIES

```typescript
// src/constants/agents.ts
export const KNOWN_AGENTS = [
  { name: 'Claude Code', binary: 'claude', model: 'claude-sonnet-4', capabilities: ['architecture', 'refactoring', 'complex-reasoning'] },
  { name: 'Gemini CLI', binary: 'gemini', model: 'gemini-2.5-pro', capabilities: ['research', 'large-context', 'breadth'] },
  { name: 'Codex CLI', binary: 'codex', model: 'codex', capabilities: ['speed', 'quick-edits', 'scripting'] },
  { name: 'Open Code', binary: 'opencode', model: 'varies', capabilities: ['multi-model', 'flexibility'] },
  { name: 'Quen Code', binary: 'qwen', model: 'qwen-2.5-coder', capabilities: ['free-tier', 'simple-tasks'] },
  { name: 'Kumi CLI', binary: 'kumi', model: 'varies', capabilities: ['general'] },
  { name: 'Quine', binary: 'quine', model: 'varies', capabilities: ['general'] },
  { name: 'Root Code', binary: 'rootcode', model: 'varies', capabilities: ['general'] },
] as const;
```

---

## WORKSPACE PATHS

```typescript
// src/constants/paths.ts
import { resolve } from 'path';

// The Agent OS workspace root — parent of dashboard/
export const WORKSPACE_ROOT = resolve(__dirname, '../../..');
export const AGENT_OS_DIR = resolve(WORKSPACE_ROOT, '.agent-os');
export const REGISTRY_DIR = resolve(AGENT_OS_DIR, 'registry');
export const SESSIONS_DIR = resolve(AGENT_OS_DIR, 'sessions');
export const LOGS_DIR = resolve(AGENT_OS_DIR, 'logs');
export const KNOWLEDGE_DIR = resolve(WORKSPACE_ROOT, 'knowledge');
export const PROJECTS_DIR = resolve(WORKSPACE_ROOT, 'projects');
export const BUSINESS_DIR = resolve(WORKSPACE_ROOT, 'business');
export const PLANNING_DIR = resolve(WORKSPACE_ROOT, 'planning');
export const DATABASE_PATH = resolve(AGENT_OS_DIR, 'agent-os.db');
```

---

## EXECUTION ORDER

Do this in order. Commit after each step.

### Step 1: Scaffold
```bash
cd dashboard
bun init
bun add next@latest react@latest react-dom@latest typescript@latest
bun add tailwindcss@latest @tailwindcss/postcss postcss
bun add drizzle-orm better-sqlite3 zod zustand nanoid
bun add -d drizzle-kit @types/better-sqlite3 @types/react @types/react-dom
bun add xterm @xterm/addon-fit @xterm/addon-web-links
bun add ws
bun add -d @types/ws
```

Create `next.config.ts`, `tsconfig.json`, `tailwind.config.ts`, `.env`, `package.json` scripts:
```json
{
  "scripts": {
    "dev": "next dev -p 4000",
    "build": "next build",
    "start": "next start -p 4000",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "test": "bun test"
  }
}
```

NOTE on node-pty: This package requires native compilation. Install it separately:
```bash
bun add node-pty
```
If node-pty fails to compile, fall back to `Bun.spawn` with `{ stdin: 'pipe', stdout: 'pipe', stderr: 'pipe' }` as the PTY layer. The session service should abstract this so the rest of the app doesn't care.

### Step 2: Database + Types (Layer 1)
- Create `src/lib/db/schema.ts` (use schema above)
- Create `src/lib/db/client.ts` (initialize SQLite connection with WAL mode)
- Create `src/lib/core/agent.types.ts`, `session.types.ts`, `project.types.ts`
- Create Zod validation schemas in `src/lib/validations/`
- Run `drizzle-kit generate` + `drizzle-kit migrate`
- **Test:** Write a test that creates the DB, inserts an agent, queries it back

### Step 3: Services (Layer 2)
- `agent-service.ts`: AgentRegistry class — CRUD agents, auto-detect via `which <binary>`
- `session-service.ts`: SessionManager class — spawn agent via PTY, track session, capture output
- `project-service.ts`: ProjectScanner — scan directories for projects
- **Test:** Write tests for agent auto-detection and project scanning (mock filesystem if needed)

### Step 4: API Routes (Layer 3)
- Wire up all API routes listed above
- Each route: validate input with Zod, call service, return JSON
- Health endpoint returns `{ status: 'ok', version: '0.1.0', uptime: ... }`
- WebSocket endpoint at `/api/sessions/ws` — clients subscribe to session output by session ID
- **Test:** curl each endpoint to verify it works

### Step 5: UI (Layer 4)
- Root layout: dark mode, font imports, global styles
- Sidebar navigation: Home, Agents, Sessions, Projects, Workspace
- Dashboard home page (route `/`)
- Agent registry page (`/agents`)
- Session monitor page (`/sessions`) + session detail page (`/sessions/[id]`)
- Project registry page (`/projects`)
- Workspace tree page (`/workspace`)
- **Test:** `bun dev` and visually verify each page works

### Step 6: Live Terminal
- Integrate xterm.js in the session detail page
- Connect to WebSocket for live output
- Handle resize, scrollback, ANSI color rendering
- **Test:** Launch an agent from the dashboard, verify terminal output streams live

---

## ACCEPTANCE CRITERIA (How to Know You're Done)

- [ ] `bun dev` starts dashboard on port 4000 without errors
- [ ] Dashboard home shows registered agents and any active sessions
- [ ] Auto-detect finds at least `claude` binary (or whatever's installed)
- [ ] Can manually register a new agent from the UI
- [ ] Can launch a session (pick agent + working dir) and see it appear in active sessions
- [ ] Session detail page shows live terminal output via xterm.js
- [ ] Project auto-scan finds directories in `projects/` that contain `package.json`
- [ ] Workspace tree shows the folder hierarchy and is expandable
- [ ] All API endpoints return valid JSON
- [ ] SQLite database persists between restarts
- [ ] UI is dark mode, uses the design system colors, feels like a power tool not a marketing site
- [ ] No TypeScript errors (`bun run build` succeeds)
- [ ] At least 5 service-level tests pass

---

## CONSTRAINTS

- **DO NOT** build Tier 2 features (workflow builder, task router, knowledge base UI, auto-context injection). Those come later.
- **DO NOT** add authentication. Single user, local only.
- **DO NOT** use npm, pnpm, or yarn. Bun only.
- **DO NOT** use pages router. App Router only.
- **DO NOT** add external cloud dependencies.
- **DO NOT** use flashy animations or bright light themes (epilepsy constraint).
- **DO** commit frequently with conventional commit messages (`feat:`, `fix:`, `chore:`).
- **DO** write tests for services before implementing features when practical.
- **DO** read `docs/ARCHITECTURE.md` for the full directory structure and data models.
- **DO** read `docs/DESIGN.md` for the complete design system and screen specifications.
