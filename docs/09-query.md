# QUERY — 一次查询的主循环

> 源码位置：`src/query.ts`（2337 行）+ `src/query/{config,deps,stopHooks,tokenBudget,transitions}.ts`
> 入口函数：`query(params)` → `queryLoop(params, consumedCommandUuids)`
> 关联模块：`src/QueryEngine.ts`（头less/SDK 封装层）·`src/cli/print.ts`（头less 驱动器）·`src/screens/REPL.tsx`（REPL 驱动器）
> 关联文档：[08-main.md](./08-main.md) · [06-bridge.md](other/06-bridge.md) · [07-feature-gates.md](other/07-feature-gates.md)

`src/query.ts` 是 Claude Code 一次"用户输入→模型流式响应→工具执行→下一轮"的**核心调度循环**。它把分散的子系统（API 调用、压缩、流式工具执行、错误恢复、预算管理、Stop Hook、Token Budget）编织成一条可恢复、可观察、可测试的状态机。其设计哲学只有一条：**State 不可变、Continue 站点集中、副作用 yield 化**。

---

## 一、整体职责

| 能力 | 说明 |
|------|------|
| 主循环驱动 | 在 while(true) 内迭代"准备 → 调模型 → 处理响应" |
| 多轮恢复 | 支持 fallback、413 压缩、media 错误、max_output_tokens 提升、stop hook 阻塞、token budget nudge |
| 工具执行 | 流式并行执行（StreamingToolExecutor）+ 兜底同步执行（runTools） |
| 压缩编排 | HISTORY_SNIP → microcompact → CONTEXT_COLLAPSE → autocompact → reactiveCompact 五级串联 |
| 状态追踪 | chainId / depth 跨 sub-agent 嵌套 + AutoCompactTrackingState 跨压缩边界 |
| 预算管理 | API `task_budget.remaining` 跨 compact 累计 + TOKEN_BUDGET 续聊 nudge |
| Hook 集成 | PostSamplingHook / StopHook / ApiQueryHook |
| 错误恢复 | FallbackTriggeredError / ImageSizeError / ImageResizeError / 通用 model_error |

---

## 二、文件结构

```
src/query.ts (2337 lines)
├── 辅助工具
│   ├── yieldMissingToolResultBlocks()           L130-156   生成缺失 tool_result
│   ├── isWithheldMaxOutputTokens()              L183-187   max_output_tokens 错误检测
│   └── MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3     L171       最大恢复次数
│
├── 类型定义
│   ├── QueryCompactionResult                    L189-191
│   ├── QueryTrackingContext                     L193-197
│   ├── PreparedQueryTurn                        L199-215   prepareQueryTurn 返回值
│   ├── PreparedModelExecutionContext            L217-222
│   ├── ModelExecutionState                      L504-511   跨阶段共享执行状态
│   ├── State                                    L1278-1291 跨轮次可变状态
│   ├── PreparedNextLoopTurn                     L1294-1306 prepareNextLoopTurn 返回值
│   └── FollowUpTurnResult / NonFollowUpOutcome  L1309-1321, 873-887
│
├── 内部状态构造
│   ├── initializeQueryLoopState()               L231-244
│   ├── createQueryTrackingContext()             L255-278   chainId / depth 嵌套
│   └── prepareQueryTurn()                       L293-462   准备本轮消息 + 压缩
│
├── 模型执行
│   ├── prepareModelExecutionContext()           L474-502   appState / model / dumpPrompts
│   └── executeModelStreamingTurn()              L529-870   流式调 API + 异常恢复
│
├── 错误处理
│   └── handleNonFollowUpTurn()                  L904-1253  非 follow-up 分支的 7 种决策
│
├── 工具执行与下轮准备
│   ├── processFollowUpTurn()                    L1339-1544 同步/流式工具执行
│   └── prepareNextLoopTurn()                    L1561-1838 附件 + 队列 + 下一 state
│
├── 公开入口
│   ├── query()                                  L1849-1869 公共 AsyncGenerator 入口
│   └── queryLoop()                              L1884-2337 while(true) 主循环
│
src/query/
├── config.ts          (46 lines)  QueryConfig 不可变快照
├── deps.ts            (40 lines)  QueryDeps 注入式 I/O
├── stopHooks.ts       (17K)       handleStopHooks 异步迭代器
├── tokenBudget.ts     (93 lines)  checkTokenBudget 续聊决策
└── transitions.ts     (4 lines)   Transition 占位（query.ts 实际用 State 模式）
```

---

## 三、`QueryParams` — 公开 API

```ts
export type QueryParams = {
  messages: Message[]                  // 当前会话消息流
  systemPrompt: SystemPrompt          // 已渲染的 system prompt
  userContext: { [k: string]: string } // 用户级上下文（CWD、git、env 等）
  systemContext: { [k: string]: string } // 系统级上下文
  canUseTool: CanUseToolFn            // 工具权限检查
  toolUseContext: ToolUseContext      // 工具执行上下文（含 abortController、options 等）
  fallbackModel?: string              // 降级模型（fallback 触发时使用）
  querySource: QuerySource            // 调模型时的源标识（用于埋点/限流）
  maxOutputTokensOverride?: number    // 单次响应最大 token
  maxTurns?: number                   // 最大轮次
  skipCacheWrite?: boolean            // 跳过 prompt cache 写入
  // API task_budget（output_config.task_budget，beta agent-budgets-2026-03-13）
  taskBudget?: { total: number }
  deps?: QueryDeps                    // 测试时可注入 fake
}
```

