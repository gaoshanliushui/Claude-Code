# hooks.ts — 钩子系统核心实现

> 源码位置：`src/utils/hooks.ts`（~5200 行，+ 中文 JSDoc）
> 调用方：`src/services/tools/toolHooks.ts`（工具钩子编排）、`src/services/tools/toolExecution.ts`（6 阶段流水线间接调用）、`src/components/messages/*`（UI 进度展示）、`src/commands/*`（SessionStart / SessionEnd 等）
> 下游：`src/utils/hooks/sessionHooks.ts`（session-scoped hooks）、`src/utils/hooks/execPromptHook.ts`（prompt 钩子）、`src/utils/hooks/execAgentHook.ts`（agent 钩子）、`src/utils/hooks/execHttpHook.ts`（HTTP 钩子）、`src/utils/hooks/AsyncHookRegistry.ts`（async 钩子注册表）

`hooks.ts` 是 Claude Code 钩子系统的"执行核心"。它定义了 30+ 种 `execute*Hooks` 公共 API（每个对应一种 hook event），并通过 **4 种执行器**（命令、prompt、agent、HTTP）+ **1 个内嵌 callback 执行器**（`executeFunctionHook`）+ **1 个 callback 钩子执行器**（`executeHookCallback`）来运行各种类型的钩子。所有公开 API 最终都委托给 `executeHooks`（REPL 内的 async generator）或 `executeHooksOutsideREPL`（REPL 之外的 Promise 包装）。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 钩子类型契约 | `HookResult`（单条结果）/ `AggregatedHookResult`（聚合结果）/ `HookBlockingError` / `ElicitationResponse`（从 MCP SDK 重新导出）—— 5 大类型统一定义 |
| 30+ 公开 execute*Hooks API | PreToolUse / PostToolUse / PostToolUseFailure / PermissionDenied / Notification / StopFailure / Stop / TeammateIdle / TaskCreated / TaskCompleted / UserPromptSubmit / SessionStart / Setup / SubagentStart / PreCompact / PostCompact / SessionEnd / PermissionRequest / ConfigChange / CwdChanged / FileChanged / InstructionsLoaded / Elicitation / ElicitationResult / StatusLine / FileSuggestion / WorktreeCreate / WorktreeRemove |
| 4 种执行器 | `execCommandHook`（shell 命令）/ `execPromptHook`（prompt 注入 LLM）/ `execAgentHook`（forked sub-agent）/ `execHttpHook`（HTTP POST） |
| 1 个 callback 执行器 | `executeFunctionHook`（in-process 内存回调，仅 session-scoped 内部用） |
| 1 个 programmatic hook 执行器 | `executeHookCallback`（编程式注册的回调，如 SDK 注入） |
| 核心 dispatcher | `executeHooks`（async generator，被 toolExecution 消费）+ `executeHooksOutsideREPL`（Promise，返回 result[] 给 SessionEnd 等） |
| 钩子匹配 | `getMatchingHooks` 按 hook event 提取 matchQuery（tool_name / agent_type / source 等），调 `hooksConfigManager`；`hasHookForEvent` 是 hot-path 快路径检查 |
| 钩子输入构造 | `createBaseHookInput` 构造所有钩子共用的 base（session_id / transcript_path / cwd / permission_mode / agent_id / agent_type） |
| 输出解析 | `parseHookOutput`（命令钩子 stdout：纯文本 or JSON）/ `parseHttpHookOutput`（HTTP 必须 JSON）/ `parseElicitationHookOutput`（MCP elicitation）/ `validateHookJson`（Zod schema）/ `processHookJSONOutput`（归一化为 HookResult 字段） |
| 异步钩子 | `executeInBackground` 转 shell 为后台；`registerPendingAsyncHook` 进 AsyncHookRegistry；asyncRewake 模式绕过 registry 直接 enqueue task-notification |
| 安全门控 | `shouldSkipHookDueToTrust`（交互模式未信任工作区跳过）/ `shouldDisableAllHooksIncludingManaged`（managed policy 全局禁用）/ `CLAUDE_CODE_SIMPLE` 环境变量（极简模式禁用） |
| 超时控制 | `TOOL_HOOK_EXECUTION_TIMEOUT_MS` 默认 10 分钟；`SESSION_END_HOOK_TIMEOUT_MS_DEFAULT` 默认 1500ms；`getSessionEndHookTimeoutMs` 支持 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 环境变量覆盖 |
| Plugin / Skill 上下文 | `MatchedHook` 类型含 `pluginRoot` / `pluginId` / `skillRoot` / `hookSource`；`isInternalHook` 区分内部 callback vs 用户配置；`getPluginHookCounts` / `getHookTypeCounts` 用于遥测 |
| 路径转换 | Windows bash 路径必须 POSIX 化（`windowsPathToPosixPath` 纯 JS regex 转换，memoized LRU-500）；PowerShell 跳过该步骤 |
| Shell 选择 | `hook.shell ?? DEFAULT_HOOK_SHELL`（Phase 2 计划接 `settings.defaultShell` 兜底，暂未实装） |
| 环境变量注入 | `CLAUDE_PROJECT_DIR`（稳定项目根）/ `CLAUDE_PLUGIN_ROOT` / `CLAUDE_SKILL_DIR` / `CLAUDE_SESSION_ID` + session 环境缓存（`getHookEnvFilePath` 写文件再 source） |
| 变量替换 | 先 `${CLAUDE_PLUGIN_ROOT}` / `${user_config.X}`，再把整个命令交给 shell（顺序匹配 MCP/LSP —— 用户配置中含 `${CLAUDE_PLUGIN_ROOT}` 字面量视为透明文本） |
| 诊断日志 | 仅 once-per-session 事件（SessionStart / Setup / SessionEnd）开启 —— 控制 diag_log 量；started/completed 在 try/finally 内，避免 setup-path 抛错孤立 started marker |
| 并行执行 | `all([...])` 并行 spawn 每个钩子，第一个有副作用的 yield 触发对应分支；preventContinuation 短路后续 |
| 进度预产 | 每个钩子执行前 yield `hook_progress` 消息（让 UI 在钩子运行前就能显示"正在跑 hook: Bash PreToolUse"） |
| OTel 集成 | `isBetaTracingEnabled` 时构造 hook_definitions JSON + `logOTelEvent('hook_execution_start' / 'hook_execution_end')` + `startHookSpan` / `endHookSpan` |
| 统计 | `hook_duration_ms` + `addToTurnHookDuration` + `tengu_repl_hook_finished` 埋点（success/blocking/nonBlockingError/cancelled 计数） |
| 快路径 | 所有钩子都是 internal callback 时同步调用 + 埋点，绕过 span/progress/abortSignal/processHookJSONOutput/resultLoop —— 实测 6.01µs → 1.8µs（-70%） |

