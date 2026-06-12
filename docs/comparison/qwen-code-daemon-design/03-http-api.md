# 03 — HTTP API & Protocol

> [← 上一篇：Design Decisions](./02-architectural-decisions.md) · [下一篇：Deployment & Client →](./04-deployment-and-client.md)

## TL;DR

`qwen serve` 启动一个绑定单 workspace 的 daemon（默认 `127.0.0.1:4170`），northbound 暴露三套 transport：

1. **REST + SSE 控制面**（本篇主体）——约 76 个 HTTP 路由 + `GET /session/:id/events` 事件流
2. **ACP HTTP** `/acp`（#4472）——官方 ACP Streamable HTTP transport
3. **ACP WebSocket**——同一 `/acp` 路径 upgrade（#4773），与 SSE 共存

三套 transport 共享同一 bridge / EventBus，业务语义一致；另有 `qwen-serve-bridge`（#4555，`sdk-typescript` 内）把 daemon 包装成 MCP stdio server 供 MCP-native client 使用。

关键协议形态：`POST /session/:id/prompt` **非阻塞**——立即返回 `202 {promptId, lastEventId}`，turn 结果经 SSE `turn_complete` / `turn_error` 按 `promptId` 关联（#4585）。全部能力经 `GET /capabilities` 的 feature tag 协商（64 个 tag，其中 9 个条件性 advertise）。

> 本篇路由清单逐条对照 qwen-code main 源码（`packages/cli/src/serve/`）核实，截至 2026-06-12。

---

## 一、通用约定

### 1.1 Base URL 与认证

| 项 | 约定 |
|---|---|
| Base URL | `http://<host>:<port>`，默认 `127.0.0.1:4170`（`--port` / `--host`） |
| 认证头 | `Authorization: Bearer <token>`（scheme 大小写不敏感；token 经 SHA-256 + `timingSafeEqual` 比较） |
| token 来源 | `--token` 或 `QWEN_SERVER_TOKEN` 环境变量 |
| 无 token | 仅允许 loopback bind（非 loopback bind 无 token 时 boot 直接拒绝启动） |
| `--require-auth` | loopback 也强制 bearer；boot 要求已配置 token；`/health` 同样收进认证之后 |
| body 上限 | `express.json({limit: '10mb'})` |

源码: `packages/cli/src/serve/auth.ts`、`runQwenServe.ts`。

### 1.2 浏览器与网络边界

- **默认拒绝浏览器**：任何带 `Origin` 头的请求返回 `403`（CLI/SDK client 不发 Origin，只有浏览器发）。`--allow-origin <pattern>` 开白名单（精确 origin 或 `*`；`*` 必须配 token）。
- **Host allowlist**：校验 `Host` 头防 DNS rebinding；loopback 来源判定只读 `req.socket.remoteAddress`，不信任 `X-Forwarded-For`。

### 1.3 `X-Qwen-Client-Id`

client 身份头，daemon 在事件 envelope 上 stamping 为 `originatorClientId`（让其他订阅者知道"是谁干的"）：

- 格式：≤128 字符 token，正则校验；非法 → `400 invalid_client_id`
- session 级 mutation 路由解析后透传；**workspace 级 mutation 路由额外校验该 id 必须在 `bridge.knownClientIds()` 注册集合内**，防伪造他人身份（#4231）

### 1.4 mutation gate 两档（`--require-auth` 门禁）

写路由统一走 mutation gate 中间件，分两档：

| 档 | 适用路由 | 行为 |
|---|---|---|
| **non-strict** `mutate()` | `/session`、`/session/:id/prompt`、`cancel`、`model`、`recap`、`btw`、`shell` 等会话操作 | 默认 passthrough；`--require-auth` 开启后强制 bearer |
| **strict** `mutate({strict: true})` | `/workspace/*` 全部写路由、`approval-mode`、`rewind`、`goal/clear`、`tasks/:taskId/cancel`、device-flow 全部 | **即便 loopback 无 token 部署也返回 `401 {code: 'token_required'}`** |

4-cell 行为矩阵：`{token 已配置 | 未配置} × {strict | non-strict}`——token 一旦配置，全局 `bearerAuth` 对所有路由生效，gate 退化为冗余防线。

### 1.5 错误形态与 `errorKind` 封闭分类

三类错误 wire 形态：

1. **请求校验错误**：`400 {error, code}`（如 `invalid_client_id`、`invalid_deadline_ms`、`workspace_mismatch`）
2. **bridge 域错误**：经 `mapDomainErrorToErrorKind` 映射到 `SERVE_ERROR_KINDS` 封闭 enum，**14 值**（acp-bridge 与 SDK 两端 lockstep，drift-insurance test 锁排序）：

