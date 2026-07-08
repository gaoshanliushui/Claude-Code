# 工具权限控制执行流程

> 本文梳理 `src/utils/permissions/permissions.ts` 的核心功能与完整执行流程，
> 解释 Claude Code 在每次工具调用前如何决定 **allow / ask / deny**。

---

## 1. 模块定位

`permissions.ts` 是 Claude Code **工具权限系统的调度中心**，被
`CanUseToolFn` 类型约束，在每次工具执行前由工具执行流程调用。

主要决策来源：

| 来源 | 说明 |
| --- | --- |
| **规则（rules）** | allow / deny / ask 三种规则，来源含 settings、CLI 参数、会话、命令 |
| **工具自身** | `tool.checkPermissions()`（Bash 子命令、Agent 子类型、Edit 路径等细分规则） |
| **模式（mode）** | `default` / `acceptEdits` / `plan` / `bypassPermissions` / `dontAsk` / `auto` |
| **Hook** | `PreToolUse` / `PermissionRequest` |
| **AI 分类器** | auto 模式下的 YOLO Classifier |

---

## 2. 关键数据结构

### 2.1 `PermissionDecision`（决策结果）

- `behavior: 'allow' | 'ask' | 'deny'`
- `message?: string` —— 面向用户的解释
- `decisionReason?: PermissionDecisionReason` —— 触发原因
- `updatedInput?` —— 工具被允许时使用的输入（可能经过工具修改）
- `suggestions?` —— 用户授权时可一键采纳的规则建议

### 2.2 `PermissionRule`（规则）

- `source: PermissionRuleSource` —— 规则来源
- `ruleBehavior: 'allow' | 'deny' | 'ask'`
- `ruleValue: PermissionRuleValue` —— `{ toolName, ruleContent? }`

### 2.3 `PermissionRuleSource`（规则来源）

`PERMISSION_RULE_SOURCES` 包含：

- `SETTING_SOURCES`（`policySettings` / `userSettings` / `projectSettings` / `localSettings` / `flagSettings`）
- `cliArg`
- `command`
- `session`

---

## 3. 总体执行流程（鸟瞰图）

```
                ┌────────────────────────────────┐
                │   hasPermissionsToUseTool     │   ← 主入口（CanUseToolFn）
                │   - 取上下文、做最终 mode 转换   │
                └──────────────┬─────────────────┘
                               │
                ┌──────────────▼─────────────────┐
                │  hasPermissionsToUseToolInner  │   ← 核心 7 步管线
                │  step 1a-1g: 拒绝链路           │
                │  step 2a-2b: 允许链路           │
                │  step 3:     passthrough → ask │
                └──────────────┬─────────────────┘
                               │
                ┌──────────────▼─────────────────┐
                │  结果为 ask 时的后处理           │
                │  - dontAsk:   ask → deny       │
                │  - auto:      AI classifier    │
                │  - headless:  hook → auto-deny │
                └────────────────────────────────┘
```

---

## 4. 核心管线详解（`hasPermissionsToUseToolInner`）

### 4.1 拒绝检查链路（step 1a – 1g）

按"短路优先级"依次执行，命中任意一步即返回：

| 步骤 | 检查内容 | 命中后行为 |
| --- | --- | --- |
| **1a** | 整工具 deny 规则（`getDenyRuleForTool`） | 返回 `deny` |
| **1b** | 整工具 ask 规则（`getAskRuleForTool`），但开启 `autoAllowBashIfSandboxed` 时被沙箱化的 Bash 命令可豁免 | 返回 `ask` |
| **1c** | 调用 `tool.checkPermissions()`（处理内容级规则，如 `Bash(npm publish:*)`） | — |
| **1d** | 工具自身返回 `deny`（如 Bash 检测到 `rm -rf /`） | 返回 `deny` |
| **1e** | 工具需要用户交互（如 `PermissionPromptTool`） | 返回 `ask` |
| **1f** | 工具返回内容级 ask 规则（即使在 bypass 模式下也优先于 bypass） | 返回 `ask` |
| **1g** | 安全检查（`.git/`、`.claude/`、`.vscode/`、shell 配置等敏感路径） | 返回 `ask` |

