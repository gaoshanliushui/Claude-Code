# toolHooks — 工具钩子链执行器

> 源码位置：`src/services/tools/toolHooks.ts`（~680 行，+ 中文 JSDoc）
> 调用方：`src/services/tools/toolExecution.ts`（6 阶段流水线）、`src/tools/AgentTool/AgentTool.tsx`（间接通过 toolExecution）
> 下游：`src/utils/hooks.ts`（`executePreToolHooks` / `executePostToolHooks` / `executePostToolUseFailureHooks`）、`src/utils/permissions/permissions.ts`（`checkRuleBasedPermissions`）

`toolHooks.ts` 是 Claude Code 工具钩子体系的"胶水层"：它本身不解析 settings.json 也不直接 spawn 钩子进程，而是把 `utils/hooks.ts` 返回的 `AggregatedHookResult` 流，按 type 标签归类成可消费的消息流 / 决策流 / 事件流，再交给 toolExecution 编排到主流程。

文件里有 3 个 async generator + 1 个 async 函数，分别对应 **PreToolUse** / **PostToolUse** / **PostToolUseFailure** 三种钩子事件，外加 **resolveHookPermissionDecision** 用于合并钩子决策与 canUseTool 决策。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 类型契约 | `PostToolUseHooksResult<Output>` 联合类型：普通消息 OR `{ updatedMCPToolOutput }` —— 区分 MCP 工具的输出改写场景 |
| PreToolUse 钩子链 | `runPreToolUseHooks` async generator：返回带 `type` discriminator 的事件流（`message` / `hookPermissionResult` / `hookUpdatedInput` / `preventContinuation` / `stopReason` / `additionalContext` / `stop`），让 toolExecution 按类型分支处理 |
| PostToolUse 钩子链 | `runPostToolUseHooks` async generator：只产出消息流（`MessageUpdateLazy` 或 `{ updatedMCPToolOutput }`），处理 hook_cancelled / hook_blocking_error / hook_stopped_continuation / hook_additional_context 四种附件 |
| PostToolUseFailure 钩子链 | `runPostToolUseFailureHooks` async generator：与 PostToolUse 对称但只产出消息（无 `preventContinuation`，失败后不继续），多接 `isInterrupt` 区分用户中断 |
| 钩子权限决策合并 | `resolveHookPermissionDecision`：合并 `hookPermissionResult` 与 `canUseTool` 的决策；核心不变量：**钩子 allow 不会绕过 settings.json 的 deny/ask 规则** |
| 交互工具穿透 | `requiresUserInteraction` + `interactionSatisfied` 判定：当钩子给 `updatedInput` 时视为已经完成 user interaction（如 headless 包装的 AskUserQuestion），跳过二次弹框 |
| forceDecision 透传 | 当钩子返回 `'ask'` 时，把 hook 的 message 作为 forceDecision 传给 canUseTool，让权限弹框显示钩子的 ask 原因而非默认文案 |
| hook_blocking_error 去重 | JSON `{decision:"block"}` 钩子会 yield 两个结果（`{blockingError}` + `{message: hook_blocking_error attachment}`），本文件用 `!attachment.type === 'hook_blocking_error'` 跳过第二个，避免 UI 显示 block 原因两次（#31301） |
| 单钩子抛错隔离 | 内层 `try/catch` 包住每个 `result` 的处理：单个钩子出错 logError + `hook_error_during_execution` 附件 + 继续下一个；不让一条钩子崩溃整条链 |
| 外层兜底 | 外层 `try/catch` 包住 `executePostToolHooks` / `executePreToolHooks` / `executePostToolUseFailureHooks` 自身抛错（Yields 失败、设置 IO 失败等），不让编排器崩溃 |
| 中断检测 | PreToolUse 每次 yield 后检查 `abortController.signal.aborted`，一旦发现立即 yield `hook_cancelled` 附件 + `stop` 信号让调用方提前 return |
| 钩子遥测 | 三种钩子事件各自有专门的 `tengu_pre_tool_hooks_cancelled` / `tengu_post_tool_hooks_cancelled` / `tengu_post_tool_failure_hooks_cancelled` / `tengu_pre_tool_hook_error` / `tengu_post_tool_hook_error` / `tengu_post_tool_failure_hook_error` 埋点，携带 durationMs、queryChainId、queryDepth、isMcp、mcpServerType |
| 共享调用 | `resolveHookPermissionDecision` 被 `toolExecution.ts`（主 query 循环）和 `REPLTool/toolWrappers.ts`（REPL 内部调用）共享，保证两套调用路径的权限语义完全一致 |

