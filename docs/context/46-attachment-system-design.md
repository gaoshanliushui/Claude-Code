# Attachment 系统：架构设计与实现分析

> 范围：`src/utils/attachments.ts`、`src/utils/messages.ts`（`normalizeAttachmentForAPI` / `normalizeMessagesForAPI`）、`src/components/messages/AttachmentMessage.tsx`、`src/components/messages/nullRenderingAttachments.ts`
>
> 本文档说明 Attachment 的职责、分类、生命周期，以及从生成到注入 LLM 请求的全流程。

---

## 一、什么是 Attachment

### 1.1 定义

`Attachment` 是一个**可辨识联合类型**（Discriminated Union），定义在 `src/utils/attachments.ts:440-717`。它表示"在用户消息之外，需要注入给模型看的额外上下文"。

与用户消息的区别：

| 维度 | 用户消息（UserMessage） | Attachment |
|------|----------------------|------------|
| 来源 | 用户键盘输入 | 系统自动产生 |
| 语义 | 用户意图 | 系统状态、上下文提示 |
| 注入方式 | 用户直接输入 | 附着在消息流中，以 `isMeta: true` 标记 |
| 可见性 | 用户可见 | 大部分对用户不可见（渲染为 null） |

### 1.2 核心作用

Attachment 系统的核心作用是**将系统层面的状态、上下文、提示和事件，以标准化格式注入到模型的对话历史中**，让模型获得超越用户输入之外的信息。具体包括：

1. **文件上下文**：用户 @ 引用的文件、IDE 中选中的行、已变更的文件
2. **系统指令**：plan mode 提示、auto mode 提示、critical system reminder
3. **环境状态**：token 用量、USD 预算、日期变更
4. **工具发现**：可用技能列表、MCP 工具变更、代理列表变更
5. **任务与事件**：后台任务状态、hook 执行结果、队友消息
6. **记忆与配置**：嵌套 CLAUDE.md、相关记忆、动态技能目录

---

## 二、Attachment 完整分类

### 2.1 文件相关

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `file` | `generateFileAttachment` | 用户 @ 引用的文件，包含完整内容 |
| `already_read_file` | `generateFileAttachment` | 文件已在上下文中且未修改，跳过重发 |
| `compact_file_reference` | `generateFileAttachment` | 压缩后的文件引用（仅路径，无内容） |
| `pdf_reference` | `tryGetPDFReference` | 大 PDF 的轻量引用（页数、大小） |
| `directory` | `processAtMentionedFiles` | @ 引用的目录内容列表 |
| `edited_text_file` | `getChangedFiles` | 被外部修改的文本文件 diff |
| `edited_image_file` | `getChangedFiles` | 被外部修改的图片文件 |

### 2.2 IDE 交互

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `selected_lines_in_ide` | `getSelectedLinesFromIDE` | 用户在 IDE 中选中的行 |
| `opened_file_in_ide` | `getOpenedFileFromIDE` | 用户在 IDE 中打开的文件 |

### 2.3 模式与指令

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `plan_mode` | `getPlanModeAttachments` | Plan mode 提示（完整/精简） |
| `plan_mode_exit` | `getPlanModeExitAttachment` | 退出 plan mode 通知 |
| `plan_mode_reentry` | `getPlanModeAttachments` | 重新进入 plan mode |
| `auto_mode` | `getAutoModeAttachments` | Auto mode 提示 |
| `auto_mode_exit` | `getAutoModeExitAttachment` | 退出 auto mode 通知 |
| `critical_system_reminder` | `getCriticalSystemReminderAttachment` | 关键系统提醒 |
| `output_style` | `getOutputStyleAttachment` | 输出风格配置 |

### 2.4 技能与工具

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `skill_listing` | `getSkillListingAttachments` | 可用技能列表 |
| `dynamic_skill` | `getDynamicSkillAttachments` | 动态发现的技能目录 |
| `skill_discovery` | `skillSearchModules.prefetch.getTurnZeroSkillDiscovery` | 技能搜索发现（feature-gated） |
| `deferred_tools_delta` | `getDeferredToolsDeltaAttachment` | 惰性工具变更通知 |
| `agent_listing_delta` | `getAgentListingDeltaAttachment` | 代理类型变更通知 |
| `mcp_instructions_delta` | `getMcpInstructionsDeltaAttachment` | MCP 指令变更通知 |
| `agent_mention` | `processAgentMentions` | 用户 @ 提及的代理 |

