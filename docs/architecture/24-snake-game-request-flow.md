# 端到端案例：用户请求 "编写网页版贪吃蛇" 的完整执行流程

> 本文以一个具体需求为线索，把 Claude Code 的所有核心模块串联起来走一遍。
>
> **用户原话**：
> > 编写网页版的贪吃蛇，要求画面要美观，吸引眼球，能用高大上的框架就用框架，生成项目计划及任务调度图。
>
> 下面逐步走查：
> - 用户输入到 queryLoop 的链路
> - 上下文管理与 5 级压缩在本场景下的触发
> - 任务分解（TodoWrite/TaskCreate）与 subagent 派生
> - 工具执行、Bash 分类器、权限拦截
> - Hook 模块在本场景的全部触发点
> - 结果验证与自省校验
> - 终止与会话持久化

---

## 0. 场景速览

| 项 | 设定 |
|----|------|
| 项目目录 | 当前 cwd，`ls` 后为空（或已有少量文件） |
| 初始 mode | `default` |
| 初始工具池 | `getAllBaseTools()` 全部已启用（~17 个核心 + feature gate 追加） |
| Built-in agents | `general-purpose` / `statusline-setup` / `explore` / `plan` / `claude-code-guide`（取决于 `BUILTIN_EXPLORE_PLAN_AGENTS` gate） |
| Hooks | 默认无（用户没在 settings.json 配）；ant build 可能自带 sessionHook |
| 持久化 | transcript 写到 `~/.claude/projects/<cwd-hash>/<sessionId>.jsonl` |

---

## 1. 用户键入 → 进入 queryLoop

### 1.1 REPL 输入

用户在终端按 Enter 提交输入，触发链路：

```
PromptInput.onSubmit
  ↓
utils/handlePromptSubmit.ts → handlePromptSubmit
  ↓
utils/processUserInput/processUserInput.ts
  ├─ processUserInputBase
  │   ├─ 解析 slash command（无 `/` 前缀，跳过）
  │   ├─ 解析 attachment（无图片粘贴，跳过）
  │   ├─ 检查 ultraplan keyword（不含，跳过）
  │   └─ 调用 processTextPrompt
  │       ├─ setPromptId(randomUUID)        ← setPromptId 启动 OTel trace
  │       ├─ startInteractionSpan(text)     ← 创建 OTel interaction span
  │       ├─ logOTelEvent('user_prompt', ...) ← OTel user_prompt 事件
  │       ├─ logEvent('tengu_input_prompt', {is_negative, is_keep_going})
  │       └─ createUserMessage(...) → return { messages, shouldQuery: true }
  ↓
executeUserPromptSubmitHooks (若有 UserPromptSubmit 钩子)
  ↓
startBackgroundSession (REPL 主线程提交点)
```

**关键源码**：`src/utils/processUserInput/processTextPrompt.ts:19-99`，`src/utils/handlePromptSubmit.ts:92`。

### 1.2 触发 UserPromptSubmit 钩子

如果用户在 `settings.json` 里配置了 `UserPromptSubmit` 钩子，它会**在用户消息被接受前**执行：

```json
// .claude/settings.json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [
        { "type": "command", "command": "echo \"$(date)\" >> ~/.claude/prompt.log" }
      ]}
    ]
  }
}
```

钩子返回非 0 退出码会**阻塞**消息提交（`queryCheckpoint('query_hooks_start')` + `queryCheckpoint('query_hooks_end')` 之间）。本场景下用户未配置，跳过。

### 1.3 启动 background session

`startBackgroundSession` 把用户消息推入 `mutableMessages`，**同步**触发 `recordTranscript(messages)` 写盘（`src/utils/sessionStorage.ts:1408`）。这一步**不等到 API 响应**——保证 `kill -9` 后 `--resume` 仍能恢复。

```ts
// src/QueryEngine.ts:209 简化版
async function* submitMessage(prompt, options) {
  // 1. push user message 到 mutableMessages
  // 2. recordTranscript ← 立即写盘
  // 3. wrappedCanUseTool ← 包一层权限拒绝追踪
  // 4. fetchSystemPromptParts ← 装配 system prompt
  // 5. yield buildSystemInitMessage
  // 6. for await (const event of query(params)) yield event
  // 7. flushSessionStorage ← 把所有 pending 状态落盘
}
```

### 1.4 拼装 System Prompt（缓存键前缀）

`fetchSystemPromptParts`（`src/utils/queryContext.ts:44`）并发执行 3 个 IO：

```ts
const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
  getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
  getUserContext(),
  getSystemContext(),
])
```

`userContext` 通常包含：
```
CWD (absolute path)
Is git repo: true
Platform: linux
Today's date: 2026/06/26
Model: claude-opus-4-8
```

`systemContext` 包含：
```
<env>...</env>
<git_status>...</git_status>
<system-reminder>...</system-reminder>
```

`defaultSystemPrompt` 由 `getSystemPrompt`（`src/constants/prompts.ts`）从 `tools` 列表拼装——这正是缓存键的关键：

```ts
const tools = assembleToolPool(toolPermissionContext, mcpTools)
// 工具按 localeCompare 排序；built-in 在前；MCP 工具追加
// 这是 system prompt 缓存命中的核心
```

