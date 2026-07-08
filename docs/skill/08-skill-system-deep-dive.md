# Skill 系统全景剖析 —— 从磁盘到模型再到执行

> 本文对照源码，完整论述 Claude Code 的 **Skill 系统**是怎么运作的：一个 skill 从磁盘文件被发现、解析成命令对象、注入模型上下文、被模型或用户触发、展开成 prompt、执行（inline / forked / remote）、直到跨 compaction 保留的完整生命周期。
>
> 关联单点文档：`docs/tool/13-skill-tool.md`（只讲 `SkillTool.call` 的三种执行分支）。本文是它的上游与全景补充。
>
> 核心源码：
> - `src/skills/loadSkillsDir.ts`（磁盘发现、解析、动态/条件 skill）
> - `src/skills/bundledSkills.ts`（内建 skill 注册）
> - `src/commands.ts`（命令聚合、`getCommands` / `getSkillToolCommands`）
> - `src/tools/SkillTool/SkillTool.ts` + `prompt.ts`（模型调用入口 + 工具描述）
> - `src/utils/attachments.ts` + `src/utils/messages.ts`（skill 列表注入 system-reminder）
> - `src/utils/processUserInput/processSlashCommand.tsx`（prompt 展开、hooks 注册、compaction 登记）
> - `src/utils/forkedAgent.ts` + `src/tools/AgentTool/runAgent.ts`（forked 执行）

---

## 0. 什么是 Skill

在 Claude Code 里，**Skill 本质上就是一条 `type: 'prompt'` 的 `Command`**：一段带 YAML frontmatter 的 Markdown（约定文件名 `SKILL.md`），它的正文在被触发时会被展开成 prompt 注入对话。它有两种触发路径，共用同一套底层机制：

1. **用户触发**：用户输入 `/skill-name args`（slash command）。
2. **模型触发**：模型发出 `Skill(skill: "name", args: "...")` tool_use（`SkillTool`）。

一个 skill 与普通的“内建 slash 命令”（`type: 'local'` / `'local-jsx'`，如 `/clear`、`/help`）的区别在于：skill 是 `prompt` 类型 —— 它不执行本地逻辑，而是**把自身内容变成给模型的 prompt**。这就是为什么 skill 与 slash command 在源码里被统一到 `Command` 抽象下、并共享 `processSlashCommand` 通道。

Skill 的能力由 frontmatter 声明（`src/skills/loadSkillsDir.ts:185 parseSkillFrontmatterFields`）：

| frontmatter 字段 | 作用 |
|------|------|
| `name` | 显示名（`userFacingName()`），默认用目录名 |
| `description` | 一行摘要，进入 skill 列表供模型匹配（缺省时从正文首行提取） |
| `when_to_use` | 何时该用；拼进 skill 列表描述辅助模型判断 |
| `allowed-tools` | 触发后额外授予的工具权限（写进 `alwaysAllowRules.command`） |
| `argument-hint` / `arguments` | 参数提示与命名参数（`$1`、`$ARGUMENTS`） |
| `model` | 覆盖模型（`inherit` 表示不覆盖） |
| `effort` | 覆盖推理强度 |
| `disable-model-invocation` | `true` 则模型不能用 `SkillTool` 调它，只能用户 `/` 调 |
| `user-invocable` | 默认 `true`；`false` 则用户不能 `/` 调，只能模型调 |
| `context` | `fork` 表示跑在隔离子代理里；缺省为 inline |
| `agent` | forked 执行时用哪个 agent 定义 |
| `hooks` | skill 自带的生命周期 hooks（触发时注册） |
| `paths` | **条件激活**：只有当操作的文件匹配这些 glob 时才把 skill 加入可用列表 |
| `shell` | `!command` 注入时用的 shell 配置 |

---

## 1. 五个来源：Skill 从哪里来

`src/commands.ts:449 loadAllCommands` 把所有命令来源并发加载后拼成一个数组。Skill 来自其中 **5 个来源**，`LoadedFrom` 类型（`loadSkillsDir.ts:67`）标记出处：

