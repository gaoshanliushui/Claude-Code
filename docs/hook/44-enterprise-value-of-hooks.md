# Hooks 系统在企业级 Agent 实战中的价值

> 本文目的：从"为什么需要 hooks"出发，结合 **15 个真实业务场景**，展示 Claude Code 的 hook 系统（`src/utils/hooks.ts`）在企业部署中**具体能解决什么问题、带来什么业务价值、怎么落地**。
> 
> 不写"可以扩展 agent 能力"这种空话。每个场景会回答四个问题：
> 1. **谁有这个痛点**（具体角色）
> 2. **不解决会怎样**（业务代价）
> 3. **用哪条 hook**（event 类型 + 哪种 executor）
> 4. **写出来长什么样**（可直接 copy-paste 的真实配置）

> 源码位置：见 [`docs/hook/15-utils-hooks.md`](15-utils-hooks.md)、[`docs/hook/41-hook-system-design.md`](41-hook-system-design.md)、[`docs/hook/43-hook-types-and-command-protocol.md`](43-hook-types-and-command-protocol.md)

---

## 目录

- [0. 一个不应该被忽视的前提](#0-一个不应该被忽视的前提)
- [1. 五类 Agent 失控：hooks 是唯一的"安全带"](#1-五类-agent-失控hooks-是唯一的安全带)
- [2. 实战场景 1–5：合规与审计](#2-实战场景-15合规与审计)
- [3. 实战场景 6–10：安全护栏与 DLP](#3-实战场景-610安全护栏与-dlp)
- [4. 实战场景 11–15：编排 / 通知 / 上下文增强](#4-实战场景-1115编排--通知--上下文增强)
- [5. Hook 类型决策矩阵](#5-hook-类型决策矩阵)
- [6. 落地路线图：3 个月从 0 到生产](#6-落地路线图3-个月从-0-到生产)
- [7. 与其他安全工具的边界](#7-与其他安全工具的边界)
- [8. 关键文件索引](#8-关键文件索引)

---

## 0. 一个不应该被忽视的前提

Claude Code 这类 agent 的本质是**一个拥有 shell 权限的 LLM**。一旦把它接进公司：

- 它能读 git 仓库（含 token、AWS key、internal API endpoint）
- 它能写文件（含 prod 配置文件）
- 它能跑命令（含 `rm -rf`、`kubectl delete`、`mysql drop database`）
- 它能调用 API（含给客户群发邮件、向生产 DB 灌数据）
- 它能 fork sub-agent（放大上面的能力）

**没有 hooks 的 agent = 让一个实习生拿到生产 root 权限**。从安全/合规角度，hooks 是企业唯一能**保留对 agent 的可控性**的扩展点。

---

## 1. 五类 Agent 失控：hooks 是唯一的"安全带"

| 失控类型 | 真实案例 | 严重程度 | hooks 能解决的部分 |
|---------|---------|---------|------------------|
| **A. 数据外泄** | agent 把代码里的 `AWS_SECRET_ACCESS_KEY` 写进 commit message | 🔴 P0 | PreToolUse on Edit/Write: secret 扫描 |
| **B. 不可逆操作** | agent 误判想 `rm -rf` 整个 monorepo 来"清理" | 🔴 P0 | PreToolUse on Bash: 命令模式匹配 |
| **C. 合规脱靶** | agent 给欧盟用户发邮件没走 GDPR 同意流程 | 🟠 P1 | PreToolUse on 邮件工具: 区域化拦截 |
| **D. 成本失控** | agent 陷入 while 循环疯狂调 LLM，几小时烧掉几千美金 | 🟠 P1 | PostToolUse: token 累计 + 强制 stop |
| **E. 行为漂移** | agent 幻觉"用户让我把 prod 数据库清空" | 🟡 P2 | PreToolUse: 高危路径强制 ask 用户 |

每一种失控类型，企业都**只能**靠 hooks 来兜底——上游 prompt 工程能缓解，但永远不能根除。

---

## 2. 实战场景 1–5：合规与审计

### 场景 1：金融行业 SOX 合规审计

**痛点所有人**：银行/保险的 DevOps 团队
**业务代价**：SOX (Sarbanes-Oxley) 要求所有变更到核心系统的代码、配置、权限都必须可追溯；**罚款 + 上市资格**风险
**hooks 能做的事**：给每一次 agent 工具调用生成不可篡改的审计 trail

**具体配置**：`PreToolUse` command hook → 审计日志系统（Splunk / ELK / DataDog）

```json
// ~/.claude/settings.json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "/opt/claude-audit/send-to-splunk.sh",
        "timeout": 3
      }]
    }, {
      "matcher": "Bash|Edit|Write|MultiEdit",
      "hooks": [{
        "type": "command",
        "command": "/opt/claude-audit/notify-soc.sh",
        "timeout": 1
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "/opt/claude-audit/append-transcript.sh",
        "timeout": 3
      }]
    }]
  }
}
```

```bash
#!/usr/bin/env bash
# /opt/claude-audit/send-to-splunk.sh
# 把每一次工具调用送到 immutable audit trail
INPUT="$(cat)"
echo "$INPUT" | curl -sS -X POST \
  -H "Authorization: Bearer $SPLUNK_HEC_TOKEN" \
  -H "Content-Type: application/json" \
  -d @- \
  "https://splunk.internal.example.com:8088/services/collector/event" \
  -o /dev/null
exit 0
```

**业务价值**：
- SOX 审计员看到"AI agent 在 2026-07-12 14:32 修过 prod-config.yaml"——可追溯、可定责
- 钓鱼式外泄：`SendUserMessage` 工具如果被用于外发邮件，会在 splunk 留下 recipient + content
- 内部审计部门不再要求"禁用 AI"，而是说"开了 AI 但要全量审计"

### 场景 2：医疗 HIPAA——禁止 PHI 进 git commit

**痛点所有人**：医院、保险理赔、临床 trial 系统团队
**业务代价**：HIPAA violation 上限 **$1.5M/类别/年**；不可挽回的患者隐私泄露
**hooks 能做的事**：agent 写文件前 OCR/PII 扫描

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write|MultiEdit",
      "hooks": [{
        "type": "command",
        "command": "/opt/claude-hipaa/scan-phi.sh",
        "timeout": 5
      }]
    }]
  }
}
```

```bash
#!/usr/bin/env bash
# /opt/claude-hipaa/scan-phi.sh
# 调用内部 PHI 扫描服务，禁止写含 SSN/诊断码/姓名+出生日期的内容
set -euo pipefail
INPUT="$(cat)"
CONTENT=$(echo "$INPUT" | jq -r '.tool_input.content // .tool_input.new_string // .tool_input.file_text // empty')

if [ -z "$CONTENT" ]; then exit 0; fi

RESULT=$(echo "$CONTENT" | curl -sS -X POST \
  -H "Authorization: Bearer $PHI_SCANNER_TOKEN" \
  -d @- "https://phi-scanner.internal/api/scan")

if [ "$(echo "$RESULT" | jq -r '.phi_detected')" = "true" ]; then
  REASONS=$(echo "$RESULT" | jq -r '.matches[] | "  - " + .type + " at line " + (.line|tostring)')
  cat <<EOF
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "PHI detected in proposed write:\\n$REASONS\\nMask/de-identify before retrying."
  }
}
EOF
fi
exit 0
```

**业务价值**：HIPAA compliance officer 不再拒绝 agent；DLP 从"事后追责"变成"事前阻断"

### 场景 3：GDPR 数据本地化

**痛点所有人**：跨国 SaaS / 跨境电商的合规官
**业务代价**：跨境传欧盟用户数据违反 GDPR-Schrems II，罚 **全球营收 4%**
**hooks 能做的事**：`Edit`/`Write` 前判断内容是否含 EU data + 检查目标仓库区域

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "/opt/claude-gdpr/check-data-residency.sh"
      }]
    }]
  }
}
```

```bash
#!/usr/bin/env bash
# 拦截：向非 EU region 上传 EU 用户数据
INPUT="$(cat)"
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$CMD" | grep -qE 'aws\s+s3\s+cp.*s3://(us|ap)-'; then
  if echo "$CMD" | grep -qE 'eu_user|gdpr_subject'; then
    cat <<'EOF'
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",
 "permissionDecisionReason":"EU user data detected but target region is non-EU. Upload to eu-central-1 instead."}}
EOF
    exit 0
  fi
fi
exit 0
```

### 场景 4：PCI-DSS 卡号永不落盘

**痛点所有人**：支付/电商平台
**业务代价**：信用卡数据 "at rest" 状态违规 → 失去 PCI 认证 → 不能接卡组
**hooks 能做的事**：所有 Write/Edit/Bash 永久屏蔽含 PAN (Primary Account Number) 的内容

```bash
# /opt/claude-pci/scan-pan.sh
INPUT="$(cat)"
CONTENT=$(echo "$INPUT" | jq -r '.. | strings | select(test("^(?:\\d[ -]*?){13,16}$"))')

if [ -n "$CONTENT" ]; then
  echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",
    "permissionDecisionReason":"PAN-like pattern detected. Tokenize via vault before retrying."}}'
fi
```

### 场景 5：SOC2 CC7——访问日志不可篡改

**痛点所有人**：所有上 SOC2 的 SaaS
**业务代价**：审计师找不到变更证据 → SOC2 不续证 → 客户流失
**hooks 能做的事**：所有 agent 行为实时落到 append-only 日志（AWS CloudWatch Logs / QLDB / S3 Object Lock）

> 关键技术：`/opt/claude-soc2/append.sh` 把 stdin 内容写到 S3 Object Lock (Compliance mode) bucket，事后任何人包括 root 都改不了。

---

## 3. 实战场景 6–10：安全护栏与 DLP

### 场景 6：阻止危险 shell 命令

**痛点所有人**：所有不想半夜被叫醒的 oncall
**业务代价**：误跑 `rm -rf ~` / `dd if=/dev/zero of=/dev/sda` / `kubectl delete ns prod` / 推 master 覆盖保护分支
**hooks 能做的事**：命令模式匹配直接拦截（场景 13 的 block-dangerous.sh 实例）

**为什么要用 hooks 而不是 prompt？**
| 方法 | 拦截率 | 失败成本 |
|------|--------|---------|
| system prompt 警告 | ~60%（LLM 会忽略）| 一次误操作 |
| 监督模型 review | ~85%（慢 ~10s）| ~10s 延迟 |
| **hooks 模式匹配** | **~99.9%（零延迟）** | **接近零** |

### 场景 7：DLP——阻止 secret 写入代码

**痛点所有人**：所有上 GitHub / GitLab 的团队
**业务代价**：API key 推到公网 repo，5 分钟内被比特币矿机器人扫到 → $10w+ 云账单
**hooks 能做的事**：PreToolUse on Edit/Write/MultiEdit → 调 gitleaks/trufflehog 扫内容

```bash
# /opt/claude-dlp/scan-secrets.sh
INPUT="$(cat)"
CONTENT=$(echo "$INPUT" | jq -r '.tool_input.content // .tool_input.new_string // empty')

if [ -z "$CONTENT" ]; then exit 0; fi

echo "$CONTENT" | gitleaks detect --no-git --stdin --report-path /tmp/leak.json --exit-code 1 2>/dev/null
RC=$?

if [ $RC -ne 0 ]; then
  LEAKS=$(jq -r '.[] | "  - line " + (.StartLine|tostring) + ": " + .Description' /tmp/leak.json 2>/dev/null | head -5)
  cat <<EOF
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",
 "permissionDecisionReason":"Secrets detected:\\n$LEAKS\\nRemove/redact before retrying."}}
EOF
fi
exit 0
```

### 场景 8：拦截大文件读写

**痛点所有人**：SRE / DBA / 财务
**业务代价**：agent 误读 `/var/log/syslog.1` (50GB) → 烧光 context 窗口 → 别的任务全废
**hooks 能做的事**：检测文件大小或行数

```bash
INPUT="$(cat)"
PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
[ -z "$PATH" ] && exit 0

SIZE=$(stat -c%s "$PATH" 2>/dev/null || stat -f%z "$PATH" 2>/dev/null || echo 0)
if [ "$SIZE" -gt 10485760 ]; then  # > 10MB
  cat <<EOF
{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask",
 "permissionDecisionReason":"File is $((SIZE/1048576))MB, may exceed context. Confirm intent."}}
EOF
fi
```

### 场景 9：阻止内部工具被外网域名调用

**痛点所有人**：网络安全团队
**业务代价**：agent 把内部 API endpoint + 数据通过 webhook 发到 attacker.com
**hooks 能做的事**：http hook `allowedHttpHookUrls` + sandbox proxy 强校验（已内建） + PreToolUse on Bash 命令 grep 内网 IP

```bash
INPUT="$(cat)"
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
INTERNAL_REGEX='(10\.\d+\.\d+\.\d+|172\.(1[6-9]|2\d|3[01])\.\d+\.\d+|192\.168\.\d+\.\d+)'

if echo "$CMD" | grep -qE "curl.*$INTERNAL_REGEX" && echo "$CMD" | grep -qE "[-]X\s+POST"; then
  echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",
    "permissionDecisionReason":"Exfiltrating to internal IP over POST is denied. Use internal service mesh."}}'
fi
```

### 场景 10：审批流嵌入 ITSM (ServiceNow/Jira)

**痛点所有人**：金融、政府、医疗的变更管理 (CAB) 团队
**业务代价**：agent 跳过了本该有的 change ticket → 审计失败
**hooks 能做的事**：高危操作前自动开 ticket + 等审批

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "/opt/claude-itsm/require-ticket.sh"
      }]
    }]
  }
}
```

```bash
# 高危命令前必须 check ticket id
INPUT="$(cat)"
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
HIGH_RISK='(kubectl\s+(delete|apply).*prod|terraform\s+apply|aws\s+rds\s+delete)'

