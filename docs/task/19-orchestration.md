# Claude Code 任务编排范式：工业流水线式 Agent 治理

> 范围：四根支柱在源码中的实现——先规划（Plan Mode + TaskCreate）、后执行（TodoWrite/TaskUpdate + Tool Runner）、强评审（Verification Agent + Hooks + Plan Verify）、分治并行（AgentTool + Coordinator + Team + Ultraplan）。
>
> 上一文（《18-task-planning.md》）讲"任务列表的数据结构与读写"，本文讲"任务列表如何驱动一个工业级 agent 治理闭环"。

## 0. 总览：四根支柱与一句工业比喻

把 Claude Code 想象成一条**数字工厂的装配线**，每个 LLM 主循环都是一台"中央调度 PLC"：

| 支柱 | 装配线等价物 | 源码入口 | 关键失败模式 |
| --- | --- | --- | --- |
| **先规划** | 工单蓝图 + 工序表 | `EnterPlanModeTool`（`src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:36`）/ `ExitPlanModeV2Tool`（`src/tools/ExitPlanModeTool/`）/ `TaskCreate`（`src/tools/TaskCreateTool/`） | 跳过规划直接动手改代码 |
| **后执行** | 自动机床按工序加工 | `TodoWrite`（`src/tools/TodoWriteTool/TodoWriteTool.ts:31`）/ `TaskUpdate`（`src/tools/TaskUpdateTool/TaskUpdateTool.ts:88`）/ StreamingToolExecutor | 工序未完工就切下一道 |
| **强评审** | 质检 + 安全门 | `VERIFICATION_AGENT`（`src/tools/AgentTool/built-in/verificationAgent.ts`）/ Hooks（`src/utils/hooks.ts:3986`）/ `tengu_hive_evidence` nudge（`TaskUpdateTool.ts:333-349`） | 自我"看起来对了"就放行 |
| **分治并行** | 多台机床 + 多个工人 | `AgentTool`（`src/tools/AgentTool/AgentTool.tsx:243`）/ Coordinator mode（`src/coordinator/coordinatorMode.ts:36`）/ Team（`src/tools/TeamCreateTool/`、`src/tools/TeamDeleteTool/`）/ Ultraplan（`src/commands/ultraplan.tsx`） | 单线程串行做完全部 |

`docs/18-task-planning.md` 中已经详细解析过的"任务数据层"是**贯穿四条支柱的工件总线**——任何一条支柱都在这条总线上读写。

## 1. 先规划（Plan First）

### 1.1 双层规划：Plan Mode 决定"是否动手"，TaskCreate 决定"做哪些步"

#### 1.1.1 Plan Mode：宏观硬门

Plan Mode 是一条**在执行前强制要求签字的硬门**。`EnterPlanModeTool.call`（`src/tools/EnterPlanModeTool/EnterPlanModeTool.ts:77`）做三件事：

```ts
// EnterPlanModeTool.ts:77
async call(_input, context) {
  if (context.agentId) throw new Error('EnterPlanMode tool cannot be used in agent contexts')
  handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')
  context.setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(
      prepareContextForPlanMode(prev.toolPermissionContext),
      { type: 'setMode', mode: 'plan', destination: 'session' },
    ),
  }))
  return { data: { message: 'Entered plan mode. You should now focus on exploring the codebase…' } }
}
```

要点：

- **agent 内部禁用**——子 agent 不能自启 plan mode（避免子任务无谓地卡规划）。这条规则由 `if (context.agentId)` 显式拦截。
- **真"硬门"是 `mode: 'plan'`**——权限上下文被切到 plan 后，所有可写工具（Edit/Write/Bash/Agent 等）默认被拒。模型只能做只读探索 + `AskUserQuestion` + ExitPlanMode。
- **Kairos/Channel 模式被禁用**——`isEnabled()`（`EnterPlanModeTool.ts:56-67`）判断 `KAIROS` 或 `KAIROS_CHANNELS` 时如果配了 channel，直接禁用 EnterPlanMode，因为"通道模式"下 ExitPlanMode 的审批对话框要 terminal。

#### 1.1.2 ExitPlanModeV2：微观审批

