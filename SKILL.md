---
name: team-tasks
description: Multi-agent pipeline coordination with Linear, DAG, and Debate modes. Use to orchestrate sequential pipelines, parallel dependency graphs, or multi-agent deliberation workflows via a shared JSON task store.
user-invocable: false
metadata: {"openclaw":{"requires":{"bins":["python3"]},"emoji":"🗂️","os":["linux","darwin"]}}
---

# Team Tasks — Multi-Agent Pipeline Coordination

## Overview

This skill coordinates multi-agent development workflows through shared JSON task files.
The CLI tool lives at: `{baseDir}/scripts/task_manager.py`

**AGI is the command center.** Worker agents do not communicate directly with each other.
All state is shared via JSON files. AGI dispatches via `sessions_send` (Linear) or `sessions_spawn` (DAG parallel), tracks state via this CLI.

### ⚙️ Prerequisites (verify before use)

1. **Python 3.12+** must be on PATH — the `metadata.requires.bins` gate checks for `python3`.
2. **Three OpenClaw config keys required** — all three must be set or cross-agent dispatch silently fails:

   ```json
   {
     "agents": {
       "list": [{
         "id": "your-orchestrator-agent",
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
       }]
     },
     "tools": {
       "agentToAgent": {
         "enabled": true,
         "allow": ["code-agent", "test-agent", "docs-agent", "monitor-bot"]
       }
     }
   }
   ```

   - `tools.agentToAgent` — default `enabled: false`; without this, `sessions_send` to another agent is silently blocked.
   - `tools.sessions.visibility` — default `"tree"` (only spawned subagents visible); set to `"agent"` to see independently configured worker agents.
   - `subagents.allowAgents` — required for `sessions_spawn` to target other agents; default allows same agent only.

3. **Data directory** is controlled by env var `TEAM_TASKS_DIR`.
   Default when unset: `~/.openclaw/data/team-tasks/`
   Override in your `openclaw.json`:
   ```json
   {
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
   The script respects this env var automatically — no hardcoded paths.

---

## Invoking the CLI

Always use the full `{baseDir}`-relative path:

```bash
python3 {baseDir}/scripts/task_manager.py <command> [args]
```

Alias for brevity in this document:
```
TM="python3 {baseDir}/scripts/task_manager.py"
```

---

## Mode A — Linear Pipeline

Sequential pipeline. Agents execute one after another in order.
**Use for:** bug fixes, simple features, step-by-step workflows.

### Setup

```bash
# 1. Create project with pipeline order
$TM init my-api \
  --goal "Build REST API with tests and docs" \
  --pipeline "code-agent,test-agent,docs-agent,monitor-bot"

# 2. Assign task descriptions to each stage
$TM assign my-api code-agent "Implement Flask REST API: GET/POST/DELETE /items"
$TM assign my-api test-agent "Write pytest tests, target 90%+ coverage"
$TM assign my-api docs-agent "Write README with API docs and examples"
$TM assign my-api monitor-bot "Security audit and deployment readiness check"
```

### Dispatch Loop (repeat until done)

```bash
# 1. Get next stage
NEXT=$($TM next my-api --json)
# → { "stage": "code-agent", "task": "Implement Flask REST API..." }

# 2. Mark in-progress
$TM update my-api code-agent in-progress

# 3. Dispatch to worker agent via sessions_send
#    Include the task description and any prior stage outputs
sessions_send(sessionKey="agent:code-agent:main", message="<task>", timeoutSeconds=120)

# 4. Save output and mark done (auto-advances to next stage)
$TM result my-api code-agent "<agent reply summary>"
$TM update my-api code-agent done
# ▶️  Next: test-agent  ← auto-advance confirmed
```

### Status Check

```bash
$TM status my-api
# 📋 Project: my-api
# 🎯 Goal: Build REST API with tests and docs
# 📊 Status: active  |  Mode: linear
# ▶️  Current: test-agent
#
#   ✅ code-agent: done
#      Output: Created app.py with 3 endpoints
#   🔄 test-agent: in-progress
#   ⬜ docs-agent: pending
#   ⬜ monitor-bot: pending
#
#   Progress: [██░░] 2/4
```

---

## Mode B — DAG (Dependency Graph)

Tasks declare dependencies; ready tasks are dispatched **in parallel** when deps complete.
**Use for:** large features, spec-driven dev, complex parallel workstreams.

### Setup

```bash
# 1. Create DAG project
$TM init my-feature --mode dag \
  --goal "Build search feature with parallel workstreams"

# 2. Add tasks — dependencies must be added BEFORE they are referenced
$TM add my-feature design      -a docs-agent  --desc "Write API spec"
$TM add my-feature scaffold    -a code-agent  --desc "Create project skeleton"
$TM add my-feature implement   -a code-agent  -d "design,scaffold" --desc "Implement API"
$TM add my-feature write-tests -a test-agent  -d "design"          --desc "Write test cases from spec"
$TM add my-feature run-tests   -a test-agent  -d "implement,write-tests" --desc "Run all tests"
$TM add my-feature write-docs  -a docs-agent  -d "implement"             --desc "Write final docs"
$TM add my-feature review      -a monitor-bot -d "run-tests,write-docs"  --desc "Final review"

# 3. Visualize dependency tree
$TM graph my-feature
```

### Dispatch Loop (parallel)

```bash
# 1. Get ALL currently dispatchable tasks
READY=$($TM ready my-feature --json)
# → [{ "id": "design", "agent": "docs-agent" }, { "id": "scaffold", "agent": "code-agent" }]

# 2. For EACH ready task — dispatch in parallel via sessions_spawn (non-blocking)
for task in $READY:
  $TM update my-feature <task.id> in-progress
  sessions_spawn(
    task="Task: <task.desc>\nPrior outputs: <task.depOutputs>",
    agentId="<task.agent>",
    label="<task.id>",
    runTimeoutSeconds=600
  )
  # → returns immediately: { status: "accepted", runId, childSessionKey }

