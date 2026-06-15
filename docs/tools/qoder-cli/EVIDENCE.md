# Qoder CLI 二进制分析证据

## ⚠️ v1.0.18 复核（2026-06-11）：Go 二进制 → Gemini CLI fork（**非** Qwen Code fork）

**重大变化**：Qoder CLI v1.0 已**从 Go 二进制完全重写为 JS bundle，基座是 Gemini CLI**。`@qoder-ai/qodercli` v1.0.18 npm 包静态分析 + 与 qwen-code / gemini-cli 源码三方对照：

- **包形态**：`bundle/qodercli.js`（34 MB esbuild bundle）+ `postinstall.cjs` 自述 "**pure JS bundle**"；copyright `Google LLC` / Apache-2.0（Gemini CLI 基座遗留版权），**不再是 Go 二进制**

- **是 Gemini CLI fork 的证据**（bundle 保留 Gemini CLI 原版内部名）：
  - `geminiIgnoreFilter`（Gemini CLI 类名；**Qwen Code 已改名为 `qwenIgnoreFilter`**——见下）
  - `[MemoryDiscovery] No GEMINI.md files found in hierarchy`（Gemini CLI 内存发现子系统日志）
  - `ContentGenerator` 26× / `web_fetch` 15× / `run_shell_command` 2× / `oauth2.googleapis` 10× / `cloudcode` 1×

- **不是 Qwen Code fork 的证据**（Qwen 专属标记在 bundle 中全部为 0）：

  | 标记 | 性质 | gemini-cli | qwen-code | **Qoder bundle** |
  |---|---|---:|---:|---:|
  | `geminiIgnoreFilter` | Gemini 原名 | 7 | 0 | **存在** |
  | `qwenIgnoreFilter` | Qwen 改名 | 0 | 5 | **0** |
  | `QWEN.md` | Qwen 上下文文件 | 0 | 211 | **0** |
  | `DashScope`/`dashscope` | Qwen provider | 0 | 164/124 | **0** |
  | `ModelScope`/`modelscope` | Qwen provider | 0 | 46/38 | **0** |
  | `chat.qwen.ai` | Qwen OAuth | 0 | 41 | **0** |
  | `qwenOAuth` | Qwen OAuth | 0 | 65 | **0** |
  | `QwenContentGenerator` | Qwen 生成器 | 0 | 24 | **0** |

  > **更正**：早前据 `arena`（11×）推断为 Qwen fork 有误——`arena` 实为 protobuf 选项 `cc_enable_arenas`/`ccEnableArenas`（Google Protocol Buffers 生成码），与 Qwen「Arena 多模型」无关；`replace` 千次多为 `String.prototype.replace`。已剔除。

- **Qoder 自有品牌**：`qoder` 747× / `.qoder` 131× / `api.qoder` 11×；配置 `.qoderignore` / `.qoder.json`；上下文文件用 **`AGENTS.md`**（+ `SKILL.md` / `MEMORY.md`）
- **macOS Seatbelt 沙箱（上游继承，非自研）**：6 个 `.sb` profile（`strict`/`restrictive`/`permissive` × `proxied`/`open`）随包分发于 `bundle/sandbox-macos-*.sb`，与 gemini-cli `packages/cli/src/utils/` 同名文件 **diff 仅 2 处**——`GEMINI_SANDBOX_PROXY_COMMAND`→`QODER_SANDBOX_PROXY_COMMAND`、注释 `--include-directories`→`--add-dir`
- **Qoder 自有**：浏览器登录 + PAT（`QODER_PERSONAL_ACCESS_TOKEN`）；自营网关 `api.qoder.sh`（全球）/ `api.qoder.com.cn`（国内）；`qoder-plugin` / `qoder-marketplace` / `qoder-enterprise-` / `remote-control`；credit 计费（`debug-credit-warning` 命令模拟 credit 耗尽告警）
- **依赖**：`sharp`（图像/多模态）+ ripgrep + Node ≥ 20；两渠道 `@latest` / `@beta`；`bundle/node_modules/` 残留 **`@google/gemini-cli-devtools`**（基座旁证）
- **字符串混淆**：大量字面量经 `_$d("Base64…")` 解码函数包裹，命令描述/工具 wire 名多数不可静态读取——以下清单为**可提取下界**

