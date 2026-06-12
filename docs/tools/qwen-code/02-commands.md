# 2. Qwen Code 命令系统——贡献者参考

> **57 个内置命令**（v0.18.0：52 个常驻 + 5 个条件启用）+ **7 个 bundled skill 命令** + MCP prompts + 用户自定义。早期缺失的 /plan、/rewind 等已落地。
>
> 源码: `packages/cli/src/services/BuiltinCommandLoader.ts`（注册所有内置命令）
>
> **注**: 条件启用的 5 个命令——`/ide`（IDE 检测成功）、`/trust`（folderTrust 开启）、`/lsp`（LSP 开启）、`/dream` `/forget`（托管自动记忆开启）。`/loop` 已从内置命令转为 bundled skill。

## 命令加载机制

命令有 4 个来源（按优先级）：

| 来源 | 加载器 | 位置 |
|------|--------|------|
| 内置命令 | `BuiltinCommandLoader` | `packages/cli/src/ui/commands/*.ts` |
| MCP 提示 | `McpPromptLoader` | MCP 服务器暴露的 prompts |
| 技能命令 | `BundledSkillLoader` | `.qwen/skills/` |
| 文件命令 | `FileCommandLoader` | `.qwen/commands/*.md` 或 `~/.qwen/commands/*.md` |

## 内置命令一览

### 核心命令

| 命令 | 别名 | 用途 | 源码 |
|------|------|------|------|
| `/help` | `?` | 显示帮助信息 | `commands/helpCommand.ts` |
| `/status` | `about` | 显示版本信息 | `commands/aboutCommand.ts` |
| `/auth` | `login` | 管理认证与登录 | `commands/authCommand.ts` |
| `/model` | — | 切换模型 | `commands/modelCommand.ts` |
| `/clear` | `reset`, `new` | 清除对话历史 | `commands/clearCommand.ts` |
| `/compress` | `summarize` | 压缩上下文 | `commands/compressCommand.ts` |
| `/compress-fast` | — | 无 LLM 的规则式上下文压缩 | `commands/compressFastCommand.ts` |
| `/context` | — | 查看 token 用量 | `commands/contextCommand.ts` |
| `/copy` | — | 复制上次回复 | `commands/copyCommand.ts` |
| `/tools` | — | 查看可用工具列表 | `commands/toolsCommand.ts` |
| `/settings` | — | 查看/修改设置 | `commands/settingsCommand.ts` |
| `/permissions` | — | 管理权限规则 | `commands/permissionsCommand.ts` |
| `/memory` | — | 查看/编辑记忆 | `commands/memoryCommand.ts` |
| `/remember` | — | 保存持久记忆 | `commands/rememberCommand.ts` |
| `/forget` | — | 删除匹配的托管记忆（条件启用） | `commands/forgetCommand.ts` |
| `/dream` | — | 整理托管自动记忆主题文件（条件启用） | `commands/dreamCommand.ts` |
| `/mcp` | — | 管理 MCP 服务器 | `commands/mcpCommand.ts` |

### 会话管理

| 命令 | 别名 | 用途 | 源码 |
|------|------|------|------|
| `/restore` | — | 恢复历史会话检查点 | `commands/restoreCommand.ts` |
| `/resume` | — | 继续上次会话 | `commands/resumeCommand.ts` |
| `/rewind` | `rollback` | 回滚对话到之前的轮次 | `commands/rewindCommand.ts` |
| `/branch` | — | 把当前对话 fork 到新会话 | `commands/branchCommand.ts` |
| `/fork` | — | 派生继承全部对话的后台代理 | `commands/forkCommand.ts` |
| `/rename` | `tag` | 会话改名/打标签 | `commands/renameCommand.ts` |
| `/delete` | — | 删除历史会话 | `commands/deleteCommand.ts` |
| `/recap` | — | 生成一行会话回顾 | `commands/recapCommand.ts` |
| `/export` | — | 导出当前会话 | `commands/exportCommand.ts` |
| `/quit` | `exit` | 退出 | `commands/quitCommand.ts` |

### 开发辅助

