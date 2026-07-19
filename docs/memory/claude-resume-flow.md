# `claude --resume` 执行流程详解

> 目标：对照本仓库源码,梳理 `claude -r <id|title> [searchTerm]` 与裸的 `claude --resume`(交互式选择器)在运行时经历的完整路径,覆盖 CLI 参数解析、会话定位、转录文件加载、消息反序列化、上下文补全、REPL 渲染与挂载,以及交互式选择器的分支。
>
> 本仓库是基于 Anthropic 官方 Claude Code CLI 的反编译/裁剪版本,所有 `feature()` 在运行时均为 `false`,因此文中涉及到的若干被门控的分支(COORDINATOR_MODE / KAIROS / BRIDGE_MODE / FALLBACK 等)在外部构建里实际不可达。文档以源码结构为主线,在相应位置标注"门控"。

---

## 1. 目录速览

| 阶段 | 关键文件 | 行号 / 符号 |
| --- | --- | --- |
| 入口与参数解析 | `src/entrypoints/cli.tsx` | `main()` |
| 默认 action | `src/main.tsx` | `run()`、`.option('-r, --resume [value]', …)`、`program.action` |
| 参数分流(取 UUID / 标题 / 文件 / 搜索词) | `src/main.tsx` | 第 3607–3957 行 |
| 转录加载 | `src/utils/conversationRecovery.ts` | `loadConversationForResume()` |
| 转录读取/JSONL 解析 | `src/utils/sessionStorage.ts` | `loadMessageLogs`、`getLastSessionLog`、`loadTranscriptFile`、`buildConversationChain` |
| 反序列化与补全 | `src/utils/conversationRecovery.ts` | `deserializeMessagesWithInterruptDetection`、`restoreSkillStateFromMessages` |
| 会话状态切换 / 元数据恢复 | `src/utils/sessionRestore.ts` | `processResumedConversation()` |
| 启动 REPL | `src/replLauncher.tsx` | `launchRepl()` |
| 交互式选择器 | `src/dialogLaunchers.tsx` + `src/screens/ResumeConversation.tsx` + `src/commands/resume/resume.tsx` | `launchResumeChooser`、`ResumeConversation`、`ResumeCommand`、`LogSelector` |
| REPL 挂载恢复 | `src/screens/REPL.tsx` | `Props.initialMessages`、`useState<MessageType[]>(initialMessages ?? [])`、`restoreReadFileState`、`useSwarmInitialization`、`useLogMessages` |
| 非交互/--print 路径 | `src/cli/print.ts` | `runHeadless` → `loadInitialMessages` |

下文按"控制流"自顶向下展开。

---

## 2. CLI 入口与 fast-path(`src/entrypoints/cli.tsx`)

```
src/entrypoints/cli.tsx:60   async function main(): Promise<void>
```

`cli.tsx` 是 Bun 真实入口,它先做一系列 fast-path,只对**未匹配任何子命令、且不是 `--version / --dump-system-prompt / --chrome-* / --daemon-worker / --remote-control / --daemon / claude ps|logs|attach|kill` 等的 argv** 调用 `cliMain`(即 `src/main.tsx` 的 `main`)。

注意:

- `--resume` 不在任何 fast-path 里短路,因此会触发对 `main.tsx` 的 lazy import。
- `cli.tsx:309` 起在没有 fast-path 命中时执行:
  ```
  const { main: cliMain } = await import('../main.jsx')
  await cliMain()
  ```

`main.tsx:main()` 负责设置 cursor、warning handler、`__CFBundleIdentifier` 嗅探、决定 `clientType`、调用 `eagerLoadSettings()` 解析 settings、最终:

```
src/main.tsx:1009    await run()
```

`run()` 实例化 Commander、注册所有 `--resume` 之类的 option、并定义根命令的 action handler,然后 `program.parseAsync(process.argv)`。

---

## 3. Commander 选项注册与 action handler

### 3.1 Option 定义

```
src/main.tsx:1235
.option('-r, --resume [value]', 'Resume a conversation by session ID, or open interactive picker with optional search term', value => value || true)
.option('--fork-session', 'When resuming, create a new session ID instead of reusing the original (use with --resume or --continue)', () => true)
...
.addOption(new Option('--resume-session-at <message id>', 'When resuming, only messages up to and including the assistant message with <message id> (use with --resume in print mode)').argParser(String).hideHelp())
```

`--resume` 用 `[value]` 形式,即:

| 写法 | Commander 解析出的 `options.resume` |
| --- | --- |
| `claude -r` | `true` |
| `claude --resume` | `true` |
| `claude -r <something>` | `'<something>'`(非空字符串)|
| `claude --resume <something>` | 同上 |

`value => value || true` 把空串规整成布尔。所有下游分支都是围绕"是 `true`、字符串、还是其它"展开。

