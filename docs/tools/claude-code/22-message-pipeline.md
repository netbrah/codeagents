# 22. Message and Prompt Pipeline — Developer Reference

> The system prompt is only one part of the model input. The complete input is a **pipeline** assembled from **multiple sources** after normalization, ordering, and injection. This document analyzes Claude Code's full message transformation flow.
>
> **Qwen Code comparison**: Qwen Code's message construction is relatively simple (system prompt + message history + tool Schema). Claude Code's pipeline includes 6 layers of transformation; `<system-reminder>` injection and Prompt Cache partitioning are key missing pieces in Qwen Code.
>
> **Acknowledgment**: Conceptual framework references the s10a chapter of [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code).

## 1. Why the System Prompt Is Not Everything

### Problem Definition

The complete input received by the model is not just "system prompt + user message". It is a combination of multiple sources:

```
Actual input received by the model:

┌─────────────────────────────────────────────┐
│ 1. System prompt (assembled from blocks)    │
│    ├─ Role definition                       │
│    ├─ Behavioral rules                      │
│    ├─ Tool usage guide                      │
│    ├─ Tone and style                        │
│    ├─ Environment info (platform/shell/date)│
│    ├─ Memory (MEMORY.md)                    │
│    └─ MCP instructions                      │
├─────────────────────────────────────────────┤
│ 2. First user message (system-injected)     │
│    └─ <system-reminder> tag                 │
│        ├─ CLAUDE.md project instructions    │
│        ├─ Git status (branch/commit/changes)│
│        └─ Current date                      │
├─────────────────────────────────────────────┤
│ 3. Conversation history                     │
│    ├─ User messages                         │
│    ├─ Assistant replies                     │
│    ├─ Tool calls + results                  │
│    └─ isMeta messages (hidden in UI but     │
│       visible to the model)                 │
├─────────────────────────────────────────────┤
│ 4. Tool Schema (transmitted separately, not  │
│    in the system prompt)                    │
│    ├─ Core tools (always loaded)            │
│    ├─ Deferred tools (loaded after          │
│    │  ToolSearch activation)                │
│    └─ MCP tools (dynamically discovered)    │
└─────────────────────────────────────────────┘
```

## 2. Pipeline Transformation Flow

```
Source                        Transform                 Output
─────                         ─────                     ─────

System prompt paragraphs[] ── paragraph cache + Feature Flag ──→ SystemPromptBlock[]
                                                        │
                     SYSTEM_PROMPT_DYNAMIC_BOUNDARY      │
                     (static/dynamic partition marker)   │
                                                        ▼
                                                  splitSysPromptPrefix()
                                                  (assign Cache Scope)
                                                        │
                                                        ▼
                                                  buildSystemPromptBlocks()
                                                  (add cache_control)
                                                        │
CLAUDE.md ────── loadClaudeMds() ─────→                 │
Git status ──── getGitStatus() ──────→  getUserContext()│
Date ────────────────────────────────→     │            │
                                           ▼            │
                                     prependUserContext()│
                                     (<system-reminder>) │
                                           │            │
Message history ── normalize ── compaction check ──→    │
                                           ▼            ▼
                                    ┌─────────────────────┐
                                    │   API request        │
                                    │  system: Block[]     │
                                    │  messages: Message[] │
                                    │  tools: ToolSpec[]   │
                                    └─────────────────────┘
```

## 3. Key Design: `<system-reminder>` Injection

Claude Code places CLAUDE.md content in the **first user message** (rather than the system prompt), wrapped in a `<system-reminder>` tag:

```typescript
createUserMessage({
  content: `<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
${claudeMdContent}
# currentDate
Today's date is ${date}.

IMPORTANT: this context may or may not be relevant...
</system-reminder>`,
  isMeta: true,  // Not shown in the UI, but visible to the model
})
```

**Why not put it in the system prompt?**
1. The static prefix of the system prompt needs to share Prompt Cache across users; CLAUDE.md is project-specific and would break the cache.
2. User-message cache granularity is finer (org-level) and does not affect the global cache.
3. `isMeta: true` ensures this "injected" message is not shown in the UI.

**Qwen Code comparison**: Qwen Code directly appends QWEN.md content to the system prompt, which breaks Prompt Cache prefix matching. Consider migrating to the `<system-reminder>` injection pattern.

## 4. Message Normalization

Raw messages come from multiple sources (user input, tool results, Hook injections) and use different formats. Normalization ensures all messages satisfy the same structural contract:

| Original Source | Normalization Handling |
|---------|-----------|
| User text input | Wrap as `{role: "user", content: [{type: "text"}]}` |
| Tool result | Wrap as `{role: "user", content: [{type: "tool_result"}]}` |
| Pasted image | Convert to `{type: "image", source: {type: "base64"}}` |
| `<system-reminder>` | Prepended as a user message with `isMeta: true` |
| Compaction summary | Replaces the message segments that were compacted |

## 5. Pipeline Position of Prompt Cache Partitioning

Cache partitioning is not done while constructing the system prompt; it is done in the **final pipeline step** (`buildSystemPromptBlocks`):

```
System prompt paragraphs[]
  │
  ├─ Paragraph 1: role definition      ← static (unchanged)
  ├─ Paragraph 2: behavioral rules     ← static
  ├─ Paragraph 3: tool guide           ← static
  ├─ ── BOUNDARY ──
  ├─ Paragraph 4: environment info     ← dynamic (changes each round)
  ├─ Paragraph 5: memory               ← dynamic (changes across sessions)
  └─ Paragraph 6: MCP instructions     ← dynamic (changes with MCP connections)
        │
        ▼
  splitSysPromptPrefix()
        │
        ├─ Static → cache_control: {scope: "global"}
        └─ Dynamic → cache_control: null (not cached)
```

## 6. Qwen Code Improvement Suggestions

| Improvement | Description | Priority |
|------|------|--------|
| **QWEN.md → system-reminder injection** | Move project instructions from the system prompt to the first user message | P1 |
| **Prompt Cache static/dynamic partitioning** | Put immutable behavioral instructions in the prefix to ensure cache hits | P1 |
| **Explicit message normalization** | Pass messages from all sources through a unified normalize function | P2 |
| **isMeta messages** | Support system-injected messages that are hidden in the UI but visible to the model | P2 |