### 2.5 嵌套记忆与相关记忆

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `nested_memory` | `getNestedMemoryAttachments` | 嵌套 CLAUDE.md 等指令文件 |
| `relevant_memories` | `getRelevantMemoryAttachments`（通过 `startRelevantMemoryPrefetch`） | 自动记忆检索 |

### 2.6 任务与提醒

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `todo_reminder` | `getTodoReminderAttachments` | TodoWrite 提醒 |
| `task_reminder` | `getTaskReminderAttachments` | Task 工具提醒 |
| `task_status` | `getUnifiedTaskAttachments` | 后台任务状态更新 |
| `verify_plan_reminder` | `getVerifyPlanReminderAttachment` | 计划验证提醒 |

### 2.7 队列命令

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `queued_command` | `getQueuedCommandAttachments` / `getAgentPendingMessageAttachments` | 排队等待处理的用户、系统或代理消息；代理待处理消息也统一转换成此类型 |

### 2.8 诊断与 Hook

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `diagnostics` | `getDiagnosticAttachments` / `getLSPDiagnosticAttachments` | IDE 或 LSP 诊断信息；二者最终都生成 `diagnostics` 类型 |
| `async_hook_response` | `getAsyncHookResponseAttachments` | 异步 Hook 响应 |
| `hook_blocking_error` | Hook 系统 | Hook 阻塞错误 |
| `hook_non_blocking_error` | Hook 系统 | Hook 非阻塞错误 |
| `hook_error_during_execution` | Hook 系统 | Hook 执行错误 |
| `hook_success` | Hook 系统 | Hook 成功 |
| `hook_cancelled` | Hook 系统 | Hook 取消 |
| `hook_stopped_continuation` | Hook 系统 | Hook 阻止继续 |
| `hook_system_message` | Hook 系统 | Hook 系统消息 |
| `hook_additional_context` | Hook 系统 | Hook 额外上下文 |
| `hook_permission_decision` | Hook 系统 | Hook 权限决策 |

### 2.9 资源与参考

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `mcp_resource` | `processMcpResourceAttachments` | MCP 资源引用 |
| `plan_file_reference` | 外部 | 计划文件引用 |

### 2.10 用量与统计

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `token_usage` | `getTokenUsageAttachment` | Token 用量（环境变量开启） |
| `output_token_usage` | `getOutputTokenUsageAttachment` | 输出 token 用量（feature-gated） |
| `budget_usd` | `getMaxBudgetUsdAttachment` | USD 预算 |
| `compaction_reminder` | `getCompactionReminderAttachment` | 压缩提醒（feature-gated） |
| `context_efficiency` | `getContextEfficiencyAttachment` | 上下文效率提示（feature-gated） |

### 2.11 队友通信（swarm）

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `teammate_mailbox` | `getTeammateMailboxAttachments` | 队友消息 |
| `team_context` | `getTeamContextAttachment` | 团队上下文（仅首次） |
| `teammate_shutdown_batch` | 外部 | 队友批量关闭通知 |

### 2.12 其他

| 类型 | 生成函数 | 说明 |
|------|---------|------|
| `date_change` | `getDateChangeAttachments` | 日期变更通知 |
| `ultrathink_effort` | `getUltrathinkEffortAttachment` | 用户请求的推理努力级别 |
| `invoked_skills` | 外部 | 已调用的技能内容 |
| `command_permissions` | 外部 | 命令权限信息 |
| `current_session_memory` | 外部 | 当前会话记忆 |
| `max_turns_reached` | 外部 | 最大轮次已到 |
| `structured_output` | 外部 | 结构化输出 |
| `bagel_console` | 外部 | Bagel 控制台输出 |
| `companion_intro` | `getCompanionIntroAttachment` | 伙伴介绍（feature-gated） |

---

## 三、生成流程：`getAttachments`（第一入口）

### 3.1 入口函数

```
getAttachments(input, toolUseContext, ideSelection, queuedCommands, messages?, querySource?, options?)
  → Promise<Attachment[]>
```

`src/utils/attachments.ts:743`

### 3.2 三阶段并行采集

