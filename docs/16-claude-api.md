# claude.ts — Anthropic API 调用核心

> 源码位置：`src/services/api/claude.ts`（3811 行）
> 调用方：`src/query.ts`（主 query 循环）、`src/services/PromptSuggestion/*`、`src/services/api/apiQueryHookHelper.ts`（API query hook）、`src/utils/sideQuery.ts`、`src/services/api/dumpPrompts.ts`
> 下游：`src/services/api/client.ts`（SDK client）、`src/services/api/withRetry.ts`（重试逻辑）、`src/services/api/errors.ts`（错误分类）、`src/services/api/logging.ts`（API 日志）、`src/services/api/promptCacheBreakDetection.ts`、`src/services/api/VCR.ts`、`src/services/compact/microCompact.ts`（cached microcompact）

`claude.ts` 是 Claude Code 与 Anthropic API 交互的"中枢神经"。它定义了 60+ 函数，覆盖从请求构造、消息规范化、流式事件处理、错误恢复、缓存断点管理、usage 累加、KV cache TTL 决策的完整链路。核心入口 `queryModel` 把这些职责拆分成 6 个子方法（context 准备 → 请求参数构造 → watchdog → 事件分发 → 完成处理 → 错误恢复），用 `QueryModelRequestContext`（不可变）+ `QueryModelRuntimeState`（可变）两个对象传递状态。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 核心编排入口 | `queryModel`（async generator，编排 5 阶段 + 6 子方法）+ `queryModelWithStreaming`（流式）/ `queryModelWithoutStreaming`（非流式，最终 AssistantMessage）/ `queryHaiku`（小型 prompt 调用，固定 Haiku 模型 + 无工具）/ `queryWithModel`（一次性调用，走完整 query pipeline） |
| 请求上下文准备 | `prepareQueryModelContext` 15 步流程：off-switch 检查、模型解析（Bedrock inference profile）、betas 计算、advisor 模型初始化、tool search 决策、动态工具加载、cached MC 配置、global cache 策略、tool schemas、消息规范化、media 截断、fingerprint、deferred tools 注入、system prompt 拼装、cache break detection、LLM span |
| 请求参数构造 | `buildQueryModelRequestParams`：1M beta（动态追加）、Bedrock beta 走 extraBodyParams、output_config（effort / task_budget / format）、max_tokens（retryContext > options > 模型默认）、adaptive vs budget thinking、context_management、prompt caching、fast mode（动态，不影响 cache key）、AFK mode（latched + isAgenticQuery）、cache editing（latched） |
| 流式 idle watchdog | `createStreamIdleWatchdog`：`CLAUDE_ENABLE_STREAM_WATCHDOG=true` 启用，默认 90s；45s 警告、90s 触发 abort + 触发非流式 fallback + Statsig 事件 + 诊断日志 |
| 流式事件分发 | `handleStreamEvent`：5 种事件（message_start / content_block_start / content_block_delta / content_block_stop / message_delta）+ 5 种 block（text / thinking / tool_use / server_tool_use / connector_text / advisor_tool_result）+ delta 类型（input_json / text / signature / connector_text / citations） |
| 流式完成校验 | `processStreamCompletion`：检查 watchdog 触发（抛错让 fallback 接管）、检查空响应（message_start 未到 / content_block 未完成 → 触发非流式 fallback） |
| 流式错误恢复 | `handleStreamingError`：区分用户 abort（ESC）与 SDK 超时、检查 fallback 是否禁用、调 `executeNonStreamingRequest` 重试、通知上层 `onStreamingFallback`（让 query.ts 把已 yield 的 assistant 打 tombstone） |
| 全局错误处理 | `handleQueryModelError`：429 / 5xx / 网关错误 / 解析错误的分类处理；off-switch 二次检查；`adjustParamsForNonStreaming` 把 max_tokens 截断到 64k；`getAssistantMessageFromError` 把异常转 AssistantMessage |
| 成功日志 | `logQueryModelSuccess`：markToolsSentToAPIState、setLastMainRequestId（仅主会话链）、logAPISuccessAndDuration（fire-and-forget，预先计算标量避免闭包 pin 数组）、兜底释放资源 |
| 非流式执行器 | `executeNonStreamingRequest`：调 `withRetry` + `getAnthropicClient`（maxRetries=0，禁用 SDK 自动重试）+ `adjustParamsForNonStreaming`（max_tokens ≤ 64k，thinking budget 同步截断）+ API timeout（API_TIMEOUT_MS 覆盖，默认 300s，remote 120s）+ 失败重试 withRetry + 529 预算累计 + 错误埋点 |
| 流式资源清理 | `cleanupStream` / `releaseStreamResources`：abort SDK stream controller + cancel Response body（双保险，避免 TLS socket 泄漏） |
| Usage 累加 | `updateUsage`（流式 delta 累加；input_tokens / cache tokens 防御 0 覆盖；cache_deleted_input_tokens 仅 cachedMC feature 启用）+ `accumulateUsage`（跨 turn 累加） |
| KV cache 决策 | `getPromptCachingEnabled`（环境变量控制 4 种粒度 disable）+ `getCacheControl`（type: 'ephemeral' + 可选 ttl: '1h' + 可选 scope: 'global'）+ `should1hCacheTTL`（用户资格 latch + GrowthBook allowlist + Bedrock 3P env 旁路） |
| 缓存断点管理 | `addCacheBreakpoints`：每个请求仅 1 个 message-level marker（Mycro page_manager 立即释放非边界 locals）；fire-and-forget forks 时把 marker 移到倒数第二条（共享前缀 no-op merge，不留 KVCC tail） |
| Media 截断 | `stripExcessMediaItems`：超过 API_MAX_MEDIA_PER_REQUEST 时按最旧优先剥；同步处理 tool_result 嵌套的 media |
| Message 转换 | `userMessageToMessageParam` / `assistantMessageToMessageParam`：加 cache_control（最后一条消息）、克隆数组内容防止 splice 污染 |
| 努力度与 task budget | `configureEffortParams`（model supports effort 时发送 EFFORT_BETA_HEADER，ant-only numeric 走 anthropic_internal.effort_override）+ `configureTaskBudgetParams`（仅 1P 启用，Beta Stainless SDK 暂未含 task_budget 用 cast） |
| 抗蒸馏 | `getExtraBodyParams`：ANTI_DISTILLATION_CC + 1P CLI + GrowthBook 启用 → `anti_distillation: ['fake_tools']` |
| Metadata | `getAPIMetadata`：user_id（含 device_id / account_uuid / session_id） + 用户的 CLAUDE_CODE_EXTRA_METADATA 合并 |
| API key 验证 | `verifyApiKey`：用 small fast model 跑 1 个 max_tokens=1 的请求；非交互模式跳过；带 retry；判 401 返回 false |
| 状态共享 | `QueryModelRequestContext`（不可变 28 字段：messages / model / betas / advisorModel / toolSearch / cachedMC / globalCache / system / tools / effort / span 等）+ `QueryModelRuntimeState`（可变 25 字段：stream / message / usage / costUSD / stopReason / start / attempt / fallback / watchdog 等） |
| Off-switch | `getDynamicConfig_BLOCKS_ON_INIT('tengu-off-switch')`：非订阅用户用非自定义 Opus 时，由 GrowthBook 控制是否提前 yield 终止消息 |
| Beta 动态追加 | 1M（仅 Sonnet 1M experiment 启用时）/ fast mode（latched）/ AFK mode（latched + 1P + isAgenticQuery）/ cache editing（latched + 1P + main thread）/ tool search（按 provider 不同） |
| Off-switch 返回 | yield `getAssistantMessageFromError(new Error(CUSTOM_OFF_SWITCH_MESSAGE), model)` 让 query.ts 看到终止消息 |
| Latch 机制 | `setAfkModeHeaderLatched` / `setFastModeHeaderLatched` / `setCacheEditingHeaderLatched` / `setThinkingClearLatched` —— 第一次发送后保持 sticky-on；`/clear` / `/compact` 时清空（避免 mid-session toggle 改 server-side cache key 浪费 50-70K tokens） |
| Adaptive thinking | 模型支持时 always `type: 'adaptive'`（无 budget）；不支持时回退 budget_tokens（min(max_tokens-1, 模型默认 / user 配置)） |

