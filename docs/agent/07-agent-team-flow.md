# Agent Team 核心运行流程

## 概述

Agent Team（团队/蜂群）是 Claude Code 的多智能体协作系统，允许一个**领导（Leader）**创建团队并**生成（Spawn）**多个**队友（Teammate）**，通过消息传递和任务列表进行协作。

**核心概念：**

| 角色 | 描述 |
|------|------|
| **Leader（领导）** | 创建和管理团队的 Claude Code 实例，持有 `TeamCreateTool` 和 `TeamDeleteTool` |
| **Teammate（队友）** | 被生成的 Claude Code 子进程，接收任务并返回结果 |
| **Team File** | 存储在 `~/.claude/teams/<name>/team.json` 的团队元数据文件 |
| **Mailbox（邮箱）** | 文件系统级消息队列，用于队友间异步通信 |
| **Task List（任务列表）** | 团队共享的任务管理系统，基于文件系统实现 |

**注意事项：** 此文档基于逆向工程代码，部分功能（如 `isAgentSwarmsEnabled()` 特征门控）可能在生产环境中被禁用。

---

## 一、架构总览

### 1.1 团队数据结构

团队信息存储在 `TeamFile` 中（`src/utils/swarm/teamHelpers.ts:64`）：

```typescript
type TeamFile = {
  name: string             // 团队名称
  description?: string     // 团队描述
  createdAt: number        // 创建时间戳
  leadAgentId: string      // 领导的 agent ID（格式：team-lead@teamName）
  leadSessionId?: string   // 领导的 session UUID
  hiddenPaneIds?: string[] // 已隐藏的 tmux pane ID
  teamAllowedPaths?: TeamAllowedPath[]  // 允许编辑的路径
  members: Array<{
    agentId: string        // agent ID（格式：agentName@teamName）
    name: string           // 队友名称
    agentType?: string     // 代理类型
    model?: string         // 模型名称
    prompt?: string        // 系统提示词
    color?: string         // 显示颜色
    planModeRequired?: boolean  // 是否需要计划模式
    joinedAt: number       // 加入时间
    tmuxPaneId: string     // tmux pane ID
    cwd: string            // 工作目录
    worktreePath?: string  // git worktree 路径
    sessionId?: string     // 子进程 session ID
    subscriptions: string[]  // 订阅列表
    backendType?: BackendType  // 后端类型（tmux / in-process / iterm2）
    isActive?: boolean     // 是否活跃
    mode?: PermissionMode  // 权限模式
  }>
}
```

### 1.2 团队文件存储路径

```
~/.claude/teams/<team-name>/team.json
```

由 `getTeamFilePath()` 和 `getTeamDir()` 函数管理（`src/utils/swarm/teamHelpers.ts`）。

### 1.3 后端类型

队友可以在三种后端模式下运行（`src/utils/swarm/backends/types.ts`）：

| 后端 | 描述 | 实现 |
|------|------|------|
| `tmux` | 在 tmux 窗格中运行 | `TmuxBackend.ts` |
| `iterm2` | 在 iTerm2 原生窗格中运行 | `ITermBackend.ts` |
| `in-process` | 在同一进程内运行 | `InProcessBackend.ts`，使用 `AsyncLocalStorage` 隔离 |

### 1.4 通信机制

队友之间通过**文件系统邮箱**（Mailbox）进行异步通信（`src/utils/teammateMailbox.ts`）：

```
~/.claude/teams/<team-name>/mailbox/<agent-name>/
  └── <message-id>.json
```

每条消息包含：
```typescript
{
  from: 'team-lead' | '<agent-name>',
  text: string,
  timestamp: string,  // ISO 8601
  type?: 'dm' | 'shutdown_request' | 'shutdown_response'
}
```

---

## 二、团队创建流程

### 2.1 TeamCreateTool

由模型调用 `TeamCreateTool` 创建团队，定义在 `src/tools/TeamCreateTool/TeamCreateTool.ts:74`。

**输入参数：**
```typescript
{
  team_name: string        // 团队名称
  description?: string     // 描述
  agent_type?: string      // 领导类型
}
```

**执行流程：**

