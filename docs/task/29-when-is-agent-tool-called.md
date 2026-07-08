# 什么时候主 Agent 会调用 Agent 工具 —— 源码级论证

> 本文**对照源码**详细论述主 Agent 在何种条件下会发出 `Agent` tool_use 块。
> 然后以一个具体需求为线索（**编写网页版贪吃蛇，要求画面炫酷、吸引眼球、添加动感音效，能用高大上的框架就用框架**）走查完整的端到端执行流程。

---

## 0. 文档定位

| 维度 | 本文 |
|------|------|
| 主题 | 何时**触发** Agent 工具的派发（含决策树、源码定位） |
| 切入点 | 主 Agent 看到的 `Agent` 工具提示词（`prompt`）+ 决策上下文 |
| 业务场景 | 网页版贪吃蛇（HTML5 Canvas + 动效 + 音效 + 高大上框架选型） |
| 与既有文档关系 | `docs/10-agent-tool.md`（AgentTool 自身实现）/ `docs/12-run-agent.md`（runAgent 内部）/ `docs/24-snake-game-request-flow.md`（另一版贪吃蛇流程）—— 本文聚焦"**决策**"维度 |

---

## 1. 主 Agent 看到 Agent 工具的"提示词"长什么样

主 Agent 之所以能"知道"何时该调用 Agent 工具，是因为 Claude Code 在拼装 system prompt 时把 **AgentTool 的 `prompt` 描述**塞给了模型。源码：`src/tools/AgentTool/prompt.ts:66-287` 的 `getPrompt()`。

### 1.1 主 Agent 实际看到的提示词骨架

简化后的最终输出（去掉 subscription 差异、coordinator 模式差异），主 Agent 看到的是：

```text
Launch a new agent to handle complex, multi-step tasks autonomously.

The Agent tool launches specialized agents (subprocesses) that autonomously
handle complex tasks. Each agent type has specific capabilities and tools
available to it.

Available agent types and the tools they have access to:
- general-purpose: General-purpose agent for researching complex questions,
  searching for code, and executing multi-step tasks. (Tools: All tools)
- Explore: Fast agent specialized for exploring codebases. (Tools: All tools
  except Agent, ExitPlanMode, FileEdit, FileWrite, NotebookEdit)
- Plan: Software architect agent for designing implementation plans. (Tools: …)
- statusline-setup: … (Tools: …)
- claude-code-guide: … (Tools: …)

When using the Agent tool, specify a subagent_type parameter to select which
agent type to use. If omitted, the general-purpose agent is used.

When NOT to use the Agent tool:
- If you want to read a specific file path, use Read or Glob instead of the
  Agent tool, to find the match more quickly
- If you are searching for a specific class definition like "class Foo", use
  Glob or Grep instead, to find the match more quickly
- If you are searching for code within a specific file or set of 2-3 files,
  use Read instead of the Agent tool, to find the match more quickly
- Other tasks that are not related to the agent descriptions above

Usage notes:
- Always include a short description (3-5 words) summarizing what the agent
  will do
- Launch multiple agents concurrently whenever possible, to maximize
  performance; to do that, use a single message with multiple tool uses
- When the agent is done, it will return a single message back to you. The
  result returned by the agent is not visible to the user. To show the user
  the result, you should send a text message back to the user with a concise
  summary of the result.
- You can optionally run agents in the background using the run_in_background
  parameter. When an agent runs in the background, you will be automatically
  notified when it completes — do NOT sleep, poll, or proactively check on its
  progress. Continue with other work or respond to the user instead.
- Foreground vs background: Use foreground (default) when you need the agent's
  results before you can proceed — e.g., research agents whose findings inform
  your next steps. Use background when you have genuinely independent work to
  do in parallel.
- To continue a previously spawned agent, use SendMessage with the agent's ID
  or name as the `to` field.
- The agent's outputs should generally be trusted
- Clearly tell the agent whether you expect it to write code or just to do
  research
- If the agent description mentions that it should be used proactively, then
  you should try your best to use it without the user having to ask for it
  first.
- If the user specifies that they want you to run agents "in parallel", you
  MUST send a single message with multiple Agent tool use content blocks.
- You can optionally set `isolation: "worktree"` to run the agent in a
  temporary git worktree.
```

### 1.2 关键的"禁用 / 反向"信号

`prompt.ts:233-240` 的 `When NOT to use the Agent tool` 段是**主 Agent 决定**"**不**调用 Agent"的负面清单：

| 反面条件 | 替代工具 | 源行 |
|----------|----------|------|
| 想读一个已知路径文件 | `Read` 或 `Glob` | prompt.ts:236 |
| 搜 "class Foo" 之类的符号定义 | `Glob` / `Grep` | prompt.ts:237 |
| 在 1-2 个文件内找一段代码 | `Read` | prompt.ts:238 |
| 与 agent 描述无关的任务 | —— | prompt.ts:239 |

这是一个**单向过滤器**：只要命中这 4 条之一，主 Agent 就被告知"别用 Agent"。

---

## 2. 决策维度：主 Agent 何时会决定调用 Agent

