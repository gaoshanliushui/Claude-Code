# toolExecution — 工具调用编排中心

> 源码位置：`src/services/tools/toolExecution.ts`（1984 行）
> 上游：`src/query.ts` 主循环 → `runToolUse`
> 下游：`src/services/tools/toolHooks.ts`（钩子链）、`src/Tool.ts`（Tool 接口）、`src/tools/*/`（具体工具实现）、`src/utils/telemetry/sessionTracing.ts`（OTel span）

`runToolUse` 是 Claude Code 主循环把模型输出的 `tool_use` 块变成实际副作用的唯一入口。它把"工具调用"这件事拆成 6 个清晰阶段（解析 → 校验 → 预处理 → 钩子 → 权限 → 执行），让权限、遥测、OTel、钩子链、错误恢复等横切关注点都收敛到这一处。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 工具解析与别名回退 | `findToolByName` 先在 `options.tools` 查；查不到时回退到 `getAllBaseTools()`，且只接受命中 `tool.aliases`（不是 primary name）的别名调用，让老 transcript（如旧 `KillShell` → 现 `TaskStop`）继续可读 |
| MCP 元信息提取 | `findMcpServerConnection` / `getMcpServerType` / `getMcpServerBaseUrlFromToolName` 把 `mcp__server__tool` 名字解析成 server 类型与 base URL，注入所有遥测字段 |
| 输入校验 | 两层：Zod schema 类型校验 → `tool.validateInput` 业务校验；deferred 工具未下发 schema 时附 `buildSchemaNotSentHint` 引导模型先 `ToolSearch select:<name>` 再 retry |
| 预处理 | `prepareToolInputForHooks`：Bash 提前启动 speculative classifier（与钩子/权限并行）；防御性剥离 `_simulatedSedEdit` 内部字段；浅拷贝上 `backfillObservableInput`（如 `notebook_path` 派生扩展名）回填给钩子/canUseTool，原始 input 留给 `tool.call()` 以稳定 transcript/VCR 哈希 |
| PreToolUse 钩子链 | `runPreToolHooksPhase` 聚合 message / additionalContext / hookPermissionResult / hookUpdatedInput / preventContinuation / stopReason / stop；超过 500ms 附计时摘要（ant-only） |
| 权限决策与拒绝 | `resolvePermissionOrDeny` 合并 hook 与 canUseTool 决策，OTel `tool_decision` 事件（按 `decisionReasonToOTelSource` 映射到 `config` / `hook` / `user_*`），auto 模式 + classifier 拒绝时跑 `PermissionDenied` 钩子判断是否允许 retry |
| 工具执行 | `executeToolAndCollectResults`：调 `tool.call()`、记录 OTel `tool.output`（含 file_path/diff/command/output）、`tengu_tool_use_success` 埋点（durationMs、preToolHookDurationMs、toolResultSizeBytes、fileExtension、git_commit_id）、非 MCP 先 addResult 再 hooks、MCP 先 hooks 后 addResult（hook 可能修改输出）、捕获 `structured_output`、applying `tool.mapToolResultToToolResultBlockParam`、run `PostToolUse` 与 `PostToolUseFailure` 钩子 |
| 进度聚合 | `streamedCheckPermissionsAndCallTool` 把 progress 事件与最终结果压进同一个 `Stream<MessageUpdateLazy>`（`enqueue` 进度 / `enqueue` 结果 / `error` / `done`），调用方用 `for await` 统一消费 |
| 错误恢复 | 顶层 `try/catch` 把任何 thrown error 转成 `<tool_use_error>` 用户消息；已 abort 时（`toolUseContext.abortController.signal.aborted`）上抛 `tengu_tool_use_cancelled` + `createToolResultStopMessage` + `withMemoryCorrectionHint(CANCEL_MESSAGE)`；MCP `McpAuthError` 时把 client 状态切到 `needs-auth` |
| 取消与中断 | `AbortError` 不走错误埋点路径（避免污染"工具出错"统计）；最终输出消息挂 `mcpMeta`（仅当 `!toolUseContext.agentId`）；agent context 里清掉 `mcpMeta` 和 `toolUseResult` 以减少 sidechain 噪音 |
| 死代码消除 | `feature('TRANSCRIPT_CLASSIFIER')` 守卫 auto-mode PermissionDenied retry；`process.env.USER_TYPE === 'ant'` 守卫钩子计时摘要、ANT-ONLY 提示；遥测字段全部走 `...(value && { key: value })` 条件 spread |

