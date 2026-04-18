# 第 11 章：架构总图 — 时序图、流程图与模块关系

---

## 11.1 整体架构层次图

```
┌────────────────────────────────────────────────────────────────────┐
│                        Claude Code 架构                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐ │
│  │  CLI     │  │  IDE     │  │  Desktop │  │  claude.ai/code     │ │
│  │ (终端)    │  │  扩展    │  │  应用     │  │  (Web)              │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────┬──────────┘ │
│       └─────────────┴─────────────┴───────────────────┘            │
│                              │                                     │
│                    ┌─────────┴─────────┐                           │
│                    │    REPL / Bridge   │                          │
│                    │  (src/bridge/)     │                          │
│                    └─────────┬─────────┘                           │
│                              │                                     │
│  ┌───────────────────────────▼──────────────────────────────────┐  │
│  │                    Agent Loop 核心层                          │  │
│  │                    (src/query.ts)                            │  │
│  │                                                              │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  Context 构建                                            │ │  │
│  │  │  buildEffectiveSystemPrompt() + getAttachmentMessages() │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          │                                   │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  Claude API 调用                                         │ │  │
│  │  │  queryModelWithStreaming() (src/services/api/claude.ts) │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          │                                   │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  工具执行层                                               │ │  │
│  │  │  runTools() → StreamingToolExecutor → 51个 Tool 实现     │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          │                                   │  │
│  │  ┌──────────┐  ┌─────────────────┐  ┌──────────────────────┐ │  │
│  │  │ Token    │  │ Auto Compaction │  │  Hook System         │ │  │
│  │  │ Budget   │  │                 │  │  (27 种事件)          │ │  │
│  │  └──────────┘  └─────────────────┘  └──────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    扩展能力层                                 │   │
│  │                                                             │   │
│  │  ┌─────────────┐  ┌────────────┐  ┌──────────────────────┐  │   │
│  │  │  Skills     │  │ Subagent & │  │  Worktree            │  │   │
│  │  │  技能系统    │  │ Teams      │  │  任务隔离              │  │   │
│  │  └─────────────┘  └────────────┘  └──────────────────────┘  │   │
│  │                                                             │   │
│  │  ┌─────────────┐  ┌────────────┐                            │   │
│  │  │  Task System│  │  MCP       │                            │   │
│  │  │  任务管理    │  │  协议支持    │                            │   │
│  │  └─────────────┘  └────────────┘                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    基础设施层                                 │   │
│  │                                                             │   │
│  │  bootstrap/state.ts（全局状态）                               │   │
│  │  utils/（200+ 工具函数）                                      │   │
│  │  services/analytics/（分析上报）                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

---

## 11.2 Agent Loop 详细时序图

```
用户                Claude Code            Claude API         工具系统
 │                      │                      │                │
 │──── 输入提示 ────────▶│                      │                │
 │                      │                      │                │
 │              buildEffectiveSystemPrompt()   │                │
 │              getAttachmentMessages()        │                │
 │                      │                      │                │
 │              normalizeMessagesForAPI()      │                │
 │                      │                      │                │
 │                      │── 流式请求 ─────────▶ │                │
 │                      │                      │                │
 │◀─── 流式显示文本 ──────│◀── streaming ─────── │                │
 │                      │                      │                │
 │                      │◀── tool_use 块 ───── │                │
 │                      │                      │                │
 │                      │── checkPermissions() ────────────────▶│
 │                      │◀─ 权限结果 ────────────────────────────│
 │                      │                      │                │
 │◀─── 显示工具执行 ──────│── call() ────────────────────────────▶│
 │                      │◀─ 工具结果 ────────────────────────────│
 │                      │                      │                │
 │              checkTokenBudget()             │                │
 │              isAutoCompactEnabled()?        │                │
 │                      │                      │                │
 │                      │── 新请求（含工具结果）──▶│               │
 │                      │       [循环继续...]    │               │
 │                      │                       │               │
 │                      │◀── 停止响应 ───────────│                │
 │                      │                       │                │
 │              handleStopHooks()               │                │
 │                      │                       │                │
 │◀─── 任务完成 ─────────│                       │                │
