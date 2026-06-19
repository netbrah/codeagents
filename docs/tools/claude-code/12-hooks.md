# 12. Hook System — Developer Reference

> The Hook system is one of the fastest-evolving subsystems in Claude Code. From the 5 events first introduced in 2025-06 to 27 events + 6 handler types by 2026-03, it underwent 9 months of continuous expansion. It addresses a fundamental problem for Code Agents: **how to give users deterministic control over Agent behavior instead of relying only on the LLM's "good intentions"**.
>
> **Qwen Code comparison**: Qwen Code has 12 event types (close to early Claude Code versions) and only the command handler. Claude Code's prompt/agent Hook types (LLM reasoning decisions) and hookify automatic rule generation are the main gaps.
>
> **v2.1.82 → v2.1.132 delta** (see [§23 §6.1](./23-recent-updates.md) for details):
> - **Conditional `if` Hooks** (Week 13, late 2026-03): Hook configuration added an `if` field for conditional logic, evolving from "all-or-nothing blocking" to "conditional blocking"
>
> ```jsonc
> {
>   "hooks": {
>     "PreToolUse": [{
>       "matcher": "Bash",
>       "if": "tool.args.command =~ /rm -rf/",
>       "command": "echo 'Blocked dangerous rm -rf' && exit 2"
>     }]
>   }
> }
> ```
>
> The main body of the 27 events / 6 handlers below remains unchanged. Conditional `if` is an **orthogonal enhancement** to them—any hook can be combined with an `if` condition. This reduces false positives.

## I. Why a Hook System Is Needed

### 1.1 Problem Definition: Uncertainty in LLM Instructions

If you write "do not modify .env files" in CLAUDE.md / QWEN.md, the model will follow it **most of the time**. But "most of the time" is not enough in production. The core problem solved by Hooks is:

| Scenario | Relying only on LLM instructions | With Hook |
|------|-------------|---------|
| "Do not delete production.config" | The model may forget during a complex reasoning chain | A `PreToolUse` Hook checks the path before Write/Edit and **blocks deterministically** |
| "Run prettier after every edit" | The model may forget or decide "it is not needed" | A `PostToolUse` Hook **always runs** after Write/Edit |
| "Tests must pass before submission" | The model may skip them | A `Stop` Hook **forces** `npm test` before finishing |
| "Log all API calls to the audit log" | The model is completely unaware of this requirement | A `PostToolUse` Hook runs silently and **uses no tokens** |

Claude Code's official documentation states:

> *"Hooks provide **deterministic control** over Claude Code's behavior, ensuring certain actions **always happen** rather than relying on the LLM to choose to run them."*

### 1.2 Design Principles

| Principle | Meaning | Implementation |
|------|------|------|
| **Determinism > probability** | Hooks are guaranteed to run at specific lifecycle points and do not depend on LLM judgment | Event-driven + matcher triggers |
| **Defense in depth** | PreToolUse Hooks trigger before permission checks, and deny decisions cannot be overridden by permission mode | Hooks take precedence over permission mode |
| **Composable** | When multiple Hooks match the same event, they all run in parallel and the strictest decision applies | deny > ask > allow |
| **Progressive complexity** | Simple scenarios only need exit codes (`0`=allow, `2`=block); advanced scenarios use structured JSON | Layered API |
| **Context-free execution** | Hooks run outside the LLM conversation and do not consume tokens | stdin/stdout pipes |

### 1.3 Evolution Timeline

