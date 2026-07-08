# SkillTool — Skill / Slash Command 调度器

> 源码位置：`src/tools/SkillTool/SkillTool.ts`（1110 行）
> 关联模块：`UI.tsx`、`prompt.ts`、`constants.ts`、`utils/processUserInput/processSlashCommand.tsx`、`tools/AgentTool/runAgent.ts`
> 工具名：`SKILL_TOOL_NAME`

`SkillTool` 是 Claude Code 中执行 `Skill(...)` tool_use 的入口，也是把 `/skill` slash command 在模型驱动下重新触发的统一通道。它承担三种执行模式：**inline**（默认，把 skill 内容展开为用户消息插入当前对话）、**forked**（`context: 'fork'` 触发，把 skill 跑在隔离的子代理里）、**remote**（`EXPERIMENTAL_SKILL_SEARCH` + ant-only，从远端 GCS/S3/HTTP 拉取 SKILL.md 后注入）。同时它把 allow/deny/ask 权限决策、tengu 遥测、`!command` 与 `$ARGUMENTS` 展开、远程 skill 缓存、compaction 保留都收敛到一处。

---

## 主要完成的工作

| 主题 | 实现要点 |
|------|----------|
| 工具定义 | `buildTool` 工厂构建：`inputSchema = { skill, args? }`；`outputSchema = union(inlineOutput, forkedOutput)`；`prompt` 走 `getPrompt(getProjectRoot())` 动态生成 skill 列表 |
| 命令发现 | `getAllCommands(context)`：本地 `getCommands(getProjectRoot())` ∪ MCP `cmd.type === 'prompt' && cmd.loadedFrom === 'mcp'`（用 `uniqBy('name')` 去重）。注释里强调只放 MCP skills（不放大写 plain MCP prompts）—— 否则模型能猜到 `mcp__server__prompt` 名绕过 discoverability 限制 |
| 三种执行模式 | **inline**（默认，`processPromptSlashCommand` 展开 skill 内容为 user messages 并附加 contextModifier）；**forked**（`command.context === 'fork'` 触发 `executeForkedSkill` 调 `runAgent`）；**remote**（`EXPERIMENTAL_SKILL_SEARCH` + `USER_TYPE === 'ant'` + `_canonical_<slug>` 命名空间触发 `executeRemoteSkill`） |
| validateInput | trim → 剥前导 `/` → 剥 canonical 前缀 → 验证已发现 → `findCommand` → 拒绝 `disableModelInvocation` 的 skill → 拒绝非 `prompt` 类型 skill；每种失败带 `errorCode`（1/2/4/5/6）方便模型理解 |
| checkPermissions | 6 步优先级：deny rule 拒绝 → canonical remote auto-grant（在 deny 之后，保留用户配置拒绝语义）→ allow rule 通过 → 安全属性白名单自动通过 → `ask`（带两条建议：exact + `:*` 前缀） |
| 安全属性白名单 | `SAFE_SKILL_PROPERTIES` Set：列出了所有"无副作用"的 `PromptCommand` 属性；任何不在表里且值非空的属性都让 skill 走 `ask`。这保证未来新增属性默认需权限 |
| Leading slash 兼容 | 接受 `/commit` 与 `commit` 两种写法；前者额外 `logEvent('tengu_skill_tool_slash_prefix')` 观测模型是否习惯用 `/` 风格 |
| 前导规则匹配 | `ruleMatches(ruleContent)`：剥前导 `/` 后做 exact match 与 `prefix:*` 前缀 match（如 `review:*` 命中 `review-pr 123`），让 `Skill(review:*)` 规则生效 |
| 进度透传 | inline 模式无进度（一次展开）；forked 模式 `onProgress({ type: 'skill_progress', agentId, prompt: skillContent })` 把 runAgent 的工具使用透传给父 UI |
| forked 路径上下文注入 | `executeForkedSkill` 调 `prepareForkedCommandContext(command, args, context)` 拿到 modifiedGetAppState/baseAgent/promptMessages/skillContent，把 `command.effort` 合并到 `agentDefinition.effort`，`override.agentId` 让 runAgent 用预生成的 agentId |
| forked 内存释放 | `agentMessages.length = 0`（提取完 resultText 后立即释放）；`finally` 调 `clearInvokedSkillsForAgent(agentId)` 防止 invokedSkills 全局 map 累积 |
| remote skill 加载 | `executeRemoteSkill`：`loadRemoteSkill(slug, url)` 从 GCS/S3/HTTP 拉取（含本地缓存），剥 YAML frontmatter，注入 `Base directory for this skill:` 头 + `${CLAUDE_SKILL_DIR}` + `${CLAUDE_SESSION_ID}` 替换，`addInvokedSkill` 注册到 compaction-preservation 状态，最后 `createUserMessage({ content: finalContent, isMeta: true })` 直接注入 |
| contextModifier | 链式修改 `ToolUseContext`：allowedTools → `alwaysAllowRules.command`（用 `Set` 去重）；model → `resolveSkillModelOverride`（保留 `[1m]` 后缀防止 opusplan→200K 触发 autocompact）；effort → `state.effortValue` |
| 遥测归一 | 三个 execution_context：`inline` / `fork` / `remote`；`_PROTO_skill_name` 路由到 PII-tagged BQ 列（全量、无脱敏），`command_name` 走脱敏 `additional_metadata`；`_PROTO_plugin_name` / `_PROTO_marketplace_name` 同理；`was_discovered` 来自 `context.discoveredSkillNames`；remote 额外带 `is_remote` / `remote_cache_hit` / `remote_load_latency_ms` |
| 嵌套 skill 检测 | `queryDepth > 0` 时 `invocation_trigger = 'nested-skill'`，否则 `claude-proactive`；`parent_agent_id` 来自 `getAgentContext()?.agentId` |
| 安全 plugin 分类 | `isOfficialMarketplaceSkill` 判断后，非 official 的 plugin `plugin_name` / `plugin_repository` 都标 `'third-party'` 保护 telemetry BQ 中下游聚合不被第三方污染 |
| Skill 注册幂等 | 注释明确：`addInvokedSkill` 和 `registerSkillHooks` 在 `processPromptSlashCommand → getMessagesForPromptSlashCommand` 内已调用，本文件不再调，避免 double-register hooks 与重建 skillContent |
| newMessages 过滤 | inline 路径用 `tagMessagesWithToolUseID` 但要先过滤：剥 `progress`、剥含 `<COMMAND_MESSAGE_TAG>` 的 user message（SkillTool 自己负责显示 command 头，不该再带一份） |
| 失败埋点 | remote 加载失败时 `logRemoteSkillLoaded({ ..., error })` 记录 cache 状态与 latencyMs 后再 throw —— 让 telemetry 能区分 cache hit/miss/error 路径 |
| `toAutoClassifierInput` | 返回 `skill ?? ''`（注释解释：skill-coach 需要 skill 名避免 false-positive "you could have used skill X"，backseat classifier 只关心是否触发，不展开 prompt） |

