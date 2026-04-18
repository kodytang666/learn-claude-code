# 第 10 章：Worktree 与任务隔离 — Git Worktree 沙箱机制

> 源码位置：[src/tools/EnterWorktreeTool/EnterWorktreeTool.ts](../../src/tools/EnterWorktreeTool/EnterWorktreeTool.ts)、[src/tools/ExitWorktreeTool/ExitWorktreeTool.ts](../../src/tools/ExitWorktreeTool/ExitWorktreeTool.ts)、[src/utils/worktree.ts](../../src/utils/worktree.ts)

---

## 10.1 什么是 Git Worktree？

Git Worktree 允许在同一个 Git 仓库中同时**检出多个分支**，每个分支在不同的目录中工作。

```bash
# 普通情况：一个目录，只能检出一个分支
/my-project/  (main 分支)

# 使用 worktree：多个目录，每个在不同分支
/my-project/                            (main 分支，主目录)
/my-project/.claude/worktrees/fix-bug/  (fix-bug worktree)
/my-project/.claude/worktrees/feat-a/   (feat-a worktree)
```

Claude Code 将所有 worktree 统一存放在**主仓库根目录下的 `.claude/worktrees/` 目录中**（不是 parent 目录），与主工作区隔离。

---

## 10.2 为什么 Claude Code 需要 Worktree？

**问题场景**：
- 用户正在 main 分支做功能 A
- 突然需要修复一个紧急 bug，或者需要在独立环境中处理 PR review
- 如果直接切换分支，会打断功能 A 的工作

**解决方案**：
- 用 Worktree 创建一个隔离目录
- Claude 在隔离目录中处理任务
- 主目录的工作完全不受影响
- 完成后通过 ExitWorktree 退出

---

## 10.3 Worktree 路径与分支命名

```typescript
// src/utils/worktree.ts

// slug 转换规则：将 "/" 替换为 "+"（用于嵌套结构）
function flattenSlug(slug: string): string {
  return slug.replace(/\//g, '+')
}

// worktree 目录路径公式：
// {gitRoot}/.claude/worktrees/{flattenSlug(slug)}
const worktreePath = path.join(
  gitRoot,
  '.claude', 'worktrees',
  flattenSlug(slug)
)

// 分支名公式：
// worktree-{flattenSlug(slug)}
// 使用 git -B 强制创建（如已存在则重置）
const branchName = `worktree-${flattenSlug(slug)}`
```

**示例**：

| slug | worktree 路径 | 分支名 |
|------|--------------|--------|
| `fix-bug` | `.claude/worktrees/fix-bug` | `worktree-fix-bug` |
| `user/feat` | `.claude/worktrees/user+feat` | `worktree-user+feat` |

---

## 10.4 validateWorktreeSlug

```typescript
// src/utils/worktree.ts

export function validateWorktreeSlug(slug: string): void {
  // 每段（以 / 分隔）只允许字母、数字、点、下划线、连字符
  // 支持嵌套结构，如 "user/feat-1"
  // 总长度不超过 64 个字符
  const segmentPattern = /^[a-zA-Z0-9._-]+$/
  const segments = slug.split('/')
  
  if (slug.length > 64) {
    throw new Error('worktree slug 不能超过 64 个字符')
  }
  
  for (const segment of segments) {
    if (!segmentPattern.test(segment)) {
      throw new Error(`slug 段 "${segment}" 只能包含字母、数字、点、下划线和连字符`)
    }
  }
}
```

注意：实际允许的字符集比文档早期版本描述的更宽，允许 `.` 和 `_`，且支持 `/` 分隔的嵌套结构。

---

## 10.5 EnterWorktreeTool

EnterWorktreeTool 的输入参数只有一个可选的 `name`，没有 `branch` 或 `base_branch`：