### v1.0.18 深度提取（2026-06-12）

**TUI slash 命令**（⚠️ 此处为早前"下界提取"，含假阴性/假阳性，**已被本文档末尾「命令清单：解码 `_$d()` 后的权威重提取」取代**——以那一节为准）：

- 明文 `name:"…",description:"…"` 注册：`/about` `/agents` `/corgi`（gemini 彩蛋）`/diff` `/feedback` `/help` `/hooks` `/login` `/marketplace` `/new` `/permissions` `/plugins` `/privacy` `/quit` `/reload` `/rewind` `/skills` `/status` `/theme` 等
- `name:"…"` 字符串命中（描述当时未解码）：`/mcp` `/model` `/memory` `/init` `/export` `/editor` `/docs` `/context` `/add-dir` `/copy` `/resume` `/tools` `/settings` `/usage` `/vim` `/statusline` `/setup-github` `/review` `/voice` `/goal` `/remote-control` 等
- ~~help 文本 `"/xxx"` 字符串~~：`/token` `/authorize` `/compare` `/metrics` `/otel` `/hook-config` 经复核为 **OAuth/Prometheus 端点或内部路由，非 slash 命令**（详末节更正表）

**工具类导出表**（esbuild re-export 明文，21 个）：`ReadFileTool` `WriteFileTool` `EditTool` `GlobTool` `RipGrepTool`（无独立 GrepTool/LSTool）`ShellTool` `WebFetchTool` `WebSearchTool` **`ImageSearchTool`**（gemini-cli 上游 0 命中，Qoder 新增）`AskUserTool` `ActivateSkillTool` `WriteTodosTool` `McpAuthTool` `TaskTool` `MemoryTool` + **Tracker 族 ×6**（`TrackerCreateTaskTool` / `TrackerUpdateTaskTool` / `TrackerGetTaskTool` / `TrackerListTasksTool` / `TrackerAddDependencyTool` / `TrackerVisualizeTool`，gemini-cli 上游已有）+ `DiscoveredMCPTool`

**CLI 子命令**（commander `("name").description(` 模式，30+）：`agent` `commit` `config`（get/set/unset/list）`doctor` `feedback` `hook` `install` `link` `list` `login` `mcp`（add/add-json/get/remove）`migrate` `mp`（marketplace）`plugin` `remote-control` `reset-project-choices` `rollback` `skill`（link/external）`status` `uninstall` `update`（`--to <version>`）`validate`

**启动参数**（commander `option()`，60+，节选 Claude Code 命名系）：`--permission-mode <mode>` `--output-style <style>` `--add-dir <dir>` `--settings <json>` `--setting-sources` `--strict-mcp-config` `--agents <json>` `--append-system-prompt` `--fork-session` `--session-id` `--allowed-tools` / `--disallowed-tools`；Qoder 特有：`--reasoning-effort <level>` `--context-window <size>` `--teleport <id>` `--remote [task]` `--remote-session <id>` `--attachment <file>` `--plugin-dir <dir>` `--list-sessions` `--delete-session <index>`

**Claude Code 兼容链**：

- `--with-claude-config`（v0.x Go 时代参数）在 v1.0 bundle 中 **0 命中，已移除**；取代者为 **`qoder migrate --from-claude`** 一次性迁移子命令（`("migrate").description(…).option("--from-claude",…)` 明文）
- marketplace 预置 **`claude-plugins-official`** 源 + 识别 **`.claude-plugin`** 目录格式
- hooks 事件全套采用 **Claude Code 命名**：`PreToolUse` 24× / `PostToolUse` 38× / `PostToolUseFailure` 17× / `UserPromptSubmit` 25× / `SessionStart` 55× / `SessionEnd` 16× / `PreCompact` 12× / `SubagentStop` 17×；gemini-cli 上游自有命名 `BeforeTool` / `AfterTool` / `BeforeAgent` / `AfterAgent` 在 bundle 中**全部为 0**（上游用 `hooks migrate` 做 Claude→Gemini 转换，源码 `packages/cli/src/commands/hooks/migrate.ts`；Qoder 反向直接原生采用 Claude 命名）
- 上下文用量 UI 把 memory 文件内部标记为 `claudeMd` 类型

