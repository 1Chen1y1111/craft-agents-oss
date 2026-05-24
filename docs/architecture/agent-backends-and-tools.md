# Agent 后端与工具系统设计

本文说明 Agent 后端抽象、Claude/Pi 两条执行路径、Sources、MCP pool 和 session-scoped tools 的协作关系。

## 分层总览

```text
SessionManager
  |
  +--> resolve LLM connection
  +--> resolve enabled sources
  +--> create/reuse AgentBackend
          |
          v
       BaseAgent
          |
          +--> ClaudeAgent
          |
          +--> PiAgent
                 |
                 v
              pi-agent-server subprocess

Sources
  |
  +--> SourceServerBuilder
  |
  +--> McpClientPool
  |
  +--> proxy tools: mcp__{slug}__{tool}

Session tools
  |
  +--> session-tools-core
  |
  +--> in-process callbacks or session-mcp-server
```

## AgentBackend 接口

核心类型：`packages/shared/src/agent/backend/types.ts`

`AgentBackend` 屏蔽不同 SDK 的差异。关键方法：

- `chat(message, attachments, options)`：核心 agent loop，返回 `AsyncGenerator<AgentEvent>`。
- `abort()` / `forceAbort()`：停止当前 turn。
- `interruptForHandoff()`：plan/auth 等交给 UI 决策时中断。
- `redirect()`：用户在处理过程中追加消息时的中途转向。
- `runMiniCompletion()`：标题、摘要、测试连接、`call_llm` 等轻量模型调用。
- `postInit()`：构造后、首次 chat 前初始化。
- `applyBridgeUpdates()`：sources 或 token 更新时应用运行时配置。
- `ensureBranchReady()`：branch session 首次消息前准备后端上下文。
- `updateRuntimeConfig()`：模型或 provider runtime 变更。

新增后端必须先回答三个问题：

- SDK 能否直接在主进程运行，还是需要子进程隔离。
- 工具调用是否支持 pre-tool-use、permission、MCP proxy。
- session resume/branch 能力如何映射到统一 session 模型。

## BaseAgent 公共能力

核心文件：`packages/shared/src/agent/base-agent.ts`

`BaseAgent` 提供跨后端公共逻辑：

- 模型和 thinking level 状态。
- `PermissionManager`。
- `SourceManager`。
- `PromptBuilder`。
- `PathProcessor`。
- `ConfigWatcherManager`。
- `UsageTracker`。
- `PrerequisiteManager`。
- source activation restart。
- session MCP tool completion callback。
- mini agent 相关工具。

后端实现应尽量复用 BaseAgent，避免在具体 SDK 类里重复 source、permission、prompt 逻辑。

## ClaudeAgent 路径

核心文件：

- `packages/shared/src/agent/claude-agent.ts`
- `packages/shared/src/agent/backend/claude/event-adapter.ts`
- `packages/shared/src/agent/session-scoped-tools.ts`

ClaudeAgent 直接使用 `@anthropic-ai/claude-agent-sdk`。特点：

- session tools 可通过 SDK in-process 注册。
- callback registry 在同进程有效。
- 事件通过 `ClaudeEventAdapter` 归一化为 `AgentEvent`。
- 支持 Claude SDK resume/fork 相关 metadata。

简化链路：

```text
SessionManager
  |
  v
ClaudeAgent.chat()
  |
  v
Claude Agent SDK
  |
  +--> built-in tools
  +--> MCP source tools
  +--> session-scoped tools
  |
  v
ClaudeEventAdapter -> AgentEvent
```

## PiAgent 路径

核心文件：

- `packages/shared/src/agent/pi-agent.ts`
- `packages/pi-agent-server/src/index.ts`
- `packages/shared/src/agent/backend/pi/event-adapter.ts`
- `packages/shared/src/agent/backend/pi/session-tool-defs.ts`

PiAgent 使用子进程隔离 Pi SDK。原因：

