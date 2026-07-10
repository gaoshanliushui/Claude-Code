# MCP 客户端实现剖析 —— `client.ts` 逐层拆解

> 本文聚焦单个文件 `src/services/mcp/client.ts`（约 3358 行，是 MCP 子系统里最大、最复杂的文件），逐层论述它**完成了哪些工作**以及**每条路径的运行流程**。
>
> 它的上层编排（`useManageMCPConnections`）、配置来源（`config.ts`）、认证 Provider（`auth.ts`）等横切机制见姊妹文档 [`09-mcp-system-deep-dive.md`](./09-mcp-system-deep-dive.md)。本文只在与 client.ts 交界处点到为止。
>
> 阅读地图：
> - §1 文件职责边界 —— client.ts 到底负责什么
> - §2 错误类型体系 —— 4 类自定义错误如何驱动重连/重认证
> - §3 needs-auth 磁盘缓存 —— 避免反复探测无 token 的 server
> - §4 fetch 包装层 —— 超时、Accept 头、claude.ai 代理 401 重试
> - §5 `connectToServer` —— 7 种 transport 的创建、连接、掉线检测、进程清理
> - §6 连接缓存与生命周期 —— memoize / ensureConnectedClient / reconnect
> - §7 工具·资源·命令发现 —— MCP 原语到内部 `Tool`/`Command` 的转换
> - §8 批量连接编排 —— 本地/远程分池并发
> - §9 工具调用执行 —— 超时、会话过期重试、URL elicitation 重试
> - §10 结果转换与大输出处理 —— 多模态内容、落盘、截断
> - §11 SDK 进程内 MCP
> - §12 关键设计取舍与不变量

---

## 1. 文件职责边界

`client.ts` 是 MCP **客户端侧的执行引擎**。它不决定“连哪些 server”（那是 `config.ts` 的事），也不管 React 状态编排（那是 `useManageMCPConnections` 的事）。它负责把**一个已经确定的 server 配置**变成一条**可用连接 + 一组内部工具**，并在工具被调用时把请求送到 server、把结果转回模型能吃的 `ContentBlockParam[]`。

按导出符号归类，它完成了 6 大块工作：

| 工作块 | 关键导出 | 作用 |
|--------|----------|------|
| **连接建立** | `connectToServer`（memoize）、`getServerCacheKey` | 依 `type` 选 transport，连接，装掉线检测与清理 |
| **连接缓存** | `ensureConnectedClient`、`clearServerCache`、`reconnectMcpServerImpl`、`areMcpConfigsEqual` | 复用连接、失效重连、配置比对 |
| **原语发现** | `fetchToolsForClient`、`fetchResourcesForClient`、`fetchCommandsForClient`（均 LRU memoize） | 拉取 tools/resources/prompts，包装成内部对象 |
| **批量编排** | `getMcpToolsCommandsAndResources`、`prefetchAllMcpResources` | 本地/远程分池并发连接所有 server |
| **调用执行** | `callMCPToolWithUrlElicitationRetry`、`callMCPTool`（私有）、`callIdeRpc` | 发起工具调用，处理超时/会话过期/elicitation |
| **结果处理** | `transformResultContent`、`transformMCPResult`、`processMCPResult`、`inferCompactSchema` | 多模态归一化、大输出落盘、schema 推断 |
| **SDK 进程内** | `setupSdkMcpClients` | 通过控制通道连接 SDK 内嵌 server |

外加 4 类自定义错误（§2）与一套 needs-auth 磁盘缓存（§3）。

---

## 2. 错误类型体系（:153–208）

client.ts 定义了 4 种错误，它们不是普通异常，而是**控制流信号**——上层根据错误类型决定“重认证 / 重连 / 直接失败”。

