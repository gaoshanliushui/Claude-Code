# `src/utils/sessionStorage.ts` 设计文档

> 单一来源,**"会话的所有持久化与恢复都在这一个文件里"**——
> 既负责把当前会话的 transcript / metadata / 子代理 transcript 写进磁盘,
> 也负责把磁盘上的 NDJSON 反向解码为 resume picker 与 REPL 用的 `LogOption` / `Message[]`。
>
> 文件约 5100 行,代码量接近 `project` 的 4% 但承担了 resume / persistence 的几乎全部核心工作。

---

## 1. 角色定位

- **写入侧**:REPL 内部产生的所有事件(transcript 消息、compression boundary、file-history snapshot、attribution snapshot、content replacement、custom title / tag / agent name / color / setting / mode / PR link / worktree-state / context-collapse commits / snapshots)在这里落盘。
- **读取侧**:把 `~/.claude/projects/<encoded-cwd>/<sid>.jsonl` 解析成 `LogOption` 列表,给 `/resume` picker 用;进一步把单个 session 解析成 `Message[]`,给 `loadConversationForResume → processResumedConversation` 用。
- **状态机**:`Project` 单例持有"当前 session"的各种缓存字段(title / tag / agent / mode / worktree / PR / lastPrompt),与 NDJSON 双向同步(写完缓存和文件,文件也回流校准缓存)。
- **IO 形态**:默认写本地文件,但同一接口还能写 v1 Session Ingress (HTTP POST 到 `remoteIngressUrl`)和 v2 CCR internal events(由 daemon 提供的 `internalEventWriter`);三种后端在 `Project` 上对调用方透明。

---

## 2. 高层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│ 调用方 (REPL / query / cli.print / /resume / ResumeConversation)    │
└────────────┬───────────────────────────────────────────┬────────────┘
             │ 写                                         │ 读
             ▼                                             ▼
┌─────────────────────────────┐         ┌────────────────────────────────┐
│ Module-level wrappers       │         │ List loaders                   │
│  recordTranscript           │         │  loadMessageLogs / loadAll…    │
│  recordSidechainTranscript  │         │  loadSameRepoMessageLogs(Prog) │
│  recordQueueOperation       │         │  getStatOnlyLogsForWorktrees   │
│  removeTranscriptMessage    │         │  getSessionFilesLite           │
│  recordFileHistorySnapshot  │         │  enrichLogs / enrichLog        │
│  recordAttributionSnapshot  │         │  searchSessionsByCustomTitle   │
│  recordContentReplacement   │         └────────────────┬───────────────┘
│  resetSessionFilePointer    │                          │
│  adoptResumedSessionFile    │                          ▼
│  recordContextCollapse{…}   │         ┌────────────────────────────────┐
│  flushSessionStorage        │         │ loadTranscriptFile             │
│  hydrateRemoteSession       │         │  ── readHeadAndTail (portable) │
│  hydrateFromCCRv2…          │         │  ── walkChainBeforeParse       │
└────────────┬────────────────┘         │  ── scanPreBoundaryMetadata    │
             │                          │  ── parseJSONL                 │
             ▼                          │  ── applyPreservedSegmentRelnk │
┌─────────────────────────────┐         │  ── applySnipRemovals          │
│ Project (singular, lazy)    │         │  ── leaf UUID computation      │
│  内部缓存:currentSession*   │         └────────────────┬───────────────┘
│  sessionFile / pendingEntries│                          │
│  writeQueues (per-file)     │                          ▼
│  flushTimer / activeDrain   │         ┌────────────────────────────────┐
│  persistToRemote(v1/v2)     │         │ loadTranscriptFromFile         │
│  reAppendSessionMetadata    │         │  ┌── loadFullLog (lite→full)   │
│  materializeSessionFile     │         │  ├── getLastSessionLog         │
│  removeMessageByUuid        │         │  └── loadAllLogsFromSessionFile│
└────────────┬────────────────┘         └────────────────────────────────┘
             │
             ▼
┌─────────────────────────────┐
│ 文件系统 / CCR ingress      │
│  ~/.claude/projects/.../…   │
│  ~/.claude/file-history/…/  │
│  ~/.claude/sessions/…/cast  │
│  + CCR v1 / v2 remote sink  │
└─────────────────────────────┘
```

---

## 3. 路径与会话 ID 工具

| 函数 | 行号 | 职责 |
| --- | --- | --- |
| `getProjectsDir()` | 199 | `~/.claude/projects/` |
| `getTranscriptPath()` | 203 | 当前 session 的 NDJSON 路径,跟随 `getSessionProjectDir()`(被 `switchActiveSession` / `switchSession` 改写) |
| `getTranscriptPathForSession(sid)` | 208 | 对其他 session 推导路径;对**当前** sid 走 `getTranscriptPath()`(gh-30217 — 否则 hooks 拿到的路径会与实际写入路径错开) |
| `getProjectDir(cwd)` | 437 | `memoize` 后的 `~/.claude/projects/<sanitizePath(cwd)>` |
| `getAgentTranscriptPath(agentId)` | 248 | `…/<sid>/subagents[/<subdir>]/agent-<agentId>.jsonl`;注册 subdir 的函数 `setAgentTranscriptSubdir` 用于 workflow 群组 |
| `sessionIdExists(sid)` | 402 | 同步 `fs.stat`,用于远程 / SDK guard |
| `agentTranscriptSubdirs` Map | 235 | 进程级缓存,workflow 群组通过 setAgentTranscriptSubdir 注册子目录 |

设计要点:

- **不要缓存** `getTranscriptPath` —— 它依赖 `getSessionProjectDir()`,而后者由 `switchActiveSession` 改写,模块级缓存会与运行期状态错位(注释 108-112 明确警告过 split-brain)。
- **Windows 大小写**:worktree 路径比对(`getStatOnlyLogsForWorktrees` 4134-4147)在 `win32` 平台做 `toLowerCase()`,避免 `C:/…` 与 `c:/…` 错配。
- **`MAX_TRANSCRIPT_READ_BYTES = 50 MB`**(230)是读取路径的安全上限;`MAX_TOMBSTONE_REWRITE_BYTES = 50 MB`(123)是 tombstone 慢路径的安全上限。

---

## 4. `Project` 单例 —— 写入侧的核心

### 4.1 单例化与生命周期

```
let project: Project | null = null      (441)
let cleanupRegistered = false           (442)

