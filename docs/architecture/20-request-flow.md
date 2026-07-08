# Claude Code 端到端请求流程：从键入"帮我写个贪吃蛇"到控制台输出

> 范围：从用户在 REPL 输入一行文本开始，到 Ink 终端渲染出最终回复的全链路。
>
> 文中代码引用均使用 `file:line` 形式，可点击跳转。

## 0. 鸟瞰：四级调用栈

```
┌────────────────────────────────────────────────────────────────────┐
│ Level 0  进程入口       src/entrypoints/cli.tsx → src/main.tsx     │
│ Level 1  REPL 启动      src/replLauncher.tsx → src/screens/REPL.tsx│
│ Level 2  用户输入        PromptInput → handlePromptSubmit          │
│                          → processUserInput / processTextPrompt    │
│ Level 3  查询循环        src/query.ts:query() → queryLoop()        │
│ Level 4  流式 API        src/QueryEngine.ts → services/api/claude  │
│ Level 5  工具执行        services/tools/toolOrchestration.runTools  │
│                          → StreamingToolExecutor → tool.call()     │
│ Level 6  渲染回流        utils/messages.handleMessageFromStream     │
│                          → Messages.tsx → ink → stdout             │
└────────────────────────────────────────────────────────────────────┘
```

## 1. Level 0 — 进程入口

### 1.1 启动脚本

`package.json:9-12`：

```json
"scripts": {
  "dev":   "bun run ./src/dev-entry.ts",
  "start": "bun run ./src/dev-entry.ts"
}
```

- 真实构建中 bun 把 CLI 入口绑到 `src/entrypoints/cli.tsx`；开发模式用 `src/dev-entry.ts` 包一层做 profile / 热重载。
- `src/main.tsx`（约 8000+ 行）负责：CLI 解析（commander）、feature flag 加载、profile checkpoint（`profileCheckpoint` 在 `src/utils/startupProfiler.ts`）、MCP/keychain 预取、worktree 快速路径（`main.tsx:247-261`）。

### 1.2 REPL 启动路径

`src/replLauncher.tsx:12-22`（极简转发器）：

```ts
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

- 动态 import 是为了避开循环依赖并把 ~840KB 的 `screens/REPL.js` 拆成独立 chunk（首屏只加载壳）。
- `App` 是 `src/components/App.tsx` 的最外层（提供 StatsProvider、NotificationsProvider、ModalProvider 等 Context）。
- `renderAndRun` 由 `entrypoints/cli.tsx` 注入，本质是 `ink.render(element)`。

`src/screens/REPL.tsx:2800-2835` 是主交互容器：

```ts
// 关键配置加载
const [,, defaultSystemPrompt, baseUserContext, systemContext] = await Promise.all([
  checkAndDisableBypassPermissionsIfNeeded(...),
  feature('TRANSCRIPT_CLASSIFIER') ? checkAndDisableAutoModeIfNeeded(...) : undefined,
  getSystemPrompt(freshTools, mainLoopModelParam, ...),
  getUserContext(),
  getSystemContext()
])