---

## 核心执行流程

### `runPreToolUseHooks`（PreToolUse 钩子链）

`runPreToolUseHooks` 是 async generator；`toolExecution.ts → runPreToolHooksPhase` 用 `for await (const result of runPreToolUseHooks(...))` 消费，按 `type` 分发：

```text
                ┌──────────────────────────────────────────────────┐
                │ runPreToolUseHooks(toolUseContext, tool, ...)    │
                └─────────────────────────┬────────────────────────┘
                                          │
        for await (const result of executePreToolHooks(
            tool.name, toolUseID, processedInput,
            toolUseContext, permissionMode,
            abortController.signal,
            undefined, // timeoutMs - use default
            toolUseContext.requestPrompt,
            tool.getToolUseSummary?.(processedInput),
        ))
                                          │
                                          ▼
        单 result 处理（try/catch 隔离每个钩子的副作用）：
        ├─ result.message → yield { type: 'message', message }
        ├─ result.blockingError (JSON {decision:"block"})
        │  → 转译为 deny hookPermissionResult（包装为 hook_blocking_error 附件
        │    的 denial message）
        ├─ result.preventContinuation → yield 'preventContinuation' + 可选 'stopReason'
        ├─ result.permissionBehavior
        │  ├─ 'allow' → yield hookPermissionResult{behavior:'allow', updatedInput}
        │  ├─ 'ask'   → yield hookPermissionResult{behavior:'ask', message:...}
        │  └─ 'deny'  → yield hookPermissionResult{behavior:..., message:...}
        ├─ result.updatedInput + 无 permissionBehavior → yield hookUpdatedInput
        ├─ result.additionalContexts → yield additionalContext 附件
        └─ abortController.signal.aborted → yield hook_cancelled + stop + return
                                          │
                                          ▼
        抛错 → logError + tengu_pre_tool_hook_error 埋点 + hook_error_during_execution 附件 + stop
```

### `runPostToolUseHooks`（PostToolUse 钩子链）

```text
                ┌──────────────────────────────────────────────────┐
                │ runPostToolUseHooks(toolUseContext, tool, ...)   │
                └─────────────────────────┬────────────────────────┘
                                          │
        let toolOutput = toolResponse
        for await (const result of executePostToolHooks(...))
                                          │
                                          ▼
        单 result 处理（try/catch 隔离）：
        ├─ result.message.attachment.type === 'hook_cancelled'
        │  → tengu_post_tool_hooks_cancelled 埋点
        │  → yield hook_cancelled 附件 + continue
        │
        ├─ result.message 存在（非 hook_blocking_error 类型）
        │  → yield { message: result.message }   （去重：见下方）
        │
        ├─ result.blockingError
        │  → yield hook_blocking_error 附件
        │
        ├─ result.preventContinuation
        │  → yield hook_stopped_continuation 附件 + return
        │
        ├─ result.additionalContexts
        │  → yield hook_additional_context 附件
        │
        └─ result.updatedMCPToolOutput + isMcpTool(tool)
           → toolOutput = 更新后的输出
           → yield { updatedMCPToolOutput: toolOutput }
                                          │
                                          ▼
        抛错 → tengu_post_tool_hook_error 埋点 + hook_error_during_execution 附件
```

**去重逻辑**：JSON `{decision:"block"}` 钩子 yield 两个结果（`{blockingError}` + `{message: hook_blocking_error attachment}`），本文件 `!result.message.attachment.type === 'hook_blocking_error'` 跳过第二个 `yield {message}`，让 `result.blockingError` 路径统一创建附件（避免 UI 显示两次 #31301）。

### `runPostToolUseFailureHooks`（PostToolUseFailure 钩子链）

