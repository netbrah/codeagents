# Claude Code Capability Borrowing Comparison: Qwen Code vs Qoder CLI

> **Question**: Qwen Code and Qoder CLI are both **sibling forks of Gemini CLI**, yet both are converging toward **Claude Code**. Using the [Qwen Code improvement proposal report (benchmarked against Claude Code, 275 items)](./qwen-code-improvement-report.md) as the reference framework, this article compares category by category **what each product borrowed from Claude Code, and how deeply**.
>
> **One-sentence conclusion**: **Both products converge toward Claude Code from different angles**—**Qwen Code borrows the "engine/depth"** (subagent orchestration, fork, multiple Hook types, `tool_search`, structured output, compression, memory, Computer Use... mostly implemented through source-code PRs); **Qoder CLI borrows the "surface/compatibility"** (CLI parameter names, Hook event names, verbatim agent definition text, tool names such as `Bash/Read/Edit`, `migrate --from-claude`, the `.claude-plugin` marketplace... to reduce migration cost for Claude Code users).
>
> **Evidence**: The Qwen side follows the [improvement report](./qwen-code-improvement-report.md) (PR-level tracking); the Qoder side is based on decoding the v1.0.18 bundle (`_$d()` deobfuscation, deterministic and reproducible; see [EVIDENCE](../tools/qoder-cli/EVIDENCE.md)). As of 2026-06-14.
>
> **Related**: [Symmetric comparison of Qwen Code vs Qoder CLI](./qwen-code-vs-qoder-cli.md) · [Qwen × Qoder integration capability comparison](./qwen-code-qoder-integration.md)

---

## 0. Why Both Borrow from Claude Code

The shared base (Gemini CLI) itself **does not include** several signature Claude Code designs—confirmed by grepping upstream gemini-cli one by one: **no** subagent system (`tools/agent/`, `subagents/`, `subagent_type`/fork, built-in agent registry), **no** Claude-named Hook events, and **no** `tool_search`. Yet these exist in both forks, proving that **each added Claude Code designs on top of the Gemini base**.

The difference lies in the **layer** they added:

| | Qwen Code | Qoder CLI v1.0 |
|---|---|---|
| Borrowing angle | **Engine/runtime depth** | **Surface/migration compatibility** |
| Typical action | **Reimplement Claude capabilities inside the engine** (PR-level, open-source and traceable) | **Align with Claude interfaces/names/text** (visible through bundle decoding) |
| Goal | Become an open-source runtime platform | Reduce switching cost for Claude Code users |
| Depth signals | Agent Team, Computer Use, Tool Search, structured output, three-tier compression... | CLI parameter surface, Hook event names, verbatim agent text, `migrate --from-claude` |

---

## 1. SubAgent System (Both Borrow Heavily, but at Different Depths)

Claude Code's subagent system is the largest area **both** products borrowed—the built-in agent description text is **verbatim identical** in Qwen source and the Qoder bundle, and both match the original Claude Code text.