模型完成规划后调 `ExitPlanModeV2Tool`（`src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`），工具流里携带：

- `plan`：完整方案文本
- `allowedPrompts`：声明需要的语义权限（`[{ tool: 'Bash', prompt: 'run tests' }, …]`）
- `teammate_approval_request`：当启用了 agent swarms 且本人是 teammate 时，给 leader 发 `plan_approval_request` 消息（`ExitPlanModeV2Tool.ts:279`），由 leader 决定
- `is_teammate_request`：标记这是 teammate 申请被 leader 审批（`ExitPlanModeV2Tool.ts:135`）

之后 UI 弹出"批准/拒绝/编辑/Ultraplan 升级"四种动作。**审批不通过则不会进入执行阶段**——这是工业流水线的"工单签字"。

### 1.2 第二层：TaskCreate 把方案拆成结构化工序

Plan 获批后，模型在执行前再调 `TaskCreate` 创建一系列 Task（详见 18-task-planning.md）。两层的关系：

```
┌─────────────────────────────────────────────────┐
│  Plan Mode（一次性）                              │
│   - 写方案文本                                     │
│   - 申请权限（allowedPrompts）                    │
│   - 拿签字                                          │
└──────────────┬───────────────────────────────────┘
               │ 审批通过
               ▼
┌─────────────────────────────────────────────────┐
│  TaskCreate（每个长任务一次）                      │
│   - 拆分子目标                                      │
│   - 设置 blocks/blockedBy 表达依赖                  │
│   - 写入 ~/.claude/tasks/<listId>/*.json          │
└──────────────┬───────────────────────────────────┘
               │
               ▼
        [执行阶段，TaskUpdate 推进]
```

**关键设计**：Plan 解决"要不要做、怎么做、需要哪些权限"，Task 解决"现在轮到哪一步、前置条件满足没"。前者是"道"，后者是"术"。

### 1.3 Ultraplan：把"先规划"推到极致

`src/commands/ultraplan.tsx` 实现了一种**比 Plan Mode 更激进的"分离式规划"**——把规划本身交给远端 Claude Code on the web（CCR）执行。

```ts
// ultraplan.tsx:24
const ULTRAPLAN_TIMEOUT_MS = 30 * 60 * 1000   // 30 分钟
```

工作流：

1. 用户说"ultraplan …"或 trigger keyword。
2. `launchUltraplan`（`ultraplan.tsx:234`）把当前种子计划 + 描述发到远端 Opus（`getUltraplanModel()`，`ultraplan.tsx:32`）。
3. 本地进入**空闲但持续轮询**状态——`pollForApprovedExitPlanMode`（`ccrSession.ts`）每 3 秒扫一次远端事件流，分类出 `approved` / `teleport` / `rejected` / `pending` / `terminated` / `unchanged`（`ccrSession.ts:50-56`）。
4. 远端规划完成 → 用户在浏览器 PlanModal 选"execute remote"或"teleport to terminal"。
5. **Teleport 路径**：plan 文本被 sentinel 标记（`ULTRAPLAN_TELEPORT_SENTINEL`）后流回本地，进入 `ultraplanPendingChoice`，REPL 挂 `UltraplanChoiceDialog` 让用户决定怎么落地。
6. **Remote 路径**：直接在 web 端执行，结束回 PR。

这条路径对应工业语境中的"**前期可研外包**"——把最昂贵、最大的规划脑力外包给独立大模型，本地只跑确定性的执行。

## 2. 后执行（Sequential Execution with Explicit State）

### 2.1 TaskList 作为"工序看板"

执行阶段，模型通过 `TaskUpdate({taskId, status: 'in_progress'})` 报工开工、`status: 'completed'` 报工完工。`src/utils/tasks.ts:541` 的 `claimTask` 提供了强一致性的声明机制（带 `checkAgentBusy` 选项），避免两个 worker 抢同一道工序。

### 2.2 自动化"自动收尾"——3+ 任务后强制自检

`src/tools/TaskUpdateTool/TaskUpdateTool.ts:333-349`：

