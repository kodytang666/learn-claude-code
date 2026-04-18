# Chapter 10: Worktree and Task Isolation — Git Worktree Sandbox Mechanism

> Source locations: [src/tools/EnterWorktreeTool/EnterWorktreeTool.ts](../../src/tools/EnterWorktreeTool/EnterWorktreeTool.ts), [src/tools/ExitWorktreeTool/ExitWorktreeTool.ts](../../src/tools/ExitWorktreeTool/ExitWorktreeTool.ts), [src/utils/worktree.ts](../../src/utils/worktree.ts)

---

## 10.1 What is Git Worktree?

Git Worktree allows you to **check out multiple branches simultaneously** within the same Git repository, with each branch living in a separate directory.

```bash
# Normal case: one directory, one branch at a time
/my-project/  (main branch)

# With worktree: multiple directories, each on a different branch
/my-project/                            (main branch, primary directory)
/my-project/.claude/worktrees/fix-bug/  (fix-bug worktree)
/my-project/.claude/worktrees/feat-a/   (feat-a worktree)
```

Claude Code places all worktrees inside **`.claude/worktrees/` at the main repository root** (not the parent directory), keeping them isolated from the primary workspace.

---

## 10.2 Why Does Claude Code Need Worktrees?

**Problem scenario**:
- The user is working on feature A on the main branch
- An urgent bug fix is needed, or a PR review must happen in an isolated environment
- Switching branches directly would interrupt feature A

**Solution**:
- Use a Worktree to create an isolated directory
- Claude works on the task inside that isolated directory
- The primary directory remains completely unaffected
- When done, exit with ExitWorktree

---

## 10.3 Worktree Path and Branch Naming

```typescript
// src/utils/worktree.ts

// slug transformation rule: replace "/" with "+" (for nested structures)
function flattenSlug(slug: string): string {
  return slug.replace(/\//g, '+')
}

// worktree directory path formula:
// {gitRoot}/.claude/worktrees/{flattenSlug(slug)}
const worktreePath = path.join(
  gitRoot,
  '.claude', 'worktrees',
  flattenSlug(slug)
)

// branch name formula:
// worktree-{flattenSlug(slug)}
// uses git -B to force-create (resets if already exists)
const branchName = `worktree-${flattenSlug(slug)}`
```

**Examples**:

| slug | Worktree path | Branch name |
|------|--------------|-------------|
| `fix-bug` | `.claude/worktrees/fix-bug` | `worktree-fix-bug` |
| `user/feat` | `.claude/worktrees/user+feat` | `worktree-user+feat` |

---

## 10.4 validateWorktreeSlug

```typescript
// src/utils/worktree.ts

export function validateWorktreeSlug(slug: string): void {
  // Each segment (separated by /) allows only letters, digits, dots, underscores, hyphens
  // Nested structures like "user/feat-1" are supported
  // Total length must not exceed 64 characters
  const segmentPattern = /^[a-zA-Z0-9._-]+$/
  const segments = slug.split('/')
  
  if (slug.length > 64) {
    throw new Error('worktree slug cannot exceed 64 characters')
  }
  
  for (const segment of segments) {
    if (!segmentPattern.test(segment)) {
      throw new Error(`slug segment "${segment}" may only contain letters, digits, dots, underscores, and hyphens`)
    }
  }
}
```

Note: the actual allowed character set is broader than earlier documentation described — `.` and `_` are permitted, and `/`-separated nested structures are supported.

---

## 10.5 EnterWorktreeTool

EnterWorktreeTool accepts only one optional `name` parameter — there is no `branch` or `base_branch`:

```typescript
// src/tools/EnterWorktreeTool/EnterWorktreeTool.ts

export const EnterWorktreeTool = buildTool({
  name: 'EnterWorktree',
  
  inputSchema: z.object({
    name: z.string().optional(),  // ← only name, no branch/base_branch
  }),
  
  async call(input, context) {
    
    // 1. Prevent nested entry
    const currentWorktree = getCurrentWorktreeSession()
    if (currentWorktree) {
      return { error: 'Already inside a worktree; cannot nest into another' }
    }
    
    // 2. Resolve to the main repo root (even if currently in a subdirectory)
    const mainRepoRoot = await resolveToMainRepoRoot()
    
    // 3. Determine slug: prefer input.name, fall back to getPlanSlug()
    const slug = input.name ?? getPlanSlug()
    
    // 4. Create worktree (core operation)
    const worktreeSession = await createWorktreeForSession(
      getSessionId(),
      slug,
      tmuxSessionName,   // tmux integration
      { prNumber }       // PR checkout support
    )
    
    // 5. Switch working directory (change dir first, then clear caches)
    process.chdir(worktreeSession.worktreePath)
    setCwd(worktreeSession.worktreePath)
    setOriginalCwd(worktreeSession.worktreePath)
    
    // 6. Persist worktree state
    saveWorktreeState(worktreeSession)
    
    // 7. Clear caches (must happen after setCwd so caches reload from the new directory)
    clearSystemPromptSections()
    clearMemoryFileCaches()           // note: plural form
    getPlansDirectory.cache?.clear?.()
    
    return { success: true, worktreePath: worktreeSession.worktreePath }
  }
})
```

**Key detail**: cache clearing happens **after** `setCwd`, not before. This ensures that subsequent CLAUDE.md and system prompt loads come from the new worktree directory.

---

## 10.6 createWorktreeForSession In Depth

```typescript
// src/utils/worktree.ts

export async function createWorktreeForSession(
  sessionId: string,
  slug: string,
  tmuxSessionName?: string,
  options?: { prNumber?: number }
): Promise<WorktreeSession>
```

The function signature is completely different from earlier documentation — there are no `branch` or `baseBranch` parameters; the branch name is derived automatically from the slug.

### 10.6.1 fast-resume: Skip Redundant Initialization

```typescript
// src/utils/worktree.ts

// If the worktree directory already exists, attempt a fast resume
const existingSha = await readWorktreeHeadSha(worktreePath)
if (existingSha) {
  // Worktree already present — restore the session directly, skip git fetch/checkout
  return restoreExistingWorktreeSession(worktreePath, sessionId)
}
```

When a user re-enters an existing worktree, there is no need to re-run fetch and checkout — the existing HEAD SHA is read directly for restoration.

### 10.6.2 PR Checkout Support

```typescript
// If prNumber is provided, perform PR checkout
if (options?.prNumber) {
  const prRef = await parsePRReference(options.prNumber)
  // Supports two forms:
  // - "#123" (numeric ID)
  // - GitHub PR URL
  await gitFetchAndCheckoutPR(worktreePath, prRef)
}
```

### 10.6.3 performPostCreationSetup: Post-Creation Initialization

```typescript
// src/utils/worktree.ts

async function performPostCreationSetup(
  worktreePath: string,
  mainRepoPath: string,
): Promise<void> {
  
  // 1. Copy settings.local.json (bring user personalization into the new environment)
  await copyLocalSettings(mainRepoPath, worktreePath)
  
  // 2. Configure core.hooksPath to point at the main repo's hooks
  //    Ensures git hooks fire correctly inside the worktree
  await exec(`git -C ${worktreePath} config core.hooksPath ${mainRepoPath}/.git/hooks`)
  
  // 3. Create directory symlinks (avoids duplicating large directories)
  //    e.g., node_modules -> main repo's node_modules
  await symlinkSharedDirectories(mainRepoPath, worktreePath)
  
  // 4. Process .worktreeinclude file
  //    Similar to .gitignore, specifies which files to copy into the worktree
  await processWorktreeInclude(mainRepoPath, worktreePath)
}
```

`performPostCreationSetup` is a critical initialization step after worktree creation, giving the worktree a complete working environment.

### 10.6.4 sparse-checkout Support

```typescript
// settings.worktree.sparsePaths configuration
// If sparsePaths is set, only check out the specified directories
if (settings.worktree?.sparsePaths?.length) {
  await configureSparseCheckout(worktreePath, settings.worktree.sparsePaths)
  session.usedSparsePaths = true
}
```

For large monorepos, `sparsePaths` lets you check out only the required subdirectories, significantly reducing disk usage and initialization time.

---

## 10.7 WorktreeSession Data Structure