与 PostToolUse **完全对称**，但有 3 处差异：
- 接收 `error: string`（格式化后的错误）+ `isInterrupt: boolean`（用户中断 vs 真实错误）而非 tool 结果；
- 没有 `updatedMCPToolOutput` 分支（失败后钩子无法改写输出）；
- 没有 `preventContinuation` 分支（失败后不再继续执行）。

### `resolveHookPermissionDecision`（钩子权限合并）

`toolExecution.ts → resolvePermissionOrDeny` 在调用 `canUseTool` 之前先把钩子的 `hookPermissionResult` 喂给这个函数，得到最终决策。

```text
                ┌──────────────────────────────────────────────────┐
                │ resolveHookPermissionDecision(                   │
                │     hookPermissionResult, tool, input, ...)     │
                └─────────────────────────┬────────────────────────┘
                                          │
        ┌─────────────────────────────────┼───────────────────────────────────┐
        │                                 │                                   │
   hookPermissionResult === 'allow'   hookPermissionResult === 'deny'   无 / 'ask'
        │                                 │                                   │
        ▼                                 ▼                                   ▼
  hookInput = hookResult.updatedInput   返回 hookPermissionResult     forceDecision =
       ?? input                         （deny 直接生效）               'ask' 时为 hookPermissionResult
        │                                                            askInput = 'ask' + updatedInput
        │                                                                ?? input
        ▼                                                                    │
  requiresInteraction && !interactionSatisfied                              │
  或 requireCanUseTool                                                      │
        │                                                                    │
        ▼                                                                    ▼
  调 canUseTool（再走一次规则匹配 + 弹框）               调 canUseTool(tool, askInput, ..., forceDecision)
        │
        │ 否则 ↓
        │
  checkRuleBasedPermissions（**核心不变量：钩子 allow 不绕过 deny/ask 规则**）
        │
        ├─ null → interactionSatisfied → 返回 hookPermissionResult（allow）
        │        否则 → log "Hook approved tool use for X, bypassing permission prompt"
        │
        ├─ 'deny' → 返回 ruleCheck（deny rule 覆盖 hook allow）
        │
        └─ 'ask' → 调 canUseTool(tool, hookInput, ...)，即使钩子允许仍弹权限框
```

**核心不变量**（注释明确指出）：钩子返回 `'allow'` 不会绕过 settings.json 中的 deny/ask 规则 —— `checkRuleBasedPermissions` 仍然适用（对应 inc-4788 安全不变量）。

---

## 关键设计取舍

- **PreToolUse 用 type discriminator 而非多条消息流**：因为 PreToolUse 需要同时传递消息（`message`）、权限决策（`hookPermissionResult`）、输入改写（`hookUpdatedInput`）、阻止信号（`preventContinuation` + `stopReason`）、附加上下文（`additionalContext`）、中断信号（`stop`）—— 用 `type` discriminator 比 union of nullable 字段更明确，调用方按 type 分发不会歧义。
- **PostToolUse 失败用消息流而不用 type discriminator**：因为 PostToolUse 只需要传递消息（普通消息 + MCP 输出改写），不需要结构化副作用。`{ updatedMCPToolOutput }` 用单独 union member 区分即可。
- **单钩子抛错隔离用内层 try/catch**：不让一条钩子崩溃整条链 —— 钩子链是用户配置或 plugin 提供的，开发者可能写出 bug，应该优雅降级。
- **外层 try/catch 兜底 executePostToolHooks 自身**：执行器内部迭代可能因 Yields 失败、设置 IO 错误等抛错 —— 编排器不能崩。
- **hook_blocking_error 去重**：JSON `{decision:"block"}` 钩子在 `executeHooks` 里 yield 两个结果；本文件让 `result.blockingError` 路径统一创建附件，跳过 `result.message` 的同类型 yield。注释明确说"exit-code-2 路径只 yield `{blockingError}`，所以不受影响"。
- **hook allow 不绕过规则匹配**：是核心安全不变量 —— 用户配置的 `Bash(rm *)` deny rule 必须能覆盖钩子的 allow 决策。注释明确说 "inc-4788 analog"。
- **`requiresUserInteraction` 穿透**：当钩子提供 `updatedInput` 时视为已经完成 user interaction（如 headless wrapper 收集了 AskUserQuestion 答案）—— 跳过二次弹框，避免重复打扰用户。
- **`forceDecision` 透传**：钩子返回 `'ask'` 时把它的 message 作为 forceDecision 传给 canUseTool，让权限弹框显示钩子的 ask 原因（如"Hook PreToolUse:Bash requires: safety check failed"）而非默认的"Bash wants to run X"。

