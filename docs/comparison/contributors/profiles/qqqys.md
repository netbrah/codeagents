# qqqys

> Qwen Code（QwenLM/qwen-code）社区贡献者深度档案。本页基于 GitHub 公开数据（用户 API + merged PR API），不含任何非公开信息。

## 速览

| 维度 | 值 |
|---|---|
| GitHub | [@qqqys](https://github.com/qqqys)（user id `266654365`，账号建于 2026-03-09）|
| 真名 | **未公开**（profile `name`/`company`/`location`/`bio` 全空；commit 身份仅为 `qqqys <qys177@gmail.com>`）|
| 2026-05 PR | **21 个 merged**（约 +17k/-2k）—— 安全/权限方向集中爆发 |
| 累计 merged PR | **42 个**（截至 2026-05-31，约 **+24.7k / -2.8k**）|
| 全部 PR（含开闭）| 59（MERGED 42 / CLOSED 12 / OPEN 5）|
| 首次 PR | **#2221**（2026-03-09 merge，与账号注册同日）`fix(core): skip openDiff in YOLO mode to prevent VS Code editor from opening` |
| 月度节奏 | 2026-03: 6 · 2026-04: 15 · 2026-05: 21（逐月加速）|
| 活动范围 | **几乎 100% 集中在 QwenLM/qwen-code**（公开事件仅见 qwen-code + 一次 skills 仓库 star/fork）|
| 角色推断 | **会话/IDE 体验起步、近月转向权限与 hook 安全的高产单仓深度贡献者（推断）** |

> 注：用户最初任务描述的"21 PR / 安全方向"对应的是 **2026-05 当月切片**；qqqys 的累计足迹比这宽得多——4 月以前主力在 session 管理（/resume、/rename、/branch、/goal）与 VS Code IDE companion，5 月才显著转向安全/权限。

## 在 Qwen Code 的工作

qqqys 的 42 个 merged PR 可清晰分为**三个演进阶段**：

### 阶段一（2026-03）：VS Code IDE companion 与终端输入修复

入场即从 IDE 集成与终端兼容性切入：

- **#2221**（首 PR）`skip openDiff in YOLO mode` —— YOLO 模式下不再弹出 VS Code 编辑器。
- **#2428** `feat(mcp): add reconnect command and implement auto-reconnect logic` —— MCP 自动重连。
- **#2437** `Refactors the VS Code file completion system to use fuzzy search` —— IDE 文件补全改模糊搜索。
- **#2445** `Display token usage in the loading/progress indicator` —— 进度指示器显示 token 用量。
- **#2457 / #2501** Windows 路径 URI 处理、代理配置透传给 CLI。
- **#2612 / #2995** 终端 keypress 处理（kitty 序列超时、Linux/Wayland 下 csiU 前缀错误）。

方向：**IDE companion + 终端输入层的兼容性与 UX 修复**。

### 阶段二（2026-04 ~ 2026-05 上旬）：会话生命周期管理（最大代码体量）

这是 qqqys 累计代码量最大的一块，围绕"会话"做了一套完整的列表 / 重命名 / 分支 / 删除 / 恢复体系：

- **#3093**（+3265）`feat(session): add rename, delete, and auto-title generation for session` —— 会话重命名、删除、自动标题生成。代表作之一。
- **#3539** `feat(session): add /branch to fork the current conversation` —— 会话分叉。
- **#3733** `support batch deletion of sessions in /delete` —— 批量删除。
- **#3880**（+1731）`searchable /resume picker with focus-aware modes` —— 可搜索的 /resume 选择器。
- **#3897**（+1327）`perf(core): bound session-list metadata reads to head/tail 64KB; pool buffer; lazy message count` —— 会话列表元数据读取性能优化（只读头尾 64KB + buffer 池 + 懒加载消息计数）。
- **#3692 / #3656** 容错解析：`parseLineTolerant`、从 `}{` 粘连记录恢复 JSONL —— 体现对 session 持久化健壮性的关注。
- **#4123**（+3476，单 PR 最大）`feat(cli): add session-scoped /goal command with judge-driven turn continuation` —— session 级 /goal 命令 + judge 驱动的多轮续跑。
- **#4180**（+1992）`feat(cli): add baseline /doctor memory diagnostics` —— /doctor 内存诊断。
- **#4159** `add fork-session resume flag`、**#4193** `Allow custom output directory for /export`。

方向：**会话管理与 CLI 命令体系（/resume、/rename、/delete、/branch、/goal、/doctor、/export）+ 持久化健壮性与性能**。

### 阶段三（2026-05）：权限 / hook / extension 安全（任务描述对应的当月主线）

5 月明显转向安全与权限治理，与任务描述吻合：

- **#4426** `fix(extension): redact credentialed source diagnostics` —— extension 诊断输出中**凭证脱敏（redact）**。
- **#4371** `fix(core): strip additional dangerous interpreter rules` —— **剥离更多危险解释器规则**（strip）。
- **#4376**（+722）`Emit PermissionDenied hooks for AUTO classifier blocks` —— AUTO classifier 拦截时**发出 PermissionDenied hook**。
- **#4354** `align session hook matcher targets` —— session hook matcher 目标对齐。
- **#4208**（+700）`Add stop hook blocking cap` —— stop hook 阻塞上限。
- **#4112** `fix(dashscope): use URL hostname check instead of regex to avoid ReDoS (CodeQL)` —— 用 hostname 检查替代正则，**修 ReDoS（CodeQL 报告）**。

方向：**权限拒绝可观测性、hook 安全边界、凭证脱敏、危险命令规则、CodeQL 安全告警修复**。

### 进行中（OPEN，沿安全/hook 主线，截至 2026-05-31）

5 个 OPEN PR 全部延续阶段三的安全/hook 方向：
- **#4572** `Harden auto mode self-modification checks`
- **#4545** `Improve hooks matcher display`
- **#4476** `Add AUTO mode denial observability and caps`
- **#4454** `feat(core): add post tool batch hooks`
- **#4377** `feat(core): add user prompt expansion hooks`

> 综合三阶段，"纯安全方向贡献者"是窄化描述。更准确：**先做会话/IDE 体验、再深入权限与 hook 安全的单仓高产贡献者（推断）**。安全只是其最近一个季度的重心。

## 跨仓库社区足迹

- **几乎单仓集中**：`gh search prs --author qqqys` 返回 59 个 PR 全部在 QwenLM/qwen-code（外加自己的 fork `qqqys/qwen-code`）。**无其它上游仓库的 merged PR 记录（公开范围内）**。
- **6 个公开仓库全为 fork、0 star**：`qwen-code`（TypeScript，主战场）、`claude-code`（标注 "Snapshot for Research"）、`A2UI`、`mermaid`、`skills`、`superpowers`。fork 列表显示其关注 agent / skills 生态（Anthropic claude-code、skills 框架），但未见原创公开项目。
- **社交体量小**：followers 4 / following 3 / public_repos 6 —— 典型的"专注单一上游、低社交曝光"画像（推断）。
- **公开事件**仅见 qwen-code 自身 + 一次对 `mattpocock/skills` 的互动，进一步佐证精力高度集中。

## 备注

- **真名/雇主/地域均未公开**，无法确认是否为 Qwen/阿里官方团队成员还是外部社区贡献者；从"账号注册当天即提交首个 core PR 并持续高产"可作弱推断为**有备而来的熟练贡献者**（推断），但不足以判定隶属关系。
- **代码体量口径**：累计 +24.7k/-2.8k 含若干 1000+ 行的 feature PR（#4123 /goal +3476、#3093 session +3265、#4180 /doctor +1992），并非小修小补堆量；同时也有大量 50 行级的精准 fix。
- **CLOSED 中多为重开/拆分**（如 #4229↔#4230 同标题、#4088↔#4123、#3988↔#3989、#3022↔#3539），属正常的 PR 重提/重构迭代，非被拒。
- 与月度榜 [`../2026-05.md`](../2026-05.md) 一致：5 月 21 PR、方向标注"权限/hook/extension 安全"。

## 免责声明

> 本档案基于 GitHub 公开 API 数据，统计窗口至 **2026-05-31（UTC）**。PR 数与代码增删量为 GitHub 报告值（含拆分式 PR，PR 数不完全等于产出体量）。"阶段划分"与"主要方向"为按 PR 标题/时间的人工归纳，非完整列表。所有无法从公开资料证实的判断均标注"（推断）"，未披露项标注"未公开"。账号隶属关系、真实身份不在可核实范围内，请勿据此推断个人信息。
