# Claude Code 权限系统深度解析

> 本文深入分析 Claude Code 如何控制工具调用的权限。
> 涉及文件：`src/utils/permissions/*`、`src/hooks/useCanUseTool.ts`

---

## 一、权限系统的整体定位

### 1.1 为什么需要权限系统？

Claude Code 作为工业级 Agent，必须在**安全性**和**易用性**之间取得平衡：

```
┌────────────────────────────────────────────────────────┐
│  工具调用请求                                          │
│         ↓                                              │
│  ┌──────────────────────────────────────────┐          │
│  │  权限系统 (permissions.ts)                 │          │
│  │  决定 allow / ask / deny                   │          │
│  └──────────────────────────────────────────┘          │
│         ↓                                              │
│  ┌──────────┬──────────────┬──────────────┐            │
│  │  Allow   │     Ask      │     Deny     │            │
│  │ 放行执行 │ 询问用户    │   拒绝执行   │            │
│  └──────────┴──────────────┴──────────────┘            │
└────────────────────────────────────────────────────────┘
```

### 1.2 权限系统的核心职责

| 职责 | 说明 |
| --- | --- |
| **规则匹配** | 检查工具调用是否匹配 allow/deny/ask 规则 |
| **模式判断** | 根据当前权限模式（default/plan/bypassPermissions 等）决策 |
| **危险检测** | 识别危险命令模式（rm -rf、git push --force 等） |
| **AI 分类** | 在 auto 模式下用 AI 分类器判断 |
| **用户交互** | 通过 UI 或 SDK 询问用户 |
| **审计追踪** | 记录所有权限决策用于审计 |

### 1.3 权限系统的目录结构

```
src/utils/permissions/
├── permissions.ts              # 主调度中心（1486 行）
├── PermissionMode.ts           # 权限模式定义
├── PermissionResult.ts         # 决策结果类型
├── PermissionRule.ts           # 规则类型
├── PermissionUpdate.ts         # 规则更新
├── PermissionUpdateSchema.ts   # 更新 schema
├── permissionRuleParser.ts     # 规则解析器
├── permissionSetup.ts          # 模式设置
├── permissionsLoader.ts        # 规则加载
├── bashClassifier.ts           # Bash 命令分类器
├── yoloClassifier.ts           # YOLO 模式分类器
├── classifierDecision.ts        # 分类器决策
├── classifierShared.ts          # 分类器共享代码
├── dangerousPatterns.ts         # 危险模式识别
├── denialTracking.ts           # 拒绝追踪（fallback）
├── shellRuleMatching.ts        # Shell 规则匹配
├── pathValidation.ts           # 路径验证
├── filesystem.ts                # 文件系统权限
├── autoModeState.ts            # Auto 模式状态
├── bypassPermissionsKillswitch.ts  # 绕过权限的终止开关
├── shadowedRuleDetection.ts    # 规则冲突检测
├── permissionExplainer.ts      # 权限解释器
├── getNextPermissionMode.ts    # 下一个模式
└── PermissionPromptToolResultSchema.ts  # 权限提示工具结果 schema
```

---

## 二、权限模式（PermissionMode）

### 2.1 五种权限模式

```ts
// src/utils/permissions/PermissionMode.ts
export type PermissionMode =
    | 'default'             // 默认：所有操作需要询问
    | 'acceptEdits'         // 接受所有编辑
    | 'plan'                // 计划模式（只读 + 退出计划）
    | 'bypassPermissions'   // 绕过所有权限检查（危险！）
    | 'dontAsk'             // 不询问，自动拒绝
    | 'auto'                // AI 自动分类器模式
```

### 2.2 模式详细对比

| 模式 | 行为 | 适用场景 |
| --- | --- | --- |
| **default** | 所有操作需要询问 | 默认安全模式 |
| **acceptEdits** | 文件编辑自动通过，其他仍询问 | 频繁编辑场景 |
| **plan** | 只读工具可用，编辑需退出 plan | 规划阶段 |
| **bypassPermissions** | 跳过所有检查 | 受信任环境（CI/CD） |
| **dontAsk** | 自动拒绝所有 | 受限制环境 |
| **auto** | AI 分类器判断 | AI 辅助决策 |

### 2.3 getNextPermissionMode

```ts
// 模式切换循环
default → acceptEdits → plan → bypassPermissions → dontAsk → default
```

`getNextPermissionMode.ts` 实现 Shift+Tab 循环切换。

### 2.4 PERMISSION_MODES 常量

```ts
export const PERMISSION_MODES = [
    'default',
    'acceptEdits',
    'plan',
    'bypassPermissions',
    'dontAsk',
] as const
```

---

## 三、PermissionRule 规则系统

### 3.1 规则的来源

