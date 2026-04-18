# 第 5 章：Subagent 与 Agent Teams — 多代理编排协议

> 源码位置：`src/tools/AgentTool/`、`src/utils/swarm/`、`src/tasks/InProcessTeammateTask/`

---

## 5.1 多代理的三种模式

Claude Code 支持三种不同的多代理模式：

```
模式 1：普通 Subagent（指定类型）
  Agent({ subagent_type: 'general-purpose', prompt: '...' })
  → 创建预定义类型的子代理，独立运行

模式 2：Fork Subagent（继承上下文）
  Agent({ prompt: '...' })  // 不指定 subagent_type
  → Fork 当前会话，子代理继承父代理的完整上下文

模式 3：Agent Teams（团队协作）
  TeamCreate → 创建团队
  → 多个 in-process 队友并发运行
  → 通过 mailbox 通信和权限同步
```

---

## 5.2 普通 Subagent：runAgent()

### 5.2.1 入口

[src/tools/AgentTool/runAgent.ts:248](../../src/tools/AgentTool/runAgent.ts#L248)

```typescript
// 注意：是 async generator（function*），不是普通 async function
// 逐条 yield Message，调用方以 for await...of 消费
export async function* runAgent({
  agentDefinition,
  promptMessages,    // 初始消息（替代旧的 directive 字符串）
  toolUseContext,
  canUseTool,        // 工具权限检查函数（子代理独立实例）
  isAsync,           // 是否后台异步运行
  forkContextMessages,
  querySource,
  override,          // { systemPrompt?, userContext?, agentId?, abortController? }
  model,
  maxTurns,
  availableTools,    // 调用方预组装的工具池
  allowedTools,      // 显式工具白名单（替换所有父代理 allow 规则）
  useExactTools,     // Fork 专用：跳过 resolveAgentTools()，直接用 availableTools
  worktreePath,
  ...
}: { ... }): AsyncGenerator<Message, void>
```

**为什么是 async generator？** 子代理的 `query()` 也是 async generator，`runAgent` 直接 yield 转发其事件，使调用方能实时观察子代理进度（用于 TaskOutput 更新），无需等待子代理完成。

### 5.2.2 代理定义（AgentDefinition）

[src/tools/AgentTool/loadAgentsDir.ts:106](../../src/tools/AgentTool/loadAgentsDir.ts#L106)

代理定义有三种来源，共用 `BaseAgentDefinition` 基础字段：

```typescript
// 所有代理共用的基础字段
type BaseAgentDefinition = {
  agentType: string
  whenToUse: string
  tools?: string[]           // 工具白名单（'*' = 父代理全部工具）
  disallowedTools?: string[]
  model?: string             // 'inherit' = 继承父代理模型
  permissionMode?: PermissionMode
  maxTurns?: number
  skills?: string[]          // 预加载的 skill 名称列表
  mcpServers?: AgentMcpServerSpec[]  // 代理专属 MCP 服务器
  hooks?: HooksSettings
  isolation?: 'worktree' | 'remote' // 隔离模式：git worktree 或远程
  background?: boolean       // 总是以后台任务方式 spawn
  memory?: 'user' | 'project' | 'local'  // 持久记忆 scope
  initialPrompt?: string     // 预置在第一个 user turn 前的提示
  omitClaudeMd?: boolean     // 省略 CLAUDE.md（Explore/Plan 优化，节省 token）
  requiredMcpServers?: string[]  // 必须存在的 MCP 服务器（否则不可用）
  criticalSystemReminder_EXPERIMENTAL?: string  // 每轮注入的关键提醒
}

// 内置代理（源码定义）
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  // getSystemPrompt 需要 toolUseContext（访问 options 判断环境）
  getSystemPrompt: (params: { toolUseContext: Pick<ToolUseContext, 'options'> }) => string
}

// 自定义代理（从 .claude/agents/*.md 加载）
type CustomAgentDefinition = BaseAgentDefinition & {
  source: 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
  filename?: string
  getSystemPrompt: () => string  // 内容来自 markdown 文件 body
}

// 插件代理（从插件系统加载）
type PluginAgentDefinition = BaseAgentDefinition & {
  source: 'plugin'
  plugin: string   // 插件名
  getSystemPrompt: () => string
}

// 联合类型
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

**代理优先级与覆盖顺序**（[src/tools/AgentTool/loadAgentsDir.ts:196](../../src/tools/AgentTool/loadAgentsDir.ts#L196)）：

```
built-in → plugin → userSettings → projectSettings → flagSettings → policySettings
```

相同 `agentType` 时，后者覆盖前者。这意味着 managed（policySettings）代理可以覆盖所有用户自定义代理，plugin 代理可以覆盖 built-in。

**`omitClaudeMd` 设计背景**：Explore、Plan 等只读代理不需要提交/PR/lint 规范（来自 CLAUDE.md），主代理会解读它们的输出。关闭后每次 spawn 节省 5-15 Gtok，跨 3400万+ Explore spawn 有显著效果。

### 5.2.3 AsyncLocalStorage 上下文隔离

子代理通过 `AsyncLocalStorage` 实现进程内隔离：

```typescript
// src/utils/agentContext.ts
type SubagentContext = {
  agentId: string
  parentAgentId?: string
  // ...
}

// 子代理运行时，所有在此 async 调用链内的代码
// 都能通过 getAgentContext() 读到正确的 agentId
function runWithAgentContext<T>(
  context: AgentContext,
  fn: () => Promise<T>
): Promise<T>
```

这意味着：
- 子代理读取 `getSessionId()` 会得到自己的 ID
- 子代理的 Todo 列表与父代理隔离
- 子代理的工具调用权限检查使用子代理自己的上下文

---

## 5.3 Fork Subagent：继承父代理上下文

这是最精妙的设计之一，也是 Prompt Cache 优化的关键。

### 5.3.1 什么是 Fork？

[src/tools/AgentTool/forkSubagent.ts:32](../../src/tools/AgentTool/forkSubagent.ts#L32)

当 `subagent_type` 省略时，且 `isForkSubagentEnabled()` 返回 true 时触发 Fork 模式。子代理继承父代理的：
- 完整对话历史（字节级相同的消息前缀）
- 系统提示（通过 `override.systemPrompt` 传递已渲染字节，不重新构建）
- 工具列表（`tools: ['*']` + `useExactTools: true`，跳过 `resolveAgentTools()`）
- 权限模式（`permissionMode: 'bubble'`）

**`isForkSubagentEnabled()` 的三个条件**（需同时满足）：
1. `feature('FORK_SUBAGENT')` 编译时 gate 开启
2. `!isCoordinatorMode()` — 与 Coordinator 模式互斥（coordinator 有自己的编排模型）
3. `!getIsNonInteractiveSession()` — 仅在交互式会话中启用

### 5.3.2 FORK_AGENT 定义

[src/tools/AgentTool/forkSubagent.ts:60-71](../../src/tools/AgentTool/forkSubagent.ts#L60-L71)
```typescript
export const FORK_AGENT = {
  agentType: 'fork',  // analytics 标识
  whenToUse: 'Implicit fork — inherits full conversation context.',
  tools: ['*'],              // 继承父代理所有工具
  maxTurns: 200,
  model: 'inherit',          // 继承父代理模型
  permissionMode: 'bubble',  // 权限请求冒泡给父代理
  source: 'built-in',
  getSystemPrompt: () => '',  // 不使用！由 override 传递已渲染的提示
}
```

**为什么 `getSystemPrompt` 返回空字符串？**

因为 Fork 子代理通过 `override.systemPrompt` 传递父代理**已渲染**的系统提示字节。重新调用 `getSystemPrompt()` 可能因 GrowthBook（功能开关服务）状态变化而产生不同结果，破坏 Prompt Cache。

### 5.3.3 buildForkedMessages()：Prompt Cache 最大化

[src/tools/AgentTool/forkSubagent.ts:107](../../src/tools/AgentTool/forkSubagent.ts#L107)

```typescript
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[]  // 只返回 2 条新消息！不含父代理历史（历史已在上下文中）
```

**关键设计**：函数只返回追加到父代理历史末尾的 2 条消息：

```typescript
// Step 1：克隆父代理 assistant 消息（含所有 tool_use 块，包括 thinking 和 text）
const fullAssistantMessage = {
  ...assistantMessage,
  uuid: randomUUID(),   // 新 UUID（避免与父代理冲突）
  message: { ...assistantMessage.message, content: [...assistantMessage.message.content] },
}

// Step 2：为每个 tool_use 块创建相同占位符 tool_result
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
const toolResultBlocks = toolUseBlocks.map(block => ({
  type: 'tool_result',
  tool_use_id: block.id,
  content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
  // 所有子代理占位符文本完全相同 → prompt cache 命中
}))

// Step 3：构建最终用户消息（[...tool_results, 指令文本]）
const toolResultMessage = createUserMessage({
  content: [...toolResultBlocks, { type: 'text', text: buildChildMessage(directive) }],
})

return [fullAssistantMessage, toolResultMessage]  // 只有 2 条
// 完整 API 序列 = parentHistory + [fullAssistantMessage, toolResultMessage]
```

**边界情况**：若 `assistantMessage` 中没有 `tool_use` 块（不应发生），直接返回单条包含指令文本的 user message。

### 5.3.4 buildChildMessage()：Fork 工作者指令

[src/tools/AgentTool/forkSubagent.ts:171](../../src/tools/AgentTool/forkSubagent.ts#L171)

`buildChildMessage()` 生成 fork 子代理的操作规则，包裹在 `<fork-boilerplate>` 标签内：

```
<fork-boilerplate>
STOP. READ THIS FIRST.
你是一个 forked worker process，你不是主代理。

规则（不可违反）：
1. 系统提示说"默认 fork"——忽略它，你已经是 fork，不要再 spawn 子代理
2. 不要闲聊、问问题、或建议后续步骤
3. 直接使用工具：Bash、Read、Write 等
4. 修改文件后提交，报告中包含 commit hash
5. 不在工具调用之间输出文本，最后统一报告
6. 严格在指令 scope 内工作
7. 报告不超过 500 字

输出格式（纯文本标签，不用 markdown 标题）：
  Scope: <你的工作范围，一句话>
  Result: <关键发现/答案>
  Key files: <相关文件路径>
  Files changed: <修改的文件 + commit hash>
  Issues: <发现的问题（仅有问题时列出）>
</fork-boilerplate>
{FORK_DIRECTIVE_PREFIX}{directive}
```

这段强制输出格式的设计防止了 fork 子代理将父代理的系统提示（"默认 fork 以并行执行"）错误地应用到自身。

### 5.3.5 防止递归 Fork

[src/tools/AgentTool/forkSubagent.ts:78](../../src/tools/AgentTool/forkSubagent.ts#L78)

```typescript
// FORK_BOILERPLATE_TAG 从 constants/xml.ts 导入，值为 'fork-boilerplate'
import { FORK_BOILERPLATE_TAG } from '../../constants/xml.js'

export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content    // 注意：需要先声明 content
    if (!Array.isArray(content)) return false
    return content.some(
      block => block.type === 'text' && block.text.includes(`<${FORK_BOILERPLATE_TAG}>`)
    )
  })
}
```

**为什么 Fork 子代理仍保留 Agent 工具？** 为保证 API 请求前缀中工具定义字节完全一致（prompt cache 需要）。但在调用时通过 `isInForkChild()` 拒绝递归 fork，而不是从工具池中移除 Agent 工具。

### 5.3.6 Worktree 隔离的 Fork

[src/tools/AgentTool/forkSubagent.ts:205](../../src/tools/AgentTool/forkSubagent.ts#L205)

当代理定义中有 `isolation: 'worktree'` 时，fork 子代理在独立的 git worktree 中运行。此时会在指令中注入 `buildWorktreeNotice()`，告知子代理：

```
你继承了工作在 {parentCwd} 的父代理的对话上下文。
你在 {worktreeCwd} 的隔离 git worktree 中运行——
同一仓库、相同相对文件结构、独立 working copy。
上下文中的路径指向父代理目录；请翻译到你的 worktree 根目录。
如果父代理可能修改了文件，请在编辑前重新读取。
你的修改仅在此 worktree 中，不影响父代理文件。
```

---

## 5.4 Agent Teams：in-process 并发队友

### 5.4.1 架构概览

```
Leader（主代理）
    │
    ├─ TeamCreate({ teammates: ['worker-a', 'worker-b'] })
    │
    ├─ worker-a 在 in-process 中运行（InProcessTeammateTask）
    │    ├─ 独立的 AgentContext（AsyncLocalStorage）
    │    ├─ 独立的消息历史
    │    └─ 通过 mailbox 与 leader 通信
    │
    └─ worker-b 在 in-process 中运行（InProcessTeammateTask）
         ├─ 独立的 AgentContext
         ├─ 独立的消息历史
         └─ 通过 mailbox 与 leader 通信