> **关键设计**：1a / 1d / 1f / 1g 在 `bypassPermissions` 模式下仍然生效 —— 这是"安全门控"。

### 4.2 允许检查链路（step 2a – 2b）

| 步骤 | 检查内容 | 命中后行为 |
| --- | --- | --- |
| **2a** | `bypassPermissions` 模式，或由 bypass 进入的 plan 模式 | 返回 `allow`（mode 原因） |
| **2b** | 整工具 allow 规则（`toolAlwaysAllowedRule`，如 `"Bash"` 而非 `"Bash(prefix:*)"`） | 返回 `allow`（rule 原因） |

> 命中时通过 `getUpdatedInputOrFallback` 保留 `tool.checkPermissions` 对 input 的可能修改。

### 4.3 Passthrough → Ask 转换（step 3）

若 `tool.checkPermissions()` 返回 `passthrough`（"我不反对，由上层决定"），
则统一转为 `ask`，并附带规则化的提示消息。

---

## 5. 主入口后处理（`hasPermissionsToUseTool`）

`hasPermissionsToUseToolInner` 给出"裸"决策后，外层根据当前模式做后处理：

### 5.1 结果为 `allow`

在 `auto` 模式下：若 `denialTracking.consecutiveDenials > 0`，调用 `recordSuccess`
重置计数（一次成功就打断"连续拒绝"streak，避免误触发上限）。

### 5.2 结果为 `ask`

按当前模式分流：

```
                ┌─────────────────┐
                │ behavior === ask│
                └────────┬────────┘
                         │
         ┌───────────────┼──────────────────┬────────────────┐
         ▼               ▼                  ▼                ▼
   ┌──────────┐    ┌──────────┐      ┌──────────────┐  ┌─────────────┐
   │ dontAsk  │    │   auto   │      │ headless /   │  │  普通交互    │
   │ ask→deny │    │ AI 分类器 │      │ async agent  │  │   直接 ask   │
   └──────────┘    └──────────┘      └──────────────┘  └─────────────┘
```

#### 5.2.1 `dontAsk` 模式

直接将 `ask` 转为 `deny`，附 `DONT_ASK_REJECT_MESSAGE` 提示。

#### 5.2.2 `auto` 模式（AI 分类器）

执行顺序：

1. **安全检查豁免**：若 `decisionReason.type === 'safetyCheck'` 且不可分类器处理，
   在 headless 下转为 `deny`（asyncAgent 原因）；否则放行走原结果。
2. **需用户交互的工具**：直接返回 ask（不进入分类器）。
3. **PowerShell 拦截**：未开启 `POWERSHELL_AUTO_MODE` 时直接放行原结果。
4. **快速路径 1（acceptEdits fast-path）**：
   用 `acceptEdits` 模式重跑一次 `tool.checkPermissions`；若能通过，
   直接 allow，省去昂贵的分类器调用。
   跳过 `Agent` 与 `REPL` 工具（它们的 checkPermissions 在 acceptEdits 下会直接 allow，
   但分类器必须看到完整的 JavaScript / 子工具调用才能判断）。
5. **快速路径 2（安全工具白名单）**：
   `classifierDecisionModule.isAutoModeAllowlistedTool(tool.name)` 为 true 时直接 allow。
6. **YOLO Classifier 调用**：
   `classifyYoloAction(messages, action, tools, permissionContext, signal)`
   记录 usage / duration / cost / 阶段信息到 analytics。
7. **分类器结果处理**：

   | 分类器结果 | 行为 |
   | --- | --- |
   | `transcriptTooLong`（不可恢复） | headless：AbortError；否则降级为 `ask` 让用户接管 |
   | `unavailable` 且 `tengu_iron_gate_closed` 开启 | `deny`（fail-closed） |
   | `unavailable` 且 fail-open | 降级为 `ask` |
   | `shouldBlock` | 累计 denial 数；命中上限则 `ask` 让用户接管；否则 `deny` |
   | 通过 | 累计 success；返回 `allow`（classifier 原因） |

