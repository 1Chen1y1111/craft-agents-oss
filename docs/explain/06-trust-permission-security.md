# 06. 信任、权限与安全：用户为什么敢把真实任务交给 Agent

## 给产品经理的一句话

Craft Agents 的安全设计不是一个单独的“安全设置页”，而是一整套让用户逐步建立信任的机制：权限模式、计划审批、工具审批、凭据加密、Source guide、消息访问控制、远程连接校验、审计日志一起构成了“可控的 Agent 执行边界”。

```mermaid
flowchart TD
  U["用户信任"] --> A["看得见 Agent 在做什么"]
  U --> B["敏感动作可批准或拒绝"]
  U --> C["凭据不明文放配置里"]
  U --> D["外部聊天不是谁都能控制"]
  U --> E["远程 server 需要 token/TLS"]
  U --> F["失败和高风险动作可追踪"]

  A --> T["透明过程"]
  B --> P["权限边界"]
  C --> S["凭据安全"]
  D --> M["消息访问控制"]
  E --> R["远程安全"]
  F --> G["审计与恢复"]
```

## 信任不是一次授权，而是逐级放权

用户一开始不会直接让 Agent 随便改系统。更自然的路径是：

```mermaid
journey
  title 用户对 Agent 的信任建立过程
  section 初次使用
    只问问题和看解释: 5: 用户
    观察 Agent 的工具调用: 4: 用户
  section 小范围授权
    允许只读 Source: 4: 用户
    对写操作逐次批准: 3: 用户
  section 稳定复用
    对固定流程使用计划审批: 4: 用户
    对低风险任务开启自动执行: 4: 用户
  section 团队治理
    设置消息 owner 和 allow-list: 4: 管理者
    查看历史和失败原因: 4: 管理者
```

## 安全边界总览

```mermaid
flowchart TD
  ROOT["Craft Agents 安全边界"] --> A["Agent 工具权限"]
  ROOT --> B["计划审批"]
  ROOT --> C["凭据存储"]
  ROOT --> D["Source 使用约束"]
  ROOT --> E["消息平台访问控制"]
  ROOT --> F["远程 RPC 鉴权"]
  ROOT --> G["本地/远程路径边界"]
  ROOT --> H["特权命令审批"]
  ROOT --> I["日志脱敏和审计"]

  A --> A1["Explore/Safe"]
  A --> A2["Ask to Edit"]
  A --> A3["Auto/Allow All"]

  C --> C1["credentials.enc"]
  C --> C2["AES-256-GCM"]
  C --> C3["OS 机器标识派生 key"]

  E --> E1["owner-only"]
  E --> E2["allow-list"]
  E --> E3["pending sender"]

  F --> F1["server token"]
  F --> F2["Web UI cookie"]
  F --> F3["拒绝非安全明文远程连接"]
```

## 权限模式的产品含义

```mermaid
flowchart LR
  SAFE["Explore/Safe"] --> SAFE1["适合探索和只读分析"]
  ASK["Ask to Edit"] --> ASK1["适合默认工作模式"]
  AUTO["Auto/Allow All"] --> AUTO1["适合高度信任的低风险流程"]

  SAFE1 --> C1["阻止或限制写操作"]
  ASK1 --> C2["敏感操作先问用户"]
  AUTO1 --> C3["减少打断，提高执行效率"]
```

| 模式 | 用户心理 | 产品价值 | 风险 |
| --- | --- | --- | --- |
| Explore/Safe | 我先看看它能分析什么 | 降低初次使用门槛 | 任务完成度可能低 |
| Ask to Edit | 重要动作先问我 | 平衡效率和控制 | 频繁审批会打断 |
| Auto/Allow All | 这个流程我已经信任 | 适合自动化和批量任务 | 需要更强的范围控制 |

## 工具执行权限流

```mermaid
sequenceDiagram
  participant AG as Agent
  participant SM as SessionManager
  participant PM as PermissionManager
  participant UI as UI/消息平台
  participant U as 用户
  participant TOOL as Tool

  AG->>SM: 请求执行工具
  SM->>PM: 检查工具风险和 permission mode
  alt 安全模式阻止
    PM-->>SM: blocked
    SM-->>AG: 告知不可执行
  else 需要询问
    PM-->>SM: permission_request
    SM->>UI: 展示工具、原因、影响
    UI->>U: 等待批准或拒绝
    U-->>UI: 决策
    UI-->>SM: permission response
    alt 批准
      SM->>TOOL: 执行工具
    else 拒绝
      SM-->>AG: 拒绝结果
    end
  else 自动允许
    SM->>TOOL: 执行工具
  end
  TOOL-->>SM: 工具结果
  SM-->>AG: 返回结果
```

