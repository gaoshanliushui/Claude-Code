# `src/entrypoints/cli.tsx` —— 启动入口与 Fast-path 分发

> 路径：`src/entrypoints/cli.tsx`（321 行）
> 角色：Claude Code CLI 的真正入口文件
> 关键性质：所有 ESM 顶层副作用集中在此；其余模块通过 `await import(...)` 动态加载

---

## 一、文件职责总览

`cli.tsx` 不是普通的"调用 main.tsx"包装层。它在 Bun 进程启动后第一个被执行，承担 **4 件不可替代** 的工作：

| 职责 | 含义 | 工程意义 |
|------|------|----------|
| ① Bun 运行时 polyfill | 用 `feature()`/`MACRO`/`BUILD_*` 全局变量替代 `bun:bundle` 的构建期宏 | 反编译/精简构建必须绕过构建期宏，否则模块求值即崩 |
| ② 进程级环境治理 | Corepack 自动 pin 关闭、CCR 容器 `--max-old-space-size=8192` | 防用户环境被改坏；防 OOM |
| ③ 子命令 fast-path | `--version`/`--claude-in-chrome-mcp`/`daemon`/`bridge` 等 11 类分支在加载 `main.tsx` **之前**短路返回 | 关键性能路径（`--version` 零模块加载） |
| ④ Stage-gating 入口 | 根据参数挑选最终调用方（绝大多数分支会落到 `import("../main.jsx")`） | `cli.tsx` 不解析任何 commander 参数——只做"是不是 special flag"的二值判定 |

> 一句话：`cli.tsx` 是 **Stage 0（before any module load）** 的精简路由器，运行时必须先于 `main.tsx` 完成所有顶层 polyfill 才能进入正常入口。

---

## 二、顶层副作用（Top-level Side Effects）

文件从第 1 行就开始执行，不在 `main()` 函数内。这些顶层副作用是 **模块求值期** 必然执行的：

### 2.1 `feature()` polyfill（行 3）

```typescript
const feature = (_name: string) => false;
```

- 原版：`feature("FOO")` 由 `bun:bundle` 在 **构建期** 内联为布尔字面量。
- 本项目：构建器未配置 `bun:bundle`，所有 `feature(...)` 必须运行时存在。
- **策略**：固定返回 `false`，**所有 feature flag 守卫的分支都不会进入**。
- 影响范围：`COORDINATOR_MODE`、`KAIROS`、`PROACTIVE`、`DAEMON`、`BRIDGE_MODE`、`BG_SESSIONS`、`TEMPLATES`、`CHICAGO_MCP`、`BYOC_ENVIRONMENT_RUNNER`、`SELF_HOSTED_RUNNER`、`DUMP_SYSTEM_PROMPT`、`ABLATION_BASELINE` 等十几处 feature flag —— **在该入口下都是死代码**。
- Bun bundler 在打包时可基于 `false` 字面量做 DCE（dead-code elimination），所以这些分支不会出现在最终 `dist/cli.js` 里。

### 2.2 `globalThis.MACRO` polyfill（行 4–14）

```typescript
if (typeof globalThis.MACRO === "undefined") {
  (globalThis as any).MACRO = {
    VERSION: "2.2.0",
    BUILD_TIME: new Date().toISOString(),
    FEEDBACK_CHANNEL: "",
    ISSUES_EXPLAINER: "",
    NATIVE_PACKAGE_URL: "",
    PACKAGE_URL: "",
    VERSION_CHANGELOG: "",
  };
}
```

- 原版：`MACRO` 在构建时被 inlined；运行时引用都来自那一行常量。
- 本项目：用运行时对象模拟。
- 触发链：`MACRO.VERSION` 在第 70 行 `--version` fast-path 中被引用——若不预先注入，`ReferenceError` 会打断 fast-path。
- `if (typeof ... === "undefined")` 的意义：允许用户在 `bunfig.toml` 中预先注入一个更精准的 MACRO（如 `BUILD_TIME` 用真实打包时间），让运行时 polyfill 礼貌后退。

### 2.3 构建时常量（行 16–18）

```typescript
(globalThis as any).BUILD_TARGET = "external";
(globalThis as any).BUILD_ENV = "production";
(globalThis as any).INTERFACE_TYPE = "stdio";
```

- `BUILD_TARGET`：`"ant"` / `"external"` 二选一。本项目固定为 `"external"`，即面向 CLI 用户的发布版；不带 ant-only 后门。
- `BUILD_ENV`：`"development"` / `"production"`。这里写死为 production，等价于"对外发布"的语义。
- `INTERFACE_TYPE`：`"stdio"` / `"web"` / `"sdk"`。stdio 表示走标准 I/O 通道（即 `bun run dev` 这种 CLI 形态）。

