# `QueryEngine.ts`：Headless/SDK 查询生命周期引擎

> 源码位置：`src/QueryEngine.ts`
> 核心类：`QueryEngine`（对话状态持久化 + `submitMessage()` 单轮生成器）
> 便捷包装：`ask()`（单场景一次性调用，内部实例化 `QueryEngine`）
> 调用方：`src/cli/print.ts`（`--prompt` / `-p`）、`@anthropic-ai/claude-code` SDK、Coordinator Mode
> 关联文档：[09-query.md](./09-query.md) · [31-bootstrap-state.md](architecture/31-bootstrap-state.md)

`QueryEngine` 是把 Claude Code 从"交互式 REPL 专用"解耦为"可嵌入 SDK 模块"的关键重构。它把原本分散在 REPL、`ask()`、`print.ts` 中的"会话状态管理、消息持久化、权限拒绝记录、成本追踪、历史裁剪"统一为一个有状态类。其设计目标是：**一个 QueryEngine 实例对应一个连续对话，多次 `submitMessage()` 调用共享同一份状态**。

---

## 1. 整体定位

### 1.1 QueryEngine vs query.ts

| 模块 | 职责 | 边界 |
| --- | --- | --- |
| `src/query.ts` | **单次查询循环**：模型调用 → 工具执行 → 下轮迭代 | 一次 query() 调用对应"0 或多轮（turns）"；不含持久化/跨轮记忆 |
| `src/QueryEngine.ts` | **会话生命周期**：系统 prompt 构建 → 用户输入处理 → 调用 query() → transcript 落盘 → SDK 事件标准化 → 状态保留给下一轮 | 一个实例对应"一整个对话"；含 messages、permissionDenials、readFileState 等状态 |

### 1.2 QueryEngine vs REPL

REPL 使用 `query.ts` 直接，因为：
- 它有 React state / AppState 承载长期状态；
- 它有自己的 `useLogMessages` 写 transcript；
- 它要渲染 UI，不需要标准化 SDK 消息。

QueryEngine 为 **Headless/SDK/Print** 设计：
- 它把"同一份对话历史、同一份读文件缓存、同一组权限拒绝记录"封装在类内部；
- 它把 `query()` 的内部事件（`Message` / `StreamEvent` / `ToolUseSummaryMessage`）统一转换为 SDK 友好的 `SDKMessage`；
- 它提供 `interrupt()`、`getMessages()`、`setModel()` 等受控接口；
- 它的 `ask()` 包装函数提供单场景一次性调用（内部实例化 `QueryEngine`，用完即丢）。

---

## 2. 类结构概览

### 2.1 `QueryEngineConfig`：初始化配置

```ts
export type QueryEngineConfig = {
  // 环境基础
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]

  // 权限控制
  canUseTool: CanUseToolFn
  handleElicitation?: ToolUseContext['handleElicitation']

  // AppState 访问（双向绑定）
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void

  // 初始状态
  initialMessages?: Message[]
  readFileCache: FileStateCache

  // 提示词定制
  customSystemPrompt?: string
  appendSystemPrompt?: string

  // 模型与思考模式
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig

  // 限额与约束
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>

  // 调试与控制
  verbose?: boolean
  replayUserMessages?: boolean
  includePartialMessages?: boolean
  setSDKStatus?: (status: SDKStatus) => void
  abortController?: AbortController
  orphanedPermission?: OrphanedPermission

  // 历史裁剪回调（HISTORY_SNIP feature gate）
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[]
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**设计要点**：
- `getAppState` / `setAppState` 注入式访问，保证 QueryEngine 可以**读写** `alwaysAllowRules.command`、`fileHistory`、`attributionState`；
- `readFileCache` 传入后，`getReadFileState()` 返回类内副本，`ask()` 在 `finally` 块写回（避免跨调用污染）；
- `snipReplay` 是条件注入，因为 `snipCompact` 是 feature-gated 的，QueryEngine 自身不能直接引用（保持 DCE 安全）。

### 2.2 QueryEngine 私有状态

```ts
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]           // 对话历史（跨 submitMessage() 持续）
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]  // 本轮累计的工具拒绝（给 SDK result）
  private totalUsage: NonNullableUsage               // 本轮累计的 token 用量
  private hasHandledOrphanedPermission = false      // 孤儿权限只处理一次的 latch
  private readFileState: FileStateCache             // 读文件缓存（跨轮复用）
  private discoveredSkillNames = new Set<string>()  // 本轮发现的 skills（跨 processUserInput 重建）
  private loadedNestedMemoryPaths = new Set<string>()  // 已加载的 memory 路径（防重复）
