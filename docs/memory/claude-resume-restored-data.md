# `claude --resume` 需要恢复的数据

> 目标:对照源码说明执行 `claude --resume <sid|title>` 时,Claude Code 必须恢复的关键数据,以及每项数据的存在理由、落点位置和恢复步骤。
>
> 通用流程与时序见 [`claude-resume-flow.md`](./claude-resume-flow.md);本文件专注于"恢复什么 / 为什么 / 怎么恢复"。

---

## 1. 概述

REPL 默认是"无状态"的:一次新启动只持有启动期间的 settings、cwd、本地 agent 与 tool,而**对话历史、token 用量、文件历史快照、plan 文件引用、自定义标题、agent 命名等都是按 session ID 持久化到磁盘**。`--resume` 的本质就是把磁盘上 `~/.claude/projects/<encoded-cwd>/<sid>.jsonl` 的 NDJSON 转录里能还原出来的运行期状态重新加载一遍,使得新启动的进程表现得"像那条旧会话从未结束"。

按数据所在位置,可以分为四类:

| 类别 | 描述 | 主要文件 |
| --- | --- | --- |
| **磁盘工件**(目录文件级别) | 转录、备份、计划、asciicast | JSONL、file-history、plans、recordings |
| **运行期进程级单例** | module-global 缓存/cwd/cost-tracker | `bootstrap/state.ts`、`cost-tracker.ts`、`sessionStorage.ts` |
| **React 端会话状态** | store、UI 缓存、消息流 | `AppState`、`useState`、各种 hook |
| **派生/计算数据** | 由原始数据组合出的中间产物 | mode 匹配、agent definitions、attribution |

> 注:本仓库为反编译版本,所有 `feature()` 返回 `false`。下文涉及被门控的项(KAIROS、COORDINATOR_MODE、CONTEXT_COLLAPSE、tengu_hawthorn_steeple 等)在外部构建实际不可达,但**调用栈存在于源码中**,doc 仍按源码结构列出。

---

## 2. 数据项清单(按恢复顺序)

| # | 数据项 | 类别 | 落点位置 | 关键恢复函数 |
| --- | --- | --- | --- | --- |
| 1 | `~/.claude/projects/<encoded-cwd>/<sid>.jsonl` 转录 | 磁盘工件 | `getSessionProjectDir()` + `loadSessionFile()` | `loadConversationForResume()` |
| 2 | `lite → full` LogOption 升级 | 内存重构 | `loadSameRepoMessageLogs()` | `loadFullLog()` |
| 3 | 消息链 `conversationChain` | 内存重构 | `buildConversationChain()` | 同上 |
| 4 | plan slug 引用 | 磁盘引用 | `setPlanSlug(sid, slug)` | `copyPlanForResume()` |
| 5 | 文件历史快照备份 | 磁盘工件 | `~/.claude/file-history/<sid>/` | `copyFileHistoryForResume()` |
| 6 | 已调用技能状态 | 进程级单例 | `addInvokedSkill(STATE)` | `restoreSkillStateFromMessages()` |
| 7 | 反序列化后的 messages | 内存数据 | `useState<MessageType[]>` | `deserializeMessagesWithInterruptDetection()` |
| 8 | 未完成回合标记 | 派生 | `turnInterruptionState` | `detectTurnInterruption()` |
| 9 | 会话 ID / 会话文件指针 | 进程级单例 | `getSessionId()` / `sessionFile` | `switchSession` + `resetSessionFilePointer` |
| 10 | 转录追加指针 | 进程级单例 | `getProject().sessionFile` | `adoptResumedSessionFile()` |
| 11 | 录制文件(`.cast`)路径 | 磁盘工件 | `~/.claude/sessions/<sid>/recording.cast` | `renameRecordingForSession()` |
| 12 | token / 成本统计 | 进程级单例 | `costTracker` | `restoreCostStateForSession()` |
| 13 | session metadata(name/color/title/tag) | 内存缓存 | `getProject()` 内 cache | `restoreSessionMetadata()` |
| 14 | worktree 还原(`cd` 回原 worktree) | 进程级单例 | `process.cwd` | `restoreWorktreeForResume()` |
| 15 | agent 定义 / mainThreadAgent | 内存缓存 | `AppState.agentDefinitions` | `restoreAgentFromSession()` |
| 16 | standalone agent 名称/颜色 | AppState | `standaloneAgentContext` | `computeStandaloneAgentContext()` |
| 17 | mode 持久化(`coordinator`/`normal`) | 进程级单例 | `modeCache` | `matchSessionMode()` + `saveMode()` |
| 18 | attribution 状态 | AppState | `AppState.attribution` | `computeRestoredAttributionState()` |
| 19 | context-collapse 提交 + 快照快照 | 进程级单例 | `contextCollapseStore` | `restoreFromEntries()`(CONTEXT_COLLAPSE) |
| 20 | 已重写 tool result 缓存(`ContentReplacementState`) | 内存缓存 | `useRef({...})` | `provisionContentReplacementState()` |
| 21 | read file state 缓存 | 内存缓存 | `readFileState` | `restoreReadFileState(messages, cwd)` |
| 22 | remote agent task 列表 | AppState | `tasks` | `restoreRemoteAgentTasks()` |
| 23 | swarm 初始化计数 | 进程级单例 | Swarm internal | `useSwarmInitialization()` |
| 24 | SessionStart hook 消息 | 内存数据 | `messages[]` 末尾追加 | `processSessionStartHooks('resume', ...)` |
| 25 | AI 标题抑制标志 | React ref | `haikuTitleAttemptedRef` | 由 `initialMessages.length > 0` 派生 |
| 26 | useLogMessages 续写标记 | 进程级单例 | `lastRecordedUuid` | mount 同步后由 hook 自动建立 |

