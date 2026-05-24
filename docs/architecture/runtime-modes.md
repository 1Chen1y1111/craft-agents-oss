# 运行模式设计

本文说明 Craft Agents 的运行形态、进程关系和启动路径。系统核心能力由 `packages/server-core` 和 `packages/shared` 提供，不同入口只负责把 UI、CLI 或浏览器请求接到同一套 RPC/session runtime 上。

## 运行模式总览

```text
+------------------+          +------------------------+
| Electron Desktop |          | Browser Web UI         |
| main + renderer  |          | apps/webui             |
+---------+--------+          +-----------+------------+
          |                               |
          | local or remote WS            | cookie-auth WS
          v                               v
+------------------------------------------------------+
| Craft Agent Server Runtime                           |
| packages/server-core + packages/shared               |
+----------------------+-------------------------------+
                       |
                       v
              sessions / agents / tools

+------------------+
| CLI              |
| apps/cli         |
+---------+--------+
          |
          | ws:// or spawn local server
          v
      same runtime
```

## Electron 本地模式

入口：`apps/electron/src/main/index.ts`

本地模式是桌面默认形态。Electron main process 同时承担三类职责：

- 宿主职责：窗口、菜单、deeplink、通知、BrowserView、系统文件打开、更新等。
- server 职责：通过 `bootstrapServer()` 启动本地 `WsRpcServer` 和 `SessionManager`。
- 平台注入：创建 Electron 版 `PlatformServices`，注入 server-core 子系统。

启动链路：

```text
app.whenReady()
  |
  +--> initialize resources, docs, permissions, themes
  |
  +--> create WindowManager and BrowserPaneManager
  |
  +--> create Electron PlatformServices
  |
  +--> bootstrapServer()
         |
         +--> WsRpcServer
         +--> SessionManager
         +--> core RPC handlers
         +--> GUI RPC handlers
         +--> model refresh service
```

renderer 不直接访问 Node/Electron 能力。preload 读取本地 WS 端口和 token，创建 `RoutedClient`，再通过 `buildClientApi()` 暴露 `window.electronAPI`。

关键文件：

- `apps/electron/src/main/index.ts`
- `apps/electron/src/preload/bootstrap.ts`
- `apps/electron/src/transport/routed-client.ts`
- `apps/electron/src/transport/build-api.ts`
- `apps/electron/src/transport/channel-map.ts`

## Electron 薄客户端模式

触发条件：设置 `CRAFT_SERVER_URL`。

薄客户端模式下，Electron 不启动本地 session runtime，只负责渲染 UI 和提供部分本地能力。preload 创建单个 `WsRpcClient` 直连远程 server。

```text
Electron renderer
  |
  v
preload WsRpcClient
  |
  v
remote WsRpcServer
  |
  v
remote SessionManager
```

安全约束：

- 非 localhost 的 `ws://` 会被拒绝。
- 远程部署应使用 `wss://`。
- token 通过 handshake 认证。

## Headless server

入口：`packages/server/src/index.ts`

Headless server 适合远程机器、Docker、CI 或 CLI 自动拉起场景。它不依赖 Electron，使用 Bun 运行。

启动职责：

- 解析环境变量和 TLS 配置。
- 可选挂载 Web UI 静态文件和 HTTP API。
- 调用 `bootstrapServer()`。
- 初始化 messaging gateway。
- 可选启动 `/health` HTTP 探针。
- 输出 `CRAFT_SERVER_URL` 和 `CRAFT_SERVER_TOKEN`。

关键环境变量：

| 变量 | 作用 |
| --- | --- |
| `CRAFT_SERVER_TOKEN` | server bearer token，必需 |
| `CRAFT_RPC_HOST` | 监听 host，默认 `127.0.0.1` |
| `CRAFT_RPC_PORT` | 监听端口，默认 `9100` |
| `CRAFT_RPC_TLS_CERT` / `CRAFT_RPC_TLS_KEY` | 启用 `wss://` |
| `CRAFT_WEBUI_DIR` | 启用 Web UI 静态资源 |
| `CRAFT_HEALTH_PORT` | 启用 `/health` 探针 |
| `CRAFT_BUNDLED_ASSETS_ROOT` | bundled assets 根目录 |

## Web UI 模式

入口：`apps/webui/src/App.tsx`

Web UI 是浏览器壳，复用 Electron renderer 的主 App。它不直接使用 Electron preload，而是在浏览器里构造兼容的 `window.electronAPI`。

```text
Browser
  |
  +--> GET /api/config
  |
  +--> createWebApi()
         |
         +--> WsRpcClient(cookie auth)
         +--> buildClientApi(CHANNEL_MAP)
         +--> web overrides
  |
  +--> lazy import Electron renderer App
```

Web override 负责替代或降级本地能力：

- 文件选择使用浏览器 `<input type="file">`。
- shell 打开 URL 使用浏览器 API。
- 窗口、菜单、更新、dock badge 等能力是 no-op 或浏览器替代。
- OAuth 使用浏览器新 tab 或当前页跳转。

关键文件：

- `apps/webui/src/App.tsx`
- `apps/webui/src/adapter/web-api.ts`
- `packages/server-core/src/webui`

## CLI 模式

入口：`apps/cli/src/index.ts`

CLI 只使用 WebSocket RPC，不复用 Electron renderer。它有两种工作方式：

- 连接已有 server：读取 `--url`、`--token` 或环境变量。
- 自包含 run：通过 `server-spawner.ts` 拉起本地 headless server，运行完清理。

```text
craft-cli run "prompt"
  |
  +--> spawn packages/server/src/index.ts
  |
  +--> CliRpcClient.connect()
  |
  +--> workspaces/create optional
  |
  +--> sessions/create
  |
  +--> sessions/sendMessage
  |
  +--> listen session:event
  |
  +--> cleanup
```

CLI 的 `CliRpcClient` 是精简实现：无自动重连、无 capability、无复杂状态，只保留 handshake、invoke、event listener。

## 模式选择原则

| 场景 | 推荐模式 |
| --- | --- |
| 普通桌面使用 | Electron 本地模式 |
| 远程机器长时间运行 session | Headless server + Electron 薄客户端 |
| 浏览器访问远程会话 | Headless server + Web UI |
| 脚本化、CI、一次性任务 | CLI `run` |
| 只读分享 transcript | Viewer |

## 维护注意事项

- 能放进 `server-core` 的运行时逻辑，不要留在 Electron app 层。
- Electron-only 能力应通过 GUI handler、client capability 或 web override 隔离。
- 新增 server 环境变量时，同时更新 README、相关启动脚本和本文。
- 改 thin-client 行为时，要检查 `RoutedClient`、Web UI adapter 和 CLI 是否需要同步调整。
