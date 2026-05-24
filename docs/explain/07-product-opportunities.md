# 07. 产品机会地图：哪些点最值得继续挖

## 给产品经理的一句话

Craft Agents 的机会不只在“模型回答更聪明”，而在于把会话、工具、Source、Skill、权限、自动化、远程和消息平台组合成可复用的业务系统。最值得挖的是那些能让用户从“一次性提问”走向“可管理、可授权、可复用、可协作”的点。

```mermaid
flowchart TD
  ROOT["产品机会总览"] --> A["降低上手门槛"]
  ROOT --> B["提高任务完成率"]
  ROOT --> C["增强信任和控制"]
  ROOT --> D["促进复用和自动化"]
  ROOT --> E["推动团队协作"]
  ROOT --> F["形成生态和扩展"]

  A --> A1["任务模板"]
  A --> A2["Source 一键接入"]
  A --> A3["新手引导"]

  B --> B1["Plan 质量"]
  B --> B2["工具失败恢复"]
  B --> B3["上下文管理"]

  C --> C1["审批解释"]
  C --> C2["权限策略"]
  C --> C3["安全审计"]

  D --> D1["自动化编辑器"]
  D --> D2["Skill 沉淀"]
  D --> D3["会话转模板"]

  E --> E1["Viewer 分享"]
  E --> E2["消息平台协作"]
  E --> E3["远程团队 server"]

  F --> F1["Source 市场"]
  F --> F2["Skill 市场"]
  F --> F3["MCP 生态"]
```

## 先看用户价值链

```mermaid
flowchart LR
  A["我有一个任务"] --> B["我能说清楚吗"]
  B --> C["Agent 有上下文吗"]
  C --> D["Agent 有工具吗"]
  D --> E["我敢让它执行吗"]
  E --> F["过程可理解吗"]
  F --> G["结果可复用吗"]
  G --> H["团队能接着用吗"]

  B -. "机会" .-> B1["任务模板/Prompt 引导"]
  C -. "机会" .-> C1["Source/Skill 推荐"]
  D -. "机会" .-> D1["工具市场/连接器"]
  E -. "机会" .-> E1["权限解释/审批体验"]
  F -. "机会" .-> F1["执行过程可视化"]
  G -. "机会" .-> G1["会话转 Skill/Automation"]
  H -. "机会" .-> H1["分享/消息/团队工作区"]
```

## 用户分层

```mermaid
flowchart TD
  ROOT["用户类型"] --> P["产品经理/业务用户"]
  ROOT --> D["工程师"]
  ROOT --> O["运营/支持"]
  ROOT --> A["管理员/团队负责人"]

  P --> P1["梳理需求、评审结果、连接业务系统"]
  P --> P2["需要看得懂过程和风险"]

  D --> D1["读代码、改代码、跑命令、接 MCP"]
  D --> D2["需要高效率和可控权限"]

  O --> O1["批量整理、客户跟进、内容生成"]
  O --> O2["需要模板化和自动化"]

  A --> A1["配置团队 Source、权限、消息访问"]
  A --> A2["需要治理、审计、可规模化"]
```

## 值得挖掘的 12 个方向

```mermaid
flowchart TD
  ROOT["12 个产品机会"] --> O1["1. 会话任务化"]
  ROOT --> O2["2. Plan 协作化"]
  ROOT --> O3["3. Source 接入产品化"]
  ROOT --> O4["4. Skill 沉淀体系"]
  ROOT --> O5["5. 自动化可视化"]
  ROOT --> O6["6. 消息平台审批中心"]
  ROOT --> O7["7. 远程 server 团队化"]
  ROOT --> O8["8. Viewer 知识库化"]
  ROOT --> O9["9. 权限和安全策略"]
  ROOT --> O10["10. 失败恢复体验"]
  ROOT --> O11["11. 运行数据分析"]
  ROOT --> O12["12. Source/Skill 生态市场"]
```

下面逐个拆。

## 机会 1：会话任务化

当前 Session 已经有标签、状态、置顶、归档、未读等能力。机会是把它从“聊天列表”进一步变成“轻量任务工作台”。

```mermaid
flowchart TD
  A["Session"] --> B["聊天记录"]
  A --> C["任务状态"]
  A --> D["负责人/参与者"]
  A --> E["标签/项目/客户"]
  A --> F["风险/待审批"]
  A --> G["产物和附件"]
  A --> H["下一步动作"]

  B --> I["从对话变成任务资产"]
  C --> I
  D --> I
  E --> I
  F --> I
  G --> I
  H --> I
```

产品问题：

- 会话状态是否应该支持自定义流程？
- 会话列表是否可以切成看板视图？
- 是否需要“待我审批”“失败待处理”“今天完成”等智能筛选？

## 机会 2：Plan 协作化

