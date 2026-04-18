# Chapter 5: Subagent and Agent Teams — Multi-Agent Orchestration Protocol

> Source location: `src/tools/AgentTool/`, `src/utils/swarm/`, `src/tasks/InProcessTeammateTask/`

---

## 5.1 Three Modes of Multi-Agent Coordination

Claude Code supports three different multi-agent modes:

```
Mode 1: Regular Subagent (specified type)
  Agent({ subagent_type: 'general-purpose', prompt: '...' })
  → Creates a predefined type of subagent, runs independently

Mode 2: Fork Subagent (inherits context)
  Agent({ prompt: '...' })  // subagent_type omitted
  → Forks the current session; subagent inherits the full parent context

Mode 3: Agent Teams (team collaboration)
  TeamCreate → create team
  → Multiple in-process teammates run concurrently
  → Communicate via mailbox with permission sync
```

---

## 5.2 Regular Subagent: runAgent()

### 5.2.1 Entry Point

[src/tools/AgentTool/runAgent.ts:248](../../src/tools/AgentTool/runAgent.ts#L248)

```typescript
// Note: async generator (function*), not a regular async function
// Yields Message one at a time; callers consume with for await...of
export async function* runAgent({
  agentDefinition,
  promptMessages,    // Initial messages (replaces the old directive string)
  toolUseContext,
  canUseTool,        // Tool permission check function (independent instance per subagent)
  isAsync,           // Whether to run as a background async task
  forkContextMessages,
  querySource,
  override,          // { systemPrompt?, userContext?, agentId?, abortController? }
  model,
  maxTurns,
  availableTools,    // Pre-assembled tool pool from the caller
  allowedTools,      // Explicit tool allowlist (replaces all parent agent allow rules)
  useExactTools,     // Fork-specific: skip resolveAgentTools(), use availableTools directly
  worktreePath,
  ...
}: { ... }): AsyncGenerator<Message, void>
```

**Why an async generator?** The subagent's `query()` is also an async generator. `runAgent` directly yields its events, allowing the caller to observe subagent progress in real time (for TaskOutput updates) without waiting for the subagent to complete.

### 5.2.2 Agent Definition (AgentDefinition)

[src/tools/AgentTool/loadAgentsDir.ts:106](../../src/tools/AgentTool/loadAgentsDir.ts#L106)

Agent definitions have three sources, all sharing `BaseAgentDefinition` base fields:

```typescript
// Base fields shared by all agents
type BaseAgentDefinition = {
  agentType: string
  whenToUse: string
  tools?: string[]           // Tool allowlist ('*' = all parent agent tools)
  disallowedTools?: string[]
  model?: string             // 'inherit' = inherit parent agent model
  permissionMode?: PermissionMode
  maxTurns?: number
  skills?: string[]          // List of skill names to preload
  mcpServers?: AgentMcpServerSpec[]  // Agent-specific MCP servers
  hooks?: HooksSettings
  isolation?: 'worktree' | 'remote' // Isolation mode: git worktree or remote
  background?: boolean       // Always spawn as a background task
  memory?: 'user' | 'project' | 'local'  // Persistent memory scope
  initialPrompt?: string     // Prompt injected before the first user turn
  omitClaudeMd?: boolean     // Omit CLAUDE.md (Explore/Plan optimization, saves tokens)
  requiredMcpServers?: string[]  // MCP servers that must exist (otherwise unavailable)
  criticalSystemReminder_EXPERIMENTAL?: string  // Critical reminder injected every turn
}

// Built-in agent (defined in source code)
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  baseDir: 'built-in'
  // getSystemPrompt requires toolUseContext (to check options for environment info)
  getSystemPrompt: (params: { toolUseContext: Pick<ToolUseContext, 'options'> }) => string
}

// Custom agent (loaded from .claude/agents/*.md)
type CustomAgentDefinition = BaseAgentDefinition & {
  source: 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
  filename?: string
  getSystemPrompt: () => string  // Content from the markdown file body
}

// Plugin agent (loaded from plugin system)
type PluginAgentDefinition = BaseAgentDefinition & {
  source: 'plugin'
  plugin: string   // Plugin name
  getSystemPrompt: () => string
}

// Union type
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

**Agent priority and override order** ([src/tools/AgentTool/loadAgentsDir.ts:196](../../src/tools/AgentTool/loadAgentsDir.ts#L196)):

```
built-in → plugin → userSettings → projectSettings → flagSettings → policySettings
```

When `agentType` is the same, later entries override earlier ones. This means managed (policySettings) agents can override all user-defined agents; plugin agents can override built-in agents.

**`omitClaudeMd` design rationale**: Read-only agents like Explore and Plan don't need commit/PR/lint guidelines from CLAUDE.md (the main agent interprets their output). Disabling it saves 5–15 Gtok per spawn; across 34M+ Explore spawns, this has a significant impact.

### 5.2.3 AsyncLocalStorage Context Isolation

Subagents achieve in-process isolation via `AsyncLocalStorage`:

```typescript
// src/utils/agentContext.ts
type SubagentContext = {
  agentId: string
  parentAgentId?: string
  // ...
}

// While a subagent runs, all code within this async call chain
// can read the correct agentId via getAgentContext()
function runWithAgentContext<T>(
  context: AgentContext,
  fn: () => Promise<T>
): Promise<T>
```

This means:
- Subagents calling `getSessionId()` get their own ID
- Subagent todo lists are isolated from the parent agent
- Subagent tool permission checks use the subagent's own context

---

## 5.3 Fork Subagent: Inheriting Parent Context

This is one of the most elegant design choices, and the key to Prompt Cache optimization.

### 5.3.1 What Is a Fork?

[src/tools/AgentTool/forkSubagent.ts:32](../../src/tools/AgentTool/forkSubagent.ts#L32)

When `subagent_type` is omitted and `isForkSubagentEnabled()` returns true, Fork mode is triggered. The subagent inherits:
- The full conversation history (byte-identical message prefix)
- The system prompt (passed via `override.systemPrompt` as already-rendered bytes, not reconstructed)
- The tool list (`tools: ['*']` + `useExactTools: true`, skipping `resolveAgentTools()`)
- The permission mode (`permissionMode: 'bubble'`)

**Three conditions for `isForkSubagentEnabled()`** (all must be met):
1. `feature('FORK_SUBAGENT')` compile-time gate is enabled
2. `!isCoordinatorMode()` — mutually exclusive with Coordinator mode (coordinator has its own orchestration model)
3. `!getIsNonInteractiveSession()` — only enabled in interactive sessions

### 5.3.2 FORK_AGENT Definition

[src/tools/AgentTool/forkSubagent.ts:60-71](../../src/tools/AgentTool/forkSubagent.ts#L60-L71)
```typescript
export const FORK_AGENT = {
  agentType: 'fork',  // analytics identifier
  whenToUse: 'Implicit fork — inherits full conversation context.',
  tools: ['*'],              // Inherit all parent agent tools
  maxTurns: 200,
  model: 'inherit',          // Inherit parent agent model
  permissionMode: 'bubble',  // Permission requests bubble up to parent agent
  source: 'built-in',
  getSystemPrompt: () => '',  // Unused! Override passes the rendered prompt
}
```

**Why does `getSystemPrompt` return an empty string?**

Because Fork subagents pass the parent agent's **already-rendered** system prompt bytes via `override.systemPrompt`. Re-calling `getSystemPrompt()` might produce different results due to GrowthBook (feature flag service) state changes, breaking the Prompt Cache.

### 5.3.3 buildForkedMessages(): Maximizing Prompt Cache

[src/tools/AgentTool/forkSubagent.ts:107](../../src/tools/AgentTool/forkSubagent.ts#L107)

```typescript
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[]  // Returns only 2 new messages! Parent history is already in context
```

**Key design**: The function returns only 2 messages appended to the parent agent's history:

```typescript
// Step 1: Clone parent agent's assistant message (including all tool_use blocks, including thinking and text)
const fullAssistantMessage = {
  ...assistantMessage,
  uuid: randomUUID(),   // New UUID (avoids conflict with parent agent)
  message: { ...assistantMessage.message, content: [...assistantMessage.message.content] },
}

// Step 2: Create identical placeholder tool_result for each tool_use block
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
const toolResultBlocks = toolUseBlocks.map(block => ({
  type: 'tool_result',
  tool_use_id: block.id,
  content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }],
  // All subagent placeholder texts are identical → prompt cache hit
}))

