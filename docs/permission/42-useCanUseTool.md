# useCanUseTool.tsx — 工具权限网关闭包工厂

> 源码位置：`src/hooks/useCanUseTool.tsx`（215 行）
> 消费方：`src/screens/REPL.tsx`（2 处调用：在主线程/子代理中各取一次闭包并通过 `ToolUseContext.canUseTool` 注入）
> 间接消费者：`src/Tool.ts:379-385` 中 `Tool.call()` 第三参即 `CanUseToolFn`；Bash / FileEdit / Skill / Agent / AskUserQuestion 等所有工具在执行前 `await canUseTool(...)`
> 协作模块：
> - `src/hooks/toolPermission/PermissionContext.ts` —— ctx 生命周期
> - `src/hooks/toolPermission/handlers/{coordinator,swarmWorker,interactive}Handler.ts` —— 3 个下游 handler
> - `src/utils/permissions/permissions.ts` —— 静态规则决策（详见 [`docs/permission/30-permission-control-flow.md`](30-permission-control-flow.md)）
> - `src/utils/classifierApprovals.ts` —— classifier 命中状态缓存
> - `src/tools/BashTool/bashPermissions.ts` —— 投机式 classifier check

`useCanUseTool` 是 Claude Code 整个"工具执行能否放行"管线的入口。**所有工具在真正调用前必须 `await canUseTool(tool, input, ctx, msg, id)`，它返回一个 `PermissionDecision`**。本文件只有一个 React hook 工厂加两个工具函数，但它是把"同步权限判定 + 异步 UI 弹框 + 多路远程响应"统一成同一条 Promise 通道的核心。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|---------|
| **`CanUseToolFn<Input>` 类型** | `useCanUseTool.tsx:35` —— 6 参签名：`(tool, input, ctx, msg, toolUseID, forceDecision?) => Promise<PermissionDecision<Input>>`，第 6 参 `forceDecision` 主要给测试与重放路径使用，跳过静态规则直接落定决策 |
| **`useCanUseTool` 闭包工厂** | `useCanUseTool.tsx:37-199` —— 接收 `(setToolUseConfirmQueue, setToolPermissionContext)` 两个 React setter，返回 `CanUseToolFn`。带 `_c(3)` React Compiler 缓存：setter 不变则复用同一闭包 |
| **Stage A — PermissionContext 构造** | `useCanUseTool.tsx:42` —— 调 `createPermissionContext` 拼出 ctx（resolve/reject/logDecision/pushToQueue/runHooks 一站式）；紧接着 `ctx.resolveIfAborted(resolve)` 短路已取消请求 |
| **Stage B — 静态规则决策** | `useCanUseTool.tsx:46` —— `hasPermissionsToUseTool(...)` 评估 allow/deny/ask；若传 `forceDecision` 直接走 `Promise.resolve(forceDecision)` |
| **Stage C — 行为分支** | `useCanUseTool.tsx:47-178` —— 按 `result.behavior` 三路分发：<br>• `allow` (`:48-63`)：yolo classifier 标记 + `logDecision` + `ctx.buildAllow(updatedInput)` resolve<br>• `deny` (`:74-101`)：auto-mode denial 记录 + notification 注入 + resolve `result`<br>• `ask` (`:102-177`)：4 段瀑布式 fallback（coordinator → swarm → bash classifier → interactive） |
| **ask 路径 — Coordinator handler** | `useCanUseTool.tsx:103-117` —— `awaitAutomatedChecksBeforeDialog` 模式下让 coordinator agent 提前裁决；带 `BASH_CLASSIFIER` feature-gate 透传 `pendingClassifierCheck` |
| **ask 路径 — Swarm worker handler** | `useCanUseTool.tsx:121-133` —— swarm worker 模式下路由到 `handleSwarmWorkerPermission`；同样透传 pending classifier |
| **ask 路径 — 投机式 Bash classifier** | `useCanUseTool.tsx:134-167` —— 仅 Bash 工具 + `BASH_CLASSIFIER` 开启时执行：取 `peekSpeculativeClassifierCheck(command)`，与 2 秒 timeout 竞速；命中 high-confidence 规则则立刻 `consumeSpeculativeClassifierCheck` 防止双重消费，并通过 `setClassifierApproval` 标记 + `ctx.buildAllow` resolve |
| **ask 路径 — Interactive handler** | `useCanUseTool.tsx:168-176` —— 最终 fallback，把 ctx + description + result + bridge/channel callbacks 传给 `handleInteractivePermission`，由后者推 React state 队列并起多路竞态 |
| **Stage D — 异常处理** | `useCanUseTool.tsx:179-187` —— `.catch` 内识别 `AbortError` / `APIUserAbortError` → 静默 `logForDebugging` + `ctx.logCancelled()` + cancel resolve；其他 error → `logError` + cancel resolve |
| **Stage D — finally 清理** | `useCanUseTool.tsx:188-190` —— 无论成败都 `clearClassifierChecking(toolUseID)`，防止 toolUseID 状态泄漏 |
| **React Compiler 缓存** | `useCanUseTool.tsx:38-39` / `:192-198` —— `_c(3)` + `$[0]/[1]/[2]` 三槽位：仅当 setter 引用变化时才重建闭包，避免每次 render 都生成新的 `canUseTool` 函数导致下游 `ToolUseContext` 失效 |
| **Analytics 元数据** | 透传 `sanitizeToolNameForAnalytics`（`:10`）给埋点；不在本文件调用，但与 `logPermissionDecision` / `recordAutoModeDenial` 配套使用 |

