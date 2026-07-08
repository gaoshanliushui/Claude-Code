# Headless 模式设计原理 — `src/cli/print.ts` 深度解读

本文档深入分析 Claude Code 的 **Headless 模式**（非交互模式、`--print` / `-p`）实现，对应源代码 `src/cli/print.ts`（5000+ 行）。

---

## 一、Headless 模式整体定位

### 1.1 什么是 Headless 模式

Headless 模式是 Claude Code 的**非交互式执行模式**，与 REPL（Read-Eval-Print Loop）模式形成对照：

| 维度 | REPL 模式 | Headless 模式 |
| --- | --- | --- |
| 交互方式 | 实时终端对话 | 一次性输入、一次性输出 |
| 适用场景 | 人类用户交互 | 脚本调用、SDK 嵌入、CI/CD |
| 输出形式 | Ink UI 渲染 | stdout（文本/JSON/Stream-JSON） |
| 消息处理 | 即时响应 | 队列批处理 |
| 工具调用权限 | 弹窗询问 | 通过控制协议或权限规则 |

### 1.2 核心特点

Headless 模式的设计有以下显著特点：

1. **协议化输入输出**：基于结构化的 NDJSON 消息流，而非自由文本
2. **无 UI 依赖**：移除所有 React/Ink 组件，纯 Node.js 进程
3. **流式响应**：支持 stream-json 输出，实时推送消息
4. **控制协议**：SDK 客户端可以发送控制消息（如中断、切换模型、设置权限模式）
5. **多任务调度**：支持后台 agent、定时任务、Proactive 模式

---

## 二、整体架构鸟瞰

```
┌─────────────────────────────────────────────────────────┐
│  Entry Point: runHeadless()                              │
├─────────────────────────────────────────────────────────┤
│  1. 初始化阶段                                           │
│     • 性能分析器、Grove 检查、GrowthBook 初始化          │
│     • 设置变化检测订阅、Proactive 激活                   │
│     • 沙箱检查与初始化                                   │
│  2. 加载初始状态                                         │
│     • 加载初始消息（continue/resume/teleport）           │
│     • 加载 MCP 客户端和工具                              │
│  3. 建立 structuredIO（StructuredIO / RemoteIO）        │
│  4. 注册各类监听器                                       │
│     • 权限模式变化监听                                   │
│     • AWS 认证状态监听                                   │
│     • 速率限制监听                                       │
│     • Elicitation 处理                                   │
│     • 钩子事件处理                                       │
│  5. 构建主循环 run()                                     │
│     • drainCommandQueue: 消费消息队列                    │
│     • 调用 ask() -> QueryEngine                          │
│     • 流式输出到 stdout                                  │
│  6. 收尾与退出                                           │
│     • 输出最终结果                                       │
│     • 优雅关闭                                           │
└─────────────────────────────────────────────────────────┘
```

---

## 三、核心数据结构与类型

### 3.1 输入输出流（StructuredIO）

Headless 模式使用一个统一的 `StructuredIO` 抽象，它有两种实现：

- **`StructuredIO`**（默认）：基于 stdin/stdout 的结构化 I/O
- **`RemoteIO`**：基于远程通信的 I/O（用于远程控制场景）

核心职责：
- 解析来自 stdin 的 NDJSON 输入
- 将输出消息写入 stdout
- 管理控制请求与响应
- 处理权限回调、钩子回调、elicitations

### 3.2 输出消息流（output）

`output: Stream<StdoutMessage>` 是 Headless 模式的核心输出通道，所有要发给 SDK 客户端或上层消费者的消息都通过它异步输出。

### 3.3 命令队列与状态管理

Headless 模式通过命令队列管理所有待处理的命令：

```ts
type QueuedCommand = {
  mode: 'prompt' | 'orphaned-permission' | 'task-notification'
  value: string | ContentBlockParam[]
  uuid?: UUID
  priority?: 'now' | 'next' | 'later'
  isMeta?: boolean
  workload?: string
  // ... 其它字段
}
```

---

## 四、`runHeadless()` 主流程详解

### 4.1 入口函数签名

```ts
export async function runHeadless(
    inputPrompt: string | AsyncIterable<string>,
    getAppState: () => AppState,
    setAppState: (f: (prev: AppState) => AppState) => void,
    commands: Command[],
    tools: Tools,
    sdkMcpConfigs: Record<string, McpSdkServerConfig>,
    agents: AgentDefinition[],
    options: { /* 大量配置项 */ },
): Promise<void>
```

参数分类：
| 参数类别 | 关键参数 |
| --- | --- |
| 应用状态 | `getAppState`、`setAppState` |
| 资源 | `commands`、`tools`、`mcpClients`、`agents` |
| 用户输入 | `inputPrompt`（字符串或异步可迭代） |
| 行为控制 | `continue`、`resume`、`resumeSessionAt`、`teleport` |
| 输出格式 | `outputFormat`、`verbose`、`jsonSchema` |
| 限制 | `maxTurns`、`maxBudgetUsd`、`taskBudget` |
| 模型 | `userSpecifiedModel`、`fallbackModel`、`thinkingConfig` |
| 系统提示 | `systemPrompt`、`appendSystemPrompt` |
| 其它 | `replayUserMessages`、`includePartialMessages`、`forkSession` |

