# AgentTool — 子代理调用总控

> 源码位置：`src/tools/AgentTool/AgentTool.tsx`（1447 行）
> 关联模块：`UI.tsx`、`runAgent.ts`、`agentToolUtils.ts`、`forkSubagent.ts`、`loadAgentsDir.ts`、`spawnMultiAgent.ts`
> 工具名：`AGENT_TOOL_NAME`（别名 `LEGACY_AGENT_TOOL_NAME`）

`AgentTool` 是 Claude Code 中唯一允许主代理把任务「外包」给另一个代理运行的入口。它把"派发"这个动作收敛到一处，承载了权限校验、代理选择、提示词构建、隔离/工作树/远程执行、后台任务、fork 路径、计费遥测等所有非平凡的子代理派发逻辑。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 工具定义 | `buildTool` 工厂构建：schema（基础 + 多代理 + 隔离）、`prompt`（动态生成可用的 agent 列表）、`call`（核心派发）、UI 渲染（`renderToolUseMessage` / `renderToolResultMessage` / `renderToolUseProgressMessage` 等） |
| 代理路由 | 解析 `subagent_type`；缺省时按 `isForkSubagentEnabled()` 选择 `FORK_AGENT`（隐式 fork）还是回退到 `general-purpose`；通过 `filterDeniedAgents` / `filterAgentsByMcpRequirements` 过滤不可用的代理 |
| 多模式执行 | 同步执行、后台执行、team-mate 派发（in-process / tmux / iTerm2）、远程 CCR 派发（ant-only）、工作树隔离、`cwd` 覆盖（KAIROS） |
| Fork 路径 | 复用父代理的 system prompt + 工具池（`useExactTools`），按 `buildForkedMessages` 克隆完整 assistant 消息 + 占位 tool_result，强制所有派生走 async 路径（统一 `<task-notification>` 交互模型） |
| 后台与前台互转 | 通过 `registerAgentForeground` 取得 `backgroundSignal`，`Promise.race` 抢占；中途被打入后台时把 foreground 迭代器的 `finally` 跑掉，再 `runAgent` 的 async 流续接；统一走 `runAsyncAgentLifecycle` 完成 finalization |
| 安全/审计 | `checkPermissions` 在 auto 模式走 classifier；`classifyHandoffIfNeeded`（`TRANSCRIPT_CLASSIFIER`）对子代理的最终输出做交接审查；`logEvent` 上报 `tengu_agent_tool_selected` / `_completed` / `_terminated` / `_remote_launched` / `_memory_loaded` / `tengu_auto_mode_decision` |
| Worktree 隔离 | 入口处 `createAgentWorktree` 派生分支；退出时 `cleanupWorktreeIfNeeded` 检查 `hasWorktreeChanges`，无改动则 `removeAgentWorktree`；hook-based worktree 永远保留 |
| 进度回放 | `Progress` 联合 `AgentToolProgress | ShellProgress`：子代理里 bash/powershell 的 `progress` 事件向上透传，保证 SDK 收到 `tool_progress`；UI 通过 `renderToolUseProgressMessage` + `renderGroupedAgentToolUse` 渲染可折叠转写 |
| 缓存友好 | fork 路径继承父 system prompt 与工具池以获得 byte-identical 前缀；普通路径独立 `assembleToolPool` 避免污染父缓存；`runWithAgentContext` 用 ALS 把 `parentSessionId` / `invokingRequestId` 带进子代理 |
| 死代码消除 | `feature('KAIROS')` / `"external" === 'ant'` / `feature('FORK_SUBAGENT')` 守卫分支；`outputSchema` 用 `lazySchema` + `.omit()` 隐藏未启用参数，避免模型看到 no-op 字段 |

---

## 核心执行流程

`AgentTool.call(input, toolUseContext, canUseTool, assistantMessage, onProgress)` 是整个派发的核心。下图按函数顺序铺平关键决策点：