```ts
const PERMISSION_RULE_SOURCES = [
    ...SETTING_SOURCES,      // userSettings, projectSettings, localSettings, etc.
    'cliArg',                // CLI 参数传入的规则
    'command',               // /permissions 命令添加的
    'session',               // 会话中动态添加的
] as const
```

### 3.2 规则的数据结构

```ts
// src/utils/permissions/PermissionRule.ts
export type PermissionRule = {
    source: PermissionRuleSource
    ruleBehavior: 'allow' | 'deny' | 'ask'
    ruleValue: PermissionRuleValue
}

export type PermissionRuleValue = {
    toolName: string
    ruleContent?: string  // 可选：限制特定子命令或参数
}
```

### 3.3 规则示例

```bash
# 允许所有 Bash 命令
Bash

# 允许 npm 命令
Bash(npm:*)

# 拒绝删除文件
Bash(rm:*)

# 允许特定 MCP 工具
mcp__server1__tool1

# 拒绝特定 agent
Agent(Explore)

# 允许 Read 工具（不指定内容）
Read
```

### 3.4 规则的解析

```ts
// src/utils/permissions/permissionRuleParser.ts
export function permissionRuleValueFromString(ruleString: string): PermissionRuleValue {
    // 解析 "Bash(npm:*)" 为 { toolName: 'Bash', ruleContent: 'npm:*' }
    // 解析 "Read" 为 { toolName: 'Read' }
}

export function permissionRuleValueToString(ruleValue: PermissionRuleValue): string {
    // 反向序列化
}
```

### 3.5 规则的存储位置

```ts
// ToolPermissionContext（在 Tool.ts 中定义）
type ToolPermissionContext = DeepImmutable<{
    mode: PermissionMode
    additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
    alwaysAllowRules: ToolPermissionRulesBySource  // { [source]: string[] }
    alwaysDenyRules: ToolPermissionRulesBySource
    alwaysAskRules: ToolPermissionRulesBySource
    isBypassPermissionsModeAvailable: boolean
    isAutoModeAvailable?: boolean
    strippedDangerousRules?: ToolPermissionRulesBySource
    shouldAvoidPermissionPrompts?: boolean
    awaitAutomatedChecksBeforeDialog?: boolean
    prePlanMode?: PermissionMode
}>
```

### 3.6 getAllowRules / getDenyRules / getAskRules

```ts
export function getAllowRules(context: ToolPermissionContext): PermissionRule[] {
    return PERMISSION_RULE_SOURCES.flatMap(source =>
        (context.alwaysAllowRules[source] || []).map(ruleString => ({
            source,
            ruleBehavior: 'allow',
            ruleValue: permissionRuleValueFromString(ruleString),
        })),
    )
}
```

**对称设计**：allow、deny、ask 三种规则使用相同的聚合逻辑。

---

## 四、PermissionResult 决策结果

### 4.1 三种基本决策

```ts
// src/utils/permissions/PermissionResult.ts
export type PermissionDecision =
    | PermissionAllowDecision
    | PermissionDenyDecision
    | PermissionAskDecision

export type PermissionAllowDecision = {
    behavior: 'allow'
    updatedInput?: object  // 可能修改工具输入
    decisionReason?: PermissionDecisionReason
}

export type PermissionDenyDecision = {
    behavior: 'deny'
    message: string
    decisionReason?: PermissionDecisionReason
}

export type PermissionAskDecision = {
    behavior: 'ask'
    message: string
    decisionReason?: PermissionDecisionReason
    // 实际询问用户的回调
    isUpdatedInput?: boolean
}
```

### 4.2 决策原因类型

```ts
export type PermissionDecisionReason =
    | { type: 'rule'; rule: PermissionRule }
    | { type: 'mode'; mode: PermissionMode }
    | { type: 'safetyCheck'; reason: string }
    | { type: 'hook'; hookName: string; reason?: string }
    | { type: 'classifier'; classifier: string; reason: string }
    | { type: 'permissionPromptTool'; permissionPromptToolName: string }
    | { type: 'sandboxOverride' }
    | { type: 'workingDir'; reason: string }
    | { type: 'subcommandResults'; reasons: Map<string, PermissionDecision> }
    | { type: 'asyncAgent'; reason: string }
    | { type: 'other'; reason: string }
```

**多种决策原因**使得 UI 能向用户清晰解释"为什么需要询问"。

### 4.3 createPermissionRequestMessage

