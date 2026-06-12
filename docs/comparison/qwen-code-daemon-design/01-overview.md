# 01 — Overview

> [← 返回 README](./README.md) · [下一篇：Design Decisions →](./02-architectural-decisions.md)

## TL;DR

**`qwen serve`** 是 Qwen Code 的 HTTP daemon 模式：一个长驻 headless 进程服务一个 workspace，把 AI 会话从"终端进程独占"变成"多客户端可同时接入的服务"。daemon 功能集已合入 main，随 v0.18.0-preview 线发布。

核心模型一行话：

```
1 daemon process = 1 workspace × N session
```

- daemon 内嵌一个 `qwen --acp` child 进程作为 runtime，N 个 session 在这个 child 内多路复用
- 多 workspace = 多 daemon 进程，由 systemd / docker / K8s 按进程自然管理
- 对外（northbound，面向 client 的协议面）提供三套 transport——REST+SSE、ACP HTTP、ACP WebSocket——外加一座 MCP stdio 桥；浏览器、桌面 app、IDE、SDK、MCP client 都能接入

**为什么需要 daemon**：普通 `qwen` 是 1 个终端进程 = 1 个会话，终端关掉会话就消失。daemon 把会话挪进长驻进程：client 可以断开重连、多个 client 同看同操作一个 session、agent 跑在远端机器而本地只做操控界面。

**阅读指引**：想直接上手 → [07 User Guide](./07-user-guide.md)；想理解设计取舍 → [02 Design Decisions](./02-architectural-decisions.md)；完整阅读路径表见 [README](./README.md)。

---

## 一、术语表：两层模型

整个 daemon 体系只有两层核心概念——**daemon process** 与 **session**。这是全系列文档的概念基石：

| 术语 | 定义 | 源码 |
|---|---|---|
| **Daemon process** | `qwen serve` 启动的长驻 HTTP 进程。启动时绑定 cwd 为唯一 workspace；内含 Express 5 server、EventBus（SSE fan-out + ring replay）、以及一个内嵌的 `qwen --acp` child | `packages/cli/src/serve/server.ts`、`packages/acp-bridge/src/bridge.ts` |
| **Session** | 一次独立的 AI 对话：transcript、权限状态、FileReadCache 各自私有。实现上是 ACP `Session` 实例，存放在 `QwenAgent.sessions: Map<sessionId, Session>` | `packages/cli/src/acp-integration/session/Session.ts` |

围绕这两层还有两个角色：

| 术语 | 定义 |
|---|---|
| **Runtime worker** | 真正执行 LLM 调用 / tool / shell / MCP 的进程。当前就是那个内嵌 `qwen --acp` child，daemon 与它之间讲标准 ACP（JSON-RPC over stdio）；未来可替换为 sandbox runner |
| **Daemon client** | 任何经 northbound 协议接入的消费者：web-shell、桌面 app、IDE 插件、SDK、MCP client。client 消费 wire 协议（HTTP/SSE/WS），不直接订阅 daemon 内存中的 EventBus |

**核心约束**：

- 1 daemon **严格绑定 1 workspace**（启动 cwd 决定；想操作另一个仓库就再起一个 daemon）
- 1 daemon **可持 N 个 session**（`sessions: Map` 多路复用，开新 session 不产生新进程）
- 同 workspace 的 N 个 session **共享同一个 `qwen --acp` child**；MCP 连接经 workspace 级共享 transport pool 复用（#4336）；FileReadCache、`Config` 等会话状态保持 per-session 私有

为什么两层就够：daemon 与 `qwen --acp` stdio 模式**心智 1:1 对齐**——daemon 本质是"把 ACP stdio 包装成 HTTP 服务"，没有引入 workspace manager、tenant 之类的中间抽象。

---

## 二、架构图

单个 daemon 的全貌（多 transport + 多客户端）：