```ts
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
  !context.agentId &&
  updates.status === 'completed'
) {
  const allTasks = await listTasks(taskListId)
  const allDone = allTasks.every(t => t.status === 'completed')
  if (allDone && allTasks.length >= 3 && !allTasks.some(t => /verif/i.test(t.subject))) {
    verificationNudgeNeeded = true
  }
}
```

翻译：**当主线程（非子 agent）一次性关闭 3+ 任务且没人叫过 verification 时，nudge 强制要求派验证子 agent**。

这条逻辑在 V1 `TodoWriteTool`（`src/tools/TodoWriteTool/TodoWriteTool.ts:76-86`）里有镜像实现——确保两套任务系统都遵循同样的"自检硬门"。

`mapToolResultToToolResultBlockParam` 把这条 nudge 翻译成可读文字追加到 tool result：

```ts
// TaskUpdateTool.ts:396-398
if (verificationNudgeNeeded) {
  resultContent += `\n\nNOTE: You just closed out 3+ tasks and none of them was a verification step.
Before writing your final summary, spawn the verification agent (subagent_type="${VERIFICATION_AGENT_TYPE}").
You cannot self-assign PARTIAL by listing caveats in your summary — only the verifier issues a verdict.`
}
```

——`VERIFICATION_AGENT_TYPE = 'verification'`（`src/tools/AgentTool/constants.ts:4`）。

### 2.3 进程内事件 + 跨进程 fs.watch 双路刷新

详见 18-task-planning.md §5.2。这里强调**执行阶段也享受同一条刷新管线**——任何 worker 写完任务，前台主线程、其它 worker、UI 全部即时可见。

### 2.4 执行单元：StreamingToolExecutor + 子 agent 互不串扰

`AgentTool.call`（`src/tools/AgentTool/AgentTool.tsx:286`）的核心模式：

- 同步派工：`run_in_background: false` → 主线程串行等待。
- 异步派工：`run_in_background: true` → 注册到 `AppState.tasks`（`type: 'local_agent'`），主线程继续走自己的循环。
- 进程隔离：`isolation: 'worktree'` → 在 git worktree 中启动 subagent（`AgentTool.tsx:146`），避免污染主工作树。

每个子 agent 携带自己的 `AsyncLocalStorage<AgentContext>`（`src/utils/agentContext.ts:93`），包含 `agentType: 'subagent'` 或 `'teammate'`。**这条机制保证并发场景下日志/权限/任务归属不串扰**——`agentContext.ts:17-22` 注释明确指出 AppState 单一共享会出现"agent A 的事件被记到 agent B 头上"。

## 3. 强评审（Strong Review Gate）

这是与同类 agent 框架**最大的差异**。Claude Code 不信任 LLM 的"自我感觉"，专门构建了多道评审关。

### 3.1 评审 Agent 本身：对抗性验证

`src/tools/AgentTool/built-in/verificationAgent.ts:10-12`：

```
Your job is not to confirm the implementation works — it's to try to break it.

You have two documented failure patterns. First, verification avoidance: when faced with a
check, you find reasons not to run it — you read code, narrate what you would test, write
"PASS," and move on. Second, being seduced by the first 80%: you see a polished UI or a
passing test suite and feel inclined to pass it, not noticing half the buttons do nothing…
```

关键约束（`verificationAgent.ts:14-22`）：

- **禁止改项目文件**——只能写到 `/tmp`。
- **每条检查必须带 `Command run` + `Output observed` 块**——`A check without a Command run block is not a PASS — it's a skip.`
- **必须以 `VERDICT: PASS|FAIL|PARTIAL` 结尾**——`PARTIAL` 仅用于环境限制（无测试框架、工具不可用），不能用作"我不确定"的退路。

```ts
// verificationAgent.ts:140-152
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
    agentType: 'verification',
    whenToUse: VERIFICATION_WHEN_TO_USE,
    color: 'red',
    background: true,
    disallowedTools: [
        AGENT_TOOL_NAME,
        EXIT_PLAN_MODE_TOOL_NAME,
        FILE_EDIT_TOOL_NAME,
        FILE_WRITE_TOOL_NAME,
        NOTEBOOK_EDIT_TOOL_NAME,
    ],
    source: 'built-in',
    baseDir: 'built-in',
    model: 'inherit',
    getSystemPrompt: () => VERIFICATION_SYSTEM_PROMPT,
    criticalSystemReminder_EXPERIMENTAL: 'CRITICAL: This is a VERIFICATION-ONLY task. …'
}
```