Plan 现在是执行前的审批契约。进一步可以变成团队协作对象。

```mermaid
flowchart TD
  A["Agent 提交 Plan"] --> B["用户查看"]
  B --> C{"用户动作"}
  C -->|批准| D["执行"]
  C -->|评论| E["要求调整"]
  C -->|转发| F["给团队评审"]
  C -->|保存| G["转成模板或 Skill"]
  E --> A
  F --> H["消息平台或 Viewer 展示"]
  G --> I["下次复用"]
```

可做方向：

- Plan diff：本次计划和上次计划哪里不同。
- Plan comment：用户在某一步旁边留言。
- Plan template：高频计划保存成团队 SOP。
- Plan risk score：系统提示哪些步骤风险高。

## 机会 3：Source 接入产品化

Source 是 Agent 能力的入口。它越容易接入，Agent 越像“能在公司系统里干活的人”。

```mermaid
flowchart TD
  A["用户想接入一个系统"] --> B{"系统类型"}
  B -->|已有模板| C["选择模板，一键 OAuth"]
  B -->|MCP| D["粘贴 MCP 配置或自动发现"]
  B -->|REST API| E["填写 OpenAPI/API 文档"]
  B -->|内部系统| F["用向导生成 API source"]
  C --> G["自动生成 config 和 guide"]
  D --> G
  E --> G
  F --> G
  G --> H["测试连接"]
  H --> I["Agent 可使用工具"]
```

值得挖：

- 常见 Source 模板库：GitHub、Linear、Slack、Gmail、Notion、Jira。
- Source 健康页：认证、工具数量、最近失败。
- Source guide 生成器：根据 API 文档生成使用说明。
- Source 权限范围说明：让用户知道授权后能做什么。

## 机会 4：Skill 沉淀体系

Skill 是组织知识进入 Agent 的低门槛方式。产品上可以从“文件夹里的说明”升级成“团队知识资产”。

```mermaid
flowchart TD
  A["一次成功会话"] --> B["提炼做法"]
  B --> C["保存成 Skill"]
  C --> D["团队成员复用"]
  D --> E["更多会话产生反馈"]
  E --> F["更新 Skill"]
  F --> D
```

可做方向：

- 从会话一键生成 Skill 草稿。
- Skill 版本管理。
- Skill 命中率和效果评分。
- Skill 推荐：用户输入任务时推荐相关 Skill。
- Skill 审核：团队管理员批准后共享。

## 机会 5：自动化可视化

目前自动化配置偏工程化。对非技术用户来说，机会是把 Event、Matcher、Condition、Action 做成可视化搭建器。

```mermaid
flowchart LR
  A["当发生"] --> B["如果满足"]
  B --> C["就执行"]
  C --> D["然后通知"]

  A --> A1["状态变化/标签变化/定时/工具失败"]
  B --> B1["工作日/某标签/某项目/某权限模式"]
  C --> C1["创建 Agent 会话/发送 webhook"]
  D --> D1["Telegram topic/Web UI 通知/邮件"]
```

非技术用户看到的应该是：

```mermaid
flowchart TD
  A["选择触发器"] --> B["设置条件"]
  B --> C["选择动作"]
  C --> D["预览会发生什么"]
  D --> E["测试运行"]
  E --> F["启用"]
  F --> G["查看运行历史"]
```

## 机会 6：消息平台审批中心

消息平台已经能承载 Plan 和 Permission 按钮。下一步可以做成“待处理事项中心”。

```mermaid
flowchart TD
  A["Agent 需要用户处理"] --> B{"请求类型"}
  B -->|Plan| C["计划审批"]
  B -->|Permission| D["工具权限审批"]
  B -->|Credential| E["凭据需要重新授权"]
  B -->|Review| F["结果待评审"]
  C --> G["消息平台按钮"]
  D --> G
  E --> H["引导回 App 安全处理"]
  F --> G
  G --> I["审批结果回到 Session"]
```

产品点：

- “所有待我审批”的列表。
- 移动端友好的审批摘要。
- 到期提醒和自动取消。
- 审批历史可追踪。

## 机会 7：远程 server 团队化

远程 server 的价值不只是“跑在另一台机器”，而是成为团队 Agent 工作基础设施。

```mermaid
flowchart TD
  A["个人本地使用"] --> B["个人远程长任务"]
  B --> C["团队共享 server"]
  C --> D["团队 Source/Skill 统一配置"]
  D --> E["消息平台接入团队群"]
  E --> F["自动化常驻运行"]
  F --> G["组织级 Agent 工作台"]
```

可挖方向：

- 团队 workspace 成员和角色。
- 远程任务队列和资源占用。
- 长任务通知和恢复。
- 团队默认 Source、Skill、权限策略。
- 管理员仪表盘。

## 机会 8：Viewer 知识库化

