# 13. System Prompt Construction -- Developer Reference

> Claude Code's system prompt is not a static block of text. It is a **runtime assembly system** composed of 20+ dynamic sections, Feature Flag conditional branches, and Prompt Cache partition markers. Understanding this mechanism is essential for optimizing Qwen Code's prompt efficiency.
>
> **Qwen Code benchmark**: Qwen Code's system prompt is relatively simple (`prompts.ts` ~1000 lines vs Claude Code ~56,000 lines). Claude Code's Prompt Cache partitioning, dynamic section caching, and `<system-reminder>` injection pattern are the main references.

## I. Why System Prompt Construction Is So Complex

### Problem Definition

The system prompt accounts for **20-40%** of the tokens in each Code Agent API call. A typical system prompt contains:

| Component | Size | Change frequency |
|---------|------|---------|
| Behavioral instructions (role/rules/tone) | ~3K tokens | Never changes |
| Tool Schema (42 tools) | ~10K tokens | Changes when tools are dynamically loaded |
| CLAUDE.md project instructions | ~2K tokens | Changes when switching projects |
| Git context (branch/status/commits) | ~500 tokens | Changes every turn |
| Memory (MEMORY.md) | ~1K tokens | Changes across sessions |
| Environment information (platform/shell/date) | ~200 tokens | Changes every turn |

**Core tension**: Most of the system prompt **does not change** (behavioral instructions and Tool Schema), but Anthropic API's Prompt Cache only hits when the prefix matches exactly. Once the Git status changes (different prefix), **the entire system prompt cache is invalidated**.

### Claude Code's Solution: Static/Dynamic Partitioning

```
┌─────────────────────────────────────────────┐
│  Static prefix (Cache Scope: global)        │
│  Behavioral instructions + tool descriptions + general rules │
│  ────── Shared across organizations; all users hit the same cache ──── │
├─ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ────────────┤
│  Dynamic suffix (Cache Scope: org or uncached) │
│  Git status + CLAUDE.md + memory + environment + language │
│  ────── May change every turn; does not affect static-prefix cache ──── │
└─────────────────────────────────────────────┘
```

**Effect**: The cache for the static prefix (~13K tokens) is **never broken by dynamic content**. API cost is reduced by 50-80% (cached tokens cost 1/10 of the normal price).

## II. Construction Flow

### 2.1 Main Entry Point

Source: `constants/prompts.ts`, core function `getSystemPrompt()`

```
getSystemPrompt(tools, model, additionalDirs, mcpClients)
  │
  ├─ Static sections (cached until /clear or /compact)
  │   ├─ Role definition ("You are Claude Code...")
  │   ├─ Behavioral rules (task execution, security, code style)
  │   ├─ Tool usage guide
  │   ├─ Tone and style
  │   └─ Output efficiency
  │
  ├─ ── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ──
  │
  ├─ Dynamic sections (may change every turn)
  │   ├─ Session-specific guidance
  │   ├─ Memory prompt (MEMORY.md)
  │   ├─ Environment information (platform/shell/date)
  │   ├─ Language settings
  │   ├─ Output style
  │   ├─ MCP instructions
  │   ├─ Scratchpad
  │   ├─ Function result clearing
  │   └─ Token budget
  │
  └─ Returns string[]
```

### 2.2 Section Caching Mechanism

Source: `constants/systemPromptSections.ts`

Two section types:

| Type | Cache | Use case |
|------|------|---------|
| `systemPromptSection(name, compute)` | Cached until `/clear` or `/compact` | Most sections |
| `DANGEROUS_uncachedSystemPromptSection(name, compute, reason)` | **Recomputed every turn**, may break cache | Highly volatile content such as Git status |

**Developer takeaway**: Qwen Code currently has no section-level caching. Every API call rebuilds the full system prompt. Introducing section caching (`useMemo` semantics) can reduce CPU and GC overhead during prompt construction.

### 2.3 CLAUDE.md Injection Method

**Key design**: CLAUDE.md content is **not in the system prompt**. Instead, it is injected as the **first user message**, wrapped in `<system-reminder>` tags:

```typescript
// api.ts: prependUserContext()
createUserMessage({
  content: `<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
${claudeMdContent}
# currentDate
Today's date is ${date}.

IMPORTANT: this context may or may not be relevant...
</system-reminder>`,
  isMeta: true,  // Not shown in the UI
})
```

**Why not put it in the system prompt?**
1. The static prefix of the system prompt needs shared cross-user caching; CLAUDE.md is project-specific and would break the cache
2. User messages have finer cache granularity (org-level), so they do not affect the global cache
3. `isMeta: true` ensures this message is not displayed in the conversation UI

### 2.4 Git Context Injection

Source: `context.ts`, function `getGitStatus()`

```typescript
// Run five git commands in parallel:
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow('git', ['--no-optional-locks', 'status', '--short']),
  execFileNoThrow('git', ['--no-optional-locks', 'log', '--oneline', '-n', '5']),
  execFileNoThrow('git', ['config', 'user.name']),
])
```

**Injection location**: appended to the end of the system prompt (the dynamic region) via `appendSystemContext()`.

