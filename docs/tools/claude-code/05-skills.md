# 5. Skills and Plugin System — Developer Reference

> Skills are Claude Code's command extension mechanism: `/commit`, `/review`, `/loop`, and `/batch` are all Skills. This document analyzes the complete implementation of 14 built-in Skills, their loading mechanism, and plugin packaging, providing a reference for Code Agent developers designing extensible command systems.
>
> **Qwen Code comparison**: Qwen Code has 3 bundled skills (loop/review/qc-helper), while Claude Code has 14. The Skill design patterns in this document (Feature Flag gating, dynamic prompt generation, multi-Agent parallelism) can be used directly as references.

## Why a Skill System Is Needed

### Problem Definition

Code Agent feature requirements keep growing — `/commit`, `/review`, `/batch`, `/loop` — but not every feature needs to be compiled into the core code:

| Requirement Type | Built-in Command | Skill |
|---------|---------|-------|
| Clear screen, switch model | ✓ (always needed) | — |
| Code review, commit | — | ✓ (iterative prompt optimization) |
| Project-specific workflow | — | ✓ (user-defined) |
| Experimental feature | — | ✓ (Feature Flag gated) |

**Core advantage**: Skills are pure Markdown — changing SKILL.md adjusts behavior, with **no need to recompile or release a new version**. This is why Qwen Code `/review` can iterate quickly (see [review improvement suggestions](../../comparison/qwen-code-review-improvements.md)).

### Competitor Skill/Extension System Comparison

| Agent | Extension Mechanism | Format | Built-in Count | User-defined |
|-------|---------|------|---------|-----------|
| **Claude Code** | Skill (SKILL.md) + Plugin (plugin.json) | Markdown + YAML Frontmatter | 14 Skills + 13 Plugins | ✓ user/project level |
| **Gemini CLI** | Skill (SKILL.md) + TOML Command | Markdown / TOML | 1 Skill (skill-creator) | ✓ .gemini/skills/ |
| **Qwen Code** | Skill (SKILL.md) | Markdown + YAML Frontmatter | 3 Skills (loop/review/qc-helper) | ✓ user/project level |
| **Copilot CLI** | Plugin (plugin.json packages agents/skills/hooks/MCP) | JSON + Markdown | 13+ Plugins | ✓ marketplace |
| **Cursor** | No standalone extension mechanism | — | — | .cursorrules |

**Key difference**: Claude Code has far more Skills (14) than competitors and covers a wide range of scenarios, from "large-scale parallel changes" (/batch, 5-30 Agents) to "capturing a session workflow" (/skillify, 4 interview rounds). Qwen Code's 3 Skills have limited coverage.

## 1. Skill Definition Format (SKILL.md)

Each Skill is a Markdown file that declares metadata through YAML Frontmatter:

```markdown
---
name: Skill display name
description: Skill description (used by the model to decide when to invoke it)
user-invocable: true          # Whether to show it in the / menu
disable-model-invocation: false # Whether to prevent proactive model invocation
allowed-tools: ["Bash", "Edit", "Read"]  # Allowed tools
argument-hint: "<argument description>"    # Argument hint
when_to_use: "When the user asks..."   # Trigger-condition description
model: sonnet                  # Model to use (optional)
effort: high                   # Reasoning effort level
context: fork                  # Execution context (fork = independent context)
---

Your skill prompt content... can use ${CLAUDE_SKILL_DIR} to reference the skill directory
```

## 2. Skill Loading Paths (Highest to Lowest Priority)

| Source | Path | Description |
|------|------|------|
| Administrator policy | policySettings in `~/.claude/settings.json` | Enterprise control, cannot be overridden |
| User level | `~/.claude/skills/` | Personal global skills |
| Project level | `<project>/.claude/skills/` | Project-shared skills (can be committed to Git) |
| Additional directory | `.claude/skills/` under directories specified by `--add-dir` | Added at runtime |
| Legacy commands | `.claude/commands/` directory (DEPRECATED) | Backward compatibility |