```text
                 ┌──────────────────────────────────────────┐
                 │  call() 接收到 model 的 Agent tool_use   │
                 └────────────────────┬─────────────────────┘
                                      │
       1. 取 appState、permissionMode、rootSetAppState（穿透 InProcessTeammate）
       2. 团队/后台能力守卫（team_name 需 Agent Swarms；InProcessTeammate 不能 spawn background）
                                      │
                                      ▼
        ┌────────────────────────────────────────────────────┐
        │  teamName && name  ⇒  spawnTeammate(...)           │
        │   ├─ isInProcessEnabled() ⇒ handleSpawnInProcess   │
        │   ├─ handleSpawnSplitPane (tmux/iTerm2)             │
        │   └─ markInProcessFallback (auto 模式回退)          │
        │   返回 TeammateSpawnedOutput，UI 标 teammate_spawned │
        └────────────────────────────────────────────────────┘
                                      │
                                      ▼
        ┌────────────────────────────────────────────────────┐
        │  effectiveType = subagent_type ?? (fork ? undef : general-purpose)
        │  isForkPath = effectiveType === undefined           │
        │  ├─ Fork: 检测递归（querySource + message 扫描），    │
        │  │       selectedAgent = FORK_AGENT                │
        │  └─ 普通: filterDeniedAgents + allowedAgentTypes    │
        │          找不到则抛错（含 deny rule 提示）          │
        └────────────────────────────────────────────────────┘
                                      │
                                      ▼
        ┌────────────────────────────────────────────────────┐
        │  effectiveIsolation = isolation ?? agentDef.isolation
        │  if remote (ant-only): 走 teleportToRemote + registerRemoteAgentTask
        │  if worktree:          createAgentWorktree(slug)
        │  if cwd:               wrapWithCwd 包装器
        └────────────────────────────────────────────────────┘
                                      │
                                      ▼
        ┌────────────────────────────────────────────────────┐
        │  Fork 路径:                                          │
        │   forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
        │                            (fallback: 重新构造)
        │   promptMessages = buildForkedMessages(prompt, assistantMessage)
        │   if worktree:  + buildWorktreeNotice
        │                                                        │
        │  普通路径:                                            │
        │   agentPrompt = selectedAgent.getSystemPrompt(...)    │
        │   enhancedSystemPrompt = await enhanceSystemPromptWithEnvDetails
        │   promptMessages = [createUserMessage(prompt)]         │
        └────────────────────────────────────────────────────┘
                                      │
                                      ▼
        ┌────────────────────────────────────────────────────┐
        │  workerTools = assembleToolPool(workerPermissionContext)
        │  shouldRunAsync = (run_in_background | def.background | isCoordinator
        │                   | forceAsync(fork) | assistantForceAsync(kairos)
        │                   | proactive) && !isBackgroundDisabled
        └─────────────────────────┬──────────────────────────┘
                                  │
                  ┌───────────────┼────────────────┐
                  │               │                │
            shouldRunAsync   shouldRunAsync   shouldRunAsync
             = false          = true           = true (from sync path,
                                  │            backgrounded mid-flight)
                  ▼               ▼                ▼
        ┌──────────────┐ ┌────────────────┐ ┌──────────────────────┐
        │ 同步执行     │ │ 直接后台       │ │ foreground 启动后     │
        │ runAgent     │ │ registerAsync  │ │ 被 backgroundAll()    │
        │ 迭代消息     │ │ runAsyncAgent  │ │ → 切到 async 续接     │
        │ 前台 race    │ │ Lifecycle      │ │   (handleBackground)  │
        │ 返回 completed│ │ 通知 + worktree│ │                      │
        └──────────────┘ └────────────────┘ └──────────────────────┘
```

### 同步路径细节

`shouldRunAsync === false` 的分支在第 832 行起：

