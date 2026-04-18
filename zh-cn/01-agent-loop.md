# 第 1 章：Agent Loop — 主查询循环

> 源码位置：`src/query.ts`（主入口）、`src/query/` 目录（子模块）

---

## 1.1 什么是 Agent Loop？

Agent Loop 是 Claude Code 的心脏。它是一个**无限循环的异步生成器**，负责：

1. 将用户消息发送给 Claude API（流式调用）
2. 接收 Claude 的响应（文本 + 工具调用）
3. 执行工具调用，把结果喂回给 Claude
4. 判断是否继续循环（或停止）
5. 处理上下文过长、Token 超限等边界情况

简单理解：Agent Loop 就是 "Claude 思考 → 行动 → 观察 → 再思考" 的引擎。

---

## 1.2 核心数据结构

### QueryParams（查询参数）

[src/query.ts:181-199](../../src/query.ts#L181-L199)
```typescript
export type QueryParams = {
  messages: Message[]          // 对话历史
  systemPrompt: SystemPrompt   // 系统提示
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn     // 工具权限检查函数
  toolUseContext: ToolUseContext
  fallbackModel?: string       // 降级模型（用于错误恢复）
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number            // 最大轮次限制
  skipCacheWrite?: boolean
  taskBudget?: { total: number }  // API 级任务预算
  deps?: QueryDeps             // 依赖注入（便于测试）
}
```

### State（循环内部状态）

[src/query.ts:204-217](../../src/query.ts#L204-L217)
```typescript
type State = {
  messages: Message[]          // 随每轮更新的消息列表
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number  // max_output_tokens 错误恢复计数
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number            // 当前循环轮次
  transition: Continue | undefined  // 上一轮的继续原因（用于测试断言）
}
```

---

## 1.3 循环结构概览

```
query()  ← 公开入口（AsyncGenerator）
  └─ queryLoop()  ← 真正的循环逻辑
       ├─ [循环外] startRelevantMemoryPrefetch()   ← 整个 user turn 只触发一次
       └─ while(true) {
            1.  启动 skill prefetch（与 API 调用并行）
            2.  yield 'stream_request_start'         ← 通知 UI 请求开始
            3.  初始化 query chain tracking
            ─── 消息准备（5 层，全在 API 调用前）───
            4.  getMessagesAfterCompactBoundary()    ← 只取 compact 边界后的消息
            5.  applyToolResultBudget()              ← 限制工具结果大小
            6.  HISTORY_SNIP（feature gate）         ← 裁剪历史
            7.  microcompact                         ← 微压缩
            8.  CONTEXT_COLLAPSE（feature gate）     ← 折叠上下文
            9.  autocompact                          ← 自动压缩
           10.  blocking limit check                 ← 超限则直接返回错误
            ─── API 调用 ────────────────────────────
           11.  流式调用 Claude API
                └─ prependUserContext() 在调用时注入 userContext
           ─── 工具执行 ─────────────────────────────
           12.  无 tool_use? → handleStopHooks() → return terminal
           13.  runTools()                           ← 执行工具调用
            ─── 工具执行后 ──────────────────────────
           14.  getAttachmentMessages()              ← 注入 queued commands 等动态上下文
           15.  消费 memory prefetch 结果
           16.  消费 skill discovery 结果
           17.  refreshTools()                       ← 刷新 MCP tools（新连接的服务器）
            ─── 停止 / 继续判断 ─────────────────────
           18.  maxTurns 达到? → return terminal
           19.  更新 state，进入下一轮
          }
```

---

## 1.4 主循环详解

```typescript
async function* queryLoop(params: QueryParams) {
  // 不变量：循环中不会被重新赋值
  const { systemPrompt, userContext, systemContext, canUseTool, maxTurns } = params

  // 可变状态。每轮通过整体替换 state 对象而非逐字段赋值来更新
  let state: State = {
    messages: params.messages,
    turnCount: 1,
    // ...
  }

  // Token Budget Tracker（TOKEN_BUDGET feature gate）
  const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null

  // 记忆预加载：在 while 循环外声明，整个 user turn 只触发一次
  // using 关键字保证生成器退出时（含异常）自动释放资源
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(   // [src/query.ts:301](../../src/query.ts#L301)
    state.messages,
    state.toolUseContext,
  )

  while (true) {
    const { messages, toolUseContext, turnCount } = state

    // ── 每轮启动：skill prefetch 与 API 调用并行 ──────────────────
    const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(...)  // [src/query.ts:331](../../src/query.ts#L331)

    yield { type: 'stream_request_start' }  // 通知 UI 本轮请求开始

    // query chain tracking：维护跨轮次的 chainId + depth，用于 analytics
    const queryTracking = { chainId: ..., depth: ... }

    // ── 消息准备（5 层，全在 API 调用前）─────────────────────────

    // 1. 只取 compact 边界之后的消息（compact 前的历史由摘要替代）
    let messagesForQuery = getMessagesAfterCompactBoundary(messages)  // [src/query.ts:365](../../src/query.ts#L365)

    // 2. 限制工具结果大小（超出 maxResultSizeChars 的结果被截断/替换）
    messagesForQuery = await applyToolResultBudget(messagesForQuery, ...)  // [src/query.ts:379](../../src/query.ts#L379)

    // 3. HISTORY_SNIP（feature gate）：裁剪过长历史
    if (feature('HISTORY_SNIP')) {
      messagesForQuery = snipModule.snipCompactIfNeeded(messagesForQuery).messages
    }

    // 4. microcompact：在 autocompact 之前运行微压缩
    messagesForQuery = await deps.microcompact(messagesForQuery, ...).messages  // [src/query.ts:414](../../src/query.ts#L414)

    // 5. CONTEXT_COLLAPSE（feature gate）：折叠可折叠的上下文
    if (feature('CONTEXT_COLLAPSE')) {
      messagesForQuery = await contextCollapse.applyCollapsesIfNeeded(...).messages
    }

    // 6. autocompact：上下文过长时压缩（在 API 调用前，非调用后）
    const { compactionResult } = await deps.autocompact(messagesForQuery, ...)  // [src/query.ts:454](../../src/query.ts#L454)

    // 7. blocking limit check：未开启自动 compact 时，超限则直接报错退出
    if (!compactionResult && isAtBlockingLimit(messagesForQuery)) {  // [src/query.ts:637](../../src/query.ts#L637)
      yield createAssistantAPIErrorMessage({ error: 'invalid_request' })
      return { reason: 'blocking_limit' }
    }

    // ── 流式调用 Claude API ───────────────────────────────────────

    // userContext（CLAUDE.md + 当前日期）在调用时通过 prependUserContext 注入
    // 不是 systemPrompt 的一部分，而是作为 messages 前缀传入
    for await (const message of deps.callModel({        // [src/query.ts:659](../../src/query.ts#L659)
      messages: prependUserContext(messagesForQuery, userContext),
      systemPrompt: fullSystemPrompt,
      tools,
      // ...
    })) {
      yield message  // 流式事件实时转发给 UI
    }

    // ── 工具执行 ─────────────────────────────────────────────────

    // 无 tool_use → Claude 已完成，执行 stop hooks 后退出
    if (toolUseBlocks.length === 0) {
      yield* handleStopHooks(...)
      return terminal
    }

    const toolResults = []
    for await (const result of runTools(assistantMessage, toolUseContext)) {
      yield result
      toolResults.push(result)
    }

    // ── 工具执行后：注入动态上下文 ───────────────────────────────

    // getAttachmentMessages 在工具执行之后调用，处理 queued commands 等
    for await (const attachment of getAttachmentMessages(null, toolUseContext, ...)) {  // [src/query.ts:1580](../../src/query.ts#L1580)
      yield attachment
      toolResults.push(attachment)
    }

    // memory prefetch 结果消费（如果已 settled）
    if (pendingMemoryPrefetch?.settledAt !== null) {
      // 追加 memory attachments 到 toolResults
    }

    // skill discovery prefetch 结果消费
    if (skillPrefetch && pendingSkillPrefetch) {
      // 追加 skill_discovery attachments 到 toolResults
    }

    // 刷新 MCP tools：让本轮内新连接的 MCP 服务器在下一轮生效
    if (toolUseContext.options.refreshTools) {   // [src/query.ts:1660](../../src/query.ts#L1660)
      const refreshedTools = toolUseContext.options.refreshTools()
      // 更新 toolUseContext.options.tools
    }

    // ── 停止 / 继续判断 ──────────────────────────────────────────

    // maxTurns 检查（注意：在工具执行后，不在工具执行前）
    if (maxTurns && nextTurnCount > maxTurns) {   // [src/query.ts:1705](../../src/query.ts#L1705)
      return { reason: 'max_turns' }
    }

    // 更新 state，进入下一轮
    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      turnCount: nextTurnCount,
      maxOutputTokensOverride: undefined,
      // ...
    }
  }
}
```

---

## 1.5 Feature Gate 模式

Claude Code 使用 Bun 的 `feature()` 函数实现**编译时功能开关**，未开启的功能在编译时不会被打包进来：

[src/query.ts:15-21](../../src/query.ts#L15-L21)
```typescript
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? (require('./services/compact/reactiveCompact.js') as ...)
  : null

const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? (require('./services/contextCollapse/index.js') as ...)
  : null
```

[src/query.ts:66-71](../../src/query.ts#L66-L71)
```typescript
const skillPrefetch = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? (require('./services/skillSearch/prefetch.js') as ...)
  : null
```

---

## 1.6 max_output_tokens 错误恢复

Claude API 有时会因输出 token 达到上限而中断，Agent Loop 有专门的恢复机制：

[src/query.ts:164-165](../../src/query.ts#L164-L165)
```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

function isWithheldMaxOutputTokens(msg): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

**恢复逻辑：**
1. 检测到 `max_output_tokens` 错误
2. 最多重试 3 次（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`）
3. 首先尝试 Reactive Compact（如果开启）
4. 若仍失败，报错给用户

**为什么要"Withhold"（扣留）这个错误消息？**

注释中解释：向 SDK 调用者（如 co-work/desktop）泄露中间错误会导致会话终止，恢复循环还在运行但没人在监听。所以在确认恢复循环是否能继续之前，扣留这个错误。

---

## 1.7 异步生成器的妙用

`query()` 是一个 `AsyncGenerator`，它能同时做到：

- **流式输出**：用 `yield` 把 API 流式事件实时转发给 UI
- **错误传播**：异常通过 `yield*` 自然传播
- **资源清理**：`using` 关键字（TC39 Explicit Resource Management）在生成器退出时自动释放资源

[src/query.ts:219-239](../../src/query.ts#L219-L239)
```typescript
export async function* query(params: QueryParams) {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  
  // 只有正常返回时才执行（throw 或 .return() 调用时跳过）
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

---

## 1.8 流程图

```
用户提交消息
      │
      ▼
  query() → queryLoop()
      │
      ├─ startRelevantMemoryPrefetch()   ← while 循环外，整个 user turn 只触发一次
      │
      ▼
  while(true) {  ←───────────────────────────────────────────────────────┐
      │                                                                  │
      ├─ startSkillDiscoveryPrefetch()   ← 每轮启动，与 API 调用并行        │
      ├─ yield 'stream_request_start'                                    │
      │                                                                  │
      │  ── 消息准备（API 调用前）──────────────────────────────────        │
      ├─ getMessagesAfterCompactBoundary()   ← 取 compact 边界后消息       │
      ├─ applyToolResultBudget()             ← 工具结果大小限制             │
      ├─ HISTORY_SNIP（feature gate）        ← 裁剪历史                    │
      ├─ microcompact                        ← 微压缩                     │
      ├─ CONTEXT_COLLAPSE（feature gate）    ← 折叠上下文                  │
      ├─ autocompact                         ← 自动压缩                   │
      ├─ blocking limit? ──YES──→ return error                           │
      │                                                                  │
      │  ── 流式 API 调用 ─────────────────────────────────────────       │
      ├─ callModel(prependUserContext(messages))  ← userContext 在此注入  │
      │     └─ yield 流式事件 ...                                         │
      │                                                                 │
      │  ── 工具执行 ──────────────────────────────────────────────       │
      ├─ 无 tool_use? ──YES──→ handleStopHooks() → return terminal       │
      ├─ runTools()                          ← 执行工具调用                │
      │     └─ yield 工具执行结果 ...                                      │
      │                                                                  │
      │  ── 工具执行后 ────────────────────────────────────────────        │
      ├─ getAttachmentMessages()             ← queued commands 等动态上下文
      ├─ 消费 memory prefetch 结果                                         │
      ├─ 消费 skill discovery 结果                                         │
      ├─ refreshTools()                      ← 刷新 MCP tools             │
      │                                                                   │
      │  ── 停止 / 继续 ───────────────────────────────────────────        │
      ├─ maxTurns 达到? ──YES──→ return terminal                          │
      │                                                                   │
      └─ 更新 state，继续循环 ──────────────────────────────────────────────┘
  }
```

---

## 小结

| 概念 | 实现方式 | 源码位置 |
|------|----------|----------|
| 主循环 | `async function* queryLoop()` | [src/query.ts:241](../../src/query.ts#L241) |
| 消息准备（5 层） | compact boundary → budget → snip → microcompact → autocompact | [src/query.ts:365-468](../../src/query.ts#L365-L468) |
| userContext 注入 | `prependUserContext(messagesForQuery, userContext)` | [src/query.ts:660](../../src/query.ts#L660) |
| 流式事件转发 | `yield event` | `src/query.ts` |
| 工具执行 | `runTools()` | `src/services/tools/toolOrchestration.ts` |
| 工具后动态上下文 | `getAttachmentMessages()` 在 runTools 之后调用 | [src/query.ts:1580](../../src/query.ts#L1580) |
| MCP tools 刷新 | `refreshTools()` 每轮工具执行后调用 | [src/query.ts:1660](../../src/query.ts#L1660) |
| 错误恢复 | `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` | [src/query.ts:164](../../src/query.ts#L164) |
| 功能开关 | `feature('FLAG_NAME')` | Bun bundle 编译时消除 |
| 资源管理 | `using` 关键字（memory prefetch 在循环外声明） | [src/query.ts:301](../../src/query.ts#L301) |
