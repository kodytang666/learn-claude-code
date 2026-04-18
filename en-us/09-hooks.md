# Chapter 9: Hook Mechanism — 27 Event Hooks and Extension Protocol

> Source location: [src/types/hooks.ts](../../src/types/hooks.ts), [src/utils/hooks.ts](../../src/utils/hooks.ts), [src/utils/hooks/](../../src/utils/hooks/)

---

## 9.1 What Are Hooks?

Hooks are the **extension point system** Claude Code provides to users. They allow users to inject custom logic at critical nodes in the Agent lifecycle:

- Pre-tool-call checks (PreToolUse)
- Post-tool-call processing (PostToolUse)
- Session startup initialization (SessionStart)
- Custom operations before/after context compaction (PreCompact / PostCompact)
- ...27 event types in total

**Implementation mechanism**: Hooks are essentially **executable commands**. Claude Code executes these commands when specific events occur, passes JSON-formatted event data as input, and reads the command's stdout/stderr and exit code as the response.

---

## 9.2 Complete Hook Event List (27 Types)

[src/entrypoints/sdk/coreTypes.ts:25-53](../../src/entrypoints/sdk/coreTypes.ts#L25-L53)

```typescript
export const HOOK_EVENTS = [
  // ── Tool-related ──
  'PreToolUse',          // Before tool invocation (strongest interception point)
  'PostToolUse',         // After tool invocation (success)
  'PostToolUseFailure',  // After tool invocation failure
  'PermissionDenied',    // After auto-mode classifier denies permission

  // ── User interaction ──
  'UserPromptSubmit',    // When user submits a prompt

  // ── Session lifecycle ──
  'SessionStart',        // On new session/resume/clear/compact
  'SessionEnd',          // When session ends (clear/logout/exit, etc.) — different from Stop!
  'Stop',                // When Claude is about to finish its current response
  'StopFailure',         // When ending due to API error (rate limit/auth/billing, etc.) (fire-and-forget)
  'Setup',               // On repository initialization/maintenance (trigger: 'init' | 'maintenance')

  // ── Subagent-related ──
  'SubagentStart',       // When subagent starts
  'SubagentStop',        // When subagent is about to finish its response
  'TeammateIdle',        // When teammate is about to enter idle state (can be blocked)

  // ── Task-related ──
  'TaskCreated',         // When task is created (can be blocked)
  'TaskCompleted',       // When task is marked complete (can be blocked)

  // ── Permission/selection-related ──
  'PermissionRequest',   // When tool needs permission confirmation (can replace UI dialog)
  'Notification',        // When Claude issues a notification

  // ── MCP Elicitation ──
  'Elicitation',         // When MCP server requests user input
  'ElicitationResult',   // After user responds to MCP elicitation

  // ── Configuration ──
  'ConfigChange',        // When config file changes during session (can be blocked)
  'InstructionsLoaded',  // When CLAUDE.md or rules file is loaded (observe-only, cannot block)

  // ── Compaction ──
  'PreCompact',          // Before compaction (trigger: 'manual' | 'auto')
  'PostCompact',         // After compaction

  // ── Worktree ──
  'WorktreeCreate',      // When worktree is created (can completely replace default implementation)
  'WorktreeRemove',      // When worktree is removed

  // ── File/path ──
  'CwdChanged',          // After working directory changes
  'FileChanged',         // When a monitored file changes (must first register via SessionStart/CwdChanged)
] as const
```

> **Stop vs SessionEnd**: `Stop` triggers when Claude normally finishes its current response (exit code 2 lets Claude continue running); `SessionEnd` triggers when the entire session closes (clear/logout/exit) with strict timeout limits (default 1500ms).

---

## 9.3 Hook Types (4 Configurable Types)

[src/utils/hooks/hooksSettings.ts:44-64](../../src/utils/hooks/hooksSettings.ts#L44-L64)

```typescript
type HookCommand =
  | { type: 'command'; command: string; shell?: string; if?: string; timeout?: number }
  | { type: 'prompt';  prompt: string;  if?: string; timeout?: number }   // AI-driven: let Claude handle hook logic
  | { type: 'agent';   prompt: string;  if?: string; timeout?: number }   // Agent type
  | { type: 'http';    url: string;     if?: string; timeout?: number }   // HTTP request
  // 'function' and 'callback' are SDK-internal types; cannot be configured in settings files
```

The most commonly used is the `command` type. The `prompt` type uses a Claude instance to decide whether to allow an operation (suitable for complex semantic decisions).

**`if` condition field**: Each hook can set an `if` condition (e.g., `"Bash(git *)"` matcher). Together with `matcher`, this forms the hook's identity — the same command string with different `if` conditions are two independent hooks.

---

## 9.4 Hook Configuration Format

Hook configuration in `.claude/settings.json` (project), `~/.claude/settings.json` (user), or `.claude/settings.local.json` (local):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/my-bash-checker.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Configuration sources (3 places)**:
- `userSettings`: `~/.claude/settings.json`
- `projectSettings`: `.claude/settings.json`
- `localSettings`: `.claude/settings.local.json` (not committed to git; personal overrides)

There are also `sessionHook` (dynamically registered via SessionStart hook response) and built-in `builtinHook`.

---

## 9.5 Exit Code Protocol

Each event type responds differently to exit codes. This is the most critical behavioral specification in the entire hook system:

[src/utils/hooks/hooksConfigManager.ts:26-264](../../src/utils/hooks/hooksConfigManager.ts#L26-L264)

| Event | Exit 0 | Exit 2 | Other |
|-------|--------|--------|-------|
| `PreToolUse` | stdout/stderr not shown | stderr → model, **block tool call** | stderr → user, continue call |
| `PostToolUse` | stdout shown (transcript mode ctrl+O) | stderr → model (injected immediately) | stderr → user |
| `PostToolUseFailure` | stdout shown (transcript mode) | stderr → model (immediately) | stderr → user |
| `UserPromptSubmit` | stdout → Claude (appended) | **Block processing + clear original prompt** + stderr → user | stderr → user |
| `SessionStart` | stdout → Claude; blocking errors ignored | — | stderr → user |
| `SessionEnd` | Normal completion | — | stderr → user |
| `Stop` | stdout/stderr not shown | stderr → model, **Claude continues running** | stderr → user |
| `StopFailure` | — | — | All ignored (fire-and-forget) |
| `Setup` | stdout → Claude; blocking errors ignored | — | stderr → user |
| `SubagentStop` | stdout/stderr not shown | stderr → subagent, **subagent continues running** | stderr → user |
| `PreCompact` | stdout appended as custom compact instruction | **Block compaction** | stderr → user, continue compact |
| `PermissionDenied` | stdout shown (transcript mode) | — | stderr → user |
| `ConfigChange` | Allow change | **Block change application** | stderr → user |
| `TeammateIdle` | stdout/stderr not shown | stderr → teammate, **prevent idle** (teammate continues working) | stderr → user |
| `TaskCreated` | stdout/stderr not shown | stderr → model, **block task creation** | stderr → user |
| `TaskCompleted` | stdout/stderr not shown | stderr → model, **block task completion** | stderr → user |
| `WorktreeCreate` | Worktree creation success (stdout=path) | — | Creation fails |
| `WorktreeRemove` | Removal success | — | stderr → user |
| `InstructionsLoaded` | Normal completion (**cannot be blocked**) | — | stderr → user |
| `CwdChanged` / `FileChanged` | Normal completion | — | stderr → user |

---

## 9.6 Hook Response Protocol (JSON Output)

Hook command stdout returns JSON (optional):

[src/types/hooks.ts:50-165](../../src/types/hooks.ts#L50-L165)

```typescript
// Sync response (most hooks)
type SyncHookResponse = {
  // ── Common fields ──
  continue?: boolean          // false = stop current operation (equivalent to exit 2 block)
  suppressOutput?: boolean    // Hide stdout from user display
  stopReason?: string         // Reason shown when continue=false
  decision?: 'approve' | 'block'  // Quick permission decision (no need to write hookSpecificOutput)
  reason?: string             // Decision explanation
  systemMessage?: string      // System warning message (shown to user)

  // ── Event-specific output ──
  hookSpecificOutput?: PreToolUseOutput | PostToolUseOutput | ...
}

// Async response (return immediately; hook continues running in background)
type AsyncHookResponse = {
  async: true
  asyncTimeout?: number  // Async timeout (milliseconds)
}
```

### Event-Specific Output (hookSpecificOutput)

[src/types/hooks.ts:71-163](../../src/types/hooks.ts#L71-L163)

**PreToolUse** (most powerful interception point):
```typescript
{
  hookEventName: 'PreToolUse',
  permissionDecision?: 'allow' | 'deny' | 'ask',  // Override permission decision
  permissionDecisionReason?: string,
  updatedInput?: Record<string, unknown>,           // Modify tool input parameters!
  additionalContext?: string,                        // Inject additional context
}
```

**PostToolUse**:
```typescript
{
  hookEventName: 'PostToolUse',
  additionalContext?: string,
  updatedMCPToolOutput?: unknown,  // Modify MCP tool output result
}
```

**UserPromptSubmit**:
```typescript
{
  hookEventName: 'UserPromptSubmit',
  additionalContext?: string,  // Append info to end of prompt
}
```

**SessionStart** (register file watching):
```typescript
{
  hookEventName: 'SessionStart',
  additionalContext?: string,
  initialUserMessage?: string,    // Auto-sent first message (simulates user input)
  watchPaths?: string[],          // Register file change watch paths (triggers FileChanged)
}
```

**PermissionDenied** (allow retry):
```typescript
{
  hookEventName: 'PermissionDenied',
  retry?: boolean,  // true = tell model this tool call can be retried
}
```

**PermissionRequest** (programmatic permission control):
```typescript
{
  hookEventName: 'PermissionRequest',
  decision:
    | { behavior: 'allow'; updatedInput?: ...; updatedPermissions?: ... }
    | { behavior: 'deny'; message?: string; interrupt?: boolean }
}
```

**CwdChanged / FileChanged** (dynamically update watch paths):
```typescript
{
  hookEventName: 'CwdChanged' | 'FileChanged',
  watchPaths?: string[],  // Dynamically update FileChanged watch path list
}
```

**WorktreeCreate** (return path):
```typescript
{
  hookEventName: 'WorktreeCreate',
  worktreePath: string,  // Absolute path of worktree created by hook
}
```

---

## 9.7 Common Input Fields for All Hooks

[src/utils/hooks.ts:301-328](../../src/utils/hooks.ts#L301-L328)

Every hook receives JSON input containing the following base fields (generated by `createBaseHookInput()`):

```typescript
{
  session_id: string,       // Current session ID
  transcript_path: string,  // Session transcript file path (can be used to read full conversation history)
  cwd: string,              // Current working directory
  permission_mode?: string, // Permission mode
  agent_id?: string,        // Subagent ID (undefined when called from main thread)
  agent_type?: string,      // Agent type
}
```

On top of base fields, each event appends its own specific fields:

```typescript
// PreToolUse additional fields
{ tool_name: string; tool_input: unknown; tool_use_id: string }

// PostToolUse additional fields
{ tool_name: string; tool_input: unknown; tool_use_id: string; tool_response: unknown }

// SessionStart additional fields
{ source: 'startup' | 'resume' | 'clear' | 'compact' }

// Setup additional fields
{ trigger: 'init' | 'maintenance' }

// PreCompact / PostCompact additional fields
{ trigger: 'manual' | 'auto'; ... }

// SubagentStart / SubagentStop additional fields
{ agent_id: string; agent_type: string }

// SessionEnd additional fields
{ reason: 'clear' | 'logout' | 'prompt_input_exit' | 'other' }

// StopFailure additional fields
{ error: 'rate_limit' | 'authentication_failed' | 'billing_error' | ... }

// ConfigChange additional fields
{ source: 'user_settings' | 'project_settings' | 'local_settings' | ...; file_path: string }

// InstructionsLoaded additional fields
{ file_path, memory_type, load_reason, globs?, trigger_file_path?, parent_file_path? }

// FileChanged additional fields
{ file_path: string; event: 'change' | 'add' | 'unlink' }
```

**CLAUDE_ENV_FILE**: When `CwdChanged` and `FileChanged` hooks execute, the `CLAUDE_ENV_FILE` environment variable points to a temp file. Hooks can write `export VAR=val` bash variable assignments to it, affecting the environment of subsequent BashTool commands.

---

## 9.8 getAllHooks and Hook Source Aggregation

[src/utils/hooks/hooksSettings.ts:92-161](../../src/utils/hooks/hooksSettings.ts#L92-L161)

```typescript
export function getAllHooks(appState: AppState): IndividualHookConfig[]
```

**Note**: `getAllHooks` accepts `appState`, not `HookEvent`. It aggregates hook configurations from all sources:

```
getAllHooks(appState)
      │
      ├─ policySettings.allowManagedHooksOnly?
      │   YES → skip user/project/local (managed hooks are hidden from UI)
      │   NO  → continue
      │
      ├─ userSettings hooks
      ├─ projectSettings hooks
      ├─ localSettings hooks
      │   (dedup: same file path not processed twice)
      │
      └─ sessionHooks (dynamically registered via SessionStart hook response)
```

Filter by event using `getHooksForEvent(appState, event)`.

---

## 9.9 Security Mechanism: Trust Check

[src/utils/hooks.ts:286-296](../../src/utils/hooks.ts#L286-L296)

```typescript
export function shouldSkipHookDueToTrust(): boolean {
  const isInteractive = !getIsNonInteractiveSession()
  if (!isInteractive) return false  // SDK mode: trust is implicit; execute directly

  // Interactive mode: all hooks must wait for workspace trust confirmation
  const hasTrust = checkHasTrustDialogAccepted()
  return !hasTrust
}
```

**All hooks require workspace trust**. Before trust is confirmed, no hook will execute — including `SessionEnd` and `SubagentStop`. This defensive measure was added after discovering a historical vulnerability (SessionEnd hook executed even when user rejected the trust dialog).

---

## 9.10 Detailed Behavior of Important Hooks

### PreToolUse: The Most Powerful Interception Point

```
Trigger: Before every tool invocation
Special capabilities:
1. Reject tool calls (exit 2, continue: false, or permissionDecision: 'deny')
2. Modify tool parameters (updatedInput) — tool receives the modified parameters
3. Inject additional context (additionalContext) — inject context messages into Claude
4. Record audit logs

Typical use cases:
- Block Bash commands containing "rm -rf /"
- Backup before all file writes
- Run lint before git push
```

### SessionStart: Session Initialization

```
Trigger: startup / resume / clear / compact (any of the four)
Special capabilities:
1. Set initial user message (initialUserMessage)
   → Makes Claude execute a task immediately on startup (auto-sends first message)

2. Register file watch paths (watchPaths)
   → Changes to these paths trigger the FileChanged hook
   → Enables "automatically notify Claude when files change"

3. Inject additional context (additionalContext)
   → E.g., inject environment info, project state, etc.
```

### Stop vs SessionEnd

```
Stop: Claude finishes its current response (normal completion of tool call chain)
  └─ exit 2 → stderr injected into model; Claude continues running
  └─ Typical use: verify task is truly complete

SessionEnd: Entire session closes (/clear, /logout, closing terminal)
  └─ Timeout limit: default 1500ms (override via CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS)
  └─ Typical use: save session state, send analytics data
  └─ Can match by reason: clear / logout / prompt_input_exit / other
```

### PermissionDenied + retry

```
Trigger: After auto-mode classifier denies a tool call
Special capability:
  Return {"hookSpecificOutput":{"hookEventName":"PermissionDenied","retry":true}}
  → Tells the model "this tool call can be retried"
  → Suitable for scenarios needing additional context to determine whether to allow
```

### InstructionsLoaded: Read-Only Observation

```
Trigger: When CLAUDE.md or rules file is loaded
load_reason enum:
  session_start, nested_traversal, path_glob_match, include, compact
Special: This hook is observation-only; exit codes cannot block loading
Use case: Logging, security auditing of which CLAUDE.md files are loaded
```

### ConfigChange: Runtime Config Change Interception

```
Trigger: Config file changes while session is running (hot reload)
source enum: user_settings, project_settings, local_settings, policy_settings, skills
exit 2 → block this change from applying to the current session
Use case: Prevent unauthorized config modifications from taking effect in a running session
```

---

## 9.11 Prompt Elicitation Protocol

This is a special mechanism allowing hook scripts to request additional input from the user and wait for a response:

[src/types/hooks.ts:28-47](../../src/types/hooks.ts#L28-L47)

```typescript
// Hook script stdout outputs this JSON; Claude Code shows a selection UI
type PromptRequest = {
  prompt: string   // Request ID (for matching subsequent response)
  message: string  // Question text displayed to user
  options: Array<{
    key: string
    label: string
    description?: string
  }>
}

// Claude Code sends user's selection to hook as stdin, or via ElicitationResult event
type PromptResponse = {
  prompt_response: string  // Request ID (matches PromptRequest.prompt)
  selected: string         // Key selected by user
}
```

This allows hook scripts to implement interactive prompts like "Choose deployment environment: [staging] [production]".

---

## 9.12 ALWAYS_EMITTED_HOOK_EVENTS

[src/utils/hooks/hookEvents.ts:18](../../src/utils/hooks/hookEvents.ts#L18)

```typescript
const ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup'] as const
```

In SDK mode, users can control which hook events are emitted via the `includeHookEvents` option. But `SessionStart` and `Setup` are always emitted, regardless of this option — they are low-noise lifecycle events that maintain backward compatibility.

---

## 9.13 Practical Use Cases

### Case 1: Automated Security Check (PreToolUse + exit 2)

```bash
#!/bin/bash
# pre-bash-check.sh

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# Block dangerous commands
if echo "$COMMAND" | grep -qE "rm -rf /|sudo rm|drop table"; then
  echo "Dangerous command detected, blocked" >&2
  exit 2  # exit 2 → stderr injected into model, block call
fi

exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{ "type": "command", "command": "/scripts/pre-bash-check.sh" }]
    }]
  }
}
```

### Case 2: Modify Tool Input Parameters (PreToolUse + updatedInput)

```bash
#!/bin/bash
# inject-readonly.sh — add --dry-run flag to all Bash commands
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# Return modified input
echo "{
  \"hookSpecificOutput\": {
    \"hookEventName\": \"PreToolUse\",
    \"updatedInput\": {
      \"command\": \"$CMD --dry-run\"
    }
  }
}"
```

### Case 3: SessionStart Register File Watching + Auto Message

```bash
#!/bin/bash
# session-init.sh
PROJECT_STATUS=$(cat .project-status.json 2>/dev/null || echo '{}')

echo "{
  \"hookSpecificOutput\": {
    \"hookEventName\": \"SessionStart\",
    \"additionalContext\": \"Project status: $PROJECT_STATUS\",
    \"watchPaths\": [\"$(pwd)/.project-status.json\"],
    \"initialUserMessage\": \"Please review the project status and report the current progress.\"
  }
}"
```

### Case 4: Stop Hook to Verify Task Completion

```bash
#!/bin/bash
# check-completion.sh — confirm Claude truly completed the task
if ! [ -f ".task-completed" ]; then
  echo "Task completion marker missing, please confirm all steps have been executed" >&2
  exit 2  # exit 2 → injected into model; Claude continues running
fi
exit 0
```

---

## 9.14 Flowchart: Complete PreToolUse Hook Flow

```
Claude decides to invoke a tool (e.g., Bash)
            │
            ▼
    canUseTool() called
            │
            ▼
    Execute PreToolUse Hooks (filtered by matcher)
            │
       ┌────┴────┐
  Matching hook  No matching hook
       │           │
       ▼           ▼
  Execute command  Default permission check
       │
  parseHookOutput(stdout)
       │
  ┌────┼──────────┬──────────────┐
  │    │          │              │
exit 0  exit 2  continue:false  updatedInput
  │    │          │              │
Allow  Block     Block          Modify input params
      Return stderr  Return stopReason  Call with modified params
      to model
            │
            ▼
       Execute tool call()
            │
            ▼
       PostToolUse Hooks
```

---

## Summary

| Hook Event | Trigger | Exit 2 Effect | Special Capability |
|------------|---------|---------------|-------------------|
| `PreToolUse` | Before tool call | Block call | Modify params, override permission |
| `PostToolUse` | After tool call | stderr→model immediately | Modify MCP tool output |
| `UserPromptSubmit` | On user submit | Block + clear original prompt | Append context |
| `SessionStart` | On new session | — | Auto first message, register file watch |
| `SessionEnd` | On session close | — | 1500ms timeout, reason matching |
| `Stop` | Before Claude ends response | Claude continues running | Task completion verification |
| `PreCompact` | Before compaction | Block compaction | Append compaction instructions |
| `PermissionDenied` | After permission denied | — | `retry: true` for model retry |
| `PermissionRequest` | On permission dialog | — | Completely replace UI permission decision |
| `TeammateIdle` | Teammate about to idle | Block idle | — |
| `TaskCreated/Completed` | Task change | Block change | — |
| `ConfigChange` | Config hot reload | Block change application | source matching |
| `InstructionsLoaded` | CLAUDE.md loaded | **No effect** (cannot block) | Observe-only |
| `WorktreeCreate` | Worktree creation | — | Completely replace default implementation |
| `CwdChanged/FileChanged` | Directory/file change | — | CLAUDE_ENV_FILE injects env vars |

**Source location overview:**
- Complete hook event list: [src/entrypoints/sdk/coreTypes.ts](../../src/entrypoints/sdk/coreTypes.ts)
- Hook types and response schema: [src/types/hooks.ts](../../src/types/hooks.ts)
- Hook execution main logic: [src/utils/hooks.ts](../../src/utils/hooks.ts)
- Hook configuration aggregation: [src/utils/hooks/hooksSettings.ts](../../src/utils/hooks/hooksSettings.ts)
- Hook event metadata (exit code behavior): [src/utils/hooks/hooksConfigManager.ts](../../src/utils/hooks/hooksConfigManager.ts)
- Hook event broadcast system: [src/utils/hooks/hookEvents.ts](../../src/utils/hooks/hookEvents.ts)