---

## 核心执行流程

### 4 种执行器 × N 种钩子事件

```text
钩子系统入口（外部）
  │
  ├─ toolExecution.ts → runPostToolUseHooks (PostToolUse)
  ├─ toolExecution.ts → runPreToolUseHooks (PreToolUse)
  ├─ toolExecution.ts → runPostToolUseFailureHooks (PostToolUseFailure)
  ├─ toolExecution.ts → executePermissionDeniedHooks (PermissionDenied)
  ├─ AgentTool.tsx → executeSubagentStartHooks (SubagentStart)
  ├─ SessionStart → executeSessionStartHooks
  ├─ SessionEnd → executeSessionEndHooks
  ├─ Notification 触发 → executeNotificationHooks
  ├─ Stop / StopFailure → executeStopHooks / executeStopFailureHooks
  ├─ Subagent 完成 → executeSubagentStopHooks
  ├─ Compact 触发 → executePreCompactHooks / executePostCompactHooks
  ├─ UserPromptSubmit → executeUserPromptSubmitHooks
  ├─ Worktree 创建/删除 → executeWorktreeCreateHook / executeWorktreeRemoveHook
  └─ 等等...
        │
        ▼
execute*Hooks 公开 API（每个都构造 hookInput + 调 executeHooks 或 executeHooksOutsideREPL）
        │
        ▼
executeHooks / executeHooksOutsideREPL
        │
        ├─ 全局守门（disableManagedHooks / CLAUDE_CODE_SIMPLE / shouldSkipHookDueToTrust）
        ├─ getMatchingHooks（按 hookInput 提取 matchQuery，过滤 matcher）
        ├─ 信号中止检查
        │
        ├─ 【快路径】所有 hook 都是 internal callback → 同步调 + 埋点 return
        │
        └─ 【正常路径】
            ├─ OTel hook_execution_start + startHookSpan
            ├─ yield 每个钩子的 hook_progress 进度消息
            ├─ all([...]) 并行 spawn 每个钩子：
            │    ├─ type === 'command'   → execCommandHook
            │    ├─ type === 'prompt'    → execPromptHook (external module)
            │    ├─ type === 'agent'     → execAgentHook (external module)
            │    ├─ type === 'http'      → execHttpHook (external module)
            │    ├─ type === 'callback'  → executeHookCallback
            │    └─ FunctionHook (session) → executeFunctionHook
            │
            ├─ 每个 executor 返回 { stdout, stderr, exitCode } 或 { json }
            │  → processHookJSONOutput 归一化 HookResult 字段
            │    ├─ continue: false → preventContinuation + stopReason
            │    ├─ decision: "block" → blockingError
            │    ├─ hookSpecificOutput.permissionDecision → 改写权限流
            │    ├─ hookSpecificOutput.updatedInput → 改写工具输入
            │    ├─ hookSpecificOutput.additionalContext → 注入 system reminder
            │    ├─ updatedMCPToolOutput → 仅 MCP 工具采纳
            │    └─ watchPaths / elicitationResponse / retry 等
            │
            ├─ 循环 yield AggregatedHookResult 给调用方
            │  ├─ message (progress / systemMessage / hook_additional_context)
            │  ├─ blockingError → hook_blocking_error 附件
            │  ├─ preventContinuation → hook_stopped_continuation 附件 + return
            │  └─ 其他副作用透传
            │
            ├─ endHookSpan + OTLP hook_execution_end（每个钩子 outcome 计数）
            ├─ hook_duration_ms + addToTurnHookDuration 统计
            └─ tengu_repl_hook_finished 埋点
```