```typescript
// src/tools/EnterWorktreeTool/EnterWorktreeTool.ts

export const EnterWorktreeTool = buildTool({
  name: 'EnterWorktree',
  
  inputSchema: z.object({
    name: z.string().optional(),  // ← 只有 name，没有 branch/base_branch
  }),
  
  async call(input, context) {
    
    // 1. 防止嵌套进入
    const currentWorktree = getCurrentWorktreeSession()
    if (currentWorktree) {
      return { error: '已在 worktree 中，不能嵌套进入另一个 worktree' }
    }
    
    // 2. 解析到主仓库根目录（即使当前在子目录中）
    const mainRepoRoot = await resolveToMainRepoRoot()
    
    // 3. 确定 slug：优先使用输入的 name，其次使用 getPlanSlug()
    const slug = input.name ?? getPlanSlug()
    
    // 4. 创建 worktree（核心操作）
    const worktreeSession = await createWorktreeForSession(
      getSessionId(),
      slug,
      tmuxSessionName,   // tmux 集成
      { prNumber }       // PR checkout 支持
    )
    
    // 5. 切换工作目录（先切换目录，再清缓存）
    process.chdir(worktreeSession.worktreePath)
    setCwd(worktreeSession.worktreePath)
    setOriginalCwd(worktreeSession.worktreePath)
    
    // 6. 持久化 worktree 状态
    saveWorktreeState(worktreeSession)
    
    // 7. 清除缓存（必须在 setCwd 之后，确保缓存从新目录加载）
    clearSystemPromptSections()
    clearMemoryFileCaches()           // 注意：复数形式
    getPlansDirectory.cache?.clear?.()
    
    return { success: true, worktreePath: worktreeSession.worktreePath }
  }
})
```

**关键细节**：缓存清除发生在 `setCwd` **之后**，不是之前。这样确保后续加载的 CLAUDE.md 和系统提示都来自新的 worktree 目录。

---

## 10.6 createWorktreeForSession 详解

```typescript
// src/utils/worktree.ts

export async function createWorktreeForSession(
  sessionId: string,
  slug: string,
  tmuxSessionName?: string,
  options?: { prNumber?: number }
): Promise<WorktreeSession>
```

函数签名完全不同于早期文档版本——没有 `branch` 和 `baseBranch` 参数，分支名由 slug 自动推导。

### 10.6.1 fast-resume：跳过重复初始化

```typescript
// src/utils/worktree.ts

// 如果 worktree 目录已存在，尝试快速恢复
const existingSha = await readWorktreeHeadSha(worktreePath)
if (existingSha) {
  // 已有 worktree，直接恢复会话，跳过 git fetch/checkout 等耗时操作
  return restoreExistingWorktreeSession(worktreePath, sessionId)
}
```

当用户重新进入已有 worktree 时，不需要重新执行 fetch 和 checkout，直接读取已有的 HEAD SHA 进行恢复。

### 10.6.2 PR checkout 支持

```typescript
// 如果提供了 prNumber，执行 PR checkout
if (options?.prNumber) {
  const prRef = await parsePRReference(options.prNumber)
  // 支持两种形式：
  // - "#123"（数字 ID）
  // - GitHub PR URL
  await gitFetchAndCheckoutPR(worktreePath, prRef)
}
```

### 10.6.3 performPostCreationSetup：创建后的初始化

```typescript
// src/utils/worktree.ts

async function performPostCreationSetup(
  worktreePath: string,
  mainRepoPath: string,
): Promise<void> {
  
  // 1. 复制 settings.local.json（用户个性化设置带入新环境）
  await copyLocalSettings(mainRepoPath, worktreePath)
  
  // 2. 配置 core.hooksPath 指向主仓库的 hooks
  //    确保 git hooks 在 worktree 中正常触发
  await exec(`git -C ${worktreePath} config core.hooksPath ${mainRepoPath}/.git/hooks`)
  
  // 3. 创建目录符号链接（避免重复存储大目录）
  //    例如 node_modules -> 主仓库的 node_modules
  await symlinkSharedDirectories(mainRepoPath, worktreePath)
  
  // 4. 处理 .worktreeinclude 文件
  //    类似 .gitignore，指定哪些文件需要复制进 worktree
  await processWorktreeInclude(mainRepoPath, worktreePath)
}
```

`performPostCreationSetup` 是 worktree 创建后的关键初始化步骤，让 worktree 拥有完整的工作环境。

### 10.6.4 sparse-checkout 支持

```typescript
// settings.worktree.sparsePaths 配置项
// 如果设置了 sparsePaths，只检出指定目录
if (settings.worktree?.sparsePaths?.length) {
  await configureSparseCheckout(worktreePath, settings.worktree.sparsePaths)
  session.usedSparsePaths = true
}
```

对于大型 monorepo，可以通过 `sparsePaths` 只检出所需子目录，大幅减少磁盘占用和初始化时间。

---

## 10.7 WorktreeSession 数据结构

