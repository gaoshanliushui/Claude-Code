# MCP 系统全景剖析 —— 从配置发现到工具调用

> 本文对照源码，完整论述 Claude Code 的 **MCP（Model Context Protocol）系统**是怎么运作的：MCP server 从多源配置被发现、经过策略过滤与去重、按 transport 建立连接、发现 tools/resources/prompts/skills、被包装成统一的 `mcp__server__tool` 工具暴露给模型、被调用时经权限/认证/会话过期重连/输出截断，以及 enterprise 策略、OAuth 认证、elicitation、SDK 进程内 transport 等横切机制。
>
> 核心源码：
> - `src/services/mcp/types.ts`（配置与连接状态的 zod schema + 类型）
> - `src/services/mcp/config.ts`（多源配置加载、策略过滤、去重、增删）
> - `src/services/mcp/client.ts`（transport 创建、连接、工具发现、调用、结果处理）
> - `src/services/mcp/useManageMCPConnections.ts`（React hook 编排连接生命周期与 AppState 状态）
> - `src/services/mcp/auth.ts`（OAuth / XAA 认证 Provider）
> - `src/services/mcp/elicitationHandler.ts`（elicitation 交互）
> - `src/services/mcp/mcpStringUtils.ts`（`mcp__server__tool` 命名）
> - `src/tools/MCPTool/MCPTool.ts`（基础工具模板）

---

## 0. MCP 在 Claude Code 里的定位

MCP 是一个标准协议（`@modelcontextprotocol/sdk`），让外部“服务器”向 Claude Code 暴露 **tools / resources / prompts / elicitation**。Claude Code 作为 MCP **客户端**：为每个配置的 server 建立一条连接，把 server 暴露的 tools 包装成内部 `Tool` 对象（名字形如 `mcp__server__tool`），混进主工具集，模型就能像调内建工具一样调它们。

MCP 与 Skill 系统的关键区别：**MCP tool 是真工具**（`isMcp: true` 的 `Tool`，模型直接 `tool_use` 调用，由 server 进程执行）；**Skill 是 prompt**（`type: 'prompt'` 的 `Command`，触发时把 markdown 注入对话）。但 MCP server 也能暴露 **prompts**，这些在源码里被当作一类特殊的 skill 处理（见 §5）。

---

## 1. 配置：MCP server 从哪里来

### 1.1 七种 transport 与 scope（types.ts）

`McpServerConfigSchema` 是 8 种 config 的 union（types.ts:124）：

| `type` | 含义 | 关键字段 |
|--------|------|----------|
| `stdio`（可省略） | 子进程，stdin/stdout 通信 | `command`、`args`、`env` |
| `sse` | Server-Sent Events | `url`、`headers`、`oauth` |
| `http` | Streamable HTTP | `url`、`headers`、`oauth` |
| `ws` | WebSocket | `url`、`headers` |
| `sse-ide` / `ws-ide` | IDE 扩展专用（内部） | `url`、`ideName` |
| `sdk` | SDK 进程内 transport 占位 | `name` |
| `claudeai-proxy` | claude.ai 网页连接器代理 | `url`、`id` |

每个 config 被包成 `ScopedMcpServerConfig = McpServerConfig & { scope, pluginSource? }`（types.ts:163）。`ConfigScope` 有 7 种（types.ts:10）：`local` / `user` / `project` / `dynamic` / `enterprise` / `claudeai` / `managed`。

连接运行时存在 6 种状态 `MCPServerConnection`（types.ts:221）：`connected` / `failed` / `needs-auth` / `pending` / `disabled`。

### 1.2 多源加载：`getMcpConfigsByScope`（config.ts:888）

各 scope 的来源：

- **project**（`.mcp.json`）：从 cwd **逐级向上到根**，逐个目录找 `.mcp.json`，**从根向 cwd 合并**（靠近 cwd 的覆盖父目录），`getProjectMcpConfigsFromCwd` 则只看当前目录（供增删改）。
- **user**：全局 config 的 `mcpServers`。
- **local**：当前项目 local config。
- **enterprise**：`getManagedFilePath()/managed-mcp.json`（policy/企业托管）。

每个文件经 `parseMcpConfigFromFilePath`（config.ts:1384）→ `safeParseJSON` → `McpJsonConfigSchema` 校验 → `parseMcpConfig`（config.ts:1297）逐 server 校验 + **环境变量展开**（`expandEnvVars`，`${VAR}`）+ Windows `npx` 需 `cmd /c` 包裹的告警。