### `executeHooks` 核心 dispatcher 的 12 步流程

```text
                ┌──────────────────────────────────────────────────┐
                │ executeHooks({hookInput, toolUseID, ...})      │
                │ (被所有 execute*Hooks 公共 API 委托)            │
                └─────────────────────────┬────────────────────────┘
                                          │
        1. 全局守门
           ├─ shouldDisableAllHooksIncludingManaged? → return
           ├─ CLAUDE_CODE_SIMPLE? → return
           └─ shouldSkipHookDueToTrust()? → return
                                          │
        2. 匹配解析                              ▼
           └─ getMatchingHooks → matchingHooks; 空数组 → return
                                          │
        3. 信号检查                              ▼
           └─ signal?.aborted? → return
                                          │
        4. 用户 vs 内部钩子分流                   ▼
           ├─ userHooks.length > 0 → 走正常路径
           └─ 仅 internal callback → 快路径（同步调用 + 埋点 return）
                                          │
        5. OTel 准备（仅 beta tracing）            ▼
           ├─ jsonStringify(getHookDefinitionsForTelemetry)
           └─ logOTelEvent('hook_execution_start', {...})
                                          │
        6. Span                                  ▼
           └─ startHookSpan(hookEvent, hookName, numHooks, hookDefs)
                                          │
        7. 进度预产                              ▼
           └─ for (matchedHook of matchingHooks)
              yield { message: { type: 'progress', data: { hook_progress, ... }}}
                                          │
        8. 并行执行                              ▼
           └─ all([...matchedHooks.map(...)]) → 独立 abortSignal/timeout
              ├─ type === 'command'  → execCommandHook
              ├─ type === 'prompt'   → execPromptHook
              ├─ type === 'agent'    → execAgentHook
              ├─ type === 'http'     → execHttpHook
              ├─ type === 'callback' → executeHookCallback
              └─ FunctionHook        → executeFunctionHook
                                          │
        9. 结果聚合                              ▼
           for (const result of all(...)):
             ├─ result.message → yield
             ├─ result.blockingError → yield hook_blocking_error + 附件
             ├─ result.preventContinuation → yield hook_stopped_continuation + return
             ├─ result.additionalContext(s) → yield hook_additional_context 附件
             ├─ result.permissionBehavior → yield hookPermissionResult
             ├─ result.updatedInput → yield passthrough
             ├─ result.updatedMCPToolOutput → yield（仅 MCP 工具采纳）
             ├─ result.elicitationResponse → yield
             └─ result.retry → yield retry 信号
                                          │
        10. Span 收尾                              ▼
            └─ endHookSpan + OTLP hook_execution_end（含 outcome 计数）
                                          │
        11. 统计                                  ▼
            ├─ getStatsStore().observe('hook_duration_ms', totalDurationMs)
            └─ addToTurnHookDuration(totalDurationMs)
                                          │
        12. 遥测                                  ▼
            └─ logEvent('tengu_repl_hook_finished', {numSuccess, numBlocking, ...})
```