**本场景下 tool 池**：`Agent` / `TaskOutput` / `Bash` / `Glob` / `Grep` / `FileRead` / `FileEdit` / `FileWrite` / `NotebookEdit` / `WebFetch` / `WebSearch` / `TodoWrite` / `TaskStop` / `AskUserQuestion` / `Skill` / `EnterPlanMode` / `ExitPlanModeV2` / `ListMcpResources` / `ReadMcpResource` / `SendMessage` ……

---

## 2. queryLoop 第一轮

进入 `queryLoop(params, consumedCommandUuids)`（`src/query.ts:1884`）。

### 2.1 入口一次性 setup

```ts
let state: State = initializeQueryLoopState(params)
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
const config = buildQueryConfig()
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)
// memory prefetch —— 异步扫描 ~/.claude/memories/*.md（user/feedback/project/reference 四类）
```

本场景下项目根目录没有 `CLAUDE.md`，也没有 `.claude/memories/`，所以 prefetch 是空操作。

### 2.2 准备本轮消息 —— 5 级压缩流水线

`prepareQueryTurn({...})`（`src/query.ts:293-462`）：

```
┌──────────────────────────────────────────────────────────────┐
│ Level 1: getMessagesAfterCompactBoundary                     │
│   → 截取最近 compact_boundary 之后的消息                     │
│   → 第一轮 history 无 boundary，messages = 整段               │
│                                                              │
│ Level 2: applyToolResultBudget                               │
│   → 给每条 tool_result 应用 maxResultSizeChars 阈值          │
│   → 无 tool_result（第一轮），跳过                           │
│                                                              │
│ Level 3: HISTORY_SNIP (ant-only)                             │
│   → feature('HISTORY_SNIP') ? snipCompactIfNeeded            │
│   → 外部 build 下此 feature gate 默认 false，跳过             │
│                                                              │
│ Level 4: microcompact                                        │
│   → deps.microcompact(messagesForQuery, toolUseContext, 'sdk'/'repl_main_thread') │
│   → 第一轮无消息可压缩，跳过                                 │
│                                                              │
│ Level 5: CONTEXT_COLLAPSE (ant-only)                         │
│   → feature('CONTEXT_COLLAPSE') ? applyCollapsesIfNeeded     │
│   → 跳过                                                     │
│                                                              │
│ Level 6: autocompact                                         │
│   → deps.autocompact(messagesForQuery, ...)                  │
│   → tokensNearLimit() 判断：contextWindow - 13_000 阈值       │
│   → 第一轮远低于阈值，跳过                                   │
│                                                              │
│ Result: messagesForQuery ≈ [userMessage]                    │
└──────────────────────────────────────────────────────────────┘
```

> **重要**：第一轮 query 时 5 级压缩**全部命中 skip 分支**，因为还没有历史消息可压缩。这与"对话越长压缩越活跃"的直觉一致。

### 2.3 blocking limit 检查

```ts
if (!compactionResult && querySource !== 'compact' && querySource !== 'session_memory' && ...) {
  if (isAtBlockingLimit) { yield apiError; return { reason: 'blocking_limit' } }
}
```

第一轮 blocking 不会触发。

### 2.4 流式调用模型

`executeModelStreamingTurn({...})`（`src/query.ts:529-870`）：

```ts
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { ... 60 行 options ... },
})) {
  // 流式消费：text / thinking / tool_use blocks
}
```

模型收到：
- **systemPrompt**：默认 prompt + userContext + systemContext + skills 列表
- **messages**：`[{role: 'user', content: "编写网页版的贪吃蛇..."}]`
- **tools**：上面列出的 ~17+ 工具

模型推理（claude-opus-4-8）会：
1. 理解需求（生成式 AI agent、游戏、视觉吸引力）
2. **判断需要先规划**：决定调用 `EnterPlanMode` 或直接 `TodoWrite` + 派生 `Explore` agent
3. 决定工具选择顺序

**第一轮典型输出（基于源码 schema 推断）**：

```xml
<thinking>
用户要做一个"高大上"的贪吃蛇，应该先理清需求和方案。我需要：
1. 先调用 Explore 子 agent 看看当前目录（万一有上下文）
2. 用 TodoWrite 把任务拆解
3. 选定技术栈：React + Vite + Tailwind + framer-motion + canvas 粒子特效
4. 创建项目骨架
5. 实现游戏逻辑
6. 美化 UI
7. 跑构建确认产物
</thinking>

<tool_use name="TodoWrite">
  <todos>
    <item><ACTIVE-FORM>规划技术栈与架构</ACTIVE-FORM><status>in_progress</status><content>规划技术栈与架构</content></item>
    <item><ACTIVE-FORM>初始化项目骨架</ACTIVE-FORM><status>pending</status><content>初始化 Vite + React + TS + Tailwind 项目</content></item>
    <item><ACTIVE-FORM>实现核心游戏逻辑</ACTIVE-FORM><status>pending</status><content>实现核心游戏逻辑（蛇移动/碰撞/食物）</content></item>
    <item><ACTIVE-FORM>实现游戏循环和状态管理</ACTIVE-FORM><status>pending</status><content>实现游戏循环和状态管理</content></item>
    <item><ACTIVE-FORM>美化 UI 与动画效果</ACTIVE-FORM><status>pending</status><content>美化 UI 与动画效果（粒子、霓虹、渐变）</content></item>
    <item><ACTIVE-FORM>构建与冒烟测试</ACTIVE-FORM><status>pending</status><content>构建与冒烟测试</content></item>
    <item><ACTIVE-FORM>生成项目计划与任务调度图</ACTIVE-FORM><status>pending</status><content>生成项目计划与任务调度图（README/Mermaid）</content></item>
  </todos>
</tool_use>
```