---

## 核心执行流程

`runToolUse(toolUse, assistantMessage, canUseTool, toolUseContext)` 是入口；它把 `streamedCheckPermissionsAndCallTool(...)` 产生的 `Stream<MessageUpdateLazy>` 通过 `for await` yield 给上层。完整的 6 阶段流水线如下：

```text
                ┌────────────────────────────────────────────────┐
                │ runToolUse(toolUse, assistantMessage, ...)    │
                └─────────────────────┬──────────────────────────┘
                                      │
                1. 工具解析 + 别名回退 + MCP 元信息提取
                   ├─ findToolByName(options.tools, toolName)
                   ├─ 找不到则查 getAllBaseTools() 仅当 tool.aliases.includes(name)
                   ├─ getMcpServerType + getMcpServerBaseUrl
                   └─ 找不到则 yield <tool_use_error> 消息后 return
                                      │
                                      ▼
                2. abort 检查
                   └─ signal.aborted? → tengu_tool_use_cancelled 埋点
                                       + createToolResultStopMessage + CANCEL_MESSAGE
                                      │
                                      ▼
        ┌──────────────────────────────────────────────────────────┐
        │ streamedCheckPermissionsAndCallTool (Stream 包装)       │
        │ 内部委托 checkPermissionsAndCallTool（5 阶段流水线）     │
        └─────────────────────────┬────────────────────────────────┘
                                  │
                                  ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 阶段 1: validateToolInput                                     │
   │   Zod safeParse → 失败: tengu_deferred_tool_schema_not_sent     │
   │     + InputValidationError 消息                                │
   │   tool.validateInput → 失败: 业务错误消息                      │
   └─────────────────────────┬──────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 阶段 2: prepareToolInputForHooks                              │
   │   Bash 启动 startSpeculativeClassifierCheck（并行）           │
   │   剥离 _simulatedSedEdit 内部字段                             │
   │   浅拷贝上 backfillObservableInput（给钩子/canUseTool 用）     │
   │   返回 { processedInput, callInput, backfilledClone }         │
   └─────────────────────────┬──────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 阶段 3: runPreToolHooksPhase                                  │
   │   聚合 runPreToolUseHooks 结果：                              │
   │     - message / additionalContext → resultingMessages        │
   │     - hookPermissionResult（allow/deny/ask）                  │
   │     - hookUpdatedInput → 替换 processedInput                  │
   │     - preventContinuation / stopReason                        │
   │     - stop → 立即 push stop message，earlyReturn              │
   │   慢路径 (>2s) 写 logForDebugging                              │
   │   ant-only (>500ms) push StopHookSummary 消息                 │
   └─────────────────────────┬──────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 阶段 4: resolvePermissionOrDeny                               │
   │   startToolSpan(attributes) + startToolBlockedOnUserSpan       │
   │   resolveHookPermissionDecision 合并 hook 决策和 canUseTool    │
   │     - hook allow + requiresInteraction → 仍调 canUseTool      │
   │     - hook allow + 无 requireCanUseTool → 走 rule-based 检查  │
   │     - 冲突时 deny rule 覆盖 hook allow                        │
   │   非 ask 决策 → OTel tool_decision + 代码编辑工具加 decision   │
   │   PermissionRequest hook 决策 → push hook_permission_decision  │
   │   allow:  push allow 埋点 + 返回决策                          │
   │   deny:                                                       │
   │     - push 拒绝埋点                                           │
   │     - push tool_result 错误消息（含 rejectContentBlocks）     │
   │     - auto + classifier 拒绝时执行 PermissionDenied hooks     │
   │       若 hookSaysRetry → push isMeta retry 提示              │
   └─────────────────────────┬──────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 阶段 5: prepareExecutionTelemetry                             │
   │   1. 应用 permissionDecision.updatedInput 覆盖 processedInput  │
   │   2. extractToolInputForTelemetry(processedInput)             │
   │   3. isToolDetailsLoggingEnabled?  提取 bash/MCP/skill 详情   │
   │      → toolParameters {bash_command, full_command, timeout,   │
   │        description, dangerouslyDisableSandbox, mcp_*, ...}    │
   │   4. decisionInfo ← toolUseContext.toolDecisions.get(toolUseID)│
   │   5. endToolBlockedOnUserSpan + startToolExecutionSpan        │
   │   6. startSessionActivity('tool_exec')                        │
   │   7. resolvedCallInput 解析：                                  │
   │      - backfilledClone + file_path 与扩展值相同 → 保持原路径  │
   │      - 否则 → processedInput 优先                             │
   └─────────────────────────┬──────────────────────────────────────┘
                              │
                              ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 阶段 6: executeToolAndCollectResults                          │
   │   try:                                                        │
   │     6.1 tool.call(resolvedCallInput, ...)                     │
   │     6.2 addToToolDuration(durationMs)                         │
   │     6.3 OTel addToolContentEvent（file_path/diff/command/...）│
   │     6.4 捕获 structured_output → push attachment              │
   │     6.5 endToolExecutionSpan({success}) + endToolSpan         │
   │     6.6 tool.mapToolResultToToolResultBlockParam             │
   │     6.7 提取 fileExtension (Read/Edit/Write/Notebook/Bash)    │
   │     6.8 tengu_tool_use_success 埋点（durationMs/SizeBytes/...）│
   │     6.9 git commit 提取 git_commit_id                          │
   │     6.10 OTel tool_result 事件（成功）                         │
   │     6.11 准备 addToolResult closure（处理 contentBlocks/...）  │
   │     6.12 非 MCP: addToolResult（先结果再 hooks）              │
   │     6.13 runPostToolUseHooks：                                 │
   │          - updatedMCPToolOutput → 改 toolOutput（MCP）         │
   │          - 其他 → resultingMessages.push(hookResult)          │
   │     6.14 MCP: addToolResult(toolOutput)（hooks 后结果）       │
   │     6.15 ant-only hook 计时摘要                                │
   │     6.16 result.newMessages → push                            │
   │     6.17 shouldPreventContinuation → push hook_stopped_...   │
   │   catch (error):                                              │
   │     - McpAuthError → 客户端状态切 needs-auth                  │
   │     - 非 AbortError → tengu_tool_use_error 埋点              │
   │       + OTel tool_result(success='false')                    │
   │     - formatError(content)                                    │
   │     - runPostToolUseFailureHooks → 合并到返回                 │
   │   finally:                                                    │
   │     - stopSessionActivity('tool_exec')                        │
   │     - toolDecisions.delete(toolUseID)                         │
   └──────────────────────────────────────────────────────────────┘
```

