# 会话持久化与恢复（sessionStorage / sessionRestore）

> 源码位置：`src/utils/sessionStorage.ts`、`src/utils/sessionRestore.ts`
> 核心单例：`Project`（transcript JSONL 的写入缓冲、去重、远端同步、查询加载）
> 关联模块：`src/utils/conversationRecovery.ts`（`loadConversationForResume`，恢复入口）、`src/hooks/useLogMessages.ts`（REPL 增量写盘）、`src/bootstrap/state.ts`（sessionId / cwd 单例）、`src/services/api/sessionIngress.ts`（v1 远端持久化）、`src/utils/sessionStoragePortable.ts`（大文件分段读取原语）
> 关联文档：[09-query.md](../09-query.md) · [32-query-engine.md](../32-query-engine.md) · [architecture/31-bootstrap-state.md](../architecture/31-bootstrap-state.md)

这两个模块共同实现 Claude Code 的"会话可恢复"能力：每一轮对话以 JSONL（每行一个 JSON entry）的形式追加写入 `~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl`；启动时通过 `--continue` / `--resume` 重建消息链、元数据（标题、标签、agent、模式、worktree、PR）、文件历史快照、todo 等运行态，并把进程切回该会话继续写入。

`sessionStorage.ts` 负责**写与读**（落盘、缓冲、去重、链构建、列表/搜索、远端 hydration）；`sessionRestore.ts` 负责**恢复编排**（拿到加载结果后切 session、恢复 agent/mode/worktree、计算初始 AppState）。

---

## 1. 文件布局与路径约定

```
~/.claude/projects/
└── <sanitizedPath(projectDir)>/        # getProjectDir()，按工作目录分组
    ├── <sessionId>.jsonl                # 主 transcript
    ├── <sessionId>/
    │   ├── subagents/
    │   │   ├── [subdir/]
    │   │   │   └── agent-<agentId>.jsonl   # 子 agent sidechain
    │   │   └── agent-<agentId>.meta.json   # agentType / worktreePath / description
    │   └── remote-agents/
    │       └── remote-agent-<taskId>.meta.json  # CCR 远端 agent 元数据
    └── ...
```

关键路径函数：

| 函数 | 作用 |
| --- | --- |
| `getProjectsDir()` | `~/.claude/projects` |
| `getProjectDir(cwd)` | memoize：`projects/` + `sanitizePath(cwd)` |
| `getTranscriptPath()` | 当前 session 的主文件，优先读 `getSessionProjectDir()`，否则按 `getOriginalCwd()` 推导 |
| `getTranscriptPathForSession(id)` | 给任意 sessionId 算路径；对"当前 session"会复用 `getTranscriptPath()`，避免 hook 看到路径漂移 |
| `getAgentTranscriptPath(agentId)` | 子 agent sidechain 文件，支持 `agentTranscriptSubdirs` 分组（如 workflow run） |

**路径一致性**是该模块反复修补的重点：symlink realpath、`switchSession`、worktree 切换、`--fork-session` 都可能导致"写在 A 目录、读时按 B 目录推导"。修复手段是统一以 `sessionProjectDir + sessionId` 为原子对（CC-34），当前 session 的路径一律走 `getSessionProjectDir()`。

---

## 2. transcript entry 类型

一行一个 JSON。顶层 `type` 区分两类：

**1. TranscriptMessage（参与对话链）**——`isTranscriptMessage()` 为 true：
- `user` / `assistant` / `attachment` / `system`

每条都带 `uuid`、`parentUuid`、`sessionId`、`cwd`、`timestamp`、`version`、`gitBranch`、`slug`、`userType`、`entrypoint`、`agentId`、`isSidechain`、`teamName/agentName` 等"session 戳"字段。这些字段在 spread 之后赋值，保证 fork/resume 时被重新盖戳为当前 session（否则 content-replacement 按 sessionId 查找会落空）。

> `progress` 消息**曾经**入链（PR #23537/#24099 之前），现已从 `isTranscriptMessage` 移除。旧 transcript 里残留的 progress 由 `loadTranscriptFile` 的 `progressBridge` 跳过并重接父链。`isChainParticipant()` 对非 progress 返回 true，用于写盘时决定是否推进 `parentUuid` 游标。

