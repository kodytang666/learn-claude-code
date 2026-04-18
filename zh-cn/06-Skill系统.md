# 第 6 章：Skill系统 — Deferred 加载与按需搜索

> 源码位置：`src/skills/loadSkillsDir.ts`、`src/tools/SkillTool/`、`src/services/skillSearch/`

---

## 6.1 什么是 Skill？

Skill 是 Claude Code 的**可扩展行为单元**。它本质上是一个 Markdown 文件，包含给 Claude 的指令。当 Claude 调用 Skill 工具时，对应的 Markdown 文件内容会被注入到上下文中，Claude 按照其中的指令行事。

**类比**：
- 工具（Tool）= 一个具体的函数（Bash、FileRead、...）
- Skill = 一本操作手册（"如何系统化调试"、"如何写测试"、...）

---

## 6.2 Skill 文件格式

```markdown
---
name: systematic-debugging
description: 系统化调试流程
whenToUse: 遇到 bug 或测试失败时使用
effort: high
---

# 系统化调试

## 第一步：重现问题
首先确认问题可以稳定重现...

## 第二步：理解根因
不要直接跳到修复，先理解为什么...

## 第三步：最小化复现
找到最小的代码片段能复现问题...
```

Frontmatter（YAML 头部）包含元数据：
- `name`：Skill标识符
- `description`：简短描述（用于Skill搜索）
- `whenToUse`：何时使用（Claude 用这个决定是否调用）
- `effort`：工作量等级（low/medium/high）

