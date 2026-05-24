# 04. 远程、多端与协作：为什么不只是一个桌面 App

## 产品视角

Craft Agents 支持 Electron 桌面端、Web UI、CLI、Viewer、远程 server。它的业务意义是：用户可以在不同场景下用同一套会话能力，而不是被固定在一个本地聊天窗口里。

```mermaid
flowchart TD
  CORE["同一套会话运行时"] --> DESK["Electron 桌面"]
  CORE --> WEB["Web UI"]
  CORE --> CLI["CLI"]
  CORE --> VIEW["Viewer"]
  CORE --> MSG["消息平台"]

  DESK --> D1["日常交互和本地能力"]
  WEB --> W1["浏览器访问远程会话"]
  CLI --> C1["脚本化和 CI"]
  VIEW --> V1["只读分享和传播"]
  MSG --> M1["在聊天软件里推进任务"]
```

## 本地模式与远程模式

```mermaid
flowchart LR
  subgraph Local["本地模式"]
    E1["Electron UI"] --> L1["本地 WsRpcServer"]
    L1 --> L2["本地 SessionManager"]
    L2 --> L3["本机文件/工具/Agent"]
  end

  subgraph Remote["远程模式"]
    E2["Electron 薄客户端"] --> R1["远程 WsRpcServer"]
    W2["Web UI"] --> R1
    CLI2["CLI"] --> R1
    R1 --> R2["远程 SessionManager"]
    R2 --> R3["远程机器文件/工具/Agent"]
  end
```

本地适合：

- 需要本机文件和桌面交互。
- 个人日常使用。
- 本机环境就是任务执行环境。

远程适合：

- 长时间任务。
- 多设备访问。
- 更强机器执行。
- 团队共享 server。
- Docker/服务器部署。

## 混合工作区路由

Electron 本地模式下，一个用户可以同时拥有本地工作区和远程工作区。系统通过 channel routing 判断请求应该去哪。

```mermaid
flowchart TD
  U["用户在 Electron 操作"] --> API["window.electronAPI"]
  API --> R{"这个能力属于哪里"}
  R -->|本地窗口/文件/菜单/更新| L["本地 Electron server"]
  R -->|会话/Source/Skill/状态| W{"当前工作区归属"}
  W -->|本地工作区| L
  W -->|远程工作区| REM["远程 server"]

  L --> OS["本机 OS 能力"]
  REM --> RS["远程 Session Runtime"]
```

对产品经理来说，这意味着一个重要原则：

> “打开窗口、选择本地文件”这类事情发生在用户电脑；“执行远程任务、读取远程工作区文件”发生在远程 server。

## 多端入口的业务定位

```mermaid
flowchart TD
  ROOT["用户入口"] --> E["Electron"]
  ROOT --> W["Web UI"]
  ROOT --> C["CLI"]
  ROOT --> V["Viewer"]
  ROOT --> M["消息平台"]

  E --> E1["高频主工作台"]
  E --> E2["本地文件和浏览器工具"]
  E --> E3["完整设置能力"]

  W --> W1["轻量访问"]
  W --> W2["远程 server 管理"]
  W --> W3["移动/非安装环境"]

  C --> C1["自动化脚本"]
  C --> C2["CI/CD"]
  C --> C3["一次性任务"]

  V --> V1["成果展示"]
  V --> V2["分享 transcript"]
  V --> V3["团队传播"]

  M --> M1["异步提醒"]
  M --> M2["审批按钮"]
  M --> M3["在工作群里继续任务"]
```

## 远程 server 的启动和访问

```mermaid
sequenceDiagram
  participant Admin as 管理者
  participant Server as 远程 Server
  participant Client as Electron/Web/CLI
  participant Runtime as 会话运行时

  Admin->>Server: 设置 CRAFT_SERVER_TOKEN
  Admin->>Server: 启动 headless server
  Server-->>Admin: 输出 CRAFT_SERVER_URL
  Admin->>Client: 配置 URL 和 token
  Client->>Server: WebSocket handshake
  Server->>Server: 校验 token 或 cookie
  Server-->>Client: 返回 clientId
  Client->>Runtime: 发起会话操作
```

