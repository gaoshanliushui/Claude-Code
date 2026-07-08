# Tool.ts — 工具系统设计原理详解

本文档深入分析 Claude Code 核心 `Tool.ts` 的设计原理、架构思想和关键实现。

---

## 一、工具系统整体定位

### 1.1 Tool.ts 在架构中的角色

`Tool.ts` 是 Claude Code 工具系统的**基础架构定义层**，它规定了所有工具的统一接口和通用实现模式。它不是一个具体工具，而是一个**工具的元模型（Meta Model）**，使得：

- 每个工具都是一个完整的自包含模块
- 所有工具遵循统一的接口规范
- 工具的渲染、权限、验证逻辑可以差异化实现
- 工具可以被统一调度、权限检查和错误处理

### 1.2 工具系统的核心理念

工具系统的设计遵循三个关键原则：

| 原则 | 说明 |
| --- | --- |
| **单一职责原则** | 每个工具只做一件事，但要把这件事做到尽善尽美 |
| **声明式配置优先** | 大部分工具通过配置（Schema、Prompt）而非代码来定义行为 |
| **安全分层检查** | 工具调用需要经过多层检查：输入验证、权限检查、预检查等 |

---

## 二、核心类型系统详解

### 2.1 Tool 接口：工具的核心契约

```typescript
export type Tool<
    Input extends AnyObject = AnyObject,
    Output = unknown,
    Progress extends ToolProgressData = ToolProgressData
> = {
    readonly name: string;
    aliases?: string[];
    searchHint?: string;
    
    // 核心执行与描述
    call(args: Input, context: ToolUseContext, canUseTool: CanUseToolFn, parentMessage: AssistantMessage, onProgress?: ToolCallProgress<Progress>): Promise<ToolResult<Output>>;
    description(input: Input, options: { isNonInteractiveSession: boolean; toolPermissionContext: ToolPermissionContext; tools: Tools }): Promise<string>;
    
    // Schema 定义
    readonly inputSchema: Input;
    readonly inputJSONSchema?: ToolInputJSONSchema;
    outputSchema?: Zod.ZodType<unknown>;
    
    // 工具性质
    inputsEquivalent?(a: Input, b: Input): boolean;
    isConcurrencySafe(input: Input): boolean;
    isEnabled(): boolean;
    isReadOnly(input: Input): boolean;
    isDestructive?(input: Input): boolean;
    isOpenWorld?(input: Input): boolean;
    requiresUserInteraction?(): boolean;
    isMcp?: boolean;
    isLsp?: boolean;
    
    // 延迟加载与搜索
    readonly shouldDefer?: boolean;
    readonly alwaysLoad?: boolean;
    
    // 权限与验证
    validateInput?(input: Input, context: ToolUseContext): Promise<ValidationResult>;
    checkPermissions(input: Input, context: ToolUseContext): Promise<PermissionResult>;
    getPath?(input: Input): string;
    preparePermissionMatcher?(input: Input): Promise<(pattern: string) => boolean>;
    
    // 系统提示词与用户展示
    prompt(options: { getToolPermissionContext: () => Promise<ToolPermissionContext>; tools: Tools; agents: AgentDefinition[]; allowedAgentTypes?: string[] }): Promise<string>;
    userFacingName(input: Partial<Input> | undefined): string;
    userFacingNameBackgroundColor?(input: Partial<Input> | undefined): keyof Theme | undefined;
    isTransparentWrapper?(): boolean;
    
    // 结果大小限制
    maxResultSizeChars: number;
    
    // 渲染相关方法（UI层）
    getToolUseSummary?(input: Partial<Input> | undefined): string | null;
    getActivityDescription?(input: Partial<Input> | undefined): string | null;
    toAutoClassifierInput(input: Input): unknown;
    mapToolResultToToolResultBlockParam(content: Output, toolUseId: string): ToolResultBlockParam;
    renderToolUseMessage(input: Partial<Input>, options: { theme: ThemeName; verbose: boolean; commands?: Command[] }): React.ReactNode;
    renderToolUseProgressMessage?(progressMessages: ProgressMessage<Progress>[], options: { tools: Tools; verbose: boolean; terminalSize?: { columns: number; rows: number }; inProgressToolCallCount?: number; isTranscriptMode?: boolean }): React.ReactNode;
    renderToolUseQueuedMessage?(): React.ReactNode;
    renderToolUseRejectedMessage?(input: Input, options: { columns: number; messages: Message[]; style?: 'condensed'; theme: ThemeName; tools: Tools; verbose: boolean; progressMessages: ProgressMessage<Progress>[]; isTranscriptMode?: boolean }): React.ReactNode;
    renderToolUseErrorMessage?(result: ToolResultBlockParam['content'], options: { progressMessages: ProgressMessage<Progress>[]; tools: Tools; verbose: boolean; isTranscriptMode?: boolean }): React.ReactNode;
    renderToolUseTag?(input: Partial<Input>): React.ReactNode;
    renderToolResultMessage?(output: Output, progressMessages: ProgressMessage<Progress>[], options: { style?: 'condensed'; theme: ThemeName; tools: Tools; verbose: boolean; isTranscriptMode?: boolean; isBriefOnly?: boolean; input?: unknown }): React.ReactNode;
    renderGroupedToolUse?(toolUses: Array<{ param: ToolUseBlockParam; isResolved: boolean; isError: boolean; isInProgress: boolean; progressMessages: ProgressMessage<Progress>[]; result?: { param: ToolResultBlockParam; output: unknown } }>, options: { shouldAnimate: boolean; tools: Tools }): React.ReactNode | null;
    
    // 工具分类与搜索优化
    isSearchOrReadCommand?(input: Input): { isSearch: boolean; isRead: boolean; isList?: boolean };
    
    // 输入回填（用于透明观察）
    backfillObservableInput?(input: Record<string, unknown>): void;
    
    // 结果截断检测
    isResultTruncated?(output: Output): boolean;
    
    // 搜索文本提取
    extractSearchText?(output: Output): string;
};
```

