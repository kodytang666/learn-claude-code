# Chapter 6: Skills System — Deferred Loading and On-Demand Search

> Source location: `src/skills/loadSkillsDir.ts`, `src/tools/SkillTool/`, `src/services/skillSearch/`

---

## 6.1 What Is a Skill?

A Skill is an **extensible behavior unit** in Claude Code. It is essentially a Markdown file containing instructions for Claude. When Claude invokes the Skill tool, the corresponding Markdown file's content is injected into the context, and Claude acts according to those instructions.

**Analogy**:
- Tool = a concrete function (Bash, FileRead, ...)
- Skill = an operations manual ("how to debug systematically", "how to write tests", ...)

---

## 6.2 Skill File Format

```markdown
---
name: systematic-debugging
description: Systematic debugging process
whenToUse: Use when encountering a bug or failing tests
effort: high
---

# Systematic Debugging

## Step 1: Reproduce the Problem
First confirm the problem can be reliably reproduced...

## Step 2: Understand the Root Cause
Don't jump straight to fixing — understand why first...

## Step 3: Minimize the Reproduction Case
Find the smallest code snippet that reproduces the issue...
```

The frontmatter (YAML header) contains metadata:
- `name`: Skill identifier
- `description`: Short description (used for skill search)
- `whenToUse`: When to use it (Claude uses this to decide whether to invoke it)
- `effort`: Effort level (low/medium/high)

