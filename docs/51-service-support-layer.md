# 服务支撑层：会话持久化、Checkpoint、链路观测

> 本文对应 Claude Code 中"服务支撑层"三大子系统的实现原理与运行时数据流。
> 与业务逻辑（REPL/QueryEngine/Tool 系统）不同，这一层的职责是让业务流程
> 具有**可持久化、可恢复、可观测**的能力——即"每一句对话被写下去、每一次
> 编辑可以回滚、每一次调用被度量并送到监控后端"。
>
> 涉及源码：
>
> **会话持久化**
> - `src/utils/sessionStorage.ts` — `Project` 类、JSONL 事务追加、materialize、re-append metadata
> - `src/utils/sessionStoragePortable.ts` — 尾部读取、字段抽取（供 lite/resume 使用）
> - `src/types/logs.ts` — `Entry` 联合类型、`LogOption`、`TranscriptMessage`
> - `src/services/api/sessionIngress.ts` — CCR v1 远端会话追加
> - `src/utils/conversationRecovery.ts` — `loadConversationForResume`
> - `src/utils/sessionRestore.ts` — `restoreSessionStateFromLog`
>
> **Checkpoint（文件历史 / 快照）**
> - `src/utils/fileHistory.ts` — `fileHistoryTrackEdit` / `fileHistoryMakeSnapshot` /
>   `fileHistoryRewind` / `applySnapshot` 等
> - `src/hooks/useFileHistorySnapshotInit.ts` — 恢复钩子
> - `src/utils/sessionStorage.ts::insertFileHistorySnapshot` — 快照落盘
>
> **链路观测**
> - `src/utils/telemetry/instrumentation.ts` — OpenTelemetry Provider 引导
> - `src/utils/telemetry/sessionTracing.ts` — Interaction / LLM / Tool / Hook 四层 Span
> - `src/utils/telemetry/perfettoTracing.ts` — Chrome/Perfetto Trace Event JSON
> - `src/utils/telemetry/betaSessionTracing.ts` — Beta 属性、内容截断
> - `src/services/api/logging.ts` — `logAPIQuery` / `logAPISuccessAndDuration` / `logAPIError`
> - `src/services/api/errors.ts::classifyAPIError` — 错误分类
> - `src/utils/telemetry/events.ts::logOTelEvent` — OTLP 事件写入

---

## 0. 三个子系统的关系与总览

```
                            ┌──────────────────────┐
      用户/模型对话           │  REPL / QueryEngine  │
      ─────────────►         │      / Tool exec     │
                            └────┬───────────┬─────┘
                                 │           │
                       ┌─────────▼───┐   ┌───▼──────────┐
                       │ 会话持久化   │   │ 链路观测      │
                       │ (JSONL)     │   │ (OTel/Perf.) │
                       │             │   │              │
                       │  Project    │   │  Interaction │
                       │  队列/批量   │   │  → LLMReq    │
                       │  materialize│   │  → Tool      │
                       │             │   │  → Hook      │
                       └────┬────────┘   └──────────────┘
                            │
                       ┌────▼───────────────┐
                       │  Checkpoint         │
                       │  (fileHistory)      │
                       │  FileHistoryState   │
                       │  快照序列 + 备份池   │
                       └────────────────────┘
```

- **会话持久化**关注"当前会话的完整事件流"—— 每一条 user/assistant/attachment/
  system/tool_use/tool_result 都以 JSONL 追加到 `~/.claude/projects/<sanitized-cwd>/
  <sessionId>.jsonl`；伴随的还有 tag/custom-title/last-prompt 等元数据 entry。
- **Checkpoint** 关注"某一条 message 触发编辑时磁盘的状态"—— 每次工具改文件之前
  copy 一份内容到 `~/.claude/file-history/<sessionId>/<hash>@v<n>`，并把这次快照
  作为一个 `file-history-snapshot` entry 追加进会话 JSONL；`/rewind` 或 SDK 调
  `fileHistoryRewind` 时按 `messageId` 找到目标快照，逐文件恢复。
- **链路观测**关注"这个会话/请求/工具调用花了多长时间、走了什么路径"——
  同一个业务事件同时被送到：**analytics 埋点 (`tengu_*`)**、**OpenTelemetry
  Trace/Logs/Metrics**、**Perfetto Chrome Trace JSON**。三者互不干扰，任意子集
  可关。

三者共享一个约束：**它们必须是"旁路"、不能阻塞主业务链路。** 因此本文重点
会围绕"如何异步、如何 backpressure、如何在异常/退出时不丢数据"来解释。

---

## 1. 会话持久化

### 1.1 存储介质与目录结构

一条会话对应磁盘上的一个 JSONL 文件（每行一个 JSON entry）：

```
~/.claude/projects/
  ├── -Users-alice-code-my-app/                    ← sanitize(cwd)
  │    ├── 019428af-....-....-................jsonl        ← 主会话
  │    ├── 019428af-.../subagents/agent-XX....jsonl        ← 子 Agent 独立文件
  │    ├── 019428af-.../subagents/workflows/<runId>/agent-YY.jsonl
  │    ├── 019428af-.../subagents/agent-XX...meta.json     ← Agent 元数据 sidecar
  │    └── 019428af-.../remote-agents/remote-agent-<id>.meta.json
  └── -Users-alice-code-other-repo/
```

路径由几个 helper 计算（`sessionStorage.ts:199-263`）：

| 函数 | 返回 |
| --- | --- |
| `getProjectsDir()` | `${claudeConfigHome}/projects` |
| `getProjectDir(cwd)` | `${projectsDir}/${sanitizePath(cwd)}`（memoized） |
| `getTranscriptPath()` | `${projectDir}/${sessionId}.jsonl`（当前会话） |
| `getTranscriptPathForSession(id)` | 其他会话——若 `id === current` 使用同一逻辑（含 `sessionProjectDir` fallback），否则以 `originalCwd` 为准 |
| `getAgentTranscriptPath(agentId)` | 子 Agent 走 `subagents/[<subdir>/]agent-<id>.jsonl` |

