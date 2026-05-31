# doudouOUC

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@doudouOUC](https://github.com/doudouOUC) |
| 真名 / 显示名 | jinye（GitHub profile `name` 字段） |
| 内部花名 | 铭戎 |
| 公司 / 组织 | Alibaba Cloud（阿里云，profile `company` 字段）；内部 Qwen 团队（maintainer 2026-05-31 确认） |
| 地区 | Beijing（北京） |
| 注册时间 | 2012-12-10（账号 ID 3007091，老账号） |
| 2026-05 qwen-code PR | 100（+184k/-26k） |
| qwen-code 累计 merged PR | 124（首 PR #3041 起，全部集中在本仓库） |
| 首次 qwen-code PR | 2026-04-09 / #3041（`[codex] fix checkpointing init in non-repo directories`） |
| 角色推断 | 内部 Qwen 团队（maintainer 2026-05-31 确认）；daemon/serve（守护进程模式）核心开发者 + telemetry（OTel 可观测性）owner（推断） |

## 在 Qwen Code 的工作

- 主要方向：
  - **daemon / serve（守护进程模式 B，F-series）主力**：`serve` HTTP 路由、SSE 事件流、会话生命周期、ACP bridge 包拆分（#4175 系列）、权限多客户端协调。2026-05 当月 PR 大量集中于此。
  - **telemetry / OpenTelemetry 可观测性 owner（推断）**：长期跟进 #3731（分 Phase 1/2/4a）、#3847（traceId/spanId 注入日志做 OTel 关联）、层级化 session tracing span、GenAI semconv 双发、TTFT 采集、metric cardinality 控制。从 5 月初到月末贯穿全 phase。
  - **core 健壮性 / rewind / 压缩**：reactive compression（上下文溢出时反应式压缩）、/rewind 会话回退命令 + 文件恢复、AbortSignal 监听器泄漏修复。
  - **Python SDK + 发布流程**：Python SDK 实现（#3494，for #3010）、PyPI 发布 workflow、release helper 工具。
- 代表作 PR：
  - [#4175 系列 acp-bridge / serve 重构](https://github.com/QwenLM/qwen-code/pull/4319) — 把 acp-bridge 拆成自给自足的包并引入 BridgeFileSystem seam，是 daemon 模式 F1 的机械化分拆主干（多 PR：#4295/#4298/#4304/#4319/#4334 等）。
  - [#3441 conversation rewind feature](https://github.com/QwenLM/qwen-code/pull/3441) — 新增双击 ESC + `/rewind` 命令的会话回退能力，后续 #4064 又补上文件恢复。
  - [#3879 reactive compression on context overflow](https://github.com/QwenLM/qwen-code/pull/3879) — 上下文溢出时的反应式压缩，core 上下文管理关键能力。
  - [#4071 hierarchical session tracing spans](https://github.com/QwenLM/qwen-code/pull/4071) — telemetry 层级化 session 追踪 span，奠定可观测性 trace 树。
  - [#3494 Python SDK implementation](https://github.com/QwenLM/qwen-code/pull/3494) — 为 #3010 实现 Python SDK，开辟 SDK 方向。
- 技术专长（推断）：分布式/服务端架构（HTTP daemon、SSE、ACP 协议、会话状态机）、OpenTelemetry 可观测性（trace/span/metric/semconv）、Node/TypeScript 工程化与包拆分重构、Python SDK 与发布流水线；早期个人仓库偏系统/算法（C/C++ 的 dlmalloc、FreeNOS、ndn-cpp）说明有较扎实的系统编程底子（推断）。

## 跨仓库 / 社区足迹

- 跨仓库 PR 量 / 其他活跃项目：公开 PR 总计 145，**全部集中在 QwenLM/qwen-code**（无其他外部仓库 PR 足迹），是高度聚焦于该项目的内部/全职贡献者特征（推断）。
- 自有项目：11 个公开仓库，多为学习/系统类（`algorithms`、`leetcode`、`Gray-Hat-Python` 原创；`dlmalloc`、`FreeNOS`、`ndn-cpp`、`DeathHandler`、`tornado` 等为 fork），另有 `code_agent`（JS）与 `qwen-code`（fork）；均 0 star，属个人练习/工作 fork 性质。
- 个人主页 / 博客 / 社交：未公开（profile `blog`/`twitter_username`/`email` 字段均为空）。
- 背景推断：阿里云员工，常驻北京，作为 Qwen Code 团队成员深度参与 daemon 模式与可观测性建设（推断，依据 profile 的 company=Alibaba Cloud + PR 全部落在 QwenLM 官方仓库且为 COLLABORATOR）。

## 备注 / 不确定项

- 真名仅取自 GitHub profile 显示名 “jinye”，未做进一步身份关联。
- 任务给出的「main 72 + daemon_mode_b_main 27 ≈ 100」当月数据中，`daemon_mode_b_main` 是非默认基分支，GitHub search/`gh pr list` 的 merged 计数（累计 124）已涵盖这些跨基分支 merge；当月 100 与累计 124 不冲突。
- 关注者 12 / 关注 9，社区影响力指标较低，符合“项目内部工程师”而非“社区网红”画像（推断）。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份/背景推断已标注“（推断）”，未公开信息标注“未公开”，不含任何私人信息。
