# AgentTool / SkillTool / MCPTool / BashTool —— 四大核心工具对比

> 源码位置：
> - `src/tools/AgentTool/AgentTool.tsx`（AgentTool.tsx + runAgent.ts + loadAgentsDir.ts + builtInAgents.ts + 6 个 built-in agent 定义）
> - `src/tools/SkillTool/SkillTool.ts`（1109 行）
> - `src/tools/MCPTool/MCPTool.ts`（77 行）
> - `src/tools/BashTool/BashTool.tsx`（1184 行，本地文件实际更大）
>
> 关联文档：[`docs/11-tool-execution.md`](11-tool-execution.md) · [`docs/12-run-agent.md`](../task/12-run-agent.md) · [`docs/17-architecture.md`](../architecture/17-architecture.md) · [`docs/23-design-and-core-modules.md`](../architecture/23-design-and-core-modules.md)

这 4 个工具是 Claude Code 的**执行核心**：

- **AgentTool** —— 派生 subagent，实现递归 agent 调用的关键
- **SkillTool** —— 加载领域技能（`.claude/skills/`），模型主动调用领域模板
- **MCPTool** —— 通过 Model Context Protocol 接入外部工具的标准化代理
- **BashTool** —— 最古老、最通用、最强大的命令执行工具

本文按"职责 → Schema → 执行流程 → 异同对比"的顺序逐一拆解。

---

## 目录

