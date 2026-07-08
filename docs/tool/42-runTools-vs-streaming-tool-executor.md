# runTools vs StreamingToolExecutor —— 两条工具执行路径的对比

> 源码位置:
> - `src/services/tools/toolOrchestration.ts`（189 行）→ `runTools` / `runToolsSerially` / `runToolsConcurrently`
> - `src/services/tools/StreamingToolExecutor.ts`（531 行）→ 类 `StreamingToolExecutor`
>
> 共同底层: `src/services/tools/toolExecution.ts` 中的 `runToolUse`
> 调度入口: `src/query.ts` 的 `queryLoop`，按 `config.gates.streamingToolExecution` 二选一
> 关联文档: [`docs/09-query.md`](../09-query.md) · [`docs/tool/11-tool-execution.md`](11-tool-execution.md) · [`docs/tool/25-streaming-tool-executor.md`](25-streaming-tool-executor.md)

Claude Code 主循环里，**"把模型输出的 `tool_use` 块变成真实副作用"** 这件事由两条路径分担：`toolOrchestration.runTools`（一次性同步路径）和 `StreamingToolExecutor`（流式并发路径）。它们共用底层 `runToolUse`，但启动时机、并发模型、错误级联、中断协调完全不同。本文逐项拆解。

---

## 目录