`getAttachments` 将附件生成任务分为**三个独立批次**，并行执行：

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1: 用户输入附件（userInputAttachments）                        │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│   │ @文件提取     │  │ MCP资源提取  │  │ 代理提及 + 技能发现  │    │
│   │ processAt-   │  │ processMcp- │  │ processAgentMentions │    │
│   │ MentionedFiles│  │ResourceAtt. │  │ + skill_discovery    │    │
│   └──────────────┘  └──────────────┘  └──────────────────────┘    │
│ 必须率先完成，为 Phase 2 的 nested_memory 提供 trigger 文件路径       │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 2: 线程安全附件（allThreadAttachments — 主线程/子代理均可）     │
│   ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌──────────────────┐ │
│   │queued_cmd │ │date_change│ │changed_   │ │nested_memory     │ │
│   │           │ │           │ │files      │ │                  │ │
│   ├───────────┤ ├───────────┤ ├───────────┤ ├──────────────────┤ │
│   │plan_mode  │ │auto_mode  │ │deferred_  │ │skill_listing     │ │
│   │           │ │           │ │tools_delta│ │                  │ │
│   ├───────────┤ ├───────────┤ ├───────────┤ ├──────────────────┤ │
│   │agent_list-│ │mcp_instru-│ │teammate_  │ │team_context      │ │
│   │ing_delta  │ │ctions_delta│ │mailbox    │ │                  │ │
│   ├───────────┤ ├───────────┤ ├───────────┤ ├──────────────────┤ │
│   │todo_      │ │critical_  │ │compaction_│ │context_efficiency│ │
│   │reminder   │ │sys_reminder│ │reminder   │ │                  │ │
│   └───────────┘ └───────────┘ └───────────┘ └──────────────────┘ │
│ 所有子代理共享，不依赖主线程独有状态                                  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 3: 主线程专用附件（mainThreadAttachments — 仅主线程）          │
│   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐             │
│   │IDE 选中行     │ │IDE 打开文件  │ │输出风格      │             │
│   │getSelected-  │ │getOpenedFile-│ │getOutput-   │             │
│   │LinesFromIDE  │ │FromIDE      │ │StyleAtt.    │             │
│   ├──────────────┤ ├──────────────┤ ├──────────────┤             │
│   │诊断信息      │ │LSP 诊断      │ │统一任务状态  │             │
│   │getDiagnostic│ │getLSP-      │ │getUnified-  │             │
│   │Attachments  │ │DiagnosticAtt│ │TaskAtt.     │             │
│   ├──────────────┤ ├──────────────┤ ├──────────────┤             │
│   │异步Hook响应  │ │Token 用量    │ │USD 预算      │             │
│   │getAsyncHook- │ │getToken-    │ │getMaxBudget-│             │
│   │ResponseAtt.  │ │UsageAtt.    │ │UsdAtt.      │             │
│   ├──────────────┤ ├──────────────┤ ├──────────────┤             │
│   │OutputToken   │ │ VerifyPlan  │ │             │             │
│   │UsageAtt.     │ │ReminderAtt. │ │             │             │
│   └──────────────┘ └──────────────┘ └──────────────┘             │
│ 这些附件依赖主线程独有状态或非并发安全实现                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 容错机制：`maybe` 包装器

每个附件生成器都包裹在 `maybe(label, fn)` 中（`attachments.ts:1006`）：

- 捕获所有同步和异步异常，返回 `[]` 而非抛出
- 5% 采样记录耗时和大小到分析事件
- 超时机制：整个 `getAttachments` 有 1 秒超时（`setTimeout → abortController.abort()`）

这意味着**任何单个附件生成器的失败都不会阻塞整个流程**，体现了"尽力而为"的设计哲学。

---

## 四、消息转换：`getAttachmentMessages`（第二入口）

### 4.1 函数签名

```
getAttachmentMessages(input, toolUseContext, ideSelection, queuedCommands, messages?, querySource?, options?)
  → AsyncGenerator<AttachmentMessage, void>
```

`src/utils/attachments.ts:2939`

### 4.2 作用

1. 调用 `getAttachments` 采集所有附件
2. 通过 `createAttachmentMessage` 将每个 `Attachment` 包裹为 `AttachmentMessage`
3. 以 AsyncGenerator 逐个 yield 给调用方

### 4.3 调用点

`getAttachmentMessages` 在两个关键位置被调用：

| 调用点 | 文件位置 | 时机 | 说明 |
|--------|---------|------|------|
| `processUserInput` | `src/utils/processUserInput/processUserInput.ts:505` | 用户输入处理阶段 | 首次 turn 的附件提取 |
| `query.ts` | `src/query.ts:1594` | 每轮工具循环后 | 中间 turn 的附件（如 queued_commands 中继） |

---

## 五、API 归一化：`normalizeAttachmentForAPI`（第三阶段）

### 5.1 函数

```
normalizeAttachmentForAPI(attachment: Attachment) → UserMessage[]
```

`src/utils/messages.ts:3505`

### 5.2 作用

将每个 `Attachment` 对象转换为**模型可理解的 `UserMessage` 数组**。大部分附件类型被包装为 `isMeta: true` 的 `UserMessage`，其内容通常包裹在 `<system-reminder></system-reminder>` 标签中，使模型知道这是系统注入的上下文而非用户消息。

