# Jeff's Agent OS

## Project Overview

A local-first web dashboard that gives a solo indie developer unified control over every AI coding agent on their machine — Claude Code, Gemini CLI, Codex CLI, Open Code, and more. Combines agent orchestration, project planning, knowledge management, and workspace organization into a single operating environment.

**Tech Stack:** Next.js 15, TypeScript, Tailwind v4, Zustand, SQLite (Drizzle ORM), Bun, xterm.js, node-pty, WebSockets
**Status:** Planning
**Owner:** Jeff Kazzee (@jeffkazzee, The Little AI Company, Salmon, Idaho)

## Development Philosophy

This project uses **vibe coding** with AI assistants. Key principles:

1. **Black Box Driven Development (BBDD)** — Contract-first, PIV loop (Plan-Implement-Verify), four-layer architecture
2. **Four-layer architecture:** Core Types → Services → Actions → UI (sealed modules)
3. **Comprehensive documentation before building** — PRD, Design, Architecture docs in `/docs`
4. **Simplify relentlessly** — Fewer files, fewer abstractions, fewer dependencies
5. **Test-driven for features** — Write the test first for Services and Actions layers
6. **Conventional commits** — Use `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`

## Architecture

```
dashboard/
├── src/
│   ├── app/                  # Next.js App Router pages
│   │   ├── api/              # API routes (REST + WebSocket)
│   │   ├── (dashboard)/      # Dashboard route group
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── core/                 # Core types (layer 1)
│   │   └── types/
│   ├── services/             # Business logic (layer 2)
│   │   ├── agent-registry/
│   │   ├── session-manager/
│   │   ├── project-registry/
│   │   └── knowledge-base/
│   ├── actions/              # Server actions / API handlers (layer 3)
│   ├── components/           # UI components (layer 4)
│   │   ├── ui/               # Primitives (Button, Input, etc.)
│   │   └── features/         # Feature components (SessionCard, AgentPanel, etc.)
│   ├── hooks/
│   ├── stores/               # Zustand stores
│   └── lib/
│       ├── db/               # Drizzle schema + migrations
│       └── utils/
├── tests/
├── public/
└── package.json
```

## Key Files

| File | Purpose | When to Update |
|------|---------|----------------|
| `project-status.md` | Current project state | Every session |
| `CHANGELOG.md` | Version history | When features/fixes complete |
| `docs/PRD.md` | Product requirements | When scope changes |
| `docs/DESIGN.md` | UI/UX decisions | When design changes |
| `docs/ARCHITECTURE.md` | Technical decisions | When architecture changes |

## Workspace Structure

This project lives inside a larger workspace:

```
Jeffs Agent OS/
├── .agent-os/          # Agent OS internal state (registry, sessions, logs)
├── business/           # Business ops (plans, finances, brand, social)
├── projects/           # All coding projects (personal, business, tools)
├── knowledge/          # Obsidian-compatible knowledge base
├── planning/           # Project planning hub (active, backlog, archive)
├── dashboard/          # THIS APP — the web dashboard
└── docs/               # PRD, Design, Architecture docs
```

## Code Style

- **Files:** kebab-case (`agent-registry.ts`)
- **Components:** PascalCase (`SessionCard.tsx`)
- **Hooks:** camelCase with `use` prefix (`useAgentStore.ts`)
- **Constants:** SCREAMING_SNAKE_CASE
- **Types:** PascalCase, exported from `core/types/`

## Key Decisions

- **No auth** — single user, local only, localhost
- **SQLite** — zero config, WAL mode, perfect for local-first
- **Agent adapter pattern** — each CLI agent gets a normalized adapter interface
- **WebSocket** for real-time session output streaming
- **node-pty** for terminal emulation (preserves colors, signals)
- **Dark mode default** — power user tool, terminal aesthetic

## Constraints

- Jeff has epilepsy — energy management is medical, not preference. Design for short, high-impact sessions.
- No cloud dependencies for core features
- Must coexist with dev servers on other ports
- Dashboard port: 4000 (to avoid conflicts with 3000)

## Environment Variables

```
DATABASE_URL=./data/agent-os.db
PORT=4000
```

## Commands

```bash
bun dev           # Start dashboard dev server
bun build         # Production build
bun test          # Run tests
bun db:migrate    # Run database migrations
```
