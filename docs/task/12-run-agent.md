# runAgent — 子代理 query 循环封装器

> 源码位置：`src/tools/AgentTool/runAgent.ts`（976 行）
> 调用方：`src/tools/AgentTool/AgentTool.tsx`（同步/异步/fork/worktree 各路径）、`src/tools/AgentTool/resumeAgent.ts`（续接后台 agent）
> 下游：`src/query.ts`（真正的 query 循环）、`src/utils/forkedAgent.ts`（createSubagentContext）、`src/tools/AgentTool/agentToolUtils.ts`（resolveAgentTools / finalizeAgentTool）、`src/services/mcp/client.ts`（MCP 客户端）、`src/utils/hooks.ts`（SubagentStart 钩子）

`runAgent` 是 AgentTool 的"执行层"。它接收 `AgentTool.call()` 准备好的全部上下文（agent 定义、prompt 消息、tool pool、canUseTool、querySource 等），构造一个完全隔离的子代理 `ToolUseContext`，把 `query(...)` 跑起来，再做 sidechain transcript 录制、MCP 清理、kill shell tasks、prompt cache tracking 等一系列 finally 兜底。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 上下文隔离 | 用 `createSubagentContext(toolUseContext, overrides)` 派生 `agentToolUseContext`：默认 mutable state 全 no-op（`shareSetAppState` 默认 false、`shareAbortController` 默认 false），仅当 `useExactTools`/`isAsync=false` 等场景显式 opt-in 共享 |
| fork 路径缓存复用 | `useExactTools=true` 时把父代理的 `toolUseContext.options.tools` 原样传入（不走 `resolveAgentTools`）、`thinkingConfig` 继承父、`isNonInteractiveSession` 继承父、`querySource` 注入 `agentOptions`（让递归 fork 守卫能跨 autocompact 命中） |
| 模型解析 | `getAgentModel`：`CLAUDE_CODE_SUBAGENT_MODEL` 环境变量优先；其次 tool-specified model 命中父 tier 时直接继承父 model（避免 Vertex 用户被降级）；Bedrock 跨区域 prefix 沿用父 |
| 上下文裁剪 | `omitClaudeMd` 标志 + `tengu_slim_subagent_claudemd` GrowthBook 守护：read-only agents（Explore/Plan）不发 CLAUDE.md（5-15 Gtok/week）；同时剥离 `gitStatus`（1-3 Gtok/week） |
| 前置钩子 | 执行 `SubagentStart` 钩子链，收集 `additionalContexts` 拼成 `hook_additional_context` attachment message 推到 `initialMessages` |
| 隔离 frontmatter hooks | `isRestrictedToPluginOnly('hooks')` 守护：仅 plugin/built-in/policySettings 的 admin-trusted agents 能注册 frontmatter hooks，普通 user agent 的 hooks 被屏蔽 |
| 技能预加载 | `agentDefinition.skills` 预加载：`resolveSkillName` 三策略（exact match → plugin 前缀 → 后缀匹配），`Promise.all` 并发加载 `skill.getPromptForCommand`，UI 显示 `formatSkillLoadingMetadata` |
| agent 专属 MCP | `initializeAgentMcpServers`：字符串引用走 `getMcpConfigByName` 复用父 client；inline `{ [name]: config }` 创建新 client（仅这些进 cleanup）；`isRestrictedToPluginOnly('mcp')` 守护同前 |
| abortController 分层 | `override?.abortController` > `isAsync ? new AbortController() : toolUseContext.abortController` —— async agent 不受父 ESC 影响，独立生命周期 |
| permission 隔离与能力 | `agentGetAppState` 包装：agent 定义 `permissionMode` 时覆盖（但 bypassPermissions / acceptEdits / auto-mode 不会覆盖父）；`shouldAvoidPermissionPrompts` 决定是否弹权限框（bubble mode 总是弹，async 不弹）；`awaitAutomatedChecksBeforeDialog` 用于 async + 可见 prompts 的场景 |
| allowedTools 作用域 | `allowedTools` 显式覆盖 `alwaysAllowRules.session`，**保留** `cliArg` 规则（SDK `--allowedTools` 显式授权跨所有 agent 生效），防止父 session 批准泄漏 |
| effort 隔离 | agent 自定义 `effort` 覆盖父 effort；与父 effort 相同时直接返回原 state（reference equality）避免 React 重渲染 |
| 工具解析 | `useExactTools ? availableTools : resolveAgentTools(...)`：wildcard（`*`）→ 全工具集；显式列表 → 按 `permissionRuleValueFromString` 解析；`Agent(worker,researcher)` 形式透传 `allowedAgentTypes` 给 AgentTool |
| 初始消息 | `filterIncompleteToolCalls(forkContextMessages)` 剔除父消息中 tool_use 没有 tool_result 的孤儿块（防止 API 报错）→ `cloneFileStateCache` 隔离 read file state → 创建/继承 userContext/systemContext → 拼 `initialMessages` |
| transcript 录制 | `recordSidechainTranscript(initialMessages, agentId)` 启动时 fire-and-forget 写初始消息；循环中每条 `isRecordableMessage` 增量追加（O(1) per message，记录 `lastRecordedUuid` 维护 parent 链） |
| metadata 持久化 | `writeAgentMetadata(agentId, { agentType, worktreePath?, description? })` 写本地 metadata，resume 时按 agentType 路由 |
| query loop | 调 `query({ messages, systemPrompt, userContext, systemContext, canUseTool, toolUseContext, querySource, maxTurns })`，把 yield 的消息按类型分流 |
| 消息流处理 | `onQueryProgress` 每条消息触发（用于检测长 stream 期间的活跃）；`stream_event.message_start.ttftMs` 上抛父 `pushApiMetricsEntry`（让父 spinner TTFT/OTPS 更新）；`attachment.max_turns_reached` 截断循环；其他 attachment 透传 |
| finally 兜底 | `mcpCleanup` 释放 inline MCP client；`clearSessionHooks` 释放 frontmatter hooks；`cleanupAgentTracking` 释放 prompt cache tracking；`readFileState.clear()` 释放克隆的 read state；`initialMessages.length = 0` 释放 fork context；`unregisterPerfettoAgent` 释放 Perfetto 注册；`clearAgentTranscriptSubdir` 释放 transcript subdir 映射；`setAppState` 移除 todos 残留；`killShellTasksForAgent` 杀 shell 任务防 zombie；`killMonitorMcpTasksForAgent` 杀 monitor tasks |
| AgentTool callback | 仅 builtin agents 且定义了 `callback` 才在正常完成时触发（用于副作用如 hook 反注册） |