| 命令 | 别名 | 用途 | 源码 |
|------|------|------|------|
| `/agents` | — | 管理子代理 | `commands/agentsCommand.ts` |
| `/skills` | — | 查看可用技能 | `commands/skillsCommand.ts` |
| `/approval-mode` | — | 切换审批模式（plan/default/auto-edit/yolo） | `commands/approvalModeCommand.ts` |
| `/plan` | — | 进入/退出计划模式 | `commands/planCommand.ts` |
| `/goal` | — | 设定目标——达成条件前持续工作 | `commands/goalCommand.ts` |
| `/tasks` | — | 后台任务调度入口 | `commands/tasksCommand.ts` |
| `/stats` | `usage` | 显示统计信息 | `commands/statsCommand.ts` |
| `/diff` | — | 显示工作区相对 HEAD 的变更统计 | `commands/diffCommand.ts` |
| `/doctor` | — | 安装与环境诊断 | `commands/doctorCommand.ts` |
| `/editor` | — | 设置外部编辑器 | `commands/editorCommand.ts` |
| `/hooks` | — | 管理 Hook 配置 | `commands/hooksCommand.ts` |
| `/init` | — | 初始化项目配置（生成 QWEN.md） | `commands/initCommand.ts` |
| `/trust` | — | 管理信任设置（条件启用） | `commands/trustCommand.ts` |
| `/lsp` | — | LSP 服务器状态（条件启用） | `commands/lspCommand.ts` |
| `/summary` | — | 生成对话摘要 | `commands/summaryCommand.ts` |
| `/setup-github` | — | 设置 GitHub Actions | `commands/setupGithubCommand.ts` |

### 终端与 UI

| 命令 | 别名 | 用途 | 源码 |
|------|------|------|------|
| `/theme` | — | 切换颜色主题 | `commands/themeCommand.ts` |
| `/statusline` | — | 配置状态栏 | `commands/statuslineCommand.ts` |
| `/vim` | — | 切换 Vim 编辑模式 | `commands/vimCommand.ts` |
| `/terminal-setup` | — | 配置终端集成 | `commands/terminalSetupCommand.ts` |
| `/ide` | — | IDE 集成管理 | `commands/ideCommand.ts` |
| `/directory` | `dir` | 目录管理 | `commands/directoryCommand.tsx` |
| `/cd` | — | 把当前会话迁移到新工作目录 | `commands/cdCommand.ts` |
| `/language` | — | 切换 UI 语言 | `commands/languageCommand.ts` |

### 信息与反馈

| 命令 | 别名 | 用途 | 源码 |
|------|------|------|------|
| `/bug` | — | 报告 Bug | `commands/bugCommand.ts` |
| `/docs` | — | 打开文档 | `commands/docsCommand.ts` |

### Qwen Code 独有命令

| 命令 | 用途 | 源码 |
|------|------|------|
| `/arena` | **Arena 模式**——多模型在隔离 Git worktree 中竞争执行 | `commands/arenaCommand.ts` |
| `/language` | **切换 UI 语言**（中/英/日/德/俄/葡） | `commands/languageCommand.ts` |
| `/insight` | **代码洞察**——分析代码库生成个性化洞察 | `commands/insightCommand.ts` |
| `/extensions` | **扩展管理**——安装/卸载/列表 | `commands/extensionsCommand.ts` |
| `/btw` | **快速旁问**——不中断主对话的侧边提问 | `commands/btwCommand.ts` |

### Bundled Skill 命令（7 个）

源码: `packages/core/src/skills/bundled/*/SKILL.md`

| 命令 | 用途 |
|------|------|
| `/review` | 代码审查（四代理并行：Correctness & Security / Code Quality / Performance / Undirected Audit） |
| `/loop` | 按计划循环执行 prompt（`/loop 5m check the build`；原 v0.13 内置命令，已转为 skill） |
| `/batch` | 多文件并行批量操作（自动发现文件、分块执行） |
| `/simplify` | 审查近期变更的复用/质量/效率问题并直接应用清理 |
| `/stuck` | 诊断卡死/变慢的 Qwen Code 会话（扫描问题进程、高 CPU） |
| `/new-app` | 从零创建应用的工作流（需求收集、技术选型、脚手架） |
| `/qc-helper` | 基于官方文档回答 Qwen Code 用法/配置/排障问题 |

