# Claude Code 设计原理与核心模块

> 本文基于 `src/` 下 TypeScript / TSX 源码逐项核对，目标是给出一份**与代码事实一致**的设计原理与核心模块说明，而不是套用第三方叙事或营销话术。每一条断言都可以在 `src/` 下找到对应实现位置。
>
> 与 `17-architecture.md` 的区别：`17-architecture.md` 是"功能讲解"（每个模块做什么），本文是"设计原理"（为什么这样做、模块之间怎么协同、关键 trade-off 是什么）。

---

## 目录

1. [顶层设计哲学](#1-顶层设计哲学)
2. [运行范式：queryLoop 状态机](#2-运行范式queryloop-状态机)
3. [整体分层架构](#3-整体分层架构)
4. [六大核心模块](#4-六大核心模块)
   - 4.1 [上下文治理：五级压缩 + 三层记忆](#41-上下文治理五级压缩--三层记忆)
   - 4.2 [Agent 调度：Subagent 派生与隔离](#42-agent-调度subagent-派生与隔离)
   - 4.3 [工具与 MCP 标准化接入](#43-工具与-mcp-标准化接入)
   - 4.4 [权限系统：四层防御 + 多种 mode](#44-权限系统四层防御--多种-mode)
   - 4.5 [Hooks + Skills + Slash Commands 三件套](#45-hooks--skills--slash-commands-三件套)
   - 4.6 [可观测性与持久化](#46-可观测性与持久化)
5. [Plan Mode：只读规划的真相](#5-plan-mode只读规划的真相)
6. [关键设计模式](#6-关键设计模式)
7. [常见误读澄清](#7-常见误读澄清)

---

## 1. 顶层设计哲学

把 1987 个文件的代码反复阅读后，可以归纳为 **8 条互相正交的设计哲学**。这些不是营销语言，是源码中**反复出现**的设计取舍。

### 1.1 Prompt 缓存命中率是一等公民

`getAllBaseTools()` 顶部的注释明确写着：

```
NOTE: This MUST stay in sync with … in order to cache the system prompt across users.
```

`assembleToolPool()` 的注释进一步解释：

> Sort each partition for prompt-cache stability, keeping built-ins as a contiguous prefix. The server's `claude_code_system_cache_policy` places a global cache breakpoint after the last prefix-matched built-in tool; a flat sort would interleave MCP tools into built-ins and invalidate all downstream cache keys whenever an MCP tool sorts between existing built-ins.

由此推导出一连串下游设计：
- 工具列表按 `localeCompare` 排序；
- MCP 工具**追加**到 built-in 之后，而非混排；
- `userContext` 在跨调用间做 `memoize`（`utils/userContext.ts`）；
- 模型固定为最新别名（`claude-opus-4-8`），保证模型级缓存复用；
- `feature('BREAK_CACHE_COMMAND')` 内部开关可强制给 system prompt 注入 `[CACHE_BREAKER: <random>]`，强制下次请求重新缓存。

**Why**：每次 cache miss 多花 ~1.25× 输入 token 成本，对 30+ 工具调用 / turn 的 agent 任务而言，cache hit 率从 50% → 90% 是利润率问题。

### 1.2 三层功能门控分层对外暴露

```ts
// 第一层：编译时（bun:bundle 静态分析）
import { feature } from 'bun:bundle'
if (feature('BUDDY')) { /* ... */ }
// 外部构建里 feature() 编译为 false，整块代码被 DCE

// 第二层：用户类型
process.env.USER_TYPE === 'ant'
// 'ant' = Anthropic 内部，'external' = 默认

// 第三层：远程 A/B 实验
import { getFeatureValue_CACHED_MAY_BE_STALE } from './services/analytics/growthbook.js'
const kairosEnabled = getFeatureValue_CACHED_MAY_BE_STALE('tengu_kairos', false)
```

外部发布版只看到 `feature()` 评估为 `false` 的代码路径，约 50 个开关构建时剪掉约 30% 的功能体积。`ant` 用户在此基础上看到 26+ 个内部命令、特殊工具（`ConfigTool`、`TungstenTool`、`REPLTool`）、`raw-read` 的 MDM 接口、20 分钟（而非 6 小时）的 GrowthBook 刷新。GrowthBook 在两者之上做 A/B 灰度。**Why**：混淆内部功能与外部功能要么泄露、要么体积膨胀，三层正交恰好把这三种隔离维度分开。

### 1.3 "REPL 视图 vs SDK 视图" 是可分离关注点

`QueryEngine.ts` 顶部的 JSDoc 写得很清楚：

> It extracts the core logic from `ask()` into a standalone class that can be used by both the headless/SDK path and (in a future phase) the REPL.

`QueryEngine` 是**状态机**：消息历史、token 用量、读取文件缓存、权限拒绝记录。`useCanUseTool` / `interactiveHelpers.tsx` / `REPL.tsx` 是**视图**：把状态机的事件流画到终端。SDK 用户（`@anthropic-ai/claude-agent-sdk`）能复用同一份对话状态机，而不需要把 Ink/React 拉进自己的进程。

### 1.4 工具优先于 Bash，但 Bash 永远兜底

`utils/permissions/permissions.ts` 与 `tools/BashTool/bashPermissions.ts` 共同实现 "**工具能拦截时一定拦截，拦不住的由 Bash 兜底**"：

- `Edit` / `Write` 工具：先做路径在白名单/黑名单里的规则匹配（`shellRuleMatching.ts`），失败再交给 Bash classifier；
- `Bash`：单独的 `bashClassifier`（用 LLM 二次判定）+ 命令解析（`shell-quote`）+ 沙箱决定（`shouldUseSandbox.ts`）。

**Why** 不让一切都走 Bash？因为给工具名 + JSON Schema 的 `tool_use` 块在 UI 上能画图标、能精确审计、能从模型侧并行执行；而 Bash 字符串是不透明的。

### 1.5 每个子 Agent 拥有独立状态机，但通过文件系统/IPC 协同

`utils/forkedAgent.ts` 的 `createSubagentContext()` 关键操作是构造一个 **`setAppState` 是 no-op 的 `ToolUseContext`**——子 Agent 看到的 AppState 是快照。但同时保留一个 `setAppStateForTasks` 通道，让子 Agent 注册的后台任务（输出文件、监视器）写回主进程 store。

子 Agent 之间的协作不是进程内对象共享，而是：
- **共享文件系统**（worktree 隔离是可选的，由 `isolation: 'worktree' | 'remote'` 决定）；
- **共享任务列表**（`~/.claude/tasks/<id>.json`）；
- **`SendMessage` 工具**（`tools/SendMessageTool/`）通过 named address 发送消息；
- **远程可达**（`tools/RemoteAgentTask/`，CCR 容器中启动）。

### 1.6 持久化是无处不在的隐式能力

`utils/sessionStorage.ts` 长达 5,105 行。每一类可恢复的运行时状态都有写盘策略：

- 消息历史：`recordTranscript`（**在用户消息被接受时就写**，不等到 API 响应——防止 `kill -9` 后无法 `--resume`）；
- 文件编辑：`fileHistoryMakeSnapshot`（可回放任意一次修改前的状态）；
- 子 Agent：`recordSidechainTranscript` + `writeAgentMetadata`；
- 待处理权限：`OrphanedPermission`（断线后下次启动恢复）；
- 遥测队列：`firstPartyEventLoggingExporter`。

`QueryEngine` 顶部的注释解释了一个微妙的设计权衡：

> fire-and-forget. Scripted calls don't --resume after kill-mid-request. The await is ~4ms on SSD, ~30ms under disk contention — the single largest controllable critical-path cost after module eval.

**单次 `await recordTranscript` 是 4ms**——对交互式 REPL 不可忽视，对 `--bare` 脚本模式可忽略。

### 1.7 提示词是可审计的产物

`--dump-system-prompt`（`entrypoints/cli.tsx`）是一个 ant-only 的 fast path：直接渲染系统 prompt 并打印退出。`getDumpPromptsPath()`（`services/api/dumpPrompts.ts`）把每个请求的真实 prompt 写到本地文件，用于 "prompt sensitivity evals"。

整个 system prompt 通过 `asSystemPrompt([...])` 构造，传入 `[] | [string] | [{type:'text', text:string, cache_control?:...}]`——每一块都是显式的、可索引的、可注入 cache_control 的。

### 1.8 错误是 typed，不是字符串

`utils/errors.ts` 定义 `AbortError`、`APIUserAbortError`、`ImageSizeError`、`ImageResizeError`、`ConfigParseError` 等。每种错误都被各层 catch 后转成对应的 `tool_result` `is_error: true` 块，而不是把异常 message 暴露给模型——避免模型在错误字符串中"看到"被设计为隐藏的细节（如系统路径、凭据）。

`FallbackTriggeredError` 是一种特殊的内部 error：API 触发 fallback 模型时 query loop 抛此异常，外层用 SDK 重试。

---

## 2. 运行范式：queryLoop 状态机

整个 CLI 不管是 REPL、SDK 还是 headless，最终都进入 `src/query.ts` 的 `queryLoop()`（约 2,337 行）。这是 Claude Code 的"心脏"。

### 2.1 `QueryParams` — 公开 API

```ts
export type QueryParams = {
  messages: Message[]                  // 当前会话消息流
  systemPrompt: SystemPrompt           // 已渲染的 system prompt
  userContext: { [k: string]: string } // 用户级上下文（CWD、git、env 等）
  systemContext: { [k: string]: string } // 系统级上下文
  canUseTool: CanUseToolFn             // 工具权限检查
  toolUseContext: ToolUseContext       // 工具执行上下文（含 abortController、options 等）
  fallbackModel?: string               // 降级模型（fallback 触发时使用）
  querySource: QuerySource             // 调模型时的源标识（用于埋点/限流）
  maxOutputTokensOverride?: number     // 单次响应最大 token
  maxTurns?: number                    // 最大轮次
  skipCacheWrite?: boolean             // 跳过 prompt cache 写入
  taskBudget?: { total: number }       // API task_budget（output_config.task_budget）
  deps?: QueryDeps                     // 测试时可注入 fake
}
```

### 2.2 `State` — 跨轮次可变状态

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

**关键设计**：7 个 continue 站点**集中通过 `state = { ... }` 整体赋值**，而不是 9 个 `state.x = ...` 散落赋值。这样每个 continue 站点的 state 都是不可变快照，状态可观察、可测试。

### 2.3 主循环骨架

```ts
async function* queryLoop(params, consumedCommandUuids) {
  // 入口一次性 setup
  let state: State = initializeQueryLoopState(params)
  const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
  const config = buildQueryConfig()
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)

  while (true) {
    // 1. 准备本轮消息（5 级压缩）
    const preparedTurn = await prepareQueryTurn({ ... })
    
    // 2. blocking limit 检查
    if (isAtBlockingLimit) { yield apiError; return { reason: 'blocking_limit' } }
    
    // 3. 流式调模型
    yield* executeModelStreamingTurn({ ..., executionState, deps })
    
    // 4. post-sampling hook
    void executePostSamplingHooks([...messagesForQuery, ...assistantMessages], ...)
    
    // 5. abort 处理
    if (signal.aborted) { return { reason: 'aborted_streaming' } }
    
    // 6. 分支：是否需要 follow-up
    if (!needsFollowUp) {
      const { outcome, emittedEvents } = await handleNonFollowUpTurn({ ... })
      if (outcome.kind === 'continue') { state = outcome.state; continue }
      return outcome.terminal
    }
    
    // 7. follow-up：执行工具 + 准备下一轮
    const followUpTurnResult = await processFollowUpTurn({ ... })
    state = nextTurnPreparation.nextState!
  }
}
```

### 2.4 `queryDeps` — I/O 依赖注入

```ts
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

**Why**：让测试不依赖 `spyOn` 单个模块（callModel / autocompact 在 6-8 个测试文件里被 spy）。使用 `typeof fn` 让签名自动同步。

### 2.5 Terminal 终止原因

| Reason | 触发位置 | 含义 |
|--------|---------|------|
| `completed` | handleNonFollowUpTurn 默认 / stop failure hook | 正常结束 |
| `aborted_streaming` | queryLoop 阶段 5 | streaming 阶段被 abort |
| `aborted_tools` | processFollowUpTurn | 工具执行阶段被 abort |
| `blocking_limit` | queryLoop 阶段 2 | 命中硬 token 上限 |
| `image_error` | executeModelStreamingTurn catch / handleNonFollowUpTurn | 图片尺寸/调整错误 |
| `model_error` | executeModelStreamingTurn catch | 通用 model error |
| `prompt_too_long` | handleNonFollowUpTurn | 压缩失败 |
| `stop_hook_prevented` | handleNonFollowUpTurn | stop hook 阻止 |
| `hook_stopped` | processFollowUpTurn | hook 阻止继续 |
| `max_turns` | prepareNextLoopTurn | 达到 maxTurns |

---

## 3. 整体分层架构

自上而下六层：

```
┌─────────────────────────────────────────────────────────────┐
│ 用户交互层  CLI / VSCode 插件 / Web 远程 / CCR 容器          │
├─────────────────────────────────────────────────────────────┤
│ 命令/Skill 扩展层  Slash 内置命令 + Skills + CLAUDE.md        │
├─────────────────────────────────────────────────────────────┤
│ 核心引擎层  queryLoop · 上下文压缩 · Subagent 编排 · Hooks  │
├─────────────────────────────────────────────────────────────┤
│ 工具执行层  MCP 标准化工具集群 · 沙盒 · 权限分级控制           │
├─────────────────────────────────────────────────────────────┤
│ 服务支撑层  会话持久化 · Checkpoint · 链路观测 · Token 计费  │
├─────────────────────────────────────────────────────────────┤
│ 基础设施层  文件系统隔离 · 状态数据库 · 向量记忆 · API 网关   │
└─────────────────────────────────────────────────────────────┘
```

横向看：

```
Cross-cutting concerns (Layer cake)
┌──────────────────────────────────────────────────┐
│  Analytics  │  GrowthBook  │  OpenTelemetry      │
├──────────────────────────────────────────────────┤
│  feature() gating  │  USER_TYPE  │  env vars      │
├──────────────────────────────────────────────────┤
│  Hooks  (utils/hooks.ts, 87 hooks)               │
├──────────────────────────────────────────────────┤
│  Permissions  (4-layer: rules → classifier → …)  │
├──────────────────────────────────────────────────┤
│  State  (AppState + Zustand-like store)          │
├──────────────────────────────────────────────────┤
│  Persistence  (transcript / file history / task) │
└──────────────────────────────────────────────────┘
```

---

## 4. 六大核心模块

下面逐一拆解。每个模块都注明源码位置和**与常见误读的差异**。

### 4.1 上下文治理：五级压缩 + 三层记忆

**源码位置**：
- 压缩：`src/services/compact/{snipCompact,microCompact,apiMicrocompact,autoCompact,reactiveCompact,sessionMemoryCompact}.ts`
- 压缩串联：`src/query.ts` 的 `prepareQueryTurn()`（约 L293-462）
- 记忆：`src/memdir/{memoryScan,memoryTypes}.ts`、`src/utils/memoryFileDetection.ts`

**核心目标**：在有限窗口内最大化保留工程有效信息，隔离噪声，减少重复读取文件带来的 Token 膨胀。

#### 4.1.1 三层记忆存储

**第一层：静态持久规则记忆（CLAUDE.md / .claude/）**
- 项目根目录 `CLAUDE.md` 存储编码规范、架构约束、校验标准；
- 会话启动自动注入 System Prompt；
- 任何压缩逻辑不会删除该区块，永久常驻。

**第二层：会话短期内存**
- 主 Agent / 每个 Subagent 分配独立消息缓冲区（`createSubagentContext`）；
- 隔离工具日志、文件源码、调试输出，互不污染；
- 区分系统指令、用户需求、工具返回、模型思考四类消息。

**第三层：长期文件记忆（`memdir`）**
- 类型：`user / feedback / project / reference`（见 `MEMORY_TYPES`）；
- 自动扫描 `.claude/memories/*.md`，按 `mtimeMs` 排序；
- `findRelevantMemories` 用 side-query 选择最相关的 N 条注入；
- 注意：**它不是向量检索**，是 LLM 文本筛选（`selectRelevantMemories` 调用 `sideQuery`）。

#### 4.1.2 五级惰性压缩流水线

按"粗到细、可逆到不可逆"排序：

```ts
async function prepareQueryTurn({ ... }) {
  let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

  // 1. applyToolResultBudget —— 超大 tool_result 截断（microcompact 的子步骤）
  messagesForQuery = await applyToolResultBudget(...)

  // 2. HISTORY_SNIP —— ant-only 硬截断老历史
  if (feature('HISTORY_SNIP')) {
    messagesForQuery = snipModule.snipCompactIfNeeded(messagesForQuery).messages
  }

  // 3. microcompact —— 增量清理大 tool result
  messagesForQuery = await deps.microcompact(messagesForQuery, toolUseContext, querySource)

  // 4. CONTEXT_COLLAPSE —— 历史投影折叠（ant-only）
  if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
    messagesForQuery = (await contextCollapse.applyCollapsesIfNeeded(...)).messages
  }

  // 5. autocompact —— token 上限前 5% 触发完整压缩
  ;({ compactionResult } = await deps.autocompact(messagesForQuery, ...))

  // reactiveCompact —— 响应式：模型返回 prompt_too_long 时再压缩
  // 不在 prepareQueryTurn 里，而在 handleNonFollowUpTurn 里触发
}
```

**每一级的作用域**：

| 层级 | 触发时机 | 是否 ant-only | 是否可逆 |
|------|---------|--------------|---------|
| `applyToolResultBudget` | 每轮 LLM 调用前 | 否 | 否（原地替换） |
| `HISTORY_SNIP` | 每轮 LLM 调用前 | 是 | 是（保留原始边界） |
| `microcompact` | 每轮 LLM 调用前 | 否 | 否 |
| `CONTEXT_COLLAPSE` | 每轮 LLM 调用前 | 是 | 是（投影保留） |
| `autocompact` | token 接近 contextWindow 时 | 否 | 否 |
| `reactiveCompact` | API 返回 prompt_too_long | 否（feature gate） | 否 |

**为什么是这个顺序**：
- snip 在 microcompact 之前：snip 是粗粒度、可逆的边界切分，microcompact 是细粒度 token 替换；先 snip 可以减少 microcompact 要处理的消息数；
- autocompact 在最末：作为最后兜底，只有前面三层都压不动时才触发。

**熔断机制**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`（`autoCompact.ts:70`），注释解释：

> Stop trying autocompact after this many consecutive failures. BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272) in a single session, wasting ~250K API calls/day globally.

**API 端 microcompact**（`apiMicrocompact.ts`）：把压缩策略下发到 API 端执行（`output_config.context_management`），客户端只声明 `clear_tool_uses_20250919` / `clear_thinking_20251015` 策略，由 API 服务端执行实际的清理。这能省掉 client→server 的来回 token 计费。

#### 4.1.3 Subagent 隔离分流

复杂代码探索、大规模重构派生独立子 Agent（`utils/forkedAgent.ts` 的 `createSubagentContext()`）：
- 子 Agent 拥有专属 `messages`、`toolUseContext`、`abortController`；
- `setAppState` 默认是 no-op（除非 `shareSetAppState: true`）；
- 执行结束仅结构化摘要回流主会话；
- 完整文件、海量终端日志不污染主上下文。

注释中的实测数据：

> Explore agent omitClaudeMd: true. Saves ~5-15 Gtok/week across 34M+ Explore spawns.

### 4.2 Agent 调度：Subagent 派生与隔离

**源码位置**：
- 入口：`src/tools/AgentTool/`（`AgentTool.tsx`、`runAgent.ts`、`forkSubagent.ts`）
- 定义：`src/tools/AgentTool/loadAgentsDir.ts`
- 内置：`src/tools/AgentTool/builtInAgents.ts` + `built-in/{generalPurposeAgent,exploreAgent,planAgent,statuslineSetup,claudeCodeGuideAgent,verificationAgent}.ts`
- 隔离：`src/utils/forkedAgent.ts`

#### 4.2.1 Agent 类型体系

```ts
type AgentDefinition =
  | BuiltInAgentDefinition   // 源码内置
  | CustomAgentDefinition    // 用户/项目/policy settings
  | PluginAgentDefinition    // 插件提供

type BaseAgentDefinition = {
  agentType: string
  whenToUse: string
  tools?: string[]
  disallowedTools?: string[]
  skills?: string[]
  mcpServers?: AgentMcpServerSpec[]
  hooks?: HooksSettings
  color?: AgentColorName
  model?: string
  effort?: EffortValue
  permissionMode?: PermissionMode
  maxTurns?: number
  isolation?: 'worktree' | 'remote'  // ant-only
  background?: boolean
  initialPrompt?: string
  memory?: AgentMemoryScope
  omitClaudeMd?: boolean              // Explore/Plan 不注入 CLAUDE.md
  // ...
}
```

#### 4.2.2 Built-in Agents 清单

```ts
// src/tools/AgentTool/builtInAgents.ts
const agents: AgentDefinition[] = [
  GENERAL_PURPOSE_AGENT,
  STATUSLINE_SETUP_AGENT,
]

if (areExplorePlanAgentsEnabled()) {
  agents.push(EXPLORE_AGENT, PLAN_AGENT)
}

// CLAUDE_CODE_GUIDE_AGENT —— 仅在非 SDK entrypoint 注入
if (isNonSdkEntrypoint) agents.push(CLAUDE_CODE_GUIDE_AGENT)

if (feature('VERIFICATION_AGENT') && getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)) {
  agents.push(VERIFICATION_AGENT)
}
```

| Agent | 角色 | 关键约束 |
|-------|------|---------|
| `general-purpose` | 默认 worker，无读写限制 | 完整工具集 |
| `Explore` | **只读**代码库探索 | `disallowedTools: [Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit]`；`omitClaudeMd: true` |
| `Plan` | **只读**方案设计 | 同 Explore |
| `statusline-setup` | 引导用户配置 statusline | `tools: ['Read', 'Edit']` |
| `claude-code-guide` | 文档问答（仅非 SDK 入口） | |
| `verification-agent` | feature-gated；A/B 测试中 | |

**注意**：源码里**没有** `ImplementAgent` / `TestAgent` 这类命名——那是社区文章编造的命名。实际工作中如果需要"专门写代码的子 agent"，需要用户在 `~/.claude/agents/` 下自定义。

#### 4.2.3 Subagent 隔离机制

```ts
// src/utils/forkedAgent.ts
export function createSubagentContext(parent: ToolUseContext): ToolUseContext {
  return {
    ...parent,
    setAppState: () => {},  // ← no-op! 子 agent 看不到主 store
    setAppStateForTasks: parent.setAppStateForTasks,  // ← 但后台任务仍写主 store
  }
}
```

- **默认隔离**：`setAppState` 是 no-op；
- **后台任务回写**：通过 `setAppStateForTasks` 让子 agent 注册的 monitor、dream 等长寿命任务把进度写回主 store；
- **可选共享**：通过 `SubagentContextOverrides.shareSetAppState` 显式 opt-in；
- **Worktree 隔离**：`isolation: 'worktree'` 启动子进程 + git worktree（ant-only）；
- **远程隔离**：`isolation: 'remote'` 把子 agent 派到 CCR 容器（ant-only）。

#### 4.2.4 `SendMessage` 工具与 Teammate

子 Agent 之间通信通过 `tools/SendMessageTool/`：
- `SendMessage` 工具：基于 named address 投递消息到指定 team member；
- 任务类型：`in_process_teammate`（同进程）或 `remote_agent`（CCR 容器）；
- 接收方通过 `task-notification` 形式在主 thread 收到，**不直接修改主 agent 的 messages**。

### 4.3 工具与 MCP 标准化接入

**源码位置**：
- 工具抽象：`src/Tool.ts`（792 行）+ `src/tools.ts`
- 工具实现：`src/tools/`（53 个子目录）
- MCP 客户端：`src/services/mcp/{client,mcpConnectionManager,types}.ts`

#### 4.3.1 `Tool<Input, Output>` 类型

```ts
type Tool<Input, Output> = {
  name: string
  description: string | (() => Promise<string>)
  inputSchema: ZodSchema | (() => ZodSchema)
  call: (input: Input, ctx: ToolUseContext) => Promise<Output>
  renderToolUseMessage: (input: Input, ui: ToolRenderContext) => ReactNode
  renderToolResultMessage: (output: Output, ui: ToolRenderContext) => ReactNode
  isEnabled: () => boolean
  isReadOnly?: () => boolean
  isDestructive?: () => boolean
  aliases?: string[]  // 兼容旧名（e.g., KillShell → TaskStop）
}
```

#### 4.3.2 `assembleToolPool()` — 缓存友好的工具池装配

```ts
// src/tools.ts:347
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // Sort each partition for prompt-cache stability, keeping built-ins as a contiguous prefix.
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

**核心要点**：built-in 工具与 MCP 工具**分区排序、合并时 built-in 在前**——避免 MCP 工具插入到 built-in 之间导致 system prompt 缓存失效。

#### 4.3.3 `getAllBaseTools()` 核心清单

| 工具 | 是否稳定 | 备注 |
|------|---------|------|
| `Agent` | ✅ | 启动 subagent |
| `TaskOutput` | ✅ | 读 background agent 输出 |
| `Bash` | ✅ | shell 执行 |
| `Glob` / `Grep` | ✅ | arm64 native build 改为 embedded bfs/ugrep |
| `FileRead` / `FileEdit` / `FileWrite` | ✅ | |
| `NotebookEdit` | ✅ | Jupyter |
| `WebFetch` / `WebSearch` | ✅ | |
| `TodoWrite` | ✅ | |
| `TaskStop` | ✅ | |
| `AskUserQuestion` | ✅ | 多选问答 |
| `Skill` | ✅ | 加载 .claude/skills/ |
| `EnterPlanMode` / `ExitPlanModeV2` | ✅ | |
| `ListMcpResources` / `ReadMcpResource` | ✅ | |
| `SendMessage` | ✅ | 派生 teammate |
| `TaskCreate` / `TaskGet` / `TaskUpdate` / `TaskList` | gate | `isTodoV2Enabled()` |
| `ToolSearch` | gate | MCP deferral |

其余工具（`ConfigTool`、`TungstenTool`、`WebBrowserTool`、`LSPTool`、`EnterWorktreeTool`、`WorkflowTool`、`CronTools`、`MonitorTool` 等）都受 `feature()` / `process.env.USER_TYPE === 'ant'` / `ENABLE_*` env var 门控。

**统计**：核心稳定工具 ~17 个；全量（含所有 feature gate）约 40-50 个。

#### 4.3.4 MCP（Model Context Protocol）

**协议层**：`src/services/mcp/client.ts`（3,348 行）实现 JSON-RPC 2.0 over stdio / SSE / HTTP / WS。

**传输类型**（`McpServerType`）：`stdio` / `sse` / `http` / `ws` / `sdk` / `sse-ide` / `ws-ide` / `claudeai-proxy`。

**配置层级**：
1. 用户级 `~/.claude/mcp.json`
2. 项目级 `.mcp.json`
3. 内置 server（`claudeai`、`chrome`、`atlas`）
4. 插件提供
5. 远程 managed settings 注入

**OAuth 流程**：Claude Code 启动 HTTP server 在随机端口 → 浏览器重定向到 MCP server 的 OAuth 页 → server 重定向回 `http://localhost:<port>/callback?code=...` → 交换 code → access token + refresh token 存到 keychain。

**Elicitation**：MCP server 可以反向问用户问题（`elicitationHandler.ts`）；`-32042` 错误码触发 URL elicitation。

**Channel 机制**：`channelAllowlist.ts` / `channelNotification.ts` / `channelPermissions.ts`——MCP server 主动推送通知到 client。

### 4.4 权限系统：四层防御 + 多种 mode

**源码位置**：
- 决策核心：`src/utils/permissions/permissions.ts`（1,486 行）
- Bash classifier：`src/utils/permissions/bashClassifier.ts` + `src/tools/BashTool/bashPermissions.ts`
- 自动模式：`src/utils/autoModeDenials.ts` + transcript classifier
- 路径校验：`src/utils/permissions/pathValidation.ts`
- React 入口：`src/hooks/useCanUseTool.tsx`
- Mode 类型：`src/types/permissions.ts`

#### 4.4.1 四层防御

```
hasPermissionsToUseTool
├─ 1a. getDenyRuleForTool           ← 黑名单永远拒绝
├─ 1b. getAllowRuleForTool          ← 白名单允许（考虑 add-dir / 路径 / 命令白名单）
├─ 1c. rule matching (path/regex/glob)
├─ 2. bash classifier (LLM 二次判定，仅 ant 真正启用)
├─ 3. auto mode classifier (基于 transcript，ant-only)
└─ 4. path validation (防 traversal / symlink / 绝对路径)
   ↓
   allow → 记录 + resolve(allow)
   deny  → 记录 + resolve(deny)
   ask   → handleInteractivePermission (REPL)
                       / handleCoordinatorPermission
                       / handleSwarmWorkerPermission
```

#### 4.4.2 PermissionMode

```ts
// src/types/permissions.ts
export const EXTERNAL_PERMISSION_MODES = [
  'default', 'acceptEdits', 'plan', 'bypassPermissions', 'dontAsk'
] as const
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

| Mode | 行为 | 适用 |
|------|------|------|
| `default` | 所有写操作询问 | 默认 |
| `plan` | 禁用写工具；只能 ExitPlanMode 退出 | 只读规划 |
| `acceptEdits` | 自动放行 Edit/Write，其它询问 | 信任编辑 |
| `bypassPermissions` | 完全自动（强提示需 sandbox） | 容器/VM |
| `dontAsk` | 静默拒绝所有询问 | CI |
| `auto`（ant-only） | transcript classifier 决策 | 内部 |
| `bubble`（ant-only） | 子 agent 升级用 | 内部 |

**注意**：这是 **6 种 mode**，不是文档中常见的"7 级权限护栏"。

#### 4.4.3 `dangerousPatterns.ts`

`utils/permissions/dangerousPatterns.ts` 在 prompt 阶段就拒绝危险命令（`rm -rf /`、`curl | bash`），不进入 classifier。

### 4.5 Hooks + Skills + Slash Commands 三件套

#### 4.5.1 Hooks（事件驱动扩展）

**源码位置**：
- 类型与注册：`src/utils/hooks.ts`（5,297 行）+ `src/utils/hooks/`
- 设置层：`src/utils/hooks/hooksSettings.ts`

**事件类型**（`HookEvent`）：
- `PreToolUse` / `PostToolUse`：工具调用前后
- `PreCompact` / `PostCompact`：压缩前后
- `SessionStart` / `SessionEnd`：会话开始/结束
- `Notification`：通知事件
- `Stop` / `SubagentStop`：停止事件
- `UserPromptSubmit`：用户输入提交时

**Hook 来源**（`HookSource`）：
```ts
type HookSource =
  | EditableSettingSource  // 'userSettings' | 'projectSettings' | 'localSettings'
  | 'policySettings'
  | 'pluginHook'
  | 'sessionHook'
  | 'builtinHook'
```

**三种 hook 类型**（`execAgentHook.ts` / `execHttpHook.ts` / `execPromptHook.ts`）：
- **agent hook**：在 hook 中调用 sub-agent 做 LLM 判定；
- **http hook**：向外部 HTTP endpoint POST 事件；
- **prompt hook**：在 hook 中调用 LLM 做 prompt 判定（带 SSRF 防护 `ssrfGuard.ts`）。

**常见误区**：Hooks 是**用户/企业在 settings.json 里自定义**的命令或脚本；它们不是 Claude Code 内置的"自动化校验流水线"。Claude Code 不集成 Sonar、ESLint、单测——如果用户需要自动化校验，必须自己写 PostToolUse hook 调相应工具。

#### 4.5.2 Skills（领域能力库）

**源码位置**：`src/skills/` + `src/utils/processUserInput/processSkillCommand.tsx`

**Skill 文件位置**：`.claude/skills/<name>/SKILL.md`，可包含 frontmatter 与正文。

**加载机制**：
- `Skill` 工具主动加载（不会自动注入，节省 token）；
- `SkillTool.tsx` 把 SKILL.md 渲染成 system prompt 的扩展；
- 启动时 `skillPrefetch` 异步预取（`startSkillDiscoveryPrefetch`）；
- 改进机制：`utils/hooks/skillImprovement.ts`（ant-only）通过 side-channel LLM 自动改写 skill。

#### 4.5.3 Slash Commands（内置命令系统）

**源码位置**：`src/commands.ts`（87 个）+ `src/commands/`

**关键命令**：
- `/plan`：进入 plan mode；
- `/compact`：手动压缩；
- `/context`：查看 token 占用；
- `/fork`：分叉会话并行试错（feature-gated `FORK_SUBAGENT`）；
- `/resume`：断点续跑；
- `/agents`：管理 subagent 定义；
- `/mcp`：MCP 服务器管理；
- `/permissions`：权限规则编辑。

### 4.6 可观测性与持久化

#### 4.6.1 四类产物

- **对话级别**：`SDKMessage`（流式输出，typed union）
- **请求级别**：`span.model_request_*` 事件 + `NonNullableUsage`（累积 token 计数）
- **用户级别**：`firstPartyEventLogger`（不含 PII 的事件流）
- **诊断级别**：`diagLogs`（`logForDiagnosticsNoPII`，结构化日志 + 自定义 channel）
- **性能级别**：`fpsTracker`、`headlessProfiler`、`startupProfiler`、`queryProfiler`——4 个不同粒度的性能采样

#### 4.6.2 四个 Profiler

| Profiler | 采样粒度 | 关键埋点 |
|----------|---------|---------|
| `startupProfiler` | 启动期 | `main_tsx_entry` / `cli_entry` / `init_function_start` / `init_configs_enabled` / `before_getSystemPrompt` / `after_getSystemPrompt` |
| `queryProfiler` | 每轮 query | `query_api_loop_start` / `query_api_streaming_start` / `query_api_streaming_end` / `query_tool_execution_start` / `query_tool_execution_end` |
| `headlessProfiler` | SDK 模式 | `headlessProfilerCheckpoint('query_started')` |
| `fpsTracker` | 渲染帧率 | 终端 UI |

#### 4.6.3 OpenTelemetry

`utils/telemetry/otel.ts` 接入 OTel meter，emit `span.model_request_*` 等事件。**但**细粒度的产品级埋点是 `tengu_*` 自定义事件（`firstPartyEventLogger`），不是 OTel span。两者并行存在。

#### 4.6.4 会话持久化

**`recordTranscript(messages)`**（`utils/sessionStorage.ts`）：
- 在用户消息 push 进 mutableMessages 时**立即**写盘；
- 不等到 API 响应；
- 防止 `kill -9` 后无法 `--resume`。

**`fileHistoryMakeSnapshot`**（`utils/fileHistory.ts`）：
- 文件级快照，写到 `<session>/file-history/<hash>.json`；
- `/rewind` 命令可回滚到任意快照。

**`--resume`**（`utils/conversationRecovery.ts`）：
- 读 transcript，构造 AppState，恢复消息历史；
- **注意**：只能恢复**消息**和**文件缓存**，无法"任意节点续跑"任务树。

**`OrphanedPermission`**：
- 断线时未完成的权限询问，写盘保存；
- 下次启动恢复。

---

## 5. Plan Mode：只读规划的真相

### 5.1 Plan Mode 的实际行为

源码中 `permissionMode = 'plan'` 只是把 `Write` / `Edit` / `NotebookEdit` / `Bash`（写操作子集）等工具**从工具列表中过滤掉**，并禁用 MCP server 的写操作。**它不会自动 spawn Explore agent**。

```ts
// InteractiveHelpers.tsx 大致逻辑
if (toolPermissionContext.mode === 'plan') {
  // Plan mode: filter out write tools, allow ExitPlanModeTool
}
```

主 Agent 想读代码库时，仍然需要**显式调用 `Agent` 工具并指定 `subagent_type: 'Explore'`**。readme.txt 中"强制派生 Explore 子 Agent 全域扫描"是臆想。

### 5.2 Plan Mode 工具差异

Plan Mode 下可用的工具子集：
- 只读：`Read` / `Glob` / `Grep` / `WebFetch` / `WebSearch`
- `Bash`（仅允许只读命令，受 `shouldUseSandbox.ts` 限制）
- `Agent`（可派生 `Explore` / `Plan` subagent）
- `EnterPlanMode`（再次进入）/ `ExitPlanModeV2`（提交 plan 退出）
- `TodoWrite` / `AskUserQuestion` / `Skill`

### 5.3 Plan Mode 退出

模型调用 `ExitPlanModeV2` 工具，传入 `plan` 内容，UI 弹窗让用户审批。用户接受 → mode 切回 `default`；驳回 → 维持 plan mode。

---

## 6. 关键设计模式

把跨 1987 个文件的实现模式归纳为 **10 条**：

### 6.1 Builder 模式（Tool）

```ts
export const BashTool: Tool = buildTool({
  name: BASH_TOOL_NAME,
  description: async () => '...',
  inputSchema: lazySchema(() => z.object({...})),
  call: async (input, ctx) => { ... },
  renderToolUseMessage: (input) => { ... },
  renderToolResultMessage: (output) => { ... },
  isEnabled: () => true,
})
```

### 6.2 策略模式（Permission Handlers）

`useCanUseTool` 根据 mode 分发到 `interactiveHandler` / `coordinatorHandler` / `swarmWorkerHandler`。

### 6.3 状态机（discriminated union）

`SpeculationState`、`TaskStatus`、`MCPServerConnection.type`、`PermissionDecisionReason` 等都是显式 discriminated union。

### 6.4 观察者（store.subscribe）

`state/store.ts` 是轻量 pub-sub；`useSyncExternalStore` 订阅。

### 6.5 上下文对象（ToolUseContext）

工具的 `call(input, ctx)` 第二个参数是 `ToolUseContext`——避免全局状态，又显式传递所有依赖。

### 6.6 懒加载 + 静态消除（feature() + require）

- 编译时 `feature('BUDDY')` 决定代码是否进入二进制；
- 运行时 `require` 让冷启动路径不付出未启用工具的解析时间。

### 6.7 缓存键稳定性（localeCompare + 排序）

`assembleToolPool` 的排序是 cache-friendly design 的范本。

### 6.8 错峰并行（prefetch + lazy import）

`startMdmRawRead` + `startKeychainPrefetch` 在 main 入口处就 spawn 子进程，让 ~135ms 后续 import 与它们并行。

### 6.9 Typed Union 优于字符串标签

`Message` 是 `AssistantMessage | UserMessage | SystemMessage | ...` —— TypeScript discriminated union；每种消息有完整类型字段。

### 6.10 Single Source of Truth

- `assembleToolPool` —— 工具池唯一来源
- `getAllBaseTools` —— 内置工具唯一来源
- `getSystemPrompt` —— 系统 prompt 唯一来源
- `fetchSystemPromptParts` —— 用户/系统上下文唯一来源

每处"我想知道有哪些工具 / 权限是什么 / prompt 长啥样"都从这同一个函数拿，避免多份定义不一致。

---

## 7. 常见误读澄清

读其他社区文档时常见的说法，与源码事实对比：

| 误读 | 事实 |
|------|------|
| "七模式分级权限" | 实际是 6 种 `PermissionMode`（`default`/`plan`/`acceptEdits`/`bypassPermissions`/`dontAsk` + ant-only `auto`/`bubble`） |
| "三级容器沙盒隔离（每个 Subagent 独立 Docker）" | 所有 Subagent 共享同一进程；isolation 只有 `'worktree'`（Git worktree）和 `'remote'`（CCR 容器） |
| "Explore/Plan/Implement/Test 四类 built-in agent" | 只有 `general-purpose / statusline-setup / explore / plan / claude-code-guide` + 可选 `verification-agent`；Implement/Test 是用户自定义 |
| "三层强制自动化校验（lint/Sonar/单测）" | Claude Code 不集成任何外部校验工具；PostToolUse hook 由用户自己写 |
| "单任务连续 5 轮校验失败自动暂停" | autocompact 的连续失败熔断是 3 次；不存在通用 5 轮校验熔断 |
| "Plan Mode 自动派生 Explore agent 全域扫描" | Plan Mode 只禁用写工具；agent 必须显式调用 `Agent` 工具 |
| "任务 DAG（带依赖图、自动并行）" | TaskCreateTool 只是 todo list，没有依赖关系 |
| "Checkpoint 快照持久化 → 任意节点续跑" | `--resume` 只恢复消息历史和文件缓存；任务树不能"任意续跑" |
| "代码幻觉率降低 35%" / "Token 消耗降低 38%" | 源码无此类指标 |
| "原生 OpenTelemetry 全链路 trace" | OpenTelemetry 仅埋部分 span；细粒度指标走自定义 `tengu_*` 事件 |
| "45+ 原生开发工具" | 核心稳定工具约 17 个；全量含 feature gate 约 40-50 个 |
| "Agent Teams 持久多 Agent 协作" | 同进程 `in_process_teammate` 任务 + `SendMessage` 工具；没有独立持久化进程 |

---

## 附录：源码目录速查

```
src/
├── entrypoints/             # 8 个进程入口（cli.tsx / sdk / assistant / coordinator / …）
├── QueryEngine.ts           # 会话状态机（SDK/headless 入口）
├── query.ts                 # API + 工具执行循环（queryLoop 2337 行）
├── query/{config,deps,stopHooks,tokenBudget,transitions}.ts
├── Tool.ts                  # 工具抽象 + builder
├── tools.ts                 # 工具注册 + assembleToolPool
├── tools/                   # 53 个工具实现
│   ├── AgentTool/           #   核心：subagent 派发
│   │   ├── built-in/        #   Explore/Plan/GeneralPurpose/StatuslineSetup/Guide/Verification
│   │   ├── runAgent.ts
│   │   ├── forkSubagent.ts
│   │   └── loadAgentsDir.ts
│   ├── BashTool/            #   Bash 工具（最复杂）
│   ├── FileEditTool/  FileReadTool/  FileWriteTool/
│   ├── SkillTool/  TodoWriteTool/  …
│   └── AgentTool/built-in/
├── commands.ts              # 87 个斜杠命令
├── commands/                # 命令实现
├── components/              # 148 个 React UI 组件
├── hooks/                   # 87 个 React hooks
├── ink/                     # 自定义 React reconciler（终端）
├── state/                   # 全局状态管理（AppStateStore 569 行）
├── services/
│   ├── api/                 # Anthropic API 客户端
│   ├── mcp/                 # MCP 集成（client.ts 3348 行）
│   ├── compact/             # 上下文压缩（6 种策略）
│   ├── analytics/           # 遥测 + GrowthBook
│   ├── oauth/               # OAuth 流程
│   ├── plugins/             # 插件加载
│   └── remoteManagedSettings/
├── tasks/                   # 后台任务管理（7 种 TaskType）
│   ├── LocalAgentTask/  RemoteAgentTask/  LocalShellTask/
│   ├── InProcessTeammateTask/  LocalWorkflowTask/
│   ├── MonitorMcpTask/  DreamTask/  LocalMainSessionTask.ts
├── utils/
│   ├── permissions/         # 权限系统（核心，4 层防御）
│   ├── hooks.ts             # 磁盘钩子（5297 行）
│   ├── sessionStorage.ts    # 会话持久化（5105 行）
│   ├── telemetry/           # 可观测性
│   ├── forkSubagent.ts / forkedAgent.ts
│   └── fileHistory.ts
├── memdir/                  # 长期记忆（memories/）
├── skills/                  # Skill 加载
├── plugins/                 # 插件加载
├── types/                   # 共享类型
└── schemas/                 # Zod schema
```

**总计**：~2000 个 TypeScript / TSX 文件，~230k 行代码。

---

## 推荐阅读

| 想深入 | 看 |
|--------|------|
| queryLoop 主循环 | [`docs/09-query.md`](../09-query.md) |
| Agent 派发 | [`docs/12-run-agent.md`](../task/12-run-agent.md) |
| 工具执行 | [`docs/11-tool-execution.md`](../tool/11-tool-execution.md) |
| Skill 系统 | [`docs/13-skill-tool.md`](../skill/13-skill-tool.md) |
| 钩子机制 | [`docs/14-tool-hooks.md`](../tool/14-tool-hooks.md) + [`docs/15-utils-hooks.md`](../hook/15-utils-hooks.md) |
| Anthropic API | [`docs/16-claude-api.md`](../16-claude-api.md) |
| 整体架构与设计哲学 | [`docs/17-architecture.md`](17-architecture.md) |
| 任务规划 | [`docs/18-task-planning.md`](../task/18-task-planning.md) |
| 多 agent 编排 | [`docs/19-orchestration.md`](../task/19-orchestration.md) |
| 请求流 | [`docs/20-request-flow.md`](20-request-flow.md) |
| 隐藏功能 / feature gate | [`docs/05-hidden-commands.md`](../other/05-hidden-commands.md) + [`docs/07-feature-gates.md`](../other/07-feature-gates.md) |
