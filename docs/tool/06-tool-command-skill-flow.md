# 工具、命令、Skill 加载与执行流程

## 概述

Claude Code 中有三个核心概念：**Tool（工具）**、**Command（命令/斜杠命令）**、**Skill（技能）**。

- **Tool**：提供给 LLM 调用的工具（Bash、Read、Edit 等），通过 API `tool_use` 机制执行。
- **Command**：用户通过 `/command` 触发的斜杠命令，分为三种类型：`prompt`（提示词扩展）、`local`（本地函数）、`local-jsx`（本地 React UI）。
- **Skill**：一种特殊类型的 Command（`type: 'prompt'`），可从文件系统、插件、MCP 服务器或 CLI 内置加载，既可由用户以 `/skill-name` 触发，也可由模型通过 `SkillTool` 调用。

---

## 一、Tool 系统

### 1.1 Tool 定义

Tool 类型定义在 `src/Tool.ts:362`，是一个接口：

```typescript
type Tool<Input, Output, P> = {
  name: string
  aliases?: string[]
  searchHint?: string
  inputSchema: Input          // Zod schema
  outputSchema?: z.ZodType    // 输出类型
  description(input, opts): Promise<string>
  prompt(options): Promise<string>
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  maxResultSizeChars: number
  // ... 渲染方法
}
```

`buildTool()` 函数（`src/Tool.ts:783`）为工具定义提供默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: () => ({ behavior: 'allow', updatedInput: input }),
  userFacingName: () => name,
}
```

### 1.2 Tool 注册

所有内置工具在 `src/tools.ts:196` 的 `getAllBaseTools()` 中注册：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    GlobTool, GrepTool,              // hasEmbeddedSearchTools 时排除
    FileReadTool, FileEditTool, FileWriteTool, NotebookEditTool,
    WebFetchTool, TodoWriteTool, WebSearchTool, TaskStopTool,
    AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    // ...条件性工具（ConfigTool, TungstenTool, WorkflowTool 等）
  ]
}
```

`getTools()` 函数（`src/tools.ts:276`）根据权限上下文过滤工具：

```
getAllBaseTools()
  → filterToolsByDenyRules()  // 移除被拒绝的工具
  → filter(isEnabled)         // 只保留启用的工具
```

### 1.3 Tool 池组装

`assembleToolPool()`（`src/tools.ts:350`）将内置工具与 MCP 工具合并：

```
getTools(permissionContext)     // 获取内置工具
  → filterToolsByDenyRules(mcpTools)  // 过滤 MCP 工具
  → sort + uniqBy(name)               // 按名称排序去重（内置优先）
```

### 1.4 Tool 执行流程

在 `src/services/tools/toolExecution.ts:337` 的 `runToolUse()` 中：

```
1. findToolByName(toolName, tools)       // 在可用工具中查找
   ↑ 未找到时，检查别名 fallback
2. 权限检查上下文（MCP 服务器类型、URL）
3. PermissionResult 检查（checkPermissions）
4. tool.call(input, context, ...)        // 执行工具
5. mapToolResultToToolResultBlockParam()  // 序列化结果
```

### 1.5 Tool 延迟加载（Deferred Tool）

通过 `ToolSearch` 系统（`src/utils/toolSearch.ts`）实现：

- 标记 `shouldDefer: true` 的工具在初始提示中不发送完整 Schema
- 模型通过 `ToolSearch` 搜索后才可使用
- `getDeferredToolsDelta()` 跟踪已声明/已移除的延迟工具

---

## 二、Command 系统

### 2.1 Command 定义

Command 类型定义在 `src/types/command.ts:205`，有三种类型：

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**三种类型：**

| 类型 | 描述 | 加载方式 | 示例 |
|------|------|----------|------|
| `prompt` | 提示词扩展 | 返回 ContentBlockParam[] | `/skills`, `/commit` |
| `local` | 本地函数 | `load()` → `call(args, context)` | `/compact`, `/clear` |
| `local-jsx` | 本地 React UI | `load()` → `call(onDone, context, args)` | `/config`, `/help` |

**CommandBase 核心字段：**

```typescript
type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  isEnabled?: () => boolean
  isHidden?: boolean
  userInvocable?: boolean      // 是否可由用户通过 /name 触发
  disableModelInvocation?: boolean  // 是否禁止模型通过 SkillTool 调用
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
}
```

### 2.2 Command 注册

