# Claude Code 上下文压缩机制深度解析

> 本文深入分析 Claude Code 如何管理上下文窗口的"长度爆炸"问题。
> 涉及文件：`src/services/compact/*`、`src/query.ts`、`src/utils/context.ts`

---

## 一、为什么需要上下文压缩？

### 1.1 上下文窗口的根本限制

Claude Code 的对话可能持续**数小时甚至数天**，会面临：

1. **Token 限制**：每个模型都有 context window 上限（如 200K）
2. **成本增长**：长对话 = 持续增长的 token 费用
3. **延迟增加**：长 prompt = 慢响应
4. **质量下降**：长 context = 模型容易"忘记"早期指令

### 1.2 压缩的核心目标

```
┌────────────────────────────────────────────────────┐
│            长对话 → 短对话                              │
│                                                      │
│  完整历史：[消息1][消息2]...[消息1000]                  │
│         ↓                                             │
│  压缩后：[摘要消息][最近N条消息]                          │
│                                                      │
│  信息保留：100% 关键信息                                 │
│  Token 减少：~70-90%                                  │
└────────────────────────────────────────────────────┘
```

### 1.3 压缩的多层次策略

Claude Code 采用**五层压缩策略**，从轻量到重量级依次触发：

| 层次 | 名称 | 触发时机 | 处理方式 |
| --- | --- | --- | --- |
| **L1** | **API Microcompact** | 每个 API 请求时 | API 服务端自动压缩 |
| **L2** | **Cached Microcompact** | 工具结果过时 | 删除/替换工具结果内容 |
| **L3** | **Microcompact（时间型）** | 工具结果超过 N 轮 | 用占位符替换 |
| **L4** | **Snip Compact** | `HISTORY_SNIP` 启用时 | 裁剪标记消息 |
| **L5** | **Full Compact** | token 接近上限 | 调用 API 总结整个对话 |
| **L6** | **Reactive Compact** | API 报错（prompt too long） | 实时压缩并重试 |
| **L7** | **Session Memory Compact** | 长期会话 | 持久化摘要到外部 |

---

## 二、压缩系统的目录结构

```
src/services/compact/
├── compact.ts                # 核心压缩逻辑（1700+ 行）
├── microCompact.ts           # Microcompact 实现
├── apiMicrocompact.ts        # API 端 microcompact
├── cachedMicrocompact.ts     # 缓存型 microcompact
├── snipCompact.ts            # Snip 压缩
├── snipProjection.ts         # Snip 边界检测
├── autoCompact.ts            # 自动压缩决策
├── reactiveCompact.ts        # 响应式压缩
├── sessionMemoryCompact.ts   # 会话记忆压缩
├── grouping.ts               # 消息分组
├── prompt.ts                 # 压缩 prompt
├── postCompactCleanup.ts     # 压缩后清理
├── compactWarningHook.ts     # 警告钩子
├── compactWarningState.ts    # 警告状态
├── timeBasedMCConfig.ts       # 时间型 microcompact 配置
└── cachedMCConfig.ts          # 缓存型 microcompact 配置
```

---

## 三、L1: API Microcompact（服务端压缩）

### 3.1 概念

API 服务端（Anthropic）会在**每次请求时**自动压缩对话历史。这是 Claude Code 不直接控制，但依赖的压缩机制。

### 3.2 相关文件

```ts
// src/services/compact/apiMicrocompact.ts
// API 端 microcompact 的逻辑处理
```

### 3.3 工作原理

```ts
// claude.ts 中的处理
const result = await queryModelWithStreaming({
    messages: normalizeMessagesForAPI(messages),
    // ...
});

// API 自动应用 microcompact
// 响应中包含 compaction_usage 信息
```

### 3.4 Token 影响

每次 API 调用都会自动应用 microcompact（如果可用），减少输入 token 但保持核心信息。

---

## 四、L2-L3: Microcompact（工具结果清理）

### 4.1 核心思想

工具结果（tool_result）是消息历史中**最占空间的部分**（可能几 MB）。Microcompact 清理旧的工具结果。

### 4.2 两种 Microcompact

#### (1) Cached Microcompact（缓存型）

```ts
// src/services/compact/cachedMicrocompact.ts
// 跟踪缓存编辑，将"未变化"的消息替换为标记
export function consumePendingCacheEdits(): CacheEditsBlock | null {
    const edits = pendingCacheEdits
    pendingCacheEdits = null
    return edits
}
```