`sanitizePath` 把 `/Users/alice/foo` 变成 `-Users-alice-foo`，使 cwd 可以直接
作为目录名；这就是为什么每个项目的会话都物理上分组在一起。

### 1.2 Entry 的类型体系

JSONL 里每一行是一个 `Entry`（`src/types/logs.ts:297-317`）：

```ts
export type Entry =
  | TranscriptMessage            // user / assistant / attachment / system
  | SummaryMessage               // AI 生成的会话摘要
  | CustomTitleMessage
  | AiTitleMessage
  | LastPromptMessage            // 供 --resume 列表显示"上次在做什么"
  | TaskSummaryMessage
  | TagMessage
  | AgentNameMessage / AgentColorMessage / AgentSettingMessage
  | PRLinkMessage
  | FileHistorySnapshotMessage   // ⭐ Checkpoint 落盘就是这个 entry
  | AttributionSnapshotMessage
  | QueueOperationMessage
  | SpeculationAcceptMessage
  | ModeEntry / WorktreeStateEntry
  | ContentReplacementEntry
  | ContextCollapseCommitEntry / ContextCollapseSnapshotEntry
```

`TranscriptMessage` 才是"真正参与父子链的对话消息"，`isTranscriptMessage` 与
`isChainParticipant` 是这一层的两条断言（`sessionStorage.ts:139-156`）：

- **`isTranscriptMessage`**：判断是否属于会话主链（user/assistant/attachment/system）。
  Progress、tag、metadata 等 entry 一律 false。
- **`isChainParticipant`**：写入 `parentUuid` 时是否把当前 message 作为下一条的
  parent；`progress` 不参与，是防止父子链在流式过程中被 fork。

### 1.3 `Project` 单例：写入路径的心脏

模块级单例 `project: Project | null`（`sessionStorage.ts:441-468`）
在第一次 `getProject()` 时懒创建，并同时注册 cleanup handler：进程结束前会先
`flush()` 所有排队的写入，再调用 `reAppendSessionMetadata()` 把 custom-title / tag /
last-prompt 等元数据"补写到文件末尾"，保证 lite 读取（只读 64 KiB 尾部）总能拿到
最新元数据。

`Project` 内部有几组关键字段（`sessionStorage.ts:533-570`）：

| 字段 | 作用 |
| --- | --- |
| `sessionFile: string \| null` | 当前会话的目标路径；null 表示"还没落盘" |
| `pendingEntries: Entry[]` | 在 `sessionFile===null` 期间缓冲的 entry |
| `writeQueues: Map<filePath, {entry,resolve}[]>` | 按目标文件分片的写队列 |
| `flushTimer / activeDrain` | 定时批量落盘（100 ms 一次，远端场景 10 ms） |
| `pendingWriteCount / flushResolvers` | 追踪没落盘的写数量，`flush()` 拿来 await |
| `remoteIngressUrl / internalEventWriter` | CCR v1 / v2 远端持久化钩子 |
| `currentSession*` | Tag/Title/AgentName 等元数据的进程内 cache |

#### 1.3.1 事务级别的追加：`appendEntry`

`appendEntry(entry, sessionId)`（`sessionStorage.ts:1131-1268`）是所有写入的
入口。核心逻辑分四段：

1. **持久化拦截**：`shouldSkipPersistence()`（962-972 行）在测试环境、
   `cleanupPeriodDays===0`、`CLAUDE_CODE_SKIP_PROMPT_HISTORY=1`、`--no-session-
   persistence` 任一命中时静默丢弃；这是 Tungsten/E2E 测试不污染用户 --resume
   列表的关键。
2. **目标文件解析**：本会话消息写 `this.sessionFile`；其他会话（比如 SDK 的
   `renameSession(id)`）写 `getExistingSessionFile(id)`（缓存 stat 结果，1287 行）。
3. **首次消息 materialize**：当 `sessionFile===null` 且这批消息里包含
   user/assistant 时，`materializeSessionFile()`（978-993 行）会：
   - 建目录、创建文件、把 mode/agentSetting 等 cache 通过
     `reAppendSessionMetadata()` 写到文件头，
   - 再把此前 buffer 在 `pendingEntries` 里的 entry 逐条重放。
   这一步保证："纯 metadata 的空会话"不会造出 JSONL——只有真正对话开始才建文件。
4. **入队 + 特殊类型处理**：整个 `appendEntry` 是一棵 `if/else` 树：
   - `summary` / `custom-title` / `ai-title` / `last-prompt` / `tag` / `agent-*` /
     `pr-link` / `mode` / `worktree-state` / `file-history-snapshot` /
     `attribution-snapshot` / `speculation-accept` / `marble-origami-*` /
     `queue-operation` —— 无脑入队（`enqueueWrite`），因为它们不参与 UUID 去重。
   - `content-replacement` —— 若带 `agentId` 则写子 Agent 文件，否则写主会话文件。
   - **TranscriptMessage**（user/assistant/attachment/system）——先查
     `getSessionMessages(sessionId)`（返回该 sessionId 已存在的 UUID Set）
     去重；主 chain 上新 UUID 加进 Set；主 chain 上的 transcript 消息还要通过
     `persistToRemote()` 走 CCR 通道（下节）。sidechain 消息**跳过去重**、
     **不走远端**，因为 sidechain 会 fork 继承主 UUID，两边都可能有同一 UUID
     的消息副本。

#### 1.3.2 队列与批量落盘：`enqueueWrite → scheduleDrain → drainWriteQueue`

每次 `enqueueWrite(filePath, entry)`（608-618 行）：

```ts
new Promise(resolve => {
  writeQueues.get(filePath).push({entry, resolve})
  scheduleDrain()
})
```

调用方通过返回的 Promise 拿"我这条已经落盘"的语义，但**默认全都 `void` 处理**——
即 fire-and-forget，主流程不阻塞。

`scheduleDrain()` 用一个 `flushTimer` 保证 100 ms（或 CCR 场景 10 ms）内所有
`enqueueWrite` 合并成一次 IO：