## 3. Skill Loading Flow

```
Startup → Scan all Skill directories
     → Read the Frontmatter of each SKILL.md
     → Parse YAML metadata
     → Deduplicate (SHA hash determines that the same file at different paths is kept once)
     → Feature Flag gate check
     → Stage conditional Skills (activate when matching files are accessed)
     → Register unconditional Skills in the global command table
```

**Conditional activation**: The Frontmatter can set a `paths` field (glob pattern). The Skill is activated only when the user operates on matching files.

## 4. Detailed Breakdown of the 14 Built-in Skills

### 4.1 /batch — Large-Scale Parallel Changes (Key Developer Focus)

**Gate**: none (always available) | **User-invocable**: yes | **Model invocation disabled**: yes

**Design philosophy**: Decompose large-scale mechanical changes (migrations, refactors, bulk renames) into 5-30 independent parallel units, each executed in its own worktree and opened as a PR.

**Three-stage orchestration**:

| Stage | Action | Key Implementation |
|------|------|---------|
| 1. Research/Plan | Understand scope and decompose into independent units | Use the `EnterPlanMode` tool, MIN=5 MAX=30 Agents |
| 2. Spawn Workers | Start all in parallel | `Agent` tool + `isolation: "worktree"` + `run_in_background: true` |
| 3. Track Progress | Render status table + PR links | Updated when background Agents complete |

**Worker template**: simplify → unit tests → e2e → commit → push → PR, ending with `PR: <url>`.

**Developer takeaway**: This is the most complex Skill in Claude Code — it shows how SKILL.md prompts can orchestrate multiple isolated Agents working in parallel. Qwen Code can implement a `/batch` command using a similar pattern.

### 4.2 /simplify — Code Simplification Review

**Gate**: none | **User-invocable**: yes

**Three-Agent parallel review**:

| Agent | Review Dimension | Checks |
|-------|---------|---------|
| Agent 1 | Code reuse | Search for existing utilities, flag duplication, suggest replacements |
| Agent 2 | Code quality | Redundant state, parameter bloat, copy-paste, stringly typed code |
| Agent 3 | Efficiency | Unnecessary computation, concurrency opportunities, hot-path bloat, TOCTOU, memory issues |

After the review, fixes are applied directly (false positives are skipped). Difference from `/review`: `/simplify` focuses on simplifying changed code, while `/review` focuses on finding issues.

### 4.3 /loop — Scheduled Loop Execution

**Gate**: `AGENT_TRIGGERS` Feature Flag | **Runtime check**: `isKairosCronEnabled()`

**Usage**: `/loop 5m /review` (run `/review` every 5 minutes)

**Parsing priority**:
1. Leading interval token: `\d+[smhd]$` (for example `5m`, `2h`)
2. Trailing "every" clause (for example `every 20m`)
3. Default 10 minutes

**Implementation**: parse interval → convert to cron expression → call the `CronCreate` tool → execute once immediately.

### 4.4 /schedule — Remote Scheduled Agent

**Gate**: `AGENT_TRIGGERS_REMOTE` + `tengu_surreal_dali` + `allow_remote_sessions` policy

**Difference from /loop**: `/loop` runs inside the local CLI session (stops when the session closes), while `/schedule` creates a **remote** cron task (survives across sessions).

**Implementation points**:
- Requires claude.ai OAuth authentication (not an API key)
- Base58-decodes MCP server ID (`mcpsrv_` format → UUID)
- Automatically checks repo access permissions (GitHub App / web-setup)
- Converts the user's local time zone to UTC cron automatically

### 4.5 /claude-api — API Development Assistance

**Gate**: `BUILDING_CLAUDE_APPS` | **Tools**: `Read`, `Grep`, `Glob`, `WebFetch`