**2. 元数据 / 辅助 entry**（不参与 parentUuid 链）：
`summary`、`custom-title`、`ai-title`、`last-prompt`、`tag`、`agent-name/-color/-setting`、`mode`、`worktree-state`、`pr-link`、`file-history-snapshot`、`attribution-snapshot`、`content-replacement`、`queue-operation`、`marble-origami-commit/-snapshot`（context collapse）、`speculation-accept`、`agent-summary` 等。

`ai-title` 与 `custom-title` 刻意分开：读侧永远优先 `customTitle`，`restoreSessionMetadata` 只缓存 custom-title，避免陈旧 AI 标题覆盖用户中途改名。

---

## 3. 写路径：`Project` 单例

`getProject()` 惰性创建进程内唯一的 `Project`，并注册一次性 `cleanup` 钩子：退出时 `flush()` 所有排队写入，再 `reAppendSessionMetadata()` 把元数据补到文件尾部。

### 3.1 延迟物化（lazy materialize）

启动时 `sessionFile = null`。期间的 entry 先进入 `pendingEntries`；**第一条 user/assistant 消息**触发 `materializeSessionFile()`：创建文件、写元数据、冲刷缓冲。这避免了"只有元数据、没有对话"的孤儿 session 文件（例如 `-c -n foo` 后立即退出）。

- `--continue/--resume`（非 fork）走 `adoptResumedSessionFile()`：文件已存在，直接把 `sessionFile` 指向它并补写元数据，让退出清理钩子能正常工作。
- fork 不 adopt：REPL 挂载时由 `useLogMessages → recordTranscript` 在新 session 文件里惰性写入。

### 3.2 批量写队列

- `enqueueWrite(filePath, entry)`：按文件分组入队，`scheduleDrain()` 用 100ms 定时器（CCR 远端模式缩短到 `REMOTE_FLUSH_INTERVAL_MS = 10ms`）合并写入。
- `drainWriteQueue()`：把一个文件的多条 entry 拼成一段，单块超过 `MAX_CHUNK_BYTES = 100MB` 时分块 `appendFile`（权限 0o600，目录缺失则递归创建 0o700）。
- `trackWrite()` / `pendingWriteCount`：跟踪非队列写操作（如 tombstone 删除）；`flush()` 等计数器归零才 resolve。
- `flush()`：取消定时器 → 等待在途 drain → 再 drain 一次 → 等待计数器归零。

### 3.3 去重与父链：`recordTranscript()`

REPL（`useLogMessages`）和 QueryEngine 每轮把"当前完整消息数组"传进来，`recordTranscript` 负责只写新增部分：

1. `cleanMessagesForLogging()` 过滤不可记录消息；外部用户还要脱敏 REPL ID。
2. `getSessionMessages(sessionId)`（memoize）拿到磁盘已有 UUID 集合。
3. 顺序扫描：已存在的消息若构成"前缀"则记录其 UUID 作为 `startingParentUuid`；第一条新消息之后的旧消息（典型：compaction 里 `messagesToKeep` 出现在新 boundary/summary 之后）不跟踪。
4. 把 `newMessages` 交给 `insertMessageChain()`，返回最后一个真正落盘的链参与者 UUID（供调用方维持增量游标）。

`insertMessageChain()` 为每条消息：
- 处理 compact boundary（`parentUuid = null`，`logicalParentUuid` 指向逻辑父）；
- tool_result 用 `sourceToolAssistantUUID` 覆盖父 UUID（对应并行 tool_use 的单块 assistant 消息）；
- 盖 session 戳后 `appendEntry()`；
- 链参与者推进 `parentUuid` 游标。

### 3.4 `appendEntry()` 的路由

根据 entry 类型决定目标文件与是否去重：
- 元数据类（summary/title/tag/mode/worktree/pr-link/snapshot/...）：总是入队追加。
- `content-replacement`：带 `agentId` 写子 agent sidechain，否则写主文件。
- 普通 TranscriptMessage：
  - sidechain（`isSidechain && agentId`）→ 子 agent 文件，**绕过主文件去重集合**（否则会污染主线 `messageSet`，导致主线后续消息挂到只存在于 agent 文件的 UUID 上）。
  - 主线 → 用 `messageSet.has(uuid)` 去重；新 UUID 入队写盘、加入集合，并对 transcript 消息调用 `persistToRemote()`。

### 3.5 远端持久化

两条路径：

