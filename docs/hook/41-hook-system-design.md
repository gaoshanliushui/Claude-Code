# Claude Code Hook 系统深度解析

> 本文深入分析 Claude Code 的 Hook 系统设计与实现。
> 涉及文件：`src/utils/hooks/*`、`src/entrypoints/agentSdkTypes.js`

---

## 一、Hook 系统整体定位

### 1.1 什么是 Hook？

Hook 是 Claude Code 在**特定生命周期事件**发生时被触发的**可定制回调**。它允许用户：

```
1. 在工具执行前后插入自定义逻辑
2. 在会话开始/结束时执行特定操作
3. 修改工具调用的输入或阻止执行
4. 发送通知、记录日志、集成外部系统
```

### 1.2 Hook 系统的核心价值

| 价值 | 说明 |
| --- | --- |
| **可扩展性** | 不修改核心代码即可扩展功能 |
| **可观测性** | 监听所有关键事件 |
| **可定制性** | 用户可定义自己的行为 |
| **可集成性** | 连接外部系统（CI/CD、监控、审计） |
| **可保护性** | 安全审计、合规检查 |

### 1.3 Hook 系统的目录结构

```
src/utils/hooks/
├── hookEvents.ts              # Hook 事件系统（订阅-发布）
├── hookHelpers.ts             # Hook 通用辅助函数
├── hooksSettings.ts           # Hook 配置类型
├── hooksConfigManager.ts      # Hook 配置管理
├── hooksConfigSnapshot.ts     # Hook 配置快照
├── sessionHooks.ts            # 会话级 Hook 管理
├── registerFrontmatterHooks.ts # Frontmatter Hook 注册
├── registerSkillHooks.ts      # 技能 Hook 注册
├── execPromptHook.ts          # Prompt Hook 执行器
├── execHttpHook.ts            # HTTP Hook 执行器
├── execAgentHook.ts           # Agent Hook 执行器
├── apiQueryHookHelper.ts      # API Query Hook 辅助
├── postSamplingHooks.ts       # 后采样 Hook
├── AsyncHookRegistry.ts       # 异步 Hook 注册表
├── fileChangedWatcher.ts      # 文件变化监听
├── skillImprovement.ts        # 技能改进
└── ssrfGuard.ts               # SSRF 防护
```

---

## 二、Hook 事件类型

### 2.1 HookEvent 完整列表

```ts
// src/entrypoints/agentSdkTypes.js
export const HOOK_EVENTS = [
    // 会话生命周期
    'SessionStart',      // 会话开始
    'SessionEnd',        // 会话结束
    'Setup',             // 初始化设置
    'SetupFailure',      // 设置失败
    
    // 用户输入
    'UserPromptSubmit',  // 用户提交 prompt
    'PreCompact',        // 压缩前
    'PostCompact',       // 压缩后
    
    // 工具调用
    'PreToolUse',        // 工具调用前
    'PostToolUse',       // 工具调用后
    'PostToolUseFailure',// 工具调用失败后
    'PermissionRequest', // 权限请求
    'PermissionDenied',  // 权限拒绝
    
    // 子 agent
    'SubagentStart',     // 子 agent 开始
    'SubagentStop',      // 子 agent 结束
    
    // 通知
    'Notification',      // 通知事件
    
    // 停止
    'Stop',              // 停止（主 agent）
    'StopFailure',       // 停止失败
    
    // 任务
    'TaskCreated',       // 任务创建
    'TaskCompleted',     // 任务完成
    
    // 用户
    'UserPromptSubmit',  // 用户 prompt 提交
    
    // 引擎
    'PreResponse',       // 响应前
    'PostResponse',      // 响应后
]
```

### 2.2 事件分类

```
┌─────────────────────────────────────────────────────┐
│  会话生命周期                                         │
│  - SessionStart / SessionEnd                          │
│  - Setup / SetupFailure                               │
├─────────────────────────────────────────────────────┤
│  用户交互                                             │
│  - UserPromptSubmit                                   │
│  - Notification                                       │
├─────────────────────────────────────────────────────┤
│  工具调用                                             │
│  - PreToolUse / PostToolUse / PostToolUseFailure     │
│  - PermissionRequest / PermissionDenied               │
├─────────────────────────────────────────────────────┤
│  Agent 管理                                            │
│  - SubagentStart / SubagentStop                       │
│  - TaskCreated / TaskCompleted                        │
├─────────────────────────────────────────────────────┤
│  压缩管理                                             │
│  - PreCompact / PostCompact                           │
├─────────────────────────────────────────────────────┤
│  生命周期控制                                         │
│  - Stop / StopFailure                                 │
└─────────────────────────────────────────────────────┘
```

### 2.3 事件类型定义

```ts
// src/entrypoints/agentSdkTypes.js
export type HookEvent =
    | 'PreToolUse'
    | 'PostToolUse'
    | 'PostToolUseFailure'
    | 'Notification'
    | 'UserPromptSubmit'
    | 'SessionStart'
    | 'SessionEnd'
    | 'Stop'
    | 'StopFailure'
    | 'SubagentStart'
    | 'SubagentStop'
    | 'PreCompact'
    | 'PostCompact'
    | 'PermissionRequest'
    | 'PermissionDenied'
    | 'Setup'
    | 'SetupFailure'
    | 'TaskCreated'
    | 'TaskCompleted'
    | 'PreResponse'
    | 'PostResponse'
```

