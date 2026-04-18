# 第 8 章：Token Budget — 令牌预算与递减收益检测

> 源码位置：[src/query/tokenBudget.ts](../../src/query/tokenBudget.ts)、[src/utils/tokenBudget.ts](../../src/utils/tokenBudget.ts)、[src/bootstrap/state.ts](../../src/bootstrap/state.ts)、[src/screens/REPL.tsx](../../src/screens/REPL.tsx)

---

## 8.1 什么是 Token Budget？

Token Budget（令牌预算）是一个让用户控制 Claude 在单次任务中**最多消耗多少 output tokens** 的机制。

**使用场景**：
- 用户输入 `+500k` 表示允许消耗最多 50 万个 tokens
- 用户输入 `spend 2M tokens` 表示允许消耗最多 200 万个 tokens
- Claude Code 会在达到预算前自动停止，避免无限循环

**注意**：Token Budget 与 Auto Compaction 是**不同**的机制：
- Auto Compaction：上下文窗口快满了，压缩历史（输入 token）
- Token Budget：用户允许的最大 output token 消费上限，防止 agentic 会话失控运行

---

## 8.2 两个独立的 Budget 机制

| 层次 | 来源 | 说明 |
|------|------|------|
| **用户层 Token Budget** | 用户在提示中输入 `+500k` | 控制整个 agentic 会话的 output token 消耗上限，客户端计数 |
| **API 层 Task Budget** | `taskBudget.total` 参数 | Claude API 的 `output_config.task_budget` beta 功能，服务端计数 |

本章主要讲用户层 Token Budget。

---

## 8.3 用户输入语法解析