### 4.2 主流程 10 大阶段

#### 阶段 1：早期退出与准备（行 494-555）

```ts
// 早期退出：ant 用户快速启动测试
if (process.env.USER_TYPE === 'ant' && isEnvTruthy('CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER')) {
    process.stderr.write(`\nStartup time: ...ms\n`)
    process.exit(0)
}

// 预下载用户设置（与 MCP/工具初始化并行）
if (feature('DOWNLOAD_USER_SETTINGS') && ...) {
    void downloadUserSettings()
}

// 订阅设置变化
settingsChangeDetector.subscribe(source => {
    applySettingsChange(source, setAppState)
    // ...
})

// Proactive 模式激活
if (... && !proactiveModule.isProactiveActive() && isEnvTruthy('CLAUDE_CODE_PROACTIVE')) {
    proactiveModule.activateProactive('command')
}

// Bun 环境下的周期 GC（防止内存累积）
if (typeof Bun !== 'undefined') {
    const gcTimer = setInterval(Bun.gc, 1000)
    gcTimer.unref()
}
```

**设计要点**：
- **预热式并行**：downloadUserSettings 在后台运行，不阻塞主流程
- **强制 GC 兜底**：长会话中通过定时 GC 避免内存累积
- **Proactive 兜底激活**：即便 main.tsx 已检查，print.ts 仍做兜底

#### 阶段 2：参数验证（行 567-585）

```ts
// 参数冲突检测
if (options.resumeSessionAt && !options.resume) { ... }
if (options.rewindFiles && !options.resume) { ... }
if (options.rewindFiles && inputPrompt) { ... }
```

**典型错误**：
- `--resume-session-at` 必须搭配 `--resume`
- `--rewind-files` 必须搭配 `--resume`，且不能有 prompt

#### 阶段 3：structuredIO 与沙箱初始化（行 587-626）

```ts
const structuredIO = getStructuredIO(inputPrompt, options)

// stream-json 模式下安装 stdout 保护
if (options.outputFormat === 'stream-json') {
    installStreamJsonStdoutGuard()
}

// 沙箱检查
const sandboxUnavailableReason = SandboxManager.getSandboxUnavailableReason()
if (sandboxUnavailableReason) {
    if (SandboxManager.isSandboxRequired()) {
        process.stderr.write(`Error: sandbox required but unavailable...`)
        gracefulShutdownSync(1)
        return
    }
    process.stderr.write(`⚠ Sandbox disabled: ...`)
}

// 沙箱初始化（与 SDK 控制协议交互）
else if (SandboxManager.isSandboxingEnabled()) {
    await SandboxManager.initialize(structuredIO.createSandboxAskCallback())
}
```

**stdout 保护机制**：
- 当使用 stream-json 输出时，任何意外写入 stdout 都会破坏 JSON 流
- `installStreamJsonStdoutGuard()` 把非 JSON 行重定向到 stderr

#### 阶段 4：钩子事件处理注册（行 628-674）

```ts
if (options.outputFormat === 'stream-json' && options.verbose) {
    registerHookEventHandler(event => {
        const message: StdoutMessage = (() => {
            switch (event.type) {
                case 'started': return /* hook_started 系统消息 */
                case 'progress': return /* hook_progress */
                case 'response': return /* hook_response */
            }
        })()
        void structuredIO.write(message)
    })
}
```

#### 阶段 5：初始化消息加载（行 676-735）

```ts
if (options.setupTrigger) {
    await processSetupHooks(options.setupTrigger)
}

const { messages: initialMessages, turnInterruptionState, agentSetting } = 
    await loadInitialMessages(setAppState, {
        continue: options.continue,
        teleport: options.teleport,
        resume: options.resume,
        // ...
    })
```

**loadInitialMessages 处理的场景**：
- `continue`: 继续上次会话
- `teleport`: 从远程会话跳转
- `resume`: 恢复历史会话（指定 session ID）
- `resumeSessionAt`: 恢复到指定消息位置
- `forkSession`: 派生新会话

#### 阶段 6：工具与权限初始化（行 795-836）

```ts
// 过滤 MCP 工具的 deny 规则
const allowedMcpTools = filterToolsByDenyRules(appState.mcp.tools, appState.toolPermissionContext)
let filteredTools = [...tools, ...allowedMcpTools]

// 权限提示工具（stdio 模式优先）
const effectivePermissionPromptToolName = options.sdkUrl
    ? 'stdio'
    : options.permissionPromptToolName

// canUseTool 函数（控制权限检查）
const canUseTool = getCanUseToolFn(effectivePermissionPromptToolName, structuredIO, ...)
```

