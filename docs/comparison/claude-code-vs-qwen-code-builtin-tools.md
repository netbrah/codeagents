# Claude Code vs Qwen Code built-in tools comparison

> **Data sources**:
> - **Claude Code v2.1.139**: `/root/.local/share/claude/versions/2.1.139` single-file ELF (231MB, x86-64), `strings` decompilation + [docs/tools/claude-code/04-tools.md](../tools/claude-code/04-tools.md)
> - **Qwen Code v0.15.10**: `/root/git/qwen-code/packages/core/src/tools/tool-names.ts` + source code in the tools directory (as of 2026-05-12; commits include PR#4002 + PR#4022 + PR#4041 + PR#4069)
> - **Companion reading**: [Tool-Search Deep-Dive](./tool-search-deep-dive.md) / [Tool-Parallelism Deep-Dive](./tool-parallelism-deep-dive.md) / [ReadFile Tool Deep-Dive](./read-file-tool-deep-dive.md) / [BriefTool Async Deep-Dive](./brieftool-async-user-messages-deep-dive.md)
> - **Update history**: Revised on 2026-05-12 after confirming in the binary that Claude Code already has a built-in `Monitor` tool. The earlier judgment that it was unique to Qwen was incorrect; sections 3.3 / 3.11 / 6 / 8 were updated accordingly.

## 1. TL;DR

| Dimension | Claude Code | Qwen Code |
|---|---|---|
| **Total built-in tools** | **~32 internal tool names + ~7 internal/conditional = ~39** (30 hits in v2.1.139 binary `strings` + ReadMcpResourceTool / ListMcpResourcesTool + conditional REPLTool / SleepTool / PowerShell, etc.) | **~21** (including the foundation for dynamic MCP registration) |
| **Loading strategy** | ToolSearch deferred loading (core ~10 always loaded / ~20+ deferred tools require `ToolSearch("select:<name>")` to fetch schema) | Partial deferral (PR#3589 + PR#4022, 9 deferred items total) |
| **Monitor tool** | Built in (confirmed in v2.1.139 binary; includes `persistent` flag + TaskStop cancellation + 200ms batch merging + auto-stop on excessive output) | Built in (introduced by PR#3791, `monitor.ts` 742 lines) |
| **Task* tool group** | TaskCreate / TaskList / TaskGet / TaskUpdate / TaskOutput / TaskStop (6 fine-grained tools) | TodoWrite + TaskStop (2 tools, covering create/update paths) |
| **MCP tools** | Dynamic `mcp__<server>__<tool>` registration + ReadMcpResourceTool / ListMcpResourcesTool bridge | Same mechanism in `mcp-tool.ts` |
| **System prompt savings** | ~60% (15K -> 6K tokens) | Partial (PR#3589 + PR#4022 and other deferred-loading work landed) |
| **Design difference** | More tools + strong categorization + deferred loading + tool description engineering (deep hints such as "silence is not success") | Leaner tool count + unified abstraction through the 4-kinds Background framework |

## 2. Tool-count comparison

| Category | Claude Code | Qwen Code | Difference |
|---|:---:|:---:|---|
| Core (always loaded) | 10 | ~12 | Qwen lacks top-level ToolSearch / StructuredOutput tools |
| Deferred (loaded on demand) | 25 | 9 | Claude's deferred pool is much larger |
| Internal / conditional | 4 | 0 | Claude has REPLTool / SleepTool / PowerShell |
| MCP dynamic | ∞ | ∞ | Both support it |
| **Total (excluding MCP)** | **~39** | **~21** | **Claude ~1.85x** |

## 3. Detailed mapping table (Claude -> Qwen correspondences)

### 3.1 File / search tools

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `Read` | `read_file` (ReadFile) | Aligned + Qwen later surpassed it | PR#3717 / 3774 / 3810 / 3932 / 4002 FileReadCache + prior-read guard chain align with Claude behavior |
| `Write` | `write_file` (WriteFile) | Aligned | partial-read acceptance semantics in PR#3932 / 4002 match Claude |
| `Edit` | `edit` (Edit) | Aligned | Qwen also has `replace_all` mode |
| `Glob` | `glob` (Glob) | Aligned | -- |
| `Grep` | `grep_search` (Grep) + `ripGrep` (separate ts file) | Dual implementation | Qwen has a generic `grep.ts` version + a high-performance `ripGrep.ts` version |
| `LS` (Claude internal?) | `list_directory` (LS / ListFiles) | Aligned | Qwen has an independent tool |

### 3.2 Execution tools

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `Bash` | `run_shell_command` (Shell) | Aligned | Claude 12,411 LOC / 18 files / 23 safety validations; Qwen `shell.ts` includes AST read-only detection |
| `PowerShell` (Windows conditional) | None | Missing in Qwen | Claude enables it only on Windows |
| `REPLTool` (internal) | None | Missing in Qwen | Used internally by Claude |
| `SleepTool` (internal) | None | Missing in Qwen | Used internally by Claude (sleep / wait) |

### 3.3 Agent / task scheduling tools

> **Binary confirmation**: v2.1.139 binary `strings` hits the following six Task* tool-name strings: `TaskCreate` / `TaskList` / `TaskGet` / `TaskUpdate` / `TaskOutput` / `TaskStop`, plus `Agent` / `Monitor` / `PushNotification` / `ScheduleWakeup` / `RemoteTrigger` / `CronCreate` / `CronList` / `CronDelete`.

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `Agent` | `agent` (Agent) | Aligned | Qwen has `agent.ts` + `fork-subagent.ts` under the agent/ subdirectory |
| `TaskCreate` / `TaskList` / `TaskGet` / `TaskUpdate` / `TaskOutput` (6 independent tools) | Single `todo_write` (TodoWrite) tool | Granularity difference | Claude separates five tools (create/list/get/update/output), making precise LLM operations easier; Qwen uses one TodoWrite tool to cover both create+update paths |
| `TaskStop` (including KillShell alias) | `task_stop` (TaskStop) | Aligned | Same pattern in Qwen (PR#3642 + PR#3836 4-kinds dispatch) |
| `Monitor` (**confirmed present in v2.1.139**) | `monitor` (Monitor) | **Aligned** (earlier docs incorrectly judged it unique to Qwen; revised after 2026-05-12 binary evidence) | See detailed comparison in section 3.12 |
| `PushNotification` (deferred) | None | Missing in Qwen | Claude asynchronous push-notification tool |
| `ScheduleWakeup` (deferred) | None | Missing in Qwen | Claude schedules future wakeups (paired with /loop dynamic mode) |
| `RemoteTrigger` (deferred) | None | Missing in Qwen | Claude remote-trigger capability (moved to section 3.10) |
| `TodoWrite` | `todo_write` (TodoWrite) | Aligned | Claude has been replaced by TaskCreate/Update; Qwen keeps it |

### 3.4 Web / fetching tools

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `WebFetch` | `web_fetch` (WebFetch) | Aligned | Claude 1,131 LOC |
| `WebSearch` | **Missing** (removed in PR#3502 -> PR#3844 OPEN to add it back) | Pending in Qwen | Qwen previously removed WebSearch (PR#3502, -1830 lines); community report #3841 cited codeagents [web-search-tool-deep-dive](./web-search-tool-deep-dive.md), driving PR#3844 OPEN to restore it |
| `NotebookEdit` | None | Missing in Qwen | Claude 587 LOC, supports editing Jupyter `.ipynb` |
| `WebFetch` Auto Clean | Has prompt-injection defense | Same-style design (PR#3844 design) |

### 3.5 LSP / Code Intelligence

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `LSP` | `lsp` (Lsp) | Aligned | Qwen `lsp.ts` has a complete implementation |

### 3.6 User-interaction tools

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `AskUserQuestion` | `ask_user_question` (AskUserQuestion) | Aligned | PR#4041 changed this Qwen tool from deferred back to always-visible to prevent prose fallback |
| `SendMessage` | `send_message` (SendMessage) | Aligned | Sends messages to background tasks |
| `Brief` (Claude deferred tool) | None | Missing in Qwen | Claude `brieftool` asynchronous user-message mechanism ([Brief Tool Deep-Dive](./brieftool-async-user-messages-deep-dive.md)) |
| `ExitPlanMode` | `exit_plan_mode` (ExitPlanMode) | Aligned | -- |
| `PlanMode*` tool group | Only `exit_plan_mode` | Partial alignment | Claude has multiple PlanMode-related tools; Qwen only exposes exit |

### 3.7 Cron / scheduling tools

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `Cron*` tool group (CronCreate / CronList / CronDelete) | `cron_create` / `cron_list` / `cron_delete` (CronCreate / CronList / CronDelete) | **Fully aligned** | Qwen splits them into three independent tools (PR#3589 ToolSearch deferred by default) |

### 3.8 Memory / Skills

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `Skill` | `skill` (Skill) | Aligned | Qwen PR#3852 path-conditional activation (based on discovered paths, more precise than Claude) |
| `Memory` series (Claude uses services/memdir rather than tools) | `save_memory` (SaveMemory) | Different implementation layer | Claude puts memory in the services layer; Qwen exposes it as a top-level tool |

### 3.9 Worktree / advanced Git

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `EnterWorktree` / `ExitWorktree` | `enter_worktree` / `exit_worktree` (**PR#4073 OPEN 2026-05-12**) + `agent` tool `isolation: 'worktree'` parameter | **Pending merge in Qwen** (under review) | **PR#4073** (+1651/-4, LaZzyMan, closes Phase A+B of #4056): two new tools directly matching Claude tool names + agent isolation parameter + worktree cleanup + dirty-state guard + slug validation. See [improvement-report 2026-05-12 second round](./qwen-code-improvement-report.md#2026-05-12-second-round) |

### 3.10 Remote / team / global tools

| Claude Code | Qwen Code | Status | Notes |
|---|---|---|---|
| `RemoteTrigger` (confirmed present in v2.1.139 binary, deferred) | None | Missing in Qwen | Claude remote-trigger capability; may correspond to the design direction of PR#3929-3931 mobile UI / pairing token |
| `Team*` tool group (Claude deferred) | None | Missing in Qwen | Claude team collaboration (shared by multiple users) |
| `Config` (Claude deferred) | None | Missing in Qwen | Claude dynamic settings tool ([Config Tool Deep-Dive](./config-tool-dynamic-settings-deep-dive.md)) |
| `StructuredOutput` (Claude core) | PR#4001 OPEN (feat: structured JSON schema output) | In progress in Qwen | Corresponds to a Claude core tool; Qwen implements it through a CLI flag (`--json-schema`) rather than a tool |
| `ToolSearch` (confirmed present in v2.1.139 binary, core) | PR#3589 partial implementation + PR#4069 adds `tools.toolSearch.enabled` switch | Partial in Qwen | Qwen implemented the deferred mechanism, but PR#4069 reveals a conflict with prefix cache (DeepSeek users reported +214% cost) |
| `Skill` (confirmed present in v2.1.139 binary) | `skill` (Skill) | Aligned | Covered in section 3.8 |

### 3.11 Qwen-only items (after revision)

| Qwen Code | Claude Code | Status | Notes |
|---|---|---|---|
| `save_memory` (SaveMemory) | Not exposed as a tool (goes through services/memdir layer) | Interface-layer difference | See section 6 |
| `4 kinds Background tasks framework` (unified scheduling for agent/shell/monitor/dream) | No equivalent abstraction found (each background type has an independent path) | **Qwen-only architecture** | Four-PR series: PR#3642 + PR#3720 + PR#3791 + PR#3836. Claude does not have a unified background-type scheduling framework (Agent + Monitor + Bash run_in_background are implemented separately) |

> **Key revision** (2026-05-12): The `monitor` tool, previously listed as "Qwen-only", was confirmed to exist in the Claude Code v2.1.139 binary (`strings` hit `JW="Monitor"`, the complete tool description contains "Start a background monitor that streams events from a long-running script", the `persistent: true` parameter, etc.). It was therefore removed from the Qwen-only list; see section 3.12 below for the comparison.

### 3.12 Monitor tool: detailed Claude vs Qwen comparison

| Dimension | Claude Code Monitor | Qwen Code monitor |
|---|---|---|
| **Tool name** | `Monitor` (PascalCase) | `monitor` (snake_case alias `Monitor`) |
| **Core positioning** | "Start a background monitor that streams events from a long-running script. Each stdout line is an event" (original description) | Same semantics; description is basically consistent |
| **stdout -> notification mapping** | Each stdout line -> one chat notification (multiple lines within 200ms are batch-merged) | Same pattern (throttling prevents stdout flooding) |
| **Do not use unbounded commands for one-shot notifications** | Description explicitly warns that `tail -f log \| grep -m 1` can hang when SIGPIPE is missing; use Bash `run_in_background` + an `until` loop instead | No equivalent warning found in Qwen description (design worth borrowing) |
| **Coverage warning ("silence is not success")** | Description explicitly says filters must match every terminal state, not only the happy path; recommends broad `grep -E "elapsed_steps=\|Traceback\|Error\|FAILED\|assert\|Killed\|OOM"` | No equivalent warning found in Qwen description (design worth borrowing) |
| **stdout buffering advice** | Strongly recommends `grep --line-buffered`; poll loops use `\|\| true` for transient failures | Qwen description has no explicit guidance |
| **Polling-frequency advice** | Remote APIs 30s+; local 0.5-1s | Qwen lets users set it freely |
| **Persistence flag** | `persistent: true` -> lives for the entire session (until TaskStop or session end) | `max_events` + `idle_timeout_ms` auto-stop |
| **Cancellation mechanism** | `TaskStop` (unified task-cancellation interface) | Same pattern |
| **stdout 200ms batch merge** | Explicit in description (avoid event flooding) | Qwen `monitor.ts` also has throttling |
| **Auto-stop on excessive output** | Explicit in description: "Monitors that produce too many events are automatically stopped" | Qwen `max_events` default 100 |
| **stderr routing** | stderr -> output file (readable by Read), does not trigger notifications; recommends `2>&1` merge | Same pattern in Qwen `monitor.ts` (output file is readable) |
| **LOC (estimate)** | Independent file size not extracted, but description text is ~6KB (with six example blocks) | `monitor.ts` 742 lines |
| **Loading strategy** | Always loaded (in deferred-tool list; requires ToolSearch to retrieve schema) | Moved to deferred after PR#4022 (aligned with Claude) |

**Assessment**: **Claude Monitor is far more rigorous than Qwen `monitor` in prompt engineering (warnings / coverage / buffering hints)**. Claude writes the common failure modes for LLM use of monitoring tools directly into the description: using an unbounded command to seek a one-shot notification, omitting `--line-buffered` from grep and causing latency, and filtering only the happy path. This is exactly Anthropic's maturity advantage in Tool Description Engineering. Qwen `monitor` has a complete implementation and aligned behavior, but its description copy is clearly shorter, leaving substantial room to learn from Claude. Qwen can directly translate Claude's six example blocks plus three quality warnings into the `monitor.ts` description.

## 4. Loading-strategy comparison

### 4.1 Claude Code: ToolSearch deferred loading

- **10 core tools are always loaded into the system prompt** (about ~6K tokens).
- **25 deferred tools** (`shouldDefer: true`) are not in the initial prompt; the model dynamically retrieves schemas through `ToolSearch("keyword")`.
- **Savings**: the system prompt shrinks from ~15K to ~6K tokens (**~60%**).

```
ToolSearch workflow:
  Model: "Need to edit a Jupyter notebook"
  ↓
  ToolSearch("notebook edit") -> returns NotebookEdit schema
  ↓
  Next model turn: NotebookEdit({...})
```

### 4.2 Qwen Code: partial deferral

- **PR#3589** introduced the ToolSearch mechanism and initially deferred five items: `cron_create`, `cron_list`, `cron_delete`, `ask_user_question`, `exit_plan_mode`, `lsp`, and MCP-related tools.
- **PR#4022** (2026-05-11) extended deferral with four items: `monitor`, `send_message`, `task_stop`, `web_fetch`, aligning with Claude's deferred strategy.
- **PR#4041** (2026-05-11) reversed one decision: `ask_user_question` was forced to `shouldDefer=false` because after deferral the model used prose instead of structured multi-choice.
- **PR#4069** (2026-05-12 OPEN) added a global `tools.toolSearch.enabled` switch + automatic disabling for `deepseek-v4-*` to avoid prefix-cache conflicts.

### 4.3 Design trade-off: token savings vs cache hit

[The fundamental tension revealed by PR#4069](./qwen-code-improvement-report.md#2026-05-12) (from real user [discussions/4065](https://github.com/QwenLM/qwen-code/discussions/4065)):

| Dimension | Before ToolSearch | After ToolSearch |
|---|---|---|
| Initial tool list | All ~30 tools | Core ~10 tools + dynamic discovery |
| Token count | Higher | Reduced by ~46% |
| Cache prefix | Stable -> 97.5% cache hit | Dynamic -> **81.5%** cache hit |
| DeepSeek daily cost | $1.05 | **$3.30** (+214%) |

DeepSeek `cache_hit` pricing is 1/120 of `cache_miss`; losing cache hits means input token price rises 120x, far exceeding the 46% token-count savings from ToolSearch. **Claude is not affected by this because its model architecture and prompt-cache mechanics differ**.

## 5. Claude-only tools (worth Qwen borrowing)

| Tool | Claude LOC | Purpose | Borrowing priority for Qwen |
|---|---|---|---|
| **WebSearch** | 569 | Search-engine queries | **P0** (PR#3844 OPEN to restore it) |
| **NotebookEdit** | 587 | Edit Jupyter `.ipynb` | P1 (data-science scenarios) |
| ~~Worktree*~~ | -- | git worktree switching / parallel multi-branch work | **Already OPEN** ([PR#4073](https://github.com/QwenLM/qwen-code/pull/4073) 2026-05-12 LaZzyMan +1651/-4, `enter_worktree` / `exit_worktree` + agent `isolation: 'worktree'`, Phase A+B of #4056) |
| **Brief** | -- | Batch collection of asynchronous user messages | P1 ([Brief Tool Deep-Dive](./brieftool-async-user-messages-deep-dive.md)) |
| **RemoteTrigger** | -- | Remote trigger | P3 (corresponds to the PR#3929-3931 remote-control direction) |
| **Team*** | -- | Team collaboration tool group | P3 (enterprise deployment) |
| **Config** | -- | Dynamic settings tool | P2 ([Config Tool Deep-Dive](./config-tool-dynamic-settings-deep-dive.md)) |
| **SleepTool** | -- | Waiting / throttling (internal) | P3 (internal use) |
| **PowerShell** | -- | Windows-platform shell | P2 (cross-platform Windows support) |

## 6. Qwen-only tools (not in Claude)

> **Revision record (2026-05-12)**: The `monitor` tool in the original table has been confirmed, by counterevidence from the v2.1.139 binary, to also be built into Claude. It has been removed from this section and moved to the detailed comparison in section 3.12.

| Tool | LOC | Purpose | Design highlight |
|---|---|---|---|
| **save_memory** (top-level tool) | ~300 | Write user-level memory (cross-session) | Claude implements this through the services/memdir layer plus an automatic memory system (an "auto memory" rule block built into the system prompt). Qwen exposes it as a top-level tool with a lower call barrier, but lacks Claude's automatic writing rules |
| **4 kinds Background tasks framework** | - | Unified scheduling for four background task types: agent/shell/monitor/dream | Four-PR series PR#3642 + PR#3720 + PR#3791 + PR#3836; Claude implements each background type separately and has no unified abstraction |

## 7. Shared tools with implementation differences

### 7.1 Read tool: Qwen later surpassed Claude

| Dimension | Claude Code | Qwen Code |
|---|---|---|
| LOC | 1,602 | 440 (read-file.ts) |
| Cache mechanism | LRU 1000 entries + mtime invalidation | session-scoped FileReadCache (`(dev, ino)` key) |
| Prior-read guard | Built in | Fully aligned by PR#3774 + PR#3932 + PR#4002 |
| Binary detection | mime + extension + 4KB sample | Same pattern in PR#4002 (KNOWN_TEXT_EXTENSIONS 50+ + mime + sample 3-step) |
| Partial read | Accepted | Accepted in PR#3932 Edit path |
| Image compression | Has token cap | Partial (see 12 borrowable items in [ReadFile Tool Deep-Dive](./read-file-tool-deep-dive.md)) |

**Assessment**: After a five-PR series across 13 days (PR#3717 -> 3810 -> 3774 -> 3932 -> 4002), Qwen has aligned with Claude's behavior. See [item-2 file read cache](./qwen-code-improvement-report-p0-p1-engine.md#item-2).

### 7.2 Bash / Shell: Claude is stricter

| Dimension | Claude Code | Qwen Code |
|---|---|---|
| LOC | 12,411 (18 files) | 3,652 (single file shell.ts) |
| Safety validation | 23 items | AST read-only detection + command allowlist |
| Sandbox mechanism | Yes | NoSandbox by default ([section 06, part 5, External Reference sandbox](./qwen-code-daemon-design/06-roadmap.md), External scope) |
| Background running | `run_in_background` field | PR#3642 BackgroundShellRegistry + PR#3894 Ctrl+B promote |
| Sed simulation | `_simulatedSedEdit` edits directly without going through shell | None |
| `dangerouslyDisableSandbox` | Yes | None (no sandbox) |

**Assessment**: Claude Bash is far more mature than Qwen across safety validation / sandbox / command classification. However, Qwen's 4-kinds Background framework (PR#3642 + PR#3720 + PR#3791 + PR#3836) is a unified background-tasks scheduling surface that Claude does not have.

### 7.3 Cron: Qwen is more fine-grained

| Dimension | Claude Code | Qwen Code |
|---|---|---|
| Tool count | Cron* group | Split into three independent tools: CronCreate / CronList / CronDelete |
| Persistence | settings | `~/.qwen/crons.json` |
| Loading mode | Deferred | Deferred (PR#3589) |

### 7.4 Skill: Qwen path-conditional activation is more precise

| Dimension | Claude Code | Qwen Code |
|---|---|---|
| Path-conditional activation | Based on tool input paths | **PR#3852 based on discovered paths** (broader coverage: result paths from glob / grep / ripGrep all trigger activation) |
| Path-spoofing defense | -- | PR#3852 only trusts result-side metadata from filesystem tools + structured ripgrep |
| Hot-reload slash command | -- | **PR#3923 (2026-05-08)** `SkillManager.addChangeListener` triggers slash-command reload |
| Symlink support | -- | **PR#3915 (2026-05-07)** allows symlinks pointing outside the skills directory |

**Assessment**: Qwen Skill **surpasses Claude** in three areas: path-conditional activation precision, hot reload, and symlink flexibility.

## 8. One-sentence summary

**Claude Code has ~39 built-in tools vs Qwen Code's ~21** (excluding dynamic MCP registration). Claude is more mature in tool count / ToolSearch deferred-loading strategy / Bash 23 safety validations / fine-grained tool subgroups (**six Task* tools** / Cron* / Worktree* / Team* / PlanMode*), and it **includes a complete Monitor tool** (previously misjudged as Qwen-only; revised after 2026-05-12 binary counterevidence). **Qwen surpasses Claude in four unique advantages: FileReadCache + prior-read guard chain (a five-PR series across 13 days aligned with Claude), the 4-kinds Background tasks framework (unified scheduling abstraction for agent/shell/monitor/dream), Skill path-conditional activation (based on discovered paths), and hot-reload + symlink flexibility**.

**Key design learning directions**:
- **Qwen -> Claude**: unified scheduling abstraction through the 4-kinds Background tasks framework (Claude's background types are still implemented separately); Skill activation based on discovered paths; the complete five-PR FileReadCache prior-read guard chain.
- **Claude -> Qwen**: WebSearch restoration (PR#3844 OPEN); NotebookEdit / Worktree* / Brief / Config tool groups; **Monitor tool description engineering** ("silence is not success", mandatory `grep --line-buffered` hint, and three anti-pattern warnings for using unbounded commands to get one-shot notifications that Qwen `monitor.ts` can directly borrow); cost-aware coordination between ToolSearch and prefix cache (PR#4069 revealed and fixed a +214% DeepSeek cost regression).

## 9. Appendix: Claude Code v2.1.139 binary-decompiled tool list (as of 2026-05-12)

Internal tool-name registrations hit by `strings /root/.local/share/claude/versions/2.1.139` (alphabetical order, 30 items):

```
Agent  AskUserQuestion  Bash  CronCreate  CronDelete  CronList
Edit  EnterPlanMode  EnterWorktree  ExitPlanMode  ExitWorktree
Glob  Grep  Monitor  NotebookEdit  PushNotification  Read
RemoteTrigger  ScheduleWakeup  Skill  TaskCreate  TaskGet  TaskList
TaskOutput  TaskStop  TaskUpdate  ToolSearch  WebFetch  WebSearch  Write
```

Plus `ReadMcpResourceTool` / `ListMcpResourcesTool` (MCP bridge), for a total of **~32 built-in tool names** (excluding dynamically registered MCP namespaces `mcp__<server>__<tool>`).

> Note: This inventory counts **internal tool-name strings** and does not distinguish always-loaded from deferred tools. Some tools (such as `ScheduleWakeup`, `Monitor`, `TaskCreate`, `CronCreate`, etc.) are deferred by default under the ToolSearch strategy and can only be called after retrieving their schema through `ToolSearch("select:<name>")`.