| Version | Time | New events | Milestone |
|------|------|---------|--------|
| v1.0.38 | 2025-06 | PreToolUse, PostToolUse, Stop, Notification, SessionStart | First introduction, [Issue #712](https://github.com/anthropics/claude-code/issues/712) |
| v2.1.0 | Early 2026 | — | Hooks can be defined in agents/skills frontmatter |
| v2.1.49 | 2026-02 | ConfigChange, WorktreeCreate, WorktreeRemove | Configuration and worktree lifecycle |
| v2.1.76 | 2026-03-14 | Elicitation, ElicitationResult, PostCompact | MCP interaction + after compaction |
| v2.1.78 | 2026-03-17 | StopFailure | Stop Hook failure recovery |
| v2.1.83 | 2026-03-25 | CwdChanged, FileChanged, PermissionDenied | File monitoring + permission denial |
| v2.1.84 | 2026-03-26 | TaskCreated | Task system integration |
| v2.1.85 | 2026-03-26 | — | `if` conditional filtering field |
| Current | 2026-04 | 27 total | 6 handler types |

> Note: Before the official introduction, the community developed [cc-hook](https://github.com/nahco314/cc-hook) as an unofficial alternative. The original requirements were inspired by React component lifecycle Hooks and Google ADK callbacks.

## II. Detailed Breakdown of the 27 Events

### Tool Execution (5 types) — **Most commonly used**

| Event | Trigger timing | Intervenable behavior | Typical use cases |
|------|---------|-----------|---------|
| `PreToolUse` | Before tool execution | Block, modify input, change permissions | Block dangerous commands, protect files |
| `PostToolUse` | After successful tool execution | Replace output, inject context | Automatic formatting, audit logs |
| `PostToolUseFailure` | After tool execution fails | Inject error context | Error analysis, automatic retry hints |
| `PermissionRequest` | When the permission system requests approval | allow/deny/ask | Automatically approve safe operations |
| `PermissionDenied` | After permission is denied | Request retry | Prompt the user to adjust permissions |

### Session + Agent Lifecycle (7 types)

| Event | Trigger timing | Typical use cases |
|------|---------|---------|
| `SessionStart` | Session start | Inject project context, set file monitoring paths |
| `SessionEnd` | Session end | Clean up temporary files, automatic commits |
| `Setup` | Initialization | Environment configuration |
| `Stop` | Agent execution stops | Run tests, generate summaries |
| `StopFailure` | Stop Hook failure | Error recovery |
| `SubagentStart` | Before Subagent starts | Permission checks, context injection |
| `SubagentStop` | After Subagent stops | Result aggregation |

### Context + Tasks + Files + Worktree + MCP + User (15 types)

| Category | Event | Trigger timing |
|------|------|---------|
| Compaction | `PreCompact` / `PostCompact` | Before/after context compaction |
| Tasks | `TaskCreated` / `TaskCompleted` / `TeammateIdle` | Task created/completed/teammate idle |
| Files | `FileChanged` / `CwdChanged` / `ConfigChange` / `InstructionsLoaded` | File/directory/configuration/instruction changes |
| Worktree | `WorktreeCreate` / `WorktreeRemove` | Git worktree creation/removal |
| User | `UserPromptSubmit` / `Notification` | User input/notification |
| MCP | `Elicitation` / `ElicitationResult` | MCP elicitation request/result |

## III. 6 Handler Types

### 3.1 Command Hook (shell command) — Basic

Receives event JSON via stdin, and the exit code controls behavior. It is the simplest and most commonly used type.

```json
{
  "type": "command",
  "command": "bash /path/to/check.sh",
  "timeout": 30,
  "if": "Bash(git *)"
}
```

**Exit Code**: `0`=allow, `2`=block, others=non-blocking error.

### 3.2 Prompt Hook (LLM reasoning decision) — Claude Code's unique innovation

Uses **another LLM** (Haiku by default, low-cost and fast) to determine whether an operation is safe. It solves **semantic-level judgment** problems that static rules cannot handle.

```json
{
  "type": "prompt",
  "prompt": "A shell command will execute: $ARGUMENTS. Is it safe? Consider: destructive operations, credential exposure, network access. Reply {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
  "model": "claude-haiku-4-5"
}
```

**Why is this needed?**

Static rules (command hooks) can block `rm -rf`, but cannot distinguish between:
- `rm -rf node_modules` (safe—cleaning dependencies)
- `rm -rf /` (disastrous—deleting the system)
- `rm -rf build/` (possibly safe—cleaning build artifacts, depending on context)

Prompt Hooks let an LLM make semantic-level judgments based on context—something command-line rule engines cannot do.

### 3.3 Agent Hook (full Agent validation)

Creates a temporary Agent that can read the transcript and use tools for **multi-turn deep validation**. It supports up to 50 interaction turns. This is suitable for complex scenarios, such as verifying whether an architectural change breaks module boundaries.

### 3.4 HTTP Hook (external Webhook)

POSTs JSON to an external HTTP endpoint. Supports SSRF protection (private IP blocking), environment variable isolation (explicit `allowedEnvVars` whitelist), and header injection protection (CRLF/NUL filtering).

### 3.5 Callback / Function Hook (internal runtime)

TypeScript functions are registered directly, with no subprocess overhead. Callback is used for system-level Hooks (file tracking, attribution), while Function is used for session-level temporary validation (such as structured output validation). These cannot be persisted to settings.json.

## IV. Hook Execution Architecture

### 4.1 Execution Flow

```
Event triggered
  │
  ├─ 1. Matching: getMatchingHooks()
  │     ├─ Simple string exact match
  │     ├─ Pipe-separated multiple values ("Write|Edit")
  │     ├─ Regex matching
  │     └─ * = match all
  │
  ├─ 2. if conditional filtering (permission rule syntax)
  │     └─ "Bash(git *)" — triggers only for git commands
  │     └─ Runs before spawning the subprocess to save overhead
  │
  ├─ 3. Cross-source deduplication (user/project/local/plugin)
  │
  ├─ 4. Execute all matching Hooks in parallel
  │     └─ Each Hook has an independent timeout
  │
  └─ 5. Result aggregation
        └─ Permission priority: deny > ask > allow > passthrough
```

### 4.2 Key Design: Hooks Take Precedence over Permissions

```
PreToolUse Hook
  │ deny
  ▼
[Blocked] ← Cannot be overridden even if permission mode = bypassPermissions

PreToolUse Hook
  │ allow
  ▼
Permission Mode check ← A Hook can allow, but the permission system still applies
```

**Meaning**: Hooks can strengthen restrictions (deny), but they cannot bypass the permission system. This is the core of "defense in depth"—Hooks are an additional security layer, not a replacement.

### 4.3 Input/Output Schema

**Common input** (received by all Hooks):
```json
{
  "session_id": "...",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {"command": "rm -rf node_modules"}
}
```

**Output capabilities**:

| Output field | Effect | Applicable events |
|---------|------|---------|
| `continue: false` | Blocks subsequent execution | All |
| `decision: 'block'` | Blocks tool execution | PreToolUse |
| `permissionDecision: 'allow'/'deny'/'ask'` | Changes permission behavior | PreToolUse, PermissionRequest |
| `updatedInput: {...}` | Modifies tool input | PreToolUse |
| `updatedMCPToolOutput` | Replaces tool output | PostToolUse |
| `additionalContext: "..."` | Injects a system message for the model | All |
| `initialUserMessage: "..."` | Sets the first user message | SessionStart |
| `watchPaths: [...]` | Monitors file changes | SessionStart, CwdChanged |
| `retry: true` | Retries after permission denial | PermissionDenied |

## V. Comparison with Competitor Hook Systems

### 5.1 Design Philosophy Differences

| Agent | Hook design philosophy | Core characteristics |
|-------|-------------|---------|
| **Claude Code** | Maximum coverage + LLM decisions | 27 events, 6 handler types, unique prompt/agent types |
| **Gemini CLI** | Type safety + model-level interception | 11 events, BeforeModel/AfterModel (model-layer interception), runtime Handler |
| **Qwen Code** | MessageBus integration | 12 events, permission decision framework, deep integration with CoreToolScheduler |
| **Cursor** | Minimum necessary | ~5 events (beforeShellExecution/afterFileEdit/stop, etc.) |
| **Copilot CLI** | Plugin ecosystem | 6 events, plugins can package hooks + agents + skills + MCP |
| **Aider** | No Hook | Relies on Git integration and configuration flags; the community fork AiderDesk has 30+ events |

### 5.2 Capability Matrix

| Capability | Claude Code | Gemini CLI | Qwen Code | Copilot CLI | Cursor |
|------|-------------|-----------|-----------|------------|--------|
| Event count | **27** | 11 | 12 | 6 | ~5 |
| Handler types | **6** (including LLM) | 2 | 1 | 1 | 1 |
| LLM-driven decisions | ✓ prompt + agent | — | — | — | — |
| Tool input modification | ✓ | ✓ | ✓ | — | ✓ |
| Tool output replacement | ✓ | ✓ (tail call) | ✓ | — | — |
| Model-layer interception | — | ✓ BeforeModel | — | — | — |
| Tool selection modification | — | ✓ BeforeToolSelection | — | — | — |
| HTTP Webhook | ✓ | — | — | — | — |
| File monitoring | ✓ FileChanged | — | — | — | — |
| Conditional filtering (if) | ✓ | matcher | matcher | — | matcher |
| Async background | ✓ async/asyncRewake | — | — | — | — |
| Synthetic LLM response | — | ✓ getSyntheticResponse | — | — | — |

### 5.3 Unique Capability Analysis

**Unique to Claude Code**:
- `prompt` / `agent` Hook types: LLM semantic-level judgment
- `FileChanged`: reactive Hooks driven by file monitoring
- `hookify`: automatically generates Hook rules from conversations
- HTTP Hook: integration with external systems

**Unique to Gemini CLI**:
- `BeforeModel` / `AfterModel`: intercepts at the LLM call layer (can inject synthetic responses)
- `BeforeToolSelection`: modifies tool selection (dynamically adjusts the available tool set)
- `runtime` handler: native TypeScript functions with zero serialization overhead

**Unique to Qwen Code**:
- MessageBus asynchronous protocol: deep integration with CoreToolScheduler
- Permission decision framework: `getPermissionDecision()` / `getPermissionDecisionReason()`

### 5.4 Security Model

| Agent | Trust model | Description |
|-------|---------|------|
| Claude Code | **Explicit trust** | All Hooks require the user to accept a workspace trust dialog |
| Gemini CLI | Implicit trust | Project Hooks are blocked in untrusted directories |
| Qwen Code | Implicit trust | Hooks execute through MessageBus |

Claude Code's explicit trust dialog is a key defense against **malicious repositories executing arbitrary commands through `.claude/settings.json`**.

## VI. Practical Use Cases

### Scenario 1: Protect critical files

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "echo $TOOL_INPUT | jq -r '.file_path // .path' | grep -qE '\\.(env|pem|key)$' && exit 2 || exit 0"
      }]
    }]
  }
}
```

### Scenario 2: LLM safety review (prompt Hook)

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "prompt",
        "prompt": "Shell command: $ARGUMENTS. Is it safe? Consider: destructive ops, credential exposure, network access, production impact. Reply {\"ok\": true} or {\"ok\": false, \"reason\": \"...\"}",
        "model": "claude-haiku-4-5"
      }]
    }]
  }
}
```

