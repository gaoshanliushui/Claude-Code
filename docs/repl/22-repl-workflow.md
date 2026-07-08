# REPL 工作流程详解

> 范围：`src/screens/REPL.tsx`（约 5421 行）—— Claude Code 交互式终端会话的**主屏幕组件**。
>
> 本文档自顶向下讲清 REPL 的职责、状态模型、生命周期、提交→查询→渲染主流程，以及取消 / 续接 / 回溯 / 队列 / transcript 等侧流程。代码引用均为 `file:line`，可点击跳转。

## 1. REPL 是什么

`REPL`（`src/screens/REPL.tsx:784`）是一个巨型 React 函数组件，由 `replLauncher.tsx` 包在 `<App>` 里渲染（见 `docs/20-request-flow.md` §1.2）。它是用户与 Claude 交互的**唯一前台界面**，承担五大职责：

| 职责 | 体现 |
| --- | --- |
| **输入采集** | 渲染 `<PromptInput>`，接收文本 / slash 命令 / bash 命令 / 粘贴内容 |
| **查询编排** | `onSubmit → onQuery → onQueryImpl`，驱动 `query()` 主循环（见 `docs/21-repl-refactor.md`） |
| **消息渲染** | 把 `messages[]` 经 `<Messages>` 渲染成终端 UI（流式文本、工具块、思考块） |
| **会话控制** | 取消（ctrl+c）、续接（/resume）、回溯（消息选择器）、队列（忙时入队）、transcript（全屏滚动） |
| **状态聚合** | 把 AppState（全局 zustand store）和本地 React state 合并成渲染所需的派生状态 |

它**不直接调用 Anthropic API**——那是 `query.ts` / `QueryEngine.ts` 的事；REPL 只负责"把用户输入喂进去、把流式事件画出来"。

## 2. Props：外部注入的初始条件

`Props`（`REPL.tsx:737`）是 REPL 与启动层的契约，关键字段：

```ts
type Props = {
    commands: Command[];              // 所有可用的 slash 命令
    initialTools: Tool[];             // 初始工具集
    initialMessages?: MessageType[];  // --resume 时的历史消息
    pendingHookMessages?: Promise<…>; // 延迟注入的 SessionStart hook 消息
    systemPrompt?: string;            // 自定义系统提示
    appendSystemPrompt?: string;      // 追加系统提示
    onBeforeQuery?: (…) => Promise<boolean>;  // 查询前置钩子（返回 false 阻止）
    onTurnComplete?: (messages) => …; // 回合完成回调（SDK / 测试用）
    mainThreadAgentDefinition?: AgentDefinition; // 主线程 agent 人格
    remoteSessionConfig?: RemoteSessionConfig;   // --remote 模式
    directConnectConfig?: DirectConnectConfig;   // claude connect 模式
    sshSession?: SSHSession;          // claude ssh 模式
    thinkingConfig: ThinkingConfig;   // 思考预算配置
};
```

注意 `commands` / `mainThreadAgentDefinition` 等都被改名为 `initial*` 解构（`REPL.tsx:784-810`），因为它们会被提升为可变 state（`/resume`、`/agents` 等命令可以中途改写）。

## 3. 状态模型

REPL 持有 **~60 个 useState/useRef**，按用途分六组：

### 3.1 会话核心状态

| 状态 | 行号 | 含义 |
| --- | --- | --- |
| `messages` / `messagesRef` | `1396` / `1397` | 完整对话消息数组；ref 镜像保证同步读取 |
| `conversationId` | `1695` | compact / clear 后递增，强制 Messages 行 key 失效重渲 |
| `mainThreadAgentDefinition` | `829` | 主线程人格（可被 /resume 改写） |
| `localCommands` | `893` | slash 命令集（可被插件 / 远程动态扩展） |

### 3.2 加载与查询守卫

```ts
const queryGuard = React.useRef(new QueryGuard()).current;        // REPL.tsx:1114
const isQueryActive = useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot);  // 1118
const [isExternalLoading, …] = useState(…);                       // 1125
const isLoading = isQueryActive || isExternalLoading;             // 1130
```