下文对每一项**给出落点、关键步骤与存在理由**。

---

## 3. 每个数据项的恢复细节

### 3.1 转录(NDJSON snapshot)— 一切的源头

**位置**:`loadConversationForResume`(`src/utils/conversationRecovery.ts:460`)

```
const filePath = join(getSessionProjectDir() ?? getProjectDir(getOriginalCwd()), `${sessionId}.jsonl`)
return loadTranscriptFile(filePath)
```

**关键步骤**

1. 调用 `getLastSessionLog(sid)`(`sessionStorage.ts:3875`),内部 `loadSessionFile(sid)` 读出整个 NDJSON,把每行归类到 `messages / summaries / customTitles / tags / agentSettings / worktreeStates / fileHistorySnapshots / attributionSnapshots / contentReplacements / contextCollapseCommits / contextCollapseSnapshot`。
2. 找到 `findLatestMessage(messages.values(), m => !m.isSidechain)`,从最后一个非 sidechain 消息出发。
3. `buildConversationChain(messages, lastMessage)` 沿 `parentUuid` 反向走回根,把链式历史重组成 `transcript`(去掉孤立分支)。

**理由**

这一份快照是所有其它"恢复"的源头。如果读不到、读错、或没拼好链,UI 显示的历史会丢失、cache key 会错位、`buildConversationChain` 失败会让整套 resume 退回到 picker。

> 备注:`getLastSessionLog` 顶部对 `getSessionMessages.cache` 的"预热"(`sessionStorage.ts:3893-3903`)是有意为之——`recordTranscript` 在 REPL mount 后会被调用,如果不在此处先填空集,会触发又一次完整的 JSONL 重复读取。

---

### 3.2 lite → full LogOption 升级

**位置**:`conversationRecovery.ts:539-541`

```
if (log) {
    if (isLiteLog(log)) log = await loadFullLog(log)
    ...
}
```

**关键步骤**

- `isLiteLog` 由 `loadSameRepoMessageLogsProgressive` 在搜索/选择阶段为节省 I/O 而设的标志。
- `loadFullLog` 把 `sessionId` 直接喂给 `loadSessionFile`,绕开摘要层。

**理由**

交互式挑选时通常是"几 KB 摘要 + 摘要索引",如果恢复时不升级到全文,后续 `deserializeMessagesWithInterruptDetection` 看到的就是被裁掉的历史,UI 上 `initialMessages` 会缺一大块。

---

### 3.3 messages 链(transcript → Reverse parent walk)

同 3.1,见 `buildConversationChain`,位于 `sessionStorage.ts` 内。这是一份逻辑结构的还原,**不是文件 I/O**,所以无法跳步骤。

**理由**

JSONL 在文件层是"行序",但消息树是按 `parentUuid`/`childUuids` 连的 forkable 结构。如果不重链,任何 compact、fork、tool_use pair 的语义都会出错(参见 `recordTranscript` 中"按 `parentUuid` 决定 `startingParentUuidHint`"的逻辑)。

---

### 3.4 plan slug 引用

**位置**:`copyPlanForResume(log, sid)`(`src/utils/plans.ts:164`)

**关键步骤**

1. `getSlugFromLog(log)` 从第一条带 `slug` 字段的消息里取 slug;
2. `setPlanSlug(targetSessionId, slug)` 把 slug 写入 slug cache,绑定到 resume 后的 session ID 上;
3. 读 `<plansDir>/<slug>.md`;若 ENOENT 且当前是 remote(CCR)环境,进入 recover 流程:
   - `findFileSnapshotEntry(log.messages, 'plan')` 找文件快照附件;
   - 否则 `recoverPlanFromMessages(log)` 从消息历史反推 plan 内容;
   - 找到就 `writeFile(planPath, recovered)` 写回磁盘。

**为什么重要**

- plan 系统命令(`/plan`、`/tasks` 之类)与 plan 文件**通过 slug 寻址**,session ID 切换后若不重新绑定,resume 后的命令会找不到原 plan 文件。
- 在 CCR(remote session)环境下,plan 文件可能根本没本地落盘,因此需要从消息历史或快照里反推恢复。

---

### 3.5 文件历史快照备份

**位置**:`copyFileHistoryForResume(log)`(`src/utils/fileHistory.ts:922`)

**关键步骤**