| 错误 | 触发点 | 语义与后果 |
|------|--------|-----------|
| `McpAuthError`（导出） | 工具调用返回 401 / `UnauthorizedError`（:3205） | 携带 `serverName`；工具执行层捕获后把 client 状态改成 `needs-auth`，提示用户 `/mcp` 重认证 |
| `McpSessionExpiredError`（私有） | 工具调用检测到会话过期（:3239） | 连接缓存已被清空；调用方应拿新 client 重试（§9 的一次性重试）|
| `McpToolCallError_..._NOT_CODE_OR_FILEPATHS`（导出） | server 返回 `isError: true`（:3153） | 继承 `TelemetrySafeError`，携带 `_meta`（MCP spec 允许 error 结果带 `_meta`）|
| `isMcpSessionExpiredError()`（导出函数，:195） | —— | 判定“会话过期”：**HTTP 404 且 JSON-RPC `-32001`**，两个信号同时满足才算，避免把通用 404（URL 错、server 没了）误判成会话过期 |

**为什么 `_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 后缀**：这些错误消息会进遥测，后缀是给作者的强制断言——消息里只有 MCP 协议级文本，不含用户文件路径或代码，可安全上报。

`isMcpSessionExpiredError` 的实现细节（:195）值得记：SDK 把响应体文本嵌进了 `error.message`，MCP server 返回 `{"error":{"code":-32001,...}}`，所以它同时匹配 `"code":-32001` 与 `"code": -32001`（有无空格两种序列化）。

---

## 3. needs-auth 磁盘缓存（:259–318）

**要解决的问题**：远程 server（http/sse/claudeai-proxy）第一次连接返回 401 后，如果不记住，每次启动、每 15 分钟都会重新发起“connect → 401 → OAuth discovery”这一整套网络往返。print 模式还会 `await` 整批连接，拖慢启动。

**方案**：把 needs-auth 状态写进 `~/.claude/mcp-needs-auth-cache.json`（`getMcpAuthCachePath`），TTL 15 分钟（`MCP_AUTH_CACHE_TTL_MS`）。

三个函数配合，且都为并发做了保护：

- `getMcpAuthCache()`（:273）—— **读缓存 memoize 成单个 Promise**（`authCachePromise`）。批量连接时 N 个 server 并发调 `isMcpAuthCached()`，共享一次文件读，而不是读 N 次。没用 lodash memoize 是因为要能把整个缓存置空（invalidate），而不是按 key 删。
- `setMcpAuthCacheEntry()`（:295）—— **写操作串行化**（`writeChain` promise 链）。多个 server 在同一批里同时返回 401 时，避免并发 read-modify-write 竞争丢数据。写完把 `authCachePromise = null` 让下次读重新读盘。
- `clearMcpAuthCache()`（:313，导出）—— 置空读缓存并删文件（best-effort）。

消费点在 §8 的 `processServer`：命中缓存或“探测过但无 token”就直接跳过连接，塞一个 `needs-auth` 状态 + `createMcpAuthTool`。

---

## 4. fetch 包装层

client.ts 有两个 fetch 包装器，都在 transport 创建时套到底层 fetch 上。

### 4.1 `wrapFetchWithTimeout`（:494，导出）

给每个请求套一个**新鲜的 60 秒超时**（`MCP_REQUEST_TIMEOUT_MS`），并保证 Streamable-HTTP 要求的 `Accept` 头。三个要点：

1. **GET 豁免超时**（:500）——MCP transport 里 GET 是长连 SSE 流，套 60s 超时会把它掐断。（OAuth discovery 的 GET 走 auth.ts 里另一个带独立超时的 fetch。）
2. **补 Accept 头**（:509）——MCP Streamable HTTP spec 要求每个 POST 声明 `application/json, text/event-stream`，严格的 server 缺了会回 406。SDK 本来会设，但经过对象展开后某些运行时会丢，这里是“上线前最后一道 wrapper”，兜底补上。
3. **用 `setTimeout` 而非 `AbortSignal.timeout()`**（:518）——后者的内部定时器只有信号被 GC 才释放，Bun 里是懒 GC，即使请求几毫秒就完成，~2.4KB 原生内存也要挂满 60s。这里手动 `clearTimeout` + 关联父 signal，请求完成即清理。

### 4.2 `createClaudeAiProxyFetch`（:374，导出）

claude.ai 连接器代理专用。附加 OAuth bearer token，401 时**强制刷新后重试一次**。

关键正确性细节（:386）：重试时用的是**当次请求实际发送的那个 token**（`sentToken`），而不是重新读 `getClaudeAIOAuthTokens()`。因为并发 401 下，另一个连接器的 `handleOAuth401Error` 会清 memoize 缓存，若此时重读会拿到新 token，传给 `handleOAuth401Error` 发现“与 keychain 相同 → 返回 false → 跳过重试”，逻辑就错了。只有 token 真的变了（`tokenChanged`）才重试，避免为每个下游确实需要认证的连接器都白白多一次往返。

---

## 5. `connectToServer` —— 连接建立核心（:598–1650）

这是文件里最长的函数，用 `memoize`（lodash）包裹，缓存键 = `getServerCacheKey(name, serverRef)` = `${name}-${JSON.stringify(serverRef)}`（:584）。同名同配置只连一次；配置变了 key 变、重连。

> 源码顶部留了 ollie 的 TODO：memoization 让复杂度陡增、收益存疑。可视为“已知的技术债”。

流程分五段：

### 5.1 依 `type` 创建 transport（:622–965）

一条大 `if/else` 按 `serverRef.type` 分派：

| type | transport | 要点 |
|------|-----------|------|
| `sse` | `SSEClientTransport` | 带 `ClaudeAuthProvider`；POST fetch 套超时+step-up 检测；`eventSourceInit` 用**不带超时**的 fetch（SSE 长连不能掐）|
| `sse-ide` | `SSEClientTransport` | IDE server 不需认证；只在有 proxy dispatcher 时才定制 fetch |
| `ws-ide` / `ws` | `WebSocketTransport` | Bun 用 `globalThis.WebSocket`（支持 headers/proxy/tls），非 Bun 用 `ws` 包的三参构造（`createNodeWsClient`）；`ws` 带 session ingress token，日志里 `Authorization` 打码 |
| `http` | `StreamableHTTPClientTransport` | 大量诊断日志；先探测 `hasOAuthTokens`——有存储的 OAuth 就不覆盖 Authorization（SDK 在 authProvider 之后才 merge requestInit）；CCR 代理无 OAuth 仍用 ingress token |
| `sdk` | —— | 抛错“应在 print.ts 处理”（真正处理见 §11） |
| `claudeai-proxy` | `StreamableHTTPClientTransport` | 用 `createClaudeAiProxyFetch`；URL = `MCP_PROXY_URL + MCP_PROXY_PATH(server_id)`；带 `X-Mcp-Client-Session-Id` |
| stdio + Claude-in-Chrome | 进程内 transport | 特判：不 spawn ~325MB 子进程，改用 `createLinkedTransportPair` 进程内跑（`inProcessServer`）|
| stdio + Computer-Use（`CHICAGO_MCP`） | 进程内 transport | 同上进程内跑（本仓库 feature 恒 false，死代码）|
| `stdio`（默认，type 可省略） | `StdioClientTransport` | `CLAUDE_CODE_SHELL_PREFIX` 可包裹命令；env 合并 `subprocessEnv()` + 配置 env；`stderr: 'pipe'` 防错误输出污染 UI |
| 其它 | —— | 抛 `Unsupported server type` |

### 5.2 stderr 采集与 Client 构造（:967–1022）

- stdio transport 在连接前就挂 stderr handler（连接期的 stderr 对调试失败很有用），累计上限 64MB 防内存爆。
- 构造 `Client`，声明 capabilities：`roots: {}` 与 `elicitation: {}`。**特意用空对象**（:1000）——发 `{form:{},url:{}}` 会让 Java MCP SDK（Spring AI）的零字段 Elicitation 类因未知属性报错。
- 注册 `ListRootsRequestSchema` handler，返回 `file://${getOriginalCwd()}`。