- `background: true`：默认后台跑，不阻塞主线程。
- `disallowedTools`：从工具表里移除所有"可写"工具——物理隔离，验证 agent 没机会作弊。
- `color: 'red'`：UI 中用红色徽章显著标识。

### 3.2 评审自动触发：3+ 任务后 nudge

见 §2.2。除了 nudge，**3+ 任务的 close-out 也会强制出 verification**。这条机制把"主线程 + 验证线程"做成工业中的"操作工 + 质检员"两段制。

### 3.3 Hooks：声明式评审关

`src/utils/hooks.ts` 实现完整的 Claude Code Hooks 协议（PreToolUse / PostToolUse / PostToolUseFailure / TaskCreated / TaskCompleted / Stop 等）。其中与评审相关的两条：

- `executeTaskCreatedHooks`（`hooks.ts:3986`）—— TaskCreate 后跑，命中 `blockingError` 就回滚创建（见 18-task-planning.md §4.1）。
- `executeTaskCompletedHooks`（`hooks.ts:4030`）—— TaskUpdate 把任务置为 `completed` 时跑，命中错误就拒绝（`TaskUpdateTool.ts:232-265`）。

这是面向**企业治理**的扩展点：团队可以注册"创建任务时必须包含 ticket 编号"或"完成任务时必须跑过测试"。

### 3.4 Plan Verify：把评审前置到规划阶段

`src/tools/EnterPlanModeTool/` + `ExitPlanModeV2Tool` 构成"先评审方案，再评审实现"的双层验证。配合 UltraReview 命令（`src/commands/review/ultrareviewCommand.tsx`），整个评审管线可以在以下任一阶段被触发：

1. **Plan 阶段**——UltraReview 命令对当前 plan 文件做集中审查。
2. **任务关单**——3+ 任务 nudge 触发 verification agent。
3. **Stop hook**——`Stop` 事件 hook 可以做最终 QA（`hooks.ts:1517-1519`）。

### 3.5 Code Review Skill：评审作为可复用能力

虽然 `code-review` skill 本身是用户目录下的 skill 配置（运行时加载），但其设计哲学——"用 verifier 风格去找问题，不为通过而通过"——与 verification agent **完全同构**。这是评审思想"工种化"的体现：可验证的 reviewer 是一类专门角色，模型可按需调度。

## 4. 分治并行（Divide and Conquer）

### 4.1 AgentTool：最基础的并行单元

`src/tools/AgentTool/AgentTool.tsx:243` 是核心调度接口，输入 schema（`AgentTool.tsx:138-149`）支持：

```ts
{
  description, prompt, subagent_type,
  model: 'sonnet' | 'opus' | 'haiku' | 'inherit',
  run_in_background,        // false=同步 / true=异步
  name, team_name, mode,    // multi-agent
  isolation: 'worktree',    // 物理隔离
  cwd                       // 改 cwd
}
```

**5 种派工模式**：

| 模式 | 参数 | 行为 | 适用场景 |
| --- | --- | --- | --- |
| 同步 subagent | `run_in_background: false` | 主线程阻塞等结果 | 短任务，需要返回内容继续主流程 |
| 异步 subagent | `run_in_background: true` | 注册到 `AppState.tasks` 即可返回 | 长任务，结果异步到达 |
| 队友（in-process） | `team_name + name` | 进程内常驻 worker，可 SendMessage 续聊 | 复杂多步任务，共享上下文 |
| 队友（tmux/iTerm2） | `team_name + name` + `teammateMode: tmux` | 独立终端窗口的 worker | 用户可视化监工 |
| Remote / Worktree | `isolation: 'remote' \| 'worktree'` | 远端或独立 worktree 中跑 | 重活隔离，不污染主目录 |

派工入口（`AgentTool.tsx:331`）：