---

## 核心执行流程

`queryModel` 把 query 流程拆成 5 个阶段，通过 6 个子方法协作。整体结构：

```text
                ┌──────────────────────────────────────────────────┐
                │ queryModel(messages, systemPrompt, ...)        │
                │ AsyncGenerator<StreamEvent | AssistantMessage   │
                │               | SystemAPIErrorMessage>          │
                └─────────────────────────┬────────────────────────┘
                                          │
        阶段 1: 准备请求上下文
        ├─ prepareQueryModelContext (子方法 1)
        │  ├─ Off-switch 检查（GrowthBook tengu-off-switch）
        │  ├─ 模型解析（Bedrock inference profile）
        │  ├─ Betas 基础列表（getMergedBetas + advisor）
        │  ├─ Advisor 模型（实验覆盖 + modelSupport 校验）
        │  ├─ Tool search 启用 + deferred tool 集合
        │  ├─ 过滤工具列表（deferred tools 仅 discovered）
        │  ├─ Cached MC 配置（feature gated）
        │  ├─ Global cache 策略（needsToolBasedCacheMarker）
        │  ├─ Tool schemas（deferLoading）
        │  ├─ normalizeMessagesForAPI + stripToolReference + stripCallerField
        │  ├─ ensureToolResultPairing（orphan tool_use/tool_result 修复）
        │  ├─ stripAdvisorBlocks（无 advisor beta 时）
        │  ├─ stripExcessMediaItems（>100 media 剥最旧）
        │  ├─ Fingerprint 计算（attribution 用）
        │  ├─ Deferred tools 注入（可用 <available-deferred-tools> 块）
        │  ├─ Chrome tool-search instructions 注入（仅 useToolSearch + 非 delta attachment）
        │  ├─ System prompt 拼装（attribution + CLI prefix + advisor + chrome）
        │  ├─ enablePromptCaching + buildSystemPromptBlocks
        │  ├─ All tools（含 advisor server tool 追加在 cache_control 后）
        │  ├─ Fast mode 决策（4 重 gate）
        │  ├─ Latch 决策（AFK / fast / cache editing / thinking clear）
        │  ├─ Effort resolveAppliedEffort
        │  ├─ recordPromptState（PROMPT_CACHE_BREAK_DETECTION 守护下）
        │  ├─ startLLMRequestSpan（beta tracing）
        │  └─ 返回 QueryModelRequestContext 或 { offSwitchTriggered: true }
        │
        ├─ 若 off-switch 触发 → yield CUSTOM_OFF_SWITCH_MESSAGE 终止 + return
        │
        ▼
        阶段 2: 初始化 RuntimeState + fire-and-forget 开始日志
        ├─ state = QueryModelRuntimeState { stream, ttftMs, usage, ... }
        └─ void options.getToolPermissionContext().then(...) logAPIQuery(...)
                                          │
        ▼
        阶段 3: withRetry + 流式调用
        ├─ generator = withRetry(() => getAnthropicClient({...}),
        │                          async (anthropic, attempt, context) => {
        │                            state.attemptNumber = attempt
        │                            state.start = Date.now()
        │                            queryCheckpoint('query_api_request_sent')
        │                            state.clientRequestId = ... (1P only)
        │                            params = buildQueryModelRequestParams (子方法 2)
        │                            captureAPIRequest(params, querySource)
        │                            result = await anthropic.beta.messages.create({...params, stream: true})
        │                            .withResponse() → 获取 stream + response + requestId
        │                            state.stream = stream
        │                            state.streamResponse = response
        │                            state.streamRequestId = requestId
        │                            watchdog = createStreamIdleWatchdog (子方法 3)
        │
        │                            for await (const part of stream) {
        │                              watchdog.reset()
        │                              yield* handleStreamEvent (子方法 4)
        │                            }
        │
        │                            processStreamCompletion (子方法 5)
        │                            → 校验完成（watchdog 触发 / 空响应）
        │                          })
        │
        ├─ try: for await (const event of generator) yield event
        │
        ▼
        阶段 4: 错误恢复（handleStreamingError 子方法 6 + handleQueryModelError）
        ├─ catch (streamingError):
        │  ├─ handleStreamingError（流式失败 → 非流式 fallback 或 throw）
        │  └─ handleQueryModelError（兜底：off-switch / adjustParams / 错误归类 / assistant message）
        │
        ├─ finally:
        │  ├─ endQueryProfile
        │  ├─ releaseStreamResources
        │  └─ endLLMRequestSpan + recordCacheBreakCheck
                                          │
        ▼
        阶段 5: 成功日志（仅未 throw 时）
        └─ logQueryModelSuccess
           ├─ markToolsSentToAPIState (cachedMC 启用时)
           ├─ setLastMainRequestId (主会话链)
           ├─ logAPISuccessAndDuration (fire-and-forget)
           └─ releaseStreamResources (双保险)
```