8. **Denial Limit 处理（`handleDenialLimitExceeded`）**：
   - 连续拒绝数 / 总拒绝数 达到 `DENIAL_LIMITS` 上限时
     - headless：`throw AbortError`
     - CLI：返回 `ask`（让用户接管），同时记录 `tengu_auto_mode_denial_limit_exceeded`
     - 达到 total 上限会重置计数（避免永久无法自恢复）

#### 5.2.3 Headless / Async Agent

先执行 `runPermissionRequestHooksForHeadlessAgent`：

- 遍历 `PermissionRequest` hook
- 若 hook 返回 `allow`：可能附带 `updatedPermissions`，会被持久化并更新 React state
- 若 hook 返回 `deny`：可选择 `interrupt`（abort 会话）
- 若所有 hook 都没给出决策 → 自动 `deny`（asyncAgent 原因），附 `AUTO_REJECT_MESSAGE`

---

## 6. 工具自身规则：`checkPermissions` 协议

每个 `Tool` 实现 `checkPermissions(input, context)`，返回 `PermissionResult`：

- `behavior: 'allow' | 'deny' | 'ask' | 'passthrough'`
- `message?` / `decisionReason?` / `suggestions?` / `updatedInput?`

典型实现举例：

- **BashTool**：解析 `command`，按子命令前缀匹配规则；多条子命令用 `subcommandResults`
  聚合（type=ask 时只显示被拒绝/需询问的部分）。
- **AgentTool**：按 `agentType` 匹配 deny / allow 规则。
- **Edit / WriteTool**：用 `checkPathSafetyForAutoEdit` 检测敏感路径，返回 `safetyCheck`。

---

## 7. 规则生命周期

### 7.1 删除规则：`deletePermissionRule`

| 来源 | 行为 |
| --- | --- |
| `policySettings` / `flagSettings` / `command` | **拒绝**（只读源，抛错） |
| `localSettings` / `userSettings` / `projectSettings` | `deletePermissionRuleFromSettings` 写入磁盘 |
| `cliArg` / `session` | 仅内存，无需持久化 |

通过 `applyPermissionUpdate` + `setToolPermissionContext` 更新 React state。

### 7.2 同步规则：`syncPermissionRulesFromDisk`

磁盘设置变更后调用：

1. 若 `allowManagedPermissionRulesOnly`：先清空所有非 policy 源
2. 先用 `replaceRules: []` 清空所有磁盘源（旧规则不会残留）
3. 通过 `convertRulesToUpdates` 把新规则转换为 `replaceRules` 并应用

### 7.3 初始加载：`applyPermissionRulesToPermissionContext`

用 `addRules` 语义把规则叠加到现有上下文（不会覆盖已有规则）。

---

## 8. 模式（Mode）速查表

| 模式 | 行为 |
| --- | --- |
| `default` | 工具需询问；用户回答后形成新规则 |
| `acceptEdits` | 文件编辑类工具自动放行；其他仍询问 |
| `plan` | 只允许只读工具；写入类需询问 |
| `bypassPermissions` | 跳过 2a-2b 的允许链路；仍尊重 deny / safetyCheck / ask 规则 |
| `dontAsk` | 所有 ask 立即转 deny（强制拒绝） |
| `auto` | AI 分类器代理决策；带快速路径与降级 |

---

## 9. 关键辅助 API

### 9.1 规则查询

- `getAllowRules(context)` / `getDenyRules(context)` / `getAskRules(context)`：聚合某类规则
- `toolAlwaysAllowedRule(context, tool)`：整工具 allow 匹配
- `getDenyRuleForTool(context, tool)` / `getAskRuleForTool(context, tool)`：整工具 deny / ask 匹配
- `getDenyRuleForAgent(context, agentToolName, agentType)`：`Agent(x)` 语法匹配
- `filterDeniedAgents(agents, context, agentToolName)`：批量过滤
- `getRuleByContentsForTool(context, tool, behavior)`：内容级规则映射

### 9.2 提示生成