---

## 三、Hook 类型

### 3.1 四种 Hook 类型

```ts
// src/utils/settings/types.ts
export type HookCommand = {
    type: 'command'   // Shell 命令
    command: string
    timeout?: number
}

export type PromptHook = {
    type: 'prompt'    // Prompt-based hook（返回 JSON 决策）
    prompt: string
    model?: string
}

export type HttpHook = {
    type: 'http'      // HTTP webhook
    url: string
    headers?: Record<string, string>
    method?: 'POST' | 'PUT'
}

export type FunctionHook = {
    type: 'function'  // 内存中的函数回调
    callback: FunctionHookCallback
    timeout?: number
    errorMessage: string
}
```

### 3.2 各类型 Hook 详解

#### (1) Command Hook

```json
{
    "type": "command",
    "command": "echo 'Tool was called' >> /tmp/audit.log",
    "timeout": 5000
}
```

**特点**：
- 执行 Shell 命令
- 通过 stdin/stdout 与主进程通信
- 适合集成外部工具

#### (2) Prompt Hook

```json
{
    "type": "prompt",
    "prompt": "Is this command safe?",
    "model": "haiku"
}
```

**特点**：
- 用 LLM 评估上下文
- 返回 JSON 决策
- 适合复杂场景判断

#### (3) HTTP Hook

```json
{
    "type": "http",
    "url": "https://api.example.com/audit",
    "headers": {
        "Authorization": "Bearer xxx"
    }
}
```

**特点**：
- 发送 HTTP 请求
- 适合集成外部服务
- 支持 SSRF 防护

#### (4) Function Hook

```ts
addFunctionHook(
    setAppState, sessionId, 'PreToolUse',
    async (messages, signal) => {
        // 自定义验证逻辑
        return true  // 通过
    },
    'Custom validation failed'
)
```

**特点**：
- 内存中的函数回调
- 仅会话级
- 性能最高（无 I/O）

---

## 四、Hook 执行器（execXxxHook）

### 4.1 execPromptHook

```ts
// src/utils/hooks/execPromptHook.ts
export async function execPromptHook(
    hook: PromptHook,
    context: HookContext,
): Promise<HookResult> {
    // 1. 构造 prompt
    const prompt = buildPrompt(context)
    
    // 2. 调用 LLM
    const response = await queryModel(prompt, hook.model)
    
    // 3. 解析决策
    return parseHookDecision(response)
}
```

### 4.2 execHttpHook

```ts
// src/utils/hooks/execHttpHook.ts
export async function execHttpHook(
    hook: HttpHook,
    context: HookContext,
): Promise<HookResult> {
    // 1. SSRF 防护
    await validateUrl(hook.url)
    
    // 2. 发送请求
    const response = await fetch(hook.url, {
        method: hook.method ?? 'POST',
        headers: hook.headers,
        body: JSON.stringify(context),
    })
    
    // 3. 解析响应
    return parseHookResponse(response)
}
```

### 4.3 execAgentHook

```ts
// src/utils/hooks/execAgentHook.ts
export async function execAgentHook(
    hook: AgentHookConfig,
    context: HookContext,
): Promise<HookResult> {
    // 启动一个专门的 agent 来处理 hook
    const result = await runForkedAgent({
        prompt: hook.prompt,
        // ...
    })
    
    return parseAgentResponse(result)
}
```

### 4.4 Prompt-based Hook 的返回格式

```json
{
    "decision": "approve" | "block" | "modify",
    "reason": "Explanation of decision",
    "updatedInput": { /* modified tool input */ }
}
```

---

## 五、Hook 事件系统（hookEvents.ts）

### 5.1 核心设计：订阅-发布模式

```ts
// src/utils/hooks/hookEvents.ts
const pendingEvents: HookExecutionEvent[] = []
let eventHandler: HookEventHandler | null = null
let allHookEventsEnabled = false

export function registerHookEventHandler(
    handler: HookEventHandler | null,
): void {
    eventHandler = handler
    if (handler && pendingEvents.length > 0) {
        // 重放之前累积的事件
        for (const event of pendingEvents.splice(0)) {
            handler(event)
        }
    }
}
```

### 5.2 三种事件类型

```ts
export type HookStartedEvent = {
    type: 'started'
    hookId: string
    hookName: string
    hookEvent: string
}

export type HookProgressEvent = {
    type: 'progress'
    hookId: string
    hookName: string
    hookEvent: string
    stdout: string
    stderr: string
    output: string
}

export type HookResponseEvent = {
    type: 'response'
    hookId: string
    hookName: string
    hookEvent: string
    output: string
    stdout: string
    stderr: string
    exitCode?: number
    outcome: 'success' | 'error' | 'cancelled'
}
```

### 5.3 事件发射

```ts
export function emitHookStarted(
    hookId: string,
    hookName: string,
    hookEvent: string,
): void {
    if (!shouldEmit(hookEvent)) return
    emit({type: 'started', hookId, hookName, hookEvent})
}

export function emitHookResponse(data: {...}): void {
    // Always log full hook output to debug log for verbose mode debugging
    const outputToLog = data.stdout || data.stderr || data.output
    if (outputToLog) {
        logForDebugging(`Hook ${data.hookName} (${data.hookEvent}) ${data.outcome}:\n${outputToLog}`)
    }
    
    if (!shouldEmit(data.hookEvent)) return
    emit({type: 'response', ...data})
}
```