### 进度与结果合并（`Stream<MessageUpdateLazy>`）

`streamedCheckPermissionsAndCallTool` 用一个自建的 `Stream` 把异步进度和最终结果压进同一个 `AsyncIterable`：

```text
工具 progress 事件（onToolProgress 回调）
   ↓ stream.enqueue(createProgressMessage(...))
Stream<MessageUpdateLazy>
   ↓ for await 消费
结果 messages（最终 tool_result / error / hook 消息 / newMessages / ...）
   ↓ stream.enqueue(...) 直至 stream.done()
```

调用方 `runToolUse` 用 `for await (const update of stream)` 逐个 yield —— 上层（主 query 循环）把 progress 消息直接发给 UI，把结果消息灌进 assistant 的工具结果块。

### 取消与中止

- **进入前已取消**（`signal.aborted`）→ `tengu_tool_use_cancelled` 埋点 + `createToolResultStopMessage` + `CANCEL_MESSAGE`（带 `withMemoryCorrectionHint` 提示模型"操作被取消"），不进入 6 阶段流水线
- **执行中取消**（`tool.call()` 抛 `AbortError`）→ catch 分支识别 `AbortError`，**跳过** `tengu_tool_use_error` 埋点（避免污染错误统计），但仍跑 `PostToolUseFailure` 钩子
- **`McpAuthError`** → `setAppState` 把 client 切到 `needs-auth`，让 UI 提示用户重新认证

---

## 关键设计取舍

