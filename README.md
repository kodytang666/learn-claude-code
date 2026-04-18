# Claude Code 源码深度解析文章

![封面](https://d3qyja3w7macx2.cloudfront.net/eyJidWNrZXQiOiJ1bXUuY29tIiwia2V5IjoicGljd2Vpa2UvdGVhY2hlci93ZWlrZS9FVmM5ZTQvMTc3NjQ5NTU4My42MzYuNTcyMzYuMjAwMjcyODUxLnBuZyIsImVkaXRzIjp7InJlc2l6ZSI6eyJmaXQiOiJjb3ZlciIsIndpZHRoIjo4MDB9LCJyb3RhdGUiOjB9fQ==)

> A deep-dive course into Claude Code's internals, built from direct source code analysis.
> 基于 Claude Code 真实源码的 Agent 工程拆解。所有内容均来自对源码的直接分析。



## Languages / 语言

| Language | Directory |
|----------|-----------|
| 中文 | [zh-cn/](zh-cn/) |
| English | [en-us/](en-us/) |

## Social

douyin ID： 286435703

juejin column： https://juejin.cn/column/7629597189515247667

## Usage / 使用方法

Clone Claude Code source code, then clone this repo into the project root — so you can jump to source locations while reading the articles.

```bash
# 1. Clone Claude Code source code
git clone claude-code codebase
cd claude-code

# 2. Clone this repo into the project root
git clone https://github.com/kodytang666/learn-claude-code.git
```

After this setup, file references in the articles (e.g. `src/query.ts`) can be opened directly in your IDE.


## Course Overview / 文章概览

This course targets two audiences:

- **Beginners**: New to Agent systems, want to understand how a production-grade Agent works end to end
- **Advanced readers**: Already building Agents, want to study Claude Code's specific design choices around prompt cache, task isolation, context compaction, and more

All content is derived from reading the actual Claude Code source code — no guesswork.


## Agent Workflow at a Glance

```
User Input
  │
  ▼
[Global Context Prompt]     ← CLAUDE.md / memory files / env info
  │
  ▼
[Agent Loop]                ← query.ts
  │
  ├─→ [Claude API streaming call]
  │
  ├─→ [Tool Execution]      ← Tool System (with deferred disclosure)
  │      ├─ TodoWrite (task tracking)
  │      ├─ Skill (on-demand skill loading)
  │      ├─ Agent (Subagent / Fork / Team)
  │      └─ Bash / FileRead / FileEdit ...
  │
  ├─→ [Token Budget check]  ← continue or stop?
  │
  ├─→ [Auto Compaction]     ← is context too long?
  │
  └─→ [Hook execution]      ← user-defined extension points
        ├─ PreToolUse / PostToolUse
        ├─ Stop / SubagentStart
        └─ PreCompact / PostCompact
```


## Table of Contents

| Chapter | File (zh-cn) | File (en-us) | Topic |
||-|-|-|
| 0 | [00-overview.md](zh-cn/00-overview.md) | [00-overview.md](en-us/00-overview.md) | Course overview & Agent workflow |
| 1 | [01-agent-loop.md](zh-cn/01-agent-loop.md) | [01-agent-loop.md](en-us/01-agent-loop.md) | **Agent Loop**: the core query engine |
| 2 | [02-系统提示的构建机制.md](zh-cn/02-系统提示的构建机制.md) | [02-system-prompt-construction.md](en-us/02-system-prompt-construction.md) | **System Prompt**: construction & priority |
| 3 | [03-工具调用、权限与动态披露.md](zh-cn/03-工具调用、权限与动态披露.md) | [03-tool-system.md](en-us/03-tool-system.md) | **Tool System**: calls, permissions & deferred disclosure |
| 4 | [04-todoWrite与任务系统.md](zh-cn/04-todoWrite与任务系统.md) | [04-todo-task-system.md](en-us/04-todo-task-system.md) | **TodoWrite & Task System**: tracking & background tasks |
| 5 | [05-subagent与AgentTeams.md](zh-cn/05-subagent与AgentTeams.md) | [05-subagent-teams.md](en-us/05-subagent-teams.md) | **Subagent & Agent Teams**: multi-agent orchestration |
| 6 | [06-Skill系统.md](zh-cn/06-Skill系统.md) | [06-skills-deferred-loading.md](en-us/06-skills-deferred-loading.md) | **Skills System**: deferred loading & on-demand search |
| 7 | [07-上下文压缩与记忆恢复.md](zh-cn/07-上下文压缩与记忆恢复.md) | [07-context-compaction.md](en-us/07-context-compaction.md) | **Context Compaction**: auto-compact & memory restoration |
| 8 | [08-token-budget.md](zh-cn/08-token-budget.md) | [08-token-budget.md](en-us/08-token-budget.md) | **Token Budget**: budget tracking & diminishing-returns detection |
| 9 | [09-hooks.md](zh-cn/09-hooks.md) | [09-hooks.md](en-us/09-hooks.md) | **Hook System**: 27 lifecycle events & extension protocol |
| 10 | [10-worktree-isolation.md](zh-cn/10-worktree-isolation.md) | [10-worktree-isolation.md](en-us/10-worktree-isolation.md) | **Worktree Isolation**: Git Worktree sandboxing |
| 11 | [11-architecture.md](zh-cn/11-architecture.md) | [11-architecture.md](en-us/11-architecture.md) | **Architecture Overview**: sequence diagrams, flowcharts & module map |


## Key Source Files

```
src/
├── query.ts                         # Agent Loop main entry point
├── query/
│   ├── tokenBudget.ts               # Token Budget logic
│   ├── config.ts                    # Query config snapshot
│   └── stopHooks.ts                 # Stop hook handling
├── utils/
│   ├── systemPrompt.ts              # buildEffectiveSystemPrompt()
│   ├── attachments.ts               # Dynamic attachments (agent list, deferred tools)
│   ├── tokenBudget.ts               # Token budget parsing (user input +500k)
│   └── hooks.ts                     # Hook execution entry point
├── services/
│   ├── api/claude.ts                # Claude API streaming call
│   ├── compact/
│   │   ├── compact.ts               # Main compaction engine
│   │   ├── autoCompact.ts           # Auto-compaction trigger
│   │   └── microCompact.ts          # Micro-compaction
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
│   └── loadSkillsDir.ts             # Skill directory scan & loading
├── tasks/
│   ├── LocalAgentTask/              # Local agent task
│   ├── LocalMainSessionTask.ts      # Main session background mode
│   └── InProcessTeammateTask/       # In-process teammate task
├── types/
│   └── hooks.ts                     # Hook types & schema definitions
└── utils/
    ├── worktree.ts                  # Git Worktree management
    └── swarm/
        └── inProcessRunner.ts       # In-process subagent runner
```

## Core Design Principles

Claude Code's architecture reflects several deliberate design decisions worth studying:

1. **Prompt Cache First**: Fork Subagent message construction ensures byte-identical prefixes to maximize prompt cache hit rate
2. **Diminishing-Returns Detection**: Token Budget doesn't just check percentage thresholds — it detects when consecutive rounds produce < 500 new tokens, preventing idle loops
3. **Layered Context Restoration**: After compaction, files (≤5, 5K each), skills (25K total budget), and MCP instructions are restored in priority order within a 50K overall limit
4. **AsyncLocalStorage Isolation**: Subagents achieve runtime context isolation via Node.js AsyncLocalStorage — no cross-process overhead
5. **Compile-Time Feature Elimination**: `feature('PROACTIVE')` and similar gates are statically analyzed at Bun bundle time; disabled branches are completely eliminated
6. **Hooks as Extension Points**: 27 Hook events cover every critical lifecycle node of the Agent, supporting dynamic intervention via external scripts
