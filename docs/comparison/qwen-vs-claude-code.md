# 18. Qwen Code vs Claude Code: In-Depth Comparison

> Qwen Code (open source, Gemini CLI fork) vs Claude Code (closed source, native Rust) -- a comprehensive comparison of two leading AI coding agents.

## Positioning Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Developer** | Anthropic | Alibaba Cloud |
| **Positioning** | Anthropic's official terminal agent | Alibaba Cloud open-source terminal agent |
| **Language** | Rust (closed source) | TypeScript (open source, Gemini CLI fork) |
| **License** | Proprietary | Apache-2.0 |
| **Models** | Claude series (locked) | Qwen OAuth + DashScope + ModelScope + Anthropic + Google + custom (6+ providers) |
| **Context** | 1M tokens (Opus 4.6) | 1M tokens (Qwen3 commercial edition) |
| **Billing** | Included in Claude Pro/Max subscriptions | Open-source software is free; pay for models yourself (OAuth free tier ended 2026-04-15; requires a Coding Plan subscription or BYOK) |
| **Plugin repository** | 13 official plugins | Extension system + Claude/Gemini format conversion |

---

## 1. Agent Loop

### Claude Code

```
User input
  -> System prompt + CLAUDE.md project instructions
  -> Claude LLM (streaming)
  -> Tool-call parsing
  -> PreToolUse Hook (validation/modification)
  -> Permission check (allow/ask/deny)
  -> Tool execution (possibly sandboxed)
  -> PostToolUse Hook (feedback)
  -> Results returned to LLM
  -> Repeat until complete
  -> Stop Hook (validate completion reasonableness)
```

- **REPL mode**: interactive sessions with streaming responses
- **Subagents**: spawn autonomous agents through the `Task` tool
- **Plan mode**: plan step by step after the user request, then execute
- **Automatic memory**: learns user preferences across sessions

### Qwen Code

```
User input
  -> System prompt + system.md project instructions
  -> ContentGenerator (multi-provider)
  -> CoreToolScheduler scheduling
  -> Hook trigger (PreToolUse)
  -> PermissionManager check
  -> Tool execution (possibly sandboxed)
  -> Hook trigger (PostToolUse)
  -> Results returned to LLM
  -> Loop detection (Levenshtein)
  -> Repeat, up to 100 turns
```

- **MAX_TURNS = 100**: hard turn limit
- **Arena mode**: multi-agent competition/collaboration
- **Token limit**: session-level token budget
- **Session recording**: JSONL persistence

### Key Differences

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| Turn limit | No explicit limit | 100 turns |
| Loop detection | Stop Hook validation | Levenshtein distance detection |
| Subagents | Task tool + Agent definitions | Subagent manager + Arena |
| Plan mode | User-triggered | Plan mode + approval workflow |
| Project instructions | CLAUDE.md | system.md |
| Session resume | `--resume` flag | `getResumedSessionData()` |

---

## 2. Tool System

### Claude Code Built-in Tools

| Agent | Description | Default permission |
|------|------|---------|
| **Read** | Read files | allow |
| **Write** | Create/write files | ask |
| **Edit** | Modify files (diff preview) | ask |
| **Bash** | Execute Shell commands | ask |
| **Glob** | File pattern matching | allow |
| **Grep** | Regex content search | allow |
| **WebFetch** | Fetch URL content | ask |
| **WebSearch** | Web search | ask |
| **AskUserQuestion** | Ask the user a question | allow |
| **Task** | Spawn subagents | allow |
| **Skill** | Load skills | allow |
| **TodoWrite** | Task management | allow |
| **NotebookEdit** | Jupyter Notebook | ask |
| **MCP tools** | External MCP servers | Configurable |

### Qwen Code Built-in Tools

| Agent | Description | Default permission |
|------|------|---------|
| **edit** | Multi-part file editing (diff) | ask |
| **write_file** | Create/overwrite files | ask |
| **read_file** | Read files | allow |
| **run_shell_command** | Shell execution | ask |
| **grep_search** | Full-text search (ripgrep) | allow |
| **glob** | File pattern matching | allow |
| **list_directory** | Directory listing | allow |
| **web_fetch** | HTTP GET requests | ask |
| **web_search** | Web search (Tavily/Google/DashScope) | ask |
| **agent** | Spawn subagents | allow |
| **skill** | Skill invocation | allow |
| **todo_write** | Task management | allow |
| **save_memory** | Persist memory to Markdown | allow |
| **exit_plan_mode** | Exit planning mode | allow |
| **ask_user_question** | User interaction | allow |
| **lsp** | Language server operations | allow |
| **MCP tools** | External MCP servers | Configurable |

