# wenshao

> Qwen Code 当月（2026-05）#2 活跃贡献者，核心 maintainer（推断）。本档案仅基于公开 GitHub 资料整理。

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@wenshao](https://github.com/wenshao)（id `1166785`） |
| 真名 | Shaojin Wen（温绍锦） |
| 内部花名 | 高铁 |
| 公司 | Alibaba（阿里巴巴）— bio 自述「Aliyun DataWorks Tech Leader」；内部 Qwen 团队（maintainer 2026-05-31 确认） |
| 地区 | 中国杭州（Hangzhou, Zhejiang） |
| 注册时间 | 2011-11-02 |
| Followers | ~3,920 |
| 公开仓库 | 141 |
| Bio | 「Aliyun DataWorks Tech Leader, OpenJDK Committer; author of Alibaba's open source projects fastjson/fastjson2/druid」 |
| 个人主页 | [openjdk.org/census#swen](https://openjdk.org/census#swen)（OpenJDK Census） |
| 2026-05 merged PR | **51**（约 +64k/-6k） |
| 累计 merged PR（qwen-code） | **108**（截至 2026-05-31） |
| 首次 merged PR | [#1791](https://github.com/QwenLM/qwen-code/pull/1791) · 2026-02-13 · `feat: add TPM throttling error handling with 1-minute retry delay` |
| author_association | `COLLABORATOR`（GitHub API 标注） |
| 角色 | 内部 Qwen 团队（maintainer 2026-05-31 确认）核心 maintainer / 高频 reviewer |

## 在 Qwen Code 的工作

**主要方向**（按 2026-05 PR 主题归纳）：

1. **`/review` 守门与 review 流水线** —— 把代码评审做成一等公民能力，包括确定性分析、autofix、安全加固、PR 行内评论、9-agent 多代理评审。
2. **TUI / CLI 交互** —— 并行 subagent 的内联面板与键盘导航（InlineParallelAgentsDisplay 系）、status line、`/recap`、`/plan`、`/btw`、`/tasks` 后台任务对话框、follow-up suggestions。
3. **核心 invariant 与正确性修复** —— `tool_use ↔ tool_result` 配对不变量、Edit/WriteFile 的「先读后写」强制、FileReadCache、截断 tool call 的多轮续接等核心可靠性问题。
4. **`qwen serve` daemon 与 SDK** —— daemon Stage 1、`--json-schema` 结构化输出、ToolSearch 按需加载、context-usage API。
5. **provider 兼容性** —— DeepSeek（thinking block 注入、reasoning effort `max` 档）、Anthropic 代理兼容、全局 prompt cache、SSE 限流识别。

**代表作 PR**（带链接）：

| PR | 标题 | 方向 |
|---|---|---|
| [#2348](https://github.com/QwenLM/qwen-code/pull/2348) | feat(skills): add bundled `/review` skill for out-of-the-box code review | review 守门起点 |
| [#2932](https://github.com/QwenLM/qwen-code/pull/2932) | feat(review): enhance `/review` with deterministic analysis, autofix, and security hardening | review 流水线 |
| [#3754](https://github.com/QwenLM/qwen-code/pull/3754) | feat(review): expand review pipeline + `qwen review` CLI subcommands | review CLI |
| [#4477](https://github.com/QwenLM/qwen-code/pull/4477) | feat(cli): dense inline panel + keyboard navigation for parallel agent fan-out | TUI 并行面板 |
| [#4176](https://github.com/QwenLM/qwen-code/pull/4176) | fix(core,cli): close tool_use↔tool_result invariant across all failure paths | 核心 invariant |
| [#3774](https://github.com/QwenLM/qwen-code/pull/3774) | feat(core): enforce prior read before Edit / WriteFile mutates a file | 正确性 |
| [#3889](https://github.com/QwenLM/qwen-code/pull/3889) | feat(cli,sdk): qwen serve daemon (Stage 1) | daemon |
| [#3589](https://github.com/QwenLM/qwen-code/pull/3589) | feat(tools): add ToolSearch for on-demand loading of deferred tool schemas | tools |

**技术专长推断**（基于 PR 主题）：TypeScript/Node.js CLI 与 TUI 工程、LLM agent 运行时正确性（消息不变量、缓存、流式解析）、多 provider 模型兼容、代码评审自动化、性能优化（如 [#3581](https://github.com/QwenLM/qwen-code/pull/3581) 把工具热路径同步 I/O 砍掉 91%）。

## 跨仓库 / 社区足迹

- **跨仓库 PR 总量**：GitHub `search/issues?q=author:wenshao+is:pr` 返回 **~1,520** 个 PR（全平台累计）。
- **组织成员**：`alibaba`、`trinodb`（Trino，公开 org 成员）。
- **自有 / 主导的知名 OSS 项目**（star 数为对应主仓库当前公开数据）：
  - **[alibaba/druid](https://github.com/alibaba/druid)** —— Java 数据库连接池 / 监控，**~28.2k stars**。
  - **[alibaba/fastjson](https://github.com/alibaba/fastjson)** —— Java JSON 库，**~25.6k stars**。
  - **[alibaba/fastjson2](https://github.com/alibaba/fastjson2)** —— fastjson 下一代，**~4.3k stars**。
  - 以上三者均在其 bio 中自述为作者，且 fastjson/druid 是阿里系最知名的 Java 中间件之一。
- **OpenJDK Committer** —— bio 与个人主页（OpenJDK Census）佐证其为 OpenJDK 贡献者。
- **个人 GitHub 仓库**：多为 Java/JDK 相关（如 `fastjson3`、`jdk_internal`、`sql-dialects`、`druid-monitor`），star 较低，主力项目在 `alibaba` org 下。
- **个人主页**：[openjdk.org/census#swen](https://openjdk.org/census#swen)（从 GitHub profile `blog` 字段链出）。

**背景推断**：资深 Java 基础软件工程师出身（fastjson/druid 作者 + OpenJDK Committer + Aliyun DataWorks Tech Leader），2026 年起把工程能力转向 Qwen Code 这个 TypeScript/Node 的 AI coding agent，半年内累计 108 个 merged PR、`COLLABORATOR` 权限、当月排名第 2，**高度符合核心 maintainer / 守门 reviewer 画像（推断）**。

## 备注 / 不确定项

- bio 与公开邮箱（`（邮箱略）`）显示其阿里身份；本档案不引用任何非公开联系方式用途。
- 「核心 maintainer」为基于 `COLLABORATOR` 权限标注 + 高频 review 类 PR + 当月排名的**推断**，未在 README/官方 wiki 单独核实其 maintainer 头衔。
- 「真名 温绍锦」中文名为业界广泛公开认知（fastjson/druid 作者），GitHub `name` 字段英文为 Shaojin Wen。
- 跨仓库 PR 总量 ~1,520 为 GitHub 搜索 API 即时返回值，可能含跨 org 历史数据，仅作量级参考。
- star 数为查询当时（2026-05-31）的公开数据，会随时间变化。

---

> **免责声明**：本档案仅基于公开 GitHub 资料、公开 OSS 活动，以及本人从 GitHub profile 链出的个人主页整理；所有推断已显式标注「（推断）」；不含任何私人/未公开身份信息。数据截至 2026-05-31，可能已过时。