---

## 调用与被调用全景

```text
主 query 循环 (query.ts)
  ↓ runToolUse(toolUse, assistantMessage, canUseTool, toolUseContext)
    ↓ streamedCheckPermissionsAndCallTool → Stream<MessageUpdateLazy>
       ↓ checkPermissionsAndCallTool（6 阶段流水线）
          │
          ├─ 阶段 3: runPreToolHooksPhase
          │  ├─ for await (const result of runPreToolUseHooks(...))  ← 本文件
          │  │  ├─ { type: 'message', message }                    → resultingMessages.push
          │  │  ├─ { type: 'hookPermissionResult', result }        → 喂给 resolvePermissionOrDeny
          │  │  ├─ { type: 'hookUpdatedInput', input }              → processedInput 替换
          │  │  ├─ { type: 'preventContinuation', true }           → 阻止后续执行
          │  │  ├─ { type: 'stopReason', reason }                  → stop message 用
          │  │  ├─ { type: 'additionalContext', message }          → resultingMessages.push
          │  │  └─ { type: 'stop' }                                → 提前 return
          │  └─ (外部: checkRuleBasedPermissions + canUseTool + resolveHookPermissionDecision ← 本文件)
          │
          ├─ 阶段 4: resolvePermissionOrDeny
          │  ├─ startToolSpan + startToolBlockedOnUserSpan
          │  ├─ resolveHookPermissionDecision(...)                 ← 本文件
          │  ├─ logOTelEvent('tool_decision')
          │  └─ deny: logEvent + tool_result 错误消息
          │
          └─ 阶段 6: executeToolAndCollectResults
             │
             ├─ 成功路径:
             │  ├─ 非 MCP 工具: await addToolResult(toolOutput, mappedToolResultBlock)
             │  │  （先 addResult 再 hooks，因为非 MCP 工具结果不会被 hook 修改）
             │  ├─ for await (const hookResult of runPostToolUseHooks(...))    ← 本文件
             │  │  ├─ { message } (attachment/progress)
             │  │  ├─ { updatedMCPToolOutput } (仅 MCP)
             │  │  └─ yield
             │  └─ MCP 工具: await addToolResult(toolOutput)
             │     （先 hooks 后 addResult，因为 hook 可能通过 updatedMCPToolOutput 改写输出）
             │
             └─ 失败路径:
                └─ for await (const hookResult of runPostToolUseFailureHooks(...))  ← 本文件
                   ├─ { message } (attachment)
                   └─ yield
```

`resolveHookPermissionDecision` 跨调用方共享：

```text
src/services/tools/toolExecution.ts (主 query 循环)
src/tools/REPLTool/toolWrappers.ts (REPL 内部调用)
  └─ resolveHookPermissionDecision(...)  ← 本文件
     ├─ 钩子 allow + requiresInteraction → 调 canUseTool
     ├─ 钩子 allow + 常规 → checkRuleBasedPermissions → 钩子 deny 覆盖 / ask 弹框
     ├─ 钩子 deny → 直接 deny
     └─ 无/钩子 ask → canUseTool(forceDecision)
```

---

## 关联模块速查