**Why frontmatter matters:**
[src/skills/loadSkillsDir.ts:212](../../src/skills/loadSkillsDir.ts#L212)
The `name`, `description`, and `whenToUse` from frontmatter are used to build the **skill_listing**, which is injected into the conversation context to inform the LLM about available skills and when to use them. The more accurate the frontmatter descriptions, the better the LLM will be at invoking the right skill at the right time.

**Tip:** The combined length of `whenToUse` and `description` should ideally not exceed 250 characters. The reason will be explained later.

---

## 6.3 Skill Directory Scanning

[src/skills/loadSkillsDir.ts:67](../../src/skills/loadSkillsDir.ts#L67)

```typescript
export type LoadedFrom =
  | 'commands_DEPRECATED'   // Legacy format compatibility
  | 'skills'                // Standard skills directory
  | 'plugin'                // Plugin source
  | 'managed'               // Admin-configured
  | 'bundled'               // Built-in skills
  | 'mcp'                   // Skills provided by MCP servers
```

Scanned directories (by loading phase — **does not represent shadowing priority**, see 6.3.1):

```
Directory scan sources:
- policySettings-specified directories     ← Enterprise policy skills
- ~/.claude/skills/                        ← User global skills
- /project/.claude/skills/                 ← Project-level skills (including parent directories)
- --add-dir extra directories
- bundled (built-in skills)                ← Claude Code's built-in skills
- plugin directories                       ← Plugin-provided skills
- MCP server-registered skills             ← Dynamically provided by MCP
```

### Loading Flow

[src/skills/loadSkillsDir.ts:78](../../src/skills/loadSkillsDir.ts#L78)
```typescript
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string { ... }

// Main loading function:
async function loadSkillsDir(dir: string, source: LoadedFrom): Promise<Command[]> {
  // 1. Scan directory, find all .md files
  const files = await loadMarkdownFilesForSubdir(dir, 'skills')
  
  // 2. Parse frontmatter for each file
  for (const file of files) {
    const { frontmatter, body } = parseFrontmatter(file.content)
    
    // 3. Extract metadata
    const name = frontmatter.name ?? basename(file.path, '.md')
    const description = coerceDescriptionToString(frontmatter.description)
    const whenToUse = frontmatter.whenToUse
    const effort = parseEffortValue(frontmatter.effort)
    
    // 4. Estimate token count (frontmatter only, not full content)
    const tokenEstimate = roughTokenCountEstimation(
      `${name}\n${description}\n${whenToUse}`
    )
    
    // 5. Parse hooks (skills can include hook configuration)
    const hooks = parseShellFrontmatter(frontmatter.hooks)
    
    // 6. Register as Command object
    skills.push({
      type: 'prompt',
      name,
      description,
      whenToUse,
      effort,
      tokenEstimate,
      filePath: file.path,
      source,
      // Note: body content is NOT loaded here!
    })
  }
  
  // 7. Deduplication (symlinks and duplicate paths)
  return deduplicateSkills(skills)
}
```

**Key design**: The loading phase only reads frontmatter, not the skill body. Body content is read only when Claude actually invokes the Skill.

### 6.3.1 Shadowing Order for Same-Named Skills

The shadowing order is determined by **two-stage merge logic**, both using first-wins (first entry wins).

#### Stage 1: `loadAllCommands()` determines relative order of sources

[src/commands.ts:460-468](../../src/commands.ts#L460-L468)
```typescript
return [
  ...bundledSkills,        // 1. Built-in skills (listed first, Array.find() matches first → highest priority)
  ...builtinPluginSkills,  // 2. Built-in plugin skills
  ...skillDirCommands,     // 3. Directory-scanned skills (user/project skills here; see below for details)
  ...workflowCommands,     // 4. Workflow commands
  ...pluginCommands,       // 5. Plugin commands
  ...pluginSkills,         // 6. Plugin skills
  ...COMMANDS(),           // 7. Built-in slash commands (listed last → lowest priority)
]
```

#### Stage 2: Internal order within `skillDirCommands` (directory scan results)

[src/skills/loadSkillsDir.ts:717-723](../../src/skills/loadSkillsDir.ts#L717-L723)
```typescript
const allSkillsWithPaths = [
  ...managedSkills,          // policySettings (enterprise policy, first → highest priority)
  ...userSkills,             // userSettings (~/.claude/skills/)
  ...projectSkillsNested,    // projectSettings (/project/.claude/skills/, including parent dirs)
  ...additionalSkillsNested, // --add-dir extra directories
  ...legacyCommands,         // Legacy /commands/ directory (last → lowest priority)
]
// first-wins deduplication (only for same physical files, compared using realpath)
```

**Relationship between Skills and Commands:** At the code level, skill commands share the same `Command` type (both `type: 'prompt'`), can be invoked by the LLM via SkillTool, and can also be called manually by users. The `.claude/commands/` directory is tagged as `commands_DEPRECATED` with the lowest priority; the author's intent is for users to use `.claude/skills/` instead. However, much `commands` naming remains in the code — this is legacy baggage and doesn't need to be distinguished conceptually.

#### Stage 3: MCP appended last

[src/tools/SkillTool/SkillTool.ts:92-93](../../src/tools/SkillTool/SkillTool.ts#L92-L93)
```typescript
return uniqBy([...localCommands, ...mcpSkills], 'name')
// MCP skills appended after all local skills; lowest priority
```

#### Final Shadowing Priority (highest to lowest)

| Priority | Source | Description |
|----------|--------|-------------|
| 1 (highest) | `bundledSkills` | Claude Code compile-time built-in skills |
| 2 | `builtinPluginSkills` | Built-in plugin skills |
| 3 | `policySettings` (managedSkills) | Enterprise admin policy directory |
| 4 | `userSettings` | `~/.claude/skills/` |
| 5 | `projectSettings` | `/project/.claude/skills/` (including parent dirs) |
| 6 | `--add-dir` extra directories | |
| 7 | `pluginCommands` / `pluginSkills` | Plugin-provided skills |
| 8 (lowest) | `mcp` | MCP server dynamically registered skills |

**Path deduplication**: Deduplication within `skillDirCommands` only applies to the **same physical file** (resolved via `realpath` after following symlinks). If two directories each have a different `writing-article.md`, both enter the `commands[]` array.

**Same-name shadowing**: `allCommands` is ordered by priority. When iterating to build `skill_listing`, the skill name from higher-priority sources is added to a Set first, and later same-named skills are filtered out.

### 6.3.2 Loading Cache and Cache Invalidation

**The skill list is not re-scanned from disk on every conversation or tool call.** Throughout the process lifecycle, scan results are cached in memory via two-layer `memoize`:

[src/skills/loadSkillsDir.ts:638](../../src/skills/loadSkillsDir.ts#L638)
```typescript
// key: cwd
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => { /* scan disk */ }
)

// [src/commands.ts:449](../../src/commands.ts#L449)
// key: cwd, merges all sources
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

For the same working directory, no matter how many conversations or tool call turns occur, `getCommands()` returns directly from memory without repeated disk scanning.

#### Three Cache Invalidation Triggers

| Trigger | Clear Function | Source Location |
|---------|----------------|-----------------|
| Skill directory file change (chokidar file watcher) | `clearCommandsCache()` | [src/utils/skills/skillChangeDetector.ts:274](../../src/utils/skills/skillChangeDetector.ts#L274) |
| Plugin install / uninstall | `clearCommandsCache()` | [src/utils/plugins/cacheUtils.ts:46](../../src/utils/plugins/cacheUtils.ts#L46) |
| User runs `/clear caches` | `clearCommandsCache()` | [src/commands/clear/caches.ts:60](../../src/commands/clear/caches.ts#L60) |

`clearCommandsCache()` clears all relevant cache layers in sequence:

[src/cli/print.ts:1824](../../src/cli/print.ts#L1824)
```typescript
const unsubscribeSkillChanges = skillChangeDetector.subscribe(() => {
  clearCommandsCache()
  void getCommands(cwd()).then(newCommands => {
    currentCommands = newCommands   // Immediately update
  })
})
```

#### File Watcher Implementation Details

File change monitoring is based on **chokidar**, watching all skill directories (`src/utils/skills/skillChangeDetector.ts`). Two key time constants prevent excessive triggering:

```typescript
const FILE_STABILITY_THRESHOLD_MS = 1000  // Wait for file write to stabilize
const RELOAD_DEBOUNCE_MS = 300            // Debounce: multiple changes within 300ms merged into one
```

Under the Bun runtime, chokidar uses **stat polling** (rather than native `fs.watch`), with a 2-second polling interval:

[src/utils/skills/skillChangeDetector.ts:62](../../src/utils/skills/skillChangeDetector.ts#L62)
```typescript
const USE_POLLING = typeof Bun !== 'undefined'
const POLLING_INTERVAL_MS = 2000
```

The reason: Bun's `FSWatcher` has a known deadlock bug (`oven-sh/bun#27469`). When the main thread closes the watcher, the file monitoring thread is delivering an event — they wait on each other causing a hang.

#### Complete Lifecycle

```
Process starts
  │
  ▼
First getCommands(cwd)
  └─ Scan disk, write results to memoize cache
  │
  ▼
All subsequent calls (same cwd)
  └─ Read from memory, no disk access
  │
  ▼
Skill file changes (or plugin changes / /clear caches)
  └─ clearCommandsCache() clears all cache layers
  │
  ▼
Next getCommands(cwd)
  └─ Re-scan disk, write new cache
```

---

## 6.4 How the LLM Knows Which Skills Are Available

### 6.4.1 SkillTool Only Tells Claude "How to Invoke"

SkillTool's inputSchema (sent with every API request) is:

[src/tools/SkillTool/SkillTool.ts:291-298](../../src/tools/SkillTool/SkillTool.ts#L291-L298)
```typescript
export const inputSchema = lazySchema(() =>
  z.object({
    skill: z
      .string()
      .describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
    args: z.string().optional().describe('Optional arguments for the skill'),
  }),
)
```

The `skill` parameter is **free-form text**, with no enum, no enumeration of available skill names. From the SkillTool schema alone, Claude can only learn "there is a Skill tool that needs a skill name" — it **doesn't know what skills are currently available**.

### 6.4.2 skill_listing AttachmentMessage: Providing Discovery Capability

The skill list is injected into the conversation context via a `skill_listing` type `AttachmentMessage`:

```
system-reminder: The following skills are available for use with the Skill tool:

- commit: Create a git commit - Use when the user asks to commit changes.
- review-pr: Review a pull request - Use when user asks to review a PR.
- systematic-debugging: Systematic debugging process - Use when encountering a bug or failing tests
```

Each skill entry contains only `name : description + whenToUse` (frontmatter fields). The skill **body is not included**.

For more about `AttachmentMessage`, see Chapter 2.

**Why description + whenToUse should not exceed 250 characters:**

[src/tools/SkillTool/prompt.ts:43-65](../../src/tools/SkillTool/prompt.ts#L43-L65)
```typescript
// verbose whenToUse strings waste turn-1 cache_creation tokens without improving match rate. 
function getCommandDescription(cmd: Command): string {
  const desc = cmd.whenToUse
    ? `${cmd.description} - ${cmd.whenToUse}`
    : cmd.description
  // Truncated to MAX_LISTING_DESC_CHARS = 250 characters
  return desc.length > MAX_LISTING_DESC_CHARS
    ? desc.slice(0, MAX_LISTING_DESC_CHARS - 1) + '…'
    : desc
}
```

### 6.4.3 Injection Timing and Position of skill_listing

**Injection timing**: At the start of each conversation turn when user input is being processed, `getAttachmentMessages()` checks `sentSkillNames` (a module-level Map). If `sentSkillNames` contains skills not yet announced, a new skill_listing attachment is generated.

[src/utils/attachments.ts:875](../../src/utils/attachments.ts#L875)
```typescript
maybe('skill_listing', () => getSkillListingAttachments(context))
```

**Position**: The `skill_listing AttachmentMessage` is appended after the user message:

[src/utils/processUserInput/processTextPrompt.ts:96-99](../../src/utils/processUserInput/processTextPrompt.ts#L96-L99)
```typescript
return {
  messages: [userMessage, ...attachmentMessages],
  //          ^user message  ^skill_listing follows immediately
  shouldQuery: true,
}
```

### 6.4.4 Format Conversion Before Sending

Before each API call, `normalizeMessagesForAPI()` converts `AttachmentMessage` to `<system-reminder>`:

[src/utils/messages.ts:3728-3736](../../src/utils/messages.ts#L3728-L3736)
```typescript
case 'skill_listing': {
  if (!attachment.content) return []
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `The following skills are available for use with the Skill tool:\n\n${attachment.content}`,
      isMeta: true,
    }),
  ])
}
```

So: every time an API request is made, `AttachmentMessage(skill_listing)` in `messages[]` is automatically converted to a `<system-reminder>`-wrapped user message.

**messages[] structure after the first turn**:

```
[
  { role: "user", content: "help me write a function" },
  { role: "user", content: "<system-reminder>\n 
      The following skills are available...
    </system-reminder>" },
  { role: "assistant", content: "Sure, I'll generate the function..." },
]
```

### 6.4.5 Delta Tracking: Preventing Duplicate skill_listing Injection

[src/utils/attachments.ts:2607](../../src/utils/attachments.ts#L2607)
```typescript
const sentSkillNames = new Map<string, Set<string>>()
// key: agentId (undefined = main thread)
// value: set of already-announced skill names
```

`getSkillListingAttachments()` checks `sentSkillNames` each turn and only injects skills **not already in the Map** as new entries. Historical `AttachmentMessage`s are not re-pushed; `normalizeMessagesForAPI` renders them every turn automatically.

Conditions that trigger re-injection:
- Skill file changes → `resetSentSkillNames()` clears the Map → all skills become "new" to the Map → next turn injects all current skills (still delta logic, just Map is cleared so delta = full set)
- `--resume` restoring session → `suppressNextSkillListing()`. Retrieves existing skill_listing from history to avoid duplication.

### 6.4.6 Behavior After Compact

When Auto Compact triggers, `compact.ts:213-220` explicitly filters out `skill_listing`:

```typescript
// skill_listing is removed from messages[] and doesn't enter the compact summary
return messages.filter(
  m => !(m.type === 'attachment' && m.attachment.type === 'skill_listing')
)
```

After compaction, `sentSkillNames` is not reset and skills are not re-injected. Content of already-invoked skills is preserved via `invoked_skills` attachment, so the LLM can invoke them. But for skills never invoked, the LLM will "forget" they exist after compact. The rationale:

> skill_listing (~4K tokens) post-compact is pure cache_creation with marginal benefit. The model still has SkillTool in its schema and invoked_skills attachment (below) preserves used-skill content.

Think about it: when will the LLM re-encounter the skill_listing?

---

## 6.5 Deferred Skill Loading

The skills system has two layers of "laziness":

### Layer 1: Skill content is read during scanning but deferred from context injection

During scanning, `loadSkillsDir()` reads the complete SKILL.md content via `fs.readFile`, storing it in the `getPromptForCommand` closure of the Command object returned by `createSkillCommand()`:

[src/skills/loadSkillsDir.ts:295-343](../../src/skills/loadSkillsDir.ts#L295-L343) (createSkillCommand)
```typescript
const markdownContent = await fs.readFile(filePath, 'utf-8')  // Read during scan!

return {
  ...
  getPromptForCommand: async () => markdownContent,  // Closure holds the content
}
```

"Deferred" means: **the content is not injected into the conversation context at scan time**. Only when Claude actively calls `Skill("xxx")` does `SkillTool.call()` retrieve the content via `getPromptForCommand()` and inject it as a tool result.

### Layer 2: skill_listing only contains summaries; full body injected on demand

Claude only sees `name + description + whenToUse` in `skill_listing`. The full instructions only appear in context after invocation:

```
LLM sees in skill_listing (summary):
- systematic-debugging: Systematic debugging process - Use when encountering a bug

LLM invokes Skill("systematic-debugging"), tool result (full content):
# Systematic Debugging
## Step 1: Reproduce the Problem
First confirm the problem can be reliably reproduced...
(Complete SKILL.md body)
```

---

## 6.6 Skill Search Prefetch

> **⚠️ Status: Pending implementation.**

This mechanism only takes effect when the `EXPERIMENTAL_SKILL_SEARCH` feature flag is enabled ([src/query.ts:66-68](../../src/query.ts#L66-L68)):

**Behavior changes when enabled**:

1. **`skill_listing` narrowed**: No longer lists user-defined skills; only built-in skills + MCP are listed.
2. **On-demand async search**: In the Agent Loop, while the LLM is streaming output, Claude Code runs a background semantic search on the user message to match relevant skills from the full skill library.
3. **Injects `skill_discovery`**: Search results are appended as a separate `skill_discovery` AttachmentMessage to `messages[]`, also converted to `<system-reminder>` for the LLM.

**Design intent** (from code comments):
> "Discovery runs while the model streams and tools execute; awaited post-tools alongside the memory prefetch consume. Replaces the blocking assistant_turn path that ran inside getAttachmentMessages (97% of those calls found nothing in prod)."

In other words: avoid blocking the main flow; hide the search within LLM response time. 97% of production calls found no new skill — making it async avoids wasting resources.

---

## 6.7 Flowchart: Complete Skill Lifecycle

```
Claude Code starts
      │
      ▼
loadSkillsDir()                        ← Scan all skill directories
  ├─ fs.readFile reads full .md files (full read into memory at startup)
  ├─ Parse frontmatter (name/description/whenToUse)
  ├─ Deduplication
  └─ createSkillCommand()            ← Register as Command object (memoize cache)
       └─ markdownContent captured in getPromptForCommand closure
          (on invocation, content retrieved from closure — no more disk reads)
      │
      ▼
User inputs first message
      │
      ▼
processUserInput()
  └─ getAttachmentMessages(input, ...)
       └─ getSkillListingAttachments()
            ├─ Check sentSkillNames (empty on first call)
            ├─ formatCommandsWithinBudget()  ← Only take name+desc+whenToUse
            └─ Return skill_listing Attachment
      │
      ▼
processTextPrompt()
  └─ messages: [UserMessage, AttachmentMessage(skill_listing)]
  //  skill_listing immediately follows user message in messages[]
      │
      ▼
Before each API call: normalizeMessagesForAPI()
  └─ AttachmentMessage(skill_listing)
       → wrapMessagesInSystemReminder(...)
       → <system-reminder>The following skills are available...</system-reminder>
  // Claude sees the skill list every turn (because AttachmentMessage stays in messages[])
      │
      ▼
Claude decides to invoke Skill based on skill_listing
  └─ tool_use: Skill({ skill: "systematic-debugging" })
      │
      ▼
SkillTool.call()
  ├─ findSkillByName()
  ├─ getPromptForCommand()           ← Retrieve full content from closure (no disk read)
  ├─ recordInvokedSkill()            ← Record invocation (for post-compact restoration)
  └─ Return full SKILL.md content to Claude
      │
      ▼
Claude acts according to skill instructions
      │
      ▼ (several turns later, compact may occur)
      │
Auto Compact triggers
  ├─ skill_listing AttachmentMessage filtered out from messages[]
  ├─ sentSkillNames not reset (skill list not re-injected)
  └─ Content of invoked skills preserved by invoked_skills attachment (truncated to 5K each)
```

---

## Summary

| Mechanism | Design Goal | Source Location |
|-----------|-------------|-----------------|
| Directory scanning | Multi-source aggregation (user/project/plugin/MCP) | `src/skills/loadSkillsDir.ts` |
| `AttachmentMessage(skill_listing)` | Inform LLM of available skills (name+desc) | [src/utils/attachments.ts:2661](../../src/utils/attachments.ts#L2661) |
| `sentSkillNames` delta tracking | Only inject new skills; avoid duplicate pushes | [src/utils/attachments.ts:2607](../../src/utils/attachments.ts#L2607) |
| `normalizeMessagesForAPI()` format conversion | AttachmentMessage → `<system-reminder>` | [src/utils/messages.ts:3728](../../src/utils/messages.ts#L3728) |
| Compact filters skill_listing | Prevent skill list from entering summary (wastes tokens) | [src/services/compact/compact.ts:218](../../src/services/compact/compact.ts#L218) |
| Deferred content injection | Full body only injected into context when Skill tool is called | `src/tools/SkillTool/SkillTool.ts` |
| Prefetch preloading | Runs in parallel, doesn't block main flow | `src/services/skillSearch/prefetch.ts` |
| Post-compact restoration of invoked skills | Invoked skill knowledge not lost after compaction | `src/services/compact/compact.ts` |
| Deduplication | Symlink and multi-source deduplication | `deduplicateSkills()` |