### 1.3 聚合：`getClaudeCodeMcpConfigs`（config.ts:1071）

这是启动主路径（不含 claude.ai）。逻辑：

1. **企业独占**：若 `doesEnterpriseMcpConfigExist()`（memoize，config.ts:1470），**只用 enterprise servers**，其余全部丢弃（企业不希望用户自加 server）。
2. **plugin-only 锁**：`isRestrictedToPluginOnly('mcp')` 为真则 user/project/local 全部置空，只保留 plugin。
3. 加载 `loadAllPluginsCacheOnly()` + 各 enabled plugin 的 `getPluginMcpServers` → `pluginMcpServers`。
4. **project server 需审批**：只有 `getProjectMcpServerStatus(name) === 'approved'` 的才进 `approvedProjectServers`（防 `.mcp.json` 投毒）。
5. **去重**（§2）。
6. **合并优先级**：`plugin < user < project < local`（`Object.assign`，后者覆盖前者）。
7. 最终再过一遍 `isMcpServerAllowedByPolicy` 策略过滤。

返回 `{ servers, errors }`。`getAllMcpConfigs`（config.ts:1258）在其上叠加 **claude.ai connector**（`fetchClaudeAIMcpConfigsIfEligible`，网络拉取，可能慢），与 manual server 去重后以最低优先级合并。

### 1.4 增删：`addMcpConfig` / `removeMcpConfig`（config.ts:625 / 769）

`/mcp add` 走这里。校验：名字仅 `[a-zA-Z0-9_-]`、保留名（`claude-in-chrome`、computer-use）、enterprise 独占时禁止添加、schema 校验、denylist/allowlist 策略、目标 scope 内不重名。`project` scope 写 `.mcp.json`（`writeMcpjsonFile` 原子写：tmp + datasync + rename，保留权限），`user`/`local` 写对应 config。

### 1.5 启用/禁用：`isMcpServerDisabled` / `setMcpServerEnabled`（config.ts:1528 / 1553）

- 普通 server：opt-out，记进 `disabledMcpServers`。
- **默认禁用的内建 server**（如 computer-use）：opt-in，需显式进 `enabledMcpServers`。
- 禁用态 server 在连接阶段直接置 `type: 'disabled'`，**不发任何网络请求**。

---

## 2. 策略过滤与去重

### 2.1 企业策略：`isMcpServerAllowedByPolicy`（config.ts:417）

三维度匹配（name / command 数组 / URL 通配）：

- **denylist 绝对优先**（`isMcpServerDenied`，合并所有来源——用户随时可自我拒绝）。
- **allowlist**：`undefined` = 不限制；`[]` = 全禁；有 command/url 条目时，stdio 必须 match 某 command、remote 必须 match 某 URL；否则按 name。
- `allowManagedMcpServersOnly` 时 allowlist 只读 managed settings。
- `filterMcpServersByPolicy`（config.ts:536）批量过滤，供 `--mcp-config` 与 SDK `mcp_set_servers` 用；**sdk 类型豁免**（无 url/command，是 SDK 占位）。

### 2.2 内容去重：`getMcpServerSignature`（config.ts:202）

计算签名：stdio → `stdio:JSON([command,...args])`；remote → `url:unwrapCcrProxyUrl(url)`（剥离 CCR 代理前缀，把被改写的 proxy URL 还原成 vendor URL 再比对）。两步去重：

- `dedupPluginMcpServers`（config.ts:223）：plugin server 与 manual 重叠时 **manual 胜**；plugin 之间 **先加载的胜**。被抑制的进 `mcp-server-suppressed-duplicate` 错误上浮到 `/plugin` UI。
- `dedupClaudeAiMcpServers`（config.ts:281）：claude.ai connector 与 **enabled** manual server 重叠时抑制（disabled manual 不算，否则两个都不跑）。

> 命名空间化（plugin 是 `plugin:x:y`、claude.ai 是 `claude.ai <Name>`）让它们 key 不冲突，所以必须靠内容签名去重。

---

## 3. 连接生命周期

### 3.1 编排入口：`useManageMCPConnections`（useManageMCPConnections.ts:143）

React hook，在 REPL 挂载时（`loadAndConnectMcpConfigs`，:861）分两阶段：