1. 取 `log.fileHistorySnapshots`(从 JSONL 里 `FileHistorySnapshotMessage` 重建);
2. 每个 snapshot 包含若干 `trackedFileBackups`,每个 backup 在磁盘上存在于:
   `~/.claude/file-history/<previousSessionId>/<backupFileName>`;
3. 用 `link()` 把它们硬链接到当前 session 目录;`EEXIST` 跳过,`ENOENT` 报错并 fallback 到 `copyFile`;
4. 全成功后 `recordFileHistorySnapshot(messageId, snapshot, false)` 把快照重建到当前会话的 JSONL(否则 `--rewind-files` 只能读上一次会话的快照,跨 session 失效)。

**为什么重要**

- 文件历史与 sessionId 强绑定(`/rewind-files <messageId>` 在 REPL 里的回滚路径要查 `file-history/<currentSid>/...`);
- 硬链接而非复制能让两 session 共享同一份大文件不变副本;
- 若迁移步骤失败,**重新记录**这一步不能被跳过,否则随后的工具调用拿不到历史 diff。

---

### 3.6 已调用技能状态(`invoked_skills`)

**位置**:`restoreSkillStateFromMessages(messages)`(`conversationRecovery.ts:385`)

**关键步骤**

- 扫描 messages 上的 `attachment.invoked_skills` 条目,对每条调用 `addInvokedSkill(name)` 加到 module-global `STATE` set;
- 同一函数最后再 `suppressNextSkillListing()`,避免 resume 后立刻重列一堆 skill。

**为什么重要**

- 一些 skill(尤其是 brief tool)会把状态写到 module-global,resume 后多次 compact 不应丢失;
- 这是一个**进程级单例**,如果不在恢复时插入,resume 后只能看到"当时的 skill"。

---

### 3.7 反序列化后的 messages(`deserializeMessagesWithInterruptDetection`)

**位置**:`conversationRecovery.ts:165-253`

**关键步骤**(流水线式)

```
1. migratedMessages    = msg.map(migrateLegacyAttachmentTypes)
2. strip invalid permissionMode values from user messages
3. filterUnresolvedToolUses(migratedMessages)         // 去掉孤立的 tool_use
4. filterOrphanedThinkingOnlyMessages(filteredToolUses)
5. filterWhitespaceOnlyAssistantMessages(filteredThinking)
6. detectTurnInterruption(filteredMessages)
   if interrupted_turn: append synthetic userMessage("Continue from where you left off.")
                          turnInterruptionState = {kind: 'interrupted_prompt', message}
7. if lastRelevant === user:
       splice(lastIndex+1, 0, createAssistantMessage({content: NO_RESPONSE_REQUESTED}))
```

**为什么每一步都必要**

| 步骤 | 失败后果 |
| --- | --- |
| `migrateLegacyAttachmentTypes` | 老的 `new_file` / `new_directory` attachment 类型没显式字段,UI 的文件名渲染、@ mention、escape 全失效 |
| 过滤 `permissionMode` | 历史上来自不同 build 的 mode 字符串会破坏运行时校验 |
| `filterUnresolvedToolUses` | API 不允许"孤立 tool_use 没有 tool_result",会 400 |
| `filterOrphanedThinkingOnlyMessages` | 流式输出在多 content_block + user 中断的情况下会留下"只见思考"的 assistant,resume 时直接 API 报错 |
| `filterWhitespaceOnlyAssistantMessages` | 用户中断在 "\n\n" 之后,会出现空 assistant,token 算错 |
| `detectTurnInterruption` + 续写 | gateway 重启 / SDK restart 场景下,resume 后第一步要 auto-continue |
| 末尾插 assistant sentinel | API 要求 `user,assistant` 严格交替;若最后一行是 user,会 400 |

---

### 3.8 未完成回合标记(`turnInterruptionState`)

**位置**:`conversationRecovery.ts:148-147`

- `deserializeMessagesWithInterruptDetection` 返回 `{messages, turnInterruptionState}`:
  - `{kind: 'none'}` —— 正常 session;
  - `{kind: 'interrupted_prompt', message}` —— 上次被打断,resume 后会注入"continue from where you left off"作为隐式 user。

**为什么重要**

主要服务于 **SDK 自动续跑** 场景;REPL 路径不强制使用这一字段,但读这一字段可以:
- 决定是否要在 prompt 输入区预填提示;
- 是否需要在底部显示 banner("Last turn was interrupted, type to continue")。

---

### 3.9 会话 ID / 会话文件指针

**位置**:`processResumedConversation`(`sessionRestore.ts:409`)

```
const sid = opts.sessionIdOverride ?? result.sessionId
if (sid) {
    switchSession(asSessionId(sid), opts.transcriptPath ? dirname(opts.transcriptPath) : null)
    await renameRecordingForSession()
    await resetSessionFilePointer()
    restoreCostStateForSession(sid)
}
```

**关键步骤**

