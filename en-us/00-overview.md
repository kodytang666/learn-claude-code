# Claude Code Source Code Deep Dive

> A hands-on teardown of Agent engineering based on Claude Code's real source code. All content is derived from direct analysis of the source.

---

## Course Overview

This course is aimed at two types of readers:

- **Beginners**: Developers who have never built an Agent system and want to understand how a production-grade Agent works end-to-end through a real-world example.
- **Advanced readers**: Developers who already have Agent development experience and want to deeply understand Claude Code's sophisticated design choices in areas like prompt cache, task isolation, and context compaction.

---

## Agent Workflow at a Glance

After Claude Code starts, a complete user interaction flows through the following sequence:

```
User Input
  │
  ▼
[Global Context Prompt Construction]  ← CLAUDE.md / memory files / env info
  │
  ▼
[Agent Loop Main Loop]                ← query.ts
  │
  ├─→ [Claude API Streaming Call]
  │
  ├─→ [Tool Execution]               ← Tool System (with dynamic disclosure)
  │      ├─ TodoWrite (task tracking)
  │      ├─ Skill (on-demand skill loading)
  │      ├─ Agent (Subagent / Fork / Team)
  │      └─ Bash / FileRead / FileEdit ...
  │
  ├─→ [Token Budget Check]           ← Continue or stop?
  │
  ├─→ [Auto Compaction Check]        ← Is context too long?
  │
  └─→ [Hook Execution]               ← User-defined extension points
        ├─ PreToolUse / PostToolUse
        ├─ Stop / SubagentStart
        └─ PreCompact / PostCompact
```

---

## Table of Contents

| Chapter | File | Core Topic |
|---------|------|------------|
| Chapter 0 | [00-overview.md](00-overview.md) | Course overview and Agent workflow |
| Chapter 1 | [01-agent-loop.md](01-agent-loop.md) | **Agent Loop**: The core engine of the main query loop |
| Chapter 2 | [02-system-prompt-construction.md](02-system-prompt-construction.md) | **Global Context Prompt**: System prompt construction and priority |
| Chapter 3 | [03-tool-system.md](03-tool-system.md) | **Tool System**: Tool invocation, permissions, and dynamic disclosure |
| Chapter 4 | [04-todo-task-system.md](04-todo-task-system.md) | **TodoWrite and Task System**: Task tracking and background tasks |
| Chapter 5 | [05-subagent-teams.md](05-subagent-teams.md) | **Subagent and Agent Teams**: Multi-agent orchestration protocol |
| Chapter 6 | [06-skills-deferred-loading.md](06-skills-deferred-loading.md) | **Skills System**: Deferred loading and on-demand search |
| Chapter 7 | [07-context-compaction.md](07-context-compaction.md) | **Context Compaction**: Auto compaction and memory restoration |
| Chapter 8 | [08-token-budget.md](08-token-budget.md) | **Token Budget**: Token budget and diminishing returns detection |
| Chapter 9 | [09-hooks.md](09-hooks.md) | **Hook Mechanism**: 27 event hooks and extension protocol |
| Chapter 10 | [10-worktree-isolation.md](10-worktree-isolation.md) | **Worktree Isolation**: Git Worktree and task sandboxing |
| Chapter 11 | [11-architecture.md](11-architecture.md) | **Architecture Overview**: Sequence diagrams, flowcharts, and module relationships |

---

## Key Source File Index

```
src/
├── query.ts                         # Agent Loop main entry point
├── query/
│   ├── tokenBudget.ts               # Token Budget logic
│   ├── config.ts                    # Query configuration snapshot
│   └── stopHooks.ts                 # Stop hook handling
├── utils/
│   ├── systemPrompt.ts              # buildEffectiveSystemPrompt()
│   ├── attachments.ts               # Dynamic attachments (agent listing, deferred tools)
│   ├── tokenBudget.ts               # Token budget parsing (user input +500k)
│   └── hooks.ts                     # Hook execution entry point
├── services/
│   ├── api/claude.ts                # Claude API streaming calls
│   ├── compact/
│   │   ├── compact.ts               # Main compaction engine
│   │   ├── autoCompact.ts           # Auto compaction trigger
│   │   └── microCompact.ts          # Micro compaction
│   └── tools/
│       ├── toolOrchestration.ts     # Tool execution orchestration
│       └── StreamingToolExecutor.ts # Streaming tool executor
├── tools/
│   ├── AgentTool/
│   │   ├── runAgent.ts              # Subagent startup
│   │   ├── forkSubagent.ts          # Fork subagent
│   │   └── loadAgentsDir.ts         # Agent definition loading
│   ├── TodoWriteTool/TodoWriteTool.ts
│   ├── SkillTool/SkillTool.ts
│   └── ToolSearchTool/ToolSearchTool.ts
├── skills/
│   └── loadSkillsDir.ts             # Skill directory scanning and loading
├── tasks/
│   ├── LocalAgentTask/              # Local agent task
│   ├── LocalMainSessionTask.ts      # Main session backgrounding
│   └── InProcessTeammateTask/       # In-process teammate task
├── types/
│   └── hooks.ts                     # Hook types and schema definitions
└── utils/
    ├── worktree.ts                  # Git Worktree management
    └── swarm/
        └── inProcessRunner.ts       # In-process subagent runner
```

---

## Core Design Philosophy

Claude Code's architecture reflects several design decisions worth studying:

1. **Prompt Cache first**: Fork Subagent message construction ensures byte-identical prefixes to maximize prompt cache hit rates.
2. **Diminishing returns detection**: Token Budget doesn't just check percentage thresholds — it also detects token growth across recent consecutive turns to prevent idle loops.
3. **Layered context restoration**: After compaction, context is restored separately for files (up to 5), skills (25K token budget), and MCP instructions.
4. **AsyncLocalStorage isolation**: Subagents achieve runtime context isolation via Node.js AsyncLocalStorage — no cross-process overhead required.
5. **Feature gate compile-time elimination**: Conditions like `feature('PROACTIVE')` are statically analyzed during Bun bundling; disabled branches are completely eliminated from the bundle.
6. **Hooks as extension points**: 27 Hook events cover every critical lifecycle node of the Agent, enabling external scripts to dynamically intervene.
