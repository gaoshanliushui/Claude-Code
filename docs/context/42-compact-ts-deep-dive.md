# `compact.ts` 深度解析：Claude Code 上下文压缩核心实现

> 本文深入分析 `src/services/compact/compact.ts` 的设计、流程和核心功能。
> 文件大小：约 1700 行代码，是 Claude Code 上下文压缩系统的调度中心。

---

## 一、文件整体定位

### 1.1 核心职责

`compact.ts` 是 Claude Code 上下文压缩系统的**调度中心**，负责：

```
┌────────────────────────────────────────────────────┐
│            compact.ts 核心职责                        │
├────────────────────────────────────────────────────┤
│  1. 调用 LLM 生成对话摘要                            │
│  2. 处理 Prompt-Too-Long (PTL) 重试                  │
│  3. 集成 Prompt Cache（Fork 模式）                   │
│  4. 重新注入关键附件（文件/Plan/Skill 等）            │
│  5. 执行 PreCompact / PostCompact Hooks              │
│  6. 处理 Partial Compact（部分压缩）                 │
│  7. 创建 Compact Boundary Marker                    │
│  8. 记录压缩指标（Analytics）                        │
│  9. 处理压缩错误和异常                              │
└────────────────────────────────────────────────────┘
```

### 1.2 在压缩系统中的位置

```
src/services/compact/
├── compact.ts          ← 调度中心（本文件）
├── microCompact.ts     # Microcompact（工具结果清理）
├── autoCompact.ts      # 自动压缩决策
├── reactiveCompact.ts  # 响应式压缩（CC-1180 PTL）
├── snipCompact.ts      # Snip 压缩
├── sessionMemoryCompact.ts # 会话记忆
├── grouping.ts         # 消息分组
├── prompt.ts           # 压缩 prompt
├── postCompactCleanup.ts # 压缩后清理
├── compactWarningHook.ts   # 警告钩子
├── compactWarningState.ts  # 警告状态
├── timeBasedMCConfig.ts   # 时间型配置
└── cachedMCConfig.ts      # 缓存型配置
```

---

## 二、核心常量与配置

### 2.1 压缩相关常量

```ts
// src/services/compact/compact.ts
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5
export const POST_COMPACT_TOKEN_BUDGET = 50_000
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
const MAX_COMPACT_STREAMING_RETRIES = 2
```

### 2.2 错误消息

```ts
export const ERROR_MESSAGE_NOT_ENOUGH_MESSAGES =
    'Not enough messages to compact.'
export const ERROR_MESSAGE_PROMPT_TOO_LONG =
    'Conversation too long. Press esc twice to go up a few messages and try again.'
export const ERROR_MESSAGE_USER_ABORT = 'API Error: Request was aborted.'
export const ERROR_MESSAGE_INCOMPLETE_RESPONSE =
    'Compaction interrupted · This may be due to network issues — please try again.'
const MAX_PTL_RETRIES = 3
const PTL_RETRY_MARKER = '[earlier conversation truncated for compaction retry]'
```

**常量说明**：
- `MAX_COMPACT_STREAMING_RETRIES = 2`：流式摘要重试次数
- `MAX_PTL_RETRIES = 3`：Prompt-Too-Long 重试次数
- `PTL_RETRY_MARKER`：标记被截断的消息

### 2.3 注释解释 Skills Truncation

```ts
// Skills can be large (verify=18.7KB, claude-api=20.1KB). Previously re-injected
// unbounded on every compact → 5-10K tok/compact. Per-skill truncation beats
// dropping — instructions at the top of a skill file are usually the critical
// part. Budget sized to hold ~5 skills at the per-skill cap.
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

**关键洞察**：保留每个 skill 的头部 5000 tokens，5 个 skill 总共 25000 tokens。

---

## 三、数据结构定义

### 3.1 CompactionResult 接口

```ts
export interface CompactionResult {
    boundaryMarker: SystemMessage              // 压缩边界标记
    summaryMessages: UserMessage[]             // 摘要消息
    attachments: AttachmentMessage[]           // 重新注入的附件
    hookResults: HookResultMessage[]           // SessionStart hooks 结果
    messagesToKeep?: Message[]                 // 部分压缩保留的消息
    userDisplayMessage?: string                // 用户可见消息
    preCompactTokenCount?: number              // 压缩前 token 数
    postCompactTokenCount?: number             // 压缩 API 调用总 token
    truePostCompactTokenCount?: number         // 真实压缩后 token
    compactionUsage?: ReturnType<typeof getTokenUsage>  // API 用量
}
```

### 3.2 RecompactionInfo

```ts
/**
 * Diagnosis context passed from autoCompactIfNeeded into compactConversation.
 * Lets the tengu_compact event disambiguate same-chain loops (H2) from
 * cross-agent (H1/H5) and manual-vs-auto (H3) compactions without joins.
 */
