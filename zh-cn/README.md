# Claude Code 源码深度解析课程（中文版）

> 基于 Claude Code 真实源码的 Agent 工程拆解。所有内容均来自对源码的直接分析。

← [返回课程首页](../README.md) | [English Version](../en-us/README.md)

---

## 课程目录

| 章节 | 文件 | 核心主题 |
|------|------|----------|
| 第 0 章 | [00-overview.md](00-overview.md) | 课程总览与 Agent 工作流全景 |
| 第 1 章 | [01-agent-loop.md](01-agent-loop.md) | **Agent Loop**：主查询循环的核心引擎 |
| 第 2 章 | [02-系统提示的构建机制.md](02-系统提示的构建机制.md) | **全局 Context Prompt**：系统提示的构建与优先级 |
| 第 3 章 | [03-工具调用、权限与动态披露.md](03-工具调用、权限与动态披露.md) | **Tool System**：工具调用、权限与动态披露 |
| 第 4 章 | [04-todoWrite与任务系统.md](04-todoWrite与任务系统.md) | **TodoWrite 与任务系统**：任务追踪与后台任务 |
| 第 5 章 | [05-subagent与AgentTeams.md](05-subagent与AgentTeams.md) | **Subagent 与 Agent Teams**：多代理编排协议 |
| 第 6 章 | [06-Skill系统.md](06-Skill系统.md) | **Skills 技能系统**：Deferred 加载与按需搜索 |
| 第 7 章 | [07-上下文压缩与记忆恢复.md](07-上下文压缩与记忆恢复.md) | **Context Compaction**：自动压缩与记忆恢复 |
| 第 8 章 | [08-token-budget.md](08-token-budget.md) | **Token Budget**：令牌预算与递减收益检测 |
| 第 9 章 | [09-hooks.md](09-hooks.md) | **Hook 机制**：27 种事件钩子与扩展协议 |
| 第 10 章 | [10-worktree-isolation.md](10-worktree-isolation.md) | **Worktree 隔离**：Git Worktree 与任务沙箱 |
| 第 11 章 | [11-architecture.md](11-architecture.md) | **架构总图**：时序图、流程图与模块关系 |