```ts
scheduleDrain() {
  if (this.flushTimer) return
  this.flushTimer = setTimeout(async () => {
    this.flushTimer = null
    this.activeDrain = this.drainWriteQueue()
    await this.activeDrain
    this.activeDrain = null
    if (this.writeQueues.size > 0) this.scheduleDrain()
  }, this.FLUSH_INTERVAL_MS)
}
```

`drainWriteQueue()`（647-688 行）按 filePath 分组，把该文件的所有 entry
`JSON.stringify + \n` 拼成一段 `content`，一旦超过 `MAX_CHUNK_BYTES = 100MB`
就先冲一次；每次 `appendToFile()` 尝试 append，如果 `ENOENT` 就 lazy mkdir 后
重试。这样：

- 高频写只会 100 ms 阻塞一次，且 fsAppendFile 是 O(1) tail write。
- 大量 entry 也不会一次性把 buffer 撑到 GB 级。
- Ctrl-C/进程退出时 `flush()`（843-863 行）会等 `activeDrain` 结束、再 drain
  剩余队列、再等 `pendingWriteCount==0`，然后 cleanup handler 再写一次尾部
  metadata（`reAppendSessionMetadata`）。

#### 1.3.3 首条 user 消息前的 buffer：`pendingEntries`

启动时的 mode/agent-name/last-prompt 都可能先到，但那时候 sessionFile 还没
materialize。`appendEntry` 在 `sessionFile===null && isCurrentSession` 时
把 entry 塞进 `pendingEntries`（1142-1145 行）；等 `insertMessageChain` 首次
遇到 user/assistant 消息（1007-1012 行）时才创建文件、重放缓冲。

这是为了：**不给"用户开了会话但没说话就退出"的场景留空文件**。—— 一次纯 CTRL-C
不会污染 --resume 列表。

#### 1.3.4 消息链的写入：`insertMessageChain` / `recordTranscript`

`insertMessageChain`（995-1086 行）是 QueryEngine 每回合调用的入口：

- 传入的 `messages` 是"这轮新产生的所有消息"，
- 每条消息被包装成 `TranscriptMessage`（41-67 行的字段：`parentUuid` /
  `logicalParentUuid` / `isSidechain` / `teamName` / `agentName` / `promptId` /
  `agentId` / `sessionId` / `timestamp` / `version` / `gitBranch` / `slug` / `cwd`
  / `entrypoint` / `userType`），
- 逐条 `appendEntry`，成功后如果 `isChainParticipant(message)` 就把 `parentUuid` 前
  移到 `message.uuid`。
- 首条 user 消息还会 cache 到 `currentSessionLastPrompt`——供 lite 读取显示。

上层的 `recordTranscript`（1412-1453 行）负责另一个关键：**去重**。因为同一批
`messages` 可能在压缩、resume、fork 里被反复传入，`recordTranscript` 先与
`getSessionMessages(sessionId)` 求差集，只把"新的一段"喂给 `insertMessageChain`，
并把父链起点（`startingParentUuid`）设为"跳过部分的最后一条"，保证不会重复写
入或断链。

#### 1.3.5 移除消息：`removeMessageByUuid`

流式失败或 tombstone 收到时（`sessionStorage.ts:873-953`）：

- 先 `open('r+')`，读尾部 64 KiB (`LITE_READ_BUF_SIZE`)；
- 用 `"\"uuid\":\"<uuid>\""` 逐字节 lastIndexOf 定位（不匹配 `parentUuid`），
- 找到就 `truncate` + rewrite 尾部（快路径，绝大多数就是最后一行）；
- 找不到就走"读全文过滤重写"慢路径，但先卡一层 `MAX_TOMBSTONE_REWRITE_BYTES = 50MB`
  防止对 GB 级会话 OOM。

### 1.4 远端持久化：Session Ingress (v1) 与 Internal Events (v2)

`persistToRemote()`（1306-1347 行）在写完本地 JSONL 后决定是否再送一份到远端：

- 若 `internalEventWriter`（CCR v2）已注入 → 用它写事件；
- 否则若 `ENABLE_SESSION_PERSISTENCE=1` 且 `remoteIngressUrl` 存在 → 调
  `sessionIngress.appendSessionLog(sessionId, entry, url)`，
- 上面两条一旦返回 failure 就 `gracefulShutdownSync(1)`——远端写入失败被视作
  "不可继续的严重错误"，避免本地/远端 divergence。

`FLUSH_INTERVAL_MS` 在 setRemoteIngressUrl / setInternalEventWriter 时会
从 100ms 调到 10ms（`REMOTE_FLUSH_INTERVAL_MS`），因为 CCR 场景对延迟更敏感。

Resume 时 v2 的读侧对应 `getInternalEventReader() / getInternalSubagentEventReader()`。

### 1.5 恢复流程：从 JSONL 到内存

`--resume` 的整体路径：

```
loadConversationForResume(sessionId, entrypoint)
  │
  ├── loadFullLog(fullPath)                 ← 解析 JSONL，构建 conversation chain
  │     └── loadTranscriptFile              ← 逐行 JSON.parse、按 parentUuid 链接
  │
  ├── copyFileHistoryForResume(log)         ← 把旧 session 的 backup 硬链到新 session
  │
  ├── restoreSkillStateFromMessages(msgs)
  │
  └── returns ResumeResult {
        messages,
        fileHistorySnapshots,
        attributionSnapshots,
        contextCollapseCommits,
        contextCollapseSnapshot,
      }
```

`restoreSessionStateFromLog(result, setAppState)`（`sessionRestore.ts:99-150`）
消费该结果，往 AppState 灌回：

- `fileHistory` state（下一章重点）
- `attribution` state（Claude 字符贡献计数，Ant-only）
- Context Collapse 的 commit log + staged snapshot
- SDK/非交互模式下的 TodoWrite 列表（从最后一条 assistant.tool_use 抽出）

对交互 REPL，`useFileHistorySnapshotInit` 钩子（`src/hooks/useFileHistorySnapshotInit.ts`）
在 mount 时用 `fileHistoryRestoreStateFromLog` 把 `initialFileHistorySnapshots`
灌到 `fileHistoryState`，并把绝对路径迁移成相对路径。

---

