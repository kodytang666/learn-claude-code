# Chapter 3: Tool System — Tool Invocation, Permissions, and Dynamic Disclosure

> Source location: `src/Tool.ts`, `src/services/tools/`, `src/tools/` (56 tool implementations)

---

## 3.1 What Is the Tool System?

Claude Code's tool system is the implementation layer for "action capabilities." Claude expresses its intent through text (e.g., "I want to perform this operation"), and the tool system is responsible for:

1. **Parsing** Claude's `tool_use` blocks
2. **Permission checking**: Is execution allowed?
3. **Executing** the tool and obtaining results
4. **Returning results** to Claude (as `tool_result`)

---

## 3.2 Tool Definition Structure

Each tool is built using the `buildTool()` factory function, following a unified interface:

```typescript
// src/Tool.ts
export type ToolDef<TInput, TOutput> = {
  // ── Required fields ──────────────────────────────────────
  name: string                    // Tool name (e.g., 'Bash', 'Read')
  maxResultSizeChars: number      // Max result size in chars (oversized results are persisted to disk)
  
  description(): Promise<string>  // Tool description (short summary sent to API)
  prompt(): Promise<string>       // Detailed usage instructions (injected into system prompt)
  inputSchema: ZodSchema          // Input schema (Zod definition, lazily loaded)
  call(input, context): Promise<TOutput>  // Execution logic
  
  // ── Optional fields ──────────────────────────────────────
  searchHint?: string             // ToolSearch keyword matching description
  strict?: boolean                // Strict JSON Schema mode
  outputSchema?: ZodSchema        // Output schema
  
  isEnabled?(): boolean           // Whether enabled (default true, can be disabled by runtime conditions)
  isReadOnly?(): boolean          // Whether read-only (used to filter in plan mode)
  isDestructive?(): boolean       // Whether irreversible (affects permission warnings)
  isConcurrencySafe?(): boolean   // Whether safe to run concurrently with other tools (default false)

  shouldDefer?: boolean           // Whether to defer disclosure (Tool Search mechanism)
  alwaysLoad?: boolean            // Force load even if shouldDefer=true (highest priority)
  
  checkPermissions?(input): Promise<PermissionResult>  // Permission check (default: allow)
  
  interruptBehavior?(): 'cancel' | 'block'
  // Behavior when user sends a new message while tool is running:
  //   'cancel' — abort current tool execution
  //   'block'  — finish execution before processing new message
  
  userFacingName?(): string       // Name shown to user (empty string = don't show)
  renderToolUseMessage?(): React.ReactNode | null  // UI rendering
}
```

### Example: TodoWriteTool Definition