### 3.2 根命令 action

`program.action(async (prompt, options) => { ... })` 在 `src/main.tsx:1253`。它的"简历分支"集中在第 3607–4011 行:

```
src/main.tsx:3607
} else if (options.resume || options.fromPr || teleport || remote !== null) {
    // Handle resume flow - from file (ant-only), session ID, or interactive selector
```

进入这一支需要至少触发 `--resume`、`--from-pr`、`--teleport`、`--remote` 其一。我们只追 `options.resume`。

### 3.3 action 内部前奏

1. **`clearSessionCaches()`**(3607 行附近)——清掉陈旧的解析结果,确保 resume 后看到的文件/skills 是新鲜的。
2. 准备三个变量:
   - `messages: MessageType[] | null = null`
   - `processedResume: ProcessedResume | undefined`
   - `maybeSessionId = validateUuid(options.resume)`
   - `searchTerm: string | undefined`(当作选择器初始查询)
3. **自定义标题精确匹配**(3636–3651):若 `options.resume` 是字符串且不是 UUID,则做 `searchSessionsByCustomTitle(value, { exact: true })`,命中唯一时把 `matchedLog` 记下、从 LogOption 里取 `getSessionIdFromLog`,落空则把字符串当成 picker 搜索词。
4. `--from-pr` / `--remote` / `--teleport` 各自的分流在 3655–3832 行,它们与 `--resume` 共用同一条 resume 管道。

### 3.4 ant-only:从 ccshare / 路径加载(3833–3917)

ant 编译时才存在的分支(`process.env.USER_TYPE === 'ant'` 直接打开字符串字面量 'ant',`build` 时死码消除):

- 若 `options.resume` 是字符串但不是 UUID:
  1. `parseCcshareId(value)` —— `ccshare://...` 形式;
  2. 否则 `resolve(options.resume)` 当作本地路径,尝试 `loadTranscriptFromFile(resolvedPath)`(`.jsonl` 走 `loadTranscriptFile` + 重建链;`.json` 走 `JSON.parse`)。

不论哪条路径,都把转录包成 `LogOption`,然后调用 `processResumedConversation(...)` 拿到 `ProcessedResume`,塞到 `processedResume`。

### 3.5 主分支:按 UUID 加载(3919–3957)

```
src/main.tsx:3920  if (maybeSessionId) {
    const sessionId = maybeSessionId
    const result = await loadConversationForResume(matchedLog ?? sessionId, undefined)
    if (!result) {
        logEvent('tengu_session_resumed', { entrypoint: 'cli_flag', success: false })
        return await exitWithError(root, `No conversation found with session ID: ${sessionId}`)
    }
    const fullPath = matchedLog?.fullPath ?? result.fullPath
    processedResume = await processResumedConversation(result, {
        forkSession: !!options.forkSession,
        sessionIdOverride: sessionId,
        transcriptPath: fullPath
    }, resumeContext)
    ...
}
```

`loadConversationForResume` 接受 `string | LogOption | undefined`,并在 `string` 分支里走 `getLastSessionLog`。

### 3.6 决定渲染路径(3972–4011)

```
src/main.tsx:3972
const resumeData = processedResume ?? (Array.isArray(messages) ? { ... } : undefined)
if (resumeData) {
    ...
    await launchRepl(root, { initialState: resumeData.initialState }, {
        ...sessionConfig,
        mainThreadAgentDefinition: resumeData.restoredAgentDef ?? mainThreadAgentDefinition,
        initialMessages: resumeData.messages,
        initialFileHistorySnapshots: resumeData.fileHistorySnapshots,
        initialContentReplacements: resumeData.contentReplacements,
        initialAgentName: resumeData.agentName,
        initialAgentColor: resumeData.agentColor,
    }, renderAndRun)
} else {
    // Show interactive selector (includes same-repo worktrees)
    await launchResumeChooser(root, { initialState }, getWorktreePaths(getOriginalCwd()), {
        ...sessionConfig,
        initialSearchQuery: searchTerm,
        forkSession: options.forkSession,
        filterByPr
    })
}
```

注意 `processedResume` 为空意味着"给定的值无法直接定位,但仍然有可能是搜索词"——这种情况才会落到选择器。若连 searchTerm 都没有(比如裸 `claude --resume`),`ResumeConversation` 内部仍会调用 `loadSameRepoMessageLogs`,列出所有可恢复对话供挑选。

---

## 4. 转录加载

### 4.1 落点:文件位置

`sessions.ts / sessionStorage.ts` 把对话写进 `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`(每条对话一行 NDJSON,不同种类的消息/快照/标题/标签/工作树态等也是 JSONL 行)。

恢复时,从这个根目录往下扫:

- 同仓库及其所有 worktree:`getWorktreePaths(getOriginalCwd())` 给出一组 cwd;
- `loadSameRepoMessageLogs(worktreePaths)` 同时扫描这若干 `<encoded-cwd>/<projectHash>/`、`sessions/index.json` 等;(参见 `src/utils/sessionStorage.ts:4079`)
- `loadAllProjectsMessageLogs()` 则扫所有 `~/.claude/projects/*` 下的会话。

### 4.2 `loadConversationForResume`

`src/utils/conversationRecovery.ts:460` 是所有 `claude --resume`/`--continue`/`<ResumeConversation>`/<SDK resume>` 的统一入口。简化后的逻辑:

```
async function loadConversationForResume(source, sourceJsonlFile) {
    let log: LogOption | null = null
    let messages: Message[] | null = null
    let sessionId: UUID | undefined

    if (source === undefined) {                    // --continue
        const logsPromise = loadMessageLogs()
        const live = ... // 拿 live 列表(LIVE_BG / daemon),做一个 skip 集
        const logs = await logsPromise
        log = logs.find(l => !skip.has(getSessionIdFromLog(l))) ?? null
    } else if (sourceJsonlFile) {                  // 显式 .jsonl 路径
        const loaded = await loadMessagesFromJsonlPath(sourceJsonlFile)
        messages = loaded.messages
        sessionId = loaded.sessionId
    } else if (typeof source === 'string') {       // sessionId
        log = await getLastSessionLog(source as UUID)
        sessionId = source as UUID
    } else {
        log = source                               // 已经拿了 LogOption
    }

    if (!log && !messages) return null

    if (log) {
        if (isLiteLog(log)) log = await loadFullLog(log)  // 进度加载时仅取摘要,在这里把全文捞回
        if (!sessionId) sessionId = getSessionIdFromLog(log)
        if (sessionId) await copyPlanForResume(log, asSessionId(sessionId))
        void copyFileHistoryForResume(log)
        messages = log.messages
        checkResumeConsistency(messages)
    }

    restoreSkillStateFromMessages(messages!)
    const deserialized = deserializeMessagesWithInterruptDetection(messages!)
    messages = deserialized.messages
    const hookMessages = await processSessionStartHooks('resume', { sessionId })
    messages.push(...hookMessages)

    return {
        messages,
        turnInterruptionState: deserialized.turnInterruptionState,
        fileHistorySnapshots: log?.fileHistorySnapshots,
        attributionSnapshots: log?.attributionSnapshots,
        contentReplacements: log?.contentReplacements,
        contextCollapseCommits: log?.contextCollapseCommits,
        contextCollapseSnapshot: log?.contextCollapseSnapshot,
        sessionId,
        agentName, agentColor, agentSetting,
        customTitle, tag, mode, worktreeSession,
        prNumber, prUrl, prRepository,
        fullPath: log?.fullPath,
    }
}
```

要点:

- **lite vs full log**:为了避免 `claude --resume` 时一次性读所有超大文件,搜索/列表阶段(`enrichLogs`)只写入一份精简摘要,只在选定后由 `loadFullLog(log)` 解析整个 JSONL。
- `copyPlanForResume` 把 `~/.claude/plans/<sid-slug>/...` 拷到当前 session slug 下,以保证 plans 命令与原会话保持引用关系。
- `copyFileHistoryForResume` 异步把文件历史快照搬到当前 session 目录下,使得 `--rewind-files` 能用。
- `processSessionStartHooks('resume', …)` 在消息流末尾追加由 hook 返回的 `HookResultMessage[]` 数组(`conversationRecovery.ts:568-572`)。

### 4.3 反序列化

`deserializeMessagesWithInterruptDetection`(`conversationRecovery.ts:165-253`)是

```
migratedMessages   = messages.map(migrateLegacyAttachmentTypes)
filteredToolUses   = filterUnresolvedToolUses(migratedMessages)
filteredThinking   = filterOrphanedThinkingOnlyMessages(filteredToolUses)
filteredMessages   = filterWhitespaceOnlyAssistantMessages(filteredThinking)
internalState      = detectTurnInterruption(filteredMessages)
if internalState.kind === 'interrupted_turn':
    append synthetic userMessage("Continue from where you left off.")
    turnInterruptionState = { kind: 'interrupted_prompt', message }
else turnInterruptionState = internalState

if lastRelevant === user:
    inject synthetic assistant "no_response_requested"
