# REPL.tsx 重构文档：拆分 onQueryImpl 为五个子函数

> 范围：`src/screens/REPL.tsx` 的 `onQueryImpl` useCallback（约 2878–3156 行）。
>
> 文中代码引用均使用 `file:line` 形式，可点击跳转。

## 1. 背景与目标

### 1.1 原始问题

`onQueryImpl` 原本是 REPL 中**最长的单一方法**（约 194 行），单文件 5421 行中它独占近 4%。它在单个 useCallback 回调中混杂了五种完全不同关注点的逻辑：

1. **运行时环境准备**（IDE 客户端、onboarding、会话标题生成、skill 注入的 allowedTools 写入）。
2. **非查询分支处理**（`!shouldQuery` 早返回路径，含 compact boundary 检测）。
3. **查询上下文加载**（systemPrompt / userContext / systemContext 并行预取、effort 覆盖、auto mode kill switch）。
4. **流式查询执行**（进入 `query()` 生成器、yield 事件到 `onQueryEvent`、BUDDY companion observer）。
5. **回合收尾**（ant-only API metrics 收集、profile 报告、状态清理、turn-complete 回调）。

这种"长方法"带来三类实际问题：

| 问题 | 表现 |
| --- | --- |
| 阅读成本 | 一段函数里同时讲 IDE / skill / compact / Claude API / metrics 五件事，新人需要逐行注释才能跟进。 |
| 调试成本 | 想加 console.log 临时调试 metrics 写入，必须越过整个 query 循环。 |
| 测试成本 | 没法单独 stub 任一阶段，所有逻辑都耦合在同一个 useCallback 中。 |

### 1.2 重构目标

把 `onQueryImpl` 拆成 **5 个子函数 + 1 个薄编排层**，每个子函数职责单一、可以独立理解。**保持行为不变**：所有原始副作用、API 调用顺序、并发窗口全部保留。

## 2. 拆分结果总览

```text
onQueryImpl (useCallback, ~17 行编排)
├── prepareQueryEnvironment()          // 同步：环境副作用
├── handleNonQueryTurn()                // 同步：早返回路径
├── loadQueryContext(): Promise<void>   // 异步：系统提示 + 上下文加载
│   ↳ return { systemPrompt, userContext, systemContext }
├── runQueryStream(): Promise<void>     // 异步：query() 生成器迭代
└── finalizeQueryTurn(): Promise<void>  // 异步：metrics + profile + reset
```

每个子函数都用 `const name = async () => { ... }` 定义在 `useCallback` 的回调体内，因此：

- 自动闭包捕获外层 hook 提供的 state/ref/handler；
- 不引入新的 React 钩子，**useCallback 的 deps 数组不变**；
- 调用栈与原版一致（调用 → 子函数），DevTools 仍然能看清因果。

## 3. 各子函数详解

### 3.1 `prepareQueryEnvironment()` — 环境副作用

位置：`src/screens/REPL.tsx:2890`

```ts
const prepareQueryEnvironment = () => {
    if (shouldQuery) { /* IDE 集成 + 关闭未提交 diff */ }
    void maybeMarkProjectOnboardingComplete();
    if (!titleDisabled && /* ... */) { /* 异步生成会话标题 */ }
    store.setState(prev => { /* 写入 skill 注入的 allowedTools */ });
};
```

**职责**：在真正开始 query 之前，把"运行环境"准备到位——IDE 客户端、onboarding、会话标题、allowedTools。

**关键不变量**：
- **必须先于 `!shouldQuery` 早返回执行**：forked slash command 的 `shouldQuery=false` 分支也依赖 allowedTools 已写入 store（`TaskUpdateTool.ts:139-143` 注释里强调过这点）。
- **不读取 `toolUseContext`**：toolUseContext 在 shouldQuery=true 时才构造，所以子函数里只能依赖 `newMessages` / 全局 store。
- **不返回任何值**：所有副作用通过 `setAppState` / `store.setState` 表达。