function getProject(): Project         (444)
```

- 第一次 `getProject()` 时 `new Project()` 并把"flush + reAppendSessionMetadata"注册为 cleanup handler。
- 测试可调用 `resetProjectFlushStateForTesting`、`resetProjectForTesting`、`setSessionFileForTesting` 等钩子。

### 4.2 进程级缓存

```
currentSessionTag / Title / AgentName / AgentColor / LastPrompt / AgentSetting / Mode
currentSessionWorktree: undefined | null | PersistedWorktreeSession
currentSessionPrNumber / PrUrl / PrRepository
sessionFile: string | null
pendingEntries: Entry[]                         (553,首个 user/assistant 之前的缓冲)
pendingWriteCount / flushResolvers              (584, / 559,trackWrite 同步计数)
writeQueues: Map<filePath, [{entry, resolve}]>  (562,per-file 顺序队列)
flushTimer / activeDrain                        (566/567, 防止并发 drain)
FLUSH_INTERVAL_MS = 100 (默认) / 10 (CCR)        (568 / 531)
MAX_CHUNK_BYTES = 100 MB                        (569)
```

设计要点:

- 三态 worktree:`undefined` = 从未触碰(不写)、`null` = 显式退出 worktree、`object` = 当前在 worktree。`reAppendSessionMetadata` 仅在三态非 undefined 时落盘。
- `currentSessionLastPrompt` 在 `insertMessageChain` 末尾 cache(1081-1083),供 `reAppendSessionMetadata` 重写到 JSONL 末尾;每次新 turn 覆盖。
- `currentSessionTitle ??= meta.customTitle`(`restoreSessionMetadata` 2779):`--name` 启动的标题优先,resume 加载的同名不会覆盖之(REPL.tsx 在 resume 前先 `clearSessionMetadata`,所以该不变量"在 resume 期间失效 / 不在 resume 期间失效"区分明确)。

### 4.3 写入节流与批量落盘

```
enqueueWrite(filePath, entry)   (608) ─┐
scheduleDrain()                  (620)  │ 100ms 单 timer 触发一次 drain
drainWriteQueue()                (647)  ├─ 按 filePath 分组,splice 出 batch
appendToFile(filePath, content)  (636)  │ 异步追加 + ENOENT 时 mkdir 兜底
flush()                         (843) ─┘ 取消 timer,等 activeDrain,最后再 drain 一次
```

特点:

- **顺序保证**:`enqueueWrite` 进入 per-file 队列,仅同一 filePath 内串行;不同 filePath 并行 drain。这意味着 `enqueueWrite(fileA)` 与 `enqueueWrite(fileB)` 实际落盘顺序与 timer 抖动相关,但**同一 file 内绝对按入队顺序**。
- **每条 resolve**:`enqueueWrite` 返回 Promise,调用方可以 await 自己那一条写完;drain 用 `resolvers` 数组同步通知。
- **chunked flush**:当累积字节 ≥ `MAX_CHUNK_BYTES` 时立即写盘并 resolve(避免单条 100MB+ 的长 tool result 让 JS buffer 爆炸)。
- **`MAX_CHUNK_BYTES` vs `:appendToFile`**:appendToFile 单次追加最多 ~100MB,虽然异步但单次 syscall;drain 切分是把巨型 entry 切成 ~2 个 OS write 的办法。
- **CCR v2 自动加速**:`setRemoteIngressUrl` 与 `setInternalEventWriter` 一旦被调用,`FLUSH_INTERVAL_MS` 立刻从 100ms 切到 10ms(1353-1354, 1363-1364)。

### 4.4 `reAppendSessionMetadata` —— 启动 / compaction / exit 共用的尾部对齐

签名 `reAppendSessionMetadata(skipTitleRefresh = false)`(723-841)。

意图:让 `custom-title / tag / agent-name / agent-color / agent-setting / mode / worktree-state / pr-link / last-prompt` 始终贴近 EOF,这样 progressive loading 时 `readLiteMetadata`(4748)扫描尾部 64KB 就能命中,不会被后续大消息挤出窗口。

关键步骤:

1. **tail 同步扫描**(`readFileTailSync`,2597):`openSync + fstatSync + readSync` 读最后 `LITE_READ_BUF_SIZE` 字节;结果是 `--resume` 在跨进程场景下能看到 SDK 改写的最新 title/tag。
2. **解析 SDK 写入的 title/tag**(`extractLastJsonStringField` from `sessionStoragePortable.ts`):用 JSON 字节搜索拿字符串字段,不用 parseJSONL。
3. **三重吸收**:
   - 不跳过 title refresh → 把 SDK 写的 title 写入自己的 `currentSessionTitle`,把可能存在的空串视为清空。
   - tag 同理。
   - lastPrompt / agent-* / mode / pr-link 是进程独占字段(SDK 不写),无条件重写。
4. **写入顺序**:lastPrompt 先,customTitle / tag 等较关键字段后——保证热字段尽量贴近 EOF。
5. **worktree-state**:仅在非 `undefined` 时落盘(让"未进入 worktree"和"显式退出 worktree"都可区分)。
6. **pr-link**:三个字段必须同时 truthy,任意缺失就放弃。

调用点:

- `materializeSessionFile`(986, 首次 user/assistant 时);
- cleanup handler(`getProject()` 注册的 `registerCleanup`, 451-462);
- `compact.ts / reactiveCompact.ts` 在 compaction 写 boundary 之前;
- 用户级导出 `reAppendSessionMetadata()`(2821) 在 REPL 显式调用;
- `adoptResumedSessionFile`(1537) 传 `skipTitleRefresh=true`,理由见注释 1528-1532:resume 刚把同一份数据从 disk 读进 cache,refreshing 又写一次会导致 `--name foo + -r <sid>` 中 `--name` 被旧 disk title 盖掉。

### 4.5 `materializeSessionFile` —— 首次 user/assistant 落地

(978-993)

```
第一次有 user/assistant 进来时:
1. ensureCurrentSessionFile()   // sessionFile = getTranscriptPath()
2. reAppendSessionMetadata()    // mode/agentSetting pre-materialization 落盘
3. flush pendingEntries          // 把 mode/setting → queue → 文件
```

需要 `reAppendSessionMetadata` 在此被调用的原因:

- `saveAgentSetting(agentSetting)`(2867) 与 `cacheSessionTitle`(2876)、`saveMode`(2885) 都是 **cache-only**,避免启动时写文件造成"metadata-only session"孤儿。
- 真要落盘的时刻就是首个 user/assistant 触发这个 materialization。

如果启用了 `--no-session-persistence` 或测试环境(`shouldSkipPersistence`,962),直接 return。

### 4.6 `appendEntry` —— 单条 dispatch 总入口

(1131-1268)

逻辑树:

```
if shouldSkipPersistence → return
isCurrentSession = sessionId === getSessionId()
if isCurrentSession && sessionFile === null:
    pendingEntries.push(entry); return   // 等 materialize