```typescript
// src/utils/worktree.ts

interface WorktreeSession {
  originalCwd: string           // Working directory before entering the worktree
  worktreePath: string          // Path to the worktree directory
  worktreeName: string          // Worktree name (= slug)
  worktreeBranch: string        // Checked-out branch name (= "worktree-{flatSlug}")
  originalBranch: string        // Branch name before entering
  originalHeadCommit: string    // HEAD commit SHA before entering
  sessionId: string             // Claude session ID
  tmuxSessionName?: string      // Associated tmux session (if any)
  hookBased?: boolean           // Whether created by a WorktreeCreate hook
  creationDurationMs?: number   // Time taken to create (milliseconds)
  usedSparsePaths?: boolean     // Whether sparse-checkout was enabled
}
```

The struct has many more fields than earlier documentation showed. `originalHeadCommit` is critical for the safety check in ExitWorktreeTool.

---

## 10.8 State Persistence

```typescript
// src/utils/worktree.ts (not sessionStorage.ts as earlier docs suggested)

// Save worktree state to project config
saveCurrentProjectConfig({ activeWorktreeSession: worktreeSession })

// Read back
const activeSession = getCurrentProjectConfig().activeWorktreeSession
```

State is saved in the **project config**, not in a standalone `sessions/{id}/worktree.json` file.

A module-level variable tracks the current session:

```typescript
// src/utils/worktree.ts
// Module-level variable (not globalState in bootstrap/state.ts)
let currentWorktreeSession: WorktreeSession | null = null

export function getCurrentWorktreeSession(): WorktreeSession | null {
  return currentWorktreeSession
}
```

---

## 10.9 ExitWorktreeTool

ExitWorktreeTool accepts two parameters and includes a safety check:

```typescript
// src/tools/ExitWorktreeTool/ExitWorktreeTool.ts

export const ExitWorktreeTool = buildTool({
  name: 'ExitWorktree',
  
  inputSchema: z.object({
    action: z.enum(['keep', 'remove']),    // Required: keep or delete the worktree after exit
    discard_changes: z.boolean().optional(), // Whether to force-discard uncommitted changes
  }),
  
  async call(input, context) {
    const worktreeState = getCurrentWorktreeSession()
    if (!worktreeState) {
      return { error: 'Not currently inside a worktree' }
    }
    
    if (input.action === 'remove') {
      // Safety check: count uncommitted changes
      const changes = await countWorktreeChanges(
        worktreeState.worktreePath,
        worktreeState.originalHeadCommit,  // baseline commit for comparison
      )
      
      if (changes > 0 && !input.discard_changes) {
        // Uncommitted changes exist and discard not explicitly confirmed → refuse deletion
        return {
          error: `worktree has ${changes} uncommitted change(s). ` +
                 `Set discard_changes: true to force removal.`
        }
      }
      
      // Clean up tmux session (if any)
      if (worktreeState.tmuxSessionName) {
        await killTmuxSession(worktreeState.tmuxSessionName)
      }
      
      await cleanupWorktree(worktreeState.worktreePath)
      
    } else {  // action === 'keep'
      await keepWorktree(worktreeState.worktreePath)
    }
    
    // Restore to original directory (required for both keep and remove)
    await restoreSessionToOriginalCwd(worktreeState)
    
    return { success: true }
  }
})
```

### 10.9.1 restoreSessionToOriginalCwd Full Flow

```typescript
async function restoreSessionToOriginalCwd(state: WorktreeSession): Promise<void> {
  // 1. Switch back to the original directory
  setCwd(state.originalCwd)
  setOriginalCwd(state.originalCwd)
  
  // 2. Update project root and hooks config snapshot if needed
  if (shouldUpdateProjectRoot) {
    setProjectRoot(state.originalCwd)
    await updateHooksConfigSnapshot()
  }
  
  // 3. Clear worktree state (write null)
  saveWorktreeState(null)
  
  // 4. Clear all caches (same order as EnterWorktreeTool)
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache?.clear?.()
}
```

---

## 10.10 keepWorktree vs cleanupWorktree

The two functions represent two completely different exit strategies:

```typescript
// keepWorktree: retain the worktree directory and branch for future use
async function keepWorktree(worktreePath: string): Promise<void> {
  // Only perform necessary cleanup (e.g., close file handles); delete nothing
  // The worktree directory and git branch are fully preserved
}

// cleanupWorktree: fully delete the worktree
async function cleanupWorktree(worktreePath: string): Promise<void> {
  // 1. git worktree remove --force {path}
  // 2. Delete the corresponding git branch worktree-{slug}
  // 3. Remove the .claude/worktrees/{slug} directory
}
```

