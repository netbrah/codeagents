# DragonnZhang

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@DragonnZhang](https://github.com/DragonnZhang) |
| 真名 / 显示名 | Dragon（GitHub profile `name` 字段） |
| 公司 / 组织 | @QwenLM（profile `company` 字段，即 Qwen 团队） |
| 地区 | 未公开（profile `location` 为空） |
| 注册时间 | 2019-07-06（账号 ID 52599892） |
| 2026-05 qwen-code PR | 11（+6,464 / -230，对应任务给出的「约 +6k/-0.2k」） |
| qwen-code 累计 merged PR | 29（+21,321 / -668，COLLABORATOR 身份） |
| 首次 qwen-code PR | 2025-12-05 提交 / 2025-12-11 merged · #1156（`feat: update references from Gemini to Qwen in setup commands and gitignore handling`） |
| 跨仓库公开 PR 总计 | 47（其中 45 落在 QwenLM/qwen-code，其余为 qwen-code-action / qwen-code-examples 等周边及零星外部 fork） |
| 角色推断 | Qwen Code 团队 collaborator，SDK / VS Code 扩展 / CLI 交互方向核心贡献者（推断） |

> 说明：任务简报中的「11 merged PR / 约 +6k/-0.2k」实为 **2026-05 当月窗口** 数据；该贡献者 **累计** merged PR 为 29，体量 +21k/-0.7k，自 2025-12 起即持续贡献。

## 在 Qwen Code 的工作

- 主要方向：
  - **SDK 健壮性**：`canUseTool` 超时在 CLI control request 中的正确传递（#4491）、把 CLI chunks 一并打进 SDK 包修复发布缺失（#4541）、SDK + CLI 的 system prompt 自定义选项（#2400）、`query` 支持以 session ID 恢复会话（#1714）。
  - **VS Code 扩展 / IDE 集成**：message edit/rewind 与 message metadata UI（#3762，+2,829）、保留 thinking 状态并恢复丢失的 edit 快照（#4147）、VS Code settings 的 JSON Schema 校验（#1830）、开发期从源码运行 ACP CLI（#4283）、dev launch config（#1784）。
  - **CLI 交互体验**：内置 status line 预设 + 交互式选择对话框（#4120，+1,654）、slash command 的 argument-hint 支持（#3593）、可打印 CSI-u 按键支持（#1827）、ESC 取消后解除输入阻塞（#1796）、plan 模式下保留被拒绝的 plan 内容（#2157）。
  - **core / 模型 provider 兼容**：Token Plan cache control（#4495）、MiMo tool-result media 处理（#4281）、surface Anthropic 空流的 provider 错误（#4540）。
  - **insight / 用户反馈方向**：个性化编程 insight command（#1593，+7,706，单 PR 体量最大）、用户反馈对话框及其临时关闭/持久提示逻辑（#1465 / #1590）、skill 的 docs audit/update helper（#2397）、skill manager 的 symlink 支持（#1690）、monitor tool 文档（#4356）。
- 代表作 PR：
  - [#1593 add insight command for personalized programming insights](https://github.com/QwenLM/qwen-code/pull/1593) — +7,706/-206，累计体量最大的单 PR，开辟个性化编程 insight 能力。
  - [#3762 add message edit/rewind and message metadata UI](https://github.com/QwenLM/qwen-code/pull/3762) — +2,829，VS Code 扩展的消息编辑/回退与元数据 UI 主干。
  - [#4120 add built-in status line presets with interactive dialog](https://github.com/QwenLM/qwen-code/pull/4120) — +1,654，CLI 内置状态栏预设 + 交互式配置对话框。
  - [#4491 honor canUseTool timeout in CLI control requests](https://github.com/QwenLM/qwen-code/pull/4491) — SDK control 协议的超时正确传递，任务点名的 SDK 方向代表作。
  - [#4281 handle MiMo tool-result media](https://github.com/QwenLM/qwen-code/pull/4281) — MiMo 模型的 tool-result media 兼容，对应任务提到的「MiMo tool-result media」。
- 技术专长（推断）：TypeScript / Node 全栈工程化、VS Code 扩展开发、CLI/TUI 交互（按键处理、对话框、status line）、SDK/协议层（ACP、control request、session 恢复）、模型 provider 兼容适配。自有仓库大量围绕前端（Vue/Nuxt）与 AI Web/GUI 测试 Agent（auto-inspector、midscene、GUI-testing-agent-benchmark），说明在前端 + AI Agent 应用层有较强积累（推断）。

## 跨仓库 / 社区足迹

- 跨仓库 PR 分布（公开 search 计数）：QwenLM/qwen-code 45、QwenLM/qwen-code-action 5、QwenLM/qwen-code-examples 3，以及个人项目（juejin-like-blog-website、savings-simulator、qwen-code-action fork）与零星外部仓库各 1：`youzan/vant`、`vuejs/core`、`microsoft/rushstack-websites`、`vkola-lab/brain2020`、`agentlabs-dev/auto-inspector`、`mrgoonie/claudekit-skills` 等——主体高度集中于 Qwen Code 官方周边生态（推断为团队内部贡献者）。
- 自有项目：33 个公开仓库，方向偏前端框架 fork（Vue `core`/`vant`、OHIF `Viewers`）与 AI/Agent 应用原创（`auto-inspector` Web 测试 AI Agent、`GUI-testing-agent-benchmark`、`midscene`、`TrendRadar` 舆情监控 MCP 工具、`antigravity-claude-proxy`、`qwen-code-action`、`qwen-code-examples`、`craft-agents-oss`、`v0-like-ui-designer`/`rapidpages`/`openui` 等多个 AI 生成 UI 项目），star 普遍较低（多为 0–6）。另有学术/课程类（`brain2020` 阿尔茨海默深度学习、Stanford CS131 笔记），推断有计算机视觉/科研背景（推断）。
- 个人主页 / 博客 / 社交：profile `blog`/`twitter_username`/`email`/`location` 字段均为空，未公开；存在个人站点仓库 `DragonnZhang.github.io`（HTML）。
- 背景推断：profile `company` 标注 `@QwenLM` 且在 qwen-code 中为 COLLABORATOR，应为 Qwen 团队成员，专注 SDK / VS Code 扩展 / CLI 交互与模型兼容方向；个人技术兴趣偏 AI Agent 与前端 UI 生成（推断，依据 company 字段 + COLLABORATOR 身份 + 个人仓库主题）。

## 备注 / 不确定项

- 真名仅取自 GitHub profile 显示名「Dragon」，未做进一步身份关联；中文真名 **未公开**。
- 任务简报「11 merged PR / +6k/-0.2k」对应 2026-05 当月；累计为 29 PR / +21k/-0.7k，二者不冲突，本档案以累计为主、当月为辅。
- PR #2138（`chore: add @DragonnZhang to CODEOWNERS`，2026-03 merged）曾将其加入 CODEOWNERS，但 **当前** `.github/CODEOWNERS` 文件中已查不到该 entry（可能后续重构移除），故 owner 范围以「曾被列为 code owner」记录，当前归属 **未公开 / 未确认**。
- followers 24 / following 86，社区影响力指标偏低，符合「团队内部工程师」而非「社区网红」画像（推断）。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份/背景推断均已标注「（推断）」，未公开信息标注「未公开」，不含任何私人信息。