---

## 一、整体定位与调用链

```
┌──────────────────────────────────────────────────────────────────┐
│                       Tool.call(args, ctx, canUseTool, ...)      │
│                                                                  │
│   BashTool / FileEditTool / AskUserQuestionTool / AgentTool …    │
└─────────────────────────────┬────────────────────────────────────┘
                              │ await canUseTool(tool, input, ctx, msg, id)
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│  useCanUseTool 闭包（src/hooks/useCanUseTool.tsx:41-191）        │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ Stage A   ctx = createPermissionContext(...)             │   │
│   │           if (ctx.resolveIfAborted(resolve)) return      │   │
│   └─────────────────────────┬────────────────────────────────┘   │
│                             ▼                                    │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ Stage B   result = hasPermissionsToUseTool(...)          │   │
│   │           or Promise.resolve(forceDecision)              │   │
│   └─────────────────────────┬────────────────────────────────┘   │
│                             ▼                                    │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ Stage C   switch (result.behavior)                      │   │
│   │   ┌───────────┐  ┌──────────┐  ┌──────────────────────┐  │   │
│   │   │  allow    │  │  deny    │  │       ask            │  │   │
│   │   │           │  │          │  │ coordinator? ──┐     │  │   │
│   │   │ yolo cls  │  │ auto-    │  │ swarm worker? ──┤     │  │   │
│   │   │ + build   │  │ mode     │  │ bash cls spec? ─┤     │  │   │
│   │   │ Allow     │  │ denial + │  │ interactive ────┘     │  │   │
│   │   │ resolve   │  │ notif +  │  │ (最终 fallback)       │  │   │
│   │   │           │  │ resolve  │  │                      │  │   │
│   │   └───────────┘  └──────────┘  └──────────────────────┘  │   │
│   └─────────────────────────┬────────────────────────────────┘   │
│                             ▼                                    │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │ Stage D   .catch(AbortError → cancel silently)          │   │
│   │           .finally(clearClassifierChecking)             │   │
│   └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────┬────────────────────────────────────┘
                              │ resolve(decision)
                              ▼
                      tool.call() 真正执行 / 抛错
```

---

## 二、关键阶段详解

### 2.1 Stage A — PermissionContext 构造

源码：`useCanUseTool.tsx:42-45`

```ts
const ctx = createPermissionContext(
  tool, input, toolUseContext, assistantMessage, toolUseID,
  setToolPermissionContext,
  createPermissionQueueOps(setToolUseConfirmQueue)
);
if (ctx.resolveIfAborted(resolve)) return;
```

- `createPermissionContext` 产出一个 ctx，持有 6 个 React setter + resolve/reject + ctx-internal 状态
- `resolveIfAborted(resolve)` 在 abort 已触发的情况下立即 resolve cancel 决策，避免后续 `.then` 链上做无用功
- 提前 return（不 resolve）会保持 Promise pending，对应"还没到决策就取消"的边缘场景

### 2.2 Stage B — 静态规则决策

源码：`useCanUseTool.tsx:46`

```ts
const decisionPromise = forceDecision !== undefined
  ? Promise.resolve(forceDecision)
  : hasPermissionsToUseTool(tool, input, toolUseContext, assistantMessage, toolUseID);
```

- 正常路径：`hasPermissionsToUseTool` 走 30-permission-control-flow 中描述的 7 步管线
- `forceDecision`：测试与 replay 通道，绕过静态规则直接落定；可被上游 `coordinatorHandler` 注入（例如 replay 时把已记录的人类决策回放一次）