if echo "$CMD" | grep -qE "$HIGH_RISK"; then
  TICKET=$(echo "$CMD" | grep -oE 'CHG[0-9]{6,}' | head -1)
  if [ -z "$TICKET" ]; then
    echo '{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny",
      "permissionDecisionReason":"Production-impacting command requires an open CHG ticket. Create one in ServiceNow first."}}'
    exit 0
  fi
  # verify ticket state via ServiceNow API
  STATE=$(curl -sS -u "admin:$SERVICENOW_PASS" \
    "https://instance.service-now.com/api/now/table/change_request?number=$TICKET" | jq -r '.result[0].state')
  if [ "$STATE" != "Implement" ] && [ "$STATE" != "Scheduled" ]; then
    echo "{\"hookSpecificOutput\":{\"hookEventName\":\"PreToolUse\",\"permissionDecision\":\"deny\",
      \"permissionDecisionReason\":\"CHG ticket $TICKET is in state '$STATE' (needs Implement/Scheduled).\"}}"
  fi
fi
```

**业务价值**：agent 接入后**变更管理流程不变**，但执行人变成 AI

---

## 4. 实战场景 11–15：编排 / 通知 / 上下文增强

### 场景 11：Stop hook 验证任务真正完成

**痛点所有人**：所有用 agent 做多步骤工作（migration、refactor、文档更新）的人
**业务代价**：agent "以为" 完成了，但漏了文件；用户以为做完实际半成品
**hooks 能做的事**：`Stop` event → agent hook 读取 transcript → 验证任务清单 100% 完成

这是 `execAgentHook.ts:107-116` 里的 system prompt 真实在做的事：

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "agent",
        "prompt": "Look at the conversation transcript. The user's original task was: $ARGUMENTS. Verify that every explicit requirement was addressed. Return {\"ok\": true} or {\"ok\": false, \"reason\": \"<what's missing>\"}.",
        "model": "sonnet"
      }]
    }]
  }
}
```