1. `switchSession(sid, projectDir)`(`bootstrap/state.ts` 内)把 `getSessionId()` 切到 disk 上那条 JSONL 关联的 sid,同时把 cwd 切到 transcriptPath 的目录(若是跨工程 resume);
2. `resetSessionFilePointer()` 让 `getProject().sessionFile = null`,强制下次写入走"重新打开"路径;
3. `adoptResumedSessionFile()`(在 `restoreWorktreeForResume` 之后)显式把 `sessionFile` 重新指向原 JSONL 路径;
4. 对于 `--fork-session`,以上两个都不调用,保持 fresh ID,后续 `recordTranscript` 在新文件落盘。

**为什么重要**

- **cost-tracker**(`restoreCostStateForSession`)按 sid 索引,不切过去 cost 重置为 0;
- **recordTranscript** 在 user/assistant 往返时按当前 sid 找目标文件,sid 没切好,resume 后写错文件 → 后续 resume 永远找不到新增的内容;
- `getSessionId` 还影响 cache key(setSessionId 之后所有 `getProject().sessionFile` 都按它写入)。

---

### 3.10 转录追加指针(`adoptResumedSessionFile`)

**位置**:`sessionStorage.ts:1534`

```
export function adoptResumedSessionFile(): void {
    getProject().sessionFile = <recorded-transcript-path>
}
```

**关键步骤**

- 必须**在 `resetSessionFilePointer` 之后**调用,因为前一步把 `sessionFile` 置 null;
- 此函数**不创建文件**,它只是把"当前 sessionFile 指针"绑到原 JSONL 路径,这样 `reAppendSessionMetadata(exit handler)`、`recordTranscript(exit handler)` 不会写错文件。

**为什么重要**

- `restoreSessionMetadata` 写入 `custom-title` / `tag` 这些条目时,会 append 到文件末尾;若指针空着,会把它们写到"刚生成的 fresh session 文件"——直接覆盖在 resume 之上,污染原会话;
- 这就是注释(`sessionRestore.ts:479-487`)反复强调"resetSessionFilePointer then adoptResumedSessionFile"的根本原因。

---

### 3.11 asciicast 录制文件

**位置**:`renameRecordingForSession`(`src/utils/asciicast.ts:82`)

**关键步骤**

- 当前进程当前录制可能位于默认路径(以 fresh ID 命名);
- resume 后把它 `mv`/rename 成 `<resumedSid>.cast`,以便 `getSessionRecordingPaths()` 在 `/share` 命令里能查到。

**为什么重要**

- `/share` 流程依赖"录制 = 当前 sid",否则分享出去的链接拿不到 asciicast 录像。

---

### 3.12 token / 成本统计

**位置**:`restoreCostStateForSession(sid)`(`src/cost-tracker.ts:130`)

**关键步骤**

- 从 cost-tracker 的 session table 里按 sid 取 token-by-model breakdown、累计 cost、上下文窗口用量;
- 写入 `costTracker.sessions[sid]` 内当前进程的视图;
- reset view(避免重复累加)。

**为什么重要**

- UI 底部 status line、settings 里的 rate-limit 计算、`/status` 都依赖累计 token;
- 不恢复会导致 UI 显示"清零",且 `--continue` 的"上次对话已写几千 token"消失。

---

### 3.13 session metadata(name / color / customTitle / tag / agentName / agentColor / worktreeSession / prLink / contextCollapse*)

**位置**:`restoreSessionMetadata(meta)`(`sessionStorage.ts:2764`)

**关键步骤**

- 仅在值"truthy"时覆盖 module-global cache(field-by-field set-if-truthy);
- 这意味着 fork 路径会主动剥离掉 `worktreeSession`(避免 "Remove worktree" 把另一会话的工作树误删)。

**为什么重要**

- `/resume` 选择器顶部那一行字(自命名 `customTitle`)、左下角指示色、tag 筛选按钮、`/status` 输出,都来自这块 cache;
- 不恢复,resume 后 UI 会"匿名";
- 同时 exit handler 调 `reAppendSessionMetadata` 会把这里的内容再写回 JSONL,逐步摘要读到这条之前用户改了名都不会落盘。

---

### 3.14 worktree 恢复(`restoreWorktreeForResume`)

**位置**:`sessionRestore.ts`(实现于 `utils/worktree.ts`)

**关键步骤**

- 若 `result.worktreeSession` 存在:`process.chdir(worktreeSession.worktreePath)` + 把当前进程 cwd 缓存到 bootstrap state(`setCwd`、`setOriginalCwd`);
- 同步把 `currentWorktreeSession` 单例更新,这样后续 `getCurrentWorktreeSession()` 返回正确值;
- 这一步 `restoreSessionMetadata` 之后做,允许我们在目录不存在时覆盖 cache。

**为什么重要**

- 用户在 worktree 里运行了 claude、退出,过几天重启会发现 worktree 路径变了/不存在;
- 不切回去,所有相对路径、git diff、git branch UI 都会变成"原 repo root",失去 worktree 隔离意义;
- 同时也支撑 exit-handler 弹出的 "Keep worktree?" 对话框。

---

### 3.15 agent 定义(`restoreAgentFromSession`)

**位置**:`sessionRestore.ts:506`

