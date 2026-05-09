# 21 — 扩展到 multi-session daemon 的演进路径

> [← 上一篇：与 Anthropic Managed Agents 对比](./20-vs-anthropic-managed-agents.md) · [下一篇：单 vs 多 Session 设计深度对比 →](./22-single-vs-multi-session-design.md) · [回到 README](./README.md)

> 当前架构走"1 Daemon Instance = 1 Session"路线（[§03 §2](./03-architectural-decisions.md#2-状态进程模型)）。本章评估**当生产规模触及单 session 模型上限时，扩展到 single-process multi-session 模式的工程成本、触发条件、推荐演进路径**——避免现在为可能不会到来的 scale 付永久工程债，同时保留未来可演进的退路。

## 一、TL;DR

**纯迁移到 OpenCode 模式 ~2-3 月**；**但大部分场景不需要——有 ~2-3w 的"资源池化"中段路径能拿 ~80% OpenCode 经济性**。

| 路径 | 工作量 | 收益 | 适用 |
|---|---|---|---|
| **A. 资源池化**（推荐）| **~2-3w** | LSP/MCP/cache 用户级共享，省 ~80% 跨 daemon 资源开销 | 同 user 多 session 场景 |
| **B. Worker threads Hybrid** | ~3-4w | 共享 V8 module + per-session isolate，cold start 50-200ms | 同 workspace 50+ session |
| **C. 纯迁移到 OpenCode 模式** | **~2-3 月** | 完全 cross-session 资源共享 | 大规模 SaaS（100+ session/机）|

**推荐**：YAGNI——直到生产监控显示具体瓶颈再投。多数项目永远停在路径 A 之前。

## 二、单 session 模型的固有代价

| 代价 | 单 daemon 影响 | 触发瓶颈条件 |
|---|---|---|
| **Cold start** | ~1-3s/session（V8 isolate + module loading） | 同 user 短 session 频繁（IDE 自动补全 / IM bot turn-by-turn）|
| **内存 baseline** | ~30-50MB/daemon × N session | 同机 100+ daemon 累积成 GB 级 |
| **跨 session 资源不复用** | 同 workspace N session 各起 LSP/MCP child | 同 workspace 多 session 集中场景（团队 IDE 协作）|
| **OS 资源消耗** | 每 daemon 占进程表项 + 文件描述符 | 单机 1000+ daemon 触 OS 上限 |
| **Cross-daemon 聚合 UI** | "我所有 background tasks"等视图需 orchestrator 聚合 | 高频跨 session 协同操作 |

**只有以上瓶颈实际发生才需要回到 multi-session daemon**——否则单 session 模型的"进程级隔离免费 + 实现简化"是优解。

## 三、路径 C：纯迁移到 OpenCode 模式（~2-3 月，最坏情况）

### 3.1 工作量明细

把单 session 模型省下的复杂度全部加回去：

| 工作项 | 工作量 | 备注 |
|---|---|---|
| `AsyncLocalStorage` Instance ctx 引入 + 改造所有 session-ctx 访问点 | 1-2w | 扫所有读 `daemon.session` 的代码改为从 ALS 取 |
| `Map<workspaceId, Instance>` + 路由 middleware | 1w | OpenCode 已验证模式 |
| LSP / MCP / FileReadCache 加 per-workspace + per-session key 隔离 | 2w | 5 PR#3717/3774/3810/3818/3741 决策都要重做 |
| Permission decision cache 加 sessionId 维度 | 0.5w | PR#3723 evaluatePermissionFlow 扩展 |
| **Subagent isolation 5 PR 套路全做**（Config wrapper / agent-local resources）| **2-3w** | **真痛点**——见 §3.2 |
| Cross-session race condition 测试覆盖 | 1w | OOM / 资源泄漏 / fd 累积 |
| Unified crash recovery（top-level uncaughtException → 仅杀 affected session）| 1w | 单 session 模型下 daemon 崩溃由 orchestrator 重启；OpenCode 模式下要在进程内做 |
| Effect-TS / LocalContext 引入或自研等价物 | 1-2w | 或不引入 Effect-TS，用纯 ALS 包一层 `Instance` API |
| 文档 + e2e + 性能基准 | 1w | |
| **合计** | **~2-3 月** | 类似 OpenCode 实现的总工作量 |

### 3.2 Subagent isolation 5 PR 套路是真痛点

Qwen 团队历史上为单 session 内 subagent 隔离做的 5 PR 套路：

- **PR#3735** subagent isolation Config wrapper（OPEN）
- **PR#3873** agent-local resources（OPEN）
- **PR#3887** isolation refactor（OPEN）
- **PR#3892** isolation tests（OPEN）
- **PR#3707** per-agent ContentGenerator + AsyncLocalStorage（OPEN，与 isolation 强相关）

这些 PR 不是单纯写代码——是反复发现 race / leak 然后修。重做 + 测试覆盖工程量**不可压缩**。

详见 [subagent-display-deep-dive §六.9](../subagent-display-deep-dive.md)。

### 3.3 好消息 + 坏消息

**好消息**：
- PR#3889 child-process API **不需要弃用**——可作为 "Mode B / D 双模式"共存：
  - Mode B（per-session daemon，当前默认）：单 session 场景
  - Mode D（multi-session daemon）：高密度场景
- Orchestrator 路由层不变，只是后端 daemon instance 内部从"1 session"扩到"N session"
- 大部分 channels / WebUI / SDK 代码无需改动（wire 协议不变）

**坏消息**：
- 资源所有权语义破坏性变化：currently `per-daemon = per-session = per-workspace` 三位一体；OpenCode 模式下要拆开。所有依赖"daemon-global = session-global"的代码点都要 grep 找出来改
- Effect-TS / LocalContext 引入会改变现有代码风格（Qwen 现在 config 显式传参；引入 ALS 后部分 API 隐式）
- 性能调优周期长——OOM / fd 泄漏 / 监听器累积等问题在压力测试下才暴露

## 四、路径 A：资源池化（推荐 ~2-3w）

**保持单 session 模型不变，把"昂贵资源"做用户级共享**。这是性价比最高的中段路径。

### 4.1 池化对象与做法

| 池化对象 | 做法 | 工作量 | 收益 |
|---|---|---|---|
| **LSP server** | 用户级 LSP daemon（每用户每 language 一份），daemon instance 通过 LSP-over-IPC（或 LSP HTTP proxy）接入 | 1w | 同 user 多 session 共享 LSP，省 ~90% LSP 内存 |
| **MCP server** | MCP servers 升级为长跑独立进程（不绑 daemon），daemon instance 用 HTTP/IPC 接入 | 1w | 同 user 多 session 共享 MCP children |
| **FileReadCache** | 抽出为用户级共享 cache 进程（IPC + invalidate broadcast），保留 PR#3774 prior-read 守卫语义 | 0.5w | 同 workspace 多 session 共享 cache 命中 |
| **V8 / module 启动** | daemon "warm pool"——orchestrator 预热 N 个 idle daemon，按需绑 session | 0.5w | cold start ~1-3s → ~50-200ms（绑定时延）|
| **Transcript 写入** | 单 SQLite 文件 + WAL 模式（多 daemon 写同一 db），或 Postgres | 0.5w | 跨 daemon transcript 聚合查询 |
| **合计** | | **~2-3w** | **拿到 OpenCode 模式 ~80% 经济性** |

### 4.2 与 OpenCode 模式的取舍

| 维度 | 路径 A（资源池化）| 路径 C（OpenCode 模式）|
|---|---|---|
| LSP / MCP 共享粒度 | 用户级（user-wide）| Workspace 级 |
| FileReadCache 命中率 | 用户级共享 | Workspace 级 |
| V8 启动开销 | warm pool 摊销（~50-200ms）| 0（同进程内复用）|
| 进程隔离强度 | OS 级（path A 不变）| 应用层（path C 牺牲）|
| Crash 半径 | 单 session（不变）| 整 daemon（牺牲）|
| 实现复杂度 | 低（外挂式池化）| 高（破坏性重构）|
| 工作量 | ~2-3w | ~2-3 月 |

**关键洞察**：80% 的 cross-session 资源经济性来自 LSP / MCP / cache 的池化，剩 20% 来自"完全消除 V8 启动开销"。**第一档投入产出比远高于第二档**。

### 4.3 路径 A 不解决什么

| 局限 | 后果 |
|---|---|
| Cold start 仍有 50-200ms（warm pool 绑定时延）| 极高频场景仍不够 |
| OS 进程表项 / fd 数 | 单机 daemon 数仍 ≤ ~500 |
| Cross-daemon 状态强一致性 | 跨 daemon 事务（如批量 permission decision）仍需 orchestrator 协调 |

如果上限是这些，再考虑路径 B / C。

## 五、路径 B：Worker threads Hybrid（中等成本 ~3-4w）

### 5.1 核心思想

**daemon 内用 Worker threads 隔离 session**——共享 V8 module cache + 每 session 独立 isolate。介于"完全 multi-session in single process"和"完全 per-session process"之间。

```
┌──────────── Daemon 进程（路径 B）────────────┐
│                                             │
│  Express HTTP Server                        │
│      │                                      │
│      ↓ session 路由                          │
│  ┌─────────────┬─────────────┬─────────────┐│
│  │ Worker T1   │ Worker T2   │ Worker T3   ││
│  │ (sess-A)    │ (sess-B)    │ (sess-C)    ││
│  │ V8 isolate  │ V8 isolate  │ V8 isolate  ││
│  │             │             │             ││
│  │ + 共享 LSP   │ + 共享 LSP   │ + 共享 LSP   ││
│  │ + 共享 MCP   │ + 共享 MCP   │ + 共享 MCP   ││
│  └─────────────┴─────────────┴─────────────┘│
│                                             │
│      ↑ MessagePort IPC                      │
│  ┌─────────────────────────────┐            │
│  │  共享资源 / Main thread      │            │
│  │  - LSP child (1 份)         │            │
│  │  - MCP children (1 套)      │            │
│  │  - FileReadCache (共享)      │            │
│  └─────────────────────────────┘            │
└─────────────────────────────────────────────┘
```

### 5.2 与路径 C 对比

| 维度 | 路径 B（Worker threads）| 路径 C（OpenCode 模式）|
|---|---|---|
| Cold start | ~50-100ms（thread spawn + isolate init）| ~10ms（同进程函数调用）|
| 隔离强度 | OS thread + V8 isolate | 应用层 ALS |
| Crash 半径 | 单 worker（thread crash 不杀 daemon）| 整 daemon |
| 内存 baseline | ~5-10MB/worker | ~0/session |
| 实现成本 | ~3-4w | ~2-3m |

### 5.3 适用场景

- **同 workspace 50+ session**（资源池化的命中率不够，要更深度共享）
- **要求 cold start < 200ms 但 process-level 隔离非必需**
- **不愿意做完整 OpenCode 模式重构**

但 Worker threads 在 Node.js 生态成熟度有限——native module / fork / debugger 等支持不全。落地前要先做技术评估。

## 六、迁移触发条件（YAGNI 检查清单）

**只在以下信号实际出现后才考虑做迁移**——避免预期式重构：

| 触发信号 | 阈值 | 推荐路径 |
|---|---|---|
| Cold start 占总时延比例 | > 10%（运营监控数据）| → 路径 A warm pool |
| 同 workspace 并发 session | > 5（密集 IDE 自动补全场景）| → 路径 A LSP/MCP 池化 |
| 单机 daemon 实例数 | > 100（SaaS 大客户）| → 路径 B Worker threads |
| LSP / MCP 内存占用占总内存 | > 50%（同 workspace N×30MB 累积）| → 路径 A LSP/MCP 池化 |
| 跨 session 状态强一致性需求 | 出现批量事务 / 跨 session 锁 | → 路径 C |
| 大客户压测要求 | 单机 1000+ session | → 路径 B/C 二选 |

**没有这些信号前**：单 session 模型 + 资源池化（路径 A）已是更优解，避免为可能不会到来的 scale 付永久工程债。

## 七、推荐演进路径

```
Stage 1   ✅ Mode B headless（PR#3889 已实现）
Stage 1.5 🆕 Mode A CLI+HttpServer（~4d）
Stage 2   🆕 orchestrator + multi-daemon（~1-2w）
Stage 3   🆕 对标 OpenCode 完整设计（~1m）
─────────── 单 session 模型上限：~50 session/机器 ──────────
Stage 4   🆕 资源池化路径 A（LSP/MCP/cache 用户级共享 + warm pool） ~2-3w
─────────── 资源池化上限：~200 session/机器 ──────────
Stage 5   ⚠️ Worker threads hybrid 路径 B  ~3-4w
─────────── Worker threads 上限：~500 session/机器 ──────────
Stage 6   ❌ 纯 OpenCode 模式迁移 路径 C  ~2-3m
```

**关键节点**：
- **Stage 1.5/2/3 是默认路径**——满足个人/团队/中等 SaaS 全部场景
- **Stage 4 在生产监控触发后启用**——多数项目永远停在这里
- **Stage 5/6 仅在大规模 SaaS 必需时**——多数项目永远不会到这步

## 八、与各章节的协同

| 章节 | 协同 |
|---|---|
| [§03 §2 状态进程模型](./03-architectural-decisions.md#2-状态进程模型) | 单 session 决策本身——本章是该决策的"未来退路评估" |
| [§05 进程模型](./05-process-model.md) | AsyncLocalStorage / LocalContext / Map<workspaceId, Instance> 章节作为路径 C 实施参考 |
| [§06 MCP / 资源共享](./06-mcp-resources.md) | per-workspace MCP / per-session FileReadCache 章节作为路径 A 池化设计参考 |
| [§08 路线图](./08-roadmap.md) | qwen-code 主线（Stage 1/1.5/2）；本章是 External Reference Architecture 的演进路径之一 |
| [§09 与 OpenCode 对比](./09-comparison-with-opencode.md) | 路径 C 等价于"完全对齐 OpenCode 模型" |
| [§16 HA](./16-high-availability.md) | 资源池化 / Worker threads 都改变 HA 模型——daemon-pool → Worker-pool |
| [§19 长跑稳定性](./19-stability-and-longevity.md) | 路径 C 把跨 session 稳定性挑战重新带回（单 session 模型天然规避）|

## 九、关键不变量

无论选择哪条路径，以下几点保持不变：

| 不变量 | 含义 |
|---|---|
| **Wire 协议** | ACP NDJSON over HTTP+SSE 不变（[§04](./04-http-api.md) / [§10](./10-protocol-compatibility.md)） |
| **Mode A / Mode B** | 双部署模式（[§03 §7](./03-architectural-decisions.md#7-daemon-部署模式clihttpserver-vs-headlesshttpserver)）在所有路径下都保留 |
| **PR#3889 已实现部分** | 路径 A/B/C 都不弃用现有 child-process API，只是增加新 mode |
| **Multi-client per session** | live collaboration 模型（[§03 §6](./03-architectural-decisions.md#6-多-client-并发请求)）所有路径都保留 |

**这意味着**：现在按单 session 模型实施的所有代码，未来任何路径都能复用——不会有"白做了"的部分。

## 十、一句话总结

**纯迁移到 OpenCode 模式 ~2-3 月**——但**大部分场景不需要**：路径 A 资源池化（~2-3w）能拿 ~80% 经济性；路径 B Worker threads（~3-4w）覆盖更深；路径 C 纯迁移仅在大规模 SaaS 必需时投。

**触发条件多数项目永远不出现**——cold start / 内存 / OS 资源 / 跨 session 一致性等瓶颈是高密度 SaaS 场景才暴露。**YAGNI**：直到生产监控显示具体瓶颈再投。

**关键洞察**：选单 session 模型不是"放弃 OpenCode 路径"，而是"延后到生产数据驱动"——所有现有代码都可平滑演进，**没有白做**。

---

[← 上一篇：与 Anthropic Managed Agents 对比](./20-vs-anthropic-managed-agents.md) · [下一篇：单 vs 多 Session 设计深度对比 →](./22-single-vs-multi-session-design.md) · [回到 README](./README.md)
