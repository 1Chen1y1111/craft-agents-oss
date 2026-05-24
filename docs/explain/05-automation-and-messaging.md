# 05. 自动化与消息平台：如何把 Agent 从聊天框变成业务流程

## 给产品经理的一句话

自动化让 Craft Agents 不再只是“用户打开应用后问一句”，而是可以在业务事件发生时自动创建会话、发送提示词、调用 webhook，并通过 Telegram、WhatsApp、Lark 等消息平台把结果和审批推回用户正在工作的地方。

```mermaid
flowchart LR
  A["业务事件发生"] --> B["AutomationSystem 判断是否匹配"]
  B --> C{"匹配成功吗"}
  C -->|否| D["忽略事件"]
  C -->|是| E["执行 actions"]
  E --> F["创建或推进 Session"]
  E --> G["调用 Webhook"]
  F --> H["Agent 执行业务任务"]
  H --> I["Session 事件流"]
  I --> J["桌面/Web UI 展示"]
  I --> K["消息平台同步"]
  K --> L["用户在聊天软件中查看、回复、审批"]
  L --> F
```

## 为什么这是一个重要业务拐点

普通 Agent 产品的使用方式是“人主动找 Agent”。自动化和消息平台让产品变成“业务事件主动找 Agent，再由 Agent 找人确认”。

```mermaid
flowchart TD
  ROOT["Agent 使用方式升级"] --> A["阶段 1：人工提问"]
  ROOT --> B["阶段 2：人工提问 + 外部工具"]
  ROOT --> C["阶段 3：事件触发 Agent"]
  ROOT --> D["阶段 4：Agent 在消息平台内协作"]
  ROOT --> E["阶段 5：流程沉淀为组织能力"]

  A --> A1["用户想到问题才打开应用"]
  B --> B1["Agent 能操作真实系统"]
  C --> C1["标签变化、定时任务、工具事件都能触发"]
  D --> D1["审批、回复、查看结果在聊天软件完成"]
  E --> E1["团队复用固定工作流"]
```

## 自动化系统的核心对象

```mermaid
flowchart TD
  AS["AutomationSystem"] --> CFG["automations.json"]
  AS --> BUS["WorkspaceEventBus"]
  AS --> MATCH["Matcher"]
  AS --> COND["Conditions"]
  AS --> ACT["Actions"]
  AS --> HIST["automations-history.jsonl"]

  MATCH --> M1["事件类型"]
  MATCH --> M2["正则 matcher"]
  MATCH --> M3["cron 定时"]
  MATCH --> M4["启用/禁用"]

  COND --> C1["时间条件"]
  COND --> C2["状态条件"]
  COND --> C3["and/or/not 组合"]

  ACT --> A1["prompt action"]
  ACT --> A2["webhook action"]
```

面向产品经理，可以这样理解：

| 对象 | 业务含义 | 例子 |
| --- | --- | --- |
| Event | 发生了一件事 | 会话状态变为 Needs Review |
| Matcher | 要不要响应这件事 | 只匹配带有 customer 标签的会话 |
| Condition | 响应前再做一层判断 | 只在工作日 9:00 到 18:00 触发 |
| Action | 匹配后做什么 | 创建 Agent 会话或调用 webhook |
| History | 自动化执行记录 | 用于排错、审计、运营分析 |

## 自动化事件类型

系统里事件大致分为 App 事件和 Agent 事件。

```mermaid
flowchart TD
  E["AutomationEvent"] --> APP["App 事件"]
  E --> AG["Agent 事件"]

  APP --> A1["LabelAdd 添加标签"]
  APP --> A2["LabelRemove 移除标签"]
  APP --> A3["LabelConfigChange 标签配置变化"]
  APP --> A4["PermissionModeChange 权限模式变化"]
  APP --> A5["FlagChange 置顶/重要变化"]
  APP --> A6["SessionStatusChange 会话状态变化"]
  APP --> A7["SchedulerTick 定时器触发"]

  AG --> G1["SessionStart 会话开始"]
  AG --> G2["UserPromptSubmit 用户提交 prompt"]
  AG --> G3["PreToolUse 工具调用前"]
  AG --> G4["PostToolUse 工具调用后"]
  AG --> G5["PostToolUseFailure 工具调用失败"]
  AG --> G6["PermissionRequest 权限请求"]
  AG --> G7["SessionEnd 会话结束"]
  AG --> G8["Notification 通知"]
  AG --> G9["PreCompact 压缩上下文前"]
```

产品上最值得关注的事件不是所有事件，而是能表达真实业务节点的事件：