```ts
// 源码: packages/acp-bridge/src/status.ts#L19
'missing_binary' | 'blocked_egress' | 'auth_env_error' | 'init_timeout'
| 'protocol_error' | 'missing_file' | 'parse_error' | 'stat_failed'
| 'budget_exhausted'           // MCP budget enforce 模式拒服务器
| 'mcp_budget_would_exceed' | 'mcp_server_spawn_failed' | 'invalid_config'  // 运行时 MCP 增删
| 'prompt_deadline_exceeded' | 'writer_idle_timeout'    // 运行时防护 flag（详 §八）
```

3. **File 路由错误**：独立的 `FsErrorKind` 封闭 enum（14 值，详 §五），wire 形态 `{errorKind, error, status, hint?}`

设计哲学：daemon 内部用富类型 `Error` 子类保留 stack/cause，序列化到 wire 时降级为封闭 enum + redacted message——SDK 客户端 `switch (err.errorKind)` 而非 message regex。

### 1.6 Capability 协商

```jsonc
GET /capabilities
{
  "v": 1,
  "protocolVersions": { "current": "v1", "supported": ["v1"] },
  "qwenCodeVersion": "0.18.0",
  "mode": "http-bridge",
  "features": [/* 64 个 tag */],
  "modelServices": [],
  "workspaceCwd": "/abs/path",            // client pre-flight，POST /session 可省 cwd
  "policy": { "permission": "first-responder" },
  "supportedLanguages": [/* 语言码 */]
}
```

- **注册表**：`SERVE_CAPABILITY_REGISTRY`（`serve/capabilities.ts`）64 个 tag，每个带 `since: 'v1'`；新增路由 additive 注册，生产/unit/integration 三套来源 lockstep（CI 锁 `toEqual`）。
- **条件性 tag（9 个）**：仅当对应运行时配置开启时 advertise——`require_auth`、`allow_origin`、`prompt_absolute_deadline`、`writer_idle_timeout`、`mcp_workspace_pool`、`mcp_pool_restart`、`workspace_settings`、`rate_limit`、`workspace_reload`。tag 在场 = 行为已开，client 据此分支而非试错。其余 55 个（如 `session_context_usage`、`non_blocking_prompt`、`auth_device_flow`）无条件 advertise。
- **`unstable_` 前缀**：`unstable_session_resume` 表示底层 ACP method 仍可能变形，client 不应 pin 其 shape。
- **多模式 tag**：`permission_mediation` 带 `modes: ['first-responder','designated','consensus','local-only']`（详 [§05 Security & Permission](./05-permission-auth.md)）；`mcp_guardrails` 带 `modes: ['warn','enforce']`。
- **协商语义**：daemon 不支持的 tag，client 灰掉对应 UI 优雅降级，不 throw。

---

## 二、顶层路由

| 方法与路径 | 用途 | 关键语义 |
|---|---|---|
| `GET /health` | 浅健康检查 | `{status:'ok'}`；loopback 且非 `--require-auth` 时认证前可达 |
| `GET /health?deep=1` | 深健康检查 | 加 `sessions` / `pendingPermissions` / `rateLimitHits` 计数；探针失败 `503 {status:'degraded'}` |
| `GET /capabilities` | 能力协商 | 见 §1.6 |
| `GET /demo` | 内置调试页 | 自包含单页 HTML，无外部依赖 |
| `/acp` | ACP transport | POST/GET/DELETE + WebSocket upgrade，详 §七 |

---

## 三、Session 路由

### 3.1 生命周期

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `POST /session` | 创建 / attach | body `cwd` 可省（用 `caps.workspaceCwd`）；跨 workspace → `400 workspace_mismatch`；cwd 长度上限 PATH_MAX | #3889 |
| `POST /session/:id/load` | 加载历史 session | ACP LoadSessionRequest 直通 | #4222 |
| `POST /session/:id/resume` | 恢复 session | 底层 `unstable_resumeSession`，tag 带 `unstable_` 前缀 | #4222 |
| `POST /session/:id/branch` | 从当前 session 分叉 | body `{name?}`；SSE `session_branched` | — |
| `DELETE /session/:id` | 关闭 + 删除 | tombstone 防 attach-after-close 竞态；SSE `session_closed` 广播 | #4240 |
| `POST /sessions/delete` | 批量删除 | body `{sessionIds: string[]}` | — |
| `POST /session/:id/detach` | client 解绑 | 不关 session，只解除本 client 关联 | — |
| `POST /session/:id/heartbeat` | client 存活心跳 | daemon 跟踪 `lastSeenAt`；rate-limit 豁免 | #4235 |

`:id` 校验：session 必须存在于 daemon 的 session map，否则 `404`。1 daemon = 1 workspace；多 workspace = 多 daemon 进程各占端口。