## 2. Checkpoint 系统（fileHistory）

Checkpoint 系统解决的问题是：**"回到上一条消息之前，文件系统应该是什么样"**。
它不是 git diff、也不是编辑器 undo，而是一种"会话级、按 message 索引"的
snapshot/restore 机制。

### 2.1 数据模型

三个核心结构（`fileHistory.ts:31-52`）：

```ts
type BackupFileName = string | null    // null 表示"该文件在这个版本不存在"

type FileHistoryBackup = {
  backupFileName: BackupFileName       // 磁盘上备份文件的名字（去掉目录部分）
  version: number                      // 单调递增的版本号
  backupTime: Date
}

type FileHistorySnapshot = {
  messageId: UUID                                          // 触发此快照的 message
  trackedFileBackups: Record<string, FileHistoryBackup>    // 相对路径 → 备份
  timestamp: Date
}

type FileHistoryState = {
  snapshots: FileHistorySnapshot[]      // 有序，最多 MAX_SNAPSHOTS = 100
  trackedFiles: Set<string>             // 会话里所有被追踪过的文件（相对路径）
  snapshotSequence: number              // 单调计数，即使 snapshots 被截断也不减
}
```

物理上备份文件写到：

```
~/.claude/file-history/<sessionId>/<sha256(path)前16位>@v<版本号>
```

见 `getBackupFileName` (725-731) 和 `resolveBackupPath` (733-741)。

### 2.2 关键状态转移

Checkpoint 的生命周期基本围绕三个入口：

| 触发点 | 函数 | 时机 |
| --- | --- | --- |
| **改文件前** | `fileHistoryTrackEdit` | 每次 Write/Edit/NotebookEdit 工具执行"之前"，先把当前内容备份为 v1 |
| **每回合结尾** | `fileHistoryMakeSnapshot` | REPL 一轮 message 结束时，把当前所有 tracked file 的最新版本组成一个 snapshot |
| **/rewind** | `fileHistoryRewind` | 用户/SDK 请求回滚到 message X 时，把 tracked file 恢复到 X 时刻的版本 |

#### 2.2.1 `fileHistoryTrackEdit`（86-193 行）

第一次编辑某个文件时被调用，三阶段：

1. **Phase 1：读状态判断是否需要 backup。** 用一个不改状态的 updater
   `state => state` 只是把当前 state 抓出来。若该文件已经出现在
   `mostRecent.trackedFileBackups[path]` 里就直接 return——`makeSnapshot` 会
   在回合结束时统一处理增量。这一条防止对 v1 的重复写（"投机执行"的重放会
   多次触发 trackEdit，如果无脑覆盖会毁掉 v1 快照）。
2. **Phase 2：Async IO——`createBackup(filePath, 1)`**。若源文件不存在，
   `backupFileName=null` 记为"文件当时不存在"；否则 `copyFile` 到
   `<hash>@v1`，`chmod` 保留权限。整个过程不持锁。
3. **Phase 3：Commit——第二次调 updater。** 再次做 raced check（另一个
   trackEdit 可能已插过），如果没被抢先就 shallow-spread：更新最后一个 snapshot
   的 `trackedFileBackups`、`trackedFiles` 加入新路径，并把这个"更新过的最后一
   个 snapshot"通过 `recordFileHistorySnapshot(id, snapshot, isSnapshotUpdate=true)`
   写进 JSONL。**`isSnapshotUpdate=true` 是与 `false` 的核心区别——
   `buildFileHistorySnapshotChain`（`sessionStorage.ts:2252-2276`）依赖这个 flag
   决定"追加新快照"还是"覆盖同 messageId 的既有快照"。**

#### 2.2.2 `fileHistoryMakeSnapshot`（198-342 行）

每回合结束由 QueryEngine/REPL 调用，同样三阶段但语义不同：

1. **Phase 1：捕获状态。**
2. **Phase 2：对每个 tracked file 做一次判断（并行 `Promise.all`）：**
   - 文件不存在了 → 记 `backupFileName=null`；
   - 文件存在但 mtime/size 没变（`checkOriginFileChanged`, 600-634 行）→
     沿用上一版 backup，`version` 不递增；
   - 内容变了 → `createBackup(file, latestVersion+1)`。
3. **Phase 3：写入新的 `FileHistorySnapshot`。** 若某文件在 phase 2 期间被
   `trackEdit` 抢先写了，则从 `state.snapshots[-1].trackedFileBackups` 继承。
   新 snapshot 追加到 `state.snapshots`，若超过 `MAX_SNAPSHOTS=100` 就 slice
   保留最后 100（历史越久越丢）。然后：
   - `recordFileHistorySnapshot(messageId, newSnapshot, false)` → 写 JSONL；
   - `notifyVscodeSnapshotFilesUpdated()` → VSCode 扩展 hook；
   - `snapshotSequence` 单调 +1（哪怕 snapshots 被截断也不减，用于外部
     "活性检测"）。

`createBackup` 的关键细节（748-798 行）：

- **先 `stat` 再 `copyFile`。** 分开处理"源文件缺失"和"backup 目录缺失"，
  避免"文件在 copyFile 成功后被删掉"这种极端 race 留下孤儿 backup。
- **`copyFile` 而不是 `readFile+writeFile`。** 保护大文件不进 V8 heap。
- **Lazy mkdir**：只在 `ENOENT` 时重试一次 `mkdir(dirname, recursive)` + `copyFile`。

#### 2.2.3 `fileHistoryRewind`（347-397 行）与 `applySnapshot`（537-591 行）

`fileHistoryRewind(setState, messageId)`：

- 用 no-op updater 抓当前 state；
- `snapshots.findLast(s => s.messageId === messageId)` 找到目标 snapshot；
- 调用 `applySnapshot(state, targetSnapshot)`：
  - 对 `state.trackedFiles` 里的每一个路径查 `targetSnapshot.trackedFileBackups[p]`；
  - 若目标 backup 缺失，则 `getBackupFileNameFirstVersion` 找该文件的第一版
    (`version===1`)——这是"你追踪它之前"的状态；
  - `backupFileName===null` → 磁盘上删除（unlink，忽略 ENOENT）；
  - 有 backupFileName → `checkOriginFileChanged` 判断和现有文件是否已一致，
    不一致就 `restoreBackup(filePath, backupFileName)`（copyFile+chmod）。

