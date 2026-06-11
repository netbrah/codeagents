# Qwen Code vs Qoder CLI 对比

> **核心结论（2026-06 复核）**：Qoder CLI **v1.0 已从 v0.x 的 Go 二进制完全重写为 JS bundle，基座是 Gemini CLI**。它与 Qwen Code 是 **Gemini CLI 的两个兄弟 fork**（均为阿里系产品），但 **Qoder 不是 Qwen Code 的 fork**——经 gemini-cli / qwen-code / Qoder bundle 三方字符串三角验证：Qoder 保留的是 **Gemini CLI 原版内部名**（`geminiIgnoreFilter`、`GEMINI.md`、`MemoryDiscovery`），而 Qwen Code 专属标记（`DashScope` / `ModelScope` / `chat.qwen.ai` / `qwenOAuth` / `QwenContentGenerator` / `QWEN.md`）在 Qoder bundle 中**全部为 0**。
>
> 证据：本地分析 `@qoder-ai/qodercli` v1.0.18（2026-06-11 npm 最新版）的 34 MB esbuild bundle，与 qwen-code / gemini-cli 源码三方对照。

## TL;DR

| | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 开发方 | 阿里云（Qwen / 通义团队）| 阿里（Qoder，v0.x 内部包 `code.alibaba-inc.com/qoder-core`）|
| 血脉 | **Gemini CLI fork** | **Gemini CLI fork**（v1.0；v0.x 曾是独立 Go 二进制）|
| 与对方关系 | — | **不是 Qwen Code 的 fork**，是 Gemini CLI 的兄弟 fork |
| 许可 | **Apache-2.0 开源** | **闭源专有**（bundle 内 Apache-2.0/Google LLC 为 Gemini CLI 基座遗留版权）|
| 分发 | npm JS（TypeScript）| **v1.0 npm JS**（34 MB esbuild bundle；v0.x 为 Go ELF）|
| 模型/认证 | Qwen + 阿里云 Coding Plan 订阅 + 任意 OpenAI·Anthropic 兼容 provider（BYOK，含 DeepSeek 深度适配）| **自营模型网关**（`api.qoder.sh` 全球 / `api.qoder.com.cn` 国内）聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek；浏览器登录 + PAT |
| 价格 | **开源软件免费，但模型调用需自付**（Qwen OAuth 免费层 **2026-04-15 已停**）| 闭源商业产品，Qoder 账号统一计费 |
| 招牌 | **daemon Mode B 多端 runtime** + telemetry + web-shell | **macOS Seatbelt 沙箱** + 企业 + 自营网关 + 国内/全球双端点 |

## 一、最大发现：Qoder v1.0 是 Gemini CLI fork，但**不是** Qwen Code fork

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
- Qoder bundle 保留了 **Gemini CLI 的原版内部名**（`geminiIgnoreFilter`、`GEMINI.md`、`MemoryDiscovery`、`ContentGenerator`），这些在 Qwen Code 里已被改掉——**说明 Qoder 是直接从 Gemini CLI fork，而非经过 Qwen Code**。
- **所有 Qwen 专属标记在 Qoder bundle 中均为 0**（`DashScope`/`ModelScope`/`chat.qwen.ai`/`qwenOAuth`/`QwenContentGenerator`/`QWEN.md`）。这些是字符串字面量，不会被 esbuild minify 改写，**为 0 即真实缺失**。若 Qoder fork 自 Qwen Code，至少应残留部分。
- Qoder 自有品牌密度极高：`qoder` 747× / `.qoder` 131× / `api.qoder` 11×，配置文件 `.qoderignore` / `.qoder.json`，上下文文件用 **`AGENTS.md`**（+ `SKILL.md` / `MEMORY.md`）。

> **更正说明**：早前曾据 `arena`（11×）与 `contentgenerator`/`replace`（千次）推断 Qoder fork 自 Qwen Code——经复核，**`arena` 实为 protobuf 描述符选项 `cc_enable_arenas`/`ccEnableArenas`**（Google Protocol Buffers C++ 生成码），与 Qwen 的「Arena 多模型」无关；`replace` 多为 `String.prototype.replace` 与 protobuf `set="replace"`。这些不能作为 fork 证据。本文表格为重新逐项核验后的结论。

## 二、共享的底座（都来自 Gemini CLI）

两者都是 Gemini CLI 的 fork，所以以下能力同源、高度相似——但这是**共同祖先**带来的，不是谁 fork 了谁：

- **ContentGenerator** 模型抽象 + 多 provider 接入
- **内置工具**：read/write/edit/`run_shell_command`/glob/grep/`web_fetch`/read_many_files
- **MCP** 协议扩展（`--mcp-config` / `/mcp` / MCP server 管理）
- **subagent** 子代理（gemini-cli 与 qwen-code 均有，Qoder bundle 内 98×）
- **slash 命令** + 会话 checkpoint/resume + 大上下文
- **非交互模式** `-p` + `--output-format` + `--yolo` + `approval-mode` + `--worktree`
- **`--with-claude-config`**：读 `.claude` 目录的 skills/commands/subagents（Claude Code 兼容）——Qoder v0.x 就有，v1.0 延续

## 三、关键分歧

### 1. 许可与定位
- **Qwen Code**：Apache-2.0 **开源**，社区驱动 + 阿里云背书；软件本身免费，但模型调用需自付
- **Qoder CLI**：**闭源专有商业产品**，自有账号 + PAT + 计费；定位企业/付费