注意：模型**不会**先调用 EnterPlanMode（这是用户的偏好——用户明确说要"生成项目计划及任务调度图"，模型直接进入实现流）。如果用户希望先纯规划，应该用 `Shift+Tab` 切到 plan mode。

### 2.5 后台事件流

`StreamingToolExecutor`（`src/services/tools/StreamingToolExecutor.ts:40`）开始接收 `tool_use` 块。TodoWrite 工具的 `isConcurrencySafe()` 默认返回 true（TodoWrite 不涉及副作用），所以会被立即入队执行。

---

## 3. 第一轮 follow-up：工具执行

### 3.1 TodoWriteTool.call()

`src/tools/TodoWriteTool/TodoWriteTool.ts:65-103`：

```ts
async call({ todos }, context) {
  const appState = context.getAppState()
  const todoKey = context.agentId ?? getSessionId()
  const oldTodos = appState.todos[todoKey] ?? []
  const allDone = todos.every(_ => _.status === 'completed')
  const newTodos = allDone ? [] : todos  // 全完成就清空

  // Structural nudge: 关闭 ≥3 个任务且无 verification step 时插入 verifier 提示
  let verificationNudgeNeeded = false
  if (
    feature('VERIFICATION_AGENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
    !context.agentId &&  // 只对主线程
    allDone &&
    todos.length >= 3 &&
    !todos.some(t => /verif/i.test(t.content))
  ) {
    verificationNudgeNeeded = true
  }

  context.setAppState(prev => ({
    ...prev,
    todos: { ...prev.todos, [todoKey]: newTodos },
  }))

  return { data: { oldTodos, newTodos, verificationNudgeNeeded } }
}
```

**结果**：todo 列表存进 AppState.todos[sessionId]；UI 状态栏底部出现 7 个 pending/in_progress 项；用户的 transcript 中显示一条 `Todos have been modified successfully...`。

**没有**触发任何 hook（TodoWrite 不是写文件工具，且没有用户配置的 PostToolUse 钩子）。

### 3.2 第二轮 queryLoop：模型继续推理

接 TodoWrite 的 `tool_result` 块，模型继续思考。常见下一步：

1. **派生 Explore 子 agent**：调用 `Agent` 工具，`subagent_type: 'Explore'`，prompt："Look for any existing config files, package.json, or related projects in the current directory that might conflict with creating a new snake game here."

   → 触发 `runAgent.ts`：
   - `getAgentModel` 选 haiku（或 ant 的 inherit）；
   - `createSubagentContext` 隔离 AppState；
   - 派生 `LocalAgentTask` 任务，taskId 36 进制 8 位随机；
   - 子 agent 在自己的 context 里只读探索；
   - 返回结构化摘要到主 agent。

2. **直接开始项目搭建**：调用 `Bash` 工具，command: `npm create vite@latest snake-game -- --template react-ts`。
   → 进入权限系统判定。

---

## 4. 权限系统介入：Bash 工具的 4 层防御

`runToolUse`（`src/services/tools/toolExecution.ts:337`）执行 Bash：

### 4.1 第 1 层：规则匹配

```ts
// utils/permissions/permissions.ts
const denyRule = getDenyRuleForTool(toolPermissionContext, 'Bash', command)
if (denyRule) return { behavior: 'deny', ...denyRule }

const allowRule = getAllowRuleForTool(toolPermissionContext, 'Bash', command)
if (allowRule) return { behavior: 'allow', ...allowRule }
```

**本场景**：用户 settings.json 没有 deny/allow 规则 → 规则层不命中。

### 4.2 第 2 层：path validation

```ts
// utils/permissions/pathValidation.ts
// 检测 command 里是否含 traversal、symlink、绝对路径逃逸
// npm 命令无路径相关风险，通过
```

### 4.3 第 3 层：bash classifier（ant-only 才生效）

```ts
// tools/BashTool/bashPermissions.ts:1497
export function startSpeculativeClassifierCheck(
  command, toolPermissionContext, signal, isNonInteractiveSession
): boolean {
  if (!isClassifierPermissionsEnabled()) return false
  // ...
  const promise = classifyBashCommand(command, cwd, allowDescriptions, 'allow', signal, isNonInteractive)
  promise.catch(() => {})
  speculativeChecks.set(command, promise)
  return true
}
```

**外部 build**：`isClassifierPermissionsEnabled()` 返回 false（ant-only）。这一层不触发。

**ant build**：会异步启动 LLM 分类器评估 `npm create vite@latest` 是否在 allow 描述清单里。结果可能：
- 命中 → 自动放行；
- 不命中 → 走 ask 弹窗。