## 计划审批是“执行前对齐”

工具审批解决的是“这一刀能不能下”，计划审批解决的是“整件事是不是要这么做”。

```mermaid
flowchart TD
  A["用户给出复杂任务"] --> B["Agent 先生成计划"]
  B --> C["计划写入 session plans"]
  C --> D["SubmitPlan 展示给用户"]
  D --> E{"用户是否认可"}
  E -->|认可| F["Agent 按计划执行"]
  E -->|想调整| G["用户补充约束"]
  G --> H["Agent 重新计划"]
  E -->|不认可| I["停止执行或转为解释"]
```

计划审批适合：

- 多步骤任务。
- 会修改多个文件或外部系统的任务。
- 团队协作中需要让别人理解的任务。
- 自动化生成的高风险任务。

## Source Guide 是柔性的安全控制

Source guide 不是代码层面的硬权限，但它是一种非常适合产品化的“业务边界说明”。

```mermaid
flowchart TD
  A["Agent 想用 Source"] --> B{"是否读过 guide.md"}
  B -->|否| C["提示 Agent 先阅读 guide"]
  C --> D["Agent 读取使用说明"]
  D --> E["理解适用范围、限制、输出解释"]
  B -->|是| E
  E --> F["允许调用 Source 工具"]
```

产品价值：

- 团队可以用自然语言写规则。
- 不需要每条边界都开发成代码。
- 让 Agent 在调用前知道“这个系统怎么用才对”。
- 适合沉淀业务 SOP 和外部 API 限制。

## 凭据分层：普通配置和敏感值分开

```mermaid
flowchart TD
  CFG["config.json / source config.json"] --> META["保存非敏感配置"]
  META --> M1["Source 名称"]
  META --> M2["API endpoint"]
  META --> M3["认证方式类型"]
  META --> M4["UI 偏好和模型元数据"]

  CREDS["credentials.enc"] --> SEC["保存敏感值"]
  SEC --> S1["LLM API Key"]
  SEC --> S2["OAuth access token"]
  SEC --> S3["refresh token"]
  SEC --> S4["source bearer/api key/basic"]
  SEC --> S5["messaging platform token"]
```

这样做的业务意义是：用户可以备份、迁移、查看配置，但不应该在普通配置文件里暴露 secret。

## 凭据类型地图

```mermaid
flowchart TD
  C["CredentialType"] --> G["Global"]
  C --> L["LLM Connection"]
  C --> W["Workspace"]
  C --> S["Source"]
  C --> M["Messaging"]

  G --> G1["anthropic_api_key"]
  G --> G2["claude_oauth"]

  L --> L1["llm_api_key"]
  L --> L2["llm_oauth"]
  L --> L3["llm_iam"]
  L --> L4["llm_service_account"]

  W --> W1["workspace_oauth"]

  S --> S1["source_oauth"]
  S --> S2["source_bearer"]
  S --> S3["source_apikey"]
  S --> S4["source_basic"]

  M --> M1["messaging_bearer"]
```

## OAuth 授权流程

```mermaid
sequenceDiagram
  participant U as 用户
  participant UI as Electron/Web
  participant CB as Callback Server
  participant OAuth as OAuth Provider
  participant SV as Server
  participant CM as CredentialManager

  U->>UI: 点击授权 Source 或 LLM
  UI->>SV: oauth:start
  SV-->>UI: 返回授权 URL、flowId、state
  UI->>CB: 启动本地 callback server
  UI->>OAuth: 打开浏览器授权
  OAuth-->>CB: redirect code + state
  CB-->>UI: 捕获 code
  UI->>SV: oauth:complete(flowId, code, state)
  SV->>OAuth: 交换 access token
  OAuth-->>SV: token/refresh token/expiresAt
  SV->>CM: 加密保存凭据
  SV-->>UI: 授权成功
```

用户看见的是“跳浏览器授权再回来”，系统内部要保证：

- state 防止串流程。
- token 不进入普通配置。
- refresh token 用于续期。
- 失败后 Source 状态要能解释清楚。

## Token 过期与刷新

```mermaid
stateDiagram-v2
  [*] --> Valid: 凭据可用
  Valid --> Expiring: 接近过期
  Expiring --> Refreshing: 尝试刷新
  Refreshing --> Valid: 刷新成功
  Refreshing --> NeedsAuth: 刷新失败
  NeedsAuth --> Authenticating: 用户重新授权
  Authenticating --> Valid: 授权成功
  Authenticating --> Unavailable: 用户取消或失败
  Unavailable --> NeedsAuth: 用户稍后重试
```