```

**关键设计**：
- `discoveredSkillNames` 在**每轮 submitMessage() 开始时清空**，但在该轮内部的两次 `processUserInputContext` 重建之间保留；
- `hasHandledOrphanedPermission` 是 **latch（一旦置 true 不再重置）**，保证同一个 QueryEngine 实例不会重复处理 `orphanedPermission`；
- `readFileState` 与外部传入的 `readFileCache` 是同一个对象引用，所以 `getReadFileState()` 返回实时状态。

### 2.3 公共接口

```ts
class QueryEngine {
  // 核心：提交一条消息，返回异步生成器
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage, void, unknown>

  // 控制：中断当前查询
  interrupt(): void

  // 访问器：获取当前状态
  getMessages(): readonly Message[]
  getReadFileState(): FileStateCache
  getSessionId(): string

  // 配置：动态换模型
  setModel(model: string): void
}
```

---

## 3. `submitMessage()` 完整流程

`submitMessage()` 现在被重构为 14 个子方法，每个子方法职责单一清晰：

```
submitMessage()
├── initializeEnvironment()           // 阶段 1：初始化环境与基础配置
├── buildSystemPrompt()               // 阶段 2：构建系统提示词
├── registerStructuredOutputHookIfNeeded()  // 阶段 3：注册结构化输出钩子
├── createInitialProcessUserInputContext()  // 阶段 4：创建初始 context
├── [孤儿权限处理]                   // 阶段 5：孤儿权限处理（内联）
├── [processUserInput 调用]          // 阶段 6：用户输入处理（内联）
├── persistUserMessages()             // 阶段 7：持久化用户消息
├── filterReplayableMessages()        // 阶段 8：过滤回放消息
├── updateToolPermissionContext()     // 阶段 9：更新权限上下文
├── rebuildProcessUserInputContext()  // 阶段 10：重建 context（第二次）
├── preloadSkillsAndPlugins()         // 阶段 11：预加载技能与插件
├── [yield buildSystemInitMessage()]  // 阶段 12：发出系统初始化消息
├── handleLocalCommandBranch()        // 分支 A：本地命令处理（shouldQuery === false）
└── executeQueryLoop()                // 分支 B：查询循环处理（shouldQuery === true）
    ├── createFileHistorySnapshots()
    ├── processQueryMessage()
    │   ├── persistBoundaryMessageIfNeeded()
    │   ├── persistMessageToTranscript()
    │   ├── handleMessageByType()
    │   │   ├── handleAssistantMessage()
    │   │   ├── handleProgressMessage()
    │   │   ├── handleStreamEvent()
    │   │   ├── handleAttachmentMessage()
    │   │   └── handleSystemMessage()
    │   └── checkQueryLimits()
    └── finalizeQueryResult()
```

### 3.1 阶段 1：`initializeEnvironment()`：初始化环境与基础配置

```ts
private initializeEnvironment(
  cwd: string,
  canUseTool: CanUseToolFn,
  userSpecifiedModel: string | undefined,
  thinkingConfig: ThinkingConfig | undefined
)
```

**核心职责**：
- 清空 `discoveredSkillNames`
- 设置工作目录 `setCwd(cwd)`
- 创建 `wrappedCanUseTool`（装饰器模式，拦截权限拒绝并记录）
- 获取初始模型与思考配置
- 返回初始化对象供后续使用

**关键点**：
- `wrappedCanUseTool` 不改变原始 `canUseTool` 的行为，但在 `result.behavior !== 'allow'` 时记录到 `this.permissionDenials`
- `sdkCompatToolName` 把 `Agent` 工具名转换为 `Task`（SDK 兼容性）

### 3.2 阶段 2：`buildSystemPrompt()`：构建系统提示词

```ts
private async buildSystemPrompt(
  tools: Tools,
  mcpClients: MCPServerConnection[],
  initialAppState: AppState,
  initialMainLoopModel: string,
  customSystemPrompt: string | undefined,
  appendSystemPrompt: string | undefined
)
```

**核心职责**：
- 并行加载 `defaultSystemPrompt`、`userContext`、`systemContext`
- 注入 Coordinator 模式特定的 userContext
- 在自定义 system prompt 模式下注入 memory mechanics prompt
- 组装最终的 systemPrompt

**关键点**：
- `fetchSystemPromptParts` 内部用 `Promise.all` 并行加载三部分，避免串行等待
- `customSystemPrompt` 替换 `defaultSystemPrompt`，但 `appendSystemPrompt` 始终追加
- `hasAutoMemPathOverride()` 检测是否设置 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`

