# 2. Claude Code User Guide

> A complete guide from installation to mastery, for both new and advanced users.

---

## Quick Start (2 Minutes)

### Installation

```bash
# Recommended method
curl -fsSL https://claude.ai/install.sh | bash

# macOS Homebrew
brew install --cask claude-code

# Verify
claude --version
```

### First Use

```bash
# Enter the project directory
cd your-project

# Start an interactive session
claude

# Ask a question directly
claude "What is this project's tech stack?"

# Pipe mode (for scripts/CI)
claude -p "Fix the bug in src/auth.ts" > fix.txt
```

On first launch, Claude Code will guide you through logging in to your Anthropic account.

---

## Daily Use

### Conversational Development

```
You: Fix the form validation on the login page
Claude: [Reads relevant files] -> [Analyzes code] -> [Edits files] -> [Runs tests]
        Fixed the validation logic in src/components/LoginForm.tsx.
        Main changes: added email format and password strength checks.

You: Also add a username length limit
Claude: [Edits files] -> Added a 3-20 character username length limit.

You: /commit
Claude: [Analyzes diff] -> [Generates message] -> Committed: "Add form validation for login page"
```

### Common Operations Cheat Sheet

| What you want to do | Command |
|-----------|------|
| Ask a question without modifying code | Ask directly (Claude will decide whether editing is needed) |
| Review code | `/review` or `/code-review --comment` |
| Commit code | `/commit` |
| Commit + push + create PR | `/commit-push-pr` |
| Simplify and optimize code | `/simplify` |
| Switch model | `/model` (Sonnet for daily work, Opus for complex tasks) |
| View cost | `/cost` |
| Compact context | `/compact` (or wait for automatic triggering) |
| Resume a previous session | `/resume` |
| Rewind to a previous state | `/rewind` or press `Esc` |

---

## Understanding the Command System

Claude Code has **~79 commands**, divided into four types:

### prompt Type (Executed by the LLM)

These commands send prompts to the LLM, and the LLM uses tools to complete the task.

```bash
/review          # Code review (4-6 parallel agents with confidence filtering)
/commit          # Git commit (analyzes diff and generates message)
/commit-push-pr  # One-click commit + push + create PR
/init            # Initialize project (generate CLAUDE.md)
/simplify        # Code simplification (3 parallel agents: reuse/quality/efficiency)
/batch           # Large-scale parallel changes (5-30 worktree agents)
```

### local-jsx Type (Local UI)

Renders terminal UI components without calling the LLM.

```bash
/model           # Model selector
/config          # Configuration panel
/permissions     # Permission management
/mcp             # MCP server management
/memory          # Memory editor
/resume          # Session resume selector
/plan            # Planning mode toggle
/diff            # View file changes
/hooks           # Hook configuration viewer
```

### local Type (Direct Execution)

Runs directly locally without calling the LLM.

```bash
/compact         # Compact context (three-layer compression algorithm)
/clear           # Clear conversation history
/context         # Visualize context usage (colored grid)
/cost            # View token consumption and cost
/doctor          # Diagnose installation health
/vim             # Toggle Vim editing mode
/voice           # Voice mode (Push-to-talk)
/rewind          # Rewind to a previous checkpoint
```

### Skill Type (Extensible)

Defined through SKILL.md files; you can create your own.

```bash
/loop 5m /review     # Run a review every 5 minutes
/schedule            # Remote scheduled tasks (CCR cloud)
/security-review     # Security review (plugin)
/simplify            # Code simplification (built-in Skill)
```

---

## Advanced Tips

### 1. Use Subagents for Parallel Speedups

```
You: Please complete these three tasks in parallel:
     1. Fix the bug in the auth module
     2. Add unit tests for utils
     3. Update the README documentation

Claude: [Starts 3 parallel subagents]
        Agent 1: Fixing auth...
        Agent 2: Writing tests...
        Agent 3: Updating docs...
        All done.
```

Claude will automatically use the Agent tool to start subagents. You can also explicitly ask it to work "in parallel".

### 2. Use /btw to Avoid Context Pollution

```
You: [Discussing a complex database migration]

You: /btw How do I use TypeScript's Record type?

Claude: [Answers in an isolated context without affecting the main conversation]
        Record<K, V> creates an object type whose keys are K and values are V...

You: [Continue the database migration discussion; the context is not polluted]
```

### 3. Customize CLAUDE.md

Create `CLAUDE.md` in the project root:

```markdown
# Project Standards

## Tech Stack
- Next.js 14 + TypeScript + Tailwind
- PostgreSQL + Prisma ORM

## Build Commands
- `pnpm test` runs tests
- `pnpm lint` runs lint checks

## Coding Standards
- Functional components + React Hooks
- Conventional Commits format
- Do not modify the prisma/migrations/ directory
```

Claude will automatically read and follow these standards in every session. `/review` also checks whether code violates rules in CLAUDE.md.

### 4. Configure Permission Automation