**其他特性字符串**：`subagent` 98× / `voice` 110×（gemini-cli 上游有 voice 能力）/ `rewind` 34× / `teleport` 12× / `planMode` 23× + `plan_mode` 17× / `outputStyle` 41× / `statusline` 6× / `quest`（词边界）11× / ACP 101×

### fork 时点断代（2026-06-13）

**结论**：v1.0 代码基线取自 **2026-04 下旬的 gemini-cli（≈ v0.40.x，04-28/04-30 发布，或同期 main）**；fork/最后一次上游同步发生在 **04-24 ~ 05-04** 之间（确凿区间 04-17 ~ 05-04），距 v1.0.0 上线（05-19）约 3 周。

**方法**：在 bundle 中探测有确切上游合入日期的字符串。探针只用 minify/混淆后仍原样保留的字面量（settings key、esbuild 导出名、随包文件实物），不用类名与描述文案。

下界探针（存在 → fork 晚于该日）：

| 上游特征 | gemini-cli 合入 | bundle |
|---|---|---:|
| strict 沙箱矩阵 + 删除 closed 档（#18876）| 2026-02-12 | 6 个 .sb 逐字在 ✓ |
| Tracker 工具族（`trackerTools.ts`）| 2026-03-03 | 导出名在 ✓ |
| `ContextManager` | 2026-03-31 | 7× ✓ |
| **`autoMemory` 设置键**（#25601）| **2026-04-17** | **17× ✓（最强确凿下界）** |
| `/voice` 命令（`voiceCommand.ts`，#24174）| **2026-04-24** | `/voice` + `off` 子命令在，voice 110×（高置信下界）|

上界探针（缺失 → fork 早于该日）：

| 上游特征 | gemini-cli 合入 | bundle |
|---|---|---:|
| **`ignoreLocalEnv` 设置键**（#26445）| **2026-05-04** | **0（最强上界）** |
| auto-memory inbox 字面量（`auto-memory-contract` / `extraction.patch` / `memory-patch`，#26338）| 2026-05-04 | 0 |
| `agentSessionSubagentEnabled`（05-17）/ `tui-tester`（05-18）/ `Local/RemoteSessionInvocation`（05-18/19）| 05 中下旬 | 0（辅助）|

**可信度要点**：`autoMemory`（04-17，存在）与 `ignoreLocalEnv`（05-04，缺失）是 settingsSchema 中同类型配置键，一存一缺基本排除"选择性删除"；且 autoMemory 体系在 bundle 中完整存在，排除"整个 memory 子系统被砍"的误判。`/voice` 下界置信度略低（理论可自研，但 fork 重写背景下概率低），保守表述为"确凿晚于 04-17，很可能晚于 04-24"。

**外部佐证**（npm 发布史）：Go 版 0.x 持续发版至 **05-18**，**1.0.0 于 05-19 上线**——4 月底 fork → 约 3 周品牌化改造（`qoder` 改名 747 处、自营网关/PAT/credit、`_$d()` 混淆、.sb 环境变量改名）→ 上线；期间 Go 版与 JS 版并行维护约一个月，说明换底座是计划内动作。版本夹逼：v0.39.0（04-23）尚无 /voice，v0.41.0（05-05）已有 `ignoreLocalEnv`——**v0.40.0/v0.40.1（04-28/04-30）恰落窗口正中**。

> 注：以上日期为上游 main 合入日；若 Qoder 曾持续 rebase，测得的是"最后一次同步点"而非"首次 fork 点"，但对"代码基线属于哪个时期"答案相同。