把 prompt.ts 里的所有**触发条件** + `src/constants/prompts.ts:328-333` 的 Session-specific 段 + 模型自身推理结合起来，决策维度可归纳为下面 6 个。

### 2.1 决策维度 1：子任务**可独立完成**（最关键）

主 Agent 看到的需求如果能拆出 1 个**自包含**的子任务，且子任务有清晰的"产出契约"（结构化摘要、文件列表、JSON），就会派发 Agent。

源码锚点：

- `prompt.ts:248` —— "Launch multiple agents concurrently whenever possible"
- `prompt.ts:271` —— "If the user specifies that they want you to run agents 'in parallel', you MUST send a single message with multiple Agent tool use content blocks"

### 2.2 决策维度 2：能匹配到某个 **specialized agent**

`prompt.ts:198-199` 把可用 agent 列成 `agentType: whenToUse (Tools: ...)`。主 Agent 看到需求后会做"语义匹配"。

源码锚点（`generalPurposeAgent.ts:27-28`）：

```ts
whenToUse:
  'General-purpose agent for researching complex questions, searching for
  code, and executing multi-step tasks. When you are searching for a keyword
  or file and are not confident that you will find the right match in the
  first few tries use this agent to perform the search for you.'
```

`exploreAgent.ts:61-62`：

```ts
const EXPLORE_WHEN_TO_USE =
    'Fast agent specialized for exploring codebases. Use this when you need
    to quickly find files by patterns (eg. "src/components/**/*.tsx"), search
    code for keywords (eg. "API endpoints"), or answer questions about the
    codebase (eg. "how do API endpoints work?"). When calling this agent,
    specify the desired thoroughness level: "quick" for basic searches,
    "medium" for moderate exploration, or "very thorough" for comprehensive
    analysis across multiple locations and naming conventions.'
```

`planAgent.ts:74-75`：

```ts
whenToUse:
  'Software architect agent for designing implementation plans. Use this
  when you need to plan the implementation strategy for a task. Returns
  step-by-step plans, identifies critical files, and considers architectural
  trade-offs.'
```

### 2.3 决策维度 3：避免主上下文被"中间噪声"污染

`src/constants/prompts.ts:332`（`getAgentToolSection`）写得最直白：

> Subagents are valuable for parallelizing independent queries or for protecting the main context window from excessive results, but they should not be used excessively when not needed. Importantly, **avoid duplicating work that subagents are already doing** - if you delegate research to a subagent, do not also perform the same searches yourself.

**经验法则**：主 Agent 自己搜会膨胀 100k+ token 时，转派 Agent。

### 2.4 决策维度 4：成本/速度 vs 质量的取舍

`exploreAgent.ts:78` 选 `haiku`（外部 build）或 `inherit`（ant build），`generalPurposeAgent.ts` 注释"intentionally omitted - uses getDefaultSubagentModel()"。

`exploreAgent.ts:51-56` 的 "fast agent" 注释：

> NOTE: You are meant to be a fast agent that returns output as quickly as possible. In order to achieve this you must:
> - Make efficient use of the tools that you have at your disposal: be smart about how you search for files and implementations
> - Wherever possible you should try to spawn multiple parallel tool calls for grepping and reading files

主 Agent 会**优先用 Explore 拿"快且便宜"的结果**，只在 Explore 不够时才用 general-purpose。

### 2.5 决策维度 5：后台派发 vs 前台派发

源码 `prompt.ts:262-265`：

> You can optionally run agents in the background using the `run_in_background` parameter. When an agent runs in the background, you will be automatically notified when it completes — do NOT sleep, poll, or proactively check on its progress.
> **Foreground vs background**: Use foreground (default) when you need the agent's results before you can proceed — e.g., research agents whose findings inform your next steps. Use background when you have genuinely independent work to do in parallel.

`run_in_background: true` 的本质是 `shouldRunAsync`（`AgentTool.tsx:614`）—— 满足以下任一条件就强制 async：
- `run_in_background === true`
- `selectedAgent.background === true`（agent 定义里写死）
- coordinator 模式
- fork 实验启用
- KAIROS 模式（assistant 模式）

### 2.6 决策维度 6：沙箱隔离（worktree / remote）

`prompt.ts:272-275` 写明 `isolation: "worktree"` 的使用场景。`AgentTool.tsx:637-640` 的实现：

```ts
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`;
  worktreeInfo = await createAgentWorktree(slug);
}
```

主 Agent 决定调子 Agent 时，会判断"这个子任务是否需要独立分支开发"，是 → 加 `isolation: worktree`。

---

## 3. 主 Agent 的 Agent 派发决策树（综合）

把上面 6 个维度汇成决策树：

```
用户新消息进入 queryLoop
  │
  ▼