`query(params)` 返回 `AsyncGenerator<...StreamEvent|Message|TombstoneMessage|ToolUseSummaryMessage, Terminal>`，调用方可以 `for await (const event of query(params))` 流式消费所有事件，并在 finally 拿到 `Terminal`。

### 3.1 `QueryDeps` — I/O 依赖注入

```ts
// src/query/deps.ts
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

**Why**: 让测试不依赖 `spyOn` 单个模块（callModel / autocompact 在 6-8 个测试文件里被 spy，模式非常 boilerplate）。使用 `typeof fn` 让签名自动同步。

### 3.2 `QueryConfig` — 不可变快照

```ts
// src/query/config.ts
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean   // Statsig: tengu_streaming_tool_execution2
    emitToolUseSummaries: boolean     // Env: CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES
    isAnt: boolean                    // Env: USER_TYPE === 'ant'
    fastModeEnabled: boolean          // Env: CLAUDE_CODE_DISABLE_FAST_MODE (取反)
  }
}
```

`buildQueryConfig()` 在 `queryLoop` 入口调用一次。**Intentionally excludes `feature()` gates** — 那些是 tree-shaking 边界，必须 inline 在被 gate 的代码块以保证 DCE。

---

## 四、State — 跨轮次可变状态

```ts
type State = {
  messages: Message[]                          // 累积消息
  toolUseContext: ToolUseContext               // 工具上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 跨压缩边界的 turnId/turnCounter
  maxOutputTokensRecoveryCount: number         // max_output_tokens 恢复次数（最多 3）
  hasAttemptedReactiveCompact: boolean         // 本会话是否已尝试 reactive compact
  maxOutputTokensOverride: number | undefined  // 当前轮次的 max_output_tokens 覆盖
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined  // 上轮异步生成中的 summary
  stopHookActive: boolean | undefined          // 上轮 stop hook 阻塞过（防止重入）
  turnCount: number                            // 当前轮次
  transition: Continue | undefined             // 上一轮为何 continue（便于测试断言）
}
```

**关键设计**: 7 个 continue 站点**集中通过 `state = { ... }` 整体赋值**，而不是 9 个 `state.x = ...` 散落赋值。这样每个 continue 站点的 state 都是不可变快照，状态可观察。

**Why 单独 `taskBudgetRemaining` 不在 State 上？** 注释解释：跨 compact 边界的 `taskBudget.remaining` 计数会触碰 7 个 continue 站点，作为 loop-local `let` 可以避免继续膨胀 State。

---

## 五、queryLoop — 主循环骨架

```ts
async function* queryLoop(params, consumedCommandUuids) {
  // 入口一次性 setup
  const deps = params.deps ?? productionDeps()
  let state: State = initializeQueryLoopState(params)
  const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
  let taskBudgetRemaining: number | undefined = undefined
  const config = buildQueryConfig()
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)

  while (true) {
    // 解构 state（只读）
    let { toolUseContext } = state
    const { messages, autoCompactTracking, maxOutputTokensRecoveryCount,
            hasAttemptedReactiveCompact, maxOutputTokensOverride,
            pendingToolUseSummary, stopHookActive, turnCount } = state

    // 1. 启动本轮 skill discovery prefetch（per-iteration，有 findWritePivot guard）
    const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(null, messages, toolUseContext)

    yield { type: 'stream_request_start' }

    // 2. 准备本轮消息上下文
    const preparedTurn = await prepareQueryTurn({ ... })
    for (const m of preparedTurn.yieldedMessages) yield m
    messagesForQuery = preparedTurn.messagesForQuery
    toolUseContext = { ...toolUseContext, messages: messagesForQuery }

    // 3. blocking limit 检查（auto-compact OFF 时）
    if (!compactionResult && querySource !== 'compact' && querySource !== 'session_memory' && ...) {
      if (isAtBlockingLimit) { yield apiError; return { reason: 'blocking_limit' } }
    }

    // 4. 流式调模型
    const executionState = { assistantMessages, toolResults, toolUseBlocks, needsFollowUp, streamingToolExecutor, currentModel }
    yield* executeModelStreamingTurn({ ..., executionState, deps })

    // 5. post-sampling hook
    void executePostSamplingHooks([...messagesForQuery, ...assistantMessages], ...)

    // 6. abort 处理
    if (signal.aborted) { ...; return { reason: 'aborted_streaming' } }

    // 7. 等待上轮 pendingToolUseSummary
    if (pendingToolUseSummary) { const summary = await pendingToolUseSummary; if (summary) yield summary }

    // 8. 分支：是否需要 follow-up
    if (!needsFollowUp) {
      const { outcome, emittedEvents } = await handleNonFollowUpTurn({ ... })
      for (const e of emittedEvents) yield e
      if (outcome.kind === 'continue') { state = outcome.state; continue }
      return outcome.terminal
    }

    // 9. follow-up：执行工具 + 准备下一轮
    const followUpTurnResult = await processFollowUpTurn({ ... })
    for (const m of followUpTurnResult.yieldedMessages) yield m
    if (followUpTurnResult.terminal) return followUpTurnResult.terminal
    updatedToolUseContext = followUpTurnResult.updatedToolUseContext

    // 10. 准备下一轮
    const nextTurnPreparation = await prepareNextLoopTurn({ ... })
    for (const m of nextTurnPreparation.yieldedMessages) yield m
    if (nextTurnPreparation.terminal) return nextTurnPreparation.terminal
    state = nextTurnPreparation.nextState!
  }
}
```

### 5.1 9 个阶段的形态

| 阶段 | 函数 | 状态位置 | 返回 |
|------|------|----------|------|
| 1 | `pendingSkillPrefetch` 启动 | loop-local | Promise |
| 2 | `prepareQueryTurn()` | mutates 7 vars | `PreparedQueryTurn` |
| 3 | blocking limit 检查 | throws / yields apiError | - |
| 4 | `executeModelStreamingTurn()` | mutates `executionState` | `Terminal` |
| 5 | post-sampling hook | void | - |
| 6 | abort 处理 | yields / returns | `{ reason: 'aborted_streaming' }` |
| 7 | pendingToolUseSummary await | yields | - |
| 8a | `handleNonFollowUpTurn()` | emits events | `{ outcome, emittedEvents }` |
| 8b | `processFollowUpTurn()` | mutates 3 vars | `FollowUpTurnResult` |
| 9 | `prepareNextLoopTurn()` | mutates 2 vars | `PreparedNextLoopTurn` |

---

## 六、`prepareQueryTurn` — 准备本轮消息

5 级压缩串联：

```ts
async function prepareQueryTurn({ ... }) {
  // 1. 截取最新 compact boundary 之后的消息
  let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

  // 2. 套用工具结果预算（持久化选项取决于 querySource）
  messagesForQuery = await applyToolResultBudget(
    messagesForQuery,
    toolUseContext.contentReplacementState,
    persistReplacements ? records => void recordContentReplacement(records, agentId).catch(logError) : undefined,
    new Set(tools.filter(t => !Number.isFinite(t.maxResultSizeChars)).map(t => t.name)),
  )

  // 3. HISTORY_SNIP 压缩（粗粒度裁剪）
  if (feature('HISTORY_SNIP')) {
    const { messages, tokensFreed, boundaryMessage } = snipModule.snipCompactIfNeeded(messagesForQuery)
    messagesForQuery = messages
    snipTokensFreed = tokensFreed
    if (boundaryMessage) yieldedMessages.push(boundaryMessage)
  }

  // 4. microcompact（更细粒度）
  const microcompactResult = await deps.microcompact(messagesForQuery, toolUseContext, querySource)
  messagesForQuery = microcompactResult.messages
  pendingCacheEdits = feature('CACHED_MICROCOMPACT') ? microcompactResult.compactionInfo?.pendingCacheEdits : undefined

  // 5. CONTEXT_COLLAPSE 折叠
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    const { messages } = await contextCollapse.applyCollapsesIfNeeded(messagesForQuery, toolUseContext, querySource)
    messagesForQuery = messages
  }

  // 6. 拼装 system prompt
  const fullSystemPrompt = asSystemPrompt(appendSystemContext(systemPrompt, systemContext))

  // 7. autocompact（自动压缩）
  ;({ compactionResult, consecutiveFailures } = await deps.autocompact(
    messagesForQuery, toolUseContext,
    { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages: messagesForQuery },
    querySource, nextTracking, snipTokensFreed,
  ))

  if (compactionResult) {
    // 记录埋点、计算 taskBudgetRemaining、build postCompactMessages
    logEvent('tengu_auto_compact_succeeded', { ... })
    if (params.taskBudget) {
      const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
      nextTaskBudgetRemaining = Math.max(0, (nextTaskBudgetRemaining ?? params.taskBudget.total) - preCompactContext)
    }
    nextTracking = { compacted: true, turnId: deps.uuid(), turnCounter: 0, consecutiveFailures: 0 }
    const postCompactMessages = buildPostCompactMessages(compactionResult)
    yieldedMessages.push(...postCompactMessages)
    messagesForQuery = postCompactMessages
  } else if (consecutiveFailures !== undefined) {
    nextTracking = { ...(nextTracking ?? {...}), consecutiveFailures }
  }

  return { messagesForQuery, tracking, compactionResult, ... }
}
```

**为什么是这个顺序？**
- **snip 在 microcompact 之前**：snip 是粗粒度、可逆的边界切分，microcompact 是细粒度 token 替换；先 snip 可以减少 microcompact 要处理的消息数
- **autocompact 在最末**：作为最后兜底，只有前面三层都压不动时才触发
- **fallback 触发时 stripSignatureBlocks**：ant 构建下，model fallback 会清空签名块以避免跨模型冲突

---

## 七、`executeModelStreamingTurn` — 流式调模型

```ts
async function* executeModelStreamingTurn({ ... }) {
  let attemptWithFallback = true
  queryCheckpoint('query_api_loop_start')

  while (attemptWithFallback) {
    attemptWithFallback = false
    try {
      let streamingFallbackOccured = false
      for await (const message of deps.callModel({
        messages: prependUserContext(messagesForQuery, userContext),
        systemPrompt: fullSystemPrompt,
        thinkingConfig: toolUseContext.options.thinkingConfig,
        tools: toolUseContext.options.tools,
        signal: toolUseContext.abortController.signal,
        options: { ... 60 行 options ... },
      })) {
        // 1. 处理 streaming fallback（中途切到 fallbackModel）
        if (streamingFallbackOccured) {
          for (const msg of executionState.assistantMessages) yield { type: 'tombstone', message: msg }
          logEvent('tengu_orphaned_messages_tombstoned', { ... })
          executionState.assistantMessages.length = 0
          executionState.toolResults.length = 0
          executionState.toolUseBlocks.length = 0
          executionState.needsFollowUp = false
          executionState.streamingToolExecutor?.discard()
          executionState.streamingToolExecutor = new StreamingToolExecutor(...)
        }

        // 2. backfillObservableInput（工具输入补全）
        let yieldMessage = message
        if (message.type === 'assistant') {
          for (const block of message.message.content) {
            if (block.type === 'tool_use' && tool?.backfillObservableInput) {
              const added = tool.backfillObservableInput(inputCopy)
              if (added) clonedContent[i] = { ...block, input: inputCopy }
            }
          }
        }

        // 3. withheld 判断（contextCollapse / reactiveCompact / media / max_output_tokens）
        let withheld = false
        if (contextCollapse?.isWithheldPromptTooLong(message, isPromptTooLongMessage, querySource)) withheld = true
        if (reactiveCompact?.isWithheldPromptTooLong(message)) withheld = true
        if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) withheld = true
        if (isWithheldMaxOutputTokens(message)) withheld = true
        if (!withheld) yield yieldMessage

        // 4. 累计 assistant 消息、tool_use 块；推给 streamingToolExecutor
        if (message.type === 'assistant') {
          executionState.assistantMessages.push(message)
          const toolUseBlocks = message.message.content.filter(c => c.type === 'tool_use')
          if (toolUseBlocks.length > 0) {
            executionState.toolUseBlocks.push(...toolUseBlocks)
            executionState.needsFollowUp = true
          }
          executionState.streamingToolExecutor?.addTool(toolUseBlocks, message)
        }

        // 5. 抽取 streamingToolExecutor 已完成的结果并 yield
        for (const result of executionState.streamingToolExecutor?.getCompletedResults() ?? []) {
          if (result.message) {
            yield result.message
            executionState.toolResults.push(...normalizeMessagesForAPI([result.message], tools).filter(m => m.type === 'user'))
          }
        }
      }

      // 6. microcompact cache edits 处理
      if (feature('CACHED_MICROCOMPACT') && pendingCacheEdits) {
        const cumulativeDeleted = usage?.cache_deleted_input_tokens ?? 0
        const deletedTokens = Math.max(0, cumulativeDeleted - pendingCacheEdits.baselineCacheDeletedTokens)
        if (deletedTokens > 0) yield createMicrocompactBoundaryMessage(...)
      }
    } catch (innerError) {
      // FallbackTriggeredError 处理：切到 fallbackModel，补缺失 tool_result
      if (innerError instanceof FallbackTriggeredError && fallbackModel) {
        executionState.currentModel = fallbackModel
        attemptWithFallback = true
        yield* yieldMissingToolResultBlocks(executionState.assistantMessages, 'Model fallback triggered')
        // ... 重置 executionState ...
        if (process.env.USER_TYPE === 'ant') messagesForQuery = stripSignatureBlocks(messagesForQuery)
        logEvent('tengu_model_fallback_triggered', { ... })
        yield createSystemMessage(`Switched to ${renderModelName(...)} due to high demand`, 'warning')
        continue
      }
      throw innerError
    }
  }
}
catch (error) {
  logError(error)
  logEvent('tengu_query_error', { ... })

  // ImageSizeError / ImageResizeError → 单独的 image_error terminal
  if (error instanceof ImageSizeError || error instanceof ImageResizeError) {
    yield createAssistantAPIErrorMessage({ content: error.message })
    return { reason: 'image_error' }
  }

  // 其他 model_error：补缺失 tool_result + assistant error message
  yield* yieldMissingToolResultBlocks(executionState.assistantMessages, errorMessage)
  yield createAssistantAPIErrorMessage({ content: errorMessage })
  logAntError('Query error', error)
  return { reason: 'model_error', error }
}
```

**关键设计要点**：

1. **Streaming fallback 期间 tombstone**：如果模型中途切到 fallback，已生成的 assistant 消息打 tombstone，避免上下文出现"无对应 tool_result 的 tool_use"。

2. **Withheld 模式**：4 类错误（contextCollapse PTL、reactiveCompact PTL、media size、max_output_tokens）不在 streaming 阶段 yield 出去，而是先暂存，给后续 handleNonFollowUpTurn 一个决策窗口。

3. **backfillObservableInput**：部分工具定义 `backfillObservableInput(input)`，在 yield 之前补全 tool_use.input，让 SDK 消费者看到完整输入。

4. **`stripSignatureBlocks`（ant-only）**：fallback 时清空签名块，避免跨模型冲突。

---

## 八、`handleNonFollowUpTurn` — 非 follow-up 分支的 7 种决策

```ts
async function handleNonFollowUpTurn({ ... }) {
  // 1. withheld 413 (prompt too long) → contextCollapse.drain
  if (isWithheld413) {
    if (feature('CONTEXT_COLLAPSE') && contextCollapse && stateTransition?.reason !== 'collapse_drain_retry') {
      const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
      if (drained.committed > 0) return { outcome: { kind: 'continue', state: { transition: { reason: 'collapse_drain_retry', committed } } } }
    }
  }

  // 2. withheld 413 / media error → reactiveCompact
  if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
    const compacted = await reactiveCompact.tryReactiveCompact({ ... })
    if (compacted) return { outcome: { kind: 'continue', state: { transition: { reason: 'reactive_compact_retry' } } } }
    // 失败：发 stop failure hook，按 image_error / prompt_too_long 终止
    void executeStopFailureHooks(lastMessage, toolUseContext)
    return { outcome: { kind: 'return', terminal: { reason: 'image_error' 或 'prompt_too_long' } } }
  }

  // 3. CONTEXT_COLLAPSE 兜底
  if (feature('CONTEXT_COLLAPSE') && isWithheld413) { ... return { reason: 'prompt_too_long' } }

  // 4. withheld max_output_tokens → tengu_otk_slot_v1 决定 escalate 或 nudge
  if (isWithheldMaxOutputTokens(lastMessage)) {
    if (capEnabled && maxOutputTokensOverride === undefined && !process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS) {
      logEvent('tengu_max_tokens_escalate', { escalatedTo: ESCALATED_MAX_TOKENS })
      return { outcome: { kind: 'continue', state: { maxOutputTokensOverride: ESCALATED_MAX_TOKENS, transition: 'max_output_tokens_escalate' } } }
    }
    if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
      // 注入 meta recovery 消息："Output token limit hit. Resume directly — no apology..."
      return { outcome: { kind: 'continue', state: { ..., transition: 'max_output_tokens_recovery', attempt: ... } } }
    }
    emittedEvents.push(lastMessage)  // 用尽恢复次数，透传原错误
  }

  // 5. 其他 API error → stop failure hook + completed
  if (lastMessage?.isApiErrorMessage) { void executeStopFailureHooks(lastMessage, toolUseContext); return { reason: 'completed' } }

  // 6. Stop hook 链
  const stopHookGenerator = handleStopHooks(...)
  while (true) {
    const next = await stopHookGenerator.next()
    if (next.done) { stopHookResult = next.value; break }
    emittedEvents.push(next.value)  // 边执行边 yield stop hook 消息
  }
  if (stopHookResult.preventContinuation) return { reason: 'stop_hook_prevented' }
  if (stopHookResult.blockingErrors.length > 0) {
    return { outcome: { kind: 'continue', state: { messages: [...messagesForQuery, ...assistantMessages, ...blockingErrors], stopHookActive: true, transition: 'stop_hook_blocking' } } }
  }

  // 7. TOKEN_BUDGET 续聊
  if (feature('TOKEN_BUDGET')) {
    const decision = checkTokenBudget(budgetTracker, agentId, budget, turnOutputTokens)
    if (decision.action === 'continue') {
      incrementBudgetContinuationCount()
      return { outcome: { kind: 'continue', state: { messages: [..., createUserMessage({ content: nudgeMessage, isMeta: true })], transition: 'token_budget_continuation' } } }
    }
    if (decision.completionEvent) logEvent('tengu_token_budget_completed', { ... })
  }

  // 默认：completed
  return { outcome: { kind: 'return', terminal: { reason: 'completed' } } }
}
```

### 8.1 7 种决策的 transition reason 标识

| 决策 | `transition.reason` | 触发条件 |
|------|---------------------|---------|
| collapse drain retry | `collapse_drain_retry` | CONTEXT_COLLAPSE 启用，drain 成功 |
| reactive compact retry | `reactive_compact_retry` | reactiveCompact 成功 |
| max tokens escalate | `max_output_tokens_escalate` | tengu_otk_slot_v1 启用 |
| max tokens recovery | `max_output_tokens_recovery` | 恢复次数 < 3，注入 meta 消息 |
| stop hook blocking | `stop_hook_blocking` | stop hook 返回 blocking errors |
| token budget continuation | `token_budget_continuation` | budget tracker 决定续聊 |
| 默认 next_turn | `next_turn` | prepareNextLoopTurn 默认 |

### 8.2 Stop hook 链（`handleStopHooks`）

异步迭代器，**边执行边 yield 进度消息**：
```ts
let stopHookResult
while (true) {
  const nextResult = await stopHookGenerator.next()
  if (nextResult.done) { stopHookResult = nextResult.value; break }
  emittedEvents.push(nextResult.value)  // 透传 stop hook 进度
}
```

返回 `{ blockingErrors: Message[], preventContinuation: boolean }`：
- `preventContinuation` → 立即终止（`stop_hook_prevented`）
- `blockingErrors.length > 0` → 注入下一轮让模型处理
- 否则 → 正常继续

### 8.3 Token Budget 决策（`checkTokenBudget`）

```ts
// src/query/tokenBudget.ts
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