### 命令钩子执行（`execCommandHook`）8 步流程

```text
execCommandHook(hook, hookEvent, hookName, jsonInput, signal, hookId, ...)
  │
  ├─ 1. Shell 选择（hook.shell ?? DEFAULT_HOOK_SHELL）
  │  └─ Windows bash → 路径必须 POSIX 化（windowsPathToPosixPath，memoized）
  │     PowerShell → 跳过转换，使用 native 路径
  │
  ├─ 2. 变量替换
  │  └─ 先 ${CLAUDE_PLUGIN_ROOT} / ${user_config.X}（顺序匹配 MCP/LSP）
  │
  ├─ 3. 环境变量
  │  ├─ CLAUDE_PROJECT_DIR（getProjectRoot() 稳定根，不受 worktree 影响）
  │  ├─ CLAUDE_PLUGIN_ROOT / CLAUDE_SKILL_DIR / CLAUDE_SESSION_ID
  │  └─ session 环境缓存（getHookEnvFilePath 写文件再 source）
  │
  ├─ 4. 子进程启动
  │  └─ wrapSpawn（spawn + StreamWrapper）→ 返回 ShellCommand
  │
  ├─ 5. 串行 yield stdout / stderr / 文件描述符事件
  │
  ├─ 6. 等待结果（respect abortSignal + timeoutMs）
  │
  ├─ 7. 后台化分支
  │  ├─ asyncRewake → executeInBackground（绕过 AsyncHookRegistry）
  │  │  └─ exit code 2 时 enqueue task-notification
  │  └─ 常规 async → executeInBackground → registerPendingAsyncHook
  │
  └─ 8. 返回 { stdout, stderr, output, status, aborted?, backgrounded? }
       → 调用方用 parseHookOutput 解析 → processHookJSONOutput 归一化
```

---

## 关键设计取舍