```ts
export function createPermissionRequestMessage(
    toolName: string,
    decisionReason?: PermissionDecisionReason,
): string {
    // 根据不同的决策原因类型生成不同的提示文本
    if (decisionReason) {
        switch (decisionReason.type) {
            case 'hook':
                return `Hook '${decisionReason.hookName}' requires approval for this ${toolName} command`
            case 'rule':
                return `Permission rule '${decisionReason.rule}' requires approval for this ${toolName} command`
            case 'classifier':
                return `Classifier '${decisionReason.classifier}' requires approval...`
            // ... 等等
        }
    }
    return `Claude requested permissions to use ${toolName}, but you haven't granted it yet.`
}
```

---

## 五、toolMatchesRule 规则匹配

### 5.1 工具名匹配

```ts
function toolMatchesRule(
    tool: Pick<Tool, 'name' | 'mcpInfo'>,
    rule: PermissionRule,
): boolean {
    // 规则必须没有内容才能整工具匹配
    if (rule.ruleValue.ruleContent !== undefined) {
        return false
    }
    
    // MCP 工具的特殊匹配
    if (tool.mcpInfo) {
        // mcp__server__tool 的规则匹配
        if (rule.ruleValue.toolName.startsWith('mcp__')) {
            // ...
        }
    }
    
    // 普通匹配
    return rule.ruleValue.toolName === tool.name
}
```

### 5.2 内容匹配（prefix）

```ts
function toolContentMatchesRule(
    tool: Tool,
    rule: PermissionRule,
    toolInput: unknown,
): boolean {
    if (rule.ruleValue.ruleContent === undefined) return false
    
    // Bash 工具特殊处理
    if (rule.ruleValue.toolName === 'Bash') {
        const command = (toolInput as { command: string }).command
        // 使用 shellRuleMatching 匹配
        return shellRuleMatches(command, rule.ruleValue.ruleContent)
    }
    
    // 通用 prefix 匹配
    const prefix = rule.ruleValue.ruleContent
    return JSON.stringify(toolInput).startsWith(prefix)
}
```

### 5.3 Shell 规则匹配

```ts
// src/utils/permissions/shellRuleMatching.ts
export function shellRuleMatches(
    command: string,
    ruleContent: string,
): boolean {
    // 支持通配符 *
    // 支持精确匹配
    // 支持前缀匹配
    if (ruleContent === '*') return true
    if (ruleContent.endsWith(':*')) {
        const prefix = ruleContent.slice(0, -2)
        return command.startsWith(prefix)
    }
    return command === ruleContent
}
```

---

## 六、hasPermissionsToUseTool 主入口

### 6.1 函数签名

```ts
// src/utils/permissions/permissions.ts
export async function hasPermissionsToUseTool(
    tool: Tool,
    input: ToolInput,
    toolUseContext: ToolUseContext,
    assistantMessage: AssistantMessage,
    toolUseID: string,
): Promise<PermissionResult>
```

**这是权限检查的核心入口**，被 `CanUseToolFn` 类型约束。

### 6.2 执行流程（关键管线）

```ts
async function hasPermissionsToUseToolInner(...): Promise<PermissionResult> {
    // step 1a: bypassPermissions mode → 直接 allow
    if (context.mode === 'bypassPermissions') return allow()
    
    // step 1b: 工具自身的 checkPermissions
    const toolCheck = await tool.checkPermissions(input, context)
    if (toolCheck.behavior !== 'passthrough') return toolCheck
    
    // step 1c: deny 规则匹配
    const denyResult = checkDenyRules(...)
    if (denyResult) return denyResult
    
    // step 1d: 安全检查（危险命令模式）
    const safetyResult = checkSafety(...)
    if (safetyResult) return safetyResult
    
    // step 1e: 工作目录检查
    const workingDirResult = checkWorkingDir(...)
    if (workingDirResult) return workingDirResult
    
    // step 1f: 子命令结果检查
    const subCommandResult = checkSubCommands(...)
    if (subCommandResult) return subCommandResult
    
    // step 1g: PreToolUse Hook
    const hookResult = await executePreToolUseHook(...)
    if (hookResult.behavior === 'deny') return hookResult
    
    // step 2a: allow 规则匹配
    const allowResult = checkAllowRules(...)
    if (allowResult) return allowResult
    
    // step 2b: acceptEdits 模式特殊处理
    if (context.mode === 'acceptEdits' && isEditTool(tool)) return allow()
    
    // step 3: passthrough → ask 转换
    return ask(reason)
}
```

### 6.3 完整的执行管线

```
1. bypassPermissions 模式？
   → 是：allow
   ↓
2. tool.checkPermissions
   → 不是 passthrough：直接返回结果
   ↓
3. deny 规则匹配
   → 匹配：deny
   ↓
4. 安全检查（dangerousPatterns）
   → 匹配：deny（safetyCheck）
   ↓
5. 工作目录检查
   → 不允许：ask（workingDir）
   ↓
6. 子命令结果（Bash 等）
   → 有 ask/deny：合并处理
   ↓
7. PreToolUse Hook
   → deny：deny
   ↓
8. allow 规则匹配
   → 匹配：allow
   ↓
9. acceptEdits 模式（仅编辑工具）
   → 是：allow
   ↓