### 命令清单：解码 `_$d()` 后的权威重提取（2026-06-13 复核）

**背景更正**：早前的命令清单是"下界提取"——只抓 `name:"x",description:"明文"` 的对象，因此**漏掉了所有描述被混淆的命令**（假阴性），又把 help/端点文本里的 `"/xxx"` 误当命令（假阳性）。本次解出混淆函数后重做，得到权威清单。

**`_$d()` 反混淆**（bundle 内函数原文）：

```js
_$d = (s, k="tBpirNfrja2H") => {
  const b = Buffer.from(s, "base64");
  for (let i=0; i<b.length; i++) b[i] ^= k.charCodeAt(i % k.length);
  return b.toString();
};
```

即 base64 解码后用固定密钥 `tBpirNfrja2H` 循环异或。命令描述形如 `description:_$d("NTEbSRNu…")`。

**提取法**：锚定 `kind:"built-in"` 反向找最近 `name:"X"`（兼容 `altNames`/模板字符串 ``description:`...` `` 等变体），另抓 `registerBundledSkill({name:"X"})`。

**内置 slash 命令（kind:"built-in"，78 个 name 含子命令；顶层 ≈ 50）**——解码后确认存在的代表项（含别名）：

`/about` `/add-dir` `/agents`(agent) `/branch` `/btw`（"Ask a quick side question without interrupting the main conversation"）`/clear` `/commands` `/compact`(summarize,compress) `/context` `/continue` `/copy` `/corgi` `/diff` `/docs` `/editor` `/export` `/feedback`(bug) `/help` `/hooks` `/init` `/insights`（分析 Qoder 会话生成报告）`/kanban`（看板后端）`/login`(signin) `/logout`(signout) `/marketplace`(market) `/mcp` `/memory` `/model` `/new` `/permissions` `/plan`（Toggle Plan Mode）`/plugins`(plugin) `/privacy` `/profile` `/qr-code`（远程会话扫码）`/quit`(exit) `/release-notes` `/remote-control` `/remote-env`（云端远程环境）`/rename` `/resume` `/review` `/rewind`(checkpoint) `/settings`(config) `/setup-github` `/shortcuts` `/skills` `/statusline` `/tasks`(bg,background，"Show background tasks panel") `/theme` `/tools` `/usage` `/vim` `/voice`；插件子命令 install/uninstall/enable/disable/update/validate；`/goal`（独立注册，子命令 `set|status|clear|pause|resume|take`）

**Bundled skills（`registerBundledSkill`，8 个）**：`batch` `debug` `loop` `mcp-config` `quest` `remember` `security-review` `simplify`

**关键更正**：

| 命令 | 早前结论 | 解码后真相 |
|---|---|---|
| `/btw` | ❌ 标"—"（Qwen 独有）| **存在**，描述与 Qwen `/btw` 几乎一字不差 |
| `/branch` `/rename` `/plan` `/compact` `/tasks` `/settings` `/tools` `/setup-github` `/export` `/goal` | ❌ 标"—"或弱证据 | **均存在**（描述曾被 `_$d()` 混淆而漏判）|
| `/loop` `/batch` `/simplify` `/remember` | ❌ 标"未见 bundled 命令" | **存在**（bundled skills，与 Qwen 同名）|
| `/token` `/authorize` `/compare` `/metrics` `/otel` `/hook-config` | ⚠️ 曾误列为命令 | **非命令**：`/token`/`/authorize` 是 OAuth 端点 URL、`/metrics` 是 Prometheus 端点、`/hook-config` 是内部路由（真命令是 `/hooks`）|

**Qwen 有、Qoder 确实没有**（name 计数全 0，确认）：`/recap` `/arena` `/insight`（单数；Qoder 是 `/insights` 会话分析，含义不同）`/language` `/terminal-setup` `/ide` `/lsp` `/trust` `/approval-mode` `/dream` `/forget` `/summary`（但 `/compact` 带 summarize 别名）。

### SubAgent 系统（仔细调研，2026-06-14 复核）

