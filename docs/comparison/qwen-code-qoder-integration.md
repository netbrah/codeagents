# Qwen Code × Qoder CLI 整合能力对照

> **用途**：本文面向 **Qwen Code 团队与 Qoder CLI 团队**，为两条 Gemini CLI 兄弟 fork 的**整合/收敛**提供中立、详尽的逐子系统能力对照。每个子系统给出**两侧实现 + 成熟度 + 整合建议**（往哪侧收敛 / 如何合并 / 保留为产品层），不偏向任一方。
>
> **定位说明**：这**不是**单向改进建议，而是双向整合参考。"整合建议"列只反映"哪侧实现更完整/更适合作为收敛基线"的工程判断，最终取舍由两团队决定。
>
> **证据基础**：Qwen Code v0.18.0 源码 + Qoder CLI v1.0.18 bundle 解码（`_$d()` = base64 后 `tBpirNfrja2H` 循环异或，确定性可复现）。详细提取见 [Qoder EVIDENCE](../tools/qoder-cli/EVIDENCE.md)；对称对比见 [Qwen Code vs Qoder CLI](./qwen-code-vs-qoder-cli.md)。截至 2026-06-14。

---

## 〇、整合总览

两者是 **Gemini CLI 的两个兄弟 fork**（均阿里系，互不衍生；Qoder 基线 ≈ gemini-cli v0.40.x / 2026-04 下旬）。**底座同源**（ContentGenerator / MCP / 内置工具基本盘 / Seatbelt / skills / slash 命令框架），分歧集中在**上层产品取向**：

- **Qwen Code** 走 **开源 runtime 平台**：daemon 多端、开放北向接口（ACP HTTP/WS + MCP-server + 三语言 SDK）、Agent Team、Dynamic Workflow、Computer Use、Tool Search、深度 telemetry。
- **Qoder CLI** 走 **闭源商业产品**：自营网关 + credit 计费 + 国内/全球双端点、Claude Code 表面对齐（CLI 参数/hooks/迁移/插件市场）、模型成本档位、Quest 规格驱动、更多内置 agent。

**收敛基线判断（工程视角）**：**引擎层以 Qwen Code 为收敛基线**（开源、北向接口完整、运行时能力宽）更省力；**Qoder 的商业层**（网关/计费/双端点/Claude 迁移链/Quest）作为**产品适配层**叠加。下表逐子系统给出依据。

**整合建议图例**：◀Q = 收敛到 Qwen 实现；▶O = 收敛到 Qoder 实现；⊕ = 两侧合并取并集；▣ = 保留为 Qoder 产品层（不进开源引擎）；≈ = 同源，择一即可。

---

## 一、基座 / 构建 / 分发

| 维度 | Qwen Code | Qoder CLI v1.0 | 整合建议 |
|---|---|---|---|
| 语言/基座 | Gemini CLI fork（TS）| Gemini CLI fork（TS，基线 v0.40.x）| ≈ 同源，统一到一支 main |
| 分发 | npm（TS 源 + 包结构）| npm（34MB esbuild 单 bundle + `_$d()` 字符串混淆）| ◀Q 开源不混淆，利于协作 |
| 上下文文件 | `QWEN.md`（兼容 GEMINI.md）| `AGENTS.md`（+ SKILL.md/MEMORY.md）| ⊕ 多文件名兼容读取 |
| 配置目录 | `~/.qwen` `.qwen/` | `~/.qoder` `.qoder/` | 收敛需统一目录 + 迁移脚本 |
| 主题 | 继承 Gemini | 10 主题（catppuccin/dracula/qoder 等）| ⊕ 取并集 |

---

## 二、模型接入 / 认证 / 计费