### 5.4 ALWAYS_EMITTED_HOOK_EVENTS

```ts
const ALWAYS_EMITTED_HOOK_EVENTS = ['SessionStart', 'Setup']

function shouldEmit(hookEvent: string): boolean {
    if (ALWAYS_EMITTED_HOOK_EVENTS.includes(hookEvent)) return true
    return allHookEventsEnabled && HOOK_EVENTS.includes(hookEvent)
}
```

**设计意图**：
- SessionStart 和 Setup 总是发出（向后兼容）
- 其他事件需要通过 `includeHookEvents` 选项启用

### 5.5 startHookProgressInterval

```ts
export function startHookProgressInterval(params: {
    hookId: string
    hookName: string
    hookEvent: string
    getOutput: () => Promise<{ stdout: string; stderr: string; output: string }>
    intervalMs?: number
}): () => void {
    let lastEmittedOutput = ''
    const interval = setInterval(async () => {
        const {stdout, stderr, output} = await params.getOutput()
        if (output === lastEmittedOutput) return  // 去重
        lastEmittedOutput = output
        emitHookProgress({...})
    }, params.intervalMs ?? 1000)
    interval.unref()
    
    return () => clearInterval(interval)
}
```

**功能**：定时 emit 进度事件，让 SDK 客户端能实时显示 hook 执行进度。

### 5.6 挂起的事件缓冲

```ts
const MAX_PENDING_EVENTS = 100

function emit(event: HookExecutionEvent): void {
    if (eventHandler) {
        eventHandler(event)
    } else {
        pendingEvents.push(event)
        if (pendingEvents.length > MAX_PENDING_EVENTS) {
            pendingEvents.shift()  // 限制缓冲大小
        }
    }
}
```

**设计意图**：在 SDK 客户端注册前累积事件，避免丢失。

---

## 六、Hook 配置（hooksSettings.ts）

### 6.1 HookMatcher 类型

```ts
// src/utils/settings/types.ts
export type HookMatcher = {
    matcher: string  // 匹配条件
    hooks: HookCommand[]  // 该匹配条件下的 hook 列表
    skillRoot?: string  // 可选：技能的根目录（来自 frontmatter）
}
```

### 6.2 matcher 的语法

```
"*"                       // 匹配所有
"Write|Edit"               // 匹配 Write 或 Edit 工具
"Bash(npm:*)"              // 匹配特定子命令
"mcp__server1__*"          // 匹配特定 MCP 服务器的所有工具
"/path/to/file"            // 匹配特定文件
```

### 6.3 完整 Hooks 配置示例

```json
{
    "hooks": {
        "PreToolUse": [
            {
                "matcher": "Write|Edit",
                "hooks": [
                    {
                        "type": "command",
                        "command": "/usr/local/bin/lint-check.sh",
                        "timeout": 10000
                    }
                ]
            },
            {
                "matcher": "Bash",
                "hooks": [
                    {
                        "type": "prompt",
                        "prompt": "Is this Bash command safe to run? Consider the context: $ARGUMENTS",
                        "model": "haiku"
                    }
                ]
            }
        ],
        "PostToolUse": [
            {
                "matcher": "*",
                "hooks": [
                    {
                        "type": "command",
                        "command": "jq -r '.tool_input' >> /tmp/tool-audit.jsonl"
                    }
                ]
            }
        ],
        "SessionStart": [
            {
                "matcher": "*",
                "hooks": [
                    {
                        "type": "command",
                        "command": "echo 'Session started at $(date)' >> ~/.claude/session.log"
                    }
                ]
            }
        ]
    }
}
```

---

## 七、Hook 注册系统

### 7.1 三个层级

```ts
// 1. 全局 settings（持久化到 settings.json）
{
    hooks: { ... }  // HookMatcher[]
}

// 2. 会话级 Hooks（内存中）
sessionHooks: SessionHooksState  // Map<sessionId, SessionStore>

// 3. 前置元数据 Hooks（来自 agent frontmatter）
// 通过 registerFrontmatterHooks 注册
```

### 7.2 sessionHooks Map 的设计精妙

```ts
/**
 * Map (not Record) so .set/.delete don't change the container's identity.
 * Mutator functions mutate the Map and return prev unchanged, letting
 * store.ts's Object.is(next, prev) check short-circuit and skip listener
 * notification.
 */
export type SessionHooksState = Map<string, SessionStore>
```

**关键洞察**：
- 用 Map 而非 Record
- 因为 Record 的展开会创建新对象，导致 store 触发通知
- Map 的 .set() 不改变容器身份，可被 `Object.is` 短路优化

注释解释：
> This matters under high-concurrency workflows: parallel() with N schema-mode agents fires N addFunctionHook calls in one synchronous tick. With a Record + spread, each call cost O(N) to copy the growing map (O(N²) total) plus fired all ~30 store listeners. With Map: .set() is O(1), return prev means zero listener fires.

### 7.3 addSessionHook

