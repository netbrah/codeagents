# 10. Prompt Suggestions — Developer Reference

> Predicts the user's next action and generates suggestions in advance; after Tab acceptance, execution starts with zero perceived latency (Speculation). Qwen Code has implemented this but keeps it disabled by default (PR#2525 ✓ merged).
>
> **Qwen Code parity target**: suggestion generation flow, 12 filtering rules (to avoid low-quality suggestions), Speculation (predict Tab acceptance and execute API calls in advance)
>
> **Internal codename**: `tengu_chomp_inflection` (GrowthBook feature flag)

## Why Prompt Suggestions Are Needed

### Problem Definition

A typical Code Agent interaction pattern is: user input → Agent execution → next user input. Between "Agent finishes execution" and "user starts typing", there is a **decision gap**—the user needs to think about "what should I ask the Agent to do next?"

For 80%+ of scenarios, the next action is predictable:

| What the Agent just did | What the user usually says |
|-------------|-------------|
| Modified code | "run the tests" |
| Tests failed | "fix this failure" |
| Created a file | "add more functionality here" |
| Reviewed a PR | "commit these changes" |

### Design Philosophy: Speculation

Claude Code's Suggestion system does more than "display suggestion text"—while generating the suggestion, it **assumes the user will accept it** and sends an inference request to the API in advance. When the user presses Tab, the result is already in flight or even returned, achieving **zero perceived latency**.

```
Traditional flow (without Speculation):
  Agent completes → Show suggestion → User presses Tab → Send API request → Wait 2-5s → Start execution
                                      ↑ Latency is here

Speculation flow:
  Agent completes → Show suggestion → Send API request at the same time → User presses Tab → Result ready → Execute immediately
                           ↑ Latency is hidden
```

### Competitive Comparison