- **30+ 公开 API 委托给 1 个核心 dispatcher**：`executeHooks` 是单一核心，所有 `execute*Hooks` API 都构造 hookInput 后委托给它 —— 行为一致性 + 安全门控一处维护。
- **快路径 vs 正常路径分流**：所有钩子都是 internal callback（attribution / sessionFileAccess）时同步调用，绕过 span/progress/abortSignal/JSON 解析全套 —— 6.01µs → 1.8µs（-70%）。
- **并行执行 + preventContinuation 短路**：`all([...])` 并行 spawn，第一个有副作用的就 yield；preventContinuation 信号 yield stopReason 后立即 return，丢弃剩余结果 —— hook 链的"短路"语义。
- **4 种执行器分别处理 4 种钩子类型**：命令走 shell、prompt 走 LLM、agent 走 forked sub-agent、HTTP 走 POST。`executeFunctionHook` + `executeHookCallback` 处理内存回调。
- **JSON 输出统一 schema**：所有 4 种执行器的输出（命令 stdout、prompt 文本、agent final、HTTP body）都归一化为 `HookJSONOutput` —— `processHookJSONOutput` 把它们转为统一 `HookResult` 字段。
- **安全门控多重叠加**：managed policy 全局禁用 + 信任门控（仅交互模式）+ `CLAUDE_CODE_SIMPLE` 环境变量 + 单钩子 timeout + 信号中止 —— defense-in-depth。
- **asyncRewake 模式设计**：Stop hook 的 blocking error（exit 2）绕过 AsyncHookRegistry 直接 enqueue task-notification，让用户输入新 prompt 时收到 hook 阻塞通知（不需要等下次 turn 触发）。
- **路径转换按平台 + shell 分支**：Windows bash 路径必须 POSIX 化（Git Bash 无法解析 Windows 路径），PowerShell 跳过 —— 同文件多 shell 共存。
- **环境变量注入顺序**：`CLAUDE_PROJECT_DIR` 用 `getProjectRoot()`（**稳定**项目根，worktree 切换不变），让 `$CLAUDE_PROJECT_DIR` 始终解析到真 repo 根。
- **变量替换顺序匹配 MCP/LSP**：先 `${CLAUDE_PLUGIN_ROOT}` / `${user_config.X}`，让用户配置中的字面量 `${CLAUDE_PLUGIN_ROOT}` 视为透明文本。
- **诊断日志仅 once-per-session**：started/completed 只在 SessionStart / Setup / SessionEnd 写 diag_log，控制日志量；try/finally 包住避免 setup-path 抛错孤立 started marker。
- **权限决策不绕过 settings.json**：`executePermissionDeniedHooks` 触发条件严格：必须 `feature('TRANSCRIPT_CLASSIFIER')` + permissionDecision 决策由 auto-mode classifier 做出（注释明确）。
- **`getMatchingHooks` 与 `hooksConfigManager.ts` 同步**：matchQuery 提取规则两处必须同步修改（注释明确警告）。
- **`processHookJSONOutput` 是输出规范化的唯一入口**：所有钩子类型的 JSON 输出（command stdout / prompt final / agent final / HTTP body）走同一解析，让 block / stop / permissionDecision / additionalContext / updatedInput / updatedMCPToolOutput 等语义跨类型一致。
- **`shouldSkipHookDueToTrust` defense-in-depth**：所有钩子都要求工作区信任，不只是可能被误用的钩子 —— 历史漏洞：SessionEnd 钩子在用户拒绝信任后仍执行 / SubagentStop 在信任建立前触发。

---

## 调用与被调用全景

