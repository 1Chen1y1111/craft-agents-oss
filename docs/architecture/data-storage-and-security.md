# 数据存储与安全设计

本文说明全局配置、工作区数据、session 文件、凭据存储、安全边界和可靠性策略。

## 存储根目录

全局配置目录由 `packages/shared/src/config/paths.ts` 决定：

```text
CRAFT_CONFIG_DIR 或 ~/.craft-agent/
```

典型结构：

```text
~/.craft-agent/
  config.json
  config-defaults.json
  credentials.enc
  docs/
  themes/
  tool-icons/
  workspaces/
  logs/
  feedback/
```

`CRAFT_CONFIG_DIR` 用于多实例开发或隔离测试环境。

## 全局配置

核心文件：`packages/shared/src/config/storage.ts`

`config.json` 保存：

- LLM connections metadata。
- 默认 LLM connection。
- 默认 thinking level。
- 工作区列表和 active workspace/session。
- UI 偏好：主题、通知、输入设置等。
- 网络代理。
- server mode 配置。
- migration markers。

敏感值不应放入 `config.json`。LLM API key、OAuth token、source credentials 应进入 credential storage。

## 工作区结构

工作区可以位于任意磁盘路径，默认在 `~/.craft-agent/workspaces/`。

```text
{workspaceRoot}/
  config.json
  sessions/
  sources/
  skills/
  labels/config.json
  statuses/config.json
  statuses/icons/
  automations.json
  automations-history.jsonl
  messaging/
```

核心文件：

- `packages/shared/src/workspaces/storage.ts`
- `packages/shared/src/labels/storage.ts`
- `packages/shared/src/statuses/storage.ts`
- `packages/shared/src/skills/storage.ts`
- `packages/shared/src/sources/storage.ts`

## Session 数据

每个 session 是一个目录：

```text
{workspaceRoot}/sessions/{sessionId}/
  session.jsonl
  attachments/
  plans/
  data/
  long_responses/
  downloads/
  meta/
```

目录含义：

| 路径 | 内容 |
| --- | --- |
| `session.jsonl` | header + messages |
| `attachments/` | 用户上传或引用附件 |
| `plans/` | `SubmitPlan` 生成的计划文件 |
| `data/` | `transform_data` 等工具输出 |
| `long_responses/` | 超大工具响应原文 |
| `downloads/` | API source 下载的二进制文件 |
| `meta/` | turn anchors、sidecar metadata |

JSONL 优点：

- header 快速读取。
- message 追加写入。
- 大会话避免整文件重写。
- 便于导入导出和恢复。

## Source 数据

```text
{workspaceRoot}/sources/{sourceSlug}/
  config.json
  guide.md
  .credential-cache.json
```

说明：

- `config.json` 描述 source 类型、auth、endpoint、tool config。
- `guide.md` 是模型使用该 source 前应阅读的说明。
- `.credential-cache.json` 是供外部 session MCP server 读取的临时凭据缓存，不是权威凭据存储。

权威凭据由 `CredentialManager` 管理。

## 凭据存储

核心文件：

- `packages/shared/src/credentials/manager.ts`
- `packages/shared/src/credentials/backends/secure-storage.ts`

当前有效 backend：`SecureStorageBackend`。

特点：

- 文件：`~/.craft-agent/credentials.enc`。
- 加密：AES-256-GCM。
- key 派生：OS 稳定机器标识 + PBKDF2。
- macOS 使用 IOPlatformUUID。
- Windows 使用 MachineGuid。
- Linux 使用 machine-id。
- 旧 hostname 派生 key 支持首次迁移。

环境变量 backend 当前 disabled，避免绕过手动录入流程。

## 安全边界

### RPC 安全

- 远程 server 必须使用 token handshake。
- token 太短或低熵会被拒绝或警告。
- 非 localhost 明文 bind 默认拒绝启动。
- Electron 薄客户端拒绝连接非 localhost 的 `ws://`。
- Web UI 通过 HTTP session cookie 参与 WS upgrade 鉴权。

### 工具安全

- permission mode 控制写操作。
- safe mode 阻止危险工具。
- source guide prerequisite 降低误用外部 API 风险。
- local stdio MCP 可按 workspace 禁用。
- path validation 防止 sessionId/path traversal。
- powershell/bash validator 检查部分命令风险。

### 凭据安全

- 普通 config 不保存 secret。
- OAuth token refresh 走 credential manager 和 token refresh manager。
- Sentry 发送前 scrub token、key、secret、password 等字段。
- 外部子进程只拿必要 credential cache 或 callback，不直接访问完整 credential manager。

## 可靠性设计

| 机制 | 目的 |
| --- | --- |
| server lock | 避免同一 config dir 多 server 并发写 |
| atomic write | 避免 config/status/label 写坏 |
| JSONL session | 降低大会话重写风险 |
| persistence queue | 顺序化 session 写入 |
| WS event replay | 断线后补发事件 |
| stale reconnect | buffer 过期时触发 UI 全量刷新 |
| subprocess isolation | Pi、WhatsApp 等重依赖隔离崩溃影响 |
| health endpoint | 支持容器或负载均衡探针 |

## 数据迁移原则

改持久化格式时：

1. 保持旧字段读取兼容。
2. 写入时可逐步升级，不要求所有旧文件一次性迁移。
3. 大文件迁移要避免启动阻塞。
4. session message schema 改动要同步 mapper 和 viewer。
5. 工作区路径要保存 portable path 时确认跨机器行为。
6. credential schema 改动要保留旧 credential id 解析。

## 远程部署注意事项

- 公网或局域网暴露 server 时使用 TLS。
- Docker 挂载持久 volume 保存 `~/.craft-agent`。
- 不要在日志中打印 token 或 credential。
- Web UI password 可短于 server token，但最终 cookie 仍由 server token secret 校验。
- 远程工作区路径属于 server 机器，不要把客户端本机路径发送给 remote runtime。
