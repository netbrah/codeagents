# pomelo-nwu

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@pomelo-nwu](https://github.com/pomelo-nwu) |
| 真名 / 显示名 | pomelo（GitHub profile `name` 字段；邮箱 `（邮箱略）`，`nwu` 疑为西北大学 Northwest University 缩写，推断） |
| 内部花名 | 山果 |
| 公司 / 组织 | 内部 Qwen 团队（阿里生态，maintainer 2026-05-31 确认）；GitHub profile `company` 公开字段为 Alipay（蚂蚁 / 支付宝） |
| 地区 | Hangzhou, China（杭州） |
| 注册时间 | 2015-01-26（账号 ID 10703060） |
| 个性签名 | "do one thing and do it well"（Unix 哲学） |
| 2026-05 qwen-code merged PR | 11（约 +5.3k/-6.4k，含 auth 统一重构 #4287 的 +3987/-5304 大改） |
| qwen-code 累计 merged PR | 65（+45,494 / −23,785） |
| 首次 qwen-code PR | 2025-07-29 / #108（`feat: Add systemPromptMappings Configuration Feature`） |
| 角色推断 | 内部 Qwen 团队成员（maintainer 2026-05-31 确认）；auth / provider 子系统 owner + i18n 国际化奠基者 + 文档/PR 流程治理者 |

> 任务简报给出的「11 merged PR / +15k-13k」对应 **2026-05 当月**；本档案补齐 **累计 65 merged PR** 的全景。

## 在 Qwen Code 的工作

- 主要方向：
  - **auth / provider 子系统 owner（推断）**：从 provider-first auth registry（#3864）、统一安装流水线，到 2026-05 的 `Connect a Provider` 大重构（#4287，把 provider 配置统一进 core、简化 `/auth` 对话框），再到移除 legacy `qwen auth` CLI 子命令改走 TUI（#3959），是认证/Provider 接入这条主线的长期负责人。期间陆续接入 OpenRouter（#3576）、Idealab（#3955）等第三方 provider，并产出 custom API key wizard 的 PRD（#3583）与向导步骤指示器 UX（#3607）。
  - **i18n 国际化奠基者**：#1058 引入 UI 与 LLM 输出的国际化框架，#1616 补齐巴西葡萄牙语（pt-BR）并重构 i18n 架构、#1332 修多语言问题。是该项目多语言能力的开创者（推断）。
  - **文档 / PR 流程治理（maintainer 级，推断）**：#1260 按 Claude Code 的组织方式重构整套 docs（+4674/−5369）、#4496 加 Working Principles 并重整 PR 模板、#3522 收紧 PR 模板校验、#3952 给 create-issue 命令加双语要求；同时维护 `QwenLM/qwen-code-docs`（43 merged PR）。这类对 agents 规范、PR 模板、issue 模板的改动通常只由有仓库治理权限者执行。
  - **模型 / Coding Plan 接入与运营**：长期更新 Coding Plan 可用模型（qwen3.5-plus/qwen3.6-plus 默认模型、glm-4.7 / kimi-k2.5 / qwen3-coder-next 等第三方模型 #1907）、调整 Qwen OAuth 免费额度（#1730 由 2000 降至 1000）、发布公告 banner。
  - **/context 命令与可观测性**：#1835 新增 `/context` 命令展示上下文窗口 token 占用拆分，#4512 校准 `/context` 拆分与真实 API 请求一致。
  - **桌面端 multi-brand**：#4581（2026-05）为 desktop 增加多品牌支持与发行构建指南（+664/−190）。
  - **安全与稳定化**：#1601 修复 `READ_ONLY_ROOT_COMMANDS` 中 awk/sed 命令注入漏洞、#4416 稳定 flaky sticky-todo 重测、#3725 清理 legacy Gemini workflows、#588 把 Gemini CLI 品牌引用全量替换为 Qwen Code。
- 代表作 PR：
  - [#4287 unify provider config in core, simplify /auth as "Connect a Provider"](https://github.com/QwenLM/qwen-code/pull/4287) — 2026-05 旗舰重构（+3987/−5304），把认证体验收敛为「连接一个 Provider」。
  - [#3864 provider-first auth registry with unified install pipeline](https://github.com/QwenLM/qwen-code/pull/3864) — provider 注册表与统一安装流水线（+8856/−5174），奠定 auth 架构。
  - [#1058 Add Internationalization Support for UI and LLM Output](https://github.com/QwenLM/qwen-code/pull/1058) — i18n 框架开山之作（+4740/−636）。
  - [#1260 restructure docs to follow the Claude Code organization](https://github.com/QwenLM/qwen-code/pull/1260) — 全量文档重构对齐 Claude Code 组织方式。
  - [#1835 add /context command](https://github.com/QwenLM/qwen-code/pull/1835) — 上下文窗口 token 占用拆分命令。
  - [#4496 add Working Principles and restructure PR template](https://github.com/QwenLM/qwen-code/pull/4496) — agents 规范 / PR 模板治理。
- 技术专长（推断）：认证 / OAuth / Provider 接入与配置统一、i18n 国际化架构、Node/TypeScript CLI 与 TUI 工程化、文档体系与项目流程治理；从个人仓库（`claude-fetch-setup` 的 fetch MCP、`add-skill`、`OpenSandbox`、`craft-agents-oss`）可见对 AI agent 工具链 / MCP / skills 生态有持续投入。

## 跨仓库 / 社区足迹

- 跨仓库 merged PR 分布（公开 search 统计）：
  - `GraphScope/portal` **77**（最早 2024-04，疑为前一段工作主线，图计算可视化前端，推断）
  - `QwenLM/qwen-code` **65**
  - `QwenLM/qwen-code-docs` **43**（2025-08 起，文档站维护）
  - `dashscope/claude-code-config` **7**（claude-code 配置 / npm 发布 / CI workflow）
  - `openclaw/openclaw` 2、`QwenLM/qwen-code-examples` 1、`modelstudioai/.github` 1、`vercel-labs/skills` 1、`Tongyi-WebInfra/gemini-cli` 1、`wenshao/codeagents` 1 等
- 自有项目：54 个公开仓库，含原创 `claude-fetch-setup`（8 star，自动为 Claude Code 配置 fetch MCP）、`add-skill`（TS，open agent skills 工具）、`OpenSandbox`（AI 应用通用沙箱）、`craft-agents-oss`、`qwen-code-channel-skills`，以及 `openclaw`（fork）、`crawl4ai`/`yt-dlp` 等 fork。整体围绕 AI agent / skills / MCP 工具链。
- 影响力指标：followers 207 / following 198（明显高于纯内部工程师画像，社区活跃度中上）。
- 个人主页 / 博客 / 社交：`blog` 字段为空、无 `twitter_username`；公开 email `（邮箱略）`。
- 背景推断：现任职 Alipay（蚂蚁 / 支付宝），常驻杭州；以 collaborator 身份深度参与 Qwen Code 的 auth/provider、i18n 与文档流程治理，承担 maintainer 级职责（推断，依据 profile company + PR 落在 QwenLM 官方仓库且涉及 PR 模板/agents 规范等治理类改动）。`GraphScope/portal` 的 77 个 PR 说明其更早活跃于阿里系图计算可视化（推断）。

## 备注 / 不确定项

- 真名仅取自 GitHub profile 显示名 "pomelo"，未做进一步身份关联；`nwu`/`czynwu` 仅为用户名与邮箱前缀，对应院校（西北大学）为推断。
- profile `company` 为 Alipay，但 qwen-code 属 QwenLM（阿里通义 / 阿里云）官方仓库 —— 同属阿里系，跨 BU 协作或公司字段未及时更新均有可能（推断）。
- 简报中的「+15k/-13k」与本档案统计的当月行数量级一致（单 #4287 即 +3987/−5304，多为重构带来的大规模增删），累计 +45.5k/−23.8k 主要由几笔 auth/i18n/docs 大重构贡献。
- `GraphScope/portal` 的 PR 时间线（2024-04 至 2025-04）早于 qwen-code 首 PR（2025-07），可能是更换团队/项目前的工作（推断）。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份/背景推断已标注"（推断）"，未公开信息标注"未公开"，不含任何私人信息。