// Step 3: Build the final user message ([...tool_results, directive text])
const toolResultMessage = createUserMessage({
  content: [...toolResultBlocks, { type: 'text', text: buildChildMessage(directive) }],
})

return [fullAssistantMessage, toolResultMessage]  // Only 2 messages
// Full API sequence = parentHistory + [fullAssistantMessage, toolResultMessage]
```

**Edge case**: If `assistantMessage` has no `tool_use` blocks (should not happen), returns a single user message containing just the directive text.

### 5.3.4 buildChildMessage(): Fork Worker Directive

[src/tools/AgentTool/forkSubagent.ts:171](../../src/tools/AgentTool/forkSubagent.ts#L171)

`buildChildMessage()` generates operating rules for the fork subagent, wrapped in `<fork-boilerplate>` tags:

```
<fork-boilerplate>
STOP. READ THIS FIRST.
You are a forked worker process. You are NOT the main agent.

Rules (non-negotiable):
1. The system prompt says "fork by default" — ignore it; you're already a fork, don't spawn subagents
2. Don't chat, ask questions, or suggest next steps
3. Use tools directly: Bash, Read, Write, etc.
4. After editing files, commit them; include the commit hash in your report
5. Don't output text between tool calls; report everything at the end
6. Work strictly within the directive scope
7. Keep your report under 500 words