```

### 5.4.2 In-process Runner（核心）

[src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts)

```typescript
// 注意：上下文是 runWithTeammateContext（不是 runWithAgentContext）
// TeammateContext 比 AgentContext 多了 teamName、mailbox、permissionMode 等字段

async function runInProcessTeammate(
  identity: TeammateIdentity,
  toolUseContext: ToolUseContext,
) {
  // 1. 克隆文件状态缓存（隔离文件读写状态，防止并发队友互相影响）
  cloneFileStateCache()

  // 2. 为队友创建独立的 AbortController（2 个！）
  const abortController = createAbortController()          // kill 整个队友
  const currentWorkAbortController = createAbortController() // 仅 abort 当前轮次

  // 3. 创建队友专用的 canUseTool 函数（权限路由，见下文）
  const canUseTool = createInProcessCanUseTool(identity, abortController)

  // 4. 在独立的 TeammateContext 中运行（AsyncLocalStorage 隔离）
  return runWithTeammateContext(teammateContext, async () => {
    // 5. 启动队友的 Agent Loop（runAgent 是 async generator）
    for await (const message of runAgent({ agentDefinition: teammate, ... })) {
      // 更新 AppState、写 sidechain 转录、检查 mailbox 消息等
    }
  })
}
```

### 5.4.3 Mailbox 通信机制

队友之间（包括与 leader）通过 "邮箱"（mailbox）进行异步通信：

```typescript
// src/utils/teammateMailbox.ts
readMailbox(teammateId: string): Message[]     // 读取收到的消息
writeToMailbox(teammateId: string, msg): void  // 发送消息