这是一个**功能非常全面**的接口，它涵盖了工具生命周期的方方面面：

### 2.2 ToolUseContext：工具执行时的上下文环境

`ToolUseContext` 是工具执行时的环境快照，它提供了工具所需要的所有外部依赖：

```typescript
export type ToolUseContext = {
    // 环境选项
    options: {
        commands: Command[];
        debug: boolean;
        mainLoopModel: string;
        tools: Tools;
        verbose: boolean;
        thinkingConfig: ThinkingConfig;
        mcpClients: MCPServerConnection[];
        mcpResources: Record<string, ServerResource[]>;
        isNonInteractiveSession: boolean;
        agentDefinitions: AgentDefinitionsResult;
        maxBudgetUsd?: number;
        customSystemPrompt?: string;
        appendSystemPrompt?: string;
        querySource?: QuerySource;
        refreshTools?: () => Tools;
    };
    
    // 状态访问与修改
    abortController: AbortController;
    readFileState: FileStateCache;
    getAppState(): AppState;
    setAppState(update: (prev: AppState) => AppState): void;
    setAppStateForTasks?(update: (prev: AppState) => AppState): void;
    
    // UI 回调
    handleElicitation?(serverName: string, params: ElicitRequestURLParams, signal: AbortSignal): Promise<ElicitResult>;
    setToolJSX?: SetToolJSXFn;
    addNotification?: (notif: Notification) => void;
    appendSystemMessage?: (msg: Exclude<SystemMessage, SystemLocalCommandMessage>) => void;
    sendOSNotification?: (opts: { message: string; notificationType: string }) => void;
    setInProgressToolUseIDs: (update: (prev: Set<string>) => Set<string>) => void;
    setHasInterruptibleToolInProgress?: (value: boolean) => void;
    setResponseLength: (update: (prev: number) => number) => void;
    pushApiMetricsEntry?(ttftMs: number): void;
    setStreamMode?: (mode: SpinnerMode) => void;
    onCompactProgress?: (event: CompactProgressEvent) => void;
    setSDKStatus?: (status: SDKStatus) => void;
    openMessageSelector?: () => void;
    
    // 状态更新
    updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void;
    updateAttributionState: (updater: (prev: AttributionState) => AttributionState) => void;
    setConversationId?: (id: UUID) => void;
    
    // 工具识别信息
    toolUseId?: string;
    
    // 附加功能
    criticalSystemReminder_EXPERIMENTAL?: string;
    preserveToolUseResults?: boolean;
    localDenialTracking?: DenialTrackingState;
    contentReplacementState?: ContentReplacementState;
    renderedSystemPrompt?: SystemPrompt;
    
    // 回调工厂
    requestPrompt?: (sourceName: string, toolInputSummary?: string | null) => (request: PromptRequest) => Promise<PromptResponse>;
    
    // 消息与限制
    messages: Message[];
    fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number };
    globLimits?: { maxResults?: number };
    
    // 工具决策缓存
    toolDecisions?: Map<string, { source: string; decision: 'accept' | 'reject'; timestamp: number }>;
    
    // 工具链追踪
    queryTracking?: QueryChainTracking;
    
    // 嵌套内存与技能
    nestedMemoryAttachmentTriggers?: Set<string>;
    loadedNestedMemoryPaths?: Set<string>;
    dynamicSkillDirTriggers?: Set<string>;
    discoveredSkillNames?: Set<string>;
    userModified?: boolean;
};
```