Output format (plain text labels, no markdown headers):
  Scope: <your work scope, one sentence>
  Result: <key findings/answers>
  Key files: <relevant file paths>
  Files changed: <modified files + commit hash>
  Issues: <issues found (only if any)>
</fork-boilerplate>
{FORK_DIRECTIVE_PREFIX}{directive}
```

This forced output format prevents fork subagents from misapplying the parent agent's system prompt ("fork by default for parallel execution") to themselves.

### 5.3.5 Preventing Recursive Forks

[src/tools/AgentTool/forkSubagent.ts:78](../../src/tools/AgentTool/forkSubagent.ts#L78)

```typescript
// FORK_BOILERPLATE_TAG is imported from constants/xml.ts, value is 'fork-boilerplate'
import { FORK_BOILERPLATE_TAG } from '../../constants/xml.js'

export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content    // Note: content must be declared first
    if (!Array.isArray(content)) return false
    return content.some(
      block => block.type === 'text' && block.text.includes(`<${FORK_BOILERPLATE_TAG}>`)
    )
  })
}
```

**Why do fork subagents still retain the Agent tool?** To ensure tool definition bytes in the API request prefix are completely consistent (required for prompt cache). But recursive forks are rejected at call time via `isInForkChild()`, rather than by removing the Agent tool from the pool.

### 5.3.6 Worktree-Isolated Fork

[src/tools/AgentTool/forkSubagent.ts:205](../../src/tools/AgentTool/forkSubagent.ts#L205)

When the agent definition has `isolation: 'worktree'`, the fork subagent runs in an isolated git worktree. The directive is then injected with `buildWorktreeNotice()`, informing the subagent:

```
You inherit the conversation context of a parent agent working in {parentCwd}.
You are running in an isolated git worktree at {worktreeCwd} —
same repository, same relative file structure, independent working copy.
Paths in the context point to the parent agent's directory; translate them to your worktree root.
If the parent agent may have modified files, re-read them before editing.
Your modifications are only in this worktree and don't affect parent agent files.
```

---

## 5.4 Agent Teams: In-Process Concurrent Teammates

### 5.4.1 Architecture Overview

```
Leader (main agent)
    │
    ├─ TeamCreate({ teammates: ['worker-a', 'worker-b'] })
    │
    ├─ worker-a runs in-process (InProcessTeammateTask)
    │    ├─ Independent AgentContext (AsyncLocalStorage)
    │    ├─ Independent message history
    │    └─ Communicates with leader via mailbox
    │
    └─ worker-b runs in-process (InProcessTeammateTask)
         ├─ Independent AgentContext
         ├─ Independent message history
         └─ Communicates with leader via mailbox
