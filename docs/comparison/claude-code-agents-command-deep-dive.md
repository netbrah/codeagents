# Claude Code `/agents` Command Deep-Dive

> **Core question**: What is Claude Code's `/agents` slash command? Why is it the best reference implementation for Stage 1.5c daemon-side state CRUD? Which designs can Qwen Code learn from?
>
> Back to the [Qwen Code improvement recommendations overview](./qwen-code-improvement-report.md)
>
> **File history**: This document was originally named `claude-code-agents-view-deep-dive.md` and was renamed to `claude-code-agents-command-deep-dive.md` on 2026-05-16 to avoid colliding with Qwen Code's `agent-view/` multi-tab interactive UI. The two are completely different forms: this document covers a **definition-management UI**, while Qwen's document covers a **runtime interactive multi-tab UI**. See [Qwen Code `agent-view` multi-tab UI Deep-Dive](./qwen-code-agent-view-deep-dive.md).
>
> **Essential clarification**: `/agents` is **not** "real-time subagent status monitoring" (that belongs to [SubAgent Display Deep-Dive](./subagent-display-deep-dive.md)), and it is **not** Qwen's "multi-subagent tabbed conversation window" (see [qwen-code-agent-view-deep-dive.md](./qwen-code-agent-view-deep-dive.md)). It is a complete **subagent definition CRUD UI**: it manages agent configuration files, does not track runtime status, and does not switch conversation context.

## 0. TL;DR

Claude Code `/agents` is a **~3042 LOC subagent definition-management UI**. It uses Ink TUI to render local React components so users can, inside the CLI:

| Operation | Implementation |
|---|---|
| **List** | 7-mode state machine, filtered by source filter (6 sources: built-in / user / project / local / policy / flag / plugin) |
| **View details** | system prompt + tools + model + color + memory + hooks + skills + isolation |
| **Create** | 11-step wizard, supports **AI-generated** agents (a long system prompt calls an LLM to automatically generate identifier / whenToUse / systemPrompt) |
| **Edit / Delete** | inline editor + delete-confirm dialog |

**Key file format**: `.claude/agents/<name>.md` markdown + YAML frontmatter (with 17+ fields).

**Meaning for Qwen Code**: This is a model for **Stage 1.5c daemon-side state CRUD**. Claude Code has already shipped a complete 7-mode state machine for managing subagents; Qwen daemon Stage 1.5c needs to expose equivalent capabilities to remote clients (through HTTP routes rather than local-jsx).

## 1. What `/agents` is not vs what it is

### Three adjacent concepts that are easy to confuse

| Concept | Meaning | Claude Code implementation | Scope of this document |
|---|---|---|---|
| **`/agents` UI** | Subagent definition CRUD | `commands/agents/agents.tsx` -> `<AgentsMenu>` | Yes, this document |
| **SubAgent runtime status display** | Background subagent progress pill / dialog | `LiveAgentPanel` + `CoordinatorTaskPanel` | No; see [subagent-display-deep-dive.md](./subagent-display-deep-dive.md) |
| **`AgentTool`** | Tool called when the main agent spawns a subagent | `tools/AgentTool/` | No; see [fork-subagent-deep-dive.md](./fork-subagent-deep-dive.md) |

This document focuses exclusively on the **`/agents` UI**: the interface for users to manage agent **definition files**. It does not cover runtime status.

### Triggering mechanism

| Trigger | User action |
|---|---|
| Slash command | Enter `/agents` in the TUI |
| Internal | `LocalJSXCommand` type, returning an `<AgentsMenu>` React component rendered locally inside the TUI |
| Wired? | No, **local-jsx, not sent over the wire** (available to Mode A local TUI users; invisible to Mode B remote clients) |

## 2. Source structure (~3042 LOC, 21 files)