export function checkTokenBudget(tracker, agentId, budget, globalTurnTokens): TokenBudgetDecision {
  if (agentId || budget === null || budget <= 0) return { action: 'stop', completionEvent: null }

  const turnTokens = globalTurnTokens
  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 收益递减：连续 3+ 次续聊、delta 太小
  const isDiminishing = tracker.continuationCount >= 3 && deltaSinceLastCheck < DIMINISHING_THRESHOLD && tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // 未达 90% 且非收益递减 → 续聊
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++
    tracker.lastDeltaTokens = deltaSinceLastCheck
    tracker.lastGlobalTurnTokens = globalTurnTokens
    return { action: 'continue', nudgeMessage, continuationCount, pct, turnTokens, budget }
  }

  // 收益递减或达到 90% → 停
  if (isDiminishing || tracker.continuationCount > 0) return { action: 'stop', completionEvent: { ... diminishingReturns: isDiminishing } }

  return { action: 'stop', completionEvent: null }
}
```

**Why**: 避免无脑耗尽 token budget；连续 3 次续聊后如果每次增量 < 500 tokens 就视作收益递减，提前 stop。

---

## 九、`processFollowUpTurn` — follow-up 分支

```ts
async function processFollowUpTurn({ ... }) {
  const yieldedMessages = []
  let shouldPreventContinuation = false
  let updatedToolUseContext = toolUseContext

  // 1. 启动工具执行（流式 vs 同步）
  const toolUpdates = streamingToolExecutor
    ? streamingToolExecutor.getRemainingResults()
    : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

  for await (const update of toolUpdates) {
    if (update.message) {
      yieldedMessages.push(update.message)
      if (update.message.type === 'attachment' && update.message.attachment.type === 'hook_stopped_continuation') {
        shouldPreventContinuation = true
      }
      toolResults.push(...normalizeMessagesForAPI([update.message], tools).filter(m => m.type === 'user'))
    }
    if (update.newContext) updatedToolUseContext = { ...update.newContext, queryTracking }
  }

  // 2. 生成 tool use summary（异步 Haiku 摘要）
  let nextPendingToolUseSummary
  if (config.gates.emitToolUseSummaries && toolUseBlocks.length > 0 && !aborted && !toolUseContextAgentId) {
    // subagent 不在 mobile UI 显示，跳过
    const lastAssistantText = lastAssistantMessage?.message.content.filter(b => b.type === 'text').at(-1)?.text
    const toolInfoForSummary = toolUseBlocks.map(block => ({ name, input, output }))
    nextPendingToolUseSummary = generateToolUseSummary({ tools, signal, isNonInteractive, lastAssistantText })
      .then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
      .catch(() => null)
  }

  // 3. abort 处理
  if (aborted) {
    if (feature('CHICAGO_MCP') && !agentId) await cleanupComputerUseAfterTurn(toolUseContext).catch(() => {})
    if (signal.reason !== 'interrupt') yieldedMessages.push(createUserInterruptionMessage({ toolUse: true }))
    if (maxTurns && nextTurnCount > maxTurns) yieldedMessages.push(createAttachmentMessage({ type: 'max_turns_reached' }))
    return { ..., terminal: { reason: 'aborted_tools' } }
  }

  // 4. hook 阻止继续
  if (shouldPreventContinuation) return { ..., terminal: { reason: 'hook_stopped' } }

  return { yieldedMessages, shouldPreventContinuation, updatedToolUseContext, nextPendingToolUseSummary }
}
```

**Why stream + sync 双路径？** `StreamingToolExecutor` 在 streaming 阶段就已经开始执行工具（能拿到的结果会更早 yield）；如果 streaming 阶段没产出或 executor 不存在，兜底走同步 `runTools`。

**Why `!toolUseContextAgentId`？** 注释解释：subagent 不在 mobile UI 显示，跳过 Haiku 摘要调用，节省成本。

---

## 十、`prepareNextLoopTurn` — 下一轮准备

```ts
async function prepareNextLoopTurn({ ... }) {
  // 1. 埋点：autocompact turn 计数
  if (tracking?.compacted) {
    tracking.turnCounter++
    logEvent('tengu_post_autocompact_turn', { turnId, turnCounter, ... })
  }

  // 2. 收集 command queue → attachments
  const sleepRan = toolUseBlocks.some(b => b.name === SLEEP_TOOL_NAME)
  const isMainThread = querySource.startsWith('repl_main_thread') || querySource === 'sdk'
  const queuedCommandsSnapshot = getCommandsByMaxPriority(sleepRan ? 'later' : 'next').filter(cmd => {
    if (isSlashCommand(cmd)) return false
    if (isMainThread) return cmd.agentId === undefined
    return cmd.mode === 'agent-notification' && cmd.agentId === currentAgentId
  })
  for await (const attachment of getAttachmentMessages(null, currentToolUseContext, null, queuedCommandsSnapshot, [...messagesForQuery, ...assistantMessages, ...toolResults], querySource)) {
    yieldedMessages.push(attachment)
    toolResults.push(attachment)
  }

  // 3. 消费 pendingMemoryPrefetch
  if (pendingMemoryPrefetch?.settledAt !== null && pendingMemoryPrefetch.consumedOnIteration === -1) {
    const memoryAttachments = filterDuplicateMemoryAttachments(await pendingMemoryPrefetch.promise, currentToolUseContext.readFileState)
    for (const memAttachment of memoryAttachments) { const msg = createAttachmentMessage(memAttachment); yieldedMessages.push(msg); toolResults.push(msg) }
    pendingMemoryPrefetch.consumedOnIteration = turnCount - 1
  }

  // 4. 消费 pendingSkillPrefetch
  if (skillPrefetchModule && pendingSkillPrefetch) {
    const skillAttachments = await skillPrefetchModule.collectSkillDiscoveryPrefetch(pendingSkillPrefetch)
    for (const att of skillAttachments) { const msg = createAttachmentMessage(att); yieldedMessages.push(msg); toolResults.push(msg) }
  }

  // 5. 消费 command queue（started lifecycle）
  const consumedCommands = queuedCommandsSnapshot.filter(cmd => cmd.mode === 'prompt' || cmd.mode === 'agent-notification')
  if (consumedCommands.length > 0) {
    for (const cmd of consumedCommands) {
      if (cmd.uuid) { consumedCommandUuids.push(cmd.uuid); notifyCommandLifecycle(cmd.uuid, 'started') }
    }
    removeFromQueue(consumedCommands)
  }

  // 6. 刷新工具列表
  if (currentToolUseContext.options.refreshTools) {
    const refreshedTools = currentToolUseContext.options.refreshTools()
    if (refreshedTools !== currentToolUseContext.options.tools) {
      currentToolUseContext = { ...currentToolUseContext, options: { ...currentToolUseContext.options, tools: refreshedTools } }
    }
  }

  // 7. BG_SESSIONS agent summary
  if (feature('BG_SESSIONS') && !agentId && taskSummaryModule.shouldGenerateTaskSummary()) {
    taskSummaryModule.maybeGenerateTaskSummary({ systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages: [...] })
  }

  // 8. maxTurns 检查
  if (maxTurns && nextTurnCount > maxTurns) {
    yieldedMessages.push(createAttachmentMessage({ type: 'max_turns_reached', maxTurns, turnCount: nextTurnCount }))
    return { ..., terminal: { reason: 'max_turns', turnCount: nextTurnCount } }
  }

  // 9. 异步生成 tool use summary
  // （同 processFollowUpTurn 中的逻辑）

  // 10. 构造 nextState
  return {
    yieldedMessages,
    updatedToolUseContext,
    nextState: {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      toolUseContext: { ...currentToolUseContext, queryTracking },
      autoCompactTracking: tracking,
      turnCount: nextTurnCount,
      maxOutputTokensRecoveryCount: 0,
      hasAttemptedReactiveCompact: false,
      pendingToolUseSummary: nextPendingToolUseSummary,
      maxOutputTokensOverride: undefined,
      stopHookActive,
      transition: { reason: 'next_turn' },
    },
    pendingToolUseSummary: nextPendingToolUseSummary,
  }
}
```

**为什么 command 过滤有 3 种模式？**
- `!isSlashCommand(cmd)`：排除 `/xxx` 命令（这些由 processSlashCommand 处理，不进 query loop）
- `isMainThread` 主线程：只看 `cmd.agentId === undefined` 的（即发到主线程的）
- 非主线程：只看发给当前 agentId 的 task-notification

**Why SleepTool 影响 priority？** 注释解释：sleep 期间排到 'later' 队列的工具，sleep 结束后才放行。

---

## 十一、`query` 公共入口

```ts
export async function* query(params): AsyncGenerator<...> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // Only reached if queryLoop returned normally. Skipped on throw (error
  // propagates through yield*) and on .return() (Return completion closes
  // both generators). This gives the same asymmetric started-without-completed
  // signal as print.ts's drainCommandQueue when the turn fails.
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**关键不变量**：
- `consumedCommandUuids` 在 queryLoop 内部被 `prepareNextLoopTurn` 收集（每个 consumed command uuid push 进去并 notify started）
- 只有 queryLoop **正常返回** 时才发 'completed' lifecycle
- 如果 queryLoop 抛错或被 .return() 关闭 → 不会发 completed，保留 "started but not completed" 的非对称信号（与 `print.ts drainCommandQueue` 一致）