| 模块 | 行数 | 职责 |
|------|-----:|------|
| `src/services/tools/toolHooks.ts` | ~680 | 本文件：钩子流聚合 + 权限决策合并 |
| `src/services/tools/toolExecution.ts` | 1984 | 6 阶段流水线的消费者（详见 `11-tool-execution.md`） |
| `src/utils/hooks.ts` | — | `executePreToolHooks` / `executePostToolHooks` / `executePostToolUseFailureHooks` / `getPreToolHookBlockingMessage` / `executePermissionDeniedHooks` 的具体实现 |
| `src/utils/permissions/permissions.ts` | — | `checkRuleBasedPermissions`（核心不变量：钩子 allow 不绕过 deny/ask 规则） |
| `src/utils/permissions/PermissionResult.ts` | — | `PermissionResult` / `PermissionDecisionReason` / `getRuleBehaviorDescription` |
| `src/utils/attachments.ts` | — | `createAttachmentMessage`（构造 `hook_cancelled` / `hook_blocking_error` / `hook_additional_context` / `hook_stopped_continuation` / `hook_error_during_execution` 附件） |
| `src/utils/toolErrors.ts` | — | `formatError`（钩子抛错时格式化错误内容给模型） |
| `src/hooks/useCanUseTool.ts` | — | `CanUseToolFn` 类型（`resolveHookPermissionDecision` 调用的真实权限函数） |
| `src/types/hooks.ts` | — | `HookProgress` 类型 |
| `src/types/message.ts` | — | `AssistantMessage` / `AttachmentMessage` / `ProgressMessage` |
| `src/services/analytics/index.ts` | — | `logEvent` + `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型 |
| `src/services/analytics/metadata.ts` | — | `sanitizeToolNameForAnalytics`（埋点时清洗工具名） |
| `src/services/mcp/utils.ts` | — | `isMcpTool`（仅 MCP 工具允许 `updatedMCPToolOutput`） |
| `src/utils/log.ts` | — | `logError`（外层兜底错误日志） |
| `src/utils/debug.ts` | — | `logForDebugging`（权限决策路径调试日志） |
| `src/Tool.ts` | — | `Tool` 接口（hook 调用需要的 `getToolUseSummary` / `requiresUserInteraction` / `name`） |

---

## 重要不变量

- **钩子 allow 不绕过 settings.json 规则**：核心安全不变量（`checkRuleBasedPermissions` 在 hook allow 之后必须仍然适用；对应 inc-4788）。用户配置的 `Bash(rm *)` deny 规则必须能覆盖钩子的 allow 决策。
- **`requiresUserInteraction` 穿透判定**：当钩子提供 `updatedInput` 时视为已经完成 user interaction（headless wrapper 收集了答案），跳过二次弹框；否则仍走 `canUseTool`。
- **`forceDecision` 透传**：钩子返回 `'ask'` 时把它的 message 作为 forceDecision 传给 canUseTool，让权限弹框显示钩子的 ask 原因。
- **`hook_blocking_error` 去重**：JSON `{decision:"block"}` 钩子 yield 两个结果；本文件让 `result.blockingError` 路径统一创建附件，跳过 `result.message` 的同类型 yield（注释明确说 #31301 修这个 bug）。
- **单钩子抛错隔离**：内层 `try/catch` 包住每个 `result` 处理 + logError + `hook_error_during_execution` 附件 + 继续下一个；不让一条钩子崩溃整条链。
- **外层 catch 兜底 executeHooks 自身**：执行器内部迭代可能因 Yields 失败抛错，编排器不能崩。
- **`updatedMCPToolOutput` 仅 MCP 工具生效**：PostToolUse 路径里 `if (result.updatedMCPToolOutput && isMcpTool(tool))` —— 非 MCP 工具即使钩子改了 MCP 输出也忽略。
- **PreToolUse 中断检测在每次 yield 后**：一旦 `abortController.signal.aborted`，立即 yield `hook_cancelled` + `stop` + return，让调用方提前 return 整个执行。
- **`resolveHookPermissionDecision` 跨调用方共享**：被 `toolExecution.ts`（主 query 循环）和 `REPLTool/toolWrappers.ts`（REPL 内部调用）共享，保证两套调用路径的权限语义完全一致。
- **PostToolUseFailure 不支持 `updatedMCPToolOutput`**：失败后钩子无法改写输出，注释明确写"没有 preventContinuation 分支（失败后不会再继续执行）"。
- **遥测事件专门命名**：`tengu_pre_tool_hooks_cancelled` / `tengu_post_tool_hooks_cancelled` / `tengu_post_tool_failure_hooks_cancelled` / `tengu_pre_tool_hook_error` / `tengu_post_tool_hook_error` / `tengu_post_tool_failure_hook_error` —— BQ 聚合时按 hook 事件 × 异常类型精确分流。