```ts
export function addSessionHook(
    setAppState: (updater: (prev: AppState) => AppState) => void,
    sessionId: string,
    event: HookEvent,
    matcher: string,
    hook: HookCommand,
    onHookSuccess?: OnHookSuccess,
    skillRoot?: string,
): void {
    addHookToSession(setAppState, sessionId, event, matcher, hook, onHookSuccess, skillRoot)
}
```

### 7.4 addFunctionHook

```ts
export function addFunctionHook(
    setAppState: (updater: (prev: AppState) => AppState) => void,
    sessionId: string,
    event: HookEvent,
    matcher: string,
    callback: FunctionHookCallback,
    errorMessage: string,
    options?: { timeout?: number; id?: string },
): string {
    const id = options?.id || `function-hook-${Date.now()}-${Math.random()}`
    const hook: FunctionHook = {
        type: 'function',
        id,
        timeout: options?.timeout || 5000,
        callback,
        errorMessage,
    }
    addHookToSession(setAppState, sessionId, event, matcher, hook)
    return id
}
```

### 7.5 removeFunctionHook

```ts
export function removeFunctionHook(
    setAppState: (updater: (prev: AppState) => AppState) => void,
    sessionId: string,
    event: HookEvent,
    hookId: string,
): void {
    // 从 sessionHooks 中移除指定 ID 的 hook
    setAppState(prev => {
        const store = prev.sessionHooks.get(sessionId)
        if (!store) return prev
        
        const eventMatchers = store.hooks[event] || []
        const updatedMatchers = eventMatchers
            .map(matcher => {
                const updatedHooks = matcher.hooks.filter(h => {
                    if (h.hook.type !== 'function') return true
                    return h.hook.id !== hookId
                })
                return updatedHooks.length > 0 ? { ...matcher, hooks: updatedHooks } : null
            })
            .filter((m): m is SessionHookMatcher => m !== null)
        
        // 更新 store
        const newHooks = updatedMatchers.length > 0
            ? { ...store.hooks, [event]: updatedMatchers }
            : Object.fromEntries(Object.entries(store.hooks).filter(([e]) => e !== event))
        
        prev.sessionHooks.set(sessionId, {hooks: newHooks})
        return prev
    })
}
```

### 7.6 SessionStore 类型

```ts
export type SessionStore = {
    hooks: {
        [event in HookEvent]?: SessionHookMatcher[]
    }
}

export type SessionHooksState = Map<string, SessionStore>

type SessionHookMatcher = {
    matcher: string
    skillRoot?: string
    hooks: Array<{
        hook: HookCommand | FunctionHook
        onHookSuccess?: OnHookSuccess
    }>
}
```

---

## 八、registerFrontmatterHooks（前置元数据 Hook）

### 8.1 概念

Agent 的 markdown frontmatter 中可以定义 hooks：

```markdown
---
name: code-reviewer
hooks:
  PreToolUse:
    - matcher: "Write|Edit"
      hooks:
        - type: command
          command: "review-changes.sh"
---
```

### 8.2 registerFrontmatterHooks

```ts
// src/utils/hooks/registerFrontmatterHooks.ts
export function registerFrontmatterHooks(
    setAppState: (f: (prev: AppState) => AppState) => void,
    agentId: AgentId,
    hooksSettings: HooksSettings,
    sourceLabel: string,
    isAgent: boolean = false,
): void {
    // 遍历 hooksSettings 注册
    for (const [event, matchers] of Object.entries(hooksSettings)) {
        for (const matcher of matchers) {
            // 注册到 sessionHooks
            for (const hook of matcher.hooks) {
                addSessionHook(
                    setAppState, agentId, event,
                    matcher.matcher, hook, undefined,
                    matcher.skillRoot,
                )
            }
        }
    }
}
```

### 8.3 clearSessionHooks

```ts
// 在 agent 结束时清理
export function clearSessionHooks(
    setAppState: (f: (prev: AppState) => AppState) => void,
    agentId: AgentId,
): void {
    setAppState(prev => {
        prev.sessionHooks.delete(agentId)
        return prev
    })
}
```

### 8.4 Agent 的 Stop → SubagentStop 转换