**特点**：
- 利用 Prompt Cache 的 `cache_edits` 字段
- 删除缓存中未变化的消息内容
- 节省 cache_creation token

#### (2) Time-Based Microcompact（时间型）

```ts
// src/services/compact/microCompact.ts
const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'

const COMPACTABLE_TOOLS = new Set<string>([
    FILE_READ_TOOL_NAME,
    GREP_TOOL_NAME,
    GLOB_TOOL_NAME,
    WEB_SEARCH_TOOL_NAME,
    WEB_FETCH_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    SHELL_TOOL_NAMES,
])
```

**特点**：
- 超过 N 轮的 tool_result 替换为 `[Old tool result content cleared]`
- 保留 tool_use_id，确保 API 知道对应关系
- 极大的 token 节省

### 4.3 触发逻辑

```ts
// microCompact.ts 中的核心逻辑
function shouldMicroCompact(
    message: ToolResultMessage,
    ageInTurns: number,
    config: TimeBasedMCConfig
): boolean {
    // 1. 是否在 COMPACTABLE_TOOLS 列表中
    if (!COMPACTABLE_TOOLS.has(message.toolName)) return false
    
    // 2. 是否超过年龄阈值
    if (ageInTurns < config.minTurnsBeforeClear) return false
    
    // 3. 是否被显式锁定（uncleared 标记）
    if (message.uncleared) return false
    
    return true
}
```

### 4.4 占位符替换

```ts
// 替换为占位符
const clearedMessage: ToolResultMessage = {
    ...message,
    content: TIME_BASED_MC_CLEARED_MESSAGE,
}
```

### 4.5 配置灵活性

```ts
// src/services/compact/timeBasedMCConfig.ts
export interface TimeBasedMCConfig {
    minTurnsBeforeClear: number  // 多少轮后清除
    // 不同工具可能有不同配置
}
```

---

## 五、L4: Snip Compact（标记裁剪）

### 5.1 概念

Snip Compact 裁剪带有特殊标记的"僵尸消息"，防止长期会话的内存泄漏。

### 5.2 触发条件

```ts
// src/tools/QueryEngine.ts
async function snipReplay(yielded: Message, store: Message[]) {
    if (!snipProjection.isSnipBoundaryMessage(yielded)) {
        return undefined
    }
    return snipCompact.snipCompactIfNeeded(store, {force: true})
}
```

### 5.3 关键设计：QueryEngine 集成

```ts
// QueryEngine.ts 中的处理
case 'system': {
    const snipResult = this.config.snipReplay?.(message, this.mutableMessages)
    if (snipResult !== undefined) {
        if (snipResult.executed) {
            this.mutableMessages.length = 0
            this.mutableMessages.push(...snipResult.messages)
        }
        break  // 不 yield 给 SDK
    }
    // ...
}
```

**关键点**：
- Snip boundary 消息是"信号"，不作为数据 yield 给 SDK
- 回调重新生成完整的消息列表（替代 mutableMessages）
- 防止长期会话的 mutableMessages 无限增长

### 5.4 设计意图

注释解释：
> The subtype check lives inside the injected callback so feature-gated strings stay out of this file (excluded-strings check).

Snip Compact 是 feature-gated 的，所以回调注入机制使得 QueryEngine 本身不直接引用 snip 模块。

---

## 六、L5: Full Compact（完整压缩）

### 6.1 核心思想

调用一个独立的 LLM API 调用，**总结整个对话历史**，生成一个摘要，替换原始消息。

### 6.2 主流程：`compactConversation`

`src/services/compact/compact.ts` 的 `compactConversation` 是核心：

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

### 6.3 完整流程

```
1. 验证消息数量
   ↓
2. 记录 preCompactTokenCount
   ↓
3. 执行 PreCompact hooks
   ↓
4. 调用 streamCompactSummary（流式生成摘要）
   ↓
5. 处理摘要结果
   ↓
6. 创建 compact boundary marker
   ↓
7. 创建 summary messages
   ↓
8. 重新注入关键附件（文件、计划、skill 等）
   ↓
9. 重新执行 SessionStart hooks
   ↓
10. 标记 post-compaction
   ↓
11. 执行 PostCompact hooks
   ↓
12. 返回 CompactionResult
```

### 6.4 streamCompactSummary：核心调用

