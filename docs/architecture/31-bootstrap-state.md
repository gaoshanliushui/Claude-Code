# `bootstrap/state.ts`：全局 Session 状态中心

> 本文聚焦 `src/bootstrap/state.ts`，解释它如何用一个 **类型化的全局变量 `STATE`** 承载 CLI 的会话期可变状态，并说明 State 中的每一族变量如何在流程中充当开关、计数器、缓存、sticky latch。

---

## 1. 模块定位

`src/bootstrap/state.ts` 是整个 `src/` 导入 DAG 的 **叶子节点（leaf）**——它**只依赖 `src/utils/`、`src/types/`、`src/entrypoints/` 内的 type** 与 Node 标准库，**不被任何 `src/utils/` 依赖**。这种"反向"的依赖关系保证了：

- 任何工具 / Hook / 服务模块都可以读 / 写全局状态；
- 状态模块本身不会因为引入了上游模块而陷入循环导入。

```ts
// 仅允许的“出边”方向
src/bootstrap/state.ts ─┬─▶ src/utils/{crypto, signal, settings, model/…}
                       ├─▶ src/types/{ids, hooks}
                       └─▶ src/entrypoints/agentSdkTypes (type)
```

`state.ts` 文件顶部还有一条强约束：

```ts
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
```

——它的存在意味着：开发者需要在加新全局变量与把它挪到对应模块私有变量之间权衡，倾向后者。

---

## 2. 核心数据结构：`State` 类型

`State` 是一个 **90+ 字段** 的全局结构。文件里所有 getter/setter 都是薄包装（thin wrapper），访问代码不直接读 `STATE.xxx`，而是通过 `getXxx()` / `setXxx()`，这样：

1. 可在 getter 中插入缓存、副本、副作用（如 `setUseCoworkPlugins` 同时调用 `resetSettingsCache()`）；
2. 测试可通过 `Object.entries(getInitialState())` 完全重置；
3. 类型系统约束取值边界（如 `getIsNonInteractiveSession()` 返回 `!STATE.isInteractive`）。

字段按职能划分为 8 个族群（见 §3）。

---

## 3. State 的八族变量

### 3.1 标识 & 路径（5 个字段）

| 字段 | 作用 |
| --- | --- |
| `originalCwd` | 进程启动时 `realpathSync` 解析后的 CWD，整个会话**不再变更**。文件操作去路径前缀、session 落盘都看它。 |
| `projectRoot` | 与 `originalCwd` 同源，**只在 `--worktree` 启动时被改写**；运行时 `EnterWorktreeTool` 不会改它——保证 skills / history / 旧 transcript 不会随临时 worktree 漂移。 |
| `cwd` | 运行时**可变**的当前目录（受 `cd` 命令、`setCwdState` 影响）。 |
| `sessionId` | 当前会话 UUID，初始值为 `randomUUID()`；每次 `/clear`、`--resume`、`regenerateSessionId` 都会刷新。 |
| `parentSessionId` | 上一个 sessionId（仅当 `regenerateSessionId({setCurrentAsParent:true})` 时设置），用于 plan-mode → implementation 等线性追踪。 |

`switchSession()` 把 `sessionId` 与 `sessionProjectDir` **绑成原子操作**（CC-34 评论），防止两个字段单独 setter 引起的漂移：

```ts
export function switchSession(sessionId, projectDir = null) {
    STATE.planSlugCache.delete(STATE.sessionId)  // 同步清理 slug map
    STATE.sessionId = sessionId
    STATE.sessionProjectDir = projectDir
    sessionSwitched.emit(sessionId)              // 通知 concurrentSessions 等订阅者
}
```

通过 `createSignal` 把“切换事件”暴露成订阅 API：

```ts
export const onSessionSwitch = sessionSwitched.subscribe
```

——这使 `bootstrap` 不必 import 任何 listener 实现，但仍能 fan-out 通知。

### 3.2 成本 / 时长 / 配额（11 个字段）