### Tool Implementation Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Editing tool** | Edit (`old_string`/`new_string`) | edit (multi-part diff + encoding detection + BOM) |
| **Shell** | Bash (4 modes: normal/sandbox/pipeline/interactive) | run_shell_command (AST analysis + Tmux/iTerm2) |
| **Search** | Grep (ripgrep syntax) | grep_search (ripgrep + gitignore) |
| **Web search backend** | Built-in | Tavily/Google/DashScope (multi-backend) |
| **Unique tools** | NotebookEdit | save_memory, exit_plan_mode, lsp |
| **Tool definition** | Rust built-in (closed source) | DeclarativeTool abstract class (TypeScript) |
| **Streaming output** | Built-in support | `updateOutput` callback |
| **Tool discovery** | Built-ins + MCP only | Command-line discovery + MCP + extensions |

### Edit Tool Deep Comparison

**Claude Code Edit**:
- `old_string` / `new_string` search-and-replace
- Optional global replacement with `replace_all`
- Diff preview; user can modify before confirming

**Qwen Code edit**:
- Same `old_string` / `new_string` pattern
- Automatic encoding detection + BOM handling
- `safeLiteralReplace()` safely handles `$` sequences
- CRLF -> LF normalization
- Occurrence count validation (ensures uniqueness)
- `modifyWithEditor()` allows intermediate editing

### Shell Tool Deep Comparison

**Claude Code Bash**:
- 4 execution modes: normal, sandbox, pipeline, interactive
- Sandbox mode: filesystem restrictions + network isolation + process isolation
- Protected directories: `.git`, `.claude`, etc.

**Qwen Code run_shell_command**:
- AST semantic analysis (`isShellCommandReadOnlyAST()`)
- Command substitution detection (`$()`, backticks) -> automatically denied
- Read-only command detection -> automatically allowed
- Background execution support (`is_background`)
- Automatically adds co-author to Git commits

---

## 3. Permission System

### Claude Code: Multi-level Granular Permissions

```
Priority (high -> low):
1. CLI flags (--settings)
2. Project local (.claude/settings.local.json)
3. Project level (.claude/settings.json)
4. User level (~/.claude/settings.json)
5. Organization level
6. Enterprise level (managed-settings.json)
7. Defaults

Per tool: allow | ask | deny
Sandbox: filesystem + network + process isolation
Bypass: --dangerously-skip-permissions (can be disabled by enterprise policy)
```

### Qwen Code: Configuration-driven Permissions

```
Priority: deny(3) > ask(2) > default(1) > allow(0)

Rule types:
- persistentRules (persisted in settings files)
- sessionRules (session-level temporary rules)

Shell analysis:
- Virtual operation extraction (file/network)
- Compound command splitting
- Relative path resolution
```

### Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Number of levels** | 5 levels (Managed -> user) | 2 levels (persistent + session) |
| **Sandbox** | Yes (file/network/process) | Yes (Docker/Podman) |
| **Enterprise control** | Yes (managed-settings) | No |
| **Permission bypass** | Yes (can be disabled) | Yes (yolo_mode) |
| **Shell analysis** | Sandbox isolation | AST semantic analysis |
| **MCP tool permissions** | `mcp__plugin__tool` format | Same as built-in tools |
| **Protected paths** | .git, .claude | .env* files require confirmation |

---

## 4. Hook System

### Claude Code Hook Events (24)

