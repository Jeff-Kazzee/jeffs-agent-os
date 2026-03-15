# Jeff's Agent OS - Architecture Document

**Version:** 0.1.0
**Last Updated:** 2026-03-14
**Author:** Jeff Kazzee + Claude
**PRD Reference:** `/docs/PRD.md`
**Design Doc Reference:** `/docs/DESIGN.md`
**Status:** Draft

---

## 1. System Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│               (Next.js Web Dashboard - localhost)                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  React Components (Tailwind CSS)                         │   │
│  │  - Agent Registry View                                   │   │
│  │  - Session Monitor                                       │   │
│  │  - Project Navigator                                     │   │
│  │  - Task Router                                           │   │
│  │  - Knowledge Base Search                                 │   │
│  └───────────────────────────┬────────────────────────────┘    │
│                              │                                   │
│  ┌──────────────────────────┴────────────────────────────┐     │
│  │  Zustand Store (Client State)                         │     │
│  │  - UI State (sidebars, modals)                        │     │
│  │  - Session State (live updates)                       │     │
│  └──────────────────────────┬────────────────────────────┘     │
└──────────────────────────────┼────────────────────────────────┘
                               │ HTTP / WebSocket
┌──────────────────────────────┼────────────────────────────────┐
│                         API Layer                              │
│           (Next.js API Routes + WebSocket Server)             │
│  ┌──────────────────────────┴────────────────────────────┐    │
│  │  REST API Routes (/api/*)                            │    │
│  │  - GET /api/agents (list registered agents)          │    │
│  │  - POST /api/sessions (create session)               │    │
│  │  - GET /api/sessions/:id (get session details)       │    │
│  │  - GET /api/projects (list projects)                 │    │
│  │  - GET /api/knowledge (search knowledge base)        │    │
│  └──────────────────────────┬────────────────────────────┘    │
│                             │                                   │
│  ┌──────────────────────────┴────────────────────────────┐    │
│  │  WebSocket Server (/ws)                              │    │
│  │  - Real-time session output streaming                │    │
│  │  - Session lifecycle events (start, end, error)      │    │
│  └──────────────────────────┬────────────────────────────┘    │
└──────────────────────────────┼────────────────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
┌───────────────────┐ ┌────────────────┐ ┌──────────────────┐
│ Session Manager   │ │ Agent Registry │ │ Project Registry │
│ - PTY launch      │ │ - Agent config │ │ - Project list   │
│ - Output capture  │ │ - Auto-detect  │ │ - Project dirs   │
│ - State tracking  │ │ - Capabilities │ │ - Metadata       │
└────────┬──────────┘ └────────────────┘ └──────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  SQLite Database (WAL mode)                         │   │
│  │  - Agents table                                     │   │
│  │  - Sessions table                                   │   │
│  │  - Projects table                                   │   │
│  │  - Session logs table                              │   │
│  │  - Workflows table (Tier 2)                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  File System (JSON + Markdown)                      │   │
│  │  - .agent-os/registry/*.json (agent definitions)    │   │
│  │  - knowledge/** (Obsidian-compatible notes)         │   │
│  │  - .agent-os/sessions/** (session metadata)         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  External Processes                                 │   │
│  │  - Child agent processes (PTY-managed)              │   │
│  │  - Claude Desktop monitoring                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Key Characteristics

- **Type:** Monolith (Next.js handles both UI and API)
- **Deployment:** Local-only, single machine (macOS primary)
- **Scale Target:** Single user, 10+ concurrent agent sessions
- **Architecture Pattern:** Four-layer (Core Types → Services → Actions → UI)
- **IPC:** WebSockets for real-time session updates
- **Process Management:** node-pty for spawning/controlling agent processes

---

## 2. Tech Stack

### Frontend

| Layer | Technology | Version | Rationale |
|-------|------------|---------|-----------|
| Framework | Next.js | 15.x with App Router | Modern meta-framework with SSR/SSG capabilities. App Router enables file-based routing. Built-in API routes eliminate need for separate backend. |
| Styling | Tailwind CSS | 4.x+ | Utility-first CSS. Zero runtime overhead. Excellent DX for rapid UI development. No config needed with v4. |
| State Management | Zustand | 4.x | Lightweight, performant store. Minimal boilerplate. Perfect for local UI state (modals, sidebars) and session state. |
| Terminal Rendering | xterm.js | 5.x | Industry-standard terminal emulator for browsers. Handles ANSI codes, resizing, proper scrollback. |
| HTTP Client | fetch API + custom wrapper | - | Native fetch is sufficient for local API. Custom wrapper for error handling and type safety. |

### Backend

| Layer | Technology | Version | Rationale |
|-------|------------|---------|-----------|
| Runtime | Bun | 1.x | Jeff's preferred runtime. Native TypeScript support. ~3x faster than Node.js for CLI use. Excellent module resolution. |
| Framework | Next.js API Routes | 15.x | Integrated with Next.js. File-based route structure. Minimal setup. No separate Express server needed. |
| Process Management | node-pty | 0.10.x | Cross-platform PTY for capturing agent output. Preserves terminal colors, dimensions, signals. Alternative: use Bun's built-in shell capabilities with fallback to node-pty. |
| WebSocket | ws library | 8.x | Lightweight WebSocket server. Can run alongside Next.js. Alternative: use Socket.io for more features (but more overhead). |
| Validation | Zod | 3.x | TypeScript-first schema validation. Used for API input/output validation and TypeScript inference. |
| Type Definitions | TypeScript | 5.x+ | Full type safety across codebase. Better DX and fewer runtime errors. |

### Database

| Type | Technology | Rationale |
|------|------------|-----------|
| Primary | SQLite (better-sqlite3 or Drizzle ORM) | Local-first, zero config. WAL mode for concurrent reads. No external server required. Backup is just copying a file. |
| ORM | Drizzle ORM (optional) | Type-safe queries with TypeScript inference. Migration support. Makes schema changes easier. Alternative: raw SQL with type stubs if simpler is better. |
| Schema | SQL DDL + TypeScript interfaces | Source of truth for data structure. Migrations tracked in version control. |

### Infrastructure

| Service | Provider | Rationale |
|---------|----------|-----------|
| Hosting | Local (localhost:3000 or 4000) | Single-user, local-only. No cloud deployment. Dev server can restart on code changes. |
| Terminal | xterm.js | Browser-native terminal rendering. No external service. |
| File System | Local disk | Knowledge base, registry, session logs stored in workspace folders. Obsidian-compatible. |

### Development Tools

| Purpose | Tool | Rationale |
|---------|------|-----------|
| Runtime & Package Manager | Bun | Fast, modern, integrated TypeScript support. |
| Linting | ESLint + Prettier | Industry standard. Automatic code formatting. |
| Testing | Bun test + Playwright | Bun's built-in test runner. Playwright for E2E browser testing. |
| CI/CD | GitHub Actions | Free for public/private repos. Standard for TypeScript projects. |
| Database | SQLite with better-sqlite3 | Zero config, single file, WAL mode for concurrency. |

---

## 3. Directory Structure

```
Jeffs Agent OS/
├── .agent-os/                               # Agent OS internal state
│   ├── config.json                          # Global configuration
│   ├── registry/                            # Agent definitions (JSON per agent)
│   │   ├── claude-code.json
│   │   ├── gemini-cli.json
│   │   └── [agent-name].json
│   ├── sessions/                            # Session metadata
│   │   └── [session-id]/
│   │       ├── metadata.json
│   │       └── output.log
│   └── logs/                                # System logs
│
├── business/                                # Business operations
│   ├── plans/
│   ├── finances/
│   ├── brand/
│   └── social-media/
│
├── projects/                                # Coding projects
│   ├── personal/
│   ├── business/
│   └── tools/
│
├── knowledge/                               # Obsidian-compatible knowledge base
│   ├── me/
│   ├── business/
│   ├── tech/
│   ├── agents/
│   ├── social/
│   └── templates/
│
├── planning/                                # Project planning
│   ├── active/
│   ├── backlog/
│   └── archive/
│
├── dashboard/                               # Next.js application (or src/)
│   ├── src/
│   │   ├── app/                             # Next.js App Router
│   │   │   ├── layout.tsx                   # Root layout
│   │   │   ├── page.tsx                     # Home page
│   │   │   ├── (main)/                      # Route group for main app
│   │   │   │   ├── agents/
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── [id]/page.tsx
│   │   │   │   ├── sessions/
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── [id]/page.tsx
│   │   │   │   ├── projects/
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── [id]/page.tsx
│   │   │   │   ├── knowledge/
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── search/page.tsx
│   │   │   │   ├── workflows/               # Tier 2
│   │   │   │   │   ├── page.tsx
│   │   │   │   │   └── [id]/page.tsx
│   │   │   │   └── layout.tsx               # Main layout
│   │   │   └── api/                         # API routes
│   │   │       ├── agents/
│   │   │       │   ├── route.ts             # GET /api/agents
│   │   │       │   ├── [id]/
│   │   │       │   │   └── route.ts         # GET /api/agents/:id
│   │   │       │   └── auto-detect/
│   │   │       │       └── route.ts         # POST /api/agents/auto-detect
│   │   │       ├── sessions/
│   │   │       │   ├── route.ts             # POST /api/sessions
│   │   │       │   ├── [id]/
│   │   │       │   │   ├── route.ts         # GET /api/sessions/:id
│   │   │       │   │   └── output/route.ts  # GET /api/sessions/:id/output
│   │   │       │   └── ws/
│   │   │       │       └── route.ts         # WebSocket endpoint
│   │   │       ├── projects/
│   │   │       │   ├── route.ts
│   │   │       │   ├── [id]/route.ts
│   │   │       │   └── auto-scan/route.ts
│   │   │       ├── knowledge/
│   │   │       │   ├── route.ts             # GET /api/knowledge (search)
│   │   │       │   └── [slug]/route.ts      # GET /api/knowledge/:slug
│   │   │       ├── workflows/               # Tier 2
│   │   │       │   ├── route.ts
│   │   │       │   └── [id]/route.ts
│   │   │       └── health/
│   │   │           └── route.ts             # GET /api/health
│   │   ├── components/
│   │   │   ├── ui/                          # Primitive UI components
│   │   │   │   ├── button.tsx
│   │   │   │   ├── card.tsx
│   │   │   │   ├── modal.tsx
│   │   │   │   ├── input.tsx
│   │   │   │   ├── dropdown.tsx
│   │   │   │   └── ...
│   │   │   ├── features/                    # Feature-specific components
│   │   │   │   ├── agent-registry/
│   │   │   │   │   ├── AgentList.tsx
│   │   │   │   │   ├── AgentCard.tsx
│   │   │   │   │   ├── RegisterAgentForm.tsx
│   │   │   │   │   └── AgentDetailModal.tsx
│   │   │   │   ├── session-monitor/
│   │   │   │   │   ├── SessionList.tsx
│   │   │   │   │   ├── SessionCard.tsx
│   │   │   │   │   ├── TerminalView.tsx
│   │   │   │   │   └── SessionOutputPanel.tsx
│   │   │   │   ├── workspace-tree/
│   │   │   │   │   ├── WorkspaceTree.tsx
│   │   │   │   │   ├── TreeNode.tsx
│   │   │   │   │   └── ProjectQuickLaunch.tsx
│   │   │   │   ├── project-registry/
│   │   │   │   │   ├── ProjectList.tsx
│   │   │   │   │   ├── ProjectCard.tsx
│   │   │   │   │   └── ProjectDetailPanel.tsx
│   │   │   │   ├── task-router/            # Tier 2
│   │   │   │   │   ├── TaskForm.tsx
│   │   │   │   │   ├── AgentSelector.tsx
│   │   │   │   │   └── TaskPreview.tsx
│   │   │   │   ├── knowledge-base/         # Tier 2
│   │   │   │   │   ├── KnowledgeSearch.tsx
│   │   │   │   │   ├── NoteViewer.tsx
│   │   │   │   │   ├── NoteEditor.tsx
│   │   │   │   │   └── WikilinkRenderer.tsx
│   │   │   │   └── workflow-builder/       # Tier 2
│   │   │   │       ├── WorkflowCanvas.tsx
│   │   │   │       ├── WorkflowStep.tsx
│   │   │   │       └── WorkflowExecutor.tsx
│   │   │   └── layout/
│   │   │       ├── Sidebar.tsx
│   │   │       ├── Header.tsx
│   │   │       ├── Navigation.tsx
│   │   │       └── MainLayout.tsx
│   │   ├── lib/
│   │   │   ├── core/                       # Core types & interfaces (Layer 1)
│   │   │   │   ├── agent.types.ts
│   │   │   │   ├── session.types.ts
│   │   │   │   ├── project.types.ts
│   │   │   │   ├── workflow.types.ts        # Tier 2
│   │   │   │   └── knowledge.types.ts
│   │   │   │
│   │   │   ├── services/                   # Business logic (Layer 2)
│   │   │   │   ├── agent-service.ts
│   │   │   │   │   ├── AgentRegistry
│   │   │   │   │   ├── AgentDetector
│   │   │   │   │   └── AgentValidator
│   │   │   │   ├── session-service.ts
│   │   │   │   │   ├── SessionManager
│   │   │   │   │   ├── PTYHandler
│   │   │   │   │   └── OutputCapture
│   │   │   │   ├── project-service.ts
│   │   │   │   │   ├── ProjectRegistry
│   │   │   │   │   └── ProjectScanner
│   │   │   │   ├── knowledge-service.ts
│   │   │   │   │   ├── KnowledgeLoader
│   │   │   │   │   ├── KnowledgeSearch
│   │   │   │   │   └── WikilinkResolver
│   │   │   │   ├── workflow-service.ts     # Tier 2
│   │   │   │   │   ├── WorkflowBuilder
│   │   │   │   │   └── WorkflowExecutor
│   │   │   │   └── task-router.ts          # Tier 2
│   │   │   │       └── TaskDispatcher
│   │   │   │
│   │   │   ├── actions/                    # API actions (Layer 3)
│   │   │   │   ├── agents/
│   │   │   │   │   ├── register-agent.ts
│   │   │   │   │   ├── list-agents.ts
│   │   │   │   │   ├── delete-agent.ts
│   │   │   │   │   └── auto-detect-agents.ts
│   │   │   │   ├── sessions/
│   │   │   │   │   ├── create-session.ts
│   │   │   │   │   ├── get-session.ts
│   │   │   │   │   ├── list-sessions.ts
│   │   │   │   │   ├── kill-session.ts
│   │   │   │   │   └── stream-session-output.ts
│   │   │   │   ├── projects/
│   │   │   │   │   ├── create-project.ts
│   │   │   │   │   ├── list-projects.ts
│   │   │   │   │   ├── get-project.ts
│   │   │   │   │   └── auto-scan-projects.ts
│   │   │   │   ├── knowledge/
│   │   │   │   │   ├── search-knowledge.ts
│   │   │   │   │   ├── get-note.ts
│   │   │   │   │   └── create-note.ts
│   │   │   │   └── workflows/              # Tier 2
│   │   │   │       ├── create-workflow.ts
│   │   │   │       └── execute-workflow.ts
│   │   │   │
│   │   │   ├── db/                         # Database layer (SQLite)
│   │   │   │   ├── schema.ts               # Drizzle schema definitions
│   │   │   │   ├── client.ts               # SQLite connection
│   │   │   │   ├── migrations/             # SQL migration files
│   │   │   │   └── queries/
│   │   │   │       ├── agents.ts
│   │   │   │       ├── sessions.ts
│   │   │   │       ├── projects.ts
│   │   │   │       ├── logs.ts
│   │   │   │       └── workflows.ts        # Tier 2
│   │   │   │
│   │   │   ├── api-client/                 # Client-side API helpers
│   │   │   │   ├── agent-api.ts
│   │   │   │   ├── session-api.ts
│   │   │   │   ├── project-api.ts
│   │   │   │   ├── knowledge-api.ts
│   │   │   │   └── workflow-api.ts         # Tier 2
│   │   │   │
│   │   │   ├── hooks/                      # Custom React hooks
│   │   │   │   ├── useAgents.ts
│   │   │   │   ├── useSessions.ts
│   │   │   │   ├── useProjects.ts
│   │   │   │   ├── useKnowledge.ts
│   │   │   │   ├── useWebSocket.ts
│   │   │   │   └── useWorkflows.ts         # Tier 2
│   │   │   │
│   │   │   ├── stores/                     # Zustand state stores
│   │   │   │   ├── ui-store.ts             # UI state (sidebars, modals)
│   │   │   │   ├── session-store.ts        # Session state (live updates)
│   │   │   │   ├── project-store.ts        # Project state
│   │   │   │   └── workflow-store.ts       # Tier 2
│   │   │   │
│   │   │   ├── utils/                      # Utility functions
│   │   │   │   ├── format-date.ts
│   │   │   │   ├── format-time.ts
│   │   │   │   ├── slugify.ts
│   │   │   │   ├── parse-ansi.ts           # Parse ANSI color codes
│   │   │   │   ├── path-utils.ts
│   │   │   │   ├── error-handler.ts
│   │   │   │   └── logger.ts
│   │   │   │
│   │   │   └── validations/                # Zod schemas
│   │   │       ├── agent-schema.ts
│   │   │       ├── session-schema.ts
│   │   │       ├── project-schema.ts
│   │   │       ├── knowledge-schema.ts
│   │   │       └── workflow-schema.ts      # Tier 2
│   │   │
│   │   ├── types/                          # Global TypeScript types
│   │   │   └── index.ts                    # Re-exports of core types
│   │   │
│   │   └── constants/
│   │       ├── agents.ts                   # Known agent binaries, defaults
│   │       ├── paths.ts                    # Workspace paths
│   │       └── limits.ts                   # Performance limits
│   │
│   ├── public/                             # Static assets
│   │   └── favicon.ico
│   │
│   ├── tests/
│   │   ├── unit/                           # Unit tests
│   │   │   ├── lib/utils/
│   │   │   └── components/
│   │   ├── integration/                    # Integration tests
│   │   │   ├── api/
│   │   │   └── services/
│   │   └── e2e/                            # End-to-end tests
│   │       └── workflows/
│   │
│   ├── next.config.js
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   ├── package.json
│   └── bun.lockb
│
├── docs/
│   ├── PRD.md                              # Product Requirements
│   ├── DESIGN.md                           # UI/UX Design
│   ├── ARCHITECTURE.md                     # This document
│   └── DEPLOYMENT.md                       # Deployment guide (future)
│
├── scripts/
│   ├── db-init.ts                          # Initialize SQLite database
│   ├── db-migrate.ts                       # Run Drizzle migrations
│   └── workspace-init.ts                   # Initialize workspace folders
│
├── CLAUDE.md                               # Root-level agent instructions
├── CHANGELOG.md                            # Version history
├── project-status.md                       # Current state tracking
└── .gitignore
```

### Naming Conventions

- **Files:** kebab-case (`agent-registry.ts`, `session-monitor.tsx`)
- **Components:** PascalCase (`AgentCard.tsx`, `SessionList.tsx`)
- **Hooks:** camelCase with `use` prefix (`useAgents.ts`, `useSessions.ts`)
- **Utils:** camelCase (`formatDate.ts`, `parseAnsi.ts`)
- **Constants:** SCREAMING_SNAKE_CASE (`MAX_CONCURRENT_SESSIONS`, `AGENT_REGISTRY_PATH`)
- **Services:** PascalCase class or factory (`SessionManager`, `AgentRegistry`)

---

## 4. Data Models

### Entity Relationship Diagram

```
┌──────────────────┐         ┌──────────────────┐
│     Agents       │         │    Projects      │
├──────────────────┤         ├──────────────────┤
│ id (PK)          │◄────┐   │ id (PK)          │
│ name             │     │   │ name             │
│ binary_path      │     │   │ path             │
│ model            │     │   │ stack            │
│ capabilities     │     │   │ status           │
│ config           │     │   │ category         │
│ created_at       │     │   │ created_at       │
│ updated_at       │     │   │ updated_at       │
└──────────────────┘     │   └──────────────────┘
         ▲               │              ▲
         │               │              │
         │ agent_id      │ project_id   │
         │               │              │
┌────────┴─────────────┬─┴──────────────┴───────┐
│      Sessions        │                        │
├──────────────────────┤                        │
│ id (PK)              │                        │
│ agent_id (FK)────────┘                        │
│ project_id (FK)──────────────────────────────┘
│ task_description     │
│ working_dir          │
│ status               │
│ start_time           │
│ end_time             │
│ exit_code            │
│ created_at           │
│ updated_at           │
└──────────────────────┘
         │
         │ session_id
         ▼
┌──────────────────────┐
│   Session Logs       │
├──────────────────────┤
│ id (PK)              │
│ session_id (FK)      │
│ output_chunk         │
│ output_type          │
│ timestamp            │
│ created_at           │
└──────────────────────┘

┌──────────────────────┐
│   Workflows (T2)     │
├──────────────────────┤
│ id (PK)              │
│ name                 │
│ steps (JSON)         │
│ status               │
│ created_at           │
│ updated_at           │
└──────────────────────┘
```

### Schema Definitions

#### Agent

```typescript
// Core type
interface Agent {
  id: string;                    // UUID primary key
  name: string;                  // "Claude Code", "Gemini CLI", etc.
  binary_path: string;           // "/usr/local/bin/claude" or full path
  model: string;                 // default model (e.g., "claude-sonnet-4")
  capabilities: string[];        // ["architecture", "refactoring", "code-review"]
  config: Record<string, any>;   // Agent-specific config (timeout, args, env)
  is_installed: boolean;         // Whether binary was found in PATH
  description?: string;          // Human-readable description
  created_at: Date;
  updated_at: Date;
}

// TypeScript interface for type safety
interface AgentConfig {
  timeout?: number;              // Command timeout in ms (default: 300000)
  env?: Record<string, string>;  // Additional environment variables
  args?: string[];               // Default command-line arguments
  working_dir?: string;          // Default working directory
  shell?: string;                // Shell to use (bash, zsh, etc.)
}

// Zod schema for validation
const AgentSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  binary_path: z.string().min(1),
  model: z.string().min(1),
  capabilities: z.string().array(),
  config: z.record(z.any()),
  is_installed: z.boolean(),
  description: z.string().optional(),
  created_at: z.date(),
  updated_at: z.date(),
});
```

#### Session

```typescript
interface Session {
  id: string;                    // UUID, unique session identifier
  agent_id: string;              // Foreign key to Agents
  project_id?: string;           // Foreign key to Projects (optional)
  task_description?: string;     // "Fix TypeScript error in auth module"
  working_dir: string;           // /Users/jeff/projects/business/product
  status: SessionStatus;         // "running" | "completed" | "failed" | "killed"
  pid?: number;                  // Process ID from PTY
  start_time: Date;
  end_time?: Date;
  exit_code?: number;            // Exit code when completed/failed
  output_size?: number;          // Total output bytes captured
  created_at: Date;
  updated_at: Date;
}

type SessionStatus =
  | "pending"                    // Session created, not yet started
  | "running"                    // Agent process is active
  | "completed"                  // Process exited with code 0
  | "failed"                     // Process exited with non-zero code
  | "killed"                     // Manually killed by user
  | "error";                     // Failed to launch

interface SessionEvent {
  type: "start" | "output" | "end" | "error";
  timestamp: Date;
  session_id: string;
  data?: {
    output?: string;             // stdout/stderr chunk
    error_message?: string;
    exit_code?: number;
  };
}
```

#### Project

```typescript
interface Project {
  id: string;                    // UUID
  name: string;                  // "Littler AI Company Website"
  path: string;                  // /Users/jeff/projects/business/website
  stack: string[];               // ["Next.js", "TypeScript", "Tailwind"]
  status: ProjectStatus;         // "planning" | "active" | "paused" | "done"
  category: ProjectCategory;     // "business" | "personal" | "tool"
  description?: string;
  repo_url?: string;             // GitHub URL if applicable
  created_at: Date;
  updated_at: Date;
}

type ProjectStatus = "planning" | "active" | "paused" | "done";
type ProjectCategory = "business" | "personal" | "tool";
```

#### Session Log

```typescript
interface SessionLog {
  id: string;                    // UUID
  session_id: string;            // Foreign key to Sessions
  output_chunk: string;          // Raw output (stdout/stderr chunk)
  output_type: "stdout" | "stderr";
  sequence: number;              // Order in session
  timestamp: Date;               // When this chunk was output
  created_at: Date;
}

// Efficient log storage strategy:
// - Store raw chunks as received from PTY
// - Truncate/archive old logs if they exceed size limits
// - Perform full-text search on indexed sessions
```

#### Knowledge Note

```typescript
interface KnowledgeNote {
  slug: string;                  // Unique identifier from filename
  title: string;                 // Front matter or first heading
  content: string;               // Full markdown content
  category: KnowledgeCategory;   // me | business | tech | agents | social | templates
  tags: string[];                // From front matter or auto-extracted
  wikilinks: string[];           // Extracted [[link]] references
  linked_from: string[];         // Back-references
  created_at: Date;
  updated_at: Date;
  file_path: string;             // Full path to markdown file
}

type KnowledgeCategory =
  | "me"
  | "business"
  | "tech"
  | "agents"
  | "social"
  | "templates";
```

#### Workflow (Tier 2)

```typescript
interface Workflow {
  id: string;                    // UUID
  name: string;                  // "Architecture → Implementation → Review"
  description?: string;
  steps: WorkflowStep[];         // Array of workflow steps
  status: "draft" | "active" | "archived";
  created_at: Date;
  updated_at: Date;
}

interface WorkflowStep {
  id: string;                    // UUID
  sequence: number;              // Order in workflow
  agent_id: string;              // Which agent runs this step
  prompt_template: string;       // Prompt with {{prev_output}} placeholder
  pause_after?: boolean;         // Require human approval before next step
  timeout?: number;              // Step timeout in ms
}

interface WorkflowExecution {
  id: string;                    // UUID
  workflow_id: string;
  status: "running" | "completed" | "failed" | "paused";
  step_results: {
    step_id: string;
    session_id: string;
    output: string;
    exit_code?: number;
  }[];
  created_at: Date;
  updated_at: Date;
}
```

### Database Schema (SQL)

```sql
-- Agents table
CREATE TABLE agents (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  binary_path TEXT NOT NULL,
  model TEXT NOT NULL,
  capabilities TEXT NOT NULL,           -- JSON array
  config TEXT NOT NULL,                 -- JSON object
  is_installed BOOLEAN NOT NULL,
  description TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_agents_name ON agents(name);
CREATE INDEX idx_agents_binary_path ON agents(binary_path);

-- Projects table
CREATE TABLE projects (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  path TEXT NOT NULL UNIQUE,
  stack TEXT NOT NULL,                  -- JSON array
  status TEXT NOT NULL,                 -- planning | active | paused | done
  category TEXT NOT NULL,               -- business | personal | tool
  description TEXT,
  repo_url TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_projects_path ON projects(path);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_category ON projects(category);

-- Sessions table
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  agent_id TEXT NOT NULL,
  project_id TEXT,
  task_description TEXT,
  working_dir TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  pid INTEGER,
  start_time DATETIME NOT NULL,
  end_time DATETIME,
  exit_code INTEGER,
  output_size INTEGER DEFAULT 0,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY(agent_id) REFERENCES agents(id) ON DELETE RESTRICT,
  FOREIGN KEY(project_id) REFERENCES projects(id) ON DELETE SET NULL
);

CREATE INDEX idx_sessions_agent_id ON sessions(agent_id);
CREATE INDEX idx_sessions_project_id ON sessions(project_id);
CREATE INDEX idx_sessions_status ON sessions(status);
CREATE INDEX idx_sessions_created_at ON sessions(created_at);

-- Session logs table
CREATE TABLE session_logs (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  output_chunk TEXT NOT NULL,
  output_type TEXT NOT NULL,             -- stdout | stderr
  sequence INTEGER NOT NULL,
  timestamp DATETIME NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY(session_id) REFERENCES sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_session_logs_session_id ON session_logs(session_id);
CREATE INDEX idx_session_logs_sequence ON session_logs(session_id, sequence);

-- Workflows table (Tier 2)
CREATE TABLE workflows (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  steps TEXT NOT NULL,                  -- JSON array
  status TEXT NOT NULL DEFAULT 'draft',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_workflows_status ON workflows(status);
```

### Database Indexes

| Table | Index | Columns | Type | Rationale |
|-------|-------|---------|------|-----------|
| agents | PRIMARY | id | UNIQUE | Fast lookups by ID |
| agents | idx_agents_name | name | UNIQUE | Prevent duplicate agent names |
| agents | idx_agents_binary_path | binary_path | INDEX | Fast lookups when auto-detecting |
| projects | PRIMARY | id | UNIQUE | Fast lookups by ID |
| projects | idx_projects_path | path | UNIQUE | Ensure one project per directory |
| projects | idx_projects_status | status | INDEX | Filter by project status |
| projects | idx_projects_category | category | INDEX | Browse by category |
| sessions | PRIMARY | id | UNIQUE | Fast lookups by ID |
| sessions | idx_sessions_agent_id | agent_id | INDEX | List sessions by agent |
| sessions | idx_sessions_project_id | project_id | INDEX | List sessions by project |
| sessions | idx_sessions_status | status | INDEX | Find running/pending sessions |
| sessions | idx_sessions_created_at | created_at | INDEX | Sort by time |
| session_logs | PRIMARY | id | UNIQUE | Bulk inserts via id |
| session_logs | idx_session_logs_session_id | session_id | INDEX | Retrieve all logs for session |
| session_logs | idx_session_logs_sequence | (session_id, sequence) | INDEX | Order logs correctly |

---

## 5. API Design

### API Style

**REST** with WebSocket for real-time streaming.

- REST for CRUD operations (agents, projects, sessions)
- WebSocket for real-time session output streaming and lifecycle events

### Authentication

- **Method:** None (single-user, local-only)
- **Token Location:** N/A
- **Refresh Strategy:** N/A

### Endpoints

#### Health Check

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/health` | Server health status | No |

#### Agents

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/agents` | List all registered agents | No |
| GET | `/api/agents/:id` | Get agent details | No |
| POST | `/api/agents` | Register new agent | No |
| POST | `/api/agents/auto-detect` | Scan PATH for known agents | No |
| PATCH | `/api/agents/:id` | Update agent config | No |
| DELETE | `/api/agents/:id` | Remove agent from registry | No |

#### Sessions

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/sessions` | List all sessions (with filters) | No |
| GET | `/api/sessions/:id` | Get session details + metadata | No |
| POST | `/api/sessions` | Create and launch new session | No |
| GET | `/api/sessions/:id/output` | Get session output (tail) | No |
| POST | `/api/sessions/:id/kill` | Kill running session | No |
| WS | `/api/sessions/ws` | WebSocket for real-time output | No |

#### Projects

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/projects` | List all projects | No |
| GET | `/api/projects/:id` | Get project details | No |
| POST | `/api/projects` | Register new project | No |
| POST | `/api/projects/auto-scan` | Scan workspace for projects | No |
| PATCH | `/api/projects/:id` | Update project | No |
| DELETE | `/api/projects/:id` | Remove project | No |

#### Knowledge Base

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/knowledge` | Search knowledge base | No |
| GET | `/api/knowledge/:slug` | Get specific note | No |
| POST | `/api/knowledge` | Create/update note | No |
| DELETE | `/api/knowledge/:slug` | Delete note | No |

#### Workflows (Tier 2)

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/workflows` | List workflows | No |
| POST | `/api/workflows` | Create workflow | No |
| POST | `/api/workflows/:id/execute` | Execute workflow | No |

### Request/Response Format

#### Standard Success Response

```json
{
  "data": {
    "id": "session-123",
    "agent_id": "claude-code-1",
    "status": "running",
    "start_time": "2026-03-14T10:30:00Z"
  },
  "meta": {
    "timestamp": "2026-03-14T10:30:05Z",
    "version": "1.0"
  }
}
```

#### Standard Error Response

```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session with ID 'invalid-id' does not exist",
    "status": 404,
    "details": [
      {
        "field": "session_id",
        "message": "ID format is invalid"
      }
    ]
  }
}
```

#### Session Creation Request

```json
{
  "agent_id": "claude-code-1",
  "project_id": "project-123",
  "working_dir": "/Users/jeff/projects/business/website",
  "task_description": "Fix the login form validation error"
}
```

#### WebSocket Message Format

```typescript
// Client connects to /api/sessions/ws?session_id=session-123

// Server sends:
interface WebSocketMessage {
  type: "start" | "output" | "end" | "error";
  session_id: string;
  timestamp: string;
  data?: {
    output?: string;
    output_type?: "stdout" | "stderr";
    exit_code?: number;
    error?: string;
  };
}

// Example: {"type":"output","session_id":"s-1","timestamp":"2026-03-14T10:30:00Z","data":{"output":"Hello world\n","output_type":"stdout"}}
```

### Rate Limiting

Local-only, single-user app — no rate limiting needed. However, implement safeguards against resource exhaustion:

| Limit | Reason |
|-------|--------|
| Max 10 concurrent sessions | System resource management |
| Session output buffer: 1GB per session | Memory management |
| Workflow steps: max 20 per workflow | Complexity cap |

---

## 6. State Management

### Client State Strategy

| State Type | Solution | Example |
|------------|----------|---------|
| Server state (async) | Native fetch + React state | Session list, agent registry (fetch on mount, manual refresh) |
| UI state | Zustand store | Sidebar open/closed, active tab, modal visibility |
| Real-time state | Zustand + WebSocket | Live session output, session status updates |
| Form state | React Hook Form (optional) | Task input, workflow builder |
| URL state | Next.js router + nuqs (optional) | Filter params (?status=running&agent=claude-code) |

### Store Structure

```typescript
// UI Store - manages sidebar, modals, navigation
interface UIStore {
  // Sidebar
  sidebarOpen: boolean;
  activeSidebarTab: "agents" | "sessions" | "projects" | "knowledge";
  toggleSidebar: () => void;
  setSidebarTab: (tab: UIStore["activeSidebarTab"]) => void;

  // Modals
  modals: {
    registerAgent?: boolean;
    createSession?: boolean;
    createProject?: boolean;
  };
  openModal: (modal: keyof UIStore["modals"]) => void;
  closeModal: (modal: keyof UIStore["modals"]) => void;

  // Session detail panel
  selectedSessionId?: string;
  selectSession: (id: string) => void;

  // Toast/notifications
  notifications: Notification[];
  addNotification: (notification: Notification) => void;
  removeNotification: (id: string) => void;
}

// Session Store - manages live session state
interface SessionStore {
  // Active sessions with live updates
  activeSessions: Map<string, SessionWithLiveData>;

  // Live output per session
  sessionOutput: Map<string, string>;

  // WebSocket connection status
  wsConnected: boolean;

  // Methods
  addSession: (session: Session) => void;
  removeSession: (sessionId: string) => void;
  updateSession: (sessionId: string, data: Partial<Session>) => void;
  appendOutput: (sessionId: string, chunk: string) => void;
  clearOutput: (sessionId: string) => void;
  setWSConnected: (connected: boolean) => void;
}

// Project Store - manages project state
interface ProjectStore {
  projects: Project[];
  selectedProjectId?: string;

  setProjects: (projects: Project[]) => void;
  selectProject: (id: string) => void;
  addProject: (project: Project) => void;
  updateProject: (id: string, data: Partial<Project>) => void;
}
```

### Data Flow

```
User Action (click Launch Session)
    ↓
Form Submission → Validate with Zod
    ↓
Call Action (createSession)
    ↓
API Route (POST /api/sessions)
    ↓
Service Layer (SessionManager.launch())
    ↓
Spawn PTY + Return session ID
    ↓
Store dispatch (SessionStore.addSession)
    ↓
Component Re-render → Show session in list
    ↓
WebSocket connects to /api/sessions/ws?session_id=X
    ↓
PTY output → Server → WebSocket → Store (SessionStore.appendOutput)
    ↓
Component Re-render → Show live output in terminal
```

---

## 7. Core Services Architecture

### Layer 1: Core Types

TypeScript interfaces defining the domain model. Immutable once created, serve as the contract between layers.

**Files:** `/src/lib/core/*.types.ts`

- `agent.types.ts` — Agent, AgentConfig, AgentCapability
- `session.types.ts` — Session, SessionStatus, SessionEvent
- `project.types.ts` — Project, ProjectStatus
- `workflow.types.ts` — Workflow, WorkflowStep (Tier 2)
- `knowledge.types.ts` — KnowledgeNote, KnowledgeCategory

### Layer 2: Services

Business logic, orchestration, external integrations. Services are classes or factories that implement core functionality. Each service is responsible for one domain.

**Files:** `/src/lib/services/*.ts`

#### AgentService

```typescript
class AgentRegistry {
  // Load agents from disk (.agent-os/registry/*.json)
  loadAgents(): Promise<Agent[]>;

  // Get single agent by ID
  getAgent(id: string): Promise<Agent>;

  // Register new agent
  registerAgent(agent: Agent): Promise<Agent>;

  // Update agent config
  updateAgent(id: string, updates: Partial<Agent>): Promise<Agent>;

  // Delete agent
  deleteAgent(id: string): Promise<void>;
}

class AgentDetector {
  // Scan PATH for known agent binaries
  autoDetect(): Promise<Agent[]>;

  // Check if agent binary exists at path
  isInstalled(binaryPath: string): boolean;
}

class AgentValidator {
  // Validate agent can be launched
  validate(agent: Agent): ValidationResult;
}
```

#### SessionService

```typescript
class SessionManager {
  // Create and launch a session
  async launch(
    agentId: string,
    workingDir: string,
    taskDescription?: string,
    projectId?: string
  ): Promise<Session>;

  // Get session by ID
  async getSession(sessionId: string): Promise<SessionWithOutput>;

  // Kill running session
  async kill(sessionId: string): Promise<void>;

  // List sessions with filters
  async listSessions(filters?: {
    agent_id?: string;
    project_id?: string;
    status?: SessionStatus;
  }): Promise<Session[]>;
}

class PTYHandler {
  // Spawn PTY and manage process
  spawn(command: string, args: string[], options: SpawnOptions): PTY;

  // Write to PTY (send input)
  write(pty: PTY, data: string): void;

  // Kill process
  kill(pty: PTY): void;
}

class OutputCapture {
  // Capture stdout/stderr chunks and save to DB
  captureOutput(session: Session, pty: PTY): AsyncIterator<OutputChunk>;
}
```

#### ProjectService

```typescript
class ProjectRegistry {
  // Register new project
  registerProject(project: Project): Promise<Project>;

  // Get project by ID
  getProject(id: string): Promise<Project>;

  // List all projects
  listProjects(filters?: { status?: ProjectStatus }): Promise<Project[]>;

  // Update project
  updateProject(id: string, updates: Partial<Project>): Promise<Project>;
}

class ProjectScanner {
  // Auto-scan workspace for projects
  scan(workspaceRoot: string): Promise<Project[]>;
}
```

#### KnowledgeService

```typescript
class KnowledgeLoader {
  // Load knowledge notes from disk
  loadNotes(category?: KnowledgeCategory): Promise<KnowledgeNote[]>;

  // Get single note
  getNote(slug: string): Promise<KnowledgeNote>;
}

class KnowledgeSearch {
  // Full-text search across knowledge base
  search(query: string, category?: KnowledgeCategory): Promise<KnowledgeNote[]>;

  // Extract wikilinks from note
  extractWikilinks(content: string): string[];
}

class WikilinkResolver {
  // Resolve [[link]] references
  resolveLink(link: string): Promise<KnowledgeNote | null>;
}
```

### Layer 3: Actions

API handlers that orchestrate services. Called by API routes. Handle HTTP-level concerns (validation, error formatting, DB operations).

**Files:** `/src/lib/actions/*/*.ts`

```typescript
// Example: createSession action
export async function createSession(input: {
  agent_id: string;
  working_dir: string;
  task_description?: string;
  project_id?: string;
}): Promise<Session> {
  // 1. Validate input with Zod
  const parsed = CreateSessionSchema.parse(input);

  // 2. Verify agent exists
  const agent = await agentRegistry.getAgent(parsed.agent_id);
  if (!agent) throw new Error("Agent not found");

  // 3. Verify working directory is valid
  if (!directoryExists(parsed.working_dir)) {
    throw new Error("Working directory does not exist");
  }

  // 4. Create session in DB
  const session = await db.insert(sessions).values({
    id: generateId(),
    agent_id: parsed.agent_id,
    project_id: parsed.project_id,
    task_description: parsed.task_description,
    working_dir: parsed.working_dir,
    status: "pending",
    start_time: new Date(),
  });

  // 5. Launch session via SessionManager
  sessionManager.launch(session);

  return session;
}
```

### Layer 4: UI Components

React components that consume actions and stores. Display data and handle user interaction.

**Files:** `/src/components/features/*/*.tsx`

---

## 8. Error Handling

### Error Categories

| Category | Status Code | Handling | Example |
|----------|-------------|----------|---------|
| Validation | 400 | Show field errors in form | Invalid working directory path |
| Not Found | 404 | Show "not found" message | Session ID doesn't exist |
| Conflict | 409 | Show conflict error | Agent name already registered |
| Agent Launch Error | 500 | Show error with logs | Binary path not found, permission denied |
| System Error | 500 | Show generic error, log details | DB corruption, out of disk space |

### Custom Error Classes

```typescript
// Base error class
class AppError extends Error {
  constructor(
    public code: string,
    public message: string,
    public statusCode: number = 500,
    public details?: Record<string, any>
  ) {
    super(message);
    this.name = "AppError";
  }
}

// Specific errors
class ValidationError extends AppError {
  constructor(message: string, details?: Record<string, any>) {
    super("VALIDATION_ERROR", message, 400, details);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super("NOT_FOUND", `${resource} with ID '${id}' not found`, 404);
  }
}

class AgentLaunchError extends AppError {
  constructor(agent_id: string, reason: string) {
    super("AGENT_LAUNCH_ERROR", `Failed to launch agent: ${reason}`, 500, { agent_id });
  }
}
```

### API Error Response Format

```typescript
// Global error handler in API routes
export async function handleError(error: unknown): Promise<Response> {
  const appError = error instanceof AppError
    ? error
    : new AppError("INTERNAL_ERROR", "An unexpected error occurred", 500);

  return Response.json(
    {
      error: {
        code: appError.code,
        message: appError.message,
        status: appError.statusCode,
        details: appError.details,
      },
    },
    { status: appError.statusCode }
  );
}
```

### Client Error Handling

```typescript
// Hook for API calls with error handling
export function useAsync<T>(
  asyncFn: () => Promise<T>,
  onError?: (error: AppError) => void
) {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<AppError | null>(null);
  const [loading, setLoading] = useState(false);

  const execute = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      const result = await asyncFn();
      setData(result);
    } catch (err) {
      const appError = normalizeError(err);
      setError(appError);
      onError?.(appError);
    } finally {
      setLoading(false);
    }
  }, [asyncFn, onError]);

  return { data, error, loading, execute };
}
```

### Logging Strategy

```typescript
// Development: colorized console output
// Production: structured JSON logs

class Logger {
  debug(message: string, data?: any) {
    if (process.env.NODE_ENV === "development") {
      console.log(`[DEBUG] ${message}`, data);
    }
  }

  info(message: string, data?: any) {
    console.log(`[INFO] ${message}`, data);
  }

  error(message: string, error?: any) {
    console.error(`[ERROR] ${message}`, error);
    // Could pipe to external service in production
  }
}
```

---

## 9. Testing Strategy

### Testing Pyramid

| Level | Tools | Coverage Target | Examples |
|-------|-------|-----------------|----------|
| Unit | Bun test | 80% | Utility functions, validators, stores |
| Integration | Bun test | Key paths | API routes, database queries |
| E2E | Playwright | Critical flows | Launch session, view output |

### Test File Structure

```
src/lib/utils/format-date.ts
src/lib/utils/format-date.test.ts  # Co-located

tests/unit/
  └── services/
      └── session-manager.test.ts

tests/integration/
  └── api/
      └── sessions.test.ts

tests/e2e/
  └── workflows/
      └── launch-session.spec.ts
```

### What to Test

**Unit tests (80% coverage):**
- Utility functions (`formatDate`, `parseAnsi`, `slugify`)
- Validators (`AgentValidator`, `ProjectValidator`)
- Zustand stores (state updates, selectors)
- Data transformations

**Integration tests:**
- API routes (request/response format, error handling)
- Service layer (SessionManager.launch, AgentRegistry.load)
- Database operations (insert, update, query filters)

**E2E tests (critical paths):**
- Register an agent → Launch session → See output
- Create project → Quick-launch agent from project
- Search knowledge base → Navigate to note

### Example Test

```typescript
import { test, expect } from "bun:test";
import { formatDate } from "@/lib/utils/format-date";

test("formatDate formats ISO strings correctly", () => {
  const date = new Date("2026-03-14T10:30:00Z");
  expect(formatDate(date)).toBe("Mar 14, 2026");
});

test("SessionManager launches agent with correct working directory", async () => {
  const manager = new SessionManager(mockPTYHandler, mockDB);
  const session = await manager.launch("agent-1", "/home/user/project");

  expect(session.working_dir).toBe("/home/user/project");
  expect(session.status).toBe("pending");
  expect(mockPTYHandler.spawn).toHaveBeenCalledWith(
    expect.objectContaining({ cwd: "/home/user/project" })
  );
});
```

---

## 10. Performance Considerations

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Dashboard load | < 1s | Time to interactive (local only) |
| Agent launch | < 2s | From click to PTY active |
| Live output latency | < 100ms | PTY → Server → Client |
| Support concurrent sessions | 10+ | No degradation |

### Optimization Strategies

#### Code Splitting

- Route-based: Load dashboard features only when navigated to
- Component-based: Lazy-load terminal view only when viewing session
- Dynamic imports for heavy libraries (xterm.js on terminal view)

```typescript
import dynamic from "next/dynamic";

const TerminalView = dynamic(() => import("@/components/features/session-monitor/TerminalView"), {
  loading: () => <Skeleton />,
});
```

#### Database Optimization

- **Indexes:** On agent_id, project_id, status, created_at for fast queries
- **WAL mode:** Enables concurrent reads while writes are in progress
- **Query optimization:** Use SELECT columns, avoid N+1 queries
- **Session log archival:** Truncate old logs or archive to separate table

#### WebSocket Optimization

- Batch output chunks: Send 10 chunks at a time instead of one per event
- Compression: Use message compression for large outputs (not critical for local)
- Connection pooling: Reuse single WebSocket for multiple sessions

#### Frontend Optimization

- **Terminal scrollback buffer:** Limit to last 1000 lines in xterm.js to prevent memory leak
- **Virtual scrolling:** For session list if > 100 sessions (rare)
- **Debounce WebSocket updates:** Don't re-render on every character

```typescript
// Example: Batch WebSocket updates
const sessionOutputBuffer: Map<string, string[]> = new Map();

function addOutput(sessionId: string, chunk: string) {
  if (!sessionOutputBuffer.has(sessionId)) {
    sessionOutputBuffer.set(sessionId, []);
  }
  sessionOutputBuffer.get(sessionId)!.push(chunk);

  // Flush every 100ms
  if (sessionOutputBuffer.get(sessionId)!.length > 10) {
    flushOutput(sessionId);
  }
}
```

---

## 11. Security Considerations

### Security Checklist

- [x] Input validation on all endpoints (Zod schemas)
- [x] No SQL injection (parameterized queries via Drizzle ORM)
- [x] XSS prevention (React auto-escaping, sanitize ANSI codes)
- [x] No CSRF (local-only, no state-changing GET requests)
- [x] Secrets in environment variables (none needed for local dev)
- [x] Agent binary path validation (whitelist known agents, verify path exists)
- [x] No remote code execution (only registered agents can launch)
- [x] Session logs stored locally only (no cloud upload)

### Sensitive Data Handling

- **Agent configuration:** Stored in SQLite, no passwords/tokens exposed
- **Terminal output:** Stored locally, may contain sensitive info — advise user to scrub logs
- **Working directories:** User can see all paths — intentional (dashboard for single user)
- **Knowledge base:** Obsidian-compatible, stored locally, can contain PII

### Agent Launch Security

```typescript
// Only allow launching agents from whitelist + custom registered agents
const KNOWN_AGENTS = new Set([
  "claude",
  "gemini",
  "codex",
  "opencode",
  "qwen",
  "kumi",
  "quine",
  "rootcode",
]);

async function launchAgent(agent: Agent, workingDir: string): Promise<Session> {
  // 1. Verify agent is registered in DB
  const registeredAgent = await agentRegistry.getAgent(agent.id);
  if (!registeredAgent) throw new Error("Agent not registered");

  // 2. Verify binary path exists and is executable
  if (!fs.existsSync(registeredAgent.binary_path)) {
    throw new Error("Agent binary not found");
  }

  // 3. Verify working directory exists
  if (!fs.existsSync(workingDir)) {
    throw new Error("Working directory does not exist");
  }

  // 4. Verify working directory is within workspace (prevent escape)
  const workspaceRoot = process.env.WORKSPACE_ROOT || "/Users/jeff/Jeffs Agent OS";
  const resolved = path.resolve(workingDir);
  const workspaceResolved = path.resolve(workspaceRoot);

  if (!resolved.startsWith(workspaceResolved)) {
    throw new Error("Working directory outside workspace");
  }

  // 5. Launch with restricted environment
  return ptyHandler.spawn(registeredAgent.binary_path, [], {
    cwd: workingDir,
    env: {
      ...process.env,
      // Remove sensitive env vars
      AWS_ACCESS_KEY_ID: undefined,
      AWS_SECRET_ACCESS_KEY: undefined,
      // Keep user's PATH
      PATH: process.env.PATH,
    },
    timeout: registeredAgent.config.timeout || 300000,
  });
}
```

### XSS Prevention in Terminal Output

```typescript
// Sanitize ANSI codes when rendering to avoid injection
import stripAnsi from "strip-ansi";

function sanitizeTerminalOutput(output: string): string {
  // Strip non-printable characters but keep ANSI color codes
  // xterm.js handles ANSI codes safely
  return output.replace(/[\x00-\x08\x0B-\x0C\x0E-\x1F\x7F]/g, "");
}

// React will auto-escape text content
<pre>{sanitizeTerminalOutput(output)}</pre>
```

---

## 12. Monitoring & Observability

### Monitoring Strategy

For single-user local app, focus on:

1. **Error tracking** — Capture unexpected errors
2. **Session lifecycle** — Log session start/end/errors
3. **Performance** — Track API response times
4. **Health checks** — Periodic checks that services work

### Logging

```typescript
// Log all session lifecycle events
logger.info("Session started", {
  session_id: session.id,
  agent_id: session.agent_id,
  working_dir: session.working_dir,
  timestamp: new Date().toISOString(),
});

logger.info("Session completed", {
  session_id: session.id,
  exit_code: session.exit_code,
  duration_ms: Date.now() - session.start_time.getTime(),
  output_size: session.output_size,
});

logger.error("Session failed to launch", {
  session_id: session.id,
  agent_id: session.agent_id,
  error: error.message,
});
```

### Health Check

```typescript
// GET /api/health
export async function GET() {
  try {
    // Check database connectivity
    await db.select().from(agents).limit(1);

    // Check workspace directories exist
    const dirs = [".agent-os", "knowledge", "projects"];
    for (const dir of dirs) {
      if (!fs.existsSync(path.join(process.env.WORKSPACE_ROOT || "", dir))) {
        throw new Error(`Directory ${dir} not found`);
      }
    }

    return Response.json({
      status: "healthy",
      timestamp: new Date().toISOString(),
      checks: {
        database: "ok",
        workspace: "ok",
      },
    });
  } catch (error) {
    return Response.json(
      {
        status: "unhealthy",
        error: error.message,
      },
      { status: 503 }
    );
  }
}
```

---

## 13. Implementation Phases

### Phase 1: Foundation (Week 1)

**Goal:** Basic dashboard with agent registry and workspace visibility.

- [x] Next.js app scaffolded with TypeScript + Tailwind
- [x] SQLite database with schema (agents, projects, sessions tables)
- [x] Agent registry: auto-detect + manual register
- [x] Workspace tree view (read-only)
- [x] Dashboard home showing agent list and quick links

**Technical:**
- Drizzle ORM migrations
- Zustand stores for UI state
- API routes: GET /api/agents, POST /api/agents/auto-detect
- Components: AgentList, WorkspaceTree, DashboardHome

### Phase 2: Core Agent Control (Week 2-3)

**Goal:** Launch agents and see live output.

- [ ] Session launcher (click agent → launch with task)
- [ ] PTY-based agent spawning with process management
- [ ] WebSocket server for real-time output streaming
- [ ] Terminal view with xterm.js
- [ ] Session monitor with live status updates
- [ ] Session history and output logging

**Technical:**
- node-pty integration
- WebSocket server alongside Next.js
- SessionManager service class
- API routes: POST /api/sessions, WS /api/sessions/ws
- Components: SessionLauncher, TerminalView, SessionList

### Phase 3: Project Registry (Week 3)

**Goal:** Track projects and scope sessions to projects.

- [ ] Project registry (manual + auto-scan)
- [ ] Project detail view with session history
- [ ] Quick-launch agent for a project
- [ ] Project filtering in session list

**Technical:**
- ProjectScanner service
- ProjectRegistry service
- API routes: GET/POST /api/projects, POST /api/projects/auto-scan
- Components: ProjectList, ProjectDetail, ProjectQuickLaunch

### Phase 4: Knowledge Base (Week 4, Tier 2)

**Goal:** Search and navigate knowledge base.

- [ ] Load knowledge notes from markdown files
- [ ] Full-text search across knowledge base
- [ ] Note viewer with wikilink rendering
- [ ] Basic note creation UI

**Technical:**
- KnowledgeLoader and KnowledgeSearch services
- WikilinkResolver
- API routes: GET /api/knowledge, GET /api/knowledge/:slug
- Components: KnowledgeSearch, NoteViewer, NoteEditor

### Phase 5: Task Router & Workflows (Week 5+, Tier 2)

**Goal:** Multi-step agent orchestration.

- [ ] Task input form with agent selection
- [ ] Task router (match task to agent via keywords)
- [ ] Workflow builder (define multi-step chains)
- [ ] Workflow execution with output forwarding

**Technical:**
- TaskDispatcher service
- WorkflowBuilder and WorkflowExecutor services
- Workflow table in SQLite
- API routes: POST /api/workflows, POST /api/workflows/:id/execute
- Components: TaskForm, WorkflowBuilder, WorkflowExecutor

---

## 14. Development Environment Setup

### Prerequisites

- macOS (primary target)
- Bun installed (`curl -fsSL https://bun.sh/install | bash`)
- Node.js 18+ (for compatibility with some packages)
- SQLite 3

### Local Development

```bash
# Install dependencies
bun install

# Initialize database
bun run scripts/db-init.ts

# Start development server
bun run dev
# Opens http://localhost:3000

# Run tests
bun test

# Format code
bun run format

# Lint
bun run lint
```

### Environment Variables

Create `.env.local`:

```
WORKSPACE_ROOT=/Users/jeff/Jeffs\ Agent\ OS
DATABASE_URL=file:./local.db
WS_PORT=3001
NODE_ENV=development
```

---

## 15. Open Technical Questions

- [ ] Should WebSocket server run in same process as Next.js or separate?
- [ ] How to handle agents that don't support stdin (interactive-only)?
- [ ] Best approach for handling long-running sessions (> 1 hour)?
- [ ] Should session logs be searchable full-text or just filterable?
- [ ] How to prevent terminal buffer from growing unbounded (max size per session)?
- [ ] Should knowledge base support version history / backups?
- [ ] How to detect Cowork sessions (process monitoring vs file watching)?

---

## 16. Implementation Notes for Developers

### Key Architectural Principles

1. **Separation of Concerns:** Services don't know about UI. Actions orchestrate services. UI calls actions.
2. **Type Safety:** Every layer has TypeScript types. Zod for runtime validation at API boundaries.
3. **Local-First:** All data stored locally. No cloud dependencies for core features.
4. **Fail Gracefully:** Errors are caught and returned to user with helpful messages.
5. **Performance:** Keep real-time latency under 100ms. Limit concurrent sessions to 10.

### Code Style

- Use TypeScript strictly (strict mode enabled)
- Prefer functional components with hooks
- Keep components small and focused
- Use Zustand for state, not Redux
- API responses follow standard format (data/meta/error)
- Error messages should be user-friendly, not technical

### Before Starting Development

1. Read this Architecture Document in full
2. Review the TypeScript types in `/src/lib/core/`
3. Understand the four-layer pattern (Types → Services → Actions → UI)
4. Set up local dev environment and verify it runs
5. Check open questions — discuss any blockers with Jeff

---

**Document Status:** This is a living document. Update it as architecture decisions change during implementation.