export type RecompactionInfo = {
    isRecompactionInChain: boolean   // 是否是链中重复压缩
    turnsSincePreviousCompact: number
    previousCompactTurnId?: string
    autoCompactThreshold: number
    querySource?: QuerySource
}
```

**用途**：传递上下文信息用于分析，无需额外 join。

---

## 四、辅助函数

### 4.1 stripImagesFromMessages

```ts
export function stripImagesFromMessages(messages: Message[]): Message[]
```

**作用**：从用户消息中移除图片和文档块，替换为文本标记。

```ts
// 替换示例
{type: 'image', ...} → {type: 'text', text: '[image]'}
{type: 'document', ...} → {type: 'text', text: '[document]'}
```

**注释解释**：
> Strip image blocks from user messages before sending for compaction. Images are not needed for generating a conversation summary and can cause the compaction API call itself to hit the prompt-too-long limit, especially in CCD sessions where users frequently attach images.

### 4.2 stripReinjectedAttachments

```ts
export function stripReinjectedAttachments(messages: Message[]): Message[]
```

**作用**：移除会被重新注入的附件，避免重复。

```ts
// 被 strip 的附件类型
{type: 'attachment', attachment: {type: 'skill_discovery' | 'skill_listing'}}
```

**注释解释**：
> skill_discovery/skill_listing are re-surfaced by resetSentSkillNames() + the next turn's discovery signal, so feeding them to the summarizer wastes tokens and pollutes the summary with stale skill suggestions.

### 4.3 truncateHeadForPTLRetry（CC-1180）

```ts
export function truncateHeadForPTLRetry(
    messages: Message[],
    ptlResponse: AssistantMessage,
): Message[] | null
```

**作用**：当压缩请求本身超过 prompt 限制时，**丢弃最老的 API-round groups** 并重试。

```ts
// 算法流程
1. 分组消息（按 API round）
2. 计算 token 缺口
3. 累积丢弃直到 token 缺口被覆盖
4. 保留至少一组
5. 如果切掉后开头不是 user，补一个 marker
```

**关键设计**：
- 按 API-round 分组（避免破坏 tool_use/tool_result 配对）
- 累积丢弃（精确控制丢弃量）
- 保留至少一组（必须有内容可总结）

**注释解释**：
> This is the last-resort escape hatch for CC-1180 — when the compact request itself hits prompt-too-long, the user is otherwise stuck. Dropping the oldest context is lossy but unblocks them.

### 4.4 buildPostCompactMessages

```ts
export function buildPostCompactMessages(result: CompactionResult): Message[]
```

**作用**：构造压缩后的消息数组顺序。

```ts
return [
    result.boundaryMarker,           // 1. 边界标记
    ...result.summaryMessages,       // 2. 摘要
    ...(result.messagesToKeep ?? []), // 3. 保留消息
    ...result.attachments,           // 4. 附件
    ...result.hookResults,           // 5. Hook 结果
]
```

**关键设计**：统一所有压缩路径的消息顺序。

### 4.5 annotateBoundaryWithPreservedSegment

```ts
export function annotateBoundaryWithPreservedSegment(
    boundary: SystemCompactBoundaryMessage,
    anchorUuid: UUID,
    messagesToKeep: readonly Message[] | undefined,
): SystemCompactBoundaryMessage
```

**作用**：为压缩边界添加保留段的 relink 元数据。

```ts
preservedSegment: {
    headUuid: keep[0]!.uuid,
    anchorUuid,
    tailUuid: keep.at(-1)!.uuid,
}
```

**用途**：保留的消息保持原始 parentUuids，加载器用此 patch head→anchor 关系。

### 4.6 mergeHookInstructions

```ts
export function mergeHookInstructions(
    userInstructions: string | undefined,
    hookInstructions: string | undefined,
): string | undefined
```

**作用**：合并用户指令和 hook 提供的指令。

```ts
// 合并规则
user + hook → user + "\n\n" + hook
empty strings → undefined
```

### 4.7 createCompactCanUseTool

```ts
export function createCompactCanUseTool(): CanUseToolFn
```

**作用**：创建压缩期间的 canUseTool，**禁止工具调用**。

```ts
return async () => ({
    behavior: 'deny',
    message: 'Tool use is not allowed during compaction',
    decisionReason: {
        type: 'other',
        reason: 'compaction agent should only produce text summary',
    },
})
```

**设计意图**：压缩只能产生文本摘要，不能调用工具。

---

## 五、`compactConversation` 主函数详解

### 5.1 函数签名

```ts
export async function compactConversation(
    messages: Message[],
    context: ToolUseContext,
    cacheSafeParams: CacheSafeParams,
    suppressFollowUpQuestions: boolean,
    customInstructions?: string,
    isAutoCompact: boolean = false,
    recompactionInfo?: RecompactionInfo,
): Promise<CompactionResult>
```

### 5.2 完整流程图

```
开始
  ↓
1. 验证消息非空
  ↓
2. 计算 preCompactTokenCount
  ↓
3. 执行 PreCompact Hooks
  ↓
4. 合并 hook 指令
  ↓
5. 设置状态（compacting、requesting）
  ↓
6. 检查 promptCacheSharingEnabled
  ↓
7. 构造压缩 prompt
  ↓
8. PTL 重试循环（最多 3 次）
  ├─ 调用 streamCompactSummary
  ├─ 检测 prompt too long
  └─ truncateHeadForPTLRetry 并重试
  ↓
9. 验证摘要结果
  ↓
10. 清理文件状态缓存
  ↓
11. 创建附件（文件/Plan/Skill/Agent）
  ↓
12. 重新执行 SessionStart Hooks
  ↓
13. 创建 boundary marker + summary
  ↓
14. 标记 post-compaction
  ↓
15. 执行 PostCompact Hooks
  ↓
16. 记录 Analytics 指标
  ↓