```mermaid
flowchart TD
  ROOT["高价值触发点"] --> A["状态变化：任务进入评审、完成、取消"]
  ROOT --> B["标签变化：客户、项目、优先级、风险"]
  ROOT --> C["权限请求：Agent 想做敏感动作"]
  ROOT --> D["工具失败：外部系统不可用或数据异常"]
  ROOT --> E["定时触发：日报、周报、巡检"]
  ROOT --> F["会话结束：沉淀总结、推送结果"]
```

## 自动化配置如何被执行

```mermaid
sequenceDiagram
  participant SM as SessionManager
  participant AS as AutomationSystem
  participant BUS as WorkspaceEventBus
  participant MH as Matcher/Condition
  participant AH as ActionHandler
  participant HIST as HistoryStore

  SM->>AS: 发出业务事件
  AS->>BUS: publish(event)
  BUS->>MH: 遍历对应事件的 matchers
  MH->>MH: 检查 enabled、matcher、cron、conditions
  alt 不匹配
    MH-->>BUS: skip
  else 匹配
    MH->>AH: 执行 actions
    AH-->>BUS: 返回执行结果
    BUS->>HIST: 写入执行历史
  end
```

## 自动化状态机

```mermaid
stateDiagram-v2
  [*] --> ConfigLoaded: 读取 automations.json
  ConfigLoaded --> Validating: 校验配置
  Validating --> Active: 校验成功
  Validating --> Disabled: 配置不可用或无 automation
  Active --> EventReceived: 收到事件
  EventReceived --> Matching: 评估 matcher
  Matching --> Skipped: 不匹配
  Matching --> ConditionChecking: matcher 命中
  ConditionChecking --> Skipped: 条件不满足
  ConditionChecking --> Executing: 条件满足
  Executing --> HistoryWritten: 写入执行历史
  Executing --> Failed: action 失败
  Failed --> HistoryWritten: 记录失败
  HistoryWritten --> Active: 等待下一次事件
  Skipped --> Active: 等待下一次事件
  Active --> Reloading: 配置变化
  Reloading --> Validating: 重新校验
```

这里的核心产品原则是：自动化失败不能拖垮主流程。用户正在进行的会话要继续，自动化失败应该进入历史和提示，而不是让 Agent 主任务崩掉。

## Matcher、Condition、Action 的决策树

```mermaid
flowchart TD
  A["事件进入自动化系统"] --> B{"事件名是否有配置"}
  B -->|否| Z["结束"]
  B -->|是| C["读取该事件下所有 matchers"]
  C --> D{"matcher 是否 enabled"}
  D -->|否| Z
  D -->|是| E{"是否满足 matcher/cron"}
  E -->|否| Z
  E -->|是| F{"是否配置 conditions"}
  F -->|否| H["执行 actions"]
  F -->|是| G{"conditions 是否全部通过"}
  G -->|否| Z
  G -->|是| H
  H --> I{"action 类型"}
  I -->|prompt| J["准备创建/发送 Agent prompt"]
  I -->|webhook| K["准备发送 HTTP 请求"]
  J --> L["返回 PendingPrompt 给 SessionManager"]
  K --> M["调用外部系统并记录结果"]
```

## 条件系统如何表达业务规则

```mermaid
flowchart TD
  C["Conditions"] --> T["time 时间条件"]
  C --> S["state 状态条件"]
  C --> L["logical 逻辑组合"]

  T --> T1["after: 09:00"]
  T --> T2["before: 18:00"]
  T --> T3["weekday: mon-fri"]
  T --> T4["timezone: Asia/Shanghai"]

  S --> S1["field: sessionStatus"]
  S --> S2["value: Needs Review"]
  S --> S3["from/to: 状态迁移"]
  S --> S4["contains: 标签包含某值"]
  S --> S5["not_value: 排除某值"]

  L --> L1["and：全部满足"]
  L --> L2["or：任意满足"]
  L --> L3["not：取反"]
```

一个产品经理容易理解的例子：

```mermaid
flowchart TD
  A["SessionStatusChange"] --> B{"状态变为 Needs Review"}
  B -->|否| X["不触发"]
  B -->|是| C{"标签包含 customer"}
  C -->|否| X
  C -->|是| D{"当前是工作日白天"}
  D -->|否| E["延后或不触发"]
  D -->|是| F["创建 Agent 会话总结客户风险"]
```

## Prompt Action：把事件变成新会话

Prompt Action 的业务意义是：某个事件发生后，系统自动让 Agent 执行一段预设任务。

