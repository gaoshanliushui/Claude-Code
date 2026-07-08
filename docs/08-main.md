# MAIN — 入口编排

> 源码位置：`src/main.tsx`（5213 行）+ `src/entrypoints/cli.tsx`
> 入口函数：`main()` → `run()` → `program.parseAsync()`
> 阶段命名风格：文件内部将 action 拆分为 `阶段 0 ~ 阶段 16` 共 17 个子函数
> 关联文档：[01-buddy.md](other/01-buddy.md) · [06-bridge.md](other/06-bridge.md) · [07-feature-gates.md](other/07-feature-gates.md)

`src/main.tsx` 是 Claude Code 整个 CLI 的中枢，负责把分散在 `src/` 各个子目录的子系统（设置、模型、Agent、Tool、MCP、Hooks、Telemetry、REPL、Print 模式等）按 commander 的命令流编织成一个统一的启动流水线。它的核心设计取舍只有一条：**`ActionContext` 共享可变状态 + 分阶段函数**，避免在单个 2000+ 行的 action 回调里堆 `let` 局部变量。

---

## 一、整体职责

| 模块 | 作用 |
|------|------|
| 顶层副作用 | 性能埋点、MDM 读取、keychain 预取、Node/Bun 调试检测、Windows PATH 防护、settings 早期加载 |
| 状态容器 | `ActionContext` 类型——唯一承载 16 个阶段之间共享的可变状态 |
| 模式分发 | 交互模式（REPL）vs Headless 模式（`-p`/`--print`）vs 子命令模式（`mcp`、`auth`、`plugin` 等） |
| 命令编排 | 通过 `@commander-js/extra-typings` 注册顶层命令、50+ 选项、12+ 顶级 option 组 |
| 初始化流水线 | `preAction` → 9 个 action 阶段 → 头less 或交互式派发 |
| 启动优化 | `profileCheckpoint` 标记、`--bare` 极简模式、并行 setup、缓存文件路径（避免破坏 API prompt cache） |

---

## 二、文件结构

```
src/main.tsx (5213 lines)
├── 顶层副作用 (L1-271)
│   ├── profileCheckpoint('main_tsx_entry')        入口埋点
│   ├── startMdmRawRead()                          并行 MDM 读取
│   ├── startKeychainPrefetch()                    并行 keychain 预取
│   ├── 同步 import ~80 个模块
│   ├── 条件 require: KAIROS / COORDINATOR_MODE / TRANSCRIPT_CLASSIFIER
│   ├── migrateBypassPermissionsAcceptedToSettings 等 14 个 migration
│   └── isBeingDebugged() → process.exit(1)         Node/Bun 调试短路
│
├── 顶层辅助函数 (L216-540)
│   ├── logManagedSettings()                       报告 policy settings
│   ├── logSessionTelemetry() / logStartupTelemetry()  Statsig 事件
│   ├── runMigrations()                            CURRENT_MIGRATION_VERSION = 11
│   ├── startDeferredPrefetches()                  first render 之后的预取
│   ├── eagerLoadSettings() / loadSettingsFromFlag  --settings 早期加载
│   ├── initializeEntrypoint()                     CLAUDE_CODE_ENTRYPOINT
│   └── _pendingConnect / _pendingAssistantChat / _pendingSSH   early argv stash
│
├── ActionContext 类型 + createActionContext (L542-841)   状态容器
│
├── 阶段 0：runPreActionHookInit (L849-909)        commander hook
├── 阶段 1：normalizeActionEntry (L917-997)        --bare / "code" prompt / KAIROS
├── 阶段 2：extractActionOptions (L1005-1230)      解析 commander options
├── 阶段 3：resolveSystemPrompts (L1232-1321)      system prompt 处理
├── 阶段 3b：resolvePermissionMode (L1329-1354)    权限模式解析
├── 阶段 4：parseDynamicMcpConfig (L1362-1473)     --mcp-config
├── 阶段 4b：applyClaudeInChromeMcp (L1479-)       Chrome / Chicago MCP
├── 阶段 5：parseAndSetChannels (L1500-)           KAIROS channels
├── 阶段 6：initializeBriefAndToolPermissions (L1550-)  tool permission context
├── 阶段 7：kickoffBackgroundMcpAndValidateFormats (L1600-)  后台 MCP
├── 阶段 8：resolveInputPromptAndLoadTools (L1847-)  stdin / tools
├── 阶段 9：runParallelSetup (L1900-)             setup() ∥ getCommands ∥ getAgents
├── 阶段 10：resolveModelAgentAndSystemPrompts (L2001-)  model + agent + system prompt
├── 阶段 11：runInteractiveSetupScreens (L2242-)  Ink mount + showSetupScreens
├── 阶段 12：runPostTrustInitialization (L2350-)  LSP / MCP / hooks / thinking
├── 阶段 13：runHeadlessMode (L2641-)             -p 模式
├── 阶段 14：assembleInteractiveInitialState (L2923-)  AppState 装配
├── 阶段 15：dispatchReplLaunch (L3181-)          REPL 启动路由
├── 阶段 16：registerSubcommandTree (L3942-4552)  所有子命令
│
├── main() (L4554-4831)                           公开入口
└── run() (L4859-5042)                            commander 编排器
```

> 行号会随着代码改动漂移，但阶段序号和命名是稳定锚点。

---

## 三、顶层副作用（文件加载即执行）

模块被 `import` 时立刻发生以下事情。设计意图是**让所有不依赖业务逻辑的"重型"操作与 ESM 模块求值并行**。