```

---

## 11.3 Subagent 创建时序图

```
主代理 Agent Loop      AgentTool          子代理 Loop        Claude API
        │                  │                  │                  │
        │── tool_use ─────▶│                  │                  │
        │     (Agent工具)   │                  │                  │
        │                  │                  │                  │
        │              加载代理定义             │                  │
        │              (loadAgentsDir)        │                  │
        │                  │                  │                  │
        │          isForkSubagentEnabled()?   │                  │
        │                  │                  │                  │
        │    [Fork 模式]    │                  │                  │
        │         buildForkedMessages()       │                  │
        │         (构建字节相同的消息前缀)        │                  │
        │                  │                  │                  │
        │    [普通模式]     │                  │                  │
        │         使用指定系统提示               │                  │
        │                  │                  │                  │
        │              runWithAgentContext()  │                  │
        │              (AsyncLocalStorage 隔离)│                  │
        │                  │                  │                  │
        │                  │── 启动子代理 ────▶ │                  │
        │                  │   Loop           │                  │
        │                  │                  │──  子代理请求 ────▶│
        │                  │                  │◀─ 子代理响应 ──────│
        │                  │                  │   [子代理循环...]  │
        │                  │                  │                  │
        │◀─ yield 进度更新──│◀── 进度事件 ───────│                  │
        │                  │                  │                  │
        │                  │◀─ 子代理完成 ──────│                  │
        │                  │                  │                  │
        │◀─ tool_result ───│                  │                  │
        │   (子代理结果)     │                  │                  │
        │                  │                  │                  │
        │   [主代理继续...]  │                  │                  │
```

---

## 11.4 Context Compaction 时序图

```
Agent Loop          AutoCompact          Compact Engine        Claude API
    │                   │                     │                    │
    │  每轮工具执行完成    │                     │                    │
    │                   │                     │                    │
    │── calculateTokenWarningState() ─────────│                    │
    │◀─ 'critical' ───────────────────────────│                    │
    │                   │                     │                    │
    │── compactConversation() ───────────────▶│                    │
    │                   │                     │                    │
    │                   │        executePreCompactHooks()          │
    │                   │                     │                    │
    │                   │        stripImagesFromMessages()         │
    │                   │                     │                    │
    │                   │        groupMessagesByApiRound()         │
    │                   │                     │                    │
    │                   │        runForkedAgent(压缩提示) ──────────▶│
    │                   │                     │◀─ 摘要文本 ──────────│
    │                   │                     │                    │
    │                   │        createCompactBoundaryMessage()    │
    │                   │                     │                    │
    │                   │        buildPostCompactMessages()        │
    │                   │         ├─ 恢复文件（≤5个）                │
    │                   │         ├─ 恢复技能（≤25K）               │
    │                   │         └─ 恢复计划                       │
    │                   │                     │                    │
    │                   │        executePostCompactHooks()         │
    │                   │                     │                    │
    │                   │        notifyCompaction() (缓存失效)      │
    │                   │                     │                    │
    │◀── 压缩后的消息列表 ───────────────────────│                    │
    │                   │                     │                    │
    │  继续 Agent Loop   │                     │                    │
    │  使用压缩后的历史    │                     │                    │