**为什么 Frontmatter 很重要：**
[src/skills/loadSkillsDir.ts:212](../../src/skills/loadSkillsDir.ts#L212)
Frontmatter 中的 name、description、whenToUse 被用来构建 **skill_listing**，注入到对话上下文，告知 LLM 当前有哪些 skill 可用及其适用场景。Frontmatter 的描述越准确，LLM 越能在合适的时机调用正确的 skill。

**小技巧：** whenToUse + description 加起来长度最好不要超过 250 个字符。原因会在后面说明。

---

## 6.3 Skill目录扫描
[src/skills/loadSkillsDir.ts:67](../../src/skills/loadSkillsDir.ts#L67)

```typescript
export type LoadedFrom =
  | 'commands_DEPRECATED'   // 旧格式兼容
  | 'skills'                // 标准Skill目录
  | 'plugin'                // 插件来源
  | 'managed'               // 管理员配置
  | 'bundled'               // 内置Skill
  | 'mcp'                   // MCP 服务器提供的Skill
```

扫描的目录（按加载阶段，**不代表遮蔽优先级**，见 6.3.1）：

```
目录扫描来源：
- policySettings 指定的目录    ← 企业策略Skill
- ~/.claude/skills/            ← 用户全局Skill
- /project/.claude/skills/     ← 项目级Skill（含父目录逐级往上）
- --add-dir 额外目录
- bundled（内置Skill）           ← Claude Code 自带
- plugin 目录                  ← 插件提供的Skill
- mcp 服务器注册的Skill          ← MCP 动态提供
```

### 加载流程

[src/skills/loadSkillsDir.ts:78](../../src/skills/loadSkillsDir.ts#L78)
```typescript
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string { ... }

// 主加载函数：
async function loadSkillsDir(dir: string, source: LoadedFrom): Promise<Command[]> {
  // 1. 扫描目录，找到所有 .md 文件
  const files = await loadMarkdownFilesForSubdir(dir, 'skills')
  
  // 2. 解析每个文件的 frontmatter
  for (const file of files) {
    const { frontmatter, body } = parseFrontmatter(file.content)
    
    // 3. 提取元数据
    const name = frontmatter.name ?? basename(file.path, '.md')
    const description = coerceDescriptionToString(frontmatter.description)
    const whenToUse = frontmatter.whenToUse
    const effort = parseEffortValue(frontmatter.effort)
    
    // 4. 估算 token 数（仅 frontmatter，不加载完整内容）
    const tokenEstimate = roughTokenCountEstimation(
      `${name}\n${description}\n${whenToUse}`
    )
    
    // 5. 解析 hooks（Skill 可以带 hook 配置）
    const hooks = parseShellFrontmatter(frontmatter.hooks)
    
    // 6. 注册为 Command 对象
    skills.push({
      type: 'prompt',
      name,
      description,
      whenToUse,
      effort,
      tokenEstimate,
      filePath: file.path,
      source,
      // 注意：body 内容不在此处加载！
    })
  }
  
  // 7. 去重处理（符号链接和重复路径）
  return deduplicateSkills(skills)
}
```

**关键设计**：加载阶段只读取 frontmatter，不加载 Skill 正文。正文内容在 Claude 实际调用 Skill 时才读取。

### 6.3.1 同名 Skill 的遮蔽顺序

遮蔽顺序由**两段合并逻辑**共同决定，均采用 first-wins（先出现者保留）。

#### 第一段：`loadAllCommands()` 决定各来源的相对顺序

[src/commands.ts:460-468](../../src/commands.ts#L460-L468)
```typescript
return [
  ...bundledSkills,        // 1. 内置Skill（最先入列，Array.find() 最先匹配 → 优先级最高）
  ...builtinPluginSkills,  // 2. 内置插件Skill
  ...skillDirCommands,     // 3. 目录扫描Skill（用户/项目 skill 在这里，细分见下）
  ...workflowCommands,     // 4. 工作流命令
  ...pluginCommands,       // 5. 插件命令
  ...pluginSkills,         // 6. 插件Skill
  ...COMMANDS(),           // 7. 内置斜杠命令（最后入列 → 优先级最低）
]
```

#### 第二段：`skillDirCommands` 内部的顺序（目录扫描结果）

[src/skills/loadSkillsDir.ts:717-723](../../src/skills/loadSkillsDir.ts#L717-L723)
```typescript
const allSkillsWithPaths = [
  ...managedSkills,          // policySettings（企业策略，最先 → 优先级最高）
  ...userSkills,             // userSettings（~/.claude/skills/）
  ...projectSkillsNested,    // projectSettings（/project/.claude/skills/，含父目录）
  ...additionalSkillsNested, // --add-dir 额外目录
  ...legacyCommands,         // 旧版 /commands/ 目录（最后 → 优先级最低）
]
// first-wins 去重（仅针对相同物理文件，用 realpath 判断）
```

**Skills 与 Commands 的关系：**  在代码层面，Skill commands 共用同一个 Command 类型（均为 type: 'prompt'），都能被 LLM 通过 SkillTool 调用，也都能由用户手动调用。`.claude/commands/` 目录标记为 `commands_DEPRECATED` 且优先级最低，作者意图是让用户用 `.claude/skills/` 替代 `.claude/commands/`。但是代码中仍保留了大量的 `commands` 命名，这应该属于历史包袱，无需区分理解。

#### 第三段：MCP 追加在最后

[src/tools/SkillTool/SkillTool.ts:92-93](../../src/tools/SkillTool/SkillTool.ts#L92-L93)
```typescript
return uniqBy([...localCommands, ...mcpSkills], 'name')
// MCP Skill追加在所有本地Skill之后，优先级最低
```

#### 最终遮蔽优先级（从高到低）

| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1（最高）| `bundledSkills` | Claude Code 编译内置Skill |
| 2 | `builtinPluginSkills` | 内置插件Skill |
| 3 | `policySettings`（managedSkills） | 企业管理员策略目录 |
| 4 | `userSettings` | `~/.claude/skills/` |
| 5 | `projectSettings` | `/project/.claude/skills/`（含父目录） |
| 6 | `--add-dir` 额外目录 | |
| 7 | `pluginCommands` / `pluginSkills` | 插件提供的Skill |
| 8（最低）| `mcp` | MCP 服务器动态注册的Skill |

**路径去重**：`skillDirCommands` 内部的去重只针对**同一物理文件**（通过 `realpath` 解析符号链接后比较），不处理同名但不同文件的情况。两个目录各有一个不同的 `writing-article.md` 时，两个都会进入 `commands[]` 数组。

**同名遮蔽**：`allCommands` 按优先级排序，在遍历生成 `skill_listing` 时，高优先级的 skill name 先被加入 Set，后面同名的 skill 会被 filter 掉。

### 6.3.2 加载缓存与缓存失效

**Skill列表不会在每次对话或每轮工具调用时重新扫描磁盘**。整个进程生命周期内，扫描结果通过两层 `memoize` 缓存在内存中：

[src/skills/loadSkillsDir.ts:638](../../src/skills/loadSkillsDir.ts#L638)
```typescript
// key：cwd
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => { /* 扫描磁盘 */ }
)

// [src/commands.ts:449](../../src/commands.ts#L449)
// key：cwd，合并所有来源
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

同一工作目录下，无论发起多少次对话、多少轮工具调用，`getCommands()` 都直接从内存返回，不会重复扫描。

#### 缓存失效的三个触发点

| 触发条件 | 清除函数 | 源码位置 |
|---------|---------|---------|
| skill 目录文件变化（chokidar 文件监听） | `clearCommandsCache()` | [src/utils/skills/skillChangeDetector.ts:274](../../src/utils/skills/skillChangeDetector.ts#L274) |
| 插件安装 / 卸载 | `clearCommandsCache()` | [src/utils/plugins/cacheUtils.ts:46](../../src/utils/plugins/cacheUtils.ts#L46) |
| 用户执行 `/clear caches` | `clearCommandsCache()` | [src/commands/clear/caches.ts:60](../../src/commands/clear/caches.ts#L60) |

`clearCommandsCache()` 会依次清除所有相关缓存层：

[src/cli/print.ts:1824](../../src/cli/print.ts#L1824)
```typescript
const unsubscribeSkillChanges = skillChangeDetector.subscribe(() => {
  clearCommandsCache()
  void getCommands(cwd()).then(newCommands => {
    currentCommands = newCommands   // 立即更新
  })
})
```

#### 文件监听的实现细节

文件变化监听基于 **chokidar**，监控所有 skill 目录（`src/utils/skills/skillChangeDetector.ts`）。有两个关键的时间常数防止触发过于频繁：

```typescript
const FILE_STABILITY_THRESHOLD_MS = 1000  // 等待文件写入稳定
const RELOAD_DEBOUNCE_MS = 300            // 防抖：300ms 内多次变化合并为一次
```

在 Bun 运行时下，chokidar 改用 **stat 轮询**（而非原生 `fs.watch`），轮询间隔 2 秒：

[src/utils/skills/skillChangeDetector.ts:62](../../src/utils/skills/skillChangeDetector.ts#L62)
```typescript
const USE_POLLING = typeof Bun !== 'undefined'
const POLLING_INTERVAL_MS = 2000
```

原因是 Bun 的 `FSWatcher` 存在已知死锁 bug（`oven-sh/bun#27469`）：在主线程关闭 watcher 时，文件监听线程正在投递事件，两者相互等待导致挂死。

#### 完整生命周期

```
进程启动
  │
  ▼
第一次 getCommands(cwd)
  └─ 扫描磁盘，结果写入 memoize 缓存
  │
  ▼
后续所有调用（同 cwd）
  └─ 直接读内存，不碰磁盘
  │
  ▼
skill 文件变化（或插件变更 / /clear caches）
  └─ clearCommandsCache() 清除全部缓存层
  │
  ▼
下一次 getCommands(cwd)
  └─ 重新扫描磁盘，写入新缓存
```

---

## 6.4 LLM 如何知道有哪些 Skill 可用

### 6.4.1 SkillTool 只告诉 Claude「如何调用」

SkillTool 的 inputSchema（随每次 API 请求一起发送）是：

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

`skill` 参数是**自由文本字符串**，没有 enum，没有列举可用的 skill 名称。Claude 从 SkillTool schema 里只能知道「有一个 Skill 工具，需要传一个 skill name」，但**不知道当前有哪些 skill**。

### 6.4.2 skill_listing AttachmentMessage：提供发现能力

Skill 列表通过 `skill_listing` 类型的 `AttachmentMessage` 注入到对话上下文中：

```
system-reminder: The following skills are available for use with the Skill tool:

- commit: Create a git commit - Use when the user asks to commit changes.
- review-pr: Review a pull request - Use when user asks to review a PR.
- systematic-debugging: 系统化调试 - 遇到 bug 或测试失败时使用
```

每条 skill 只包含 `name : description + whenToUse`（frontmatter 字段），**不包含** skill 正文。

关于 `AttachmentMessage` 的介绍详见章节 2

**为什么 description + whenToUse 长度不要超过 250 个字符：**

[src/tools/SkillTool/prompt.ts:43-65](../../src/tools/SkillTool/prompt.ts#L43-L65)
```typescript
// verbose whenToUse strings waste turn-1 cache_creation tokens without improving match rate. 
function getCommandDescription(cmd: Command): string {
  const desc = cmd.whenToUse
    ? `${cmd.description} - ${cmd.whenToUse}`
    : cmd.description
  // 截断到 MAX_LISTING_DESC_CHARS = 250 字符
  return desc.length > MAX_LISTING_DESC_CHARS
    ? desc.slice(0, MAX_LISTING_DESC_CHARS - 1) + '…'
    : desc
}
```

### 6.4.3 skill_listing 的注入时机与位置

**注入时机**：每轮对话开始处理用户输入时，`getAttachmentMessages()` 检查 `sentSkillNames`（模块级 Map），如果 `sentSkillNames` 存在尚未宣告过的 skill，则生成新的 skill_listing attachment。

[src/utils/attachments.ts:875](../../src/utils/attachments.ts#L875)
```typescript
maybe('skill_listing', () => getSkillListingAttachments(context))
```

**位置**：`skill_listing AttachmentMessage` 追加在用户消息之后：

[src/utils/processUserInput/processTextPrompt.ts:96-99](../../src/utils/processUserInput/processTextPrompt.ts#L96-L99)
```typescript
return {
  messages: [userMessage, ...attachmentMessages],
  //          ^用户消息    ^skill_listing 紧跟其后
  shouldQuery: true,
}
```

### 6.4.4 发送前的格式转换

每次调用 API 前 `normalizeMessagesForAPI()` 会将 `AttachmentMessage` 转换为 `<system-reminder>`：

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

即：每次向 API 发请求时，`messages[]` 里的 `AttachmentMessage(skill_listing)` 自动转换为 `<system-reminder>` 包裹的 user message。

**第一轮后 messages[] 结构**：

```
[
  { role: "user", content: "帮我写函数" },
  { role: "user", content: "<system-reminder>\n 
      The following skills are available...
    </system-reminder>" },
  { role: "assistant", content: "好的，我帮你生成函数..." },
]
```

### 6.4.5 delta 追踪：Messages 防止重复注入 skill_listing

[src/utils/attachments.ts:2607](../../src/utils/attachments.ts#L2607)
```typescript
const sentSkillNames = new Map<string, Set<string>>()
// key: agentId（undefined 表示主线程）
// value: 已宣告的 skill name 集合
```

`getSkillListingAttachments()` 每轮检查 `sentSkillNames`，只把**不在 Map 里的 skill** 当作新增注入一条 `AttachmentMessage`。历史中已有的 `AttachmentMessage` 不会重复 push，`normalizeMessagesForAPI` 每轮自动渲染它们。

触发重新注入的条件：
- skill 文件变化 → `resetSentSkillNames()` 清空 Map → 所有 skill 对 Map 来说都变成"新的" → 下一轮注入当前全部 skill（本质仍是 delta 逻辑，只是 Map 清空后 delta = 全量）
- `--resume` 恢复会话 → `suppressNextSkillListing()` 。获取历史记录中已有的 skill_listing，避免重复

### 6.4.6 Compact 后的行为

Auto Compact 触发时，`compact.ts:213-220` 明确过滤掉 `skill_listing`：

```typescript
// skill_listing 被从 messages[] 中删除，不进入 compact 摘要
return messages.filter(
  m => !(m.type === 'attachment' && m.attachment.type === 'skill_listing')
)
```

压缩后 `sentSkillNames` 不会重新注入。已调用过的 skill 内容由 `invoked_skills` attachment 保留，LLM 可以调用它们。但对于从未调用过的 skill，compact 后 LLM 会忘掉它们的存在。理由：

> skill_listing (~4K tokens) post-compact is pure cache_creation with marginal benefit. The model still has SkillTool in its schema and invoked_skills attachment (below) preserves used-skill content. 

思考：LLM 会在何时重新生成 skill_listing ？

---

## 6.5 Skill的延迟加载（Deferred Loading）

Skill系统有两层"懒"：

### 第一层：Skill内容在扫描时读入，但延迟注入上下文

扫描阶段，`loadSkillsDir()` 通过 `fs.readFile` 读取完整的 SKILL.md 内容，存入 `createSkillCommand()` 返回的 Command 对象的 `getPromptForCommand` 闭包里：

[src/skills/loadSkillsDir.ts:295-343](../../src/skills/loadSkillsDir.ts#L295-L343)（createSkillCommand）
```typescript
const markdownContent = await fs.readFile(filePath, 'utf-8')  // 扫描时就读了！

return {
  ...
  getPromptForCommand: async () => markdownContent,  // 闭包持有内容
}
```

"延迟"指的是：**内容不会在扫描时注入对话上下文**，只有 Claude 主动调用 `Skill("xxx")` 时，`SkillTool.call()` 才通过 `getPromptForCommand()` 取出内容，作为 tool result 注入。

### 第二层：skill_listing 中只有摘要，正文按需注入

Claude 在 `skill_listing` 里只看到 name + description + whenToUse。完整指令只有调用后才出现在上下文里：

```
LLM 看到的 skill_listing（摘要）：
- systematic-debugging: 系统化调试 - 遇到 bug 时使用

LLM 调用 Skill("systematic-debugging") 后的 tool result（完整内容）：
# 系统化调试
## 第一步：重现问题
首先确认问题可以稳定重现...
（完整 SKILL.md 正文）
```

---

## 6.6 Skill搜索预加载（Skill Prefetch）

> **⚠️ 状态：待实现。** 

该机制仅在 `EXPERIMENTAL_SKILL_SEARCH` feature flag 开启时生效（[src/query.ts:66-68](../../src/query.ts#L66-L68)）：

**开启后的行为变化**：

1. **`skill_listing` 收窄**：不再列出用户自定义 skill，仅保留 内置 Skill + MCP
2. **按需异步搜索**：在 Agent Loop 中，LLM 流式输出期间，Claude Code 后台对用户消息做语义搜索，从全量 skill 库中匹配相关 skill
3. **注入 `skill_discovery`**：搜索结果作为独立的 `skill_discovery` AttachmentMessage 追加到 `messages[]`，同样转换为 `<system-reminder>` 发给 LLM

**设计意图**（来自代码注释）：
> "Discovery runs while the model streams and tools execute; awaited post-tools alongside the memory prefetch consume. Replaces the blocking assistant_turn path that ran inside getAttachmentMessages (97% of those calls found nothing in prod)."

即：避免阻塞主流程，将搜索隐藏在 LLM 响应时间内。生产环境 97% 的调用找不到新 skill，异步化后不浪费资源。

---

## 6.7 流程图：Skill的完整生命周期

```
Claude Code 启动
      │
      ▼
loadSkillsDir()                        ← 扫描所有Skill目录
  ├─ fs.readFile 读取完整 .md 文件（启动时全量读入内存）
  ├─ 解析 frontmatter（name/description/whenToUse）
  ├─ 去重
  └─ createSkillCommand()            ← 注册为 Command 对象（memoize 缓存）
       └─ markdownContent 捕获进 getPromptForCommand 闭包
          （调用时直接从闭包取，不再读磁盘）
      │
      ▼
用户输入第一条消息
      │
      ▼
processUserInput()
  └─ getAttachmentMessages(input, ...)
       └─ getSkillListingAttachments()
            ├─ 检查 sentSkillNames（首次为空）
            ├─ formatCommandsWithinBudget()  ← 只取 name+desc+whenToUse
            └─ 返回 skill_listing Attachment
      │
      ▼
processTextPrompt()
  └─ messages: [UserMessage, AttachmentMessage(skill_listing)]
  //  skill_listing 紧跟用户消息之后进入 messages[]
      │
      ▼
每次 API 调用前：normalizeMessagesForAPI()
  └─ AttachmentMessage(skill_listing)
       → wrapMessagesInSystemReminder(...)
       → <system-reminder>The following skills are available...</system-reminder>
  // Claude 每轮都看到 skill 列表（因为 AttachmentMessage 一直在 messages[] 里）
      │
      ▼
Claude 根据 skill_listing 决定调用Skill
  └─ tool_use: Skill({ skill: "systematic-debugging" })
      │
      ▼
SkillTool.call()
  ├─ findSkillByName()
  ├─ getPromptForCommand()           ← 从闭包取出完整内容（不再读磁盘）
  ├─ recordInvokedSkill()            ← 记录调用（compact 后恢复用）
  └─ 返回完整 SKILL.md 内容给 Claude
      │
      ▼
Claude 按Skill指令行事
      │
      ▼（若干轮后可能发生 compact）
      │
Auto Compact 触发
  ├─ skill_listing AttachmentMessage 从 messages[] 中被过滤掉
  ├─ sentSkillNames 不 reset（不重新注入 skill 列表）
  └─ 已调用 skill 的内容由 invoked_skills attachment 保留（截断到 5K/个）
```

---

## 小结

| 机制 | 设计目标 | 源码位置 |
|------|----------|----------|
| 目录扫描 | 多来源聚合（用户/项目/插件/MCP） | `src/skills/loadSkillsDir.ts` |
| `AttachmentMessage(skill_listing)` | 告知 LLM 有哪些 skill 可用（name+desc） | [src/utils/attachments.ts:2661](../../src/utils/attachments.ts#L2661) |
| `sentSkillNames` delta 追踪 | 只注入新增 skill，避免重复 push | [src/utils/attachments.ts:2607](../../src/utils/attachments.ts#L2607) |
| `normalizeMessagesForAPI()` 格式转换 | AttachmentMessage → `<system-reminder>` | [src/utils/messages.ts:3728](../../src/utils/messages.ts#L3728) |
| Compact 过滤 skill_listing | 不让 skill 列表进入摘要（浪费 token） | [src/services/compact/compact.ts:218](../../src/services/compact/compact.ts#L218) |
| 延迟内容注入 | 正文只在 Skill tool 调用时注入上下文 | `src/tools/SkillTool/SkillTool.ts` |
| Prefetch 预加载 | 并行运行，不阻塞主流程 | `src/services/skillSearch/prefetch.ts` |
| Compact 后恢复已调用 skill | 压缩后不丢失Skill知识 | `src/services/compact/compact.ts` |
| 去重 | 符号链接和多来源去重 | `deduplicateSkills()` |