```
commands/
├─ agents/
│  ├─ agents.tsx          11 LOC   slash command entrypoint -> returns <AgentsMenu>
│  └─ index.ts            10 LOC   command registration

components/agents/
├─ AgentsMenu.tsx        799 LOC   * 7-mode state-machine controller
├─ AgentsList.tsx        439 LOC   list view (filtered by source)
├─ AgentDetail.tsx       219 LOC   detail view (includes markdown-rendered system prompt)
├─ AgentEditor.tsx       177 LOC   inline editor
├─ AgentNavigationFooter.tsx 25 LOC  navigation footer (keyboard hints)
├─ ToolSelector.tsx      561 LOC   * tool selector (includes merged MCP tool display)
├─ ColorPicker.tsx       111 LOC   agent color selection
├─ ModelSelector.tsx      67 LOC   model selection
├─ agentFileUtils.ts     272 LOC   markdown file read/write (YAML frontmatter)
├─ generateAgent.ts      197 LOC   * AI agent generation
├─ validateAgent.ts      109 LOC   field validation
├─ types.ts               27 LOC   ModeState type definitions
└─ utils.ts               18 LOC   utils

components/agents/new-agent-creation/
├─ CreateAgentWizard.tsx  96 LOC   wizard provider
└─ wizard-steps/         1503 LOC  * 11 wizard steps
   ├─ LocationStep.tsx     79 LOC   choose where to place it (user/project/local)
   ├─ MethodStep.tsx       79 LOC   Generate or Manual
   ├─ GenerateStep.tsx    142 LOC   AI generation (Generate path only)
   ├─ TypeStep.tsx        102 LOC   agent type naming
   ├─ PromptStep.tsx      127 LOC   system prompt editing
   ├─ DescriptionStep.tsx 122 LOC   whenToUse description
   ├─ ToolsStep.tsx        60 LOC   tool selection
   ├─ ModelStep.tsx        51 LOC   model selection
   ├─ ColorStep.tsx        83 LOC   color
   ├─ MemoryStep.tsx      112 LOC   memory scope (conditional GrowthBook gate)
   ├─ ConfirmStep.tsx     377 LOC   * confirmation + file write
   └─ ConfirmStepWrapper.tsx 73 LOC
```

**Total: ~3042 LOC**. Starred items are core components.

## 3. The 7-mode state machine (AgentsMenu.tsx)

```typescript
// components/agents/types.ts
export type ModeState =
  | { mode: 'main-menu' }
  | { mode: 'list-agents'; source: SettingSource | 'all' | 'built-in' }
  | ({ mode: 'agent-menu' } & WithAgent & WithPreviousMode)
  | ({ mode: 'view-agent' } & WithAgent & WithPreviousMode)
  | { mode: 'create-agent' }
  | ({ mode: 'edit-agent' } & WithAgent & WithPreviousMode)
  | ({ mode: 'delete-confirm' } & WithAgent & WithPreviousMode)
```

### State-transition diagram

```
                 main-menu
                     ↓ select source
               list-agents (filtered by source)
                     ↓ select agent | create-new
         ┌───────────┼────────────────────┐
         ↓           ↓                    ↓
    agent-menu  create-agent       (cancel -> list-agents)
         ↓           ↓
    ┌────┼────┐   wizard 11-step
    ↓    ↓    ↓
 view-  edit- delete-confirm
 agent  agent      ↓
                  yes -> write file -> list-agents
                  no  -> previousMode
```

### Five core keyboard interactions

| Key | Behavior |
|---|---|
| `Enter` | Select |
| `Esc` | Return to `previousMode` |
| `j/k` or `↑/↓` | Move focus |
| `Ctrl+C` (x 2) | Exit dialog (once to confirm, twice to force quit) |
| `Tab` | Switch source filter |

## 4. Agent definition file format (key standard)

`.claude/agents/<name>.md` is **markdown + YAML frontmatter**:

```markdown
---
name: code-reviewer
description: "Reviews code for quality and security. Use when user asks for review after writing code.
              <example>User: 'review my recent PR'</example>"
tools: Bash, Edit, Read, Glob, Grep
disallowedTools: WriteFile
model: claude-opus-4-7
effort: high
permissionMode: ask
color: blue
memory: project
mcpServers:
  - filesystem
  - github
hooks:
  pre-prompt: |
    echo "Reviewing branch $(git branch --show-current)"
skills:
  - code-review
  - security-audit
maxTurns: 20
background: false
isolation: worktree   # ant-only: worktree | remote
initialPrompt: "Start by listing recent commits..."
criticalSystemReminder_EXPERIMENTAL: "Always verify with tests."
omitClaudeMd: false   # saves ~5-15 Gtok/week for read-only agents
requiredMcpServers:
  - github
---

You are an elite code reviewer specialized in...
```

### Complete field table (17+ fields)

| Field | Type | Required | Meaning |
|---|---|:---:|---|
| `name` (= `agentType`) | string | Yes | agent identifier (lowercase + hyphen, 2-4 words) |
| `description` (= `whenToUse`) | string | Yes | When to use this agent (includes `<example>` tags) |
| `tools` | `string[]` | No | Allowed tools (default `*`, all tools) |
| `disallowedTools` | `string[]` | No | Disallowed tools |
| `model` | string | No | model name (`inherit` means follow the main agent) |
| `effort` | enum / int | No | thinking effort level |
| `permissionMode` | enum | No | `ask` / `allow` / `deny` / `yolo` |
| `mcpServers` | string[] / record | No | Specific MCP servers (name references or inline definitions) |
| `hooks` | record | No | session-scoped hooks |
| `skills` | string[] | No | Preloaded skill names |
| `color` | enum | No | UI distinguishing color |
| `memory` | enum | No | `user` / `project` / `local` -- persistent memory scope |
| `maxTurns` | int | No | Maximum agentic turns |
| `background` | bool | No | Always a background task |
| `isolation` | enum | No | `worktree` / `remote` (ant-only) |
| `initialPrompt` | string | No | Automatically prepended to the first turn |
| `criticalSystemReminder_EXPERIMENTAL` | string | No | **Re-injected on every user turn** to ensure critical constraints are not forgotten |
| `omitClaudeMd` | bool | No | **Saves CLAUDE.md context**; read-only agents (Explore/Plan) enable it to save ~5-15 Gtok/week |
| `requiredMcpServers` | string[] | No | Required MCP server patterns |

## 5. Six settings sources (placement strategy)

| Source | Path | Sharing scope | git commit? | Editable in UI? |
|---|---|---|---|:---:|
| `built-in` | Embedded in code | All users | -- | No (read-only) |
| `userSettings` | `~/.claude/agents/*.md` | Current user | No | Yes |
| `projectSettings` | `<cwd>/.claude/agents/*.md` | All project collaborators | Yes, can commit | Yes |
| `localSettings` | `<cwd>/.claude/agents/*.md` (gitignore) | Personal project-level | No | Yes |
| `policySettings` | managed path (enterprise) | Organization-enforced | -- | No (read-only) |
| `flagSettings` | Injected at runtime by CLI flag | This launch | -- | No |
| `plugin` | Provided by plugins (dynamic) | Users with the plugin installed | -- | No |

`AgentsList.tsx` lets users filter agents from different sources via the `source` filter. Agents with the same name are resolved by priority (flagSettings > projectSettings > localSettings > userSettings > built-in).

## 6. The 11-step create wizard

```typescript
// components/agents/new-agent-creation/CreateAgentWizard.tsx:69
const steps = [
  LocationStep,    // 0  choose where to place it
  MethodStep,      // 1  Generate / Manual
  GenerateStep,    // 2  AI generation (Generate only)
  TypeStep,        // 3  agent type naming
  PromptStep,      // 4  system prompt
  DescriptionStep, // 5  whenToUse
  ToolsStep,       // 6  tool selection
  ModelStep,       // 7  model
  ColorStep,       // 8  color
  ...(isAutoMemoryEnabled() ? [MemoryStep] : []),  // 9  GrowthBook conditional gate
  ConfirmStep,     // 10 confirmation + file write
]
```

### Key wizard technical points