| 字段 | 说明 |
| --- | --- |
| `totalCostUSD` | 累计 API 费用（USD）。 |
| `totalAPIDuration` / `…WithoutRetries` | 累计 API 时长（含 / 不含重试）。 |
| `totalToolDuration` | 工具累计执行时长。 |
| `turn*` 五元组 | 当前轮（turn）的 hook / tool / classifier 时长与计数；进入下一轮时被 `reset*` 清零。 |
| `startTime` / `lastInteractionTime` | 起会话墙钟、最后一次交互时刻；后者用 `flushInteractionTime` 防抖，避免按键风暴中反复调 `Date.now()`。 |
| `totalLinesAdded/Removed` | 累计编辑行数。 |

**作用**：所有成本 / 配额统计 / 状态栏显示都从这里取；`resetCostState()` 在 /clear 与 session restore 处使用，`setCostStateForRestore()` 在 `--resume` 时把 `.jsonl` 中的累计值写回。

### 3.3 模型 & 模式（6 个字段）

| 字段 | 流程控制 |
| --- | --- |
| `mainLoopModelOverride` | `getMainLoopModelOverride()`：用户用 `--model` 或中途切换的覆盖；模型选择器读它。 |
| `initialMainLoopModel` | 启动时的初始值，`getModelOptions` 用它派生候选列表。 |
| `modelStrings` | lazy init，由 `getModelStrings()` 触发；外部应使用 `utils/model/modelStrings.ts`。 |
| `isInteractive` | REPL vs headless 决策开关；`getIsNonInteractiveSession() = !isInteractive` 决定 1P vs 3P 鉴权路径。 |
| `kairosActive` | kairos feature gate（ant-only）激活态。 |
| `strictToolResultPairing` | HFI 训练模式下置 true 时，`ensureToolResultPairing` 不再补合成 `tool_result`，而是直接抛错——避免让模型在虚假 result 上做条件反射。 |

### 3.4 Telemetry / Logger / Tracer（10 个字段）

进程启动时为 `null`，由 `setMeter(...)` 一并初始化八个 OTel 计数器（`sessionCounter` / `locCounter` / `prCounter` / `commitCounter` / `costCounter` / `tokenCounter` / `codeEditToolDecisionCounter` / `activeTimeCounter`）。每个 `get*Counter()` 都是同步 null-safe 的：CI/无 telemetry 时调用不会抛。

`loggerProvider`、`meterProvider`、`tracerProvider`、`eventLogger`、`statsStore` 分别对应 OpenTelemetry 三大 provider + 内置 stats。`setUseCoworkPlugins` 还顺便调用 `resetSettingsCache()` —— 体现了 setter 不仅是赋值，而是“模式变更的副作用点”。

### 3.5 权限 / 信任 / 旁路（5 个字段）

| 字段 | 流程角色 |
| --- | --- |
| `sessionBypassPermissionsMode` | 本次会话是否处于 `--dangerously-skip-permissions`。**不持久化**——重启回退到 defaults。 |
| `sessionTrustAccepted` | 信任目录弹窗接受后置 true，**不写盘**——同会话内启用受信任目录相关功能。 |
| `sessionPersistenceDisabled` | 关闭 `.jsonl` 持久化（脚本场景），保证脚本退出不留痕。 |
| `kairosActive` | 已在 3.3 列出，是 kairos 模式的“激活标记”。 |
| `userMsgOptIn` | `SendUserMessage` 的 opt-in：未开启时所有调用短路。 |

`preferThirdPartyAuthentication()` 同时读 `isInteractive` 与 `clientType`：

```ts
return getIsNonInteractiveSession() && STATE.clientType !== 'claude-vscode'
```

——非交互式且非 VSCode 时走 1P OAuth。

### 3.6 模式切换附件 / UI sticky 通知（4 个字段）

Plan mode 与 auto mode 各用一对标志做“一次性附件”的触发：

```ts
handlePlanModeTransition(fromMode, toMode):
    if toMode === 'plan' && fromMode !== 'plan':
        STATE.needsPlanModeExitAttachment = false        // 防重入
    if fromMode === 'plan' && toMode !== 'plan':
        STATE.needsPlanModeExitAttachment = true         // 触发附件
```

由 `needsPlanModeExitAttachment` 配合 `hasExitedPlanMode` 决定 `/re-enter plan` 时给出提示；同样模式的 `handleAutoModeTransition` 跳过 auto ↔ plan 互转（让 plan 单独处理）。

`lspRecommendationShownThisSession` 是“一次一条 UI 通知”的 latch。

### 3.7 提示词缓存 sticky-latch（4 个字段）