| 维度 | Qwen Code | Qoder CLI v1.0 | 整合建议 |
|---|---|---|---|
| 模型接入 | **BYOK 任意 provider**（OpenAI/Anthropic/Gemini 兼容 + DeepSeek 双协议）+ 阿里云 Coding Plan | **自营网关** `api.qoder.sh`/`api.qoder.com.cn` 聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek | ⊕ 引擎保 BYOK，网关作为一种 provider 后端 |
| 模型选择粒度 | `inherit`/`fast`/具体模型名 | **6 档成本梯** `inherit/lite/efficient/auto/performance/ultimate`（映射到网关）| ⊕ 引擎加"档位别名→具体模型"映射层，两种并存 |
| 认证 | Qwen OAuth〔免费层 2026-04-15 停〕/ Coding Plan / API key | 浏览器登录 + PAT（`QODER_PERSONAL_ACCESS_TOKEN`）| ⊕ 多 auth-type 并存 |
| 计费 | 软件免费，模型自付 | credit 统一计费（含 credit 耗尽告警）| ▣ 计费是商业层，不进开源引擎 |
| 国内/全球 | 单端点 | **双端点**（合规）| ▣ 产品层；引擎支持 endpoint 配置即可 |

---

## 三、命令系统

- **Qwen Code**：57 内置命令（v0.18.0，52 常驻 + 5 条件）+ 7 bundled skills + MCP prompts + `.qwen/commands/*.md`
- **Qoder CLI**：78 个 `kind:"built-in"` 命令名（含子命令，顶层 ≈ 50）+ 8 bundled skills（`batch/debug/loop/mcp-config/quest/remember/security-review/simplify`）+ `/goal`

**重合度高**（解码后确认）——`/btw` `/branch` `/rename` `/plan` `/compact` `/tasks` `/goal` `/diff` `/statusline` `/tools` `/settings` `/setup-github` 及 skills `loop/batch/simplify` **两侧同名同义**。

| 类别 | 仅 Qwen | 仅 Qoder | 整合建议 |
|---|---|---|---|
| 工程化 | `/arena`（多模型竞争）`/insight`（代码洞察）`/fork`（后台代理）`/recap` `/summary` `/approval-mode` | `/quest`（规格驱动）`/insights`（会话分析）`/kanban`（看板）`/profile` | ⊕ 取并集 |
| 平台/UI | `/language`（6 语言 UI）`/ide` `/lsp` `/terminal-setup` `/trust` | `/qr-code` `/remote-env` `/start`·`/disconnect`（远程）`/voice` `/corgi` | ⊕；远程命令归 Qoder 产品层 |
| 记忆 | 托管自动记忆 `/dream` `/forget` `/remember` | `/memory` + `remember` skill + `/manage` | ◀Q Qwen 托管记忆更完整 |
| 命令变量 | `@{}` `!{}` `{{args}}` | 插件分发 | ⊕ |

> 早前误判已更正：`/btw` 两侧都有（描述逐字相同），`/recap` 仅 Qwen。`/token` `/metrics` `/otel` 在 Qoder 是 OAuth/Prometheus 端点，非命令。

---

## 四、工具系统

文件/搜索/Shell/Web 基本盘**同源**。下表只列分化项。

| 工具 | Qwen Code | Qoder CLI v1.0 | 整合建议 |
|---|---|---|---|
| 文件/编辑/Shell/glob/grep/web_fetch | ✅ 同源 | ✅ 同源 | ≈ |
| `notebook_edit`（Jupyter）| ✅ | ❌ | ◀Q |
| `web_search` | **已移除** | ✅ `WebSearchTool` | ⊕ 引擎恢复（Qoder 侧留着）|
| `save_memory` | **已移除**（改托管记忆）| ✅ `MemoryTool` | ◀Q 托管记忆替代 |
| `ImageSearchTool`（多模态）| ❌ | ✅（依赖 `sharp`）| ▶O 引擎吸收 |
| 任务管理 | `todo_write` + Agent Team `task_create/update/list`（blocks/blockedBy）| **Tracker DAG ×6**（含 `AddDependency` + **`Visualize`**）| ⊕ Qwen 任务体系 + Qoder 的 Visualize |
| `tool_search`（延迟加载）| ✅ 模型可调用 | ❌（仅 `deferred_tools_delta` 通知）| ◀Q |
| `monitor`（后台监控）| ✅ | ❌ | ◀Q |
| `cron`（定时）| ✅（默认开）| ❌ | ◀Q |
| `enter/exit_worktree` 工具 | ✅ 运行中进出 | ❌（仅 `--worktree` 参数）| ◀Q |
| `structured_output`（`--json-schema`）| ✅ | ❌ | ◀Q |
| `lsp` | ✅（实验）| ❌ | ◀Q |
| computer_use ×9 | ✅（默认开，deferred）| ❌ | ◀Q |
| `workflow` | ✅ | ❌ | ◀Q |
| MCP 动态工具 | `mcp__server__tool` | `DiscoveredMCPTool` | ≈ |
| MCP 认证工具 | — | `McpAuthTool` | ⊕ |