- **CCR v2 internal events**：`setInternalEventWriter` 注册后，transcript 消息作为 worker internal event 写出；flush 间隔 10ms。读取侧对应 `hydrateFromCCRv2InternalEvents()`（前台事件写主文件，子 agent 事件按 `agent_id` 分组写各自文件，服务端已按最新 compact boundary 过滤）。
- **v1 Session Ingress**：`ENABLE_SESSION_PERSISTENCE` + `remoteIngressUrl` 时通过 `sessionIngress.appendSessionLog` POST；失败打点并 `gracefulShutdownSync(1)`。对应 `hydrateRemoteSession()`：拉取远端日志整体覆盖本地文件。

`hydrateRemoteSession` / `hydrateFromCCRv2InternalEvents` 都先 `switchSession()` 再写文件，最后 `setRemoteIngressUrl()` 确保"先同步再开启持久化"。

### 3.6 Tombstone 删除

`removeMessageByUuid()` 删除流式失败遗留的孤儿消息：
- 快路径：读尾部 64KB（`LITE_READ_BUF_SIZE`），按字节搜索 `"uuid":"<id>"`，定位行边界后 `ftruncate` + 重写尾部。目标通常是最后一条。
- 慢路径：尾部没找到且文件 < 50MB（`MAX_TOMBSTONE_REWRITE_BYTES`）时整文件重写；超过则放弃并打点（防 OOM）。

### 3.7 元数据缓存与 re-append

`Project` 缓存当前 session 的 title/tag/agent/mode/worktree/PR 等字段。元数据 entry 会随对话增长被挤出读取侧的尾部窗口，因此两个时机要把它们重新追加到 EOF：

- **compaction 前后**（`compact.ts` / `reactiveCompact.ts`）
- **进程退出**（cleanup 钩子）

`reAppendSessionMetadata(skipTitleRefresh)`：
1. 同步读尾部（`readFileTailSync`），吸收外部写入者（SDK renameSession/tagSession）的最新 title/tag，避免陈旧缓存覆盖。
2. 按顺序重新追加：`last-prompt` → `custom-title` → `tag` → `agent-name/-color/-setting` → `mode` → `worktree-state` → `pr-link`（title/tag 更靠近 EOF）。
3. worktree 用三态：`undefined`（未碰过，不写）/ `null`（已退出 worktree）/ 对象（在 worktree 中），用于区分崩溃中途退出与正常退出。

写盘 API 分两类：
- **只改缓存**（避免启动期产生元数据-only 文件）：`saveAgentSetting`、`cacheSessionTitle`、`saveMode`、`saveWorktreeState`（文件不存在时）。
- **立即追加**（文件已存在或显式改名/标签/PR）：`saveCustomTitle`、`saveTag`、`saveAgentName/Color`、`linkSessionToPR`、`saveAiGeneratedTitle`、`saveTaskSummary`。

`restoreSessionMetadata(meta)` 在 resume 时把读到的值填回缓存（`??=` 保证 `--name` 优先级高于磁盘标题）；`clearSessionMetadata()` 在 `/clear` 新会话时清空。

### 3.8 sidechain / 子 agent

- `recordSidechainTranscript()` 以 `isSidechain=true` 写子 agent 文件，带 `agentId` 与可选 `startingParentUuid`。
- `writeAgentMetadata` / `readAgentMetadata`：sidecar `.meta.json` 记录 `agentType`、`worktreePath`、`description`，让 resume 时即便没传 `subagent_type` 也能正确路由（而不是静默降级为 general-purpose）。
- `setAgentTranscriptSubdir()` 支持把一组相关子 agent（如 workflow run）归到子目录。
- `writeRemoteAgentMetadata` / `listRemoteAgentMetadata` / `deleteRemoteAgentMetadata`：CCR 远端 agent 身份持久化（状态总是实时从 CCR 拉，本地只存身份）。

---

## 4. 读路径：从 JSONL 到可恢复对话链

### 4.1 `loadTranscriptFile()`（核心解析器）

返回一组 Map：`messages`、`summaries`、`customTitles`、`tags`、`agentNames/Colors/Settings`、`prNumbers/Urls/Repositories`、`modes`、`worktreeStates`、`fileHistorySnapshots`、`attributionSnapshots`、`contentReplacements`（按 sessionId）、`agentContentReplacements`（按 agentId）、`contextCollapseCommits`（数组，提交有序）、`contextCollapseSnapshot`（last-wins）、`leafUuids`。

