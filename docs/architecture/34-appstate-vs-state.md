# AppState 与 State 对比分析：`src/state/AppStateStore.ts` vs `src/bootstrap/state.ts`

本文档深入分析 Claude Code 中两个状态管理系统：**AppState**（应用状态） 和 **State**（全局会话状态），并详细比较它们的异同。

---

## 一、整体定位

Claude Code 项目中存在两个**看似相似但定位完全不同**的状态管理系统：

| 维度 | `bootstrap/state.ts` | `state/AppStateStore.ts` |
| --- | --- | --- |
| **核心类** | `STATE`（全局单例）+ getter/setter | `AppState` + `createStore`（观察者模式） |
| **存储位置** | 模块顶层变量（私有） | 通过 `Store<T>` 抽象的实例 |
| **设计目标** | DAG 叶子节点，最小依赖 | React 友好，支持订阅通知 |
| **访问模式** | 函数式（getter/setter） | 函数式 + 响应式（subscribe） |
| **使用场景** | 跨模块的基础状态、计数器、ID | UI 状态、用户偏好、业务逻辑 |
| **响应性** | 无（直接读写） | 有（`subscribe()` + `setState()` 通知） |
| **不可变性** | 大部分字段可变 | 默认 `DeepImmutable` 包装 |

---

## 二、`bootstrap/state.ts` — 全局会话状态

### 2.1 核心特征

```ts
// 注释明确警告：DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
```

**特点**：
- 是一个**叶子节点**（leaf），只依赖 `src/utils/`、`src/types/`、`src/entrypoints/` 内的类型
- 不被任何 `src/utils/` 依赖
- 所有字段都是模块级变量，通过大量 getter/setter 函数访问

### 2.2 State 的字段分类（90+ 字段）

按职能划分为 8 大族群：

#### (1) 标识与路径（5 个字段）
- `originalCwd` / `projectRoot` / `cwd` / `sessionId` / `parentSessionId`

#### (2) 成本与配额（11 个字段）
- `totalCostUSD`、`totalAPIDuration`、`turnHookDurationMs` 等

#### (3) 模型与模式（6 个字段）
- `mainLoopModelOverride`、`initialMainLoopModel`、`isInteractive`、`kairosActive` 等

#### (4) Telemetry / Logger / Tracer（10 个字段）
- `meter`、`sessionCounter`、`locCounter`、`statsStore` 等

#### (5) 权限与信任（5 个字段）
- `sessionBypassPermissionsMode`、`sessionTrustAccepted`、`sessionPersistenceDisabled` 等

#### (6) 模式切换附件（4 个字段）
- `needsPlanModeExitAttachment`、`needsAutoModeExitAttachment`

#### (7) 提示词缓存 sticky-latch（4 个字段）
- `afkModeHeaderLatched`、`fastModeHeaderLatched`、`cacheEditingHeaderLatched`、`thinkingClearLatched`

#### (8) 其它（lifecycle/hook/cron/cache/compact）
- `sessionCreatedTeams`、`sessionCronTasks`、`registeredHooks`、`invokedSkills` 等

### 2.3 典型使用模式

```ts
// 获取 sessionId
const sessionId = getSessionId()

// 切换 session（原子操作）
switchSession(newId, projectDir)

// 增加成本
addToTotalCostState(cost, modelUsage, model)

// 获取当前 token 用量
const tokens = getTotalOutputTokens()
```

### 2.4 latch 字段（特殊的不可回退语义）

```ts
// 粘性 latch：一旦置 true 永不回退
afkModeHeaderLatched: boolean | null  // null=未评估, true=已锁定

// 服务端 prompt cache 约 50-70K token，TTL ~5 分钟
// 中途关闭会让下一次请求 bust cache，反而更慢
```

**为什么是 tri-state（`null` / `true`）而非 `boolean`？**
- `null` = 尚未评估（可以在评估后决定开启或关闭）
- `true` = 已锁定开启（永不回退，避免 bust cache）
- 这避免了"中途回退"的语义陷阱

### 2.5 测试隔离机制

