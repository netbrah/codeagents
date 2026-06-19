# 10. CLAUDE.md Writing Guide

> How to write an effective CLAUDE.md project instruction file so Claude Code accurately understands your project in every session. Based on Anthropic's official documentation, community best practices (Builder.io, HumanLayer, Arize), and our reverse engineering analysis of the Claude Code binary.

---

## 1. What Is CLAUDE.md

CLAUDE.md is Claude Code's **project instruction file**. Every time a session starts, Claude Code automatically reads this file and injects its contents into the system prompt as behavioral guidance for the entire session.

In essence, it is a **project manual written for AI**:

- Tell Claude Code what tech stack your project uses
- How to build, test, and deploy it
- Which coding conventions to follow
- Which constraints must not be touched

Just as `.editorconfig` standardizes editor behavior, CLAUDE.md standardizes AI agent behavior.

---

## 2. File Hierarchy and Loading Order

Based on binary analysis, Claude Code loads instruction files in the following order; later-loaded content can override earlier content:

| Level | Path | Scope | Committed to Git? |
|------|------|--------|---------------|
| 1 | `~/.claude/CLAUDE.md` | Personal global | No |
| 2 | Project root `CLAUDE.md` | Project-shared | Yes (recommended) |
| 3 | Subdirectory `CLAUDE.md` | Module-level | Yes (optional) |

**Supplemental configuration**: `.claude/settings.local.json` is used for local setting overrides and is not committed to Git.

### Typical Uses for Each Level

**Global `~/.claude/CLAUDE.md`**: Personal coding style preferences that apply across all projects.

```markdown
# Personal Preferences
- Prefer a functional programming style
- Use camelCase for variable names
- Use English for commit messages
- Reply to me in Chinese
```

**Project root `CLAUDE.md`**: Team-shared project standards, committed to version control.

```markdown
# Project: my-saas-app
## Tech Stack
TypeScript + Next.js 15 + Prisma + PostgreSQL

## Build Commands
- Development: `pnpm dev`
- Test: `pnpm test`
- Type check: `pnpm typecheck`
```

**Subdirectory `CLAUDE.md`**: Rules for a specific module that apply only when working in that directory.

```markdown
# packages/shared-ui
This package is a shared UI component library. Every component must:
- Export Storybook stories
- Include unit tests
- Use CSS Modules; inline styles are prohibited
```

---

## 3. Recommended Structure: The WHAT / WHY / HOW Framework

An excellent CLAUDE.md should answer three questions:

### WHAT — What This Project Is

```markdown
## Project Overview
Backend API for an e-commerce platform serving 500,000 daily active users.
- Tech stack: Python 3.12 + FastAPI + SQLAlchemy 2.0 + Redis
- Database: PostgreSQL 16, read/write split
- Deployment: Kubernetes on AWS EKS
- Key dependencies: Stripe (payments), SendGrid (email), S3 (file storage)
```

### WHY — Why It Is Done This Way

```markdown
## Architecture Decisions
- Use the CQRS pattern to separate reads and writes because the read/write ratio is 100:1
- All APIs must be idempotent because clients retry failed requests
- ORM lazy loading is prohibited because it causes N+1 queries
- Monetary calculations must use Decimal; float is prohibited
```

### HOW — How to Operate

```markdown
## Development Commands
- Install dependencies: `uv sync`
- Start development server: `uv run uvicorn app.main:app --reload`
- Run all tests: `uv run pytest`
- Run a single test: `uv run pytest tests/test_orders.py -k "test_create_order"`
- Generate migration: `uv run alembic revision --autogenerate -m "description"`
- Format code: `uv run ruff format .`
- Type check: `uv run mypy app/`

## Coding Standards
- Every API endpoint must have a Pydantic response model
- Database queries are encapsulated in Repository classes
- Error handling uses custom exception classes; do not throw bare HTTPException
- Tests use factory_boy to generate test data; hardcoding is prohibited

## Git Workflow
- Branch naming: `feat/xxx`, `fix/xxx`, `refactor/xxx`
- Commit messages: Conventional Commits format
- PRs must include tests, with coverage no lower than 80%
```

---

## 4. Length Recommendations

**Recommended: keep it under 200 lines**. According to community practice data, CLAUDE.md files under 200 lines have an adherence rate of about 80%; after exceeding 500 lines, adherence drops significantly.