```text
外部钩子触发点
  ├─ toolExecution.ts 6 阶段流水线
  │  ├─ 阶段 3 (PreToolUse)  → executePreToolHooks
  │  ├─ 阶段 6 (PostToolUse) → executePostToolHooks
  │  └─ 失败路径             → executePostToolUseFailureHooks
  ├─ toolExecution.ts resolvePermissionOrDeny
  │  └─ auto-mode classifier 拒绝 → executePermissionDeniedHooks
  │
  ├─ AgentTool / runAgent
  │  └─ executeSubagentStartHooks / executeSubagentStopHooks
  │
  ├─ SessionStart / Setup → executeSessionStartHooks / executeSetupHooks
  ├─ SessionEnd           → executeSessionEndHooks
  ├─ Notification 触发   → executeNotificationHooks
  ├─ Stop / StopFailure   → executeStopHooks / executeStopFailureHooks
  ├─ UserPromptSubmit     → executeUserPromptSubmitHooks
  ├─ Compact 触发         → executePreCompactHooks / executePostCompactHooks
  ├─ PermissionRequest    → executePermissionRequestHooks
  ├─ ConfigChange         → executeConfigChangeHooks
  ├─ CwdChanged           → executeCwdChangedHooks
  ├─ FileChanged          → executeFileChangedHooks
  ├─ InstructionsLoaded   → executeInstructionsLoadedHooks
  ├─ Elicitation          → executeElicitationHooks / executeElicitationResultHooks
  ├─ StatusLine           → executeStatusLineCommand
  ├─ FileSuggestion       → executeFileSuggestionCommand
  ├─ WorktreeCreate/Remove → executeWorktreeCreateHook / executeWorktreeRemoveHook
  └─ TaskCreated/Completed / TeammateIdle → 对应 execute*Hooks
        │
        ▼
execute*Hooks 公开 API（构造 hookInput + 委托）
        │
        ▼
executeHooks (REPL 内)         executeHooksOutsideREPL (REPL 外)
  │  ↓                             │  ↓
  │ async generator                Promise<HookOutsideReplResult[]>
  │ 用于 yield 到 model 上下文      用于 SessionEnd / Notification / Worktree
  │                                不暴露给 model
        │
        ▼
匹配钩子列表（getMatchingHooks）→ 快路径（internal callback）or 正常路径
        │
        ▼
并行 spawn 每条钩子
  ├─ type='command'  → execCommandHook → spawn shell → stdout/stderr
  ├─ type='prompt'   → execPromptHook → LLM prompt → text
  ├─ type='agent'    → execAgentHook → forked sub-agent → final message
  ├─ type='http'     → execHttpHook → HTTP POST → body
  ├─ type='callback' → executeHookCallback → hook.callback() → JSON
  └─ FunctionHook    → executeFunctionHook → in-memory callback → boolean
        │
        ▼
parseHookOutput / parseHttpHookOutput / parseElicitationHookOutput → validateHookJson
        │
        ▼
processHookJSONOutput → 归一化 HookResult 字段
  ├─ continue: false       → preventContinuation + stopReason
  ├─ decision: "block"     → blockingError
  ├─ permissionDecision    → 改写或直接给出 allow/deny/ask
  ├─ updatedInput          → 改写工具输入
  ├─ additionalContext     → 注入 system reminder
  └─ updatedMCPToolOutput  → 仅 MCP 工具采纳
```

---

## 关联模块速查

