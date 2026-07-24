# Claude Code 多 Agent 系统架构深度解析

> 本文深入分析 Claude Code 如何支持多 agent 协作（Multi-Agent Orchestration）。
> 涉及文件：`src/tools/AgentTool/*`、`src/utils/forkedAgent.ts`、`src/tasks/*`

---

## 一、整体定位：为什么需要多 Agent？

### 1.1 单 Agent 的局限性

Claude Code 的主 agent（main thread）虽然强大，但面临三个核心挑战：

1. **上下文窗口有限**：长任务会快速消耗 token
2. **职责过载**：搜索、规划、执行、验证不应都由一个 agent 完成
3. **无法并行**：长任务串行执行效率低

### 1.2 多 Agent 的解决方案

Claude Code 通过 **AgentTool** 让主 agent 派生子 agent 来分担工作：

```
                    ┌─────────────────┐
                    │   主 Agent        │
                    │   (main thread)  │
                    └────────┬────────┘
                             │ 派生
            ┌────────────────┼────────────────┐
            ↓                ↓                ↓
     ┌──────────┐      ┌──────────┐     ┌──────────┐
     │ Explore  │      │  Plan    │     │ general  │
     │ (搜索)   │      │ (规划)   │     │ purpose  │
     └──────────┘      └──────────┘     └──────────┘
```

### 1.3 核心价值

| 价值 | 说明 |
| --- | --- |
| **职责分离** | 搜索 agent 只读，规划 agent 只规划 |
| **并行执行** | 多个子 agent 可同时运行 |
| **上下文隔离** | 子 agent 有自己的消息历史 |
| **Prompt Cache 复用** | fork 子 agent 共享父级 cache |
| **专业化** | 每个 agent 有专用的工具和 prompt |

---

## 二、Agent 系统的目录结构

```
src/tools/AgentTool/
├── AgentTool.tsx              # AgentTool 主实现（1448 行）
├── runAgent.ts                # Agent 运行时核心（976 行）
├── constants.ts               # 常量定义
├── loadAgentsDir.ts           # Agent 定义加载
├── agentToolUtils.ts          # 工具函数
├── agentColorManager.ts       # 颜色管理
├── agentDisplay.ts            # 显示管理
├── agentMemory.ts             # Memory 机制
├── agentMemorySnapshot.ts     # Memory 快照
├── UI.tsx                     # UI 渲染
├── prompt.ts                  # Prompt 生成
├── forkSubagent.ts             # Fork 子 agent 实现
├── resumeAgent.ts              # Agent 恢复
├── builtInAgents.ts            # 内置 agent 注册
└── built-in/                  # 内置 agent 定义
    ├── generalPurposeAgent.ts  # 通用 agent
    ├── exploreAgent.ts         # 搜索 agent
    ├── planAgent.ts            # 规划 agent
    ├── claudeCodeGuideAgent.ts # 代码指南 agent
    ├── statuslineSetup.ts      # 状态栏配置 agent
    └── verificationAgent.ts    # 验证 agent
```

---

## 三、Agent 类型体系

### 3.1 内置 Agent（Built-in）

`src/tools/AgentTool/builtInAgents.ts` 注册的所有 agent：

```ts
export function getBuiltInAgents(): AgentDefinition[] {
    const agents: AgentDefinition[] = [
        GENERAL_PURPOSE_AGENT,    // 通用 agent
        STATUSLINE_SETUP_AGENT,  // 状态栏配置
    ]

    if (areExplorePlanAgentsEnabled()) {
        agents.push(EXPLORE_AGENT, PLAN_AGENT)
    }

    // ... 更多 agent
    return agents
}
```

### 3.2 各 Agent 的特点

#### (1) General-Purpose Agent（通用）

```ts
export const GENERAL_PURPOSE_AGENT: BuiltInAgentDefinition = {
    agentType: 'general-purpose',
    whenToUse: 'General-purpose agent for researching complex questions...',
    tools: ['*'],               // 可访问所有工具
    source: 'built-in',
    baseDir: 'built-in',
    getSystemPrompt: getGeneralPurposeSystemPrompt,
}
```

**特点**：
- 可访问所有工具（`tools: ['*']`）
- 默认模型（`getDefaultSubagentModel()`）
- 通用任务处理

#### (2) Explore Agent（搜索）

```ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
    agentType: 'Explore',
    whenToUse: 'Fast agent specialized for exploring codebases...',
    disallowedTools: [
        AGENT_TOOL_NAME,
        EXIT_PLAN_MODE_TOOL_NAME,
        FILE_EDIT_TOOL_NAME,
        FILE_WRITE_TOOL_NAME,
        NOTEBOOK_EDIT_TOOL_NAME,
    ],
    source: 'built-in',
    model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
    omitClaudeMd: true,
}
```

**特点**：
- **只读模式**：禁用所有修改工具
- **快速模型**：使用 haiku
- **跳过 CLAUDE.md**：减少 token 消耗
- **并行搜索**：鼓励并发 grep/read

#### (3) Plan Agent（规划）

类似 Explore agent，专门用于制定计划。

#### (4) Verification Agent（验证）

```ts
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
    agentType: 'verification',
    // 验证实现是否符合计划
}
```

#### (5) Status Line Setup Agent