| Scenario | Recommended action |
|----------|--------------------|
| Task complete, PR already merged | `remove` |
| Task paused, will continue later | `keep` |
| Switching to main task, returning later | `keep` |
| Exploratory task, no need to preserve | `remove` |

---

## 10.11 createAgentWorktree vs createWorktreeForSession

The two functions serve different scenarios and differ fundamentally:

```typescript
// createWorktreeForSession: for user sessions (called by EnterWorktreeTool)
// - Uses getCwd() to determine the base directory
// - Modifies the global currentWorktreeSession state
// - Persists state to project config

// createAgentWorktree: for subagents (called by the Agent tool)
export async function createAgentWorktree(
  agentId: string,
  slug: string,
): Promise<string> {
  
  // Key difference 1: uses findCanonicalGitRoot() instead of getCwd()
  // Always creates from the main repo root, even if currently in a subdirectory
  const gitRoot = await findCanonicalGitRoot()
  
  // Key difference 2: does NOT modify global currentWorktreeSession
  // The subagent's worktree is independent and does not affect the main session state
  const worktreePath = await doCreateWorktree(gitRoot, slug)
  
  return worktreePath
}
```

**Key distinction**: `createAgentWorktree` never touches global worktree state — the subagent's worktree lifecycle is completely independent.

---

## 10.12 EPHEMERAL_WORKTREE_PATTERNS: Identifying Temporary Worktrees

```typescript
// src/utils/worktree.ts

// Regex patterns identifying temporary (auto-cleanable) worktrees
export const EPHEMERAL_WORKTREE_PATTERNS = [
  /^agent-/,    // Worktrees created by subagents (createAgentWorktree)
  /^wf_/,       // Workflow task worktrees
  /^bridge-/,   // Bridge tool worktrees
  /^job-/,      // Batch job worktrees
]

export function isEphemeralWorktree(slug: string): boolean {
  return EPHEMERAL_WORKTREE_PATTERNS.some(pattern => pattern.test(slug))
}
```

Used by `cleanupStaleAgentWorktrees()` to determine which worktrees are eligible for automatic cleanup.

---

## 10.13 Automatic Cleanup: cleanupStaleAgentWorktrees

```typescript
// src/utils/worktree.ts

// Clean up stale temporary worktrees (inactive for 30 days)
export async function cleanupStaleAgentWorktrees(): Promise<void> {
  const allWorktrees = await listWorktrees(gitRoot)
  const now = Date.now()
  const STALE_THRESHOLD_MS = 30 * 24 * 60 * 60 * 1000  // 30 days
  
  for (const wt of allWorktrees) {
    if (!isEphemeralWorktree(wt.slug)) continue
    
    const lastActive = await getWorktreeLastActiveTime(wt.path)
    if (now - lastActive > STALE_THRESHOLD_MS) {
      await cleanupWorktree(wt.path)
    }
  }
}
```

Only worktrees matching `EPHEMERAL_WORKTREE_PATTERNS` are automatically cleaned up. Worktrees created manually by the user (via EnterWorktreeTool) are not deleted automatically by default.

---

## 10.14 Worktree and Subagent Isolation

When a subagent runs inside a worktree, it maintains its own context through `AsyncLocalStorage`:

```
Main agent (main branch, cwd: /project)
        │
        ├─ Enter worktree (cwd: /project/.claude/worktrees/fix-bug)
        │
        └─ Create subagent
              │
              ├─ Subagent's cwd is inherited from the parent's current cwd
              │  (i.e., /project/.claude/worktrees/fix-bug)
              │
              └─ Subagent operates on files inside the worktree
                 without affecting the main agent's file state cache
```

**File state cache cloning** (from [src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts)):

```typescript
// src/utils/swarm/inProcessRunner.ts
const fileStateCache = cloneFileStateCache()
// Subagent receives a deep copy of the parent agent's file cache
// Subagent reads/modifications do not affect the parent's cache
```

---

## 10.15 WorktreeCreate Hook: Full Customization

```typescript
// WorktreeCreate is a special hook:
// If configured, Claude Code delegates worktree creation entirely to the hook
// The hook is responsible for returning the worktreePath

// Example configuration (settings.json):
{
  "hooks": {
    "WorktreeCreate": [{
      "hooks": [{
        "type": "command",
        "command": "/scripts/create-remote-worktree.sh"
      }]
    }]
  }
}
```

