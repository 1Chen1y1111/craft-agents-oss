# 02. 会话生命周期：一次任务从开始到完成怎么走

## 为什么会话是核心业务对象

在 Craft Agents 里，Session 不只是聊天记录。它更像一个“任务容器”：

- 记录用户目标。
- 记录 Agent 的思考和执行过程。
- 记录工具调用和结果。
- 管理权限、计划、凭据请求。
- 保存附件、生成文件、长响应、下载产物。
- 承载标签、状态、归档、未读等协作信息。

```mermaid
flowchart TD
  S["Session 会话"] --> M["消息历史"]
  S --> T["工具执行记录"]
  S --> P["计划 Plan"]
  S --> A["附件 Attachments"]
  S --> D["数据产物 Data/Downloads"]
  S --> G["权限状态"]
  S --> L["标签 Labels"]
  S --> ST["业务状态 Status"]
  S --> SH["分享/导入导出"]
```

## 一次会话的总流程

```mermaid
flowchart TD
  A["用户进入工作区"] --> B["创建或打开会话"]
  B --> C["输入任务目标"]
  C --> D["系统保存用户消息"]
  D --> E["选择 Agent 后端和模型"]
  E --> F["加载上下文：历史、技能、数据源"]
  F --> G["Agent 开始执行"]
  G --> H{"是否需要工具"}
  H -->|不需要| R["直接生成回答"]
  H -->|需要| I["调用工具"]
  I --> J{"是否敏感操作"}
  J -->|是| K["请求权限或计划审批"]
  K --> L{"用户是否批准"}
  L -->|批准| I
  L -->|拒绝| X["中断或改走安全路径"]
  J -->|否| M["执行工具"]
  M --> N["返回工具结果"]
  N --> G
  R --> O["保存最终结果"]
  X --> O
  O --> P["更新会话状态/未读/标签"]
  P --> Q["可继续、归档、分享、自动化"]
```

## 会话状态机

```mermaid
stateDiagram-v2
  [*] --> Created: 创建会话
  Created --> Idle: 初始化完成
  Idle --> Processing: 用户发送消息
  Processing --> WaitingPermission: 需要批准工具
  Processing --> WaitingPlanApproval: Agent 提交计划
  Processing --> WaitingCredential: 需要认证或凭据
  WaitingPermission --> Processing: 用户批准
  WaitingPermission --> Interrupted: 用户拒绝
  WaitingPlanApproval --> Processing: 用户接受计划
  WaitingPlanApproval --> Interrupted: 用户不接受或修改
  WaitingCredential --> Processing: 凭据完成
  WaitingCredential --> Interrupted: 凭据失败或取消
  Processing --> Completed: 本轮完成
  Processing --> Interrupted: 用户取消或重定向
  Processing --> Error: 执行失败
  Completed --> Idle: 等待下一轮
  Interrupted --> Idle: 可继续输入
  Error --> Idle: 用户可重试
  Idle --> Archived: 用户归档
  Archived --> Idle: 取消归档
```

## 用户、界面、后端、Agent 的协作时序

```mermaid
sequenceDiagram
  participant U as 用户
  participant UI as 界面
  participant SM as 会话管理器
  participant AG as Agent
  participant Tool as 工具/数据源

  U->>UI: 输入任务
  UI->>SM: 发送消息
  SM->>SM: 保存用户消息
  SM->>AG: 启动或恢复 Agent
  AG->>SM: 输出文本增量
  SM->>UI: 推送 text_delta
  AG->>Tool: 请求调用工具
  Tool-->>AG: 返回结果
  AG->>SM: 推送 tool_start/tool_result
  SM->>UI: 展示执行过程
  AG->>SM: complete
  SM->>SM: 保存最终消息
  SM->>UI: 更新会话状态
```

## 权限审批分支

```mermaid
sequenceDiagram
  participant AG as Agent
  participant SM as 会话管理器
  participant UI as 界面
  participant U as 用户
  participant Tool as 工具

  AG->>SM: 想执行敏感工具
  SM->>SM: 判断当前 permission mode
  alt Explore 安全模式
    SM-->>AG: 阻止写操作
    AG->>SM: 改为解释或只读方案
  else Ask to Edit
    SM->>UI: 展示权限请求
    UI->>U: 说明工具、影响、原因
    U->>UI: 批准或拒绝
    UI->>SM: 返回决定
    alt 批准
      SM->>Tool: 执行工具
      Tool-->>SM: 工具结果
      SM-->>AG: 返回结果
    else 拒绝
      SM-->>AG: 告知拒绝
    end
  else Auto
    SM->>Tool: 自动执行
    Tool-->>SM: 工具结果
    SM-->>AG: 返回结果
  end
```

## 计划审批分支

