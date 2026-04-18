# Chapter 8: Token Budget — Token Budget and Diminishing Returns Detection

> Source location: [src/query/tokenBudget.ts](../../src/query/tokenBudget.ts), [src/utils/tokenBudget.ts](../../src/utils/tokenBudget.ts), [src/bootstrap/state.ts](../../src/bootstrap/state.ts), [src/screens/REPL.tsx](../../src/screens/REPL.tsx)

---

## 8.1 What Is a Token Budget?

The Token Budget is a mechanism that lets users control the **maximum number of output tokens Claude can consume in a single task**.

**Use cases**:
- User inputs `+500k` to allow consuming up to 500,000 tokens
- User inputs `spend 2M tokens` to allow consuming up to 2,000,000 tokens
- Claude Code automatically stops before the budget is exhausted, preventing runaway loops

**Note**: Token Budget and Auto Compaction are **different** mechanisms:
- Auto Compaction: Context window is nearly full; compress history (input tokens)
- Token Budget: Maximum output token consumption limit set by the user, preventing agentic sessions from running out of control

---

## 8.2 Two Independent Budget Mechanisms

| Layer | Source | Description |
|-------|--------|-------------|
| **User-layer Token Budget** | User inputs `+500k` in prompt | Controls the output token consumption ceiling for the entire agentic session; client-side counting |
| **API-layer Task Budget** | `taskBudget.total` parameter | Claude API's `output_config.task_budget` beta feature; server-side counting |

This chapter focuses primarily on the user-layer Token Budget.

---

## 8.3 User Input Syntax Parsing