- `QueryGuard` 是一个状态机（`utils/QueryGuard.ts`），用 `tryStart()` / `end()` / `forceEnd()` 原子地管理 idle→running→idle 转换，**防止并发 query**。
- `isQueryActive` 通过 `useSyncExternalStore` 订阅守卫——这是"本地 query 是否在飞"的**唯一真相源**。
- `isExternalLoading` 覆盖远程 / 后台任务前台化等不走 queryGuard 的路径。
- `isLoading` 是二者的或——驱动 spinner 显隐。

### 3.3 流式渲染状态

| 状态 | 行号 | 含义 |
| --- | --- | --- |
| `streamMode` | `1050` | spinner 模式（responding / thinking / …） |
| `streamingToolUses` | `1061` | 正在流式输出的 tool_use 块 |
| `streamingThinking` | `1062` | 正在流式输出的 thinking 块 |
| `streamingText` | `1675` | 正在流式输出的 assistant 文本（落地前的预览） |
| `responseLengthRef` | `1641` | 已接收的响应字符数（驱动 spinner 动画，避免 re-render） |

### 3.4 输入状态

| 状态 | 行号 | 含义 |
| --- | --- | --- |
| `inputValue` / `inputValueRef` | `1545` / `1546` | 当前输入框文本（初值来自 `consumeEarlyInput()`） |
| `inputMode` | `1586` | prompt / bash / memory 模式 |
| `pastedContents` | `1637` | 粘贴的图片 / 长文本（按 id 索引） |
| `stashedPrompt` | `1587` | 临时暂存的草稿 |

### 3.5 弹窗 / 权限队列

| 状态 | 行号 | 含义 |
| --- | --- | --- |
| `toolJSX` | `1246` | local-jsx 命令（如 /config）渲染的 JSX |
| `toolUseConfirmQueue` | `1315` | 待确认的工具权限请求队列 |
| `sandboxPermissionRequestQueue` | `1320` | 沙箱网络权限请求队列 |
| `promptQueue` | `1324` | 通用 prompt 请求队列（elicitation 等） |
| `isMessageSelectorVisible` | `1692` | 消息选择器（回溯）是否打开 |

### 3.6 屏幕 / 派生状态

```ts
const [screen, setScreen] = useState<Screen>('prompt');           // 915，'prompt' | 'transcript'
const deferredMessages = useDeferredValue(messages);              // 1532
const displayedMessages = viewedAgentTask ? … : usesSyncMessages ? messages : deferredMessages;  // 4792
const showSpinner = (!toolJSX || …) && toolUseConfirmQueue.length === 0 && … ;  // 1886
```

- `useDeferredValue(messages)` 把渲染压力延迟：流式高频更新时 `deferredMessages` 落后于 `messages`，让输入框保持响应。
- `displayedMessages` 是最终上屏的数组——可能是 teammate 的消息、同步消息或延迟消息（`REPL.tsx:4792`）。

## 4. 生命周期

### 4.1 挂载

```ts
useEffect(() => {
    logForDebugging(`[REPL:mount] …`);
    return () => logForDebugging(`[REPL:unmount] …`);
}, [disabled]);                                                   // REPL.tsx:823
```

挂载即开始一系列 effect：MCP 连接管理、IDE 自动连接、防睡眠、leader 队列注册等。

### 4.2 初始消息处理

`processInitialMessage`（`REPL.tsx:3253`，由 `useEffect` 在 `isLoading` 变 false 时触发，`:3246-3362`）：

- 来源：CLI 参数（`claude "帮我写贪吃蛇"`）或 plan mode 退出携带的初始消息。
- 若 `clearContext`（plan mode 退出）→ 先 `clearConversation` 再注入。
- 通过 `initialMessageRef` 保证只处理一次。

### 4.3 续接（--resume）

`resume`（`REPL.tsx:1949`）：

- 重放历史 `initialMessages`；
- 通过 `restoreReadFileState`（`:2185`）重建已读文件状态；
- `initialContentReplacements` 重建内容替换记录。

