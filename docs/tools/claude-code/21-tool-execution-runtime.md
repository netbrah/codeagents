# 21. Tool Execution Runtime — Developer Reference

> When the model returns 5 tool calls at once, which can run in parallel? Which must be serial? How should concurrent results be ordered? How should long-running tools report progress? Can tools modify shared context? These questions belong to the **tool execution runtime**, not tool registration.
>
> **Qwen Code comparison**: Qwen Code's `CoreToolScheduler` already implements parallel Agent tools + serial execution for other tools. However, it lacks progress messages, context-modifier merging, and concurrency-safety classification.
>
> **Acknowledgment**: The conceptual framework references the s02b chapter of [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code).

## 1. Tool Registration vs Tool Execution Runtime

| Layer | Concern | Typical code |
|---|-----------|---------|
| **Tool registration** (covered in 04-tools.md) | Schema definitions, name mapping, permission declarations | `ToolSpec`, `ToolDispatchMap` |
| **Tool execution runtime** (this article) | Concurrent scheduling, progress reporting, result ordering, context merging | `ToolExecutionBatch`, `TrackedTool` |

## 2. Core Problems

### 2.1 Concurrency-Safety Classification

Not every tool can run in parallel; explicit classification is required:

| Classification | Meaning | Typical tools |
|------|------|---------|
| **Concurrency-safe** | Can run in parallel with tools of the same class without corrupting shared state | `read_file`, `grep`, `glob`, read-only MCP |
| **Not concurrency-safe** | Modifies shared state and must run serially | `write_file`, `edit`, tools that modify application state |
| **Context modifier** | Not only returns a result, but also modifies the execution environment of later tools | `cd` (changes working directory), permission tools |

### 2.2 Concurrent Execution Model

Claude Code's `StreamingToolExecutor` partitions a batch of tool calls and executes them concurrently:

```
Model returns: [read_file(a.ts), read_file(b.ts), edit(c.ts), grep("TODO")]
                                                    │
                                                    ▼
                                          Concurrency-safe partition
                                                    │
                                    ┌───────────────┼───────────┐
                                    │               │           │
                              ┌─────▼─────┐   ┌────▼────┐  ┌──▼──┐
                              │ Parallel  │   │ Serial  │  │Par.2│
                              │ batch 1   │   │ batch   │  │grep │
                              │ read(a.ts)│   │edit(c.ts)│ └──┬──┘
                              │ read(b.ts)│   └────┬────┘     │
                              └─────┬─────┘        │          │
                                    │               │          │
                                    └───────────────┼──────────┘
                                                    │
                                          Result ordering (original order)
                                                    │
                                          Context modifier merging
                                                    │
                                          Return to main loop
```

### 2.3 Gemini CLI's Wave-Based Scheduler

Gemini CLI (Qwen Code's upstream) introduced `scheduler.ts` in v0.35, using wave-based scheduling:

```
[read, read, write, read]
  → Wave 1: [read, read] runs concurrently
  → Wave 2: [write] runs serially
  → Wave 3: [read] runs
```

Qwen Code has not backported this feature. See [backport report #40](../comparison/qwen-code-gemini-upstream-report-details.md#item-40).

### 2.4 Progress Messages

Long-running tools (such as `npm install` and `git clone`) should report progress during execution instead of leaving the user facing silence:

```
Tool starts → "Installing dependencies..."
            → "Installed 42/100 packages..."
            → "Running postinstall scripts..."
Tool completes → return result
```

### 2.5 Result Ordering

Tools executed in parallel may complete in any order. But the results returned to the model should preserve the **original call order**; otherwise the model may become confused.

## 3. Improvement Directions for Qwen Code

| Improvement | Current status | Recommendation | Priority |
|------|---------|------|--------|
| Concurrency-safety classification | Agent tools run in parallel; others run serially | Mark all tools with `concurrencySafe: boolean` | P2 |
| Wave-based scheduling | Not backported | Copy `scheduler.ts` from upstream | P2 |
| Progress messages | None | Emit progress events when tool execution exceeds 3 seconds | P2 |
| Context modifier merging | None | Identify tools that modify CWD/environment variables and run them serially | P3 |