**对应原始代码**：`onQueryImpl` 2878-2943 行（IDE 客户端、onboarding、会话标题、allowedTools 写入四段连续代码）。

### 3.2 `handleNonQueryTurn()` — 早返回路径

位置：`src/screens/REPL.tsx:2972`

```ts
const handleNonQueryTurn = () => {
    if (!shouldQuery) {
        if (newMessages.some(isCompactBoundaryMessage)) {
            setConversationId(randomUUID());
            if (feature('PROACTIVE') || feature('KAIROS')) {
                proactiveModule?.setContextBlocked(false);
            }
        }
        resetLoadingState();
        setAbortController(null);
    }
};
```

**职责**：处理 `shouldQuery=false` 的早返回路径。

**关键不变量**：
- **总是先检查 shouldQuery** 而非由调用方决定——这样调用方可以无脑调用 `handleNonQueryTurn()` 而无需 if 包裹。
- **proactive 模块的 `setContextBlocked(false)` 仅在 compact boundary 触发后调用**：避免非 compact 场景（如单纯的 forked command）误唤醒 proactive ticks。
- **`setAbortController(null)` 必须清空**：否则 CancelRequestHandler 的 `canCancelRunningTask` 仍返回 true，ctrl+c 会被路由到 `onCancel()`（无操作）而非 double-press exit。

**对应原始代码**：`onQueryImpl` 2945-2962 行（含原始的 `if (!shouldQuery) { ... return; }`）。

### 3.3 `loadQueryContext()` — 上下文加载

位置：`src/screens/REPL.tsx:3022`

```ts
const loadQueryContext = async (): Promise<{ systemPrompt, userContext, systemContext }> => {
    // effort 覆盖（可选）
    if (effort !== undefined) { toolUseContext.getAppState = () => ({ ...prev, effortValue: effort }); }

    queryCheckpoint('query_context_loading_start');
    const [, , defaultSystemPrompt, baseUserContext, systemContext] = await Promise.all([
        checkAndDisableBypassPermissionsIfNeeded(...),
        TRANSCRIPT_CLASSIFIER ? checkAndDisableAutoModeIfNeeded(...) : undefined,
        getSystemPrompt(freshTools, mainLoopModelParam, ...),
        getUserContext(),
        getSystemContext(),
    ]);

    const userContext = { ...baseUserContext, ...getCoordinatorUserContext(...), ...focusInfo };
    const systemPrompt = buildEffectiveSystemPrompt({ ... });
    toolUseContext.renderedSystemPrompt = systemPrompt;

    return { systemPrompt, userContext, systemContext };
};
```

**职责**：把 systemPrompt / userContext / systemContext 三元组加载完毕。

**关键设计**：
- **并行预取 5 个 Promise**：bypass 检查、auto mode 检查、系统提示、用户上下文、系统上下文——总延迟 = max(5)，非 sum。
- **返回值用对象解构**而非闭包变量：调用方通过 `const { systemPrompt, userContext, systemContext } = await loadQueryContext()` 显式接收。
- **effort 覆盖写入 toolUseContext.getAppState 而非全局 store**（`onQueryImpl:3026-3032`）：保持 override 仅作用于本轮，避免 background agents / UI subscribers（Spinner、LogoV2）看到。

**对应原始代码**：`onQueryImpl` 2963-3005 行（含 effort override 块、Promise.all、userContext 合并、buildEffectiveSystemPrompt 调用）。

### 3.4 `runQueryStream()` — 流式查询执行

位置：`src/screens/REPL.tsx:3076`

```ts
const runQueryStream = async (): Promise<void> => {
    queryCheckpoint('query_query_start');
    resetTurnHookDuration();
    resetTurnToolDuration();
    resetTurnClassifierDuration();
    for await (const event of query({ messages, systemPrompt, userContext, systemContext,
                                      canUseTool, toolUseContext, querySource })) {
        onQueryEvent(event);
    }
    if (feature('BUDDY')) {
        void fireCompanionObserver(messagesRef.current, /* set companionReaction */);
    }
    queryCheckpoint('query_end');
};
```

