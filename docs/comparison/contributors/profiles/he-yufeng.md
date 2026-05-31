# he-yufeng

> Qwen Code 贡献者档案 · 数据截至 2026-05-31

## 速览

| 项 | 值 |
|---|---|
| GitHub | [@he-yufeng](https://github.com/he-yufeng) |
| 真名 / 显示名 | Yufeng He（GitHub profile `name` 字段） |
| 公司 / 组织 | Moonshot AI（Kimi，profile `company` 字段）— 注意：并非 Qwen/阿里方人员（见备注） |
| 地区 | Hong Kong（香港） |
| 教育 / 背景 | MS CS @ HKU（香港大学）；3× ACM-ICPC Silver；曾实习 Baidu / Maimai / Kuaishou（均取自 profile `bio`） |
| 注册时间 | 2018-06-09（账号 ID 40085740） |
| 2026-05 qwen-code merged PR | 6（+1,128 / −204，合计 1,332 LOC） |
| qwen-code 累计 merged PR | 6（全部集中在 2026-05-28 ~ 05-31 一周内） |
| 首次 qwen-code PR | 2026-05-28 / #4607（`fix(core): use undici fetch for IDE proxy requests`，merged 05-29） |
| 角色推断 | 外部社区贡献者（非 collaborator）；多 OSS 项目「bug 修复型」高产 drive-by contributor（推断） |

## 在 Qwen Code 的工作

6 个 merged PR 全部为 `fix(...)` 前缀的缺陷修复，无 feature，集中在 core 协议正确性与 CLI/ACP 细节，单周完成。归纳方向：

- **OpenAI/兼容协议消息正确性**（最大单 PR）：#4622 `enforce adjacent tool results`（+673/−111，本月最大）改 `openaiContentGenerator/converter.ts`，强制 tool_call 与对应 tool result 在请求里相邻——这是发往 OpenAI 兼容后端时常见的"tool message 顺序/配对"约束修复。
- **IDE 集成 / 网络栈**：#4607 `use undici fetch for IDE proxy requests` 改 `ide/ide-client.ts`，让 IDE proxy 请求走 undici fetch（绕开默认 fetch 在代理场景下的行为问题）。
- **多语言输出一致性**：#4636 `apply output language to side queries` 改 `config.ts` + `utils/sideQuery.ts`，把用户的 output language 设置也应用到"侧查询"（如标题生成等辅助 LLM 调用）。
- **健壮性 / 错误处理**：#4632 `harden context error text collection`（`utils/contextLengthError.ts`），加固上下文超长错误文本的收集。
- **ACP 鉴权清理**：#4639 `drop discontinued Qwen OAuth method` 改 `acp-integration/acpAgent.ts` + `authMethods.ts`，移除已停用的 Qwen OAuth 鉴权方式。
- **CLI/TUI 细节**：#4635 `hide completed sticky todos` 改 `ui/components/StickyTodoList.tsx` + `ui/utils/todoSnapshot.ts`，已完成的置顶 todo 不再显示。

代表作 PR：

- [#4622 enforce adjacent tool results](https://github.com/QwenLM/qwen-code/pull/4622) — 本月体量最大（+673/−111）、技术含量最高，触及 OpenAI 兼容内容生成器的消息序列正确性，是其在本仓库最具代表性的工作。
- [#4607 use undici fetch for IDE proxy requests](https://github.com/QwenLM/qwen-code/pull/4607) — 任务已知项，IDE proxy 网络栈修复，也是其首个被合入的 qwen-code PR。

> 另有 2 个 PR 截至 2026-05-31 仍 OPEN（#4596 git submodule 抓取、#4600 auto approval 指示器），1 个被 CLOSE（#4638，与已合入的 #4622 主题重复）。当月在本仓库的实际尝试 ≥ 9 个 PR。

技术专长（推断）：LLM agent 协议层（OpenAI/Gemini/Anthropic 兼容的消息序列、tool call 配对、reasoning/citation 元数据处理）、Python 与 TypeScript 双栈、agent/eval 基础设施。从其大量自有仓库看，对"轻量编码 agent"（CoreCoder ~1.4k LoC、AnyCoder）、LLM 成本/可观测性工具（TokenTracker、PromptDiff、AgentProbe、LiteBench）有持续投入（推断）。

## 跨仓库 / 社区足迹

- **公开 merged PR 总计 109**，分布在 30+ 仓库，qwen-code 仅占 6 个——是典型的**广撒网 bug-fix 型外部贡献者**，而非单一项目深耕者（推断）。前几大目标仓库：
  - 25× `AstrBotDevs/AstrBot`、11× `microsoft/agent-framework`、8× `UKGovernmentBEIS/inspect_ai`、7× `kvcache-ai/Mooncake`、7× `vllm-project/vllm`、6× `QwenLM/qwen-code`、4× `Arize-ai/phoenix`、3× `HKUDS/LightRAG`、3× `strands-agents/sdk-python`、3× `OpenHands/software-agent-sdk` 等。
  - 内容多为协议/适配器层修复，如 vllm #43243（Qwen3 XML JSON 参数解析）、strands-agents（vllm reasoning deltas、Gemini safety 元数据、并发 tool result 顺序）、microsoft/agent-framework（Foundry headers、citation get_url 元数据）、modelcontextprotocol/go-sdk（MCP implementation metadata）。主题与其 qwen-code 工作高度一致：**LLM 后端兼容性与消息正确性**。
- **自有项目（共 104 个公开仓库，多为原创 Python）**：`CoreCoder`（★1034，极简编码 agent，~1.4k LoC，前身 NanoCoder）、`FindJobs-Agent`（★223）、`RepoWiki`（★152，DeepWiki 开源替代）、`ContractGuard`（★146）、`RuleForge`（★100，自动生成 CLAUDE.md/.cursorrules）、`GitSense`（★89）、`TokenTracker`（★61）、`AgentProbe`（★39）等；另有量化/RL 交易类（DRL-MultiFactorTrading、MTSIR3-GAN）反映其量化与 ML 背景。
- **关注度**：362 followers / following 13，影响力高于纯内部工程师，符合"活跃 OSS 个人开发者 + AI researcher"画像（推断）。
- **个人主页 / 社交**：LinkedIn `in/yufenghe`（profile `blog` 字段）；个人博客仓库 `he-yufeng.github.io`、`universeplayer.github.io`；profile `email` 未公开、`twitter_username` 未公开。

## 备注 / 不确定项

- **身份冲突点（重要）**：GitHub profile `company` 明确写 "Moonshot AI（Kimi）"，bio 自述 "AI Researcher @ Moonshot AI (Kimi)"。即他对 **竞品方** Qwen Code 的贡献属于**外部社区贡献**，几乎可以确定**不是** QwenLM 团队成员/collaborator（推断，依据：账号注册较晚、qwen-code PR 全部为本周新出现的 drive-by fix、且其全网 PR 呈现明显的"多仓库 bug-fix 巡游"模式）。本档案未独立核实其 Moonshot 雇佣关系，均取自公开 profile 自述。
- 真名 "Yufeng He" 仅取自 GitHub profile 显示名，未做进一步身份关联。
- 当月 6 merged 与累计 6 merged 一致（首 PR 即在 2026-05-28），无历史积累；若其后续持续提交，累计数会快速变化。
- bio 中的 ACM-ICPC、HKU、各厂实习经历均为 profile 自述，未独立核实。

---
> 免责声明：本档案仅基于公开 GitHub 资料与公开 OSS 活动整理，截至 2026-05-31。所有身份/背景推断已标注"（推断）"，未公开信息标注"未公开"，不含任何私人信息。