```ts
if (teamName && name) {
    // 派 teammate
    const result = await spawnTeammate({...}, toolUseContext)
    return { data: { status: 'teammate_spawned', ...result.data } }
}
// 否则走普通 subagent
```

### 4.2 Coordinator 模式：把"分治并行"制度化

`src/coordinator/coordinatorMode.ts:36` 的 `isCoordinatorMode()` 检测 `CLAUDE_CODE_COORDINATOR_MODE=1`。开启后，主线程的 system prompt 切换到协调者版本（`getCoordinatorSystemPrompt()`，`coordinatorMode.ts:111`），核心指令：

> "You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers."

四阶段工作流（`coordinatorMode.ts:198-209`）：

```
| Phase         | Who               | Purpose                              |
|---------------|-------------------|--------------------------------------|
| Research      | Workers (parallel)| Investigate codebase, find files     |
| Synthesis     | Coordinator (you) | Read findings, craft implementation  |
| Implementation| Workers           | Make targeted changes per spec       |
| Verification  | Workers           | Test changes work                    |
```

并发原则（`coordinatorMode.ts:213`）：

> "Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible — don't serialize work that can run simultaneously and look for opportunities to fan out. **To launch workers in parallel, make multiple tool calls in a single message.**"

Worker 工具集过滤（`coordinatorMode.ts:29-34`）：worker 不能创建 team、不能发消息、不能发 synthetic output。Coordinator 独占这些"管理类"工具。

### 4.3 Team：把"分工"持久化

`src/tools/TeamCreateTool/TeamCreateTool.ts` + `src/tools/TeamDeleteTool/TeamDeleteTool.ts`：

- `TeamCreate`：在 `~/.claude/teams/<teamName>/` 落盘 team config（lead + members + plan），把 `teamContext` 注入 AppState。
- `TeamDelete`（`TeamDeleteTool.ts:71`）：清理 team 目录 + worktree + leader 标识 + 任务命名空间（`clearLeaderTeamName`）。
- 任务命名空间绑定：`setLeaderTeamName`（`src/utils/tasks.ts:33`）让 `getTaskListId` 切到 teamName，所有 worker 共享同一份任务文件（详见 18-task-planning.md §3.1）。

### 4.4 Mailbox：队友之间不直连

`src/utils/mailbox.ts` 实现进程内的 mailbox 抽象（`send` / `poll` / `receive` / `subscribe`），加上 `src/utils/teammateMailbox.ts` 把 mailbox 持久化到 `~/.claude/teams/<name>/inboxes/<agent>.json`。这样：

- 队友之间不通过共享内存通信，**所有通信走 mailbox**。
- 跨进程（tmux teammate）通过 fs.watch 同步。
- `SendMessage` 工具（`src/tools/SendMessageTool/SendMessageTool.ts:73`）封装 mailbox write；`receive` 拿到后注入主对话。

### 4.5 Ultraplan：把"分治"推到异地

见 §1.3。除了"分离式规划"，Ultraplan 还提供"远端执行"模式（CCR 端跑完直接 PR）。这等价于"**把整条流水线外包给另一个工厂**"——本地是调度席，远端是生产线。

### 4.6 BuiltIn Agents：可调度的角色库

`src/tools/AgentTool/built-in/builtInAgents.ts:39-67` 注册的官方角色：

```ts
const agents: AgentDefinition[] = [
    GENERAL_PURPOSE_AGENT,     // 通用
    STATUSLINE_SETUP_AGENT,
]
if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT)
}
if (isNonSdkEntrypoint) {
    agents.push(CLAUDE_CODE_GUIDE_AGENT)
}
if (feature('VERIFICATION_AGENT') && getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)) {
    agents.push(VERIFICATION_AGENT)
}
```

四个固定角色（`general-purpose` / `statusline-setup` / `Explore` / `Plan`）+ 一个 `claude-code-guide` + 一个 `verification`，加 `CLAUDE_CODE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` 可关闭内置。

## 5. 串接：Query 主循环如何驱动四支柱

四个支柱不是独立存在——它们由 `src/query.ts` / `QueryEngine.ts` 编织成一条主循环。

### 5.1 主循环的伪代码骨架