### 2.3 Stage C — `allow` 分支

源码：`useCanUseTool.tsx:48-63`

```ts
if (result.behavior === "allow") {
  if (ctx.resolveIfAborted(resolve)) return;
  // TRANSCRIPT_CLASSIFIER 开启 + auto-mode 命中 → 标记 yolo classifier 批准
  if (feature("TRANSCRIPT_CLASSIFIER") && result.decisionReason?.type === "classifier"
      && result.decisionReason.classifier === "auto-mode") {
    setYoloClassifierApproval(toolUseID, result.decisionReason.reason);
  }
  ctx.logDecision({ decision: "accept", source: "config" });
  resolve(ctx.buildAllow(result.updatedInput ?? input, {
    decisionReason: result.decisionReason
  }));
  return;
}
```

要点：

- `resolveIfAborted` 二次检查：在静态规则评估期间 abort 可能已触发
- yolo classifier 标记：仅在 `auto-mode` 分支下做，bash_allow 分支留给 speculative 路径处理
- `logDecision` 写入 permission 决策日志，供后续 `/permissions` 面板展示与 telemetry 上报
- `ctx.buildAllow` 把 `updatedInput` 注入决策结果，让上游 tool 用工具预处理后的 input 继续执行

### 2.4 Stage C — `deny` 分支

源码：`useCanUseTool.tsx:74-101`

```ts
case "deny": {
  logPermissionDecision({ tool, input, toolUseContext, messageId, toolUseID },
                        { decision: "reject", source: "config" });
  // auto-mode 命中规则被拒 → 记录 denial + 注入通知
  if (feature("TRANSCRIPT_CLASSIFIER") && result.decisionReason?.type === "classifier"
      && result.decisionReason.classifier === "auto-mode") {
    recordAutoModeDenial({ toolName: tool.name, display: description,
                            reason: result.decisionReason.reason ?? "", timestamp: Date.now() });
    toolUseContext.addNotification?.({
      key: "auto-mode-denied",
      priority: "immediate",
      jsx: <><Text color="error">{tool.userFacingName(input).toLowerCase()} denied by auto mode</Text>
            <Text dimColor={true}> · /permissions</Text></>
    });
  }
  resolve(result);
  return;
}
```

要点：

- `description`（`:64-69`）取自 `tool.description(input, ...)`，用于通知文案
- auto-mode denial 触发 `recordAutoModeDenial` 写入降级计数器，达到阈值会让 auto mode 退出
- `addNotification` 注入到 AppState notification 队列，由 REPL 顶部 toast 组件消费
- 直接 `resolve(result)` 透传原 deny decision，不重新包装

### 2.5 Stage C — `ask` 分支（4 段瀑布）

源码：`useCanUseTool.tsx:102-177`

`ask` 行为下，需要用户/自动化/远程 UI 给出最终决策。本文件按"越快越准越先"原则串联 4 个尝试：

```
ask 分支
   │
   ├─ 1. coordinator (awaitAutomatedChecksBeforeDialog === true)
   │     └─ if (coordinatorDecision) { resolve(...); return; }
   │
   ├─ 2. swarm worker (always)
   │     └─ if (swarmDecision) { resolve(...); return; }
   │
   ├─ 3. bash classifier speculative (Bash 工具 + BASH_CLASSIFIER + 没有 coordinator 模式)
   │     └─ if (matches && high confidence) { resolve(buildAllow(...)); return; }
   │
   └─ 4. interactive handler (final fallback)
         └─ handleInteractivePermission({ctx, description, result, ...}, resolve)
            ↳ 不在此处 resolve，把 resolve 闭包交给 handler
```

#### 2.5.1 Coordinator handler（`:103-117`）

- 仅在 `appState.toolPermissionContext.awaitAutomatedChecksBeforeDialog` 为真时触发
- 由 coordinator agent 提前裁决（人类看不到弹框）
- `feature("BASH_CLASSIFIER")` 时透传 `pendingClassifierCheck` 让 coordinator 可以综合 classifier 结果

#### 2.5.2 Swarm worker handler（`:121-133`）

- 总是触发
- 用于 swarm worker 模式下把权限询问路由到 swarm 协调层
- 同样透传 `pendingClassifierCheck`

#### 2.5.3 Bash classifier speculative（`:134-167`）