| Event | Trigger timing |
|------|---------|
| **PreToolUse** | Before tool execution |
| **PostToolUse** | After successful execution |
| **PostToolUseFailure** | After execution failure |
| **UserPromptSubmit** | When user submits input |
| **Stop** | When the agent attempts to stop |
| **StopFailure** | Termination due to API error |
| **SubagentStart** | Subagent starts |
| **SubagentStop** | Subagent stops |
| **SessionStart** | Session starts |
| **SessionEnd** | Session ends |
| **PermissionRequest** | Permission request |
| **Notification** | Notification sent |
| **PreCompact** | Before context compaction |
| **PostCompact** | After context compaction |
| **TaskCompleted** | Task completed |
| **TeammateIdle** | Teammate idle |
| **InstructionsLoaded** | Instructions finished loading |
| **ConfigChange** | Configuration change |
| **WorktreeCreate** | Worktree created |
| **WorktreeRemove** | Worktree removed |
| **Elicitation** | Asking the user a question |
| **ElicitationResult** | User answer result |
| **CwdChanged** | Working directory changed (added in v2.1.83) |
| **FileChanged** | File-change detection (added in v2.1.83) |

### Qwen Code Hook Events

| Event | Trigger timing |
|------|---------|
| **PreToolUse** | Before tool execution |
| **PostToolUse** | After successful execution |
| **PostToolUseFailure** | After execution failure |
| **UserPromptSubmit** | When user submits input |
| **SessionStart** | Session starts |
| **SessionEnd** | Session ends |
| **SubagentStart** | Subagent starts |
| **SubagentStop** | Subagent stops |
| **PreCompact** | Before context compaction |
| **Stop** | Agent stops |
| **Notification** | Notification sent |
| **PermissionRequest** | Permission request |

### Hook Implementation Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Number of events** | **24** | 12 |
| **Claude-only events (12)** | StopFailure, PostCompact, TaskCompleted, TeammateIdle, InstructionsLoaded, ConfigChange, WorktreeCreate, WorktreeRemove, Elicitation, ElicitationResult, **CwdChanged**, **FileChanged** | -- |
| **Shared events (12)** | PreToolUse, PostToolUse, PostToolUseFailure, UserPromptSubmit, Stop, SubagentStart, SubagentStop, SessionStart, SessionEnd, PreCompact, Notification, PermissionRequest | All 12 |
| **Hook types** | Prompt (LLM-driven) + Command (script) | Command (script) |
| **Execution method** | Subprocess JSON stdin/stdout | Subprocess JSON stdin/stdout |
| **Timeout** | Configurable | Configurable |
| **Enterprise control** | Yes (allowManagedHooksOnly) | No |
| **Prompt Hook** | Yes (LLM reasoning decisions) | No |

**Claude Code's unique Prompt Hook**:
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [
        {"type": "prompt", "prompt": "Check whether this edit introduces a security vulnerability..."}
      ]
    }]
  }
}
```
Prompt Hooks use LLM reasoning to make decisions. They are more flexible than pure scripts, but slower. Qwen Code has no equivalent concept.

---

## 5. Plugin/Extension System

### Claude Code: Plugin Architecture

```
plugin-name/
├── .claude-plugin/plugin.json   # Metadata
├── commands/                    # Slash commands (*.md)
├── agents/                      # Subagent definitions (*.md)
├── skills/                      # Skill knowledge (*.md)
├── hooks/                       # Event handlers
└── .mcp.json                    # MCP servers
```

**13 official plugins**:

| Plugin | Description |
|------|------|
| code-review | 4 parallel agents review PRs with confidence scoring |
| feature-dev | 7-stage feature development workflow |
| commit-commands | Git workflows such as `/commit` and `/commit-push-pr` |
| pr-review-toolkit | 6 specialized agents (comments/tests/errors/types/quality/simplification) |
| security-guidance | PreToolUse Hook warns about injection/XSS/eval |
| hookify | Conversational Hook rule creation |
| agent-sdk-dev | Agent SDK development scaffolding |
| ralph-wiggum | Self-referential iteration loop (write -> test -> debug) |
| plugin-dev | Plugin development toolkit (7 skills) |
| explanatory-output-style | Educational implementation explanations |
| learning-output-style | Interactive learning mode |
| frontend-design | High-quality UI/UX implementation |
| claude-opus-4-5-migration | Opus 4.5 migration |

### Qwen Code: Extension System

```
Extensions support three formats:
├── Qwen Code native extensions (Git clone/Release)
├── Claude plugin conversion (claude-converter.ts)
└── Gemini extension conversion (gemini-converter.ts)
```

**Extension components**: MCP servers, Skills, Subagents, Hooks

### Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Number of plugins** | 13 official | Community-first |
| **Plugin format** | Markdown + JSON | Markdown + JSON |
| **Cross-tool compatibility** | Claude Code only | Yes, converts Claude/Gemini formats |
| **Installation method** | Plugin marketplace / manual | Git clone / Release |
| **Enterprise control** | Yes (strictKnownMarketplaces) | No |

---

## 6. Skill System

### Claude Code Skills

```markdown
<!-- skills/my-skill/SKILL.md -->
---
name: my-skill
description: Trigger-condition description
allowed-tools: [Read, Edit, Bash]
---