10. PermissionRequest Hook
    → allow/deny：直接返回
    ↓
11. auto 模式 + YOLO 分类器
    → 分类器决策
    ↓
12. 默认：ask
```

### 6.4 Deny 规则检查（先于 Allow）

```ts
// 为什么 deny 在 allow 之前？
// 答案：安全优先。即使有 allow 规则，如果 deny 规则匹配，仍然拒绝。

const denyRules = getDenyRules(context)
for (const rule of denyRules) {
    if (toolMatchesRule(tool, rule)) {
        return {
            behavior: 'deny',
            message: `Denied by rule '${rule.ruleValue}' from ${rule.source}`,
            decisionReason: {type: 'rule', rule},
        }
    }
}
```

### 6.5 Allow 规则检查

```ts
const allowRules = getAllowRules(context)
for (const rule of allowRules) {
    if (toolMatchesRule(tool, rule)) {
        // 整工具匹配
        return {
            behavior: 'allow',
            updatedInput: input,
            decisionReason: {type: 'rule', rule},
        }
    }
    // 内容匹配（如 Bash(npm:*))
    if (toolContentMatchesRule(tool, rule, input)) {
        return {
            behavior: 'allow',
            updatedInput: input,
            decisionReason: {type: 'rule', rule},
        }
    }
}
```

---

## 七、Bash 命令特殊处理

### 7.1 dangerousPatterns 危险模式

```ts
// src/utils/permissions/dangerousPatterns.ts
const DANGEROUS_PATTERNS = [
    /rm\s+-rf\s+\//,              // 删除根目录
    /:\(\)\s*{\s*:\|:&\s*};:/,    // fork 炸弹
    /mkfs/,                       // 格式化磁盘
    /dd\s+if=.*of=\/dev\/sd/,     // 磁盘擦写
    /curl.*\|.*sh/,               // 远程脚本执行
    /git\s+push\s+--force/,       // 强制推送
    // ... 更多
]
```

### 7.2 bashPermissions 子命令解析

```ts
// src/tools/BashTool/bashPermissions.ts
export async function checkBashPermissions(
    command: string,
    context: ToolUseContext,
): Promise<PermissionResult> {
    // 1. 解析子命令
    const subCommands = parseShellCommand(command)
    
    // 2. 检查每个子命令
    const results = new Map<string, PermissionDecision>()
    for (const subCmd of subCommands) {
        const result = checkSubCommand(subCmd, context)
        results.set(subCmd, result)
    }
    
    // 3. 综合决策
    if (results.has('deny')) return aggregateDeny(results)
    if (results.has('ask')) return aggregateAsk(results)
    return aggregateAllow(results)
}
```

### 7.3 destructiveCommandWarning 危险命令警告

```ts
// src/tools/BashTool/destructiveCommandWarning.ts
// 检测 rm、mv、chmod 等破坏性命令
// 在执行前显示警告
```

---

## 八、Auto 模式 + YOLO 分类器

### 8.1 Auto 模式概述

```ts
if (context.mode === 'auto' && classifierDecisionModule) {
    // 1. 检查分类器是否可用
    // 2. 检查 denial tracking
    // 3. 调用 YOLO 分类器
    // 4. 根据分类器结果决策
}
```

### 8.2 YOLO Classifier

```ts
// src/utils/permissions/yoloClassifier.ts
export async function classifyYoloAction(
    tool: Tool,
    input: unknown,
    context: ToolUseContext,
): Promise<{
    decision: 'allow' | 'deny' | 'ask'
    reason: string
    confidence: number
}>
```

**YOLO 分类器**用 AI 评估每个工具调用是否安全：
- 读取文件的操作通常 allow
- 危险命令通常 deny
- 不确定的 ask

### 8.3 Bash Classifier

```ts
// src/utils/permissions/bashClassifier.ts
// 专门针对 Bash 命令的 AI 分类器
// 比通用分类器更精确
```

### 8.4 denialTracking 拒绝追踪

```ts
// src/utils/permissions/denialTracking.ts
const DENIAL_LIMITS = {
    perSession: 5,        // 每会话最多 5 次 deny
    perTurn: 2,           // 每回合最多 2 次 deny
    fallbackThreshold: 3, // 超过阈值 fallback 到询问
}

