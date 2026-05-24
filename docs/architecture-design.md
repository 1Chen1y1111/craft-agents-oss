# Craft Agents 架构设计文档

最后更新：2026-05-24

本文基于当前仓库源码静态分析生成，目标是帮助开发者理解系统边界、运行时拓扑、核心数据流和扩展点。文档聚焦源码结构与架构关系，不替代 `README.md` 的用户安装说明，也不替代 `docs/cli.md` 的 CLI 命令参考。

细分专题文档见 `docs/architecture/`：

- `docs/architecture/runtime-modes.md`：运行模式与进程拓扑。
- `docs/architecture/rpc-transport.md`：WebSocket RPC、channel routing、断线恢复。
- `docs/architecture/session-runtime.md`：SessionManager、消息发送、持久化。
- `docs/architecture/agent-backends-and-tools.md`：Agent 后端、Sources、MCP、session tools。
- `docs/architecture/data-storage-and-security.md`：配置、工作区、凭据、安全边界。
- `docs/architecture/ui-clients.md`：Electron/Web/Viewer UI 架构。
- `docs/architecture/automation-and-messaging.md`：自动化与消息网关。

## 1. 架构目标

Craft Agents 是一个面向多 Agent 会话的桌面/远程/命令行系统。它把“用户界面”“会话运行时”“Agent 后端”“工具与外部数据源”拆成相对独立的层，使同一套会话能力可以运行在 Electron 本地进程、headless server、Web UI 和 CLI 中。

核心设计目标：

- 多入口：Electron 桌面端、Web UI、CLI、只读 Viewer 都围绕同一套会话和 RPC 协议工作。
- 可远程：Electron 可以作为薄客户端连接远程 server，本地和远程工作区通过路由表区分。
- 多后端：Claude Agent SDK 与 Pi SDK 后端通过统一 `AgentBackend` 接口接入。
- 可扩展工具：Sources、MCP、API source、session-scoped tools 通过统一工具注册与执行路径进入 Agent。
- 可恢复：会话、消息、附件、计划、长响应、下载文件等都落盘到工作区；RPC 传输也支持断线重连与事件回放。

## 2. 总体拓扑

```text
+--------------------+       +---------------------+
| Electron Renderer  |       | Web UI              |
| React + Jotai      |       | Browser adapter     |
+---------+----------+       +----------+----------+
          |                             |
          | electronAPI / WsRpcClient   | WsRpcClient
          v                             v
+--------------------------------------------------+
| RPC Transport                                     |
| WsRpcServer / WsRpcClient / RoutedClient          |
| packages/server-core/src/transport                |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
| RPC Handlers                                      |
| sessions, sources, skills, settings, files, etc.  |
| packages/server-core/src/handlers/rpc             |
| apps/electron/src/main/handlers for GUI-only APIs |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
| Session Runtime                                   |
| SessionManager + AutomationSystem + Messaging     |
| packages/server-core/src/sessions                 |
+-------------------------+------------------------+
                          |
                          v
+--------------------------------------------------+
| Agent Backend                                     |
| BaseAgent -> ClaudeAgent / PiAgent                |
| packages/shared/src/agent                         |
+-----------+-----------------------------+--------+
            |                             |
            v                             v
+------------------------+       +-----------------------+
| Claude Agent SDK       |       | Pi Agent subprocess   |
| @anthropic-ai/...      |       | packages/pi-agent-... |
+------------------------+       +-----------------------+

+--------------------------------------------------+
| Shared Persistence and Domain Logic               |
| config, credentials, sessions, workspaces, MCP,   |
| sources, skills, labels, statuses, automations    |
| packages/shared                                  |
+--------------------------------------------------+
```

## 3. Monorepo 模块职责