if !isCurrentSession:
    resolve via getExistingSessionFile (缓存 positive results,1145)
switch entry.type:
    summary / custom-title / ai-title / last-prompt / task-summary / tag /
    agent-name / agent-color / agent-setting / pr-link /
    file-history-snapshot / attribution-snapshot / speculation-accept /
    mode / worktree-state / marble-origami-commit / marble-origami-snapshot /
    queue-operation:
        enqueueWrite(file, entry)   // 总是追加,不查重
    content-replacement:
        走 agent 文件(sessionId-keyed vs agentId-keyed 双路) — 实现 AgentTool resume 与 /resume 各自正确的归属。
    其他(TranscriptMessage / user/assistant/attachment/system):
        const messageSet = await getSessionMessages(sessionId)
        isAgentSidechain = isSidechain && agentId !== undefined
        isNewUuid = !messageSet.has(entry.uuid)
        if isAgentSidechain || isNewUuid:
            enqueueWrite(target, entry)
            if !isAgentSidechain:
                messageSet.add(entry.uuid)        // 本地 messageSet 增量
                if isTranscriptMessage:
                    persistToRemote(sessionId, entry)
```

**dedup 用意**(`messageSet`):`recordTranscript` 等模块级包装已做 "已记录 UUID 不重写";`appendEntry` 也独立检查 `messageSet` —— 任何路径(REPL streaming、QueryEngine 单独写入、cleanup 批量 flush、bridges)都受同一闸门。注释 1250-1259 给出了关键约束:**sidechain UUID 不能入主 messageSet**,否则 recordTranscript 跨主链时把它们当作主链成员 dedup,后续用户消息 parentUuid 指向 agent-only UUID,`buildConversationChain` 撞墙。

**入口分流**:

- `recordTranscript`(`sessionStorage.ts` 1412-1453):批量入口,做去重后再投递 `insertMessageChain`;返回最后真正记录的 chain-participant UUID 或 prefix-tracked UUID,使 `useLogMessages` 维持正确的 parent chain。
- `recordSidechainTranscript`(1455):与 `recordTranscript` 不同,它永远跳过 dedup,直接调 `insertMessageChain(..., isSidechain=true, agentId, startingParentUuid)`。
- 各类 `record*` 模块级包装都转调 `Project` 同名方法,通过 `trackWrite` 计数 pending write,便于 `flush()` 等待。

### 4.7 `insertMessageChain` —— 把一组消息挂上 parent 链

(995-1086)

每条消息做:

```
transcriptMessage = {
    parentUuid     : isCompactBoundary ? null : effectiveParentUuid,
    logicalParentUuid: isCompactBoundary ? parentUuid : undefined,
    isSidechain, agentId, agentName (from teamInfo),
    promptId (only when user), …message,
    userType : getUserType(),              // 强制覆盖:SDK resume / fork 的 SerializedMessage 不带
    entrypoint : getEntrypoint(),
    cwd, sessionId, timestamp, version, gitBranch, slug
}
```

**`sessionId` 必须在 spread 之后**(注释 1048-1058):这是 `--fork-session` 期间的救命细节——如果保留源 sid,会让 content-replacement 记录的 sid mismatch 查找失败,模型判定 FROZEN。

**`sourceToolAssistantUUID` 覆盖 parentUuid**(1033-1039):`tool_result` 消息以创建时记录的 assistant UUID 作为 parent,而不是顺序 parent,从而保证每个 TR 跟自己的 tool_use 同 source(message.id)。

**末尾缓存 `currentSessionLastPrompt`**(1078-1083):取第一个非 sidechain 消息的 user text,过滤换行后截断 200 字符,写入 cache 供 `reAppendSessionMetadata` 输出。

### 4.8 Tombstone(`removeMessageByUuid`)

(873-953)

两条路径:

```
fast path (target in last LITE_READ_BUF_SIZE bytes):
    open file, fstat, read tail
    byte-search for `"uuid":"<target>"`
    locate lineStart / lineEnd via 0x0a
    fh.truncate(absLineStart)
    if afterLen > 0: fh.write(tail, lineEnd, afterLen, absLineStart)   // 后段往左挪

slow path (target beyond tail or huge file):
    if fileSize > MAX_TOMBSTONE_REWRITE_BYTES:
        logDebug + return                        // 避免 OOM
    else:
        readFile whole file, split, filter entry.uuid===target
        writeFile back
```

`afterLen === 0`(目标就是最后一行)退化成单次 ftruncate,这是 stop-the-world 路径里最快的部分。慢路径只有在大文件才走 — 50MB 阈值是为了和"GB 级 session 文件"妥协。

### 4.9 远程持久化

```
setRemoteIngressUrl(url)              (1349):登记 v1 ingress URL,同时把 FLUSH 切到 10ms
setInternalEventWriter(writer)        (1358):登记 v2 CCR 内部事件 writer
setInternalEventReader / Subagent Reader (1367/1374):登记 resume 用的回拉 reader

persistToRemote(sessionId, entry)     (1306):
    if isShuttingDown → return
    if internalEventWriter exists → 走 v2 writer
    elif !ENABLE_SESSION_PERSISTENCE || !remoteIngressUrl → return
    else sessionIngress.appendSessionLog(sessionId, entry, url)
        失败 → logEvent tengu_session_persistence_failed + gracefulShutdownSync(1)