export function recordDenial(state: DenialTrackingState): DenialTrackingState {
    // 记录 deny 次数
    // 超过阈值后 fallback 到询问
}
```

**设计目的**：在 auto 模式下，避免模型被"困在死循环"中（每次都被自动拒绝）。

### 8.5 classifierDecision

```ts
// src/utils/permissions/classifierDecision.ts
// 综合分类器决策
export async function getClassifierDecision(
    tool: Tool,
    input: unknown,
    context: ToolUseContext,
): Promise<PermissionDecision>
```

---

## 九、Hook 系统集成

### 9.1 PreToolUse Hook

```ts
// 在 hasPermissionsToUseToolInner 中
const hookResult = await executePreToolUseHook(toolName, input, context)
if (hookResult.behavior === 'deny') {
    return {
        behavior: 'deny',
        message: hookResult.message,
        decisionReason: {type: 'hook', hookName: 'PreToolUse', reason: hookResult.message},
    }
}
```

### 9.2 PermissionRequest Hook

```ts
// 自动询问之前的最后一道关卡
const permissionRequestResult = await executePermissionRequestHooks(tool, input, context)
if (permissionRequestResult.behavior !== 'passthrough') {
    return permissionRequestResult
}
```

### 9.3 PostToolUse Hook（执行后）

```ts
// 在工具执行后触发
await executePostToolUseHook(toolName, input, result, context)
```

---

## 十、PermissionPromptTool（自定义权限工具）

### 10.1 概念

允许用户**自定义一个 MCP 工具**作为权限提示工具，处理权限请求。

### 10.2 使用流程

```ts
// 1. 用户传入 --permission-prompt-tool 参数
const effectivePermissionPromptToolName = options.sdkUrl
    ? 'stdio'
    : options.permissionPromptToolName

// 2. 创建 canUseTool 函数
const canUseTool = getCanUseToolFn(
    effectivePermissionPromptToolName,
    structuredIO,
    () => getAppState().mcp.tools,
    onPermissionPrompt,
)
```

### 10.3 createCanUseToolWithPermissionPrompt

```ts
// src/cli/print.ts
export function createCanUseToolWithPermissionPrompt(
    permissionPromptTool: PermissionPromptTool,
): CanUseToolFn {
    return async (tool, input, context, assistantMessage, toolUseId, forceDecision) => {
        // 1. 先走通用权限检查
        const mainResult = forceDecision ?? await hasPermissionsToUseTool(
            tool, input, context, assistantMessage, toolUseId,
        )
        
        if (mainResult.behavior === 'allow' || mainResult.behavior === 'deny') {
            return mainResult
        }
        
        // 2. 询问权限提示工具
        const abortPromise = new Promise<'aborted'>((resolve) => {
            context.abortController.signal.addEventListener('abort', () => resolve('aborted'), {once: true})
        })
        
        const toolCallPromise = permissionPromptTool.call(
            {tool_name: tool.name, input, tool_use_id: toolUseId},
            context, canUseTool, assistantMessage,
        )
        
        const raceResult = await Promise.race([toolCallPromise, abortPromise])
        if (raceResult === 'aborted') {
            return {behavior: 'deny', message: 'Permission prompt was aborted.', ...}
        }
        
        // 3. 转换结果为 PermissionResult
        return permissionPromptToolResultToPermissionDecision(...)
    }
}
```

### 10.4 PermissionPromptToolResultSchema

```ts
// src/utils/permissions/PermissionPromptToolResultSchema.ts
export const permissionToolOutputSchema = z.object({
    behavior: z.enum(['allow', 'deny']),
    updatedInput: z.unknown().optional(),
    message: z.string().optional(),
})
```

---

## 十一、bypassPermissions 模式

### 11.1 模式行为

```ts
// 直接 allow 所有操作
if (context.mode === 'bypassPermissions') {
    return {
        behavior: 'allow',
        updatedInput: input,
        decisionReason: {type: 'mode', mode: 'bypassPermissions'},
    }
}
```

### 11.2 终止开关

```ts
// src/utils/permissions/bypassPermissionsKillswitch.ts
export function isBypassPermissionsModeDisabled(): boolean {
    // 检查是否被 settings 禁用
    // 例如组织策略禁止 bypassPermissions
}
```

### 11.3 启动条件

```ts
// 必须使用 --dangerously-skip-permissions 启动
// 否则 sessionBypassPermissionsMode 为 false
```

### 11.4 sessionIsBypassPermissionsModeAvailable

```ts
// ToolPermissionContext 中的字段
isBypassPermissionsModeAvailable: boolean
```

只有在 CLI 启动时设置 `--dangerously-skip-permissions`，这个字段才为 true。

---

## 十二、Auto 模式深入

### 12.1 mode 自动分类器

```ts
// src/utils/permissions/autoModeState.ts
// 跟踪 auto 模式的运行状态
export function isAutoModeGateEnabled(): boolean {
    return feature('TRANSCRIPT_CLASSIFIER') && 
           getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_mode_gate', true)
}
```

### 12.2 分类器不可用时

```ts
// src/utils/messages.ts
export function buildClassifierUnavailableMessage(tool: Tool): string {
    return `Auto mode: classifier is unavailable, falling back to ${tool.name} for safety`
}
```

### 12.3 Auto 拒绝消息

```ts
export const AUTO_REJECT_MESSAGE = 'Auto mode rejected this tool call based on its analysis'
```

### 12.4 Don't Ask 拒绝消息

```ts
export const DONT_ASK_REJECT_MESSAGE = "Tool call denied because 'dontAsk' mode doesn't allow permission requests"
```

---

## 十三、AdditionalWorkingDirectories 目录权限

### 13.1 概念

用户可以指定**额外的工作目录**，允许访问这些目录中的文件。

```ts
type ToolPermissionContext = {
    // ...
    additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
    // ...
}

