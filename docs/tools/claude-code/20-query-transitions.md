# 20. Query State Transition Model — Developer Reference

> Why is an Agent's core loop not a simple "loop until end_turn"? Because a single query can continue to the next round for **6 completely different reasons**, and each reason requires different handling. This article analyzes Claude Code's query transition model.
>
> **Qwen Code comparison**: Qwen Code's loop has similar logic (tool completion → continue, token truncation → continue), but it has no explicit TransitionReason type, which makes logs hard to read and tests imprecise.
>
> **Acknowledgment**: The conceptual framework references the s00c chapter of [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code).

## 1. Why Explicit Transition Reasons Are Needed

### Problem Definition

A query (user input → Agent processing → output) may span multiple turns. After each turn ends, the system must decide whether to continue.

But "continue" is not one reason; it is **a family of reasons**:

| Transition reason | Meaning | Handling |
|---------|------|---------|
| `tool_result` | Tool execution has completed and the model needs the result | Inject the result into the messages and continue reasoning |
| `max_tokens` | The output token limit was reached and the response was truncated | Ask the model to continue the unfinished output |
| `compaction` | The context was compacted and needs realignment | Inject the compacted summary and retry the current round |
| `retry` | The transport layer failed; retry after backoff | Resend the same request (possibly with a different model) |
| `stop_hook` | A Stop Hook says "it should not end yet" | Inject Hook feedback and continue |
| `budget` | The budget strategy allows continuation | Continue normally |

If all of these are collapsed into `continue`, three problems quickly get worse:

1. **Unreadable logs**: You cannot tell whether continuation happened "after tool completion" or "after retry".
2. **Imprecise tests**: You cannot assert that "this continuation happened because of compaction".
3. **Blurred developer mental model**: During debugging, you do not know why the loop did not stop.

### Claude Code's Solution

Each cross-turn transition carries an explicit `TransitionReason`:

```
Turn N ends
  │
  ├─ Evaluate transition reason
  │     ├─ tool_use_result → TransitionReason.TOOL_RESULT
  │     ├─ max_tokens hit → TransitionReason.MAX_TOKENS
  │     ├─ compact triggered → TransitionReason.COMPACTION
  │     ├─ transport error → TransitionReason.RETRY
  │     ├─ stop hook reject → TransitionReason.STOP_HOOK
  │     └─ budget allows → TransitionReason.BUDGET
  │
  └─ Turn N+1 adjusts behavior based on the reason
        ├─ TOOL_RESULT → inject tool result into messages
        ├─ MAX_TOKENS → ask the model to continue writing
        ├─ COMPACTION → inject summary + rebuild system prompt
        ├─ RETRY → backoff delay + possible model downgrade
        └─ STOP_HOOK → inject Hook feedback
```

## 2. State Transition Diagram

```
                    ┌──────────────────────────┐
                    │     IDLE (waiting input)  │
                    └────────────┬─────────────┘
                                 │ User input
                                 ▼
                    ┌──────────────────────────┐
                    │   QUERYING (API request)  │◄──────────────┐
                    └────────────┬─────────────┘               │
                                 │ Streaming response           │
                                 ▼                             │
                    ┌──────────────────────────┐               │
                    │  PROCESSING (response)    │               │
                    └────────────┬─────────────┘               │
                                 │                             │
                    ┌────────────┼─────────────┐               │
                    │            │             │               │
                    ▼            ▼             ▼               │
              ┌──────────┐ ┌─────────┐ ┌───────────┐          │
              │ end_turn │ │tool_use │ │max_tokens │          │
              │ → IDLE   │ │→ EXEC   │ │→ CONTINUE │          │
              └──────────┘ └────┬────┘ └─────┬─────┘          │
                                │             │                │
                                ▼             │                │
                    ┌──────────────────┐      │                │
                    │ TOOL_EXECUTING   │      │                │
                    │ (perm→Hook→exec) │      │                │
                    └────────┬─────────┘      │                │
                             │ Complete       │                │
                             ▼                ▼                │
                    ┌──────────────────────────────┐           │
                    │ TRANSITION                    │           │
                    │ reason = TOOL_RESULT          │───────────┘
                    │       | MAX_TOKENS            │
                    │       | COMPACTION            │
                    │       | RETRY                 │
                    │       | STOP_HOOK             │
                    └──────────────────────────────┘
```

## 3. Improvement Directions for Qwen Code

Qwen Code has similar loop logic in `CoreToolScheduler` and `client.ts`, but transition reasons are implicit (determined through if-else branches) and there is no explicit `TransitionReason` type.

**Recommendation**:

```typescript
// Add a TransitionReason enum
enum TransitionReason {
  TOOL_RESULT = 'tool_result',
  MAX_TOKENS = 'max_tokens',
  COMPACTION = 'compaction',
  RETRY = 'retry',
  STOP_HOOK = 'stop_hook',
  BUDGET = 'budget',
}

// Record the reason on every cross-turn transition
interface QueryTransition {
  reason: TransitionReason;
  metadata: Record<string, unknown>; // Reason-specific data
  timestamp: number;
}
```

**Benefits**: log readability + test precision + debugging efficiency. Implementation cost is very low (~50 lines for the enum + logging changes).