- **Phase 1**：`getClaudeCodeMcpConfigs(dynamicMcpConfig, claudeaiPromise)` 拿到 claudeCode 配置（plugin 在此被去重抑制，避免与 Phase 2 的 claude.ai connector 同时连同一 server）。过滤掉 disabled，调 `getMcpToolsCommandsAndResources(onConnectionAttempt, enabledConfigs)`。
- **Phase 2**：await `claudeaiPromise`，去重后把 claude.ai server 先以 `pending` 进 UI，再 `getMcpToolsCommandsAndResources` 连接。
- `claudeaiPromise` 在 Phase 1 启动 fetch、Phase 2 await（memoized，不二次拉取），与 `loadAllPluginsCacheOnly()` **重叠**而非串行。
- 依赖 `authVersion`（登录/登出后重连）、`pluginReconnectKey`（`/reload-plugins` 后重连）。

**批量更新 AppState**（:216 `flushPendingUpdates`）：16ms 时间窗内累积各 server 的 `onConnectionAttempt` 回调，一次 `setAppState` flush——避免 N 个 server 各触发一次重渲染。按 server 前缀 `getMcpPrefix(name)` 先 reject 旧的同 server tools/commands 再追加新的。

### 3.2 并发与连接：`getMcpToolsCommandsAndResources`（client.ts:2226）

- disabled server 直接置 `type:'disabled'`，不连。
- **needs-auth 短路**（:2307）：remote server 若 15min TTL 内刚 401，或探测过但无 token，直接置 `needs-auth` 并只挂 `createMcpAuthTool`（让用户 `/mcp` 触发认证），避免每次启动都打失败的网络请求。
- **分两组并发**（:2264）：local（stdio/sdk，进程 spawn 资源重）用低并发 `getMcpServerConnectionBatchSize()`；remote（网络）用高并发。`processBatched` 控制并发。
- 每个 server `connectToServer(name, config, stats)` → 成功后并发 `fetchToolsForClient` / `fetchCommandsForClient` / `fetchMcpSkillsForClient`（feature `MCP_SKILLS`）/ `fetchResourcesForClient`。
- 有 resources 的首个 server 额外挂 `ListMcpResourcesTool` + `ReadMcpResourceTool`（全局只挂一次，`resourceToolsAdded` 守卫）。
- `onConnectionAttempt({ client, tools, commands, resources })` 回调把结果灌进 AppState。

### 3.3 transport 创建与连接：`connectToServer`（client.ts:595，memoize）

按 `serverRef.type` 分支构造 transport：

| type | transport | 认证 |
|------|-----------|------|
| `sse` | `SSEClientTransport` | `ClaudeAuthProvider` + 合并 headers；EventSource 用**不带 timeout 的 fetch**（长连接），普通请求用 `wrapFetchWithTimeout` |
| `sse-ide` | `SSEClientTransport`（无认证） | IDE 锁文件 token |
| `ws` / `ws-ide` | `WebSocketTransport`（Bun 原生 WS / node `ws`） | headers；ws-ide 用 `X-Claude-Code-Ide-Authorization` |
| `http` | `StreamableHTTPClientTransport` | `ClaudeAuthProvider`；有 OAuth token 时不用 session-ingress token |
| `claudeai-proxy` | `StreamableHTTPClientTransport` | claude.ai OAuth + `X-Mcp-Client-Session-Id` |
| `sdk` | 抛错——交给 `setupSdkMcpClients`（§6） | — |
| `stdio`/缺省 | `StdioClientTransport` | `subprocessEnv() ∪ env`；`stderr: 'pipe'`（防污染 UI）；支持 `CLAUDE_CODE_SHELL_PREFIX` |
| 特殊：`claude-in-chrome` / computer-use | **进程内** MCP server（`createLinkedTransportPair`，`InProcessTransport`） | 避免 spawn ~325MB 子进程 |

构造 `new Client({name:'claude-code',...}, {capabilities:{roots, elicitation}})`（:985），声明 `roots`（让 server 问工作目录）与 `elicitation`（空对象——给 Java SDK 留兼容）。

设置 `ListRootsRequestSchema` handler 返回 `file://<originalCwd>`（:1009）。

**连接超时**（:1048）：`Promise.race([client.connect(transport), timeout(getConnectionTimeoutMs())])`，超时关 transport 并抛 telemetry-safe 错误。