```mermaid
sequenceDiagram
  participant BUS as EventBus
  participant PH as PromptHandler
  participant SM as SessionManager
  participant AG as Agent
  participant MSG as MessagingGateway

  BUS->>PH: matcher 命中 prompt action
  PH->>PH: 展开环境变量和 @mentions
  PH-->>SM: PendingPrompt
  SM->>SM: 创建新 session 或选择目标 session
  SM->>AG: 发送自动化 prompt
  AG->>SM: 返回执行事件
  SM->>MSG: 广播 session event
  MSG->>MSG: 如果绑定消息平台则推送
```

Prompt Action 可挖的产品价值：

- 可以把高频 SOP 做成自动执行。
- 可以把会话状态流转变成下一步动作。
- 可以通过标签把不同业务线分流。
- 可以用 `@source` 和 `@skill` 让自动任务带上特定能力。

## Webhook Action：把 Agent 事件告诉外部系统

Webhook Action 的业务意义是：Craft Agents 不只接收外部事件，也能把内部事件推给外部系统。

```mermaid
sequenceDiagram
  participant BUS as EventBus
  participant WH as WebhookHandler
  participant EXT as 外部系统
  participant HIST as HistoryStore

  BUS->>WH: matcher 命中 webhook action
  WH->>WH: 展开 URL、headers、body
  WH->>EXT: HTTP GET/POST/PUT/PATCH/DELETE
  EXT-->>WH: 返回状态码和响应
  WH->>HIST: 记录 success、statusCode、duration、attempts
```

典型场景：

```mermaid
flowchart TD
  ROOT["Webhook 适合的业务场景"] --> A["把完成状态同步到项目管理系统"]
  ROOT --> B["把 Agent 失败事件发送到告警系统"]
  ROOT --> C["把客户总结推到 CRM"]
  ROOT --> D["把安全敏感操作写入审计系统"]
  ROOT --> E["触发企业内部工作流平台"]
```

## 定时任务：从被动响应到主动巡检

```mermaid
flowchart TD
  A["SchedulerTick"] --> B["读取 cron matchers"]
  B --> C{"当前时间是否命中 cron"}
  C -->|否| D["等待下一次 tick"]
  C -->|是| E{"timezone 是否匹配"}
  E -->|否| D
  E -->|是| F["执行 actions"]
  F --> G["生成日报/周报/巡检会话"]
```

定时任务非常适合产品化成模板：

```mermaid
flowchart LR
  A["日报模板"] --> B["每天 18:00 汇总今天完成和阻塞"]
  C["周报模板"] --> D["每周五汇总项目进展"]
  E["风险巡检"] --> F["每天早上检查失败任务和待审批"]
  G["客户跟进"] --> H["每周一检查关键客户邮件和工单"]
```

## 消息平台总体架构

消息平台是另一个入口。它既可以接收用户在外部聊天软件里的消息，也可以把会话执行过程发出去。

```mermaid
flowchart TD
  subgraph Workspace["每个 Workspace"]
    REG["MessagingGatewayRegistry"] --> GW["MessagingGateway"]
    GW --> BS["BindingStore"]
    GW --> PS["PendingSendersStore"]
    GW --> RT["Router"]
    GW --> CMD["Commands"]
    GW --> REN["Renderer"]
    GW --> PTR["PlanTokenRegistry"]
  end

  GW --> TG["Telegram Adapter"]
  GW --> WA["WhatsApp Adapter"]
  GW --> LK["Lark Adapter"]

  RT --> SM["SessionManager"]
  SM --> REN
  REN --> TG
  REN --> WA
  REN --> LK
```

## 消息平台中的核心对象

```mermaid
flowchart TD
  M["Messaging"] --> P["Platform 平台"]
  M --> C["Channel/Chat 外部聊天"]
  M --> B["Binding 绑定"]
  M --> S["Session 会话"]
  M --> O["Owner 拥有者"]
  M --> A["Access Mode 访问模式"]
  M --> R["Response Mode 响应模式"]

  B --> B1["把某个聊天线程绑定到某个 session"]
  A --> A1["open"]
  A --> A2["owner-only"]
  A --> A3["allow-list"]
  A --> A4["inherit"]
  R --> R1["streaming"]
  R --> R2["progress"]
  R --> R3["final_only"]
```

## 外部消息如何进入会话

