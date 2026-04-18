# 第 9 章：Hook 机制 — 27 种事件钩子与扩展协议

> 源码位置：[src/types/hooks.ts](../../src/types/hooks.ts)、[src/utils/hooks.ts](../../src/utils/hooks.ts)、[src/utils/hooks/](../../src/utils/hooks/)

---

## 9.1 什么是 Hook？

Hook 是 Claude Code 提供给用户的**扩展点系统**。它允许用户在 Agent 生命周期的关键节点注入自定义逻辑：

- 工具调用前检查（PreToolUse）
- 工具调用后处理（PostToolUse）
- 会话启动时初始化（SessionStart）
- 上下文压缩前后的自定义操作（PreCompact / PostCompact）
- ...共 **27 种事件**

**实现机制**：Hook 本质上是**可执行命令**。Claude Code 在特定事件发生时执行这些命令，传入 JSON 格式的事件数据，读取命令的 stdout/stderr 和 exit code 作为响应。

---

## 9.2 完整的 Hook 事件列表（27 种）

[src/entrypoints/sdk/coreTypes.ts:25-53](../../src/entrypoints/sdk/coreTypes.ts#L25-L53)

```typescript
export const HOOK_EVENTS = [
  // ── 工具相关 ──
  'PreToolUse',          // 工具调用前（最强拦截点）
  'PostToolUse',         // 工具调用后（成功）
  'PostToolUseFailure',  // 工具调用失败后
  'PermissionDenied',    // 权限被 auto 模式分类器拒绝后

  // ── 用户交互 ──
  'UserPromptSubmit',    // 用户提交提示时

  // ── 会话生命周期 ──
  'SessionStart',        // 新会话/resume/clear/compact 时
  'SessionEnd',          // 会话结束时（clear/logout/exit 等）—— 与 Stop 不同！
  'Stop',                // Claude 即将结束本次响应时
  'StopFailure',         // 因 API 错误（限速/认证/计费等）结束时（fire-and-forget）
  'Setup',               // 仓库初始化/维护时（trigger: 'init' | 'maintenance'）

  // ── 子代理相关 ──
  'SubagentStart',       // 子代理启动时
  'SubagentStop',        // 子代理即将结束响应时
  'TeammateIdle',        // 队友即将进入 idle 状态时（可阻止）

  // ── 任务相关 ──
  'TaskCreated',         // 任务被创建时（可阻止）
  'TaskCompleted',       // 任务被标记完成时（可阻止）

  // ── 权限/选择相关 ──
  'PermissionRequest',   // 工具需要权限确认时（可替代 UI 弹窗）
  'Notification',        // Claude 发出通知时

  // ── MCP Elicitation ──
  'Elicitation',         // MCP server 请求用户输入时
  'ElicitationResult',   // 用户回答 MCP elicitation 后

  // ── 配置 ──
  'ConfigChange',        // 配置文件在会话中变化时（可阻止）
  'InstructionsLoaded',  // CLAUDE.md 或规则文件被加载时（仅观测，不可阻止）

  // ── 压缩 ──
  'PreCompact',          // 压缩前（trigger: 'manual' | 'auto'）
  'PostCompact',         // 压缩后

  // ── Worktree ──
  'WorktreeCreate',      // 创建 worktree 时（可完全替换默认实现）
  'WorktreeRemove',      // 移除 worktree 时

  // ── 文件/路径 ──
  'CwdChanged',          // 工作目录变化后
  'FileChanged',         // 监听的文件变化时（需先通过 SessionStart/CwdChanged 注册）
] as const
```

> **Stop vs SessionEnd**：`Stop` 在 Claude 正常结束本次响应时触发（exit code 2 可让 Claude 继续运行）；`SessionEnd` 在整个会话关闭时触发（clear/logout/exit），有严格超时限制（默认 1500ms）。

---

## 9.3 Hook 类型（4 种可配置类型）

[src/utils/hooks/hooksSettings.ts:44-64](../../src/utils/hooks/hooksSettings.ts#L44-L64)

```typescript
type HookCommand =
  | { type: 'command'; command: string; shell?: string; if?: string; timeout?: number }
  | { type: 'prompt';  prompt: string;  if?: string; timeout?: number }   // AI 驱动：让 Claude 处理 hook 逻辑
  | { type: 'agent';   prompt: string;  if?: string; timeout?: number }   // Agent 类型
  | { type: 'http';    url: string;     if?: string; timeout?: number }   // HTTP 请求
  // 'function' 和 'callback' 是 SDK 内部类型，不可在设置文件中配置
```

最常用的是 `command` 类型。`prompt` 类型让一个 Claude 实例来判断是否允许操作（适合复杂的语义判断）。

**`if` 条件字段**：每个 hook 都可设置 `if` 条件（如 `"Bash(git *)"` 匹配器），与 `matcher` 共同构成 hook 身份标识——同一 command 字符串配合不同 `if` 是两个独立 hook。

---

## 9.4 Hook 配置格式

Hook 配置在 `.claude/settings.json`（项目）、`~/.claude/settings.json`（用户）或 `.claude/settings.local.json`（本地）中：

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

**配置来源（3 处）**：
- `userSettings`：`~/.claude/settings.json`
- `projectSettings`：`.claude/settings.json`
- `localSettings`：`.claude/settings.local.json`（不提交到 git，个人覆盖）

另有 `sessionHook`（通过 SessionStart hook 响应动态注册）和内置 `builtinHook`。

---

## 9.5 Exit Code 协议

每种事件对 exit code 的响应不同。这是整个 Hook 系统中最关键的行为规范：

[src/utils/hooks/hooksConfigManager.ts:26-264](../../src/utils/hooks/hooksConfigManager.ts#L26-L264)

| 事件 | Exit 0 | Exit 2 | 其他 |
|------|--------|--------|------|
| `PreToolUse` | stdout/stderr 不展示 | stderr → 模型，**阻止工具调用** | stderr → 用户，继续调用 |
| `PostToolUse` | stdout 显示（transcript 模式 ctrl+O）| stderr → 模型（立即注入）| stderr → 用户 |
| `PostToolUseFailure` | stdout 显示（transcript 模式）| stderr → 模型（立即）| stderr → 用户 |
| `UserPromptSubmit` | stdout → Claude（追加）| **阻止处理 + 清除原始 prompt** + stderr → 用户 | stderr → 用户 |
| `SessionStart` | stdout → Claude；阻塞错误忽略 | — | stderr → 用户 |
| `SessionEnd` | 正常完成 | — | stderr → 用户 |
| `Stop` | stdout/stderr 不展示 | stderr → 模型，**Claude 继续运行** | stderr → 用户 |
| `StopFailure` | — | — | 全忽略（fire-and-forget）|
| `Setup` | stdout → Claude；阻塞错误忽略 | — | stderr → 用户 |
| `SubagentStop` | stdout/stderr 不展示 | stderr → 子代理，**子代理继续运行** | stderr → 用户 |
| `PreCompact` | stdout 追加为自定义压缩指令 | **阻止压缩** | stderr → 用户，继续压缩 |
| `PermissionDenied` | stdout 显示（transcript 模式）| — | stderr → 用户 |
| `ConfigChange` | 允许变更 | **阻止变更应用** | stderr → 用户 |
| `TeammateIdle` | stdout/stderr 不展示 | stderr → 队友，**防止 idle**（队友继续工作）| stderr → 用户 |
| `TaskCreated` | stdout/stderr 不展示 | stderr → 模型，**阻止任务创建** | stderr → 用户 |
| `TaskCompleted` | stdout/stderr 不展示 | stderr → 模型，**阻止任务完成** | stderr → 用户 |
| `WorktreeCreate` | worktree 创建成功（stdout=路径）| — | 创建失败 |
| `WorktreeRemove` | 移除成功 | — | stderr → 用户 |
| `InstructionsLoaded` | 正常完成（**不可阻止**）| — | stderr → 用户 |
| `CwdChanged` / `FileChanged` | 正常完成 | — | stderr → 用户 |

---

## 9.6 Hook 响应协议（JSON 输出）

Hook 命令的 stdout 返回 JSON（可选）：

[src/types/hooks.ts:50-165](../../src/types/hooks.ts#L50-L165)

```typescript
// 同步响应（大多数 hook）
type SyncHookResponse = {
  // ── 通用字段 ──
  continue?: boolean          // false = 停止当前操作（等同于 exit 2 阻止）
  suppressOutput?: boolean    // 隐藏 stdout 不显示给用户
  stopReason?: string         // continue=false 时显示的原因
  decision?: 'approve' | 'block'  // 快捷权限决策（不需要写 hookSpecificOutput）
  reason?: string             // 决策说明
  systemMessage?: string      // 系统警告消息（显示给用户）

  // ── 事件专属输出 ──
  hookSpecificOutput?: PreToolUseOutput | PostToolUseOutput | ...
}

// 异步响应（立即返回，hook 在后台继续运行）
type AsyncHookResponse = {
  async: true
  asyncTimeout?: number  // 异步超时（毫秒）
}
```

### 各事件的专属输出（hookSpecificOutput）

[src/types/hooks.ts:71-163](../../src/types/hooks.ts#L71-L163)

**PreToolUse**（最强大的拦截点）：
```typescript
{
  hookEventName: 'PreToolUse',
  permissionDecision?: 'allow' | 'deny' | 'ask',  // 覆盖权限决策
  permissionDecisionReason?: string,
  updatedInput?: Record<string, unknown>,           // 修改工具输入参数！
  additionalContext?: string,                        // 注入额外上下文
}
```

**PostToolUse**：
```typescript
{
  hookEventName: 'PostToolUse',
  additionalContext?: string,
  updatedMCPToolOutput?: unknown,  // 修改 MCP 工具的输出结果
}
```

**UserPromptSubmit**：
```typescript
{
  hookEventName: 'UserPromptSubmit',
  additionalContext?: string,  // 在提示末尾追加信息
}
```

**SessionStart**（注册文件监听）：
```typescript
{
  hookEventName: 'SessionStart',
  additionalContext?: string,
  initialUserMessage?: string,    // 自动发送的第一条消息（模拟用户输入）
  watchPaths?: string[],          // 注册文件变化监听路径（触发 FileChanged）
}
```

**PermissionDenied**（允许重试）：
```typescript
{
  hookEventName: 'PermissionDenied',
  retry?: boolean,  // true = 告诉模型可以重试这个工具调用
}
```

**PermissionRequest**（编程式权限控制）：
```typescript
{
  hookEventName: 'PermissionRequest',
  decision:
    | { behavior: 'allow'; updatedInput?: ...; updatedPermissions?: ... }
    | { behavior: 'deny'; message?: string; interrupt?: boolean }
}
```

**CwdChanged / FileChanged**（动态更新监听路径）：
```typescript
{
  hookEventName: 'CwdChanged' | 'FileChanged',
  watchPaths?: string[],  // 动态更新 FileChanged 的监听路径列表
}
```

**WorktreeCreate**（返回路径）：
```typescript
{
  hookEventName: 'WorktreeCreate',
  worktreePath: string,  // hook 创建的 worktree 绝对路径
}
```

---

## 9.7 Hook 输入的公共字段

[src/utils/hooks.ts:301-328](../../src/utils/hooks.ts#L301-L328)

每个 hook 收到的 JSON 输入都包含以下 base 字段（`createBaseHookInput()` 生成）：

```typescript
{
  session_id: string,       // 当前会话 ID
  transcript_path: string,  // 会话转录文件路径（可用于读取完整对话历史）
  cwd: string,              // 当前工作目录
  permission_mode?: string, // 权限模式
  agent_id?: string,        // 子代理 ID（main thread 调用时为 undefined）
  agent_type?: string,      // 代理类型
}
```

在 base 字段之上，各事件追加自己的专属字段：

```typescript
// PreToolUse 额外字段
{ tool_name: string; tool_input: unknown; tool_use_id: string }

// PostToolUse 额外字段
{ tool_name: string; tool_input: unknown; tool_use_id: string; tool_response: unknown }

// SessionStart 额外字段
{ source: 'startup' | 'resume' | 'clear' | 'compact' }

// Setup 额外字段
{ trigger: 'init' | 'maintenance' }

// PreCompact / PostCompact 额外字段
{ trigger: 'manual' | 'auto'; ... }

// SubagentStart / SubagentStop 额外字段
{ agent_id: string; agent_type: string }

// SessionEnd 额外字段
{ reason: 'clear' | 'logout' | 'prompt_input_exit' | 'other' }

// StopFailure 额外字段
{ error: 'rate_limit' | 'authentication_failed' | 'billing_error' | ... }

// ConfigChange 额外字段
{ source: 'user_settings' | 'project_settings' | 'local_settings' | ...; file_path: string }

// InstructionsLoaded 额外字段
{ file_path, memory_type, load_reason, globs?, trigger_file_path?, parent_file_path? }

// FileChanged 额外字段
{ file_path: string; event: 'change' | 'add' | 'unlink' }
```

**CLAUDE_ENV_FILE**：`CwdChanged` 和 `FileChanged` hook 执行时，环境变量 `CLAUDE_ENV_FILE` 指向一个临时文件，hook 可以向其写入 `export VAR=val` 形式的 bash 变量，影响后续 BashTool 命令的环境。

---

## 9.8 getAllHooks 与 Hook 来源聚合

[src/utils/hooks/hooksSettings.ts:92-161](../../src/utils/hooks/hooksSettings.ts#L92-L161)

```typescript
export function getAllHooks(appState: AppState): IndividualHookConfig[]
```

**注意**：`getAllHooks` 接受 `appState` 而不是 `HookEvent`。它聚合所有来源的 hook 配置：

```
getAllHooks(appState)
      │
      ├─ policySettings.allowManagedHooksOnly?
      │   YES → 跳过 user/project/local（受管 hook 对 UI 隐藏）
      │   NO  → 继续
      │
      ├─ userSettings hooks
      ├─ projectSettings hooks
      ├─ localSettings hooks
      │   （去重：同一文件路径不重复处理）
      │
      └─ sessionHooks（由 SessionStart hook 响应动态注册）
```

按事件过滤用 `getHooksForEvent(appState, event)`。

---

## 9.9 安全机制：信任检查

[src/utils/hooks.ts:286-296](../../src/utils/hooks.ts#L286-L296)

```typescript
export function shouldSkipHookDueToTrust(): boolean {
  const isInteractive = !getIsNonInteractiveSession()
  if (!isInteractive) return false  // SDK 模式：信任隐式成立，直接执行

  // 交互模式：所有 hook 都必须等待 workspace 信任确认
  const hasTrust = checkHasTrustDialogAccepted()
  return !hasTrust
}
```

**所有 hook 均需要 workspace 信任**。信任确认前，任何 hook 都不会执行——包括 `SessionEnd` 和 `SubagentStop`。这是在发现历史漏洞（用户拒绝信任对话框时 SessionEnd hook 仍执行）后加入的防御性措施。

---

## 9.10 重要 Hook 的详细行为

### PreToolUse：最强大的拦截点

```
触发时机：每次工具调用前
特殊能力：
1. 拒绝工具调用（exit 2 或 continue: false 或 permissionDecision: 'deny'）
2. 修改工具参数（updatedInput）— 工具拿到的是修改后的参数
3. 注入额外上下文（additionalContext）— 注入 Claude 的上下文消息
4. 记录审计日志

典型用例：
- 禁止执行包含 "rm -rf /" 的 Bash 命令
- 在所有文件写入前备份
- 在 git push 前运行 lint
```

### SessionStart：会话初始化

```
触发时机：startup / resume / clear / compact 任意一种
特殊能力：
1. 设置初始用户消息（initialUserMessage）
   → 让 Claude 一启动就执行某个任务（自动发送第一条消息）

2. 注册文件监听路径（watchPaths）
   → 这些路径变化后触发 FileChanged hook
   → 可以实现"文件变化时自动提醒 Claude"

3. 注入额外上下文（additionalContext）
   → 例如注入环境信息、项目状态等
```

### Stop vs SessionEnd

```
Stop：Claude 本次响应结束（正常完成工具调用链）
  └─ exit 2 → stderr 注入模型，Claude 继续运行
  └─ 典型用例：检查任务是否真正完成

SessionEnd：整个会话关闭（/clear, /logout, 关闭 terminal）
  └─ 超时限制：默认 1500ms（可通过 CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS 覆盖）
  └─ 典型用例：保存会话状态、发送统计数据
  └─ 可通过 reason 匹配：clear / logout / prompt_input_exit / other
```

### PermissionDenied + retry

```
触发时机：auto 模式分类器拒绝了工具调用后
特殊能力：
  返回 {"hookSpecificOutput":{"hookEventName":"PermissionDenied","retry":true}}
  → 告诉模型"这个工具调用可以重试"
  → 适合需要额外上下文才能判断是否允许的场景
```

### InstructionsLoaded：只读观测

```
触发时机：CLAUDE.md 或规则文件被加载时
load_reason 枚举：
  session_start, nested_traversal, path_glob_match, include, compact
特别：这个 hook 是纯观测，exit code 无法阻止加载
用途：日志记录、安全审计哪些 CLAUDE.md 被加载了
```

### ConfigChange：运行时配置变更拦截

```
触发时机：会话运行中配置文件变化（热重载）
source 枚举：user_settings, project_settings, local_settings, policy_settings, skills
exit 2 → 阻止这次变更应用到当前会话
用途：防止未授权的配置修改在运行中的会话中生效
```

---

## 9.11 Prompt Elicitation 协议

这是一个特殊机制，允许 Hook 脚本向用户请求额外输入并等待响应：

[src/types/hooks.ts:28-47](../../src/types/hooks.ts#L28-L47)

```typescript
// Hook 脚本 stdout 输出这个 JSON，Claude Code 显示选择 UI
type PromptRequest = {
  prompt: string   // 请求 ID（用于匹配后续响应）
  message: string  // 显示给用户的问题文本
  options: Array<{
    key: string
    label: string
    description?: string
  }>
}

// Claude Code 将用户选择作为 stdin 发送给 hook，或通过 ElicitationResult 事件触发
type PromptResponse = {
  prompt_response: string  // 请求 ID（与 PromptRequest.prompt 对应）
  selected: string         // 用户选择的 key
}
```

这允许 Hook 脚本实现类似"选择部署环境：[staging] [production]"的交互式提示。

---

## 9.12 ALWAYS_EMITTED_HOOK_EVENTS

[src/utils/hooks/hookEvents.ts:18](../../src/utils/hooks/hookEvents.ts#L18)

```typescript
const ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup'] as const
```

SDK 模式下，用户可以通过 `includeHookEvents` 选项控制哪些 hook 事件被发出。但 `SessionStart` 和 `Setup` 始终会发出，不受此选项影响——它们是低噪音的生命周期事件，向后兼容。

---

## 9.13 实际应用案例

### 案例 1：自动化安全检查（PreToolUse + exit 2）

```bash
#!/bin/bash
# pre-bash-check.sh

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# 阻止危险命令
if echo "$COMMAND" | grep -qE "rm -rf /|sudo rm|drop table"; then
  echo "检测到危险命令，已阻止" >&2
  exit 2  # exit 2 → stderr 注入模型，阻止调用
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

### 案例 2：修改工具输入参数（PreToolUse + updatedInput）

```bash
#!/bin/bash
# inject-readonly.sh — 为所有 Bash 命令添加 --dry-run 标志
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')

# 返回修改后的输入
echo "{
  \"hookSpecificOutput\": {
    \"hookEventName\": \"PreToolUse\",
    \"updatedInput\": {
      \"command\": \"$CMD --dry-run\"
    }
  }
}"
```

### 案例 3：SessionStart 注册文件监听 + 自动消息

```bash
#!/bin/bash
# session-init.sh
PROJECT_STATUS=$(cat .project-status.json 2>/dev/null || echo '{}')

echo "{
  \"hookSpecificOutput\": {
    \"hookEventName\": \"SessionStart\",
    \"additionalContext\": \"项目状态：$PROJECT_STATUS\",
    \"watchPaths\": [\"$(pwd)/.project-status.json\"],
    \"initialUserMessage\": \"请先查看项目状态并告知当前进度。\"
  }
}"
```

### 案例 4：Stop hook 验证任务完成

```bash
#!/bin/bash
# check-completion.sh — 确认 Claude 真的完成了任务
if ! [ -f ".task-completed" ]; then
  echo "任务未完成标记，请确认所有步骤已执行" >&2
  exit 2  # exit 2 → 注入模型，Claude 继续运行
fi
exit 0
```

---

## 9.14 流程图：PreToolUse Hook 完整流程

```
Claude 决定调用工具（如 Bash）
            │
            ▼
    canUseTool() 调用
            │
            ▼
    执行 PreToolUse Hooks（按匹配器过滤）
            │
       ┌────┴────┐
  有匹配 hook    无匹配 hook
       │           │
       ▼           ▼
  执行命令     默认权限检查
       │
  parseHookOutput(stdout)
       │
  ┌────┼──────────┬──────────────┐
  │    │          │              │
exit 0  exit 2  continue:false  updatedInput
  │    │          │              │
允许  阻止       阻止           修改输入参数
      返回 stderr  返回 stopReason  用修改后参数调用
      给模型
            │
            ▼
       执行工具 call()
            │
            ▼
       PostToolUse Hooks
```

---

## 小结

| Hook 事件 | 触发时机 | exit 2 效果 | 特殊能力 |
|-----------|----------|------------|----------|
| `PreToolUse` | 工具调用前 | 阻止调用 | 修改参数、覆盖权限 |
| `PostToolUse` | 工具调用后 | stderr→模型立即注入 | 修改 MCP 工具输出 |
| `UserPromptSubmit` | 用户提交时 | 阻止+清除原始 prompt | 追加上下文 |
| `SessionStart` | 新会话时 | — | 自动首条消息、注册文件监听 |
| `SessionEnd` | 会话关闭时 | — | 1500ms 超时，reason 匹配 |
| `Stop` | Claude 结束响应前 | Claude 继续运行 | 任务完成验证 |
| `PreCompact` | 压缩前 | 阻止压缩 | 追加压缩指令 |
| `PermissionDenied` | 权限被拒后 | — | `retry: true` 让模型重试 |
| `PermissionRequest` | 权限弹窗时 | — | 完全替代 UI 权限决策 |
| `TeammateIdle` | 队友将 idle | 阻止 idle | — |
| `TaskCreated/Completed` | 任务变更时 | 阻止变更 | — |
| `ConfigChange` | 配置热重载时 | 阻止变更应用 | source 匹配 |
| `InstructionsLoaded` | CLAUDE.md 加载时 | **无效**（不可阻止）| 纯观测 |
| `WorktreeCreate` | worktree 创建时 | — | 完全替换默认实现 |
| `CwdChanged/FileChanged` | 目录/文件变化 | — | CLAUDE_ENV_FILE 注入环境变量 |

**源码位置总览：**
- Hook 事件完整列表：[src/entrypoints/sdk/coreTypes.ts](../../src/entrypoints/sdk/coreTypes.ts)
- Hook 类型和响应 Schema：[src/types/hooks.ts](../../src/types/hooks.ts)
- Hook 执行主逻辑：[src/utils/hooks.ts](../../src/utils/hooks.ts)
- Hook 配置聚合：[src/utils/hooks/hooksSettings.ts](../../src/utils/hooks/hooksSettings.ts)
- Hook 事件元数据（exit code 行为）：[src/utils/hooks/hooksConfigManager.ts](../../src/utils/hooks/hooksConfigManager.ts)
- Hook 事件广播系统：[src/utils/hooks/hookEvents.ts](../../src/utils/hooks/hookEvents.ts)