[src/tools/TodoWriteTool/TodoWriteTool.ts:31-54](../../src/tools/TodoWriteTool/TodoWriteTool.ts#L31-L54)
```typescript
export const TodoWriteTool = buildTool({
  name: TODO_WRITE_TOOL_NAME,
  searchHint: 'manage the session task checklist',
  maxResultSizeChars: 100_000,
  strict: true,
  
  async description() { return DESCRIPTION },
  async prompt() { return PROMPT },
  
  get inputSchema() { return inputSchema() },    // Lazily loaded schema
  get outputSchema() { return outputSchema() },
  
  userFacingName() { return '' },  // Empty string = don't show tool name in UI
  
  shouldDefer: true,    // Defer disclosure to Claude
  
  isEnabled() { return !isTodoV2Enabled() },  // Only enabled in v1 mode
  
  async checkPermissions(input) {
    return { behavior: 'allow', updatedInput: input }  // No permission confirmation needed
  },
})
```

---

## 3.3 Tool Execution Orchestration

[src/services/tools/toolOrchestration.ts](../../src/services/tools/toolOrchestration.ts)

An assistant message returned by Claude may contain multiple `tool_use` blocks. `runTools()` orchestrates their execution.

### Concurrency Strategy: Batching by isConcurrencySafe

Tools are not all executed concurrently. They are batched based on `isConcurrencySafe()` ([src/services/tools/toolOrchestration.ts:26](../../src/services/tools/toolOrchestration.ts#L26)):

```
Suppose Claude returns 4 tool calls: [Read, Bash, Edit, Read]
Where Edit.isConcurrencySafe() = false, rest = true

Batch result:
  Batch 1: [Read, Bash]  → concurrent execution
  Batch 2: [Edit]        → exclusive execution (waits for batch 1 to finish)
  Batch 3: [Read]        → concurrent execution (waits for batch 2 to finish)
```

**Rule**: Consecutive safe tools are merged into a batch for concurrent execution; any unsafe tool runs alone in a batch (serial).

**Concurrency limit** ([src/services/tools/toolOrchestration.ts:8](../../src/services/tools/toolOrchestration.ts#L8)):
```typescript
function getMaxToolUseConcurrency(): number {
  return parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
}
// Default max 10 concurrent tools; adjustable via env var
```

### StreamingToolExecutor

[src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts)

The core class for handling streaming tool calls (executing tools while Claude's output is still streaming). Manages a concurrency constraint queue:

```typescript
// Concurrency execution check (StreamingToolExecutor.ts)
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

**Sibling Abort Controller**: Each `StreamingToolExecutor` instance holds a child AbortController. When one concurrent tool errors, it immediately terminates all sibling tools (without terminating the entire query turn):

```typescript
// Child of toolUseContext.abortController.
// Fires when a Bash tool errors so sibling subprocesses die immediately
// instead of running to completion. Aborting this does NOT abort the parent.
private siblingAbortController: AbortController
```

### Complete Tool Execution Flow

```
Claude returns tool_use block
      │
      ▼
findToolByName()          ← Look up tool implementation
      │
      ▼
backfillObservableInput() ← Tool injects derived fields (doesn't affect call() raw input)
      │
      ▼
PreToolUse hooks          ← Can modify input, provide permission decision, inject context
      │
      ▼
Permission check (checkPermissions + canUseTool)
      ├─ allow → proceed
      ├─ deny  → return error tool_result, skip execution
      └─ ask   → pause and wait for user confirmation
      │
      ▼
call(input, toolUseContext)  ← Execute tool
      │
      ▼
applyToolResultBudget()   ← Result size limit (see 3.6)
      │
      ▼
PostToolUse hooks          ← Can modify output
      │
      ▼
Return tool_result to Claude
```

---

## 3.4 Permission System

Tools must pass permission checks before execution. This is the core of Claude Code's security model.

### Permission Check Function

```typescript
// src/hooks/useCanUseTool.tsx
export type CanUseToolFn = (
  toolName: string,
  input: unknown,
  context: ToolPermissionContext,
) => Promise<PermissionResult>
```

### Permission Result Types

```typescript
// src/utils/permissions/PermissionResult.ts
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: unknown }      // Allow (can modify input)
  | { behavior: 'deny'; message: string }              // Deny (with reason)
  | { behavior: 'ask'; prompt: string }                // Requires user confirmation
```

### Permission Modes

[src/types/permissions.ts](../../src/types/permissions.ts)

| Mode | Description |
|------|-------------|
| `default` | Standard permission check; requires user confirmation for dangerous operations |
| `acceptEdits` | Automatically accept file edit operations |
| `bypassPermissions` | Bypass all permission checks (automation/CI mode) |
| `dontAsk` | Deny all permission requests (read-only protection mode) |
| `plan` | Plan mode: allow reads, deny writes |
| `auto` | Auto classifier mode (requires TRANSCRIPT_CLASSIFIER feature gate) |

Note: The `bubble` mode mentioned in older documentation does not exist in the source code.

### Hooks Can Modify Permissions

`PreToolUse` Hooks can:
- Approve or deny tool calls
- Modify tool input parameters
- Add additional context

[src/types/hooks.ts:72-85](../../src/types/hooks.ts#L72-L85)
```typescript
z.object({
  hookEventName: z.literal('PreToolUse'),
  permissionDecision: permissionBehaviorSchema().optional(),  // approve/deny
  permissionDecisionReason: z.string().optional(),
  updatedInput: z.record(z.string(), z.unknown()).optional(),  // modify input
  additionalContext: z.string().optional(),                    // inject context
})
```

---

## 3.5 Dynamic Tool Disclosure (Deferred Tool Disclosure)

This is one of Claude Code's most elegant optimization designs. **Not all tools are included in the `tools` parameter on every API call.**

### Background

Claude Code has 56+ tools. If every API call passes the complete schemas for all tools, it would:
- Consume a large number of tokens (tool descriptions can be very long)
- Reduce Claude's "focus" (too many tools can cause confusion)

### Solution: shouldDefer Flag

```typescript
// In tool definition
shouldDefer: true   // This tool is not directly disclosed to Claude
```

Tools marked `shouldDefer: true` **do not** appear in the `tools` array of each API request.

### ToolSearch Mechanism

When Claude wants to use a deferred tool, it first searches for it via the `ToolSearch` tool:

```
Claude wants to use a capability (e.g., TodoWrite)
        │
        ▼
Claude calls ToolSearch("manage task checklist")
        │
        ▼
ToolSearch returns matching tool names and descriptions
        │
        ▼
Claude now knows the tool name and can call it
```

```typescript
// src/utils/toolSearch.ts
export function isToolSearchEnabled(): boolean { ... }
export function isDeferredToolsDeltaEnabled(): boolean { ... }

// API response processing: extract discovered tool names
export function extractDiscoveredToolNames(response): string[] { ... }
```

### extractDiscoveredToolNames: Dynamic Tool Loading

After the ToolSearch tool is called, its result contains `tool_reference` blocks. `extractDiscoveredToolNames()` scans message history to extract all tool names that have been discovered ([src/utils/toolSearch.ts:545](../../src/utils/toolSearch.ts#L545)):

```typescript
export function extractDiscoveredToolNames(messages: Message[]): Set<string> {
  const discoveredTools = new Set<string>()
  for (const msg of messages) {
    // compact boundary restores previously discovered tools
    if (msg.type === 'system' && msg.subtype === 'compact_boundary') {
      const carried = msg.compactMetadata?.preCompactDiscoveredTools
      if (carried) for (const name of carried) discoveredTools.add(name)
    }
    // tool_reference blocks in tool_result
    for (const block of ...) {
      if (isToolReferenceWithName(block)) discoveredTools.add(block.tool_name)
    }
  }
  return discoveredTools
}
```

**Purpose**: Only discovered tools are included in subsequent API requests' `tools` array. This solves the problem of too many MCP tools — 100 MCP tools don't all get sent to the model at the start of a conversation.

When compaction occurs, `preCompactDiscoveredTools` is written to the compact boundary message, ensuring discovered tools are not lost after compaction.

### Deferred Tool Hint in Attachments

```typescript
// src/utils/attachments.ts
getDeferredToolsDeltaAttachment(toolUseContext)
// Returns a message telling Claude:
// "The following additional tools can be discovered via ToolSearch: [tool summary list]"
```

### Dynamic Disclosure Flowchart

```
API request construction phase:
┌─────────────────────────────────────────┐
│  Full tool list (56+)                    │
│                                         │
│  Directly disclosed (tools parameter):  │
│  ├─ Bash                                │
│  ├─ FileRead                            │
│  ├─ FileEdit                            │
│  ├─ Agent                               │
│  ├─ Skill                               │
│  ├─ ToolSearch                          │
│  └─ ... (tools not marked shouldDefer)  │
│                                         │
│  Deferred disclosure (attachment hint): │
│  ├─ TodoWrite    (shouldDefer: true)    │
│  ├─ TaskCreate   (shouldDefer: true)    │
│  ├─ EnterWorktree (shouldDefer: true)   │
│  └─ ...                                 │
└─────────────────────────────────────────┘
```

---

## 3.6 Tool Result Size Limits and Persistence

Tool results can be very large (Bash output, file reads, etc.). Claude Code has a three-layer budget mechanism ([src/utils/toolResultStorage.ts](../../src/utils/toolResultStorage.ts)):

**Layer 1: Per-tool declared limit**

Each tool declares its own limit via `maxResultSizeChars`:
- `TodoWriteTool.maxResultSizeChars = 100_000`
- `ReadTool.maxResultSizeChars = Infinity` (Read tool has its own maxTokens, doesn't use persistence)
- Default limit is 50,000 characters

The GrowthBook feature flag `tengu_satin_quoll` can dynamically override thresholds per tool name. Final threshold: `Math.min(declared value, 50k default)`.

**Layer 2: Per-result persistence**

Results exceeding the threshold are written to disk ([src/utils/toolResultStorage.ts:272](../../src/utils/toolResultStorage.ts#L272)), and the original content is replaced with a reference string:

```
<persisted-output>
Output too large (26KB). Full output saved to: /path/to/session/tool-results/xxx.txt

Preview (first 2KB):
[First 2000 bytes of content preview]
</persisted-output>
```

Claude sees this reference and can retrieve the full content via the Read tool when needed.

**Layer 3: Per-message aggregate budget**

The total size of all tool results in a single message is capped at 200,000 characters ([src/utils/toolResultStorage.ts:421](../../src/utils/toolResultStorage.ts#L421)). When multiple tools run concurrently and exceed this, the excess is proportionally compressed.

---

## 3.7 Tool Type Classification

| Category | Tool Names | Description |
|----------|-----------|-------------|
| **File operations** | FileRead, FileEdit, FileWrite, Glob, Grep | Read, write, and search files |
| **Shell** | Bash | Execute shell commands |
| **Network** | WebSearch, WebFetch | Search and fetch web pages |
| **Agent** | Agent, SendMessage | Create subagents, send messages |
| **Task** | TodoWrite, TaskCreate, TaskUpdate, TaskStop, TaskGet, TaskList | Task management |
| **Skill** | Skill, ToolSearch | Invoke skills, search tools |
| **Team** | TeamCreate, TeamDelete | Create agent teams |
| **Worktree** | EnterWorktree, ExitWorktree | Git Worktree management |
| **Planning** | EnterPlanMode, ExitPlanMode | Plan mode |
| **Cron** | CronCreate, CronDelete, CronList | Scheduled tasks |
| **Memory** | NotebookEdit, ReadMcpResource | Notebook, MCP resources |
| **Special** | Sleep, AskUserQuestion, RemoteTrigger | Sleep, prompt user, remote trigger |

---

## 3.8 Tool Context (ToolUseContext)

[src/Tool.ts:158](../../src/Tool.ts#L158)

All tools' `call(input, toolUseContext)` receive this context object:

```typescript
type ToolUseContext = {
  // ── Configuration ──────────────────────────────────────
  options: {
    commands: Command[]
    tools: Tools              // Currently available tool list (note: inside options)
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }
  
  // ── State read/write ────────────────────────────────
  getAppState(): AppState     // Read global state (permission context, task list, etc.)
  setAppState(f): void        // Atomically update state (note: not updateAppState)
  readFileState: FileStateCache  // File read cache (LRU)
  
  // ── Cancellation control ────────────────────────────
  abortController: AbortController  // For aborting the entire query
  
  // ── Session identification ────────────────────────
  agentId?: AgentId           // Subagents only; main thread uses getSessionId()
  // Note: no sessionId field; call getSessionId() function instead
  
  // ── Prompt Cache optimization ───────────────────────
  renderedSystemPrompt?: string  // Parent agent's rendered system prompt bytes (fork scenario)
  queryTracking: { chainId, depth }  // Query chain tracking (analytics)
  
  // ── Permissions ────────────────────────────────────
  // Note: canUseTool and permissionMode are NOT in ToolUseContext
  // Permission mode is accessed via toolUseContext.getAppState().toolPermissionContext.mode
  
  // ── UI callbacks (optional, REPL mode only) ────────
  setToolJSX?: SetToolJSXFn
  appendSystemMessage?: (msg) => void
  sendOSNotification?: (opts) => void
}
```

**Common misconceptions**:
- `permissionMode` is not a direct field of `ToolUseContext`; access it via `getAppState().toolPermissionContext.mode`
- `canUseTool` is a parameter of `runTools()`, not in `ToolUseContext`
- The `sessionId` field does not exist; call the module-level `getSessionId()` function

---

## Summary

```
Complete tool execution flow:

Claude returns tool_use
     │
     ▼
findToolByName()      ← Look up tool implementation
     │
     ▼
isEnabled()?          ← Is tool enabled in current mode?
     │
     ▼
checkPermissions()    ← Permission check (can be intercepted by PreToolUse Hook)
     │
     ├─ allow → call()  → Execute tool
     ├─ deny  → Return error tool_result
     └─ ask   → Pause, wait for user confirmation
               │
               ▼
          applyToolResultBudget()   ← Result size limit
               │
               ▼
          Return tool_result to Claude
```

| Concept | Source Location |
|---------|-----------------|
| Tool definition interface | [src/Tool.ts](../../src/Tool.ts) |
| Tool registry list | [src/tools.ts](../../src/tools.ts) |
| Tool execution orchestration (with batching concurrency strategy) | [src/services/tools/toolOrchestration.ts](../../src/services/tools/toolOrchestration.ts) |
| Streaming tool execution (with sibling abort) | [src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts) |
| Permission check function | [src/hooks/useCanUseTool.tsx](../../src/hooks/useCanUseTool.tsx) |
| Permission mode definitions | [src/types/permissions.ts](../../src/types/permissions.ts) |
| Deferred tool disclosure + dynamic tool loading | [src/utils/toolSearch.ts](../../src/utils/toolSearch.ts) |
| Result persistence (three-layer budget) | [src/utils/toolResultStorage.ts](../../src/utils/toolResultStorage.ts) |