```mermaid
sequenceDiagram
  participant U as 外部聊天用户
  participant AD as PlatformAdapter
  participant AC as AccessControl
  participant RT as Router/Commands
  participant GW as MessagingGateway
  participant SM as SessionManager

  U->>AD: 在 Telegram/WhatsApp/Lark 发消息
  AD->>RT: IncomingMessage
  RT->>AC: 校验发送者是否有权访问
  alt 无权限
    AC-->>AD: 发送拒绝说明或静默丢弃
    AC->>GW: 记录 pending sender
  else 有权限
    RT->>GW: 查找 channel/thread binding
    alt 已绑定 session
      GW->>SM: sendMessage(sessionId, text, attachments)
    else 是命令
      GW->>RT: 执行 /new、/bind、/pair 等命令
    else 未绑定
      GW-->>AD: 提示先绑定或创建 session
    end
  end
```

## Session 事件如何推回消息平台

```mermaid
sequenceDiagram
  participant SM as SessionManager
  participant GW as MessagingGateway
  participant REN as Renderer
  participant AD as PlatformAdapter
  participant U as 外部聊天用户

  SM->>GW: session event
  GW->>GW: 找到绑定的 chats/topics
  GW->>REN: 渲染为平台消息
  REN->>REN: 根据 responseMode 决定更新方式
  alt progress
    REN->>AD: 编辑同一条进度消息
  else final_only
    REN->>AD: 完成后发送最终结果
  else streaming
    REN->>AD: 流式发送多条消息
  end
  AD-->>U: 用户在聊天软件看到结果
```

## Plan 和 Permission 的按钮审批

消息平台里最有产品价值的不是“转发消息”，而是让用户在外部聊天软件里完成关键审批。

```mermaid
flowchart TD
  A["Agent 生成 Plan 或 Permission 请求"] --> B["Renderer 生成按钮消息"]
  B --> C["PlanTokenRegistry 或 permission requestId 记录凭证"]
  C --> D["平台发送 Approve/Deny 按钮"]
  D --> E{"用户点击按钮"}
  E -->|批准| F["Gateway 校验 token、sender、binding"]
  E -->|拒绝| G["Gateway 校验 requestId、sender、binding"]
  F --> H["通知 SessionManager 继续执行"]
  G --> I["通知 SessionManager 拒绝或中断"]
  H --> J["清除或失效按钮"]
  I --> J
```

这个机制让长任务可以脱离桌面 App 持续推进：

```mermaid
journey
  title 用户在通勤中审批远程 Agent 任务
  section 任务执行
    Agent 远程运行: 4: Agent
    发现需要批准计划: 3: Agent
  section 外部触达
    Telegram 收到计划摘要: 5: 用户
    用户点击批准按钮: 5: 用户
  section 任务继续
    Agent 继续执行: 4: Agent
    最终结果推送到聊天: 5: 用户
```

## 访问控制是消息平台的生命线

```mermaid
flowchart TD
  A["收到外部刺激"] --> B{"sender 是否是 bot"}
  B -->|是| X["静默丢弃，避免机器人互刷"]
  B -->|否| C{"是绑定前命令吗"}
  C -->|是| D{"workspace accessMode"}
  D -->|open| OK["允许"]
  D -->|owner-only| E{"sender 是否 owner"}
  E -->|是| OK
  E -->|否| DENY["拒绝并记录 pending sender"]

  C -->|否，路由到已有 binding| F{"binding accessMode"}
  F -->|open| OK
  F -->|allow-list| G{"sender 在 allowedSenderIds 吗"}
  G -->|是| OK
  G -->|否| DENY
  F -->|inherit| D
```

从业务上看，消息访问控制和 Agent 工具权限是两条线：

```mermaid
flowchart LR
  A["消息访问控制"] --> A1["谁能从聊天软件控制会话"]
  B["Agent 工具权限"] --> B1["Agent 能不能执行写操作或敏感工具"]

  A1 --> C["防止外部人乱发指令"]
  B1 --> D["防止 Agent 未经批准修改系统"]
```

## WhatsApp Worker 为什么独立

WhatsApp 依赖更复杂，系统用独立 worker 子进程隔离它。

```mermaid
sequenceDiagram
  participant AD as WhatsAppAdapter
  participant WK as WhatsApp Worker
  participant WA as WhatsApp Network

  AD->>WK: WorkerCommand NDJSON
  WK->>WA: 连接、登录、收发消息
  WA-->>WK: incoming、QR、pairing、send result
  WK-->>AD: WorkerEvent NDJSON
  AD->>AD: 转成 Gateway 事件
```

业务价值：