### 3.2 Prompt、取消与旁路操作

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `POST /session/:id/prompt` | 发起 turn（**非阻塞**） | 立即 `202 {promptId, lastEventId}`；body `prompt` = 非空 content block 数组；可选 `deadlineMs`（仅当 server 配置 `--prompt-deadline-ms` 时生效，只能缩短不能延长）；结果经 SSE `turn_complete` / `turn_error` 按 `promptId` 关联 | #4585 |
| `POST /session/:id/cancel` | 取消当前 turn | SSE 广播 `prompt_cancelled`（语义为"已请求取消"）；originator 的 SSE 断连同样触发取消 | #3889 |
| `POST /session/:id/btw` | 旁路提问（side question） | body `{question}`；单轮、无工具，不打断主 prompt 流 | #4610 |
| `POST /session/:id/shell` | daemon 端直接执行 shell（`!` 前缀） | **显式 opt-in**：需 `qwen serve --enable-session-shell` 且已配置 bearer token 才生效，调用方 clientId 必须已绑定目标 session（REST / ACP `_qwen/session/shell` / bridge 三入口同一策略）。body `{command}`，bypass LLM；输出走 SSE `user_shell_command` / `user_shell_result`；命令与结果注入 LLM history | #4576 #5031 |
| `POST /session/:id/recap` | "我做到哪了"一句话摘要 | fast model 侧查询；`{sessionId, recap}`，`recap` 可为 `null`（历史过短/瞬时失败，仍 200） | — |
| `POST /session/:id/tasks/:taskId/cancel` | 取消 background task | strict gate | — |
| `POST /session/:id/goal/clear` | 清除 session goal | strict gate | — |

**202 协作模式**：client 先拿 `lastEventId` 作为订阅起点再开 SSE，保证 prompt 期间事件不丢帧；旧的 blocking 客户端若 await 成功 body 会把 202 误判为失败，SDK 已改为 await `turn_complete` by `promptId`。

### 3.3 事件订阅与权限投票

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `GET /session/:id/events` | SSE 事件流 | `Last-Event-ID` 重放；`?maxQueued=N`（[16, 2048]，默认 256）；`?snapshot=1` 重放完发 `session_snapshot`；每 session 订阅者上限 64，超出 `429 subscriber_limit_exceeded` + `Retry-After`（4xx 让 EventSource 终止重连，避免放大负载） | #3889 |
| `POST /session/:id/permission/:requestId` | session 级权限投票 | bounded record：同 sessionId 同 requestId 只能投一次；输者收 `permission_already_resolved` | #4232 |
| `POST /permission/:requestId` | 顶层权限投票 | 同上，无 session 前缀的早期形态，保留兼容 | #3889 |

权限流为异步 first-responder 模型（agent 发 SSE `permission_request` → 任意 client POST 应答 → `permission_resolved` 广播），多客户端仲裁策略见 [§05 Security & Permission](./05-permission-auth.md)。

### 3.4 配置、元数据与状态查询

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `POST /session/:id/model` | 切换模型 | SSE `model_switched` / `model_switch_failed` | #3889 |
| `POST /session/:id/approval-mode` | 切审批模式 | strict gate；`{mode, persist?}`，`persist:true` 同时写 workspace settings；SSE `approval_mode_changed` | #4250 |
| `POST /session/:id/language` | 运行时语言切换 | `{language, syncOutputLanguage?}`，白名单校验（`caps.supportedLanguages`） | #4705 |
| `PATCH /session/:id/metadata` | 改 session 元数据 | `{displayName}`；SSE `session_metadata_updated` | — |
| `GET /session/:id/context` | 上下文状态 | — | — |
| `GET /session/:id/context-usage` | token 使用分布 | `?detail=true` 展开明细；web-shell 渲染用 | #4573 |
| `GET /session/:id/stats` | session 统计 | — | — |
| `GET /session/:id/supported-commands` | 可用 slash command | — | — |
| `GET /session/:id/tasks` | background task 快照 | 走 bridge status 旁路，**不排 prompt FIFO**——prompt streaming 中可查 | #4578 |
| `GET /session/:id/hooks` | session 级 hooks | — | — |

### 3.5 Rewind

| 方法与路径 | 用途 | 关键语义 |
|---|---|---|
| `GET /session/:id/rewind/snapshots` | 列出可回退快照 | — |
| `POST /session/:id/rewind` | 回退到快照 | strict gate；SSE `session_rewound` |

---

## 四、Workspace 路由

