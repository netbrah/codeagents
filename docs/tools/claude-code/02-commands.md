# 2. Claude Code Command System: Developer Reference

> Claude Code has four command types and about 79 slash commands, far more than Qwen Code's ~40. This document analyzes its command registration, loading, and permission design as an architecture reference for Code Agent developers.
>
> **Qwen Code comparison**: command registration mechanism (BuiltinCommandLoader vs. four type loaders), command classification strategy (prompt/local-jsx/local/skill), and permission-control granularity

## Why Four Command Types Are Needed

### Problem Definition

A Code Agent's slash commands have very different execution requirements:

| Requirement | Example | Needs LLM? | Needs UI? | Execution Method |
|------|------|-----------|---------|---------|
| "Help me review this PR" | `/review` | ✓ | — | Send a prompt to the LLM |
| "Switch models" | `/model` | — | ✓ selection dialog | Local React component |
| "Clear the screen" | `/clear` | — | — | Execute code directly |
| "Commit code" | `/commit` | ✓ | — | External Skill definition |

With only one command type, either everything goes through the LLM (should `/clear` call the API too?), or everything executes locally (how would `/review` reason?). Claude Code solves this with 4 types.

### Command System Comparison with Competitors

| Agent | Command Count | Command Types | Extension Mechanism |
|-------|--------|---------|---------|
| **Claude Code** | ~79 | 4 types (prompt/local-jsx/local/skill) | Skill + Plugin |
| **Gemini CLI** | ~30 | 2 types (built-in + TOML command) | TOML files + Skill |
| **Qwen Code** | ~40 | 2 types (built-in + Skill) | BundledSkillLoader |
| **Copilot CLI** | ~20 | 2 types (built-in + Plugin) | Plugin system |
| **Cursor** | ~15 | 1 type (built-in) | None |

---

## Command Type Overview

Claude Code divides slash commands into four types by implementation method:

| Type | Identifier | Count | Execution Method | Calls LLM? |
|------|------|------|----------|-------------|
| **prompt** | `prompt` | 6 | Sends a preset Prompt to the LLM; the LLM calls tools to complete the task | Yes |
| **local-jsx** | `local-jsx` | ~50 | Renders local React/Ink terminal UI components | No |
| **local** | `local` | ~11 | Executes local logic directly without UI rendering | No |
| **skill** | `skill/plugin` | several | Extension commands registered through the plugin system or Skill mechanism | Depends |

The **prompt type** is the most powerful command type: these commands construct a dedicated Prompt (including context information), send it to the Claude model, and let the model autonomously call Bash, Edit, Read, and other tools to complete complex tasks such as committing code or reviewing PRs.

The **local-jsx type** is the largest group, covering daily operations such as configuration management, session management, and UI settings. They render interactive React/Ink components in the terminal (selection lists, form input, status displays, etc.) without consuming API tokens.

The **local type** is the lightest command type. It directly executes local logic (clearing history, compacting context, showing cost, etc.) without UI rendering or LLM calls.

The **skill type** is provided by the plugin system or built-in Skill mechanism and can dynamically register new commands at runtime.

---

## prompt Type Commands (6)

These 6 commands share a common characteristic: they combine specific context and instructions into a Prompt sent to the Claude model, which then autonomously uses tools (Bash, Read, Edit, Grep, etc.) to complete the task. Users see the full LLM execution process, including tool calls.

---

### /review (`/code-review` plugin)

