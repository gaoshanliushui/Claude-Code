# `claude --resume` 的难点与克服

> 本文不谈"做对了什么",只谈"做错很容易、正确很难、且必须在反编译版本里依然如此"的部分。
> 每条**难点**都对应源码里可被验证的具体代码,**克服方案**给出文件:行号以及为什么这是唯一安全(或较安全)的选择。
>
> 范围:覆盖 `src/utils/sessionStorage.ts`、`src/utils/conversationRecovery.ts`、`src/utils/sessionRestore.ts`、`src/main.tsx`、`src/screens/REPL.tsx`、`src/utils/plans.ts`、`src/utils/fileHistory.ts`、`src/utils/toolResultStorage.ts`、`src/commands/clear/conversation.ts` 等关键路径。其它细节文档参考:
> - [`claude-resume-flow.md`](./claude-resume-flow.md) —— 全流程
> - [`claude-resume-restored-data.md`](./claude-resume-restored-data.md) —— 26 项恢复数据
> - [`session-storage-design.md`](./session-storage-design.md) —— `sessionStorage.ts` 内部机制

---

## 0. 难点的总览(7 大类,共 25 个具体问题)

| 类别 | 涉及难点 |
| --- | --- |
| **A. 反向重建 chain 的正确性** | A1 多 fork 分支;A2 cycle;A3 dangling parent;A4 streaming 切分导致的 sibling 丢失;A5 legacy progress;A6 nested uuid 歧义 |
| **B. append-only 文件的回放** | B1 snip 中段删除后的 parent 悬挂;B2 compact 后的 preserved segment 拼接;B3 tombstone 与 50MB OOM 兜底 |
| **C. 性能与延迟** | C1 大文件 precompact skip;C2 fork 分支预剔除;C3 lite 头尾读;C4 progressive enrichment |
| **D. 元数据 EOF 对齐** | D1 custom-title 被大消息挤出 64KB tail;D2 AI title 不得 re-append;D3 cache-only 元数据落盘时机 |
| **E. 进程级状态机** | E1 switchSession 与 sessionFile 指针;E2 restoreSessionMetadata 顺序约束;E3 --name 与 disk title 冲突;D4 fork 不能继承 worktreeSession |
| **F. 多方写入协调** | F1 sidechain UUID 不能入主 messageSet;F2 remote 写入 epoch mismatch;F3 fork + content-replacement 预填 |
| **G. UX / 兼容性兜底** | G1 lite→full 升级失败兜底;G2 Resume 取消 / 错误显示;G3 bash-mode 与 slash command 标题;G4 win32 worktree 大小写;G5 --no-session-persistence |

下面逐条展开。

---

## A. 反向重建 chain 的正确性

### A1. 一个 session 文件可能包含多个 leaf —— fork / branch

**难点**

JSONL 是 append-only,每次 fork / rewind / `/clear` 都会留下"孤儿分支"。`buildConversationChain` 从某个 leaf 反向走 `parentUuid`,但**怎么知道 leaf 是哪一个**?

- 用户主动 fork 出新 chain(显式写入新消息),原 leaf 仍存在;
- `Ctrl-Z`/rewind 留一段旧链路,新链路写入时旧链路成为孤儿;
- `/clear` 在文件里直接插入 system message,而不是物理截断,旧 leaf 仍可定位。

**解决**

`loadTranscriptFile` 在 stage 6(3744-3796)显式计算 leaf 集合:

```
parentUuids = new Set(allMessages.map(m => m.parentUuid).filter(non-null))
terminalMessages = allMessages.filter(m => !parentUuids.has(m.uuid))   // 没有子节点 = leaf
```

从每个 terminal 回溯到最近的 user/assistant 祖先,把 UUID 加入 `leafUuids`。这一集合就是该文件所有"用户视角的对话尾"。

**取舍**:对 `/insights` 等需要全分支分析的场景,`loadAllLogsFromSessionFile` 调 `loadTranscriptFile(..., {keepAllLeaves: true})`,并跳过 `walkChainBeforeParse`(3580-3585)—— 后者会主动剔除死 fork 分支。

门控 `tengu_pebble_leaf_prune`(3737)在标准 leaf 之上增加启发:若 terminal 自身不是 user/assistant,先向上回溯找最近的 user/assistant 祖先,并且只有当祖先"没有 user/assistant 子节点"时才认它是 leaf;否则该祖先其实是 mid-conversation 节点,真实 leaf 是它之后某处(progress child 与 tool_result child 共存时的歧义场景,注释 3750-3764)。

---

### A2. cycle —— parent 链可以成环

**难点**

理论上 JSONL 的 `parentUuid` 应该指向之前的 UUID,但用户主动写、fork 错误、并发 append 都可能形成回环。

**解决**

`buildConversationChain`(2080-2095):

```
while (currentMsg) {
    if (seen.has(currentMsg.uuid)) { logError("Cycle detected") + logEvent("tengu_chain_parent_cycle") + break }
    seen.add(currentMsg.uuid)
    transcript.push(currentMsg)
    currentMsg = currentMsg.parentUuid ? messages.get(currentMsg.parentUuid) : undefined
}
```

`seen` 是 `Set<UUID>`,环检测 O(1)。遇到环就**返回 partial transcript**(注释 2083-2084:"Returning partial transcript")—— 优于直接抛错把整个 resume 退回 picker。

stage 6 leaf 计算同样有 `hasCycle` 哨兵(3794)。

---

### A3. dangling parent —— chain 末尾指向不存在的 UUID

**难点**

正常 chain 在 root 处 `parentUuid = null` 终止。但 fork / snip 之后,某些消息的 `parentUuid` 指向已被物理删除的 UUID,`buildConversationChain` 在 `messages.get(undefined)` 处终止,**之前所有祖先都会丢失**。

**解决**

注释 3421-3422 明确承认:`A dangling parent (uuid not in file) is the normal termination for forked sessions and post-boundary chains -- same semantics as buildConversationChain`。