| 路径 | 职责 |
| --- | --- |
| `apps/electron` | 桌面应用。Main process 负责 Electron 生命周期、窗口、菜单、BrowserView、本地 OS 能力和嵌入式 server；renderer 负责 React UI。 |
| `apps/webui` | 浏览器版 UI。通过 `/api/config` 获取 WS 地址，构建 browser-compatible `electronAPI`，复用 Electron renderer 的主 App。 |
| `apps/cli` | WebSocket RPC CLI。可连接已有 server，也可在 `run` / `--validate-server` 中自动拉起本地 headless server。 |
| `apps/viewer` | 只读会话查看器。加载本地上传或 `/s/{id}` 共享会话，复用 `@craft-agent/ui` 的 SessionViewer。 |
| `packages/server-core` | 可复用 server 基座：WebSocket RPC、bootstrap、PlatformServices、核心 RPC handlers、SessionManager、WebUI HTTP 适配。 |
| `packages/server` | Bun headless server 入口。配置 TLS、WebUI、health endpoint、messaging gateway，并调用 `bootstrapServer`。 |
| `packages/shared` | 业务域核心：agent 后端、配置、凭据、sources、MCP、sessions、workspaces、skills、automations、协议、i18n 等。 |
| `packages/core` | 跨层基础类型与轻量工具，尤其是 message/session/workspace/server 类型与 mapper。 |
| `packages/ui` | 可复用 React UI 组件：聊天、Markdown、代码/差异查看器、overlay、terminal 等。 |
| `packages/session-tools-core` | Session-scoped tools 的规范、schema、handlers 和运行时安全工具。 |
| `packages/session-mcp-server` | 给 Codex/Pi 等外部 SDK 子进程使用的 stdio MCP server，复用 `session-tools-core`。 |
| `packages/pi-agent-server` | Pi SDK 隔离子进程。通过 JSONL 与主进程通信，避免把 Pi SDK 重依赖直接打进 Electron main。 |
| `packages/messaging-gateway` | Telegram、WhatsApp、Lark 等消息平台网关，负责绑定、路由、渲染和按钮回调。 |
| `packages/messaging-whatsapp-worker` | WhatsApp Baileys worker 子进程，通过 NDJSON 与 gateway adapter 通信。 |

## 4. 运行模式

### 4.1 Electron 本地模式

Electron main 在 `apps/electron/src/main/index.ts` 中完成启动：

1. 初始化 Sentry、i18n、资源路径、工具脚本 PATH、deeplink、window manager。
2. 构建 Electron 版 `PlatformServices`。
3. 调用 `bootstrapServer` 启动本地 `WsRpcServer`。
4. 创建 `SessionManager`，注册 core RPC handlers 和 GUI-only handlers。
5. preload 读取本地 WS 端口和 token，创建 `RoutedClient`。
6. renderer 通过 `window.electronAPI` 调用 RPC。

本地模式下，工作区可以是本地或远程。`RoutedClient` 会把本地 OS/Electron 能力留在本地 server，把会话运行时请求路由到拥有该工作区的 server。

```text
Renderer
   |
   v
Preload buildClientApi()
   |
   v
RoutedClient
   |---------------------- LOCAL_ONLY ---------------------> local WsRpcServer
   |
   +------------------ REMOTE_ELIGIBLE --------------------> local or remote WsRpcServer
```

### 4.2 Electron 薄客户端模式

当 `CRAFT_SERVER_URL` 存在时，Electron 不启动本地 session runtime，只创建窗口并让 preload 中的 `WsRpcClient` 直连远程 server。所有 channel 都走远程连接。对非 localhost 的 `ws://` 明文连接会被拒绝，要求使用 `wss://`。

### 4.3 Headless server

`packages/server/src/index.ts` 是独立 Bun server：

- 必须提供 `CRAFT_SERVER_TOKEN`，也支持 `--generate-token` 生成强 token。
- 可通过 `CRAFT_RPC_TLS_CERT` / `CRAFT_RPC_TLS_KEY` 启用 `wss://`。
- 可通过 `CRAFT_WEBUI_DIR` 在同一端口挂载 Web UI。
- 可通过 `CRAFT_HEALTH_PORT` 启用 `/health` 探针。
- 非本地地址绑定且未启用 TLS 时默认拒绝启动，除非显式传 `--allow-insecure-bind`。

### 4.4 CLI

`apps/cli` 使用精简版 `CliRpcClient`：

- 普通命令连接 `CRAFT_SERVER_URL` / `CRAFT_SERVER_TOKEN`。
- `send` 订阅 `session:event` 并流式输出 text/tool/error/complete。
- `run` 会自动 spawn 本地 server、创建临时 session、发送 prompt、清理 session。
- `--validate-server` 执行端到端 server 验证，包括 sources、skills、tools 和 cleanup。