> 后续任何代码读这三个变量都拿到"外部生产版 CLI"的语义；它们与 `feature()` 配合形成"三层门控"（详见 `docs/architecture/17-architecture.md` §4.1）。

### 2.4 Corepack 自动 pin 修复（行 22）

```typescript
process.env.COREPACK_ENABLE_AUTO_PIN = "0";
```

原版由 Claude Code 自己写的 eslint 规则 `custom-rules/no-top-level-side-effects` 抑制警告，因为这是一个 **故意** 的顶层副作用。

**问题**：Corepack 检测到 `packageManager` 字段时会自动 `yarn add` 把版本写进用户的 package.json。Claude Code 不希望用户的 json 被第三方工具修改。

### 2.5 CCR 容器内存上限（行 26–33）

```typescript
if (process.env.CLAUDE_CODE_REMOTE === "true") {
  const existing = process.env.NODE_OPTIONS || "";
  process.env.NODE_OPTIONS = existing
    ? `${existing} --max-old-space-size=8192`
    : "--max-old-space-size=8192";
}
```

- `CLAUDE_CODE_REMOTE=true`：用户处于 Anthropic 托管的 CCR（Cloud Code Runner）容器中，物理内存 16GB。
- Node/V8 默认 `--max-old-space-size` 为 1.5GB 不足以容纳长会话；这里强制提到 8GB。
- `existing` 拼接保证不破坏用户预设的 `--inspect-brk=...` 等调试选项。

### 2.6 Ablation baseline（行 35–53）—— DCE 死块

```typescript
if (feature("ABLATION_BASELINE") && process.env.CLAUDE_CODE_ABLATION_BASELINE) { ... }
```

注意这是放在顶层（不在 `main()` 内）的，原因写在注释里：

> BashTool/AgentTool/PowerShellTool capture `DISABLE_BACKGROUND_TASKS` into module-level consts at import time — init() runs too late.

这些 Tool 在 import 时就把环境变量值"快照"到模块级 `const`，所以必须在 `main.tsx` 触发它们 import 之前就把环境变量设好。

但因为 `feature("ABLATION_BASELINE")` 恒为 `false`，整个 if 块在构建产物里被 DCE 掉，本项目实际无运行时开销。

---

## 三、`main()` 的 fast-path 路径图

进入 `main()` 后，参数被分阶段分发。下面是完整 **决策树**（按代码顺序）：

```
process.argv.slice(2) = args
│
├─ args.length===1 && args[0] in {--version,-v,-V}
│   └─→ console.log(`${MACRO.VERSION} (Claude Code)`)   ← 零模块加载
│
├─ profileCheckpoint("cli_entry")                        ← 加载启动 profiler
│
├─ args[0] === "--dump-system-prompt"   [DUMP_SYSTEM_PROMPT DCE 掉]
│
├─ args[0] === "--claude-in-chrome-mcp"
│   └─→ runClaudeInChromeMcpServer()
├─ args[0] === "--chrome-native-host"
│   └─→ runChromeNativeHost()
├─ args[0] === "--computer-use-mcp"     [CHICAGO_MCP DCE 掉]
│
├─ args[0] === "--daemon-worker"        [DAEMON DCE 掉]
│
├─ args[0] in {remote-control, rc, remote, sync, bridge}  [BRIDGE_MODE DCE 掉]
│   ├─ enableConfigs()
│   ├─ auth check → BRIDGE_LOGIN_ERROR
│   ├─ getBridgeDisabledReason() (GrowthBook gate, fresh)
│   ├─ checkBridgeMinVersion()
│   ├─ waitForPolicyLimitsToLoad() + isPolicyAllowed("allow_remote_control")
│   └─→ bridgeMain(args.slice(1))
│
├─ args[0] === "daemon"                [DAEMON DCE 掉]
│   └─→ daemonMain()
│
├─ args[0] in {ps, logs, attach, kill} || args.includes("--bg"|"--background")
│   │                                  [BG_SESSIONS DCE 掉]
│   └─→ bg.psHandler / logsHandler / attachHandler / killHandler / handleBgFlag
│
├─ args[0] in {new, list, reply}       [TEMPLATES DCE 掉]
│   └─→ templatesMain(args); process.exit(0)   ← 注意 process.exit 而非 return
│
├─ args[0] === "environment-runner"    [BYOC_ENVIRONMENT_RUNNER DCE 掉]
│
├─ args[0] === "self-hosted-runner"    [SELF_HOSTED_RUNNER DCE 掉]
│
├─ tmux + worktree fast-path
│   ├─ hasTmuxFlag = args has --tmux || --tmux=classic
│   ├─ hasWorktree = -w || --worktree || --worktree=...
│   ├─ enableConfigs()
│   ├─ isWorktreeModeEnabled() (GrowthBook gate)
│   └─→ execIntoTmuxWorktree(args)
│       ├─ result.handled → return
│       └─ result.error   → exitWithError()
│
├─ args.length===1 && args[0] in {--update,--upgrade}
│   └─→ 把 argv[2] 改写为 "update"
│
├─ args.includes("--bare")
│   └─→ process.env.CLAUDE_CODE_SIMPLE = "1"
│      (必须在 commander 求值前生效，否则 gates 触发延迟)
│
└─ 兜底分支：full CLI
    ├─ startCapturingEarlyInput()
    ├─ profileCheckpoint("cli_before_main_import")
    ├─ const { main: cliMain } = await import("../main.jsx")
    ├─ profileCheckpoint("cli_after_main_import")
    └─ await cliMain();
```