---

## 五、SubAgent（六维详对照）

**前提**：subagent 体系**两侧都非 Gemini 继承**（上游无 `tools/agent/`、`subagents/`、`subagent_type`/fork/内置注册表），是**各自叠加的 Claude Code 式 agent 系统**——内置 agent 描述文案两侧**逐字相同**（均 Claude 原文）。

### 5.1 内置 agent

| | Qwen Code（4）| Qoder CLI（8，更多）|
|---|---|---|
| 共有 | `general-purpose`(默认) `Explore` `fork`(隐式) `statusline-setup` | 同左 4 个 |
| Qoder 额外 | — | **`Plan`**（架构师）`SaveMemory` `skill-fork` `skill-extractor`（从历史会话提炼 skill）|

**整合**：⊕ 取并集（Qoder 的 Plan/SaveMemory/skill-extractor 补入）。

### 5.2 定义与调用

| 维度 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| 定义格式 | MD + YAML frontmatter | MD + YAML（`agent-creator/SKILL.md` 明文）| ≈ |
| 定义层级 | 4 级（builtin/user/project/extension）| 2 级（user/project）| ◀Q 层级更全 |
| 工具命名 | Gemini 系（`run_shell_command`）| **Claude 系**（`Bash`/`Read`/`Edit`…）| 收敛需统一命名 + 别名兼容 |
| 隐式 fork | ✅ FORK_AGENT（继承全对话 + 字节级 cache parity + ALS 嵌套检测）| ✅ **同款**（触发逻辑逐字相同）| ≈ |
| isolation | `worktree` | `enum[default,worktree]` | ≈ |
| run_in_background | ✅ | ✅ | ≈ |
| 权限按 agent | per-agent tools/disallowedTools | ✅ + **规则按 `subagent_type` 维度匹配** | ⊕ Qoder 的 type 维度规则补入 |

### 5.3 模型与资源（分化）

| 维度 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| 模型 | `inherit`/`fast`/具体名 | 6 档成本梯 | ⊕（同二节）|
| 思考力度 | runConfig | **`effort`**(low/med/high/max) per-agent | ▶O 引擎加 per-agent effort |
| 超时 | runConfig | **`timeoutMins`** per-agent | ▶O 引擎加 per-agent timeout |
| 轮次 | `maxTurns` | `maxTurns` | ≈ |

### 5.4 多代理协作（深浅不同，非有无）

| 能力 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| teammate 概念 | ✅ Claude 式 | ✅（`teammate_mailbox`/`teammateName`/`teammate_shutdown_batch`）| — |
| 协作深度 | **完整 Agent Team**：生命周期事件（joined/idle/exited/status/approval，`teammate_message` 35×）+ broadcast + 共享任务板 `~/.qwen/tasks/{team}` + `team_*` 工具 | **仅结果聚合**：`teammate_mailbox` 汇集多 subagent `{from,text,state}` | ◀Q 收敛到 Qwen 完整实现 |
| 编排工具 | cron/monitor/workflow | — | ◀Q |
| 嵌套 | depth 追踪（ALS），fork 禁止再 spawn | depth 追踪（`SubAgentLifecycle`）| ≈ |
| 并发 | 单消息多 Agent block | 进程级 `concurrencyLimit`(默认 10) | ⊕ |

### 5.5 进度展示