### 4.1 环境、设置、init 与 preflight

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `GET /workspace/env` | env 诊断 | 诊断 cell 用 `errorKind` 封闭分类 | #4247 |
| `GET /workspace/preflight` | preflight 诊断 | 同上（`stat_failed` 等） | #4247 |
| `GET /workspace/settings` | 读 settings | — | — |
| `POST /workspace/settings` | 写单个 settings key | strict gate；`{scope, key, value}`；条件 tag `workspace_settings`；SSE `settings_changed` | — |
| `POST /workspace/reload` | settings 热重载 | strict gate；diff settings keys 只刷真变化的 idle session，不重启 daemon；SSE `settings_reloaded`；条件 tag `workspace_reload` | #4965 |
| `POST /workspace/reload-env` | env 热重载（`.env` / `settings.env`） | strict gate；boot 时快照全部文件 key，支持正确删除已移除的 key；刷新所有 idle session 的认证（新 API key 即时生效）；SSE `env_reloaded`；SDK `DaemonClient.reloadEnv()` | #4924 |
| `POST /workspace/init` | 脚手架 `QWEN.md` | strict gate；`{force?}`，已存在默认 `409 workspace_init_conflict`；**纯机械创建不调 LLM**；SSE `workspace_initialized` | #4250 |
| `GET /workspace/:id/sessions` | session 列表（分页） | `:id` = URL-encoded 绝对 workspace 路径，与绑定 workspace 不符 → `400 workspace_mismatch`；`?cursor=&size=` cursor 分页，非法 cursor → `400 invalid_cursor` | #4902 |

### 4.2 Tools、skills、extensions、hooks、providers

| 方法与路径 | 用途 | 关键语义 |
|---|---|---|
| `GET /workspace/tools` | 工具清单 | — |
| `POST /workspace/tools/:name/enable` | 启/禁工具 | strict gate；body `{enabled: boolean}`（**没有独立 disable 路由**）；直接写 settings `tools.disabled` 列表，SSE `tool_toggled`；live session 已注册工具不追溯卸载，下次 ACP child 启动生效 |
| `GET /workspace/skills` | 已加载 skill | — |
| `GET /workspace/extensions` | extensions 清单 | — |
| `GET /workspace/hooks` | workspace 级 hooks | — |
| `GET /workspace/providers` | 模型 provider 清单 | — |

### 4.3 Memory 与 Agents

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `GET /workspace/memory` | 读层级 QWEN.md 状态 | workspace + global（`~/.qwen`）两级 | #4249 |
| `POST /workspace/memory` | 写 memory | strict gate；`{scope: 'workspace'|'global', ...}` append/replace；SSE `memory_changed` | #4249 |
| `GET /workspace/agents` | 列 subagent | 包装 `SubagentManager`；built-in / extension agent 只读 | #4249 |
| `GET /workspace/agents/:agentType` | 读单个 agent 定义 | — | — |
| `POST /workspace/agents` | 创建 agent | strict gate；SSE `agent_changed` | #4249 |
| `POST /workspace/agents/:agentType` | 更新 agent | strict gate | — |
| `DELETE /workspace/agents/:agentType` | 删除 agent | strict gate | — |
| `POST /workspace/agents/generate` | LLM 生成 agent 定义 | strict gate；tag `workspace_agent_generate` | — |

### 4.4 MCP 管理

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `GET /workspace/mcp` | MCP server 状态快照 | 含 budget 计数；pool 开启时每 server cell 带 `entryCount` / `entrySummary` | #4241 |
| `GET /workspace/mcp/:server/tools` | 单 server 工具清单 | — | — |
| `POST /workspace/mcp/servers` | 运行时新增 server | strict gate；同 fingerprint no-op；SSE `mcp_server_added`；server 名校验（字母数字 `_-`、≤256、拒 `__proto__` 等保留名） | #4552 |
| `DELETE /workspace/mcp/servers/:name` | 运行时移除 server | strict gate；真移除才发 `mcp_server_removed`（`not_present` 幂等跳过不发） | #4552 |
| `POST /workspace/mcp/:server/restart` | 重启单 server | strict gate；`?entryIndex=N`（或 `*`）选 pool 内条目；budget 预检不通过返回 `200 {restarted:false, skipped:true, reason}`（`budget_would_exceed` / `in_flight` / `disabled`）而非级联拒绝；SSE `mcp_server_restarted` / `mcp_server_restart_refused` | #4251 |
| `POST /workspace/mcp/:server/enable` `…/disable` `…/authenticate` `…/clear-auth` | 单 server 启停 / OAuth | strict gate；同一 for 循环注册的 4 个动作路由 | — |

MCP budget 防护（`--mcp-client-budget` + `--mcp-budget-mode={warn,enforce,off}`）超阈值经 SSE `mcp_budget_warning` / `mcp_child_refused_batch` 推送。

