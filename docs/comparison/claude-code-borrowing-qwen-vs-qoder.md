# Claude Code 能力借鉴对比：Qwen Code vs Qoder CLI

> **问题**：Qwen Code 与 Qoder CLI 都是 **Gemini CLI 的兄弟 fork**，却又都在向 **Claude Code** 看齐。本文以 [Qwen Code 改进建议报告（对标 Claude Code，275 项）](./qwen-code-improvement-report.md) 为参照骨架，逐类对比**两个产品各自从 Claude Code 借鉴了什么、借到多深**。
>
> **一句话结论**：**两家从不同角度收敛到 Claude Code**——**Qwen Code 借的是"引擎/深度"**（subagent 编排、fork、hooks 多类型、tool_search、结构化输出、压缩、记忆、Computer Use……多以源码 PR 落地）；**Qoder CLI 借的是"表面/兼容"**（CLI 参数命名、hook 事件命名、agent 定义文案逐字、工具命名 `Bash/Read/Edit`、`migrate --from-claude`、`.claude-plugin` 市场……为降低 Claude Code 用户迁移成本）。
>
> **证据**：Qwen 侧以 [改进报告](./qwen-code-improvement-report.md)（PR 级追踪）为准；Qoder 侧基于 v1.0.18 bundle 解码（`_$d()` 反混淆，确定性可复现，详 [EVIDENCE](../tools/qoder-cli/EVIDENCE.md)）。截至 2026-06-14。
>
> **相关**：[Qwen Code vs Qoder CLI 对称对比](./qwen-code-vs-qoder-cli.md) · [Qwen × Qoder 整合能力对照](./qwen-code-qoder-integration.md)

---

## 〇、为什么两家都借 Claude Code

两者基座（Gemini CLI）本身**不含** Claude Code 的若干招牌设计——逐一 grep 上游 gemini-cli 确认：**无** subagent 体系（`tools/agent/`、`subagents/`、`subagent_type`/fork、内置 agent 注册表）、**无** Claude 命名的 hook 事件、**无** `tool_search`。这些在两家 fork 里却都存在，证明**都是各自在 Gemini 基座上"补"了 Claude Code 的设计**。

差别在补的**层次**：

| | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 借鉴角度 | **引擎/运行时深度** | **表面/迁移兼容** |
| 典型动作 | 把 Claude 的能力**重新实现进引擎**（PR 级，开源可追）| 把 Claude 的**接口/命名/文案对齐**（bundle 解码可见）|
| 目的 | 做成开源 runtime 平台 | 降低 Claude Code 用户切换成本 |
| 深度信号 | Agent Team、Computer Use、Tool Search、结构化输出、三档压缩…… | CLI 参数面、hook 事件名、agent 文案逐字、`migrate --from-claude` |

---

## 一、SubAgent 体系（两家都重度借鉴，深度不同）

Claude Code 的 subagent 系统是两家**共同**借鉴的最大块——内置 agent 描述文案在 Qwen 源码与 Qoder bundle 中**逐字相同**，且都是 Claude Code 原文。

| Claude Code 能力 | Qwen Code 借鉴 | Qoder CLI 借鉴 |
|---|---|---|
| `subagent_type` 工具参数 | ✅ 源码 `tools/agent/agent.ts` | ✅ TaskTool（解码触发逻辑逐字相同）|
| 内置 `general-purpose`/`Explore` agent | ✅（文案逐字）| ✅（文案逐字）|
| 内置 `Plan` 架构师 agent | —（Qwen 未内置 Plan agent，有 `enter_plan_mode`）| ✅ `Plan`（"Software architect agent…"）|
| 内置 `statusline-setup` agent | ✅ | ✅ |
| **Fork Subagent**（省略 type=隐式 fork，继承全上下文）| ✅ FORK_AGENT（[报告 item-2](./qwen-code-improvement-report.md)，PR#2936）| ✅ **同款**（描述+触发逻辑逐字相同）|
| `isolation: worktree` 字段 | ✅ | ✅ `enum[default,worktree]` |
| `/agents` 管理 UI | ✅（create/manage 向导）| ✅（create/list）|
| **Coordinator/Swarm 多 Agent 编排** | ✅ **完整 Agent Team**（生命周期事件 + broadcast + 共享任务板，[报告 item-14/25](./qwen-code-improvement-report.md)）| ⚠️ **仅 `teammate_mailbox` 结果聚合**（无编排）|
| agent 细粒度工具访问控制 | ✅（3 层 allowlist，PR#3064/#3066）| ✅ + **规则按 `subagent_type` 维度匹配** |
| Agent 恢复与续行 | ✅（PR#3739）| 未见 |