```
const { agentDefinition: restoredAgent, agentType: resumedAgentType } =
    restoreAgentFromSession(result.agentSetting, currentMainAgent, agentDefinitions)
```

**关键步骤**

- `result.agentSetting` 来自 `LogOption.agentSetting`,JSONL 中是 `agent-setting` 条目,记录当时启动时用的 agent 名;
- 在 `agentDefinitions.activeAgents / allAgents` 列表中查到一个同名 agent,把它的定义一并写入 `restoredAgentDef`;
- 同时把 `agentType` 注入到初始 `AppState.agent`,UI 顶部的 agent 指示随之刷新。

**为什么重要**

- launch flag `--agent foo` 或 settings.json 里的 `agent:` 决定了系统 prompt / 可用工具列表;
- 不恢复,resume 后会被默认 agent 替换,行为上看起来像"会话被改了 system prompt"——尤其当原 agent 拥有 narrow permissions / 专用工具时,这些工具会立刻"消失"。

---

### 3.16 standalone agent context(name / color)

**位置**:`computeStandaloneAgentContext(agentName, agentColor)`(`sessionRestore.ts:175`)

**关键步骤**

- 收集历史会话里通过 `/rename`、`/color`、swarm spawn 设置的 agent 名/颜色;
- 包装成 `standaloneAgentContext` 注入 `initialState.standaloneAgentContext`。

**为什么重要**

- 让本地 agent 在 swarm 里仍能被同名寻址、染色 / 在 /agents 面板里仍可用;
- 是 swarm routing 的依据,缺失会导致 SendMessage 工具 routing 失败。

---

### 3.17 mode 持久化(coordinator/normal)

**位置**:`matchSessionMode(result.mode)` + `saveMode(currentMode)`(COORDINATOR_MODE 门控)

**关键步骤**

1. resume 时 `result.mode` 与"当前进程模式"做比对,不一致就发 warning system message + 重新拉 agent definitions;
2. `saveMode(currentMode)` 把当前模式写到 `~/.claude/mode.json` 之类 cache,以便未来的 `--continue` 沿用同一模式。

**为什么重要**

- coordinator 与 normal 走不同的 prompt / 不同 agent 集合;混用会导致 user 看到一个不一致的界面。

---

### 3.18 attribution 状态

**位置**:`computeRestoredAttributionState(result)`(`sessionRestore.ts`、`utils/attribution.ts`)

**关键步骤**

- 用 `attributionSnapshots`(JSONL 内 `attribution-snapshot` 条目)按 `leafUuid` 重放,重建"哪些行是 user、哪些是 model、哪些是 tool" 的归属 map;
- 注入 `initialState.attribution`(`AppState.attribution`)。

**为什么重要**

- `/attribution` 命令会展示每个被引用行归属,以及成本归因;
- 不恢复,resume 后追溯"刚才那次 tool call 出错在哪里"变得困难。

---

### 3.19 context-collapse commit + snapshot

**位置**:`restoreFromEntries(commits, snapshot)`(`services/contextCollapse/persist.js`,CONTEXT_COLLAPSE 门控)

**关键步骤**

- `commits`:压缩历史 commit 链,resume 后下一次自动 compact 应在此基础上继续;
- `snapshot`:staged queue 与 spawn 状态的快照,resume 后本地 sub-agent / 任务列表能复原。

**为什么重要**

- 压缩是模型的硬上限管理;若 resume 后丢失 commit 链,第一次自动压缩的"delta base" 会算错,导致压缩前后重复大量 token。

---

### 3.20 ContentReplacementState(已重写 tool result 缓存)

**位置**:`provisionContentReplacementState(initialMessages, initialContentReplacements)`(`toolResultStorage.ts:447`)

**关键步骤**

1. `reconstructContentReplacementState(messages, records)` 扫描 messages 收集 candidate tool_use_id,放入 `seenIds`;
2. 对 `initialContentReplacements`(JSONL 中 `content-replacement` 条目)按 candidate 命中者,把 `replacements.set(toolUseId, replacement)` 写回;
3. REPL 在 `useRef({...})` 持有一份(ref 用法见 `REPL.tsx:1681`),每次 `query()` 把"如果 tool result 已经被替换过,直接复用旧值"。

**为什么重要**

- 内容替换场景(big tool result 截断、binary asset 重写)的目标是**节约 token 并保持 cache 命中**;
- 替换字符串是与模型实际收到的字节一一对应的,resume 后若丢失,等价于把原始大 payload 又发一遍:既多花 token,又破坏 prompt cache key。

---

### 3.21 Read file state 缓存

**位置**:`restoreReadFileState(initialMessages, cwd)`(`REPL.tsx:2156` 起的内部 callback)

**关键步骤**

- 遍历 messages 中的 `<tool_use name="Read">` 工具记录,把"读了哪个文件、读时 snapshot、读到行"塞进 `readFileState`(`toolUseContext` 子结构);
- mount 阶段一次性执行,这样后续 Read 工具对同一文件能正确判断 "sinceLastRead" / 无需重读。

**为什么重要**