**大文件优化栈**（三层，全部可被 `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` 关闭）：

1. **fd 级 pre-boundary 裁剪**（`readTranscriptForLoad`，在 portable 模块）：文件 > `SKIP_PRECOMPACT_THRESHOLD` 时，只读取最后一个 compact boundary 之后的字节；attribution-snapshot 行在 fd 层就跳过，峰值内存≈输出大小而非文件大小。被裁掉区域里的会话级元数据由 `scanPreBoundaryMetadata()` 用字节级 marker 前向扫描补回（`METADATA_TYPE_MARKERS`，整 chunk 无 marker 时不切行）。
2. **解析前死支裁剪**（`walkChainBeforeParse`）：在 buffer 上用字节扫描建立 `[lineStart, lineEnd, parentStart]` 索引，从最后一个非-sidechain 叶子沿 `parentUuid` 走到根，只把活链 + 元数据按原序拼回。对 fork 密集的 transcript（41MB / 99% 死支）实测 parseJSONL 从 56ms→3.9ms。依赖两个不变量：transcript 消息 `parentUuid` 是首键；顶层 uuid 用 `","timestamp":"` 后缀 + 花括号深度消歧（`pickDepthOneUuidCandidate`）。保留段（preservedSegment）边界时禁用，因为这些消息保留旧 parentUuid、要在解析后由 `applyPreservedSegmentRelinks` 拼接。
3. 解析后：`parseJSONL<Entry>(buf)` 逐行分发到各 Map。

**旧 progress 桥接**：遇到 legacy progress 记录 `progress → parent`（链式解析连续 progress），后续消息的 `parentUuid` 若落在桥上则重写为最近的非 progress 祖先。

**后处理**：
- `applyPreservedSegmentRelinks()`：处理"保留段"压缩——把保留消息的头 relink 到 anchor、anchor 的其他孩子 relink 到 tail，并清零保留 assistant 消息的陈旧 usage（否则 resume 后立即触发自动压缩螺旋）；最后物理删除绝对最后边界之前的非保留消息。若 tail→head 回放在磁盘上断裂，则放弃 prune 整段加载（安全降级）。
- `applySnipRemovals()`：处理 HISTORY_SNIP 中段删除——按 `snipMetadata.removedUuids` 删除，并把删除区后方幸存者的 dangling `parentUuid` 沿被删消息的父链回溯到第一个未删祖先（带路径压缩）。
- **leaf 计算**：所有非父节点（terminal）向上回溯到最近的 user/assistant 作为叶子；`tengu_pebble_leaf_prune` 开启时会跳过"有 user/assistant 孩子"的中间节点。检测到环打点 `tengu_transcript_parent_cycle`。

### 4.2 链构建：`buildConversationChain()`

从叶子沿 `parentUuid` 反向走到根（带环检测），再 reverse。然后 `recoverOrphanedParallelToolResults()` 处理 DAG 拓扑：

流式输出时 N 个并行 tool_use 产生 N 个单块 assistant 消息（不同 uuid、相同 `message.id`），每个 tool_result 的 `parentUuid` 指向各自的 assistant。单父链表遍历只会保留一条分支，漏掉兄弟 assistant 和它们的 tool_result。恢复逻辑按 `message.id` 分组，把链外的兄弟 assistant + 其 tool_result 按 timestamp 排序后插到链上锚点之后，保证每个 tool_result 落在其 tool_use 之后、且组内连续以便后续合并。旧 transcript 的 progress-fork 形态也一并兼容。

### 4.3 会话列表与渐进加载

- `fetchLogs(limit)`：`getSessionFilesLite` 只扫描各文件头部/尾部的有限窗口（`readLiteMetadata` / `readHeadAndTail`）拿到 `firstPrompt`、title、tag、messageCount、时间等，返回 `messages: []` + `sessionId` 的 **lite log**。
- `isLiteLog(log)` / `loadFullLog(log)`：选中某个会话时再 `loadTranscriptFile` → 找最近叶子 → `buildConversationChain` → 填充完整字段（包括按 sessionId 过滤的 contentReplacements、contextCollapseCommits 等）。
- `loadTranscriptFromFile(path)`：支持 `.jsonl` 和旧版 `.json`（数组或 `{messages}`）。
- `searchSessionsByCustomTitle()`：跨同 repo worktree 加载并按标题模糊/精确匹配，按 sessionId 去重、按时间排序。
- `getLastSessionLog(sessionId)`：单文件全量加载，并预热 `getSessionMessages` 缓存（仅当缓存为空，避免用陈旧磁盘快照覆盖未 flush 的 UUID）。
- `checkResumeConsistency(chain)`：找最近 `turn_duration` 检查点，对比其记录的 `messageCount` 与实际链位置，上报 `delta` 用于监控写→读往返漂移（snip/compact/parallel-TR 类 bug）。