**三种权限模式**：
1. **普通模式**：通过 `hasPermissionsToUseTool` 检查规则
2. **stdio 模式**：通过 `structuredIO.createCanUseTool` 与 SDK 通信
3. **自定义工具模式**：通过用户提供的 MCP 工具处理

#### 阶段 7：流式消息消费循环（行 851-915）

```ts
const needsFullArray = options.outputFormat === 'json' && options.verbose
const messages: SDKMessage[] = []
let lastMessage: SDKMessage | undefined

for await (const message of runHeadlessStreaming(...)) {
    // 简化输出转换
    if (transformToStreamlined) {
        const transformed = transformToStreamlined(message)
        if (transformed) await structuredIO.write(transformed)
    } else if (options.outputFormat === 'stream-json' && options.verbose) {
        await structuredIO.write(message)
    }

    // 跟踪最后一条消息（用于 exit code）
    if (... 不是控制消息、不是 stream_event ...) {
        if (needsFullArray) messages.push(message)
        lastMessage = message
    }
}
```

**过滤的内部消息类型**：
- `control_response`、`control_request`、`control_cancel_request`
- `session_state_changed`、`task_notification`、`task_started` 等内部系统事件
- `stream_event`、`keep_alive`
- `streamlined_text` 等转换产物

#### 阶段 8：最终结果输出（行 917-957）

```ts
switch (options.outputFormat) {
    case 'json':
        if (!lastMessage || lastMessage.type !== 'result') throw new Error('No messages returned')
        if (options.verbose) {
            writeToStdout(jsonStringify(messages) + '\n')  // 完整消息数组
        } else {
            writeToStdout(jsonStringify(lastMessage) + '\n')  // 仅最后一条
        }
        break
    case 'stream-json':
        // 已在前面流式输出
        break
    default:
        // 默认文本模式：根据 result subtype 输出对应文本
        switch (lastMessage.subtype) {
            case 'success': /* 输出 result.result */
            case 'error_during_execution': /* 输出 "Execution error" */
            case 'error_max_turns': /* 输出 "Error: Reached max turns..." */
            // ...
        }
}
```

#### 阶段 9：内存提取与退出（行 959-974）

```ts
// 记录 Headless 性能指标
logHeadlessProfilerTurn()

// 排空内存提取（异步任务）
if (feature('EXTRACT_MEMORIES') && isExtractModeActive()) {
    await extractMemoriesModule.drainPendingExtraction()
}

// 优雅关闭（根据 exit code）
gracefulShutdownSync(
    lastMessage?.type === 'result' && lastMessage.is_error ? 1 : 0
)
```

---

## 五、`runHeadlessStreaming()` 流式主循环

这是 Headless 模式的核心循环，定义一个 `AsyncIterable<StdoutMessage>`。

### 5.1 主要组成

```ts
function runHeadlessStreaming(
    structuredIO: StructuredIO,
    mcpClients: MCPServerConnection[],
    commands: Command[],
    tools: Tools,
    initialMessages: Message[],
    canUseTool: CanUseToolFn,
    sdkMcpConfigs: Record<string, McpSdkServerConfig>,
    // ... 其它参数
): AsyncIterable<StdoutMessage>
```

### 5.2 内部状态机

```ts
let running = false  // 是否正在运行
let runPhase: 'draining_commands' | 'waiting_for_agents' | 'finally_flush' | 'finally_post_flush' | undefined
let inputClosed = false  // 输入是否已关闭
let shutdownPromptInjected = false  // 关闭提示是否已注入
let heldBackResult: StdoutMessage | null = null  // 被暂存的 result 消息
let abortController: AbortController | undefined
```

### 5.3 主循环 `run()`

```ts
const run = async () => {
    if (running) return  // 重入保护
    running = true
    notifySessionStateChanged('running')
    
    try {
        // 1. 解析待执行的命令（drainCommandQueue）
        // 2. 调用 ask() 执行模型推理
        // 3. 转发消息到 output 流
        // 4. 后台 agent 完成检测
    } finally {
        notifySessionStateChanged('idle')
        running = false
        idleTimeout.start()
    }
}
```

### 5.4 关键设计模式

#### (a) 批处理与连续 prompt

```ts
const batch: QueuedCommand[] = [command]
if (command.mode === 'prompt') {
    while (canBatchWith(command, peek(isMainThread))) {
        batch.push(dequeue(isMainThread)!)
    }
}
```

**作用**：在一个长轮次期间到达的多个 prompt 消息会被合并成一个 turn，避免每个消息都触发一次完整的模型调用。

#### (b) 暂存 result 直到后台 agent 完成

```ts
if (message.type === 'result') {
    const currentState = getAppState()
    if (getRunningTasks(currentState).some(t => 
        (t.type === 'local_agent' || t.type === 'local_workflow') && 
        isBackgroundTask(t)
    )) {
        heldBackResult = message  // 暂存，等待后台任务完成
    } else {
        heldBackResult = null
        output.enqueue(message)
    }
}
```