---

## 十二、调用方

### 12.1 SDK / 头less 路径（`src/QueryEngine.ts` + `src/cli/print.ts`）

`QueryEngine` 是 query 的轻量封装层，给 SDK 暴露 `submitMessage(prompt, options)` 异步迭代器。每轮 turn = 一次 `query(params)` 调用。`print.ts` 驱动器通过 `runHeadless` 间接使用。

### 12.2 REPL 路径（`src/screens/REPL.tsx`）

REPL 在用户提交消息后构造 `queryParams`，并通过 `startBackgroundSession` 走 `query()` 异步迭代器。**关键代码**（REPL.tsx:2591）：
```ts
startBackgroundSession({
  messages: [...messagesRef.current, ...uniqueNotifications],
  queryParams: {
    systemPrompt, userContext, systemContext, canUseTool, toolUseContext,
    querySource: getQuerySourceForREPL()
  },
  description: terminalTitle,
  setAppState,
  agentDefinition: mainThreadAgentDefinition
})
```

### 12.3 Fork agent / compact agent（`src/services/compact/autoCompact.ts`）

`autoCompactIfNeeded` 内部会 fork 一个 compact agent，调用 `queryModelWithoutStreaming`（不是 `query()` 本身）。这是单独的 stream-less API 调用路径。