**Core design**:
- **Automatic language detection**: scans project files to identify Python/TypeScript/Java/Go/Ruby/C#/PHP/curl
- **Lazy-loaded documentation**: 247KB of Markdown documentation is loaded only when invoked (does not consume startup tokens)
- **Language filtering**: injects only the API docs for detected languages
- **Template variables**: `{{OPUS_ID}}`, `{{SONNET_ID}}`, and others are dynamically replaced with current model IDs

**Developer takeaway**: Shows how a Skill can optimize token usage through lazy-loading large documentation + automatic language detection. Qwen Code can use a similar pattern to assist DashScope/OpenAI API development.

### 4.6 /skillify — Capture a Session Workflow as a Skill

**Gate**: none (internal users only) | **Model invocation disabled**: yes

**Four-round interview flow**:

| Round | Content |
|------|------|
| 1 | Confirm skill name, description, goal, and success criteria |
| 2 | High-level steps, parameters, inline vs fork context, save location |
| 3 | Step breakdown: execution type (direct/Task Agent/Teammate/human), deliverables, checkpoints |
| 4 | Confirm trigger phrases and caveats |

**Template variables**: `{{sessionMemory}}`, `{{userMessages}}`, and `{{userDescriptionBlock}}` are extracted from the current session.

Finally, a complete SKILL.md is generated and saved.

### 4.7 /remember — Review and Promote Memory Levels

**Gate**: `isAutoMemoryEnabled()` | **Internal users only**

**Workflow**:
1. Collect all memory layers (CLAUDE.md / CLAUDE.local.md / auto-memory / team memory)
2. Classify each auto-memory entry → target layer:
   - CLAUDE.md: project conventions (visible to all contributors)
   - CLAUDE.local.md: personal preferences
   - Team memory: organization-level knowledge
   - Keep auto-memory: temporary notes
3. Identify cleanup items (duplicate/outdated/conflicting)
4. Output a structured proposal (not executed automatically; requires user confirmation)

### 4.8 /update-config — Configure Settings

**Gate**: none | **Tool**: `Read`

**Dynamic prompt generation**: dynamically generates JSON Schema from the Zod SettingsSchema to keep documentation synchronized with code.

**Hook validation flow** (7 steps): deduplication check → build → pipe-test → write → validate → prove → handoff.

### 4.9 /keybindings-help — Keybinding Customization

**Gate**: `isKeybindingCustomizationEnabled()` | **User-invocable**: no (triggered automatically)

**Dynamic tables**: The Available Contexts and Available Actions tables are dynamically generated from the `KEYBINDING_CONTEXTS` and `KEYBINDING_ACTIONS` source constants, ensuring the documentation always matches the code.

### 4.10 /debug — Session Diagnostics

**Gate**: none | **Tools**: `Read`, `Grep`, `Glob`

**Implementation**: automatically enables debug logs → reads the last 64KB (avoids memory spikes) → displays the last 20 lines → optionally starts a `CLAUDE_CODE_GUIDE_AGENT_TYPE` Subagent to help interpret them.

### 4.11 /stuck — Hang Diagnostics (Internal Only)

Diagnoses Claude Code hangs/high CPU/zombie processes and automatically sends a diagnostic report to the #claude-code-feedback Slack channel.

### 4.12 /verify — Run Verification (Internal Only)

Runs application verification to check whether code changes behave as expected.

### 4.13 /claude-in-chrome — Browser Automation

**Gate**: `shouldAutoEnableClaudeInChrome()` | **Tools**: all `mcp__claude-in-chrome__*`

Automatically invokes MCP tools from the Chrome extension (tabs_context, click, fill, screenshot, and others).

### 4.14 /lorem-ipsum — Long-Context Testing (Internal Only)

Generates filler text with a specified token count. Uses a prevalidated list of 200+ single-token English words, capped at 500K tokens.

## 5. Summary of Key Design Patterns