17. 返回 CompactionResult
```

### 5.3 核心代码详解

#### 5.3.1 初始化阶段

```ts
async function compactConversation(...) {
    try {
        // 验证消息
        if (messages.length === 0) {
            throw new Error(ERROR_MESSAGE_NOT_ENOUGH_MESSAGES)
        }
        
        // 计算压缩前 token
        const preCompactTokenCount = tokenCountWithEstimation(messages)
        
        // 获取 appState
        const appState = context.getAppState()
        void logPermissionContextForAnts(appState.toolPermissionContext, 'summary')
        
        // 通知 UI：开始 hooks
        context.onCompactProgress?.({
            type: 'hooks_start',
            hookType: 'pre_compact',
        })
    }
}
```

#### 5.3.2 PreCompact Hooks

```ts
// 设置 SDK 状态
context.setSDKStatus?.('compacting')

// 执行 PreCompact Hooks
const hookResult = await executePreCompactHooks({
    trigger: isAutoCompact ? 'auto' : 'manual',
    customInstructions: customInstructions ?? null,
}, context.abortController.signal)

// 合并 hook 指令
customInstructions = mergeHookInstructions(
    customInstructions,
    hookResult.newCustomInstructions,
)
const userDisplayMessage = hookResult.userDisplayMessage
```

#### 5.3.3 UI 状态更新

```ts
// 显示请求模式
context.setStreamMode?.('requesting')
context.setResponseLength?.(() => 0)
context.onCompactProgress?.({type: 'compact_start'})
```

#### 5.3.4 Prompt Cache 配置

```ts
// 3P default: true — forked-agent path reuses main conversation's prompt cache.
// Experiment (Jan 2026) confirmed: false path is 98% cache miss, costs ~0.76% of
// fleet cache_creation (~38B tok/day), concentrated in ephemeral envs (CCR/GHA/SDK)
// with cold GB cache and 3P providers where GB is disabled. GB gate kept as kill-switch.
const promptCacheSharingEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_compact_cache_prefix',
    true,
)
```

#### 5.3.5 构造压缩 Prompt

```ts
const compactPrompt = getCompactPrompt(customInstructions)
const summaryRequest = createUserMessage({
    content: compactPrompt,
})
```

#### 5.3.6 PTL 重试循环

```ts
let messagesToSummarize = messages
let retryCacheSafeParams = cacheSafeParams
let ptlAttempts = 0

for (; ;) {
    summaryResponse = await streamCompactSummary({
        messages: messagesToSummarize,
        summaryRequest,
        appState,
        context,
        preCompactTokenCount,
        cacheSafeParams: retryCacheSafeParams,
    })
    
    summary = getAssistantMessageText(summaryResponse)
    if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break
    
    // CC-1180: compact request itself hit prompt-too-long
    ptlAttempts++
    const truncated = ptlAttempts <= MAX_PTL_RETRIES
        ? truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
        : null
    
    if (!truncated) {
        logEvent('tengu_compact_failed', {
            reason: 'prompt_too_long',
            preCompactTokenCount,
            promptCacheSharingEnabled,
            ptlAttempts,
        })
        throw new Error(ERROR_MESSAGE_PROMPT_TOO_LONG)
    }
    
    logEvent('tengu_compact_ptl_retry', {
        attempt: ptlAttempts,
        droppedMessages: messagesToSummarize.length - truncated.length,
        remainingMessages: truncated.length,
    })
    
    messagesToSummarize = truncated
    retryCacheSafeParams = {
        ...retryCacheSafeParams,
        forkContextMessages: truncated,
    }
}
```

#### 5.3.7 摘要结果验证

```ts
if (!summary) {
    logForDebugging(
        `Compact failed: no summary text in response. Response: ${jsonStringify(summaryResponse)}`,
        {level: 'error'},
    )
    logEvent('tengu_compact_failed', {
        reason: 'no_summary',
        preCompactTokenCount,
        promptCacheSharingEnabled,
    })
    throw new Error('Failed to generate conversation summary - response did not contain valid text content')
} else if (startsWithApiErrorPrefix(summary)) {
    logEvent('tengu_compact_failed', {
        reason: 'api_error',
        preCompactTokenCount,
        promptCacheSharingEnabled,
    })
    throw new Error(summary)
}
```

#### 5.3.8 文件状态清理

```ts
// Store the current file state before clearing
const preCompactReadFileState = cacheToObject(context.readFileState)

// Clear the cache
context.readFileState.clear()
context.loadedNestedMemoryPaths?.clear()

// Intentionally NOT resetting sentSkillNames: re-injecting the full
// skill_listing (~4K tokens) post-compact is pure cache_creation with
// marginal benefit.
```

#### 5.3.9 并行生成附件

```ts
// Run async attachment generation in parallel
const [fileAttachments, asyncAgentAttachments] = await Promise.all([
    createPostCompactFileAttachments(
        preCompactReadFileState,
        context,
        POST_COMPACT_MAX_FILES_TO_RESTORE,
    ),
    createAsyncAgentAttachmentsIfNeeded(context),
])

const postCompactFileAttachments: AttachmentMessage[] = [
    ...fileAttachments,
    ...asyncAgentAttachments,
]
```

#### 5.3.10 添加特殊附件

```ts
// 添加 Plan 文件附件
const planAttachment = createPlanAttachmentIfNeeded(context.agentId)
if (planAttachment) {
    postCompactFileAttachments.push(planAttachment)
}

// 添加 Plan Mode 附件
const planModeAttachment = await createPlanModeAttachmentIfNeeded(context)
if (planModeAttachment) {
    postCompactFileAttachments.push(planModeAttachment)
}