### 4.4 延迟 hook 消息

`pendingHookMessages` 是个 Promise：REPL **先渲染**，等 SessionStart hook 解析完，通过 `useDeferredHookMessages`（`:1345`）注入。第一次 API 调用前会 await 它。

## 5. 主流程：提交 → 查询 → 渲染

```
用户键入 + Enter
   │
   ▼
PromptInput.onSubmit ──► REPL.onSubmit (REPL.tsx:3364)
   │                         │ 处理 typeahead / 转发
   ▼                         ▼
handlePromptSubmit ──► executeUserInput ──► onQuery (REPL.tsx:3157)
                                              │ queryGuard.tryStart()
                                              │ setMessages([...新消息])
                                              │ onBeforeQuery 钩子
                                              ▼
                                         onQueryImpl (REPL.tsx:2878)
                                              │ 1. prepareQueryEnvironment
                                              │ 2. handleNonQueryTurn (早返回)
                                              │ 3. loadQueryContext
                                              │ 4. runQueryStream
                                              │    └─ for await query() → onQueryEvent
                                              │ 5. finalizeQueryTurn
                                              ▼
                                         onQueryEvent (REPL.tsx:2801)
                                              │ handleMessageFromStream
                                              ▼
                                         setMessages / setStreamingText / …
                                              ▼
                                         <Messages> 重渲染 → ink → stdout
```

### 5.1 onSubmit（输入入口）

`onSubmit`（`REPL.tsx:3364`）是 PromptInput 的提交回调。它处理 typeahead 建议、speculation accept，然后转发到 `handlePromptSubmit`（`utils/handlePromptSubmit.ts:120`，详见 `docs/20-request-flow.md` §2）。

### 5.2 onQuery（并发守卫 + 收尾）

`onQuery`（`REPL.tsx:3157`）是 query 的真正入口，职责：

1. **swarm 标记**：teammate 模式下标记自己 active（`:3158-3166`）。
2. **并发守卫**：`queryGuard.tryStart()` 返回 generation 号，若返回 null（已在跑）→ 把新消息入队 enqueue 后返回（`:3168-3185`）。
3. **追加消息 + 重置计时**：`setMessages([...new])`、`resetTimingRefs()`、清空 streaming 状态（`:3186-3197`）。
4. **前置钩子**：`mrOnBeforeQuery` + `onBeforeQueryCallback`（返回 false 则中止，`:3200-3215`）。
5. **调用 onQueryImpl**（`:3217`）。
6. **finally 收尾**（`:3218+`）：`queryGuard.end()` 原子转回 idle、记录完成时间、turn duration 消息、**自动回溯**（用户在无响应时中断 → 恢复其 prompt，`:3300+`）。

### 5.3 onQueryImpl（五子函数编排）

详见 `docs/21-repl-refactor.md`。五个子函数：
1. `prepareQueryEnvironment`（`:2890`）：IDE / onboarding / 会话标题 / allowedTools。
2. `handleNonQueryTurn`（`:2972`）：`!shouldQuery` 早返回。
3. `loadQueryContext`（`:3022`）：systemPrompt / userContext / systemContext 并行加载。
4. `runQueryStream`（`:3076`）：`for await query()` → `onQueryEvent`。
5. `finalizeQueryTurn`（`:3111`）：API metrics / profile / reset / onTurnComplete。

### 5.4 onQueryEvent（流式事件分发）

`onQueryEvent`（`REPL.tsx:2801`）把 query() yield 的每个事件交给 `handleMessageFromStream`（`utils/messages.ts:2930`），回调里：

- **compact boundary**：全屏模式保留 scrollback、非全屏直接 reset；递增 `conversationId`（`:2803-2825`）。
- **ephemeral progress**（sleep/bash tick）：替换前一条同名 progress 而非追加，防止 messages 数组膨胀（`:2826-2845`）。
- **普通消息**：`setMessages(old => [...old, newMessage])`（`:2846`）。
- **partial delta**：只更新 `responseLength`（驱动 spinner），不入 messages（`:2860+`）。

