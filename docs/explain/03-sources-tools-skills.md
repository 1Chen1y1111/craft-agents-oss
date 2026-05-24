# 03. Sources、Tools、Skills：Agent 为什么能做真实业务

## 三个概念先讲清楚

```mermaid
flowchart TD
  A["用户要完成任务"] --> B["Agent 需要能力"]
  B --> C["Source 数据源/系统入口"]
  B --> D["Tool 可执行动作"]
  B --> E["Skill 做事方法"]

  C --> C1["Slack、Gmail、Linear、GitHub、Notion"]
  C --> C2["本地文件、数据库、内部 API"]
  C --> C3["MCP server"]

  D --> D1["读文件"]
  D --> D2["改文件"]
  D --> D3["调接口"]
  D --> D4["搜索/浏览/生成数据"]

  E --> E1["团队 SOP"]
  E --> E2["领域知识"]
  E --> E3["固定工作步骤"]
```

一句话区别：

- Source 解决“Agent 能接触哪些业务系统和数据”。
- Tool 解决“Agent 能实际做哪些动作”。
- Skill 解决“Agent 应该按什么方法做事”。

## 能力从哪里来

```mermaid
flowchart LR
  U["用户意图"] --> A["Agent"]
  A --> P["系统提示和会话历史"]
  A --> SK["Skills"]
  A --> SO["Sources"]
  SO --> MCP["MCP 工具"]
  SO --> API["API 包装工具"]
  SO --> LOCAL["本地文件能力"]
  A --> ST["Session Tools"]
  A --> BUILTIN["内置读写/命令工具"]

  P --> R["更懂上下文"]
  SK --> R
  MCP --> R
  API --> R
  LOCAL --> R
  ST --> R
  BUILTIN --> R
```

## Source 的生命周期

```mermaid
stateDiagram-v2
  [*] --> Created: 创建 Source
  Created --> Configured: 填写连接信息
  Configured --> NeedsAuth: 需要认证
  Configured --> Ready: 无需认证
  NeedsAuth --> Authenticating: 用户开始 OAuth 或输入凭据
  Authenticating --> Ready: 认证成功
  Authenticating --> Failed: 认证失败
  Ready --> Active: 在会话中启用
  Active --> ToolAvailable: 工具可被 Agent 使用
  ToolAvailable --> TokenExpired: token 过期
  TokenExpired --> Refreshing: 自动刷新
  Refreshing --> ToolAvailable: 刷新成功
  Refreshing --> NeedsAuth: 刷新失败
  Failed --> Configured: 用户修正配置
  Active --> Disabled: 用户禁用
  Disabled --> Active: 用户重新启用
```

产品上，Source 生命周期对应一组关键体验：

- 创建是否简单。
- 认证是否顺畅。
- 出错时是否知道如何修复。
- 启用后 Agent 是否知道怎么用。
- token 过期是否自动恢复。

## Source 接入类型

```mermaid
flowchart TD
  S["Source"] --> MCP["MCP Source"]
  S --> API["API Source"]
  S --> LOCAL["Local Source"]

  MCP --> M1["HTTP/SSE MCP"]
  MCP --> M2["stdio 本地进程 MCP"]

  API --> A1["公开 API"]
  API --> A2["Bearer/Header/Query/Basic"]
  API --> A3["OAuth API"]
  API --> A4["Google/Slack/Microsoft 特殊 OAuth"]

  LOCAL --> L1["本地文件系统"]
  LOCAL --> L2["工作区目录"]
  LOCAL --> L3["用户附加文件"]
```

## Source 从配置到工具可用

```mermaid
sequenceDiagram
  participant U as 用户
  participant UI as 界面
  participant SS as Source 存储
  participant CB as SourceServerBuilder
  participant CP as McpClientPool
  participant AG as Agent

  U->>UI: 添加 Source
  UI->>SS: 保存 config.json 和 guide.md
  U->>UI: 完成认证
  UI->>SS: 记录认证状态
  AG->>CB: 请求构建 Source 工具
  CB->>SS: 读取 Source 配置
  CB->>CB: 合并凭据和 headers
  CB->>CP: 注册 MCP/API server
  CP->>CP: 拉取工具列表
  CP-->>AG: 暴露 proxy tools
  AG->>AG: 在任务中选择合适工具
```

## 工具调用路径

```mermaid
sequenceDiagram
  participant AG as Agent
  participant PM as 权限管理
  participant POOL as MCP 工具池
  participant EXT as 外部系统
  participant SM as 会话管理器
  participant UI as 界面

  AG->>SM: 准备调用工具
  SM->>PM: 检查权限模式
  alt 需要用户确认
    SM->>UI: 展示权限请求
    UI-->>SM: 用户批准
  end
  SM->>POOL: 执行 mcp__source__tool
  POOL->>EXT: 调用外部系统
  EXT-->>POOL: 返回结果
  POOL-->>SM: 工具结果
  SM-->>AG: 结果进入 Agent 上下文
  SM->>UI: 展示 tool_result
```