// 添加 Skill 附件
const skillAttachment = createSkillAttachmentIfNeeded(context.agentId)
if (skillAttachment) {
    postCompactFileAttachments.push(skillAttachment)
}
```

#### 5.3.11 重新声明 Delta 附件

```ts
// Compaction ate prior delta attachments. Re-announce from the current
// state so the model has tool/instruction context on the first
// post-compact turn. Empty message history → diff against nothing →
// announces the full set.
for (const att of getDeferredToolsDeltaAttachment(
    context.options.tools,
    context.options.mainLoopModel,
    [],
    {callSite: 'compact_full'},
)) {
    postCompactFileAttachments.push(createAttachmentMessage(att))
}

for (const att of getAgentListingDeltaAttachment(context, [])) {
    postCompactFileAttachments.push(createAttachmentMessage(att))
}

for (const att of getMcpInstructionsDeltaAttachment(
    context.options.mcpClients,
    context.options.tools,
    context.options.mainLoopModel,
    [],
)) {
    postCompactFileAttachments.push(createAttachmentMessage(att))
}
```

#### 5.3.12 SessionStart Hooks

```ts
context.onCompactProgress?.({
    type: 'hooks_start',
    hookType: 'session_start',
})

// Execute SessionStart hooks after successful compaction
const hookMessages = await processSessionStartHooks('compact', {
    model: context.options.mainLoopModel,
})
```

#### 5.3.13 创建 Boundary Marker

```ts
const boundaryMarker = createCompactBoundaryMessage(
    isAutoCompact ? 'auto' : 'manual',
    preCompactTokenCount ?? 0,
    messages.at(-1)?.uuid,
)

// Carry loaded-tool state — the summary doesn't preserve tool_reference
// blocks, so the post-compact schema filter needs this to keep sending
// already-loaded deferred tool schemas to the API.
const preCompactDiscovered = extractDiscoveredToolNames(messages)
if (preCompactDiscovered.size > 0) {
    boundaryMarker.compactMetadata.preCompactDiscoveredTools = [
        ...preCompactDiscovered,
    ].sort()
}
```

#### 5.3.14 创建 Summary Message

```ts
const transcriptPath = getTranscriptPath()
const summaryMessages: UserMessage[] = [
    createUserMessage({
        content: getCompactUserSummaryMessage(
            summary,
            suppressFollowUpQuestions,
            transcriptPath,
        ),
        isCompactSummary: true,
        isVisibleInTranscriptOnly: true,
    }),
]
```

#### 5.3.15 计算真实 Token

```ts
// Previously "postCompactTokenCount" — renamed because this is the
// compact API call's total usage (input_tokens ≈ preCompactTokenCount),
// NOT the size of the resulting context.
const compactionCallTotalTokens = tokenCountFromLastAPIResponse([
    summaryResponse,
])

// Message-payload estimate of the resulting context. The next iteration's
// shouldAutoCompact will see this PLUS ~20-40K for system prompt + tools +
// userContext (via API usage.input_tokens).
const truePostCompactTokenCount = roughTokenCountEstimationForMessages([
    boundaryMarker,
    ...summaryMessages,
    ...postCompactFileAttachments,
    ...hookMessages,
])
```

#### 5.3.16 Analytics 记录

```ts
const compactionUsage = getTokenUsage(summaryResponse)

logEvent('tengu_compact', {
    preCompactTokenCount,
    postCompactTokenCount: compactionCallTotalTokens,
    truePostCompactTokenCount,
    autoCompactThreshold: recompactionInfo?.autoCompactThreshold ?? -1,
    willRetriggerNextTurn:
        recompactionInfo !== undefined &&
        truePostCompactTokenCount >= recompactionInfo.autoCompactThreshold,
    isAutoCompact,
    querySource: querySourceForEvent,
    queryChainId: (context.queryTracking?.chainId ?? ''),
    queryDepth: context.queryTracking?.depth ?? -1,
    isRecompactionInChain: recompactionInfo?.isRecompactionInChain ?? false,
    turnsSincePreviousCompact: recompactionInfo?.turnsSincePreviousCompact ?? -1,
    previousCompactTurnId: (recompactionInfo?.previousCompactTurnId ?? ''),
    compactionInputTokens: compactionUsage?.input_tokens,
    compactionOutputTokens: compactionUsage?.output_tokens,
    compactionCacheReadTokens: compactionUsage?.cache_read_input_tokens ?? 0,
    compactionCacheCreationTokens: compactionUsage?.cache_creation_input_tokens ?? 0,
    compactionTotalTokens: compactionUsage
        ? compactionUsage.input_tokens +
          (compactionUsage.cache_creation_input_tokens ?? 0) +
          (compactionUsage.cache_read_input_tokens ?? 0) +
          compactionUsage.output_tokens
        : 0,
    promptCacheSharingEnabled,
    // analyzeContext walks every content block (~11ms on a 4.5K-message session)
    // purely for this telemetry breakdown.
    ...(() => {
        try {
            return tokenStatsToStatsigMetrics(analyzeContext(messages))
        } catch (error) {
            logError(error as Error)
            return {}
        }
    })(),
})
```

#### 5.3.17 Cache Reset

```ts
// Reset cache read baseline so the post-compact drop isn't flagged as a break
if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
    notifyCompaction(
        context.options.querySource ?? 'compact',
        context.agentId,
    )
}
markPostCompaction()
```

#### 5.3.18 Session Metadata

```ts
// Re-append session metadata (custom title, tag) so it stays within
// the 16KB tail window that readLiteMetadata reads for --resume display.
reAppendSessionMetadata()
```

#### 5.3.19 Session Transcript（KAIROS only）

```ts
// Write a reduced transcript segment for the pre-compaction messages
// (assistant mode only). Fire-and-forget — errors are logged internally.
if (feature('KAIROS')) {
    void sessionTranscriptModule?.writeSessionTranscriptSegment(messages)
}
```

#### 5.3.20 PostCompact Hooks

```ts
context.onCompactProgress?.({
    type: 'hooks_start',
    hookType: 'post_compact',
})