模型推理：能否把需求拆成 1+ 个自包含子任务？
  │
  ├─ 否 → 不调 Agent，主 Agent 自己用 Read/Glob/Grep/Bash/Edit/Write 解决
  │
  └─ 是 → 进入下一步
            │
            ▼
         子任务有 1 个匹配的 specialized agent 吗？
            │
            ├─ 是 → subagent_type 设为该 agent
            │        （Explore 搜代码 / Plan 制定方案 / claude-code-guide 答 API）
            │
            └─ 否 → subagent_type 省略，默认 general-purpose
            │
            ▼
         主 Agent 接下来还需要这个子 Agent 的结果吗？
            │
            ├─ 是 → 同步派发（默认），等结果继续
            │
            └─ 否（主 Agent 能并行做别的）→ run_in_background: true
            │
            ▼
         子任务会改大量文件、需要独立分支吗？
            │
            ├─ 是 → isolation: "worktree"
            │
            └─ 否 → 默认（共享 cwd）
            │
            ▼
         prompt 是否简短、直白、给足上下文？
            │
            ├─ prompt.ts:103 强调："Brief the agent like a smart colleague
            │  who just walked into the room — it hasn't seen this conversation,
            │  doesn't know what you've tried, doesn't understand why this
            │  task matters."
            │
            └─ 模型要写一份完整的 briefing
```

---

## 4. 源码定位速查表（决策相关的关键行号）

| 决策点 | 文件 : 行号 | 说明 |
|--------|------------|------|
| 工具的"何时用"骨架 | `src/tools/AgentTool/prompt.ts:202-212` | `shared` 段说明 |
| 反面清单（"何时不用"） | `src/tools/AgentTool/prompt.ts:233-240` | `whenNotToUseSection` |
| 并发派发要求 | `src/tools/AgentTool/prompt.ts:248` | 单消息多 tool_use |
| 写好 prompt 的指引 | `src/tools/AgentTool/prompt.ts:99-113` | `writingThePromptSection` |
| 前台 vs 后台语义 | `src/tools/AgentTool/prompt.ts:262-265` | foreground vs background |
| worktree 隔离 | `src/tools/AgentTool/prompt.ts:272-275` | `isolation: worktree` |
| General-purpose 描述 | `src/tools/AgentTool/built-in/generalPurposeAgent.ts:27-28` | whenToUse |
| Explore 描述 | `src/tools/AgentTool/built-in/exploreAgent.ts:61-62` | whenToUse + 详尽度 |
| Plan 描述 | `src/tools/AgentTool/built-in/planAgent.ts:74-75` | whenToUse |
| 主 Agent 看到的 session guidance | `src/constants/prompts.ts:328-333` | `getAgentToolSection` |
| Explore vs 直接搜的明确分工 | `src/constants/prompts.ts:389-396` | "more than 3 queries" 切到 Explore |
| 决定 effective agent | `src/tools/AgentTool/AgentTool.tsx:369` | `effectiveType = subagent_type ?? ...` |
| 决定 sync/async | `src/tools/AgentTool/AgentTool.tsx:614` | `shouldRunAsync` |
| 决定 worktree | `src/tools/AgentTool/AgentTool.tsx:637-640` | `effectiveIsolation === 'worktree'` |
| agent 完成事件 | `src/tools/AgentTool/agentToolUtils.ts:323-336` | `tengu_agent_tool_completed` |
| 完成通知回流到主循环 | `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `enqueueAgentNotification` |

---

## 5. 业务场景：用户输入

```
"编写网页版的贪吃蛇，要求画面要炫酷，吸引眼球，添加动感的音效，
 能用高大上的框架就用框架"
```

### 5.1 需求拆解

| 维度 | 关键词 | 决策含义 |
|------|--------|----------|
| 媒介 | 网页版 | 浏览器可执行（HTML/JS/TS/WASM） |
| 视觉 | 炫酷、吸引眼球 | 需要粒子/后处理/Canvas/WebGL 动效 |
| 音频 | 动感音效 | Web Audio API + 程序化音效（吃食物、撞墙、升级） |
| 技术栈 | "能用高大上就用高大上" | 暗示 React/Vue/Three.js/TypeScript/Vite/Tailwind 之类 |

**复杂度判定**：这显然不是单文件 demo。涉及多个技术领域（渲染、音频、状态管理、构建、动效），是**典型的多步、多领域实现任务**。

### 5.3 不会走的分支

- **不会**走 `isolation: worktree`（因为是新建项目，不是改现成代码）
- **不会**走 `run_in_background: true`（Plan 的结果主 Agent 还要用）
- **不会**派 `general-purpose` 搜代码（用 Explore 即可）
- **不会**调 `Bash claude-code-guide` —— 这个 agent 是用来答 Claude Code 自身使用问题的

### 5.4 选型预期

主 Agent 在 Plan agent 反馈后，最终选型大概率是：

```text
- Vite 5 + React 19 + TypeScript
- 渲染：Canvas 2D（蛇身）+ WebGL/Three.js（粒子、辉光后处理）
- 状态：Zustand（轻量）
- 样式：TailwindCSS 4（utility-first） + CSS Modules
- 动效：framer-motion（UI 过渡） + GSAP（时间线动画）
- 音频：Web Audio API 直接写（程序化合成音效，比 mp3 体积小）
- 测试：Vitest
- 构建：Vite + ESBuild
```

### 5.2 决策预期

根据第 3 节的决策树，主 Agent 看到这段需求后：