```
TeamCreateTool.call({team_name, description, agent_type})
  → 检查是否已有团队: appState.teamContext?.teamName
    → 已有团队 → 抛出错误 "A leader can only manage one team at a time"
  → generateUniqueTeamName(team_name)  // 如果名称重复则生成新 slug
  → formatAgentId('team-lead', teamName)  // 生成领导 agent ID
  → 创建 TeamFile 对象，包含领导成员
  → writeTeamFileAsync(teamName, teamFile)  // 写入磁盘
  → registerTeamForSessionCleanup(teamName)  // 注册会话清理
  → resetTaskList(taskListId)  // 重置任务列表
  → ensureTasksDir(taskListId)  // 创建任务目录
  → setLeaderTeamName(taskListId)  // 设置领导团队名称
  → setAppState 更新 teamContext
    teamContext: {
      teamName,
      teamFilePath,
      leadAgentId,
      teammates: { [leadAgentId]: { name, color, cwd, ... } }
    }
  → 返回 { team_name, team_file_path, lead_agent_id }
```

### 2.2 团队上下文

团队创建后，`AppState.teamContext` 被设置，后续的所有生成操作都基于此上下文：

```typescript
teamContext: {
  teamName: string
  teamFilePath: string
  leadAgentId: string
  teammates: {
    [agentId: string]: {
      name: string
      agentType?: string
      color: string
      tmuxSessionName: string
      tmuxPaneId: string
      cwd: string
      spawnedAt: number
    }
  }
}
```

---

## 三、队友生成流程

### 3.1 入口：spawnTeammate()

队友生成的核心函数在 `src/tools/shared/spawnMultiAgent.ts:1088`：

```typescript
export async function spawnTeammate(config, context) {
  return handleSpawn(config, context)
}
```

### 3.2 Spawn 决策流程

```
handleSpawn(input, context)
  → isInProcessEnabled()?
    → YES → handleSpawnInProcess()    // 同一进程内生成
    → NO  → detectAndGetBackend()     // 检测可用后端
      → 后端可用?
        → YES → useSplitPane?
          → YES → handleSpawnSplitPane()      // 分屏模式（默认）
          → NO  → handleSpawnSeparateWindow() // 独立窗口模式（遗留）
      → 后端不可用（自动模式）?
        → markInProcessFallback()
        → handleSpawnInProcess()    // 降级到进程内模式
```

### 3.3 分屏模式（handleSpawnSplitPane）

`src/tools/shared/spawnMultiAgent.ts:305`

```
handleSpawnSplitPane(input, context)
  → resolveTeammateModel(model, leaderModel)  // 解析模型
  → generateUniqueTeammateName(name, teamName) // 生成唯一名称
  → sanitizeAgentName(uniqueName)              // 清理名称
  → formatAgentId(name, teamName)              // 生成 agent ID
  → detectAndGetBackend()                      // 检测后端（tmux / iTerm2）
  → 如果是 iTerm2 且未设置 → 显示 It2SetupPrompt 交互提示
  → assignTeammateColor(teammateId)            // 分配颜色
  → createTeammatePaneInSwarmView(name, color) // 创建 tmux 窗格
  → enablePaneBorderStatus()                   // 启用边框状态（第一个队友）
  → 构建启动命令:
    cd <cwd> && env <env> <binaryPath> \
      --agent-id <id> --agent-name <name> --team-name <team> \
      --agent-color <color> --parent-session-id <sessionId> \
      [--plan-mode-required] [--model <model>] [flags...]
  → sendCommandToPane(paneId, command)          // 发送命令到窗格
  → setAppState 更新 teamContext.teammates       // 注册队友
  → registerOutOfProcessTeammateTask()           // 注册后台任务
  → 写入 TeamFile.members                        // 持久化成员
  → writeToMailbox(name, { from: 'team-lead', text: prompt })  // 发送初始指令
  → 返回 { teammate_id, agent_id, name, color, ... }
```

### 3.4 独立窗口模式（handleSpawnSeparateWindow）

`src/tools/shared/spawnMultiAgent.ts:545`

与分屏模式类似，但：
- 使用 `tmux new-window` 在独立窗口中创建
- 不使用分屏布局
- 适用于传统 tmux 工作流

### 3.5 进程内模式（handleSpawnInProcess）

`src/tools/shared/spawnMultiAgent.ts:840`

