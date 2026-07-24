# Task 工具族 —— 后台任务与任务清单管理

> 源码位置：
> - `src/tools/TaskOutputTool/TaskOutputTool.tsx`（591 行）
> - `src/tools/TaskListTool/TaskListTool.ts`（117 行）
> - `src/tools/TaskGetTool/TaskGetTool.ts`（129 行）
> - `src/tools/TaskCreateTool/TaskCreateTool.ts`（139 行）
> - `src/tools/TaskStopTool/TaskStopTool.ts`（132 行）
> - `src/tools/TaskUpdateTool/TaskUpdateTool.ts`（407 行）
>
> 关联文档：[`docs/18-task-planning.md`](18-task-planning.md) · [`docs/26-filewrite-and-todowrite-tools.md`](../tool/26-filewrite-and-todowrite-tools.md) · [`docs/23-design-and-core-modules.md`](../architecture/23-design-and-core-modules.md)

这 6 个工具构成了 Claude Code 的**后台任务与任务清单管理子系统**。它们分为两个独立但互补的体系：

1. **任务状态系统**（V1 TodoWrite）：用 `TodoWrite` 维护单次会话的轻量任务清单（详见 [`docs/26-filewrite-and-todowrite-tools.md`](../tool/26-filewrite-and-todowrite-tools.md)）。
2. **后台任务系统**：7 种 `TaskType`（`local_bash` / `local_agent` / `remote_agent` / `in_process_teammate` / `local_workflow` / `monitor_mcp` / `dream`），用 6 个工具管理生命周期。

本文逐一拆解每个工具，重点关注 **Schema 设计、call 内部流程、与 AppState 的协作、Hook 触发点**。

---

## 目录

