# Claude Code 源码深度解析

> 基于 Claude Code 真实源码的 Agent 工程拆解。所有内容均来自对源码的直接分析。

---

## 文章定位

本文章面向两类读者：

- **入门者**：从未开发过 Agent 系统，希望通过一个工业级真实案例理解 Agent 的完整工作原理
- **进阶者**：已有 Agent 开发经验，希望深入理解 Claude Code 在 prompt cache、任务隔离、上下文压缩等方面的精妙设计

---

## Agent 工作流全景

Claude Code 启动后，一次完整的用户交互按照如下顺序流转：

```
用户输入
  │
  ▼
[全局 Context Prompt 构建]  ← CLAUDE.md / 记忆文件 / 环境信息
  │
  ▼
[Agent Loop 主循环]          ← query.ts
  │
  ├─→ [Claude API 流式调用]
  │
  ├─→ [工具执行]              ← Tool System（含动态披露）
  │      ├─ TodoWrite（任务追踪）
  │      ├─ Skill（按需加载技能）
  │      ├─ Agent（Subagent / Fork / Team）
  │      └─ Bash / FileRead / FileEdit ...
  │
  ├─→ [Token Budget 检查]    ← 是否继续还是停止？
  │
  ├─→ [Auto Compaction 检查] ← 上下文是否过长？
  │
  └─→ [Hook 执行]            ← 用户自定义扩展点
        ├─ PreToolUse / PostToolUse
        ├─ Stop / SubagentStart
        └─ PreCompact / PostCompact
```

---

## 文章目录

| 章节 | 文件 | 核心主题 |
|------|------|----------|
| 第 0 章 | [00-overview.md](00-overview.md) | 文章总览与 Agent 工作流全景 |
| 第 1 章 | [01-agent-loop.md](01-agent-loop.md) | **Agent Loop**：主查询循环的核心引擎 |
| 第 2 章 | [02-context-prompt.md](02-context-prompt.md) | **全局 Context Prompt**：系统提示的构建与优先级 |
| 第 3 章 | [03-tool-system.md](03-tool-system.md) | **Tool System**：工具调用、权限与动态披露 |
| 第 4 章 | [04-todo-task-system.md](04-todo-task-system.md) | **TodoWrite 与任务系统**：任务追踪与后台任务 |
| 第 5 章 | [05-subagent-teams.md](05-subagent-teams.md) | **Subagent 与 Agent Teams**：多代理编排协议 |
| 第 6 章 | [06-skills-deferred-loading.md](06-skills-deferred-loading.md) | **Skills 技能系统**：Deferred 加载与按需搜索 |
| 第 7 章 | [07-context-compaction.md](07-context-compaction.md) | **Context Compaction**：自动压缩与记忆恢复 |
| 第 8 章 | [08-token-budget.md](08-token-budget.md) | **Token Budget**：令牌预算与递减收益检测 |
| 第 9 章 | [09-hooks.md](09-hooks.md) | **Hook 机制**：18 种事件钩子与扩展协议 |
| 第 10 章 | [10-worktree-isolation.md](10-worktree-isolation.md) | **Worktree 隔离**：Git Worktree 与任务沙箱 |
| 第 11 章 | [11-architecture.md](11-architecture.md) | **架构总图**：时序图、流程图与模块关系 |

---

## 关键源码文件索引

```
src/
├── query.ts                         # Agent Loop 主循环入口
├── query/
│   ├── tokenBudget.ts               # Token Budget 逻辑
│   ├── config.ts                    # 查询配置快照
│   └── stopHooks.ts                 # 停止钩子处理
├── utils/
│   ├── systemPrompt.ts              # buildEffectiveSystemPrompt()
│   ├── attachments.ts               # 动态附件（代理列表、延迟工具）
│   ├── tokenBudget.ts               # 令牌预算解析（用户输入 +500k）
│   └── hooks.ts                     # Hook 执行入口
├── services/
│   ├── api/claude.ts                # Claude API 流式调用
│   ├── compact/
│   │   ├── compact.ts               # 主压缩引擎
│   │   ├── autoCompact.ts           # 自动压缩触发
│   │   └── microCompact.ts          # 微压缩
│   └── tools/
│       ├── toolOrchestration.ts     # 工具执行编排
│       └── StreamingToolExecutor.ts # 流式工具执行器
├── tools/
│   ├── AgentTool/
│   │   ├── runAgent.ts              # 子代理启动
│   │   ├── forkSubagent.ts          # Fork 子代理
│   │   └── loadAgentsDir.ts         # 代理定义加载
│   ├── TodoWriteTool/TodoWriteTool.ts
│   ├── SkillTool/SkillTool.ts
│   └── ToolSearchTool/ToolSearchTool.ts
├── skills/
│   └── loadSkillsDir.ts             # 技能目录扫描与加载
├── tasks/
│   ├── LocalAgentTask/              # 本地代理任务
│   ├── LocalMainSessionTask.ts      # 主会话后台化
│   └── InProcessTeammateTask/       # in-process 队友任务
├── types/
│   └── hooks.ts                     # Hook 类型与 Schema 定义
└── utils/
    ├── worktree.ts                  # Git Worktree 管理
    └── swarm/
        └── inProcessRunner.ts       # in-process 子代理运行器
```

---

## 核心设计哲学

Claude Code 的架构体现了几个值得深思的设计决策：

1. **Prompt Cache 优先**：Fork Subagent 的消息构建确保字节级相同前缀，最大化 prompt cache 命中率
2. **收益递减检测**：Token Budget 不只检查百分比阈值，还检测最近连续轮次的 token 增长量，防止无效循环
3. **分层上下文压缩**：压缩后分别为文件（5个上限）、技能（25K token 预算）、MCP 指示恢复 context
4. **AsyncLocalStorage 隔离**：子代理通过 Node.js AsyncLocalStorage 实现运行时上下文隔离，无需跨进程
5. **Feature Gate 编译时消除**：`feature('PROACTIVE')` 等条件在 Bun bundle 时静态分析，未开启的分支被完全消除
6. **Hook 即扩展点**：18 种 Hook 事件覆盖 Agent 生命周期的全部关键节点，支持外部脚本动态介入