```ts
export function resetStateForTests(): void {
    if (process.env.NODE_ENV !== 'test') {
        throw new Error('resetStateForTests can only be used in tests')
    }
    Object.entries(getInitialState()).forEach(([key, value]) => {
        STATE[key as keyof State] = value as never
    })
    // ...
}
```

- 防御性 `if (NODE_ENV !== 'test')` throw 防止生产环境误用
- 用 `Object.entries(getInitialState())` 自动覆盖所有新字段

---

## 三、`state/AppStateStore.ts` — 应用业务状态

### 3.1 核心数据结构

#### (1) Store 抽象（基于观察者模式）

```ts
// src/state/store.ts
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
    getState: () => T
    setState: (updater: (prev: T) => T) => void
    subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
    initialState: T,
    onChange?: OnChange<T>,
): Store<T> {
    let state = initialState
    const listeners = new Set<Listener>()

    return {
        getState: () => state,
        setState: (updater) => {
            const prev = state
            const next = updater(prev)
            if (Object.is(next, prev)) return  // 优化：相同引用不触发
            state = next
            onChange?.({newState: next, oldState: prev})
            for (const listener of listeners) listener()
        },
        subscribe: (listener) => {
            listeners.add(listener)
            return () => listeners.delete(listener)
        },
    }
}
```

**关键设计**：
- `Object.is(next, prev)` 短路：如果 `setState` 返回相同引用，跳过通知（性能优化）
- 不可变更新模式：`setState(prev => next)` 而不是直接修改
- 订阅返回 unsubscribe 函数（类似 EventEmitter）

#### (2) AppState 类型

```ts
export type AppState = DeepImmutable<{
    settings: SettingsJson
    verbose: boolean
    mainLoopModel: ModelSetting
    toolPermissionContext: ToolPermissionContext
    agent: string | undefined
    kairosEnabled: boolean
    remoteSessionUrl: string | undefined
    replBridgeEnabled: boolean
    // ... 50+ 字段
}> & {
    // 不被 DeepImmutable 包装的部分（包含函数类型）
    tasks: { [taskId: string]: TaskState }
    agentNameRegistry: Map<string, AgentId>
    mcp: { clients, tools, commands, resources, pluginReconnectKey }
    plugins: { enabled, disabled, commands, errors, ... }
    // ...
}
```

### 3.2 AppState 的字段分类

#### (1) 配置与偏好（10 个字段）
- `settings`、`verbose`、`mainLoopModel`、`mainLoopModelForSession`

#### (2) UI 状态（10 个字段）
- `expandedView`、`isBriefOnly`、`selectedIPAgentIndex`、`coordinatorTaskIndex`、`viewSelectionMode`、`footerSelection`

#### (3) 远程会话与 Bridge（20+ 字段）
- `remoteSessionUrl`、`remoteConnectionStatus`、`replBridgeEnabled`、`replBridgeConnected`、`replBridgeSessionActive` 等

#### (4) MCP 服务器（5 个字段，结构化）
- `mcp.clients`、`mcp.tools`、`mcp.commands`、`mcp.resources`、`mcp.pluginReconnectKey`

#### (5) 插件系统（7 个字段，结构化）
- `plugins.enabled`、`plugins.disabled`、`plugins.errors`、`plugins.installationStatus`、`plugins.needsRefresh`

#### (6) 任务与文件（4 个字段）
- `tasks`、`foregroundedTaskId`、`viewingAgentTaskId`、`fileHistory`

#### (7) 团队协作（3 个字段）
- `teamContext`、`standaloneAgentContext`、`inbox`

#### (8) 提示与推测（5 个字段）
- `promptSuggestion`、`speculation`、`speculationSessionTimeSavedMs`、`skillImprovement`

#### (9) 流程控制（5 个字段）
- `initialMessage`、`pendingPlanVerification`、`denialTracking`、`activeOverlays`、`fastMode`

### 3.3 默认值工厂

```ts
export function getDefaultAppState(): AppState {
    // Teammate 启动时检查是否需要 plan_mode_required
    const teammateUtils = require('../utils/teammate.js')
    const initialMode: PermissionMode =
        teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired()
            ? 'plan'
            : 'default'

    return {
        settings: getInitialSettings(),
        tasks: {},
        agentNameRegistry: new Map(),
        // ... 50+ 字段
    }
}
```

