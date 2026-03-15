# Jeff's Agent OS - Product Requirements Document

**Version:** 0.1.0
**Last Updated:** 2026-03-14
**Author:** Jeff Kazzee + Claude
**Status:** Draft

---

## 1. Executive Summary

Jeff's Agent OS is a local-first web dashboard and workspace system that gives a solo indie developer unified control over every AI coding agent on their machine — Claude Code, Gemini CLI, Codex CLI, Open Code, Quen Code, Kumi CLI, Quine, Root Code, and any future CLI agent. It combines agent orchestration, project planning, knowledge management, and workspace organization into a single operating environment.

**One-liner:** A local command center that lets one developer orchestrate multiple AI coding agents, plan projects, and manage context from a single dashboard.

---

## 2. Problem Statement

### The Problem

Solo developers using multiple AI coding agents have no unified way to:
- See what all their agents are doing at a glance
- Route tasks to the right agent based on model strengths
- Chain agents together in multi-step workflows
- Keep project context, business plans, and personal knowledge organized in one place
- Maintain persistent context that survives across sessions and agents

Each agent is an isolated terminal session. Context gets lost between sessions. There's no memory of what was done where, by which agent, or why.

### Current Solutions & Their Limitations

- **Terminal multiplexers (tmux/zellij):** Can view multiple agents but can't control or orchestrate them. No awareness of what each agent is doing.
- **Cursor/IDE integration:** Locked to one agent at a time. No multi-agent orchestration.
- **Claude Desktop Cowork:** Great for single-agent sessions but no cross-agent visibility.
- **Manual task switching:** Developer becomes the router, manually pasting context between agents. Does not scale.
- **Scattered files:** Business plans in Notion, knowledge in random markdown files, project context in CLAUDE.md files per repo. No unified knowledge base.

### Why Now?

- CLI agent ecosystem has exploded (10+ viable agents available as of early 2026)
- Each agent has different strengths (Claude for architecture, Gemini for breadth, Codex for speed)
- Jeff is already running 4+ agents daily and losing context/time to manual coordination
- The workspace folder (Jeffs Agent OS) already exists — it needs structure and a control plane

---

## 3. Goals & Success Metrics

### Primary Goal

Reduce Jeff's cognitive overhead of managing multiple AI agents by 50%+ — measured by time spent switching context, re-explaining tasks, and manually routing work.

### Success Metrics

| Metric | Target | Timeframe |
|--------|--------|-----------|
| Agents registered & launchable from dashboard | 6+ | MVP launch |
| Tasks successfully routed to agents | 10/day | First week |
| Time to find project context | < 10 seconds | First week |
| Agent sessions logged & searchable | 100% | Ongoing |
| Daily active use by Jeff | Every coding day | First month |

### Non-Goals (Explicitly Out of Scope)