在 `src/commands.ts:258` 中，所有命令通过静态导入和 `COMMANDS` 列表注册：

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir, agents, clear, compact, config, help, mcp, memory,
  skills, plan, review, session, theme, ...
  // 条件性命令（proactive, bridge, voiceCommand 等）
])
```

`builtInCommandNames` 快照所有内置命令名称：

```typescript
export const builtInCommandNames = memoize((): Set<string> =>
  new Set(COMMANDS().flatMap(_ => [_.name, ...(_.aliases ?? [])]))
)
```

### 2.3 Command 加载流程

`getCommands()`（`src/commands.ts:477`）是命令加载的入口：

```
getCommands(cwd)
  → loadAllCommands(cwd)       // 备忘录化，高效加载
     → getSkills(cwd)          // 并行加载技能来源
       → getSkillDirCommands(cwd)   // 文件系统技能
       → getPluginSkills()          // 插件技能
       → getBundledSkills()         // 内置打包技能（同步）
       → getBuiltinPluginSkills()   // 内置插件技能
     → getPluginCommands()          // 插件命令
     → getWorkflowCommands(cwd)     // 工作流命令
     → COMMANDS()                   // 内置命令
  → 合并: [bundled, builtinPlugin, skillDir, workflow, plugin, builtin]
  → meetsAvailabilityRequirement()  // 过滤可用性
  → isCommandEnabled()               // 过滤启用状态
  → 合并 dynamicSkills               // 动态发现的技能
```

### 2.4 斜杠命令处理流程

用户输入 `/command` 时，在 `src/utils/processUserInput/processSlashCommand.tsx:342` 处理：

```
processSlashCommand(inputString, ...)
  → parseSlashCommand(inputString)     // 解析 /command [args] 格式
  → hasCommand(commandName, commands)  // 检查命令是否存在
  → getMessagesForSlashCommand(...)     // 按类型分发
    → type 'local-jsx':                // 加载模块 → 渲染 React UI
      → command.load() → mod.call(onDone, context, args)
    → type 'local':                    // 加载模块 → 执行函数
      → command.load() → mod.call(args, context)
    → type 'prompt':                   // 展开提示词
      → command.context === 'fork'?
        → executeForkedSlashCommand()  // 子代理执行
        → getMessagesForPromptSlashCommand()  // 内联展开
          → command.getPromptForCommand(args, context)
          → 插入消息: [metadata, skillContent, attachments, permissions]
```

---

## 三、Skill 系统

### 3.1 Skill 的本质

Skill 是 `type: 'prompt'` 的 Command，具有以下特点：

- 内容是 Markdown 提示词，执行时展开为 `ContentBlockParam[]`
- 支持参数替换（`$ARGUMENTS` 或命名参数）
- 支持 Shell 命令执行（`` `!command` `` 语法）
- 支持权限控制（allowedTools）
- 支持执行上下文：`inline`（当前对话）或 `fork`（子代理）

### 3.2 Skill 的来源

#### 3.2.1 打包技能（Bundled Skills）

在 `src/skills/bundledSkills.ts` 中定义，通过 `registerBundledSkill()` 注册：

```typescript
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  // 处理 files 字段（提取参考文件到磁盘）
  // 创建 Command 对象，push 到 bundledSkills[]
}

export function getBundledSkills(): Command[] {
  return [...bundledSkills]  // 返回副本防止外部修改
}
```

初始化在 `src/main.tsx:2172` 调用 `initBundledSkills()`，见 `src/skills/bundled/index.ts`：

```typescript
export function initBundledSkills(): void {
  registerUpdateConfigSkill()     // /update-config
  registerKeybindingsSkill()      // /keybindings-help
  registerVerifySkill()           // /verify
  registerDebugSkill()            // /debug
  registerLoremIpsumSkill()       // /lorem-ipsum
  registerSkillifySkill()         // /skillify
  registerRememberSkill()         // /remember
  registerSimplifySkill()         // /simplify
  registerBatchSkill()            // /batch
  registerStuckSkill()            // /stuck
  registerClaudeApiSkill()        // /claude-api
  registerLoopSkill()             // /loop
  // ... 条件性注册
}
```

#### 3.2.2 文件系统技能（File-based Skills）

在 `src/skills/loadSkillsDir.ts` 中实现，从目录加载：

**加载目录（按优先级）：**

```
1. 管理策略目录:  <managed-path>/.claude/skills/
2. 用户目录:      ~/.claude/skills/
3. 项目目录:      <project>/.claude/skills/   （从项目根到 CWD 的所有层级）
4. 附加目录:      --add-dir 指定的目录
5. 遗留命令目录:  <config>/.claude/commands/   （已废弃）
```

**文件格式：**

- `/skills/` 目录：只支持 `skill-name/SKILL.md` 目录格式
- `/commands/` 目录（遗留）：支持 `SKILL.md` 目录格式和单 `.md` 文件格式

**SKILL.md 格式：**

```markdown
---
name: My Skill
description: 技能描述
when_to_use: 使用场景
arguments: <arg1> [arg2]
allowed-tools: [Bash, Read]
context: fork          # inline（默认）| fork
agent: general-purpose # 子代理类型
model: claude-sonnet-4-20250514
effort: high
user-invocable: true
paths: ["src/**/*.ts"]  # 条件性技能
---