### 2.3 工具权限上下文

```typescript
export type ToolPermissionContext = DeepImmutable<{
    mode: PermissionMode;
    additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>;
    alwaysAllowRules: ToolPermissionRulesBySource;
    alwaysDenyRules: ToolPermissionRulesBySource;
    alwaysAskRules: ToolPermissionRulesBySource;
    isBypassPermissionsModeAvailable: boolean;
    isAutoModeAvailable?: boolean;
    strippedDangerousRules?: ToolPermissionRulesBySource;
    shouldAvoidPermissionPrompts?: boolean;
    awaitAutomatedChecksBeforeDialog?: boolean;
    prePlanMode?: PermissionMode;
}>;
```

---

## 三、核心辅助类型解析

### 3.1 ToolResult：工具返回值的统一包装

```typescript
export type ToolResult<Output> = {
    data: Output;
    newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[];
    contextModifier?: (context: ToolUseContext) => ToolUseContext;
    mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> };
};
```

这是一个精心设计的返回值包装，它允许工具不仅返回数据，还可以：

- `data`：工具处理后返回的主数据
- `newMessages`：注入新消息到对话历史（如命令执行前的警告）
- `contextModifier`：在调用链中修改后续工具看到的 Context（高级功能）
- `mcpMeta`：MCP 协议元数据（用于与 MCP 工具互操作）

### 3.2 工具进度机制

```typescript
export type ToolCallProgress<P extends ToolProgressData> = (progress: ToolProgress<P>) => void;
```

工具执行可以是异步的且耗时很长（如文件复制、代码搜索），进度机制用于实时反馈。进度是事件驱动的，通过 `onProgress` 回调传递。

### 3.3 工具验证结果

```typescript
export type ValidationResult =
    | { result: true }
    | { result: false; message: string; errorCode: number };
```

---

## 四、buildTool：工具工厂与默认值填充

这是 Tool.ts 中最重要的工具之一，它使用**默认值填充模式**，使得工具定义可以只提供差异化的配置，其他使用安全默认值。

### 4.1 设计思路

```typescript
const TOOL_DEFAULTS = {
    isEnabled: () => true,
    isConcurrencySafe: (_input?: unknown) => false,
    isReadOnly: (_input?: unknown) => false,
    isDestructive: (_input?: unknown) => false,
    checkPermissions: (input: { [key: string]: unknown }, _ctx?: ToolUseContext): Promise<PermissionResult> => Promise.resolve({ behavior: 'allow', updatedInput: input }),
    toAutoClassifierInput: (_input?: unknown) => '',
    userFacingName: (_input?: unknown) => '',
};
```

这些默认值遵循 **fail-closed 原则**（安全优先）：

| 默认值 | 设计考虑 |
| --- | --- |
| `isConcurrencySafe: false` | 假设工具不是并发安全的（保守） |
| `isReadOnly: false` | 假设工具会修改文件系统（安全） |
| `checkPermissions: 'allow'` | 通用权限检查通常在工具外部进行，工具自定义时从 `allow` 开始 |

### 4.2 类型级别的实现保障

```typescript
type BuiltTool<D> = Omit<D, DefaultableToolKeys> & {
    [K in DefaultableToolKeys]-?: K extends keyof D
        ? undefined extends D[K]
            ? ToolDefaults[K]
            : D[K]
        : ToolDefaults[K];
};
```