### 3.3 阶段 3：`registerStructuredOutputHookIfNeeded()`：注册结构化输出钩子

```ts
private registerStructuredOutputHookIfNeeded(
  jsonSchema: Record<string, unknown> | undefined,
  tools: Tools,
  setAppState: (f: (prev: AppState) => AppState) => void
)
```

**核心职责**：
- 检查是否存在 `SYNTHETIC_OUTPUT_TOOL_NAME`
- 如果存在且 `jsonSchema` 提供，注册 `registerStructuredOutputEnforcement`

### 3.4 阶段 4：`createInitialProcessUserInputContext()`：创建初始 context

```ts
private createInitialProcessUserInputContext(
  commands: Command[],
  tools: Tools,
  verbose: boolean,
  mainLoopModel: string,
  thinkingConfig: ThinkingConfig,
  mcpClients: MCPServerConnection[],
  customSystemPrompt: string | undefined,
  appendSystemPrompt: string | undefined,
  agents: AgentDefinition[],
  getAppState: () => AppState,
  setAppState: (f: (prev: AppState) => AppState) => void,
  setSDKStatus: ((status: SDKStatus) => void) | undefined
)
```

**核心职责**：
- 构建第一次 `ProcessUserInputContext`
- **关键设计**：这次的 `setMessages` 是可变的，可以修改 `this.mutableMessages`（用于 orphaned permission 与 processUserInput）

### 3.5 阶段 5：孤儿权限处理（内联）

```ts
if (orphanedPermission && !this.hasHandledOrphanedPermission) {
  this.hasHandledOrphanedPermission = true
  for await (const message of handleOrphanedPermission(...)) {
    yield message
  }
}
```

**核心职责**：
- 仅在第一次 `submitMessage()` 时处理孤儿权限
- 使用 latch 保证只处理一次
- 直接 yield 处理过程中的 SDKMessage

### 3.6 阶段 6：用户输入处理（内联）

```ts
const {
  messages: messagesFromUserInput,
  shouldQuery,
  allowedTools,
  model: modelFromUserInput,
  resultText,
} = await processUserInput({ ... })

this.mutableMessages.push(...messagesFromUserInput)
const messages = [...this.mutableMessages]  // 本轮查询快照
```

**核心职责**：
- 调用 `processUserInput` 处理用户输入
- 把返回的消息推入 `mutableMessages`
- 创建本轮查询的消息快照

### 3.7 阶段 7：`persistUserMessages()`：持久化用户消息

```ts
private async persistUserMessages(
  messages: Message[],
  messagesFromUserInput: Message[],
  persistSession: boolean
)
```

**核心职责**：
- 在进入 `query()` 之前就持久化用户消息，防止中途杀死导致 transcript 丢失
- `--bare` 模式下 fire-and-forget，否则 await 保证落盘
- 在 `CLAUDE_CODE_EAGER_FLUSH` 或 `CLAUDE_CODE_IS_COWORK` 为 true 时立即 flush

**为什么在 query() 之前写盘？**
- `for await (const message of query())` 只有在 API 返回响应后才会 yield
- 如果用户在发送后立刻点"停止"（或进程被 SIGKILL），query() 可能还没 yield 任何东西
- 在进入 query() 之前就 `recordTranscript()`，保证用户消息一旦接受就持久化

### 3.8 阶段 8：`filterReplayableMessages()`：过滤需要回放的消息

```ts
private filterReplayableMessages(
  messagesFromUserInput: Message[],
  replayUserMessages: boolean
)
```

**核心职责**：
- 筛选真正需要 replay 的用户消息（排除 meta、tool_result、非用户创作的消息）
- 总是包含 compact_boundary 消息

### 3.9 阶段 9：`updateToolPermissionContext()`：更新工具权限上下文

```ts
private updateToolPermissionContext(
  allowedTools: string[] | undefined,
  setAppState: (f: (prev: AppState) => AppState) => void
)
```

**核心职责**：
- 把 `processUserInput` 返回的 `allowedTools` 更新到 AppState 的 `alwaysAllowRules.command`

### 3.10 阶段 10：`rebuildProcessUserInputContext()`：重建 context（第二次）