const postCompactHookResult = await executePostCompactHooks({
    trigger: isAutoCompact ? 'auto' : 'manual',
    compactSummary: summary,
}, context.abortController.signal)

const combinedUserDisplayMessage = [
    userDisplayMessage,
    postCompactHookResult.userDisplayMessage,
].filter(Boolean).join('\n')
```

#### 5.3.21 返回结果

```ts
return {
    boundaryMarker,
    summaryMessages,
    attachments: postCompactFileAttachments,
    hookResults: hookMessages,
    userDisplayMessage: combinedUserDisplayMessage || undefined,
    preCompactTokenCount,
    postCompactTokenCount: compactionCallTotalTokens,
    truePostCompactTokenCount,
    compactionUsage,
}
```

### 5.4 错误处理

```ts
} catch (error) {
    // Only show the error notification for manual /compact.
    // Auto-compact failures are retried on the next turn and the
    // notification is confusing when compaction eventually succeeds.
    if (!isAutoCompact) {
        addErrorNotificationIfNeeded(error, context)
    }
    throw error
} finally {
    context.setStreamMode?.('requesting')
    context.setResponseLength?.(() => 0)
    context.onCompactProgress?.({type: 'compact_end'})
    context.setSDKStatus?.(null)
}
```

---

## 六、`streamCompactSummary` 核心函数

### 6.1 函数签名

```ts
async function streamCompactSummary({
    messages,
    summaryRequest,
    appState,
    context,
    preCompactTokenCount,
    cacheSafeParams,
}): Promise<AssistantMessage>
```

### 6.2 核心设计：Fork 优先 + Streaming 兜底

```
streamCompactSummary
    ↓
1. 检查 promptCacheSharingEnabled
    ↓
2. Fork 模式（runForkedAgent）
    ├─ 共享主线程 cache
    ├─ maxTurns: 1
    ├─ skipCacheWrite: true
    └─ 失败 → 回退
    ↓
3. Streaming 模式（queryModelWithStreaming）
    ├─ retry: MAX_COMPACT_STREAMING_RETRIES 次
    └─ thinking disabled
```

### 6.3 Fork 模式（核心优化）

```ts
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
const result = await runForkedAgent({
    promptMessages: [summaryRequest],
    cacheSafeParams,
    canUseTool: createCompactCanUseTool(),
    querySource: 'compact',
    forkLabel: 'compact',
    maxTurns: 1,
    skipCacheWrite: true,
    overrides: {abortController: context.abortController},
})

const assistantMsg = getLastAssistantMessage(result.messages)
const assistantText = assistantMsg ? getAssistantMessageText(assistantMsg) : null

// Guard isApiErrorMessage: query() catches API errors (including
// APIUserAbortError on ESC) and yields them as synthetic assistant
// messages. Without this check, an aborted compact "succeeds" with
// "Request was aborted." as the summary — the text doesn't start with
// "API Error" so the caller's startsWithApiErrorPrefix guard misses it.
if (assistantMsg && assistantText && !assistantMsg.isApiErrorMessage) {
    if (!assistantText.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) {
        logEvent('tengu_compact_cache_sharing_success', {
            preCompactTokenCount,
            outputTokens: result.totalUsage.output_tokens,
            cacheReadInputTokens: result.totalUsage.cache_read_input_tokens,
            cacheCreationInputTokens: result.totalUsage.cache_creation_input_tokens,
            cacheHitRate: ...,
        })
    }
    return assistantMsg
}
```

**关键设计**：
- 不设置 `maxOutputTokens`（避免 cache key 不匹配）
- 使用 `skipCacheWrite: true`（不污染主 cache）
- 检查 `isApiErrorMessage`（避免 API 错误被误认为成功）

### 6.4 keep-alive 信号

```ts
// Send keep-alive signals during compaction to prevent remote session
// WebSocket idle timeouts from dropping bridge connections. Compaction
// API calls can take 5-10+ seconds, during which no other messages
// flow through the transport — without keep-alives, the server may
// close the WebSocket for inactivity.
const activityInterval = isSessionActivityTrackingActive()
    ? setInterval(() => {
        sendSessionActivitySignal()
        context.setSDKStatus?.('compacting')
    }, 30_000, context.setSDKStatus)
    : undefined
```

**注释解释**：
> Send keep-alive signals during compaction to prevent remote session WebSocket idle timeouts from dropping bridge connections.

### 6.5 Streaming 模式（回退）

```ts
// Regular streaming path (fallback when cache sharing fails or is disabled)
const retryEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_compact_streaming_retry',
    false,
)
const maxAttempts = retryEnabled ? MAX_COMPACT_STREAMING_RETRIES : 1

