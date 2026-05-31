# shenyankm

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@shenyankm](https://github.com/shenyankm) |
| 真名 / 显示名 | Yan Shen（GitHub profile `name` 字段） |
| 公司 / 组织 | 未公开（profile `company` 字段为空） |
| 地区 | NanJing（南京，profile `location` 字段） |
| 邮箱 | （邮箱略）（profile 公开 `email` 字段） |
| 注册时间 | 2024-05-30（账号 ID 171212926，较新账号） |
| 2026-05 qwen-code PR | 5（+10,030/-4,532，约 +10k/-5k） |
| qwen-code 累计 merged PR | 8（首 PR #3270 起，全部集中在本仓库） |
| 首次 qwen-code PR | 2026-04-15 merged / #3270（`fix(cli): ignore literal Tab input in BaseTextInput`，创建于 2026-04-14） |
| 角色推断 | CLI / TUI（Ink 终端 UI）层 UX 修补型外部贡献者（author_association = CONTRIBUTOR）（推断） |

## 在 Qwen Code 的工作

方向归纳（依据 8 个 merged PR 标题 + 体量分布自行归纳）：shenyankm 的贡献几乎全部落在 **`cli` 包的终端 UI / 交互体验层**，少量触及 OpenAI 兼容 provider 的解析。可分为三条线：

- **TUI 组件与布局（Ink 渲染层）主线**：sticky todo 面板（新增并随后压缩其紧凑度）、`BaseTextInput` 的输入处理修复（忽略字面 Tab 输入）、statusline preset 排序稳定化、导出格式补全的键盘导航。这些是面向交互细节的 UX 打磨，单 PR 体量从几十行到一两千行不等。
- **国际化 / i18n 主线（最大体量）**：单 PR #3871 `core built-in i18n coverage` 一次性 +6,253/-4,423，是其当月体量的主要来源；另有早前被关闭的 #3328（本地化 slash command 描述 + 内置 UI 语种覆盖）说明其在 i18n 方向上有持续投入，是把 CLI 内置文案体系化做多语言覆盖的推动者（推断）。
- **OpenAI 兼容 provider 解析（少量跨界）**：#3677 解析 MiniMax 的 thinking 标签，触及 `openai` provider 的响应解析路径，是其唯一一条偏后端/协议解析的贡献。

代表作 PR：

- [#3871 core built-in i18n coverage](https://github.com/QwenLM/qwen-code/pull/3871) — +6,253/-4,423，当月最大体量，系统化覆盖内置 UI 的国际化文案，是其 i18n 方向的主干代表作。
- [#3507 add sticky todo panel to app layouts](https://github.com/QwenLM/qwen-code/pull/3507) — 在应用布局中引入常驻 todo 面板的 TUI 新能力（随后 #3647 又补上紧凑化），是其 TUI 组件线的代表。
- [#4062 add configurable plansDirectory for Plan Mode](https://github.com/QwenLM/qwen-code/pull/4062) — 为 Plan Mode 增加可配置的 plans 目录，触及配置与 Plan Mode 工作流。
- [#3677 parse MiniMax thinking tags](https://github.com/QwenLM/qwen-code/pull/3677) — OpenAI 兼容层解析 MiniMax 的 thinking 标签，唯一一条 provider 解析向贡献。
- [#3701 improve export format completion navigation](https://github.com/QwenLM/qwen-code/pull/3701) — 改进导出格式补全的键盘导航体验。

技术专长（推断）：Node / TypeScript + Ink 终端 UI 组件开发，i18n 文案体系化，键盘 / 输入交互细节，以及 OpenAI 兼容 provider 的响应解析；整体画像偏“前端 / TUI 交互层工程师”而非核心后端 / daemon 方向（推断）。

## 跨仓库 / 社区足迹

- 跨仓库 PR 量：公开 PR（含开/闭）共 12 个，**全部集中在 QwenLM/qwen-code**，无其他外部仓库 PR 足迹。
- PR 状态分布：8 merged、2 closed-unmerged（#3328 i18n、#3668 `feat(stats): add current session billing estimates`）、2 open（#4564 `feat(stats): expose token usage for cost visibility`、#4653 `feat(core): respect configurable agent ignore files`）。其中 stats / 计费可见性方向尝试过两次（#3668 被关闭、#4564 仍 open），说明在该方向有持续探索但尚未落地（推断）。
- author_association = **CONTRIBUTOR**（非 COLLABORATOR），结合较新的账号（2024-05 注册）与较低的社区指标（followers 2 / following 1、仅 2 个公开仓库），画像更接近 **外部社区贡献者** 而非项目内部 / 官方团队成员（推断）。
- 自有项目：2 个公开仓库 —— `qwen-code`（fork，用于提 PR）、`zhihu-lks`（TypeScript，无描述、0 star），无可据以判断背景的实质内容。
- 个人主页 / 博客 / 社交：未公开（profile `blog`/`twitter_username`/`bio` 字段均为空；`email` 公开为 （邮箱略））。

## 备注 / 不确定项

- 真名仅取自 GitHub profile 显示名 “Yan Shen”，未做进一步身份关联。
- 任务给出「2026-05 共 5 merged PR，约 +10k/-5k」与实测一致（5 PR / +10,030/-4,532）；注意 #4634 创建于 2026-05-30 但 merge 于 2026-05-31，本档案按 mergedAt 归入 5 月统计，故 5 月 merged 计为 5（#4634/#4062/#3871/#3701/#3677）。累计 8 含 4 月的 #3270/#3507/#3647。
- 当月体量高度集中：单个 i18n PR #3871 即贡献了 5 月 +10k 行中的约 6.2k，其余 4 个 PR 体量均较小，所谓“单 PR 体量较大”主要由这一个 i18n PR 拉动。
- 角色为“修补 / UX 打磨型外部贡献者”属基于 PR 方向 + author_association 的推断，非官方身份确认。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份 / 背景推断已标注“（推断）”，未公开信息标注“未公开”，不含任何私人信息。
