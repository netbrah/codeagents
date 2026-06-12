# 06 — Roadmap & Ecosystem

> [← 上一篇：Security & Permission](./05-permission-auth.md) · [回到 README](./README.md)

## TL;DR

`qwen serve` daemon 从第一个原型（2026-05-13，#3889）到功能集合入 main（2026-06-11，#4490）用了约一个月，当前随 **v0.18.0-preview** 线发布。日常开发在 `daemon_mode_b_main` 长期 integration 分支进行，周期性反向 merge 回 main（#4490 是首次）。

本篇回答四个问题：**已经落地了什么 → 正在做什么 → 按什么原则演进 → 远期还有哪些方向**，并附与 OpenCode / Anthropic Managed Agents 的定位对照。文中 `#NNNN` 均指 QwenLM/qwen-code 仓库的 PR / issue 编号。

**一句话定位**：qwen-code 主线只交付 **daemon building block**（Unix 风格、可组合、协议表面克制）；多租户 / 跨机编排 / SaaS 平台层留给外部集成方实现（见 §五）。这与 OpenCode（端到端平台）和 Anthropic Managed Agents（托管服务）构成谱系上的三个点。

---

## 一、已落地

### 1.1 里程碑

| 时间（2026）| 里程碑 | 出处 |
|---|---|---|
| 05-13 | `qwen serve` headless daemon 雏形：session 创建 / prompt / SSE 事件流 / cancel / 模型切换 / permission 投票主链路 | #3889 |
| 05-15 | 边界收紧为 **1 daemon = 1 workspace × N session**；Mode B（headless）定为主线，Mode A（`qwen --serve` 带 TUI）进 parking lot | #4113 / #4156 |
| 05-16 ~ 05-18 | 生产化基线：capability registry / typed event 协议契约、会话生命周期（load / resume / close）、可靠性（heartbeat / 背压 / 重放）、只读控制面 + 诊断、写路由门禁、文件读写安全边界、OAuth device-flow | #4175（rollout tracker，能力明细见 §1.2）|
| 05-19 ~ 05-28 | 架构抽取与生态：acp-bridge 独立包、MCP 共享连接池、PermissionMediator、SDK UI 层、web-shell、ACP HTTP transport、非阻塞 prompt、MCP 桥 qwen-serve-bridge | #4319 / #4336 / #4335 / #4328 / #4353 / #4380 / #4472 / #4585 / #4555 |
| 06-08 ~ 06-10 | rate limiting、workspace 配置热重载、web-shell 打磨 | #4861 / #4965 |
| 06-11 | **daemon 功能集合入 main**，随 v0.18.0-preview 线发布；同日：官方桌面 app、ACP/REST 29-method parity、ACP WebSocket transport | #4490 / #3778 / #4827 / #4773 |

一个月的节奏可以读成四步：**先钉边界**（1 daemon = 1 workspace）→ **再补多 client 生产 must-have**（身份 / 生命周期 / 可靠性 / 写门禁）→ **然后抽架构**（acp-bridge 独立包 / MCP pool / PermissionMediator）→ **最后开客户端生态**（SDK / web-shell / 桌面 app / ACP parity）。期间日常开发在 `daemon_mode_b_main` integration 分支累积，main 始终保持可发布。

### 1.2 已交付能力（按主题）

**协议面**

- capability registry + protocol versions：新能力一律以 feature tag 增量暴露，client 按能力协商 / 降级（#4191）
- typed event schema + SDK reducer：`data: unknown` 收敛为 discriminated union，client 不再各自拼状态（#4217）
- 非阻塞 prompt：`POST /session/:id/prompt` 立即返回 `202 {promptId}`，结果经 SSE 按 promptId 关联（#4585）
- northbound 三协议并存：自有 REST+SSE、ACP HTTP（#4472）、ACP WebSocket（#4773，与 REST 29-method parity #4827）；外加把 daemon 包装成 MCP server 的 `qwen-serve-bridge`（#4555）

**会话生命周期**