### 3.4 典型使用模式

```ts
// 1. 创建 store
const store = createStore<AppState>(getDefaultAppState())

// 2. 订阅变化（React 组件会用 useSyncExternalStore）
const unsubscribe = store.subscribe(() => {
    // 重新渲染
})

// 3. 更新状态（不可变更新）
store.setState(prev => ({
    ...prev,
    expandedView: 'tasks',
}))

// 4. 读取状态
const state = store.getState()
```

---

## 四、AppState 在流程控制中的核心作用

### 4.1 通过 `setAppState` 控制业务流程

AppState 是**业务逻辑的中央控制点**，多个核心流程通过它进行协调：

#### (1) 权限模式切换
```ts
// 状态字段：toolPermissionContext.mode
setAppState(prev => ({
    ...prev,
    toolPermissionContext: {
        ...prev.toolPermissionContext,
        mode: 'bypassPermissions',
    }
}))
```

**影响**：工具执行时，`hasPermissionsToUseTool()` 会读取这个字段决定是否绕过权限检查。

#### (2) 会话切换
```ts
// 状态字段：agent, kairosEnabled, remoteSessionUrl
setAppState(prev => ({
    ...prev,
    agent: 'custom-agent',
    kairosEnabled: true,
}))
```

**影响**：影响系统提示词、工具权限、UI 显示。

#### (3) UI 视图控制
```ts
// 状态字段：expandedView, viewSelectionMode, footerSelection
setAppState(prev => ({
    ...prev,
    expandedView: 'tasks',
    selectedIPAgentIndex: 2,
}))
```

**影响**：REPL 渲染时根据这些字段决定显示哪个面板。

#### (4) 任务生命周期
```ts
// 状态字段：tasks, foregroundedTaskId, viewingAgentTaskId
setAppState(prev => ({
    ...prev,
    tasks: {
        ...prev.tasks,
        [taskId]: newTaskState,
    },
    foregroundedTaskId: taskId,
}))
```

**影响**：控制任务在 UI 中的显示状态，是否前台运行。

#### (5) 插件热重载
```ts
// 状态字段：mcp.pluginReconnectKey
setAppState(prev => ({
    ...prev,
    mcp: {
        ...prev.mcp,
        pluginReconnectKey: prev.mcp.pluginReconnectKey + 1,
    }
}))
```

**影响**：MCP 连接的 effects 读取这个值作为依赖触发重连。

### 4.2 通过订阅实现响应式流程

AppState 的 Store 抽象支持响应式更新：

```ts
// React 组件中（通过 useSyncExternalStore）
const state = useSyncExternalStore(
    store.subscribe,
    () => store.getState()
)

// 普通模块中
const unsubscribe = store.subscribe(() => {
    const newState = store.getState()
    // 处理状态变化
})
```

**典型流程**：
1. 用户在 UI 中操作（如切换权限模式）
2. 业务逻辑调用 `setAppState(...)`
3. Store 通知所有订阅者
4. React 组件重新渲染
5. UI 显示新状态

### 4.3 关键流程控制字段详解

#### (1) `initialMessage` - 启动消息队列

```ts
initialMessage: {
    message: UserMessage
    clearContext?: boolean
    mode?: PermissionMode
    allowedPrompts?: AllowedPrompt[]
} | null
```

**作用**：
- 在 REPL 空闲时，如果有 `initialMessage`，REPL 会处理它
- 支持从 CLI 参数或 plan mode 退出时触发的初始消息
- `clearContext` 标志表示是否清空上下文

#### (2) `pendingPlanVerification` - 计划验证状态

```ts
pendingPlanVerification?: {
    plan: string
    verificationStarted: boolean
    verificationCompleted: boolean
}
```

**作用**：退出 plan mode 后触发后台验证，跟踪验证进度。

#### (3) `denialTracking` - 拒绝追踪

```ts
denialTracking?: DenialTrackingState
```

**作用**：YOLO / headless 模式下追踪工具拒绝次数，超过阈值回退到弹窗询问。

#### (4) `activeOverlays` - 活动浮层