### 5.3 转换策略

不同附件类型采用不同的转换策略：

| 策略 | 示例类型 | 说明 |
|------|---------|------|
| **模拟工具调用** | `directory`, `file` | 构造 `tool_use` + `tool_result` 对，模拟 FileRead 或 bash 调用 |
| **系统提醒** | `plan_mode`, `auto_mode`, `edited_text_file` | 包裹在 `<system-reminder>` 中的纯文本，`isMeta: true` |
| **纯消息** | `token_usage`, `budget_usd` | 包裹在 `<system-reminder>` 中的简短提示 |
| **空转换** | `hook_success`, `hook_cancelled` | 返回 `[]`，不在模型中注入任何内容 |
| **特殊处理** | `teammate_mailbox`, `skill_discovery` | 在 `switch` 之前由 feature-gated 的 `if` 分支处理 |

### 5.4 转换到 `normalizeMessagesForAPI`

`normalizeMessagesForAPI`（`src/utils/messages.ts:2013`）是消息进入 API 层之前的最后一步处理。在 `case 'attachment':` 分支（`:2295`）中：

1. 调用 `normalizeAttachmentForAPI` 将 Attachment 转为 UserMessage[]
2. 可选的 feature-gated 后处理（`tengu_chair_sermon`）
3. 如果上一个消息已经是 UserMessage，合并内容
4. 否则追加到结果数组

---

## 六、UI 渲染

### 6.1 `AttachmentMessage.tsx`

`src/components/messages/AttachmentMessage.tsx` 负责在终端中渲染附件。

- 少部分附件类型有**可视化渲染**（如 `file` 显示行数、`selected_lines_in_ide` 显示选中行数）
- 大部分附件类型在 `NULL_RENDERING_TYPES` 中定义，渲染为 `null`（不可见）

### 6.2 空渲染优化

`src/components/messages/nullRenderingAttachments.ts`:

- 定义 `NULL_RENDERING_TYPES` 常量数组，包含 30+ 种在 UI 中无可见输出的附件类型
- `Messages.tsx` 在渲染前用 `isNullRenderingAttachment` 过滤这些消息
- 这样做**不消耗 200 条消息的渲染预算**，避免不可见附件占用 UI 资源

### 6.3 编译时强制

`AttachmentMessage.tsx` 的 `switch` 的 `default` 分支包含：

```typescript
attachment.type satisfies NullRenderingAttachmentType | 'skill_discovery' | 'teammate_mailbox' | 'bagel_console'
```

这确保**新增的 Attachment 类型必须**要么在 switch 中有显式 case，要么在 `NULL_RENDERING_TYPES` 中有条目——否则 TypeScript 编译错误。

---

## 七、关键设计约束

### 7.1 去重机制

| 去重目标 | 机制 | 文件 |
|---------|------|------|
| 已读文件 | `readFileState` LRU 缓存 | `getChangedFiles` 中检查 `mtime ≤ timestamp` |
| 已加载记忆 | `loadedNestedMemoryPaths` Set | `memoryFilesToAttachments` |
| 已发送技能 | `sentSkillNames` Map（按 agentId 隔离） | `getSkillListingAttachments` |
| 已注入记忆 | `readFileState` 跨轮次累积 | `filterDuplicateMemoryAttachments` |
| 当前轮记忆 | `alreadySurfaced` Set | `collectSurfacedMemories` |

### 7.2 权限检查

大多数文件读取类附件在生成前执行：

```typescript
isFileReadDenied(filename, appState.toolPermissionContext)
```

### 7.3 预算控制

| 约束 | 限制 | 说明 |
|------|------|------|
| 记忆文件行数 | 200 行 | `MAX_MEMORY_LINES` |
| 记忆文件字节数 | 4096 bytes | `MAX_MEMORY_BYTES` |
| 关联记忆会话累积 | 60 KB | `RELEVANT_MEMORIES_CONFIG.MAX_SESSION_BYTES` |
| 关联记忆每轮注入 | 5 个文件 | `findRelevantMemories` 的 `slice(0, 5)` |
| 记忆去重 | 跨轮次 | `readFileState` 累积追踪 |
| 技能列表过滤 | 30 个 | `FILTERED_LISTING_MAX` |

### 7.4 类型安全

- `Attachment` 是**可辨识联合类型**，每个分支有精确的 `type` 字段
- `normalizeAttachmentForAPI` 的 `switch` 需要全覆盖（`useExhaustiveSwitchCases` eslint 规则，但因 `teammate_mailbox`/`skill_discovery`/`bagel_console` 的 feature-gated 处理而放宽）
- `AttachmentMessage.tsx` 的 `default` 分支使用 `satisfies` 确保新类型必须被处理

