# 4. Qwen Code 工具系统——贡献者参考

> **24 个默认可用工具 + 17 个条件工具（共 41）** + MCP 动态工具（v0.18.0）。早期缺失的 ToolSearch（延迟加载）、Cron 系列（定时，#4950 起默认启用）、NotebookEdit（Jupyter）均已落地；新增 EnterPlanMode + Plan Approval Gate（#4853）；Agent Team、workflow、computer use 工具族为条件启用。

源码: `packages/core/src/config/config.ts#createToolRegistry`（工具注册）
> 工具名称定义: `packages/core/src/tools/tool-names.ts`

## 工具注册机制

| 来源 | 注册方式 | 位置 |
|------|---------|------|
| 核心工具 | `createToolRegistry` 懒加载工厂注册（首次使用才 `import()`） | `packages/core/src/tools/*.ts` |
| MCP 工具 | `McpClientManager` 动态发现 | `packages/core/src/tools/mcp-client-manager.ts` |
| 发现的工具 | `ToolRegistry` 运行时注册 | `packages/core/src/tools/tool-registry.ts` |

> `--bare` 模式只注册 4 个工具（read_file/edit/notebook_edit/run_shell_command）。

## 默认可用工具（24 个）

### 文件操作

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **ReadFile** | Read | 读取文件内容，支持行范围、图片/PDF | `file_path`, `offset?`, `limit?` | `tools/read-file.ts` |
| **WriteFile** | Edit | 创建/覆写文件 | `file_path`, `content` | `tools/write-file.ts` |
| **Edit** | Edit | 精确文本替换（old_string → new_string） | `file_path`, `old_string`, `new_string`, `replace_all?` | `tools/edit.ts` |
| **NotebookEdit** | Edit | Jupyter notebook 单元格编辑 | `notebook_path`, … | `tools/notebook-edit.ts` |
| **ListFiles** | Read | 列出目录内容 | `path`, `ignore?`, `file_filtering_options?` | `tools/ls.ts` |

### 搜索

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **Grep**（`grep_search`） | Search | 正则搜索——ripgrep 优先，多级回退 | `pattern`, `path?`, `glob?`, `limit?` | `tools/ripGrep.ts` / `tools/grep.ts` |
| **Glob** | Search | 文件模式匹配 | `pattern`, `path?` | `tools/glob.ts` |
| **ToolSearch** | Read | 搜索延迟加载的工具并按需载入 schema | `query`, `max_results?` | `tools/tool-search.ts` |

### 执行

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **Shell** | Execute | 执行 Shell 命令 | `command`, `is_background?`, `timeout?`, `description?`, `directory?` | `tools/shell.ts` |
| **EnterWorktree** / **ExitWorktree** | Other | 运行中进出隔离 git worktree | — | `tools/enter-worktree.ts` 等 |

### 网络

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **WebFetch** | Fetch | 抓取 URL 内容并用 AI 处理 | `url`, `prompt` | `tools/web-fetch.ts` |

> 早期的 **WebSearch**（Tavily/Google/DashScope 多 provider）与 **SaveMemory** 工具已从注册表移除——记忆改为托管自动记忆体系（`/remember` `/forget` `/dream` 命令）。

### 任务管理

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **TodoWrite** | Other | 创建/管理结构化任务列表 | `todos[]` | `tools/todoWrite.ts` |
| **TaskStop** | Other | 停止后台任务 | `task_id` | `tools/task-stop.ts` |
| **Monitor** | Other | 监控后台条件/进程 | — | `tools/monitor.ts` |
| **CronCreate / CronList / CronDelete** | Other | 定时任务调度（#4950 起默认启用，`QWEN_CODE_DISABLE_CRON=1` 关闭） | — | `tools/cron-*.ts` |

### 代理与技能

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **Agent** | Other | 启动子代理处理复杂任务 | `skill?`, 动态参数 | `tools/agent.ts` |
| **SendMessage** | Other | 向其他代理发送消息（Agent Team） | — | `tools/send-message.ts` |
| **Skill** | Read | 执行技能 | `skill` | `tools/skill.ts` |
| **EnterPlanMode** | Other | 进入规划模式 + Plan Approval Gate（SDK 模式下不注册，#4853） | （无） | `tools/enterPlanMode.ts` |
| **ExitPlanMode** | Other | 退出规划模式（SDK 模式下不注册） | （无） | `tools/exitPlanMode.ts` |

### 交互

| 工具 | Kind | 用途 | 关键参数 | 源码 |
|------|------|------|---------|------|
| **AskUserQuestion** | Other | 执行期间向用户提问 | `questions[]` | `tools/askUserQuestion.ts` |

## 条件工具（17 个）

| 工具 | 条件 | 源码 |
|------|------|------|
| **LSP** | `isLspEnabled()` 且 LSP client 就绪 | `tools/lsp.ts` |
| **TeamCreate / TeamDelete / TaskCreate / TaskUpdate / TaskList** | `isAgentTeamEnabled()`（Agent Team 多代理协作） | `tools/team-*.ts`, `tools/task-*.ts` |
| **Workflow** | `isWorkflowsEnabled()`（多代理工作流脚本编排） | `tools/workflow.ts` |
| **ComputerUse ×9**（list_apps/click/scroll/drag/type_text/press_key 等） | `isComputerUseEnabled()`（上游 MCP server 实现，按需揭示） | `tools/computer-use/` |
| **StructuredOutput** | `--json-schema` 运行时注册（结构化输出终点合约） | `tools/syntheticOutput.ts` |

## MCP 动态工具

MCP 工具通过 `McpClientManager` 在运行时动态发现：

```
McpClientManager → McpClient.list_tools() → DiscoveredMCPTool[]
```

MCP 工具以 `mcp__serverName__toolName` 格式注册到 `ToolRegistry`。

## 工具权限分类

每个工具有 Kind 属性，影响权限行为：

| Kind | 默认权限行为 | 工具（代表） |
|------|-------------|------|
| Read | 较宽松 | ReadFile, ListFiles, Grep, Glob, Skill, ToolSearch |
| Edit | 需确认 | WriteFile, Edit, NotebookEdit |
| Execute | 严格确认 | Shell |
| Fetch | 需确认 | WebFetch |
| Other | 视情况 | TodoWrite, Agent, ExitPlanMode, LSP, AskUserQuestion, Cron/Task/Team 族 |

## 工具调度管线

源码: `packages/core/src/core/coreToolScheduler.ts`（1790 行）

```
LLM 工具调用请求
    │
    ▼
参数校验 → validating 状态
    │
    ▼
权限检查 → getDefaultPermission()
    │  deny > ask > allow > default
    ▼
Hook 触发 → PreToolUse
    │
    ▼
用户确认 → awaiting_approval（plan/default 模式）
    │
    ▼
工具执行 → executing 状态
    │  支持实时输出流、超时控制
    ▼
Hook 触发 → PostToolUse / PostToolUseFailure
    │
    ▼
结果返回 LLM → 继续循环或完成
```