[src/utils/tokenBudget.ts:1-29](../../src/utils/tokenBudget.ts#L1-L29)

### Supported Syntax

```typescript
// Shorthand form: must be at start or end of prompt to avoid false matches in natural language
const SHORTHAND_START_RE = /^\s*\+(\d+(?:\.\d+)?)\s*(k|m|b)\b/i  // At start
const SHORTHAND_END_RE   = /\s\+(\d+(?:\.\d+)?)\s*(k|m|b)\s*[.!?]?\s*$/i  // At end

// Verbose form: can appear anywhere in the prompt
const VERBOSE_RE = /\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b/i
```

Supported units: `k` (thousands), `m` (millions), `b` (billions); case-insensitive; decimals supported (e.g., `1.5M`).

**Syntax examples**:

| Input | Parsed Result |
|-------|--------------|
| `+500k` | 500,000 (must be at prompt start) |
| `fix this bug +2M` | 2,000,000 (shorthand at end) |
| `spend 1.5M tokens` | 1,500,000 (verbose, anywhere) |
| `use 500k tokens` | 500,000 (verbose, anywhere) |
| `+2b` | 2,000,000,000 (billions) |

### parseTokenBudget and findTokenBudgetPositions

```typescript
// Parse and return the numeric budget (null means no budget syntax)
export function parseTokenBudget(text: string): number | null

// Return all budget syntax positions in text (for removing syntax so Claude doesn't see it)
export function findTokenBudgetPositions(
  text: string,
): Array<{ start: number; end: number }>
// Note: returns an array, not a single object; doesn't include budget value; returns [] not null on no match
```

---

## 8.4 Budget Parsing Timing: REPL.tsx

[src/screens/REPL.tsx:2925-2928](../../src/screens/REPL.tsx#L2925-L2928)

Budget parsing does **not** happen in `query.ts` — it happens in `REPL.tsx` when the user presses Enter to submit a message:

```typescript
// Executed each time user submits a new message (src/screens/REPL.tsx)
if (feature('TOKEN_BUDGET')) {
  const parsedBudget = input ? parseTokenBudget(input) : null
  // 1. Set outputTokensAtTurnStart = current accumulated output tokens (baseline snapshot)
  // 2. Set currentTurnTokenBudget = parsedBudget ?? getCurrentTurnTokenBudget()
  //    Note: if no budget syntax in new input, carry over the previous turn's budget
  // 3. Reset budgetContinuationCount = 0
  snapshotOutputTokensForTurn(parsedBudget ?? getCurrentTurnTokenBudget())
}
```

`snapshotOutputTokensForTurn(budget)` does three things:

```typescript
// src/bootstrap/state.ts:733-737
export function snapshotOutputTokensForTurn(budget: number | null): void {
  outputTokensAtTurnStart = getTotalOutputTokens()  // Snapshot current accumulated value as baseline
  currentTurnTokenBudget = budget
  budgetContinuationCount = 0
}
```

This enables `getTurnOutputTokens()` to correctly compute **new output tokens in this turn**:

```typescript
export function getTurnOutputTokens(): number {
  return getTotalOutputTokens() - outputTokensAtTurnStart  // Relative delta, not cumulative total
}
```

**Design detail**: The budget is set at the REPL layer so it remains active across multiple `continue` iterations of the agent loop, rather than being managed per query call.

---

## 8.5 BudgetTracker: Tracking Consecutive Turns

[src/query/tokenBudget.ts:6-20](../../src/query/tokenBudget.ts#L6-L20)

```typescript
export type BudgetTracker = {
  continuationCount: number        // Number of "continues" so far (incremented on each continue)
  lastDeltaTokens: number          // Token delta from the last check
  lastGlobalTurnTokens: number     // Cumulative token count at last check (for computing current delta)
  startedAt: number                // Session start timestamp (for computing durationMs)
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

`BudgetTracker` is created at each query loop entry point (`feature('TOKEN_BUDGET') ? createBudgetTracker() : null`), tracking all agent turn data since the current user submission.

---

## 8.6 Core Decision Logic: checkTokenBudget()

[src/query/tokenBudget.ts:45-93](../../src/query/tokenBudget.ts#L45-L93)

```typescript
const COMPLETION_THRESHOLD = 0.9      // 90%
const DIMINISHING_THRESHOLD = 500     // Minimum delta for diminishing returns detection (500 tokens)

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,   // Subagents are not subject to budget constraints
  budget: number | null,         // User-set budget (null = no budget)
  globalTurnTokens: number,      // Total new output tokens this turn (getTurnOutputTokens())
): TokenBudgetDecision {

  // Rule 1: Subagent / no budget / budget ≤ 0 → stop immediately, no completion event
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  const turnTokens = globalTurnTokens
  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // Rule 2: Diminishing returns detection
  // Condition: continuationCount >= 3 (already continued 3 times)
  //   AND current turn delta < 500 tokens
  //   AND previous turn delta < 500 tokens
  // Note: Only checks the most recent two turns (current + previous), NOT "all 3 consecutive < 500"
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // Rule 3: No diminishing AND under 90% → continue (send nudge message)
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

  // Rule 4: Diminishing OR already continued (when over 90%) → stop, with completion event
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

  // Rule 5: First turn directly exceeds 90% (never continued) → stop, no event
  return { action: 'stop', completionEvent: null }
}
```

### TokenBudgetDecision Types

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
  } | null   // null = no event (subagent/no budget/first turn exceeds limit)
}
```

---

## 8.7 Decision Tree

```
checkTokenBudget() is called
          │
          ▼
  agentId present OR budget = null OR budget ≤ 0?
     │              │
    YES             NO
     │              │
stop (null event)   ▼
              Compute deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens
              Compute pct (usage percentage)
                     │
                     ▼
              isDiminishing?
              (continuationCount≥3 AND current delta<500 AND last delta<500)
                │        │
               YES        NO
                │         │
                │         ▼
                │   turnTokens < budget × 90%?
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
               continuationCount > 0?
                  │         │
                 YES        NO
                  │         │
           stop (with event)  stop (null event)
           includes diminishingReturns    (first turn exceeds limit)
```

---

## 8.8 Nudge Message (Continue Prompt Message)

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

**Actual message example**:
```
Stopped at 45% of token target (225,000 / 500,000). Keep working — do not summarize.
```

**Note**: The message starts with `"Stopped at X% of token target..."` (not "You have consumed..."), and ends with the explicit instruction **"do not summarize"**, preventing Claude from starting to wrap up before the budget is exhausted.

This message is injected into the conversation with `isMeta: true` in `query.ts` (not visible in UI, but sent to Claude API):

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
  continue  // Continue agent loop
}
```

---

## 8.9 Global Token State (bootstrap/state.ts)

[src/bootstrap/state.ts:724-743](../../src/bootstrap/state.ts#L724-L743)

```typescript
let outputTokensAtTurnStart = 0       // Output token baseline at turn start
let currentTurnTokenBudget: number | null = null  // User-set budget value
let budgetContinuationCount = 0       // Current turn's "continue" count (read by UI status bar)

// Relative delta for this turn = cumulative total - this turn's baseline
export function getTurnOutputTokens(): number {
  return getTotalOutputTokens() - outputTokensAtTurnStart
}

export function getCurrentTurnTokenBudget(): number | null {
  return currentTurnTokenBudget
}

// Called by REPL layer: snapshot baseline + set budget + reset count
export function snapshotOutputTokensForTurn(budget: number | null): void {
  outputTokensAtTurnStart = getTotalOutputTokens()
  currentTurnTokenBudget = budget
  budgetContinuationCount = 0
}

// Called by query.ts: increment on each continue (for UI status bar display)
export function incrementBudgetContinuationCount(): void {
  budgetContinuationCount++
}

// Read by UI status bar: shows current continuation count
export function getBudgetContinuationCount(): number {
  return budgetContinuationCount
}
```

**Key insight**: `continuationCount` exists in both `BudgetTracker` (for logic decisions) and `bootstrap/state.ts`'s `budgetContinuationCount` (for UI display). Both are incremented in parallel on each continue.

---

## 8.10 Integration with Agent Loop

[src/query.ts:1308-1354](../../src/query.ts#L1308-L1354)

```typescript
// Create BudgetTracker (at query loop entry; one per user submission)
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null

// After each agent loop turn (after tool execution):
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!,
    toolUseContext.agentId,
    getCurrentTurnTokenBudget(),   // Read from bootstrap/state
    getTurnOutputTokens(),         // Relative delta for this turn
  )

  if (decision.action === 'continue') {
    incrementBudgetContinuationCount()  // Update UI count
    // state.messages = [..., nudgeMessage (isMeta: true)]
    continue  // Continue loop
  }

  // action === 'stop'
  if (decision.completionEvent) {
    logEvent('tengu_token_budget_completed', {
      ...decision.completionEvent,  // continuationCount, pct, turnTokens, budget, diminishingReturns, durationMs
      queryChainId,
      queryDepth,
    })
  }
  // Exit loop, return terminal
}
```

**After user submission ends** (REPL layer), reset the budget to avoid affecting the next turn:
```typescript
// src/screens/REPL.tsx: after query completes
snapshotOutputTokensForTurn(null)  // Clear budget (budget=null)
```

---

## 8.11 API Task Budget (Server-Side Budget)

[src/query.ts:193-197](../../src/query.ts#L193-L197)

```typescript
// Query parameter (independent from the +500k mechanism)
taskBudget?: { total: number }
// API task_budget (output_config.task_budget, beta task-budgets-2026-03-13)
// Distinct from the tokenBudget +500k auto-continue feature.
// `total` is the budget for the whole agentic turn;
// `remaining` is computed per iteration from cumulative API usage.
```

`{ total, remaining }` is passed on each API call; `remaining` is updated after each iteration.

**Updating `remaining` after compaction**: Compaction replaces message history, reducing the new context token count. `remaining` must be adjusted to deduct old context tokens for consistency:
```typescript
if (params.taskBudget) {
  const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

Comparison with user-layer Token Budget:

| Dimension | User-layer (+500k) | API-layer (taskBudget) |
|-----------|-------------------|----------------------|
| Counting location | Client-side, `getTurnOutputTokens()` | Server-side, API counting |
| Consequence of exceeding | Claude Code stops calling API | API rejects request (returns error) |
| Purpose | Prevent agentic sessions from running out of control | API-level consumption control |

---

## 8.12 Design Thinking Behind Diminishing Returns Detection

Why not just stop when "over 90%"?

**Problem scenario**: Claude is executing a long task, consuming many tokens, but gets stuck in a loop at some point (repeatedly calling tools, repeatedly generating the same output). Token consumption might only be at 50%, but each turn only adds a few dozen tokens — Claude is effectively "spinning idle."

**Solution**: When `continuationCount >= 3`, if the most recent two turns both have delta `< 500`, declare diminishing returns and force a stop.

Note the precise condition for `isDiminishing`: it's NOT "3 consecutive turns all < 500," but rather "already at or past turn 3 (continuationCount >= 3), AND the most recent two turns (current + previous) both had small deltas." In practice, it can only trigger at the 4th `checkTokenBudget` call (after 3 continues, continuationCount = 3, checked on the 4th call).

**Why this is smarter than a pure percentage check**:

| Scenario | Handling |
|----------|----------|
| Normal work, 80% consumed | Continue (still creating value) |
| Idling, 50% consumed, only +100 tokens per turn | Stop after 4th check |
| First turn directly exceeds 90% | Stop; no completionEvent (no continuationCount data) |

---

## Summary

| Mechanism | Description | Source Location |
|-----------|-------------|-----------------|
| Syntax parsing | `+500k`/`use 2M tokens`/`spend 1.5b tokens`; start/end/anywhere | `src/utils/tokenBudget.ts` |
| Budget set timing | REPL calls `snapshotOutputTokensForTurn()` on user submit | `src/screens/REPL.tsx:2926` |
| `getTurnOutputTokens()` | Relative delta = cumulative total − this turn's baseline | `src/bootstrap/state.ts:726` |
| BudgetTracker | Tracks consecutive turns and deltas; one created per query | `src/query/tokenBudget.ts` |
| Diminishing returns detection | `continuationCount≥3` AND most recent two turns both delta `<500` | `checkTokenBudget()` |
| Nudge Message | `"Stopped at X% of token target... Keep working — do not summarize."` | `getBudgetContinuationMessage()` |
| Subagent exemption | When `agentId` is present, not subject to budget → stop (null event) | `checkTokenBudget():51` |
| UI count sync | `incrementBudgetContinuationCount()` syncs status bar display | `src/bootstrap/state.ts:741` |
| API Task Budget | Server-side counting (independent mechanism); updates remaining after compaction | [src/query.ts:193](../../src/query.ts#L193) |