```ts
async function streamCompactSummary({
    messages,
    summaryRequest,
    appState,
    context,
    preCompactTokenCount,
    cacheSafeParams,
}): Promise<AssistantMessage> {
    // 1. 优先尝试 Fork 模式（共享 prompt cache）
    if (promptCacheSharingEnabled) {
        try {
            const result = await runForkedAgent({
                promptMessages: [summaryRequest],
                cacheSafeParams,
                canUseTool: createCompactCanUseTool(),  // 压缩期间禁止工具调用
                querySource: 'compact',
                forkLabel: 'compact',
                maxTurns: 1,
                skipCacheWrite: true,
            })
            // ...
        } catch (error) {
            // 回退到 streaming 路径
        }
    }
    
    // 2. 回退：Streaming 模式
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
        thinkingConfig: {type: 'disabled'},  // 压缩期间禁用 thinking
        tools: [FileReadTool],  // 只允许 FileRead
        // ...
        maxOutputTokensOverride: Math.min(
            COMPACT_MAX_OUTPUT_TOKENS,
            getMaxOutputTokensForModel(context.options.mainLoopModel),
        ),
    })
}
```

### 6.5 Prompt Cache 共享（Fork 模式）

```ts
// 关键优化：压缩摘要也走 Fork 路径，复用主 prompt cache
const result = await runForkedAgent({
    promptMessages: [summaryRequest],
    cacheSafeParams,  // 包含主线程的系统 prompt 和 tools
    skipCacheWrite: true,  // 不写新的 cache
    // ...
})
```

**为什么这样设计**：
- 压缩摘要的 prompt 与主线程大部分相同（系统 prompt、工具定义）
- 走 Fork 路径可以**完全复用**主线程的 prompt cache
- 节省 cache_creation token

注释解释：
> DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's prompt cache by sending identical cache-key params...

### 6.6 PTL 重试机制（CC-1180）

如果压缩请求本身超过 prompt 限制，触发重试：

```ts
// CC-1180: compact request itself hit prompt-too-long
for (; ; ) {
    summaryResponse = await streamCompactSummary({...})
    summary = getAssistantMessageText(summaryResponse)
    if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break
    
    // Truncate oldest API-round groups and retry
    ptlAttempts++
    const truncated = ptlAttempts <= MAX_PTL_RETRIES
        ? truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
        : null
    if (!truncated) throw new Error(ERROR_MESSAGE_PROMPT_TOO_LONG)
    messagesToSummarize = truncated
}
```

**关键洞察**：丢弃最老的 API-round groups，保留至少一组可以总结。

### 6.7 truncateHeadForPTLRetry

```ts
export function truncateHeadForPTLRetry(
    messages: Message[],
    ptlResponse: AssistantMessage,
): Message[] | null {
    const groups = groupMessagesByApiRound(messages)
    if (groups.length < 2) return null
    
    // 累积丢弃直到 token 缺口被覆盖
    const tokenGap = getPromptTooLongTokenGap(ptlResponse)
    let dropCount = 0
    let acc = 0
    for (const g of groups) {
        acc += roughTokenCountEstimationForMessages(g)
        dropCount++
        if (acc >= tokenGap) break
    }
    
    // 保留至少一组
    dropCount = Math.min(dropCount, groups.length - 1)
    if (dropCount < 1) return null
    
    const sliced = groups.slice(dropCount).flat()
    
    // 如果切掉后的第一条是 assistant，前面补一个 user marker
    if (sliced[0]?.type === 'assistant') {
        return [
            createUserMessage({content: PTL_RETRY_MARKER, isMeta: true}),
            ...sliced,
        ]
    }
    return sliced
}
```

**设计精妙**：
- 按 API-round 分组（避免破坏 tool_use/tool_result 配对）
- 累积丢弃直到 token 缺口被覆盖
- 保留至少一组可以总结
- 如果切掉后开头不是 user，补一个 marker

### 6.8 压缩后的附件注入

压缩完成后，需要**重新注入关键信息**：

```ts
// 1. 文件附件（最近读过的文件）
const fileAttachments = await createPostCompactFileAttachments(
    preCompactReadFileState,
    context,
    POST_COMPACT_MAX_FILES_TO_RESTORE,  // 默认 5
    messagesToKeep,
)

// 2. 异步 Agent 附件
const asyncAgentAttachments = await createAsyncAgentAttachmentsIfNeeded(context)

// 3. 计划文件附件
const planAttachment = createPlanAttachmentIfNeeded(context.agentId)

// 4. Plan Mode 附件
const planModeAttachment = await createPlanModeAttachmentIfNeeded(context)

// 5. Skill 附件
const skillAttachment = createSkillAttachmentIfNeeded(context.agentId)

// 6. 工具增量附件（重新声明）
for (const att of getDeferredToolsDeltaAttachment(...)) {
    postCompactFileAttachments.push(createAttachmentMessage(att))
}
// Agent 列表增量
// MCP 指令增量
```