**stderr 累积**（:966）：stdio server 的 stderr 进 64MB 上限的 `stderrOutput`，连接成功后 `logMCPError` 一次再清空。

### 3.4 失败与认证降级

- `UnauthorizedError`（SSE/HTTP）→ `handleRemoteAuthFailure(name, serverRef, type)` 置 `needs-auth`，挂 `createMcpAuthTool`。
- claudeai-proxy 401 → 同样降级 needs-auth。
- IDE 连接失败/成功打 `tengu_mcp_ide_server_connection_*` 遥测。

### 3.5 掉线检测与重连（:1216–1402）

SDK 的 transport 在连接失败时调 `onerror` **但不调 `onclose`**，而 CC 用 `onclose` 触发重连。bridge 方案：

- `client.onerror` 增强：识别 `isTerminalConnectionError`（ECONNRESET/ETIMEDOUT/EPIPE/...），连续 `MAX_ERRORS_BEFORE_RECONNECT=3` 次后 `closeTransportAndRejectPending`。
- HTTP/claudeai-proxy 检测 **session 过期**（404 + JSON-RPC -32001，`isMcpSessionExpiredError`）→ 关 transport 让 pending tool call 失败，下次重连拿新 session id。
- SSE 重连耗尽（"Maximum reconnection attempts"）→ 关 transport。
- `client.onclose`：清 `connectToServer.cache` + `fetchToolsForClient/Resources/Commands/Skills` 的 LRU cache（按 server name key）——重连产生新连接对象，不清缓存会返回旧 tools。

`ensureConnectedClient`（:1688）在每次 callTool 前调一次 `connectToServer`（memoize 命中即返回缓存的 connected client；缓存因掉线被清则重新连）。

`reconnectMcpServerImpl`（:2137）支持显式重连（`/mcp` 触发）。

---

## 4. 工具发现与命名

### 4.1 发现：`fetchToolsForClient`（client.ts:1743，LRU memoize）

- 仅当 `client.capabilities?.tools` 存在才 `tools/list`。
- `recursivelySanitizeUnicode(result.tools)`（防异常 unicode 注入 prompt）。
- SDK server 在 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` 时 **跳过 `mcp__` 前缀**，让 MCP tool 能按原名覆盖内建工具（`mcpInfo` 仍用于权限检查）。
- 每个 tool 包装成 `{...MCPTool, name, mcpInfo, isMcp:true, ...}`，覆盖 `description`/`prompt`/`checkPermissions`/`call` 等。
- 读取 `tool._meta` 的 `anthropic/searchHint`（折叠空白）、`anthropic/alwaysLoad`（是否总加载）。
- 读 `tool.annotations`：`readOnlyHint` → `isConcurrencySafe`/`isReadOnly`；`destructiveHint`；`openWorldHint`。

### 4.2 命名：`mcp__server__tool`（mcpStringUtils.ts）

- `buildMcpToolName(server, tool)` = `mcp__${normalize(server)}__${normalize(tool)}`。
- `mcpInfoFromString` 反解（已知限制：server 名含 `__` 会误解析）。
- `getToolNameForPermissionCheck`：MCP tool 用全限定名做权限匹配，防 deny 规则误伤同名 MCP 替代工具。
- `getMcpDisplayName` / `extractMcpToolDisplayName`：UI 显示剥前缀与 `(MCP)` 后缀。

### 4.3 基础模板：`MCPTool`（MCPTool.ts:27）

`MCPTool` 是占位 `buildTool({ isMcp:true, name:'mcp', ... })`，所有真正字段都在 `fetchToolsForClient` 里被覆盖。`inputSchema` 用 `z.object({}).passthrough()`（MCP tool 自带 schema）。`maxResultSizeChars: 100_000`。

### 4.4 resources / prompts / skills

- `fetchResourcesForClient`（:2000）：`resources/list`，挂 `ListMcpResourcesTool`/`ReadMcpResourceTool`。
- `fetchCommandsForClient`（:2033）：`prompts/list`，server 暴露的 prompt 被包成 `Command`（`type:'prompt'`，`loadedFrom:'mcp'`）。这些是 **MCP-as-skill**——能被 `SkillTool` 调（见 `getAllCommands` 过滤 `loadedFrom==='mcp'`），但模型猜名也能调到。
- `fetchMcpSkillsForClient`（feature `MCP_SKILLS`）：从 `skill://` resources 发现 skill，走 `mcpSkillBuilders.ts` registry（`createSkillCommand`/`parseSkillFrontmatterFields`，绕开 import 环）。