### Scenario 3: Automatically format after edits

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "npx prettier --write \"$(echo $TOOL_INPUT | jq -r '.file_path // .path')\"",
        "if": "Write(*.ts)|Write(*.tsx)|Edit(*.ts)|Edit(*.tsx)"
      }]
    }]
  }
}
```

### Scenario 4: Force tests before finishing

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "npm test 2>&1 | tail -20; exit $?",
        "statusMessage": "Running tests..."
      }]
    }]
  }
}
```

### Scenario 5: Re-inject critical context after compaction

```json
{
  "hooks": {
    "PostCompact": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "echo '{\"hookSpecificOutput\":{\"hookEventName\":\"PostCompact\",\"additionalContext\":\"REMINDER: This project uses pnpm, not npm. Always use pnpm commands.\"}}'"
      }]
    }]
  }
}
```

## VII. hookify: Automatically Generating Rules from Conversations

This is the most innovative design in Claude Code's Hook system—**turning user frustration into persistent behavioral constraints**.

### 7.1 Problem

Users repeatedly correct the same Agent mistakes: "do not use npm; we use pnpm", "do not modify .env", "stop deleting tests". Each correction consumes tokens but leaves no trace, so the next session repeats the mistake.

### 7.2 Solution

```bash
/hookify                          # Analyze the conversation and automatically identify behaviors that need constraints
/hookify forbid using rm -rf commands     # Specify the behavior to block
/hookify list                     # View existing rules
/hookify configure                # Enable/disable rules
```