### 6.9 createPostCompactFileAttachments

```ts
export async function createPostCompactFileAttachments(
    readFileState: Record<string, { content: string; timestamp: number }>,
    toolUseContext: ToolUseContext,
    maxFiles: number,
    preservedMessages: Message[] = [],
): Promise<AttachmentMessage[]> {
    // 1. 排除已存在在 preservedMessages 中的文件
    const preservedReadPaths = collectReadToolFilePaths(preservedMessages)
    
    // 2. 按时间排序最近的文件
    const recentFiles = Object.entries(readFileState)
        .map(([filename, state]) => ({filename, ...state}))
        .filter(file => 
            !shouldExcludeFromPostCompactRestore(file.filename, toolUseContext.agentId) &&
            !preservedReadPaths.has(expandPath(file.filename))
        )
        .sort((a, b) => b.timestamp - a.timestamp)
        .slice(0, maxFiles)  // 只保留最新的 5 个
    
    // 3. 并发读取文件内容
    const results = await Promise.all(
        recentFiles.map(async file => {
            const attachment = await generateFileAttachment(file.filename, {
                ...toolUseContext,
                fileReadingLimits: {
                    maxTokens: POST_COMPACT_MAX_TOKENS_PER_FILE,  // 5000
                },
            }, 'tengu_post_compact_file_restore_success',
               'tengu_post_compact_file_restore_error',
               'compact')
            return attachment ? createAttachmentMessage(attachment) : null
        }),
    )
    
    // 4. 控制总 token 预算
    let usedTokens = 0
    return results.filter((result): result is AttachmentMessage => {
        if (result === null) return false
        const attachmentTokens = roughTokenCountEstimation(jsonStringify(result))
        if (usedTokens + attachmentTokens <= POST_COMPACT_TOKEN_BUDGET) {  // 50000
            usedTokens += attachmentTokens
            return true
        }
        return false
    })
}
```

**关键设计**：
- 排除 plan 文件、CLAUDE.md 文件
- 按时间排序，只取最近 5 个
- 每个文件最多 5000 tokens
- 总共最多 50000 tokens

### 6.10 createSkillAttachmentIfNeeded

```ts
export function createSkillAttachmentIfNeeded(agentId?: string): AttachmentMessage | null {
    const invokedSkills = getInvokedSkillsForAgent(agentId)
    if (invokedSkills.size === 0) return null
    
    // 按最新调用排序
    let usedTokens = 0
    const skills = Array.from(invokedSkills.values())
        .sort((a, b) => b.invokedAt - a.invokedAt)
        .map(skill => ({
            name: skill.skillName,
            path: skill.skillPath,
            content: truncateToTokens(skill.content, POST_COMPACT_MAX_TOKENS_PER_SKILL),  // 5000
        }))
        .filter(skill => {
            const tokens = roughTokenCountEstimation(skill.content)
            if (usedTokens + tokens > POST_COMPACT_SKILLS_TOKEN_BUDGET) {  // 25000
                return false
            }
            usedTokens += tokens
            return true
        })
    
    if (skills.length === 0) return null
    
    return createAttachmentMessage({type: 'invoked_skills', skills})
}
```

**注释解释**：
> Skills can be large (verify=18.7KB, claude-api=20.1KB). Previously re-injected unbounded on every compact → 5-10K tok/compact.

每个 skill 限制 5000 tokens，总共 25000 tokens。

### 6.11 stripImagesFromMessages

```ts
export function stripImagesFromMessages(messages: Message[]): Message[] {
    return messages.map(message => {
        if (message.type !== 'user') return message
        const content = message.message.content
        if (!Array.isArray(content)) return message
        
        let hasMediaBlock = false
        const newContent = content.flatMap(block => {
            if (block.type === 'image') {
                hasMediaBlock = true
                return [{type: 'text' as const, text: '[image]'}]
            }
            if (block.type === 'document') {
                hasMediaBlock = true
                return [{type: 'text' as const, text: '[document]'}]
            }
            // 也处理 tool_result 内部的 image/document
            // ...
        })
        // ...
    })
}
```

**关键洞察**：图片/文档不影响摘要生成，但会占用大量 token。替换为标记。