1. **能否拆成子任务？** ✅ 是的（选型 / 架构 / 实现 / 测试 / 验证）
2. **有匹配的 specialized agent 吗？** ⚠️ Plan agent 命中（"plan the implementation strategy"） → 主 Agent 大概率会**先派 Plan 拿到实现方案**，再开始写代码
3. **还会派 Explore 吗？** 是的，先看 `cwd` 里是否已有 `package.json`、README、相关项目
4. **前台还是后台？** 串行（要等 Plan 的结果才能继续）
5. **要不要 worktree？** 取决于模型判断；通常这种"新建项目"会直接共享 cwd

---

## 6. 完整执行流程（按时间线）

### 6.1 时序图总览

```
T=0ms   用户在 REPL 输入需求
T=20ms  handlePromptSubmit → processTextPrompt
T=30ms  fetchSystemPromptParts（拼装 system prompt + Agent 工具的 prompt）
T=50ms  startBackgroundSession → recordTranscript 立即落盘
T=80ms  queryLoop 第一轮 → 模型流式推理
         │
         ├─ 看到需求 → 决定先派 Explore 探查 cwd
         └─ 发出 Agent tool_use (subagent_type: Explore)
                │
                ▼
T=200ms  AgentTool.call() 进入派发路径
         ├─ effectiveType = "Explore"
         ├─ selectedAgent = EXPLORE_AGENT（model = haiku, omitClaudeMd: true）
         ├─ isForkPath = false（因为 subagent_type 显式给出）
         ├─ 父 agent 同步等待
         └─ runAgent() 启动子 queryLoop
                │
                ▼
T=200ms~5s  Explore agent 在子 queryLoop 里
            ├─ ls / find / Glob 探索当前目录
            ├─ 找 package.json、README
            └─ 返回结构化摘要
                │
                ▼
T=5s    主 queryLoop 拿到 tool_result
         │
         ▼
T=5s+1  queryLoop 第二轮：模型推理
         │
         ├─ 决定派 Plan agent 制定架构
         └─ 发出 Agent tool_use (subagent_type: Plan)
                │
                ▼
T=5s+1  Plan agent（只读）
         ├─ Read context（cwd 内容）
         ├─ 设计：React 19 + Vite + TS + Tailwind + Three.js（3D 蛇）
         │   + Zustand 状态 + Web Audio API + Howler.js / 自实现
         │   + Canvas 2D/3D 动效、粒子系统、霓虹辉光
         ├─ 列出关键文件：src/main.tsx, src/game/, src/audio/, src/effects/
         └─ 返回 plan
                │
                ▼
T=30s   主 queryLoop 拿到 plan
         │
         ▼
T=30s+  主 queryLoop 开始执行：
         ├─ TodoWrite 列出 7-10 个任务
         ├─ Bash: npm create vite@latest snake -- --template react-ts
         ├─ 权限：用户选 Yes
         ├─ Bash: cd snake && npm install
         ├─ FileWrite: src/game/engine.ts（蛇逻辑）
         ├─ FileWrite: src/audio/sound.ts（Web Audio）
         ├─ FileWrite: src/effects/particles.ts（粒子系统）
         ├─ FileWrite: src/effects/neonGlow.ts（辉光 shader）
         ├─ FileWrite: src/components/Game.tsx
         ├─ FileWrite: src/components/HUD.tsx
         ├─ FileEdit: src/App.tsx, index.css
         └─ Bash: npm run build（验证构建）
                │
                ▼
T=180s  构建通过
         │
         ├─ Bash: npm run dev（run_in_background: true）
         │   → LocalShellTask 注册，UI 显示 "1 running"
         └─ 模型主动总结
                │
                ▼
T=200s  end_turn → queryLoop 终止
         ├─ handleStopHooks（无用户配置，跳过）
         ├─ executePostSamplingHooks（注册的内置）
         └─ flushSessionStorage 落盘
```

### 6.2 详细逐步解析

#### 6.2.1 T=0~80ms：用户输入到 queryLoop

`src/utils/handlePromptSubmit.ts:92` 的 `handlePromptSubmit` → `processUserInput` → `processTextPrompt`（`src/utils/processUserInput/processTextPrompt.ts:19-99`）：

- `setPromptId(randomUUID)` 启动 OTel trace
- `startInteractionSpan(text)` 创建 interaction span
- `logEvent('tengu_input_prompt')` 上报
- `createUserMessage(content: 需求文本)` 生成 user message
- `recordTranscript(messages)` **立即**写盘（保证 kill -9 后 resume 仍能恢复）

`fetchSystemPromptParts`（`src/utils/queryContext.ts:44`）并发 3 个 IO：
- `getSystemPrompt(tools, mainLoopModel, ...)` → 默认 prompt + session-specific
- `getUserContext()` → CWD/git/date/model
- `getSystemContext()` → env/git_status

其中 `getSystemPrompt` 内部会调 `AgentTool.prompt()`（`AgentTool.tsx:243-272`），把第 1 节那段长长的"何时用 Agent"提示词拼到 system prompt 里。

**关键点**：这一步决定了主 Agent 推理时**能感知**"我有 Agent 工具可用、什么时候该派 Agent"。

#### 6.2.2 T=80ms~5s：queryLoop 第一轮 —— Explore 派发

