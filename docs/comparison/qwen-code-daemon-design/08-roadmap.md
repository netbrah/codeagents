# 08 — 路线图

> [← 上一篇：权限 / 认证](./07-permission-auth.md) · [下一篇：与 OpenCode 详细对比 →](./09-comparison-with-opencode.md)

> 渐进式落地，每阶段都可独立交付价值：
> - **Stage 1**（~1 周，✅ PR#3889 ~95% 实现）—— Mode B headless `qwen serve`
> - **Stage 1.5**（~4 天增量）—— Mode A CLI + HttpServer（`qwen --serve`）
> - **Stage 2**（~1-2 周）—— orchestrator 雏形 + multi-daemon 管理
> - **Stage 3**（~1 月）—— mDNS / OpenAPI / WebSocket 双向 / 多 token / 集群部署 / 企业鉴权

## 总览

```
Stage 1 (~1 周, ✅ PR#3889 ~95% MERGED): Mode B headless qwen serve daemon
  └─ Express 5 HTTP server + ACP NDJSON over HTTP+SSE
  └─ 1 daemon instance = 1 session（与 PR#3889 child-process-per-session 一致）
  └─ 远端 client / WebUI / IDE 接入

Stage 1.5 (~4d 增量): Mode A CLI + HttpServer
  └─ qwen --serve [--port N] flag
  └─ TUI 启动后挂同一套 Express HTTP server
  └─ TUI 作为 in-process EventBus subscriber（client #0）
  └─ 用户在 CLI + WebUI/IDE/IM bot 同 session 协作

Stage 2 (~1-2 周): orchestrator 雏形 + multi-daemon 管理
  └─ qwen-coordinator
  └─ sessionScope routing: single / user / thread
  └─ daemon instance discovery / spawn / cleanup
  └─ SDK HttpTransport + Web UI cross-daemon aggregate

Stage 3 (~1 月): 对标 OpenCode 完整设计
  └─ mDNS / OpenAPI / WebSocket 双向 / 权限流深化
  └─ 多 token / 集群部署文档 / 企业鉴权
```

---

## Stage 1：实验性 `--http-bridge` flag（~1 周）