```ts
if (feature("BASH_CLASSIFIER") && result.pendingClassifierCheck
    && tool.name === BASH_TOOL_NAME
    && !appState.toolPermissionContext.awaitAutomatedChecksBeforeDialog) {
  const speculativePromise = peekSpeculativeClassifierCheck((input as { command: string }).command);
  if (speculativePromise) {
    const raceResult = await Promise.race([
      speculativePromise.then(_temp),
      new Promise(_temp2)   // 2 秒 timeout
    ]);
    // 命中 high-confidence rule → 立即放行
    if (raceResult.type === "result" && raceResult.result.matches
        && raceResult.result.confidence === "high") {
      consumeSpeculativeClassifierCheck((input as { command: string }).command);
      const matchedRule = raceResult.result.matchedDescription ?? undefined;
      if (matchedRule) setClassifierApproval(toolUseID, matchedRule);
      ctx.logDecision({ decision: "accept", source: { type: "classifier" } });
      resolve(ctx.buildAllow(result.updatedInput ?? input, {
        decisionReason: { type: "classifier", classifier: "bash_allow",
                          reason: `Allowed by prompt rule: "${raceResult.result.matchedDescription}"` }
      }));
      return;
    }
  }
}
```

要点：

- 仅 Bash 工具参与投机式 classifier（其他工具走 interactive）
- `peekSpeculativeClassifierCheck(command)` 取出先前流式预产出的 classifier Promise，不存在则跳过
- `Promise.race` 与 2 秒 timeout 竞速；timeout 不会放行，只防止 classifier 慢响应卡死 ask 流程
- `consumeSpeculativeClassifierCheck` 标记该 command 的投机结果已被消费，避免后续重复计算
- 命中规则后用 `decisionReason.type: "classifier", classifier: "bash_allow"` 标记来源，便于下游 UI 显示 ✓ 状态

#### 2.5.4 Interactive handler（`:168-176`）— 最终 fallback

```ts
handleInteractivePermission({
  ctx,
  description,
  result,
  awaitAutomatedChecksBeforeDialog: appState.toolPermissionContext.awaitAutomatedChecksBeforeDialog,
  bridgeCallbacks: feature("BRIDGE_MODE") ? appState.replBridgePermissionCallbacks : undefined,
  channelCallbacks: feature("KAIROS") || feature("KAIROS_CHANNELS")
                    ? appState.channelPermissionCallbacks : undefined
}, resolve);
```

- 把 ctx、decision result、bridge / channel 回调一次性塞给 handler
- **不立即 resolve**：resolve 闭包交给 handler，由 handler 内部 push React state 队列 + 监听 bridge/channel/hook/classifier 多个来源，谁先到谁调 resolve
- feature-flag 控制 bridge / channel 是否启用，详见 [`docs/permission/43-interactive-permission-popup-flow.md`](43-interactive-permission-popup-flow.md) 阶段 3

### 2.6 Stage D — 异常处理与清理

源码：`useCanUseTool.tsx:179-190`

```ts
.catch(error => {
  if (error instanceof AbortError || error instanceof APIUserAbortError) {
    logForDebugging(`Permission check threw ${error.constructor.name} for tool=${tool.name}: ${error.message}`);
    ctx.logCancelled();
    resolve(ctx.cancelAndAbort(undefined, true));
  } else {
    logError(error);
    resolve(ctx.cancelAndAbort(undefined, true));
  }
}).finally(() => {
  clearClassifierChecking(toolUseID);
});
```

要点：

- **AbortError 与其他 error 区分对待**：abort 是用户主动取消，debug 级别日志；其他 error 是 bug，需要 `logError`
- 两类 error 最终都 `cancelAndAbort(undefined, true)` —— resolve 一个 deny decision + 触发 abortController.signal
- `.finally` 兜底清理 classifier 标记，无论决策路径是否正常结束，避免 toolUseID 状态泄漏到下一次请求

---

## 三、与 Tool 接口的契约

`Tool.call()` 接口（`src/Tool.ts:379-385`）：

```ts
call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,            // ← 本文件产物
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<P>,
): Promise<ToolResult<Output>>
```

- 每个工具实现 `call()` 时，**必须在做实际工作前** `await canUseTool(tool, input, context, parentMessage, toolUseID)`
- 若工具内部还有子工具调用（例如 AgentTool fork subagent），需要复用父 ctx 的 `canUseTool` 或构造自己的 ctx
- 决策为 deny 时工具应 throw 或返回 `{ is_error: true }`，让模型收到反馈继续对话

---

## 四、关键文件索引