### 4.5 Auth 与 device-flow

| 方法与路径 | 用途 | 关键语义 | 来源 |
|---|---|---|---|
| `GET /workspace/auth/providers` | provider 目录 | 每个带配置步骤描述（protocol / baseUrl / apiKey / models / advancedConfig） | — |
| `GET /workspace/auth/status` | 认证状态 | `{pendingDeviceFlows[], supportedDeviceFlowProviders[]}` | — |
| `POST /workspace/auth/provider` | 安装 / 切换 provider | strict gate；默认拒私网 baseUrl（防 SSRF），daemon 未实现安装时 `501` | — |
| `POST /workspace/auth/device-flow` | 发起 OAuth device flow（RFC 8628） | strict gate；secret 4 路 redaction（serialize / inspect / toString / JSON.stringify），凭证文件 0o600 | #4255 |
| `GET /workspace/auth/device-flow/:id` | 查询 flow 状态 | strict gate + 仅 initiator 可读验证材料 | — |
| `DELETE /workspace/auth/device-flow/:id` | 取消 flow | strict gate；`204` 幂等（已终态同样 204） | — |

flow 全程经 SSE 推送：`auth_device_flow_started` / `_throttled` / `_authorized` / `_failed` / `_cancelled`（workspace 级事件，非 session-keyed）。

---

## 五、File 路由（workspace 根限定）

路径不带 `/workspace` 前缀，挂在顶层；全部经 `WorkspaceFileSystem` 安全边界。

| 方法与路径 | 用途 | 关键参数 / 语义 |
|---|---|---|
| `GET /file` | 读文本文件 | `?path&line&limit&maxBytes`；`limit` ≤ 2000 行，`maxBytes` ≤ 256 KB |
| `GET /file/bytes` | 读原始字节窗口 | `?path&offset&maxBytes`（默认窗口 64 KB）；独立 tag `workspace_file_bytes` |
| `POST /file/write` | 写文件（原子） | `{path, content, mode: 'create'|'replace', expectedHash?, bom?, encoding?, lineEnding?}`；**`replace` 必带 `expectedHash`**（`sha256:<64hex>` content-hash 前置），`create` 撞已存在文件 → `409 file_already_exists`；写入 temp+rename 原子落盘；`201`（created）/ `200`（replaced）返回新 `hash` |
| `POST /file/edit` | 文本替换编辑 | `{path, oldText, newText, expectedHash}`（必带）；`text_not_found` / `ambiguous_text_match` 拒绝 |
| `GET /list` | 列目录 | `?path&includeIgnored` |
| `GET /stat` | 文件元信息 | `?path` |
| `GET /glob` | glob 匹配 | `?pattern&cwd&maxResults&includeIgnored`（maxResults 默认 5000） |

**安全边界**（#4282 / #4269 / #4280）：

- 路径 realpath 规范化后必须落在 workspace 根内：越界 → `400 path_outside_workspace`；symlink 链逃逸 / 循环 → `400 symlink_escape`
- 默认遵守 `.gitignore` / `.qwenignore`（`includeIgnored=true` 显式越过）
- 并发写防护：`expectedHash` 不匹配 → `409 hash_mismatch`（client 先读拿 hash 再写）
- audit hook 记录 `originatorClientId` + SHA-256 哈希后的路径
- 写路由 strict gate；读路由跟随全局 bearer

`FsErrorKind` 封闭 enum（独立于 §1.5 的 bridge enum，14 值，每值绑定默认 HTTP status）：`path_outside_workspace`、`symlink_escape`、`path_not_found`、`binary_file`、`file_too_large`、`hash_mismatch`、`file_already_exists`、`text_not_found`、`ambiguous_text_match`、`untrusted_workspace`、`permission_denied`、`io_error`、`internal_error`、`parse_error`。源码: `packages/cli/src/serve/fs/errors.ts`。

---

## 六、SSE 事件目录

### 6.1 帧结构与连接行为

```
event: session_update
id: 42
data: {"id":42,"v":1,"type":"session_update","data":{...},"originatorClientId":"...","_meta":{"serverTimestamp":...}}
```

- envelope `{id?, v: 1, type, data, originatorClientId?}`；synthetic 帧（如 `state_resync_required`、`replay_complete`）无 `id`，不占 per-session 单调序列
- 连接建立即发 `retry: 3000`（EventSource 断线 3s 重试）；每 15s 发 `: heartbeat` 注释帧探测死连接
- 写路径带背压：`res.write` 满则 await `drain`，全部写经 per-connection 串行链

### 6.2 重连与流控协议