The `worktreePath` returned by the hook is recorded in `WorktreeSession.hookBased = true`.

**Use cases**:
- Creating worktrees on a remote server (local + remote collaboration)
- Integrating with CI/CD systems to automatically trigger pipelines
- Custom branch strategies and naming conventions

---

## 10.16 Isolation Level Comparison

| Isolation mechanism | Isolation level | Use case |
|--------------------|-----------------|----------|
| **Worktree + directory isolation** | Filesystem level | Parallel work on different branches |
| **AsyncLocalStorage** | In-process async context | Subagent context (agentId, todo list) |
| **File cache cloning** | In-process memory level | Subagent file read state |
| **AbortController** | Async control flow | Independent cancellation signal per subagent |
| **sparse-checkout** | Git object layer | Directory-level isolation in large monorepos |

---

## 10.17 Complete Flow Diagram

```
User (via Claude) calls EnterWorktree
            │
            ▼
  Already inside a worktree?
       │           │
      YES           NO
       │             │
   Return error  validateWorktreeSlug()
                       │
                       ▼
              resolveToMainRepoRoot()
                       │
                       ▼
              slug = input.name ?? getPlanSlug()
                       │
                       ▼
            Does the worktree directory exist?
               │               │
              YES               NO
               │                │
        fast-resume         WorktreeCreate Hook configured?
        (skip fetch)            │           │
               │               YES           NO
               │                │            │
               │          Execute Hook   git -B worktree-{slug}
               │          hookBased=true     |  + fetch origin
               │               └─────┬───────┘
               │                      │
               └──────────────────────┘
                               │
                               ▼
                   performPostCreationSetup()
                   (copy settings, hooksPath, symlinks)
                               │
                               ▼
                   process.chdir(worktreePath)
                   setCwd / setOriginalCwd
                               │
                               ▼
                   saveCurrentProjectConfig({activeWorktreeSession})
                               │
                               ▼
                   clearSystemPromptSections()   ← after setCwd
                   clearMemoryFileCaches()
                   getPlansDirectory.cache.clear()
                               │
                               ▼
                   Claude works inside the worktree
                               │
                               ▼
                   User calls ExitWorktree(action, discard_changes?)
                               │
                ┌────────────────┴────────────────┐
          action=remove                      action=keep
                │                                 │
    countWorktreeChanges()               keepWorktree()
       Changes exist & not confirmed?              │
        │       │                                 │
       YES      NO                                │
        │       │                                 │
   Return error  cleanupWorktree()                │
                │                                 │
                └──────────────┬──────────────────┘
                                  │
                            restoreSessionToOriginalCwd()
                            (restore setCwd + clear caches + clear state)
```

---

## Summary

| Mechanism | Role | Source location |
|-----------|------|-----------------|
| `EnterWorktreeTool` | Create worktree and switch cwd (only `name?` parameter) | [src/tools/EnterWorktreeTool/](../../src/tools/EnterWorktreeTool/) |
| `ExitWorktreeTool` | Exit worktree (`action: keep\|remove`, with safety check) | [src/tools/ExitWorktreeTool/](../../src/tools/ExitWorktreeTool/) |
| `createWorktreeForSession(sessionId, slug, tmux?, opts?)` | Main worktree creation flow | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `createAgentWorktree(agentId, slug)` | Subagent-dedicated, does not touch global state | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `validateWorktreeSlug()` | Prevent path injection; allows `[a-zA-Z0-9._-]` + `/` | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `flattenSlug()` | `/` → `+` for path and branch name construction | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `performPostCreationSetup()` | Copy settings, configure hooksPath, create symlinks | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `countWorktreeChanges()` | Safety check before `remove` | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `keepWorktree / cleanupWorktree` | Two exit strategies | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `EPHEMERAL_WORKTREE_PATTERNS` | 4 regexes identifying temporary worktrees | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `cleanupStaleAgentWorktrees()` | Auto-cleanup of temporary worktrees after 30 days | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `saveCurrentProjectConfig` | Worktree state persistence | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `cloneFileStateCache()` | Subagent file cache isolation | [src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts) |
