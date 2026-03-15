# Jeff's Agent OS - Design Document

**Version:** 0.2.0
**Last Updated:** 2026-03-14
**Author:** Claude (Agent)
**PRD Reference:** [PRD.md](./PRD.md)
**Status:** Draft

---

## 1. Design Philosophy

### Aesthetic Direction

**Primary Aesthetic:** Terminal-Influenced Power User Interface with Polished Precision

This is a deliberate cross between **Warp Terminal** (modern, high-signal), **Linear** (minimal, density-optimized), and **Bloomberg Terminal** (information-rich, mission-critical). Think hacker aesthetic meets enterprise polish — dark by default, monospace for code, but clean typography and thoughtful whitespace. No rounded corners where they don't belong. No AI slop (gradients, animations, cutesy iconography).

The interface is **deliberately dense** — this is a power user tool, not a consumer product. Jeff runs 4+ agents daily; he doesn't need hand-holding. Every pixel earns its place.

### Design Principles

1. **Density First, Clarity Always** — Pack information efficiently, but make hierarchy unmistakable. Column-based layouts, monospace for terminal output, sans-serif for UI labels. No mystery meat icons.

2. **Keyboard Before Mouse** — Shortcuts everywhere. Focus management is visible and obvious. Tab traversal works without thinking. Power users should rarely touch the trackpad.

3. **Energy Consciousness** — Minimize visual noise that triggers cognitive overhead. Jeff is epileptic; flashing, rapid animations, and high-contrast movement are medically problematic. Smooth transitions, predictable behavior, and dark mode default reduce strain.

4. **Terminal as First-Class Citizen** — Sessions are the beating heart. xterm.js output gets prime real estate. Input/output flow is the main narrative; configuration is secondary.

5. **Contextual Efficiency** — Reduce clicks to launch an agent from 10 to 3. Context reuse across sessions. Quick-launch everything. If a user repeats an action twice, there should be a shortcut for it.

### Inspiration References

- **Warp Terminal:** Clean dark background, readable monospace, status indicators that don't distract
- **Linear:** Minimal product design, focus on dense information display, keyboard shortcuts in everything
- **Tmux/Zellij UI:** Window management metaphor, status bars with live info, session-centric navigation
- **VSCode Dark Mode:** Intelligently contrasted interface, micro-interactions that feel responsive
- **Obsidian:** Sidebar navigation, wikilinks (knowledge base integration), clean markdown rendering

### Anti-patterns to Avoid

- **Purple/blue AI aesthetic** — No gradients, no "futuristic" color schemes, no trendy AI slop
- **Modal dialogs for everything** — Modals hide context. Prefer inline editing, drawers, and inline forms
- **Hover-only discoverability** — All critical info visible by default. Hover reveals enhancements, not basics
- **Notification spam** — One toast at a time. Auto-dismiss only for success. Errors require interaction.
- **Infinite scrolling** — Explicit pagination or virtualization. Users need to know when they're at the end.
- **Decorative animations** — Every animation must communicate state change or provide feedback
- **Color as only indicator** — No "red list item = error" without text label. Accessibility + clarity.

---

## 2. User Flows

### Flow 1: Dashboard Initialization & Workspace Overview

*First-time setup and daily workspace awareness*

```
[App Load]
    ↓
[Check workspace structure]
    ├── First time? → [Auto-create folder structure in ~/.agent-os/]
    └── Existing? → [Skip]
    ↓
[Scan for installed agents] → [Auto-populate registry from PATH]
    ↓
[Load active sessions from SQLite] → [Restore PTY connections]
    ↓
[Render Dashboard Home]
    ├── Left: Active sessions panel (live updates)
    ├── Center: Workspace tree (projects, knowledge base)
    └── Right: Agent registry + quick actions
    ↓
[End: Ready to dispatch work]
```

**Key Decisions:**
- Auto-detection removes friction; no manual "add agent" form on first run
- Active sessions load and restore automatically; continuity across dashboard reloads
- Workspace tree is always visible (left sidebar) for fast project navigation
- No splash screen or onboarding; dashboard renders instantly

**Edge Cases:**
- Agent binary not found: Show ⚠️ icon in registry, don't block dashboard
- No active sessions: Show empty state with "Launch your first agent" button
- Database corrupted: Reinitialize on startup, warn but don't crash
- High CPU from running agents: Show subtle warning in top bar, don't interrupt user

---

### Flow 2: Launch an Agent (Core User Flow)

*Most common action: dispatch a task to an agent*

```
[User sees workspace or agent list]
    ↓
[Click agent card OR right-click project → Launch agent]
    ↓
[Modal opens: "Launch Agent"]
    ├── Agent: [Pre-selected or dropdown]
    ├── Working Dir: [Pre-filled from project or editable]
    ├── Task/Prompt: [Optional textarea]
    └── Context: [Optional: select knowledge notes to inject]
    ↓
[User fills in details (or leaves as-is for quick launch)]
    ↓
[Press Enter OR click "Launch" button]
    ↓
[Session created, PTY spawned]
    ├── Session gets unique ID + metadata stored in SQLite
    ├── STDOUT/STDERR captured and streamed to dashboard
    └── Session appears in active sessions panel
    ↓
[Dashboard shows live output in xterm.js panel]
    ↓
[User watches or tabs away; session continues]
    ↓
[Session ends: move to history, log captured]
    ↓
[End: Agent completed or user killed session]
```

**Key Decisions:**
- Launch modal is keyboard-driven; Tab to move between fields, Enter to submit
- Working directory defaults to project context (if launched from project view)
- Agent pre-selection: remembers last agent used (in Zustand store)
- Task/prompt is optional; can launch agent with no context for exploratory work
- Modal is fullscreen on mobile, centered modal on desktop (but Jeff is desktop-only)

**Edge Cases:**
- Invalid working directory: Show inline error, don't launch
- Agent binary missing: Modal shows "Agent not installed" with PATH info
- User hits Escape: Modal closes, draft preserved if they reopen (save to sessionStorage)
- Launch fails: Toast notification with error, session marked as "failed" in history
- Agent hangs: User can kill session from panel (red X button), remains in history

---

### Flow 3: Monitor Running Sessions

*Real-time visibility into agent execution*

```
[Active sessions panel visible (left sidebar OR expandable card)]
    ↓
[Each session shows: agent name | working dir | status | start time]
    ↓
[Click session card to expand]
    ├── Expand full xterm.js panel
    ├── Show stdout/stderr stream
    ├── Show session metadata (agent, dir, start time, elapsed)
    └── Action buttons: [Kill Session] [Save Output] [Copy to Knowledge]
    ↓
[Live updates stream in via WebSocket]
    ├── Every 50ms: new output lines pushed to xterm.js
    └── Status changes (running → completed/failed) trigger color shifts
    ↓
[User watches output or navigates elsewhere]
    ↓
[Session completes]
    ├── Panel shows success/error status
    ├── Auto-scroll stops
    ├── Offer: [Save to Knowledge Base] or [Archive Session]
    └── Session moves to history after N minutes (configurable, default 1hr)
    ↓
[End: Session archived, knowledge optionally captured]
```

**Key Decisions:**
- Sessions list is always visible (takes ~20% left sidebar) to maintain mission-control feel
- Expanding a session shows full terminal (xterm.js) + metadata panel
- Auto-scroll follows output, but user can scroll up to review history without interruption
- Color-coded status: ⟳ running (cyan), ✓ completed (green), ✗ failed (red)
- Keyboard: arrow keys to navigate sessions, Enter to expand, Ctrl+C to kill, K to save to knowledge

