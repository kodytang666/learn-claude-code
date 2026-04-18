# 第 4 章：TodoWrite 与任务系统

> 源码位置：`src/tools/TodoWriteTool/`、`src/tasks/`、`src/utils/tasks.ts`

---

## 4.1 两个容易混淆的概念

Claude Code 中有两个相关但不同的"任务"概念：

| 概念 | 说明 | 存储位置 |
|------|------|----------|
| **Todo（任务清单）** | Claude 在当前会话中规划的工作步骤 | `AppState.todos` |
| **Task（系统任务）** | 运行时管理的实体（子代理、Shell 命令、后台会话等）| `AppState.tasks` |

本章先讲 TodoWrite（任务清单），再讲 Task System（系统任务）。

---

## 4.2 TodoWrite：Claude 的工作计划本

### 4.2.1 数据模型

```typescript
// src/utils/todo/types.ts
type TodoItem = {
  content: string                                    // 任务描述
  status: 'pending' | 'in_progress' | 'completed'
  activeForm: string                                 // 任务的主动形式（如 "Reading src/foo.ts"）
}

type TodoList = TodoItem[]
```

### 4.2.2 工具实现

[src/tools/TodoWriteTool/TodoWriteTool.ts:65-115](../../src/tools/TodoWriteTool/TodoWriteTool.ts#L65-L115)
```typescript
async call({ todos }, context) {
  const appState = context.getAppState()
  
  // 每个代理有自己的 todo 列表（用 agentId 或 sessionId 区分）
  const todoKey = context.agentId ?? getSessionId()
  const oldTodos = appState.todos[todoKey] ?? []
  
  // 关键逻辑：如果所有任务都已完成，AppState 中清空为 []
  const allDone = todos.every(_ => _.status === 'completed')
  const newTodos = allDone ? [] : todos

  // 验证提醒：仅当 allDone && 3+ 项 && 无验证步骤时才触发
  let verificationNudgeNeeded = false
  if (
    feature('VERIFICATION_AGENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
    !context.agentId &&   // 只在主线程触发，不在子代理
    allDone &&            // 必须是关闭列表的那一刻
    todos.length >= 3 &&
    !todos.some(t => /verif/i.test(t.content))
  ) {
    verificationNudgeNeeded = true
  }

  // 注意：API 是 setAppState（接收 prev → newState 函数），不是 updateAppState
  context.setAppState(prev => ({
    ...prev,
    todos: { ...prev.todos, [todoKey]: newTodos },
  }))

  // 返回值中 newTodos 是原始输入 todos（非清空后的 []）
  return {
    data: { oldTodos, newTodos: todos, verificationNudgeNeeded },
  }
}
```

### 4.2.3 关键设计点

**1. 所有完成即清空**

当所有 todo 都标记为 `completed`，AppState 中的列表被清空为 `[]` 而不是保留所有已完成项。这避免了 token 浪费（每次 API 调用都要序列化已完成的任务）。注意：工具返回值中的 `newTodos` 字段仍为原始 `todos` 输入，清空仅影响 AppState 存储层。

**2. agentId 隔离**

每个子代理有独立的 todo 列表（通过 `agentId` 作为 key）。主线程用 `sessionId`。这样并行运行的子代理不会互相干扰。

**3. shouldDefer: true**

TodoWrite 被标记为延迟披露，不在每次 API 请求中直接包含其 Schema。Claude 通过 ToolSearch 找到它。

**4. 与 TodoV2（Task API）互斥**

[src/tools/TodoWriteTool/TodoWriteTool.ts:53](../../src/tools/TodoWriteTool/TodoWriteTool.ts#L53) 中：

```typescript
isEnabled() {
  return !isTodoV2Enabled()
}
```

[src/utils/tasks.ts:133](../../src/utils/tasks.ts#L133) 中 `isTodoV2Enabled()` 在以下情况返回 `true`（即 TodoWrite 被禁用）：
- 环境变量 `CLAUDE_CODE_ENABLE_TASKS=1`
- 非交互式会话（SDK 模式）

这意味着 SDK 用户默认使用 Task API 而非 TodoWrite。

**5. verificationNudgeNeeded**

这是一个"行为引导"机制。触发条件（需同时满足）：
- Feature flag `VERIFICATION_AGENT` 已开启（编译时 gate）
- GrowthBook flag `tengu_hive_evidence` 为 true
- 当前在主线程（`!context.agentId`）
- **当前调用正在关闭整个列表（`allDone === true`）**
- 任务数 ≥ 3
- 没有任何任务内容匹配 `/verif/i`

满足时，工具结果会追加提示，要求 Claude 在写最终摘要前先 spawn 验证代理。

---

## 4.3 Task System：运行时实体管理

Task System 管理着 Claude Code 中所有的"运行中的实体"。

### 4.3.0 TaskStateBase：所有任务的公共基础

[src/Task.ts:45](../../src/Task.ts#L45)

```typescript
type TaskStateBase = {
  id: string           // 任务 ID（带类型前缀，见下表）
  type: TaskType       // 'local_bash' | 'local_agent' | 'remote_agent' | 'in_process_teammate' | 'local_workflow' | 'monitor_mcp' | 'dream'
  status: TaskStatus   // 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  description: string
  toolUseId?: string   // 触发此任务的 tool_use block ID（用于通知回调）
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string   // 磁盘输出文件路径（快照，/clear 后不变）
  outputOffset: number // 已读取字节偏移量（用于增量 delta 读取）
  notified: boolean    // 是否已发送完成通知（防止重复通知）
}
```

**Task ID 前缀表**（[src/Task.ts:79](../../src/Task.ts#L79)）：

| 类型 | 前缀 | 示例 |
|------|------|------|
| `local_bash` | `b` | `b3f9a2c1` |
| `local_agent` | `a` | `a7d8e4f2` |
| `remote_agent` | `r` | `r2a1b3c4` |
| `in_process_teammate` | `t` | `t5e6f7a8` |
| `local_workflow` | `w` | `w9b0c1d2` |
| `monitor_mcp` | `m` | `m3e4f5a6` |
| `dream` | `d` | `d7b8c9d0` |
| 主会话（特殊） | `s` | `s1a2b3c4` |

> `local_agent` 和主会话后台化共用 `a` 前缀（`LocalAgentTaskState`），但主会话后台化通过 `LocalMainSessionTask.ts` 中独立的 `generateMainSessionTaskId()` 使用 `s` 前缀，与普通子代理区分。

### 4.3.1 TaskState 联合类型

[src/tasks/types.ts](../../src/tasks/types.ts)

```typescript
// 注意：类型名是 TaskState，不是 Task
type TaskState =
  | LocalShellTaskState        // 长运行 Shell 命令
  | LocalAgentTaskState        // 本地子代理（含后台化主会话）
  | RemoteAgentTaskState       // 远程代理
  | InProcessTeammateTaskState // in-process 队友（团队协作）
  | LocalWorkflowTaskState     // 工作流
  | MonitorMcpTaskState        // MCP 监控
  | DreamTaskState             // 记忆整合子代理（auto-dream）
```

> **注意**：`LocalMainSessionTask` **不是**独立的联合类型成员。它是 `LocalAgentTaskState` 的子类型：
> ```typescript
> // src/tasks/LocalMainSessionTask.ts
> type LocalMainSessionTaskState = LocalAgentTaskState & { agentType: 'main-session' }
> ```
> 这意味着后台化主会话与普通子代理共用同一个 `type: 'local_agent'` 标识符，通过 `agentType` 字段区分。

### 4.3.2 主会话后台化（LocalMainSessionTask）

这是一个特殊功能：用户按 **Ctrl+B 两次**可以将当前对话"后台化"，然后开始新的对话。后台化后，原查询继续在后台独立运行。

[src/tasks/LocalMainSessionTask.ts](../../src/tasks/LocalMainSessionTask.ts)

```typescript
// LocalMainSessionTaskState 不是独立类型，而是 LocalAgentTaskState 的子类型
type LocalMainSessionTaskState = LocalAgentTaskState & {
  agentType: 'main-session'  // 唯一区分标识
}
// type: 'local_agent'（继承）
// taskId: 's' 前缀（区分普通代理的 'a' 前缀）
// messages?: Message[]（继承，存储后台查询的消息历史）
// isBackgrounded: boolean（继承，true = 后台，false = 前台展示中）
// 无独立 transcript 字段
```

关键函数：

```typescript
// 注册后台会话任务，返回 { taskId, abortSignal }
registerMainSessionTask(description, setAppState, agentDefinition?, abortController?)

// 启动真正的后台查询（wrap runWithAgentContext + query()）
startBackgroundSession({ messages, queryParams, description, setAppState })

// 将后台任务切回前台展示
foregroundMainSessionTask(taskId, setAppState): Message[]
```

**为什么需要隔离的转录路径？** 后台任务使用 `getAgentTranscriptPath(agentId)` 而不是主会话的路径。若用同一路径，`/clear` 会意外覆盖后台会话数据。后台任务通过 `initTaskOutputAsSymlink()` 创建软链接，`/clear` 重链接时不影响后台任务的历史记录。

### 4.3.3 本地代理任务（LocalAgentTask）

每个子代理对应一个 `LocalAgentTaskState`：

[src/tasks/LocalAgentTask/LocalAgentTask.tsx:116](../../src/tasks/LocalAgentTask/LocalAgentTask.tsx#L116)

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string          // 区分子类型，'main-session' = 后台化主会话
  model?: string
  abortController?: AbortController
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress   // 进度信息（包含 toolUseCount、tokenCount）
  retrieved: boolean         // 结果是否已被取回
  messages?: Message[]       // 代理的消息历史（UI 展示用）
  lastReportedToolCount: number
  lastReportedTokenCount: number
  isBackgrounded: boolean    // false = 前台展示中，true = 后台运行
  pendingMessages: string[]  // SendMessage 排队的消息，在工具轮边界处理
  retain: boolean            // UI 持有此任务（阻止驱逐）
  diskLoaded: boolean        // 是否已从磁盘加载 sidechain JSONL
  evictAfter?: number        // 驱逐时间戳（任务完成后设置）
}
```

> **注意**：文档中常见误写 `toolUseCount` 为 LocalAgentTaskState 的直接字段，实际它在 `progress.toolUseCount` 中。`status` 字段继承自 `TaskStateBase`（`'running' | 'completed' | 'failed' | 'stopped'`）。

进度追踪通过专门的辅助函数：

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx
createProgressTracker()           // 初始化 ProgressTracker（分别追踪 input/output tokens）
updateProgressFromMessage()       // 从 assistant message 累积 token 和工具调用数
getProgressUpdate()               // 生成 AgentProgress 快照（供 UI 消费）
createActivityDescriptionResolver() // 通过 tool.getActivityDescription() 生成人类可读描述
```

**Token 追踪的精妙设计**：`ProgressTracker` 分开存储 `latestInputTokens`（Claude API 累积值，取最新）和 `cumulativeOutputTokens`（逐轮累加），避免重复计数。

### 4.3.4 in-process 队友任务（InProcessTeammateTask）

这是 Agent Teams 的核心数据结构：

[src/tasks/InProcessTeammateTask/types.ts](../../src/tasks/InProcessTeammateTask/types.ts)

```typescript
type TeammateIdentity = {
  agentId: string        // e.g., "researcher@my-team"
  agentName: string      // e.g., "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string  // Leader 的 sessionId
}

type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity        // 队友身份（存储在 AppState 中的 plain data）
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController         // 终止整个队友
  currentWorkAbortController?: AbortController  // 终止当前轮次
  awaitingPlanApproval: boolean             // 是否等待 plan 审批
  permissionMode: PermissionMode            // 独立权限模式（Shift+Tab 切换）
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  messages?: Message[]              // UI 展示用（上限 50 条，TEAMMATE_MESSAGES_UI_CAP）
  pendingUserMessages: string[]     // 查看该队友时用户输入的队列消息
  isIdle: boolean                   // 是否处于空闲（等待 leader 指令）
  shutdownRequested: boolean        // 是否已请求关闭
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

> **常见误解**：`mailbox` 字段不存在于 `InProcessTeammateTaskState`。队友间的通信邮箱存储在运行时上下文 `teamContext.inProcessMailboxes`（AsyncLocalStorage 中），不在 AppState 里。`pendingUserMessages` 是用户从 UI 发给该队友的消息队列，与邮箱是两回事。

**内存上限设计**：`messages` 字段上限 50 条（`TEAMMATE_MESSAGES_UI_CAP`），超出后从头部截断。原因是生产环境出现过单个 whale session 启动 292 个 agent、内存达 36.8GB 的情况，根本原因正是此字段持有第二份完整消息副本。

### 4.3.5 记忆整合任务（DreamTask）

[src/tasks/DreamTask/DreamTask.ts](../../src/tasks/DreamTask/DreamTask.ts)

```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'    // starting → updating（首个 Edit/Write 后翻转）
  sessionsReviewing: number          // 正在整理的会话数量
  filesTouched: string[]             // 被 Edit/Write 触碰的文件（不完整，仅 pattern-match 到的）
  turns: DreamTurn[]                 // assistant 轮次（工具调用折叠为计数）
  abortController?: AbortController
  priorMtime: number                 // 用于 kill 时回滚 consolidationLock 时间戳
}
```

DreamTask 是"auto-dream"记忆整合的 UI 表面层。它不改变子代理运行逻辑，只是让原本不可见的 fork agent 在 footer pill 和 Shift+Down 对话框中可见。子代理按 4 阶段 prompt 运行（orient → gather → consolidate → prune），但 DreamTask 不解析阶段，只通过工具调用类型推断 phase。

---

## 4.4 任务注册与生命周期框架

[src/utils/task/framework.ts](../../src/utils/task/framework.ts)

### 4.4.1 registerTask：注册与恢复

```typescript
registerTask(task: TaskState, setAppState): void
```

注册一个新任务时，有两条路径：

**新建**（`existing === undefined`）：
1. 将 task 写入 `AppState.tasks[task.id]`
2. 向 SDK 事件队列发出 `task_started` 事件

**恢复/替换**（`existing !== undefined`，如 `resumeAgentBackground`）：
1. 合并保留以下 UI 状态（避免用户正在查看的面板闪烁）：
   - `retain`：UI 持有标记
   - `startTime`：面板排序稳定性
   - `messages`：用户刚发送的消息还未落盘
   - `diskLoaded`：避免重复加载 sidechain JSONL
   - `pendingMessages`：待处理消息队列
2. **不**发出 `task_started`（防止 SDK 重复计数）

### 4.4.2 任务完成通知（XML 格式）

子代理/后台任务完成时，通过 `enqueuePendingNotification` 将 XML 推入消息队列：

```xml
<task_notification>
  <task_id>a7d8e4f2</task_id>
  <tool_use_id>toolu_01xxx</tool_use_id>  <!-- 可选 -->
  <output_file>/tmp/.../tasks/a7d8e4f2.output</output_file>
  <status>completed</status>
  <summary>Task "修复登录 bug" completed successfully</summary>
</task_notification>
```

这段 XML 在下一轮 API 调用前作为 `user` 消息被注入 messages，使 LLM 感知到后台任务完成。

### 4.4.3 evictTerminalTask：两级驱逐

任务完成后并不立即从 `AppState.tasks` 中删除，而是两级驱逐：

```
任务完成 → status='completed' + notified=true
  │
  ├─ 如果 retain=true 或 evictAfter > Date.now()
  │    → 保留（UI 正在展示，等待 30s grace period 后驱逐）
  │
  └─ 否则 → 立即从 AppState.tasks 中删除（eagerly evict）
               作为保底，generateTaskAttachments() 也会在下次 poll 时驱逐
```

`PANEL_GRACE_MS = 30_000`（30秒）是 coordinator panel 中 agent 任务的展示宽限期，确保用户能看到结果后再消失。

---

## 4.5 后台 API 任务工具

用户（通过 Claude）可以创建和管理后台任务：

```typescript
// TaskCreateTool / TaskUpdateTool / TaskStopTool / TaskGetTool / TaskListTool
```

这些工具允许 Claude 自己创建和监控后台任务，实现真正的异步多任务处理。注意：这套 Task API 工具仅在 TodoV2 模式下启用（即 `isTodoV2Enabled() === true`），与 TodoWrite 互斥。

---

## 4.6 任务输出的磁盘管理

[src/utils/task/diskOutput.ts](../../src/utils/task/diskOutput.ts)

### 4.6.1 输出文件路径

```typescript
// 注意：路径不是 .claude/tasks/，而是项目临时目录下的会话子目录
getTaskOutputPath(taskId) → `{projectTempDir}/{sessionId}/tasks/{taskId}.output`
```

**为什么包含 sessionId？** 防止同一项目的并发 Claude Code 会话互相踩踏输出文件。路径在首次调用时被 memoize（`let _taskOutputDir`），`/clear` 触发 `regenerateSessionId()` 时不会重新计算，确保跨 `/clear` 存活的后台任务仍能找到自己的文件。

### 4.6.2 DiskTaskOutput 写队列

任务输出通过 `DiskTaskOutput` 类异步写入磁盘：

```typescript
class DiskTaskOutput {
  append(content: string): void  // 入队，自动触发 drain
  flush(): Promise<void>         // 等待队列清空
  cancel(): void                 // 丢弃队列（任务被 kill 时）
}
```

**核心设计要点**：

1. **写队列**：`#queue: string[]` 平铺数组，单个 drain 循环消费，chunk 写入后立即可被 GC，避免 `.then()` 链持有引用导致内存膨胀

2. **5GB 上限**：超限后追加截断标记并停止写入：
   ```
   [output truncated: exceeded 5GB disk cap]
   ```

3. **O_NOFOLLOW 安全**：Unix 上用 `O_NOFOLLOW` flag 打开文件，防止沙箱中的攻击者通过创建软链接将 Claude Code 写入任意宿主文件

4. **事务追踪**：所有 fire-and-forget 异步操作（`initTaskOutput`、`evictTaskOutput` 等）注册到 `_pendingOps: Set<Promise>`，测试可通过 `allSettled` 等待全部完成，防止跨测试的 ENOENT 竞争

### 4.6.3 增量输出读取（OutputOffset）

`TaskStateBase.outputOffset` 记录已消费的字节偏移，实现增量读取：

```typescript
// 仅读取 fromOffset 之后的新内容（最多 8MB）
getTaskOutputDelta(taskId, fromOffset): Promise<{ content: string; newOffset: number }>
```

`framework.ts` 中的 `generateTaskAttachments()` 在每次 poll 时调用 `getTaskOutputDelta`，将新增内容附加到 `task_status` attachment 中推送给 LLM，避免重复加载完整输出文件。

### 4.6.4 关键函数对比

| 函数 | 是否删磁盘文件 | 是否清内存 | 适用场景 |
|------|------|------|------|
| `evictTaskOutput(taskId)` | **否** | 是（flush 后清 Map）| 任务完成，结果已消费 |
| `cleanupTaskOutput(taskId)` | **是** | 是 | 彻底清理（测试、取消） |
| `flushTaskOutput(taskId)` | 否 | 否 | 读取前确保写入完成 |

---

## 4.7 Cron 定时任务

通过 Cron 工具，可以创建定期自动运行的任务：

```typescript
// CronCreate / CronDelete / CronList 工具
// 底层通过 src/utils/cron.ts 管理

// 使用示例（Claude 会这样调用）：
// CronCreate({ schedule: '0 9 * * 1', command: '/review-pr' })
// → 每周一早上 9 点自动运行 /review-pr
```

---

## 4.8 流程图：任务的完整生命周期

```
用户输入复杂任务
      │
      ▼
Claude 调用 TodoWrite         ← 规划工作步骤
  todos: [
    { content: '读取代码', status: 'pending' },
    { content: '修改功能', status: 'pending' },
    { content: '运行测试', status: 'pending' },
  ]
      │
      ▼
Claude 开始执行第一步
TodoWrite: { status: 'in_progress' }  ← 标记进行中
      │
      ├─ 如果需要并行工作：
      │    Agent({ subagent_type: 'general-purpose', ... })
      │         │
      │         ▼
      │    创建 LocalAgentTaskState（id: 'a-xxxxxxxx'，agentId 同值）
      │    子代理在独立上下文中运行
      │         │
      │         ▼
      │    子代理有自己的 Todo 列表（隔离）
      │
      ▼
每步完成后：
TodoWrite: { status: 'completed' }    ← 标记完成
      │
      ▼
所有步骤完成：
TodoWrite: todos.every(done) → 清空列表 []
      │
      ▼
Agent Loop 检测到无工具调用 → 停止
```

---

## 小结

| 组件 | 职责 | 源码位置 |
|------|------|----------|
| `TodoWriteTool` | 会话内工作计划追踪（交互模式） | [src/tools/TodoWriteTool/TodoWriteTool.ts](../../src/tools/TodoWriteTool/TodoWriteTool.ts) |
| Task API 工具 | 后台任务创建与管理（SDK/非交互模式） | `TaskCreateTool` / `TaskStopTool` 等 |
| `TaskStateBase` | 所有任务的公共字段（含 id、status、outputOffset） | [src/Task.ts](../../src/Task.ts) |
| `LocalMainSessionTask` | 主会话后台化（`LocalAgentTaskState` 子类型） | [src/tasks/LocalMainSessionTask.ts](../../src/tasks/LocalMainSessionTask.ts) |
| `LocalAgentTaskState` | 子代理生命周期管理 | [src/tasks/LocalAgentTask/LocalAgentTask.tsx](../../src/tasks/LocalAgentTask/LocalAgentTask.tsx) |
| `InProcessTeammateTaskState` | 团队队友状态（含内存上限 50 条） | [src/tasks/InProcessTeammateTask/types.ts](../../src/tasks/InProcessTeammateTask/types.ts) |
| `DreamTaskState` | 记忆整合子代理的 UI 表面层 | [src/tasks/DreamTask/DreamTask.ts](../../src/tasks/DreamTask/DreamTask.ts) |
| 任务框架 | registerTask / evictTerminalTask / 通知 XML | [src/utils/task/framework.ts](../../src/utils/task/framework.ts) |
| 磁盘输出管理 | DiskTaskOutput 写队列 / 增量读取 / 5GB 上限 | [src/utils/task/diskOutput.ts](../../src/utils/task/diskOutput.ts) |
| Cron 定时任务 | 自动化周期任务 | `CronCreate/Delete/List` 工具 |

**关键设计约定总结**：

| 约定 | 说明 |
|------|------|
| `LocalMainSessionTask` 不是独立类型 | 它是 `LocalAgentTaskState & { agentType: 'main-session' }` |
| TaskState（不是 Task）| 联合类型的正确名称 |
| `setAppState`（不是 `updateAppState`）| AppState 更新的实际 API |
| `toolUseCount` 在 `progress` 内 | 不是 `LocalAgentTaskState` 的直接字段 |
| TodoWrite 与 Task API 互斥 | 通过 `isTodoV2Enabled()` 在 `isEnabled()` 中切换 |