type AdditionalWorkingDirectory = {
    // 路径元数据
    path: string
    // 权限：readonly / readwrite
    access: 'readonly' | 'readwrite'
}
```

### 13.2 路径验证

```ts
// src/utils/permissions/pathValidation.ts
export function isPathInWorkingDirectory(
    filePath: string,
    context: ToolUseContext,
): boolean {
    // 1. 主 cwd
    if (isInDirectory(filePath, context.cwd)) return true
    
    // 2. 额外工作目录
    for (const [dirPath] of context.toolPermissionContext.additionalWorkingDirectories) {
        if (isInDirectory(filePath, dirPath)) return true
    }
    
    return false
}
```

### 13.3 workingDir 决策原因

```ts
// 如果文件不在工作目录中
if (!isPathInWorkingDirectory(filePath, context)) {
    return {
        behavior: 'ask',
        message: `${filePath} is outside the working directory`,
        decisionReason: {type: 'workingDir', reason: `...`},
    }
}
```

---

## 十四、filesystem.ts 文件系统权限

### 14.1 路径权限检查

```ts
// src/utils/permissions/filesystem.ts
export function checkFilePermissions(
    filePath: string,
    operation: 'read' | 'write',
    context: ToolUseContext,
): PermissionResult
```

### 14.2 符号链接处理

```ts
// 处理符号链接，避免绕过权限检查
const realPath = await fs.realpath(filePath)
```

### 14.3 受保护目录

```ts
const PROTECTED_DIRECTORIES = [
    '/etc',
    '/usr',
    '/bin',
    '/sbin',
    // Windows: C:\Windows\System32
]
```

---

## 十五、permissionSetup 模式设置

### 15.1 模式切换函数

```ts
// src/utils/permissions/permissionSetup.ts
export function transitionPermissionMode(
    currentMode: PermissionMode,
    newMode: PermissionMode,
    context: ToolPermissionContext,
): ToolPermissionContext
```

### 15.2 处理特殊转换

```ts
// plan 模式切换到 default 时
// 保留之前的额外工作目录
// 保留规则

// bypassPermissions 切换到 default 时
// 保留规则，但实际不再 bypass
```

### 15.3 Mode 标题

```ts
// PermissionMode.ts
export function permissionModeTitle(mode: PermissionMode): string {
    switch (mode) {
        case 'default': return 'Default'
        case 'acceptEdits': return 'Accept Edits'
        case 'plan': return 'Plan Mode'
        case 'bypassPermissions': return 'Bypass Permissions'
        case 'dontAsk': return 'Don't Ask'
        case 'auto': return 'Auto Mode'
    }
}
```

---

## 十六、PermissionUpdate 规则更新

### 16.1 applyPermissionUpdate

```ts
// src/utils/permissions/PermissionUpdate.ts
export function applyPermissionUpdate(
    update: PermissionUpdate,
    context: ToolPermissionContext,
): ToolPermissionContext
```

### 16.2 三种更新目的地

```ts
type PermissionUpdateDestination =
    | 'userSettings'      // 用户设置
    | 'projectSettings'   // 项目设置
    | 'localSettings'     // 本地设置
    | 'session'           // 会话设置
```

### 16.3 更新规则

```ts
// 用户批准后
const update: PermissionUpdate = {
    destination: 'userSettings',
    behavior: 'allow',
    rule: 'Bash(npm:*)',
}

const newContext = applyPermissionUpdate(update, context)
```

### 16.4 persistPermissionUpdates

```ts
// 持久化到磁盘
export function persistPermissionUpdates(
    updates: PermissionUpdate[],
): Promise<void>
```

---

## 十七、permissionRuleParser 规则解析

### 17.1 规则字符串格式

```
ToolName
ToolName(content)
```

### 17.2 解析示例

```ts
permissionRuleValueFromString('Bash')
// → { toolName: 'Bash', ruleContent: undefined }

permissionRuleValueFromString('Bash(npm:*)')
// → { toolName: 'Bash', ruleContent: 'npm:*' }

permissionRuleValueFromString('Agent(Explore)')
// → { toolName: 'Agent', ruleContent: 'Explore' }