Viewer 是只读分享。继续发展可以成为“Agent 产出知识库”。

```mermaid
flowchart TD
  A["高价值 Session"] --> B["分享 Viewer"]
  B --> C["团队成员阅读过程"]
  C --> D["收藏/评论/标注"]
  D --> E["沉淀到知识库"]
  E --> F["转成 Skill 或模板"]
  F --> G["反哺新任务"]
```

适合的内容：

- 需求分析过程。
- 代码改动解释。
- 客户问题排查。
- 竞品研究。
- 自动化运行案例。

## 机会 9：权限和安全策略

当前有基础权限模式。团队化之后，需要从“单次审批”发展到“策略管理”。

```mermaid
flowchart TD
  ROOT["权限策略"] --> A["按工具类型"]
  ROOT --> B["按 Source"]
  ROOT --> C["按工作区"]
  ROOT --> D["按用户角色"]
  ROOT --> E["按自动化"]
  ROOT --> F["按风险等级"]

  A --> A1["读文件允许，写文件询问"]
  B --> B1["GitHub 写操作询问，Slack 发消息询问"]
  C --> C1["生产工作区更严格"]
  D --> D1["管理员可 Auto，成员默认 Ask"]
  E --> E1["自动化默认 safe"]
  F --> F1["高风险命令必须二次确认"]
```

产品目标：让用户不是在每次弹窗里做重复判断，而是建立可解释的默认策略。

## 机会 10：失败恢复体验

Agent 失败并不可怕，可怕的是用户不知道下一步怎么办。

```mermaid
flowchart TD
  A["失败发生"] --> B["识别失败类型"]
  B --> C["用人话解释"]
  C --> D["给出下一步按钮"]
  D --> E{"用户选择"}
  E -->|重试| F["按原计划重试"]
  E -->|换路径| G["让 Agent 重新规划"]
  E -->|补凭据| H["进入认证流程"]
  E -->|人工接管| I["导出上下文或生成操作指南"]
```

可做方向：

- 失败分类：权限、认证、工具、网络、模型、上下文过长。
- 一键重试。
- 自动生成排查会话。
- 失败后推荐更安全的替代路径。

## 机会 11：运行数据分析

当用户从个人使用进入团队使用，数据分析会变得关键。

```mermaid
flowchart TD
  ROOT["运营分析"] --> A["任务量"]
  ROOT --> B["完成质量"]
  ROOT --> C["工具使用"]
  ROOT --> D["审批效率"]
  ROOT --> E["自动化效果"]
  ROOT --> F["Source 健康"]

  A --> A1["新建 session 数"]
  A --> A2["活跃工作区数"]
  B --> B1["完成率/Error 率"]
  B --> B2["用户继续追问率"]
  C --> C1["高频工具"]
  C --> C2["失败工具"]
  D --> D1["审批等待时长"]
  D --> D2["批准/拒绝比例"]
  E --> E1["触发次数"]
  E --> E2["节省的人工操作"]
  F --> F1["认证过期"]
  F --> F2["连接失败"]
```

## 机会 12：Source/Skill 生态市场

长期看，Craft Agents 的扩展价值来自生态。

```mermaid
flowchart TD
  A["用户有新业务场景"] --> B{"是否已有能力包"}
  B -->|有| C["安装 Source/Skill 模板"]
  B -->|没有| D["用户或社区创建"]
  D --> E["分享给团队或社区"]
  E --> F["更多人安装"]
  F --> G["反馈和评分"]
  G --> H["能力包质量提升"]
  H --> C
```

生态对象可以包括：

- Source 模板。
- Skill 模板。
- 自动化模板。
- 消息平台场景包。
- 行业 SOP 包。

## 优先级评估框架

```mermaid
flowchart TD
  A["评估一个产品机会"] --> B["用户痛感是否高"]
  A --> C["是否能提升完成率"]
  A --> D["是否能提升信任"]
  A --> E["是否能带来复用"]
  A --> F["是否能促进团队传播"]
  A --> G["实现复杂度是否可控"]
  A --> H["是否依赖底层改造"]

  B --> SCORE["机会评分"]
  C --> SCORE
  D --> SCORE
  E --> SCORE
  F --> SCORE
  G --> SCORE
  H --> SCORE
```

建议用下面的方式粗排：

| 机会 | 用户价值 | 传播价值 | 实现复杂度 | 建议 |
| --- | --- | --- | --- | --- |
| 会话任务化 | 高 | 中 | 中 | 优先 |
| Source 接入产品化 | 高 | 高 | 高 | 优先但分阶段 |
| 自动化可视化 | 高 | 高 | 高 | 做 MVP |
| 消息审批中心 | 高 | 高 | 中 | 优先 |
| 权限策略 | 高 | 中 | 中 | 随团队化推进 |
| Viewer 知识库化 | 中 | 高 | 中 | 适合增长 |
| 运行数据分析 | 中 | 中 | 中 | 团队版需要 |
| 生态市场 | 高 | 高 | 高 | 长期方向 |

