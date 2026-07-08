# ToolUseContext vs ToolPermissionContext —— 两个「Context」的职责边界

> 源码位置:
> - `ToolUseContext` 定义：`src/Tool.ts:158-300`
> - `ToolPermissionContext` 定义：`src/Tool.ts:123-148`（`DeepImmutable<{...}>`）
> - 消费入口：`src/hooks/useCanUseTool.tsx:35`（`CanUseToolFn` 签名）
> - 包装写入：`src/hooks/toolPermission/PermissionContext.ts:102-146`（`applyPermissionUpdates` + `setToolPermissionContext`）
> - 子代理克隆：`src/utils/forkedAgent.ts:345-374`（`createSubagentContext`）
> - 工具描述取数：`src/hooks/useCanUseTool.tsx:64-69`
> - API 埋点取数：`src/services/api/claude.ts:1746` / `2859` / `3286-3344`
>
> 关联文档：
> - [`docs/09-query.md`](../09-query.md) —— 主循环如何把两个 context 串起来
> - [`docs/permission/30-permission-control-flow.md`](30-permission-control-flow.md) —— 权限决策完整链路
> - [`docs/permission/40-permission-system-design.md`](40-permission-system-design.md) —— 权限系统设计
> - [`docs/tool/42-runTools-vs-streaming-tool-executor.md`](../tool/42-runTools-vs-streaming-tool-executor.md) —— 同序列对比文

`ToolUseContext` 与 `ToolPermissionContext` 名字相近、嵌套使用，但**一个面向"工具调用如何被执行"的运行时环境，一个面向"工具调用是否被允许"的安全规则集合**。本文把这两个类型拆开，逐项对比：定义、字段、生命周期、可变性、谁拥有、谁消费、典型用例。

---

## 目录