---

## 核心执行流程

`runAgent(opts)` 是 async generator；调用方通过 `for await (const msg of runAgent(...))` 拿到逐条消息。下图按执行顺序铺平关键决策点：

```text
                ┌──────────────────────────────────────────────────┐
                │ runAgent({agentDefinition, promptMessages, ...})│
                └─────────────────────────┬────────────────────────┘
                                          │
        1. 上下文装载
           ├─ appState + permissionMode + rootSetAppState
           ├─ resolvedAgentModel = getAgentModel(...)
           ├─ agentId = override?.agentId ?? createAgentId()
           ├─ (可选) setAgentTranscriptSubdir(agentId, transcriptSubdir)
           └─ (可选) registerPerfettoAgent + ant-only dump path log
                                          │
                                          ▼
        2. Fork context 准备
           ├─ contextMessages = forkContextMessages ? filterIncompleteToolCalls(...) : []
           ├─ initialMessages = [...contextMessages, ...promptMessages]
           ├─ agentReadFileState = forkContextMessages ? cloneFileStateCache(...) : createFileStateCacheWithSizeLimit(...)
           └─ (并发) baseUserContext = override?.userContext ?? getUserContext()
                  baseSystemContext = override?.systemContext ?? getSystemContext()
                                          │
                                          ▼
        3. 上下文裁剪
           ├─ shouldOmitClaudeMd = agentDefinition.omitClaudeMd && !override?.userContext && GrowthBook
           │  → 剔除 userContext.claudeMd（read-only agents 节省 5-15 Gtok/week）
           └─ 若是 Explore/Plan → 剔除 systemContext.gitStatus（节省 1-3 Gtok/week）
                                          │
                                          ▼
        4. agentGetAppState 包装（permission 覆盖 + allowedTools + effort）
           ├─ 若 agentDefinition.permissionMode 且父 mode 非 bypass/acceptEdits/auto
           │  → 覆盖 mode
           ├─ shouldAvoidPermissionPrompts：
           │    bubble mode → false（永远弹）
           │    !canShowPermissionPrompts & isAsync → true（不弹）
           ├─ isAsync && !shouldAvoidPrompts → awaitAutomatedChecksBeforeDialog=true
           ├─ allowedTools → 替换 alwaysAllowRules.session（保留 cliArg）
           └─ agentDefinition.effort 覆盖 state.effortValue（reference equality 短路）
                                          │
                                          ▼
        5. 工具解析
           ├─ useExactTools ? availableTools : resolveAgentTools(...)  (wildcard / explicit / Agent(x,y))
           └─ (异步) initializeAgentMcpServers → 合并 agent 专属 MCP tools，uniqBy name
                                          │
                                          ▼
        6. System prompt 准备
           ├─ agentSystemPrompt = override?.systemPrompt ?? asSystemPrompt(getAgentSystemPrompt(...))
           └─ agentAbortController = override?.abortController ?? (isAsync ? new AC() : parent)
                                          │
                                          ▼
        7. 前置钩子 + 技能预加载
           ├─ executeSubagentStartHooks → additionalContexts → push hook_additional_context
           ├─ agentDefinition.hooks && (非 plugin-only || admin-trusted)
           │  → registerFrontmatterHooks(rootSetAppState, agentId, ..., isAgent=true)
           └─ agentDefinition.skills?:
              ├─ resolveSkillName 3 策略（exact / plugin 前缀 / 后缀）
              ├─ Promise.all 加载 skill.getPromptForCommand(...)
              └─ push isMeta user message with formatSkillLoadingMetadata
                                          │
                                          ▼
        8. 子代理上下文隔离
           ├─ createSubagentContext(toolUseContext, {
           │    options: agentOptions,
           │    agentId, agentType, messages: initialMessages,
           │    readFileState: agentReadFileState,
           │    abortController: agentAbortController,
           │    getAppState: agentGetAppState,
           │    shareSetAppState: !isAsync,
           │    shareSetResponseLength: true,
           │    criticalSystemReminder_EXPERIMENTAL,
           │    contentReplacementState
           │  })
           ├─ preserveToolUseResults 时设 agentToolUseContext.preserveToolUseResults = true
           └─ onCacheSafeParams? → push CacheSafeParams（给后台 summarization 派生 progress）
                                          │
                                          ▼
        9. 持久化（fire-and-forget）
           ├─ void recordSidechainTranscript(initialMessages, agentId)
           └─ void writeAgentMetadata(agentId, {agentType, worktreePath?, description?})
                                          │
                                          ▼
        10. query 循环
            for await (const message of query({...})) {
              onQueryProgress?.()
              // stream_event.message_start → pushApiMetricsEntry(ttftMs)（让父 spinner TTFT 更新）
              // attachment.max_turns_reached → break
              // 其他 attachment → yield
              // isRecordableMessage → recordSidechainTranscript(增量追加, O(1)) + yield
            }
                                          │
                                          ▼
        11. abort 检查 + builtin callback
            ├─ agentAbortController.signal.aborted → throw AbortError()
            └─ isBuiltInAgent && agentDefinition.callback → callback()
                                          │
                                          ▼
        12. finally 兜底清理
            ├─ mcpCleanup()                         ← inline MCP client 关闭
            ├─ clearSessionHooks(rootSetAppState, agentId)
            ├─ cleanupAgentTracking(agentId)        ← feature('PROMPT_CACHE_BREAK_DETECTION')
            ├─ agentToolUseContext.readFileState.clear()
            ├─ initialMessages.length = 0          ← 释放 fork context
            ├─ unregisterPerfettoAgent(agentId)
            ├─ clearAgentTranscriptSubdir(agentId)
            ├─ setAppState: 移除 prev.todos[agentId]  ← 防 whale session todo 残留
            ├─ killShellTasksForAgent(agentId)      ← 防 PPID=1 zombie
            └─ feature('MONITOR_TOOL') ? killMonitorMcpTasksForAgent(...)
```