专门用于配置状态栏。

#### (6) Code Guide Agent

为非 SDK 入口提供的代码指南。

### 3.3 ONE_SHOT_BUILTIN_AGENT_TYPES 常量

```ts
// src/tools/AgentTool/constants.ts
export const ONE_SHOT_BUILTIN_AGENT_TYPES: ReadonlySet<string> = new Set([
    'Explore',
    'Plan',
])

// 注释：Built-in agents that run once and return a report —
// the parent never SendMessages back to continue them.
// Skip the agentId/SendMessage/usage trailer for these
// to save tokens (~135 chars × 34M Explore runs/week).
```

**关键洞察**：Explore 和 Plan 是**一次性 agent**，运行一次就返回报告，父 agent 不会再发消息继续它们。跳过 trailer 可以节省大量 token。

---

## 四、Agent 定义规范

### 4.1 AgentDefinition 类型

```ts
export type AgentDefinition = {
    agentType: string           // agent 类型名（如 "general-purpose"）
    whenToUse: string           // 何时使用（提示给模型）
    tools?: string[]            // 允许的工具列表
    disallowedTools?: string[]  // 禁止的工具列表
    prompt: string              // 系统提示词
    model?: string              // 使用的模型
    effort?: EffortValue         // 推理强度
    permissionMode?: PermissionMode  // 权限模式
    mcpServers?: AgentMcpServerSpec[]  // 专属 MCP 服务器
    hooks?: HooksSettings       // 前置 hooks
    maxTurns?: number            // 最大轮次
    skills?: string[]            // 预加载技能
    initialPrompt?: string       // 初始 prompt
    memory?: 'user' | 'project' | 'local'  // memory 范围
    background?: boolean         // 是否默认后台运行
    isolation?: 'worktree' | 'remote'  // 隔离模式
}
```

### 4.2 前置元数据（Frontmatter）

Agent 可以通过 markdown 文件的 frontmatter 定义：

```yaml
---
name: code-reviewer
description: Reviews code for style and correctness
tools: [Read, Grep, Glob]
disallowedTools: [Write, Edit]
model: sonnet
---

Your agent is to review code...
```

### 4.3 Agent 加载机制

`src/tools/AgentTool/loadAgentsDir.ts` 从多个源加载 agent：

```ts
// 加载内置 agent
const builtinAgents = getBuiltInAgents()

// 加载插件 agent
const pluginAgents = loadPluginAgents()

// 加载 markdown 文件中的 agent
const markdownAgents = loadMarkdownFilesForSubdir('agents')

// 加载 JSON agent
const jsonAgents = loadAgentFromJson(...)

// 合并所有 agent
const allAgents = mergeAgents(builtinAgents, pluginAgents, markdownAgents, jsonAgents)
```

---

## 五、AgentTool 的输入输出

### 5.1 输入 Schema

```ts
// 基础 schema
const baseInputSchema = z.object({
    description: z.string().describe('A short (3-5 word) description of the agent'),
    prompt: z.string().describe('The agent for the agent to perform'),
    subagent_type: z.string().optional().describe('The type of specialized agent'),
    model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
    run_in_background: z.boolean().optional(),
})

// 完整 schema（包含多 agent 参数）
const fullInputSchema = baseInputSchema.merge(multiAgentInputSchema).extend({
    isolation: z.enum(['worktree', 'remote']).optional(),
    cwd: z.string().optional(),
})
```

### 5.2 输出 Schema

```ts
const outputSchema = z.union([
    // 同步执行结果
    z.object({
        status: z.literal('completed'),
        prompt: z.string(),
        // ... agent 的实际输出
    }),
    // 异步启动结果
    z.object({
        status: z.literal('async_launched'),
        agentId: z.string(),
        description: z.string(),
        prompt: z.string(),
        outputFile: z.string(),
        canReadOutputFile: z.boolean().optional(),
    }),
])
```

### 5.3 状态枚举

| 状态 | 含义 |
| --- | --- |
| `completed` | 同步执行完成 |
| `async_launched` | 异步启动（在后台运行） |
| `teammate_spawned` | 启动了 teammate（KAIROS 模式） |
| `remote_launched` | 远程启动（CCR 环境） |

---

## 六、子 Agent 的执行流程

### 6.1 整体流程图

```
主 Agent 调用 AgentTool
    ↓
call() (AgentTool.tsx)
    ↓
参数验证和 agent 查找
    ↓
判断同步/异步/后台/远程
    ↓
设置隔离（worktree/remote）
    ↓
启动子 agent
    ↓
runAgent() 核心执行
    ↓
循环 yield 消息
    ↓
完成/超时/中止
    ↓
清理资源（MCP、hooks、缓存）
```

### 6.2 runAgent 核心

`src/tools/AgentTool/runAgent.ts` 是子 agent 运行的核心：