### 2. 模型与认证（都不再有第一方免费额度）
- **Qwen Code**：三种认证——① Qwen OAuth（**免费层 2026-04-15 已停**，新请求被拒）② 阿里云 Coding Plan（付费月度订阅，含 Qwen/GLM/Kimi/Minimax）③ BYOK API key（OpenAI/Anthropic/Gemini 兼容，DeepSeek 双协议深度适配，详 [Qwen Code 对 DeepSeek 的支持](./qwen-code-deepseek-support.md)）。**无厂商锁定，可接任意 provider**
- **Qoder CLI**：**自营模型网关**（`api.qoder.sh` 全球 / `api.qoder.com.cn` 国内双端点）聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek，用户通过 Qoder 账号统一计费；浏览器登录 + PAT（CI 用）

### 3. 沙箱安全（Qoder 的差异化投入）
- **Qoder v1.0** 内置 **macOS Seatbelt 沙箱 6 档 profile**（`strict`/`restrictive`/`permissive` × `proxied`/`open`，`(deny default)` + 选择性 allow）——把沙箱做成产品级多档策略
- **Qwen Code** 继承 Gemini CLI 的沙箱能力，但未像 Qoder 这样产品化成 6 档命名 profile

### 4. 运行时形态（Qwen Code 的差异化投入）
- **Qwen Code**：**daemon Mode B**——`qwen serve` 长驻 daemon + 多 client（TUI/web-shell/ACP HTTP/ACP WebSocket/MCP stdio/官方桌面 app），完整 telemetry，是一整套 runtime 平台（详 [daemon 设计](./qwen-code-daemon-design/README.md)）
- **Qoder CLI**：**无 daemon runtime**——单进程 CLI（有 `remote-control` / ACP 编辑器集成，但没有 Qwen 那套 daemon 多端 multiplexing 架构）

### 5. 各自招牌功能
| Qwen Code 独有 | Qoder CLI 独有 |
|---|---|
| daemon Mode B 多端 runtime | 自营模型网关（统一计费）|
| telemetry 分层 tracing（#3731）| macOS Seatbelt 6 档沙箱 |
| web-shell + 官方桌面 app | 企业部署 + 国内/全球双端点 |
| DashScope/ModelScope/Qwen OAuth 原生接入 | Quest 规格驱动模式（v0.x，v1.0 存续待确认）|
| Arena 多模型竞争 + Workflow tool | `/hook-config` / `/plugins` / `/remote-control` |

## 四、横向速览表

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 血脉 | Gemini CLI fork | Gemini CLI fork（**非** Qwen Code fork）|
| 许可 | Apache-2.0 开源 | 闭源专有 |
| 价格 | 软件免费，模型自付（OAuth 免费层已停）| 商业计费（自营网关）|
| 模型 | Qwen + Coding Plan + 任意 provider（BYOK）| 自营网关聚合 8+ 模型 |
| 认证 | Qwen OAuth〔停〕/ Coding Plan / API key | 浏览器登录 + PAT |
| 工具 | read/edit/shell/glob/grep/web + MCP | **同源**（共享 Gemini CLI 工具）|
| subagent | ✅ | ✅（继承自 Gemini CLI）|
| Arena 多模型 | ✅ 招牌 | ❌（bundle 无 Arena；"arena" 实为 protobuf）|
| 沙箱 | Gemini CLI 继承 | **macOS Seatbelt 6 档 profile** |
| daemon runtime | ✅ **Mode B 多端** | ❌ 无 |
| telemetry | ✅ 深度（OTel 分层 tracing）| 有 `/otel` `/metrics` 但深度未知 |
| 桌面 app | ✅ 官方（#3778）| Qoder IDE（独立产品线）|
| 上下文文件 | `QWEN.md` | `AGENTS.md`（+ SKILL.md / MEMORY.md）|
| Claude 配置兼容 | — | `--with-claude-config` |
| 国内/全球 | 单端点 | **双端点**（qoder.com.cn / qoder.sh）|

## 五、一句话总结

**Qwen Code 与 Qoder CLI v1.0 是 Gemini CLI 的两个兄弟 fork，都出自阿里系，但互不衍生**：Qoder v1.0 把自研的 Go 二进制（v0.x）推倒，改从 **Gemini CLI** 重写，套上自营模型网关 + 账号计费 + macOS 沙箱 + 企业能力，做成闭源付费产品；Qwen Code 则在 Gemini CLI 基础上加 DashScope/ModelScope/Qwen OAuth/Arena/daemon，走开源路线。**两者底座（工具/ContentGenerator/MCP/subagent/大上下文）同源相似**，差异在**商业模式（开源 vs 闭源）、模型接入（BYOK 任意 provider vs 自营网关统一计费）、运行时（daemon 平台 vs 单进程 CLI）、安全（沙箱产品化）**。对用户：要开源 + BYOK 任意 provider + daemon 多端选 Qwen Code；要企业统一计费 + 开箱沙箱 + 国内合规选 Qoder。

> **注**：Qwen Code 的 Qwen OAuth 免费层已于 **2026-04-15 终止**（此前 04-13 先从 1000 次/天降到 100 次/天）。现在两者都没有第一方免费额度——但 Qwen Code 仍可 BYOK 接入低价 provider（如 DeepSeek、ModelScope 等）或自部署开源模型。

---

> **免责声明**：Qoder CLI **闭源**，本文基于 `@qoder-ai/qodercli` v1.0.18 npm bundle 的静态字符串/结构分析（2026-06-11），与 qwen-code / gemini-cli 源码三方对照，无 Qoder 源码。"fork 自 Gemini CLI、非 Qwen Code" 为基于上表三角验证（Gemini 原版名保留 + Qwen 专属标记全 0）的强证据推断。Quest/Experts 等 v0.x 功能在 v1.0 的存续未逐一验证。Qwen Code 数据来自本仓库持续跟踪与源码（`docs/users/configuration/auth.md`、`README.md`）。截至 2026-06-12。