### 消息分类（query 循环内）

```text
query(...) yield 的消息类型
       │
       ├─ stream_event (含 message_start + ttftMs)
       │  → toolUseContext.pushApiMetricsEntry(ttftMs)（不让上层 UI 失活）
       │  → continue
       │
       ├─ attachment
       │  ├─ type === 'max_turns_reached' → break（不再 yield）
       │  └─ 其他 → yield（透传给 AgentTool，structured_output 等由上层处理）
       │
       └─ Message
          ├─ type === 'assistant' | 'user' | 'progress' | ('system' && subtype === 'compact_boundary')
          │  → isRecordableMessage 为 true
          │  → recordSidechainTranscript([message], agentId, lastRecordedUuid) 增量追加
          │  → 若非 progress 类型，更新 lastRecordedUuid
          │  → yield
          │
          └─ 其他（SystemMessage 大多数 subtype）→ 不录制，不 yield
```

### SubagentStart 钩子流

```text
executeSubagentStartHooks(agentId, agentDefinition.agentType, abortSignal)
       ↓
逐条聚合 additionalContexts
       ↓
若有 contexts:
  createAttachmentMessage({
    type: 'hook_additional_context',
    content: [...additionalContexts],
    hookName: 'SubagentStart',
    toolUseID: randomUUID(),
    hookEvent: 'SubagentStart',
  })
       ↓
push 到 initialMessages
```

