# GAP Analysis: team-tasks vs Claude Code Agent Teams

> Last updated: 2026-03  
> OpenClaw docs reference: https://docs.openclaw.ai  
> Claude Code docs reference: https://docs.anthropic.com/en/docs/claude-code/agent-teams

---

## Background

This document compares **team-tasks** (an OpenClaw skill) with **Claude Code's native Agent Teams** feature. Both solve multi-agent task coordination but in fundamentally different environments and with different tradeoffs.

team-tasks runs inside OpenClaw (a self-hosted AI gateway). Claude Code Agent Teams run inside the `claude` CLI tool on a developer's local machine. They are **not direct competitors** — they serve different deployment contexts — but the comparison clarifies design decisions and feature gaps in each.

---

## Capability Comparison Matrix

| Capability | team-tasks (OpenClaw) | Claude Code Agent Teams |
|---|---|---|
| **Orchestration mode** | Linear / DAG / Debate | Parallel sub-agents |
| **Task state persistence** | JSON file on disk | In-memory per session |
| **State survives crash/restart** | ✅ Yes (JSON on disk) | ❌ No |
| **Parallel dispatch** | ✅ DAG mode (`ready --json`) | ✅ Native |
| **Dependency graph** | ✅ Full DAG with cycle detection | ❌ No DAG — flat parallel only |
| **Agent dispatch mechanism** | `sessions_send` / `sessions_spawn` | `claude` sub-process spawn |
| **Cross-session visibility** | Requires `tools.agentToAgent` + `visibility` config | Native (shared filesystem) |
| **Progress tracking** | ✅ Per-task status + logs + output | ❌ No built-in tracking |
| **Debate / deliberation mode** | ✅ Yes (positions → cross-review → synthesis) | ❌ No |
| **Resume after failure** | ✅ `reset` command | ❌ Manual restart |
| **Multi-agent on remote server** | ✅ Native (OpenClaw runs on VPS/server) | ⚠️ Local machine only |
| **Channel integration** | ✅ Telegram/WhatsApp/Slack/etc. | ❌ Terminal only |
| **Workspace sharing** | Via `--workspace` flag + shared path | Via `CLAUDE.md` + shared cwd |
| **Token cost per task** | Low (CLI, no model calls) | High (each sub-agent = full context) |

---

## Key Gaps: team-tasks vs Claude Code Agent Teams

### Where Claude Code Agent Teams wins

**1. Zero-config sub-agent spawning**  
Claude Code spawns sub-agents natively with `Task` tool calls — no JSON state files, no CLI setup, no `tools.allow` configuration required. The orchestrator just calls `Task(description="...", prompt="...")` and Claude handles the rest.

**2. Shared filesystem as implicit state bus**  
Sub-agents in Claude Code share the local filesystem by default. There is no need for explicit output passing — agents can read each other's files directly.

**3. Tighter IDE integration**  
Claude Code has native integration with VS Code and JetBrains. team-tasks is purely CLI-driven.

---

### Where team-tasks wins

**1. Durable state across crashes and restarts**  
All project state is stored as JSON on disk. If the orchestrator crashes mid-pipeline, the project state is preserved and the pipeline can resume with `next` or `ready`. Claude Code sub-agents have no equivalent durability.

**2. DAG dependency graph with cycle detection**  
team-tasks supports arbitrary task dependency graphs, parallel dispatch of all unblocked tasks simultaneously, and prevents circular dependencies at task creation time. Claude Code's `Task` tool is flat — all sub-agents are dispatched in parallel with no dependency ordering.

**3. Debate mode**  
The three-phase deliberation workflow (initial positions → cross-review → synthesis) has no equivalent in Claude Code.

**4. Runs on remote servers + all chat channels**  
team-tasks runs anywhere OpenClaw runs — a Debian VPS, a cloud server — and dispatches agents over Telegram, WhatsApp, Slack, etc. Claude Code is a local dev tool.

**5. Low per-operation cost**  
The CLI tool itself makes no model calls. Only the dispatched agents consume tokens. Claude Code's `Task` tool spins up a full sub-agent context per invocation.

---

## Critical OpenClaw Configuration Requirements

The original version of this project documented `sessions_send` in `tools.allow` but omitted two other required configurations. **All three are needed** for team-tasks to work correctly:

### 1. `tools.agentToAgent` — Cross-agent targeting (REQUIRED)

Cross-agent `sessions_send` is **disabled by default**. Without this, `sessions_send` to another agent's session key will silently fail or be blocked.

```json
{
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allow": ["code-agent", "test-agent", "docs-agent", "monitor-bot"]
    }
  }
}
```

Reference: https://docs.openclaw.ai/gateway/configuration-reference — `tools.agentToAgent`

### 2. `tools.sessions.visibility` — Session scope (REQUIRED for cross-agent)

Default is `"tree"` (only current session + spawned subagent sessions visible). team-tasks orchestrates **separately configured agents**, not spawned subagents, so the visibility must be widened:

```json
{
  "tools": {
    "sessions": {
      "visibility": "agent"
    }
  }
}
```

Options: `self` | `tree` | `agent` | `all`. Use `"agent"` to allow the orchestrator to see all sessions belonging to its own agent ID.

Reference: https://docs.openclaw.ai/concepts/session-tool

### 3. `tools.allow` for the orchestrator agent

Use the `group:sessions` shorthand (available since OpenClaw v2026):

```json
{
  "agents": {
    "list": [{
      "id": "orchestrator-agent",
      "tools": {
        "allow": ["group:sessions", "exec", "read"]
      }
    }]
  }
}
```

`group:sessions` expands to: `sessions_list, sessions_history, sessions_send, sessions_spawn, session_status`.

Reference: https://docs.openclaw.ai/tools/multi-agent-sandbox-tools — Tool groups

---

## sessions_send vs sessions_spawn — Which to use?

team-tasks currently uses `sessions_send` for all dispatch. Here is when to use each:

| Tool | Blocking? | Best for |
|---|---|---|
| `sessions_send` | ✅ Blocking (waits for reply) | Linear mode — dispatch one agent, wait for result, then proceed |
| `sessions_spawn` | ❌ Non-blocking (returns immediately) | DAG mode — dispatch multiple agents in parallel, collect results asynchronously |

For **DAG mode**, `sessions_spawn` is more efficient: dispatch all `ready` tasks simultaneously, then collect results as each announces completion. `sessions_send` with a high `timeoutSeconds` also works but ties up the orchestrator context.

```
# Linear dispatch — sessions_send is correct
sessions_send(sessionKey="agent:code-agent:main", message=task, timeoutSeconds=300)

# DAG parallel dispatch — sessions_spawn is more efficient
sessions_spawn(task=task_desc, agentId="code-agent", label="implement-api")
sessions_spawn(task=task_desc, agentId="test-agent", label="write-tests")
# Both run in parallel; collect via announce callbacks
```

Reference: https://docs.openclaw.ai/tools/subagents

---

## Credential Isolation Warning

Each agent in OpenClaw has its own isolated auth store at `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`. **Credentials are not shared between agents.** Worker agents (code-agent, test-agent, etc.) must each be authenticated separately. Never reuse `agentDir` across agents.

Reference: https://docs.openclaw.ai/tools/multi-agent-sandbox-tools

---

## Summary

team-tasks fills real gaps that Claude Code Agent Teams does not address: durable state, DAG dependencies, debate mode, and remote/channel-based deployment. The main area where Claude Code wins is zero-configuration simplicity for local development workflows.

For production multi-agent pipelines that need to survive restarts, track progress across many tasks, or run on a remote server with chat channel integration, team-tasks provides capabilities Claude Code does not offer.
