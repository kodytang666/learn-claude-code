# Chapter 11: Architecture Overview — Sequence Diagrams, Flow Charts, and Module Relationships

---

## 11.1 Overall Architecture Layers

```
┌────────────────────────────────────────────────────────────────────┐
│                      Claude Code Architecture                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐ │
│  │  CLI     │  │  IDE     │  │  Desktop │  │  claude.ai/code     │ │
│  │(terminal)│  │extension │  │  app     │  │  (Web)              │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────┬──────────┘ │
│       └─────────────┴─────────────┴───────────────────┘            │
│                              │                                     │
│                    ┌─────────┴─────────┐                           │
│                    │    REPL / Bridge   │                          │
│                    │  (src/bridge/)     │                          │
│                    └─────────┬─────────┘                           │
│                              │                                     │
│  ┌───────────────────────────▼──────────────────────────────────┐  │
│  │                    Agent Loop Core Layer                      │  │
│  │                    (src/query.ts)                            │  │
│  │                                                              │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  Context Construction                                    │ │  │
│  │  │  buildEffectiveSystemPrompt() + getAttachmentMessages() │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          │                                   │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  Claude API Call                                         │ │  │
│  │  │  queryModelWithStreaming() (src/services/api/claude.ts) │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          │                                   │  │
│  │  ┌─────────────────────────────────────────────────────────┐ │  │
│  │  │  Tool Execution Layer                                    │ │  │
│  │  │  runTools() → StreamingToolExecutor → 51 Tool impls     │ │  │
│  │  └─────────────────────────────────────────────────────────┘ │  │
│  │                          │                                   │  │
│  │  ┌──────────┐  ┌─────────────────┐  ┌──────────────────────┐ │  │
│  │  │ Token    │  │ Auto Compaction │  │  Hook System         │ │  │
│  │  │ Budget   │  │                 │  │  (27 event types)    │ │  │
│  │  └──────────┘  └─────────────────┘  └──────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Extension Capabilities Layer              │   │
│  │                                                             │   │
│  │  ┌─────────────┐  ┌────────────┐  ┌──────────────────────┐  │   │
│  │  │  Skills     │  │ Subagent & │  │  Worktree            │  │   │
│  │  │  system     │  │ Teams      │  │  task isolation       │  │   │
│  │  └─────────────┘  └────────────┘  └──────────────────────┘  │   │
│  │                                                             │   │
│  │  ┌─────────────┐  ┌────────────┐                            │   │
│  │  │  Task System│  │  MCP       │                            │   │
│  │  │  management │  │  protocol  │                            │   │
│  │  └─────────────┘  └────────────┘                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Infrastructure Layer                      │   │
│  │                                                             │   │
│  │  bootstrap/state.ts (global state)                          │   │
│  │  utils/ (200+ utility functions)                            │   │
│  │  services/analytics/ (analytics reporting)                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

---

## 11.2 Agent Loop Detailed Sequence Diagram

```
User              Claude Code            Claude API         Tool System
 │                      │                      │                │
 │──── Input prompt ───▶│                      │                │
 │                      │                      │                │
 │              buildEffectiveSystemPrompt()   │                │
 │              getAttachmentMessages()        │                │
 │                      │                      │                │
 │              normalizeMessagesForAPI()      │                │
 │                      │                      │                │
 │                      │── Streaming request ▶│                │
 │                      │                      │                │
 │◀─── Stream text ──────│◀── streaming ────────│                │
 │                      │                      │                │
 │                      │◀── tool_use block ───│                │
 │                      │                      │                │
 │                      │── checkPermissions() ────────────────▶│
 │                      │◀─ Permission result ───────────────────│
 │                      │                      │                │
 │◀─── Show tool exec ───│── call() ────────────────────────────▶│
 │                      │◀─ Tool result ─────────────────────────│
 │                      │                      │                │
 │              checkTokenBudget()             │                │
 │              isAutoCompactEnabled()?        │                │
 │                      │                      │                │
 │                      │── New request (with tool results) ───▶│
 │                      │       [loop continues...]             │
 │                      │                      │                │
 │                      │◀── Stop response ─────│                │
 │                      │                      │                │
 │              handleStopHooks()              │                │
 │                      │                      │                │
 │◀─── Task complete ────│                      │                │
