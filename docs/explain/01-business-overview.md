# 01. 业务全景：Craft Agents 到底在解决什么问题

## 给产品经理的一句话

Craft Agents 是一个“让用户以会话为中心，调度多个 AI Agent、外部数据源和工具来完成真实工作的系统”。它的业务重点不是模型本身，而是把“意图 -> 工具执行 -> 权限确认 -> 结果沉淀 -> 自动化复用”做成一个完整工作流。

```mermaid
flowchart LR
  Q["用户说出目标"] --> C["系统创建会话"]
  C --> A["选择合适 Agent/模型"]
  A --> K["读取上下文和技能"]
  K --> S["连接外部数据源"]
  S --> T["调用工具执行任务"]
  T --> G{"是否需要用户确认"}
  G -->|需要| P["展示计划/权限/凭据请求"]
  P --> T
  G -->|不需要| R["产出结果"]
  R --> H["沉淀为历史、文件、状态、标签"]
  H --> O["可分享、可继续、可自动化"]
```

## 这不是普通聊天产品

普通聊天产品的业务闭环通常是：

```mermaid
flowchart LR
  U["用户提问"] --> M["模型回答"] --> U
```

Craft Agents 的业务闭环更长：

```mermaid
flowchart LR
  U["用户提出任务"] --> W["工作区"]
  W --> S["会话"]
  S --> A["Agent"]
  A --> D["数据源"]
  A --> T["工具"]
  T --> F["文件/系统/API 变更"]
  F --> V["用户审核"]
  V --> R["结果沉淀"]
  R --> N["通知/分享/自动化"]
  N --> U
```

这意味着产品设计重点不只是“回答好不好”，还包括：

- 用户如何组织任务。
- Agent 如何知道哪些外部系统可用。
- 什么时候必须让用户确认。
- 执行过程如何透明。
- 结果如何沉淀为可追踪的工作记录。
- 是否能从一次性聊天升级为可复用流程。

## 关键角色

```mermaid
flowchart TD
  PM["产品经理/业务用户"] --> APP["Craft Agents"]
  DEV["工程师/技术用户"] --> APP
  OPS["运营/支持人员"] --> APP
  ADMIN["团队管理员"] --> APP

  APP --> MODEL["AI 模型提供方"]
  APP --> API["外部业务系统"]
  APP --> MCP["MCP 工具生态"]
  APP --> MSG["消息平台"]
  APP --> FILE["本地/远程文件系统"]

  MODEL --> M1["Anthropic Claude"]
  MODEL --> M2["OpenAI/Google/Copilot 等经 Pi 接入"]

  API --> A1["Slack/Gmail/Linear/GitHub/Notion"]
  MCP --> A2["本地命令/远程工具/自定义 MCP"]
  MSG --> A3["Telegram/WhatsApp/Lark"]
```

## 产品能力地图

```mermaid
flowchart TB
  ROOT["Craft Agents 产品能力"] --> C1["任务组织"]
  ROOT --> C2["Agent 执行"]
  ROOT --> C3["外部能力接入"]
  ROOT --> C4["信任与控制"]
  ROOT --> C5["协作与分发"]
  ROOT --> C6["自动化"]

  C1 --> C11["Workspace 工作区"]
  C1 --> C12["Session 会话"]
  C1 --> C13["标签和状态"]
  C1 --> C14["归档/置顶/未读"]

  C2 --> C21["Claude 后端"]
  C2 --> C22["Pi 后端"]
  C2 --> C23["模型/Thinking Level"]
  C2 --> C24["流式执行过程"]

  C3 --> C31["Sources 数据源"]
  C3 --> C32["Skills 技能"]
  C3 --> C33["MCP 工具"]
  C3 --> C34["REST API 包装工具"]

  C4 --> C41["权限模式"]
  C4 --> C42["计划审批"]
  C4 --> C43["凭据/OAuth"]
  C4 --> C44["远程 TLS/token"]

  C5 --> C51["Electron 桌面"]
  C5 --> C52["Web UI"]
  C5 --> C53["CLI"]
  C5 --> C54["Viewer 分享"]

  C6 --> C61["事件触发"]
  C6 --> C62["定时任务"]
  C6 --> C63["Webhook"]
  C6 --> C64["消息平台审批"]
```

## 核心业务对象