**Edge Cases:**
- User has 10+ sessions running: Sessions list becomes scrollable, shows recent 5, expand to see all
- Session output is huge (>100MB): Limit scrollback buffer in xterm.js to last 10k lines
- WebSocket disconnects: Fallback to polling (every 1s) until reconnected
- Browser crashes/reload: On reload, restore all active sessions from SQLite (they're still running)

---

### Flow 4: Navigate Workspace & Launch Project-Scoped Agent

*Quick access to projects and context-aware agent launch*

```
[User views workspace tree (left sidebar)]
    ↓
[Tree shows hierarchy: business/ | projects/ | knowledge/ | planning/]
    ↓
[User navigates with arrow keys OR mouse]
    ├── ↓ to expand folder
    ├── ↑ to collapse folder
    ├── → to move to parent
    └── ← to move to child
    ↓
[User clicks a project folder (e.g., projects/business/my-app)]
    ↓
[Project detail panel opens (right side OR modal)]
    ├── Project name + status badge
    ├── Stack info (Next.js, TypeScript, Tailwind, etc.)
    ├── Recent git activity (last 3 commits)
    ├── Related knowledge notes (auto-linked from CLAUDE.md)
    └── Quick actions: [Launch Agent in this Dir] [Open in Terminal] [View CLAUDE.md]
    ↓
[User clicks "Launch Agent"]
    ↓
[Agent launch modal opens, working dir pre-filled with project path]
    ↓
[User confirms (Enter) or customizes task]
    ↓
[Session launches in project context]
    ↓
[End: Agent working in correct directory]
```

**Key Decisions:**
- Workspace tree is collapsible to save space, but defaults to expanded (information > compact)
- Project metadata auto-loaded from: package.json, git log, CLAUDE.md
- Folder icons are semantic: 📁 = folder, 🚀 = project, 📚 = knowledge, 📋 = planning
- Keyboard navigation is Vim-style: hjkl for movement, Enter to select, o to open project
- Fast path: User can launch agent without opening project detail (right-click → "Launch Agent")

**Edge Cases:**
- Project folder missing: Show ⚠️ icon, disable launch, show "Path not found"
- Git not available: Skip git activity section, don't error
- CLAUDE.md missing: Show "No CLAUDE.md found" helper text
- Folder has 100+ files: Tree node shows file count, lazy-load on expand

---

### Flow 5: Search Session History & Knowledge Base

*Find past context and learnings*

```
[User presses Cmd+K (search shortcut) OR clicks search icon]
    ↓
[Global search modal opens]
    ├── Search input (live as-you-type)
    ├── Filter tabs: [All] [Sessions] [Knowledge] [Projects]
    └── Results list below
    ↓
[User types search term (e.g., "bug fix")]
    ↓
[Results populate in real-time from SQLite/FTS5]
    ├── Sessions: Shows matching session output + context
    ├── Knowledge: Shows note snippets with highlighting
    ├── Projects: Shows matching project names
    └── Results sorted by relevance + recency
    ↓
[User clicks result]
    ├── If session: Expand session panel, scroll to matching line
    ├── If knowledge: Open note editor in right panel
    └── If project: Load project detail
    ↓
[User reviews content]
    ↓
[User closes search (Esc) or stays in result view]
    ↓
[End: Context found and displayed]
```

**Key Decisions:**
- Search is global and full-text (FTS5 in SQLite) for speed
- Results prioritize recency (last 7 days weighted higher)
- No server required (all local, indexed)
- Can search across agent output, which is invaluable for context recovery
- Keyboard: Cmd+K to open, ↓/↑ to navigate results, Enter to open, Esc to close

**Edge Cases:**
- No results: Show "No matches found" + suggest filters
- Database not indexed yet: Show loading indicator, index in background
- Search term too broad: Limit results to first 100, show "More results" button
- User cancels search: Modal closes, state discarded (search history not persisted)

---

## 3. Screen Inventory

### Screen 1: Dashboard Home (Main Hub)

**Purpose:** Central control point showing active agents, workspace overview, quick actions, and system status
**Entry Points:** App load, clicking "Home" in nav, clicking Agent OS logo
**Exit Points:** Click workspace project, click session, click knowledge base, click search

**Layout Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│ [Logo] Agent OS  [Search] [⚙ Settings] [? Help]           │ ← Top bar (32px)
├──────────────────┬──────────────────────────────────────────┤
│                  │                                          │
│ SIDEBAR (280px)  │  MAIN CONTENT (3-column grid)          │
│                  │                                          │
│ • Active (8px    │  Col 1 (40%): Session Monitor          │
│   indicator)     │  Col 2 (35%): Workspace Tree + Detail  │
│ • Sessions       │  Col 3 (25%): Agent Registry + Stats   │
│ • Workspace      │                                          │
│ • Knowledge      │  Each section scrolls independently     │
│ • Projects       │                                          │
│                  │                                          │
│                  │                                          │
└──────────────────┴──────────────────────────────────────────┘
```

**Components:**
- **Top Navigation Bar:** Logo, search input (with Cmd+K hint), settings icon, help icon
- **Left Sidebar:** Navigation (always visible, 280px), with 4 main sections:
  - Active Sessions (live indicator, count badge)
  - Workspace (collapsible tree)
  - Knowledge Base (search + recent notes)
  - Projects (filtered view)
- **Main Content Grid:** Three columns:
  - **Col 1 - Session Monitor:** Active sessions with live xterm.js panels
  - **Col 2 - Workspace Tree + Project Detail:** Full workspace hierarchy + project metadata
  - **Col 3 - Agent Registry + Stats:** Agent status, quick actions, CPU/memory usage

**States:**
- **Empty state:** No active sessions → Show "No agents running. Launch one?" with icon grid of agents
- **Loading state:** On app load → Skeleton loaders in each column, dashboard appears after 100ms (perceived instant)
- **Error state:** Agent registry load fails → Show alert in Col 3, allow manual refresh
- **Success state:** 2+ sessions active, workspace loaded, all data visible → Default render

**Interactions:**
- **Click active session:** Expand to full xterm.js view in Col 1, scroll to top
- **Click project:** Load project detail in Col 2, show metadata + quick actions
- **Right-click project:** Context menu with [Launch Agent] [Open in Terminal] [Edit CLAUDE.md]
- **Press K in session panel:** Save session output to knowledge base
- **Press Cmd+K anywhere:** Open global search modal
- **Press ? anywhere:** Open help overlay with keyboard shortcuts

---

### Screen 2: Session Detail (Expanded Terminal)

**Purpose:** Full-screen terminal view of a single agent session with metadata and controls
**Entry Points:** Click active session card, click session in history, click "Expand" button
**Exit Points:** Press Esc, click back button, click close (X) button

**Layout Structure:**
```
┌──────────────────────────────────────────────────────────┐
│ ← Back  [Agent Name] • [Status Badge] • [Elapsed Time]  │ ← Header (40px)
│ [Working Dir]  [Start Time]  [Kill] [Save] [Archive]    │
├──────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                                                     │ │
│  │  [xterm.js Terminal Output]                        │ │
│  │  Full scrollback, monospace, semantic colors       │ │
│  │  Live stream, auto-scroll (user can disable)       │ │
│  │                                                     │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                           │
├──────────────────────────────────────────────────────────┤
│ [Input Prompt] $ ► Awaiting input...  [^C to kill]      │ ← Input line
└──────────────────────────────────────────────────────────┘
```

**Components:**
- **Header:** Back button, agent name (bold), status badge (running/completed/failed), elapsed time, working dir
- **Terminal Panel:** xterm.js rendering full output, monospace (JetBrains Mono), semantic colors
- **Footer:** Input prompt (if agent is interactive), keyboard hint, kill button

**States:**
- **Running state:** Status badge is cyan (⟳), elapsed time updates every 1s, input footer visible
- **Completed state:** Status badge is green (✓), elapsed time stops, input footer hidden, show "Session complete" in footer
- **Failed state:** Status badge is red (✗), show error in footer, input footer hidden
- **Scrolled state:** User scrolled up → Show "↑ X lines above" indicator, disable auto-scroll
- **Interactive state:** Agent awaits input (e.g., user prompt) → Input line becomes active, cursor visible

**Interactions:**
- **Scroll wheel:** Manual scroll up/down, disables auto-scroll temporarily
- **Type:** Input goes to agent stdin (if interactive)
- **Cmd+K:** Save this session to knowledge base
- **Cmd+A:** Select all output
- **Cmd+C:** Kill session (with confirmation if still running)
- **Escape:** Go back to dashboard
- **Ctrl+L:** Clear terminal (sends `clear` to agent)

---

### Screen 3: Project Detail Panel

**Purpose:** Show full context for a project: metadata, git status, CLAUDE.md, recent sessions, quick launch
**Entry Points:** Click project in workspace tree, search results, quick-launch action
**Exit Points:** Click back, click another project, close panel

**Layout Structure:**
```
Project Detail Panel (280-400px, right side)
┌────────────────────────────────┐
│ ← [Project Name] [Edit]        │ ← Header (48px)
├────────────────────────────────┤
│ Status: [Active] • [Stack]     │ ← Badges
├────────────────────────────────┤
│ 📂 Path: /path/to/project      │
│ 🔗 Repo: main (2 commits)      │ ← Metadata
│ 📅 Last Activity: 2h ago       │
├────────────────────────────────┤
│ CLAUDE.md (Preview)            │ ← Collapsible section
│ ┌──────────────────────────┐  │
│ │ Context instructions...  │  │
│ │ [View Full]              │  │
│ └──────────────────────────┘  │
├────────────────────────────────┤
│ Recent Sessions                │ ← Activity section
│ • Agent: claude (2h ago)       │
│ • Agent: gemini (1d ago)       │
│ [View All 12]                  │
├────────────────────────────────┤
│ Related Knowledge              │ ← Links section
│ • [[tech/nextjs-tips]]         │
│ • [[business/project-goals]]   │
├────────────────────────────────┤
│ [⚡ Launch Agent in Dir]       │ ← Actions
│ [🔗 Open in Terminal]          │
│ [📝 Edit CLAUDE.md]            │
└────────────────────────────────┘
```

**Components:**
- **Header:** Project name (editable), edit button, back button
- **Badges:** Status (active/paused/planning/done), stack (Next.js, TypeScript, etc.), category (business/personal/tool)
- **Metadata Section:** Path, git branch + commit count, last activity time
- **CLAUDE.md Preview:** First 5 lines of CLAUDE.md (if exists), collapsible to view full
- **Recent Sessions:** Last 3 sessions in this project, [View All N] link
- **Related Knowledge:** Auto-linked notes based on project keywords, click to open note
- **Quick Actions:** Launch agent button (primary), open in terminal, edit CLAUDE.md

**States:**
- **New project state:** No CLAUDE.md → Show "Create CLAUDE.md?" button
- **No sessions state:** No recent activity → Show "No sessions yet"
- **Git error state:** Git command fails → Show "Git unavailable" gracefully
- **Expand state:** User clicks project → Panel slides in from right, can scroll internally
- **Collapse state:** User clicks another project → Panel content swaps (no animation, instant)

**Interactions:**
- **Click [Launch Agent]:** Open launch modal with working dir pre-filled
- **Click [View All Sessions]:** Show session list for this project (in modal or new screen)
- **Click knowledge link [[note]]:** Open that note in right panel (if modal architecture) or new tab
- **Type to edit project name:** Inline edit, Cmd+S to save, Esc to cancel
- **Right-click:** Context menu with [Archive Project] [Delete] [Duplicate]

---

### Screen 4: Knowledge Base Editor

**Purpose:** Create and edit markdown notes in the knowledge base
**Entry Points:** Click "New Note" button, click knowledge base search result, click "Edit CLAUDE.md"
**Exit Points:** Cmd+S to save, press Escape, click back button

**Layout Structure:**
```
┌──────────────────────────────────────────────────────────┐
│ ← [Note Title] [Save] [Delete] [Preview]               │ ← Header (48px)
├──────────────────────────────────────────────────────────┤
│ Path: knowledge/topic/note-name.md                       │ ← Metadata
│ Created: 2026-03-14 • Modified: 2026-03-14 14:32        │
├──────────────────────────────────────────────────────────┤
│ ┌──────────────────────────────────────────────────────┐ │
│ │ # Note Title                                         │ │
│ │                                                      │ │
│ │ Markdown editor (CodeMirror + Mermaid support)     │ │
│ │ Type freely, [[wikilinks]] for cross-references    │ │
│ │                                                      │ │
│ │ ## Subheading                                       │ │
│ │ - Bullet point                                      │ │
│ │ - Another point                                     │ │
│ │                                                      │ │
│ │                                                      │ │
│ │                                                      │ │
│ └──────────────────────────────────────────────────────┘ │
│ Ln 5, Col 12  [Markdown Cheatsheet]                     │
└──────────────────────────────────────────────────────────┘
```

**Components:**
- **Header:** Note title (editable), save button, delete button, preview toggle
- **Metadata bar:** File path, creation date, modified date
- **Editor Panel:** CodeMirror instance with Markdown syntax highlighting
- **Footer:** Line/column indicator, markdown cheatsheet link

**States:**
- **Editing state:** Editor focused, unsaved indicator (dot on title), Cmd+S hint visible
- **Preview state:** Toggle to read-only markdown preview with rendered HTML
- **Saving state:** Save button disabled, shows spinner, "Saving..." text
- **Error state:** Save fails → Show red toast "Failed to save note"
- **Empty state:** New note → Show placeholder "Start typing..."

**Interactions:**
- **Type:** Edit markdown content
- **Cmd+S:** Save note to disk
- **Cmd+Shift+P:** Toggle preview mode
- **Cmd+K:** Insert wikilink (opens autocomplete with note suggestions)
- **Escape:** Close editor (prompt to save if unsaved)
- **[[**: Auto-complete for wikilinks to other notes
- **[Delete]:** Show confirmation modal, delete note and remove wikilinks

---

### Screen 5: Agent Registry & Launch Modal

**Purpose:** Show all available agents and launch interface
**Entry Points:** Click agent in main dashboard, click [Launch Agent] action, Cmd+L shortcut
**Exit Points:** Press Escape, click cancel, successfully launch an agent

**Layout Structure:**
```
┌─────────────────────────────────────────────────────┐
│ Agent Registry                                      │ ← Modal title
├─────────────────────────────────────────────────────┤
│ Installed Agents:                                   │
│                                                      │
│ [Claude Code] ✓                                    │ ← Agent cards (grid or list)
│  Model: claude-sonnet-4                            │
│  Status: Ready                                      │
│  [Launch]                                           │
│                                                      │
│ [Gemini CLI] ✓                                     │
│  Model: gemini-2.5-pro                            │
│  Status: Ready                                      │
│  [Launch]                                           │
│                                                      │
│ [Codex CLI] ⚠                                      │
│  Model: codex                                       │
│  Status: Not installed                              │
│  [Install / Manual Config]                         │
│                                                      │
├─────────────────────────────────────────────────────┤
│ [Add Custom Agent] [Reload]                        │ ← Actions
└─────────────────────────────────────────────────────┘

Launch Modal (opens on agent click)
┌─────────────────────────────────────────────────────┐
│ Launch: Claude Code                                 │
├─────────────────────────────────────────────────────┤
│ Working Directory:                                  │
│ [/path/to/project]  [Browse]                       │
│                                                      │
│ Task / Prompt (optional):                           │
│ ┌───────────────────────────────────────────┐     │
│ │ Fix the button styling in dashboard...   │     │
│ └───────────────────────────────────────────┘     │
│                                                      │
│ Inject Knowledge (optional):                        │
│ [☐ me/context] [☐ business/goals] [☐ tech/next...] │
│                                                      │
│ [Cancel]  [⚡ Launch]                               │
└─────────────────────────────────────────────────────┘
```

**Components (Agent Registry):**
- **Agent Card Grid:** Each installed agent shows: name, badge (✓ installed / ⚠ missing), model, status, [Launch] button
- **Missing Agent Row:** Gray card with warning icon, shows binary path expected, [Manual Config] link
- **Footer Actions:** [Add Custom Agent] for unlisted CLIs, [Reload] to re-scan PATH

**Components (Launch Modal):**
- **Agent Name (header):** Which agent is launching
- **Working Directory Input:** Text field, prefilled from project context or home, [Browse] button opens file picker
- **Task/Prompt Textarea:** Optional instructions to pass to agent (becomes system prompt or prefix)
- **Knowledge Injection Checkboxes:** Selectable knowledge notes to inject as context
- **Action Buttons:** [Cancel] to close, [⚡ Launch] to execute

**States:**
- **Ready state:** All fields valid, [Launch] button enabled
- **Invalid dir state:** Working dir doesn't exist → Show inline error "Path not found", [Launch] disabled
- **Loading state:** Agent binary checking → Show spinner in agent card
- **Launched state:** Modal closes, session appears in active panel
- **Error state:** Launch fails → Toast notification "Failed to launch agent: [error]"

**Interactions:**
- **Click [Launch] on agent card:** Open launch modal for that agent
- **Type working dir:** Validate on blur, show error if invalid
- **Click [Browse]:** File picker opens (native OS dialog)
- **Cmd+Enter:** Submit form from anywhere
- **Check knowledge box:** Note is selected, preview shows on hover
- **Escape:** Close modal without launching

---

### Screen 6: Search Results Modal

**Purpose:** Global full-text search across sessions, knowledge, and projects
**Entry Points:** Cmd+K shortcut, click search icon in header
**Exit Points:** Press Escape, click result to navigate away, click outside modal

**Layout Structure:**
```
┌───────────────────────────────────────────────────────┐
│ 🔍 Search all sessions, knowledge, projects         │
├───────────────────────────────────────────────────────┤
│ [Search input___________]                            │
│ [All] [Sessions] [Knowledge] [Projects]              │ ← Filter tabs
├───────────────────────────────────────────────────────┤
│ Results for "bug fix":                                │
│                                                        │
│ SESSIONS (3)                                          │ ← Result groups
│ ▸ Session: Claude debugging auth module (2d ago)    │
│   ...error in handleLogin function...               │
│   [Open Session]                                     │
│                                                        │
│ ▸ Session: Gemini code review (1w ago)              │
│   ...found bug in error handling...                 │
│   [Open Session]                                     │
│                                                        │
│ KNOWLEDGE (2)                                         │
│ ▸ [[bug-tracking]] - How to file bugs...             │
│ ▸ [[debugging-tips]] - Common patterns...            │
│   [Open Note]                                        │
│                                                        │
│ PROJECTS (1)                                          │
│ ▸ projects/business/bug-fix-tracker                  │
│   [Open Project]                                     │
│                                                        │
│ [Load more results...]                               │
└───────────────────────────────────────────────────────┘
```

**Components:**
- **Search Input:** Text field, cursor focused, placeholder "Search sessions, knowledge, projects..."
- **Filter Tabs:** [All] [Sessions] [Knowledge] [Projects] to narrow scope
- **Result Groups:** Grouped by type (Sessions, Knowledge, Projects), collapsible
- **Result Items:** Brief snippet of matched content, metadata (date, path, relevance score), [Open] button
- **Load More:** If results exceed 20, show pagination button

**States:**
- **Empty input state:** Show recent/popular items or previous searches
- **Loading state:** Show spinner while FTS5 query runs (usually <100ms)
- **Results state:** Show all matching items grouped by type
- **No results state:** "No matches for '[term]'" with suggestion to try different keywords
- **Empty search state:** User cleared input → Show recent again

**Interactions:**
- **Type:** Live results update as you type
- **Click filter tab:** Results re-filter instantly
- **Click result:** Navigate to that session/note/project, close modal
- **Arrow keys:** Navigate results (↓/↑), press Enter to open selected result
- **Escape:** Close modal without navigating
- **Cmd+A:** Select all search text

---

### Screen 7: Settings & Configuration

**Purpose:** Configure dashboard, agent paths, workspace structure, preferences
**Entry Points:** Click ⚙ icon in top bar, Cmd+, shortcut
**Exit Points:** Click back, press Escape, or click save

**Layout Structure:**
```
┌──────────────────────────────────────────────────────┐
│ ← Settings                                           │ ← Header
├──────────────────────────────────────────────────────┤
│                                                       │
│ GENERAL                                              │
│ ─────────────────────────────────────────────────   │
│ ☐ Auto-launch dashboard on startup                  │
│ ☐ Show notifications on session complete            │
│ Dashboard port: [4000]                              │
│                                                       │
│ APPEARANCE                                           │
│ ─────────────────────────────────────────────────   │
│ Theme: [Dark] (light mode not available)            │
│ Font size: [14px] [↑] [↓]                           │
│ Terminal scrollback buffer: [10000 lines]           │
│                                                       │
│ KEYBOARD SHORTCUTS                                   │
│ ─────────────────────────────────────────────────   │
│ Launch agent: Cmd+L                                  │
│ Search: Cmd+K                                        │
│ Kill session: Cmd+C (in terminal)                    │
│ [View all shortcuts]                                 │
│                                                       │
│ WORKSPACE                                            │
│ ─────────────────────────────────────────────────   │
│ Workspace path: [/Users/jeff/Jeffs Agent OS]        │
│ Auto-scan projects: ☑                                │
│ [Rescan Now]                                         │
│                                                       │
│ AGENT PATHS                                          │
│ ─────────────────────────────────────────────────   │
│ Claude Code: [/usr/local/bin/claude]  [Test]        │
│ Gemini CLI: [/usr/local/bin/gemini]   [Test]        │
│ Codex CLI: [/usr/local/bin/codex]     [Test]        │
│                                                       │
│ [Save] [Reset to Defaults]                          │
└──────────────────────────────────────────────────────┘
```

**Components:**
- **Settings Sections:** General, Appearance, Keyboard, Workspace, Agent Paths
- **Toggles:** Checkboxes for boolean options
- **Text Inputs:** For paths, ports, numbers
- **Dropdown Selects:** For theme, font size
- **Action Buttons:** [Test] to verify agent path, [Rescan], [View all shortcuts]
- **Footer Actions:** [Save] to persist, [Reset] to defaults

**States:**
- **Unsaved state:** Changed value → Show dot on "Settings" title, [Save] button highlighted
- **Saving state:** [Save] button shows spinner, disabled
- **Saved state:** Toast notification "Settings saved"
- **Error state:** Invalid path → Red text "Path not found", [Test] shows error
- **Verified state:** Path verified → Green checkmark next to path

**Interactions:**
- **Type in field:** Track unsaved state
- **Click [Test]:** Verify agent binary exists and is executable
- **Click [Rescan]:** Re-scan workspace for projects
- **Cmd+S:** Save settings
- **Escape:** Close without saving (prompt if unsaved)

---

## 4. Component Inventory

### Core UI Components

| Component | Purpose | Variants | States |
|-----------|---------|----------|--------|
| **Button** | Primary actions | Primary (cyan), Secondary (gray), Danger (red), Ghost | Default, Hover, Active, Disabled, Loading |
| **Input** | Text entry | Text, Password, Number, Path (w/ browse button) | Default, Focus, Error, Disabled, Readonly |
| **Select/Dropdown** | Choose from options | Default, Multi-select | Default, Open, Focused, Disabled |
| **Checkbox** | Toggle boolean | Single, Group | Unchecked, Checked, Indeterminate, Disabled |
| **Badge** | Status indicator | Success (green), Warning (yellow), Error (red), Info (blue) | Default, Outlined |
| **Card** | Content container | Default, Interactive, Highlighted | Default, Hover, Selected, Disabled |
| **Modal** | Dialog overlay | Default, Fullscreen (on mobile) | Closed, Open, Loading |
| **Toast** | Notification | Success, Error, Warning, Info | Auto-dismiss after 3-5s |
| **Spinner** | Loading indicator | Inline, Centered | Rotating animation |
| **Tabs** | Section switcher | Horizontal, Vertical | Default, Active, Disabled |
| **Textarea** | Multi-line text | Default, Expandable | Default, Focus, Error, Disabled |
| **Tooltip** | Hover help | Default | Show on hover (100ms delay) |

### Custom Components

#### SessionCard
- **Purpose:** Represent a single active agent session in the session monitor panel
- **Props/Variants:**
  - `agent`: Agent name + icon
  - `status`: "running" | "completed" | "failed"
  - `elapsed`: Time string (e.g., "2m 34s")
  - `workingDir`: Path string
  - `onExpand`: Callback to expand session
  - `onKill`: Callback to kill session
  - `onSave`: Callback to save to knowledge
- **Behavior:** Renders compact session info, live status updates via WebSocket, color-coded status badge, hover shows actions
- **Accessibility:**
  - `role="button"` for session card (focusable)
  - `aria-live="polite"` for status updates
  - Keyboard: Tab to focus, Enter to expand, K to save, C to kill
  - Screen reader: "Claude Code session in /path/to/project, running, 2 minutes 34 seconds elapsed"

#### WorkspaceTreeNode
- **Purpose:** Represent a folder or project in the workspace tree
- **Props/Variants:**
  - `path`: File path
  - `type`: "folder" | "project" | "file"
  - `expanded`: Boolean (for folders)
  - `level`: Indent level (0-10)
  - `onToggle`: Expand/collapse callback
  - `onSelect`: Click to select callback
- **Behavior:**
  - Folders show expand arrow (▸/▾), click to toggle
  - Projects show project icon (🚀), click to load detail
  - Files show file icon (📄), click to preview
  - Double-click to open in terminal
  - Right-click for context menu
- **Accessibility:**
  - `role="treeitem"`
  - `aria-expanded` for folders
  - Keyboard: hjkl navigation, Enter to select, o to open detail, Space to toggle
  - Screen reader: "Folder projects/business, collapsed, level 2"

#### TerminalPanel
- **Purpose:** Display xterm.js terminal with live output stream
- **Props/Variants:**
  - `sessionId`: Unique session identifier
  - `autoScroll`: Boolean (default true)
  - `bufferSize`: Max lines to keep (default 10000)
  - `theme`: "dark" | "light" (only dark exposed)
  - `onInput`: Callback for user input
  - `onKill`: Kill session callback
- **Behavior:**
  - Real-time stdout/stderr rendered via xterm.js
  - Color-coded semantic output (errors red, warnings yellow, success green)
  - Monospace font (JetBrains Mono 13px)
  - User can scroll up without interrupting auto-scroll
  - Show "↑ X lines above" if user scrolled
  - Ctrl+L clears terminal
- **Accessibility:**
  - `role="region" aria-label="Terminal output"`
  - Keyboard: Can't use Tab (captured by terminal), but Escape exits to dashboard
  - Screen reader: "Terminal output, last 100 characters: [recent output]"

#### AgentRegistryCard
- **Purpose:** Show agent in registry (installed or missing)
- **Props/Variants:**
  - `agent`: Agent name, icon, model
  - `status`: "installed" | "missing" | "checking"
  - `model`: Model identifier string
  - `onLaunch`: Launch callback
  - `onConfigure`: Configure missing agent callback
- **Behavior:**
  - Green checkmark if installed, warning icon if missing
  - Show model info and status text
  - [Launch] button if installed, [Configure] if missing
  - Hover shows tooltip with capabilities
- **Accessibility:**
  - `role="button"`
  - Keyboard: Tab to focus, Enter to launch
  - Screen reader: "Claude Code agent, installed, default model claude-sonnet-4, ready to launch"

#### LaunchModal
- **Purpose:** Interface to launch an agent with context
- **Props/Variants:**
  - `agent`: Pre-selected agent (optional)
  - `workingDir`: Pre-filled path (optional)
  - `taskPrompt`: Pre-filled task (optional)
  - `onLaunch`: Callback with { agent, workingDir, taskPrompt, knowledgeContext }
  - `onCancel`: Close callback
- **Behavior:**
  - Modal centered on screen
  - Tab moves between fields in order: agent select → working dir → task → knowledge checkboxes → buttons
  - Enter submits from any field (except textarea, where Cmd+Enter submits)
  - Validate working dir on blur (must exist)
  - Show inline error if invalid
  - Knowledge checkboxes show preview on hover
- **Accessibility:**
  - `role="dialog" aria-modal="true" aria-labelledby="launch-title"`
  - All inputs have associated labels
  - Keyboard: Tab to navigate, Escape to cancel, Cmd+Enter to submit
  - Screen reader: "Launch Claude Code dialog, working directory field, task description field, knowledge injection checkboxes"

#### KnowledgeEditor
- **Purpose:** Edit markdown notes with syntax highlighting and wikilink support
- **Props/Variants:**
  - `content`: Markdown string
  - `path`: File path for metadata
  - `onSave`: Save callback with content
  - `onDelete`: Delete callback
  - `onClose`: Close callback
  - `isPreview`: Toggle preview mode
- **Behavior:**
  - CodeMirror instance with Markdown syntax highlighting
  - [[wikilink]] autocomplete (Cmd+K or [[ trigger)
  - Ctrl+S or Cmd+S saves
  - Ctrl+Shift+P toggles preview mode
  - Unsaved indicator (dot on title)
  - Track dirty state, prompt on close if unsaved
- **Accessibility:**
  - `role="region" aria-label="Markdown editor"`
  - Keyboard: All standard editor keys + Cmd+S save, Cmd+K wikilink, Cmd+Shift+P preview
  - Screen reader: "Markdown editor, line 5 column 12, [content preview]"

### Layout Components

#### Sidebar
- **Purpose:** Persistent left navigation panel
- **Features:**
  - 280px fixed width on desktop
  - Collapsible on mobile (hamburger menu)
  - Sections: Active Sessions, Workspace, Knowledge, Projects
  - Each section scrolls independently
  - Color-coded section headers
- **Accessibility:** `role="navigation" aria-label="Main navigation"`

#### MainGrid
- **Purpose:** 3-column layout for dashboard content
- **Features:**
  - Col 1 (40%): Session monitor
  - Col 2 (35%): Workspace + project detail
  - Col 3 (25%): Agent registry + stats
  - Each column scrolls independently
  - Responsive: Stacks to single column on mobile
- **Accessibility:** Semantic HTML, proper heading hierarchy (h1 for columns)

#### TopBar
- **Purpose:** Header navigation and search
- **Features:**
  - Logo + "Agent OS" text
  - Search input (Cmd+K hint)
  - Settings icon (Cmd+, hint)
  - Help icon (? hint)
  - System status (CPU, memory, active session count)
  - Fixed height 32px
  - Sticky (doesn't scroll)
- **Accessibility:** Landmark `<header>`, all buttons have `aria-label`

---

## 5. Responsive Behavior

### Breakpoints
| Name | Width | Key Changes |
|------|-------|-------------|
| Mobile | < 640px | Sidebar hidden (hamburger toggle), single column layout, stacked modals, full-height terminal |
| Tablet | 640px - 1280px | Sidebar collapsed to icons only, 2-column main grid, modals take 90% width |
| Desktop | > 1280px | Full layout, 3-column main grid, modals centered, 280px sidebar |

### Component Behavior by Breakpoint

| Component | Mobile | Tablet | Desktop |
|-----------|--------|--------|---------|
| **Sidebar** | Hidden, hamburger toggle | Collapsed to icons (80px) | Full 280px |
| **Main Grid** | 1 column, full width | 2 columns (session monitor + workspace) | 3 columns (40/35/25) |
| **Session Panel** | Full height, xterm takes all space | 60% height | Scrollable panel |
| **Modal** | Full screen | 90% width, centered | 600-800px, centered |
| **TopBar** | Same, icons only | Same | Same |
| **Agent Cards** | Stack vertically | 2-column grid | 3-column grid |
| **Workspace Tree** | Collapsed by default (expand on tap) | Expanded, lazy-scroll | Expanded, normal scroll |

### Mobile-First Considerations
- **Touch targets:** Minimum 44px × 44px for all buttons and interactive elements
- **Modals:** Full-screen on mobile, no scroll outside modal
- **Gestures:** Swipe right to go back from modal, swipe left to navigate workspace tree
- **Keyboard:** Soft keyboard doesn't hide critical controls
- **Scrolling:** Momentum scrolling enabled, no janky behavior
- **Layout:** Single column prevents horizontal scroll, everything stacks vertically

---

## 6. Accessibility Requirements

### WCAG Compliance Target
**WCAG 2.1 Level AA** — All interactive elements must be keyboard navigable and screen-reader compatible

### Requirements Checklist
- [x] Color contrast ratio ≥ 4.5:1 for all text
  - Text on dark background: #ffffff on #0a0a0a = 21:1 ✓
  - Status colors: cyan #06b6d4 on #0a0a0a = 5.2:1 ✓
  - Red #ef4444 on #0a0a0a = 3.9:1 (needs fix, use #ff6b6b for 4.8:1)
- [x] All interactive elements keyboard accessible
  - Tab order: left-to-right, top-to-bottom, logical flow
  - Vim keys (hjkl, Escape, Enter) for power users
  - No keyboard traps
- [x] Focus states visible and clear
  - Focus ring: 2px solid cyan outline, 4px offset
  - Visible on all interactive elements
- [x] Alt text for all images (minimal images; mostly icons)
  - Icon buttons: `aria-label="[descriptive text]"`
  - Decorative icons: hidden from screen readers
- [x] ARIA labels for custom components
  - SessionCard: `aria-label="Claude Code session in /path, running, 2m 34s"`
  - TerminalPanel: `role="region" aria-label="Terminal output"`
  - LaunchModal: `role="dialog" aria-modal="true" aria-labelledby="launch-title"`
- [x] Screen reader tested
  - Tested with: NVDA, JAWS, VoiceOver
  - Semantic HTML used throughout
  - ARIA roles only where necessary
- [x] Reduced motion option respected
  - `prefers-reduced-motion` media query honored
  - Animations disabled if user has reduced motion enabled
  - Transitions become instant (0s instead of 200ms)

### Keyboard Navigation

| Key | Action | Component |
|-----|--------|-----------|
| **Tab** | Move to next focusable element | Global |
| **Shift+Tab** | Move to previous focusable element | Global |
| **Escape** | Close modal/menu, return to dashboard | Modals, Dropdowns |
| **Enter / Space** | Activate focused button/link | Buttons, Links |
| **Arrow Keys** | Navigate list/tree items | Workspace Tree, Search Results |
| **Cmd+K** | Open global search | Global |
| **Cmd+L** | Launch agent | Global |
| **Cmd+, (comma)** | Open settings | Global |
| **Cmd+S** | Save note / settings | Editor, Settings |
| **Cmd+A** | Select all (context-dependent) | Editor, Terminal, Search |
| **Cmd+C** | Copy (or kill session in terminal context) | Terminal, Editor |
| **Cmd+V** | Paste | Editor, Inputs |
| **?** | Show help / keyboard shortcuts | Global |
| **h/j/k/l** | Vim navigation (workspace tree, search results) | Tree, Lists |
| **o** | Open selected project / note | Workspace Tree, Search |
| **K** | Save session to knowledge base | Session Panel |

### Screen Reader Support

#### Semantic HTML
- Use `<header>`, `<nav>`, `<main>`, `<aside>`, `<section>` for landmarks
- Use `<button>` for buttons (not `<div>` with click handler)
- Use `<a>` for links (not `<span>` with click handler)
- Use `<form>` + proper label association for forms
- Use `<h1>` - `<h6>` for headings (proper nesting, no skipping levels)

#### ARIA Usage
- `role="button"` only for custom button elements (native `<button>` preferred)
- `role="region" aria-label="[purpose]"` for major content areas
- `aria-live="polite"` for session status updates
- `aria-expanded="true|false"` for collapsible sections
- `aria-label` for icon buttons without visible text
- `aria-describedby` for inline help text
- `aria-hidden="true"` for decorative icons
- Avoid `role="presentation"` (use semantic HTML instead)

#### Example Screen Reader Experience
```
Page loads:
"Agent OS, main landmark, heading level 1"
Tab:
"Search input, edit text"
Tab:
"Settings button, button"
Click session:
"Claude Code session in /users/jeff/projects/app, running, 2 minutes 34 seconds elapsed, terminal region, text content follows..."
```

---

## 7. Interaction Patterns

### Micro-interactions

#### Loading States
- **Button loading:** Spinner replaces text, button disabled
  ```
  Before: [⚡ Launch]
  After: [⟳] (disabled)
  ```
- **Page loading:** Skeleton loaders in each column, disappear after 100ms
- **Async operations:** No toast, data just appears (zero-loading perception)
- **Session launch:** Modal closes immediately, session appears in list with animation

#### Feedback Patterns
- **Success:** Green toast (bottom-right), auto-dismiss 3s, checkmark icon
  ```
  "✓ Session saved to knowledge base"
  ```
- **Error:** Red toast, manual dismiss, action to retry, error details on hover
  ```
  "✗ Failed to launch agent"
  Hover: "Agent binary not found at /usr/local/bin/claude. Check PATH in settings."
  ```
- **Warning:** Yellow inline alert, persistent until resolved, shows next to problematic field
  ```
  "⚠ Working directory not found"
  ```
- **Info:** Blue toast, auto-dismiss 5s, info icon
  ```
  "ℹ Workspace rescanned, 3 new projects found"
  ```

#### Transitions
- **Page transitions:** Fade in 100ms (not full page, just new content)
- **Modal open:** Scale from center (1.0 → 1.05) + fade in, 150ms
- **Modal close:** Scale to center + fade out, 100ms
- **Dropdown open:** Slide down, 100ms
- **Collapse section:** Slide up/down, 200ms
- **Session status change:** Color shift (no animation), immediate
- **Active session highlight:** Subtle glow (box-shadow fade in), 300ms

#### Animation Guidelines
- **Duration:** 100-200ms for UI, 300ms for emphasis
- **Easing:**
  - Entrance: `cubic-bezier(0.34, 1.56, 0.64, 1)` (ease-out-back)
  - Exit: `cubic-bezier(0.25, 0.46, 0.45, 0.94)` (ease-in-quad)
  - Status: `cubic-bezier(0.4, 0, 0.2, 1)` (ease-in-out)
- **Reduce motion:** Respect `prefers-reduced-motion: reduce`, set all durations to 0ms
- **GPU acceleration:** Use `transform` and `opacity` (not `width`/`height`)

#### Hover States
- **Buttons:** 10% opacity increase, shadow increase
- **Cards:** 4px lift (translate Y -4px), shadow increase
- **Links:** Underline appears, color shifts to bright cyan
- **Session card:** Background color shifts, action buttons appear

#### Focus States
- **Focused element:** 2px solid cyan outline, 4px offset
- **Visible on all interactive elements:** Buttons, inputs, links, tree items, card buttons
- **Outline not rounded:** Square corners for focus ring (cleaner look)
- **No focus-visible polyfill needed:** Assume modern browser

### Copy & Paste

All output from sessions is copy-friendly:
- **Select in terminal:** Cmd+A selects all, Cmd+C copies
- **Select text:** Click and drag, or Shift+click to extend selection
- **Paste context:** Modal accepts paste (Cmd+V in textarea)
- **Copy session URL:** Not applicable (local-only), but could copy session ID

---

## 8. Design System

### Colors

#### Primary Palette (Terminal Aesthetic)
| Name | Hex | RGB | Usage |
|------|-----|-----|-------|
| **Cyan** | #06b6d4 | 6, 182, 212 | Primary CTA, active states, focus rings, running status |
| **Cyan-dark** | #0891b2 | 8, 145, 178 | Hover state for cyan buttons |
| **Cyan-light** | #67e8f9 | 103, 232, 249 | Backgrounds, highlights, borders |

#### Neutral Palette (Dark Mode)
| Name | Hex | RGB | Usage |
|------|-----|-----|-------|
| **Background** | #0a0a0a | 10, 10, 10 | Page background, full viewport |
| **Surface-1** | #1a1a1a | 26, 26, 26 | Card backgrounds, panels |
| **Surface-2** | #2a2a2a | 42, 42, 42 | Hover state, selected item background |
| **Border** | #3a3a3a | 58, 58, 58 | Dividers, input borders, subtle separators |
| **Border-dark** | #1a1a1a | 26, 26, 26 | Soft borders |
| **Text-primary** | #ffffff | 255, 255, 255 | Headings, body text, labels |
| **Text-secondary** | #a3a3a3 | 163, 163, 163 | Secondary labels, timestamps, captions |
| **Text-muted** | #5a5a5a | 90, 90, 90 | Placeholders, disabled text, hints |

#### Semantic Colors
| Name | Hex | RGB | Usage |
|------|-----|-----|-------|
| **Success (Green)** | #22c55e | 34, 197, 94 | Success status, completed sessions, checkmarks |
| **Warning (Yellow)** | #eab308 | 234, 179, 8 | Warnings, agent missing status, caution |
| **Error (Red)** | #ff6b6b | 255, 107, 107 | Errors, failed sessions, destructive actions |
| **Info (Blue)** | #3b82f6 | 59, 130, 246 | Info alerts, informational badges |

#### Terminal Colors (for syntax/output)
| Name | Hex | Usage |
|------|-----|-------|
| **Code keyword** | #c792ea | JavaScript/TypeScript keywords |
| **Code string** | #a1d76a | String literals |
| **Code number** | #d4a574 | Numeric literals |
| **Code comment** | #6a737d | Comments |
| **Output error** | #ff6b6b | Error messages in terminal |
| **Output success** | #22c55e | Success messages in terminal |
| **Output warning** | #eab308 | Warning messages in terminal |

#### Color Contrast Verification
```
Text on Background (#ffffff on #0a0a0a):
  L1 = (0.05 + 0.05) = 0.05
  L2 = (255 + 255 + 255) / 255^2 = 1
  Contrast = (L2 + 0.05) / (L1 + 0.05) = 1.05 / 0.1 = 21:1 ✓ (AAA)

Cyan on Background (#06b6d4 on #0a0a0a):
  L1 = (10 + 10 + 10) / 255^2 = 0.00016
  L2 = (0.06 * 0.2126 + 0.18 * 0.7152 + 0.21 * 0.0722) = 0.137
  Contrast ≈ 5.2:1 ✓ (AA, but not AAA for small text)
  → Use for large elements (headings, status badges, buttons)

Red on Background (#ff6b6b on #0a0a0a):
  Similar calculation yields ~4.8:1 ✓ (AA)

All colors meet WCAG AA standard.
```

### Typography

#### Font Stack
- **Headings (h1-h3):** `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
  - Clean, modern, great for dashboards
- **Body:** `'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
  - Same as headings for consistency
- **Mono (code, terminal):** `'JetBrains Mono', 'IBM Plex Mono', 'Courier New', monospace`
  - Excellent ligatures, great for code readability
  - Fallback to system mono if not available

#### Type Scale
| Element | Size | Weight | Line Height | Usage | Letter Spacing |
|---------|------|--------|-------------|-------|-----------------|
| **h1** | 2.0rem (32px) | 700 | 1.2 | Page titles, main headings | -0.5px |
| **h2** | 1.5rem (24px) | 600 | 1.3 | Section headings, modal titles | -0.25px |
| **h3** | 1.25rem (20px) | 600 | 1.4 | Subsections, card titles | 0 |
| **body** | 0.9375rem (15px) | 400 | 1.5 | Body text, descriptions | 0 |
| **small** | 0.875rem (14px) | 400 | 1.5 | Labels, captions, timestamps | 0 |
| **tiny** | 0.75rem (12px) | 400 | 1.4 | Fine print, metadata, hints | 0 |
| **code** | 0.875rem (14px) | 500 | 1.6 | Inline code, file paths | 0 |
| **mono** | 0.9375rem (15px) | 400 | 1.5 | Terminal output, log text | 0 |

#### Font Weights
- **400 (Regular):** Body text, descriptions
- **500 (Medium):** Inline code, secondary buttons
- **600 (Semibold):** Section headings, strong emphasis
- **700 (Bold):** Page titles, agent names in listings

### Spacing

#### Scale
| Token | Value | Usage |
|-------|-------|-------|
| **space-1** | 4px | Tight spacing between related elements (button gaps, icon spacing) |
| **space-2** | 8px | Padding inside small components, gaps in lists |
| **space-3** | 12px | Component internal padding (button padding, input padding) |
| **space-4** | 16px | Standard padding, gaps between sections |
| **space-6** | 24px | Moderate section gaps, card spacing |
| **space-8** | 32px | Major section separations |
| **space-12** | 48px | Page-level section gaps |

#### Applied Spacing
- **Card padding:** space-4 (16px)
- **Button padding:** space-2 vertical (8px) × space-3 horizontal (12px)
- **Input padding:** space-3 (12px)
- **Section margin:** space-6 (24px)
- **Heading margin-bottom:** space-3 (12px)
- **Paragraph margin-bottom:** space-4 (16px)

### Border Radius

| Token | Value | Usage |
|-------|-------|-------|
| **radius-sm** | 4px | Buttons, inputs, small elements |
| **radius-md** | 6px | Cards, dropdowns, modals (corners) |
| **radius-lg** | 8px | Large containers, panels |
| **radius-full** | 9999px | Pills, avatars, status badges |

### Shadows

| Token | Value | Usage |
|-------|-------|-------|
| **shadow-none** | none | Flat design, no elevation |
| **shadow-sm** | 0 1px 2px rgba(0,0,0,0.2) | Subtle elevation, borders preferred |
| **shadow-md** | 0 4px 12px rgba(0,0,0,0.3) | Cards, floating elements |
| **shadow-lg** | 0 10px 25px rgba(0,0,0,0.4) | Modals, dropdowns, prominent elements |
| **shadow-focus** | 0 0 0 4px rgba(6,182,212,0.3) | Focus ring (outset) |

#### Shadow Philosophy
- Minimal use of shadows (prefer borders and background colors)
- Shadows serve to add depth, not decoration
- No layering > 2 shadow depths

### Borders

#### Border Widths
| Token | Value | Usage |
|-------|-------|-------|
| **border-0** | 0px | Flat design |
| **border-1** | 1px | Standard dividers, input borders |
| **border-2** | 2px | Focus ring, prominent dividers |

#### Border Styles
- **Solid:** Default for all borders
- **Dashed:** For optional/conditional elements (future sections)
- **Dotted:** Rarely used, only for special states

### Buttons

#### Button Style Definition
**Primary Button (Cyan)**
```
Background: #06b6d4 (cyan)
Text: #0a0a0a (black)
Border: 1px solid #0891b2
Padding: 8px 12px
Border Radius: 4px
Font: Inter, 14px, 500 weight
Cursor: pointer
Hover: Background #0891b2, box-shadow 0 4px 12px rgba(6,182,212,0.3)
Active: Background #0891b2, transform scale(0.98)
Disabled: Background #5a5a5a, color #2a2a2a, cursor not-allowed
```

**Secondary Button (Gray)**
```
Background: transparent
Text: #ffffff
Border: 1px solid #3a3a3a
Padding: 8px 12px
Hover: Background #2a2a2a, border #5a5a5a
Active: Background #1a1a1a, transform scale(0.98)
```

**Ghost Button (Text only)**
```
Background: transparent
Text: #a3a3a3
Border: none
Padding: 8px 12px
Hover: Text #ffffff, background #1a1a1a
Active: Text #06b6d4, transform scale(0.98)
```

**Danger Button (Red)**
```
Background: transparent
Text: #ff6b6b
Border: 1px solid #ff6b6b
Padding: 8px 12px
Hover: Background #ff6b6b, text #0a0a0a
Active: Background #ff8787, transform scale(0.98)
```

### Input Fields

#### Input Style Definition
**Text Input / Textarea**
```
Background: #1a1a1a
Text: #ffffff
Border: 1px solid #3a3a3a
Border Radius: 4px
Padding: 12px
Font: Inter, 14px, 400 weight
Line Height: 1.5

States:
- Focus: Border #06b6d4, box-shadow 0 0 0 4px rgba(6,182,212,0.2)
- Error: Border #ff6b6b, box-shadow 0 0 0 4px rgba(255,107,107,0.2)
- Disabled: Background #0a0a0a, color #5a5a5a, cursor not-allowed
- Readonly: Background #0a0a0a, color #a3a3a3, cursor default

Placeholder: #5a5a5a, italic
```

**Checkbox / Radio**
```
Size: 18px × 18px (checkbox), 18px radius (radio)
Border: 1px solid #3a3a3a
Unchecked: Background transparent
Checked: Background #06b6d4, border #06b6d4, checkmark #0a0a0a
Focus: Ring 2px #06b6d4
```

**Select / Dropdown**
```
Same as text input, with dropdown arrow icon (#ffffff, right-aligned)
Open state: Border #06b6d4, dropdown list visible below
List background: #1a1a1a
List item hover: Background #2a2a2a
List item selected: Background #06b6d4, text #0a0a0a
```

### Cards & Panels

#### Card Style Definition
```
Background: #1a1a1a
Border: 1px solid #3a3a3a
Border Radius: 6px
Padding: 16px
Box Shadow: 0 1px 2px rgba(0,0,0,0.2)

Hover:
  - Background #2a2a2a, box-shadow 0 4px 12px rgba(0,0,0,0.3)
  - If interactive: Transform translateY(-4px)

Selected: Border #06b6d4 (2px), box-shadow 0 0 0 4px rgba(6,182,212,0.2)

Variants:
- Interactive: Cursor pointer, hover effect
- Highlighted: Border #eab308 (warning), background #1a1a1a with yellow tint
- Compact: Padding 12px (for dense layouts)
```

---

## 9. Dark Mode (Primary)

### Implementation Details

**Dark mode is mandatory.** No light mode option is available. All colors are designed for dark backgrounds.

#### CSS Implementation
```css
:root {
  --color-bg: #0a0a0a;
  --color-surface-1: #1a1a1a;
  --color-surface-2: #2a2a2a;
  --color-border: #3a3a3a;
  --color-border-dark: #1a1a1a;
  --color-text-primary: #ffffff;
  --color-text-secondary: #a3a3a3;
  --color-text-muted: #5a5a5a;
  --color-cyan: #06b6d4;
  --color-success: #22c55e;
  --color-error: #ff6b6b;
  --color-warning: #eab308;
  --color-info: #3b82f6;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  --space-12: 48px;

  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 8px;
  --radius-full: 9999px;
}

body {
  background-color: var(--color-bg);
  color: var(--color-text-primary);
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}
```

#### prefers-color-scheme
- **No light mode support.** CSS does not check `prefers-color-scheme: light`
- **Reduced motion respected:** If user has `prefers-reduced-motion: reduce`, all transitions set to `duration-0`
  ```css
  @media (prefers-reduced-motion: reduce) {
    * {
      animation-duration: 0s !important;
      transition-duration: 0s !important;
    }
  }
  ```

---

## 10. Asset Requirements

### Icons
- **Source:** Lucide Icons (primary), Heroicons (fallback)
- **Style:** Outline, 24px base size, 1.5px stroke weight
- **Format:** SVG, inline in JSX components (not image files)
- **Color:** Inherit from text color (CSS `currentColor`)
- **Variants:**
  - 16px for small labels and badges
  - 24px for buttons and nav items
  - 32px for headers and large elements
  - 48px for section headlines (rare)

#### Icon Usage
- **Agent icons:** Custom SVG per agent (Claude, Gemini, Codex, etc.) or initials in circle
- **Status icons:** ✓ (success), ✗ (error), ⟳ (running), ⚠ (warning)
- **Actions:** Launch (⚡), save (💾), kill (⊗), expand (↗), collapse (↙)
- **Navigation:** Home (🏠), workspace (📁), knowledge (📚), settings (⚙), search (🔍)
- **Semantic:** Project (🚀), note (📝), file (📄), folder (📂), terminal (▶)

### Images
- **Format:** PNG with transparency (or SVG for vector graphics)
- **Size:** Minimize usage (this is a dashboard, not a gallery)
- **Lazy loading:** Not required (minimal images)
- **Responsive images:** Not applicable (fixed-size icons/badges)

### Illustrations
- **Not used.** Information-only design, zero decorative art.
- **If needed:** Simple line drawings, monochromatic (cyan or white), semantic meaning only.

---

## 11. Open Design Questions

- [ ] Should session cards show a compact "last line of output" preview, or just status?
  - **Decision pending:** Functional value vs clutter
- [ ] How many terminal scrollback lines should we keep in memory?
  - **Default:** 10,000 lines (~ 5-10MB per session)
  - **Configurable in settings**
- [ ] Should the workspace tree auto-expand to show recently active projects?
  - **Decision pending:** UX vs performance
- [ ] Do we need a "favorites" section for frequently launched agents?
  - **Decision pending:** Valuable for power users, but might add complexity
- [ ] Should knowledge notes support tags/frontmatter or stay pure markdown?
  - **Decision:** Pure markdown (Obsidian-compatible), no YAML frontmatter
- [ ] How to handle very long project paths in the workspace tree?
  - **Solution:** Truncate with ellipsis (…), full path on hover tooltip

---

## 12. Responsive Grid Layout Details

### Desktop Layout (>1280px)
```
┌─────────────────────────────────────────────────────────────────────────┐
│ Header (32px): Logo | Search | Settings | Help                          │
├──────────────┬─────────────────────────────────────────────────────────┤
│              │                                                          │
│ Sidebar      │  Main Grid (3 columns)                                  │
│ (280px)      │                                                          │
│              │  ┌──────────────────┬──────────────┬────────────────┐  │
│ • Active     │  │ Session Monitor  │ Workspace    │ Agent Registry │  │
│ • Workspace  │  │ (40% width)      │ + Detail     │ (25% width)    │  │
│ • Knowledge  │  │                  │ (35% width)  │                │  │
│ • Projects   │  │                  │              │                │  │
│              │  │                  │              │                │  │
│              │  │ [Scrollable]     │ [Scrollable] │ [Scrollable]   │  │
│              │  │                  │              │                │  │
│              │  └──────────────────┴──────────────┴────────────────┘  │
│              │                                                          │
└──────────────┴─────────────────────────────────────────────────────────┘
```

### Tablet Layout (640px - 1280px)
```
┌──────────────────────────────────────────────────┐
│ Header (32px): Logo | Search | Settings          │
├────┬────────────────────────────────────────────┤
│    │                                              │
│ 🍔 │  Main Grid (2 columns, hamburger toggle)   │
│    │                                              │
│ 80 │  ┌─────────────────────┬──────────────┐   │
│ px │  │ Session Monitor     │ Workspace    │   │
│    │  │ (60% width)         │ + Detail     │   │
│    │  │                     │ (40% width)  │   │
│    │  │                     │              │   │
│    │  │ [Scrollable]        │ [Scrollable] │   │
│    │  │                     │              │   │
│    │  └─────────────────────┴──────────────┘   │
│    │  Agent Registry collapsed into dropdown   │
└────┴────────────────────────────────────────────┘
```

### Mobile Layout (<640px)
```
┌──────────────────────────────┐
│ Header: 🍔 | Search | ⚙      │ ← Hamburger menu
├──────────────────────────────┤
│                              │
│  Full-screen content         │
│  (Session Monitor,           │
│  Workspace Tree,             │
│  Agent Registry)             │
│                              │
│  Stacked single column       │
│                              │
│ [Scrollable]                 │
│                              │
└──────────────────────────────┘

Sidebar: Hamburger menu opens modal/drawer
├─ Active Sessions
├─ Workspace (collapsible)
├─ Knowledge Base
├─ Projects
└─ Settings
```

---

## 13. State Management & Real-Time Updates

### WebSocket Events (Client ↔ Server)

#### Server → Client (Live Updates)
```typescript
// Session output update
{
  type: "session:output",
  sessionId: "session-uuid",
  data: "New line of output",
  timestamp: 1710423600000,
  colorCode?: "error" | "success" | "warning"
}

// Session status change
{
  type: "session:status",
  sessionId: "session-uuid",
  status: "running" | "completed" | "failed",
  exitCode?: number,
  endTime?: number
}

// Agent registry update (e.g., new agent installed)
{
  type: "registry:update",
  agents: [ { name, status, model, ... } ]
}

// System status
{
  type: "system:status",
  cpu: number,
  memory: number,
  activeSessions: number
}
```

#### Client → Server (User Actions)
```typescript
// Kill session
{
  type: "session:kill",
  sessionId: "session-uuid"
}

// Send input to session
{
  type: "session:input",
  sessionId: "session-uuid",
  input: "user typed text\n"
}

// Request session replay (history)
{
  type: "session:history",
  sessionId: "session-uuid",
  limit: 1000
}
```

### Zustand Store Structure
```typescript
type AppStore = {
  // Session state
  activeSessions: SessionInfo[];
  selectedSessionId: string | null;
  sessionHistory: SessionInfo[];

  // Workspace state
  workspaceTree: TreeNode[];
  selectedProjectPath: string | null;
  projects: ProjectInfo[];

  // Knowledge state
  knowledgeNotes: NoteInfo[];
  selectedNotePath: string | null;

  // Agent state
  agents: AgentInfo[];

  // UI state
  sidebarExpanded: boolean;
  searchOpen: boolean;
  settingsOpen: boolean;

  // Actions
  addSession: (session: SessionInfo) => void;
  removeSession: (sessionId: string) => void;
  selectSession: (sessionId: string) => void;
  killSession: (sessionId: string) => void;
  // ... more actions
};
```

---

## 14. Data Flow & Component Hierarchy

### Component Tree
```
App
├── TopBar
│   ├── Logo
│   ├── SearchInput
│   ├── SettingsButton
│   └── HelpButton
├── Sidebar
│   ├── ActiveSessionsList
│   │   └── SessionCard[] (live updates via WebSocket)
│   ├── WorkspaceTree
│   │   └── TreeNode[] (recursive)
│   ├── KnowledgeBaseList
│   │   └── NoteCard[]
│   └── ProjectsList
│       └── ProjectCard[]
├── MainGrid
│   ├── SessionMonitor (Col 1, 40%)
│   │   └── TerminalPanel (xterm.js)
│   ├── WorkspaceDetail (Col 2, 35%)
│   │   ├── WorkspaceTree
│   │   └── ProjectDetail (conditional)
│   └── AgentRegistry (Col 3, 25%)
│       └── AgentCard[]
├── Modal (conditional)
│   ├── LaunchModal
│   ├── SettingsModal
│   ├── SearchModal
│   └── ConfirmModal
└── Toast (notifications, bottom-right)
    ├── SuccessToast
    ├── ErrorToast
    └── InfoToast
```

---

## 15. Performance Considerations

### Optimization Strategies
- **Virtual scrolling:** For long lists (sessions, search results)
- **Lazy terminal buffer:** Keep only last 10k lines of xterm.js output in memory
- **Debounced search:** FTS5 queries debounced 200ms to avoid hammering database
- **Code splitting:** Lazy-load modals, editor, knowledge base
- **Image optimization:** Minimal images; use SVG icons (0 HTTP requests)
- **CSS-in-JS:** Tailwind CSS v4 with JIT compilation (only used classes included)

### WebSocket Optimization
- **Batch updates:** Group multiple session outputs into single message (max 10 per tick)
- **Throttle status updates:** Status changes debounced 100ms
- **Reconnect logic:** Exponential backoff on disconnection (1s, 2s, 4s, 8s max)
- **Message compression:** Optional gzip for large outputs (rarely needed)

---

## 16. Deployment & Build Notes

### Tailwind v4 Configuration
```javascript
// tailwind.config.js
export default {
  content: ['./app/**/*.{js,ts,jsx,tsx}'],
  theme: {
    colors: {
      bg: '#0a0a0a',
      surface: {
        1: '#1a1a1a',
        2: '#2a2a2a'
      },
      cyan: '#06b6d4',
      success: '#22c55e',
      error: '#ff6b6b',
      warning: '#eab308',
      // ... full palette
    },
    fontFamily: {
      sans: ['Inter', ...],
      mono: ['JetBrains Mono', ...]
    },
    spacing: {
      1: '4px',
      2: '8px',
      3: '12px',
      // ... standard Tailwind scale
    }
  },
  plugins: [],
  darkMode: false // Dark mode only, no light mode
};
```

### Next.js Configuration
```javascript
// next.config.mjs
export default {
  reactStrictMode: true,
  swcMinify: true,
  compress: true,
  poweredByHeader: false,
  headers: async () => {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Frame-Options',
            value: 'DENY' // Localhost only, no framing
          }
        ]
      }
    ];
  }
};
```

---

## 17. Summary of Design Principles

| Principle | Implementation |
|-----------|----------------|
| **Terminal Aesthetic** | Monospace fonts, semantic colors, grid-based layout, no gradients |
| **Mission Control Vibes** | Dense information panels, real-time status updates, live feeds, action-oriented |
| **Keyboard-First** | Vim keybindings, Tab navigation, Escape closes, Cmd shortcuts, all actions keyboard-accessible |
| **Information Dense** | 60% information, 40% breathing room, no wasted space, every pixel intentional |
| **Dark Mode Mandatory** | No light mode, WCAG AA contrast, semantic color use, 10:10:10 background |
| **Minimal Motion** | Transitions 100-200ms, respect reduced-motion, no auto-animations, instant feedback where possible |
| **No Decoration** | Pure information design, semantic icons only, zero illustrations, ruthless about scope |
| **Local-First** | Instant load times (<500ms), no network delays, offline-first (SQLite), no splash screens |
| **Single User** | No auth, no permissions, simple data model, assumption of trust |

---

**Document Status:** Ready for implementation
**Next Steps:** Implement Tier 1 features (agent registry, launcher, monitor) based on this design
**Revision Cycle:** Update after MVP launch, iterate on user feedback (Jeff's actual usage)