```

---

## 11.3 Subagent Creation Sequence Diagram

```
Main Agent Loop      AgentTool          Subagent Loop        Claude API
        │                  │                  │                  │
        │── tool_use ─────▶│                  │                  │
        │   (Agent tool)    │                  │                  │
        │                  │                  │                  │
        │              Load agent definition  │                  │
        │              (loadAgentsDir)        │                  │
        │                  │                  │                  │
        │          isForkSubagentEnabled()?   │                  │
        │                  │                  │                  │
        │   [Fork mode]     │                  │                  │
        │         buildForkedMessages()       │                  │
        │         (build byte-identical msg prefix)              │
        │                  │                  │                  │
        │   [Normal mode]   │                  │                  │
        │         Use specified system prompt │                  │
        │                  │                  │                  │
        │              runWithAgentContext()  │                  │
        │              (AsyncLocalStorage isolation)             │
        │                  │                  │                  │
        │                  │── Start subagent▶│                  │
        │                  │   Loop           │                  │
        │                  │                  │── Subagent req ──▶│
        │                  │                  │◀─ Subagent resp ──│
        │                  │                  │  [subagent loop..]│
        │                  │                  │                  │
        │◀─ yield progress ─│◀── Progress event│                  │
        │                  │                  │                  │
        │                  │◀─ Subagent done ──│                  │
        │                  │                  │                  │
        │◀─ tool_result ───│                  │                  │
        │  (subagent result)│                  │                  │
        │                  │                  │                  │
        │  [Main agent continues...]          │                  │
```

---

## 11.4 Context Compaction Sequence Diagram

```
Agent Loop          AutoCompact          Compact Engine        Claude API
    │                   │                     │                    │
    │  After each tool execution round        │                    │
    │                   │                     │                    │
    │── calculateTokenWarningState() ─────────│                    │
    │◀─ 'critical' ───────────────────────────│                    │
    │                   │                     │                    │
    │── compactConversation() ───────────────▶│                    │
    │                   │                     │                    │
    │                   │        executePreCompactHooks()          │
    │                   │                     │                    │
    │                   │        stripImagesFromMessages()         │
    │                   │                     │                    │
    │                   │        groupMessagesByApiRound()         │
    │                   │                     │                    │
    │                   │        runForkedAgent(compact prompt) ──▶│
    │                   │                     │◀─ Summary text ─────│
    │                   │                     │                    │
    │                   │        createCompactBoundaryMessage()    │
    │                   │                     │                    │
    │                   │        buildPostCompactMessages()        │
    │                   │         ├─ Restore files (≤5)           │
    │                   │         ├─ Restore skills (≤25K)        │
    │                   │         └─ Restore plans                │
    │                   │                     │                    │
    │                   │        executePostCompactHooks()         │
    │                   │                     │                    │
    │                   │        notifyCompaction() (cache invalid)│
    │                   │                     │                    │
    │◀── Compacted message list ──────────────│                    │
    │                   │                     │                    │
    │  Continue Agent Loop                    │                    │
    │  using compacted history                │                    │