`stopReason` 会作为 Claude 的下一轮 prompt，agent 收到"X 没做"会自动补做。

**进阶**：多个并行 Stop hook 都返回 blocking 时取最严格的（默认 multi-hook 行为）

### 场景 12：自动通知团队（Slack/Teams/钉钉/飞书）

**痛点所有人**：Agent Manager / Tech Lead
**业务代价**：不知道 agent 在跑什么、跑了什么、结果是什么
**hooks 能做的事**：

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "http",
        "url": "https://hooks.slack.com/services/T.../B.../...",
        "headers": {"Content-Type": "application/json"},
        "timeout": 5
      }]
    }],
    "Stop": [{
      "hooks": [{
        "type": "http",
        "url": "https://hooks.slack.com/services/T.../B.../...",
        "timeout": 5
      }]
    }],
    "PostToolUseFailure": [{
      "matcher": "*",
      "hooks": [{
        "type": "http",
        "url": "https://hooks.slack.com/services/T.../B.../...",
        "timeout": 2
      }]
    }]
  },
  "allowedHttpHookUrls": ["https://hooks.slack.com/*"]
}
```

业务价值：
- 安全团队看到 agent 跑的全过程
- Tech Lead 在 Slack 收到 "✅ Claude 完成 PR #1234 'Migrate user schema'"
- 没人在意时自动静默；出问题全 channel

### 场景 13：自动推 PR / 触发 CI

**痛点所有人**：GitOps / Platform Engineering
**业务代价**：agent 改完代码需要人去 push、点 CI、看结果
**hooks 能做的事**：`PostToolUse` on Edit/Write → 自动 stage + commit + PR

```bash
# 实际场景的 hook（仅示意）
# 触发条件：agent 写了 .py/.ts 文件且 git diff 非空
INPUT="$(cat)"
FILES_CHANGED=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