1. 用 `runWithAgentContext` 包裹（提供 analytics attribution），用 `wrapWithCwd` 包裹（提供 cwd 覆盖）
2. `registerAgentForeground` 拿到 `foregroundTaskId` 和 `backgroundSignal`；`getAutoBackgroundMs()` 由 `CLAUDE_AUTO_BACKGROUND_TASKS` 或 GrowthBook `tengu_auto_background_agents` 开启（默认 120s）
3. 第一次 `onProgress` 推送 prompt 元数据
4. `while (true)` 循环：`Promise.race([nextMessage, backgroundPromise])`
   - `nextMessage`：`runAgent` 的 async iterator
   - `backgroundPromise`：用户的 `backgroundAll()` 触发
5. 后台接管路径（`raceResult.type === 'background'`）：
   - 调 `agentIterator.return(undefined)` 跑完 foreground 的 `finally`（释放 MCP 连接、prompt cache 追踪、session hooks），1s 超时避免挂死
   - 把已经累积的消息灌进新 `tracker`
   - 用 `task.abortController` 再起一次 `runAgent({ isAsync: true, ... })` 的 async 流
   - 完成后 `completeAsyncAgent` → `enqueueAgentNotification`
   - finally 中清理 `clearInvokedSkillsForAgent` / `clearDumpState`（不清理 worktree —— 由 async 路径自己清）
6. `nextMessage` 路径：
   - `bash_progress` / `powershell_progress` 透传给 `onProgress`（SDK 收到 `tool_progress`）
   - 正常消息塞入 `agentMessages`，调用 `updateProgressFromMessage` + `emitTaskProgress`（VS Code subagent panel + SDK 事件）
   - 每条 assistant 消息调 `setResponseLength` 给主代理 spinner 累加 token
7. 异常处理：AbortError 直接 throw（保留中断语义）；普通错误存入 `syncAgentError` 后继续 —— 若累积到 assistant 消息则 finalization 后返回，否则重新抛
8. `finally` 中：清 JSX、停止 summarization、`unregisterAgentForeground`、按状态发 SDK `task_notification`、`clearInvokedSkillsForAgent` / `clearDumpState`、调用 `cleanupWorktreeIfNeeded`（已后台化则跳过）
9. `classifyHandoffIfNeeded`（`TRANSCRIPT_CLASSIFIER` + auto 模式）对子代理最终输出做交接审查，把警告拼到 `content` 前面
10. 返回 `{ data: { status: 'completed', prompt, ...agentResult, ...worktreeResult } }`

### 异步路径细节

`shouldRunAsync === true` 的分支在第 733 行起：

1. `registerAsyncAgent` 拿到 `agentBackgroundTask`（独立 `abortController`，不受父 ESC 影响）
2. 如果带 `name`，写 `appState.agentNameRegistry`，供 `SendMessage({ to: name })` 路由
3. `runWithAgentContext` + `wrapWithCwd` 包裹的 `void` fire-and-forget：
   - 调用 `runAsyncAgentLifecycle`（agentToolUtils.ts），该函数会：
     - 启动 summarization（若 `isCoordinator || isForkSubagentEnabled || getSdkAgentProgressSummariesEnabled`）
     - 持续 `for await (msg of makeStream)`，每条消息更新 progress tracker、emit `task_progress` SDK 事件、把消息 append 到 retain 任务的 `messages`
     - 完成后 `finalizeAgentTool` → `completeAsyncAgent`（先于 classifier / worktree，把 gh-20236 的 TaskOutput 阻塞风险压到 0）
     - `classifyHandoffIfNeeded` + `cleanupWorktreeIfNeeded` + `enqueueAgentNotification`
4. `return { data: { status: 'async_launched', agentId, description, prompt, outputFile, canReadOutputFile } }`

### Map 结果到 tool_result

`mapToolResultToToolResultBlockParam`（第 1346 行）按 `status` 分支：
- `teammate_spawned`：返回带 `agent_id` / `name` / `team_name` 的"已生成"文本
- `remote_launched`：返回带 `taskId` / `session_url` / `output_file` 的远程链接
- `async_launched`：返回带 `agentId` + 是否可读 `outputFile` 的两条提示
- `completed`：拼上 `agentId`（供 SendMessage 续接）和 `<usage>` 块；对 ONE_SHOT_BUILTIN_AGENT_TYPES（Explore、Plan）若无 worktree 提示就丢掉 trailer 节省 ~135 字符 × 34M calls/week
- 末尾用 `data satisfies never` 静态检查全覆盖