- 不重建的话,resume 后第一个 Read 相同文件会被视为"未读",自动 read again,浪费 token;
- 同时影响 `getTasksToRead` / 文件批处理等需要"读历史"的高级工具。

---

### 3.22 remote agent tasks

**位置**:`restoreRemoteAgentTasks({...})`(`REPL.tsx:2170` 调用,`utils/remoteAgents.ts` 实现)

**关键步骤**

- onMount 触发,通过 `getAppState / setAppState` 把当前 session 下挂载的远端 agent 重新拉回任务列表;
- 配合 `tasks` 子 store 填 entries。

**为什么重要**

- UI `TaskList` 面板、swarm UI 都需要这条信息;
- 不恢复会导致用户"看不到自己的 fork agent"。

---

### 3.23 swarm 初始化

**位置**:`useSwarmInitialization(setAppState, initialMessages, ...)`(`REPL.tsx:983`)

**关键步骤**

- 订阅 initialMessages 重新计算 swarm 内部计数(leaves / spawns / 子任务)、重新建立 promise chains。

**为什么重要**

- swarm 数据流是从消息 tree 派生;消息流换了根,swarm 必须重新初始化,否则子节点找不到 parent。

---

### 3.24 SessionStart hook 消息(`processSessionStartHooks('resume', {sessionId})`)

**位置**:`utils/sessionStart.ts:35`

**关键步骤**

- 按配置中的 hooks,触发 `SessionStart.resume` 事件;
- 每个 hook 可以返:`additionalContext`(拼到 system prompt)、`hookSpecificOutput.json`(写到 messages 末尾作为 system 消息)、`hookSpecificOutput.hookEventName: 'SessionStart', additionalContext: ...`;
- 结果 `HookResultMessage[]` 接到 `messages.push(...)`。

**为什么重要**

- 用户的 `.claude/settings.json` 可能声明"resume 时输出项目上下文"或"插一条提醒"。这种 hook 输出**必须出现在恢复后的消息流里**;
- 顺序上它发生在 `deserializeMessages…` 之后,返回的消息不带清理逻辑(直接 push)。

---

### 3.25 AI 标题抑制标志

**位置**:`REPL.tsx:1308-1310`

```
const haikuTitleAttemptedRef = useRef((initialMessages?.length ?? 0) > 0)
```

**理由**

- AI 标题策略默认在 user 第一次发问后触发,用 haiku 模型重写会话名;
- 恢复型会话已经有名字(很可能还是用户自定的),**不该再被 AI 重写**;
- `length > 0` 是判别"是否从盘上恢复了消息"的唯一信号,所以 resume 路径把它置 true。

---

### 3.26 useLogMessages 续写标记

**位置**:`REPL.tsx:4040`

```
useLogMessages(messages, messages.length === initialMessages?.length)
```

**关键步骤**

- 该 hook 在 mount 时读 `recordTranscript` 内部 cache 拉取 `messageSet`(已被 `getLastSessionLog` 预热);
- 首次 mount 时把 `lastRecordedUuid` 设为 `initialMessages.at(-1)?.uuid`,后续 user 输入触发的 `recordTranscript` 不会再写已经存在的 UUID(避免重复)。

**理由**

- 没有这个 ref,resume 后第一次 user 消息会让所有历史消息**再被 recordTranscript 写一遍**(JSONL 行数翻倍);
- 同函数内部 `parentUuid` 也是基于 `messageSet` 的"是否在新消息里"判断。

---

## 4. 顺序约束:为什么必须按这个顺序

`processResumedConversation`(`sessionRestore.ts:409`)是显式排序的,任何打乱都会留下 bug。下表列出关键的不变量:

```
 1. switchSession(sid)             ─┐
 2.   renameRecordingForSession()    │ 后续模块按"当前 sid"取数据
 3.   resetSessionFilePointer()      │ 让 sessionFile 变 null
 4.   restoreCostStateForSession()  ─┘
 5. restoreSessionMetadata(...)      ─ 写入"custom-title/tag..."的 cache
 6. restoreWorktreeForResume(...)    ─ 用 5 写完的 cache 做覆盖(cwd 切换)
 7. adoptResumedSessionFile()        ─ 把 sessionFile 重新指回原 JSONL
 8. (CONTEXT_COLLAPSE) restoreFromEntries(...)  ─ 跟 6/7 顺序无关,但要在 REPL 渲染前
 9. restoreAgentFromSession(...)     ─ 写到 mainThreadAgentDefinition + AppState.agent
10. saveMode(...)                    ─ mode cache
11. computeRestoredAttributionState(...)  ─ AppState.attribution
12. computeStandaloneAgentContext(...)    ─ AppState.standaloneAgentContext
13. updateSessionName(result.agentName)   ─ 同步 session name 写回
14. refreshAgentDefinitionsForModeSwitch(...) ─ 6/9/17 都触发模式不一致时的 agent 列表重算
15. return ProcessedResume           ─ 最后再构造返回值
```

为什么不许乱序:

- **3 → 7** 不能反过来:`resetSessionFilePointer` 必须先把指针清掉,否则 `adoptResumedSessionFile` 写入会被覆盖逻辑忽略。
- **5 → 6** 必须成立:`restoreSessionMetadata` 先把 `worktreeSession` 写进 cache,然后 `restoreWorktreeForResume` 才能在 cache 之上做 cd 覆盖。
- **9 → 14**:`refreshAgentDefinitionsForModeSwitch` 要看到 `modeWarning` 才决定要不要刷新;mode 来自 9 的 `agentSetting`,所以 9 必须先做完。
- **13 → 12**:`updateSessionName` 必须在 `computeStandaloneAgentContext` 之后(或并行)执行,以确保 React 后续 setState 不会重置名称。

另一个并行的恢复,发生在 `REPL` onMount effect(`REPL.tsx:2164-2178`):

```
useEffect(() => {
    if (initialMessages && initialMessages.length > 0) {
        restoreReadFileState(initialMessages, getOriginalCwd())
        void restoreRemoteAgentTasks({...})
    }
}, [])
```

它只在 REPL 已经渲染拿到 `initialMessages` 后才执行,不需要参与 `processResumedConversation` 流程。

---

## 5. 派生 / 由原始数据组合的项

下面这些条目不是单独从 JSONL 里"读取一个值",而是用上述原始字段做组合 / 计算后注入:

| 派生项 | 输入 | 派生逻辑 | 注入位置 |
| --- | --- | --- | --- |
| `AppState.agent` | `result.agentSetting` | `restoreAgentFromSession` | `ProcessResume.initialState.agent` |
| `AppState.attribution` | `result.attributionSnapshots` | `computeRestoredAttributionState` | 同上 |
| `AppState.standaloneAgentContext` | `result.agentName / agentColor` | `computeStandaloneAgentContext` | 同上 |
| `AppState.agentDefinitions` | `modeWarning` + current `agentDefinitions` | `refreshAgentDefinitionsForModeSwitch` | 同上 |
| `ProcessedResume.messages` | `result.messages + hookMessages` | `loadConversationForResume` push hookMessages | `launchRepl` → `REPL.initialMessages` |
| `REPL.messages` | `initialMessages` | `useState<MessageType[]>(initialMessages ?? [])` | `REPL.tsx:1359` |
| `useLogMessages.lastRecordedUuid` | `initialMessages.at(-1)?.uuid` | mount 时取末尾 | `useLogMessages` 内 |

---

## 6. 已知边界场景(会部分跳过恢复)

| 场景 | 跳过哪些恢复 | 原因 |
| --- | --- | --- |
| `--print / -p` 模式 | 不走 `launchRepl` / `processResumedConversation`,改走 `loadInitialMessages` + `processResumedConversation`(同一函数,但省略 cost / standaloneAgentContext / mode 切换);`REPL-only` 项(read file state、swarm init、title suppress) 全部不执行 | print 没有 UI 渲染这些状态无意义 |
| `--fork-session` | `restoreCostStateForSession` 不调;`worktreeSession` 在 `restoreSessionMetadata` 时被显式剥离;`sessionFile` 指针不切到原 JSONL,而是新建;`contentReplacements` 转写到新文件 | fork 不应"接管"原 session |
| `--no-session-persistence` | `processResumedConversation` 仍会调用,但 `recordTranscript` 走 `shouldSkipPersistence`,几乎所有"写入"相关跳过 | 显式要求不落盘 |
| `--teleport` / `--remote` | 走 `processMessagesForTeleportResume` / 远程 fetch,与本地 resume 分流 | 跨进程恢复,sid 不在本机 |
| `SessionStart:resume` hook 抛错 | `processResumedConversation` 抛错会冒泡,整个 resume 流程 cancel(对应 main.tsx `exitWithError`) | hook 不可 part-apply,部分恢复会留下不一致状态 |
| `lite` log 中找不到 full | 失败 → `loadConversationForResume` 返回 `null` → main.tsx 走 `exitWithError("No conversation found with session ID: ...")` | 没有完整数据就不能继续 |
| `recordContentReplacement` 写入失败(fork) | fork 后第一次 `query()` 会把所有 tool_use_id 标为 FROZEN,导致 prompt cache miss 和 overage | 见注释:`sessionRestore.ts:452-462` |
| `restoreWorktreeForResume` 落到已删除 worktree | `process.chdir` 会抛错,resume 中止(`main.tsx:exitWithError`) | 显式失败比静默 fallback 安全 |

---

## 7. 一图总览