> 标 `[XXX DCE 掉]` 的分支：本项目 `feature(XXX)` 返回 `false`，bun bundler 把整个 `if (feature(...) && ...)` 求值为 `false && ...`，DCE 阶段删除分支体。所以表格左侧列出的分支在实际打包结果里 **不会存在**——但源码里仍在位置记录了原版支持的入口。

---

## 四、关键设计决策

### 4.1 为什么不直接 `import "../main.jsx"`？

原版也使用动态 `import`：

```typescript
const { main: cliMain } = await import("../main.jsx");
```

三个理由（按重要性）：

1. **fast-path 早退**：`--version` fast-path 必然要在加载 `main.tsx` 之前完成返回；`main.tsx` import 会同步求值 commander、analytics sinks、policyLimits、AppState 等几十个模块——`--version` 不应该付出这个代价。
2. **profile 维度分离**：`profileCheckpoint("cli_before_main_import")` / `"cli_after_main_import"` 是启动性能基准的关键切片；只有 dynamic import 能准确测出 import 阶段耗时。
3. **避免循环依赖**：`main.tsx` 内部通过工具依赖网最终会回到 `entrypoints/init.ts`；用 `await import` 把这条链放在非求值期。

### 4.2 为什么 `enableConfigs()` 在 fast-path 里手动调用？

`utils/config.js` 内的 `enableConfigs()` 是 **main.tsx 默认在 `cliMain()` 之前会调用一次** 的副作用。bridge/daemon/bg/tmux 这几个 fast-path 都跳过了 `main.tsx`，必须自行补调，否则配置文件加载、用户级 settings、CLAUDE.md 解析都不会发生。

这意味着：每个 fast-path 分支作者必须知道 "我需要哪些 main.tsx 的前置副作用"。这条隐性契约在 [`docs/own/RECORD.md`](./RECORD.md) 中有间接记录。

### 4.3 `--bare` 必须在 commander 求值前设置环境变量

注释里写明了：

> `--bare`: set SIMPLE early so gates fire during module eval / commander option building (not just inside the action handler).

因为某些 gate 检查依赖 `process.env.CLAUDE_CODE_SIMPLE`，而 commander 的 `--bare` 解析只有在 `cliMain()` 内执行。如果把设置放在 action handler 里太迟，模块导入期间的 gate 会拿到未设置的值。

### 4.4 `templatesMain` 用 `process.exit(0)` 而非 `return`

注释里写：

> `process.exit` (not `return`) — mountFleetView's Ink TUI can leave event loop handles that prevent natural exit.

Ink TUI 挂上去的 stdin raw mode、事件循环句柄可能阻止 Node 自然退出。`process.exit(0)` 强制结束。其他 fast-path 大多用 `return`——因为它们的执行路径短，没有 TUI 句柄残留。

### 4.5 growth gate 顺序：auth → disabled → version → policy

bridge 入口（行 152–175）做了 4 层 gate：

```
1. getClaudeAIOAuthTokens()?.accessToken ── 无 token 拒绝
2. await getBridgeDisabledReason()       ── GrowthBook 内核 gate
3. checkBridgeMinVersion()               ── 最小版本检查
4. waitForPolicyLimitsToLoad() + isPolicyAllowed("allow_remote_control")
```

注释强调 **第 1 步必须在第 2 步之前**：

> Auth check must come before the GrowthBook gate check — without auth, GrowthBook has no user context and would return a stale/default false.

如果 GrowthBook 在没有用户上下文时返回 stale 的 false，会误判 Bridge 不可用，但用户其实可以登录启用——必须在拿到 token 之后再读 GrowthBook。

`getBridgeDisabledReason` 虽然内部 await GB init 返回 fresh 值，但 GrowthBook 的 init 仍然需要 auth header，所以这个顺序是 auth-required chain。