- load / resume HTTP 化（暂带 `unstable_` 前缀）+ session metadata + close/delete + per-request `sessionScope`（#4222 / #4240 / #4209）
- 服务端 shell：`!` 命令直接执行不经 LLM，输出流式回放并注入会话历史（#4576）
- workspace 配置热重载：统一 settings 重载，只刷真正变化的 key（#4965）

**安全**

- 基线防护：bearer token timing-safe 比对、`0.0.0.0` 无 token 拒绝启动、Host allowlist（#3889）
- 中心化 mutation gate + `--require-auth`：所有写路由走同一门禁，不允许 per-route 散装鉴权（#4236）
- FileSystemService 文件边界：workspace 根限定、symlink 逃逸拒绝、原子写（temp+rename）、content-hash 前置条件（#4250 / #4280）
- OAuth device-flow 由 daemon 代办，凭证不过 client；`--allow-origin` CORS 白名单；`--rate-limit` 令牌桶（#4255 / #4527 / #4861）

**多 client 协调**

- daemon-stamped clientId（client 不可自报身份）+ session-scoped permission 投票 + PermissionMediator 4 策略（#4231 / #4232 / #4335）
- client heartbeat、SSE replay 扩容 + `slow_client_warning` 背压信号、ring 淘汰时 `state_resync_required` 缺口检测（#4235 / #4237 / #4360）
- MCP workspace 级共享连接池：N session 复用同一 MCP transport，预算封顶（#4336）

**客户端生态**

- SDK：`DaemonClient` / `DaemonSessionClient` + 共享 UI transcript 层，TUI / channel / web / IDE 适配共用（#4201 / #4328 / #4353）
- web-shell 浏览器终端（#4380）、官方桌面 app（#3778）、IDE / channel adapter behind flag（#4199 / #4203）

**可观测性**

- 只读 status routes：MCP / skills / providers / context（#4241）
- preflight + env 诊断 + `errorKind` 封闭错误分类——返回 actionable 失败原因而非布尔状态，远端 client 不会 "silently lose tools"（#4251）
- OTel tracing（prompt lifecycle / tool spans）+ daemon 文件日志（#4321 / #4559）

路由与事件完整参考见 [03 — HTTP API & Protocol](./03-http-api.md)；权限与认证细节见 [05 — Security & Permission](./05-permission-auth.md)。

---

## 二、进行中（截至 2026-06-12）

实施跟踪：#4175（rollout tracker）；#4490 合入 main 后的剩余工作在 #4511 / #4514 按 tier 分级。未关 backlog：

| 条目 | 内容 | 出处 |
|---|---|---|
| 客户端身份 Tier-2 | pair token + per-client revocation——现状是共享 bearer token + daemon-stamped clientId，撤销粒度只有整个 token | #4514 |
| 会话路由转正 | `loadSession` / `resume` 去掉 `unstable_` 前缀，契约在多 client 重连场景验证后冻结 | #4514 |
| Tier-3 剩余 | branch / rewind / restore 的 HTTP 化、`--max-body-size`、`/extensions` 只读面、multi-daemon 协调 | #4514 |
| side-channel 协调 | A2 / A5 两项跨 client side-channel 设计未落地 | #4511 |
| TUI daemon-client adapter | 本地 TUI 接 daemon 的适配层，scope 仍在收敛；**本地 TUI 默认 in-process 直连不变**（见 §三）| — |
| 容器化部署参考件 | systemd / launchd / nohup 模板已有；Docker / K8s 参考件待补 | — |

以上均为 additive 工作，不改变已合入 main 的协议表面（§三 原则）；client 通过 `/capabilities` feature tag 发现新能力落地与否。

**相邻方向（不依赖 daemon）**：dynamic workflows——LLM 写编排脚本 fan-out 子任务——走 in-process tool 路线（#4721 / #4732），与 daemon 是平行能力，详见 [Claude Code Dynamic Workflows Deep-Dive](../claude-code-dynamic-workflows-deep-dive.md)。

---

## 三、设计与合入原则

一个月内 30+ 个 PR 渐进合入而 main 始终可发布，依靠的是下面这组原则。它们不只是史实，也继续约束后续所有 daemon PR：