## 6. 侧流程

### 6.1 取消（ctrl+c）

`CancelRequestHandler`（render 树 `:4707` / `:4858`）+ `onCancel`（`:2322`）：

- ctrl+c 调 `abortController.abort('user-cancel')` → `queryGuard.forceEnd()`。
- finally 里检测 `reason === 'user-cancel'` 且无实质响应 → **自动回溯**：rewind 对话并恢复用户 prompt（`onQuery` finally，`:3300+`）。

### 6.2 队列（忙时入队）

当 `queryGuard.isActive` 时：
- `handlePromptSubmit` 把输入 `enqueue()`（`handlePromptSubmit.ts:336`）。
- 当前 turn 结束后，`executeQueuedInput`（`REPL.tsx:4111`）逐个消费队列。
- `<PromptInputQueuedCommands>`（`:4910`）在 UI 显示待处理队列。

### 6.3 回溯（消息选择器）

- `handleShowMessageSelector`（`:3885`）打开 `<MessageSelector>`（`:5321`）。
- 选中某条用户消息 → `rewindConversationTo`（`:3894`）/ `handleRestoreMessage`（`:3978`）截断对话并恢复输入。
- `restoreMessageSync`（`:3945`）是同步版本（自动回溯路径用）。

### 6.4 续接 / 远程 / 后台

- `useRemoteSession` / `useDirectConnect`：`--remote` / `connect` 模式走 CCR 引擎，置 `isExternalLoading`。
- `useSessionBackgrounding`：后台任务前台化。
- `handleIncomingPrompt`（`:4251`）：bridge（移动端）、proactive tick、scheduled task 等外部来源的 prompt 统一入口。

### 6.5 Transcript 全屏模式

`screen === 'transcript'` 时走早返回（`REPL.tsx:4655`）：

- 全屏 + 虚拟滚动（`FullscreenLayout` + `ScrollBox`）展示完整历史。
- `<TranscriptSearchBar>`（`:4712`）支持 `/` 搜索、`n`/`N` 跳转。
- 非全屏 / kill switch 退回 legacy 渲染（30 条上限 + dump 到 scrollback）。

## 7. 渲染树

REPL 有**两个渲染分支**，互斥：

### 7.1 Transcript 分支（`REPL.tsx:4655-4768`）

```
<KeybindingSetup>
  <AnimatedTerminalTitle/>
  <GlobalKeybindingHandlers/>
  <CommandKeybindingHandlers/>
  <ScrollKeybindingHandler/>           // 滚动键
  <CancelRequestHandler/>
  <FullscreenLayout scrollable={<Messages.../>} bottom={搜索栏 or TranscriptModeFooter}/>
</KeybindingSetup>
```

### 7.2 主分支（`REPL.tsx:4837-5414`）

```
<KeybindingSetup>
  <AnimatedTerminalTitle/>                       // 终端标题动画
  <GlobalKeybindingHandlers/>                    // 全局快捷键
  <VoiceKeybindingHandler/>                       // 语音模式（可选）
  <CommandKeybindingHandlers/>                    // 命令快捷键
  <ScrollKeybindingHandler/>                      // 滚动
  <MessageActionsKeybindings/>                    // 消息操作（可选）
  <CancelRequestHandler/>                         // ctrl+c
  <MCPConnectionManager>                          // MCP 连接生命周期
    <FullscreenLayout
      overlay={权限弹窗}
      modal={居中弹窗}
      scrollable={
        <TeammateViewHeader/>
        <Messages messages={displayedMessages} … streamingText={…}/>  // ★ 核心消息渲染
        <AwsAuthStatusBox/>
        {placeholderText && <UserTextMessage/>}    // 用户输入回显占位
        {toolJSX && <Box>{toolJSX.jsx}</Box>}       // local-jsx 命令 UI
        <Box flexGrow={1}/>
        {showSpinner && <SpinnerWithVerb …/>}       // ★ spinner（含 TaskListV2）
        {isFullscreen && <PromptInputQueuedCommands/>}
      }
      bottom={
        {permissionStickyFooter}
        <PromptInput …/>                            // ★ 输入框
        <MessageSelector …/>                        // 回溯弹窗
      }
    />
  </MCPConnectionManager>
</KeybindingSetup>
```

