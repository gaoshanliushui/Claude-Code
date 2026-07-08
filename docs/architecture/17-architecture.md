# Claude Code 架构与设计原理综合分析

> 本文基于 `src/` 下 1,987 个 TypeScript / TSX 源文件的还原版本（来自 `@anthropic-ai/claude-code` npm 包的 source map），从架构师视角系统论述 Claude Code（CLI）的设计原理、组织哲学与核心模块。前 16 篇文档（`01-buddy.md` ~ `16-claude-api.md`）分别剖析了具体功能与子系统，本文做整体抽象与设计原理的归纳。

---

## 目录

1. [设计哲学总览](#1-设计哲学总览)
2. [整体架构鸟瞰](#2-整体架构鸟瞰)
3. [核心模块详解](#3-核心模块详解)
   - [3.1 入口与生命周期：`cli.tsx` / `main.tsx`](#31-入口与生命周期clitsx--maintsx)
   - [3.2 查询引擎：`query.ts` 与 `QueryEngine.ts`](#32-查询引擎queryts-与-queryenginets)
   - [3.3 工具系统：`Tool.ts` / `tools/`](#33-工具系统toolts--tools)
   - [3.4 Agent 派发：`tools/AgentTool/`](#34-agent-派发toolsagenttool)
   - [3.5 权限系统：`utils/permissions/`](#35-权限系统utilspermissions)
   - [3.6 UI 渲染：Ink + React + 148 个组件](#36-ui-渲染ink--react--148-个组件)
   - [3.7 状态管理：`state/AppState.tsx` 与 `state/store.ts`](#37-状态管理stateappstatetsx-与-storests)
   - [3.8 会话持久化与恢复](#38-会话持久化与恢复)
   - [3.9 上下文压缩：`services/compact/`](#39-上下文压缩servicescompact)
   - [3.10 钩子系统：`utils/hooks.ts`（5,297 行）](#310-钩子系统utilshooksts5297-行)
   - [3.11 MCP 集成：`services/mcp/`](#311-mcp-集成servicesmcp)
   - [3.12 任务系统：`tasks/`](#312-任务系统tasks)
4. [横切关注点](#4-横切关注点)
   - [4.1 三层功能门控](#41-三层功能门控)
   - [4.2 性能与启动优化](#42-性能与启动优化)
   - [4.3 提示词缓存策略](#43-提示词缓存策略)
   - [4.4 可观测性 / 遥测 / 增长实验](#44-可观测性--遥测--增长实验)
5. [多模式架构](#5-多模式架构)
6. [关键设计模式总结](#6-关键设计模式总结)
7. [总结：这套架构给我们什么启示](#7-总结这套架构给我们什么启示)

---

## 1. 设计哲学总览

通过对源码的系统阅读，可以提炼出 Claude Code 的 **12 条设计哲学**。这些原则不是单点技术，而是贯穿 1,987 个文件的整体取向。

### 1.1 **以 Prompt 缓存命中率为第一类性能指标**

`getAllBaseTools()` 注释里有一句被强调的 NOTE（`tools.ts:193`）：

> This MUST stay in sync with … in order to cache the system prompt across users.

`assembleToolPool` 显式注释了 *为什么* 要把 built-in 工具放在 MCP 工具之前排序：Statsig 服务端的缓存断点策略在最后一个 built-in 工具之后放置全局缓存断点；如果把 MCP 工具混排进 built-in 工具之间，每一次新 MCP 工具都会让所有下游缓存键失效。

整个 prompt 构造层在围绕 **"怎样让更多请求共享同一个前缀字节"** 来设计：工具列表按 `localeCompare` 排序、MCP 工具压平前缀后追加、`userContext` 在跨调用间做 `memoize`、模型固定为最新别名（`claude-opus-4-8`）以保证模型级缓存复用。

### 1.2 **三层功能门控是产品分层的基础**

```ts
// 编译时
feature('BUDDY')         // bun:bundle dead-code elimination
// 用户类型
process.env.USER_TYPE === 'ant'
// 远程配置
getFeatureValue_CACHED_MAY_BE_STALE('tengu_kairos', false)
```

外部发布版只看到 `feature()` 评估为 `false` 的代码路径——约 50 个开关构建时剪掉约 30% 的功能体积。`ant` 用户在此基础上看到 26+ 个内部命令、特殊工具、`raw-read` 的 MDM 接口、20 分钟（而非 6 小时）的 GrowthBook 刷新。远程 GrowthBook 在两者之上做 A/B 灰度。

### 1.3 **"REPL 视图 vs SDK 视图" 是一个可分离关注点**

`QueryEngine.ts` 顶部的 JSDoc 写得很清楚：

> It extracts the core logic from `ask()` into a standalone class that can be used by both the headless/SDK path and (in a future phase) the REPL.

`QueryEngine` 是状态机：消息历史、token 用量、读取文件缓存、权限拒绝记录。`useCanUseTool`/`interactiveHelpers.tsx`/`REPL.tsx` 是视图：把状态机的事件流画到终端。

这种 **"引擎与表现分离"** 的设计让 SDK 用户（`@anthropic-ai/claude-agent-sdk`）能复用同一份对话状态机，而不需要把 Ink/React 拉进自己的进程。

### 1.4 **工具优先于 Bash，但 Bash 永远兜底**

`utils/permissions/permissions.ts`（1,486 行）和 `tools/BashTool/bashPermissions.ts` 共同实现了 **"工具能拦截时一定拦截，拦不住的由 Bash 兜底"**。例如：

- `Edit` / `Write` 工具：先做路径在白名单/黑名单里的规则匹配（`shellRuleMatching.ts`），失败再交给 Bash classifier
- `Bash`：单独的 `bashClassifier`（用 LLM 二次判定） + 命令解析（`shell-quote`）+ 沙箱决定（`shouldUseSandbox.ts`）

为什么不让一切都走 Bash？因为给工具名+JSON Schema 的 `tool_use` 块在 UI 上能画图标、能精确审计、能从模型侧并行执行；而 Bash 字符串是不透明的。

### 1.5 **每个子 Agent 拥有独立状态机，但通过文件系统/IPC 协同**

`utils/forkedAgent.ts` 的 `createSubagentContext()` 关键操作是构造一个 **setAppState 是 no-op 的 ToolUseContext**——子 Agent 看到的 AppState 是快照。但同时保留一个 `setAppStateForTasks` 通道，让子 Agent 注册的后台任务（输出文件、监视器）写回主进程 store。

子 Agent 之间的协作不是进程内对象共享，而是：
- **共享文件系统**（worktree 隔离是可选的）
- **共享任务列表**（`~/.claude/tasks/<id>.json`）
- **`SendMessage` 工具**（`tools/SendMessageTool/`）通过 named address 发送消息
- **远程可达**（`tools/RemoteAgentTask/`，CCR 容器中启动）

### 1.6 **持久化是无处不在的隐式能力**

`utils/sessionStorage.ts` 长达 5,105 行。每一类可恢复的运行时状态都有写盘策略：

- 消息历史：`recordTranscript`（**在用户消息被接受时就写**，不等到 API 响应——防止 `kill -9` 后无法 `--resume`）
- 文件编辑：`fileHistoryMakeSnapshot`（可回放任意一次修改前的状态）
- 子 Agent：`recordSidechainTranscript` + `writeAgentMetadata`
- 待处理权限：`OrphanedPermission`（断线后下次启动恢复）
- 遥测队列：`firstPartyEventLoggingExporter`

`QueryEngine` 的注释解释了一个微妙的设计权衡：

> fire-and-forget. Scripted calls don't --resume after kill-mid-request. The await is ~4ms on SSD, ~30ms under disk contention — the single largest controllable critical-path cost after module eval.

**单次 `await recordTranscript` 是 4ms**——对交互式 REPL 不可忽视，对 `--bare` 脚本模式可忽略。

### 1.7 **提示词是可审计的产物**

`--dump-system-prompt`（`entrypoints/cli.tsx:53`）是一个 ant-only 的 fast path：直接渲染系统 prompt 并打印退出。`getDumpPromptsPath()`（`services/api/dumpPrompts.ts`）把每个请求的真实 prompt 写到本地文件，用于"prompt sensitivity evals"。

整个系统 prompt 通过 `asSystemPrompt([...])` 构造，传入 `[] | [string] | [{type:'text', text:string, cache_control?:...}]`——每一块都是显式的、可索引的、可注入 cache_control 的。

### 1.8 **能力发现是懒加载的**

文件 `tools.ts` 的 50+ 工具里有超过一半通过 `feature()` 守门，并通过 `require()`（而不是 `import`）延迟加载。原因：

- `feature()` 在 bun 编译时已决定，让 `require` 块在外部构建中变成空代码
- 懒 `require` 让冷启动路径不付出它们的解析时间
- `isToolSearchEnabledOptimistic` 在 `optimistic` 阶段先加进工具列表，请求时再 defer，避免运行时反复改 tool schema（改 schema 会让 prompt 缓存失效）

`utils/permissions/permissions.ts` 用 `require('./yoloClassifierPrompts')` 也遵循同样原则。

### 1.9 **每一种可观测性都有专门的产物**

- **对话级别**：`SDKMessage`（流式输出，typed union）
- **请求级别**：`span.model_request_*` 事件 + `NonNullableUsage`（累积的 token 计数）
- **用户级别**：`firstPartyEventLogger`（不包含 PII 的事件流）
- **诊断级别**：`diagLogs`（`logForDiagnosticsNoPII`，结构化日志 + 自定义 channel）
- **性能级别**：`fpsTracker`、`headlessProfiler`、`startupProfiler`、`queryProfiler`——4 个不同粒度的性能采样
- **可重现**：`recordTranscript` + `dumpPrompts` + `getDumpPromptsPath` 一起支撑"给我看到那个具体请求"

`getTelemetryAttributes()`（`utils/telemetryAttributes.ts`）把 OpenTelemetry 风格的 attributes 集中生成，跨多个 sink 复用。

### 1.10 **可移植性是显式可选的**

整个 `vendor/` 目录存的是原始 napi binding 的 shim；`shims/` 下的 6 个包（`@ant/claude-for-chrome-mcp`、`@ant/computer-use-input` 等）是 ant-only 功能的替代实现。`Bun.build` 之外的运行由 `getPlatform()`/`isWindows()`/`setShellIfWindows()` 这些适配器处理。

`native-ts/` 目录里是 Rust 绑定的 TS 包装——Bun binary 中嵌入的 ripgrep 替代品（`bfs`/`ugrep`）走 ARGV0 trick；找不到时降级到 npm 的 `GlobTool`/`GrepTool`。

### 1.11 **安全是多层防御**

权限系统是 *4 层* 防御：

1. **规则匹配**（`shellRuleMatching.ts`）：path/regex/glob 规则白名单/黑名单
2. **bash classifier**（`utils/permissions/yoloClassifier.ts`）：用 LLM 二次判定"这个 Bash 命令是否安全"
3. **自动模式 (auto mode)**：基于 transcript classifier 的"用户意图-工具风险"评估
4. **路径校验**（`utils/permissions/pathValidation.ts`）：防 traversal、防 symlink、防绝对路径逃逸

加上 `add-dir` 白名单目录、`isInProtectedNamespace` 守卫等纵深防御。危险命令（`rm -rf /`、`curl | bash`）由 `dangerousPatterns.ts` 在 prompt 阶段就拒绝。

### 1.12 **错误是 typed，不是字符串**

`utils/errors.ts` 定义 `AbortError`、`APIUserAbortError`、`ImageSizeError`、`ImageResizeError`、`ConfigParseError` 等。每种错误都被各层 catch 后转成对应的 `tool_result` `is_error: true` 块，而不是把异常 message 暴露给模型——避免模型在错误字符串中"看到"被设计为隐藏的细节（如系统路径、凭据）。

`FallbackTriggeredError` 是一种特殊的内部 error：API 触发 fallback 模型时 query loop 抛此异常，外层用 SDK 重试。

---

## 2. 整体架构鸟瞰

```
                       ┌────────────────────────────────────────────────┐
                       │  Bootstrap / Init  (entrypoints/init.ts)        │
                       │  加载 config → 装入 env → 启动 telemetry        │
                       │  加载 GrowthBook → 启动 MDM/Keychain prefetch   │
                       └────────────────────┬───────────────────────────┘
                                            │
                                            ▼
                ┌───────────────────────────────────────────────────────┐
                │  CLI 入口  (entrypoints/cli.tsx → main.tsx)            │
                │  - commander 解析参数                                  │
                │  - fast path: --version / --dump-system-prompt        │
                │  - 决定 mode: REPL / SDK / teleport / assistant       │
                └────────────────────┬──────────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
     ┌────────────────┐     ┌─────────────────┐     ┌────────────────┐
     │ REPL.tsx       │     │ QueryEngine     │     │ SDK entrypoint │
     │ (interactions) │     │ (ask engine)    │     │ (agent SDK)    │
     │ Ink + React    │     │ state machine   │     │ JSON streaming │
     └────────┬───────┘     └────────┬────────┘     └────────┬───────┘
              │                      │                      │
              │  useCanUseTool       │  runTools            │  same engine
              │  PermissionRequest   │  StreamingToolExec   │
              ▼                      ▼                      ▼
                       ┌────────────────────────────┐
                       │  Tools  (50+ tool defs)    │
                       │  Bash / Edit / Agent / MCP │
                       └─────────────┬──────────────┘
                                     │
                                     ▼
                       ┌────────────────────────────┐
                       │  Anthropic Messages API    │
                       │  + Bedrock / Vertex /     │
                       │  Foundry (provider layer)  │
                       └────────────────────────────┘
```

横向看：

```
       Cross-cutting concerns (Layer cake)
  ┌──────────────────────────────────────────────────┐
  │  Analytics  │  GrowthBook  │  OpenTelemetry      │
  ├──────────────────────────────────────────────────┤
  │  feature() gating  │  USER_TYPE  │  env vars      │
  ├──────────────────────────────────────────────────┤
  │  Hooks  (utils/hooks.ts, 87 hooks)               │
  ├──────────────────────────────────────────────────┤
  │  Permissions  (4-layer: rules → classifier → …)  │
  ├──────────────────────────────────────────────────┤
  │  State  (AppState + Zustand-like store)          │
  ├──────────────────────────────────────────────────┤
  │  Persistence  (transcript / file history / task) │
  └──────────────────────────────────────────────────┘
```

---

## 3. 核心模块详解

### 3.1 入口与生命周期：`cli.tsx` / `main.tsx`

**`src/entrypoints/cli.tsx`** 是所有 CLI 路径的"路由表"。它有意识地保持薄：先做 `COREPACK_ENABLE_AUTO_PIN` 环境变量修正、CCR 容器内存调大、然后只对极少量 fast path 走完立即返回（`--version` 零导入、`--dump-system-prompt` 仅几个模块、`--claude-in-chrome-mcp` 仅 1 个 import、`--daemon-worker=…` 由 supervisor 调用）。

其他路径统一进入 `main.tsx` 的"慢路径"。

**`src/main.tsx`**（5,391 行）是真正的入口巨无霸。它的前 21 行就显示了 Anthropic 对启动时间的细致追求：

```ts
// 1. profileCheckpoint marks entry before heavy module evaluation begins
// 2. startMdmRawRead fires MDM subprocesses (plutil/reg query) so they run in
//    parallel with the remaining ~135ms of imports below
// 3. startKeychainPrefetch fires both macOS keychain reads (OAuth + legacy API
//    key) in parallel — isRemoteManagedSettingsEligible() otherwise reads them
//    sequentially via sync spawn inside applySafeConfigEnvironmentVariables()
//    (~65ms on every macOS startup)
profileCheckpoint('main_tsx_entry');
startMdmRawRead();
startKeychainPrefetch();
```

**`init()`** 来自 `entrypoints/init.ts`——用 `memoize` 包裹保证单次执行。它完成：
- 加载 `enableConfigs()`，把 settings 读入全局 config
- 应用 `applySafeConfigEnvironmentVariables`（在信任弹窗之前只注入安全 env）
- 加载 OAuth 账户信息、policy limits、remote managed settings
- 启动 telemetry、设置 OTel meter
- 注册 graceful shutdown handler
- 应用 mTLS 配置、proxy 配置、scratchpad 目录

`main.tsx` 后续负责：
- commander 解析 CLI 参数
- 决定 mode（`isReplModeEnabled` / `isCoordinatorMode` / `isAssistantMode` / `isCCR`）
- 加载内置 skills、plugins、agent definitions
- 加载 MCP 配置 + 预连接 + prefetch official MCP URLs
- 构造 `AppState` 并 `launchRepl`（交互）或 `runQuery`（SDK）

### 3.2 查询引擎：`query.ts` 与 `QueryEngine.ts`

`QueryEngine.ts` 把"会话"建模为一个有状态对象：

```ts
class QueryEngine {
    private config: QueryEngineConfig
    private mutableMessages: Message[]            // 对话历史
    private abortController: AbortController      // 取消信号
    private permissionDenials: SDKPermissionDenial[]
    private totalUsage: NonNullableUsage          // 累积 token
    private readFileState: FileStateCache         // 文件读取缓存（避免重读）
    private discoveredSkillNames = new Set<string>()
    private loadedNestedMemoryPaths = new Set<string>()
}
```

`submitMessage(prompt)` 是核心方法——它是一个 `AsyncGenerator<SDKMessage, void>`，每次新消息进来重新走完整个流程：

1. `setCwd(cwd)` — 切换工作目录
2. `wrappedCanUseTool` — 包一层以追踪 `permissionDenials`
3. `fetchSystemPromptParts()` — 拼装系统 prompt（default + custom + append + memory）
4. `processUserInput` — 处理用户输入（可触发 slash command、文件附件、image 等）
5. `recordTranscript(messages)` — **用户消息一旦 push 进 mutableMessages 就立即写盘**
6. `getSlashCommandToolSkills` + `loadAllPluginsCacheOnly` — 缓存式并发拉取 skills 和 plugins
7. `yield buildSystemInitMessage(...)` — SDK 第一个事件
8. 调用 `query()`（`query.ts`）——真正的 API 调用 + 工具执行循环
9. `flushSessionStorage` — 落盘所有 pending 状态

**`query.ts`** 是 **API 调用 + 工具执行的循环**（2,342 行）。它的关键路径：

```ts
async function* query(...) {
    while (true) {
        // 1. 决定本轮是否触发 compact（microcompact / auto compact）
        if (isAutoCompactEnabled() && tokensNearLimit) {
            compactBoundary = await runAutoCompact(messages, ...)
        }
        // 2. 准备 system prompt（追加 cache 断点）
        // 3. 调用 Anthropic API（流式）
        // 4. 流式消费消息：text / thinking / tool_use
        // 5. 工具执行：runTools() → StreamingToolExecutor
        // 6. 构造 tool_result 块，塞回 messages
        // 7. 决定是否继续：end_turn / tool_use / pause_turn / refusal
    }
}
```

`query.ts` 顶部那一长串 `require()` 块注释里有 **DCE 标签**——每个 feature 都有自己的可选加载点。`HISTORY_SNIP` 启用时加载 `snipCompact.js`+`snipProjection.js`，否则这两个字符串根本不在二进制里。

### 3.3 工具系统：`Tool.ts` / `tools/`

**`Tool.ts`**（792 行）是工具的"接口定义 + builder"。它定义：

- `Tool<Input, Output>` 类型（name / description / inputSchema / call / renderToolUseMessage / renderToolResultMessage / isEnabled / …）
- `buildTool()` 工厂函数 + `toolMatchesName()` 名字匹配（含 MCP `mcp__server__tool` 形式）
- `ToolUseContext` — 工具执行时的上下文（options + abort + state get/set + 各种回调）
- `getEmptyToolPermissionContext` — 默认权限上下文
- `SetToolJSXFn` — 工具主动给 UI 注入 JSX 的钩子
- `CompactProgressEvent` — compact 进度事件

**`tools.ts`** 列出 50+ 工具并按 feature gate 装配。`assembleToolPool()` 是 single source of truth——REPL 模式和 coordinator worker 都通过它合并内置工具 + MCP 工具 + 权限过滤。

**每个工具的实现都遵循同样的样板**（以 BashTool 为例）：

```
tools/BashTool/
├── BashTool.tsx          // 工具定义（call + renderToolUseMessage + renderToolResultMessage + isEnabled）
├── UI.tsx                // 调用过程中的实时 UI（BackgroundHint 等）
├── prompt.ts             // 工具的 prompt 描述（缓存关键）
├── toolName.ts           // 常量 'Bash'
├── bashCommandHelpers.ts // 命令解析（shell-quote）
├── bashPermissions.ts    // 权限决策
├── bashSecurity.ts       // 危险命令检测
├── modeValidation.ts     // 模式校验（plan mode 不允许？）
├── pathValidation.ts     // 路径白名单校验
├── readOnlyValidation.ts // 只读模式校验
├── shouldUseSandbox.ts   // 是否启用 sandbox-runtime
├── destructiveCommandWarning.tsx
└── BashToolResultMessage.tsx
```

工具的"渲染"在两个层面：
- `renderToolUseMessage`（调用方主动控制，返回静态 JSX）
- `renderToolResultMessage`（结果消息）
- `UI.tsx` 中的运行中组件（通过 `SetToolJSXFn` 注入到 PromptInput 旁边）

**`assembleToolPool()` 的精妙注释**值得引用：

```ts
// Sort each partition for prompt-cache stability, keeping built-ins as a
// contiguous prefix. The server's claude_code_system_cache_policy places a
// global cache breakpoint after the last prefix-matched built-in tool; a flat
// sort would interleave MCP tools into built-ins and invalidate all downstream
// cache keys whenever an MCP tool sorts between existing built-ins.
```

**这是 prompt caching 友好的关键**：内置工具前缀稳定，MCP 工具作为后缀追加——下次新装一个 MCP server 不会让已经缓存的 system prompt 失效。

### 3.4 Agent 派发：`tools/AgentTool/`

Agent 工具是 Claude Code 的"递归心脏"——它让模型可以调用自己（带不同身份/工具/权限/上下文）。

`AgentTool.tsx` 顶部有 `lazySchema()` 的 Zod 模式声明：基础字段（`description`/`prompt`/`subagent_type`/`model`/`run_in_background`）+ 多 agent 字段（`name`/`team_name`/`mode`）+ 隔离（`isolation: 'worktree' | 'remote'`）+ `cwd`。

`runAgent.ts` 实现了核心逻辑：
1. 选择模型（`getAgentModel`）
2. 加载 `AgentDefinition`（从 `~/.claude/agents/` 或插件）
3. `initializeAgentMcpServers` —— agent frontmatter 里的 MCP server（区分新建 vs 引用）
4. 构造 `ToolUseContext`（`createSubagentContext` 关键：setAppState 是 no-op，避免污染主 store）
5. **从主 messages 构造 fork 的 messages**（`buildForkedMessages` 截取主对话的 fork point）
6. 调用 `query()` 进入子对话
7. 把子对话的 `tool_use` 块转成 `agentId` trailer
8. 写到 `getTaskOutputPath(agentId)` 文件（`utils/task/diskOutput.ts`）
9. 注册后台任务到 `LocalAgentTask` 或 `RemoteAgentTask`

子 agent 的产出文件路径在 `utils/task/diskOutput.ts`——这是**"长任务流式输出"** 的关键。即使子 agent 还在跑，主 agent 也可以用 `TaskOutputTool` 增量读取它的输出。

**`forkSubagent.ts`** 区分了 fork（继承父上下文 + 仅延续子部分）和一次性调用（独立上下文）。`/fork` 命令由 `feature('FORK_SUBAGENT')` 门控。

**`loadAgentsDir.ts`** 是 agent 定义的加载器——发现 `~/.claude/agents/*.md`、解析 YAML frontmatter、合并插件提供的 agent、按 priority 排序。前端 `built-in/` 目录是 ant-only 预置 agent（generalPurpose、Explore、Plan 等）。

**`agentColorManager.ts`** 给每个 agent 分配稳定的颜色——这是 UI 体验细节：并行多 agent 时用户能凭颜色区分"哪个 agent 在说话"。

### 3.5 权限系统：`utils/permissions/`

权限系统是 Claude Code 最精密的部分之一——它需要同时处理：

- **同步规则**（用户在 settings.json 写的 allow/deny 规则）
- **异步 LLM 判定**（bash classifier / auto mode classifier）
- **可中断**（用户按 Esc 中断确认弹窗）
- **可重放**（重连时恢复 in-flight 询问）
- **可降级**（subagent 没有 UI 时直接拒绝）
- **可审计**（每一次决策都打 telemetry）

**`utils/permissions/permissions.ts`**（1,486 行）实现了 `hasPermissionsToUseTool`——它返回 `PermissionResult`：

```ts
type PermissionResult =
  | { behavior: 'allow';  updatedInput?; decisionReason? }
  | { behavior: 'deny';   message; decisionReason? }
  | { behavior: 'ask' } // 弹窗让用户决定
```

调用链：

```
hasPermissionsToUseTool
├─ 1a. getDenyRuleForTool     ← 黑名单永远拒绝
├─ 1b. getAllowRuleForTool    ← 白名单允许（考虑 add-dir / 文件路径 / 命令白名单）
├─ 1c. (YOLO classifier)      ← 风险评分
├─ 1d. (auto mode classifier) ← 用户意图风险评估
└─ 结果
   ├─ allow   → setYoloClassifierApproval / logDecision / resolve(allow)
   ├─ deny    → recordAutoModeDenial / logDecision / resolve(deny)
   └─ ask     → handleInteractivePermission (REPL)
                          或 handleCoordinatorPermission (coordinator)
                          或 handleSwarmWorkerPermission (swarm worker)
```

**`hooks/useCanUseTool.tsx`** 是 React 层入口——它构造 `PermissionContext`、包装 hook、调用 `hasPermissionsToUseTool`、根据结果分发到三个 handler（REPL / coordinator / swarm worker）。

**`hooks/toolPermission/`** 子目录有 4 个文件做具体实现：
- `PermissionContext.ts` —— 创建上下文、buildAllow/buildDeny、queue ops
- `handlers/interactiveHandler.ts` —— REPL 弹窗
- `handlers/coordinatorHandler.ts` —— coordinator 的 worker（无 UI）
- `handlers/swarmWorkerHandler.ts` —— 群组 worker
- `permissionLogging.ts` —— 决策日志

### 3.6 UI 渲染：Ink + React + 148 个组件

Claude Code 的 UI 是 **Ink + React 18** 跑的终端 React 渲染。`src/ink/` 是一个**完整自定义的 React reconciler**（reconciler.ts、renderer.ts、layout/、events/、termio/）——因为标准 React-DOM 渲染不了 ANSI 转义序列。

**`ink.ts`** 重新导出 `Box`/`Text` 等基础组件，并自动套上 `ThemeProvider`（主题感知——ThemedBox/ThemedText），这样所有 `ink()` 调用点不需要手动套主题。

`App.tsx` 树形结构：

```
App
├─ AppStateProvider            ← 全局状态
│  └─ StatsProvider             ← 性能采样
│     └─ FpsMetricsProvider     ← FPS 采样
│        └─ BootstrapBoundary   ← 错误边界
│           └─ <REPL />          ← 真正的 UI 根
```

**REPL.tsx** 是状态机的"渲染"层：把 AppState + 当前消息 + 任务状态画成终端字符。它用 `useSyncExternalStore` 订阅 AppState 变更。

**组件分层**（典型 148 个）：

- **基础**：`Box`、`Text`、`Spinner`、`Link`、`Button`
- **消息**：`Messages`、`MessageRow`、`MessageResponse`、`MessageSelector`、`VirtualMessageList`
- **输入**：`PromptInput`、`BaseTextInput`、`TextInput`、`VimTextInput`、`SearchBox`
- **工具**：`FileEditToolDiff`、`BashToolResultMessage`、`MCPServerApprovalDialog`、`FileEditToolUseRejectedMessage`
- **权限**：`PermissionRequest`、`BypassPermissionsModeDialog`、`ClaudeMdExternalIncludesDialog`
- **任务**：`TaskListV2`、`AgentProgressLine`、`CoordinatorAgentStatus`、`TeammateViewHeader`
- **状态**：`Spinner`、`StatusLine`、`Stats`、`CostThresholdDialog`、`TokenWarning`、`MemoryUsageIndicator`
- **对话框**：`ModelPicker`、`ThemePicker`、`OutputStylePicker`、`LanguagePicker`、`LogSelector`
- **IDE**：`IdeStatusIndicator`、`IdeOnboardingDialog`、`ShowInIDEPrompt`

**关键模式：工具驱动的 JSX 注入**

`SetToolJSXFn` 是工具在执行时主动渲染 JSX 的能力。例如 BashTool 跑 30 秒时可以在 PromptInput 旁边显示进度条：

```ts
toolUseContext.setToolJSX?.({
    jsx: <BashModeProgress ... />,
    shouldHidePromptInput: false,
    showSpinner: true,
})
```

这是 Ink 的核心扩展能力——传统 React 假设组件树稳定，而 Claude Code 允许工具动态往 UI 里塞东西。

### 3.7 状态管理：`state/AppState.tsx` 与 `state/store.ts`

**`state/store.ts`**（仅 34 行）实现一个轻量 store：

```ts
export type Store<T> = {
    getState(): T
    setState(updater: (prev: T) => T): void
    subscribe(listener: () => void): () => void
}
```

**`state/AppStateStore.ts`**（569 行）定义 `AppState` 类型——它是 **DeepImmutable** 的"上帝对象"，包含 80+ 字段：

- `settings: SettingsJson` —— 用户设置
- `toolPermissionContext: ToolPermissionContext` —— 权限上下文
- `mainLoopModel: ModelSetting` —— 当前模型
- `kairosEnabled: boolean` —— KAIROS 模式开关
- `remoteSessionUrl: string` —— 远程会话 URL
- `mcp: { tools, resources, clients }` —— MCP 状态
- `tasks: TaskState[]` —— 任务状态
- `agents: AgentDefinitionsResult` —— agent 定义
- `todos: TodoList` —— TODO
- `fileHistory: FileHistoryState` —— 文件历史
- `koPromotions: ...`、`microcompactBoundary: ...`、`speculation: SpeculationState` —— 各种"边界状态"

**`AppStateProvider`** 用 React Context 暴露 store，并提供 `useAppState`/`useSetAppState` hooks。

**关键创新：`SpeculationState`**（`AppStateStore.ts:52-78`）——基于用户输入**预测**下一个命令，提前渲染 UI（节省 ~150ms 感知延迟）。当用户敲下 Enter 时，"预测"已经渲染好了。

**子 agent 隔离**：

```ts
// utils/forkedAgent.ts
export function createSubagentContext(parent: ToolUseContext): ToolUseContext {
    return {
        ...parent,
        setAppState: () => {},  // ← no-op! 子 agent 看不到主 store
        setAppStateForTasks: parent.setAppStateForTasks,  // ← 但后台任务仍写主 store
    }
}
```

子 agent 看到的是父 store 的快照，但它的任何 setAppState 不会污染父——这是子 agent 隔离的关键。

### 3.8 会话持久化与恢复

**`utils/sessionStorage.ts`**（5,105 行）实现了完整的状态持久化：

- `recordTranscript(messages)` —— 写入 JSONL transcript
- `recordSidechainTranscript(agentId, messages)` —— 子 agent transcript
- `writeAgentMetadata` —— agent metadata
- `clearAgentTranscriptSubdir` —— 清理

**Transcript 写入策略**（`QueryEngine.ts:444-467` 注释）：

> The for-await below only calls recordTranscript when ask() yields an assistant/user/compact_boundary message — which doesn't happen until the API responds. If the process is killed before that (e.g. user clicks Stop in cowork seconds after send), the transcript is left with only queue-operation entries; getLastSessionLog filters those out, returns null, and --resume fails with "No conversation found".

**对话完整性高于 throughput**。`--bare` 模式可以 fire-and-forget（脚本不 --resume），交互模式必须 await 完。

**`utils/conversationRecovery.ts`** 实现 `--resume` —— 读 transcript、构造 `AppState`、恢复消息历史、恢复文件缓存、恢复后台任务。

**`utils/fileHistory.ts`** 实现 **文件级快照**——每次 FileEdit/FileWrite 前 `fileHistoryMakeSnapshot` 写入 `<session>/file-history/<hash>.json`。`/rewind` 命令可以回滚到任意快照。

### 3.9 上下文压缩：`services/compact/`

Claude Code 有 **4 层** 上下文压缩策略：

1. **microCompact**（`microCompact.ts`）——每次 turn 后增量清理大 tool result（保 < N 行）
2. **apiMicrocompact**（`apiMicrocompact.ts`）——让 API 端做 microcompact（更省 token）
3. **autoCompact**（`autoCompact.ts`）——在 token 上限前 5% 触发完整压缩
4. **reactiveCompact**（`reactiveCompact.ts`，`feature('REACTIVE_COMPACT')`）——响应式：模型调用了大输出工具后立即触发
5. **snipCompact**（`snipCompact.ts`，`feature('HISTORY_SNIP')`）——ant-only：硬截断老历史
6. **sessionMemoryCompact**（`sessionMemoryCompact.ts`）——把老对话转成 session memory 文件
7. **timeBasedMCConfig**（`timeBasedMCConfig.ts`）——按时间窗口的 microcompact 配置

`compact.ts`（1,705 行）实现主路径——把整段对话发给模型生成 summary，构造 `compact_boundary` 系统消息。

`compactWarningHook.ts` + `compactWarningState.ts` —— UI 在 token 上限接近时弹出警告。

### 3.10 钩子系统：`utils/hooks.ts`（5,297 行）

钩子系统让用户在 `settings.json` 注册外部命令——Claude Code 在以下事件触发时调用这些命令：

- `PreToolUse` / `PostToolUse` —— 工具调用前后
- `PreCompact` / `PostCompact` —— 压缩前后
- `SessionStart` / `SessionEnd` —— 会话开始/结束
- `Notification` —— 通知事件
- `Stop` / `SubagentStop` —— 停止事件

`utils/hooks/AsyncHookRegistry.ts` 维护异步钩子队列——`utils/hooks/execAgentHook.ts` / `execHttpHook.ts` / `execPromptHook.ts` 分别处理 agent、HTTP、prompt 三种钩子类型。

`utils/hooks/sessionHooks.ts` —— 会话级钩子（持续 7 天）。

`utils/hooks/postSamplingHooks.ts` —— 采样后钩子（每条 assistant 消息后）。

`utils/hooks/ssrfGuard.ts` —— HTTP 钩子的 SSRF 防护。

`registerFrontmatterHooks.ts` —— 从 agent frontmatter 注册的钩子。

`utils/hooks/skillImprovement.ts` —— 技能改进钩子（自动改写 skill）。

**87 个 hook 文件**（`src/hooks/`）——React 层的 hook（`use*` 前缀）独立于这个磁盘钩子系统；不要混淆。

### 3.11 MCP 集成：`services/mcp/`

`services/mcp/client.ts`（3,348 行）实现 MCP 协议——JSON-RPC 2.0 over stdio / SSE / HTTP。

**`MCPConnectionManager.tsx`**（仅 72 行但承载 React UI）——管理连接生命周期。

**`types.ts`** 定义 `MCPServerConnection` 状态机：
- `pending` → `connected` → `failed` / `disabled`
- `auth` 子状态机（OAuth、API key、env var）

**MCP 配置层级**（`config.ts`）：
- 用户级 `~/.claude/mcp.json`
- 项目级 `.mcp.json`
- 内置 server（`claudeai`、`chrome`、`atlas`）
- 插件提供
- 远程 managed settings 注入

**OAuth 流程**（`oauthPort.ts` + `xaaIdpLogin.ts`）：
1. Claude Code 启动 HTTP server 在随机端口
2. 浏览器重定向用户到 MCP server 的 OAuth 页
3. server 重定向回 `http://localhost:<port>/callback?code=...`
4. 交换 code → access token + refresh token
5. 存储到 keychain

**MCP 工具通过 `assembleToolPool()` 注入到工具列表**——但 `assembleToolPool` 的注释解释了 *为什么* 放在 built-in 之后：保护 system prompt 缓存。

**`elicitationHandler.ts`** —— MCP elicitation 协议支持（让 server 反向问用户问题）。`-32042` 错误码触发 URL elicitation。

**`channelAllowlist.ts` / `channelNotification.ts` / `channelPermissions.ts`** —— MCP channel 机制（server 主动推送通知到 client）。

### 3.12 任务系统：`tasks/`

`Task.ts`（125 行）定义 7 种任务类型：

```ts
type TaskType =
  | 'local_bash'        // b  // 本地 shell
  | 'local_agent'       // a  // 本地子 agent
  | 'remote_agent'      // r  // 远程 agent (CCR)
  | 'in_process_teammate' // t // 同进程 teammate
  | 'local_workflow'    // w  // workflow 脚本
  | 'monitor_mcp'       // m  // MCP 监视
  | 'dream'             // d  // KAIROS 自动做梦
```

**任务 ID 用 36 进制 8 位随机**（`Task.ts:96-106`）：

> 36^8 ≈ 2.8 trillion combinations, sufficient to resist brute-force symlink attacks.

任务 ID 是文件路径的一部分（输出文件名 `<id>.txt`）——防 symlink attack。

**`tasks/LocalAgentTask/`** 实现本地 agent 任务——负责 progress tracking、completion notification、summary generation。

**`tasks/RemoteAgentTask/`** 实现 CCR 容器中的远程 agent。

**`tasks/LocalShellTask/`** 实现后台 bash。

**`tasks/InProcessTeammateTask/`** —— 同进程 teammate（多 agent 协作模式）。

**`tasks/LocalWorkflowTask/`** —— workflow 脚本执行。

**`tasks/MonitorMcpTask/`** —— MCP 监视（轮询 server 状态）。

**`tasks/LocalMainSessionTask.ts`** —— 主会话自身也作为任务管理。

`tasks/types.ts` 定义 `TaskState` ——状态机：`pending → running → completed/failed/killed`，可暂停、可恢复、可通知。

**`tasks/pillLabel.ts`** —— UI 底部的状态 pill（"3 running, 1 failed"）。

---

## 4. 横切关注点

### 4.1 三层功能门控

```ts
// 第一层：编译时（bun:bundle 静态分析）
import { feature } from 'bun:bundle'
if (feature('BUDDY')) { /* ... */ }
// 外部构建里 feature() 编译为 false，整块代码被 DCE

// 第二层：用户类型
process.env.USER_TYPE === 'ant'
// 'ant' = Anthropic 内部，'external' = 默认

// 第三层：远程 A/B 实验
import { getFeatureValue_CACHED_MAY_BE_STALE } from './services/analytics/growthbook.js'
const kairosEnabled = getFeatureValue_CACHED_MAY_BE_STALE('tengu_kairos', false)
```

三层门控互相正交。`docs/07-feature-gates.md` 列出了 50+ 编译开关的完整清单。

### 4.2 性能与启动优化

启动时间是 Claude Code 的关键 UX 指标——`startupProfiler` 在每个关键点打点：

- `main_tsx_entry` —— 进程启动
- `cli_entry` —— cli.tsx 入口
- `init_function_start` —— init() 开始
- `init_configs_enabled` —— 配置加载完
- `before_getSystemPrompt` / `after_getSystemPrompt` —— system prompt 准备
- `before_skills_plugins` / `after_skills_plugins` —— skills/plugins 缓存加载
- `system_message_yielded` —— SDK 第一条事件
- `before_skills_plugins` 等

**优化手段**：
- `startMdmRawRead()` + `startKeychainPrefetch()` 并行 spawn 子进程
- OpenTelemetry 懒加载（`init.ts:45-46` 注释：~400KB 延迟到 telemetry 实际初始化）
- gRPC exporter 进一步懒加载
- 工具懒 require（`tools.ts` 大量 `require`）
- 启动早期 fast path（`--version` 零导入）

### 4.3 提示词缓存策略

`docs/16-claude-api.md` 已经详细讨论了 prompt caching。这里补充几个 Claude Code 特有的设计：

1. **`getAllBaseTools()` 必须稳定**——注释里写着 "MUST stay in sync with Statsig"——这意味着如果 Anthropic 想给外部用户加工具，必须同步更新 cloud-side 的缓存策略
2. **`assembleToolPool` 的排序**——built-in 工具按 `localeCompare` 排序后压平；MCP 工具追加其后
3. **MCP 工具排序**——同一 MCP server 的工具也按 `localeCompare` 排序；新装 server 不会插入到 built-in 之间
4. **`getUserContext` memoize**——只算一次（不依赖 per-request 的 cwd）
5. **`feature('BREAK_CACHE_COMMAND')`** —— 内部开关，强制给 system prompt 注入 `[CACHE_BREAKER: <random>]` 字符串，强制下次请求重新缓存

### 4.4 可观测性 / 遥测 / 增长实验

- **First-party event logger**（`services/analytics/firstPartyEventLogger.ts`）—— 不含 PII 的事件流
- **Datadog**（`datadog.ts`） —— 内部指标
- **OpenTelemetry** —— 跨语言可观测性
- **GrowthBook** —— 远程配置 + A/B 实验
- **Perfetto**（`utils/telemetry/perfettoTracing.ts`） —— 性能追踪可视化
- **`logForDiagnosticsNoPII`** —— 诊断日志（结构化 + 自定义 channel）
- **`firstPartyEventLoggingExporter`** —— 批量导出

**`growthbook.ts`** 顶部注释暗示了 GrowthBook 的复杂度——`isEqual`/`memoize`/`signal` 等等。每个 fetch 都带 `GrowthBook-Client-Key`、`lastSeen` 等头。

### 4.5 错误与降级

每个错误类型都有明确的处理路径：
- `AbortError` —— 用户中断
- `APIUserAbortError` —— SDK 中断
- `ImageSizeError` / `ImageResizeError` —— 图像处理
- `ConfigParseError` —— 配置错误
- `FallbackTriggeredError` —— API 触发 fallback 模型
- `PROMPT_TOO_LONG_ERROR_MESSAGE` —— prompt 超长

每一类都被包成 typed exception，外层用 `categorizeRetryableAPIError` 分类是否可重试。

---

## 5. 多模式架构

Claude Code **不是一个 CLI**——它是一个 **可执行各种 agent 工作负载的通用宿主**。不同 mode 共享同一个 query engine + tools，但 UI、权限、协作模式不同。

| Mode | Feature Gate | 入口 | 用途 |
|---|---|---|---|
| **REPL** | (默认) | `launchRepl` | 交互式终端 |
| **SDK** | (默认) | `entrypoints/sdk/` | JSON 流式 API 给 agent SDK |
| **CCR Remote** | `CCR_REMOTE_SETUP` | `--remote` | 远程会话 |
| **Coordinator** | `COORDINATOR_MODE` | `CLAUDE_CODE_COORDINATOR_MODE` | 多 agent 协作（主 = 调度，worker = 执行） |
| **Assistant** (KAIROS) | `KAIROS` | `assistant: true` | 持久助手 |
| **Brief** | `KAIROS_BRIEF` | `/brief` | 简报模式 |
| **Proactive** | `PROACTIVE` | `/proactive` | 主动模式（没活找活） |
| **Bridge** | `BRIDGE_MODE` | `bridge/` | 远程控制（被 claude.ai / 手机控制） |
| **Voice** | `VOICE_MODE` | `voice/` | 语音模式 |
| **Daemon** | `DAEMON` | `--daemon-worker` | 守护进程 |
| **REPL (sandbox)** | `REPL_TOOL_NAME` | `tools/REPLTool/` | REPL 工具内嵌（ant-only） |
| **Ultraplan** | `ULTRAPLAN` | `utils/ultraplan/` | 云端深度规划 |
| **Telnet/Teleport** | (ant-only) | `utils/teleport.tsx` | 会话传送 |
| **UDS Inbox** | `UDS_INBOX` | (ant-only) | Unix socket 收件箱 |
| **Workflow** | `WORKFLOW_SCRIPTS` | `tools/WorkflowTool/` | 工作流脚本 |
| **MCP servers** | `CHICAGO_MCP` | `--computer-use-mcp` | Claude 内嵌为 MCP server |

每种 mode 都对应一个 feature flag 或 env var；启动时根据 flag + 命令行 + settings 决定进入哪种 mode。所有 mode 共享 `QueryEngine`——这是核心抽象。

---

## 6. 关键设计模式总结

### 6.1 **Builder 模式（Tool）**

```ts
export const BashTool: Tool = buildTool({
    name: BASH_TOOL_NAME,
    description: async () => '...',
    inputSchema: lazySchema(() => z.object({...})),
    call: async (input, ctx) => { ... },
    renderToolUseMessage: (input) => { ... },
    renderToolResultMessage: (output) => { ... },
    isEnabled: () => true,
    isReadOnly: () => false,
    // ...
})
```

### 6.2 **策略模式（Permission Handlers）**

`useCanUseTool` 根据 mode 分发到 `interactiveHandler` / `coordinatorHandler` / `swarmWorkerHandler`。

### 6.3 **状态机（AppState 字段）**

`SpeculationState`、`TaskStatus`、`MCPServerConnection.type` 等都是显式 discriminated union。

### 6.4 **观察者（store.subscribe）**

`state/store.ts` 是轻量 pub-sub；`useSyncExternalStore` 订阅。

### 6.5 **上下文对象（ToolUseContext）**

工具的 `call(input, ctx)` 第二个参数是 `ToolUseContext`——避免全局状态，又显式传递所有依赖。

### 6.6 **懒加载 + 静态消除（feature() + require）**

- 编译时 `feature('BUDDY')` 决定代码是否进入二进制
- 运行时 `require` 让冷启动路径不付出未启用工具的解析时间

### 6.7 **缓存键稳定性（localeCompare + 排序）**

`assembleToolPool` 的排序是 cache-friendly design 的范本。

### 6.8 **错峰并行（prefetch + lazy import）**

`startMdmRawRead` + `startKeychainPrefetch` 在 main 入口处就 spawn 子进程，让 ~135ms 后续 import 与它们并行。

### 6.9 **Typed Union 优于字符串标签**

`Message` 是 `AssistantMessage | UserMessage | SystemMessage | ...` —— TypeScript discriminated union；每种消息有完整类型字段。

### 6.10 **Single Source of Truth 模式**

- `assembleToolPool` —— 工具池唯一来源
- `getAllBaseTools` —— 内置工具唯一来源
- `getSystemPrompt` —— 系统 prompt 唯一来源
- `fetchSystemPromptParts` —— 用户/系统上下文唯一来源

每处"我想知道有哪些工具 / 权限是什么 / prompt 长啥样"都从这同一个函数拿，避免多份定义不一致。

---

## 7. 总结：这套架构给我们什么启示

读完 1,987 个 TypeScript 文件，可以总结出 **5 条跨工程的启示**：

### 7.1 **把"工具调用"当作 first-class 概念**

不要让模型只能发 Bash 字符串——给每个常见动作一个工具，给工具一个 `description` + `input_schema`。这换来：UI 渲染、权限拦截、并行执行、可审计的输入。

### 7.2 **Agent 系统本身就是工程难点**

> model ↔ model 的对话状态如何隔离？父子消息如何 fork？远程/本地如何无缝切换？

Claude Code 用 `createSubagentContext(setAppState: no-op)` 解决了子 agent 隔离；用 `setAppStateForTasks` 解决了"长寿命任务需要写回主 store"；用 `recordSidechainTranscript` 解决了"子对话的 transcript 独立"。**这些不是问题，是设计**。

### 7.3 **Prompt caching 是金钱问题**

每次 cache miss 多花 ~1.25× 输入成本；30+ 次工具调用 / turn 的 agent 任务里，cache hit 率从 50% 提升到 90% 直接影响利润率。Claude Code 把"工具列表稳定排序"作为 first-class concern——`getAllBaseTools` 注释里甚至写 "MUST stay in sync with Statsig"。

### 7.4 **三层门控 > 单一开关**

```
编译时 (build size)  ←——→  用户类型 (ant vs external)  ←——→  GrowthBook (A/B)
```

每一层都解决不同问题。混在一起要么泄露内部功能给外部用户，要么代码体积膨胀。

### 7.5 **"业务逻辑与表现分离"是 agent 系统的关键**

`QueryEngine` 处理状态；`REPL.tsx` 处理表现；SDK 用同一份引擎但完全跳开表现。这让 Claude Code 同时服务交互用户、SDK 集成、远程控制、CCR 容器——**一份引擎，四种体验**。

---

**最后**：这份架构的核心不是 1,987 个文件，而是 **"一个持久的、跨进程的、可递归的 agent 状态机"** 的精确工程化。Claude Code 之所以能成为"AI 工程师的最佳工具"，不是因为它有什么神奇的技术，而是因为 Anthropic 把 *每一个* 状态转移都打点、*每一次* 决策都审计、*每一处* 缓存键都优化——这才是工程的可敬之处。

---

## 附录：源码目录速查

```
src/
├── entrypoints/         # 8 个进程入口
│   ├── cli.tsx          #   顶层路由
│   ├── init.ts          #   启动初始化
│   ├── sdk/             #   Claude Agent SDK 接口
│   ├── agentSdkTypes.ts #   SDK 共享类型
│   └── sandboxTypes.ts  #   沙箱类型
├── QueryEngine.ts       # 会话状态机
├── query.ts             # API + 工具执行循环
├── Tool.ts              # 工具抽象 + builder
├── tools.ts             # 50+ 工具注册
├── tools/               # 53 个工具实现
│   ├── AgentTool/       #   Agent 派发（核心）
│   ├── BashTool/        #   Bash 工具（最复杂）
│   ├── FileEditTool/    #   文件编辑
│   ├── FileReadTool/    #   文件读取
│   ├── FileWriteTool/   #   文件写入
│   ├── SkillTool/       #   Skill 加载
│   ├── ListMcpResourcesTool/ # MCP 资源
│   ├── ScheduleCronTool/     # 定时任务
│   ├── WorkflowTool/         # 工作流脚本
│   ├── SyntheticOutputTool/  # 结构化输出
│   ├── SnipTool/             # 历史截断
│   └── ...                  # 其他 40+ 工具
├── commands.ts          # 87 个斜杠命令注册
├── commands/            # 命令实现
├── components/          # 148 个 React UI 组件
│   ├── App.tsx
│   ├── design-system/   #   主题与基础组件
│   ├── messages/        #   消息渲染
│   ├── permissions/     #   权限 UI
│   ├── tasks/           #   任务 UI
│   ├── teams/           #   群组 UI
│   └── ...
├── hooks/               # 87 个 React hooks
│   ├── useCanUseTool.tsx
│   ├── toolPermission/
│   └── ...
├── ink/                 # 自定义 React reconciler（终端）
├── state/               # 全局状态管理
│   ├── store.ts         #   极简 store
│   ├── AppStateStore.ts #   AppState 类型
│   └── AppState.tsx     #   Provider/Consumer
├── services/            # 业务服务
│   ├── api/             #   Anthropic API 客户端
│   ├── mcp/             #   MCP 集成
│   ├── compact/         #   上下文压缩
│   ├── analytics/       #   遥测 + GrowthBook
│   ├── oauth/           #   OAuth 流程
│   ├── plugins/         #   插件加载
│   ├── remoteManagedSettings/ # 远程管理设置
│   ├── lsp/             #   LSP 客户端
│   └── ...
├── tasks/               # 后台任务管理（7 种类型）
│   ├── LocalAgentTask/
│   ├── RemoteAgentTask/
│   ├── LocalShellTask/
│   ├── InProcessTeammateTask/
│   ├── LocalWorkflowTask/
│   ├── MonitorMcpTask/
│   └── DreamTask/
├── utils/               # 333 个工具函数
│   ├── permissions/     #   权限系统（核心）
│   ├── hooks.ts         #   磁盘钩子（5,297 行）
│   ├── compact/         #   压缩策略
│   ├── model/           #   模型管理
│   ├── settings/        #   配置
│   ├── sessionStorage.ts #  会话持久化（5,105 行）
│   ├── telemetry/       #   可观测性
│   ├── proactive/       #   主动模式
│   └── ...
├── types/               # 共享类型
├── schemas/             # Zod schema
├── utils/               # 工具函数
│
├── 横向特性目录（feature gate 控）:
├── buddy/               # BUDDY 电子宠物
├── assistant/           # KAIROS 助手模式
├── coordinator/         # 多 Agent 编排
├── bridge/              # 远程控制桥接（31 文件）
├── proactive/           # 主动模式
├── voice/               # 语音模式
├── vim/                 # Vim 模式
├── outputStyles/        # 输出样式
├── moreright/           # （探索中）
├── memdir/              # Memory 目录
├── skills/              # Skill 加载
│   └── bundled/         #   内置 Skills
└── plugins/             # 插件加载
    └── bundled/         #   内置插件
```

**总计：1,987 个 TypeScript / TSX 文件，~230k 行代码。**

---

> **下一步阅读建议**：
> - 想深入工具调用 → `docs/11-tool-execution.md`
> - 想理解 Agent 派发 → `docs/12-run-agent.md`
> - 想理解钩子机制 → `docs/15-utils-hooks.md`
> - 想理解 Anthropic API 集成 → `docs/16-claude-api.md`
> - 想看隐藏功能 → `docs/05-hidden-commands.md` + `docs/07-feature-gates.md`