```
handleSpawnInProcess(input, context)
  → resolveTeammateModel(model, leaderModel)
  → generateUniqueTeammateName(name, teamName)
  → sanitizeAgentName(uniqueName)
  → formatAgentId(name, teamName)
  → assignTeammateColor(teammateId)
  → 查找 CustomAgentDefinition（如果指定了 agent_type）
  → spawnInProcessTeammate(config, context)    // 创建进程内上下文
    → 创建 TeammateContext（AsyncLocalStorage）
    → 注册 InProcessTeammateTask
    → 创建 AbortController
  → startInProcessTeammate({...})              // 启动执行循环（fire-and-forget）
    → 在后台异步执行 runInProcessTeammate()
  → 更新 AppState.teamContext
  → 写入 TeamFile.members
  → 返回 SpawnOutput
```

**关键区别：** 进程内队友**不**通过邮箱发送初始提示词，而是通过 `startInProcessTeammate()` 直接传递。

---

## 四、进程内队友运行循环

### 4.1 入口：startInProcessTeammate()

`src/utils/swarm/inProcessRunner.ts:1544`

```typescript
export function startInProcessTeammate(config: InProcessRunnerConfig): void {
  // 在后台异步运行
  runInProcessTeammate(config).catch(error => { ... })
}
```

### 4.2 主循环：runInProcessTeammate()

`src/utils/swarm/inProcessRunner.ts:883`

```
runInProcessTeammate(config)
  → 创建 AgentContext（用于分析归因）
  → 构建系统提示词:
    getSystemPrompt() + TEAMMATE_SYSTEM_PROMPT_ADDENDUM
    + 自定义代理指令（如果有）
  → 解析 agent 定义（注入团队必需工具）
  → 添加初始用户提示词
  → 主循环:
    while (!abortController.signal.aborted && !shouldExit) {
      ↓
      1. 标记任务状态为 running
      ↓
      2. runAgent() 执行代理循环
         → query() 发送 API 请求
         → 处理 tool_use 响应
         → 处理 SendMessage、TaskCreate/Get/Update/List 等工具
      ↓
      3. 处理完成/中断:
         → 如果是中断（Escape）→ 记录中断消息
         → 标记任务为 idle
         → 发送 idle 通知给领导
      ↓
      4. waitForNextPromptOrShutdown()
         → 轮询邮箱（每 500ms 一次）
         → 检查 shutdown_request
         → 检查新任务提示
      ↓
      5. 根据邮箱消息类型处理:
         case 'shutdown_request':
           → 将关闭请求传递给模型处理
           → 模型通过 SendMessageTool 响应
         case 'dm' (新任务):
           → 添加到对话历史
           → 继续循环（回到 runAgent）
         case 'abort':
           → 退出循环
    }
  → 清理: evictTaskOutput, evictTerminalTask, unregisterTask
```

### 4.3 等待循环：waitForNextPromptOrShutdown()

`src/utils/swarm/inProcessRunner.ts:689`

```
waitForNextPromptOrShutdown(identity, abortController, ...)
  → 循环轮询，直到 abort 或收到消息:
    → 读取邮箱所有未读消息
    → 优先处理 shutdown_request（最高优先级）
    → 处理 team-lead 消息（优先于队友消息）
    → 返回 { type: 'shutdown_request', request } 或
           { type: 'dm', text, from } 或
           { type: 'abort' }
```

### 4.4 空闲通知

队友完成工作后，通过邮箱发送空闲通知给领导：

```
sendIdleNotification(identity, reason)
  → writeToMailbox(identity.agentName, {
      from: identity.agentName,
      type: 'dm',
      text: idleNotificationMessage
    }, identity.teamName)
  → 更新 TeamFile: setMemberActive(teamName, agentName, false)
```

---

## 五、通信流程

### 5.1 SendMessageTool

`src/tools/SendMessageTool/SendMessageTool.ts` 定义了队友间的通信协议。

**支持的消息类型：**

| 类型 | 描述 | 方向 |
|------|------|------|
| `dm` | 普通消息 | 任意方向 |
| `broadcast` | 广播给所有队友 | 领导 → 所有人 |
| `shutdown_request` | 请求关闭 | 领导 → 队友 |
| `shutdown_response` | 响应关闭请求（approve/reject） | 队友 → 领导 |
| `plan_approval_request` | 请求计划审批 | 队友 → 领导 |
| `plan_approval_response` | 计划审批响应 | 领导 → 队友 |

### 5.2 消息发送

```
handleMessage({to, message, ...})
  → writeToMailbox(to, { from: senderName, text: message, timestamp })
  → 返回 { success: true, message: "Message sent to <name>" }
```