### 12.4 ApiQueryHook（`src/utils/hooks/apiQueryHookHelper.ts`）

轻量级 model 调用包装，给 hook 系统使用：
```ts
const response = await queryModelWithoutStreaming({ ... })
```

不走 query loop 也不走 streaming——直接用非流式 API。

---

## 十三、Terminal 终止原因

| Reason | 触发位置 | 含义 |
|--------|---------|------|
| `completed` | handleNonFollowUpTurn 默认 / stop failure hook | 正常结束 |
| `aborted_streaming` | queryLoop 阶段 6 | streaming 阶段被 abort |
| `aborted_tools` | processFollowUpTurn | 工具执行阶段被 abort |
| `blocking_limit` | queryLoop 阶段 3 | 命中硬 token 上限 |
| `image_error` | executeModelStreamingTurn catch / handleNonFollowUpTurn | 图片尺寸/调整错误 |
| `model_error` | executeModelStreamingTurn catch | 通用 model error |
| `prompt_too_long` | handleNonFollowUpTurn | 压缩失败 |
| `stop_hook_prevented` | handleNonFollowUpTurn | stop hook 阻止 |
| `hook_stopped` | processFollowUpTurn | hook 阻止继续 |
| `max_turns` | prepareNextLoopTurn | 达到 maxTurns |