**关键点：Rewind 不改 FileHistoryState 本身。** 只操作磁盘。回滚后你依然可以
基于同一份历史继续往前走；下一次 makeSnapshot 会因为磁盘和上一版 backup
一致而不新增版本。

`fileHistoryGetDiffStats` / `fileHistoryHasAnyChanges`（414-531 行）是 dry-run
版本，只算/判断哪些文件会变——分别配合"显示回滚 diff"和"决定是否显示 rewind
按钮"。

### 2.3 Checkpoint 如何跨会话延续：`copyFileHistoryForResume`

`copyFileHistoryForResume(log)`（922-1046 行）在 `/resume` 时被调：

- 从旧 session 的 `~/.claude/file-history/<prevSessionId>/` 硬链（fallback 拷贝）
  所有 backup 到 `~/.claude/file-history/<newSessionId>/`；
- 硬链失败（EEXIST 跳过，ENOENT 报错，其他错走 copy fallback）；
- 每完成一个 snapshot 的迁移就 `recordFileHistorySnapshot(id, snapshot, false)`
  把它重写到新会话的 JSONL；这样新会话下次自己 `--resume` 时能读回。

硬链的意义：**几乎不多占磁盘**，同一份内容多次 resume 分支只有一份 inode。

### 2.4 什么时候被禁用

`fileHistoryEnabled()`（63-78 行）：

- 交互模式：`config.fileCheckpointingEnabled !== false` 且
  `!CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING`；
- 非交互（SDK）：默认关，除非 `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=1`
  且没被 disable。

任何一个入口——`trackEdit / makeSnapshot / rewind / hasAnyChanges / getDiffStats /
canRestore`——都是**开关未开就 no-op**，工具依然照常读写文件，只是没有"事后回滚
能力"。

---

## 3. 链路观测（Telemetry / Tracing）

Claude Code 的观测栈同时跑三条链路：

1. **Analytics 埋点 (`tengu_*`)** — 只出简单事件到 BigQuery/第一方后端，是历史最老的一路。
2. **OpenTelemetry (OTel)** — Provider/Exporter 完全遵循 OTel 官方 API，兼容
   customer 的可观测平台。
3. **Perfetto Trace** — Chrome/Perfetto 格式 JSON，方便离线时序可视化（Ant-only）。

它们共用同一批业务事件源，但输出通道独立、可分别开关。

### 3.1 Provider 初始化：`initializeTelemetry`

`src/utils/telemetry/instrumentation.ts::initializeTelemetry`（421 行起）在
`entrypoints/init.ts` 早期被调用，做四件事：

1. **`bootstrapTelemetry()`（87-117 行）** —— 把 `ANT_OTEL_*` 变量拷成
   `OTEL_*`；设定默认 `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta`。
2. **`initializePerfettoTracing()`（`perfettoTracing.ts:253-335`）** ——
   由 `CLAUDE_CODE_PERFETTO_TRACE=1|<path>` 触发；开启后会：
   - 记录 `startTimeMs`，`tracePath` 设为
     `~/.claude/traces/trace-<sessionId>.json`；
   - 启动 stale-span 清理定时器（每 60 s 清 > 30 min 的 pending span）；
   - `registerCleanup` 让退出时写 trace；`beforeExit` 兜底；`exit` 时同步写；
   - 若 `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=N` 设定，就 N 秒周期性 `writeFile`。
3. **OTLP Readers / Exporters（130-322 行）** —— 读 `OTEL_METRICS_EXPORTER` /
   `OTEL_LOGS_EXPORTER` / `OTEL_TRACES_EXPORTER`（逗号分隔），每一项按
   `console` / `otlp` / `prometheus` 分支：
   - `otlp` 再看 `OTEL_EXPORTER_OTLP_PROTOCOL`：`grpc` / `http/json` /
     `http/protobuf`——**动态 import** 对应 exporter 包（`@opentelemetry/exporter-
     trace-otlp-{grpc,http,proto}` 等），未使用的 protocol 完全不进 bundle。
4. **Provider 装配** —— 用 `resourceFromAttributes` 合并 `service.name`、`service.version`、
   `os.*`、`host.arch`、env 侦测的 resource，然后：
   - `BasicTracerProvider(spanProcessors=[BatchSpanProcessor(exporter)])`
   - `LoggerProvider(processors=[BatchLogRecordProcessor(exporter)])`
   - `MeterProvider(readers=[PeriodicExportingMetricReader])`
   分别 setGlobal 并存到 bootstrap state。
5. **Beta Tracing（353-419 行）** —— 若 `ENABLE_BETA_TRACING_DETAILED=1` 且
   `BETA_TRACING_ENDPOINT` 设置，则**单独**跑一路 OTLP HTTP trace/logs 到该
   endpoint（用于内部详细调试），不复用 3P 的 exporter 列表。
6. **shutdown 竞速** —— `endInteractionSpan()`，然后把 loggerProvider /
   tracerProvider / meterProvider 的 `forceFlush→shutdown` 拴在 timeout
   （默认 2000 ms）里，防止慢的 OTLP endpoint 挂住退出。

### 3.2 四层 Span：Interaction → LLM Request → Tool → Hook

`sessionTracing.ts` 定义了业务侧的 span 抽象。它做了三件重要的事：

1. **把 OTel `Span` 用 `AsyncLocalStorage` 挂在异步上下文里**——`interactionContext` /
   `toolContext`（65-77 行），保证同一异步链路上 `getCurrentSpan()` 拿得到父。
2. **同时开 Perfetto 平行 span**——每个 startXxxSpan 都会先看
   `isPerfettoTracingEnabled()` 调对应 `perfettoTracing` 里的
   `startXxxPerfettoSpan(...)`，把 `perfettoSpanId` 挂在 `SpanContext.perfettoSpanId`。
