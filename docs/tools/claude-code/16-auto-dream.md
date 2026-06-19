# 16. Auto Dream Memory Consolidation: Developer Reference

> Auto Dream is Claude Code's background memory consolidation system: when the user is idle, it automatically forks a hidden Subagent to organize, deduplicate, and update MEMORY.md. The name comes from the memory consolidation process of human REM sleep.
>
> **Qwen Code comparison**: Qwen Code has no equivalent mechanism. Memory files only grow and never shrink, so long-term use accumulates many outdated, conflicting, or duplicate entries.

## 1. Why Automatic Memory Consolidation Is Needed

### Problem Definition

A Code Agent's memory system (CLAUDE.md / MEMORY.md) is **append-only**: user preferences, project conventions, and work progress are continuously appended. But:

| Problem | Consequence |
|------|------|
| The same fact is recorded 3 times (with different wording) | Wasted tokens and information noise |
| Outdated information is not deleted ("the project uses webpack" after migrating to vite) | The Agent acts on incorrect assumptions |
| Conflicting entries coexist ("use tabs for indentation" + "use spaces for indentation") | Inconsistent Agent behavior |
| MEMORY.md exceeds 200 lines | Too much invalid context is loaded at startup |

### Auto Dream's Solution

When the user is idle (at least 24 hours + 5 sessions later), Claude Code automatically forks a background Agent to perform four-stage memory consolidation:

```
Three-gate trigger conditions (all must be satisfied):
  1. Time since last dream ≥ 24 hours
  2. Sessions since last dream ≥ 5
  3. Consolidation lock acquired (prevents concurrent dreams)
```

## 2. Four-Stage Consolidation Flow

| Stage | Name | Action |
|------|------|------|
| 1 | **Orient** | `ls` the memory directory, read MEMORY.md, browse existing topic files |
| 2 | **Gather Signal** | Collect new information from three sources: daily logs → drifted memories → transcript search |
| 3 | **Consolidate** | Write/update memory files: merge overlapping entries, delete contradictory facts, convert relative dates to absolute dates |
| 4 | **Prune & Index** | Keep MEMORY.md < 200 lines AND ~25KB; delete stale pointers; resolve contradictions between the index and file contents |

Source: `services/autoDream/consolidationPrompt.ts`

### Key Constraints

- The consolidation process takes about **8-10 minutes**
- It runs fully in the background and does not interrupt the user
- It uses a fork Subagent and inherits the current context
- **Mutually exclusive with Kairos**: Kairos mode has its own disk-skill dream mechanism

## 3. Memory Consolidation Compared with Competitors

| Agent | Memory System | Automatic Consolidation | Conflict Handling |
|-------|---------|---------|---------|
| **Claude Code** | MEMORY.md + topic files | ✓ Auto Dream (4 stages) | Delete contradictions and merge overlaps |
| **Gemini CLI** | GEMINI.md | — | — |
| **Qwen Code** | QWEN.md + simple notes | — | — |
| **Copilot CLI** | copilot-instructions.md | — | — |

## 4. Qwen Code Improvement Suggestions

Implement a simplified Auto Dream:
1. At session end, check whether QWEN.md exceeds 200 lines
2. If it does, fork a Subagent to clean it up (merge duplicates, delete stale entries, compress to within 200 lines)
3. A full 4-stage process is not required: starting with "deduplicate + delete stale entries" can solve 80% of the problem