### 4.4 对外数据形状：`LogOption`

`convertToLogOption()` 把一条链转成列表/恢复用的描述符：`date`、`messages`（经 `removeExtraFields` 剥掉 `isSidechain/parentUuid`）、`firstPrompt`、`messageCount`（`countVisibleMessages` 只数有可见文本/图片的 user/assistant）、`created/modified`、`leafUuid`、`summary`、`customTitle/tag`、`gitBranch`、`projectPath`、各类 snapshots 与 `contentReplacements`。

`extractFirstPrompt()` / `getFirstMeaningfulUserMessageTextContent()` 提取标题用首条 prompt：跳过 meta、compact summary、IDE 上下文标签（`<ide-...>`）、hook 输出、内置斜杠命令（如 `/model`），但保留带参数的自定义命令、`! <bash>` 输入。

---

## 5. 恢复编排：`sessionRestore.ts`

`sessionStorage` 给出"磁盘上有什么"，`sessionRestore` 决定"如何把进程切到那个会话并准备首屏状态"。真正的入口是 `conversationRecovery.ts:loadConversationForResume()`：

```
loadConversationForResume(source, sourceJsonlFile)
  ├─ 定位：--continue(最近) / sid / .jsonl 路径 / 已有 LogOption
  ├─ lite → loadFullLog()
  ├─ copyPlanForResume / copyFileHistoryForResume
  ├─ checkResumeConsistency()
  ├─ restoreSkillStateFromMessages()
  ├─ deserializeMessagesWithInterruptDetection()  // 未结束的 tool_use、turnInterruptionState
  └─ processSessionStartHooks('resume') 追加 hook 消息
```

它返回 `ResumeLoadResult`（消息 + 全部会话元数据 + `fullPath`），交给：

### 5.1 `processResumedConversation()`（CLI `--continue/--resume` 主路径，`main.tsx` 调用）

按顺序：

1. **模式匹配**：`COORDINATOR_MODE` 下用 `modeApi.matchSessionMode(result.mode)`，若切换模式则追加一条 warning system message。
2. **session ID 切换**：
   - 非 fork：`switchSession(sid, transcriptDirname?)`（跨目录 resume 时传文件目录作为 sessionProjectDir）→ 重命名 asciicast 录音 → `resetSessionFilePointer()` → `restoreCostStateForSession()`。
   - fork：保留新启动 sessionId；若源会话有 content-replacement，先 `recordContentReplacement()` 种入新文件（否则源 tool_use_id 被判为 FROZEN，cache miss）。
3. **恢复元数据缓存** `restoreSessionMetadata(...)`；fork 路径剥离 `worktreeSession`（fork 不接管原会话 worktree，避免 fork 退出时误删）。
4. **worktree 恢复** `restoreWorktreeForResume()`：
   - 若 `--worktree` 已创建新 worktree（`getCurrentWorktreeSession()` 真值），以新的为准并 `saveWorktreeState(fresh)`。
   - 否则若记录了 worktree，`process.chdir` 进去（ENOENT 表示目录已删，则写 `null` 标记退出），`setCwd/setOriginalCwd`、`restoreWorktreeSession`，并清 memory/systemPrompt/plans 缓存。
5. **接管文件**：非 fork `adoptResumedSessionFile()` 指向已存在文件并补元数据。
6. **context-collapse 恢复**：`CONTEXT_COLLAPSE` 下 `restoreFromEntries(commits, snapshot)`（无条件执行以 reset store，防止 `/resume` 到无 commit 会话时残留旧日志）。
7. **agent 恢复** `restoreAgentFromSession()`：若 CLI 没显式 `--agent`，按 `agentSetting` 找回 `AgentDefinition`，`setMainThreadAgentType` + 应用其 model override；agent 已不存在则降级默认。
8. **持久化当前 mode** `saveMode(...)`。
9. **计算首屏前的初始状态**：`computeRestoredAttributionState()`、`computeStandaloneAgentContext()`（name/color）、`updateSessionName`、`refreshAgentDefinitionsForModeSwitch()`（模式切换时重新派生内置 agent 并合并 CLI agents）。
10. 返回 `ProcessedResume`：`messages`、snapshots、`contentReplacements`、agent 名色、`restoredAgentDef`、拼好的 `initialState`（含 `agent`、`attribution`、`standaloneAgentContext`、`agentDefinitions`）。