模型推理时看到 system prompt 里的 Agent 描述和 `EXPLORE_WHEN_TO_USE`，决定**先派 Explore 探查目录**（避免覆盖现有文件、避免依赖冲突）。发出的 tool_use：

```xml
<tool_use name="Agent">
  <description>Survey current directory for conflicts</description>
  <prompt>Quickly survey the current working directory at the project root.
  List any package.json, README, .gitignore, or framework files that might
  conflict with creating a new Vite + React + TypeScript project here. I'm
  about to scaffold a web-based snake game with heavy visual effects and
  Web Audio. Report findings in 5 sentences or less.</prompt>
  <subagent_type>Explore</subagent_type>
</tool_use>
```

`AgentTool.call()` 收到这个 tool_use 后：

1. 解析 `subagent_type: "Explore"`，`effectiveType = "Explore"`（`AgentTool.tsx:369`）
2. `selectedAgent = EXPLORE_AGENT`（`AgentTool.tsx:382-402`）
3. `isForkPath = false`，走普通路径
4. `shouldRunAsync = false`（前台，同步等待）—— 因为 `run_in_background` 没设
5. `effectiveIsolation = undefined`（共享 cwd）
6. `runAgent()` 启动子 queryLoop

**Explore agent 的特殊配置**（`exploreAgent.ts:64-83`）：

```ts
{
  agentType: 'Explore',
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 节省 ~5-15 Gtok/week
  disallowedTools: [
    AGENT_TOOL_NAME, EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
  ],
}
```

`omitClaudeMd: true` 意味着 Explore agent **不会**被注入项目级 `CLAUDE.md`。注释解释（`loadAgentsDir.ts:128-131`）：

> Read-only agents (Explore, Plan) don't need commit/PR/lint guidelines — the main agent has full CLAUDE.md and interprets their output.

子 queryLoop 跑完后，`runAgent` 提取 `lastAssistantMessage` 的文本作为 `output` 字段回流。`finalizeAgentTool`（`agentToolUtils.ts:277-358`）打 `tengu_agent_tool_completed` 事件后返回。

主 queryLoop 拿到的 `tool_result` 形如：

```xml
<tool_result tool_use_id="agent-explore-1">
  <output>The current directory contains only a .git/ folder. No
   package.json, README, or framework files. Safe to create a fresh
   Vite + React + TypeScript project at ./snake.</output>
  <agentId>a1b2c3d4</agentId>
  <isError>false</isError>
</tool_result>
```

#### 6.2.3 T=5s~30s：queryLoop 第二轮 —— Plan 派发

主 Agent 拿到 Explore 的"目录是空的"结论后，决定下一步：派 **Plan agent** 设计架构。

```xml
<tool_use name="Agent">
  <description>Design snake game architecture</description>
  <prompt>Design the implementation plan for a web-based Snake game with the
  following requirements:

  1. Visually stunning (cool effects, eye-catching)
  2. Dynamic sound effects (eating food, hitting wall, level up)
  3. Use a "high-end" framework stack (React 19 + Vite + TypeScript is
     the suggested baseline; consider Three.js for 3D neon glow,
     framer-motion for UI, Web Audio API for procedural sound effects)

  Context: project directory is empty (only .git/). I'll do the
  implementation after you finish. I need:
  - Recommended tech stack with justification
  - File structure (e.g., src/game/, src/audio/, src/effects/)
  - 3-5 critical files to read/create
  - Risk areas (Web Audio autoplay policy, performance on mobile)

  Output the plan only — do not write any code or files.</prompt>
  <subagent_type>Plan</subagent_type>
</tool_use>
```

`Plan agent`（`planAgent.ts:73-92`）的特殊配置：

```ts
{
  agentType: 'Plan',
  model: 'inherit',  // 复用主 Agent 的模型
  omitClaudeMd: true,
  disallowedTools: [
    AGENT_TOOL_NAME, EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
  ],
}
```

Plan agent 的 system prompt（`planAgent.ts:14-71`）开头就是 **CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS** —— 写入工具全在 `disallowedTools` 里，物理上写不了。

Plan agent 跑完后返回结构化方案：

```text
Recommended Stack:
- Vite 5 + React 19 + TypeScript
- Three.js (r170+) for 3D neon glow post-processing
- framer-motion for UI transitions (start screen → game → game over)
- Zustand for state management (lightweight, no boilerplate)
- Web Audio API (no library) for procedural sound — square/triangle
  oscillator with envelope for "beep boop" retro arcade feel
- TailwindCSS 4 for utility-first styling
- Vitest for unit tests (engine logic)

File Structure:
src/
├── main.tsx
├── App.tsx
├── game/
│   ├── engine.ts        # 蛇逻辑：移动/碰撞/吃食物
│   ├── types.ts
│   └── constants.ts
├── audio/
│   └── sound.ts         # Web Audio 合成音效
├── effects/
│   ├── particles.ts     # 粒子系统（吃食物爆开）
│   └── neonGlow.ts      # 辉光后处理 shader
├── components/
│   ├── Game.tsx         # Canvas + Three.js 渲染
│   ├── HUD.tsx          # 分数/等级 UI
│   └── StartScreen.tsx
└── hooks/
    └── useGameLoop.ts   # requestAnimationFrame 循环

Critical Files for Implementation:
- src/game/engine.ts
- src/audio/sound.ts
- src/effects/particles.ts
- src/effects/neonGlow.ts
- src/components/Game.tsx

Risks:
1. Web Audio autoplay policy — must wait for first user interaction
   (click "Start" button) before AudioContext.resume()
2. Mobile performance — Three.js post-processing is GPU-heavy; provide
   quality settings toggle
3. Audio context lifecycle — must suspend when tab is hidden
```