```

---

## 11.5 Hook 执行时序图（PreToolUse 示例）

```
工具执行层         Hook 系统           外部 Shell 脚本      权限系统
    │                  │                     │                  │
    │── PreToolUse ───▶│                     │                  │
    │   事件触发        │                     │                  │
    │                  │                     │                  │
    │            getAllHooks('PreToolUse')   │                  │
    │            sortMatchersByPriority()    │                  │
    │                  │                     │                  │
    │            [对每个匹配的 hook:]          │                  │
    │                  │── 执行 Shell 命令 ──▶│                  │
    │                  │   (传入 JSON 事件数据)│                  │
    │                  │                     │  执行检查逻辑...   │
    │                  │◀─ stdout (JSON 响应) ─│                │
    │                  │                     │                  │
    │            解析响应:                    │                  │
    │            - continue: false?          │                  │
    │            - updatedInput?             │                  │
    │            - permissionDecision?       │                  │
    │                  │                     │                  │
    │◀── 权限结果 ───────│                    │                  │
    │   (allow/deny/   │                     │                  │
    │    modified input)│                    │                  │
    │                  │                     │                  │
    │  如果 allow:      │                     │                  │
    │── call(工具执行) ─▶│                    │                   │
```

---

## 11.6 Agent Teams 通信时序图

```
Leader（主代理）      In-Process Runner     Worker 代理       Mailbox 系统
      │                    │                   │                  │
      │── TeamCreate() ───▶│                   │                  │
      │                    │                   │                  │
      │            [为每个 worker:]             │                  │
      │            runWithAgentContext() ─────▶│                  │
      │            (AsyncLocalStorage 隔离)     │                  │
      │                    │                   │                  │
      │                    │──  runAgent() ──▶ │                  │
      │                    │                   │                  │
      │                    │                   │── 工具调用 ──────▶│
      │                    │                   │   需要权限        │
      │                    │                   │                  │
      │                    │◀─ 权限请求 ───────────────────────────│
      │                    │                   │                  │
      │◀─ 权限请求 ─────────│                   │                  │
      │   (冒泡给 leader)   │                   │                  │
      │                    │                   │                  │
      │  用户确认 或         │                   │                  │
      │  leader 决策        │                   │                  │
      │                    │                   │                  │
      │── 权限响应 ───────────────────────────────────────────────▶│
      │                    │                   │                  │
      │                    │◀─ 继续执行 ───────────────────────────│
      │                    │                   │                  │
      │◀─ 进度更新 ──────────│◀─ 进度事件 ────────│                  │
      │                    │                   │                  │
      │                    │◀─ Worker 完成 ─────│                  │
      │                    │   写入 Mailbox     │                  │
      │                    │                   │                  │
      │◀─ 所有 Worker 完成 ──│                   │                  │
```

---

## 11.7 模块依赖关系图

```
                   ┌─────────────────┐
                   │  bootstrap/     │
                   │  state.ts       │  ← 全局状态（最少依赖）
                   └────────┬────────┘
                            │ (所有模块依赖)
           ┌────────────────┼────────────────┐
           │                │                │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼───────┐
    │  utils/     │  │  services/  │  │  types/     │
    │  （工具函数 ）│  │  （核心服务 ）│  │  （类型定义）│
    └──────┬──────┘  └──────┬──────┘  └─────┬───────┘
           │                │               │
           └────────┬───────┘               │
                    │                       │
             ┌──────▼──────┐                │
             │  query.ts   │◀───────────────┘
             │ （主循环）    │
             └──────┬──────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
  ┌─────▼─────┐ ┌───▼───┐ ┌────▼────┐
  │  tools/   │ │skills/│ │tasks/   │
  │（51个工具） │ │（技能）│ │（任务）  │
  └─────┬─────┘ └───┬───┘ └────┬────┘
        │           │          │
        └───────────┴──────────┘
                    │
             ┌──────▼──────┐
             │  Claude API │
             └─────────────┘
```

---

## 11.8 数据流图：一次完整的工具调用

```
用户提示
    │
    ▼
[消息加入 messages 数组]
    │
    ▼
[buildEffectiveSystemPrompt]
  Override? → Coordinator? → Agent(Proactive)? → Agent? → Custom? → Default
  + Append
    │
    ▼
[getAttachmentMessages]
  + 记忆文件（CLAUDE.md）
  + 代理列表 Delta
  + 延迟工具 Delta
  + MCP 指示 Delta
    │
    ▼