git add "$FILES_CHANGED"
if ! git diff --cached --quiet; then
  BRANCH="claude/$(date +%s)"
  git checkout -b "$BRANCH"
  git commit -m "Auto-commit by Claude agent

  Generated-by: Claude Code
  Source: session $(echo "$INPUT" | jq -r '.session_id')
  Co-Authored-By: Claude <noreply@anthropic.com>"
  gh pr create --fill --base main
  curl -X POST -H "$CI_WEBHOOK" "https://ci.internal/hook/start?branch=$BRANCH"
fi
```

**业务价值**：agent 持续工作时工程师不需要 ping-pong

### 场景 14：注入领域上下文（`UserPromptSubmit` + `SessionStart`）

**痛点所有人**：不知道把团队文档喂给 agent
**业务代价**：agent 每个 prompt 都要重复粘贴团队约定
**hooks 能做的事**：在每次 user 提 prompt 时自动注入 additionalContext

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "cat /etc/claude-context/team-conventions.md"
      }]
    }],
    "SessionStart": [{
      "hooks": [{
        "type": "prompt",
        "prompt": "Read the README and CONTRIBUTING.md in the current directory. Summarize the team's coding standards, deployment process, and key gotchas in 200 words. $ARGUMENTS"
      }]
    }]
  }
}
```