---

## 5. 工具调用执行

模型 `tool_use` 一个 `mcp__server__tool` → 走通用 tool 执行 → 该 tool 的 `call`（client.ts:1833）：

1. 发 `started` progress。
2. **session 过期重试**（`MAX_SESSION_RETRIES=1`）：`ensureConnectedClient` 拿连接 → `callMCPTool`；若 `McpSessionExpiredError`，清缓存重连重试一次。
3. `callMCPToolWithUrlElicitationRetry`（:2813）：处理 `-32042` URL elicitation——展示 URL、等 completion 通知、重试。
4. `callMCPTool`（:3029）：
   - `client.callTool({name, arguments, _meta:{'claudecode/toolUseId'}})`，传 `signal` + `timeout=getMcpToolTimeoutMs()` + `onprogress`（透传 server progress）。
   - `Promise.race` 自带 timeout（防 SDK 内部 timeout 失效，如 SSE 流中断）。
   - 每 30s log 一次"still running"。
   - `result.isError === true` → 抛 `McpToolCallError`（带 `_meta`，SDK 消费者仍能收到）。
   - 401/`UnauthorizedError` → 抛 `McpAuthError`（被上层捕获置 needs-auth）。
5. `processMCPResult`（:2720）处理输出：
   - `transformMCPResult` 转换 content（image 下采样、blob 持久化等）。
   - IDE tool 直通（不进模型）。
   - 小输出直返；**大输出**（`mcpContentNeedsTruncation`）：若 `ENABLE_MCP_LARGE_OUTPUT_FILES` 开则 `persistToolResult` 存盘 + 返回"读取指引"（`getLargeOutputInstructions`），否则截断；含 image 的回退截断。

### 5.1 权限

MCP tool 的 `checkPermissions`（:1814）默认 `passthrough`（交通用权限系统按 `mcp__server__tool` 名匹配 allow/deny 规则），建议规则加进 localSettings。auto-mode 分类器用 `mcpToolInputToAutoClassifierInput`（`k=v` 拼接）。

---

## 6. SDK MCP（进程内）