Edit `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "Bash(pnpm test)", "Bash(pnpm lint)",
      "Bash(git:*)", "Bash(gh pr:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)", "Write(.env*)"
    ]
  }
}
```

With this, Claude will no longer ask for confirmation every time it runs tests, lint, or git operations.

### 5. Extend Capabilities with MCP

Create `.mcp.json` (project-level):

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {"DATABASE_URL": "postgresql://localhost:5432/mydb"}
    }
  }
}
```

Claude can now query your database directly.

### 6. Hook Automation

Use `/hookify` to automatically create a Hook:

```
You: /hookify Do not commit directly on the main branch

Claude: [Analyzes requirement] -> Created .claude/hookify.block-main-push.local.md
        The rule takes effect immediately; no restart required.
```

Or manually create `.claude/hookify.warn-env.local.md`:

```markdown
---
name: warn-env-file
enabled: true
event: bash
pattern: cat.*\.env|echo.*\.env
action: warn
---

WARNING: Detected an operation that may read a .env file. Confirm that sensitive information will not be leaked.
```

### 7. Use /plan Mode for Safe Exploration

```bash
/plan          # Enter planning mode (read-only; does not modify any files)

You: Analyze this project's architecture and draw the module dependencies

Claude: [Only reads files, does not edit]
        Project architecture analysis:
        src/
        ├── api/ -> depends on services/
        ├── services/ -> depends on models/
        └── models/ -> independent
        Suggest moving utils/auth.ts to services/ ...

/plan          # Exit planning mode and restore normal editing capabilities
```

### 8. Use Models Efficiently

| Scenario | Recommended model | Command |
|------|---------|------|
| Daily coding | Sonnet 4.6 | `/model sonnet` (default) |
| Complex reasoning | Opus 4.6 | `/model opus` |
| Quick Q&A | Haiku 4.5 | `/model haiku` |
| Large-file analysis | Sonnet 4.6[1m] | `/model sonnet[1m]` |

Opus is 5x more expensive than Sonnet but has stronger reasoning. Use Sonnet for simple tasks and Opus for complex architecture decisions.

---

## Workflow Examples

### Workflow 1: Fix a Bug

```bash
claude                              # Start
You: User reports that the session is lost after login       # Describe the issue
# Claude automatically: reads code -> locates issue -> fixes -> runs tests
/review                             # Self-review
/commit                             # Commit
```

### Workflow 2: Develop a New Feature

```bash
/plan                               # Enter planning mode first
You: Add user avatar upload
# Claude performs read-only analysis, does not modify files, and outputs an implementation plan
/plan                               # Exit planning mode

You: Implement according to the plan
# Claude edits files, creates components, and updates routes
/simplify                           # Improve code quality
/review                             # Review
/commit-push-pr                     # Commit + push + create PR
```

### Workflow 3: Continuous Monitoring

```bash
/loop 10m /review                   # Review code every 10 minutes
# Or
/schedule                           # Set up a remote scheduled task
```

### Workflow 4: Large-Scale Refactoring

```bash
/batch Replace all console.log calls with logger.info
# Claude automatically:
# 1. Analyzes scope
# 2. Starts 5-30 parallel agents (each in an independent worktree)
# 3. Each agent handles part of the files
# 4. Aggregates results
```

---

## Cost Management

### View Cost

```bash
/cost              # View current session cost
/usage             # View overall usage
/stats             # Detailed statistics
```

### Money-Saving Tips

| Tip | Method | Savings |
|------|------|------|
| Use smaller models for simple tasks | `/model haiku` for answering questions | 5-10x |
| Compact promptly | Run `/compact` at task boundaries | Reduces accumulated tokens |
| Use caching | Keep session continuity (`claude -c`) | Prompt Caching takes effect |
| Subscription vs API | Choose Max $100 if daily average is >$3.3 | Capped maximum |
| Plan first | Use `/plan` to think clearly before acting | Reduces rework |

### Plan Comparison

| Plan | Monthly fee | Best for |
|------|------|------|
| Pro | $20 | Light use |
| Max 5x | $100 | Daily development (Opus by default) |
| Max 20x | $200 | Heavy use |
| API | Pay-as-you-go | CI/CD and batch tasks |

---

## FAQ

### Claude Does Not Understand My Project

```bash
/init              # Let Claude analyze the project and generate CLAUDE.md
```

Or manually create `CLAUDE.md` to tell Claude the project's tech stack and conventions.

### The Context Is Too Long and Claude Starts Forgetting

```bash
/compact           # Compact manually
/compact Keep the database-related discussion    # Compact while preserving a specific topic
```

### I Want to Go Back to an Earlier State

```bash
/rewind            # Or press Esc
# Select a rewind point and restore files and conversation
```

### Claude Performed an Operation I Did Not Want

Press `Esc` to interrupt the current operation immediately. Use `/rewind` to roll back.

Set permissions to prevent this:
```json
{"permissions": {"deny": ["Bash(rm -rf:*)", "Write(.env*)"]}}
```

### I Want to Switch Between Multiple Projects

```bash
claude --resume    # Resume a previous session
claude -c          # Continue the most recent conversation
```

### I Want Claude to Be More Concise / More Detailed

```bash
/config            # Open the configuration panel and adjust output style
# Or specify it in CLAUDE.md:
# "Keep answers concise, no more than 3 sentences each time"
```

---

## Security Notes

Claude Code includes one of the industry's strictest security systems:

- **28 operation-blocking rules** (such as prohibiting force push, prohibiting deletion of shared resources, and prohibiting data exfiltration)
- **Two-stage security classifier** (in auto mode, an independent LLM evaluates the risk of each operation)
- **5-layer settings precedence** (Managed > CLI > Local > Project > User)
- **Sandbox isolation** (macOS Seatbelt / Linux Docker)

You do not need to configure any security settings -- it is secure by default. But if you run in auto mode (`--permission-mode auto`), the security classifier provides extra protection.

---

---

## High-Frequency Tips (from Community Best Practices)

> The following tips synthesize high-quality guides from Builder.io, Anthropic official documentation, DataCamp, awesome-claude-code, and others.

### 9. Use `.claudeignore` to Reduce Token Waste

Create `.claudeignore` in the project root (similar to .gitignore):

```
node_modules/
dist/
build/
*.min.js
*.map
coverage/
.next/
```

> **Effect:** A typical Node.js project reduces token consumption by ~25%. Claude will not read ignored files.

### 10. Feedback Loop: Let Claude Self-Verify

**Highest-leverage technique** (officially recommended by Anthropic; improves quality by 2-3x):

```
You: Implement an email validation function.
     Test cases: user@example.com -> true, "invalid" -> false.
     Run the tests after implementation.