```ts
export async function* runAgent({
    agentDefinition,        // agent 定义
    promptMessages,         // 用户消息
    toolUseContext,         // 父上下文
    canUseTool,             // 权限检查函数
    isAsync,                // 是否异步
    forkContextMessages,    // fork 时的父消息
    querySource,            // 查询源标识
    override,               // 覆盖（model、system prompt 等）
    model,                  // 模型覆盖
    maxTurns,               // 最大轮次
    availableTools,         // 可用工具
    allowedTools,           // 允许的工具
    worktreePath,           // worktree 路径
    // ... 其它参数
}): AsyncGenerator<Message, void> {
    // 1. 解析 agent 模型
    const resolvedAgentModel = getAgentModel(...)
    
    // 2. 创建 agent ID
    const agentId = override?.agentId ?? createAgentId()
    
    // 3. 注册 Perfetto 追踪
    registerPerfettoAgent(agentId, agentDefinition.agentType, parentId)
    
    // 4. 处理 fork 上下文
    const contextMessages = forkContextMessages 
        ? filterIncompleteToolCalls(forkContextMessages) 
        : []
    
    // 5. 克隆文件状态缓存
    const agentReadFileState = cloneFileStateCache(toolUseContext.readFileState)
    
    // 6. 获取用户/系统上下文
    const [userContext, systemContext] = await Promise.all([
        override?.userContext ?? getUserContext(),
        override?.systemContext ?? getSystemContext(),
    ])
    
    // 7. 优化 context（read-only agent 跳过 claudeMd/gitStatus）
    const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && ...
    
    // 8. 解析 agent 权限模式
    const agentGetAppState = () => {
        // 覆盖权限模式
        // 设置 shouldAvoidPermissionPrompts
        // 设置 awaitAutomatedChecksBeforeDialog
        // 作用域 allowedTools
    }
    
    // 9. 解析可用工具
    const resolvedTools = useExactTools 
        ? availableTools 
        : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
    
    // 10. 构建 agent 系统提示
    const agentSystemPrompt = await getAgentSystemPrompt(...)
    
    // 11. 决定 abortController
    const agentAbortController = override?.abortController 
        ? override.abortController
        : isAsync 
            ? new AbortController()  // 异步 agent 独立
            : toolUseContext.abortController  // 同步 agent 共享
    
    // 12. 执行 SubagentStart hooks
    for await (const hookResult of executeSubagentStartHooks(...)) {
        // 收集 additionalContexts
    }
    
    // 13. 注册 frontmatter hooks
    if (agentDefinition.hooks && hooksAllowedForThisAgent) {
        registerFrontmatterHooks(...)
    }
    
    // 14. 预加载技能
    for (const skillName of skillsToPreload) {
        // 加载技能并添加到初始消息
    }
    
    // 15. 初始化专属 MCP 服务器
    const { clients, tools, cleanup } = await initializeAgentMcpServers(...)
    
    // 16. 创建子 agent 上下文
    const agentToolUseContext = createSubagentContext(toolUseContext, {
        options: agentOptions,
        agentId,
        agentType: agentDefinition.agentType,
        messages: initialMessages,
        readFileState: agentReadFileState,
        abortController: agentAbortController,
        getAppState: agentGetAppState,
        shareSetAppState: !isAsync,
        shareSetResponseLength: true,
        contentReplacementState,
    })
    
    // 17. 记录初始消息（sidechain transcript）
    void recordSidechainTranscript(initialMessages, agentId)
    void writeAgentMetadata(agentId, {...})
    
    // 18. 主循环：调用 query()
    try {
        for await (const message of query({
            messages: initialMessages,
            systemPrompt: agentSystemPrompt,
            userContext: resolvedUserContext,
            systemContext: resolvedSystemContext,
            canUseTool,
            toolUseContext: agentToolUseContext,
            querySource,
            maxTurns: maxTurns ?? agentDefinition.maxTurns,
        })) {
            // 转发 stream_event 用于指标
            if (message.type === 'stream_event' && 
                message.event.type === 'message_start' && 
                message.ttftMs != null) {
                toolUseContext.pushApiMetricsEntry?.(message.ttftMs)
                continue
            }
            
            // 处理 max_turns 信号
            if (message.type === 'attachment' && 
                message.attachment.type === 'max_turns_reached') {
                break
            }
            
            // 记录到 sidechain transcript
            if (isRecordableMessage(message)) {
                await recordSidechainTranscript([message], agentId, lastRecordedUuid)
                if (message.type !== 'progress') {
                    lastRecordedUuid = message.uuid
                }
            }
            
            yield message
        }
    } finally {
        // 清理：MCP、hooks、缓存、文件状态
        await mcpCleanup()
        clearSessionHooks(rootSetAppState, agentId)
        cleanupAgentTracking(agentId)
        agentToolUseContext.readFileState.clear()
        initialMessages.length = 0
        unregisterPerfettoAgent(agentId)
        clearAgentTranscriptSubdir(agentId)
        // 清理 todos、bash tasks、monitor tasks
        rootSetAppState(prev => {
            const {[agentId]: _removed, ...todos} = prev.todos
            return {...prev, todos}
        })
        killShellTasksForAgent(agentId, ...)
    }
}
```

### 6.3 子 Agent 的生命周期

```
创建（创建 AgentId、注册 Perfetto）
    ↓
初始化（解析 model、tools、context、prompt）
    ↓
启动 hooks（SubagentStart）
    ↓
预加载（skills、MCP servers）
    ↓
执行循环（query()）
    ↓
完成/中止
    ↓
清理资源（finally 块）
```

---

## 七、上下文隔离与共享

### 7.1 消息隔离