`setupSdkMcpClients`（:3262）：SDK 嵌入场景下，server 是 SDK 自己（在宿主进程里）。用 `SdkControlClientTransport`（不走网络，通过 `sendMcpMessage` 回调与宿主通信），`client.connect` 后同样 `fetchToolsForClient`。工具走标准 `mcp__` 前缀（除非 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX`）。

`vscodeSdkMcp.ts` 特化 `claude-vscode`：存 client 引用，收 `log_event` 通知转 telemetry，发 `file_updated` 通知 VSCode 文件变更，下发 experiment gates / auto-mode state。

`areMcpConfigsAllowedWithEnterpriseMcpConfig`（config.ts:1494）：企业 MCP 独占下，只允许 `claude-vscode` 这一种 sdk server 例外。

---

## 7. 认证（auth.ts）

`ClaAuthProvider` 实现 SDK 的 `OAuthClientProvider`：

- `discoverAuthorizationServerMetadata` / `discoverOAuthServerInfo` 发现 OAuth metadata（PKCE）。
- token 存 **secure storage**（macOS keychain 等），`clearKeychainCache` 失效缓存。
- 本地起回调 server（`oauthPort.ts` `findAvailablePort` + `buildRedirectUri`）收 authorization code。
- `wrapFetchWithStepUpDetection`：检测 403 step-up，触发重新授权。
- **XAA（Cross-App Access）**（`xaa.ts` / `xaaIdpLogin.ts`）：server 标 `oauth.xaa:true` 时走 IdP（OIDC）换 token，`performCrossAppAccess` 做 token 交换。
- `hasMcpDiscoveryButNoToken` + `isMcpAuthCached`（15min TTL）：避免反复探测无 token 的 server。
- `McpAuthError` 在 callTool 401 时上抛，置 server 为 `needs-auth`，挂 `createMcpAuthTool` 引导用户 `/mcp` 认证。

---

## 8. Elicitation（elicitationHandler.ts）

server 可向用户发起表单/URL 交互（`elicitation` capability）。`registerElicitationHandler`（:68）设 `ElicitRequestSchema` handler：

- `mode: 'form'` vs `'url'`。
- form 走 `executeElicitationHooks`（可被 hook 程序化应答）否则弹 UI。
- url 模式：展示 URL，等 `ElicitationCompleteNotificationSchema`（server 完成通知），`onWaitingDismiss` 支持 retry/cancel。
- 错误重试（-32042）由 `callMCPToolWithUrlElicitationRetry` 驱动。
- 连接初始化期先装一个默认返回 `cancel` 的 handler（:1191），等 `onConnectionAttempt` 里 `registerElicitationHandler` 覆盖。

---

## 9. 端到端时序

```
启动 (REPL mount)
 └─ useManageMCPConnections(dynamicMcpConfig)
     └─ loadAndConnectMcpConfigs
         ├─ Phase1: getClaudeCodeMcpConfigs(dynamic, claudeaiPromise)
         │    ├─ getMcpConfigsByScope × {enterprise,user,project,local}
         │    │    └─ parseMcpConfigFromFilePath + expandEnvVars + schema校验
         │    ├─ loadAllPluginsCacheOnly + getPluginMcpServers → pluginMcpServers
         │    ├─ project server 审批过滤 (getProjectMcpServerStatus)
         │    ├─ dedupPluginMcpServers (签名去重, manual > plugin)
         │    └─ 合并 plugin<user<project<local + isMcpServerAllowedByPolicy
         ├─ filter disabled → getMcpToolsCommandsAndResources(onConnectionAttempt)
         │    └─ per server: connectToServer (memoize)
         │         ├─ 按 type 造 transport (sse/http/ws/stdio/in-process/sdk-throws)
         │         ├─ new Client({roots, elicitation})
         │         ├─ client.connect(transport) ‖ timeout(getConnectionTimeoutMs)
         │         ├─ UnauthorizedError → needs-auth + createMcpAuthTool
         │         ├─ onerror: 终端错误 3 次/session 过期/SSE 耗尽 → close
         │         └─ onclose: 清 connectToServer + fetch* LRU cache
         │    └─ 并发 fetchTools/Commands/Skills/Resources → onConnectionAttempt
         │         └─ flushPendingUpdates (16ms 批量) → setAppState.mcp.{clients,tools,commands,resources}
         └─ Phase2: await claudeaiPromise → dedupClaudeAiMcpServers → pending 进 UI → 连接

模型 tool_use mcp__server__tool
 └─ MCPTool.call (client.ts:1833)
     ├─ ensureConnectedClient (memoize 命中/重连)
     ├─ callMCPToolWithUrlElicitationRetry (-32042 → 展示URL→等通知→重试)
     ├─ callMCPTool: client.callTool(name,args,_meta,signal,timeout,onprogress)
     │    ├─ isError:true → McpToolCallError(带_meta)
     │    ├─ 401 → McpAuthError → needs-auth
     │    └─ processMCPResult: transform → 截断/存盘(getLargeOutputInstructions) → MCPToolResult
     └─ return { data: content } → tool_result
