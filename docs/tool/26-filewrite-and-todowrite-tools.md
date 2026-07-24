# FileWriteTool 与 TodoWriteTool 源码解析

> 源码位置：
> - `src/tools/FileWriteTool/FileWriteTool.ts`（435 行）
> - `src/tools/TodoWriteTool/TodoWriteTool.ts`（115 行）
>
> 关联文档：[`docs/11-tool-execution.md`](11-tool-execution.md) · [`docs/18-task-planning.md`](../agent/18-task-planning.md) · [`docs/23-design-and-core-modules.md`](../architecture/23-design-and-core-modules.md)

这两个工具属于 Claude Code 最常用、也最能体现 "**写文件 + 跟踪任务**" 这两条核心交互路径。FileWriteTool 负责把模型给出的内容落到磁盘，TodoWriteTool 负责把模型的"打算做什么"显式化给用户并固化在 AppState 里。

本文按"职责 → schema → 权限/校验 → call 内部流程 → 与外部系统的协作"顺序逐一拆解。

---

## 目录

- [FileWriteTool —— 全量覆写文件](#filewritetool--全量覆写文件)
  1. [职责与边界](#1-职责与边界)
  2. [Schema 与输入规范化](#2-schema-与输入规范化)
  3. [权限判定链](#3-权限判定链)
  4. [validateInput 的 4 重防线](#4-validateinput-的-4-重防线)
  5. [call 内部的执行顺序](#5-call-内部的执行顺序)
  6. [辅助系统联动](#6-辅助系统联动)
  7. [返回与 UI 渲染](#7-返回与-ui-渲染)
  8. [设计取舍与坑点](#8-设计取舍与坑点)
- [TodoWriteTool —— 会话任务清单](#todowritetool--会话任务清单)
  1. [职责与定位](#1-职责与定位-1)
  2. [V1 vs V2 切换](#2-v1-与-v2-切换)
  3. [Schema 与零 UI 设计](#3-schema-与零-ui-设计)
  4. [权限与分类器适配](#4-权限与分类器适配)
  5. [call 的核心逻辑](#5-call-的核心逻辑)
  6. [Verification Nudge 机制](#6-verification-nudge-机制)
  7. [Tool Result 回流格式](#7-tool-result-回流格式)
- [两者在 queryLoop 中的协作](#两者在-queryloop-中的协作)
- [附录：与其他工具的对比](#附录与其他工具的对比)

---

# FileWriteTool —— 全量覆写文件

## 1. 职责与边界

FileWriteTool 的职责**严格限定**为：

> 把一段完整文本写入**单个文件**，覆盖或新建。

它不做：
- 不做局部替换（那是 `FileEditTool` 的活）；
- 不做多次原子替换（那是 `MultiEditTool` 的活）；
- 不做二进制写入（编码假设 utf8，含 `LF` 强制行尾）。

源码注释（`FileWriteTool.ts:300-304`）明确说明 **写策略**：模型发的 `content` 是字面意义，不做隐式 line-ending 改写：

> Write is a full content replacement — the model sent explicit line endings in `content` and meant them. Do not rewrite them. Previously we preserved the old file's line endings (or sampled the repo via ripgrep for new files), which silently corrupted e.g. bash scripts with \r on Linux when overwriting a CRLF file or when binaries in cwd poisoned the repo sample.

## 2. Schema 与输入规范化

```ts
// FileWriteTool.ts:56-66
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe(
      'The absolute path to the file to write (must be absolute, not relative)'
    ),
    content: z.string().describe('The content to write to the file'),
  }),
)
```

**严格对象**（`strictObject`）—— 任何未声明字段直接报错，避免模型幻觉出的字段被静默忽略。

### 2.1 输出 Schema

```ts
// FileWriteTool.ts:67-89
const outputSchema = lazySchema(() =>
  z.object({
    type: z.enum(['create', 'update']),
    filePath: z.string(),
    content: z.string(),
    structuredPatch: z.array(hunkSchema()),
    originalFile: z.string().nullable(),  // null 表示新文件
    gitDiff: gitDiffSchema().optional(),
  }),
)
```

`structuredPatch` 是 UI 渲染 diff 用的（`hunkSchema` 来自 `FileEditTool/types.ts`）；`gitDiff` 仅在 remote 模式下计算。

### 2.2 输入预处理：`backfillObservableInput`

```ts
// FileWriteTool.ts:125-131
backfillObservableInput(input) {
  // hooks.mdx documents file_path as absolute; expand so hook allowlists
  // can't be bypassed via ~ or relative paths.
  if (typeof input.file_path === 'string') {
    input.file_path = expandPath(input.file_path)
  }
}
```

**关键安全设计**：`expandPath` 把 `~` 和相对路径解析成绝对路径，**在 hook 看到 input 之前**完成。这样用户的 hook allowlist（如 `Write(./src/**)`）不会被 `~/src/foo.ts` 或 `./src/foo.ts` 绕过。

注释解释原因：

> expand so hook allowlists can't be bypassed via `~` or relative paths.

注意：原 API-bound input **不被 mutate**（`Tool.ts:481-486` 注释明确 `Not re-applied when a hook/permission returns a fresh updatedInput`），所以 `backfillObservableInput` 不会破坏 prompt cache。

## 3. 权限判定链

### 3.1 `checkPermissions`

```ts
// FileWriteTool.ts:135-142
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkWritePermissionForTool(
    FileWriteTool,
    input,
    appState.toolPermissionContext,
  )
}
```

委托给 `utils/permissions/filesystem.ts:checkWritePermissionForTool`（4 层防御的核心实现）：

1. **Deny 规则** → 一票否决
2. **Allow 规则** → 自动放行
3. **路径白名单**（`additionalWorkingDirectories`）
4. **未匹配** → 返回 `{ behavior: 'ask' }`，弹窗让用户决定

### 3.2 `preparePermissionMatcher`

```ts
// FileWriteTool.ts:132-134
async preparePermissionMatcher({ file_path }) {
  return pattern => matchWildcardPattern(pattern, file_path)
}
```

让 hook 的 `if` 条件（如 `"Write(/tmp/**)"`）能基于**完整文件路径**而不是工具名做匹配。具体匹配算法在 `utils/permissions/shellRuleMatching.ts`。

## 4. validateInput 的 4 重防线

`validateInput` 在权限检查之后、执行 `call` 之前运行（`Tool.ts:489-492`）。FileWriteTool 实现了 4 道防线：

### 4.1 第一道：团队记忆密钥防护

```ts
// FileWriteTool.ts:153-160
const secretError = checkTeamMemSecrets(fullFilePath, content)
if (secretError) {
  return { result: false, message: secretError, errorCode: 0 }
}
```

防止模型把 API key / 密码写入团队共享的 memory 文件。

### 4.2 第二道：路径级 deny 规则

```ts
// FileWriteTool.ts:163-177
const denyRule = matchingRuleForInput(
  fullFilePath,
  appState.toolPermissionContext,
  'edit',
  'deny',
)
if (denyRule !== null) {
  return {
    result: false,
    message: 'File is in a directory that is denied by your permission settings.',
    errorCode: 1,
  }
}
```

比 `checkPermissions` 更细致 —— 在 `edit` 命名空间下查路径匹配规则。注意 `errorCode: 1` 用于诊断统计。

### 4.3 第三道：UNC 路径安全绕过

```ts
// FileWriteTool.ts:179-184
// SECURITY: Skip filesystem operations for UNC paths to prevent NTLM credential leaks.
// On Windows, fs.existsSync() on UNC paths triggers SMB authentication which could
// leak credentials to malicious servers. Let the permission check handle UNC paths.
if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
  return { result: true }
}
```

**这是个 Windows 安全设计**：UNC 路径（如 `\\server\share\...`）会让 `fs.existsSync` 触发 SMB 认证，可能泄露 NTLM 凭据。这里**直接放行 validateInput**，让权限系统的 ask 弹窗处理 —— 用户确认后才会走 `call` 实际 IO。

### 4.4 第四道：必须先读过 + mtime 检查

```ts
// FileWriteTool.ts:186-219
const fs = getFsImplementation()
let fileMtimeMs: number
try {
  const fileStat = await fs.stat(fullFilePath)
  fileMtimeMs = fileStat.mtimeMs
} catch (e) {
  if (isENOENT(e)) {
    return { result: true }   // 新文件 → 通过
  }
  throw e
}

const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false,
    message: 'File has not been read yet. Read it first before writing to it.',
    errorCode: 2,
  }
}

const lastWriteTime = Math.floor(fileMtimeMs)
if (lastWriteTime > readTimestamp.timestamp) {
  return {
    result: false,
    message: 'File has been modified since read, either by the user or by a linter. Read it again before attempting to write it.',
    errorCode: 3,
  }
}

return { result: true }
```

**两道独立校验**：

1. **必须先 Read 过**（`readFileState.get` 返回 undefined 或 `isPartialView`）—— 防止模型盲写覆盖现有内容；
2. **mtime 守门**：用户 / linter 在模型读过之后又改了文件，必须重新读。

注释（`FileWriteTool.ts:208-210`）解释 mtime 复用原因：

> Reuse mtime from the stat above — avoids a redundant statSync via getFileModificationTime. The readTimestamp guard above ensures this block is always reached when the file exists.

## 5. call 内部的执行顺序

`call` 是 FileWriteTool 的核心，435 行中占了 200 行。严格按**阶段顺序**：

```
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: 路径解析与 Skill 发现（L57-66）                          │
│   - expandPath(file_path) → 全路径                                │
│   - dirname → 父目录                                              │
│   - discoverSkillDirsForPaths + activateConditionalSkillsForPaths │
├─────────────────────────────────────────────────────────────────┤
│ Stage 2: 副作用前置 hook（L74-83）                                │
│   - diagnosticTracker.beforeFileEdited                            │
│   - mkdir parent dir（避免 lazy-mkdir-on-ENOENT 触发虚假错误）    │
│   - fileHistoryTrackEdit（备份旧版本，支持 /rewind）               │
├─────────────────────────────────────────────────────────────────┤
│ Stage 3: 原子读写（L86-117）                                       │
│   ⛔ 不要在这段插入 async 操作                                     │
│   - readFileSyncWithMetadata                                      │
│   - 检查 mtime 是否变化（再次！）                                 │
│   - 若变化且非 partial read，比对 CRLF-normalized 内容            │
│   - 变化 → throw FILE_UNEXPECTEDLY_MODIFIED_ERROR                │
│   - 未变 → writeTextContent                                       │
├─────────────────────────────────────────────────────────────────┤
│ Stage 4: 副作用后置 hook（L120-145）                              │
│   - LSP didChange / didSave 通知                                  │
│   - VSCode 通知（diff view）                                      │
│   - readFileState.set（更新 read timestamp）                       │
│   - logEvent tengu_write_claudemd（如写 CLAUDE.md）               │
│   - 可选 fetchSingleFileGitDiff（remote 模式）                     │
├─────────────────────────────────────────────────────────────────�
│ Stage 5: 输出与埋点（L148-186）
│   - getPatchForDisplay 生成 diff
│   - countLinesChanged 计数
│   - logFileOperation
│   - return { data, type: 'create'|'update' }
```

### 5.1 Stage 2 关键注释

**`mkdir` 必须 OUTSIDE atomic 段**（`FileWriteTool.ts:249-254`）：

> Ensure parent directory exists before the atomic read-modify-write section. Must stay OUTSIDE the critical section below (a yield between the staleness check and writeTextContent lets concurrent edits interleave), and BEFORE the write (lazy-mkdir-on-ENOENT would fire a spurious tengu_atomic_write_error inside writeFileSyncAndFlush_DEPRECATED before ENOENT propagates back).

### 5.2 Stage 3 原子性保证

源码注释（`FileWriteTool.ts:266-267`）非常明确：

> Load current state and confirm no changes since last read.
> **Please avoid async operations between here and writing to disk to preserve atomicity.**

这是 Rust-like 的 critical section：read → check → write 必须是同步三连。如果中间 `await` 任何东西，**另一个并发 Edit 可能插队**，导致 "模型基于 A 状态写的，覆盖了 B 状态" 的 lost-update 问题。

Windows 场景的额外校验（`FileWriteTool.ts:286-293`）：

```ts
const isFullRead =
  lastRead &&
  lastRead.offset === undefined &&
  lastRead.limit === undefined
// meta.content is CRLF-normalized — matches readFileState's normalized form.
if (!isFullRead || meta.content !== lastRead.content) {
  throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
}
```

注释解释：

> Timestamp indicates modification, but on Windows timestamps can change without content changes (cloud sync, antivirus, etc.). For full reads, compare content as a fallback to avoid false positives.

### 5.3 写入策略

```ts
// FileWriteTool.ts:305
writeTextContent(fullFilePath, content, enc, 'LF')
```

编码 = 旧文件编码（默认 utf8），**强制 LF 行尾**。`writeTextContent` 的实现里会原子地写临时文件 + rename。

## 6. 辅助系统联动

### 6.1 LSP 集成

```ts
// FileWriteTool.ts:308-326
const lspManager = getLspServerManager()
if (lspManager) {
  clearDeliveredDiagnosticsForFile(`file://${fullFilePath}`)
  lspManager.changeFile(fullFilePath, content).catch(...)
  lspManager.saveFile(fullFilePath).catch(...)
}
```

两个 LSP 通知：
- `didChange`：内容已改 → 触发 TS 服务器重新计算类型；
- `didSave`：已落盘 → 触发 server 跑 lint / 类型检查 → diagnostics 回流。

`clearDeliveredDiagnosticsForFile` 清除上次缓存的诊断，避免 UI 显示旧错误。

### 6.2 VSCode 集成

```ts
// FileWriteTool.ts:329
notifyVscodeFileUpdated(fullFilePath, oldContent, content)
```

VSCode 端收到通知后刷新 diff view（在 Source Control 面板里画红色/绿色块）。

### 6.3 Skill 发现

```ts
// FileWriteTool.ts:233-245
const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd)
if (newSkillDirs.length > 0) {
  for (const dir of newSkillDirs) {
    dynamicSkillDirTriggers?.add(dir)
  }
  addSkillDirectories(newSkillDirs).catch(() => {})
}
activateConditionalSkillsForPaths([fullFilePath], cwd)
```

每次写文件都触发一次 Skill 扫描：发现新目录 + 激活条件性 Skill（如 "写 `*test*` 时激活 testing skill"）。

### 6.4 File History（`/rewind` 支撑）

```ts
// FileWriteTool.ts:255-264
if (fileHistoryEnabled()) {
  await fileHistoryTrackEdit(
    updateFileHistoryState,
    fullFilePath,
    parentMessage.uuid,
  )
}
```

**关键注释**（`FileWriteTool.ts:255-258`）：

> Backup captures pre-edit content — safe to call before the staleness check (idempotent v1 backup keyed on content hash; if staleness fails later we just have an unused backup, not corrupt state).

写之前备份旧内容到 `<session>/file-history/<hash>.json`，用户 `/rewind` 命令可回滚。

### 6.5 git diff（仅 remote 模式）

```ts
// FileWriteTool.ts:344-357
if (
  isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_quartz_lantern', false)
) {
  const startTime = Date.now()
  const diff = await fetchSingleFileGitDiff(fullFilePath)
  if (diff) gitDiff = diff
  logEvent('tengu_tool_use_diff_computed', {
    isWriteTool: true,
    durationMs: Date.now() - startTime,
    hasDiff: !!diff,
  })
}
```

remote 模式（Claude.ai web / mobile）+ `tengu_quartz_lantern` GrowthBook 启用时计算真实 git diff，让用户在网页端看到 git diff 而不是 mock diff。

### 6.6 CLAUDE.md 写入埋点

```ts
// FileWriteTool.ts:340-342
if (fullFilePath.endsWith(`${sep}CLAUDE.md`)) {
  logEvent('tengu_write_claudemd', {})
}
```

专门埋点 —— 写 CLAUDE.md 是高信号事件（修改项目规则），用于统计和回溯。

## 7. 返回与 UI 渲染

### 7.1 数据返回

```ts
// FileWriteTool.ts:359-393（update 分支）
if (oldContent) {
  const patch = getPatchForDisplay({...})
  const data = {
    type: 'update' as const,
    filePath: file_path,
    content,
    structuredPatch: patch,
    originalFile: oldContent,
    ...(gitDiff && { gitDiff }),
  }
  countLinesChanged(patch)
  logFileOperation({...})
  return { data }
}
```

- `type: 'update'` 或 `'create'` 由 `oldContent` 是否为 null 决定；
- `structuredPatch` 用于 UI 渲染 diff；
- `originalFile` 给 FileEdit 的 `getPatchForDisplay` 提供对照；
- `countLinesChanged` 累积 session 级别的行变更数（analytics 用）。

### 7.2 Tool Result 回流给模型

```ts
// FileWriteTool.ts:418-433
mapToolResultToToolResultBlockParam({ filePath, type }, toolUseID) {
  switch (type) {
    case 'create':
      return { tool_use_id: toolUseID, type: 'tool_result',
        content: `File created successfully at: ${filePath}` }
    case 'update':
      return { tool_use_id: toolUseID, type: 'tool_result',
        content: `The file ${filePath} has been updated successfully.` }
  }
}
```

**极简文本**给模型 —— 不让模型看到自己的 diff（避免自我确认偏差），只让它知道"成功"。

### 7.3 extractSearchText 设计

```ts
// FileWriteTool.ts:146-152
extractSearchText() {
  // Transcript render shows either content (create, via HighlightedCode)
  // or a structured diff (update). The heuristic's 'content' allowlist key
  // would index the raw content string even in update mode where it's NOT
  // shown — phantom. Under-count: tool_use already indexes file_path.
  return ''
}
```

返回空串 —— transcript 搜索不索引这个工具的输出，因为 update 模式下显示的是 diff 而不是 raw content，避免"phantom"（声称有但没渲染）。

## 8. 设计取舍与坑点

### 8.1 强制 LF 行尾

`writeTextContent(fullFilePath, content, enc, 'LF')` —— 不论原文件是 CRLF 还是 CR，统一写 LF。

**取舍**：可能改变 Windows-only 项目的行尾约定，但避免 "覆盖 bash 脚本时混入 \r 导致脚本坏掉" 的真实 bug。

### 8.2 不在 `call` 内重新读文件做完整比对

只在 **mtime 变化且非 partial read** 时做内容比对。partial read（带 offset/limit）不做内容比对 —— 因为 partial read 拿到的不是完整文件，无法 diff。

### 8.3 vsync / antivirus 不一致

Windows 上 mtime 可能因为云同步或杀毒软件而无内容修改地变化。FileWriteTool 通过 partial/full read 区分 + content 比对降级避免误报。

### 8.4 不阻塞 staging / commit

FileWriteTool 不调 `git add` / `git commit` —— 那是用户的活。工具层面只负责"文件已写入磁盘"。

---

# TodoWriteTool —— 会话任务清单

## 1. 职责与定位

TodoWriteTool 维护一个**当前会话的任务清单**，把模型"打算做什么"显式化：

- UI 层：底部状态栏 + 任务列表面板；
- 模型层：作为提示工程的一部分，让模型在长任务中保持结构化自我跟踪；
- 持久化：在 AppState 里，session 终止时不写盘（仅 transcript 里有工具调用记录）。

**关键定位**：TodoWriteTool 不替代 TaskCreate/TaskUpdate（V2 系统），它只是 V1 的轻量 todo list。

## 2. V1 与 V2 切换

```ts
// TodoWriteTool.ts:52-54
isEnabled() {
  return !isTodoV2Enabled()
}
```

`isTodoV2Enabled()` 检查 GrowthBook / env 配置 —— 启用 V2 时 TodoWriteTool **自动禁用**，由 `TaskCreate` / `TaskGet` / `TaskUpdate` / `TaskList` 四个工具替代。

V2 工具的差异：
- 支持 `blockedBy` / `blocks` 依赖关系；
- 任务可分配给特定 agent（`owner` 字段）；
- 支持 `metadata` 自定义键值；
- 持久化到文件而非 AppState；
- 跨 subagent 协作。

## 3. Schema 与零 UI 设计

### 3.1 输入 Schema

```ts
// TodoWriteTool.ts:13-17
const inputSchema = lazySchema(() =>
    z.strictObject({
        todos: TodoListSchema().describe('The updated todo list'),
    }),
)
```

`TodoListSchema`（`utils/todo/types.ts`）定义每条 todo 的形状：
```ts
{
  status: 'pending' | 'in_progress' | 'completed'
  content: string
  activeForm: string  // 进行中时 spinner 显示
}
```

### 3.2 零 UI 设计

```ts
// TodoWriteTool.ts:49-53
userFacingName() {
  return ''
},
shouldDefer: true,
isEnabled() {
  return !isTodoV2Enabled()
},
renderToolUseMessage() {
  return null
},
```

- `userFacingName() => ''` —— 不显示工具名；
- `renderToolUseMessage() => null` —— 调用时不画 UI 元素；
- `shouldDefer: true` —— 当 ToolSearch 启用时可延迟加载。

**原因**：todo list 的 UI 由专用的"任务面板"组件渲染（在 REPL 底部状态栏），不需要每个 `TodoWrite` 调用都画一个 tool_use block。

## 4. 权限与分类器适配

```ts
// TodoWriteTool.ts:58-61
async checkPermissions(input) {
  // No permission checks required for todo operations
  return {behavior: 'allow', updatedInput: input}
},
```

todo 操作**永远自动放行** —— 不写文件、不跑命令、不访问网络。

```ts
// TodoWriteTool.ts:55-57
toAutoClassifierInput(input) {
  return `${input.todos.length} items`
}
```

**关键设计**：返回 `count` 而不是 todo 内容。auto mode classifier 评估权限时不需要看 todo 详情（避免泄漏），只看数量级用于日志。

## 5. call 的核心逻辑

```ts
// TodoWriteTool.ts:65-103
async call({todos}, context) {
  const appState = context.getAppState()
  const todoKey = context.agentId ?? getSessionId()
  const oldTodos = appState.todos[todoKey] ?? []
  const allDone = todos.every(_ => _.status === 'completed')
  const newTodos = allDone ? [] : todos  // ← 关键：全部完成 → 清空列表

  let verificationNudgeNeeded = false
  if (
    feature('VERIFICATION_AGENT') &&
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
    !context.agentId &&                  // 只对主线程
    allDone &&
    todos.length >= 3 &&
    !todos.some(t => /verif/i.test(t.content))
  ) {
    verificationNudgeNeeded = true
  }

  context.setAppState(prev => ({
    ...prev,
    todos: {
      ...prev.todos,
      [todoKey]: newTodos,
    },
  }))

  return {
    data: { oldTodos, newTodos: todos, verificationNudgeNeeded }
  }
}
```

### 5.1 todoKey 的两重含义

```ts
const todoKey = context.agentId ?? getSessionId()
```

- **主线程**：用 `sessionId`，所有 todos 在同一个会话里共享；
- **subagent**：用 `agentId`，每个 subagent 有独立的 todo list，互不污染。

### 5.2 全部完成 → 清空

```ts
const allDone = todos.every(_ => _.status === 'completed')
const newTodos = allDone ? [] : todos
```

**这是一个反直觉的设计**：当所有 todo 都是 `completed`，实际写入 AppState 的**是空数组**，而不是"完整完成的列表"。

**为什么？** 让 UI 不再显示已完成的任务列表，避免视觉噪声。模型下一次 TodoWrite 调用又会重新传完整列表（包括已完成的）—— 在工具的输入上仍能 diff 出 `oldTodos` vs `newTodos`。

### 5.3 主线程判断

```ts
!context.agentId
```

`context.agentId` 为 `undefined` → 主线程；否则是 subagent。`verificationNudgeNeeded` **只在主线程**触发 —— subagent 的 todo 完结不需要 verifier。

## 6. Verification Nudge 机制

源码注释（`TodoWriteTool.ts:73-75`）解释了动机：

> Structural nudge: if the main-thread agent is closing out a 3+ item list and none of those items was a verification step, append a reminder to the tool result. Fires at the exact loop-exit moment where skips happen ("when the last task closed, the loop exited").

**判定条件**（4 个 AND）：
1. `feature('VERIFICATION_AGENT')` 编译时启用；
2. `getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)` GrowthBook A/B 开启；
3. `!context.agentId`（主线程）；
4. `allDone && todos.length >= 3 && !todos.some(t => /verif/i.test(t.content))`（≥3 个任务全部完成，且没有任何任务标题含 "verif"）。

满足时 `mapToolResultToToolResultBlockParam` 追加提示：

```ts
// TodoWriteTool.ts:104-114
mapToolResultToToolResultBlockParam({verificationNudgeNeeded}, toolUseID) {
  const base = `Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable`
  const nudge = verificationNudgeNeeded
    ? `\n\nNOTE: You just closed out 3+ tasks and none of them was a verification step. Before writing your final summary, spawn the verification agent (subagent_type="${VERIFICATION_AGENT_TYPE}"). You cannot self-assign PARTIAL by listing caveats in your summary — only the verifier issues a verdict.`
    : ''
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: base + nudge,
  }
}
```

**这是提示，不是强制** —— 模型可以忽略。但 `VERIFICATION_AGENT_TYPE` 强制让 verifier 发 verdict，避免模型用 "看起来还行" 的自我评语掩盖问题。

## 7. Tool Result 回流格式

```ts
// TodoWriteTool.ts:104-114
const base = `Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable`
const nudge = verificationNudgeNeeded
  ? `\n\nNOTE: ...`  // 见 §6
  : ''
return {
  tool_use_id: toolUseID,
  type: 'tool_result',
  content: base + nudge,
}
```

注意 model-facing 的 tool_result **不包含 todo 详情**（避免泄漏 + 让模型依赖自己的 prompt 上下文）。模型看到的只是 "成功" 提示，可选的 nudge。

---

# 两者在 queryLoop 中的协作

下面以"用户要求生成网页版贪吃蛇"为场景，展示两个工具的协作时序：

```
T=0    queryLoop 第一轮
       模型推理 → "先用 TodoWrite 拆解任务"
       ↓
       tool_use: TodoWrite({
         todos: [
           { content: "规划技术栈与架构", status: "in_progress", activeForm: "规划技术栈" }
           { content: "初始化 Vite + React + TS + Tailwind 项目", status: "pending", activeForm: "..." }
           ...7 项
         ]
       })
       ↓
       runToolUse → TodoWriteTool.call
       ├─ todoKey = sessionId
       ├─ oldTodos = [] (空)
       ├─ allDone = false
       ├─ verificationNudgeNeeded = false (todos[0].status != 'completed')
       └─ setAppState({ todos: { [sessionId]: [...7项] } })
       ↓
       UI 底部状态栏出现 7 项任务列表，第一项显示 spinner

T=3s   模型继续 → "跑 npm create vite@latest"
       tool_use: Bash(...)
       ↓
       权限弹窗，用户选 Yes → 执行

T=10s  模型推进 todo
       tool_use: TodoWrite({
         todos: [
           { content: "规划技术栈与架构", status: "completed", activeForm: "..." }
           { content: "初始化 Vite + React + TS + Tailwind 项目", status: "in_progress", activeForm: "..." }
           ...7 项 (改 status)
         ]
       })
       ↓
       TodoWriteTool.call
       ├─ oldTodos = [...原 7项...]
       ├─ allDone = false
       ├─ verificationNudgeNeeded = false
       └─ setAppState({ todos: { [sessionId]: [...新 7项] } })
       ↓
       UI 重新渲染：第一项打勾，第二项 spinner

T=15s  tool_use: FileWrite({ file_path: "/abs/path/snake-game/src/App.tsx", content: "..." })
       ↓
       runToolUse → FileWriteTool.call
       ├─ Stage 1: expandPath + skill discover
       ├─ Stage 2: mkdir parent dir, fileHistoryTrackEdit 备份
       ├─ Stage 3 (atomic):
       │   ├─ readFileSyncWithMetadata → ENOENT（新文件）
       │   ├─ meta === null → skip staleness check
       │   └─ writeTextContent → 落盘
       ├─ Stage 4: LSP didChange + VSCode notify + readFileState.set
       ├─ Stage 5: type='create', structuredPatch=[], originalFile=null
       └─ return { data: { type: 'create', ... } }
       ↓
       mapToolResultToToolResultBlockParam → "File created successfully at: ..."

T=140s 所有 7 项 todo 都标记 completed
       tool_use: TodoWrite({
         todos: [
           ...7 项全部 status: 'completed'
         ]
       })
       ↓
       TodoWriteTool.call
       ├─ oldTodos = [...前 7项]
       ├─ allDone = true ← 关键
       ├─ todos.length >= 3 = true
       ├─ !todos.some(t => /verif/i.test(...)) = true (没验证任务)
       ├─ feature('VERIFICATION_AGENT') = false (外部 build)
       ├─ verificationNudgeNeeded = false ← 因为 feature gate false
       └─ newTodos = [] ← 清空！
       ↓
       setAppState({ todos: { [sessionId]: [] } })
       UI 任务列表清空
       ↓
       mapToolResultToToolResultBlockParam → "Todos have been modified successfully..."
       无 nudge（feature gate）
```

---

# 附录：与其他工具的对比

| 维度 | FileWriteTool | FileEditTool | TodoWriteTool | Bash |
|------|--------------|--------------|---------------|------|
| 输入 | file_path + content | file_path + old_string + new_string | todos 数组 | command |
| 副作用 | 写磁盘（全量覆盖） | 写磁盘（局部替换） | 写 AppState | 执行 shell |
| 原子性 | read-modify-write 同步三连 | read-modify-write 同步三连 | setAppState 同步 | N/A |
| 必须先读？ | ✅（validateInput 强制） | ✅（同） | ❌ | ❌ |
| LSP 通知？ | ✅（didChange + didSave） | ✅ | ❌ | ❌ |
| FileHistory 备份？ | ✅ | ✅ | ❌ | ❌ |
| VSCode diff view？ | ✅ | ✅ | ❌ | ❌ |
| 权限 | filesystem check + ask | filesystem check + ask | always allow | bash classifier + ask |
| 分类器输入 | file_path + content 全文 | old_string + new_string | "N items" | command 全文 |

---

## 推荐阅读

- [`docs/11-tool-execution.md`](11-tool-execution.md) —— 工具执行路径
- [`docs/14-tool-hooks.md`](14-tool-hooks.md) —— FileWrite/FileEdit 的 PreToolUse/PostToolUse 钩子
- [`docs/18-task-planning.md`](../agent/18-task-planning.md) —— TodoWrite 与 TaskCreate 的 V1/V2 切换
- [`docs/24-snake-game-request-flow.md`](../architecture/24-snake-game-request-flow.md) —— 这两个工具在端到端 case 中的实际调用
- [`docs/25-streaming-tool-executor.md`](25-streaming-tool-executor.md) —— 并发执行器对这两个工具的处理（FileWrite 通常 isConcurrencySafe，TodoWrite 总是 safe）