// 核心循环入口
for await (const event of query({
  messages, systemPrompt, userContext, systemContext,
  canUseTool, toolUseContext, querySource: getQuerySourceForREPL()
})) {
  onQueryEvent(event)   // ← 每一帧流入 UI
}
```

`getQuerySourceForREPL()` 标记当前 query 是 `repl_main` / `repl_compact` / `named_agent` 等之一（`src/constants/querySource.ts`），用于上游区分行为。

## 2. Level 1 — REPL 内部的"提交"协议

### 2.1 输入组件

`src/components/PromptInput/PromptInput.tsx:984-1110` 暴露 `onSubmit`：

```ts
const onSubmit = useCallback(async (inputParam, isSubmittingSlashCommand = false) => {
  // 1. 处理 typeahead 建议
  // 2. 处理 queued commands
  // 3. 实际调用：
  await onSubmitProp(inputParam, {
    /* PromptInputHelpers: clearBuffer, resetHistory, … */
  })
}, [...])
```

`PromptInput` 内部是 ink `<TextInput>`，原生 readline 风格（编辑/历史/粘贴/补全都由 `useTextInput` hook 提供）。

### 2.2 REPL 把 onSubmit 桥到 handlePromptSubmit

`src/screens/REPL.tsx` 内部把 `onSubmit` 路由到 `handlePromptSubmit`：

```ts
// 隐式调用链：PromptInput.onSubmit → onSubmitProp → REPL 内部 handler
// → handlePromptSubmit(params)
```

`src/utils/handlePromptSubmit.ts:120-211` 干的事：

1. **空输入早退**（`:188`）。
2. **退出命令拦截**（`exit` / `quit` / `:q`）→ 派发 `/exit` 命令。
3. **解析 reference**（图片/粘贴文本）→ `parseReferences` + `expandPastedTextRefs`。
4. **slash command 识别**（`!skipSlashCommands` 时不触发）→ 命中"local-jsx immediate"则直接 render JSX（如 `/config`、`/doctor`），不走 query。
5. **忙时入队**（`queryGuard.isActive`）→ 走 `enqueue({...})`，等当前 turn 跑完再消费。

非 immediate、非空、非退出、非忙 → 走 **`executeUserInput`**（`handlePromptSubmit.ts:396`）：

```ts
async function executeUserInput(params) {
  // 1. queryGuard.reserve()  → 拿 turn 占用
  // 2. onBeforeQuery?.()      → 业务前置钩子
  // 3. setAppState({ mainLoopModel: chosenModel })
  // 4. onQuery(messages, abortController, ...)
}
```

`onQuery` 是 REPL 注入的回调（`REPL.tsx:2887-2986`），最终触发 `onQueryImpl`。

## 3. Level 2 — processUserInput / processTextPrompt

### 3.1 路由分发

`src/utils/processUserInput/processUserInput.ts:149-525`：

```ts
queryCheckpoint('query_process_user_input_base_start')
// ...
if (queuedCommands?.length) {
  await processQueuedCommands(...)
} else {
  // 文本
  await processTextPrompt(...)
}
queryCheckpoint('query_process_user_input_base_end')

// 前置钩子
queryCheckpoint('query_hooks_start')
// Stop / PreUserSubmit / UserPromptSubmit hook
queryCheckpoint('query_hooks_end')
```

- `processTextPrompt`（`src/utils/processUserInput/processTextPrompt.ts`）做：图片 resize / paste 处理 / 解析 `@file` reference / 构造用户消息。

### 3.2 processTextPrompt 核心

`processTextPrompt.ts` 调 `createUserMessage(...)` 生成消息，调 `onQuery`（=REPL 的 `onQuery`），把消息塞进主循环。**到此 LLM 还没有被调用**——只构造好了 `messages` 数组 + `canUseTool` 上下文。

## 4. Level 3 — query() 主循环

### 4.1 入口

`src/query.ts:1850`：

```ts
export async function* query(params: QueryParams): AsyncGenerator<...> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // 仅在正常返回时，触发已消费命令的 completed 通知
  for (const uuid of consumedCommandUuids) notifyCommandLifecycle(uuid, 'completed')
  return terminal
}
```

`queryLoop`（`query.ts:1885`）是真正的多轮循环：每轮调用一次 LLM，解析 `tool_use` → 走工具 → 收集 `tool_result` → 进入下一轮，直到模型不再发 `tool_use` 或命中终止条件。

### 4.2 关键状态字段

`QueryParams` 包含（`query.ts:1896-1900`）：

```ts
const {
  systemPrompt, userContext, systemContext, canUseTool,
  toolUseContext, querySource, maxTurns, ...
} = params
```

- `canUseTool`：经 `getCanUseToolFn` 包装的权限判定 + hook 链。
- `toolUseContext`：包含 `tools / mcpTools / agentDefinitions / abortController / getAppState / setAppState`。

### 4.3 每轮迭代（`queryLoop` 主体）

`src/query.ts:584-640` 启动流式 API：

```ts
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    getToolPermissionContext: async () => toolUseContext.getAppState().toolPermissionContext,
    model: executionState.currentModel,
    fastMode: appState.fastMode,                          // 极速模式
    fallbackModel,                                        // 失败时降级
    onStreamingFallback: () => { streamingFallbackOccured = true },
    querySource,
    agents: toolUseContext.options.agentDefinitions.activeAgents,
    allowedAgentTypes: toolUseContext.options.agentDefinitions.allowedAgentTypes,
    effortValue: appState.effortValue,
    agentId: toolUseContext.agentId,
    taskBudget: ...,                                      // 自治任务预算
  },
})) { ... }
```

`deps.callModel` 的实现在 `src/QueryEngine.ts:518-808`（`apiQueryLoop`）—— 它把消息转给 `queryModel` 真正发 HTTP 请求到 Anthropic API，并把流式事件 yield 出来。

### 4.4 流式事件分类

`QueryEngine.ts:518-808` 内：

- `message_start` → 记录 `usage`（input tokens）。
- `content_block_start`（`type: 'text' | 'tool_use' | 'thinking' | ...`）→ 初始化累积器。
- `content_block_delta`（`text_delta` / `input_json_delta` / `thinking_delta`）→ 累加。
- `content_block_stop` → 把累积器打包成 `AssistantMessage` yield。
- `message_delta`（带 `stop_reason`）→ 结束本轮。
- `message_stop` → 标记 done。

## 5. Level 4 — 真正的 HTTP / Anthropic SDK

### 5.1 入口

`src/services/api/claude.ts`：

- `queryModelWithStreaming({ ... })`（`claude.ts:752`）— 流式版本。
- `queryModelWithoutStreaming({ ... })`（`claude.ts:709`）— 非流式（用于 verify、skill 改进等侧通道）。
- `queryModel`（`claude.ts:2932`）— orchestrator，组装请求体 → 调 SDK → 处理流。

### 5.2 SSE 解析

SSE（`event: ...` / `data: {...}`）通过 `claude.ts:2053-2202` 的 `content_block_*` switch 解析，关键分支：

```ts
case 'content_block_start':
  switch (part.content_block.type) {
    case 'tool_use': /* 记录 id + name + 准备 input 累加器 */
    case 'text':     /* 准备 text 累加器 */
    case 'thinking': /* 准备 thinking 累加器 */
  }
  break