| 行 | 副作用 | 节省时间 |
|----|--------|---------|
| 12 | `profileCheckpoint('main_tsx_entry')` | 启动性能基线 |
| 16 | `startMdmRawRead()` — 并行 spawn `plutil`/`reg query` | 与剩余 ~135ms imports 并行 |
| 20 | `startKeychainPrefetch()` — 并行读取 macOS OAuth + legacy API key | 节省 ~65ms（避免 `applySafeConfigEnvironmentVariables` 内同步读取） |
| 209 | `profileCheckpoint('main_tsx_imports_loaded')` | import 阶段结束标记 |
| 266 | `isBeingDebugged()` → `process.exit(1)` | 阻断 Node 调试模式下的开发流程 |

### 3.1 Windows PATH 防护

```ts
process.env.NoDefaultCurrentDirectoryInExePath = '1';
```

必须在**任何**命令执行之前设置，防止 Windows 从当前目录加载可执行文件（PATH hijacking）。注释明确引用了微软文档。

### 3.2 调试短路

```ts
if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

通过三路检测（`process.execArgv`、`NODE_OPTIONS`、`inspector.url()`）识别 Node/Bun 调试模式。Ant 内部构建保留调试能力。

### 3.3 KAIROS / COORDINATOR_MODE 条件 require

```ts
const coordinatorModeModule = feature('COORDINATOR_MODE') ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS') ? require('./assistant/index.js') : null;
const kairosGate = feature('KAIROS') ? require('./assistant/gate.js') : null;
```

使用 `feature('XXX')` 编译期判断 + `require` 而非 `import`，实现 **dead code elimination**。这是处理多 SKU（external / ant）共享代码库的标配。

### 3.4 早期 argv 暂存

```ts
const _pendingConnect: PendingConnect | undefined = feature('DIRECT_CONNECT') ? {...} : undefined;
const _pendingAssistantChat: PendingAssistantChat | undefined = feature('KAIROS') ? {...} : undefined;
const _pendingSSH: PendingSSH | undefined = feature('SSH_REMOTE') ? {...} : undefined;
```

这些全局 stash 在 `main()` 阶段被写入，作用是在 commander 接管 argv 之前保留 `cc://`、`assistant [sessionId]`、`ssh <host>` 等特殊参数。`run()` 之后会基于 `_pending*` 决定 REPL 走哪条分支（见阶段 15）。

---

## 四、ActionContext — 共享可变状态容器

```ts
type ActionContext = {
  // 0. commander 入参
  options: any; prompt: string | undefined;

  // 1. 早期归一化
  kairosEnabled: boolean;
  assistantTeamContext: Awaited<ReturnType<typeof initializeAssistantTeam>> | undefined;

  // 2. option 解析（worktree、tmux、teammate、SDK、teleport、remote、RC...）
  worktreeName, worktreePRNumber, worktreeEnabled, tmuxEnabled;
  storedTeammateOpts, sdkUrl, teleport, remote, remoteControl, ...;

  // 3. prompt / 权限
  systemPrompt, appendSystemPrompt, permissionMode, permissionModeNotification;

  // 4. MCP
  dynamicMcpConfig, strictMcpConfig, mcpConfigPromise, devChannels, enableClaudeInChrome;

  // 5. tool 权限 + 输入
  toolPermissionContext, warnings, inputPrompt, effectivePrompt, tools, jsonSchema;

  // 6. model / agent / advisor
  userSpecifiedModel, effectiveModel, resolvedInitialModel, advisorModel;
  allAgents, cliAgents, agentDefinitions, mainThreadAgentDefinition;

  // 7. 并行 setup
  setupPromise, commandsPromise, agentDefsPromise;

  // 8. interactive setup 后
  root, getFpsMetrics, stats, onboardingShown;

  // 9. post-trust
  mcpClients, mcpTools, mcpCommands, mcpPromise, hooksPromise, hookMessages;
  thinkingEnabled, thinkingConfig, setupTrigger;

  // 10. interactive 最终态
  initialState, sessionConfig, resumeContext;
};
```

`createActionContext(prompt, options)` 用默认值填充每个字段（约 100 个 `key: defaultValue`）。后续每个阶段函数都接收 `ctx: ActionContext` 并按需 mutate。这是把 ~2800 行的单一 callback 拆成 16 个小函数的关键。

**Why**: 注释解释，原 action 回调里有 ~80 个 `let` 局部变量，签名太大且数据流向不明确。改用对象后，引用稳定、签名小、数据流显式。

---

## 五、main() — 公开入口

```ts
export async function main() {
  profileCheckpoint('main_function_start');

  // 1. -d2e → --debug-to-stderr 别名
  if (process.argv.includes('-d2e')) { process.argv = ...; }

  // 2. Windows PATH 防护
  process.env.NoDefaultCurrentDirectoryInExePath = '1';

  // 3. 初始化 warning handler + SIGINT handler
  initializeWarningHandler();
  process.on('exit', resetCursor);
  process.on('SIGINT', ...);  // print 模式跳过，print.ts 自己处理

  // 4. cc:// URL 重写（DIRECT_CONNECT）
  if (feature('DIRECT_CONNECT')) { ... }

  // 5. --handle-uri / macOS __CFBundleIdentifier deep link（LODESTONE）
  if (feature('LODESTONE')) { ... }

  // 6. claude assistant [sessionId]（KAIROS）
  if (feature('KAIROS') && _pendingAssistantChat) { ... }

  // 7. claude ssh <host> [dir]（SSH_REMOTE）
  if (feature('SSH_REMOTE') && _pendingSSH) { ... }

  // 8. 决定 isInteractive / isNonInteractive
  const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY;
  setIsInteractive(!isNonInteractive);
  initializeEntrypoint(isNonInteractive);

  // 9. 决定 clientType（cli / github-action / sdk-ts / sdk-py / remote / ...）
  const clientType = ...;
  setClientType(clientType);
  setQuestionPreviewFormat(...);

  // 10. tag remote-control sessions
  if (process.env.CLAUDE_CODE_ENVIRONMENT_KIND === 'bridge') {
    setSessionSource('remote-control');
  }
  profileCheckpoint('main_client_type_determined');

  // 11. 早期加载 settings（在 init() 之前）
  eagerLoadSettings();
  profileCheckpoint('main_before_run');

  // 12. commander 编排
  await run();
  profileCheckpoint('main_after_run');
}
```