## 子命令

| 父命令 | 子命令 | 用途 |
|--------|--------|------|
| `/agents` | `manage` | 管理子代理（查看/编辑/删除） |
| `/agents` | `create` | 创建新子代理 |
| `/arena` | `start` | 启动 Arena 会话 |
| `/arena` | `stop` | 停止 Arena 会话 |
| `/arena` | `status` | 查看 Arena 状态 |
| `/arena` | `select` / `choose` | 选择最佳结果并合并 diff |
| `/directory` | `add` | 添加目录到工作区 |
| `/directory` | `show` | 显示所有工作区目录 |
| `/export` | `html` / `md` / `json` / `jsonl` | 导出为不同格式 |
| `/extensions` | `explore` | 打开扩展市场 |
| `/extensions` | `install` | 安装扩展 |
| `/extensions` | `manage` | 管理已安装扩展 |
| `/hooks` | `list` / `enable` / `disable` | 列出/启用/禁用 Hook |
| `/ide` | `status` / `install` / `enable` / `disable` | IDE 集成管理 |
| `/language` | `ui` | 设置 UI 语言 |
| `/memory` | `show` / `add` / `refresh` | 查看/添加/刷新记忆 |
| `/stats` | `model` / `tools` | 模型/工具使用统计 |

## 其他命令类型

> 来源：[官方文档](https://qwenlm.github.io/qwen-code-docs/zh/users/features/commands/)

除斜杠命令外，Qwen Code 还支持 3 种命令类型：

### @ 命令（文件/目录注入）

```bash
@src/auth/login.ts       # 将文件内容注入为上下文
@src/components/          # 递归读取目录下所有文本文件注入为上下文
```

### ! 命令（Shell 执行）

```bash
!git status              # 直接执行 Shell 命令
!npm test                # 运行测试
!                        # 单独 ! 进入 Shell 模式
```

### 自定义命令（Markdown 文件）

在 `~/.qwen/commands/`（用户级）或 `.qwen/commands/`（项目级）放置 `.md` 文件：

```markdown
---
name: deploy
description: 部署到测试环境
---

请执行以下部署步骤：
1. 运行 !{npm run build} 构建项目
2. 读取 @{deploy.config.json} 获取配置
3. 使用 {{args}} 作为目标环境参数
```

支持变量（处理顺序：`@{...}` → `!{...}` → `{{args}}`）：
- `{{args}}`——参数注入
- `!{command}`——Shell 命令执行
- `@{filepath}`——文件内容注入

**目录映射规则**：`commands/git/commit.md` → `/git:commit`

> **注**：TOML 格式命令文件已弃用但仍受支持，便于旧命令迁移。`!` 命令执行时自动设置 `QWEN_CODE=1` 环境变量。`@` 路径含空格时用 `\` 转义（如 `@My\ Documents/file.txt`）。

项目级命令优先于用户级。

## CLI 参数

```bash
# 交互模式
qwen

# 直接提问
qwen "解释这段代码"

# 非交互模式
qwen --non-interactive --prompt "重构 auth 模块"

# 使用特定模型
qwen --model qwen3-coder-plus

# 指定 API Key
qwen --api-key $DASHSCOPE_API_KEY

# 恢复会话
qwen --resume

# 指定会话 ID
qwen --session-id <id>
```

## 与 Gemini CLI 命令对比

| 命令 | Gemini CLI | Qwen Code | 差异 |
|------|-----------|-----------|------|
| `/arena` | ❌ | ✅ | Qwen 新增 |
| `/language` | ❌ | ✅ | Qwen 新增 |
| `/insight` | ❌ | ✅ | Qwen 新增 |
| `/extensions` | ❌ | ✅ | Qwen 新增（Gemini 用 `/mcp`） |
| `/btw` | ❌ | ✅ | Qwen 新增 |
| `/approval-mode` | `/plan` | `/approval-mode` | 重命名，增加 auto-edit/yolo |
| 其余命令 | ✅ | ✅ | 继承，功能一致 |