| 维度 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| 进度事件 | LiveAgentPanel + pill+dialog | `subagentProgress{agentName,state,recentActivity}` | — |
| 常驻并行面板 | ✅ **LiveAgentPanel**（always-on）+ InlineParallelAgentsDisplay | ❌ 内联树形 + `/tasks` 面板 | ◀Q 收敛到 LiveAgentPanel |
| 实时活动流 | 偏状态粒度 | **`recentActivity[]`**（thought/action/tool_call/tool_result 滚动）| ⊕ 把 Qoder 的活动流粒度并入 LiveAgentPanel |

---

## 六、高级能力（Qwen 独有，Qoder 全无）

| 能力 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| **Dynamic Workflow** | ✅ `workflow`（`WorkflowOrchestrator` + sandbox，JS 脚本 `phase/agent/parallel/pipeline` 确定性 fan-out，≤1000 agent/run）| ❌（bundle 全量核查无）| ◀Q |
| **Computer Use** | ✅ 9 工具（默认开 + deferred，首次装上游二进制 + macOS 权限探测）| ❌ | ◀Q |
| **Tool Search** | ✅ `tool_search`（延迟工具按需 query+加载 schema 省 token）| ❌（`deferred_tools_delta` 仅可用性变化通知）| ◀Q |

> 这三项都不来自 Gemini 上游，是 Qwen "做成 runtime 平台" 路线的自建。整合方向明确：以 Qwen 实现为准，Qoder 直接吸收。

---

## 七、被集成 / 北向接口（差距最大）

| 通道 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| HTTP daemon | ✅ `qwen serve` REST+SSE（1 daemon=1 workspace×N session）| ❌ | ◀Q |
| ACP | ✅ stdio + HTTP + WS（29 `_qwen/*` parity）| ⚠️ 仅 stdio（`acpMode`，Gemini 继承）| ◀Q（HTTP/WS）|
| 作 MCP server 被调 | ✅ `qwen-serve-bridge` | ❌（`qoder mcp` 无 serve）| ◀Q |
| SDK | ✅ TS/Python/Java | ❌（无公开 SDK）| ◀Q |
| Web/桌面客户端 | web-shell（React 组件）+ webui + 官方桌面 app | Qoder IDE（独立产品）| ◀Q（嵌入组件）|
| headless | `-p`/`--output-format`/`--json-schema`/`--bare` | `-p`/`--output-format`/`--headless` | ≈ |
| 远程控制 | daemon 多端 attach（开放协议）| **remote-control daemon**（`--remote`/`--teleport`/`/qr-code`，绑 Qoder 云）| ⊕ 开放协议为主，Qoder 云远程作产品层 |

> 这是整合中**最该以 Qwen 为基线**的子系统——Qwen 已是开放平台，Qoder 的远程能力是云闭环，可作为基于 Qwen daemon 的一种部署形态重做。

---

## 八、Hooks / 扩展 / 插件生态

| 维度 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| Hook 事件 | 10 事件 **Claude Code 命名**（`PreToolUse`/`PostToolUse`/`PostToolUseFailure`/`UserPromptSubmit`/`SessionStart`·`End`/`PreCompact`/`Stop`·`SubagentStop`/`Notification`）| 同 10 事件（计数 24~55×）| ≈ 两侧同构，配置可互用 |
| Hook 类型 | command + HTTP + Function + Async + SSRF 防护 | command + `/hook-config` | ◀Q Qwen hook 类型更全（17K 行）|
| 扩展机制 | extensions（Gemini 继承）+ `.qwen/commands` | **plugins + marketplace**（多源 + `.claude-plugin` 格式 + `claude-plugins-official` 源）| ⊕ Qwen extensions + Qoder 的 marketplace 多源 + Claude 插件市场互操作 |
| 子代理管理 | `/agents` create/manage | `/agents` + `qoder agent` + meta-skill `agent-creator`/`skill-creator` | ⊕ 吸收 Qoder meta-skill |

---

## 九、沙箱 / 安全 / telemetry