return { messages, turnInterruptionState }
```

它给了 SDK 一个钩子用来检测"上一次被外部重启打断"的回合,但终端 REPL 路径并不用这个 `turnInterruptionState`,只是把 messages 直接挂进 UI。

`restoreSkillStateFromMessages` 会从历史消息里还原 `invoked_skills` 附件,以便多次 `/compact` 之后这些 skill 还能在恢复后的状态里看到。

`checkResumeConsistency`(`sessionStorage.ts:2228`)定位最后一条 `turn_duration` checkpoint,比对 `expected` 与 `actual` 并上报 `tengu_resume_consistency_delta`,**纯观测**,不影响恢复结果。

---

## 5. `processResumedConversation` —— 把恢复数据组装成 `ProcessedResume`

`src/utils/sessionRestore.ts:409` 是 `--continue / --resume / 交互选择器 / SDK resume` 等所有恢复路径共用的事后整理函数。

返回值类型:

```
type ProcessedResume = {
    messages: Message[]
    fileHistorySnapshots?: FileHistorySnapshot[]
    contentReplacements?: ContentReplacementRecord[]
    agentName?: string
    agentColor?: AgentColorName
    restoredAgentDef?: AgentDefinition
    initialState: AppState
}
```

主要步骤:

1. **模式对齐**(COORDINATOR_MODE 门控):若 `matchSessionMode(result.mode)` 返回 warning,把它当成一条 system 消息塞进 `messages`。

2. **`switchSession(sid, dirname(transcriptPath))`** —— 切换 `getCurrentSessionId()`、把"asciicast 录音"重命名(`renameRecordingForSession`)指向新 ID、清空 `getSessionFilePointer()`、从 `costTracker` 里调 `restoreCostStateForSession` 还原 token 与花费统计。

3. **fork-session 兜底**(`--fork-session`):保持刚才 `program.startup` 时生成的新 session ID,但把 `contentReplacements` 通过 `recordContentReplacement` 预先写到 JSONL 里,以避免 fork 后的第一次 query 把所有缓存判成 FROZEN。

4. **`restoreSessionMetadata`**:把 `customTitle / tag / agentName / agentColor / mode / worktreeSession / prLink / contextCollapse*` 灌回本地 cache(非 fork 路径包含 worktreeSession)。

5. **`restoreWorktreeForResume`**:若原会话最后处在某个 worktree 里,自动 `process.chdir` 切回去(`result.worktreeSession.worktreePath`),并 `adoptResumedSessionFile()` 把当前 `sessionFile` 指针指向刚才那条 JSONL。这样 `reAppendSessionMetadata` 在 `exit` 时不会因 `sessionFile == null` 而跳过。

6. **CONTEXT_COLLAPSE 门控**:用 `services/contextCollapse/persist.js` 的 `restoreFromEntries` 把 staged snapshot / commits 队列恢复。

7. **`restoreAgentFromSession`**(`sessionStorage.ts`/`sessionRestore.ts`)—— 优先以 `--agent` / settings 里指定的 agent 为主,但若 `result.agentSetting` 存在并且当前命令没有显式覆盖,就把那个 agent 加载进 `mainThreadAgentDefinition`,并写到 `AppState.agent`。

8. **`saveMode(...)`** / **`computeRestoredAttributionState`** / **`computeStandaloneAgentContext`** —— 把"上次会话曾经使用的 mode / attribution 状态 / standalone agent context"写回 AppState。

9. **`updateSessionName(result.agentName)`** —— 通过 `setSessionName` 显式同步当前会话名,以让 `sessionStorage` 的 metadata 在退出时用对的名字重新落到 JSONL。

10. 最后构造 `ProcessedResume`,把经过 hook 注入的 `messages`、文件历史快照、内容替换记录、agent 名/色、恢复后的 `agentDefinitions` 一起打包,在 `main.tsx` 调用 `launchRepl` 时作为 `initialMessages / initialFileHistorySnapshots / initialContentReplacements / initialAgentName / initialAgentColor / mainThreadAgentDefinition` 传入。

---

## 6. `launchRepl` —— 把恢复数据变成一个 Ink 树

`src/replLauncher.tsx:12`:

```
export async function launchRepl(root, appProps, replProps, renderAndRun) {
    const { App }    = await import('./components/App.js')
    const { REPL }   = await import('./screens/REPL.js')
    await renderAndRun(root,
        <App {...appProps}>            // initialState
            <REPL {...replProps} />    // initialMessages, ...
        </App>
    )
}
```

`appProps` 包含 `getFpsMetrics / stats / initialState`(后者来自 `resumeData.initialState`),`replProps` 把 `initialMessages / initialFileHistorySnapshots / initialContentReplacements / initialAgentName / initialAgentColor / mainThreadAgentDefinition` 全部透给 REPL。

`renderAndRun`(`src/interactiveHelpers.ts`) 拿到 `Root`(Ink 的渲染器),`Ink.render(root, <App>…</App>)` 把整个树挂上去。Ink 由 `src/ink/ink.tsx` 暴露:`pause / resume / repaint / unmount`,进入 alt screen、绑定 stdin。

---

## 7. REPL 挂载后的恢复(`src/screens/REPL.tsx`)

### 7.1 Props 与初始 messages

```
src/screens/REPL.tsx:700-744     Props
src/screens/REPL.tsx:747-773     REPL({...})
src/screens/REPL.tsx:1359        const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? [])
```

`messages` 这个 React state 决定了后续 `useLogMessages(messages, ...)` 的入参(`REPL.tsx:4040`),而 `useLogMessages` 会在 mount 后负责**续写**这段历史到当前 sessionId 的 JSONL(因为 `processResumedConversation` 已经 `adoptResumedSessionFile()` 指向了原文件,所以续写是 append 到原文)。

### 7.2 几个恢复相关 hook

| hook/调用 | 位置 | 作用 |
| --- | --- | --- |
| `restoreReadFileState(initialMessages, getOriginalCwd())` | `REPL.tsx:2167-2178` 的 onMount effect | 把 Read 工具的 `lastReadAt / version` 缓存重建,避免用户恢复会话后接着 Read 同一文件时被当成"未读"。 |
| `restoreRemoteAgentTasks` | 同上 effect | 重新挂载远端 agent 的 task。 |
| `useSwarmInitialization(setAppState, initialMessages, …)` | `REPL.tsx:983` | 让 swarm / teammate 入口读过恢复后的消息流,恢复内部计数器。 |
| `useLogMessages(messages, length === initialMessages?.length)` | `REPL.tsx:4040` | 维护 `lastRecordedUuid`、`recordTranscript`、续写历史(只在 mount 阶段停止,以免与 React state 形成循环写入)。 |
| `provisionContentReplacementState(initialMessages, initialContentReplacements)` | `REPL.tsx:1681` | 用恢复出的 `contentReplacements` 重建一个 `Map<toolUseId, ContentReplacementState>`,避免把以前已重写过的内容再发出去。 |
| `haikuTitleAttemptedRef = useRef((initialMessages?.length ?? 0) > 0)` | `REPL.tsx:1308-1310` | 通过 `len > 0` 抑制后续生成 AI 标题的副作用——恢复型会话不应再起手重写标题。 |
| `useProactive / useScheduledTasks` | `REPL.tsx:919-…` | 与 resume 同时跑;若 resume 时钩子的 `SessionStart.resume` 报错,这些 hook 会显示通知。 |
| `<SuspenseBoundary>` 全套环境通知 | `REPL.tsx:920-947` | 提示用户 LSP/MCP 状态变化,但不阻塞。 |

### 7.3 第一次 query 的衔接

`useLogMessages` 的 mount 副作用把 `lastRecordedUuid` 设到初始 messages 的最后一条 UUID,这样用户键入任意输入触发 `processUserInput → query()` 时,`RecordTranscript` 写到的就是从这个 UUID 之后开始的"新内容",不会重复历史。

若用户主动键入之前什么都不做,REPL 维持一个 prompt 行等待输入。这就是 `claude --resume <sid>` 的最终 UX:看到原会话历史,然后像往常一样键入下一条 user prompt 即可。

---

## 8. 交互式选择器路径

### 8.1 何时进入

`main.tsx:3998-4011` 在 `processedResume` 为空时调用 `launchResumeChooser`。三种触发场景:

1. **裸 `claude --resume`**(即 `options.resume === true`)——无 ID、无标题、无路径、无搜索词 → 显示本仓库所有 `LogOption`,可继续按 worktree/branch/tag 筛选。
2. **`claude --resume foo`** —— `foo` 既不是 UUID,也无法做精确标题匹配 → 把 `foo` 当搜索词传给 `LogSelector.initialSearchQuery`,进入预填搜索的选择器。
3. **`claude --from-pr` / `--teleport`(字符串:任务 ID)** —— 在 `main.tsx` 早一步被解析为别的 resume 通道(详见 3655–3832 行)。

### 8.2 `launchResumeChooser`

`src/dialogLaunchers.tsx:117`:

```
export async function launchResumeChooser(root, appProps, worktreePathsPromise, resumeProps) {
    const [worktreePaths, { ResumeConversation }, { App }] =
        await Promise.all([
            worktreePathsPromise,
            import('./screens/ResumeConversation.js'),
            import('./components/App.js'),
        ])
    await renderAndRun(root,
        <App {...appProps}>
            <KeybindingSetup>
                <ResumeConversation {...resumeProps} worktreePaths={worktreePaths} />
            </KeybindingSetup>
        </App>
    )
}
```

`worktreePathsPromise` 由调用方传入,通常是 `getWorktreePaths(getOriginalCwd())`(`main.tsx:4005`)——其结果用来在选择器里"是否跨 worktree 过滤"。

### 8.3 `ResumeConversation`

`src/screens/ResumeConversation.tsx:87`。它把 `App` 内的 state 切换到一个 picker-first 的视图(`<ResumeCommand />`)并自带 `onResume`:

- 当用户选中一条历史 → 调用 `await context.resume?.(sessionId, log, entrypoint)`(`resume.tsx:198`),其中 `context.resume` 是 REPL 通过 `LocalJSXCommandContext` 注入的回调;
- `context.resume` 内部会重新走一次 `loadConversationForResume → processResumedConversation`,然后**就地**调用 `setMessages(...)` 把新历史替换进去(`<LocalJSXCommandContext.setMessages>` 在 REPL/Command 执行流程里被共用);也可以选择 `exit + relaunch`(参考 `restoreSessionStateFromLog` 在 REPL.tsx:1985 的使用,这是 interactive `<ResumeConversation>` 的另一种写法)。

### 8.4 `ResumeCommand` 与 `LogSelector`

`src/commands/resume/resume.tsx:89` 渲染 `<LogSelector />`(`src/components/LogSelector.tsx`)。`LogSelector` 自身内含:

- 标签搜索 / 分支过滤 / worktree 过滤;
- 树形合并(fork 会话被合并为父 + 子节点);
- `agenticSessionSearch`(可在历史 transcript 文本上做语义检索);
- 选中后调用上层 `onSelect(log)`(`resume.tsx:136`),经过 `validateUuid`、`loadFullLog(lite)`、`checkCrossProjectResume`(`src/utils/crossProjectResume.ts`),最终 `onResume(sessionId, fullLog, 'slash_command_picker' | 'slash_command_session_id' | 'slash_command_title')`。

Resume 调用栈其实和 `claude -r <sid>` 是一致的,只是 `entrypoint` 是 `slash_command_*` 而非 `cli_flag`。

### 8.5 跨工程恢复处理

`checkCrossProjectResume` 决定:

- **同 cwd 或同 worktree** —— 直接 resume。
- **跨工程** —— 把"如何在该工程恢复"的命令行(`claude -r <sid>` + 路径)复制到剪切板,提示用户去对应目录执行。
- **同 repo 不同 worktree** —— 通过命令直接把 cwd 切过去再 resume。

---

## 9. `--print` 路径(`src/cli/print.ts`)

`nonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY`(`main.tsx:958`),一旦命中,`main.tsx:3076-3113` 直接 `runHeadless(inputPrompt, ..., { continue, resume, ... })`,不再走 REPL。

`runHeadless`(`src/cli/print.ts:455`)随后调用 `loadInitialMessages`(同文件 4904 行),里头的关键路径与上文一致:

```
if (options.continue) { /* loadConversationForResume(undefined) */ }
else if (options.resume) {
    if (/* options.resume 看起来像 .jsonl */) loadMessagesFromJsonlPath(opts.resume)
    else loadConversationForResume(opts.resume)
}
```

并把恢复出的 `messages` 喂给后续的 `query()` 主循环(`/turn` 流)。这一条流没有 Ink 渲染与 `<REPL>`,只使用 `QueryEngine` / `query()` 的核心:消息体 → prompt → tools → 收尾。

`--resume-session-at <message-id>` 同样在 `loadInitialMessages` 里被消费,只在头无 (`--print`)模式下生效,把恢复历史裁到指定 message(包括)为止。

---

## 10. 时序图(交互式 REPL 主分支)

```
[CLI]   bun run dev → cli.tsx:main()
        └─ if no fast-path → import("../main.jsx").main()