**作用**：当有后台 agent 在运行时，先不返回 result，避免 SDK 客户端误以为会话结束。

#### (c) SDK 事件实时刷新

```ts
// 在 result 消息前后刷新 SDK 事件队列
if (message.type === 'result') {
    for (const event of drainSdkEvents()) {
        output.enqueue(event)
    }
    // ... output result
}
```

**作用**：保证 `task_started`、`task_progress` 等事件能实时出现在流中。

#### (d) 工作负载上下文传递

```ts
await runWithWorkload(cmd.workload ?? options.workload, async () => {
    for await (const message of ask({...})) { ... }
})
```

**作用**：在后台 agent 内部继承 workload 上下文，确保正确的计费归属。

---

## 六、stdin 消息处理

### 6.1 stdin 消息循环

```ts
void (async () => {
    let initialized = false
    for await (const message of structuredIO.structuredInput) {
        // 1. 非 user 事件：completed 通知、控制请求处理
        // 2. user 消息：去重 -> 入队 -> 启动 run()
    }
    inputClosed = true
    // ... 清理工作
})()
```

### 6.2 控制协议（Control Protocol）

Headless 模式实现了完整的 **JSON 控制协议**，允许 SDK 客户端实时控制会话：

| 控制请求 | 作用 |
| --- | --- |
| `interrupt` | 中断当前查询 |
| `end_session` | 终止会话 |
| `initialize` | 初始化会话（传递 system prompt、agents 等） |
| `set_permission_mode` | 设置权限模式 |
| `set_model` | 切换模型 |
| `set_max_thinking_tokens` | 设置思考 token 上限 |
| `mcp_status` | 查询 MCP 服务器状态 |
| `get_context_usage` | 获取上下文使用情况 |
| `mcp_message` | 转发 MCP 消息到 SDK 服务器 |
| `rewind_files` | 回滚文件修改 |
| `cancel_async_message` | 取消队列中的异步消息 |
| `seed_read_state` | 注入文件状态种子 |
| `mcp_set_servers` | 动态添加/移除 MCP 服务器 |
| `reload_plugins` | 重载插件 |
| `mcp_reconnect` | 重连 MCP 服务器 |
| `mcp_toggle` | 启用/禁用 MCP 服务器 |
| `channel_enable` | 启用 channel |
| `mcp_authenticate` | MCP OAuth 认证 |
| `mcp_oauth_callback_url` | OAuth 回调 |
| `claude_authenticate` | Claude OAuth 认证 |
| `claude_oauth_callback` | Claude OAuth 回调 |
| `mcp_clear_auth` | 清除 MCP 认证 |
| `apply_flag_settings` | 应用 flag 设置 |
| `get_settings` | 获取当前设置 |
| `stop_task` | 停止任务 |
| `generate_session_title` | 生成会话标题 |
| `side_question` | 侧边问题（fork 模式） |
| `set_proactive` | 设置 Proactive 模式 |
| `remote_control` | 远程控制开关 |

### 6.3 消息去重机制

```ts
// 跟踪最近 10000 条消息 UUID（环形缓冲区）
const MAX_RECEIVED_UUIDS = 10_000
const receivedMessageUuids = new Set<UUID>()
const receivedMessageUuidsOrder: UUID[] = []

function trackReceivedMessageUuid(uuid: UUID): boolean {
    if (receivedMessageUuids.has(uuid)) return false
    receivedMessageUuids.add(uuid)
    receivedMessageUuidsOrder.push(uuid)
    if (receivedMessageUuidsOrder.length > MAX_RECEIVED_UUIDS) {
        // 驱逐最旧的
        const toEvict = receivedMessageUuidsOrder.splice(0, ...)
        for (const old of toEvict) receivedMessageUuids.delete(old)
    }
    return true
}
```

**双重去重检查**：
1. 历史去重（`doesMessageExistInSession`）：从 transcript 文件中查
2. 运行时去重（`receivedMessageUuids`）：本次会话中已处理过

### 6.4 初始化请求处理（handleInitializeRequest）

```ts
async function handleInitializeRequest(
    request: SDKControlInitializeRequest,
    requestId: string,
    initialized: boolean,
    output: Stream<StdoutMessage>,
    commands: Command[],
    modelInfos: ModelInfo[],
    structuredIO: StructuredIO,
    enableAuthStatus: boolean,
    options: { /* ... */ },
    agents: AgentDefinition[],
    getAppState: () => AppState,
): Promise<void>
```

**处理流程**：
1. 检查是否已初始化
2. 应用 stdin 传入的 systemPrompt / appendSystemPrompt
3. 合并 stdin 传入的 agents
4. 重新评估 main thread agent
5. 注册钩子回调
6. 设置 init json schema
7. 构造 init response
8. 发送 auth status 初始状态

---

## 七、权限与认证机制

### 7.1 三种权限模式

#### (1) 普通模式（无 permissionPromptToolName）

```ts
return async (tool, input, ...) =>
    forceDecision ?? (await hasPermissionsToUseTool(...))
```