case 'content_block_delta':
  switch (delta.type) {
    case 'text_delta':       onUpdateLength(delta.text)
    case 'input_json_delta': toolInputAccumulator += delta.partial_json
    case 'thinking_delta':   thinkingAccumulator += delta.thinking
  }
  break
case 'content_block_stop':
  // 把累加器打包成 AssistantMessage yield 给上游
```

### 5.3 Bedrock / Vertex 兼容

`claude.ts` 通过 `getAPIProvider()` 分发到：

- `firstParty` → Anthropic 官方 Anthropic SDK
- `bedrock` → AWS Bedrock ConverseStream
- `vertex` → GCP Vertex Anthropic
- `foundry` → Azure AI Foundry

这部分对调用方透明。

## 6. Level 5 — 工具执行

### 6.1 runTools 总入口

`src/services/tools/toolOrchestration.ts:19`：

```ts
export async function* runTools(
  toolUseBlocks, assistantMessages, canUseTool, toolUseContext
): AsyncGenerator<...>
```

被 `query.ts:1395` 在检测到 `executionState.toolUseBlocks.length > 0` 时调用：

```ts
: runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

### 6.2 StreamingToolExecutor：并发与依赖

`src/services/tools/StreamingToolExecutor.ts:40`：

```ts
export class StreamingToolExecutor { ... }
```

- 维护一组"待执行" tool_use 块。
- 已声明可并发（`isConcurrencySafe() === true`）的块立刻并行 `Promise.all`。
- 串行工具按声明顺序逐个执行。
- 每一块走完整流程：PreToolUse hook → 权限判定 → `tool.call()` → PostToolUse hook → 构 `tool_result` 消息 yield。

### 6.3 canUseTool 包装器

`canUseTool`（`QueryEngine.ts:660` 装配）合并了：

- `runPreToolUseHooks` — 钩子链（用户 settings 里的 hook）。
- 权限规则检查（`allow` / `ask` / `deny`）。
- `autoMode` 分类器（如果启用了 auto mode）。
- 真实的 `tool.call(input, context)` 触发器。

权限被拒时，工具不真执行，而是返回结构化 `permission_denied` 的 `tool_result`，模型在下一轮看到后重试或道歉。

### 6.4 tool.call — 实际工具

以 `FileWriteTool` 为例（写 `index.html`）：

`src/tools/FileWriteTool/FileWriteTool.ts` 定义 `buildTool({ name, call })`：