```ts
private rebuildProcessUserInputContext(
  messages: Message[],
  commands: Command[],
  tools: Tools,
  verbose: boolean,
  mainLoopModel: string,
  thinkingConfig: ThinkingConfig,
  mcpClients: MCPServerConnection[],
  customSystemPrompt: string | undefined,
  appendSystemPrompt: string | undefined,
  agents: AgentDefinition[],
  getAppState: () => AppState,
  setAppState: (f: (prev: AppState) => AppState) => void,
  setSDKStatus: ((status: SDKStatus) => void) | undefined,
  originalContext: ProcessUserInputContext
)
```

**核心职责**：
- 构建第二次 `ProcessUserInputContext`
- **关键设计**：这次的 `setMessages` 是 no-op，防止后续代码再修改
- 复用第一次 context 的 `updateFileHistoryState` / `updateAttributionState`（避免重复代码）

### 3.11 阶段 11：`preloadSkillsAndPlugins()`：预加载技能与插件

```ts
private async preloadSkillsAndPlugins()
```

**核心职责**：
- 并行加载技能（`getSlashCommandToolSkills(getCwd())`）与插件（`loadAllPluginsCacheOnly()`）
- 使用 profiler 打点

**为什么是 `loadAllPluginsCacheOnly()`？**
- SDK/Headless 场景启动时不能阻塞网络
- CCR 通过 `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` 或 `CLAUDE_CODE_PLUGIN_SEED_DIR` 提前填充缓存
- 要刷新插件，请使用 `/reload-plugins`

### 3.12 阶段 12：发出系统初始化消息（内联）

```ts
yield buildSystemInitMessage({ ... })
headlessProfilerCheckpoint('system_message_yielded')
```

**`buildSystemInitMessage` 的作用**：
- SDK 流的第一条消息
- 携带会话元数据（cwd、可用工具、模型、权限模式、slash 命令、skills、plugins、fast mode）
- 调用方（如 VSCode 扩展）用它渲染工具选择器、命令菜单等

### 3.13 分支 A：`handleLocalCommandBranch()`：本地命令分支（shouldQuery === false）

```ts
private async *handleLocalCommandBranch(...)
```

**核心职责**：
- 直接从 `messagesFromUserInput` 构造 SDK 消息 yield
- 本地命令输出作为 synthetic assistant message yield
- 压缩边界消息直接 yield
- 持久化本地命令结果
- yield 最终的 success result 消息

**关键点**：
- 不需要调用 `query()`，完全本地处理
- `local_command` 类型消息被转换为 synthetic assistant message（这样 RC 渲染为 assistant 风格文本）

### 3.14 分支 B：`executeQueryLoop()`：查询循环处理（shouldQuery === true）

这是最复杂的分支，它又分解为多个子方法：

#### 3.14.1 `createFileHistorySnapshots()`：创建文件历史快照

```ts
private createFileHistorySnapshots(
  messagesFromUserInput: Message[],
  persistSession: boolean,
  setAppState: (f: (prev: AppState) => AppState) => void
)
```

**核心职责**：
- 对真正的"用户消息"创建文件历史快照
- fire-and-forget 处理（void 前缀）

#### 3.14.2 `processQueryMessage()`：处理单条查询消息

```ts
private async processQueryMessage({ ... })
```

**核心职责**：
- 调用 `persistBoundaryMessageIfNeeded()` 处理 compact boundary 前的 flush
- 把消息推入 snapshot 并持久化到 transcript
- 在第一次 transcript 记录后 ack initial user messages
- 调用 `handleMessageByType()` 按类型分发处理
- 调用 `checkQueryLimits()` 检查限额
- 返回更新后的状态（turnCount、hasAcknowledgedInitialMessages 等）

##### 3.14.2.1 `persistBoundaryMessageIfNeeded()`：持久化边界消息

```ts
private async persistBoundaryMessageIfNeeded(...)
```

在 compact boundary 前，flush `preservedSegment` tail 之前的消息到 transcript。

##### 3.14.2.2 `persistMessageToTranscript()`：持久化单条消息

```ts
private async persistMessageToTranscript(...)
```

- assistant 消息：fire-and-forget（因为 `claude.ts` 会突变 `message.usage/stop_reason`）
- 其它消息：await 保证落盘

##### 3.14.2.3 `handleMessageByType()`：按类型分发处理

```ts
private async handleMessageByType({ ... })
```