### `handleStreamEvent` 流式事件分发（5 事件 × 5 block 类型）

```text
stream event:
  ├─ 'message_start'
  │  → state.partialMessage = message
  │  → state.ttftMs = Date.now() - state.start
  │  → state.usage = updateUsage(state.usage, message.usage)
  │  → state.research (ant-only)
  │
  ├─ 'content_block_start'
  │  ├─ tool_use → state.contentBlocks[index] = {...block, input: ''}
  │  ├─ server_tool_use → {...block, input: ''}
  │  │  └─ name === 'advisor' → state.isAdvisorInProgress = true + 埋点
  │  ├─ text → {...block, text: ''} (SDK 偶尔发重复，置空避免)
  │  ├─ thinking → {...block, thinking: '', signature: ''}
  │  └─ other (含 advisor_tool_result) → state.isAdvisorInProgress = false + 埋点
  │
  ├─ 'content_block_delta'
  │  ├─ connector_text_delta → contentBlock.connector_text += delta.connector_text
  │  ├─ citations_delta → TODO (未来支持)
  │  ├─ input_json_delta → contentBlock.input += delta.partial_json
  │  ├─ text_delta → contentBlock.text += delta.text
  │  ├─ signature_delta → contentBlock.signature = delta.signature
  │  └─ thinking_delta → contentBlock.thinking += delta.thinking
  │
  ├─ 'content_block_stop'
  │  ├─ 检查 SDK 兼容性 bug（anthropic_internal + standard tool 并存）
  │  ├─ 构造 AssistantMessage { content, model, usage, stop_reason, ... }
  │  ├─ normalizeContentFromAPI (修正顺序 + 类型)
  │  ├─ stripAdvisorBlocks (无 advisor beta 时)
  │  ├─ state.research 写回（ant-only）
  │  ├─ checkResponseForCacheBreak (cachedMC 启用时)
  │  ├─ state.newMessages.push(message)
  │  ├─ state.usage / costUSD 累加
  │  └─ yield message
  │
  └─ 'message_delta'
     ├─ state.usage = updateUsage(state.usage, delta.usage)
     ├─ state.stopReason = delta.stop_reason
     ├─ state.costUSD = calculateUSDCost(state.usage, state.partialMessage.model)
     ├─ 检测 refusal / context_window_exceeded / max_tokens → 抛错或 yield 错误
     └─ write stopReason 回 state.newMessages[last] (mutation 而非替换)
```