```ts
// src/tools/AgentTool/runAgent.ts
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

**关键设计**：子 agent 触发 `SubagentStop` 而非 `Stop`。

---

## 九、Hook 执行流程

### 9.1 通用执行流程

```ts
async function executeHook(
    hook: HookCommand | FunctionHook,
    context: HookContext,
): Promise<HookResult> {
    const startTime = Date.now()
    emitHookStarted(hook.id, hook.name, context.event)
    
    // 启动进度监控
    const stopProgress = startHookProgressInterval({...})
    
    try {
        let result: HookResult
        
        // 根据 hook 类型分发
        switch (hook.type) {
            case 'command':
                result = await execCommandHook(hook, context)
                break
            case 'prompt':
                result = await execPromptHook(hook, context)
                break
            case 'http':
                result = await execHttpHook(hook, context)
                break
            case 'function':
                result = await execFunctionHook(hook, context)
                break
        }
        
        emitHookResponse({
            hookId: hook.id,
            hookName: hook.name,
            hookEvent: context.event,
            ...result,
            outcome: 'success',
        })
        
        return result
    } catch (error) {
        emitHookResponse({
            hookId: hook.id,
            hookName: hook.name,
            hookEvent: context.event,
            outcome: 'error',
            ...
        })
        throw error
    } finally {
        stopProgress()
    }
}
```

### 9.2 HookResult 类型

```ts
export type HookResult = {
    decision: 'approve' | 'block' | undefined  // approve=继续, block=阻止
    reason?: string                            // 决策原因
    updatedInput?: object                      // 修改后的输入
    additionalContexts?: string[]              // 注入到上下文的额外信息
    hookSpecificOutput?: Record<string, unknown>
}
```

---

## 十、PostCompactHook 与 PreCompactHook

### 10.1 PreCompact Hook

```ts
// src/utils/hooks.ts
export async function executePreCompactHooks(
    params: { trigger: 'auto' | 'manual'; customInstructions: string | null },
    signal: AbortSignal,
): Promise<AggregatedHookResult>
```

**触发时机**：在 `compactConversation` 开始时
**用途**：让用户自定义压缩行为、注入额外上下文

### 10.2 PostCompact Hook

```ts
export async function executePostCompactHooks(
    params: { trigger: 'auto' | 'manual'; compactSummary: string },
    signal: AbortSignal,
): Promise<AggregatedHookResult>
```

**触发时机**：在压缩完成后
**用途**：通知、记录、清理

### 10.3 AggregatedHookResult

```ts
export type AggregatedHookResult = {
    userDisplayMessage?: string        // 用户可见消息
    newCustomInstructions?: string    // 合并后的指令
    // ...
}
```

---

## 十一、PreToolUse 与 PostToolUse Hooks

### 11.1 PreToolUse Hook

```ts
// 在权限检查之前或之后触发
async function executePreToolUseHook(
    toolName: string,
    input: unknown,
    context: ToolUseContext,
): Promise<PermissionDecision> {
    const hooks = getHooksForEvent('PreToolUse', toolName)
    
    for (const hook of hooks) {
        const result = await executeHook(hook, {
            event: 'PreToolUse',
            toolName,
            toolInput: input,
            // ...
        })
        
        if (result.decision === 'block') {
            return {
                behavior: 'deny',
                message: result.reason,
                decisionReason: {type: 'hook', hookName: hook.name, reason: result.reason},
            }
        }
        
        if (result.updatedInput) {
            // 修改输入
            input = result.updatedInput
        }
    }
    
    return {behavior: 'passthrough'}
}
```

### 11.2 PreToolUse 决策

| 决策 | 含义 |
| --- | --- |
| `approve` | 允许工具执行 |
| `block` | 阻止工具执行 |
| `modify` | 修改输入后执行 |

### 11.3 PostToolUse Hook

```ts
// 在工具执行后触发
async function executePostToolUseHook(
    toolName: string,
    input: unknown,
    result: unknown,
    context: ToolUseContext,
): Promise<void> {
    const hooks = getHooksForEvent('PostToolUse', toolName)
    
    for (const hook of hooks) {
        await executeHook(hook, {
            event: 'PostToolUse',
            toolName,
            toolInput: input,
            toolResult: result,
            // ...
        })
    }
}
```

### 11.4 PostToolUseFailure Hook

```ts
// 在工具执行失败后触发
async function executePostToolUseFailureHook(
    toolName: string,
    input: unknown,
    error: Error,
    context: ToolUseContext,
): Promise<void> {
    // 类似 PostToolUse，但传入 error
}
```

---

## 十二、Subagent Hooks

### 12.1 SubagentStart Hook

```ts
// src/tools/AgentTool/runAgent.ts
for await (const hookResult of executeSubagentStartHooks(
    agentId,
    agentDefinition.agentType,
    agentAbortController.signal,
)) {
    if (hookResult.additionalContexts?.length > 0) {
        additionalContexts.push(...hookResult.additionalContexts)
    }
}
```

**触发时机**：子 agent 开始时
**用途**：注入额外上下文、记录审计

### 12.2 SubagentStop Hook

```ts
// 在子 agent 结束时
async function executeSubagentStopHooks(
    agentId: AgentId,
    agentType: string,
    signal: AbortSignal,
): Promise<void>
```

### 12.3 Hook 在 agent 中的应用

```ts
// SubagentStart hook 注入的上下文作为 user message
if (additionalContexts.length > 0) {
    const contextMessage = createAttachmentMessage({
        type: 'hook_additional_context',
        content: additionalContexts,
        hookName: 'SubagentStart',
        toolUseID: randomUUID(),
        hookEvent: 'SubagentStart',
    })
    initialMessages.push(contextMessage)
}
```

---

## 十三、AsyncHookRegistry 异步 Hook 注册表

### 13.1 概念

某些 hook 是**异步启动**的（如远程服务调用），需要在完成后回调。

```ts
// src/utils/hooks/AsyncHookRegistry.ts
type AsyncHook = {
    hookId: string
    promise: Promise<unknown>
    resolve: (result: unknown) => void
    reject: (error: Error) => void
}
```

### 13.2 finalizePendingAsyncHooks

```ts
// 在会话结束时清理
export async function finalizePendingAsyncHooks(): Promise<void> {
    const pending = getPendingAsyncHooks()
    await Promise.allSettled(pending.map(h => h.promise))
}
```

**使用场景**：在会话关闭时等待所有异步 hook 完成。

---

## 十四、fileChangedWatcher 文件变化监听

### 14.1 概念

某些 Hook 需要在文件变化时触发（如 lint 工具）。

```ts
// src/utils/hooks/fileChangedWatcher.ts
export class FileChangedWatcher {
    watch(path: string, callback: () => void): void
    unwatch(path: string): void
}
```

### 14.2 与 PostToolUse 配合

```ts
// 监听 Write/Edit 工具修改的文件
fileChangedWatcher.on('file-changed', (path) => {
    // 触发 PostToolUse hook
})
```

---

## 十五、SSRF Guard 安全防护

### 15.1 概念

HTTP Hook 可能被用于 SSRF（Server-Side Request Forgery）攻击，需要防护。

```ts
// src/utils/hooks/ssrfGuard.ts
export async function validateHookUrl(url: string): Promise<void> {
    const parsed = new URL(url)
    
    // 禁止内网地址
    if (isPrivateIP(parsed.hostname)) {
        throw new Error(`Hook URL ${url} points to a private IP address`)
    }
    
    // 禁止 localhost
    if (parsed.hostname === 'localhost') {
        throw new Error(`Hook URL ${url} uses localhost`)
    }
}
```

### 15.2 防护规则

```ts
const BLOCKED_HOSTNAMES = [
    'localhost',
    '127.0.0.1',
    '0.0.0.0',
    '::1',
    // 内网 IP 范围
    '10.0.0.0/8',
    '172.16.0.0/12',
    '192.168.0.0/16',
    '169.254.0.0/16',  // link-local
]
```

---

## 十六、hooksConfigManager 配置管理

### 16.1 配置加载

```ts
// src/utils/hooks/hooksConfigManager.ts
export function loadHooksConfig(): HooksSettings {
    return readSettingsJson().hooks || {}
}
```

### 16.2 配置合并

```ts
export function mergeHooksConfigs(
    ...configs: HooksSettings[]
): HooksSettings {
    const merged: HooksSettings = {}
    
    for (const config of configs) {
        for (const [event, matchers] of Object.entries(config)) {
            if (!merged[event]) merged[event] = []
            merged[event].push(...matchers)
        }
    }
    
    return merged
}
```

### 16.3 hooksConfigSnapshot

```ts
// 创建配置的不可变快照
export function createHooksConfigSnapshot(
    config: HooksSettings,
): HooksConfigSnapshot
```

**用途**：避免在 hook 执行期间配置被修改导致不一致。

---

## 十七、executeUserPromptSubmitHooks

### 17.1 UserPromptSubmit Hook

```ts
export async function executeUserPromptSubmitHooks(
    prompt: string,
    context: ToolUseContext,
): Promise<AggregatedHookResult> {
    const hooks = getHooksForEvent('UserPromptSubmit')
    
    let processedPrompt = prompt
    
    for (const hook of hooks) {
        const result = await executeHook(hook, {
            event: 'UserPromptSubmit',
            userPrompt: processedPrompt,
            // ...
        })
        
        // Hook 可以修改 prompt
        if (result.updatedPrompt) {
            processedPrompt = result.updatedPrompt
        }
        
        // 可以阻止 prompt
        if (result.decision === 'block') {
            throw new Error('User prompt blocked by hook')
        }
    }
    
    return {finalPrompt: processedPrompt}
}
```

### 17.2 使用场景

```json
{
    "hooks": {
        "UserPromptSubmit": [
            {
                "matcher": "*",
                "hooks": [
                    {
                        "type": "prompt",
                        "prompt": "Check if this user prompt contains any PII or sensitive information. If yes, return decision: block. Otherwise, return decision: approve."
                    }
                ]
            }
        ]
    }
}
```

---

## 十八、executeSessionStartHooks

### 18.1 SessionStart Hook

```ts
// src/utils/sessionStart.ts
export async function processSessionStartHooks(
    trigger: 'init' | 'maintenance' | 'compact',
    options: { model: string },
): Promise<Message[]> {
    const hooks = getHooksForEvent('SessionStart')
    
    const hookMessages: Message[] = []
    
    for (const hook of hooks) {
        const result = await executeHook(hook, {
            event: 'SessionStart',
            trigger,
            // ...
        })
        
        if (result.additionalContexts?.length > 0) {
            // 添加到模型上下文
            for (const ctx of result.additionalContexts) {
                hookMessages.push(createAttachmentMessage({
                    type: 'hook_additional_context',
                    content: [ctx],
                    hookName: hook.name,
                    toolUseID: randomUUID(),
                    hookEvent: 'SessionStart',
                }))
            }
        }
    }
    
    return hookMessages
}
```

### 18.2 SessionStart 的三个触发点

```ts
// 1. 会话初始化
processSessionStartHooks('init', {model: 'claude-opus-4-8'})