```
while (not done):
  user_input = await read_user()
  if user_input matches /ultraplan: launchUltraplan()   // §1.3
  elif command: handle_command()
  else:
    state = QueryEngine.run(user_input)   // 主循环
```

`QueryEngine.run` 的关键节点（基于已有源码线索）：

1. **载入 system prompt**——根据 `isCoordinatorMode()` / `isPlanModeRequired()` 切换对应 prompt（`coordinatorMode.ts:111`）。
2. **载入工具表**——根据 feature gate 决定显隐（`tools.ts:220` 等）。
3. **进入 LLM turn**——发出 system + tools + 消息历史。
4. **解析响应**——文本块直接流给 UI；tool_use 块入队。
5. **执行 tool_use 块**（StreamingToolExecutor）——并发执行并发声明，串行执行带依赖。
6. **特殊处理**：
   - `EnterPlanMode` → 切权限模式 + 弹模态（`EnterPlanModeTool.ts:77`）。
   - `ExitPlanMode` → 弹审批 UI（`ExitPlanModeV2Tool.ts:192`）。
   - `Agent` → 派工（同步阻塞 / 异步入队 / 派 teammate）。
   - `TaskCreate` / `TaskUpdate` → 走 §1.2 / §2 数据层。
7. **PostToolUse 钩子链**——`runPreToolUseHooks` + `runPostToolUseHooks`（`hooks.ts:1766`）。
8. **assemble tool_result → 下一轮**。

### 5.2 工具可见性的 feature gate

`src/tools.ts:220` 的关键是**多个 feature gate 控制一组工具的可见性**：

```ts
...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
...(feature('VERIFICATION_AGENT') && getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) ? [VERIFICATION_AGENT] : []),
```

模型在某次 session 中看到的工具集，是这些 gate 的交集。这避免了"模型请求 SDK 不存在的团队工具"或"非交互模式看到了交互模式专属的 UI 工具"。

### 5.3 Coordinator 模式的特殊循环

Coordinator 模式下，主循环变成"**事件驱动**"——worker 完成事件作为 `<task-notification>` 用户消息注入（`coordinatorMode.ts:144-160`）。协调者收到通知后只做两件事：决定派下一个 worker，还是合成结果给用户。**没有"主线程等 worker"——一切都是 mailbox 事件**。

## 6. 完整实例

假设用户请求：

> "给 src/auth/ 加一个会话来防止 CSRF，框架自选。"

**典型协作流**：

1. **规划（先规划）**
   - 模型判定复杂任务，调用 `EnterPlanMode` → 权限切到 plan。
   - 在 plan mode 中做只读探索（Read / Grep / Glob）。
   - 调用 `AskUserQuestion` 问"框架选 express 还是 koa"。
   - 用户答"express" → 调 `ExitPlanMode` 提交 plan + `allowedPrompts: [{tool: 'Bash', prompt: 'run tests'}, {tool: 'Bash', prompt: 'install dependencies'}]`。
   - 用户在 UI 审批通过。

2. **拆分工序（后执行 §1.2）**
   - 模型调 `TaskCreate` ×4：
     - `#1 Add csrf middleware skeleton`
     - `#2 Integrate middleware into Express app`
     - `#3 Write unit tests for token generation/verification`
     - `#4 Update API docs`
   - 设置 `#2` `blockedBy: ['#1']`，`#3` `blockedBy: ['#2']`，`#4` `blockedBy: ['#1', '#2', '#3']`。

3. **执行（后执行 §2）**
   - 调 `TaskUpdate({taskId: '1', status: 'in_progress'})` → 写代码 → `status: 'completed'`。
   - 重复到 `#4`。

4. **强评审关触发（强评审 §2.2 / §3.1）**
   - 关 #4 时，4 个任务全 closed 且无 verification → nudge 触发。
   - 调 `Agent({subagent_type: 'verification', prompt: 'Verify CSRF middleware. Run integration tests with curl + supertest...'})`。
   - Verification agent 给出 `VERDICT: FAIL`，原因是 session cookie 没用 `httpOnly` 标记。
   - 主线程回 `TaskUpdate({taskId: '2', status: 'in_progress'})` 重做 #2。