`main()` 的核心职责是**为 run() 准备干净的环境**：
- argv 重写（cc://、assistant、ssh）让 commander 看到的形状符合预期
- 决定 mode / clientType / entrypoint（影响后续所有 subsystem）
- 早期加载 settings（不依赖 init 之后才能读）

---

## 六、run() — Commander 编排器

```ts
async function run(): Promise<CommanderCommand> {
  profileCheckpoint('run_function_start');

  const program = new CommanderCommand()
    .configureHelp(createSortedHelpConfig())
    .enablePositionalOptions();
  profileCheckpoint('run_commander_initialized');

  // 关键：使用 preAction hook 在命令执行前做异步初始化
  program.hook('preAction', async thisCommand => {
    await runPreActionHookInit(thisCommand);
  });

  // 注册 60+ 选项 + 默认 action
  program.name('claude')...
    .option('--debug [filter]', ...)
    .option('-p, --print', ...)
    .option('--bare', ...)
    .option('--init', ...)
    .option('--output-format <format>', ...)
    .option('--continue', ...)
    .option('--resume [value]', ...)
    .option('--model <model>', ...)
    .option('--agent <agent>', ...)
    .addOption(new Option('--worktree [name]', ...))
    // ...
    .action(async (prompt, options) => {
      // 阶段 1-15 全部在这里按序调用
      const ctx = createActionContext(prompt, options);
      await normalizeActionEntry(ctx);
      await extractActionOptions(ctx);
      resolveSystemPrompts(ctx);
      resolvePermissionMode(ctx);
      await parseDynamicMcpConfig(ctx);
      await applyClaudeInChromeMcp(ctx);
      parseAndSetChannels(ctx);
      await initializeBriefAndToolPermissions(ctx);
      kickoffBackgroundMcpAndValidateFormats(ctx);
      await resolveInputPromptAndLoadTools(ctx);
      await runParallelSetup(ctx);
      await resolveModelAgentAndSystemPrompts(ctx);

      if (ctx.isNonInteractiveSession) {
        if (ctx.initOnly) { ...; gracefulShutdownSync(0); return; }
        await runHeadlessMode(ctx);
        return;
      }

      await runInteractiveSetupScreens(ctx);
      if (process.exitCode !== undefined) return;
      const postTrustResult = await runPostTrustInitialization(ctx);
      if (postTrustResult.earlyReturn) return;
      assembleInteractiveInitialState(ctx);
      await dispatchReplLaunch(ctx);
    })
    .version(`${MACRO.VERSION} (Claude Code)`, '-v, --version', '...');

  // 更多顶层 option（worktree、tmux、teammate、SDK、teleport、remote、ant-only）
  program.option('-w, --worktree [name]', ...);
  // ...

  // 优化：-p 模式跳过 52 个子命令注册（~65ms）
  const isPrintMode = process.argv.includes('-p') || process.argv.includes('--print');
  const isCcUrl = process.argv.some(a => a.startsWith('cc://') || a.startsWith('cc+unix://'));
  if (isPrintMode && !isCcUrl) {
    await program.parseAsync(process.argv);
    return program;
  }

  // 交互模式 / 子命令模式：注册完整子命令树
  registerSubcommandTree(program);
  await program.parseAsync(process.argv);

  profileCheckpoint('main_after_run');
  profileReport();
  return program;
}
```

### 6.1 preAction hook — commander 的"前置 init"

```ts
async function runPreActionHookInit(thisCommand: CommanderCommand): Promise<void> {
  // 1. 等待顶层副作用中启动的并行读取
  await Promise.all([ensureMdmSettingsLoaded(), ensureKeychainPrefetchCompleted()]);
  // 2. 全局 init：settings、telemetry、auth、tengu_init
  await init();
  // 3. 设置 process.title = 'claude'
  // 4. 附加 logEvent sink（PR #11106 之前 logEvent 是直发；现在会排队直到 sink attach）
  initSinks();
  // 5. 转发 --plugin-dir 到子命令
  // 6. 跑 14 个 migration
  runMigrations();
  // 7. 加载远程 managed settings（非阻塞）
  void loadRemoteManagedSettings();
  // 8. 上传用户 settings（UPLOAD_USER_SETTINGS 特性门）
}
```

**Why use `preAction`?** Commander 的帮助命令（`--help`）会触发 parse 但不触发 action。在 `preAction` 里做 init 可以让 `--help` 跳过 100ms+ 的 init 工作。

### 6.2 `-p` 模式跳过子命令注册

`isPrintMode && !isCcUrl` 的分支直接 `parseAsync` 不调用 `registerSubcommandTree(program)`。注释说明：
- 52 个子命令注册耗时约 65ms
- 主要是 `isBridgeEnabled()` 调用（25ms settings Zod parse + 40ms sync keychain subprocess）
- 两者都被 `try/catch` 在 `enableConfigs()` 之前吞掉，恒为 false

### 6.3 default action — 16 阶段流水线

