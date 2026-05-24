# UI 客户端设计

本文说明 Electron renderer、Web UI、Viewer 和共享 UI 包之间的关系，以及跨端 UI 改动的边界。

## UI 拓扑

```text
+-----------------------------+
| apps/electron/src/renderer  |
| App, ChatPage, settings     |
+--------------+--------------+
               |
               | imports
               v
+-----------------------------+
| packages/ui                 |
| chat, markdown, overlays    |
+-----------------------------+

+-----------------------------+
| apps/webui                  |
| browser adapter + shell     |
+--------------+--------------+
               |
               | lazy imports Electron App
               v
      same renderer UI

+-----------------------------+
| apps/viewer                 |
| readonly transcript viewer  |
+--------------+--------------+
               |
               | imports
               v
         packages/ui
```

## Electron renderer

入口：`apps/electron/src/renderer/main.tsx`

主要职责：

- 初始化 React。
- 初始化 i18n。
- 初始化 Sentry renderer。
- 挂载 Jotai provider。
- 挂载 ThemeProvider。
- 渲染 `App` 和全局 Toaster。

`App.tsx` 是 renderer 的状态中枢：

- app state：loading、onboarding、workspace picker、ready。
- 工作区加载。
- session metadata 和 lazy message loading。
- LLM connections、sources、skills、labels、statuses。
- 订阅 `session:event`。
- 调用 `useEventProcessor()` 处理流式事件。
- 组装 `AppShellContext`，供页面和组件使用。

## 聊天页

核心文件：`apps/electron/src/renderer/pages/ChatPage.tsx`

职责：

- 读取当前 session atom。
- 确保消息已加载。
- 管理输入框 draft。
- 获取 pending permission / credential。
- 处理 session menu、labels、sources、model、status。
- 渲染 `ChatDisplay`。

UI 状态主要通过 Jotai atom family 分离，避免整个 session list 每次流式 token 都重渲染。

## Event processor

路径：`apps/electron/src/renderer/event-processor`

设计目标：

- 把 server 的 `session:event` 转成 UI state mutation。
- 把副作用集中管理，例如 toast、restore input、background task 更新。
- 让 ChatDisplay 尽量只消费已经整理好的 session state。

简化链路：

```text
WsRpcClient event: session:event
  |
  v
App subscription
  |
  v
useEventProcessor()
  |
  +--> Jotai atom updates
  +--> side effects
  +--> navigation/toast
```

## Web UI 复用策略

Web UI 不重写业务 UI，而是构造浏览器版 `window.electronAPI` 后复用 Electron renderer App。

```text
apps/webui/src/App.tsx
  |
  +--> fetch /api/config
  |
  +--> createWebApi()
  |      |
  |      +--> WsRpcClient
  |      +--> buildClientApi(CHANNEL_MAP)
  |      +--> web overrides
  |
  +--> lazy import apps/electron/src/renderer/App
```

这意味着：

- renderer 组件不能直接 import `electron`。
- 本地能力必须通过 `window.electronAPI`。
- 如果 Electron 有新 API，Web UI 可能要在 `web-api.ts` 提供 override。

## Viewer

路径：`apps/viewer`

Viewer 是只读 transcript 应用，不参与 session runtime。它可以：

- 从 `/s/api/{id}` 加载共享 session。
- 从本地上传 session JSON。
- 使用 `@craft-agent/ui` 的 `SessionViewer` 展示消息。
- 打开 code、diff、terminal、JSON、文档 overlay。

Viewer 适合验证 `packages/ui` 的只读展示能力，但不能代表 Electron runtime 行为。

## 共享 UI 包

路径：`packages/ui`

主要组件域：

- `components/chat`：会话展示、turn card、plan、inline execution。
- `components/markdown`：Markdown、Mermaid、LaTeX、图片、PDF、HTML、datatable、spreadsheet 等 block。
- `components/code-viewer`：代码和 diff 展示。
- `components/terminal`：终端输出。
- `components/annotations`：富文本 annotation overlay。
- `components/ui`：通用 UI primitives。

抽象原则：

- 不依赖 Electron。
- 通过 `PlatformContext` 或 props 接收平台行为。
- 只读展示逻辑优先放在 `packages/ui`。
- 与 session runtime 强绑定的写操作留在 app renderer。

## 跨端 UI 示例

```text
+---------------------------------------------------+
| AppShell                                          |
| +-------------+ +-------------+ +---------------+ |
| | Workspace   | | SessionList | | ChatPage      | |
| | Sidebar     | |             | |               | |
| +-------------+ +-------------+ | ChatDisplay   | |
|                                 | Composer      | |
|                                 +---------------+ |
+---------------------------------------------------+
          |
          v
 window.electronAPI
          |
          v
 local or remote WS RPC
```

## 新增 UI 功能检查清单

1. 判断功能是否应跨 Electron/Web/Viewer 复用。
2. 纯展示组件优先放 `packages/ui`。
3. 需要 session runtime 的能力通过 `electronAPI`。
4. 新增 API 时同步 `channel-map.ts`、`shared/types.ts`、Web override。
5. 确认移动端/compact mode 行为。
6. 确认远程工作区下本地路径、打开文件、系统 shell 行为。
7. 对 streaming UI，优先更新单 session atom，避免全局列表重渲染。

## 常见风险

- 在 renderer 组件中直接使用 Node/Electron API，导致 Web UI build 失败。
- 在共享 UI 包中引入 app-specific 状态。
- 新增 `electronAPI` 后忘记 Web UI override。
- 把本地文件路径传给远程 server。
- 流式事件更新了大对象，导致输入框或列表卡顿。