```

`/mcp` 命令（`local-jsx`，`immediate`）渲染管理 UI，支持 `enable|disable [server]`，调 `setMcpServerEnabled` 触发重连。

---

## 10. 关键设计取舍与不变量

- **企业独占绝对优先**：`doesEnterpriseMcpConfigExist()` 为真则只用 enterprise servers，连 user 自加的都不要；企业 MCP 独占下 SDK server 仅 `claude-vscode` 例外。
- **denylist 合并所有来源、allowlist 受 `allowManagedMcpServersOnly` 约束**：用户始终能自我拒绝；allowlist 可被企业锁定只读 managed。
- **内容签名去重**：plugin/claude.ai 与 manual 命名空间隔离不撞 key，靠 `stdio:cmd` / `url:unwrapCcrProxyUrl` 签名识别“同一底层 server”，manual 优先，省 token（避免 `mcp__slack__*` 与 `mcp__claude_ai_Slack__*` 双连）。
- **disabled 不发网络请求**：连接阶段直接置 `disabled`，避免无用连接；needs-auth 15min TTL + 无 token 探测记忆避免反复 401。
- **local/remote 分组并发**：stdio 进程 spawn 重，低并发；remote 纯网络，高并发。
- **掉线 bridge**：SDK 的 onerror 不触发 onclose，CC 用终端错误计数 + session 过期检测 + SSE 耗尽检测主动 `client.close()`，让 pending callTool 失败并清缓存重连。
- **onclose 清所有 fetch* LRU**：重连产生新连接对象，按 server name key 的 tools/resources 缓存必须清，否则返回旧数据。
- **输出三段处理**：直返 / 存盘+读取指引（`ENABLE_MCP_LARGE_OUTPUT_FILES`）/ 截断；含 image 回退截断（持久化 JSON 破坏压缩）。
- **进程内 server**：Chrome/computer-use MCP 跑在进程内（`createLinkedTransportPair`），省 ~325MB 子进程。
- **MCP tool 是真工具**：与 skill 的 prompt 注入不同，MCP tool 由 server 进程执行，结果经 `processMCPResult` 进 tool_result；但 MCP **prompts** 被当 skill 处理（`loadedFrom:'mcp'`），可经 `SkillTool` 调且**不执行 shell**（远端不可信）。
- **unicode sanitize**：server 返回的 tool 数据 `recursivelySanitizeUnicode`，防异常字符注入 deferred-tool 列表。
- **权限用全限定名**：`getToolNameForPermissionCheck` 用 `mcp__server__tool`，防 deny 规则误伤同名 MCP 替代工具。

---

## 11. 关联模块速查

| 模块 | 职责 |
|------|------|
| `src/services/mcp/types.ts` | config schema（8 种 transport）、`MCPServerConnection` 6 态、`MCPCliState` |
| `src/services/mcp/config.ts` | 多源加载、`expandEnvVars`、策略过滤、签名去重、增删、enable/disable |
| `src/services/mcp/client.ts` | transport 创建、`connectToServer`、`fetchToolsForClient`、`callMCPTool`、`processMCPResult`、掉线重连、`setupSdkMcpClients` |
| `src/services/mcp/useManageMCPConnections.ts` | React hook 编排、两阶段加载、批量 AppState 更新、reconnect timer |
| `src/services/mcp/auth.ts` | `ClaudeAuthProvider`（OAuth/PKCE）、XAA、step-up、token secure storage |
| `src/services/mcp/elicitationHandler.ts` | elicitation form/url 交互、完成通知、错误重试 |
| `src/services/mcp/mcpStringUtils.ts` | `mcp__server__tool` 命名与反解、权限名 |
| `src/services/mcp/normalization.ts` | server/tool 名归一化 |
| `src/services/mcp/headersHelper.ts` | 静态 + 动态 headers 合并 |
| `src/services/mcp/envExpansion.ts` | `${VAR}` 展开 |
| `src/services/mcp/InProcessTransport.ts` | `createLinkedTransportPair`（进程内 server） |
| `src/services/mcp/SdkControlTransport.ts` | SDK 进程内 transport（不走网络） |
| `src/services/mcp/vscodeSdkMcp.ts` | `claude-vscode` 特化（通知/门控下发） |
| `src/services/mcp/claudeai.ts` | claude.ai connector 配置拉取与去重 |
| `src/services/mcp/officialRegistry.ts` | 官方 registry |
| `src/services/mcp/channelPermissions.ts` | channel 权限 relay（KAIROS） |
| `src/tools/MCPTool/MCPTool.ts` | 基础工具模板（字段被 client.ts 覆盖） |
| `src/tools/MCPTool/prompt.ts` + `UI.tsx` + `classifyForCollapse.ts` | 工具描述占位、渲染、折叠分类 |
| `src/tools/McpAuthTool/McpAuthTool.ts` | `createMcpAuthTool`（needs-auth 引导） |
| `src/tools/ListMcpResourcesTool` / `ReadMcpResourceTool` | resources 访问工具 |
| `src/utils/mcpValidation.ts` | 输出大小估计、截断判定 |
| `src/utils/mcpOutputStorage.ts` | 大输出持久化、blob 处理 |
| `src/utils/mcpWebSocketTransport.ts` | `WebSocketTransport`（Bun/node 适配） |
| `src/skills/mcpSkills.ts` | `fetchMcpSkillsForClient`（从 `skill://` resource 发现 skill） |
| `src/commands/mcp/mcp.tsx` | `/mcp` 管理 UI（enable/disable/connect） |
| `src/utils/plugins/mcpPluginIntegration.ts` | `getPluginMcpServers`（插件 MCP） |
| `src/utils/plugins/mcpbHandler.ts` | MCPB（DXT 包）manifest 与 user config |