```typescript
// src/utils/worktree.ts

interface WorktreeSession {
  originalCwd: string           // 进入 worktree 前的原始目录
  worktreePath: string          // worktree 目录路径
  worktreeName: string          // worktree 名称（= slug）
  worktreeBranch: string        // 检出的分支名（= "worktree-{flatSlug}"）
  originalBranch: string        // 进入前的原始分支名
  originalHeadCommit: string    // 进入前的原始 HEAD commit SHA
  sessionId: string             // Claude 会话 ID
  tmuxSessionName?: string      // 关联的 tmux session（如有）
  hookBased?: boolean           // 是否由 WorktreeCreate hook 创建
  creationDurationMs?: number   // 创建耗时（毫秒）
  usedSparsePaths?: boolean     // 是否启用了 sparse-checkout
}
```

字段比早期文档多得多，其中 `originalHeadCommit` 对于 ExitWorktreeTool 的安全检查至关重要。

---

## 10.8 状态持久化

```typescript
// src/utils/worktree.ts（非早期版本所说的 sessionStorage.ts）

// 保存 worktree 状态到项目配置中
saveCurrentProjectConfig({ activeWorktreeSession: worktreeSession })

// 读取
const activeSession = getCurrentProjectConfig().activeWorktreeSession
```

状态保存在**项目配置**（project config）中，不是独立的 `sessions/{id}/worktree.json` 文件。

模块级变量追踪当前会话：

```typescript
// src/utils/worktree.ts
// 模块级变量（不是 bootstrap/state.ts 中的 globalState）
let currentWorktreeSession: WorktreeSession | null = null

export function getCurrentWorktreeSession(): WorktreeSession | null {
  return currentWorktreeSession
}
```

---

## 10.9 ExitWorktreeTool

ExitWorktreeTool 有两个参数，并包含安全检查：

```typescript
// src/tools/ExitWorktreeTool/ExitWorktreeTool.ts

export const ExitWorktreeTool = buildTool({
  name: 'ExitWorktree',
  
  inputSchema: z.object({
    action: z.enum(['keep', 'remove']),   // 必填：退出后保留还是删除 worktree
    discard_changes: z.boolean().optional(), // 是否强制丢弃未提交的变更
  }),
  
  async call(input, context) {
    const worktreeState = getCurrentWorktreeSession()
    if (!worktreeState) {
      return { error: '当前不在 worktree 中' }
    }
    
    if (input.action === 'remove') {
      // 安全检查：统计未提交的变更
      const changes = await countWorktreeChanges(
        worktreeState.worktreePath,
        worktreeState.originalHeadCommit,  // 用于对比的基准 commit
      )
      
      if (changes > 0 && !input.discard_changes) {
        // 有未提交变更且未明确确认丢弃 → 拒绝删除
        return {
          error: `worktree 有 ${changes} 处未提交变更。` +
                 `如需强制删除，请设置 discard_changes: true`
        }
      }
      
      // 清理 tmux session（如有）
      if (worktreeState.tmuxSessionName) {
        await killTmuxSession(worktreeState.tmuxSessionName)
      }
      
      await cleanupWorktree(worktreeState.worktreePath)
      
    } else {  // action === 'keep'
      await keepWorktree(worktreeState.worktreePath)
    }
    
    // 恢复到原始目录（keep/remove 都需要）
    await restoreSessionToOriginalCwd(worktreeState)
    
    return { success: true }
  }
})
```

### 10.9.1 restoreSessionToOriginalCwd 完整流程

```typescript
async function restoreSessionToOriginalCwd(state: WorktreeSession): Promise<void> {
  // 1. 切换回原始目录
  setCwd(state.originalCwd)
  setOriginalCwd(state.originalCwd)
  
  // 2. 必要时更新项目根目录和 hooks 快照
  if (shouldUpdateProjectRoot) {
    setProjectRoot(state.originalCwd)
    await updateHooksConfigSnapshot()
  }
  
  // 3. 清除 worktree 状态（写入 null）
  saveWorktreeState(null)
  
  // 4. 清除所有缓存（顺序同 EnterWorktreeTool）
  clearSystemPromptSections()
  clearMemoryFileCaches()
  getPlansDirectory.cache?.clear?.()
}
```

---

## 10.10 keepWorktree vs cleanupWorktree

两个函数代表两种完全不同的退出策略：

```typescript
// keepWorktree：保留 worktree 目录和分支，以便后续继续使用
async function keepWorktree(worktreePath: string): Promise<void> {
  // 只执行必要的清理（如关闭文件句柄），不删除任何文件
  // worktree 目录和 git 分支完整保留
}

// cleanupWorktree：彻底删除 worktree
async function cleanupWorktree(worktreePath: string): Promise<void> {
  // 1. git worktree remove --force {path}
  // 2. 删除对应的 git 分支 worktree-{slug}
  // 3. 清理 .claude/worktrees/{slug} 目录
}
```