| Agent | Intelligent completion | Speculative execution | Filtering mechanism |
|-------|---------|---------|---------|
| **Claude Code** | ✓ Generated after each turn ends | ✓ Speculation + shared Prompt Cache | 12 rules |
| **Gemini CLI** | ✓ Follow-up suggestions | — | Basic filtering |
| **Qwen Code** | ✓ Implemented but disabled by default | ✓ Implemented (PR#2525) | Needs improvement |
| **Copilot CLI** | — | — | — |
| **Cursor** | ✓ Tab completion | ✓ Speculative edits | Editor-level filtering |

### Key Risk: The Cost of Low-Quality Suggestions

If suggestion quality is poor (for example, repeatedly suggesting "continue" or suggesting operations that are already complete), users will **stop reading suggestions**—which is worse than having no suggestions, because occasional good suggestions will also be ignored. Claude Code addresses this with 12 filtering rules.

## Feature Overview

After each assistant response completes, Claude Code automatically predicts what the user may type next and displays it in the input box as blue-purple hint text. The user can accept it with Tab/Enter, or simply type over it.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  stopHooks.ts                                                   │
│  query/stopHooks.ts#L139                                        │
│  Triggered after each assistant response completes              │
│  void executePromptSuggestion(stopHookContext)                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│  executePromptSuggestion()                                      │
│  services/PromptSuggestion/promptSuggestion.ts#L184             │
│  Only handles main-thread requests where                        │
│  querySource === 'repl_main_thread'                             │
│                                                                 │
│  1. tryGenerateSuggestion() — guard checks + generation + filter│
│  2. Write to AppState.promptSuggestion                          │
│  3. If Speculation is enabled → startSpeculation()              │
└────────────────────┬────────────────────────────────────────────┘
                     │
              ┌──────┴──────┐
              ▼             ▼
┌──────────────────────┐  ┌──────────────────────────────────────┐
│ usePromptSuggestion  │  │ startSpeculation()                   │
│ hooks/               │  │ services/PromptSuggestion/           │
│ usePromptSuggestion  │  │ speculation.ts                       │
│ .ts                  │  │ Pre-executes the agent with the       │
│                      │  │ suggestion as hypothetical input      │
│                      │  │ (enabled only for internal Anthropic  │
│                      │  │ users)                               │
│ Manages UI display   │  └──────────────────────────────────────┘
│ Tab/Enter acceptance │
│ Telemetry logging    │
└──────────────────────┘
```

## Generation Flow

### Trigger Entry Point

After each assistant response completes, generation is started asynchronously in fire-and-forget mode during the stop hooks phase:

```typescript
// Source: query/stopHooks.ts#L138-139
if (!isEnvDefinedFalsy(process.env.CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION)) {
  void executePromptSuggestion(stopHookContext)
}
```

Both `--bare` mode (minimal mode that skips hooks, LSP, plugin sync, and so on) and `-p` (non-interactive pipe mode) skip suggestion generation.

### API Invocation Method

Source: `services/PromptSuggestion/promptSuggestion.ts#L294-352`

Calls `runForkedAgent()` to send an independent API request:

- **Cache reuse**: Reuses the main conversation's `cacheSafeParams` and deliberately does not override any API parameters (no `effortValue`, `maxOutputTokens`, etc.) to ensure it hits the main conversation's prompt cache
- **Independent markers**: `querySource: "prompt_suggestion"`, `forkLabel: "prompt_suggestion"`
- **No transcript writes**: `skipTranscript: true, skipCacheWrite: true`
- **No tools**: All tool calls are rejected through the `canUseTool` callback (`behavior: "deny"`), so the model can only return plain text

> **Historical lesson**: According to source comments (`promptSuggestion.ts#L308-318`), Anthropic internally tried setting `effort:'low'` to reduce suggestion cost, but the cache hit rate crashed from 92.7% to 61% (45x cache write spike). The billing cache key includes more parameters than the documentation describes; any difference will bust the cache.

### Suggestion Prompt

Source: `services/PromptSuggestion/promptSuggestion.ts#L258-287`, constant `SUGGESTION_PROMPT`

```
[SUGGESTION MODE: Suggest what the user might naturally type next into Claude Code.]

FIRST: Look at the user's recent messages and original request.

Your job is to predict what THEY would type - not what you think they should do.

THE TEST: Would they think "I was just about to type that"?

EXAMPLES:
User asked "fix the bug and run tests", bug is fixed → "run the tests"
After code written → "try it out"
Claude offers options → suggest the one the user would likely pick, based on conversation
Claude asks to continue → "yes" or "go ahead"
Task complete, obvious follow-up → "commit this" or "push it"
After error or misunderstanding → silence (let them assess/correct)

Be specific: "run the tests" beats "continue".

NEVER SUGGEST:
- Evaluative ("looks good", "thanks")
- Questions ("what about...?")
- Claude-voice ("Let me...", "I'll...", "Here's...")
- New ideas they didn't ask about
- Multiple sentences

Stay silent if the next step isn't obvious from what the user said.

Format: 2-12 words, match the user's style. Or nothing.

Reply with ONLY the suggestion, no quotes or explanation.
```

The prompt is indexed through the `PromptVariant` type (Source: `promptSuggestion.ts#L31-35`), which defines two variants, `'user_intent'` and `'stated_intent'`; both map to the same template. `getPromptVariant()` is currently hard-coded to always return `'user_intent'`. `'stated_intent'` is a reserved variant and is not used.

## Filtering Mechanism

Source: `services/PromptSuggestion/promptSuggestion.ts#L354-456`, function `shouldFilterSuggestion`

Generated suggestions are strictly screened by 12 filtering rules; any suggestion that fails the criteria is silently discarded:

| Filtering rule | Description | Match examples |
|----------|------|----------|
| `done` | Content is exactly "done" | `done` |
| `meta_text` | Model outputs meta-description rather than a real prediction | "nothing to suggest", "silence", "nothing found" |
| `meta_wrapped` | Meta-reasoning wrapped in brackets or parentheses | `(silence — ...)`, `[no suggestion]` |
| `error_message` | API error information leaks into output | "api error: ...", "prompt is too long", "image was too large" |
| `prefixed_label` | Has a `word: ` label prefix | "Next step: run tests" |
| `too_few_words` | Fewer than 2 words (slash commands and specific words are allowed) | A single ordinary word (non-whitelisted word) |
| `too_many_words` | More than 12 words | An overly long sentence |
| `too_long` | ≥100 characters | — |
| `multiple_sentences` | Contains multiple sentences (`/[.!?]\s+[A-Z]/`) | "Do this. Then that." |
| `has_formatting` | Contains line breaks or Markdown formatting | Contains `\n`, `*`, `**` |
| `evaluative` | Evaluative or thank-you statements | "looks good", "thanks", "perfect", "awesome" |
| `claude_voice` | Starts in the model's own voice | "Let me...", "I'll...", "Here's...", "You should..." |

**Word whitelist** (Source: `promptSuggestion.ts#L403-424`; not filtered even when only 1 word):

| Category | Words |
|------|------|
| Affirmative words | yes, yeah, yep, yea, yup, sure, ok, okay |
| Action words | push, commit, deploy, stop, continue, check, exit, quit |
| Negative words | no |

## Interaction Methods

| Action | Effect | Telemetry `acceptMethod` |
|------|------|---------------------|
| **Tab** | Accepts the suggestion into the input box (can continue editing before submitting) | `tab` |
| **Enter** (when the input box is empty) | Accepts the suggestion and submits it directly | `enter` |
| **→** (right arrow) | Accepts the suggestion into the input box | — |
| Start typing other content | The suggestion automatically disappears and Speculation is aborted | — |
| Ignore (submit new content directly) | The suggestion is replaced by a new prediction after the next conversation turn | `ignored` |

Acceptance decision logic (Source: `hooks/usePromptSuggestion.ts#L116-117`):
- Tab pressed: `acceptedAt > shownAt`
- Or: final user-submitted content === suggestion text (empty Enter scenario)

## State Data Structure

Source: `state/AppStateStore.ts#L385-393`

```typescript
promptSuggestion: {
  text: string | null           // Suggestion text content; null when there is no suggestion
  promptId: 'user_intent' | 'stated_intent' | null  // Prompt variant identifier; null when there is no suggestion
  shownAt: number               // First render timestamp (Date.now() ms); 0 when not shown
  acceptedAt: number            // Tab acceptance timestamp (Date.now() ms); 0 when not accepted
  generationRequestId: string | null  // Associated API request ID (for RL dataset association)
}
```

> **Default value semantics**: `shownAt` and `acceptedAt` are both `number` types, with `0` meaning "not triggered". The timestamp unit is the millisecond value returned by `Date.now()` (ms since Unix epoch). Acceptance uses `acceptedAt > shownAt` (Source: `usePromptSuggestion.ts#L116`), so `0 > 0` being false means not accepted. Whenever a new suggestion is written, both fields are reset to `0`.

## Suppression Conditions (Three-Layer Guards)

### Initialization Guard

Source: `promptSuggestion.ts#L37-94`, function `shouldEnablePromptSuggestion`

| Check order | Condition | Result |
|----------|------|------|
| 1 | Environment variable is explicitly falsy (`0`/`false`/`no`/`off`) | Force disabled |
| 2 | Environment variable is explicitly truthy (`1`/`true`/`yes`/`on`) | Force enabled |
| 3 | Environment variable is unset or an empty string → proceed to subsequent checks | — |
| 4 | GrowthBook flag `tengu_chomp_inflection` is false | Disabled |
| 5 | Non-interactive mode (`-p`, piped input, SDK) | Disabled |
| 6 | Swarm teammate (non-leader) | Disabled |
| 7 | `settings.promptSuggestionEnabled !== false` | Follows the setting value |

> **Environment variable parsing** (Source: `utils/envUtils.ts#L32-47`): `isEnvDefinedFalsy()` returns true only when the variable is set and its value is `0`/`false`/`no`/`off` (case-insensitive); `isEnvTruthy()` returns true only when the value is `1`/`true`/`yes`/`on`. When the variable is unset (`undefined`) or an empty string, both return false and evaluation continues to GrowthBook and subsequent checks.

### Runtime Guard

Source: `promptSuggestion.ts#L107-119`, function `getSuggestionSuppressReason`

| Condition | Suppression reason |
|------|----------|
| `promptSuggestionEnabled === false` | `disabled` |
| Pending Worker/Sandbox permission request exists | `pending_permission` |
| MCP elicitation queue is non-empty | `elicitation_active` |
| Plan mode is active | `plan_mode` |
| External user and rate limit triggered | `rate_limit` |

### Pre-Generation Guard

Source: `promptSuggestion.ts#L125-182`, function `tryGenerateSuggestion`

| Condition | Suppression reason |
|------|----------|
| AbortController has been aborted | `aborted` |
| Fewer than 2 assistant response turns | `early_conversation` |
| Previous response was an API error | `last_response_error` |
| Total token count of parent messages (`input_tokens + cache_creation_input_tokens + output_tokens`) > 10,000 | `cache_cold` |

## Configuration Methods

| Method | Description |
|------|------|
| `/config` → "Prompt suggestions" | Toggle in the interactive configuration menu |
| Set `"promptSuggestionEnabled": false` in `settings.json` | Persistently disable |
| Environment variable `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=0` | Force disable (highest priority) |
| Environment variable `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=1` | Force enable (highest priority) |
| GrowthBook feature flag `tengu_chomp_inflection` | Server-side gradual rollout control |

## UI Styling

The suggestion text is rendered using the theme's `suggestion` color:

| Theme | Color |
|------|------|
| Light | `rgb(87, 105, 247)` (blue-purple) |
| Dark | `rgb(177, 185, 249)` (light blue-purple) |
| High Contrast Light | `rgb(51, 102, 255)` |
| High Contrast Dark | `rgb(153, 204, 255)` |
| ANSI Light | `ansi:blue` |
| ANSI Dark | `ansi:blueBright` |

## Telemetry Events

### Initialization Event

Event name: `tengu_prompt_suggestion_init` (Source: `promptSuggestion.ts#L41-92`)

| Field | Description |
|------|------|
| `enabled` | Whether enabled |
| `source` | Decision source: `env` / `growthbook` / `non_interactive` / `swarm_teammate` / `setting` |

### Result Event

Event name: `tengu_prompt_suggestion` (Source: `hooks/usePromptSuggestion.ts#L120-157`, `promptSuggestion.ts#L462-523`)

| Field | Description |
|------|------|
| `source` | `cli` (TUI) or `sdk` (API consumer) |
| `outcome` | `accepted` / `ignored` / `suppressed` |
| `prompt_id` | `user_intent` / `stated_intent` |
| `reason` | Suppression reason (only when suppressed) |
| `acceptMethod` | `tab` / `enter` (only when CLI and accepted) |
| `timeToAcceptMs` | Milliseconds from display to acceptance |
| `timeToIgnoreMs` | Milliseconds from display to ignore |
| `timeToFirstKeystrokeMs` | Milliseconds from display to first keystroke |
| `wasFocusedWhenShown` | Whether the terminal had focus when the suggestion appeared |
| `similarity` | `finalInput.length / suggestion.length` (similarity) |

> Internal Anthropic users (`USER_TYPE === 'ant'`) additionally log the raw `suggestion` and `userInput` text for RL dataset training.

## Speculation

Prompt Suggestions are the trigger for the deeper **Speculation** system. After a suggestion is generated, the system immediately uses that suggestion as hypothetical user input and pre-executes one agent response turn.

### Enablement Conditions

Source: `speculation.ts#L337-343`

```typescript
export function isSpeculationEnabled(): boolean {
  const enabled =
    process.env.USER_TYPE === 'ant' &&
    (getGlobalConfig().speculationEnabled ?? true)
  return enabled
}
```

> **Note**: Speculation is enabled only for internal Anthropic users (`USER_TYPE === 'ant'`); external users only use the Prompt Suggestions text prediction feature.

### Core Parameters

Source: `services/PromptSuggestion/speculation.ts#L58-70`

```typescript
const MAX_SPECULATION_TURNS = 20    // Maximum number of speculation turns
const MAX_SPECULATION_MESSAGES = 100 // Maximum number of messages

// Tools allowed to execute during speculation
const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
const SAFE_READ_ONLY_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', 'TaskGet', 'TaskList'
])
```

### File Isolation Mechanism (Copy-on-Write Overlay)

Source: `speculation.ts#L80-81, #L402-715`

- Speculative execution runs in an isolated directory: `$CLAUDE_TEMP_DIR/speculation/{pid}/{id}/`
- Write operations use Copy-on-Write: on the first write, the original file is copied into the overlay directory; subsequent reads and writes are redirected to the overlay
- Write operations outside the CWD are rejected
- On acceptance: overlay files are copied back to the main directory (`copyOverlayToMain`); on abort: the overlay is deleted directly (`safeRemoveOverlay`)

### Boundary Detection

`CompletionBoundary` type (Source: `state/AppStateStore.ts#L41-50`):

| Boundary type | Trigger condition | Behavior |
|----------|----------|------|
| `complete` | Agent completes naturally | Records `outputTokens` |
| `bash` | Non-read-only Bash command (read-only commands such as `ls`/`grep`/`cat` are allowed to execute) | Abort speculation |
| `edit` | File edit but insufficient permissions (not in `acceptEdits`/`bypassPermissions` mode) | Abort speculation |
| `denied_tool` | Tool not in the allowlist (records `detail`: URL/path/command, truncated to the first 200 characters) | Abort speculation |

Speculation tracks the message count in real time through the `onMessage` callback. When it reaches `MAX_SPECULATION_MESSAGES` (100), it automatically aborts to prevent speculation from running indefinitely (Source: `speculation.ts#L637-641`).

### Pipeline Mechanism

Source: `speculation.ts#L345-400`, function `generatePipelinedSuggestion`

After speculative execution completes, if the user has not responded yet, the next suggestion is generated immediately. When the user accepts the current suggestion, **only if speculation completed naturally (`boundary.type === 'complete'`)** will the pipelined suggestion be promoted to the new displayed suggestion and start a new round of speculation; if speculation was aborted due to a bash/edit/denied_tool boundary, the pipelined suggestion is discarded (Source: `speculation.ts#L928-929`).

```
User sends message → Claude replies
  → Generate suggestion A → Start speculation A
    → speculation A completes → Generate pipelined suggestion B
      → User accepts A → Promote B to current suggestion → Start speculation B
        → ...
```

Speculation uses an independent fork label: `querySource: 'speculation'`, `forkLabel: 'speculation'` (Source: `speculation.ts#L633-634`), distinct from suggestion generation's `querySource: 'prompt_suggestion'`.

### Post-Acceptance Handling

After the user accepts a suggestion, the acceptance flow is completed by two cooperating functions:
- `handleSpeculationAccept()` (Source: `speculation.ts#L835-991`): React layer, responsible for state updates, message injection, and pipeline promotion
- `acceptSpeculation()` (Source: `speculation.ts#L717-800`): lower layer, responsible for overlay writeback, transcript recording, and `timeSavedMs` calculation

Execution steps:

0. **User message injection first**: Immediately displays the user's input in the UI to ensure instant visual feedback (Source: `speculation.ts#L875-876`)

1. **Message sanitization** (`prepareMessagesForInjection`, Source: `speculation.ts#L203-271`):
   - Filters `thinking` and `redacted_thinking` blocks
   - Removes `tool_use`/`tool_result` pairs that did not complete successfully
   - Removes interrupt messages (`INTERRUPT_MESSAGE`)
   - Filters all-whitespace text messages (to avoid API 400 errors)
   - If speculation did not complete, discards the trailing assistant message (models that do not support prefill reject requests ending with an assistant turn)

2. **File state merge**: Merges the file state cache read by speculation into the main conversation to avoid duplicate reads (Source: `speculation.ts#L910-917`)

3. **Overlay writeback**: Copies modified files from the overlay directory back to the main directory

4. **Feedback message injection** (only for `USER_TYPE === 'ant'`, Source: `speculation.ts#L273-308`):
   ```
   [ANT-ONLY] Speculated 3 tool uses · 1,234 tokens · +2.1s saved (5.3s this session)
   ```

5. **Transcript recording**: Writes a `speculation-accept` entry to the JSONL transcript (Source: `speculation.ts#L784-794`) for calculating cumulative time saved in the session

### Speculation Telemetry

Event name: `tengu_speculation` (Source: `speculation.ts#L124-153`)

| Field | Description |
|------|------|
| `speculation_id` | Speculation session UUID (first 8 characters) |
| `outcome` | `accepted` / `aborted` / `error` |
| `duration_ms` | Time spent on speculative execution |
| `suggestion_length` | Length of the suggestion text |
| `tools_executed` | Number of tool calls that successfully returned results (counts `tool_result && !is_error`) |
| `completed` | Whether a boundary was reached (`boundary !== null`) |
| `boundary_type` | `complete` / `bash` / `edit` / `denied_tool` |
| `boundary_tool` | Tool name that triggered the boundary |
| `boundary_detail` | Command/path that triggered the boundary (truncated to the first 200 characters) |
| `message_count` | Total number of speculation messages (only when accepted) |
| `time_saved_ms` | Milliseconds from speculation start to `min(acceptance time, boundary completion time)` (only when accepted) |
| `is_pipelined` | Whether the speculation was generated by the pipeline |

## Source File Index

| File | LOC | Responsibility |
|------|-----|------|
| `services/PromptSuggestion/promptSuggestion.ts` | 524 | Core service: enablement checks, generation, filtering, telemetry |
| `services/PromptSuggestion/speculation.ts` | 992 | Speculative execution: overlay isolation, boundary detection, pipeline |
| `hooks/usePromptSuggestion.ts` | 178 | React Hook: UI state management, acceptance/display/telemetry |
| `components/PromptInput/PromptInput.tsx` | — | Input box component: integrates suggestion display and Enter acceptance |
| `components/PromptInput/useTypeahead.tsx` | — | Tab/→ key acceptance and ghost text rendering |
| `state/AppStateStore.ts` | — | State definitions: `promptSuggestion` + `speculation` + `speculationSessionTimeSavedMs` |
| `query/stopHooks.ts` | — | Entry point: fire-and-forget call in stop hooks |
| `components/Settings/Config.tsx` | — | Toggle switch in the `/config` menu |