---

## 核心执行流程

`SkillTool.call(input, context, canUseTool, parentMessage, onProgress?)` 是核心入口。下图按调用顺序铺平三种模式的分支决策：

```text
                ┌────────────────────────────────────────────────┐
                │ validateInput(input, context) — 预检           │
                │   1. trim + 剥前导 /                           │
                │   2. 远程 canonical 拦截 (EXPERIMENTAL_SKILL_SEARCH│
                │      + ant-only): slug 不存在 → errorCode 6    │
                │   3. findCommand → 不存在 → errorCode 2         │
                │   4. disableModelInvocation → errorCode 4       │
                │   5. type !== 'prompt' → errorCode 5            │
                └─────────────────────────┬──────────────────────┘
                                          │
                ┌─────────────────────────▼──────────────────────┐
                │ checkPermissions(input, context)              │
                │   1. deny rule (ruleMatches) → behavior:'deny'│
                │   2. canonical remote (在 deny 之后) → 'allow' │
                │   3. allow rule (ruleMatches) → 'allow'        │
                │   4. SAFE_SKILL_PROPERTIES 白名单 → 'allow'    │
                │   5. 默认 → 'ask' + 两条建议（exact + :* 前缀）│
                └─────────────────────────┬──────────────────────┘
                                          │
                ┌─────────────────────────▼──────────────────────┐
                │ call(input, context, canUseTool, parent)       │
                └─────────────────────────┬──────────────────────┘
                                          │
        ┌─────────────────────────────────┼────────────────────────────────────┐
        │                                 │                                    │
   remote canonical                   forked path                       inline path
   (_canonical_<slug>)                (context === 'fork')                (default)
        │                                 │                                    │
        ▼                                 ▼                                    ▼
executeRemoteSkill                  executeForkedSkill                  recordSkillUsage
  ├─ loadRemoteSkill                 ├─ recordSkillUsage                 ├─ processPromptSlashCommand
  ├─ logRemoteSkillLoaded           ├─ prepareForkedCommandContext      │  ├─ findCommand
  ├─ 剥 frontmatter + 注入 dir      ├─ agentDefinition =               │  ├─ getMessagesForPromptSlashCommand
  ├─ 替换 CLAUDE_SKILL_DIR/ID       │    command.effort merged in       │  │  ├─ 加载 SKILL.md / 用户定义
  ├─ addInvokedSkill(compaction 保留)│ ├─ runAgent({                    │  │  ├─ executeShellCommandsInPrompt
  └─ createUserMessage(isMeta)       │     agentDefinition,             │  │  │  (展开 !`command` 和 ```!command```)
                                     │     promptMessages,               │  │  ├─ substituteArguments ($ARGUMENTS)
                                     │     override: {agentId},          │  │  └─ addInvokedSkill + registerSkillHooks
                                     │     isAsync: false,                │  └─ 返回 messages + allowedTools + model
                                     │     querySource: 'agent:custom',  │
                                     │     availableTools:               │
                                     │       context.options.tools        │
                                     │   })                              │
                                     │ ├─ onProgress({skill_progress})  │
                                     │ ├─ extractResultText              │
                                     │ ├─ agentMessages.length = 0       │
                                     │ └─ finally: clearInvokedSkills    │
                                     ▼                                    ▼
                          ┌────────────────────────────────────────────────┐
                          │ return { data, newMessages?, contextModifier } │
                          │   data: {success, commandName,                  │
                          │          status: 'forked'|'inline',            │
                          │          allowedTools?, model?, agentId?,      │
                          │          result?}                              │
                          │   newMessages: tagMessagesWithToolUseID(...)  │
                          │   contextModifier:                             │
                          │     ├─ allowedTools → alwaysAllowRules.command │
                          │     ├─ model → mainLoopModel 保留 [1m] 后缀    │
                          │     └─ effort → state.effortValue             │
                          └────────────────────────────────────────────────┘
```

