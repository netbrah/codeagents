# Qwen Code 改进建议 — 对标 Qoder CLI

> 基于 Qwen Code 源码（v0.18.0）与 Qoder CLI v1.0.18 bundle 解码（`@qoder-ai/qodercli`，闭源，`_$d()` 反混淆）的逐项比对，识别 Qwen Code 可从 Qoder 借鉴的 **12 项**能力。
>
> **前置判断**：两家是 **Gemini CLI 的兄弟 fork**（详 [Qwen Code vs Qoder CLI](./qwen-code-vs-qoder-cli.md)）。**在绝大多数维度 Qwen Code 领先**（开放 daemon/ACP HTTP+WS/MCP-server 桥/三语言 SDK、Dynamic Workflow、Computer Use、Tool Search、完整 Agent Team、telemetry 深度——Qoder 这些全无）。因此本报告**比 Codex(30)/OpenCode(35) 报告短**，且多为 P2/P3；Qoder 的核心差异化（自营网关 + credit 计费 + 国内/全球双端点）是**商业模式，不可借鉴**。
>
> **相关报告**：
> - [Claude Code 改进建议报告（275 项）](./qwen-code-improvement-report.md)
> - [Codex CLI 对标改进（30 项）](./qwen-code-codex-improvements.md)
> - [OpenCode 对标改进（35 项）](./qwen-code-opencode-improvements.md)
> - [Gemini CLI backport（61 项）](./qwen-code-gemini-upstream-report.md)

---

## 一、改进项矩阵