| 模块 | 行数 | 职责 |
|------|-----:|------|
| `src/utils/hooks.ts` | ~5200 | 本文件：30+ 公开 execute*Hooks API、4 种执行器、核心 dispatcher、JSON 解析、安全门控 |
| `src/utils/hooks/sessionHooks.ts` | — | session-scoped hooks 存储（`addFunctionHook` / `getSessionHooks` / `clearSessionHooks`） |
| `src/utils/hooks/AsyncHookRegistry.ts` | — | `registerPendingAsyncHook`（async 钩子注册表） |
| `src/utils/hooks/execPromptHook.ts` | — | `execPromptHook`（prompt 类型钩子执行器，调 LLM） |
| `src/utils/hooks/execAgentHook.ts` | — | `execAgentHook`（agent 类型钩子执行器，forked sub-agent） |
| `src/utils/hooks/execHttpHook.ts` | — | `execHttpHook`（HTTP 类型钩子执行器，POST 请求） |
| `src/utils/hooks/hooksConfigSnapshot.ts` | — | `getHooksConfigFromSnapshot` / `shouldDisableAllHooksIncludingManaged` / `shouldAllowManagedHooksOnly` |
| `src/utils/hooks/hooksSettings.ts` | — | `getHookDisplayText` / `isHookEqual` |
| `src/utils/hooks/hooksConfigManager.ts` | — | matcher 解析（与 `getMatchingHooks` 同步维护） |
| `src/utils/hooks/hookEvents.ts` | — | `emitHookStarted` / `emitHookResponse` / `startHookProgressInterval` |
| `src/utils/hooks/registerFrontmatterHooks.ts` | — | `registerFrontmatterHooks`（agent frontmatter hooks 注册） |
| `src/services/tools/toolHooks.ts` | ~680 | 工具钩子编排层（详见 `14-tool-hooks.md`）：消费 `executeHooks` 的 yield 流 |
| `src/services/tools/toolExecution.ts` | 1984 | 6 阶段流水线调用方（详见 `11-tool-execution.md`） |
| `src/services/analytics/index.ts` | — | `logEvent` + `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型 |
| `src/services/analytics/growthbook.ts` | — | GrowthBook feature flag 查询 |
| `src/telemetry/sessionTracing.ts` | — | `startHookSpan` / `endHookSpan` / `isBetaTracingEnabled` |
| `src/telemetry/events.ts` | — | `logOTelEvent`（`hook_execution_start` / `hook_execution_end`） |
| `src/types/hooks.ts` | — | `HookJSONOutput` / `hookJSONOutputSchema` / `HookCallback` / `PromptRequest` / `PermissionRequestResult` |
| `src/types/message.ts` | — | `HookResultMessage` / `AttachmentMessage` |
| `src/utils/attachments.ts` | — | `createAttachmentMessage`（构造 hook_cancelled / hook_blocking_error / hook_stopped_continuation / hook_additional_context / hook_error_during_execution 附件） |
| `src/utils/permissions/PermissionResult.ts` | — | `PermissionResult` 类型 |
| `src/utils/combinedAbortSignal.ts` | — | `createCombinedAbortSignal`（外部 signal + timeout 合并） |
| `src/utils/messageQueueManager.ts` | — | `enqueuePendingNotification`（task-notification 入队） |
| `src/utils/slowOperations.ts` | — | `jsonStringify` / `jsonParse` |
| `src/utils/log.ts` | — | `logError`（错误日志） |
| `src/utils/debug.ts` | — | `logForDebugging`（调试日志） |
| `src/utils/diagLogs.ts` | — | `logForDiagnosticsNoPII`（诊断日志，不含 PII） |
| `src/utils/errors.ts` | — | `errorMessage` / `getErrnoCode` |
| `src/utils/ShellCommand.ts` | — | `wrapSpawn` / `ShellCommand`（shell 命令封装） |
| `src/utils/bash/shellPrefix.ts` | — | `formatShellPrefixCommand`（shell 前缀格式化） |
| `src/utils/subprocessEnv.ts` | — | `subprocessEnv`（子进程环境变量） |
| `src/utils/windowsPaths.ts` | — | `windowsPathToPosixPath`（Windows 路径 POSIX 化，memoized LRU-500） |
| `src/utils/shell/powershellDetection.ts` | — | `getCachedPowerShellPath` |
| `src/utils/shell/shellProvider.ts` | — | `DEFAULT_HOOK_SHELL` |
| `src/utils/shell/shellProvider.ts` | — | `buildPowerShellArgs` |
| `src/utils/plugins/pluginOptionsStorage.ts` | — | `loadPluginOptions` / `substituteUserConfigVariables` |
| `src/utils/plugins/pluginDirectories.ts` | — | `getPluginDataDir` |
| `src/utils/plugins/schemas.ts` | — | `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` |
| `src/utils/sessionStorage.ts` | — | `getTranscriptPathForSession` / `getAgentTranscriptPath` |
| `src/utils/sessionEnvironment.ts` | — | `getHookEnvFilePath` / `invalidateSessionEnvCache` |
| `src/utils/cwd.ts` | — | `getCwd`（当前工作目录） |
| `src/bootstrap/state.ts` | — | `getSessionId` / `getProjectRoot` / `getIsNonInteractiveSession` / `getRegisteredHooks` / `getStatsStore` / `addToTurnHookDuration` / `getOriginalCwd` / `getMainThreadAgentType` |
| `src/state/AppState.ts` | — | `AppState` 类型（sessionHooks 在 appState 中） |
| `src/Tool.ts` | — | `findToolByName` / `Tools` / `ToolUseContext` 类型 |
| `src/entrypoints/agentSdkTypes.ts` | — | 30+ `*HookInput` 类型 |
| `src/types/statusLine.ts` | — | `StatusLineCommandInput` |
| `src/types/plugin.ts` | — | `LoadedPlugin` / `PluginError` |

---

## 重要不变量

- **所有钩子都要求工作区信任（interactive mode）**：`shouldSkipHookDueToTrust` 是 defense-in-depth 核心安全门控 —— 历史漏洞（SessionEnd 在拒绝信任后执行 / SubagentStop 在信任建立前触发）促使此检查。
- **钩子 allow 不绕过 settings.json 规则**：核心安全不变量（与 `toolHooks.ts → resolveHookPermissionDecision` 配合，详见 `14-tool-hooks.md`）。
- **30+ 公开 API 委托给 1 个 dispatcher**：所有 `execute*Hooks` 都构造 hookInput 后调 `executeHooks` —— 行为一致性 + 安全门控一处维护。
- **4 种执行器类型独立**：命令（shell）/ prompt（LLM）/ agent（forked sub-agent）/ HTTP（POST）+ Function 回调 + Callback 钩子 —— 各自处理一种类型。
- **JSON 输出统一 schema**：所有 4 种执行器的输出都归一化为 `HookJSONOutput` —— `processHookJSONOutput` 统一解析，让 block / stop / permissionDecision / additionalContext / updatedInput / updatedMCPToolOutput 语义跨类型一致。
- **并行执行 + preventContinuation 短路**：`all([...])` 并行 spawn；preventContinuation yield stopReason 后立即 return —— hook 链的"短路"语义。
- **快路径优化**：所有钩子都是 internal callback（attribution / sessionFileAccess）时同步调用 + 埋点 return —— 6.01µs → 1.8µs（-70%）。
- **`getMatchingHooks` 与 `hooksConfigManager.ts` 同步**：matchQuery 提取规则两处必须同步修改（注释明确警告）。
- **`processHookJSONOutput` 是输出规范化的唯一入口**：block / stop / permissionDecision / additionalContext / updatedInput / updatedMCPToolOutput 跨类型一致。
- **asyncRewake 模式绕过 AsyncHookRegistry**：Stop hook 的 blocking error（exit 2）直接 enqueue task-notification，让用户输入新 prompt 时收到 hook 阻塞通知。
- **`updatedMCPToolOutput` 仅 MCP 工具采纳**：调用方 `runPostToolUseHooks` 显式检查 `isMcpTool(tool)`，非 MCP 工具即使钩子改了 MCP 输出也忽略。
- **`shouldSkipHookDueToTrust` defense-in-depth**：所有钩子都要求工作区信任，不仅是可能被误用的钩子。
- **环境变量注入顺序**：`${CLAUDE_PLUGIN_ROOT}` / `${user_config.X}` 顺序匹配 MCP/LSP —— 用户配置中含 `${CLAUDE_PLUGIN_ROOT}` 字面量视为透明文本。
- **Windows bash 路径必须 POSIX 化**：Git Bash 无法解析 Windows 路径；`windowsPathToPosixPath` 纯 JS regex 转换，memoized LRU-500。
- **`CLAUDE_PROJECT_DIR` 用稳定项目根**：`getProjectRoot()` 不受 worktree 影响，让 `$CLAUDE_PROJECT_DIR` 始终解析到真 repo 根。
- **诊断日志仅 once-per-session**：SessionStart / Setup / SessionEnd 写 diag_log；try/finally 包住避免 setup-path 抛错孤立 started marker。
- **line ending 兼容性**：文件由 Edit 工具写入 LF；Git autocrlf 会在下次 checkout 时自动转换为 CRLF（warning "LF will be replaced by CRLF" 自愈）。