```

---

## 11.5 Hook Execution Sequence Diagram (PreToolUse Example)

```
Tool Exec Layer    Hook System          External Shell Script   Permission System
    │                  │                     │                  │
    │── PreToolUse ───▶│                     │                  │
    │   event fires     │                     │                  │
    │                  │                     │                  │
    │            getAllHooks('PreToolUse')   │                  │
    │            sortMatchersByPriority()    │                  │
    │                  │                     │                  │
    │            [For each matching hook:]   │                  │
    │                  │── Execute shell ───▶│                  │
    │                  │   (pass JSON event) │                  │
    │                  │                     │  Run check logic  │
    │                  │◀─ stdout (JSON resp) ─│                │
    │                  │                     │                  │
    │            Parse response:             │                  │
    │            - continue: false?          │                  │
    │            - updatedInput?             │                  │
    │            - permissionDecision?       │                  │
    │                  │                     │                  │
    │◀── Permission result ──│               │                  │
    │   (allow/deny/         │               │                  │
    │    modified input)     │               │                  │
    │                  │                     │                  │
    │  If allow:       │                     │                  │
    │── call(execute) ─▶│                    │                  │
```

---

## 11.6 Agent Teams Communication Sequence Diagram

```
Leader (main agent)   In-Process Runner     Worker agent       Mailbox System
      │                    │                   │                  │
      │── TeamCreate() ───▶│                   │                  │
      │                    │                   │                  │
      │            [For each worker:]          │                  │
      │            runWithAgentContext() ─────▶│                  │
      │            (AsyncLocalStorage isolation)│                  │
      │                    │                   │                  │
      │                    │── runAgent() ────▶│                  │
      │                    │                   │                  │
      │                    │                   │── Tool call ────▶│
      │                    │                   │   needs permission│
      │                    │                   │                  │
      │                    │◀─ Permission req ───────────────────│
      │                    │                   │                  │
      │◀─ Permission req ───│                   │                  │
      │   (bubbled to leader)│                  │                  │
      │                    │                   │                  │
      │  User confirms or   │                   │                  │
      │  leader decides     │                   │                  │
      │                    │                   │                  │
      │── Permission resp ──────────────────────────────────────▶│
      │                    │                   │                  │
      │                    │◀─ Resume exec ──────────────────────│
      │                    │                   │                  │
      │◀─ Progress update ──│◀─ Progress event ──│                  │
      │                    │                   │                  │
      │                    │◀─ Worker done ─────│                  │
      │                    │   Write to Mailbox │                  │
      │                    │                   │                  │
      │◀─ All workers done ─│                   │                  │
