# Chapter 7: Context Compaction — Auto Compaction and Memory Restoration

> Source location: [src/services/compact/compact.ts](../../src/services/compact/compact.ts), [src/services/compact/autoCompact.ts](../../src/services/compact/autoCompact.ts)

---

## 7.1 Why Is Context Compaction Needed?

The Claude API has a context window limit (typically 200K tokens). In long coding sessions, conversation history grows continuously:

- Code file reads (potentially thousands of lines)
- Tool call results (Bash output, search results)
- Back-and-forth conversation messages

When context approaches the limit, there are two options:
1. Simply truncate history (losing important information)
2. **Intelligent compaction**: Let Claude summarize previous conversation, preserving key information

Claude Code chose option 2 and implemented a multi-layer automatic trigger mechanism.

---

## 7.2 Compaction Types

| Type | File | Trigger Method |
|------|------|----------------|
| **Full Compact** | `compact.ts` | Manual `/compact` or automatic trigger |
| **Auto Compact** | `autoCompact.ts` | Triggered automatically when token usage reaches threshold |
| **Micro Compact** | `microCompact.ts` | Incremental compaction for individual large tool results |
| **API Context Management** | `apiMicrocompact.ts` | Leverages API beta features |
| **Reactive Compact** | `reactiveCompact.ts` | Triggered after `max_output_tokens` error (feature-gated) |
| **Session Memory Compact** | `sessionMemoryCompact.ts` | Attempted first before Auto Compact; based on cross-session long-term memory |

---

## 7.3 Core Constants and Restoration Budget