| 原则 | 含义 |
|---|---|
| 每 PR 可单独合入 | 自带完整测试；合入后 main 仍可发布；每个能力可独立回滚 |
| 向后兼容 additive-only | 不移除已有 route / event 字段 / CLI 行为；新增字段 optional；旧 client + 新 daemon 与新 client + 旧 daemon 都能工作 |
| capability tag 协商 | 新能力一律经 `/capabilities` feature tag 暴露；client 检测不到 tag 时降级或隐藏对应 UI，不靠版本号猜测 |
| client adapter 先 behind flag | TUI / channel / IDE 适配先 flag 门禁试点，默认路径不切换；验证完成才扩大默认面 |
| 不要为不需要的东西付费 | 本地单用户 TUI **永远 in-process 直连、永远不走网络**；daemon 只服务真正需要多 client / 远程接入的场景（zero-cost abstraction）|
| daemon 是 runtime owner | client 只做 view + command surface，不复制 runtime 逻辑，避免每个 client 长出一条平行 runtime |
| passthrough ≠ gap | #4516（`/compress`、`/_meta`）与 #4515（`/stats`、`/export`）撤回：这些能力已可经 `POST /prompt` 的 slash-passthrough 达成。**已有 passthrough 的 surface 是 polish 不是 gap**——新增专用 route 必须有真实 client demand 验证，防止 API 表面无限膨胀 |

---

## 四、远期候选

> 以下无 issue 跟踪，属候选而非承诺。原协议补齐清单中的 WebSocket 双向 transport 已随 #4773 落地，移出本节。

- **OpenAPI codegen**：为 REST 面生成 OpenAPI schema，供任意语言 client codegen（现状：SDK 手写 TypeScript types）
- **Prometheus metrics**：在已有 OTel tracing 之上补 metrics 面 + `/health?deep=1` 深度健康检查
- **mDNS 服务发现**：局域网内 daemon 自动发现，配合多 daemon 共存的 instance token 路径
- **`POST /ext/:method` 扩展点 + client capability reverse RPC**：client 向 daemon 注册反向能力（editor / clipboard / browser / notification / file_picker），严格 scoped——不允许 silently 成为 MCP / skill 的执行 fallback
- **native in-process**：去掉 `qwen --acp` child 桥接，daemon 直接 import `QwenAgent`，省去 child 内存与 IPC 延迟；仅在高密度部署（数百 session / 机）证明必要时推进
- **Mode A revisit**（#4156 parking lot）：`qwen --serve` = TUI super-client + 内嵌 HTTP server，保零网络 TUI 体验的同时对其他 client 开放；待 Mode B 契约完全稳定后再评估

---

## 五、External Reference Architecture（参考架构，非路线图）

> 本节是给外部集成方（商业平台 / k8s operator / 云厂商）的设计参考蓝图，**不在 qwen-code 项目路线图内**。主线交付并保持稳定的协议表面（§一）是这些平台层的地基——平台组件只依赖公开 HTTP/SSE 契约，不需要 fork daemon。

### 5.1 跨 daemon orchestrator

1 daemon = 1 workspace 意味着多 workspace / 多机部署需要一层 coordinator：

- sessionId → daemonUrl 注册表 + 跨 daemon 聚合 API（list / health）
- daemon 进程池：spawn-per-workspace、端口分配、idle 回收
- sticky routing + failover：daemon crash 后经 transcript resume 在新进程重建会话

单机单 workspace 完全不需要本节；这是容量或隔离需求触发的增量层。

### 5.2 多租户 + OIDC + Quota + Audit

**关键约束**：同 daemon 内 N session 共享同一 `qwen --acp` child 的 OS 权限——**多 tenant 不可共享一个 daemon**。租户隔离必须绑定在 daemon process 层：1 tenant = 1 workspace（推荐）或 1 tenant = 1 daemon process（高安全）。其上的平台组件：

- tenant 抽象 + workspace ACL；AuthN 网关（Bearer / OIDC / mTLS / cookie）
- quota engine（sliding window + reservation）；audit log 多通道（jsonl / syslog / OpenTelemetry / Kafka）

