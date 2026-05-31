# chiga0

> Qwen Code（QwenLM/qwen-code）daemon 团队第 2 作者，TUI / SDK UI 层与跨客户端协议主力贡献者。

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@chiga0](https://github.com/chiga0)（id 24784430，注册于 2016-12） |
| 真名 | **ChiGao**（GitHub profile `name` 字段；真实姓名拼写未公开） |
| 内部花名 | 高琦 |
| 个人站 | chigao.site｜location "Happy City"（占位，真实地点未公开）｜邮箱未公开 |
| 2026-05 PR | **36 merged**（+51.9k / -3.9k），分支拆分 main 27 + daemon_mode_b_main 9 |
| 累计 merged PR（qwen-code） | **40**（+57.1k / -4.3k）；另有 6 open + 27 closed |
| 首次 merged PR | **2026-04-07**，#2770 `feat: support verbose and compact mode switcher with ctrl-o` |
| 最新 merged PR | 2026-05-28，#4585（daemon non-blocking POST /prompt） |
| 公司/组织 | 内部 Qwen 团队（阿里生态，maintainer 2026-05-31 确认） |
| 角色推断 | **daemon 团队第 2 作者**（内部 Qwen 团队成员，maintainer 2026-05-31 确认）—— 仅次于 doudouOUC（99 PR）；专注 daemon **SDK / TUI 前端 + ACP 协议传输 + 跨客户端实时同步** |

## 在 Qwen Code 的工作

加入时间约 2026-04 上旬，起点是 TUI 体验优化（compact / verbose 模式切换），随后两个月快速成长为 daemon track 的核心前端 + 协议作者。工作可分四条主线：

**1. daemon SDK / 共享 UI 层（体量最大）**
- #4328 `feat(daemon): add shared UI transcript layer`（+6.1k/-2.0k）——daemon 共享 UI transcript 基础设施
- #4353 `feat(sdk/daemon-ui): unified completeness follow-up to #4328`（+8.5k/-0.1k）——上述的完备化跟进，是其单 PR 体量最大的一笔
- #4201 `DaemonSessionClient skeleton` → #4225 `harden daemon session client` → #4231 `daemon-stamped client identity` → #4232 `session-scoped permission route`：SDK 侧 daemon 会话客户端从骨架到加固的连续演进

**2. ACP 协议 + HTTP 传输**
- #4472 `feat(daemon): ACP Streamable HTTP transport at /acp [RFD #721]`（+6.1k/-0.3k）——为 daemon 实现 ACP（Agent Client Protocol）的 Streamable HTTP 传输层
- #4217 `feat(protocol): typed daemon event schema v1`（+1.1k）——daemon 事件的强类型 schema v1

**3. 跨客户端实时同步 + side-channel 协调**
- #4484 `feat(daemon+sdk): cross-client real-time sync completeness`（+0.8k）
- #4510 `cross-client sync follow-up cleanup`（epoch-reset resync / approval-mode serialization / catch-up indicator，+1.2k）
- #4546 `in-session model switch reaches the bus (A1)` / #4539 `voterClientId to permission_resolved (A4)` / #4557 清理 —— 一组以 A1/A4/A5 编号的 side-channel 协调改动（对应 open PR #4511 的 `docs(design): daemon side-channel coordination` 设计文档）
- #4585 `non-blocking POST /prompt — return 202 with promptId`（+0.5k）—— daemon HTTP API 异步化

**4. TUI 渲染 / 启动性能（main 分支）**
- 渲染体验：#3680 expand TUI markdown rendering（+7.4k）、#3710 自定义 banner、#3591 TUI flicker foundation、#3352 dual-output sidecar mode、#4050 表格 ANSI color 跨行保留、#3968 窄终端渲染
- 性能：#3994 progressive MCP availability（MCP 不再阻塞首次输入，+2.1k）、#4070 code-split lowlight 降低启动 V8 解析成本、#4166 setTools() 刷新 systemInstruction
- ink 依赖治理：#3860 升级 ink 6→7 → #4083 因 Static-remount 回归 revert 7→6 → #4119 在上游修复落地后再次升级到 7.0.3，期间把根因修复推到 ink 上游（见下）

> 分支说明：daemon 工作放在长期 integration 分支 `daemon_mode_b_main`，因此其 PR 分散在 `main`（27）与 daemon 分支（9）两处。详见 [daemon 设计文档](../../qwen-code-daemon-design/README.md)。

## 跨仓库社区足迹

| 仓库 | 关系 | 内容 |
|---|---|---|
| [vadimdemedes/ink](https://github.com/vadimdemedes/ink) | 上游修复 | 2 merged PR（#948 / #950，2026-05）——修复 `<Static>` 通过 key change remount 时丢新 item / 残留 stale 输出。**与其在 qwen-code 的 ink 6↔7 升级回退闭环直接相关**：把 TUI 回归的根因修到了 ink 上游 |
| [antvis/G6](https://github.com/antvis/G6) | 文档贡献 | 4 merged PR（2025-03~04）——改造内置布局文档（Dagre / AntvDagre / Mindmap），中文文档为主 |
| chiga0/custom-agent + custom-agent-docs | 个人项目 | 自建 TypeScript "Custom Agent" 实现仓 + 规划/架构/ADR/多代理协调文档仓（2026-05 创建，活跃推送）——（推断）个人在探索自研多代理编程 agent |
| 其他个人 repo | 个人项目 | ai-story-game（TS）、rss-reader（PWA，2 star）、react-tailwind-shadcn-template、open-data-template；fork 了 bolt.diy / warp / claude-code-source-code / claw-code 等 agent 相关项目 |

跨仓库 PR 合计约 102（含个人仓 fork PR），公开仓库 13 个。技术栈集中在 **TypeScript + React + 终端 UI（ink）+ Vite/Tailwind/shadcn 前端工具链**。

## 备注

- **专长（推断）**：前端 / 终端 UI 工程师背景——证据链一致：qwen-code 的 TUI 渲染与 ink 升级治理、向 ink 上游提交 React reconciler 级修复、个人仓全是 React/Vite/shadcn 前端模板与 PWA。在 daemon track 中承担"SDK + UI 层 + 协议传输"这一前端到协议的纵向切面，与 doudouOUC 偏后端/telemetry/F-series 形成分工互补。
- **角色（推断）**：daemon 团队第 2 作者，是把 daemon 能力暴露给客户端（TUI / IDE / channel / SDK 多端）的关键人，主导 ACP HTTP transport 与跨客户端同步。
- 真实姓名仅有 GitHub `name`=ChiGao，公司 / 真实地点 / 邮箱 / 社交账号均**未公开**。
- 与本项目（wenshao/codeagents）也有交集：fork 了 codeagents 仓并提交了 PR #131–#134（属本仓社区贡献，非 Qwen 上游）。

---

> **免责声明**：截至 **2026-05-31**，数据来自 GitHub 公开 API（用户 profile / merged PR 列表 / 跨仓库 search / repos 列表），PR 数与代码增删量为 GitHub 报告值。daemon 作者含大量拆分式小 PR，PR 数不完全等于产出体量。标注"（推断）"的为基于公开信息的推测，非本人确认；标"未公开"的为公开资料中查不到。统计口径与官方 [Contributors wiki](https://github.com/QwenLM/qwen-code/wiki/Contributors)（累计 merged PR 全量榜）不同，不可直接对照名次。