[CLI]   main.tsx:main()
        ├─ init state, settings
        └─ run()
[CLI]   run()
        ├─ new CommanderCommand().option('-r, --resume [value]', ...)
        └─ program.parseAsync(argv)
[CLI]   program.action(prompt, options)  // --resume resolved
        └─ options.resume truthy → enter resume branch (line 3607)
        ├─ clearSessionCaches()
        ├─ maybeSessionId = validateUuid(options.resume)
        ├─ if non-UUID string → searchSessionsByCustomTitle(value, exact)
        ├─ if (Bun external/internal = 'ant') && non-UUID:
        │     ├─ parseCcshareId → loadCcshare → ...
        │     └─ else resolve(path) → loadTranscriptFromFile → ...
        ├─ if maybeSessionId:
        │     result = await loadConversationForResume(matchedLog ?? sessionId, undefined)
        │     processedResume = await processResumedConversation(result, opts, resumeContext)
        └─ if processedResume → launchRepl(...)
          else               → launchResumeChooser(...)

[LOAD]  loadConversationForResume()
        ├─ getLastSessionLog(sid)
        │     └─ loadSessionFile(sid) → loadTranscriptFile(<projDir>/<sid>.jsonl)
        ├─ isLiteLog? loadFullLog(log)
        ├─ copyPlanForResume(log, sid)
        ├─ copyFileHistoryForResume(log)
        ├─ restoreSkillStateFromMessages(messages)
        ├─ deserializeMessagesWithInterruptDetection(messages)
        └─ processSessionStartHooks('resume', {sessionId})  // 追加 hook 消息