## 推荐路线图

```mermaid
gantt
  title 产品机会路线图示意
  dateFormat  YYYY-MM-DD
  section 近期：降低门槛和建立信任
  会话任务化基础视图        :a1, 2026-06-01, 30d
  审批卡信息优化            :a2, after a1, 20d
  Source 健康和认证引导     :a3, 2026-06-10, 35d
  section 中期：复用和协作
  自动化可视化 MVP          :b1, 2026-07-15, 45d
  消息审批中心              :b2, 2026-07-20, 40d
  会话转 Skill 草稿         :b3, 2026-08-01, 35d
  section 长期：团队和生态
  团队远程 server 管理      :c1, 2026-09-01, 60d
  Source/Skill 模板市场     :c2, 2026-10-01, 75d
  组织级分析仪表盘          :c3, 2026-10-15, 60d
```

这只是产品节奏示例，不代表代码里已经有这些日期计划。

## 从 MVP 开始的三个切入点

```mermaid
flowchart TD
  ROOT["建议 MVP"] --> A["MVP 1：待处理中心"]
  ROOT --> B["MVP 2：自动化模板"]
  ROOT --> C["MVP 3：Source 健康页"]

  A --> A1["聚合 Plan、Permission、Credential、Review"]
  A --> A2["先做 Web/Electron 内列表，再扩到消息平台"]

  B --> B1["提供日报、失败巡检、状态评审 3 个模板"]
  B --> B2["用户不用写 JSON，只填变量"]

  C --> C1["展示连接状态、认证状态、工具数量、最近错误"]
  C --> C2["降低 Source 接入和维护成本"]
```

为什么这三个优先：

- 都直接降低非技术用户使用门槛。
- 都围绕已有架构对象，不需要完全重造底层。
- 都能产生明确指标。
- 都能支持后续团队化和自动化。

## 关键指标树

```mermaid
flowchart TD
  NSM["北极星：成功完成并被复用的 Agent 工作数"] --> A["完成"]
  NSM --> B["信任"]
  NSM --> C["复用"]
  NSM --> D["协作"]

  A --> A1["session complete rate"]
  A --> A2["tool success rate"]
  A --> A3["failure recovery rate"]

  B --> B1["plan approve rate"]
  B --> B2["permission approve rate"]
  B --> B3["credential success rate"]

  C --> C1["session to skill rate"]
  C --> C2["automation created rate"]
  C --> C3["template reused rate"]

  D --> D1["viewer share rate"]
  D --> D2["messaging binding count"]
  D --> D3["external approval count"]
```

## 需要避免的产品陷阱

```mermaid
flowchart TD
  ROOT["常见陷阱"] --> A["只优化聊天体验，忽略任务闭环"]
  ROOT --> B["让非技术用户手写太多配置"]
  ROOT --> C["权限弹窗只给按钮不给影响说明"]
  ROOT --> D["自动化触发了但用户不知道为什么"]
  ROOT --> E["Source 连接失败没有可操作修复路径"]
  ROOT --> F["远程和本地路径概念混淆"]
  ROOT --> G["分享结果只展示答案，不展示过程"]
```

对应的设计原则：

```mermaid
flowchart LR
  A["看得懂"] --> B["敢批准"]
  B --> C["能复用"]
  C --> D["可协作"]
  D --> E["可治理"]
```

## 最适合讲给业务团队的产品故事

```mermaid
flowchart TD
  A["一个 PM 创建需求分析会话"] --> B["Agent 连接 Linear、GitHub、Slack"]
  B --> C["生成计划并请求审批"]
  C --> D["PM 在 Telegram 批准"]
  D --> E["Agent 汇总需求、风险、相关代码"]
  E --> F["会话状态变为 Needs Review"]
  F --> G["自动化生成评审摘要"]
  G --> H["Viewer 分享给团队"]
  H --> I["团队把流程保存成 Skill 和自动化模板"]
```

这个故事串起了本仓库最核心的业务资产：

- 会话是任务容器。
- Source 让 Agent 进入业务系统。
- Plan 和 Permission 建立信任。
- 消息平台让审批不受端限制。
- 自动化让流程复用。
- Viewer 和 Skill 让成果传播和沉淀。

## 本文结论

Craft Agents 的产品机会应该围绕一个主线推进：把用户的一次成功使用，转成团队可复用的工作方式。

```mermaid
flowchart LR
  A["一次成功会话"] --> B["形成可理解过程"]
  B --> C["建立用户信任"]
  C --> D["沉淀 Skill/模板"]
  D --> E["自动化复用"]
  E --> F["消息平台协作"]
  F --> G["团队级扩散"]
  G --> A
```