这里最重要的产品体验是：不要只告诉用户“失败”，要告诉用户“哪个 Source 的认证过期了、需要做什么”。

## 远程 RPC 安全

```mermaid
flowchart TD
  A["Client 连接 server"] --> B{"连接目标是什么"}
  B -->|localhost| C["允许本地开发或本机连接"]
  B -->|非 localhost ws 明文| D["默认拒绝或警告"]
  B -->|wss/TLS| E["继续 handshake"]
  C --> E
  E --> F{"token/cookie 是否有效"}
  F -->|否| G["拒绝连接"]
  F -->|是| H["建立 WebSocket RPC"]
  H --> I["按 channel routing 执行业务请求"]
```

产品上的解释方式：

- 本机连接可以更宽松。
- 远程 server 必须有 token。
- 公网或局域网暴露时应该用 TLS。
- Electron 薄客户端不应该随便连非安全明文远程地址。

## 本地路径与远程路径不能混淆

```mermaid
flowchart TD
  U["用户在客户端选择路径"] --> A{"当前工作区在哪里"}
  A -->|本地工作区| B["路径属于用户电脑"]
  A -->|远程工作区| C["路径属于远程 server"]
  B --> D["本地工具可读取"]
  C --> E["远程 runtime 才能读取"]
  D --> F["执行结果回传 UI"]
  E --> F
```

这是远程产品最容易让用户困惑的地方。文案和 UI 需要清晰区分“你电脑上的文件”和“远程机器上的文件”。

## 消息访问控制与工具权限是两层门

```mermaid
flowchart TD
  A["外部聊天用户发消息"] --> B{"消息访问控制"}
  B -->|无权| X["拒绝或记录 pending sender"]
  B -->|有权| C["消息进入 Session"]
  C --> D["Agent 理解指令"]
  D --> E{"Agent 工具权限"}
  E -->|无权| Y["阻止或请求批准"]
  E -->|有权| F["执行工具"]
```

对用户解释：

- 消息访问控制回答“谁能进来发指令”。
- Agent 工具权限回答“进来之后 Agent 能做多大动作”。

## 消息平台访问控制矩阵

```mermaid
flowchart TD
  A["Incoming Message/Button"] --> B{"senderIsBot"}
  B -->|是| DROP["静默丢弃"]
  B -->|否| C{"是否已有 binding"}
  C -->|否| D{"workspace platform accessMode"}
  D -->|open| ALLOW["允许执行绑定前命令"]
  D -->|owner-only| E{"sender 是 owner 吗"}
  E -->|是| ALLOW
  E -->|否| PENDING["拒绝并记录 pending sender"]

  C -->|是| F{"binding accessMode"}
  F -->|open| ROUTE["路由到 session"]
  F -->|allow-list| G{"在 allow-list 吗"}
  G -->|是| ROUTE
  G -->|否| PENDING
  F -->|inherit| D
```

## 特权命令审批

某些命令即使用户授权了 Agent，也需要更强约束。系统中存在特权执行 broker，用于给高权限命令做审批、过期和审计。

```mermaid
sequenceDiagram
  participant AG as Agent
  participant PB as PrivilegedExecutionBroker
  participant U as 用户
  participant EX as Execution Path
  participant LOG as Audit Log

  AG->>PB: 创建特权执行请求
  PB->>PB: 校验命令是否符合策略
  PB->>LOG: 写入 request_created
  PB-->>U: 展示 reason、impact、TTL
  U->>PB: approve/deny
  PB->>PB: 校验 requestId、hash、是否过期
  alt 通过且批准
    PB->>LOG: 写入 approved
    PB->>EX: 允许进入执行路径
  else 拒绝、过期或策略不允许
    PB->>LOG: 写入 denied/expired/blocked
    PB-->>AG: 返回失败原因
  end
```

这里有一个很好的产品点：高风险操作不只问“是否批准”，还要展示“原因、影响、有效时间、命令摘要”。

## 数据存储边界