### mapToolResultToToolResultBlockParam（forked vs inline 分流）

```text
Output
  ├─ status === 'forked'  → tool_result 文本 "Skill \"$name\" completed (forked execution).\n\nResult:\n$result"
  └─ inline (默认)
       └─ tool_result 文本 "Launching skill: $commandName"
```

注意：inline 路径返回的 `data` 不直接包含结果，结果在 `newMessages` 里（已经拼进 conversation）；forked 路径返回 `data.result`，因为 forked 跑在子代理里，结果不会自动入主对话 —— 模型要看到摘要得靠这段 tool_result 文本来 bridge。

### Forked 路径与 Inline 路径的核心差异

| 维度 | Inline | Forked | Remote |
|------|--------|--------|--------|
| 触发条件 | 默认 | `command.context === 'fork'` | `_canonical_<slug>` 前缀 + experimental flag |
| 结果落地 | newMessages → 主对话 | 子代理 → `data.result` | newMessages → 主对话（isMeta） |
| Token 上下文 | 用主代理上下文 | 独立子代理上下文 | 用主代理上下文 |
| `!command` 展开 | 走 `executeShellCommandsInPrompt` | 同 inline（forked 也用 prepared context） | 不展开（remote 是 declarative markdown） |
| `$ARGUMENTS` 替换 | 走 `substituteArguments` | 同 inline | 替换 `${CLAUDE_SKILL_DIR}` + `${CLAUDE_SESSION_ID}` |
| 进度回调 | 无（一次展开完成） | `onProgress({type: 'skill_progress'})` 透传 runAgent 工具使用 | 无 |
| Compaction 保留 | `processSlashCommand` 内部 `addInvokedSkill` | forked 路径 `clearInvokedSkillsForAgent(agentId)` finally 清理 | `addInvokedSkill` 注册后由系统保留 |
| 权限语义 | 白名单自动 allow / ask | 同 inline（权限检查在 call 之前完成） | remote canonical ant-only auto-allow（在 deny 之后） |

---

## 关键设计取舍