这是 state.ts 中最精巧的一族设计：

```ts
afkModeHeaderLatched:        boolean | null
fastModeHeaderLatched:       boolean | null
cacheEditingHeaderLatched:   boolean | null
thinkingClearLatched:        boolean | null
```

四个字段都是 **tri-state**：`null` = 尚未评估，`true` = 已锁定开启。一旦置 true，**整个会话剩余生命周期不再回退为 false**。原因：

> 服务端的 prompt cache 大约 50–70K token，TTL ~5 分钟；中途把请求头关掉会立刻让下一次请求 bust cache，反而比一直开着更慢。

| Latch | 触发场景 | 不回退收益 |
| --- | --- | --- |
| `afkModeHeaderLatched` | 第一次进入 auto/AFK 模式时置 true | 模式来回切换不会让 ~70K prompt cache 失效 |
| `fastModeHeaderLatched` | 第一次启用 fast mode 时置 true | cooldown 反复进出不会双 bust |
| `cacheEditingHeaderLatched` | 第一次启用 cached microcompact 时置 true | GrowthBook 开关变化不破坏 cache |
| `thinkingClearLatched` | 上次 API 距今 > 1h（确认 cache miss）时置 true | 已 warm 的新缓存不会因回切 keep:'all' 被破坏 |

`clearBetaHeaderLatches()` 唯一在 `/clear` 与 `/compact` 时调用，**显式允许新会话重新评估**。

### 3.8 其它（lifecycle/hook/cron/cache/compact）

| 字段 | 角色 |
| --- | --- |
| `sessionId` / `parentSessionId` | 见 3.1 |
| `sessionCreatedTeams: Set<string>` | 子 agent 在本会话用 `TeamCreate` 创建的团队 ID，`gracefulShutdown` 时清理防泄漏（gh-32730）。 |
| `sessionCronTasks: SessionCronTask[]` | `CronCreate({durable:false})` 的内存版计时任务，不写 `.claude/scheduled_tasks.json`。 |
| `registeredHooks` | SDK 回调 + 原生 plugin hook 的注册表，按 `HookEvent` 索引；merge-only，清理由 `clearRegisteredHooks` / `clearRegisteredPluginHooks` 提供。 |
| `invokedSkills: Map<key, {skillName, skillPath, content, invokedAt, agentId}>` | 跨 compaction 保留已调用的 skill。key 是 `${agentId ?? ''}:${skillName}` 复合键防止不同 agent 之间覆盖。 |
| `planSlugCache: Map<sessionId, wordSlug>` | 计划会话的 word-slug 缓存。`switchSession` / `regenerateSessionId` 主动清理 outgoing key。 |
| `teleportedSessionInfo` | 由 `setTeleportedSessionInfo` 注入；`markFirstTeleportMessageLogged` 是“已埋首条消息”的 latch。 |
| `lastAPIRequest` / `lastAPIRequestMessages` / `lastClassifierRequests` | 给 `/bug` 命令与 `/share` 转储使用的“最后一次请求快照”。 |
| `cachedClaudeMdContent` | 打破 `yoloClassifier → claudemd → filesystem → permissions` 循环依赖的缓存（context.ts 写入、classifier 读）。 |
| `slowOperations` | ant-only 调试条：每次 `addSlowOperation` 维护一个 10 条上限、TTL 10s 的环形记录。`getSlowOperations()` 用 stable reference 优化（空时返回 `EMPTY_SLOW_OPERATIONS` 常量，让 React `Object.is` 跳过 setState）。 |
| `pendingPostCompaction` | 单次消费：`markPostCompaction()` 在压缩后置 true；`consumePostCompaction()` 由 `logAPISuccess` 调用并自动清零，用于区分“compression 引起的 miss”与“TTL 过期”。 |
| `lastMainRequestId` / `lastApiCompletionTimestamp` | shutdown 阶段送给推理端的 cache eviction 提示，配合 `tengu_api_success` 中的 `timeSinceLastApiCallMs` 关联空闲时长与 cache miss。 |
| `systemPromptSectionCache` | system prompt 各 section 的渲染缓存；`clearSystemPromptSectionState()` 在 layout-level 失效时使用。 |
| `additionalDirectoriesForClaudeMd` | `--add-dir` 注入的额外目录列表，专为 CLAUDE.md 加载。 |
| `allowedChannels` / `hasDevChannels` | `--channels` 解析后逐条存放；带 `dev:true` 的条目获得 allowlist 旁路。 |