for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    // Reset state for retry
    let hasStartedStreaming = false
    let response: AssistantMessage | undefined
    context.setResponseLength?.(() => 0)
    
    // Check if tool search is enabled
    const useToolSearch = await isToolSearchEnabled(...)
    
    const tools: Tool[] = useToolSearch
        ? uniqBy([FileReadTool, ToolSearchTool, ...context.options.tools.filter(t => t.isMcp)], 'name')
        : [FileReadTool]
    
    const streamingGen = queryModelWithStreaming({
        messages: normalizeMessagesForAPI(
            stripImagesFromMessages(
                stripReinjectedAttachments([
                    ...getMessagesAfterCompactBoundary(messages),
                    summaryRequest,
                ]),
            ),
            context.options.tools,
        ),
        systemPrompt: asSystemPrompt([
            'You are a helpful AI assistant tasked with summarizing conversations.',
        ]),
        thinkingConfig: {type: 'disabled' as const},  // 禁用 thinking
        tools,
        signal: context.abortController.signal,
        options: {
            async getToolPermissionContext() {
                const appState = context.getAppState()
                return appState.toolPermissionContext
            },
            model: context.options.mainLoopModel,
            toolChoice: undefined,
            isNonInteractiveSession: context.options.isNonInteractiveSession,
            hasAppendSystemPrompt: !!context.options.appendSystemPrompt,
            maxOutputTokensOverride: Math.min(
                COMPACT_MAX_OUTPUT_TOKENS,
                getMaxOutputTokensForModel(context.options.mainLoopModel),
            ),
            querySource: 'compact',
            agents: context.options.agentDefinitions.activeAgents,
            mcpTools: [],
            effortValue: appState.effortValue,
        },
    })
}
```

### 6.6 流式进度追踪

```ts
const streamIter = streamingGen[Symbol.asyncIterator]()
let next = await streamIter.next()

while (!next.done) {
    const event = next.value
    
    // 第一次出现 text 时切换到 responding 模式
    if (
        !hasStartedStreaming &&
        event.type === 'stream_event' &&
        event.event.type === 'content_block_start' &&
        event.event.content_block.type === 'text'
    ) {
        hasStartedStreaming = true
        context.setStreamMode?.('responding')
    }
    
    // 累积 token 数
    if (
        event.type === 'stream_event' &&
        event.event.type === 'content_block_delta' &&
        event.event.delta.type === 'text_delta'
    ) {
        const charactersStreamed = event.event.delta.text.length
        context.setResponseLength?.(length => length + charactersStreamed)
    }
    
    if (event.type === 'assistant') {
        response = event
    }
    
    next = await streamIter.next()
}
```

### 6.7 错误清理

```ts
} finally {
    clearInterval(activityInterval)
}
```

---

## 七、附件生成函数

### 7.1 createPostCompactFileAttachments

```ts
export async function createPostCompactFileAttachments(
    readFileState: Record<string, { content: string; timestamp: number }>,
    toolUseContext: ToolUseContext,
    maxFiles: number,
    preservedMessages: Message[] = [],
): Promise<AttachmentMessage[]>
```

**算法流程**：
1. 收集保留消息中已读过的文件路径
2. 过滤掉 plan 文件和 CLAUDE.md
3. 按时间排序，取最近 `maxFiles` 个
4. 并发读取文件内容
5. 控制总 token 预算（50000）

```ts
// 过滤排除文件
const recentFiles = Object.entries(readFileState)
    .map(([filename, state]) => ({filename, ...state}))
    .filter(file =>
        !shouldExcludeFromPostCompactRestore(file.filename, toolUseContext.agentId) &&
        !preservedReadPaths.has(expandPath(file.filename))
    )
    .sort((a, b) => b.timestamp - a.timestamp)
    .slice(0, maxFiles)
```

### 7.2 createPlanAttachmentIfNeeded

```ts
export function createPlanAttachmentIfNeeded(
    agentId?: AgentId,
): AttachmentMessage | null
```

**作用**：如果存在 plan 文件，创建 plan_file_reference 附件。

### 7.3 createSkillAttachmentIfNeeded

```ts
export function createSkillAttachmentIfNeeded(
    agentId?: string,
): AttachmentMessage | null
```

**算法**：
1. 获取 agent 调用的 skills
2. 按最近调用排序
3. 截断每个 skill 到 5000 tokens
4. 控制总预算 25000 tokens

```ts
let usedTokens = 0
const skills = Array.from(invokedSkills.values())
    .sort((a, b) => b.invokedAt - a.invokedAt)
    .map(skill => ({
        name: skill.skillName,
        path: skill.skillPath,
        content: truncateToTokens(skill.content, POST_COMPACT_MAX_TOKENS_PER_SKILL),
    }))
    .filter(skill => {
        const tokens = roughTokenCountEstimation(skill.content)
        if (usedTokens + tokens > POST_COMPACT_SKILLS_TOKEN_BUDGET) return false
        usedTokens += tokens
        return true
    })
```

### 7.4 createPlanModeAttachmentIfNeeded

```ts
export async function createPlanModeAttachmentIfNeeded(
    context: ToolUseContext,
): Promise<AttachmentMessage | null>
```

**作用**：如果当前在 plan mode，注入 plan mode 提醒附件。

### 7.5 createAsyncAgentAttachmentsIfNeeded

```ts
export async function createAsyncAgentAttachmentsIfNeeded(
    context: ToolUseContext,
): Promise<AttachmentMessage[]>
```

**作用**：为后台 agent 创建附件，避免重复启动。

### 7.6 truncateToTokens

```ts
function truncateToTokens(content: string, maxTokens: number): string {
    if (roughTokenCountEstimation(content) <= maxTokens) {
        return content
    }
    const charBudget = maxTokens * 4 - SKILL_TRUNCATION_MARKER.length
    return content.slice(0, charBudget) + SKILL_TRUNCATION_MARKER
}