1. [一句话定位](#1-一句话定位)
2. [定义位置与嵌套关系](#2-定义位置与嵌套关系)
3. [字段逐项对比](#3-字段逐项对比)
4. [可变性与生命周期](#4-可变性与生命周期)
5. [谁拥有 / 谁消费 / 谁替换](#5-谁拥有--谁消费--谁替换)
6. [典型调用路径](#6-典型调用路径)
7. [子代理的边界处理](#7-子代理的边界处理)
8. [常见混淆点](#8-常见混淆点)
9. [总结表](#9-总结表)

---

## 1. 一句话定位

| 类型 | 一句话 |
|---|---|
| `ToolUseContext` | **一次工具调用所能看见的全部运行时环境**：会话消息、abort 信号、UI 回调、subagent 标记、AppState 读写句柄等 |
| `ToolPermissionContext` | **当前生效的权限规则快照**：权限模式、allow/deny/ask 规则集、附加工作目录、bypass 是否可用等 |

> 一句话记忆法：**`ToolUseContext` 是"我能用什么"，`ToolPermissionContext` 是"我被允许做什么"**。

---

## 2. 定义位置与嵌套关系

两者都定义在 `src/Tool.ts`。`ToolUseContext` 是个**结构庞杂的可变对象**（140+ 行），`ToolPermissionContext` 是用 `DeepImmutable<{...}>` 包裹的**纯数据快照**。

关键嵌套：

```ts
// src/Tool.ts:123-148
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

`ToolUseContext` **不直接**嵌套 `ToolPermissionContext` 字段——它是**通过 `getAppState()` 间接访问**：

```ts
// src/Tool.ts:182-183
getAppState(): AppState
setAppState(f: (prev: AppState) => AppState): void
```

而 `AppState.toolPermissionContext` 才是真正承载权限快照的位置。换句话说：

```
ToolUseContext
  └─ getAppState() → AppState
                       └─ toolPermissionContext: ToolPermissionContext
```

两层间接，是为了**让权限策略独立于工具运行时**——权限变化时不影响 `ToolUseContext` 引用本身，权限消费者通过 `getAppState()` 拉最新值。

---

## 3. 字段逐项对比

### `ToolUseContext` 的字段（`src/Tool.ts:158-300`）

按用途分组：

| 分组 | 字段 | 用途 |
|---|---|---|
| **运行时配置** | `options.commands` / `debug` / `mainLoopModel` / `tools` / `verbose` / `thinkingConfig` / `mcpClients` / `mcpResources` / `isNonInteractiveSession` / `agentDefinitions` / `maxBudgetUsd` / `customSystemPrompt` / `appendSystemPrompt` / `querySource` / `refreshTools` | 来自 query 参数，工具可读的"环境快照" |
| **中止控制** | `abortController: AbortController` | 整 turn 取消的统一信号 |
| **会话历史** | `messages: Message[]` / `readFileState: FileStateCache` | 当前对话消息列表 + 文件读取缓存（防重复读） |
| **状态读写** | `getAppState()` / `setAppState()` / `setAppStateForTasks?` | AppState 的反应式读写（注意 subagent 下 `setAppState` 是 no-op） |
| **UI / OS 回调** | `setToolJSX?` / `addNotification?` / `appendSystemMessage?` / `sendOSNotification?` / `setStreamMode?` / `setSDKStatus?` / `openMessageSelector?` / `requestPrompt?` | 工具执行中向 REPL 输出内容、弹窗、通知 |
| **进度/遥测** | `setInProgressToolUseIDs` / `setHasInterruptibleToolInProgress?` / `setResponseLength` / `pushApiMetricsEntry?` / `onCompactProgress?` | 进度提示、遥测埋点 |
| **子代理标记** | `agentId?: AgentId` / `agentType?: string` / `toolUseId?: string` | 区分 subagent 调用，hook 据此走不同分支 |
| **缓存与一致性** | `contentReplacementState?` / `renderedSystemPrompt?` / `updateFileHistoryState` / `updateAttributionState` | fork 子代理的缓存共享 |
| **去重 / 追踪** | `nestedMemoryAttachmentTriggers?` / `loadedNestedMemoryPaths?` / `dynamicSkillDirTriggers?` / `discoveredSkillNames?` / `toolDecisions?` / `queryTracking?` | 各种防重与链路追踪 |
| **特殊钩子** | `handleElicitation?` / `userModified?` / `requireCanUseTool?` / `fileReadingLimits?` / `globLimits?` / `localDenialTracking?` / `preserveToolUseResults?` / `criticalSystemReminder_EXPERIMENTAL?` | 边界场景：MCP URL 弹窗、speculation 覆写、子代理拒绝计数等 |

**数量：约 30 个顶层字段、其中半数带 `?`。**

### `ToolPermissionContext` 的字段（`src/Tool.ts:123-138`）

| 字段 | 用途 |
|---|---|
| `mode: PermissionMode` | 当前权限模式（`default` / `acceptEdits` / `bypassPermissions` / `plan` / `auto` 等） |
| `additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>` | 用户在 REPL 用 `/directories` 或 `--add-dir` 添加的目录白名单 |
| `alwaysAllowRules: ToolPermissionRulesBySource` | 始终允许的规则（按来源分组：user / project / session / flag） |
| `alwaysDenyRules: ToolPermissionRulesBySource` | 始终拒绝的规则 |
| `alwaysAskRules: ToolPermissionRulesBySource` | 始终弹窗询问的规则 |
| `isBypassPermissionsModeAvailable: boolean` | bypass 模式在当前环境下是否可用 |
| `isAutoModeAvailable?: boolean` | 自动模式（基于 classifier）是否可用 |
| `strippedDangerousRules?: ToolPermissionRulesBySource` | 危险规则剥离集（防止 auto 模式误用） |
| `shouldAvoidPermissionPrompts?: boolean` | **关键标志位**：当前 subagent 应当避免弹权限框（如后台任务） |
| `awaitAutomatedChecksBeforeDialog?: boolean` | 弹权限框前先等自动检查（coordinator worker 场景） |
| `prePlanMode?: PermissionMode` | 模型自主进入 plan 模式前，缓存原模式以便退出时恢复 |

**数量：11 个字段。**

### 字段数量与复杂度的差异

- `ToolUseContext` **结构大、字段多**——它是工具执行所需的全部环境快照；
- `ToolPermissionContext` **结构小、字段少**——只承载"当前权限状态"的必要数据；
- 共同点：两者都**不存直接的应用业务状态**（如 TodoWrite 状态），业务状态走 AppState 自己的 slice。

---

## 4. 可变性与生命周期

| 维度 | `ToolUseContext` | `ToolPermissionContext` |
|---|---|---|
| **类型系统可变性** | 字段大多**可读写**，少数回调只读 | 整个对象 `DeepImmutable<{...}>` —— **TS 层禁止 mutate** |
| **运行时更新方式** | 字段级直读直写 / `setAppState(f)` 触发反应式更新 | **整对象替换**：`applyPermissionUpdates(prev, updates)` → `setToolPermissionContext(newContext)` |
| **谁负责替换** | `setAppState(f)` 是 React 风格的 reducer | `setToolPermissionContext`（AppStore 上的 setter）+ `applyPermissionUpdates` 工具函数（`src/hooks/toolPermission/PermissionContext.ts:143-145`） |
| **典型更新时机** | 工具执行中向 `messages` push / `setInProgressToolUseIDs` | 用户在 `/permissions` 切换模式、添加工作目录、调整规则 |
| **轮次内是否变化** | **会**（同一 turn 内 `messages` 追加、`setInProgressToolUseIDs` 状态变更） | **会**（同一 turn 内用户可能临时改 mode），但**整对象替换**、引用必变 |
| **跨 turn 是否变化** | 主循环每轮重写 `toolUseContext`（见 `src/query.ts:361-364`） | 取决于用户是否操作 `/permissions`；跨 turn 一般不变 |

> 核心：`ToolPermissionContext` 是**纯数据快照**，`ToolUseContext` 是**带回调的执行环境**。

---

## 5. 谁拥有 / 谁消费 / 谁替换

### `ToolUseContext`

| 角色 | 位置 |
|---|---|
| **构造方** | `src/QueryEngine.ts`（SDK 入口）+ `src/screens/REPL.tsx`（REPL 入口）+ `src/utils/forkedAgent.ts:345`（subagent fork） |
| **所有者** | 持有者是一次 query 调用方，**不存进 AppState**，只在 `query.ts` 局部流转 |
| **消费者** | `toolExecution.ts` 的 `runToolUse`、`StreamingToolExecutor`、`runTools`、`useCanUseTool`、所有 `tool.call()` 实现 |
| **替换方** | 主循环每轮基于 `state` 重建（`src/query.ts:361-364`），fork 子代理时基于父 context 克隆（`src/utils/forkedAgent.ts:345`） |

### `ToolPermissionContext`

| 角色 | 位置 |
|---|---|
| **构造方** | `getEmptyToolPermissionContext()`（`src/Tool.ts:140-148`） + `createDisabledBypassPermissionsContext()`（`src/state/AppState.tsx:114`） |
| **所有者** | **AppState 的一片**（`AppState.toolPermissionContext`），全局唯一 |
| **消费者** | `hasPermissionsToUseTool`（规则匹配，权限决策）、`useCanUseTool`（弹框/拒绝）、`tool.description(input, { toolPermissionContext, ... })`（生成 UI 描述）、`claude.ts:1746/2859`（埋点） |
| **替换方** | `setToolPermissionContext` + `applyPermissionUpdates`（`src/hooks/toolPermission/PermissionContext.ts:143-145`），`/permissions` 命令、`createDisabledBypassPermissionsContext`（remote settings 加载时） |

---

## 6. 典型调用路径

### 6.1 权限决策路径（涉及两者）

```
queryLoop
  └─ for await (const message of deps.callModel(...))   // src/query.ts:660
        └─ options.getToolPermissionContext()            // src/services/api/claude.ts:1746
              → toolUseContext.getAppState().toolPermissionContext
                  (仅用于埋点记录，不做决策)
  ...
  └─ toolUseBlocks → runToolUse / StreamingToolExecutor
        └─ runToolUse → permission phase
              └─ hasPermissionsToUseTool(...)             // src/utils/permissions/permissions.ts
                    ├─ 读 toolUseContext.getAppState().toolPermissionContext  (决策依据)
                    └─ 读 toolUseContext.options.tools   (工具元数据)
              └─ useCanUseTool(...)                       // src/hooks/useCanUseTool.tsx:35
                    ├─ 取 toolUseContext.getAppState().toolPermissionContext
                    ├─ 调 tool.description(input, { toolPermissionContext: appState.toolPermissionContext, ... })
                    └─ 根据 mode/rules 决定 ask/allow/deny
```

注意 `options.getToolPermissionContext()` 是**给 API 层埋点用的**，不是权限决策入口——真正决策走 `useCanUseTool`。

### 6.2 工具描述生成路径（只读 ToolPermissionContext 的子集）

```ts
// src/hooks/useCanUseTool.tsx:64-69
const description = await tool.description(input as never, {
  isNonInteractiveSession: toolUseContext.options.isNonInteractiveSession,
  toolPermissionContext: appState.toolPermissionContext,
  tools: toolUseContext.options.tools
})
```

工具的 `description()` 只接收**只读子集**——不暴露整个 `ToolUseContext`。这避免了工具实现依赖不稳定的执行环境字段。

### 6.3 权限规则变更路径（替换 ToolPermissionContext）

```
/permissions 命令
  └─ 修改规则集（add/remove rule）
  └─ applyPermissionUpdates(prev, updates)               // src/hooks/toolPermission/PermissionContext.ts:143
  └─ setToolPermissionContext(newContext)                // AppState setter
        → AppState.toolPermissionContext 整体被新对象替换
        → useCanUseTool 后续取的都是新值
```

注意是**整对象替换**——内部 `Map`/`Record` 都新建，引用比较直接判定变化。

---

## 7. 子代理的边界处理

这是两者关系中**最微妙**的地方。`createSubagentContext`（`src/utils/forkedAgent.ts:345-374`）生成子代理的 `ToolUseContext`，但它会**改写 `getAppState()` 包装**：

```ts
// src/utils/forkedAgent.ts:357-374
const getAppState: ToolUseContext['getAppState'] = overrides?.getAppState
  ? overrides.getAppState
  : overrides?.shareAbortController
    ? parentContext.getAppState
    : () => {
        const state = parentContext.getAppState()
        if (state.toolPermissionContext.shouldAvoidPermissionPrompts) {
          return state
        }
        return {
          ...state,
          toolPermissionContext: {
            ...state.toolPermissionContext,
            shouldAvoidPermissionPrompts: true,
          },
        }
      }
```

含义：

1. **子代理不直接拿到父 `getAppState`**——而是包装后的版本；
2. 包装内**强制注入 `shouldAvoidPermissionPrompts: true`**——子代理不会弹权限框，避免后台任务卡住；
3. `shouldAvoidPermissionPrompts` 已经在原 context 是 true 时**直接复用**原对象，避免无谓重建；
4. `shareAbortController: true`（交互型 agent）则共享父 `getAppState`，因为交互型 agent 需要正常弹框。

> 这条封装路径说明：**子代理看到的"权限环境"和父进程不一样**，是由 `ToolUseContext` 的 `getAppState` 包装层注入的，而**不是直接修改 `ToolPermissionContext` 对象本身**——后者是 `DeepImmutable`，TS 层禁止这种就地修改。

---

## 8. 常见混淆点

### 混淆 1：`canUseTool(tool, input, toolUseContext, ...)` 的入参

```ts
// src/hooks/useCanUseTool.tsx:35
export type CanUseToolFn = (
  tool: ToolType,
  input: Record<string, unknown>,
  toolUseContext: ToolUseContext,   // ← 拿到的是整个 ToolUseContext，不是 ToolPermissionContext
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: PermissionDecision<Input>,
) => Promise<PermissionDecision<Input>>
```

权限决策回调拿到的是 `ToolUseContext`，需要自己 `getAppState().toolPermissionContext` 取权限快照——**入参里没有现成的 `ToolPermissionContext`**。

### 混淆 2：`tool.description()` 的入参

```ts
// src/Tool.ts:389-393
description(
  input: ...,
  options: {
    isNonInteractiveSession: boolean
    toolPermissionContext: ToolPermissionContext
    tools: Tools
  }
): Promise<string>
```

工具实现拿到的 `toolPermissionContext` 是**只读快照**——不能调 setter。所以工具描述里如果想根据 mode 调整文案，直接读 `mode` 字段即可。

### 混淆 3：`getToolPermissionContext()` 是 API 埋点用，不是权限决策用

```ts
// src/services/api/claude.ts:667-670、1746、2859、3286-3344
async getToolPermissionContext() {
  const appState = toolUseContext.getAppState()
  return appState.toolPermissionContext
}
```

这个回调只在 API 调用前后**打日志/埋点**，告诉后端当前权限模式。它**不参与**权限决策——决策走 `useCanUseTool`。

### 混淆 4：AppState 不是 ToolUseContext

`AppState` 是全局 Zustand store；`ToolUseContext` 是一次 query 调用的局部环境。两者通过 `getAppState()` 桥接。

---

## 9. 总结表

| 维度 | `ToolUseContext` | `ToolPermissionContext` |
|---|---|---|
| **定位** | 工具执行的运行时环境 | 工具执行的权限规则快照 |
| **字段数量** | ~30 个顶层字段 | 11 个 |
| **可变性** | 字段可读写 | `DeepImmutable`，只能整对象替换 |
| **典型更新方式** | 字段直写 / `setAppState(f)` | `applyPermissionUpdates` + `setToolPermissionContext` |
| **存哪里** | 局部变量，在 `query.ts` / `forkedAgent.ts` 流转 | `AppState.toolPermissionContext` |
| **谁拥有** | 一次 query 调用方 | AppState store（全局唯一） |
| **谁消费** | 工具本身、`runToolUse`、权限决策、UI 回调 | 权限决策、API 埋点、工具描述生成 |
| **子代理处理** | `createSubagentContext` 克隆 + `getAppState` 包装 | 通过 `getAppState` 包装注入 `shouldAvoidPermissionPrompts=true` |
| **替换成本** | 整对象浅克隆 | 整对象替换（deep clone，因为 DeepImmutable） |
| **使用频率** | 每个工具调用都用到 | 每个权限决策点用到 |

简而言之：**`ToolUseContext` 装"执行工具需要的全部环境"，`ToolPermissionContext` 装"当前是否允许这个工具"**。前者是大而全的执行容器，后者是小而纯的策略快照。前者通过 `getAppState()` 间接引用后者，因此权限策略可以独立变更而不影响工具运行时。