```ts
async call({ file_path, content }, context) {
  // 1. 沙箱检查
  // 2. 写文件
  // 3. 返回 { data: { filePath, content } }
}
```

返回的 `data` 经 `mapToolResultToToolResultBlockParam` 转成 Anthropic API 期望的 `tool_result` 块（`type: 'tool_result', tool_use_id, content`），由 `queryLoop` 写入 `messages` 进入下一轮。

## 7. Level 6 — 渲染回流

### 7.1 事件流回 REPL

`onQueryEvent`（`REPL.tsx:2616-2692`）：

```ts
const onQueryEvent = useCallback((event) => {
  handleMessageFromStream(
    event,
    newMessage => { setMessages(old => [...old, newMessage]) },     // 完整消息
    newContent => { setResponseLength(L => L + newContent.length) },// 增量长度
    setStreamMode, setStreamingToolUses, /* tombstone */,
    setStreamingThinking, /* metrics */, onStreamingText
  )
}, [setMessages, setResponseLength, ...])
```

### 7.2 handleMessageFromStream

`src/utils/messages.ts:2930-2980`：

- 收到 `stream_event`（partial delta）→ 不入 `messages` 数组，只更新 `responseLength` 让 spinner 动起来。
- 收到完整 `assistant` 消息 → 入 `messages` 数组。
- 收到 `tombstone` → 删消息（用于流式回退时丢弃孤儿消息）。
- 收到 `tool_result` → 入 `messages` 数组。
- 收到 `progress`（ephemeral 工具的 tick）→ 替换前一条同名 progress，不堆积。

### 7.3 Ink 渲染

`src/components/messages/Messages.tsx` 接收 `messages` 数组，按 `type` 分发到具体渲染器：

- `assistant` → `<AssistantMessage>`（含 thinking / text / tool_use 三个子组件）
- `user` → `<UserMessage>`
- `tool_result` → `<ToolResultMessage>`
- `progress` → `<ProgressMessage>`
- `attachment` → `<AttachmentMessage>`

ink 把 React 元素 diff 到 stdout：

- 文本用 ANSI 控制字符原地刷新（`figures.tick` / 颜色 / 粗体）。
- 工具块以"框 + 折叠/展开"形式呈现，权限 prompt 直接占满底部。

### 7.4 终止

- 模型不再生成 `tool_use` 且没有 `isApiError` → `queryLoop` 走完最后一轮 → `query()` 收尾 → `onQueryImpl` 调 `resetLoadingState()`。
- `messages` 数组里的最后一条 `assistant` 消息被 ink 渲染成最终回复——用户看到的就是模型直接吐出的 Markdown（代码块、表格、列表都来自这个 `AssistantMessage.message.content` 中的 `text` 块）。

## 8. 完整时序图

```
User      PromptInput    handlePromptSubmit    processUserInput    query()    QueryEngine    Anthropic API    runTools    Messages.tsx
 │            │                 │                     │               │             │                │            │           │
 │ Enter键    │                 │                     │               │             │                │            │           │
 │───────────▶│                 │                     │               │             │                │            │           │
 │            │ onSubmit(input) │                     │               │             │                │            │           │
 │            │────────────────▶│                     │               │             │                │            │           │
 │            │                 │ executeUserInput    │               │             │                │            │           │
 │            │                 │────────────────────▶│               │             │                │            │           │
 │            │                 │                     │ onQuery       │             │                │            │           │
 │            │                 │                     │──────────────▶│             │                │            │           │
 │            │                 │                     │               │ queryLoop   │                │            │           │
 │            │                 │                     │               │────────────▶│                │            │           │
 │            │                 │                     │               │  callModel │                │            │           │
 │            │                 │                     │               │             │ POST /messages  │            │           │
 │            │                 │                     │               │             │───────────────▶│            │           │
 │            │                 │                     │               │             │                │ SSE stream │           │
 │            │                 │                     │               │             │◀───────────────│            │           │
 │            │                 │                     │               │ yield stream_events            │           │
 │            │                 │                     │               │─────────────────────────────▶│           │
 │            │                 │                     │               │ AssistantMessage{text:"..."}   │           │
 │            │                 │                     │               │───────────────────────────────────▶│       │
 │            │                 │                     │               │             │  tool_use{...}  │           │
 │            │                 │                     │               │ runTools    │                │            │           │
 │            │                 │                     │               │──────────────────────────────────────────▶│
 │            │                 │                     │               │             │  PermissionRequest       │      │
 │            │                 │                     │               │             │  (user approves)         │      │
 │            │                 │                     │               │ tool.call()│                │            │           │
 │            │                 │                     │               │ tool_result: write success  │           │
 │            │                 │                     │               │───────────────────────────────────▶│       │
 │            │                 │                     │               │ for await (next round) …    │           │
 │            │                 │                     │               │  (最终轮无 tool_use)         │           │
 │            │                 │                     │               │  yield AssistantMessage     │           │
 │            │                 │                     │               │───────────────────────────────────▶│       │
 │            │                 │                     │               │             │  resetLoading   │           │
 │            │                 │                     │               │ return term │                │           │
 │            │                 │                     │               │────────────▶│                │           │
 │            │                 │                     │               │  渲染最终消息到 ink → stdout  │           │
 │            │                 │                     │               │──────────────────────────────────────────▶│
```