`isAgent=true` 让 registerFrontmatterHooks 把 Stop 钩子自动转换为 SubagentStop（子代理触发的是 SubagentStop 而非 Stop），同 admin-trusted gate（`isRestrictedToPluginOnly('hooks')` 守 plugin-only 时只放行 admin-trusted 来源）。

---

## 关键设计取舍

- **三层隔离 (createSubagentContext)**：默认 mutable state 全 no-op；只通过显式 `shareSetAppState` / `shareAbortController` / `shareSetResponseLength` opt-in 共享 —— 子代理不会因为 UI 渲染抖动、abort 风暴、response metrics 累积影响父代理。
- **fork 路径 byte-identical 前缀**：useExactTools 时 `agentOptions.tools` 用父原样 + `thinkingConfig` 继承 + `querySource` 注入 —— 让 prompt cache key 命中父代理的 cache 链。但仍然用 `agentAbortController`（独立）、`messages` 用 fork context（独立）、`getAppState` 走包装（独立）—— 隔离生命周期，共享缓存前缀。
- **agent 模型解析**：tool 指定的 model 命中父 tier 时直接继承父 model（`aliasMatchesParentTier`）—— 防止 Vertex 用户被默认 resolve 降级到 Anthropic API 的 Opus（issue #30815）；仅裸家族别名命中，`opus[1m]` / `best` / `opusplan` 透传（语义超出 "same tier as parent"）。
- **CLAUDE.md 裁剪**：仅 Explore/Plan 读不发，省 5-15 Gtok/week —— Explore/Plan 的输出会被主代理二次解读，CLAUDE.md 的 commit/PR/lint 规则对它们是死重。`tengu_slim_subagent_claudemd=false` 可关。
- **gitStatus 裁剪**：Explore/Plan 也不带 gitStatus（最多 40KB，明确标 stale）—— 它们需要 git 信息会自己跑 `git status`，省 1-3 Gtok/week。
- **allowedTools 作用域**：保留 `cliArg` 规则（SDK `--allowedTools` 显式授权跨所有 agent 生效）；只替换 `session` 规则（防父 session 批准的"下次自动允许"泄漏到子代理）。
- **MCP 清理作用域**：仅 `newlyCreatedClients` 进 cleanup —— 字符串引用（memoized shared client）不清理；inline `{ [name]: config }` 创建的进 cleanup。
- **frontmatter hooks 隔离作用域**：用 `isAgent=true` 让子代理的 Stop 钩子自动变 SubagentStop（避免子代理触发主代理的 Stop 钩子）；同 admin-trusted gate。
- **finally 兜底全覆盖**：MCP / hooks / cache tracking / read state / fork context / Perfetto / transcript subdir / todos / shell tasks / monitor tasks —— 子代理无论正常完成、abort、throw 都得彻底清理；whale session（数百个 agent 串行）不残留孤儿。
- **fire-and-forget 持久化**：`recordSidechainTranscript` / `writeAgentMetadata` 用 `void ... .catch(logForDebugging)` —— 持久化失败不应阻塞 agent 运行（transcript 漏一条不影响主流程）。

---

## 调用与被调用全景