## Skill 的产品含义

Skill 可以理解为“给 Agent 的业务培训材料”。它通常不是工具本身，而是告诉 Agent：

- 这个领域怎么判断好坏。
- 做某类任务时要按什么步骤。
- 有哪些内部术语。
- 哪些坑不能踩。
- 输出格式应该是什么。

```mermaid
flowchart TD
  T["任务"] --> A["Agent"]
  A --> H{"是否有相关 Skill"}
  H -->|没有| G["按通用能力处理"]
  H -->|有| S["读取 Skill"]
  S --> M["理解团队方法"]
  M --> R["更稳定的输出"]

  S --> S1["流程步骤"]
  S --> S2["领域规则"]
  S --> S3["格式要求"]
  S --> S4["示例和反例"]
```

## Source Guide 的作用

每个 Source 可以带 `guide.md`。它像“使用说明书”，告诉 Agent：

- 这个 Source 是什么。
- 什么时候应该用。
- 调用前要知道哪些限制。
- 输出应该如何解释。

```mermaid
flowchart TD
  A["Agent 想使用某个 Source 工具"] --> B{"是否读过 guide.md"}
  B -->|没有| C["提醒必须先读 guide"]
  C --> D["Agent 调用 Read 工具读取 guide"]
  D --> E["标记 Source 已理解"]
  E --> F["允许调用 Source 工具"]
  B -->|读过| F
  F --> G["调用外部工具"]
```

业务价值：

- 降低 Agent 误用外部系统的概率。
- 让团队可以把“使用边界”写进文档。
- 给非技术用户提供一种可维护的控制方式。

## Session Tools 是什么

Session Tools 是系统给 Agent 的“自管理工具”。它们不是连接外部业务系统，而是帮助 Agent 管理当前任务。

```mermaid
flowchart TD
  ST["Session Tools"] --> P["SubmitPlan 提交计划"]
  ST --> CV["config_validate 校验配置"]
  ST --> SV["source_test 测试数据源"]
  ST --> OAuth["source_oauth_trigger 发起授权"]
  ST --> Cred["source_credential_prompt 请求凭据"]
  ST --> Data["transform_data 转换数据"]
  ST --> Render["render_template 渲染模板"]
  ST --> LLM["call_llm 调用小模型"]
  ST --> Spawn["spawn_session 创建子会话"]
  ST --> Labels["set_session_labels 设置标签"]
  ST --> Status["set_session_status 设置状态"]
```

从产品角度看，Session Tools 让 Agent 不只是“执行任务”，还可以“组织任务”。

## Agent 能力增强路径

```mermaid
flowchart LR
  A["普通聊天"] --> B["加会话历史"]
  B --> C["加 Skill"]
  C --> D["加 Source"]
  D --> E["加工具执行"]
  E --> F["加权限审批"]
  F --> G["加自动化"]
  G --> H["变成可运营的业务流程"]
```

## 典型业务场景

```mermaid
flowchart TD
  ROOT["业务场景"] --> S1["销售/客户成功"]
  ROOT --> S2["研发/代码协作"]
  ROOT --> S3["运营/内容"]
  ROOT --> S4["管理/流程自动化"]

  S1 --> A1["连接 Gmail/Calendar/CRM"]
  S1 --> A2["整理客户上下文"]
  S1 --> A3["生成跟进建议"]

  S2 --> B1["连接 GitHub/Linear"]
  S2 --> B2["读代码和 issue"]
  S2 --> B3["修改文件并生成说明"]

  S3 --> C1["连接文档/表格/社媒"]
  S3 --> C2["批量整理素材"]
  S3 --> C3["生成发布内容"]

  S4 --> D1["状态变化触发任务"]
  S4 --> D2["消息平台审批"]
  S4 --> D3["自动创建后续会话"]
```

## 产品经理值得挖的点

| 方向 | 可问的问题 | 可能机会 |
| --- | --- | --- |
| Source 市场 | 用户最想连哪些系统？ | 官方模板、推荐连接、行业包 |
| Source 诊断 | 为什么 Source 配不成功？ | 连接体检、错误修复向导 |
| Skill 沉淀 | 成功会话能否转 Skill？ | 一键沉淀 SOP |
| Tool 透明度 | 用户是否看懂工具做了什么？ | 更好的工具卡片、影响预览 |
| Guide 体验 | 非技术用户会不会写 guide？ | 可视化 guide builder |
| 权限策略 | 哪些工具常被拒绝？ | 更细颗粒的权限模板 |

## 本文结论

```mermaid
flowchart LR
  A["Source 决定数据边界"] --> B["Tool 决定行动边界"]
  B --> C["Skill 决定方法边界"]
  C --> D["Permission 决定信任边界"]
  D --> E["Agent 才能进入真实业务流程"]
```

