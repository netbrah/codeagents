# Qwen Code Daemon（`qwen serve`）系列文档

> Qwen Code 的 HTTP daemon 运行时：一个长驻进程把一个 workspace 的 N 个 AI session 同时服务给多种客户端（web / 桌面 / IDE / SDK / MCP）。本系列覆盖架构、API、部署、安全与使用指南。
>
> **当前状态（2026-06-12）**：daemon 功能集**已合入 main**（[PR#4490](https://github.com/QwenLM/qwen-code/pull/4490)），随 **v0.18.0-preview** 线发布。开发在 `daemon_mode_b_main` 长期 integration 分支进行，周期性反向 merge 回 main。

## 一、daemon 是什么

```text
                  ┌─────────────────────────────────────────┐
                  │            qwen serve daemon            │
                  │    1 process = 1 workspace × N session  │
                  │  HttpAcpBridge + EventBus + 内嵌一个     │
                  │  qwen --acp child（N session 多路复用）  │
                  └────────────────────┬────────────────────┘
                                       │
    ┌────────────┬────────────┬────────┴───────┬────────────────┬───────────┐
    ▼            ▼            ▼                ▼                ▼           ▼
 REST + SSE   ACP HTTP     ACP WebSocket    MCP stdio        官方桌面 app   ACP stdio
 /session/*   /acp         WS /acp          qwen-serve-      packages/    （in-process
 /workspace/* (#4472)      (#4773)          bridge (#4555)   desktop       TUI 直连）
    │            │                             │             (#3778)
 web-shell    Zed / Goose 等                Qoder / Claude
 webui + SDK  ACP-native client            Desktop / Cursor
 channels     （41 个 _qwen/* method，#4827 达 REST parity）
```

普通 `qwen` 是"一个终端进程 = 一个会话"。`qwen serve` 把会话搬进一个**长驻 headless daemon**：

- **1 daemon = 1 workspace × N session**——启动时绑定 cwd 为唯一 workspace，N 个 session 在 daemon 内多路复用；多 workspace = 多 daemon 进程（OS 进程级隔离，systemd / docker / K8s 天然对应）
- **多客户端同时接入**——同一个 session 可以被浏览器（web-shell）、桌面 app、IDE、SDK、甚至另一台机器上的客户端同时观看与操作；事件经 SSE/WebSocket fan-out，prompt 按 FIFO 串行
- **三套 northbound 协议 + MCP 桥**——自有 REST+SSE、标准 ACP（HTTP 与 WebSocket 两种 transport），以及把 daemon 包装成 MCP server 的 `qwen-serve-bridge`；qwen-code 是目前唯一同时打 ACP + MCP 两个开放标准的 daemon（Claude/Cursor 闭门，Goose 仅 ACP）

## 二、关键能力一览

| 能力 | 说明 | 来源 |
|---|---|---|
| 非阻塞 prompt | `POST /session/:id/prompt` 立即返回 `202 {promptId}`，结果经 SSE `turn_complete`/`turn_error` 按 promptId 关联 | #4585 |
| 事件流 + 断线重放 | SSE ring buffer + `Last-Event-ID` 重连重放；ring 淘汰时推 `state_resync_required` 强制 client 重同步 | #4237 / #4360 |
| 多 client 权限投票 | PermissionMediator 4 策略：first-responder（默认）/ designated / consensus / local-only | #4335 |
| MCP 连接池 | workspace 级共享 transport pool，N session 复用同一 MCP server 连接，预算封顶 | #4336 |
| 文件安全边界 | workspace 根限定 + symlink 逃逸拒绝 + 原子写（temp+rename）+ content-hash 前置条件 | #4250 / #4280 |
| OAuth 设备流代理 | RFC 8628 device-flow 由 daemon 代办，凭证不过 client；BrandedSecret 四路脱敏 | #4255 |
| 认证与防护 | Bearer token + `--require-auth` 写路由门禁；`0.0.0.0` 无 token 拒绝启动；`--allow-origin` CORS 白名单；`--rate-limit` 令牌桶 | #3889 / #4236 / #4527 / #4861 |
| 服务端 shell | `POST /session/:id/shell` 直接执行 `!` 命令（不经 LLM），输出流式回放并注入会话历史 | #4576 |
| 会话生命周期 | load / resume / branch / detach / rewind snapshots / recap / stats / context-usage / tasks 快照 | 见 [§03](./03-http-api.md) |
| 可观测性 | OTel tracing（prompt lifecycle / tool spans / per-prompt traceId）+ daemon 文件日志 + `errorKind` 封闭错误分类 | #4321 / #4559 |
| 配置热重载 | `POST /workspace/reload` 统一 settings 重载，只刷真正变化的 key | #4965 |

## 三、9 个核心架构决策

| # | 决策 | 选择 |
|---|---|---|
| 1 | session 跨 client 共享 | 默认 `sessionScope: 'single'`，可共享 |
| 2 | 状态进程模型 | **1 daemon = 1 workspace × N session**（OS 进程级隔离）|
| 3 | MCP 生命周期 | workspace 级共享 transport pool |
| 4 | FileReadCache | per-session 严格私有 |
| 5 | Permission flow | 4 策略 mediator（first-responder 默认）|
| 6 | 多 client 并发 | 同 session prompt FIFO 串行 + 事件 fan-out |
| 7 | 部署模式 | **Mode B（headless `qwen serve`）主线**；Mode A（`qwen --serve` 带 TUI）暂停 |
| 8 | server / client / runtime 边界 | 4 层分离，acp-bridge 独立包 |
| 9 | northbound transport | dual additive：自有 REST+SSE 与标准 ACP 共存 |

每条决策的备选项与依据见 [§02 Architectural Decisions](./02-architectural-decisions.md)。

## 四、文档地图

| # | 文档 | 内容 | 适合 |
|---|---|---|---|
| **01** | [Overview](./01-overview.md) | 术语、架构图、双部署模式、资源经济性 | 第一次了解 daemon |
| **02** | [Architectural Decisions](./02-architectural-decisions.md) | 9 个核心决策的备选项与取舍依据 | 理解"为什么这样设计" |
| **03** | [HTTP API & Protocol](./03-http-api.md) | 完整路由参考 + SSE 事件 + ACP transport + capability 协商 | 写 client / 接 API |
| **04** | [Deployment & Client](./04-deployment-and-client.md) | 部署拓扑、多 client 协调、远端接入 | 设计客户端 / 拓扑选型 |
| **05** | [Security & Permission](./05-permission-auth.md) | 三层权限模型、认证、多租户边界 | 安全评估 / 企业部署 |
| **06** | [Roadmap & Ecosystem](./06-roadmap.md) | 已落地里程碑（压缩史）、进行中、远期方向、业界对照 | 了解进展与方向 |
| **07** | [User Guide](./07-user-guide.md) | 启动 flag、安全配方、部署 shape、客户端接入、故障排查 | **直接上手用** |

### 阅读路径

| 你想 | 读 | 时间 |
|---|---|---|
| 立刻跑起来 | 07 快速上手 + 客户端接入 | ~10 min |
| 快速理解架构 | 本页 → 01 → 02 | ~20 min |
| 写一个 client / 集成 | 03 → 04 → 07 | ~1 h |
| 安全 / 多租户评估 | 05 → 07 安全配方 | ~40 min |
| 全面了解 | 01 → 07 顺序读 | ~2 h |

## 五、团队分工（upstream）

| 方向 | 主力 |
|---|---|
| server 架构 / 路由 / 可靠性 | doudouOUC |
| SDK / ACP transport / 跨 client 同步 | chiga0 |
| web-shell（React web client）| ytahdn |
| MCP 桥（qwen-serve-bridge）| jifeng |
| review / 架构守门 | wenshao（maintainer）|

## 六、进行中与后续

- **backlog**（[#4514](https://github.com/QwenLM/qwen-code/issues/4514)）：pair token + per-client 撤销、`loadSession`/`resume` 转正（去 `unstable_` 前缀）、branch/rewind HTTP 化、multi-daemon 协调等
- **side-channel 协调**（[#4511](https://github.com/QwenLM/qwen-code/issues/4511)）：A2 / A5 两项设计未落地
- **TUI daemon-client adapter**：本地 TUI 接 daemon 的适配层（scope 仍在收敛；本地 TUI 默认 in-process 直连不变）
- **容器化部署模板**：systemd / launchd / nohup 模板已有，Docker / K8s 参考件待补
- **相邻方向**：dynamic workflows（LLM 写编排脚本 fan-out）走 in-process tool 路线、不依赖 daemon，见 [Claude Code Dynamic Workflows Deep-Dive](../claude-code-dynamic-workflows-deep-dive.md)

实施跟踪：[#4175](https://github.com/QwenLM/qwen-code/issues/4175)（rollout tracker）。

---

> **免责声明**：本系列由 codeagents 项目维护，基于源码与 PR 跟踪写成，不代表 Qwen Code 团队官方路线图。daemon 处于 preview 阶段，API（尤其 `unstable_` 前缀路由）可能继续变化。截至 2026-06-12。