// 2. 维护操作（如 reconnect）
processSessionStartHooks('maintenance', {model: 'claude-opus-4-8'})

// 3. 压缩后
processSessionStartHooks('compact', {model: 'claude-opus-4-8'})
```

---

## 十九、Notification Hook

### 19.1 概念

发送通知（如系统通知、Slack 消息）给用户。

```ts
export async function executeNotificationHooks(
    params: {
        message: string
        notificationType: string  // 'info' | 'warning' | 'error' | 'permission_prompt'
    },
    context: ToolUseContext,
): Promise<void> {
    const hooks = getHooksForEvent('Notification')
    
    for (const hook of hooks) {
        await executeHook(hook, {
            event: 'Notification',
            message: params.message,
            type: params.notificationType,
        })
    }
}
```

### 19.2 使用场景

```json
{
    "hooks": {
        "Notification": [
            {
                "matcher": "*",
                "hooks": [
                    {
                        "type": "command",
                        "command": "osascript -e 'display notification \"$MESSAGE\" with title \"Claude Code\"'"
                    }
                ]
            }
        ]
    }
}
```

---

## 二十、Task Hooks（任务事件）

### 20.1 TaskCreated / TaskCompleted

```ts
// 在 async agent 完成时触发
async function notifyTaskCompleted(task: TaskState) {
    await executeNotificationHooks({
        message: `Task ${task.description} completed`,
        notificationType: 'task_completed',
    }, context)
}
```

### 20.2 自动后台任务的 Hook 集成

```ts
// LocalAgentTask.ts 中
if (task.type === 'local_agent' && task.status === 'completed') {
    await executeTaskCompletedHooks(task)
}
```

---

## 二十一、Stop Hook

### 21.1 Stop Hook 触发

```ts
// 主 agent 完成一轮对话后
async function onStop(context: ToolUseContext) {
    const hooks = getHooksForEvent('Stop')
    
    for (const hook of hooks) {
        const result = await executeHook(hook, {
            event: 'Stop',
            sessionId: context.sessionId,
            // ...
        })
        
        if (result.decision === 'block') {
            // Hook 阻止停止，让 agent 继续
            throw new Error('Stop blocked by hook')
        }
    }
}
```

### 21.2 Stop Hook 的高级用法

```json
{
    "hooks": {
        "Stop": [
            {
                "matcher": "*",
                "hooks": [
                    {
                        "type": "prompt",
                        "prompt": "Review the agent's response. If it didn't complete the user's request, return decision: block. Otherwise, return decision: approve."
                    }
                ]
            }
        ]
    }
}
```

**设计意图**：让 LLM 评估 agent 的响应质量，如果不满意就阻止停止。

---

## 二十二、registerSkillHooks

### 22.1 概念

技能（Skill）也可以定义 hooks（来自 SKILL.md frontmatter）。

```ts
// src/utils/hooks/registerSkillHooks.ts
export function registerSkillHooks(
    setAppState: (f: (prev: AppState) => AppState) => void,
    skillName: string,
    hooksSettings: HooksSettings,
): void {
    for (const [event, matchers] of Object.entries(hooksSettings)) {
        for (const matcher of matchers) {
            for (const hook of matcher.hooks) {
                addSessionHook(
                    setAppState, `skill:${skillName}`,
                    event as HookEvent,
                    matcher.matcher, hook, undefined,
                    skillName,  // skillRoot
                )
            }
        }
    }
}
```

### 22.2 技能 hook 的生命周期

```
技能加载 → registerSkillHooks → hook 激活
技能卸载 → clearSessionHooks('skill:${name}') → hook 清理
```

---

## 二十三、postSamplingHooks 后采样

### 23.1 概念

后采样 hook 在模型响应后、返回给用户之前触发。

```ts
// src/utils/hooks/postSamplingHooks.ts
export async function executePostSamplingHooks(
    response: AssistantMessage,
    context: ToolUseContext,
): Promise<AssistantMessage> {
    const hooks = getHooksForEvent('PostResponse')
    
    let processed = response
    
    for (const hook of hooks) {
        const result = await executeHook(hook, {
            event: 'PostResponse',
            response: processed,
        })
        
        if (result.updatedResponse) {
            processed = result.updatedResponse
        }
    }
    
    return processed
}
```

### 23.2 使用场景

- 内容审查（filter 敏感信息）
- 自动格式化
- 添加额外注释

---

## 二十四、apiQueryHookHelper

### 24.1 概念

API Query Hook 是 SDK 用户的钩子调用辅助。

```ts
// src/utils/hooks/apiQueryHookHelper.ts
export async function executeApiQueryHook(
    request: HookRequest,
    context: ToolUseContext,
): Promise<HookResponse> {
    // 序列化
    const serialized = serializeForHook(request)
    
    // 发送给 SDK 客户端
    const response = await apiClient.sendHook(serialized)
    
    // 反序列化
    return deserializeHookResponse(response)
}
```

---

## 二十五、错误处理

### 25.1 Hook 超时

```ts
async function executeWithTimeout<T>(
    fn: () => Promise<T>,
    timeoutMs: number,
): Promise<T> {
    return Promise.race([
        fn(),
        new Promise<T>((_, reject) => {
            setTimeout(() => reject(new Error(`Hook timeout after ${timeoutMs}ms`)), timeoutMs)
        }),
    ])
}
```

### 25.2 Hook 失败的处理

```ts
try {
    result = await executeHook(hook, context)
} catch (error) {
    // 默认行为：记录错误但继续执行
    logError(`Hook ${hook.name} failed: ${error}`)
    
    // 某些事件允许失败继续（如 PostToolUse）
    // 某些事件必须成功（如 PreToolUse 阻塞）
    if (isBlockingEvent(event)) {
        throw error
    }
}
```

### 25.3 Hook 的熔断

```ts
// 失败的 hook 计数
if (failureCount > MAX_FAILURES) {
    // 暂时禁用该 hook
    skipHook = true
}
```

---

## 二十六、可观测性

### 26.1 Hook 执行日志

```ts
logForDebugging(`Hook ${hook.name} (${event}) executed`, {
    duration: Date.now() - startTime,
    decision: result.decision,
})
```

### 26.2 Hook 事件统计

```ts
logEvent('tengu_hook_executed', {
    hookEvent: event,
    hookName: hook.name,
    hookType: hook.type,
    duration: Date.now() - startTime,
    outcome: result.outcome,
})
```

### 26.3 调试输出

```ts
// stream-json 模式下的 hook 输出
if (options.outputFormat === 'stream-json' && options.verbose) {
    registerHookEventHandler(event => {
        structuredIO.write({
            type: 'system',
            subtype: `hook_${event.type}`,
            ...event,
        })
    })
}
```

---

## 二十七、设计要点总结

### 27.1 Hook 系统的关键设计

| 设计 | 理由 |
| --- | --- |
| **事件订阅-发布** | 解耦 hook 触发和消费 |
| **四类 Hook 类型** | 覆盖不同场景（命令、Prompt、HTTP、函数） |
| **Map 而非 Record** | 性能优化（避免 store 通知） |
| **始终发出 SessionStart/Setup** | 向后兼容 |
| **进度事件缓冲** | 不丢失早期事件 |
| **SSRF 防护** | 安全考量 |
| **AgentID 隔离** | 子 agent hook 不污染父会话 |

### 27.2 关键设计决策

```ts
// 1. 多种 Hook 类型应对不同场景
//    - command: 集成外部工具
//    - prompt: AI 辅助决策
//    - http: 远程服务
//    - function: 内存回调