### 5.3 带超时的连接（:1024–1163）

`Promise.race([connectPromise, timeoutPromise])`，超时 = `getConnectionTimeoutMs()`（`MCP_TIMEOUT` 或 30s）。超时或失败都会关掉 `inProcessServer` 与 `transport`。

失败分支按 transport 精细化处理：sse/http/claudeai-proxy 遇 `UnauthorizedError`/401 时走 `handleRemoteAuthFailure`（:342）——发 `tengu_mcp_server_needs_auth`、写 needs-auth 缓存、返回 `{type:'needs-auth'}`（**不抛异常**，降级）。IDE server 失败发专用遥测事件。

### 5.4 掉线检测与自动重连（:1224–1410）

连上后拿 capabilities、serverVersion、instructions（instructions 超 `MAX_MCP_DESCRIPTION_LENGTH`=2048 截断）。然后装两个增强 handler：

**`client.onerror`（:1274）** —— SDK 在连接失败时只调 onerror 不调 onclose，而 CC 靠 onclose 触发重连，这里桥接这个缺口：
- 按错误消息打细分日志（ECONNRESET/ETIMEDOUT/ECONNREFUSED/EPIPE/…）。
- **会话过期**（http/claudeai-proxy + `isMcpSessionExpiredError`）→ `closeTransportAndRejectPending('session expired')`。
- **`Maximum reconnection attempts`** → SDK 耗尽 SSE 重连的“确定性放弃”信号 → 关闭。
- **累计终端错误**（`isTerminalConnectionError`，:1257）→ 连续达 `MAX_ERRORS_BEFORE_RECONNECT`=3 次就关闭；非终端错误重置计数器。