// 消息类型识别
isPermissionResponse(msg): boolean            // 权限批准/拒绝
isShutdownRequest(msg): boolean               // 关闭请求
createIdleNotification(): Message             // 队友完成通知
```

### 5.4.4 权限同步（Permission Sync）

[src/utils/swarm/inProcessRunner.ts:128](../../src/utils/swarm/inProcessRunner.ts#L128) — `createInProcessCanUseTool()`

当队友的工具调用需要用户确认（`behavior === 'ask'`）时，走以下路由：

```
Worker 调用工具
  │
  ├─ hasPermissionsToUseTool() 返回 'allow'/'deny' → 直接通过/拒绝
  │
  └─ 返回 'ask'（需要用户确认）
       │
       ├─【主路径】setToolUseConfirmQueue 可用（leader 的 TUI 在线）
       │    → 将权限请求推入 leader 的 ToolUseConfirm 队列
       │    → 显示带 worker badge（名称 + 颜色）的确认对话框
       │    → 等待用户操作，与 leader 自己的工具确认使用相同 UI
       │
       └─【备用路径】TUI bridge 不可用（headless / 后台模式）
            → 通过 mailbox 向 leader 发送权限请求
            → 以 500ms 间隔轮询（PERMISSION_POLL_INTERVAL_MS）
            → 等待 leader 响应后继续