| 场景 | 推荐 action |
|------|------------|
| 任务完成，已合并 PR | `remove` |
| 任务暂停，稍后继续 | `keep` |
| 需要切换主任务，稍后回来 | `keep` |
| 探索性任务，不需要保留 | `remove` |

---

## 10.11 createAgentWorktree vs createWorktreeForSession

两个函数用于不同场景，有本质区别：

```typescript
// createWorktreeForSession：用于用户会话（EnterWorktreeTool 调用）
// - 使用 getCwd() 确定基准目录
// - 修改全局 currentWorktreeSession 状态
// - 状态持久化到项目配置

// createAgentWorktree：用于子代理（Agent 工具调用）
export async function createAgentWorktree(
  agentId: string,
  slug: string,
): Promise<string> {
  
  // 关键区别 1：使用 findCanonicalGitRoot() 而不是 getCwd()
  // 确保总是从主仓库根目录创建，即使当前在子目录
  const gitRoot = await findCanonicalGitRoot()
  
  // 关键区别 2：不修改全局 currentWorktreeSession
  // 子代理的 worktree 是独立的，不影响主会话状态
  const worktreePath = await doCreateWorktree(gitRoot, slug)
  
  return worktreePath
}
```

**关键区别**：`createAgentWorktree` 不触碰全局 worktree 状态，子代理的 worktree 生命周期完全独立。

---

## 10.12 EPHEMERAL_WORKTREE_PATTERNS：临时 Worktree 识别

```typescript
// src/utils/worktree.ts

// 识别临时（可自动清理）worktree 的正则模式
export const EPHEMERAL_WORKTREE_PATTERNS = [
  /^agent-/,    // 子代理创建的 worktree（createAgentWorktree）
  /^wf_/,       // workflow 任务 worktree
  /^bridge-/,   // bridge 工具 worktree
  /^job-/,      // 批处理 job worktree
]

export function isEphemeralWorktree(slug: string): boolean {
  return EPHEMERAL_WORKTREE_PATTERNS.some(pattern => pattern.test(slug))
}
```

用于 `cleanupStaleAgentWorktrees()` 中的自动清理判断。

---

## 10.13 自动清理：cleanupStaleAgentWorktrees

```typescript
// src/utils/worktree.ts

// 清理过期的临时 worktree（30 天未活跃）
export async function cleanupStaleAgentWorktrees(): Promise<void> {
  const allWorktrees = await listWorktrees(gitRoot)
  const now = Date.now()
  const STALE_THRESHOLD_MS = 30 * 24 * 60 * 60 * 1000  // 30 天
  
  for (const wt of allWorktrees) {
    if (!isEphemeralWorktree(wt.slug)) continue
    
    const lastActive = await getWorktreeLastActiveTime(wt.path)
    if (now - lastActive > STALE_THRESHOLD_MS) {
      await cleanupWorktree(wt.path)
    }
  }
}
```

只有匹配 `EPHEMERAL_WORKTREE_PATTERNS` 的 worktree 才会被自动清理，用户手动创建的 worktree（通过 EnterWorktreeTool）默认不会被自动删除。

---

## 10.14 Worktree 与子代理的隔离

当子代理在 worktree 中运行时，通过 `AsyncLocalStorage` 维护自己的上下文：

```
主代理（main 分支，cwd: /project）
        │
        ├─ 进入 worktree（cwd: /project/.claude/worktrees/fix-bug）
        │
        └─ 创建子代理
              │
              ├─ 子代理的 cwd 继承自父代理的当前 cwd
              │  （即 /project/.claude/worktrees/fix-bug）
              │
              └─ 子代理在 worktree 中操作文件
                 不会影响主代理的文件状态缓存
```

**文件状态缓存克隆**（来自 [src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts)）：

```typescript
// src/utils/swarm/inProcessRunner.ts
const fileStateCache = cloneFileStateCache()
// 子代理获得父代理文件缓存的深拷贝
// 子代理读取/修改文件不会影响父代理的缓存
```

---

## 10.15 WorktreeCreate Hook：完全自定义