`closeTransportAndRejectPending`（:1248）有 `hasTriggeredClose` 防重入，调 `client.close()` 而非只调 `onclose`——因为要经 SDK 的 `_onclose()` 拒绝所有挂起的请求（让卡住的 `callTool()` promise 以 `-32000 Connection closed` 失败），否则 pending 调用会永久挂起。

**`client.onclose`（:1382）** —— 清 memoize 缓存让下次操作重连：删 `connectToServer.cache` 该 key，并删 `fetchTools/Resources/Commands ForClient.cache`（按 server name），否则重连后会读到旧连接的过期 tools。

### 5.5 进程清理 `cleanup`（:1412–1578）

注册进 `registerCleanup`（:1582），进程退出时执行。分两类：

- **进程内 server**：直接 `inProcessServer.close()` + `client.close()`。
- **stdio server**：**信号升级链**——先 `SIGINT`（等 100ms）→ 不行 `SIGTERM`（等 400ms）→ 还不行 `SIGKILL`，总预算 500ms（含 600ms 绝对失败保护定时器），用 `process.kill(pid, 0)` 轮询进程是否已退出。为什么要这么复杂：`StdioClientTransport.close()` 只发 abort signal，很多 server（尤其 Docker 容器）需要显式 SIGINT/SIGTERM 才优雅退出。

成功路径最后发 `tengu_mcp_server_connection_succeeded`，返回 `{name, client, type:'connected', capabilities, serverInfo, instructions, config, cleanup}`。

---

## 6. 连接缓存与生命周期

| 函数 | 行 | 作用 |
|------|----|------|
| `ensureConnectedClient`（导出） | 1697 | 拿一个保证 connected 的 client。SDK server 直接返回；其余走 `connectToServer`（命中缓存或重连）。非 connected 抛错。**每次工具调用前都调它**（§9），是重连的关键入口。 |
| `clearServerCache`（导出） | 1657 | 先 cleanup 旧连接，再删 connectToServer + 三个 fetch 缓存。会话过期、重连时调。 |
| `reconnectMcpServerImpl`（导出） | 2146 | UI 层重连实现。先 `clearKeychainCache()`（另一进程如 VSCode 扩展改过 token 时读新的）→ clearServerCache → connectToServer → 并发拉 tools/commands/skills/resources → 若无人已加则附上 `ListMcpResourcesTool`/`ReadMcpResourceTool`。 |
| `areMcpConfigsEqual`（导出） | 1719 | 比对两份配置是否等价（排除 `scope` 元数据），靠 `JSON.stringify` 比。用于检测配置变更是否需重连。 |

---

## 7. 工具·资源·命令发现

三个 `memoizeWithLRU` 函数，缓存键都是 `client.name`（重连后稳定），容量 `MCP_FETCH_CACHE_SIZE`=20。

### 7.1 `fetchToolsForClient`（:1752）—— MCP tool → 内部 `Tool`

`tools/list` → `recursivelySanitizeUnicode` 净化 → 每个 tool `map` 成基于 `MCPTool` 模板的对象。要点：

