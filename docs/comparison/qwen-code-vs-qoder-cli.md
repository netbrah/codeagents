# Qwen Code vs Qoder CLI 对比

> **核心结论（2026-06 重大变化）**：Qoder CLI **v1.0 已从 Go 二进制完全重写为 Qwen Code 同源的 JS fork**——两者现在是**同属阿里巴巴的 Gemini CLI 血脉兄弟 fork**，共享底座（ContentGenerator / 工具 / MCP / Arena / subagent），但走向截然不同：Qoder 是**闭源商业产品 + 自营模型网关 + 企业/沙箱**，Qwen Code 是**开源免费 + 多 provider + daemon runtime**。
>
> 证据：本地分析 `@qoder-ai/qodercli` v1.0.18（2026-06-11 npm 最新版）bundle。

## TL;DR

| | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 开发方 | 阿里云（Qwen 团队）| 阿里巴巴（Qoder / 通义灵码系）|
| 血脉 | **Gemini CLI fork** | **v1.0 = Qwen Code fork**（v0.x 曾是独立 Go 二进制）|
| 许可 | **Apache-2.0 开源** | **闭源专有**（基于 Apache-2.0 base 做闭源商业产品）|
| 分发 | npm JS（TypeScript）| **v1.0 npm JS（从 Go 二进制重写）** |
| 模型 | Qwen + DashScope/ModelScope + 多 provider（含 DeepSeek 深度适配）| **自营模型网关**（api.qoder.sh / qoder.com.cn）聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek |
| 认证 | Qwen OAuth（**1000 次/天免费**）+ 多 provider API key | 浏览器登录 + **PAT**（自有账号体系，商业计费）|
| 招牌 | **daemon Mode B 多端 runtime** + telemetry + web-shell + 免费 | **macOS Seatbelt 沙箱** + 企业 + 自营网关 + Quest 规格驱动（v0.x）|

## 一、最大发现：Qoder v1.0 是 Qwen Code 的 fork

### v0.x → v1.0 的彻底重写

| | Qoder v0.1.35（2026-03）| Qoder v1.0.18（2026-06-11）|
|---|---|---|
| 语言 | **Go**（静态链接 43MB ELF）| **TypeScript/JS**（34MB esbuild bundle）|
| 内部包 | `code.alibaba-inc.com/qoder-core/qodercli/` | Qwen Code / Gemini CLI 同源 |
| 架构 | 自研 Go 二进制 | **fork 自 Qwen Code** |

**fork 自 Qwen Code 的证据**（v1.0.18 bundle）：
- `contentgenerator` 41 处——Gemini CLI / Qwen Code 的 `ContentGenerator` 抽象
- 工具名全是 Gemini 系：`replace`（1265×，Gemini CLI 的 edit 工具名）/ `task`（197×）/ `shell` / `glob` / `grep` / `run_shell_command` / `web_fetch` / `write_file`
- **`arena`（11×）**——**Arena 多模型竞争是 Qwen Code 的招牌功能**，Qoder 直接继承
- `subagent`（222×）/ `--worktree` / `--yolo` / `approval-mode`——Qwen Code / Gemini CLI 的 flag
- bundle 内含博客链接 `.../blog/qwen-coder-qoder`——直接坐实 Qwen↔Qoder 关系
- postinstall.cjs 自述 "pure JS bundle"，copyright `Google LLC` / Apache-2.0（Gemini CLI 血脉）

> **这是一次阿里内部 fork**：Qoder（通义灵码系）放弃自研 Go 二进制，改 fork 同公司的开源 Qwen Code，再做成闭源商业产品。Apache-2.0 允许这样做，但"开源基座 → 闭源产品"这一步值得注意。

## 二、共享的底座（都来自 Gemini CLI / Qwen Code）

两者共享同一套架构，所以以下能力高度相似：

- **ContentGenerator** 模型抽象 + 多 provider 接入
- **内置工具**：read/write/edit(`replace`)/shell(`run_shell_command`)/glob/grep/web_fetch/read_many_files
- **MCP** 协议扩展（`--mcp-config` / `/mcp` / MCP server 管理）
- **subagent** 子代理 + **Arena** 多模型竞争
- **slash 命令** + 会话 checkpoint/resume + 1M token 上下文
- **非交互模式** `-p` + `--output-format` + `--yolo` + `approval-mode` + `--worktree`
- **`--with-claude-config`**：读 `.claude` 目录的 skills/commands/subagents（Claude Code 兼容）——Qoder v0.x 就有，v1.0 延续

## 三、关键分歧

### 1. 许可与定位
- **Qwen Code**：Apache-2.0 **开源免费**，社区驱动 + 阿里云背书，无商业化
- **Qoder CLI**：**闭源专有商业产品**，自有账号 + PAT + 计费；定位企业/付费