每个子 agent 有**自己的消息历史**：

```ts
// 父消息 → 子消息
const contextMessages: Message[] = forkContextMessages
    ? filterIncompleteToolCalls(forkContextMessages)  // 过滤不完整的工具调用
    : []
const initialMessages: Message[] = [...contextMessages, ...promptMessages]

// 每个 agent 有自己的 sidechain transcript
void recordSidechainTranscript(initialMessages, agentId)
```

### 7.2 文件状态隔离

```ts
// 子 agent 克隆父的 readFileState（避免污染）
const agentReadFileState = forkContextMessages !== undefined
    ? cloneFileStateCache(toolUseContext.readFileState)
    : createFileStateCacheWithSizeLimit(READ_FILE_STATE_CACHE_SIZE)

// 清理时释放内存
agentToolUseContext.readFileState.clear()
```

### 7.3 状态共享

通过 `getAppState` 闭包共享 AppState：

```ts
const agentGetAppState = () => {
    const state = toolUseContext.getAppState()
    // 可以基于父状态修改权限模式等
    return {
        ...state,
        toolPermissionContext: {
            ...state.toolPermissionContext,
            mode: agentDefinition.permissionMode,
            shouldAvoidPermissionPrompts: true,  // 异步 agent 不显示 UI
        },
    }
}
```

### 7.4 同步 vs 异步 Agent

| 类型 | setAppState | abortController | 适用场景 |
| --- | --- | --- | --- |
| **同步** | 共享 | 共享父 | 一次性任务，主 agent 等待 |
| **异步** | 独立 | 独立 | 后台任务，主 agent 不等待 |

```ts
// createSubagentContext
const agentToolUseContext = createSubagentContext(toolUseContext, {
    // ...
    shareSetAppState: !isAsync,        // 异步 agent 不共享 setAppState
    shareSetResponseLength: true,      // 两者都共享
    abortController: agentAbortController,  // 独立 abort controller
})
```

---

## 八、关键子系统

### 8.1 Subagent 上下文创建

`src/utils/forkedAgent.ts` 中的 `createSubagentContext`：

```ts
export function createSubagentContext(
    parentContext: ToolUseContext,
    overrides: Partial<ToolUseContext> & {
        agentId?: AgentId
        agentType?: string
        shareSetAppState?: boolean
    }
): ToolUseContext {
    return {
        ...parentContext,
        ...overrides,
        agentId: overrides.agentId,
        agentType: overrides.agentType,
        // 异步 agent 不共享 setAppState（避免污染）
        setAppState: overrides.shareSetAppState 
            ? parentContext.setAppState 
            : () => {},  // no-op
        setAppStateForTasks: parentContext.setAppStateForTasks,
        // 独立的 abortController
        abortController: overrides.abortController ?? parentContext.abortController,
        // 独立的消息数组
        messages: overrides.messages ?? parentContext.messages,
        // 独立的 readFileState
        readFileState: overrides.readFileState ?? parentContext.readFileState,
        // 独立的 querySource
        querySource: overrides.querySource ?? parentContext.querySource,
    }
}
```

### 8.2 工具解析（resolveAgentTools）

`src/tools/AgentTool/agentToolUtils.ts`：

```ts
export function resolveAgentTools(
    agentDefinition: AgentDefinition,
    availableTools: Tools,
    isAsync: boolean,
): { resolvedTools: Tools } {
    // 1. 应用 tools 过滤（如果是数组）
    if (Array.isArray(agentDefinition.tools)) {
        if (agentDefinition.tools.includes('*')) {
            // 通配符：所有工具
            return { resolvedTools: availableTools }
        }
        // 白名单
        return {
            resolvedTools: availableTools.filter(t => 
                agentDefinition.tools!.includes(t.name)
            )
        }
    }
    
    // 2. 应用 disallowedTools 黑名单
    if (agentDefinition.disallowedTools) {
        return {
            resolvedTools: availableTools.filter(t =>
                !agentDefinition.disallowedTools!.includes(t.name)
            )
        }
    }
    
    return { resolvedTools: availableTools }
}
```

### 8.3 专属 MCP 服务器

`runAgent.ts` 中的 `initializeAgentMcpServers`：

```ts
async function initializeAgentMcpServers(
    agentDefinition: AgentDefinition,
    parentClients: MCPServerConnection[],
): Promise<{
    clients: MCPServerConnection[]
    tools: Tools
    cleanup: () => Promise<void>
}> {
    // 1. 如果 agent 没有专属 MCP，返回父的
    if (!agentDefinition.mcpServers?.length) {
        return {
            clients: parentClients,
            tools: [],
            cleanup: async () => {},
        }
    }
    
    // 2. 检查 plugin-only 锁定
    if (isRestrictedToPluginOnly('mcp') && !isSourceAdminTrusted(agentDefinition.source)) {
        return { clients: parentClients, tools: [], cleanup: async () => {} }
    }
    
    // 3. 区分：引用 vs 内联定义
    const agentClients: MCPServerConnection[] = []
    const newlyCreatedClients: MCPServerConnection[] = []
    const agentTools: Tool[] = []
    
    for (const spec of agentDefinition.mcpServers) {
        if (typeof spec === 'string') {
            // 引用现有 server（不清理）
            const client = await connectToServer(spec, config)
            agentClients.push(client)
        } else {
            // 内联定义（需要清理）
            const client = await connectToServer(name, config)
            agentClients.push(client)
            newlyCreatedClients.push(client)
        }
        
        // 获取工具
        if (client.type === 'connected') {
            const tools = await fetchToolsForClient(client)
            agentTools.push(...tools)
        }
    }
    
    // 4. 返回合并的客户端 + agent 工具 + 清理函数
    return {
        clients: [...parentClients, ...agentClients],
        tools: agentTools,
        cleanup: async () => {
            // 只清理新建的客户端
            for (const client of newlyCreatedClients) {
                if (client.type === 'connected') {
                    await client.cleanup()
                }
            }
        },
    }
}
```