[src/services/compact/compact.ts:122-131](../../src/services/compact/compact.ts#L122-L131)
```typescript
// Max files to restore after compact
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5

// Total token budget for file restoration post-compact
export const POST_COMPACT_TOKEN_BUDGET = 50_000

// Max tokens per file restoration
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000

// Max tokens per skill restoration
// Note: verify=18.7KB, claude-api=20.1KB; previously unlimited, caused 5-10K tokens per compact
// Truncation is better than omission — the head of skill files contains the most important instructions
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000

// Total token budget for skill restoration (approximately 5 skills)
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000

// Max retries for compact streaming request (PTL retries)
const MAX_PTL_RETRIES = 3
```

---

## 7.4 Threshold Calculation System (4 Levels)

[src/services/compact/autoCompact.ts:62-144](../../src/services/compact/autoCompact.ts#L62-L144)

Auto Compact introduces 4 different token warning thresholds, all calculated in `calculateTokenWarningState()`:

```typescript
export const AUTOCOMPACT_BUFFER_TOKENS   = 13_000   // Auto compact buffer
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000  // Warning threshold buffer
export const ERROR_THRESHOLD_BUFFER_TOKENS   = 20_000  // Error threshold buffer
export const MANUAL_COMPACT_BUFFER_TOKENS    = 3_000   // Blocking limit buffer
```

### Effective Context Window (effectiveContextWindow)

```
effectiveContextWindow
  = min(contextWindow, CLAUDE_CODE_AUTO_COMPACT_WINDOW)
  - min(getMaxOutputTokensForModel(model), 20_000)
```

Reserves `min(maxOutput, 20_000)` tokens for summary output (based on p99.99 measured maximum summary of 17,387 tokens).

### 4 Thresholds

| Threshold | Formula | Meaning |
|-----------|---------|---------|
| `autoCompactThreshold` | `effectiveWindow - 13_000` | Trigger auto compaction |
| `warningThreshold` | `threshold - 20_000` | Show orange percentage warning |
| `errorThreshold` | `threshold - 20_000` | Show red error state (same as warning, different behavior) |
| `blockingLimit` | `effectiveWindow - 3_000` | Completely block new input |

> Note: `threshold` inside `calculateTokenWarningState`: when auto compact is enabled = `autoCompactThreshold`; when disabled = `effectiveContextWindow`.

### calculateTokenWarningState Signature

```typescript
// src/services/compact/autoCompact.ts
export function calculateTokenWarningState(
  tokenUsage: number,  // Current token count (not the messages array)
  model: string,       // Model name (for querying context window size)
): {
  percentLeft: number                // Percentage remaining (0-100)
  isAboveWarningThreshold: boolean   // Whether above warning threshold
  isAboveErrorThreshold: boolean     // Whether above error threshold
  isAboveAutoCompactThreshold: boolean  // Whether triggering auto compact
  isAtBlockingLimit: boolean         // Whether at blocking limit
}
```

**Note**: Signature is `(tokenUsage: number, model: string)` not `(messages, contextLimit)`. Return value is an object with 5 boolean fields, not `'warning' | 'critical' | null`.

`getAutoCompactThreshold` also accepts a `model` parameter:

```typescript
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

  // Supports CLAUDE_AUTOCOMPACT_PCT_OVERRIDE env var override (for testing)
  const envPercent = process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
  if (envPercent) {
    const percentageThreshold = Math.floor(effectiveContextWindow * (parsed / 100))
    return Math.min(percentageThreshold, autocompactThreshold)
  }
  return autocompactThreshold
}
```

---

## 7.5 isAutoCompactEnabled and shouldAutoCompact

### isAutoCompactEnabled()

```typescript
export function isAutoCompactEnabled(): boolean {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return false
  // Only disables auto compact; manual /compact is preserved
  if (isEnvTruthy(process.env.DISABLE_AUTO_COMPACT)) return false
  return getGlobalConfig().autoCompactEnabled
}
```

Check order: `DISABLE_COMPACT` → `DISABLE_AUTO_COMPACT` → user config.

### shouldAutoCompact() Recursion Guard

[src/services/compact/autoCompact.ts:160-238](../../src/services/compact/autoCompact.ts#L160-L238)

Compaction itself launches a forked agent (querySource is `'compact'`). Allowing compaction to trigger again inside these sub-calls would cause a deadlock. Therefore `shouldAutoCompact()` has multiple recursion guards:

```typescript
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed = 0,
): Promise<boolean> {
  // Guard 1: Self-recursion (compact's forked agent cannot trigger compact again)
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }

  // Guard 2: feature gate - marble_origami (ctx-agent) would break main thread if triggered
  if (feature('CONTEXT_COLLAPSE') && querySource === 'marble_origami') {
    return false
  }

  // Guard 3: feature gate - reactive compact mode prohibits proactive compaction
  if (feature('REACTIVE_COMPACT') && getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false
  }

  // Guard 4: Context Collapse mode already manages context; shouldn't compete with autocompact
  if (feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled()) {
    return false
  }

  if (!isAutoCompactEnabled()) return false

  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

---

## 7.6 autoCompactIfNeeded: Circuit Breaker + Session Memory Priority

[src/services/compact/autoCompact.ts:241-351](../../src/services/compact/autoCompact.ts#L241-L351)

### Circuit Breaker

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
```

When the context is irrecoverably past the limit (e.g., `prompt_too_long`), retrying every turn only wastes API calls. The circuit breaker stops trying after 3 consecutive failures, passing state between query loop turns via `AutoCompactTrackingState.consecutiveFailures`.

### Complete Execution Flow

```typescript
export async function autoCompactIfNeeded(
  messages,
  toolUseContext,
  cacheSafeParams,
  querySource?,
  tracking?,
  snipTokensFreed?,
): Promise<{ wasCompacted: boolean; compactionResult?: CompactionResult; consecutiveFailures?: number }>
```

Execution order:

```
1. Check DISABLE_COMPACT env var
2. Check circuit breaker (consecutiveFailures >= 3 → skip)
3. shouldAutoCompact() — recursion guard + token check
4. First attempt trySessionMemoryCompaction()   ← Try session memory compact first
   ├─ Success → notifyCompaction() + markPostCompaction() + return
   └─ Failure → proceed to main compact
5. compactConversation()                        ← Main compact
   ├─ Success → consecutiveFailures: 0 (reset)
   └─ Failure → consecutiveFailures: prevFailures + 1
```

**Key design**: Session Memory Compact is attempted before Full Compact because Session Memory is incremental trimming rather than a full rewrite, making it more Prompt Cache friendly.

---

## 7.7 Main Compaction Function: compactConversation

[src/services/compact/compact.ts:387-763](../../src/services/compact/compact.ts#L387-L763)

**Note**: `compactConversation` is a regular `async function`, returning `Promise<CompactionResult>` — not an async generator.

```typescript
export async function compactConversation(
  messages: Message[],
  context: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  suppressFollowUpQuestions: boolean,
  customInstructions?: string,
  isAutoCompact: boolean = false,
  recompactionInfo?: RecompactionInfo,
): Promise<CompactionResult>
```

### CompactionResult Interface

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage       // Compact boundary message
  summaryMessages: UserMessage[]      // Summary messages (1 message)
  attachments: AttachmentMessage[]    // Restored attachments
  hookResults: HookResultMessage[]    // SessionStart hook results
  messagesToKeep?: Message[]          // Original messages to keep (reactive/SM compact)
  userDisplayMessage?: string         // Message displayed to user
  preCompactTokenCount?: number
  postCompactTokenCount?: number      // Total token consumption of the compact API call (not result size)
  truePostCompactTokenCount?: number  // Actual token estimate of messages after compaction
  compactionUsage?: ReturnType<typeof getTokenUsage>
}
```

### Execution Steps in Detail

[src/services/compact/compact.ts:395-762](../../src/services/compact/compact.ts#L395-L762)

```
Step 1: Execute PreCompact hooks
  ├─ hookResult.newCustomInstructions merged into customInstructions
  └─ hookResult.userDisplayMessage saved for later

Step 2: stripImagesFromMessages(messages)
  └─ Replace images with [image] text placeholders, avoiding the compact request itself triggering prompt-too-long

Step 3: stripReinjectedAttachments(messages)
  └─ Filter skill_discovery/skill_listing attachments (will be re-injected after compact; no need to include in summary)

Step 4: PTL retry loop (up to MAX_PTL_RETRIES = 3 times)
  ├─ streamCompactSummary() — stream Claude to generate summary
  ├─ If summary starts with PROMPT_TOO_LONG_ERROR_MESSAGE:
  │   └─ truncateHeadForPTLRetry() truncates oldest API turns → retry
  └─ If retries exhausted → throw ERROR_MESSAGE_PROMPT_TOO_LONG

Step 5: Save preCompactReadFileState snapshot
  └─ preCompactReadFileState = cacheToObject(context.readFileState)
     !! Must be saved BEFORE readFileState.clear() !!

Step 6: Clear state caches
  ├─ context.readFileState.clear()
  ├─ context.loadedNestedMemoryPaths?.clear()
  └─ sentSkillNames intentionally NOT reset (re-injection benefit is minimal and wastes cache_creation)

Step 7: Build attachments in parallel
  ├─ createPostCompactFileAttachments(preCompactReadFileState, context, 5)
  │   └─ Uses Step 5 snapshot, up to 5 files, each ≤5K tokens
  └─ createAsyncAgentAttachmentsIfNeeded(context)

Step 8: Build additional attachments
  ├─ createPlanAttachmentIfNeeded()      — Current plan
  ├─ createPlanModeAttachmentIfNeeded()  — Plan mode instructions
  ├─ createSkillAttachmentIfNeeded()     — Invoked skills
  ├─ getDeferredToolsDeltaAttachment(tools, model, [], ...)  — Deferred tools (diff against [])
  ├─ getAgentListingDeltaAttachment(context, [])            — Agent listing (diff against [])
  └─ getMcpInstructionsDeltaAttachment(mcpClients, tools, model, [])  — MCP instructions (diff against [])

  Note: Passing empty [] as history means delta = "re-announce everything", ensuring the model knows all tools post-compact

Step 9: Execute SessionStart hooks (trigger='compact')
  └─ processSessionStartHooks('compact', { model }) → hookMessages

Step 10: Create boundaryMarker
  ├─ createCompactBoundaryMessage(isAutoCompact ? 'auto' : 'manual', preCompactTokenCount, lastMsgUuid)
  └─ Extract preCompactDiscoveredTools (list of loaded Deferred tool names) into compactMetadata
     Ensures schema filter continues to correctly send loaded Deferred tool schemas after compaction

Step 11: notifyCompaction() + markPostCompaction() + reAppendSessionMetadata()
  └─ Reset prompt cache baseline to avoid post-compact cache changes being misreported as cache breaks

Step 12: Execute PostCompact hooks
  └─ executePostCompactHooks({ trigger, compactSummary }, signal)

Return CompactionResult
```

---

## 7.8 buildPostCompactMessages

[src/services/compact/compact.ts:330-337](../../src/services/compact/compact.ts#L330-L337)

```typescript
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,
    ...result.summaryMessages,
    ...(result.messagesToKeep ?? []),
    ...result.attachments,
    ...result.hookResults,
  ]
}
```

**Note**: This is a very simple function that just assembles the fields of `CompactionResult` into a message array in a fixed order. The complex restoration logic (files, skills, delta attachments) happens inside `compactConversation()` and is already included in the `attachments` field.

Order: `boundaryMarker → summaryMessages → messagesToKeep → attachments → hookResults`

---

## 7.9 Image Removal (stripImagesFromMessages)

[src/services/compact/compact.ts:145-200](../../src/services/compact/compact.ts#L145-L200)

```typescript
export function stripImagesFromMessages(messages: Message[]): Message[] {
  return messages.map(message => {
    if (message.type !== 'user') return message  // Only process user messages

    const content = message.message.content
    if (!Array.isArray(content)) return message

    let hasMediaBlock = false
    const newContent = content.flatMap(block => {
      if (block.type === 'image') {
        hasMediaBlock = true
        return [{ type: 'text', text: '[image]' }]
      }
      if (block.type === 'document') {
        hasMediaBlock = true
        return [{ type: 'text', text: '[document]' }]
      }
      // Handle images nested inside tool_result
      if (block.type === 'tool_result' && Array.isArray(block.content)) {
        // ... same replacement with '[image]' / '[document]'
      }
      return [block]
    })

    if (!hasMediaBlock) return message
    return { ...message, message: { ...message.message, content: newContent } }
  })
}
```

**Why remove images?** CCD (Claude.ai Desktop) users frequently attach images. If the compact API call itself contains too many images, it triggers prompt-too-long errors. After removing images, only text placeholders remain, still letting Claude know "there was an image there."

---

## 7.10 stripReinjectedAttachments

[src/services/compact/compact.ts:211-223](../../src/services/compact/compact.ts#L211-L223)

```typescript
export function stripReinjectedAttachments(messages: Message[]): Message[] {
  if (feature('EXPERIMENTAL_SKILL_SEARCH')) {
    return messages.filter(
      m =>
        !(
          m.type === 'attachment' &&
          (m.attachment.type === 'skill_discovery' ||
            m.attachment.type === 'skill_listing')
        ),
    )
  }
  return messages
}
```

`skill_discovery` and `skill_listing` attachments will be re-injected via the delta mechanism in the next turn after compaction — no need to include them in the summary prompt and waste tokens.

---

## 7.11 PTL Retry Mechanism (truncateHeadForPTLRetry)

[src/services/compact/compact.ts:243-291](../../src/services/compact/compact.ts#L243-L291)

When the compact API request itself triggers `prompt-too-long` (CC-1180 scenario), the user can't be left stuck. Auto-retry is needed:

```
Up to MAX_PTL_RETRIES = 3 retries
  │
  ▼
truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
  ├─ Parse tokenGap from PTL error response
  ├─ Drop oldest API-round groups accumulated by tokenGap
  │   └─ If gap can't be parsed → drop 20% of groups proportionally
  ├─ Keep at least 1 group (avoid having nothing to summarize)
  └─ If first message after trimming is an assistant message → insert PTL_RETRY_MARKER user message at head
     (API requires first message role to be user)

  On consecutive retries, first detect and remove previously inserted PTL_RETRY_MARKER to avoid false group 0 affecting statistics
```

This is a "lossy but user-unblocking" escape hatch — losing the oldest context is better than being completely stuck.

---

## 7.12 Micro Compact

[src/services/compact/microCompact.ts:41-50](../../src/services/compact/microCompact.ts#L41-L50)

Micro compaction handles in-place compaction of individual large tool results, without rewriting the entire conversation history. Only applies to specific tools:

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,    // Read
  ...SHELL_TOOL_NAMES,    // Bash / Shell
  GREP_TOOL_NAME,         // Grep
  GLOB_TOOL_NAME,         // Glob
  WEB_SEARCH_TOOL_NAME,   // WebSearch
  WEB_FETCH_TOOL_NAME,    // WebFetch
  FILE_EDIT_TOOL_NAME,    // Edit
  FILE_WRITE_TOOL_NAME,   // Write
])
```

For each eligible tool call result that exceeds the token threshold, micro compaction replaces its content with `[Old tool result content cleared]`, freeing space without affecting the conversation structure.

---

## 7.13 Session Memory Compact

[src/services/compact/sessionMemoryCompact.ts](../../src/services/compact/sessionMemoryCompact.ts)

Session Memory Compact is a pre-candidate for Full Compact. `autoCompactIfNeeded()` tries `trySessionMemoryCompaction()` before calling `compactConversation()`:

- **Difference**: Session Memory is incremental trimming (deleting summarizable old messages), not a full rewrite.
- **More Prompt Cache friendly**: Preserves prefix structure, fewer cache misses.
- **Fallback on failure**: If Session Memory compaction is insufficient, proceed with Full Compact.
- **Consistent post-processing**: Both call `setLastSummarizedMessageId(undefined)` + `runPostCompactCleanup()` + `notifyCompaction()`.

---

## 7.14 Hook Integration

Compaction has three hook points:

```typescript
// PreCompact Hook: before compaction
await executePreCompactHooks({ trigger: 'auto' | 'manual', customInstructions }, signal)
// Users can:
// - Inject additional "don't compact this info" instructions (newCustomInstructions)
// - Log that compaction occurred
// - Inject user display messages (userDisplayMessage)

// SessionStart Hook: after summary generation, before boundaryMarker creation
await processSessionStartHooks('compact', { model })
// Result written to CompactionResult.hookResults as hookMessages

// PostCompact Hook: after compaction completes
await executePostCompactHooks({ trigger, compactSummary }, signal)
// Users can:
// - Inject restoration info (userDisplayMessage)
// - Notify external systems
```

---

## 7.15 Flowchart: Complete Auto Compaction Flow

```
Agent Loop turn ends
        │
        ▼
autoCompactIfNeeded()
        │
        ├─ DISABLE_COMPACT=true → skip
        ├─ consecutiveFailures >= 3 → circuit breaker skip
        │
        ▼
shouldAutoCompact()
        │
        ├─ querySource in [session_memory, compact, marble_origami] → false
        ├─ !isAutoCompactEnabled() → false
        ├─ tokenCount < autoCompactThreshold → false
        └─ true ↓
        │
        ▼
trySessionMemoryCompaction()
        │
   ┌────┴─────┐
 Success    Failure
   │           │
   ▼           ▼
notifyCompaction  compactConversation()
markPostCompaction    │
return wasCompacted   ▼
              [PreCompact hooks]
                      │
              stripImagesFromMessages()
              stripReinjectedAttachments()
                      │
              [PTL retry loop ≤3 times]
              streamCompactSummary()
                      │
              preCompactReadFileState = snapshot()
              readFileState.clear()
                      │
              [Parallel] createPostCompactFileAttachments
                         createAsyncAgentAttachments
                      │
              [Serial] plan/planMode/skill/deferred/agent/mcp attachments
                      │
              processSessionStartHooks('compact')
                      │
              createCompactBoundaryMessage()
              ├─ Write preCompactDiscoveredTools
                      │
              notifyCompaction()
              markPostCompaction()
              reAppendSessionMetadata()
                      │
              [PostCompact hooks]
                      │
              return CompactionResult
                      │
              ├─ Success: consecutiveFailures = 0
              └─ Failure: consecutiveFailures++
                      │
                      ▼
               Continue Agent Loop
```

---

## Summary

| Mechanism | Trigger | Core Operation | Source Location |
|-----------|---------|----------------|-----------------|
| Auto Compact decision | Each agent loop turn | `calculateTokenWarningState(tokenUsage, model)` | `autoCompact.ts` |
| Circuit breaker | ≥3 consecutive failures | Stop retrying, avoid wasting API calls | `autoCompact.ts:70` |
| Session Memory Compact | Before Full Compact | Incremental trimming, cache-friendly | `sessionMemoryCompact.ts` |
| Main compaction | Manual or auto | `compactConversation()` regular async function | `compact.ts:387` |
| PTL retry | Compact itself triggers prompt-too-long | `truncateHeadForPTLRetry()` ≤3 times | `compact.ts:243` |
| Image removal | Before compaction | Replace image/document with text placeholders | `stripImagesFromMessages()` |
| Attachment removal | Before compaction | Filter skill_discovery/listing | `stripReinjectedAttachments()` |
| File restoration | After compaction | Up to 5 files, each ≤5K tokens; uses pre-compact snapshot | `createPostCompactFileAttachments()` |
| sentSkillNames | After compaction | **Intentionally NOT reset**, avoids cache_creation waste | `compact.ts:526-530` |
| Delta attachment re-injection | After compaction | deferred/agent/mcp fully re-announced against `[]` baseline | `compact.ts:567-585` |
| preCompactDiscoveredTools | boundaryMarker | Persist list of loaded Deferred tools | `compact.ts:606-611` |
| Micro compaction | Individual large result | In-place replacement with placeholder text | `microCompact.ts` |
| Pre/Post Hook | Before/after compaction | User-defined extensions | `hooks.ts` |
