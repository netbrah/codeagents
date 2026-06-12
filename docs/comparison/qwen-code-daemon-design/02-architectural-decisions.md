# 02 — Architecture Decisions

> [← 上一篇：Overview](./01-overview.md) · [下一篇：HTTP API & Protocol →](./03-http-api.md)

## TL;DR

本篇回答"**为什么这样设计**"：`qwen serve` daemon 的 9 个核心架构决策，每条给出备选项、取舍依据与现状落点。最核心的是决策 2 状态进程模型——**1 daemon = 1 workspace × N session 多路复用**：与 `qwen --acp` stdio 的 1:1 心智对齐、OS 进程级隔离、cgroup quota 与 K8s 天然契合、blast radius 最小。贯穿所有决策的设计原则是"**不要为你不需要的东西付费**"（见文末 [设计原则](#设计原则--不要为你不需要的东西付费)）。

daemon 功能集已合入 qwen-code main（PR#4490，v0.18.0-preview 线），本篇所有"现状"均指 main 分支当前实现。

## 术语速查

完整术语表见 [01 — Overview](./01-overview.md)。本篇高频术语：

| 术语 | 含义 |
|---|---|
| **Daemon process** | `qwen serve` 进程；启动时绑定 cwd = 单 workspace |
| **Session** | daemon 内嵌 `qwen --acp` child 的 `QwenAgent.sessions: Map` 内一条会话 |
| **acp-bridge** | 独立包 `packages/acp-bridge`：HttpAcpBridge、EventBus、PermissionMediator 等 server 侧核心原语 |
| **fan-out** | 1 事件源 → N 订阅者的广播：1 个 session 的事件流推给该 session 全部订阅 client |
| **Runtime worker** | 真正执行 tool / shell / MCP / skills / 文件操作的运行时；当前是 `qwen --acp` child |

## 决策总表

| # | 决策 | 选择 | 出处 |
|---|---|---|---|
| 1 | session 跨 client 共享 | 默认 `sessionScope: 'single'`（attach 共享）；per-request `'thread'` 显式隔离 | #3889 |
| 2 | 状态进程模型 | **1 daemon = 1 workspace × N session 多路复用**；多 workspace = 多 daemon 进程 | #4113 |
| 3 | MCP 生命周期 | workspace 级共享 transport pool + per-session 视图 + 预算封顶 | #4336 |
| 4 | FileReadCache | per-session 严格私有 | #3717 / #3774 |
| 5 | Permission flow | 复用 in-process 决策核心；`ask` 外化为 SSE 事件；PermissionMediator 4 策略仲裁 | #3723 / #4335 |
| 6 | 多 client 并发 | 同 session prompt FIFO 串行 + 事件 fan-out + 跨 session 并行 | #3889 |
| 7 | 部署模式 | Mode B（headless `qwen serve`）主线；Mode A（`qwen --serve` 带 TUI）暂停 | #4156 |
| 8 | server / client / runtime 边界 | 4 层分离；acp-bridge 已物理拆包；daemon-native renderer 为目标形态 | #3803 |
| 9 | northbound transport | dual additive：自有 REST+SSE 与标准 ACP（HTTP + WebSocket）共存 | #4472 / #4773 |

---

## 决策 1 — session 是否跨 client 共享

### 背景：两种正交拓扑

"多 client"实际包含两种完全不同的需求（#3889 review 中厘清）：

| 拓扑 | 形态 | 典型场景 | 主要压力 |
|---|---|---|---|
| **P1 — 1:N**（multi-end sync）| 1 session × N client 订阅同一事件流 | 桌面 + 手机 mirror / Web UI attach / pair programming | EventBus fan-out |
| **P2 — N:1**（resource sharing）| N session × 1 用户在同 workspace 并行 | **IDE 多窗口**（不同分支 / 子目录）/ mobile app 多对话 | N session 共享同一 `qwen --acp` child |

**关键观察**：IDE 多窗口是 P2 不是 P1——多窗口是为了并行做不同的事，不是同一会话的多个视图（Cursor / Claude Code / OpenCode / Gemini CLI 均为单进程 N-session 模型）。daemon 必须同时服务两种拓扑。

### 选择

**默认 `sessionScope: 'single'`**：同 daemon 的多个 client 自动 attach 到现有 session（"first POST creates, subsequent POST attaches"）→ 覆盖 P1 live collaboration。attach 到现有 session 不受 `--max-sessions` 上限限制，空闲 daemon 不会拒绝老用户重连（源码: `packages/cli/src/serve/types.ts`）。

**Per-request `sessionScope: 'thread'` override**：显式新建隔离 session → 覆盖 P2。REST（`POST /session`）与 ACP transport（`session/new` params）均支持该参数（源码: `packages/cli/src/serve/server.ts`、`packages/cli/src/serve/acpHttp/dispatch.ts`）。

### 共享语义（`single` 模式）

| 操作 | 行为 |
|---|---|
| Client A 发 prompt | Client B 通过 SSE 看到完整事件流 |
| Client B 同时发 prompt | 同 session FIFO 串行——B 排队等 A 完成（决策 6）|
| A 触发 permission 请求 | 按 PermissionMediator 策略仲裁，默认任一 client 可应答（决策 5）|
| A 断开 | daemon 不受影响；其他 client 继续观察 |
| 所有 client 断开 | session 进入 idle，可被回收策略处理 |

### 安全与隔离边界

- ✓ **跨 workspace = 跨 daemon 进程 = OS 进程级隔离**
- ⚠️ **同 daemon 同 workspace 的多 client 互相可见**——这是有意设计（collaboration 是目标场景）
- ⚠️ **同 daemon 的 N session 共享 OS 权限**（同一 `qwen --acp` child、同 UID、同 workspace 文件视图）——`thread` scope 提供的是**会话状态隔离**，不是安全隔离

**多租户约束**：安全隔离必须在 daemon 进程层做——orchestrator 按 1 daemon = 1 tenant × 1 workspace 绑定。详 [05 — Security & Permission](./05-permission-auth.md) 与 [06 — Roadmap](./06-roadmap.md)。

---

## 决策 2 — 状态进程模型（核心决策）

### 选择

**1 daemon process = 1 workspace × N session 多路复用。**

`qwen serve` 启动时绑定 cwd 为唯一 workspace，内嵌单个 `qwen --acp` child；`QwenAgent.sessions: Map<sessionId, Session>` 提供 N session 多路复用。多 workspace 部署 = 多 daemon 进程。

```
qwen serve（绑定 cwd = /work/repo-a）
├─ Express HTTP front + bearer auth + Host allowlist
├─ EventBus（per-session fan-out + ring replay + Last-Event-ID 重连）
└─ qwen --acp child（workspace = /work/repo-a）
   └─ QwenAgent.sessions: Map → {sess-1, sess-2, sess-3}
```

**关键不变式**：

- **N session 共享**：同一 `qwen --acp` child 进程、workspace 上下文、MCP transport pool（决策 3）
- **N session 各自持有**：`Config` / `ToolRegistry` / `FileReadCache`（per-session 隔离单元，决策 4）
- **跨 workspace = 多 daemon 进程**：systemd `MemoryMax=` / cgroup / docker `--memory` 直接等于 per-workspace quota
- **crash 半径 = 1 workspace**：daemon 或其 child 退出时，该 workspace 全部 session 收 `session_died`；其他 daemon 不受影响
- **K8s 天然契合**：1 pod = 1 daemon = 1 workspace
- **与 `qwen --acp` stdio 1:1 心智对齐**：daemon 只是把 ACP stdio 包装成 HTTP，不引入中间路由抽象
- **容量参考**：默认 `--max-sessions 20`；per-session RSS 约 ~30-50MB，设计上 N≈50 后内存与 FD 压力显著（源码: `packages/cli/src/serve/types.ts`）

### 多 workspace 部署 = 多 daemon 进程

```bash
qwen serve --port 8001 --workspace /work/repo-a   # repo-b → :8002，repo-c → :8003 …
```

跨 workspace 的发现与路由由 client / orchestrator 侧负责，daemon 自身只管"当前 workspace"。

> **被否决的备选项**：daemon 内置 multi-workspace 路由层（`byWorkspaceChannel: Map` 等抽象）曾被实现过，后经 #4113 移除——它在 daemon 内重复了 OS 进程管理器已经做好的事（隔离、quota、生命周期），还破坏了与 ACP stdio 的 1:1 心智。代码中若见 `byWorkspaceChannel` / `ChannelInfo` / `Workspace Bridge` 均为已删除的遗留符号。

### 为什么不是"1 daemon 服务多 cwd"——ACP 三层现状

ACP 协议本身支持 per-session cwd，但 qwen-code 实现层用不了：

| 层 | multi-cwd 能力 | 实际情况 |
|---|---|---|
| ACP 协议 `newSession({cwd})` | ✅ 协议允许 per-call cwd | 设计意图允许 1 个 ACP child 服务多 cwd |
| `packages/channels/base/` | ✅ API 保留 per-session cwd 参数 | ❌ 实际只用 `config.cwd` 单值（源码: `packages/channels/base/src/ChannelBase.ts`）|
| `acpAgent.ts` 实现 | ❌ `this.settings = loadSettings(cwd)` 是 instance-wide，新 cwd 会覆盖旧值 | 跨 cwd 多 session 会互相污染 settings / hooks |

**结论**：实现层的真实约束就是"1 个 ACP child = 1 cwd × N session"——daemon 的 1 workspace 绑定与之 100% 同构。解除该约束需要把 settings/hooks 从 instance-wide 重构为 per-session，属远期工作。

### 与 OpenCode 的设计差异

OpenCode 默认 1 daemon 服务多 workspace（`Map<workspace, Instance>` 应用层隔离），主场景是 IDE + Web SDK + 个人开发者多项目。Qwen Code 主场景含 IM channels、多 tenant SaaS、K8s 部署——对**强隔离 / quota / blast radius** 要求更高，1 daemon = 1 workspace 更契合。

| 维度 | OpenCode（多 workspace 应用层隔离）| Qwen（1 daemon = 1 workspace）|
|---|---|---|
| 启动时间 | ~2-3s | ~2-3s |
| 新 workspace 首 session | <100ms（同 daemon 内）| ~2-3s（启新 daemon）|
| 同 workspace 第 N session | <50ms | <200ms |
| 100 同 workspace session 内存 | ~100-150MB | 类似 |
| 100 跨 workspace session 内存 | ~200MB（共 daemon）| ~10-15GB（100 daemon × baseline）|
| 隔离强度 | 应用层 | **OS 进程级（最强）**|
| Blast radius | 整个 daemon | **1 workspace** |
| Quota 颗粒度 | 需应用层抽象 | **cgroup / systemd 直接套用** |

> 以上时延/内存为估算量级（`~`）。跨 workspace 高密度场景 OpenCode 更省内存，Qwen 用内存换 OS 级真隔离 + 原生 quota + K8s 契合——在其主场景下值得。业界对照详 [06 — Roadmap](./06-roadmap.md)。

### 工程约束

| 约束 | 保障方式 |
|---|---|
| daemon 主线程**永不**调用 `process.chdir()` | CI grep audit |
| 顶层 `uncaughtException` handler：log + graceful exit | top-level handler |
| 启动时绑定的 workspace 不可变 | daemon 启动后 cwd 固化 |
| `qwen --acp` child 退出 = daemon 进入终态 | 全部 session 收 `session_died` 后 daemon 退出 |

---

## 决策 3 — MCP server 生命周期

### 选择

**Workspace 级共享 transport pool**（#4336）：同 daemon 的 N session 复用同一组 MCP server 连接，不再每 session 各 spawn 一套。

```
QwenAgent（daemon 级）
├─ McpTransportPool ──── stdio / websocket MCP 连接（workspace 级共享）
├─ WorkspaceMcpBudget ── 连接数预算封顶（--mcp-client-budget）
└─ session-1..N
   └─ SessionMcpView ─── per-session 投影：过滤 + trust 修饰后注入
                          session 自己的 ToolRegistry / PromptRegistry
```

源码: `packages/core/src/tools/mcp-transport-pool.ts`、`mcp-pool-entry.ts`、`mcp-workspace-budget.ts`、`session-mcp-view.ts`；pool 由 `QwenAgent` 构造（`packages/cli/src/acp-integration/acpAgent.ts`）。

### 备选项对比

| 方案 | 资源代价 | 隔离性 | 结论 |
|---|---|---|---|
| **per-session MCP**（旧默认）| N session × 一套 MCP children，单个 MCP server ~50-200MB，重复 spawn + 重复 discovery | 最强（session 间零共享）| ❌ 同 workspace 同配置下纯浪费 |
| **workspace 级 pool**（现状）| 同 workspace 同配置的 MCP server 只起一份 | transport 共享，**工具可见性 / trust 仍 per-session**（经 `SessionMcpView` 投影）| ✅ 当前实现 |
| daemon 间全局 pool | 更省 | ❌ 跨 workspace 共享 MCP 进程，破坏决策 2 的 OS 隔离边界 | ❌ 不做 |

### 实现要点与边界

| 要点 | 说明 |
|---|---|
| 池化范围 | 默认只池化 `stdio` + `websocket` transport（真 OS 子进程，状态可观测可隔离）；可经 `QWEN_SERVE_MCP_POOL_TRANSPORTS` 调整 |
| SDK MCP server 不入池 | 始终 per-session 直连（需经 ACP control plane 的 per-session 路由）|
| 预算 | `--mcp-client-budget=N` 由 workspace 级 `WorkspaceMcpBudget` 统一记账，取代 per-session 各自记账；超限抛同一 `BudgetExhaustedError` |
| 回收 / 恢复 | pool entry 带 drain 延迟与 idle 回收，连接恢复由 entry 的 reconnect 策略接管（per-session 模式则沿用 30s 健康检查 + 自动重连）|
| 逃生门 | `--no-mcp-pool`（即 `QWEN_SERVE_NO_MCP_POOL=1`）回退 per-session 模式，便于对照与回滚 |
| 跨 daemon | 不共享——跨 workspace 的 MCP 进程天然 OS 隔离（决策 2）|

### 依据

1. **MCP transport 是无 session 概念的重资源**：MCP 协议没有 qwen session 的概念，同配置连接可安全复用；而 50-200MB × N session 的重复 spawn 在 IDE / channels 多 session 场景下不可接受。
2. **共享 transport ≠ 共享可见性**：trust 决策、工具开关是会话语义，必须留在 per-session 层——`SessionMcpView` 把池里的工具快照按 session 配置过滤 + trust 修饰后才注入 session 自己的 registry。
3. **池化范围保守**：只池化状态可观测的 transport，SDK MCP 这类与 session 控制面耦合的路径保持直连，避免把"池"变成隐式全局状态。

---

## 决策 4 — FileReadCache 共享语义

### 选择

**Per-session 严格私有**。每个 session 的 `Config` 持有自己的 `FileReadCache` 实例（源码: `packages/core/src/config/config.ts`、`packages/core/src/services/fileReadCache.ts`），不向其他 session 泄漏。

### 依据

1. **prior-read enforcement 依赖 per-session 语义**（#3774）：cache miss 的含义是"**当前 session** 没读过该文件"→ 拒绝 Edit/Write（错误码 `EDIT_REQUIRES_PRIOR_READ` / `FILE_CHANGED_SINCE_READ`，源码: `packages/core/src/tools/tool-error.ts`）。共享 cache 会让"别的 session 读过"也算"读过"，安全语义直接失效。
2. **invalidation 路径与会话历史操作耦合**：`setHistory` / `truncateHistory` / `resetChat` 等历史操作触发 cache 清理（源码: `packages/core/src/core/client.ts`）——这些操作都是 per-session 的，共享 cache 会把单个 session 的历史操作放大成全 daemon 的 invalidation。
3. **跨 session 重复读的代价小**：FileReadCache 节省的是 LLM token（避免重复把文件内容塞进上下文），不是磁盘 I/O——磁盘侧 OS page cache 已经兜底，共享收益边际。

### 曾评估的拆分方案

理论上可拆两层："content cache（`path+mtime → bytes`）跨 session 共享" + "per-session read set（保留 prior-read enforcement）"。不做的原因：OS page cache 已兜底磁盘收益，而拆分需要重构统一的 cache 与全部 invalidation 路径，复杂度收益比不成立。仅在单机 500+ session 实测出瓶颈时再评估。

### daemon 内资源共享汇总

| 资源 | 共享范围 | 隔离机制 |
|---|---|---|
| Provider registry | daemon 全局 | 不可变 |
| Skill registry | daemon 全局 + path-conditional | 不可变 + per-tool-call 激活 |
| Auth credentials | per-daemon | 跨 daemon OS 进程级隔离 |
| **MCP transport** | **workspace 级 pool（决策 3）** | per-session `SessionMcpView` 过滤 + trust；预算封顶 |
| MCP 可见性 / trust | per-session | session 自己的 ToolRegistry |
| LSP server | per-session / 实现相关 | 随 session config 初始化 |
| Session state | per-session | SessionService + transcript JSONL |
| **FileReadCache** | **per-session 严格私有（本决策）** | per-session Config 持有 |
| Permission flow | per-tool-call | workspace/global scope 决策文件 per-daemon 共享（详 [05](./05-permission-auth.md)）|
| Background shell / task | per-task | task ID + sessionId 关联 |

---

## 决策 5 — Permission flow

### 选择

三段式：**复用 in-process 决策核心 + `ask` 外化为 SSE 事件 + PermissionMediator 多 client 仲裁**。

1. **决策核心不分叉**：daemon 与 CLI / ACP 模式走同一 `evaluatePermissionFlow()` L1→L4 链（#3723，源码: `packages/core/src/core/permissionFlow.ts`）——权限规则只有一份实现。
2. **`ask` 不阻塞 HTTP**：决策结果为 `ask` 时，daemon 推 `permission_request` SSE 事件给该 session 的所有订阅 client，原工具调用挂起等待 `POST /session/:id/permission/:requestId` 应答。
3. **多 client 仲裁由 PermissionMediator 决定**（#4335，源码: `packages/acp-bridge/src/permissionMediator.ts`）：

| Policy | 语义 | 适用 |
|---|---|---|
| `first-responder`（默认）| 任一 client 先答先得，其余收"已被应答" | 单用户多端 / 协作 |
| `designated` | 仅指定 client 的应答有效，他人投票 403 | 有 bot 旁观的场景，防 bot 抢答 |
| `consensus` | 需达到 quorum 票数 | 多人共管的敏感 workspace |
| `local-only` | 仅本机 client 可应答，远端投票 403 | 远端只读观察 |

策略为 daemon 级配置（bridge 启动参数，缺省 `first-responder`）；daemon 对未实现的策略返回 501 结构化错误，client 可提示"daemon 版本过旧"。投票路由按 sessionId 隔离，A session 的请求不会泄漏给 B session 的订阅者。

完整三层权限模型、审计与认证详 [05 — Security & Permission](./05-permission-auth.md)。

---

## 决策 6 — 多 client 并发

### 选择

**同 session prompt FIFO 串行 + 事件 fan-out + 跨 session 并行。**

- 每 session 一条 `promptQueue` FIFO（源码: `packages/acp-bridge/src/bridge.ts`），并满足 **no-poison**：单个 prompt 失败或取消不会卡死队列，后续 prompt 正常消费。
- 模型切换、approval-mode 切换同样走 per-session FIFO，防止并发改配置产生竞态。
- 事件 fan-out：每 session 的 EventBus 维护订阅者集合，`SessionNotification` 广播给全部订阅 client（源码: `packages/acp-bridge/src/eventBus.ts`；`packages/cli/src/serve/eventBus.ts` 为兼容 re-export）。

```
Client A → POST /session/:id/prompt
Client B / C → GET /session/:id/events（SSE 已订阅）

daemon 处理 prompt → SessionNotification 流
  ├─ A（发起方）
  ├─ B ← fan-out
  └─ C ← fan-out
```

### 操作矩阵

| 操作 | 谁能做 | 冲突处理 |
|---|---|---|
| 发 prompt | 任何 client | 同 session FIFO，后到的排队 |
| 应答 permission_request | 按决策 5 的 policy | 默认 first-responder，其余收"已被应答" |
| 取消 | 任何 client | `POST /session/:id/cancel` |
| 设置 model / mode | 任何 client | per-session FIFO 生效，所有 client 收到通知 |

### 依据

ACP 协议的不变式是 **one active prompt per session**（client → agent → response 的同步语义），daemon 跟随该约束，在其上叠加事件 fan-out 实现"多 client 协作观察"。同 session 并发 prompt 收益几乎为零、复杂度极高（LLM 调用并行化、工具调用竞态、FileReadCache 一致性），不做。需要并行就开新 session（决策 1 的 `thread` scope）——并行的正确颗粒度是 session，不是 prompt。

---

## 决策 7 — 部署模式：Mode B 主线，Mode A 暂停

### 选择

| 模式 | 启动命令 | TUI | 状态 |
|---|---|---|---|
| **Mode B：headless daemon** | `qwen serve [--port N]` | ❌ | **主线**：服务器 / 容器 / K8s / 所有跨进程 client 的统一 runtime |
| Mode A：TUI 进程内嵌 HTTP server | `qwen --serve [--port N]` | ✅ 本地 | 暂停（#4156 parking lot）|

### 依据

1. **跨进程 client（channels / web / IDE / SDK）只能走 Mode B**——它们本来就在 daemon 进程之外，headless HTTP server 是唯一可行边界。先把这条主路径的 event / control / client contract 做稳。
2. **Mode A 引入第二条 in-process 暴露路径**：TUI 与 server 同进程会带来"TUI 本地状态变更如何进入 typed event 流、TUI 退出是否带走 daemon"等一组未解问题（清单见 [04 — Deployment & Client](./04-deployment-and-client.md)）。在 Mode B contract 稳定前不开第二战线。
3. **本地单用户场景根本不需要 daemon**：见文末设计原则——本地 TUI 永远 in-process 直连，不为 daemon 付费。Mode A 的潜在价值仅在"本地多 client 协作"（IDE + TUI 同 session）这一显式 opt-in 场景，届时再评估。

### TUI 接入语义

- **本地单用户 `qwen` TUI = in-process 直连，永不走网络**——这是 daemon 项目的最高优先级 UX 约束，任何默认行为变更不得破坏。
- **TUI daemon-client adapter**（#4266，scope 收敛中）= opt-in 进阶能力，仅用于用户主动选择的多 client 协作；**永远 behind flag，不进入默认迁移**。
- channels / web / IDE / 远端 TUI 经 `DaemonSessionClient` 接入——daemon 是它们唯一可行路径。

---

## 决策 8 — server / client / runtime 边界

### 选择

daemon 架构按 4 层收敛，依赖方向单向向下：

```text
client adapters / output sinks（TUI / channel / web / IDE / JSONL / stream-json）
  ↓ 只依赖
daemon client / protocol 层（DaemonClient / DaemonSessionClient /
  typed event schema / reducer / reconnect / heartbeat / capability 协商）
  ↓ HTTP/SSE/WS 边界
daemon server / control plane（qwen serve：routes / auth / EventBus
  projection / ACP bridge / workspace state CRUD / permission 仲裁）
  ↓ runtime 边界
runtime worker / sandbox runner（现状 qwen --acp child；未来可替换为
  隔离 sandbox runner——tools / shell / MCP / skills / LSP / 文件操作）
```

**Client 只对接 HTTP/SSE/WS + client SDK，不 import / 订阅 daemon 内存对象。** EventBus 是 server 侧 fan-out 原语，对外的契约是 typed SSE events + reducer。

### 包边界（已物理落地）

server 核心原语已拆为独立包 `packages/acp-bridge`（HttpAcpBridge / EventBus / PermissionMediator 等）；`qwen serve` 入口与 HTTP routes 在 `packages/cli/src/serve`；client / protocol 层即官方 SDK（`packages/sdk-typescript` / `sdk-python` / `sdk-java`）；`webui` / `web-shell` / `vscode-ide-companion` / `zed-extension` / `desktop` 为 client adapters。

**硬约束**：

| 约束 | 含义 |
|---|---|
| Server 不依赖 adapters | daemon server 不能 import 任何 renderer |
| Adapters 只依赖 client / protocol 层 | 不可直接拿 `HttpAcpBridge` / EventBus 内部对象 |
| typed events + reducer 属 client / protocol 层 | 不散落在 server 或某个 adapter 私有实现里 |
| Output sinks 也是 adapter | JSONL / stream-json 消费同一 typed event 流，不各自驱动 runtime |

### Daemon-native renderer 是目标形态

所有渲染面消费同一 typed event contract：

```text
daemon SSE/WS event → shared reducer / view model → renderer 投影
  （Ink TUI / DOM chat / terminal 视图 / IDE panel / channel cards / JSONL sink）
```

**PTY proxy 不是主线**：代理 terminal 字节流会绕开 typed event / reducer 收敛，并重新耦合 runtime 进程生命周期，只保留为兼容 / demo / debug fallback。

### Runtime worker / sandbox runner 边界

当前 runtime 是 `qwen --acp` child；企业 / 云形态可替换为隔离 sandbox runner，约束不变：

| 项 | 要求 |
|---|---|
| Runtime locality | MCP / skills / shell / LSP / provider auth / 文件操作跟随 runtime worker 所在主机，不跟随 visual client |
| Failure isolation | runner 崩溃走 typed diagnostic event + restart/recreate，不等价于 control plane 必然挂 |
| Client capability 反向 RPC | 只覆盖 editor / clipboard / browser / notification 等显式 client-local 能力 |
| No hidden fallback | 不允许把 client 本地的 MCP / skills / shell 当隐式 fallback 执行面 |

### Remote-control = control overlay

远程控制只是 daemon 的 facade / ingress，不拥有独立 runtime 或 event protocol。两种合法形态——**local daemon + relay**（远端 UI → relay → 本机 `qwen serve`）与 **remote daemon + gateway**（远端 UI → gateway → 远端 `qwen serve`）——都复用同一 client / protocol 层，差异只在 routing / auth / pairing，不在 runtime protocol。

---

## 决策 9 — northbound transport：dual additive

### 选择

daemon 同时暴露**自有 REST+SSE** 与**标准 ACP**两套北向协议，共享同一 `HttpAcpBridge` + EventBus；ACP 侧有 HTTP（#4472）与 WebSocket（#4773）两种 transport：

```text
qwen 自有 REST + SSE                标准 ACP
POST /session/:id/prompt            POST/GET/DELETE /acp（Streamable HTTP）
GET  /session/:id/events（SSE）     WS /acp（WebSocket，与 SSE 共存）
POST /workspace/*
        ↓                                ↓
        └───── 共享 HttpAcpBridge + EventBus ─────┘
                        ↓
                 qwen --acp child
```

| 维度 | qwen 自有 REST + SSE | 标准 ACP（HTTP / WebSocket）|
|---|---|---|
| 端点 | `/session/:id/*`、`/workspace/*` 多端点 | `/acp` 单端点 |
| 协议 | qwen 自有 REST 方言 | 标准 ACP JSON-RPC（Streamable HTTP / WS）|
| 适合 client | 官方 SDK / web-shell / webui / channels / curl | Zed / Goose 等 ACP-native client |
| 开关 | 始终开启（daemon 主功能）| `QWEN_SERVE_ACP_HTTP=0` 可关 |
| 同一 session 跨协议访问 | ✅ 共享 bridge + EventBus，事件流一致 | ✅ 同左 |

> 此外 daemon 还有一个 MCP stdio facade（`qwen-serve-bridge`，#4555）把 daemon 包装成 MCP server 供 Qoder / Claude Desktop 等接入——它是 client / protocol 层之上的 adapter，不是第三套 server 协议。详 [03 — HTTP API & Protocol](./03-http-api.md)。

### 依据：为什么 dual 而不是单选

**战略层——ACP 是 AI agent 的 LSP**：

| 类比 | LSP（编辑器生态）| ACP（agent 生态）|
|---|---|---|
| 解决的问题 | IDE × 语言的 N×M 适配 | client × agent 的 N×M 适配 |
| 实现后 | 一个 LSP server 走遍所有 IDE | 一个 ACP client 可驱动任何 ACP agent |

AI agent 工具正处在 LSP 出现前的那个 inflection point。只有自有 REST 时，每个第三方 client 都要实现 qwen 私有协议，接入门槛极高；走 ACP 标准后，已实现 ACP 的 client（Zed / Goose / 未来 SDK）零成本接入，qwen 享受协议生态的网络效应。

**竞争层——"开放 vs 闭门"差异化**：

| Agent | 协议开放性 |
|---|---|
| Claude Code | 闭门：自有协议 + 二进制混淆 |
| Cursor | 闭门：私有协议 |
| **qwen-code** | **ACP + MCP 双开放标准** |
| Goose | ACP 开放标准 |

模型质量之外，"开放协议生态"是 qwen 可选的差异化轴——做闭源独占 client 的厂商没有动力做这件事。

**架构层——为什么 additive 而不是替换**：现有 webui、3 个官方 SDK、VSCode companion、channels 全部依赖 REST surface，替换 = 全部重写；且 ACP 的 HTTP transport 规范仍在演进（RFD #721），押注替换为时过早。Dual additive 让现有客户零迁移、ACP 早期 adopter 有入口、`QWEN_SERVE_ACP_HTTP=0` 提供回退，且**同一 session 可从任一协议访问**——mixed-client 协作天然成立。

### 实现要点

| 要点 | 说明 |
|---|---|
| `/acp` 单端点 | POST `{initialize}` 返回 capabilities + `Acp-Connection-Id` header，后续请求带该 header；GET 为 SSE 流，DELETE 关连接 |
| WebSocket transport | `WS /acp` 与 SSE 共存（不替换），双向帧替代 SSE+POST 组合，适合高频交互 client |
| Vendor 扩展 | 在 `initialize` 响应 `_meta` 下按域名 key 声明（ACP 惯例），client feature-detect 后调用 `_qwen/` 方法 |
| 功能 parity | REST 的 session / workspace / file / auth / memory / agents 能力以 `_qwen/` 方法镜像暴露（#4827 起步，现 41 个方法）；model / mode 走标准 `session/set_config_option` |
| 共享 EventBus | `/acp` 流与 `/session/:id/events` SSE 事件一致，跨协议访问同一 session 无差异 |
| 实现位置 | 源码: `packages/cli/src/serve/acpHttp/`（jsonRpc / sseStream / wsStream / connectionRegistry / dispatch）|

### Trade-offs

| 利 | 弊（与缓解）|
|---|---|
| ACP 生态网络效应（Zed / Goose / 未来 SDK）| 多一套 transport 代码——缓解：共享同一 bridge，transport 层很薄 |
| 开放标准是差异化轴 | ACP 规范演进需跟进——缓解：env 开关可禁用，REST 主功能不受影响 |
| 现有客户零迁移 | 两套 wire 需保 parity——缓解：vendor 方法批量镜像 + 共享 bridge 保证行为一致 |
| 同 session 跨协议 mixed-client 协作 | client 选择困惑——缓解：文档明确（官方 SDK 走 REST，ACP-native client 走 `/acp`）|

### 与决策 8 的关系

本决策只在 **server 层**增加北向协议，4 层边界不变：client SDK（`DaemonClient` / `DaemonSessionClient`）仍走 REST；ACP-native client 是与官方 SDK 并列的另一类 adapter。

---

## 设计原则 — 不要为你不需要的东西付费

贯穿多个决策的同一条原则（借鉴 C++ 的 **zero-cost abstraction**）：**不主动开启的功能，不应施加运行时代价。**

最重要的应用：**本地单用户 TUI 是最高优先级 UX，永远 in-process 直连，不走 daemon**——单用户不需要 daemon，就不应被强加 daemon 的复杂度：

| 维度 | 本地单用户 TUI（永远 in-process）| 跨进程 client（IDE / channel / web / 远端 TUI）走 daemon |
|---|---|---|
| 启动延迟 | 直起 Ink，零网络 | ~50-200ms loopback handshake 可接受 |
| 进程数 | 1 个 binary | 2+（daemon + client）必然 |
| port / token / discovery / lifecycle | **不需要** | 必要成本 |
| 多 client 协作 | 不需要（单用户）| 主动选择此场景才走 daemon |

**推论**：

- ❌ "本地 TUI 经 loopback 连 `qwen serve`"不能成为默认迁移目标；auto-daemon UX（`qwen` 启动时自动发现/拉起 daemon）仅适用于用户主动选择多 client 协作
- ✅ TUI daemon adapter 永远 behind flag；daemon 默认接入只面向本来就跨进程的 client（channel / web / IDE）
- 同理：MCP pool 提供 `--no-mcp-pool` 逃生门（决策 3）、ACP transport 提供 `QWEN_SERVE_ACP_HTTP=0` 开关（决策 9）、`thread` scope 按需显式开启（决策 1）——能力默认服务主场景，代价只由选择它的人承担

---

> **免责声明**：本文基于 qwen-code main 分支源码与公开 PR/issue 写成，不代表 Qwen Code 团队官方设计文档。daemon 处于 preview 阶段，实现细节（尤其 vendor 方法面与环境变量开关）可能继续变化。截至 2026-06-12。

下一篇：[03 — HTTP API & Protocol →](./03-http-api.md)
