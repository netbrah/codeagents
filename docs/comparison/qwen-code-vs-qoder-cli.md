# Qwen Code vs Qoder CLI 对比

> **核心结论（2026-06 复核）**：Qoder CLI **v1.0 已从 v0.x 的 Go 二进制完全重写为 JS bundle，基座是 Gemini CLI**。它与 Qwen Code 是 **Gemini CLI 的两个兄弟 fork**（均为阿里系产品），但 **Qoder 不是 Qwen Code 的 fork**——经 gemini-cli / qwen-code / Qoder bundle 三方字符串三角验证：Qoder 保留的是 **Gemini CLI 原版内部名**（`geminiIgnoreFilter`、`GEMINI.md`、`MemoryDiscovery`），而 Qwen Code 专属标记（`DashScope` / `ModelScope` / `chat.qwen.ai` / `qwenOAuth` / `QwenContentGenerator` / `QWEN.md`）在 Qoder bundle 中**全部为 0**。
>
> 证据：本地分析 `@qoder-ai/qodercli` v1.0.18（2026-06-11 npm 最新版）的 34 MB esbuild bundle，与 qwen-code / gemini-cli 源码三方对照（详细提取记录见 [Qoder CLI EVIDENCE](../tools/qoder-cli/EVIDENCE.md)）。
>
> **方法论提示**：Qoder bundle 对字符串字面量做了混淆（`_$d(base64)` = base64 解码后用密钥 `tBpirNfrja2H` 循环异或）。**命令名是明文字面量**（可靠），**描述被 `_$d()` 混淆**。本文命令清单已**解码 `_$d()` 后重做**（锚定 `kind:"built-in"` 提取 + `registerBundledSkill`），是权威清单——早前依赖明文 name+desc 的"下界提取"曾产生假阴性（漏 `/btw` `/compact` `/tasks` 等）与假阳性（把 OAuth/Prometheus 端点 `/token` `/metrics` 误当命令），现已更正。工具/参数侧的 wire 名仍部分混淆，标注下界。详见 [EVIDENCE：命令清单解码重提取](../tools/qoder-cli/EVIDENCE.md#命令清单解码-_d-后的权威重提取2026-06-13复核)。

## TL;DR

| | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 开发方 | 阿里云（Qwen / 通义团队）| 阿里（Qoder，v0.x 内部包 `code.alibaba-inc.com/qoder-core`）|
| 血脉 | **Gemini CLI fork** | **Gemini CLI fork**（v1.0；基线 ≈ 2026-04 下旬 v0.40.x，见[断代](#fork-时点2026-年-4-月下旬基线--gemini-cli-v040x)；v0.x 曾是独立 Go 二进制）|
| 与对方关系 | — | **不是 Qwen Code 的 fork**，是 Gemini CLI 的兄弟 fork |
| 许可 | **Apache-2.0 开源** | **闭源专有**（bundle 内 Apache-2.0/Google LLC 为 Gemini CLI 基座遗留版权）|
| 分发 | npm JS（TypeScript）| **v1.0 npm JS**（34 MB esbuild bundle；v0.x 为 Go ELF）|
| 模型/认证 | Qwen + 阿里云 Coding Plan 订阅 + 任意 OpenAI·Anthropic 兼容 provider（BYOK，含 DeepSeek 深度适配）| **自营模型网关**（`api.qoder.sh` 全球 / `api.qoder.com.cn` 国内）聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek；浏览器登录 + PAT |
| 价格 | **开源软件免费，但模型调用需自付**（Qwen OAuth 免费层 **2026-04-15 已停**）| 闭源商业产品，Qoder 账号统一计费（bundle 含 credit 耗尽告警逻辑）|
| 招牌 | **daemon Mode B 多端 runtime** + Arena 多模型 + Agent Team/cron/workflow 工具族 + telemetry | **自营网关统一计费** + plugins/marketplace 生态 + **Claude Code 式 CLI/hooks 表面** + 国内/全球双端点 |
| 被集成 | **开放平台**：HTTP daemon + ACP(stdio/HTTP/WS) + MCP-server 桥 + TS/Python/Java SDK（"别人来集成我"）| headless + ACP stdio + **Qoder 云远程控制闭环**（"我的云来控制我"）|

## 一、血脉：两个兄弟 fork，互不衍生

### v0.x → v1.0 的彻底重写

| | Qoder v0.1.35（2026-03）| Qoder v1.0.18（2026-06-11）|
|---|---|---|
| 语言 | **Go**（静态链接 43 MB ELF）| **TypeScript/JS**（34 MB esbuild bundle）|
| 内部包 | `code.alibaba-inc.com/qoder-core/qodercli/` | **Gemini CLI 基座** |
| 架构 | 自研 Go 二进制（用 openai-go / anthropic-sdk-go）| **fork 自 Gemini CLI** |

v0.x 是阿里 Qoder 团队**自研的 Go 实现**，与 Gemini CLI / Qwen Code 无代码关系。v1.0 是一次推倒重来，改用 **Gemini CLI 的 TypeScript 基座**重写。

### 三方三角验证：基座是 Gemini CLI，不是 Qwen Code

判别方法：Qwen Code 自身也是 Gemini CLI 的 fork，它把 Gemini 的若干标识**重命名**了（`GEMINI.md`→`QWEN.md`、`geminiIgnoreFilter`→`qwenIgnoreFilter`），并新增了 DashScope/ModelScope/Qwen OAuth 等专属代码。因此只要看 Qoder bundle 里留的是 **Gemini 原名** 还是 **Qwen 改名/新增**，就能判定它从哪一支 fork。

字符串出现次数（gemini-cli 源码 / qwen-code 源码 / Qoder v1.0.18 bundle）：

| 标记 | 性质 | gemini-cli | qwen-code | **Qoder bundle** |
|---|---|---:|---:|---:|
| `geminiIgnoreFilter` | Gemini CLI 原版类名（Qwen 已改名）| 7 | **0** | **存在** ✅ |
| `GEMINI.md` / `MemoryDiscovery` | Gemini CLI 内存发现子系统 | 203 / 18 | 9 / — | **存在**（日志 "No GEMINI.md files found"）✅ |
| `ContentGenerator` | Gemini CLI 模型抽象 | 多 | 多 | 26 ✅ |
| `web_fetch` / `run_shell_command` | Gemini CLI 工具名 | 多 | 多 | 15 / 2 ✅ |
| `oauth2.googleapis` / `cloudcode` | Gemini CLI / Google 残留 | 多 | 多 | 10 / 1 ✅ |
| `qwenIgnoreFilter` | **Qwen 改名版** | 0 | 5 | **0** ❌ |
| `QWEN.md` | **Qwen 改名版上下文文件** | 0 | 211 | **0** ❌ |
| `DashScope` / `dashscope` | **Qwen 专属 provider** | 0 | 164 / 124 | **0** ❌ |
| `ModelScope` / `modelscope` | **Qwen 专属 provider** | 0 | 46 / 38 | **0** ❌ |
| `chat.qwen.ai` | **Qwen OAuth 端点** | 0 | 41 | **0** ❌ |
| `qwenOAuth` | **Qwen OAuth 逻辑** | 0 | 65 | **0** ❌ |
| `QwenContentGenerator` | **Qwen 专属生成器** | 0 | 24 | **0** ❌ |

**判读**：

- Qoder bundle 保留了 **Gemini CLI 的原版内部名**，这些在 Qwen Code 里已被改掉——**说明 Qoder 直接从 Gemini CLI fork，而非经过 Qwen Code**。
- **所有 Qwen 专属标记在 Qoder bundle 中均为 0**。这些是字符串字面量，不会被 esbuild minify 改写，**为 0 即真实缺失**。
- Qoder 自有品牌密度极高：`qoder` 747× / `.qoder` 131× / `api.qoder` 11×，配置文件 `.qoderignore` / `.qoder.json`，上下文文件用 **`AGENTS.md`**（+ `SKILL.md` / `MEMORY.md`）。

进一步的基座旁证（v1.0.18 包内实物）：

- **`/corgi` 彩蛋**：Gemini CLI 的招牌 corgi mode 以明文注册保留在 Qoder bundle 中（`name:"corgi",description:"Toggles corgi mode"`）
- **`@google/gemini-cli-devtools`**：npm 包 `bundle/node_modules/` 下残留 Google 官方 devtools 子包
- **6 个 macOS Seatbelt profile 逐字继承**：`sandbox-macos-{strict,restrictive,permissive}-{proxied,open}.sb` 与 gemini-cli 仓库同名文件 diff 仅 2 处——`GEMINI_SANDBOX_PROXY_COMMAND`→`QODER_SANDBOX_PROXY_COMMAND`、注释里 `--include-directories`→`--add-dir`

### fork 时点：2026 年 4 月下旬（基线 ≈ gemini-cli v0.40.x）

用"特征断代夹逼"可以进一步定出 fork 时间——在 bundle 中探测有确切上游合入日期的字面量（settings key / 导出名 / 随包文件，均不受 minify 与 `_$d()` 混淆影响）：

| 方向 | 探针 | gemini-cli 合入 | bundle |
|---|---|---|---:|
| 下界（存在）| strict 沙箱矩阵（#18876）| 2026-02-12 | ✓ |
| 下界（存在）| Tracker 工具族 | 2026-03-03 | ✓ |
| 下界（存在）| **`autoMemory` 设置键**（#25601）| **2026-04-17** | **17×** |
| 下界（存在）| `/voice` 命令（#24174）| **2026-04-24** | ✓ |
| 上界（缺失）| **`ignoreLocalEnv` 设置键**（#26445）| **2026-05-04** | **0** |
| 上界（缺失）| auto-memory inbox 字面量（#26338）| 2026-05-04 | 0 |

`autoMemory`（在）与 `ignoreLocalEnv`（缺）是 settingsSchema 里同类配置键，一存一缺基本排除选择性删除。结合 npm 发布史——Go 版 0.x 发版至 05-18、**1.0.0 于 05-19 上线**——可推断：**fork/最后一次上游同步发生在 04-24 ~ 05-04 之间（确凿区间 04-17 起），基线约为 gemini-cli v0.40.x（04-28/04-30），随后用约 3 周完成品牌化改造上线 v1.0**；期间 Go 版与 JS 版并行维护约一个月，说明换底座是计划内动作。完整探针记录见 [EVIDENCE：fork 时点断代](../tools/qoder-cli/EVIDENCE.md#fork-时点断代2026-06-13)。

> 注：`arena` 字符串（11×）曾被误读为 Qwen「Arena 多模型」痕迹，实为 protobuf 生成码的 `cc_enable_arenas` 选项，不构成 fork 证据。

## 二、共享的底座（都来自 Gemini CLI）

两者都是 Gemini CLI 的 fork，以下能力同源——这是**共同祖先**带来的，不是谁 fork 了谁：

- **ContentGenerator** 模型抽象 + 多 provider 接入
- **内置工具**：read/write/edit/shell/glob/ripgrep/web_fetch + Todo + subagent（详见 [五、内置工具对比](#五内置工具对比)）
- **MCP** 协议扩展（`--mcp-config` / `/mcp` / MCP server 管理）
- **subagent** 子代理（Qoder bundle 内 `subagent` 98×、`SubagentStop` hook 17×）
- **skills**（SKILL.md）+ **slash 命令系统** + 会话 checkpoint/resume
- **macOS Seatbelt 沙箱 profile**（两家都从上游带出，矩阵略有差异，见 [十、关键分歧](#十关键分歧)）
- **voice mode**（gemini-cli 上游能力；Qoder bundle `voice` 110×，Qwen Code 未启用为命令）
- **非交互模式**（`-p`）+ `--output-format` + `--worktree` + ACP 编辑器协议

## 三、内置 slash 命令对比

- **Qwen Code**：**57 个内置命令**（v0.18.0：52 个常驻 + 5 个条件启用 `/ide` `/trust` `/lsp` `/dream` `/forget`；源码 `packages/cli/src/services/BuiltinCommandLoader.ts`），另有 **7 个 bundled skill 命令**（`/review` `/loop` `/batch` `/simplify` `/stuck` `/new-app` `/qc-helper`，源码 `packages/core/src/skills/bundled/`）+ MCP prompts + `.qwen/commands/*.md` 自定义命令。
- **Qoder CLI**：解码 `_$d()` 后提取到 **78 个 `kind:"built-in"` 命令名**（含子命令，顶层 ≈ 50）+ **8 个 bundled skills**（`batch` `debug` `loop` `mcp-config` `quest` `remember` `security-review` `simplify`）+ `/goal`（独立注册）。下表描述均为解码后原文。

### 命令对照（按功能分组）

> ✓ = 解码确认存在（名称明文 + 描述 `_$d()` 解码）；— = name 计数为 0，确认不存在。Qoder 描述为解码后原文节选。

**会话与上下文**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 新对话 / 清屏 | `/clear` | `/clear` + `/new` ✓ |
| 压缩上下文 | `/compress` + `/compress-fast`（无 LLM 规则式）| **`/compact`**（别名 `summarize` `compress`）✓ |
| token 用量 | `/context` | `/context`（"Visualize current context window usage"）✓ |
| 继续会话 | `/resume` | `/resume` + **`/continue`**（继续本项目最近会话）✓ |
| 检查点回滚 | `/restore` + `/rewind` | `/rewind`（别名 `checkpoint`）✓ |
| 会话分支 | `/branch`（fork 当前对话到新会话）| **`/branch`**（"Create a new session branch from the current conversation"）✓ |
| 会话改名/删除/回顾 | `/rename` `/delete` `/recap` | **`/rename`** ✓；`/delete` `/recap` — |
| 导出会话 | `/export`（html/md/json/jsonl）| `/export`（"Export the current conversation to a file or clipboard"）✓ |
| 复制回复 | `/copy` | `/copy`（"Copy the last assistant response… /copy N for Nth"）✓ |
| 会话摘要 | `/summary` | —（并入 `/compact`）|
| 退出 | `/quit`（`exit`）| `/quit`（`exit`）✓ |

**配置与系统**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 帮助 | `/help`（`?`）| `/help` ✓ + `/shortcuts`（快捷键面板）|
| 版本信息 | `/status`（`about`）| `/about` ✓ + `/status` ✓ + `/release-notes` ✓ |
| 登录认证 | `/auth`（`login`）| `/login`（`signin`）+ `/logout`（`signout`）✓ |
| 切换模型 | `/model` | `/model`（"Set or manage model configuration"）✓ |
| 设置 | `/settings` | **`/settings`**（别名 `config`，"View and edit application settings"）✓ + `/profile` ✓ |
| 权限 | `/permissions` | `/permissions` ✓ |
| 审批模式 | `/approval-mode`（plan/default/auto-edit/yolo）+ `/plan` | **`/plan`**（"Toggle Plan Mode"）✓；审批走 `--permission-mode` 参数 |
| 记忆 | `/memory` + 托管自动记忆 `/remember` `/forget` `/dream` | `/memory` ✓ + **`remember`**（skill）+ `/manage`（auto-memory TUI）|
| 项目初始化 | `/init`（生成 QWEN.md）| `/init`（"Analyzes the project and creates a tailored context file"，AGENTS.md）✓ |
| 主题 | `/theme` | `/theme` ✓ |
| Vim 模式 | `/vim` | `/vim` ✓ |
| 外部编辑器 | `/editor` | `/editor` ✓ |
| 状态栏 | `/statusline` | `/statusline`（sub-agent 读 shell 配置生成）✓ |
| 自检诊断 | `/doctor` | `qoder doctor`（CLI 子命令）+ `debug`（skill）|
| 信任设置 | `/trust` | — |
| 隐私声明 | — | `/privacy` ✓ |
| 工作区目录 | `/directory`（`dir` add/show）+ `/cd`（会话迁移工作目录）| `/add-dir` ✓ + `/list`（列工作区目录）✓ |
| UI 语言 | `/language`（中/英/日/德/俄/葡）| — |
| 终端集成 | `/terminal-setup` | — |
| IDE 集成 | `/ide` | —（有 ACP 编辑器协议，无 `/ide` 命令）|
| 用量统计 | `/stats`（`usage`）| `/usage`（"…for the current billing period"）✓ + `/insights`（会话分析报告）✓ |
| 遥测 | —（CLI 参数 `--telemetry-*`）| —（`/metrics` `/otel` 为端点字符串，非命令）|
| 文档 | `/docs` | `/docs` ✓ |
| 反馈/报障 | `/bug` | `/feedback`（别名 `bug`）✓ |

**工具与扩展**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 工具列表 | `/tools` | **`/tools`**（"List available tools"）✓ |
| MCP 管理 | `/mcp` | `/mcp` ✓ + `mcp-config`（skill）+ `/reload`（`refresh`，重载 agents/tools）✓ |
| Hooks 管理 | `/hooks`（list/enable/disable）| `/hooks`（"Open hooks management panel"）✓（`/hook-config` 为内部路由非命令）|
| 子代理 | `/agents`（manage/create）| `/agents`（别名 `agent`）✓ |
| 技能 | `/skills` | `/skills` ✓ + `/link`（链接本地 skill）✓ + `skill-fork`（forked skill 执行）|
| 扩展/插件 | `/extensions`（explore/install/manage）| `/plugins`（别名 `plugin`；install/uninstall/enable/disable/update/validate）✓ + `/marketplace`（`market` add/list/remove/update）✓ |

**开发工作流**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 代码审查 | `/review`（bundled skill，四代理并行）| `/review`（"…find actionable issues"）✓ + `security-review`（skill）✓ |
| Git diff | `/diff` | `/diff`（"Show uncommitted git changes"）✓ |
| GitHub Actions | `/setup-github` | **`/setup-github`**（"Set up GitHub Actions"）✓ |
| 多模型竞争 | `/arena`（start/stop/status/select）| — |
| 后台任务/代理 | `/tasks`（调度入口）+ `/fork`（派生继承全对话的后台代理）| **`/tasks`**（别名 `bg` `background`，"Show background tasks panel"）✓ |
| 长任务目标 | `/goal`（"keep working until the condition is met"）| **`/goal`**（"Manage persistent goals"；子命令 `set\|status\|clear\|pause\|resume\|take`）✓ |
| 旁问 | `/btw` | **`/btw`**（"Ask a quick side question without interrupting the main conversation"）✓ |
| 洞察 | `/insight`（代码洞察）| `/insights`（**会话**分析报告，含义不同）✓ |
| Skill 工作流 | `/loop` `/batch` `/simplify` `/stuck` `/new-app` `/qc-helper`（bundled skills）| **`loop` `batch` `simplify`** + `remember` `debug`（bundled skills，前三与 Qwen 同名）✓ |
| 规格驱动 | — | `quest`（bundled skill，v0.x 招牌延续）✓ |
| 语音 | — | `/voice`（"hold Space to record"，`off`/`hold` 子命令）✓ |
| 远程控制 | — | `/remote-control` ✓ + `/remote-env`（云端环境）+ `/qr-code`（扫码配对）+ `/start`/`/disconnect`（连接/断开远程）|
| 看板 | — | `/kanban`（看板后端集成）✓ |
| 彩蛋 | — | `/corgi`（gemini-cli 上游彩蛋）✓ |

**小结**（解码 `_$d()` 后修正）：两边**命令重合度远高于早前判断**——`/btw` `/branch` `/rename` `/plan` `/compact` `/tasks` `/goal` `/tools` `/settings` `/setup-github` `/diff` `/statusline` 以及 bundled skills `loop`/`batch`/`simplify` 双方同名同义（共同上游 + 平行借鉴 Claude/Qwen 表面）。**Qwen 真正独有**：`/arena` 多模型竞争、`/fork` 后台代理、`/insight` 代码洞察、`/recap` `/summary`、`/language` 多语言、`/ide` `/lsp` `/terminal-setup` `/trust` `/approval-mode`、托管记忆 `/dream` `/forget`。**Qoder 真正独有**：`/kanban` 看板、`/qr-code` + `/remote-env` + `/start`/`/disconnect` 远程套件、`/insights` 会话分析、`/profile`、`/privacy` 合规、自营网关计费相关。

## 四、CLI 子命令与启动参数

### CLI 子命令

| 功能 | Qwen Code（yargs，`packages/cli/src/commands/`）| Qoder CLI v1.0（commander，bundle 提取）|
|---|---|---|
| daemon | **`qwen serve`**（Mode B 长驻 runtime）| — |
| MCP | `qwen mcp` | `qoder mcp`（add/add-json/get/list/remove）|
| 扩展 | `qwen extensions` | `qoder plugin` + `qoder mp`（marketplace）|
| Hooks | `qwen hooks` | `qoder hook` |
| 审查 | `qwen review` | —（仅 TUI `/review`）|
| 认证 | `qwen auth` | `qoder login` |
| 发布渠道 | `qwen channel` | —（npm `@latest` / `@beta` 双渠道）|
| 子代理 | —（TUI `/agents`）| `qoder agent` |
| 技能 | —（TUI `/skills`）| `qoder skill`（含 link / external）|
| 配置 | —（TUI `/settings`）| `qoder config`（get/set/unset/list）|
| 自身运维 | —（npm 升级）| `qoder install` / `update`（`--to <version>`）/ `rollback` / `doctor` |
| AI 提交统计 | — | `qoder commit`（v0.x 延续，AI 贡献占比统计）|
| 配置迁移 | — | **`qoder migrate --from-claude`** |
| 远程控制 | — | `qoder remote-control` |
| 反馈 | —（TUI `/bug`）| `qoder feedback` |
| 重置 | — | `qoder reset-project-choices` |

### 启动参数

Qwen Code 在 `packages/cli/src/config/config.ts` 注册 55 个 yargs 选项；Qoder bundle 提取到 60+ 个 commander `option()`。两边共有的基本盘（同源 + 行业惯例）：`--model` `-p`（print/prompt）`-c/--continue` `-r/--resume` `--session-id` `--fork-session` `--system-prompt` `--append-system-prompt` `--allowed-tools` `--mcp-config` `--input-format` `--output-format` `--worktree` `--proxy` `--debug`。

各自独有的参数暴露了产品取向：

| 参数域 | Qwen Code 独有 | Qoder CLI v1.0 独有 |
|---|---|---|
| 审批/权限 | `--approval-mode <plan\|default\|auto-edit\|yolo>`、`--yolo` | `--permission-mode <mode>`（Claude Code 命名）、`--disallowed-tools` |
| 模型接入 | `--auth-type`、`--openai-api-key` / `--openai-base-url` / `--openai-logging`（BYOK 系）| `--reasoning-effort <level>`、`--context-window <size>`、`--agent <name>`、`--agents <json>` |
| 沙箱 | `--sandbox`、`--sandbox-image`、`--sandbox-session-id` | —（沙箱存在但未提取到对应参数）|
| 遥测 | `--telemetry` + `--telemetry-target/-otlp-endpoint/-otlp-protocol/-log-prompts/-outfile` | — |
| 会话治理 | `--max-session-turns`、`--max-tool-calls`、`--max-wall-time`、`--chat-recording` | `--max-output-tokens`、`--list-sessions`、`--delete-session <index>` |
| 远程/多端 | `--acp` / `--experimental-acp`（daemon 见 `qwen serve`）| `--remote [task]`、`--remote-session <id>`、`--remote-control <id>`、**`--teleport <id>`** |
| 目录注入 | `--include-directories` | `--add-dir`（Claude Code 命名）|
| 输出/输入 | `--json-fd` / `--json-file` / `--json-schema`、`--include-partial-messages`、`--input-file`、`--screen-reader` | `--output-style <style>`（Claude Code 概念）、`--attachment <file>` |
| 扩展 | `--extensions` / `--list-extensions`、`--experimental-skills` / `--experimental-lsp` | `--plugin-dir <dir>`、`--settings <json>` / `--setting-sources`、`--strict-mcp-config`（Claude Code 命名）|
| 其他 | `--channel`、`--bare`、`--disabled-slash-commands`、`--core-tools` / `--exclude-tools` | `--from-claude`（migrate 子命令）|

> **观察**：Qoder v1.0 的参数命名大面积对齐 **Claude Code 的 CLI 表面**（`--permission-mode`、`--output-style`、`--add-dir`、`--settings`/`--setting-sources`、`--strict-mcp-config`、`--agents <json>`），上游 Gemini CLI 与 Qwen Code 均无这些命名。结合 `qoder migrate --from-claude` 与 hooks 事件命名（见下节），Qoder 的产品策略是**用 Gemini CLI 的引擎，模仿 Claude Code 的用户界面**，降低 Claude Code 用户的迁移成本。

## 五、内置工具对比

Qwen Code v0.18.0：**24 个默认可用工具 + 17 个条件工具**（懒加载注册，源码 `packages/core/src/config/config.ts#createToolRegistry`、`packages/core/src/tools/tool-names.ts`）+ MCP 动态工具。Qoder：bundle 导出表可见 **21 个工具类**；工具对 LLM 暴露的 wire 名被 `_$d()` 混淆，下表用类名对照（用户可配置的工具名为 Claude Code 风格 `Bash`/`Read`/`Write`/`Edit`/`Glob`/`Grep`/`WebSearch`/`WebFetch`，见包内 `builtin/agent-creator/SKILL.md` 明文）。

| 功能 | Qwen Code v0.18.0 | Qoder CLI v1.0（bundle 导出类名）| 备注 |
|---|---|---|---|
| 读/写/编辑文件 | `read_file` / `write_file` / `edit` | `ReadFileTool` / `WriteFileTool` / `EditTool` | 同源 |
| Jupyter 编辑 | `notebook_edit` | — | Qwen 独有 |
| 列目录 | `list_directory` | —（未见 LSTool 导出）| Qoder 疑似并入 glob/shell |
| 正则搜索 | `grep_search`（ripgrep 优先 + 回退）| `RipGrepTool`（无独立 GrepTool）| 同源（ripgrep 随包分发）|
| 文件模式 | `glob` | `GlobTool` | 同源 |
| Shell | `run_shell_command` | `ShellTool` | 同源 |
| 网页抓取 | `web_fetch` | `WebFetchTool` | 同源 |
| 网页搜索 | **已移除**（web_search 不在当前注册表）| `WebSearchTool` | **反转**：上游能力 Qwen 砍了、Qoder 留着 |
| **图片搜索** | — | **`ImageSearchTool`** | **Qoder 新增**（gemini-cli 上游为 0；依赖 `sharp` 多模态）|
| Todo 列表 | `todo_write` | `WriteTodosTool` | 同源 |
| 任务管理 | `task_create` / `task_update` / `task_list` / `task_stop`（Agent Team）| **`Tracker*Tool` ×6**（Create/Update/Get/List/AddDependency/Visualize，上游子系统）| 两套体系：Qwen 任务绑定多代理调度，Qoder Tracker 带依赖 DAG + 可视化 |
| 定时任务 | `cron_create` / `cron_list` / `cron_delete`（v0.18 起默认启用）| — | Qwen 独有 |
| 多代理协作 | `agent` + `send_message` + `team_create` / `team_delete`（完整 Agent Team）| `TaskTool` + `teammate_mailbox`（仅结果聚合）| 详见 [§六.4](#64-多代理协作深度分化非有无)：teammate 两家都有，深浅不同 |
| 长期记忆 | **save_memory 已移除**——改为托管自动记忆（`/remember` `/forget` `/dream` 命令体系）| `MemoryTool` | **反转**：上游工具 Qwen 改道、Qoder 留着 |
| 技能执行 | `skill` | `ActivateSkillTool` | 同源 |
| 计划模式 | `enter_plan_mode` + `exit_plan_mode`（含 Plan Approval Gate）| —（`planMode` 23× / `plan_mode` 17×，工具名混淆）| 两边都有 plan 模式 |
| 向用户提问 | `ask_user_question` | `AskUserTool` | 同源 |
| worktree | `enter_worktree` / `exit_worktree` 工具 | —（仅 `--worktree` 启动参数）| Qwen 把 worktree 工具化（运行中可进出）|
| 工具延迟加载 | `tool_search`（搜索后按需加载 schema）| — | Qwen 独有（省系统提示 token）|
| 后台监控 | `monitor` | — | Qwen 独有 |
| workflow 编排 | `workflow`（条件）| —（确认无，见 [5.1](#51-dynamic-workflow-与-computer-use-专项)）| Qwen 独有（多代理工作流脚本）|
| computer use | `computer_use__*` ×9（默认开，工具延迟揭示）| —（确认无，见 [5.1](#51-dynamic-workflow-与-computer-use-专项)）| Qwen 独有 |
| LSP | `lsp`（条件，`--experimental-lsp`）| —（未见导出）| Qwen 独有 |
| 结构化输出 | `structured_output`（`--json-schema` 时注册）| — | Qwen 独有 |
| MCP 认证 | —（CLI `qwen mcp` 管理）| `McpAuthTool` | |
| MCP 动态工具 | `mcp__server__tool` 动态注册 | `DiscoveredMCPTool` | 同源 |

**差异要点**：文件/搜索/Shell/Web 基本盘同源。**Qwen 的工具面显著更宽**——Agent Team（team/task/send_message）、cron 定时、workflow 编排、computer use、tool_search 延迟加载、notebook_edit、monitor 都是 Qoder 没有的；**Qoder 的增量**是 ImageSearchTool（自有）+ Tracker 任务 DAG（跟进上游）。有趣的**反转项**：`web_search` 和 `save_memory` 这两个上游工具，Qwen 主动移除（记忆改托管自动化、搜索待重做），Qoder 原样保留。

### 5.1 Dynamic Workflow 与 Computer Use 专项

这两项是 Qwen 有、**Qoder 经 bundle 全量核查确认没有**的高级能力。

**Dynamic Workflow（动态多代理工作流）**

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 工具 | `workflow`（`isWorkflowsEnabled()`，env `QWEN_CODE_ENABLE_WORKFLOWS=1` 或设置开启）| ❌ 无 |
| 实现 | `tools/workflow/` + `agents/runtime/`（`WorkflowOrchestrator` + `workflow-sandbox` + `workflow-prompts`）| — |
| 编程模型 | 注入 JS 脚本跑确定性编排：全局 `phase(title)` / `log(msg)` / **`agent(prompt,{label})`** / `args`；并发 **`parallel([()=>agent()…])`** + **`pipeline(items,…stages)`** fan-out，dispatch 层限流，单 run 上限 1000 个 `agent()` | — |
| 定位 | 把"多 subagent 怎么编排"从模型即兴变成**可复现脚本**（顺序/并行/流水线确定性 fan-out）| — |
| bundle 实证 | — | `workflow` 仅 4×（MIME `application/vnd.oai.workflows` + 遥测字段 `workflow_name`），`WorkflowOrchestrator`/`pipeline(`/`phase(`/`orchestrat` **全 0** |

**Computer Use（计算机操作 / GUI 自动化）**

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 工具 | **9 个** `computer_use__*`：`list_apps`/`get_app_state`/`click`/`scroll`/`drag`/`type_text`/`press_key`/`set_value`/`perform_secondary_action`（`computerUseEnabled` **默认 true**，但 deferred——按需揭示）| ❌ 无 |
| 实现 | `tools/computer-use/`（`bootstrap`/`client`/`schemas`/`registration`/`permission-detector`/`install-state`）——首次用时安装**上游二进制**，macOS 探测辅助功能权限 | — |
| bundle 实证 | — | `computer_use`/`computer-use`/`ComputerUse`/`computerUse` + `list_apps`/`get_app_state`/`press_key` **全 0**（`screenshot`/`type_text` 命中均为语法高亮词表 token，非工具）|

> **小结**：Dynamic Workflow（确定性多代理编排）与 Computer Use（GUI 自动化）都是 **Qwen 独有、Qoder 完全没有**的能力。两者都不来自 Gemini 上游：computer use 走上游 MCP server + bundled 二进制，workflow 是 Qwen 自建编排运行时——属于 Qwen "把自己做成 runtime 平台" 路线的延伸；Qoder 走"单进程 + 自营网关"路线，未投入这两块。

## 六、SubAgent 能力对比

**重要前提（仔细调研后修正）**：subagent 体系**并非来自 Gemini CLI 基座**——上游 gemini-cli **没有** `packages/core/src/tools/agent/`、没有 `subagents/`、没有 `subagent_type`/fork/内置 agent 注册表。**两家是各自在 Gemini 基座上叠加了一套 Claude Code 式 agent 系统**：内置 agent 的描述文案（`general-purpose`/`Explore`/`Plan`/`fork`）在 Qwen 源码与 Qoder bundle 中**逐字相同**，且都是 Claude Code 的 agent 定义原文。这与"两家都向 Claude Code 表面看齐"的总体判断一致。下为源码（Qwen v0.18.0）与 bundle 解码（Qoder v1.0.18）逐项对照。

### 6.1 Agent 定义与内置 agent

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 格式 | Markdown + YAML frontmatter | Markdown + YAML frontmatter（`builtin/agent-creator/SKILL.md` 明文规范）|
| 必填字段 | `name` `description` `systemPrompt`(body) | `name` `description` + body |
| 可选字段 | `tools` `disallowedTools` `model` `color` `maxTurns` `approvalMode`/`permissionMode` `background` `runConfig` | `tools` `disallowedTools` `model` `color` `maxTurns` **`timeoutMins`** **`effort`** `skills` |
| 定义层级 | **4 级**：builtin / user(`~/.qwen/agents/`) / project(`.qwen/agents/`) / extension（project > user）| **2 级**：user(`~/.qoder/agents/`) / project(`.qoder/agents/`，project 优先）|
| 工具命名 | Gemini 系（`read_file`/`run_shell_command`…）| **Claude Code 系**（`Bash`/`Read`/`Write`/`Edit`/`Glob`/`Grep`/`WebSearch`/`WebFetch`）|
| **内置 agent** | **4 个**：`general-purpose`（默认）、`Explore`（`model:'fast'`，强制只读）、`statusline-setup`、`fork`（隐式） | **8 个（更多）**：`general-purpose`、`Explore`、**`Plan`**（架构师）、`fork`、**`SaveMemory`**、`statusline-setup`、`skill-fork`、`skill-extractor`（从历史会话提炼可复用 skill）|

> 内置 agent 的描述在两家**逐字相同**（均 Claude Code 原文，gemini-cli 上游为 0）——Qoder 比 Qwen 还多 `Plan`/`SaveMemory`/`skill-extractor` 三个。

### 6.2 调用与 LLM 接口（Agent 工具参数）

| 参数 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 选择 agent | `subagent_type`（省略 + fork 开启 = 隐式 fork）| `subagent_type`（默认 `general-purpose`；**同样省略即 fork**）|
| 任务 | `prompt` + `description` | `prompt` + `description`（"3-5 word"）|
| 隔离 | `isolation: 'worktree'`（worktree/默认）| `isolation: enum["default","worktree"]`（per-invocation 覆盖 agent 定义）|
| 后台 | `run_in_background`（完成时通知）| `run_in_background`（"notified when it completes"）|
| per-call 模型 | `model` 覆盖 | —（模型走 agent 定义的档位）|
| 用户入口 | `/agents`（create/manage）+ `@`提及 | `/agents`（create/list）+ `@agent-name` + `/agents reload` |
| 权限按 agent 隔离 | 每 agent `tools`/`disallowedTools` | ✅ + **规则按 `subagent_type` 维度匹配**（`Sri(rules, name, subagent_type)`）|

> 两家的 fork 触发逻辑**逐字相同**：`subagent_type ?? (forkEnabled ? undefined : 'general-purpose')`，`undefined` → 走隐式 fork（继承全对话上下文，fork worker 不能再 spawn）。**fork 不是 Qwen 独有**。

### 6.3 模型与资源控制（真正的分化点）

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 模型指定 | **别名 + 具体名**：`inherit`（默认=主模型）/ `fast`（配置的快模型）/ 任意具体模型名（BYOK 任意 provider）| **6 档固定梯**：`inherit`/`lite`/`efficient`/`auto`/`performance`/`ultimate`（绑定自营网关 credit 计费）|
| 思考力度 | runConfig | **`effort`**：`low`/`medium`/`high`/`max`（per-agent）|
| 轮次上限 | `maxTurns` | `maxTurns` |
| 超时 | runConfig timeout | **`timeoutMins`**（per-agent，分钟）|

> 两家都有 `inherit`。差别：Qoder 给的是 **6 档固定能力梯 + effort + timeoutMins** 一套**面向 credit 计费的资源旋钮**（用多强、想多久、跑多少轮全参数化）；Qwen 是 **`fast` 别名 + 具体模型名 + BYOK**，成本取决于用户接的 provider。

### 6.4 多代理协作（深度分化，非有无）

| 能力 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 单层 subagent | ✅ Agent 工具 | ✅ Task 工具（`task`/`delegate`）|
| 隐式 fork | ✅ `FORK_AGENT`（继承全对话 + 字节级 cache parity + ALS 检测嵌套）| ✅ **同款**（描述逐字相同，触发逻辑相同）|
| teammate 概念 | ✅ Claude 式 coordinator/teammate | ✅ **也有**（`teammate_mailbox`/`teammateName`/`teammate_shutdown_batch`）|
| **协作深度** | **完整 Agent Team**：`team_create`/`team_delete` + `task_create`/`update`/`list` + `send_message`；**teammate 生命周期事件**（`teammate_joined`/`_idle`/`_exited`/`_status`/`_approval`，`teammate_message` 35×）+ **broadcast** + 共享任务板 `~/.qwen/tasks/{team}` | **仅结果聚合**：`teammate_mailbox` 把多个并行 subagent 的 `{from, text, state}` 汇成一个邮箱视图 + `teammate_shutdown_batch` 批量收尾；**无生命周期事件、无 broadcast、无 team_* 工具、无持久任务板** |
| 编排工具 | `cron`（定时）/ `monitor`（监控）/ `workflow`（脚本编排）/ computer-use | — |
| 嵌套 | 父子 depth 追踪（ALS）；fork worker 禁止再 spawn | depth 追踪（`SubAgentLifecycle`，`depth:1`）|
| 并发上限 | 并行 = 单消息多 Agent block | 进程级 `concurrencyLimit`（默认 10）|

> **修正**：早前写"Qoder 无团队层"过于绝对。准确说法是——**teammate/mailbox 概念两家都有**（都源自 Claude Code），但 Qwen 把它做成了**完整 Agent Team**（生命周期事件 + 代理间消息 broadcast + 持久共享任务板 + cron/monitor/workflow 编排工具），Qoder 只到**并行结果聚合**（mailbox 收集多 subagent 输出，无代理间通信、无持久协作状态）。

### 6.5 进度展示（SubAgent 视图）

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 进度事件 | LiveAgentPanel + pill+dialog（PR#3909/#3768）| `subagentProgress` 事件 `{agentName, state, recentActivity}` |
| 状态机 | running/completed/error… | `pending → queued → running / in_progress → completed / error` |
| 实时活动 | transcript inline + panel | `recentActivity[]`（type `thought`/`action`/`tool_call`/`tool_result`，运行中 `"Working..."`）|
| 渲染 | LiveAgentPanel 专用面板 | 树形盒绘 `├└│⎿`（Gemini/Claude 血脉）+ `elapsed`/`duration` 计时 |
| 并行多 agent | ✅ **LiveAgentPanel**（always-on 锚定 footer）+ InlineParallelAgentsDisplay（`/review` 9-agent 密集面板）| 并行结果汇入 `teammate_mailbox` 视图；**无 always-on 常驻面板**，进度走内联树形 + 按需 `/tasks` |
| 后台面板 | `/tasks` + 4-kind background tasks | `/tasks`（`bg`/`background`）`custom_dialog` 面板，统一跟踪后台 subagent + shell |

### 6.6 一句话判读

**比想象中更同源**——subagent 体系两家都不是从 Gemini 继承，而是**各自叠了一套 Claude Code 式 agent 系统**（内置 agent 文案逐字相同；fork 触发逻辑相同；teammate/mailbox 概念都有）。Qoder 内置 agent 甚至比 Qwen 多（8 vs 4，含 Plan/SaveMemory/skill-extractor）。真正的分化只剩两点：① **资源模型**——Qoder 用 6 档固定模型梯 + effort + timeoutMins 把成本参数化（服务 credit 计费），Qwen 用 `fast` 别名 + 具体模型名 + BYOK；② **协作深度**——同样有 teammate/mailbox，但 Qwen 做成了**完整 Agent Team**（生命周期事件 + broadcast + 持久共享任务板 + cron/monitor/workflow 编排 + 常驻 LiveAgentPanel），Qoder 只到**并行结果聚合**（mailbox 收集输出，无代理间通信/编排/常驻面板）。要真正的多代理编排选 Qwen；要按档位控成本 + 与 Claude Code agent 配置同构、且够用单层并行选 Qoder。

## 七、被集成能力（作为服务端 / 可嵌入对象）

"被集成" = 该 Agent 能否作为**服务端 / 可嵌入对象被别的系统驱动**（北向接口）。这是两家**差距最大的维度**：Qwen Code 把自己做成了开放的 runtime 平台，Qoder 仍以单进程 CLI 为主、远程能力绑定自家云。

| 集成通道 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| **HTTP daemon** | ✅ **`qwen serve`**——REST + SSE，1 daemon = 1 workspace × N session，完整控制面（session/prompt/权限/工具/MCP/rewind 全路由）| ❌ 无开放 HTTP daemon（远程见下「remote-control」）|
| **ACP（Agent Client Protocol）** | ✅ **三形态**：stdio（`qwen --acp`）+ HTTP（#4472）+ WebSocket（#4773），29 个 `_qwen/*` extension method 与 REST 对等 | ⚠️ **仅 stdio**（`--acp` / `acpMode`，Gemini CLI 继承 + `acpAuthenticatedType`），无 HTTP/WS |
| **作为 MCP server 被调** | ✅ **`qwen-serve-bridge`**（`packages/sdk-typescript/src/daemon-mcp/serve-bridge`）把 daemon 包装成 MCP server，Claude Desktop / Cursor / Qoder 等 MCP host 即插即用 | ❌ **仅 MCP 客户端**（`qoder mcp add/get/list/remove/auth`，**无 `serve` 子命令**）——不能被当 MCP server 调用 |
| **SDK（程序化嵌入）** | ✅ **TypeScript / Python / Java**（`packages/sdk-{typescript,python,java}`，`DaemonClient`/`DaemonSessionClient` typed client）| ❌ **无公开 SDK**（bundle 内仅残留 `@google/gemini-cli-devtools` 的 .d.ts）|
| **Web / 桌面客户端** | ✅ web-shell（React 组件，可嵌入任意 React 应用）+ webui（共享层）+ 官方桌面 app | Qoder IDE（独立产品线，闭源）|
| **headless / 非交互** | ✅ `-p` + `--output-format`（text/json/stream-json）+ **`--json-schema`（结构化输出）** + `--bare` + `--input-format` + `--include-partial-messages` | ✅ `-p/--print` + `--output-format` + `--input-format`（stream-json）+ `--headless` |
| **远程控制** | 经 daemon + SDK 多端 attach（同 session 多 client 订阅，开放协议）| ✅ **remote-control daemon**：`/remote-control`（"Start the remote-control daemon"）+ `--remote-control`（"headless remote-control worker"）+ `--remote`（创建云端会话返回 access URL）+ `--teleport`/`--remote-session`（attach 远程会话）+ `/qr-code` 扫码配对——**但走 Qoder 云中转、由 Qoder web/mobile 一方控制** |
| **IDE 集成** | vscode-ide-companion + zed-extension（stdio `--acp`）| 标准 ACP stdio（Zed 等 ACP-native 编辑器）；注：bundle 内 `zed` 713× 实为内部状态库 `zedState`/`zedProps`，非 Zed 编辑器集成 |
| **CI / GitHub** | `qwen setup-github` + 非交互模式 | `/setup-github` + 非交互 `-p`（自营网关计费）|

**判读**：两家都能**被 headless 驱动**（`-p` + 输出格式）和**经标准 ACP stdio attach**（编辑器集成基本盘同源）。真正的分水岭在**"作为服务端被任意第三方集成"**：

- **Qwen Code = 开放平台**——daemon REST/SSE + ACP HTTP/WS + **MCP-server 桥** + 三语言 SDK + 可嵌入 web 组件，全部走开放标准协议，任何第三方都能 attach / 嵌入 / 远程多端协作（详见 [daemon 设计系列](./qwen-code-daemon-design/README.md)）。
- **Qoder CLI = 单进程 + 自家云闭环**——无开放 HTTP daemon、不能作 MCP server、无 SDK；它**确实有 remote-control daemon 与云端远程会话**（`--remote`/`--teleport`/`/qr-code`），但这是**为 Qoder 自家 web/mobile 端控制**设计的闭环（云中转 + 扫码配对），不是给第三方的开放 API。

一句话：**Qwen 让"别人来集成我"，Qoder 让"我的云来控制我"**。需要把 Agent 嵌进自研系统 / IDE / 多端编排选 Qwen；只需 headless 跑批或用 Qoder 官方 web/mobile 远程盯一个跑在 devbox 上的会话，Qoder 够用。

## 八、Hooks 与扩展生态

### Hooks：两家不约而同改用 Claude Code 事件命名

Gemini CLI 上游的 hooks 子系统用自有命名（`BeforeTool` / `BeforeAgent` / `AfterAgent`…），并提供 `hooks migrate` 命令把 Claude Code hook 配置**转换**成自有命名（源码 `packages/cli/src/commands/hooks/migrate.ts`：`PreToolUse→'BeforeTool'`）。而两个阿里 fork 都**直接采用了 Claude Code 的事件分类**——上游命名在两者中均为 0：

| Hook 事件 | Qwen Code（源码 `packages/core/src/hooks/`）| Qoder bundle（字符串计数）|
|---|---|---:|
| `PreToolUse` | ✅ | 24 |
| `PostToolUse` | ✅ | 38 |
| `PostToolUseFailure` | ✅ | 17 |
| `UserPromptSubmit` | ✅ | 25 |
| `SessionStart` / `SessionEnd` | ✅ | 55 / 16 |
| `PreCompact` | ✅ | 12 |
| `Stop` / `SubagentStop` | ✅ | 多 / 17 |
| `Notification` | ✅ | 多 |

即：**两边 hooks 事件模型同为 Claude Code 10 事件分类**，配置可互相参考。管理入口：Qwen `/hooks` + `qwen hooks`；Qoder `/hooks` + `/hook-config` + `qoder hook`。

### 扩展生态：extensions vs plugins + marketplace

| | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 机制 | **extensions**（继承 Gemini CLI 扩展体系）| **plugins + marketplace**（自建/深度改造）|
| 管理 | `/extensions` explore/install/manage + `--extensions` | `/plugins` install/uninstall/enable/disable/update/validate + `/marketplace` add/list/remove/update |
| 市场 | 扩展市场（explore 打开）| 多 marketplace 源可注册；bundle 中预置 **`claude-plugins-official`** 源，并识别 **`.claude-plugin`** 目录格式 |
| 自定义命令 | `.qwen/commands/*.md`（`@{}` `!{}` `{{args}}` 变量）| 插件分发（`--plugin-dir` 本地开发）|
| 技能 | `.qwen/skills/` + `--experimental-skills` | `SKILL.md`（10×）+ `qoder skill link/external` + `skill-fork` 执行 |
| 子代理 | `/agents` 创建管理 | `qoder agent` + `--agents <json>` |

注意 `claude-plugins-official`：Qoder 的 marketplace 不仅服务自有插件，还**直接对接 Claude Code 官方插件市场格式**——与下一节的迁移策略一致。

## 九、Claude Code 兼容性

两边都把 Claude Code 当作事实标准对齐，但深度不同：

| 兼容点 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| Hook 事件命名 | ✅ 10 事件 Claude Code 分类 | ✅ 同 |
| CLI 参数命名 | 部分（`--allowed-tools` `--append-system-prompt` `--fork-session`…）| **大面积**（另含 `--permission-mode` `--output-style` `--add-dir` `--settings` `--setting-sources` `--strict-mcp-config` `--agents <json>`）|
| 配置迁移 | — | **`qoder migrate --from-claude`** 一次性迁移命令 |
| 插件市场格式 | — | `.claude-plugin` 格式 + `claude-plugins-official` 市场源 |
| 声明式 agent 定义 | ✅ frontmatter 支持 `tools`/`disallowedTools`/`mcpServers`/`hooks`（CC 2.1.168 parity，#4996/#4935）| ✅ frontmatter 支持 `tools`/`disallowedTools`/`model`/`effort`/`skills` 等，工具名用 Claude Code 命名（`builtin/agent-creator/SKILL.md`）|
| 上下文文件 | `QWEN.md`（兼容 GEMINI.md）| `AGENTS.md`；上下文用量 UI 内部把 memory 文件标记为 `claudeMd` 类型 |

> **变化说明**：Qoder v0.x（Go 时代）的运行时参数 `--with-claude-config`（直接读 `.claude` 目录）在 v1.0 bundle 中已**不存在**（字符串 0 次），被一次性迁移命令 `qoder migrate --from-claude` 取代。

## 十、关键分歧

### 1. 许可与定位

- **Qwen Code**：Apache-2.0 **开源**，社区驱动 + 阿里云背书；软件本身免费，但模型调用需自付
- **Qoder CLI**：**闭源专有商业产品**，自有账号 + PAT + 计费；定位企业/付费

### 2. 模型与认证（都不再有第一方免费额度）

- **Qwen Code**：三种认证——① Qwen OAuth（**免费层 2026-04-15 已停**，新请求被拒）② 阿里云 Coding Plan（付费月度订阅，含 Qwen/GLM/Kimi/Minimax）③ BYOK API key（OpenAI/Anthropic/Gemini 兼容，DeepSeek 双协议深度适配，详 [Qwen Code 对 DeepSeek 的支持](./qwen-code-deepseek-support.md)）。**无厂商锁定，可接任意 provider**
- **Qoder CLI**：**自营模型网关**（`api.qoder.sh` 全球 / `api.qoder.com.cn` 国内双端点）聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek，Qoder 账号统一计费（credit 制，bundle 含 credit 耗尽告警）；浏览器登录 + PAT（CI 用）

### 3. 沙箱：同源，非任何一方的差异化

两家的 macOS Seatbelt 沙箱都继承自 Gemini CLI，均为 6 个 `.sb` profile，矩阵略有不同：

- **Qoder v1.0**：`{strict, restrictive, permissive} × {proxied, open}`——与 gemini-cli 当前 HEAD 逐字一致（仅环境变量改名 `QODER_SANDBOX_PROXY_COMMAND`）
- **Qwen Code**：`{restrictive, permissive} × {closed, open, proxied}`——跟的是上游较早矩阵（无 strict 档，多 closed 档；源码 `packages/cli/src/utils/sandbox-macos-*.sb`），另暴露 `--sandbox` / `--sandbox-image` 容器沙箱参数

### 4. 运行时形态（Qwen Code 的差异化投入）

- **Qwen Code**：**daemon Mode B**——`qwen serve` 长驻 daemon + 多 client（TUI/web-shell/ACP HTTP/ACP WebSocket/MCP stdio/官方桌面 app），完整 telemetry，是一整套 runtime 平台（详 [daemon 设计](./qwen-code-daemon-design/README.md)）
- **Qoder CLI**：**无 daemon runtime**——单进程 CLI，但有自己的远程形态：`--remote [task]` / `--remote-session` / `--teleport <id>` / `remote-control`（走 Qoder 云端账号体系，而非本地 daemon 多路复用）

### 5. 各自招牌功能

| Qwen Code 独有 | Qoder CLI v1.0 独有 |
|---|---|
| daemon Mode B 多端 runtime + web-shell + 官方桌面 app | 自营模型网关 + credit 统一计费 + 国内/全球双端点 |
| Arena 多模型竞争（`/arena`）| Quest 规格驱动（`/quest`，v0.x 延续）|
| Agent Team 多代理（team/task/send_message 工具族）+ cron 定时 + workflow 编排 + computer use | plugins + marketplace 生态（含 Claude 插件市场对接）|
| telemetry 分层 tracing（OTel，#3731）+ DashScope/ModelScope/Qwen OAuth 原生接入 | Claude Code 式 CLI 表面 + `migrate --from-claude` 迁移链 |
| `/arena` `/insight` `/fork` `/recap` + 托管自动记忆 + 多语言 UI（注：`/btw` `/tasks` `/branch` 两边都有）| `/kanban` 看板 + ImageSearchTool + Tracker 任务 DAG + `/qr-code`/`/remote-env` 远程套件 + `/insights` 会话分析 |

## 十一、横向速览表

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 血脉 | Gemini CLI fork | Gemini CLI fork（**非** Qwen Code fork）|
| 许可 | Apache-2.0 开源 | 闭源专有 |
| 价格 | 软件免费，模型自付（OAuth 免费层已停）| 商业计费（自营网关，credit 制）|
| 模型 | Qwen + Coding Plan + 任意 provider（BYOK）| 自营网关聚合 8+ 模型 |
| 认证 | Qwen OAuth〔停〕/ Coding Plan / API key | 浏览器登录 + PAT |
| 内置命令 | **57**（52 常驻 + 5 条件）+ 7 bundled skill（源码确认，v0.18.0）| **78 个 `kind:"built-in"` name**（含子命令，顶层 ≈ 50）+ 8 bundled skill + `/goal`（解码 `_$d()` 后权威清单）|
| CLI 子命令 | 7（serve/mcp/extensions/hooks/review/auth/channel）| 15+（含 migrate/doctor/rollback/commit/config）|
| 启动参数 | 55（yargs）| 60+（commander）|
| 核心工具 | **24 默认 + 17 条件**（cron 默认开；team/task/workflow/computer-use×9 条件）+ MCP | 21 类导出（含 Tracker×6、ImageSearch）+ MCP |
| Hook 事件 | 10（Claude Code 命名）| 10（Claude Code 命名）|
| subagent | 4 内置 agent + fork + **完整 Agent Team**（生命周期事件/broadcast/共享任务板）+ 常驻 LiveAgentPanel | **8 内置 agent**（含 Plan/SaveMemory/skill-extractor）+ fork + **仅 teammate_mailbox 结果聚合**（无代理间通信）+ 内联树形 / `/tasks` 面板。注：两家 agent 体系均非 Gemini 继承，都是 Claude 式叠加 |
| Arena 多模型 | ✅ 招牌 | ❌（bundle 无 Arena；"arena" 实为 protobuf）|
| Dynamic Workflow | ✅ `workflow` 工具（phase/agent/parallel/pipeline 确定性编排）| ❌（bundle 全量核查无）|
| Computer Use | ✅ 9 个 `computer_use__*`（默认开，上游二进制）| ❌（bundle 全量核查无）|
| 任务管理 | task_create/update/list + cron + monitor 工具族 | Tracker DAG ×6 工具（上游跟进）|
| 沙箱 | Seatbelt 6 profile（旧矩阵）+ 容器参数 | Seatbelt 6 profile（新矩阵，上游逐字继承）|
| daemon runtime | ✅ **Mode B 多端** | ❌ 无（远程走云端 `--teleport`/`remote-control`）|
| **被集成（北向接口）** | ✅ **开放平台**：HTTP daemon + ACP(stdio/HTTP/WS) + **MCP-server 桥** + TS/Python/Java SDK + 可嵌入 web 组件 | ⚠️ **headless + ACP stdio + Qoder 云远程控制**；无开放 daemon / 不能作 MCP server / 无 SDK（详 [§七](#七被集成能力作为服务端--可嵌入对象)）|
| telemetry | ✅ 深度（OTel 分层 tracing + 5 个 CLI 参数）| 自有遥测端点（`/metrics` Prometheus / `/otel` 为端点，非 slash 命令），深度未知 |
| 插件生态 | extensions + `.qwen/commands` | plugins + marketplace（对接 Claude 插件市场）|
| 桌面 app | ✅ 官方（#3778）| Qoder IDE（独立产品线）|
| 上下文文件 | `QWEN.md` | `AGENTS.md`（+ SKILL.md / MEMORY.md）|
| Claude 迁移 | —（hook 命名兼容）| `migrate --from-claude` + CLI 表面对齐 |
| 国内/全球 | 单端点 | **双端点**（qoder.com.cn / qoder.sh）|

## 十二、一句话总结

**Qwen Code 与 Qoder CLI v1.0 是 Gemini CLI 的两个兄弟 fork，都出自阿里系，但互不衍生，且对齐的对象不同**：Qwen Code 在 Gemini 基座上加 DashScope/ModelScope/Arena/daemon 多端 runtime，把自己做成**开源 BYOK 的 runtime 平台**；Qoder v1.0 在同一基座上套自营网关 + credit 计费 + plugins marketplace，并把 CLI 参数、hooks、插件市场、迁移工具全部**向 Claude Code 的用户表面看齐**，做成闭源商业产品。底座（工具/ContentGenerator/MCP/subagent/沙箱）同源相似；选型看五点——**商业模式**（开源 vs 闭源）、**模型接入**（BYOK 任意 provider vs 自营网关统一计费）、**运行时**（本地 daemon 多端 vs 云端 remote/teleport）、**被集成**（开放平台：daemon+ACP+MCP-server+SDK，"别人来集成我" vs 单进程 + Qoder 云远程控制闭环，"我的云来控制我"）、**生态迁移**（从 Gemini/Qwen 系迁移选 Qwen Code，从 Claude Code 迁移 Qoder 接得更顺）。

> **被集成是两家差距最大的维度**：Qwen Code 暴露开放标准协议 + 三语言 SDK + 可嵌入组件，任意第三方都能把它 attach / 嵌入 / 多端编排；Qoder 无开放 HTTP daemon、不能作 MCP server、无 SDK，远程能力是绑定自家云、由 Qoder web/mobile 一方控制的闭环。要把 Agent 嵌进自研系统 / IDE / 多端选 Qwen，只需 headless 跑批或用 Qoder 官方端远程盯会话则 Qoder 够用。

> **注**：Qwen Code 的 Qwen OAuth 免费层已于 **2026-04-15 终止**（此前 04-13 先从 1000 次/天降到 100 次/天）。现在两者都没有第一方免费额度——但 Qwen Code 仍可 BYOK 接入低价 provider（如 DeepSeek、ModelScope 等）或自部署开源模型。

---

> **免责声明**：Qoder CLI **闭源**，本文 Qoder 侧数据基于 `@qoder-ai/qodercli` v1.0.18 npm bundle 的静态分析（2026-06-11/12，命令清单 06-13 解码 `_$d()` 后复核），与 qwen-code / gemini-cli 源码三方对照，无 Qoder 源码。**命令名为明文字面量、描述经 `_$d()` 确定性解码**（命令侧已是权威清单，不再是下界）；工具/参数侧 wire 名仍部分混淆，标注下界。"fork 自 Gemini CLI、非 Qwen Code"为三角验证（Gemini 原版名保留 + Qwen 专属标记全 0 + corgi/devtools/沙箱 profile 实物）的强证据推断。Qwen Code 数据来自源码（qwen-code v0.18.0 @ 2026-06-13 HEAD）与本仓库文档。截至 2026-06-13。