### 4.5 Web UI

`apps/webui` 是浏览器壳：

1. `/api/config` 返回可连接的 WS URL。
2. 浏览器通过 cookie 完成 WebSocket upgrade 鉴权，不传 bearer token。
3. `createWebApi()` 复用 Electron 的 `CHANNEL_MAP` 和 `buildClientApi()`。
4. 对文件对话框、窗口管理、通知、shell 等本地能力做 web override。
5. 最终 lazy-load `apps/electron/src/renderer/App.tsx`。

## 5. RPC 与传输层

核心文件：

- `packages/server-core/src/transport/server.ts`
- `packages/server-core/src/transport/client.ts`
- `packages/shared/src/protocol/types.ts`
- `packages/shared/src/protocol/channels.ts`
- `packages/shared/src/protocol/routing.ts`
- `apps/electron/src/transport/channel-map.ts`
- `apps/electron/src/transport/build-api.ts`
- `apps/electron/src/transport/routed-client.ts`

传输协议使用统一 `MessageEnvelope`：

- `handshake` / `handshake_ack`：协议版本、workspaceId、token、clientId、clientCapabilities。
- `request` / `response`：RPC 调用和返回。
- `event`：server push。
- `sequence_ack`：客户端确认事件序号，用于重连回放。
- `error`：握手或传输级错误。

`WsRpcServer` 负责：

- WebSocket/HTTP/TLS 监听。
- token 或 WebUI session cookie 鉴权。
- 协议版本校验。
- handler 注册与 60s handler timeout。
- heartbeat ping/pong。
- client capability 反向调用。
- 每客户端事件 ring buffer、断线保留、重连 replay/stale 标记。

`WsRpcClient` 负责：

- 自动连接和重连。
- request/response 关联与 timeout。
- push event 订阅。
- `sequence_ack`。
- client-side capability handler。
- 暴露 connection state 给 UI banner。

Channel 分类是混合本地/远程架构的关键约束：

- `LOCAL_ONLY_CHANNELS`：窗口、菜单、本地文件对话框、系统 shell、更新、BrowserView、本地设置等。
- `REMOTE_ELIGIBLE_CHANNELS`：session runtime、sources、skills、labels、statuses、LLM connections、server 状态等。

新增 RPC 时必须同时更新 channel 常量、handler、channel map 和 routing 表，否则 exhaustiveness 测试会失败。

## 6. Server bootstrap 与平台注入

`bootstrapServer` 位于 `packages/server-core/src/bootstrap/headless-start.ts`，是 Electron 和 headless 共用的启动模板。

启动流程：

1. 校验 server token 强度。
2. 创建 `PlatformServices`，headless 默认用 `createHeadlessPlatform()`。
3. 初始化 `~/.craft-agent` 配置目录、默认配置和 server lock。
4. 创建 model refresh service、SessionManager、WsRpcServer。
5. 创建 `OAuthFlowStore` 和 handler dependency bag。
6. 注册 RPC handlers。
7. 将 `wsServer.push` 注入 SessionManager 作为 session event sink。
8. 初始化 SessionManager。
9. 启动 model refresh service。

`PlatformServices` 是 server-core 避免直接依赖 Electron 的关键抽象：

- Electron 实现包装 `app`、`shell`、`nativeImage`、logger、Sentry 等。
- Headless 实现使用 console logger 和 `sharp` 图像处理。
- GUI-only 能力通过 optional methods 或 client capabilities 处理。

## 7. 会话运行时

核心文件：`packages/server-core/src/sessions/SessionManager.ts`。

SessionManager 是系统的会话协调器，承担以下职责：

- 初始化工作区、配置 watcher、自动化系统和已有 session metadata。
- 创建、读取、删除、导入、导出 session。
- 管理 session 状态、标签、置顶、归档、未读、模型、thinking level、permission mode。
- 处理 `sendMessage`、流式 AgentEvent、取消、redirect、permission/credential 响应。
- 管理工作目录、附件、计划、long responses、binary downloads。
- 管理 source runtime、MCP pool、credential refresh、source activation retry。
- 管理 branch/fork、remote transfer、session summary 和 title 生成。
- 向 UI、messaging gateway 和 automations 广播事件。

