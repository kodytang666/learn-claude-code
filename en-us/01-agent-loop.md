# Chapter 1: Agent Loop — The Main Query Loop

> Source location: `src/query.ts` (main entry), `src/query/` directory (submodules)

---

## 1.1 What Is the Agent Loop?

The Agent Loop is the heart of Claude Code. It is an **infinitely running async generator** responsible for:

1. Sending user messages to the Claude API (via streaming)
2. Receiving Claude's responses (text + tool calls)
3. Executing tool calls and feeding results back to Claude
4. Deciding whether to continue looping (or stop)
5. Handling edge cases like context overflow and token limits

Put simply: the Agent Loop is the engine that drives the "Claude thinks → acts → observes → thinks again" cycle.

---

## 1.2 Core Data Structures

### QueryParams

[src/query.ts:181-199](../../src/query.ts#L181-L199)
```typescript
export type QueryParams = {
  messages: Message[]          // Conversation history
  systemPrompt: SystemPrompt   // System prompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn     // Tool permission check function
  toolUseContext: ToolUseContext
  fallbackModel?: string       // Fallback model (for error recovery)
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number            // Maximum turn limit
  skipCacheWrite?: boolean
  taskBudget?: { total: number }  // API-level task budget
  deps?: QueryDeps             // Dependency injection (for testing)
}
```

### State (Internal Loop State)

[src/query.ts:204-217](../../src/query.ts#L204-L217)
```typescript
type State = {
  messages: Message[]          // Message list, updated each turn
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number  // max_output_tokens error recovery count
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number            // Current loop turn count
  transition: Continue | undefined  // Previous turn's continue reason (for test assertions)
}
```

---

## 1.3 Loop Structure Overview

```
query()  ← Public entry point (AsyncGenerator)
  └─ queryLoop()  ← The actual loop logic
       ├─ [Outside loop] startRelevantMemoryPrefetch()   ← Triggered once per user turn
       └─ while(true) {
            1.  Start skill prefetch (parallel with API call)
            2.  yield 'stream_request_start'              ← Notify UI that request is starting
            3.  Initialize query chain tracking
            ─── Message preparation (5 layers, all before API call) ───
            4.  getMessagesAfterCompactBoundary()         ← Only take messages after compact boundary
            5.  applyToolResultBudget()                   ← Limit tool result size
            6.  HISTORY_SNIP (feature gate)               ← Trim history
            7.  microcompact                              ← Micro compaction
            8.  CONTEXT_COLLAPSE (feature gate)           ← Collapse context
            9.  autocompact                              ← Auto compaction
           10.  blocking limit check                      ← Return error if over limit
            ─── API Call ────────────────────────────────
           11.  Streaming Claude API call
                └─ prependUserContext() injects userContext at call time
           ─── Tool Execution ──────────────────────────
           12.  No tool_use? → handleStopHooks() → return terminal
           13.  runTools()                                ← Execute tool calls
            ─── Post-tool Execution ────────────────────
           14.  getAttachmentMessages()                   ← Inject queued commands and other dynamic context
           15.  Consume memory prefetch result
           16.  Consume skill discovery result
           17.  refreshTools()                            ← Refresh MCP tools (newly connected servers)
            ─── Stop / Continue Decision ──────────────
           18.  maxTurns reached? → return terminal
           19.  Update state, enter next turn
          }
```

---

## 1.4 Main Loop Walkthrough

```typescript
async function* queryLoop(params: QueryParams) {
  // Invariants: not reassigned within the loop
  const { systemPrompt, userContext, systemContext, canUseTool, maxTurns } = params

  // Mutable state. Updated each turn by replacing the entire state object, not field-by-field
  let state: State = {
    messages: params.messages,
    turnCount: 1,
    // ...
  }

  // Token Budget Tracker (TOKEN_BUDGET feature gate)
  const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null

  // Memory prefetch: declared outside the while loop, triggered only once per user turn
  // The `using` keyword ensures automatic resource cleanup when the generator exits (including on error)
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(   // [src/query.ts:301](../../src/query.ts#L301)
    state.messages,
    state.toolUseContext,
  )

  while (true) {
    const { messages, toolUseContext, turnCount } = state

    // ── Each turn start: skill prefetch runs parallel with API call ──────────────────
    const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(...)  // [src/query.ts:331](../../src/query.ts#L331)

    yield { type: 'stream_request_start' }  // Notify UI that this turn's request is starting

    // query chain tracking: maintains chainId + depth across turns for analytics
    const queryTracking = { chainId: ..., depth: ... }

    // ── Message preparation (5 layers, all before API call) ─────────────────────────

    // 1. Only take messages after the compact boundary (history before compact is replaced by summary)
    let messagesForQuery = getMessagesAfterCompactBoundary(messages)  // [src/query.ts:365](../../src/query.ts#L365)

    // 2. Limit tool result size (results exceeding maxResultSizeChars are truncated/replaced)
    messagesForQuery = await applyToolResultBudget(messagesForQuery, ...)  // [src/query.ts:379](../../src/query.ts#L379)

    // 3. HISTORY_SNIP (feature gate): trim overly long history
    if (feature('HISTORY_SNIP')) {
      messagesForQuery = snipModule.snipCompactIfNeeded(messagesForQuery).messages
    }

    // 4. microcompact: run micro compaction before autocompact
    messagesForQuery = await deps.microcompact(messagesForQuery, ...).messages  // [src/query.ts:414](../../src/query.ts#L414)

    // 5. CONTEXT_COLLAPSE (feature gate): collapse collapsible context
    if (feature('CONTEXT_COLLAPSE')) {
      messagesForQuery = await contextCollapse.applyCollapsesIfNeeded(...).messages
    }

    // 6. autocompact: compress when context is too long (before API call, not after)
    const { compactionResult } = await deps.autocompact(messagesForQuery, ...)  // [src/query.ts:454](../../src/query.ts#L454)

    // 7. blocking limit check: if auto compact is disabled and over limit, report error and exit
    if (!compactionResult && isAtBlockingLimit(messagesForQuery)) {  // [src/query.ts:637](../../src/query.ts#L637)
      yield createAssistantAPIErrorMessage({ error: 'invalid_request' })
      return { reason: 'blocking_limit' }
    }

    // ── Streaming Claude API call ─────────────────────────────────────────────────

    // userContext (CLAUDE.md + current date) is injected at call time via prependUserContext
    // Not part of systemPrompt; passed as a message prefix prepended to the messages array
    for await (const message of deps.callModel({        // [src/query.ts:659](../../src/query.ts#L659)
      messages: prependUserContext(messagesForQuery, userContext),
      systemPrompt: fullSystemPrompt,
      tools,
      // ...
    })) {
      yield message  // Stream events forwarded to UI in real time
    }

    // ── Tool Execution ─────────────────────────────────────────────────────────

    // No tool_use → Claude is done, execute stop hooks and exit
    if (toolUseBlocks.length === 0) {
      yield* handleStopHooks(...)
      return terminal
    }

    const toolResults = []
    for await (const result of runTools(assistantMessage, toolUseContext)) {
      yield result
      toolResults.push(result)
    }

    // ── Post-tool execution: inject dynamic context ────────────────────────────

    // getAttachmentMessages is called after tool execution, handles queued commands etc.
    for await (const attachment of getAttachmentMessages(null, toolUseContext, ...)) {  // [src/query.ts:1580](../../src/query.ts#L1580)
      yield attachment
      toolResults.push(attachment)
    }

    // Consume memory prefetch result (if settled)
    if (pendingMemoryPrefetch?.settledAt !== null) {
      // Append memory attachments to toolResults
    }

    // Consume skill discovery prefetch result
    if (skillPrefetch && pendingSkillPrefetch) {
      // Append skill_discovery attachments to toolResults
    }

    // Refresh MCP tools: make newly connected MCP servers available in the next turn
    if (toolUseContext.options.refreshTools) {   // [src/query.ts:1660](../../src/query.ts#L1660)
      const refreshedTools = toolUseContext.options.refreshTools()
      // Update toolUseContext.options.tools
    }

    // ── Stop / Continue decision ──────────────────────────────────────────────

    // maxTurns check (note: after tool execution, not before)
    if (maxTurns && nextTurnCount > maxTurns) {   // [src/query.ts:1705](../../src/query.ts#L1705)
      return { reason: 'max_turns' }
    }

    // Update state, enter next turn
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

## 1.5 Feature Gate Pattern

Claude Code uses Bun's `feature()` function for **compile-time feature flags**. Disabled features are not bundled:

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

## 1.6 max_output_tokens Error Recovery

The Claude API sometimes interrupts when output token limits are reached. The Agent Loop has a dedicated recovery mechanism:

[src/query.ts:164-165](../../src/query.ts#L164-L165)
```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

function isWithheldMaxOutputTokens(msg): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

**Recovery logic:**
1. Detect `max_output_tokens` error
2. Retry up to 3 times (`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`)
3. First attempt Reactive Compact (if enabled)
4. If still failing, report error to user

**Why "withhold" the error message?**

As the comment explains: leaking intermediate errors to SDK callers (like co-work/desktop) would terminate the session — the recovery loop would still be running but nobody would be listening. So the error is withheld until it's confirmed whether the recovery loop can continue.

---

## 1.7 The Elegance of Async Generators

`query()` is an `AsyncGenerator`, which lets it simultaneously:

- **Stream output**: Use `yield` to forward API streaming events to the UI in real time
- **Propagate errors**: Exceptions propagate naturally through `yield*`
- **Clean up resources**: The `using` keyword (TC39 Explicit Resource Management) automatically releases resources when the generator exits

[src/query.ts:219-239](../../src/query.ts#L219-L239)
```typescript
export async function* query(params: QueryParams) {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  
  // Only executed on normal return (skipped on throw or .return() call)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

---

## 1.8 Flowchart

```
User submits message
      │
      ▼
  query() → queryLoop()
      │
      ├─ startRelevantMemoryPrefetch()   ← Outside while loop, triggered once per user turn
      │
      ▼
  while(true) {  ←──────────────────────────────────────────────────────┐
      │                                                                   │
      ├─ startSkillDiscoveryPrefetch()   ← Started each turn, parallel with API call  │
      ├─ yield 'stream_request_start'                                     │
      │                                                                   │
      │  ── Message preparation (before API call) ─────────────────────  │
      ├─ getMessagesAfterCompactBoundary()   ← Get messages after compact boundary  │
      ├─ applyToolResultBudget()             ← Limit tool result size     │
      ├─ HISTORY_SNIP (feature gate)         ← Trim history               │
      ├─ microcompact                        ← Micro compaction           │
      ├─ CONTEXT_COLLAPSE (feature gate)     ← Collapse context           │
      ├─ autocompact                         ← Auto compaction            │
      ├─ blocking limit? ──YES──→ return error                            │
      │                                                                   │
      │  ── Streaming API call ────────────────────────────────────────  │
      ├─ callModel(prependUserContext(messages))  ← userContext injected here  │
      │     └─ yield streaming events ...                                  │
      │                                                                   │
      │  ── Tool execution ────────────────────────────────────────────  │
      ├─ No tool_use? ──YES──→ handleStopHooks() → return terminal        │
      ├─ runTools()                          ← Execute tool calls         │
      │     └─ yield tool results ...                                      │
      │                                                                   │
      │  ── Post-tool execution ───────────────────────────────────────  │
      ├─ getAttachmentMessages()             ← Dynamic context: queued commands etc.
      ├─ Consume memory prefetch result                                    │
      ├─ Consume skill discovery result                                    │
      ├─ refreshTools()                      ← Refresh MCP tools          │
      │                                                                   │
      │  ── Stop / Continue ───────────────────────────────────────────  │
      ├─ maxTurns reached? ──YES──→ return terminal                        │
      │                                                                   │
      └─ Update state, continue loop ──────────────────────────────────┘
  }
```

---

## Summary

| Concept | Implementation | Source Location |
|---------|---------------|-----------------|
| Main loop | `async function* queryLoop()` | [src/query.ts:241](../../src/query.ts#L241) |
| Message preparation (5 layers) | compact boundary → budget → snip → microcompact → autocompact | [src/query.ts:365-468](../../src/query.ts#L365-L468) |
| userContext injection | `prependUserContext(messagesForQuery, userContext)` | [src/query.ts:660](../../src/query.ts#L660) |
| Streaming event forwarding | `yield event` | `src/query.ts` |
| Tool execution | `runTools()` | `src/services/tools/toolOrchestration.ts` |
| Post-tool dynamic context | `getAttachmentMessages()` called after runTools | [src/query.ts:1580](../../src/query.ts#L1580) |
| MCP tools refresh | `refreshTools()` called after each turn's tool execution | [src/query.ts:1660](../../src/query.ts#L1660) |
| Error recovery | `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` | [src/query.ts:164](../../src/query.ts#L164) |
| Feature flags | `feature('FLAG_NAME')` | Eliminated at Bun bundle compile time |
| Resource management | `using` keyword (memory prefetch declared outside loop) | [src/query.ts:301](../../src/query.ts#L301) |