| 机制 | 行为 |
|---|---|
| `Last-Event-ID` 重放 | 重连带此 header，从 ring buffer 续传 missed events（per-session ring 默认 8000 帧，`--ring-size` 可调） |
| `replay_complete` | 重放循环结束的哨兵帧（含 `replayedCount`，零重放也发），client 据此撤掉 catch-up 指示 |
| ring 淘汰 → `state_resync_required` | 请求的 id 已被淘汰时发出；**不断流**——daemon 继续重放幸存帧 + live 帧，但 reducer 应视积累状态为脏，先 `load` 重建再继续应用增量 |
| `?snapshot=1` | `replay_complete` 后补发 `session_snapshot`（`currentModelId`、`currentApprovalMode`），重连 client 免一次往返即可 seed reducer |
| `slow_client_warning` | 订阅队列 75% 满时的 soft 警告（队列深度 `?maxQueued=N`，[16, 2048]，默认 256） |
| `client_evicted` | 队列溢出（或 writer idle 超时，见 §八）后踢出该订阅者并断流，daemon 不再积压内存 |
| 订阅者上限 | 每 session 64 个，超出新连接 `429` + `Retry-After: 5` |

### 6.3 事件类型清单（43 个）

权威清单 `DAEMON_KNOWN_EVENT_TYPE_VALUES`，源码: `packages/sdk-typescript/src/daemon/events.ts`。

**Turn 流（8）**

| 事件 | 含义 |
|---|---|
| `session_update` | ACP `SessionNotification` 直通：LLM chunk / tool call / tool result / usage |
| `turn_complete` | prompt 成功结束，按 `promptId` 关联，含 `stopReason` |
| `turn_error` | prompt 失败，按 `promptId` 关联，含 `message` / `code`（deadline 超时即在此带 `prompt_deadline_exceeded`） |
| `prompt_cancelled` | 取消广播（显式 cancel 或 originator 断连），语义为"已请求取消" |
| `followup_suggestion` | 每个干净 assistant turn 后 server 端生成的 ghost-text 建议（#4507） |
| `user_shell_command` / `user_shell_result` | `POST /session/:id/shell` 的命令与流式输出 |
| `session_snapshot` | `?snapshot=1` 时重放后补发的状态快照 |

**权限（5）**

| 事件 | 含义 |
|---|---|
| `permission_request` | agent 请求审批，与 requestId 配对 |
| `permission_resolved` | first-responder 已应答，广播结果 |
| `permission_already_resolved` | 投票输者收到（bounded record） |
| `permission_partial_vote` | 仅 `consensus` 策略下：已收到部分票 |
| `permission_forbidden` | 投票者被策略拒绝（`designated` 非指定人 / `consensus` 匿名票 / `local-only` 远程票） |

**Session 生命周期与配置（8）**

| 事件 | 含义 |
|---|---|
| `model_switched` / `model_switch_failed` | 模型切换结果 |
| `approval_mode_changed` | 审批模式变更 |
| `session_metadata_updated` | `PATCH metadata` 之后广播；ACP 子进程记录 session title 时也即时广播 `{sessionId, displayName, titleSource}`（经 `qwen/notify/session/title-update` 侧信道，client 无需等列表轮询，#5035） |
| `session_closed` | `DELETE /session/:id` 之后广播（`reason: 'client_close'` 等） |
| `session_died` | ACP child 退出 → 该 daemon 全部 session 死亡 |
| `session_rewound` / `session_branched` | rewind / branch 完成 |

**流控与传输（5）**

| 事件 | 含义 |
|---|---|
| `slow_client_warning` / `client_evicted` | 见 §6.2 |
| `stream_error` | 流内错误帧 |
| `state_resync_required` / `replay_complete` | 见 §6.2 |

**Workspace 广播（13）**——经全部活跃 session 的 bus fan-out，informational（读后写仍以 read-after-write 为准）：

| 事件 | 触发 |
|---|---|
| `memory_changed` / `agent_changed` | memory / agents 写路由 |
| `tool_toggled` | tools enable 路由 |
| `settings_changed` / `settings_reloaded` | settings 写 / `POST /workspace/reload` |
| `env_reloaded` | `POST /workspace/reload-env` |
| `workspace_initialized` | `POST /workspace/init` |
| `mcp_server_added` / `mcp_server_removed` | 运行时 MCP 增删 |
| `mcp_server_restarted` / `mcp_server_restart_refused` | MCP 重启 |
| `mcp_budget_warning` / `mcp_child_refused_batch` | MCP budget 越线（rate-limited push） |

**Auth device-flow（5）**：`auth_device_flow_started` / `_throttled` / `_authorized` / `_failed` / `_cancelled`（workspace 级，session reducer no-op）。

**向前兼容**：SDK `narrowDaemonEvent()` 对未知 `type` 返回 `kind: 'unknown'` 而非 throw——旧 SDK 静默丢弃新 daemon 的新事件，无需协议版本升级。

