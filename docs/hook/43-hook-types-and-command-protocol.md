# Hook 类型详解与 Command Hook 输入输出协议

> 源码位置:
> - dispatcher 主入口：`src/utils/hooks.ts`
> - command 执行器：`src/utils/hooks.ts:830-1418` 的 `execCommandHook`
> - prompt 执行器：`src/utils/hooks/execPromptHook.ts:21`
> - agent 执行器：`src/utils/hooks/execAgentHook.ts:36`
> - http 执行器：`src/utils/hooks/execHttpHook.ts:123`
> - callback / session hook：`src/utils/hooks/sessionHooks.ts:1-` 的 `FunctionHook` / `addFunctionHook`
> - 输入构造：`src/utils/hooks.ts:302-329` 的 `createBaseHookInput`
> - 输出解析：`src/utils/hooks.ts:383-452` 的 `validateHookJson` + `parseHookOutput`
> - 输出处理：`src/utils/hooks.ts:569-` 的 `processHookJSONOutput`
> - 类型定义：`src/types/hooks.ts`

> 关联文档：
> - [`docs/hook/15-utils-hooks.md`](15-utils-hooks.md) —— hooks.ts 总入口与 30+ 公共 API
> - [`docs/hook/41-hook-system-design.md`](41-hook-system-design.md) —— Hook 系统设计
> - [`docs/hook/42-useCanUseTool.md`](42-useCanUseTool.md) —— 工具权限入口与 PermissionRequest hook 集成

`hooks.ts` 中的 `executeHooks` dispatcher 是统一入口，但 `hook.type` 决定了实际执行路径。本文先概览 5 种 hook 类型，再**重点拆解 command hook 的输入输出协议**——这是用户最常写、最容易踩坑的一种。

---

## 目录