5. **分治并行（分治并行 §4）**
   - 复杂依赖时，主线程也可以在第一轮就并行派 Explore + Plan subagent：
     - `Agent({subagent_type: 'Explore', prompt: 'Find existing CSRF usage in monorepo'})`
     - `Agent({subagent_type: 'Plan', prompt: 'Draft implementation plan based on findings'})`
   - 两者同时跑，结果都到 mailbox，主线程用 `SendMessage` 续聊。

6. **结束**
   - Verification agent 给 `VERDICT: PASS`。
   - 主线程汇总，输出 `## Approved Plan` + 改 diff 给用户。

## 7. 设计哲学总结

| 决策 | 工业流水线对应 | 源码依据 |
| --- | --- | --- |
| **硬门 + 软门**双层规划 | 工单签字 + 工序看板 | `EnterPlanModeTool` 强切权限 + `TaskCreate` 软规划 |
| **Plan 必须列权限需求** | 工单带工具领用清单 | `ExitPlanModeV2` 的 `allowedPrompts` |
| **任务自报工** | 工序看板自动勾 | `TaskUpdate` 即写即刷新（`useTasksV2` 5s 折叠） |
| **3+ 任务后强 review** | 装配线完成后过质检 | `tengu_hive_evidence` nudge（`TaskUpdateTool.ts:333-349`） |
| **verification agent 物理隔离** | 质检员不参与制造 | `disallowedTools` 移除可写工具（`verificationAgent.ts:143-148`） |
| **verification 必须带证据** | 质检报告含实测数据 | `Command run` + `Output observed` 块（`verificationAgent.ts:82-92`） |
| **VERDICT: PASS/FAIL/PARTIAL 三态** | 质检有标准结论 | `verificationAgent.ts:118-128` |
| **Hooks 链** | 工厂工位可挂传感器 | `executeTaskCreated/CompletedHooks`（`hooks.ts:3986, 4030`） |
| **AgentTool 五种派工** | 一个调度席派五种工 | `run_in_background` / `name` / `team_name` / `isolation` / `cwd`（`AgentTool.tsx:138-149`） |
| **Coordinator mode** | 调度席 vs 生产线分岗 | `CLAUDE_CODE_COORDINATOR_MODE=1` + `getCoordinatorSystemPrompt`（`coordinatorMode.ts:36, 111`） |
| **mailbox 而非共享内存** | 车间靠看板不靠喊话 | `mailbox.ts` + `teammateMailbox.ts` + `SendMessageTool` |
| **Ultraplan 异地规划/执行** | 异地分厂/外包 | `launchUltraplan` + `pollForApprovedExitPlanMode`（`ultraplan.tsx:234`） |
| **每个 subagent 自己的 AsyncLocalStorage 上下文** | 工人带工牌不串号 | `AsyncLocalStorage<AgentContext>`（`agentContext.ts:93`） |
| **可验证的 reviewer 是一类专门角色** | 质检员是独立工种 | `VERIFICATION_AGENT`（`builtInAgents.ts:54-57`） |
| **nudge 写成工具返回字符串而不是 system reminder** | 派工单上写明下一步 | `verificationNudgeNeeded` 字段（`TaskUpdateTool.ts:81`） |
| **任务 list 文件锁 + high water mark** | 工位先到先得，编号不复用 | `tasks.ts:102, 271` |
| **plan mode agent 内部禁用** | 工人不能自起新单 | `EnterPlanModeTool.ts:78-80` 显式抛错 |

## 8. 一句话总结

> Claude Code 的"工业化 agent 治理"=**用 Plan Mode 强制规划、用 TaskList 把方案拆成有依赖的工序、用 TaskUpdate 推动工序状态、用 StreamingToolExecutor 串行执行、用 Verification Agent + Hooks + 3+ 任务 nudge 强制多道质检、用 AgentTool + Coordinator + Team + Mailbox + Ultraplan 实现"本地同步/异步/异端"三种并行**——四根支柱由 Query 主循环编织成一条"规划→拆解→执行→评审→分治并行"的数字装配线，目标是让 LLM 在工程意义上**可治理、可重放、可审计**。