- WhatsApp 连接异常不影响主应用。
- 登录、重连、附件下载可以隔离处理。
- 子进程协议稳定后，平台适配可独立迭代。

## Telegram Topic 与自动化的结合

自动化配置可以指定 Telegram forum topic。这样某类自动化生成的会话可以自动进入一个固定讨论区。

```mermaid
flowchart TD
  A["自动化 matcher 命中"] --> B{"配置 telegramTopic 吗"}
  B -->|否| C["创建普通自动化 session"]
  B -->|是| D{"是否已绑定 Telegram supergroup"}
  D -->|否| C
  D -->|是| E{"topic 是否存在"}
  E -->|不存在| F["创建 forum topic"]
  E -->|已存在| G["复用 topic"]
  F --> H["绑定 session 到该 topic"]
  G --> H
  H --> I["后续进度推送到同一 topic"]
```

产品机会：

- 客户自动化进入“客户风险”话题。
- 研发自动化进入“构建失败”话题。
- 销售自动化进入“待跟进线索”话题。

## 典型业务剧本

### 剧本 1：会话进入评审后自动总结

```mermaid
flowchart TD
  A["会话状态变为 Needs Review"] --> B["Automation 匹配 SessionStatusChange"]
  B --> C["条件：标签包含 product"]
  C --> D["Prompt Action：总结本次任务变更、风险、待确认点"]
  D --> E["创建新 Agent 会话"]
  E --> F["结果推送到 Telegram 产品评审 topic"]
  F --> G["PM 在群里继续追问"]
```

### 剧本 2：工具失败后自动告警

```mermaid
flowchart TD
  A["Agent 调用 GitHub 工具失败"] --> B["PostToolUseFailure"]
  B --> C["Automation 匹配失败事件"]
  C --> D["Webhook Action 推送到告警系统"]
  C --> E["Prompt Action 创建排查会话"]
  E --> F["Agent 分析失败原因"]
  F --> G["消息平台通知负责人"]
```

### 剧本 3：每天自动生成团队日报

```mermaid
flowchart TD
  A["每天 18:00 SchedulerTick"] --> B["cron 命中"]
  B --> C["Prompt Action"]
  C --> D["Agent 读取今天完成的 sessions"]
  D --> E["汇总完成项、阻塞项、需要人工审批项"]
  E --> F["发送到团队群"]
```

## 产品指标建议

```mermaid
flowchart TD
  ROOT["自动化与消息指标"] --> A["配置指标"]
  ROOT --> B["触发指标"]
  ROOT --> C["执行指标"]
  ROOT --> D["协作指标"]
  ROOT --> E["安全指标"]

  A --> A1["automation 数量"]
  A --> A2["启用率"]
  A --> A3["模板使用率"]

  B --> B1["事件触发次数"]
  B --> B2["matcher 命中率"]
  B --> B3["condition 拦截率"]

  C --> C1["action 成功率"]
  C --> C2["prompt session 完成率"]
  C --> C3["webhook 成功率和耗时"]

  D --> D1["消息绑定数"]
  D --> D2["外部回复数"]
  D --> D3["按钮审批率"]

  E --> E1["拒绝访问次数"]
  E --> E2["pending sender 数"]
  E --> E3["重复按钮点击拦截数"]
```

## 产品上值得继续深挖的点

```mermaid
flowchart TD
  ROOT["机会点"] --> A["自动化模板市场"]
  ROOT --> B["可视化自动化编辑器"]
  ROOT --> C["自动化运行历史和调试台"]
  ROOT --> D["消息平台审批中心"]
  ROOT --> E["按团队/角色配置访问控制"]
  ROOT --> F["失败自动重试和人工接管"]
  ROOT --> G["自动化效果分析"]

  A --> A1["日报、周报、客户跟进、失败巡检"]
  B --> B1["非技术用户不需要手写 JSON"]
  C --> C1["告诉用户为什么触发或没触发"]
  D --> D1["集中处理 Plan、Permission、Credential 提醒"]
  E --> E1["团队 owner、项目 owner、临时协作者"]
  F --> F1["失败后转为消息提醒或创建排查会话"]
  G --> G1["展示节省时间、完成率、阻塞点"]
```

## 本文结论

自动化和消息平台把 Craft Agents 从“一个会话工具”扩展成“业务流程执行层”。

```mermaid
flowchart LR
  A["事件"] --> B["自动化判断"]
  B --> C["Agent 执行"]
  C --> D["消息平台触达"]
  D --> E["用户审批/回复"]
  E --> C
  C --> F["结果沉淀"]
  F --> G["流程复用"]
```