```
                                ┌──────────────────────────────────────┐
                                │ Disk: ~/.claude/                     │
                                │   projects/<cwdHash>/<sid>.jsonl     │
                                │     ├─ messages                      │
                                │     ├─ summaries / customTitles      │
                                │     ├─ fileHistorySnapshots          │
                                │     ├─ attributionSnapshots          │
                                │     ├─ contentReplacements           │
                                │     ├─ contextCollapse{Commits…}     │
                                │     ├─ worktreeStates                │
                                │     └─ agentName/Color/Tag/PR…        │
                                │   plans/<slug>.md                    │
                                │   file-history/<sid>/*               │
                                │   sessions/<sid>/recording.cast      │
                                └──────────────┬───────────────────────┘
                                               │
                       loadTranscriptFile / loadSessionFile
                                               ▼
                          loadConversationForResume()
                            ├─ lite→full upgrade
                            ├─ copyPlanForResume   ┐
                            ├─ copyFileHistory…    ├─ 磁盘迁移动作
                            ├─ restoreSkillState…  │
                            ├─ deserializeMessages( … + sentinel + interruption )
                            └─ processSessionStartHooks('resume')
                                               │
                                               ▼
                          processResumedConversation( )
                            ├─ switchSession  → resetSessionFilePointer
                            ├─ renameRecordingForSession
                            ├─ restoreCostStateForSession
                            ├─ restoreSessionMetadata
                            ├─ restoreWorktreeForResume (cd + worktreeCache)
                            ├─ adoptResumedSessionFile
                            ├─ restoreFromEntries(contextCollapse)
                            ├─ restoreAgentFromSession → AppState.agent
                            ├─ matchSessionMode + saveMode
                            ├─ computeRestoredAttributionState
                            ├─ computeStandaloneAgentContext
                            └─ updateSessionName
                                               │
                                               ▼
                          ProcessedResume
                            { messages, fileHistorySnapshots, contentReplacements,
                              agentName, agentColor, restoredAgentDef,
                              initialState { agent, attribution, standaloneAgentContext,
                                            agentDefinitions, ... } }
                                               │
                                               ▼
                          launchRepl( initialState, { initialMessages,
                                                      initialFileHistorySnapshots,
                                                      initialContentReplacements,
                                                      initialAgentName,
                                                      initialAgentColor,
                                                      mainThreadAgentDefinition } )
                                               │
                                               ▼
                          REPL (mount)
                            ├─ useState<MessageType[]>(initialMessages)
                            ├─ useLogMessages(messages, length === initial…)
                            ├─ provisionContentReplacementState(...)
                            ├─ haikuTitleAttemptedRef = initialMessages.length > 0
                            ├─ onMount effect:
                            │     ├─ restoreReadFileState(initialMessages, cwd)
                            │     └─ restoreRemoteAgentTasks(...)
                            └─ useSwarmInitialization(setAppState, initialMessages…)
```

---

## 8. 一句话总结

> **`claude --resume` 要恢复的数据,本质上就是"把上一次的 session 在进程内外的全部状态重新演一遍"**——
> 磁盘工件迁回去(plan / file-history / asciicast)、模块级单例重新填充(`sessionId / cost / contextCollapse / readFileState / skillState`)、React 树用 `initialMessages` 一次性 seed(消息流 + title 标志 + content replacement)、AppState 透过 `initialState` 加载(attribution / agent / standalone / mode / definitions)。
> 
> 这套数据的**完整性**取决于 `switchSession → adoptResumedSessionFile → restoreSessionMetadata → restoreWorktreeForResume` 这条主链不被打断;一旦中间步骤被旁路,session 指针就会偏向 fresh 目录,resume 后写的每一条都会落错位置。

---

## 9. 锚点速查

```
src/utils/conversationRecovery.ts:460   loadConversationForResume
src/utils/conversationRecovery.ts:165   deserializeMessagesWithInterruptDetection
src/utils/conversationRecovery.ts:385   restoreSkillStateFromMessages
src/utils/sessionStorage.ts:3875        getLastSessionLog
src/utils/sessionStorage.ts:1509        resetSessionFilePointer
src/utils/sessionStorage.ts:1534        adoptResumedSessionFile
src/utils/sessionStorage.ts:2764        restoreSessionMetadata
src/utils/sessionStorage.ts:2228        checkResumeConsistency
src/utils/sessionRestore.ts:175         computeStandaloneAgentContext
src/utils/sessionRestore.ts:276         ProcessedResume
src/utils/sessionRestore.ts:409         processResumedConversation
src/utils/plans.ts:164                  copyPlanForResume
src/utils/fileHistory.ts:922            copyFileHistoryForResume
src/utils/toolResultStorage.ts:447      provisionContentReplacementState
src/utils/toolResultStorage.ts:960      reconstructContentReplacementState
src/utils/asciicast.ts:82               renameRecordingForSession
src/cost-tracker.ts:130                 restoreCostStateForSession
src/main.tsx:3607-4011                  resume branch (CLI 编排)
src/screens/REPL.tsx:700-744            Props(initialMessages etc.)
src/screens/REPL.tsx:1308              haikuTitleAttemptedRef
src/screens/REPL.tsx:1359              messages useState
src/screens/REPL.tsx:1681              provisionContentReplacementState
src/screens/REPL.tsx:2156              restoreReadFileState
src/screens/REPL.tsx:2164-2178          onMount effect
src/screens/REPL.tsx:4040              useLogMessages
src/screens/ResumeConversation.tsx     interactive picker 同名恢复路径
src/services/contextCollapse/persist.js restoreFromEntries (CONTEXT_COLLAPSE)
```