| 路径 | 作用 |
| --- | --- |
| `src/hooks/useCanUseTool.tsx:35` | `CanUseToolFn<Input>` 类型签名 |
| `src/hooks/useCanUseTool.tsx:37-199` | `useCanUseTool` 闭包工厂主体 |
| `src/hooks/useCanUseTool.tsx:42` | `createPermissionContext` 调用 |
| `src/hooks/useCanUseTool.tsx:46` | 静态规则决策入口（含 forceDecision 旁路） |
| `src/hooks/useCanUseTool.tsx:48-63` | `allow` 分支 |
| `src/hooks/useCanUseTool.tsx:74-101` | `deny` 分支 |
| `src/hooks/useCanUseTool.tsx:102-177` | `ask` 分支 4 段瀑布 |
| `src/hooks/useCanUseTool.tsx:134-167` | 投机式 Bash classifier |
| `src/hooks/useCanUseTool.tsx:168-176` | Interactive handler 最终 fallback |
| `src/hooks/useCanUseTool.tsx:179-190` | 异常处理 + finally 清理 |
| `src/hooks/toolPermission/PermissionContext.ts` | ctx 生命周期 / resolve / cancel / logDecision / pushToQueue |
| `src/hooks/toolPermission/handlers/interactiveHandler.ts` | 弹框发起 + 多路竞态 |
| `src/hooks/toolPermission/handlers/coordinatorHandler.ts` | coordinator 模式裁决 |
| `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts` | swarm worker 路由 |
| `src/hooks/toolPermission/permissionLogging.ts` | `logPermissionDecision` |
| `src/utils/permissions/permissions.ts` | 静态规则核心 |
| `src/utils/classifierApprovals.ts` | `setYoloClassifierApproval` / `setClassifierApproval` / `clearClassifierChecking` |
| `src/tools/BashTool/bashPermissions.ts` | `peekSpeculativeClassifierCheck` / `consumeSpeculativeClassifierCheck` |
| `src/utils/autoModeDenials.ts` | `recordAutoModeDenial`（auto-mode 计数器） |
| `src/screens/REPL.tsx` | 唯一调用方，2 处 |

---

## 五、典型执行链路（BashTool 跑 `npm install`）

```
1. 用户输入 "安装依赖"
2. Claude API 流式返回 tool_use(name="Bash", input={"command":"npm install"})
3. QueryEngine 派发 BashTool.call(input, ctx, canUseTool, …)
4. BashTool.call():
     await canUseTool(BashTool, {command:"npm install"}, ctx, msg, toolUseID)
5. useCanUseTool 闭包:
     a) createPermissionContext → ctx
     b) hasPermissionsToUseTool → { behavior: "ask" }
     c) 拿 tool.description(input) → "Install npm dependencies"
     d) coordinator? false (用户没开 auto 模式)
     e) swarm worker? null decision
     f) bash classifier? peekSpeculativeClassifierCheck("npm install") 返回 undefined
        (没命中预产出) → 跳过
     g) handleInteractivePermission({ctx, description, result, …}, resolve)
6. handleInteractivePermission:
     ctx.pushToQueue({tool, input, onAllow, onReject, …})  → REPL 重渲染
     + 监听 bridge / channel / hook 竞态
7. 用户在弹框里按下 "Yes"
     toolUseConfirm.onAllow(input, []) → resolveOnce(ctx.buildAllow(...))
8. canUseTool 闭包 resolve(allow decision)
9. BashTool.call() 拿到 decision, 真正 spawn shell 跑 npm install
10. result 回灌 messages → 下一轮 API
```

---

## 六、关联文档

- [`docs/permission/30-permission-control-flow.md`](30-permission-control-flow.md) —— 静态规则决策（Stage B 详细管线）
- [`docs/permission/40-permission-system-design.md`](40-permission-system-design.md) —— 权限系统设计
- [`docs/permission/41-bash-tool-permission-control.md`](41-bash-tool-permission-control.md) —— Bash 工具细分规则
- [`docs/permission/42-toolUseContext-vs-toolPermissionContext.md`](42-toolUseContext-vs-toolPermissionContext.md) —— ToolUseContext vs ToolPermissionContext
- [`docs/permission/43-interactive-permission-popup-flow.md`](43-interactive-permission-popup-flow.md) —— Interactive handler 与多路竞态（ask 路径的终点）
- [`docs/hook/15-utils-hooks.md`](../hook/15-utils-hooks.md) —— Hook 系统执行核心
- [`docs/hook/41-hook-system-design.md`](../hook/41-hook-system-design.md) —— Hook 系统设计
