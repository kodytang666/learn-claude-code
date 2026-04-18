# Claude Code 源码深度解析文章（中文版）

> 基于 Claude Code 真实源码的 Agent 工程拆解。所有内容均来自对源码的直接分析。

← [返回文章首页](../README.md) | [English Version](../en-us/README.md)


## 社交账号

douyin ID： 286435703

juejin column： https://juejin.cn/column/7629597189515247667

## 使用方法

Clone Claude Code 源码，然后在项目根目录下 clone 本仓库，即可在阅读文章时直接跳转到源码位置。

```bash
# 1. Clone Claude Code 源码
git clone claude-code codebase
cd claude-code

# 2. 在项目根目录下 clone 本仓库
git clone https://github.com/kodytang666/learn-claude-code.git
```

完成后，文章中引用的源码路径（如 `src/query.ts`）即可在 IDE 中直接跳转打开。

## 文章目录

| 章节 | 文件 | 核心主题 |
|-----|------|--------|
| 第 0 章 | [00-overview](00-overview.md) | 文章总览与 Agent 工作流全景 |
| 第 1 章 | [01-agent-loop](01-agent-loop.md) | **Agent Loop**：主查询循环的核心引擎 |
| 第 2 章 | [02-系统提示的构建机制](02-系统提示的构建机制.md) | **全局 Context Prompt**：系统提示的构建与优先级 |
| 第 3 章 | [03-工具调用、权限与动态披露](03-工具调用、权限与动态披露.md) | **Tool System**：工具调用、权限与动态披露 |
| 第 4 章 | [04-todoWrite与任务系统](04-todoWrite与任务系统.md) | **TodoWrite 与任务系统**：任务追踪与后台任务 |
| 第 5 章 | [05-subagent与AgentTeams](05-subagent与AgentTeams.md) | **Subagent 与 Agent Teams**：多代理编排协议 |
| 第 6 章 | [06-Skill系统.md](06-Skill系统.md) | **Skills 技能系统**：Deferred 加载与按需搜索 |
| 第 7 章 | [07-上下文压缩与记忆恢复](07-上下文压缩与记忆恢复.md) | **Context Compaction**：自动压缩与记忆恢复 |
| 第 8 章 | [08-token-budget](08-token-budget.md) | **Token Budget**：令牌预算与递减收益检测 |
| 第 9 章 | [09-hooks](09-hooks.md) | **Hook 机制**：27 种事件钩子与扩展协议 |
| 第 10 章 | [10-worktree-isolation](10-worktree-isolation.md) | **Worktree 隔离**：Git Worktree 与任务沙箱 |
| 第 11 章 | [11-architecture](11-architecture.md) | **架构总图**：时序图、流程图与模块关系 |