[RESTORE] processResumedConversation()
        ├─ (COORDINATOR_MODE) matchSessionMode
        ├─ switchSession(sid, dirname(transcriptPath))
        ├─ renameRecordingForSession()
        ├─ restoreCostStateForSession(sid)
        ├─ restoreSessionMetadata(...)
        ├─ restoreWorktreeForResume(worktreeSession)
        ├─ adoptResumedSessionFile()
        ├─ (CONTEXT_COLLAPSE) restoreFromEntries(commits, snapshot)
        ├─ restoreAgentFromSession(agentSetting, ...)
        ├─ computeRestoredAttributionState / computeStandaloneAgentContext
        ├─ updateSessionName(agentName)
        └─ return ProcessedResume { messages, fileHistorySnapshots, contentReplacements, agentName, agentColor, restoredAgentDef, initialState }

[RENDER] launchRepl()
        ├─ import App / REPL
        └─ renderAndRun(root, <App initialState><REPL initialMessages=... initialFileHistorySnapshots=... initialContentReplacements=... initialAgentName=... initialAgentColor=... mainThreadAgentDefinition=... /></App>)

[UI]    REPL:mount
        ├─ useState<MessageType[]>(initialMessages ?? [])            // 把整段历史塞进 React state
        ├─ mount-effect:
        │     ├─ restoreReadFileState(initialMessages, getOriginalCwd())
        │     └─ restoreRemoteAgentTasks(...)
        ├─ useSwarmInitialization(setAppState, initialMessages, ...)
        ├─ provisionContentReplacementState(initialMessages, initialContentReplacements)
        ├─ haikuTitleAttemptedRef.current = initialMessages.length > 0    // 别再重写标题
        └─ useLogMessages(messages, length === initialMessages?.length)    // 不重写历史,只续写