const SKILL_TRUNCATION_MARKER = 
    '\n\n[... skill content truncated for compaction; use Read on the skill path if you need the full text]'
```

### 7.7 shouldExcludeFromPostCompactRestore

```ts
function shouldExcludeFromPostCompactRestore(
    filename: string,
    agentId?: AgentId,
): boolean
```

**排除的文件**：
- Plan 文件
- CLAUDE.md 及其子目录文件

### 7.8 collectReadToolFilePaths

```ts
function collectReadToolFilePaths(messages: Message[]): Set<string>
```

**作用**：扫描消息中的 Read tool_use，收集文件路径。

```ts
// 用于避免重新注入已经在 preservedMessages 中的文件
// Skips Reads whose tool_result is a dedup stub
```

---

## 八、`partialCompactConversation` 部分压缩

### 8.1 函数签名

```ts
export async function partialCompactConversation(
    allMessages: Message[],
    pivotIndex: number,
    context: ToolUseContext,
    cacheSafeParams: CacheSafeParams,
    userFeedback?: string,
    direction: PartialCompactDirection = 'from',
): Promise<CompactionResult>
```

### 8.2 两个方向

```ts
export type PartialCompactDirection = 'from' | 'up_to'
```

| 方向 | 含义 | Cache 影响 |
| --- | --- | --- |
| `'from'` | 总结 pivot **之后**的消息，保留之前的 | 无 cache 命中 |
| `'up_to'` | 总结 pivot **之前**的消息，保留之后的 | 保留 cache 命中 |

### 8.3 'up_to' 必须过滤旧压缩

```ts
// 'up_to' must strip old compact boundaries/summaries: for 'up_to',
// summary_B sits BEFORE kept, so a stale boundary_A in kept wins
// findLastCompactBoundaryIndex's backward scan and drops summary_B.
// 'from' keeps them: summary_B sits AFTER kept (backward scan still
// works), and removing an old summary would lose its covered history.
const messagesToKeep =
    direction === 'up_to'
        ? allMessages.slice(pivotIndex)
            .filter(m =>
                m.type !== 'progress' &&
                !isCompactBoundaryMessage(m) &&
                !(m.type === 'user' && m.isCompactSummary)
            )
        : allMessages.slice(0, pivotIndex).filter(m => m.type !== 'progress')
```

### 8.4 Prompt Cache 优化

```ts
// 'up_to' prefix hits cache directly; 'from' sends all (tail wouldn't cache).
let apiMessages = direction === 'up_to' ? messagesToSummarize : allMessages
let retryCacheSafeParams =
    direction === 'up_to'
        ? {...cacheSafeParams, forkContextMessages: messagesToSummarize}
        : cacheSafeParams
```

### 8.5 Preserved Segment Annotation

```ts
// 'from': prefix-preserving → boundary; 'up_to': suffix → last summary
const anchorUuid =
    direction === 'up_to'
        ? (summaryMessages.at(-1)?.uuid ?? boundaryMarker.uuid)
        : boundaryMarker.uuid