走通用权限规则检查。

#### (2) stdio 模式（permissionPromptToolName === 'stdio'）

```ts
return structuredIO.createCanUseTool(onPermissionPrompt)
```

通过 stdio 控制协议与 SDK 客户端通信，远程决定 allow/deny。

#### (3) 自定义 MCP 工具模式

```ts
return async (...) => {
    if (!resolved) {
        const permissionPromptTool = mcpTools.find(...)
        resolved = createCanUseToolWithPermissionPrompt(permissionPromptTool)
    }
    return resolved(...)
}
```

通过用户提供的 MCP 工具处理权限提示。

### 7.2 createCanUseToolWithPermissionPrompt

```ts
export function createCanUseToolWithPermissionPrompt(
    permissionPromptTool: PermissionPromptTool
): CanUseToolFn
```

**核心逻辑**：
1. 先走通用权限检查（`hasPermissionsToUseTool`）
2. 如果是 allow/deny 直接返回
3. 否则并发调用 permission prompt 工具 + 监听 abort 信号
4. 处理结果转换

### 7.3 MCP OAuth 流程

Headless 模式支持完整的 MCP OAuth 认证：

1. `mcp_authenticate`：启动 OAuth 流程，发送 authUrl 给 SDK 客户端
2. SDK 客户端在浏览器中完成认证
3. `mcp_oauth_callback_url`：回调 token 完成交换
4. 自动重连 MCP 服务器

### 7.4 Claude OAuth 流程

类似地，Claude OAuth 通过：
1. `claude_authenticate`：启动 OAuth，返回 manualUrl 和 automaticUrl
2. SDK 客户端在浏览器完成认证
3. `claude_oauth_callback`：传回 authorization code
4. `claude_oauth_wait_for_completion`：等待流程完成

---

## 八、MCP 服务器管理

### 8.1 MCP 客户端的三种来源

```ts
const allMcpClients = [
    ...appState.mcp.clients,           // 来自 main.tsx 初始连接
    ...sdkClients,                      // SDK 提供的 MCP 服务器
    ...dynamicMcpState.clients,         // 动态添加的 MCP 服务器
]
```

### 8.2 动态 MCP 服务器更新

```ts
async function updateSdkMcp() {
    const currentServerNames = new Set(Object.keys(sdkMcpConfigs))
    const connectedServerNames = new Set(sdkClients.map(c => c.name))
    
    const hasNewServers = ...
    const hasRemovedServers = ...
    const hasPendingSdkClients = sdkClients.some(c => c.type === 'pending')
    const hasFailedSdkClients = sdkClients.some(c => c.type === 'failed')
    
    if (haveServersChanged) {
        // 清理已删除的服务器
        // 重新初始化 SDK MCP 服务器
        // 更新 appState
    }
}
```

### 8.3 Elicitation 处理

```ts
function registerElicitationHandlers(clients: MCPServerConnection[]) {
    for (const connection of clients) {
        connection.client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
            // 1. 运行 elicitation 钩子
            const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
            if (hookResponse) return hookResponse
            
            // 2. 通过控制协议委托给 SDK
            const rawResult = await structuredIO.handleElicitation(...)
            
            // 3. 运行 result 钩子
            const result = await runElicitationResultHooks(...)
            return result
        })
    }
}
```

### 8.4 Channel 支持

```ts
function handleChannelEnable(
    requestId: string,
    serverName: string,
    connectionPool: readonly MCPServerConnection[],
    output: Stream<StdoutMessage>,
): void
```

**Channel 是一种特殊的 MCP 通知**：
- 仅允许 allowlist 中的插件
- 通道消息以 priority:'next' 加入队列
- 在 turn 之间被消费

---

## 九、桥接器（Bridge）支持

### 9.1 远程控制模式

```ts
// 启用 bridge
const handle = await initReplBridge({
    onInboundMessage(msg) { /* 处理入站消息 */ },
    onPermissionResponse(response) { /* 转发权限响应 */ },
    onInterrupt() { /* 中断处理 */ },
    onSetModel(model) { /* 模型切换 */ },
    onSetMaxThinkingTokens(maxTokens) { /* 思考 token */ },
    onStateChange(state, detail) { /* 状态变化 */ },
})

// 转发消息到 bridge
bridgeHandle.writeMessages(newMessages)
```

### 9.2 Bridge 设计目的

Bridge 允许远程 IDE（如 Claude.ai Web）作为 SDK 客户端连接到一个 headless 进程：
- **消息转发**：把本地会话消息流式推送到远程
- **权限代理**：远程 UI 处理权限确认
- **模型/思考配置**：远程控制本地行为

### 9.3 forwardMessagesToBridge 实现

```ts
let bridgeLastForwardedIndex = 0

function forwardMessagesToBridge(): void {
    if (!bridgeHandle) return
    const startIndex = Math.min(bridgeLastForwardedIndex, mutableMessages.length)
    const newMessages = mutableMessages
        .slice(startIndex)
        .filter(m => m.type === 'user' || m.type === 'assistant')
    bridgeLastForwardedIndex = mutableMessages.length
    if (newMessages.length > 0) {
        bridgeHandle.writeMessages(newMessages)
    }
}
```