permissionRuleValueFromString('mcp__server1__tool1')
// → { toolName: 'mcp__server1__tool1', ruleContent: undefined }
```

### 17.3 序列化

```ts
permissionRuleValueToString({ toolName: 'Bash', ruleContent: 'npm:*' })
// → 'Bash(npm:*)'
```

---

## 十八、permissionsLoader 规则加载

### 18.1 加载顺序

```ts
// src/utils/permissions/permissionsLoader.ts
export function loadPermissionRules(): {
    userSettings: string[]
    projectSettings: string[]
    localSettings: string[]
    // ...
}
```

**加载优先级**（从高到低）：
1. localSettings（本地，最优先）
2. projectSettings（项目）
3. userSettings（用户）
4. policySettings（组织策略，最严格）

### 18.2 shouldAllowManagedPermissionRulesOnly

```ts
// 只能使用 policySettings 提供的规则（其他源被禁用）
export function shouldAllowManagedPermissionRulesOnly(): boolean {
    return getFeatureValue_CACHED_MAY_BE_STALE('tengu_managed_permissions', false)
}
```

**用途**：企业部署中，组织策略可以锁定规则来源。

### 18.3 deletePermissionRuleFromSettings

```ts
// 删除规则
export function deletePermissionRuleFromSettings(
    rule: string,
    destination: PermissionUpdateDestination,
): Promise<void>
```

---

## 十九、shadowedRuleDetection 规则冲突检测

### 19.1 概念

检测规则的**shadowing**（遮蔽）：一个规则被另一个规则遮蔽而永远不会生效。

```ts
// src/utils/permissions/shadowedRuleDetection.ts
export function findShadowedRules(
    rules: PermissionRule[],
): PermissionRule[]  // 返回被遮蔽的规则
```

### 19.2 典型场景

```bash
# 规则 1（被遮蔽）
Allow Bash(rm:*)

# 规则 2（覆盖规则 1）
Deny Bash(rm:rf:*)
```

### 19.3 使用

```ts
// 在用户添加规则时提示
const shadowed = findShadowedRules(allRules)
if (shadowed.length > 0) {
    showWarning(`Rules ${shadowed} are shadowed and won't take effect`)
}
```

---

## 二十、permissionExplainer 解释器

### 20.1 概念

为用户**解释为什么需要询问**，帮助用户做出决策。

### 20.2 使用场景

```ts
// 在 UI 中显示
"Claude wants to run: rm -rf node_modules/
This command was flagged as potentially dangerous.
[Allow] [Deny] [Explain more]"
```

### 20.3 实现

```ts
export function explainPermissionRequest(
    tool: Tool,
    input: unknown,
    context: ToolUseContext,
): string {
    // 根据不同的 reason 生成解释
}
```

---

## 二十一、ShouldAvoidPermissionPrompts

### 21.1 概念

某些场景下**不显示权限提示**（自动拒绝或自动通过）。

```ts
// ToolPermissionContext 中的字段
shouldAvoidPermissionPrompts?: boolean
```

### 21.2 使用场景

| 场景 | shouldAvoidPermissionPrompts |
| --- | --- |
| 后台 agent（async） | true（自动拒绝） |
| Bubble mode | false（冒泡到父终端） |
| 压缩压缩期间 | true（禁止工具） |
| 子 agent 内部 | true（避免无限循环） |

### 21.3 createCompactCanUseTool

```ts
export function createCompactCanUseTool(): CanUseToolFn {
    return async () => ({
        behavior: 'deny',
        message: 'Tool use is not allowed during compaction',
        decisionReason: {
            type: 'other',
            reason: 'compaction agent should only produce text summary',
        },
    })
}
```

---

## 二十二、awaitAutomatedChecksBeforeDialog

### 22.1 概念

在显示权限对话框前，**等待自动化检查**（classifier、hooks）完成。

```ts
// ToolPermissionContext 中的字段
awaitAutomatedChecksBeforeDialog?: boolean
```

### 22.2 使用场景

- 后台 agent 但可以显示 prompt
- coordinator workers
- 需要 hook + classifier 都完成才显示 UI

---

## 二十三、错误处理与边界情况

### 23.1 规则解析失败

```ts
// 如果规则字符串无法解析，跳过而不是失败
try {
    const rule = permissionRuleValueFromString(ruleString)
    rules.push(rule)
} catch (error) {
    logError(`Invalid rule: ${ruleString}`)
    // 跳过此规则，继续处理其他规则
}
```

### 23.2 循环依赖防护

```ts
// 防止 hasPermissionsToUseTool 被递归调用
if (context.abortController.signal.aborted) {
    return {behavior: 'deny', message: 'Aborted', ...}
}
```

### 23.3 工具内部权限错误

```ts
// 如果 tool.checkPermissions 抛错
try {
    return await tool.checkPermissions(input, context)
} catch (error) {
    logError(error)
    // 降级为 ask
    return ask('tool check failed')
}
```

---

## 二十四、可观测性

### 24.1 日志事件

```ts
logEvent('tengu_permission_decision', {
    tool: toolName,
    decision: decision.behavior,
    reason: decision.decisionReason?.type,
    source: rule.source,
    mode: context.mode,
})
```

### 24.2 调试日志

```ts
logForDebugging(`Permission decision: ${toolName} → ${decision.behavior}`, {
    reason: decision.decisionReason,
    matchedRule: matchingRule,
})
```

---

## 二十五、关键设计模式

### 25.1 规则优先匹配

```
规则按 PERMISSION_RULE_SOURCES 顺序聚合
deny 规则先于 allow 规则检查（安全优先）
按 source 优先级覆盖（session > command > cliArg > settings）
```

### 25.2 装饰器模式

```ts
const wrappedCanUseTool: CanUseToolFn = async (...args) => {
    const result = await canUseTool(...args)
    // 拦截 + 副作用
    if (result.behavior !== 'allow') {
        this.permissionDenials.push(...)
    }
    return result
}
```

### 25.3 责任链模式

```
bypassPermissions → tool.checkPermissions → deny rules → safety check → 
working dir → subcommands → hook → allow rules → mode → auto classifier → ask
```

每一层都可能终止检查链。

### 25.4 配置 vs 代码

```bash
# 规则（配置）
settings.json:
{
    "permissions": {
        "allow": ["Bash(npm:*)"],
        "deny": ["Bash(rm:*)"]
    }
}