return {
    boundaryMarker: annotateBoundaryWithPreservedSegment(
        boundaryMarker,
        anchorUuid,
        messagesToKeep,
    ),
    summaryMessages,
    messagesToKeep,
    attachments: postCompactFileAttachments,
    hookResults: hookMessages,
    userDisplayMessage: postCompactHookResult.userDisplayMessage,
    preCompactTokenCount,
    postCompactTokenCount,
    compactionUsage,
}
```

---

## 九、关键设计模式

### 9.1 多层防御

```
PTL 重试（CC-1180）→ Fork 模式 → Streaming 兜底
```

### 9.2 Prompt Cache 优化

```ts
// 关键：不设置 maxOutputTokens（避免 cache key 不匹配）
// 关键：skipCacheWrite: true（不污染主 cache）
```

### 9.3 资源清理（finally）

```ts
} finally {
    context.setStreamMode?.('requesting')
    context.setResponseLength?.(() => 0)
    context.onCompactProgress?.({type: 'compact_end'})
    context.setSDKStatus?.null)
}
```

### 9.4 并行附件生成

```ts
const [fileAttachments, asyncAgentAttachments] = await Promise.all([
    createPostCompactFileAttachments(...),
    createAsyncAgentAttachmentsIfNeeded(...),
])
```

### 9.5 增量附件重新声明

```ts
// 压缩会丢失之前的 delta 附件
// 从当前状态重新宣布
for (const att of getDeferredToolsDeltaAttachment(..., [])) {
    postCompactFileAttachments.push(createAttachmentMessage(att))
}
```

---

## 十、关键设计决策

| 决策 | 理由 |
| --- | --- |
| **Fork 优先** | 共享主线程 cache，节省 0.76% 的全局 cache_creation |
| **不设置 maxOutputTokens** | 避免 cache key 不匹配 |
| **skipCacheWrite** | 不污染主 cache |
| **PTL 重试 3 次** | 给用户最后机会，避免卡死 |
| **truncateHeadForPTLRetry** | 按 API-round 分组，保留至少一组 |
| **stripImagesFromMessages** | 图片不影响摘要，但会撑爆 prompt |
| **stripReinjectedAttachments** | 避免重复注入 |
| **文件按时间排序** | 最近读过的文件最可能需要 |
| **Skill 按时间排序 + 截断** | 节省 5-10K tokens/compact |
| **排除 Plan/CLAUDE.md** | 这些文件已经在其他附件中 |
| **Cache Sharing 通过 fork** | 复用主 cache |

---

## 十一、文件依赖关系

```
compact.ts 依赖：
├── src/bootstrap/state.ts            # Session 状态
├── src/services/api/claude.ts       # queryModelWithStreaming
├── src/services/api/errors.ts        # 错误处理
├── src/services/api/withRetry.ts     # 重试逻辑
├── src/services/analytics/*         # 指标
├── src/services/compact/grouping.ts  # 消息分组
├── src/services/compact/prompt.ts    # 压缩 prompt
├── src/utils/forkedAgent.ts          # runForkedAgent
├── src/utils/hooks.ts                # PreCompact/PostCompact Hooks
├── src/utils/messages.ts             # 消息工具
├── src/utils/sessionStart.ts         # SessionStart Hooks
├── src/utils/sessionStorage.ts       # 持久化
└── src/utils/tokens.ts               # Token 计算

compact.ts 被依赖：
├── src/services/compact/autoCompact.ts
├── src/services/compact/reactiveCompact.ts
├── src/cli/print.ts
└── src/query.ts
```

---

## 十二、错误处理总结

### 12.1 三种主要错误

```ts
ERROR_MESSAGE_NOT_ENOUGH_MESSAGES = 'Not enough messages to compact.'
ERROR_MESSAGE_PROMPT_TOO_LONG = 'Conversation too long. Press esc twice to go up a few messages and try again.'
ERROR_MESSAGE_USER_ABORT = 'API Error: Request was aborted.'
ERROR_MESSAGE_INCOMPLETE_RESPONSE = 'Compaction interrupted · This may be due to network issues — please try again.'
```

### 12.2 错误处理策略

```ts
} catch (error) {
    // 只在手动 compact 时显示错误
    // auto-compact 失败会在下次重试
    if (!isAutoCompact) {
        addErrorNotificationIfNeeded(error, context)
    }
    throw error
} finally {
    // 清理资源
}
```

### 12.3 addErrorNotificationIfNeeded

```ts
function addErrorNotificationIfNeeded(
    error: unknown,
    context: Pick<ToolUseContext, 'addNotification'>,
) {
    if (
        !hasExactErrorMessage(error, ERROR_MESSAGE_USER_ABORT) &&
        !hasExactErrorMessage(error, ERROR_MESSAGE_NOT_ENOUGH_MESSAGES)
    ) {
        context.addNotification?.({
            key: 'error-compacting-conversation',
            text: 'Error compacting conversation',
            priority: 'immediate',
            color: 'error',
        })
    }
}
```

---

## 十三、可观测性

### 13.1 主要 Analytics 事件

| 事件 | 触发时机 | 关键字段 |
| --- | --- | --- |
| `tengu_compact` | 压缩完成 | preCompactTokenCount、truePostCompactTokenCount、willRetriggerNextTurn |
| `tengu_compact_failed` | 压缩失败 | reason: prompt_too_long/no_summary/api_error |
| `tengu_compact_ptl_retry` | PTL 重试 | attempt、droppedMessages、remainingMessages |
| `tengu_compact_cache_sharing_success` | Fork cache 共享成功 | cacheHitRate |
| `tengu_compact_cache_sharing_fallback` | Fork cache 共享失败 | reason |
| `tengu_compact_streaming_retry` | Streaming 重试 | attempt、hasStartedStreaming |
| `tengu_partial_compact` | 部分压缩完成 | direction、hasUserFeedback |
| `tengu_partial_compact_failed` | 部分压缩失败 | reason、direction |

### 13.2 关键指标

```ts
// 压缩后是否还会触发再次压缩
willRetriggerNextTurn:
    recompactionInfo !== undefined &&
    truePostCompactTokenCount >= recompactionInfo.autoCompactThreshold,

// Cache 命中率
cacheHitRate:
    result.totalUsage.cache_read_input_tokens > 0
        ? result.totalUsage.cache_read_input_tokens /
          (result.totalUsage.cache_read_input_tokens +
           result.totalUsage.cache_creation_input_tokens +
           result.totalUsage.input_tokens)
        : 0,
```

---

## 十四、总结

### 14.1 compact.ts 的核心价值

| 价值 | 说明 |
| --- | --- |
| **完整压缩流程** | 从 PreCompact Hooks 到 PostCompact Hooks |
| **多层重试** | PTL 重试 + Fork + Streaming |
| **Prompt Cache 优化** | Fork 模式共享主 cache |
| **关键信息保留** | 文件、Plan、Skill、AsyncAgent |
| **可观测性** | 完整 Analytics 记录 |
| **错误处理** | 优雅失败、自动重试 |

### 14.2 关键设计哲学

```ts
// 1. 防御性编程
//    - PTL 重试 3 次
//    - Fork 失败回退到 Streaming
//    - Streaming 失败再重试

// 2. 性能优化
//    - 共享 cache（fork 模式）
//    - 不设置 maxOutputTokens
//    - skipCacheWrite

// 3. 用户体验
//    - 状态可见（compacting、requesting）
//    - 错误友好（消息提示）
//    - 可中断（ESC 取消）

// 4. 可观测性
//    - 详细 Analytics
//    - 调试日志
//    - 性能指标
```

**理解 `compact.ts`，就是理解 Claude Code 如何让长会话可持续的关键！** 🎯