**职责**：进入 query() 异步生成器，把每个流式事件 yield 给 onQueryEvent。

**关键设计**：
- **时长累计器（hookMs/toolMs/classifierMs）先 reset**：保证只统计本轮。
- **checkpoint('query_query_start') 和 ('query_end') 包住整个 query 流**：便于 profile。
- **BUDDY observer 仅在流结束后触发**：避免在生成过程中频繁改 companionReaction。

**对应原始代码**：`onQueryImpl` 3006-3027 行（含 queryCheckpoint、resetTurn*、for-await、companion observer、queryCheckpoint）。

### 3.5 `finalizeQueryTurn()` — 回合收尾

位置：`src/screens/REPL.tsx:3111`

```ts
const finalizeQueryTurn = async (): Promise<void> => {
    if ("external" === 'ant' && apiMetricsRef.current.length > 0) {
        // 计算 P50、组装 ApiMetricsMessage、写入 messages
    }
    resetLoadingState();
    logQueryProfileReport();
    await onTurnComplete?.(messagesRef.current);
};
```

**职责**：query 流结束后做收尾。

**关键不变量**：
- **ant-only metrics 必须先于 `resetLoadingState()` 写入**：`resetLoadingState` 会清空 apiMetricsRef。
- **P50 计算跨多请求**：多轮 tool-use 时每个 round 各记一份 entries，最终取中位数（`median()` in `REPL.tsx:474`）。
- **`onTurnComplete?.()` 是回调而非同步**：用于测试 / SDK 集成监听 turn 边界。

**对应原始代码**：`onQueryImpl` 3029-3070 行（metrics 计算 + resetLoadingState + logQueryProfileReport + onTurnComplete）。

## 4. 编排层（薄）

`onQueryImpl` 主体现在仅 17 行（2878-3155 中的非子函数定义部分）：

```ts
const onQueryImpl = useCallback(async (messagesIncludingNewMessages, newMessages, abortController, shouldQuery, additionalAllowedTools, mainLoopModelParam, effort) => {
    // ... 五个子函数定义 ...

    prepareQueryEnvironment();                                  // 1. 环境副作用

    if (!shouldQuery) {                                         // 2. 早返回
        handleNonQueryTurn();
        return;
    }

    const toolUseContext = getToolUseContext(/* ... */);        // 3. 构造上下文
    const { tools: freshTools, mcpClients: freshMcpClients } = toolUseContext.options;

    const { systemPrompt, userContext, systemContext } =        // 4. 加载上下文
        await loadQueryContext();

    await runQueryStream();                                     // 5. 流式查询
    await finalizeQueryTurn();                                  // 6. 收尾
}, [/* 14 deps，未变 */]);
```

这与原版的**调用顺序、并发窗口、错误传播路径**完全一致——每个子函数就是原代码的物理搬家，未引入任何新的 if/try/await 边界。

## 5. 完成的工作

| 阶段 | 工作 | 涉及位置 |
| --- | --- | --- |
| 1 | 阅读原始 `onQueryImpl`（194 行）并识别 5 个关注点 | `REPL.tsx:2878-3071` |
| 2 | 提取 `prepareQueryEnvironment` | `REPL.tsx:2890-2956` |
| 3 | 提取 `handleNonQueryTurn` | `REPL.tsx:2972-2990` |
| 4 | 提取 `loadQueryContext`（含 Promise.all 并行加载 + 返回值） | `REPL.tsx:3022-3060` |
| 5 | 提取 `runQueryStream`（含 for-await + companion observer） | `REPL.tsx:3076-3099` |
| 6 | 提取 `finalizeQueryTurn`（含 P50 metrics + resetLoadingState） | `REPL.tsx:3111-3154` |
| 7 | 编排层串联五个子函数 | `REPL.tsx:2878-3156`（编排代码约 17 行） |
| 8 | 修复 Unicode 转义问题（部分中文注释曾被工具保存为 `\uXXXX` 字面量） | 通过 `scripts/fix_chinese_escapes.py` 一次性转码 |

## 6. 流程

