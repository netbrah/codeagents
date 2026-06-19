# 23. Recent Updates (v2.1.82 → v2.1.132)

> The main body of this series is based on **v2.1.81 binary reverse engineering** (2026-03-25). This document summarizes key incremental updates from 2026-03-26 through **2026-05-06 (v2.1.132)**, providing new-feature references for Code Agent developers.

> **Data sources**: [Claude Code official changelog](https://code.claude.com/docs/en/changelog) + [What's New](https://code.claude.com/docs/en/whats-new) + GitHub releases. Other chapters in this series have not deeply reverse-engineered v2.1.132, so the implementation details below may be incomplete.

## 1. Version Overview

| Scope | Period | Key Point |
|---|---|---|
| Documentation baseline | v2.1.81 (2026-03-25) | Main content of the series |
| Current latest | **v2.1.132** (2026-05-06) | Updates covered in this document |
| Release cadence | Biweekly feature drops (Week 13-17) | Released every Sunday |

## 2. Five Major Cloud-Connected Features (Most Important)

### 2.1 Computer Use in CLI (research preview)

**Released in Week 14 (2026-03-30 → 04-03)**.

| Dimension | Description |
|---|---|
| Trigger | Claude calls the `computer_use` tool inside the CLI |
| Capability | Open local GUI applications, click UI, and perform visual verification |
| Value | Connects "code changes" with "verification of actual runtime behavior"; adds a GUI-check loop beyond headless automation |
| Relationship to the Computer Use API | CLI-layer wrapper over the Anthropic Computer Use API |
| Risk | Requires explicit permission; Auto Mode defaults to strict approval |

**Takeaways for Code Agent developers**:
- Close the loop from LLM edits → run → observe UI verification (complementing the traditional read/edit/bash toolset)
- Use multimodal vision for visual regression testing
- Qwen Code has ❌ no equivalent capability, making this a potential differentiation point

### 2.2 Auto Mode (research preview)

**Released in Week 13 (2026-03-23 → 27)**.

```
Manual mode (default):
  Every permission_request → user approval

--dangerously-skip-permissions:
  Skip everything ⚠ dangerous

Auto Mode (new):
  Permission classifier model → automatically approve safe actions / block dangerous ones
  Sits between the two extremes above
```

**Approximate classifier behavior** (inferred):
- Input: current tool call + parameters + context
- Output: safe / risky / requires-confirmation
- Learning signal: user's historical approval choices

**Takeaways for Code Agent developers**:
- Smarter than a simple allowlist (context-aware, not just pattern matching)
- Complements Qwen Code daemon's [§05 permission flow 4 modes](../../comparison/qwen-code-daemon-design/05-permission-auth.md) — could become a fifth "auto-classified" mode

### 2.3 Ultraplan (early preview)

**Released in Week 15 (2026-04-06 → 10)**.

```
Local CLI drafts a plan
    ↓
Pushes it to Anthropic cloud web editor
    ↓
Non-authors (team members) review / comment
    ↓
Returns to CLI for execution (run locally or in the cloud)
```

**Key architectural points**:
- A Plan is not plain text; it is structured (steps / dependencies / artifacts)
- Cloud web editor and CLI synchronize bidirectionally
- Cloud environments are created automatically (on first use)

### 2.4 Ultrareview (public research preview)

**Released in Week 17 (2026-04-20 → 24)**.

```
Trigger: /ultrareview <PR#> or /ultrareview (current branch)
    ↓
Cloud fleet spawns multiple review agents in parallel (different perspectives)
    ↓
Aggregates findings → pushes them back to local CLI / Desktop
```

**Estimated fleet composition** (inferred from public behavior):
- security agent
- correctness agent
- style / convention agent
- performance agent
- test coverage agent

**Takeaways for Code Agent developers**:
- Parallel multi-agent review is a solution for scenarios where a single agent struggles
- Similar to Qwen Code [§subagent-display 4 kinds](../../comparison/subagent-display-deep-dive.md), but cloud-based
- The "Ultraplan / Ultrareview" naming pattern indicates a cloud-augmented CLI

### 2.5 Routines on Web (Week 16)

**Released in Week 16 (2026-04-13 → 17)**.

```
Web console:
  cron / GitHub event / API call
    ↓ schedule
  Routine (templated cloud agent)
    ↓ execute
  Results → back to web / CLI notifications
```

**New `/usage` command** (same week): view token burn / API call / compute usage.

## 3. Models and Reasoning (Week 16)

| Change | Description |
|---|---|
| **Default model** | Max / Team Premium default upgraded from Opus 4.6 to **Opus 4.7** |
| **New effort level** | `xhigh` (between `high` and `max`) |
| **`/effort` slider** | Interactive visual tuning instead of command-line args |

Older documentation ([§03-architecture](./03-architecture.md)) should update the default Opus 4.6 wording to 4.7. The pricing structure should remain similar.

## 4. CLI / Environment (v2.1.126-132)

### 4.1 Additions / Improvements

| Type | Item | Version | Description |
|---|---|---|---|
| ✨ | Native binaries released | Week 16 | Faster startup (previously Bun-packaged JS) |
| ✨ | `--plugin-url` flag | v2.1.129 | Fetch plugin packages from URLs, no longer local fs only |
| ✨ | `CLAUDE_CODE_SESSION_ID` env | v2.1.132 | Bash subprocesses can access session context |
| ✨ | `CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN` env | v2.1.132 | Do not switch to fullscreen rendering; preserve terminal scrollback |
| ✨ | `CLAUDE_CODE_FORCE_SYNC_OUTPUT` env | v2.1.129 | Force synchronous output for debugging |
| 🔧 | Gateway model discovery | v2.1.126 | `/model` picker reads model list from gateway `/v1/models` (opt-in: `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY`) |

### 4.2 OAuth / Authentication

| Improvement | Version | Description |
|---|---|---|
| MCP server OAuth: paste code | v2.1.126 | Manually paste code when browser callback is unreachable |
| Expanded scope for `--dangerously-skip-permissions` | v2.1.126 | Now also skips protected-path prompts |
| `--permission-mode` resume fix | v2.1.132 | Fixed bug where `--permission-mode` was ignored during `--resume` |

## 5. New Slash Commands (Week 13-17)

| Command | Period | Description |
|---|---|---|
| `/ultrareview` | Week 17 | Cloud fleet parallel review (see §2.4) |
| `/ultraplan` | Week 15 | Cloud-collaborative plan (see §2.3) |
| `/autofix-pr` | Week 15 | Trigger PR auto-fix from terminal (web equivalent) |
| `/usage` | Week 16 | View token / API / compute usage |
| `/team-onboarding` | Week 15 | Package the current setup into a replayable guide |
| `/theme` | Week 17 | Customize color themes (also supported from plugins) |
| `/loop` behavior change | Week 15 | Without an interval, becomes self-pacing instead of busy-looping |

The command list in [§02-commands](./02-commands.md) needs updating (currently 79 commands at the v2.1.81 baseline; add the items above).

## 6. Hooks System Enhancements

### 6.1 Conditional `if` Hooks (Week 13)

Previously, hooks could only run unconditionally. They now support:

```jsonc
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "if": "tool.args.command =~ /rm -rf/",
      "command": "echo 'Blocked dangerous rm -rf' && exit 2"
    }]
  }
}
```

**Significance**: Evolves from "all-or-nothing blocking" to "conditional blocking," reducing false positives.

[§12-hooks](./12-hooks.md) needs an explanation of the `if` field.

### 6.2 Monitor Tool (Week 15)

Background events flow into the conversation — logs / webhooks / file changes enter Claude context in real time.

```bash
claude monitor --logs /var/log/app.log --webhook https://...
# Claude automatically reacts when it sees log changes
```

Similar to Qwen Code [PR#3684 / PR#3791 monitor task kind](../../comparison/subagent-display-deep-dive.md); the two designs are converging.

## 7. IDE / Terminal

| Item | Description |
|---|---|
| ✨ Native VS Code extension | Highlight code blocks / invoke Claude inline / view modification suggestions directly in the editor |
| ✨ Tool Search / Lazy Loading | Tools activate on demand, reducing startup overhead (v2.1.76) |
| ✨ Flicker-free alt-screen | Improved `/tui fullscreen` stability (no blank screen after sleep/wake) |
| ✨ Custom themes | Provided by `/theme` or plugins |
| ✨ Session Recap | Summary of what happened while the terminal was unfocused |
| 🔧 Native PowerShell tool | Windows opt-in via `CLAUDE_CODE_USE_POWERSHELL_TOOL` |

## 8. MCP / Plugins

| Item | Description |
|---|---|
| 🔧 Per-tool MCP result-size override | Can set a single tool output limit up to **500K** |
| ✨ Plugin executables on Bash `PATH` | Plugins can inject binaries into the shell environment |
| 🔧 Improved MCP retry logic | Clearer connection-failure status (v2.1.132) |

[§14-mcp](./14-mcp.md) needs result-size override details.

## 9. Stability / Fixes

### 9.1 Key Bug Fixes

| Item | Version | Description |
|---|---|---|
| 🔧 Vim mode text corruption | v2.1.132 | NFD (decomposed) accented chars were corrupted by operators |
| 🔧 Pasted text starting with `/` swallowed | v2.1.132 | Pasted text beginning with `/` was silently dropped |
| 🔧 stdio MCP memory leak | v2.1.132 | RSS grew without bound in long-running sessions (10GB+) |
| 🔧 Bedrock / Vertex prompt caching | v2.1.132 | Fixed 400 errors |
| 🔧 Terminal emoji rendering | v2.1.132 | ZWJ sequences + Indic scripts |

### 9.2 ⚠ Only Breaking Change

**`Ctrl+O` behavior changed** (v2.1.110):
- Before: modal-toggle (switch between modes)
- After: toggle normal/verbose transcript view

Impact is small (rare shortcut), but user scripts that depend on the old behavior need adjustment.

## 10. Update Relationships with Other Chapters in This Series

The following chapters in this series need corresponding updates:

| Chapter | Update Point |
|---|---|
| [§01-overview](./01-overview.md) | Add Computer Use / Auto Mode / Ultraplan / Ultrareview to the capability matrix |
| [§02-commands](./02-commands.md) | Add 7 new slash commands (§5) |
| [§03-architecture](./03-architecture.md) | Default model Opus 4.6 → 4.7; new `xhigh` effort level; native binaries |
| [§06-settings](./06-settings.md) | New env vars (§4.1) |
| [§12-hooks](./12-hooks.md) | Conditional `if` hooks (§6.1) |
| [§14-mcp](./14-mcp.md) | Per-tool result-size override (§8) |
| [§15-telemetry](./15-telemetry-feature-flags.md) | New feature flags (gateway model discovery, etc.) |

## 10. Addendum: Session Fork Mode vs Qwen `/branch` (2026-05-08 Update)

Claude Code's session "branch off" is implemented via the **`--fork-session` CLI flag** (verified against v2.1.133 binary):

```bash
claude --resume --fork-session             # Resume + fork (choose session in picker)
claude --resume <session-id> --fork-session # Copy specified session
claude --continue --fork-session            # Continue previous session but fork
```

Error constraint: "`--session-id` can only be used with `--continue` or `--resume` if `--fork-session` is also specified."

**Qwen Code implementation from the same period** ([PR#3539](https://github.com/QwenLM/qwen-code/pull/3539) ✓ 2026-05-08 MERGED, +1538/-18):

| Dimension | Claude `--fork-session` | Qwen `/branch` (`/fork` alias) |
|---|---|---|
| Entry point | CLI flag (at startup) | Slash command (during runtime) |
| Self-described comparison | — | "Mirrors Claude Code's `/branch`" |
| Persistence (public) | Not public | Full JSONL copy + per-record `forkedFrom: {sessionId, messageUuid}` |
| Atomic creation | Not public | `fs.openSync 'wx' 0o600` (no TOCTOU) |
| Rollback safe | Not public | "core first, UI last" sequence: finalize → forkSession → loadSession → config.startNewSession → init → UI swap |
| Hook distinction | Not public | Separate `SessionStartSource.Branch` enum (does not reuse `Resume`) |
| Title collision | Not public | `(Branch N)` cap 99 → timestamp fallback |
| User experience | Startup flag | Instant during runtime (more convenient) |

**Comparative insight**: Claude uses a CLI flag model (one-time decision at startup), while Qwen uses a slash command model (instant fork during runtime). Both patterns have suitable scenarios. Qwen chose the more convenient slash command, but its engineering implementation quality is higher: atomic creation, rollback safety, and a dedicated hook enum are all semantic boundaries added by Qwen.

See [SubAgent §6.11](../../comparison/subagent-display-deep-dive.md#%E5%B7%B2%E8%90%BD%E5%9C%B0-11branch-alias-fork-session-%E5%88%86%E6%94%AFpr3539) for details.

## 11. Implications for Qwen Code Daemon Design

| New Claude Code Feature | Corresponding Position / Takeaway for Qwen daemon Design |
|---|---|
| Computer Use in CLI | ❌ Qwen has no equivalent; worth considering as differentiation |
| Auto Mode | [§05 permission flow](../../comparison/qwen-code-daemon-design/05-permission-auth.md) could add a fifth "auto-classified" mode |
| Ultraplan / Ultrareview (cloud fleet) | Same direction as [§06 §7 vs Anthropic Managed Agents](../../comparison/qwen-code-daemon-design/06-roadmap.md); External Reference Architecture can package similar products |
| Routines on Web | Application-layer scenario for clustered deployment in [§06 §5 External Reference multi-tenancy](../../comparison/qwen-code-daemon-design/06-roadmap.md) |
| Monitor tool (background events flow into conversation) | Qwen already has [PR#3684/3791 monitor task kind](../../comparison/subagent-display-deep-dive.md), indicating convergent design |
| Native binaries | Startup optimization direction; synergizes with [§06 §5 External Reference long-running stability](../../comparison/qwen-code-daemon-design/06-roadmap.md) in the Qwen daemon design |
| Conditional `if` hooks | Worth referencing when deciding whether Qwen Code hooks should introduce this mechanism |

## 12. One-Sentence Summary

**Across the ~6 weeks from v2.1.81 to v2.1.132, Claude Code focused on cloud collaboration (Computer Use / Auto Mode / Ultraplan / Ultrareview / Routines) + default model upgrade to Opus 4.7 + conditional Hooks + startup optimization (native binaries / tool lazy loading) + MCP enhancements (per-tool size override / plugin binaries on PATH) + many long-running bug fixes (stdio MCP leak 10GB+ / vim NFD / paste `/` swallow). The only breaking change is the Ctrl+O behavior change (minor impact). Design trend: CLI tools and cloud augmentation coexist (fleet of agents / cloud routines / web review) — sharing the same lineage as Anthropic Managed Agents while preserving the local CLI as the primary workspace.**

---

[← Back to README](./README.md)