#### 6.2.4 T=30s~120s：主 Agent 自行实现

主 Agent 拿到 plan 后，**不再派 Agent**（因为剩下的工作都是写代码，Agent 不能替它写），主 Agent 自己在 queryLoop 里一气呵成：

```xml
<tool_use name="TodoWrite">
  <todos>
    <item><ACTIVE-FORM>初始化 Vite + React + TS 项目</ACTIVE-FORM>
          <status>in_progress</status>
          <content>初始化 Vite + React + TS 项目</content></item>
    <item><ACTIVE-FORM>安装 Three.js / framer-motion / zustand / tailwind</ACTIVE-FORM>
          <status>pending</status>
          <content>安装 Three.js / framer-motion / zustand / tailwind</content></item>
    <item><ACTIVE-FORM>实现核心游戏逻辑</ACTIVE-FORM>
          <status>pending</status>
          <content>实现 src/game/engine.ts（蛇移动/碰撞/食物）</content></item>
    <item><ACTIVE-FORM>实现 Web Audio 音效</ACTIVE-FORM>
          <status>pending</status>
          <content>实现 src/audio/sound.ts（吃食物/撞墙/升级音效）</content></item>
    <item><ACTIVE-FORM>实现粒子系统</ACTIVE-FORM>
          <status>pending</status>
          <content>实现 src/effects/particles.ts（粒子爆开）</content></item>
    <item><ACTIVE-FORM>实现 Three.js 辉光后处理</ACTIVE-FORM>
          <status>pending</status>
          <content>实现 src/effects/neonGlow.ts（辉光 shader）</content></item>
    <item><ACTIVE-FORM>实现 React 组件层</ACTIVE-FORM>
          <status>pending</status>
          <content>实现 Game/HUD/StartScreen 组件</content></item>
    <item><ACTIVE-FORM>配置 Tailwind 与主题</ACTIVE-FORM>
          <status>pending</status>
          <content>配置 Tailwind 与主题</content></item>
    <item><ACTIVE-FORM>构建与冒烟测试</ACTIVE-FORM>
          <status>pending</status>
          <content>构建与冒烟测试</content></item>
  </todos>
</tool_use>
```

接下来一连串工具调用（按顺序）：

```xml
<tool_use name="Bash">
  <description>Scaffold Vite React TS project</description>
  <command>npm create vite@latest snake -- --template react-ts</command>
</tool_use>
<tool_use name="Bash">
  <description>Install deps and runtime libs</description>
  <command>cd snake && npm install && npm install three @types/three zustand framer-motion tailwindcss@4</command>
</tool_use>
<tool_use name="FileWrite">
  <file_path>snake/src/game/engine.ts</file_path>
  <content>// 蛇逻辑：移动、碰撞、吃食物、加速
export type Point = { x: number; y: number };
export type GameState = {
  snake: Point[];
  food: Point;
  direction: 'up' | 'down' | 'left' | 'right';
  score: number;
  level: number;
  status: 'idle' | 'running' | 'paused' | 'gameover';
};

export class SnakeEngine {
  state: GameState;
  // ...
}
</content>
</tool_use>
<tool_use name="FileWrite">
  <file_path>snake/src/audio/sound.ts</file_path>
  <content>// Web Audio 合成：吃食物（短促上扬方波）/ 撞墙（低频锯齿）/ 升级（三和弦）
export class SoundEngine {
  private ctx: AudioContext;
  // ...
  eat() { /* 800Hz 方波 50ms 衰减 */ }
  crash() { /* 100Hz 锯齿 200ms + noise burst */ }
  levelUp() { /* 440/660/880Hz 三和弦 100ms each */ }
}
</content>
</tool_use>
<!-- ... 等等 -->
```

每一轮都走完整的"tool_use → 权限判定 → 执行 → tool_result"循环。注意：

- **`FileWrite` / `FileEdit` 默认要弹窗**（`mode === 'default'`），用户需逐个确认
- **`Bash` 也默认要弹窗**，但用户可以在第一次选 "Yes, and don't ask again for `npm *`" 之后免弹窗
- **大输出截断**：如果 `npm install` 输出超 `maxResultSizeChars = 25_000`（`Tool.ts:466`），会被 `applyToolResultBudget` 截断并把完整版写盘（`prepareQueryTurn` 的 Level 2）

#### 6.2.5 T=120s~180s：构建验证

模型主动跑：

```xml
<tool_use name="Bash">
  <description>Build to verify no TS errors</description>
  <command>cd snake && npm run build</command>
</tool_use>
```

构建成功 → 工具链通了。

#### 6.2.6 T=180s：启动 dev server 后台化