- 对 fork:返回的 transcript 只是一支 chain,UI 显示当前 leaf 的祖先,缺失的祖先在历史上不存在(在另一支),这是正确的;
- 对 snip / compact:`applySnipRemovals` / `applyPreservedSegmentRelinks` 显式把 dangling parent 修好(A4 与 B1)。

---

### A4. streaming 切分导致 sibling assistant / tool_result 丢失

**难点**

`claude.ts` 的 streaming(`~2024` 行)在 `content_block_stop` 时**每个 content block 单独 emit 一个 AssistantMessage**,于是 N 个并行 tool_use → N 条 message,**不同 uuid 但相同 `message.id`**(API 层的同一轮响应)。

`buildConversationChain` 是单链表 walk,沿 `parentUuid` 走只能保留一支,**丢掉所有 sibling assistant 与 sibling tool_result**。legacy progress(PR #23537 之前)还会在每个 tool_use asst 下生成一个 progress 子,TR 子链挂在另一支,walk 跟错。

**解决**

`recoverOrphanedParallelToolResults`(2122-2210):

1. `chainAssistants = chain.filter(type === 'assistant')` —— 拿到已经在链上的所有 assistant;
2. `anchorByMsgId` 用 chain 上"同一 message.id 的最后一个 assistant"作为锚点;
3. `siblingsByMsgId` 用 `message.id` 反向建索引,sibling assistant 全部找到;
4. `toolResultsByAsst` 用 `parentUuid` 反向建索引,每条 TR 挂回它的源 asst;
5. 对链上每个 assistant group,**timestamp 排序**(stable sort 保留 JSONL 写入顺序),把 off-chain sibling 与 off-chain TR **splice 进 anchor 之后**,保持相邻。

`logEvent("tengu_chain_parallel_tr_recovered", { recovered_count })` 上报实际抢救量,作为回归监视。

这是个**事后修补** —— 写入端无法修改 stream 行为(必须逐 block emit),只能在读端救。

---

### A5. legacy `progress` 条目不再属于 `Entry` 类型

**难点**

PR #24099 把 `progress` 从 `Entry` 类型 union 移除(已发的 progress 不再写盘),但**老 transcript 里**仍有 `{"type":"progress","uuid":...,"parentUuid":...}` 行。

`messages.get(progress_uuid)` 返回 undefined —— `buildConversationChain` 在第一条非 progress 消息指向 progress 时直接断链。

**解决**

`isLegacyProgressEntry`(169-178)做运行时检测(因为 TypeScript 已收窄为 `never`),`loadTranscriptFile` stage 4 在 parseJSONL 后单独维护 `progressBridge: Map<UUID, UUID | null>`(3629):

```
if isLegacyProgressEntry(entry):
    parent = entry.parentUuid
    progressBridge.set(uuid, parent && progressBridge.has(parent) ? progressBridge.get(parent) : parent)
```

**chain-resolve**:连续 N 个 progress 折叠成一次查找;后续 message 若 parent 落在 bridge 里,直接重写到 bridge 解析后的真实祖先(3649-3651)。

这是个**在线桥接** —— 不改 disk 内容,也不要求全量重写 transcript。

---

### A6. nested uuid 歧义 —— `agent_progress` 与 `mcpMeta`

**难点**

`walkChainBeforeParse` 想用字节级判定把 transcript 行筛成"transcript 行" vs "metadata 行"。判定规则:

- 顶层以 `{"parentUuid":` 开头 → transcript 行;
- 顶层 `uuid` 必须紧跟 `","timestamp":"` 后缀,定位 uuid 起点。

但两种情况会污染这条规则:

1. `agent_progress` 在 `data.message` 里**嵌套一个完整 Message**,序列化时 nested message 的 `"uuid":"<36>","timestamp":"..."` 字节出现在顶层 uuid 之前 → 多重后缀匹配;
2. `toolUseResult / mcpMeta` 把 `Record<string, unknown>` 序列化在顶层 uuid **之后**,server 控制的 payload 可能含 `{uuid:"<36>",timestamp:"..."}` → 也是多重后缀匹配。

**解决**

`pickDepthOneUuidCandidate`(3281-3310):

- 收集**所有**后缀匹配位置;
- 若 ≥2 个候选,用 brace-depth 扫描(`pickDepthOneUuidCandidate` 内部)找 depth=1 的那个;
- 若仅 1 个,直接采用;
- 0 个匹配时回退到首个 `"uuid":"` 起点。

comment 3344-3360 用本地 25k+ 行实测确认:depth-1 命中在合法 transcript 100% 准确。回退路径仅针对 progress 之类"timestamp 排在 uuid 之前"的极少数边界情况。

---

## B. append-only 文件的回放

### B1. snip 删了中段,幸存者 parentUuid 悬挂

**难点**

`/snip` 与 `HISTRY_SNIP`(`/force-snip` 之类)在文件中**只插入 boundary entry**(记下 `removedUuids`),**不**物理删除。survivor 的 parent 指向 removed 区域,walk 时撞墙。

**解决**

`applySnipRemovals`(1986-2043):

```
1. 收集所有 to-delete:遍历 messages,取每个 boundary.snipMetadata.removedUuids
2. 删 messages 中的 to-delete,记录 deletedParent (uuid → 自己的 parent)
3. 修幸存者:
   for each [uuid, msg] in messages:
       if msg.parentUuid && toDelete.has(msg.parentUuid):
           msg.parentUuid = resolve(msg.parentUuid)   // 走 deletedParent 直到非删除祖先
4. resolve() 用 path compression 把走过路径全部写回 map(O(α) 摊销)
```

`resolve(start)`:从 start 在 `deletedParent` 上往回走,直到非删除祖先。然后把 path 上每个 `deletedParent[p]` 都写成那个最终祖先,后续查询命中缓存。

末尾 `logEvent("tengu_snip_resume_filtered", {removed_count, relinked_count})` 报告这次 resume 实际删了多少、修了多少;回归监视。

---

### B2. compact 后的 preserved segment 拼接

**难点**

`compact_boundary` 的元数据里有 `preservedSegment: { headUuid, tailUuid, anchorUuid }`。语义是"压缩中保留 head↔tail 之间的消息,但要重新挂到 anchor(最近 boundary 的上一条 summary)"。

难在两处:

- 物理上,`preserved` 消息在 disk 上**仍带原始 pre-compact `parentUuid`**(因为 recordTranscript dedup 跳过,无法重写)。`walkChainBeforeParse` 会因为"看不到 preserved 段"而错误剔除;
- 逻辑上,boundary 处 chain 应截断,但 preserved 段应被 splice 进去。

**解决**

`applyPreservedSegmentRelinks`(1843-1960):

```
1. 找 absoluteLastBoundary 和 lastSegBoundary(可能不同:一次 reactive compact 后再 manual /compact,seg stale)
2. walk tail→head 校验 preservedUuids 是否都在 messages map 内
   不在 → logEvent tengu_relink_walk_broken + 直接 return(让 resume 加载完整历史)
3. 真正修补:
   a. head.parentUuid = anchor
   b. anchor 的其它子节点 → tail(sibling splice)
   c. preserved 段的 assistant message 把 input_tokens / output_tokens / cache_creation_input_tokens / cache_read_input_tokens 全部清零
      注释 1924-1927:disk 上的 input_tokens 反映 pre-compact context(~190K),不归零的话
      resume → 第一次 autocompact 会算成严重 overage,触发"立即 autocompact"循环
4. 修剪所有 absoluteLastBoundary 之前、不在 preservedUuids 的消息
```

`walkChainBeforeParse` 接收 `hasPreservedSegment` 标志(3547),有 seg 时**跳过预剔除**(3580-3585),把活儿留给 `applyPreservedSegmentRelinks`。

---

### B3. tombstone 写回与 50MB OOM 兜底

**难点**

流式 streaming 偶尔会插入"孤立消息"(服务器 200 后客户端断流),需要事后 tombstone 掉。但 append-only 文件**没有 random write**,只能 truncate + 后段往前挪。

- 目标消息通常在最后 64KB,truncate 一次就行;
- 但若在文件中部,**必须读全文**做 in-memory splice;
- GB 级文件全文读 + writeFile 一次,JS heap 直接爆。

**解决**

`removeMessageByUuid`(873-953)双路径:

**fast path**(消息在 tail 64KB):

```
open file, fstat → size
read tail LITE_READ_BUF_SIZE bytes
byte-search `"uuid":"<target>"` 在 tail
定位 lineStart (上一个 0x0a + 1) 和 lineEnd (下一个 0x0a 或 EOL)
fh.truncate(absLineStart)
if afterLen > 0:
    fh.write(tail, lineEnd, afterLen, absLineStart)
```

0x0a 字节扫描安全(注释 902):UTF-8 多字节序列里 0x0a 永不出现。

**slow path**(目标在 tail 外 或 临近):

```
if fileSize > MAX_TOMBSTONE_REWRITE_BYTES (50MB):
    logForDebugging("skipping tombstone: too large", warn) + return
else:
    readFile 全文 + split by '\n' + filter entry.uuid===target + writeFile back
```

MAX_TOMBSTONE_REWRITE_BYTES = 50MB(注释 122-123:Session files can grow to multiple GB,inc-3930);超大文件放弃 tombstone 是显式取舍,优于 OOM 让 process 死掉。

---

## C. 性能与延迟

### C1. 大文件预跳过 precompact

**难点**

`/compact` 在 JSONL 里插入 `compact_boundary` entry,边界之前的消息已经进入 summary,**对 resume 无用**。但又不能简单"全删"—— boundary 之前的 metadata(mode / agent-setting / tag / pr-link 等)仍需恢复。

**解决**

`loadTranscriptFile` stage 2(3542-3562):

```
if size > SKIP_PRECOMPACT_THRESHOLD:           // 默认 ~5MB,见 portable.ts
    scan = readTranscriptForLoad(file, size)
    buf = scan.postBoundaryBuf                 // 跳过 precompact 字节
    if scan.boundaryStartOffset > 0:
        metadataLines = await scanPreBoundaryMetadata(file, scan.boundaryStartOffset)
```

`scanPreBoundaryMetadata`(3163-3230)走 byte-level marker pass:

- `METADATA_TYPE_MARKERS` 是 9 个固定 marker 的 Buffer 数组;
- 绝大多数 chunk 没有 marker → 直接 `buf.indexOf(NEWLINE)` 跳过,不开新 string;
- 有 marker 才切 line 边界、push 进 metadataLines;
- `if (carry.length > 64 * 1024) carry = null` 防病态大行(注释 3214-3216);

实测(注释 3527-3535):151MB / 84% stale attr-snaps,这一阶段把 RSS 从 316MB 压到 155MB。

env-kill-switch `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` 在测试 / 调试时关闭这层跳过,让"加载一切"的旧行为可复现。

---

### C2. fork 分支预剔除

**难点**

JSONL append-only 性质意味着历史 fork / rewind 留下的死分支**永远在文件里**。`buildConversationChain` 反向 walk 时会丢弃它们,但 parseJSONL 已经把它们全 parse 完 —— 99% dead 的文件浪费 56ms 解析时间。

**解决**

`walkChainBeforeParse`(3312-3472)byte-level pre-filter:

1. 扫所有行,以 `{"parentUuid":` 前缀(line 的稳定前 8 字节)做 transcript / metadata 分流;
2. 解析每个 transcript 行的 `parentStart` / uuid;
3. 从 leaf 反向 walk parentUuid,得到"live chain"行集合 + 字节数;
4. 仅当能省**至少 50% 字节**才拼接裁剪后 buffer(注释 3440-3449:实测在 "107MB / 69% dead entries / 30% dead bytes" 处 memcpy 主导,需保守 50% gate);
5. merge 阶段用 subarray view + 一次 Buffer.concat。

`pickDepthOneUuidCandidate` 是这一步的关键依赖(A6)。

两个跳过条件:

- `hasPreservedSegment === true` —— 那些消息保留 pre-compact parentUuid,预剔除会误杀;
- `keepAllLeaves === true` —— `/insights` 等场景要看所有 fork 分支。

---

### C3. lite 头尾读

**难点**

`/resume` 列出会话要快,几百个 session 不可能全解析。

**解决**

`getSessionFilesLite`(4981-5022)只 `Promise.all + stat`:**不读文件**。每条 log 只带 `sessionId / mtime / size / ctime / projectPath`,mark `isLite: true` + `messages: []`。

`readLiteMetadata`(4745-4819)读**首尾各 64KB**:

- head:`isSidechain / projectPath / teamName / agentSetting / firstPrompt / customTitle / aiTitle`;
- tail:`lastPrompt / customTitle / summary / tag / gitBranch / prNumber / prUrl / prRepository`。

`extractLastJsonStringField` / `extractJsonStringField` 是字节级 substring 搜索,**不 JSON.parse**。`extractFirstPromptFromChunk` 是 fallback,顺序尝试多个字段并加防 truncate 处理(`extractJsonStringFieldPrefix`,4927)。

用户选了一条 lite log 后,`loadFullLog`(2955-3062)才会把整个 JSONL 解析成完整 `LogOption` —— 把"expensive parse"推迟到用户真实选择之后。

---

### C4. progressive enrichment

**难点**

picker 首屏要 50 条,继续滚动 / 搜索要更多。每条 enrich 要 stat + 64KB 头尾读。

**解决**

`INITIAL_ENRICH_COUNT = 50`(4583,~6.4MB I/O)首屏。`enrichLogs(0, count)` 返回 `{logs, nextIndex}`,UI 用 `nextIndex` 决定何时再 fetch 一批(调用方 responsibility)。

`enrichLog` 内部还做了**会话筛选**:

- `isSidechain === true` 跳过(子代理会话不该出现在主链 picker);
- `teamName` 存在时跳过(团队会话 = swarm,不该出现在个人 picker);

注释 5056-5060 解决了另一个坑:enrich 失败的会话(首消息超 16KB 摘要失败)曾被默默丢弃,导致 crash 后无法 resume;现在给 firstPrompt 一个 `'(session)'` fallback,让会话**仍然**出现在 picker 里。

---

## D. 元数据 EOF 对齐

### D1. custom-title 被大消息挤出 tail 窗口

**难点**

`/resume` picker 用 `readLiteMetadata` 扫尾 64KB 找 `customTitle`。但用户在 /rename 后,session 可能继续写入几 MB 消息,把 custom-title entry 推离尾窗口。

用户感受:**自己起的名字消失,变成自动提取的 firstPrompt**。看似"resume 拿错了文件",实际是元数据丢失。

**解决**

`reAppendSessionMetadata`(723-841)在三处被自动调用:

1. **`materializeSessionFile`**(986)—— session 首次 user/assistant 时把 mode / agentSetting / title 等落盘;
2. **cleanup handler**(451-462)—— 进程退出时 flush + re-append(注释 451-456:不 flush 会留下 stale tail metadata);
3. **compact 边界写入前**(`compact.ts / reactiveCompact.ts`)—— 让 metadata 紧跟新 boundary 之后,不要在 precompact 区间被跳过。

`reAppendSessionMetadata` 写顺序(注释 768):

```
lastPrompt → customTitle → tag → agent-name → agent-color →
agent-setting → mode → worktree-state → pr-link
```

`lastPrompt` 写在最前—— `extractLastJsonStringField` 总会找最后一个匹配,把"最近 user 在做什么"放在最远离 customTitle 的位置,而让 user-visible 的字段尽量贴近 EOF。

**外部写者兼容性**(注释 728-764):re-append 前**先 sync 扫尾**(用同一个 `LITE_READ_BUF_SIZE`),把 SDK renameSession / tagSession 写的更新值吸收进自己的 cache,然后才写 —— 避免 SDK 在我们开着 session 时改了 title,我们用 stale cache 把它盖回去。

---

### D2. AI title 不得 re-append

**难点**

如果 `reAppendSessionMetadata` 把 AI 生成过的标题也 re-append 到 EOF,可能出现:

- 用户进入 session → Haiku 写 `ai-title: "fix login bug"` → cache 写入;
- 用户 `/rename foo` → 写 `custom-title: "foo"`;
- compaction → `reAppendSessionMetadata` 把 `ai-title` 也写进去 → 但 cache 早已因为 customTitle 优先被清空 —— 重复但还好;
- 更糟:**用户没 rename,session 退出前 haiku 写过 ai-title,但我们退出时把 ai-title 与 custom-title 都 append,re-append 时 cache 里只剩 ai-title** —— 因为某些竞态,custom-title 已经被 evict 缓存,这时 ai-title 会被错误地提升为"用户视角可见的 title"。

**解决**

`saveAiGeneratedTitle`(2673-2679)**不更新 `currentSessionTitle` cache**,`reAppendSessionMetadata` 也**不**写 ai-title(723-841 行列表里没有 `aiTitle`)。

注释 2646-2672 详细说明"load-bearing"四点:

1. **读偏好**:readLiteMetadata 用 `customTitle ?? aiTitle`,custom-title 永远胜出;
2. **resume 安全**:restoreSessionMetadata 只填 customTitle,不填 aiTitle,cache 永远不持 ai-title;
3. **CAS 语义**:VS Code 的 `onlyIfNoCustomTitle` 检查 `customTitle` 字段,AI title 不会覆盖用户 title;
4. **指标**:`tengu_session_renamed` 不为 AI title 触发,保持 telemetry 干净。

`saveCustomTitle` 显式分两个函数 / 两种 entry type / 两种 field name,这是结构性保证而非运行时校验 —— 即使有人复制粘贴错误,改 JSONL line 的 type 字段就会立刻显形。

---

### D3. cache-only 元数据落盘时机

**难点**

`saveAgentSetting('foo')`、`saveMode('coordinator')`、`cacheSessionTitle('bar')` 都是 **cache-only**(2867、2885、2876),立即落盘会留下"metadata-only session 文件"(只有 cache 信息,没有任何 message)。

**解决**

`materializeSessionFile`(978-993):

```
if shouldSkipPersistence: return
ensureCurrentSessionFile()
reAppendSessionMetadata()                  // 把 cache-only 字段落盘
if pendingEntries.length > 0:
    for each entry in pendingEntries: appendEntry(entry)
```

触发时机:**首次 user/assistant** 进来时(993-1012):

```
if sessionFile === null && messages.some(m => m.type === 'user' || m.type === 'assistant'):
    await this.materializeSessionFile()
```

注释 985:"mode/agentSetting are cache-only pre-materialization; write them now"。Hook progress / attachment 消息则不触发 materialization,因为它们是 UI 元数据,不该提前把 session 文件造出来。

---

## E. 进程级状态机

### E1. switchSession 与 sessionFile 指针

**难点**

`switchSession(sid, projectDir)` 把 `getSessionId()` 切到新 sid,但 `getProject().sessionFile` 仍是上一次的指针。后续 `recordTranscript` 会写到错误文件;`reAppendSessionMetadata` 会写到错误文件。

**解决**

显式三步顺序(`sessionRestore.ts` 442-451):

```
switchSession(sid, dirname(transcriptPath))
await renameRecordingForSession()
await resetSessionFilePointer()              // 把 sessionFile = null + pendingEntries = []
restoreCostStateForSession(sid)
```

`resetSessionFilePointer`(1509)清空指针,让下次写入走 lazy-materialize 路径(appendEntry 1131-1145):

```
if sessionFile === null:
    pendingEntries.push(entry); return
```

**然后** `adoptResumedSessionFile`(1534-1538):

```
project.sessionFile = getTranscriptPath()    // 现在 getTranscriptPath 推导的是 resumed 的 sid
project.reAppendSessionMetadata(true)        // skipTitleRefresh=true(注释 1528-1532)
```

`skipTitleRefresh` 关键:resume 刚把同一份 disk 内容读进 cache,这里再 refresh 会把"刚刚 resume 时 `--name foo` 还没生效"的 cache 用 stale disk value 盖回去。注释里直接给出 `--name foo + -r <sid>` 的失败场景。

---

### E2. restoreSessionMetadata 与 worktree cd 的顺序约束

**难点**

`restoreWorktreeForResume` 要 `process.chdir(worktreePath)`,但 worktree path 来自 disk;如果 `restoreSessionMetadata` 还没把 `worktreeSession` 写入 cache,`restoreWorktreeForResume` 拿不到 cache 上下文。

但反过来:**先 cd 后 restore metadata**,在某些边界情况(目录不存在)里我们想用 cache 覆盖路径决定要不要 cd。

**解决**

固定顺序(`sessionRestore.ts` 469-487):

```
restoreSessionMetadata(...)                  // 1. cache 写入,包括 worktreeSession
restoreWorktreeForResume(result.worktreeSession)   // 2. 基于 cache 决定 cd 路径
adoptResumedSessionFile()                    // 3. 把 sessionFile 指针重新指向 resumed JSONL
```

注释 476-479 解释为什么顺序 1→2 而非 2→1:"Done after restoreSessionMetadata (which caches the worktree state from the transcript) so if the directory is gone we can override the cache before adoptResumedSessionFile writes it."

---

### E3. --name 与 disk title 冲突

**难点**

用户 `claude --name "My Project" -r <sid>`:

- disk 有 customTitle "old session title";
- --name 把 `currentSessionTitle` cache 设成 "My Project";
- 如果 `adoptResumedSessionFile` 走默认 `reAppendSessionMetadata()`(带 tail refresh),会扫尾 → 看到 disk "old session title" → 覆盖 cache → "My Project" 丢了。

**解决**

`adoptResumedSessionFile` 显式调 `reAppendSessionMetadata(true)`,`skipTitleRefresh=true`(1537)。

注释 1528-1532 给出完整原因链:

> skipTitleRefresh: restoreSessionMetadata populated the cache from the same disk read microseconds ago, so refreshing from the tail here is a no-op — unless --name was used, in which case it would clobber the fresh CLI title with the stale disk value. After this write, disk == cache and later calls (compaction, exit cleanup) absorb SDK writes normally.

也就是:**首次 resume 把 cache 拉满,但 CLI 的 `--name` 此时已先一步覆盖了 `currentSessionTitle` cache** —— disk refresh 是 redundant 且有害。后续 compaction / exit 走完整 refresh,SDK 写者已经写完才被读到。

---

### E4. fork 不应继承 worktreeSession

**难点**

`--fork-session` 复制磁盘 transcript 但起新 sid。如果 fork 写入 `worktreeSession` 到 disk,后续原 session 的"Remove worktree"对话框会把**仍然在用的** worktree 删掉。

**解决**

`sessionRestore.ts:470-472`:

```
restoreSessionMetadata(
    opts.forkSession ? {...result, worktreeSession: undefined} : result,
)
```

fork 路径**显式剥离** `worktreeSession`。注释 466-469:"Fork doesn't take ownership of the original session's worktree — a 'Remove' on the fork's exit dialog would delete a worktree the original session still references — so strip worktreeSession from the fork path so the cache stays unset."

同样的,`--fork-session` 也**不**调 `restoreCostStateForSession`、`restoreWorktreeForResume`、`adoptResumedSessionFile`,只在 `result.contentReplacements?.length` 时调 `recordContentReplacement`(452-463)—— 把 replacement 预填到 fork 的新 sid,避免 cache miss。

---

## F. 多方写入协调

### F1. sidechain UUID 不能进主 messageSet

**难点**

子代理的消息与主线程消息**写到不同文件**(主: `<sid>.jsonl`,sidechain:`<sid>/subagents/agent-<id>.jsonl`),UUID 仍可能撞(`--fork-session` 时主链继承 fork 来的消息,UUID 一样)。

`recordTranscript` 的 dedup 用 `messageSet` 检查"是否已记录",**如果 sidechain UUID 进了主 messageSet**:

1. 后续 sidechain 的消息被 dedup 跳过,sidechain 文件残缺;
2. 主线程消息若 `parentUuid` 指向 sidechain UUID,`buildConversationChain` 撞 wall。

**解决**

`Project.appendEntry` stage 注释 1234-1244 完整给出约束:

> Skip dedup for agent sidechain LOCAL writes — they go to a separate file, and fork-inherited parent messages share UUIDs with the main session transcript. Deduping against the main session's set would drop them, leaving the persisted sidechain transcript incomplete (resume-of-fork loads a 10KB file instead of the full 85KB inherited context).

> The sidechain bypass applies ONLY to the local file write — remote persistence (session-ingress) uses a single Last-Uuid chain per sessionId, so re-POSTing a UUID it already has 409s and eventually exhausts retries → gracefulShutdownSync(1). See inc-4718.

代码 `Project.appendEntry`(1245-1265):

```
const isAgentSidechain = entry.isSidechain && entry.agentId !== undefined
const isNewUuid = !messageSet.has(entry.uuid)
if (isAgentSidechain || isNewUuid):
    enqueueWrite(target, entry)              // sidechain 不查重
    if !isAgentSidechain:
        messageSet.add(entry.uuid)            // 但只把主链 UUID 加进 messageSet
        if isTranscriptMessage: persistToRemote(sessionId, entry)
```

---

### F2. remote 写入 epoch mismatch

**难点**

CCR v2 的 internal-event reader 在 epoch 不匹配(主 session 序号过期)时返回 409。如果 worker 还在 `await getInternalEventReader()`,daemon 已经 abort,worker 继续 POST 会进入无限 retry。

**解决**

`hydrateFromCCRv2InternalEvents`(1716-1721):

```
if (error.message === 'CCRClient: Epoch mismatch (409)') {
    throw error                            // 主动抛出,worker 退出
}
```

抛出而非 swallow —— 让 worker 显式 fail-fast,daemon 才能恢复正确 epoch 后再 spawn。

v1 (`sessionIngress.appendSessionLog`)失败则**触发 gracefulShutdownSync**(1343-1346):

```
if (!success) {
    logEvent('tengu_session_persistence_failed', {})
    gracefulShutdownSync(1, 'other')
}
```

注释 1335:HTTP POST 失败意味着 server 收不到,本地 session 与远端逐渐 drift —— **继续运行只会让状态越差越大**,所以显式关停。

---

### F3. fork + content-replacement 预填

**难点**

`ContentReplacementState` 记录"哪些 tool_use_id 的原始 result 太大,我们用一句 placeholder 替掉了,replacement 字符串 byte-identical 替回给模型"。

fork 拿到的 transcript 里**含**这些 tool_use_id(`messages` 里),但**没有**对应的 `content-replacement` entry(sessionId 仍指源 sid)。

如果 fork 不预填:

- 第一次 `query()` 把 message 中所有 tool_use_id 视为"FROZEN" → cache miss → 把原始大 payload 重发;
- prompt cache 命中率为 0,token overage 永久。

**解决**

`sessionRestore.ts:452-463`:

```
} else if (result.contentReplacements?.length) {
    // --fork-session keeps the fresh startup session ID. useLogMessages will
    // copy source messages into the new JSONL via recordTranscript, but
    // content-replacement entries are a separate entry type only written by
    // recordContentReplacement (which query.ts calls for newlyReplaced, never
    // the pre-loaded records). Without this seed, `claude -r {newSessionId}`
    // finds source tool_use_ids in messages but no matching replacement records
    // → they're classified as FROZEN → full content sent (cache miss, permanent
    // overage). insertContentReplacement stamps sessionId = getSessionId() =
    // the fresh ID, so loadTranscriptFile's keyed lookup will match.
    await recordContentReplacement(result.contentReplacements)
}
```

`recordContentReplacement` 写入到 **当前 sid 的 JSONL**,sourcing is `result.contentReplacements`(磁盘上的旧 sid 的条目)—— 通过 `insertContentReplacement` 写入"refresh 当前 sid entry",注释明确"`stamps sessionId = getSessionId() = the fresh ID`"。

之后 `loadTranscriptFile` 的 `contentReplacements.set(sessionId, ...)` 在 stage 4 用新 sid 做 key,正好命中 fork 预填的条目。

---

## G. UX / 兼容性兜底

### G1. lite→full 升级失败

**难点**

`loadFullLog`(2955-3062)若 `loadTranscriptFile` 抛错(磁盘损坏、权限丢失、文件被外部 truncate),resume 应该**退回 lite log** 让用户至少能看到这条 session 在 picker 里出现,而不是抛错整个 picker 都打不开。

**解决**

```
try {
    ...
    return { ...log, messages: ..., firstPrompt: ..., ... }
} catch {
    return log                          // 原 lite log 直接返回
}
```

整个 try 包住 `loadTranscriptFile + findLatestMessage + buildConversationChain + buildFileHistorySnapshotChain`,注释 3059:"If loading fails, return the original log"。

Picker 端 `isLiteLog` 仍能让 lite log 显示(只是 firstPrompt / messageCount 为空),用户至少能选——选了下次 loadFullLog 仍失败也是显式 UX 路径。

---

### G2. Resume 取消 / 错误显示

**难点**

`<ResumeCommand>` 让用户选 `Cancel` 时,要把"取消"信息展示给用户,但又**不要触发 query**(resume command 不应自动 send)。

**解决**

`src/commands/resume/resume.tsx:174-176`:

```
function handleCancel() {
    onDone('Resume cancelled', { display: 'system' })
}
```

`onDone` 的第二个参数 `{ display: 'system' }` 让 result message 用 system channel 而非 user channel,不显示在 transcript / 不发到模型。

错误显示对称:`'No conversations found to resume'` (line 113) / `'Failed to load conversations'` (line 118) —— 通过 `onDone(msg)` 显式渲染。

---

### G3. bash-mode 与 slash command 标题

**难点**

`<bash-input>git status</bash-input>` 形式的 session,firstPrompt 应该是 `! git status`(用户实际输入),而不是 `<bash-input>...` 这种 XML 标签。

slash command(`/clear` / `/model`)的 firstPrompt 也不该是原始 XML,要分两种情况:

- 内置命令(`/model sonnet`) → 跳过,不提供 title;
- 自定义命令 + 有 args(`/review reticulate splines`) → 保留,如 `/review reticulate splines`;
- 自定义命令但没 args(`/review`) → 跳过。

**解决**

`getFirstMeaningfulUserMessageTextContent`(1750-1816)三阶段判断:

1. **command-name-tag** 提取 → 检查 `builtInCommandNames().has(name)`,命中就 continue;
2. **bash-input** 提取 → 直接返回 `! ${bashInput}`(在 SKIP_FIRST_PROMPT_PATTERN 之前判断,注释 1799-1804);
3. **SKIP_FIRST_PROMPT_PATTERN** —— `<小写 letter 开头 xml tag>` + `[Request interrupted by user]` 一律跳过(IDE context / hook output / autonomous tick 等)。

`SKIP_FIRST_PROMPT_PATTERN` 用**通用正则**而非 allowlist(注释 119-120):Kept in sync with sessionStoragePortable.ts — generic pattern avoids an ever-growing allowlist that falls behind as new notification types ship。

---

### G4. Windows worktree 大小写

**难点**

Windows 上 `git worktree list` 可能输出 `C:/Users/...`,而 `~/.claude/projects/<encoded>/` 写入时记录的是 `c:/users/...`(或反过来)。直接 string compare 会让 prefix 匹配失败,session 在 picker 里失踪。

**解决**

`getStatOnlyLogsForWorktrees`(4134-4147):

```
const caseInsensitive = process.platform === 'win32'
const indexed = worktreePaths.map(wt => ({
    path: wt,
    prefix: caseInsensitive ? sanitized.toLowerCase() : sanitized,
}))
```

`dirName === prefix || dirName.startsWith(prefix + '-')`(4170)时只看 lowercased 版本。再按 prefix 长度倒序排(4147)—— 更具体的 prefix 先匹配,避免短 prefix 抢匹配(`-code-myrepo` 不应匹配 `-code-myrepo-worktree1`)。

---

### G5. `--no-session-persistence` 全链路一致性

**难点**

session storage 有 4 个写入入口:

- `Project.appendEntry`(主入口,过 `shouldSkipPersistence`);
- `appendEntryToFile`(sync fs.appendFileSync,re-append metadata 用,**不**过 shouldSkipPersistence);
- `writeAgentMetadata / writeRemoteAgentMetadata`(子代理侧车文件);
- `linkSessionToPR / saveCustomTitle / saveTaskSummary / saveTag`(直接 appendEntryToFile,不过 shouldSkipPersistence)。

如果不统一处理,`--no-session-persistence` 可能仍然在 disk 留下 custom-title / tag / agent metadata 文件,污染 `/resume` picker。

**解决**

```
shouldSkipPersistence(962):
    isEnvTruthy('TEST_ENABLE_SESSION_PERSISTENCE')              // 反向 opt-in 测试开关
    || (getNodeEnv() === 'test' && !allowTestPersistence)
    || getSettings_DEPRECATED()?.cleanupPeriodDays === 0
    || isSessionPersistenceDisabled()                          // --no-session-persistence
    || isEnvTruthy('CLAUDE_CODE_SKIP_PROMPT_HISTORY')
```

注释 962-971 解释:`Shared guard for appendEntry and materializeSessionFile so both skip consistently. The env var is set by tmuxSocket.ts so Tungsten-spawned test sessions don't pollute the user's --resume list.`

`materializeSessionFile`(982)独立检查一次(注释 979-981):`reAppendSessionMetadata writes via appendEntryToFile (not appendEntry) so it would bypass the per-entry persistence check and create a metadata-only file despite --no-session-persistence.`—— 这是个**显式承认"re-append 旁路"的细节**,不修就会破坏契约。

其它 save* 函数(`saveCustomTitle` 等)目前**仍**走 `appendEntryToFile`,这是已知不一致。`linkSessionToPR` 注释里说"for the current session so reAppendSessionMetadata can re-write after compaction" —— 故意保留以维持 cache 一致性,但需要知道这是个 trade-off。

---

## H. 综合考量

### H1. 恢复不止"读盘",而是"重建进程内一致视图"

`sessionStorage.ts` 的真正难点不是"把 JSONL parse 成对象"。真正的难点是:

- JSONL 是 append-only,所有"删除"都是 marker;
- transcript 与 metadata 与 session-scoped state 都混在同一个文件;
- main chain 与 sidechain 用同一 UUID 空间但写不同文件;
- 缓存与磁盘之间有"最后一公里"的对齐责任(`reAppendSessionMetadata`);
- 进程级单例的字段被三类消费者(REPL / 命令系统 / cost-tracker)同时读,改动一处必须不破坏其它两处;
- `--resume` / `--continue` / `--fork-session` / `--teleport` / `--remote` / picker 共用一套 `loadConversationForResume → processResumedConversation` 流水线但要接受不同的入口副作用。

### H2. 主要教训

1. **事后修补优于事先规范**:`recoverOrphanedParallelToolResults` 是 read-side fix,snip 是 boundary-time fix,A5 progressBridge 是 legacy compat —— 这些都不能在 write side 解决(streaming 必须逐 block emit,snip 不能物理删,legacy transcript 不能改)。
2. **结构性正确优于运行时校验**:`ai-title` 与 `custom-title` 拆成两个 entry type / 两个 field name,从结构上保证优先级;`messageSet` 与 `agentContentReplacements` 分两个 Map,从结构上隔离 main vs sidechain;`--fork-session` 显式 `worktreeSession: undefined`,从结构上隔离 fork 副作用。
3. **顺序约束写进注释,不能只靠口头**:`sessionRestore.ts` 把 9 步顺序写进代码块,关键不变量附 inline 注释,否则后人重排就会留下难以追踪的 bug。
4. **保底 fallback**:lite→full 升级失败退回 lite log,resume consistency delta 仅 telemetry 不阻塞,`precompact skip` 50MB 兜底拒绝 OOM —— 优雅降级比硬错更有用。
5. **环境 kill-switch 是救命线**:`CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP`、`tengu_pebble_leaf_prune`、`tengu_hawthorn_steeple`(content replacement)等门控让"激进优化"可在生产环境下回退。
6. **同步 fs 是双刃剑**:`appendEntryToFile` 用 `fs.appendFileSync`,在 cleanup handler / materialization 等"必须立即可见"的场景下是必要的;但这也意味着 `--no-session-persistence` 必须单独守门。

### H3. 仍然存在的技术债

| 风险 | 表现 | 缓解 |
| --- | --- | --- |
| 50MB OOM tombstone 兜底 | 大文件中部 orphan 消息无法清除 | 仅 log 警告;依赖 future incr-3930 重构 |
| save* 函数 绕过 shouldSkipPersistence | `--no-session-persistence` 下可能仍有 metadata-only 文件 | 在 materialization 守门,但子代理 metadata / PR link 等仍写 |
| legacy `progress` 行**永不能从老 transcript 中清掉** | disk 长期冗余 | bridge 是 read-side 临时方案;真正清理需要 rewrite 老 transcript |
| `pickDepthOneUuidCandidate` 假设 nested `Message` JSON 合法 | malformed JSON 时 fallback 到 last candidate | 无 fallback 验证;只能假设记录端不会写病态 JSONL |
| `walkChainBeforeParse` 50% gate 在边界场景可能误判 | 节省微小时反而增加 memcpy | 当前阈值来自实测,无自适应 |
| `--no-session-persistence` 与 reAppend 的不对称 | 写 metadata 的 appendEntryToFile 不守门 | 已知 trade-off,需要 all-or-nothing rewrite |

---

## 9. 锚点速查

```
src/utils/sessionStorage.ts:962          shouldSkipPersistence 守卫
src/utils/sessionStorage.ts:1131         appendEntry 主 dispatch (sidechain bypass)
src/utils/sessionStorage.ts:873          removeMessageByUuid fast/slow path
src/utils/sessionStorage.ts:1509         resetSessionFilePointer
src/utils/sessionStorage.ts:1534         adoptResumedSessionFile (skipTitleRefresh=true)
src/utils/sessionStorage.ts:1729         extractFirstPrompt
src/utils/sessionStorage.ts:1750         getFirstMeaningfulUserMessageTextContent
src/utils/sessionStorage.ts:1843         applyPreservedSegmentRelinks
src/utils/sessionStorage.ts:1986         applySnipRemovals
src/utils/sessionStorage.ts:2080         buildConversationChain (cycle 检测)
src/utils/sessionStorage.ts:2122         recoverOrphanedParallelToolResults
src/utils/sessionStorage.ts:2228         checkResumeConsistency
src/utils/sessionStorage.ts:2623         saveCustomTitle (与 ai-title 分离)
src/utils/sessionStorage.ts:2673         saveAiGeneratedTitle (不 cache, 不 re-append)
src/utils/sessionStorage.ts:2955         loadFullLog (失败兜底)
src/utils/sessionStorage.ts:3163         scanPreBoundaryMetadata
src/utils/sessionStorage.ts:3281         pickDepthOneUuidCandidate
src/utils/sessionStorage.ts:3312         walkChainBeforeParse (fork-branch 预剔除)
src/utils/sessionStorage.ts:3478         loadTranscriptFile (七阶段主解析器)
src/utils/sessionStorage.ts:3629         progressBridge (legacy progress 兼容)
src/utils/sessionStorage.ts:4745         readLiteMetadata (头尾 64KB 字节扫描)
src/utils/sessionStorage.ts:4981         getSessionFilesLite (stat only)
src/utils/sessionStorage.ts:5083         enrichLogs (progressive)
src/utils/conversationRecovery.ts:165    deserializeMessagesWithInterruptDetection
src/utils/conversationRecovery.ts:385    restoreSkillStateFromMessages
src/utils/conversationRecovery.ts:460    loadConversationForResume (主入口)
src/utils/sessionRestore.ts:276          ProcessedResume 类型
src/utils/sessionRestore.ts:409          processResumedConversation (9 步顺序)
src/utils/sessionRestore.ts:442-451      switchSession / renameRecording / reset / costRestore
src/utils/sessionRestore.ts:452-463      fork 路径 content-replacement 预填
src/utils/sessionRestore.ts:469-487      restoreMetadata / worktree / adopt 顺序
src/main.tsx:3607-4011                  resume 主分支编排
src/screens/REPL.tsx:1359               useState<MessageType[]>(initialMessages)
src/screens/REPL.tsx:1681               provisionContentReplacementState
src/screens/REPL.tsx:2164-2178          onMount restoreReadFileState
src/utils/plans.ts:164                  copyPlanForResume (slug 绑定 + CCR 恢复)
src/utils/fileHistory.ts:922             copyFileHistoryForResume (硬链接迁移)
src/utils/toolResultStorage.ts:447      provisionContentReplacementState (cache key)
src/utils/toolResultStorage.ts:960      reconstructContentReplacementState
```

---

## 10. 一句话总结

> **`claude --resume` 的真正难点,不在"把 JSONL parse 回对象",而在于:**
> 1. **append-only 文件的因果回放** —— snip / compact / fork / streaming 切分都可能在文件里留下死分支、悬挂 parent、嵌套歧义;
> 2. **三态混合的链路结构** —— main / sidechain / metadata / fork / resume 共享同一 UUID 空间却跨多个文件与状态机;
> 3. **进程级状态与磁盘视图的最后一公里** —— `reAppendSessionMetadata`、`adoptResumedSessionFile(skipTitleRefresh=true)`、`??=`、`fork 剥离 worktreeSession` 这套代码把"CLI 启动标志 / 缓存 / disk"三者强一致性维护;
> 4. **事后修补与运行时校验的取舍** —— `recoverOrphanedParallelToolResults`、`progressBridge`、`pickDepthOneUuidCandidate` 都是 read-side 救场,因为 write side 不可改(streaming / legacy / nested JSON);
> 5. **优雅降级而非硬错** —— lite→full 失败、50MB OOM 拒绝、50% gate skip、cycle 返回 partial transcript、snip 缺失字段用 pre-fix 行为。
> 
> 这些克服手段的共同点是"把约束显式化":分离 entry type / 分离 messageSet / 分离 cache / 分离写入路径 / 分离 dispatch 阶段,再附上详尽的注释解释"为什么"—— 让反编译版本也能保持与原版相同的不变量。