# Claude Code 长期记忆系统执行流程

> 本文系统梳理 Claude Code 在"跨会话、跨项目"层面上的所有记忆机制，
> 解释它们如何协同工作，使模型在每次新会话中能"记得"用户的偏好、项目背景和
> 历史决策。
>
> 涉及源码：
>
> - `src/utils/claudemd.ts` — 静态 CLAUDE.md / rules 加载
> - `src/memdir/` — AutoMem 自动长期记忆（核心）
> - `src/memdir/teamMemPaths.ts` & `teamMemPrompts.ts` — TeamMem
> - `src/memdir/paths.ts` — 路径解析
> - `src/memdir/memoryScan.ts` — 文件扫描
> - `src/memdir/findRelevantMemories.ts` — 检索相关记忆
> - `src/memdir/memoryTypes.ts` — 记忆类型
> - `src/memdir/memoryAge.ts` — 记忆老化
> - `src/services/extractMemories/extractMemories.ts` — turn-end 提取
> - `src/services/autoDream/autoDream.ts` — 夜间整合
> - `src/services/SessionMemory/sessionMemory.ts` — 会话内压缩
> - `src/services/compact/sessionMemoryCompact.ts` — 会话压缩
> - `src/tools/AgentTool/agentMemory.ts` — 智能体作用域记忆
> - `src/utils/attachments.ts` — 检索注入
> - `src/utils/hooks.ts` — `InstructionsLoaded` 钩子
> - `src/main.tsx` — 启动装载

---

## 1. Claude Code 记忆的层次结构

Claude Code 实际上有 **5 种相互独立又互补的记忆机制**，按"作用域"和"生命周期"划分：

| 机制 | 物理形态 | 生命周期 | 写入方 | 读取方 |
| --- | --- | --- | --- | --- |
| **CLAUDE.md 家族** | Markdown 文件（User/Project/Local/Managed/.claude/rules） | 永久（git-tracked 或本地） | 人类手动 | 每次会话启动时全量加载 |
| **AutoMem**（个人） | 主题文件 + `MEMORY.md` 索引 + 可选 daily log | 跨会话、按项目 | 模型（fork agent 自动提取） | 启动加载 `MEMORY.md` + 查询时按需检索主题文件 |
| **TeamMem** | 团队共享 AutoMem 子目录 | 跨会话、跨成员 | 模型（fork agent） | 启动加载 + 查询检索 |
| **Agent Memory** | 每个 Agent 一个 `MEMORY.md` | 跨会话、跨作用域（user/project/local） | 子 Agent 自己 | Agent 启动时加载 |
| **Session Memory** | 单文件 `session.md` | 单次会话（用于 /compact） | fork agent 周期性提取 | /compact 时注入压缩版 |

