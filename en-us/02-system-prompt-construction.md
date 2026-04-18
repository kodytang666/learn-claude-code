# Chapter 2: Global Context Prompt — System Prompt Construction

> Source location: `src/utils/systemPrompt.ts`, `src/utils/attachments.ts`, `src/constants/prompts.ts`

---

## 2.1 Why Does the System Prompt Matter?

In LLM applications, the system prompt defines the AI's identity, capability boundaries, and behavioral norms. Claude Code's system prompt is not static — it is **dynamically constructed** based on the current mode (normal / Coordinator / Proactive / custom Agent) and assembled before each API call.

---

## 2.2 Five Sources of the System Prompt

```
Priority (highest to lowest):

  0. Override prompt     ← Special scenarios like loop mode; completely replaces all others
  1. Coordinator prompt  ← Autonomous agent coordinator mode (env var CLAUDE_CODE_COORDINATOR_MODE=1)
  2. Agent prompt        ← Custom agent defined by mainThreadAgentDefinition
     ├── Proactive mode: Agent prompt is "appended" after Default
     └── Normal mode: Agent prompt "replaces" Default
  3. Custom prompt       ← User-supplied via --system-prompt CLI argument
  4. Default prompt      ← Claude Code's built-in standard prompt
  + Append prompt        ← Always appended at the end (except when Override is active)
```

---

## 2.3 Core Function: buildEffectiveSystemPrompt()