```

### 5.4.2 In-Process Runner (Core)

[src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts)

```typescript
// Note: context is runWithTeammateContext (not runWithAgentContext)
// TeammateContext has more fields than AgentContext: teamName, mailbox, permissionMode, etc.

async function runInProcessTeammate(
  identity: TeammateIdentity,
  toolUseContext: ToolUseContext,
) {
  // 1. Clone file state cache (isolate file read/write state, preventing concurrent teammates from interfering)
  cloneFileStateCache()

  // 2. Create independent AbortControllers for the teammate (2 of them!)
  const abortController = createAbortController()          // Kill the entire teammate
  const currentWorkAbortController = createAbortController() // Only abort the current turn

  // 3. Create a teammate-specific canUseTool function (permission routing, see below)
  const canUseTool = createInProcessCanUseTool(identity, abortController)

  // 4. Run in an independent TeammateContext (AsyncLocalStorage isolation)
  return runWithTeammateContext(teammateContext, async () => {
    // 5. Start the teammate's Agent Loop (runAgent is an async generator)
    for await (const message of runAgent({ agentDefinition: teammate, ... })) {
      // Update AppState, write sidechain transcript, check mailbox messages, etc.
    }
  })
}
```

### 5.4.3 Mailbox Communication Mechanism

Teammates (including with the leader) communicate asynchronously via "mailboxes":

```typescript
// src/utils/teammateMailbox.ts
readMailbox(teammateId: string): Message[]     // Read received messages
writeToMailbox(teammateId: string, msg): void  // Send a message

// Message type identification
isPermissionResponse(msg): boolean            // Permission approval/denial
isShutdownRequest(msg): boolean               // Shutdown request
createIdleNotification(): Message             // Teammate completion notification
```

### 5.4.4 Permission Sync

[src/utils/swarm/inProcessRunner.ts:128](../../src/utils/swarm/inProcessRunner.ts#L128) — `createInProcessCanUseTool()`

When a teammate's tool call requires user confirmation (`behavior === 'ask'`), it routes as follows:

```
Worker calls tool
  │
  ├─ hasPermissionsToUseTool() returns 'allow'/'deny' → pass/deny directly
  │
  └─ Returns 'ask' (requires user confirmation)
       │
       ├─【Primary path】setToolUseConfirmQueue available (leader's TUI is online)
       │    → Push permission request to leader's ToolUseConfirm queue
       │    → Show confirmation dialog with worker badge (name + color)
       │    → Wait for user action; uses same UI as leader's own tool confirmations
       │
       └─【Fallback path】TUI bridge unavailable (headless / background mode)
            → Send permission request to leader via mailbox
            → Poll at 500ms intervals (PERMISSION_POLL_INTERVAL_MS)
            → Continue after leader responds
```

**Bash command pre-processing**: Before sending to the leader, use the classifier (`BASH_CLASSIFIER` feature gate) to auto-approve. If the classifier approves, skip leader confirmation. This differs from leader handling (leader races; worker waits sequentially).

```typescript
// Key functions provided by leaderPermissionBridge.ts
getLeaderToolUseConfirmQueue()   // Get leader's ToolUseConfirm state setter
getLeaderSetToolPermissionContext() // Get leader's permission context
```

---

## 5.5 Coordinator Mode: Autonomous Agent Orchestration

[src/coordinator/coordinatorMode.ts](../../src/coordinator/coordinatorMode.ts)

Coordinator mode is dedicated to long-running autonomous tasks. Enabled via environment variable; mutually exclusive with Fork Subagent.

```typescript
// Check whether currently in coordinator mode
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}