### 流式错误恢复（`handleStreamingError` → `executeNonStreamingRequest`）

```text
流式失败 (catch in queryModel)
  │
  ├─ handleStreamingError:
  │  ├─ APIUserAbortError + signal.aborted → throw (用户主动 ESC)
  │  ├─ APIUserAbortError + !signal.aborted → throw APIConnectionTimeoutError (SDK 内部超时)
  │  │
  │  ├─ 检查 fallback 是否禁用
  │  │  ├─ CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK env var
  │  │  ├─ GrowthBook tengu_disable_streaming_to_non_streaming_fallback
  │  │  └─ 若禁用 → 埋点 + throw
  │  │
  │  └─ 允许 fallback:
  │     ├─ state.didFallBackToNonStreaming = true
  │     ├─ options.onStreamingFallback()  (通知 query.ts 把已 yield 的 assistant 打 tombstone)
  │     ├─ 埋点 tengu_streaming_fallback_to_non_streaming + tengu_nonstreaming_fallback_started
  │     └─ yield* executeNonStreamingRequest(clientOptions, retryOptions, paramsFromContext, ...)
  │        ├─ withRetry(() => getAnthropicClient({maxRetries: 0}))
  │        ├─ adjustParamsForNonStreaming(params, MAX_NON_STREAMING_TOKENS=64000)
  │        │  ├─ max_tokens = min(params.max_tokens, 64000)
  │        │  └─ 若 thinking.budget_tokens > max_tokens - 1 → 截断
  │        ├─ anthropic.beta.messages.create({...}, {signal, timeout})
  │        │  ├─ fallbackTimeoutMs = API_TIMEOUT_MS override | remote=120s | default=300s
  │        │  └─ 失败 → 埋点 tengu_nonstreaming_fallback_error + throw (让 withRetry 重试)
  │        ├─ 529 错误累计（让总 529s-before-fallback 计数一致）
  │        └─ 返回 AssistantMessage + yield SystemAPIErrorMessage（中间状态消息）
  │
  └─ 仍失败 → handleQueryModelError:
     ├─ 二次 off-switch 检查
     ├─ APIConnectionTimeoutError → adjustParamsForNonStreaming 重试
     ├─ 429 / 5xx → adjustParams 减少 max_tokens 后重试
     ├─ 网关错误 → 直接重试
     ├─ 解析错误 → getAssistantMessageFromError 转 AssistantMessage
     └─ 未分类 → throw (让外层兜底)
```

---

## 关键设计取舍

- **6 个子方法 + ctx/state 双对象传递**：避免散落几十个 let 变量；ctx 不可变（除 betas 在 buildRequestParams 中追加），state 可变（流式资源 / usage / costUSD / watchdog 状态等）。
- **流式 vs 非流式 fallback**：流式失败时调 `executeNonStreamingRequest`，`max_tokens` 截断到 64k（API 非流式上限），thinking budget 同步截断（保持 `max_tokens > budget_tokens` 约束）；client 端设 `maxRetries: 0` 禁用 SDK 自动重试，统一走 withRetry。
- **Single cache marker per request**：Mycro page_manager turn-to-turn eviction 只在 `cache_store_int_token_boundaries` 之外的位置释放 local-attention KV pages。两个 markers 会让倒数第二个位置被保护，多存活一个 turn —— 单 marker 让它们立即释放。fire-and-forget forks（`skipCacheWrite`）把 marker 移到倒数第二条：共享前缀 no-op merge，不留 KVCC tail。
- **Input tokens 防御 0 覆盖**：`message_delta` 可能发 `input_tokens: 0`，不能覆盖 `message_start` 设的真实值 —— `updateUsage` 用 `> 0` 守卫保留旧值。
- **Sticky-on latch 机制**：AFK / fast / cache editing / thinking clear 四个 beta header 一旦首次发送就保持 —— mid-session toggle 不会改 server-side cache key，浪费 50-70K tokens；`/clear` / `/compact` 时清空。Per-call gate（isAgenticQuery, querySource）保持 per-call 让非 agentic query 维持自己的稳定 header 集合。
- **Fast mode 动态决策（不影响 cache key）**：speed='fast' 由当前 retry context 决定（cooldown 可抑制），但 fast mode beta header 一旦 latch 就保持 —— speed 变化在 server-side 是无 key 影响，header 变化却会 bust cache。
- **Adaptive vs budget thinking**：模型支持 adaptive 时 always `type: 'adaptive'`（无 budget），不支持时回退 budget_tokens（min(max_tokens-1, 默认/用户配置)）。Adaptive 让模型自主决定预算，避免手动设置偏差。
- **Tool search 仅 discovered deferred tools**：动态工具加载让模型按需通过 `tool_reference` 块发现工具，避免预声明所有 deferred tools + 解除 tool quantity 限制。`extractDiscoveredToolNames` 从消息历史提取已 discovered 的工具名。
- **Global cache scope 策略**：MCP 工具是 per-user → dynamic tool section → 不能全局缓存。当 global cache 启用但有 non-deferred MCP 工具时，`needsToolBasedCacheMarker=true` 让 system prompt 不带 cache_control（避免 cache break）。
- **Off-switch 双层检查**：prepareQueryModelContext 检查一次 + handleQueryModelError 检查一次 —— 即使 GrowthBook 在请求中更新也来得及响应。
- **Watermark idle watchdog**：默认关闭（不破坏现有 happy path），仅 `CLAUDE_ENABLE_STREAM_WATCHDOG=true` 启用；触发后立即 abort 流，让 catch 走非流式 fallback（避免会话无限挂起）。
- **Adaptive thinking 与 thinking_clear 协调**：`clearAllThinking` 由 `setThinkingClearLatched` 决定（仅当 last API completion > 1h 前）—— 长时间未交互后重新启动 turn 时清掉 thinking（避免 stale thinking 干扰新 turn）。
- **Capped default max tokens**：slot-reservation cap 默认 8k（BQ p99 output = 4,911 tokens，32k/64k defaults over-reserve 8-16×）；命中 cap 时 query.ts 一次 64k retry (`max_output_tokens_escalate`)。`Math.min` 保留模型原生低默认值（如 claude-3-opus 4k）。
- **Client request ID 仅 1P**：3P providers 不记录 client request id（inc-4029 class），只为 firstParty + 1P base url 生成 randomUUID；流式 timeout 拿不到 server request id 时仍能关联服务端日志。
- **Tool result content cloning**：`userMessageToMessageParam` 克隆数组内容防止 splice 污染原始 message（`insertCacheEditsBlock` 多次调用会共享同一数组）。
- **`releaseStreamResources` 双保险**：try/catch finally 已释放一次，`logQueryModelSuccess` 再释放一次 —— 防止 finally 之前抛错导致资源泄漏。