### 6.12 CompactResult 接口

```ts
export interface CompactionResult {
    boundaryMarker: SystemMessage           // compact_boundary 系统消息
    summaryMessages: UserMessage[]           // 摘要消息
    attachments: AttachmentMessage[]         // 关键附件
    hookResults: HookResultMessage[]         // session_start hooks 结果
    messagesToKeep?: Message[]               // 部分压缩时保留的消息
    userDisplayMessage?: string              // 用户可见消息
    preCompactTokenCount?: number            // 压缩前 token
    postCompactTokenCount?: number           // 压缩 API 调用总 token
    truePostCompactTokenCount?: number       // 真实压缩后 token
    compactionUsage?: ReturnType<typeof getTokenUsage>  // API 用量
}
```

### 6.13 buildPostCompactMessages

```ts
export function buildPostCompactMessages(result: CompactionResult): Message[] {
    return [
        result.boundaryMarker,
        ...result.summaryMessages,
        ...(result.messagesToKeep ?? []),
        ...result.attachments,
        ...result.hookResults,
    ]
}
```

**顺序很重要**：
1. **boundary marker**（标记压缩边界）
2. **summary messages**（摘要）
3. **messagesToKeep**（保留的消息）
4. **attachments**（重新注入的附件）
5. **hookResults**（SessionStart hooks 结果）

---

## 七、L6: Reactive Compact（响应式压缩）

### 7.1 概念

当 API 报错 `prompt_too_long` 时，**实时压缩并重试**。

### 7.2 触发流程

```
API 请求 → 报错 prompt_too_long
    ↓
reactiveCompact.ts 介入
    ↓
自动 truncate 旧消息
    ↓
重新请求 API
    ↓
成功 / 继续重试
```

### 7.3 与 Full Compact 的区别

| 特征 | Full Compact | Reactive Compact |
| --- | --- | --- |
| **触发** | 接近 token 上限 | API 报错 |
| **方式** | 调用独立 LLM 生成摘要 | 直接截断消息 |
| **质量** | 高（语义保留） | 低（信息可能丢失） |
| **时间** | 长（需要 API 调用） | 短（立即截断） |
| **可见性** | 显示"Compacting" | 用户无感知 |

### 7.4 关键差异

```ts
// reactiveCompact.ts 的处理
// "the proper retry loop that peels from the tail"
// 而 truncateHeadForPTLRetry 是 "dumb-but-safe fallback"
```

---

## 八、L7: Session Memory Compact（长期记忆）

### 8.1 概念

将长期会话的摘要**持久化到磁盘**，跨会话保留关键信息。

### 8.2 相关文件

```ts
// src/services/compact/sessionMemoryCompact.ts
export function trySessionMemoryCompaction(
    context: ToolUseContext
): Promise<void>
```

### 8.3 工作原理

1. 检测会话是否足够长
2. 提取关键摘要
3. 写入 session memory 文件
4. 下次会话启动时恢复

---

## 九、Auto Compact（自动压缩决策）

### 9.1 核心思想

基于 token 使用情况自动决定何时压缩。

### 9.2 阈值计算

```ts
// src/services/compact/autoCompact.ts

const AUTOCOMPACT_BUFFER_TOKENS = 13_000
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
const MANUAL_COMPACT_BUFFER_TOKENS = 3_000

export function getAutoCompactThreshold(model: string): number {
    const effectiveContextWindow = getEffectiveContextWindowSize(model)
    const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
    
    // Override for easier testing
    const envPercent = process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
    if (envPercent) {
        const parsed = parseFloat(envPercent)
        if (!isNaN(parsed) && parsed > 0 && parsed <= 100) {
            const percentageThreshold = Math.floor(effectiveContextWindow * (parsed / 100))
            return Math.min(percentageThreshold, autocompactThreshold)
        }
    }
    
    return autocompactThreshold
}
```

### 9.3 getEffectiveContextWindowSize

```ts
export function getEffectiveContextWindowSize(model: string): number {
    const reservedTokensForSummary = Math.min(
        getMaxOutputTokensForModel(model),
        MAX_OUTPUT_TOKENS_FOR_SUMMARY  // 20000
    )
    let contextWindow = getContextWindowForModel(model, getSdkBetas())
    
    // 支持环境变量覆盖
    const autoCompactWindow = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW
    if (autoCompactWindow) {
        const parsed = parseInt(autoCompactWindow, 10)
        if (!isNaN(parsed) && parsed > 0) {
            contextWindow = Math.min(contextWindow, parsed)
        }
    }
    
    return contextWindow - reservedTokensForSummary
}
```