Plan 是一个非常重要的业务机制：它把“Agent 准备怎么做”变成用户可确认的契约。

```mermaid
flowchart TD
  A["Agent 判断任务复杂或有风险"] --> B["生成计划文件"]
  B --> C["提交 SubmitPlan"]
  C --> D["界面展示计划卡片"]
  D --> E{"用户选择"}
  E -->|接受| F["系统发送：计划已批准"]
  F --> G["Agent 按计划执行"]
  E -->|修改想法| H["用户补充要求"]
  H --> I["Agent 重新规划"]
  E -->|拒绝| J["任务中断或转为只读说明"]
```

产品价值：

- 用户更容易理解 Agent 接下来要做什么。
- 风险较高任务可以先对齐。
- 计划可以沉淀为团队工作方法。
- 消息平台也可以通过按钮批准计划。

## Credential/OAuth 分支

很多业务系统需要认证，例如 Slack、Google、Microsoft、GitHub。会话里可能出现“需要重新登录或输入凭据”的中断点。

```mermaid
flowchart TD
  A["Agent 需要使用某个 Source"] --> B{"Source 是否已认证"}
  B -->|已认证| C["正常调用工具"]
  B -->|未认证或过期| D["触发认证请求"]
  D --> E{"认证类型"}
  E -->|OAuth| F["打开浏览器授权"]
  E -->|API Key/Token| G["请求用户输入凭据"]
  F --> H{"授权是否成功"}
  G --> H
  H -->|成功| I["保存凭据"]
  I --> C
  H -->|失败| J["标记 Source 需要处理"]
  J --> K["Agent 告知用户无法继续该能力"]
```

## 会话中的业务状态

```mermaid
flowchart LR
  S["Session"] --> OPEN["Open 类状态"]
  S --> CLOSED["Closed 类状态"]

  OPEN --> BACKLOG["Backlog"]
  OPEN --> TODO["Todo"]
  OPEN --> REVIEW["Needs Review"]

  CLOSED --> DONE["Done"]
  CLOSED --> CANCELLED["Cancelled"]

  S --> FLAG["置顶/重要"]
  S --> ARCHIVE["归档"]
  S --> UNREAD["未读"]
  S --> LABEL["标签"]
```

这些字段让会话从“聊天记录”变成“任务管理对象”。产品上可以继续挖：

- 状态是否适合团队真实工作流。
- 标签是否能表达项目、优先级、客户、风险。
- 未读和通知是否帮助用户跟进长任务。
- 会话列表是否可以变成轻量任务看板。

## 分支与转移

用户可能希望从某条消息开始另开一条思路，或者把 session 转移到远程工作区。

```mermaid
flowchart TD
  A["原始会话"] --> B{"用户动作"}
  B -->|从某条消息分支| C["创建 Branch 会话"]
  C --> D["复制截止点前的上下文"]
  D --> E["新会话继续探索另一条路线"]

  B -->|转移到远程工作区| F["导出 Session Bundle"]
  F --> G{"Bundle 是否很大"}
  G -->|小| H["直接 RPC 导入"]
  G -->|大| I["分块传输"]
  H --> J["目标工作区出现新会话"]
  I --> J
```

## 会话生命周期中的关键体验点

```mermaid
flowchart TD
  ROOT["体验关键点"] --> A["开始：用户是否知道该如何描述任务"]
  ROOT --> B["过程：用户是否看得懂 Agent 在做什么"]
  ROOT --> C["风险：用户是否知道批准会带来什么影响"]
  ROOT --> D["等待：长任务是否有进度和通知"]
  ROOT --> E["结果：输出是否能被继续使用"]
  ROOT --> F["复用：这次任务是否能转成模板/技能/自动化"]
```

## 产品指标建议

| 阶段 | 可观察指标 | 代表什么 |
| --- | --- | --- |
| 创建会话 | 新会话数、首条消息发送率 | 用户是否成功开始任务 |
| 执行过程 | 工具调用数、权限请求批准率 | Agent 是否真的在行动，用户是否信任 |
| 中断点 | plan 修改率、auth 失败率、permission 拒绝率 | 哪里需要更清楚的解释 |
| 完成 | complete 率、error 率、cancel 率 | 任务完成质量 |
| 沉淀 | 标签/状态使用率、归档率、分享率 | 会话是否变成可管理资产 |
| 复用 | branch 数、automation 创建数、skill 创建数 | 用户是否从一次性使用走向复用 |

## 本文结论

会话生命周期可以被理解为：

```mermaid
flowchart LR
  A["用户意图"] --> B["会话承载"]
  B --> C["Agent 执行"]
  C --> D["权限/计划/凭据协调"]
  D --> E["结果输出"]
  E --> F["状态化管理"]
  F --> G["复用和协作"]
```