---

## 十四、关键设计原则

1. **State 不可变 + 集中 Continue 站点**  
   7 个 continue 站点都通过 `state = { ... }` 整体赋值，避免 9 个 `state.x = ...` 散落。State 是不可变快照，便于测试断言。

2. **副作用 yield 化**  
   所有用户可见的事件（streaming 消息、tool 结果、stop hook 进度、post-sampling hook）都通过 `yield` 暴露给调用方，调用方可以 `for await` 收集。

3. **5 级压缩串联**  
   snip → microcompact → CONTEXT_COLLAPSE → autocompact → reactiveCompact，按"粗到细、可逆到不可逆"排序。

4. **Withheld 模式**  
   4 类错误（PTL / media / max_output_tokens / CONTEXT_COLLAPSE PTL）在 streaming 阶段先 withhold，给后续 handleNonFollowUpTurn 一个决策窗口（reactive compact 二次压缩、escalate、recovery message）。

5. **Streaming fallback 期间 tombstone**  
   模型中途切 fallbackModel 时，已生成的 assistant 消息打 tombstone，避免上下文出现"无对应 tool_result 的 tool_use"。

6. **Deps 注入 + 不可变 Config 快照**  
   `QueryDeps` 注入 4 个 I/O（callModel / microcompact / autocompact / uuid），`QueryConfig` 快照 4 个 gate。测试时直接传 fake，避免 spyOn 散落。