1. **`isAutoMemoryEnabled()` GrowthBook gate**: MemoryStep is a progressive rollout and uses a feature flag to control user exposure.
2. **Generate / Manual branch**: the Generate path goes through `GenerateStep` (AI generation pre-fills later steps); the Manual path skips GenerateStep.
3. **Data transfer**: `WizardProvider<AgentWizardData>` maintains wizard data; each step has the shape `({ data, setData, next, prev }) => ReactNode`.
4. **Cancel**: pressing `Esc` in any step triggers `onCancel`, returns to main-menu, and writes no file.

## 7. AI-generated agents (standout feature)

`components/agents/generateAgent.ts` calls the LLM with a ~1500-token system prompt to generate a complete agent configuration.

### System prompt excerpt

```
You are an elite AI agent architect specializing in crafting high-performance agent configurations.

When a user describes what they want an agent to do, you will:

1. **Extract Core Intent**: identify fundamental purpose / key responsibilities / success criteria
2. **Design Expert Persona**: create a compelling expert identity
3. **Architect Comprehensive Instructions**: behavioral boundaries + methodology + edge-case handling
4. **Optimize for Performance**: decision framework + self-checking + escalation
5. **Create Identifier**: lowercase + hyphen + 2-4 words + avoid "helper" / "assistant"
6. **Example agent descriptions**: add <example> tags to whenToUse

Output JSON:
{
  "identifier": "code-reviewer",
  "whenToUse": "...",
  "systemPrompt": "You are an elite code reviewer..."
}
```

### Implementation flow

```ts
// generateAgent.ts
const response = await queryModelWithoutStreaming({
  systemPrompt: asSystemPrompt(AGENT_CREATION_SYSTEM_PROMPT),
  messages: prependUserContext([createUserMessage(userDescription)]),
  toolPermissionContext: getEmptyToolPermissionContext(),
})

const parsed = jsonParse(response.content[0].text)
// -> { identifier, whenToUse, systemPrompt }

// Inject into wizard data and pre-fill later TypeStep / PromptStep / DescriptionStep
```

**Key design points**:
- Use **`queryModelWithoutStreaming`** rather than the main agent loop: an independent sub-query that does not pollute the main conversation.
- **`prependUserContext`** injects CLAUDE.md / project context, so generated agents automatically align with project conventions.
- **`toolPermissionContext: empty`**: the LLM cannot call tools and can only generate text.
- **`asSystemPrompt`** typed wrapper prevents misuse.
- Output JSON uses a strict schema (`{identifier, whenToUse, systemPrompt}`).

### `logEvent` analytics instrumentation

```ts
logEvent({
  eventName: 'tengu_agent_generated',
  fields: { /* AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS */ },
})
```

Note the type name `I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`: Anthropic uses type-level checks to force confirmation that telemetry does not contain sensitive data.

## 8. ToolSelector: merged MCP tool display

`ToolSelector.tsx` (561 LOC) is the largest component inside the wizard. Key capabilities:

1. **Merged display of built-in + MCP tools** using the `useMergedTools(tools, mcpTools, toolPermissionContext)` hook.
2. **Wildcard handling**: `*` means all tools are allowed; the UI displays "All tools".
3. **invalidTools warnings**: if an agent references a nonexistent tool, render `figures.warning` plus a yellow hint.
4. **Nested namespaces**: MCP tool names like `mcp__server__tool_name` are grouped for display.
5. **resolveAgentTools(agent, tools, false)** resolves the final allowed set.

## 9. Comparison with Qwen Code