### 7.3 How It Works

1. Analyze the most recent 10-15 user messages and use NLP to identify frustration signals ("do not do this", "stop", "why again...")
2. Identify the Agent behavior patterns that caused the frustration
3. Generate a `.claude/hookify.{rule-name}.local.md` rule file
4. The rule file uses YAML Frontmatter to define event types, regex matching, and display messages
5. **Takes effect immediately**—no restart required

### 7.4 Developer Takeaways

hookify demonstrates an automated path from "**conversation → rules**": users' natural-language corrections are structured into persistent Hook rules. This is much friendlier than asking users to manually edit settings.json and write Hook configuration. Qwen Code could implement a similar mechanism: analyze users' negative expressions about Agent behavior and automatically generate `.qwen/hooks/` rules.

## VIII. Recommendations for Improving the Qwen Code Hook System

### P1: Expand Event Types

Qwen Code currently has 12 event types, close to early Claude Code versions (v1.0.38). Recommended priorities:

| New event | Value | Difficulty |
|---------|------|------|
| `PostCompact` | Re-inject critical context after compaction | Low |
| `FileChanged` | Reactive Hooks driven by file monitoring | Medium |
| `TaskCreated` / `TaskCompleted` | Multi-Agent task tracking | Low |
| `StopFailure` | Stop Hook failure recovery | Low |
| `PermissionDenied` | Guidance after permission denial | Low |

### P1: if Conditional Filtering

Currently, all Hooks that match a matcher are triggered. Add an `if` field that supports permission rule syntax (for example, `"Bash(git *)"` triggers only for git commands) to avoid unnecessary Hook execution overhead.

### P2: prompt-Type Hook

Claude Code's core innovation. Use an LLM for semantic-level safety judgments, going beyond the capability boundaries of static rules. Implementation points:
- Use a small model by default (such as qwen3.5-flash) to reduce cost and latency
- Inject event data through the `$ARGUMENTS` placeholder
- Return `{"ok": true/false, "reason": "..."}`

### P2: HTTP Hook

Support POSTing to external webhooks, opening the door to CI/CD integration, audit logs, Slack notifications, and similar scenarios. SSRF protection is required.

### P3: hookify Automatic Rule Generation

Analyze users' negative expressions about the Agent and automatically generate Hook rules. In the long term, this is key to improving user experience—from "manual configuration" to "conversational learning".
