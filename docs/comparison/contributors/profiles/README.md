# Qwen Code 贡献者档案集 · 2026 年 5 月活跃者

> 对 [2026 年 5 月月活跃贡献者](../2026-05.md) 里 **当月 ≥3 merged PR 的 19 人**逐一建的深度档案。每人一个文件，含 GitHub 资料 / 在 qwen-code 的累计足迹 / 跨仓库与社区背景 / 角色推断。
>
> **关键口径提醒**：月活页表格用的是**当月 PR 数**；本索引「累计 PR」列是该人在 qwen-code 的**历史累计 merged PR**（多数远高于当月数）。数据截至 2026-05-31，全部来自公开 GitHub API + 公开 OSS 足迹，身份/背景推断已在各档案标注「（推断）」。

## 汇总表

| 贡献者 | 真名 / 显示名 | 累计 PR | 首次 PR | 角色 / 方向 |
|---|---|--:|---|---|
| [doudouOUC](./doudouOUC.md) | jinye（阿里云，北京）| 124 | 2026-04-09 | daemon F-series 核心 + telemetry owner（推断）|
| [wenshao](./wenshao.md) | 温绍锦（阿里，杭州）| 108 | 2026-02-13 | **核心 maintainer / 守门 reviewer**；fastjson·Druid 作者、OpenJDK Committer |
| [yiliang114](./yiliang114.md) | 易良（阿里，杭州）| 115 | 2025-12-10 | VSCode IDE companion / WebUI / LSP owner |
| [chiga0](./chiga0.md) | ChiGao | 40 | 2026-04-07 | **daemon 团队第 2 作者**；SDK/TUI 前端 + ACP HTTP transport |
| [LaZzyMan](./LaZzyMan.md) | 顾盼（武汉大学，lazzy.site）| 92 | 2026-01-06 | 核心能力/模型层（compaction / 权限 / worktree / auto-memory）|
| [tanzhenxin](./tanzhenxin.md) | 未公开 | 256 | 2025-07-30 | core/runtime 长期核心（provider 适配 / subagent / compaction）|
| [qqqys](./qqqys.md) | 未公开 | 42 | 2026-03-09 | 单仓高产（IDE → 会话生命周期 → 权限安全）|
| [pomelo-nwu](./pomelo-nwu.md) | pomelo（蚂蚁，杭州）| 65 | 2025-07-29 | auth/provider owner + i18n 奠基 + 流程治理 |
| [DragonnZhang](./DragonnZhang.md) | Dragon（@QwenLM）| 29 | 2025-12-11 | Qwen 团队 collaborator；SDK / VSCode / CLI |
| [kkhomej33-netizen](./kkhomej33-netizen.md) | 未公开 | 12 | 2026-03-06 | CLI/终端运行时社区贡献者（推断）|
| [Jerry2003826](./Jerry2003826.md) | JerryLee（墨尔本）| 8 | 2026-04-24 | 跨 AI Agent 生态社区 core-fix 贡献者（含 5/25 burst 观察）|
| [B-A-M-N](./B-A-M-N.md) | "John London"（化名，推断）| 13 | 2026-04-12 | local-first / Ollama 配置认证方向外部贡献者 |
| [he-yufeng](./he-yufeng.md) | Yufeng He（**Moonshot AI / Kimi**，香港）| 6 | 2026-05-28 | 外部 drive-by bug-fix；⚠️ 供职竞品 Moonshot |
| [shenyankm](./shenyankm.md) | Yan Shen（南京）| 8 | 2026-04-15 | CLI/TUI 交互层 + i18n UX 外部贡献者 |
| [huww98](./huww98.md) | 胡玮文（阿里云 ACK / K8s）| 10 | 2026-03-04 | 性能/构建/内存"外科手术型"；主业云原生存储 |
| [dreamWB](./dreamWB.md) | 未公开 | 9 | 2026-04-14 | 终端输入交互 / 跨终端兼容前端（推断）|
| [DennisYu07](./DennisYu07.md) | 未公开 | 42 | 2026-01-07 | **hooks 系统核心架构者 / 事实 owner** |
| [BZ-D](./BZ-D.md) | Edenman（南京大学 + 阿里云，杭州）| 11 | 2026-04-10 | CLI/终端交互（/diff + headless 护栏）|
| [ytahdn](./ytahdn.md) | 未公开 | 3 | 2026-05-21 | **daemon 团队第 3 作者**；web-shell 全栈 |

## 几个跨档案观察

- **核心团队 vs 社区长尾分层明显**：累计 PR 上百的（tanzhenxin 256 / doudouOUC 124 / yiliang114 115 / wenshao 108 / LaZzyMan 92 / pomelo-nwu 65）几乎都是阿里/蚂蚁/Qwen 系或长期 collaborator；社区贡献者（Jerry2003826 / B-A-M-N / he-yufeng / shenyankm / kkhomej33-netizen）累计多在个位到十几。
- **daemon 团队 = doudouOUC（后端 F-series）+ chiga0（SDK/TUI 前端）+ ytahdn（web-shell）**，分工互补；jifeng（MCP bridge）当月 2 PR 未达 ≥3 门槛故未单列。
- **子系统 owner 清晰**：hooks → DennisYu07 / auth → pomelo-nwu / VSCode-WebUI-LSP → yiliang114 / compaction+模型 → LaZzyMan / daemon → doudouOUC+chiga0。
- **最意外的一条**：[he-yufeng](./he-yufeng.md) 自报供职 **Moonshot AI（Kimi）**——竞品方，对 Qwen Code 的 6 个 fix 是纯外部社区贡献。
- **最知名 OSS 背景**：[wenshao](./wenshao.md)（fastjson ~25.6k★ / Druid ~28.2k★ / OpenJDK Committer）。
- **真业不在 AI Agent 的高手**：[huww98](./huww98.md) 全网 561 merged PR 绝大多数在云原生存储（alibaba-cloud-csi-driver / Kubernetes 生态），qwen-code 只是顺手，但能把修复打到 ink 依赖根部。

---

> **免责声明**：全部档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。身份/雇主/背景的推断已在各档案标注「（推断）」，未公开信息标注「未公开」，不含任何私人信息。累计 PR 数为统计时点 GitHub 报告值，会随时间变化。