### 2. 模型与认证
- **Qwen Code**：Qwen OAuth（**1000 次/天免费**）/ DashScope / ModelScope / 任意 OpenAI·Anthropic 兼容 provider（DeepSeek 双协议深度适配，详 [Qwen Code 对 DeepSeek 的支持](./qwen-code-deepseek-support.md)）
- **Qoder CLI**：**自营模型网关**（`api.qoder.sh` 全球 / `api.qoder.com.cn` 国内双端点）聚合 Claude/GPT/Gemini/GLM/Kimi/Minimax/DeepSeek，用户通过 Qoder 账号统一计费；浏览器登录 + PAT（CI 用）

### 3. 沙箱安全（Qoder 的差异化投入）
- **Qoder v1.0** 内置 **macOS Seatbelt 沙箱 6 档 profile**（`strict`/`restrictive`/`permissive` × `proxied`/`open`，`(deny default)` + 选择性 allow）——把沙箱做成产品级多档策略（类似 Codex/Gemini CLI 的 seatbelt 路线）
- **Qwen Code** 继承 Gemini CLI 的沙箱能力，但未像 Qoder 这样产品化成 6 档命名 profile

### 4. 运行时形态（Qwen Code 的差异化投入）
- **Qwen Code**：**daemon Mode B**——`qwen serve` 长驻 daemon + 多 client（TUI/web-shell/ACP HTTP/ACP WebSocket/MCP stdio/官方桌面 app），完整 telemetry，是一整套 runtime 平台（详 [daemon 设计](./qwen-code-daemon-design/README.md)）
- **Qoder CLI**：**无 daemon runtime**——是单进程 CLI（有 `remote-control` / ACP 编辑器集成，但没有 Qwen 那套 daemon 多端 multiplexing 架构）

### 5. 各自招牌功能
| Qwen Code 独有 | Qoder CLI 独有 |
|---|---|
| daemon Mode B 多端 runtime | 自营模型网关（统一计费）|
| telemetry 分层 tracing（#3731）| macOS Seatbelt 6 档沙箱 |
| web-shell + 官方桌面 app | 企业部署 + 国内/全球双端点 |
| 免费 OAuth 1000/天 | Quest 规格驱动模式（v0.x，v1.0 存续待确认）|
| Workflow tool（port Claude dynamic workflows） | `/hook-config` / `/plugins` / `/remote-control` |

## 四、横向速览表

| 维度 | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| 血脉 | Gemini CLI fork | **Qwen Code fork**（v0.x Go 二进制 → v1.0 重写）|
| 许可 | Apache-2.0 开源 | 闭源专有 |
| 价格 | 免费（1000/天 OAuth）| 商业计费（自营网关）|
| 模型 | Qwen + 多 provider | 自营网关聚合 8+ 模型 |
| 认证 | Qwen OAuth + API key | 浏览器登录 + PAT |
| 工具 | read/edit/shell/glob/grep/web + MCP | **同**（共享 Gemini 系工具）|
| Arena 多模型 | ✅ 招牌 | ✅ 继承自 Qwen Code |
| subagent | ✅ | ✅ |
| 沙箱 | Gemini CLI 继承 | **macOS Seatbelt 6 档 profile** |
| daemon runtime | ✅ **Mode B 多端** | ❌ 无 |
| telemetry | ✅ 深度（OTel 分层 tracing）| 有 `/otel` `/metrics` 但深度未知 |
| 桌面 app | ✅ 官方（#3778）| Qoder IDE（独立产品线）|
| Claude 配置兼容 | — | `--with-claude-config` |
| 国内/全球 | 单端点 | **双端点**（qoder.com.cn / qoder.sh）|

## 五、一句话总结

**Qoder CLI v1.0 = Qwen Code 的闭源商业化分身**：阿里内部把开源的 Qwen Code fork 出来，套上自营模型网关 + 账号计费 + macOS 沙箱 + 企业能力，做成闭源付费产品。**两者底座（工具/Arena/subagent/MCP/1M 上下文）几乎一样**，差异在**商业模式（开源免费 vs 闭源付费）、模型接入（多 provider vs 自营网关）、运行时（daemon 平台 vs 单进程 CLI）、安全（沙箱产品化）**。对用户：要免费 + 多 provider + daemon 多端选 Qwen Code；要企业统一计费 + 开箱沙箱 + 国内合规选 Qoder。

---

> **免责声明**：Qoder CLI **闭源**，本文基于 `@qoder-ai/qodercli` v1.0.18 npm bundle 的静态字符串/结构分析（2026-06-11），无源码。"fork 自 Qwen Code" 为基于 bundle 内 `arena`/Gemini 系工具/`qwen-coder-qoder` 博客链接/ContentGenerator 等强证据的推断。Quest/Experts 等 v0.x 功能在 v1.0 的存续未逐一验证。模型清单部分来自 v0.x 文档 + v1.0 端点。Qwen Code 数据来自本仓库持续跟踪。截至 2026-06-12。