# 技能内容

使用 $ARGUMENTS 或 {{arg1}} 引用参数。

!`echo "这是 Shell 命令"`
```

**加载流程：**

```
loadSkillsFromSkillsDir(basePath, source)
  → readdir(basePath)                         // 读取目录
  → 每个条目: join(skillDir, 'SKILL.md')       // 读取 SKILL.md
  → parseFrontmatter(content)                 // 解析 frontmatter
  → parseSkillFrontmatterFields(frontmatter)  // 解析字段
  → createSkillCommand({...})                 // 创建 Command 对象
    → getPromptForCommand(args, context)       // 返回:
      → 参数替换（substituteArguments）
      → Shell 命令执行（executeShellCommandsInPrompt）
      → [{type: 'text', text: finalContent}]
```

#### 3.2.3 动态技能（Dynamic Skills）

在文件操作过程中自动发现（`src/skills/loadSkillsDir.ts:861`）：

```
discoverSkillDirsForPaths(filePaths, cwd)
  → 对每个文件路径，向上遍历到 CWD
  → 检查 .claude/skills/ 目录是否存在
  → 跳过 gitignored 目录
  → 返回新发现的目录（按深度排序）

addSkillDirectories(dirs)
  → loadSkillsFromSkillsDir() 加载每个目录
  → 存入 dynamicSkills Map（深层路径优先）
  → 发出 skillsLoaded 信号
```

#### 3.2.4 条件性技能（Conditional Skills）

在 `src/skills/loadSkillsDir.ts:997` 中，具有 `paths` frontmatter 的技能：

- 启动时加载但不激活，存入 `conditionalSkills` Map
- 当文件操作匹配路径时激活：
  ```
  activateConditionalSkillsForPaths(filePaths, cwd)
    → 使用 ignore 库匹配 gitignore 风格模式
    → 匹配成功 → 移入 dynamicSkills
    → 发出 skillsLoaded 信号
  ```

#### 3.2.5 插件技能（Plugin Skills）

通过 `src/utils/plugins/loadPluginCommands.ts` 加载，在 `getCommands()` 中合并。

#### 3.2.6 MCP 技能（MCP Skills）

通过 MCP 服务器获取，存储在 `AppState.mcp.commands` 中，在 `SkillTool.getAllCommands()` 中合并。

### 3.3 Skill 的筛选与导出

`src/commands.ts` 为 SkillTool 提供专门的筛选函数：

```typescript
// 所有可被模型调用的 prompt 命令（SkillTool 可见）
getSkillToolCommands(cwd)
  → getCommands(cwd)
  → filter: type='prompt' && !disableModelInvocation
            && source !== 'builtin'
            && (loadedFrom 为 bundled/skills/commands_DEPRECATED
                || hasUserSpecifiedDescription || whenToUse)

// 用户可调用的斜杠技能（/skills 列表可见）
getSlashCommandToolSkills(cwd)
  → getCommands(cwd)
  → filter: type='prompt' && source !== 'builtin'
            && (hasUserSpecifiedDescription || whenToUse)
            && (loadedFrom 为 skills/plugin/bundled || disableModelInvocation)
```

### 3.4 Skill 调用流程

#### 3.4.1 用户通过斜杠调用

```
用户输入: /skill-name [args]
  → processSlashCommand(inputString, ...)
    → parseSlashCommand() 解析出 commandName 和 args
    → getMessagesForSlashCommand(commandName, args, ...)
      → type 'prompt':
        → context === 'fork'?
          → executeForkedSlashCommand()  // 子代理执行
          → getMessagesForPromptSlashCommand()  // 内联展开
            → command.getPromptForCommand(args, context)
              → 参数替换、Shell 命令执行
              → 返回 [{type: 'text', text: content}]
            → 构建消息: [metadata, content, attachments, permissions]