| 消息类型 | 处理方法 | 核心操作 |
| --- | --- | --- |
| `tombstone` | - | 控制信号，跳过 |
| `assistant` | `handleAssistantMessage()` | 捕获 stop_reason、推入 mutableMessages、yield normalized |
| `progress` | `handleProgressMessage()` | 内联持久化（防止 dedup 问题）、yield normalized |
| `user` | - | 推入 mutableMessages、yield normalized |
| `stream_event` | `handleStreamEvent()` | 累积 token 用量、捕获真实 stop_reason（从 message_delta）、可选 yield partial |
| `attachment` | `handleAttachmentMessage()` | 提取 structured_output、处理 max_turns_reached 提前退出、yield queued_command replay |
| `stream_request_start` | - | 跳过 |
| `system` | `handleSystemMessage()` | 处理 snip boundary、处理 compact boundary（释放前置消息给 GC）、yield api_retry |
| `tool_use_summary` | - | yield tool_use_summary |

###### `handleAssistantMessage()`：处理助理消息

```ts
private handleAssistantMessage(...)
```

- 捕获可能已设置的 `stop_reason`（合成消息）
- 推入 `mutableMessages`
- yield normalized 消息

###### `handleStreamEvent()`：处理流式事件

```ts
private handleStreamEvent(...)
```

- `message_start`：重置 `currentMessageUsage`
- `message_delta`：更新 `currentMessageUsage`，捕获真实 `stop_reason`
- `message_stop`：累积 `this.totalUsage`
- 可选 yield partial message（当 `includePartialMessages` 为 true）

**为什么 stop_reason 要从 stream_event.message_delta 取？**
- assistant 消息在 `content_block_stop` 时 yield，此时 `message.stop_reason` 是 null
- 真实 stop_reason 只在 message_delta 中到达

###### `handleAttachmentMessage()`：处理附件消息

```ts
private handleAttachmentMessage({ ... })
```

- `structured_output`：提取数据到 `structuredOutputFromTool`
- `max_turns_reached`：提前退出并 yield error
- `queued_command`：yield user message replay

###### `handleSystemMessage()`：处理系统消息

```ts
private handleSystemMessage(message: Message & { type: 'system' })
```

- snip boundary：调用 `config.snipReplay` 回调，裁剪历史
- compact boundary：释放前置消息给 GC（`splice(0, boundaryIdx)`），yield compact_boundary 消息
- api_error：yield api_retry 消息

##### 3.14.2.4 `checkQueryLimits()`：检查查询限额

```ts
private async checkQueryLimits({ ... })
```

- 检查 `max_budget_usd` 限额是否超支
- 检查 `structured_output` 重试次数是否超限
- 如超限，yield error result 并返回 `{ shouldReturn: true, resultMessage: ... }`

#### 3.14.3 `finalizeQueryResult()`：完成查询并生成最终结果

```ts
private async *finalizeQueryResult({ ... })
```

**核心职责**：
- 用 `findLast` 找到最后一条有效消息（assistant 或 user，避免 progress/attachment）
- flush transcript 缓冲（在 yield result 之前）
- 检查结果是否成功（`isResultSuccessful`）
- 失败：yield `error_during_execution`，包含诊断前缀与本轮内的错误
- 成功：提取文本结果，yield `success` result

**为什么用 findLast 找 assistant | user？**
- Stop hooks 在 query() 结束后 yield progress/attachment 消息
- 这些消息被推入 `messages` 内联
- 只用 `last(messages)` 会拿到 progress/attachment，而不是真正的 assistant 响应

---

## 4. `ask()` 便捷包装函数

### 4.1 签名与实现

```ts
export async function* ask({ ... }) {
  const engine = new QueryEngine({
    ...,
    readFileCache: cloneFileStateCache(getReadFileCache()),  // 克隆避免污染
    ...(feature('HISTORY_SNIP')
      ? { snipReplay: (yielded, store) => { ... } }
      : {}),
  })

  try {
    yield* engine.submitMessage(prompt, { uuid: promptUuid, isMeta })
  } finally {
    setReadFileCache(engine.getReadFileState())  // 写回缓存
  }
}
```

### 4.2 设计要点

- `ask()` 内部创建 `QueryEngine` 实例，用完即丢（一次性使用）
- `readFileCache` 通过 `cloneFileStateCache()` 克隆（防止内部读操作污染外部缓存）
- `finally` 块把 `engine.getReadFileState()` 写回外部（让外部能拿到新增读缓存）
- `snipReplay` 回调条件注入（仅当 `feature('HISTORY_SNIP')` 为 true）

---

## 5. Feature Gates 与条件行为