[USER]  用户键入下一条 prompt
        └─ processUserInput → query() → RecordTranscript(append)
                                  → read JSONL(原 sid 文件),追加新消息
```

---

## 11. 关键状态/目录速查

| 名称 | 出处 | 摘要 |
| --- | --- | --- |
| `~/.claude/projects/<encoded-cwd>/<sid>.jsonl` | `getSessionProjectDir()` / `loadSessionFile` | 同 cwd 下所有对话的 NDJSON 存储,每行一条 `Entry`(`Message|Summary|CustomTitle|FileHistorySnapshot|AttributionSnapshot|WorktreeState|…`)。|
| `~/.claude/projects/<encoded-cwd>/sessions-index.json` | `loadMessageLogsProgressive` | 进度加载时的索引,体积小时直接列出;有索引时跳过 JSONL 摘要阶段(`isLiteLog` 才会触发)。 |
| `~/.claude/plans/<slug>/` | `copyPlanForResume` | plan 文件 resume 后从 src session 拷贝到当前 slug。 |
| `~/.claude/file-history/<sid>/` | `copyFileHistoryForResume` | 文件历史快照 resume 时按需拷出。 |
| `~/.claude/sessions/<sid>/recording.cast` | `renameRecordingForSession` | asciicast 录制,resume 时按新 ID 重命名以供 `/share`。 |
| `AppState`(`src/state/AppState.tsx`) | `setAppState` | 全局 zustand 风格 store,挂载 `initialState`,resume 路径通过 `ProcessedResume.initialState` 注入 agentDefinitions / mainLoopModel / attribution / standaloneAgentContext 等。 |
| `ProcessedResume`(`src/utils/sessionRestore.ts:276`) | 工厂方法 | `messages / fileHistorySnapshots / contentReplacements / agentName / agentColor / restoredAgentDef / initialState` —— 把磁盘上的 LogOption 与 AppState 状态桥接起来。 |
| `messagesRef`(`REPL.tsx:1360`) | 复位信号 | Ref 与 `messages` state 双写,保证 race 下的 reducer 可见性。 |

---

## 12. 已知陷阱 / 易踩点

1. **UUID vs 标题**:`-r foo bar` 中空格分词,Commander 仅取 `foo` 作为 value;若要传入 custom title 需引号包住。`validateUuid()` 仅匹配标准 UUID,不接受短链。
2. **`--fork-session` 与 worktree**:fork 不接管原 worktree,因此 `restoreWorktreeForResume` 被跳过(`ProcessedResume` 不再 `cd`);但会拿到原始 `contentReplacements` 以保证 cache 命中。
3. **`--resume-session-at` 仅在 `--print` 模式**:头无模式下,这一选项通过 `loadInitialMessages` 裁剪;REPL 模式下传了也不会被消费,而是会在打印 help 中标识 `hideHelp()`。
4. **lite log**:resume 前若只看摘要就以 `isLite === true` 返回,`loadFullLog` 才是把全文加载到内存的入口;中间任一处打断(断信号、错误)就会回到 picker。
5. **`SessionStart` hook**:resume 会以 `'resume'` trigger 调用 `processSessionStartHooks`,其返回的 `HookResultMessage[]` 会追加在 messages 最末;任何一段 hook 抛错都会冒泡,导致整个 resume 失败。
6. **跨工程恢复**:`checkCrossProjectResume` 在 `slash_command_picker` 路径会主动切 cd;CLI 的 `claude -r <sid>` 不做这件事,只在 `cross-project` 时给出复制到剪贴板的命令提示。
7. **fork 与 isolated transcript**:`fork` 不复用源 sid 文件,而是新建一个 JSONL;`--continue` 在 fork 路径下,`recordContentReplacement` 必须先于 `query()` 跑完,否则第一次对话所有 tool_use_id 都会被分类为 FROZEN。
8. **`sourceJsonlFile` 分流**:`loadConversationForResume` 接受第二参数,仅 `print.ts`(`-r path.jsonl`)使用;REPL 路径总是 `undefined`,所有路径输入都已经在 main.tsx 里被 `loadTranscriptFromFile` 提前归一化。

---

## 13. 附录:相关代码锚点

```
src/entrypoints/cli.tsx                        cli.tsx:main,fast-path
src/main.tsx:1130                              run(): Commander 程序注册
src/main.tsx:1235                              --resume 与 --fork-session option
src/main.tsx:1253                              program.action 默认 handler
src/main.tsx:3076-3113                         -p/--print 路径(包含 options.resume)
src/main.tsx:3607-3957                         resume 分支主流程
src/utils/conversationRecovery.ts:460          loadConversationForResume
src/utils/conversationRecovery.ts:165          deserializeMessagesWithInterruptDetection
src/utils/sessionStorage.ts:3875               getLastSessionLog
src/utils/sessionStorage.ts:4079               loadSameRepoMessageLogs
src/utils/sessionStorage.ts:2228               checkResumeConsistency
src/utils/sessionRestore.ts:276                ProcessedResume 类型
src/utils/sessionRestore.ts:409                processResumedConversation
src/replLauncher.tsx:12                        launchRepl
src/dialogLaunchers.tsx:117                    launchResumeChooser
src/screens/ResumeConversation.tsx:87          ResumeConversation
src/commands/resume/resume.tsx:89              ResumeCommand
src/commands/resume/resume.tsx:195             /resume command 的 call()
src/components/LogSelector.tsx                 通用列表/搜索框
src/screens/REPL.tsx:700-744                   Props(initialMessages / initialFileHistorySnapshots / initialContentReplacements / initialAgentName / initialAgentColor)
src/screens/REPL.tsx:1359                      useState<MessageType[]>(initialMessages ?? [])
src/screens/REPL.tsx:2164-2178                 onMount restoreReadFileState + restoreRemoteAgentTasks
src/screens/REPL.tsx:1985                      restoreSessionStateFromLog(interactive 分支使用)
src/screens/REPL.tsx:4040                      useLogMessages
src/cli/print.ts:455                           runHeadless
src/cli/print.ts:4904                          loadInitialMessages
src/types/command.ts:93                        LocalJSXCommandContext.resume
src/types/command.ts:100                       ResumeEntrypoint
src/utils/crossProjectResume.ts                checkCrossProjectResume
```

---

## 14. 小结

`claude --resume` 的核心是 **"把磁盘上的 NDJSON 会话快照,在不破坏当前 session ID(除非 fork)的前提下,塞回内存并渲染到 UI"**:

1. **定位**:CLI 参数 → UUID / 标题 / ccshare / 路径 / 搜索词;
2. **加载**:`loadConversationForResume` 把 LogOption 反序列化、补上 hook、消解中断、补 sentinel;
3. **状态机切换**:`processResumedConversation` 在 `bootstrap/state` 里 `switchSession`、恢复 cost/worktree/agent/mode/attribution,把 `--resume` 期间 AI 已经看过的历史重新贴回新一次启动的上下文;
4. **挂载**:`launchRepl` → `<App><REPL initialMessages=...>`;REPL 用 `useState` 直接以恢复的消息作为初始 messages,然后由 `useLogMessages` 只把后续新增的部分续写到 JSONL。
5. 若没有"可立即定位"的 ID/标题/路径,`launchResumeChooser` 同样复用 REPL 的命令回调,选中后再走一遍同样的加载+状态切换流程,只是入口换成了 interactive path。

`claude --resume` 实际上是 "REPL 已经启动后,把磁盘的 transcript 当作 `initialMessages` 一次性注入",因此不会破坏 REPL 已经创建好的进程级 / 终端级状态(本地 agent、sub-task、tool permission),只会改写消息、cost、metadata、worktree 这几项可热替换的数据。
