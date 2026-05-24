# 自动化与消息网关设计

本文说明自动化系统和外部消息平台如何围绕 session runtime 扩展。两者都不应破坏核心 Agent 流程：自动化错误要隔离，消息平台只是 session event 的消费者和用户输入的入口之一。

## 总体关系

```text
SessionManager
  |
  +--> AutomationSystem
  |       |
  |       +--> EventBus
  |       +--> PromptHandler
  |       +--> WebhookHandler
  |       +--> EventLogHandler
  |
  +--> EventSink
          |
          +--> WsRpcServer.push -> UI
          |
          +--> MessagingGatewayRegistry
                    |
                    +--> Workspace MessagingGateway
                              |
                              +--> Telegram
                              +--> WhatsApp
                              +--> Lark
```

## 自动化系统

核心文件：

- `packages/shared/src/automations/automation-system.ts`
- `packages/shared/src/automations/event-bus.ts`
- `packages/shared/src/automations/handlers`
- `packages/shared/src/automations/validation.ts`
- `packages/shared/src/scheduler/scheduler-service.ts`

每个工作区拥有一个 `AutomationSystem` 实例。它负责读取 `automations.json`，校验配置，创建 event bus 和 handler。

## 自动化配置

默认文件：

```text
{workspaceRoot}/automations.json
{workspaceRoot}/automations-history.jsonl
```

`automations.json` 定义 matcher 和 actions。系统会：

- 读取并校验配置。
- 为缺失 matcher ID 的配置 backfill ID。
- 订阅工作区事件。
- 将执行历史写入 `automations-history.jsonl`。
- 启动时按保留策略压缩历史。

## 自动化事件来源

事件来源包括：

- session metadata 变化，例如 labels/status。
- agent events，例如 tool use。
- app events。
- schedule/cron tick。
- webhook 或 SDK bridge。

SessionManager 负责把核心 session 变化转换为自动化事件。

## 自动化执行模型

```text
event emitted
  |
  v
WorkspaceEventBus
  |
  v
matcher evaluation
  |
  +--> PromptHandler
  |      |
  |      +--> onPromptsReady
  |              |
  |              v
  |         SessionManager creates or sends prompt session
  |
  +--> WebhookHandler
  |
  +--> EventLogHandler
```

自动化设计原则：

- 自动化错误不能中断 agent 主流程。
- handler 通过 callback 回到 SessionManager，不直接 import server-core。
- 配置 reload 要可失败，失败时保留空配置或旧行为，避免 app 崩溃。
- 自动化 session 创建要经过 SessionManager，保证持久化和事件一致。

## Messaging bootstrap

核心文件：

- `packages/messaging-gateway/src/bootstrap.ts`
- `packages/messaging-gateway/src/registry.ts`
- `packages/messaging-gateway/src/gateway.ts`

Electron 和 headless server 都必须通过 `createMessagingBootstrap()` 接入，避免两条 host 路径发散。

接入流程：

```text
createMessagingBootstrap()
  |
  +--> registry
  |
  +--> pass registry into HandlerDeps
  |
  +--> bootstrapServer()
  |
  +--> handle.setPublisher(wsServer.push)
  |
  +--> sessionManager.setEventSink(handle.wrapSink(baseSink))
  |
  +--> initializeWorkspaces(localWorkspaceIds)
```

## MessagingGatewayRegistry

Registry 是 workspace 级网关管理器：

- 管理每个 workspace 的 `MessagingGateway`。
- 管理 pairing code。
- 管理 `messaging/config.json`。
- 管理平台 adapter 生命周期。
- 作为 RPC handlers 的 `messagingRegistry` dependency。
- 给 SessionManager 安装 automation topic binder hook。

远程工作区规则：

- 本地 Electron 只初始化本地 owned workspaces 的 messaging。
- remote-owned workspace 的 messaging 在 remote server 上运行。

## MessagingGateway

每个 workspace 一个 `MessagingGateway`。

内部组件：

- `BindingStore`：外部 chat/thread 到 session 的绑定。
- `PendingSendersStore`：pending sender 状态。
- `Router`：incoming message 路由到 session。
- `Commands`：处理 `/pair`、绑定、控制命令。
- `Renderer`：把 session event 渲染成平台消息。
- `PlanTokenRegistry`：plan approval button token。
- platform adapters：Telegram、WhatsApp、Lark。

```text
incoming platform message
  |
  v
Adapter
  |
  v
Router / Commands
  |
  v
SessionManager.sendMessage()

session event
  |
  v
Renderer
  |
  v
Adapter.send(...)
```

## WhatsApp worker

核心文件：

- `packages/messaging-gateway/src/adapters/whatsapp`
- `packages/messaging-whatsapp-worker/src/worker.ts`
- `packages/messaging-whatsapp-worker/src/protocol.ts`

WhatsApp 使用 Baileys，运行在独立 worker 子进程中。主进程和 worker 通过 NDJSON 通信。

协议方向：

```text
WhatsAppAdapter
  |
  | WorkerCommand NDJSON
  v
whatsapp-worker
  |
  | WorkerEvent NDJSON
  v
WhatsAppAdapter
```

worker 负责：

- Baileys auth state。
- QR 或 pairing code。
- 连接状态。
- incoming message 和附件下载。
- send text/file 结果。

主进程负责：

- 生命周期和重连策略。
- 将 worker incoming event 转为 gateway incoming message。
- 将 gateway send 操作转成 worker command。

## 权限与访问控制

消息平台有自己的访问控制：

- workspace messaging config。
- platform owners。
- channel binding access mode。
- pre-binding access。
- pending sender。
- button callback 访问校验。

这与 Agent tool permission 是两层不同边界：

- messaging access 控制“谁能从外部聊天控制 session”。
- Agent permission 控制“Agent 能执行哪些工具或写操作”。

## 扩展新消息平台

1. 实现 `PlatformAdapter`。
2. 在 registry 中接入配置和 runtime 状态。
3. 实现 incoming message、button press、send text/file。
4. 接入 access control。
5. 处理 pairing 或 credential setup。
6. 增加 UI/RPC handlers 所需状态。
7. 补 router、commands、renderer 和 adapter 测试。

## 常见风险

- 直接在 Electron 或 headless host 中构造 registry，导致双 host 行为分叉。
- 消息平台错误影响 SessionManager 主流程。
- plan/permission button 未做幂等，导致重复批准或拒绝。
- remote-owned workspace 在本地错误启动 messaging。
- worker stdout 混入非 JSON 日志，破坏 NDJSON 协议。