| 对象 | 产品含义 | 用户能感知到什么 |
| --- | --- | --- |
| Workspace | 一组业务上下文和配置的容器 | 我的项目、团队空间、客户空间 |
| Session | 一次任务或一串连续工作 | 一个对话、一项任务、一条工作记录 |
| Agent | 执行任务的智能体后端 | 不同模型能力、不同执行方式 |
| Source | 外部系统或数据入口 | 连接 Slack、Linear、Gmail、GitHub |
| Skill | 让 Agent 学会特定做法的说明书 | 团队 SOP、领域知识、工作套路 |
| Tool | Agent 可调用的实际动作 | 读文件、改代码、调 API、发消息 |
| Permission | 执行边界 | 是否允许 Agent 改东西 |
| Plan | 执行前承诺 | Agent 先列计划，用户批准再做 |
| Automation | 可复用流程 | 状态变化后自动触发任务 |
| Messaging Binding | 外部聊天与会话绑定 | 在 Telegram/WhatsApp 里推进任务 |

## 价值链路

```mermaid
flowchart LR
  I["输入：用户意图"] --> C["上下文：工作区/历史/技能/数据源"]
  C --> E["执行：Agent + 工具"]
  E --> G["治理：权限/计划/凭据"]
  G --> O["输出：文本/文件/API 变更/状态"]
  O --> R["沉淀：会话历史/标签/状态"]
  R --> X["扩展：分享/自动化/消息平台"]

  I -. "越自然越好" .-> UX1["低门槛"]
  C -. "越丰富越强" .-> UX2["上下文优势"]
  G -. "越透明越可信" .-> UX3["信任建立"]
  X -. "越可复用越规模化" .-> UX4["组织扩散"]
```

## 三个业务飞轮

### 1. 能力飞轮

```mermaid
flowchart LR
  A["用户提出更多任务"] --> B["发现缺少数据源/工具"]
  B --> C["添加 Source 或 Skill"]
  C --> D["Agent 能做更多事"]
  D --> E["用户更依赖系统"]
  E --> A
```

### 2. 信任飞轮

```mermaid
flowchart LR
  A["Agent 展示过程"] --> B["用户看见工具调用"]
  B --> C["敏感操作需要确认"]
  C --> D["用户敢授权更多范围"]
  D --> E["任务完成度提升"]
  E --> A
```

### 3. 组织飞轮

```mermaid
flowchart LR
  A["个人完成任务"] --> B["会话可分享"]
  B --> C["团队看到可复用流程"]
  C --> D["沉淀 Skill/Automation"]
  D --> E["更多团队成员采用"]
  E --> A
```

## 端到端用户旅程

```mermaid
journey
  title 用户从首次使用到团队复用的旅程
  section 首次上手
    打开应用: 4: 用户
    选择模型连接: 3: 用户
    创建工作区: 4: 用户
  section 完成第一个任务
    发起会话: 5: 用户
    等待 Agent 分析: 4: Agent
    查看工具执行过程: 4: 用户
    批准敏感操作: 3: 用户
    得到结果: 5: 用户
  section 扩展能力
    添加 Source: 4: 用户
    添加 Skill: 4: 用户
    复用上下文: 5: 用户
  section 团队扩散
    分享会话: 4: 用户
    创建自动化: 3: 用户
    接入消息平台: 4: 团队
```

## 产品经理最该追问的问题

```mermaid
flowchart TD
  ROOT["产品追问"] --> Q1["用户最常创建哪类会话"]
  ROOT --> Q2["哪些 Source 是高频刚需"]
  ROOT --> Q3["用户在哪些权限点犹豫"]
  ROOT --> Q4["Plan 审批是否提升信任"]
  ROOT --> Q5["会话结果是否能转为团队资产"]
  ROOT --> Q6["自动化触发是否真的减少重复劳动"]
  ROOT --> Q7["消息平台入口是否带来更高响应率"]
  ROOT --> Q8["远程 server 是否解决长任务痛点"]
```

## 本文结论

Craft Agents 的业务逻辑可以概括为：

```mermaid
flowchart LR
  A["把用户意图"] --> B["放进可追踪会话"]
  B --> C["交给可切换 Agent"]
  C --> D["使用可扩展工具"]
  D --> E["在可控权限下执行"]
  E --> F["沉淀为可复用资产"]
  F --> G["通过自动化和消息入口规模化"]
```