action 回调内部按线性顺序调用 16 个阶段函数（详见第七节）。模式分流在阶段 11 之后：
- `isNonInteractiveSession` → `runHeadlessMode`
- 否则 → `runInteractiveSetupScreens` → `runPostTrustInitialization` → `assembleInteractiveInitialState` → `dispatchReplLaunch`

---

## 七、Action 处理阶段详解

### 阶段 0：`runPreActionHookInit`（preAction hook）

初始化 settings、auth、telemetry、migration。详见 §6.1。

### 阶段 1：`normalizeActionEntry`

```ts
async function normalizeActionEntry(ctx: ActionContext): Promise<void> {
  // 1. --bare 标记
  if (options.bare) { process.env.CLAUDE_CODE_SIMPLE = '1'; }
  // 2. "code" 当作空 prompt（仅做埋点）
  if (ctx.prompt === 'code') { ...; ctx.prompt = undefined; }
  // 3. 单字 prompt 埋点
  if (ctx.prompt && !/\s/.test(ctx.prompt) && ctx.prompt.length > 0) { logEvent('tengu_single_word_prompt', ...); }
  // 4. KAIROS assistant 模式门控
  if (feature('KAIROS') && options.assistant) { markAssistantForced(); }
  if (feature('KAIROS') && isAssistantMode() && !options.agentId) {
    if (!checkHasTrustDialogAccepted()) {
      console.warn('Assistant mode disabled: directory is not trusted.');
    } else {
      ctx.kairosEnabled = await kairosGate.isKairosEnabled();
      if (ctx.kairosEnabled) {
        options.brief = true; setKairosActive(true);
        ctx.assistantTeamContext = await initializeAssistantTeam();
      }
    }
  }
}
```

关键设计：assistant 模式**信任门**——`.claude/settings.json` 是 attacker-controllable 的，必须等用户接受 trust dialog 后才能激活 KAIROS。

### 阶段 2：`extractActionOptions`

把 commander options destructure 出来塞进 ctx。涵盖 worktree、tmux、teammate、SDK URL、teleport、remote、RC、sessionId、file download、agentsJson、baseTools、allowedTools、disallowedTools、outputFormat、inputFormat、verbose、init、initOnly、maintenance、disableSlashCommands、ide、debug、debugToStderr、includeHookEvents、includePartialMessages、dangerouslySkipPermissions、allowDangerouslySkipPermissions、sessionName、permissionModeCli、mcpConfig、tasksOption 等 50+ 字段。

### 阶段 3：`resolveSystemPrompts`

处理 `--system-prompt` / `--system-prompt-file` / `--append-system-prompt` / `--append-system-prompt-file`，并对 tmux teammate 追加 `TEAMMATE_SYSTEM_PROMPT_ADDENDUM`。

### 阶段 3b：`resolvePermissionMode`

```ts
const { mode, notification } = initialPermissionModeFromCLI({
  permissionModeCli: ctx.permissionModeCli,
  dangerouslySkipPermissions: ctx.dangerouslySkipPermissions
});
setSessionBypassPermissionsMode(mode === 'bypassPermissions');
if (feature('TRANSCRIPT_CLASSIFIER')) {
  if (ctx.options.enableAutoMode || ctx.permissionModeCli === 'auto' || mode === 'auto' || !ctx.permissionModeCli && isDefaultPermissionModeAuto()) {
    autoModeStateModule?.setAutoModeFlagCli(true);
  }
}
```

### 阶段 4：`parseDynamicMcpConfig`

校验 `--mcp-config` 的 JSON 字符串或文件路径，合并多条配置，强制 enterprise policy（`allowedMcpServers` / `deniedMcpServers`），阻断保留名（`claude-in-chrome`、`computer-use`）。

### 阶段 4b：`applyClaudeInChromeMcp`

注入 Claude in Chrome MCP 服务，处理 Chicago MCP（computer use），触发 enterprise policy 检查。

### 阶段 5：`parseAndSetChannels`

KAIROS 特性门下的 channels 注册。

### 阶段 6：`initializeBriefAndToolPermissions`

构造 `toolPermissionContext`（bypassPermissions / plan / default / acceptEdits / auto 等模式），并处理过宽的 bash permission / 危险 permission 告警。

### 阶段 7：`kickoffBackgroundMcpAndValidateFormats`

启动后台 MCP 配置解析（`mcpConfigPromise` / `claudeaiConfigPromise`），并校验 JSON schema 格式。

### 阶段 8：`resolveInputPromptAndLoadTools`

```ts
ctx.effectivePrompt = ctx.prompt || '';
ctx.inputPrompt = await getInputPrompt(ctx.effectivePrompt, ctx.inputFormat ?? 'text');
maybeActivateProactive(ctx.options);  // 必须在 getTools() 之前
let tools = getTools(ctx.toolPermissionContext);
if (feature('COORDINATOR_MODE') && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
  tools = applyCoordinatorToolFilter(tools);
}
ctx.tools = tools;
// 处理 --json-schema → 注入 SyntheticOutputTool
```

**Why `maybeActivateProactive` before `getTools()`?** `SleepTool.isEnabled()` 检查 `isProactiveActive()`，必须先激活才能让 Sleep 进入工具列表。

### 阶段 9：`runParallelSetup` — 关键并行优化