**判读**：subagent 的**定义与基本调用**两家借得一样深（文案都直接搬 Claude）。但 Claude 的**多 Agent 编排深层**（Coordinator/Swarm）——Qwen 重新实现成完整 Agent Team，Qoder 止步结果聚合。Qoder 反而内置了**更多** Claude agent（多 `Plan`/`SaveMemory`/`skill-extractor`）。

---

## 二、Hooks（两家都改用 Claude 事件命名）

Gemini CLI 上游 hooks 用自有命名（`BeforeTool`/`AfterAgent`…），并提供 `hooks migrate` 把 Claude 配置**转换**成自有名。**两个阿里 fork 却都直接采用了 Claude Code 的事件命名**（上游命名在两者中均为 0）。

| Claude Code Hook 能力 | Qwen Code | Qoder CLI |
|---|---|---|
| 10 事件命名（`PreToolUse`/`PostToolUse`/`PostToolUseFailure`/`UserPromptSubmit`/`SessionStart`·`End`/`PreCompact`/`Stop`·`SubagentStop`/`Notification`）| ✅ 全套 | ✅ 全套（计数 24~55×）|
| **HTTP Hooks**（POST JSON）| ✅（[报告，PR#2827](./qwen-code-improvement-report.md)）| ❌ 仅 command + `/hook-config` |
| Function Hook / Async Hook | ✅ | ❌ |
| SSRF 防护 | ✅（`ssrfGuard.ts`）| ❌ |
| `CLAUDE_PROJECT_DIR`/`CLAUDE_PLUGIN_ROOT` 环境变量 | 部分 | ✅（Claude 插件兼容）|

**判读**：**事件命名两家都借**（配置可互用）；但 Claude hook 的**类型扩展**（HTTP/Function/Async）只有 Qwen 重新实现，Qoder 停在 command。

---

## 三、CLI 表面 / 参数命名（Qoder 借得远比 Qwen 多）

这是 **Qoder 借鉴 Claude Code 最显著**的一块——Qoder v1.0 的 CLI 参数面大面积对齐 Claude Code，上游 Gemini CLI 与 Qwen Code 均无这些命名。

| Claude Code CLI 参数/概念 | Qwen Code | Qoder CLI |
|---|---|---|
| `--permission-mode` | ❌（用 `--approval-mode`）| ✅ |
| `--output-style` / Output Styles | ❌ | ✅（`outputStyle` 4×）|
| `--add-dir` | ❌（用 `--include-directories`）| ✅ |
| `--settings` / `--setting-sources` | ❌ | ✅ |
| `--strict-mcp-config` | ❌ | ✅ |
| `--agents <json>` | ❌ | ✅ |
| `--allowed-tools`/`--append-system-prompt`/`--fork-session` | ✅ 部分 | ✅ |
| 工具命名 `Bash`/`Read`/`Write`/`Edit`/`Glob`/`Grep`/`WebSearch`/`WebFetch` | ❌（保留 Gemini 系 `run_shell_command`/`read_file`）| ✅ **全套 Claude 命名** |

**判读**：**Qoder 把 Claude Code 的用户界面几乎照搬**（参数 + 工具命名），目的是让 Claude Code 用户"零学习"上手；Qwen 保留 Gemini 系命名，只借少量参数。这是两家借鉴角度差异最直白的体现。

---

## 四、迁移与生态兼容（Qoder 独有的 Claude 借鉴）

| Claude Code 兼容能力 | Qwen Code | Qoder CLI |
|---|---|---|
| **一键迁移** `migrate --from-claude` | ❌ | ✅（一次性迁移 MCP/hooks/commands/subagents）|
| `.claude-plugin` 插件格式 | ❌ | ✅（marketplace 识别）|
| `claude-plugins-official` 市场源 | ❌ | ✅ 预置 |
| 上下文用量 UI 标 `claudeMd` 类型 | — | ✅ |
| 读取 `CLAUDE.md`（运行时）| ✅（读取）| v0.x 有 `--with-claude-config`，**v1.0 移除**，改迁移命令 |

**判读**：**迁移工具链是 Qoder 几乎独占的 Claude 借鉴**——服务"从 Claude Code 切过来"的用户。Qwen 仅做到 hook 命名兼容，无迁移命令。

---

## 五、引擎/运行时能力（Qwen 借得远比 Qoder 深）

Claude Code 的这些**引擎级**能力，Qwen 多数已重新实现（报告里 PR 级追踪），Qoder 基本未投入。

| Claude Code 能力 | Qwen Code | Qoder CLI |
|---|---|---|
| **Computer Use**（GUI 自动化）| ✅ 9 工具（默认开 + deferred）| ❌（bundle 全量核查无）|
| **Tool Search / 延迟工具加载** | ✅ `tool_search`（[报告，PR#3589](./qwen-code-improvement-report.md)）| ❌（仅 `deferred_tools_delta` 通知）|
| **Structured Output**（`--json-schema`）| ✅（PR#3598）| ❌ |
| **多层上下文压缩** | ✅ 三档梯度（PR#4345）| `/compact`（单档）|
| **会话记忆 + Auto Dream** | ✅（PR#3087 托管记忆）| `/memory` + `SaveMemory`（无 auto-dream）|
| **Speculation 预测执行** | ✅（PR#2525）| ❌ |
| **Mid-Turn Queue Drain** | ✅（PR#2854）| 未见 |
| **Notebook Edit** | ✅（PR#3900）| ❌ |
| **FileIndex 模糊搜索** | ✅（fzf 风格）| 未见 |
| **Fast Model / Advisor 顾问模型** | ✅ 多场景 | ❌（`fastModel`/`advisor` 0）|
| Commit Attribution（AI 贡献统计）| ✅ 独家超 Claude（git notes）| ✅ `commit` 子命令（AI 占比）|
| Plan 模式 + Approval Gate | ✅ `enter_plan_mode`（PR#4853）| ✅ `/plan`（Toggle Plan Mode）|
| Thinking 块渲染/折叠 | ✅（web-shell #4977）| ⚠️ 有 `thinking`/`reasoning` 渲染，折叠未确认 |

**判读**：Claude Code 的**运行时深度能力**几乎是 **Qwen 单方面在追**——Computer Use / Tool Search / 结构化输出 / 分层压缩 / Speculation / 队列注入 Qoder 全无。这与"Qwen 做 runtime 平台"路线一致。

---

## 六、被集成 / 北向接口（Claude Code 的平台能力）

Claude Code 有 SDK、Bridge、daemon-style 控制面。两家借鉴差距巨大。

| Claude Code 平台能力 | Qwen Code | Qoder CLI |
|---|---|---|
| SDK（程序化嵌入）| ✅ TS/Python/Java | ❌ 无公开 SDK |
| 作为 MCP server 被调 | ✅ `qwen-serve-bridge` | ❌ 仅 MCP 客户端 |
| Remote Control Bridge（手机/浏览器驱动）| ✅ Channels + daemon | ✅ `--remote`/`/qr-code`（绑 Qoder 云）|
| ACP HTTP/WS | ✅ | ⚠️ 仅 stdio |
| `/teleport` 跨端迁移 | 开发中 | ✅ `--teleport`（Qoder 云）|

**判读**：Claude 的**开放平台能力**仍是 Qwen 在系统性借鉴（SDK/MCP-server/ACP HTTP-WS）；Qoder 的远程是绑定自家云的闭环。详 [被集成对比](./qwen-code-vs-qoder-cli.md#七被集成能力作为服务端--可嵌入对象)。

---

## 七、命令与 meta-skill（两家都借 Claude 命令习惯）

Claude Code 的若干招牌命令/skill 两家都搬了，Qoder 在 meta-skill 上更全。

| Claude Code 命令/skill | Qwen Code | Qoder CLI |
|---|---|---|
| `/rewind`（时间线回滚）| ✅ | ✅（别名 `checkpoint`）|
| `/statusline`（自定义状态栏，用 sub-agent）| ✅ | ✅（"Uses a sub-agent…"）|
| `/security-review` | ✅（bundled skill）| ✅（bundled skill）|
| `/context`（上下文可视化）| ✅ | ✅ |
| `/compact` | ✅ `/compress` | ✅ |
| **skill-creator meta-skill** | ✅（`skills/builtin/skill-creator`）| ✅（`builtin/skill-creator`）|
| **agent-creator meta-skill** | —（`/agents` 向导）| ✅（`builtin/agent-creator` 引导 skill）|

**判读**：命令层两家都向 Claude 习惯靠拢；**meta-skill（教用户写 agent/skill）Qoder 更全**——把 Claude 的"创作引导"也搬了。

---

## 八、量化小结：各借了 Claude Code 多少

> 按本文 ~50 个 Claude Code 能力点粗略归类（非穷尽，仅示意借鉴**密度与角度**）。

| 借鉴类别 | Qwen Code | Qoder CLI |
|---|---|---|
| **引擎/运行时深度**（subagent 编排、fork、tool_search、computer use、结构化输出、压缩、记忆、speculation…）| ★★★★★ 系统性重实现（PR 级）| ★★ 仅 fork + 基本 subagent |
| **表面/UX 对齐**（CLI 参数、工具命名、output-style、permission-mode）| ★★ 少量参数 | ★★★★★ 几乎照搬 |
| **Hook 事件命名** | ★★★★★（+ HTTP/Function/Async 扩展）| ★★★（命名对齐，类型未扩）|
| **迁移/生态兼容**（migrate/.claude-plugin/market）| ★ 仅 hook 命名 | ★★★★★ 迁移链完整 |
| **平台/被集成**（SDK/MCP-server/ACP HTTP-WS）| ★★★★★ | ★ 远程绑自家云 |
| **命令/meta-skill 习惯** | ★★★★ | ★★★★（meta-skill 更全）|

---

## 九、结论：两种"向 Claude 收敛"的路径

- **Qwen Code = 借 Claude 的"内功"**。把 Claude Code 的引擎能力（多 Agent 编排、fork、tool_search、computer use、结构化输出、分层压缩、记忆、SDK/Bridge）**重新实现进开源引擎**，目标是做成与 Claude Code 同级的 runtime 平台。借鉴**深、内化、可追溯到 PR**。
- **Qoder CLI = 借 Claude 的"皮肤"**。把 Claude Code 的**用户界面**（CLI 参数、工具命名、output-style、hook 事件名、agent 定义文案、迁移链、插件市场格式）**对齐到位**，目标是让 Claude Code 用户**无痛迁移**到自营网关计费产品。借鉴**广、表层、面向兼容**。

两家在 subagent 定义、hook 命名、命令习惯上**借得一样多**（都直接搬 Claude 文案/命名）；分野在：**引擎深度 Qwen 远超**，**迁移/表面兼容 Qoder 远超**。这恰好对应两家的产品定位——开源 runtime 平台 vs 闭源迁移友好的商业产品。

---

> **免责声明**：Qwen 侧借鉴状态以 [改进报告](./qwen-code-improvement-report.md)（持续 PR 追踪）为准，可能滞后于最新 main；Qoder 侧基于 v1.0.18 bundle 静态解码（无源码），借鉴判断按 Claude Code 公开特征比对。"借鉴"指能力/接口相似性，不含代码同源主张（两家代码基座均为 Gemini CLI）。截至 2026-06-14。