---

## 4. getInitialState：一次性初始化与可重置语义

`getInitialState()` 是 State 形状的 **唯一权威定义**，`resetStateForTests()` 用 `Object.entries(getInitialState())` 取所有键值后整体重写 `STATE`：

```ts
export function resetStateForTests(): void {
    if (process.env.NODE_ENV !== 'test') {
        throw new Error('resetStateForTests can only be called in tests')
    }
    Object.entries(getInitialState()).forEach(([k, v]) => {
        STATE[k as keyof State] = v as never
    })
    outputTokensAtTurnStart = 0
    currentTurnTokenBudget = null
    budgetContinuationCount = 0
    sessionSwitched.clear()
}
```

——这种 **“同一份初始值模板”被 init 与 reset 共用** 的写法，是单例测试可重置的核心：所有新字段只要放进 `getInitialState()`，测试间隔离就自动成立。

`cwd` 解析逻辑也只在这里出现一次：先 `realpathSync` 解符号链接 + NFC 归一化，失败（如云盘的 EPERM）则退回 raw cwd。注释明确说这是为了与 `shell.ts setCwd` 行为一致。

---

## 5. State 之外的模块级变量

state.ts 故意把一些“热路径标志”放在模块顶层闭包内，**不写入 STATE**，以避免：

1. 无意义的全局写；
2. 测试需要 reset 它们的额外负担；
3. 影响 `Object.entries(getInitialState())` 模板。

包括：

| 变量 | 用途 |
| --- | --- |
| `outputTokensAtTurnStart` / `currentTurnTokenBudget` / `budgetContinuationCount` | turn 内的 token 预算快照与续算计数 |
| `interactionTimeDirty` | `updateLastInteractionTime()` 的防抖标志；在 Ink 渲染前由 `flushInteractionTime()` 一次性 `Date.now()` |
| `scrollDraining` / `scrollDrainTimer` + 常量 `SCROLL_DRAIN_IDLE_MS=150` | 滚动期间让后台 interval 退避，避免抢事件循环。`waitForScrollIdle()` 用来在昂贵 I/O 前 await |

---

## 6. State 变量在 5 个核心流程中的具体作用

### 6.1 会话生命周期

```
启动 ─[getInitialState()]─▶ originalCwd / projectRoot / sessionId 一次性确定
   │
   ├─ /resume            ─[switchSession(id, dir)]─▶ sessionId + sessionProjectDir 原子切换
   ├─ /clear             ─[regenerateSessionId()]─▶ 生成新 sessionId，旧 plan-slug 清空
   ├─ /compact           ─[markPostCompaction()]──▶ 下一轮 API success 标记 isPostCompaction
   ├─ gracefulShutdown   ─> cleanupSessionTeams(sessionCreatedTeams) 删盘上团队
   └─ Ctrl+C / SIGTERM   ─> cleanupSessionCrons(sessionCronTasks)
```

### 6.2 提示词缓存稳定性

`createSignal` 与 latch 字段共同保证：

1. **第一次进入 auto**：`afkModeHeaderLatched := true`，header 之后永远带上；
2. **缓存 TTL 过期**：`thinkingClearLatched := true`（>1h），下一轮请求把旧的 thinking 清空，warm 新缓存；
3. **新会话**：`/clear` / `/compact` 调用 `clearBetaHeaderLatches()` 重新评估。

——这套机制使得 **runtime 设置可在用户视角下动态切换**，但 **API 视图下保持 header 稳定**。

### 6.3 计量 / 限额 / 状态栏

```
每条 tool call ─[addToToolDuration]─────────────▶ totalToolDuration, turnToolDurationMs++
每条 API ──────[addToTotalDurationState]────────▶ totalAPIDuration(±retries), cost++
每次成功 ──────[setLastMainRequestId/setLastApiCompletionTimestamp]──▶ shutdown 写报告
turn 切换 ─────[resetTurnToolDuration]──────────▶ 下轮计数从零开始
clear/reset ──[resetCostState]───────────────▶ 清空 (--bare 脚本 / /clear)
--resume ─────[setCostStateForRestore]──────▶ 从 .jsonl 恢复数字
```