- **Type:** prompt (built-in Skill) + code-review plugin (multi-agent orchestration)
- **Function:** Automated code review, multi-agent parallel audit + confidence scoring to filter false positives
- **Author:** Boris Cherny (boris@anthropic.com), official Anthropic plugin
- **Source:** [`plugins/code-review/`](https://github.com/anthropics/claude-code/tree/main/plugins/code-review)

**Usage examples:**
```bash
# Local review (output to terminal)
/code-review

# Review and post inline PR comments
/code-review --comment
```

**Allowed tools** (restricted by source Frontmatter):
```
Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*),
Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*),
Bash(gh pr list:*), mcp__github_inline_comment__create_inline_comment
```

**9-step pipeline (Source: `commands/code-review.md`):**

| Step | Agent | Model | Task |
|------|------|------|------|
| 1 | Pre-check | Haiku | Check whether to skip: closed/draft/trivial/already reviewed. Claude-generated PRs are still reviewed |
| 2 | Collect guidelines | Haiku | Collect all relevant CLAUDE.md files in the repository (root + directories of files touched by the PR) |
| 3 | Change summary | Sonnet | Generate a structured summary of PR changes |
| 4a | CLAUDE.md compliance #1 | Sonnet | Audit whether code violates CLAUDE.md rules (only checks CLAUDE.md files matching file paths) |
| 4b | CLAUDE.md compliance #2 | Sonnet | Redundant audit (double-check to reduce missed issues) |
| 4c | Bug scan | **Opus** | Focus only on the diff itself and do not read extra context. Mark only major defects |
| 4d | Security/logic analysis | **Opus** | Analyze security risks and logic errors in newly added code |
| 5 | Parallel validation | Opus/Sonnet | Each flagged issue is confirmed by an independent validation agent (Opus validates bugs, Sonnet validates compliance) |
| 6 | Filtering | — | Remove issues that did not pass validation |
| 7 | Output | — | Output the review report in the terminal; if there is no `--comment`, stop here |
| 8 | Comment preparation | — | Internally list all comments planned for publication (self-check, not public) |
| 9 | Post comments | — | Post inline PR comments through MCP GitHub tools |

**Review dimensions (explicitly defined in source):**

| Dimension | What Is Checked | Agent |
|------|----------|------|
| **Compilation/parsing errors** | Syntax errors, type errors, missing imports, unresolved references | Agent 3 (Opus) |
| **Logic errors** | Clear logic problems that produce incorrect results regardless of input | Agent 3 (Opus) |
| **Security issues** | Security risks in newly added code | Agent 4 (Opus) |
| **CLAUDE.md compliance** | Whether code violates project rules (must quote the specific violated rule) | Agent 1-2 (Sonnet) |

**Explicitly excluded false positives (source explicitly says not to flag):**
- Issues in existing code (not introduced by the PR)
- Code that looks like a bug but is actually correct
- Nitpicks that a senior engineer would not flag
- Issues a linter would catch (do not run a linter to validate)
- General code quality issues (unless CLAUDE.md explicitly requires them)
- Issues explicitly suppressed by lint-ignore comments in the code

**Filtering mechanism (sources: command source + README):**

Command prompt implementation: Step 5 starts **parallel validation agents**. Each flagged issue is independently confirmed by a separate agent. Issues that fail validation are removed in Step 6.

Confidence scoring system described in README (for context):

| Score | Meaning |
|------|------|
| 0 | Not confident; false positive |
| 25 | Somewhat confident; may be real |
| 50 | Moderately confident; real but minor |
| 75 | Highly confident; real and important |
| 100 | Absolutely certain; definitely real |

The README states an 80-point threshold (≥80 passes), but the actual filtering in the command source is implemented as a binary decision by validation agents (pass/fail).

**PR comment format (defined in source):**
```markdown
## Code review

Found 3 issues:

1. Missing error handling for OAuth callback (CLAUDE.md says "Always handle OAuth errors")
   https://github.com/owner/repo/blob/abc123.../src/auth.ts#L67-L72

2. Memory leak: OAuth state not cleaned up
   https://github.com/owner/repo/blob/abc123.../src/auth.ts#L88-L95
```

Link requirements: must use full SHA (not abbreviated), `#L` markers, and at least 1 line of context.

**Observed impact** (source: [TechCrunch report](https://techcrunch.com/2026/03/09/anthropic-launches-code-review-tool-to-check-flood-of-ai-generated-code/)):
- PRs receiving substantive review comments increased from 16% to 54%
- False-positive rate <1%

---

**Related plugin: pr-review-toolkit (6 specialized review agents)**

| Agent | Focus Area |
|------|---------|
| comment-analyzer | Comment accuracy, documentation completeness, comment rot |
| test-coverage | Test coverage, edge cases, test quality |
| error-handling | Error-handling completeness, exceptional flows |
| type-design | Type design, interface contracts |
| code-quality | Code quality, complexity, maintainability |
| simplify | Code simplification and redundancy removal |

---

### /batch

- **Type:** prompt (built-in Skill)
- **Function:** Orchestrates large-scale parallel changes by splitting a large task into multiple subtasks executed in parallel
- **Source:** [Official documentation](https://code.claude.com/docs/en/slash-commands)

**Usage examples:**
```bash
/batch Replace all console.log calls with logger.info
/batch Add error handling to all API endpoints
```

**How it works:**
- Analyzes task scope and identifies subtasks that can run in parallel
- Starts multiple subagents to execute changes in parallel
- Aggregates results and change reports

---

### /debug

- **Type:** prompt (built-in Skill)
- **Function:** Troubleshoots session problems by analyzing debug logs
- **Source:** [Official documentation](https://code.claude.com/docs/en/slash-commands)

**Usage examples:**
```bash
/debug
/debug Why did the previous step fail?
```

**How it works:**
- Reads the session's debug log files
- Analyzes errors, exceptions, and abnormal behavior patterns
- Provides troubleshooting suggestions and fixes

---

### /commit

- **Type:** prompt (commit-commands plugin)
- **Function:** Analyzes staged Git changes, automatically generates a commit message, and runs `git commit`
- **Source:** [`plugins/commit-commands/`](https://github.com/anthropics/claude-code/tree/main/plugins/commit-commands)

**Usage examples:**
```
/commit
/commit Please write the commit message in Chinese
```

**How it works:**
1. Collects `git status` and `git diff --staged` information
2. Sends the changes and repository context to the LLM
3. The LLM analyzes the nature of the changes (feature, fix, refactor, etc.) and generates a conventional commit message
4. Calls the Bash tool to run `git commit -m "..."`
5. By default, appends a `Co-Authored-By: Claude ...` signature at the end of the commit message

**Implementation details:**
- The LLM examines recent commit history to match the repository's commit-message style
- Automatically detects unstaged files and reminds the user
- Does not run `git add`; the user must stage files themselves
- Does not run `git push`; it only creates a local commit

---

### /commit-push-pr

- **Type:** prompt
- **Function:** One-stop workflow for commit + push + Pull Request creation
- **Alias:** None

**Usage examples:**
```
/commit-push-pr
/commit-push-pr This PR fixes an XSS vulnerability on the login page
```

**How it works:**
1. Uses the same flow as `/commit` to analyze and commit changes
2. Pushes the current branch to the remote repository (setting upstream if needed)
3. Calls `gh pr create` to create a Pull Request
4. The LLM automatically generates the PR title and description (including change summary, test plan, etc.)
5. Returns the created PR link

**Implementation details:**
- Depends on the `gh` CLI to create the PR
- The LLM examines all commits to be merged (not just the latest one) to write the PR description
- Automatically detects the base branch (usually main or master)

---

### /init

- **Type:** prompt
- **Function:** Analyzes the project structure and generates or updates the `CLAUDE.md` project instruction file
- **Alias:** None

**Usage examples:**
```
/init
/init Please focus on the configuration of this Python project
```

**How it works:**
1. The LLM scans the project directory structure, configuration files (package.json, pyproject.toml, etc.), README, and so on
2. Analyzes the project's technology stack, build system, test framework, and coding conventions
3. Generates a `CLAUDE.md` file containing:
   - Project overview and technology stack
   - Common commands (build, test, lint)
   - Code style conventions
   - Project-specific AI behavior instructions
4. If `CLAUDE.md` already exists, analyzes the current content and updates it incrementally

**Implementation details:**
- `CLAUDE.md` acts like a project-level system prompt and is automatically loaded at session startup
- Supports three levels of CLAUDE.md: `~/.claude/CLAUDE.md` (global), project root, and subdirectories
- Generated content is based on actual project configuration and avoids generic template text

---

### /init-verifiers

- **Type:** prompt
- **Function:** Creates verifier Skills for the project to automate validation of code changes
- **Alias:** None

**Usage examples:**
```
/init-verifiers
```

**How it works:**
1. The LLM analyzes the project structure and identifies available validation methods (lint, type-check, test, etc.)
2. Generates verifier Skill configuration defining automatic checks after code changes
3. The verifier Skill can be automatically triggered in later code modification workflows

**Implementation details:**
- A verifier is a special Skill bound to code-change events
- It can validate type safety, code style, test pass rate, and more
- Generated configuration is stored under the `.claude/` directory

---

### /insights

- **Type:** prompt
- **Function:** Generates a session insight report / year-in-review
- **Alias:** None

**Usage examples:**
```
/insights
```

**How it works:**
1. Collects current or historical session data (token usage, tool calls, task completion, etc.)
2. The LLM analyzes usage patterns and generates an insight report
3. May include usage frequency, common operations, efficiency metrics, and other statistics

---

## local-jsx Type Commands (~50)

These commands render interactive React/Ink UI components in the terminal. They do not call the LLM, do not consume API tokens, and respond very quickly. They are grouped by function below.

---

### Session Management

#### /resume

- **Type:** local-jsx
- **Function:** Resumes a previously interrupted session by selecting one from the historical session list
- **Alias:** Equivalent to the CLI `--resume` option

**Usage examples:**
```
/resume
```

**Implementation details:**
- Renders an interactive list showing recent sessions (including timestamps and summaries)
- Restores the full conversation history and context after selection
- Session data is stored under `~/.claude/projects/<project-hash>/`

#### /session

- **Type:** local-jsx
- **Function:** Displays the remote session URL and QR code (alias: /remote)
- **Alias:** `/remote`

**Usage examples:**
```
/session
/remote
```

**Implementation details:**
- Displays the remote session URL and QR code
- Can be used to connect to the current session from another device

#### /rename

- **Type:** local-jsx
- **Function:** Renames the current session with a meaningful name
- **Alias:** None

**Usage examples:**
```
/rename Fix login bug
/rename
```

**Implementation details:**
- If no name argument is provided, renders a text-input UI
- The session name appears in the `/resume` list for easier identification

#### /tag

- **Type:** local-jsx
- **Function:** Tags the current session for classification and retrieval
- **Alias:** None

**Usage examples:**
```
/tag bugfix
/tag
```

#### /export

- **Type:** local-jsx
- **Function:** Exports the conversation history of the current session
- **Alias:** None

**Usage examples:**
```
/export
/export json
```

**Implementation details:**
- Supports exporting as JSON, Markdown, and other formats
- Exported content includes the full conversation messages and tool-call records

#### /rewind

- **Type:** local-jsx
- **Function:** Rewinds to a previous checkpoint, restoring code and/or conversation history
- **Alias:** `/checkpoint`

**Usage examples:**
```
/rewind
/checkpoint
```

**Implementation details:**
- Renders a checkpoint-selection UI listing available historical nodes
- Supports restoring code changes and/or conversation history
- Uses the Git checkpoint mechanism to roll back file state

---

### Models and Configuration

#### /model

- **Type:** local-jsx
- **Function:** Switches the AI model used by the current session
- **Alias:** None

**Usage examples:**
```
/model
/model opus
/model sonnet
```

**Implementation details:**
- Renders a model-selection list showing available models and their characteristics
- Available models depend on subscription plan and API configuration
- Common options: `claude-sonnet-4-6`, `claude-opus-4-6`, `claude-haiku-4-5`
- Switching takes effect immediately and affects all subsequent LLM calls

#### /config

- **Type:** local-jsx
- **Function:** Opens or edits Claude Code configuration files
- **Alias:** None

**Usage examples:**
```
/config
/config set theme dark
```

**Implementation details:**
- Renders a configuration editing UI showing currently effective options
- Configuration priority (high to low): system settings -> workspace -> user settings -> default
- Configuration file paths: `~/.claude/settings.json` (user-level), `.claude/settings.json` (project-level)

#### /effort

- **Type:** local-jsx
- **Function:** Adjusts the model's reasoning effort
- **Alias:** None

**Usage examples:**
```
/effort
/effort high
/effort low
```

**Implementation details:**
- Renders a slider or option list (low / medium / high)
- Lower effort reduces token consumption and latency, suitable for simple tasks
- Higher effort increases reasoning depth, suitable for complex analysis tasks
- Corresponds to `thinking` / `budget_tokens` settings in API parameters

#### /fast

- **Type:** local-jsx
- **Function:** Quickly switches to low-latency mode (using a faster model or lower effort)
- **Alias:** None

**Usage examples:**
```
/fast
```

**Implementation details:**
- One-click switch to Haiku or low-effort Sonnet mode
- Suitable for simple operations that need quick responses
- Running it again can switch back to the default mode

#### /permissions

- **Type:** local-jsx
- **Function:** Manages tool permission settings (allow/deny/always allow, etc.)
- **Alias:** `/allowed-tools`

**Usage examples:**
```
/permissions
```

**Implementation details:**
- Renders a permission-management UI listing all tools and their current permission states
- Permissions are divided into `allow`, `deny`, and `ask` (ask every time)
- Settings are stored in the `permissions` field of `~/.claude/settings.json`
- Supports different permission policies by path and by project

#### /hooks

- **Type:** local-jsx
- **Function:** Manages Prompt Hook configuration (hook functions for 24 event types)
- **Alias:** None

**Usage examples:**
```
/hooks
```

**Implementation details:**
- Renders a Hook management UI listing configured Hooks and their trigger events
- Supported event types include PreToolUse, PostToolUse, Notification, Stop, and others
- Hooks can be Shell commands or LLM-reasoning-driven decision logic
- Configuration is stored in the `hooks` field of settings.json

#### /privacy-settings

- **Type:** local-jsx
- **Function:** Manages privacy settings (data collection, telemetry, and related options)
- **Alias:** None

**Usage examples:**
```
/privacy-settings
```

#### /rate-limit-options

- **Type:** local-jsx
- **Function:** Views and configures rate-limit-related options
- **Alias:** None

**Usage examples:**
```
/rate-limit-options
```

**Implementation details:**
- Shows current API rate-limit status
- Can configure behavior when limits are reached (wait/switch model/notify)

#### /extra-usage

- **Type:** local-jsx
- **Function:** Views extra usage information and quota details
- **Alias:** None

**Usage examples:**
```
/extra-usage
```

---

### Context and Files

#### /add-dir

- **Type:** local-jsx
- **Function:** Adds an extra directory to the working context of the current session
- **Alias:** None

**Usage examples:**
```
/add-dir /path/to/another/project
/add-dir ../shared-lib
```

**Implementation details:**
- Renders a directory-selection UI or directly accepts a path argument
- After being added, LLM file-operation tools can access that directory
- Does not change the actual working directory; it only expands file access scope
- Suitable for referencing code across projects

#### /diff

- **Type:** local-jsx
- **Function:** Shows a diff view of all file changes Claude made in the current session
- **Alias:** None

**Usage examples:**
```
/diff
```

**Implementation details:**
- Renders a colored diff view showing changes in all modified files
- Tracks changes through the Git checkpoint mechanism
- Useful for reviewing Claude's modifications before committing

#### /status

- **Type:** local-jsx
- **Function:** Shows status information for the current session and project
- **Alias:** None

**Usage examples:**
```
/status
```

**Implementation details:**
- Shows current model, session duration, and token usage
- Shows project information (Git branch, working directory)
- Shows MCP server connection status

#### /plan

- **Type:** local-jsx
- **Function:** Toggles Plan Mode, where the LLM only analyzes and plans without making modifications
- **Alias:** None

**Usage examples:**
```
/plan
/plan on
/plan off
```

**Implementation details:**
- When enabled, the LLM's available tools are restricted to read-only tools (Read, Glob, Grep, WebFetch, etc.)
- Mutating tools such as Write, Edit, MultiEdit, and Bash are disabled
- Suitable for first asking Claude to analyze the problem and propose a plan, then manually confirming before execution
- Running `/plan` again switches back to normal mode

#### /tasks

- **Type:** local-jsx
- **Function:** Views and manages background tasks (asynchronous tasks created through the Task tool)
- **Alias:** `/bashes`

**Usage examples:**
```
/tasks
```

**Implementation details:**
- Lists all active and completed background tasks
- Shows each task's status, progress, and output summary
- Tasks are created by the LLM through the TaskCreate tool for parallel processing

#### /brief

- **Type:** local-jsx
- **Function:** Toggles concise output mode to reduce Claude's verbosity
- **Alias:** None

**Usage examples:**
```
/brief
/brief on
/brief off
```

**Implementation details:**
- When enabled, adds concise-output instructions to the system prompt
- Claude reduces explanatory text and outputs results more directly

#### /btw

- **Type:** local-jsx
- **Function:** Inserts an aside (by the way) into the current conversation flow without affecting the main task
- **Alias:** None

**Usage examples:**
```
/btw By the way, please check the version number in package.json
```

---

### Tools and Extensions

#### /mcp

- **Type:** local-jsx
- **Function:** Manages MCP (Model Context Protocol) server connections
- **Alias:** None

**Usage examples:**
```
/mcp
```

**Implementation details:**
- Renders an MCP server management UI showing configured and connected servers
- Supports three transport protocols: Stdio, SSE, and Streamable-HTTP
- MCP tools are registered into the LLM's tool list in the `mcp__serverName__toolName` format
- Configuration is stored in the `mcpServers` field of settings.json

#### /plugin

- **Type:** local-jsx
- **Function:** Manages Claude Code plugins (install, uninstall, view, configure)
- **Alias:** `/plugins`, `/marketplace`

**Usage examples:**
```
/plugin
/plugin install security-review
```

**Implementation details:**
- Renders a plugin management UI with marketplace browsing and installed plugin list
- Plugins can register new slash commands, tools, Hooks, and more
- Known official plugins include security-review, pr-comments, code-review, etc.

#### /skills

- **Type:** local-jsx
- **Function:** Views and manages available Skills
- **Alias:** None

**Usage examples:**
```
/skills
```

**Implementation details:**
- Lists all registered Skills, including built-in Skills and plugin-provided Skills
- Skills are reusable capability modules that the LLM invokes through the Skill tool
- Supports enabling/disabling Skills

#### /agents

- **Type:** local-jsx
- **Function:** Manages multi-agent configuration (Teammates feature)
- **Alias:** None

**Usage examples:**
```
/agents
```

**Implementation details:**
- Renders an agent management UI for viewing and configuring Teammates
- Teammates use tmux/iTerm2 split panes, with each agent in an independent worktree
- Each agent can be assigned a different model and role

#### /ide

- **Type:** local-jsx
- **Function:** Configures IDE integration (VS Code, JetBrains, and other editor connections)
- **Alias:** None

**Usage examples:**
```
/ide
```

---

### Memory and Authentication

#### /memory

- **Type:** local-jsx
- **Function:** Views and edits Claude's project memory files (CLAUDE.md)
- **Alias:** None

**Usage examples:**
```
/memory
```

**Implementation details:**
- Renders a memory management UI showing all currently effective CLAUDE.md files
- Supports editing global (`~/.claude/CLAUDE.md`) and project-level (`./CLAUDE.md`) memory
- Memory content is automatically loaded into the system prompt at each session startup

#### /login

- **Type:** local-jsx
- **Function:** Logs in to a Claude account (Anthropic account or OAuth authentication)
- **Alias:** None

**Usage examples:**
```
/login
```

**Implementation details:**
- Renders the OAuth authentication flow UI
- Supports direct Anthropic accounts and enterprise SSO
- Authentication tokens are stored in the system keychain or under `~/.claude/`

#### /logout

- **Type:** local-jsx
- **Function:** Logs out of the current account and clears authentication tokens
- **Alias:** None

**Usage examples:**
```
/logout
```

---

### UI and Display

#### /color

- **Type:** local-jsx
- **Function:** Configures the color scheme for terminal output
- **Alias:** None

**Usage examples:**
```
/color
```

**Implementation details:**
- Renders a color-selection UI
- Can switch between light/dark/custom color schemes

#### /theme

- **Type:** local-jsx
- **Function:** Switches the Claude Code terminal theme
- **Alias:** None

**Usage examples:**
```
/theme
/theme dark
/theme light
```

**Implementation details:**
- Renders a theme-selection UI and previews different themes
- Affects syntax highlighting, diff display, UI components, and other visual elements

#### /copy

- **Type:** local-jsx
- **Function:** Copies Claude's last response to the system clipboard
- **Alias:** None

**Usage examples:**
```
/copy
```

**Implementation details:**
- Calls the system clipboard API (pbcopy/xclip/xsel, etc.)
- Copies the response content as plain text

#### /stats

- **Type:** local-jsx
- **Function:** Shows detailed statistics for the current session
- **Alias:** None

**Usage examples:**
```
/stats
```

**Implementation details:**
- Renders a statistics panel containing:
  - Total tokens (input/output/cache)
  - Tool-call counts and distribution
  - Session duration
  - Model usage

#### /usage

- **Type:** local-jsx
- **Function:** Shows API usage and quota information
- **Alias:** None

**Usage examples:**
```
/usage
```

**Implementation details:**
- Renders a usage panel showing API usage in the current billing cycle
- Shows remaining quota for subscription users
- Shows cost statistics for API users

#### /help

- **Type:** local-jsx
- **Function:** Shows help information for all available commands
- **Alias:** None

**Usage examples:**
```
/help
/help commit
```

**Implementation details:**
- Renders categorized command lists, each with a short description
- Supports viewing detailed help for a specific command

#### /feedback

- **Type:** local-jsx
- **Function:** Submits feedback to the Anthropic team
- **Alias:** None

**Usage examples:**
```
/feedback
```

**Implementation details:**
- Renders a feedback form UI
- Can submit session context along with the feedback

---

### Remote and Integrations

#### /remote-control

- **Type:** local-jsx
- **Function:** Starts remote control mode and bridges to the claude.ai/code browser interface
- **Alias:** `/rc`

**Usage examples:**
```
/remote-control
```

**Implementation details:**
- Renders a remote-control configuration UI
- Registers the session with the Anthropic API (WebSocket/SSE bidirectional communication)
- Allows terminal sessions to be operated from a browser
- Supports cross-device remote operation

#### /remote-env

- **Type:** local-jsx
- **Function:** Configures remote environment settings (Claude Code instances on remote servers)
- **Alias:** None

**Usage examples:**
```
/remote-env
```

#### /desktop

- **Type:** local-jsx
- **Function:** Continues the current session in Claude Desktop (alias: /app)
- **Alias:** `/app`

**Usage examples:**
```
/desktop
/app
```

#### /mobile

- **Type:** local-jsx
- **Function:** Shows a QR code to download the Claude mobile app (aliases: /ios, /android)
- **Alias:** `/ios`, `/android`

**Usage examples:**
```
/mobile
/ios
/android
```

#### /install-github-app

- **Type:** local-jsx
- **Function:** Installs the Claude Code GitHub App into the repository (for automatic PR review and related features)
- **Alias:** None

**Usage examples:**
```
/install-github-app
```

**Implementation details:**
- Renders an installation guide UI
- Guides the user through GitHub App authorization and repository binding
- After installation, supports automated tasks triggered by GitHub events

#### /web-setup

- **Type:** local-jsx
- **Function:** Configures Web integration settings (Chrome extension, etc.)
- **Alias:** None

**Usage examples:**
```
/web-setup
```

---

### System

#### /exit

- **Type:** local-jsx
- **Function:** Exits the Claude Code session
- **Alias:** `/quit`, Ctrl+C twice, Ctrl+D

**Usage examples:**
```
/exit
```

#### /install

- **Type:** local-jsx
- **Function:** Installs the native Claude Code build
- **Alias:** None

**Usage examples:**
```
/install
```

#### /upgrade

- **Type:** local-jsx
- **Function:** Upgrades to the Max plan for higher rate limits and more Opus usage
- **Alias:** None

**Usage examples:**
```
/upgrade
```

**Implementation details:**
- Guides the user to upgrade to the Max plan
- Shows a comparison of rate limits and Opus usage across plans

#### /terminal-setup

- **Type:** local-jsx
- **Function:** Configures the terminal environment (shell integration, shortcuts, etc.)
- **Alias:** None

**Usage examples:**
```
/terminal-setup
```

#### /install-slack-app

- **Type:** local-jsx
- **Function:** Installs the Claude Slack app
- **Alias:** None

**Usage examples:**
```
/install-slack-app
```

#### /reload-plugins

- **Type:** local-jsx
- **Function:** Activates pending plugin changes in the current session
- **Alias:** None

**Usage examples:**
```
/reload-plugins
```

#### /chrome

- **Type:** local-jsx
- **Function:** Claude in Chrome (Beta) setup
- **Alias:** None

**Usage examples:**
```
/chrome
```

#### /passes

- **Type:** local-jsx
- **Function:** Shares a free Claude Code trial week with friends and earns extra usage
- **Alias:** None

**Usage examples:**
```
/passes
```

#### /think-back

- **Type:** local-jsx
- **Function:** Claude Code 2025 year-in-review
- **Alias:** None

**Usage examples:**
```
/think-back
```

**Implementation details:**
- Shows a Claude Code usage review for 2025
- Summarizes annual usage data and milestones

#### /branch

- **Type:** local-jsx
- **Function:** Creates a branch of the current conversation (alias: /fork)
- **Alias:** `/fork`

**Usage examples:**
```
/branch
/fork
```

**Implementation details:**
- Creates a branch of the current conversation, forking a new conversation path
- Supports exploring different solutions in different branches

---

## local Type Commands (~11)

These commands execute directly locally and neither call the LLM nor render complex UI. They are the lightest commands and execute fastest.

---

### /compact

- **Type:** local
- **Function:** Compacts the current context window, preserving key information and freeing token space
- **Alias:** None

**Usage examples:**
```
/compact
/compact Please preserve the context about database migration
```

**How it works: three-layer compression system:**

1. **Micro compression:** Automatically truncates and summarizes overly long tool outputs, reducing token usage at the message level
2. **Auto compression:** Automatically triggers full compression when context usage reaches ~95%, using the dedicated `compact-2026-01-12` API endpoint
3. **Manual compression:** Triggered when the user runs `/compact`, optionally with preservation instructions

**Implementation details:**
- Uses the dedicated `compact-2026-01-12` API endpoint for compression
- Compression is performed by a specialized model, not the user's chat model
- After compression, key decisions, code-change records, and unfinished tasks are retained
- Optional arguments specify context that should be preserved with priority
- Compression ratio is usually between 3:1 and 10:1 depending on conversation content

---

### /clear

- **Type:** local
- **Function:** Completely clears the current conversation history and starts over
- **Alias:** None

**Usage examples:**
```
/clear
```

**Implementation details:**
- Thoroughly clears the conversation message list
- CLAUDE.md and the system prompt are reloaded on the next conversation turn
- Does not affect Git history or file-system state
- Different from `/compact`: `/clear` preserves no context

---

### /context

- **Type:** local
- **Function:** Visualizes current context-window usage
- **Alias:** None

**Usage examples:**
```
/context
```

**Implementation details:**
- Displays a colored grid showing the usage ratio of context space
- Different colors represent different content types (system prompt, conversation, tool output, etc.)
- Shows used / total token counts
- Gives compression suggestions when approaching capacity

---

### /cost

- **Type:** local
- **Function:** Shows token consumption and cost for the current session
- **Alias:** None

**Usage examples:**
```
/cost
```

**Implementation details:**
- Shows input tokens, output tokens, and cache tokens
- Calculates estimated cost according to model pricing
- Shows cumulative cost and cost for the current conversation
- For subscription users, shows quota usage rather than cost

---

### /doctor

- **Type:** local
- **Function:** Diagnoses Claude Code installation health
- **Alias:** None

**Usage examples:**
```
/doctor
```

**Implementation details:**
- Runs a series of diagnostic checks:
  - Authentication status (whether API Key / OAuth token is valid)
  - Configuration file syntax check (whether settings.json is valid)
  - MCP server connection tests (reachability of each server)
  - Git environment checks (git version, repository state)
  - Shell environment checks (PATH, required tools)
  - Network connectivity checks (API endpoint reachability)
- Outputs a diagnostic report and marks items as passed/failed/warning
- Provides fix suggestions for failed items

---

### /vim

- **Type:** local
- **Function:** Toggles Vim editing mode and Normal editing mode
- **Alias:** None

**Usage examples:**
```
/vim
```

**Implementation details:**
- Switches the editing mode of the terminal input box
- Vim mode supports hjkl movement, i/a to enter insert mode, Esc to return to normal mode, etc.
- State is persisted to user configuration

---

### /voice

- **Type:** local
- **Function:** Toggles voice mode (push-to-talk voice input)
- **Alias:** None

**Usage examples:**
```
/voice
```

**Implementation details:**
- Starts microphone recording and connects to a speech-to-text API through WebSocket
- Push-to-talk mode: hold a configured key to speak, then release to automatically transcribe into text input
- The transcription result is filled into the input box and sent after user confirmation
- Requires system microphone permission

---

### /keybindings

- **Type:** local
- **Function:** Opens the keybindings configuration file
- **Alias:** None

**Usage examples:**
```
/keybindings
```

**Implementation details:**
- Opens the `~/.claude/keybindings.json` configuration file
- Allows custom shortcut mappings inside the terminal

---

### /release-notes

- **Type:** local
- **Function:** Shows release notes for the current version
- **Alias:** None

**Usage examples:**
```
/release-notes
```

**Implementation details:**
- Fetches and displays the changelog for the currently installed version (v2.1.81)
- Includes new features, bug fixes, breaking changes, and other information

---

### /files

- **Type:** local
- **Function:** Lists files loaded in the current context
- **Alias:** None

**Usage examples:**
```
/files
```

**Implementation details:**
- Shows all files loaded into LLM context through the Read tool
- Includes file paths and approximate token usage
- Helps understand which files the LLM has currently "seen"

---

## Skill Type Commands

Skill commands are provided by the plugin system or built-in Skill mechanism and can be dynamically registered at runtime. They are invoked by the LLM through the Skill tool and can also be executed directly as slash commands.

---

### /loop

- **Type:** skill (built-in)
- **Function:** Executes a specified command or prompt repeatedly at a fixed interval (default every 10 minutes)
- **Source extraction:** Full definition reverse-engineered from the v2.1.81 binary
- **Trigger condition:** Automatically triggered when the user requests recurring tasks such as "periodically", "every", or "continuously monitor"

**Usage examples:**
```bash
/loop /review                    # Review code every 10 minutes (default interval)
/loop 5m /babysit-prs            # Check PR status every 5 minutes
/loop 30m check the deploy       # Check deployment every 30 minutes
/loop 1h /standup 1              # Run standup every hour
/loop check the deploy every 20m # Specify interval in natural language
```

**Argument format (defined in source):**
- Interval format: `Ns` (seconds), `Nm` (minutes), `Nh` (hours), `Nd` (days)
- Minimum granularity: 1 minute
- Default interval: `10m` (source variable `OeH = "10m"`)
- If no interval is specified, the default is used automatically

**Implementation mechanism:**
- `/loop` registers the target command as a recurring task
- Each execution is equivalent to re-sending the specified prompt or slash command
- Applicable scenarios: CI/CD monitoring, PR patrols, continuous review, deployment-status polling
- Use `/tasks` to view and manage running recurring tasks

**Difference from /schedule:**
- `/loop` is a **local loop**: it continues running in the current session and stops when the terminal closes
- `/schedule` is **remote scheduling**: it creates a cron task in the Anthropic cloud, independent of the local session

---

### /schedule

- **Type:** skill (built-in)
- **Function:** Creates and manages remotely scheduled Agent tasks using cron (CCR = Claude Code Remote)
- **Source extraction:** Full `RemoteTrigger` tool definition reverse-engineered from the v2.1.81 binary
- **Tool dependencies:** `RemoteTrigger` (loaded on demand through `ToolSearch`), `AskUserQuestion`

**Usage examples:**
```bash
/schedule                        # Interactively choose an operation (create/list/update/run)
/schedule list                   # List all scheduled tasks
/schedule create                 # Create a new scheduled task
/schedule run <trigger_id>       # Run the specified task immediately
```

**Supported operations (source `RemoteTrigger` tool):**

| Operation | Parameters | Description |
|------|------|------|
| `list` | — | List all triggers |
| `get` | `trigger_id` | Get details for a single trigger |
| `create` | `body: {...}` | Create a trigger |
| `update` | `trigger_id`, `body: {...}` | Partially update a trigger |
| `run` | `trigger_id` | Run a trigger immediately |

**Note:** Delete is not supported; it must be done on the `https://claude.ai/code/scheduled` web page.

**Create body structure (defined in source):**
```json
{
  "name": "AGENT_NAME",
  "cron_expression": "0 9 * * 1-5",
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "ENVIRONMENT_ID",
      "session_context": {
        "model": "claude-sonnet-4-6",
        "sources": [
          {"git_repository": {"url": "https://github.com/ORG/REPO"}}
        ],
        "allowed_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep"]
      },
      "events": [{"data": {"uuid": "<v4-uuid>", "type": "user", "content": [{"type": "text", "text": "your task prompt"}]}}]
    }
  }
}
```

**Architecture:**
- Each trigger creates a fully isolated remote session (CCR) in the Anthropic cloud
- Remote sessions have independent Git checkouts, tool sets, and optional MCP connections
- Connects to MCP servers configured on claude.ai through MCP Connector (`mcpsrv_` prefix)
- Authentication is handled internally through OAuth (curl is not used)

---

### /security-review

- **Type:** skill (provided by plugin)
- **Function:** Analyzes code for security vulnerabilities
- **Alias:** None

**Usage examples:**
```
/security-review
/security-review src/auth/
```

**Implementation details:**
- Provided by the security-review plugin
- Analyzes common security vulnerabilities in code:
  - SQL injection, XSS, CSRF
  - Hardcoded credentials and insecure encryption
  - Privilege escalation and path traversal
- Generates a security review report grouped by severity

---

### /pr-comments

- **Type:** skill (provided by plugin)
- **Function:** Handles comments on Pull Requests, automatically replying or making changes
- **Alias:** None

**Usage examples:**
```
/pr-comments
/pr-comments 123
```

**Implementation details:**
- Provided by the pr-comments plugin
- Reads review comments on a PR
- The LLM analyzes comment content and automatically generates replies or modifies code
- Supports inline comments and general comments
- Uses the `gh` CLI to interact with the GitHub API

---

### /simplify

- **Type:** skill (built-in)
- **Function:** Reviews changed code for reuse, quality, and efficiency, then automatically fixes discovered issues
- **Source extraction:** Full prompt reverse-engineered from the v2.1.81 binary (`q7_` variable)

**Usage examples:**
```bash
/simplify
/simplify Focus on performance issues
```

**Three-phase workflow (defined in source):**

**Phase 1: Identify changes** — Run `git diff` (or `git diff HEAD`) to get all changed files

**Phase 2: Start three parallel review agents** — Start them simultaneously through the Agent tool; each agent receives the full diff

| Agent | Review Dimension | Checks |
|------|----------|--------|
| **Agent 1: Code reuse** | Search for existing utility functions | 1. Search for existing utilities/helpers that can replace new code 2. Flag new functions that duplicate existing functionality 3. Flag inline logic that can be replaced by existing tools (handwritten string handling, manual path operations, custom environment checks, ad-hoc type guards, etc.) |
| **Agent 2: Code quality** | Check hacky patterns | 1. **Redundant state**: duplicated existing state, derivable cached values 2. **Parameter bloat**: adding new parameters instead of refactoring 3. **Copy-paste variants**: near-duplicate blocks should be unified into an abstraction 4. **Abstraction leakage**: exposing internal details that should be encapsulated 5. **Stringly typed code**: using raw strings instead of existing constants/enums/branded types 6. **Unnecessary JSX nesting**: wrapper elements with no layout purpose 7. **Unnecessary comments**: comments explaining "what" (good names already explain it), narrative comments, comments aimed at callers; delete them and keep only non-obvious "why" comments |
| **Agent 3: Efficiency review** | Check performance issues | 1. **Unnecessary work**: redundant computation, repeated file reads, repeated API calls, N+1 patterns 2. **Missed concurrency**: independent operations that can run in parallel are executed serially 3. **Hot-path bloat**: blocking work added to startup or per-request/per-render hot paths 4. **No-op updates in loops**: unconditional state updates in polling/timers/event handlers (should guard with change detection) 5. **Unnecessary existence checks**: checking whether a file/resource exists before operating on it (TOCTOU anti-pattern); operate directly and handle errors 6. **Memory**: unbounded data structures, missing cleanup, event-listener leaks 7. **Overly broad operations**: reading an entire file when only part is needed, loading all items just to filter one |

**Phase 3: Fix issues** — Automatically fixes code according to the findings from the three agents

---

### /claude-api

- **Type:** skill
- **Function:** Auxiliary features related to the Claude API
- **Alias:** None

**Usage examples:**
```
/claude-api
```

---

### /update-config

- **Type:** skill
- **Function:** Updates configuration through conversation
- **Alias:** None

**Usage examples:**
```
/update-config
```

---

### /keybindings-help

- **Type:** skill
- **Function:** Shows shortcut help and available key mappings
- **Alias:** None

**Usage examples:**
```
/keybindings-help
```

---

### /statusline

- **Type:** skill
- **Function:** Configures the Claude Code statusline UI
- **Alias:** None

**Usage examples:**
```
/statusline
```

---

### /suggestions

- **Type:** skill
- **Function:** Analyzes usage data and suggests improvements
- **Alias:** None

**Usage examples:**
```
/suggestions
```

---

## Command Quick Index Table

| Command | Type | One-Sentence Description |
|------|------|-----------|
| `/review` | prompt | Reviews code changes and generates a review report |
| `/commit` | prompt | Analyzes the staged area, automatically generates a commit message, and commits |
| `/commit-push-pr` | prompt | One-stop flow for commit + push + PR creation |
| `/init` | prompt | Analyzes the project and generates CLAUDE.md |
| `/init-verifiers` | prompt | Creates code verifier Skills |
| `/insights` | prompt | Session insights / year-in-review |
| `/resume` | local-jsx | Resumes a historical session |
| `/session` | local-jsx | Shows the remote session URL and QR code (alias: /remote) |
| `/rename` | local-jsx | Renames the current session |
| `/tag` | local-jsx | Tags a session |
| `/export` | local-jsx | Exports conversation history |
| `/rewind` | local-jsx | Rewinds to a previous checkpoint (alias: /checkpoint) |
| `/model` | local-jsx | Switches AI model |
| `/config` | local-jsx | Edits configuration |
| `/effort` | local-jsx | Adjusts reasoning effort |
| `/fast` | local-jsx | Switches to fast mode |
| `/permissions` | local-jsx | Manages tool permissions (alias: /allowed-tools) |
| `/hooks` | local-jsx | Manages Prompt Hooks |
| `/privacy-settings` | local-jsx | Privacy settings |
| `/rate-limit-options` | local-jsx | Rate-limit options |
| `/extra-usage` | local-jsx | Extra usage information |
| `/add-dir` | local-jsx | Adds a working directory |
| `/diff` | local-jsx | Views file-change diff |
| `/status` | local-jsx | Shows status information |
| `/plan` | local-jsx | Toggles Plan Mode |
| `/tasks` | local-jsx | Manages background tasks (alias: /bashes) |
| `/brief` | local-jsx | Toggles concise mode |
| `/btw` | local-jsx | Inserts an aside |
| `/mcp` | local-jsx | Manages MCP servers |
| `/plugin` | local-jsx | Manages plugins (aliases: /plugins, /marketplace) |
| `/skills` | local-jsx | Views Skill list |
| `/agents` | local-jsx | Manages multiple agents |
| `/ide` | local-jsx | IDE integration configuration |
| `/memory` | local-jsx | Manages project memory |
| `/login` | local-jsx | Logs in to an account |
| `/logout` | local-jsx | Logs out of an account |
| `/color` | local-jsx | Configures color scheme |
| `/theme` | local-jsx | Switches theme |
| `/copy` | local-jsx | Copies response to clipboard |
| `/stats` | local-jsx | Shows session statistics |
| `/usage` | local-jsx | Shows API usage |
| `/help` | local-jsx | Shows help |
| `/feedback` | local-jsx | Submits feedback |
| `/remote-control` | local-jsx | Remote control mode (alias: /rc) |
| `/remote-env` | local-jsx | Remote environment settings |
| `/desktop` | local-jsx | Continues current session in Claude Desktop (alias: /app) |
| `/mobile` | local-jsx | Shows QR code to download the Claude mobile app (aliases: /ios, /android) |
| `/install-github-app` | local-jsx | Installs GitHub App |
| `/install-slack-app` | local-jsx | Installs Claude Slack app |
| `/reload-plugins` | local-jsx | Activates pending plugin changes in the current session |
| `/chrome` | local-jsx | Claude in Chrome (Beta) setup |
| `/web-setup` | local-jsx | Web integration settings |
| `/exit` | local-jsx | Exits session (alias: /quit) |
| `/install` | local-jsx | Installs native Claude Code build |
| `/upgrade` | local-jsx | Upgrades to Max plan for higher rate limits and more Opus usage |
| `/terminal-setup` | local-jsx | Terminal environment setup |
| `/passes` | local-jsx | Shares free trial week with friends and earns extra usage |
| `/think-back` | local-jsx | Claude Code 2025 year-in-review |
| `/branch` | local-jsx | Creates a branch of the current conversation (alias: /fork) |
| `/compact` | local | Compacts context (three-layer compression system) |
| `/clear` | local | Clears conversation history |
| `/context` | local | Visualizes context usage |
| `/cost` | local | Shows token consumption and cost |
| `/doctor` | local | Diagnoses installation health |
| `/vim` | local | Toggles Vim editing mode |
| `/voice` | local | Toggles voice-input mode |
| `/keybindings` | local | Opens shortcut configuration |
| `/release-notes` | local | Shows version release notes |
| `/files` | local | Lists files in context |
| `/loop` | skill | Executes commands repeatedly |
| `/schedule` | skill | Schedules tasks |
| `/security-review` | skill | Security vulnerability analysis |
| `/pr-comments` | skill | Handles PR comments |
| `/simplify` | skill | Simplifies code |
| `/claude-api` | skill | Claude API assistance |
| `/update-config` | skill | Conversational configuration updates |
| `/keybindings-help` | skill | Shortcut help |
| `/statusline` | skill | Configures Claude Code statusline UI |
| `/suggestions` | skill | Analyzes usage data and suggests improvements |

---

> This document is based on analysis of the Claude Code v2.1.81 binary. The command list and behavior may change as versions are updated.