3. **在 OTel 关闭时依然给 Perfetto 起 dummy span**——`if (!isAnyTracingEnabled()) { … }`
   分支里用 `getTracer().startSpan('dummy')` 只作为 anchor，避免破坏
   `getCurrentSpan()` 的语义。

四层 span 的层级、字段与调用点：

| Span | 起点 | 终点 | 关键属性 |
| --- | --- | --- | --- |
| `claude_code.interaction` | 每次用户提交 prompt（REPL 提交前调 `startInteractionSpan`） | REPL 结束一轮 / `endInteractionSpan()` 显式关 | `user_prompt`（默认 `<REDACTED>`；`OTEL_LOG_USER_PROMPTS=1` 才明文）、`user_prompt_length`、`interaction.sequence` |
| `claude_code.llm_request` | `services/api/claude.ts` 打 API 前 `startLLMRequestSpan(model, newContext, msgs, fastMode)` | `logAPISuccessAndDuration` / `logAPIError` 里 `endLLMRequestSpan(span, {…})` | `model`、`llm_request.context=interaction\|standalone`、`speed=fast\|normal`、`query_source`，end 时补：`input_tokens/output_tokens/cache_read/cache_creation/attempt/ttft_ms/response.has_tool_call/model_output/thinking_output` |
| `claude_code.tool` | 工具执行前 `startToolSpan(name, attrs, input)` | 工具执行后 `endToolSpan(result, tokens)` | `tool_name`、input via `addBetaToolInputAttributes`，end 补 `duration_ms`、`result_tokens` |
| `claude_code.tool.blocked_on_user` / `.execution` / `claude_code.hook` | `checkPermissionsAndCallTool` 的权限阻塞、真正执行、Hook 触发 | 对应 end | 权限拒绝原因、hook_event、hook_name |

内部的 `activeSpans` 是一个 `Map<spanId, WeakRef<SpanContext>>`；`strongSpans`
是同 key 的强引用副本，只在 span 生命周期内持有，`end` 后从两者删除。这个双
Map + `SPAN_TTL_MS=30min` 的 stale cleanup（86 行前后）是保护
"忘记 endSpan"造成的内存泄漏。

`executeInSpan(name, fn, attrs)`（788-833 行）是给同步/异步业务方便包裹用的
高阶 helper——自动 start/end、异常时 `recordException`。

### 3.3 Perfetto：Chrome Trace Event

Perfetto 用的数据结构是 Chrome Trace Event 格式（`perfettoTracing.ts:47-69`）：

```ts
type TraceEvent = {
  name: string; cat: string
  ph: 'B' | 'E' | 'X' | 'i' | 'C' | 'b' | 'n' | 'e' | 'M'   // 阶段类型
  ts: number                                                  // 微秒
  pid: number; tid: number
  dur?: number
  args?: Record<string, unknown>
  id?: string; scope?: string
}
```

事件流由两个数组承载：`metadataEvents`（process/thread name、parent link，`ph='M'`）
和 `events`（业务 B/E/X/i 事件）。合并输出结构：

```ts
buildTraceDocument() = jsonStringify({
  traceEvents: [...metadataEvents, ...events],
  metadata: {
    session_id, trace_start_time, agent_count, total_event_count
  }
})
```

Agent 层级建模是关键（`getCurrentAgentInfo`, 147-167）：主 agent 的
`processId=1`，子 Agent 用 `processIdCounter++`，`threadId=djb2Hash(agentName)`。
子 Agent spawn 时调 `registerAgent(id, name, parentId)`，`emitProcessMetadata`
写入 process/thread 名字和 parent link，Perfetto UI 可以据此画出树状 track。

事件容量控制：

- `MAX_EVENTS=100_000`——每 60 s 检查一次，超过就 `splice(0, MAX/2)` 掉最旧一半，
  并插入一个 `trace_truncated` instant marker，让 UI 上看得见"这里断层了"。
- `STALE_SPAN_TTL_MS=30min` + `STALE_SPAN_CLEANUP_INTERVAL_MS=60s`：
  `pendingSpans` 里超时的 span 会自动 emit "end + evicted:true"，防止悬空 span
  永远等不到 close。

写盘策略（`initializePerfettoTracing` 的三重保底）：
1. `registerCleanup(async () => writePerfettoTrace())` —— 正常退出（async, 全量）；
2. `process.on('beforeExit', () => void writePerfettoTrace())` —— 兜底 async；
3. `process.on('exit', () => writePerfettoTraceSync())` —— **同步写**，最后一道防线；
4. 若 `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=N`，则每 N 秒周期性写全量 snapshot
   （不 truncate 事件，只是 dump 一份）。

`traceWritten` 布尔量防止同一次退出重复写。

### 3.4 API 层的埋点：`logAPIQuery` / `logAPISuccessAndDuration` / `logAPIError`

`services/api/logging.ts`：

- **`logAPIQuery`（171-233 行）** —— 请求前埋点，`tengu_api_query` 事件带
  model/messagesLength/temperature/betas/permissionMode/querySource/queryTracking
  等。
- **`logAPISuccessAndDuration`（581-796 行）** —— 请求成功、流处理完成后：
  1. 探测 gateway（`detectGateway` 通过 header 前缀或 baseUrl 后缀识别
     litellm/helicone/portkey/cloudflare-ai-gateway/kong/braintrust/databricks）；
  2. 扫描 `newMessages` 计算 textContentLength / thinkingContentLength /
     toolUseContentLengths / connectorTextBlockCount；
  3. `logAPISuccess` 发 `tengu_api_success`（约 40 个字段的一大坨埋点）；
  4. `logOTelEvent('api_request', {...})` 发 OTLP event；
  5. 组织 modelOutput / thinkingOutput / hasToolCall（Beta tracing 用）；
  6. `endLLMRequestSpan(llmSpan, {success:true, tokens, ttftMs, requestSetupMs,
     attemptStartTimes})` —— 关掉 OTel + Perfetto 的 LLM span；
  7. Teleport 首消息成功特殊事件。
