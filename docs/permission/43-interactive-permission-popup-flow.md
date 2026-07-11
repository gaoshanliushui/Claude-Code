# 交互式权限弹框 / 询问弹框的完整执行流程

> 源码位置:
> - `CanUseToolFn` 签名：`src/hooks/useCanUseTool.tsx:35`
> - 闭包主体（4 阶段流水线）：`src/hooks/useCanUseTool.tsx:37-199`
> - `PermissionContext` 工厂：`src/hooks/toolPermission/PermissionContext.ts:350`
> - 交互处理器（pushToQueue + 多路竞态）：`src/hooks/toolPermission/handlers/interactiveHandler.ts:57-232`
> - 静态规则入口：`src/utils/permissions/permissions.ts`（见 [`30-permission-control-flow.md`](30-permission-control-flow.md)）
> - 弹框根组件 + 工具 → UI 派发表：`src/components/permissions/PermissionRequest.tsx:47-82`、`146-216`
> - 询问弹框子组件：`src/components/permissions/AskUserQuestionPermissionRequest/`
> - 队列消费方：`src/screens/REPL.tsx`
> - 工具执行派发：`src/QueryEngine.ts` / `src/query.ts`

> 关联文档：
> - [`docs/permission/30-permission-control-flow.md`](30-permission-control-flow.md) —— 静态规则如何决策 allow/deny/ask
> - [`docs/permission/40-permission-system-design.md`](40-permission-system-design.md) —— 权限系统设计
> - [`docs/permission/41-bash-tool-permission-control.md`](41-bash-tool-permission-control.md) —— Bash 工具的细分权限
> - [`docs/permission/42-toolUseContext-vs-toolPermissionContext.md`](42-toolUseContext-vs-toolPermissionContext.md) —— 两个 Context 的职责边界

`30-permission-control-flow.md` 解释了「模型请求执行工具时，如何决策 allow / ask / deny」。
本文接着向下游走：当决策结果是 **ask** 时，弹框如何被推到 UI、用户如何与之交互、决策又如何回流到工具执行链路。

---

## 目录