```xml
<tool_use name="Bash">
  <description>Start dev server in background</description>
  <command>cd snake && npm run dev</command>
  <run_in_background>true</run_in_background>
</tool_use>
```

`BashTool` 检测到 `run_in_background: true`，启动 `LocalShellTask`（`src/tasks/LocalShellTask`），注册到 `appState.backgroundShellTasks`。UI 底部状态栏出现 "1 running" pill。

注意：这一步**没有**用 Agent 工具派发 —— dev server 是个长进程，必须用 Bash 工具的后台模式。

#### 6.2.7 T=200s：end_turn + 终止

模型判断所有任务完成，发出 end_turn 停止位。`queryLoop` 走 `handleNonFollowUpTurn`：

```ts
return { outcome: { kind: 'return', terminal: { reason: 'completed' } } }
```

接下来：

1. `handleStopHooks()`（`src/query/stopHooks.ts`）—— 用户没配 Stop 钩子，跳过
2. `executePostSamplingHooks()` —— 内置 hook：生成下一步建议、turn 摘要
3. `flushSessionStorage()` —— 把整段 transcript 写盘到 `~/.claude/projects/<cwd-hash>/<sessionId>.jsonl`
4. `consumedCommandUuids` 全部 notify completed

---

## 7. 本场景下 Agent 工具调用的完整时间表

| 时刻 | 调用类型 | subagent_type | 同步/后台 | 隔离 | 触发原因（决策维度） |
|------|----------|---------------|-----------|------|---------------------|
| T=80ms~5s | 第 1 次派发 | `Explore` | 同步 | 共享 cwd | 维度 1+2：探查目录、避免冲突 |
| T=5s~30s | 第 2 次派发 | `Plan` | 同步 | 共享 cwd | 维度 1+2：多领域、需架构设计 |
| T=30s 后 | **不再派 Agent** | —— | —— | —— | 剩下的全是写代码，主 Agent 自己干 |

**关键观察**：整个流程只派了 2 次 Agent —— 一次用于只读探查，一次用于只读设计。**实现阶段不派 Agent**，因为：

1. Agent 工具的产出是"摘要文本"，不适合"写大量代码"
2. 写代码会膨胀上下文，正是 Agent 要避免的事
3. 主 Agent 自己做实现可以保持代码风格的连贯性

---

## 8. 决策树 vs 实际行为的对照

| 决策维度 | 文档约定（`prompt.ts`） | 本场景实际表现 | 一致性 |
|----------|------------------------|---------------|--------|
| 维度 1：可独立完成 | 子任务自包含 | Explore 探查 + Plan 设计都是只读子任务 | ✅ |
| 维度 2：匹配 specialized agent | whenToUse 语义匹配 | 命中 Explore（搜代码）和 Plan（设计实现） | ✅ |
| 维度 3：避免上下文污染 | 中间结果不进主上下文 | Explore/Plan 的"摘要"足够短，主上下文保持干净 | ✅ |
| 维度 4：成本/速度 vs 质量 | haiku 跑 Explore，inherit 跑 Plan | Explore 用 haiku，Plan 用 inherit（ant build 下） | ✅ |
| 维度 5：前台 vs 后台 | 默认前台，需要时后台 | 都用前台（要等结果） | ✅ |
| 维度 6：worktree 隔离 | 大改动时用 | 新建项目，不破坏现有代码 → 不用 | ✅ |

---

## 9. 异常分支：本场景下不走的路径

### 9.1 不会走 `run_in_background`

原因：Plan 的结果主 Agent 还要用。如果 `run_in_background: true`，模型会收到"等到通知"的 prompt（`prompt.ts:262-265`），必须等子 agent 完成后才会回来。同步派发更直接。

### 9.2 不会走 `isolation: worktree`

原因：项目目录是空的，没有 git 提交可"隔离"。worktree 适用于"在已有代码库上做实验"，本场景是"从零开始"。

### 9.3 不会派 `general-purpose`

原因：Explore（只读探查）和 Plan（只读设计）已经把"信息收集"和"方案设计"覆盖了。`general-purpose` 的语义是"researching complex questions, searching for code, and executing multi-step tasks"（`generalPurposeAgent.ts:28`），当 specialized agent 不够用时兜底。

### 9.4 不会派 `claude-code-guide`

原因：用户问的是"如何写贪吃蛇"，不是"如何使用 Claude Code"。`claude-code-guide` 的 whenToUse 是"Answer questions about Claude Code specifically"。

### 9.5 不会调 `Bash claude-code-guide` 后台

同 9.4。

### 9.6 不会触发 `verificationNudgeNeeded`

`TodoWriteTool.ts:75-86` 的 verification 提示需要：
- `feature('VERIFICATION_AGENT')` 启用
- `getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)` 返回 true
- 至少 3 个 todo 关闭
- 没有任何 todo 内容含 "verif"

外部 build 默认都关。即使是 ant build，主 Agent 也会被提示"spawn verification agent"，但这取决于模型是否选择遵守（`prompt.ts` 里没有强制）。

---

## 10. 5 级压缩在本场景的触发回顾

参考 `docs/09-query.md`：

