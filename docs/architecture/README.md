# 架构细分文档导航

本目录从 `docs/architecture-design.md` 衍生，按维护场景拆分专题。总览文档回答“系统长什么样”，本目录回答“某一层怎么工作、改动时要碰哪些文件、有哪些边界不能越过”。

## 阅读顺序

建议按下面顺序阅读：

1. `runtime-modes.md`：先理解 Electron、本地 server、远程 server、Web UI、CLI 的运行形态。
2. `rpc-transport.md`：理解所有前端入口如何通过统一 WS RPC 进入后端。
3. `session-runtime.md`：理解会话创建、消息发送、持久化和事件广播。
4. `agent-backends-and-tools.md`：理解 Claude/Pi 后端、sources、MCP、session tools 如何接入。
5. `data-storage-and-security.md`：理解用户数据、凭据、安全边界和远程部署约束。
6. `ui-clients.md`：理解 Electron renderer、Web UI、Viewer 和共享 UI 的关系。
7. `automation-and-messaging.md`：理解自动化和外部消息平台如何围绕 session event 扩展。

## 文档列表

| 文档 | 主题 | 适合场景 |
| --- | --- | --- |
| `runtime-modes.md` | 运行模式与进程拓扑 | 排查启动、远程连接、WebUI/CLI 行为 |
| `rpc-transport.md` | WebSocket RPC 与 channel routing | 新增 RPC、排查断线重连、远程工作区路由 |
| `session-runtime.md` | SessionManager 与会话生命周期 | 改 session、消息流、branch、导入导出、状态标签 |
| `agent-backends-and-tools.md` | AgentBackend、Sources、MCP、session tools | 新增模型后端、source、工具、安全工具约束 |
| `data-storage-and-security.md` | 配置、工作区、凭据、安全与可靠性 | 改持久化格式、迁移、权限、远程部署 |
| `ui-clients.md` | Electron/Web/Viewer UI 架构 | 改聊天 UI、跨端 UI、平台能力抽象 |
| `automation-and-messaging.md` | 自动化系统与消息网关 | 改 automations、Telegram/WhatsApp/Lark 集成 |

## 维护约定

- 总览文档只保留系统级视角，细节放到本目录。
- 新增 RPC 时同时更新 `rpc-transport.md` 和相关专题文档。
- 新增后端、source 或 session tool 时更新 `agent-backends-and-tools.md`。
- 改 session 文件格式或配置路径时更新 `data-storage-and-security.md`。
- 改跨端 UI 或 `electronAPI` surface 时更新 `ui-clients.md`。