You: [Paste screenshot] Implement this design. After finishing, compare a screenshot against the original, list the differences, and fix them.
```

The key: **give Claude a way to check its own work** -- a test command, linter output, screenshot comparison, etc.

### 11. CLAUDE.md Best Practices

**Control length:** Keep it under 200 lines. If it is too long, rules may be ignored (~80% adherence vs a drop when too long).

**WHAT/WHY/HOW framework:**
```markdown
## WHAT (Tech stack and structure)
Next.js 14 + TypeScript + Prisma

## WHY (Purpose and constraints)
E-commerce platform; must be SEO-friendly and support multiple languages

## HOW (Workflow rules)
- Write tests before implementation
- Run pnpm lint before committing
```

**Multiple levels:**
- `~/.claude/CLAUDE.md` — Personal global defaults
- `project root/CLAUDE.md` — Project-shared instructions
- `subdirectory/CLAUDE.md` — Module-level override

**Key distinction:** CLAUDE.md is **advisory** (~80% adherence). Use **Hooks** for rules that require 100% enforcement.

### 12. `/clear` Discipline

```bash
# After completing one task, before starting another
/clear

# Still wrong after 3 fixes? Do not keep patching -- clear and start over
/clear
# Restart with a better initial prompt
```

> **Community consensus:** A clean session + a better prompt is almost always better than a long session + multiple corrections.

### 13. Plan Before Execution (the Most Commonly Recommended Pattern)

```
You: I want to add Google OAuth login.
     First create a plan; do not write code.
     List the files and steps that need changes.

Claude: [Read-only analysis] Plan:
        1. Install next-auth
        2. Modify src/auth/...
        ...

You: The plan looks good. Start implementation.
     After writing it, run the test suite and fix any failures.
```

> **Rule of thumb:** If you can describe the diff in one sentence, skip planning. Otherwise, plan first.

### 14. Separate Writer and Reviewer

```bash
# Session A: implementation
claude
You: Implement API rate-limiting middleware

# Session B (new session): review
claude
You: Review @src/middleware/rateLimiter.ts
     Check edge cases, races, and consistency with existing middleware

# Session A: fixes
You: Here is the review feedback: [paste Session B output]
     Please fix these issues.
```

> **Principle:** A new session does not have the bias of "the code I just wrote", so the review is more objective.

### 15. Use `@` to Reference Files

```
You: Follow the implementation pattern in @src/widgets/HotDogWidget.tsx
     and create a new CalendarWidget

You: Fix the token refresh logic around @src/auth/session.ts:42
```

> **One good example file is better than ten paragraphs of description.**

---

## Further Reading

- [Complete Reference for 79 Commands](../tools/claude-code/02-commands.md)
- [Technical Architecture (Decompiled Analysis)](../tools/claude-code/03-architecture.md)
- [Skills and Plugin System](../tools/claude-code/05-skills.md)
- [Settings and Security (Including 28 BLOCK Rules)](../tools/claude-code/06-settings.md)
- [Configuration Example Comparison](./config-examples.md)
- [/review Deep Dive](../comparison/review-command.md)
- [/simplify Deep Dive](../comparison/simplify-command.md)
- [Official Documentation](https://code.claude.com/docs)
