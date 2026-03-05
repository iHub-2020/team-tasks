# OpenClaw Multi-Agent 官方文档参考

> 来源：https://docs.openclaw.ai  
> 页面：/concepts/multi-agent · /concepts/session-tool · /tools/subagents · /tools/multi-agent-sandbox-tools · /gateway/configuration-reference · /gateway/security  
> 说明：本文件所有代码块均直接来自 OpenClaw 官方文档原文。标注 ⚠️ 的部分是原项目与官方不一致之处。

---

## ⚠️ 与原项目不匹配的关键差异

| 原项目文档内容 | 官方实际要求 | 严重程度 |
|---|---|---|
| 只提 `sessions_send` 放入 `tools.allow` | 还需要 `tools.agentToAgent` 才能跨 agent 发送 | 🔴 缺失会静默失败 |
| 未提 `tools.sessions.visibility` | 默认 `"tree"` 无法看到其他 agent 的 session | 🔴 缺失会静默失败 |
| DAG 模式全程用 `sessions_send`（阻塞） | 官方推荐并行场景用 `sessions_spawn`（非阻塞） | 🟡 功能受限 |
| 未提 `subagents.allowAgents` | `sessions_spawn` 跨 agent 需要显式 allowlist | 🟡 跨 agent spawn 会被拒 |
| 路径硬编码 `/home/ubuntu/clawd/...` | 官方示例均用 `~/.openclaw/workspace-<n>` | 🟡 跨平台不兼容 |

---

## 1. 多 Agent 路由配置

**来源：** https://docs.openclaw.ai/concepts/multi-agent

标准多 agent 配置示例（官方原始代码）：

```json
{
  "agents": {
    "list": [
      { "id": "home", "default": true, "workspace": "~/.openclaw/workspace-home" },
      { "id": "work", "workspace": "~/.openclaw/workspace-work" }
    ]
  },
  "bindings": [
    { "agentId": "home", "match": { "channel": "whatsapp", "accountId": "personal" } },
    { "agentId": "work",  "match": { "channel": "whatsapp", "accountId": "biz" } }
  ]
}
```

带完整工具和沙箱配置的 agent（官方 Family Bot 示例，原始代码）：

```json
{
  "agents": {
    "list": [
      {
        "id": "family",
        "name": "Family",
        "workspace": "~/.openclaw/workspace-family",
        "identity": { "name": "Family Bot" },
        "groupChat": {
          "mentionPatterns": ["@family", "@familybot", "@Family Bot"]
        },
        "sandbox": { "mode": "all", "scope": "agent" },
        "tools": {
          "allow": [
            "exec", "read",
            "sessions_list", "sessions_history",
            "sessions_send", "sessions_spawn", "session_status"
          ],
          "deny": ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"]
        }
      }
    ]
  },
  "bindings": [
    {
      "agentId": "family",
      "match": {
        "channel": "whatsapp",
        "peer": { "kind": "group", "id": "120363999999999999@g.us" }
      }
    }
  ]
}
```

> **⚠️ 差异说明：** 官方示例的 `tools.allow` 里 session 工具是逐个列出的，不是 `"group:sessions"` 简写。两种写法均被官方支持（见第 6 节），但官方示例代码本身用展开形式。

---

## 2. 跨 Agent 通信：三个必须同时配置的项

**⚠️ 原项目只提了 `tools.allow`，遗漏了以下两项。缺少任一项都会导致静默失败。**

### 2a. `tools.agentToAgent` — 跨 agent targeting 开关

**来源：** https://docs.openclaw.ai/gateway/configuration-reference（官方原始代码）

```json
{
  "tools": {
    "agentToAgent": {
      "enabled": false,
      "allow": ["home", "work"]
    }
  }
}
```

> 默认 `enabled: false`。`allow` 列表控制本 agent 可以向哪些 agent 的 session 发送消息。  
> 官方原文：**Cross-agent targeting still requires `tools.agentToAgent`.**

### 2b. `tools.sessions.visibility` — Session 可见范围

**来源：** https://docs.openclaw.ai/gateway/configuration-reference（官方原始代码）

```json
{
  "tools": {
    "sessions": {
      "visibility": "tree"
    }
  }
}
```

| 值 | 可见范围 |
|---|---|
| `"self"` | 只有当前 session key |
| `"tree"` | 当前 session + 它 spawn 出的 subagent sessions（**默认值**） |
| `"agent"` | 当前 agent id 下的所有 sessions |
| `"all"` | 所有 sessions |

> Sandbox clamp：当 session 在沙箱中运行且 `agents.defaults.sandbox.sessionToolsVisibility: "spawned"` 时，visibility 会被强制锁定为 `"tree"`，无视此配置。

### 2c. 通信专用 Agent 完整示例

**来源：** https://docs.openclaw.ai/tools/multi-agent-sandbox-tools（官方 Communication-only Agent 原始代码）

```json
{
  "tools": {
    "sessions": { "visibility": "tree" },
    "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
    "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
  }
}
```