| 优先级 | 改进点 | Qwen Code 现状 | 难度 | 来源 |
|:------:|--------|----------------|:----:|------|
| **P2** | [模型成本档位抽象](#item-1) — `lite/efficient/auto/performance/ultimate` 让用户按"成本/能力"选而非记模型名 | `inherit`/`fast`/具体名 | 中 | agent-creator SKILL.md |
| **P2** | [Tracker 任务 DAG + 可视化](#item-2) — 任务依赖拓扑（`AddDependency`）+ `Visualize` 树形渲染 | Agent Team `task_create` 有 blocks/blockedBy，无 Visualize | 中 | `Tracker*Tool` ×6 |
| **P2** | [Quest 规格驱动引导编排](#item-3) — spec → 确认 → 用专门 subagent 执行的引导式工作流 | `workflow` 是程序化脚本，无引导式 spec UX | 大 | `quest` bundled skill |
| **P2** | [外部 agent 迁移工具链](#item-4) — `migrate --from-claude` 一次性迁移 + Claude 插件市场格式 | hook 命名兼容，无迁移命令 | 中 | `qoder migrate` |
| **P2** | [Claude 插件市场互操作](#item-5) — 识别 `.claude-plugin` + `claude-plugins-official` 源 | extensions 三格式转换器（无 Claude 插件市场对接）| 中 | marketplace |
| **P3** | [per-agent 资源治理](#item-6) — `effort`(low/med/high/max) + `timeoutMins` 写进 agent frontmatter | `maxTurns`，无 per-agent effort/timeout | 小 | agent frontmatter |
| **P3** | [ImageSearchTool](#item-7) — 图像搜索工具（多模态，依赖 `sharp`）| 无图像搜索 | 中 | `ImageSearchTool` |
| **P3** | [会话分析报告](#item-8) — `/insights` 分析历史会话生成报告 | telemetry 有数据，无用户向会话分析 | 中 | `/insights` |
| **P3** | [agent/skill 创作 meta-skill](#item-9) — `agent-creator`/`skill-creator` 内置引导 skill | `/agents` create 向导，无 meta-skill 指南 | 小 | `builtin/*-creator` |
| **P3** | [skill-extractor agent](#item-10) — 从历史会话提炼可复用 skill 的专门 agent | auto-skill 抽取，无独立 extractor agent | 小 | `skill-extractor` agent |
| **P3** | [strict 沙箱档](#item-11) — Seatbelt 新增 `strict` 档（比 restrictive 更严）| `restrictive/permissive × closed/open/proxied`（无 strict）| 小 | `sandbox-macos-strict-*.sb` |
| **P3** | [subagent 实时活动流](#item-12) — `recentActivity` 把 subagent 的 thought/action 滚动展示 | LiveAgentPanel 显示状态，活动流粒度不同 | 小 | `subagentProgress` |

---

<a id="item-1"></a>

### 1. 模型成本档位抽象（P2）

**问题**：Qwen Code 的 subagent `model` 字段接受 `inherit`/`fast`/具体模型名（BYOK）。用户要换"更便宜"或"更强"的模型必须**知道具体模型名**——跨 provider 时心智负担重（`qwen3.5-plus` vs `claude-opus-4-8` vs `deepseek-...`）。

**Qoder 的做法**：6 档**能力梯抽象**（`agent-creator/SKILL.md` 明文）：`inherit`（默认=主模型）→ `lite`（简单问答）→ `efficient`（日常编码）→ `auto`（复杂多步）→ `performance`（硬工程）→ `ultimate`（最高质量）。用户按"我要多强/多省"选，由网关映射到具体模型。

**Qwen Code 借鉴方向**：在 BYOK 之上加一层**可选的成本档位别名**（如 `model: cheap|balanced|strong`），由用户在 settings 里把档位映射到各 provider 的具体模型；agent frontmatter 与 `--model` 都支持档位名。保留具体名直选不变。

**实现成本**：~1 周。**收益**：跨 provider 选模型从"记型号"变成"选档位"，多 provider 场景体验显著改善。

---

<a id="item-2"></a>

### 2. Tracker 任务 DAG + 可视化（P2）

**问题**：Qwen Code 的任务体系有两套——扁平 `todo_write` 和 Agent Team 的 `task_create/update/list`（支持 blocks/blockedBy 拓扑）。但**缺少可视化**：用户看不到任务依赖图。

**Qoder 的做法**：继承 Gemini 上游的 **Tracker 工具族 6 个**——`TrackerCreateTask` / `TrackerUpdateTask` / `TrackerGetTask` / `TrackerListTasks` / **`TrackerAddDependency`**（显式建依赖边）/ **`TrackerVisualize`**（渲染任务拓扑）。Qwen Code fork 时砍掉了 Tracker，保留扁平 TodoWrite。

**Qwen Code 借鉴方向**：给 Agent Team / TodoWrite 加一个 **`task_visualize`** 渲染（依赖 DAG 树形/图形），复用已有的 blocks/blockedBy 数据。Tracker 本就是上游代码，可 backport（详 [Gemini upstream 报告](./qwen-code-gemini-upstream-report.md)）。

**实现成本**：~1 周。**收益**：复杂多任务时依赖关系可视，调试编排更直观。

---

<a id="item-3"></a>

### 3. Quest 规格驱动引导编排（P2）

**问题**：Qwen Code 的 `workflow` 是**程序化**编排（写 JS 脚本调 `agent()`/`parallel()`/`pipeline()`），面向能写脚本的高级用户。普通用户要"先出规格、确认、再执行"的引导式特性开发流程，没有现成路径。

**Qoder 的做法**：`quest` bundled skill——解码描述为 **"Intelligent workflow orchestrator that guides users through feature development using specialized subagents"**。即 spec → 用户确认 → 调用专门 subagent 分步执行的**引导式工作流**（v0.x 招牌功能延续）。

**Qwen Code 借鉴方向**：在 `workflow` 程序化能力之外，加一个**引导式 spec skill**——先和用户敲定规格文档，确认后用 subagent（可复用 `Plan` agent）分阶段执行。与 Qwen 的 `enter_plan_mode` + Plan Approval Gate 天然契合。

**实现成本**：~2-3 周。**收益**：把多代理编排的门槛从"会写脚本"降到"会确认规格"。

---

<a id="item-4"></a>

### 4. 外部 agent 迁移工具链（P2）

**问题**：从 Claude Code / Cursor 切到 Qwen Code 的用户要手动重配 MCP/hooks/commands/subagents。Qwen 目前只做到 **hook 事件命名兼容**（Claude Code 10 事件），无一键迁移。

**Qoder 的做法**：`qoder migrate --from-claude` 一次性迁移命令 + CLI 参数面对齐 Claude Code（`--permission-mode`/`--output-style`/`--add-dir`/`--settings`）+ marketplace 识别 `.claude-plugin` 格式。

**参考**：与 [Codex 报告 item-29 External Agent 迁移](./qwen-code-codex-improvements.md#item-29) 同向（Codex 有 `external-agent-migration` crate）。

**Qwen Code 借鉴方向**：`qwen import --from claude-code|cursor` 子命令——扫 `~/.claude`，把 MCP/hooks/commands/subagents 保守转换进 `.qwen`，带 import ledger 可审计/撤销。

**实现成本**：~2 周。**收益**：用户迁移摩擦从"30 分钟重配"降到"一行命令导入"。

---

<a id="item-5"></a>

### 5. Claude 插件市场互操作（P2）

**问题**：Qwen Code 的 extensions 有三格式转换器（Claude/Gemini/Qwen），但**不对接 Claude Code 官方插件市场**——无法直接装 Claude 生态的插件。

**Qoder 的做法**：marketplace 预置 **`claude-plugins-official`** 源 + 识别 **`.claude-plugin`** 目录格式，即插即用 Claude 插件。

**Qwen Code 借鉴方向**：extensions marketplace 增加 Claude 插件市场源 + `.claude-plugin` 解析，把 Claude 插件生态纳入可安装范围。

**实现成本**：~2 周。**收益**：直接复用 Claude Code 插件生态。

---

<a id="item-6"></a>

### 6. per-agent 资源治理（effort + timeoutMins）（P3）

**问题**：Qwen Code agent frontmatter 有 `maxTurns`，但**无 per-agent 思考力度和超时**——一个"快速分类"agent 和一个"深度审计"agent 用同样的全局 effort。

**Qoder 的做法**：agent frontmatter 支持 **`effort`**（`low`/`medium`/`high`/`max`）+ **`timeoutMins`**（per-agent 超时）。把"这个 agent 该想多深、最多跑多久"写进定义。

**Qwen Code 借鉴方向**：agent frontmatter 加 `effort` + `timeoutMins` 字段，per-agent 覆盖全局。Qwen 已有 reasoning effort 概念，扩展到 per-agent 即可。

**实现成本**：~3 天。**收益**：agent 级成本/深度精细控制。

---

<a id="item-7"></a>

### 7. ImageSearchTool（P3）

**问题**：Qwen Code 无图像搜索能力。多模态任务（"找设计稿里的某组件"）无内置工具。

**Qoder 的做法**：`ImageSearchTool`（gemini-cli 上游为 0，Qoder 自加；依赖 `sharp` 多模态）。

**Qwen Code 借鉴方向**：评估是否加 `image_search` 工具（或经 MCP 接入），服务多模态/前端场景。

**实现成本**：~1 周。**收益**：多模态任务补齐（优先级低，受众窄）。

---

<a id="item-8"></a>

### 8. 会话分析报告（/insights）（P3）

**问题**：Qwen Code 有深度 telemetry（OTel），但**无用户向的会话分析报告**——用户看不到"我这周的会话模式/成本/常用工具"。

**Qoder 的做法**：`/insights`——"Generate a report analyzing your Qoder sessions"，把历史会话分析成个性化报告。

**参考**：与 [Copilot 的 `/chronicle`](../tools/copilot-cli/02-commands.md)（会话史检索/洞察/cost-tips）同向。

**Qwen Code 借鉴方向**：`/insights` 命令复用 telemetry/ChatRecording 数据生成用户向报告（成本、常用工具、模式）。

**实现成本**：~1-2 周。**收益**：用户自省 + 成本优化抓手。

---

<a id="item-9"></a>

### 9. agent/skill 创作 meta-skill（P3）

**问题**：Qwen Code 有 `/agents` create 向导，但**无引导用户写 agent/skill 的 meta-skill**——新手不知道 frontmatter 怎么写、描述怎么写才能被正确委派。

**Qoder 的做法**：内置 `agent-creator` / `skill-creator` 两个引导 skill（`builtin/*/SKILL.md`，明文），手把手教用户写有效的 agent/skill 定义（含 description 写法、工具白名单、模型档位选择最佳实践）。Claude Code 也有 skill-creator。

**Qwen Code 借鉴方向**：把 `/agents create` 的隐性知识固化成 `agent-creator`/`skill-creator` bundled skill，可被 `@` 调用引导创作。

**实现成本**：~3 天（主要是写 SKILL.md 文案）。**收益**：降低自定义 agent/skill 门槛。

---

<a id="item-10"></a>

### 10. skill-extractor agent（P3）

**问题**：Qwen Code 有 auto-skill（自动抽取），但**无独立的 skill 提炼 agent**让用户显式触发"把这次会话的可复用模式提炼成 skill"。

**Qoder 的做法**：`skill-extractor` 内置 agent——"Extracts reusable skills from past conversation sessions"。

**Qwen Code 借鉴方向**：提供一个显式 `skill-extractor` agent（或 `/extract-skill` 命令），让用户主动从历史会话沉淀 skill，补充 auto-skill 的被动路径。

**实现成本**：~3 天。**收益**：skill 沉淀从被动变可主动触发。

---

<a id="item-11"></a>

### 11. strict 沙箱档（P3）

**问题**：Qwen Code 的 macOS Seatbelt 矩阵是 `restrictive/permissive × closed/open/proxied`（跟的上游较早矩阵），**无 `strict` 档**——最严档位缺失。

**Qoder 的做法**：跟进上游新矩阵 `strict/restrictive/permissive × proxied/open`，多了比 restrictive 更严的 `strict` 档（`(deny default)` + 极少 allow）。

**Qwen Code 借鉴方向**：backport 上游 `strict` profile（[Gemini upstream](./qwen-code-gemini-upstream-report.md)，本就是上游代码），给安全敏感场景最严档位。

**实现成本**：~1 天。**收益**：补齐最严沙箱档。

---

<a id="item-12"></a>

### 12. subagent 实时活动流（P3）

**问题**：Qwen Code 的 LiveAgentPanel 显示 subagent **状态**（running/completed），但活动粒度偏粗。

**Qoder 的做法**：`subagentProgress.recentActivity[]` 把 subagent 的近期动作（type `thought`/`action`/`tool_call`/`tool_result`）滚动展示，运行中显示 `"Working..."` 思考流。

**Qwen Code 借鉴方向**：LiveAgentPanel 增加 `recentActivity` 滚动窗（最近 N 条 thought/tool），让用户看到 subagent "正在干什么"而非仅"在运行"。Qwen web-shell 已有 thinking 折叠（#4977），可复用。

**实现成本**：~3 天。**收益**：subagent 执行透明度提升。

---

## 二、不可借鉴（Qoder 商业模式 / 已落后项）

| Qoder 特性 | 为何不列入 |
|---|---|
| 自营网关 + credit 统一计费 + 国内/全球双端点 | **商业模式**，开源 BYOK 路线不适用 |
| `--remote`/`--teleport`/`/qr-code` 远程控制 | 绑定 Qoder 云闭环；Qwen 的 daemon + SDK 是更开放的等价物（[§七 被集成](./qwen-code-vs-qoder-cli.md#七被集成能力作为服务端--可嵌入对象)）|
| daemon / ACP HTTP+WS / MCP-server / SDK / Workflow / Computer Use / Tool Search | **Qwen 已领先，Qoder 无**（无需借鉴）|

## 三、Qwen Code 已显著领先 Qoder 的维度（对照）

| 维度 | Qwen Code | Qoder CLI |
|---|---|---|
| 被集成（北向接口）| 开放 daemon REST/SSE + ACP HTTP/WS + MCP-server 桥 + TS/Python/Java SDK | headless + ACP stdio + Qoder 云远程闭环 |
| Dynamic Workflow | ✅ `workflow`（确定性 fan-out 编排）| ❌ 无 |
| Computer Use | ✅ 9 工具（默认开 + deferred）| ❌ 无 |
| Tool Search | ✅ `tool_search`（延迟工具按需加载）| ❌ 无 |
| 多代理协作 | 完整 Agent Team（生命周期事件 + broadcast + 共享任务板）| 仅 `teammate_mailbox` 结果聚合 |
| 编排工具 | cron + monitor + workflow | ❌ 无 |
| telemetry | OTel 深度分层 tracing | 自有端点，深度未知 |
| 开源 | Apache-2.0 | 闭源 |

---

> **免责声明**：Qoder CLI 闭源，本文 Qoder 侧基于 `@qoder-ai/qodercli` v1.0.18 bundle 静态解码（`_$d()` XOR 反混淆，确定性可复现），无源码。Qwen Code 侧基于 v0.18.0 源码。改进项为对标分析，非 Qwen 团队承诺。截至 2026-06-14。