### 8.4 Skill 预加载

```ts
// 加载 agent frontmatter 中指定的 skills
for (const skillName of skillsToPreload) {
    const allSkills = await getSkillToolCommands(getProjectRoot())
    const resolvedName = resolveSkillName(skillName, allSkills, agentDefinition)
    if (!resolvedName) {
        logForDebugging(`[Agent: ${agentDefinition.agentType}] Warning: Skill '${skillName}' not found`)
        continue
    }
    
    const skill = getCommand(resolvedName, allSkills)
    if (skill.type !== 'prompt') continue
    
    const content = await skill.getPromptForCommand('', toolUseContext)
    
    initialMessages.push(
        createUserMessage({
            content: [{type: 'text', text: formatSkillLoadingMetadata(...)}, ...content],
            isMeta: true,  // 元消息，不显示在 transcript
        })
    )
}
```

### 8.5 Hooks 注册

```ts
// 注册 agent frontmatter 的 hooks
if (agentDefinition.hooks && hooksAllowedForThisAgent) {
    registerFrontmatterHooks(
        rootSetAppState,
        agentId,
        agentDefinition.hooks,
        `agent '${agentDefinition.agentType}'`,
        true,  // isAgent - 将 Stop 转换为 SubagentStop
    )
}
```

**关键点**：子 agent 触发 `SubagentStop`，而**不是** `Stop`。这是 AgentTool 在 `isAgent=true` 时自动转换的。

---

## 九、Fork Subagent：Prompt Cache 优化

### 9.1 Fork 的核心思想

Fork subagent 是 Claude Code 的一项性能优化：

> 让子 agent **完全继承父级上下文**，使 API 请求前缀**字节级一致**，从而命中 prompt cache。

### 9.2 Fork 触发条件

```ts
// src/tools/AgentTool/forkSubagent.ts
export function isForkSubagentEnabled(): boolean {
    if (feature('FORK_SUBAGENT')) {
        if (isCoordinatorMode()) return false
        if (getIsNonInteractiveSession()) return false
        return true
    }
    return false
}
```

**Fork 触发**：
- feature gate `FORK_SUBAGENT` 启用
- 不在 coordinator 模式
- 不在非交互会话（headless）

### 9.3 Fork 的实现

```ts
export const FORK_AGENT = {
    agentType: FORK_SUBAGENT_TYPE,
    tools: ['*'],
    maxTurns: 200,
    model: 'inherit',
    permissionMode: 'bubble',
    source: 'built-in',
    baseDir: 'built-in',
    getSystemPrompt: () => '',
}
```

**关键设计**：
- `tools: ['*']`：继承所有工具
- `model: 'inherit'`：继承父模型
- `permissionMode: 'bubble'`：权限请求冒泡到父终端
- `getSystemPrompt: () => ''`：使用 override 传入的 system prompt

### 9.4 Fork 的字节级一致

```ts
// 构建 fork 的消息列表
export function buildForkedMessages(
    parentMessages: Message[]
): Message[] {
    // 1. 保留完整的父 assistant 消息
    // 2. 用占位符替换所有 tool_result
    for (const message of parentMessages) {
        if (message.type === 'user') {
            // 替换 tool_result 内容
            for (const block of message.message.content) {
                if (block.type === 'tool_result') {
                    block.content = FORK_PLACEHOLDER_RESULT
                }
            }
        }
    }
    return parentMessages
}
```

**关键洞察**：用占位符替换 tool_result 而不是删除，确保消息结构完全一致。

### 9.5 递归 Fork 防护

```ts
export function isInForkChild(messages: MessageType[]): boolean {
    return messages.some(m => {
        if (m.type !== 'user') return false
        const content = m.message.content
        if (!Array.isArray(content)) return false
        return content.some(
            block =>
                block.type === 'text' &&
                block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
        )
    })
}
```

防止 fork → fork → fork 的无限递归。

---

## 十、异步 Agent 与任务管理

### 10.1 异步 Agent 类型

```ts
// 后台 agent（async_launched 状态）
const asyncOutputSchema = z.object({
    status: z.literal('async_launched'),
    agentId: z.string(),
    description: z.string(),
    prompt: z.string(),
    outputFile: z.string(),
    canReadOutputFile: z.boolean().optional(),
})
```

### 10.2 LocalAgentTask

异步 agent 通过 `LocalAgentTask` 管理：