---

## 关键设计取舍

- **Fork vs subagent_type**：fork 路径继承父代理的 system prompt 和工具池（`useExactTools`），可命中父代理的 prompt cache；`subagent_type` 路径独立 `assembleToolPool`、独立 system prompt，绝不污染父 cache。两者目标场景不同。
- **后台统一交互**：开启 fork 路径后 `forceAsync = true` —— 全部派生走 async，统一收 `<task-notification>`，避免 sync 阻塞主 turn 导致 cron catch-up 阻塞所有用户输入（注释见 600-612 行）。
- **错误恢复**：同步路径不立即 throw，会尝试用累积到的消息 finalization 后再返回；抛 AbortError 时保留中断语义。Async 路径在 `killAsyncAgent` 后再清 worktree / 发通知，确保 TaskOutput 不被 git 操作阻塞。
- **Worktree 协同**：清理逻辑只在「agent 真的没改动」时删除（`hasWorktreeChanges` 对比 `headCommit`），hook-based worktree 永远保留（无法检测 VCS 变化）。Fork + worktree 时 `buildWorktreeNotice` 提醒子代理翻译父上下文里的路径、重新读取文件。
- **遥测归一**：所有结果路径都通过 `logEvent` 上报同一组 key，便于在 BI 里按 `agent_type` / `is_async` / `is_built_in_agent` 切片；`tengu_cache_eviction_hint` 告诉推理层子代理结束、可以驱逐该 cache 链。
- **死代码消除**：`outputSchema` 用 `lazySchema` + `feature('KAIROS')` / `isBackgroundTasksDisabled` / `isForkSubagentEnabled()` 控制字段暴露 —— `.omit()` 在 Zod 4 里类型会塌缩到 unknown，所以 schema 用 `return` 表达式返回 narrow 类型，而 `AgentToolInput` 显式 widen 保留所有可选字段。
- **类型安全 + dead code 平衡**：`TeammateSpawnedOutput` 与 `RemoteLaunchedOutput` 不放进 `outputSchema`（避免启用前产生 schema 噪音），但在 TS 类型上导出，用 `as unknown as { data: Output }` cast —— UI.tsx 用 export 的 `RemoteLaunchedOutput` 做 discriminated-union narrowing 而非 ad-hoc 断言。

---

## 调用与被调用全景

```text
主代理 query → runToolUse (toolExecution.ts)
                ├─ findToolByName → AgentTool
                └─ AgentTool.call(args, toolUseContext, canUseTool, assistantMessage, onProgress)
                   ├─ prompt()  ← getPrompt(filteredAgents, isCoordinator, allowedAgentTypes)
                   ├─ resolveTeamName / isTeammate / isInProcessTeammate 守卫
                   ├─ (multi-agent) spawnTeammate → handleSpawn → handleSpawnInProcess / handleSpawnSplitPane
                   ├─ (remote isolation, ant-only) teleportToRemote + registerRemoteAgentTask
                   ├─ (fork) buildForkedMessages + isInForkChild 守卫
                   ├─ (worktree) createAgentWorktree + wrapWithCwd
                   ├─ (async)  registerAsyncAgent → runAsyncAgentLifecycle → runAgent → enqueueAgentNotification
                   ├─ (sync)   runAgent iterator + Promise.race(backgroundSignal) → 中途 backgrounded 切 async
                   ├─ finalization:  finalizeAgentTool + classifyHandoffIfNeeded + cleanupWorktreeIfNeeded
                   └─ mapToolResultToToolResultBlockParam → 拼 tool_result 文本

UI:
  renderToolUseMessage          ← 简单显示 description
  renderToolResultMessage       ← async_launched / remote_launched / completed 三种 layout
  renderToolUseProgressMessage  ← 折叠转写 + condensed 模式（终端过窄时只显示计数）
  renderToolUseRejectedMessage  ← renderToolUseProgressMessage + FallbackToolUseRejectedMessage
  renderToolUseErrorMessage     ← renderToolUseProgressMessage + FallbackToolUseErrorMessage
  renderGroupedAgentToolUse     ← 多 Agent 并发时按行聚合（带 "Done (N tool uses · M tokens · Xs)"）
```