## 9. 文件 / 方法速查表（按调用顺序）

| 阶段 | 入口 / 方法 | 文件 : 行 |
| --- | --- | --- |
| 启动 | `bun run dev-entry.ts` | `package.json:9-12` |
| 启动 | `main()` | `src/main.tsx` |
| 启动 | `launchRepl` | `src/replLauncher.tsx:12` |
| 启动 | `<App><REPL/></App>` | `src/replLauncher.tsx:19` |
| 输入 | `PromptInput` | `src/components/PromptInput/PromptInput.tsx:194` |
| 输入 | `onSubmit` | `src/components/PromptInput/PromptInput.tsx:984` |
| 提交 | `handlePromptSubmit` | `src/utils/handlePromptSubmit.ts:120` |
| 提交 | `executeUserInput` | `src/utils/handlePromptSubmit.ts:396` |
| 提交 | `processUserInput` | `src/utils/processUserInput/processUserInput.ts:149` |
| 提交 | `processTextPrompt` | `src/utils/processUserInput/processTextPrompt.ts` |
| 提交 | `onQuery` (REPL) | `src/screens/REPL.tsx:2887` |
| 提交 | `onQueryImpl` | `src/screens/REPL.tsx:2693` |
| Query | `query()` | `src/query.ts:1850` |
| Query | `queryLoop()` | `src/query.ts:1885` |
| Query | `deps.callModel` | `src/query.ts:591` |
| Query | `apiQueryLoop` | `src/QueryEngine.ts:518-808` |
| API | `queryModelWithStreaming` | `src/services/api/claude.ts:752` |
| API | `queryModel` | `src/services/api/claude.ts:2932` |
| API | SSE 解析 | `src/services/api/claude.ts:2053-2202` |
| 工具 | `runTools` | `src/services/tools/toolOrchestration.ts:19` |
| 工具 | `StreamingToolExecutor` | `src/services/tools/StreamingToolExecutor.ts:40` |
| 工具 | `canUseTool` (QueryEngine 装配) | `src/QueryEngine.ts:660` |
| 工具 | `tool.call()` | 各 `src/tools/<Name>/<Name>Tool.ts` |
| 渲染 | `onQueryEvent` | `src/screens/REPL.tsx:2616` |
| 渲染 | `handleMessageFromStream` | `src/utils/messages.ts:2930` |
| 渲染 | `<Messages>` | `src/components/messages/Messages.tsx` |
| 渲染 | ink → stdout | `ink.render` 在 `src/entrypoints/cli.tsx` |

## 10. 一次"贪吃蛇"请求的完整调用链

1. 用户在终端敲 `claude` → `bun run dev-entry.ts` → `main()` → `launchRepl`。
2. 终端出现 `▌` 提示符，ink 启动 React 渲染循环。
3. 用户键入 `帮我生成一个贪吃蛇的游戏` → `Enter`。
4. `PromptInput.onSubmit` 触发 → `handlePromptSubmit({ input: '…' })`：
   - 非空、非 slash、非 exit → 走 `executeUserInput`。
   - 不在忙状态 → 同步进入 `onQuery`。