[src/utils/tokenBudget.ts:1-29](../../src/utils/tokenBudget.ts#L1-L29)

### 支持的语法

```typescript
// 简写形式（Shorthand）：必须在 prompt 开头或结尾，避免自然语言误匹配
const SHORTHAND_START_RE = /^\s*\+(\d+(?:\.\d+)?)\s*(k|m|b)\b/i  // 开头
const SHORTHAND_END_RE   = /\s\+(\d+(?:\.\d+)?)\s*(k|m|b)\s*[.!?]?\s*$/i  // 结尾

// 完整形式（Verbose）：可在 prompt 任意位置出现
const VERBOSE_RE = /\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b/i
```

支持的单位：`k`（千）、`m`（百万）、`b`（十亿），大小写不敏感，支持小数（如 `1.5M`）。

**语法示例**：

| 输入 | 解析结果 |
|------|---------|
| `+500k` | 500,000（必须在 prompt 开头）|
| `fix this bug +2M` | 2,000,000（结尾简写）|
| `spend 1.5M tokens` | 1,500,000（verbose，任意位置）|
| `use 500k tokens` | 500,000（verbose，任意位置）|
| `+2b` | 2,000,000,000（十亿）|

### parseTokenBudget 与 findTokenBudgetPositions

```typescript
// 解析并返回数字预算（null 表示无预算语法）
export function parseTokenBudget(text: string): number | null

// 返回所有预算语法在 text 中的位置（用于从 prompt 中移除语法，避免 Claude 看到）
export function findTokenBudgetPositions(
  text: string,
): Array<{ start: number; end: number }>
// 注：返回数组而非单个对象；不含 budget 值；无匹配时返回 [] 而非 null
```

---

## 8.4 预算解析的时机：REPL.tsx

[src/screens/REPL.tsx:2925-2928](../../src/screens/REPL.tsx#L2925-L2928)

预算解析**不在** `query.ts` 中，而是在用户按下 Enter 提交消息时，在 `REPL.tsx` 中完成：

```typescript
// 每次用户提交新消息时执行（src/screens/REPL.tsx）
if (feature('TOKEN_BUDGET')) {
  const parsedBudget = input ? parseTokenBudget(input) : null
  // 1. 设置 outputTokensAtTurnStart = 当前累计 output tokens（基线快照）
  // 2. 设置 currentTurnTokenBudget = parsedBudget ?? getCurrentTurnTokenBudget()
  //    注：如果新输入里没有预算语法，则沿用上一轮的预算
  // 3. 重置 budgetContinuationCount = 0
  snapshotOutputTokensForTurn(parsedBudget ?? getCurrentTurnTokenBudget())
}
```

`snapshotOutputTokensForTurn(budget)` 完成三件事：

```typescript
// src/bootstrap/state.ts:733-737
export function snapshotOutputTokensForTurn(budget: number | null): void {
  outputTokensAtTurnStart = getTotalOutputTokens()  // 快照当前累计值作为基线
  currentTurnTokenBudget = budget
  budgetContinuationCount = 0
}
```

这样 `getTurnOutputTokens()` 就能正确计算**这一轮新增**的 output tokens：

```typescript
export function getTurnOutputTokens(): number {
  return getTotalOutputTokens() - outputTokensAtTurnStart  // 相对增量，不是累计总量
}
```

**设计细节**：预算在 REPL 层设置，是为了让预算跨 agent loop 多次 `continue` 迭代持续有效，而不是每次 query 调用单独管理。

---

## 8.5 BudgetTracker：追踪连续轮次

[src/query/tokenBudget.ts:6-20](../../src/query/tokenBudget.ts#L6-L20)

```typescript
export type BudgetTracker = {
  continuationCount: number        // 已"继续"的次数（每次 continue 递增）
  lastDeltaTokens: number          // 上一次检查时的 token 增量
  lastGlobalTurnTokens: number     // 上一次检查时的累计 token 数（计算当前增量用）
  startedAt: number                // 会话开始时间戳（计算 durationMs 用）
}

export function createBudgetTracker(): BudgetTracker {
  return {
    continuationCount: 0,
    lastDeltaTokens: 0,
    lastGlobalTurnTokens: 0,
    startedAt: Date.now(),
  }
}
```

`BudgetTracker` 在每次 query loop 入口处创建（`feature('TOKEN_BUDGET') ? createBudgetTracker() : null`），跟踪从本次用户提交开始的所有 agent 轮次数据。

---

## 8.6 核心判断逻辑：checkTokenBudget()

[src/query/tokenBudget.ts:45-93](../../src/query/tokenBudget.ts#L45-L93)

```typescript
const COMPLETION_THRESHOLD = 0.9      // 90%
const DIMINISHING_THRESHOLD = 500     // 递减检测最小增量（500 tokens）

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,   // 子代理不受 budget 约束
  budget: number | null,         // 用户设置的预算（null = 无预算）
  globalTurnTokens: number,      // 本轮新增的 output token 总数（getTurnOutputTokens()）
): TokenBudgetDecision {

  // 规则 1：子代理 / 无预算 / 预算 ≤ 0 → 立即停止，不带完成事件
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  const turnTokens = globalTurnTokens
  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 规则 2：递减收益检测
  // 条件：continuationCount >= 3（已经继续了 3 次）
  //   AND 本轮增量 < 500 token
  //   AND 上一轮增量 < 500 token
  // 注：只检查最近两轮（当前 + 上一轮），不是"连续 3 轮全部 <500"
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // 规则 3：无递减 且 未超过 90% → 继续（发送 nudge 消息）
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++
    tracker.lastDeltaTokens = deltaSinceLastCheck
    tracker.lastGlobalTurnTokens = globalTurnTokens
    return {
      action: 'continue',
      nudgeMessage: getBudgetContinuationMessage(pct, turnTokens, budget),
      continuationCount: tracker.continuationCount,
      pct,
      turnTokens,
      budget,
    }
  }

  // 规则 4：有递减 或 已曾 continue 过（超过 90% 时） → 停止，带完成事件
  if (isDiminishing || tracker.continuationCount > 0) {
    return {
      action: 'stop',
      completionEvent: {
        continuationCount: tracker.continuationCount,
        pct,
        turnTokens,
        budget,
        diminishingReturns: isDiminishing,
        durationMs: Date.now() - tracker.startedAt,
      },
    }
  }

  // 规则 5：从未 continue 就超过 90%（第一轮直接超限） → 停止，无事件
  return { action: 'stop', completionEvent: null }
}
```

### TokenBudgetDecision 类型

```typescript
type ContinueDecision = {
  action: 'continue'
  nudgeMessage: string
  continuationCount: number
  pct: number
  turnTokens: number
  budget: number
}

type StopDecision = {
  action: 'stop'
  completionEvent: {
    continuationCount: number
    pct: number
    turnTokens: number
    budget: number
    diminishingReturns: boolean
    durationMs: number
  } | null   // null = 无事件（子代理/无预算/第一轮超限）
}
```

---

## 8.7 决策树

```
checkTokenBudget() 被调用
          │
          ▼
  agentId 存在 OR budget = null OR budget ≤ 0？
     │              │
    YES             NO
     │              │
stop（null事件）      ▼
              计算 deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens
              计算 pct（使用百分比）
                     │
                     ▼
              isDiminishing？
              （continuationCount≥3 AND 本轮delta<500 AND 上轮delta<500）
                │        │
               YES        NO
                │         │
                │         ▼
                │   turnTokens < budget × 90%？
                │      │         │
                │     YES        NO
                │      │         │
                │      ▼         │
                │   continue     │
                │   ─────────    │
                │   tracker.continuationCount++
                │   return nudgeMessage
                │               │
                └──────┬─────────┘
                       │
                       ▼
               continuationCount > 0？
                  │         │
                 YES        NO
                  │         │
           stop（有事件）  stop（null事件）
           含 diminishingReturns    （第一轮直接超限）
```

---

## 8.8 Nudge Message（继续提示消息）

[src/utils/tokenBudget.ts:66-73](../../src/utils/tokenBudget.ts#L66-L73)

```typescript
export function getBudgetContinuationMessage(
  pct: number,
  turnTokens: number,
  budget: number,
): string {
  const fmt = (n: number): string => new Intl.NumberFormat('en-US').format(n)
  return `Stopped at ${pct}% of token target (${fmt(turnTokens)} / ${fmt(budget)}). Keep working \u2014 do not summarize.`
}
```

**实际消息示例**：
```
Stopped at 45% of token target (225,000 / 500,000). Keep working — do not summarize.
```

**注意**：消息以 `"Stopped at X% of token target..."` 开头（不是 "You have consumed..."），且末尾明确指令 **"do not summarize"**，防止 Claude 在预算未耗尽时就开始总结收尾。

这条消息在 `query.ts` 中以 `isMeta: true` 注入到对话中（不显示在 UI，但发送给 Claude API）：

```typescript
// src/query.ts
if (decision.action === 'continue') {
  incrementBudgetContinuationCount()
  state = {
    messages: [
      ...messagesForQuery,
      ...assistantMessages,
      createUserMessage({
        content: decision.nudgeMessage,
        isMeta: true,
      }),
    ],
    // ...
  }
  continue  // 继续 agent loop
}
```

---

## 8.9 全局 Token 状态（bootstrap/state.ts）

[src/bootstrap/state.ts:724-743](../../src/bootstrap/state.ts#L724-L743)

```typescript
let outputTokensAtTurnStart = 0       // 本轮开始时的 output token 基线
let currentTurnTokenBudget: number | null = null  // 用户设置的预算值
let budgetContinuationCount = 0       // 本轮"继续"次数（UI 状态栏读取）

// 本轮相对增量 = 累计总量 - 本轮基线
export function getTurnOutputTokens(): number {
  return getTotalOutputTokens() - outputTokensAtTurnStart
}

export function getCurrentTurnTokenBudget(): number | null {
  return currentTurnTokenBudget
}

// REPL 层调用：快照基线 + 设置预算 + 重置计数
export function snapshotOutputTokensForTurn(budget: number | null): void {
  outputTokensAtTurnStart = getTotalOutputTokens()
  currentTurnTokenBudget = budget
  budgetContinuationCount = 0
}

// query.ts 调用：每次 continue 时递增（UI 状态栏展示用）
export function incrementBudgetContinuationCount(): void {
  budgetContinuationCount++
}

// UI 状态栏读取：显示当前已继续次数
export function getBudgetContinuationCount(): number {
  return budgetContinuationCount
}
```

**关键理解**：`continuationCount` 同时存在于 `BudgetTracker`（用于逻辑判断）和 `bootstrap/state.ts` 的 `budgetContinuationCount`（用于 UI 展示），两者在每次 continue 时并行递增。

---

## 8.10 与 Agent Loop 的集成

[src/query.ts:1308-1354](../../src/query.ts#L1308-L1354)

```typescript
// 创建 BudgetTracker（query loop 入口，每次用户提交创建一个）
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null

// 每个 agent loop 轮次结束后（工具执行完成后）：
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!,
    toolUseContext.agentId,
    getCurrentTurnTokenBudget(),   // 从 bootstrap/state 读取
    getTurnOutputTokens(),         // 本轮相对增量
  )

  if (decision.action === 'continue') {
    incrementBudgetContinuationCount()  // 更新 UI 计数
    // state.messages = [..., nudgeMessage（isMeta: true）]
    continue  // 继续 loop
  }

  // action === 'stop'
  if (decision.completionEvent) {
    logEvent('tengu_token_budget_completed', {
      ...decision.completionEvent,  // continuationCount, pct, turnTokens, budget, diminishingReturns, durationMs
      queryChainId,
      queryDepth,
    })
  }
  // 退出 loop，返回 terminal
}
```

**用户提交结束后**（REPL 层），重置预算避免影响下一轮：
```typescript
// src/screens/REPL.tsx：query 完成后
snapshotOutputTokensForTurn(null)  // 清除预算（budget=null）
```

---

## 8.11 API Task Budget（服务端预算）

[src/query.ts:193-197](../../src/query.ts#L193-L197)

```typescript
// query 参数（独立于 +500k 机制）
taskBudget?: { total: number }
// API task_budget (output_config.task_budget, beta task-budgets-2026-03-13)
// Distinct from the tokenBudget +500k auto-continue feature.
// `total` is the budget for the whole agentic turn;
// `remaining` is computed per iteration from cumulative API usage.
```

每次 API 调用传入 `{ total, remaining }`，其中 `remaining` 在每轮迭代后更新。

**Compaction 后的 remaining 更新**：压缩会替换消息历史，新上下文 token 数减少，`remaining` 需相应扣除旧上下文的 token 以保持一致性：
```typescript
if (params.taskBudget) {
  const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

与用户层 Token Budget 的区别：

| 维度 | 用户层（+500k）| API 层（taskBudget）|
|------|--------------|-------------------|
| 计数位置 | 客户端，`getTurnOutputTokens()` | 服务端，API 计数 |
| 超限后果 | Claude Code 停止调用 API | API 拒绝请求（返回错误）|
| 目的 | 防止 agentic 会话失控运行 | API 层面的消费控制 |

---

## 8.12 递减收益检测的设计思考

为什么不只检查"超过 90%"就停止？

**问题场景**：Claude 执行一个长任务，消耗了大量 token，但在某个阶段陷入循环（重复调用工具、重复生成相同输出）。这时 token 消耗可能才到 50%，但每轮只新增几十个 token，Claude 实际上已经"空转"了。

**解决方案**：在 `continuationCount >= 3` 的前提下，若最近两轮 delta 均 `< 500`，判定为递减收益，强制停止。

注意 `isDiminishing` 的精确条件：不是"连续 3 轮都 < 500"，而是"已经到了第 3+ 轮（continuationCount >= 3），且最近两轮（当前轮次 + 上一轮）的 delta 都很小"。实际上在第 4 次 `checkTokenBudget` 调用时才可能触发（前 3 次 continue 后，第 4 次检查时 continuationCount = 3）。

**比单纯百分比更智能的表现**：

| 情形 | 处理 |
|------|------|
| 正常工作，消耗 80% | 继续（仍在创造 value）|
| 空转，消耗 50%，每轮仅增 100 token | 第 4 次检查后停止 |
| 第一轮直接超 90% | 停止，不带 completionEvent（无 continuationCount 数据）|

---

## 小结

| 机制 | 说明 | 源码位置 |
|------|------|----------|
| 语法解析 | `+500k`/`use 2M tokens`/`spend 1.5b tokens`，支持首/尾/任意位置 | `src/utils/tokenBudget.ts` |
| 预算设置时机 | 用户提交时 REPL 调用 `snapshotOutputTokensForTurn()` | `src/screens/REPL.tsx:2926` |
| `getTurnOutputTokens()` | 相对增量 = 累计总量 − 本轮基线 | `src/bootstrap/state.ts:726` |
| BudgetTracker | 追踪连续轮次和 delta，每次 query 创建一个 | `src/query/tokenBudget.ts` |
| 递减收益检测 | `continuationCount≥3` AND 最近两轮 delta 均 `<500` | `checkTokenBudget()` |
| Nudge Message | `"Stopped at X% of token target... Keep working — do not summarize."` | `getBudgetContinuationMessage()` |
| 子代理豁免 | `agentId` 存在时不受预算约束 → stop（null 事件）| `checkTokenBudget():51` |
| UI 计数同步 | `incrementBudgetContinuationCount()` 同步状态栏显示 | `src/bootstrap/state.ts:741` |
| API Task Budget | 服务端计数（独立机制），compaction 后更新 remaining | [src/query.ts:193](../../src/query.ts#L193) |