// When resuming a session, automatically match the session's stored mode (flip env var if necessary)
export function matchSessionMode(sessionMode: 'coordinator' | 'normal' | undefined): string | undefined

// Inject the list of worker's available tools to the coordinator (so it knows what to delegate)
export function getCoordinatorUserContext(scratchpadDir?: string): string
```

> **Note**: There is **no** `getCoordinatorSystemPrompt()` function in the source code. The coordinator's system prompt comes from the standard path in `buildEffectiveSystemPrompt()`. The coordinator injects the tool list via `userContext`, not via a separate prompt function.

**Mutual exclusion with Fork**:
```typescript
// forkSubagent.ts
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // ← mutually exclusive
    ...
  }
}
```

**Worker tool set restrictions** (`src/constants/tools.ts`):

```typescript
// ASYNC_AGENT_ALLOWED_TOOLS restricts the worker's tool pool
// Excludes: TeamCreate/TeamDelete (advanced orchestration tools), SendMessage, SyntheticOutput
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
// Worker = ASYNC_AGENT_ALLOWED_TOOLS - INTERNAL_WORKER_TOOLS
```

---

## 5.6 Sequence Diagram: Fork Subagent Prompt Cache Optimization

```
Time →

Parent agent history messages (shared Prompt Cache):
┌──────────────────────────────────────────┐
│  user: "Complete the following 3 tasks..." │
│  assistant: [tool_use: agent(task1)]       │
│             [tool_use: agent(task2)]       │
│             [tool_use: agent(task3)]       │
└──────────────────────────────────────────┘
                    │
    buildForkedMessages() creates subagent messages for each task:
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
  Subagent 1:   Subagent 2:  Subagent 3:
  [...shared]   [...shared]  [...shared]
  [assistant]   [assistant]  [assistant]   ← Exactly the same
  [user:        [user:       [user:
    placeholder]  placeholder]  placeholder] ← Exactly the same
    task1 directive] task2 directive] task3 directive]  ← Only this line differs

  ↑ Prompt Cache hit ↑  ↑ hit ↑           ↑ hit ↑
  Only the trailing directive line differs; prefix bytes are identical, maximizing cache hits
```

---

## Summary

| Mechanism | Use Case | Key Feature | Source Location |
|-----------|----------|-------------|-----------------|
| Regular Subagent | Specialized tasks with specified type | Async generator; independent context, system prompt, tool set | [src/tools/AgentTool/runAgent.ts](../../src/tools/AgentTool/runAgent.ts) |
| Fork Subagent | Parallel execution of similar tasks | Inherits parent context; `buildForkedMessages` returns only 2 messages; maximizes Prompt Cache | [src/tools/AgentTool/forkSubagent.ts](../../src/tools/AgentTool/forkSubagent.ts) |
| Agent Teams | Collaborative tasks requiring communication | `runWithTeammateContext` isolation; permission routing prefers leader TUI | [src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts) |
| Coordinator mode | Autonomous long-running tasks | Mutually exclusive with Fork; env var controlled; `matchSessionMode` auto-restores | [src/coordinator/coordinatorMode.ts](../../src/coordinator/coordinatorMode.ts) |
| AsyncLocalStorage | In-process context isolation | No cross-process overhead; Subagents use AgentContext, Teammates use TeammateContext | [src/utils/agentContext.ts](../../src/utils/agentContext.ts) |

**Key design conventions summary**:

| Convention | Description |
|------------|-------------|
| `runAgent` is `async function*` | Not a regular async function; yields Message as async generator |
| `buildForkedMessages` returns 2 messages | Returns only the 2 appended messages; full sequence = parentHistory + 2 messages |
| Fork subagents retain the Agent tool | To preserve prompt cache; recursion blocked at call time via `isInForkChild()` |
| `getCoordinatorSystemPrompt()` does not exist | Coordinator injects tool list via `getCoordinatorUserContext()` |
| Primary permission route is leader TUI | Mailbox is the headless/background fallback, not the primary path |
| Teammates use `runWithTeammateContext` | Not `runWithAgentContext`; TeammateContext has additional fields |