### 4.4 第 4 层：auto mode classifier（仅 `permissionMode === 'auto'`，ant-only）

本场景 mode = `default`，跳过。

### 4.5 用户交互弹窗

由于 1-4 层都不自动放行，`BashTool` 触发 `useCanUseTool` → `handleInteractivePermission`：

```tsx
// components/permissions/BashPermissionRequest/BashPermissionRequest.tsx
<Dialog title="Allow this command?">
  <Select
    options={[
      { label: 'Yes', value: 'yes' },
      { label: `Yes, and don't ask again for npm create vite:*`, value: 'yes-apply-suggestions' },
      // ...
      { label: 'No', value: 'no' },
    ]}
    onChange={decision => resolve(decision)}
  />
</Dialog>
```

**此时 Hook 触发点**：`PreToolUse` 钩子（如果用户配置了）——例如一个 `npm-audit` 钩子会先扫描 registry 看依赖安全。

**用户选择 Yes** → 调用 `BashTool.call()` → `runCommand` → sandbox 执行（如果 `shouldUseSandbox` 判定为需要）。

### 4.6 Bash 执行结果回流

`ShellCommand` 实例（`utils/ShellCommand.ts`）执行 `npm create vite@latest`，产生 stdout/stderr/exitCode。`runToolUse` 把结果组装成：

```xml
<tool_result tool_use_id="abc">
  <stdout>... npm install logs ...</stdout>
  <stderr></stderr>
  <interrupted>false</interrupted>
  <isImage>false</isImage>
  <returnCodeInterpretation>npm create exited 0</returnCodeInterpretation>
</tool_result>
```

**PostToolUse 钩子触发点**（若用户配置了）：
- 例 1：`format-on-save` 钩子跑 prettier；
- 例 2：`package-audit` 钩子对 `package.json` 跑 npm audit；
- 例 3：`notification` 钩子发邮件通知 PR 创建。

---

## 5. 上下文继续累积 —— 第二轮 queryLoop

TodoWrite 已记录 7 个任务。当前 `in_progress` = "规划技术栈与架构"。

模型接下来会继续：
1. 把 in_progress 任务推进到 "completed"；
2. 把 "初始化项目骨架" 设为 in_progress；
3. 用 Bash 跑 `cd snake-game && npm install`。

**5 级压缩再次触发**（`prepareQueryTurn` 第二轮）：

```
Level 1: getMessagesAfterCompactBoundary  → 第一轮无 boundary
Level 2: applyToolResultBudget           → Bash 输出可能超 100k 字符？
                                          maxResultSizeChars 默认 25_000
                                          npm install 满输出被截断 → 写盘 + 引用
Level 3: HISTORY_SNIP                    → 跳过（feature gate）
Level 4: microcompact                    → 开始工作：清掉之前 Bash 工具的 stdout 缓存
Level 5: CONTEXT_COLLAPSE                → 跳过
Level 6: autocompact                     → tokensNearLimit = false（还早）
```

**microcompact 工作机制**（`services/compact/microCompact.ts`）：
- 找到每个 tool_use 块最近的对应 tool_result；
- 若 tool_result 超过 maxResultSizeChars，按"首尾保留 + 中间删除"策略截断；
- 用 cache_control 让 API 端只重算被替换的部分。

---

## 6. 派生 Explore 子 agent：实际会发生

到第二轮或第三轮，模型可能会调用：

```xml
<tool_use name="Agent">
  <description>Survey current directory for conflicts</description>
  <prompt>Quickly survey the current working directory. List any package.json, 
  README, .gitignore, or framework files that might conflict with creating a new 
  Vite + React + TypeScript project here. Report findings in 5 sentences or less.</prompt>
  <subagent_type>Explore</subagent_type>
</tool_use>
```

### 6.1 `AgentTool.runAgent` 执行流程

`src/tools/AgentTool/runAgent.ts`：

```ts
// 1. 解析 AgentDefinition（来自 loadAgentsDir）
const agentDef = context.options.agentDefinitions.activeAgents
  .find(a => a.agentType === 'Explore')

// 2. 选择模型
const model = agentDef.model === 'inherit' ? mainLoopModel : agentDef.model
// Explore 默认 'haiku'（外部 build）或 'inherit'（ant build）

// 3. createSubagentContext
const subagentToolUseContext = createSubagentContext(parentToolUseContext, {
  agentId: newAgentId,  // 36 进制 8 位 taskId
  agentType: 'Explore',
  // 默认：setAppState 是 no-op（隔离主 store）
})

// 4. buildForkedMessages
const forkMessages = buildForkedMessages(messages, agentId)
// 子 agent 看到的消息是主对话的一个 snapshot（截取 fork point）

// 5. 启动 LocalAgentTask
const task = registerMainSessionTask(newAgentId, agentDef, ...)
return { data: { output: extractResultText(task), agentId: newAgentId } }
```

### 6.2 子 agent 的隔离细节

```ts
// src/utils/forkedAgent.ts:createSubagentContext
return {
  ...parent,
  setAppState: () => {},  // ← 主 store 看不到子 agent 的状态变更
  setAppStateForTasks: parent.setAppStateForTasks,  // ← 但长寿命任务可回写
  // 子 agent 的 todos 会写到 AppState.todos[agentId]
  // 子 agent 的 setToolJSX 默认 no-op（除非 shareSetToolJSX）
}
```

**Explore agent 的特殊配置**（`src/tools/AgentTool/built-in/exploreAgent.ts:64`）：

```ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: 'Fast agent specialized for exploring codebases...',
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 节省 ~5-15 Gtok/week
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

