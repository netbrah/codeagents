# Qoder CLI 二进制分析证据

## ⚠️ v1.0.18 复核（2026-06-11）：Go 二进制 → Qwen Code fork

**重大变化**：Qoder CLI v1.0 已**从 Go 二进制完全重写为 JS bundle，fork 自 Qwen Code**（Gemini CLI 血脉）。`@qoder-ai/qodercli` v1.0.18 npm 包静态分析证据：

- **包形态**：`bundle/qodercli.js`（34 MB esbuild bundle）+ `postinstall.cjs` 自述 "**pure JS bundle**"；copyright `Google LLC` / Apache-2.0（Gemini CLI 血脉），**不再是 Go 二进制**
- **fork 自 Qwen Code 的证据**（bundle 字符串）：`contentgenerator` 41× / 工具名全 Gemini 系（`replace` 1265× · `task` 197× · `shell` · `glob` 43× · `grep` 23× · `run_shell_command` · `web_fetch`）/ **`arena` 11×（Qwen Code 招牌多模型）** / `subagent` 222× / 博客链接 `.../blog/qwen-coder-qoder`
- **新增 macOS Seatbelt 沙箱**：6 个 `.sb` profile（`strict`/`restrictive`/`permissive` × `proxied`/`open`，`(deny default)` + 选择性 allow）
- **Qoder 自有**：浏览器登录 + PAT（`QODER_PERSONAL_ACCESS_TOKEN`）；自营网关 `api.qoder.sh`（全球）/ `api.qoder.com.cn`（国内）；`qoder-plugin` / `qoder-marketplace` / `qoder-enterprise-` / `remote-control`
- **slash 命令**：`/help` `/mcp` `/mcp-config` `/hook-config` `/plugins` `/usage` `/remote-control` `/token` `/authorize` `/compare` `/metrics` `/otel`
- **依赖**：`sharp`（图像/多模态）+ ripgrep + Node ≥ 20；两渠道 `@latest` / `@beta`（Node 原生兼容版）

完整对比见 [Qwen Code vs Qoder CLI](../../comparison/qwen-code-vs-qoder-cli.md)。

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