---

## 文件职责速查

| 文件 | 行数 | 职责 |
|------|-----:|------|
| `AgentTool.tsx` | 1447 | 工具定义、`call()` 主流程、team/remote/fork/async 分发、worktree 协调、mapToolResultToToolResultBlockParam |
| `UI.tsx` | ~760 | 全部 React 渲染：进度折叠、聚合展示、转写模式、键盘快捷键提示、结果摘要 |
| `runAgent.ts` | — | 真正的子代理 query 循环：调用 API、流式吐消息、应用 permission、写入 sessionStorage |
| `agentToolUtils.ts` | ~686 | `finalizeAgentTool`（耗时/Token 统计 + 遥测）、`classifyHandoffIfNeeded`（交接审查）、`runAsyncAgentLifecycle`（async 路径共享 lifecycle） |
| `forkSubagent.ts` | 210+ | `isForkSubagentEnabled` 守卫、`FORK_AGENT` 合成定义、`buildForkedMessages` 字节级缓存复用、`buildChildMessage` 强约束工作指令、`buildWorktreeNotice` 路径翻译提示 |
| `loadAgentsDir.ts` | — | 从 `/agents/*.md`、JSON 配置、plugin 加载 agent 定义；`hasRequiredMcpServers` 校验依赖；`isBuiltInAgent` / `isCustomAgent` / `isPluginAgent` 类型守卫 |
| `prompt.ts` | 280+ | `getPrompt` 拼装 `Agent` 工具描述：可用的 agent 列表、用法、fork 提示、示例、coordinator 模式精简版 |
| `built-in/generalPurposeAgent.ts` | — | `GENERAL_PURPOSE_AGENT` 内置定义（无 subagent_type 时的兜底） |
| `agentColorManager.ts` | — | `setAgentColor` / `getAgentColor`：grouped UI 上色 |
| `constants.ts` | — | `AGENT_TOOL_NAME` / `LEGACY_AGENT_TOOL_NAME` / `ONE_SHOT_BUILTIN_AGENT_TYPES`（Explore、Plan） |
| `resumeAgent.ts` | — | 续接已派生的 agent（`SendMessage` / `chat:resumeAgent` 入口），与本文件配合 |
| `spawnMultiAgent.ts` | 1090+ | 团队 teammate 派发：`handleSpawn` → in-process / split-pane / separate window |

---

## 重要不变量

- **abort controller 隔离**：async agent 的 abortController 来自 `registerAsyncAgent`，**不**继承父的 `toolUseContext.abortController`；主代理按 ESC 时不会杀掉后台 agent，必须通过 `chat:killAgents` 显式终止。Sync agent 走父 controller。
- **agentId 稳定性**：`createAgentId()` 在 `call()` 入口就建好，贯穿 worktree slug、async 注册、map 回去的 `data.agentId`，保证 SendMessage 续接始终能命中。
- **缓存破坏零容忍**：fork 路径必须用父的 `renderedSystemPrompt` 而非重新调 `getSystemPrompt` —— 后者会因 GrowthBook 冷热状态差异 bust 父 prompt cache（注释见 forkSubagent.ts:54-58）。
- **顺序保障**：async 路径里 `completeAsyncAgent` 必须在 `classifyHandoffIfNeeded` / `cleanupWorktreeIfNeeded` 之前（gh-20236：classifier 调 API、git exec 都可能挂；status 转换不该被它们门控）。
- **Cleanup 对称**：foreground 切到 async 时，foreground 的 cleanup（summarization stop、dumpState、invoked skills）让 async 路径的 finally 接手；worktree 清理则反之 —— sync 切 async 时由 async 路径清理，避免双重清理。