---

## 七、ACP transport（`/acp`）

REST 之外的标准协议入口，与 REST 共享同一 bridge / EventBus（additive，REST 面不受影响）。`QWEN_SERVE_ACP_HTTP=0` 可整体关闭。

### 7.1 ACP Streamable HTTP

单一 `/acp` 端点上的 wire shape：

| 请求 | 行为 |
|---|---|
| `POST {initialize}` | `200` + capabilities JSON + `Acp-Connection-Id` 响应头 |
| `POST {其他 method}` | `202`；回复经长存 SSE 流异步交付 |
| `GET` + `Acp-Connection-Id` 头 | connection 级 SSE 流 |
| `GET` + connection + `Acp-Session-Id` 头 | session 级 SSE 流 |
| `DELETE` | `202` 拆除 connection |

connection 级 SSE 断开后有 10s 宽限期（重连可续），超时回收 `ownedSessions` 与连接槽位。

### 7.2 ACP WebSocket

同一 `/acp` 路径 HTTP upgrade（#4773）。WS 绕过 Express 中间件栈，因此在 upgrade 握手自行完成三件事：bearer 校验（`Authorization` 头，SHA-256 比较；loopback 无 token 部署放行）、Host 校验（防 DNS rebinding）、消息级 rate-limit（读类 method 计入 read 档；`heartbeat` / `update_metadata` 豁免）。实现为 transport 无关接口 `transportStream.ts` + `wsStream.ts` / `sseStream.ts` 两个 adapter——同一 dispatch 代码服务两种流。SSE 与 WebSocket **共存**，不互相替代。

### 7.3 `_qwen/*` extension namespace 与 REST parity

`/acp` dispatch 处理 9 个标准 ACP method（`authenticate`、`session/new|load|resume|list|close|cancel|prompt|set_config_option`）+ **41 个 `_qwen/*` vendor method**（其中 29 个由 parity 工作 #4827 补齐，使 ACP-native client——Zed / Goose / IDE——能做 REST 能做的全部）：

| 分组 | method（`_qwen/` 前缀省略） |
|---|---|
| session 基础 | `session/heartbeat` `session/context` `session/supported_commands` `session/update_metadata` |
| session 扩展 | `session/recap` `session/btw` `session/shell` `session/detach` `session/context_usage` `session/tasks` `sessions/delete` |
| workspace 状态 | `workspace/mcp` `workspace/skills` `workspace/providers` `workspace/env` `workspace/preflight` `workspace/tools` `workspace/mcp/tools` |
| workspace 变更 | `workspace/init` `workspace/set_tool_enabled` `workspace/restart_mcp_server` `workspace/mcp/servers/add` `workspace/mcp/servers/remove` |
| memory | `workspace/memory` `workspace/memory/write` |
| file | `file/read` `file/read_bytes` `file/stat` `file/list` `file/glob` `file/write` `file/edit` |
| auth | `workspace/auth/status` `workspace/auth/device_flow/start` `workspace/auth/device_flow/get` `workspace/auth/device_flow/cancel` |
| agents | `workspace/agents/list` `workspace/agents/get` `workspace/agents/create` `workspace/agents/update` `workspace/agents/delete` |

### 7.4 MCP stdio 桥（`qwen-serve-bridge`）

`sdk-typescript` 内置 bin（#4555）：把 daemon 的 REST+SSE 面包装成 **MCP stdio server**，MCP-native client（Claude Code、Cursor 等）零改动把整个 qwen daemon 当一个 MCP server 用。源码: `packages/sdk-typescript/src/daemon-mcp/serve-bridge/`。

---

## 八、运行时防护 flag 对 API 行为的影响