Skill content, in Markdown format...
```

- Automatic loading: natural-language matching based on trigger conditions
- Invoked through the `Skill` tool
- Supports distribution in plugins

### Qwen Code Skills

```yaml
# skills/my-skill/SKILL.md
---
name: my-skill
description: Skill description
tools: [bash, grep, read_file]
extends: bundled  # Optional: extend built-in skills
---

Skill content, in Markdown format...
```

- 4 levels: project / user / extension / bundled
- Supports `extends: bundled` to extend built-in skills
- Dynamically injects an XML list into tool descriptions

### Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Trigger method** | Natural-language conditions | Explicit invocation |
| **Levels** | Inside plugins | project/user/extension/bundled |
| **Skill composition** | Through plugins | `extends: bundled` |
| **Distribution** | Plugin marketplace | Git / extensions |

---

## 7. MCP Support

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Transports** | stdio, SSE, HTTP, WebSocket | stdio, SSE |
| **OAuth** | Yes (SSE servers) | Yes |
| **Configuration** | `.mcp.json` or `plugin.json` | `settings.json` |
| **Tool naming** | `mcp__plugin_name_server__tool` | Same level as built-in tools |
| **Environment variables** | `${CLAUDE_PLUGIN_ROOT}`, etc. | `${HOME}`, etc. |
| **Server management** | Plugin level | Global configuration |

---

## 8. Subagents/Multi-agent

### Claude Code

```markdown
<!-- agents/code-explorer.md -->
---
name: code-explorer
description: Explore codebase structure
  <example>Triggered when the user asks to understand the project architecture</example>
model: sonnet
tools: [Read, Grep, Glob]
---

You are a code exploration expert...
```

- **Natural-language triggering**: `<example>` tags in `description` teach Claude when to trigger the agent
- **Model selection**: sonnet / opus / haiku / inherit
- **Tool restrictions**: explicitly list available tools
- **Color labels**: blue / yellow / green / red

### Qwen Code

```typescript
interface SubagentConfig {
  name: string;
  description: string;
  tools?: string[];
  systemPrompt: string;
  level: 'session' | 'project' | 'user' | 'extension' | 'builtin';
  modelConfig?: Partial<ModelConfig>;
  color?: string;
}
```

- **5 levels**: session / project / user / extension / builtin
- **Arena mode**: three collaboration modes: Team / Swarm / Arena
- **Multiple terminal backends**: Tmux / iTerm2 / in-process
- **Independent models**: each subagent can configure a different model

### Comparison

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Trigger method** | Natural language + manual | Manual / agent tool |
| **Agent definition** | Markdown | TypeScript / TOML |
| **Arena** | No | Yes (Team/Swarm/Arena) |
| **Visual parallelism** | No | Yes (Tmux/iTerm2 panes) |
| **Model selection** | sonnet/opus/haiku/inherit | Any configured model |
| **Official agents** | Multiple agents across 13 plugins | Built-in builtin agents |

---

## 9. Configuration System

### Claude Code

```
~/.claude/
├── settings.json          # User global settings
├── CLAUDE.md              # User-level instructions
└── Plugin data

.claude/
├── settings.json          # Project settings
├── settings.local.json    # Local settings (not committed)
├── CLAUDE.md              # Project instructions
├── commands/              # Custom commands
└── Plugin data

5-level priority: enterprise -> organization -> user -> project -> local -> CLI flags
```

### Qwen Code

```
~/.qwen/
├── settings.json          # Global settings
├── locales/               # 6 language packs
├── skills/                # User skills
├── agents/                # User agents
└── tmp/<hash>/chats/      # Session storage