```ts
// src/tasks/LocalAgentTask/LocalAgentTask.ts

// 注册异步 agent
export function registerAsyncAgent(
    agentId: AgentId,
    taskType: 'local_agent' | 'local_workflow',
    description: string,
    // ...
)

// 完成异步 agent
export function completeAgentTask(
    agentId: AgentId,
    result: AgentResult
): void

// 失败
export function failAgentTask(
    agentId: AgentId,
    error: Error
): void
```

### 10.3 任务通知机制

异步 agent 完成后，主 agent 收到通知：

```ts
// 任务通知格式
const notificationText = `<task-id>${taskId}</task-id>
<tool-use-id>${toolUseId}</tool-use-id>
<output-file>${outputFile}</output-file>
<status>${status}</status>
<summary>${summary}</summary>`

// 推入消息队列
enqueue({
    mode: 'agent-notification',
    value: notificationText,
    // ...
})
```

### 10.4 自动后台模式

```ts
function getAutoBackgroundMs(): number {
    if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS) || 
        getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents', false)) {
        return 120_000  // 2 分钟后自动后台
    }
    return 0
}
```

超过 2 分钟的任务自动转为后台执行。

---

## 十一、Agent Swarm（多 Agent 协作）

### 11.1 Swarm 模式（KAIROS feature gate）

当启用 KAIROS 时，多个 agent 可以组成一个团队：

```ts
const multiAgentInputSchema = z.object({
    name: z.string().optional().describe('Name for the spawned agent. Makes it addressable via SendMessage({to: name}) while running.'),
    team_name: z.string().optional().describe('Team name for spawning. Uses current team context if omitted.'),
    mode: permissionModeSchema().optional().describe('Permission mode for spawned teammate'),
})
```

### 11.2 Teammate 状态

```ts
// AppState 中的团队上下文
teamContext?: {
    teamName: string
    teamFilePath: string
    leadAgentId: string
    selfAgentId?: string       // 当前 agent 的 ID
    selfAgentName?: string     // 当前 agent 的名字
    isLeader?: boolean         // 是否是 leader
    teammates: {
        [teammateId: string]: {
            name: string
            agentType?: string
            color?: string
            tmuxSessionName: string  // tmux session 名
            tmuxPaneId: string       // tmux pane ID
            cwd: string
            spawnedAt: number
        }
    }
}
```

### 11.3 Inbox 消息系统

Team 成员之间通过 Inbox 通信：

```ts
inbox: {
    messages: Array<{
        id: string
        from: string
        text: string
        timestamp: string
        status: 'pending' | 'processing' | 'processed'
        color?: string
        summary?: string
    }>
}
```

### 11.4 tmux 集成

Swarm 中的 agent 运行在**独立的 tmux pane** 中：

```ts
function spawnTeammate({
    name,
    agentType,
    teamName,
    // ...
}): {
    tmux_session_name: string
    tmux_window_name: string
    tmux_pane_id: string
}
```

### 11.5 Shutdown 协议

当输入关闭时，team-lead 必须关闭团队：

```ts
const SHUTDOWN_TEAM_PROMPT = `<system-reminder>
You are running in non-interactive mode and cannot return a response to the user until your team is shut down.

You MUST shut down your team before preparing your final response:
1. Use requestShutdown to ask each team member to shut down gracefully
2. Wait for shutdown approvals
3. Use the cleanup operation to clean up the team
4. Only then provide your final response to the user
</system-reminder>`
```

---

## 十二、Remote Agent（CCR 环境）

### 12.1 远程启动

```ts
const remoteLaunchedOutput = {
    status: 'remote_launched',
    taskId: string,
    sessionUrl: string,
    description: string,
    prompt: string,
    outputFile: string,
}
```

### 12.2 Remote Task 管理

```ts
// src/tasks/RemoteAgentTask/RemoteAgentTask.ts
export function checkRemoteAgentEligibility(
    prompt: string,
    context: ToolUseContext
): Promise<RemoteAgentCheck>

export function registerRemoteAgentTask(
    taskId: string,
    agentType: string,
    prompt: string
): void

export function getRemoteTaskSessionUrl(taskId: string): string
```

### 12.3 Isolation: "remote" 选项

```ts
isolation: z.enum(['worktree', 'remote']).optional()
```

- `worktree`：本地 git worktree
- `remote`：在 CCR（Claude Code Remote）环境中运行

### 12.4 Teleport 机制

```ts
// src/utils/teleport.ts
export function teleportToRemote(
    sessionUrl: string,
    options: TeleportOptions
): Promise<TeleportResult>
```

允许将当前会话跳转到远程环境。

---

## 十三、Worktree Isolation

### 13.1 创建临时 Worktree

```ts
// src/utils/worktree.ts
export async function createAgentWorktree(
    agentId: string,
    baseDir: string
): Promise<string>  // returns worktree path

export async function hasWorktreeChanges(
    worktreePath: string
): Promise<boolean>

export async function removeAgentWorktree(
    worktreePath: string
): Promise<void>
```

### 13.2 Worktree 生命周期

```
创建 worktree
    ↓
agent 在 worktree 中工作
    ↓
完成/中止
    ↓
检查 worktree 是否有变更
    ↓
合并（如果有）or 删除
```

### 13.3 metadata 持久化

```ts
// 在 agent metadata 中记录 worktree path
void writeAgentMetadata(agentId, {
    agentType: agentDefinition.agentType,
    worktreePath,  // 持久化
    description,
})
```

