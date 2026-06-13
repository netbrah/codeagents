# Qwen Code vs Qoder CLI 对比

> **核心结论（2026-06 复核）**：Qoder CLI **v1.0 已从 v0.x 的 Go 二进制完全重写为 JS bundle，基座是 Gemini CLI**。它与 Qwen Code 是 **Gemini CLI 的两个兄弟 fork**（均为阿里系产品），但 **Qoder 不是 Qwen Code 的 fork**——经 gemini-cli / qwen-code / Qoder bundle 三方字符串三角验证：Qoder 保留的是 **Gemini CLI 原版内部名**（`geminiIgnoreFilter`、`GEMINI.md`、`MemoryDiscovery`），而 Qwen Code 专属标记（`DashScope` / `ModelScope` / `chat.qwen.ai` / `qwenOAuth` / `QwenContentGenerator` / `QWEN.md`）在 Qoder bundle 中**全部为 0**。
>
> 证据：本地分析 `@qoder-ai/qodercli` v1.0.18（2026-06-11 npm 最新版）的 34 MB esbuild bundle，与 qwen-code / gemini-cli 源码三方对照（详细提取记录见 [Qoder CLI EVIDENCE](../tools/qoder-cli/EVIDENCE.md)）。
>
> **方法论提示**：Qoder bundle 对大量字符串字面量做了混淆（`_$d("Base64…")` 解码函数），命令描述、工具对 LLM 暴露的名称等多数不可静态读取。本文 Qoder 侧的命令/工具/参数清单是**静态可提取的下界**，非穷尽全集；语义标注按明文名称推断。

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
- **macOS Seatbelt 沙箱 profile**（两家都从上游带出，矩阵略有差异，见 [八、关键分歧](#八关键分歧)）
- **voice mode**（gemini-cli 上游能力；Qoder bundle `voice` 110×，Qwen Code 未启用为命令）
- **非交互模式**（`-p`）+ `--output-format` + `--worktree` + ACP 编辑器协议

## 三、内置 slash 命令对比

- **Qwen Code**：**57 个内置命令**（v0.18.0：52 个常驻 + 5 个条件启用 `/ide` `/trust` `/lsp` `/dream` `/forget`；源码 `packages/cli/src/services/BuiltinCommandLoader.ts`），另有 **7 个 bundled skill 命令**（`/review` `/loop` `/batch` `/simplify` `/stuck` `/new-app` `/qc-helper`，源码 `packages/core/src/skills/bundled/`）+ MCP prompts + `.qwen/commands/*.md` 自定义命令。
- **Qoder CLI**：bundle 静态提取到 **45+ 个**命令注册（明文 `name/description` 对 + 命令名字符串 + help 文本），描述多数被混淆，下表为可识别下界。

### 命令对照（按功能分组）

> ✓ = 明文注册（名称+描述均可读）；✓* = bundle 字符串命中，描述被混淆，语义按名称推断；— = 未发现对应命令。

**会话与上下文**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 新对话 / 清屏 | `/clear` | `/new` ✓ |
| 压缩上下文 | `/compress` + `/compress-fast`（无 LLM 规则式）| —（`PreCompact` hook 存在，命令名未提取到）|
| token 用量 | `/context` | `/context` ✓* |
| 继续会话 | `/resume` | `/resume` ✓* |
| 检查点回滚 | `/restore` + `/rewind` | `/rewind`（别名 `checkpoint`）✓ |
| 会话分支 | `/branch`（fork 当前对话到新会话）| —（仅参数 `--fork-session`）|
| 会话改名/删除/回顾 | `/rename` `/delete` `/recap` | —（仅参数 `--list-sessions` `--delete-session`）|
| 导出会话 | `/export`（html/md/json/jsonl）| `/export` ✓* |
| 复制回复 | `/copy` | `/copy` ✓* |
| 会话摘要 | `/summary` | — |
| 退出 | `/quit`（`exit`）| `/quit`（`exit`）✓ |

**配置与系统**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 帮助 | `/help`（`?`）| `/help` ✓ |
| 版本信息 | `/status`（`about`）| `/about` ✓ + `/status` ✓ |
| 登录认证 | `/auth`（`login`）| `/login` ✓ + `/token` ✓* + `/authorize` ✓* |
| 切换模型 | `/model` | `/model` ✓* + `/compare` ✓*（按名推断为模型对比）|
| 设置 | `/settings` | `/settings` ✓* |
| 权限 | `/permissions` | `/permissions` ✓ |
| 审批模式 | `/approval-mode`（plan/default/auto-edit/yolo）+ `/plan` | —（用 `--permission-mode` 参数 + `plan_mode` 字符串）|
| 记忆 | `/memory` + 托管自动记忆 `/remember` `/forget` `/dream` | `/memory` ✓* |
| 项目初始化 | `/init`（生成 QWEN.md）| `/init` ✓*（上下文文件为 AGENTS.md）|
| 主题 | `/theme` | `/theme` ✓ |
| Vim 模式 | `/vim` | `/vim` ✓* |
| 外部编辑器 | `/editor` | `/editor` ✓* |
| 状态栏 | `/statusline` | `/statusline` ✓* |
| 自检诊断 | `/doctor` | `qoder doctor`（CLI 子命令）|
| 信任设置 | `/trust` | — |
| 隐私声明 | — | `/privacy` ✓ |
| 工作区目录 | `/directory`（`dir` add/show）+ `/cd`（会话迁移工作目录）| `/add-dir` ✓* |
| UI 语言 | `/language`（中/英/日/德/俄/葡）| — |
| 终端集成 | `/terminal-setup` | — |
| IDE 集成 | `/ide` | —（有 ACP 编辑器协议，无 `/ide` 命令）|
| 用量统计 | `/stats`（`usage`）| `/usage` ✓* |
| 遥测 | —（CLI 参数 `--telemetry-*`）| `/metrics` ✓* + `/otel` ✓* |
| 文档 | `/docs` | `/docs` ✓* |
| 反馈/报障 | `/bug` | `/feedback` ✓ |

**工具与扩展**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 工具列表 | `/tools` | `/tools` ✓* |
| MCP 管理 | `/mcp` | `/mcp` ✓* + `/mcp-config` ✓* + `/reload`（`refresh`，重载 MCP server）✓ |
| Hooks 管理 | `/hooks`（list/enable/disable）| `/hooks` ✓ + `/hook-config` ✓* |
| 子代理 | `/agents`（manage/create）| `/agents` ✓ |
| 技能 | `/skills` | `/skills` ✓ + `skill-fork` ✓（forked skill 执行）|
| 扩展/插件 | `/extensions`（explore/install/manage）| `/plugins`（install/uninstall/enable/disable/update/validate）✓ + `/marketplace`（`market` add/list/remove/update）✓ |

**开发工作流**

| 功能 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 代码审查 | `/review`（bundled skill，四代理并行）| `/review` ✓* + `/security-review` ✓* |
| Git diff | `/diff` | `/diff`（Show uncommitted git changes）✓ |
| GitHub Actions | `/setup-github` | `/setup-github` ✓* |
| 多模型竞争 | `/arena`（start/stop/status/select）| — |
| 后台任务/代理 | `/tasks`（调度入口）+ `/fork`（派生继承全对话的后台代理）| — |
| 长任务目标 | `/goal`（"keep working until the condition is met"）| `/goal` ✓*（子命令 pause/resume/hold/status/clear 明文）|
| 代码洞察 | `/insight` | — |
| 旁问 | `/btw` | — |
| Skill 工作流 | `/loop` `/batch` `/simplify` `/stuck` `/new-app` `/qc-helper`（bundled skills）| —（skills 机制存在，未见 bundled 命令明文）|
| 规格驱动 | — | `/quest` ✓*（v0.x 招牌功能，v1.0 存续）|
| 语音 | — | `/voice` ✓*（off 子命令明文；上游 gemini-cli 能力）|
| 远程控制 | — | `/remote-control` ✓* |
| 彩蛋 | — | `/corgi` ✓（gemini-cli 上游彩蛋）|

**小结**：~30 个命令两边同名/同义（同一上游的痕迹，外加 `/goal` `/diff` `/statusline` 这类两边各自新增却撞名的功能）。Qwen 独有命令偏**工程化工作流与会话治理**（`/arena` `/tasks` `/branch` `/fork` `/insight` `/btw` `/plan` `/recap` + 托管记忆三件套 + 6 个 bundled skill）；Qoder 独有命令偏**产品运营面**（`/plugins` + `/marketplace` 生态、`/usage` `/token` `/metrics` `/otel` 计费遥测、`/quest` `/voice` `/remote-control` 形态扩展、`/privacy` 合规）。

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
| 多代理协作 | `agent` + `send_message` + `team_create` / `team_delete`（Agent Team）| `TaskTool`（单 subagent）| Qwen 团队化（代理间消息）|
| 长期记忆 | **save_memory 已移除**——改为托管自动记忆（`/remember` `/forget` `/dream` 命令体系）| `MemoryTool` | **反转**：上游工具 Qwen 改道、Qoder 留着 |
| 技能执行 | `skill` | `ActivateSkillTool` | 同源 |
| 计划模式 | `enter_plan_mode` + `exit_plan_mode`（含 Plan Approval Gate）| —（`planMode` 23× / `plan_mode` 17×，工具名混淆）| 两边都有 plan 模式 |
| 向用户提问 | `ask_user_question` | `AskUserTool` | 同源 |
| worktree | `enter_worktree` / `exit_worktree` 工具 | —（仅 `--worktree` 启动参数）| Qwen 把 worktree 工具化（运行中可进出）|
| 工具延迟加载 | `tool_search`（搜索后按需加载 schema）| — | Qwen 独有（省系统提示 token）|
| 后台监控 | `monitor` | — | Qwen 独有 |
| workflow 编排 | `workflow`（条件）| — | Qwen 独有（多代理工作流脚本）|
| computer use | `computer_use__*` ×9（条件，MCP 上游实现）| — | Qwen 独有 |
| LSP | `lsp`（条件，`--experimental-lsp`）| —（未见导出）| Qwen 独有 |
| 结构化输出 | `structured_output`（`--json-schema` 时注册）| — | Qwen 独有 |
| MCP 认证 | —（CLI `qwen mcp` 管理）| `McpAuthTool` | |
| MCP 动态工具 | `mcp__server__tool` 动态注册 | `DiscoveredMCPTool` | 同源 |

**差异要点**：文件/搜索/Shell/Web 基本盘同源。**Qwen 的工具面显著更宽**——Agent Team（team/task/send_message）、cron 定时、workflow 编排、computer use、tool_search 延迟加载、notebook_edit、monitor 都是 Qoder 没有的；**Qoder 的增量**是 ImageSearchTool（自有）+ Tracker 任务 DAG（跟进上游）。有趣的**反转项**：`web_search` 和 `save_memory` 这两个上游工具，Qwen 主动移除（记忆改托管自动化、搜索待重做），Qoder 原样保留。

## 六、Hooks 与扩展生态

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

## 七、Claude Code 兼容性

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

## 八、关键分歧

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
| `/insight` `/btw` `/tasks` `/branch` `/fork` + 托管自动记忆 + 多语言 UI | ImageSearchTool + Tracker 任务 DAG + `--teleport` 远程会话 + `/voice` |

## 九、横向速览表

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 血脉 | Gemini CLI fork | Gemini CLI fork（**非** Qwen Code fork）|
| 许可 | Apache-2.0 开源 | 闭源专有 |
| 价格 | 软件免费，模型自付（OAuth 免费层已停）| 商业计费（自营网关，credit 制）|
| 模型 | Qwen + Coding Plan + 任意 provider（BYOK）| 自营网关聚合 8+ 模型 |
| 认证 | Qwen OAuth〔停〕/ Coding Plan / API key | 浏览器登录 + PAT |
| 内置命令 | **57**（52 常驻 + 5 条件）+ 7 bundled skill（源码确认，v0.18.0）| **45+**（静态提取下界）|
| CLI 子命令 | 7（serve/mcp/extensions/hooks/review/auth/channel）| 15+（含 migrate/doctor/rollback/commit/config）|
| 启动参数 | 55（yargs）| 60+（commander）|
| 核心工具 | **24 默认 + 17 条件**（cron 默认开；team/task/workflow/computer-use×9 条件）+ MCP | 21 类导出（含 Tracker×6、ImageSearch）+ MCP |
| Hook 事件 | 10（Claude Code 命名）| 10（Claude Code 命名）|
| subagent | ✅ `/agents` + **Agent Team**（team/send_message）| ✅ `qoder agent`（单 subagent）|
| Arena 多模型 | ✅ 招牌 | ❌（bundle 无 Arena；"arena" 实为 protobuf）|
| 任务管理 | task_create/update/list + cron + monitor 工具族 | Tracker DAG ×6 工具（上游跟进）|
| 沙箱 | Seatbelt 6 profile（旧矩阵）+ 容器参数 | Seatbelt 6 profile（新矩阵，上游逐字继承）|
| daemon runtime | ✅ **Mode B 多端** | ❌ 无（远程走云端 `--teleport`/`remote-control`）|
| telemetry | ✅ 深度（OTel 分层 tracing + 5 个 CLI 参数）| `/metrics` `/otel` 命令，深度未知 |
| 插件生态 | extensions + `.qwen/commands` | plugins + marketplace（对接 Claude 插件市场）|
| 桌面 app | ✅ 官方（#3778）| Qoder IDE（独立产品线）|
| 上下文文件 | `QWEN.md` | `AGENTS.md`（+ SKILL.md / MEMORY.md）|
| Claude 迁移 | —（hook 命名兼容）| `migrate --from-claude` + CLI 表面对齐 |
| 国内/全球 | 单端点 | **双端点**（qoder.com.cn / qoder.sh）|

## 十、一句话总结

**Qwen Code 与 Qoder CLI v1.0 是 Gemini CLI 的两个兄弟 fork，都出自阿里系，但互不衍生，且对齐的对象不同**：Qwen Code 在 Gemini 基座上加 DashScope/ModelScope/Arena/daemon 多端 runtime，把自己做成**开源 BYOK 的 runtime 平台**；Qoder v1.0 在同一基座上套自营网关 + credit 计费 + plugins marketplace，并把 CLI 参数、hooks、插件市场、迁移工具全部**向 Claude Code 的用户表面看齐**，做成闭源商业产品。底座（工具/ContentGenerator/MCP/subagent/沙箱）同源相似；选型看四点——**商业模式**（开源 vs 闭源）、**模型接入**（BYOK 任意 provider vs 自营网关统一计费）、**运行时**（本地 daemon 多端 vs 云端 remote/teleport）、**生态迁移**（从 Gemini/Qwen 系迁移选 Qwen Code，从 Claude Code 迁移 Qoder 接得更顺）。

> **注**：Qwen Code 的 Qwen OAuth 免费层已于 **2026-04-15 终止**（此前 04-13 先从 1000 次/天降到 100 次/天）。现在两者都没有第一方免费额度——但 Qwen Code 仍可 BYOK 接入低价 provider（如 DeepSeek、ModelScope 等）或自部署开源模型。

---

> **免责声明**：Qoder CLI **闭源**，本文 Qoder 侧数据基于 `@qoder-ai/qodercli` v1.0.18 npm bundle 的静态字符串/结构分析（2026-06-11/12），与 qwen-code / gemini-cli 源码三方对照，无 Qoder 源码。bundle 存在 `_$d()` 字符串混淆，命令/工具/参数清单为**可提取下界**，标注 ✓* 的条目语义按名称推断、未经运行验证。"fork 自 Gemini CLI、非 Qwen Code"为三角验证（Gemini 原版名保留 + Qwen 专属标记全 0 + corgi/devtools/沙箱 profile 实物）的强证据推断。Qwen Code 数据来自源码（qwen-code v0.18.0 @ 2026-06-13 HEAD）与本仓库文档。截至 2026-06-13。