- **`logAPIError`（235-396 行）** —— 请求失败：
  1. 从 error 头（若 `APIError`）或 fallback headers 抽 gateway、status；
  2. `classifyAPIError(error)` 分类；
  3. `extractConnectionErrorDetails` 拿 socket/TLS 层原因写 debug log；
  4. `tengu_api_error` 事件 + `logOTelEvent('api_error', …)`；
  5. `endLLMRequestSpan(llmSpan, {success:false, statusCode, error, attempt})`。

**注意**：这两个函数都拿到 `llmSpan?: Span` 形参，是因为一个 LLM span 里可能
经历多次 retry，每次 attempt 都可能是 success 或 error，`endLLMRequestSpan`
里会把 `attemptStartTimes` 拆成 Perfetto 子 span，让 UI 上能看清"哪一次重试
成功、哪一次超时"。

### 3.5 tool 层的观测

`services/tools/toolExecution.ts` 的执行 loop 大致长这样：

```
startToolSpan(name, attrs, input)
  → startToolBlockedOnUserSpan()        (如果需要用户授权，见 span.type='tool.blocked_on_user')
    → endToolBlockedOnUserSpan()
  → startToolExecutionSpan()             (span.type='tool.execution')
    → tool.call(...)
    → endToolExecutionSpan({success, error})
endToolSpan(resultText, resultTokens)
```

`endToolSpan` 里通过 `addBetaToolResultAttributes` 把 result 也塞到 OTel 属性
里（受 `OTEL_LOG_TOOL_CONTENT=1` 二次控制，且经 `truncateContent` 截断上限）。
tool.execution 是 tool 内部真正 IO/CPU 花费的时间，把它和 tool 分开是为了
和"用户授权耗时"区分——Perfetto UI 上会看到两条相邻但意义不同的 track。

`addToolContentEvent(name, attrs)` 允许工具在执行过程中显式添加 span event
（比如 Bash 打印一段大 stdout），只在 `OTEL_LOG_TOOL_CONTENT=1` 时才写入。

### 3.6 三大子系统的相互串联

这里要点破一件关键事：**Checkpoint 的 snapshot 落盘是通过会话持久化写进 JSONL 的
`file-history-snapshot` entry**——所以：

- 会话持久化的 `Project.appendEntry` 特殊分支处理 `file-history-snapshot` 类型（1189-1191 行），
- 会话恢复 (`loadFullLog → buildFileHistorySnapshotChain`) 用 `messageId` 去重
  重放，得到 `fileHistorySnapshots[]`，
- `restoreSessionStateFromLog` 再把这个数组喂给 `fileHistoryRestoreStateFromLog`
  重建 FileHistoryState，
- 然后 Checkpoint 系统就可以再次 `applySnapshot` 到磁盘。

而**链路观测的 endLLMRequestSpan / endToolSpan 与埋点 `tengu_*` 是同一处业务
事件的双通道输出**，`logAPISuccessAndDuration` 一次调用就同时驱动：

1. `logAPISuccess`（analytics BigQuery）
2. `logOTelEvent`（OTel logs）
3. `endLLMRequestSpan` → OTel span end + Perfetto span end

这解释了为什么服务支撑层"看起来复杂"——不是因为有很多层，而是因为**同一个业务
点必须驱动 2~3 条相互独立、可分别关的旁路**，任何一条挂了都不能影响主流程。

---

## 4. 运行过程实录：一次用户 turn 的服务支撑层视角

以"用户输入 `请把 foo.ts 里的 X 改成 Y` → REPL 提交 → 一次 LLM 请求 →
Claude 调用 Edit 工具 → 完成"为例，服务支撑层的时间线：

```
t0  REPL onSubmit(prompt)
    ├── sessionStorage.recordTranscript([userMsg])
    │     └── Project.insertMessageChain
    │           └── materializeSessionFile()               ← 第一句话触发建文件
    │                 ├── mkdir + reAppendSessionMetadata (mode/agent-*)
    │                 └── replay pendingEntries
    │           └── appendEntry(userMsg) → enqueueWrite
    ├── sessionTracing.startInteractionSpan(prompt)         ← OTel + Perfetto span 起
    │
t1  QueryEngine.runTurn()
    ├── logAPIQuery({...})                                  ← tengu_api_query
    ├── sessionTracing.startLLMRequestSpan(model,...)        ← OTel + Perfetto span 起
    ├── services/api/claude.queryModelWithStreaming(...)     ← 真正 HTTPS 流
    │     └── APIError? → logAPIError → endLLMRequestSpan(fail)
    │     └── 正常 → logAPISuccessAndDuration
    │           ├── logAPISuccess → tengu_api_success
    │           ├── logOTelEvent('api_request', {...})
    │           └── endLLMRequestSpan(span, {success, tokens, ttft, attempts})
    │                 ├── endLLMRequestPerfettoSpan(...)   ← Perfetto: E event
    │                 └── span.setAttributes(...) + span.end()
    │
t2  Assistant 返回 tool_use=Edit
    ├── recordTranscript([assistantMsg]) → enqueueWrite
    │
t3  toolExecution runs Edit
    ├── startToolSpan('Edit', {...}, input)                  ← OTel + Perfetto tool span
    ├── fileHistoryTrackEdit(setState, path, msgId)
    │     ├── phase1: 检查 latest snapshot 是否已含此文件
    │     ├── phase2: async createBackup(path, v=1)           ← ~/.claude/file-history/<sid>/<hash>@v1
    │     └── phase3: state 更新 + recordFileHistorySnapshot(id, snap, true)
    │           └── Project.insertFileHistorySnapshot → appendEntry(file-history-snapshot)
    │                 → enqueueWrite → 100ms 后 fsAppendFile 一批落 JSONL
    ├── Edit.call() 实际写文件
    ├── recordTranscript([toolResultMsg]) → enqueueWrite
    └── endToolSpan(result, tokens)
          ├── endToolPerfettoSpan(...)
          └── OTel span.end()
    
t4  QueryEngine loop 无更多 tool_use，本轮结束
    ├── fileHistoryMakeSnapshot(setState, lastMsgId)
    │     ├── 遍历 trackedFiles，diff mtime/size 决定是否 createBackup(v=2)
    │     ├── new FileHistorySnapshot 追加，snapshots 超 100 则丢最旧
    │     └── recordFileHistorySnapshot(id, snap, false) → enqueueWrite
    └── endInteractionSpan()
          ├── endInteractionPerfettoSpan(...)
          └── span.setAttributes({interaction.duration_ms}) + span.end()

t5  用户再输一条 → 重复 t0 起，只是 materialize 已经完成不再触发；
    每 100ms 一次 Project.drainWriteQueue 把这一批 entries 追加到 JSONL。

t∞ 进程退出（Ctrl-C 或 exit）：
    cleanup handler:
      1. Project.flush()         等 timer + activeDrain + queue 完
      2. reAppendSessionMetadata 补写 last-prompt/tag/title 到尾部
      3. writePerfettoTrace()    写 ~/.claude/traces/trace-<sid>.json
      4. tracerProvider.forceFlush + shutdown（限时 2s）
```