这样 resume 时能恢复到正确的 cwd。

---

## 十四、Agent Resume（恢复）

### 14.1 恢复机制

```ts
// src/tools/AgentTool/resumeAgent.ts
export async function* resumeAgent({
    agentId,
    toolUseId,
}): AsyncGenerator<Message> {
    // 1. 从 sidechain transcript 读取消息历史
    const messages = await readSidechainTranscript(agentId)
    
    // 2. 查找未完成的工具调用
    const unresolvedToolUse = findUnresolvedToolUse(messages)
    
    // 3. 重新执行未完成的工具
    // 4. yield 给调用方
}
```

### 14.2 Task Notification 后的恢复

```ts
// 父 agent 收到 agent-notification 后
// 可以选择：
// 1. 直接读取结果（如果可以）
// 2. 调用 SendMessage 继续对话
// 3. 调用 resumeAgent 恢复执行
```

### 14.3 恢复的挑战

- **进度丢失**：异步 agent 长时间运行后进度信息可能丢失
- **状态恢复**：worktree 路径、文件状态需要恢复
- **中断处理**：父 agent 中断时如何恢复子 agent

---

## 十五、性能优化

### 15.1 Token 节省策略

```ts
// 1. Explore/Plan 跳过 CLAUDE.md
const shouldOmitClaudeMd = 
    agentDefinition.omitClaudeMd && !override?.userContext &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)

// 2. Explore/Plan 跳过 gitStatus（自己运行 git status 获取新鲜数据）
const {gitStatus: _omittedGitStatus, ...systemContextNoGit} = baseSystemContext

// 3. Explore/Plan 禁用 thinking
thinkingConfig: useExactTools
    ? toolUseContext.options.thinkingConfig
    : {type: 'disabled' as const},

// 4. ONE_SHOT_BUILTIN_AGENT_TYPES 跳过 trailer
// ~135 chars × 34M Explore runs/week = 巨大节省
```

### 15.2 Prompt Cache 复用

```ts
// Fork subagent：字节级一致
// 所有 fork 孩子产生相同的 API 请求前缀
// 命中 server-side prompt cache

// 普通 subagent：上下文隔离
// 每个 subagent 有自己的 transcript
// 不命中 cache（但避免污染父的 cache）
```

### 15.3 并行执行

```ts
// 父 agent 可以同时启动多个子 agent
// 每个子 agent 独立运行
// 通过 agent-notification 机制报告完成
```

### 15.4 工具过滤

```ts
// Explore/Plan 禁用修改工具
disallowedTools: [
    AGENT_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
]
// 减少工具数量 = 减少 token
```

---

## 十六、清理与资源管理

### 16.1 finally 块的清理

```ts
try {
    // 子 agent 执行
} finally {
    // 清理 1：专属 MCP 服务器
    await mcpCleanup()
    
    // 清理 2：session hooks
    if (agentDefinition.hooks) {
        clearSessionHooks(rootSetAppState, agentId)
    }
    
    // 清理 3：prompt cache tracking
    if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
        cleanupAgentTracking(agentId)
    }
    
    // 清理 4：文件状态缓存
    agentToolUseContext.readFileState.clear()
    
    // 清理 5：消息数组
    initialMessages.length = 0
    
    // 清理 6：Perfetto 追踪
    unregisterPerfettoAgent(agentId)
    
    // 清理 7：transcript subdir 映射
    clearAgentTranscriptSubdir(agentId)
    
    // 清理 8：todos
    rootSetAppState(prev => {
        if (!(agentId in prev.todos)) return prev
        const {[agentId]: _removed, ...todos} = prev.todos
        return {...prev, todos}
    })
    
    // 清理 9：background bash tasks
    killShellTasksForAgent(agentId, ...)
    
    // 清理 10：monitor MCP tasks
    mcpMod.killMonitorMcpTasksForAgent(agentId, ...)
}
```

**关键洞察**：每个资源都有明确的清理路径，避免内存泄漏。

### 16.2 transcript 持久化

```ts
// 每次 query() yield 消息时，记录到 sidechain transcript
if (isRecordableMessage(message)) {
    await recordSidechainTranscript([message], agentId, lastRecordedUuid)
    if (message.type !== 'progress') {
        lastRecordedUuid = message.uuid
    }
}
```

---

## 十七、Agent 系统的关键设计模式

### 17.1 继承 + 特化模式

每个 agent 都是 **base + 特化** 的组合：

```ts
{
    // 继承：所有 agent 的基础
    agentType, whenToUse,
    
    // 特化：每个 agent 的差异
    tools: ['*'],            // 通用 agent
    disallowedTools: [...],  // 只读 agent
    model: 'haiku',          // 快速 agent
    permissionMode: 'plan',  // 规划 agent
}
```

### 17.2 装饰器模式（Context Override）

```ts
const agentGetAppState = () => {
    const state = toolUseContext.getAppState()
    // 装饰：在父状态基础上修改
    return {
        ...state,
        toolPermissionContext: {
            ...state.toolPermissionContext,
            mode: agentDefinition.permissionMode,
            shouldAvoidPermissionPrompts: true,
        },
    }
}
```

### 17.3 资源管理（RAII 模式）