```
loadAllCommands(cwd)  ← memoize by cwd
  └─ Promise.all([ getSkills(cwd), getPluginCommands(), getWorkflowCommands(cwd) ])

getSkills(cwd)  (commands.ts:353)
  ├─ getSkillDirCommands(cwd)     → loadedFrom: 'skills' | 'commands_DEPRECATED' | 'managed'
  ├─ getPluginSkills()            → loadedFrom: 'plugin'
  ├─ getBundledSkills()           → loadedFrom: 'bundled'   (同步，启动时注册)
  └─ getBuiltinPluginSkillCommands() → 内建插件 skill

最终拼接顺序 (loadAllCommands 返回):
  [ ...bundledSkills, ...builtinPluginSkills, ...skillDirCommands,
    ...workflowCommands, ...pluginCommands, ...pluginSkills, ...COMMANDS() ]
```

| 来源 | `loadedFrom` | 磁盘位置 / 注册方式 |
|------|-------------|------|
| **磁盘 /skills/** | `'skills'` | `~/.claude/skills/<name>/SKILL.md`（user）、`.claude/skills/`（project）、`<managed>/.claude/skills/`（policy），逐级向上到 home |
| **遗留 /commands/** | `'commands_DEPRECATED'` | `.claude/commands/`，支持 `SKILL.md` 目录格式与单 `.md` 文件 |
| **Bundled** | `'bundled'` | 编译进二进制，`registerBundledSkill()` 启动时注册 |
| **Plugin** | `'plugin'` | 已安装插件提供 |
| **MCP** | `'mcp'` | MCP server 暴露的 prompt（`feature('MCP_SKILLS')`），存在 `AppState.mcp.commands`，不进 `getCommands()` |

### 1.1 磁盘发现：`getSkillDirCommands`（loadSkillsDir.ts:638）

`memoize(cwd)` 缓存。核心步骤：

1. 计算三类目录：`userSkillsDir`（`~/.claude/skills`）、`managedSkillsDir`（policy）、`projectSkillsDirs`（从 cwd 逐级向上到 home，`getProjectDirsUpToHome('skills', cwd)`）。
2. **策略门控**：`isRestrictedToPluginOnly('skills')` 为真则锁定非插件 skill；`isSettingSourceEnabled` 分别控制 user/project 来源；`CLAUDE_CODE_DISABLE_POLICY_SKILLS` 关闭 managed。
3. **`--bare` 模式**：跳过所有自动发现，只加载显式 `--add-dir` 目录。
4. 并发从各目录 `loadSkillsFromSkillsDir`（只认 `<dir>/SKILL.md` 目录格式）+ 遗留 `loadSkillsFromCommandsDir`。
5. **按文件身份去重**（`getFileIdentity` 用 `realpath` 解 symlink），处理软链/重叠父目录导致的重复；first-wins。
6. **分离条件 skill**：带 `paths` frontmatter 的 skill 不直接返回，而是存进 `conditionalSkills` map，等匹配文件被触碰时再激活（见 §5）。
7. 返回**无条件 skill** 列表。

每个 skill 通过 `createSkillCommand`（loadSkillsDir.ts:270）变成 `Command` 对象，其中最关键的是 **`getPromptForCommand(args, ctx)`** 闭包 —— 这是触发时把 SKILL.md 正文变成 prompt 的函数（见 §4）。

### 1.2 Bundled skill：`registerBundledSkill`（bundledSkills.ts:53）

内建 skill 在模块初始化时 `push` 进 `bundledSkills[]` 数组。特点：

- 可携带 `files`（附带脚本/参考文件），首次调用时 `extractBundledSkillFiles` **懒解压到磁盘**（memoize promise 防并发重复写），并在 prompt 前面 prepend `Base directory for this skill: <dir>`。
- `isEnabled` 可动态开关；`userInvocable: false` 则 `isHidden: true`。

---

## 2. 聚合与过滤：谁能进入模型视野

`getCommands(cwd)`（commands.ts:476）在 `loadAllCommands` 之上叠加**运行时过滤**（不 memoize，保证 `/login` 等鉴权变化立即生效）：

- `meetsAvailabilityRequirement`：按 `availability`（claude-ai / console）过滤。
- `isCommandEnabled`：`isEnabled()` 判断。
- 追加 `getDynamicSkills()`（会话中动态发现的 skill，插在 plugin skill 之后、内建命令之前）。

在此之上有两个面向不同消费者的过滤器：

**`getSkillToolCommands`（commands.ts:563）—— 模型能通过 SkillTool 调的**：

```
cmd.type === 'prompt'
  && !cmd.disableModelInvocation      // 排除仅用户可调的
  && cmd.source !== 'builtin'         // 排除内建 slash 命令
  && (loadedFrom ∈ {bundled, skills, commands_DEPRECATED}  // 这些有自动描述
      || cmd.hasUserSpecifiedDescription || cmd.whenToUse)  // plugin/mcp 必须显式描述
```

**`getSlashCommandToolSkills`（commands.ts:586）—— 狭义“skill”**（`skills`/`plugin`/`bundled` 或 `disableModelInvocation`）。

> 关键区别：用户可用 `/name` 调所有 `userInvocable !== false` 的命令；模型只能用 `SkillTool` 调 `getSkillToolCommands` 过滤后的子集。`disable-model-invocation` 与 `user-invocable` 两个 flag 正交地控制这两条路径。

---

## 3. 注入模型：Skill 列表如何进入上下文

模型并不会一次性看到所有 SKILL.md 正文（太贵）。它只看到一个**紧凑的 skill 名+描述清单**，正文在被调用时才加载。这是 skill 系统“渐进披露（progressive disclosure）”的核心。

### 3.1 SkillTool 的工具描述（prompt.ts:173 `getPrompt`）

`SkillTool.prompt` 返回一段固定说明，告诉模型：
- 用户提到 “slash command / `/xxx`” 就是指 skill，用本工具调；
- **可用 skill 列在 system-reminder 里**；
- 匹配到 skill 是 **BLOCKING REQUIREMENT** —— 必须先调工具再回答；
- 看到 `<command-name>` 标签说明 skill 已加载，直接跟随指令，不要重复调。

### 3.2 Skill 清单作为 attachment 注入（attachments.ts:2661）

`getSkillListingAttachments` 在每轮收集 attachment 时运行：

1. 若当前 agent 的工具集里没有 `SkillTool` → 返回空（子代理可能没这工具）。
2. 取 `getSkillToolCommands(cwd)` ∪ MCP skills（`uniqBy('name')`）。
3. **增量注入**：`sentSkillNames`（按 agentId 分桶）记录已发过的 skill；只发**新增**的。首批 `isInitial: true`。
4. `--resume` 时 `suppressNext` 把当前全部标记为已发（转写里已有清单），只发后续增量。
5. `formatCommandsWithinBudget`（prompt.ts:70）在 **上下文窗口 1%（字符）预算**内格式化：
   - 优先全描述；超预算则 bundled skill 永不截断，其余按 `MAX_LISTING_DESC_CHARS=250` 截断；极端情况非 bundled 退化为“仅名字”。

### 3.3 清单变成 system-reminder（messages.ts:3731）

`skill_listing` attachment 被 `wrapMessagesInSystemReminder` 包裹成一条 `isMeta` 用户消息：

```
<system-reminder>
The following skills are available for use with the Skill tool:

- commit: Create a git commit ...
- pdf: Extract and manipulate PDF ...
- review-pr: ...
</system-reminder>
```

这就是 SkillTool prompt 里说的“列在 system-reminder 里”的那份清单。模型据此匹配意图并决定调哪个 skill。

---

## 4. 触发与展开：从 SKILL.md 到 prompt

无论用户 `/commit` 还是模型 `Skill(commit)`，最终都会调 `command.getPromptForCommand(args, ctx)`。这个闭包（loadSkillsDir.ts:344）做的事：

```
1. 若有 baseDir → prepend "Base directory for this skill: <baseDir>\n\n"
2. substituteArguments(content, args, argumentNames)   // $ARGUMENTS / $1 / $2 ...
3. 替换 ${CLAUDE_SKILL_DIR}  → skill 自身目录（Windows 反斜杠转正斜杠）
4. 替换 ${CLAUDE_SESSION_ID} → 当前 session id
5. 若 loadedFrom !== 'mcp':
     executeShellCommandsInPrompt(...)   // 展开 !`cmd` 和 ```! cmd ``` 注入
        —— 注入时把 skill 的 allowed-tools 塞进 alwaysAllowRules.command
6. 返回 [{ type: 'text', text: finalContent }]
```

> **安全边界**：MCP skill 是远端不可信内容，`loadedFrom === 'mcp'` 时**绝不执行**内联 shell（第 5 步跳过）。远程 canonical skill 同理（见 13-skill-tool.md）。

### 4.1 两条触发路径汇入同一展开逻辑

**用户 `/skill`** → `processSlashCommand`（processSlashCommand.tsx:309）→ 校验命令存在 → `getMessagesForSlashCommand`。若 `userInvocable === false` 直接拒绝（模型专用）。

**模型 `Skill(...)`** → `SkillTool.validateInput`（存在性/类型/`disableModelInvocation` 预检）→ `checkPermissions`（allow/deny/ask + 安全属性白名单）→ `SkillTool.call`：
- `context === 'fork'` → `executeForkedSkill`（§6）；
- 否则 inline → `processPromptSlashCommand` → `getMessagesForPromptSlashCommand`。

`getMessagesForPromptSlashCommand`（processSlashCommand.tsx:827）是 inline 展开的核心：

```
result = await command.getPromptForCommand(args, context)   // §4 展开
├─ 若 command.hooks 且允许 → registerSkillHooks(...)         // 注册 skill 自带 hooks
├─ addInvokedSkill(name, skillPath, skillContent, agentId)  // 登记以便 compaction 保留 (§7)
├─ metadata = formatCommandLoadingMetadata(command, args)   // <command-message>/<command-name> 头
├─ getAttachmentMessages(...) { skipSkillDiscovery: true }  // 处理 @-mention（但不让 SKILL.md 触发再发现）
└─ 返回 messages = [
     createUserMessage(metadata),                  // UI 显示 "Skill(name)"
     createUserMessage(result, isMeta),            // 展开后的 skill 正文（注入模型）
     ...attachmentMessages,
     command_permissions attachment                // allowedTools + model
   ]
```

回到 `SkillTool.call`：它把这些 `newMessages` 用 `tagMessagesWithToolUseID` 打标（过滤掉 `<command-message>` 头，因为 SkillTool 自己负责显示），并返回 `contextModifier` 把 `allowedTools` / `model` / `effort` 应用到后续 `ToolUseContext`。inline 路径的 tool_result 只是 `"Launching skill: <name>"` —— 真正内容已作为 `newMessages` 进了对话。

---

## 5. 动态发现与条件激活：Skill 随文件操作出现

Skill 不止在启动时加载。`FileReadTool` / `FileWriteTool` / `FileEditTool` 在操作文件时会**顺带发现新 skill**（fire-and-forget，非阻塞）：

```
// FileWriteTool.ts:234 等
newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd)
if (newSkillDirs.length) {
    dynamicSkillDirTriggers?.add(dir)          // 供 attachment 展示
    addSkillDirectories(newSkillDirs).catch(()=>{})   // 后台加载，不 await
}
activateConditionalSkillsForPaths([fullFilePath], cwd) // 条件 skill 激活
```

两种机制（loadSkillsDir.ts）：

- **`discoverSkillDirsForPaths`（:861）**：从被操作文件的父目录**向上走到 cwd（不含 cwd）**，查找嵌套的 `.claude/skills`。已查过的路径记进 `dynamicSkillDirs` 避免每次 Read 重复 stat。命中的目录还要过 `isPathGitignored`（挡住 `node_modules/**/.claude/skills` 静默加载）。发现后 `addSkillDirectories`（:923）合并进 `dynamicSkills` map（深路径覆盖浅路径），并 `skillsLoaded.emit()` 让缓存失效。
- **`activateConditionalSkillsForPaths`（:997）**：遍历带 `paths` frontmatter 的 `conditionalSkills`，用 `ignore`（gitignore 风格）匹配相对路径；命中就把 skill 从 `conditionalSkills` 移到 `dynamicSkills`，并记进 `activatedConditionalSkillNames`（跨缓存清理存活）。

新激活的 skill 通过 `getDynamicSkills()` 进入 `getCommands`，再经 §3 的增量注入让模型下一轮就能看到。这实现了“**打开某个子项目的文件 → 该子项目的 skill 自动可用**”和“**编辑匹配 glob 的文件 → 相关 skill 才出现**”。

---

## 6. 三种执行模式

`SkillTool.call` 按分支进入不同执行模式（详见 `docs/tool/13-skill-tool.md`）：

| 模式 | 触发 | 结果落地 | Token 上下文 | shell 展开 |
|------|------|----------|-------------|-----------|
| **inline**（默认） | 无 `context: fork` | `newMessages` → 主对话 | 主代理 | 走 `executeShellCommandsInPrompt` |
| **forked** | `context === 'fork'` | 子代理 → `data.result` 摘要 | 独立子代理 | 同 inline（在 prepared context 里展开） |
| **remote** | `_canonical_<slug>` + `EXPERIMENTAL_SKILL_SEARCH` + ant | `newMessages`（isMeta） | 主代理 | 不展开（远端 declarative markdown） |

**forked**（`executeForkedSkill` + `forkedAgent.ts:191 prepareForkedCommandContext`）：`getPromptForCommand` 展开为 `promptMessages` → 选 `command.agent ?? 'general-purpose'` 的 agent 定义 → `runAgent({ ..., override: {agentId}, isAsync: false })` 在隔离上下文跑完 → `extractResultText` 取最后一条 assistant 文本作为结果 → `finally: clearInvokedSkillsForAgent(agentId)`。forked 的价值是把大 skill 的 token 开销隔离出主对话。

用户在 slash 路径也有对应的 `executeForkedSlashCommand`（processSlashCommand.tsx:62），在 assistant/kairos 模式下还能后台 fire-and-forget 并把结果 re-enqueue。

---

## 7. 跨 Compaction 保留

Skill 正文一旦进入对话就占 token，压缩（compaction）时可能被裁掉。为避免“模型正在按某 skill 指令工作，指令却被压没了”，触发时调 `addInvokedSkill`（state.ts:1518）把 `{skillName, skillPath, content, agentId}` 存进 `STATE.invokedSkills`，key 为 `${agentId}:${skillName}`。压缩后由清理逻辑（`postCompactCleanup.ts` / `conversationRecovery.ts`）据此**重新注入 skill 内容**，并按 agentId 过滤，防止跨代理泄漏。forked 路径用完即 `clearInvokedSkillsForAgent` 释放。

---

## 8. 端到端时序（模型触发 inline skill）

```
启动
 └─ registerBundledSkill × N          (bundledSkills[] 填充)
 └─ loadSkillsDir 注册 MCP builders