- **三种模式共享 call() 入口**：inline / forked / remote 在 `call()` 里按顺序判断分支，但每种模式都做相同的 `recordSkillUsage` + 自身执行 + `tengu_skill_tool_invocation` 遥测 —— 观测一致。
- **canonical remote auto-grant 放在 deny 之后**：用户配置的 `Skill(_canonical_:*)` deny 规则仍然生效（防 curated skill 误用），但 default 行为是 auto-grant。注释明确说"same pattern as safe-properties auto-allow below" —— 保持规则优先级的一致性。
- **安全属性白名单 vs allow list**：`SAFE_SKILL_PROPERTIES` 是 allowlist，不是 blocklist —— 未来 `PromptCommand` 新增任何属性，默认就走 `ask`，不会因为漏写 deny 规则而静默放行。注释明确指出这是有意为之。
- **`!command` 与 `$ARGUMENTS` 不在 forked 路径独立实现**：`executeForkedSkill` 调 `prepareForkedCommandContext`（forkedAgent.ts）拿到 `promptMessages`，那里已经走过展开；本文件不再重复。
- **inline 路径不调用 runAgent**：inline 路径只是把 skill 内容展开为 user messages 插入主对话，没有独立子代理；token 上下文共享主代理。forked 路径才走 runAgent（独立 agentId、独立 abortController）。
- **remote canonical 不带 shell 展开**：注释明确说"remote skills are declarative markdown so no slash-command expansion (no !command substitution, no $ARGUMENTS interpolation) is needed"。避免在远端内容里执行命令的 security risk。
- **forked 内存释放双保险**：循环结束后 `agentMessages.length = 0` 立即释放子代理消息数组；finally 再 `clearInvokedSkillsForAgent(agentId)` 清空 invokedSkills 状态。
- **Leading slash 兼容**：注释说"for compatibility" —— 老 transcript 可能用 `/commit`，新 model 可能用 `commit`；本文件同时接受。
- **`toAutoClassifierInput` 返回纯 skill 名**：不展开 prompt（inline 路径 prompt 会进 `processedCommand.messages`，但 classifier 不需要完整 prompt，只需要知道"这是 X skill"，避免 auto-mode 分类器把 skill 内部内容误判为不安全）。
- **PII vs 脱敏双通道**：`_PROTO_*` 走 BQ PII-tagged 列（全量）；`command_name` / `plugin_name` 走 `additional_metadata`（脱敏后给 general-access dashboard）。注释明确说明这条路由规则，避免下游误用。
- **official plugin vs third-party plugin 区分**：`isOfficialMarketplaceSkill` 后，非 official 的 `plugin_name` / `plugin_repository` 都标 `'third-party'` —— BQ 聚合时能过滤掉 third-party 噪音，只看 official 插件的真实使用情况。
- **newMessages 过滤 COMMAND_MESSAGE_TAG**：`processPromptSlashCommand` 产出的 user message 里会带 `<COMMAND_MESSAGE_TAG>` 头（让 SkillTool UI 知道这是 command 来源）；本文件需要把这个 tag 剥掉再 `tagMessagesWithToolUseID` —— 否则 UI 会显示 command 头两次。

---

## 调用与被调用全景

```text
模型发出 tool_use: { name: SKILL_TOOL_NAME, input: { skill, args? } }
  ↓ runToolUse (toolExecution.ts)
    ├─ validateInput (预检)
    ├─ checkPermissions (allow/deny/ask)
    └─ call(input, context, canUseTool, parentMessage, onProgress)
       │
       ├─ remote canonical 拦截 (EXPERIMENTAL_SKILL_SEARCH + ant-only)
       │  └─ executeRemoteSkill
       │     ├─ remoteSkillModules.loadRemoteSkill(slug, meta.url)
       │     ├─ parseFrontmatter(content, skillPath) 剥 YAML
       │     ├─ 注入 "Base directory for this skill: $dir" + 替换 CLAUDE_SKILL_DIR/ID
       │     ├─ addInvokedSkill(commandName, skillPath, finalContent, agentId)
       │     └─ createUserMessage({ content: finalContent, isMeta: true })
       │
       ├─ forked path (command.context === 'fork')
       │  └─ executeForkedSkill
       │     ├─ prepareForkedCommandContext(command, args, context)
       │     │   └─ forkedAgent.ts: skillContent + modifiedGetAppState + baseAgent + promptMessages
       │     └─ runAgent({ agentDefinition, promptMessages, override: {agentId}, isAsync: false })
       │        └─ query(...) → 消息流 → extractResultText → clear memory
       │
       └─ inline path (default)
          └─ processPromptSlashCommand → getMessagesForPromptSlashCommand
             ├─ findCommand (commands.ts)
             ├─ loadSkillsDir / loadPluginCommands / loadBundledSkills (按 command.source)
             ├─ skill.getPromptForCommand(args, context) 加载 SKILL.md
             ├─ executeShellCommandsInPrompt 展开 !`cmd` 和 ```!cmd```
             ├─ substituteArguments 替换 $ARGUMENTS / $1 / ${CLAUDE_SESSION_ID}
             ├─ addInvokedSkill (compaction 保留)
             ├─ registerSkillHooks (skill 自带 hooks)
             └─ 返回 messages + allowedTools + model + effort
          
          return { data: {success, commandName, allowedTools?, model?},
                   newMessages: tagMessagesWithToolUseID(processedMessages),
                   contextModifier: 链式修改 allowedTools/model/effort }
```