| Length | Adherence rate | Recommendation |
|------|--------|------|
| < 100 lines | ~90% | Ideal length for small projects |
| 100-200 lines | ~80% | Recommended upper limit for medium and large projects |
| 200-500 lines | ~60% | Consider splitting into subdirectories or Skills |
| > 500 lines | < 50% | Must be split, otherwise key instructions will be ignored |

**Reason**: CLAUDE.md content consumes the context window. Overly long instructions compete with code content and conversation history for limited token space.

---

## 5. What to Write

### Write Commands Claude Cannot Guess

```markdown
# Good — Custom scripts that Claude cannot infer from package.json
- Database reset: `./scripts/reset-db.sh --seed`
- Generate API client: `make gen-client`
- Local HTTPS: `mkcert -install && ./scripts/dev-ssl.sh`
```

### Write Conventions That Differ from Defaults

```markdown
# Good — Explicitly state non-standard practices
- Test files live in the `__tests__/` directory (not alongside source files)
- Use tabs for indentation, width 4 (not spaces)
- Import order: stdlib > third-party > local, with a blank line between groups
```

### Write Special Test Runner Usage

```markdown
# Good — Non-standard flags and environment variables
- Integration tests: `TEST_DB=true pytest tests/integration/ -x --timeout=30`
- Snapshot updates: `pnpm test -- -u`
- Coverage report: `pytest --cov=app --cov-report=html`
```

### Write Architecture Decisions and Constraints

```markdown
# Good — Prevent Claude from making incorrect architecture changes
- Do not introduce a new ORM (currently using raw SQL + sqlc)
- All new endpoints must go through API Gateway; do not expose services directly
- Frontend state management uses only Zustand; do not introduce Redux
```

### Write Common Pitfalls

```markdown
# Good — Avoid repeated mistakes
- After `pnpm install`, you must run `pnpm prisma generate`, otherwise type errors occur
- On M1 Mac, `sharp` needs the `--platform=linux` flag (Docker environment)
- In CI, when `NODE_ENV=test`, the database connection string is read from `.env.test`
```

---

## 6. What Not to Write

### Do Not Write Information Claude Can Infer from Code

```markdown
# Bad — Claude can learn this by reading package.json
- The project uses React 18
- ESLint is used for linting
- TypeScript 5.3 is used
```

### Do Not Write Standard Conventions

```markdown
# Bad — Claude already knows these
- Functions should have clear names
- Code should have comments
- Do not use var; use const/let
```

### Do Not Write Detailed API Documentation

```markdown
# Bad — Too long; link instead of inlining
## API Endpoint List
POST /api/users - Create user, parameters: name, email, password...
GET /api/users/:id - Get user details...
(50 more endpoints follow)

# Good — Point to documentation
- API documentation is in `docs/api.md` or at http://localhost:8000/docs
```

### Do Not Describe File by File

```markdown
# Bad — Claude can discover project structure by itself with glob/grep
- src/utils/format.ts: formatting utility functions
- src/utils/validate.ts: validation utility functions
- src/hooks/useAuth.ts: authentication Hook
```

---

## 7. Examples for Different Project Types

### Python (FastAPI + pytest)

```markdown
# CLAUDE.md

## Stack
Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic, pytest

## Commands
- Install: `uv sync`
- Dev: `uv run uvicorn app.main:app --reload --port 8000`
- Test: `uv run pytest -x -q`
- Single test: `uv run pytest tests/test_foo.py::test_bar -v`
- Migrate: `uv run alembic upgrade head`
- Lint: `uv run ruff check . --fix`

## Conventions
- Routes are defined in `app/routers/`, one file per resource
- Business logic belongs in `app/services/`, not in route functions
- All database operations go through the `app/repositories/` layer
- Tests use httpx.AsyncClient; do not use TestClient (async compatibility)
- Environment variables are read through pydantic Settings in `app/config.py`

## Restrictions
- Do not modify existing migration files in `alembic/versions/`
- Do not operate on the database session directly in the route layer
```

### TypeScript (Next.js + Prisma)

```markdown
# CLAUDE.md

## Stack
TypeScript 5.5, Next.js 15 (App Router), Prisma 6, TailwindCSS 4

## Commands
- Install: `pnpm install && pnpm prisma generate`
- Dev: `pnpm dev`
- Test: `pnpm vitest run`
- Test watch: `pnpm vitest`
- Build: `pnpm build`
- DB push: `pnpm prisma db push`

## Conventions
- Use Server Components by default; add "use client" only when necessary
- API Routes live in `app/api/` and use Route Handlers
- Data fetching uses Server Actions or Route Handlers; do not use useEffect + fetch
- Use Tailwind for styling; do not create CSS files
- Use Zod for all form validation and API input validation

## Gotchas
- After `pnpm install`, you must run `pnpm prisma generate`
- After modifying `prisma/schema.prisma`, restart the dev server
```