.qwen/
├── settings.json          # Project settings
├── system.md              # Project instructions
├── skills/                # Project skills
└── agents/                # Project agents

4-level priority: defaults -> global -> project -> environment variables -> CLI
```

---

## 10. Unique Features

### Only Claude Code Has

| Feature | Description |
|------|------|
| **Prompt Hook** | Hook decisions driven by LLM reasoning |
| **5-level settings priority** | Enterprise -> organization -> user -> project -> local -> CLI |
| **Bash sandbox** | Filesystem + network + process isolation (4 modes) |
| **13 official plugins** | code-review, feature-dev, security, etc. |
| **Natural-language agent triggering** | `<example>` tags teach the LLM when to trigger agents |
| **ralph-wiggum iteration** | Stop Hook prevents exit and repeats automatically |
| **hookify conversational rules** | Create Hook rules through chat |
| **Enterprise control** | managed-settings, strictKnownMarketplaces |
| **Git Worktree** | Isolated branches for parallel work |
| **Automatic memory** | Learns user preferences across sessions |
| **NotebookEdit** | Jupyter Notebook editing |
| **StopFailure event** | Special handling for API errors |

### Only Qwen Code Has

| Feature | Description |
|------|------|
| **Open source + BYOK** | Apache-2.0 software is free; models can connect to any provider |
| **6+ providers** | Qwen OAuth + DashScope + ModelScope + Anthropic + Google + custom |
| **Arena mode** | Multi-agent competition/collaboration (Team/Swarm/Arena) |
| **Tmux/iTerm2 panes** | Visual parallel agents |
| **6-language UI** | Chinese/English/Japanese/German/Russian/Portuguese |
| **Extension format conversion** | Automatic conversion of Claude/Gemini extensions |
| **Loop detection** | Levenshtein-distance repeated-call detection |
| **save_memory** | Persist memory to Markdown |
| **LSP tools** | Language Server Protocol operations |
| **Session Token limit** | Hard token budget |
| **Shell AST analysis** | Command semantic-level security checks |
| **PostToolUseFailure** | Dedicated Hook for tool failure |
| **PermissionRequest event** | Hook for permission dialogs |

---

## 11. Suitable Scenarios

| Scenario | Recommendation | Reason |
|------|------|------|
| **Complex reasoning** | Claude Code | Claude models have the strongest reasoning capability |
| **Enterprise deployment** | Claude Code | 5-level settings + enterprise control + sandbox |
| **Low cost/self-hosting** | Qwen Code | Open source and free + BYOK can connect to low-cost providers such as DeepSeek |
| **Multi-model switching** | Qwen Code | Flexible switching across 6+ providers |
| **Chinese-language development** | Qwen Code | 6-language UI + strong Chinese capability in Qwen models |
| **Code review** | Claude Code | code-review plugin (4 parallel agents) |
| **Multi-agent collaboration** | Qwen Code | Arena + Tmux visualization |
| **Security-sensitive** | Claude Code | Sandbox + Prompt Hook + enterprise control |
| **Open-source customization** | Qwen Code | Full source code + Apache-2.0 |
| **Plugin ecosystem** | Claude Code | 13 official plugins + marketplace |

---

## 12. Summary

**Claude Code** is a closed-source but highly complete commercial agent: native Rust performance, LLM-driven Prompt Hook decisions, 5-level enterprise control, and 13 official plugins form the most mature ecosystem. The limits are model lock-in and the paid entry barrier.

**Qwen Code** is the most feature-rich open-source agent: Apache-2.0 open source + BYOK with any provider (including low-cost options such as DeepSeek), flexible access to 6+ providers, an Arena multi-agent framework, and a 6-language internationalized UI make it the most attractive open-source option. However, as a Gemini CLI fork, some code still carries upstream traces; model calls require a Coding Plan subscription or BYOK (OAuth free tier ended 2026-04-15).

The two have **high similarity** in their tool systems, Hook architecture, and skill systems (both use declarative tools + event Hooks + Markdown skills), indicating that the industry is converging on a shared agent-architecture pattern.

---

*Analysis based on the Claude Code plugin repository (v2.1.81) and local Qwen Code source, as of March 2026.*
