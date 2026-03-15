# Gemini CLI Instructions

Read these files before working on this project:

1. **`AGENTS.md`** — Setup commands, dev workflow, testing, code style, architecture rules
2. **`MISSION.md`** — Phase 1 MVP scope, execution order, data models, design system
3. **`CLAUDE.md`** — Project conventions and dev philosophy (applies to ALL agents)
4. **`project-status.md`** — Current state, what's done, what's in progress
5. **`planning/active/phase-1-taskboard.md`** — Task board with dependency graph, sprints, acceptance criteria

## Quick Reference

- **Runtime:** Bun (not npm/pnpm/yarn)
- **Framework:** Next.js 15 App Router
- **Port:** 4000
- **Database:** SQLite via better-sqlite3 + Drizzle ORM
- **OS:** Windows 11 — use `where` instead of `which` for binary detection
- **Available tasks:** See `planning/active/phase-1-taskboard.md`
- **Coordination:** See task board coordination protocol section