### Rust (Cargo Workspace)

```markdown
# CLAUDE.md

## Stack
Rust 1.82, Cargo workspace, tokio async runtime

## Structure
Cargo workspace with 3 crates:
- `crates/core/` — Core business logic, no IO dependencies
- `crates/server/` — HTTP server (axum)
- `crates/cli/` — Command-line tool (clap)

## Commands
- Build: `cargo build`
- Test all: `cargo test --workspace`
- Test one crate: `cargo test -p core`
- Run server: `cargo run -p server`
- Clippy: `cargo clippy --workspace -- -D warnings`
- Format: `cargo fmt --all`

## Conventions
- Error handling uses thiserror (library code) and anyhow (application code)
- All pub functions must have documentation comments
- unsafe code needs comments explaining why it is safe
- New dependencies must be declared in [workspace.dependencies] in the root Cargo.toml
```

### Monorepo

```markdown
# CLAUDE.md (root directory)

## Structure
pnpm workspace monorepo:
- `apps/web` — Next.js frontend
- `apps/api` — NestJS backend
- `packages/shared` — Shared types and utility functions
- `packages/ui` — Shared UI components

## Commands
- Install: `pnpm install` (run from the root directory)
- Dev all: `pnpm dev` (turbo starts all apps in parallel)
- Test all: `pnpm test`
- Test single package: `pnpm --filter @repo/web test`
- Build: `pnpm build`

## Important
- After modifying `packages/shared`, rebuild it: `pnpm --filter @repo/shared build`
- Cross-package references use the workspace protocol: `"@repo/shared": "workspace:*"`
- Do not directly import code from another app inside an app
```

---

## 8. CLAUDE.md vs Hooks: Advisory vs Deterministic

Instructions in CLAUDE.md are **advisory**; Claude Code's adherence rate is about 80%. For rules that must be enforced 100% of the time, use Hooks.

| Comparison dimension | CLAUDE.md | Hooks |
|---------|-----------|-------|
| Execution method | Interpreted by the LLM and then followed autonomously | Hard-coded script execution |
| Adherence rate | ~80% (advisory) | 100% (deterministic) |
| Suitable scenarios | Coding style, architecture preferences | Security checks, formatting, automated tests |
| Configuration location | `CLAUDE.md` file | `hooks` field in `.claude/settings.json` |

**Practical recommendation**:

```
CLAUDE.md: Do not commit code containing TODO      -> 80% adherence
Hook: Automatically run lint after PostToolUse     -> 100% execution
```

Put **preferences** in CLAUDE.md and **mandatory rules** in Hooks.

---

## 9. Progressive Disclosure

Do not stuff all information into CLAUDE.md. Use Claude Code's Skills mechanism for layering:

| Level | What to put there | When it loads |
|------|--------|---------|
| CLAUDE.md | Core information needed in every session | Automatically loaded at session start |
| Skills (`.claude/skills/`) | Detailed guidance for specific tasks | Loaded on demand (via `/skill` or automatic matching) |
| External docs | API references, design documents | Claude proactively reads files |

```markdown
# CLAUDE.md contains only a summary
## Database
Uses PostgreSQL + Prisma. For migration standards, see .claude/skills/database-migrations.md

## Deployment
Uses GitHub Actions + AWS ECS. For the deployment flow, see .claude/skills/deployment.md
```

This keeps CLAUDE.md concise, injects detailed guidance on demand, and maximizes context space utilization.

---

## 10. Maintenance Strategy

CLAUDE.md should be maintained like code:

### Prune Regularly

- Review once a month and remove outdated information
- If Claude already gets a rule right on its own (by inferring from code), delete it
- Merge duplicate rules

### Validate by Observation

```bash
# Start a new session, give Claude a relevant task, and observe whether it follows CLAUDE.md
# If it ignores a rule, consider:
# 1. Is the rule worded unclearly? -> Rewrite it
# 2. Is the rule buried in too much content? -> Simplify
# 3. Must the rule be followed? -> Use a Hook instead
```

### Version Control