### 9.4 多次失败的熔断

```ts
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
```

**注释解释**：
> Stop trying autocompact after this many consecutive failures.

避免陷入"压缩失败 → 再压缩 → 再失败"的无限循环。

### 9.5 警告状态

```ts
export function calculateTokenWarningState(
    tokenUsage: number,
    model: string,
): {
    percentLeft: number
    isAboveWarningThreshold: boolean
    isAboveErrorThreshold: boolean
    isAboveAutoCompactThreshold: boolean
}
```

不同的阈值触发不同的 UI 提示：

| 阈值 | 缓冲 | UI 行为 |
| --- | --- | --- |
| **WARNING** | 20000 | 显示警告，建议手动压缩 |
| **ERROR** | 20000 | 显示错误 |
| **AUTOCOMPACT** | 13000 | 自动触发压缩 |

---

## 十、Partial Compact（部分压缩）

### 10.1 概念

用户从消息选择器中选择一个 pivot 点，**只压缩 pivot 一侧的消息**。

### 10.2 核心实现

```ts
// src/services/compact/compact.ts
export async function partialCompactConversation(
    allMessages: Message[],
    pivotIndex: number,
    context: ToolUseContext,
    cacheSafeParams: CacheSafeParams,
    userFeedback?: string,
    direction: PartialCompactDirection = 'from',
): Promise<CompactionResult>
```

### 10.3 两个方向

```ts
export type PartialCompactDirection = 'from' | 'up_to'

// 'from': 总结 pivot 之后的消息，保留之前的
// - Prompt cache for kept (earlier) messages is preserved
// - boundary marker 在前

// 'up_to': 总结 pivot 之前的消息，保留之后的
// - Prompt cache is invalidated since the summary precedes the kept messages
// - 必须在 messagesToKeep 中过滤掉旧的 compact_boundary 和 summary
```

### 10.4 关键差异

```ts
const messagesToSummarize =
    direction === 'up_to'
        ? allMessages.slice(0, pivotIndex)
        : allMessages.slice(pivotIndex)

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

### 10.5 Prompt Cache 优化

```ts
// 'up_to' prefix hits cache directly; 'from' sends all (tail wouldn't cache).
let apiMessages = direction === 'up_to' ? messagesToSummarize : allMessages
let retryCacheSafeParams =
    direction === 'up_to'
        ? {...cacheSafeParams, forkContextMessages: messagesToSummarize}
        : cacheSafeParams
```

**关键洞察**：
- `'up_to'` 模式：前缀相同，命中 cache
- `'from'` 模式：所有消息都发送，cache 无效

### 10.6 Preserved Segment

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
    // ...
}
```

---

## 十一、Snip Compact（标记裁剪）深入

### 11.1 何时触发

```ts
// QueryEngine.ts 中的处理
case 'system': {
    const snipResult = this.config.snipReplay?.(message, this.mutableMessages)
    if (snipResult !== undefined) {
        if (snipResult.executed) {
            this.mutableMessages.length = 0
            this.mutableMessages.push(...snipResult.messages)
        }
        break  // 不 yield
    }
}
```

### 11.2 snipCompactIfNeeded

```ts
// src/services/compact/snipCompact.ts
export function snipCompactIfNeeded(
    store: Message[],
    options: { force?: boolean }
): { messages: Message[]; executed: boolean }
```

**功能**：检测并裁剪"僵尸消息"（被标记但未使用的内容）。

### 11.3 设计意图

防止长期对话中：
- 多次压缩后的残留 summary
- 未完成工具调用后的悬挂消息
- 重复的上下文注入

---

## 十二、Hook 系统集成

### 12.1 PreCompact Hook

```ts
// 在压缩前执行
const hookResult = await executePreCompactHooks(
    {
        trigger: isAutoCompact ? 'auto' : 'manual',
        customInstructions: customInstructions ?? null,
    },
    context.abortController.signal,
)

// 合并 hook 的指令
customInstructions = mergeHookInstructions(
    customInstructions,
    hookResult.newCustomInstructions,
)
const userDisplayMessage = hookResult.userDisplayMessage
```

### 12.2 PostCompact Hook

```ts
// 在压缩后执行
const postCompactHookResult = await executePostCompactHooks(
    {
        trigger: isAutoCompact ? 'auto' : 'manual',
        compactSummary: summary,
    },
    context.abortController.signal,
)
```