**第一部分：5 种 Hook 类型概览**
1. [5 种类型一览](#1-5-种类型一览)
2. [command hook —— 执行 shell 命令](#2-command-hook--执行-shell-命令)
3. [prompt hook —— 单次 LLM 评判](#3-prompt-hook--单次-llm-评判)
4. [agent hook —— Forked sub-agent 多轮推理](#4-agent-hook--forked-sub-agent-多轮推理)
5. [http hook —— 外部 HTTP 调用](#5-http-hook--外部-http-调用)
6. [callback hook —— 进程内函数回调](#6-callback-hook--进程内函数回调)
7. [对照表：何时选哪个](#7-对照表何时选哪个)

**第二部分：Command Hook 输入输出协议**
8. [输入协议](#8-输入协议)
9. [环境变量与变量替换](#9-环境变量与变量替换)
10. [输出协议](#10-输出协议)
11. [退出码语义](#11-退出码语义)
12. [JSON 输出的完整 schema](#12-json-输出的完整-schema)
13. [实际示例：PreToolUse 鉴权审计](#13-实际示例pretooluse-鉴权审计)
14. [常见陷阱](#14-常见陷阱)

**第三部分：附录**
15. [关键文件索引](#15-关键文件索引)

---

# 第一部分：5 种 Hook 类型概览

## 1. 5 种类型一览

| 类型 | 触发函数 | 执行环境 | 默认超时 | 入参方式 | 出参方式 | 配置位置 |
|------|---------|---------|---------|---------|---------|---------|
| `command` | `execCommandHook` (`hooks.ts:830`) | 子进程 shell (bash / pwsh) | 10 分钟 | `$ARGUMENTS` 替换 + 环境变量 | stdout/stderr + exit code | `settings.json` / plugin |
| `prompt` | `execPromptHook` (`execPromptHook.ts:21`) | 单次 LLM 调用 (Haiku) | 30 秒 | `$ARGUMENTS` 替换为 JSON | JSON `{ok, reason?}` | `settings.json` / plugin |
| `agent` | `execAgentHook` (`execAgentHook.ts:36`) | Forked sub-agent (multi-turn) | 60 秒 | `$ARGUMENTS` 替换 + transcript path | JSON via StructuredOutput 工具 | `settings.json` / plugin |
| `http` | `execHttpHook` (`execHttpHook.ts:123`) | HTTP POST | 10 分钟 | JSON body | raw HTTP response | `settings.json` / plugin |
| `callback` | `executeHookCallback` (in-process) | 同进程回调函数 | 5 秒 | 直接传消息数组 | boolean | **仅内存**：只能通过 `addFunctionHook()` 注入，不能写 `settings.json` |

**关键差别**：
- `command` / `prompt` / `agent` / `http` 都允许用户在 `settings.json` 里声明，跨 session 持久化
- `callback` 是 session-scoped，只能程序化注入（SDK / 内部模块），不持久化
- 所有 5 种最终都被同一个 `executeHooks` 调度

---

## 2. command hook —— 执行 shell 命令

来源：`src/utils/hooks.ts:830-1418` 的 `execCommandHook`

### 工作机制
1. 接收 `HookCommand & { type: 'command' }`，包含 `command`、`shell`、`timeout`、可选 `requestPrompt` 等
2. **Shell 选择**（`hooks.ts:873-875`）：`hook.shell → DEFAULT_HOOK_SHELL`，默认 bash，可指定 `powershell`
3. **Windows 路径处理**（`:891-894`）：
   - bash: `windowsPathToPosixPath()` 把 `C:\Users\foo` 转成 `/c/Users/foo`
   - PowerShell: 跳过转换，用原生路径
4. **变量替换**：先 `${CLAUDE_PLUGIN_ROOT}` 等 plugin 变量，再 `${user_config.X}` 用户配置
5. **环境变量注入**：`CLAUDE_PROJECT_DIR` / `CLAUDE_PLUGIN_ROOT` / `CLAUDE_SESSION_ID` 等
6. **后台运行支持**：`executeInBackground()` 把命令转后台进程；`registerPendingAsyncHook()` 进 `AsyncHookRegistry`
7. **诊断日志**：仅 SessionStart / Setup / SessionEnd 三个 once-per-session 事件开启

### 输出解析
- `parseHookOutput` (`hooks.ts:400-452`) 处理文本或 JSON 输出
- 详见本文第二部分"输出协议"

### 典型用途
- **审计日志**：把每次 `PreToolUse` 写到 `/var/log/claude.jsonl`
- **通知**：执行 `Bash` 工具前给 Slack 发 webhook
- **CI 集成**：Stop hook 触发下游流水线
- **本地副作用**：SessionStart 启动 docker-compose、PreCompact 清理临时文件

### 配置示例
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "echo \"$CLAUDE_TOOL_INPUT\" >> ~/.claude-audit.log",
        "timeout": 5
      }]
    }]
  }
}
```

---

## 3. prompt hook —— 单次 LLM 评判

来源：`src/utils/hooks/execPromptHook.ts:21-211`

### 工作机制
1. **输入替换**（`:34-35`）：`addArgumentsToPrompt(hook.prompt, jsonInput)` 把 prompt 模板里的 `$ARGUMENTS` 替换为 hook JSON 输入
2. **跳过递归**（`:41-42`）：直接 `createUserMessage({content: processedPrompt})`，绕开 `processUserInput`，否则会触发 `UserPromptSubmit` 钩子导致无限递归
3. **附加消息**：如果传入了 `messages` 参数（前置会话上下文），会拼在前面
4. **LLM 查询**（`:62-100`）：
   - 用 `queryModelWithoutStreaming`（非流式，单次性）
   - 模型：`hook.model ?? getSmallFastModel()`（默认 Haiku）
   - `outputFormat.type = "json_schema"` 强约束输出
5. **响应解析**：Zod schema 校验 → outcome 四种态（success/blocking/non_blocking_error/cancelled）

### 关键限制
- **不能调用工具**（与 `agent` hook 核心区别）
- **更便宜的 token**：默认 Haiku
- **更高的一致性**：单次调用、无工具噪声，输出可预测

### 典型用途
语义化 Allow/Deny、内容审计、响应校验、规则建议

---

## 4. agent hook —— Forked sub-agent 多轮推理

来源：`src/utils/hooks/execAgentHook.ts:36-343`

最强大的一种 hook：完整子 agent，能用任何工具。

### 工作机制
1. **Skill 解析**：与 prompt hook 一样的 `$ARGUMENTS` 替换
2. **构造脚本式 agent 上下文**：
   - **transcript path** 注入 system prompt，让 agent 看完整对话历史
   - **systemPrompt**：明确的"验证 stop condition"任务声明
   - **MAX_AGENT_TURNS = 50**：硬上限
3. **特殊工具集**：
   - 父 ctx 的所有工具，过滤掉 `ALL_AGENT_DISALLOWED_TOOLS`
   - 过滤掉现有 `SYNTHETIC_OUTPUT_TOOL_NAME` 避免 schema 冲突
   - 追加 `structuredOutputTool`
4. **多轮循环**：`for await (const message of query({...}))`

### 与 prompt hook 的核心区别

| 维度 | prompt hook | agent hook |
|------|-------------|------------|
| 工具调用 | ❌（强约束 JSON） | ✅（完整工具集） |
| 轮数 | 1 轮 | 多轮（≤50） |
| 上下文 | 仅 `messages` + hook 输入 | 完整 transcript + 当前 messages |
| 模型 | 仅 Haiku | Haiku/Sonnet/Opus 任选 |
| 默认超时 | 30s | 60s |
| 典型场景 | 判断 | 验证 + 操作 |

### 典型用途
Stop hook 验证、复杂 context 检查、结构化验证

---

## 5. http hook —— 外部 HTTP 调用

来源：`src/utils/hooks/execHttpHook.ts:123-`

### 工作机制
1. **URL 允许列表校验**：读 `settings.allowedHttpHookUrls`，`*` 通配符匹配
   - `undefined` → 不限制
   - `[]` → 全部拒绝
   - 非空 → 必须匹配至少一个 pattern
2. **超时**：默认 10 分钟
3. **Header 构造**：
   - `Content-Type: application/json` 强制
   - 用户 header 做 `interpolateEnvVars(value, allowedEnvVars)` —— 替换 `$VAR_NAME`
   - **allowlist 双重过滤**：policy 与 hook 的 `allowedEnvVars` 取交集
4. **沙箱代理路由**：启用了 sandbox 时强制走代理，代理做域名 allowlist 强校验
5. **SSRF 防护**：`ssrfGuardedLookup`（防 DNS rebinding + 私有 IP 直连）

### 安全边界

| 维度 | 控制 |
|------|------|
| URL 域名 | `allowedHttpHookUrls` allowlist + sandbox proxy |
| Env 变量泄漏 | `allowedEnvVars` 交集过滤 |
| HTTP header 注入 | `sanitizeHeaderValue` 剥除 `\r\n\0` |
| 私有 IP / SSRF | `ssrfGuardedLookup` |

### 典型用途
公司内部 webhook、外部 CI 触发、Slack/Discord/Teams 通知、SIEM 上报

---

## 6. callback hook —— 进程内函数回调

来源：`src/utils/hooks/sessionHooks.ts:1-`

### 与前面 4 种的本质区别
**不能**通过 `settings.json` 配置。只能通过 SDK / 内部模块**程序化注入**到 sessionHooks 状态。

### 类型定义
```ts
type FunctionHookCallback = (
  messages: Message[],
  signal?: AbortSignal,
) => boolean | Promise<boolean>

type FunctionHook = {
  type: 'function'
  id?: string              // 可选唯一 ID，用于 removeFunctionHook
  timeout?: number         // 默认 5000ms
  callback: FunctionHookCallback
  errorMessage: string
  statusMessage?: string
}
```

### 注入方式
通过 `addFunctionHook(setAppState, sessionId, event, matcher, callback, errorMessage, opts?)`，返回 hook ID，可通过 `removeFunctionHook(...)` 按 ID 移除。

### Map-backed 状态
`SessionHooksState = Map<string, SessionStore>`（用 Map 不是 Record，因为 high-concurrency 下 `.set()` 是 O(1) 且 `mutator + return prev` 让监听器短路）。

### 典型用途
- **SDK 注入**：用户在 SDK 里用 JS 注册一个 callback hook
- **内部 Claude Code 功能**：
  - `registerStructuredOutputEnforcement` —— 给 agent hook 注册 session 级 stop hook
  - 投机 context 加载、文件访问分析、身份验证、知识截止提醒

---

## 7. 对照表：何时选哪个

| 你想做的事 | 选哪个 hook | 原因 |
|-----------|-----------|------|
| 把 hook 输出写文件 / 发 webhook / 启后台进程 | **`command`** | shell 能做一切，又快又便宜 |
| 用 LLM 判断"对/不对" | **`prompt`** | 单轮 Haiku，不能乱动工具，输出一致可预测 |
| 验证"任务真的完成了"需要读文件、跑命令 | **`agent`** | 多轮 + 工具，能"看完再说" |
| 把事件发到公司内部系统 / 外部 SaaS | **`http`** | 跨进程边界，且有 allowlist + 沙箱代理保安全 |
| SDK / 内部模块要注入一段程序化校验逻辑 | **`callback`** | session 级，最快（同步、零 spawn 成本），不能持久化 |

---

# 第二部分：Command Hook 输入输出协议

本部分聚焦 `command` hook——用户写最多的那种——的完整输入输出契约。这是用户最容易踩坑的地方，**写错了不会报错，只会安静地失败**。

## 8. 输入协议

### 8.1 总体：JSON 在 stdin 上传一行

Command hook 通过 **stdin 接收 hook 输入**，格式是 **JSON 字符串 + 末尾换行符**（见 `hooks.ts:1280-1299` 的 `stdinWritePromise`）：

```js
child.stdin.write(jsonInput + '\n', 'utf8')
child.stdin.end()
```

**末尾换行符是关键**。没有它，bash `read -r line` 会 EOF 提前退出；变量会被填充但 `if read -r line; then ...` 分支会跳过（参见 gh-30509 / CC-161 注释在 `hooks.ts:1085-1088`）。

### 8.2 输入 JSON 结构：BaseHookInput + Per-Event 扩展

`createBaseHookInput` (`hooks.ts:302-329`) 是所有 hook 输入的公共底座：

```typescript
{
  // ----- 公共字段（所有 hook 都有） -----
  session_id: string,         // 当前 session ID
  transcript_path: string,    // JSONL transcript 文件路径
  cwd: string,                // 当前工作目录
  permission_mode?: string,   // 'default' / 'acceptEdits' / 'plan' / 'bypassPermissions' / 'dontAsk' / 'auto'
  agent_id?: string,          // 子 agent 的 ID（仅子 agent 调用时存在）
  agent_type?: string,        // agent type 名（subagent 优先于主线程 --agent 参数）
  
  // ----- 事件特定字段（按 hook_event_name 扩展） -----
  hook_event_name: string,    // 'PreToolUse' / 'PostToolUse' / ... 见 HOOK_EVENTS
  // 各事件再加自己的字段，例如：
  // PreToolUse → tool_name, tool_input
  // PostToolUse → tool_name, tool_input, tool_response
  // SessionStart → source ('startup' / 'resume' / 'clear')
  // UserPromptSubmit → prompt
  // Stop → stop_hook_active
  // ... 共 30+ 事件类型
}
```

完整的事件列表与字段定义见 [`src/types/hooks.ts`](../../src/types/hooks.ts)（`PreToolUseHookInput` / `PostToolUseHookInput` / ... / `ElicitationResultHookInput` 等）。

### 8.3 实际收到的 JSON 示例（PreToolUse）

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/audit.sh
INPUT=$(cat)
echo "$INPUT" | jq -r '"[\(.timestamp // \"\")] \(.tool_name): \(.tool_input.command // \"\")"' >> ~/.claude-audit.log
exit 0
```

上游会传入的 stdin 内容是：

```json
{
  "session_id": "abc123-...",
  "transcript_path": "/Users/foo/.claude/projects/abc123.jsonl",
  "cwd": "/Users/foo/myproject",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "rm -rf node_modules",
    "description": "Clean install"
  }
}
```

### 8.4 Bash 一行命令读取输入

最常见的写法：

```bash
#!/usr/bin/env bash
set -euo pipefail
INPUT="$(cat)"   # stdin 一行 JSON
# 解析
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
# 判定...
echo "{\"decision\": \"block\", \"reason\": \"forbidden\"}"
exit 0
```

**Node / Python 也一样**：直接 `process.stdin.read()` 或 `sys.stdin.read()` 拿到 JSON 字符串，解析即可。

### 8.5 异步 hook 与 `requestPrompt`

特殊场景：hook 启动后**不立刻返回决策**，而是想先问用户一个问题：

```json
{
  "type": "command",
  "command": "my-interactive-audit-tool",
  "timeout": 300
}
```

- 若传入了 `requestPrompt` 函数（默认 REPL 上下文），hook stdout 写入符合 `promptRequestSchema` 的 JSON 行时，会被自动解析成 prompt 请求发到 UI
- 实现细节见 `hooks.ts:1156-1193`
- prompt 响应通过 `child.stdin.write(...)` 回传 hook 进程

普通用户写 hook 时**不需要关心这个**，只有 SDK / 高级插件才会用。

---

## 9. 环境变量与变量替换

### 9.1 自动注入的环境变量

`hooks.ts:964-1009` 在 spawn 时强制设置以下环境变量：

| 变量 | 说明 | 设置条件 |
|------|------|---------|
| `CLAUDE_PROJECT_DIR` | 稳定项目根（**不**是 worktree 路径） | 总是 |
| `CLAUDE_PLUGIN_ROOT` | Plugin 根目录 | `pluginRoot` 存在时 |
| `CLAUDE_PLUGIN_DATA` | Plugin 数据目录 | `pluginId` 存在时 |
| `CLAUDE_PLUGIN_OPTION_<KEY>` | 用户在 plugin manifest.userConfig 里配置的每个选项 | `pluginOpts` 存在时 |
| `CLAUDE_ENV_FILE` | hook 可写入的 .sh 文件路径（写 env var 给后续 hook 用）| SessionStart / Setup / CwdChanged / FileChanged 的 bash hook |
| `CLAUDE_SESSION_ID` | Session ID | 总是（继承 `subprocessEnv()`） |
| `CLAUDE_CODE_SHELL_PREFIX` | 全局命令前缀（POSIX quoted）| 该环境变量已设置时 |

注意 `CLAUDE_PLUGIN_ROOT` 对 plugin **和** skill 都生效（`:990-992`），保持向后兼容。

### 9.2 命令字符串里的变量替换（先后顺序敏感）

执行命令前先做两次替换（`hooks.ts:905-939`）：

1. **Plugin / Skill 变量**：顺序很关键
   - `${CLAUDE_PLUGIN_ROOT}` → `toHookPath(pluginRoot)`
   - `${CLAUDE_PLUGIN_DATA}` → `toHookPath(getPluginDataDir(pluginId))`
   - **使用函数式 `.replace()`**（`:928, 931`）防止路径里的 `$` 被模式解释（例如 `\\server\c$\plugin`）
2. **Plugin userConfig 变量**：`substituteUserConfigVariables(command, pluginOpts)`，缺 key 时抛错

Windows bash 路径走 `windowsPathToPosixPath()`（`:891-894`），PowerShell 跳过转换用原生路径。

### 9.3 `.sh` 自动前缀

Windows bash 路径下（`:945-949`）：如果命令含 `.sh`，自动 prepend `bash `，避免 `.sh` 在 Windows 默认 handler 里被"打开"而不是执行。PowerShell 跳过这一步（`.ps1` 直接 native 执行）。

### 9.4 `CLAUDE_CODE_SHELL_PREFIX`

环境变量 `CLAUDE_CODE_SHELL_PREFIX` 会把命令包一层 POSIX quote 后的前缀（`hooks.ts:955-958`，详见 15-utils-hooks.md 的"变量替换"行）。**PowerShell 跳过这个前缀**——POSIX quote 对 PS 无意义。

---

## 10. 输出协议

### 10.1 总体：stdout 决定 outcome

command hook 通过 **stdout** 决定它对主流程的影响。一共三种合法的输出模式：

1. **不输出任何 JSON** → 视为成功，hook 跑完了就完了（"非阻塞通过"）
2. **stdout 输出 JSON 起始（以 `{` 开头）** → 走 `parseHookOutput` → `processHookJSONOutput` 解释
3. **stdout 输出纯文本** → 也走 `parseHookOutput`，但被当成 plain text（`parseHookOutput:400-408` 的 trim + startsWith('{') 判断）

stderr 仅作为诊断输出，不影响 outcome（除非 `[WARN]` `[ERROR]` 被特殊 hook 识别）。

### 10.2 JSON 输出的顶层 schema

合法 JSON 输出顶层允许的字段（来源 `parseHookOutput` 的 schema hint 在 `hooks.ts:417-446`，以及 `processHookJSONOutput` 的实际处理）：

```typescript
{
  // 通用字段（所有事件）
  continue?: boolean,                    // false → preventContinuation = true，停止主流程
  suppressOutput?: boolean,               // 抑制 Claude 看到 hook output（避免循环）
  stopReason?: string,                    // 当 continue=false 时告诉 Claude 为什么不继续
  decision?: 'approve' | 'block',         // 简化版权限决策
  reason?: string,                        // 给用户的解释文本
  systemMessage?: string,                 // 给用户看的系统消息（不发给模型）
  
  // 旧版权限字段（部分事件）
  permissionDecision?: 'allow' | 'deny' | 'ask',
  
  // 事件特定输出
  hookSpecificOutput?: {
    // PreToolUse：权限决策 + 修改输入
    hookEventName: 'PreToolUse',
    permissionDecision?: 'allow' | 'deny' | 'ask',
    permissionDecisionReason?: string,
    updatedInput?: object,                // 修改后的工具输入
    
    // UserPromptSubmit：注入上下文
    hookEventName: 'UserPromptSubmit',
    additionalContext: string,            // 必填
    
    // PostToolUse：注入上下文 + 修改 MCP 输出
    hookEventName: 'PostToolUse',
    additionalContext?: string,
    
    // PostToolUseFailure：注入上下文
    hookEventName: 'PostToolUseFailure',
    additionalContext?: string,
    
    // SessionStart：上下文 + 首条消息 + 文件监听
    hookEventName: 'SessionStart',
    additionalContext?: string,
    initialUserMessage?: string,
    watchPaths?: string[],
    
    // Setup / SubagentStart：上下文
    hookEventName: 'Setup' | 'SubagentStart',
    additionalContext?: string,
    
    // PermissionDenied：重试
    hookEventName: 'PermissionDenied',
    retry?: boolean,
    
    // PermissionRequest：内联 allow/deny + updatedInput
    hookEventName: 'PermissionRequest',
    decision?: { behavior: 'allow' | 'deny', updatedInput?: object },
    
    // Elicitation：响应
    hookEventName: 'Elicitation',
    action?: 'accept' | 'decline' | 'dismiss',
    content?: object,
    
    // FileChanged / WorktreeCreate
    hookEventName: 'FileChanged',
    watchPaths?: string[],
    // WorktreeCreate 返回 worktree 路径
  }
}
```

完整 schema 由 `hookJSONOutputSchema()`（在 `hookHelpers.ts`）定义为 Zod schema，hook 输出必须满足它。

### 10.3 JSON 行为如何映射到 HookResult

`processHookJSONOutput` (`hooks.ts:569-`) 把上面的 schema 翻译成内部 `HookResult`：

| JSON 字段 | 对应 HookResult 字段 |
|----------|---------------------|
| `continue: false` | `preventContinuation: true` + `stopReason` |
| `decision: 'approve'` | `permissionBehavior: 'allow'` |
| `decision: 'block'` | `permissionBehavior: 'deny'` + `blockingError` |
| `systemMessage: '...'` | `result.systemMessage`（给用户） |
| `hookSpecificOutput.permissionDecision: 'allow'` | `permissionBehavior: 'allow'` |
| `hookSpecificOutput.permissionDecision: 'deny'` | `permissionBehavior: 'deny'` + `blockingError` |
| `hookSpecificOutput.permissionDecision: 'ask'` | `permissionBehavior: 'ask'`（弹出对话框） |
| `hookSpecificOutput.updatedInput` | `result.updatedInput`（替换原 tool_input） |
| `hookSpecificOutput.additionalContext` | `result.additionalContext`（注入到上下文） |
| `hookSpecificOutput.initialUserMessage` | `result.initialUserMessage`（替换用户首条消息） |
| `hookSpecificOutput.watchPaths` | `result.watchPaths`（触发 FileChanged 监听） |

### 10.4 输出截断：`suppressOutput`

`suppressOutput: true` 让 Claude **看不到** hook 的 stdout。常用于：审计 hook（只是写文件，不想让模型看到）、修改用户输入的 hook（额外的"我改了 prompt"信息可能会让模型分心）。

否则 hook 的 stdout 会作为 attachment 消息出现在 transcript 里。

---

## 11. 退出码语义

`execCommandHook` 退出码的处理：

| Exit Code | 语义 | 来源 |
|-----------|------|------|
| **0** | 成功（无论 stdout 是不是 JSON）| `hooks.ts:1318` `code ?? 1`（默认 1） |
| **2** | **阻塞错误**（传统 Unix 惯例）| 用户实现自己控制 — 当前 dispatcher 主要看 stdout JSON 不看 exit code，但 abort / EPIPE 路径会置 `status: 1` |
| **ABORT_ERR**（即 signal 中止）| 视为取消 → outcome: `cancelled` | `hooks.ts:1383-1391` |
| **EPIPE**（stdin 关闭）| hook 命令提前退出 → status 1, stderr 提示 | `hooks.ts:1371-1382` |
| **其他非零** | 通用错误 → outcome: `non_blocking_error` | `hooks.ts:1392-1400` |

**目前主要靠 JSON stdout 决定 outcome**，退出码只影响 outcome 是 `success` 还是 `non_blocking_error`/`cancelled`，不影响 `blocking`。

但有一条传统规则：**任何事件都用 `decision: "block"` 字段来阻塞**，而不是靠退出码 2。

---

## 12. JSON 输出的完整 schema

完整的 hook JSON 输出 schema 由 `hookJSONOutputSchema()` Zod schema 定义。helper 函数：

- `parseHookOutput(stdout)`：解析 + 校验 → `{ json, plainText, validationError }`
- `validateHookJson(jsonString)`：纯 schema 校验

校验失败时（`hooks.ts:392-397`），返回 `validationError` 字符串，包含详细字段错误，**不影响 hook outcome**——hook 仍然 outcome: `non_blocking_error`，错误信息进 hook attachment transcript 让用户看到。

每个事件要求的 `hookEventName` 必须**严格匹配**当前事件，否则抛 `Hook returned incorrect event name` 错误（`hooks.ts:666-672`）。这是为了防止用户写错 hookEventName 而默默失效。

---

## 13. 实际示例：PreToolUse 鉴权审计

一个真实可工作的 bash command hook：禁止 `rm -rf` 命令：

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/block-dangerous.sh
set -euo pipefail
INPUT="$(cat)"

# 提取待执行命令
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')

# 只对 Bash 工具生效
if [ "$TOOL_NAME" != "Bash" ]; then
    exit 0
fi

# 危险命令模式
if echo "$COMMAND" | grep -qE '(rm\s+-[a-z]*[rf][a-z]*\s+/|:\(\)\s*\{.*\};\s*:|dd\s+if=.*of=/dev/)'; then
    # 阻断：返回 PreToolUse JSON
    cat <<'EOF'
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Refused: command matches dangerous pattern (destructive operation)"
  }
}
EOF
    exit 0
fi

# 允许
cat <<'EOF'
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Pattern check passed"
  }
}
EOF
exit 0
```

### 配置

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {"type": "command", "command": "~/.claude/hooks/block-dangerous.sh", "timeout": 5}
        ]
      }
    ]
  }
}
```

### 行为
- 用户让 Claude 跑 `rm -rf /tmp/foo` → hook 返回 `permissionDecision: "deny"` → 工具被阻止 + 模型收到 `Refused: command matches dangerous pattern` 作为 feedback
- 用户让 Claude 跑 `ls -la` → hook 返回 `allow` → 工具正常执行

### 简化版：直接用顶层 `decision`

```bash
# 简化输出（不需要 hookSpecificOutput）
cat <<'EOF'
{
  "decision": "block",
  "reason": "Cannot run rm -rf"
}
EOF
```

效果一样，但**不如 hookSpecificOutput 精细**——`hookSpecificOutput.permissionDecision` 能精确指定 `allow / deny / ask` 三态，而顶层 `decision` 只有 `approve / block` 两态。

---

## 14. 常见陷阱

### 14.1 ❌ 忘了 stdin 末尾换行

```bash
# 错：read 会因 EOF 提前退出
echo -n "$INPUT_JSON"   # ❌

# 对：stdin 末尾要有 \n
echo "$INPUT_JSON"      # echo 自动加 \n
```

### 14.2 ❌ 把 JSON 写到 stderr

stderr 是诊断通道，不会被 parseHookOutput 解析。

```bash
echo '{"decision": "block"}' >&2   # ❌ 写到 stderr → 等于什么都没做
echo '{"decision": "block"}'       # ✅ stdout
```

### 14.3 ❌ JSON 里有语法错误

`parseHookOutput` 静默将 stdout 当成 plain text 处理；你需要主动检查 `processHookJSONOutput` 返回的 `validationError`。

```bash
# 错：trailing comma
echo '{"decision": "block",}'   # ❌
# 对
echo '{"decision": "block"}'   # ✅
```

### 14.4 ❌ hookEventName 与当前事件不匹配

`processHookJSONOutput:667-672` 会**抛** `Hook returned incorrect event name: expected 'X' but got 'Y'`。这会导致整个 hook outcome: `non_blocking_error`。

```json
// 错：PreToolUse hook 里返回 SessionStart 块
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",  // ❌
    "additionalContext": "..."
  }
}
```

### 14.5 ❌ `continue: false` 不带 `stopReason`

`continue: false` 会阻止主流程继续，但 `stopReason` 是给 Claude 解释为什么不继续的字段。不带的话 Claude 会很困惑。

### 14.6 ❌ PowerShell hook 用了 bash 特性

PowerShell 跳过：
- `windowsPathToPosixPath` 路径转换（用原生 `C:\...`）
- `.sh` 自动 prepend bash
- `CLAUDE_CODE_SHELL_PREFIX` POSIX quote
- `CLAUDE_ENV_FILE`（bash-only）

`.ps1` 里直接用 `$env:FOO = 'bar'` 写 PS env 即可。

### 14.7 ❌ 假设 hook 是同步返回

async hook 行为：
- 配置 `async: true` / `asyncRewake: true` → spawn 后立即 backgrounded（见 `hooks.ts:1078-1112`）
- 首次 stdout 输出 `{"async": true}` → 也会 backgrounded（`:1195-1247`）
- 后面用 `AsyncHookRegistry` 收尾

普通用户**不要**用 async，会把状态管理搞乱。

### 14.8 ❌ 假设 cwd 是项目根

`cwd` 来自 `getCwd()`，可能是 worktree 内路径；当 worktree 删除时 `AsyncLocalStorage` 里仍是旧路径。execCommandHook 校验一次（`hooks.ts:1014-1021`），不可用时 fallback 到 `getOriginalCwd()`。

---

# 第三部分：附录

## 15. 关键文件索引

| 路径 | 作用 |
|------|------|
| `src/utils/hooks.ts:302-329` | `createBaseHookInput` —— 所有 hook 输入的公共底座 |
| `src/utils/hooks.ts:383-398` | `validateHookJson` —— Zod schema 校验 |
| `src/utils/hooks.ts:400-452` | `parseHookOutput` —— stdout 解析（JSON 还是 plain text） |
| `src/utils/hooks.ts:454-488` | `parseHttpHookOutput` —— HTTP hook 用，必须是 JSON |
| `src/utils/hooks.ts:569-` | `processHookJSONOutput` —— JSON 翻译成 HookResult |
| `src/utils/hooks.ts:830-1418` | `execCommandHook` 主实现 |
| `src/utils/hooks.ts:855-857` | 诊断日志仅 once-per-session 事件触发 |
| `src/utils/hooks.ts:873-875` | Shell 选择：hook.shell → DEFAULT_HOOK_SHELL |
| `src/utils/hooks.ts:891-894` | Windows 路径转换（bash vs PowerShell） |
| `src/utils/hooks.ts:905-939` | 命令字符串变量替换 |
| `src/utils/hooks.ts:945-949` | Windows 下 `.sh` 自动 prepend bash |
| `src/utils/hooks.ts:951-958` | `CLAUDE_CODE_SHELL_PREFIX` 处理 |
| `src/utils/hooks.ts:960-1009` | 环境变量构造 |
| `src/utils/hooks.ts:1014-1021` | cwd 校验与 fallback |
| `src/utils/hooks.ts:1024-1067` | spawn（bash vs PowerShell） |
| `src/utils/hooks.ts:1078-1112` | Config-based async hook backgrounding |
| `src/utils/hooks.ts:1085-1091` | async path 提前写 stdin |
| `src/utils/hooks.ts:1156-1193` | `requestPrompt` 协议 |
| `src/utils/hooks.ts:1195-1247` | First-line async detection |
| `src/utils/hooks.ts:1280-1299` | 同步路径写 stdin |
| `src/utils/hooks.ts:1366-1401` | EPIPE / ABORT_ERR / 通用错误处理 |
| `src/utils/hooks/execPromptHook.ts:21-211` | `execPromptHook` 主体 |
| `src/utils/hooks/execAgentHook.ts:36-343` | `execAgentHook` 主体 |
| `src/utils/hooks/execHttpHook.ts:49-58` | `getHttpHookPolicy` —— URL allowlist |
| `src/utils/hooks/execHttpHook.ts:64-68` | `urlMatchesPattern` —— `*` 通配符 |
| `src/utils/hooks/execHttpHook.ts:76-79` | `sanitizeHeaderValue` —— 防 header 注入 |
| `src/utils/hooks/execHttpHook.ts:89-108` | `interpolateEnvVars` —— 允许列表 env 替换 |
| `src/utils/hooks/execHttpHook.ts:123-` | `execHttpHook` 主体 |
| `src/utils/hooks/sessionHooks.ts:15-31` | `FunctionHook` / `FunctionHookCallback` 类型 |
| `src/utils/hooks/sessionHooks.ts:42-46` | `SessionStore` 类型 |
| `src/utils/hooks/sessionHooks.ts:62` | `SessionHooksState = Map` 设计依据 |
| `src/utils/hooks/sessionHooks.ts:93-115` | `addFunctionHook` 程序化注入 |
| `src/utils/hooks/sessionHooks.ts:120-162` | `removeFunctionHook` |
| `src/utils/hooks/sessionHooks.ts:175-216` | `addHookToSession` 内部 helper |
| `src/utils/hooks/sessionHooks.ts:282-293` | `convertToHookMatchers` 过滤 FunctionHook |
| `src/utils/hooks/sessionHooks.ts:437-447` | `clearSessionHooks` |
| `src/utils/hooks/hookHelpers.ts` | `addArgumentsToPrompt` / `createStructuredOutputTool` / `hookResponseSchema` |
| `src/utils/hooks/ssrfGuard.ts` | SSRF 防护 DNS 解析 |
| `src/utils/proxy.ts` | `getProxyUrl` / `shouldBypassProxy` |
| `src/types/hooks.ts` | 所有 `*HookInput` 接口定义 + HookCallback 类型 |