# Chapter 4: TodoWrite and the Task System

> Source location: `src/tools/TodoWriteTool/`, `src/tasks/`, `src/utils/tasks.ts`

---

## 4.1 Two Easily Confused Concepts

Claude Code has two related but distinct notions of "task":

| Concept | Description | Storage Location |
|---------|-------------|------------------|
| **Todo (task checklist)** | Work steps that Claude plans within the current session | `AppState.todos` |
| **Task (system task)** | Runtime-managed entities (subagents, shell commands, background sessions, etc.) | `AppState.tasks` |

This chapter covers TodoWrite (task checklist) first, then the Task System (system tasks).

---

## 4.2 TodoWrite: Claude's Work Planner

### 4.2.1 Data Model

```typescript
// src/utils/todo/types.ts
type TodoItem = {
  content: string                                    // Task description
  status: 'pending' | 'in_progress' | 'completed'
  activeForm: string                                 // Active form of the task (e.g., "Reading src/foo.ts")
}

type TodoList = TodoItem[]
```

### 4.2.2 Tool Implementation

[src/tools/TodoWriteTool/TodoWriteTool.ts:65-115](../../src/tools/TodoWriteTool/TodoWriteTool.ts#L65-L115)
```typescript
async call({ todos }, context) {
  const appState = context.getAppState()
  
  // Each agent has its own todo list (keyed by agentId or sessionId)
  const todoKey = context.agentId ?? getSessionId()
  const oldTodos = appState.todos[todoKey] ?? []
  
  // Key logic: if all todos are completed, AppState stores an empty []
  const allDone = todos.every(_ => _.status === 'completed')
  const newTodos = allDone ? [] : todos

  // Verification nudge: only triggered when allDone && 3+ items && no verification step
  let verificationNudgeNeeded = false
  if (
    feature('VERIFICATION_AGENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
    !context.agentId &&   // Only on main thread, not in subagents
    allDone &&            // Must be at the moment of closing the list
    todos.length >= 3 &&
    !todos.some(t => /verif/i.test(t.content))
  ) {
    verificationNudgeNeeded = true
  }

  // Note: API is setAppState (receives a prev → newState function), not updateAppState
  context.setAppState(prev => ({
    ...prev,
    todos: { ...prev.todos, [todoKey]: newTodos },
  }))

  // Return value: newTodos is the original todos input (not the cleared [])
  return {
    data: { oldTodos, newTodos: todos, verificationNudgeNeeded },
  }
}
```

### 4.2.3 Key Design Points

**1. Clear when all done**

When all todos are marked `completed`, the list in AppState is cleared to `[]` rather than retaining all completed items. This avoids wasting tokens (serializing completed tasks in every API call). Note: the `newTodos` field in the tool's return value still contains the original `todos` input; the clearing only affects the AppState storage layer.

**2. agentId isolation**

Each subagent has its own todo list (keyed by `agentId`). The main thread uses `sessionId`. This way, parallel subagents don't interfere with each other.

**3. shouldDefer: true**

TodoWrite is marked for deferred disclosure and is not directly included in every API request. Claude discovers it via ToolSearch.

**4. Mutually exclusive with TodoV2 (Task API)**

[src/tools/TodoWriteTool/TodoWriteTool.ts:53](../../src/tools/TodoWriteTool/TodoWriteTool.ts#L53):

```typescript
isEnabled() {
  return !isTodoV2Enabled()
}
```

[src/utils/tasks.ts:133](../../src/utils/tasks.ts#L133): `isTodoV2Enabled()` returns `true` (i.e., TodoWrite is disabled) when:
- Environment variable `CLAUDE_CODE_ENABLE_TASKS=1`
- Non-interactive session (SDK mode)

This means SDK users use the Task API by default, not TodoWrite.

**5. verificationNudgeNeeded**

This is a "behavioral nudge" mechanism. Trigger conditions (all must be met):
- Feature flag `VERIFICATION_AGENT` is enabled (compile-time gate)
- GrowthBook flag `tengu_hive_evidence` is true
- Currently on main thread (`!context.agentId`)
- **This call is closing the entire list (`allDone === true`)**
- Task count ≥ 3
- No task content matches `/verif/i`

When triggered, the tool result appends a prompt instructing Claude to spawn a verification agent before writing the final summary.

---

## 4.3 Task System: Runtime Entity Management

The Task System manages all "running entities" in Claude Code.

### 4.3.0 TaskStateBase: Common Base for All Tasks

[src/Task.ts:45](../../src/Task.ts#L45)

```typescript
type TaskStateBase = {
  id: string           // Task ID (with type prefix, see table below)
  type: TaskType       // 'local_bash' | 'local_agent' | 'remote_agent' | 'in_process_teammate' | 'local_workflow' | 'monitor_mcp' | 'dream'
  status: TaskStatus   // 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  description: string
  toolUseId?: string   // The tool_use block ID that triggered this task (for notification callbacks)
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string   // Disk output file path (snapshot; unchanged after /clear)
  outputOffset: number // Read byte offset (for incremental delta reads)
  notified: boolean    // Whether completion notification has been sent (to prevent duplicates)
}
```

**Task ID Prefix Table** ([src/Task.ts:79](../../src/Task.ts#L79)):

| Type | Prefix | Example |
|------|--------|---------|
| `local_bash` | `b` | `b3f9a2c1` |
| `local_agent` | `a` | `a7d8e4f2` |
| `remote_agent` | `r` | `r2a1b3c4` |
| `in_process_teammate` | `t` | `t5e6f7a8` |
| `local_workflow` | `w` | `w9b0c1d2` |
| `monitor_mcp` | `m` | `m3e4f5a6` |
| `dream` | `d` | `d7b8c9d0` |
| Main session (special) | `s` | `s1a2b3c4` |

> `local_agent` and main session backgrounding share the `a` prefix (`LocalAgentTaskState`), but main session backgrounding uses `s` prefix via the independent `generateMainSessionTaskId()` in `LocalMainSessionTask.ts`, distinguishing it from regular subagents.

### 4.3.1 TaskState Union Type

[src/tasks/types.ts](../../src/tasks/types.ts)

```typescript
// Note: the type is named TaskState, not Task
type TaskState =
  | LocalShellTaskState        // Long-running shell command
  | LocalAgentTaskState        // Local subagent (including backgrounded main session)
  | RemoteAgentTaskState       // Remote agent
  | InProcessTeammateTaskState // In-process teammate (team collaboration)
  | LocalWorkflowTaskState     // Workflow
  | MonitorMcpTaskState        // MCP monitor
  | DreamTaskState             // Memory consolidation subagent (auto-dream)
```

> **Note**: `LocalMainSessionTask` is **not** an independent union type member. It is a subtype of `LocalAgentTaskState`:
> ```typescript
> // src/tasks/LocalMainSessionTask.ts
> type LocalMainSessionTaskState = LocalAgentTaskState & { agentType: 'main-session' }
> ```
> This means backgrounded main sessions share the same `type: 'local_agent'` identifier as regular subagents, distinguished by the `agentType` field.

### 4.3.2 Main Session Backgrounding (LocalMainSessionTask)

This is a special feature: pressing **Ctrl+B twice** backgrounds the current conversation so the user can start a new one. The original query continues running independently in the background.

[src/tasks/LocalMainSessionTask.ts](../../src/tasks/LocalMainSessionTask.ts)

```typescript
// LocalMainSessionTaskState is not an independent type; it's a subtype of LocalAgentTaskState
type LocalMainSessionTaskState = LocalAgentTaskState & {
  agentType: 'main-session'  // The unique distinguishing identifier
}
// type: 'local_agent' (inherited)
// taskId: 's' prefix (distinguishes from regular agents' 'a' prefix)
// messages?: Message[] (inherited, stores background query message history)
// isBackgrounded: boolean (inherited; true = background, false = currently in foreground)
// No independent transcript field
```

Key functions:

```typescript
// Register a background session task, returns { taskId, abortSignal }
registerMainSessionTask(description, setAppState, agentDefinition?, abortController?)

// Start the actual background query (wraps runWithAgentContext + query())
startBackgroundSession({ messages, queryParams, description, setAppState })

// Bring a background task back to foreground
foregroundMainSessionTask(taskId, setAppState): Message[]
```

**Why does it need an isolated transcript path?** Background tasks use `getAgentTranscriptPath(agentId)` instead of the main session's path. If they used the same path, `/clear` would accidentally overwrite background session data. Background tasks create symlinks via `initTaskOutputAsSymlink()`, and `/clear` relinks without affecting the background task's history.

### 4.3.3 Local Agent Task (LocalAgentTask)

Each subagent corresponds to a `LocalAgentTaskState`:

[src/tasks/LocalAgentTask/LocalAgentTask.tsx:116](../../src/tasks/LocalAgentTask/LocalAgentTask.tsx#L116)

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string          // Subtype discriminator; 'main-session' = backgrounded main session
  model?: string
  abortController?: AbortController
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress   // Progress info (includes toolUseCount, tokenCount)
  retrieved: boolean         // Whether result has been retrieved
  messages?: Message[]       // Agent message history (for UI display)
  lastReportedToolCount: number
  lastReportedTokenCount: number
  isBackgrounded: boolean    // false = currently in foreground, true = running in background
  pendingMessages: string[]  // Messages queued from SendMessage, processed at tool turn boundaries
  retain: boolean            // UI holds this task (blocks eviction)
  diskLoaded: boolean        // Whether sidechain JSONL has been loaded from disk
  evictAfter?: number        // Eviction timestamp (set after task completes)
}
```

> **Note**: It's a common mistake to list `toolUseCount` as a direct field of `LocalAgentTaskState`; it's actually at `progress.toolUseCount`. The `status` field is inherited from `TaskStateBase` (`'running' | 'completed' | 'failed' | 'stopped'`).

Progress tracking uses dedicated helper functions:

```typescript
// src/tasks/LocalAgentTask/LocalAgentTask.tsx
createProgressTracker()           // Initialize ProgressTracker (tracks input/output tokens separately)
updateProgressFromMessage()       // Accumulate tokens and tool call count from assistant messages
getProgressUpdate()               // Generate AgentProgress snapshot (for UI consumption)
createActivityDescriptionResolver() // Generate human-readable descriptions via tool.getActivityDescription()
```

**Elegant token tracking design**: `ProgressTracker` separately stores `latestInputTokens` (Claude API cumulative value, take the latest) and `cumulativeOutputTokens` (accumulated per turn), avoiding double-counting.

### 4.3.4 In-process Teammate Task (InProcessTeammateTask)

This is the core data structure for Agent Teams:

[src/tasks/InProcessTeammateTask/types.ts](../../src/tasks/InProcessTeammateTask/types.ts)

```typescript
type TeammateIdentity = {
  agentId: string        // e.g., "researcher@my-team"
  agentName: string      // e.g., "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string  // Leader's sessionId
}

type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity        // Teammate identity (plain data stored in AppState)
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController         // Terminates the entire teammate
  currentWorkAbortController?: AbortController  // Aborts only the current turn
  awaitingPlanApproval: boolean             // Whether waiting for plan approval
  permissionMode: PermissionMode            // Independent permission mode (toggled with Shift+Tab)
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  messages?: Message[]              // For UI display (capped at 50, TEAMMATE_MESSAGES_UI_CAP)
  pendingUserMessages: string[]     // Queue of messages from user input while viewing this teammate
  isIdle: boolean                   // Whether in idle state (waiting for leader instructions)
  shutdownRequested: boolean        // Whether shutdown has been requested
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

> **Common misconception**: The `mailbox` field does not exist in `InProcessTeammateTaskState`. The communication mailbox between teammates is stored in the runtime context `teamContext.inProcessMailboxes` (in AsyncLocalStorage), not in AppState. `pendingUserMessages` is the queue of messages sent by the user from the UI to this teammate — different from the mailbox.

**Memory cap design**: The `messages` field is capped at 50 entries (`TEAMMATE_MESSAGES_UI_CAP`), truncated from the head when exceeded. The reason: in production, a whale session was observed spawning 292 agents with 36.8GB memory usage, the root cause being this field holding a second full copy of the message history.

### 4.3.5 Memory Consolidation Task (DreamTask)

[src/tasks/DreamTask/DreamTask.ts](../../src/tasks/DreamTask/DreamTask.ts)

```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'    // starting → updating (flips after first Edit/Write)
  sessionsReviewing: number          // Number of sessions being consolidated
  filesTouched: string[]             // Files touched by Edit/Write (incomplete; only pattern-matched ones)
  turns: DreamTurn[]                 // Assistant turns (tool calls collapsed to a count)
  abortController?: AbortController
  priorMtime: number                 // Timestamp for rollback of consolidationLock on kill
}
```

DreamTask is the UI surface layer for "auto-dream" memory consolidation. It doesn't change subagent run logic — it just makes the otherwise-invisible fork agent visible in the footer pill and Shift+Down dialog. The subagent runs on a 4-phase prompt (orient → gather → consolidate → prune), but DreamTask doesn't parse phases; it infers `phase` from tool call types.

---

## 4.4 Task Registration and Lifecycle Framework

[src/utils/task/framework.ts](../../src/utils/task/framework.ts)

### 4.4.1 registerTask: Registration and Restoration

```typescript
registerTask(task: TaskState, setAppState): void
```

There are two paths when registering a new task:

**New** (`existing === undefined`):
1. Write task to `AppState.tasks[task.id]`
2. Emit `task_started` event to the SDK event queue

**Restore/Replace** (`existing !== undefined`, e.g., `resumeAgentBackground`):
1. Merge and preserve the following UI state (to avoid flickering for panels the user is viewing):
   - `retain`: UI hold flag
   - `startTime`: panel ordering stability
   - `messages`: user just sent messages not yet persisted to disk
   - `diskLoaded`: avoid re-loading sidechain JSONL
   - `pendingMessages`: pending message queue
2. **Do not** emit `task_started` (prevents SDK double-counting)

### 4.4.2 Task Completion Notification (XML Format)

When a subagent/background task completes, `enqueuePendingNotification` pushes XML into the message queue:

```xml
<task_notification>
  <task_id>a7d8e4f2</task_id>
  <tool_use_id>toulu_01xxx</tool_use_id>  <!-- optional -->
  <output_file>/tmp/.../tasks/a7d8e4f2.output</output_file>
  <status>completed</status>
  <summary>Task "fix login bug" completed successfully</summary>
</task_notification>
```

This XML is injected as a `user` message into the messages array before the next API call, making the LLM aware that a background task has completed.

### 4.4.3 evictTerminalTask: Two-Level Eviction

Tasks are not immediately deleted from `AppState.tasks` upon completion. Instead, eviction happens in two levels:

```
Task completes → status='completed' + notified=true
  │
  ├─ If retain=true or evictAfter > Date.now()
  │    → Retain (UI is showing; wait for 30s grace period before eviction)
  │
  └─ Otherwise → Immediately delete from AppState.tasks (eager eviction)
               As a safety net, generateTaskAttachments() also evicts on next poll
```

`PANEL_GRACE_MS = 30_000` (30 seconds) is the display grace period for agent tasks in the coordinator panel, ensuring results are visible to the user before disappearing.

---

## 4.5 Background API Task Tools

Users (via Claude) can create and manage background tasks:

```typescript
// TaskCreateTool / TaskUpdateTool / TaskStopTool / TaskGetTool / TaskListTool
```

These tools allow Claude to create and monitor background tasks, enabling true async multi-tasking. Note: these Task API tools are only enabled in TodoV2 mode (i.e., `isTodoV2Enabled() === true`), mutually exclusive with TodoWrite.

---

## 4.6 Disk Management of Task Output

[src/utils/task/diskOutput.ts](../../src/utils/task/diskOutput.ts)

### 4.6.1 Output File Path

```typescript
// Note: path is not .claude/tasks/, but a session subdirectory under the project temp dir
getTaskOutputPath(taskId) → `{projectTempDir}/{sessionId}/tasks/{taskId}.output`
```

**Why include sessionId?** To prevent concurrent Claude Code sessions on the same project from clobbering each other's output files. The path is memoized on first call (`let _taskOutputDir`); when `/clear` triggers `regenerateSessionId()`, it doesn't recalculate, ensuring background tasks that survive `/clear` can still find their files.

### 4.6.2 DiskTaskOutput Write Queue

Task output is asynchronously written to disk via the `DiskTaskOutput` class:

```typescript
class DiskTaskOutput {
  append(content: string): void  // Enqueue; automatically triggers drain
  flush(): Promise<void>         // Wait for queue to empty
  cancel(): void                 // Discard queue (when task is killed)
}
```

**Core design points**:

1. **Write queue**: `#queue: string[]` flat array, consumed by a single drain loop; chunks can be GC'd immediately after writing, avoiding memory bloat from `.then()` chains holding references.

2. **5GB limit**: When exceeded, appends a truncation marker and stops writing:
   ```
   [output truncated: exceeded 5GB disk cap]
   ```

3. **O_NOFOLLOW safety**: On Unix, files are opened with the `O_NOFOLLOW` flag, preventing attackers in the sandbox from creating symlinks to redirect Claude Code writes to arbitrary host files.

4. **Transaction tracking**: All fire-and-forget async operations (`initTaskOutput`, `evictTaskOutput`, etc.) are registered in `_pendingOps: Set<Promise>`, so tests can wait for all to complete via `allSettled`, preventing ENOENT race conditions across tests.

### 4.6.3 Incremental Output Reading (OutputOffset)

`TaskStateBase.outputOffset` records the consumed byte offset for incremental reading:

```typescript
// Read only new content after fromOffset (up to 8MB)
getTaskOutputDelta(taskId, fromOffset): Promise<{ content: string; newOffset: number }>
```

`generateTaskAttachments()` in `framework.ts` calls `getTaskOutputDelta` on each poll, appending new content to the `task_status` attachment for the LLM, avoiding repeated loading of the full output file.

### 4.6.4 Key Function Comparison

| Function | Deletes disk file | Clears memory | Use case |
|----------|------------------|---------------|----------|
| `evictTaskOutput(taskId)` | **No** | Yes (flush then clear Map) | Task complete, result consumed |
| `cleanupTaskOutput(taskId)` | **Yes** | Yes | Full cleanup (testing, cancellation) |
| `flushTaskOutput(taskId)` | No | No | Ensure writes complete before reading |

---

## 4.7 Cron Scheduled Tasks

Via Cron tools, you can create tasks that run automatically on a schedule:

```typescript
// CronCreate / CronDelete / CronList tools
// Managed internally via src/utils/cron.ts

// Usage example (how Claude would call it):
// CronCreate({ schedule: '0 9 * * 1', command: '/review-pr' })
// → Automatically run /review-pr every Monday at 9 AM
```

---

## 4.8 Flowchart: Complete Task Lifecycle

```
User inputs complex task
      │
      ▼
Claude calls TodoWrite         ← Plan work steps
  todos: [
    { content: 'Read code', status: 'pending' },
    { content: 'Modify feature', status: 'pending' },
    { content: 'Run tests', status: 'pending' },
  ]
      │
      ▼
Claude begins executing the first step
TodoWrite: { status: 'in_progress' }  ← Mark in progress
      │
      ├─ If parallel work is needed:
      │    Agent({ subagent_type: 'general-purpose', ... })
      │         │
      │         ▼
      │    Create LocalAgentTaskState (id: 'a-xxxxxxxx', agentId same value)
      │    Subagent runs in isolated context
      │         │
      │         ▼
      │    Subagent has its own Todo list (isolated)
      │
      ▼
After each step completes:
TodoWrite: { status: 'completed' }    ← Mark complete
      │
      ▼
When all steps complete:
TodoWrite: todos.every(done) → Clear list []
      │
      ▼
Agent Loop detects no tool calls → Stop
```

---

## Summary

| Component | Responsibility | Source Location |
|-----------|----------------|-----------------|
| `TodoWriteTool` | In-session work plan tracking (interactive mode) | [src/tools/TodoWriteTool/TodoWriteTool.ts](../../src/tools/TodoWriteTool/TodoWriteTool.ts) |
| Task API tools | Background task creation and management (SDK/non-interactive mode) | `TaskCreateTool` / `TaskStopTool` etc. |
| `TaskStateBase` | Common fields for all tasks (including id, status, outputOffset) | [src/Task.ts](../../src/Task.ts) |
| `LocalMainSessionTask` | Main session backgrounding (`LocalAgentTaskState` subtype) | [src/tasks/LocalMainSessionTask.ts](../../src/tasks/LocalMainSessionTask.ts) |
| `LocalAgentTaskState` | Subagent lifecycle management | [src/tasks/LocalAgentTask/LocalAgentTask.tsx](../../src/tasks/LocalAgentTask/LocalAgentTask.tsx) |
| `InProcessTeammateTaskState` | Team teammate state (with 50-entry memory cap) | [src/tasks/InProcessTeammateTask/types.ts](../../src/tasks/InProcessTeammateTask/types.ts) |
| `DreamTaskState` | UI surface layer for memory consolidation subagent | [src/tasks/DreamTask/DreamTask.ts](../../src/tasks/DreamTask/DreamTask.ts) |
| Task framework | registerTask / evictTerminalTask / notification XML | [src/utils/task/framework.ts](../../src/utils/task/framework.ts) |
| Disk output management | DiskTaskOutput write queue / incremental reads / 5GB cap | [src/utils/task/diskOutput.ts](../../src/utils/task/diskOutput.ts) |
| Cron scheduled tasks | Automated periodic tasks | `CronCreate/Delete/List` tools |

**Key design conventions summary**:

| Convention | Description |
|------------|-------------|
| `LocalMainSessionTask` is not an independent type | It is `LocalAgentTaskState & { agentType: 'main-session' }` |
| TaskState (not Task) | The correct name for the union type |
| `setAppState` (not `updateAppState`) | The actual API for AppState updates |
| `toolUseCount` is inside `progress` | Not a direct field of `LocalAgentTaskState` |
| TodoWrite and Task API are mutually exclusive | Switched via `isTodoV2Enabled()` in `isEnabled()` |