[src/utils/systemPrompt.ts:41-120](../../src/utils/systemPrompt.ts#L41-L120)
```typescript
export function buildEffectiveSystemPrompt({
  mainThreadAgentDefinition,
  toolUseContext,
  customSystemPrompt,
  defaultSystemPrompt,
  appendSystemPrompt,
  overrideSystemPrompt,
}: {
  mainThreadAgentDefinition: AgentDefinition | undefined
  toolUseContext: Pick<ToolUseContext, 'options'>
  customSystemPrompt: string | undefined
  defaultSystemPrompt: string[]
  appendSystemPrompt: string | undefined
  overrideSystemPrompt?: string | null
}): SystemPrompt {
  
  // Priority 0: Override (return immediately, ignoring all other prompts)
  if (overrideSystemPrompt) {
    return asSystemPrompt([overrideSystemPrompt])
  }
  
  // Priority 1: Coordinator mode
  // Note: uses lazy require to avoid circular dependencies
  if (
    feature('COORDINATOR_MODE') &&
    isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE) &&
    !mainThreadAgentDefinition
  ) {
    const { getCoordinatorSystemPrompt } = require('../coordinator/coordinatorMode.js')
    return asSystemPrompt([
      getCoordinatorSystemPrompt(),
      ...(appendSystemPrompt ? [appendSystemPrompt] : []),
    ])
  }

  // Compute the Agent prompt (if a custom agent definition exists)
  const agentSystemPrompt = mainThreadAgentDefinition
    ? isBuiltInAgent(mainThreadAgentDefinition)
      ? mainThreadAgentDefinition.getSystemPrompt({ toolUseContext: { options: toolUseContext.options } })
      : mainThreadAgentDefinition.getSystemPrompt()
    : undefined

  // Priority 2 (Proactive mode): Agent prompt appended after Default
  if (agentSystemPrompt && isProactiveActive_SAFE_TO_CALL_ANYWHERE()) {
    return asSystemPrompt([
      ...defaultSystemPrompt,
      `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
      ...(appendSystemPrompt ? [appendSystemPrompt] : []),
    ])
  }

  // Priority 2/3/4: Agent > Custom > Default, then append Append
  return asSystemPrompt([
    ...(agentSystemPrompt
      ? [agentSystemPrompt]
      : customSystemPrompt
        ? [customSystemPrompt]
        : defaultSystemPrompt),
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}
```

---

## 2.4 Decision Tree

```
Call buildEffectiveSystemPrompt()
              │
              ▼
    overrideSystemPrompt present?
         │         │
        YES        NO
         │          │
         ▼          ▼
    [Override]  COORDINATOR_MODE env var?
                     │            │
                    YES           NO
                     │            │
                     ▼            ▼
               [Coordinator]  mainThreadAgentDefinition present?
                                   │              │
                                  YES             NO
                                   │              │
                                   ▼              ▼
                          isProactiveActive()?  customSystemPrompt present?
                               │       │            │         │
                              YES      NO           YES       NO
                               │       │             │        │
                               ▼       ▼             ▼        ▼
                          [Default  [Agent only]  [Custom]  [Default]
                          + Agent]
                               │       │             │        │
                               └───────┴─────────────┴────────┘
                                                │
                                                ▼
                                   + appendSystemPrompt (if present)
```

---

## 2.5 Attachment Messages

Beyond the system prompt, Claude Code injects dynamic context into each API call. Attachment messages have type `AttachmentMessage`, and are converted to API format (typically a `<system-reminder>`-wrapped user message) by `normalizeMessagesForAPI()`.

`getAttachmentMessages()` manages over 50 attachment types, grouped by trigger scenario ([src/utils/attachments.ts:743](../../src/utils/attachments.ts#L743)):

### User Input Related (processed on each user submission)

| Attachment Type | Trigger Condition |
|---|---|
| `skill_listing` | New skills not yet announced to the LLM (determined by sentSkillNames) |
| `skill_discovery` | When EXPERIMENTAL_SKILL_SEARCH is enabled; semantic search results |
| `nested_memory` | When user @-mentions a file; automatically loads conditional CLAUDE.md rules from that file's directory |
| `agent_mentions` | User message mentions a @agent |

### Global Delta (injected only when content changes, to avoid redundant tokens)

| Attachment Type | Trigger Condition | Description |
|---|---|---|
| `deferred_tools_delta` | Model supports tool references + tool search is enabled | Tells Claude which tools can be discovered via ToolSearch, instead of stuffing all tools into the tools parameter |
| `agent_listing_delta` | Available agent definitions have changed | Tells Claude which `subagent_type` values it can invoke |
| `mcp_instructions_delta` | First injection after MCP server connects | MCP server usage instructions |
| `date_change` | Date has crossed midnight | Updates the current date |
| `plan_mode` / `plan_mode_exit` | Plan mode entered/exited | |
| `todo_reminder` / `task_reminder` | There are incomplete tasks | Reminds Claude of current todos |
| `teammate_mailbox` | New mail in Coordinator mode | |

### Main Thread Only (IDE integration, etc.)

| Attachment Type | Trigger Condition |
|---|---|
| `selected_lines_in_ide` | Code is selected in IDE |
| `opened_file_in_ide` | A file is opened in IDE |
| `critical_system_reminder` | System messages requiring strong reminders |

### Key Design Principles for Attachments

- **Delta mode**: Global attachments are only injected when their content changes, avoiding redundant token consumption.
- **AttachmentMessage container**: A unified internal data structure that doesn't directly correspond to API format; `normalizeMessagesForAPI()` converts it before each API call.
- **Async injection of relevant_memories**: CLAUDE.md-related memories are prefetched via `startRelevantMemoryPrefetch()` and consumed after tool execution (see Chapter 1).

---

## 2.6 userContext and CLAUDE.md

`userContext` is a `{ [k: string]: string }` key-value map, built by `getUserContext()` ([src/context.ts:155](../../src/context.ts#L155)), containing two items:

```typescript
return {
  ...(claudeMd && { claudeMd }),            // Merged CLAUDE.md content (see below)
  currentDate: `Today's date is ${getLocalISODate()}.`,
}
```

`userContext` is **not** part of the systemPrompt. At API call time, it is **prepended to the messages list** as a `<system-reminder>` user message via `prependUserContext()` ([src/query.ts:660](../../src/query.ts#L660)).

### CLAUDE.md Memory File Loading Order

`getMemoryFiles()` loads files in the following order ([src/utils/claudemd.ts:803](../../src/utils/claudemd.ts#L803)):

```
1. Managed (enterprise policy)
   /etc/claude-code/CLAUDE.md
   ~/.claude-code/rules/*.md

2. User (user global, if not disabled in userSettings)
   ~/.claude/CLAUDE.md
   ~/.claude/rules/*.md

3. Project (traversing from Git root down to CWD, each directory in order)
   CLAUDE.md              ← Project instructions committed to repo
   .claude/CLAUDE.md
   .claude/rules/*.md
   CLAUDE.local.md        ← Local private instructions, not committed to repo
```

Git worktree special handling: when running from a nested worktree, the main repository's Project-level files are skipped to avoid double-loading.

### What filterInjectedMemoryFiles Actually Does

The claim that it "filters already-injected files (to avoid duplicates)" is inaccurate. The actual logic ([src/utils/claudemd.ts:1142](../../src/utils/claudemd.ts#L1142)) is:

```typescript
export function filterInjectedMemoryFiles(files: MemoryFileInfo[]): MemoryFileInfo[] {
  const skipMemoryIndex = getFeatureValue_CACHED_MAY_BE_STALE('tengu_moth_copse', false)
  if (!skipMemoryIndex) return files          // Default: return all files
  return files.filter(f => f.type !== 'AutoMem' && f.type !== 'TeamMem')
}
```

By default, **no files are filtered**. Only when the `tengu_moth_copse` feature flag is enabled are `AutoMem` and `TeamMem` types filtered out (these two types are separately injected asynchronously via the `relevant_memories` attachment).

### Conditional Rules (nested_memory)

When a user @-mentions a file, Claude Code automatically looks up the CLAUDE.md in that file's directory and injects matching conditional rules as a `nested_memory` attachment. This is the mechanism behind subdirectory-scoped CLAUDE.md rules.

---

## 2.7 systemContext and Git Status

Alongside `userContext`, `systemContext` is built by `getSystemContext()` ([src/context.ts:116](../../src/context.ts#L116)) and contains:

```typescript
return {
  ...(gitStatus && { gitStatus }),       // Current git workspace status (optional)
  ...(cacheBreaker && { cacheBreaker }), // Cache breaker, ant-only (optional)
}
```

`systemContext` is injected differently from `userContext` — it is **appended to the end of the systemPrompt array** via `appendSystemContext()` ([src/query.ts:449](../../src/query.ts#L449)), becoming part of the system prompt:

```typescript
// src/utils/api.ts
export function appendSystemContext(
  systemPrompt: SystemPrompt,
  context: { [k: string]: string },
): string[] {
  return [
    ...systemPrompt,
    Object.entries(context)
      .map(([key, value]) => `${key}: ${value}`)  // Format: "gitStatus: ..."
      .join('\n'),
  ].filter(Boolean)
}
```

| | userContext | systemContext |
|---|---|---|
| **Content** | CLAUDE.md + current date | Git status + cache breaker |
| **Injection method** | `prependUserContext()` → `<system-reminder>` prepended to messages | `appendSystemContext()` → appended to end of systemPrompt |
| **Caching** | `memoize`, loaded once per session | `memoize`, loaded once per session |

---

## 2.8 System Prompt Token Analysis

After context construction, Claude Code analyzes token usage:

```typescript
// src/utils/contextAnalysis.ts
analyzeContext(messages, systemPrompt)
// → tokenStatsToStatsigMetrics()  used for analytics reporting
```

This provides input data for Auto Compaction (Chapter 7).

---

## 2.9 Coordinator Mode System Prompt

Coordinator mode is Claude Code's "autonomous agent orchestration" mode, where the system prompt is completely different:

```typescript
// src/coordinator/coordinatorMode.ts
export function getCoordinatorSystemPrompt(): string {
  // Returns coordinator-specific prompt:
  // - How to assign tasks to worker agents
  // - How to communicate via mailbox
  // - Tool set restrictions (ASYNC_AGENT_ALLOWED_TOOLS)
  // - Task completion criteria
}

export function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') && 
         isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
}
```

The Coordinator also has its own worker context injection:

```typescript
getCoordinatorUserContext()
// Injects the list of available worker tools into user messages
// Lets the coordinator know what capabilities it can delegate to workers
```

---

## 2.10 System Prompt Cache Optimization

The Claude API supports **Prompt Cache**. Claude Code maximizes cache hits through:

1. **Stable system prompt content**: Avoid changing system prompt content in each loop iteration.
2. **Delta attachment mode**: Only append new content when there are changes, avoiding changes to the entire message list.
3. **Fork Subagent byte-level consistency**: When creating a subagent, pass the "already-rendered" system prompt bytes rather than re-calling `getSystemPrompt()` (which may produce different results due to GrowthBook state changes).

[src/tools/AgentTool/forkSubagent.ts:56-71](../../src/tools/AgentTool/forkSubagent.ts#L56-L71)
```typescript
// FORK_AGENT definition:
// The getSystemPrompt here is unused: the fork path passes
// `override.systemPrompt` with the parent's already-rendered system prompt
// bytes, threaded via `toolUseContext.renderedSystemPrompt`. 
// Reconstructing by re-calling getSystemPrompt() can diverge (GrowthBook 
// cold→warm) and bust the prompt cache; threading the rendered bytes is 
// byte-exact.
export const FORK_AGENT = {
  // ...
  getSystemPrompt: () => '',  // Unused; override passes parent's rendered prompt
}
```

---

## Summary

| Mechanism | Purpose | Source Location |
|-----------|---------|-----------------|
| `buildEffectiveSystemPrompt()` | Combines system prompt by priority | [src/utils/systemPrompt.ts:41](../../src/utils/systemPrompt.ts#L41) |
| `userContext` | CLAUDE.md + current date, inserted as `<system-reminder>` at message list front | [src/context.ts:155](../../src/context.ts#L155) |
| `systemContext` | Git status + cache breaker, appended to end of systemPrompt | [src/context.ts:116](../../src/context.ts#L116) |
| Attachment Messages | Dynamic context injection (50+ types, delta mode to avoid duplication) | [src/utils/attachments.ts:743](../../src/utils/attachments.ts#L743) |
| CLAUDE.md memory files | Multi-level loading (managed → user → project), injected via userContext | [src/utils/claudemd.ts:803](../../src/utils/claudemd.ts#L803) |
| `nested_memory` | Auto-loads directory rules when user @-mentions a file | `src/utils/attachments.ts` |
| Coordinator prompt | Autonomous agent orchestration mode | `src/coordinator/coordinatorMode.ts` |
| Prompt Cache optimization | Fork subagent passes rendered bytes, ensuring byte-level consistency | [src/tools/AgentTool/forkSubagent.ts:56](../../src/tools/AgentTool/forkSubagent.ts#L56) |