- Commit the project root CLAUDE.md to Git
- Review CLAUDE.md changes during PR Review
- Note major changes in the commit message

---

## 11. Cross-Tool Compatibility

CLAUDE.md is not only read by Claude Code. The following tools also load it:

| Agent | Reads CLAUDE.md? | Native instruction file |
|------|-------------------|-------------|
| Claude Code | Yes (native) | `CLAUDE.md` |
| Copilot CLI | Yes | `.github/copilot-instructions.md` |
| Codex CLI | No | `CODEX.md` / `AGENTS.md` |
| Gemini CLI | No | `GEMINI.md` |
| Kimi CLI | No | `AGENTS.md` |

If a team uses both Claude Code and Copilot CLI, a single CLAUDE.md can serve both tools.

> If you need one instruction file to cover more tools, see the [AGENTS.md Configuration Guide](agents-md.md).

---

## 12. .claudeignore Reduces Token Waste

The `.claudeignore` file tells Claude Code to ignore specific files and directories, reducing unnecessary file reads and context usage. Its syntax is the same as `.gitignore`.

```gitignore
# .claudeignore
node_modules/
dist/
build/
.next/
coverage/
*.min.js
*.lock
vendor/
__pycache__/
*.pyc
```

**Typical effect**: In a medium-sized Node.js project, after adding `.claudeignore`, Claude Code's file search speed improves by about 30% and noisy files in the context decrease by more than 50%.

---

## 13. Common Mistakes and Fixes

### Mistake 1: Writing It Like a README

```markdown
# Bad — This is an introduction for humans, not instructions for AI
## Welcome to MyApp!
MyApp is a revolutionary platform that helps users manage their tasks
efficiently. It was founded in 2023 and has over 10,000 users...
```

```markdown
# Good — Tell the AI directly what it needs to know
## Stack: TypeScript + Next.js 15 + Supabase
## Build: `pnpm build`
## Test: `pnpm vitest run`
```

### Mistake 2: Contradictory Instructions

```markdown
# Bad — Contradictory
- All functions must have JSDoc comments
- Code should be self-explanatory and use fewer comments
```

```markdown
# Good — Clearly define the scope
- Exported public API functions must have JSDoc comments
- Internal private functions should add comments only when the logic is complex
```

### Mistake 3: Being Too Vague

```markdown
# Bad — Too vague for AI to execute
- Write good code
- Pay attention to security
- Performance should be good
```

```markdown
# Good — Specific and executable
- All SQL queries use parameterization; string concatenation is prohibited
- Add caching when API response time exceeds 200ms
- List endpoints must be paginated, with default page_size=20
```

### Mistake 4: Outdated Information

Regularly check these easily outdated items:
- Dependency version numbers (let Claude read them from package.json/Cargo.toml)
- Team member lists (they do not need to be in CLAUDE.md)
- Deprecated API endpoints (clean them up promptly)
- Completed migration notes (delete after completion)

### Mistake 5: Putting Secrets in CLAUDE.md

```markdown
# Bad — CLAUDE.md will be committed to Git
- API Key: sk-xxxxx
- Database password: mypassword123

# Good — Point to environment variables
- All secrets are read from environment variables; see `.env.example`
- For local development secrets, ask the team lead
```

---

## Summary

| Principle | Description |
|------|------|
| Concise | Keep it under 200 lines and include only information Claude needs but cannot infer |
| Specific | Provide executable commands and rules; avoid vague descriptions |
| Layered | Put core instructions in CLAUDE.md and detailed guidance in Skills |
| Maintained | Review and prune it like code; delete outdated content |
| Complementary | Use CLAUDE.md for advisory rules and Hooks for mandatory rules |

## Related Resources

- [AGENTS.md Configuration Guide](./agents-md.md) — Cross-agent instruction files + symlink (CLAUDE.md -> AGENTS.md)
- [Long-Term Memory vs Project Instructions](../comparison/memory-system-deep-dive.md) — 4-layer CLAUDE.md + auto-memory
- [System Prompts and Prompt Engineering](../comparison/system-prompt-deep-dive.md) — 8-module system prompt architecture
- [Claude Code Overview](../tools/claude-code/01-overview.md) — 79 commands + 24 Hook events
- [Claude Code User Guide](./claude-code-user-guide.md) — 15 practical tips
- [Skill Design Guide](./skill-design.md) — Writing SKILL.md + allowed-tools whitelist
- [Configuration Examples](./config-examples.md) — settings.json format and examples