> **🚀 实现状态（2026-05-07）**：Stage 1 由 [**PR#3889**](https://github.com/QwenLM/qwen-code/pull/3889) `feat(cli,sdk): qwen serve daemon (Stage 1)` 落地（OPEN，+7698/-46，23 commits）。明确引用 issue #3803。详见本节末 [Stage 1 PR#3889 实现 audit](#stage-1-pr3889-实现-audit2026-05-07)。

### 目标

**最小改动让用户先用上 daemon 模式**——通过把现有 ACP agent 包装成 HTTP→stdio 桥接，零业务逻辑变更，验证多 client 场景的需求与痛点。

### 实现

```
[现有] qwen --acp                 → stdio NDJSON ACP agent
[新增] qwen serve --http-bridge   → 启 Express 5 HTTP server（复用 vscode-ide-companion 已有栈）
                                  → 内部启 ACP agent 子进程（pipe stdio）
                                  → HTTP body ↔ stdio NDJSON 桥接
```

**特点**：
- ACP agent 本身不改一行代码
- daemon 进程 **依然 spawn ACP agent 子进程**（不是真正的 daemon 内 in-process core）
- 单 ACP session = 单 stdio 子进程 = 多 client 排队访问

### 工作清单

| 任务 | 工作量 | 文件 |
|---|---|---|
| 新建 `packages/server/` 包 | 0.5d | `packages/server/package.json` |
| `qwen serve` CLI cmd | 0.5d | `packages/cli/src/cli/cmd/serve.ts`（仿 OpenCode）|
| Express 5 HTTP server scaffold（复用 ide-server.ts CORS+Bearer+Origin lock 模板）| 0.5d | `packages/server/src/index.ts` |
| HTTP→stdio bridge | 2d | `packages/server/src/bridge/HttpAcpBridge.ts` |
| Auth middleware | 0.5d | bearer token 校验 |
| `/session/*` 路由 | 1d | 复用 ACP request schema |
| SSE event stream | 1d | NDJSON → SSE 适配 |
| 文档 + 示例 + e2e 测试 | 1d | |
| **合计** | **~7-8 天 / 1 人** | ~700-1000 行新增代码 |

### Stage 1 局限

- **同 session 多 client = 排队**（stdio 子进程一次只能处理一个 prompt）
- **每个新 session = 新 stdio 子进程**（启动开销没省）
- **跨 session 资源不共享**（LSP / MCP 各打一套）

### 价值

- 用户立刻能用 SDK over HTTP / Web UI / VSCode 直连 daemon
- 暴露多 client 真实场景（哪些 API 用得多 / pain points）
- 为 Stage 2 设计提供数据

### Stage 1 PR#3889 实现 audit（2026-05-07）

[**PR#3889**](https://github.com/QwenLM/qwen-code/pull/3889) `feat(cli,sdk): qwen serve daemon (Stage 1)` —— OPEN，**+7698/-46 / 23 commits**。

#### 1️⃣ 体量与预估对比

| 维度 | 预估（本节工作清单）| 实际（PR#3889）| 倍数 |
|---|---|---|---|
| LOC | ~700-1000 行 | **~5100 LOC**（commits 总和；剔除测试 ~3000 LOC）| **5x-7x** |
| 工作量 | ~7-8 天 / 1 人 | 多周（含 23 commits 含多轮 audit）| 几周 vs 1 周 |
| 提交数 | — | **23 commits**：8 实现 + 4 self-audit + 5 review rounds + 4 docs + 2 misc | — |

**超出原因**（设计 → 实现的工程现实）：

| 原因 | 详情 |
|---|---|
| **EventBus + ring replay + Last-Event-ID 重连** | 原 §16 §五 计划 Stage 6 HA 才详做，PR#3889 提前到 Stage 1（client_evicted overflow + bounded subscriber queues 都做了）|
| **Timing-safe bearer compare** | §07 设计为 Bearer，PR#3889 加 SHA-256 + `crypto.timingSafeEqual` + 401 uniform across no-header/bad-scheme/wrong-token，对应 §12 §3.5 side-channel 防御（设计在 §12 但 Stage 1 实现）|
| **IPv6 loopback ergonomics** | `::1` / `[::1]` / `host.docker.internal` 等 LOOPBACK_BINDS 边界，原设计未具体化 |
| **EventBus correctness** | `client_evicted` overflow / replay ring / AsyncIterable abort handling 等几百行 |
| **Self-audit + reviewer rounds** | 23 commits 中 9 轮 audit（self-audit 1-10 + reviewer rounds 1-7）—— 这是 PR#3889 体量超出的最大来源；表明 self-review + 多模型 audit（claude-opus-4-7 / gpt-5.5）流程非常严格 |
| **DaemonClient SDK** | §04 没单独估算 SDK 端，但 sibling 同步实现 `parseSseStream` / `DaemonHttpError` |
| **child-crash recovery** | reviewer round 4 加，原设计未含 |

#### 2️⃣ 实现的 9 个 STAGE1_FEATURES（capabilities envelope）

```
['health', 'capabilities', 'session_create', 'session_list',
 'session_prompt', 'session_cancel', 'session_events',
 'session_set_model', 'permission_vote']
```

逐项映射设计章节：

| Feature | 路由 | 设计章节 |
|---|---|---|
| `health` | `GET /health` | §04 §一 |
| `capabilities` | `GET /capabilities`（9 tags）| §10 §三（Stage 1 协议兼容性）|
| `session_create` | `POST /session` | §04 §一 |
| `session_list` | `GET /workspace/:id/sessions` | §04 §一 |
| `session_prompt` | `POST /session/:id/prompt`（per-session FIFO + no-poison）| §04 §一 + 决策 §6 prompt FIFO |
| `session_cancel` | `POST /session/:id/cancel` | §04 §一 |
| `session_events` | `GET /session/:id/events` SSE + Last-Event-ID + 15s heartbeat | §04 §三 + §16 §五 SSE 重连 |
| `session_set_model` | `POST /session/:id/model`（publishes `model_switched`）| §04 §一 |
| `permission_vote` | `POST /permission/:requestId` first-responder | §04 §三 + §03 §6 决策 + §07 §3 |

#### 3️⃣ 9 commits breakdown（核心实现部分）

| Commit | 关注章节 |
|---|---|
| `61f2f59a1` scaffold `qwen serve` Express + auth + Host allowlist + /health + /capabilities | §04 §一 + §07 §1 |
| `8d7c03a5f` HttpAcpBridge spawn `qwen --acp` per workspace + ACP 10s init + sessionScope:single | §03 §1 + §05 进程模型 |
| `ca996ecb5` POST /prompt FIFO + /cancel + SessionNotFoundError | §04 §一 + 决策 §6 |
| `41aa95094` EventBus + SSE Last-Event-ID + 15s heartbeat + ring replay + client_evicted overflow | §04 §三 + §16 §五 + §18 §五 |
| `6ee655f0a` POST /permission first-responder vote + cancelSession resolves outstanding | §03 §6 决策 + §07 §3 |
| `8206a64b5` SDK DaemonClient + DaemonHttpError + parseSseStream | §10 SDK / ACP 协议兼容性 |
| `a8ce5e08d` /workspace/:id/sessions + /session/:id/model + errorMessage helper | §04 §一 |
| `ad0e6ec06` audit round 1: timing-safe bearer / coalesce spawnOrAttach / parseLastEventId / IPv6 / failOnError | §07 §1 + §12 §3.5 |
| 后续 14 commits（self-audit 2-10 + reviewer rounds 1-7）| 持续 audit |

#### 4️⃣ 设计 vs 实现对应度评估

| 章节 | 对应度 |
|---|---|
| §03 §1 sessionScope='single' default | **100%** ✓ |
| §03 §6 prompt FIFO + first responder | **100%** ✓ |
| §04 §一 路由表 | **100%**（9 路由全实现）|
| §04 §二.2 复用 ACP zod schema | **100%** ✓ |
| §04 §三 SSE / WebSocket | **80%**（SSE 完整 / WebSocket Stage 1.5 deferred）|
| §07 §1 Bearer token | **100%** + 加 timing-safe compare + 401 uniform |
| §07 §6.1 0.0.0.0 拒绝默认 | **100%** ✓ |
| §10 capabilities envelope | **100%**（9 tags 实现）|
| §16 §五 SSE Last-Event-ID 重连 | **100%**（ring + replay + 15s heartbeat）|
| §18 §五 liveness 协议 | **75%**（heartbeat 间隔 15s vs 设计 30s——更激进；client_evicted overflow 已实现）|
| §17 远端 CLI / Capability 反向 RPC | **0%**（Stage 1 不含；Stage 1.5/2 deferred）|
| §11/§12/§14/§15 多租户 / 持久化 | **0%**（Stage 4-6 才做）|

**综合**：~95% Stage 1 范畴内的设计决策 1:1 实现；少数偏差都是**设计向更严格演进**（timing-safe / 401 uniform / 15s heartbeat 比 30s 更激进 / IPv6 ergonomics），不是简化。

#### 5️⃣ 经验沉淀

| 经验 | 详情 |
|---|---|
| **EventBus 在 Stage 1 就需要完整实现** | 原计划 Stage 6 HA 详做，但 SSE Last-Event-ID 重连是 Stage 1 用户必需，无法 deferred |
| **Timing-safe / 401 uniform 等 side-channel 防御 Stage 1 就要做** | §12 §3.5 设计放在多租户章节，但 PR#3889 在 Stage 1 单租户也做了——开源 daemon 默认就该这么严 |
| **IPv6 loopback ergonomics 不能省略** | 容器化 / Docker / `host.docker.internal` 是常见用例，loopback 处理细节比预想复杂 |
| **多轮 self-audit 流程的价值** | PR#3889 用 9+ 轮 audit（含 claude-opus-4-7 / gpt-5.5 审）—— self-review 比 reviewer 抓的问题更多 |
| **child-crash recovery 是必需的** | reviewer round 4 才补；spawn 子进程模式下，子进程崩溃时 daemon 必须 graceful 处理而不是把错误传播给所有 SSE clients |
| **PR 体量 ~5x-7x 预估是常态** | 工程文档预估 vs 实际几乎总是 5x，因为 audit + 边界 + ergonomics 占大头 |

#### 6️⃣ Stage 1 不含 / 推到 Stage 1.5+ 的能力

| 能力 | 状态 |
|---|---|
| `WS /session/:id`（双向 WebSocket）| Stage 1.5 |
| `POST /file/read`、`/file/write` | Stage 2+（agent 已有 fs，daemon-only file API 给远端用）|
| `HttpTransport` 适配器（替 ProcessTransport）| Stage 2+ |
| Mobile / browser UI | 待与 PR#3929-3931 协调（参考 [§17 远端 CLI 模式](./17-remote-cli-mode.md)）|
| Pairing token / LAN URL | 待与 PR#3929-3931 协调 |
| Orchestrator（multi-daemon spawn / route / cleanup）| Stage 2 重点 |
| Mode A（CLI + HttpServer，`qwen --serve`）| Stage 1.5（~4d 增量）|

---

## Stage 2：Orchestrator 雏形 + multi-daemon 管理（~1-2 周）

### 目标

**`qwen-coordinator` 上线**——管理多个 daemon instance 的生命周期，提供 sessionScope 路由 + cross-daemon aggregate API。Stage 1/1.5 让单 daemon instance 跑起来；Stage 2 让 multi-session 部署成形。

### 工作清单

| 任务 | 工作量 | 关键文件 / 说明 |
|---|---|---|
| `qwen-coordinator` CLI cmd | 1d | `packages/coordinator/src/cli/cmd.ts`（独立 binary 或 `qwen serve --coordinator` 子命令）|
| sessionScope 路由策略实现 | 2d | `single` / `user` / `thread` 三档（[§03 §1](./03-architectural-decisions.md#1-session-是否跨-client-共享)）|
| daemon instance 注册表 | 1d | sessionId → daemonUrl + token 映射，初版用内存 Map |
| daemon spawn / cleanup | 2d | child_process spawn `qwen --acp`，`/health` 监测 + 自动 reap |
| `POST /coordinator/sessions` + `route` 路由 | 2d | [§04 §8.2](./04-http-api.md#82-新增-orchestrator-层-apistage-2) |
| SDK HttpTransport 加 `coordinatorUrl` 配置 | 1-2d | 直连 vs 经 orchestrator 二选一 |
| Web UI cross-daemon aggregate（"我所有 background tasks"）| 2d | `GET /coordinator/sessions/:id/aggregate` |
| 集成测试 + 文档 | 2d | |
| **合计** | **~1.5 周 / 1 人**（或 1 周 / 2 人）| ~1500-2500 行 |

### Stage 2 验收

- ✓ Orchestrator 路由 sessionScope `single` / `user` / `thread`
- ✓ 多 daemon instance spawn / cleanup
- ✓ SDK 通过 `coordinatorUrl` 自动找到正确 daemon
- ✓ Web UI 跨 daemon 聚合视图
- ❌ mDNS 发现（推到 Stage 3）
- ❌ OpenAPI 自动生成（推到 Stage 3）

---

## Stage 3：mDNS / OpenAPI / 企业鉴权 / 集群（~1 月）

### 目标

**生产级 daemon**——支持团队部署、多租户、零摩擦发现。

### 工作清单

| 任务 | 工作量 | 说明 |
|---|---|---|
| Workspace routing 中间件 | 5-7d | URL `/workspace/:id/*` 与 host header 双路由 |
| mDNS 服务发现 | 1d | `bonjour-service`（OpenCode 同款）—— `_qwen._tcp.local` |
| OpenAPI codegen | 3-5d | `@asteasolutions/zod-to-openapi` 从 ACP zod schema 生成 spec + SDK 验证（Hono 切换则改 `hono-openapi`）|
| WebUI 直接跑在 daemon 上 | 5-7d | 静态资源 mount，`/ui/*` 直接 serve |
| 多 token + workspace allowlist | 5-7d | `tokens.json` + per-token user-id |
| 企业认证（OIDC / SSO）| 7-10d | OAuth 2.0 / OIDC discovery |
| 审计日志 | 3-5d | 每次工具调用 / 权限决策 / 文件操作 写 audit.log |
| 配额管理（每 user / 每 workspace）| 5-7d | LLM token 用量 + tool call 频率限流 |
| 跨 client 审批 UX | 3-5d | "primary client" 概念 / 多 majority 决策 |
| 集群多实例 / 负载均衡文档 | 5-7d | sticky session + Redis state（可选）|
| 健康检查端点 / Metrics（Prometheus）| 3-5d | `/metrics` 标准 OpenMetrics |
| 文档 + 例子 + 性能基准 | 7-10d | |
| **合计** | **~6-8 周 / 2-3 人** | ~5000-8000 行 |

### Stage 3 验收

| 维度 | 要求 |
|---|---|
| 多用户 | 支持 10+ 并发 user，每 user 多 session |
| 性能 | 单 daemon 进程并发 100 sessions 无明显性能下降 |
| 可观测性 | OpenMetrics + audit log + traceId（PR#3847 OPEN 已铺路）|
| 部署 | Docker image + helm chart + 集群部署文档 |
| 兼容 | OpenAPI spec 稳定 + SDK 版本兼容矩阵 |
| 安全 | OIDC / SSO + 多 token + 速率限制 + CSP |

---

## 时间线甘特图

```
                  Week 1   Week 2   Week 3   Week 4   ...   Week 8   Week 12
Stage 1           ████
Stage 2                   ████████████████
Stage 3                                    ████████████████████████████████
                                           (同时多人并行)

里程碑:
   end Week 1: --http-bridge flag GA, 用户首批反馈
   end Week 4: qwen serve 原生 daemon GA
   end Week 12: 企业级 daemon 1.0
```

## 风险与缓解

| 风险 | 缓解 |
|---|---|
| 单 daemon instance OOM / race condition | 进程崩溃由 orchestrator 自动重启；transcript JSONL 持久化保证恢复 |
| MCP server 跨 session 状态泄漏 | per-server `requiresPerSession` flag fallback |
| FileReadCache 与 history rewrite 同步问题 | PR#3810 已修 5 路径，新加 daemon 路径需类似 audit |
| Bearer token 泄漏 | 默认 0.0.0.0 binding 拒绝启动（无 token）|
| `process.chdir()` 误调 | 落地后 grep audit + CI 守卫 |
| 与现有 ACP agent 行为不一致 | Stage 1 stdio 桥接持续保留作 reference impl |

## Stage 0：前置 PR 完成度（确认已就绪）

进入 Stage 1 前确认以下 PR 已合并：

| PR | 状态 | 必要性 |
|---|---|---|
| PR#3717 FileReadCache | ✅ 已合并 | session-scoped cache 是 daemon 必备 |
| PR#3810 FileReadCache 5 路径 invalidation | ✅ 已合并 | 长 session 正确性 |
| PR#3723 共享 permission flow | ✅ 已合并 | daemon 加第 4 mode 的基础 |
| PR#3739 Background agent resume + transcript-first fork | ✅ 已合并 | daemon 重启 / 跨 client 续行 |
| PR#3642 `/tasks` + background shell pool | ✅ 已合并 | 跨 session 任务调度 |
| PR#3818 MCP rediscovery coalesce | ✅ 已合并 | MCP pool 共享 |
| PR#3836 Kind framework 4 消费者 | ✅ 已合并 | 跨 client 任务可见性 |

✓ **全部 PR 在 2026-05-06 之前已合并**——daemon 化的所有前置基础已就绪。

---

下一篇：[09-与 OpenCode 详细对比 →](./09-comparison-with-opencode.md)