UI 渲染：
```text
renderToolUseMessage(input)        ← 显示 /skill 或 skill 名
renderToolUseProgressMessage(...)  ← 只在 forked 路径触发（inline 没 progress）
                                     显示子代理工具使用折叠转写
renderToolResultMessage(output)    ← forked → 'Done' / inline → 'Successfully loaded skill (N tools allowed, model)'
renderToolUseRejectedMessage(...)  ← renderToolUseProgressMessage + FallbackToolUseRejectedMessage
renderToolUseErrorMessage(...)     ← 同 rejected 但用 FallbackToolUseErrorMessage
```

---

## 关联模块速查

| 模块 | 行数 | 职责 |
|------|-----:|------|
| `src/tools/SkillTool/SkillTool.ts` | 1110 | 本文件：三种执行模式入口、validate/checkPermissions/call、contextModifier、tengu 遥测 |
| `src/tools/SkillTool/UI.tsx` | — | `renderToolResultMessage` / `renderToolUseMessage` / `renderToolUseProgressMessage` / `renderToolUseRejectedMessage` / `renderToolUseErrorMessage` |
| `src/tools/SkillTool/prompt.ts` | — | `getPrompt(getProjectRoot())` 动态生成 SkillTool 的描述（可用 skill 列表、用例） |
| `src/tools/SkillTool/constants.ts` | — | `SKILL_TOOL_NAME` 工具名常量 |
| `src/utils/processUserInput/processSlashCommand.tsx` | — | `processPromptSlashCommand` / `getMessagesForPromptSlashCommand` / `executeForkedSlashCommand`；SKILL.md 加载、`!command` 展开、`$ARGUMENTS` 替换、addInvokedSkill、registerSkillHooks |
| `src/utils/promptShellExecution.ts` | — | `executeShellCommandsInPrompt`（解析 ```!command``` 和 !`command`，调 Bash/PowerShell） |
| `src/utils/frontmatterParser.ts` | — | `parseFrontmatter`（剥 YAML frontmatter） |
| `src/utils/forkedAgent.ts` | — | `prepareForkedCommandContext`（forked 路径上下文准备）、`extractResultText` |
| `src/tools/AgentTool/runAgent.ts` | 976 | forked skill 调用的子代理 query 循环（详见 `12-run-agent.md`） |
| `src/commands.ts` | — | `findCommand` / `getCommands` / `builtInCommandNames` / 命令注册表（含 lazy skills / plugin / bundled） |
| `src/types/command.ts` | — | `PromptCommand`（含 `context: 'inline' \| 'fork'` 字段）、`Command`、`CommandBase` |
| `src/skills/loadSkillsDir.ts` | — | 从 `~/.claude/skills/` 加载 SKILL.md（lazy + dynamic） |
| `src/skills/bundledSkills.ts` | — | bundled skills 加载 |
| `src/utils/plugins/loadPluginCommands.ts` | — | 插件提供的 skill 加载 |
| `src/utils/plugins/pluginIdentifier.ts` | — | `parsePluginIdentifier` / `isOfficialMarketplaceName`（plugin 解析） |
| `src/utils/telemetry/pluginTelemetry.ts` | — | `buildPluginCommandTelemetryFields`（plugin 字段归一） |
| `src/utils/permissions/permissions.ts` | — | `getRuleByContentsForTool`（按 rule content 索引 allow/deny 规则） |
| `src/services/analytics/index.ts` | — | `logEvent` + `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` / `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` 类型 |
| `src/services/skillSearch/remoteSkillLoader.ts` | — | `loadRemoteSkill`（从 GCS/S3/HTTP 拉 SKILL.md，含本地缓存）+ `getDiscoveredRemoteSkill` |
| `src/services/skillSearch/remoteSkillState.ts` | — | `stripCanonicalPrefix` + `isSkillSearchEnabled`（session-scoped 状态） |
| `src/services/skillSearch/telemetry.ts` | — | `logRemoteSkillLoaded`（cache hit/miss + latency 埋点） |
| `src/services/skillSearch/featureCheck.ts` | — | feature flag 检查 |
| `src/bootstrap/state.ts` | — | `addInvokedSkill` / `clearInvokedSkillsForAgent` / `getSessionId` |
| `src/constants/xml.ts` | — | `COMMAND_MESSAGE_TAG`（user message 中标识 command 源） |
| `src/utils/suggestions/skillUsageTracking.ts` | — | `recordSkillUsage`（skill 使用频次统计） |
| `src/utils/agentContext.ts` | — | `getAgentContext()`（嵌套 agent 上下文） |
| `src/utils/model/model.ts` | — | `resolveSkillModelOverride`（保留 opus[1m] 后缀） |
| `src/utils/uuid.ts` | — | `createAgentId`（forked 路径预生成 agentId） |
| `src/tools/utils.ts` | — | `getToolUseIDFromParentMessage` / `tagMessagesWithToolUseID` |
| `src/utils/messages.ts` | — | `createUserMessage` / `normalizeMessages` |

