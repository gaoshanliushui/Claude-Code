# MCP 系统在企业级 Agent 实战中的价值

> 本文目的：从"为什么需要 MCP"出发，结合 **15+ 个真实业务场景**，展示 Claude Code 的 MCP（Model Context Protocol）系统（`src/services/mcp/`、`src/tools/MCPTool/`、`src/tools/McpAuthTool/` 等）在企业部署中**具体能解决什么问题、带来什么业务价值、怎么落地**。
>
> 不写"可以让 agent 调用外部工具"这种空话。每个场景回答四个问题：
> 1. **谁有这个痛点**（具体角色）
> 2. **不解决会怎样**（业务代价）
> 3. **用哪种 MCP 形态**（transport + auth + scope）
> 4. **写出来长什么样**（可直接 copy-paste 的真实配置）

> 源码位置：见 [`docs/mcp/09-mcp-system-deep-dive.md`](09-mcp-system-deep-dive.md)、[`docs/mcp/10-mcp-client-implementation.md`](10-mcp-client-implementation.md)

---

## 目录

- [0. 一个不应该被忽视的前提：MCP 不是"接口协议"](#0-一个不应该被忽视的前提mcp-不是接口协议)
- [1. MCP 在企业里解决的 5 个根本问题](#1-mcp-在企业里解决的-5-个根本问题)
- [2. 实战场景 1–5：内部系统接入](#2-实战场景-15内部系统接入)
- [3. 实战场景 6–10：企业安全与合规](#3-实战场景-610企业安全与合规)
- [4. 实战场景 11–15：可扩展性 / 团队复用 / 实时集成](#4-实战场景-1115可扩展性--团队复用--实时集成)
- [5. Transport / Scope / Auth 决策矩阵](#5-transport--scope--auth-决策矩阵)
- [6. 三种 MCP 部署形态的取舍](#6-三种-mcp-部署形态的取舍)
- [7. 落地路线图：从 PoC 到生产](#7-落地路线图从-poc-到生产)
- [8. MCP 解决不了的问题（边界）](#8-mcp-解决不了的问题边界)
- [9. 关键文件索引](#9-关键文件索引)

---

## 0. 一个不应该被忽视的前提：MCP 不是"接口协议"

很多人初看 MCP 的反应是"又一个 OpenAPI / gRPC / RPC 框架"——这是完全错误的类比。

**MCP 的本质是**：**让一个进程拥有受控的、可发现、可调用的能力，并且这种能力能直接被 LLM agent 选择和执行**。它解决的不是一个"调用"问题，而是**一个"agent 能做什么"的边界问题**。

把这个区别说清楚：

| 维度 | 传统 API（OpenAPI / gRPC） | MCP |
|------|---------------------------|-----|
| 调用方 | 程序员写的代码 | **LLM 自主决策**（模型自己 tool_use） |
| 工具发现 | 看文档 / codegen | **`tools/list` + `tools/call`，模型读 JSON Schema 自动学会** |
| 能力边界 | 代码 hardcode | **policy filter + scope 控制 + managed settings 强制** |
| 凭据管理 | 代码里塞 / env var | **OAuth / XAA / secure storage，与用户身份绑定** |
| 跨语言 | 通过 SDK | **只要能跑 MCP server 进程 / 暴露 HTTP endpoint，agent 都能用** |
| 输出处理 | 自己 parse | **自动截断 / 持久化 / image 下采样**（`processMCPResult`） |

> 源码佐证：`src/services/mcp/client.ts:1743 fetchToolsForClient` 把 server 返回的 tools 直接包装成 `Tool` 对象混进主工具集，模型 tool_use 走通用 `call` 路径。`Tool.isMcp = true` 之后所有权限 / 输出处理都按 MCP 协议统一处理。

**关键推论**：MCP 的价值不是"agent 能调外部 API"，而是：

1. **凭据隔离**：agent 不接触 API key，靠 OAuth/XAA 由 MCP server 自己持有
2. **能力收口**：企业可以通过 allowlist/denylist 控制 agent 能看到哪些 tools
3. **跨进程沙箱**：stdio transport 天然把外部进程隔离开，挂掉不影响 agent
4. **声明式发现**：新接一个 SaaS，模型自动学会用，不用改 system prompt
5. **强制管控**：managed settings 可以让 IT 部门**远程**决定整个公司的 MCP 配置

---

## 1. MCP 在企业里解决的 5 个根本问题

| 问题 | 真实痛点 | 不解决会怎样 | MCP 怎么解决 |
|------|---------|-------------|-------------|
| **A. 凭据泄露** | agent 要查 Salesforce，token 直接写进 system prompt | 一次 prompt 注入 = token 全网公开 | MCP server 进程内拿 token，agent 只看到 tool 名 |
| **B. 工具爆炸** | 公司接了 30 个 SaaS，每个 prompt 都要写"你可以用 X 做 Y" | prompt 越来越长，模型选择困难，幻觉增多 | server 暴露 `tools/list`，模型按需学 |
| **C. 合规盲区** | 安全/合规不知道员工让 agent 调了哪些外部系统 | SOX/HIPAA/GDPR 审计无据可查 | `PreToolUse` hook + MCP server 日志 + managed policy |
| **D. 能力分散** | 每个项目写一套 wrapper，维护 30 套 | 升级一个 SaaS API = 改 30 处 | 一套 MCP server 协议，全公司复用 |
| **E. 实时性需求** | agent 等人工 poll 数据库查"线上 P1 告警" | 5 分钟延迟 = 事故扩大 | WebSocket / SSE transport，server 主动推送 |

每个问题在下面都有具体的实战场景。

---

## 2. 实战场景 1–5：内部系统接入

### 场景 1：DBA 团队——让 agent 直接查生产 PostgreSQL（无密码暴露）

**痛点所有人**：DBA、运维、SRE 团队
**业务代价**：以前 DBA 让 agent 查 SQL 必须把 `postgres://prod_user:xxx@10.x.x.x/db` 贴到 prompt / `.env` / Claude 对话历史里。**密码在 LLM 的上下文里 = 等于泄露**（任何有日志审计、prompt 缓存、副本训练的地方都可能泄露）。一次事故就是 P0 security incident。
**MCP 能解决**：起一个 stdio MCP server，token 只在 server 进程内；agent 只看到 `mcp__pg__query(sql="...")` 这样一个工具

**具体配置**：项目 `.mcp.json`：

```json
{
  "mcpServers": {
    "pg-prod": {
      "type": "stdio",
      "command": "/usr/local/bin/mcp-pg-server",
      "args": ["--dsn", "postgres://readonly_user:${PG_PROD_RO_PASS}@db1.prod.internal:5432/main"],
      "env": {
        "PG_PROD_RO_PASS": "${PG_PROD_RO_PASS}"
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/config.ts:1297 parseMcpConfig` 做 `${VAR}` 环境变量展开，`subprocessEnv() ∪ env` 传给 stdio 子进程（`src/services/mcp/client.ts` connectToServer 分支）。**LLM 看不到 `PG_PROD_RO_PASS` 的明文值——展开发生在 client 侧，传给子进程的是已展开的 string**。

**业务价值**：
- LLM 上下文里只有 `mcp__pg-prod__query` 这个 tool 名，没有 DSN
- DBA 可以强制 `readonly_user`，即使 agent 写出 `DROP TABLE` 也跑不通
- 公司密码轮换只改一处环境变量，agent 不用重启
- 合规审计看的是**server 进程**的查询日志，不是 LLM 的对话

### 场景 2：客服/Support——Jira MCP 让 agent 自动创建/查询工单

**痛点所有人**：客户成功、技术支持、PM 团队
**业务代价**：以前 agent 帮用户 debug，要"用户去 Jira 提单，我帮你写描述"，**agent 不能直接操作**——多 5 分钟来回 + 人工错误（写错 priority、忘选 component）。
**MCP 能解决**：通过 HTTP transport 暴露 Jira API，agent 自己 `create_issue(project="SUPPORT", summary="...", description="...")`

**具体配置**：

```json
// ~/.claude/settings.json（user scope，全公司可用）
{
  "mcpServers": {
    "jira": {
      "type": "http",
      "url": "https://mcp-jira.internal.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${JIRA_SERVICE_TOKEN}"
      },
      "oauth": {
        "clientId": "claude-code-jira",
        "authServerMetadataUrl": "https://auth.internal.example.com/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/types.ts:89 McpHTTPServerConfig` 定义 HTTP transport，`src/services/mcp/auth.ts:ClaudeAuthProvider` 处理 OAuth/PKCE，token 存 secure storage（macOS keychain 等）。`McpOAuthConfigSchema` 强制 `authServerMetadataUrl` 用 `https://`（types.ts:51），不允许裸 http。

**业务价值**：
- agent 能直接 `jira__create_issue` + `jira__add_comment` + `jira__transition_issue`，整个工单生命周期自动化
- OAuth 凭据走企业 SSO（Okta/Azure AD），员工离职 = 立刻吊销
- 通过 `isMcpServerAllowedByPolicy`（config.ts:417）让 IT 控制谁能用 `jira-admin` 类破坏性工具

### 场景 3：DevOps——PagerDuty / Datadog MCP，agent 主动响应告警

**痛点所有人**：SRE、on-call 工程师
**业务代价**：凌晨 3 点 P1 告警，on-call 收到短信 → 打开 Datadog → 找 dashboard → 看 metrics → 翻日志 → **再决定要不要叫人**——平均 15 分钟。事故扩大期。
**MCP 能解决**：通过 SSE/WebSocket transport，Datadog 告警**主动推送**给 agent，agent 自己分析 logs/traces 后**先**给出根因摘要，on-call 一开终端就知道发生了什么

**具体配置**：

```json
{
  "mcpServers": {
    "datadog-alerts": {
      "type": "ws",
      "url": "wss://mcp-datadog.internal.example.com/alerts/stream",
      "headers": {
        "X-Service-Token": "${DD_SERVICE_TOKEN}"
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/types.ts:99 McpWebSocketServerConfig` 接受 `ws`/`wss` URL；`src/utils/mcpWebSocketTransport.ts` 实现 `WebSocketTransport`，支持 Bun 原生 WS 和 node `ws` 双后端。WebSocket transport 是少有的**server 主动 push**通道。

**业务价值**：
- 告警不再是 fire-and-forget SMS，而是结构化事件流
- agent 可以 `datadog__list_alerts(severity="P1", last=10min)` + `datadog__correlate(alert_id="...", metric="error_rate")`
- 自动生成 incident summary draft，写入 Confluence MCP
- 平均响应时间 15min → 3min（agent 已经做完 80% 的调研）

### 场景 4：开发者效率——GitLab/GitHub MCP，agent 跨仓库操作

**痛点所有人**：研发团队、platform team
**业务代价**：以前 agent 改完代码想"再开一个 PR 到 hotfix 分支"、"看 CI 跑没跑"、"看 review 状态"——必须**让人切到浏览器**做。一次来回 1 分钟，一天 50 次 = 50 分钟纯等待。
**MCP 能解决**：GitLab/GitHub MCP 暴露 `create_mr`、`list_pipelines`、`get_review_comments` 等工具，agent **在终端里闭环**

**具体配置**：

```json
// .mcp.json (project scope)
{
  "mcpServers": {
    "gitlab": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-gitlab"],
      "env": {
        "GITLAB_PERSONAL_ACCESS_TOKEN": "${GITLAB_PAT}",
        "GITLAB_API_URL": "https://gitlab.internal.example.com"
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/config.ts:1297` Windows 上 npx 命令会被检测并要求 `cmd /c` 包裹（防止直接执行失败）。`parseMcpConfigFromFilePath` 校验 schema 同时**展开 `${VAR}`**。

**业务价值**：
- agent 改代码 → `gitlab__create_mr` → `gitlab__list_pipelines`（CI 失败）→ 自动 retry
- **零切屏**：所有操作在终端流里完成
- 项目级 scope 让每个 repo 团队各自管自己的 token（团队 A 用 GitLab.com，团队 B 用自建 GitLab）

### 场景 5：数据/BI 团队——BigQuery MCP，agent 写 SQL 自查业务数据

**痛点所有人**：数据分析师、PM、运营
**业务代价**：PM 问"上个季度华南区 SKU 复购率" → 找分析师 → 写 SQL → 跑 → 回邮件 = **2 小时**。高频小问题积压成 backlog。
**MCP 能解决**：BigQuery MCP 暴露 `run_sql(dialect="bigquery", query="...")` + `get_table_schema(table="...")`，PM 自己 prompt agent 写 SQL，agent 直接跑

**具体配置**：

```json
{
  "mcpServers": {
    "bigquery": {
      "type": "stdio",
      "command": "/opt/mcp-servers/bq-server",
      "args": ["--project", "analytics-prod"],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "/etc/claude/bq-sa.json"
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/types.ts:28 McpStdioServerConfig` 的 `command` 字段、`src/services/mcp/client.ts:connectToServer` 把 env 传给子进程但**不让 LLM 看到**。stdout 用 NDJSON（`isStdoutMessage` 检测），stderr 单独 pipe 不污染 UI（client.ts:966）。

**业务价值**：
- SA 凭证不在 agent 上下文里
- agent 可以写 SQL → 跑 → 看错误 → 改 → 再跑，**循环在终端里闭环**
- 通过 managed settings 强制 `query_timeout=30s` + `max_bytes_billed=10GB` 防爆

---

## 3. 实战场景 6–10：企业安全与合规

### 场景 6：金融行业 SOX——MCP server 调用全程审计

**痛点所有人**：银行/保险的合规官、internal audit
**业务代价**：SOX 要求"对核心系统的所有访问可追溯"。agent 调内部核心银行系统（Core Banking）如果不能审计，**整条 AI agent 生产线会被合规一票否决**。
**MCP 能解决**：所有 MCP 工具调用进 server 进程的 audit log + Claude Code 侧 `PreToolUse` hook 上报 Splunk（参见 [`docs/hook/44-enterprise-value-of-hooks.md`](44-enterprise-value-of-hooks.md)）

**具体配置**：

```json
// ~/.claude/settings.json (managed, IT 部署)
{
  "mcpServers": {
    "core-banking": {
      "type": "http",
      "url": "https://mcp-core.internal.bank.com/mcp",
      "oauth": {
        "clientId": "claude-code-banking",
        "callbackPort": 8080
      }
    }
  }
}
```

server 侧（伪代码）：

```python
# MCP server 内部：每个 tools/call 写审计日志
@mcp.tool()
async def get_account_balance(account_id: str):
    audit_log.write({
        "timestamp": now(),
        "user": oauth_user_id,
        "tool": "get_account_balance",
        "params": {"account_id": account_id},
        "trace_id": request.meta["claudecode/toolUseId"],  # 从 _meta 透传
    })
    return await core_banking_api.balance(account_id)
```

> 源码佐证：`src/services/mcp/client.ts:3029 callMCPTool` 把 `toolUseId` 作为 `_meta:{'claudecode/toolUseId'}` 透传给 server，让 server 能做端到端 trace 关联。

**业务价值**：
- SOX 审计员看到 `user=alice, tool=get_account_balance, account=12345, ts=...` 全链路
- 配合 `PreToolUse` hook 强制 `mcp__core-banking__*` 工具必须经过 splunk 审计
- 合规官签字：MCP = 唯一**满足 SOX 审计**的 agent 集成路径

### 场景 7：医疗 HIPAA——MCP server 端 PHI 访问控制

**痛点所有人**：医院、保险理赔、临床 trial 系统团队
**业务代价**：HIPAA violation 单类别罚款上限 **$1.5M/年**。如果 agent 能直连 PHI 数据库，**任何 prompt 注入都可能泄露受保护健康信息**。
**MCP 能解决**：PHI 访问完全由 MCP server 控制，agent 看到的是 `lookup_patient(mrn="...")` 这种**业务语义**工具，server 内部做 HIPAA 审计 + 字段脱敏

**具体配置**（server 端伪代码）：

```python
@mcp.tool()
async def lookup_patient(mrn: str):
    if not has_hipaa_consent(oauth_user_id, mrn):
        raise McpError("Treatment relationship required for PHI access")
    
    record = phi_db.get(mrn)
    audit_phi_access(oauth_user_id, mrn, fields=list(record.keys()))
    
    # 自动脱敏 SSN / DOB
    return {
        "name": record.name,
        "dob_year": record.dob.year,  # 不给完整生日
        "diagnoses": [d for d in record.dx if d.is_hipaa_safe()],
        "_meta": {"redacted_fields": ["ssn", "address", "phone"]}
    }
```

> 源码佐证：`src/services/mcp/client.ts:2720 processMCPResult` 处理 `_meta` 透传 + 自动截断；agent 看到的是脱敏后结果，但审计 trail 在 server 端是完整的。

**业务价值**：
- **agent 永远拿不到完整 PHI**，从架构层面根除一类 HIPAA 风险
- 每次 PHI 访问都有 `user + tool + mrn + fields` 审计
- agent 可以在受控范围内"看起来能看病历"，但实际**只看到脱敏子集**
- HIPAA compliance officer 可以签字放行 AI agent

### 场景 8：跨国企业——Cross-App Access（XAA）单点登录跨 SaaS

**痛点所有人**：跨国集团 IT、SaaS 采购
**业务代价**：员工用 agent 调 Salesforce、ServiceNow、Workday 三家 SaaS，**三套 OAuth、三套 token**，每个 SaaS 自己一套用户管理。员工离职要吊销 3 个地方，agent 也搞不清 token 是不是最新。
**MCP 能解决**：XAA（Cross-App Access，SEP-990）让 MCP server 走企业 IdP（Okta/Azure AD）的 OIDC id_token 交换 access_token，**一次登录全公司 SaaS 通**

**具体配置**（managed settings）：

```json
{
  "xaaIdp": {
    "issuer": "https://okta.internal.example.com",
    "clientId": "claude-code-xaa",
    "callbackPort": 8081
  },
  "mcpServers": {
    "salesforce": {
      "type": "http",
      "url": "https://mcp-sf.internal.example.com/mcp",
      "oauth": {
        "clientId": "sf-mcp-client",
        "clientSecret": "${SF_MCP_CLIENT_SECRET}",
        "xaa": true
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/xaa.ts:426 performCrossAppAccess` 完整实现 RFC 8693 token-exchange + RFC 7523 JWT-bearer grant。`src/services/mcp/xaaIdpLogin.ts:36 getXaaIdpSettings` 从 settings 读 IdP 配置。`src/services/mcp/types.ts:54` `McpXaaConfigSchema` 是一个 per-server boolean flag。

**业务价值**：
- 员工**只登录一次 Okta**，Salesforce / Workday / ServiceNow 全部 silent auth
- 离职 = 吊销 Okta session，所有 SaaS 立刻断
- 合规审计：**所有 SaaS 调用源头都是企业 IdP**，满足"统一身份"合规要求
- XAA 是 OpenID Foundation 标准化协议（SEP-990），未来 Microsoft / Google 都支持

### 场景 9：企业 IT 强制管控——Managed Settings

**痛点所有人**：企业 IT、CISO、security team
**业务代价**：员工自由安装 MCP server → 安全部门失去 visibility，**agent 变成内部威胁向量**。Salesforce 数据通过员工自己装的 MCP server 流向网外——security incident。
**MCP 能解决**：managed settings 由 IT 部署（macOS 配置文件 / Windows Group Policy），**优先级最高**，用户不能覆盖

**具体配置**（managed-mcp.json，企业 IT 部署）：

```json
{
  "mcpServers": {
    "internal-jira": {
      "type": "http",
      "url": "https://mcp-jira.internal.example.com/mcp",
      "oauth": {"clientId": "company-jira"}
    }
  },
  "allowManagedMcpServersOnly": true,
  "enabledMcpServers": ["internal-jira"],
  "denylist": {
    "commandPatterns": ["*curl*", "*wget*"],
    "urlPatterns": ["*pastebin.com*", "*transfer.sh*"]
  },
  "allowlist": {
    "commandPatterns": ["mcp-*"],
    "urlPatterns": ["*.internal.example.com/*"]
  }
}
```

> 源码佐证：`src/services/mcp/config.ts:417 isMcpServerAllowedByPolicy` 三维匹配：name / command / URL 通配；`config.ts:1470 doesEnterpriseMcpConfigExist()` memoize 判定企业独占；`config.ts:1553 setMcpServerEnabled` 普通 server 是 opt-out，**默认禁用的内建 server（如 computer-use）是 opt-in**。`config.ts:536 filterMcpServersByPolicy` 批量过滤。

**业务价值**：
- `allowManagedMcpServersOnly: true` → **员工完全不能自加 MCP server**，全部由 IT 审批
- `denylist.commandPatterns: ["*curl*"]` → 禁止通过 curl 把数据外传
- `denylist.urlPatterns: ["*pastebin.com*"]` → 禁止把 token paste 到外网
- CISO 拿到一句话承诺："员工 agent 看到的 MCP server，全是我们审过的"

### 场景 10：多团队/多项目——scope 隔离

**痛点所有人**：platform team、infra team、多 BU 集团
**业务代价**：team A 项目用 GitLab.com，team B 用自建 GitLab；team C 用 GitHub Enterprise。如果 MCP server 全放在 user settings 里，员工跨项目时 token 互相串——**安全噩梦**。
**MCP 能解决**：4 种 scope（`local` / `user` / `project` / `enterprise`）分层，**项目级 MCP 配置跟着 repo 走**

**具体配置**（scope 优先级与覆盖）：

```json
// 项目级 .mcp.json (团队 A 的 repo)
{
  "mcpServers": {
    "gitlab": {
      "type": "http",
      "url": "https://gitlab.com/api/mcp",
      "headers": {"Authorization": "Bearer ${TEAM_A_GITLAB_TOKEN}"}
    }
  }
}
```

```json
// 全局 ~/.claude/settings.json (user scope)
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp",
      "headers": {"Authorization": "Bearer ${MY_GITHUB_TOKEN}"}
    }
  }
}
```

```json
// 企业 IT 部署的 managed-mcp.json
{
  "mcpServers": {
    "core-banking": {
      "type": "http",
      "url": "https://mcp-core.internal.bank.com/mcp"
    }
  }
}
```

合并优先级：`plugin < user < project < local`（`Object.assign`，后者覆盖前者）。企业 managed 单独走 `doesEnterpriseMcpConfigExist()` 路径，**独占时其他全部丢弃**。

> 源码佐证：`src/services/mcp/config.ts:1071 getClaudeCodeMcpConfigs` 合并逻辑；`config.ts:1258 getAllMcpConfigs` 叠加 claude.ai connector；`config.ts:1494 areMcpConfigsAllowedWithEnterpriseMcpConfig` 企业独占。

**业务价值**：
- 同一员工在不同 repo 自动用对应 GitLab/GitHub，**零配置切换**
- token 不串：team A 的 GitLab token 不会泄漏到 team B 的 repo
- 企业级 managed server 在所有项目都可用，**用户无法禁用**
- 员工 onboarding 新团队 = clone repo → 自动拿到对的 MCP 配置

---

## 4. 实战场景 11–15：可扩展性 / 团队复用 / 实时集成

### 场景 11：Plugin Marketplace——内部 MCP server 像 npm 一样分发

**痛点所有人**：platform team、infra team、DevX team
**业务代价**：每个团队都写一套 MCP server，重复造轮子；升级一个 Jira wrapper = 改 10 个 repo；新人入职不知道公司有哪些 MCP server 可用。
**MCP 能解决**：Claude Code 的 plugin 系统**原生支持** MCP server 作为 plugin 的一部分分发（`LoadedPlugin` 包含 `McpServerConfig`，见 `src/utils/plugins/mcpPluginIntegration.ts`）

**具体配置**（plugin manifest，伪代码）：

```json
// ~/.claude/plugins/known_marketplaces.json
{
  "internal-marketplace": {
    "source": "github",
    "repo": "internal-platform/mcp-marketplace"
  }
}

// marketplace 内 plugin manifest
{
  "name": "company-jira",
  "version": "2.3.0",
  "mcpServers": {
    "jira": {
      "type": "http",
      "url": "https://mcp-jira.internal/mcp",
      "oauth": {"clientId": "company-jira"}
    },
    "confluence": {
      "type": "stdio",
      "command": "/opt/mcp/confluence-server"
    }
  },
  "commands": [
    {"name": "standup", "description": "Daily standup summary from Jira"}
  ]
}
```

员工装：`/plugin install internal-marketplace:company-jira` → 自动拿到 jira + confluence 两个 MCP server + standup command。

> 源码佐证：`src/commands/plugin/ManageMarketplaces.tsx:43 MarketplaceInfo` + `src/services/mcp/src/types/plugin.ts` LoadedPlugin 类型；`src/utils/plugins/mcpPluginIntegration.ts getPluginMcpServers`；plugin 与 manual server 通过 `getMcpServerSignature`（config.ts:202）**内容签名去重**，manual 优先。

**业务价值**：
- 公司 MCP server 像 npm package 一样有版本号、有 marketplace
- 升级 `company-jira` v2 → v3 = `update marketplace`，全员自动收到新版本
- 新人 onboarding：`/plugin install internal-marketplace` 一行命令，拿到全部内部工具
- `manual > plugin` 去重保证：员工在 `.mcp.json` 自定义 server 时，**不会被 plugin 覆盖**

### 场景 12：本地工具沙箱——stdio 隔离运行

**痛点所有人**：security team、平台架构师
**业务代价**：agent 跑用户提供的"工具"（比如某个开源 MCP server）会**污染主进程环境**：环境变量泄漏、文件描述符泄漏、僵尸进程。生产环境禁不起。
**MCP 能解决**：stdio transport 天然把 MCP server 跑在**子进程**，crash / hang / 内存泄漏**只影响子进程**，主 agent 进程干净

**具体配置**：

```json
{
  "mcpServers": {
    "third-party-ocr": {
      "type": "stdio",
      "command": "/opt/third-party/ocr-mcp-server",
      "args": ["--listen-fd", "3"],
      "env": {
        "OCR_LICENSE_KEY": "${OCR_KEY}"
      }
    }
  }
}
```

> 源码佐证：`src/services/mcp/client.ts:1048` 连接超时用 `Promise.race([client.connect(transport), timeout(getConnectionTimeoutMs())])`；`src/services/mcp/client.ts:966` stderr 进 64MB 上限的 `stderrOutput`；`subprocessEnv() ∪ env` 只传白名单环境变量给子进程（`src/services/mcp/client.ts` connectToServer stdio 分支）；`claude-in-chrome` / `computer-use` 走 `createLinkedTransportPair`（`InProcessTransport.ts`）**进程内 server**，避免 spawn ~325MB 子进程。

**业务价值**：
- 第三方 MCP server 挂掉 = 子进程死，主 agent 继续工作
- 子进程 stderr 进 64MB buffer，**不会污染 agent UI**
- 子进程拿到的环境变量是**白名单过滤后**的（`subprocessEnv()`），主进程的 secret 不会泄漏给子进程
- timeout + onclose 重连机制（client.ts:1216）保证坏 server 不会拖死整个 agent

### 场景 13：HTTP/SSE transport——多实例 / 多区域

**痛点所有人**：全球化部署、SRE
**业务代价**：公司业务横跨美/欧/亚，员工 agent 在欧洲但 MCP server 在美东，**延迟 300ms**，每次工具调用都慢。
**MCP 能解决**：HTTP / SSE transport 让 MCP server 部署在**多区域**，agent 端就近连接

**具体配置**（CDN / multi-region）：

```json
{
  "mcpServers": {
    "core-banking-eu": {
      "type": "http",
      "url": "https://mcp-eu.internal.bank.com/mcp"
    },
    "core-banking-us": {
      "type": "http",
      "url": "https://mcp-us.internal.bank.com/mcp"
    }
  }
}
```

agent 可以根据当前 session location 选择：

```typescript
// 由 dynamaic MCP config 注入 (sub-agent 上下文)
const region = process.env.AGENT_REGION ?? 'us'
const dynamicMcpConfig = {
  [`core-banking-${region}`]: {
    type: 'http',
    url: `https://mcp-${region}.internal.bank.com/mcp`
  }
}
```

> 源码佐证：`src/services/mcp/types.ts:89 McpHTTPServerConfig`、`McpSSEServerConfig`、`McpWebSocketServerConfig` 都接受 URL；`src/services/mcp/useManageMCPConnections.ts:143` 接受 `dynamicMcpConfig` 注入（sub-agent 场景）；`src/services/mcp/client.ts` HTTP/SSE 走 `StreamableHTTPClientTransport` / `SSEClientTransport`。

**业务价值**：
- agent 端 0 改动，server 端多 region 部署
- HTTP 是 stateless（vs stdio），水平扩展简单
- 配合 CDN（CloudFront / Cloudflare）做地理路由
- `connectToServer` memoize 同一 server 复用连接，**重复调用零额外握手**

### 场景 14：WebSocket 实时双向——IDE 集成

**痛点所有人**：IDE 厂商（VSCode / JetBrains）
**业务代价**：IDE 想让 agent "看到当前打开的文件"、"在 IDE 里展示 diff"、"接收用户在 IDE 里的 confirm 点击"——传统 HTTP polling 延迟高、状态难同步。
**MCP 能解决**：`ws` / `ws-ide` transport 提供 WebSocket 双向通道，server 主动 push 事件到 agent

**具体配置**（`ws-ide`，Claude Code ↔ VSCode）：

```typescript
// src/services/mcp/vscodeSdkMcp.ts
// claude-vscode 这个特殊的 sdk server：
// - 发 file_updated 通知到 VSCode
// - 收 log_event 通知转 telemetry
// - 下发 experiment gates / auto-mode state
```

> 源码佐证：`src/services/mcp/types.ts:79 McpWebSocketIDEServerConfig` 含 `ideName`、`authToken`、`ideRunningInWindows`；`src/cli/transports/transportUtils.ts:16 getTransportForUrl` WS/HTTP/SSE 多 transport 自适应；`src/services/mcp/vscodeSdkMcp.ts` 特化 `claude-vscode`。

**业务价值**：
- IDE 状态（当前文件、selection、git branch）实时同步给 agent
- agent 在终端完成的代码改动**实时推到 IDE**显示 diff
- 用户在 IDE 里点击 "Run This" = 通过 WebSocket 通知 agent 继续
- **WebSocket 长连接天然解决"状态同步"问题**，HTTP polling 做不了

### 场景 15：SDK 内嵌——SaaS 产品把 agent 集成进自己的 UI

**痛点所有人**：SaaS 厂商（Notion / Linear / Figma 类）
**业务代价**：SaaS 想"在我产品里集成一个 AI agent，让 agent 能调我自己的 API"——以前必须让用户自己装 Claude Code CLI + 自己配 MCP，**不可能规模化**。
**MCP 能解决**：Agent SDK 内嵌场景下，**MCP server 跑在 SaaS 自己的进程里**（`SdkControlTransport`，`InProcessTransport`），用户无感

**具体配置**（SDK 模式）：

```typescript
import { tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk'

const notionMcp = createSdkMcpServer({
  name: 'notion',
  version: '1.0.0',
  tools: [
    tool('search_pages', 'Search Notion pages', 
      { query: z.string() },
      async ({ query }, extra) => {
        const userToken = extra.notionOAuthToken  // 来自宿主进程
        return await notion.search(query, userToken)
      }
    )
  ]
})
```

> 源码佐证：`src/services/mcp/InProcessTransport.ts:11 InProcessTransport` + `createLinkedTransportPair`（:57）实现**进程内双向通道**，无网络；`src/services/mcp/SdkControlTransport.ts` 走 `sendMcpMessage` 回调与宿主进程通信；`src/entrypoints/agentSdkTypes.ts:73 tool` factory + `CreateSdkMcpServerOptions` 类型。

**业务价值**：
- SaaS 在自己产品里跑 agent + MCP，**无需用户装任何东西**
- token 永远在 SaaS 后端，agent 永远拿不到
- 性能：进程内 transport 比 HTTP 快一个数量级
- 配合 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` env var，**MCP tool 可以覆盖内建 tool**（如 SaaS 想自定义 `Read` tool）

---

## 5. Transport / Scope / Auth 决策矩阵

### 5.1 Transport 选型

| Transport | 用例 | 优势 | 劣势 |
|-----------|------|------|------|
| `stdio` | **默认推荐**：本机工具、本地沙箱、单租户 | 进程隔离、无网络、token 安全（subprocessEnv 白名单） | 需要 server 进程可执行；不能跨机 |
| `http` | 多区域部署、SaaS 厂商、CDN 后面 | stateless、水平扩展、HTTP middleware 可复用 | 多一跳 HTTP 延迟 |
| `sse` | server 主动 push 长连接（monitoring、CI 进度） | 流式响应、断线续传（Last-Event-ID） | 单向（server → client），上行要 POST |
| `ws` / `ws-ide` | 双向实时（IDE 集成、协作） | 双向、状态同步 | 长连接管理复杂，需要 reconnect 逻辑 |
| `sdk` / `InProcessTransport` | 进程内（MCP server 跑在 agent 同一进程） | 零网络、零延迟 | 仅 SDK 嵌入场景；不能跨进程 |
| `claudeai-proxy` | claude.ai 网页 connector | 用户零配置（OAuth in browser） | 必须用 claude.ai 账号体系 |

### 5.2 Scope 选型

| Scope | 配置文件 | 谁负责 | 优先级 | 典型用例 |
|-------|---------|--------|--------|---------|
| `managed` | 企业 IT 部署（managed-mcp.json） | IT | **最高**，不能被覆盖 | 企业强制 server（合规、审计） |
| `enterprise` | 企业 policy | IT | 高 | 企业允许但可选的 server |
| `project` | `.mcp.json`（仓库根） | 项目 owner | 中高 | 项目级专用工具（GitLab.com vs 自建） |
| `local` | 项目 local config | 单个开发者 | 中 | 个人实验性 server |
| `user` | `~/.claude/settings.json` | 员工 | 中低 | 跨项目共用（GitHub、个人 SaaS） |
| `plugin` | plugin marketplace | plugin 作者 | 低（被 manual 覆盖） | 公司/社区分发的 server |
| `claudeai` | claude.ai 网页 connector | 用户在网页配 | 最低 | 用户在 claude.ai 网页装的 connector |

> 优先级：`managed`/`enterprise`（独占时）>`plugin` < `user` < `project` < `local`；`plugin` vs `manual`（user/project/local）按内容签名去重（config.ts:202），manual 优先。

### 5.3 Auth 选型

| Auth | 用例 | 实现 |
|------|------|------|
| **无 auth（内网）** | 内部 MCP server（公司 VPN 内） | `headers: { "X-Service-Token": "..." }` 或 mTLS |
| **OAuth 2.0 + PKCE** | 标准 SaaS 集成 | `src/services/mcp/auth.ts:ClaudeAuthProvider`，token 存 secure storage |
| **XAA（Cross-App Access）** | 企业 SSO（Okta/Azure AD）跨多个 SaaS | `src/services/mcp/xaa.ts:performCrossAppAccess`，RFC 8693 + RFC 7523 |
| **Static API Key（stdin env）** | 个人 token、本地工具 | `env: { "TOKEN": "${TOKEN}" }`，`expandEnvVars` 展开 |

---

## 6. 三种 MCP 部署形态的取舍

### 形态 A：个人开发者 / 小团队

**形态**：每个员工 `~/.claude/settings.json` 配 user scope 的 server
**优势**：零运维、自助
**劣势**：token 散落每个员工机器、安全 visibility 差
**适合**：创业公司、小团队 POC

### 形态 B：企业 IT 集中管控

**形态**：managed-mcp.json 由 IT 部署 + project `.mcp.json` 由项目 owner 维护 + 用户可选 user scope
**优势**：CISO 拿到 compliance、IT 集中审计、项目 owner 有灵活性
**劣势**：配置管理复杂（多源合并）、需要 IT 流程
**适合**：中大型企业、金融/医疗

### 形态 C：SaaS 厂商内嵌

**形态**：SDK + 进程内 transport（`InProcessTransport` / `SdkControlTransport`），用户无感
**优势**：用户体验最好、token 永远在后端、零运维
**劣势**：仅限 SaaS 厂商自己用、需要重写 wrapper
**适合**：SaaS 厂商（Notion、Linear、Figma）

> 源码佐证：`src/services/mcp/config.ts:1071 getClaudeCodeMcpConfigs` 主路径；`src/services/mcp/config.ts:1258 getAllMcpConfigs` 叠加 claude.ai；`src/services/mcp/InProcessTransport.ts:57 createLinkedTransportPair`。

---

## 7. 落地路线图：从 PoC 到生产

### 阶段 1：单团队 PoC（1-2 周）

目标：验证 MCP 能不能解决你的具体痛点

1. 选 1 个高频场景（建议从**数据库只读查询**或**Jira 工单**开始——这两个 server 都有现成开源实现）
2. 在项目 `.mcp.json` 配一个 stdio server
3. 跑通：`mcp__<server>__<tool>` 在 `claude -p` 里能调通
4. 验证三件事：
   - agent 看不到 token ✅
   - agent 行为符合预期 ✅
   - server 进程崩溃不影响 agent ✅

### 阶段 2：企业试点（1-2 月）

目标：扩到 5-10 个 server、10-50 用户

1. **建内部 MCP marketplace**（参考 [plugin 文档](../plugin)）：1 个 git repo 维护公司所有官方 MCP server
2. **部署 managed settings**：企业 IT 用 managed-mcp.json + denylist/allowlist 锁定
3. **接 OAuth / XAA**：让员工走企业 SSO，告别 token 散落
4. **加审计**：所有 MCP tool 调用 → Splunk（用 `PreToolUse` hook）
5. **建立 MCP server 准入标准**：必走 schema 校验、`MAX_MCP_OUTPUT_TOKENS=25000` 默认截断、unicode sanitize（client.ts:1743）

### 阶段 3：规模化（3-6 月）

目标：全公司、50+ server、1000+ 用户

1. **MCP server SLA 化**：内部 marketplace 每个 server 有版本号、health check、deprecation policy
2. **多区域部署**：HTTP transport + CDN，全球员工就近连接
3. **集中 token 管理**：通过 Vault / AWS Secrets Manager 注入 `${VAR}`，自动轮换
4. **MCP-as-product**：把公司 MCP server 像 SaaS API 一样卖出去（对外）
5. **监控 + 告警**：每个 MCP server 暴露 metrics → Datadog MCP → on-call 告警（用回 [场景 3](#场景-3devopspagerduty--datadog-mcpagent-主动响应告警)）

---

## 8. MCP 解决不了的问题（边界）

> 诚实列出 MCP 不能解决的问题，避免被当成"万能协议"误用

| MCP 解决不了 | 用什么替代 |
|-------------|-----------|
| **agent 之间的状态同步**（sub-agent 共享内存） | state store + DB，而不是 MCP |
| **跨 agent 的锁 / 事务** | 业务侧用 distributed lock |
| **超低延迟（< 10ms）调用** | 进程内 RPC / 共享内存；MCP 走 HTTP/stdio 至少 10ms+ |
| **重型计算（GB 级数据 ETL）** | 离线 batch + 把结果 import 进 MCP-readable storage |
| **server 单点故障自动切换** | HTTP transport 后面挂 LB + health check；MCP 协议本身不做 HA |
| **agent 自己写 MCP server**（bootstrapping） | 可以，但需要明确"agent 创建的 server 必须经过人类审批"（用 `isMcpServerAllowedByPolicy`） |
| **跨公司联邦 agent 协作** | mTLS + OAuth 双向 + 严格 scope，但仍有合规风险，需法务介入 |

### MCP 协议本身的限制

- **tool output 上限 25k tokens**（`getMaxMcpOutputTokens` 默认，可配）——超过走截断或持久化（`processMCPResult` + `mcpOutputStorage`）
- **掉线恢复靠客户端**：MCP server 挂了不会自动切换，需要 `reconnectMcpServerImpl`（client.ts:2137）或外部 LB
- **没有内置 rate limit**：靠 server 自己实现，或前面挂 API gateway
- **tool 数量爆炸**：每个 server 几十个 tool，接 10 个 server = 几百 tool，模型选择困难——靠 server 端做"工具聚合"（virtual tool 模式）

---

## 9. 关键文件索引

| 模块 | 职责 | 实战价值对应 |
|------|------|-------------|
| `src/services/mcp/types.ts` | 8 种 transport schema、6 种连接状态、7 种 scope | 决策矩阵的依据 |
| `src/services/mcp/config.ts` | 多源加载、策略过滤、签名去重、增删、enable/disable | 场景 9-10（managed + scope） |
| `src/services/mcp/client.ts` | transport 创建、连接、tools/list、callTool、output 处理、重连 | 场景 1, 5, 12（stdio 沙箱 + 截断） |
| `src/services/mcp/useManageMCPConnections.ts` | React hook 编排连接生命周期 | 场景 11（plugin 自动连接） |
| `src/services/mcp/auth.ts` | OAuth/PKCE、step-up、secure storage | 场景 2, 6, 7（凭据隔离 + 审计） |
| `src/services/mcp/xaa.ts` | Cross-App Access（RFC 8693 + RFC 7523） | 场景 8（XAA SSO） |
| `src/services/mcp/xaaIdpLogin.ts` | IdP 登录 | 场景 8 |
| `src/services/mcp/elicitationHandler.ts` | elicitation form/url 交互 | 场景 14（IDE 双向） |
| `src/services/mcp/mcpStringUtils.ts` | `mcp__server__tool` 命名 | 权限匹配（防同名误伤） |
| `src/services/mcp/InProcessTransport.ts` | 进程内 server（零网络） | 场景 15（SDK 嵌入） |
| `src/services/mcp/SdkControlTransport.ts` | SDK 进程内 transport | 场景 15 |
| `src/services/mcp/vscodeSdkMcp.ts` | `claude-vscode` 特化 | 场景 14 |
| `src/tools/MCPTool/MCPTool.ts` | 工具基础模板 | 所有场景 |
| `src/tools/McpAuthTool/McpAuthTool.ts` | 401 后的 OAuth 引导 | 场景 2, 8 |
| `src/tools/ListMcpResourcesTool/` `ReadMcpResourceTool/` | resources 访问 | 场景 3, 11（拉告警 / 查 plugin 资源） |
| `src/utils/mcpValidation.ts` | 输出大小估计、截断判定（默认 25k tokens） | 场景 12（防 OOM） |
| `src/utils/mcpOutputStorage.ts` | 大输出持久化（`ENABLE_MCP_LARGE_OUTPUT_FILES`） | 场景 5（BI 大 query 结果） |
| `src/commands/mcp/mcp.tsx` | `/mcp` 管理 UI | 运维（enable/disable/connect） |
| `src/utils/plugins/mcpPluginIntegration.ts` | plugin 暴露的 MCP server | 场景 11（marketplace） |
| `src/utils/plugins/mcpbHandler.ts` | MCPB（DXT）包 manifest | 场景 11（manifest 打包） |

---

## 10. 一句话总结

> **MCP 在企业里的核心价值，是把 agent 的"能力边界"从"prompt 里写死的 function"升级成"可治理、可审计、可隔离、可扩展的系统级能力"**。
>
> 没有 MCP：agent = 拿到 root 权限的实习生，**不可控**。
> 有 MCP：agent = 受控的、带 SSO 的、每一步可审计的"特权员工"，**可生产化**。
>
> 这就是为什么 Anthropic 把 MCP 做成开放协议、Microsoft / Google / OpenAI 全部接入——它不是 Anthropic 的"功能"，而是 AI agent 走向生产的**基础设施层**。

---

## 附录：与其他方案的对比

### MCP vs Function Calling（OpenAI 风格）

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| 工具定义 | 主程序 hardcode JSON Schema | server 动态 `tools/list` |
| 工具位置 | 同进程 | 任意进程 / 任意机器 |
| 凭据 | 主程序持有 | server 持有 |
| 多租户 | 自己实现 | scope + managed settings |
| 协议开放 | 厂商私有 | Anthropic 主导、各家接入 |

### MCP vs LangChain Tools / LlamaIndex Tools

| 维度 | LangChain/LlamaIndex | MCP |
|------|---------------------|-----|
| 工具定义 | Python 代码写 | server 协议声明 |
| 跨语言 | Python/JS 各一套 | 任何语言都能写 server |
| 跨进程 | 不支持 | stdio / HTTP / WS |
| 凭据隔离 | 主进程持有 | server 持有 |
| 企业管控 | 自己实现 | scope + managed settings |

### MCP vs gRPC

| 维度 | gRPC | MCP |
|------|------|-----|
| 调用方 | 程序员代码 | LLM agent |
| Schema | protobuf IDL | JSON Schema |
| 流式 | 双向 stream | SSE / WebSocket |
| 发现 | reflection（可选） | `tools/list` 必选 |
| 凭据 | TLS + metadata | OAuth/XAA/secure storage |
| 用途 | 微服务通信 | agent ↔ 工具 |

**gRPC 是给程序员用的 RPC，MCP 是给 agent 用的 RPC**。两者不冲突，可以共存。

---

> **下一步阅读**：
> - 想看 MCP 协议细节：[`docs/mcp/09-mcp-system-deep-dive.md`](09-mcp-system-deep-dive.md)
> - 想看客户端实现：[`docs/mcp/10-mcp-client-implementation.md`](10-mcp-client-implementation.md)
> - 想看 hook 与 MCP 协同：[`docs/hook/44-enterprise-value-of-hooks.md`](../hook/44-enterprise-value-of-hooks.md)