7. **taskBudget.remaining 跨 compact 累计**  
   每次 autocompact 触发时，从 total 减去 preCompactContext；后续每次 compact 累计扣减。

8. **subagent 跳过 tool use summary + stop hook**  
   `toolUseContextAgentId` 存在时跳过 Haiku 摘要和 stop hook 链（subagent 不在 mobile UI 显示）。

9. **command lifecycle 不对称**  
   只有 queryLoop 正常返回时发 'completed'，throw 或 .return() 时不发——保留 "started but not completed" 信号。

---

## 十五、修改指南

| 想做的事 | 改哪里 |
|---------|--------|
| 新增 continue reason | 在 handleNonFollowUpTurn 或 prepareNextLoopTurn 写 `state: { transition: { reason: 'xxx' } }` |
| 新增压缩层级 | 在 prepareQueryTurn 的 5 级串联中插入；保证顺序：粗到细 |
| 调整 fallback 逻辑 | executeModelStreamingTurn 的 catch (FallbackTriggeredError) |
| 调整 stop hook 行为 | handleNonFollowUpTurn 阶段 6 |
| 调整续聊策略 | tokenBudget.ts 的 COMPLETION_THRESHOLD / DIMINISHING_THRESHOLD |
| 新增 I/O 依赖 | QueryDeps + productionDeps + queryLoop 解构 |
| 调整 abort 处理 | queryLoop 阶段 6 + processFollowUpTurn 阶段 3 |
| 调整 task budget 累计 | prepareQueryTurn autocompact 分支 + prepareQueryTurn reactive compact 分支 |
| 新增 hook 类型 | postSamplingHooks / stopHooks / apiQueryHookHelper |

