# 会话运行时设计

本文说明 `SessionManager` 的职责边界、会话生命周期、消息发送流程、事件模型和持久化格式。

## 核心位置

- `packages/server-core/src/sessions/SessionManager.ts`
- `packages/shared/src/sessions/storage.ts`
- `packages/shared/src/sessions/jsonl.ts`
- `packages/shared/src/sessions/persistence-queue.ts`
- `packages/server-core/src/handlers/rpc/sessions.ts`

`SessionManager` 是 server runtime 的中心协调器。它不只是 session CRUD，而是把 workspace、AgentBackend、sources、MCP pool、automations、messaging、持久化和 UI push 串起来。

## 主要职责

| 职责 | 说明 |
| --- | --- |
| workspace 初始化 | 加载工作区、配置 watcher、默认 labels/statuses、automations |
| session CRUD | 创建、读取、删除、导入、导出、转移 |
| 消息发送 | 处理 user message、attachments、agent streaming、redirect、cancel |
| 状态管理 | 名称、置顶、归档、未读、labels、statuses、notes、sources |
| 后端管理 | 创建或复用 Claude/Pi backend，处理模型和 thinking level |
| 工具运行时 | 构建 sources、MCP pool、session tools、permission callbacks |
| 持久化 | JSONL session、附件、plans、data、long responses、downloads |
| 事件广播 | `session:event`、session list changed、unread summary、files changed |
| 自动化 | 发布 app/agent/session metadata event |
| 消息平台 | 通过 wrapped event sink 推给 messaging gateway |

## Session 生命周期

```text
sessions:create
  |
  v
createSession(workspaceId, options)
  |
  +--> generate human-readable session id
  +--> create session directories
  +--> write initial session.jsonl
  +--> load into memory
  +--> push session list update

sessions:sendMessage
  |
  v
sendMessage()
  |
  +--> persist user message
  +--> start or resume AgentBackend
  +--> stream AgentEvent
  +--> persist assistant/tool events
  +--> push session:event
  +--> finalize complete/error/interrupted

sessions:delete
  |
  v
delete disk folder + cleanup runtime
```

## 消息发送流程

```text
UI / CLI
  |
  v
RPC sessions:sendMessage(sessionId, message, ...)
  |
  v
SessionManager.sendMessage()
  |
  +--> validate session and workspace
  +--> append user message
  +--> resolve runtime config
  |      |
  |      +--> LLM connection
  |      +--> model
  |      +--> thinking level
  |      +--> permission mode
  |      +--> enabled sources
  |
  +--> create/reuse AgentBackend
  |
  +--> build source servers and MCP pool
  |
  +--> for await (event of agent.chat())
          |
          +--> normalize event
          +--> update memory
          +--> persist JSONL
          +--> push session:event
```

## AgentEvent 到 UI 事件

Agent 后端输出统一 `AgentEvent`。SessionManager 根据事件类型更新 session 并推送 `RPC_CHANNELS.sessions.EVENT`。

常见事件：

- `text_delta`：助手文本增量。
- `tool_start`：工具开始。
- `tool_result`：工具结果。
- `permission_request`：请求用户批准。
- `credential_request`：请求用户输入或 OAuth。
- `plan_submitted`：计划提交，等待用户接受。
- `error`：运行错误。
- `complete`：当前 turn 完成。
- `interrupted`：用户取消、plan/auth handoff、redirect 等。

Renderer 侧由 `apps/electron/src/renderer/event-processor` 消费这些事件并更新 Jotai state。

## 持久化格式

每个 session 存储在工作区内：

```text
{workspaceRoot}/sessions/{sessionId}/
  session.jsonl
  attachments/
  plans/
  data/
  long_responses/
  downloads/
  meta/
```

`session.jsonl` 设计：

- 第一行是 session header。
- 后续每行是一条消息或事件映射后的存储对象。
- metadata 快读可以只读 header。
- 大 transcript 可以追加写入，避免频繁重写整个 JSON。

相关工具：

- `readSessionHeader()`
- `readSessionJsonl()`
- `writeSessionJsonl()`
- `sessionPersistenceQueue`

## 并发与持久化

SessionManager 同时维护内存态和磁盘态。持久化需要顺序化，避免流式事件高频写入导致文件损坏或乱序。

```text
AgentEvent stream
  |
  v
memory session update
  |
  v
sessionPersistenceQueue
  |
  v
session.jsonl
```

涉及 metadata 的写入有 guard，用于避免 watcher 误把自身原子写触发的文件事件当成外部修改。

## Branch 与远程转移

SessionManager 支持：

- session branch/fork。
- Pi turn anchors sidecar。
- Claude SDK session fork metadata。
- remote session export/import。
- 大 bundle chunked transfer。
- 转移时生成 conversation summary，供目标 server 继续上下文。

转移路径：

```text
source workspace
  |
  +--> sessions:export or exportRemoteTransfer
  |
  +--> optional chunked transfer
  |
  v
target remote workspace
  |
  +--> sessions:import
```

## Permission 与 handoff

permission mode 存在 session runtime 中，影响工具调用：

- `safe`：探索模式，阻止写操作。
- `ask`：写操作需要用户批准。
- `allow-all`：自动批准。

Plan 和 auth 是 handoff 点：

```text
Agent calls SubmitPlan or auth trigger
  |
  v
SessionManager creates UI-visible pending state
  |
  v
agent.interruptForHandoff(...)
  |
  v
user responds
  |
  v
sessions:sendMessage or respondToCredential continues
```

## 文件 watcher 与配置变化

SessionManager 会监听工作区配置相关文件，典型用途：

- sources 改动后刷新可用工具。
- skills 改动后刷新 mention 和 prompt。
- automations 改动后 reload。
- labels/statuses 改动后推送 UI。

新增 watcher 时要注意：

- 不要监听过宽目录导致高频事件。
- 自己写入的文件要有去抖或 guard。
- 远程工作区的本地/远程 ownership 要明确。

## 扩展建议

- 新增 session 字段：先确认是否属于 header、message 还是 sidecar。
- 新增事件类型：同时更新 AgentEvent、SessionEvent、event processor、UI 展示。
- 新增后台任务：用 session event 推进度，不要让 RPC handler 等任务结束。
- 新增导入导出内容：更新 bundle validation 和 transfer 逻辑。