1. [两类弹框](#1-两类弹框)
2. [调用栈全景](#2-调用栈全景)
3. [阶段 1 — 模型触发工具调用](#3-阶段-1--模型触发工具调用)
4. [阶段 2 — canUseTool 闭包：权限检查流水线](#4-阶段-2--canusetool-闭包权限检查流水线)
5. [阶段 3 — interactiveHandler：发起弹框与多路竞态](#5-阶段-3--interactivehandler发起弹框与多路竞态)
6. [阶段 4 — React 端渲染弹框](#6-阶段-4--react-端渲染弹框)
7. [阶段 5 — 用户决策与回流](#7-阶段-5--用户决策与回流)
8. [阶段 6 — 回到 tool.call()：执行或失败](#8-阶段-6--回到-toolcall执行或失败)
9. [关键并发 / 状态设计要点](#9-关键并发--状态设计要点)
10. [典型链路举例（BashTool）](#10-典型链路举例bashtool)
11. [关键文件索引](#11-关键文件索引)

---

## 1. 两类弹框

Claude Code 中的弹框分两类，但它们**共用同一条 Promise resolve 通道**：

| 类型 | 触发场景 | 典型例子 |
| --- | --- | --- |
| **权限弹框** | 模型想执行某个工具，但该工具未在静态规则里被允许 | 跑 `rm -rf`、`Edit` 一个新文件、执行 `npm install` |
| **询问弹框** | 模型显式调用 `AskUserQuestion` 工具 | 让用户在 2~4 个选项里选一个交互方式 |

两者都走同一个 `canUseTool(tool, input, ctx, msg, id)` 回调，由 React/Ink 端用 React state 把"待处理项"推到 UI。

---

## 2. 调用栈全景

```
用户输入 prompt
  └─ REPL.onQuery                          (src/screens/REPL.tsx)
      └─ QueryEngine.startQuery            (src/QueryEngine.ts)
          └─ query() 主循环                (src/query.ts)
              └─ 流式接收 message_stop / tool_use 块
                  └─ tool.call(args, ctx, canUseTool, parent, onProgress)
                                            ↑
                                  关键入口：canUseTool
                  └─ BashTool.call() / FileEditTool.call() / AskUserQuestionTool.call() …
                      └─ await canUseTool(tool, input, ctx, message, toolUseID)
                                            ↓
                       ┌─────────────────────────────────────────────┐
                       │ canUseTool 闭包（useCanUseTool.tsx:37）     │
                       ├─────────────────────────────────────────────┤
                       │ 1. hasPermissionsToUseTool 静态规则检查     │
                       │ 2. coordinator / swarm / bash classifier    │
                       │ 3. handleInteractivePermission ← 关键       │
                       └──────────────────┬──────────────────────────┘
                                          │ result.behavior === 'ask'
                                          ↓
                       ┌─────────────────────────────────────────────┐
                       │ interactiveHandler.ts:57                   │
                       │   ctx.pushToQueue(toolUseConfirm)          │
                       │   + 启动 bridge / channel / hook 竞态      │
                       └──────────────────┬──────────────────────────┘
                                          ↓
                                  React 状态: toolUseConfirmQueue
                                          ↓
                       REPL 渲染 <PermissionRequest /> 组件
                                          ↓
                       permissionComponentForTool(tool) 派发
                                          ↓
                       ┌────────────────┬───────────────────────────┐
                       │ BashTool       │ AskUserQuestionTool        │
                       │ → BashPR       │ → AskUserQuestionPR        │
                       │ FileEditTool   │ ExitPlanModeTool            │
                       │ → FileEditPR   │ → ExitPlanModePR            │
                       │ …              │ Fallback                    │
                       └────────────────┴───────────────────────────┘
                                          ↓
                       用户键盘输入 → onAllow / onReject
                                          ↓
                          resolveOnce(decision) → 回到阶段 2
```

---

## 3. 阶段 1 — 模型触发工具调用

- `query.ts` 主循环解析流式事件，累积 `tool_use` 块。
- `QueryEngine` 把每个 `tool_use` 派发到对应 `tool.call(args, ctx, canUseTool, parentMessage, onProgress)`。
- `canUseTool` 这个参数是在 `useCanUseTool.tsx:37` 工厂里构造的闭包，被注入到 `Tool.call()` 的第三参里（参见 `src/Tool.ts:379-385` 的接口签名）。

---

## 4. 阶段 2 — canUseTool 闭包：权限检查流水线

源代码：`src/hooks/useCanUseTool.tsx:37-199`

闭包内部按顺序做四件事：

### 4.1 构造 PermissionContext

调用 `createPermissionContext(...)`，ctx 持有 `resolve / cancel / logDecision / pushToQueue / runHooks` 等所有响应能力。

### 4.2 静态规则判定

`hasPermissionsToUseTool(...)`（`src/utils/permissions/permissions.ts`，详见 `30-permission-control-flow.md`）返回：

- `{ behavior: 'allow', ... }` → 直接 `ctx.buildAllow()` 并 resolve（**不弹框**）
- `{ behavior: 'deny', ... }` → 直接 resolve（**不弹框**）
- `{ behavior: 'ask', ... }` → 进入 4.3 / 4.4

### 4.3 自动化预检（feature-flagged 路径）

按顺序尝试以下自动化裁决，任一返回非空 decision 就直接 resolve、不再弹框：

| 处理器 | 文件 | 作用 |
| --- | --- | --- |
| `handleCoordinatorPermission` | `coordinatorHandler.ts` | 主线程处于 `awaitAutomatedChecksBeforeDialog` 模式时由 coordinator agent 提前裁决 |
| `handleSwarmWorkerPermission` | `swarmWorkerHandler.ts` | swarm worker 模式下的特殊路由 |
| **Bash classifier**（`BASH_CLASSIFIER`） | `bashPermissions.ts` | 后台异步检查 bash 命令是否命中 prompt rule；命中则自动放行 + 显示 ✓ 1~3 秒 |

### 4.4 本地弹框入口

所有自动化路径都没裁决时，调用 `handleInteractivePermission(params, resolve)`，进入下一阶段。

---

## 5. 阶段 3 — interactiveHandler：发起弹框与多路竞态

源代码：`src/hooks/toolPermission/handlers/interactiveHandler.ts:57-232`

这是弹框的**真正发起者**，也是把"同步式 CLI prompt"映射成"异步事件循环"的关键。

### 5.1 构造 ToolUseConfirm 并入队

```ts
ctx.pushToQueue({
  assistantMessage, tool, description, input,
  toolUseContext, toolUseID,
  permissionResult: result,
  permissionPromptStartTimeMs,
  onUserInteraction() { ... },     // 用户开始操作时取消 classifier
  onAbort() { resolveOnce(...deny...) },
  async onAllow(updatedInput, updates, feedback, contentBlocks) {
    resolveOnce(await ctx.handleUserAllow(...))
  },
  onReject(feedback, contentBlocks) {
    resolveOnce(ctx.cancelAndAbort(...))
  },
  async recheckPermission() { ... },   // 模式切换时重新检查
})
```

`pushToQueue` 实际就是调用 `setToolUseConfirmQueue` 把对象推入 React state（`AppState.toolUseConfirmQueue`），触发 REPL 重渲染。

### 5.2 启动多个并行"竞态"

为了让用户可以在手机 / Web / Telegram 上同时响应，handler 会同时挂起以下来源：

| 来源 | 何时挂起 | 怎么挂起 |
| --- | --- | --- |
| **本地 UI**（REPL 弹框） | 总是 | `pushToQueue` 写入 React state |
| **Bridge (CCR / claude.ai)** | `bridgeCallbacks` 存在时 | 发 `bridgeRequestId` 到远端 UI，订阅 `onResponse` |
| **Channel (Telegram / iMessage)** | `KAIROS` / `KAIROS_CHANNELS` 开启且工具不需要额外交互字段时 | 通过 channel MCP `notification` 发送，订阅 `channelRequestId` |
| **PermissionRequest hooks** | `!awaitAutomatedChecksBeforeDialog` | 异步执行 `ctx.runHooks(...)`，先返回者通过 `claim()` 抢占 |
| **Bash classifier** | Bash 工具且有 `pendingClassifierCheck` | `executeAsyncClassifierCheck(...)` 异步判定，命中则 1~3 秒 ✓ 后自动放行 |

所有这些路径共享 `claim()` 原子锁，谁先到谁赢；后续响应被 `if (!claim()) return` 拦截。

### 5.3 Promise 桥接

`createResolveOnce(resolve)`（`interactiveHandler.ts:70`）生成 `resolveOnce / isResolved / claim` 三件套；`onAllow / onReject / recheckPermission` 各自包裹 `claim()`，确保只 resolve 一次。

### 5.4 中止信号

`abortController.signal` 监听 `abort` 事件——用户按 Esc、Ctrl+C 或父进程取消时，所有竞态订阅都被回收，handler 调 `ctx.cancelAndAbort(...)` resolve 一个 deny 决策。

---

## 6. 阶段 4 — React 端渲染弹框

源代码：`src/components/permissions/PermissionRequest.tsx:146-216`

REPL 监视 `toolUseConfirmQueue`，从队首取一个 `toolUseConfirm` 渲染 `<PermissionRequest toolUseConfirm={...} />`。

### 6.1 PermissionRequest 组件流程

1. `useKeybinding("app:interrupt", ...)` 绑定 Esc / Ctrl+C → 取消
2. `useNotifyAfterTimeout(notificationMessage, "permission_prompt")` → 终端失焦超过阈值时发系统通知（`"Claude needs your permission to use Bash"`）
3. `permissionComponentForTool(tool)` 大型 switch 分发到具体弹框组件
4. 把 `onAllow / onReject / recheckPermission / onUserInteraction / onAbort / onDismissCheckmark` 透传给子组件

### 6.2 工具 → 弹框组件派发表

来源：`src/components/permissions/PermissionRequest.tsx:47-82`

| 工具 | 弹框组件 |
| --- | --- |
| `FileEditTool` | `FileEditPermissionRequest` |
| `FileWriteTool` | `FileWritePermissionRequest` |
| `BashTool` | `BashPermissionRequest` |
| `PowerShellTool` | `PowerShellPermissionRequest` |
| `WebFetchTool` | `WebFetchPermissionRequest` |
| `NotebookEditTool` | `NotebookEditPermissionRequest` |
| `ExitPlanModeV2Tool` | `ExitPlanModePermissionRequest` |
| `EnterPlanModeTool` | `EnterPlanModePermissionRequest` |
| `SkillTool` | `SkillPermissionRequest` |
| `AskUserQuestionTool` | `AskUserQuestionPermissionRequest` |
| `ReviewArtifactTool` | `ReviewArtifactPermissionRequest`（`REVIEW_ARTIFACT` feature，否则 `FallbackPermissionRequest`） |
| `WorkflowTool` | `WorkflowPermissionRequest`（`WORKFLOW_SCRIPTS` feature，否则 fallback） |
| `MonitorTool` | `MonitorPermissionRequest`（`MONITOR_TOOL` feature，否则 fallback） |
| `GlobTool` / `GrepTool` / `FileReadTool` | `FilesystemPermissionRequest` |
| 其他 | `FallbackPermissionRequest` |

### 6.3 AskUserQuestion 弹框的细分

源代码：`src/components/permissions/AskUserQuestionPermissionRequest/AskUserQuestionPermissionRequest.tsx`

子结构：

| 组件 | 位置 | 职责 |
| --- | --- | --- |
| `AskUserQuestionPermissionRequestBody` | `…:75` | 状态机壳，管理多题导航、答案聚合 |
| `QuestionView` | `QuestionView.tsx:43` | 单题的标准视图（选项 + 备注 + 导航栏） |
| `PreviewQuestionView` | `PreviewQuestionView.tsx:41` | 带 preview 面板的侧边布局 |
| `SubmitQuestionsView` | `…` | 多题最后一题的提交面板 |
| `QuestionNavigationBar` | `…` | 顶部 Tab 导航条 |
| `PreviewBox` | `…` | preview 内容的渲染容器 |

用户交互：

- ↑ / ↓ 切换选项，数字键 `1~9` 直跳
- `n` 进入备注输入；`ctrl+g` 调外部编辑器
- `Tab` 在多题之间切换
- `Enter` 选择当前项；最末题 `Enter` → `onRespondToClaude()` → `toolUseConfirm.onAllow(answers, ...)`
- `Esc` → `onCancel()` → `onReject()`

### 6.4 普通权限弹框（以 BashPermissionRequest 为例）

- 显示命令预览 / diff
- 选项：`Yes` / `Yes, and don't ask again for this command` / `No`
- ← / → 切换选项，`Enter` 确认，`Esc` 拒绝
- 拒绝时可输入 feedback 反馈给模型
- "don't ask again" 路径在 `onAllow` 里把规则写入 `permissionUpdates`，由 `ctx.persistPermissions(...)` 持久化

---

## 7. 阶段 5 — 用户决策与回流

| 触发 | 行为 | 副作用 |
| --- | --- | --- |
| 用户确认（按下"是"） | `onAllow(updatedInput, permissionUpdates, feedback, contentBlocks)` | 持久化 `permissionUpdates`（"don't ask again"）→ `ctx.buildAllow(...)` → resolve allow 决策 |
| 用户拒绝 + 可选 feedback | `onReject(feedback, contentBlocks)` | `ctx.cancelAndAbort(feedback, undefined, contentBlocks)` → resolve deny 决策 |
| Esc / Ctrl+C | `onAbort()` | `ctx.cancelAndAbort(undefined, true)` → resolve deny + abort controller |
| Classifier 命中 | classifier 内部直接调 `ctx.buildAllow` | resolve allow，并显示 ✓ 1~3 秒（`getTerminalFocused()` 决定时长） |
| Bridge / Channel 回包 | handler 内部 `resolveOnce(...)` | 调 `cancelRequest` 通知远端 UI 关闭 |
| Hook 先返回 | `ctx.runHooks` 回调 | `claim()` 抢占，弹框 `removeFromQueue` |

`onUserInteraction()` 在用户首次按键时被调用（带 200ms grace period 防误触），用来：
1. 取消 classifier 异步检查（`clearClassifierChecking`）
2. 清除弹框上的 "classifier running" 指示器

---

## 8. 阶段 6 — 回到 tool.call()：执行或失败

`canUseTool(...)` 返回 `PermissionDecision` 后：

- `behavior === 'allow'`：tool 拿到 `updatedInput`（可能经过工具预处理），调用 `tool.call(...)` 真正执行
  - Bash → spawn shell 跑命令
  - Edit → 写文件
  - AskUserQuestion → 把用户答案作为 `toolUseResult` 返回
- `behavior === 'deny'`：tool 抛错 / 返回 `is_error: true` 的结果，模型收到反馈继续对话

执行结果回灌到 messages，进入下一轮 API 调用。

---

## 9. 关键并发 / 状态设计要点

### 9.1 Promise ↔ React 状态桥

`canUseTool` 闭包持有一个未 resolve 的 Promise；UI 通过 `setToolUseConfirmQueue` 把 `onAllow / onReject` 包装进 state；用户按键 → 调闭包 → resolve Promise。这是把"阻塞式 prompt"映射成"异步事件循环"的关键抽象。

### 9.2 `claim()` 原子锁

桥、channel、hook、classifier、用户五个来源都可能先到。`createResolveOnce` 用闭包里的 boolean 标记位做 CAS，第二次 resolve 静默 no-op。

### 9.3 多个并发工具的排队

`toolUseConfirmQueue` 是数组；当前队首的弹框处理完才轮到下一个；classifier 命中的 ✓ 状态可保留 1~3 秒后 `removeFromQueue`。

### 9.4 `setStickyFooter` 通道

仅在 fullscreen 模式下生效；`ExitPlanModePermissionRequest` 等长内容弹框用它让响应选项始终可见，用户滚动计划时不会丢失按钮。

### 9.5 `awaitAutomatedChecksBeforeDialog`

主线程处于 coordinator 模式时，coordinator agent 可以代替用户先做决策，本地弹框不弹出。

### 9.6 Notification 路径

`useNotifyAfterTimeout` 监听终端焦点 / 交互间隔，超时则触发 OS notification（iTerm2 / Kitty / Ghostty / bell）。该 hook 通过 `addNotification` 注入到 React state 队列，不阻塞主流程。

---

## 10. 典型链路举例（BashTool）

```
用户: "请删除 tmp 目录"
  → Claude API 流式返回 tool_use(name="Bash", input={"command":"rm -rf tmp"})
  → QueryEngine 派发 BashTool.call(input, ctx, canUseTool, …)
  → BashTool.call() 在执行前 await canUseTool(BashTool, input, …)
       ├─ hasPermissionsToUseTool → behavior="ask"（规则里没有匹配）
       ├─ coordinator / classifier / hook 无自动裁决
       └─ handleInteractivePermission:
            ├─ ctx.pushToQueue({tool, input, onAllow, onReject, …})
            └─ 监听 bridge / channel
  → React state 更新 → REPL 重渲染 <PermissionRequest>
       └─ permissionComponentForTool(BashTool) → <BashPermissionRequest>
            ├─ 显示命令 "rm -rf tmp"
            ├─ 高亮 "No" 选项
            └─ 等待键盘
  → 用户按 ← 切到 "Yes" + Enter
       └─ toolUseConfirm.onAllow(input, [])
            └─ resolveOnce(ctx.buildAllow(...))
  → canUseTool 闭包 resolve
  → BashTool.call() 真正执行 shell
  → result 回灌 messages → 下一轮 API
```

---

## 11. 关键文件索引

| 路径 | 作用 |
| --- | --- |
| `src/hooks/useCanUseTool.tsx:37` | canUseTool 闭包工厂，所有弹框的总入口 |
| `src/hooks/useCanUseTool.tsx:35` | `CanUseToolFn` 类型签名 |
| `src/hooks/toolPermission/handlers/interactiveHandler.ts:57` | pushToQueue + 多路竞态 |
| `src/hooks/toolPermission/PermissionContext.ts:350` | ctx 生命周期管理 |
| `src/utils/permissions/permissions.ts` | 静态规则 → allow/deny/ask |
| `src/components/permissions/PermissionRequest.tsx:47-82` | 工具 → 弹框组件派发表 |
| `src/components/permissions/PermissionRequest.tsx:146-216` | 弹框根组件 |
| `src/components/permissions/AskUserQuestionPermissionRequest/` | 询问弹框实现 |
| `src/components/permissions/BashPermissionRequest/` | Bash 权限弹框 |
| `src/screens/REPL.tsx` | 顶层 React 渲染 |
| `src/QueryEngine.ts` / `src/query.ts` | 驱动 `tool.call()` 派发 |
| `src/Tool.ts:158-300` | `ToolUseContext` 定义 |
| `src/Tool.ts:379-385` | `Tool.call()` 接口签名 |
| `src/Tool.ts:103-114` | `SetToolJSXFn` 类型 |