**特点**：
- **增量转发**：通过索引游标避免重复发送
- **类型过滤**：只转发 user 和 assistant 消息
- **多次调用**：每轮中间 + 每轮后各调用一次

---

## 十、性能优化机制

### 10.1 多种 Profiler

Headless 模式集成了多个 Profiler：

```ts
import {
    headlessProfilerStartTurn,
    headlessProfilerCheckpoint,
    logHeadlessProfilerTurn,
} from 'src/utils/headlessProfiler.js'

import {
    startQueryProfile,
    logQueryProfileReport,
} from 'src/utils/queryProfiler.js'
```

**Profiler 检查点**：
- `runHeadless_entry`
- `after_grove_check`
- `before_loadInitialMessages`
- `after_loadInitialMessages`
- `after_modelStrings`
- `before_runHeadlessStreaming`
- `run_entry`
- `before_ask`
- `after_updateSdkMcp`

### 10.2 消息流式输出

```ts
// 在 ask() 内部 yield 消息时立即转发
if (message.type === 'result') {
    for (const event of drainSdkEvents()) output.enqueue(event)
    output.enqueue(message)
}
```

**延迟优化**：
- 默认 json 模式：只输出最后一条
- stream-json 模式：实时输出
- json + verbose：输出完整数组

### 10.3 内存优化

```ts
// 周期性 GC
if (typeof Bun !== 'undefined') {
    const gcTimer = setInterval(Bun.gc, 1000)
    gcTimer.unref()
}
```

### 10.4 idleTimeout

```ts
const idleTimeout = createIdleTimeoutManager(() => !running)
```

自动清理无活动会话的资源。

---

## 十一、Proactive 模式

### 11.1 Proactive 是什么

Proactive 模式让模型在无用户输入时**自主循环思考**：

```ts
const scheduleProactiveTick = feature('PROACTIVE') || feature('KAIROS')
    ? () => {
        setTimeout(() => {
            if (!proactiveModule?.isProactiveActive() || 
                proactiveModule.isProactivePaused() ||
                inputClosed) {
                return
            }
            const tickContent = `<${TICK_TAG}>${new Date().toLocaleTimeString()}</${TICK_TAG}>`
            enqueue({
                mode: 'prompt',
                value: tickContent,
                uuid: randomUUID(),
                priority: 'later',
                isMeta: true,
            })
            void run()
        }, 0)
    }
    : undefined
```

### 11.2 关闭检测与团队关闭提示

当输入关闭但有后台 agent 在运行时，注入 `SHUTDOWN_TEAM_PROMPT`：

```ts
const SHUTDOWN_TEAM_PROMPT = `<system-reminder>
You are running in non-interactive mode and cannot return a response to the user until your team is shut down.

You MUST shut down your team before preparing your final response:
1. Use requestShutdown to ask each team member to shut down gracefully
2. Wait for shutdown approvals
3. Use the cleanup operation to clean up the team
4. Only then provide your final response to the user

The user cannot receive your response until the team is completely shut down.
</system-reminder>

Shut down your team and prepare your final response for the user.`
```

---

## 十二、Prompt 建议生成

### 12.1 功能介绍

SDK 客户端可以开启 `promptSuggestions` 选项，让模型在每次回答后**生成下一步的 prompt 建议**：

```ts
if (options.promptSuggestions && !isEnvDefinedFalsy('CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION')) {
    const state = suggestionState
    state.abortController?.abort()
    const localAbort = new AbortController()
    suggestionState.abortController = localAbort
    
    const cacheSafeParams = getLastCacheSafeParams()
    if (!cacheSafeParams) {
        logSuggestionSuppressed('sdk_no_params', undefined, undefined, 'sdk')
    } else {
        ref.promise = (async () => {
            const result = await tryGenerateSuggestion(
                localAbort,
                mutableMessages,
                getAppState,
                cacheSafeParams,
                'sdk'
            )
            if (!result || localAbort.signal.aborted) return
            
            const suggestionMsg = {
                type: 'prompt_suggestion',
                suggestion: result.suggestion,
                uuid: randomUUID(),
                session_id: getSessionId(),
            }
            
            // 如果 result 被暂存，suggestion 也暂存
            if (heldBackResult) {
                suggestionState.pendingSuggestion = suggestionMsg
            } else {
                suggestionState.lastEmitted = lastEmittedEntry
                output.enqueue(suggestionMsg)
            }
        })()
        suggestionState.inflightPromise = ref.promise
    }
}
```

### 12.2 缓存安全参数复用

`getLastCacheSafeParams()` 获取上次 turn 的缓存快照，复用相同的 system prompt 等参数，让建议生成也**命中 prompt cache**。

---

## 十三、Cron 调度器

### 13.1 Cron 调度