```ts
const { clients, tools, cleanup } = await initializeAgentMcpServers(...)

try {
    // 使用资源
} finally {
    // 确保清理
    await cleanup()
}
```

### 17.4 配置驱动 vs 代码驱动

```ts
// 配置驱动（frontmatter）
{
    "tools": ["Read", "Grep"],
    "model": "haiku"
}

// 代码驱动（built-in）
export const EXPLORE_AGENT = {
    agentType: 'Explore',
    disallowedTools: [...],
    model: 'haiku',
}
```

---

## 十八、Agent 系统的完整调用链

```
主 Agent 调用 AgentTool
    ↓
AgentTool.call() (AgentTool.tsx)
    ↓
┌─────────────────────────────────────┐
│ 1. 参数验证                          │
│ 2. 查找 agent definition             │
│ 3. 判断执行模式                      │
│    - sync vs async vs background     │
│    - local vs remote vs worktree     │
│ 4. 处理 isolation 设置              │
│ 5. 创建 background task（如果是）     │
└─────────────────────────────────────┘
    ↓
runAgent() (runAgent.ts)
    ↓
┌─────────────────────────────────────┐
│ 6. 创建 AgentId                     │
│ 7. 解析 model、tools、context        │
│ 8. 初始化专属 MCP servers            │
│ 9. 预加载 skills                     │
│ 10. 执行 SubagentStart hooks          │
│ 11. 创建 subagent context             │
└─────────────────────────────────────┘
    ↓
query() (query.ts) - 核心循环
    ↓
┌─────────────────────────────────────┐
│ for await (const message of query()) │
│   - yield 每个事件                    │
│   - 记录到 sidechain transcript       │
│   - 转发给父 agent                    │
└─────────────────────────────────────┘
    ↓
清理阶段（finally 块）
    ↓
结果返回父 agent
```

---

## 十九、未来演进方向

### 19.1 已规划的特性

- **多模态 agent**：处理图像、视频、音频
- **更智能的 agent 选择**：根据任务自动选择 agent
- **agent 间通信优化**：更高效的 Inbox 系统
- **可观察性增强**：更好的追踪和调试

### 19.2 当前限制

- **嵌套深度**：agent 不能无限制嵌套（递归 fork 防护）
- **资源消耗**：每个 agent 都有独立的内存开销
- **同步等待**：同步 agent 会阻塞主 agent

---

## 二十、核心要点总结

### 20.1 Claude Code 多 Agent 系统的关键洞察

1. **专业化分工**：每个 agent 有特定的工具、模型、权限
2. **上下文隔离**：每个 agent 有自己的消息历史和文件状态
3. **资源管理**：严格的 cleanup 防止内存泄漏
4. **性能优化**：通过 fork 共享 prompt cache，节省 token
5. **灵活执行**：同步、异步、后台、远程多种模式

### 20.2 关键设计决策

| 决策 | 理由 |
| --- | --- |
| 单一 model 标识 | 简化配置，子 agent 决定 model |
| 不共享 setAppState（异步） | 避免污染父状态 |
| 独立 abortController | 支持独立取消 |
| 专属 MCP servers | agent 可有独立工具集 |
| 子 transcript | 隔离存储，支持 resume |
| Fork 字节级一致 | 命中 prompt cache |
| ONE_SHOT 跳过 trailer | 节省 token |

### 20.3 关键文件速查

| 文件 | 行数 | 作用 |
| --- | --- | --- |
| `src/tools/AgentTool/AgentTool.tsx` | 1448 | AgentTool 主类 |
| `src/tools/AgentTool/runAgent.ts` | 976 | 子 agent 运行时 |
| `src/tools/AgentTool/forkSubagent.ts` | - | Fork 优化 |
| `src/tools/AgentTool/loadAgentsDir.ts` | - | Agent 加载 |
| `src/utils/forkedAgent.ts` | - | 子 agent 上下文 |
| `src/tasks/LocalAgentTask/` | - | 异步任务管理 |
| `src/tasks/RemoteAgentTask/` | - | 远程任务管理 |

---

## 二十一、给学习者的建议

### 21.1 入门路径

1. **先读 `runAgent.ts`**：理解子 agent 的完整生命周期
2. **读 `AgentTool.tsx`**：理解 AgentTool 主类
3. **读 `loadAgentsDir.ts`**：理解 agent 如何被加载
4. **读 `built-in/exploreAgent.ts`**：理解具体 agent 的实现
5. **最后读 `forkSubagent.ts`**：理解 Prompt Cache 优化

### 21.2 实践建议

- **实现一个简单的子 agent**：在测试环境中
- **观察 transcript 文件**：理解消息持久化
- **修改 agent 参数**：测试不同配置的差异
- **启用 fork feature**：观察 cache 命中

### 21.3 关键洞察

理解 Claude Code 的多 agent 系统需要把握以下关键点：

1. **agent 是有状态的**：有自己的消息、状态、配置
2. **资源严格管理**：每个资源都有清理路径
3. **性能是核心**：每个设计都考虑 token 和 cache
4. **灵活的执行模式**：同步、异步、后台、远程
5. **专业化分工**：每个 agent 做一件事做到最好

**掌握这些，你就能设计出工业级的多 agent 系统！** 🎯