# 复杂逻辑（代码）
yoloClassifier.ts - AI 分类
shellRuleMatching.ts - Shell 规则匹配
dangerousPatterns.ts - 危险模式识别
```

---

## 二十六、关键文件速查

| 文件 | 作用 |
| --- | --- |
| `src/utils/permissions/permissions.ts` | 主调度中心（1486 行） |
| `src/utils/permissions/PermissionMode.ts` | 权限模式定义 |
| `src/utils/permissions/PermissionResult.ts` | 决策结果类型 |
| `src/utils/permissions/PermissionRule.ts` | 规则类型 |
| `src/utils/permissions/bashClassifier.ts` | Bash AI 分类器 |
| `src/utils/permissions/yoloClassifier.ts` | YOLO AI 分类器 |
| `src/utils/permissions/dangerousPatterns.ts` | 危险模式识别 |
| `src/utils/permissions/denialTracking.ts` | 拒绝追踪 |
| `src/utils/permissions/permissionsLoader.ts` | 规则加载 |
| `src/utils/permissions/permissionRuleParser.ts` | 规则解析 |
| `src/utils/permissions/permissionSetup.ts` | 模式设置 |
| `src/utils/permissions/PermissionUpdate.ts` | 规则更新 |
| `src/utils/permissions/shadowedRuleDetection.ts` | 规则冲突检测 |

---

## 二十七、设计要点总结

### 27.1 Claude Code 权限系统的关键洞察

1. **多层防御**：规则匹配、模式判断、AI 分类、Hook 系统
2. **deny 优先**：安全永远第一
3. **来源优先级**：session > command > cliArg > settings
4. **AI 辅助**：auto 模式用 AI 决策不确定操作
5. **拒绝追踪**：避免 auto 模式陷入死循环
6. **危险模式**：内置常见危险命令识别
7. **用户友好**：清晰的解释和错误信息

### 27.2 关键设计决策

| 决策 | 理由 |
| --- | --- |
| **规则聚合** | 多个来源的规则合并处理 |
| **deny 先于 allow** | 安全优先 |
| **AI 分类器** | 处理边界情况 |
| **拒绝追踪** | 防止 auto 模式死循环 |
| **危险模式识别** | 内置常见威胁 |
| **Hook 集成** | 用户可自定义权限逻辑 |
| **PermissionPromptTool** | SDK 用户可自定义提示 UI |

### 27.3 权限系统的设计哲学

```ts
// 1. 安全优先（Deny before Allow）
if (matchesDenyRule(tool, input)) return deny()

// 2. 多种检查层（Defense in Depth）
if (tool.checkPermissions) ... // 工具自身
if (safetyCheck(tool, input)) ... // 危险模式
if (workingDirCheck(tool, input)) ... // 工作目录
if (subCommandCheck(tool, input)) ... // 子命令
if (preToolUseHook(tool, input)) ... // Hook
if (matchesAllowRule(tool, input)) return allow()
if (mode === 'acceptEdits') ... // 模式
if (autoClassifier(tool, input)) ... // AI
return ask()

// 3. 用户可定制（Hooks, PermissionPromptTool）
// 4. 可观测性（日志、统计）
// 5. 失败安全（Fail-Safe：默认 ask）
```

**理解 Claude Code 的权限系统，就能理解如何设计一个既安全又易用的 AI Agent！** 🎯