- **6 阶段流水线而非一坨**：每阶段用 `return` 早出（`validationResult.kind === 'error'`、`preHookResult.earlyReturn`、`permResult.denied`），错误就近处理，调用方不需要 `if/else` 嵌套。每阶段都返回结构化结果（messages、permissionDecision、resolvedCallInput 等），让单元测试可以独立注入。
- **processedInput vs callInput 分离**：`processedInput` 给钩子/`canUseTool`（含 backfill），`callInput` 给 `tool.call()`（原始模型输入）。`file_path` 例子：bash backfill 把 `file_path` 扩展成绝对路径，钩子/权限需要绝对路径做检查，但 `tool.call()` 用原始路径保持 transcript/VCR 哈希稳定 —— 文件编辑工具的 round-trip 测试才能稳定。
- **Bash speculative classifier**：`startSpeculativeClassifierCheck` 在 Bash 命令离开流水线之前并行启动分类器校验，避免它和 PreToolUse 钩子、canUseTool 串行。这对 auto 模式下的命令延迟优化非常关键（注释见 778-791 行）。
- **MCP vs 非 MCP 顺序差异**：
  - 非 MCP：先 `addToolResult`，再跑 PostToolUse 钩子 —— 因为非 MCP 工具结果不会被 hook 修改
  - MCP：先跑 PostToolUse 钩子，再 `addToolResult(toolOutput)` —— 因为 hook 可能通过 `updatedMCPToolOutput` 字段改写 MCP 输出
  - 这种顺序差异在 1651-1718 行明确分两段处理
- **OTel `tool_decision` 事件源词汇表**：`decisionReasonToOTelSource` 严格按 `permissionLogging.ts:81` 的 5 个值（`config` / `hook` / `user_permanent` / `user_temporary` / `user_reject`）映射，让交互式路径（`/permissions` UI）与非交互式路径（tool_decision 事件）的语义保持一致。`permissionPromptTool` 走 SDK host 的 `decisionClassification`（host 知道 once/always/cache hit 的真实情况）；无 classification 时保守回退（allow → user_temporary，deny → user_reject）。
- **deferred 工具的 schema 缺失提示**：当模型用 `defer_loading` 的工具时，claude.ts 在 dispatch 时会做 schema-filter scan，扫不到的工具 schema 不会下发到 API；模型写出的 typed 参数会变成 string，Zod 解析失败。`buildSchemaNotSentHint` 在解析失败时附上"先 `ToolSearch select:<name>` 再 retry"的提示，让模型自纠。注释里说"occasional misfires acceptable" —— 偶尔误指（Haiku、tst-auto 阈值下）只多一次往返。
- **transcript 哈希稳定**：`resolvedCallInput` 在 `backfilledClone` 的 `file_path` 与原 `callInput.file_path` 不一致时优先用原 `file_path`（保持 transcript 字节级一致）。`permissionDecision.updatedInput` 仅作用于 processedInput 链路，不污染 callInput。
- **agent context 抑制**：子代理内（`toolUseContext.agentId`）的 tool result 不带 `toolUseResult` 字符串 + 不带 `mcpMeta`，因为子代理自己看 transcript、不会写磁盘。

---

## 调用与被调用全景