### 5.2 `restoreSessionStateFromLog()`（运行态回灌）

被 SDK（`print.ts`）和交互式（`REPL.tsx`、`main.tsx`）共用，把加载结果回灌进 `AppState`：

- **文件历史**：`fileHistoryRestoreStateFromLog(snapshots, setter)`。
- **提交归因**（`COMMIT_ATTRIBUTION`）：`attributionRestoreStateFromLog`。
- **context-collapse**：同上 `restoreFromEntries`。
- **TodoWrite**（非 v2 tasks 时）：`extractTodosFromTranscript()` 从末尾向前找最后一个 `TodoWrite` tool_use，用 `TodoListSchema` 校验后塞进 `AppState.todos[sessionId]`，使 SDK `--resume` 无需文件持久化也能保住 todo。

`computeRestoredAttributionState()` / `computeStandaloneAgentContext()` 是纯函数版本，用于渲染前算初始值（CLI 路径偏好"渲染前算好"而非渲染后 setAppState）。

### 5.3 worktree 的中途切换

- `exitRestoredWorktree()`：会话内 `/resume` 切到另一个会话前撤销当前恢复的 worktree——切回 originalCwd、清缓存、`restoreWorktreeSession(null)`。否则会停在旧 worktree 目录，且切到另一个 worktree 会被守卫拦住。CLI 启动路径不需要它（启动时 `getCurrentWorktreeSession()` 仅在 `--worktree` 时为真，已由"fresh 优先"处理）。

---

## 6. 端到端流程

### 6.1 写盘（运行中）

```
QueryEngine/REPL 每轮产出 messages[]
  └─ useLogMessages (REPL) / QueryEngine (SDK)
       └─ recordTranscript(messages, teamInfo, parentHint)
            ├─ cleanMessagesForLogging
            ├─ getSessionMessages(sessionId)  ← 磁盘 UUID 集合(memoize)
            ├─ 前缀去重，得 newMessages + startingParentUuid
            └─ Project.insertMessageChain()
                 ├─ 首条 user/assistant → materializeSessionFile()
                 ├─ 逐条盖 session 戳 → appendEntry()
                 │    ├─ 主线新 UUID: enqueueWrite + messageSet.add + persistToRemote
                 │    └─ sidechain: 写 agent 文件，不动主集合
                 └─ 更新 currentSessionLastPrompt
       （100ms 定时）drainWriteQueue → appendFile(0o600)
退出/compact → flush() → reAppendSessionMetadata()
```

### 6.2 恢复（`--continue/--resume`）

```
main.tsx
  └─ loadConversationForResume()         conversationRecovery.ts
       ├─ (lite) loadFullLog → loadTranscriptFile + buildConversationChain
       ├─ checkResumeConsistency
       └─ deserialize + hooks → ResumeLoadResult
  └─ processResumedConversation(result)  sessionRestore.ts
       ├─ switchSession / resetSessionFilePointer
       ├─ restoreSessionMetadata
       ├─ restoreWorktreeForResume
       ├─ adoptResumedSessionFile（非 fork）
       ├─ restoreAgentFromSession + saveMode
       └─ 组装 initialState
  └─ restoreSessionStateFromLog()        回灌 AppState（fileHistory/attribution/collapse/todos）
  └─ 渲染 REPL / 跑 print
后续写入 → recordTranscript（已 adopt 的文件继续追加；fork 由 useLogMessages 惰性新建）
```

### 6.3 `/resume`（会话中切换）

交互式通过 `ResumeConversation.tsx` 选定会话 → 同样 `loadConversationForResume` → 若当前在恢复的 worktree 中先 `exitRestoredWorktree()` → `restoreSessionStateFromLog` 等回灌，REPL 用新 messages 重渲染。

---

## 7. 关键设计约束与坑点