### 4.6 `feature()` 必须保持 inline

每一处 `feature("XXX")` 都写在 if 条件里，不能被拆出函数或变量：

```typescript
if (feature("BRIDGE_MODE") && args[0] === "remote-control")
```

如果写成：

```typescript
const bridgeMode = feature("BRIDGE_MODE")
if (bridgeMode && args[0] === "remote-control")
```

bun bundler 的 DCE 就识别不出来 `feature("BRIDGE_MODE")` 在构建期应被替换为字面量，整个分支会保留在最终 bundle 里。

---

## 五、本项目对该文件的"主要完成的工作"

> 详见 [`docs/own/RECORD.md`](./RECORD.md) §3.3，本节摘要其与 cli.tsx 相关的部分。

### 5.1 反编译后 polyfill 注入

原版的 `cli.tsx` 假定 bun bundler 已经把 `feature()`/`MACRO`/`BUILD_*` 替换为常量。本项目还原后无法直接依赖 `bun:bundle` 宏，所以文件顶端全部以 **运行时模拟** 实现。这是最关键的"必须改"步骤——否则：

- `feature()` 引用 → ReferenceError
- `MACRO.VERSION` 引用 → ReferenceError
- 所有 `if (feature(...))` 在运行时仍会被求值，导致原版 DCE 的死分支（如 `bridge`、`daemon`）执行错误路径

### 5.2 命令行选项微调

由反编译错误产生的 commander 短标志 `-d2e, --debug-to-stderr` 已知出现非法短串。`cli.tsx` 内部不解析这些标志，由 `main.tsx` 处理；本项目的 commander 修复在 `main.tsx` 而非此文件。

### 5.3 顶层副作用在 Bun 下的正确性

由于 Bun 采用 ESM 严格解析、顶层副作用执行顺序与 Node 略有差异（特别是 `process.env` 写入），本文件的 4 处顶层赋值（env、CCRP max-old-space、Corepack、Ablation）都在 Bun 上验证过：

- `process.env.COREPACK_ENABLE_AUTO_PIN = "0"` 在 Bun ESM top-level 也即时生效
- `--max-old-space-size` 拼接 Bun 也接受
- Ablation 块因 `feature()` = false 被 Bun DCE，无副作用

### 5.4 已 DCE 的 dead branch 保留（不删）

虽然 `feature("DAEMON")`、`feature("BRIDGE_MODE")` 等都恒为 `false`，源码里 11 处 fast-path 分支体仍保留：

- 一是 **可读性**——告诉读者原版支持这些入口
- 二是 **searchability**——开发者搜 `--claude-in-chrome-mcp` 时能跳到入口处
- 三是 **未来兼容**——若日后启用对应 feature flag，这些分支立即生效，不必重新编写

它们在构建产物 `dist/cli.js` 中不存在（被 DCE），运行时无任何开销。

---

## 六、流程图（启动到 REPL）

```
Bun 进程启动
   │
   ▼
cli.tsx 顶层执行 ── polyfill ── env 设置
   │
   ▼
main()
   │
   ├─ fast-path 命中 ──→ 直接返回
   │
   └─ 兜底分支
        │
        ▼
      startCapturingEarlyInput()
        │
        ▼
      import("../main.jsx")
        │
        ├─ enableConfigs / initSinks / getClaudeAIOAuthTokens
        ├─ Commander 解析 argv
        ├─ 根据 commands 路由
        │     ├─ 无命令 → 启动 REPL（TUI）
        │     ├─ -p / --print → pipe mode 跑 query() 并退出
        │     ├─ update / login / logout → 单一动作退出
        │     └─ 其他子命令 → 各自 handler
        │
        ▼
      await cliMain();   ← REPL / pipe mode 完成
        │
        ▼
      process exit
```

---

## 七、相关文件指针

| 文件 | 关系 |
|------|------|
| `src/main.tsx` | 兜底分支的真正 CLI 主体（Commander + REPL） |
| `src/utils/startupProfiler.ts` | `profileCheckpoint()` 实现 |
| `src/utils/earlyInput.ts` | `startCapturingEarlyInput()` 实现 |
| `src/entrypoints/init.ts` | `enableConfigs()` / `initSinks()` 实现 |
| `src/types/global.d.ts` | `MACRO` / `BUILD_*` 的 TypeScript 类型声明 |
| `src/types/internal-modules.d.ts` | `bun:bundle` 占位（cli.tsx 不直接用） |
| `docs/architecture/17-architecture.md` §3.1 / §4.1 | 入口与三层门控的整体论述 |
| `docs/own/RECORD.md` §3.3 | 该文件的运行时修复历史 |