```text
主 query 循环 (query.ts)
  ↓ runToolUse(toolUse, assistantMessage, canUseTool, toolUseContext)
     ├─ 工具解析: findToolByName(options.tools) + getAllBaseTools() 别名回退
     ├─ MCP 解析: getMcpServerType + getMcpServerBaseUrl
     ├─ abort 检查: 短路返回
     └─ streamedCheckPermissionsAndCallTool
        ↓ Stream<MessageUpdateLazy>
        ├─ 进度: onToolProgress → stream.enqueue(createProgressMessage)
        └─ 结果: 6 阶段流水线
           1. validateToolInput
              ├─ tool.inputSchema.safeParse
              ├─ buildSchemaNotSentHint (deferred 工具)
              └─ tool.validateInput
           2. prepareToolInputForHooks
              ├─ startSpeculativeClassifierCheck (Bash)
              ├─ 剥离 _simulatedSedEdit
              └─ tool.backfillObservableInput
           3. runPreToolHooksPhase
              └─ runPreToolUseHooks → toolHooks.ts
           4. resolvePermissionOrDeny
              ├─ startToolSpan + startToolBlockedOnUserSpan
              ├─ resolveHookPermissionDecision
              │  └─ canUseTool → checkRuleBasedPermissions / 弹权限框
              ├─ logOTelEvent('tool_decision')
              └─ 拒绝: executePermissionDeniedHooks (auto+classifier)
           5. prepareExecutionTelemetry
              ├─ extractToolInputForTelemetry
              ├─ endToolBlockedOnUserSpan + startToolExecutionSpan
              └─ startSessionActivity('tool_exec')
           6. executeToolAndCollectResults
              ├─ tool.call(resolvedCallInput, ...)
              ├─ addToolContentEvent (OTel tool.output)
              ├─ endToolExecutionSpan + endToolSpan
              ├─ tool.mapToolResultToToolResultBlockParam
              ├─ logEvent('tengu_tool_use_success')
              ├─ parseGitCommitId (Bash/PowerShell git commit)
              ├─ logOTelEvent('tool_result')
              ├─ addToolResult (processToolResultBlock)
              ├─ runPostToolUseHooks (MCP 优先以更新输出)
              └─ catch: runPostToolUseFailureHooks
```

---

## 关联模块速查

| 模块 | 行数 | 职责 |
|------|-----:|------|
| `src/services/tools/toolExecution.ts` | 1984 | 本文件：6 阶段流水线、Stream 包装、MCP 元信息、遥测与 OTel 编排 |
| `src/services/tools/toolHooks.ts` | — | `runPreToolUseHooks` / `runPostToolUseHooks` / `runPostToolUseFailureHooks` / `resolveHookPermissionDecision` / `executePermissionDeniedHooks` 的具体实现；定义 `PostToolUseHooksResult`（区分 message vs `updatedMCPToolOutput`） |
| `src/Tool.ts` | — | `Tool` 接口（call / inputSchema / validateInput / backfillObservableInput / mapToolResultToToolResultBlockParam 等）、`buildTool` 工厂（带 7 个默认实现）、`findToolByName`（用 `toolMatchesName` 匹配主名和 alias） |
| `src/utils/telemetry/sessionTracing.ts` | — | OTel span 工具：`startToolSpan` / `startToolBlockedOnUserSpan` / `startToolExecutionSpan` / `endToolSpan` / `endToolBlockedOnUserSpan` / `endToolExecutionSpan` / `addToolContentEvent` |
| `src/utils/telemetry/events.ts` | — | `logOTelEvent`（`tool_decision` / `tool_result` 事件） |
| `src/services/analytics/metadata.ts` | — | `sanitizeToolNameForAnalytics` / `extractToolInputForTelemetry` / `mcpToolDetailsForAnalytics` / `extractSkillName` / `extractMcpToolDetails` / `getFileExtensionForAnalytics` / `getFileExtensionsFromBashCommand` / `isToolDetailsLoggingEnabled` |
| `src/utils/toolResultStorage.ts` | — | `processToolResultBlock`（大输出 spill to disk / 截断 / 行数限制）、`processPreMappedToolResultBlock`（hook 已 mapping 的场景） |
| `src/utils/toolErrors.ts` | — | `formatError`（统一错误格式）/ `formatZodValidationError`（Zod 错误 → 用户可读消息） |
| `src/hooks/useCanUseTool.ts` | — | `CanUseToolFn` 类型 + 真实 `canUseTool` 实现（弹权限框 / rule-based 检查） |
| `src/hooks/toolPermission/permissionLogging.ts` | — | `isCodeEditingTool` / `buildCodeEditToolAttributes`（代码编辑工具的 OTel 属性扩展） |
| `src/utils/toolSearch.ts` | — | `isToolSearchEnabledOptimistic` / `isToolSearchToolAvailable` / `extractDiscoveredToolNames`（用于 `buildSchemaNotSentHint`） |
| `src/utils/permissions/PermissionResult.ts` | — | `PermissionResult` / `PermissionDecisionReason` 类型 |
| `src/utils/sessionActivity.ts` | — | `startSessionActivity` / `stopSessionActivity`（用于工具执行期间的 session 活跃追踪） |
| `src/utils/messages.ts` | — | `createUserMessage` / `createProgressMessage` / `createToolResultStopMessage` / `createStopHookSummaryMessage` / `withMemoryCorrectionHint` |
| `src/utils/errors.ts` | — | `AbortError` / `ShellError` / `TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` / `getErrnoCode`（fs 错误码提取） |
| `src/services/mcp/client.ts` | — | `McpAuthError` / `McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`（MCP 错误类型） |
| `src/services/mcp/utils.ts` | — | `isMcpTool` / `getMcpServerScopeFromToolName` / `getLoggingSafeMcpBaseUrl` |
| `src/services/mcp/normalization.ts` | — | `normalizeNameForMCP`（MCP server 名归一化） |
| `src/tools/BashTool/bashPermissions.ts` | — | `startSpeculativeClassifierCheck`（Bash 命令分类器） |
| `src/tools/ToolSearchTool/prompt.ts` | — | `TOOL_SEARCH_TOOL_NAME` / `isDeferredTool` |
| `src/types/hooks.ts` | — | `HookResult` / `AggregatedHookResult` / `HookProgress` / `PermissionRequestResult` 类型定义 |
| `src/types/message.ts` | — | `Message` / `UserMessage` / `ProgressMessage` / `AttachmentMessage` / `StopHookInfo` |
| `src/utils/hooks.ts` | — | `executePermissionDeniedHooks`（auto 模式 classifier 拒绝后的 retry 决策） |
| `src/tools/shared/gitOperationTracking.ts` | — | `parseGitCommitId`（从 `git commit` 输出提取 commit hash 丰富遥测） |