1. **append-only + parentUuid 链**：JSONL 永不重写（tombstone 除外），rewind/fork/snip/compact 都靠"读时沿父链选择活支 + 元数据 last-wins"。因此写盘必须保证父在子之前（append 顺序天然满足）。
2. **session 戳在 spread 之后**：fork/resume 时源消息带旧 sessionId，必须重新盖戳，否则 content-replacement 按 sessionId 查找落空 → FROZEN 误判 → cache 永久超用。
3. **sidechain 与主文件去重集合隔离**：子 agent 继承主线父消息 UUID，绝不能加入主 `messageSet`，否则主线丢消息、远端 409。
4. **元数据尾窗**：lite 列表只读头尾窗口，`reAppendSessionMetadata` 保证 title/tag 不被挤出；外部 SDK 写入通过 tail refresh 吸收。
5. **worktree 三态与 fresh 优先**：`undefined/null/对象` 区分未碰/已退出/在其中；`--worktree` 新建的 worktree 永远覆盖转录里的陈旧状态。
6. **大文件 OOM 防护**：tombstone 50MB 上限、transcript 读取 50MB 上限、三层裁剪、attr-snap fd 级跳过、64KB carry 上限。
7. **特性开关**：`CONTEXT_COLLAPSE`、`COORDINATOR_MODE`、`COMMIT_ATTRIBUTION`、`PROACTIVE/KAIROS`、`BG_SESSIONS` 等在本反编译版中 `feature()` 恒为 false，相关分支为死代码（但读取侧的兼容逻辑如 progressBridge、snip/relink 仍对旧 transcript 生效）。
8. **`feature()` 恒 false 的影响**：远端持久化（`ENABLE_SESSION_PERSISTENCE` 是 env 控制，非 feature flag）、CCR、context-collapse、coordinator、attribution 在本构建中默认不走；本地 JSONL 读写、worktree、agent sidechain、title/tag/PR、todo 恢复均可用。

---

## 8. 主要导出速查

**sessionStorage.ts**

- 路径/目录：`getProjectsDir`、`getProjectDir`、`getTranscriptPath(ForSession)`、`getAgentTranscriptPath`、`sessionIdExists`
- 写：`recordTranscript`、`recordSidechainTranscript`、`recordQueueOperation`、`recordFileHistorySnapshot`、`recordAttributionSnapshot`、`recordContentReplacement`、`recordContextCollapseCommit/Snapshot`、`removeTranscriptMessage`、`flushSessionStorage`
- 元数据：`saveCustomTitle`、`saveAiGeneratedTitle`、`saveTaskSummary`、`saveTag`、`saveAgentName/Color/Setting`、`saveMode`、`saveWorktreeState`、`linkSessionToPR`、`cacheSessionTitle`、`restoreSessionMetadata`、`clearSessionMetadata`、`reAppendSessionMetadata`、`getCurrentSessionTag/Title/AgentColor`
- 文件接管：`resetSessionFilePointer`、`adoptResumedSessionFile`
- 子 agent：`set/clearAgentTranscriptSubdir`、`write/readAgentMetadata`、`write/read/list/deleteRemoteAgentMetadata`
- 远端：`hydrateRemoteSession`、`hydrateFromCCRv2InternalEvents`、`setRemoteIngressUrlForTesting`、`setInternalEventWriter/Reader`
- 读：`loadTranscriptFile`、`loadTranscriptFromFile`、`loadFullLog`、`fetchLogs`、`getLastSessionLog`、`searchSessionsByCustomTitle`、`isLiteLog`、`getSessionIdFromLog`
- 链/工具：`buildConversationChain`、`checkResumeConsistency`、`isTranscriptMessage`、`isChainParticipant`、`isEphemeralToolProgress`、`getFirstMeaningfulUserMessageTextContent`、`removeExtraFields`、`clearSessionMessagesCache`、`doesMessageExistInSession`

**sessionRestore.ts**

- 编排：`processResumedConversation`、`restoreSessionStateFromLog`
- worktree：`restoreWorktreeForResume`、`exitRestoredWorktree`
- agent/mode：`restoreAgentFromSession`、`refreshAgentDefinitionsForModeSwitch`
- 纯计算：`computeRestoredAttributionState`、`computeStandaloneAgentContext`
- 类型：`ProcessedResume`、`ResumeResult`