简化后的消息发送链路：

```text
UI / CLI
  |
  v
RPC sessions:sendMessage
  |
  v
SessionManager.sendMessage()
  |
  +--> load session + append user message + persist JSONL
  |
  +--> resolve LLM connection + create/reuse AgentBackend
  |
  +--> build sources / MCP pool / session tools
  |
  +--> for await agent.chat(...)
  |        |
  |        +--> text_delta / tool_start / tool_result / error / complete
  |
  +--> update in-memory session + persist + push session:event
```

会话持久化位于 `packages/shared/src/sessions/storage.ts`，每个 session 存在工作区下：

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

`session.jsonl` 第一行是 header，后续行是消息。这个设计让 metadata 快速读取和大 transcript 追加写入可以分离。

## 8. Agent 后端

统一接口位于 `packages/shared/src/agent/backend/types.ts` 的 `AgentBackend`。核心方法包括：

- `chat()`：返回 `AsyncGenerator<AgentEvent>`。
- `abort()` / `forceAbort()` / `interruptForHandoff()` / `redirect()`。
- `runMiniCompletion()`：标题、摘要、连接测试等轻量模型调用。
- `postInit()`、`applyBridgeUpdates()`、`ensureBranchReady()`。
- 模型、thinking level、permission、source、capability 相关方法。

后端创建由 `packages/shared/src/agent/backend/factory.ts` 负责：

- `providerType: anthropic` -> `ClaudeAgent`
- `providerType: pi` / `pi_compat` -> `PiAgent`

`BaseAgent` 提供跨后端公共能力：

- permission mode 管理。
- source 状态管理与 prompt 注入。
- prompt builder、path processor、config watcher。
- usage tracking。
- mini agent / `call_llm` / `spawn_session` 等 session tool 协调。

后端差异：

- `ClaudeAgent` 直接使用 `@anthropic-ai/claude-agent-sdk`，事件经 `ClaudeEventAdapter` 转成统一 AgentEvent。
- `PiAgent` 通过 JSONL 子进程 `packages/pi-agent-server` 驱动 `@mariozechner/pi-coding-agent`，主进程负责工具执行、权限和事件适配。

Pi 子进程通信模型：

```text
PiAgent (main process)
   |
   | JSONL stdin/stdout
   v
pi-agent-server
   |
   +--> Pi SDK session
   +--> proxy tool definitions
   +--> pre_tool_use_request
   +--> tool_execute_request
   +--> mini_completion / llm_query
```

## 9. Sources、MCP 与 session tools

Sources 是外部能力接入的主要模型，定义在 `packages/shared/src/sources`。

Source 类型：

- `mcp`：远程 HTTP/SSE MCP 或本地 stdio MCP。
- `api`：REST API，被包装成 in-process MCP server。
- `local`：本地文件/工作区类 source。

工作区 source 存储结构：

```text
{workspaceRoot}/sources/{sourceSlug}/
  config.json
  guide.md
  .credential-cache.json
```

关键组件：

- `SourceServerBuilder`：把 `LoadedSource + credential` 转为 MCP server config 或 API MCP server。
- `McpClientPool`：主进程集中管理 MCP/API source 连接，生成 `mcp__{slug}__{tool}` proxy tool。
- `SourceManager`：Agent 内部的 source 状态跟踪、source context 注入、guide 读取约束提示。
- `TokenRefreshManager`：OAuth / renew endpoint token refresh。
- `SourceCredentialManager`：source 级凭据读写。

Session-scoped tools 使用两层结构：

- `packages/session-tools-core`：schema、tool defs、handler 的单一事实来源。
- `packages/session-mcp-server`：stdio MCP server，供外部 SDK 子进程访问相同工具。

这批工具包括 `SubmitPlan`、配置校验、source 测试、OAuth trigger、credential prompt、数据转换、模板渲染、session 自管理等。需要回到主进程的操作通过 callback 消息或 HTTP callback 完成。

## 10. 配置、工作区与凭据

全局配置目录来自 `packages/shared/src/config/paths.ts`：

```text
CRAFT_CONFIG_DIR 或 ~/.craft-agent/
  config.json
  config-defaults.json
  credentials.enc
  docs/
  themes/
  tool-icons/
  workspaces/
```