**反框架发现**：subagent 体系**不是 Gemini CLI 基座带来的**——上游 gemini-cli 无 `tools/agent/`、无 `subagents/`、无 `subagent_type`/fork/内置 agent 注册表（逐一 grep 确认）。**Qwen 与 Qoder 各自在 Gemini 基座上叠了一套 Claude Code 式 agent 系统**：内置 agent 描述文案在两侧**逐字相同**且是 Claude Code 原文。

**Qoder 内置 agent（`kind:"local"`，解码 8 个）**：

| agent | 描述（解码） |
|---|---|
| `general-purpose` | General-purpose agent for researching complex questions… |
| `Explore` | Fast agent specialized for exploring codebases…（与 Qwen 逐字相同）|
| `Plan` | Software architect agent for designing implementation plans（Claude 的 Plan agent）|
| `fork` | **Implicit fork — inherits full conversation context. Not selectable via subagent_type…**（与 Qwen FORK_AGENT 逐字相同）|
| `SaveMemory` | Writes and reads memory, preferences or facts across ALL future… |
| `statusline-setup` | Use this agent to configure the user's custom status line… |
| `skill-fork` | Forked skill execution |
| `skill-extractor` | Extracts reusable skills from past conversation sessions… |

对比 Qwen 内置 4 个（`general-purpose`/`Explore`(`model:'fast'`)/`statusline-setup`/`fork`）——**Qoder 反而更多**（多 Plan/SaveMemory/skill-extractor）。

**Task 工具（subagent，`task`/`delegate`）参数（解码描述）**：
- `subagent_type`："The type of specialized agent to use for this task"，默认 `general-purpose`；触发逻辑 `subagent_type ?? (WCe()?void 0:"general-purpose")`——省略且 fork 开启即隐式 fork（**与 Qwen 逐字相同**）
- `isolation`：`enum["default","worktree"]`，"Use worktree to force execution in an isolated git worktree"
- `run_in_background`："notified when it completes"
- `description`："A short (3-5 word) description"
- 权限按 agent 维度匹配：`Sri(rules, name, subagent_type)`

**多代理协作（teammate）——两家都有，深浅不同**：
- Qoder：`teammate_mailbox`(4×，渲染 `{from:agentName, text:result, timestamp, state}`)、`teammateName`(6)、`teammate_shutdown_batch`(1)——**仅并行结果聚合**，无生命周期事件、无 broadcast、无 `team_*` 工具、无持久任务板（`createTeam` 1× 实为 protobuf 枚举假阳性）
- Qwen（对照）：`teammate_message`(35) + 生命周期 `teammate_joined`/`_idle`/`_exited`/`_status`/`_approval` + broadcast + 共享任务板 `~/.qwen/tasks/{team}` + `team_create`/`team_delete`/`task_*` 工具——**完整 Agent Team**

**进度展示（subagentProgress + /tasks 面板）**：

```
{ isSubagentProgress:true, agentName, state, recentActivity:[{id,type,content,status}] }
```
- 状态机：`pending`(49) → `queued`(8) → `running`(87) / `in_progress`(23) → `completed`(97) / `error`
- `recentActivity`：type ∈ `thought`/`action`/`tool_call`/`tool_result`，运行中 `{type:"thought",content:"Working...",status:"running"}`
- 渲染：树形盒绘 `├ └ │ ⎿` + `elapsed`/`duration` 计时
- `/tasks`（`bg`/`background`）：`custom_dialog` "background tasks panel"，跟踪后台 subagent + shell
- **无常驻并行面板**：未见 Qwen `LiveAgentPanel`/`InlineParallelAgentsDisplay`；并行 subagent 结果汇入 `teammate_mailbox` 视图

> **更正记录**：早前（06-13）误判"Qoder 无 fork / 无团队层 / 内置 agent 仅 2 个"。仔细解码后：fork **两家都有**（逐字相同）、teammate **两家都有**（Qoder 仅 mailbox 聚合）、Qoder 内置 agent **8 个**（比 Qwen 多）。提取时间 2026-06-14，描述经 `_$d()` 确定性解码。