**omitClaudeMd**: Explore agent 不注入项目级 CLAUDE.md。注释：

> Read-only agents (Explore, Plan) don't need commit/PR/lint guidelines — the main agent has full CLAUDE.md and interprets their output. Saves ~5-15 Gtok/week across 34M+ Explore spawns.

### 6.3 Explore 子 agent 内部运行

子 agent 进入自己的 `queryLoop`，只读探索当前目录。完成后，runAgent 把 `lastAssistantMessage` 的文本提取出来作为 `output` 字段回流。

主 agent 的下一轮 query 看到：

```xml
<tool_result tool_use_id="agent-xxx">
  <output>The current directory contains only a .git/ folder. No package.json, 
  README, or framework files. Safe to create a fresh Vite + React + TypeScript 
  project at ./snake-game.</output>
  <agentId>a1b2c3d4</agentId>
  <isError>false</isError>
</tool_result>
```

---

## 7. 主线程继续：写代码

模型根据 Explore agent 的反馈继续推进 todo list，把"初始化项目骨架"设为 completed，开始"实现核心游戏逻辑"。

### 7.1 文件编辑的工具选择

模型会用 `FileEdit`（精确字符串替换）或 `FileWrite`（整体覆盖）：
- 新文件 → `FileWrite`；
- 改文件 → `FileEdit`（带 old_string/new_string 验证）。

每次 `FileWrite` / `FileEdit` 调用都会触发：
1. **PreToolUse 钩子**（若配置）；
2. **权限系统**（默认 mode 下 `FileWrite` 总要弹窗）；
3. **文件快照**：`fileHistoryMakeSnapshot(path, content)` 写到 `<session>/file-history/<hash>.json`（`utils/fileHistory.ts`）；
4. **执行**；
5. **PostToolUse 钩子**（若配置）；
6. **PostToolUse 钩子 → 改写输出**：通过 `updatedMCPToolOutput` 字段（仅 MCP 工具生效）；

### 7.2 Bash 跑 `npm run dev` 后台化

```xml
<tool_use name="Bash">
  <command>npm run dev</command>
  <run_in_background>true</run_in_background>
</tool_use>
```

`BashTool` 启动后台 shell，注册到 `tasks/LocalShellTask`：

```ts
// src/utils/ShellCommand.ts
class ShellCommand {
  status: 'running' | 'backgrounded' | 'completed' | 'killed'
  // ...
}
```

UI 底部出现 "1 running" 状态 pill。

---

## 8. Hook 模块在本场景下的全部触发点

下面汇总**所有可能**触发 Hook 的位置（按时间顺序），以及默认状态下哪些被实际触发。

### 8.1 Hook 事件完整清单

`src/entrypoints/agentSdkTypes.ts:HOOK_EVENTS`：

| 事件 | 触发时机 | 本场景是否触发 |
|------|---------|----------------|
| `UserPromptSubmit` | 用户提交消息时 | 默认无；用户配置则触发 |
| `PreToolUse` | 每次工具调用前 | 默认无；用户配置则触发 |
| `PostToolUse` | 每次工具调用后 | 默认无；用户配置则触发 |
| `PreCompact` | 压缩前 | autocompact 触发时可能 |
| `PostCompact` | 压缩后 | 同上 |
| `Notification` | 通知类事件 | 视配置 |
| `Stop` | 主 agent 结束一轮 | 每轮结束 |
| `SubagentStop` | 子 agent 结束 | 子 agent 完成时 |
| `SessionStart` | 会话开始 | 启动时 |
| `SessionEnd` | 会话结束 | 退出时 |

### 8.2 Hook 的 4 种执行类型

`src/schemas/hooks.ts:182-189`：

```ts
HookCommandSchema = z.discriminatedUnion('type', [
  BashCommandHookSchema,  // type: 'command'
  PromptHookSchema,       // type: 'prompt'
  AgentHookSchema,        // type: 'agent'
  HttpHookSchema,         // type: 'http'
])
```

| 类型 | 实现 | 本场景用途 |
|------|------|-----------|
| `command` | spawn shell 命令 | `PreToolUse` 时跑 lint；`PostToolUse` 时跑 formatter |
| `prompt` | 调小模型做 prompt 判定 | `PreToolUse` 时让 Haiku 评估"这条命令是否安全" |
| `agent` | 派生 subagent 做完整任务 | `Stop` 时让 verifier 检查产物 |
| `http` | POST 到外部 endpoint | `PostCompact` 时通知监控；`Notification` 时发 webhook |

### 8.3 Hook 链匹配规则

`src/utils/hooks/hooksSettings.ts:92`：