```ts
let cronScheduler: CronScheduler | null = null
if (feature('AGENT_TRIGGERS') && cronSchedulerModule && cronGate?.isKairosCronEnabled()) {
    cronScheduler = cronSchedulerModule.createCronScheduler({
        onFire: prompt => {
            if (inputClosed) return
            enqueue({
                mode: 'prompt',
                value: prompt,
                uuid: randomUUID(),
                priority: 'later',
                isMeta: true,
                workload: WORKLOAD_CRON,
            })
            void run()
        },
        isLoading: () => running || inputClosed,
        getJitterConfig: cronJitterConfigModule?.getCronJitterConfig,
        isKilled: () => !cronGate?.isKairosCronEnabled(),
    })
    cronScheduler.start()
}
```

### 13.2 设计特点

- **后台运行**：定时任务触发后自动入队，不阻塞主循环
- **互斥保护**：`run()` mutex 防止并发 turn
- **输入关闭时忽略**：`inputClosed` 时不再接受新任务

---

## 十四、文件重写（Rewind Files）

### 14.1 功能说明

Headless 模式支持 `--rewind-files <user_message_uuid>`，把文件状态恢复到指定消息时刻：

```ts
async function handleRewindFiles(
    userMessageId: UUID,
    appState: AppState,
    setAppState: (updater: (prev: AppState) => AppState) => void,
    dryRun: boolean,
): Promise<RewindFilesResult> {
    if (!fileHistoryEnabled()) return {canRewind: false, error: 'File rewinding is not enabled.'}
    if (!fileHistoryCanRestore(appState.fileHistory, userMessageId)) {
        return {canRewind: false, error: 'No file checkpoint found for this message.'}
    }
    if (dryRun) {
        const diffStats = await fileHistoryGetDiffStats(appState.fileHistory, userMessageId)
        return {canRewind: true, filesChanged: ..., insertions: ..., deletions: ...}
    }
    try {
        await fileHistoryRewind(updater => setAppState(...), userMessageId)
    } catch (error) {
        return {canRewind: false, error: ...}
    }
    return {canRewind: true}
}
```

### 14.2 Dry-run 模式

`dryRun: true` 时只返回统计信息（哪些文件会变更、增删多少行），不实际执行。

---

## 十五、文件保存（FILE_PERSISTENCE）

```ts
if (feature('FILE_PERSISTENCE') && turnStartTime !== undefined) {
    void executeFilePersistence(
        turnStartTime,
        abortController.signal,
        result => {
            output.enqueue({
                type: 'system',
                subtype: 'files_persisted',
                files: result.files,
                failed: result.failed,
                processed_at: new Date().toISOString(),
                uuid: randomUUID(),
                session_id: getSessionId(),
            })
        },
    )
}
```

**作用**：每轮 turn 结束后异步保存文件，生成 `files_persisted` 事件供 SDK 消费。

---

## 十六、ReadFileState 同步

### 16.1 SDK 客户端可注入文件状态

当 SDK 客户端观察到一次 Read 但 transcript 中已丢失时，可以通过 `seed_read_state` 控制消息重新注入：

```ts
} else if (message.request.subtype === 'seed_read_state') {
    try {
        const normalizedPath = expandPath(message.request.path)
        const diskMtime = Math.floor((await stat(normalizedPath)).mtimeMs)
        if (diskMtime <= message.request.mtime) {
            const raw = await readFile(normalizedPath, 'utf-8')
            // 处理 BOM、CRLF
            const content = (
                raw.charCodeAt(0) === 0xfeff ? raw.slice(1) : raw
            ).replaceAll('\r\n', '\n')
            pendingSeeds.set(normalizedPath, {
                content,
                timestamp: diskMtime,
                offset: undefined,
                limit: undefined,
            })
        }
    } catch {
        // ENOENT 等错误忽略
    }
    sendControlResponseSuccess(message)
}
```

### 16.2 Pending Seeds 应用

```ts
// 在 setReadFileCache 中应用
setReadFileCache: cache => {
    readFileState = cache
    for (const [path, seed] of pendingSeeds.entries()) {
        const existing = readFileState.get(path)
        if (!existing || seed.timestamp > existing.timestamp) {
            readFileState.set(path, seed)
        }
    }
    pendingSeeds.clear()  // 一次性使用
}
```

**关键**：
- **mtime 校验**：避免注入过期内容
- **路径标准化**：expandPath 统一处理 `~`、相对路径
- **BOM/CRLF 规范化**：与 readFileInRange 保持一致
- **一次性使用**：应用后立即清除

---

## 十七、错误处理与优雅退出

### 17.1 SIGINT 处理

```ts
const sigintHandler = () => {
    logForDiagnosticsNoPII('info', 'shutdown_signal', {signal: 'SIGINT'})
    if (abortController && !abortController.signal.aborted) {
        abortController.abort()
    }
    void gracefulShutdown(0)
}
process.on('SIGINT', sigintHandler)
```