这是一个精妙的类型设计：

- 如果工具定义了某个属性（可选属性）且提供了值，使用工具提供的
- 如果工具定义了某个属性但未提供值（为`undefined`），使用默认值
- 如果工具未定义某个属性，使用默认值

这样在类型层面就保证了：工具接口上的必选方法在运行时一定存在。

### 4.3 实际使用示例

```typescript
// 文件工具中的典型使用
const FileReadTool = buildTool({
    name: FILE_READ_TOOL_NAME,
    
    // 只有差异化的配置需要写
    description(input, options) {
        // ...实现
    },
    
    call(input, context, canUseTool, parentMessage, onProgress) {
        // ...实现
    },
    
    isEnabled() {
        return true;
    },
    
    isReadOnly() {
        return true; // 读工具不需要写权限
    },
    
    isConcurrencySafe() {
        return true; // 读操作是并发安全的
    },
    
    // 渲染相关
    renderToolUseMessage,
    renderToolResultMessage,
    // ...其他渲染方法
    
    // 使用默认值的方法可以不写
    // isDestructive: 自动使用默认 false
    // checkPermissions: 自动使用默认 allow
    // toAutoClassifierInput: 自动使用默认 ''
});
```

---

## 五、工具系统的重要概念

### 5.1 Deferred Tools vs Always Load Tools

```typescript
readonly shouldDefer?: boolean;
readonly alwaysLoad?: boolean;
```

这是工具系统的一个关键性能优化：

| 模式 | 何时使用 | 工作原理 |
| --- | --- | --- |
| `shouldDefer: true` | 大量工具（如 MCP 工具） | 初始不加载工具 Schema，只在模型需要工具搜索时才加载 |
| `alwaysLoad: true` | 核心工具（Read、Write、Bash） | 总是加载完整 Schema，让模型第一时间可用 |

这种设计使得系统可以同时拥有：
- 大量工具的扩展性
- 启动的快速响应

### 5.2 搜索提示（searchHint）

```typescript
searchHint?: string; // "3-10 words, no trailing period"
```

用于工具搜索的关键词匹配（配合 `shouldDefer` 使用），帮助模型快速发现工具。

### 5.3 只读工具 vs 破坏性工具

```typescript
isReadOnly(input: Input): boolean;  // 工具是否只读
isDestructive?(input: Input): boolean;  // 工具是否会做不可逆操作（删除、覆盖）
```

这些信息用于：
- 权限策略判断
- UI 展示（危险操作需要高亮）
- 自动模式的决策

### 5.4 并发安全性

```typescript
isConcurrencySafe(input: Input): boolean;
```

判断工具是否可以被安全地并行调用。这对于工具批量执行很重要。

### 5.5 工具别名（aliases）

```typescript
aliases?: string[];
```

用于工具重命名时的向后兼容，让旧名称仍然能找到工具。

---

## 六、消息渲染层设计解析

工具系统包含完整的 UI 渲染方法，这是 Claude Code 设计的一个独特之处：**工具不仅能执行，还知道如何展示自己的执行过程和结果。**

### 6.1 渲染方法一览

| 方法 | 渲染内容 | 何时显示 |
| --- | --- | --- |
| `renderToolUseMessage` | 工具调用的输入参数 | 模型刚调用工具时 |
| `renderToolUseProgressMessage` | 工具执行中的进度 | 工具执行期间 |
| `renderToolUseQueuedMessage` | 工具在队列中等待执行 | 工具排队时（可选） |
| `renderToolUseRejectedMessage` | 工具被权限拒绝 | 用户拒绝工具使用时（可选） |
| `renderToolUseErrorMessage` | 工具执行出错 | 工具执行失败时（可选） |
| `renderToolResultMessage` | 工具执行成功的结果 | 工具执行完成后 |
| `renderToolUseTag` | 工具调用的附加标签 | 工具调用旁边（可选） |
| `renderGroupedToolUse` | 批量工具调用的分组渲染 | 多个同类工具调用时（可选） |

### 6.2 压缩展示相关功能

```typescript
isSearchOrReadCommand?(input: Input): {
    isSearch: boolean;
    isRead: boolean;
    isList?: boolean;
};
getToolUseSummary?(input: Partial<Input> | undefined): string | null;
isResultTruncated?(output: Output): boolean;
```