```ts
activeOverlays: ReadonlySet<string>
```

**作用**：追踪当前活动的浮层（Select 对话框等），用于 Escape 键协调。

#### (5) `fastMode` - 快速模式

```ts
fastMode?: boolean
```

**作用**：启用/禁用快速模式，影响请求头 beta。

#### (6) `effortValue` - 推理强度

```ts
effortValue?: EffortValue
```

**作用**：控制模型推理强度。

#### (7) `speculation` - 推测执行

```ts
speculation: SpeculationState
```

**作用**：追踪推测执行状态（如用户输入时的预测回复）。

---

## 五、两者的核心差异对比

### 5.1 设计哲学差异

| 维度 | `bootstrap/state.ts` | `state/AppStateStore.ts` |
| --- | --- | --- |
| **设计目的** | 跨模块基础状态的共享存储 | 业务逻辑的中央状态机 |
| **依赖关系** | DAG 叶子，不被 `utils/` 依赖 | 可依赖其他模块 |
| **响应性** | 直接读写，无订阅 | 观察者模式，支持订阅 |
| **访问方式** | 函数式 getter/setter | store.getState()/setState() + 订阅 |
| **数据特性** | 跨进程的基础状态、计数器 | UI 状态、用户偏好、业务配置 |
| **持久化** | 部分字段持久化（如 `sessionId`） | 不持久化，进程级 |
| **测试隔离** | `resetStateForTests` 自动覆盖 | `getDefaultAppState()` 重建 |

### 5.2 字段数量与复杂度

| 维度 | `bootstrap/state.ts` | `state/AppStateStore.ts` |
| --- | --- | --- |
| 字段数量 | 90+ | 50+ |
| 嵌套结构 | 浅（大部分是扁平字段） | 深（`mcp`、`plugins`、`teamContext` 等嵌套结构） |
| 函数类型字段 | 极少（除了 `Map`、`Set`） | 有（`replContext` 中的 console 方法等） |
| 特殊类型 | `Map`、`Set`（用于 cache、latch 等） | `Map`、`Set`、`ReadonlySet`（用于注册表） |

### 5.3 触发更新的方式差异

```ts
// bootstrap/state.ts: 直接调用 setter
setMainLoopModelOverride('claude-opus-4-8')
addToTotalCostState(cost, modelUsage, model)

// state/AppStateStore.ts: 通过 setState 不可变更新
store.setState(prev => ({
    ...prev,
    mainLoopModel: 'claude-opus-4-8',
}))
```

### 5.4 触发通知的方式差异

```ts
// bootstrap/state.ts: 不通知，需要手动检查
setMainLoopModelOverride('claude-opus-4-8')
// 其它模块需要主动调用 getMainLoopModel() 才能知道变化

// state/AppStateStore.ts: 自动通知所有订阅者
store.setState(prev => ({...prev, mainLoopModel: 'claude-opus-4-8'}))
// 所有 store.subscribe() 注册的回调会自动被调用
```

### 5.5 与子系统的关系

| 子系统 | 依赖 | 数据流向 |
| --- | --- | --- |
| 成本追踪 | `bootstrap/state.ts` (写) → 报表生成器 (读) | 单向 |
| REPL UI | `state/AppStateStore.ts` (读写) → React 渲染 | 双向响应式 |
| 工具权限 | `state/AppStateStore.ts` (读) → 工具执行 | 单向读取 |
| Hook 系统 | `bootstrap/state.ts` (注册表) | 注册 + 触发 |
| Telemetry | `bootstrap/state.ts` (计数器) → 远端 | 单向聚合 |

---

## 六、两者的协作关系

虽然 AppState 和 State 是两个独立的系统，但它们经常协同工作：

### 6.1 启动时的初始化流程

```ts
// main.tsx 中
async function init() {
    // 1. 初始化 bootstrap/state.ts（基础会话状态）
    setSessionId(randomUUID())
    setCwdState(process.cwd())
    setIsInteractive(true)
    
    // 2. 初始化 AppState（业务状态）
    const appStateStore = createStore<AppState>(getDefaultAppState())
    
    // 3. 把 AppState store 注入到全局
    setAppStateStore(appStateStore)
    
    // 4. 在 QueryEngine 初始化时传入
    const engine = new QueryEngine({
        getAppState: () => appStateStore.getState(),
        setAppState: (fn) => appStateStore.setState(fn),
        // ...
    })
}
```