### 6.4 模式附件（Plan / Auto）

```
进入 plan   ─[handlePlanModeTransition(_, 'plan')]──▶ 清 needsPlanModeExitAttachment
退出 plan   ─[handlePlanModeTransition('plan', _)]──▶ 置 true ──▶ UI 一次性提示
进入 auto   ─[handleAutoModeTransition(_, 'auto')]──▶ 清 needsAutoModeExitAttachment
退出 auto   ─[handleAutoModeTransition('auto', _)]──▶ 置 true ──▶ UI 一次性提示
auto↔plan   ───────────────────────────────────────────▶ 跳过（plan 单独管）
```

### 6.5 测试隔离

单测不需要也不应该持久跨用例状态：

```
beforeEach ─[resetStateForTests()]─▶ 同一份 getInitialState() 重新铺一遍 STATE
                           ─────────▶ 重置模块级闭包变量
                           ─────────▶ 清空 sessionSwitched 信号订阅表
```

`resetStateForTests` 防御性 `if (process.env.NODE_ENV !== 'test') throw` 保证生产代码无法误用。

---

## 7. 鸟瞰图

```
                      ┌────────────────────────────┐
                      │       getInitialState()      │   ← 唯一权威模板
                      └────────────┬───────────────┘
                                   │
                          init (process start)
                                   │
                                   ▼
                      ┌────────────────────────────┐
                      │   STATE : State (module)    │◀────────┐
                      │  - 90+ 字段                 │         │
                      └────────────┬───────────────┘         │
                                   │                         │
                ┌──────────────────┼──────────────────┐      │
                ▼                  ▼                  ▼      │
        sessionId / cwd    cost & duration    latch / mode    │
        toggle             tracking           bits           │
                │                  │                  │      │
                ▼                  ▼                  ▼      │
        switchSession()   addTo…(…) / mark…(…)  setter 包装 │
                │                  │                  │      │
                └──────────────────┴──────────────────┘      │
                                   │                         │
              resetStateForTests() ┴── Object.entries(getInitialState()) 整体回灌
                                                           │
                                              ┌────────────┴────────────┐
                                              ▼                         ▼
                                       测试隔离                     生产恢复
                                                          setCostStateForRestore()
```

---

## 8. 设计要点复盘

1. **类型化全局变量**：用强类型 `State` + 大量 getter/setter，把“看似散落的全局变量”绑成可推导、可单测的整体。
2. **bootstrap 是 DAG 叶子**：使得任何上游模块都能订阅它的状态变化（`onSessionSwitch`），反过来它不依赖任何业务模块，从根本上避免循环。
3. **getter/setter 不只是封装**：很多 setter 内置副作用（`setUseCoworkPlugins` 清 settings 缓存；`regenerateSessionId` 清 plan-slug；`switchSession` 触发信号），把“状态机”逻辑下沉到本文件。
4. **latch vs dynamic**：用“一旦置 true 不再回退”的 latch 把“用户能切换但 API 视图稳定”解耦，避免 prompt-cache bust。
5. **session-only vs persistent**：用 `session*` 前缀显式标记“不写盘的字段”——同一份 `setCostStateForRestore` / `resetCostState` 对照下，restore 不会带着 `sessionBypassPermissionsMode` 等“本会话意愿”回写到新会话。
6. **模块级 vs STATE**：纯 UI 节流（`scrollDraining`、`interactionTimeDirty`）留在闭包里，不进 STATE——避免污染模板、减少测试 reset 的耦合。
7. **测试重置只需一份模板**：`resetStateForTests()` 用 `Object.entries(getInitialState())` 自动覆盖所有新增字段——这是把“模板即 schema”的好处发挥到极致。

---

## 9. 维护提醒

- 添加新字段前请确认**它真的需要跨模块共享**——大多数情况更适合放进业务模块自身；
- 新字段如果需要“永不回退”的语义，请参考 latch 模式（`boolean | null`，避免 `false` 这种语义陷阱）；
- 给 setter 加副作用时，确保它在 reset / restore 路径下不会漏触发；
- 任何对 `sessionId` 的修改都应经过 `regenerateSessionId` / `switchSession`，否则 `planSlugCache` 等子表会留下 stale key。