```ts
async function runParallelSetup(ctx: ActionContext): Promise<void> {
  // 1. 注册 bundled 插件和技能（纯内存 <1ms）
  if (process.env.CLAUDE_CODE_ENTRYPOINT !== 'local-agent') {
    initBuiltinPlugins();
    initBundledSkills();
  }
  // 2. 并行启动 setup() + getCommands() + getAgentDefinitionsWithOverrides()
  ctx.setupPromise = setup(preSetupCwd, ctx.permissionMode, ctx.allowDangerouslySkipPermissions, ctx.worktreeEnabled, ...);
  ctx.commandsPromise = ctx.worktreeEnabled ? null : getCommands(preSetupCwd);
  ctx.agentDefsPromise = ctx.worktreeEnabled ? null : getAgentDefinitionsWithOverrides(preSetupCwd);
  await ctx.setupPromise;
  // 3. 头less 模式下提前 spawn git status / getUserContext / ensureModelStringsInitialized()
  if (getIsNonInteractiveSession()) {
    applyConfigEnvironmentVariables();
    void getSystemContext();
    void getUserContext();
    void ensureModelStringsInitialized();
  }
}
```

**Why parallelize?** `setup()` 的 ~28ms 主要是 `startUdsMessaging`（socket bind），不与 `getCommands` 的文件 I/O 争抢 CPU/IO。`--worktree` 路径除外（`setup()` 会 process.chdir，commands/agents 需要 post-chdir cwd）。

### 阶段 10：`resolveModelAgentAndSystemPrompts`

```ts
// 1. --name 缓存
cacheSessionTitle(ctx.options.name);

// 2. Ant 模型别名（capybara-fast 等）需要 GrowthBook 预热
if ("external" === 'ant' && explicitModel && ...) {
  await initializeGrowthBook();
}

// 3. 解析 model + 合并 CLI agents + 设置 mainThreadAgentDefinition
const commands = await (ctx.commandsPromise ?? getCommands(currentCwd));
const { activeAgents, allAgents } = await (ctx.agentDefsPromise ?? getAgentDefinitionsWithOverrides(currentCwd));
// ... 合并 cliAgents、解析 mainThreadAgentDefinition、注入系统提示 ...

// 4. 计算 effectiveModel（CLI > agent.model > default）
ctx.effectiveModel = userSpecifiedModel ?? (mainThreadAgentDefinition?.model !== 'inherit' ? mainThreadAgentDefinition.model : undefined);

// 5. Advisor 模型校验
// 6. tmux teammate 自定义 agent 追加 prompt
// 7. 激活 brief、proactive、assistant 模式
maybeActivateBrief(ctx.options);
if (defaultView === 'chat' && isBriefEntitled()) { setUserMsgOptIn(true); }
if (proactive && !isCoordinatorMode()) { appendSystemPrompt += proactivePrompt; }
if (feature('KAIROS') && ctx.kairosEnabled) { appendSystemPrompt += getAssistantSystemPromptAddendum(); }
```

### 阶段 11：`runInteractiveSetupScreens`（仅交互模式）

```ts
async function runInteractiveSetupScreens(ctx: ActionContext): Promise<void> {
  // 1. 创建 Ink root（patchConsole 会吞掉 console 输出，所以 headless 模式不能用）
  const ctxRender = getRenderContext(false);
  root = await createRoot(ctxRender.renderOptions);

  // 2. 记录 startup 时间（在任何阻塞 dialog 之前）
  logEvent('tengu_timer', { event: 'startup', durationMs: Math.round(process.uptime() * 1000) });

  // 3. 跑 setup screens（trust dialog / OAuth / onboarding / resume picker）
  const onboardingShown = await showSetupScreens(root, ...);

  // 4. 解析 --remote-control 门控
  if (feature('BRIDGE_MODE') && ctx.remoteControlOption !== undefined) {
    ctx.remoteControl = (await getBridgeDisabledReason()) === null;
  }

  // 5. 处理 agent memory snapshot 更新（ant-only）

  // 6. onboarding 后刷新 auth 相关服务
  if (ctx.onboardingShown) {
    void refreshRemoteManagedSettings();
    void refreshPolicyLimits();
    resetUserCache();
    refreshGrowthBookAfterAuthChange();
    void clearTrustedDeviceToken();
    void enrollTrustedDevice();
  }

  // 7. 校验 force login org
  const orgValidation = await validateForceLoginOrg();
  if (!orgValidation.valid) { await exitWithError(root, orgValidation.message); }
}
```

### 阶段 12：`runPostTrustInitialization`（仅交互模式）

信任建立后：
1. 初始化 LSP manager（**故意延后**——防止 plugin LSP 在 untrusted 目录执行代码）
2. 显示 settings 校验错误
3. 后台预取 quota / passes / fastMode / bootstrap
4. 解析 `mcpConfigPromise`，分离 `regularMcpConfigs` / `sdkMcpConfigs`
5. 启动 MCP 预连接（prefetchAllMcpResources）—— **不阻塞 REPL render 或 turn 1 TTFT**
6. 启动 hooks（仅当非 initOnly/init/maintenance/nonInteractive/continue/resume）
7. 设置 thinking config（adaptive / disabled / enabled with budget）
8. 处理 `--init-only` 短路
9. logSessionTelemetry / logStartupTelemetry

### 阶段 13：`runHeadlessMode`（`-p` 模式）

```ts
async function runHeadlessMode(ctx: ActionContext): Promise<void> {
  if (stream-json/json) setHasFormattedOutput(true);
  applyConfigEnvironmentVariables();  // 信任被绕过，全量 env
  initializeTelemetryAfterTrust();

  // 启动 SessionStart hooks（与 MCP / plugin / print.ts import 重叠）
  const sessionStartHooksPromise = ...;

  // 校验 force login org
  await validateForceLoginOrg();

  // 过滤 prompt / local commands
  const commandsHeadless = ctx.disableSlashCommands ? [] : ctx.commands.filter(...);

  // 创建 headlessStore
  const headlessStore = createStore({ ...defaultState, mcp: { clients, commands, tools }, ... }, onChangeAppState);

  // Print-mode MCP：per-server 增量推送
  await connectMcpBatch(regularMcpConfigs, 'regular');
  // 5 秒超时后后台继续 claude.ai connectors
  const claudeaiTimedOut = await Promise.race([claudeaiConnect, timeout(5000)]);

  // 启动 deferred prefetches + 后台 housekeeping
  if (!isBareMode()) {
    startDeferredPrefetches();
    void startBackgroundHousekeeping();
  }
  logSessionTelemetry();

  // 调用 print.ts
  const { runHeadless } = await import('src/cli/print.js');
  void runHeadless(ctx.inputPrompt, () => headlessStore.getState(), headlessStore.setState, ...);
}
```

