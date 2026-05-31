# Qwen Code 贡献者档案集 · 2026 年 5 月活跃者

> 对 [2026 年 5 月月活跃贡献者](../2026-05.md) 里 **当月 ≥3 merged PR 的 19 人**逐一建的深度档案。每人一个文件，含 GitHub 资料 / 在 qwen-code 的累计足迹 / 跨仓库与社区背景 / 角色。
>
> **内部 vs 社区**：项目 maintainer（高铁 / wenshao）2026-05-31 提供了内部团队 **花名（alias）↔ GitHub** 映射，据此区分**内部 Qwen 团队（阿里生态）**与**社区/外部**贡献者。19 人中 14 内部 + 5 社区。
>
> **关键口径提醒**：月活页表格用的是**当月 PR 数**；本索引「累计 PR」列是该人在 qwen-code 的**历史累计 merged PR**（多数远高于当月数）。数据截至 2026-05-31，公开 GitHub API + 公开 OSS 足迹；**不收录个人联系方式（邮箱已脱敏）**。

## 内部 Qwen 团队（阿里生态，maintainer 2026-05-31 确认）

| 花名 | 贡献者 | 真名 / 显示名 | 累计 PR | 首次 PR | 角色 / 方向 |
|---|---|---|--:|---|---|
| 愚远 | [tanzhenxin](./tanzhenxin.md) | 未公开 | 256 | 2025-07-30 | core/runtime 长期核心（provider 适配 / subagent / compaction）|
| 铭戎 | [doudouOUC](./doudouOUC.md) | jinye（北京）| 124 | 2026-04-09 | **daemon 团队**：F-series 后端核心 + telemetry owner |
| 今井 | [yiliang114](./yiliang114.md) | 易良（杭州）| 115 | 2025-12-10 | VSCode IDE companion / WebUI / LSP owner |
| 高铁 | [wenshao](./wenshao.md) | 温绍锦（杭州）| 108 | 2026-02-13 | **核心 maintainer / 守门 reviewer**；fastjson·Druid 作者、OpenJDK Committer |
| 甄卓 | [LaZzyMan](./LaZzyMan.md) | 顾盼（lazzy.site）| 92 | 2026-01-06 | 核心能力/模型层（compaction / 权限 / worktree / auto-memory）|
| 山果 | [pomelo-nwu](./pomelo-nwu.md) | pomelo | 65 | 2025-07-29 | auth/provider owner + i18n 奠基 + 流程治理 |
| 俊宇 | [DennisYu07](./DennisYu07.md) | 未公开 | 42 | 2026-01-07 | **hooks 系统核心架构者 / 事实 owner** |
| 衍星 | [qqqys](./qqqys.md) | 未公开 | 42 | 2026-03-09 | IDE → 会话生命周期 → 权限安全 |
| 高琦 | [chiga0](./chiga0.md) | ChiGao | 40 | 2026-04-07 | **daemon 团队第 2 作者**：SDK/TUI 前端 + ACP HTTP transport |
| 绿龙 | [DragonnZhang](./DragonnZhang.md) | Dragon（@QwenLM）| 29 | 2025-12-11 | SDK / VSCode / CLI |
| 克竟 | [BZ-D](./BZ-D.md) | Edenman（杭州）| 11 | 2026-04-10 | CLI/终端交互（/diff + headless 护栏）|
| 元昌 | [huww98](./huww98.md) | 胡玮文（阿里云 ACK）| 10 | 2026-03-04 | 性能/构建/内存"外科手术型"（主业云原生存储）|
| 维斌 | [dreamWB](./dreamWB.md) | 未公开 | 9 | 2026-04-14 | 终端输入交互 / 跨终端兼容前端 |
| 钉萁 | [ytahdn](./ytahdn.md) | 未公开 | 3 | 2026-05-21 | **daemon 团队第 3 作者**：web-shell 全栈 |

## 社区 / 外部贡献者

| 贡献者 | 真名 / 显示名 | 累计 PR | 首次 PR | 角色 / 方向 |
|---|---|--:|---|---|
| [B-A-M-N](./B-A-M-N.md) | "John London"（化名，推断）| 13 | 2026-04-12 | local-first / Ollama 配置认证方向 |
| [kkhomej33-netizen](./kkhomej33-netizen.md) | 未公开 | 12 | 2026-03-06 | CLI/终端运行时 |
| [Jerry2003826](./Jerry2003826.md) | JerryLee（墨尔本）| 8 | 2026-04-24 | 跨 AI Agent 生态 core-fix（含 5/25 burst 观察）|
| [shenyankm](./shenyankm.md) | Yan Shen（南京）| 8 | 2026-04-15 | CLI/TUI 交互层 + i18n UX |
| [he-yufeng](./he-yufeng.md) | Yufeng He（**Moonshot AI / Kimi**，香港）| 6 | 2026-05-28 | drive-by bug-fix；⚠️ 供职竞品 Moonshot |

## 其他内部团队成员（当月 <3 PR，未单独建档）

> maintainer 提供的内部花名映射里，还有 9 人当月活跃度低于 ≥3 PR 门槛，未单独建档，记于此备查：

沐目（zhangxy-zju，NotebookEdit）/ 继风（jifeng，**daemon 团队**：qwen-serve-bridge MCP server）/ 西岐（finleymaa）/ 乾离（gwinthis，telemetry skill-RT）/ 飞伦（ellasafy）/ 叶公（callmeYe）/ 清学（cwtuan）/ 沅沅（yuanyuanAli，Feishu channel）/ 垂虹（Mingholy）

## 几个跨档案观察

- **内部团队是绝对主力**：累计 PR 上百的（tanzhenxin 256 / doudouOUC 124 / yiliang114 115 / wenshao 108）全是内部；社区贡献者累计多在个位到十几。
- **daemon 团队 4 人** = 铭戎 doudouOUC（后端 F-series）+ 高琦 chiga0（SDK/TUI 前端）+ 钉萁 ytahdn（web-shell）+ 继风 jifeng（MCP bridge）。
- **子系统 owner 清晰**：hooks → 俊宇 DennisYu07 / auth → 山果 pomelo-nwu / VSCode-WebUI-LSP → 今井 yiliang114 / compaction+模型 → 甄卓 LaZzyMan / daemon → 铭戎+高琦。
- **最意外**：社区贡献者里 [he-yufeng](./he-yufeng.md) 自报供职 **Moonshot AI（Kimi）**——竞品方对 Qwen Code 的纯外部贡献。
- **最知名 OSS 背景**：[wenshao](./wenshao.md)（高铁，fastjson ~25.6k★ / Druid ~28.2k★ / OpenJDK Committer）。

---

> **免责声明**：全部档案仅基于公开 GitHub 资料、公开 OSS 活动，以及 maintainer 提供的内部花名映射整理，截至 2026-05-31。**不收录个人联系方式（邮箱已脱敏）**；剩余推断已标「（推断）」、未公开信息标「未公开」。累计 PR 数为统计时点 GitHub 报告值，会随时间变化。如有当事人希望调整或移除，请提 issue。