从这个时间线看几件事：

1. **主业务链路只做 in-memory 状态更新和 fire-and-forget `enqueueWrite`**——
   不 await 落盘，所以 100ms 的批处理延迟不会拖慢 REPL。
2. **Checkpoint 与会话持久化耦合但方向单一**——Checkpoint 只写 entry 给
   sessionStorage，不反向读；恢复时 sessionStorage 提供 snapshot 序列，
   Checkpoint 用它重建 in-memory `FileHistoryState`。
3. **观测三通道相互独立**——关掉 Perfetto/OTel/analytics 其中任何一路，其他
   两路依然工作。
4. **崩溃恢复的关键窗口**是"`enqueueWrite` 排队但还没落盘"的那 100ms——SIGKILL
   下丢失是有意为之，其他任何退出路径都会走 `flush()`。

---

## 5. 关键设计取舍与不变式

| 取舍 | 选择 | 原因 |
| --- | --- | --- |
| **JSONL vs. SQLite** | JSONL 追加 | Append-only + 每行独立 parse，损坏一行不影响其他；便于外部工具/`grep` |
| **buffer 100ms 才落盘** | Yes（远端 10ms） | 主流程零阻塞；同一 flush 批量 append 命中一次 syscall |
| **首消息才 materialize** | Yes | 空会话不进 `--resume` 列表 |
| **快照 messageId 索引** | Yes（不是 hash） | rewind UX 是"回到那一条对话之前"，天然按消息定位 |
| **备份用 copyFile** | Yes（不是 readFile+writeFile） | 大文件不吃 heap；`chmod` 保留原权限 |
| **Backup 硬链跨 session** | Yes（fallback copy） | 一份内容多次 fork 只一份 inode |
| **Perfetto 单独一路** | Yes（不是 OTel exporter） | 输出 Chrome 格式 JSON 便于离线可视化；Ant-only, 独立启停 |
| **stale-span TTL 清理** | Yes（30min） | 防止业务忘 end 造成 pendingSpans 无限增长 |
| **shutdown timeout** | 2000ms | 慢的 OTLP endpoint 不能拖住进程退出 |
| **remote persistence 失败 → gracefulShutdown(1)** | Yes | 本地/远端不允许 divergence，宁停勿错 |

服务支撑层要维护的不变式（invariants）也就这几条：

1. **JSONL 上的 parentUuid 链在 `isChainParticipant` 消息之间必须连续**——`insertMessageChain`
   + `recordTranscript` + `getSessionMessages` 三者合力保证。
2. **同一个 sessionId 下 UUID 唯一**——`getSessionMessages` set 去重；例外是子 Agent
   sidechain 的本地文件写（因为写到不同文件）。
3. **FileHistorySnapshot 的 `messageId` 与 JSONL 里对应的 assistant/user 消息一一对应**
   ——`fileHistoryMakeSnapshot(messageId)` 与 REPL 结束一轮时的最后一条 message 对齐。
4. **Span 的父子关系 = AsyncLocalStorage 的层次关系**——interaction > llm_request /
   tool > tool.execution / hook。
5. **Perfetto 的 pid=1 恒为主 agent，subagent pid 单调递增**——AsyncLocalStorage +
   `agentIdToProcessId` 保证。

---

## 6. 从代码位置回到系统能力

如果你只想快速定位读哪一段：

- 要看**一条消息如何落地**：`Project.insertMessageChain` → `appendEntry` →
  `enqueueWrite` → `drainWriteQueue` → `appendToFile`。
- 要看**crash-safe/退出保护**：`getProject()` 里的 `registerCleanup`、
  `Project.flush`、`reAppendSessionMetadata`。
- 要看**resume 全链路**：`loadConversationForResume` → `loadTranscriptFile`
  → `buildFileHistorySnapshotChain` → `restoreSessionStateFromLog`
  → `fileHistoryRestoreStateFromLog` / `attributionRestoreStateFromLog`。
- 要看**每次编辑做了什么快照**：`fileHistoryTrackEdit` 三阶段 +
  `fileHistoryMakeSnapshot` 三阶段 + `createBackup`。
- 要看**回滚**：`fileHistoryRewind` → `applySnapshot` → `restoreBackup`。
- 要看**一次 LLM 请求被观测了几次**：`logAPIQuery`（发起）→ `startLLMRequestSpan`
  → `logAPISuccessAndDuration`（成功） / `logAPIError`（失败）
  → `endLLMRequestSpan` → `endLLMRequestPerfettoSpan`。
- 要看**exporter 是怎么按 protocol 动态加载的**：`getOtlpReaders` /
  `getOtlpTraceExporters` / `getOtlpLogExporters` 里的 `switch(protocol)`。

三条子系统是同一层的三条"横梁"，每一条都可以独立读懂；理解服务支撑层的关键
是把它们放到"事件流+旁路输出"的框架里来看：**会话持久化**是事件的事务日志，
**Checkpoint** 是文件系统的时间轴，**链路观测**是这些事件的第二/第三份副本
被送往可视化/监控后端。业务层不需要知道它们的存在，但它们保证了 Claude Code
在崩溃、回滚、性能分析上具备一个成熟工程系统应有的能力。