```text
AgentTool.call(input, toolUseContext, ...)         resumeAgent.ts
       │                                                  │
       ├─ 同步路径                                          │
       │   for await (msg of runAgent({                   │
       │     agentDefinition: selectedAgent,               │
       │     promptMessages,                              │
       │     availableTools: workerTools,                  │
       │     isAsync: false,                              │
       │     ...                                          │
       │   }))                                            │
       │                                                  │
       ├─ 异步路径                                          │
       │   runAsyncAgentLifecycle → runAgent({            │
       │     agentDefinition,                             │
       │     promptMessages,                              │
       │     availableTools,                              │
       │     isAsync: true,                               │
       │     ...                                          │
       │   })                                             │
       │                                                  │
       ├─ Fork 路径                                        │
       │   runAgent({                                    │
       │     useExactTools: true,                         │
       │     forkContextMessages: toolUseContext.messages,│
       │     override: { systemPrompt: forkParentSystemPrompt },
       │     ...                                          │
       │   })                                             │
       │                                                  │
       └─ Worktree / CWD override                          │
           runWithCwdOverride(worktreePath, () => runAgent(...))
```

---

## 关联模块速查

| 模块 | 行数 | 职责 |
|------|-----:|------|
| `src/tools/AgentTool/runAgent.ts` | 976 | 本文件：上下文隔离、agent lifecycle、finally 兜底 |
| `src/tools/AgentTool/AgentTool.tsx` | 1447 | 调用方：team/fork/remote/worktree/async 多路径，详见 `10-agent-tool.md` |
| `src/tools/AgentTool/agentToolUtils.ts` | ~686 | `resolveAgentTools` / `filterDeniedAgents` / `getAgentModel` / `finalizeAgentTool` / `classifyHandoffIfNeeded` / `runAsyncAgentLifecycle` |
| `src/tools/AgentTool/loadAgentsDir.ts` | — | `AgentDefinition` 类型 / `getPrompt` / `isBuiltInAgent` |
| `src/tools/AgentTool/forkSubagent.ts` | 210+ | `FORK_AGENT` / `isForkSubagentEnabled` / `buildForkedMessages` / `buildWorktreeNotice` / `buildChildMessage` |
| `src/utils/forkedAgent.ts` | — | `createSubagentContext`（核心隔离器）、`CacheSafeParams`（prompt cache sharing 锚点）、`ForkedAgentParams`（fork spawn 接口）、`SubagentContextOverrides`（`shareSetAppState` / `shareAbortController` 等显式 opt-in 开关） |
| `src/query.ts` | — | 真正的 query 循环：prepareQueryTurn → 流式 model 调用 → processFollowUpTurn → 压缩/microcompact 决策；yield `StreamEvent | RequestStartEvent | Message | AttachmentMessage | TombstoneMessage | ToolUseSummaryMessage` |
| `src/utils/model/agent.ts` | — | `getAgentModel`（alias tier 匹配、Bedrock region prefix 沿用） |
| `src/services/mcp/client.ts` | — | `connectToServer`（memoized）/ `fetchToolsForClient` / `McpAuthError` |
| `src/services/mcp/config.ts` | — | `getMcpConfigByName`（按名字查 MCP config） |
| `src/services/api/promptCacheBreakDetection.ts` | — | `cleanupAgentTracking`（PROMPT_CACHE_BREAK_DETECTION 守护下的 tracking state 释放） |
| `src/services/api/dumpPrompts.ts` | — | `getDumpPromptsPath`（ant-only API 调用 dump 路径） |
| `src/utils/hooks/registerFrontmatterHooks.ts` | — | `registerFrontmatterHooks`（isAgent=true 时 Stop → SubagentStop） |
| `src/utils/hooks/sessionHooks.ts` | — | `clearSessionHooks`（按 agentId 清理 session-scoped hooks） |
| `src/utils/hooks.ts` | — | `executeSubagentStartHooks`（SubagentStart 钩子链） |
| `src/utils/sessionStorage.ts` | — | `setAgentTranscriptSubdir` / `clearAgentTranscriptSubdir` / `recordSidechainTranscript` / `writeAgentMetadata` |
| `src/utils/fileStateCache.ts` | — | `createFileStateCacheWithSizeLimit` / `cloneFileStateCache`（Read 工具的文件状态缓存） |
| `src/utils/systemPromptType.ts` | — | `SystemPrompt` 类型 / `asSystemPrompt` |
| `src/utils/telemetry/perfettoTracing.ts` | — | `isPerfettoTracingEnabled` / `registerAgent` / `unregisterAgent`（Perfetto trace 层级） |
| `src/tasks/LocalShellTask/killShellTasks.ts` | — | `killShellTasksForAgent`（防 PPID=1 zombie） |
| `src/utils/settings/pluginOnlyPolicy.ts` | — | `isRestrictedToPluginOnly` / `isSourceAdminTrusted`（plugin-only 守护） |
| `src/services/analytics/growthbook.ts` | — | `getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)`（CLAUDE.md 裁剪开关） |
| `src/commands.ts` | — | `getCommand` / `getSkillToolCommands` / `hasCommand`（skill 名称解析） |
| `src/constants/querySource.ts` | — | `QuerySource` 类型（`agent:builtin:<type>` / `agent:builtin:fork` 等） |
| `src/constants/prompts.ts` | — | `DEFAULT_AGENT_PROMPT` / `enhanceSystemPromptWithEnvDetails` |
| `src/utils/processUserInput/processSlashCommand.ts` | — | `formatSkillLoadingMetadata`（动态 import 避免循环依赖） |
| `src/utils/attachments.ts` | — | `createAttachmentMessage`（`hook_additional_context` / `structured_output` 等 attachment 构造） |
| `src/utils/messages.ts` | — | `createUserMessage` |
| `src/utils/errors.ts` | — | `AbortError`（finally 后检查 abort 状态时抛） |
| `src/types/ids.ts` | — | `AgentId` 类型 |
| `src/types/message.ts` | — | `Message` / `UserMessage` / `AssistantMessage` / `ProgressMessage` / `SystemCompactBoundaryMessage` 等 |