# 3. On each agent announce (result arrives asynchronously):
$TM result my-feature <task.id> "<agent announce result>"
$TM update my-feature <task.id> done
# 🟢 Unblocked: <newly-unblocked tasks> ← check stdout

# 4. Re-run $TM ready to pick up newly unblocked tasks
# Repeat until all done
```

**Key DAG behaviours (verified):**
- `ready --json` includes `depOutputs` field — pass prior stage results directly to agents.
- Cycle detection runs on every `add` — circular deps are rejected immediately.
- Partial failure: unrelated branches continue; only downstream tasks of a failed task block.

---

## Mode C — Debate (Multi-Agent Deliberation)

Same question → multiple agents → positions → cross-review → synthesis.
**Use for:** code reviews, architecture decisions, competing hypotheses.

### ⚠️ Rules
- Add **all** debaters **before** calling `round start`. Adding debaters after `round start` will error.

### Setup & Full Flow

```bash
# 1. Create debate project
$TM init security-review --mode debate \
  --goal "Review auth module for security vulnerabilities"

# 2. Add debaters with roles (ALL before round start)
$TM add-debater security-review code-agent  --role "security expert focused on injection attacks"
$TM add-debater security-review test-agent  --role "QA engineer focused on edge cases"
$TM add-debater security-review monitor-bot --role "ops engineer focused on deployment risks"

# 3. Start initial round — dispatch prompts to each debater
$TM round security-review start
# 🗣️  Debate Round 1 (initial) started

# 4. Dispatch to each debater via sessions_send, collect positions
$TM round security-review collect code-agent  "Found SQL injection in login()"
$TM round security-review collect test-agent  "Missing input validation on email field"
$TM round security-review collect monitor-bot "No rate limiting on auth endpoints"
# ✅ Round 1 complete. Next: round security-review cross-review

# 5. Generate cross-review prompts (each agent gets others' positions)
$TM round security-review cross-review

# 6. Dispatch cross-review prompts, collect reviews
$TM round security-review collect code-agent  "Agree on validation. Rate limiting is critical."
$TM round security-review collect test-agent  "SQL injection is most severe. Adding rate limit tests."
$TM round security-review collect monitor-bot "Both findings valid. Recommending WAF layer."

# 7. Synthesize — outputs all positions + cross-reviews for final synthesis
$TM round security-review synthesize
```

---

## CLI Reference

| Command | Mode | Usage |
|---------|------|-------|
| `init` | all | `init <project> -g "goal" [-m linear\|dag\|debate] [-p "a,b,c"]` |
| `add` | dag | `add <project> <task-id> -a <agent> [-d "dep1,dep2"] [--desc "..."]` |
| `add-debater` | debate | `add-debater <project> <agent-id> [--role "..."]` |
| `round` | debate | `round <project> start\|collect\|cross-review\|synthesize` |
| `status` | all | `status <project> [--json]` |
| `assign` | linear/dag | `assign <project> <stage> "desc"` |
| `update` | linear/dag | `update <project> <stage> pending\|in-progress\|done\|failed\|skipped` |
| `next` | linear | `next <project> [--json]` |
| `ready` | dag | `ready <project> [--json]` |
| `graph` | dag | `graph <project>` |
| `result` | linear/dag | `result <project> <stage> "output"` |
| `log` | linear/dag | `log <project> <stage> "msg"` |
| `reset` | linear/dag | `reset <project> [stage] [--all]` |
| `history` | linear/dag | `history <project> <stage>` |
| `list` | all | `list` |

### Status Values

| Value | Icon | Meaning |
|-------|------|---------|
| `pending` | ⬜ | Waiting for dispatch |
| `in-progress` | 🔄 | Agent is working |
| `done` | ✅ | Completed |
| `failed` | ❌ | Failed — blocks downstream tasks |
| `skipped` | ⏭️ | Intentionally skipped |

---

## Common Pitfalls

### ❌ Linear: stage ID is the agent name, NOT a number
```bash
# WRONG — "stage '1' not found"
$TM assign my-project 1 "Build API"

# CORRECT
$TM assign my-project code-agent "Build API"
```

### ❌ DAG: dependency must exist before being referenced
```bash
# WRONG — "dependency 'design' not found"
$TM add my-project implement -a code-agent -d "design"

# CORRECT — add deps first
$TM add my-project design    -a docs-agent --desc "Write spec"
$TM add my-project implement -a code-agent -d "design" --desc "Implement"
```

### ❌ Debate: cannot add debaters after round starts
```bash
# WRONG
$TM round my-debate start
$TM add-debater my-debate new-agent  # Error!

# CORRECT — add all debaters before starting
$TM add-debater my-debate agent-a
$TM add-debater my-debate agent-b
$TM round my-debate start
```

### ❌ Cross-agent dispatch silently fails
Three config keys must all be set. Missing any one causes silent failure with no error message:
- `tools.agentToAgent.enabled: true` + `allow` list
- `tools.sessions.visibility: "agent"` (default `"tree"` cannot see independent worker agents)
- `subagents.allowAgents` (required for `sessions_spawn` to target other agents)

See Prerequisites section for the complete `openclaw.json` example.

---

## Data Storage

Project files are stored as JSON. Path resolution order:
1. `TEAM_TASKS_DIR` environment variable (set via `skills.entries.team-tasks.env` in openclaw.json)
2. Default fallback: `~/.openclaw/data/team-tasks/`

```bash
# Check current effective path
echo ${TEAM_TASKS_DIR:-~/.openclaw/data/team-tasks}

# List all projects
$TM list
```
