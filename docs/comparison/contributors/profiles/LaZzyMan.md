# LaZzyMan

> Qwen Code（QwenLM/qwen-code）核心贡献者深度档案。本页基于公开 GitHub 资料（用户 API、merged PR 列表、跨仓库搜索）整理，推断性结论均标注「（推断）」。

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@LaZzyMan](https://github.com/LaZzyMan)（id 23273105，2016-11 注册）|
| 真名 | 顾盼（Gu Pan，GitHub profile `name` 字段公开）|
| 单位 / 地区 | Wuhan University（武汉大学，profile `company` 字段）|
| 个人站 / 邮箱 | http://lazzy.site ｜ zeusdream7@gmail.com（公开）|
| 2026-05 当月 | **22 merged PR，+38.8k/-2.6k**（[月活榜](../2026-05.md) 第 5）|
| 累计（qwen-code）| **92 merged PR，+100.3k/-22.8k**（all-state 103）|
| 首次 merged PR | **2026-01-06**（[#1383](https://github.com/QwenLM/qwen-code/pull/1383) 非交互模式工具执行反馈）|
| 最近 merged PR | 2026-05-29（[#4623](https://github.com/QwenLM/qwen-code/pull/4623)）|
| 月度分布 | 1月 17 / 2月 11 / 3月 25 / 4月 17 / 5月 22（持续高频，无断档）|
| 角色推断 | Qwen Code **核心能力 / 模型层重度贡献者**——上下文压缩、权限/审批、worktree、auto-memory/skill 四大子系统的主要设计与重构者（推断为团队成员或长期 maintainer 级协作者）|

## 在 Qwen Code 的工作

5 个月、92 个 merged PR、净增约 +10 万行，贡献横跨多个核心子系统。按主题归纳：

### 1. 上下文压缩（compaction）—— 反复打磨的主线
- [#4599](https://github.com/QwenLM/qwen-code/pull/4599)（5月，BREAKING，+3034/-1315）：用 **summary + restoration attachments** 替换 tail-preservation 压缩——5 月的二次重构主力。
- [#4345](https://github.com/QwenLM/qwen-code/pull/4345)（5月，BREAKING，+4076/-227）：重设计 auto-compaction 阈值为**三档 ladder**。
- [#4101](https://github.com/QwenLM/qwen-code/pull/4101)（5月，+1530）：压缩前剥离 inline media。
- [#4623](https://github.com/QwenLM/qwen-code/pull/4623)（5月）：截图触发的压缩在通知里标注准确。
- [#2659](https://github.com/QwenLM/qwen-code/pull/2659)：`/compress` 正确处理 tool-heavy 会话。
- （推断）compaction 子系统在该项目内基本由其主导设计与多轮重构。

### 2. 权限 / 审批（permissions）
- [#4151](https://github.com/QwenLM/qwen-code/pull/4151)（5月，+5204/-61，当月最大）：新增 **Auto approval 模式 + LLM classifier**。
- [#2283](https://github.com/QwenLM/qwen-code/pull/2283)（3月，+10557/-1285）：`Feat: support permission`——权限体系奠基性大 PR。
- [#2637](https://github.com/QwenLM/qwen-code/pull/2637)：人类可读权限标签、deny 规则反馈、多目录搜索。
- [#4386](https://github.com/QwenLM/qwen-code/pull/4386)：command substitution 改为 ask 而非 deny。

### 3. Worktree（多分支隔离）—— Phase A→D 全程
- [#4073](https://github.com/QwenLM/qwen-code/pull/4073)（5月，+3248）：通用 worktree 支持，`EnterWorktree/ExitWorktree` + Agent 隔离。
- [#4174](https://github.com/QwenLM/qwen-code/pull/4174)（5月，+4718）：Phase C——会话持久化、hooksPath、Footer + WorktreeExitDialog、三模式 `--resume`。
- [#4381](https://github.com/QwenLM/qwen-code/pull/4381)（5月，+3719）：Phase D——启动 `--worktree` flag + symlinkDirectories + PR refs。
- （推断）worktree 整条特性线由其端到端实现。

### 4. Auto-memory / auto-dream / auto-skill
- [#3087](https://github.com/QwenLM/qwen-code/pull/3087)（4月，+9810/-2738）：托管式 **auto-memory + auto-dream** 系统（单 PR 体量最大）。
- [#3673](https://github.com/QwenLM/qwen-code/pull/3673)：autoSkill 后台项目技能抽取。
- [#4547](https://github.com/QwenLM/qwen-code/pull/4547)（5月）：auto-dream / auto-skill **默认开启** + `/memory` 开关。
- [#4489](https://github.com/QwenLM/qwen-code/pull/4489) / [#4394](https://github.com/QwenLM/qwen-code/pull/4394)：防止 auto-skill 覆盖已有 skill；加载 `QWEN.local.md` 项目本地上下文。

### 5. Computer-use（零配置）
- [#4590](https://github.com/QwenLM/qwen-code/pull/4590)（5月，+4869/-1）：通过 open-computer-use MCP 的**零配置内置 computer-use**。

### 6. 扩展 / MCP / Slash 命令 / 平台兼容（长尾但密集）
- 扩展系统：[#1534](https://github.com/QwenLM/qwen-code/pull/1534)（`Feat/extension`，+14290/-6837，最大单 PR）、[#2008](https://github.com/QwenLM/qwen-code/pull/2008) 扩展管理 TUI、[#1831](https://github.com/QwenLM/qwen-code/pull/1831) MCP 管理 TUI、Claude marketplace 插件选择 UI（[#1592](https://github.com/QwenLM/qwen-code/pull/1592)）。
- Slash 命令：能力化过滤替换白名单（[#3283](https://github.com/QwenLM/qwen-code/pull/3283) Phase 1 / [#3377](https://github.com/QwenLM/qwen-code/pull/3377) Phase 2）+ 命令发现改进（[#3736](https://github.com/QwenLM/qwen-code/pull/3736)）。
- ACP：[#2690](https://github.com/QwenLM/qwen-code/pull/2690) 跨客户端权限流对齐、[#2344](https://github.com/QwenLM/qwen-code/pull/2344) FileSystemService 接口 ACP 对齐、SSE/HTTP MCP（[#3574](https://github.com/QwenLM/qwen-code/pull/3574)）。
- Windows / 编码兼容：CRLF/LF 保留（[#2707](https://github.com/QwenLM/qwen-code/pull/2707)）、非 UTF-8 编码保留（[#2073](https://github.com/QwenLM/qwen-code/pull/2073)）、Windows 路径/快捷键多处修复、Kitty 键盘协议（[#3544](https://github.com/QwenLM/qwen-code/pull/3544)）。
- 还参与版本 bump（[#3541](https://github.com/QwenLM/qwen-code/pull/3541)）、auth CLI 命令（[#2440](https://github.com/QwenLM/qwen-code/pull/2440)）等——（推断）有发布 / 维护权限。

**主题画像（推断）**：横跨「模型/上下文核心层（compaction、auto-memory）」+「Agent 能力层（permission、worktree、computer-use）」+「平台兼容（Windows、编码、ACP）」三层，是少见的纵贯式核心贡献者，而非单一模块专家。

## 跨仓库社区足迹

跨仓库 merged PR 搜索（公开）：

| 仓库 | merged PR | 说明 |
|---|--:|---|
| QwenLM/qwen-code | 92 | 主战场（见上）|
| CS-Tao/GTD-Visualization | 9 | 全球恐怖袭击数据可视化系统（高校协作项目）|
| LaZzyMan/Flow-VIS | 6 | 自有项目：大规模轨迹数据前端可视化系统 |
| LaZzyMan/terrorsim_rear_end | 3 | 自有：恐怖袭击可视化后台（Python）|
| LaZzyMan/lib-whu | 2 | 自有：武大相关（Vue）|

公开仓库 40 个，主线偏 **GIS / 地理空间可视化 + 轨迹数据**（Cesium、QGIS-ISODATA 插件、pywebplot、TaxiDataProcessing、GPS 机器学习），并有多个 iOS/Swift 个人 app（EatWhich、TraceSecret 等）。fork 列表含 `qwen-code` / `claude-code` / `OpenSpec` / Zed `extensions`——（推断）个人技术兴趣从早期 GIS/数据可视化转向 AI coding agent 生态，与武大背景一致。

## 备注

- **真名公开**：GitHub profile `name` 字段直接显示「顾盼」，`company` 显示 Wuhan University，均为公开信息，非推断。
- **22 vs 92 的口径**：「22」是 **2026 年 5 月当月** merged 数（[月活榜](../2026-05.md) 口径）；「92」是 2026-01 至 2026-05 在 qwen-code 的**累计** merged 数。两者不可混用。
- 该贡献者在月活榜的「主要方向」被归纳为 *compaction 二次重构（BREAKING）+ computer-use 零配置 + worktree Phase D + auto-dream 默认开*，与本档案逐 PR 核对一致。
- 是否为 QwenLM 正式员工 / 官方 maintainer：**未公开**（profile 无明确雇主声明，company 仅填高校）；其涉及版本 bump 与 auth/发布相关 PR，提示具备较高仓库权限（推断）。

---

> **免责声明**：本档案数据截至 **2026-05-31**，来自 GitHub 公开 API（`users/LaZzyMan`、merged PR 列表、`gh search prs`）。PR 数与代码增删为 GitHub 报告值；含部分大型重构 PR，行数不完全等于净产出。真名、单位、邮箱、个人站为 GitHub profile 公开字段。标注「（推断）」的为基于公开信息的合理推测，非官方确认；标注「未公开」的项 GitHub 公开资料未提供。