**调试技巧**：
- `queryCheckpoint('xxx')` 在关键节点埋点，配合 queryProfiler 查看
- `headlessProfilerCheckpoint('query_started')` 在头less 模式下记录 query 入口
- 在 deps 注入 fake 单元测试：`query({ ...params, deps: { callModel: fakeCall, microcompact: fakeMicro, autocompact: fakeAuto, uuid: () => 'test-uuid' } })`
- 检查 `state.transition.reason` 确认哪条恢复路径触发
- 监控 `tengu_query_error` 事件查看 model_error 上下文
- 监控 `tengu_post_autocompact_turn` 事件看 autocompact 后 turn 计数

---

## 十六、性能与可观测性

### 16.1 关键埋点

| 事件名 | 触发时机 | 字段 |
|--------|---------|------|
| `tengu_auto_compact_succeeded` | autocompact 成功 | pre/post token count, queryChainId, queryDepth |
| `tengu_post_autocompact_turn` | autocompact 后 turn 计数 | turnId, turnCounter, queryChainId |
| `tengu_orphaned_messages_tombstoned` | streaming fallback | orphanedMessageCount, queryChainId |
| `tengu_model_fallback_triggered` | FallbackTriggeredError 触发 | original_model, fallback_model, queryChainId |
| `tengu_query_error` | 通用 model error | assistantMessages count, toolUses count, queryChainId |
| `tengu_query_before_attachments` / `tengu_query_after_attachments` | 附件收集前后 | counts, queryChainId |
| `tengu_token_budget_completed` | token budget 用尽 | continuationCount, pct, diminishingReturns, durationMs |
| `tengu_max_tokens_escalate` | ESCALATED_MAX_TOKENS 触发 | escalatedTo |
| `tengu_streaming_tool_execution_used` / `tengu_streaming_tool_execution_not_used` | 工具执行模式 | tool_count, queryChainId |

### 16.2 queryProfiler 埋点

```ts
queryCheckpoint('query_api_loop_start')
queryCheckpoint('query_api_streaming_start')
queryCheckpoint('query_api_streaming_end')
queryCheckpoint('query_fn_entry')
queryCheckpoint('query_setup_start')
queryCheckpoint('query_setup_end')
queryCheckpoint('query_tool_execution_start')
queryCheckpoint('query_tool_execution_end')
queryCheckpoint('query_recursive_call')
```

### 16.3 chainId 跨 sub-agent 嵌套

```ts
const queryTracking = toolUseContext.queryTracking
  ? { chainId: toolUseContext.queryTracking.chainId, depth: toolUseContext.queryTracking.depth + 1 }
  : { chainId: deps.uuid(), depth: 0 }
```

- 同一 chainId = 同一逻辑任务
- depth = sub-agent 嵌套深度
- 跨 sub-agent 嵌套时 chainId 不变，只递增 depth
- 所有埋点事件都带 chainId + depth，便于后端聚合分析

---

## 十七、与 QueryEngine 的关系

`src/QueryEngine.ts` 是 query 的**轻量封装层**，目标是为 SDK 暴露更简单的 `submitMessage(prompt)` API。区别：

| 维度 | `query()` | `QueryEngine` |
|------|-----------|---------------|
| 形态 | async generator 函数 | class（每次 submitMessage = 一次 query 调用） |
| 状态 | 函数内 let state | 实例字段（mutableMessages, totalUsage, permissionDenials） |
| 适用 | 一次性 / 短期任务 | SDK 长生命周期会话 |
| 驱动 | REPL 主动 startBackgroundSession | SDK 端 submitMessage |
| 流式 | 默认 streaming | 走 streaming + includePartialMessages |

二者**共享** deps 注入、transition reason 模式、continue 站点集中化等设计。