```

#### 3.4.2 模型通过 SkillTool 调用

模型调用 `SkillTool` 工具时，在 `src/tools/SkillTool/SkillTool.ts:333` 处理：

```
SkillTool.call({skill, args}, context, ...)
  → validateInput:
    → 检查格式、去除前导 /
    → getAllCommands(context) 获取所有命令（含 MCP 技能）
    → findCommand() 查找命令
    → 检查 type='prompt' 和 !disableModelInvocation
  → checkPermissions:
    → 检查拒绝/允许规则
    → 检查安全属性（auto-allow）
    → 默认: ask（请求用户确认）
  → call:
    → 远程规范技能（ant-only，experimental）:
      → executeRemoteSkill() 从 AKI/GCS 加载
    → 本地命令:
      → context === 'fork'?
        → executeForkedSkill()
          → prepareForkedCommandContext()
          → runAgent() 作为子代理执行
          → 返回 { success, commandName, status: 'forked', result }
        → processPromptSlashCommand()
          → command.getPromptForCommand(args, context)
          → 构建消息: [metadata, content, attachments, permissions]
          → 返回 { success, commandName, status: 'inline', allowedTools, model }
```

### 3.5 Skill 执行上下文

#### Inline（默认）

- Skill 内容展开到当前对话
- 模型直接处理 skill 的提示词和后续工具调用
- 适用于短小、无需隔离的技能

#### Fork（子代理）

- Skill 在独立子代理中执行（`runAgent()`）
- 拥有独立的上下文窗口和 token 预算
- 结果返回给主代理
- 适用于需要隔离或长时间运行的技能
- 可通过 `agent` 字段指定子代理类型（如 `general-purpose`、`Bash`）

---

## 四、完整执行流程时序

### 启动流程

```
main.tsx
  → initBuiltinPlugins()        // 注册内置插件
  → initBundledSkills()         // 注册打包技能
  → setup()                     // 初始化环境
  → getTools(permissionContext)  // 获取工具列表
  → getCommands(cwd)            // 加载所有命令和技能
    → loadAllCommands(cwd)      // 内部备忘录化
      → getSkills(cwd)          // 并行加载技能
      → getPluginCommands()     // 插件命令
      → getWorkflowCommands()   // 工作流命令
      → COMMANDS()              // 内置命令
  → launchRepl() / query()      // 启动 REPL 或直接查询
```

### 运行时交互

```
用户输入 /skill-name
  → processSlashCommand()
  → getMessagesForPromptSlashCommand()
  → skill.getPromptForCommand()
  → 注入消息到对话
  → query() 发送给 API
  → 模型返回 tool_use → runToolUse()
  → 工具执行 → 结果返回给模型
  → 继续对话...

模型决定调用 SkillTool
  → SkillTool.validateInput()
  → SkillTool.checkPermissions()
  → SkillTool.call()
    → executeForkedSkill() 或 processPromptSlashCommand()
    → 注入 skill 内容到对话
  → 继续对话...
```

---

## 五、关键文件索引

| 文件 | 作用 |
|------|------|
| `src/Tool.ts` | Tool 类型定义、`buildTool()` 工厂函数 |
| `src/types/command.ts` | Command 类型定义（CommandBase、PromptCommand、LocalCommand、LocalJSXCommand） |
| `src/tools.ts` | 工具注册（`getAllBaseTools`、`getTools`、`assembleToolPool`） |
| `src/commands.ts` | 命令注册（`COMMANDS`、`getCommands`、`getSkillToolCommands`、`getSlashCommandToolSkills`） |
| `src/services/tools/toolExecution.ts` | 工具执行（`runToolUse`） |
| `src/tools/SkillTool/SkillTool.ts` | SkillTool 工具实现（`validateInput`、`checkPermissions`、`call`、`executeForkedSkill`） |
| `src/tools/SkillTool/prompt.ts` | SkillTool 提示词生成（`getPrompt`、`getLimitedSkillToolCommands`） |
| `src/skills/loadSkillsDir.ts` | 文件系统技能加载（`loadSkillsFromSkillsDir`、`createSkillCommand`、`parseSkillFrontmatterFields`、`getSkillDirCommands`） |
| `src/skills/bundledSkills.ts` | 打包技能注册（`registerBundledSkill`、`getBundledSkills`） |
| `src/skills/bundled/index.ts` | 打包技能初始化（`initBundledSkills`） |
| `src/skills/mcpSkillBuilders.ts` | MCP 技能构建器注册表 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 斜杠命令处理（`processSlashCommand`、`getMessagesForSlashCommand`、`processPromptSlashCommand`） |
| `src/utils/forkedAgent.ts` | 子代理执行上下文准备（`prepareForkedCommandContext`） |
| `src/utils/toolSearch.ts` | 工具延迟加载系统 |
| `src/main.tsx` | 入口点，启动时初始化所有系统 |