# 04 — Deployment & Client Experience

> [← 上一篇：HTTP API & Protocol](./03-http-api.md) · [下一篇：Security & Permission →](./05-permission-auth.md)

## TL;DR

daemon 已合入 main（[PR#4490](https://github.com/QwenLM/qwen-code/pull/4490)，随 v0.18.0 正式版发布，2026-06-12），形态只有一种：**`qwen serve` headless 长驻进程**，模型是 **1 daemon = 1 workspace × N session**；多 workspace = 多 daemon 进程。

客户端经三套北向协议 + 一座 MCP 桥接入，生态已落地多个真实 client：

- **REST + SSE**：web-shell（浏览器终端组件）、webui（共享 UI 层）、TypeScript / Python / Java SDK
- **ACP HTTP / WebSocket**（`/acp`）：Zed、Goose 等 ACP-native client；官方桌面 app 基于 ACP SDK
- **MCP 桥**（qwen-serve-bridge）：Qoder、Claude Desktop、Cursor 等 MCP host

两条铁律贯穿全篇：

1. **workspace 与 daemon 永远同机**——runtime 能力跟着 daemon host 走，不跟着 UI 走；"本地 workspace + 远端 daemon" 是反模式。
2. **本地单用户 TUI 永远 in-process**——`qwen` 默认直连 core，零网络栈，不走 daemon；daemon 只服务远端 / 跨进程 / 多 client 场景。

多 client 协调的基础设施（subscriber 注册、liveness、断线重放、first-responder 权限投票）已实现；active typer 提示与 takeover 属设计预留。

---

## 一、部署模式与拓扑

### 1.1 唯一形态：headless `qwen serve`

daemon 进程没有 TUI，由 systemd / pm2 / Docker 等进程管理器托管；重启后 session 不驻留内存，client 通过 `POST /session/:id/load` / `resume` 从 transcript 重建。早期还设计过 "TUI 进程内嵌 HTTP server" 的 Mode A（`qwen --serve`），已搁置，背景详 [02 — Architecture Decisions](./02-architectural-decisions.md)。

北向接口一览（协议细节详 [03 — HTTP API & Protocol](./03-http-api.md)）：

| 接口 | 端点 | 典型 client | 出处 |
|---|---|---|---|
| REST + SSE | `/session/*`、`/workspace/*`、`GET /session/:id/events` | web-shell / SDK / 自研 client | #3889 起 |
| ACP HTTP | `POST /acp`（JSON-RPC over HTTP）| ACP-native client | #4472 |
| ACP WebSocket | `WS /acp`（同一端点协议升级）| 同上，双向长连接 | #4773 |
| MCP 桥 | qwen-serve-bridge：client 侧 stdio MCP server ↔ daemon REST | 任意 MCP host | #4555 |

`POST /session/:id/prompt` 是非阻塞的：立即返回 `202 {promptId}`，结果经 SSE `turn_complete` / `turn_error` 按 promptId 关联（#4585）。

### 1.2 进程模型：1 daemon = 1 workspace × N session

- workspace 在 daemon 启动时绑定（cwd），生命周期内不变；文件访问被限定在 workspace 根内。
- N 个 session 在 daemon 内多路复用，同一 session 可被多个 client 同时订阅（见[§三](#三多-client-协调)）。
- **多 workspace = 多 daemon 进程**。OS 进程级隔离直接映射运维单元：一个 workspace 对应一个 systemd template unit / 一个 Docker container / 一个 K8s pod，无需 daemon 内部做租户隔离。

落地示例（1 workspace : 1 运维单元）：

```bash
# systemd template：%i 即 workspace 名，一实例一 workspace
systemctl start qwen-serve@repo-a    # WorkingDirectory=/srv/ws/repo-a
systemctl start qwen-serve@repo-b    # WorkingDirectory=/srv/ws/repo-b

# Docker：一容器一 workspace；绑非 loopback 时必须带 token（无 token 拒绝启动）
docker run -d -v /srv/ws/repo-a:/workspace -w /workspace -p 4170:4170 \
  <qwen-code-image> qwen serve --hostname 0.0.0.0 --port 4170 \
  --token "$QWEN_TOKEN" --require-auth
```

常用 flag 还有 `--max-sessions`、`--allow-origin`、`--rate-limit`、`--session-idle-timeout-ms` 等（源码: `packages/cli/src/commands/serve.ts`）；快速上手详 [07 — 用户使用文档](./07-user-guide.md)。

### 1.3 拓扑铁律：workspace 与 daemon 同机

> "Runtime capabilities follow the daemon/runtime host, not the visual client."（设计讨论出处：#3803）

UI 跑在哪台机器不重要，**runtime（fs / shell / MCP / 凭证）永远在 daemon host 上 evaluate**。由此得出三种基本拓扑的取舍：

| 拓扑 | workspace | daemon | 判定 |
|---|---|---|---|
| Local-Local | 本机 | 本机 | ✅ 单人开发默认 |
| Remote-Remote | 远端（与 daemon 同机）| 远端 | ✅ 推荐的远程形态，类比 GitHub Codespaces / Coder |
| Local-Remote | 本机 | 远端 | ❌ 反模式——daemon 看不到本地 fs / MCP / skills / env，只能靠 mount 或 sync 续命 |

### 1.4 六种部署 shape

按 client 与 runtime 的位置组合，常见形态六种（出处：#3803）：

| # | Shape | Client | Daemon + workspace | 说明 |
|---|---|---|---|---|
| 1 | 本地单用户 TUI 🌟 | `qwen` TUI（in-process）| **不存在 daemon** | 零网络栈，永久默认 UX（见[§2.3](#23-本地-tui永远-in-process)）|
| 2 | 本机多 client 协作 | IDE / 本地浏览器 / SDK | 本机 loopback `qwen serve` | 跨进程协作才需要 daemon；loopback 默认免 token |
| 3 | Web client - 远端 | 浏览器（web-shell）| 远端 devbox / pod | 推荐的云端形态；需 token + CORS / BFF（见[§2.6](#26-channel--web-bff-安全边界)）|
| 4 | 本地 IDE - 本机 daemon | IDE extension | 本机 loopback | IDE 负责发现/启动 daemon；workspace 必须与 IDE 打开的目录一致 |
| 5 | Channel + daemon | IM bot backend | 本机或远端 | 手机 IM 控制本机 workspace 是**合法远控形态**：控制入口远端化，runtime 仍在本机，不违反铁律 |
| 6 | Remote-control overlay | web / mobile 控制面 | 本机 daemon + relay，或远端 daemon + gateway | 后置；定位为 daemon facade，复用同一套 client contract，不另造 runtime 协议 |

两点补充：

- **Web client 是 daemon-native renderer，不是 PTY proxy**。web-shell 消费 typed events + reducer 渲染（与桌面/IDE client 同一 contract）；代理 terminal 字节流的 PTY proxy 只适合 demo / debug，不是目标架构——它会重新耦合进程生命周期并绕开 typed event contract。
- **Shape 6 的两种形态**只是部署与 routing 不同：本机 daemon + 出站 relay（保留本机 workspace、git state、MCP、凭证，daemon 可保持 loopback-only）；或远端 daemon + gateway（SaaS / devbox，平台负责租户隔离与配额）。两者都复用 SDK 的 daemon client 与 typed event contract。

### 1.5 远程开发的两条路径

| 路径 | 做法 | 体验 |
|---|---|---|
| SSH + 单进程 | SSH 进远端机器直接跑 `qwen` | 完整本地 TUI 体验，不涉及 daemon |
| daemon + 远端 client | 远端机器跑 `qwen serve`，本地用浏览器（web-shell）/ IDE / SDK attach | 多端同看一个 session、断线重连、长任务不随终端退出 |

两条路径都满足 "workspace 与 daemon（或 CLI）同机" 铁律，按是否需要多 client / 长驻特性选择即可。

---

## 二、客户端生态现状

### 2.1 已落地 client 一览（qwen-code main）

| Client | 位置 | 接入方式 | 说明 | 出处 |
|---|---|---|---|---|
| web-shell | `packages/web-shell` | REST+SSE（经 SDK + webui）| 浏览器终端 UI，React 组件可嵌入任意应用；daemon 第一个真实 web 消费者。已支持图片上传回显（#4922）、goal/task 工作流 + task 审批（#4856）、内联 `/settings` 面板与 context 用量 UI（#4944/#4958）、与原生 CLI 对齐的 tool_group/thinking 渲染（#4975/#4977）、SSE 断线重连加固（#4952） | #4380 |
| webui | `packages/webui` | —（共享层）| daemon UI 共享层：typed events → transcript blocks → React bindings（`daemon-react-sdk`）| #4328 / #4353 |
| 官方桌面 app | `packages/desktop` | ACP SDK（捆绑 qwen-code runtime）| Electron 桌面 app；发布链已接 mac 签名 + 公证（App Store Connect API key）与 Windows 自动更新 | #3778 / #5013 / #5028 |
| TypeScript SDK | `packages/sdk-typescript` | REST+SSE | `DaemonClient` / `DaemonSessionClient` typed client；web-shell / webui / IDE spike 的共同底座 | — |
| Python / Java SDK | `packages/sdk-python` / `sdk-java` | REST+SSE | 程序化接入 daemon | — |
| qwen-serve-bridge | `packages/sdk-typescript/src/daemon-mcp/serve-bridge` | stdio MCP ↔ daemon REST | 把 daemon 包装成 MCP server，Qoder / Claude Desktop / Cursor 等 MCP host 即插即用 | #4555 |
| ACP-native client | daemon `/acp` 端点 | ACP HTTP / WebSocket | Zed、Goose 等；29 method parity | #4472 / #4773 / #4827 |
| `/demo` 调试页 | daemon 内置 | REST+SSE | 单文件 HTML、零依赖、免构建，开发调试用 | — |

> 注：仓库内置的 zed-extension 目前仍 spawn `qwen --acp`（stdio 单进程形态）；`/acp` HTTP/WS 端点的意义在于让 ACP-native client 也能 attach **长驻** daemon，而不是每次对话起一个新进程。

### 2.2 复用分层：SDK → webui → web-shell

web 系 client 不是各写一套 HTTP 拼接，而是三层复用（#4328 / #4353 / #4380）：

```text
@qwen-code/sdk        DaemonClient / DaemonSessionClient
                      （typed HTTP+SSE client：prompt / events / permission / model …）
        ↓
@qwen-code/webui      共享 daemon UI 层
                      typed events → transcript blocks（reducer）→ React bindings
                      （DaemonSessionProvider / DaemonWorkspaceProvider / useStreamingState …）
        ↓
@qwen-code/web-shell  浏览器终端组件（开箱即用，亦可只用底层自行渲染）
```

两个要点：

- client 共享的是 **typed event contract + reducer**，不是 daemon 内部的 EventBus——`GET /session/:id/events` 是 EventBus 的对外 SSE 投影，client 永远隔着 wire。
- 自研 client 应从 SDK 或 webui 层接入，拿到与官方 client 完全一致的转写状态机；裸拼 REST 只在 SDK 未覆盖的语言里才有必要。

### 2.3 本地 TUI：永远 in-process

**`qwen` 默认（无 serve）= 传统单进程，TUI 直接 import core，零网络栈。** 这不是尚未迁移的过渡状态，而是明确的**永久设计**，也是 daemon 项目中优先级最高的 UX 约束：

- **启动延迟**：直起 TUI vs 先握手 loopback HTTP；
- **心智模型**：单 binary 单进程 vs client/server 分层；
- **代码路径**：TUI 直调 core/Config vs 所有 dialog 都要 wire 到 HTTP 路由；
- **与 `qwen --acp` stdio 形态对齐**：单 binary 心智不被 daemon 分层破坏。

> 🌟 **设计原则：不要为你不需要的东西付费。** 本地单用户场景不需要 port / token / discovery / 进程生命周期管理，就不应承担这些代价。

需要 daemon 的场景是清晰的另一类：远端 client、跨进程多 client 协作（IDE + 浏览器同看一个 session）、channel bot、web 服务。本地 TUI 不在此列，也没有把它默认迁移到 daemon 的计划。

### 2.4 尚未接入 / 进行中的 client

| Client | 现状 |
|---|---|
| 本地 TUI 的 daemon-client adapter | 未落地。默认路径永远 in-process（上节原则）；"TUI attach 远端 daemon" 的 opt-in adapter 属未来工作 |
| channels（Telegram / 钉钉 / 飞书 / 微信）| 适配后置。`DaemonChannelBridge` 原语已进 `packages/channels/base`，但各 channel runtime 尚未接线 daemon transport，默认仍 in-process |
| VS Code companion | daemon-backed connection spike 已在源码（`packages/vscode-ide-companion/src/services/daemonIdeConnection.ts`，复用 TS SDK），默认连接方式未切换 |
| remote-control 系 | 后置。定位改为 daemon facade（Shape 6）：复用 daemon client contract 做控制面，不再自带平行 runtime / event log |

### 2.5 TUI 与 wire 的边界：dialog 成本分析

daemon 路线早期最大的质疑是："TUI 十几个交互 dialog（`/memory`、`/mcp`、`/auth`……）都是进程内 UI，wire 化成本不可接受。" 逐项成本分析推翻了这个判断——绝大多数 dialog 本质是 **state CRUD**，每项约 0.5~1 天就能变成 HTTP 路由。这些路由现已全部在 main：

| TUI dialog | wire 化 | daemon 路由（main 已实现）| 出处 |
|---|---|---|---|
| `/memory` 编辑 | ✅ | workspace memory CRUD | #4249 |
| `/agents` 管理 | ✅ | workspace agents CRUD | #4249 |
| `/mcp` 查看/重启 | ✅ | `GET /workspace/mcp` + `POST /workspace/mcp/:server/restart` | #4250 |
| `/tools` 启停 | ✅ | `POST /workspace/tools/:name/enable` | #4250 |
| `/approval-mode` | ✅ | `POST /session/:id/approval-mode` | #4250 |
| `/init` | ✅ | `POST /workspace/init` | #4250 |
| `/resume` | ✅ | `POST /session/:id/load` / `resume` | — |
| model / session 选择 | ✅ | `POST /session/:id/model`、`GET /workspace/:id/sessions` | — |
| `/auth` OAuth 登录 | ✅ | RFC 8628 device-flow 由 daemon 代办，凭证不经过 client | #4255 |
| `/ide` IDE 集成 | ⚠️ 本地特例 | 语义取决于 "IDE 在哪台机器"，无通用 wire 化方案 | — |

源码: `packages/cli/src/serve/server.ts`。结论：control-plane parity 已不是远端 client 的障碍，远端 UI 能完成除 `/ide` 外的全部管理操作——这也是 web-shell 能作为完整客户端存在的前提。

### 2.6 Channel / Web BFF 安全边界

daemon 对浏览器的默认姿态是**拒绝**：识别到跨源 browser `Origin` 的请求直接拒掉（deny-wall）。放行只有两条显式途径：

```text
✅ 默认允许:
   Channel bot backend  → qwen serve          （服务端直连）
   Browser → Web BFF    → qwen serve          （服务端 BFF 代理，token 留在服务端）
   Browser → daemon 自托管页面（/demo 等）      （同源，无跨源 Origin）

✅ 显式开门（运维决定）:
   Browser → qwen serve                        （--allow-origin 白名单命中才放行，#4527）

❌ 其余一律拒绝:
   Browser → qwen serve                        （跨源 Origin 默认 deny）
```

部署 web-shell 时对应两种形态：

- **可信 / 单用户环境**（本机、内网 devbox）：`--allow-origin` 放行宿主页面源，浏览器带 Bearer token 直连。
- **多用户 / 公网**：必须走宿主应用的 BFF——**daemon token 永远不进不受信的前端代码**，浏览器只认 BFF 的会话。

**Session isolation**：daemon 默认 `sessionScope: 'single'`（session 可共享）；channel / web BFF 服务多个互不相关的用户时，应在 `POST /session` 显式声明 `sessionScope: 'thread'` 获得严格隔离（#4209）。**不要把不相关用户的会话静默复用到同一个 daemon session**——那是对话上下文污染与隐私越界。更粗粒度的隔离用进程模型解决：每用户 / 每租户一个 daemon。

### 2.7 事件消费契约（自研 backend 参考）

用 SDK / webui 接入的 client 由现成 reducer 处理事件；直接消费 SSE 的自研 backend（BFF、channel bot、非 JS 语言）按下表映射（事件协议详 [03 — HTTP API & Protocol](./03-http-api.md)）：

| Daemon SSE event | client / BFF 处理 |
|---|---|
| `session_update`（`agent_message_chunk`）| 追加 assistant 文本 |
| `session_update`（`agent_thought_chunk`）| 可选 thinking 流，默认可隐藏 |
| `session_update`（`tool_call` 等）| 渲染 tool 状态卡片 / 消息 |
| `permission_request` / `permission_resolved` | 弹出审批交互 / 关闭审批 |
| `turn_complete` / `turn_error` | 按 `promptId` 收尾对应 turn |
| `model_switched` | 更新会话元数据 |
| `state_resync_required` | 全量重同步（见[§3.3](#33-attach--reconnect-原则)）|
| `client_evicted` | 当前连接已被剔除：提示用户并重连 |
| `session_died` | 通知用户、停止流 |
| **未知 event type** | **忽略或记 debug，绝不 fatal**——daemon 会持续新增事件类型 |

---

## 三、多 Client 协调

同一 session 允许多个 client 同时 attach：事件向所有订阅者 fan-out，prompt 按 FIFO 串行，权限请求谁先应答算谁的。总原则是**协调 > 排斥**（live collaboration first）——不靠锁把第二个 client 挡在门外。

### 3.1 已实现（main）

| 机制 | 行为 | 出处 |
|---|---|---|
| Subscriber 注册 | daemon 按 session 维护订阅者集合；默认每 session 上限 64，超限新 SSE 连接收 `429 subscriber_limit_exceeded` | 源码: `packages/cli/src/serve/server.ts` |
| SSE keepalive | 服务端每 15s 推 heartbeat 注释帧，探测 TCP 已死的连接 | 同上 |
| 即时剔除 | 连接关闭（`req.close`）立刻摘除订阅者，不等超时 | 同上 |
| 背压保护 | 每订阅者 bounded queue；慢消费者溢出时收终态 `client_evicted` 帧后被剔除 | 同上 |
| 主动心跳 / 离席 | `POST /session/:id/heartbeat` 续活、`POST /session/:id/detach` 显式离席 | 同上 |
| 断线重放 | SSE ring buffer + `Last-Event-ID` 重连重放；ring 已淘汰则推 `state_resync_required` 要求 client 全量重同步 | #4237 / #4360 |
| Client 身份 | client 经 `X-Qwen-Client-Id` 报身份，daemon 在事件上盖 `originatorClientId`——client 据此过滤自己动作的回声 | #4231 |
| 权限投票 | PermissionMediator 四策略：**first-responder**（默认，先应答先得，其余应答返回已解决）/ designated / consensus / local-only（仅 loopback 连接可投）| #4335，详 [05 — Security & Permission](./05-permission-auth.md) |
| Prompt 串行 | 同 session prompt FIFO 排队；`202 {promptId}` 让每个 client 关联自己的 turn 结果 | #4585 |
| 状态广播 | `POST /session/:id/model` 等变更即时生效并广播（如 `model_switched`），所有 client 同步 | — |

### 3.2 设计预留（未实现）

以下协调能力在设计中讨论过，main 尚无实现，接入方不要假设其存在：

- **Active typer 提示**："X is typing..." 同侪提示与让出机制；
- **Takeover 流程**：检测到原 client 失联后，其他 client 显式接管会话控制权；
- **按 client 类型的配额 / exclusive 模式**：如 IM bot 单独限额、同类型 client 互斥；
- **按 client 类型差异化的空闲超时**（mobile / web / cli 不同宽限）。

当前语义下"接管"无需协议支持：session 不属于任何 client，新 client attach 后直接发 prompt 即可。

### 3.3 Attach / Reconnect 原则

远端 client 每次连接（含重连）都应把 daemon 状态当作**冷状态**重新获取：

| # | 原则 | 做法 |
|---|---|---|
| 1 | attach / reconnect 必须 re-fetch | 用 `Last-Event-ID` 重放补事件；用 `GET /capabilities`、`GET /session/:id/context` 等读路由拿终态 |
| 2 | 不假设所有变更都有事件推送 | approval-mode / memory / MCP 等管理面状态应视为 server-side opaque state，需要时主动查询 |
| 3 | 收到 `state_resync_required` 即全量重同步 | ring buffer 已淘汰你错过的事件，增量补不回来 |

---

## 四、Runtime locality 与环境契约

### 4.1 daemon 是 runtime owner

MCP、skills、shell、tool 执行、provider 凭证、文件访问——全部在 **daemon host / pod** 上 evaluate，与 client 所在机器无关：

```text
┌────────────────────────────────────────────────────────┐
│ daemon host / pod                                      │
│   ├─ filesystem（workspace / skills / settings）        │
│   ├─ network（MCP HTTP/SSE / provider API egress）      │
│   ├─ process execution（shell / MCP 子进程）            │
│   ├─ credentials（OAuth token / API key）               │
│   ├─ env vars（cloud CLI / docker / kubeconfig）        │
│   └─ Unix sockets / SSH agent / browser profile        │
└────────────────────────────────────────────────────────┘
              ↑ HTTP / SSE / WS
   ┌──────────┴───────────────┐
   │ client（web / IDE / SDK）│ ← 不拥有 runtime：只渲染、发 prompt、应答 permission
   └──────────────────────────┘
```

部署时的具体含义：

| # | 含义 | 例子 |
|---|---|---|
| 1 | stdio MCP server 在 daemon host spawn | daemon host 必须装有 `node` / `uv` / docker CLI 等二进制及其 env / secrets |
| 2 | HTTP MCP server 从 daemon host 访问 | daemon host 需要到 MCP endpoint 的 outbound egress |
| 3 | "本地资源" 指 daemon host 本地 | `localhost`、Unix socket、挂载卷、kubeconfig、SSH agent 都解析到 daemon host，不是 client 机器 |
| 4 | skills 必须在 daemon 文件系统 | client 机器上的 `~/.qwen/skills` 对远端 daemon 不可见 |
| 5 | 锁死 egress 的 pod 会静默失能 | 无出网策略时 MCP discovery / tool call 失败 |

### 4.2 这不意味着 daemon 必须开放公网

- daemon 只需要**已配置的 provider / MCP / skills 实际所需**的网络面；
- 生产推荐 deny-by-default egress + 显式 allowlist；
- 但要记住：**client 连得通某服务 ≠ daemon 连得通**——检查连通性永远站在 daemon host 视角。

### 4.3 部署诊断路由

为避免远端 client 遭遇 "tools 静默消失"，daemon 提供 daemon-host 视角的只读诊断（main 已实现，源码: `packages/cli/src/serve/server.ts`）：

| 路由 | 返回 |
|---|---|
| `GET /workspace/mcp` | 每个 MCP server 的 `status` + `errorKind`（missing binary / blocked egress / auth 错误 / init 超时等封闭分类）|
| `GET /workspace/skills` | 每个 skill 的加载状态与错误 |
| `GET /workspace/env` | daemon host 关键环境快照（secrets 脱敏）|
| `GET /workspace/preflight` | 启动与配置 readiness 整体检查：providers / MCP / skills / binaries / egress |

远端 client 排障第一步永远是打这几个路由，而不是检查自己机器的环境。

---

## 五、Client Capability 反向 RPC（设计预留）

> **状态**：设计预留，main 未实现（无 `capability_request` 事件）。记录于此作为接口演进方向。

daemon 拥有 runtime，但有些资源天然长在 client 机器上：编辑器、剪贴板、默认浏览器、系统通知、文件选择框。设计构想是一组**反向 RPC**：daemon 经 SSE 下发请求，client 在本地执行后经 HTTP 回执。

```text
1. agent 代码:   client.openEditor({ path: '/work/foo.py', line: 42 })
2. daemon → SSE: { type: 'capability_request', requestId: 'r1',
                   capability: 'open_editor', params: {...} }
3. client:       本地 spawn editor（code -g foo.py:42）
4. client → daemon: POST /capability/r1 { status: 'ok' }
5. daemon → agent:  { status: 'ok' }
```

| Capability | 用途 | client 端实现示意 |
|---|---|---|
| `open_editor` | 在 client 本地打开文件 / 跳转行号 | spawn `code` / `vim` 等 |
| `clipboard` | 读写 client 剪贴板 | pbcopy / xclip / clip.exe |
| `open_browser` | client 本地浏览器开 URL | `open` / `xdg-open` |
| `notification` | client 系统通知 | notify-send / osascript |
| `file_picker` | client 本地文件选择 | OS native dialog |

配套机制：client 连接时注册自己支持的 capability；多 client 同 session 时 daemon 优先派发给支持者；无人支持则向 agent 返回错误，由 agent 提示用户手动操作。

**Scope 边界**（与[§四](#四runtime-locality-与环境契约)互为补充）：这 5 类是**显式 delegate 给 client 本地资源**的能力，**不是** MCP / skills / shell 的 general fallback——"用 client 本地 docker 跑 MCP"、"读 client 本地 skills 目录" 都不在此列；那类需求若成立，需要独立设计。

---

> **免责声明**：本文基于 qwen-code main 分支源码（`packages/cli/src/serve`、`packages/web-shell`、`packages/webui`、`packages/sdk-typescript` 等）与公开 PR 记录整理，状态截至 2026-06-13（v0.18.0）。daemon 功能持续演进，路由与包结构可能变化，以源码为准。

> [← 上一篇：HTTP API & Protocol](./03-http-api.md) · [下一篇：Security & Permission →](./05-permission-auth.md)