```

---

## 11.7 Module Dependency Graph

```
                   ┌─────────────────┐
                   │  bootstrap/     │
                   │  state.ts       │  ← Global state (minimal deps)
                   └────────┬────────┘
                            │ (all modules depend on this)
           ┌────────────────┼────────────────┐
           │                │                │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼───────┐
    │  utils/     │  │  services/  │  │  types/     │
    │(utility fns)│  │(core svcs)  │  │(type defs)  │
    └──────┬──────┘  └──────┬──────┘  └─────┬───────┘
           │                │               │
           └────────┬───────┘               │
                    │                       │
             ┌──────▼──────┐                │
             │  query.ts   │◀───────────────┘
             │ (main loop) │
             └──────┬──────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
  ┌─────▼─────┐ ┌───▼───┐ ┌────▼────┐
  │  tools/   │ │skills/│ │tasks/   │
  │(51 tools) │ │(skills│ │(tasks)  │
  └─────┬─────┘ └───┬───┘ └────┬────┘
        │           │          │
        └───────────┴──────────┘
                    │
             ┌──────▼──────┐
             │  Claude API │
             └─────────────┘
```

---

## 11.8 Data Flow Diagram: A Complete Tool Call

```
User prompt
    │
    ▼
[Message added to messages array]
    │
    ▼
[buildEffectiveSystemPrompt]
  Override? → Coordinator? → Agent(Proactive)? → Agent? → Custom? → Default
  + Append
    │
    ▼
[getAttachmentMessages]
  + Memory files (CLAUDE.md)
  + Agent list delta
  + Deferred tool delta
  + MCP instructions delta
    │
    ▼
[normalizeMessagesForAPI]
  Filter internal message types
  Normalize format
    │
    ▼
[queryModelWithStreaming]
  tools: [directly disclosed tools]
  system: [systemPrompt]
  messages: [normalized messages]
    │
    ▼
[Streaming response]
  text_blocks → yield → UI display
  tool_use_blocks → collect
    │
    ▼
[runTools] (concurrent execution)
  for each tool_use:
    1. findToolByName()
    2. PreToolUse Hook
    3. checkPermissions()
       - allow → call()
       - deny → error result
       - ask → wait for user
    4. call(input, context)
    5. applyToolResultBudget()
    6. PostToolUse Hook
    return tool_result
    │
    ▼
[Continue?]
  No tool calls → Stop Hooks → return
  maxTurns reached → return
  Token Budget exceeded → return
  Auto Compact → compact → continue
  else → update messages → next round
```

---

## 11.9 Key Design Decision Summary

### Decision 1: AsyncGenerator as the Main Loop

**Choice**: `async function* query()` rather than a plain function

**Rationale**:
- Streaming responses can be forwarded to the UI in real time via `yield`
- Errors propagate through the generator's throw mechanism
- The `using` keyword enables automatic resource cleanup
- Callers can terminate early via `.return()`

### Decision 2: AsyncLocalStorage for In-Process Isolation

**Choice**: AsyncLocalStorage rather than multiple processes or threads

**Rationale**:
- No inter-process communication overhead
- Subagents can access the parent agent's memory structures directly (with controlled isolation)
- Isolation granularity is controlled via `cloneFileStateCache()`
- Permission sync is handled through a mailbox rather than IPC

### Decision 3: Deferred Skill and Tool Disclosure

**Choice**: `shouldDefer` + ToolSearch rather than disclosing everything upfront

**Rationale**:
- Full schemas for 50+ tools consume a large number of tokens
- Most sessions use only a small subset of tools
- Claude discovers tools on demand via ToolSearch without losing capability

### Decision 4: Diminishing Returns Detection

**Choice**: Check incremental trends, not just percentages

**Rationale**:
- Pure percentage checks cannot detect "spinning" loops
- "Spinning" is only flagged when 2 consecutive deltas are both < 500 tokens
- Requires `continuationCount >= 3` as a precondition to avoid false positives at startup

### Decision 5: Prompt Cache Byte-Level Consistency

**Choice**: Fork subagent receives the already-rendered prompt; it is not regenerated

**Rationale**:
- GrowthBook (feature flags) may return different values during "cold→warm" transitions
- Regenerating the system prompt could break the Prompt Cache
- Passing the parent agent's already-rendered bytes guarantees byte-level identity

### Decision 6: Layered Context Restoration

**Choice**: After compaction, restore by priority and budget (≤5K per file, ≤5K per skill, ≤25K total skills, ≤50K overall)

**Rationale**:
- Unlimited restoration would consume a large number of tokens every compact cycle
- The beginning of skill files is most important; truncation is better than omission
- Fixed budgets guarantee predictable token consumption

---

## 11.10 File Count Statistics (Claude Code Source Scale)

| Module | File count | Key files |
|--------|-----------|-----------|
| `src/tools/` | 51 tool directories | AgentTool, BashTool, TodoWriteTool, SkillTool... |
| `src/utils/` | 200+ utility functions | systemPrompt.ts, attachments.ts, hooks.ts... |
| `src/services/` | Core services | api/claude.ts, compact/compact.ts... |
| `src/tasks/` | 8 task types | LocalAgentTask, InProcessTeammateTask... |
| `src/types/` | Type definitions | hooks.ts, message.ts... |
| `src/hooks/` | React hooks | 87 UI hooks |
| `src/skills/` | Skill loading | loadSkillsDir.ts, bundledSkills.ts |

---

This architecture overview covers Claude Code's core operating mechanisms. Each preceding chapter dives deep into a specific module — reading alongside the source code is highly recommended.