**来源：** https://docs.openclaw.ai/gateway/security（官方 Security 文档原始代码）

```json
{
  "tools": {
    "sessions": { "visibility": "tree" },
    "allow": [
      "sessions_list", "sessions_history",
      "sessions_send", "sessions_spawn", "session_status",
      "whatsapp", "telegram", "slack", "discord"
    ],
    "deny": [
      "read", "write", "edit", "apply_patch",
      "exec", "process", "browser", "canvas",
      "nodes", "cron", "gateway", "image"
    ]
  }
}
```

---

## 3. sessions_send — 阻塞式发送

**来源：** https://docs.openclaw.ai/tools（官方 Tools 总览原文）

参数：
- `sessionKey`（必填；接受 session key 或 sessionId）
- `message`（必填）
- `timeoutSeconds?`（0 = fire-and-forget；>0 = 等待最多 N 秒）

官方原文行为说明：
- `sessions_send` runs a reply-back ping-pong (reply `REPLY_SKIP` to stop; max turns via `session.agentToAgent.maxPingPongTurns`, 0–5).
- After the ping-pong, the target agent runs an announce step; reply `ANNOUNCE_SKIP` to suppress the announcement.
- `status: "ok"` confirms the agent run finished, **not** that the announce was delivered.
- 超时时返回 `{ runId, status: "timeout", error }`，run 仍继续，可后续用 `sessions_history` 查询。

**适用场景（team-tasks）：** Linear 模式 — 派发给一个 worker agent，等待结果，再推进下一阶段。

---

## 4. sessions_spawn — 非阻塞并行派发

**来源：** https://docs.openclaw.ai/tools/subagents  
**来源：** https://docs.openclaw.ai/concepts/session-tool

参数（官方原文）：
- `task`（必填）
- `label?`（日志/UI 标签）
- `agentId?`（目标 agent id；跨 agent 需 allowAgents 配置）
- `model?`（覆盖该次 run 的模型；无效值报错）
- `thinking?`（覆盖 thinking 级别）
- `runTimeoutSeconds?`（默认 `agents.defaults.subagents.runTimeoutSeconds`，未设则 0；>0 时超时中止）
- `thread?`（默认 false；true = 请求 channel thread 绑定）
- `mode?`（`run` | `session`；默认 `run`；`session` 需 `thread: true`）
- `cleanup?`（`delete` | `keep`，默认 `keep`）
- `sandbox?`（`inherit` | `require`，默认 `inherit`）

返回值（立即返回，非阻塞）：
```json
{ "status": "accepted", "runId": "...", "childSessionKey": "agent:code-agent:subagent:uuid" }
```

官方原文关键说明：
- Always non-blocking: returns `{ status: "accepted", runId, childSessionKey }` immediately.
- Sub-agents default to the **full tool set minus session tools**.
- Sub-agents are **not** allowed to call `sessions_spawn`（除非 `maxSpawnDepth >= 2`）.
- Sub-agent announce is **best-effort**. If the gateway restarts, pending "announce back" work is lost.

**⚠️ 跨 agent spawn 需要额外配置（原项目完全未提）：**

**来源：** https://docs.openclaw.ai/gateway/configuration-reference

```json
{
  "agents": {
    "list": [{
      "id": "orchestrator",
      "subagents": {
        "allowAgents": ["code-agent", "test-agent", "docs-agent", "monitor-bot"]
      }
    }]
  }
}
```

> `["*"]` = 允许任意 agent。官方原文：`subagents.allowAgents`: allowlist of agent ids for `sessions_spawn` (`["*"]` = any; default: same agent only).

**适用场景（team-tasks）：** DAG 模式并行派发 — 同时 spawn 所有 ready 任务，各自独立运行，完成后 announce 结果。

---

## 5. sessions_send vs sessions_spawn 选择指南

**来源：** https://docs.openclaw.ai/tools/subagents

| 工具 | 阻塞 | team-tasks 适用场景 |
|---|---|---|
| `sessions_send` | ✅ 等待 reply 返回 | **Linear 模式** — 逐步派发，等结果，再推进 |
| `sessions_spawn` | ❌ 立即返回 runId | **DAG 模式** — 并行派发所有 ready 任务 |

---

## 6. 工具组简写（group:*）

**来源：** https://docs.openclaw.ai/tools/multi-agent-sandbox-tools（官方原文）

| 简写 | 展开为 |
|---|---|
| `group:runtime` | `exec`, `bash`, `process` |
| `group:fs` | `read`, `write`, `edit`, `apply_patch` |
| `group:sessions` | `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status` |
| `group:memory` | `memory_search`, `memory_get` |
| `group:ui` | `browser`, `canvas` |
| `group:automation` | `cron`, `gateway` |
| `group:messaging` | `message` |
| `group:nodes` | `nodes` |
| `group:openclaw` | 所有内置 OpenClaw 工具（不含 provider 插件） |

---