- `createPermissionRequestMessage(toolName, decisionReason)`：根据决策原因类型
  生成用户友好的提示文本（支持 rule / hook / classifier / mode / subcommandResults
  / sandboxOverride / workingDir / safetyCheck / asyncAgent 等分类）

### 9.3 拒绝追踪

- `createDenialTrackingState()` / `recordSuccess(state)` / `recordDenial(state)` /
  `shouldFallbackToPrompting(state)`：auto 模式下分类器的连续 / 总拒绝计数
- `persistDenialState(context, state)`：普通场景写 appState；async 子 agent 写 `localDenialTracking`
- `handleDenialLimitExceeded(...)`：达到上限时降级为 ask 或 abort

### 9.4 仅规则检查

- `checkRuleBasedPermissions(tool, input, context)`：只跑 step 1a-1g，不跑 mode 转换、
  hook、分类器。用于 PreToolUse hook 预检 / bypass 模式前置检查。

---

## 10. 完整调用链示例

以"用户敲了一条 `npm publish`"为例：

```
1. ToolUseContext 流程调用 hasPermissionsToUseTool(Bash, {command:"npm publish"})
2. hasPermissionsToUseToolInner:
   - 1a-1b: 整工具 deny/ask 规则未命中
   - 1c: Bash.checkPermissions 解析命令，命中规则 Bash(npm publish:*) → ask (rule)
   - 1d-1g: 提前在 1f 返回 ask
3. hasPermissionsToUseTool 主入口看到 ask：
   - 模式 = default → 直接返回 ask
4. UI 弹出权限询问，附 suggestions（"Bash(npm publish:*)"）
5. 用户选"始终允许" → 调用 applyPermissionUpdate 把规则写到 projectSettings
6. 下一次同样命令 → 命中 allow 规则 → 直接放行
```

若同样的命令发生在 `auto` 模式下：

```
1-2. 同上，结果为 ask (rule, content-specific ask)
3. 主入口：模式 = auto → 走 AI 分类器
4. 快速路径：
   - acceptEdits 不允许 npm publish（不是文件编辑）
   - Bash 不在白名单
5. 调用 classifyYoloAction → 分类器根据工具调用上下文判定
   - 判定为安全 → 允许
   - 判定为危险 → deny（带 reason 进入 denial tracking）
6. 连续 / 总拒绝达上限 → 降级为 ask 让用户接管
```

---

## 11. 设计要点回顾

1. **多层决策来源**：规则 → 工具自身 → 模式 → Hook → AI 分类器，可逐层叠加也可短路。
2. **安全优先**：deny 规则、安全检查、需用户交互的工具在任何模式下都生效。
3. **降级策略**：分类器不可用 / transcript 超长 / 拒绝过多时一律降级为人工询问，避免误判永久卡死。
4. **状态隔离**：async 子 agent 用 `localDenialTracking` 避开全局 React state，保证并发隔离。
5. **规则持久化分级**：磁盘源（user/project/local）、会话源、CLI 参数、命令源各有不同写入语义。

---

## 12. 相关文件

| 文件 | 作用 |
| --- | --- |
| `src/utils/permissions/permissions.ts` | 本文档主体 |
| `src/utils/permissions/PermissionResult.ts` | 决策类型定义 |
| `src/utils/permissions/PermissionRule.ts` | 规则类型定义 |
| `src/utils/permissions/PermissionMode.ts` | 模式定义与标题 |
| `src/utils/permissions/PermissionUpdate.ts` | 规则增删改查 |
| `src/utils/permissions/permissionsLoader.ts` | 规则从 settings 加载 |
| `src/utils/permissions/denialTracking.ts` | auto 模式拒绝计数 |
| `src/utils/permissions/yoloClassifier.ts` | YOLO 分类器 |
| `src/utils/permissions/classifierDecision.ts` | 分类器快速路径 / 白名单 |
| `src/utils/permissions/permissionRuleParser.ts` | 规则字符串解析 |
| `src/hooks/useCanUseTool.ts` | `CanUseToolFn` 类型 |
| `src/tools/BashTool/toolName.ts` 等 | 各工具的 `checkPermissions` 实现 |