这三个方法配合，实现了工具结果的**智能压缩显示**：

1. 判断工具是否属于搜索/读/列表类型
2. 提供一个简短摘要替代完整展示
3. 告知 UI 结果是否被截断，需要展开查看

### 6.3 实际工具渲染示例分析

我们以 `FileReadTool` 的渲染逻辑为例：

```typescript
export function renderToolUseMessage(
    { file_path, offset, limit, pages }: Partial<Input>,
    { verbose }: { verbose: boolean }
): React.ReactNode {
    // 1. 特殊场景：agent 输出文件
    if (getAgentOutputTaskId(file_path)) {
        return '';  // 不在括号中显示任何内容，特殊逻辑处理
    }
    
    // 2. 正常场景
    const displayPath = verbose ? file_path : getDisplayPath(file_path);
    
    // 3. 有页面范围的情况（PDF 读取）
    if (pages) {
        return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>;
    }
    
    // 4. 有行范围的情况
    if (verbose && (offset || limit)) {
        // 渲染详细行范围
    }
    
    // 5. 默认：只显示路径
    return <FilePathLink filePath={file_path}>{displayPath}</FilePathLink>;
}
```

### 6.4 透明包装工具

```typescript
isTransparentWrapper?(): boolean;
```

有些工具本身不直接工作，而是包装其他工具调用（如 AgentTool）。这类工具标记为透明，让 UI 只显示内层工具的工作。

---

## 七、权限验证与检查流程

工具系统有三层验证机制：

```typescript
// 第一层：输入验证
validateInput?(input: Input, context: ToolUseContext): Promise<ValidationResult>;

// 第二层：权限检查
checkPermissions(input: Input, context: ToolUseContext): Promise<PermissionResult>;

// 第三层：预检查（实际调用时）
getPath?(input: Input): string;  // 可以用于路径合法性检查
preparePermissionMatcher?(input: Input): Promise<(pattern: string) => boolean>;
```

### 7.1 validateInput 设计用途

在工具执行前进行输入验证，目的是尽早失败，避免进入权限检查环节。

典型应用场景：
- 文件路径格式检查
- 命令格式验证
- 参数范围验证

### 7.2 checkPermissions 设计用途

这是工具自定义权限检查的钩子。虽然大多数权限检查通过通用的权限规则处理，但某些工具需要特殊逻辑。

### 7.3 preparePermissionMatcher 设计用途

这个方法允许工具把自己的输入转换为权限规则能理解的模式。例如：
- `BashTool` 可以把命令转换为 "git push" 这样的模式
- `FileEditTool` 可以把路径转换为完整路径模式

---

## 八、内存优化与结果管理

### 8.1 maxResultSizeChars 限制

```typescript
readonly maxResultSizeChars: number;  // 或者 Infinity
```

防止工具结果太大导致：
- 内存溢出
- Token 消耗过高
- UI 卡顿

当工具结果超过限制时，会被保存到文件并提供摘要替代。

`Read` 工具设置为 `Infinity`，原因是：
- Read 工具已经有自己的读取限制（offset/limit）
- Read->文件->Read 的循环可能产生问题

### 8.2 contentReplacementState 内容替换机制

```typescript
contentReplacementState?: ContentReplacementState;
```

用于工具结果的预算管理，当结果过大时进行替换。

---

## 九、搜索与自动分类相关设计

### 9.1 toAutoClassifierInput

```typescript
toAutoClassifierInput(input: Input): unknown;
```

将工具输入转换为安全分类器能够理解的输入。例如：
- 对 Bash，只提供命令部分，不提供敏感数据
- 对 Read，只提供文件名

### 9.2 extractSearchText

```typescript
extractSearchText?(output: Output): string;
```

从工具输出中提取可搜索的文本，用于历史记录搜索。

---

## 十、重要工具辅助函数

### 10.1 toolMatchesName — 名称匹配

```typescript
export function toolMatchesName(
    tool: { name: string; aliases?: string[] },
    name: string
): boolean {
    return tool.name === name || (tool.aliases?.includes(name) ?? false);
}
```

### 10.2 findToolByName — 工具查找