### 阶段 14：`assembleInteractiveInitialState`

构造 `AppState`：toolPermissionContext、agentDefinitions、mcp、plugins、notifications、todos、fileHistory、speculation、teamContext 等约 100 个字段。

```ts
saveGlobalConfig(current => ({ ...current, numStartups: (current.numStartups ?? 0) + 1 }));
setImmediate(() => { void logStartupTelemetry(); logSessionTelemetry(); });

// sessionConfig
const sessionConfig = {
  debug, commands, initialTools, mcpClients, autoConnectIdeFlag, mainThreadAgentDefinition,
  disableSlashCommands, dynamicMcpConfig, strictMcpConfig, systemPrompt, appendSystemPrompt,
  taskListId, thinkingConfig, ...(uploaderReady && { onTurnComplete })
};
```

### 阶段 15：`dispatchReplLaunch` — REPL 启动路由

按优先级链式判断：

```ts
async function dispatchReplLaunch(ctx: ActionContext): Promise<void> {
  if (ctx.options.continue) return runContinueFlow(ctx);
  if (feature('DIRECT_CONNECT') && _pendingConnect?.url) return runDirectConnectFlow(ctx);
  if (feature('SSH_REMOTE') && _pendingSSH?.host) return runSshFlow(ctx);
  if (feature('KAIROS') && _pendingAssistantChat && (_pendingAssistantChat.sessionId || _pendingAssistantChat.discover)) return runAssistantFlow(ctx);
  if (ctx.options.resume || ctx.options.fromPr || ctx.teleport || ctx.remote !== null) return runResumeFlow(ctx);
  if (ctx.options.teleport) return runTeleportFlow(ctx);
  if (ctx.remote !== null) return runRemoteFlow(ctx);
  if (ctx.options.rewindFiles) return runRewindFilesFlow(ctx);

  return runFreshSessionFlow(ctx);
}
```

每个分支调用对应的嵌套辅助函数，调用 `launchRepl/launchResumeChooser` 后 return，控制流不会 fall through。

### 阶段 16：`registerSubcommandTree`（`run()` 末尾）

50+ 顶级子命令注册，分组：
- `mcp`（serve、add、remove、list、get、add-json、add-from-claude-desktop、reset-project-choices）
- `server`（DIRECT_CONNECT）
- `ssh`（SSH_REMOTE）
- `open`（DIRECT_CONNECT headless）
- `auth`（login、logout、status 等）
- `plugin`（validate、list、marketplace、install、uninstall、enable、disable、update）
- `setup-token`
- `agents`
- `auto-mode`（TRANSCRIPT_CLASSIFIER）
- `doctor` / `update` / `install`
- `up` / `rollback`（ant-only）
- `log` / `error` / `export` / `task` / `completion`（ant-only）

---

## 八、子命令注册与 fast-path 优化

`src/entrypoints/cli.tsx` 在 `run()` 之前提供一系列 fast-path：

| argv | 行为 | 跳过内容 |
|------|------|---------|
| `--version` / `-v` / `-V` | 直接输出 `MACRO.VERSION` | 0 imports |
| `--dump-system-prompt` | 输出渲染后的 system prompt | 整个 CLI |
| `--claude-in-chrome-mcp` / `--chrome-native-host` / `--computer-use-mcp` | 启动对应 MCP server | commander |
| `--daemon-worker` | 启动 daemon worker | enableConfigs / analytics |
| `remote-control` / `rc` / `remote` / `sync` / `bridge` | 启动 bridge | 完整 CLI 加载 |
| `daemon` | 启动 daemon supervisor | 完整 CLI 加载 |
| `ps` / `logs` / `attach` / `kill` / `--bg` | session management | 完整 CLI 加载 |
| `new` / `list` / `reply`（TEMPLATES） | template jobs | 完整 CLI 加载 |
| `environment-runner` / `self-hosted-runner` | BYOC runner | 完整 CLI 加载 |
| `--tmux` + `--worktree` | exec into tmux worktree | 完整 CLI 加载 |
| `--update` / `--upgrade` | 重写为 `update` 子命令 | commander 选项校验 |
| `--bare` | 提前设置 `CLAUDE_CODE_SIMPLE=1` | 让所有 gated 路径走极简模式 |

之后通过 `dynamic import('../main.js')` 加载完整 CLI。设计目标：**最短的 happy path 不付出模块加载成本**。

---

## 九、关键技术点

### 9.1 Settings 路径缓存（避免破坏 API prompt cache）

```ts
// --settings JSON 字符串时，用 contentHash 作为临时文件名（不是 UUID）
settingsPath = generateTempFilePath('claude-settings', '.json', { contentHash: trimmedSettings });
```

**Why?** Settings 路径会出现在 Bash 工具的 sandbox `denyWithinAllow` 列表里，进而出现在 tool description → API 请求里。随机 UUID 会让每次 `query()` 调用都让 prompt cache prefix 失效，导致 12× input token 成本。Content hash 保证相同 settings → 相同路径。