```text
            ┌────────────────────── clients ──────────────────────┐
            │  web-shell   桌面 app   IDE 插件 (VS Code / Zed)      │
            │  SDK (TypeScript / Python / Java)   channels   MCP 客户端│
            └──────┬───────────┬───────────┬───────────┬──────────┘
                   │ REST+SSE  │ ACP HTTP  │ ACP WS    │ MCP stdio
                   │ /session/*│ POST /acp │ WS /acp   │ qwen-serve-bridge
                   │ /workspace/*          │           │
┌──────────────────▼───────────▼───────────▼───────────▼──────────┐
│ qwen serve daemon（1 process，cwd = /work/repo-a）                │
│ ├─ Express 5 HTTP server：bearer auth / Host allowlist / CORS    │
│ ├─ EventBus：per-session fan-out + ring replay + Last-Event-ID 重连│
│ └─ qwen --acp child（runtime worker，ACP over stdio）             │
│    ├─ QwenAgent.sessions: Map → { sess-1, sess-2, sess-3 }       │
│    ├─ per-session transcript / FileReadCache / 权限状态           │
│    └─ MCP 连接：workspace 级共享 transport pool                    │
└──────────────────────────────────────────────────────────────────┘
```

纵向看是 4 层边界，每层可独立替换：

```text
client adapters（UI / 输出格式投影）
  → daemon client / protocol 层（typed events + shared reducer，SDK 提供）
  → qwen serve HTTP/SSE control plane
  → runtime worker（当前 qwen --acp child，未来可换 sandbox runner）
```

所有 client——包括 web、IDE、channel——都消费同一套 typed event 流，再投影到各自的 UI 或输出格式；没有任何 client 拿到 daemon 内部状态的特权通道。分层依据详见 [02 Design Decisions](./02-architectural-decisions.md)。

**多 workspace 部署**（本地多项目 / 多用户）就是多个 daemon 进程：

```text
qwen serve (cwd=/work/repo-a) :8001    ← daemon-1
qwen serve (cwd=/work/repo-b) :8002    ← daemon-2
qwen serve (cwd=/work/repo-c) :8003    ← daemon-3
        ↓ systemd / docker-compose / k8s 各管 1 process
OS 进程级隔离 + per-daemon cgroup quota
```

跨 workspace 的协调（IM bot 管多项目、WebUI 多 workspace 概览、IDE multi-root）由 **client / orchestrator 侧**做端点发现与路由——daemon 自身只认"当前 workspace"。多租户 / 跨 daemon 路由 / SaaS 化部署属于外部平台的实施范围，不在 daemon 主线 scope 内（方向性讨论见 [06 Roadmap](./06-roadmap.md)）。

---

## 三、部署模式：为什么 headless 优先

| 模式 | 启动命令 | 形态 | 状态 |
|---|---|---|---|
| **Mode B：headless server** | `qwen serve [--port N]` | 纯 HTTP 服务，无 TUI | **主线** |
| **Mode A：CLI + server** | `qwen --serve [--port N]` | 本地 TUI + 顺带开 HTTP 端口 | 暂停推进 |

两者的区别：Mode B 是"为服务而生"的独立 daemon；Mode A 是"我正在终端里用的会话，顺带暴露出去让手机 / 浏览器接管"。选择 Mode B 作主线的依据：

- **单一 runtime 表面**：所有 client 走同一套 HTTP/SSE 协议，协议缺口在真实 client 接入时暴露得最快；不需要同时维护"in-process TUI 视角"和"wire 视角"两套语义
- **Mode A 是 Mode B 的子集**：Mode A 的价值（远端接管本地会话）依赖 daemon 侧 event / control / client contract 全部成熟——先做 B，A 所需的一切自然就绪，届时再评估
- **本地体验不受影响**：单机直接用 `qwen` 的 TUI 默认仍是 in-process 直连，不经网络、零 daemon 开销

### 部署形态与同址不变式

Mode B 之下有 3 种部署形态：

| 形态 | daemon 在哪 | workspace 在哪 | 推荐 |
|---|---|---|---|
| 本地单机 | 本机 | 本机 | ✅ 默认体验，daemon 在 loopback 起 |
| 远端 runtime | 远端主机 / pod | **同一**远端主机 / volume | ✅ Codespaces 风格：远端跑，本地看 |
| 本地 workspace + 远端 daemon | 远端 | 本机 | ❌ 不推荐 |

第三种不工作的原因即**核心不变式——daemon host = workspace host**：