```typescript
export function findToolByName(tools: Tools, name: string): Tool | undefined {
    return tools.find(t => toolMatchesName(t, name));
}
```

---

## 十一、MCP 工具的特殊支持

Tool.ts 包含完整的 MCP（Model Context Protocol）工具支持：

```typescript
isMcp?: boolean;
mcpInfo?: { serverName: string; toolName: string };
mcpMeta?: { _meta?: Record<string, unknown>; structuredContent?: Record<string, unknown> };
```

这些使得 Claude Code 可以无缝集成 MCP 生态的所有工具。

---

## 十二、LSP 工具的特殊支持

```typescript
isLsp?: boolean;
```

用于 Language Server Protocol 工具的特殊标记。

---

## 十三、工具架构设计总结

### 13.1 Tool.ts 设计的关键亮点

| 设计 | 优点 |
| --- | --- |
| **接口与实现分离** | Tool.ts 是定义层，具体工具是实现层，关注点明确分离 |
| **声明式配置优先** | 大部分工具只需配置，减少样板代码 |
| **完整的生命周期设计** | 输入、验证、权限、执行、渲染、错误处理全程覆盖 |
| **依赖注入 Context** | ToolUseContext 把所有外部依赖集中管理，易于测试和扩展 |
| **可扩展的默认值模式** | buildTool + 默认值让工具定义非常简洁 |
| **UI 渲染与工具逻辑同处一体** | 工具自己知道如何展示自己，不需要外部映射表 |

### 13.2 工具开发的"快乐路径"

开发新工具的典型工作流：

1. 定义 `inputSchema`（Zod Schema）
2. 实现 `call` 方法
3. 实现 `description` 方法（用于系统提示词）
4. 实现关键渲染方法（至少 `renderToolUseMessage`、`renderToolResultMessage`）
5. 配置属性（`isReadOnly`、`isConcurrencySafe` 等）
6. 其他按需实现（权限检查、验证、进度等）

### 13.3 工具系统的职责边界

Tool.ts 关注：
- **工具接口定义**
- **工具工厂函数**
- **通用类型和辅助函数**

Tool.ts 不关注：
- **工具的具体实现**
- **工具的调度逻辑**
- **权限决策逻辑**（只是提供钩子）
- **工具的消息处理流程**

这种划分使得系统架构清晰，责任明确。

---

## 十四、工具示例：FileReadTool 分析

### 14.1 FileReadTool 的核心设计思想

FileReadTool 是一个**只读、并发安全、功能丰富**的工具，它的特点是：

- 支持多种文件类型：文本、图片、PDF、Jupyter Notebook
- 支持分页/分段读取
- 有文件变更检测，避免重复读取相同内容
- 丰富的渲染方式，针对不同文件类型差异化展示

### 14.2 FileReadTool 的关键实现

(内容省略，请参考 FileReadTool 的实际代码查看完整实现)

---

## 十五、常见问题与维护要点

### 15.1 何时添加新工具 vs 扩展现存工具

| 情景 | 推荐做法 |
| --- | --- |
| 完全不同的功能 | 添加新工具 |
| 相同功能的不同方式 | 扩展现存工具 |
| MCP 生态的工具 | 集成 MCP 工具而非重写 |

### 15.2 安全第一原则

- 新工具默认为非只读（除非明确证明安全）
- 破坏性操作必须有显式确认
- 工具输入验证应该尽量严格

### 15.3 性能考虑

- 工具结果大小必须受控（`maxResultSizeChars`）
- 考虑使用延迟加载（`shouldDefer: true`）
- 文件读取工具需要考虑内存使用

---

## 十六、总结

`Tool.ts` 是 Claude Code 工具系统的坚实基础，它通过精心设计的类型系统和接口规范，使得工具开发变得既规范又灵活。这套设计的优点在于：

1. **关注点分离**：接口定义与实现分离，通用功能与具体工具分离
2. **安全分层**：验证、权限、预检查分层处理
3. **开发体验好**：默认值填充、声明式配置让简单工具的开发非常简洁
4. **可扩展性强**：支持 MCP、LSP、延迟加载等高级功能
5. **用户体验完整**：从调用到进度到结果，工具知道如何展示每个环节

理解 Tool.ts，是理解 Claude Code 工具系统的第一步。