```

- v2 与 v1 互斥:**先注册 v2 就完全不走 v1**(v1 仅在 internalEventWriter 不存在时回退)。
- sessionIngress.appendSessionLog 内部对每个 UUID 一次 POST,失败重试若干次后返回 false(详细在 `services/api/sessionIngress.ts`)。
- `appendEntry` 在 `if (isTranscriptMessage)` 时调 `persistToRemote`,即 metadata 条目(summary / title / tag / worktree-state 等)**不**走远端,仅本地。

### 4.10 Hydrate(反向导入)

- `hydrateRemoteSession(sid, ingressUrl)`(1591-1626):`switchSession` → mkdir `<projectDir>` → 用 `sessionIngress.getSessionLogs(...)` 拉所有 entries → 一次性 `writeFile(<sid>.jsonl, content)`。清空转录,只靠 ingress;`writeFile` 截断式写入,空远端等于空本地文件。
- `hydrateFromCCRv2InternalEvents(sid)`(1636-1727):CCR v2 走 `internalEventReader` + `internalSubagentEventReader`,按 agentId 分桶写到对应 `agent-<id>.jsonl`;main transcript 仍写到 `<sid>.jsonl`。异常若是 `'CCRClient: Epoch mismatch (409)'` 直接 re-throw(让 worker 退出,避免 epoch race)。

---

## 5. 加载管道:`loadTranscriptFile` 与后续处理

**这是文件最重的部分**。`loadTranscriptFile` 是个有状态流水线,从 `parseJSONL`、跳过 precompact 区间、剔除无效 fork 分支、再到 metadata 与 leaf 计算共 7 个阶段。

### 5.1 阶段 1 — `readHeadAndTail` 提供 64KB 头尾读(由 `sessionStoragePortable.ts` 实现)

`readLiteMetadata` 调用后:

- **head** 提取 `isSidechain / projectPath(cwd) / teamName / agentSetting / firstPrompt / customTitle / aiTitle`(行 4755-4769);
- **tail** 提取 `lastPrompt / customTitle / summary / tag / gitBranch / prNumber / prUrl / prRepository`(行 4775-4803)。

仅在 `getSessionFilesLite + enrichLog` 路径上用到;resume 的精确路径绕开此步、读全文后做结构化 parse。

### 5.2 阶段 2 — env-gated precompact skip

```
if !isEnvTruthy(CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP):
    stat(filePath).size
    if size > SKIP_PRECOMPACT_THRESHOLD:
        const scan = readTranscriptForLoad(filePath, size)   // portable
        buf = scan.postBoundaryBuf
        hasPreservedSegment = scan.hasPreservedSegment
        if scan.boundaryStartOffset > 0:
            metadataLines = await scanPreBoundaryMetadata(filePath, scan.boundaryStartOffset)
```

跳过 precompact 区间有两个不变量:

- `readTranscriptForLoad`(portable) 找到最后一条 compact boundary,**跳过 boundary 之前的所有字节**;但 metadata 仍要恢复(因为 mode / agent-setting / tag / pr-link 是在 boundary 之前的某个尾部写入的)。
- `scanPreBoundaryMetadata`(3163-3230)用 byte-level marker 匹配(METADATA_TYPE_MARKERS),绝大部分 chunk 不含任何 marker → 跳过 line split;有 marker 才切到边界、收集 line 范围。
- 边界守卫:`if (carry.length > 64 * 1024) carry = null` 防止病态大行把 carry 挤爆。

### 5.3 阶段 3 — `walkChainBeforeParse` 提前剔除无效 fork

```
if !keepAllLeaves && !hasPreservedSegment && !DISABLE_PRECOMPACT_SKIP && buf.length > PRECOMPACT_THRESHOLD:
    buf = walkChainBeforeParse(buf)