- [全景对比表](#全景对比表)
- [AgentTool —— 派生 subagent](#agenttool--派生-subagent)
- [SkillTool —— 加载领域技能](#skilltool--加载领域技能)
- [MCPTool —— MCP 标准化接入](#mcptool--mcp-标准化接入)
- [BashTool —— shell 命令执行](#bashtool--shell-命令执行)
- [四者的协同模式](#四者的协同模式)
- [共性与差异总结](#共性与差异总结)

---

## 全景对比表

| 维度 | AgentTool | SkillTool | MCPTool | BashTool |
|------|-----------|-----------|---------|----------|
| **行数** | 数千行（拆分多个文件） | 1109 | 77 | 1184+ |
| **调用对象** | 另一个 agent | skill 文件 / 远程 skill | 任意 MCP 服务器 | shell 命令 |
| **隔离机制** | 独立 context + 独立 toolUseContext | inline（修改当前 context）或 fork（独立 sub-agent） | MCP 服务器进程 | 共享 cwd + sandbox |
| **典型耗时** | 秒~分钟（多轮） | 即时（inline）或分钟（fork） | 即时（RPC） | 即时~分钟 |
| **可后台化** | ✅ 通过 `run_in_background` | ❌（fork 已是独立任务） | ❌ | ✅ 通过 `run_in_background` |
| **持久化输出** | 写到 `<session>/<agentId>.txt` | inline 注入 messages | MCP 自管理 | 写 `<taskId>.txt` |
| **权限处理** | 子 agent 继承父 context，单独的 canUseTool | allowlist 自动通过；否则 ask | passthrough（由 MCP 自身处理） | 4 层防御 + bash classifier |
| **并发安全** | ❌（独立执行） | ❌ | ✅ | ✅（只读命令） / ❌（写命令） |
| **是否消耗 token 配额** | ✅ 独立计数 | ✅（fork）/ ❌（inline） | ❌（MCP 自管） | ❌ |
| **特殊形态** | 可派生嵌套 subagent | 可 fork 为 sub-agent | 单一占位符，被 mcpClient 替换 | 模拟 sed 编辑 |

---

# AgentTool —— 派生 subagent

## 1. 职责

AgentTool 是 Claude Code 实现**递归 agent 调用**的关键。模型调用 `Agent(prompt, subagent_type: 'Explore')` 时，主 agent 会启动一个**完全独立的子 agent**，拥有自己的：
- 独立的 `messages`（主对话的 fork snapshot）
- 独立的 `toolUseContext`（默认 `setAppState` 是 no-op）
- 独立的 abortController（失败/取消不影响主 agent）
- 独立的任务追踪（注册为 `LocalAgentTask`）

执行完毕后，**只有结构化摘要**回流到主对话，避免子 agent 的内部 tool_use 噪声污染主上下文。

## 2. Schema

```ts
// AgentTool.tsx 主要字段
inputSchema = z.strictObject({
  description: z.string(),         // 简短描述
  prompt: z.string(),              // 详细指令
  subagent_type: z.string(),       // Explore/Plan/general-purpose 等
  model: z.enum(['sonnet','opus','haiku']).optional(),  // 覆盖模型
  run_in_background: z.boolean().optional(),
  isolation: z.enum(['worktree','remote']).optional(),  // ant-only
  // ... 其他字段
})
```

## 3. 执行流程

```
Tool Use: Agent(prompt, subagent_type)
    ↓
AgentTool.call(input, context)
    ↓
1. 解析 AgentDefinition
   - getAgentModel(input.model, agentDef)
   - 检查 isolation（worktree/remote）
    ↓
2. createSubagentContext(parentToolUseContext, options)
   - 设置 setAppState 为 no-op（隔离主 store）
   - 保留 setAppStateForTasks（后台任务可回写）
    ↓
3. buildForkedMessages(parentMessages, agentId)
   - 截取主对话的 fork point
   - 构造子 agent 的初始 messages
    ↓
4. 注册 LocalAgentTask
   - 36 进制 8 位随机 taskId
   - task type = 'local_agent'
   - 通过 setAppStateForTasks 写入主 store（让主 UI 能看到子 agent 状态）
    ↓
5. 调用 query(params) 进入子 agent 的 queryLoop
   - 子 agent 有完整的 5 级压缩、Plan Mode、TodoWrite 等
   - 但 setAppState 写自己的 todos
    ↓
6. 流式消费消息直到子 agent end_turn
    ↓
7. 提取最终结果
   - extractResultText(agentMessages)
   - 写回到 getTaskOutputPath(agentId)
    ↓
8. 返回 { agentId, result, prompt }
   - 主 agent 看到的是单条 tool_result，干净简短
```

## 4. 6 种 built-in subagent（仅 2 种特殊化）

```ts
// builtInAgents.ts:45
const agents = [
  GENERAL_PURPOSE_AGENT,    // 默认 worker，无读写限制
  STATUSLINE_SETUP_AGENT,   // 引导用户配置 statusline
]
if (areExplorePlanAgentsEnabled()) {
  agents.push(EXPLORE_AGENT, PLAN_AGENT)  // ← 两个专门的"只读"agent
}
if (isNonSdkEntrypoint) {
  agents.push(CLAUDE_CODE_GUIDE_AGENT)
}
if (feature('VERIFICATION_AGENT') && GrowthBook) {
  agents.push(VERIFICATION_AGENT)
}
```

**重要**：源码中**只有 5 个 built-in agent**（不含 `ImplementAgent` / `TestAgent` 等），其中只有 **Explore / Plan** 是专门优化的"只读子 agent"。

---

# SkillTool —— 加载领域技能

## 1. 职责

SkillTool 让模型**主动调用领域模板**（`.claude/skills/<name>/SKILL.md`）。模型输入 skill 名（可带参数），SkillTool 把 skill 内容（带 `$ARGUMENTS` 替换）展开成完整 prompt，注入当前对话。

## 2. Schema

```ts
// SkillTool.ts:292-300
inputSchema = z.object({
  skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
  args: z.string().optional().describe('Optional arguments for the skill'),
})
```

**注意**：用 `z.object` 而非 `z.strictObject` —— SkillTool 接受额外字段（兼容性）。

## 3. 执行流程

SkillTool 有 **3 种执行路径**：

### 3.1 远程 skill（ant-only experimental）

```ts
if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
  const slug = remoteSkillModules!.stripCanonicalPrefix(commandName)
  if (slug !== null) {
    return executeRemoteSkill(slug, commandName, parentMessage, context)
  }
}
```

`_canonical_<slug>` 形式的 skill 名 → 从 AKI/GCS 加载 SKILL.md → 包装成 user message 注入。

### 3.2 Fork 执行（独立 sub-agent）

```ts
if (command?.type === 'prompt' && command.context === 'fork') {
  return executeForkedSkill(command, commandName, args, context, canUseTool, parentMessage, onProgress)
}
```

带 `context: fork` 的 skill → 派生 sub-agent 执行（与 AgentTool 类似，但 skill 模式）。

`executeForkedSkill` 流程：
1. `createAgentId()` 生成 36 进制 8 位 ID；
2. `prepareForkedCommandContext` 解析 `$ARGUMENTS`、选定 agent、构造 prompt messages；
3. `runAgent` 启动 sub-agent；
4. 流式消费消息，progress 通过 `onProgress` 上报；
5. 返回 `{ status: 'forked', agentId, result }`。

### 3.3 Inline 执行（默认）

```ts
const { processPromptSlashCommand } = await import('src/utils/processUserInput/processSlashCommand.js')
const processedCommand = await processPromptSlashCommand(commandName, args || '', commands, context)
```

通过 `processPromptSlashCommand` 处理（与 REPL 中 `/commit` 命令走相同路径）：
- 替换 `$ARGUMENTS`、`!` command substitution；
- 把 skill 内容作为 user message 注入；
- 返回 `newMessages` + `contextModifier`。

**`contextModifier` 是关键设计**：返回 `{ jsx, contextModifier }`，REPL 把 skill 执行的 allowed tools / model 覆盖 / effort 等合并到后续 tool_use 的 context。

## 4. validateInput 与 checkPermissions

```ts
// validateInput: 检查 skill 名合法 + 存在 + 是 prompt 类型 + 不禁用模型调用
async validateInput({skill}, context) {
  const trimmed = skill.trim()
  if (!trimmed) return { result: false, ... errorCode: 1 }
  
  const normalizedCommandName = trimmed.startsWith('/') ? trimmed.substring(1) : trimmed
  
  // 远程 canonical skill（ant-only）
  if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
    const slug = remoteSkillModules!.stripCanonicalPrefix(normalizedCommandName)
    if (slug !== null) {
      const meta = remoteSkillModules!.getDiscoveredRemoteSkill(slug)
      if (!meta) return { result: false, errorCode: 6 }
      return { result: true }
    }
  }
  
  const commands = await getAllCommands(context)  // 含 MCP skills
  const foundCommand = findCommand(normalizedCommandName, commands)
  if (!foundCommand) return { result: false, errorCode: 2 }
  if (foundCommand.disableModelInvocation) return { result: false, errorCode: 4 }
  if (foundCommand.type !== 'prompt') return { result: false, errorCode: 5 }
  return { result: true }
}
```

```ts
// checkPermissions: 5 层判定
// 1. deny 规则 → behavior: 'deny'
// 2. remote canonical skill → behavior: 'allow'（ant-only experimental）
// 3. allow 规则 → behavior: 'allow'
// 4. SAFE_SKILL_PROPERTIES only → behavior: 'allow'（白名单）
// 5. 默认 → behavior: 'ask'（弹窗）
```

**SAFE_SKILL_PROPERTIES 白名单**（`SkillTool.ts:876`）：PromptCommand 的所有"元数据字段"（name, description, model, effort, pluginInfo 等）都是安全的；只有用户内容相关字段需要权限。

## 5. invoke 后的上下文修改

```ts
contextModifier(ctx) {
  let modifiedContext = ctx

  // 1. allowedTools 合并到 alwaysAllowRules
  if (allowedTools.length > 0) {
    modifiedContext = { ...modifiedContext, getAppState() {
      return {
        ...appState,
        toolPermissionContext: {
          ...appState.toolPermissionContext,
          alwaysAllowRules: { command: [...new Set([..., ...allowedTools])] }
        }
      }
    }}
  }

  // 2. model 覆盖（保留 [1m] 后缀）
  if (model) {
    modifiedContext = { ...modifiedContext, options: { ...modifiedContext.options, mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel) }}
  }

  // 3. effort 覆盖
  if (effort !== undefined) {
    // setAppState 一层累加
  }

  return modifiedContext
}
```

---

# MCPTool —— MCP 标准化接入

## 1. 职责

MCPTool 是 **MCP（Model Context Protocol）服务器的工具的统一代理**。它的源码只有 77 行，**因为实际逻辑全在 `mcpClient.ts` 中**。MCPTool 本质是一个**占位符 + 名字混淆器**：模型写 `mcp__server__tool(args)`，MCPTool 把它转发给对应的 MCP 服务器进程。

## 2. Schema

```ts
// MCPTool.ts:14
export const inputSchema = lazySchema(() => z.object({}).passthrough())
```

**`passthrough()`**：允许任意字段 —— 因为每个 MCP 工具都有自己的 schema。

```ts
// MCPTool.ts:17-19
outputSchema = lazySchema(() =>
  z.string().describe('MCP tool execution result')
)
```

输出是**字符串**（MCP 的 `CallToolResult.content[0].text`）。

## 3. 关键字段

```ts
// MCPTool.ts:27
isMcp: true,
name: 'mcp',  // 被 mcpClient.ts 覆盖为 'mcp__<server>__<tool>'
async call() { return { data: '' } }  // stub，实际由 mcpClient 重写
async checkPermissions(): Promise<PermissionResult> {
  return { behavior: 'passthrough', message: 'MCPTool requires permission.' }
},
```

**`behavior: 'passthrough'`** 是关键 —— 表示权限系统**不接管** MCP 工具的权限判定，由 MCP 服务器自己处理。

## 4. 实际调用流程（mcpClient.ts 内部）

模型写 `mcp__github__create_issue({title: ..., body: ...})`：

```
模型 → tool_use block (name: 'mcp__github__create_issue')
    ↓
queryLoop.executeModelStreamingTurn
    ↓
StreamingToolExecutor.addTool(block)
    ↓
runToolUse(block) → checkPermissions → tool.call()
    ↓
mcpClient.ts 覆盖的 MCPTool.call:
  1. 解析 mcp__server__tool 格式 → serverName='github', toolName='create_issue'
  2. 找到对应 MCPServerConnection（jsonRPC client）
  3. 构造 CallToolRequestParams
  4. jsonRPC.request('tools/call', params)
  5. 等待 response（带 timeout）
  6. 解析 CallToolResult → 提取 content[0].text
  7. 返回 { data: content[0].text }
    ↓
back to model: tool_result.content = "Issue created at https://..."
```

## 5. MCP 8 种传输类型

```ts
// toolExecution.ts:272
export type McpServerType =
  | 'stdio' | 'sse' | 'http' | 'ws'
  | 'sdk' | 'sse-ide' | 'ws-ide' | 'claudeai-proxy'
```

8 种传输各有不同的 client 实现（`StdioMCPClient` / `SSEMCPClient` / `HttpMCPClient` 等）。

## 6. 与其他工具的关键区别

| 特征 | MCPTool | 其他工具 |
|------|---------|---------|
| 进程模型 | 独立 MCP 服务器进程 | 同进程内工具函数 |
| Schema 来源 | 运行时从 MCP 服务器的 `tools/list` 拉取 | 编译时 Zod 声明 |
| 权限模式 | passthrough（让 MCP 决定） | 4 层防御 |
| 输出 | 字符串 | 类型化对象 |
| 跨平台 | MCP 服务器自己处理 | Claude Code 内置 |

---

# BashTool —— shell 命令执行

## 1. 职责

BashTool 是**最复杂**的工具（1184+ 行）。它执行 shell 命令、处理输出、支持后台化、解析权限、管理 sandbox。

## 2. Schema

```ts
// BashTool.tsx:262-282
fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional()).describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
  description: z.string().optional().describe(...),  // 主动描述
  run_in_background: semanticBoolean(z.boolean().optional()).describe(...),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional()).describe(...),
  _simulatedSedEdit: z.object({  // 内部用，绝不暴露给模型
    filePath: z.string(), newContent: z.string()
  }).optional()
}))
```

**`_simulatedSedEdit` 内部字段**（`BashTool.tsx:284-287`）：

> Always omit `_simulatedSedEdit` from the model-facing schema. It is an internal-only field set by `SedEditPermissionRequest` after the user approves a sed edit preview. **Exposing it in the schema would let the model bypass permission checks and the sandbox** by pairing an innocuous command with an arbitrary file write.

**沙箱绕过防护**：如果 `_simulatedSedEdit` 暴露给模型，模型可以发 `Bash(command: "ls", _simulatedSedEdit: {filePath: "/etc/passwd", newContent: "..."})` 绕过权限。

```ts
// 始终 omit _simulatedSedEdit
const inputSchema = lazySchema(() => isBackgroundTasksDisabled
  ? fullInputSchema().omit({ run_in_background: true, _simulatedSedEdit: true })
  : fullInputSchema().omit({ _simulatedSedEdit: true })
)
```

## 3. validateInput：阻塞 sleep 模式

```ts
// BashTool.tsx:562-575
async validateInput(input: BashToolInput) {
  if (feature('MONITOR_TOOL') && !isBackgroundTasksDisabled && !input.run_in_background) {
    const sleepPattern = detectBlockedSleepPattern(input.command)
    if (sleepPattern !== null) {
      return {
        result: false,
        message: `Blocked: ${sleepPattern}. Run blocking commands in the background...`,
        errorCode: 10
      }
    }
  }
  return { result: true }
}
```

```ts
// BashTool.tsx:359-374
export function detectBlockedSleepPattern(command: string): string | null {
  const parts = splitCommand_DEPRECATED(command)
  const first = parts[0]?.trim() ?? ''
  const m = /^sleep\s+(\d+)\s*$/.exec(first)
  if (!m) return null
  const secs = parseInt(m[1]!, 10)
  if (secs < 2) return null
  const rest = parts.slice(1).join(' ').trim()
  return rest ? `sleep ${secs} followed by: ${rest}` : `standalone sleep ${secs}`
}
```

**为什么阻塞 sleep N（N ≥ 2）**：
- `sleep N` 单独存在 → "你在等什么？" → 应该用 `run_in_background: true`；
- `sleep N && check` → 应该用 `Monitor { command: check }`；
- 浮点 `sleep 0.5` 允许（用于速率限制 / pacing）。

## 4. preparePermissionMatcher：智能 hook 匹配

```ts
// BashTool.tsx:483-506
async preparePermissionMatcher({ command }) {
  // Hook `if` filtering is "no match → skip hook" (deny-like semantics), so
  // compound commands must fire the hook if ANY subcommand matches. Without
  // splitting, `ls && git push` would bypass a `Bash(git *)` security hook.
  const parsed = await parseForSecurity(command)
  if (parsed.kind !== 'simple') {
    return () => true  // parse-unavailable: fail safe
  }
  // Match on argv (strips leading VAR=val) so `FOO=bar git push` still matches `Bash(git *)`
  const subcommands = parsed.commands.map(c => c.argv.join(' '))
  return pattern => {
    const prefix = permissionRuleExtractPrefix(pattern)
    return subcommands.some(cmd => {
      if (prefix !== null) {
        return cmd === prefix || cmd.startsWith(`${prefix} `)
      }
      return matchWildcardPattern(pattern, cmd)
    })
  }
}
```

**关键设计**：compound commands 必须 fire hook if ANY subcommand matches。否则 `ls && git push` 会绕过 `Bash(git *)` 安全钩子。

**Fail-safe 策略**：parse 失败 → 返回 `() => true`（让 hook 跑），绝不静默跳过。

## 5. isSearchOrReadCommand：UI 折叠分类

```ts
// BashTool.tsx:130-207
export function isSearchOrReadBashCommand(command: string): {
  isSearch: boolean; isRead: boolean; isList: boolean
}
```

精确分类（**5 类**）：

| 类别 | 命令示例 | 折叠为 |
|------|---------|--------|
| `isSearch` | `find`, `grep`, `rg` | "Searched N files" |
| `isRead` | `cat`, `head`, `jq`, `awk`, `sort` | "Read N files" |
| `isList` | `ls`, `tree`, `du` | "Listed N directories" |
| `isSilent` | `mv`, `cp`, `rm`, `mkdir` | "Done"（无输出） |
| 其他 | npm test, build | 显示完整输出 |

**关键算法**：用 `splitCommandWithOperators` 解析 pipeline，所有子命令必须都是同一类（search/read/list）。`echo`/`printf`/`true`/`false` 视为**语义中性**（不影响分类）—— 这让 `ls dir && echo "---" && ls dir2` 仍归类为 read。

## 6. call 流程：执行、超时、后台化

```ts
// BashTool.tsx:662-858
async call(input, toolUseContext, ...) {
  if (input._simulatedSedEdit) {
    return applySedEdit(input._simulatedSedEdit, toolUseContext, parentMessage)
  }

  const commandGenerator = runShellCommand({ input, abortController, ... })
  // ↑ 内部是 async generator，yield 进度消息

  do {
    generatorResult = await commandGenerator.next()
    if (!generatorResult.done && onProgress) {
      onProgress({ toolUseID: ..., data: { type: 'bash_progress', output, fullOutput, ... } })
    }
  } while (!generatorResult.done)

  result = generatorResult.value

  // 1. 处理退出码（error → throw ShellError）
  // 2. 大输出持久化（> 30K chars → 写到 <taskId>.txt）
  // 3. 提取 claude-code-hint（剥离后传给 plugin recommendation）
  // 4. 图片输出检测 + resize
  // 5. 返回 { stdout, stderr, interrupted, isImage, ... }
}
```

### 6.1 `runShellCommand` 内部（`BashTool.tsx:865-1183`）

复杂的异步生成器，处理 **3 种触发后台化的路径**：

1. **`shellCommand.onTimeout`**：超过 `timeoutMs` → 自动后台化（如果 `shouldAutoBackground` 且命令不在 `DISALLOWED_AUTO_BACKGROUND_COMMANDS`）；
2. **`feature('KAIROS') && getKairosActive() && isMainThread`**：助手模式下超过 `ASSISTANT_BLOCKING_BUDGET_MS` (15s) → 自动后台化；
3. **`run_in_background: true`**：模型显式请求 → 立即 spawn 为后台任务。

进度循环：

```ts
while (true) {
  const progressSignal = createProgressSignal()  // Promise resolved by poller
  const result = await Promise.race([resultPromise, progressSignal])
  
  if (result !== null) {
    // 命令完成
    if (result.backgroundTaskId !== undefined) {
      // race 边缘：backgrounding fire 了但命令也完成了
      markTaskNotified(result.backgroundTaskId, setAppState)
      return { ...result, backgroundTaskId: undefined }
    }
    if (foregroundTaskId) unregisterForeground(foregroundTaskId, setAppState)
    shellCommand.cleanup()
    return result
  }
  
  // Progress 更新
  if (backgroundShellId) {
    return { backgroundTaskId: backgroundShellId, ... }
  }
  
  if (foregroundTaskId && shellCommand.status === 'backgrounded') {
    return { backgroundTaskId: foregroundTaskId, backgroundedByUser: true }
  }
  
  // 输出进度消息
  yield { type: 'progress', fullOutput, output, elapsedTimeSeconds, ... }
}
```

### 6.2 `mapToolResultToToolResultBlockParam`：模型看到的格式

```ts
// BashTool.tsx:593-661
mapToolResultToToolResultBlockParam({interrupted, stdout, stderr, isImage, backgroundTaskId, backgroundedByUser, assistantAutoBackgrounded, structuredContent, persistedOutputPath, persistedOutputSize}, toolUseID) {
  // 1. structuredContent 优先（MCP 兼容）
  if (structuredContent?.length > 0) return { content: structuredContent }
  
  // 2. 图片输出 → image content block
  if (isImage) return buildImageToolResult(stdout, toolUseID)
  
  // 3. 大输出 → <persisted-output> 包装（让模型知道去 Read 完整文件）
  if (persistedOutputPath) {
    processedStdout = buildLargeToolResultMessage({ filepath, originalSize, preview, hasMore })
  }
  
  // 4. backgrounded task → 附加 backgroundInfo 字符串
  if (backgroundTaskId) {
    backgroundInfo = assistantAutoBackgrounded
      ? `Command exceeded the assistant-mode blocking budget (${ASSISTANT_BLOCKING_BUDGET_MS / 1000}s) and was moved to the background...`
      : backgroundedByUser
        ? `Command was manually backgrounded by user with ID: ${backgroundTaskId}. Output is being written to: ${outputPath}`
        : `Command running in background with ID: ${backgroundTaskId}. Output is being written to: ${outputPath}`
  }
  
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: [processedStdout, errorMessage, backgroundInfo].filter(Boolean).join('\n'),
    is_error: interrupted,
  }
}
```

**重要注释**（`BashTool.tsx:582-585`）：

> `BashToolResultMessage` shows `<OutputLine content={stdout}>` + stderr. **UI never shows `persistedOutputPath` wrapper, `backgroundInfo`** — those are model-facing (mapToolResult... below).

`persistedOutputPath` wrapper 和 `backgroundInfo` **只给模型看**——UI 用 `data.stdout`（已含 inline 截断）。

### 6.3 用户面工具名（userFacingName）的性能陷阱

```ts
// BashTool.tsx:540
userFacingName(input) {
  if (!input) return 'Bash'
  if (input.command) {
    const sedInfo = parseSedEditCommand(input.command)  // ← 注意：每次 render 都跑
    if (sedInfo) {
      return fileEditUserFacingName({ file_path: sedInfo.filePath, old_string: 'x' })
    }
  }
  // Env var FIRST: shouldUseSandbox → splitCommand_DEPRECATED → shell-quote's
  // `new RegExp` per call. userFacingName runs per-render for every bash
  // message in history; with ~50 msgs + one slow-to-tokenize command, this
  // exceeds the shimmer tick → transition abort → infinite retry (#21605).
  return isEnvTruthy(process.env.CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR) && shouldUseSandbox(input) ? 'SandboxedBash' : 'Bash'
}
```

注释解释了 `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR` 必须在 `shouldUseSandbox` **之前**检查的原因——`shouldUseSandbox` 调用 `splitCommand_DEPRECATED` 和 `shell-quote` 的 `new RegExp`，每次 render 都跑。~50 条 bash 历史 × 慢命令 = 超过 React 的 shimmer tick → transition abort → 死循环。

## 7. `applySedEdit`：内嵌 sed 直接落盘

```ts
// BashTool.tsx:397-456
async function applySedEdit(simulatedEdit, toolUseContext, parentMessage) {
  const absoluteFilePath = expandPath(filePath)
  const encoding = detectFileEncoding(absoluteFilePath)
  
  const originalContent = await fs.readFile(absoluteFilePath, { encoding })
  
  // Track file history
  if (fileHistoryEnabled() && parentMessage) {
    await fileHistoryTrackEdit(toolUseContext.updateFileHistoryState, absoluteFilePath, parentMessage.uuid)
  }
  
  // Detect line endings
  const endings = detectLineEndings(absoluteFilePath)
  writeTextContent(absoluteFilePath, newContent, encoding, endings)
  
  notifyVscodeFileUpdated(absoluteFilePath, originalContent, newContent)
  
  toolUseContext.readFileState.set(absoluteFilePath, {
    content: newContent,
    timestamp: getFileModificationTime(absoluteFilePath),
    offset: undefined, limit: undefined
  })
  
  return { data: { stdout: '', stderr: '', interrupted: false } }  // sed 成功无输出
}
```

**为什么需要这个**：`SedEditPermissionRequest` 让用户预览 sed 修改（如 `s/foo/bar/g`），用户批准后 `applySedEdit` **直接调用 FileWriteTool 的逻辑**而不是运行 sed——保证用户预览的内容与实际写入完全一致。

---

# 四者的协同模式

下面以"用户要求生成网页版贪吃蛇"为例，展示 4 个工具的典型协作：

```
T=0    用户输入 → queryLoop
       模型推理 → 决定用 AgentTool 派生 Explore subagent

T=10ms tool_use: Agent(
         prompt: "Survey current directory for conflicts",
         subagent_type: 'Explore'
       )
       ↓
       AgentTool.call
       ├─ getAgentModel → haiku
       ├─ createSubagentContext (setAppState no-op)
       ├─ buildForkedMessages (fork 主对话)
       ├─ 注册 LocalAgentTask, taskId='a1b2'
       ├─ 启动子 queryLoop (独立 5 级压缩)
       │     子 agent 调用 Read/Bash/Glob 探索目录
       │     返回 "directory is empty, safe to create"
       └─ 主 agent 收到 {agentId, result: "directory is empty..."}

T=3s   tool_use: Skill(skill: 'web-artifacts-builder', args: '贪吃蛇游戏')
       ↓
       SkillTool.call
       ├─ findCommand → 找到 web-artifacts-builder skill
       ├─ checkPermissions → 假设 SAFE_SKILL_PROPERTIES only → allow
       ├─ processPromptSlashCommand → 替换 $ARGUMENTS，构造完整 prompt
       ├─ 返回 inline newMessages
       └─ contextModifier.allowedTools → 修改 canUseTool 集合

       tool_use: Bash(command: "npm create vite@latest snake-game -- --template react-ts")
       ↓
       BashTool.call
       ├─ validateInput → detectBlockedSleepPattern 无 → pass
       ├─ checkPermissions → 4 层防御 → 用户 ack
       ├─ runShellCommand
       │     ├─ exec(command, ...) → spawn bash 子进程
       │     ├─ Promise.race(resultPromise, progressSignal)
       │     └─ yield 进度（如果 > 2s）
       ├─ result.stdout → 30K chars → 截断 + persist
       └─ mapToolResultToToolResultBlockParam → "Vite project created"

T=20s  tool_use: mcp__github__create_issue({
         title: "WIP: snake-game"
       })
       ↓
       MCPTool.call (实际是 mcpClient 覆盖的)
       ├─ 解析 mcp__github__create_issue → serverName='github'
       ├─ jsonRPC.request('tools/call', params)
       └─ 返回 content[0].text = "Issue created at https://..."

T=30s  模型继续推进... 最终 end_turn
```

---

# 共性与差异总结

## 1. 共同的设计原则

四个工具都遵循 Claude Code 的 8 条核心设计：

| 原则 | 体现 |
|------|------|
| **类型化 union** | Zod schema + `OutputSchema` typed union |
| **副作用 yield 化** | 流式 progress 消息（onProgress） |
| **缓存键稳定性** | 工具列表固定排序（`assembleToolPool`） |
| **权限 4 层防御** | 所有写操作工具都走 rules → classifier → ask 路径 |
| **持久化** | transcript 写盘 + tool result 写文件 |
| **可观测性** | `logEvent('tengu_*')` 埋点 |
| **错误 typed union** | `AbortError` / `ShellError` / `ImageSizeError` 等 |
| **懒加载** | `lazySchema` + `require()` 条件加载 |

## 2. 核心差异

| 维度 | AgentTool | SkillTool | MCPTool | BashTool |
|------|-----------|-----------|---------|----------|
| **进程模型** | 同进程子 agent | inline 修改 context 或同进程子 agent | 独立 MCP 服务器进程 | 同进程 fork 子 shell |
| **持久化** | `<session>/<agentId>.txt` | inline messages（不持久化）/ fork 模式持久化 | MCP 服务器自己管 | `<taskId>.txt` |
| **token 消耗** | 独立配额（独立 query） | inline 不消耗 / fork 消耗 | MCP 自管 | 不消耗 |
| **可中断** | ✅（子 agent abort） | inline 不可中断 / fork 可中断 | RPC timeout | ✅（abortController） |
| **可后台** | ✅ | fork 模式天然后台 | ❌ | ✅ |
| **并发** | 独立 | inline 顺序 / fork 独立 | RPC 同步 | 是（只读）/ 否（写） |
| **权限** | 继承父 | 5 层（deny/remote/allow/safe/ask） | passthrough | 4 层防御 |
| **副作用范围** | 隔离 setAppState | inline 修改当前 context | 调用外部 | 同进程文件/网络 |

## 3. 设计哲学差异

- **AgentTool**：**隔离 + 上下文压缩**——子 agent 独立，但通过 fork_messages 共享历史；
- **SkillTool**：**模板注入 + 上下文覆盖**——把 skill prompt 展开后注入当前对话，通过 `contextModifier` 修改 allowedTools / model / effort；
- **MCPTool**：**协议抽象 + 进程隔离**——通过 MCP 协议把外部工具统一成 Claude Code 内部接口；
- **BashTool**：**最大化能力 + 严格管控**——shell 是最强大的工具，所以也是最复杂的（1184 行），需要 sandbox、classifier、模拟 sed 等特殊处理。

## 4. 何时使用哪个？

| 用户需求 | 应该用 |
|---------|--------|
| "帮我探索一下代码库" | AgentTool + `subagent_type: 'Explore'` |
| "按 PR review 流程检查我的代码" | SkillTool + `skill: 'review-pr'` |
| "查 GitHub issue" | MCPTool + `mcp__github__list_issues` |
| "跑一下测试" | BashTool + `command: 'npm test'` |
| "找一下文件" | 优先 AgentTool（Explore agent），简单场景用 GlobTool |
| "执行领域流程" | SkillTool（如果存在对应 skill） |
| "调用外部 API" | 优先 MCPTool（如果 MCP 服务器已配），否则 BashTool (curl) |

---

## 附录：关键源码速查

| 模块 | 文件 | 关键函数 |
|------|------|---------|
| AgentTool 入口 | `src/tools/AgentTool/AgentTool.tsx` | `call(input, context)` |
| Subagent 隔离 | `src/utils/forkedAgent.ts` | `createSubagentContext` |
| Built-in agents | `src/tools/AgentTool/builtInAgents.ts` + `built-in/*.ts` | `getBuiltInAgents()` |
| Subagent 执行 | `src/tools/AgentTool/runAgent.ts` | `runAgent(params)` |
| Skill 解析 | `src/utils/processUserInput/processSlashCommand.js` | `processPromptSlashCommand` |
| 远程 skill | `src/services/skillSearch/remoteSkillLoader.ts` | `loadRemoteSkill(slug, url)` |
| MCP 客户端 | `src/services/mcp/client.ts` (3348 行) | `callTool` |
| MCP 连接管理 | `src/services/mcp/mcpConnectionManager.tsx` | `connect(serverConfig)` |
| Bash 执行 | `src/utils/Shell.ts` + `src/utils/ShellCommand.ts` | `exec(command, signal, shell, opts)` |
| Bash 分类器 | `src/utils/permissions/bashClassifier.ts` | `classifyBashCommand` |
| 沙箱适配 | `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager.shouldUseSandbox` |
| 文件历史 | `src/utils/fileHistory.ts` | `fileHistoryTrackEdit` |

---

## 推荐阅读

- [`docs/11-tool-execution.md`](11-tool-execution.md) —— 工具执行通用路径
- [`docs/12-run-agent.md`](../task/12-run-agent.md) —— Subagent 派生与 fork
- [`docs/14-tool-hooks.md`](14-tool-hooks.md) —— PreToolUse / PostToolUse 钩子
- [`docs/25-streaming-tool-executor.md`](25-streaming-tool-executor.md) —— 4 个工具如何被并发执行
- [`docs/27-task-tools.md`](../task/27-task-tools.md) —— 后台任务工具族
- [`docs/23-design-and-core-modules.md`](../architecture/23-design-and-core-modules.md) —— 整体设计原理