**Reason for `--no-optional-locks`**: prevents `git status` from acquiring repository locks (avoids blocking other git operations).

### 2.5 Feature Flag Conditional Sections

```typescript
// prompts.ts - compile-time condition (build-time DCE)
...(feature('PROACTIVE') || feature('KAIROS')
  ? [getProactiveSection()]
  : [])

// Runtime condition
...(process.env.USER_TYPE === 'ant'
  ? [getAntModelOverrideSection()]
  : [])
```

Twenty-two Feature Flags control different system-prompt sections. In external builds, Kairos/Proactive-related prompt sections are completely removed.

## III. Prompt Cache Optimization Details

### 3.1 Three Cache Modes

Source: `utils/api.ts`, function `splitSysPromptPrefix()`

| Mode | Usage condition | Static prefix cache | Dynamic content cache |
|------|---------|------------|------------|
| Global Cache | 1P (direct Anthropic API access) | `scope: 'global'` (shared across organizations) | `scope: null` (uncached) |
| Org Cache | When MCP tools are present | `scope: 'org'` (organization-level) | `scope: 'org'` |
| Default Org | 3P Provider | `scope: 'org'` | `scope: 'org'` |

### 3.2 Tool Schema Caching Strategy

Tool Schema is constructed through `toolToAPISchema()` and supports `cache_control` markers:

- Core tool Schemas include `cache_control` (high cache hit rate)
- Dynamically loaded tools (activated by ToolSearch) do not include `cache_control` (to avoid polluting the cache)
- MCP tool Schemas are not sent in `defer_loading` mode

### 3.3 Practical Impact of Cache Hit Rate

```
Token cost composition for each API call:
┌──────────────────────────┬─────────┬──────────┐
│ Component                │ Token   │ Cache status │
├──────────────────────────┼─────────┼──────────┤
│ System prompt (static prefix) │ ~13,000 │ ✓ global cache │
│ System prompt (dynamic suffix) │ ~2,000  │ × uncached     │
│ Tool Schema              │ ~8,000  │ ✓ org cache    │
│ CLAUDE.md (user message) │ ~2,000  │ ✓ org cache    │
│ Conversation history     │ varies  │ partially cached │
├──────────────────────────┼─────────┼──────────┤
│ Cache-hit tokens         │ ~23,000 │ 1/10 price     │
│ Uncached tokens          │ ~2,000  │ full price     │
└──────────────────────────┴─────────┴──────────┘
```

## IV. Competitor System Prompt Comparison

| Agent | System prompt construction | Prompt Cache | CLAUDE.md equivalent |
|-------|-------------|-------------|-----------------|
| **Claude Code** | Dynamic assembly (~56K lines of code) | Static/dynamic partitioning + global/org cache | CLAUDE.md (user-message injection) |
| **Gemini CLI** | `PromptProvider.getCoreSystemPrompt()` | No explicit Prompt Cache management | GEMINI.md |
| **Qwen Code** | `prompts.ts` (~1K lines) | Basic cache | QWEN.md |
| **Copilot CLI** | agent YAML definition | Relies on API-side cache | copilot-instructions.md |

## V. Qwen Code Improvement Recommendations

### P0: Static/Dynamic Partitioning for the System Prompt

Put stable behavioral instructions and Tool Schema in the prefix, and volatile content such as Git status in the suffix. Ensure the byte sequence of the prefix is stable to maximize the cache hit rate.

### P1: Section-Level Caching

Use `useMemo`-style caching for sections that change infrequently (Tool Schema and behavioral rules), avoiding reserialization every turn.

### P1: CLAUDE.md -> QWEN.md User-Message Injection

Move QWEN.md content from the system prompt into the first user message (`<system-reminder>` tag), avoiding project-specific content that breaks the system prompt cache.

### P2: Parallel Git Context Fetching

Follow Claude Code's pattern of using `Promise.all()` to run five git commands in parallel, reducing context collection latency.

## VI. Supplement: System Prompt Differences Between CLI and Agent SDK

> Reference: SDK vs CLI system prompt comparison report from [claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)

The system prompts sent to the API by Claude Code CLI and Agent SDK are **completely different**:

| Dimension | Claude CLI (Claude Code) | Agent SDK (default) | Agent SDK (`claude_code` preset) |
|------|-------------------------|------------------|----------------------------------|
| Base prompt | ~269 tokens (modular) | Minimal prompt | Reuses the CLI prompt |
| Tool definitions | 18+ built-in tools | User-defined | Reuses the CLI tools |
| Conditional loading | 110+ system-prompt strings loaded by feature conditions | Unconditional loading | Partial conditional loading |
| Security review | ~2,610-token extended security instructions (conditional) | None | Present |
| Project context | CLAUDE.md + settings + hooks | None | User-injected |

**Implication for Qwen Code**: If Qwen Code provides an SDK mode in the future (such as `@qwen-code/sdk`), it needs to decide whether to reuse the full CLI system prompt or provide a reduced version. Claude Code's experience is: the SDK is minimal by default, but provides a `claude_code` preset so users can choose the full mode.