```ts
export function getAllHooks(appState: AppState): IndividualHookConfig[] {
  // 合并来源：
  // 1. userSettings (~/settings.json)
  // 2. projectSettings (.claude/settings.json)
  // 3. localSettings (.claude/settings.local.json)
  // 4. sessionHooks (本次会话注册的临时 hooks)
  // 5. pluginHook / builtinHook
  
  // matcher: e.g., 'Write' 或 '*'
  // if 条件: e.g., 'Bash(git *)' —— 只对匹配的 tool_use 触发
}
```

### 8.4 Hook 退出码语义

```ts
// 退出码 0 → 成功，继续
// 退出码 2 → 阻塞错误，把 stderr 注入 tool_result 阻止继续
// 其它  → 记录但不阻塞
```

`async: true` 时不阻塞当前 turn，hook 后台跑；`asyncRewake: true` + 退出码 2 时唤醒模型。

### 8.5 本场景下 Hook 的典型配置示例（推荐）

```json
// .claude/settings.json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [
        { "type": "command", "command": "pbcopy < /dev/stdin" }  // macOS：把 prompt 拷到剪贴板
      ]}
    ],
    "PreToolUse": [
      { "matcher": "Bash", "hooks": [
        { "type": "command", "command": "echo \"$(date) $CLAUDE_TOOL_INPUT\" >> ~/.claude/bash.log" }
      ]}
    ],
    "PostToolUse": [
      { "matcher": "Edit|Write|MultiEdit", "hooks": [
        { "type": "command", "command": "npx prettier --write \"$CLAUDE_FILE_PATHS\"" }
      ]}
    ],
    "Stop": [
      { "hooks": [
        { "type": "agent", "prompt": "Verify the snake game builds and renders correctly. Check package.json scripts, src/App.tsx, and any game logic files." }
      ]}
    ]
  }
}
```

---

## 9. 结果验证与自省校验

### 9.1 Claude Code 的"自省"到底做了什么

源码中没有 readme.txt 描述的"三层强制校验流水线"。实际验证机制有 **3 类**，但都不是强制性：

#### 9.1.1 PostToolUse 钩子（用户自定义）

用户在 `settings.json` 里配的 PostToolUse 命令。如果用户配了 `npm test` 或 `tsc --noEmit`，则每次写代码后自动跑。

#### 9.1.2 Verification Nudge（feature-gated）

`src/tools/TodoWriteTool/TodoWriteTool.ts:75-86`：

```ts
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
  !context.agentId &&
  allDone &&
  todos.length >= 3 &&
  !todos.some(t => /verif/i.test(t.content))
) {
  verificationNudgeNeeded = true
}
```

当 TodoWrite 检测到全部完成且 ≥3 个任务时，**仅追加一段提示**到 tool_result：

> NOTE: You just closed out 3+ tasks and none of them was a verification step. Before writing your final summary, spawn the verification agent (subagent_type="general-purpose" with prompt "verify"). You cannot self-assign PARTIAL by listing caveats in your summary — only the verifier issues a verdict.

这是**提示**而非强制。模型可以忽略。

#### 9.1.3 Verification Agent（feature-gated）

`VERIFICATION_AGENT` 在 `tengu_hive_evidence` GrowthBook flag 启用后才会注入到 built-in agents（`builtInAgents.ts:64-69`）。

```ts
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)
) {
  agents.push(VERIFICATION_AGENT)
}
```

注意：外部 build 默认 feature gate 是 `false`，这个 agent **不会**出现在用户可见的 agent 列表里。

### 9.2 本场景下"自省"的实际路径

外部 build、用户没配 Stop 钩子时：

1. 模型完成所有 todo；
2. 主动总结产出；
3. 主动跑 `npm run build`（Bash 工具）确认无编译错误；
4. 输出最终 summary 给用户。

**没有**任何 Claude Code 自动跑 lint / 测试 / Sonar / 覆盖率。

### 9.3 Stop 钩子链（query.ts:6）

```ts
// src/query/stopHooks.ts:handleStopHooks
async function* handleStopHooks(...) {
  // 异步迭代器，逐个 yield stop hook 的进度
  // 用户配的 Stop 钩子在这里执行
  // 返回 { blockingErrors, preventContinuation }
}
```

`blockingErrors` 会被注入下一轮让模型继续；`preventContinuation` 立即终止（`reason: 'stop_hook_prevented'`）。

---

## 10. queryLoop 终止与持久化

### 10.1 终止条件

模型完成所有任务后，调用 `end_turn` 停止位（`stop_reason: 'end_turn'`）。`queryLoop` 的 `handleNonFollowUpTurn` 命中默认分支：

```ts
return { outcome: { kind: 'return', terminal: { reason: 'completed' } } }
```

### 10.2 stop hook 链执行

`handleStopHooks(...)` 在返回 `completed` 前执行所有用户配置的 Stop 钩子。本场景用户没配，跳过。

### 10.3 PostSamplingHook 触发

```ts
// src/query.ts queryLoop 阶段 5
void executePostSamplingHooks([...messagesForQuery, ...assistantMessages], ...)
```

PostSamplingHook 是**内置** hook 机制（不暴露在 settings.json），通过 `registerPostSamplingHook()` 注册。本场景下注册的几个 hook 包括：
- `promptSuggestion`：每轮后异步生成下一步建议（side-channel）；
- `postTurnSummary`：异步生成 turn 摘要。

### 10.4 会话持久化