- **命名**：`buildMcpToolName(serverName, toolName)` = `mcp__server__tool`。SDK server 且 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` 为真时**跳过前缀**（让 MCP tool 能按名覆盖内建工具）。
- **`_meta` 提示**：`anthropic/searchHint`（折叠空白，防注入 deferred-tool 列表）、`anthropic/alwaysLoad`。
- **能力标注**从 `tool.annotations` 派生：`isConcurrencySafe`/`isReadOnly`=`readOnlyHint`、`isDestructive`=`destructiveHint`、`isOpenWorld`=`openWorldHint`；`isSearchOrReadCommand` 走 `classifyMcpToolForCollapse`。
- **`checkPermissions`**：默认 `passthrough`，附一条 `addRules` 建议（允许该全限定名，写 localSettings）。
- **`call`**（:1842）：核心执行入口，见 §9。
- **末尾过滤 `isIncludedMcpTool`**（:572）——IDE server 只保留 `mcp__ide__executeCode`/`getDiagnostics` 两个白名单工具。
- Claude-in-Chrome / Computer-Use 的 stdio server 会 merge 各自的工具渲染覆盖。

### 7.2 `fetchResourcesForClient`（:2009）

`resources/list`，每个 resource 附上 `server` 名。无 capabilities.resources 直接返回 `[]`。

### 7.3 `fetchCommandsForClient`（:2042）—— MCP prompt → `Command`

`prompts/list` → 净化 → 每个 prompt 成 `type:'prompt'` 的 `Command`：
- 名字 `mcp__<normalized-server>__<prompt>`，`userFacingName` 用 `prompt.name`（非 title，避免空格破坏 slash 解析）。
- `getPromptForCommand(args)`：空格分割参数 → `zipObject(argNames, argsArray)` → `client.getPrompt()` → 每条 message content 走 `transformResultContent` → flat。

---

## 8. 批量连接编排（:2227–2482）

### 8.1 `getMcpToolsCommandsAndResources`（:2235，导出）

给所有 server 建连、发现原语，每完成一个就回调 `onConnectionAttempt`（流式，不等全部）。流程：

1. **分区**：disabled server 直接回 `{type:'disabled'}`，绝不发网络请求；其余进 `configEntries`。
2. 统计各 transport 数量（遥测用）。
3. **本地/远程分池**：`isLocalMcpServer`（stdio/sdk）用较低并发（`getMcpServerConnectionBatchSize`，默认 3，因要 spawn 进程），远程用较高并发（默认 20）。两池 `Promise.all` 同时跑。
4. `processServer`（:2291）单 server 流程：disabled 跳过 → **needs-auth 缓存/无 token 跳过**（§3，塞 `createMcpAuthTool`）→ `connectToServer` → 非 connected 回降级状态 → connected 则并发 `fetchTools/Commands/Skills/Resources` → 首个支持 resources 的 server 附资源工具（`resourceToolsAdded` 全局一次）→ 回调。

`processBatched`（:2227）现在只是 `pMap(items, processor, {concurrency})` 的薄封装。注释记录了 2026-03 的替换原因：旧的定长顺序批处理会让批 N 里一个慢 server 卡住整个批 N+1，即使其它 19 个槽空闲；`pMap` 一完成一个就释放槽，慢 server 只占一个槽。

### 8.2 `prefetchAllMcpResources`（:2417，导出）

包一层 Promise，用计数器（`pendingCount`/`completedCount`）等 `getMcpToolsCommandsAndResources` 的所有回调完成，聚合 clients/tools/commands，发 `tengu_mcp_tools_commands_loaded` 遥测。**不 memoize**——只在启动/reconfig 调 2-3 次，内层已缓存，且 main.tsx 每次传新配置对象会导致 memoize 泄漏。

---

## 9. 工具调用执行

模型调 `mcp__server__tool` 时进入 `fetchToolsForClient` 里定义的 `call`（:1842），它是三层重试的最外层：

### 9.1 `call` —— 会话过期重试层（:1842–1980）

- 提取 `toolUseId`（`extractToolUseId`），构造 `_meta: {'claudecode/toolUseId'}`，发 `started` progress。
- `for` 循环，`MAX_SESSION_RETRIES`=1：
  - `ensureConnectedClient(client)`（可能重连）→ `callMCPToolWithUrlElicitationRetry(...)`。
  - 成功发 `completed` progress，返回 `{data: content, mcpMeta?}`。
  - 捕获 `McpSessionExpiredError` 且 attempt < 1 → `continue`（拿新 client 重试）。
  - 其它错误发 `failed` progress，并**包装 MCP SDK 错误**给遥测：裸 `Error` → 用 message 前 200 字符；`McpError` → 用 `McpError ${code}`。

### 9.2 `callMCPToolWithUrlElicitationRetry` —— URL elicitation 重试层（:2822，导出/@internal 测试用）

`MAX_URL_ELICITATION_RETRIES`=3。调 `callToolFn`（默认 `callMCPTool`，可注入测试）；捕获 `McpError` 且 `code === UrlElicitationRequired`（-32042）时：
- 从 `error.data.elicitations` 校验出合法的 `ElicitRequestURLParams`（mode='url' + url/elicitationId/message 都是 string）。
- 逐个 elicitation：先跑 `runElicitationHooks`（可编程解决）→ 未解决则 `handleElicitation`（print/SDK 模式经 structuredIO 发控制请求）或 REPL 队列（`ElicitationDialog` 两阶段 consent/waiting 流）→ `runElicitationResultHooks` 后处理 → 非 accept 就返回“被拒”文本，accept 就 loop 回去重试工具调用。

### 9.3 `callMCPTool` —— 实际发起层（:3038，私有）

- **双重超时**：`Promise.race([client.callTool(...), timeoutPromise])`，超时 = `getMcpToolTimeoutMs()`（`MCP_TOOL_TIMEOUT` 或默认 ~27.8 小时 `DEFAULT_MCP_TOOL_TIMEOUT_MS`）。用自己的 timeout 兜底 SDK 内部超时失效（如 SSE 流中途断）的情况。
- 长任务每 30s 打一次 progress 日志（`progressInterval`）；SDK 的 `onprogress` 转成 `mcp_progress` 事件。
- **`isError: true`**（:3133）→ 抽 `content[0].text` 或 `error` → 抛 `McpToolCallError`。
- 成功 → 若命中 `detectCodeIndexingFromMcpServerName` 发代码索引遥测 → `processMCPResult`（§10）→ 返回 `{content, _meta, structuredContent}`。
- **catch 分支**是错误分类中枢（:3188）：
  - 401 / `UnauthorizedError` → `McpAuthError`（发 `tengu_mcp_tool_call_auth_error`）。
  - 会话过期（404+-32001，或 http/claudeai-proxy 上的 `-32000 Connection closed`）→ `clearServerCache` → 抛 `McpSessionExpiredError`（被 §9.1 捕获重试）。
  - `AbortError`（用户按 esc）→ 静默返回 `{content: undefined}`，不 logspew。

`callIdeRpc`（:2125，导出）是给 IDE 工具直接走 RPC 的薄封装，直接调 `callMCPTool`。

---

## 10. 结果转换与大输出处理

### 10.1 `transformResultContent`（:2487，导出）—— 多模态归一化

按 `content.type` 分派成 `ContentBlockParam[]`：
- `text` → 原样文本块。
- `audio` → base64 解码后 `persistBlobToTextBlock`（落盘 + 返回文件路径文本块）。
- `image` → `maybeResizeAndDownsampleImageBuffer`（压缩、限维）→ base64 image 块。
- `resource` → 内嵌 `text` 直接带前缀；`blob` 若是图片走图片压缩，否则 `persistBlobToTextBlock`。
- `resource_link` → 拼一行 `[Resource link: name] uri (description)` 文本。

`persistBlobToTextBlock`（:2607）把二进制落盘（`persistBinaryContent`），失败则返回一行“无法保存”说明，成功返回 `getBinaryBlobSavedMessage`（含文件路径），避免把裸 base64 灌进上下文。

### 10.2 `transformMCPResult`（:2671，导出）—— 三种结果形态

按优先级识别：`toolResult`（转 string）> `structuredContent`（`jsonStringify` + `inferCompactSchema` 推断 jq 友好签名）> `content` 数组（逐项 `transformResultContent`）。都不匹配抛“unexpected response format”。

`inferCompactSchema`（:2653，导出）递归生成紧凑类型签名如 `{title: string, items: [{id: number}]}`，深度默认 2，对象最多列 10 个键。

### 10.3 `processMCPResult`（:2729，导出）—— 大输出决策

拿到归一化 content 后：
- `ide` server 结果不发给模型，直接返回。
- `mcpContentNeedsTruncation` 判断是否过大；不大直接返回。
- 过大时：`ENABLE_MCP_LARGE_OUTPUT_FILES` 显式关 → 走旧的 `truncateMcpContentIfNeeded`；含图片 → 也截断（落盘 JSON 会破坏图片压缩与可视）；否则 → `persistToolResult` 落盘，返回 `getLargeOutputInstructions`（文件路径 + schema 描述，引导模型用 jq/分页读）。落盘失败 → 返回带字符数的错误提示。
- 每个分支都发 `tengu_mcp_large_result_handled` 遥测（outcome: truncated/persisted + reason）。

---

## 11. SDK 进程内 MCP（:3272，`setupSdkMcpClients` 导出）

SDK MCP server 跑在 SDK 同进程内，不经 `connectToServer`。为每个配置建 `SdkControlClientTransport(name, sendMcpMessage)`（经控制通道收发 `JSONRPCMessage`），`Promise.allSettled` 并行连接，成功则拉 tools（`fetchToolsForClient`），组装 `type:'connected'` 的 `MCPServerConnection`（scope 标 `dynamic`），失败标 `failed`。返回 `{clients, tools}`。

`ensureConnectedClient` 对 `config.type === 'sdk'` 直接返回、不重连（§6），正是因为这类连接的生命周期由控制通道而非 client.ts 管理。

---

## 12. 关键设计取舍与不变量

1. **memoize 是双刃剑**：`connectToServer` 用 lodash memoize 复用连接，但 onclose/clearServerCache 必须同步清 4 个缓存（connectToServer + 3 个 fetch），否则重连后拿到旧 tools。作者自己在 TODO 里质疑其复杂度收益。
2. **降级而非抛异常**：远程认证失败返回 `{type:'needs-auth'}` 而非抛错，让 UI 能展示“需认证”并给 `createMcpAuthTool`，用户 `/mcp` 即可恢复。
3. **needs-auth 磁盘缓存 + 读 memoize + 写串行**：三层配合，把“反复探测无 token server”的网络成本压到 15 分钟一次，且并发安全。
4. **超时无处不在但语义不同**：连接 30s、单请求 60s（GET 豁免）、工具调用默认 ~27.8 小时。每处都手动 clearTimeout 防 Bun 懒 GC 内存泄漏。
5. **SDK onerror/onclose 缺口靠 CC 桥接**：SDK 连接失败只调 onerror，CC 手动累计终端错误达阈值后 `client.close()` 以拒绝挂起请求 + 触发重连。
6. **进程清理的信号升级链**：stdio server 走 SIGINT→SIGTERM→SIGKILL，因 SDK 的 close 只发 abort，Docker 等需显式信号。
7. **大输出优先落盘而非截断**（含图片除外），返回文件路径 + schema，引导模型按需读取，省 token。
8. **遥测消息安全后缀**：`_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 是强制断言，保证上报文本不含用户代码/路径。

---

## 关联模块速查

| 模块 | 与 client.ts 的关系 |
|------|--------------------|
| `config.ts` | 提供 `getAllMcpConfigs`/`isMcpServerDisabled`，决定连哪些 server |
| `useManageMCPConnections.ts` | 上层 React 编排，调 `getMcpToolsCommandsAndResources`/`reconnectMcpServerImpl` |
| `auth.ts` | `ClaudeAuthProvider`、`wrapFetchWithStepUpDetection`、`hasMcpDiscoveryButNoToken` |
| `elicitationHandler.ts` | `runElicitationHooks`/`runElicitationResultHooks`，被 §9.2 调用 |
| `mcpStringUtils.ts` / `normalization.ts` | `buildMcpToolName`、`normalizeNameForMCP` 命名 |
| `MCPTool/MCPTool.ts` | `fetchToolsForClient` 的基础模板 |
| `utils/mcpValidation.ts` / `utils/mcpOutputStorage.ts` | 截断判定与二进制落盘 |
| `SdkControlTransport.ts` / `InProcessTransport.ts` | SDK 与进程内 transport |
