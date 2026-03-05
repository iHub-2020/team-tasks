# OpenClaw Multi-Bot 协同网关配置最佳实践指南

本文档总结了基于 OpenClaw Gateway 架构下让 PM（项目经理bot）调度其他干系 Bot（DEV/QA/DOCS/OP）的核心避坑指南与连通性实践。

## 1. PM 必须配置高级 Tools 权限
如果仅给 PM 配置 `"profile": "messaging"` 是无法跨 Bot 唤醒的。PM 需要显式调用 `openclaw-action` 系工具：
- 必须将 `sessions_send`、`sessions_spawn`、`sessions_list`、`sessions_history` 放入其自带的 `tools.allow` 权限组中。

## 2. 跨会话核心参数：tools.agentToAgent
跨会话的拦截报错 `Agent-to-agent messaging denied by tools.agentToAgent.allow` 意味着触发了网关底层的 ACL。
- 必须在全局配置的 `tools.agentToAgent.allow` 数组中不仅填入目标 Bot 的 ID，**也要填入发送端（PM）的 ID**。
- 正确示例：`"allow": ["reyantech_pm", "reyantech_dev", "reyantech_qa", "reyantech_docs", "reyantech_op"]`

## 3. 避免子 Agent（subagents）权限泄露配置位置
- PM 的 agent 配置中需要通过 `"subagents": { "allowAgents": ["reyantech_dev", "reyantech_qa"...] }` 来放开跨组通信。
- 像 `runTimeoutSeconds` 等细分控制不要错误配置在 Agent 层级，只应配置在 `agents.defaults.subagents` 或全局配置中。
- `tools.sessions.visibility: "all"` 应作为系统级全局配置，以允许所有 Bot 根据 ACL 查看相关群聊 sessions。

## 4. Telegram Supergroup 的群组 ID 隐形升级坑
当有足够数量的 Bot 或人员加入一个普通的 Telegram 群聊时，Telegram 服务端会**隐式将该群提升为 Supergroup**。
**后果：** 
- 原有的普通群聊 ID（一般为 `-5xxxxxxxxx` 开头）会立即失效。
- 升级后的 Supergroup ID 会被前置追加 `-100` （变为 `-1005xxxxxxxxx`，或者是全新的 `-100` 号段，例如 `-1003826811001`）。
- **破局方式：** 若遇到 `400: Bad Request: group chat was upgraded to a supergroup chat`。请使用 `openclaw sessions list` 查询日志或捕捉最新交互，替换原配置中 `mentionPatterns` 及 `groups` 中的群 ID。

## 5. 多 Bot 沙箱隔离：Workspace 配置
在多并发场景中需要强烈推荐配置各个独立 Bot 的 `workspace` 映射。通过为 DEV、QA 等建立各自分离的工作目录（例如 `workspace-reyantech-dev`），即便 PM 用并发命令 `sessions_spawn` 时派发了异构计算任务，它们读写文件也是基于各自独立的物理隔离，不会发生并行读写同一文件的“踩踏冲突”。