```ts
// src/QueryEngine.ts submitMessage finally
finally {
  setReadFileCache(engine.getReadFileState())
  // 真正落盘由 utils/sessionStorage.ts:flushSessionStorage 异步完成
}
```

`flushSessionStorage` 把 transcript 写入 `<project>/<sessionId>.jsonl`。**关键**：每条消息带 `uuid` 和 `parentUuid` 形成链表，支持 `--resume` 时按 parent chain 重建。

`fileHistory/` 目录保存所有 FileEdit/FileWrite 前的快照，支持 `/rewind` 回滚。

`consumedCommandUuids` 在 query 正常返回时通知 lifecycle completed；throw 时不发（保留 "started but not completed" 信号）。

---

## 11. 完整时序图

```
T=0    用户在 REPL 输入 "编写网页版的贪吃蛇..."
T=10ms onSubmit → handlePromptSubmit → processUserInputBase
       → processTextPrompt
       ├─ setPromptId (OTel trace)
       ├─ startInteractionSpan
       ├─ logEvent('tengu_input_prompt')
       └─ createUserMessage
T=20ms UserPromptSubmit 钩子链（如配置）
T=30ms startBackgroundSession
       ├─ push user message 到 mutableMessages
       ├─ recordTranscript → 写盘到 ~/.claude/projects/<hash>/<session>.jsonl
       ├─ wrappedCanUseTool
       ├─ fetchSystemPromptParts (并发 3 个 IO)
       │   ├─ getSystemPrompt  → 拼装 system prompt + 缓存键
       │   ├─ getUserContext   → CWD/git/date/model
       │   └─ getSystemContext → env/git_status
       └─ yield buildSystemInitMessage
T=80ms queryLoop 第一轮
       ├─ pendingMemoryPrefetch 启动（空操作）
       ├─ prepareQueryTurn
       │   ├─ Level 1-6 全部 skip（无历史）
       │   └─ messagesForQuery = [user]
       ├─ isAtBlockingLimit → false
       └─ executeModelStreamingTurn
            └─ deps.callModel (stream)
                 ↓ [stream chunks]
                 ├─ text: "我先规划..."
                 ├─ thinking blocks
                 └─ tool_use: TodoWrite
T=3s    StreamingToolExecutor 完成 TodoWrite
       ├─ TodoWriteTool.call
       │   ├─ setAppState({ todos: [...] })
       │   └─ return { data: { oldTodos, newTodos, verificationNudgeNeeded: false } }
       └─ tool_result 回流
T=3.5s  queryLoop 第二轮
       ├─ prepareQueryTurn
       │   ├─ Level 4 microcompact 命中（TodoWrite 的 result 小）
       │   ├─ Level 1-3 skip
       │   └─ tokensNearLimit = false
       └─ executeModelStreamingTurn (stream)
            ├─ text: "用 React + Vite + TypeScript + Tailwind + framer-motion"
            └─ tool_use: Agent (Explore)
T=5s    Explore 子 agent 启动
       ├─ createSubagentContext (setAppState no-op)
       ├─ 启动 LocalAgentTask (taskId: 36 进制 8 位)
       ├─ 派生子 queryLoop（model = haiku）
       │   ├─ Read / Bash (ls / find) → 探索目录
       │   └─ 返回结构化摘要
       └─ 主 agent 收到 tool_result
T=7s    queryLoop 第三轮
       ├─ 推进 todo (init → completed, install → in_progress)
       ├─ tool_use: Bash (npm create vite@latest snake-game -- --template react-ts)
       └─ runToolUse
            ├─ 权限判定：deny/allow 规则无 → 进入 ask
            ├─ PreToolUse 钩子（如配置）
            ├─ BashPermissionRequest 弹窗
            └─ 用户点 Yes
T=15s   Bash 执行 npm create
       ├─ ShellCommand 执行（可能进 sandbox）
       ├─ 输出超大 → applyToolResultBudget 截断 + 写盘
       └─ PostToolUse 钩子（如配置）
T=20s   queryLoop 第四轮
       ├─ microcompact 替换 Bash 的 tool_result（截断版）
       ├─ tool_use: Bash (cd snake-game && npm install)
       ├─ tool_use: FileWrite (src/App.tsx)
       │   ├─ fileHistoryMakeSnapshot
       │   ├─ FileWriteTool.call
       │   └─ PostToolUse 钩子（format/prettier）
       └─ ...
T=120s  多个 FileWrite / FileEdit 完成核心代码
       ├─ microcompact 持续截断大工具结果
       ├─ TodoWrite 持续推进
       └─ 模型主动跑 Bash (npm run build)
T=130s  构建成功，模型跑 Bash (npm run dev, run_in_background=true)
       ├─ LocalShellTask 注册
       └─ UI 状态栏 "1 running"
T=140s  最终轮 query
       ├─ TodoWrite 全部完成（注意 allDone=true 清空 todos）
       ├─ 触发 verificationNudgeNeeded 检查（feature gate 默认 false，跳过）
       ├─ 模型输出最终总结
       └─ end_turn
T=145s  queryLoop 终止
       ├─ handleNonFollowUpTurn → terminal { reason: 'completed' }
       ├─ handleStopHooks（无 Stop 钩子，跳过）
       ├─ executePostSamplingHooks (注册的内置 hooks)
       ├─ flushSessionStorage 落盘 transcript
       └─ consumedCommandUuids 全部 notify completed
```

