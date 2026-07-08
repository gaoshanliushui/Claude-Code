# StreamingToolExecutor —— 流式工具并发执行器

> 源码位置：`src/services/tools/StreamingToolExecutor.ts`（531 行）
> 调用方：`src/query.ts` 的 `executeModelStreamingTurn`（仅在 feature gate `streamingToolExecution` 启用时实例化）
> 兜底路径：当 streaming executor 不可用时，`src/query.ts:processFollowUpTurn` 退回到同步 `runTools`
> 关联文档：[`docs/09-query.md`](../09-query.md) · [`docs/11-tool-execution.md`](11-tool-execution.md)

`StreamingToolExecutor` 是 Claude Code 在流式（streaming）调用模型时，**边接收 `tool_use` 块边并发执行工具**的协调器。它把"模型还在流式输出 → 工具已经并行跑起来 → 结果按接收顺序回流"这条流水线串成一条可恢复、可中断、可清理的状态机。

---

## 目录

1. [职责定位](#1-职责定位)
2. [数据结构](#2-数据结构)
3. [整体流程](#3-整体流程)
4. [并发控制策略](#4-并发控制策略)
5. [取消与中断传播](#5-取消与中断传播)
6. [进度消息的优先输出](#6-进度消息的优先输出)
7. [结果回流与有序性保证](#7-结果回流与有序性保证)
8. [Streaming Fallback 处理](#8-streaming-fallback-处理)
9. [与 query.ts 的协作时序](#9-与-queryts-的协作时序)
10. [关键设计取舍](#10-关键设计取舍)
11. [已知限制](#11-已知限制)

---

## 1. 职责定位

Claude Code 调用模型时采用 **streaming** 模式：API 一边生成 token，一边把 `tool_use` 块推过来。理想情况下，**模型流式输出与工具并行执行** —— 减少用户感知的端到端延迟。

`StreamingToolExecutor` 解决 4 个核心问题：

| 问题 | 解法 |
|------|------|
| 工具可能**并发不安全**（如 `Edit` 改同一文件、`Bash` 共享 cwd） | `isConcurrencySafe` 标识 + 互斥区 |
| 工具结果必须**按用户消息顺序**回流给模型 | `getCompletedResults` 按入队顺序 yield |
| **Bash 错误**应让其他并行子进程立即退出 | `siblingAbortController` 级联取消 |
| 流式输出可能中途被 **fallback 到另一个模型**，旧结果需丢弃 | `discard()` 标记，所有 in-flight 工具转 synthetic error |

文件顶部注释（**`StreamingToolExecutor.ts:34-39`**）定义了三条契约：

> Executes tools as they stream in with concurrency control.
> - Concurrent-safe tools can execute in parallel with other concurrent-safe tools
> - Non-concurrent tools must execute alone (exclusive access)
> - Results are buffered and emitted in the order tools were received

---

## 2. 数据结构

### 2.1 `TrackedTool` —— 单个工具的完整生命周期记录

```ts
// StreamingToolExecutor.ts:21-32
type TrackedTool = {
  id: string                          // tool_use_id（API 返回的 id）
  block: ToolUseBlock                 // 原始 tool_use 块（name + input）
  assistantMessage: AssistantMessage  // 触发它的 assistant 消息
  status: ToolStatus                  // queued / executing / completed / yielded
  isConcurrencySafe: boolean          // 是否可与其他并发安全工具并行
  promise?: Promise<void>             // 收集结果的 async 任务（用于 await）
  results?: Message[]                 // 已收集的非进度结果
  pendingProgress: Message[]          // 进度消息（独立通道，立即 yield）
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>
}
```

### 2.2 `ToolStatus` —— 4 态机

```ts
// StreamingToolExecutor.ts:19
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

状态转移图：

```
       addTool()
          ↓
       queued ──→ executing ──→ completed ──→ yielded
                                  (getCompletedResults 取走)
```

- `queued` —— 已入队，等待并发条件允许
- `executing` —— `runToolUse` 正在跑
- `completed` —— 已收集完所有 `update.message`，等待 yield
- `yielded` —— 结果已被 `getCompletedResults` 消费，永不再 yield

### 2.3 Executor 实例状态

```ts
// StreamingToolExecutor.ts:41-51
class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  private toolUseContext: ToolUseContext
  private hasErrored = false                       // 是否有一个 Bash 失败
  private erroredToolDescription = ''             // 用于 synthetic error 信息
  private siblingAbortController: AbortController // 子控制器（见 §5）
  private discarded = false                        // streaming fallback 标记
  private progressAvailableResolve?: () => void   // 进度唤醒信号
}
```

`tools` 数组**保留入队顺序** —— `getCompletedResults` 遍历它来保证结果顺序。

---

## 3. 整体流程

### 3.1 高层时序

```
queryLoop
  │
  ├─[模型 streaming 输出]── tool_use 块 A、B、C 陆续到达
  │                              ↓
  │                  streamingToolExecutor.addTool(block)
  │                              ↓
  │                  tools.push(...); processQueue()
  │                              ↓
  │                  [A 启动] [B 启动] [C 排队，等 A 或 B 完成]
  │                              ↓
  │                  yield* getCompletedResults()  // 边收边 yield
  │                              ↓
  │                  [stream 结束]
  │                              ↓
  │                  yield* getRemainingResults()  // 等剩下的
  │                              ↓
  │                  [所有结果回流] → 进入下一轮
```

### 3.2 方法调用链

```
addTool(block, assistantMessage)              // 接收新工具
  ├─ findToolByName → toolDefinition
  ├─ 若不存在 → 直接 push 一个 completed tool（含 synthetic error）
  ├─ toolDefinition.inputSchema.safeParse(block.input)
  │     ├─ 解析成功 → 调用 toolDefinition.isConcurrencySafe(parsedInput.data)
  │     └─ 解析失败 → isConcurrencySafe = false（保守降级）
  └─ void this.processQueue()
        └─ 遍历 tools，对 queued 且 canExecuteTool=true 的工具：
              await this.executeTool(tool)
                ├─ 创建 per-tool child abortController
                ├─ 调用 runToolUse(block, ..., {abortController})
                ├─ for await (update of generator)
                │     ├─ 错误检测 → 标记 / 级联取消
                │     ├─ 进度消息 → pendingProgress
                │     └─ 结果消息 → messages
                └─ tool.status = 'completed'
```

---

## 4. 并发控制策略

### 4.1 `canExecuteTool` 判定

```ts
// StreamingToolExecutor.ts:129-135
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return (
    executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
  )
}
```

翻译：

| 当前 executing 情况 | 新工具 isConcurrencySafe | 能否启动 |
|--------------------|--------------------------|---------|
| 空 | * | ✅ |
| 全是 concurrencySafe | true | ✅（并行加入） |
| 全是 concurrencySafe | false | ❌（独占：等当前清空） |
| 有非 concurrencySafe | true | ❌（独占：等非并发工具完成） |
| 有非 concurrencySafe | false | ❌（独占：等） |

### 4.2 `processQueue` 遍历逻辑

```ts
// StreamingToolExecutor.ts:140-151
private async processQueue(): Promise<void> {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue

    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)
    } else {
      // Can't execute this tool yet, and since we need to maintain order for non-concurrent tools, stop here
      if (!tool.isConcurrencySafe) break
    }
  }
}
```

**关键点**：当遇到一个**非并发**工具但当前无法执行时，`break` 跳出循环 —— 因为非并发工具有顺序约束，不能"跳过它去执行后面的"。

### 4.3 `isConcurrencySafe` 的判定时机

```ts
// StreamingToolExecutor.ts:104-113
const parsedInput = toolDefinition.inputSchema.safeParse(block.input)
const isConcurrencySafe = parsedInput?.success
  ? (() => {
      try {
        return Boolean(toolDefinition.isConcurrencySafe(parsedInput.data))
      } catch {
        return false
      }
    })()
  : false
```

工具可以**基于输入**判定并发安全性（不是只看工具名）：

- `Edit` 改 `foo.ts` 与 `Edit` 改 `bar.ts` 可以并行；
- `Edit` 改 `foo.ts` 与 `Edit` 改 `foo.ts` 不应并行；
- `Bash(npm test)` 与 `Bash(npm install)` 同 cwd 不应并行。

各工具的 `isConcurrencySafe(input)` 实现：

| 工具 | 行为 |
|------|------|
| `TodoWrite` | true（无副作用） |
| `Read` / `Glob` / `Grep` | true（只读） |
| `WebFetch` / `WebSearch` | true（独立网络 IO） |
| `Edit` / `Write` | 通常 true（但每个工具内部仍可能锁文件） |
| `Bash` | 由 `BashTool.isConcurrencySafe` 判定（同 cwd、互斥命令前缀等） |

---

## 5. 取消与中断传播

这是整个文件最精密的部分。Claude Code 有 **3 层 AbortController 嵌套**：

```
parentContext.abortController (query.ts 的主控制器)
        │
        ├─ StreamingToolExecutor.siblingAbortController  (level 1)
        │       │
        │       ├─ toolAbortController (per-tool, level 2)
        │       │       │
        │       │       └─ Bash 子进程 listen 的是 toolAbortController.signal
        │       │
        │       └─ toolAbortController (另一个工具)
```

### 5.1 创建时机

```ts
// StreamingToolExecutor.ts:59-61（构造时）
this.siblingAbortController = createChildAbortController(
  toolUseContext.abortController,
)
```

```ts
// StreamingToolExecutor.ts:301-303（每个工具启动时）
const toolAbortController = createChildAbortController(
  this.siblingAbortController,
)
```

### 5.2 Bash 错误级联取消

```ts
// StreamingToolExecutor.ts:354-364
if (isErrorResult) {
  thisToolErrored = true
  // Only Bash errors cancel siblings. Bash commands often have implicit
  // dependency chains (e.g. mkdir fails → subsequent commands pointless).
  // Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.
  if (tool.block.name === BASH_TOOL_NAME) {
    this.hasErrored = true
    this.erroredToolDescription = this.getToolDescription(tool)
    this.siblingAbortController.abort('sibling_error')
  }
}
```

**关键设计决策**：只有 `Bash` 错误才触发级联取消。原因（源码注释）：

> Only Bash errors cancel siblings. Bash commands often have implicit dependency chains (e.g. mkdir fails → subsequent commands pointless). Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.

**级联路径**：
```
Bash tool error
  → this.siblingAbortController.abort('sibling_error')
  → 所有其他 toolAbortController.signal 收到 abort 事件
  → 它们的 Bash 子进程（如果监听 signal）退出
  → for await 循环下次检查 getAbortReason() → 'sibling_error'
  → createSyntheticErrorMessage 生成 "Cancelled: parallel tool call X errored"
```

### 5.3 abort 冒泡到 query 控制器

每个 per-tool controller 注册一个 abort 监听器：

```ts
// StreamingToolExecutor.ts:304-318
toolAbortController.signal.addEventListener(
  'abort',
  () => {
    if (
      toolAbortController.signal.reason !== 'sibling_error' &&
      !this.toolUseContext.abortController.signal.aborted &&
      !this.discarded
    ) {
      this.toolUseContext.abortController.abort(
        toolAbortController.signal.reason,
      )
    }
  },
  {once: true},
)
```

**冒泡条件**（三个都不满足才冒泡）：
- reason ≠ `'sibling_error'` —— 级联取消不应终结整个 turn；
- query 主 controller 未 aborted —— 避免重复 abort；
- executor 未 discarded —— fallback 期间不冒泡。

**典型冒泡场景**：用户在权限弹窗里按 ESC 拒绝 —— `PermissionContext.ts: cancelAndAbort` 会 abort 当前工具的 controller，进而冒泡到 query 主 controller，触发 query.ts 的 post-tool abort 检查并结束本轮。

注释解释了设计原因（**`StreamingToolExecutor.ts:296-300`**）：

> Permission-dialog rejection also aborts this controller (PermissionContext.ts cancelAndAbort) — that abort must bubble up to the query controller so the query loop's post-tool abort check ends the turn. Without bubble-up, ExitPlanMode "clear context + auto" sends REJECT_MESSAGE to the model instead of aborting (#21056 regression).

### 5.4 用户中断 vs 中断行为

每个工具有 `interruptBehavior: 'cancel' | 'block'`（默认 `'block'`）：
- `'block'` —— 新消息等待当前工具完成；
- `'cancel'` —— 新消息到达时立即取消。

```ts
// StreamingToolExecutor.ts:210-241
private getAbortReason(tool): 'sibling_error' | 'user_interrupted' | 'streaming_fallback' | null {
  if (this.discarded) return 'streaming_fallback'
  if (this.hasErrored) return 'sibling_error'
  if (this.toolUseContext.abortController.signal.aborted) {
    // 'interrupt' means the user typed a new message while tools were running.
    if (this.toolUseContext.abortController.signal.reason === 'interrupt') {
      return this.getToolInterruptBehavior(tool) === 'cancel' ? 'user_interrupted' : null
    }
    return 'user_interrupted'
  }
  return null
}
```

**判定优先级**：`streaming_fallback` > `sibling_error` > `user_interrupted` > null。

`user_interrupted` 的 synthetic error 信息用 `REJECT_MESSAGE`（带 memory correction hint），UI 显示 "User rejected edit" 而非 "Error editing file"：

```ts
// StreamingToolExecutor.ts:160-172
if (reason === 'user_interrupted') {
  return createUserMessage({
    content: [{
      type: 'tool_result',
      content: withMemoryCorrectionHint(REJECT_MESSAGE),
      is_error: true,
      tool_use_id: toolUseId,
    }],
    toolUseResult: 'User rejected tool use',
    sourceToolAssistantUUID: assistantMessage.uuid,
  })
}
```

### 5.5 `updateInterruptibleState` —— 中断可见性

```ts
// StreamingToolExecutor.ts:254-260
private updateInterruptibleState(): void {
  const executing = this.tools.filter(t => t.status === 'executing')
  this.toolUseContext.setHasInterruptibleToolInProgress?.(
    executing.length > 0 &&
    executing.every(t => this.getToolInterruptBehavior(t) === 'cancel'),
  )
}
```

UI 层据此判断：当前是否可以显示 "按 Ctrl+C 立即取消" 的提示。

---

## 6. 进度消息的优先输出

`runToolUse` 通过 `AsyncGenerator<MessageUpdateLazy>` 流式产出两类消息：
- **进度消息**（`type: 'progress'`）—— 用于实时显示（如 Bash 跑 30 秒时的进度条）；
- **结果消息**（`type: 'user'` 含 `tool_result`）—— 最终结果。

`StreamingToolExecutor` 把它们分流：

```ts
// StreamingToolExecutor.ts:366-381
if (update.message) {
  // Progress messages go to pendingProgress for immediate yielding
  if (update.message.type === 'progress') {
    tool.pendingProgress.push(update.message)
    // Signal that progress is available
    if (this.progressAvailableResolve) {
      this.progressAvailableResolve()
      this.progressAvailableResolve = undefined
    }
  } else {
    messages.push(update.message)
  }
}
```

**`getCompletedResults` 中 progress 立即 yield**（不管 status）：

```ts
// StreamingToolExecutor.ts:417-422
for (const tool of this.tools) {
  // Always yield pending progress messages immediately, regardless of tool status
  while (tool.pendingProgress.length > 0) {
    const progressMessage = tool.pendingProgress.shift()!
    yield {message: progressMessage, newContext: this.toolUseContext}
  }
  ...
}
```

`progressAvailableResolve` 是一个一次性 promise resolver：

```ts
// StreamingToolExecutor.ts:477-484
// Also wait for progress to become available
const progressPromise = new Promise<void>(resolve => {
  this.progressAvailableResolve = resolve
})

if (executingPromises.length > 0) {
  await Promise.race([...executingPromises, progressPromise])
}
```

`Promise.race` 让 `getRemainingResults` 在**任一工具产出进度**或**任一工具完成**时立即醒来，而不是等到某个工具全部跑完。

---

## 7. 结果回流与有序性保证

### 7.1 `getCompletedResults` —— 同步 generator

```ts
// StreamingToolExecutor.ts:412-440
* getCompletedResults(): Generator<MessageUpdate, void> {
  if (this.discarded) return

  for (const tool of this.tools) {
    // 1. 优先吐 pending progress
    while (tool.pendingProgress.length > 0) {
      const progressMessage = tool.pendingProgress.shift()!
      yield {message: progressMessage, newContext: this.toolUseContext}
    }

    if (tool.status === 'yielded') continue  // 已消费，跳过

    if (tool.status === 'completed' && tool.results) {
      tool.status = 'yielded'  // 标记防重复
      for (const message of tool.results) {
        yield {message, newContext: this.toolUseContext}
      }
      markToolUseAsComplete(this.toolUseContext, tool.id)
    } else if (tool.status === 'executing' && !tool.isConcurrencySafe) {
      break  // 顺序约束：非并发还在跑，停下等
    }
  }
}
```

**有序性三规则**：

1. **入队顺序**：遍历 `this.tools` 数组（保留入队顺序）；
2. **进度优先**：进度消息在 `for` 循环最开头立即 yield；
3. **顺序等待**：遇到 `executing` 的非并发工具，`break` 让调用方去 await。

### 7.2 `getRemainingResults` —— 异步 generator

```ts
// StreamingToolExecutor.ts:453-490
async* getRemainingResults(): AsyncGenerator<MessageUpdate, void> {
  if (this.discarded) return

  while (this.hasUnfinishedTools()) {
    await this.processQueue()

    for (const result of this.getCompletedResults()) {
      yield result
    }

    // 如果还在跑、没新完成、没新进度，等任意一个
    if (
      this.hasExecutingTools() &&
      !this.hasCompletedResults() &&
      !this.hasPendingProgress()
    ) {
      const executingPromises = this.tools
        .filter(t => t.status === 'executing' && t.promise)
        .map(t => t.promise!)

      const progressPromise = new Promise<void>(resolve => {
        this.progressAvailableResolve = resolve
      })

      if (executingPromises.length > 0) {
        await Promise.race([...executingPromises, progressPromise])
      }
    }
  }

  // 最后一轮 sweep
  for (const result of this.getCompletedResults()) {
    yield result
  }
}
```

**Wake-up 触发条件**（任一满足即醒来）：
- 某个工具 `status='completed'`（`hasCompletedResults()`）；
- 某个工具产出新 progress 消息（`progressAvailableResolve`）；
- stream 结束，调用方主动驱动。

### 7.3 `markToolUseAsComplete` —— UI 同步

```ts
// StreamingToolExecutor.ts:521-530
function markToolUseAsComplete(
  toolUseContext: ToolUseContext,
  toolUseID: string,
) {
  toolUseContext.setInProgressToolUseIDs(prev => {
    const next = new Set(prev)
    next.delete(toolUseID)
    return next
  })
}
```

这个 callback 通知 UI 层把对应 tool_use 从 "in-progress" 列表移除。

---

## 8. Streaming Fallback 处理

当模型中途被 fallback 到另一个模型（`FallbackTriggeredError`），query.ts 会调用 `streamingToolExecutor.discard()`：

```ts
// StreamingToolExecutor.ts:69-71
discard(): void {
  this.discarded = true
}
```

**效果**：
- 后续 `addTool` 调用不再触发 `processQueue` —— 虽然 push 进队列但没人跑（其实 `addTool` 仍然会 `processQueue`，但 processQueue 不执行，因为已 discarded 的工具在执行时会拿到 `streaming_fallback` 错误）；
- 已 executing 的工具在 for await 循环里检测到 `discarded` → 收到 `'streaming_fallback'` reason → 生成 `<tool_use_error>Error: Streaming fallback - tool execution discarded</tool_use_error>`；
- `getCompletedResults` 和 `getRemainingResults` 第一行就 `return`。

为什么 fallback 必须丢弃？因为旧模型产出的 `tool_use` 块是基于它自己的训练对齐风格，切换模型后这些块**可能引用了不存在的工具名或参数**，继续执行会污染主对话上下文。注释解释：

> Called when streaming fallback occurs and results from the failed attempt should be abandoned. Queued tools won't start, and in-progress tools will receive synthetic errors.

---

## 9. 与 query.ts 的协作时序

下面把 `queryLoop` 一次完整 streaming turn 与 executor 的交互列出：

```
queryLoop.executeModelStreamingTurn()
  │
  ├─ [stream chunk 1] assistant message text
  ├─ [stream chunk 2] tool_use block: Bash(npm install)
  │     ↓
  │     streamingToolExecutor.addTool(block, assistantMessage)
  │       ├─ findToolByName → BashTool
  │       ├─ safeParse input → { command: 'npm install', run_in_background: false, ... }
  │       ├─ BashTool.isConcurrencySafe(input) → true（同 cwd 默认并发安全）
  │       ├─ push to tools: { status: 'queued', isConcurrencySafe: true }
  │       └─ void processQueue()
  │             ↓ canExecuteTool: executingTools.length === 0 → true
  │             ↓ executeTool(tool)
  │                   ├─ status = 'executing'
  │                   ├─ createChildAbortController
  │                   └─ for await (const update of runToolUse(...))
  │                         └─ ... （Bash 子进程启动）
  │
  ├─ [stream chunk 3] tool_use block: Read(README.md)
  │     ↓
  │     streamingToolExecutor.addTool(block, ...)
  │       ├─ ReadTool.isConcurrencySafe → true
  │       └─ processQueue → canExecuteTool: executingTools is [Bash] (concurrent-safe) AND new tool is concurrent-safe → true
  │             ↓ executeTool(Read) 并行启动
  │
  ├─ yield* getCompletedResults()    // 流式边收边 yield
  │     ├─ Read 立即吐进度（无）→ 检查 status
  │     ├─ Read.status === 'queued'，不是 completed，break
  │     └─ (实际：会立刻检查 Bash 是否也完成，未完成则 break)
  │
  ├─ [stream chunk 4-N] tool_use block: Edit(src/App.tsx)
  │     ↓
  │     streamingToolExecutor.addTool(block, ...)
  │       ├─ EditTool.isConcurrencySafe({file_path: 'src/App.tsx'}) → 看工具实现
  │       ├─ 若 Edit 与 Bash 同 cwd，且 EditTool 判定为不安全 → isConcurrencySafe: false
  │       └─ processQueue → canExecuteTool: false → break
  │
  ├─ [stream 结束]
  │     ↓
  │     yield* getRemainingResults()
  │       ├─ 第一个循环：await processQueue（启动 Edit 等可以启动的）
  │       ├─ getCompletedResults（吐新完成的）
  │       ├─ 如果还在跑：Promise.race(executingPromises, progressPromise)
  │       └─ ... 直到所有 status === 'yielded'
  │
  └─ tools[].results 全部回流到主 messages 数组 → 进入下一轮 query
```

---

## 10. 关键设计取舍

### 10.1 顺序保证 vs 并发速度

`canExecuteTool` 的"全清空或全并发安全"二分法，让 **Bash + Edit + Read** 这种组合在多数情况下能并行（Bash 跑测试时 Read 可同时读其他文件）。但 `Edit` + `Edit` 改同一文件时，工具自身的 `isConcurrencySafe` 应返回 false（保守）。

### 10.2 只有 Bash 错误级联

注释解释：

> Only Bash errors cancel siblings. Bash commands often have implicit dependency chains (e.g. mkdir fails → subsequent commands pointless). Read/WebFetch/etc are independent — one failure shouldn't nuke the rest.

这避免了一种情况：用户让 Claude "读 3 个文件并行 + 跑一个 Bash"，其中 1 个文件 Read 失败，Bash 不会被打断。

### 10.3 abort 冒泡条件精确

`signal.reason !== 'sibling_error'` 这个守卫非常关键 —— 它防止 **Bash 失败 → sibling_cancel → 整个 turn 被终止** 的级联，确保 query.ts 能继续处理已完成的工具结果。

注释解释：

> Permission-dialog rejection also aborts this controller (PermissionContext.ts cancelAndAbort) — that abort must bubble up to the query controller so the query loop's post-tool abort check ends the turn. Without bubble-up, ExitPlanMode "clear context + auto" sends REJECT_MESSAGE to the model instead of aborting (#21056 regression).

### 10.4 进度消息独立通道

进度不进 `messages` 而进 `pendingProgress`，并在 `getCompletedResults` 的最外层立即 yield，让 UI 有"工具正在跑"的可视反馈。

### 10.5 'yielded' 状态防止重复 yield

`getCompletedResults` 把每个 tool 的 status 从 `'completed'` 改为 `'yielded'`，下一次再调用就不会重复吐结果 —— 解决 generator 重入问题（虽然 generator 不应该重入，但防御性写法）。

### 10.6 unknown tool 立即合成错误

```ts
// StreamingToolExecutor.ts:78-102
const toolDefinition = findToolByName(this.toolDefinitions, block.name)
if (!toolDefinition) {
  this.tools.push({
    ...
    status: 'completed',
    results: [createUserMessage({
      content: [{
        type: 'tool_result',
        content: `<tool_use_error>Error: No such tool available: ${block.name}</tool_use_error>`,
        is_error: true,
        tool_use_id: block.id,
      }],
      ...
    })],
  })
  return
}
```

模型产出了不存在的工具名（如 hallucinate），不会阻塞整个 stream —— 立即生成错误回流，让模型自我修正。

### 10.7 输入解析失败的保守降级

```ts
// StreamingToolExecutor.ts:104-113
const parsedInput = toolDefinition.inputSchema.safeParse(block.input)
const isConcurrencySafe = parsedInput?.success
  ? Boolean(toolDefinition.isConcurrencySafe(parsedInput.data))
  : false
```

如果输入 schema 验证失败，**强制** `isConcurrencySafe: false` —— 宁可慢一点串行执行，不要因为错误的输入把不一致状态写到并发执行的工具里。

---

## 11. 已知限制

### 11.1 Context modifiers 不支持并发工具

源码注释（**`StreamingToolExecutor.ts:388-395`**）：

> NOTE: we currently don't support context modifiers for concurrent tools. None are actively being used, but if we want to use them in concurrent tools, we need to support that here.

```ts
if (!tool.isConcurrencySafe && contextModifiers.length > 0) {
  for (const modifier of contextModifiers) {
    this.toolUseContext = modifier(this.toolUseContext)
  }
}
```

只有**非并发**工具的 `contextModifiers` 会被应用。concurrent 工具虽然能产出 modifier，但被忽略。

### 11.2 abortController 的冒泡时序

`siblingAbortController.abort('sibling_error')` 触发后，已 executing 的工具的 abort 监听器会先看到 reason 是 `'sibling_error'`，**不会** 冒泡 —— 这是设计意图。但若 sibling_error 之后用户又按 ESC，新一轮 abort 进入时，reason 不再是 `'sibling_error'`，会冒泡 —— 此时 query.ts 的 post-tool 检查会结束整个 turn（即使大部分工具已完成）。

### 11.3 `discard()` 后 `addTool` 仍会触发 processQueue

```ts
// StreamingToolExecutor.ts:114-123
this.tools.push({...status: 'queued', ...})
void this.processQueue()
```

即使 `this.discarded = true`，`addTool` 仍然 push 并调用 `processQueue`。`processQueue` 会启动 `executeTool`，但 `executeTool` 内部第一件事是检查 `getAbortReason`（见 §5.4）—— 因为 `discarded=true` → 返回 `'streaming_fallback'` → 直接合成错误，不进入 `runToolUse`。

这是一层冗余保护：理论上 `discard()` 之后 `query.ts` 不应再调用 `addTool`，但即使调用了也不会出错。

### 11.4 `progressAvailableResolve` 一次性

每次 `processQueue` 循环前会重新注册：

```ts
// StreamingToolExecutor.ts:476-479
const progressPromise = new Promise<void>(resolve => {
  this.progressAvailableResolve = resolve
})
```

`resolve` 之后立即置 undefined：

```ts
// StreamingToolExecutor.ts:372-374
if (this.progressAvailableResolve) {
  this.progressAvailableResolve()
  this.progressAvailableResolve = undefined
}
```

如果 resolve 之前没有 await，第二次到达的 progress 事件不会触发新 promise —— 但下一次 processQueue 会重新注册，所以不影响。

---

## 附录：与 query.ts 的具体集成点

| query.ts 位置 | 调用方式 | 说明 |
|--------------|---------|------|
| `executeModelStreamingTurn` 内 | `new StreamingToolExecutor(...)` | 仅当 `config.gates.streamingToolExecution` 为 true 时实例化 |
| streaming chunk 中 | `executor.addTool(block, assistantMessage)` | 每收到一个 `tool_use` 块 |
| streaming 中 | `for (const r of executor.getCompletedResults()) yield r` | 边收边 yield |
| streaming 结束 | `yield* executor.getRemainingResults()` | 等所有完成 |
| `FallbackTriggeredError` 处理 | `executor.discard()` | 中断所有 in-flight 工具 |
| 无 streamingToolExecution gate 时 | `runTools(...)` 兜底 | 同步执行 |

相关埋点事件（`query.ts`）：

| 事件 | 触发时机 |
|------|---------|
| `tengu_streaming_tool_execution_used` | query.ts 实例化 StreamingToolExecutor 时 |
| `tengu_streaming_tool_execution_not_used` | 退化为 runTools 时 |
| `tengu_streaming_fallback_tool_discarded` | discard() 被调用时 |

---

## 推荐阅读

- [`docs/09-query.md`](../09-query.md) —— queryLoop 主循环，了解 executor 在哪一段被驱动
- [`docs/11-tool-execution.md`](11-tool-execution.md) —— `runTools` / `runToolUse` 同步路径
- [`docs/14-tool-hooks.md`](14-tool-hooks.md) —— PreToolUse / PostToolUse 钩子如何在 tool 生命周期内介入
- [`docs/16-claude-api.md`](../16-claude-api.md) —— Anthropic API 流式返回的 `tool_use` 块结构
- [`docs/23-design-and-core-modules.md`](../architecture/23-design-and-core-modules.md) —— 整体设计原理