- Pi SDK 依赖较重。
- Electron main bundle 不直接承载 Pi runtime。
- JSONL 协议更容易隔离崩溃、stderr 和工具执行。

```text
PiAgent
  |
  | JSONL
  v
pi-agent-server
  |
  +--> createAgentSession()
  +--> register proxy tools
  +--> emit pre_tool_use_request
  +--> emit tool_execute_request
  +--> emit event
  |
  v
PiAgent maps to AgentEvent
```

Pi 的 session tools 由于运行在外部 MCP server 或子进程中，不能直接使用主进程 callback registry。BaseAgent 通过 `handleSessionMcpToolCompletion()` 在主进程补发 SubmitPlan/auth 等 callback。

## LLM connection 到后端选择

核心文件：`packages/shared/src/agent/backend/factory.ts`

映射原则：

| `providerType` | 后端 |
| --- | --- |
| `anthropic` | `ClaudeAgent` |
| `pi` | `PiAgent` |
| `pi_compat` | `PiAgent` |

`resolveSessionConnection()` 会结合 session、workspace 和全局默认连接确定本次使用的 provider、model、authType 和 runtime。

## Sources 与 MCP pool

Sources 定义在 `packages/shared/src/sources`。

```text
LoadedSource
  |
  +--> credentials
  |
  v
SourceServerBuilder
  |
  +--> MCP config: http / sse / stdio
  |
  +--> API source: in-process MCP server
  |
  v
McpClientPool.sync()
  |
  +--> connect/disconnect clients
  +--> list tools
  +--> proxy tool defs
```

`McpClientPool` 负责：

- 集中管理 MCP client 生命周期。
- 缓存 tool list。
- 将源工具命名为 `mcp__{slug}__{toolName}`。
- 执行 tool call 并处理大响应、二进制响应。
- 过滤 local stdio MCP 是否允许。

## SourceManager

核心文件：`packages/shared/src/agent/core/source-manager.ts`

职责：

- 跟踪 active、intended active、inactive sources。
- 生成 `<sources>` context block 注入消息。
- 提醒模型读取 source `guide.md`。
- 标识 needs auth 或 failed sources。
- 在 source 激活后触发当前 turn restart，让新工具进入下一轮上下文。

## Session-scoped tools

核心包：`packages/session-tools-core`

这些工具不是某个 source 的外部工具，而是 session runtime 提供给 Agent 的自管理能力。

典型工具：

- `SubmitPlan`
- `config_validate`
- `skill_validate`
- `mermaid_validate`
- `source_test`
- `source_oauth_trigger`
- `source_credential_prompt`
- `update_user_preferences`
- `transform_data`
- `script_sandbox`
- `render_template`
- `call_llm`
- `spawn_session`
- `set_session_labels`
- `set_session_status`
- `list_sessions`

接入方式：

- Claude：优先 in-process session tools。
- Pi/Codex 类外部 runtime：通过 `packages/session-mcp-server` stdio MCP server。
- 部分工具需要主进程 callback 或 HTTP callback。

## Tool permission 边界

工具权限由多层共同控制：

- permission mode：safe / ask / allow-all。
- `PermissionManager` 和 pre-tool-use checks。
- session tool safe allow/block 列表。
- source guide prerequisite。
- local MCP workspace setting。
- path validation 和 working directory boundary。

新增工具时要明确：

- 是否读写文件。
- 是否访问网络。
- 是否需要用户 credential。
- safe mode 下是否允许。
- 结果是否可能很大或包含二进制。

## 新增工具或 source 检查清单

1. 定义 schema 和描述，优先放在 `session-tools-core` 或 source config。
2. 明确权限模式行为。
3. 明确是否需要主进程 callback。
4. 明确结果大小和附件保存策略。
5. 更新 Claude 与 Pi 的工具注册路径。
6. 更新 tests：tool defs parity、permission、source builder、MCP pool。