## Web UI 的业务意义

Web UI 不是单独产品线，而是远程 server 的浏览器入口。

```mermaid
flowchart LR
  A["用户打开浏览器"] --> B["登录 Web UI"]
  B --> C["获取 WS 配置"]
  C --> D["连接远程 server"]
  D --> E["复用 Electron App UI"]
  E --> F["管理远程会话"]
```

价值：

- 不安装桌面端也能看远程任务。
- 适合临时访问。
- 适合移动设备和受限环境。
- 可以作为团队 server 的入口。

限制：

- 本地文件打开、桌面通知、系统菜单等能力需要降级。
- 文件选择和 OAuth 行为需要浏览器替代方案。

## CLI 的业务意义

CLI 让 Craft Agents 从“交互工具”变成“可脚本化能力”。

```mermaid
flowchart TD
  A["脚本/CI 调用 craft-cli"] --> B{"是否已有 server"}
  B -->|有| C["连接远程或本地 server"]
  B -->|没有| D["自动拉起临时 server"]
  C --> E["创建或选择 workspace"]
  D --> E
  E --> F["创建 session"]
  F --> G["发送 prompt"]
  G --> H["流式输出结果"]
  H --> I["清理或保留 session"]
```

适合场景：

- CI 中生成报告。
- 批量检查项目。
- 每天定时跑一次分析。
- 与其他系统脚本串联。

## Viewer 与分享

Viewer 是只读会话展示，不参与执行。

```mermaid
flowchart LR
  A["完成一个有价值会话"] --> B["导出或分享"]
  B --> C["Viewer 读取 session"]
  C --> D["展示文本、工具、代码、diff、附件"]
  D --> E["团队成员理解过程"]
  E --> F["产生复用或反馈"]
```

产品价值：

- 把 Agent 执行过程变成可传播案例。
- 帮团队理解“结果是怎么来的”。
- 降低新用户学习成本。
- 可以作为营销/支持/内部知识库材料。

## 会话转移

```mermaid
flowchart TD
  A["源工作区会话"] --> B["导出 Session Bundle"]
  B --> C["包含消息、附件、元数据"]
  C --> D{"Bundle 大小"}
  D -->|小| E["直接传输"]
  D -->|大| F["分块传输"]
  E --> G["目标工作区导入"]
  F --> G
  G --> H["新 server 可继续会话"]
```

业务含义：

- 用户可以把本地探索转移到远程执行。
- 团队可以迁移重要会话。
- 长任务不被单台电脑限制。

## 协作模式

```mermaid
flowchart TD
  ROOT["协作方式"] --> A["同一 workspace 多会话"]
  ROOT --> B["分享 Viewer"]
  ROOT --> C["远程 server 多端访问"]
  ROOT --> D["消息平台绑定"]
  ROOT --> E["会话转移"]

  A --> A1["任务分组"]
  B --> B1["只读传播"]
  C --> C1["多设备继续"]
  D --> D1["群里审批/回复"]
  E --> E1["跨环境迁移"]
```

## 产品经理值得挖的点

| 方向 | 关键问题 | 产品机会 |
| --- | --- | --- |
| 远程 server | 用户为什么需要远程？ | 长任务、团队 server、云托管 |
| Web UI | 哪些功能必须浏览器可用？ | 轻量管理台、移动端审批 |
| CLI | 哪些任务适合脚本化？ | 模板命令、CI 集成 |
| Viewer | 分享后谁会看？ | 团队案例库、可公开分享页 |
| 会话迁移 | 用户什么时候想转移？ | 本地到远程一键迁移 |
| 多端一致性 | 不同入口体验是否一致？ | 统一导航、状态同步 |

## 本文结论

```mermaid
flowchart LR
  A["桌面端负责深度操作"] --> B["远程 server 负责持续执行"]
  B --> C["Web UI 负责轻量访问"]
  C --> D["CLI 负责自动化调用"]
  D --> E["Viewer 负责传播结果"]
  E --> F["消息平台负责异步协作"]
```

