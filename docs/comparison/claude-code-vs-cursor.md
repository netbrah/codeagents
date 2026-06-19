# 15. Claude Code vs Cursor: Terminal Agent vs AI IDE

> Claude Code (Anthropic's terminal-native agent) vs Cursor (AI-native IDE) -- a comprehensive comparison of two major commercial AI coding tools. One is rooted in the terminal; the other reshapes the editor. Together they represent two paths for AI-assisted development.

## Positioning Comparison

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| **Developer** | Anthropic | Anysphere |
| **License** | Proprietary (closed source) | Proprietary (closed source) |
| **Form factor** | Terminal CLI agent | VS Code fork IDE |
| **Pricing** | API token pay-as-you-go / Max subscription | Subscription (Free / Pro / Business) |
| **Core models** | Claude series (locked) | Multi-model (Claude / GPT / Gemini) |
| **Architecture** | Native Rust CLI | Electron + VS Code extension architecture |
| **Implementation language** | Rust | TypeScript |
| **Operating systems** | macOS / Linux (terminal) | macOS / Linux / Windows |
| **Context window** | 1M tokens (Opus 4.6) | Model-dependent, up to 200k tokens |
| **Target users** | Terminal-heavy users and advanced developers | Full spectrum of developers and teams |

---

## 1. Interaction Model

### Claude Code: Terminal CLI

```
$ claude "Refactor the auth module"
> Analyzing project structure...
> Reading src/auth/*.ts (12 files)
> Creating refactor plan...
> Applying file changes...
> Running tests for validation...
✓ Refactor complete, modified 8 files
```

- **REPL interaction**: real-time conversation with AI in the terminal, with streaming output
- **Pipeline integration**: works seamlessly with Unix tools such as `git`, `grep`, and `jq`
- **Headless mode**: `claude -p` supports scripted invocation and CI/CD integration
- **SSH-friendly**: can be used directly on remote servers without a GUI

### Cursor: AI-native IDE

```
Cursor editor interface
├── Code editor area (real-time inline suggestions)
├── Chat panel (side conversation)
├── Composer (multi-file editing agent)
└── Terminal (built-in + AI assistance)
```

- **Inline completion**: press Tab to accept AI code suggestions; experience is close to Copilot
- **Visual diffs**: preview AI edits directly in the editor and accept/reject chunk by chunk
- **Cmd+K editing**: select code and modify it with natural-language instructions
- **GUI interaction**: all operations can be done through mouse/shortcuts

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Interaction method | Plain-text terminal | Graphical IDE |
| Learning curve | Requires terminal experience | Zero-cost adoption for VS Code users |
| Remote development | Native SSH support | Requires Remote SSH extension |
| Code preview | Text diff output | Visual diff panel |
| Automation | Native pipelines/scripts | Requires extra configuration |
| Startup speed | Milliseconds | Seconds (Electron) |

---

## 2. Agent Capabilities

### Claude Code

```
User input
  -> System prompt + CLAUDE.md project instructions
  -> Claude LLM (streaming)
  -> Tool calls (Bash / Read / Edit / Write / Grep / Glob)
  -> Permission check (allow / ask / deny)
  -> Tool execution (possibly sandboxed)
  -> Results returned to LLM
  -> Repeat until complete
```

- **Agent tool**: spawns subagents through `Task` to process subtasks in parallel
- **Plan mode**: plan first, then execute; users can approve each step
- **Automatic memory**: learns project preferences across sessions and writes to CLAUDE.md
- **Toolchain**: Bash / Read / Edit / Write / Grep / Glob / Agent / WebFetch, etc.

### Cursor

```
User input (Chat / Composer / Cmd+K)
  -> Context collection (@-references + codebase index)
  -> LLM call (model chosen by user)
  -> Code generation / edit suggestions
  -> User review (visual diff)
  -> Apply changes
```

- **Composer**: multi-file editing agent that supports cross-file refactors
- **Background Agent**: autonomously runs tasks in a cloud sandbox (similar to Claude Code's headless mode)
- **Bug Finder**: automatically scans the codebase for potential issues
- **Inline editing**: use natural language to modify selected code directly with Cmd+K

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Agent autonomy | Highly autonomous; can execute dozens of consecutive steps | Composer is autonomous; Chat requires confirmation |
| Subagents | Parallel subagents spawned by the Task tool | Background Agent (cloud sandbox) |
| Tool calls | Direct system-command calls | Indirect calls through IDE APIs |
| File modifications | Writes directly to the filesystem | Editor buffers, requiring user confirmation |
| Test execution | Runs directly through Bash | Runs in the built-in terminal |
| Multitasking | Multiple terminal windows | Multiple Composer sessions |

---

## 3. Context Management

### Claude Code

- **1M-token context**: Opus 4.6 supports an ultra-long context window
- **Automatic compaction**: automatically compresses conversation history when context nears the limit
- **CLAUDE.md hierarchy**: project root -> subdirectories -> user level, loaded progressively
- **File reading**: reads file contents on demand, precisely controlling context usage
- **`--resume`**: restores previous session context

```
Context-source priority:
1. System prompt (built-in)
2. CLAUDE.md (project/directory/user level)
3. User input
4. Tool execution results (file contents, command output)
5. Automatic memory (cross-session learning)
```

### Cursor

- **@-reference system**: precise references such as `@file`, `@folder`, `@code`, `@web`, `@docs`
- **Codebase index**: automatically builds a semantic index for the whole project and supports fuzzy search
- **.cursorrules**: project-level instruction file similar to CLAUDE.md
- **Automatic context**: the editor automatically attaches the current file, selected code, and terminal output
- **Documentation index**: `@docs` can index external documentation as context

```
Context sources:
1. System prompt + .cursorrules
2. @-references (explicitly specified by the user)
3. Current file / selected code
4. Codebase index (semantic retrieval)
5. Terminal output / lint errors
```

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Context window | 1M tokens | Model-dependent, up to about 200k |
| Context acquisition | Active reading (on demand) | Passive indexing + @-references |
| Project understanding | Runtime dynamic exploration | Prebuilt index + semantic search |
| Compaction strategy | Automatic conversation compaction | Truncation / new session |
| Project instructions | CLAUDE.md (multi-level) | .cursorrules |
| Cross-session memory | Automatic memory + CLAUDE.md | No persistent memory |

---

## 4. Model Support

### Claude Code

- **Locked to Claude series**: Sonnet 4 / Opus 4.6 / Haiku
- **Model switching**: switch within a session using the `/model` command
- **Automatic routing**: simple tasks use Haiku; complex tasks use Opus
- **Third-party access**: non-Claude models are not supported

### Cursor

- **Multi-model support**:
  - Claude series (Sonnet / Opus)
  - GPT series (GPT-4o / o1 / o3)
  - Gemini series (2.5 Pro)
  - Custom models (via API key)
- **Model selector**: choose a different model for each conversation
- **Included quota**: subscriptions include a certain number of fast requests

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Model lock-in | Claude only | Multi-provider |
| Strongest model | Opus 4.6 (1M context) | Depends on user choice |
| Model switching | `/model` command | Dropdown menu |
| Custom models | Not supported | API key access supported |
| Billing model | Based on token usage | Fast requests included in subscription |

---

## 5. Extensibility

### Claude Code

```toml
# MCP server configuration (~/.claude/settings.json)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

- **MCP protocol**: standardized tool extension for GitHub / Jira / databases, etc.
- **Prompt Hooks**: PreToolUse / PostToolUse / Stop Hooks for custom workflows
- **Custom slash commands**: define project commands under `.claude/commands/`
- **13 official plugins**: GitHub / Linear / Sentry, etc.

### Cursor

```
Cursor extension system:
├── VS Code Extension Marketplace (fully compatible)
├── .cursorrules (project instructions)
├── MCP servers (tool extensions)
├── @-reference custom documentation
└── Custom AI rules
```

- **VS Code extensions**: inherits the entire VS Code extension ecosystem
- **MCP support**: also supports MCP protocol access to external tools
- **Rules system**: project-level / user-level rule files
- **Documentation index**: add custom documentation sources through `@docs`

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Plugin ecosystem | MCP + 13 official plugins | VS Code Marketplace + MCP |
| Extension scale | Limited but focused | Massive (VS Code ecosystem) |
| Workflow customization | Prompt Hooks (code-level) | Rules (declarative) |
| Project instructions | CLAUDE.md + custom commands | .cursorrules + Rules |
| Protocol standard | Native MCP support | MCP support + extension APIs |

---

## 6. Git Integration

### Claude Code

```bash
# Native Git operations
$ claude "Commit the current changes"
> Running git status...
> Analyzing changes...
> Generating commit message...
> Running git add + git commit
✓ Committed: fix: resolve auth token refresh race condition

# PR workflow
$ claude "Create a PR"
> Analyzing branch diff...
> Generating PR title and description...
> Running gh pr create
✓ PR #142 created
```

- **Direct git calls**: executes all git commands through the Bash tool
- **Intelligent commits**: automatically analyzes changes and generates commit messages
- **PR creation**: integrates with the `gh` CLI and automatically generates PR descriptions
- **Conflict resolution**: understands merge-conflict markers and resolves them automatically

### Cursor

- **GUI Git panel**: inherits VS Code's Source Control panel
- **AI Commit Message**: one-click commit-message generation
- **Visual diff**: graphical diff comparison
- **Branch management**: operate branches through the graphical interface

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Git operations | Direct command-line execution | GUI panel + command line |
| Commit generation | Deep analysis + automatic execution | One-click message generation |
| PR workflow | End-to-end automation | Requires manual operations |
| Conflict resolution | AI automatic resolution | Visual manual resolution |
| CI/CD integration | Native script integration | Requires extra configuration |

---

## 7. Teams and Enterprise

### Claude Code

```
Settings priority (5 levels):
1. System-level settings (Anthropic built-in)
2. Enterprise policy (administrator)
3. Workspace settings (project level)
4. User settings (personal)
5. Project CLAUDE.md
6. Directory-level CLAUDE.md
7. User-level CLAUDE.md
```

- **Teammates**: shared agent instances for team collaboration
- **Enterprise policies**: administrators control permissions, tool access, and model selection
- **Max subscription**: team-level usage management
- **SSO/SCIM**: enterprise identity-management integration

### Cursor

- **Business plan**: team management + centralized billing
- **Admin console**: member management and usage monitoring
- **Privacy Mode**: enterprise-grade privacy mode; code is not used for training
- **Centralized rules**: team-shared .cursorrules

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Settings hierarchy | 5-level priority system | Project level + user level |
| Team collaboration | Teammates shared agents | Business-plan team management |
| Management granularity | Fine-grained enterprise policies | Admin console |
| Identity management | SSO / SCIM | SSO |
| Audit logs | Session-level tracing | Usage statistics |

---

## 8. Security Model

### Claude Code

```
Three-level permission system:
┌─────────────────────────────────────┐
│  allow  — Automatically allow (allowlisted tools) │
│  ask    — Ask the user each time (default)        │
│  deny   — Deny execution (denylist)               │
└─────────────────────────────────────┘

Sandbox mechanism:
├── macOS: Seatbelt sandbox-exec
├── Linux: Docker container isolation
└── Network: allowed by default, configurable restrictions
```

- **Filesystem sandbox**: restricts the directories the agent can access
- **Command allowlist**: precisely controls executable shell commands
- **Prompt Hooks**: PreToolUse Hook can intercept dangerous operations
- **Network control**: can restrict the agent's network access scope

### Cursor

- **Privacy Mode**: when enabled, code is not stored on Cursor servers
- **SOC 2 certification**: enterprise-grade security compliance
- **Local processing**: code index is built locally
- **Code not used for training**: explicit commitment not to train models on user code

### Key Differences

| Dimension | Claude Code | Cursor |
|------|------------|--------|
| Execution sandbox | OS-level sandbox (Seatbelt/Docker) | IDE process isolation |
| Permission control | Three-level permissions + tool granularity | Privacy Mode switch |
| Data privacy | Local execution, conversations transmitted via API | Controlled by Privacy Mode |
| Network restrictions | Configurable network allowlist | No fine-grained network control |
| Code access | On-demand reading, user-controllable | Indexes the whole project |
| Compliance certification | Depends on Anthropic API policies | SOC 2 |

---

## 9. Pricing Comparison

### Claude Code

| Plan | Price | Description |
|------|------|------|
| **Direct API billing** | Per token | Sonnet ~$3/$15 per 1M tokens (input/output) |
| **Claude Pro** | $20/month | Includes limited Claude Code usage |
| **Claude Max** | $100-200/month | Large Claude Code usage allowance |
| **Enterprise API** | Negotiated usage-based | Volume discounts + SLA |

### Cursor

| Plan | Price | Description |
|------|------|------|
| **Hobby** | Free | 2,000 completions + 50 slow requests |
| **Pro** | $20/month | Unlimited completions + 500 fast requests |
| **Business** | $40/month/person | Team management + admin console |
| **Enterprise** | Custom | SSO + audit + custom deployment |

### Cost Analysis

| Usage scenario | Claude Code estimate | Cursor estimate |
|----------|------------------|-------------|
| Light use (10 conversations/day) | ~$20/month (Pro) | $0 (Hobby is enough) |
| Medium use (30 conversations/day) | ~$100/month (Max) | $20/month (Pro) |
| Heavy use (100+ conversations/day) | ~$200/month (Max) | $20/month (Pro, with slow requests) |
| Team (10 people) | ~$2000/month | $400/month (Business) |

---

## Selection Recommendations

| User profile | Recommended tool | Reason |
|----------|----------|------|
| **Terminal-heavy users** | Claude Code | Native terminal experience, pipeline integration, SSH-friendly |
| **VS Code users** | Cursor | Zero-cost migration and familiar IDE experience |
| **Full-stack solo developers** | Claude Code | End-to-end automation and strong agent autonomy |
| **Frontend developers** | Cursor | Visual previews and strong inline-completion experience |
| **DevOps / SRE** | Claude Code | Script integration and CI/CD automation |
| **Team collaboration (budget-sensitive)** | Cursor | Subscription pricing with predictable costs |
| **Large codebases** | Use both | Claude Code for deep analysis + Cursor for daily editing |
| **Remote server development** | Claude Code | Direct SSH access without a GUI |
| **Multi-language/multi-model needs** | Cursor | Flexible multi-model selection |
| **Security-sensitive environments** | Claude Code | Fine-grained permission control + OS-level sandbox |

---

## Conclusion

Claude Code and Cursor represent two different paths for AI-assisted programming:

**Claude Code's core strengths**:
- Terminal-native, deeply integrated with the Unix toolchain
- 1M-token ultra-long context, suitable for understanding large codebases
- Highly autonomous agent capabilities for end-to-end complex tasks
- Fine-grained security controls (sandbox + three-level permissions)
- 5-level settings system for flexible enterprise management

**Cursor's core strengths**:
- Visual GUI experience with low learning cost
- Multi-model support without lock-in to a single provider
- VS Code extension ecosystem with a huge plugin catalog
- Predictable subscription pricing with a free tier
- Smooth inline-completion experience for daily coding

**Best practice**: the two are not mutually exclusive. Many developers use a combined strategy: Cursor for daily coding and small edits, Claude Code for large refactors, code reviews, and automation tasks. The complementarity of terminal and IDE often unlocks the greatest value from AI coding tools.
