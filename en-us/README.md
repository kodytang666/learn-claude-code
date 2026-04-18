# Claude Code Internals: A Source Code Deep Dive (English)

> A deep-dive course into Claude Code's internals, built from direct source code analysis.

← [Back to Course Home](../README.md) | [中文版](../zh-cn/README.md)

---

## Table of Contents

| Chapter | File | Topic |
|---------|------|-------|
| 0 | [00-overview.md](00-overview.md) | Course overview & Agent workflow |
| 1 | [01-agent-loop.md](01-agent-loop.md) | **Agent Loop**: the core query engine |
| 2 | [02-system-prompt-construction.md](02-system-prompt-construction.md) | **System Prompt**: construction & priority |
| 3 | [03-tool-system.md](03-tool-system.md) | **Tool System**: calls, permissions & deferred disclosure |
| 4 | [04-todo-task-system.md](04-todo-task-system.md) | **TodoWrite & Task System**: tracking & background tasks |
| 5 | [05-subagent-teams.md](05-subagent-teams.md) | **Subagent & Agent Teams**: multi-agent orchestration |
| 6 | [06-skills-deferred-loading.md](06-skills-deferred-loading.md) | **Skills System**: deferred loading & on-demand search |
| 7 | [07-context-compaction.md](07-context-compaction.md) | **Context Compaction**: auto-compact & memory restoration |
| 8 | [08-token-budget.md](08-token-budget.md) | **Token Budget**: budget tracking & diminishing-returns detection |
| 9 | [09-hooks.md](09-hooks.md) | **Hook System**: 27 lifecycle events & extension protocol |
| 10 | [10-worktree-isolation.md](10-worktree-isolation.md) | **Worktree Isolation**: Git Worktree sandboxing |
| 11 | [11-architecture.md](11-architecture.md) | **Architecture Overview**: sequence diagrams, flowcharts & module map |