---

## 重要不变量

- **三种模式都走 `tengu_skill_tool_invocation` 埋点**：`execution_context` 区分 `'inline' / 'fork' / 'remote'` —— BQ 聚合时能精确分流。
- **canonical remote auto-grant 放在 deny 之后**：用户配置的 `Skill(_canonical_:*)` deny 仍然生效。
- **inline 路径不调 `runAgent`**：inline 把 skill 内容展开为 newMessages 直接插入主对话；只有 `context === 'fork'` 才走 `executeForkedSkill → runAgent`。
- **remote 路径不做 shell 展开**：注释明确说"remote skills are declarative markdown" —— `${CLAUDE_SKILL_DIR}` / `${CLAUDE_SESSION_ID}` 替换是必要的（让模型解析相对路径），但 `!command` / `$ARGUMENTS` 不展开（防远端内容执行命令）。
- **forked 路径独立 agentId**：`override.agentId` 让 runAgent 用预生成的 agentId；forked 路径的 `querySource: 'agent:custom'` 让它独立于 builtin agent 路由。
- **invoke 幂等性**：`addInvokedSkill` 和 `registerSkillHooks` 在 `processPromptSlashCommand` 内部已调用，本文件 inline 路径不再调，避免 double-register。
- **forked 路径的 finally 清理**：clearInvokedSkillsForAgent(agentId) 必须在 finally（防止 invoke 中途抛错导致状态泄漏）。`agentMessages.length = 0` 在 finally 之前立即释放内存。
- **leading slash 双写兼容**：`/commit` 和 `commit` 都接受，剥前导 `/` 后做规则匹配；带 `/` 时额外打 `tengu_skill_tool_slash_prefix` 观测埋点。
- **`ruleMatches` 前缀 match**：`Skill(review:*)` 规则命中 `review-pr 123` 等带空格 args 的 skill —— 允许按前缀批量授权一个 skill family。
- **`SAFE_SKILL_PROPERTIES` 是 allowlist**：任何不在表里且有非空值的属性都让 skill 走 `ask`，未来 `PromptCommand` 新增属性默认需权限。
- **`toAutoClassifierInput` 返回 skill 名而不展开 prompt**：skill-coach 需要 skill 名避免 false-positive "you could have used skill X"，backseat classifier 只关心是否触发。
- **`newMessages` 过滤 `COMMAND_MESSAGE_TAG`**：本文件 inline 路径剥掉含 `<COMMAND_MESSAGE_TAG>` 的 user message —— SkillTool UI 已经显示 command 来源，不该再带一份。
- **`contextModifier` 链式应用**：`allowedTools` 用 `Set` 去重；`model` 用 `resolveSkillModelOverride` 保留 opus[1m] 后缀防止 opusplan→200K 触发 autocompact；`effort` 覆盖 `state.effortValue`。
- **PII vs 脱敏双通道**：`_PROTO_skill_name` / `_PROTO_plugin_name` / `_PROTO_marketplace_name` 走 PII-tagged BQ 列（全量无脱敏），`command_name` / `plugin_name` / `plugin_repository` 走 `additional_metadata`（脱敏）。
- **official plugin vs third-party plugin 区分**：非 official 的 plugin `plugin_name` / `plugin_repository` 都标 `'third-party'` 保护 BQ 聚合。