```mermaid
flowchart TD
  ROOT["~/.craft-agent 或 CRAFT_CONFIG_DIR"] --> CFG["全局 config.json"]
  ROOT --> CREDS["credentials.enc"]
  ROOT --> LOGS["logs"]
  ROOT --> WS["workspaces"]

  WS --> W1["workspace config.json"]
  WS --> SESS["sessions"]
  WS --> SRC["sources"]
  WS --> SK["skills"]
  WS --> LAB["labels/statuses"]
  WS --> AUTO["automations.json/history"]
  WS --> MSG["messaging"]

  SESS --> SJ["session.jsonl"]
  SESS --> ATT["attachments"]
  SESS --> PLANS["plans"]
  SESS --> DATA["data/downloads/long_responses"]

  SRC --> SCFG["source config.json"]
  SRC --> GUIDE["guide.md"]
  SRC --> CACHE[".credential-cache.json"]
```

关键点：

- `session.jsonl` 便于追加和恢复。
- `credentials.enc` 是权威敏感凭据存储。
- Source 下的 `.credential-cache.json` 是给外部子进程使用的临时缓存，不是权威来源。
- logs 和 history 应该适合排错，但不能泄漏 token。

## 风险点和产品缓解策略

```mermaid
flowchart TD
  ROOT["主要风险"] --> A["Agent 误操作"]
  ROOT --> B["凭据泄露"]
  ROOT --> C["外部聊天被陌生人控制"]
  ROOT --> D["远程 server 暴露"]
  ROOT --> E["用户看不懂审批影响"]
  ROOT --> F["自动化误触发"]

  A --> A1["权限模式、计划审批、工具审批"]
  B --> B1["加密存储、日志脱敏、普通配置不放 secret"]
  C --> C1["owner-only、allow-list、pending sender"]
  D --> D1["token、TLS、拒绝非安全明文远程连接"]
  E --> E1["审批卡展示工具名、原因、影响、范围"]
  F --> F1["enabled 开关、conditions、历史记录、调试台"]
```

## 审批卡应该讲清楚什么

```mermaid
flowchart TD
  CARD["审批卡内容"] --> A["要执行什么"]
  CARD --> B["为什么要执行"]
  CARD --> C["会影响哪里"]
  CARD --> D["是否可撤销"]
  CARD --> E["这次授权持续多久"]
  CARD --> F["批准后下一步是什么"]
  CARD --> G["拒绝后会怎样"]
```

产品文案不应该只写“Allow”。更好的结构是：

| 字段 | 目的 |
| --- | --- |
| 工具名 | 让用户知道动作类型 |
| 目标对象 | 让用户知道影响范围 |
| Agent 理由 | 让用户知道为什么需要 |
| 预计影响 | 让用户评估风险 |
| 操作按钮 | 批准、拒绝、查看详情 |

## 失败恢复流程

```mermaid
flowchart TD
  A["执行失败"] --> B{"失败类型"}
  B -->|权限拒绝| C["提示用户可修改任务或批准权限"]
  B -->|凭据过期| D["提示重新授权 Source"]
  B -->|外部系统失败| E["建议重试或创建排查会话"]
  B -->|远程断线| F["重连并刷新 session 状态"]
  B -->|自动化失败| G["写入 history，主会话继续"]
  C --> H["用户决定下一步"]
  D --> H
  E --> H
  F --> H
  G --> H
```

## 信任相关指标

```mermaid
flowchart TD
  ROOT["信任指标"] --> A["审批行为"]
  ROOT --> B["权限模式"]
  ROOT --> C["凭据健康"]
  ROOT --> D["远程连接"]
  ROOT --> E["消息访问"]
  ROOT --> F["失败恢复"]

  A --> A1["permission 批准率"]
  A --> A2["permission 拒绝率"]
  A --> A3["plan 修改率"]
  A --> A4["审批等待时长"]

  B --> B1["默认模式分布"]
  B --> B2["用户切到 Auto 的比例"]
  B --> B3["Auto 后失败率"]

  C --> C1["认证成功率"]
  C --> C2["token 刷新失败率"]
  C --> C3["credential health issue 数"]

  D --> D1["token handshake 失败数"]
  D --> D2["stale reconnect 次数"]
  D --> D3["远程任务完成率"]

  E --> E1["拒绝访问次数"]
  E --> E2["pending sender 转 owner/allow-list 比率"]
  E --> E3["按钮重复点击拦截数"]

  F --> F1["失败后重试率"]
  F --> F2["失败后继续完成率"]
```

## 本文结论

信任产品的核心不是“把所有风险藏起来”，而是把风险转成用户能理解、能决策、能追踪的流程。

```mermaid
flowchart LR
  A["透明"] --> B["可批准"]
  B --> C["可撤回或拒绝"]
  C --> D["可恢复"]
  D --> E["可审计"]
  E --> F["用户敢授权更多"]
  F --> A
```

