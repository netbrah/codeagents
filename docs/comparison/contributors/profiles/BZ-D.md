# BZ-D

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@BZ-D](https://github.com/BZ-D) |
| 真名 / 显示名 | Edenman（GitHub profile `name` 字段） |
| 内部花名 | 克竟 |
| 公司 / 组织 | 内部 Qwen 团队（阿里生态；maintainer 2026-05-31 确认为内部团队成员）。GitHub 公开资料：Alibaba Cloud @Hangzhou（阿里云，杭州；profile `location` 字段）、南京大学在读研究生（profile `company`=Nanjing University） |
| 专业方向 | 软件工程 / AI Agent Engineering（profile `bio`） |
| 地区 | Hangzhou（杭州） |
| 注册时间 | 2020-06-28（账号 ID 67549719） |
| 2026-05 qwen-code PR | 当月 5 个 merged（#3491/#3916/#4037/#4277/#4502，约 +7.6k 行）；任务给定的「+9k」量级与实际相符 |
| qwen-code 累计 merged PR | 11（+10,322 / -316），另有 4 个 CLOSED 未合（共发起 15 个） |
| 首次 qwen-code PR | 2026-04-09 发起 / 2026-04-10 合入 · #3060（`fix: prevent Shift+Tab from accepting prompt placeholder suggestion`） |
| 角色 | 内部 Qwen 团队成员（maintainer 2026-05-31 确认）。CLI / 终端交互体验方向贡献者：headless runaway-protection guardrails + `/diff` 交互式 diff 体系 + 终端 OSC 转义（超链接/主题/复制）专项 |

## 在 Qwen Code 的工作

- 主要方向（按 PR 聚类）：
  - **`/diff` 交互式 diff 体系（代表作主线）**：从 #3491 落地 `/diff` 命令 + git diff 统计工具（+3,513 行，单 PR 最大），到 #4277 升级为「per-turn /diff with interactive dialog」（每轮对话级别的交互式 diff 对话框，+2,192 行）。这是其在 qwen-code 最完整的一条功能演进线。
  - **headless / 非交互模式 runaway-protection guardrails**：#4502（+1,093 行，引用 #4103）为 headless / 非交互运行加防失控护栏（runaway protection）。注：同名的 #4105 / #4103 早期实现为 CLOSED，最终以 #4502 合入，说明该特性经过多轮迭代收敛（推断）。
  - **终端 OSC 转义序列体验专项**：OSC 8 包裹 markdown 链接使换行后的 URL 仍可点击（#4037）；MCP OAuth 授权 URL 换行后可点击（#3489）；OAuth URL 的 OSC 52 复制热键（#3393）；OSC 11 / `auto` 终端明暗主题自动探测（#3460，早期 #3082 为 CLOSED 版本）。这是一组对终端转义序列细节高度敏感的 UX 打磨工作。
  - **MCP 健壮性 / 初始化收尾**：从健康状态注册表中剔除已禁用的 MCP server（#3916）、MCP 初始化完成后清理残留空行（#3509）、`mcp add` 增加 OAuth 配置 flag（#3442）。
- 代表作 PR：
  - [#3491 add /diff command and git diff statistics utility](https://github.com/QwenLM/qwen-code/pull/3491) — 单 PR +3,513 行，奠定 `/diff` 能力地基。
  - [#4277 per-turn /diff with interactive dialog](https://github.com/QwenLM/qwen-code/pull/4277) — 把 `/diff` 升级为每轮交互式对话框（+2,192 行），任务标注为其代表作之一。
  - [#4502 headless / non-interactive runaway-protection guardrails (#4103)](https://github.com/QwenLM/qwen-code/pull/4502) — 非交互模式防失控护栏（+1,093 行），任务标注的核心方向。
  - [#4037 wrap markdown links in OSC 8](https://github.com/QwenLM/qwen-code/pull/4037) — 终端超链接 OSC 8 体验（+1,661 行）。
- 未合入但体现兴趣方向的尝试（CLOSED）：#4402「stream-driven tool dispatch — Phases 1–4」（流式工具分发，分 4 个 Phase 的较大改造）、#4084 runtime.json sidecar follow-ups、#4105 runaway-protection 早期版、#3082 主题探测早期版。说明其也涉足 core 层的工具调度链路改造（推断）。
- 技术专长（推断）：CLI / TUI 交互体验、终端转义序列（OSC 8 超链接 / OSC 52 复制 / OSC 11 主题探测）、git diff 呈现、非交互/headless 运行安全护栏、MCP server 生命周期与 OAuth 接入。整体画像偏「终端体验 + CLI 健壮性」工程师，而非后端/分布式方向（推断）。

## 跨仓库 / 社区足迹

- 跨仓库 PR：除 QwenLM/qwen-code（15 个 PR）外，在 [hengyunabc/mcp2cli](https://github.com/hengyunabc/mcp2cli) 有 1 个已合入 PR —— [#2 fix(schema): handle nested array MCP parameters](https://github.com/hengyunabc/mcp2cli/pull/2)（2026-05-21 合入），处理嵌套数组 MCP 参数的 schema，与其在 qwen-code 的 MCP 方向一致。
- 自有项目：28 个公开仓库，多为南京大学软件工程课程 / 学术课题，技术栈横跨 Java、Vue/TypeScript、Python：
  - `casebase-backend`(Java) / `casebase-frontend`(TS) — 学术课题「司法案例外规内化」前后端
  - `COLLECT_FrontEnd`(Vue ★1) — 众包测试分发平台
  - `SimilarCodeRetrieve-System`(Java) — 相似代码检索系统
  - `COVID-19-Data-Science`(Python ★3)、`FAKE_METER-Implementation`(自动化测试课程作业)、`SE2-Documents`(★7) / `SE2-Codes`(★2) 等课程产物
  - 整体 star 数偏低（最高 ★7），属个人学习 / 课程 / 学术性质（推断）。
- 个人主页 / 博客 / 社交：未公开（profile `blog` 为空、`twitter_username` 为空、`email` 为 null）。
- 社区影响力指标：关注者 12 / 关注 7，偏低，符合「在校研究生 + 项目内贡献者」而非社区网红画像（推断）。

## 备注 / 不确定项

- 任务给定「5 merged PR / 约 +9k / 方向 headless runaway-protection (#4502 #4103) + per-turn /diff (#4277)」：方向描述与实际完全吻合；「5 merged」对应的是 **2026-05 当月**合入数（累计为 11）；「+9k」与当月体量（约 +7.6k）量级一致、与累计 +10.3k 同数量级，不冲突。
- 真名仅取自 GitHub profile 显示名「Edenman」，未做进一步身份关联；profile 同时写明南京大学在读 + 阿里云 @Hangzhou，推断为阿里云实习/在职研究生身份（推断）。
- #4103 / #4105 与最终合入的 #4502 为同一特性的多轮迭代；本档案以实际 merged 的 #4502 为准。
- 是否为 QwenLM 官方团队 COLLABORATOR 未在公开 profile 中明确标注；从 PR 全部进入官方仓库且功能成体系推断其为深度参与者（推断），但雇佣/团队归属未公开。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份/背景推断已标注「（推断）」，未公开信息标注「未公开」，不含任何私人信息。