---

## 重要不变量

- **`useExactTools` 时绝不调 `resolveAgentTools`**：fork 路径必须保留父工具池 byte-identical 以命中 cache —— 任何 `filterToolsForAgent` 介入会破坏 cache key。
- **`thinkingConfig` 永远随 useExactTools 走**：fork 继承父（cache 命中）；普通子代理强制 `{ type: 'disabled' }`（成本控制）。
- **`agentAbortController` 永不等于 `toolUseContext.abortController`**（async 路径）：async agent 必须有独立生命周期，父按 ESC 不能杀掉子。`override?.abortController` 优先级最高（如 backgrounded 续接复用 `task.abortController`）。
- **`agentOptions.querySource` 仅 useExactTools 注入**：让 AgentTool.call() 的递归 fork 守卫能跨 autocompact 命中（autocompact 改写 messages，但不改 context.options）。
- **`createSubagentContext` 默认全 no-op**：仅 `shareSetAppState: !isAsync` / `shareSetResponseLength: true` 是显式开 —— 父子隔离是默认行为，共享是显式 opt-in。
- **filterIncompleteToolCalls 必跑**：fork context 里若有 tool_use 没有对应 tool_result 的孤儿块，不剔除会触发 API 错误。
- **`omitClaudeMd` 守护父显式覆盖**：若 `override?.userContext` 非空（说明调用方主动提供了 userContext），不剥 CLAUDE.md —— 不破坏显式意图。
- **`tengu_slim_subagent_claudemd` 杀开关**：默认 true（剥 CLAUDE.md），但 GrowthBook 控制可关。
- **Explore/Plan 是 read-only**：默认剥离 CLAUDE.md + gitStatus + 走 `filterToolsForAgent`（自带 disallow 列表）—— 它们的输出主代理会解读，不需要完整工程化上下文。
- **finally 必须在所有路径跑**：query 循环 throw、abort、max_turns_reached 截断、callback throw 都触发 —— 9 项清理（mcp / hooks / cache / read state / fork context / Perfetto / transcript / todos / shell tasks / monitor tasks）全覆盖。
- **killShellTasksForAgent 必须跑**：`run_in_background` shell loop（fake-logs.sh 这类）会变成 PPID=1 zombie —— 必须显式杀。
- **transcript 录制 O(1) per message**：`lastRecordedUuid` 维护 parent 链，append-only，避免重复扫描。
- **fire-and-forget 持久化失败不阻塞 agent**：`void recordSidechainTranscript(...).catch(logForDebugging)` —— transcript 漏一条不致命。
