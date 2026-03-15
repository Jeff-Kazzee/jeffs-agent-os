# Gemini CLI Instructions

This project uses a universal agent onboarding document. Read these files:

1. **`AGENTS.md`** — Task board, coordination protocol, quality gates, file map
2. **`MISSION.md`** — Phase 1 MVP scope, execution order, data models, design system
3. **`CLAUDE.md`** — Code style and architecture conventions (applies to ALL agents, not just Claude)
4. **`project-status.md`** — Current state, what's done, what's in progress

## Quick Reference

- **Runtime:** Bun (not npm/pnpm/yarn)
- **Framework:** Next.js 15 App Router
- **Port:** 4000
- **Database:** SQLite via better-sqlite3 + Drizzle ORM
- **OS:** Windows 11 — use `where` instead of `which` for binary detection
- **Available tasks:** See AGENTS.md Section 4
- **Coordination:** See AGENTS.md Section 5