[normalizeMessagesForAPI]
  过滤内部消息类型
  规范化格式
    │
    ▼
[queryModelWithStreaming]
  tools: [直接披露的工具]
  system: [systemPrompt]
  messages: [normalized messages]
    │
    ▼
[流式响应]
  text_blocks → yield → UI 显示
  tool_use_blocks → 收集
    │
    ▼
[runTools] (并发执行)
  for each tool_use:
    1. findToolByName()
    2. PreToolUse Hook
    3. checkPermissions()
       - allow → call()
       - deny → error result
       - ask → 等待用户
    4. call(input, context)
    5. applyToolResultBudget()
    6. PostToolUse Hook
    返回 tool_result
    │
    ▼
[是否继续？]
  无工具调用 → Stop Hooks → return
  maxTurns 达到 → return
  Token Budget 超限 → return
  Auto Compact → 压缩 → 继续
  else → 更新 messages → 下一轮
```

---

## 11.9 关键设计决策总结

### 决策 1：AsyncGenerator 作为主循环

**选择**：`async function* query()` 而非普通函数

**原因**：
- 流式响应可以通过 `yield` 实时转发给 UI
- 错误可以通过生成器的抛出机制传播
- 可以用 `using` 关键字自动清理资源
- 调用者可以通过 `.return()` 提前终止

### 决策 2：AsyncLocalStorage 进程内隔离

**选择**：AsyncLocalStorage 而非多进程或多线程

**原因**：
- 无进程通信开销
- 子代理可以直接访问父代理的内存结构（经过隔离的）
- 通过 `cloneFileStateCache()` 控制隔离粒度
- 权限同步通过 mailbox 而非 IPC 实现

### 决策 3：延迟技能和工具披露

**选择**：shouldDefer + ToolSearch 而非全量披露

**原因**：
- 50+ 工具的全量 Schema 消耗大量 token
- 大多数会话只用到少数工具
- Claude 通过 ToolSearch 按需发现工具，不降低能力

### 决策 4：收益递减检测

**选择**：不只检查百分比，还检测增量趋势

**原因**：
- 纯百分比检测无法发现"空转"循环
- 连续 2 次 delta 均 < 500 token 才判定为"空转"
- 要求 `continuationCount >= 3` 作为前置条件，避免刚开始就误判

### 决策 5：Prompt Cache 字节级一致

**选择**：Fork 子代理传递已渲染提示，不重新生成

**原因**：
- GrowthBook（功能开关）在 "cold→warm" 状态下可能返回不同值
- 重新生成系统提示可能破坏 Prompt Cache
- 传递父代理的已渲染字节确保字节级相同

### 决策 6：分层上下文恢复

**选择**：压缩后按优先级和预算恢复（每文件上限 5K、每技能上限 5K、技能总量 25K、整体上限 50K）

**原因**：
- 无限恢复会导致每次 compact 都消耗大量 token
- 技能文件头部最重要，截断比丢弃好
- 固定预算保证可预测的 token 消耗

---

## 11.10 文件量统计（Claude Code 源码规模）

| 模块 | 文件数 | 关键文件 |
|------|--------|----------|
| `src/tools/` | 51 个工具目录 | AgentTool, BashTool, TodoWriteTool, SkillTool... |
| `src/utils/` | 200+ 工具函数 | systemPrompt.ts, attachments.ts, hooks.ts... |
| `src/services/` | 核心服务 | api/claude.ts, compact/compact.ts... |
| `src/tasks/` | 8 种任务类型 | LocalAgentTask, InProcessTeammateTask... |
| `src/types/` | 类型定义 | hooks.ts, message.ts... |
| `src/hooks/` | React Hooks | 87个 UI hooks |
| `src/skills/` | 技能加载 | loadSkillsDir.ts, bundledSkills.ts |

---

这张架构全图覆盖了 Claude Code 的核心工作机制。每个章节深入剖析了具体模块——结合源码阅读效果更佳。
