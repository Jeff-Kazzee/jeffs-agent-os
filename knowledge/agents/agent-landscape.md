# AI Coding Agent Landscape (2026)

## Jeff's Active Agents

### Claude Code
- **Binary:** `claude`
- **Strengths:** Architecture, refactoring, complex reasoning, multi-file edits
- **Model:** claude-sonnet-4 (default), claude-opus-4 (heavy lifting)
- **Notes:** Primary agent. Best for system design and large refactors.

### Gemini CLI
- **Binary:** `gemini`
- **Strengths:** Large context window, research, breadth of knowledge
- **Model:** gemini-2.5-pro
- **Notes:** Good for research tasks and working with large codebases.

### Codex CLI
- **Binary:** `codex`
- **Strengths:** Speed, quick edits, scripting
- **Notes:** Good for rapid iteration on small tasks.

### Open Code
- **Binary:** `opencode`
- **Strengths:** Multi-model support, flexibility
- **Notes:** Useful for switching models mid-task.

### Quen Code
- **Binary:** `qwen`
- **Strengths:** Free tier, good for simple tasks
- **Model:** qwen-2.5-coder
- **Notes:** Free vibes. Use for tasks that don't need heavy reasoning.

### Others (Less Frequently Used)
- **Kumi CLI** (`kumi`)
- **Quine** (`quine`)
- **Root Code** (`rootcode`)

## Agent Selection Heuristic

| Task Type | Recommended Agent | Why |
|-----------|------------------|-----|
| Architecture/design | Claude Code | Best reasoning, system-level thinking |
| Large codebase edits | Gemini CLI | Largest context window |
| Quick bug fixes | Codex CLI | Fastest turnaround |
| Research/exploration | Gemini CLI | Broad knowledge |
| Free tier work | Quen Code | No cost |
| Multi-model experiments | Open Code | Model switching |