```

`walkChainBeforeParse` 是一个手写 NDJSON 解析器(3312-3472),它:

1. 对每行判断是否形如 `{"parentUuid":...`;若是,把行号、UUID 偏移、parent 偏移记到 stride-3 数组;
2. 用 PARENT_PREFIX / `"uuid":"` / `","timestamp":"` 这些固定 key 字节序,做 0 个 string 解码的字节级判定;
3. 用 SIDECHAIN_TRUE 反向扫,定位 leaf slot;
4. 用 Map<uuid, slot> 反向 walk parentUuid 直到 root;
5. **只在能省至少 50% 字节时**才拼接裁剪后的 buffer(注释 3440-3449 的实测 gate);
6. 最终用 Buffer.concat subarray views merge。

**两个不变量**(注释 3247-3265):

1. `recordTranscript` 把 `parentUuid` 作为 serialize 时第一个 key(JSON.stringify 保序);因此 `{"parentUuid":` 是稳定的 transcript 行前缀。
2. 顶层 uuid 检测靠 `uuid_key + 36 字符 + ","timestamp":"` 后缀;有多个候选时靠 `pickDepthOneUuidCandidate` 做 brace-depth scan 选取 depth=1 的那个。

性能实测(注释 3241-3243):41MB / 99% dead branch 上,parseJSONL 时间从 56ms → 3.9ms。

### 5.4 阶段 4 — parseJSONL 与 per-type routing

```
const entries = parseJSONL<Entry>(buf)

const progressBridge = new Map<UUID, UUID | null>()   // 3641,legacy progress 桥接

for (const entry of entries) {
    if isLegacyProgressEntry(entry):
        // chain-resolve through consecutive progress
        progressBridge.set(uuid, parent ? progressBridge.get(parent) ?? parent : parent)
        continue
    if isTranscriptMessage(entry):
        if (entry.parentUuid && progressBridge.has(entry.parentUuid)):
            entry.parentUuid = progressBridge.get(entry.parentUuid) ?? null
        messages.set(uuid, entry)
        if isCompactBoundaryMessage(entry):
            contextCollapseCommits.length = 0     // stale 的 commits / snapshot 砍掉
            contextCollapseSnapshot = undefined
    else if entry.type === 'custom-title' && entry.sessionId:
        customTitles.set(sessionId, customTitle)
    else if entry.type === 'tag' && entry.sessionId:
        tags.set(sessionId, tag)
    ...(其他 metadata type 类似走 session-scoped Map)

if metadataLines (from precompact scan):
    parse + 同上路由 -- 用来恢复 precompact 之前但非 stale 的 metadata
}
```

**Legacy progress bridge** 是 PR #24099 的反向兼容:旧 transcript 仍然包含 `type: 'progress'` 的条目,它们已经不在 Entry union,但仍以 `uuid` 与 `parentUuid` 存在。如果不桥接,后续 message 的 `parentUuid` 直接指向 progress,`buildConversationChain` 在 `messages.get(progress_uuid)` 时返回 `undefined`,链截断。桥接办法是 chain-resolve:把"连续 progress 段"折叠成一个 bridge,后面真正 message 进来时若它的 parent 在 bridge 里,直接重写到 bridge 解析后的真实祖先。

**Compact boundary 中段清场**:遇到 boundary 时清空 `contextCollapseCommits` 数组 + 重置 snapshot。这与 `applyPreservedSegmentRelinks`(5.5)合作——precompact 区间已被跳过,若 boundary 之后没有 seg-preservedUuids,剩下 stale commit 自然不应再保留。

### 5.5 阶段 5 — 后处理:Preserved Segment 与 Snip

```
applyPreservedSegmentRelinks(messages)   (1843-1960)
applySnipRemovals(messages)              (1986-2043)
```

`applyPreservedSegmentRelinks`:

- 找**最后一个** compact boundary 与最后一个 seg-boundary(同 boundary / 不同 boundary);
- 验证 seg-tail → seg-head 在 messages map 内能走通(`tengu_relink_walk_broken` 上报);
- tail splice:anchor 的其它子节点 → seg.tail;
- head splice:seg.head.parentUuid = anchor;
- 修剪所有 precompact 区间但不在 preservedUuids 的消息。

`applySnipRemovals`:

- 找出所有 `snipMetadata.removedUuids`;
- 删除这些 UUID;
- **还必须对幸存者做 parent relink**:because transcripts are append-only,survivors 的 parentUuid 可能指向被删段。`resolve(start)` 在 `deletedParent` map 上向后走直到非删除祖先,然后 path-compress 写回 map;
- `tengu_snip_resume_filtered` 上报 `(removed_count, relinked_count)` 用于回归监视。

### 5.6 阶段 6 — Leaf UUID 计算

```
const leafUuids = new Set<UUID>()
const parentUuids = new Set(allMessages.map(m=>m.parentUuid).filter(non-null))
const terminalMessages = allMessages.filter(m => !parentUuids.has(m.uuid))

if (feature('tengu_pebble_leaf_prune'):
    // prune leaves whose ancestry has user/assistant children
else:
    // 原始策略:每个 terminal 回溯到最近 user/assistant 祖先,加入 leafUuids
```

`leafUuid` 是 `--resume` / `getLastSessionLog` / `loadAllLogsFromSessionFile` 决定"哪些 chain 算会话的尾巴"的依据。

### 5.7 阶段 7 — 解析后导出元数据表

返回 `messages / summaries / customTitles / tags / agentNames / agentColors / agentSettings / prNumbers / prUrls / prRepositories / modes / worktreeStates / fileHistorySnapshots / attributionSnapshots / contentReplacements / agentContentReplacements / contextCollapseCommits / contextCollapseSnapshot / leafUuids`。

注意 `contentReplacements` 与 `agentContentReplacements` 是分开的 Map:前者按 sessionId(主链 `/resume`),后者按 agentId(sidechain / AgentTool resume)。两路都在 stage 4 的 entry type 分发处写入。

### 5.8 消费阶段

| 函数 | 输入 | 输出 |
| --- | --- | --- |
| `loadTranscriptFromFile(file)` | 绝对路径(.jsonl 或 .json) | `LogOption` 单条 |
| `loadFullLog(log)` | lite log | full log(`isLite===true` 且 `messages.length===0` 时触发) |
| `getLastSessionLog(sid)` | sid | `LogOption | null`(用 leaf-walk 找到 last non-sidechain message + `buildConversationChain`) |
| `getAgentTranscript(agentId)` | agentId | 该 agent 的 sidechain messages + contentReplacements |
| `loadAllLogsFromSessionFile(file, projectPath?)` | 文件路径 | 每条 leaf 一个 LogOption(给 `/insights` / 离线分析) |

**`getLastSessionLog` 顺便预热 `getSessionMessages` cache**(注释 3893-3903):避免 REPL mount 后 `recordTranscript` 再触发一次全文件读。guard `if (!cache.has(sid))` 防止 mid-session 反复 reset 掉未落盘 UUID。

---

## 6. List-loaders(给 `/resume` picker 用)

按调用顺序自上而下:

```
fetchLogs(limit)
  ├── getSessionFilesWithMtime(projectDir)        // 并行 stat
  ├── getSessionFilesLite(projectDir, limit, cwd) // 纯 filesystem metadata
  ├── trackSessionBranchingAnalytics(logs)        // 上报 tengu_session_forked_branches_fetched
  └── return logs

loadMessageLogs(limit)
  ├── fetchLogs(limit)
  └── enrichLogs(0, INITIAL_ENRICH_COUNT=50)      // 渐进式增强前 50 条
      └── enrichLog(lite)
          ├── readLiteMetadata(file, size, buf)    // 头 64KB + 尾 64KB byte-level extract
          │   ├── extractLastJsonStringField(tail, 'customTitle') ?? …
          │   └── extractFirstPromptFromChunk(head) (复杂 fallback)
          └── 过滤:sidechain / agent / 'teamName 团队' 跳过

loadAllProjectsMessageLogs(limit, opts)
  ├── skipIndex=true → loadAllProjectsMessageLogsFull (全解析,给 /insights)
  └── 否则 loadAllProjectsMessageLogsProgressive
        └── enrollProjects / statAll / enrichLogs 分桶去重

loadSameRepoMessageLogs(worktreePaths, limit)
  → loadSameRepoMessageLogsProgressive
      └── getStatOnlyLogsForWorktrees(paths):
          paths.length ≤ 1 → 直接 getSessionFilesLite(cwd)
          否则遍历 ~/.claude/projects/* 找最长前缀匹配的子目录,stat 后 dedupe by sessionId
```

**`searchSessionsByCustomTitle(query, {exact?})`**(3071-3112):

- worktree-aware 扫描(`getWorktreePaths(getOriginalCwd())`);
- 一次性 `enrichLogs(0, all)` 让所有候选都拿到 customTitle;
- title 大小写不敏感比较(`exact` 则 `===` 否则 `.includes`);
- 去重(同一 sessionId 保留 `modified` 最新);
- 按时间倒序。

**性能要点**:

- `INITIAL_ENRICH_COUNT = 50`(4583):UI 首屏只需要前 50 条,渐进式继续。~6.4MB 头部 + 尾部 I/O,对多数会话足够。
- `readLiteMetadata` 使用 `extractJsonStringFieldPrefix` 而不是 `JSON.parse`:即使首行被 64KB 截断,也能拿到 truncated 内容。
- `enrichLog` 跳过 sidechain、agent 团队会话(行 5061-5073),它们不该出现在 `/resume`。
- Lite logs 之间对相同 sessionId 通过 `deduplicateLogsBySessionId` 取最新 `modified` 的副本。

---

## 7. 写入辅助函数

| 函数 | 关键行为 |
| --- | --- |
| `saveCustomTitle(sid, title, fullPath?, source='user')` | 直接 `appendEntryToFile` `custom-title` 行;若 `sid === getSessionId()` 则还把 `currentSessionTitle` 写到 cache,触发 `tengu_session_renamed` 上报 |
| `saveAiGeneratedTitle(sid, aiTitle)` | 写 `ai-title` 行,**不**进 `currentSessionTitle` cache、不再 re-append;**不**触发 session_renamed(注释 2646-2672)。这样 user 重命名在 mid-session 不会被 stale AI title 覆盖 |
| `saveTaskSummary(sid, summary)` | 滚动快照(`claude ps` 读尾条最新);同样不 re-append |
| `saveTag(sid, tag, fullPath?)` | 与 customTitle 对称 |
| `linkSessionToPR(sid, n, url, repo, fullPath?)` | 同时写 `pr-link` 行 + 更新 `currentSessionPr*` cache(供 `reAppendSessionMetadata` 在 compaction/exit 后保持尾部对齐) |
| `saveAgentName / saveAgentColor` | 直接写 + 更新 cache + 触发 tengu_agent_* |
| `saveAgentSetting(agent)` | **cache-only** —— 见 `materializeSessionFile` 的注释(2863-2868) |
| `cacheSessionTitle(title)` | **cache-only** —— `--name` 路径 |
| `saveMode(mode)` | **cache-only** —— 同上 |
| `saveWorktreeState(s)` | strip 掉 `creationDurationMs / usedSparsePaths` 后(2899-2913)再写;同时 cache-only 写在 sessionFile 已存在时同步追加一行 |

设计原则:"只要数据进 cache,exit handler 会负责把它写出去"——所以业务代码大多只是 cache → 写到 disk 不一定立即发生;真要立即写盘时(如 compaction 后)走 `reAppendSessionMetadata()`。

---

## 8. 清理、转换、Pre-write

| 函数 | 位置 | 用途 |
| --- | --- | --- |
| `removeExtraFields(transcript)` | 1818 | 反序列化前剥离 `parentUuid / isSidechain` 字段,产出 `SerializedMessage[]`(只保留 UI 用的字段) |
| `cleanMessagesForLogging(messages, all)` | 4456 | 写盘前清洗:`isLoggableMessage` 过滤 `progress` 等;外部 build 还走 `transformMessagesForExternalTranscript` 把 REPL tool_use 包成 native 工具调用 |
| `isLoggableMessage(m)` | 4357 | `progress` 一律不写;外部 build 的 `attachment` 默认不写(除非 `hook_additional_context` + `CLAUDE_CODE_SAVE_HOOK_ADDITIONAL_CONTEXT`) |
| `transformMessagesForExternalTranscript` | 4402 | external users:把 `(assistant tool_use=REPL_TOOL_NAME, user tool_result=its_id)` 整对删去,`isVirtual` 提级成 normal 消息 |
| `collectReplIds(all)` | 4375 | **必须从全集**收集 REPL tool_use_id,因为异步切片中 tool_use 与 tool_result 落在不同 `recordTranscript` 调用 |
| `applyPreservedSegmentRelinks / applySnipRemovals` | (5.5) | 边界 / snip 的链修补 |

---

## 9. 与 resume / fork / 其它需求的接口契约

### 9.1 `--resume <sid>` 流程用到本文件的函数

```
main.tsx resume branch:
    loadConversationForResume(sid, undefined)
        └─ getLastSessionLog(sid)
            ├─ loadSessionFile(sid)               // 含 sidebar maps
            ├─   ↓ prewarm getSessionMessages.cache
            ├─ buildConversationChain
            └─ buildFileHistorySnapshotChain + attributionSnapshots + contentReplacements
    processResumedConversation(...)
        ├─ restoreSessionMetadata(...)
        ├─ adoptResumedSessionFile()             // tail re-append with skipTitleRefresh=true
        └─ (后续 state machine in sessionRestore.ts)
    launchRepl(... initialMessages ...)

REPL on-mount:
    restoreReadFileState(initialMessages, getOriginalCwd())
    useLogMessages(messages, length===initial.length)
        └─ recordTranscript → getSessionMessages(sid)   ← cache 预热后只查 delta
```

### 9.2 `--continue` 与 `--fork-session`

- `--continue`:`loadConversationForResume(undefined)` —— 与 `--resume <sid>` 一致的 `getLastSessionLog` 路径,只是 source 不传 sid 让 `loadMessageLogs` 挑最近的非 live-session。
- `--fork-session`:复用 `loadConversationForResume` 拿到 messages,但 `adoptResumedSessionFile` **不调**;`recordTranscript` 在新 sessionId 的新文件上从这次 messages 末尾继续写。同时 `recordContentReplacement` 必须先跑一遍以预填新 session 的 content-replacement 缓存(注释 sessionRestore.ts:452-462)。

### 9.3 远端 / CCR(v1 / v2)

- v1:`ENABLE_SESSION_ESSION_PERSISTENCE=1` + `remoteIngressUrl`;`persistToRemote` 走 `sessionIngress.appendSessionLog`;`hydrateRemoteSession` 反向 sync。
- v2:`setInternalEventWriter`/`Reader` 由 daemon 注入;不再走 v1；`FLUSH_INTERVAL_MS` 切到 10ms。`hydrateFromCCRv2InternalEvents` 对 `subagent` 也有 reader。

### 9.4 `--no-session-persistence` / 测试 / cleanup

- `shouldSkipPersistence`(962):`NODE_ENV === 'test'`(除非 `TEST_ENABLE_SESSION_PERSISTENCE=1`)、`cleanupPeriodDays === 0`、`isSessionPersistenceDisabled()`、`CLAUDE_CODE_SKIP_PROMPT_HISTORY=1` 任一为真都跳过;
- `materializeSessionFile` 同样过这个 guard,避免 metadata-only session 孤儿文件;
- `reAppendSessionMetadata` 走的是 `appendEntryToFile`(直接 fs appendFileSync),**不**经过 `appendEntry`,所以这个 guard 对它无效——这就是为什么 `materializeSessionFile` 内还必须独立检查一次(978-993)。

### 9.5 Snip / Force-snip / HISTORY_SNIP 门控

- `applySnipRemovals` 在 load 阶段隐形生效,**不需要** `recordTranscript` 介入;
- `EPHEMERAL_PROGRESS_TYPES`(`bash_progress / powershell_progress / mcp_progress[/ sleep_progress]` 门控):`isEphemeralToolProgress` 用作"UI-only 进度,落盘时跳过"。
- `isChainParticipant` 排除 `progress`,所以即使 progress 被 append 到 disk(legacy transcript),parent 链 walk 时也跳过它(被 `progressBridge` 桥接后再参与)。

---

## 10. 性能、内存、容错要点

| 维度 | 处理 |
| --- | --- |
| 大文件(`>5MB` 默认、`>50MB` 兜底) | `walkChainBeforeParse` 主动剔除 fork-branch;`readTranscriptForLoad` 跳过 precompact 区间;attr-snapshot 在 fd 层就剥掉——三者一起保证"151MB 文件 / 84% stale"只分配 ~32MB |
| O(n) 步数 | `findLatestMessage`(2050,单次扫描)替代 `filter + sort + [0]`;`getSessionFilesWithMtime`(4532)用 `Promise.all` 并行 stat;`bulk write + flush timer` 替代每条独立 syscall |
| Tombstone 失败兜底 | tail path 用 byte-scan,慢路径在 ≥ 50MB 时直接放弃(959 行注释:避免 OOM) |
| Pending write 计数 | `trackWrite` + `pendingWriteCount` + `flushResolvers` —— 让 `flush()` 能正确等待"非队列"操作(`removeMessageByUuid`)完成 |
| 父子消歧 | `pickDepthOneUuidCandidate` 解决 nested Message (`agent_progress`) 与 `mcpMeta` 可能撞 `","timestamp":"` 后缀的情况 |
| 桥接 | `progressBridge` 用 chain-resolve 折叠连续 progress entry,O(n) 一次扫描即可 |
| Round-trip 健康监测 | `checkResumeConsistency`(2228)对 `turn_duration` checkpoint 上报 `tengu_resume_consistency_delta`,监测 resume 重建链与写入时不一致 |
| Snip round-trip 健康 | `applySnipRemovals` 末段 `tengu_snip_resume_filtered` 上报 `(removed, relinked)`,回归 snip 删除 |

---

## 11. 与其它模块的接口

| 来源 | 用途 |
| --- | --- |
| `services/api/sessionIngress.ts` | v1 session ingress 上行 / 下行 |
| `bootstrap/state.ts` | `getSessionId / switchSession / getSessionProjectDir / isSessionPersistenceDisabled / getPromptId / getPlanSlugCache` |
| `utils/cleanupRegistry.ts` | `registerCleanup(flush + reAppend)` —— 进程退出兜底 |
| `utils/slowOperations.ts` | `jsonParse / jsonStringify`(避免 hot path JSON 解析的 UI 卡顿) |
| `utils/sessionStoragePortable.ts` | `readHeadAndTail / readTranscriptForLoad / extractJsonStringField / SKIP_PRECOMPACT_THRESHOLD / LITE_READ_BUF_SIZE` |
| `commands.ts` | `builtInCommandNames` —— 给 `getFirstMeaningfulUserMessageTextContent` 区分 slash command |
| `commands/clear/conversation.ts`、`cli/print.ts` | `restoreSessionMetadata / resetSessionFilePointer` 的消费者 |
| `services/compact/compact.ts`、`reactiveCompact.ts` | 在 compact 边界写入前调 `reAppendSessionMetadata`,保持元数据贴近 EOF |
| `services/contextCollapse/persist.js` | CONTEXT_COLLAPSE 门控恢复(`restoreFromEntries`) + `recordContextCollapse{Commit,Snapshot}` 双向 |
| `concurrentSessions.js` | `updateSessionName` —— `saveAgentName` 触发 |
| `getWorktreePaths` | `loadSameRepoMessageLogs(Progressive)` 用以做 worktree 范围检索 |
| `formattedFileSize` | tombstone slow-path 上报 |

---

## 12. 一图总览(从 worker 写到从 worker 读)

```
worker.call                  worker.boot
    │                            │
    │ insertMessageChain         │ getTranscriptPath + materialize (lazy)
    │ recordTranscript           │
    │ recordSidechainTranscript  │ loadTranscriptFile
    │ recordFileHistorySnapshot  │   ├─ readHeadAndTail (lite)
    │ recordAttributionSnapshot  │   ├─ walkChainBeforeParse
    │ recordContentReplacement   │   ├─ parseJSONL + type dispatch
    │ recordContextCollapseCommit│   ├─ applyPreservedSegmentRelinks
    │ recordContextCollapseSnap… │   ├─ applySnipRemovals
    │ recordQueueOperation       │   └─ leaf UUID set
    │ removeTranscriptMessage    │
    ▼                            ▼
Project.appendEntry       loadTranscriptFile
    ├─ enqueueWrite(per-file queue, 100/10ms timer)
    ├─ trackWrite(pend counter)
    └─ persistToRemote (v1 ingress | v2 internal events)
                          │
                          ▼
                  ~/.claude/projects/<cwdHash>/<sid>.jsonl
                  ~/.claude/projects/<cwdHash>/<sid>/subagents/agent-<agentId>.jsonl
                  ~/.claude/projects/<cwdHash>/<sid>/remote-agents/remote-agent-<taskId>.meta.json
                  ~/.claude/projects/<cwdHash>/<sid>/<agentId>.meta.json       (writeAgentMetadata)
                  ~/.claude/sessions/<sid>/recording.cast
                  ~/.claude/file-history/<sid>/<backupFileName>
                  ~/.claude/plans/<slug>.md
                          │
                          ▼
                  loadConversationForResume → processResumedConversation → launchRepl
                  loadMessageLogs → resume picker UI
```

---

## 13. 一句话总结

> **`sessionStorage.ts` 是 Claude Code 的**"transcript 单一可信存储层"**——既管把每次 API 响应、metadata 变更、compaction、context-collapse、prompt 摘要、file-history snapshot、attribution 与 content-replacement 一行行有序落盘(走 Project 单例 + 100/10ms flush timer + per-file queue),又把磁盘上的 NDJSON 反向解码为 resume picker / REPL 用的 `LogOption` 与 `Message[]`(走多阶段 parser:`walkChainBeforeParse` 剔除死 fork、`progressBridge` 兼容旧 transcript、`applyPreservedSegmentRelinks / applySnipRemovals` 修补 compaction / snip 后的链、`leafUuid` 锚定 chain 尾),同时把当前会话的"缓存视图"(title / tag / agent / mode / worktree / pr / lastPrompt)在 `reAppendSessionMetadata` 框架下与 EOF 严格对齐,确保 SDK、resume、远端 ingress 三条入口都能拿到一致的视图。**

---

## 14. 关键代码锚点

```
src/utils/sessionStorage.ts:199          getProjectsDir
src/utils/sessionStorage.ts:203          getTranscriptPath
src/utils/sessionStorage.ts:208          getTranscriptPathForSession
src/utils/sessionStorage.ts:248          getAgentTranscriptPath / subdirs
src/utils/sessionStorage.ts:533          class Project
src/utils/sessionStorage.ts:608          enqueueWrite (per-file queue)
src/utils/sessionStorage.ts:620          scheduleDrain (100/10ms timer)
src/utils/sessionStorage.ts:647          drainWriteQueue (chunk flush)
src/utils/sessionStorage.ts:690          resetSessionFile (Project 私有)
src/utils/sessionStorage.ts:723          reAppendSessionMetadata (tail-aware)
src/utils/sessionStorage.ts:843          flush (cancel timer + wait drain)
src/utils/sessionStorage.ts:873          removeMessageByUuid (fast + slow path)
src/utils/sessionStorage.ts:962          shouldSkipPersistence
src/utils/sessionStorage.ts:978          materializeSessionFile (lazy create)
src/utils/sessionStorage.ts:995          insertMessageChain (parent walk + lastPrompt)
src/utils/sessionStorage.ts:1088         insertFileHistorySnapshot
src/utils/sessionStorage.ts:1116         insertContentReplacement (sessionId vs agentId 双路)
src/utils/sessionStorage.ts:1131         appendEntry (主 dispatch:dedup + sidechain bypass)
src/utils/sessionStorage.ts:1306         persistToRemote (v1 / v2)
src/utils/sessionStorage.ts:1349         setRemoteIngressUrl / setInternalEventWriter
src/utils/sessionStorage.ts:1412         recordTranscript (dedup-aware batch)
src/utils/sessionStorage.ts:1509         resetSessionFilePointer
src/utils/sessionStorage.ts:1534         adoptResumedSessionFile (with skipTitleRefresh)
src/utils/sessionStorage.ts:1545         recordContextCollapseCommit
src/utils/sessionStorage.ts:1567         recordContextCollapseSnapshot
src/utils/sessionStorage.ts:1591         hydrateRemoteSession (v1 ingress)
src/utils/sessionStorage.ts:1636         hydrateFromCCRv2InternalEvents
src/utils/sessionStorage.ts:1729         extractFirstPrompt
src/utils/sessionStorage.ts:1750         getFirstMeaningfulUserMessageTextContent
src/utils/sessionStorage.ts:1818         removeExtraFields
src/utils/sessionStorage.ts:1843         applyPreservedSegmentRelinks
src/utils/sessionStorage.ts:1986         applySnipRemovals
src/utils/sessionStorage.ts:2050         findLatestMessage (O(n) single pass)
src/utils/sessionStorage.ts:2073         buildConversationChain (parent walk)
src/utils/sessionStorage.ts:2122         recoverOrphanedParallelToolResults
src/utils/sessionStorage.ts:2228         checkResumeConsistency (telemetry)
src/utils/sessionStorage.ts:2298         loadTranscriptFromFile (.jsonl / .json)
src/utils/sessionStorage.ts:2483         convertToLogOption
src/utils/sessionStorage.ts:2577         appendEntryToFile (sync fs.appendFileSync)
src/utils/sessionStorage.ts:2597         readFileTailSync (sync fd scan)
src/utils/sessionStorage.ts:2623         saveCustomTitle
src/utils/sessionStorage.ts:2673         saveAiGeneratedTitle (不 cache、不 re-append)
src/utils/sessionStorage.ts:2687         saveTaskSummary
src/utils/sessionStorage.ts:2696         saveTag
src/utils/sessionStorage.ts:2711         linkSessionToPR
src/utils/sessionStorage.ts:2764         restoreSessionMetadata
src/utils/sessionStorage.ts:2798         clearSessionMetadata
src/utils/sessionStorage.ts:2825         saveAgentName / saveAgentColor
src/utils/sessionStorage.ts:2867         saveAgentSetting (cache-only)
src/utils/sessionStorage.ts:2876         cacheSessionTitle (cache-only)
src/utils/sessionStorage.ts:2885         saveMode (cache-only)
src/utils/sessionStorage.ts:2895         saveWorktreeState
src/utils/sessionStorage.ts:2933         getSessionIdFromLog
src/utils/sessionStorage.ts:2946         isLiteLog
src/utils/sessionStorage.ts:2955         loadFullLog (lite→full)
src/utils/sessionStorage.ts:3071         searchSessionsByCustomTitle
src/utils/sessionStorage.ts:3163         scanPreBoundaryMetadata (byte marker)
src/utils/sessionStorage.ts:3312         walkChainBeforeParse (fork cull)
src/utils/sessionStorage.ts:3478         loadTranscriptFile (主解析器)
src/utils/sessionStorage.ts:3824         loadSessionFile (单文件 wrapper)
src/utils/sessionStorage.ts:3848         getSessionMessages (memoized)
src/utils/sessionStorage.ts:3875         getLastSessionLog
src/utils/sessionStorage.ts:3945         loadMessageLogs
src/utils/sessionStorage.ts:3969         loadAllProjectsMessageLogs
src/utils/sessionStorage.ts:3984         loadAllProjectsMessageLogsFull
src/utils/sessionStorage.ts:4024         loadAllProjectsMessageLogsProgressive
src/utils/sessionStorage.ts:4079         loadSameRepoMessageLogs
src/utils/sessionStorage.ts:4092         loadSameRepoMessageLogsProgressive
src/utils/sessionStorage.ts:4119         getStatOnlyLogsForWorktrees
src/utils/sessionStorage.ts:4196         getAgentTranscript
src/utils/sessionStorage.ts:4250         extractAgentIdsFromMessages
src/utils/sessionStorage.ts:4277         extractTeammateTranscriptsFromTasks
src/utils/sessionStorage.ts:4303         loadSubagentTranscripts
src/utils/sessionStorage.ts:4331         loadAllSubagentTranscriptsFromDisk
src/utils/sessionStorage.ts:4357         isLoggableMessage
src/utils/sessionStorage.ts:4375         collectReplIds (全集 REPL tool_use_id)
src/utils/sessionStorage.ts:4402         transformMessagesForExternalTranscript
src/utils/sessionStorage.ts:4456         cleanMessagesForLogging
src/utils/sessionStorage.ts:4484         findUnresolvedToolUse
src/utils/sessionStorage.ts:4532         getSessionFilesWithMtime (Promise.all stat)
src/utils/sessionStorage.ts:4583         INITIAL_ENRICH_COUNT = 50
src/utils/sessionStorage.ts:4745         readLiteMetadata (head + tail)
src/utils/sessionStorage.ts:4824         extractFirstPromptFromChunk
src/utils/sessionStorage.ts:4927         extractJsonStringFieldPrefix (truncated-safe)
src/utils/sessionStorage.ts:4961         deduplicateLogsBySessionId
src/utils/sessionStorage.ts:4981         getSessionFilesLite (stat only)
src/utils/sessionStorage.ts:5029         enrichLog (lite → full lite)
src/utils/sessionStorage.ts:5083         enrichLogs (progressive)
```