每轮对话收集 attachment
 └─ getSkillListingAttachments
      → getSkillToolCommands(cwd) 过滤
      → 增量 diff (sentSkillNames)
      → formatCommandsWithinBudget (1% 预算)
      → skill_listing attachment
 └─ messages.ts: <system-reminder> "The following skills are available..."

模型读清单，意图匹配 → 发出 tool_use Skill(skill:"pdf", args:"...")
 └─ SkillTool.validateInput   (存在? prompt型? 非 disableModelInvocation?)
 └─ SkillTool.checkPermissions (deny→allow→安全白名单→ask)
 └─ SkillTool.call
      → context==='fork'? 否 → inline
      → processPromptSlashCommand → getMessagesForPromptSlashCommand
           → getPromptForCommand(args, ctx)
                · Base dir 头 · $ARGUMENTS 替换 · ${CLAUDE_SKILL_DIR}/${SESSION_ID}
                · executeShellCommandsInPrompt (!`cmd` 展开)
           → registerSkillHooks (若有)
           → addInvokedSkill (compaction 保留)
           → 组装 newMessages (metadata + 正文 + attachments + permissions)
      → tagMessagesWithToolUseID (过滤 command-message 头)
      → return { data:{success, commandName}, newMessages, contextModifier }
           contextModifier: allowedTools→alwaysAllowRules / model / effort