```
┌─────────────────────────────────────────────────────────────────────┐
│                       一次完整会话的生命周期                          │
│                                                                     │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│   │ 启动装载  │ → │ 用户交互  │ → │ turn-end │ → │ /compact │        │
│   │ (CLAUDE.md│   │          │   │ extract  │   │ Session  │        │
│   │ + MEMORY │   │  按需     │   │  fork    │   │ Memory   │        │
│   │ .md +   │   │  recall  │   │  agent   │   │  注入    │        │
│   │ Agent   │   │  主题文件 │   │          │   │          │        │
│   │ memory) │   │          │   │          │   │          │        │
│   └──────────┘   └──────────┘   └──────────┘   └──────────┘         │
│                                          │                          │
│                                          ▼                          │
│                                  ┌──────────────┐                   │
│                                  │  后台        │                   │
│                                  │  auto-dream  │                   │
│                                  │  (夜间整合)   │                   │
│                                  └──────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

下面按"作用域"从大到小（跨多会话 → 单次会话内）逐一展开。

---

## 2. CLAUDE.md 家族：手工维护的静态记忆

这是最古老、也最透明的记忆机制。用户用自然语言写下指令，Claude Code 在每次启动时全部读入 system prompt。

### 2.1 四种类型 + rules 目录

`src/utils/claudemd.ts:790` 的 `getMemoryFiles()`（被 `memoize` 缓存）按以下顺序加载：

| 类型 | 路径 | 来源 | 是否可外部 include |
| --- | --- | --- | --- |
| **Managed** | `<configDir>/managed/CLAUDE.md` + `<configDir>/managed/.claude/rules/*.md` | 管理员策略 | 否 |
| **User** | `~/.claude/CLAUDE.md` + `~/.claude/rules/*.md` | 用户偏好 | 是 |
| **Project** | `<cwd>/CLAUDE.md` + `<cwd>/.claude/CLAUDE.md` + `<cwd>/.claude/rules/*.md` | 团队共享 | 受 `claudeMdExternalIncludesApproved` 控制 |
| **Local** | `<cwd>/CLAUDE.local.md` | 本地（gitignored） | 否 |
| **additionalDirectories** | 通过 `--add-dir` 指定的目录 | 用户显式 | 是 |

### 2.2 加载流程（`getMemoryFiles` 步骤）

```
getMemoryFiles(forceIncludeExternal)
  │
  ├─ 1. Managed —— processMemoryFile + processMdRules(managedClaudeRulesDir)
  │
  ├─ 2. User（如果 userSettings 启用）
  │      processMemoryFile(~/.claude/CLAUDE.md) + ~/.claude/rules/*.md
  │      注：User memory 总允许外部 include
  │
  ├─ 3. Project / Local（向上 walk CWD）
  │      对每层目录：
  │        - 读 CLAUDE.md（Project，projectSettings 启用时）
  │        - 读 .claude/CLAUDE.md（Project）
  │        - 读 .claude/rules/*.md（Project）
  │        - 读 CLAUDE.local.md（Local，localSettings 启用时）
  │      → 处理 nested worktree：跳过 main repo 内的 Project 类型（避免重复加载）
  │
  ├─ 4. additionalDirectories（CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD 启用时）
  │      对每个 --add-dir 读同样的 3 个文件
  │
  ├─ 5. Memdir 入口（AutoMem MEMORY.md）—— 仅 isAutoMemoryEnabled
  │      读 getAutoMemEntrypoint()，type = 'AutoMem'
  │
  └─ 6. TeamMem 入口（feature('TEAMMEM') 启用 + isTeamMemoryEnabled）
        读 teamMemPaths.getTeamMemEntrypoint()，type = 'TeamMem'
```

### 2.3 关键路径：`processMemoryFile`

`processMemoryFile(path, type, processedPaths, includeExternal)` 内部行为：

1. **去重**：`processedPaths: Set<string>` 用规范化路径去重，避免同一文件被多次加载。
2. **排除规则**：`isClaudeMdExcluded(filePath, type)` 比对 `getInitialSettings().claudeMdExcludes`（picomatch 模式，支持 realpath 解析处理 symlink）。
3. **@imports 解析**：在文件 frontmatter 之后使用 `@path/to/other.md` 单行语法引用其他文件，递归深度受限（避免循环）。
4. **External includes（`!`反引号）**：`!`\`cat path/to/file.md\`` 这种语法在 include 时实际执行；受 `includeExternal` 参数控制（Project/Managed 默认 false，需 `claudeMdExternalIncludesApproved` 显式开启）。
5. **Conditional rules（`paths:` frontmatter）**：只有当用户操作匹配 glob 模式时才注入。

### 2.4 排除机制

`isClaudeMdExcluded` + `resolveExcludePatterns`（`claudemd.ts:547`）：

- 支持用户配置的 glob 模式（如 `["**/node_modules/**", "/tmp/**"]`）
- 绝对路径模式额外做 `realpath` 解析（macOS `/tmp` → `/private/tmp`）

### 2.5 `InstructionsLoaded` 钩子

`claudemd.ts:1042` 在每次加载结束后，对每个文件 fire `InstructionsLoaded` 钩子（fire-and-forget）：

- **AutoMem / TeamMem 不触发**：它们属于另一套记忆系统
- `forceIncludeExternal=true` 调用不触发：避免 `getExternalClaudeMdIncludes()` 双重触发
- 钩子参数：`{ path, type, reason: 'session_start'|'resume'|..., globs, parentFilePath }`

### 2.6 token 计数与显示

`src/utils/analyzeContext.ts:320` 的 `countMemoryFileTokens` 用 `countTokensWithFallback`（API + Haiku 回退）精确计数，用于 `/context` 命令的 UI 展示。

---

## 3. AutoMem：自动长期记忆（核心）

这是 Claude Code 真正的"长期记忆"：模型**自己**在合适的时候把值得记住的事情**结构化**地写进文件，下次会话自动加载并按需检索更深层的细节。

### 3.1 物理结构

默认路径：`<configDir>/projects/<sanitized-cwd>/memory/`

```
memory/
├── MEMORY.md           ← 索引（始终加载到 system prompt）
├── user_role.md        ← 主题文件
├── feedback_testing.md
├── project_release_plan.md
├── reference_internal_tools.md
└── logs/               ← KAIROS 模式专用
    └── YYYY/MM/YYYY-MM-DD.md   ← 每日追加日志
```

可被以下覆盖：
- `CLAUDE_CODE_REMOTE_MEMORY_DIR` 环境变量（远端 mount 模式）
- `CLAUDE_CODE_AUTO_MEMORY_DIR` 设置项
- `tengu_auto_memory` 特性开关（默认启用）
- `autoMemoryEnabled: false` 在 `settings.json` 中关闭
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` 关闭

### 3.2 四种记忆类型

`src/memdir/memoryTypes.ts:14` 定义的封闭类型集：

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| 类型 | 用途 | 例 |
| --- | --- | --- |
| **user** | 用户的身份、角色、目标 | "User is a senior backend engineer at Acme Corp" |
| **feedback** | 用户对行为的纠正/期望 | "User prefers bun over npm" |
| **project** | 项目上下文（代码不能告诉你的） | "Project ships 2026-Q3; v2 migration in progress" |
| **reference** | 外部系统指针 | "Linear project: ENG-1234, Slack: #eng-platform" |

文件 frontmatter 格式（`MEMORY_FRONTMATTER_EXAMPLE`）：

```markdown
---
name: User role
description: Senior backend engineer focused on payments reliability
type: user
---

User is a senior backend engineer focused on payments reliability.
```

`MEMORY.md` 是**索引**（没有 frontmatter），每行一个指针：

```markdown
- [User role](user_role.md) — senior backend engineer, payments focus
- [Bun preference](feedback_testing.md) — use bun, not npm
- [Release plan](project_release_plan.md) — Q3 2026, v2 migration
```

### 3.3 加载入口：`loadMemoryPrompt`

`src/memdir/memdir.ts:419` 的 `loadMemoryPrompt()` 在 `main.tsx` 启动时由 `getSystemPrompt()` 调用。

分支逻辑：

```
loadMemoryPrompt()
  │
  ├─ KAIROS 模式 + 自动开启 + getKairosActive()
  │    └─ buildAssistantDailyLogPrompt()  ← 指示模型写日志，不维护 MEMORY.md
  │
  ├─ TEAMMEM 启用 + isTeamMemoryEnabled()
  │    └─ buildCombinedMemoryPrompt()  ← 同时描述 auto + team 目录
  │
  └─ auto 启用（默认）
       └─ buildMemoryLines() → 描述 auto memory 的元规则
       注：MEMORY.md 的内容由 claudemd.ts 单独注入到 user-context
```

返回的 prompt 是**指令文本**（告诉模型"你应该记录什么、怎么写"），**不**包含 `MEMORY.md` 的内容。`MEMORY.md` 的实际内容在 `claudemd.ts` 通过 `processMemoryFile` 当作一个普通文件注入（`type: 'AutoMem'`），路径由 `getAutoMemEntrypoint()` 给出。

### 3.4 写入：何时写、写到哪

模型的"写"行为由 `loadMemoryPrompt` 的指令驱动：

1. **常规模式**（默认）：
   - 维护 `MEMORY.md`（1 行/条目） + 主题文件
   - 主题文件按**语义**组织（不是按时间）
   - 写新条目前先检查是否可更新现有文件
   - 内容超过 200 行/25KB 时**先**截断（`truncateEntrypointContent`），避免索引过大
2. **KAIROS 模式**（`feature('KAIROS')` + `getKairosActive()`）：
   - 只追加 `logs/YYYY/MM/YYYY-MM-DD.md`，append-only
   - 不维护 `MEMORY.md`
   - 路径用 `YYYY-MM-DD` 模式而非当前日期，**保持 prompt cache 不失效**（日期由 `date_change` attachment 动态注入）
   - 夜间的 `auto-dream` 流程把日志"蒸馏"成 `MEMORY.md` + 主题文件

### 3.5 提取：turn-end 自动 fork

模型在主流程中**可能**会自己写记忆。但更稳健的路径是后台自动 fork agent 在每个 turn 结束后提取关键信息。

入口：`src/services/extractMemories/extractMemories.ts:initExtractMemories()`

主流程（在 `main.tsx:5031` `run` 中）：

```
每个 turn 结束（handleStopHooks）
  ↓
runExtraction(context)
  │
  ├─ 1. 跳过条件
  │      - getIsRemoteMode() → 跳过
  │      - !isAutoMemoryEnabled() → 跳过
  │      - countModelVisibleMessagesSince() < N → 跳过（节省资源）
  │      - hasMemoryWritesSince() → 跳过（主 agent 已经写了，后台是冗余）
  │
  ├─ 2. 准备 manifest
  │      scanMemoryFiles() → 已存在的记忆列表
  │      formatMemoryManifest() → 格式化为文本
  │
  ├─ 3. Fork subagent
  │      runForkedAgent({
  │        promptMessages: [buildExtractPrompt(messages, manifest)],
  │        canUseTool: createAutoMemCanUseTool(),  // 只能 Read/Write/Edit 记忆目录
  │        querySource: 'extract_memories',
  │        overrides: { readFileState: ... },
  │      })
  │
  └─ 4. 推进 cursor
        lastExtractedUuid = lastMessage.uuid
```

`createAutoMemCanUseTool` 的限制：只允许 Read、Write、Edit、Glob、Grep 工具，且文件路径必须通过 `isAutoMemPath()` 校验（防止 agent 越权写其他文件）。

`buildExtractPrompt`（`prompts.ts`）指示 subagent：
- 复用 `getMemoryBaseDir()`/`getAutoMemPath()` 的路径
- 用 `MEMORY.md` 索引 + 4 个类型分类
- 增量提取：基于 manifest 决定是 update 还是 add
- **不要重复**：如果主 agent 已经写了 `hasMemoryWritesSince() === true` → 跳过

### 3.6 检索：query-time 按需拉取

`MEMORY.md` 只提供"标题级"的索引。完整内容只在**模型需要时**才被注入。

入口：`src/utils/attachments.ts` 中的 `getMemoryPrefetchAttachment`（异步 prefetch）

`MemoryPrefetch` 类型（`attachments.ts:2346`）：

```typescript
type MemoryPrefetch = {
  promise: Promise<Attachment[]>
  settledAt: number | null
  consumedOnIteration: number
  [Symbol.dispose]: () => void
}
```

主流程：

```
query.ts 启动 → memoryPrefetch = startMemoryPrefetch(query, toolUseContext, signal)
  ↓
事件循环每轮 await
  ↓
attachment collection point
  ├─ if prefetch settled && not consumed → consume
  │     - findRelevantMemories(query, memoryDir, signal, recentTools, alreadySurfaced)
  │       ├─ scanMemoryFiles(memoryDir, signal)  ← 读 frontmatter
  │       ├─ selectRelevantMemories(query, headers, signal, recentTools)
  │       │     - 用 sonnet sideQuery 在 frontmatter 描述里挑最相关的 5 个
  │       │     - JSON schema 强制返回 { selected_memories: string[] }
  │       │     - querySource: 'memdir_relevance'
  │       │     - 256 max_tokens
  │       └─ 返回 [{ path, mtimeMs }]
  │     - 把这些 .md 文件**全文**作为 attachment 注入对话
  └─ 记录 `consumedOnIteration` 用于后续去重
```

#### `findRelevantMemories` 细节（`src/memdir/findRelevantMemories.ts:39`）

- `scanMemoryFiles` 只读每个文件的前 30 行（frontmatter 部分），避免 IO
- `selectRelevantMemories` 用 `getDefaultSonnetModel()` + `sideQuery`（一个轻量级 API 调用，**不**消耗主模型）
- `recentTools` 提示：避免为正在使用的工具选择其"使用文档"类 reference
- `alreadySurfaced` 避免重复推送同一文件
- 失败时返回 `[]`（不会阻塞主流程）
- 命中时 `logMemoryRecallShape`（`feature('MEMORY_SHAPE_TELEMETRY')`）记录分布

#### `scanMemoryFiles`（`src/memdir/memoryScan.ts:35`）

- `readdir(memoryDir, { recursive: true })` 递归扫描
- 跳过 `MEMORY.md`（已在 system prompt）
- 限制 `MAX_MEMORY_FILES = 200`
- 按 `mtimeMs` 降序排，取前 200
- 读前 `FRONTMATTER_MAX_LINES = 30` 行（用 `readFileInRange` 内部 stat）
- 解析 frontmatter 拿到 `description`（用于 selector）和 `type`（用于 manifest 显示）

### 3.7 老化：`memoryAge.ts`

记忆文件 `mtime` 越新，selector 越倾向选择。详细规则见 `memoryAge.ts`（实际策略：fresher = higher priority, capped to avoid hyper-recent bias）。

### 3.8 边界与路径校验

`src/memdir/paths.ts:274` `isAutoMemPath(absolutePath)`：

- 用 `normalize(absolutePath)` 规范化（防止 `..` 绕过）
- 检查是否在 `getAutoMemPath()` 之下
- `isAutoMemPath` 用于：
  - `createAutoMemCanUseTool` 路径校验
  - `hasMemoryWritesSince` 判断主 agent 是否写了记忆
  - Path Safety Check（在 `checkPathConstraints` 中识别为"安全路径"，避免触发敏感路径 ask）

---

## 4. TeamMem：团队共享记忆

源代码：`src/memdir/teamMemPaths.ts` & `teamMemPrompts.ts`

`feature('TEAMMEM')` 启用 + `isTeamMemoryEnabled()` 进一步受 `tengu_herring_clock` GrowthBook 旗标控制。

### 4.1 路径

`getTeamMemPath()` = `getAutoMemPath() + 'team/'` —— 是 auto mem 的子目录（不是独立目录）。

### 4.2 行为差异

- 写：fork agent 既可以写 auto 也可以写 team（看指令内容）
- 读：`buildCombinedMemoryPrompt` 同时描述两个目录
- 同步（ant-only）：定期把 auto → team 双向同步

### 4.3 关闭顺序

`isTeamMemoryEnabled()` 检查 `isAutoMemoryEnabled()`，因此**关闭 auto 自动关闭 team**。

---

## 5. Agent Memory：智能体作用域

源代码：`src/tools/AgentTool/agentMemory.ts`

每个 agent type（如 `general-purpose`、`statusline-setup`）可以声明 `memory: 'user' | 'project' | 'local'` 字段启用持久记忆。

### 5.1 三种 scope

| Scope | 路径 | 适用范围 | 是否 git 共享 |
| --- | --- | --- | --- |
| `user` | `~/.claude/agent-memory/<agentType>/` | 跨项目 | 否（个人） |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | 项目内所有成员 | 是（git tracked） |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | 当前机器 | 否（gitignored） |

### 5.2 加载

`loadAgentMemoryPrompt(agentType, scope)` 在 AgentDetail 启动时调用：

1. `ensureMemoryDirExists(memoryDir)`（fire-and-forget，mkdir 父链）
2. `buildMemoryPrompt({ displayName: 'Persistent Agent Memory', memoryDir, extraGuidelines: [scopeNote, ...] })`

`scopeNote` 解释该 scope 的语义边界（如 "user-scope, keep learnings general"）。

### 5.3 写入

Agent 自己用 Write/Edit 工具写入自己的 `MEMORY.md`（`getAgentMemoryEntrypoint(agentType, scope)`）。

`isAgentMemoryPath(absolutePath)` 用于 `createCanUseTool` 路径校验（仅允许写自己的 scope 目录）。

---

## 6. Session Memory：会话内压缩辅助

源代码：`src/services/SessionMemory/sessionMemory.ts` & `sessionMemoryCompact.ts`

**不是跨会话**的记忆——是 `/compact` 时给压缩后的对话补充"丢失的细节"。

### 6.1 触发

`initSessionMemory()` 在 `main.tsx` 启动时调用：

```
initSessionMemory()
  │
  ├─ 跳过条件
  │      - getIsRemoteMode() → 跳过
  │      - !isAutoCompactEnabled() → 跳过
  │
  └─ registerPostSamplingHook(extractSessionMemory)
```

### 6.2 后台提取（每个 turn 采样后）

`extractSessionMemory`（`sessionMemory.ts:272`）流程：

```
每个 turn 结束 → 异步触发（不阻塞主对话）
  │
  ├─ 1. shouldExtractMemory(messages)
  │      - 已初始化 && tokens >= initThreshold (10K)
  │      - (tokenThreshold && toolCallThreshold) || (tokenThreshold && no tool calls)
  │      - tokensSinceLast >= minTokensBetweenUpdate (5K)
  │      - toolCallsSinceLast >= toolCallsBetweenUpdates (3)
  │
  ├─ 2. setupSessionMemoryFile()
  │      - mkdir <configDir>/session-memory/<sessionId>/
  │      - 写模板（如有）到 session.md
  │      - FileReadTool 读当前 session.md 内容
  │
  ├─ 3. runForkedAgent({ ... createMemoryFileCanUseTool(path) })
  │      - subagent 只能 Edit session.md
  │      - 指令：基于对话提取关键 facts，append 到 session.md
  │
  └─ 4. recordExtractionTokenCount() + updateLastSummarizedMessageIdIfSafe()
```

### 6.3 `/compact` 时的注入

`src/services/compact/sessionMemoryCompact.ts:SessionMemoryCompactConfig`：

- `minTokens: 10_000`（compact 后最少保留 10K tokens）
- `minTextBlockMessages: 5`（最少保留 5 条带文本的消息）
- `maxTokens: 40_000`（最多保留 40K tokens）

`getSessionMemoryContent()` 在 `/compact` 流程中被读取并注入压缩后的对话中，确保不丢失关键事实。

`waitForSessionMemoryExtraction()` 在 `/compact` 时调用，确保任何正在进行的提取完成（15s timeout，1min 后视为 stale 不再等）。

---

## 7. Auto-Dream：夜间记忆整合

源代码：`src/services/autoDream/autoDream.ts`

类比人睡眠时整理记忆。KAIROS 模式下特别重要——把 daily logs 蒸馏成 `MEMORY.md` + 主题文件。

### 7.1 触发条件（三道关卡）

```
initAutoDream() 注册 post-sampling hook
  │
  └─ 每个 turn → runAutoDream(context, appendSystemMessage)
       │
       ├─ Gate 1: isGateOpen()
       │    - KAIROS active → false
       │    - remote mode → false
       │    - !isAutoMemoryEnabled() → false
       │    - !isAutoDreamEnabled() → false
       │
       ├─ Gate 2: Time gate
       │    - hoursSince = (now - lastConsolidatedAt) / 3_600_000
       │    - < minHours (24h 默认) → return
       │
       ├─ Gate 3: Session gate
       │    - listSessionsTouchedSince(lastAt) → N sessions
       │    - N < minSessions (5 默认) → return
       │
       ├─ Gate 4: Lock
       │    - tryAcquireConsolidationLock() → 防止多进程冲突
       │    - 拿不到锁 → return
       │
       └─ runForkedAgent({ prompt: buildConsolidationPrompt(), ... })
```

### 7.2 Consolidation Prompt

`src/services/autoDream/consolidationPrompt.ts` 指示 subagent：

1. 读取 `lastAt` 之后的所有 session transcript（`listSessionsTouchedSince` 给出 IDs）
2. 提取新事实，分类到 4 种类型
3. **更新** `MEMORY.md` 索引（合并/去重/重新分类）
4. **创建/更新**主题文件
5. **更新** `consolidationLock.ts:writeLastConsolidatedAt(now)` —— 触发下次时间门

### 7.3 DreamTask

consolidation 通过 `tasks/DreamTask/DreamTask.js` 注册成一个 task，用户能在 UI 看到状态（"running" / "last ran 2h ago"）。

`MemoryFileSelector.tsx` 显示 auto-dream 状态和 "running" 指示器。

---

## 8. 记忆注入的总图：system prompt 怎么长这样

当主 agent 启动时，`getSystemPrompt()` 调用顺序大致为：

```
getSystemPrompt(tools, model)
  │
  ├─ systemPromptSection('claudemd', ...)  ← 注入所有 CLAUDE.md
  │      ├─ Managed → User → Project → Local → additionalDirectories
  │      ├─ AutoMem MEMORY.md (type: 'AutoMem')   ← 索引被注入
  │      └─ TeamMem MEMORY.md (type: 'TeamMem')
  │
  ├─ systemPromptSection('memory', loadMemoryPrompt())
  │      └─ 描述"如何写"记忆的指令文本
  │
  ├─ systemPromptSection('agentMemory', loadAgentMemoryPrompt())
  │      └─ 子 agent 的记忆指令 + 索引
  │
  ├─ tools / agents / skills / slash commands 描述
  │
  └─ 其他部分
```

`systemPromptSection(key, value)` 用 `key` 缓存：相同的 key 只在内容变化时重算，便于 prompt cache 命中。

**关键点**：`MEMORY.md` 索引被作为 system prompt 一部分**全量加载**；但 4 种类型下的具体内容文件只在 `findRelevantMemories()` 触发时**按需注入**为 user message attachment。

---

## 9. 完整时序图

```
                            ┌──────────┐
                            │ 用户启动  │
                            │  claude  │
                            └────┬─────┘
                                 │
   ┌─────────────────────────────▼──────────────────────────────┐
   │ main.tsx → run()                                            │
   │  ├─ getSystemPrompt() ──────────────────────────────────► │ 装载 CLAUDE.md + MEMORY.md
   │  │                                                          │ + Agent memory 指令
   │  │                                                          │
   │  ├─ initSessionMemory() ──► registerPostSamplingHook       │
   │  ├─ initExtractMemories() ──► registerStopHook             │
   │  ├─ initAutoDream() ──► registerPostSamplingHook           │
   │  │                                                          │
   │  └─ query loop:                                              │
   │     │                                                       │
   │     ▼                                                       │
   │     user input  ──►  attachments = await getAttachments()   │
   │                       │                                     │
   │                       ├─ startMemoryPrefetch(query)         │
   │                       │     └─ async: sonnet picks N files │
   │                       │                                     │
   │                       ▼                                     │
   │     [user, prefetched_memory_attachments, ...]              │
   │     │                                                       │
   │     ▼                                                       │
   │     API call (with full system prompt + memories in msg)    │
   │     │                                                       │
   │     ▼                                                       │
   │     tool_use / text response                                │
   │     │                                                       │
   │     ├─ on each turn end:                                    │
   │     │    ├─ extractMemories ─────► fork subagent            │
   │     │    │                       writes/updates memory files│
   │     │    │                                                     │
   │     │    ├─ sessionMemoryExtract ───► fork subagent            │
   │     │    │                       updates session.md            │
   │     │    │                                                     │
   │     │    └─ autoDream ────► (gated, may or may not run)       │
   │     │                                                          │
   │     └─ user submits again or /compact                        │
   │          │                                                   │
   │          ├─ /compact:                                         │
   │          │    └─ sessionMemoryCompact 注入 session.md 内容   │
   │          │       + 压缩后的对话                               │
   │          │                                                   │
   │          └─ exit / new session                                │
   └──────────────────────────────────────────────────────────────┘
```

---

## 10. 关键源码速查

| 关注点 | 入口函数 | 文件:行 |
| --- | --- | --- |
| 启动装载 CLAUDE.md | `getMemoryFiles` | `src/utils/claudemd.ts:790` |
| 装载 AutoMem 指令 | `loadMemoryPrompt` | `src/memdir/memdir.ts:419` |
| 获取 AutoMem 路径 | `getAutoMemPath` | `src/memdir/paths.ts:223` |
| 写指令文本（meta） | `buildMemoryLines` | `src/memdir/memdir.ts:199` |
| Agent memory 装载 | `loadAgentMemoryPrompt` | `src/tools/AgentTool/agentMemory.ts:138` |
| Agent memory 路径 | `getAgentMemoryDir` | `src/tools/AgentTool/agentMemory.ts:52` |
| 扫描记忆 | `scanMemoryFiles` | `src/memdir/memoryScan.ts:35` |
| 检索相关记忆 | `findRelevantMemories` | `src/memdir/findRelevantMemories.ts:39` |
| Turn-end 提取 | `initExtractMemories` | `src/services/extractMemories/extractMemories.ts` |
| AutoMem 路径校验 | `isAutoMemPath` | `src/memdir/paths.ts:274` |
| Agent memory 路径校验 | `isAgentMemoryPath` | `src/tools/AgentTool/agentMemory.ts:68` |
| Session 提取 | `initSessionMemory` / `extractSessionMemory` | `src/services/SessionMemory/sessionMemory.ts:357, 272` |
| Session 压缩 | `SessionMemoryCompactConfig` | `src/services/compact/sessionMemoryCompact.ts:47` |
| Auto-dream 整合 | `initAutoDream` / `runAutoDream` | `src/services/autoDream/autoDream.ts:122, 125` |
| 记忆预取 | `startMemoryPrefetch` | `src/utils/attachments.ts:2356` |
| 指令加载钩子 | `executeInstructionsLoadedHooks` | `src/utils/hooks.ts` |
| 4 种类型定义 | `MEMORY_TYPES` | `src/memdir/memoryTypes.ts:14` |
| 索引截断 | `truncateEntrypointContent` | `src/memdir/memdir.ts:57` |
| KAIROS 日志路径 | `getAutoMemDailyLogPath` | `src/memdir/paths.ts:246` |
| Team mem 启用判断 | `isTeamMemoryEnabled` | `src/memdir/teamMemPaths.ts` |
| Consolidation prompt | `buildConsolidationPrompt` | `src/services/autoDream/consolidationPrompt.ts` |

---

## 11. 设计要点

### 11.1 分层与互不污染

- **CLAUDE.md**：人类写、人类管、git 友好
- **AutoMem**：模型写、模型管、个人
- **TeamMem**：模型写、人类/团队共享、git 友好
- **AgentMemory**：每个 agent 独立空间，可被 git 共享
- **SessionMemory**：单会话、用于压缩

每层**不重叠**：CLAUDE.md 不写入运行时的"对话摘要"；AutoMem 不写静态偏好（但可以记录"我了解到你偏好 X"，让下次会话自动加载）。

### 11.2 写入策略

- **CLAUDE.md 写入**：用户必须手动 Edit
- **AutoMem 写入**：
  - 主 agent 可在 turn 期间直接 Write（指令在 `loadMemoryPrompt` 中给出）
  - 后台 fork agent 在 turn-end 增量提取（**自动**）
  - Auto-dream 在跨多 session 后再整合（**去重 / 重新分类**）
- 写入路径有 `isAutoMemPath` / `isAgentMemoryPath` 严格校验，防止越权

### 11.3 读取策略

- **CLAUDE.md**：每次启动**全量**加载（不区分是否相关）
- **AutoMem**：
  - `MEMORY.md` 索引全量加载（~25KB 上限）
  - 主题文件**按需**通过 sonnet selector 检索后注入
- **AgentMemory**：子 agent 启动时全量加载索引

### 11.4 写入-读取一致性

写入端（fork agent）的"manifest" 和读取端（recall）的 "selector" 用**同一个 `scanMemoryFiles`** + **同一个 frontmatter 解析**，保证两边对"哪些文件存在、描述是什么"的理解一致。

### 11.5 性能保护

- 记忆 selector 用 sonnet 单独调用（`sideQuery`），不消耗主模型的 thinking budget
- 记忆主题文件**只读 frontmatter**（前 30 行），不读全文（节省 IO）
- `MAX_MEMORY_FILES = 200` 防止无限增长
- `consolidationLock` 防止多进程并发 dream
- `SESSION_SCAN_INTERVAL_MS = 10min` 防止 session 列表扫描过频

### 11.6 缓存策略

- `getMemoryFiles` 整个函数被 `memoize`，但有 `forceIncludeExternal` 旁路
- `systemPromptSection(key, value)` 按 key 缓存，便于 prompt cache 命中
- `MEMORY.md` 路径在 prompt 中**写成模式**（KAIROS: `logs/YYYY/MM/YYYY-MM-DD.md`），日期由 `date_change` attachment 动态注入，**不**让 prompt cache 失效

### 11.7 安全设计

- `isAutoMemPath` 规范化路径 + 前缀匹配，防止 `..` 绕过
- `createAutoMemCanUseTool` 限制 agent 只能读/写记忆目录
- `createMemoryFileCanUseTool` 限制 session 提取只能 Edit 单一 session.md
- 外部 include 需 `claudeMdExternalIncludesApproved` 显式开启
- `getInitialSettings().claudeMdExcludes` 提供用户级 glob 排除

---

## 12. 与"普通上下文"的关系

- **MEMORY.md** = 始终在 system prompt 中的"长期记忆目录"
- **主题文件** = 按 query 检索注入的"长期记忆细节"
- **conversation** = 当次会话的短期上下文
- **/compact** = 短期上下文压缩时，session memory 提供"已丢失细节的备份"

四者组合使 Claude Code 在不同"时间尺度"上都有合适的记忆能力。

---

## 13. 相关文档

- `docs/36-how-to-read-claude-code.md` — 如何读懂 Claude Code 源码
- `docs/architecture/31-bootstrap-state.md` — 启动装载
- `docs/architecture/34-appstate-vs-state.md` — AppState 状态管理
- `docs/permission/41-bash-tool-permission-control.md` — Bash 工具权限
- `docs/hook/` — 钩子系统
- `src/services/compact/` — 压缩系统
- `src/utils/hooks.ts` — InstructionsLoaded 等钩子实现