| Claude Code capability | Qwen Code borrowing | Qoder CLI borrowing |
|---|---|---|
| `subagent_type` tool parameter | ✅ Source `tools/agent/agent.ts` | ✅ TaskTool (decoded trigger logic is verbatim identical) |
| Built-in `general-purpose`/`Explore` agent | ✅ (verbatim text) | ✅ (verbatim text) |
| Built-in `Plan` architect agent | — (Qwen does not include a built-in Plan agent; it has `enter_plan_mode`) | ✅ `Plan` ("Software architect agent...") |
| Built-in `statusline-setup` agent | ✅ | ✅ |
| **Fork Subagent** (omitting type = implicit fork, inherits full context) | ✅ FORK_AGENT ([report item-2](./qwen-code-improvement-report.md), PR#2936) | ✅ **Same design** (description + trigger logic are verbatim identical) |
| `isolation: worktree` field | ✅ | ✅ `enum[default,worktree]` |
| `/agents` management UI | ✅ (create/manage wizard) | ✅ (create/list) |
| **Coordinator/Swarm multi-agent orchestration** | ✅ **Complete Agent Team** (lifecycle events + broadcast + shared task board, [report item-14/25](./qwen-code-improvement-report.md)) | ⚠️ **Only `teammate_mailbox` result aggregation** (no orchestration) |
| Fine-grained agent tool access control | ✅ (3-layer allowlist, PR#3064/#3066) | ✅ + **rules match by `subagent_type` dimension** |
| Agent recovery and continuation | ✅ (PR#3739) | Not seen |

**Interpretation**: Both borrow the **definition and basic invocation** of subagents to the same depth (both directly copy Claude text). But for Claude's **deeper multi-agent orchestration** (Coordinator/Swarm), Qwen reimplemented a complete Agent Team, while Qoder stops at result aggregation. Conversely, Qoder includes **more** Claude agents by default (additional `Plan`/`SaveMemory`/`skill-extractor`).

---

## 2. Hooks (Both Switch to Claude Event Names)

Upstream Gemini CLI Hooks use their own names (`BeforeTool`/`AfterAgent`...) and provide `hooks migrate` to **convert** Claude configurations into those names. **Both Alibaba forks instead directly adopt Claude Code's event names** (the upstream names count as 0 in both).

| Claude Code Hook capability | Qwen Code | Qoder CLI |
|---|---|---|
| 10 event names (`PreToolUse`/`PostToolUse`/`PostToolUseFailure`/`UserPromptSubmit`/`SessionStart`·`End`/`PreCompact`/`Stop`·`SubagentStop`/`Notification`) | ✅ Full set | ✅ Full set (counts 24~55×) |
| **HTTP Hooks** (POST JSON) | ✅ ([report, PR#2827](./qwen-code-improvement-report.md)) | ❌ Command only + `/hook-config` |
| Function Hook / Async Hook | ✅ | ❌ |
| SSRF protection | ✅ (`ssrfGuard.ts`) | ❌ |
| `CLAUDE_PROJECT_DIR`/`CLAUDE_PLUGIN_ROOT` environment variables | Partial | ✅ (Claude plugin compatibility) |

**Interpretation**: **Both borrow the event names** (configurations can be shared); but only Qwen reimplements Claude Hook **type extensions** (HTTP/Function/Async), while Qoder stops at commands.

---

## 3. CLI Surface / Parameter Naming (Qoder Borrows Far More Than Qwen)

This is the area where **Qoder's borrowing from Claude Code is most visible**—the CLI parameter surface of Qoder v1.0 is broadly aligned with Claude Code, while upstream Gemini CLI and Qwen Code do not have these names.

| Claude Code CLI parameter/concept | Qwen Code | Qoder CLI |
|---|---|---|
| `--permission-mode` | ❌ (uses `--approval-mode`) | ✅ |
| `--output-style` / Output Styles | ❌ | ✅ (`outputStyle` 4×) |
| `--add-dir` | ❌ (uses `--include-directories`) | ✅ |
| `--settings` / `--setting-sources` | ❌ | ✅ |
| `--strict-mcp-config` | ❌ | ✅ |
| `--agents <json>` | ❌ | ✅ |
| `--allowed-tools`/`--append-system-prompt`/`--fork-session` | ✅ Partial | ✅ |
| Tool names `Bash`/`Read`/`Write`/`Edit`/`Glob`/`Grep`/`WebSearch`/`WebFetch` | ❌ (keeps Gemini-style `run_shell_command`/`read_file`) | ✅ **Full Claude naming set** |

**Interpretation**: **Qoder almost copies Claude Code's user interface** (parameters + tool names) so Claude Code users can start with "zero learning"; Qwen keeps Gemini-style names and borrows only a small number of parameters. This is the clearest expression of the difference in borrowing angle.

---

## 4. Migration and Ecosystem Compatibility (Claude Borrowing Unique to Qoder)

| Claude Code compatibility capability | Qwen Code | Qoder CLI |
|---|---|---|
| **One-click migration** `migrate --from-claude` | ❌ | ✅ (one-time migration of MCP/hooks/commands/subagents) |
| `.claude-plugin` plugin format | ❌ | ✅ (marketplace recognition) |
| `claude-plugins-official` marketplace source | ❌ | ✅ Preconfigured |
| Context-usage UI labels `claudeMd` type | — | ✅ |
| Reads `CLAUDE.md` (runtime) | ✅ (reads it) | v0.x had `--with-claude-config`; **v1.0 removed it** and replaced it with a migration command |

**Interpretation**: **The migration toolchain is a Claude borrowing almost exclusive to Qoder**—serving users who are "switching over from Claude Code." Qwen only achieves Hook-name compatibility and has no migration command.

---

## 5. Engine/Runtime Capabilities (Qwen Borrows Far More Deeply Than Qoder)

For these **engine-level** Claude Code capabilities, Qwen has reimplemented most of them (PR-level tracking in the report), while Qoder has invested little.

| Claude Code capability | Qwen Code | Qoder CLI |
|---|---|---|
| **Computer Use** (GUI automation) | ✅ 9 tools (enabled by default + deferred) | ❌ (full bundle audit found none) |
| **Tool Search / deferred tool loading** | ✅ `tool_search` ([report, PR#3589](./qwen-code-improvement-report.md)) | ❌ (only `deferred_tools_delta` notification) |
| **Structured Output** (`--json-schema`) | ✅ (PR#3598) | ❌ |
| **Multi-layer context compression** | ✅ Three-tier gradient (PR#4345) | `/compact` (single tier) |
| **Session memory + Auto Dream** | ✅ (PR#3087 managed memory) | `/memory` + `SaveMemory` (no auto-dream) |
| **Follow-up / Prompt Suggestion** (suggest next prompt after a turn) | ✅ `followup_suggestion`(42×)/`prompt_suggestion`(15×)/`PromptSuggestion`(32×) | ✅ **Same design, same name**—`PromptSuggestion` subsystem (model generation + accept/reject validation) + event `{type:"prompt_suggestion",suggestion,uuid,sessionId}`; also `awareness_nudge`/`generate_session_title` fast-model assisted generation |
| **Speculation prediction + pre-execution** (zero-latency Tab acceptance) | ✅ (PR#2525, suggestion + speculative pre-execution) | ⚠️ **Only the suggestion half, no pre-execution** (`speculat`/`ghostText`/`Tab to accept` all 0) |
| **Mid-Turn Queue Drain (mid-turn injection)** | ✅ `drainQueue` (32×, [report item-6](./qwen-code-improvement-report.md), PR#2854) | ✅ **Same pattern, called steering**—`InjectionService.addInjection(input,"user_steering")` queues + `getInjectionsAfter(startIndex,"user_steering")` consumes within the turn + wraps injection in `<user_input>` ("User steering update" / `steering-ack`) |
| **Notebook Edit** | ✅ (PR#3900) | ❌ |
| **FileIndex fuzzy search** | ✅ (fzf-style) | Not seen |
| **Fast Model / Advisor model** | ✅ Multiple scenarios | ❌ (`fastModel`/`advisor` 0) |
| Commit Attribution (AI contribution statistics) | ✅ Uniquely exceeds Claude (git notes) | ✅ `commit` subcommand (AI percentage) |
| Plan mode + Approval Gate | ✅ `enter_plan_mode` (PR#4853) | ✅ `/plan` (Toggle Plan Mode) |
| Thinking block rendering/collapsing | ✅ (web-shell #4977) | ⚠️ Has `thinking`/`reasoning` rendering; collapsing unconfirmed |

**Interpretation**: Claude Code's **heavy runtime capabilities** (Computer Use / Tool Search / structured output / layered compression / Speculation **pre-execution**) are mostly pursued by Qwen alone. But **interaction-experience runtime capabilities** are borrowed by both—mid-turn injection (Qwen `drainQueue` / Qoder `user_steering`) and Follow-up Suggestion (both call it `prompt_suggestion`/`PromptSuggestion`, derived from Claude). Pattern: **Qoder also fills in runtime capabilities aimed at user interaction, but does not invest in heavy platform/automation-oriented capabilities**.

---

## 6. Being Integrated / Northbound Interfaces (Claude Code's Platform Capabilities)

Claude Code has an SDK, Bridge, and daemon-style control plane. The borrowing gap between the two products is huge.

| Claude Code platform capability | Qwen Code | Qoder CLI |
|---|---|---|
| SDK (programmatic embedding) | ✅ TS/Python/Java | ❌ No public SDK |
| Invoked as an MCP server | ✅ `qwen-serve-bridge` | ❌ MCP client only |
| Remote Control Bridge (phone/browser-driven) | ✅ Channels + daemon | ✅ `--remote`/`/qr-code` (tied to Qoder Cloud) |
| ACP HTTP/WS | ✅ | ⚠️ stdio only |
| `/teleport` cross-device migration | In development | ✅ `--teleport` (Qoder Cloud) |

**Interpretation**: Claude's **open platform capabilities** are still being systematically borrowed by Qwen (SDK/MCP-server/ACP HTTP-WS); Qoder's remote feature is a closed loop tied to its own cloud. See [integration comparison](./qwen-code-vs-qoder-cli.md#%E4%B8%83%E8%A2%AB%E9%9B%86%E6%88%90%E8%83%BD%E5%8A%9B%E4%BD%9C%E4%B8%BA%E6%9C%8D%E5%8A%A1%E7%AB%AF--%E5%8F%AF%E5%B5%8C%E5%85%A5%E5%AF%B9%E8%B1%A1).

---

## 7. Commands and Meta-Skills (Both Borrow Claude Command Habits)

Both products have copied several signature Claude Code commands/skills, and Qoder is more complete on meta-skills.

| Claude Code command/skill | Qwen Code | Qoder CLI |
|---|---|---|
| `/rewind` (timeline rollback) | ✅ | ✅ (alias `checkpoint`) |
| `/statusline` (custom status line, using a sub-agent) | ✅ | ✅ ("Uses a sub-agent...") |
| `/security-review` | ✅ (bundled skill) | ✅ (bundled skill) |
| `/context` (context visualization) | ✅ | ✅ |
| `/compact` | ✅ `/compress` | ✅ |
| **skill-creator meta-skill** | ✅ (`skills/builtin/skill-creator`) | ✅ (`builtin/skill-creator`) |
| **agent-creator meta-skill** | — (`/agents` wizard) | ✅ (`builtin/agent-creator` guidance skill) |

**Interpretation**: At the command layer, both products move toward Claude habits; **Qoder is more complete in meta-skills (teaching users to write agents/skills)**—it also copies Claude's "creation guidance."

---

## 8. Quantitative Summary: How Much Claude Code Each Borrowed

> Roughly categorized by ~50 Claude Code capability points in this article (not exhaustive; only intended to show borrowing **density and angle**).

| Borrowing category | Qwen Code | Qoder CLI |
|---|---|---|
| **Engine/runtime depth** (subagent orchestration, fork, `tool_search`, computer use, structured output, compression, memory, speculation...) | ★★★★★ Systematic reimplementation (PR-level) | ★★ Only fork + basic subagent |
| **Surface/UX alignment** (CLI parameters, tool names, output-style, permission-mode) | ★★ A few parameters | ★★★★★ Almost copied wholesale |
| **Hook event names** | ★★★★★ (+ HTTP/Function/Async extensions) | ★★★ (name alignment, types not extended) |
| **Migration/ecosystem compatibility** (migrate/.claude-plugin/market) | ★ Hook names only | ★★★★★ Complete migration chain |
| **Platform/being integrated** (SDK/MCP-server/ACP HTTP-WS) | ★★★★★ | ★ Remote tied to own cloud |
| **Command/meta-skill habits** | ★★★★ | ★★★★ (more complete meta-skills) |

---

## 9. Conclusion: Two Paths of "Converging Toward Claude"

- **Qwen Code = borrowing Claude's "internal strength"**. It **reimplements Claude Code's engine capabilities** (multi-agent orchestration, fork, `tool_search`, computer use, structured output, layered compression, memory, SDK/Bridge) **inside an open-source engine**, aiming to become a runtime platform at the same level as Claude Code. The borrowing is **deep, internalized, and traceable to PRs**.
- **Qoder CLI = borrowing Claude's "skin"**. It **aligns Claude Code's user interface** (CLI parameters, tool names, output-style, Hook event names, agent definition text, migration chain, plugin marketplace format) **as completely as possible**, aiming to let Claude Code users **migrate painlessly** to a self-operated gateway billing product. The borrowing is **broad, surface-level, and compatibility-oriented**.

Both borrow equally heavily in subagent definitions, Hook names, and command habits (both directly copy Claude text/names). The split is: **Qwen is far deeper in engine depth**, while **Qoder is far stronger in migration/surface compatibility**. This maps exactly to their product positioning—an open-source runtime platform vs a closed-source, migration-friendly commercial product.

---

> **Disclaimer**: Qwen's borrowing status follows the [improvement report](./qwen-code-improvement-report.md) (continuous PR tracking) and may lag the latest main branch; Qoder's side is based on static decoding of the v1.0.18 bundle (no source code), with borrowing judgments made by comparing against public Claude Code characteristics. "Borrowing" means capability/interface similarity and does not claim code common origin (both codebases are based on Gemini CLI). As of 2026-06-14.
