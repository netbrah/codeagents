# Claude Code Async Tasks Deep Dive — Background Shell + Monitor

> **Core question**: What are `1 shell, 1 monitor` in the status bar? Why is Claude Code the only tool that makes "agent async tasks" a first-class capability?
>
> **Conclusion first**: Claude Code v2.1.120 productizes "the agent is still working in the background" as a three-part system: **two kinds of counters + unified management UI + notification mechanism**. Qwen Code v0.16.0 has completed the 5-PR stack (PR#3076 + PR#3471 + PR#3488 + PR#3642 + PR#3684 all merged), covering subagent control, the Bash bg pool, and the Monitor tool, approaching Claude Code's full design. OpenCode still has no equivalent system.

## 1. `1 shell, 1 monitor` in the status bar

A real session running background bash + Monitor at the same time:

```
✻ Sautéed for 37s · 1 shell, 1 monitor still running          ← per-turn status line
──────────────────────────────────────────────────────────────
❯
──────────────────────────────────────────────────────────────
  /tmp/cc-bg-test
  ⏵⏵ auto mode on · 1 shell, 1 monitor                         ← Footer second line
```

Meaning of the two numbers:

| Counter | Source | When it appears |
|---|---|---|
| `N shell` | Number of background Bash processes | Any `Bash(..., run_in_background: true)` |
| `N monitor` | Number of active Monitor tool instances | LLM calls the `Monitor` tool |

Real capture file: [`screenshots/claude-code-bg-tasks-90x35.txt`](./screenshots/claude-code-bg-tasks-90x35.txt)

## 2. Differences between the two kinds of background tasks

### Shell (background Bash)

Comes from the `Bash` tool + `run_in_background: true`; fundamentally it is **subprocess hosting**. Common scenarios:

- Dev servers: `npm run dev` / `vite` / `webpack --watch`
- File watching: `tail -f` / `inotifywait -m`
- Long tasks: `pytest` / `cargo build`

Observed invocation:

```
● Bash(tail -f /tmp/.../watched.log)
  ⎿ Running in the background (↓ to manage)

● Started tail -f /tmp/.../watched.log in the background (ID: b64zw7iij). 
  It will keep running and capture any lines appended to the file.
```

Characteristics:
- **One completion notification**: the agent receives a `<task-notification>` with `status: completed` (including exit code), but the notification **does not include stdout contents**
- **stdout written to disk**: complete output is written to a task output file under the session temporary directory (the agent reads that path with the `Read` tool)
- **5 GB disk limit** (`MAX_TASK_OUTPUT_BYTES`); exceeding it triggers SIGKILL
- **Termination methods**: natural exit / user presses `x` / agent calls `TaskStop(shell_id=...)` (including the `KillShell` alias)

### Monitor (event-stream listener)

Comes from the `Monitor` tool; fundamentally it is **long-running listening + push notifications**. Common scenarios:

- Watch log errors: `tail -f log | grep ERROR`
- Watch file changes: `inotifywait -m --format '%e %f' /watched/dir`
- Poll a remote endpoint: `while true; do gh api ...; sleep 30; done`
- WebSocket listener

Observed invocation:

```
● Monitor(ERROR lines in /tmp/.../watched.log)
  ⎿ Monitor started · task bd4lm9aqb · persistent

● Monitor armed (ID: bd4lm9aqb) — tail -F piped through grep --line-buffered "ERROR" 
  on /tmp/.../watched.log. Each ERROR line that appears will arrive as a notification.
```

Characteristics:
- **Notification per event**: the **content of each stdout line goes directly into `<task-notification>`** (the agent does not need to read another file)
- **Multiple lines within 200ms are batched into one notification** ("multiline output from a single event groups naturally")
- **Lifecycle**: default `timeout_ms: 300000` (5 minutes) / `persistent: true` (session lifetime)
- **Overload protection**: automatically stops when too many messages are produced (avoids token explosion)
- **Exit conditions**: script exits / timeout / `TaskStop(task_id=...)`

### Differences at a glance

| Dimension | Shell (Bash bg) | Monitor |
|---|---|---|
| Trigger | `Bash` + `run_in_background: true` | `Monitor` tool |
| Notification count | **1** (on completion, includes exit code) | **N** (one per event, 200ms batching) |
| stdout contents | Written to task output file; agent reads with `Read` | Directly inside notification |
| Default lifecycle | Process exits naturally | 5-minute timeout |
| Push limit | Only one completion event | Automatic throttling + self-stop on overload |
| Status-bar counter | `N shell` | `N monitor` |
| Termination | `TaskStop(shell_id=...)` or `KillShell` alias | `TaskStop(task_id=...)` |

**Key insight**: both are **push-mode**, but their cardinality differs. Shell sends "one notification on completion" (good for build/test), while Monitor sends "one notification per event" (good for watch/log listening). This matches the decision tree in the Monitor tool description:

> - **One** ("tell me when the server is ready / the build finishes") → use **Bash with `run_in_background`**
> - **One per occurrence, indefinitely** → Monitor with an unbounded command

## 3. Status-bar display mechanism

Footer two-line layout (v2.1.120):

```
  /tmp/cc-bg-test                              ← Line 1: cwd
  ⏵⏵ auto mode on · 1 shell, 1 monitor         ← Line 2: mode + async task counters
```

`auto mode on` is always present (unless the user switches modes); the later `· 1 shell, 1 monitor` is **appended on demand**:

| Trigger condition | Display | Observed? |
|---|---|---|
| No background tasks | `⏵⏵ auto mode on (shift+tab to cycle)` | ✅ |
| Only 1 bg shell | `⏵⏵ auto mode on · 1 shell` | ✅ |
| Both | `⏵⏵ auto mode on · 1 shell, 1 monitor` | ✅ |
| Only 1 monitor | `⏵⏵ auto mode on · 1 monitor` | ⚠️ Inferred (pure monitor not tested separately) |
| Multiple | `⏵⏵ auto mode on · 3 shells, 2 monitors` | ⚠️ Inferred (plural form references the "2 active shells" UI title) |

After each turn completes there is also an **inline status line**:

```
✻ Cooked for 7s · 1 shell still running
✻ Sautéed for 37s · 1 shell, 1 monitor still running
```

This is the timing + background-task reminder shown after replying to the user: **right after the user reads the LLM response, they are reminded that N tasks are still running in the background**. This "double prompt" prevents the state from being ignored.

## 4. Trigger mechanisms

### 1. The LLM explicitly sets `run_in_background: true`

The most common case: the LLM decides "this is a long task" and proactively backgrounds it:

```
Bash({
  command: "npm run dev",
  run_in_background: true
})
```

This applies to obvious long-running tasks (dev server / watch / sleep).

### 2. The command is in `COMMON_BACKGROUND_COMMANDS` and times out

For common long commands such as `npm` / `node` / `python` / `cargo` / `make` / `docker` / `webpack` / `vite` / `jest` / `pytest`, **timeout automatically moves them to the background** to avoid occupying the agent main thread.

Source reference: [04-tools.md#643-655](../tools/claude-code/04-tools.md)

### 3. Kairos mode backgrounds automatically after 15 seconds

`ASSISTANT_BLOCKING_BUDGET_MS = 15_000` — **in assistant mode, any foreground bash that runs longer than 15 seconds is automatically moved to the background**. The LLM has a strict 15-second "foreground wait budget"; after timeout, the agent is freed.

```
Start Bash → progress appears after 2 seconds (PROGRESS_THRESHOLD_MS)
           ↓
       automatically background after 15 seconds (Kairos mode)
           ↓
       removed from foreground, enters shell pool, status bar +1 shell
```

### 4. User manually backgrounds with `Ctrl+B`

When foreground bash runs too long and the user wants to do something else, pressing Ctrl+B immediately backgrounds it, and the main prompt can receive the next prompt.

## 5. Management UI (`/tasks` / `/bashes` / `↓` key)

### List view

Press `↓` or enter `/tasks` (alias `/bashes`):

```
  Background tasks
  2 active shells

  ❯ ERROR lines in /tmp/.../watched.log (running)        ← Monitor (highlighted)
    tail -f /tmp/.../watched.log (running)                ← Shell

  ↑/↓ to select · Enter to view · x to stop · ←/Esc to close
```

The UI title uses `2 active shells` to refer to both kinds **collectively as shells** (the naming carries over: internally Monitor is a wrapped shell pipeline, as shown by the Script field in the Monitor details below), but the Footer status bar **classifies them precisely** as `1 shell, 1 monitor`; the Footer should be treated as authoritative.

### Shell details

Select a bash bg item and press Enter:

```
  Shell details

  Status:   running
  Runtime:  32s
  Command:  sleep 60

  Output:
  No output available

  ← to go back · Esc/Enter/Space to close · x to stop
```

Shows runtime + command string + accumulated stdout output (truncated).

### Monitor details

Select a Monitor item:

```
  Monitor details

  Status:   running
  Runtime:  1m 27s
  Script:   tail -F /tmp/.../watched.log | grep --line-buffered "ERROR"

  Output:
  No output available
```

Shows the full monitor script. Internally the Monitor tool wraps `tail -f log | grep ERROR` into a complete bash command before execution, so the Script field is the **final expanded shell script**.

## 6. Monitor notification mechanism (agent perspective)

Monitor's unique design is "each stdout line = one chat notification." From the agent's perspective:

```
Monitor starts → agent continues doing other work → Monitor detects an ERROR line
                                          ↓
                  Inject <system-reminder><task-notification>:
                  script stdout line content goes directly into the notification body
                                          ↓
                  agent sees notification + content in the next turn and decides whether to respond
```

**Difference from Bash bg**: Bash bg completion notifications **carry only status/exit code**. Full stdout is written to a `tasks/<id>.output` file, and the agent must call `Read` again to see it. Monitor puts the content directly into the notification: zero extra reads.

Implementation details (from the Monitor tool description):

> Stdout lines within 200ms are batched into a single notification, so multiline output from a single event groups naturally.
>
> Monitors that produce too many events are automatically stopped; restart with a tighter filter if this happens.

Engineering details (also from the tool description):
- Use `grep --line-buffered` to force line buffering (otherwise pipe buffering can delay output by minutes)
- Failure tolerance: `curl ... || true` prevents one failed request from stopping the whole monitor
- Recommended polling interval: local 0.5-1s / remote 30s+ (API rate limits)

## 7. Relationship to foreground/background Bash

```
Bash tool call
    │
    ├─ run_in_background=true explicitly
    │    or
    │   command belongs to COMMON_BACKGROUND_COMMANDS and times out
    │    or
    │   Kairos 15s automatic backgrounding
    │    or
    │   user Ctrl+B
    │       ↓
    │   enters shell pool
    │   Footer displays +1 shell
    │   receives 1 notification on completion (including exit code)
    │   stdout lands in the session temp directory and is read with Read
    │   terminated via TaskStop(shell_id=id)
    │
    └─ default foreground
        ↓
       waits synchronously for exit
       shows a progress bar after PROGRESS_THRESHOLD_MS=2s
```

```
Monitor tool call
    │
    └─ always background + push-style
        ↓
       enters monitor pool
       Footer displays +1 monitor
       pushes every stdout line as a notification to the agent
       terminated via TaskStop(task_id=id)
```

The two pools share the `/tasks` management UI, but the status-bar counters are classified separately.

## 8. Corresponding capabilities in other agents

| Capability | Claude Code v2.1.120 | Qwen Code v0.16.0 | OpenCode v1.14.24 |
|---|---|---|---|
| Background Bash process | ✅ `run_in_background` + automatic backgrounding + Ctrl+B + shell pool + output written to disk | ✅ **Completed in v0.16.0**: `is_background: true` + BackgroundShellRegistry (PR#3642) + `/tasks` command + output written to disk + settle notification | ✗ `bash` tool has **no background parameter** (Source: `tool/bash.ts#53-59`) |
| Background Subagent launch | ✅ `Agent(..., run_in_background: true)` + completion notification | ✅ PR#3076 (merged 2026-04-17) `Agent` tool supports background + lifecycle events + headless/SDK consistency | ✗ None |
| Parent→child mid-flight control (task_stop / send_message) | ✅ `TaskStopTool` (`task_id` + `shell_id` + `KillShell` alias) + `SendMessage` tool | ✅ **Completed in v0.16.0**: PR#3471 (merged) `task_stop` + `send_message` + per-agent transcript JSONL (**subagent only**; Bash bg is controlled via backgroundShellRegistry `task_stop` shell_id) | ✗ None |
| Parent reads background transcript | ✅ Monitor directly pushes lines + `<task-notification>` / Subagent transcript readable | ✅ **Completed in v0.16.0**: parent can read live transcript (ChatRecord JSONL, sharing schema with main session); Bash bg settle notification includes output path | ✗ None |
| Monitor / event stream | ✅ Independent `Monitor` tool + 200ms throttling + self-stop on overload | ✅ **Completed in v0.16.0**: PR#3684 `Monitor` tool + `MonitorRegistry` (635 lines) + 200ms throttling + self-stop on overload | ✗ None |
| Status-bar counters | ✅ `N shell, M monitor` real-time classification | ✅ **Completed in v0.16.0**: PR#3488 (merged) background subagent pill + combined dialog; Bash bg / Monitor also included in counters (PR#3720/#3791) | ✗ None |
| Unified management UI | ✅ `/tasks` (`/bashes` alias) + `↓` key + separate detail views for Shell/Monitor | ✅ **Completed in v0.16.0**: `/tasks` command (PR#3642) + Combined Background tasks dialog (PR#3720/#3791) + detail view (PR#3488) | ✗ No task panel |
| Notification push (event → LLM) | ✅ `<task-notification>` system-message injection (subagent + shell + monitor) | ✅ subagent completion notification (PR#3076) + mid-flight transcript (PR#3471) + Monitor event notification (PR#3684) + Bash bg settle (PR#3642) | ✗ |

**Claude Code remains the most complete design, but Qwen Code v0.16.0 has completed the full 5-PR stack. The main remaining gaps are: (1) streaming tool execution as API response chunks arrive, (2) automatic Kairos backgrounding (move to background after 15s timeout), and (3) manual Ctrl+B foreground→background UX (Qwen Code has a foreground→background promote mechanism, but the UX differs).**

### Related Qwen Code PRs (complete v0.16.0 status)

Qwen Code completed the full 5-PR stack in v0.16.0, covering Bash bg pool, Monitor, subagent control, and UI:

```
PR#3076 ✅ MERGED (2026-04-17) — background subagent launch
PR#3642 ✅ MERGED              — Bash bg pool + /tasks command
PR#3684 ✅ MERGED              — Monitor tool + MonitorRegistry
PR#3471 ✅ MERGED              — model-facing agent control (task_stop, send_message, transcript)
PR#3488 ✅ MERGED              — UI: pill + combined dialog + detail view
```

And later extensions:

```
PR#3720 ✅ MERGED — Background shells included in tasks dialog
PR#3791 ✅ MERGED — Monitor entries included in tasks dialog
PR#3801 ✅ MERGED — /tasks command includes Monitor + interactive-mode hint
```

**[PR#3642](https://github.com/QwenLM/qwen-code/pull/3642)** `feat(core): managed background shell pool with /tasks command` (**merged**) — adds complete shell-pool management to `is_background: true` for the `shell` tool: subprocess registration (`BackgroundShellRegistry`, 339 lines), output written to disk, settle notification (including exit code), and `/tasks` text command. This matches the core of Claude Code's shell pool.

**[PR#3684](https://github.com/QwenLM/qwen-code/pull/3684)** `feat(core): event monitor tool with throttled stdout streaming` (**merged**) — adds the `Monitor` tool + `MonitorRegistry` (635 lines), pushes each stdout line back to the agent as a notification, batches/throttles at 200ms, and self-stops on overload. This fully matches Claude Code's Monitor tool.

**[PR#3471](https://github.com/QwenLM/qwen-code/pull/3471)** `feat(core): model-facing agent control (task_stop, send_message, per-agent transcript)` (**merged**) — `task_stop` tool + `send_message` tool + per-agent transcript JSONL.

**[PR#3488](https://github.com/QwenLM/qwen-code/pull/3488)** `feat(cli): background-agent UI — pill, combined dialog, detail view` (**merged**) — status-line pill counters + combined dialog + per-item detail view.

The 5 components matching Claude Code (v0.16.0 status):

| Component | Qwen Code v0.16.0 status |
|---|---|
| 1. Shell pool | ✅ PR#3642 (BackgroundShellRegistry, 339 lines) |
| 2. Monitor pool | ✅ PR#3684 (MonitorRegistry, 635 lines) |
| 3. Status-bar aggregator | ✅ PR#3488 + PR#3720 + PR#3791 (full shell + monitor + subagent coverage) |
| 4. Management UI | ✅ PR#3488 + PR#3720 + PR#3791 (/tasks + combined dialog) |
| 5. Notification injector | ✅ PR#3076 (subagent) + PR#3684 (monitor) + PR#3642 (bash bg settle) |

## 9. Why this design matters

### User perspective: avoid anxiety that "the agent is secretly still working"

Without a status-bar reminder:
- You think the conversation ended → but a dev server is still occupying port 8080
- You press Ctrl+D to exit → a background task silently leaks into the OS (orphan process)
- A Monitor notification suddenly appears → you do not know why

The visibility of `1 shell, 1 monitor` turns "passive transparency" into "active awareness," consistent with the first [Heuristic Evaluation principle](https://www.nngroup.com/articles/ten-usability-heuristics/): users should be able to see system status.

### Agent perspective: async concurrency unlocks more complex workflows

The **Bash bg + Monitor combination** can build:

```
1. Start dev server (Bash bg)
2. Start log monitor watching for ERROR (Monitor)
3. Run typecheck + tests at the same time (foreground)
4. At any point, monitor pushes back an ERROR notification → agent debugs immediately
```

This "agent writes code while monitoring" pattern is **already possible** in Qwen Code v0.16.0: v0.16.0 added BackgroundShellRegistry (Bash bg pool, PR#3642) and the Monitor tool (PR#3684). Monitor event push gives the agent a concept of "interrupt," and therefore a primitive for "waiting for external conditions." In v0.15.2, `is_background` was a simple fork-and-detach (no pool / no output collection); that barrier is removed in v0.16.0.

### Implementation perspective: minimum required core components

Inferring from external behavior (not source-confirmed), a complete design needs at least:

1. **Shell pool**: subprocess registry + lifecycle management (PID / output file path / kill handle)
2. **Monitor pool**: watch-script registration + stdout streaming filters + notification debounce (200ms batching)
3. **Status-bar aggregator**: counts across the two pools → renders `· N shell, M monitor` in the Footer
4. **Management UI**: list view + detail view (Shell shows Command/Runtime/Output; Monitor shows full Script)
5. **Notification injector**: wraps shell completion events + monitor stdout lines into `<task-notification>` and injects them into the next-turn LLM context

Any agent that wants to copy this design needs all five components. **Qwen Code v0.16.0 completed all of them through the 5-PR stack**:

| Component | Qwen Code v0.16.0 status |
|---|---|
| 1. Shell pool | ✅ [PR#3642](https://github.com/QwenLM/qwen-code/pull/3642) (BackgroundShellRegistry, 339 lines) |
| 2. Monitor pool | ✅ [PR#3684](https://github.com/QwenLM/qwen-code/pull/3684) (MonitorRegistry, 635 lines) |
| 3. Status-bar aggregator | ✅ [PR#3488](https://github.com/QwenLM/qwen-code/pull/3488) + [PR#3720](https://github.com/QwenLM/qwen-code/pull/3720) + [PR#3791](https://github.com/QwenLM/qwen-code/pull/3791) |
| 4. Management UI | ✅ PR#3642 (`/tasks` command) + PR#3720 + PR#3791 (combined dialog) |
| 5. Notification injector | ✅ PR#3076 (subagent) + PR#3684 (monitor per event) + PR#3642 (bash bg settle) |

**Remaining gaps**: (1) automatic Kairos backgrounding (Claude Code moves tasks to the background after a 15s timeout); (2) manual Ctrl+B backgrounding (Qwen Code has foreground→background promote, but the UX differs); (3) streaming tool execution immediately as API stream chunks arrive (StreamingToolExecutor).

## 10. Source-code analysis (based on leaked source)

Source location: `/root/git/claude-code-leaked/` (v2.1.x deobfuscated source, 1934 files). The following lists files + line numbers + key code.

### 10.1 Core data model

**`tasks/LocalShellTask/guards.ts#9-33`** — Shell and Monitor share the same type and are distinguished only by the `kind` field:

```ts
export type BashTaskKind = 'bash' | 'monitor'

export type LocalShellTaskState = TaskStateBase & {
  type: 'local_bash' // Keep as 'local_bash' for backward compatibility with persisted session state
  command: string
  result?: { code: number; interrupted: boolean }
  shellCommand: ShellCommand | null
  isBackgrounded: boolean
  agentId?: AgentId
  // UI display variant. 'monitor' → shows description instead of command,
  // 'Monitor details' dialog title, distinct status bar pill.
  kind?: BashTaskKind
}
```

The source comment states directly: **Monitor is a UI display variant of a `local_bash` task**. The differences are:
1. The list shows description rather than command
2. The detail dialog title is "Monitor details"
3. The status-bar pill counts it separately

### 10.2 Status-bar text generation

**`tasks/pillLabel.ts#11-30`** — the `getPillLabel` function is used by both the Footer pill and the inline turn-duration line:

```ts
export function getPillLabel(tasks: BackgroundTaskState[]): string {
  const n = tasks.length
  const allSameType = tasks.every(t => t.type === tasks[0]!.type)
  if (allSameType) {
    switch (tasks[0]!.type) {
      case 'local_bash': {
        const monitors = count(
          tasks,
          t => t.type === 'local_bash' && t.kind === 'monitor',
        )
        const shells = n - monitors
        const parts: string[] = []
        if (shells > 0)
          parts.push(shells === 1 ? '1 shell' : `${shells} shells`)
        if (monitors > 0)
          parts.push(monitors === 1 ? '1 monitor' : `${monitors} monitors`)
        return parts.join(', ')
      }
      // ... 6 other task types
    }
  }
  return `${n} background ${n === 1 ? 'task' : 'tasks'}`
}
```

The source confirms **singular/plural handling** (`shells` / `monitors`) and **mixed display** (`1 shell, 1 monitor` joined with `, `).

### 10.3 Full set of 7 background task types

`tasks/types.ts#13-21` + `pillLabel.ts` define **7 task types** (not just shell + monitor):

| Type | Source | Footer pill text |
|---|---|---|
| `local_bash` (kind='bash') | Bash tool + run_in_background | `N shell(s)` |
| `local_bash` (kind='monitor') | Monitor tool | `N monitor(s)` |
| `local_agent` | Agent tool + run_in_background | `N local agent(s)` |
| `remote_agent` | Cloud sessions | `N cloud session(s)`; ultraplan uses `◆ ultraplan ready` / `◇ ultraplan needs your input` |
| `in_process_teammate` | TeamCreate / multi-agent collaboration | `N team(s)` |
| `local_workflow` | (independent workflow type, PR likely swarm-related) | `N background workflow(s)` |
| `monitor_mcp` | (MCP server monitoring, **different from the Monitor tool**) | `N monitor(s)` |
| `dream` | Auto Dream system | `dreaming` (no count, solo mode) |

**Note**: `monitor_mcp` and `local_bash kind=monitor` both display `N monitor`, but the **underlying task types are different**: the former is MCP server health monitoring, while the latter is a Monitor tool instance.

Source: `tasks/pillLabel.ts#34-67`

### 10.4 Background logic in the Bash tool

**`tools/BashTool/BashTool.tsx`** key constants (observed line numbers):

```ts
const PROGRESS_THRESHOLD_MS = 2000;            // #55
const ASSISTANT_BLOCKING_BUDGET_MS = 15_000;   // #57
const COMMON_BACKGROUND_COMMANDS = [           // #265
  'npm', 'yarn', 'pnpm', 'node', 'python', 'python3',
  'go', 'cargo', 'make', 'docker', 'terraform',
  'webpack', 'vite', 'jest', 'pytest',
  'curl', 'wget', 'build', 'test', 'serve', 'watch', 'dev',
] as const;
```

Note: there are **22 actual items** (the earlier [04-tools.md excerpt](../tools/claude-code/04-tools.md#L651) listed only 10). The full list includes `yarn / pnpm / python3 / go / terraform / curl / wget`, etc.

**Kairos automatic backgrounding trigger** (`BashTool.tsx#974-985`):

```ts
// blocking commands after ASSISTANT_BLOCKING_BUDGET_MS so the agent can keep
if (feature('KAIROS') && getKairosActive() && isMainThread &&
    !isBackgroundTasksDisabled && run_in_background !== true) {
  // ... setTimeout(..., ASSISTANT_BLOCKING_BUDGET_MS).unref()
}
```

Automatic backgrounding happens only when Kairos mode is enabled + main thread + the user did not explicitly set `run_in_background: true`.

**Sleep interception + Monitor recommendation** (`BashTool.tsx#525-530`) — when a Bash command contains `sleep > 2s`, Claude directly **refuses to execute** and returns an error suggesting Monitor:

```ts
if (feature('MONITOR_TOOL') && !isBackgroundTasksDisabled && !input.run_in_background) {
  // ... if (sleepPattern) return Blocked
  message: `Blocked: ${sleepPattern}. Run blocking commands in the background with 
  run_in_background: true — you'll get a completion notification when done. 
  For streaming events (watching logs, polling APIs), use the Monitor tool. 
  If you genuinely need a delay (rate limiting, deliberate pacing), keep it under 2 seconds.`
}
```

This is the actual error-message text. When the LLM writes `sleep 60`, this message guides it to use Monitor or `run_in_background: true`.

### 10.5 MAX_TASK_OUTPUT_BYTES disk-output limit

**`utils/task/diskOutput.ts#30`**:

```ts
export const MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024  // 5 GB
```

### 10.6 TaskStopTool (including KillShell alias)

**`tools/TaskStopTool/TaskStopTool.ts#11-18, #38-46`**:

```ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    task_id: z.string().optional().describe('The ID of the background task to stop'),
    // shell_id is accepted for backward compatibility with the deprecated KillShell tool
    shell_id: z.string().optional().describe('Deprecated: use task_id instead'),
  }),
)

export const TaskStopTool = buildTool({
  name: TASK_STOP_TOOL_NAME,
  searchHint: 'kill a running background task',
  // KillShell is the deprecated name - kept as alias for backward compatibility
  // with existing transcripts and SDK users
  aliases: ['KillShell'],
  // ...
})
```

`shell_id` exists only for replay compatibility with old transcripts from the KillShell era; **new SDKs should use only `task_id`**.

### 10.7 Lazy loading of the Monitor tool

**`tools.ts#39-40, #237`**:

```ts
const MonitorTool = feature('MONITOR_TOOL')
  ? require('./tools/MonitorTool/MonitorTool.js').MonitorTool
  : undefined
// ...
...(MonitorTool ? [MonitorTool] : []),  // add to tools list
```

The Monitor implementation directory `tools/MonitorTool/` is **not included** in the leaked source (dead-code elimination after the feature gate removed it), but the `feature('MONITOR_TOOL')` call confirms it is controlled by a GrowthBook feature flag.

### 10.8 Footer pill rendering

**`components/PromptInput/PromptInputFooterLeftSide.tsx#17`** imports `BackgroundTaskStatus`:

```ts
import { BackgroundTaskStatus } from '../tasks/BackgroundTaskStatus.js';
```

**`components/tasks/BackgroundTaskStatus.tsx#10, #25-92`** calls `getPillLabel` to render mainPill + teammatePills (stacked display when multiple teams exist).

### 10.9 BackgroundTasksDialog (`/tasks` management UI)

**`components/tasks/BackgroundTasksDialog.tsx#409`** — UI title:

```ts
{runningBashCount !== 1 ? 'active shells' : 'active shell'}
```

Note that this uses only `runningBashCount` (total local_bash count, including monitors); other task types are counted separately.

**#414** lists all supported task types for the keyboard shortcut `x` to stop:

```ts
...((currentSelection?.type === 'local_bash' || 
     currentSelection?.type === 'local_agent' || 
     currentSelection?.type === 'in_process_teammate' || 
     currentSelection?.type === 'local_workflow' || 
     currentSelection?.type === 'monitor_mcp' || 
     currentSelection?.type === 'dream' || 
     currentSelection?.type === 'remote_agent') && 
    currentSelection.status === 'running' 
  ? [<KeyboardShortcutHint key="kill" shortcut="x" action="stop" />] : [])
```

All 7 task types support stopping with the `x` key.

### 10.10 ShellDetailDialog (details for both Shell + Monitor)

**`components/tasks/ShellDetailDialog.tsx#164`** — dynamic title:

```ts
const t9 = isMonitor ? "Monitor details" : "Shell details";
```

**#177, #193, #253** — render the three fields `Status:` / `Runtime:` / `Output:`. It is the same component; only the title switches based on `isMonitor`.

### 10.11 Source of the inline turn status line

**`components/messages/SystemTextMessage.tsx#508, #568`**:

```ts
// #508 — calculate backgroundTaskSummary
return running.length > 0 ? getPillLabel(running) : null;

// #568 — concatenate turn timing + bg summary
const t8 = backgroundTaskSummary && ` · ${backgroundTaskSummary} still running`;
//                                    ↑ middle dot ·
const t7 = showTurnDuration && `${verb} for ${duration}`;
// Render: <Text dimColor>{t7}{budgetSuffix}{t8}</Text>
//   →  ✻ Cooked for 7s · 1 shell, 1 monitor still running
```

`getPillLabel` drives both the Footer pill and the turn-end status line here, so the two displays are **guaranteed consistent** (same function).

## 11. Binary analysis (v2.1.119)

Binary: `/root/.local/share/claude/versions/2.1.119`, 245 MB ELF Linux x86-64, **not stripped** (symbol table + JS source strings retained).

### 11.1 Verifying UI strings

```bash
strings 2.1.119 | grep -E "^(1 shell|1 monitor|Monitor details|Shell details)$"
```

Output:

```
1 shell
1 monitor
Monitor details
Shell details
```

✅ All 4 UI strings **exist exactly** in the binary. "Monitor details" / "Shell details" each appears 6 times (different code paths).

### 11.2 Monitor tool description

The complete Monitor tool schema can be found inside the binary:

```js
var mL="Monitor",wH6='Start a background monitor that streams events from a long-running script. ...'
```

This is the Monitor tool description source string seen by the LLM, proving that the Monitor tool is indeed in the binary (even though leaked source lacks the `tools/MonitorTool/` directory).

### 11.3 Verifying COMMON_BACKGROUND_COMMANDS

```bash
strings 2.1.119 | grep -E "^(yarn|pnpm|webpack|vite|terraform|cargo)$" | sort -u
```

Output: `cargo / pnpm / terraform / vite / webpack / yarn` — these 22 command names all exist as standalone strings in the binary (the minifier did not expand array-literal strings).

### 11.4 Minifier loss

Source constant names such as `ASSISTANT_BLOCKING_BUDGET_MS` / `MAX_TASK_OUTPUT_BYTES` / `PROGRESS_THRESHOLD_MS` have **already been inlined by esbuild into literal values** (15000 / 5368709120 / 2000) in the binary, so grep cannot find the names. But source maps can restore them (if available), and observed behavior fully matches the source-defined numbers.

## 12. Details of the other 5 task types

Section 10.3 gave an overview table of 7 task types. This section goes deeper into the capabilities, trigger scenarios, and source details of the remaining 5 types (excluding the shell + monitor dual forms of `local_bash`).

### 12.1 `local_agent` (background Subagent)

**Trigger**: Agent tool + `run_in_background: true`:

```ts
Agent({
  description: "Research X in parallel",
  prompt: "...",
  subagent_type: "researcher",
  run_in_background: true   // ← does not block the parent agent
})
```

**Characteristics** ([`tasks/LocalAgentTask/LocalAgentTask.tsx#33-110`](file:/root/git/claude-code-leaked/tasks/LocalAgentTask/LocalAgentTask.tsx)):

```ts
export type AgentProgress = {
  toolUseCount: number;       // how many tools the child agent has called
  tokenCount: number;         // cumulative tokens
  currentActivity?: string;   // what it is currently doing
  // ...
}

export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent';
  // includes progress tracker, reported to the parent agent in real time
}
```

**Real-time progress return**: every tool call by the child agent updates the progress visible to the parent agent. The Footer pill shows `1 local agent`, and the details dialog (`AsyncAgentDetailDialog.tsx`) shows token usage + a timeline of tool activity.

**On completion**: the child agent's final answer + total tokens + elapsed time are pushed back to the parent agent through `<task-notification>`, and the parent agent sees them in the next-turn context.

**Typical scenarios**: parallel research across multiple independent subtasks (one worktree per child agent), long-task delegation (let the child agent run slowly while the parent keeps coordinating).

### 12.2 `remote_agent` (Cloud Sessions)

**Trigger**: one of the following slash commands:
- `/autofix-pr <PR>` — fix PR test / lint errors
- `/ultrareview <PR>` — multi-agent cloud review
- `/ultraplan` — long plan reasoning
- Other `RemoteTaskType` values (source `REMOTE_TASK_TYPES` array inside `RemoteAgentTask.tsx`)

**Characteristics** ([`tasks/RemoteAgentTask/RemoteAgentTask.tsx#21-50`](file:/root/git/claude-code-leaked/tasks/RemoteAgentTask/RemoteAgentTask.tsx)):

```ts
export type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent';
  remoteTaskType: RemoteTaskType;  // autofix-pr / ultrareview / ultraplan / ...
  sessionId: string;               // remote session ID
  command: string;
  title: string;
  todoList: TodoList;              // child todos returned in real time
  log: SDKMessage[];               // remote SDK message stream (pollRemoteSessionEvents)
  isLongRunning?: boolean;
  pollStartedAt: number;           // prevents clock drift on restore
  isRemoteReview?: boolean;        // /ultrareview marker
  reviewProgress?: {                // <remote-review-progress> heartbeat parsing
    stage?: 'finding' | 'verifying' | 'synthesizing';
    bugsFound: number;
    bugsVerified: number;
    bugsRefuted: number;
  };
}
```

**Remote execution**: the task runs in an Anthropic Cloud container; local `pollRemoteSessionEvents` pulls events every N seconds to backfill `todoList` + `log`. Even if the local Claude Code process exits, the remote task **continues** and can be resumed on the next startup.

**ultraplan special visual**: [`tasks/pillLabel.ts#36-50`](file:/root/git/claude-code-leaked/tasks/pillLabel.ts) uses diamond symbols to distinguish phases:

```
remote_agent + ultraplanPhase === 'plan_ready'      → ◆ ultraplan ready
remote_agent + ultraplanPhase === 'needs_input'     → ◇ ultraplan needs your input
remote_agent + ultraplanPhase === undefined         → ◇ ultraplan
```

`◆` solid = waiting for the user to review the plan; `◇` hollow = still running or waiting for input.

### 12.3 `in_process_teammate` (same-process Team member)

**Trigger**: the `TeamCreate` tool creates a team, and team members start via the team definition; or the user presses `f` in `/tasks` to foreground into the teammate view.

**Characteristics** ([`tasks/InProcessTeammateTask/types.ts#13-50`](file:/root/git/claude-code-leaked/tasks/InProcessTeammateTask/types.ts)):

```ts
export type TeammateIdentity = {
  agentId: string;          // "researcher@my-team"
  agentName: string;
  teamName: string;
  parentSessionId: string;  // Leader's session ID
  planModeRequired: boolean;
}

export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity;
  permissionMode: PermissionMode;     // ← independent cycle, switched with Shift+Tab
  awaitingPlanApproval: boolean;
  abortController?: AbortController;        // stop WHOLE teammate
  currentWorkAbortController?: AbortController;  // abort only current turn
  result?: AgentToolResult;
  progress?: AgentProgress;
}
```

**Key differences from `local_agent`**:
1. **Same process vs child process**: teammate is isolated inside the main process with AsyncLocalStorage; subagent is a forked child process
2. **Independent permission cycle**: each teammate has its own `permissionMode`; Shift+Tab switching does not affect others
3. **Bidirectional interaction**: the user can press `f` to foreground into a teammate view and send prompts directly to the teammate (not possible for subagents)
4. **Two-level abort**: `abortController` stops the entire teammate; `currentWorkAbortController` aborts only the current turn so the teammate enters a new turn

**Typical scenario**: multiple concurrently collaborating role agents (researcher / reviewer / writer), with the user freely switching between leader and teammate views.

### 12.4 `local_workflow` (Anthropic internal only)

**Gate**: `feature('WORKFLOW_SCRIPTS')` + `build_flags.yaml`; the **external build dead-code-eliminates the entire ~1.3K-line workflow engine**. Source comment:

```ts
// WORKFLOW_SCRIPTS is ant-only (build_flags.yaml). Static imports would leak
// ~1.3K lines into external builds. Gate with feature() + require so the
// bundler can dead-code-eliminate the branch.
```

**Typical scenario** (inferred, not observed): internal release / build / deploy automation scripts that let the agent trigger predefined workflows. External users **cannot trigger or see this task type**.

The `tasks/LocalWorkflowTask/` directory in leaked source is empty, proving that dead-code elimination removed the whole implementation. Only the `Task.ts#11` union type + `BackgroundTasksDialog.tsx#82-86` UI branch remain (to avoid breaking types when support is added later).

### 12.5 `monitor_mcp` (MCP server health monitoring)

**This is not the same thing as the Monitor tool**. Core differences:

| Dimension | Monitor tool | `monitor_mcp` task |
|---|---|---|
| Internal task type | `local_bash` + `kind='monitor'` | `monitor_mcp` (independent type) |
| ID prefix | `b` (inherits local_bash) | `m` |
| Triggered by | LLM agent calling the Monitor tool | MCP client manager automatically |
| Monitored object | Any script (`tail -f` / `inotifywait` / poll loop) | MCP server startup state / health |
| Footer text | `N monitor` | `N monitor` (**same text for both!**) |
| Details dialog | Reuses `ShellDetailDialog` ("Monitor details") | Handled independently (`BackgroundTasksDialog#392`) |

**Handling Footer text collision**: [`tasks/pillLabel.ts#67`](file:/root/git/claude-code-leaked/tasks/pillLabel.ts) gives `monitor_mcp` its own case: "1 monitor" / "N monitors". But **when both are present** (`local_bash:monitor` + `monitor_mcp`), the pill degrades to `N background tasks` because the `allSameType` check fails.

**Typical scenario**: MCP server slow startup, disconnect retries, initialization progress. Leaked source has no `tasks/MonitorMcpTask/` implementation (feature-gated dead code), but 4 references in `BackgroundTasksDialog.tsx#198, #278, #392, #537` prove it is active in production.

### 12.6 `dream` (Auto Dream memory consolidation)

**Trigger**: **fully automatic**. The Auto Dream system forks a child agent every N turns / at a token threshold to consolidate the user's `CLAUDE.md` memory. No explicit user or LLM trigger is required.

Source: [`tasks/DreamTask/DreamTask.ts#1-26`](file:/root/git/claude-code-leaked/tasks/DreamTask/DreamTask.ts):

```ts
// Background task entry for auto-dream (memory consolidation subagent).
// Makes the otherwise-invisible forked agent visible in the footer pill and
// Shift+Down dialog. The dream agent itself is unchanged — this is pure UI
// surfacing via the existing task registry.

const MAX_TURNS = 30  // display only the last 30 turns

// No phase detection — the dream prompt has a 4-stage structure
// (orient/gather/consolidate/prune) but we don't parse it. Just flip from
// 'starting' to 'updating' when the first Edit/Write tool_use lands.
export type DreamPhase = 'starting' | 'updating'

export type DreamTurn = {
  text: string
  toolUseCount: number  // tool calls are collapsed into a count
}
```

**Dream agent 4-stage prompt** (revealed by source comments; the dream agent itself does not parse stages and only switches starting → updating based on whether an Edit/Write tool appears):
1. **orient** — read existing memory and build understanding
2. **gather** — collect new information from the current round
3. **consolidate** — integrate into memory
4. **prune** — remove stale entries

**Footer display**: solo `dreaming` (no count N) — `pillLabel.ts#65` has a dedicated `case 'dream': return 'dreaming'`.

**Typical scenario**: near the end of a long session, the LLM summarizes conversation insights → automatically writes them into `~/.claude/CLAUDE.md`. User-visible behavior: a `dreaming` pill suddenly appears, and a few minutes later `~/.claude/CLAUDE.md` has several new memories.

### 12.7 `LocalMainSessionTask` (backgrounding the main session by pressing Ctrl+B twice)

**This is not a new type; it is a special use of `local_agent`**. Source [`tasks/LocalMainSessionTask.ts#1-10`](file:/root/git/claude-code-leaked/tasks/LocalMainSessionTask.ts):

```ts
/**
 * LocalMainSessionTask - Handles backgrounding the main session query.
 *
 * When user presses Ctrl+B twice during a query, the session is "backgrounded":
 * - The query continues running in the background
 * - The UI clears to a fresh prompt
 * - A notification is sent when the query completes
 *
 * This reuses the LocalAgentTask state structure since the behavior is similar.
 */
```

**Trigger**: the user presses **Ctrl+B twice** while the main session is in progress. The whole main-session query is moved to the background, and the UI is immediately cleared to accept a new prompt.

**Typical scenario**: you assigned the agent a long task (for example "refactor the whole module"), it has run for 30 seconds and is still not done, but you want to insert a new instruction. Ctrl+B Ctrl+B backgrounds the current turn and opens a new conversation; when the background turn finishes, a notification returns.

This is effectively **the user proactively forking themselves**, corresponding to `local_agent` (where the agent proactively forks a child agent).

### 12.8 Summary: async capability spectrum

Grouped by "agent knows" vs "user knows":

| Task type | Actively created by agent | User-triggerable | User can interact independently |
|---|---|---|---|
| `local_bash` (shell) | ✅ run_in_background=true | ✅ Ctrl+B (foreground → background) | ❌ only stop / view output |
| `local_bash` (monitor) | ✅ Monitor tool | ❌ | ❌ only stop |
| `local_agent` | ✅ Agent + run_in_background | ❌ | ❌ only stop / view progress |
| `LocalMainSessionTask` | ❌ | ✅ Ctrl+B Ctrl+B | ❌ |
| `remote_agent` | ❌ | ✅ /autofix-pr / /ultrareview / /ultraplan | ✅ review when Plan is ready |
| `in_process_teammate` | ✅ TeamCreate | ✅ team startup | ✅ **press `f` to foreground and interact directly** |
| `local_workflow` | (ant-only) | (ant-only) | ? |
| `monitor_mcp` | ❌ (system automatic) | ❌ | ❌ only stop |
| `dream` | ❌ (system automatic) | ❌ | ❌ only view progress |

**Strongest capability**: `in_process_teammate`: it supports both background concurrency and foreground interaction. Others are either purely passive or only support stop/view.

**Most ordinary capability**: `local_bash` + `local_agent`: agent-initiated forked "helpers" that report back when done.

**Most cloud-native**: `remote_agent`: remote tasks keep running even when local is offline.

## 13. Reproduction commands from real testing

```bash
mkdir -p /tmp/cc-bg-test && cd /tmp/cc-bg-test
tmux new-session -d -s cc -x 90 -y 35 'cd /tmp/cc-bg-test && claude'
sleep 6
tmux send-keys -t cc Enter            # trust directory

# 1. Trigger Bash bg
tmux send-keys -t cc "Run 'sleep 60' as a background bash command" Enter
sleep 8

# 2. Trigger Monitor
tmux send-keys -t cc "Use the Monitor tool to watch /tmp/cc-bg-test/watched.log for ERROR lines" Enter
sleep 30

# 3. Inspect status bar
tmux capture-pane -t cc -p | tail -5
# Should see:
#   /tmp/cc-bg-test
#   ⏵⏵ auto mode on · 1 shell, 1 monitor

# 4. Open management UI
tmux send-keys -t cc "/tasks" Enter
sleep 2
tmux capture-pane -t cc -p | tail -10

# 5. Clean up
tmux kill-session -t cc
rm -rf /tmp/cc-bg-test
```

## Evidence sources

### Real testing

- Claude Code v2.1.120 running inside tmux 90×35 capture: [`screenshots/claude-code-bg-tasks-90x35.txt`](./screenshots/claude-code-bg-tasks-90x35.txt)

### Source (leaked, v2.1.x, path `/root/git/claude-code-leaked/`)

| File | Key lines | Content |
|---|---|---|
| `tasks/LocalShellTask/guards.ts` | 9, 33 | `BashTaskKind = 'bash' \| 'monitor'` + `kind?: BashTaskKind` field + comments |
| `tasks/LocalShellTask/LocalShellTask.tsx` | 522 LOC | Main shell task implementation |
| `tasks/pillLabel.ts` | 11-30 | `getPillLabel` status-bar text generation (including singular/plural) |
| `tasks/types.ts` | 13-21 | Union type for 7 BackgroundTaskState variants |
| `tools/BashTool/BashTool.tsx` | 55, 57, 241, 265, 525, 974 | Constant definitions + run_in_background schema + sleep interception |
| `tools/TaskStopTool/TaskStopTool.ts` | 11-18, 38-46 | task_id / shell_id schema + `aliases: ['KillShell']` |
| `tools.ts` | 39-40, 237 | Monitor tool lazy load + `feature('MONITOR_TOOL')` gate |
| `utils/task/diskOutput.ts` | 30 | `MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024` |
| `components/messages/SystemTextMessage.tsx` | 508, 568 | Inline turn status line uses `getPillLabel` + ` · ... still running` |
| `components/tasks/BackgroundTaskStatus.tsx` | 10, 25-92 | Footer pill component |
| `components/tasks/BackgroundTasksDialog.tsx` | 409, 414 | `/tasks` management UI (`active shell(s)` title + support for 7 task types) |
| `components/tasks/ShellDetailDialog.tsx` | 164, 177-253 | Shell/Monitor detail view (shared component, `isMonitor ? "Monitor details" : "Shell details"`) |
| `components/PromptInput/PromptInputFooterLeftSide.tsx` | 17 | Imports `BackgroundTaskStatus` to render in Footer |

### Binary (`/root/.local/share/claude/versions/2.1.119`, 245 MB ELF)

| Verification item | Command | Result |
|---|---|---|
| UI strings | `strings 2.1.119 \| grep -E "^(1 shell\|1 monitor\|Monitor details\|Shell details)$"` | All 4 items exist exactly |
| Monitor tool name | `strings 2.1.119 \| grep 'mL="Monitor"'` | `var mL="Monitor"` |
| Monitor description | Complete tool description string | Exists in full |
| 22 background commands | `strings 2.1.119 \| grep -E "^(yarn\|pnpm\|webpack\|...)$"` | All exist exactly |
| Constant numbers | `nm -a 2.1.119` or source map | Inlined by esbuild; names disappeared but values (15000/5368709120/2000) remain |

### Public documentation

- [04-tools.md §4.4.7 Background process management](../tools/claude-code/04-tools.md), [03-architecture.md](../tools/claude-code/03-architecture.md)
- [EVIDENCE.md](../tools/claude-code/EVIDENCE.md) (based on earlier v2.1.x binary decompilation)

### Related Qwen Code PRs (all merged in v0.16.0)

- [PR#3076](https://github.com/QwenLM/qwen-code/pull/3076) `feat: background subagents` (merged 2026-04-17, Agent background launch)
- [PR#3471](https://github.com/QwenLM/qwen-code/pull/3471) `feat(core): model-facing agent control` (merged, task_stop / send_message / per-agent transcript JSONL)
- [PR#3488](https://github.com/QwenLM/qwen-code/pull/3488) `feat(cli): background-agent UI` (merged, pill + combined dialog + detail view)
- [PR#3642](https://github.com/QwenLM/qwen-code/pull/3642) `feat(core): managed background shell pool with /tasks command` (merged)
- [PR#3684](https://github.com/QwenLM/qwen-code/pull/3684) `feat(core): event monitor tool with throttled stdout streaming` (merged)
- [PR#3720](https://github.com/QwenLM/qwen-code/pull/3720) `feat(cli): wire background shells into combined Background tasks dialog` (merged)
- [PR#3791](https://github.com/QwenLM/qwen-code/pull/3791) `feat(cli): wire Monitor entries into combined Background tasks dialog` (merged)

> **Disclaimer**:
> - Claude Code real testing used the v2.1.120 binary; source analysis used the v2.1.x leaked dump (versions may differ slightly, but the core architecture is stable)
> - Qwen Code portions were based partly on a 2026 Q1 draft and rechecked against v0.16.0 on 2026-05-22; Qwen Code PR status has been fully updated to the actual v0.16.0 state (PR#3471/#3488, marked OPEN in the original document, have been merged, and PR#3642/#3684 were added to cover Bash bg pool and Monitor)
> - Binary decompilation may lose some metadata; actual trigger conditions for `local_workflow` / `monitor_mcp` among the 7 task types were not empirically verified in this article