---

## 重要不变量

- **工具解析顺序**：先 `options.tools`（用户当前可见的工具），再回退 `getAllBaseTools()`（基础工具池）以命中 alias。alias 回退只接受 `tool.aliases.includes(toolName)` —— **不**接受 primary name 匹配，否则会让老工具的别名遮盖新工具。
- **Stream 单次消费**：`Stream<MessageUpdateLazy>` 在 `streamedCheckPermissionsAndCallTool` 内创建；进度回调（`onToolProgress`）和最终结果（`then/catch/finally`）都向它 enqueue。`stream.done()` 触发后消费者自然终止。
- **OTel span 配对**：
  - `startToolSpan` ↔ `endToolSpan`（含 success/failure 的输出）
  - `startToolBlockedOnUserSpan` ↔ `endToolBlockedOnUserSpan`（decision + source）
  - `startToolExecutionSpan` ↔ `endToolExecutionSpan`（success/error）
  - 三组 span 在 `executeToolAndCollectResults` 的 try/catch/finally 中完整配对
- **MCP auth 错误自动恢复**：`McpAuthError` 时 `setAppState` 把 client 切到 `needs-auth` —— UI 会自动弹出重新认证对话框，无需主循环介入
- **AbortError 与错误的区分**：`AbortError` 不写 `tengu_tool_use_error` 埋点（避免取消被统计为错误），但仍跑 `PostToolUseFailure` 钩子让 hook 能感知
- **transcript 字节级稳定**：`resolvedCallInput.file_path` 优先保留 `callInput.file_path`（原模型输入）—— 同一次同一条消息的 transcript/VCR 哈希不会因 backfill 改变
- **agent context 抑制规则**：`toolUseContext.agentId` 存在时（子代理内），`toolUseResult` 字符串不写入 user message、`mcpMeta` 也不透传 —— 因为子代理自己看 transcript、不写磁盘
- **MCP vs 非 MCP hook 顺序差异**：MCP 工具必须先跑 PostToolUse 钩子再 addResult（hook 可改 `updatedMCPToolOutput`）；非 MCP 工具先 addResult 再跑钩子（hook 不会改结果）
- **deferred 工具 retry 提示**：仅在 `isToolSearchEnabledOptimistic` + `isToolSearchToolAvailable` + `isDeferredTool` 都满足时附 `buildSchemaNotSentHint` —— 避免指向不可调用的 `ToolSearch`（注释见 583-588 行）
- **死代码消除**：`feature('TRANSCRIPT_CLASSIFIER')` 守卫 auto-mode PermissionDenied retry；`process.env.USER_TYPE === 'ant'` 守卫 hook 计时摘要、ANT-ONLY 提示