| 轮次 | 触发层级 | 原因 |
|------|---------|------|
| T=80ms 第一轮 | 全部 skip | 无历史 |
| T=5s 第二轮 | microcompact 截断 Explore 的 tool_result | 子 agent 摘要过长（>25k 字符） |
| T=30s+ 实现轮 | applyToolResultBudget | `npm install` 输出超大 |
| T=120s | microcompact | 多个 FileWrite 的 result 累积 |
| 不触发 | autocompact | tokens 远低于 168k 阈值 |

---

## 11. 子 Agent 的隔离机制（为什么 Explore 的修改不影响主 Agent）

`src/utils/forkedAgent.ts:createSubagentContext`：

```ts
return {
  ...parent,
  setAppState: () => {},  // ← 主 store 看不到子 agent 的状态变更
  setAppStateForTasks: parent.setAppStateForTasks,  // ← 但长寿命任务可回写
};
```

Explore agent 写文件时也会被 `disallowedTools` 阻断 —— 所以**双重保险**：API 层面禁用 + 系统提示词强调"READ-ONLY"。

`runAgent.ts` 内部还会 `createSubagentContext` + 隔离 `toolUseContext` + 独立 `queryLoop` —— 主 Agent 和子 Agent 互不污染。

---

## 12. 主 Agent 自我反思的时机

主 Agent 完成所有 todo 后，**不会**自动触发 self-verification。但有 3 个"软提醒"机制：

1. **Verification Nudge**（feature-gated）—— 9.6 节
2. **PostToolUse 钩子**（用户自定义）—— 可挂 `npm test` / `tsc --noEmit`
3. **Stop 钩子**（用户自定义）—— 可挂 verifier agent

本场景下，外部 build 都不触发 —— 主 Agent 依靠**自身推理**决定"够不够好"。

---

## 13. 总结：决策流程的关键 takeaway

### 13.1 主 Agent 调用 Agent 工具的 6 个核心条件

1. **子任务独立**：能写出清晰的输入/输出契约
2. **匹配 specialized agent**：Explore（只读搜索）、Plan（只读设计）、general-purpose（兜底）
3. **保护主上下文**：避免大量中间结果进入主 context
4. **成本可控**：能用 haiku 的就别用 opus
5. **同步 vs 后台**：要看主 Agent 是否能并行做别的
6. **worktree 隔离**：涉及大改动时用

### 13.2 哪些场景**不**调 Agent

- 已知文件路径，用 Read 直接读
- 搜索 "class Foo" 类符号，用 Glob/Grep
- 在 1-2 个文件内找代码，用 Read
- 与任何 agent 描述都不相关的任务
- 需要写代码的任务（Agent 只能产出摘要）

### 13.3 本场景（贪吃蛇）的最终派发模式

```text
主 queryLoop
  ├─ 派 Agent(Explore)        # T=80ms~5s    探查 cwd
  ├─ 派 Agent(Plan)            # T=5s~30s    设计架构
  └─ 自行写代码（不派 Agent） # T=30s~200s
```

**两阶段派发 + 一阶段实现**，这是 Claude Code 处理"多领域、多步实现"类需求的典型模式。

---

## 附录 A：相关源码速查

| 模块 | 文件 | 关键函数/行 |
|------|------|-----------|
| Agent 工具的 prompt | `src/tools/AgentTool/prompt.ts` | `getPrompt()` 行 66-287 |
| Agent 工具 call | `src/tools/AgentTool/AgentTool.tsx` | `call()` 行 286-1447 |
| Explore agent 定义 | `src/tools/AgentTool/built-in/exploreAgent.ts` | `EXPLORE_AGENT` 行 64-83 |
| Plan agent 定义 | `src/tools/AgentTool/built-in/planAgent.ts` | `PLAN_AGENT` 行 73-92 |
| general-purpose agent | `src/tools/AgentTool/built-in/generalPurposeAgent.ts` | `GENERAL_PURPOSE_AGENT` 行 25-34 |
| 主 Agent 看到的 session guidance | `src/constants/prompts.ts` | `getAgentToolSection` 行 328-333 |
| 主循环 | `src/query.ts` | `queryLoop` 1884 行起 |
| 5 级压缩 | `src/query.ts` | `prepareQueryTurn` 行 293-462 |
| Agent 任务调度 | `src/tools/AgentTool/runAgent.ts` | `runAgent()` |
| 子 agent 隔离 | `src/utils/forkedAgent.ts` | `createSubagentContext` |
| 异步任务系统 | `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `registerAsyncAgent` |

## 附录 B：相关文档

- `docs/10-agent-tool.md`：AgentTool 工具自身的实现详解
- `docs/12-run-agent.md`：runAgent 内部流程
- `docs/24-snake-game-request-flow.md`：另一版贪吃蛇流程（侧重"全链路"）
- `docs/09-query.md`：queryLoop 完整拆解
- `docs/11-tool-execution.md`：工具执行路径
- `docs/19-orchestration.md`：多 agent 编排
- `docs/23-design-and-core-modules.md`：设计原理与核心模块

---

**本文档版本**：2026-06-28
**适用源码版本**：Claude Code dev 分支（截至 cf139f2）