- [全景图：工具与 TaskType 的映射](#全景图工具与-tasktype-的映射)
- [TaskOutputTool —— 读取后台任务输出](#taskoutputtool--读取后台任务输出)
- [TaskCreateTool —— 创建任务清单条目](#taskcreatetool--创建任务清单条目)
- [TaskUpdateTool —— 更新任务状态/依赖/所有者](#taskupdatetool--更新任务状态依赖所有者)
- [TaskListTool —— 列出当前任务清单](#tasklisttool--列出当前任务清单)
- [TaskGetTool —— 按 ID 查询单个任务](#taskgettool--按-id-查询单个任务)
- [TaskStopTool —— 终止运行中的后台任务](#taskstoptool--终止运行中的后台任务)
- [整体协作时序](#整体协作时序)

---

## 全景图：工具与 TaskType 的映射

```ts
// src/Task.ts:15
type TaskType =
  | 'local_bash'        // 本地 shell
  | 'local_agent'       // 本地子 agent
  | 'remote_agent'      // 远程 agent (CCR 容器)
  | 'in_process_teammate' // 同进程 teammate
  | 'local_workflow'    // workflow 脚本
  | 'monitor_mcp'       // MCP 监视
  | 'dream'             // KAIROS 自动做梦
```

| 工具 | 作用对象 | 是否需要 V2 | 并发安全 |
|------|---------|-------------|---------|
| `TaskOutputTool` | 7 种 TaskType 全部（统一输出接口） | 否（但仅 ant） | ✅ 只读 |
| `TaskCreateTool` | 任务清单条目（V2） | ✅ `isTodoV2Enabled` | ✅ |
| `TaskUpdateTool` | 任务清单条目（V2） | ✅ `isTodoV2Enabled` | ✅ |
| `TaskListTool` | 任务清单查询（V2） | ✅ `isTodoV2Enabled` | ✅ 只读 |
| `TaskGetTool` | 单任务查询（V2） | ✅ `isTodoV2Enabled` | ✅ 只读 |
| `TaskStopTool` | 7 种 TaskType 全部（仅 running 状态） | 否 | ✅ |

**关键设计**：TaskOutput/TaskStop **不依赖 V2 开关**，因为它们处理的是后台任务（任何时候都有），而 TaskCreate/Update/List/Get 是 V2 任务清单系统（与 TodoWrite 互斥）。

---

# TaskOutputTool —— 读取后台任务输出

## 1. 职责

TaskOutputTool 是**唯一**允许模型读取后台任务（`local_bash` / `local_agent` / `remote_agent` / `in_process_teammate` 等）输出的工具。

**关键事实**（`TaskOutputTool.tsx:160-183`）：在 ant 外部 build 中**已弃用**，推荐改用 `Read` 工具读磁盘上的 transcript 文件：

```ts
async description() {
  return '[Deprecated] — prefer Read on the agent output file path'
},
async prompt() {
  return `DEPRECATED: Prefer using the Read tool on the task's output file path instead. ...`
}
isEnabled() {
  return "external" !== 'ant'  // 外部 build 不启用
}
```

**但** `isEnabled` 判断的是 `"external" !== 'ant'` —— 实际是 `true`，所以外部 build **也启用**。这是源码中的注释/逻辑不一致（实际上是 `false`，因为 external === external），外部 build 用户也能用 TaskOutputTool。

`aliases: ['AgentOutputTool', 'BashOutputTool']` —— 向后兼容老 transcript。

## 2. Schema 与并发性

```ts
// TaskOutputTool.tsx:31-36
const inputSchema = lazySchema(() => z.strictObject({
    task_id: z.string().describe('The agent ID to get output from'),
    block: semanticBoolean(z.boolean().default(true)).describe('Whether to wait for completion'),
    timeout: z.number().min(0).max(600000).default(30000).describe('Max wait time in ms')
}));
```

三个字段：
- `task_id`：36 进制 8 位随机 ID；
- `block`：默认 true（等待任务完成），false 用于快速检查；
- `timeout`：最长 600 秒（10 分钟），默认 30 秒。

```ts
// TaskOutputTool.tsx:162-164
isConcurrencySafe(_input) {
  return this.isReadOnly?.(_input) ?? false
},
isReadOnly(_input) {
  return true
}
```

**为什么并发安全？** 因为 `getTaskOutputData` 只读 `appState.tasks` + 磁盘文件，不修改任何状态。

## 3. call 内部流程

### 3.1 入口校验

```ts
// TaskOutputTool.tsx:216-220
const appState = toolUseContext.getAppState()
const task = appState.tasks?.[task_id] as TaskState | undefined
if (!task) {
  throw new Error(`No task found with ID: ${task_id}`)
}
```

任务不存在 → 直接抛错（不静默）—— 让模型知道 task_id 错了。

### 3.2 非阻塞分支（`block=false`）

```ts
// TaskOutputTool.tsx:221-242
if (!block) {
  if (task.status !== 'running' && task.status !== 'pending') {
    // Mark as notified
    updateTaskState(task_id, toolUseContext.setAppState, t => ({
      ...t, notified: true
    }))
    return { data: { retrieval_status: 'success' as const, task: await getTaskOutputData(task) } }
  }
  return { data: { retrieval_status: 'not_ready' as const, task: await getTaskOutputData(task) } }
}
```

**已完成任务**：标记 `notified: true`（防止重复弹通知）→ 返回 success。

**未完成任务**：返回 `not_ready`，不修改任务状态。

### 3.3 阻塞分支（`block=true`）

```ts
// TaskOutputTool.tsx:244-283
// 1. 推送 progress 消息
if (onProgress) {
  onProgress({
    toolUseID: `task-output-waiting-${Date.now()}`,
    data: { type: 'waiting_for_task', taskDescription: task.description, taskType: task.type }
  })
}

// 2. 等待（最多 timeout ms）
const completedTask = await waitForTaskCompletion(task_id, toolUseContext.getAppState, timeout, toolUseContext.abortController)

if (!completedTask) {
  return { data: { retrieval_status: 'timeout' as const, task: null } }
}

if (completedTask.status === 'running' || completedTask.status === 'pending') {
  return { data: { retrieval_status: 'timeout' as const, task: await getTaskOutputData(completedTask) } }
}

// 3. 标记 notified
updateTaskState(task_id, toolUseContext.setAppState, t => ({ ...t, notified: true }))
return { data: { retrieval_status: 'success' as const, task: await getTaskOutputData(completedTask) } }
```

### 3.4 等待循环

```ts
// TaskOutputTool.tsx:118-144
async function waitForTaskCompletion(taskId, getAppState, timeoutMs, abortController) {
  const startTime = Date.now()
  while (Date.now() - startTime < timeoutMs) {
    if (abortController?.signal.aborted) throw new AbortError()
    const state = getAppState()
    const task = state.tasks?.[taskId]
    if (!task) return null
    if (task.status !== 'running' && task.status !== 'pending') return task
    await sleep(100)  // 每 100ms 轮询
  }
  const finalState = getAppState()
  return finalState.tasks?.[taskId] ?? null
}
```

**关键细节**：
- **100ms 轮询**：避免依赖事件通知（任务状态机内部没事件订阅）；
- **abort 抛 `AbortError`**：用户按 Ctrl+C 立即取消；
- **超时返回当前状态**：调用方判断是 timeout 还是 success。

### 3.5 `getTaskOutputData` 类型分发

```ts
// TaskOutputTool.tsx:61-116
async function getTaskOutputData(task: TaskState): Promise<TaskOutput> {
  let output: string
  if (task.type === 'local_bash') {
    const bashTask = task as LocalShellTaskState
    const taskOutputObj = bashTask.shellCommand?.taskOutput
    if (taskOutputObj) {
      const stdout = await taskOutputObj.getStdout()
      const stderr = taskOutputObj.getStderr()
      output = [stdout, stderr].filter(Boolean).join('\n')
    } else {
      output = await getTaskOutput(task.id)
    }
  } else {
    output = await getTaskOutput(task.id)
  }

  const baseOutput: TaskOutput = { task_id, task_type, status, description, output }

  // 类型特定字段
  if (task.type === 'local_bash') return { ...baseOutput, exitCode: bashTask.result?.code ?? null }
  if (task.type === 'local_agent') {
    const cleanResult = agentTask.result ? extractTextContent(agentTask.result.content, '\n') : undefined
    return { ...baseOutput, prompt: agentTask.prompt, result: cleanResult || output, output: cleanResult || output, error: agentTask.error }
  }
  if (task.type === 'remote_agent') return { ...baseOutput, prompt: remoteTask.command }
  return baseOutput
}
```

**关键注释**（`TaskOutputTool.tsx:96-100`）：

> Prefer the clean final answer from the in-memory result over the raw JSONL transcript on disk. The disk output is a symlink to the full session transcript (every message, tool use, etc.), not just the subagent's answer. The in-memory result contains only the final assistant text content blocks.

对 agent 任务，**优先用内存里的 `result.content`**（只含最终回答），而非磁盘上的 JSONL transcript（含全部 tool_use 等噪声）。

## 4. 输出格式与 UI 渲染

### 4.1 给模型的 tool_result

```ts
// TaskOutputTool.tsx:285-310
mapToolResultToToolResultBlockParam(data, toolUseID) {
  const parts: string[] = []
  parts.push(`<retrieval_status>${data.retrieval_status}</retrieval_status>`)
  if (data.task) {
    parts.push(`<task_id>${data.task.task_id}</task_id>`)
    parts.push(`<task_type>${data.task.task_type}</task_type>`)
    parts.push(`<status>${data.task.status}</status>`)
    if (data.task.exitCode !== undefined && data.task.exitCode !== null) {
      parts.push(`<exit_code>${data.task.exitCode}</exit_code>`)
    }
    if (data.task.output?.trim()) {
      const { content } = formatTaskOutput(data.task.output, data.task.task_id)
      parts.push(`<output>\n${content.trimEnd()}\n</output>`)
    }
    if (data.task.error) {
      parts.push(`<error>${data.task.error}</error>`)
    }
  }
  return { tool_use_id: toolUseID, type: 'tool_result', content: parts.join('\n\n') }
}
```

**结构化标签**给模型 —— `<retrieval_status>` / `<task_id>` / `<task_type>` 等，让模型能精确解析。

### 4.2 UI 渲染（TaskOutputResultDisplay）

`TaskOutputResultDisplay`（`TaskOutputTool.tsx:356-588`）按 task_type 分发：

| task_type | UI 组件 |
|-----------|---------|
| `local_bash` | `BashToolResultMessage`（复用 Bash 输出样式） |
| `local_agent` | `AgentPromptDisplay` + `AgentResponseDisplay` + 行数统计 |
| `remote_agent` | 状态 + 折叠输出（`ctrl+o` 展开） |
| 其他 | 通用描述 + 截断 500 字符 |

**特殊细节**：verbose 模式下显示完整结果，非 verbose 显示 `Read output (Ctrl+O to expand)`。

## 5. Progress 渲染

```ts
// TaskOutputTool.tsx:326-339
renderToolUseProgressMessage(progressMessages) {
  const lastProgress = progressMessages[progressMessages.length - 1]
  const progressData = lastProgress?.data as { taskDescription?: string; taskType?: string } | undefined
  return <Box flexDirection="column">
    {progressData?.taskDescription && <Text>&nbsp;&nbsp;{progressData.taskDescription}</Text>}
    <Text>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Waiting for task{' '}
      <Text dimColor>(esc to give additional instructions)</Text>
    </Text>
  </Box>
}
```

UI 显示 `Waiting for task <description> (esc to give additional instructions)`，提示用户可中断。

---

# TaskCreateTool —— 创建任务清单条目

## 1. 职责

TaskCreateTool 是 V2 任务清单系统的**创建**入口（替代 TodoWriteTool 的部分能力）。它把模型给出的 subject/description 写到磁盘文件，形成**持久化任务清单**。

## 2. Schema

```ts
// TaskCreateTool.ts:18-33
const inputSchema = lazySchema(() =>
  z.strictObject({
    subject: z.string().describe('A brief title for the agent'),
    description: z.string().describe('What needs to be done'),
    activeForm: z.string().optional().describe('Present continuous form shown in spinner when in_progress'),
    metadata: z.record(z.string(), z.unknown()).optional().describe('Arbitrary metadata to attach to the agent'),
  }),
)
```

四个字段：
- `subject`：短标题（如 "实现核心游戏逻辑"）；
- `description`：详细描述；
- `activeForm`：进行中时 spinner 显示（如 "实现核心游戏逻辑"）；
- `metadata`：任意键值对（高级用法）。

## 3. V2 开关

```ts
// TaskCreateTool.ts:68-70
isEnabled() {
  return isTodoV2Enabled()
}
```

`isTodoV2Enabled()` 检查 GrowthBook / env 配置。V2 启用时，TodoWriteTool 自动禁用，反之亦然（互斥）。

## 4. call 内部流程

```ts
// TaskCreateTool.ts:80-129
async call({ subject, description, activeForm, metadata }, context) {
  // 1. 写入磁盘
  const taskId = await createTask(getTaskListId(), {
    subject, description, activeForm,
    status: 'pending',      // 默认 pending
    owner: undefined,        // 暂未分配
    blocks: [],              // 默认无依赖
    blockedBy: [],           // 默认无被依赖
    metadata,
  })

  // 2. 触发 TaskCreated 钩子
  const blockingErrors: string[] = []
  const generator = executeTaskCreatedHooks(taskId, subject, description, getAgentName(), getTeamName(), undefined, context?.abortController?.signal, undefined, context)
  for await (const result of generator) {
    if (result.blockingError) {
      blockingErrors.push(getTaskCreatedHookMessage(result.blockingError))
    }
  }

  // 3. 钩子返回阻塞错误 → 删除任务 + 抛错
  if (blockingErrors.length > 0) {
    await deleteTask(getTaskListId(), taskId)
    throw new Error(blockingErrors.join('\n'))
  }

  // 4. 自动展开任务面板
  context.setAppState(prev => {
    if (prev.expandedView === 'tasks') return prev
    return { ...prev, expandedView: 'tasks' as const }
  })

  return { data: { task: { id: taskId, subject } } }
}
```

**关键设计**：

1. **TaskCreated 钩子可阻断**：用户配置的钩子若返回 blocking error，自动 `deleteTask` 撤销（不留垃圾数据），再抛错给模型。
2. **自动展开任务面板**：UI 状态 `expandedView = 'tasks'`，让用户立即看到新任务（除非已经在 task 视图）。

## 5. 输出

```ts
// TaskCreateTool.ts:130-137
mapToolResultToToolResultBlockParam(content, toolUseID) {
  const { task } = content as Output
  return { tool_use_id: toolUseID, type: 'tool_result', content: `Task #${task.id} created successfully: ${task.subject}` }
}
```

极简文本：`Task #1 created successfully: 实现核心游戏逻辑`。

---

# TaskUpdateTool —— 更新任务状态/依赖/所有者

## 1. 职责

TaskUpdateTool 是 V2 任务清单的**核心**工具。模型用它推进任务状态（pending → in_progress → completed）、设置依赖关系（`addBlocks` / `addBlockedBy`）、分配所有者。

## 2. Schema

```ts
// TaskUpdateTool.ts:33-66
const inputSchema = lazySchema(() => {
  const TaskUpdateStatusSchema = TaskStatusSchema().or(z.literal('deleted'))

  return z.strictObject({
    taskId: z.string().describe('The ID of the agent to update'),
    subject: z.string().optional(),
    description: z.string().optional(),
    activeForm: z.string().optional(),
    status: TaskUpdateStatusSchema.optional(),  // 多了 'deleted' 字面量
    addBlocks: z.array(z.string()).optional().describe('Task IDs that this agent blocks'),
    addBlockedBy: z.array(z.string()).optional().describe('Task IDs that block this agent'),
    owner: z.string().optional(),
    metadata: z.record(z.string(), z.unknown()).optional(),
  })
})
```

**注意**：`status` 字段除 `'pending' | 'in_progress' | 'completed'` 外，**还接受 `'deleted'`** —— 这是 TaskUpdateTool 独有的"删除任务"语义。

## 3. call 内部流程

### 3.1 阶段一：基础字段更新

```ts
// TaskUpdateTool.ts:140-211
// Auto-expand agent list when updating tasks
context.setAppState(prev => {
  if (prev.expandedView === 'tasks') return prev
  return { ...prev, expandedView: 'tasks' as const }
})

// Check if agent exists
const existingTask = await getTask(taskListId, taskId)
if (!existingTask) {
  return { data: { success: false, taskId, updatedFields: [], error: 'Task not found' } }
}

const updatedFields: string[] = []
const updates: { ... } = {}

// 比较后写入 updates
if (subject !== undefined && subject !== existingTask.subject) {
  updates.subject = subject
  updatedFields.push('subject')
}
// ... description / activeForm / owner 同理

// Auto-set owner: teammate marks agent in_progress 时自动获取 owner
if (
  isAgentSwarmsEnabled() &&
  status === 'in_progress' &&
  owner === undefined &&
  !existingTask.owner
) {
  const agentName = getAgentName()
  if (agentName) {
    updates.owner = agentName
    updatedFields.push('owner')
  }
}

// metadata 合并（null = 删除）
if (metadata !== undefined) {
  const merged = { ...(existingTask.metadata ?? {}) }
  for (const [key, value] of Object.entries(metadata)) {
    if (value === null) {
      delete merged[key]
    } else {
      merged[key] = value
    }
  }
  updates.metadata = merged
  updatedFields.push('metadata')
}
```

**设计细节**：
- **差异比较**：`if (subject !== existingTask.subject)` —— 不写无变化的字段，避免触发 `updatedFields` 累积；
- **metadata 合并语义**：`null` 表示删除该键，非 null 表示 set/update；
- **auto-set owner**：注释（`TaskUpdateTool.ts:185-187`）：

  > Auto-set owner when a teammate marks a task as in_progress without explicitly providing an owner. This ensures the task list can match todo items to teammates for showing activity status.

### 3.2 阶段二：状态更新（删除或 completed 触发钩子）

```ts
// TaskUpdateTool.ts:212-270
if (status !== undefined) {
  // Handle deletion
  if (status === 'deleted') {
    const deleted = await deleteTask(taskListId, taskId)
    return { data: { success: deleted, taskId, updatedFields: deleted ? ['deleted'] : [], error: deleted ? undefined : 'Failed to delete agent', statusChange: deleted ? {from: existingTask.status, to: 'deleted'} : undefined } }
  }

  // For regular status updates, validate and apply if different
  if (status !== existingTask.status) {
    // Run TaskCompleted hooks when marking a agent as completed
    if (status === 'completed') {
      const blockingErrors: string[] = []
      const generator = executeTaskCompletedHooks(taskId, existingTask.subject, existingTask.description, getAgentName(), getTeamName(), undefined, context?.abortController?.signal, undefined, context)
      for await (const result of generator) {
        if (result.blockingError) {
          blockingErrors.push(getTaskCompletedHookMessage(result.blockingError))
        }
      }
      if (blockingErrors.length > 0) {
        return { data: { success: false, taskId, updatedFields: [], error: blockingErrors.join('\n') } }
      }
    }
    updates.status = status
    updatedFields.push('status')
  }
}
```

**与 TaskCreate 的差异**：
- `status === 'completed'` 触发 `executeTaskCompletedHooks`，但**不删除任务**（completed 是合法状态，删除是另一种动作）；
- `status === 'deleted'` 走 deleteTask 分支；
- 钩子返回 blocking errors → 不更新（返回 `success: false`）但**不删任务**。

### 3.3 阶段三：owner 变更通知 mailbox

```ts
// TaskUpdateTool.ts:276-298
if (updates.owner && isAgentSwarmsEnabled()) {
  const senderName = getAgentName() || 'team-lead'
  const senderColor = getTeammateColor()
  const assignmentMessage = JSON.stringify({
    type: 'task_assignment',
    taskId,
    subject: existingTask.subject,
    description: existingTask.description,
    assignedBy: senderName,
    timestamp: new Date().toISOString(),
  })
  await writeToMailbox(updates.owner, { from: senderName, text: assignmentMessage, timestamp: new Date().toISOString(), color: senderColor }, taskListId)
}
```

当 owner 变更且 swarms 启用时，写 mailbox 给新 owner —— teammate 会收到 `task_assignment` 消息。

### 3.4 阶段四：blocks/blockedBy 关系

```ts
// TaskUpdateTool.ts:300-324
if (addBlocks && addBlocks.length > 0) {
  const newBlocks = addBlocks.filter(id => !existingTask.blocks.includes(id))
  for (const blockId of newBlocks) {
    await blockTask(taskListId, taskId, blockId)
  }
  if (newBlocks.length > 0) updatedFields.push('blocks')
}

if (addBlockedBy && addBlockedBy.length > 0) {
  const newBlockedBy = addBlockedBy.filter(id => !existingTask.blockedBy.includes(id))
  for (const blockerId of newBlockedBy) {
    await blockTask(taskListId, blockerId, taskId)  // ← 注意参数顺序
  }
  if (newBlockedBy.length > 0) updatedFields.push('blockedBy')
}
```

**方向性**：
- `addBlocks: ['X']` 表示 "this task blocks X" → `blockTask(taskListId, taskId, 'X')` 在 task 和 X 之间建边；
- `addBlockedBy: ['Y']` 表示 "this task is blocked by Y" → `blockTask(taskListId, 'Y', taskId)` 反向建边。

### 3.5 阶段五：Verification Nudge

```ts
// TaskUpdateTool.ts:333-349
let verificationNudgeNeeded = false
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

**逻辑**：主线程 + VERIFICATION_AGENT feature + GrowthBook 启用 + 刚刚把某任务设为 completed + 全部任务都 completed 且 ≥3 项 + 没有任何 verif 任务 → 追加 verifier 提示。

注释（`TaskUpdateTool.ts:328-332`）：

> Mirrors the TodoWriteTool nudge for V1 sessions; this covers V2 (interactive CLI). TaskUpdateToolOutput is @internal so this field does not touch the public SDK surface.

## 4. 输出

```ts
// TaskUpdateTool.ts:364-405
mapToolResultToToolResultBlockParam(content, toolUseID) {
  const { success, taskId, updatedFields, error, statusChange, verificationNudgeNeeded } = content as Output
  if (!success) {
    // Return as non-error so it doesn't trigger sibling tool cancellation in StreamingToolExecutor
    return { tool_use_id: toolUseID, type: 'tool_result', content: error || `Task #${taskId} not found` }
  }
  let resultContent = `Updated task #${taskId} ${updatedFields.join(', ')}`
  if (statusChange?.to === 'completed' && getAgentId() && isAgentSwarmsEnabled()) {
    resultContent += '\n\nTask completed. Call TaskList now to find your next available agent or see if your work unblocked others.'
  }
  if (verificationNudgeNeeded) {
    resultContent += `\n\nNOTE: You just closed out 3+ tasks and none of them was a verification step. Before writing your final summary, spawn the verification agent ...`
  }
  return { tool_use_id: toolUseID, type: 'tool_result', content: resultContent }
}
```

**关键设计**：失败时返回**非 error** 的 tool_result。注释解释（`TaskUpdateTool.ts:374-378`）：

> Return as non-error so it doesn't trigger sibling tool cancellation in StreamingToolExecutor. "Task not found" is a benign condition (e.g., task list already cleaned up) that the model can handle.

防止 `StreamingToolExecutor` 把 TaskUpdate 失败当作 sibling_error，错误地取消其他并行的 tool_use。

---

# TaskListTool —— 列出当前任务清单

## 1. 职责

TaskListTool 是 V2 任务清单的**只读视图**。无参数调用，返回当前所有任务的精简列表（不含 description 等大字段）。

## 2. Schema

```ts
// TaskListTool.ts:13-14
const inputSchema = lazySchema(() => z.strictObject({}))
```

**空 schema** —— 无任何参数。

## 3. call 内部流程

```ts
// TaskListTool.ts:65-90
async call() {
  const taskListId = getTaskListId()
  const allTasks = (await listTasks(taskListId)).filter(
    t => !t.metadata?._internal,  // 过滤内部任务
  )

  // 已完成的 id 集合 —— 用于清理 blockedBy
  const resolvedTaskIds = new Set(
    allTasks.filter(t => t.status === 'completed').map(t => t.id)
  )

  const tasks = allTasks.map(task => ({
    id: task.id,
    subject: task.subject,
    status: task.status,
    owner: task.owner,
    blockedBy: task.blockedBy.filter(id => !resolvedTaskIds.has(id)),
  }))

  return { data: { tasks } }
}
```

**关键过滤**：
1. `metadata._internal === true` 的任务不返回（隐藏内部任务）；
2. `blockedBy` 中**已完成的任务**被过滤掉（避免显示 "blocked by #5 [completed]" 这种没用的依赖）。

## 4. 输出格式

```ts
// TaskListTool.ts:91-115
mapToolResultToToolResultBlockParam(content, toolUseID) {
  const { tasks } = content as Output
  if (tasks.length === 0) {
    return { tool_use_id: toolUseID, type: 'tool_result', content: 'No tasks found' }
  }
  const lines = tasks.map(task => {
    const owner = task.owner ? ` (${task.owner})` : ''
    const blocked = task.blockedBy.length > 0
      ? ` [blocked by ${task.blockedBy.map(id => `#${id}`).join(', ')}]`
      : ''
    return `#${task.id} [${task.status}] ${task.subject}${owner}${blocked}`
  })
  return { tool_use_id: toolUseID, type: 'tool_result', content: lines.join('\n') }
}
```

**给模型的可读格式**：

```
#1 [pending] 实现核心游戏逻辑
#2 [in_progress] 美化 UI 与动画效果 (general-purpose)
#3 [pending] 构建与冒烟测试 [blocked by #1, #2]
```

---

# TaskGetTool —— 按 ID 查询单个任务

## 1. 职责

TaskGetTool 按 ID 查询**完整**任务详情（含 description / blocks / blockedBy），区别于 TaskListTool 的精简列表。

## 2. Schema

```ts
// TaskGetTool.ts:13-21
const inputSchema = lazySchema(() =>
  z.strictObject({
    taskId: z.string().describe('The ID of the agent to retrieve'),
  }),
)
```

**单参数**：`taskId`。

## 3. call 内部流程

```ts
// TaskGetTool.ts:73-98
async call({ taskId }) {
  const taskListId = getTaskListId()
  const task = await getTask(taskListId, taskId)

  if (!task) {
    return { data: { task: null } }  // 返回 null（非 throw）
  }

  return {
    data: {
      task: {
        id: task.id,
        subject: task.subject,
        description: task.description,
        status: task.status,
        blocks: task.blocks,
        blockedBy: task.blockedBy,
      },
    },
  }
}
```

**与 TaskListTool 区别**：返回 `description` + 双向依赖关系（`blocks` + `blockedBy`）。

**任务不存在**：返回 `{ task: null }` 而非 throw —— 允许模型容错处理。

## 4. 输出格式

```ts
// TaskGetTool.ts:99-127
mapToolResultToToolResultBlockParam(content, toolUseID) {
  const { task } = content as Output
  if (!task) {
    return { tool_use_id: toolUseID, type: 'tool_result', content: 'Task not found' }
  }

  const lines = [
    `Task #${task.id}: ${task.subject}`,
    `Status: ${task.status}`,
    `Description: ${task.description}`,
  ]

  if (task.blockedBy.length > 0) {
    lines.push(`Blocked by: ${task.blockedBy.map(id => `#${id}`).join(', ')}`)
  }
  if (task.blocks.length > 0) {
    lines.push(`Blocks: ${task.blocks.map(id => `#${id}`).join(', ')}`)
  }

  return { tool_use_id: toolUseID, type: 'tool_result', content: lines.join('\n') }
}
```

**给模型**：

```
Task #1: 实现核心游戏逻辑
Status: in_progress
Description: 用 React + framer-motion 实现核心游戏循环
Blocked by: #0
Blocks: #2
```

---

# TaskStopTool —— 终止运行中的后台任务

## 1. 职责

TaskStopTool 终止任何 `status === 'running'` 的后台任务（Bash / agent / workflow 等）。它**不**操作 V2 任务清单条目（V2 用 TaskUpdate 的 `'deleted'` status）。

**向后兼容**：`aliases: ['KillShell']` —— 老 transcript 里的 KillShell 调用仍能工作。

## 2. Schema

```ts
// TaskStopTool.ts:10-19
const inputSchema = lazySchema(() =>
  z.strictObject({
    task_id: z.string().optional().describe('The ID of the background agent to stop'),
    shell_id: z.string().optional().describe('Deprecated: use task_id instead'),
  }),
)
```

**两个字段都 optional**，但 validateInput 要求**至少一个**存在（`task_id ?? shell_id`）。

## 3. validateInput 三重校验

```ts
// TaskStopTool.ts:60-91
async validateInput({ task_id, shell_id }, { getAppState }) {
  const id = task_id ?? shell_id
  if (!id) return { result: false, message: 'Missing required parameter: task_id', errorCode: 1 }

  const task = appState.tasks?.[id] as TaskStateBase | undefined
  if (!task) return { result: false, message: `No task found with ID: ${id}`, errorCode: 1 }

  if (task.status !== 'running') {
    return { result: false, message: `Task ${id} is not running (status: ${task.status})`, errorCode: 3 }
  }

  return { result: true }
}
```

**校验顺序**：
1. `id` 必须存在；
2. 任务必须存在；
3. 任务必须**正在运行**（不是 pending/completed/failed）。

**注意**：pending 状态也**不能 stop** —— 因为 pending 表示还没启动，没有进程可杀。

## 4. call 内部流程

```ts
// TaskStopTool.ts:107-130
async call({ task_id, shell_id }, { getAppState, setAppState, abortController }) {
  const id = task_id ?? shell_id
  if (!id) throw new Error('Missing required parameter: task_id')

  const result = await stopTask(id, { getAppState, setAppState })

  return {
    data: {
      message: `Successfully stopped task: ${result.taskId} (${result.command})`,
      task_id: result.taskId,
      task_type: result.taskType,
      command: result.command,
    },
  }
}
```

`stopTask` 在 `src/tasks/stopTask.ts` 实现 —— 内部按 task.type 分发到不同清理逻辑：
- `local_bash` → `kill(subprocess.pid)`；
- `local_agent` → `agentTask.abortController.abort()`；
- `monitor_mcp` → 关闭 monitor 等。

## 5. 输出

```ts
// TaskStopTool.ts:98-104
mapToolResultToToolResultBlockParam(output, toolUseID) {
  return { tool_use_id: toolUseID, type: 'tool_result', content: jsonStringify(output) }
}
```

`jsonStringify` 把整个 output 对象序列化给模型 —— 模型能解析 task_id / task_type / command 字段。

## 6. UI 兼容性

```ts
// TaskStopTool.ts:46
userFacingName: () => (process.env.USER_TYPE === 'ant' ? '' : 'Stop Task')
```

外部 build 显示 "Stop Task"，ant build 不显示（用于 SDK 等场景）。

---

# 整体协作时序

下面以"用户要求生成网页版贪吃蛇 + 后台跑 dev server"为例，串联所有 6 个工具：

```
T=0     用户输入 → queryLoop 第一轮
        模型推理："需要 (1) TaskCreate 拆解 (2) 派生 Explore agent"

T=10ms  tool_use: TaskCreate(subject: "规划技术栈", description: "...", activeForm: "规划技术栈中")
        ↓
        TaskCreateTool.call
        ├─ createTask → 写 ~/.claude/tasks/<list-id>/1.json
        ├─ executeTaskCreatedHooks (空)
        ├─ setAppState({ expandedView: 'tasks' })
        └─ return { task: { id: '1', subject: '规划技术栈' } }
        ↓
        UI 任务面板自动展开

        tool_use: TaskCreate(subject: "初始化项目", ...) → id: '2'
        tool_use: TaskCreate(subject: "实现核心逻辑", ...) → id: '3'
        tool_use: TaskCreate(subject: "美化 UI", ...) → id: '4'
        tool_use: TaskCreate(subject: "构建测试", ...) → id: '5'
        tool_use: TaskCreate(subject: "生成计划图", ...) → id: '6'

        tool_use: TaskUpdate(taskId: '3', addBlockedBy: ['1', '2'])
        tool_use: TaskUpdate(taskId: '5', addBlockedBy: ['3', '4'])

T=200ms 模型推进
        tool_use: TaskUpdate(taskId: '1', status: 'in_progress')
        ↓
        TaskUpdateTool.call
        ├─ status !== existingTask.status
        ├─ 触发 executeTaskCompletedHooks (空)
        ├─ updateTask → 写 1.json
        └─ return { success: true, updatedFields: ['status'], statusChange: {from: 'pending', to: 'in_progress'} }
        ↓
        UI 显示 #1 在进行中

T=5s    tool_use: TaskUpdate(taskId: '1', status: 'completed')
        ↓
        TaskUpdateTool.call
        ├─ status === 'completed' 触发 executeTaskCompletedHooks (空)
        ├─ updateTask → 写 1.json
        └─ return { success: true, updatedFields: ['status'], statusChange: {from: 'in_progress', to: 'completed'} }
        ↓
        UI 显示 #1 完成

T=10s   模型检查清单
        tool_use: TaskList()
        ↓
        TaskListTool.call
        ├─ listTasks → 读所有 .json
        ├─ 过滤 metadata._internal
        ├─ 过滤已完成任务的 blockedBy
        └─ return { tasks: [...] }
        ↓
        mapToolResultToToolResultBlockParam
        模型看到:
        #1 [completed] 规划技术栈
        #2 [in_progress] 初始化项目
        #3 [pending] 实现核心逻辑 [blocked by #1, #2]
        #4 [pending] 美化 UI [blocked by #3]
        ...

T=20s   模型跑 dev server (后台)
        tool_use: Bash(command: "npm run dev", run_in_background: true)
        ↓
        BashTool.call → 注册 LocalShellTask, id: 'a1b2c3d4'
        ↓
        queryLoop 进入下一轮，模型继续推进

T=25s   模型想看 dev server 输出
        tool_use: TaskOutput(task_id: 'a1b2c3d4', block: false, timeout: 5000)
        ↓
        TaskOutputTool.call
        ├─ 非阻塞分支
        ├─ task.status === 'running' → return { retrieval_status: 'not_ready', task: {...} }
        └─ 模型收到 "Task is still running"

T=30s   tool_use: TaskOutput(task_id: 'a1b2c3d4', block: true, timeout: 60000)
        ↓
        TaskOutputTool.call
        ├─ 阻塞分支
        ├─ onProgress({ type: 'waiting_for_task', ... }) → UI 显示 "Waiting for task..."
        ├─ waitForTaskCompletion 轮询 100ms
        ├─ 30s 后 dev server 完成启动（或失败）
        ├─ task.status !== 'running' → return success
        └─ 模型拿到完整的 dev server stdout/stderr

T=60s   用户按 Ctrl+C 想终止 dev server
        工具调用: TaskStop(task_id: 'a1b2c3d4')
        ↓
        TaskStopTool.call
        ├─ stopTask → kill dev server 子进程
        └─ return { message: 'Successfully stopped task: a1b2c3d4' }

T=65s   模型最终化任务
        tool_use: TaskGet(taskId: '3')
        ↓
        拿到完整 description → 决定是否需要 verify
        tool_use: TaskUpdate(taskId: '3', status: 'completed')
        ...
        tool_use: TaskUpdate(taskId: '4', status: 'completed')
        ...
        tool_use: TaskUpdate(taskId: '6', status: 'completed')
        ↓
        最后一条 TaskUpdate 触发 verificationNudgeNeeded
        mapToolResultToToolResultBlockParam 追加:
        "NOTE: You just closed out 3+ tasks and none of them was a verification step. ..."
```

---

# 附录：6 个工具对比表

| 维度 | TaskOutput | TaskCreate | TaskUpdate | TaskList | TaskGet | TaskStop |
|------|-----------|------------|-----------|----------|---------|----------|
| **依赖 V2** | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **读写** | 只读 | 写 | 写 | 只读 | 只读 | 写（终止） |
| **持久化** | 内存 + 磁盘 | 磁盘 | 磁盘 | 磁盘读 | 磁盘读 | 内存 |
| **输入参数** | task_id, block, timeout | subject, description, ... | taskId, 多个 optional | 无 | taskId | task_id 或 shell_id |
| **触发钩子** | 无 | TaskCreated | TaskCompleted (status=completed) | 无 | 无 | 无 |
| **Nudge 机制** | 无 | 无 | Verification Nudge (status=completed) | 无 | 无 | 无 |
| **自动展开 UI** | 无 | ✅ `expandedView='tasks'` | ✅ 同 | 无 | 无 | 无 |
| **错误处理** | throw | throw + 自动 deleteTask | return success:false 不 throw | return [] | return null | throw via stopTask |
| **task_id vs taskId** | `task_id`（snake） | 无 | `taskId`（camel） | 无 | `taskId` | `task_id`（snake） |
| **aliases** | `AgentOutputTool`, `BashOutputTool` | 无 | 无 | 无 | 无 | `KillShell` |
| **并发安全** | ✅（只读） | ✅ | ✅ | ✅ | ✅ | ✅ |

**命名不一致**：注意 TaskOutput / TaskStop 用 `task_id`（snake_case），TaskCreate/Update/List/Get 用 `taskId`（camelCase）。这是不同工具在不同时期实现的遗留差异。

---

# 推荐阅读

- [`docs/18-task-planning.md`](18-task-planning.md) —— V1 vs V2 任务系统演进
- [`docs/26-filewrite-and-todowrite-tools.md`](../tool/26-filewrite-and-todowrite-tools.md) —— TodoWriteTool 的 V1 实现
- [`docs/19-orchestration.md`](19-orchestration.md) —— 多 agent 编排
- [`docs/12-run-agent.md`](12-run-agent.md) —— Subagent 派发（`AgentTool`）
- [`docs/25-streaming-tool-executor.md`](../tool/25-streaming-tool-executor.md) —— 这些工具的并发执行路径
- [`docs/23-design-and-core-modules.md`](../architecture/23-design-and-core-modules.md) —— 整体设计原理
