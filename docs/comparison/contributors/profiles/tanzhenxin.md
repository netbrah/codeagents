# tanzhenxin

> Qwen Code（QwenLM/qwen-code）核心层（core）方向主力贡献者之一。本档案仅依据 GitHub 公开数据，未公开信息一律标注「未公开」，主观判断标注「（推断）」。

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@tanzhenxin](https://github.com/tanzhenxin)（id 480123，账号注册 2010-11-13） |
| 真名 | 未公开（GitHub `name` / `bio` / `company` / `location` 均为空） |
| 内部花名 | 愚远 |
| 2026-05 当月 | **21 merged PR / +13k/-6k**，在 Qwen Code 月活榜并列第 6 |
| 累计（QwenLM/qwen-code，merged） | **256 PR**（自 2025-07 起，全量 merged） |
| 首次 PR | **2025-07-30**（#142「Fix Default Model Configuration and Fallback Behavior」+ #141 OpenAI usage logging，同日并入） |
| 跨仓库足迹 | qwen-code 256 + GraphScope/portal 3 + agent-client-protocol 1 + Homebrew/homebrew-core 1（均 merged） |
| 公司/组织 | 内部 Qwen 团队（阿里生态，maintainer 2026-05-31 确认） |
| 角色 | Qwen Code 内部团队成员（maintainer 2026-05-31 确认），core/runtime 长期核心贡献者，专注模型层适配（OpenAI/Anthropic/DeepSeek/DashScope）、subagent 系统、上下文/compaction、shell/PTY 与 ACP 集成 |

## 在 Qwen Code 的工作

### 当月（2026-05）方向归纳

5 月 21 个 merged PR 高度集中在 **core/runtime 与 subagent 体系**（标题人工归纳，非完整列表）：

- **Subagent 体系深化**：`feat(cli): route foreground subagents through pill+dialog while running`（#3768）、`refactor(core): TaskBase envelope + foreground subagent persistence`（#3970）、`fix(core): auto-compact subagent context to prevent overflow`（#3735）、`fix(core): per-agent ContentGenerator view via AsyncLocalStorage`（#3707）、`feat(subagents): use fastModel for Explore subagent`（#4086）、`fix(cli): show tool details in subagent approval banner`（#3956）。
- **跨认证 / 快速模型（cross-auth fast models）**：`fix(core): support cross-auth fast side queries`（#4117）、`feat(core): extend cross-auth fast models to agents`（#4153）、`refactor(core): route side-query LLM calls through runSideQuery chokepoint`（#3775，把所有侧查询 LLM 调用收敛到单一入口）。
- **模型 Provider 适配与请求可观测性**：`fix(anthropic): allow cache_control on tool_result blocks`（#4121）、`fix(core): correct context-usage Footer for prompt size and Anthropic caches`（#4109）、`fix(core): log the OpenAI request actually sent on the wire`（#3767）、`fix(core): log internal OpenAI JSON requests`（#4081）、`fix(core): tag subagent OpenAI JSON logs`（#4099）。
- **配置 / 上下文 / 工具 UX**：`feat(core): support QWEN_HOME env var to customize config directory`（#2953）、`fix(core): preserve read-before-write state across idle microcompaction`（#4243）、`feat(tools): keep ask_user_question always-visible to surface clarification UX`（#4041）。
- **测试与工程维护**：`test(integration): pin simple-mcp-server to legacy MCP path`（#4164）、`test(perf): skip daemon baseline harness under sandbox`（#4234）、`fix(test): repair stale --json-schema integration assertion`（#4075）、`refactor(cli): revert dynamic slash command LLM translation`（#4145）。

一句话概括：当月主要做 **subagent 运行时（前台 pill/dialog、持久化、上下文 auto-compact、per-agent ContentGenerator）+ 跨认证 fast-model 侧查询通路 + OpenAI/Anthropic provider 请求日志与缓存修复**。

### 累计代表作（2025-07 至今，按主题归纳）

tanzhenxin 是 Qwen Code 从 Gemini CLI 分叉早期就深度参与的核心贡献者，长期工作主线（推断）覆盖：

- **去 Google 化 / Provider 适配**：早期 `Migrate web search from Google/Gemini to Tavily API`（#329）、`refactor web-fetch tool to remove google genai dependency`（#340）；后续 `Add Anthropic provider`（#1331）、`Add Gemini provider, remove legacy Google OAuth`（#1297）、`DeepSeek V3.2 Thinking Mode Integration`（#1134）、多模态输入支持（image/PDF/audio，#1564）。
- **Subagent 系统奠基**：`🎯 Subagents Feature`（#573）、Plan Mode（#658）、`Add Explore agent and rename TaskTool to AgentTool`（#2489）、跨 provider 子代理模型选择（#2698）、background subagents（#3076/#3379）、`model-facing agent control（task_stop/send_message）`（#3471）。
- **Shell / PTY / 跨平台**：`enable PTY by default`（#2108）、多个 Windows ConPTY / 编码 / 引号修复（#2347/#2349/#2389/#2423 等）、`Replace spawn with execFile for memory-safe command execution`（#1068）。
- **上下文与可靠性**：`microcompaction for idle context cleanup`（#3006）、Edit 工具 fuzzy matching（#1025）、`security: prevent command injection via newline bypass`（#1638）。
- **ACP（Agent Client Protocol）集成**：JetBrains `session/set_model`（#1521）、ENOENT 错误码集中化（#1550）、subagent text/reasoning 流式（#1626）等多个 `fix(acp)` PR。
- **发布工程**：长期承担 version bump / release / e2e workflow / upstream gemini-cli 同步（多次 `Sync upstream gemini-cli vX.Y`）。

> 月度产出节奏（merged PR）：2025-07 起步 2，2025-08 起稳定在 12–32/月，2026-04 达峰值 **62**，2026-05 为 21。

## 跨仓库社区足迹

| 仓库 | merged PR | 内容 |
|---|--:|---|
| [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code) | 256 | 核心贡献主战场（见上） |
| [GraphScope/portal](https://github.com/GraphScope/portal) | 3 | 2025-06 前端/基础设施类（`Infra tzx` #769/#764、`use customized playwright-mcp` #770）——「tzx」缩写与 tanzhenxin 一致（推断同一人） |
| [agentclientprotocol/agent-client-protocol](https://github.com/agentclientprotocol/agent-client-protocol)（fork 自 zed-industries 系） | 1 | `Add Qwen Code link to agents page`（#331）——把 Qwen Code 登记进 ACP 官方 agents 列表 |
| [Homebrew/homebrew-core](https://github.com/Homebrew/homebrew-core) | 1 | 单个 formula 贡献 |

公开仓库 5 个（均为 fork：claude-code、claude-code-config、agent-client-protocol、homebrew-core、Qwen-Agent），无个人原创开源项目对外可见；followers 63 / following 2。

## 备注

- **内部团队成员**：项目 maintainer 于 2026-05-31 确认 tanzhenxin 为 Qwen Code 内部团队成员，内部花名「愚远」。其 GraphScope/portal 的早期贡献（2025-06，去 mcp / playwright-mcp 方向）与在 Qwen Code 的 Agent/MCP 工作主题连贯（Qwen / GraphScope 均为阿里系开源项目）。**真名、职位、地理位置 GitHub 仍未公开**，花名为额外信息、不替代真名，不做实名断言。
- 任务背景给出的「21 merged PR / +13k/-6k」对应的是 **2026-05 当月**口径；其在 qwen-code 的**累计** merged PR 为 256，二者不可混淆。
- 本档案聚焦其在 Qwen Code 的工作，方向归类（subagent / provider 适配 / shell-PTY / ACP）为基于 PR 标题的人工归纳。

---

> **免责声明**：本档案数据截至 **2026-05-31**，全部来自 GitHub 公开 API（用户资料、merged PR 列表、跨仓库 PR 搜索、repos 列表）。PR 数量与代码增删量为 GitHub 报告值；「主要方向」「代表作」按 PR 标题人工归纳，非完整清单。真名、雇主等个人信息 GitHub 未公开，凡推断均已标注「（推断）」，未作实名披露。