```

**Bash 命令的预处理**：在发送给 leader 前，先用 classifier（`BASH_CLASSIFIER` feature gate）自动审批，若分类器批准则跳过 leader 确认。这与 leader 的处理不同（leader 是竞速，worker 是串行等待）。

```typescript
// leaderPermissionBridge.ts 提供的关键函数
getLeaderToolUseConfirmQueue()   // 获取 leader 的 ToolUseConfirm 状态设置函数
getLeaderSetToolPermissionContext() // 获取 leader 的权限上下文
```

---

## 5.5 Coordinator 模式：自主代理编排

[src/coordinator/coordinatorMode.ts](../../src/coordinator/coordinatorMode.ts)

Coordinator 模式专用于长时间自主运行的任务。通过环境变量开启，与 Fork Subagent 互斥。

```typescript
// 判断当前是否处于 coordinator 模式
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}

// 恢复会话时，自动匹配会话存储的模式（必要时翻转 env var）
export function matchSessionMode(sessionMode: 'coordinator' | 'normal' | undefined): string | undefined

// 向 coordinator 注入 worker 的可用工具列表（coordinator 据此决定分配什么能力给 worker）
export function getCoordinatorUserContext(scratchpadDir?: string): string
```

> **注意**：源码中**没有** `getCoordinatorSystemPrompt()` 函数。Coordinator 的系统提示来自 `buildEffectiveSystemPrompt()` 中的标准路径（coordinator 模式通过 `userContext` 注入工具列表，不是独立的 prompt 函数）。

**与 Fork 的互斥关系**：
```typescript
// forkSubagent.ts
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // ← 互斥
    ...
  }
}
```

**Worker 工具集限制**（`src/constants/tools.ts`）：

```typescript
// ASYNC_AGENT_ALLOWED_TOOLS 限制 worker 的工具池
// 不包含：TeamCreate/TeamDelete（高级编排工具）、SendMessage、SyntheticOutput
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
// Worker = ASYNC_AGENT_ALLOWED_TOOLS - INTERNAL_WORKER_TOOLS
```

---

## 5.6 时序图：Fork Subagent 的 Prompt Cache 优化

```
时间 →