### 12.3 SessionStart Hook（压缩后重新执行）

```ts
// 压缩会丢失之前 hook 注入的上下文
// 重新执行以恢复
const hookMessages = await processSessionStartHooks('compact', {
    model: context.options.mainLoopModel,
})
```

---

## 十三、缓存优化策略

### 13.1 Prompt Cache 共享

```ts
// 压缩摘要也走 Fork 路径
const result = await runForkedAgent({
    promptMessages: [summaryRequest],
    cacheSafeParams,  // 主线程的 cache
    skipCacheWrite: true,
    // ...
})

// 注释解释：
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params...
```

### 13.2 统计可观测性

```ts
logEvent('tengu_compact_cache_sharing_success', {
    preCompactTokenCount,
    outputTokens: result.totalUsage.output_tokens,
    cacheReadInputTokens: result.totalUsage.cache_read_input_tokens,
    cacheCreationInputTokens: result.totalUsage.cache_creation_input_tokens,
    cacheHitRate:
        result.totalUsage.cache_read_input_tokens > 0
            ? result.totalUsage.cache_read_input_tokens /
              (result.totalUsage.cache_read_input_tokens +
               result.totalUsage.cache_creation_input_tokens +
               result.totalUsage.input_tokens)
            : 0,
})
```

### 13.3 缓存 break 检测

```ts
import {notifyCompaction} from '../api/promptCacheBreakDetection.js'

// 压缩后重置 cache read 基线
if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
    notifyCompaction(
        context.options.querySource ?? 'compact',
        context.agentId,
    )
}
markPostCompaction()  // 标记下次 API success 是 post-compaction
```

**目的**：避免压缩后的 cache miss 被误报为 cache break。

---

## 十四、压缩的副作用管理

### 14.1 文件状态清理

```ts
// 压缩后清空文件缓存
const preCompactReadFileState = cacheToObject(context.readFileState)
context.readFileState.clear()
context.loadedNestedMemoryPaths?.clear()

// 但保留最重要的几个文件（重新注入）
const fileAttachments = await createPostCompactFileAttachments(
    preCompactReadFileState,
    context,
    POST_COMPACT_MAX_FILES_TO_RESTORE,
)
```

### 14.2 Session Metadata 重写

```ts
// 重写 session metadata，确保不被压缩出去
reAppendSessionMetadata()
```

**注释解释**：
> Re-append session metadata (custom title, tag) so it stays within the 16KB tail window that readLiteMetadata reads for --resume display.

### 14.3 Skill 重置

```ts
// 不重置 sentSkillNames
// Intentionally NOT resetting sentSkillNames: re-injecting the full
// skill_listing (~4K tokens) post-compact is pure cache_creation with
// marginal benefit.
```

**设计权衡**：保留 sentSkillNames 避免重新注入 ~4K tokens 的 skill listing。

---

## 十五、压缩相关的关键常量

```ts
// compact.ts 中的关键常量
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5        // 恢复文件数
export const POST_COMPACT_TOKEN_BUDGET = 50_000           // 附件总 token
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000     // 单文件 token
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000    // 单 skill token
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000    // skill 总 token

// autoCompact.ts 中的关键常量
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
export const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// microCompact.ts 中的常量
const IMAGE_MAX_TOKEN_SIZE = 2000

// grouping.ts
export const COMPACT_MAX_OUTPUT_TOKENS = 8000  // 压缩摘要输出上限
```

---

## 十六、错误处理与重试

### 16.1 PTL 重试

```ts
// truncateHeadForPTLRetry - 最后手段
// 注释：
// This is the last-resort escape hatch for CC-1180 — when the compact request
// itself hits prompt-too-long, the user is otherwise stuck.
```

### 16.2 错误分类

```ts
export const ERROR_MESSAGE_NOT_ENOUGH_MESSAGES =
    'Not enough messages to compact.'
export const ERROR_MESSAGE_PROMPT_TOO_LONG =
    'Conversation too long. Press esc twice to go up a few messages and try again.'
export const ERROR_MESSAGE_USER_ABORT = 'API Error: Request was aborted.'
export const ERROR_MESSAGE_INCOMPLETE_RESPONSE =
    'Compaction interrupted · This may be due to network issues — please try again.'
```

### 16.3 通知机制

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

**关键点**：用户中断和"消息不足"不显示错误通知。

### 16.4 Auto Compact 重试控制