> 文件读写、shell、LSP、provider 凭证、MCP server、skill 发现与执行，**全部在 daemon 所在环境求值**。daemon 不可能看到另一台机器上的文件。想"远端跑、本地看"，把 workspace 一起放到远端。

部署拓扑与多 client 协调的完整讨论见 [04 Deployment & Client](./04-deployment-and-client.md)。

---

## 四、资源经济性

daemon 模型的成本论证分两个方向：同 workspace 加 session 几乎免费，加 workspace 按进程线性计价。

**同 workspace N session**（估算，继承 ACP 原生 multi-session）：

| 同 workspace session 数 | 内存 RSS | 说明 |
|---|---|---|
| 1 | ~60-100 MB | baseline |
| 5 | ~60-100 MB | N session 共享同一个 `qwen --acp` child |
| 10 | ~80-150 MB | 同上，增量主要是各 session 的 transcript |

边际成本低的来源：

- **进程零增量**：开新 session 只是 Map 里多一条记录，attach 已有 daemon 开新 session 冷启动 <200ms（无进程 spawn）
- **OAuth refresh 1× per daemon**：N 个 session 共享凭证刷新
- **MCP 连接封顶**：workspace 级 transport pool 让 N session 复用同一组 MCP server 连接——例如 4 个 session、budget 为 2 时，MCP child 进程上限是 2 而非 8

**多 workspace 部署成本**：M 个 workspace = M 个 daemon 进程，每个 daemon 的 HTTP front 部分 baseline ~30-50 MB（Express server + EventBus + auth）。例如 5 workspace × 5 session 同机 ≈ ~450-750 MB。

这笔开销买到的是：

- **OS 进程级隔离**：跨 workspace 即跨进程，是操作系统能给的最强隔离
- **资源 quota 直接对应**：systemd `MemoryMax=` / docker `--memory` / K8s pod limit 天然就是 per-workspace quota，无需自建记账
- **blast radius 最小**：一个 daemon crash 只影响一个 workspace
- **可观测性直接**：`htop` / `ps` 里 1 行进程 = 1 个 workspace
- **心智简单**：保持 daemon / session 两层，无中间抽象

反方向的备选——单个 super-daemon 进程内管 M 个 workspace——省下的内存有限，却要在应用层自建 workspace 路由、资源记账与故障隔离。早期曾存在这样的多 workspace 路由层，后被移除（#4113），代码里偶见的 `byWorkspaceChannel` / `ChannelInfo` 等符号即其遗留。完整取舍见 [02 Design Decisions](./02-architectural-decisions.md)。

---

## 五、现状

截至 2026-06-12：

- **在 main 上**：daemon 功能集经首次 `daemon_mode_b_main → main` 反向 merge 进入 main（#4490），随 **v0.18.0-preview** 线发布
- **开发节奏**：日常开发在 `daemon_mode_b_main` 长期 integration 分支进行，周期性反向 merge 回 main
- **协议面**：REST+SSE 全量路由；ACP HTTP transport `/acp`（#4472）与 ACP WebSocket（#4773）作为标准协议入口，并通过 29 个 `_qwen/*` extension method 与 REST 达成能力对等（#4827）；`qwen-serve-bridge` 把 daemon 包装成 MCP server 供 MCP-native client 使用（#4555）
- **prompt 语义**：`POST /session/:id/prompt` 非阻塞，立即返回 `202 {promptId}`，结果经 SSE `turn_complete` / `turn_error` 按 promptId 关联（#4585）
- **客户端**（均在 main 的 `packages/` 下）：web-shell（React web client）、webui（共享 daemon UI 层）、sdk-typescript / sdk-python / sdk-java、官方桌面 app（#3778）、vscode-ide-companion、zed-extension、channels（IM 接入）
- **进行中**：pair token 与 per-client 撤销、`loadSession` / `resume` 路由去 `unstable_` 前缀等 backlog 项（#4514）

---

> **免责声明**：daemon 处于 preview 阶段，API（尤其 `unstable_` 前缀路由）可能继续变化。本文基于源码与 PR 跟踪写成，不代表 Qwen Code 团队官方路线图。数据截至 2026-06-12。

下一篇：[02 — Design Decisions →](./02-architectural-decisions.md)