---

## 调用与被调用全景

```text
query.ts 主 query 循环
  ├─ queryModelWithStreaming（流式）
  │  └─ withStreamingVCR(messages, async function*() {
  │       yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  │     })
  │
  ├─ queryModelWithoutStreaming（非流式，最终 AssistantMessage）
  │  └─ withStreamingVCR(messages, async function*() {
  │       yield* queryModel(...)
  │     }) → 消费流取最后 assistant message
  │
  └─ 内部子调用:
     ├─ apiQueryHookHelper.ts → queryModelWithoutStreaming（用于 API query hook 抽样）
     ├─ sideQuery.ts → queryModel（部分场景）
     └─ dumpPrompts.ts → verifyApiKey（验证 API key 有效性）

queryHaiku (小型 prompt 调用)
  └─ withVCR(messages, async () => queryModelWithoutStreaming({
       model: getSmallFastModel(), thinking: disabled, tools: []
     }))

queryWithModel (一次性 model 调用)
  └─ withVCR(messages, async () => queryModelWithoutStreaming({...}))

queryModel 主流程
  ├─ 阶段 1: prepareQueryModelContext
  │  ├─ getDynamicConfig_BLOCKS_ON_INIT (off-switch)
  │  ├─ getInferenceProfileBackingModel (Bedrock)
  │  ├─ getMergedBetas (model betas + advisor)
  │  ├─ getExperimentAdvisorModels / modelSupportsAdvisor
  │  ├─ isToolSearchEnabled / extractDiscoveredToolNames
  │  ├─ getCachedMCConfig (cachedMC feature gated)
  │  ├─ shouldUseGlobalCacheScope
  │  ├─ Promise.all(tools.map(toolToAPISchema))
  │  ├─ normalizeMessagesForAPI + stripToolReference + stripCallerField
  │  ├─ ensureToolResultPairing
  │  ├─ stripAdvisorBlocks (无 advisor beta)
  │  ├─ stripExcessMediaItems
  │  ├─ computeFingerprintFromMessages
  │  ├─ formatDeferredToolLine + createUserMessage prepend
  │  ├─ getAttributionHeader + getCLISyspromptPrefix + ADVISOR_TOOL_INSTRUCTIONS + CHROME_*
  │  ├─ buildSystemPromptBlocks
  │  ├─ 4 个 latch 决策 (afk / fast / cache editing / thinking clear)
  │  ├─ resolveAppliedEffort
  │  ├─ recordPromptState (PROMPT_CACHE_BREAK_DETECTION gated)
  │  └─ startLLMRequestSpan (beta tracing)
  │
  ├─ 阶段 2: 初始化 RuntimeState + logAPIQuery (fire-and-forget)
  │
  ├─ 阶段 3: withRetry + 流式调用
  │  ├─ getAnthropicClient({maxRetries: 0})
  │  ├─ anthropic.beta.messages.create({...params, stream: true})
  │  ├─ .withResponse() → stream + response + requestId
  │  ├─ createStreamIdleWatchdog (子方法 3)
  │  └─ for await (part of stream) {
  │       watchdog.reset();
  │       yield* handleStreamEvent (子方法 4: message_start / content_block_* / message_delta)
  │     }
  │
  ├─ 阶段 4: 错误恢复
  │  ├─ handleStreamingError (子方法 6)
  │  │  ├─ 用户 abort → throw
  │  │  ├─ fallback 禁用 → 埋点 + throw
  │  │  └─ fallback 启用 → executeNonStreamingRequest
  │  │     ├─ withRetry + getAnthropicClient (maxRetries=0)
  │  │     ├─ adjustParamsForNonStreaming (max_tokens ≤ 64k, thinking budget 同步)
  │  │     ├─ anthropic.beta.messages.create + timeout
  │  │     └─ 失败 → 埋点 tengu_nonstreaming_fallback_error + throw (让 withRetry 重试)
  │  │
  │  └─ handleQueryModelError
  │     ├─ 二次 off-switch 检查
  │     ├─ adjustParams 减少 max_tokens 后重试
  │     └─ 错误归类 + getAssistantMessageFromError
  │
  └─ 阶段 5: logQueryModelSuccess
     ├─ markToolsSentToAPIState (cachedMC gated)
     ├─ setLastMainRequestId (主会话链)
     ├─ logAPISuccessAndDuration (fire-and-forget)
     └─ releaseStreamResources (双保险)
```