### 6.2 在 QueryEngine 中的双重使用

```ts
// src/QueryEngine.ts
class QueryEngine {
    constructor(config) {
        // AppState 通过 getAppState/setAppState 注入
        this.getAppState = config.getAppState
        this.setAppState = config.setAppState
    }
    
    // bootstrap/state.ts 中的全局函数
    async submitMessage(prompt) {
        // ... 读取 sessionId（bootstrap）
        const sessionId = getSessionId()
        
        // ... 读取 AppState（store）
        const appState = this.getAppState()
    }
}
```

### 6.3 信息流的角色分工

```
┌─────────────────────────────────────────────────┐
│ bootstrap/state.ts (全局单例)                    │
│ - sessionId / cwd / cost / metrics              │
│ - 跨进程持久化的关键信息                          │
│ - 所有模块可访问                                  │
└─────────────────────────────────────────────────┘
                       ↑
                       │ 读取
                       │
┌─────────────────────────────────────────────────┐
│ state/AppStateStore.ts (Store 实例)             │
│ - toolPermissionContext / settings / UI state  │
│ - 业务逻辑状态                                    │
│ - 支持订阅通知                                    │
└─────────────────────────────────────────────────┘
                       ↓
                  UI Components
                  （React 渲染）
```

---

## 七、具体使用场景对比

### 7.1 何时使用 bootstrap/state.ts

**适合**：
- 跨模块的基础数据（sessionId、cwd、cost 累计）
- 不需要响应式更新的数据
- 跨进程持久化的状态
- 计数器、统计指标
- 缓存和 latch（性能优化）

**示例**：
```ts
// 工具执行时读取
const totalCost = getTotalCostUSD()
if (totalCost > maxBudget) {
    // 触发预算超限
}

// 获取 sessionId 写入文件
const sessionId = getSessionId()
await saveToFile(`session-${sessionId}.json`, content)
```

### 7.2 何时使用 AppState

**适合**：
- UI 状态（视图模式、选中项、面板显示）
- 业务配置（权限模式、模型选择、thinking 开关）
- 用户偏好（verbose 模式、输出格式）
- 多个组件需要协同的状态

**示例**：
```ts
// 用户切换权限模式
setAppState(prev => ({
    ...prev,
    toolPermissionContext: {
        ...prev.toolPermissionContext,
        mode: 'bypassPermissions',
    }
}))

// React 组件自动重新渲染
const state = useSyncExternalStore(
    store.subscribe,
    () => store.getState()
)
```

### 7.3 决策树

```
需要存储状态？
├── 是 → 数据是否需要响应式更新？
│   ├── 是 → 使用 AppState（Store）
│   └── 否 → 数据是否跨模块共享？
│       ├── 是 → 基础数据？ → 使用 bootstrap/state.ts
│       │        业务数据？ → 使用 AppState（虽然不需要响应式）
│       └── 否 → 使用模块内的局部变量
└── 否 → 使用局部变量
```

---

## 八、底层设计原理

### 8.1 为什么 bootstrap/state.ts 不用 Store？

**原因**：
1. **DAG 叶子约束**：`bootstrap/state.ts` 不被 `src/utils/` 依赖。如果使用 Store，需要引入观察者模式，但这会增加包的复杂度
2. **简单访问模式**：90% 的字段是简单读取，不需要订阅
3. **性能优化**：直接读取比 Store.getState() 调用更高效
4. **类型安全**：getter/setter 函数签名更明确

### 8.2 为什么 AppState 不用模块单例？

**原因**：
1. **测试隔离**：AppState 是实例级的，每个 QueryEngine 可以有独立的 store
2. **响应式需求**：UI 组件需要订阅变化
3. **不可变性**：AppState 默认 `DeepImmutable`，避免意外突变
4. **集成测试**：可以在测试中创建独立的 store 进行隔离测试

### 8.3 Store 的不可变性设计