`config.json` 存储全局 LLM connections、默认工作区、UI 偏好、server mode 设置等。敏感数据不放在 config 中。

工作区可以位于任意磁盘路径；默认在 `~/.craft-agent/workspaces/`。典型结构：

```text
{workspaceRoot}/
  config.json
  sessions/
  sources/
  skills/
  labels/config.json
  statuses/config.json
  statuses/icons/
  automations.json
  automations-history.jsonl
  messaging/
```

凭据管理位于 `packages/shared/src/credentials`：

- `CredentialManager` 是统一入口。
- 当前写 backend 是 `SecureStorageBackend`。
- 凭据存储在 `~/.craft-agent/credentials.enc`。
- 文件格式使用 AES-256-GCM，key 由 OS 稳定机器标识 + PBKDF2 派生。
- 环境变量 backend 目前显式 disabled。

## 11. 自动化系统

自动化位于 `packages/shared/src/automations`，核心门面是 `AutomationSystem`。

每个工作区一个 AutomationSystem：

- 读取并验证 `automations.json`。
- 创建 `WorkspaceEventBus`。
- 注册 `PromptHandler`、`WebhookHandler`、`EventLogHandler`。
- 可启用 scheduler 处理 cron/scheduled events。
- 跟踪 session metadata diff，以便 label/status/tool/session 变化触发自动化。
- 写入并压缩 `automations-history.jsonl`。

SessionManager 初始化工作区时创建自动化系统，并在 session 事件、metadata 更新、agent event 等位置发布事件。

## 12. Messaging Gateway

消息平台能力位于 `packages/messaging-gateway`，由 Electron 和 headless server 共同通过 `createMessagingBootstrap()` 接入。

设计边界：

- `MessagingGatewayRegistry` 按 workspace 管理 gateway 实例、配置、pairing、runtime 状态。
- `MessagingGateway` 负责 adapter、router、commands、renderer、binding store。
- session event sink 被 `createFanOutSink()` 包装，同时推给 UI 和 messaging gateway。
- Telegram、WhatsApp、Lark 是平台 adapter。
- WhatsApp adapter 使用 `packages/messaging-whatsapp-worker` 子进程隔离 Baileys 状态。

```text
SessionManager event
   |
   v
wrapped EventSink
   |------------------> WsRpcServer.push -> UI
   |
   +------------------> MessagingGatewayRegistry
                          |
                          v
                       Workspace Gateway
                          |
             +------------+------------+
             v            v            v
          Telegram     WhatsApp       Lark
                         |
                         v
              whatsapp-worker NDJSON
```

## 13. UI 层

Electron renderer 位于 `apps/electron/src/renderer`：

- `main.tsx` 初始化 React、Jotai、ThemeProvider、i18n、Sentry。
- `App.tsx` 负责 app state、onboarding、workspace/session 加载、事件订阅和 context 组装。
- `ChatPage.tsx` 负责单会话聊天页面。
- `event-processor` 将 `session:event` 转成 UI state updates 和 side effects。
- Jotai atom family 用于按 session 隔离消息和 metadata，避免全量 session array 导致内存与重渲染问题。

共享 UI 位于 `packages/ui`：

- chat display、markdown 渲染、terminal output、code/diff viewer、annotations、overlay 等。
- Viewer 和 Electron renderer 都复用这些组件。

Web UI 复用 Electron renderer，所以新增跨端 UI 时应优先避免直接依赖 Electron API；需要平台能力时通过 `window.electronAPI` 抽象或在 `apps/webui/src/adapter/web-api.ts` 中提供 override。

## 14. 安全与可靠性设计

安全相关设计：

- RPC 远程 server 强制 token；弱 token 启动失败。
- 非 localhost 的明文 `ws://` server bind 默认被阻止。
- WebUI 使用 cookie session 鉴权，WS upgrade 可验证 session cookie。
- 凭据用加密文件保存，敏感值不写入普通 config。
- 文件路径、sessionId、workspace path 有多处 validation/sanitize。
- permission modes 控制工具写入能力：Explore、Ask to Edit、Auto。
- local stdio MCP 可按 workspace 设置过滤。

可靠性相关设计：