### 9.2 Migration 版本号

```ts
const CURRENT_MIGRATION_VERSION = 11;
function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateEnableAllProjectMcpServersToSettings();
    resetProToOpusDefault();
    migrateSonnet1mToSonnet45();
    migrateLegacyOpusToCurrent();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    migrateReplBridgeEnabledToRemoteControlAtStartup();
    if (feature('TRANSCRIPT_CLASSIFIER')) resetAutoModeOptInForDefaultOffer();
    if ("external" === 'ant') migrateFennecToOpus();
    // 持久化版本号
  }
}
```

每次 bump `CURRENT_MIGRATION_VERSION` 强制重跑迁移。

### 9.3 启动性能埋点

整条流水线用 `profileCheckpoint(name)` 标记关键时间点：

```
main_tsx_entry
main_tsx_imports_loaded
main_function_start
main_warning_handler_initialized
main_client_type_determined
main_before_run
main_after_run
eagerLoadSettings_start
eagerLoadSettings_end
preAction_start
preAction_after_mdm
preAction_after_init
preAction_after_sinks
preAction_after_migrations
preAction_after_remote_settings
preAction_after_settings_sync
run_function_start
run_commander_initialized
run_main_options_built
run_before_parse
run_after_parse
action_handler_start
action_after_input_prompt
action_tools_loaded
action_before_setup
action_after_setup
action_commands_loaded
action_mcp_configs_loaded
before_connectMcp
after_connectMcp
after_connectMcp_claudeai
before_validateForceLoginOrg
before_print_import
after_print_import
startup
```

配合 `--debug` 输出到 stderr，可绘制启动 timeline。`profileReport()` 在末尾汇总。

### 9.4 早返回 / 早退出

- `--init-only`：`processSetupHooks('init')` + `processSessionStartHooks('startup')` → `gracefulShutdownSync(0)`
- `--help`：commander 直接打印 help，`preAction` 内的 init 不会执行
- `--version` / `--dump-system-prompt`：cli.tsx fast-path 早返回
- 阶段 11 之后如果 `process.exitCode !== undefined`（用户拒绝 trust dialog）：直接 return

### 9.5 安全护栏

| 位置 | 护栏 |
|------|------|
| L4566 | `process.env.NoDefaultCurrentDirectoryInExePath = '1'`（Windows PATH hijack） |
| L266 | `isBeingDebugged()` → `process.exit(1)`（阻断调试模式） |
| 阶段 6 | 过宽 bash permission / 危险 permission 告警 |
| 阶段 11 | `validateForceLoginOrg()` 校验 org |
| 阶段 12 | `checkAndDisableBypassPermissions()` Statsig 门控 |
| 阶段 4 | `filterMcpServersByPolicy()` enterprise policy |
| KAIROS 阶段 1 | trust dialog 未接受则不激活 assistant 模式 |
| LSP | trust 之后才初始化 LSP（防止 plugin LSP 在 untrusted 目录执行代码） |

---

## 十、子系统调用清单

`main.tsx` 调用到的子系统（部分）：

| 子系统 | 路径 | 阶段 |
|--------|------|------|
| `init` | `src/entrypoints/init.ts` | preAction |
| `setup` | `src/setup.ts` | 阶段 9 |
| `getCommands` | `src/commands.ts` | 阶段 9-10 |
| `getAgentDefinitionsWithOverrides` | `src/tools/AgentTool/loadAgentsDir.ts` | 阶段 9-10 |
| `getTools` | `src/tools.ts` | 阶段 8 |
| `prefetchAllMcpResources` | `src/services/mcp/client.ts` | 阶段 12-13 |
| `getMcpToolsCommandsAndResources` | `src/services/mcp/client.ts` | 阶段 13 |
| `processSessionStartHooks` / `processSetupHooks` | `src/utils/sessionStart.ts` | 阶段 12-13 |
| `runHeadless` | `src/cli/print.ts` | 阶段 13 |
| `launchRepl` | `src/replLauncher.tsx` | 阶段 15 |
| `showSetupScreens` | `src/interactiveHelpers.tsx` | 阶段 11 |
| `logTenguInit` | 本文件 | 阶段 13 后 |
| `logSessionTelemetry` / `logStartupTelemetry` | 本文件 | 阶段 14 |
| `startDeferredPrefetches` | 本文件 | 阶段 14（import 时） |

---

## 十一、典型调用链

### 交互模式 happy path