### 5.3 广播

```
handleBroadcast({message, ...})
  → 读取 TeamFile 获取所有成员
  → 过滤掉自己
  → 对每个成员: writeToMailbox(name, { from, text, timestamp })
  → 返回 { message: "Message broadcast to N teammate(s)" }
```

### 5.4 关闭流程

```
// 领导请求关闭队友
handleShutdownRequest({to, reason})
  → 生成 request_id
  → 创建 shutdownRequestMessage
  → writeToMailbox(to, { from, text: jsonStringify(message) })
  → 返回 { success: true, message: "Shutdown request sent" }

// 队友批准关闭
handleShutdownApproval({request_id, message, ...})
  → 创建 shutdownResponseMessage
  → writeToMailbox(TEAM_LEAD_NAME, { from, text: jsonStringify(message) })
  → 如果是进程内队友 → 触发 AbortController.abort()

// 队友拒绝关闭
handleShutdownRejection({request_id, reason, ...})
  → 创建 shutdownResponseMessage（approve: false）
  → writeToMailbox(TEAM_LEAD_NAME, { from, text: jsonStringify(message) })
```

### 5.5 计划审批流程

```
// 队友请求审核
handlePlanApproval({to, request_id, plan})
  → writeToMailbox(to, { from, text: jsonStringify(planApprovalRequest) })

// 领导批准/拒绝
handlePlanRejection / handlePlanApproval
  → writeToMailbox(agentName, { from, text: jsonStringify(response) })
```

---

## 六、团队关闭流程

### 6.1 TeamDeleteTool

由模型调用 `TeamDeleteTool` 销毁团队，定义在 `src/tools/TeamDeleteTool/TeamDeleteTool.ts:32`。

**执行流程：**

```
TeamDeleteTool.call({}, context)
  → 获取 teamName: appState.teamContext?.teamName
  → 读取 TeamFile
  → 检查活跃成员（非领导、isActive !== false）
    → 有活跃成员 → 返回错误:
      "Cannot cleanup team with N active member(s): use requestShutdown first"
  → cleanupTeamDirectories(teamName)      // 清理目录和 worktree
  → unregisterTeamForSessionCleanup(teamName)  // 取消会话清理注册
  → clearTeammateColors()                 // 清除颜色分配
  → clearLeaderTeamName()                 // 清除领导团队名称
  → setAppState 清除 teamContext 和 inbox
  → 返回 { success: true, message: "Cleaned up directories..." }
```

### 6.2 目录清理

`cleanupTeamDirectories()`（`src/utils/swarm/teamHelpers.ts:641`）：

```
cleanupTeamDirectories(teamName)
  → 读取 TeamFile 获取 worktree 路径列表
  → 删除团队目录: ~/.claude/teams/<team-name>/
  → 删除任务目录: ~/.claude/tasks/<team-name>/
  → 对每个 worktree 路径:
    → destroyWorktree(path)
      → 读取 .git 文件获取主仓库路径
      → git worktree remove --force <path>
      → 失败时: rm -rf <path>
```

### 6.3 会话级清理

当领导进程异常退出（SIGINT/SIGTERM）时，`cleanupSessionTeams()` 负责清理：

```
cleanupSessionTeams()  // 在 gracefulShutdown 中注册
  → 获取所有会话创建的团队
  → killOrphanedTeammatePanes(teamName)  // 先杀死队友窗格
    → 读取 TeamFile
    → 对每个 pane 后端成员:
      → getBackendByType(type).killPane(paneId, useExternalSession)
  → cleanupTeamDirectories(teamName)  // 清理目录
  → 清空会话团队列表
```

---

## 七、完整生命周期时序

### 创建阶段

```
模型 → TeamCreateTool.call({team_name: "my-team"})
  → 写入 team.json
  → 设置 AppState.teamContext
  → 返回 { team_name, team_file_path, lead_agent_id }
```

### 生成阶段

```
模型 → AgentTool.call({name: "researcher", team_name: "my-team", prompt: "调研...", run_in_background: true})
  → spawnTeammate({name: "researcher", prompt: "调研...", team_name: "my-team"}, context)
  → handleSpawn()
    → tmux 后端:
      → 创建 tmux 窗格
      → 发送启动命令（claude --agent-id ...）
      → 写入 team.json
      → 通过邮箱发送初始提示词
    → in-process 后端:
      → spawnInProcessTeammate()
      → startInProcessTeammate()
      → 直接传递提示词
      → runInProcessTeammate() 后台运行
  → 返回 { teammate_id, name, color, ... }
```