### 17.2 异常处理（catch 块）

```ts
} catch (error) {
    try {
        await structuredIO.write({
            type: 'result',
            subtype: 'error_during_execution',
            // ... 包含错误信息
        })
    } catch {
        // 即使 error result 写不出去，也要继续 shutdown
    }
    suggestionState.abortController?.abort()
    gracefulShutdownSync(1)
    return
}
```

### 17.3 finally 块

```ts
} finally {
    runPhase = 'finally_flush'
    await structuredIO.flushInternalEvents()
    runPhase = 'finally_post_flush'
    if (!isShuttingDown()) {
        notifySessionStateChanged('idle')
        // 排空 SDK 事件
        for (const event of drainSdkEvents()) {
            output.enqueue(event)
        }
    }
    running = false
    idleTimeout.start()
}
```

---

## 十八、Headless 与其它模式的关系

```
┌──────────────────────────────────────────────────┐
│                  Claude Code                     │
├──────────────────────────────────────────────────┤
│                                                  │
│   ┌──────────────┐      ┌──────────────┐         │
│   │ REPL Mode    │      │ Headless     │         │
│   │ (交互式 UI)  │      │ Mode (print) │         │
│   └──────┬───────┘      └──────┬───────┘         │
│          │                     │                 │
│          └──────┬──────────────┘                 │
│                 │                                │
│                 ▼                                │
│         ┌──────────────┐                         │
│         │  QueryEngine │  <── ask()             │
│         └──────────────┘                         │
│                 │                                │
│                 ▼                                │
│         ┌──────────────┐                         │
│         │   query.ts   │                         │
│         └──────────────┘                         │
│                                                  │
│   ┌──────────────┐                               │
│   │   SDK Mode   │                               │
│   │ (Process     │  <── 也用 QueryEngine          │
│   │  Transport)  │      + 结构化输入输出          │
│   └──────────────┘                               │
│                                                  │
└──────────────────────────────────────────────────┘
```

Headless 模式在底层复用 `QueryEngine` 和 `query.ts`，但增加了：
- 结构化 I/O
- 命令队列与批处理
- 控制协议
- 权限工具集成
- Proactive / Cron 调度

---

## 十九、关键文件依赖关系

| 依赖 | 作用 |
| --- | --- |
| `src/QueryEngine.ts` | 提供 `ask()` 函数 |
| `src/utils/sessionStorage.ts` | 持久化与恢复 |
| `src/services/mcp/*` | MCP 服务器管理 |
| `src/utils/permissions/*` | 权限规则 |
| `src/utils/hooks.ts` | 钩子事件 |
| `src/utils/queryHelpers.ts` | 辅助函数 |
| `src/utils/headlessProfiler.ts` | 性能分析 |
| `src/cli/structuredIO.ts` | 结构化 I/O 实现 |
| `src/cli/remoteIO.ts` | 远程 I/O 实现 |
| `src/bridge/*` | 远程控制桥接 |
| `src/services/api/grove.ts` | Grove 校验 |

---

## 二十、Headless 设计要点复盘

### 20.1 设计亮点

| 亮点 | 说明 |
| --- | --- |
| **协议化输出** | stream-json 让 SDK 客户端实时消费消息 |
| **丰富控制协议** | 30+ 种控制请求，支持几乎所有运行时配置 |
| **批处理优化** | 连续 prompt 合并为一次模型调用 |
| **后台任务协同** | held-back result 等待后台 agent |
| **优雅降级** | 各种 feature gate 缺失时的兜底逻辑 |
| **内存管理** | 周期 GC、UUID 环形缓冲、idleTimeout |
| **权限灵活性** | 三种模式适应不同部署场景 |
| **远程桥接** | Bridge 支持远程 IDE 作为前端 |

### 20.2 适用场景总结

- **CI/CD 集成**：脚本式调用 Claude Code
- **SDK 嵌入**：把 Claude Code 作为库使用
- **IDE 集成**：通过 Bridge 与 IDE 通信
- **批处理任务**：定时任务、批量文件操作
- **Proactive 模式**：自主思考的 agent 服务

### 20.3 与 REPL 的边界

| 方面 | Headless | REPL |
| --- | --- | --- |
| UI | 无 | Ink + React |
| 输入 | stdin/stdio | 交互式终端 |
| 输出 | stdout NDJSON | 终端渲染 |
| 权限 | 通过 stdio 控制 | 弹窗 |
| 后台 agent | 通过 held-back | 通过 UI |
| 性能优化重点 | 启动时间 + 流式输出 | 响应即时性 |

---

## 二十一、参考与扩展

如需深入了解：
- `src/QueryEngine.ts` —— 核心查询生命周期
- `src/cli/structuredIO.ts` —— 结构化 I/O 实现
- `src/cli/remoteIO.ts` —— 远程 I/O 实现
- `src/services/mcp/*` —— MCP 服务器管理
- `src/utils/hooks.ts` —— 钩子系统
- `src/utils/permissions/*` —— 权限规则