```
$ claude
↓
src/entrypoints/cli.tsx::main()
  ├─ argv 不命中任何 fast-path
  ├─ import('../main.js')
  └─ cliMain()
       ↓
src/main.tsx::main()
  ├─ argv 重写（cc:///assistant/sssh）
  ├─ 设置 NoDefaultCurrentDirectoryInExePath
  ├─ initializeWarningHandler / SIGINT
  ├─ 决定 isInteractive=true / clientType='cli'
  ├─ eagerLoadSettings()
  └─ run()
       ├─ 创建 CommanderCommand，hook preAction
       ├─ 注册 60+ 选项
       ├─ !isPrintMode → registerSubcommandTree(program)
       └─ program.parseAsync(argv)
            ├─ preAction hook → runPreActionHookInit()
            │   ├─ ensureMdmSettingsLoaded + ensureKeychainPrefetchCompleted
            │   ├─ init()  ← settings/auth/telemetry
            │   ├─ process.title = 'claude'
            │   ├─ initSinks()
            │   ├─ --plugin-dir 转发
            │   ├─ runMigrations()
            │   ├─ void loadRemoteManagedSettings / loadPolicyLimits
            │   └─ void uploadUserSettingsInBackground
            └─ default .action() 触发
                 ├─ 阶段 1-10 同步完成
                 ├─ isNonInteractiveSession=false
                 ├─ 阶段 11：runInteractiveSetupScreens
                 │   ├─ createRoot (Ink)
                 │   ├─ showSetupScreens()  ← trust dialog / OAuth / onboarding
                 │   ├─ onboardingShown 后刷新 GrowthBook / 信任设备注册
                 │   └─ validateForceLoginOrg
                 ├─ 阶段 12：runPostTrustInitialization
                 │   ├─ initializeLspServerManager
                 │   ├─ 显示 settings 校验错误
                 │   ├─ 启动 quota/bootstrap/fastMode 预取
                 │   ├─ 解析 mcpConfigPromise
                 │   ├─ prefetchAllMcpResources  ← 异步
                 │   ├─ 启动 SessionStart hooks  ← 异步
                 │   └─ 设置 thinking config
                 ├─ 阶段 14：assembleInteractiveInitialState
                 │   ├─ 构造 AppState
                 │   └─ 构造 sessionConfig / resumeContext
                 └─ 阶段 15：dispatchReplLaunch → runFreshSessionFlow
                      ├─ launchResumeChooser (如有 --resume)
                      └─ launchRepl(root, { initialState }, sessionConfig, renderAndRun)
                           └─ 用户开始对话
```

### 头less 模式 happy path

```
$ echo "summarize this" | claude -p
↓
src/entrypoints/cli.tsx::main()  (fast-path 不命中)
  └─ cliMain() → import('../main.js') → main() → run()
       ├─ preAction hook → runPreActionHookInit() (同交互模式)
       └─ default .action() 触发
            ├─ 阶段 1-12 完成
            ├─ isNonInteractiveSession=true
            ├─ initOnly=false
            └─ 阶段 13：runHeadlessMode
                 ├─ setHasFormattedOutput
                 ├─ applyConfigEnvironmentVariables (全量)
                 ├─ initializeTelemetryAfterTrust
                 ├─ 启动 SessionStart hooks (异步)
                 ├─ validateForceLoginOrg
                 ├─ 创建 headlessStore
                 ├─ connectMcpBatch(regularMcpConfigs)
                 ├─ connectMcpBatch(claudeaiConfigs)  ← 5s 超时后台继续
                 ├─ startDeferredPrefetches / startBackgroundHousekeeping
                 ├─ logSessionTelemetry
                 └─ import('src/cli/print.js') → runHeadless(...)
                      └─ 调用 print.ts 处理 stdin → 调 API → 输出结果
```

### 头less init-only

```
$ claude --init-only
↓
...
├─ preAction 跑完
├─ 阶段 1-12 完成
├─ isNonInteractiveSession=true, initOnly=true
├─ processSetupHooks('init', { forceSyncExecution: true })
├─ processSessionStartHooks('startup', { forceSyncExecution: true })
└─ gracefulShutdownSync(0)
```

---

## 十二、修改指南

| 想做的事 | 改哪里 |
|---------|--------|
| 新增 commander option | `run()` 的 `.action()` 之前的 `.option()` / `.addOption()` 链 |
| 新增 commander 子命令 | `registerSubcommandTree()` |
| 新增 init 阶段 | 在 `action` 回调里加一行；`ActionContext` 添加字段；如有复杂逻辑新建 `阶段 N` 函数 |
| 新增 KAIROS 模式 | 用 `feature('KAIROS')` gate；遵循 assistant / brief / proactive 已有模式 |
| 新增 migration | `runMigrations()` + bump `CURRENT_MIGRATION_VERSION` |
| 新增 startup prefetch | `startDeferredPrefetches()`（在 `setup()` 之后调用） |
| 新增 fast-path | `src/entrypoints/cli.tsx::main()` 顶部 |
| 新增 help 文本 | `program.helpOption(...)` 或 `.description(...)` |
| 改 print 模式行为 | `runHeadlessMode()` + `src/cli/print.ts` |
| 改 REPL 启动路径 | `dispatchReplLaunch()` 的优先级链 |

**调试技巧**：
- `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER=1`：跑通第一帧渲染就退出（用于测 startup perf）
- `CLAUDE_CODE_SIMPLE=1`：等价 `--bare`
- `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1`：不改 process.title
- `--debug [filter]`：开启 debug 日志，过滤器语法 `"api,hooks"` 或 `"!1p,!file"`
- `SHOULD_PROFILE` 编译期宏：开启更详细的 profile 报告

---

## 十三、关键设计原则总结

1. **并行优于串行**：MDM 读取 / keychain 预取 / setup() / getCommands / getAgentDefinitionsWithOverrides / git status / getSystemContext 全部并行启动
2. **早期加载 vs 延后加载**：settings（无依赖）早期加载；LSP（依赖 trust）信任后加载；bundled 插件/技能（纯内存）在 setup() 之前加载
3. **cache-friendly**：settings 路径用 contentHash 避免破坏 API prompt cache；fast-path 用 dynamic import 避免完整 CLI 加载
4. **trust 优先**：所有可能执行代码的操作（apiKeyHelper / MCP / LSP / plugin）必须等 trust dialog 接受
5. **特性门 + 条件 require**：`feature('X')` 编译期 DCE + `require` 而非 `import` 实现 SKU 隔离
6. **状态对象化**：把 ~80 个 `let` 局部变量压成单一 `ActionContext`，让 16 个阶段函数签名小巧、数据流显式
7. **preAction hook 而非 --help 前 init**：让 `--help` 跳过 100ms+ 的初始化
8. **fast-path 优先级**：argv 重写 → fast-path in cli.tsx → registerSubcommandTree → parse