// 2. 性能优化
//    - Map 而非 Record（避免 O(N²) 复制）
//    - 事件缓冲（避免丢失）
//    - 去重进度输出

// 3. 安全考虑
//    - SSRF 防护
//    - 超时控制
//    - 失败不影响主流程（除非阻塞事件）

// 4. 可观测性
//    - 详细日志
//    - 性能统计
//    - 事件流（供 SDK 消费）
```

### 27.3 Hook 系统的架构哲学

```ts
// Hook 系统是 Claude Code 的"神经系统"
// 它在关键事件点连接外部世界
// 同时保持核心逻辑的解耦和可测试性

// 设计原则：
// 1. 关注点分离（Hook 不影响主流程）
// 2. 类型安全（多种 Hook 类型各有 schema）
// 3. 性能优先（Map、缓冲、去重）
// 4. 用户友好（清晰的错误信息）
// 5. 可扩展（易于添加新事件类型）
```

---

## 二十八、关键文件速查

| 文件 | 作用 |
| --- | --- |
| `src/utils/hooks/hookEvents.ts` | 事件系统（订阅-发布） |
| `src/utils/hooks/sessionHooks.ts` | 会话级 Hook 管理 |
| `src/utils/hooks/registerFrontmatterHooks.ts` | Frontmatter Hook 注册 |
| `src/utils/hooks/hooksSettings.ts` | Hook 配置类型 |
| `src/utils/hooks/hooksConfigManager.ts` | 配置管理 |
| `src/utils/hooks/execPromptHook.ts` | Prompt Hook 执行 |
| `src/utils/hooks/execHttpHook.ts` | HTTP Hook 执行 |
| `src/utils/hooks/execAgentHook.ts` | Agent Hook 执行 |
| `src/utils/hooks/postSamplingHooks.ts` | 后采样 Hook |
| `src/utils/hooks/AsyncHookRegistry.ts` | 异步 Hook 注册表 |
| `src/utils/hooks/ssrfGuard.ts` | SSRF 防护 |

---

## 二十九、给学习者的建议

### 29.1 入门路径

1. **先读 `hookEvents.ts`**：理解事件系统设计
2. **读 `sessionHooks.ts`**：理解 Map 而非 Record 的设计精妙
3. **读 `registerFrontmatterHooks.ts`**：理解 Agent 的 hook 集成
4. **读 `execPromptHook.ts`**：理解 Prompt-based Hook
5. **最后读 `ssrfGuard.ts`**：理解安全考虑

### 29.2 实践建议

- **添加自己的 Hook**：修改 `~/.claude/settings.json`
- **观察 Hook 输出**：用 `--verbose` 参数
- **测试 Prompt Hook**：用 LLM 评估安全
- **监控 Hook 性能**：用 Performance Profiler

### 29.3 关键洞察

理解 Claude Code 的 Hook 系统需要把握以下关键点：

1. **事件驱动架构**：通过事件系统连接各个生命周期
2. **Map 而非 Record**：性能优化的精妙设计
3. **多种 Hook 类型**：覆盖不同场景
4. **安全防护**：SSRF、超时、失败处理
5. **可扩展性**：易于添加新事件和 Hook 类型

**掌握这些，你就能设计出灵活的、可扩展的事件驱动系统！** 🎯