| flag / env | 影响 | 对应 capability tag |
|---|---|---|
| `--rate-limit`（+ `--rate-limit-prompt/-mutation/-read/-window-ms`） | opt-in 令牌桶（连续滴漏补充），默认 off 保持向后兼容。三档（按 `60s` 窗口默认）：**prompt 10 / mutation 30 / read 120**；按 client id（缺省退 IP）计 bucket；超限 `429` + `Retry-After` + `X-RateLimit-*` 头。豁免：OPTIONS、`/health`、`/demo`、heartbeat、SSE events、`/acp`（WS 消息在 §7.2 内部计费）。limiter 内部错误 fail-open。命中计数暴露在 `/health?deep=1` | `rate_limit`（#4861） |
| `--prompt-deadline-ms` / `QWEN_SERVE_PROMPT_DEADLINE_MS` | server 端 wallclock 上限。**非阻塞 202 模型下到期不再返回 HTTP 504**——deadline timer abort 进行中的 turn，错误以 SSE `turn_error`（`prompt_deadline_exceeded`）按 promptId 交付。per-request body `deadlineMs` 只能缩短上限；flag 未配置时 body 字段无效。上限 2^31-1 ms（boot 校验防 Node timer 溢出） | `prompt_absolute_deadline` |
| `--writer-idle-timeout-ms` / `QWEN_SERVE_WRITER_IDLE_TIMEOUT_MS` | 每条 SSE 连接的写空闲上限：n ms 内无成功 flush（heartbeat 或真事件都算）→ 发终结帧 `client_evicted {reason:'writer_idle_timeout'}` 并断流。生产建议 ≥30000ms——低于 15s heartbeat 间隔会在首个心跳前误踢健康连接 | `writer_idle_timeout` |
| `--allow-origin <pattern>...` | 放行匹配 Origin 的浏览器请求（默认全拒）；`*` 必须配 token；配置的 pattern 列表**不回显**在 capabilities（防 recon 枚举可信 origin） | `allow_origin` |
| `--require-auth` | 全部路由（含 `/health`）强制 bearer；strict/non-strict gate 区别消失；boot 无 token 拒绝启动 | `require_auth` |
| `--ring-size` | per-session 事件 ring 容量（默认 8000），影响 `Last-Event-ID` 可重放窗口 | — |
| `QWEN_SERVE_NO_MCP_POOL=1` | 关闭 workspace 级 MCP 连接池，回退 per-session client；`mcp_workspace_pool` / `mcp_pool_restart` 两个 tag 同步消失 | `mcp_workspace_pool` |

这组 flag 全部走"**条件性 capability tag**"模式：flag 开 → tag 在场 → client 才启用对应行为（如发 `deadlineMs`、预期 `writer_idle_timeout` 踢出），符合 v1 additive 规则。

---

## 九、Wire 兼容性与版本演进

> **术语**：**wire** = 两端点间实际传输的字节流；**schema** = zod 契约。"wire 字节级一致" = 不同 transport 序列化 bit-for-bit 相同；"不出 wire" = 仅 daemon 内处理不暴露给 client（详 [§04 Deployment & Client](./04-deployment-and-client.md)）。

### 9.1 单进程 vs daemon：4 层兼容矩阵

| 层 | 单进程（`qwen --acp` stdio） | Daemon（`qwen serve`） | 兼容性 |
|---|---|---|---|
| **Schema 层** | ACP zod schema | **复用同一组 schema**（HTTP body = `PromptRequest` 等同构） | 100% |
| **Wire 层** | stdio NDJSON | HTTP + SSE / WebSocket | 字节级不兼容 |
| **业务逻辑层** | `Session.handleXxx()` 直接调用 | **同一函数**（HTTP body 解为 ACP request 后调用） | 100% 同源 |
| **SDK 抽象层** | `ProcessTransport` | `DaemonClient` / `DaemonSessionClient` | 用户代码 0 改动 |

协议层 0 设计成本：ACP 已验证过 session 生命周期 / permission / cancel / resume，daemon 唯一新增的是传输层桥接。

### 9.2 双向 RPC 的异步化

ACP 本质是双向 RPC（agent 也能向 client 发起 `requestPermission` / `readTextFile`），但 HTTP 只有 client→daemon 一个方向。daemon 把 agent→client 方向全部改写为 **SSE event + client 回投**：

```
stdio:  agent ──requestPermission──→ client（同步往返）
HTTP:   agent → SSE permission_request → 任一 client POST /permission/:requestId → 解锁
```

挂起请求带超时与默认拒绝兜底，详 [§05 Security & Permission](./05-permission-auth.md)。

### 9.3 Additive 演进规则（`v: 1` 不破坏）

- 旧 route 不移除，旧 event envelope 不破坏；新能力一律 additive 注册 + capability tag
- client 通过 tag 决定启用新 UI，缺 tag 优雅降级，**不能因为 daemon 缺新能力而崩溃**
- SDK 对未知事件 type 返 `kind: 'unknown'`（§6.3），对未知 capability tag 忽略——双向向前兼容
- 例外窗口：`unstable_` 前缀 tag（如 `unstable_session_resume`）对应的 shape 仍可能变化

---

> **免责声明**: daemon 功能集已随 v0.18.0 正式版发布（2026-06-12）；`unstable_` 前缀路由 / capability 的 shape 仍可能变化。本篇路由与事件清单核实自 qwen-code main 分支源码（`packages/cli/src/serve/`、`packages/acp-bridge/src/`、`packages/sdk-typescript/src/daemon/`），截至 2026-06-13，后续版本可能增删。

下一篇：[04 — Deployment & Client →](./04-deployment-and-client.md)
