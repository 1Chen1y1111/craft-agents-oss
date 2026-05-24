# RPC 与传输层设计

本文说明 WebSocket RPC 协议、channel routing、客户端 API 生成和断线恢复机制。所有交互入口最终都通过这层访问 server runtime。

## 核心目标

- 提供 Electron、Web UI、CLI、远程工作区共用的 RPC 通道。
- 支持 request/response、server push、client capability 反向调用。
- 支持断线重连和事件回放，降低长会话 UI 丢事件风险。
- 支持本地/远程混合路由，让 Electron 可以同时操作本机 OS 能力和远程工作区 session。

## 协议对象

协议类型定义在 `packages/shared/src/protocol/types.ts`。

`MessageEnvelope` 是唯一 wire format：

| 字段 | 作用 |
| --- | --- |
| `id` | 请求关联 ID |
| `type` | `handshake`、`request`、`response`、`event` 等 |
| `channel` | RPC channel 名称 |
| `args` | request 或 event 参数 |
| `result` | response 返回值 |
| `error` | 结构化错误 |
| `protocolVersion` | 协议版本 |
| `workspaceId` | client 所属工作区 |
| `token` | bearer token |
| `clientId` | server 分配的连接 ID |
| `clientCapabilities` | client 可被 server 反向调用的能力 |
| `registeredChannels` | server 已注册 channel 列表 |
| `seq` / `lastSeq` | 事件可靠投递序号 |
| `reconnectClientId` | 重连时声明旧 client ID |
| `stale` | server buffer 过期，client 需要全量刷新 |

## Server 端

核心文件：`packages/server-core/src/transport/server.ts`

`WsRpcServer` 职责：

- 监听 WS 或 WSS。
- 可选承载 HTTP handler，用于同端口 Web UI。
- handshake 超时和协议版本检查。
- bearer token 或 Web UI cookie session 鉴权。
- 注册和分发 RPC handler。
- RPC handler 60s 超时保护。
- server push 按 target 投递。
- client capability 反向调用。
- heartbeat 检测断连。
- 断线 client buffer 保留和 replay。

server push 目标类型：

```text
{ to: "all" }
{ to: "workspace", workspaceId }
{ to: "client", clientId }
```

## Client 端

核心文件：`packages/server-core/src/transport/client.ts`

`WsRpcClient` 职责：

- 建立连接并发送 handshake。
- 维护 pending request map。
- 订阅 push event。
- 注册 capability handlers。
- 自动重连和 exponential backoff。
- 周期性发送 `sequence_ack`。
- 暴露 `TransportConnectionState` 给 UI。

连接状态包括：

- `idle`
- `connecting`
- `connected`
- `reconnecting`
- `disconnected`
- `failed`

## 断线重连与事件回放

```text
client receives event seq=10
  |
  +--> stores lastSeenSeq=10
  |
  +--> periodically sends sequence_ack lastSeq=10

network disconnect
  |
  v
server keeps disconnected client buffer
  |
  v
client reconnects with reconnectClientId + lastSeq
  |
  +--> buffer available: server replays missed events
  |
  +--> buffer expired: handshake_ack stale=true
```

UI 收到 stale reconnect 后应刷新 session 列表或消息数据。Electron renderer 里对应逻辑位于 `useTransportConnectionState`、`useStaleSessionRecovery` 和相关 session load 工具。

## Channel 常量与分类

Channel 常量定义在 `packages/shared/src/protocol/channels.ts`。

混合本地/远程路由定义在 `packages/shared/src/protocol/routing.ts`：

- `LOCAL_ONLY_CHANNELS`：必须在本地 Electron server 执行。
- `REMOTE_ELIGIBLE_CHANNELS`：在拥有当前工作区的 server 执行。

```text
Renderer API call
  |
  v
RoutedClient.invoke(channel)
  |
  +--> isLocalOnly(channel) = true
  |       |
  |       v
  |    localClient
  |
  +--> false
          |
          v
       workspaceClient
       local or remote
```

典型 LOCAL_ONLY：

- Electron 窗口和菜单。
- 本地文件对话框。
- OS shell open file/show in folder。
- BrowserView browser pane。
- 自动更新。
- 本地外观、输入、power 设置。

典型 REMOTE_ELIGIBLE：

- session CRUD 和消息发送。
- sources、skills、labels、statuses。
- LLM connections。
- server status 和 workspace runtime。
- 文件读取、workspace 文件搜索。

## Electron API 生成

Electron 和 Web UI 共用 `CHANNEL_MAP`：

```text
CHANNEL_MAP method -> RPC channel
  |
  v
buildClientApi()
  |
  v
window.electronAPI.method(...)
```

关键文件：

- `apps/electron/src/transport/channel-map.ts`
- `apps/electron/src/transport/build-api.ts`
- `apps/electron/src/shared/types.ts`

新增 API 时要保证类型、channel map 和 handler 同步。

## GUI-only handler 与 core handler

Core handler 位于 `packages/server-core/src/handlers/rpc`，可被 Electron 和 headless server 共用。

Electron GUI-only handler 位于 `apps/electron/src/main/handlers`，处理本地 OS 或 Electron 特有能力。

```text
registerCoreRpcHandlers()
  sessions
  sources
  skills
  settings
  labels
  ...

registerGuiRpcHandlers()
  browser pane
  native workspace UI
  GUI system/settings handlers
```

## 新增 RPC 检查清单

1. 在 `channels.ts` 增加 channel 常量。
2. 在 `routing.ts` 分类到 LOCAL_ONLY 或 REMOTE_ELIGIBLE。
3. 在 core 或 GUI handler 注册 `server.handle(channel, handler)`。
4. 在 `channel-map.ts` 增加 API method 映射。
5. 在 `apps/electron/src/shared/types.ts` 补 `ElectronAPI` 类型。
6. 如涉及 Web UI，确认 `web-api.ts` 是否需要 override。
7. 补 routing exhaustive 测试或 handler 测试。

## 易错点

- 不要把远程工作区需要的 runtime channel 放进 LOCAL_ONLY。
- 不要把本地 OS path 操作放进 REMOTE_ELIGIBLE，否则远程 server 会拿到本机路径。
- handler 返回值必须可 JSON 序列化。
- 长耗时操作不要阻塞 RPC handler 过久，应通过 session event 或后台任务回传进度。
- 新增 client capability 时要在 preload/Web adapter 注册，并考虑 Web UI 是否支持。