1. [抽象定位](#1-抽象定位)
2. [何时启动、何时回流](#2-何时启动何时回流)
3. [并发与隔离模型](#3-并发与隔离模型)
4. [错误、取消、中断](#4-错误取消中断)
5. [进度消息（`progress`）的处理](#5-进度消息progress的处理)
6. [Context Modifier 的支持差异](#6-context-modifier-的支持差异)
7. [中断状态对外暴露](#7-中断状态对外暴露)
8. [工具未找到时的差异](#8-工具未找到时的差异)
9. [生命周期方法对比](#9-生命周期方法对比)
10. [query.ts 如何在两条路径间切换](#10-queryts-如何在两条路径间切换)
11. [一次性总结](#11-一次性总结)

---

## 1. 抽象定位

| 维度 | `runTools`（`toolOrchestration.ts:19`） | `StreamingToolExecutor` |
|---|---|---|
| **何时调用** | 模型流式响应**结束后**，一次性拿到完整 `tool_use` 列表才启动 | 模型**流式过程中**，每收到一个 `tool_use` block **立即入队**并尝试启动 |
| **结果何时回流** | `async function*` 生成器，`runTools` 返回值由 `query.ts` 一次性 `for await` 遍历 | 多个对外方法（`getCompletedResults` / `getRemainingResults`）穿插在流式循环里，**边产出边收** |
| **形态** | 纯函数（自由生成器） | **类**（`StreamingToolExecutor`），有内部状态 + 多个 query 阶段共用同一个实例 |
| **核心调用** | `runToolUse` | `runToolUse`（同一份底层） |

调度入口在 `src/query.ts:1377-1393`：

```ts
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

也就是说 `StreamingToolExecutor` 路径的「全权执行」是 `query.ts` 里的「**一边 stream model，一边 addTool + getCompletedResults**」（流式穿插）+「**流结束后 getRemainingResults 收尾**」两步拼成的；`runTools` 路径只走第二步。

---

## 2. 何时启动、何时回流

- **`runTools`**（`toolOrchestration.ts:19`）是一次性的 `async function*`，接收完整 `toolUseMessages[]` 才开始。
- **`StreamingToolExecutor.addTool`**（`StreamingToolExecutor.ts:76`）在 `query.ts:846-853` 里，每当流式消息里**包含 `tool_use` block 就立刻调用**——`streamingToolExecutor.addTool(toolBlock, assistantMessage)`，内部 `void this.processQueue()` 直接尝试启动下一个可执行的工具。

> 实际效果：流式模式下，**模型还在打字时，Read / Grep 等并发安全工具已经在跑**；非流式模式，得等整个回复打完才开始。

---

## 3. 并发与隔离模型

两者都用 `isConcurrencySafe` 分批（并发批 vs 互斥批），但实现粒度不同。

### `runTools` —— 批级 partition + `all(...)` 上限 10

- `partitionToolCalls`（`toolOrchestration.ts:91-116`）把工具按「**连续相同 concurrency-safe 级别**」切 batch：
  - 安全且与上一批同安全级 → 合并；
  - 否则开新 batch。
- **整批**一起提交给 `runToolsConcurrently`（用 `utils/generators.js` 的 `all(..., getMaxToolUseConcurrency())`，上限默认 10，由 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 覆盖）。
- 批与批之间**串行**（非 safe 批单独走 `runToolsSerially`）。

### `StreamingToolExecutor` —— 逐个入队 + 内部 `canExecuteTool`

- **逐个**判断：每 `addTool` 进来后，看当前 `executing` 列表；
- 规则（`StreamingToolExecutor.ts:129`）：

  ```
  无 executing → 任何 tool 都能执行
  有 executing 且当前 tool safe 且所有 executing 也 safe → 可以并发
  否则不能执行，排队
  ```

- 一旦遇到**非 safe 工具排在前面**，整个队列会**停在那里**（`processQueue` 第 148 行 `if (!tool.isConcurrencySafe) break`），等它跑完再继续。

`runToolsConcurrently` 还多一个细节：并发的每个工具**各自捕获同一个 `toolUseContext`**（不再像 serial 那样把 `currentContext` 串起来）。所以并发路径里，**目前不支持 `contextModifier`**（`toolOrchestration.ts:388-395` 的注释明确说「we currently don't support context modifiers for concurrent tools」），`runTools` 在尾部会把 `queuedContextModifiers` **延后**应用到 `currentContext`（`toolOrchestration.ts:54-63`），但仅限于「同一批并发」结束后。

---

## 4. 错误、取消、中断

`runTools` 里没有错误短路，只靠 `runToolUse` 自身抛错。
`StreamingToolExecutor` 有**完整的兄弟取消 + 中断分类**：

| 行为 | 实现位置 |
|---|---|
| **Bash 出错 → 兄弟全死** | `StreamingToolExecutor.ts:357-363`：仅当 `tool.block.name === BASH_TOOL_NAME` 时 `this.siblingAbortController.abort('sibling_error')` |
| **子 abort 不应终结整个 turn** | `StreamingToolExecutor.ts:46-48` 的注释 + `StreamingToolExecutor.ts:301-318`：**冒泡策略**——子 controller 的 abort 若 reason 非 `sibling_error` 且父未 abort，才向上冒泡（避免兄弟级中断污染主循环） |
| **合成错误消息** | `createSyntheticErrorMessage`（`StreamingToolExecutor.ts:153-205`）区分三种 reason：`sibling_error` / `user_interrupted` / `streaming_fallback`，前者用工具描述，后两者用固定文案 + `REJECT_MESSAGE` |
| **用户中断时区分 `cancel` vs `block` 工具** | `getToolInterruptBehavior`（`StreamingToolExecutor.ts:233-241`）：只有 `interruptBehavior === 'cancel'` 的工具会被中断消息取消 |
| **streaming fallback 丢弃** | `discard()` 把 `this.discarded = true`，正在跑的兄弟通过 `getAbortReason` 拿到 `streaming_fallback` reason，产出对应合成错误 |

> 关键设计：**只有 Bash 错误会级联**。Read / WebFetch 等「独立型」工具的失败**不应该**拖垮兄弟（注释见 `StreamingToolExecutor.ts:355-363`：Bash 命令往往有隐式依赖链，如 `mkdir` 失败后续命令就无意义；其他工具是相互独立的）。

---

## 5. 进度消息（`progress`）的处理

`runTools` 把所有 `update.message` 一视同仁地 yield 出去。
`StreamingToolExecutor` 在 `executeTool`（`StreamingToolExecutor.ts:366-378`）里：

- `progress` 类型消息进 `tool.pendingProgress`，**立即**通过 `progressAvailableResolve` 通知 `getRemainingResults` 解锁；
- 其他消息先存到 `tool.results`，等 `completed` 后才按顺序 yield（`getCompletedResults` 第 412-440 行）。

> 这条配合 `query.ts:856-871` 的「流式循环里穿插 `getCompletedResults()`」，实现「**用户先看到 grep 的进度刷屏，工具结果最后统一回灌给模型**」的效果。

---

## 6. Context Modifier 的支持差异

`contextModifier` 是工具在执行过程中**改写 `ToolUseContext`** 的机制（典型用途：工具往 `readFileState` 里追加一个文件指纹，下一轮 Read 工具就知道「这个文件刚被读过」）。

| 路径 | 支持方式 |
|---|---|
| `runTools` serial | 立即应用到 `currentContext`（`toolOrchestration.ts:140-147`） |
| `runTools` concurrent | 暂存到 `queuedContextModifiers`，**整批结束后**一次性应用（`toolOrchestration.ts:42-63`） |
| `StreamingToolExecutor` 非 safe 工具 | 整批结束后应用（`StreamingToolExecutor.ts:391-395`） |
| `StreamingToolExecutor` safe 并发工具 | **不支持**（注释 `StreamingToolExecutor.ts:388-390`：目前没有并发工具在使用 modifier，若未来需要再支持） |

---

## 7. 中断状态对外暴露

只有 `StreamingToolExecutor` 调用 `this.toolUseContext.setHasInterruptibleToolInProgress?.(...)`（`StreamingToolExecutor.ts:254-260`），作用是「**当前所有 executing 工具都是 cancel 行为时，告诉 UI 可以放行下一次用户中断**」。`runTools` 完全没有这个钩子。

> 这条是 ESC / Ctrl+C 体验的关键：UI 通过这个 state 决定「再按一次是否真的终止 turn」。

---

## 8. 工具未找到时的差异

- `StreamingToolExecutor.addTool`（`StreamingToolExecutor.ts:78-102`）直接合成 `<tool_use_error>Error: No such tool available: ...</tool_use_error>`，**不调用** `runToolUse`。
- `runTools` 依赖底层 `runToolUse` 的处理（工具缺失时通常会走 `findToolByName` 返回 `undefined` 的分支，再由 `runToolUse` 合成错误）。

---

## 9. 生命周期方法对比

| 方法 | `runTools` | `StreamingToolExecutor` |
|---|---|---|
| `runTools(...)` | ✅ 入口，`async function*` | ❌ |
| `addTool(block, msg)` | ❌ | ✅ 流式增量加入 |
| `getCompletedResults()` | ❌ | ✅ 非阻塞轮询，流式过程中穿插用 |
| `getRemainingResults()` | ❌ | ✅ 流结束后阻塞等待所有 |
| `discard()` | ❌ | ✅ 流式 fallback 时丢弃 |
| `getUpdatedContext()` | ❌ | ✅ 暴露被 modifier 修改过的 context |
| `markToolUseAsComplete` | 局部（`toolOrchestration.ts:179-188`） | 重复实现一次（`StreamingToolExecutor.ts:521-530`，**这里反而有重复代码**） |

---

## 10. query.ts 如何在两条路径间切换

这是 `query.ts:561-581` 的运行时决策：

```ts
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(...)
  : null
```

- **开启 `streamingToolExecution` 时**：
  - 模型流式响应过程中 → `query.ts:846-871` 调用 `streamingToolExecutor.addTool(...)` + `streamingToolExecutor.getCompletedResults()`，**提前并行**触发 Read / Grep 等安全工具，同时收进度消息；
  - 流结束后 → `query.ts:1391-1393` 调用 `streamingToolExecutor.getRemainingResults()` 收尾。
- **关闭时**（本仓库默认，见 `feature()` polyfill）：走 `runTools(...)`，等模型**完整回复**后才开始执行。

也就是说 **`StreamingToolExecutor` 是 `runTools` 的「提前 + 增强版」**，只多了「**提前并行**」「**进度立刻回吐**」「**Bash 错误级联终止**」这三件事；**核心的工具执行逻辑都是同一个 `runToolUse`**（`toolExecution.ts`），两者共享底座。

---

## 11. 一次性总结

| 关注点 | `runTools` | `StreamingToolExecutor` |
|---|---|---|
| 启动时机 | 流结束 | 流过程中 |
| 并发模型 | 批级（`all`，上限 10） | 逐个入队 + 内部 `canExecuteTool` 判断 |
| 错误级联 | 无 | Bash 出错 → 兄弟 `siblingAbortController.abort` |
| 中断分类 | 无 | `cancel` / `block` 工具分桶 |
| Fallback 丢弃 | 无 | `discard()` + 合成错误消息 |
| 进度消息 | 跟着 yield | 单独 `pendingProgress` + 立即 yield |
| Context modifier 支持 | 并发路径延后应用 | 仅非 safe 工具支持 |
| 是否暴露 `setHasInterruptibleToolInProgress` | 否 | 是 |
| 形态 | 纯函数生成器 | 类，持有状态 |

简而言之：**`runTools` 是「等模型说完，一次跑完所有工具」；`StreamingToolExecutor` 是「边说边跑、互相感知」**。两者共用 `runToolUse`，后者在前者基础上多承担了「**并发 + 错误级联 + 中断协调 + 进度流式回吐**」这些**模型还没说完就要并行干活**才需要的复杂性。