---

## 关联模块速查

| 模块 | 行数 | 职责 |
|------|-----:|------|
| `src/services/api/claude.ts` | 3811 | 本文件：API 调用核心 + 6 子方法编排 |
| `src/services/api/client.ts` | — | `getAnthropicClient`（SDK client 构造，maxRetries/source/fetchOverride） |
| `src/services/api/withRetry.ts` | — | `withRetry` / `CannotRetryError` / `is529Error` / `RetryContext` |
| `src/services/api/errors.ts` | — | `getAssistantMessageFromError` / `CUSTOM_OFF_SWITCH_MESSAGE` / `API_ERROR_MESSAGE_PREFIX` |
| `src/services/api/logging.ts` | — | `logAPIQuery` / `logAPISuccessAndDuration` / `logAPIError` / `EMPTY_USAGE` / `GlobalCacheStrategy` / `NonNullableUsage` |
| `src/services/api/promptCacheBreakDetection.ts` | — | `recordPromptState` / `checkResponseForCacheBreak` / `CACHE_TTL_1HOUR_MS` |
| `src/services/api/VCR.ts` | — | `withVCR` / `withStreamingVCR`（VCR 测试录制/回放） |
| `src/services/api/dumpPrompts.ts` | — | `getDumpPromptsPath` / `verifyApiKey`（ant-only API call dump） |
| `src/services/compact/microCompact.ts` | — | `consumePendingCacheEdits` / `getPinnedCacheEdits` / `markToolsSentToAPIState` / `pinCacheEdits` |
| `src/services/compact/apiMicrocompact.ts` | — | `getAPIContextManagement`（API context management 策略） |
| `src/services/compact/cachedMicrocompact.ts` | — | `isCachedMicrocompactEnabled` / `isModelSupportedForCacheEditing` / `getCachedMCConfig` |
| `src/services/analytics/index.ts` | — | `logEvent` + `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` |
| `src/services/analytics/growthbook.ts` | — | `getFeatureValue_CACHED_MAY_BE_STALE` / `getDynamicConfig_BLOCKS_ON_INIT` |
| `src/services/api/claudeAiLimits.ts` | — | `currentLimits` / `extractQuotaStatusFromError` / `extractQuotaStatusFromHeaders` |
| `src/telemetry/sessionTracing.ts` | — | `startLLMRequestSpan` / `endLLMRequestSpan` / `isBetaTracingEnabled` |
| `src/constants/betas.ts` | — | 全部 beta header 常量（AFK_MODE / CONTEXT_1M / EFFORT / FAST_MODE / PROMPT_CACHING_SCOPE / REDACT_THINKING / STRUCTURED_OUTPUTS / TASK_BUDGETS / ADVISOR / CACHE_EDITING 等） |
| `src/constants/system.ts` | — | `getAttributionHeader` / `getCLISyspromptPrefix` |
| `src/constants/apiLimits.ts` | — | `API_MAX_MEDIA_PER_REQUEST` |
| `src/constants/querySource.ts` | — | `QuerySource` 类型 |
| `src/context/notifications.ts` | — | `Notification` 类型 |
| `src/bootstrap/state.ts` | — | `getAfkModeHeaderLatched` / `setAfkModeHeaderLatched` / `getCacheEditingHeaderLatched` / `getFastModeHeaderLatched` / `getPromptCache1hAllowlist` / `getPromptCache1hEligible` / `getLastApiCompletionTimestamp` / `getSessionId` / `getThinkingClearLatched` / `setPromptCache1hEligible` / `setPromptCache1hAllowlist` 等 |
| `src/utils/model/model.ts` | — | `normalizeModelStringForAPI` / `parseUserSpecifiedModel` / `getDefaultOpusModel` / `getDefaultSonnetModel` / `getSmallFastModel` / `isNonCustomOpusModel` / `getRuntimeMainLoopModel` |
| `src/utils/model/bedrock.ts` | — | `getInferenceProfileBackingModel` |
| `src/utils/model/aliases.ts` | — | `MODEL_ALIASES` / `ModelAlias` |
| `src/utils/model/providers.ts` | — | `getAPIProvider` / `isFirstPartyAnthropicBaseUrl` |
| `src/utils/model/agent.ts` | — | `getAgentModel` |
| `src/utils/modelCost.ts` | — | `calculateUSDCost`（USD 成本计算） |
| `src/utils/context.ts` | — | `CAPPED_DEFAULT_MAX_TOKENS` / `getModelMaxOutputTokens` / `getSonnet1mExpTreatmentEnabled` / `getMaxThinkingTokensForModel` |
| `src/utils/effort.ts` | — | `resolveAppliedEffort` / `modelSupportsEffort` / `EffortValue` |
| `src/utils/thinking.ts` | — | `modelSupportsAdaptiveThinking` / `modelSupportsThinking` / `ThinkingConfig` |
| `src/utils/betas.ts` | — | `getBedrockExtraBodyParamsBetas` / `getMergedBetas` / `getModelBetas` / `getToolSearchBetaHeader` / `modelSupportsStructuredOutputs` / `shouldIncludeFirstPartyOnlyBetas` / `shouldUseGlobalCacheScope` |
| `src/utils/api.ts` | — | `CacheScope` / `logAPIPrefix` / `splitSysPromptPrefix` / `toolToAPISchema` |
| `src/utils/fastMode.ts` | — | `isFastModeAvailable` / `isFastModeCooldown` / `isFastModeEnabled` / `isFastModeSupportedByModel` |
| `src/utils/advisor.ts` | — | `ADVISOR_TOOL_INSTRUCTIONS` / `getExperimentAdvisorModels` / `isAdvisorEnabled` / `isValidAdvisorModel` / `modelSupportsAdvisor` |
| `src/utils/claudeInChrome/common.ts` | — | `CLAUDE_IN_CHROME_MCP_SERVER_NAME` |
| `src/utils/claudeInChrome/prompt.ts` | — | `CHROME_TOOL_SEARCH_INSTRUCTIONS` |
| `src/utils/toolSearch.ts` | — | `extractDiscoveredToolNames` / `isDeferredToolsDeltaEnabled` / `isToolSearchEnabled` |
| `src/utils/mcpInstructionsDelta.ts` | — | `isMcpInstructionsDeltaEnabled` |
| `src/utils/auth.ts` | — | `getOauthAccountInfo` / `isClaudeAISubscriber` |
| `src/utils/fingerprint.ts` | — | `computeFingerprintFromMessages`（attribution fingerprint） |
| `src/utils/systemPromptType.ts` | — | `SystemPrompt` / `asSystemPrompt` |
| `src/utils/messages.ts` | — | `createAssistantAPIErrorMessage` / `createUserMessage` / `ensureToolResultPairing` / `normalizeContentFromAPI` / `normalizeMessagesForAPI` / `stripAdvisorBlocks` / `stripCallerFieldFromAssistantMessage` / `stripToolReferenceBlocksFromUserMessage` |
| `src/utils/tokens.ts` | — | `tokenCountFromLastAPIResponse` |
| `src/utils/slowOperations.ts` | — | `jsonStringify` |
| `src/utils/json.ts` | — | `safeParseJSON` |
| `src/utils/array.ts` | — | `count` |
| `src/utils/contentArray.ts` | — | `insertBlockAfterToolResults` |
| `src/utils/envValidation.ts` | — | `validateBoundedIntEnvVar` |
| `src/utils/envUtils.ts` | — | `isEnvTruthy` |
| `src/utils/errors.ts` | — | `errorMessage` |
| `src/utils/debug.ts` | — | `logForDebugging` |
| `src/utils/diagLogs.ts` | — | `logForDiagnosticsNoPII` |
| `src/utils/generators.ts` | — | `returnValue` |
| `src/utils/log.ts` | — | `captureAPIRequest` / `logError` |
| `src/utils/agentContext.ts` | — | `getAgentContext`（用于嵌套 agent 上下文） |
| `src/utils/sessionActivity.ts` | — | `startSessionActivity` / `stopSessionActivity` |
| `src/utils/config.ts` | — | `getOrCreateUserID` |
| `src/utils/headlessProfiler.ts` | — | `headlessProfilerCheckpoint` |
| `src/utils/queryProfiler.ts` | — | `endQueryProfile` / `queryCheckpoint` |
| `src/Tool.ts` | — | `Tool` / `Tools` / `ToolUseContext` / `QueryChainTracking` / `getEmptyToolPermissionContext` / `toolMatchesName` |
| `src/tools/AgentTool/loadAgentsDir.ts` | — | `AgentDefinition` 类型 |
| `src/tools/ToolSearchTool/prompt.ts` | — | `TOOL_SEARCH_TOOL_NAME` / `isDeferredTool` / `formatDeferredToolLine` |
| `src/state/AppState.ts` | — | `AppState` 类型 |
| `src/services/mcp/utils.ts` | — | `isToolFromMcpServer` |
| `src/services/lsp/manager.ts` | — | `getInitializationStatus`（LSP 初始化状态，决定是否 defer LSP tool） |
| `src/services/PromptSuggestion/speculation.ts` | — | `startSpeculation`（调用 queryHaiku 跑小型抽样） |
| `src/utils/permissions/autoModeState.ts` | — | `isAutoModeActive`（lazy require，`feature('TRANSCRIPT_CLASSIFIER')` 守护） |
| `src/types/connectorText.ts` | — | `ConnectorTextBlock` / `ConnectorTextDelta` / `isConnectorTextBlock` |
| `src/types/message.ts` | — | `Message` / `AssistantMessage` / `UserMessage` / `StreamEvent` / `SystemAPIErrorMessage` |
| `src/types/ids.ts` | — | `AgentId` 类型 |
| `src/entrypoints/agentSdkTypes.ts` | — | `BetaMessage` / `BetaMessageStreamParams` / `BetaRawMessageStreamEvent` / `BetaContentBlock` / `BetaContentBlockParam` / `BetaImageBlockParam` / `BetaJSONOutputFormat` / `BetaOutputConfig` / `BetaRequestDocumentBlock` / `BetaStopReason` / `BetaToolChoiceAuto` / `BetaToolChoiceTool` / `BetaToolResultBlockParam` / `BetaToolUnion` / `BetaUsage` / `BetaMessageDeltaUsage` / `MessageParam` 等 |
| `src/entrypoints/agentSdkTypes.ts` | — | `ClientOptions`（SDK client 配置） |
| `@anthropic-ai/sdk/error` | — | `APIConnectionTimeoutError` / `APIError` / `APIUserAbortError` |
| `@anthropic-ai/sdk/streaming.mjs` | — | `Stream`（流类型，含 controller.signal 用于 abort） |
| `@anthropic-ai/sdk/resources/index.mjs` | — | `TextBlockParam` |
| `src/cost-tracker.ts` | — | `addToTotalSessionCost` |