| Feature | 作用 | 注入方式 |
| --- | --- | --- |
| `HISTORY_SNIP` | 历史裁剪，防止长期对话内存泄漏 | `snipReplay` 回调注入 |
| `COORDINATOR_MODE` | Coordinator 模式下注入额外 userContext | `getCoordinatorUserContext` 条件导入 |
| `SYNTHETIC_OUTPUT_TOOL` | 结构化输出工具 | 检查工具存在，注册 hook |

---

## 6. 环境变量影响

| 环境变量 | 作用 |
| --- | --- |
| `CLAUDE_CODE_EAGER_FLUSH` | 在关键路径上立即 flushSessionStorage |
| `CLAUDE_CODE_IS_COWORK` | 同上（Cowork 场景） |
| `MAX_STRUCTURED_OUTPUT_RETRIES` | 结构化输出最大重试次数（默认 5） |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | 自动注入 memory mechanics prompt |

---

## 7. 状态持久化总结

### 7.1 跨 `submitMessage()` 调用保持的状态

| 状态 | 类型 | 作用 |
| --- | --- | --- |
| `mutableMessages` | `Message[]` | 对话历史（持续增长，compact 时裁剪） |
| `readFileState` | `FileStateCache` | 读文件缓存（跨轮复用） |
| `totalUsage` | `NonNullableUsage` | 累计 token 用量（跨轮） |
| `permissionDenials` | `SDKPermissionDenial[]` | 累计权限拒绝 |
| `hasHandledOrphanedPermission` | `boolean` | 孤儿权限处理 latch（永不重置） |
| `loadedNestedMemoryPaths` | `Set<string>` | 已加载的 memory 路径（防重复） |

### 7.2 每轮重置的状态

| 状态 | 重置位置 |
| --- | --- |
| `discoveredSkillNames` | `this.discoveredSkillNames.clear()` |

---

## 8. 设计要点复盘

### 8.1 单一职责原则

每个子方法只做一件事，并且做好一件事：
- `initializeEnvironment()` 只负责初始化
- `persistUserMessages()` 只负责持久化用户消息
- `handleAssistantMessage()` 只处理助理消息

### 8.2 装饰器模式

`wrappedCanUseTool` 用装饰器模式拦截权限检查，不改变原始行为，只增加记录拒绝事件的功能。

### 8.3 回调注入与 Feature Gate 隔离

`snipReplay` 通过配置注入，而非直接引用 snip 模块，保证 DCE 安全：
- feature 为 false 时，snip 模块被 tree-shake 掉
- QueryEngine 自身不含任何 feature-gated 的字符串

### 8.4 尽早持久化与 eager flush

- 用户消息在进入 `query()` 之前就持久化（防止中途杀死丢失）
- 在关键节点检查 flush 标志并立即落盘

### 8.5 引用 vs 索引：环形缓冲区水位标记

用 `getInMemoryErrors().at(-1)` 的引用而非索引，避免环形缓冲区 shift 导致水位线滑动。

---

## 9. 维护提醒

### 9.1 添加新 message type 处理时

请记得更新以下位置：
1. `this.mutableMessages.push(message)`（内部状态）
2. `messages.push(message)`（本轮快照）
3. `recordTranscript(messages)`（持久化）
4. `yield* normalizeMessage(message)` 或对应 yield 分支
5. 在 `handleMessageByType()` 中添加对应 case
6. 如需要，创建专门的处理方法（如 `handleXxxMessage()`）

### 9.2 添加新限额检查时

请参考 `error_max_turns` / `error_max_budget_usd` 的模式：
- 先 flush（如果需要）
- 再 yield error result
- 最后 return

### 9.3 修改 context 结构时

注意区分两次 `processUserInputContext` 的区别：
- 第一次：`setMessages` 可变
- 第二次：`setMessages` 是 no-op

---

## 10. 相关文件速查

| 文件 | 作用 |
| --- | --- |
| `src/query.ts` | 查询主循环 |
| `src/utils/queryHelpers.ts` | `handleOrphanedPermission` / `isResultSuccessful` / `normalizeMessage` |
| `src/utils/messages/systemInit.ts` | `buildSystemInitMessage` / `sdkCompatToolName` |
| `src/utils/messages/mappers.ts` | `localCommandOutputToSDKAssistantMessage` / `toSDKCompactMetadata` |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts` |
| `src/utils/sessionStorage.ts` | `recordTranscript` / `flushSessionStorage` |
| `src/cost-tracker.ts` | `getTotalAPIDuration` / `getTotalCost` / `getModelUsage` |