## 7. Sub-Agent 全局参数配置

**来源：** https://docs.openclaw.ai/gateway/configuration-reference（官方原始代码）

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "model": "minimax/MiniMax-M2.5",
        "maxConcurrent": 1,
        "runTimeoutSeconds": 900,
        "archiveAfterMinutes": 60
      }
    }
  }
}
```

Sub-agent 工具策略覆盖（官方原始代码）：

```json
{
  "tools": {
    "subagents": {
      "tools": {
        "deny": ["gateway", "cron"],
        "allow": ["read", "exec", "process"]
      }
    }
  }
}
```

> 当 `maxSpawnDepth >= 2` 时，depth-1 orchestrator sub-agents 额外获得 `sessions_spawn`、`subagents`、`sessions_list`、`sessions_history`，使其能管理自己的子 agent。

---

## 8. 工具过滤顺序（排查 tool 被 block 时用）

**来源：** https://docs.openclaw.ai/tools/multi-agent-sandbox-tools（官方原文）

> Each level can further restrict tools, but **cannot grant back** denied tools from earlier levels.

1. Tool profile（`tools.profile` 或 `agents.list[].tools.profile`）
2. Provider tool profile
3. 全局 tool policy（`tools.allow` / `tools.deny`）
4. Provider tool policy
5. Agent 专属 tool policy（`agents.list[].tools.allow/deny`）
6. Agent provider policy
7. Sandbox tool policy
8. Subagent tool policy

---

## 9. 针对 team-tasks 的完整 openclaw.json

在官方示例的基础上，结合你的 Debian 服务器目录结构（截图中的 workspace 命名）适配：

```json
{
  "agents": {
    "defaults": {
      "subagents": {
        "maxConcurrent": 4,
        "runTimeoutSeconds": 600,
        "archiveAfterMinutes": 60
      }
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "workspace": "~/.openclaw/workspace-main",
        "sandbox": { "mode": "off" },
        "subagents": {
          "allowAgents": ["code-agent", "test-agent", "docs-agent", "monitor-bot"]
        },
        "tools": {
          "sessions": { "visibility": "agent" },
          "allow": [
            "exec", "read",
            "sessions_list", "sessions_history",
            "sessions_send", "sessions_spawn", "session_status"
          ]
        }
      },
      {
        "id": "code-agent",
        "workspace": "~/.openclaw/workspace-reyantech-dev",
        "tools": {
          "allow": ["exec", "read", "write", "edit", "apply_patch"],
          "deny": ["sessions_send", "sessions_spawn", "sessions_list", "sessions_history", "session_status"]
        }
      },
      {
        "id": "test-agent",
        "workspace": "~/.openclaw/workspace-reyantech-qa",
        "tools": {
          "allow": ["exec", "read", "write"],
          "deny": ["sessions_send", "sessions_spawn", "sessions_list", "sessions_history", "session_status"]
        }
      },
      {
        "id": "docs-agent",
        "workspace": "~/.openclaw/workspace-reyantech-docs",
        "tools": {
          "allow": ["read", "write", "edit"],
          "deny": ["sessions_send", "sessions_spawn", "sessions_list", "sessions_history", "session_status"]
        }
      },
      {
        "id": "monitor-bot",
        "workspace": "~/.openclaw/workspace-reyantech-op",
        "tools": {
          "allow": ["read", "exec"],
          "deny": ["sessions_send", "sessions_spawn", "sessions_list", "sessions_history", "session_status"]
        }
      }
    ]
  },
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allow": ["code-agent", "test-agent", "docs-agent", "monitor-bot"]
    }
  },
  "skills": {
    "entries": {
      "team-tasks": {
        "enabled": true,
        "env": {
          "TEAM_TASKS_DIR": "/home/reyan/.openclaw/data/team-tasks"
        }
      }
    }
  }
}
```

---

## 10. 诊断命令

**来源：** https://docs.openclaw.ai/tools/multi-agent-sandbox-tools（官方原文）

```bash
# 查看所有 agent 及其 binding
openclaw agents list --bindings

# 解释当前 sandbox/tool 限制
openclaw sandbox explain

# 自动修复/迁移配置
openclaw doctor

# 查看 sandbox 容器（如果启用了沙箱）
docker ps --filter "name=openclaw-sbx-"

# 实时监控路由/sandbox/tool 事件
tail -f ~/.openclaw/logs/gateway.log | grep -E "routing|sandbox|tools|sessions"
```

---

## 11. 凭证隔离警告

**来源：** https://docs.openclaw.ai/concepts/multi-agent（官方原文）

> Auth profiles are per-agent. Each agent reads from its own:  
> `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`  
>
> Main agent credentials are **not** shared automatically.  
> **Never reuse `agentDir` across agents** (it causes auth/session collisions).  
> If you want to share creds, copy `auth-profiles.json` into the other agent's `agentDir`.

worker agents（code-agent、test-agent 等）需要各自独立认证，不能复用 main agent 的凭证。