5. `processUserInput` → `processTextPrompt` → `createUserMessage({ content: '…' })` → `onQuery(messages, ...)`。
6. REPL `onQueryImpl` → 加载 system prompt / tools / mcp / 构造 `toolUseContext`。
7. `for await (event of query({...}))` 启动：
   - **Round 1**：`deps.callModel` → `apiQueryLoop` → `queryModel` → POST `/v1/messages`（带 systemPrompt + tools + 历史）。
   - SSE 回流：模型说"我来分析需求并设计代码" + 输出 thinking 块 + 输出 text 块 + 1 个 `tool_use: Write{file_path: 'index.html', content: '<!DOCTYPE html>…'}`。
   - 完整 assistant message + tool_use 块通过 `handleMessageFromStream` → `setMessages` → 实时显示在 UI。
8. `runTools` 触发 `StreamingToolExecutor`：
   - PreToolUse hook（如有）→ 权限对话框（自动 mode 下分类器决议；普通 mode 下用户按 y/n）。
   - 用户批准 → `FileWriteTool.call(...)` → 磁盘写入。
   - PostToolUse hook → 返回 `{ data: { filePath, content } }` → `mapToolResultToToolResultBlockParam` → 构 `tool_result` 块。
9. 回到 `queryLoop` 下一轮：
   - 把 `tool_result` 消息加到 messages。
   - **Round 2**：再次 `callModel` → 模型说"再写一个 style.css 和 game.js" → 两个 `tool_use: Write` 块。
   - 并发跑（`isConcurrencySafe() === true`）→ 同时落盘两个文件。
10. **Round N** 模型不再发 `tool_use` → 最后一轮直接吐文字（"贪吃蛇游戏已生成，运行 `open index.html` 即可"）→ `queryLoop` 终止。
11. `query()` 收尾：`onQueryImpl` 调 `resetLoadingState()`，移除 spinner。
12. 用户在终端看到完整回复 + 中间过程的工具展开块。

## 11. 设计要点与权衡

| 决策 | 价值 | 源码依据 |
| --- | --- | --- |
| CLI → REPL 用动态 import | 拆 chunk，减少首屏 | `replLauncher.tsx:14-18` |
| 工具 / hook / 权限在同一 `canUseTool` 中串联 | 单点扩展，统一拦截 | `QueryEngine.ts:660` |
| Tool 写成 `buildTool({ call, ... })` 高阶函数 | 自动获得 maxResultSizeChars / 并发标识 / 分类器输入等元数据 | 各 `src/tools/*/*.ts` |
| `StreamingToolExecutor` 把多 tool_use 块并发化 | 一个回合能落多文件 | `StreamingToolExecutor.ts:40` |
| 流式事件回 UI 用 `handleMessageFromStream` 集中处理 | 完整消息入 `messages`、partial 只更新长度 | `messages.ts:2930` |
| `ephemeral` 工具的 progress 替换而非追加 | 防止 13k+ sleep_progress 把 messages 撑爆 | `REPL.tsx:2640-2659` |
| Tombstone 消息支持流式回退 | 不会留下半截"孤儿"消息 | `messages.ts:2955`, `query.ts:643` |
| `queryGuard.reserve()` 串行化 turn | 防止多输入并发改 AppState | `handlePromptSubmit.ts` 入口 |
| `runPreToolUseHooks` 阻塞时可改写工具输入 | hook 能力上限不止于"否决"，还能"变换" | `utils/hooks.ts:691, 741` |
| Bedrock / Vertex / Foundry 透明分派 | 用户零成本切到任意 provider | `services/api/claude.ts` |

## 12. 一句话总结

> Claude Code 的"从键入到输出"是一条**React/ink 驱动的流式管道**：PromptInput 收集文本 → handlePromptSubmit / processUserInput 包装消息 → query() 进入多轮 queryLoop → 每轮 QueryEngine 通过 Anthropic SDK 拉 SSE → 流式事件由 handleMessageFromStream 分流（完整消息入 messages、partial 只动 length）→ tool_use 块经 StreamingToolExecutor 并发执行并把 tool_result 喂回下一轮 → 最后一轮无 tool_use 时模型吐最终文字 → React 重渲输出到 stdout。整条管线以 AsyncGenerator 为骨架、checkpoint 为观测点，**让"打字 → 思考 → 写文件 → 打字"在同一屏上无中断**地呈现给用户。