| Pattern | Skills Using It | Developer Takeaway |
|------|------------|-----------|
| **Multi-Agent parallelism** | batch (5-30 Agents), simplify (3 Agents) | Orchestrate parallel work with prompts, no code changes required |
| **Feature Flag gating** | loop, schedule, claude-api | Experimental features can be safely rolled out gradually |
| **Lazy-loading large docs** | claude-api (247KB) | Avoid consuming tokens at startup |
| **Dynamic prompt generation** | update-config (Schema→docs), keybindings (constants→tables) | Keep documentation synchronized with code |
| **Four-round interview** | skillify | Interactive guidance for creating complex Skills |
| **Worktree isolation** | batch | Parallel Agents do not interfere with each other |
| **Conditional activation** | keybindings, remember, loop | Load on demand to reduce menu noise |
| **Template variables** | skillify (`{{sessionMemory}}`), claude-api (`{{OPUS_ID}}`) | Inject context at runtime |
| **Model invocation disabled** | batch, skillify, debug | User-triggered only, preventing the LLM from invoking it on its own |

## 6. Plugin System

Claude Code manages plugins through the `/plugin` command and supports installation from a marketplace.

### Plugin Structure
```
.claude-plugin/
  plugin.json            # Plugin metadata and configuration
  commands/              # Custom slash commands
  agents/                # Agent templates
  skills/                # Skill definitions
  hooks/                 # Hook scripts
  .mcp.json              # Plugin MCP server configuration
```

### Official Plugins at a Glance

| Plugin | Core Capability | Implementation Pattern |
|------|---------|---------|
| **code-review** | Multi-Agent parallel PR review + confidence filtering | 9-step pipeline, 4 parallel Agents |
| **pr-review-toolkit** | 6 specialized review Agents | Comments/tests/error handling/types/quality/simplification |
| **feature-dev** | 7-stage guided feature development | Discovery → exploration → questions → architecture → implementation → review → reflection |
| **security-guidance** | Secure coding guidance | PreToolUse Hook intercepts unsafe operations |
| **hookify** | Automatically create Hooks from conversation analysis | Detect frustration signals → generate rule file → take effect immediately |
| **commit-commands** | Git commit/push/create PR | Analyze diff + historical style + generate message |

### feature-dev 7-Stage Flow

| Stage | Core Action |
|------|---------|
| 1. Discovery | Understand requirements and confirm the problem and constraints |
| 2. Exploration | 2-3 parallel explorer Agents each investigate different aspects |
| 3. Questions | **Critical**: identify ambiguity, edge cases, integration points, and ask the user |
| 4. Architecture | 2-3 parallel architect Agents: minimal change vs clean architecture vs pragmatic balance |
| 5. Implementation | Implement according to the selected plan |
| 6. Review | Code review and testing |
| 7. Reflection | Retrospective and summary |

## Supplement: Skill Discovery in Monorepos

> Reference: the Monorepo Skills discovery report from [claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)

**Skills and CLAUDE.md load differently**: CLAUDE.md traverses upward through the directory tree (ancestor loading), while Skills use nested-directory discovery on demand.

### Discovery Levels

| Level | Path | Scope |
|------|------|---------|
| Enterprise | Managed settings | All users in the organization |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All projects |
| Project | `.claude/skills/<name>/SKILL.md` | Current project |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | When the plugin is enabled |
| **Nested directory** | `packages/frontend/.claude/skills/<name>/SKILL.md` | **Discovered on demand when editing files in that directory** |

### On-Demand Discovery

Nested Skills are not preloaded at session startup; they are discovered only when the user edits files in the corresponding subdirectory. A Skill's description field is always in context (so Claude knows which Skills are available), but the full content is loaded only when the Skill is invoked. Skills preloaded by Subagents (the `benefits-from` field) are the exception: their full content is injected when the Agent starts.

**Qwen Code comparison**: Qwen Code's Skill discovery mechanism comes from the Gemini CLI fork; it needs verification for nested-directory on-demand discovery and separated description/content loading.