多租户边界的安全分析见 [05 — Security & Permission](./05-permission-auth.md)。

### 5.3 Shell sandbox

主线默认 **NoSandbox**：agent 以 daemon 进程的 OS 权限执行 shell / tools。参考分层方向：

- daemon 收敛为 control plane（session / event / permission 协调）
- tool / shell / MCP / skills 执行下沉到独立 runtime worker：本地 sandbox（OS user 切换 / Linux namespace / 容器）或远程 runner（SSH / k8s Job）
- sandbox 崩溃 ≠ daemon 崩溃：daemon 发 typed diagnostic event 并重建 runner
- MCP stdio child 在 runtime 侧运行，网络 egress 从 runtime host 出，而不是从 client 机器出

### 5.4 SaaS 部署

1 daemon = 1 workspace 的 OS 进程模型与云原生原语直接对应：1 pod = 1 daemon = 1 workspace；cgroup / systemd `MemoryMax=` / docker `--memory` 即 per-workspace quota。SaaS 化还需自建：

- k8s native 编排（StatefulSet + PVC + service mesh）
- 持久层外置：Postgres 状态 + Redis 缓存 + S3 transcript 归档
- 多 region 调度与就近路由

---

## 六、业界对照

### 6.1 vs OpenCode（最相似的开源竞品）

| 维度 | OpenCode | qwen-code daemon |
|---|---|---|
| 进程模型 | 单 daemon 多 workspace（in-process Map + ALS 应用层隔离）| **1 daemon = 1 workspace × N session**；多 workspace = 多进程（OS 级隔离）|
| HTTP 框架 | Hono | Express 5 |
| 协议 schema | OpenAPI codegen | 复用 ACP schema + 自有 REST/SSE 双面并存 |
| session 跨 client 共享 | 否（per-SDK call）| 默认可共享（`sessionScope: 'single'`）+ 多 client 权限投票 |
| 默认安全 | 无 token 警告后仍启动 | 无 token + `0.0.0.0` **拒绝启动** |
| 持久化 | SQLite | transcript JSONL + 权限决策 SQLite |

取舍：跨 workspace 高密度场景 OpenCode 内存更省（单进程复用）；qwen 用多进程换 **OS 级隔离 + cgroup quota 直接套用 + k8s 天然契合 + blast radius 限于单 workspace**。qwen 独有面：IM channels 多渠道路由（钉钉 / Telegram / 微信 / Slack）与 ACP + MCP 双开放标准 northbound。设计哲学也相反：OpenCode 走端到端平台路线，qwen-code 保持 Unix 风格可组合的 building block，平台层留给外部（§五）。

### 6.2 vs Anthropic Managed Agents

| 维度 | Anthropic Managed Agents | qwen-code daemon |
|---|---|---|
| 本质 | 云托管 SaaS agent runtime（闭源）| 自托管 daemon building block（Apache-2.0）|
| 模型 | Claude only | 任意 provider（DashScope / Claude / OpenAI / 自部署）|
| 多租户 / quota / 编排 | 平台托管 | 外部集成方按 §五 自建 |
| 接入面 | API | REST+SSE / ACP / MCP 桥 / IM channels / Java SDK |

可比对象不是 daemon 本体，而是"daemon + §五 参考架构包装出来的托管产品"。关键差异：

- **自托管 vs 托管**：daemon 自带数据主权与部署自由，代价是平台层（§五）需要自己实现和运营
- **模型中立**：不绑定单一模型供应商，provider / model 可按 session 切换
- **后台任务跨 client 可见**：kind framework 的后台任务（agent / shell / monitor / dream，#3836）在 daemon 模式下对所有 attached client 可见，纯 API 形态的托管服务难有等价物

---

> **免责声明**：本篇基于源码与 PR / issue 跟踪整理，不代表 Qwen Code 团队官方路线图；§四为推断候选、§五为参考架构，均非任何团队的开发承诺。daemon 处于 preview 阶段，API（尤其 `unstable_` 前缀路由）可能继续变化。截至 2026-06-12。

下一篇：[07 — User Guide →](./07-user-guide.md) · [回 README](./README.md)