### Dynamic Workflow 与 Computer Use（2026-06-14 核查）

bundle 全量字符串核查，**两项 Qoder 均无**（Qwen 两项皆有）：

- **Dynamic Workflow**：`WorkflowOrchestrator` / `pipeline(` / `phase(` / `orchestrat` / `fan-out` **全 0**；`workflow` 4× 仅为 MIME 类型 `application/vnd.oai.workflows` + 遥测字段 `workflow_name`，非编排工具。（Qwen：`tools/workflow/` + `agents/runtime/WorkflowOrchestrator`，注入 JS 脚本 `phase`/`agent`/`parallel`/`pipeline` 确定性 fan-out，单 run ≤1000 `agent()`）
- **Computer Use**：`computer_use` / `computer-use` / `ComputerUse` / `computerUse` / `list_apps` / `get_app_state` / `press_key` **全 0**；`screenshot`(1) / `type_text`(1) 命中均在语法高亮词表内，非工具。（Qwen：`tools/computer-use/` 9 工具 `computer_use__*`，`computerUseEnabled` 默认 true + deferred 揭示，首次用安装上游二进制 + macOS 权限探测）

完整对比见 [Qwen Code vs Qoder CLI §5.1](../../comparison/qwen-code-vs-qoder-cli.md#51-dynamic-workflow-与-computer-use-专项)。

### Follow-up / Prompt Suggestion —— Qoder 支持（2026-06-14 核查）

Qoder **实现了**轮后"建议下一步 prompt"（= Claude Code 的 PromptSuggestion / Qwen 的 `followup_suggestion`），**内部命名与 Qwen 相同**：

- **子系统**：`[PromptSuggestion]` 日志全套——`Failed to generate suggestion` / `Raw suggestion output` / `Rejected suggestion`（经 `Xbn()` 校验拒绝劣质）/ `Accepted suggestion`（模型生成，带 `promptId`/`sessionId`）。
- **事件**：`{type:"prompt_suggestion", suggestion:string, uuid, sessionId}`，内部归类为 `agent_summary`。
- **同族 fast-model 辅助生成**：`prompt_suggestion`(6×) + `awareness_nudge`(2×，类 nudge) + `generate_session_title`/`title_generation`（自动会话标题）。
- **对照**：Qwen `followup_suggestion`(42×)/`prompt_suggestion`(15×)/`PromptSuggestion`(32×)——同名同源（Claude Code）。
- **甄别**：`suggestions:joi(...)`/`suggestions:[...]` 为**权限建议**（"Allow X?" 的 allow/deny 选项），`follow-up`(1×) 为反馈表单"Email for follow-up"，均非此特性。

**但无 Speculation 预执行**：`speculat`/`speculative`/`ghostText`/`Tab to accept` **全 0**——Qoder 只生成建议，不投机预执行（对比 Qwen PR#2525 含预执行）。即**借了 PromptSuggestion 的"建议半"，未借"预执行半"**。

### Mid-Turn Queue Drain（中途注入 / steering）—— Qoder 支持（2026-06-14 核查）

Qoder **实现了**中途输入注入（= Claude Code 的 steering / Qwen 的 `drainQueue`），命名为 **steering**：

- **机制**：`InjectionService`——用户在 agent 运行中提交输入 → `addInjection(input, "user_steering")` 排队（带 `isEnabled()` 开关）；轮次循环用 `getInjectionsAfter(this.startIndex, "user_steering")` 捞取**开始后**排队的注入并消费。
- **注入格式**：`tmi(A)` 包成 `<user_input>…</user_input>`；`nmi(A)` 生成 `"User steering update:\n<user_input>…"`；`omi(A)` 生成 `"User hints:\n…"`；确认 `promptId:"steering-ack"`。
- **UI**：React hook 监听 `user_steering` 注入（`injectionListeners`），输入触发 `addInjection(i,"user_steering")` + `{type:"hint",text:i}`。
- **甄别**：`drain`(5)/`queued`(4) 命中为 Node socket/stream backpressure（`writableNeedDrain`/`kestrel.queued_connections`），`pendingMessage`(3) 为 gRPC 协议 listener，均非用户输入队列——真机制是 `user_steering` 注入。

**结论**：Qoder ✅ 支持中途注入（steering）。与 Qwen `drainQueue`、Claude Code steering **能力等价、命名不同**。早前对标文档误标"未见"，已更正。

### Tool Search（工具延迟加载）—— Qoder 不支持（2026-06-14 核查）

- **无模型可调用的 search 工具**：`tool_search` / `ToolSearchTool` / `search_tools` / `load_tool` / `discover_tools`（工具名）**全 0**；工具导出表仅 `DiscoveredMCPTool` / `DiscoveredTool` / `ImageSearchTool` / `WebSearchTool`，无延迟加载工具。
- **有但不同机制**：`deferred_tools_delta`（1×）渲染 `"Tools now available"` / `"Tools no longer available"`（与 `mcp_instructions_delta` 同类的 transcript delta）+ `deferred_tool_use`（schema 可选字段）。这是**工具可用性变化的推送通知**，非模型主动 query+加载延迟 schema。
- **血脉**：`deferred_tools_delta` 在 Qwen 源码与 gemini-cli 上游均无 → Qoder 自有；`tool_search` 是 Qwen 自加（上游无）。
- **影响**：Qoder 工具**全量注册进 system prompt**，无 Qwen `tool_search` 那种把大量 MCP/外部工具 deferred 起来省 token 的能力。

对比见 [Qwen Code vs Qoder CLI §5.2](../../comparison/qwen-code-vs-qoder-cli.md#52-tool-search工具延迟加载)。

### 被集成 / 北向接口（2026-06-13）

Qoder 作为"被集成对象"的能力，bundle 实证：

- **ACP**：仅 stdio——`--acp` flag + 配置项 `acpMode`（默认 false）+ `acpAuthenticatedType` + `acp-session`，从 Gemini CLI 继承。**无 HTTP/WS ACP**（对比 Qwen `/acp` HTTP+WS）。
- **MCP**：**仅客户端**——`qoder mcp` 子命令为 `add` / `add-json` / `get` / `list` / `remove` / `auth`，**无 `serve`**。即 Qoder 不能被当 MCP server 调用（对比 Qwen `qwen-serve-bridge`）。
- **SDK**：**无公开 SDK 包**。`bundle/` 下唯一 `.d.ts` 是残留的 `@google/gemini-cli-devtools`。
- **headless**：`-p/--print`（commander `option("-p, --print")`）+ `--output-format` + `--input-format`（含 `stream-json`，27×）+ `--headless`（3×）。
- **远程控制**（Qoder 自有，解码描述）：`/remote-control`="Start the remote-control daemon"；`--remote-control`="Run as a headless remote-control worker"；`--remote`="Create a cloud remote session and print its access URL"；`--teleport`="Load and attach to a remote session"；`--remote-session`="Cold-load and attach to an existing remote session"；`/remote-env`="Choose the default cloud remote environment"；`/qr-code`="Print a scannable QR for the current remote session"；`/start`="Connect the current session to a remote endpoint"。→ 一套**绑定 Qoder 云、由 web/mobile 端控制**的闭环，非开放第三方 API。
- **甄别**：bundle 内 `zed` 713× 经核为**内部状态库**（`zedState` 184× / `zedProps` 61× / `zedUserClient` / `zedTranscript`），`zed-acp`/`zedAcp`/`zed editor` 均为 0——**不是 Zed 编辑器集成**（IDE 集成走标准 ACP stdio）。

**结论**：Qoder 被集成面 = headless CLI + 标准 ACP stdio + Qoder 云远程控制；**无开放 HTTP daemon / 无 MCP-server / 无 SDK**。对比见 [Qwen Code vs Qoder CLI §七 被集成能力](../../comparison/qwen-code-vs-qoder-cli.md#七被集成能力作为服务端--可嵌入对象)。

**结论**：Qoder CLI v1.0 与 Qwen Code 是 **Gemini CLI 的两个兄弟 fork**（均阿里系），但 **Qoder 直接 fork 自 Gemini CLI，未经过 Qwen Code**。完整对比见 [Qwen Code vs Qoder CLI](../../comparison/qwen-code-vs-qoder-cli.md)。

---

## 基本信息（v0.x 历史 — Go 二进制时代，v1.0 已重写取代）
- 包: @qoder-ai/qodercli v0.1.35
- 二进制: /usr/local/lib/node_modules/@qoder-ai/qodercli/bin/qodercli
- 格式: ELF 64-bit LSB executable, x86-64, statically linked, stripped
- 大小: 43 MB
- 语言: Go
- 内部包根: code.alibaba-inc.com/qoder-core/qodercli/

## CLI --help 完整输出
```
Qoder is a powerful terminal-based AI assistant that helps with software development tasks.

Usage:
  qodercli [flags]
  qodercli [command]

Job Management Commands:
  jobs         List concurrent job(s)
  rm           Remove concurrent job(s)

Additional Commands:
  commit       Commit AI-generated code and record AI contribution statistics
  completion   Generate the autocompletion script for the specified shell
  feedback     Submit feedback with optional images
  help         Help about any command
  install      Install qodercli to the standard location
  mcp          Manage MCP (Model Context Protocol) servers
  status       Show account and CLI status
  update       Check remote version and self-update to the latest release

Flags:
  --agents, --allowed-tools, --attachment, --branch, -c/--continue,
  --dangerously-skip-permissions, --disallowed-tools, --experimental-mcp-load,
  --input-format, --max-output-tokens, --max-turns, --model, -f/--output-format,
  --path, -p/--print, -q/--quiet, -r/--resume, --summarize-tool,
  -v/--version, --with-claude-config, -w/--workspace, --worktree, --yolo
```

## MCP --help 输出
```
Available Commands: add, auth, get, list, remove
```

## status 输出
```
Version: 0.1.35
Username: [redacted]
Email: [redacted]@alibaba-inc.com
User Type: personal_standard
Plan: Free
```

## 内部包结构（30+ 包）
acp/, cmd/(commit/endpoint/feedback/install/jobs/login/mcp/status/update),
core/(agent/auth/config/mcp/permission/rule), tui/(components/event/render/state/styles/theme),
packages/

## SDK 引用计数
openai-go: 11,464 | anthropic-sdk-go: 3,901 | mcp-go: 744 | cobra: 1,009

## API 端点
api1.qoder.sh, api3.qoder.sh, center.qoder.sh, openapi.qoder.sh,
download.qoder.com, docs.qoder.com, forum.qoder.com,
api.openai.com (BYOK), dashscope.aliyuncs.com (BYOK)

## Machine ID
getMachineKey 函数存在于二进制中（设备标识）

## 10 个内置主题
catppuccin, dracula, flexoki, gruvbox, monokai, onedark, qoder, tokyonight, tron + default

## 提取时间
2026-03-25

## /review 反编译分析

### /review-code 和 /review-pr
- 类型: Skill（非内置命令）
- 系统提示: "When users reference '/review-pr', they are referring to a skill"
- 调用方式: skill: "review-pr", args: "123"

### isSpecReviewScenario
- 位置: core/agent/provider.(*qoderClient).isSpecReviewScenario
- review 被识别为 Quest 特殊场景
- 独立的模型选择逻辑

### 模板系统
- 错误: "failed to load code review template"
- review 使用服务端模板（非硬编码 prompt）
- 可热更新（无需客户端升级）

### GitHub Action
```yaml
- name: Run Qoder Code Review
  uses: QoderAI/qoder-action@v0
  with:
    prompt: |
      /review-pr
      REPO:owner/repo PR_NUMBER:123
      OUTPUT_LANGUAGE:English
```

### 独有能力
- 双命令分离（/review-code + /review-pr）
- OUTPUT_LANGUAGE 多语言输出
- 服务端模板热更新
