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
- **新增 macOS Seatbelt 沙箱**：6 个 `.sb` profile（`strict`/`restrictive`/`permissive` × `proxied`/`open`，`(deny default)` + 选择性 allow）
- **Qoder 自有**：浏览器登录 + PAT（`QODER_PERSONAL_ACCESS_TOKEN`）；自营网关 `api.qoder.sh`（全球）/ `api.qoder.com.cn`（国内）；`qoder-plugin` / `qoder-marketplace` / `qoder-enterprise-` / `remote-control`
- **slash 命令**：`/help` `/mcp` `/mcp-config` `/hook-config` `/plugins` `/usage` `/remote-control` `/token` `/authorize` `/compare` `/metrics` `/otel`
- **依赖**：`sharp`（图像/多模态）+ ripgrep + Node ≥ 20；两渠道 `@latest` / `@beta`

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