```ts
// 不在自动压缩时显示通知
// Only show the error notification for manual /compact.
// Auto-compact failures are retried on the next turn and the notification is
// confusing when compaction eventually succeeds.
```

---

## 十七、可观测性与统计

### 17.1 核心指标

```ts
logEvent('tengu_compact', {
    preCompactTokenCount,
    postCompactTokenCount: compactionCallTotalTokens,
    truePostCompactTokenCount,
    autoCompactThreshold: recompactionInfo?.autoCompactThreshold ?? -1,
    willRetriggerNextTurn:
        recompactionInfo !== undefined &&
        truePostCompactTokenCount >= recompactionInfo.autoCompactThreshold,
    isAutoCompact,
    querySource,
    queryChainId,
    queryDepth,
    isRecompactionInChain,
    turnsSincePreviousCompact,
    previousCompactTurnId,
    compactionInputTokens,
    compactionOutputTokens,
    compactionCacheReadTokens,
    compactionCacheCreationTokens,
    compactionTotalTokens,
    promptCacheSharingEnabled,
})
```

### 17.2 分析上下文

```ts
// 分析上下文组成（用于统计）
analyzeContext(messages)
```

**注释解释**：
> analyzeContext walks every content block (~11ms on a 4.5K-message session) purely for this telemetry breakdown. Computed here, past the compaction-API await, so the sync walk doesn't starve the render loop before compaction even starts.

### 17.3 Cache Sharing 监控

```ts
logEvent('tengu_compact_cache_sharing_success', {
    preCompactTokenCount,
    outputTokens,
    cacheReadInputTokens,
    cacheCreationInputTokens,
    cacheHitRate,
})

logEvent('tengu_compact_cache_sharing_fallback', {
    reason: 'no_text_response' | 'error',
    preCompactTokenCount,
})
```

---

## 十八、设计要点总结

### 18.1 关键设计决策

| 决策 | 理由 |
| --- | --- |
| **多层压缩策略** | 不同场景不同优化 |
| **Fork 模式压缩** | 复用主线程 prompt cache |
| **重新注入关键信息** | 避免压缩丢失关键上下文 |
| **保留最近文件内容** | 避免重新读取 |
| **过滤图片/文档** | 摘要不需要，节省 token |
| **PTL 重试** | 用户不至于卡死 |
| **失败熔断** | 避免无限重试 |
| **Hook 集成** | 允许用户自定义压缩行为 |

### 18.2 性能优化重点

```
1. Prompt Cache 复用（最大优化）
2. Microcompact 减少工具结果
3. Snip Compact 裁剪僵尸消息
4. 选择性重新注入（避免冗余）
5. 并行附件生成（不阻塞主流程）
```

### 18.3 用户体验考虑

```
1. 显示 "Compacting..." 状态
2. keep-alive 信号（防止 WebSocket 超时）
3. 用户可中断（ESC 取消）
4. 错误友好提示
5. 不影响后续回合（cache 重置）
```

---

## 十九、核心要点总结

### 19.1 Claude Code 压缩系统的关键洞察

1. **多层防御**：L1-L7 不同层次的压缩策略
2. **Prompt Cache 感知**：压缩路径专门为 cache 优化
3. **失败熔断**：避免无限重试循环
4. **关键信息保留**：文件、计划、skill、agents 等
5. **用户友好**：可见状态、可中断、错误提示

### 19.2 关键文件速查

| 文件 | 作用 |
| --- | --- |
| `src/services/compact/compact.ts` | 完整压缩核心 |
| `src/services/compact/microCompact.ts` | Microcompact |
| `src/services/compact/autoCompact.ts` | 自动压缩决策 |
| `src/services/compact/reactiveCompact.ts` | 响应式压缩 |
| `src/services/compact/snipCompact.ts` | Snip 压缩 |
| `src/services/compact/sessionMemoryCompact.ts` | 会话记忆 |
| `src/utils/context.ts` | Context 相关常量 |
| `src/services/api/promptCacheBreakDetection.ts` | Cache break 检测 |

### 19.3 关键概念

- **Compact Boundary**：标记压缩边界的特殊系统消息
- **Preserved Segment**：部分压缩保留的消息段
- **Recompaction**：连续压缩（circuit breaker）
- **Prompt Cache Sharing**：压缩走 Fork 路径共享 cache
- **Post-Compact Attachments**：压缩后必须重新注入的附件

**理解 Claude Code 的压缩系统，就能理解如何设计一个高性能、长会话的 AI Agent！** 🎯