---

## 重要不变量

- **6 个子方法 + ctx/state 双对象**：ctx 不可变，state 可变；避免散落几十个 let 变量。`buildQueryModelRequestParams` 是唯一修改 `state.lastRequestBetas` 的子方法，保持纯函数性质。
- **每请求 1 个 message-level cache marker**：Mycro page_manager 立即释放非边界 locals；fire-and-forget forks 把 marker 移到倒数第二条（共享前缀 no-op merge，不留 KVCC tail）。
- **input_tokens 防御 0 覆盖**：`updateUsage` 用 `> 0` 守卫，保留 `message_start` 设的真实值不被 `message_delta` 的 0 覆盖。
- **Sticky-on latch 机制**：AFK / fast / cache editing / thinking clear 四个 beta header 一旦首次发送就保持；`/clear` / `/compact` 时清空。Per-call gate（isAgenticQuery, querySource）保持 per-call。
- **Adaptive thinking 优先**：模型支持时 always `type: 'adaptive'`（无 budget）；不支持时回退 budget_tokens（min(max_tokens-1, 默认/用户配置)）。**重要**：改这个选择前需通知 model launch DRI 和 research。
- **Capped default max tokens**：slot-reservation cap 默认 8k；命中 cap 时 query.ts 一次 64k retry（`max_output_tokens_escalate`）。`Math.min` 保留模型原生低默认值。
- **Off-switch 双层检查**：prepareQueryModelContext 检查一次 + handleQueryModelError 检查一次 —— GrowthBook mid-request 更新也能响应。
- **Client request ID 仅 1P**：3P providers 不记录 client request id（inc-4029 class），流式 timeout 拿不到 server request id 时仍能关联服务端日志。
- **Tool result content cloning**：`userMessageToMessageParam` 克隆数组内容防止 `insertCacheEditsBlock` splice 污染原始 message。
- **Single SDK client maxRetries=0**：禁用 SDK 自动重试，统一走 `withRetry` 手动实现（让 529 预算、adjustParams、off-switch 等集中处理）。
- **`MAX_NON_STREAMING_TOKENS = 64_000`**：非流式 fallback 时 max_tokens 上限（API 10min × 128k tokens/hour 派生）；thinking budget 同步截断（保持 `max_tokens > budget_tokens` 约束）。
- **API timeout**：默认 300s；`API_TIMEOUT_MS` env var 覆盖；`CLAUDE_CODE_REMOTE` 时 120s（CCR container idle-kill 5min 内清理）。
- **Fallback 触发条件**：流式失败 + 未禁用 fallback + `onStreamingFallback` 让 query.ts 把已 yield 的 assistant 打 tombstone（防双倍工具调用 inc-4258）。
- **`isMaxTokensCapEnabled` 默认 3P false**：`tengu_otk_slot_v1` GrowthBook 控制；3P providers 不验证 cap（Bedrock/Vertex native 行为）。
- **`createStreamIdleWatchdog` 默认关闭**：仅 `CLAUDE_ENABLE_STREAM_WATCHDOG=true` 启用；不破坏现有 happy path；触发后立即 abort → catch 走非流式 fallback。
- **fire-and-forget `logAPIQuery` / `logAPISuccessAndDuration`**：预先计算标量避免 `.then()` 闭包 pin 住整个 `messagesForAPI` 数组（保留 GC 能力）。
- **`releaseStreamResources` 双保险**：try/catch finally 已释放一次，`logQueryModelSuccess` 再释放一次 —— 防止 finally 之前抛错导致资源泄漏。
- **`CLIENT_REQUEST_ID_HEADER` 关联**：1P 时生成 randomUUID 作为 clientRequestId，让 timeout（拿不到 server request id）能关联服务端日志。