### 运行阶段

```
进程内队友:
  runInProcessTeammate()
    → 主循环:
      → runAgent() 处理任务
      → 完成 → 发送 idle 通知
      → 等待新消息（waitForNextPromptOrShutdown）
      → 收到新任务 → 继续 runAgent()
      → 收到 shutdown_request → 传递给模型决策

tmux 队友:
  claude --agent-id researcher@my-team --team-name my-team ...
    → 启动后读取邮箱
    → 处理初始提示词
    → 通过 SendMessageTool 与领导通信
    → 完成工作后发送 idle 通知
```

### 销毁阶段

```
模型 → SendMessageTool.call({to: "researcher", message: {type: "shutdown_request"}})
  → 写入关闭请求到邮箱

队友进程处理 shutdown_request:
  → 模型决定批准
  → SendMessageTool.call({to: "team-lead", message: {type: "shutdown_response", approve: true}})
  → 队友进程退出

模型 → TeamDeleteTool.call({})
  → 检查无活跃成员
  → cleanupTeamDirectories()
  → 清除 AppState.teamContext
  → 返回 { success: true }
```

### 异常退出清理

```
SIGINT/SIGTERM（领导进程）
  → gracefulShutdown
  → cleanupSessionTeams()
    → killOrphanedTeammatePanes(teamName)  // 杀死所有队友窗格
    → cleanupTeamDirectories(teamName)      // 清理目录
```

---

## 八、关键函数索引

| 文件 | 函数 | 作用 |
|------|------|------|
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | `TeamCreateTool.call()` | 创建团队并写入 team.json |
| `src/tools/TeamDeleteTool/TeamDeleteTool.ts` | `TeamDeleteTool.call()` | 销毁团队、清理目录和 worktree |
| `src/tools/shared/spawnMultiAgent.ts` | `spawnTeammate()` | 生成队友的入口函数 |
| `src/tools/shared/spawnMultiAgent.ts` | `handleSpawnSplitPane()` | tmux 分屏模式生成 |
| `src/tools/shared/spawnMultiAgent.ts` | `handleSpawnSeparateWindow()` | tmux 独立窗口模式生成 |
| `src/tools/shared/spawnMultiAgent.ts` | `handleSpawnInProcess()` | 进程内模式生成 |
| `src/tools/shared/spawnMultiAgent.ts` | `registerOutOfProcessTeammateTask()` | 注册外部进程队友后台任务 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `handleMessage()` | 发送私信 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `handleBroadcast()` | 广播消息 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `handleShutdownRequest()` | 发送关闭请求 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `handleShutdownApproval()` | 批准关闭 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `handleShutdownRejection()` | 拒绝关闭 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | `handlePlanApproval()` | 计划审批请求 |
| `src/utils/swarm/inProcessRunner.ts` | `startInProcessTeammate()` | 启动进程内队友（fire-and-forget） |
| `src/utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()` | 进程内队友主运行循环 |
| `src/utils/swarm/inProcessRunner.ts` | `waitForNextPromptOrShutdown()` | 等待新消息或关闭指令 |
| `src/utils/swarm/inProcessRunner.ts` | `sendIdleNotification()` | 发送空闲通知 |
| `src/utils/swarm/teamHelpers.ts` | `readTeamFile()` / `writeTeamFileAsync()` | 读写团队文件 |
| `src/utils/swarm/teamHelpers.ts` | `cleanupTeamDirectories()` | 清理团队目录和 worktree |
| `src/utils/swarm/teamHelpers.ts` | `registerTeamForSessionCleanup()` | 注册会话清理 |
| `src/utils/swarm/teamHelpers.ts` | `cleanupSessionTeams()` | 清理所有会话创建的团队 |
| `src/utils/swarm/teamHelpers.ts` | `setMemberActive()` | 设置成员活跃状态 |
| `src/utils/teamDiscovery.ts` | `getTeammateStatuses()` | 获取队友状态列表 |
| `src/utils/teammateMailbox.ts` | `writeToMailbox()` / `readMailbox()` | 邮箱读写 |
| `src/utils/agentSwarmsEnabled.ts` | `isAgentSwarmsEnabled()` | 检查特征门控 |