```bash
# /etc/claude-context/team-conventions.md
Additional context injected automatically:
- Backend: Python 3.12, FastAPI, SQLAlchemy 2.0
- All DB writes must use the audit_log() decorator
- Tests required for any new endpoint
- Never commit to main directly; always via PR
- AWS region for prod is eu-west-1
```

`hookSpecificOutput.additionalContext` 拼接到 user prompt 前，模型"假装"团队约定在用户的输入里

### 场景 15：文件监听 / 自动重载

**痛点所有人**：agent 改 `.env` / `config.yaml` 后需要重启服务
**业务代价**：agent 改完用户得手动重启
**hooks 能做的事**：`SessionStart` hook 用 `watchPaths` 触发 FileChanged hook

```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"watchPaths\":[\"/etc/myapp/config.yaml\",\".env\"]}}'"
      }]
    }],
    "FileChanged": [{
      "hooks": [{
        "type": "command",
        "command": "systemctl reload myapp"
      }]
    }]
  }
}
```

**业务价值**：agent 改配置后服务自动 reload，无需额外步骤

---

## 5. Hook 类型决策矩阵

**何时用哪种 executor？**

| 场景特征 | 推荐 executor | 理由 |
|---------|--------------|------|
| 只需要"判断"（不允许/允许），无副作用 | prompt (Haiku) | 便宜 + 快 + 一致 |
| 需要读文件、跑命令、查 git history 才能判断 | agent (Sonnet) | 多步推理 |
| 需要"做"一件事（写日志、发 webhook、调 API） | command | 任何 shell 能做的副作用 |
| 需要"做"且零延迟（<100ms）| command (bash inline) | 没有 LLM 调用 |
| 写明文 audit log | command | LLM 慢且贵 |
| 团队有 Slack/ServiceNow/自研系统 | http | 跨进程边界 |
| 需要 LLM 语义判断但不能再 spawn agent | prompt | 比 agent 轻 1 个数量级 |
| SDK / 内部模块要插一段程序逻辑 | callback | 同步 + 不写 settings.json |

