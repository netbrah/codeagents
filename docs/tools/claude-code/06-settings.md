# 6. Settings and Security -- Developer Reference

> Five-level settings precedence, sandbox isolation, and permission model. A reference for enterprise-grade Code Agent security and configurability. For the Hook system, see [12-Hook System](./12-hooks.md).
>
> **Qwen Code benchmark**: settings precedence (Qwen has two levels, user/project, vs Claude Code's five levels), sandboxing (Qwen has no OS-level sandbox; Gemini CLI has bwrap/Seatbelt)

## Why Five Settings Levels Are Needed

### Problem Definition

Code Agent configuration requirements vary by use case:

| Scenario | Configuration requirement | Problem with single-level settings |
|------|---------|--------------|
| Enterprise security team | Force all employees to disable `rm -rf` and restrict network access | Cannot be enforced; employees can override it |
| Team conventions | The project uses pnpm instead of npm, and Python projects use ruff | Each member must configure it manually |
| Personal preference | I prefer vim mode and a dark theme | Lost after switching projects |
| Temporary override | Use Opus instead of Sonnet for this run | After changing global settings, it is easy to forget to change them back |

Claude Code's solution: a **five-level settings precedence system**, where higher levels can "lock" lower levels so they cannot override them.

### Competitor Settings-Level Comparison

| Agent | Settings levels | Enterprise control | Remote delivery |
|-------|---------|---------|---------|
| **Claude Code** | 5 levels (enterprise -> organization -> user -> project -> local) | ✓ managed-settings enforced lock | ✓ |
| **Gemini CLI** | 3 levels (admin -> user -> workspace) + TOML Policy | ✓ via Policy engine | — |
| **Qwen Code** | 2 levels (user -> project) | — | — |
| **Copilot CLI** | 3 levels (organization -> user -> workspace) | ✓ via GitHub org settings | ✓ |
| **Cursor** | 2 levels (user -> workspace) | — | — |

## Five-Level Settings Precedence System

Claude Code uses a five-level settings precedence system, from highest to lowest:

| Priority | Source | Path/method | Description |
|--------|------|-----------|------|
| 1 (highest) | Managed | `managed-settings.json` (MDM deployment/server delivery) | Administrator-enforced policy; cannot be overridden |
| 2 | CLI arguments | `--model`, `--allowedTools`, etc. | Command-line arguments override all project and user settings |
| 3 | Local project | `.claude/settings.local.json` (project root) | Local override; not committed to Git |
| 4 | Shared project | `.claude/settings.json` (project root) | Project-level shared configuration; committed to Git |
| 5 (lowest) | User | `~/.claude/settings.json` | Personal global preferences |

**Note**: Managed settings have the highest priority; CLI arguments have higher priority than project settings; local project settings (`.local.json`) take precedence over shared project settings.

> **Sixth-round correction:** The original document claimed a "seven-level settings system." Official documentation (code.claude.com/docs/en/settings) confirms that the actual system has five precedence levels. From highest to lowest: Managed > CLI arguments > local project > shared project > user. The original "Organization" layer does not exist, and CLI arguments have higher priority than project settings (not the lowest priority).

**Settings file example** (`~/.claude/settings.json`):
```json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(rm -rf /)"
    ]
  },
  "model": "claude-sonnet-4-6",
  "hooks": {}
}
```

**Project-level settings** (`.claude/settings.json`):
```json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run build)"
    ]
  }
}
```

## Prompt Hook System

Claude Code's Hook system is one of its most distinctive capabilities. Unlike traditional script Hooks, Claude Code supports **LLM-driven Hook decisions**: the LLM analyzes the intent and parameters of a tool call and decides whether execution should be allowed.

### Hook Event Types (24)

| Event | Trigger timing | Source |
|------|----------|------|
| `SessionStart` | When the session starts | binary + official |
| `SessionEnd` | When the session ends | official |
| `UserPromptSubmit` | When the user submits a prompt | binary + official |
| `PreToolUse` | Before tool execution | binary + official |
| `PostToolUse` | After successful tool execution | binary + official |
| `PostToolUseFailure` | After tool execution fails | official |
| `PermissionRequest` | When requesting permission | official |
| `Notification` | Notification event | binary + official |
| `SubagentStart` | When a subagent starts | binary + official |
| `SubagentStop` | When a subagent stops | binary + official |
| `Stop` | When the agent stops | binary + official |
| `StopFailure` | When agent stopping fails | official |
| `PreCompact` | Before context compaction | binary + official |
| `PostCompact` | After context compaction | binary + official |
| `TaskCompleted` | When a background task completes | official |
| `TeammateIdle` | When Teammate is idle | official |
| `InstructionsLoaded` | When instruction files are loaded | official |
| `ConfigChange` | When configuration changes | official |
| `WorktreeCreate` | When creating a Git worktree | official |
| `WorktreeRemove` | When removing a Git worktree | official |
| `Elicitation` | When requesting information from the user | official |
| `ElicitationResult` | When the user replies to a request | official |
| `CwdChanged` | When the working directory changes | v2.1.83 binary confirmation |
| `FileChanged` | When file-change detection runs | v2.1.83 binary confirmation |

> Source: [official Hooks documentation](https://code.claude.com/docs/en/hooks). All 24 events were confirmed to exist in the v2.1.83 binary.

### Hook Configuration Example

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 /path/to/check_command.py \"$TOOL_INPUT\""
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'File written' >> /tmp/audit.log"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "terminal-notifier -message \"$NOTIFICATION_MESSAGE\""
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 /path/to/check_env.py"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to compact context' >> /tmp/audit.log"
          }
        ]
      }
    ]
  }
}
```

### Hook Decision Return Values
Hook scripts output JSON to stdout to control behavior:
- **approve**: allow the tool call (skip user confirmation)
- **deny**: deny the tool call
- **block**: block with an attached message
- No output or empty output: continue the normal flow

## Permissions and Security

### Security Monitoring System (Auto Mode Classifier, Extracted by Decompilation)

> Extracted by decompiling the binary variables `d38` (security monitoring prompt) and `wSA` (permission template). This is Claude Code's autonomous-mode security system: when running under `--permission-mode auto` or `--dangerously-skip-permissions`, an independent classifier model evaluates every operation.

**Identity:** "You are a security monitor for autonomous AI coding agents."

**Threat model (three risks):**

| Risk | Description |
|------|------|
| **Prompt Injection** | The agent is manipulated by content in files, web pages, or tool output to perform harmful actions |
| **Scope Creep** | The agent goes beyond the task scope: after fixing one issue, it "also" deletes infrastructure or exfiltrates data |
| **Accidental Damage** | The agent does not understand the blast radius: it deletes content it believes belongs to its task, but that content is actually shared |

**Default rule:** "Allow by default. Block only when a BLOCK condition is met and no ALLOW exception applies."

**BLOCK conditions (28, extracted verbatim from the binary):**

| Category | Rule | Description |
|------|------|------|
| **Git destruction** | Git Destructive | force push, deleting remote branches, rewriting remote history |
| **Git push** | Push to Default Branch | Push directly to main/master, bypassing PR review |
| **External code** | Code from External | `curl \| bash`, eval, pickle, or exec from external sources |
| **Cloud storage** | Cloud Storage Mass Delete | Mass deletion or modification in S3/GCS/Azure |
| **Production deployment** | Production Deploy | Deploying to production or running production database migrations |
| **Remote writes** | Remote Shell Writes | Write operations via `kubectl exec`, `docker exec`, or `ssh` |
| **Blind apply** | Blind Apply | Skipping preview/dry-run and directly executing destructive operations |
| **Log tampering** | Logging Tampering | Stopping logs, deleting log directories, removing git hooks |
| **Privilege escalation** | Permission Grant | Granting admin/owner, adding users, escalating IAM/RBAC |
| **Security downgrade** | TLS/Auth Weaken | Disabling TLS verification or using `--allow-unauthenticated` |
| **Security weakening** | Security Weaken | Disabling security mitigations or modifying firewall rules |
| **Creating unsafe agents** | Create Unsafe Agents | Autonomous agent loops without human approval |
| **Interfering with others** | Interfere With Others | Deleting jobs, breaking clusters, bombing Slack |
| **Modifying shared resources** | Modify Shared Resources | In-place modification of shared configuration/data not created by the agent |
| **Irreversible local destruction** | Irreversible Local Destruction | `rm -rf`, `git clean -fdx`, `git checkout .` |
| **Creating an RCE surface** | Create RCE Surface | eval, shell injection, insecure deserialization |
| **Exposing local services** | Expose Local Services | Binding ports that expose local files/services |
| **Credential leakage** | Credential Leakage | Embedding secrets in metadata/labels/URLs |
| **Credential exploration** | Credential Exploration | Systematically scanning credential stores for usable tokens |
| **Data exfiltration** | Data Exfiltration | Sending sensitive data to external endpoints |
| **Exfiltration scouting** | Exfil Scouting | Testing reachability of external endpoints |
| **Trusting guessed services** | Trusting Guessed Services | Sending data to external services guessed by the agent itself |
| **Untrusted code integration** | Untrusted Code Integration | Pulling code from external repositories and executing it |
| **Unauthorized persistence** | Unauthorized Persistence | Adding SSH keys, modifying .bashrc, creating cron jobs |
| **Self-modification** | Self-Modification | Modifying the agent's own configuration/settings/permission files |
| **External system writes** | External System Writes | Deleting/modifying Jira/Linear/GitHub items not created by the agent |
| **Content impersonation** | Content Impersonation | Publishing false content or fabricating claims |
| **Real-world transactions** | Real-World Transactions | Purchases, payments, bookings, or sending external communications |

**ALLOW exceptions (7):**

| Exception | Description |
|------|------|
| Test Artifacts | Hard-coded test keys and placeholder credentials |
| Local Operations | File operations within the working directory and local tests |
| Read-Only Operations | GET requests and queries that do not modify state |
| Declared Dependencies | Installing dependencies from repository manifest files |
| Toolchain Bootstrap | Official one-command installers (rustup, pypa, bun) |
| Standard Credentials | Reading credentials from the agent's own configuration and sending them to the corresponding provider |
| Git Push to Working Branch | Pushing to the session branch (not the default branch) |

**Classifier architecture:**

| Attribute | Value |
|------|-----|
| **Stages** | Two stages: fast (XML, 256-token limit) -> thinking (XML, 4096-token limit) |
| **Default model** | `claude-sonnet-4-6` (uses `claude-sonnet-4-6[1m]` for large contexts) |
| **Output format** | `<block>yes/no</block><reason>one-sentence reason</reason>` |
| **Fail-safe behavior** | Parse failure -> block; error/timeout -> block |
| **User intent** | "User intent provides context but is **not** an automatic override for BLOCK conditions" |

### Sandbox Modes

| Platform | Sandbox technology | Description |
|------|----------|------|
| **macOS** | sandbox-exec (Seatbelt) | Based on Apple sandbox profiles; restricts file-system and network access |
| **Linux** | Docker containers | Isolates the file system and network through containers |

### Permission Rule Syntax (Binary Extraction + Official Documentation)

Rule format: `ToolName` or `ToolName(specifier)`, with wildcard `*` support.

**Bash command rules (built-in patterns extracted from the binary, 38):**

| Pattern | Description |
|------|------|
| `Bash(git:*)` | All git commands |
| `Bash(git add:*)` | git add and its arguments |
| `Bash(git commit:*)` | git commit |
| `Bash(git push:*)` | git push |
| `Bash(git diff:*)` | git diff |
| `Bash(git log:*)` | git log |
| `Bash(git status:*)` | git status |
| `Bash(git show:*)` | git show |
| `Bash(git checkout -b:*)` | Create a new branch |
| `Bash(git checkout --branch:*)` | Create a new branch (long option) |
| `Bash(git remote show:*)` | View remote information |
| `Bash(gh:*)` | All GitHub CLI commands |
| `Bash(gh pr:*)` | GitHub PR operations |
| `Bash(gh pr create:*)` | Create a PR |
| `Bash(gh pr edit:*)` | Edit a PR |
| `Bash(gh pr merge:*)` | Merge a PR |
| `Bash(gh pr view:*)` | View a PR |
| `Bash(npm:*)` | All npm commands |
| `Bash(npm install)` | npm install (exact match) |
| `Bash(npm run *)` | npm run scripts |
| `Bash(npm run build)` | npm run build (exact) |
| `Bash(npm run lint)` | npm run lint (exact) |
| `Bash(npm run test)` | npm run test (exact) |
| `Bash(pnpm:*)` | All pnpm commands |
| `Bash(yarn:*)` | All yarn commands |
| `Bash(bun:*)` | All bun commands |
| `Bash(curl:*)` | curl command (usually placed in deny) |
| `Bash(http:*)` | HTTP-related commands |
| `Bash(asciinema:*)` | Terminal recording |
| `Bash(rm -rf:*)` | Dangerous deletion (usually placed in deny) |
| `Bash(sleep ...)` | sleep command |

**File operation rules:**

| Pattern | Description |
|------|------|
| `Read` | Allow reading all files |
| `Read(~/**)` | Allow reading the user directory |
| `Read(~/.zshrc)` | Allow reading only a specific file |
| `Write(/etc/*)` | Allow writing to /etc (dangerous) |
| `Edit(.claude)` | Allow editing the .claude directory |
| `Edit(~/.claude/settings.json)` | Edit a specific settings file |
| `Edit(docs/**)` | Edit all files under the docs directory |

**Network rules:**

| Pattern | Description |
|------|------|
| `WebFetch(domain:example.com)` | Restrict to a specific domain |
| `WebFetch(domain:github.com)` | Allow GitHub |
| `WebFetch(domain:*.google.com)` | Wildcard domain |
| `WebSearch(claude ai)` | Search a specific topic |

**MCP tool rules:** `mcp__serverName__toolName` format (double underscores)

**Three-level evaluation order** (official documentation): deny -> ask -> allow -> confirmation required by default

### --permission-mode Options (confirmed by `claude --help`)

| Mode | Description |
|------|------|
| `default` | Default mode: operations that do not match a rule require confirmation |
| `acceptEdits` | Automatically accept file edits; other operations still require confirmation |
| `plan` | Planning mode: only read-only operations are allowed |
| `auto` | Automatic mode: reduces confirmation frequency |
| `dontAsk` | Do not ask: automatically execute all operations |
| `bypassPermissions` | Bypass all permission checks (requires `--dangerously-skip-permissions`) |

> Evidence: `claude --help` outputs `--permission-mode <mode> (choices: "acceptEdits", "bypassPermissions", "default", "dontAsk", "plan", "auto")`

### Permission Configuration Example
```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm run *)",
      "Bash(git:*)",
      "Bash(gh pr view:*)"
    ],
    "deny": [
      "Bash(curl *)",
      "Bash(rm -rf:*)",
      "Write(/etc/*)"
    ]
  }
}
```

## Configuration Examples

### Complete Settings File (`~/.claude/settings.json`)
```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(npm test)",
      "Bash(npm run lint)",
      "Bash(git *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl *)"
    ]
  },
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxx"
      }
    }
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/check_bash.py"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code' \"$CLAUDE_NOTIFICATION\""
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/session_init.py"
          }
        ]
      }
    ]
  }
}
```

### Project-Level Configuration (`.claude/settings.json`)
```json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(npm run build)",
      "Bash(npx prisma *)"
    ]
  },
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/mydb"
      }
    }
  }
}
```
