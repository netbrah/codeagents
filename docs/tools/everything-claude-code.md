# Everything Claude Code (ECC) — AI Agent Enhancement System

**Repository:** [github.com/affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)
**License:** MIT
**Website:** [ecc.tools](https://ecc.tools)
**Version:** v1.9.0 (2026-03-21)
**Last updated:** 2026-03

## Overview

Everything Claude Code (ECC for short) is a **performance optimization and enhancement system** for AI Agents. Through 28 specialized Agents, 125+ Skills, 60 slash commands, 34 rules, and Hook automation, it upgrades tools such as Claude Code from coding assistants into comprehensive development platforms with continuous learning, persistent memory, security scanning, and research-driven development.

ECC was created by affaanmustafa (Anthropic Hackathon winner, September 2025) and iterated through 10+ months of daily production use. As of March 2026, the GitHub repository has ~112k stars, ~14.6k forks, 113 contributors, 816 commits, and 11 releases.

**Core positioning:** ECC is not a standalone Agent tool, but an **enhancement layer** installed onto existing Agents to provide additional Agents, Skills, Commands, Rules, and Hooks.

## Supported Agent Tools

ECC is the first enhancement plugin spanning mainstream AI coding tools:

| Tool | Agent Count | Command Count | Skill Count | Hook Events | Rule Count |
|------|---------|--------|---------|----------|--------|
| **Claude Code** | 28 | 60 | 125+ | 8 event types | 34 |
| **Cursor IDE** | Shared (AGENTS.md) | Shared | Shared | 15 event types | 34 |
| **Codex CLI/App** | Shared | Instruction-based | 16 | — | Instruction-based |
| **OpenCode** | 12 | 31 | 37 | 11 event types | 13 |

Core designs of the cross-tool architecture:
- **AGENTS.md** serves as a universal cross-tool file readable by all tools
- **DRY Adapter pattern**: Cursor's stdin JSON is transformed through `adapter.js`, then reuses Claude Code Hook scripts
- **SKILL.md format**: a cross-tool common format for Skill definitions

## System Requirements

- Claude Code CLI v2.1.0+
- Node.js (for the Hook script runtime)

---

## Core Components

### 1. 28 Specialized Agents

ECC Agents are Markdown files with YAML frontmatter defining `name`, `description`, `tools` (available tool list), and `model` (recommended model). Agents handle delegated tasks with restricted scope and tool access.

| Category | Agent | Description |
|------|-------|------|
| **Planning** | planner | Creates implementation blueprints |
| **Architecture** | architect | System architecture design |
| **Testing** | tdd-guide, e2e-runner | TDD workflow guidance and E2E testing |
| **Review** | code-reviewer, security-reviewer, cpp-reviewer, go-reviewer, python-reviewer, typescript-reviewer, java-reviewer, kotlin-reviewer, rust-reviewer, database-reviewer | Code review by language/domain |
| **Build Fixing** | build-error-resolver, cpp-build-resolver, go-build-resolver, java-build-resolver, kotlin-build-resolver, rust-build-resolver, pytorch-build-resolver | Build error resolution by ecosystem |
| **Operations** | refactor-cleaner, doc-updater, docs-lookup, chief-of-staff, loop-operator, harness-optimizer | Refactoring, documentation, loop operations, and performance optimization |

Agent file format example:

```markdown
---
name: code-reviewer
description: Reviews code for quality, security, and maintainability
tools: ["Read", "Grep", "Glob", "Bash"]
model: opus
---
You are a senior code reviewer...
```

### 2. 125+ Skills

Skills are workflow definitions and domain knowledge bases stored in SKILL.md format and invoked through commands or Agents. They cover the following categories:

| Category | Skill Examples |
|------|-----------|
| **Coding Standards** | TypeScript, Python, Go, C++, Java, Swift, Perl, Rust, and PHP coding guidelines |
| **Framework Patterns** | Django, Spring Boot, Laravel, Next.js, and PyTorch best practices |
| **Testing** | TDD workflow, Playwright E2E, verification loops, Eval Harness |
| **Security** | AgentShield integration and security review |
| **DevOps** | Docker, deployment, database migrations, API design |
| **AI/ML** | Cost-aware LLM pipelines, regex-vs-LLM decisions, on-device foundation models |
| **Operations** | continuous-learning v2, autonomous-loops, strategic-compact |
| **Business/Content** | Article writing, market research, investor materials, content engine |

### 3. 60 Slash Commands

| Category | Commands |
|------|------|
| **Development Process** | `/plan`, `/tdd`, `/code-review`, `/build-fix`, `/e2e`, `/refactor-clean` |
| **Security and Quality** | `/security-scan`, `/verify`, `/eval`, `/quality-gate` |
| **Learning and Memory** | `/learn`, `/checkpoint`, `/instinct-import`, `/instinct-export`, `/instinct-status`, `/evolve`, `/prune`, `/skill-create` |
| **Orchestration** | `/orchestrate`, `/sessions`, `/multi-plan`, `/multi-execute`, `/model-route` |
| **Operations** | `/harness-audit`, `/loop-start`, `/pm2`, `/setup-pm` |

### 4. 34 Rules

Rules are **always-on guidance principles** loaded into the system prompt and divided into:

- `common/`: language-independent rules (always installed) — coding style, Git workflow, testing, performance, patterns, security
- Language-specific directories: `typescript/`, `python/`, `golang/`, `swift/`, `php/`, `java/`, `rust/`, `cpp/`, `kotlin/`, `perl/`

Installation locations: user-level `~/.claude/rules/` or project-level `.claude/rules/`.

### 5. Hook Automation

Hooks trigger automation based on Claude Code's 24 event types:

| Hook Event | Purpose |
|----------|------|
| PreToolUse | Intercept before tool calls (for example, block sensitive operations) |
| PostToolUse | Process after tool calls (for example, automatically check `console.log` after Edit) |
| Stop | Save state when a session ends |
| SessionStart | Load context when a session starts |
| SessionEnd | Save context when a session ends |

Runtime control:

```bash
export ECC_HOOK_PROFILE=standard     # minimal | standard | strict
export ECC_DISABLED_HOOKS="pre:bash:tmux-reminder,post:edit:typecheck"
```

---

## Continuous Learning System (Instincts v2)

ECC's Instinct system is a core differentiating capability that can **automatically learn user patterns from sessions**:

```
Patterns produced during a session
  → /learn extracts patterns → Instinct (with confidence score)
  → /evolve clusters related Instincts → Skill
  → /skill-create generates Skill from Git history
```

| Command | Function |
|------|------|
| `/instinct-status` | Shows learned Instincts and their confidence scores |
| `/instinct-import` | Imports another person's Instincts |
| `/instinct-export` | Exports Instincts for team sharing |
| `/evolve` | Clusters related Instincts into a reusable Skill |
| `/prune` | Deletes expired pending Instincts (30-day TTL) |
| `/learn` | Extracts patterns from a session |
| `/learn-eval` | Extracts, evaluates, and saves patterns |

Each Instinct includes a confidence score and supports import/export for sharing across teams.

---

## Memory Persistence

Cross-session memory persistence is implemented through session lifecycle Hooks:

| Hook Script | Function |
|----------|------|
| `session-start.js` | Loads context when a session starts |
| `session-end.js` | Saves state when a session ends |
| `pre-compact.js` | Saves state before compaction |
| `suggest-compact.js` | Strategic compaction suggestions |
| `evaluate-session.js` | Extracts patterns from a session |

**Strategic compaction** (`strategic-compact`): suggests `/compact` at logical breakpoints instead of waiting for automatic compaction at 95% context.

Compaction best practices:
- Compact after research/exploration is complete, after milestones are reached, and after debugging is complete
- **Do not** compact in the middle of implementation (variable names, file paths, and intermediate state may be lost)
- Recommended setting: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE: 50` (compact earlier to preserve quality)

---

## Installation

### Method 1: Plugin Installation (Recommended)

```bash
# Install from Plugin Marketplace
/plugin marketplace add affaan-m/everything-claude-code
/plugin install everything-claude-code@everything-claude-code
```

> **Note**: Rules must be installed manually (a limitation of the Plugin system), under `~/.claude/rules/` or `.claude/rules/`.

### Method 2: Manual Installation

```bash
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code
npm install

# Select a language ecosystem to install
./install.sh typescript   # macOS/Linux
.\install.ps1 typescript   # Windows
# Or
npx ecc-install typescript
```

v1.9.0 supports manifest-based selective installation — you can install only the components you need.

### Token Optimization Recommendations

```json
{
  "model": "sonnet",
  "env": {
    "MAX_THINKING_TOKENS": "10000",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "50",
    "CLAUDE_CODE_SUBAGENT_MODEL": "haiku"
  }
}
```

| Setting | Default | Recommended | Effect |
|------|--------|--------|------|
| `model` | opus | sonnet | ~60% cost reduction |
| `MAX_THINKING_TOKENS` | 31,999 | 10,000 | ~70% thinking cost reduction |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 95 | 50 | Earlier compaction to preserve quality |

---

## Workflow Examples

### TDD Development Workflow

```
/everything-claude-code:plan "Add user authentication with OAuth"
  → planner agent creates an implementation blueprint

/tdd
  → tdd-guide agent enforces test-first development

/code-review
  → code-reviewer agent reviews code quality
```

### Typical Interaction Pattern

```
User command → command triggers Agent (restricted tools + model)
  → Agent references Skill (domain knowledge)
  → Hook automatically triggers on tool events
  → Rules are always active as background guidance
```

---

## Ecosystem Tools

### AgentShield

A security audit tool for scanning security risks in Claude Code configuration.

```bash
npx ecc-agentshield scan
```

- 1,282 tests
- 102 static analysis rules
- 5 scan categories

### Skill Creator

The `/skill-create` command supports local analysis and can also access advanced features through a GitHub App.

### ECC Tools GitHub App

Marketplace app with free/pro/enterprise tiers.

---

## Version History

| Version | Date | Highlights |
|------|------|------|
| **v1.9.0** | 2026-03 | Manifest-based selective installation, 6 new Agents (TS/Java/Kotlin/PyTorch), SQLite state storage, 12 language ecosystem rules |
| **v1.8.0** | 2026-03 | "Harness Performance System" repositioning, Hook reliability refactor, `ECC_HOOK_PROFILE` runtime control, 997 tests |
| **v1.7.0** | 2026-02 | Codex App + CLI support, 5 business/content Skills, 992 tests |
| **v1.6.0** | 2026-02 | Codex CLI support, AgentShield integration (1,282 tests/102 rules), GitHub Marketplace App |
| **v1.4.0** | 2026-02 | Interactive installation wizard, PM2 & multi-Agent orchestration, multilingual rule architecture, Chinese translation |
| **v1.3.0** | 2026-02 | Full OpenCode integration (12 Agents/24 commands/16 Skills/3 native tools) |
| **v1.2.0** | 2026-02 | Python/Django + Java Spring Boot support, session management, Continuous Learning v2 |

---

## Project Statistics

| Metric | Value |
|------|------|
| Stars | ~112k |
| Forks | ~14.6k |
| Contributors | 113 |
| Commits | 816 |
| Releases | 11 |
| Internal tests | 997+ |
| Supported languages | 7 (English, Portuguese, Simplified Chinese, Traditional Chinese, Japanese, Korean, Turkish) |
| Code composition | JavaScript 81.5%, Python 6.3%, Rust 5.8%, Shell 4.6%, TypeScript 1.8% |

> **Disclaimer**: The data above is based on a March 2026 GitHub repository snapshot and may be outdated.

## Limitations

- **Host tool dependency**: ECC is not a standalone Agent; it must be installed on tools such as Claude Code/Cursor/Codex/OpenCode
- **Token overhead**: 125+ Skills, 34 Rules, and 28 Agents increase system prompt Token usage — use together with Token optimization settings
- **MCP management**: Recommended to keep MCP servers < 10 and tools < 80; every MCP tool description consumes Token
- **Plugin limitation**: Rules must be installed manually; the Plugin system does not yet support automatic Rule distribution

## Evidence Sources

| Data | Source |
|------|------|
| Project structure and component counts | GitHub README + repository directory structure |
| Version history | CHANGELOG.md + Release page |
| Installation methods | README.md installation section |
| Cross-tool architecture | README.md "Cross-Harness Support" section |
| AgentShield | README.md "Ecosystem Tools" section |