主循环把 newMessages 拼进对话 → 模型带着展开的 skill 指令继续本轮
```

用户触发（`/pdf ...`）路径把开头换成 `processSlashCommand` → `getMessagesForSlashCommand`，其余展开逻辑与 inline 完全一致。

---

## 9. 关键设计取舍与不变量

- **渐进披露**：模型只看名字+描述清单（1% 预算 + 250 字符/条上限），正文调用时才加载 —— 用 turn-1 的少量 token 换全量 skill 的可发现性。
- **增量注入 + 按 agent 分桶**：`sentSkillNames` 避免每轮重发整张清单；`--resume` 用 `suppressNext` 防重复；子代理若无 SkillTool 不注入。
- **两条触发路径共用一套展开**：用户 `/` 与模型 `Skill(...)` 都汇入 `getPromptForCommand`，`user-invocable` 与 `disable-model-invocation` 两 flag 正交控制两条路径的可见性。
- **MCP / remote skill 不执行 shell**：远端内容不可信，`loadedFrom==='mcp'` 与 remote 路径跳过 `executeShellCommandsInPrompt`，只做变量替换。
- **文件身份去重**：`realpath` 解 symlink，避免同一 SKILL.md 经不同路径/软链被加载多次；first-wins 保留来源优先级。
- **动态发现非阻塞**：`addSkillDirectories` 不 await，文件操作不因 skill 加载卡顿；已查目录缓存避免重复 stat；gitignore 挡住依赖目录里的 skill。
- **条件 skill 惰性激活**：带 `paths` 的 skill 直到匹配文件被触碰才进入可用列表 —— 减少无关 skill 对清单预算的占用。
- **compaction 保留按 agentId 过滤**：`invokedSkills` 以 `agentId:name` 为 key，压缩后重注入不跨代理泄漏；forked 用完即清。
- **forked 隔离 token**：`context: fork` 让重型 skill 跑在独立子代理，结果以摘要回主对话，保护主上下文预算。

---

## 10. 关联模块速查

| 模块 | 职责 |
|------|------|
| `src/skills/loadSkillsDir.ts` | 磁盘发现、frontmatter 解析、`createSkillCommand`、动态/条件 skill |
| `src/skills/bundledSkills.ts` | `registerBundledSkill` / `getBundledSkills`，附带文件懒解压 |
| `src/skills/mcpSkillBuilders.ts` | 打破 import 环的 write-once registry（给 MCP skill 发现用 builders） |
| `src/commands.ts` | `loadAllCommands` / `getCommands` / `getSkillToolCommands` / `getSlashCommandToolSkills` / `getSkills` |
| `src/tools/SkillTool/SkillTool.ts` | 模型调用入口：validate / checkPermissions / call（inline/forked/remote） |
| `src/tools/SkillTool/prompt.ts` | `getPrompt`（工具描述）、`formatCommandsWithinBudget`（清单预算裁剪） |
| `src/utils/attachments.ts` | `getSkillListingAttachments`（增量清单注入）、动态 skill attachment |
| `src/utils/messages.ts` | `skill_listing` → `<system-reminder>` 包裹 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `processSlashCommand` / `processPromptSlashCommand` / `getMessagesForPromptSlashCommand` / `formatSkillLoadingMetadata` / forked slash |
| `src/utils/forkedAgent.ts` | `prepareForkedCommandContext` / `extractResultText` |
| `src/tools/AgentTool/runAgent.ts` | forked skill 的子代理 query 循环；frontmatter skill 预加载 |
| `src/tools/File{Read,Write,Edit}Tool` | 文件操作触发 `discoverSkillDirsForPaths` / `activateConditionalSkillsForPaths` |
| `src/bootstrap/state.ts` | `addInvokedSkill` / `getInvokedSkillsForAgent` / `getSessionId`（compaction 保留） |
| `src/utils/argumentSubstitution.ts` | `substituteArguments`（`$ARGUMENTS` / `$1`） |
| `src/utils/promptShellExecution.ts` | `executeShellCommandsInPrompt`（`!command` 注入） |
| `src/utils/frontmatterParser.ts` | `parseFrontmatter` 及各字段解析 |