- server lock 防止同一 config dir 下多个 server 并发运行。
- session 使用 JSONL，支持追加和 header 快读。
- `sessionPersistenceQueue` 负责顺序化持久化。
- WS 传输有 heartbeat、reconnect、event replay、stale refresh。
- SessionManager 支持 flush all sessions 和 cleanup。
- Pi、WhatsApp 等重依赖通过子进程隔离，避免污染主进程 bundle。

## 15. 构建与发布

根 `package.json` 使用 Bun workspaces。主要脚本：

- `bun run electron:start`：构建并启动 Electron。
- `bun run electron:dev`：开发模式启动 Electron。
- `bun run server:start`：启动 headless server。
- `bun run server:prod`：构建 subprocess + WebUI 后启动 server。
- `bun run webui:build`：构建 Web UI。
- `bun run server:build:subprocess`：构建 `session-mcp-server` 和 `pi-agent-server`。
- `bun run typecheck:all`：跨 core/shared/server/electron/ui 等包类型检查。
- `bun run test`：运行 Bun 测试和 isolated tests。
- `bun run validate:ci`：类型、共享测试、文档工具 smoke、i18n 检查。

Electron build 由 `apps/electron` 的 esbuild、Vite 和 electron-builder 组合完成。headless server build 位于 `scripts/build-server.ts`。

## 16. 常见扩展路径

### 新增 RPC 能力

1. 在 `packages/shared/src/protocol/channels.ts` 增加 channel。
2. 在 `packages/shared/src/protocol/routing.ts` 放入 `LOCAL_ONLY_CHANNELS` 或 `REMOTE_ELIGIBLE_CHANNELS`。
3. 在 core handler 或 Electron GUI handler 中注册实现。
4. 在 `apps/electron/src/transport/channel-map.ts` 映射到 `electronAPI` 方法。
5. 更新 `apps/electron/src/shared/types.ts` 的 API 类型。
6. 补 handler、routing 或 transport 测试。

### 新增 source 类型或认证方式

1. 扩展 `packages/shared/src/sources/types.ts`。
2. 更新 source config validation/storage。
3. 更新 `SourceServerBuilder` 生成 MCP/API runtime。
4. 更新 credential manager 或 token refresh 逻辑。
5. 更新 UI 设置页与 session source activation 流程。
6. 补 source-builder、credential、MCP pool 测试。

### 新增 Agent 后端

当前源码的实际后端是 Claude 与 Pi。若新增后端，应保持以下边界：

1. 实现 `AgentBackend`，优先继承 `BaseAgent`。
2. 增加 event adapter，把 SDK 原生事件转换成统一 `AgentEvent`。
3. 在 backend factory 和 provider config 中注册 provider。
4. 处理 permission、source tools、session tools、mini completion、branch readiness。
5. 明确是否需要子进程隔离；重依赖或 runtime 冲突优先隔离。

### 新增跨端 UI

1. 业务状态优先放在 renderer context、hooks 或 Jotai atoms。
2. 纯 UI 组件优先放入 `packages/ui`。
3. 需要后端能力时通过 `electronAPI`，不要在 React 组件里直接 import Electron。
4. Web 不支持的能力要在 `apps/webui/src/adapter/web-api.ts` 提供 no-op 或浏览器替代实现。
5. 如果涉及工作区运行时，确认 channel 是 LOCAL_ONLY 还是 REMOTE_ELIGIBLE。

## 17. 架构约束与注意事项

- `server-core` 不应直接依赖 Electron；需要平台能力时走 `PlatformServices`、handler deps 或 client capabilities。
- Electron GUI handlers 应只放本地 UI/OS 能力，核心 session/source/skill 能力应放 server-core。
- `packages/shared` 是业务逻辑中心，但要避免从 shared 反向依赖 app 层。
- `RoutedClient` 的 channel routing 是远程工作区正确性的关键，新增 channel 不能跳过分类。
- Source guide 读取、permission modes、credential prompt 都是安全边界的一部分，改工具链时要同步考虑。
- Session storage 是用户数据格式，字段迁移要兼容旧文件。
- 子进程协议是稳定边界：Pi server JSONL、session MCP stdio、WhatsApp worker NDJSON 都需要保持向后兼容或显式迁移。