| Dimension | Claude Code `/agents` | Current Qwen Code |
|---|---|---|
| Slash command | `/agents` local-jsx | `/agents` slash command (simple implementation) |
| Agent file format | markdown + YAML frontmatter | Same style (already used by PR#3873 subagent config isolation) |
| Wizard steps | 11 steps | Simpler (no wizard) |
| AI-generated agents | Yes, long system prompt + LLM | No **equivalent capability** |
| 6 source layers | built-in / user / project / local / policy / flag / plugin | Simpler |
| Tool selection UI | Includes merged MCP tools + invalid warnings | Simpler |
| `isolation: worktree/remote` | Yes, ant-only | PR#4073 Worktree* tools OPEN ([builtin-tools section 3.10](./claude-code-vs-qwen-code-builtin-tools.md), P1 learning item) |
| `omitClaudeMd` optimization | Yes, saves ~5-15 Gtok/week | No |
| `criticalSystemReminder_EXPERIMENTAL` | Yes, re-injected on every user turn | No |
| `memory` scope (user/project/local persistence) | Yes + automatic snapshot restore (GrowthBook gated) | No |
| `effort` thinking level | Yes, enum / int | Partial (see [reasoning-effort-deep-dive.md](./reasoning-effort-deep-dive.md)) |
| Per-agent `hooks` | Yes, session-scoped | PR#4072 Agent hooks via headless subagent OPEN |
| Overall LOC | ~3042 | Less |

## 10. Lessons for Qwen Code daemon design

### 10.1 A model for Stage 1.5c daemon-side state CRUD

Claude Code `/agents` is a typical **local-jsx, not-on-the-wire** dialog. This fully matches [daemon-design section 04, part 2, "wire only carries agent-user conversation; TUI mutations do not go over the wire"](./qwen-code-daemon-design/04-deployment-and-client.md). **Stage 1.5c needs to expose equivalent capabilities to remote clients**:

```
Current (Stage 1):
  Mode A local TUI user -> /agents -> local render inside Ink TUI Yes
  Mode B remote client -> ??? -> cannot see daemon-side agent state No

Stage 1.5c adds wire routes:
  GET    /workspace/agents              -- list agents (with source filter)
  GET    /workspace/agents/:agentType   -- agent details
  POST   /workspace/agents              -- create agent
  PUT    /workspace/agents/:agentType   -- edit agent
  DELETE /workspace/agents/:agentType   -- delete agent
  POST   /workspace/agents/generate     -- AI-generate agent * (inspired by Claude)
  GET    /workspace/agents/sources      -- list all source paths
```

See [daemon-design section 06, part 3, Stage 1.5c](./qwen-code-daemon-design/06-roadmap.md).

### 10.2 File format + source layering can be adopted directly

Claude's markdown + YAML frontmatter is a **de facto standard** (OpenAI Codex / Cursor / Cline all use similar formats). **Qwen should remain compatible** for interoperability and user migration across tools. The six source layers (built-in / user / project / local / policy / flag / plugin) are also a reasonable abstraction and are recommended for direct porting.

### 10.3 AI-generated agents are a missing standout feature

Current Qwen Code has no `/agents generate` capability. This can be a **Stage 2+ candidate**:
- Similar to `/goal`, use the LLM-as-judge idea (see [`goal-command-deep-dive.md`](./goal-command-deep-dive.md)).
- But the task is "generate an agent definition", not "judge goal completion".
- Implementation: `queryModelWithoutStreaming` + long system prompt + JSON output schema validation.
- UI: call it from the "Generate" branch inside the wizard.

### 10.4 `omitClaudeMd` optimization is worth porting

Claude reports that enabling `omitClaudeMd` for read-only agents such as Explore / Plan saves **~5-15 Gtok/week**. At Qwen Code's user scale, an equivalent optimization could save substantial token cost. This is a **P1 learning item**.

Implementation:
1. Add an `omitClaudeMd?: boolean` field to `AgentDefinition`.
2. In `newSessionConfig(cwd, ..., agentType)`, check this field; if true, skip CLAUDE.md injection in `prependUserContext`.
3. Enable it by default for built-in read-only agents such as `Explore` / `Plan`.
4. Add an emergency switch if needed (such as Claude's `tengu_slim_subagent_claudemd`).

### 10.5 `criticalSystemReminder_EXPERIMENTAL`: re-injected every turn

The `criticalSystemReminder_EXPERIMENTAL` field in an agent definition is **re-injected on every user turn** into the context, ensuring critical constraints (such as "Always verify with tests") are not forgotten as context rolls forward.

Implementation:
1. Add a `criticalSystemReminder?: string` field to `AgentDefinition`.
2. Each time the agent loop constructs a user-turn message, prepend `<system-reminder>${criticalSystemReminder}</system-reminder>`.
3. Combine it with Qwen's existing `system-reminder` mechanism (see [improvement-report item-26](./qwen-code-improvement-report.md)).

### 10.6 Coordinate `isolation: 'worktree'` with PR#4073

Claude has already shipped this capability (ant-only `worktree | remote`). Qwen Code PR#4073 Worktree* tools is OPEN. When an agent definition explicitly declares `isolation: 'worktree'`, the agent should spawn and run in an independent worktree (see [claude-code-vs-qwen-code-builtin-tools.md section 3.10](./claude-code-vs-qwen-code-builtin-tools.md), P1 learning item).

### 10.7 GrowthBook conditional step (progressive rollout model)

`MemoryStep` uses a GrowthBook gate (`isAutoMemoryEnabled()`) to control exposure. This is Anthropic's model for progressive rollout with feature flags before a new feature reaches GA. Qwen Code can adopt this pattern for risky features such as AI-generated agents or `omitClaudeMd`.

## 11. Summary of potential P0 / P1 learning items

| Priority | Learning item | Effort | Key benefit |
|:------:|---|---|---|
| **P0** | `/agents` daemon-side state CRUD (Stage 1.5c) | ~0.5d / route x 7 routes ~= ~3-5d | Remote clients get Mode A equivalence (solves thin-shell limitation) |
| **P0** | `omitClaudeMd` for read-only agents | ~1d | Saves ~5-15 Gtok/week token cost |
| **P1** | AI-generated agents (`/agents generate`) | ~2-3d | Better UX + lower entry barrier |
| **P1** | `criticalSystemReminder_EXPERIMENTAL` per agent | ~1d | Prevents critical constraints from being forgotten in long sessions |
| **P1** | `isolation: 'worktree'` agent definition field (depends on PR#4073) | ~0.5d (add field) + PR#4073 | Isolates agent workspaces |
| **P2** | Six source layers (including plugin / policy) | ~2-3d | Enterprise deployment + plugin ecosystem |
| **P2** | 11-step wizard UI | ~3-5d | Lowers barrier for creating agents |
| **P2** | ToolSelector merged MCP display | ~1-2d | UI completeness |

## 12. Related documents

- [Agent Creation Wizard Deep-Dive](./interactive-agent-creation-deep-dive.md) -- **deeper analysis of the wizard part in section 6 of this document** (an actionable blueprint for Qwen Code to learn from the wizard UX)
- [SubAgent Display Deep-Dive](./subagent-display-deep-dive.md) -- **runtime status** display (not definition management)
- [Fork Subagent Deep-Dive](./fork-subagent-deep-dive.md) -- Subagent launch mechanism
- [Agent Tool Access Control Deep-Dive](./agent-tool-access-control-deep-dive.md) -- tool permission control
- [Agent Memory Persistence Deep-Dive](./agent-memory-persistence-deep-dive.md) -- the `memory` scope field
- [Reasoning Effort Deep-Dive](./reasoning-effort-deep-dive.md) -- comparison of the `effort` field
- [Claude Code vs Qwen Code built-in tools comparison](./claude-code-vs-qwen-code-builtin-tools.md) -- Worktree* P1 learning item
- [Qwen Code Daemon architecture design section 04](./qwen-code-daemon-design/04-deployment-and-client.md) -- Stage 1.5c daemon-side state CRUD design
- [Qwen Code Daemon architecture design section 06](./qwen-code-daemon-design/06-roadmap.md) -- Stage 1.5c roadmap

---

> **Data source**: decompiled source for claude-code v2.1.139 (`/root/git/claude-code-leaked/components/agents/`); analyzed on 2026-05-15. All LOC figures are based on the sourcemap-simplified version; actual release line counts may differ due to compression / optimization.