```
用户输入 → handlePromptSubmit → executeUserInput
                                    ↓
                               onQuery（外部 useCallback）
                                    ↓
                              onQueryImpl（本文件重构对象）
                                    ↓
        ┌─────────────────────────────────────────────────┐
        │ 1. prepareQueryEnvironment                      │
        │    IDE / onboarding / 会话标题 / allowedTools    │
        └─────────────────────────────────────────────────┘
                                    ↓
                          ┌─────────────────┐
                          │ shouldQuery?    │
                          └─────────────────┘
                          Yes ↓           ↓ No
        ┌──────────────────────────┐    ┌──────────────────────────┐
        │ 2'. getToolUseContext    │    │ 2. handleNonQueryTurn    │
        │     构造 toolUseContext   │    │     compact / reset      │
        │     提取 freshTools/...   │    │     直接 return          │
        └──────────────────────────┘    └──────────────────────────┘
                          ↓
        ┌─────────────────────────────────────────────────┐
        │ 3. loadQueryContext                             │
        │    5 个 Promise 并行                            │
        │    + effort 覆盖                                │
        │    返回 {systemPrompt, userContext, systemContext}│
        └─────────────────────────────────────────────────┘
                          ↓
        ┌─────────────────────────────────────────────────┐
        │ 4. runQueryStream                               │
        │    for-await query() 生成器                     │
        │    每个 event → onQueryEvent                    │
        │    + companion observer（如果 BUDDY）            │
        └─────────────────────────────────────────────────┘
                          ↓
        ┌─────────────────────────────────────────────────┐
        │ 5. finalizeQueryTurn                            │
        │    计算 ApiMetrics（P50）                        │
        │    resetLoadingState                            │
        │    logQueryProfileReport                        │
        │    onTurnComplete 回调                          │
        └─────────────────────────────────────────────────┘
```

## 7. 设计要点与权衡

| 决策 | 理由 | 源码依据 |
| --- | --- | --- |
| 子函数定义为 `const name = ... = () => {...}` 而非 `useCallback` | 子函数仅在每次 onQueryImpl 执行时用一次，无需稳定引用；不增加 hook 数量 → 不影响 useCallback deps | `REPL.tsx:2890, 2972, 3022, 3076, 3111` |
| `loadQueryContext` 返回对象而非闭包变量 | 调用方通过解构显式接收，副作用更清晰 | `REPL.tsx:3061` |
| `handleNonQueryTurn` 内部自检 `if (!shouldQuery)` | 调用方写 `handleNonQueryTurn(); return;` 即可，不必写 if | `REPL.tsx:2992-2995` |
| `prepareQueryEnvironment` 无返回值 | 副作用全部通过 store / setAppState 表达，保持同步 | `REPL.tsx:2890` |
| `runQueryStream` 无返回值 | 流已结束，调用方不需要数据 | `REPL.tsx:3076` |
| `finalizeQueryTurn` async | `onTurnComplete?.()` 返回 Promise（用于测试 / SDK 等待） | `REPL.tsx:3153` |
| 保留 useCallback 原 14 项 deps | 子函数只闭包 useCallback 内变量，外部依赖未变 → 无回归风险 | `REPL.tsx:3156` |
| 不引入新文件 / 不拆 component | 单文件改动最小，git diff 集中，方便 review 与回滚 | 整文件 |
| 不改英文注释 | 原作者的英文注释保留——其中包含历史背景 / tengu_* 实验代号 / 性能数据 | 各子函数内 |

## 8. 一句话总结

> 重构后的 `onQueryImpl` 是一个 **5 子函数 + 薄编排层** 的 useCallback：环境副作用走 `prepareQueryEnvironment`、早返回走 `handleNonQueryTurn`、上下文加载走 `loadQueryContext`（5 路并行，返回对象）、流式执行走 `runQueryStream`、收尾走 `finalizeQueryTurn`。每个子函数都附带中文注释说明职责与关键不变量，行为与原版完全等价，但 194 行的"巨型方法"被压缩为 17 行的"可读编排"。