```typescript
// WorktreeCreate 是特殊的 hook：
// 如果配置了此 hook，Claude Code 会把 worktree 创建完全委托给 hook
// hook 负责返回 worktreePath

// 配置示例（settings.json）：
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

hook 返回的 `worktreePath` 会被记录在 `WorktreeSession.hookBased = true`。

**适用场景**：
- 在远程服务器上创建 worktree（本地 + 远程协作）
- 与 CI/CD 系统集成，自动触发 pipeline
- 自定义分支策略和命名规则

---

## 10.16 隔离级别对比

| 隔离机制 | 隔离级别 | 使用场景 |
|---------|---------|---------|
| **Worktree + 目录隔离** | 文件系统级 | 不同分支的并行工作 |
| **AsyncLocalStorage** | 进程内 async 上下文 | 子代理上下文（agentId、todo 列表）|
| **文件缓存克隆** | 进程内内存级 | 子代理文件读取状态 |
| **AbortController** | 异步控制流 | 每个子代理独立的取消信号 |
| **sparse-checkout** | Git 对象层 | 大型 monorepo 的目录级隔离 |

---

## 10.17 完整流程图

```
用户（通过 Claude）调用 EnterWorktree
            │
            ▼
  检查当前是否已在 worktree？
       │           │
      YES           NO
       │             │
    返回错误    validateWorktreeSlug()
                       │
                       ▼
              resolveToMainRepoRoot()
                       │
                       ▼
              slug = input.name ?? getPlanSlug()
                       │
                       ▼
            worktree 目录是否已存在？
               │               │
              YES               NO
               │                │
        fast-resume         有 WorktreeCreate Hook？
        （跳过 fetch）          │           │
               │              YES           NO
               │               │            │
               │          执行 Hook     git -B worktree-{slug}
               │          hookBased=true    |  + fetch origin
               │               └─────┬──────┘
               │                      │
               └──────────────────────┘
                               │
                               ▼
                   performPostCreationSetup()
                   （复制 settings, hooksPath, symlinks）
                               │
                               ▼
                   process.chdir(worktreePath)
                   setCwd / setOriginalCwd
                               │
                               ▼
                   saveCurrentProjectConfig({activeWorktreeSession})
                               │
                               ▼
                   clearSystemPromptSections()   ← 在 setCwd 之后
                   clearMemoryFileCaches()
                   getPlansDirectory.cache.clear()
                               │
                               ▼
                   Claude 在 worktree 中工作
                               │
                               ▼
                   用户调用 ExitWorktree(action, discard_changes?)
                               │
                ┌────────────────┴────────────────┐
          action=remove                      action=keep
                │                                 │
    countWorktreeChanges()               keepWorktree()
       有变更且未确认？                              │
        │       │                                 │
       YES      NO                                │
        │       │                                 │
     返回错误  cleanupWorktree()                   │
                │                                 │
                └──────────────┬──────────────────┘
                                  │
                            restoreSessionToOriginalCwd()
                            （setCwd 恢复 + 清缓存 + 清状态）
```

---

## 小结

| 机制 | 作用 | 源码位置 |
|------|------|----------|
| `EnterWorktreeTool` | 创建 worktree 并切换 cwd（仅 `name?` 参数）| [src/tools/EnterWorktreeTool/](../../src/tools/EnterWorktreeTool/) |
| `ExitWorktreeTool` | 退出 worktree（`action: keep\|remove`，含安全检查）| [src/tools/ExitWorktreeTool/](../../src/tools/ExitWorktreeTool/) |
| `createWorktreeForSession(sessionId, slug, tmux?, opts?)` | 执行 worktree 创建主流程 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `createAgentWorktree(agentId, slug)` | 子代理专用，不修改全局状态 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `validateWorktreeSlug()` | 防止路径注入，允许 `[a-zA-Z0-9._-]` + `/` | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `flattenSlug()` | `/` → `+`，用于路径和分支名构造 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `performPostCreationSetup()` | settings 复制、hooksPath 配置、symlink 创建 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `countWorktreeChanges()` | remove 前的安全检查 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `keepWorktree / cleanupWorktree` | 两种退出策略 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `EPHEMERAL_WORKTREE_PATTERNS` | 4 个正则，识别临时 worktree | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `cleanupStaleAgentWorktrees()` | 30 天自动清理临时 worktree | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `saveCurrentProjectConfig` | worktree 状态持久化 | [src/utils/worktree.ts](../../src/utils/worktree.ts) |
| `cloneFileStateCache()` | 子代理文件缓存隔离 | [src/utils/swarm/inProcessRunner.ts](../../src/utils/swarm/inProcessRunner.ts) |