父代理历史消息（共享 Prompt Cache）：
┌──────────────────────────────────────────┐
│  user: "完成以下三个任务..."              │
│  assistant: [tool_use: agent(task1)]      │
│             [tool_use: agent(task2)]      │
│             [tool_use: agent(task3)]      │
└──────────────────────────────────────────┘
                    │
    buildForkedMessages() 为每个任务创建子代理消息：
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
  子代理1消息:   子代理2消息:  子代理3消息:
  [...共享历史]  [...共享历史]  [...共享历史]
  [assistant]   [assistant]   [assistant]   ← 完全相同
  [user:        [user:        [user:
    placeholder]  placeholder]  placeholder] ← 完全相同
    task1指令]    task2指令]    task3指令]   ← 只有这行不同

  ↑ Prompt Cache 命中 ↑  ↑ 命中 ↑          ↑ 命中 ↑
  只有末尾指令行不同，前缀字节完全相同，最大化缓存命中
```

---

## 小结

| 机制 | 使用场景 | 关键特性 | 源码位置 |
|------|----------|----------|----------|
| 普通 Subagent | 指定类型的专业任务 | async generator；独立上下文、系统提示、工具集 | [src/tools/AgentTool/runAgent.ts](../../src/tools/AgentTool/runAgent.ts) |
| Fork Subagent | 并行执行相似任务 | 继承父代理上下文；`buildForkedMessages` 仅返回 2 条新消息；最大化 Prompt Cache | [src/tools/AgentTool/forkSubagent.ts](../../src/tools/AgentTool/forkSubagent.ts) |
| Agent Teams | 协作任务，需通信 | `runWithTeammateContext` 隔离；权限路由优先走 leader TUI | [src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts) |
| Coordinator 模式 | 自主长期任务 | 与 Fork 互斥；env var 控制；`matchSessionMode` 自动恢复 | [src/coordinator/coordinatorMode.ts](../../src/coordinator/coordinatorMode.ts) |
| AsyncLocalStorage | 进程内上下文隔离 | 无跨进程开销；Subagent 用 AgentContext，Teammate 用 TeammateContext | [src/utils/agentContext.ts](../../src/utils/agentContext.ts) |

**关键设计约定总结**：

| 约定 | 说明 |
|------|------|
| `runAgent` 是 `async function*` | 不是普通 async function；以 async generator yield Message |
| `buildForkedMessages` 返回 2 条 | 只返回追加的 2 条；完整序列 = parentHistory + 2条 |
| Fork 子代理保留 Agent 工具 | 为保证 prompt cache，不移除工具；通过 `isInForkChild()` 在调用时阻断 |
| `getCoordinatorSystemPrompt()` 不存在 | Coordinator 通过 `getCoordinatorUserContext()` 注入工具列表 |
| 权限路由主路径是 leader TUI | mailbox 是 headless/后台的 fallback，不是主路径 |
| Teammate 用 `runWithTeammateContext` | 不是 `runWithAgentContext`；TeammateContext 含更多字段 |