---

## 12. 5 级压缩的实际触发时刻回顾

| 时刻 | 触发层级 | 原因 |
|------|---------|------|
| T=80ms 第一轮 | 全部 skip | 无历史 |
| T=3.5s 第二轮 | microcompact 开始有活干 | TodoWrite tool_result |
| T=20s Bash npm create | **applyToolResultBudget** | Bash 输出超大 |
| T=30s+ 持续 | **microcompact** | 多个 Bash/FileWrite 大 result |
| 假设 token 接近 168k (200k - 13k - 20k) | **autocompact** | 调用 compactConversation |
| 假设 reactiveCompact feature enabled | reactiveCompact | API 返回 prompt_too_long 时 |

---

## 13. 本场景下没有触发的功能

| 功能 | 原因 |
|------|------|
| `EnterPlanMode` / `ExitPlanModeV2` | 用户没切到 plan mode |
| `Bash` bash classifier | 外部 build 下 isClassifierPermissionsEnabled()=false |
| `auto mode classifier` | mode='default' |
| `CONTEXT_COLLAPSE` / `HISTORY_SNIP` | ant-only feature gates |
| `verificationNudgeNeeded` | feature('VERIFICATION_AGENT')=false |
| `ToolSearch` | `ENABLE_TOOL_SEARCH` 未设 |
| MCP deferral | 同上 |
| PostToolUse hook | 用户未配置 |
| Stop hook | 用户未配置 |
| `ExitWorktree` | 未进入 worktree mode |

---

## 14. 与 readme.txt 的偏差澄清

| readme.txt 描述 | 实际源码行为 |
|----------------|-------------|
| "Plan Mode 强制派生 Explore 全域扫描" | Plan Mode 仅禁写工具；Explore 必须显式 Agent 调用 |
| "三层强制校验（lint/Sonar/单测）" | Claude Code 不集成任何校验工具；PostToolUse 钩子由用户配 |
| "派生 4 类专用 Subagent (Explore/Plan/Implement/Test)" | 只有 Explore/Plan 是 built-in；Implement/Test 是用户自定义 |
| "5 层压缩 Budget Reduction → Snip → Microcompact → Context Collapse → Auto-compact" | 实际顺序：applyToolResultBudget(子步) → snip → microcompact → CONTEXT_COLLAPSE → autocompact → reactiveCompact |
| "30+ 工具" | 核心稳定工具 ~17 个 |
| "Bash 自动跑 lint" | 无此功能 |

---

## 附录：关键源码速查

| 模块 | 文件 | 行数 |
|------|------|------|
| 主循环 | `src/query.ts` | 2,337 |
| QueryEngine | `src/QueryEngine.ts` | 1,303 |
| 5 级压缩串联 | `src/query.ts` `prepareQueryTurn` | 293-462 |
| microcompact | `src/services/compact/microCompact.ts` | - |
| autocompact | `src/services/compact/autoCompact.ts` | - |
| reactiveCompact | `src/services/compact/reactiveCompact.ts` | - |
| 权限系统 | `src/utils/permissions/permissions.ts` | 1,486 |
| Bash 分类器 | `src/utils/permissions/bashClassifier.ts` + `src/tools/BashTool/bashPermissions.ts` | 1,527 |
| Hook 系统 | `src/utils/hooks.ts` | 5,297 |
| Hook Schema | `src/schemas/hooks.ts` | 222 |
| TodoWrite | `src/tools/TodoWriteTool/TodoWriteTool.ts` | 113 |
| Agent 派发 | `src/tools/AgentTool/runAgent.ts` | - |
| 子 agent 隔离 | `src/utils/forkedAgent.ts` | - |
| Built-in agents | `src/tools/AgentTool/builtInAgents.ts` + `built-in/` | - |
| StreamingToolExecutor | `src/services/tools/StreamingToolExecutor.ts` | - |
| PostToolUse 钩子链 | `src/services/tools/toolHooks.ts` | - |
| Bash 工具 | `src/tools/BashTool/BashTool.tsx` | 1,000+ |
| 会话持久化 | `src/utils/sessionStorage.ts` | 5,105 |
| transcript 写入 | `src/utils/sessionStorage.ts` `recordTranscript` | 1,408 |

---

## 推荐阅读

- [`docs/09-query.md`](../09-query.md)：queryLoop 完整拆解
- [`docs/11-tool-execution.md`](../tool/11-tool-execution.md)：工具执行路径
- [`docs/12-run-agent.md`](../task/12-run-agent.md)：Subagent 派生与 fork
- [`docs/14-tool-hooks.md`](../tool/14-tool-hooks.md) + [`docs/15-utils-hooks.md`](../hook/15-utils-hooks.md)：Hook 系统
- [`docs/18-task-planning.md`](../task/18-task-planning.md)：TodoWrite 与 TaskCreate
- [`docs/20-request-flow.md`](20-request-flow.md)：请求流转
- [`docs/23-design-and-core-modules.md`](23-design-and-core-modules.md)：设计原理与核心模块（含 readme.txt 偏差澄清）