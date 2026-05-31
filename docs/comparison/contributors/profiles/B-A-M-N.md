# B-A-M-N

> Qwen Code 贡献者深度档案。本档案仅基于公开的 GitHub 资料（用户 profile、PR 列表、仓库元数据）编写，未涉及任何非公开信息。

## 速览表

| 项目 | 内容 |
|------|------|
| GitHub | [@B-A-M-N](https://github.com/B-A-M-N)（user id `180903032`，2024-09 注册） |
| 显示名 | John London（GitHub `name` 字段，疑似笔名/化名，**未公开**真实身份） |
| 个人简介 | "AI Systems Architect · Local-first · Deterministic-by-design. Open to contracts, consulting, and full-time roles." |
| 主页 | https://bamnlanding.lovable.app |
| 公开邮箱 | （邮箱略）（profile 公开字段） |
| 公开仓库 | 49 个（Python 主导：29 Python / 3 JS / 3 Go / 3 TS / 1 C#） |
| Qwen Code 2026-05 | 7 个 merged PR，约 **+2,586 行**（净增） |
| Qwen Code 累计 | 13 个 merged PR，约 **+4,535 行**（截至 2026-05-31） |
| 首次合入 PR | 2026-04-12（[#3123](https://github.com/QwenLM/qwen-code/pull/3123) `fix(core): handle empty OAuth refresh response body`） |
| 当前 open PR | 3 个（#3973/#3974/#3975，MCP 持久化、本地模型重试、`/directory remove`） |
| 角色推断 | 配置/认证与本地模型集成方向的稳定外部贡献者，非维护者（推断） |

## 在 Qwen Code 的工作

B-A-M-N 在不到两个月内（2026-04-12 起）密集贡献，13 个 merged PR 高度聚焦于
**配置解析、认证/凭证流、本地模型适配与 CLI 健壮性**，几乎全部为 `fix`/`refactor`
（仅 2 个 `feat`），呈现典型的"打磨边缘 case、统一既有逻辑"的工程风格，与其
self-bio 中 "Local-first · Deterministic-by-design" 的定位一致。

按方向归纳全部 13 个 merged PR：

**1. 模型/配置优先级与环境变量（最大主题）**
- #3567 `fix(cli): respect OPENAI_MODEL precedence in CLI model resolution`
- #3645 `fix(cli): correct model precedence — argv > settings > auth env vars`（+553）
- #3815 `fix(core): use per-model settings for fast model side queries`（+544）
- #3629 `fix(config): support QWEN_CODE_API_TIMEOUT_MS across OAuth and non-OAuth paths`
- #3653 `refactor(config): dedupe QWEN_CODE_API_TIMEOUT_MS env override logic`

**2. 认证 / OAuth 流健壮性**
- #3123 `fix(core): handle empty OAuth refresh response body`（首个 PR）

**3. 自动记忆（auto-memory）非阻塞化**
- #3814 `fix(core): prevent auto-memory recall from blocking main request`
- #3866 `fix(memory): address code review feedback for auto-memory recall`

**4. 设置迁移 / settings.json 安全写回**
- #3861 `fix(cli): preserve comments and formatting in settings.json during migration write-back`

**5. 工具执行权限统一（feat）**
- #3723 `feat(core): add shared permission flow for tool execution unification`（+461）

**6. 成本统计（feat）**
- #3780 `Feat/stats model cost estimation rebase`（+819，单 PR 最大）

**7. 搜索/输入交互与跨平台**
- #3981 `fix(search): make empty-query exit synchronous and normalize Windows Backspace`

**8. 工程卫生**
- #3782 `fix(vscode-companion): align package eslint config with root and style cleanup`

**当月（2026-05）做了什么**：7 个 merged PR 集中在两条线——(a) 把
auto-memory recall 从阻塞主请求改为非阻塞并跟进 review 反馈（#3814/#3866）；
(b) 围绕 per-model 设置、成本估算、settings 迁移写回与跨平台输入修复（#3815/
#3780/#3861/#3981）。代表作为 **#3780 成本估算（+819）** 与 **#3815 per-model
快速侧查询设置（+544）**。三个尚未合入的 open PR（MCP 头部/删除持久化、本地模型
"model-unloaded" 重试、`/directory remove`）进一步坐实其 **本地模型 + 配置可靠性**
关注点（推断）。

## 跨仓库社区足迹

公开 PR 搜索显示 B-A-M-N 的开源活动以 Qwen Code 为绝对重心，同时向多个 LLM/agent
生态项目提交过 PR：

| 仓库 | merged | 备注 |
|------|--------|------|
| QwenLM/qwen-code | 13 | 主战场（另有 34 条 PR 记录含 open/closed） |
| B-A-M-N/prforge | 3 | 自有项目 |
| josstei/maestro-orchestrate | 2 | 外部 agent 编排项目 |
| B-A-M-N/OpenDeepWikiX | 1 | 自有 fork |
| B-A-M-N/get-stuff-done | 1 | 自有 fork（"Get-shit-done" 分叉） |

未合入但提交过 PR 的外部仓库还包括 `mistralai/mistral-vibe`（6 条）、`ollama/ollama`
（2 条）、`firecrawl/firecrawl`、`termly-dev/termly-cli` 等，方向一致围绕本地推理与
agent 工具链（推断）。

**自有仓库（49 个，Python 主导）** 集中体现其 "local-first / 分布式 Ollama 编排"
专长，star 较高的代表作：

- **Sheppard**（9★，Python）— 面向 Ollama 的 AI agent，含记忆/自动化
- **FlockParser**（4★，Python）— 分布式文档 RAG，带 GPU/CPU 智能调度
- **SOLLOL**（4★，Python）— "Super Ollama Load Balancer"，性能感知路由
- **SynapticLlamas**（3★）/ **hydra** / **HydraContext** / **Amnesic** / **Archivist**
  — 多 agent 编排、确定性上下文分段/去重/缓存、上下文卸载协议等

这些自有项目的主题（确定性上下文管理、本地模型负载均衡、记忆/上下文卸载）与其在
Qwen Code 中贡献的方向（per-model 设置、auto-memory 非阻塞、本地模型重试）形成
清晰呼应。

## 备注

- 显示名 "John London" 与简介中的 "AI Systems Architect"、邮箱别名
  "benevolentjoker"、自托管 landing page 综合判断，B-A-M-N 更像是独立开发者/
  顾问的个人品牌账号，公开身份信息以化名呈现，**真实姓名未公开**。
- profile 标注 `hireable: true`，明确表示 "Open to contracts, consulting, and
  full-time roles"，可解释其在多个上游项目高频提交 fix 类 PR 的动机（建立可见度，推断）。
- 在 Qwen Code 中尚无合并代码以外的维护者权限信号（无 review/triage 公开痕迹），
  归类为**高产外部贡献者**而非项目维护者（推断）。

## 免责声明

> 本档案基于 2026-05-31 前的公开 GitHub 数据（用户 profile、merged/open PR 列表、
> 跨仓库搜索与仓库元数据）编写。PR 数量、行数与方向归纳截至 **2026-05-31**，此后
> 可能变化。所有身份与角色判断已标注"（推断）"，无法核实的信息标注"未公开"。
> 未使用任何非公开渠道信息。