```ts
export type AppState = DeepImmutable<{
    // 大部分字段
    settings: SettingsJson
    // ...
}> & {
    // 不被 DeepImmutable 包装的部分
    tasks: { [taskId: string]: TaskState }  // TaskState 包含函数类型
}
```

**为什么混合使用？**
- `DeepImmutable` 保证大部分字段不可变
- 含有函数类型的字段（如 `TaskState.console`、`REPLHookContext`）不能被 DeepImmutable 处理，需要单独列出
- 这种设计在保证类型安全的同时，保留必要的灵活性

---

## 九、最佳实践

### 9.1 使用 bootstrap/state.ts 的最佳实践

```ts
// ✅ 正确：使用 getter/setter 函数
const sessionId = getSessionId()
addToTotalCostState(cost, modelUsage, model)

// ❌ 错误：直接访问 STATE（不可访问，私有）
// STATE.sessionId  // 编译错误
```

### 9.2 使用 AppState 的最佳实践

```ts
// ✅ 正确：不可变更新
store.setState(prev => ({
    ...prev,
    settings: newSettings,
}))

// ❌ 错误：直接修改（违反不可变性）
store.setState(prev => {
    prev.settings = newSettings  // TS 错误
    return prev
})
```

### 9.3 何时选择哪个系统

```
决策流程：

1. 数据是否需要 React 组件订阅？
   → 是：AppState
   
2. 数据是否是跨模块基础状态（sessionId、cost、cwd）？
   → 是：bootstrap/state.ts
   
3. 数据是否需要复杂的不可变性保证？
   → 是：AppState
   
4. 数据是否需要 latch 语义（永不回退）？
   → 是：bootstrap/state.ts（使用 `boolean | null` 类型）
   
5. 默认：bootstrap/state.ts（更轻量）
```

---

## 十、文件依赖关系

```
src/bootstrap/state.ts (DAG 叶子)
├── 不被 src/utils/ 依赖
├── 只被业务模块依赖
└── 提供 90+ getter/setter 函数

src/state/AppStateStore.ts (依赖较多)
├── 依赖 src/Tool.ts, src/utils/* 等
├── 被 QueryEngine, REPL, sdk 等依赖
└── 导出 AppState 类型 + getDefaultAppState() 函数

src/state/store.ts (Store 抽象)
├── 极简实现（35 行）
├── 不依赖任何业务代码
└── 可独立使用
```

---

## 十一、未来扩展考虑

### 11.1 bootstrap/state.ts 的限制

随着字段增多（已达 90+），维护成本上升。可能的演进方向：
- 拆分：按子系统（session、cost、permissions 等）拆分为多个文件
- 类型化：使用更严格的类型（如 `Brand<string>`）
- 移除：把不常访问的字段移到各自的模块

### 11.2 AppState 的扩展空间

- **持久化**：当前 AppState 不持久化，可以考虑持久化关键字段（如偏好设置）
- **中间件**：扩展 `createStore` 支持中间件（类似 Redux）
- **DevTools**：添加时间旅行调试支持

---

## 十二、总结

| 维度 | `bootstrap/state.ts` | `state/AppStateStore.ts` |
| --- | --- | --- |
| 定位 | 全局会话状态 | 应用业务状态 |
| 设计模式 | 单例 + 函数式 | 观察者模式（Store） |
| 响应性 | 无 | 有（subscribe） |
| 主要场景 | 基础数据、计数器、缓存 | UI 状态、业务配置、用户偏好 |
| 复杂度 | 90+ 字段，扁平结构 | 50+ 字段，含嵌套结构 |
| 测试隔离 | `resetStateForTests` | 创建独立 store |
| 适用决策 | 跨模块基础共享数据 | 需要响应式更新的状态 |

**核心洞察**：
- **bootstrap/state.ts** 是"低级基础状态"，强调**简单性**和**DAG 叶子地位**
- **state/AppStateStore.ts** 是"高级业务状态"，强调**响应性**和**可观测性**
- 两者通过**职责清晰划分**避免了状态混乱，是一个优秀的状态分层设计

理解这两个系统的异同，对于理解 Claude Code 的架构至关重要。