| 维度 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| macOS Seatbelt | `restrictive/permissive × closed/open/proxied`（旧矩阵）| `strict/restrictive/permissive × proxied/open`（新矩阵，含 **strict** 档）| ⊕ 合并矩阵，保留 strict + closed 全档 |
| 容器沙箱 | `--sandbox`/`--sandbox-image` | 未见 | ◀Q |
| telemetry | **OTel 深度分层 tracing** + 5 CLI 参数 + 阿里云 RUM | 自有端点（`/metrics` Prometheus、`/otel`）| ◀Q OTel 更深；Qoder 端点作可选 exporter |

---

## 十、会话 / 上下文 / 记忆 / Claude 兼容 / 招牌

| 维度 | Qwen Code | Qoder CLI | 整合建议 |
|---|---|---|---|
| 会话分支/回滚 | `/branch` `/fork` `/rewind` `/restore` + 跨会话快照 | `/branch` `/rewind` `/continue` | ⊕ 取并集 |
| 上下文压缩 | 三档梯度 auto-compaction（warn/auto/hard）+ `/compress-fast` | `/compact`（summarize/compress）| ◀Q 分层压缩更成熟 |
| 记忆 | 托管自动记忆（auto-dream/auto-skill + `/remember`·`/forget`·`/dream`）| `/memory` + `remember`/`SaveMemory` + `/manage` | ◀Q Qwen 托管记忆体系更完整 |
| Claude 迁移 | hook 命名兼容 | **`migrate --from-claude`** + CLI 参数面对齐（`--permission-mode`/`--output-style`/`--add-dir`）+ `.claude-plugin` | ▶O 引擎吸收迁移链 |
| Quest 规格驱动 | `workflow`（程序化）+ `enter_plan_mode` | **`quest`**（spec→确认→subagent 引导编排）| ▶O 引擎吸收引导式 spec UX |
| Arena 多模型竞争 | ✅ `/arena`（worktree 隔离竞争）| ❌ | ◀Q |
| 国际化 | 6 语言 UI | 未见多语言 | ◀Q |

---

## 十一、整合路线建议（汇总）

**以 Qwen Code 引擎为收敛基线**（开源 + 北向接口完整 + 运行时能力宽），叠加 **Qoder 的产品层与若干引擎增量**：

**收敛到 Qwen（◀Q，Qoder 吸收）**：被集成全栈（daemon/ACP HTTP-WS/MCP-server/SDK）、Dynamic Workflow、Computer Use、Tool Search、完整 Agent Team、cron/monitor、worktree 工具化、structured_output、三档压缩、托管记忆、Arena、6 语言 UI、深度 telemetry、4 级 agent 层级、LiveAgentPanel。

**Qwen 吸收 Qoder 引擎增量（▶O / ⊕）**：① 模型成本档位别名（6 档→映射层）；② per-agent `effort`+`timeoutMins`；③ ImageSearchTool；④ Tracker `Visualize` 任务可视化；⑤ 规则按 `subagent_type` 维度匹配；⑥ Plan/SaveMemory/skill-extractor 内置 agent；⑦ agent-creator/skill-creator meta-skill；⑧ marketplace 多源 + Claude 插件市场互操作；⑨ `migrate --from-claude` 迁移链 + Claude CLI 参数面；⑩ Quest 引导式 spec 编排；⑪ strict 沙箱档；⑫ subagentProgress recentActivity 活动流粒度；⑬ web_search 工具恢复。

**保留为 Qoder 产品层（▣，不进开源引擎）**：自营网关 + credit 计费 + 国内/全球双端点 + Qoder 账号体系 + 云端远程控制（基于 Qwen daemon 重做）。

**需统一的收敛工作项**：① 工具命名（Gemini 系 vs Claude 系 → 统一 + 别名兼容）；② 配置目录（`.qwen` vs `.qoder` → 统一 + 迁移）；③ 上下文文件（`QWEN.md` vs `AGENTS.md` → 多名兼容读取）；④ 命令去重（~25 同名命令对齐实现）；⑤ bundle 混淆（开源后去 `_$d()`）。

---

> **免责声明**：Qoder CLI 闭源，本文 Qoder 侧基于 v1.0.18 bundle 静态解码（确定性可复现），无源码；Qwen Code 侧基于 v0.18.0 源码。"整合建议"为基于能力成熟度的工程判断，非两团队决策。截至 2026-06-14。