三个核心 UI（带 ★）：
- **`<Messages>`**（`:4870`）：渲染 `displayedMessages` + 流式文本/思考/工具块。
- **`<SpinnerWithVerb>`**（`:4901`）：加载动画，内嵌 `TaskListV2` 任务清单（见 `docs/18-task-planning.md`）。
- **`<PromptInput>`**（`:5292`）：输入框，`onSubmit` 回到 §5.1。

## 8. 键盘事件分层

REPL 用多个 `*KeybindingHandlers` 组件分层处理键盘，按**挂载顺序**决定优先级：

| 组件 | 职责 | 关键约束 |
| --- | --- | --- |
| `GlobalKeybindingHandlers` | 全局快捷键（如 ctrl+r transcript） | 最外层 |
| `CommandKeybindingHandlers` | slash 命令快捷键 | `isActive` 由 local-jsx 状态门控 |
| `ScrollKeybindingHandler` | 滚动（PgUp/PgDn/j/k） | **必须挂在 CancelRequestHandler 之前**——ctrl+c 带选区时复制而非取消（`:4845-4848`） |
| `CancelRequestHandler` | ctrl+c 取消 / 双击退出 | 读 `cancelRequestProps` |
| `useInput` (内联, `:4475`/`:4533`) | 局部键处理 | transcript 模式专用 |

## 9. 关键设计要点

| 决策 | 理由 | 源码依据 |
| --- | --- | --- |
| `messagesRef` 镜像 `messages` | 异步回调（队列、hook、调度任务）需同步读最新消息，不等 React 调度 | `REPL.tsx:1397, 1412` |
| `useSyncExternalStore` 订阅 queryGuard | "是否在查询"必须是单一真相源，跨组件一致 | `REPL.tsx:1118` |
| `useDeferredValue(messages)` | 流式高频更新时延迟渲染压力，保输入框响应 | `REPL.tsx:1532` |
| `conversationId` 递增 | compact/clear 后强制 Messages 行 key 失效，避免 stale memo | `REPL.tsx:1695, 2954` |
| ephemeral progress 替换不追加 | 防止 sleep/bash 的每秒 tick 把 messages 撑到 13k+ | `REPL.tsx:2826-2845` |
| ScrollKeybinding 挂在 Cancel 之前 | ctrl+c 带选区复制、无选区取消 | `REPL.tsx:4845-4848` |
| 自动回溯（user-cancel + 无响应） | 用户秒退时不丢失刚输入的 prompt | `REPL.tsx:3300+` |
| transcript 走早返回独立分支 | 全屏虚拟滚动与主分支互斥，避免双 ScrollBox 占 250MB | `REPL.tsx:4655-4664` |
| 忙时 enqueue + executeQueuedInput | 串行化 turn，防并发改 AppState | `handlePromptSubmit.ts:336`, `REPL.tsx:4111` |
| `isExternalLoading` 独立于 queryGuard | 远程/后台任务不走 queryGuard，需独立 spinner 控制 | `REPL.tsx:1125-1130` |

## 10. 一句话总结

> REPL 是 Claude Code 的**主交互组件**：它用 `QueryGuard` 守卫并发、用 `messages[]`（+ ref 镜像 + deferred 派生）承载对话、用 `onSubmit → onQuery → onQueryImpl → onQueryEvent` 四级链路把用户输入变成流式 API 调用再变回屏幕上的消息，并用 `toolJSX` / `toolUseConfirmQueue` / 消息选择器 / transcript 等侧通道覆盖权限确认、回溯、全屏浏览等场景。整个组件以"AppState 全局状态 + 本地 React state + ref 镜像"三层状态模型为骨架，把异步流式、并发守卫、键盘分层、虚拟滚动这些复杂度收拢在一个文件里，对外只暴露 `Props` 一个契约。