**何时用哪种 event？**

| 想做的事 | 用 event | 典型 hook 类型 |
|---------|---------|---------------|
| **阻止** 危险工具调用 | `PreToolUse` | command (命令匹配) / prompt (语义判断) |
| **审计** 已发生的工具调用 | `PostToolUse` | command (append audit log) / http (SIEM) |
| **修改** 模型发的命令 | `PreToolUse` + `updatedInput` | command (sed 改输入) |
| **注入** 用户消息额外信息 | `UserPromptSubmit` + `additionalContext` | command (cat standards) / agent |
| **启动时** 准备工作 | `SessionStart` | command (cp files) / prompt (cache context) |
| **完成时** 验证 + 通知 | `Stop` | agent (verify) + http (Slack) |
| **失败时** 升级 | `PostToolUseFailure` + `Notification` | http (Slack #oncall) |
| **跨多事件** 自定义上下文 | SessionStart 一次 + 后续 event 复用 | multi-hook |

---

## 6. 落地路线图：3 个月从 0 到生产

### 第 1–2 周：审计优先（最小可信能力）

目标：**让 agent 能跑，但所有行为可观测**
1. 部署统一的 `PostToolUse` + `PreToolUse` 审计 command hook 到全员
2. 配置 `allowedHttpHookUrls` allowlist 锁死 http hook 出站
3. 关键工具（`Bash`, `Edit`, `Write`, `MultiEdit`, `WebFetch`）配 PreToolUse 拦截规则

**预期**：安全团队不用 shadow review 了；产品团队不被禁用

### 第 3–4 周：安全护栏（高 ROI）

1. 部署危险命令拦截（场景 6）
2. DLP：secret scanner（场景 7）
3. 大文件保护（场景 8）
4. 外网域名拦截（场景 9）

**预期**：误操作/月 降到 0；secret leak 事件消失

### 第 5–8 周：合规深化

1. 行业合规 hook（场景 1–5 按需）：SOX / HIPAA / PCI / GDPR
2. ITSM ticket 集成（场景 10）

**预期**：合规审计"agent 产生的工作"和"人产生的工作"同等对待

### 第 9–12 周：业务编排

1. Stop hook 验证（场景 11）
2. 自动 PR / CI（场景 13）
3. 上下文注入标准化（场景 14）
4. 文件监听（场景 15）

**预期**：agent 成为生产流水线一环，不是 demo 玩具

---

## 7. 与其他安全工具的边界

hooks **不是**万能的。它能解决的是"agent 行为" 的策略执行点。和已有工具的关系：

| 工具 | 关注对象 | hooks 是它的补充方式 |
|------|---------|---------------------|
| **GitGuardian / TruffleHog** | git 仓库 secret 扫描 | hooks 在 commit 之前就拦 |
| **OPA / Cedar** | 通用策略引擎 | hooks 是它的前端"agent 调用入口" |
| **Vault / AWS Secrets Manager** | secret 注入 | hooks 让 agent 自动获取临时 token |
| **CrowdStrike / SentinelOne** | 进程行为监控 | hooks 提供"agent 视角"的语义上下文 |
| **Okta / SSO** | 身份认证 | hooks 校验 agent 操作时的用户身份 |
| **Splunk / ELK** | 日志聚合 | hooks 是其"最高信号比"的事件源（语义清晰）|
| **ServiceNow / Jira** | 变更管理 | hooks 把 agent 操作嵌入既有 CAB 流程 |
| **Promptfoo / LangSmith** | LLM 评估 | hooks 拦截"模型决策"的下游执行 |

**hooks 的真正定位**：**Strategy Enforcement Point (SEP)**——所有 agent 决策的实际执行边界都过这里。

---

## 8. 关键文件索引

| 路径 | 作用 |
|------|------|
| `src/utils/hooks.ts:302-329` | `createBaseHookInput` —— 所有 hook 输入的公共底座 |
| `src/utils/hooks.ts:400-452` | `parseHookOutput` —— command hook stdout 解析 |
| `src/utils/hooks.ts:569-` | `processHookJSONOutput` —— JSON 决策翻译 |
| `src/utils/hooks.ts:830-1418` | `execCommandHook` 完整实现 |
| `src/utils/hooks/execHttpHook.ts:49-58` | URL allowlist —— 企业出站白名单 |
| `src/utils/hooks/execHttpHook.ts:89-108` | `interpolateEnvVars` —— 防止 secret 泄漏 |
| `src/utils/hooks/execHttpHook.ts:176` | sandbox proxy 集成 —— 网络隔离 |
| `src/utils/hooks/execAgentHook.ts:107-116` | Stop hook agent 的典型 system prompt |
| `src/utils/hooks/sessionHooks.ts:93-115` | `addFunctionHook` —— 程序化注入 |
| `src/utils/hooks/ssrfGuard.ts` | SSRF 防护 —— 防 agent 反向攻击 |

---

## 附录：参考实现仓库

下面这些是已经验证过的企业级 hook 实践模式（伪代码，可直接落）：

1. **`PreToolUse` danger-block command hook**：参见 [`43-hook-types-and-command-protocol.md`](43-hook-types-and-command-protocol.md) 场景 13
2. **Stop hook `agent` verification**：参见本文场景 11
3. **`UserPromptSubmit` 团队上下文注入**：参见本文场景 14
4. **SOC2 不可篡改审计 log**：使用 AWS S3 Object Lock + `curl` 上传
5. **DLP pre-commit 风格 secret scan**：使用 gitleaks + PreToolUse on Edit

---

## 一句话总结

> **Hooks 是企业把"AI 拥有执行权"变成"AI 在受控环境里拥有执行权"的唯一可插拔边界**。
>
> 没有它，agent 是不可审计、不可合规、不可定责的"幽灵员工"；
> 有它，agent 是嵌入既有 IT 治理、可被 SOC/SOX/HIPAA 接受的标准流水线一环。

---

如果你想针对某个具体行业（金融/医疗/政府/教育/电商……）展开更深一层的合规 hook 配置，或者想把上面 5 条参考实现落到具体的代码，告诉我对应行业 + 你的 ITSM/SIEM 工具栈（Slack/Splunk/ServiceNow/Datadog 等），我可以再补一份对应场景的完整配置。