- Building a SaaS product (this is Jeff's personal tool first)
- Mobile app (desktop-only, local-first)
- Cloud sync or multi-device (single machine)
- Replacing any individual agent (we orchestrate, not replace)
- Building custom AI models
- IDE plugin (dashboard is the interface, not an IDE extension)

---

## 4. User Personas

### Primary Persona: Jeff

- **Who:** 34yo solo indie dev, self-taught fullstack TypeScript, 2 years daily coding, epileptic (energy = medical constraint)
- **Goals:** Ship products faster by leveraging multiple AI agents without cognitive overload. Keep business, personal, and project context organized.
- **Pain Points:** Context lost between agent sessions. Manual routing of tasks. Scattered business/personal knowledge. No visibility into what agents are doing across repos.
- **Tech Comfort:** Very high — runs 4+ CLI agents daily, builds dev tools
- **Quote:** "I shouldn't have to be the goddamn router between my own agents."

### Secondary Persona: Indie Hacker (Future)

- **Who:** Solo builder running 2-3 AI agents, wants a personal dev OS
- **Goals:** Same as Jeff but may have different agent preferences
- **Note:** This persona is Tier 4 — we're building for Jeff first, generalizing later

---

## 5. User Stories & Requirements

### Epic 1: Agent Registry & Lifecycle

#### User Story 1.1: Register an Agent
**As** Jeff
**I want to** register a CLI agent (e.g., Claude Code, Gemini CLI) with its binary path, model info, and capabilities
**So that** the system knows what agents are available and how to launch them

**Acceptance Criteria:**
- [ ] Can add an agent with: name, binary path, default model, description, capabilities tags
- [ ] Auto-detect installed agents by scanning PATH for known binaries (claude, gemini, codex, opencode, etc.)
- [ ] Agent config stored in `.agent-os/registry/` as JSON files
- [ ] Dashboard shows all registered agents with status (installed/not found)

#### User Story 1.2: Launch an Agent Session
**As** Jeff
**I want to** launch any registered agent from the dashboard with optional context (working directory, task description, files)
**So that** I don't have to open a terminal and manually navigate

**Acceptance Criteria:**
- [ ] Click-to-launch any registered agent
- [ ] Specify working directory (file picker or type path)
- [ ] Optional: attach task description / prompt
- [ ] Session opens in an embedded terminal or new terminal window
- [ ] Session ID generated and tracked

#### User Story 1.3: Monitor Running Agents
**As** Jeff
**I want to** see all currently running agent sessions with their status, output, and working directory
**So that** I know what's happening across all my agents at a glance

**Acceptance Criteria:**
- [ ] Dashboard shows active sessions with: agent name, working dir, start time, status
- [ ] Live output stream (tail of stdout/stderr) visible per session
- [ ] Can expand/collapse session output
- [ ] Session list auto-updates

---

### Epic 2: Task Orchestration

#### User Story 2.1: Route a Task to an Agent
**As** Jeff
**I want to** describe a task and assign it to a specific agent (or let the system recommend one)
**So that** I can dispatch work without context-switching

**Acceptance Criteria:**
- [ ] Task input: description, target repo/directory, priority, estimated complexity
- [ ] Manual agent selection from dropdown
- [ ] System suggests best agent based on task type and agent capabilities
- [ ] Task creates a session and launches the agent with the task prompt

#### User Story 2.2: Chain Multiple Agents
**As** Jeff
**I want to** create multi-step workflows where output from one agent feeds into another
**So that** I can build complex pipelines (e.g., Claude architects → Codex implements → Gemini reviews)

**Acceptance Criteria:**
- [ ] Workflow builder: define steps with agent + prompt per step
- [ ] Output from step N available as context for step N+1
- [ ] Can pause between steps for human review
- [ ] Workflow saved as reusable template
- [ ] Execution log for the entire workflow

#### User Story 2.3: Parallel Execution
**As** Jeff
**I want to** run multiple agents in parallel on different tasks
**So that** I can maximize throughput during high-energy windows

**Acceptance Criteria:**
- [ ] Launch multiple sessions simultaneously
- [ ] Dashboard shows all parallel sessions
- [ ] Resource awareness (warn if system is overloaded)
- [ ] Can kill/pause individual sessions

---

### Epic 3: Workspace & Project Management

#### User Story 3.1: Project Registry
**As** Jeff
**I want to** register projects with their repo path, stack info, and current status
**So that** the dashboard knows about all my active work

**Acceptance Criteria:**
- [ ] Add project: name, path, stack, status (planning/active/paused/done), category (business/personal/tool)
- [ ] Auto-detect projects by scanning known directories for package.json / CLAUDE.md
- [ ] Project detail view shows: recent git activity, open issues, agent sessions
- [ ] Quick-launch agent scoped to a project's directory

#### User Story 3.2: Organized Workspace
**As** Jeff
**I want to** have a clear folder hierarchy separating business, personal, and knowledge
**So that** everything has a place and context doesn't bleed

**Acceptance Criteria:**
- [ ] Workspace structure created and documented (see Workspace Hierarchy below)
- [ ] Dashboard shows workspace tree with quick navigation
- [ ] Can create new projects from templates into the correct category folder
- [ ] Folder conventions enforced (README in each project, CLAUDE.md in each repo)

---

### Epic 4: Knowledge Base

#### User Story 4.1: Obsidian-Compatible Knowledge Vault
**As** Jeff
**I want to** maintain a knowledge base with context about me, the business, tech learnings, agent notes, and social media activity
**So that** I (and my agents) can always find relevant context

**Acceptance Criteria:**
- [ ] Knowledge base is Obsidian-compatible markdown files in `knowledge/` folder
- [ ] Organized by topic: me, business, tech, agents, social, templates
- [ ] Supports wikilinks (`[[note-name]]`) for cross-referencing
- [ ] Dashboard has search across all knowledge base files
- [ ] Can create/edit notes from dashboard

#### User Story 4.2: Agent Context Injection
**As** Jeff
**I want to** automatically inject relevant knowledge base content into agent sessions
**So that** agents have context without me manually pasting it

**Acceptance Criteria:**
- [ ] When launching an agent, can select knowledge notes to inject as context
- [ ] Auto-suggest relevant notes based on project/task keywords
- [ ] Context injected as system prompt prefix or file attachment (agent-dependent)
- [ ] Track which context was injected per session

#### User Story 4.3: Session Knowledge Capture
**As** Jeff
**I want to** capture learnings, decisions, and important outputs from agent sessions back into the knowledge base
**So that** knowledge grows over time and doesn't get lost in terminal scrollback

**Acceptance Criteria:**
- [ ] One-click save session output (or selection) to knowledge base
- [ ] Auto-tag captured content with: agent, project, date, topic
- [ ] Captured content searchable from dashboard
- [ ] Option to auto-capture session summaries on session end

---

### Epic 5: Cowork / Claude Desktop Integration

#### User Story 5.1: Cowork Session Awareness
**As** Jeff
**I want to** see when I'm running a Claude Desktop Cowork session and what it's working on
**So that** the Agent OS has full visibility even for GUI-based agent work

**Acceptance Criteria:**
- [ ] Detect active Cowork sessions (monitor Claude Desktop process)
- [ ] Show Cowork session in the active sessions panel
- [ ] Display working directory if available
- [ ] Log Cowork session start/end times

#### User Story 5.2: Workspace Analysis from Cowork
**As** Jeff
**I want to** use Cowork sessions to analyze and work within the Agent OS workspace
**So that** Cowork and the Agent OS are complementary, not competing

**Acceptance Criteria:**
- [ ] Agent OS workspace is the Cowork-selected folder
- [ ] Cowork can read/write to all workspace areas
- [ ] Agent OS dashboard shows when Cowork is active in the workspace

---

## 6. Feature Specifications

### Tier 1: Essential (MVP Must-Haves)

#### Feature 1.1: Agent Registry
- **Description:** JSON-based registry of CLI agents with auto-detection and manual config
- **Technical Notes:** Scan PATH for known agent binaries. Store in `.agent-os/registry/`. Schema defined in TypeScript types.
- **Priority:** P0

#### Feature 1.2: Agent Launcher
- **Description:** Launch any registered agent from the dashboard with working dir and optional prompt
- **Technical Notes:** Use `child_process.spawn` / Bun shell to launch agents. Capture stdout/stderr via PTY. Generate session IDs.
- **Priority:** P0

#### Feature 1.3: Session Monitor
- **Description:** Real-time view of all active agent sessions with live output streaming
- **Technical Notes:** PTY-based output capture. WebSocket from backend to dashboard for live updates. Session state in SQLite.
- **Priority:** P0

#### Feature 1.4: Dashboard Home
- **Description:** Main dashboard showing: active sessions, registered agents, recent activity, quick actions
- **Technical Notes:** Next.js app with Tailwind. Local-only (localhost). No auth needed (single user).
- **Priority:** P0

#### Feature 1.5: Workspace Tree
- **Description:** Navigable view of the workspace folder hierarchy with project quick-launch
- **Technical Notes:** Read filesystem, display as tree. Click project to see details or launch agent.
- **Priority:** P0

#### Feature 1.6: Project Registry
- **Description:** Register and track projects with status, category, and agent session history
- **Technical Notes:** SQLite table for projects. Auto-scan for package.json in workspace dirs.
- **Priority:** P0

---

### Tier 2: Valuable (Post-MVP)

#### Feature 2.1: Task Router
- **Description:** Task dispatch system — describe task, pick or auto-select agent, launch with context
- **Rationale:** Valuable but MVP works with manual agent selection
- **Priority:** P1

#### Feature 2.2: Workflow Builder
- **Description:** Multi-step agent chains with output forwarding and pause points
- **Rationale:** Full orchestration is the vision but complex to build. MVP is single-agent dispatch.
- **Priority:** P1

#### Feature 2.3: Knowledge Base UI
- **Description:** Full CRUD for knowledge base notes with search and wikilink support
- **Rationale:** Can use Obsidian directly for MVP. Dashboard integration adds convenience.
- **Priority:** P1

#### Feature 2.4: Context Injection Engine
- **Description:** Auto-inject relevant knowledge into agent sessions based on project/task
- **Rationale:** Powerful but requires semantic search / keyword matching. Manual injection works for MVP.
- **Priority:** P1

#### Feature 2.5: Session Log & Search
- **Description:** Persistent session history with full-text search over agent output
- **Rationale:** Critical for long-term value but not blocking MVP launch.
- **Priority:** P1

#### Feature 2.6: Parallel Execution Manager
- **Description:** Launch and manage multiple concurrent agent sessions with resource awareness
- **Rationale:** Jeff already runs agents in parallel manually. System awareness adds safety but isn't blocking.
- **Priority:** P1

---

### Tier 3: Nice-to-Have (Future)

- **Agent Performance Analytics:** Track token usage, task completion rates, time per task by agent
- **Smart Agent Recommendation:** ML-based task routing using historical success data
- **Git Integration:** Show git status, branches, recent commits per project in dashboard
- **Calendar/Energy Integration:** Suggest heavy vs light tasks based on time of day and Jeff's energy patterns
- **Social Media Tracker:** Log social media posts, engagement, content ideas in knowledge base
- **Template System:** Project templates with pre-configured agent assignments

---

### Tier 4: Out of Scope (Not Building)

- **SaaS/multi-user:** This is Jeff's personal tool. No auth, no multi-tenancy.
- **Cloud hosting:** Runs locally only. No cloud infrastructure.
- **IDE integration:** Dashboard is the interface. Not building VS Code / Cursor extensions.
- **Custom LLM hosting:** We orchestrate existing agents, not host models.
- **Mobile app:** Desktop browser only.
- **Agent development:** We don't build agents, we control existing ones.

---

## 7. Technical Constraints

### Platform Requirements
- Must run on macOS (Jeff's primary machine)
- Local-only, no external network dependencies for core features
- Single-user, no authentication required
- Must work alongside existing development workflow (no port conflicts with dev servers)

### Technology Constraints
- **Runtime:** Bun (Jeff's preferred runtime)
- **Framework:** Next.js 15+ with App Router (Jeff's primary stack)
- **Styling:** Tailwind CSS v4+
- **State:** Zustand for client state
- **Database:** SQLite via better-sqlite3 or Drizzle ORM (local-first, zero config)
- **Terminal:** xterm.js for embedded terminal rendering
- **IPC:** WebSockets for real-time session updates

### Performance Requirements
- Dashboard loads in < 1 second (it's local)
- Agent launch < 2 seconds from click
- Live output stream latency < 100ms
- Support 10+ concurrent sessions without degradation

### Security Requirements
- No secrets in codebase
- Agent binary paths validated before execution
- No remote code execution — only registered agents can be launched
- Session logs stored locally only

---

## 8. Dependencies & Integrations

### External Dependencies

| Dependency | Purpose | Status | Risk |
|------------|---------|--------|------|
| Claude Code CLI | Primary agent | Installed | Low |
| Gemini CLI | Secondary agent | Installed | Low |
| Codex CLI | Speed-focused agent | Installed | Low |
| Open Code | Alternative agent | Installed | Low |
| Quen Code / Kumi / Quine / Root Code | Additional agents | Varies | Low (optional) |
| Claude Desktop | Cowork integration | Installed | Medium (process detection) |
| Obsidian | Knowledge base viewer | Optional | Low (markdown-compatible) |
| node-pty | PTY for terminal capture | npm package | Low |
| xterm.js | Terminal rendering in browser | npm package | Low |

### Internal Dependencies
- Workspace folder structure must exist before dashboard can navigate it
- Agent registry must be populated before any orchestration features work
- SQLite database must be initialized on first run

---

## 9. Timeline & Milestones

### Phase 1: Foundation (Week 1)
- [ ] Next.js dashboard app scaffolded in `dashboard/`
- [ ] SQLite database initialized with schema
- [ ] Agent registry: auto-detect + manual add
- [ ] Basic dashboard home with agent list
- [ ] Workspace tree view

### Phase 2: Core Agent Control (Week 2-3)
- [ ] Agent launcher with PTY-based session management
- [ ] Live session monitor with xterm.js
- [ ] Session logging to SQLite
- [ ] Project registry with auto-scan
- [ ] Quick-launch agent scoped to project

### Phase 3: Orchestration (Week 4-5)
- [ ] Task router with agent selection
- [ ] Workflow builder (basic: 2-3 step chains)
- [ ] Parallel session management
- [ ] Knowledge base UI (read + search)
- [ ] Context injection (manual)

### Phase 4: Intelligence Layer (Week 6+)
- [ ] Session log search
- [ ] Auto-context injection
- [ ] Cowork session detection
- [ ] Agent performance tracking
- [ ] Session knowledge capture

---

## 10. Risks & Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| PTY complexity across agents | High | High | Start with simple spawn, upgrade to PTY. Test each agent individually. |
| Agent CLI APIs change | Medium | Medium | Abstract agent interface. Adapters per agent. Easy to update one without breaking others. |
| Scope creep (this vision is massive) | High | High | Ruthless Tier 1 focus. Ship MVP with registry + launcher + monitor only. |
| SQLite concurrency with many sessions | Low | Medium | WAL mode. Single-writer pattern. |
| xterm.js rendering performance | Medium | Medium | Limit scrollback buffer. Lazy-load session output. |
| Energy management (Jeff's epilepsy) | Always | High | Design for short, high-impact sessions. Quick-launch everything. Minimize cognitive load in UI. |

---

## 11. Open Questions

- [ ] Should the dashboard run on a fixed port (e.g., 4000) or be configurable?
- [ ] How to handle agent-specific prompt formats when injecting context? (Claude uses XML, Gemini uses markdown, etc.)
- [ ] Should session logs include token usage? (Requires parsing agent output for cost info)
- [ ] Best approach for Cowork detection — process monitoring vs file watching?
- [ ] Should the knowledge base support tags/frontmatter or keep it pure Obsidian-style?
- [ ] How to handle agents that don't support piped stdin (interactive-only agents)?

---

## 12. Appendix

### A. Workspace Hierarchy

```
Jeffs Agent OS/
├── .agent-os/                    # Agent OS internal state
│   ├── config.json               # Global configuration
│   ├── registry/                 # Agent definitions (JSON per agent)
│   ├── sessions/                 # Session metadata & logs
│   └── logs/                     # System logs
│
├── business/                     # Business operations
│   ├── plans/                    # Business plans, strategy docs
│   ├── finances/                 # Budget, revenue, expenses
│   ├── brand/                    # Brand guidelines, assets
│   └── social-media/             # Content calendar, posts, analytics
│
├── projects/                     # All coding projects
│   ├── personal/                 # Personal apps & experiments
│   ├── business/                 # Revenue-generating products
│   └── tools/                    # Internal dev tools & utilities
│
├── knowledge/                    # Obsidian-compatible knowledge base
│   ├── me/                       # Personal context (bio, preferences, health, goals)
│   ├── business/                 # Business context (The Little AI Company, clients, market)
│   ├── tech/                     # Technical learnings, stack notes, patterns
│   ├── agents/                   # Agent-specific knowledge (strengths, configs, tips)
│   ├── social/                   # Online presence, social media strategy, audience
│   └── templates/                # Reusable templates for notes, plans, etc.
│
├── planning/                     # Project planning hub
│   ├── active/                   # Currently active project PRDs/plans
│   ├── backlog/                  # Ideas, future projects
│   └── archive/                  # Completed project plans
│
├── dashboard/                    # The Agent OS web dashboard (Next.js app)
│
├── docs/                         # Agent OS documentation
│   ├── PRD.md                    # This document
│   ├── DESIGN.md                 # UI/UX design document
│   └── ARCHITECTURE.md           # Technical architecture document
│
├── CLAUDE.md                     # Root-level agent instructions
├── CHANGELOG.md                  # Version history
└── project-status.md             # Current state tracking
```

### B. Supported Agents (Initial)

| Agent | Binary | Model Default | Strengths |
|-------|--------|---------------|-----------|
| Claude Code | `claude` | claude-sonnet-4 | Architecture, refactoring, complex reasoning |
| Gemini CLI | `gemini` | gemini-2.5-pro | Breadth, large context, research |
| Codex CLI | `codex` | codex | Speed, quick edits, scripting |
| Open Code | `opencode` | varies | Multi-model support, flexibility |
| Quen Code | `qwen` | qwen-2.5-coder | Free tier, good for simple tasks |
| Kumi CLI | `kumi` | varies | TBD |
| Quine | `quine` | varies | TBD |
| Root Code | `rootcode` | varies | TBD |

### C. Glossary

- **Agent:** An AI coding assistant accessible via CLI (e.g., `claude`, `gemini`)
- **Session:** A single invocation of an agent with a specific task/context
- **Workflow:** A multi-step chain of agent sessions where output feeds forward
- **Registry:** The database of known/installed agents and their configurations
- **Knowledge Base:** The Obsidian-compatible markdown vault for persistent context
- **Task Router:** The system that matches tasks to the best-fit agent

### D. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-03-14 | Initial draft |