---

## 八、性能优化

### 8.1 并行采集

`getAttachments` 将附件生成分为三个批次，每个批次内部使用 `Promise.all` 并行执行。30+ 个附件生成器并发运行，互不阻塞。

### 8.2 超时保护

顶层 `getAttachments` 设置 1 秒超时，超时后 `abortController.abort()` 终止仍在执行的附件生成器。

### 8.3 懒加载与预取

| 策略 | 实现 | 说明 |
|------|------|------|
| 记忆预取 | `startRelevantMemoryPrefetch` | 在用户消息处理时异步启动，收集时零等待 |
| 技能发现预取 | `startSkillDiscoveryPrefetch` | 同样异步，避免阻塞主循环 |
| 动态技能发现 | 仅在文件操作触发时才执行 | 避免无谓的目录扫描 |

### 8.4 实时计算

`getDateChangeAttachments` 使用 `setLastEmittedDate` 缓存上次日期，避免每轮都重复计算。

### 8.5 采样日志

`maybe` 包装器仅 5% 的调用记录日志，减少日志 I/O 压力。

---

## 九、扩展指南：如何添加新的 Attachment 类型

### 步骤

1. **在 `src/utils/attachments.ts` 中定义类型**

   在 `Attachment` 联合类型中添加新分支，确保 `type` 字段是字符串字面量。

2. **实现生成函数**

   - 如果归入某阶段，在 `getAttachments` 的对应批次中添加 `maybe` 调用
   - 使用 `maybe(label, fn)` 包装以保证容错

3. **在 `normalizeAttachmentForAPI` 中添加转换**

   在 `src/utils/messages.ts` 的 switch 中添加 case，将新类型转为 `UserMessage[]`。通常策略是包裹在 `<system-reminder>` 中。

4. **在 `AttachmentMessage.tsx` 中添加渲染（可选）**

   - 如果需要在终端中显示，添加 switch case 返回 React 节点
   - 如果不需要显示，在 `NULL_RENDERING_TYPES` 中添加类型名

5. **在 `nullRenderingAttachments.ts` 中添加条目（如果需要隐藏）**

### 注意事项

- **性能**：使用 `maybe` 包装，保证失败不阻塞
- **类型安全**：确保 `satisfies` 检查通过
- **去重**：考虑是否需要在 `readFileState` / `sentSkillNames` 等机制中注册
- **权限**：如果涉及文件读取，添加 `isFileReadDenied` 检查
- **线程安全**：考虑子代理是否也需要此附件（Phase 2 vs Phase 3）
- **feature-gated**：如果仅在 Anthropic 内部使用，加 `feature('FLAG')` 条件

---

## 十、完整生命周期

```
用户输入 → processUserInput
  │
  ├── getAttachmentMessages → getAttachments (Phase 1-3)
  │     └── createAttachmentMessage → AttachmentMessage[]
  │
  └── processTextPrompt(..., attachmentMessages)
        └── 返回 { messages: [userMsg, ...attachmentMessages] }
              │
              ↓ 进入 query.ts 主循环
         queryLoop()
              │
              ├── 每轮工具循环后再次调用 getAttachmentMessages
              │     (用于 queued_commands 中继、记忆预取收集)
              │
              └── 发送到 API 层
                    │
                    ├── normalizeMessagesForAPI
                    │     └── case 'attachment':
                    │           └── normalizeAttachmentForAPI
                    │                 └── UserMessage[] (isMeta: true)
                    │
                    └── claude.ts → Anthropic API
                          │
                          └── 响应流回 → UI 渲染
                                └── AttachmentMessage.tsx
                                      ├── 有 UI 类型 → 渲染可见内容
                                      └── null 渲染类型 → 过滤（不占渲染预算）
```

---

## 参考

- `src/utils/attachments.ts` — Attachment 类型定义、所有生成函数、`getAttachments`、`getAttachmentMessages`
- `src/utils/messages.ts` — `normalizeAttachmentForAPI`、`normalizeMessagesForAPI`、`reorderAttachmentsForAPI`
- `src/components/messages/AttachmentMessage.tsx` — 终端渲染
- `src/components/messages/nullRenderingAttachments.ts` — 空渲染类型定义
- `src/query.ts:1594` — 中间 turn 附件调用点
- `src/utils/processUserInput/processUserInput.ts:505` — 首次 turn 附件调用点
- `src/services/api/claude.ts:1267` — API 层消息归一化