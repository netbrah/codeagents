# 7. Session, Memory, and Context Management—Developer Reference

> One of Claude Code's core engineering advantages: 5-layer incremental compaction, layered CLAUDE.md memory, Auto Dream automatic organization, Team Memory synchronization, and crash recovery. This is the area where Qwen Code and Claude Code differ the most (~73,000 lines of code).
>
> **Qwen Code comparison**: context compaction (only a single 70% manual compaction vs 5 automatic layers), memory system (simple notes vs CLAUDE.md + Auto Dream), crash recovery (none vs 3 detection types + synthetic continuation)

## Why Context Management Is a Core Competency for Code Agents

### Problem Definition

Code Agent sessions are **10-100x longer** than chatbot conversations—a refactoring task may last 200+ turns and consume 500K+ tokens. The context window (even a 1-million-token window) eventually fills up. What happens then determines whether the Agent remains usable:

| Strategy | Approach | Consequence |
|------|------|------|
| No compaction | Error and terminate when context is full | Long tasks cannot complete |
| Single compaction (current Qwen Code) | Full compaction after exceeding 70% | Loses many details at once and may forget key decisions |
| 5-layer incremental compaction (Claude Code) | Escalates from lightweight to heavyweight layers | Preserves information as much as possible and transitions smoothly |

### Why "Simple Compaction" Is Not Enough

- Tool output occupies 60-80% of context (one `npm test` may output 500 lines)
- The information value of old tool output decays over time (a `git diff` from 3 turns ago is stale)
- But the Agent's **decision reasoning** does not decay ("I chose strategy B because A had performance issues")
- Simple compaction cannot distinguish "discardable tool output" from "non-discardable decision context"

Claude Code's 5-layer compaction solves this precisely—each layer targets different content types and starts trimming from the lowest-value information.

### Competitor Context Management Comparison

| Agent | Compaction strategy | Memory system | Crash recovery |
|-------|---------|---------|---------|
| **Claude Code** | 5-layer incremental + Tool Output Masking | CLAUDE.md + Auto Dream + Team Memory | 3 detection types + synthetic continuation |
| **Gemini CLI** | Single-threshold compaction (default 50%) + Tool Output Masking | GEMINI.md | None |
| **Qwen Code** | Single-threshold compaction (70%) | QWEN.md + simple notes | None |
| **Copilot CLI** | Context management + checkpoints | copilot-instructions.md | Limited |
| **Cursor** | VS Code session management | .cursorrules | IDE-level recovery |
>
> **Counting rules**: Source line counts are based on `wc -l` statistics for TypeScript files. The "Total directory scale" column in section 7.1.1 is the total for all files in the subsystem; the number in parentheses is the exact LOC for the core file. The ~73,000-line figure for the whole article is the total for all session/memory-related directories, including auxiliary modules such as token management, conversation recovery, and concurrency management.

## 7.1 Architecture Overview

### 7.1.1 Subsystem Panorama

| Subsystem | Total directory scale | Core file (single-file LOC) | Responsibility |
|--------|-----------|----------------------|------|
| **Session storage** | ~9,938 LOC | `utils/sessionStorage.ts` (5,105) | JSONL persistence, session recovery, concurrency management |
| **Context compaction** | ~3,960 LOC | `services/compact/compact.ts` (1,706) | 5-layer incremental compaction strategy |
| **Token management** | ~3,327 LOC | `utils/analyzeContext.ts` (1,382) | Budget allocation, usage estimation, threshold warnings |
| **CLAUDE.md memory** | ~1,876 LOC | `utils/claudemd.ts` (1,480) | 6-layer discovery, `@include` directives, conditional loading |
| **Memdir system** | ~1,736 LOC | `memdir/memdir.ts` (507) | Memory directory management, path resolution |
| **Automatic memory extraction** | ~769 LOC | `services/extractMemories/extractMemories.ts` (616) | Background memory extraction after conversations end |
| **SessionMemory** | ~1,026 LOC | `services/SessionMemory/sessionMemory.ts` (495) + `prompts.ts` (324) + `utils.ts` (207) | Lightweight in-session memory cache |
| **autoDream** | ~550 LOC | `services/autoDream/autoDream.ts` (325) | Background memory consolidation (24h + 5-session gates) |
| **Team memory sync** | ~2,167 LOC | `services/teamMemorySync/index.ts` (1,257) | Local↔cloud bidirectional synchronization |
| **Worktree** | ~2,888 LOC | `utils/worktree.ts` (1,519) | Git worktree isolated sessions |
| **File checkpoints** | ~1,116 LOC | `utils/fileHistory.ts` (1,116) | File-level snapshots/rollback |

> **Counting note**: "Total directory scale" is the total LOC of all files in that subsystem, including auxiliary files, type definitions, UI components, and so on. "Core file" is the largest single file, with exact LOC in parentheses.

### 7.1.2 Session Lifecycle

```
┌─────────────┐     ┌─────────────────────────────┐     ┌─────────────┐
│  Create      │     │  Active                      │     │  Resume/    │
│             │     │                             │     │  Archive    │
│ Lazy JSONL   │────▶│ Append messages              │────▶│ --resume    │
│ Register PID │     │ PID status updates           │     │ --continue  │
│ Init memory  │     │ Tool result persistence      │     │ --fork      │
│             │     │  ┌──────────────────┐        │     │ Conversation│
│             │     │  │ Context compaction│        │     │ chain replay│
│             │     │  │ 5-layer / token   │        │     │             │
│             │     │  │ budget            │        │     │             │
│             │     │  └──────────────────┘        │     │             │
└─────────────┘     └─────────────────────────────┘     └─────────────┘
```

**Key designs**:
- Session files are **created lazily**: the JSONL file is created only when the first user/assistant message appears; before that, metadata is cached in memory in `pendingEntries`
- **Append-only JSONL**: messages are appended only and not updated in place. Deletion uses tail-splice for efficiency
- **PID-file concurrency management**: `~/.claude/sessions/{pid}.json` tracks running sessions; `claude ps` can display them
- **Context compaction** is an **asynchronous subtask** in the active phase, not a separate linear phase—it triggers automatically when token usage exceeds thresholds and can also be triggered manually with `/compact`

## 7.2 Session Storage

### 7.2.1 Storage Format

Source: `utils/sessionStorage.ts`

**Session file path**: `~/.claude/projects/{sanitized_project_path}/{sessionId}.jsonl`

One JSON entry per line, with core fields:

```json
{
  "type": "user|assistant|attachment|system|last-prompt|custom-title|tag|...",
  "uuid": "...",
  "parentUuid": "...",
  "timestamp": "...",
  "sessionId": "..."
}
```

**Metadata entry types** (persisted through `reAppendSessionMetadata()`):

| Entry type | Purpose |
|---------|------|
| `last-prompt` | Last user prompt text |
| `custom-title` | Session title set by the user/AI |
| `tag` | Session tag |
| `agent-name` / `agent-color` / `agent-setting` | Agent context |
| `mode` | `coordinator` vs `normal` mode |
| `worktree-state` | Worktree session state (three states: undefined/null/object) |
| `pr-link` | Associated PR metadata |
| `file-history-snapshot` | File checkpoint snapshot |
| `content-replacement` | Cache-optimization replacement record |

**Subagent transcript path**: `{projectDir}/{sessionId}/subagents/agent-{agentId}.jsonl`

**File permissions**: `0o600` (read/write by owner only), directories `0o700`

### 7.2.2 Key Constants

Source: `utils/sessionStorage.ts`

| Constant | Value | Purpose |
|------|---|------|
| `MAX_TRANSCRIPT_READ_BYTES` | 50 MB | Prevent OOM |
| `MAX_TOMBSTONE_REWRITE_BYTES` | 50 MB | Safety limit for tail-splice |
| `FLUSH_INTERVAL_MS` | 100 ms | Write-queue drain interval |
| `MAX_CHUNK_BYTES` | 100 MB | Maximum single write chunk |

### 7.2.3 Project Singleton and Write Queue

Source: `utils/sessionStorage.ts` — `Project` class

`Project` is the singleton that manages all writes and uses a **write-queue pattern**:
- Entries are cached in `writeQueues` (a per-file Map)
- Drained every 100ms via `scheduleDrain()` / `drainWriteQueue()`
- Supports `flushSessionStorage()` to wait for draining

Cached metadata fields:
```
currentSessionTag, currentSessionTitle, currentSessionAgentName,
currentSessionAgentColor, currentSessionLastPrompt,
currentSessionAgentSetting, currentSessionMode ('coordinator' | 'normal'),
currentSessionWorktree (three states), currentSessionPrNumber, ...
```

### 7.2.4 Message Chain Construction

Source: `utils/sessionStorage.ts` — `buildConversationChain(byUuid, tip)`

Starting from a leaf node, it walks back through the `parentUuid` chain to the root to build the full conversation tree. `progress` entries are excluded from the chain—they are temporary UI state and contain data that can cause chain forks during recovery (bugs #14373, #23537).

### 7.2.5 Lightweight Metadata Reading

Source: `utils/sessionStoragePortable.ts`

`readLiteMetadata()` reads only the last ~64KB tail window of the JSONL file to extract metadata, without scanning the whole file. When a session exits, `reAppendSessionMetadata()` rewrites metadata to EOF so the tail window can find the latest metadata.

> **Design rationale**: For long sessions (tens of thousands of messages), full-file scans can take several seconds. The tail-window approach reduces metadata reads to milliseconds.

## 7.3 Session Recovery

### 7.3.1 Recovery Entrypoints

```bash
# Resume the most recent session
claude --resume

# Resume a specified session
claude --resume <session-id>

# Continue the previous conversation
claude -c

# Resume and create a new session ID (do not overwrite the original session)
claude --resume <session-id> --fork-session

# Resume from an associated PR
claude --from-pr 123
```

### 7.3.2 Recovery Flow

Source: `utils/conversationRecovery.ts` → `utils/sessionRestore.ts`

```
loadConversationForResume(source)
  ├── 1. Locate JSONL file (latest / session-id / preloaded)
  ├── 2. Deserialize + filter
  │     ├── migrateLegacyAttachmentTypes()    # new_file→file, new_directory→directory
  │     ├── filterUnresolvedToolUses()        # remove unpaired tool_use/tool_result
  │     ├── filterOrphanedThinkingOnlyMessages()
  │     └── filterWhitespaceOnlyAssistantMessages()
  ├── 3. Detect turn interruption (detectTurnInterruption)
  │     ├── 'none': last turn completed
  │     ├── 'interrupted_prompt': user sent a message but model did not respond
  │     └── 'interrupted_turn': interrupted during model execution → inject "Continue from where you left off."
  ├── 4. Restore Skill state
  └── 5. Trigger SessionStart hooks (resume type)

processResumedConversation(result)
  ├── 1. Match coordinator/normal mode
  ├── 2. Set session ID (or fork a new ID)
  ├── 3. Restore session metadata (title, tag, Agent, Worktree, PR)
  ├── 4. Restore Worktree directory (chdir)
  ├── 5. Restore Agent definitions and model overrides
  └── 6. Update PID file
```

### 7.3.3 Turn Interruption Detection

Source: `utils/conversationRecovery.ts` — `detectTurnInterruption()`

| Interruption type | Condition | Recovery behavior |
|---------|---------|---------|
| `none` | Last entry is assistant/system/meta | Resume directly |
| `interrupted_prompt` | Last entry is a normal user message | Inject "Continue from where you left off." |
| `interrupted_turn` | Last entry is `tool_result` or `attachment` | Convert to `interrupted_prompt` and inject continuation message |

**Special handling**:
- **Brief mode**: `SendUserMessage` tool_result is terminal state (turn completed) and is not misclassified as interrupted
- **API error messages**: skipped when finding the last turn-related message

### 7.3.4 Cross-Project Resume

Source: `utils/crossProjectResume.ts`

When the session specified by `--resume` belongs to a different project directory, the system detects the cross-project case and handles it separately:
- **Same repo worktree**: resume directly without changing directories
- **Different project**: return a `cd '/path' && claude --resume <sessionId>` command and let the caller decide whether to execute it

> **Note**: Worktree detection is currently enabled only for users with `USER_TYPE === 'ant'` (staged rollout). Non-ant users always use the command-suggestion path.

## 7.4 Context Compaction (5-Layer Incremental Strategy)

Context compaction is one of Claude Code's most complex subsystems. It is divided into 5 layers by increasing cost; each layer triggers when the previous layer cannot free enough space.

### 7.4.1 Compaction Layer Overview

| Layer | Mechanism | Trigger timing | Information loss | Source |
|------|------|---------|---------|------|
| **L1: Cache-edit micro-compaction** | `cache_edits` API deletes old tool results | Before every API call (Ant only) | Tool output text | `services/compact/microCompact.ts` |
| **L2: Time-based micro-compaction** | Clear expired tool results in place | When server cache has expired | Tool output text | `services/compact/microCompact.ts` |
| **L3: SessionMemory compaction** | Trim old messages based on SessionMemory | Tried **first** after auto-compaction triggers (shares threshold with L4, ~83.5%) | Old conversation turns | `services/compact/sessionMemoryCompact.ts` |
| **L4: Full compaction (summary)** | Fork a subagent to generate a summary replacing history | After L3 fails, manual `/compact` | Most conversation details | `services/compact/compact.ts` |
| **L5: Reactive compaction** | Emergency compaction after API 413 | API returns prompt_too_long | Potentially severe | `query.ts` |

### 7.4.2 Token Budget Calculation

Source: `services/compact/autoCompact.ts`, `utils/tokens.ts`

```
Raw context window:       200,000 tokens (default model)
                          1,000,000 tokens (Sonnet 4 / Opus 4-6)

Summary reservation:      min(modelMaxOutput, 20,000) tokens
                          (based on p99.99 = 17,387 tokens)

Effective context window: rawWindow - summaryReserved
                          = 200,000 - 20,000 = 180,000

Auto-compaction threshold: effectiveWindow - 13,000
                          = 180,000 - 13,000 = 167,000 (~83.5%)

Warning threshold:        autoCompactThreshold - 20,000
Error threshold:          autoCompactThreshold - 20,000
Blocking limit:           effectiveWindow - 3,000
```

**Token estimation function** `tokenCountWithEstimation()` (`utils/tokens.ts`):
- Takes the `usage` from the last API response (input + output + cache tokens)
- Adds a rough estimate for messages appended since then
- Pads by a 4/3 ratio (conservative estimate)
- Handles parallel tool calls by walking back to the first sibling message with the same `message.id`

### 7.4.3 L1/L2: Micro-Compaction

Source: `services/compact/microCompact.ts` (531 LOC)

**Compactable tool set** (`COMPACTABLE_TOOLS`): FileRead, Shell, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite

**L1: Cache-edit micro-compaction** (Ant only, feature gate: `CACHED_MICROCOMPACT`):
- Uses Anthropic `cache_edits` API to delete tool results directly in the cached prefix
- **Does not invalidate the cache**—no need to resend content
- Only changes cache deletion instructions; local content is unchanged

**L2: Time-based micro-compaction**:
- Checks whether the time since the last assistant message exceeds the threshold (server cache has expired)
- Replaces old tool result content with `[Old tool result content cleared]`
- Because the cache has expired, clearing content is "free" (does not increase rewrite cost)

> **Design rationale**: Micro-compaction runs only on the main thread (`isMainThreadSource`). The source comment says: "Forked agents (session_memory, prompt_suggestion, etc.) would cause the main thread to try deleting tools that don't exist in its own conversation."

### 7.4.4 L3: SessionMemory Compaction

Source: `services/compact/sessionMemoryCompact.ts` (631 LOC)

Experimental structure-aware compaction, tried before full compaction.

**Configuration** (`DEFAULT_SM_COMPACT_CONFIG`):

| Parameter | Default | Purpose |
|------|--------|------|
| `minTokens` | 10,000 | Minimum tokens to keep |
| `minTextBlockMessages` | 5 | Minimum messages containing text blocks to keep |
| `maxTokens` | 40,000 | Hard upper token limit to keep |

**Flow**:
1. Check whether SessionMemory is available and non-empty
2. Find the previous compaction's `lastSummarizedMessageId`
3. Calculate the trim point: extend backward to satisfy minimum token/message constraints
4. `adjustIndexToPreserveAPIInvariants()` ensures `tool_use`/`tool_result` pairs or thinking blocks are not split
5. Create a compaction boundary message and build post-compaction messages

> **Design rationale**: `adjustIndexToPreserveAPIInvariants()` handles a subtle streaming artifact—"streaming yields separate messages per content block (thinking, tool_use, etc.) with the same message.id but different uuids". Splitting these groups would cause "API error: orphan tool_result references non-existent tool_use".

### 7.4.5 L4: Full Compaction

Source: `services/compact/compact.ts` (1,706 LOC)

**Core function**: `compactConversation()`

**Full compaction flow**:

```
1. Execute PreCompact hooks (merge hook instructions and user instructions)
2. Build compaction prompt (prompt.ts)
3. Try the forked-agent path (reuse the main thread's prompt cache)
   ├── Success → use the result
   └── Failure → fall back to the streaming path (regular API call)
4. If the response is prompt_too_long:
   └── truncateHeadForPTLRetry() discards the oldest API-round group and retries (up to 3 times)
5. Post-compaction cleanup:
   ├── Clear file-read cache
   ├── Regenerate attachments (files, plans, Skills, deferred tools, MCP instructions, Agent list)
   └── Execute SessionStart hooks + PostCompact hooks
6. Reset cache-break detection baseline (notifyCompaction)
7. Record telemetry (`tengu_compact` event)
```

**Key constants**:

| Constant | Value | Purpose |
|------|---|------|
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Maximum number of files reinjected after compaction |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Total token budget for reinjected files |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | Maximum tokens per file |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | Total Skill budget (~5 Skills) |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | Maximum tokens per Skill |
| `MAX_PTL_RETRIES` | 3 | prompt-too-long retry count |

**Partial compaction** (`partialCompactConversation`):
- `'from'` direction: summarize messages **after** `pivotIndex` (preserves prompt-cache prefix)
- `'up_to'` direction: summarize messages **before** `pivotIndex` (invalidates cache)

**`CompactionResult` interface**:
```typescript
interface CompactionResult {
  boundaryMarker: string           // Compaction boundary marker
  summaryMessages: Message[]       // Generated summary messages (replace the compacted conversation)
  attachments: Attachment[]        // Attachments (preserved)
  hookResults: HookResult[]        // Hook execution results
  messagesToKeep?: Message[]       // Retained messages that were not compacted
  preCompactTokenCount?: number    // Token count before compaction
  postCompactTokenCount?: number   // Token count after compaction
}
```

### 7.4.6 Compaction Prompt

Source: `services/compact/prompt.ts` (374 LOC)

**Prompt structure** (3 sections):

```
┌─────────────────────────────────┐
│ NO_TOOLS_PREAMBLE               │ ← "Respond with TEXT ONLY" (placed first)
│   Prevent Sonnet 4.6+ adaptive  │   Sonnet 4.6 failure rate 2.79% vs 0.01% for 4.5
│   thinking                      │
├─────────────────────────────────┤
│ <analysis> scratch block        │ ← Draft area for the model to organize thoughts
├─────────────────────────────────┤
│ 9-section summary template      │
│  1. Primary Request and Intent  │
│  2. Key Technical Concepts      │
│  3. Files and Code Sections     │
│  4. Errors and Fixes            │
│  5. Problem Solving             │
│  6. All User Messages           │
│  7. Pending Tasks               │
│  8. Current Work                │
│  9. Optional Next Step          │
├─────────────────────────────────┤
│ NO_TOOLS_TRAILER                │ ← Reinforce "do not call tools"
└─────────────────────────────────┘
```

`formatCompactSummary()` processes output: removes the `<analysis>` draft block and replaces `<summary>` XML tags with plain-text headings.

### 7.4.7 Auto-Compaction Trigger

Source: `services/compact/autoCompact.ts` (351 LOC)

**Trigger gating chain**:

```
1. Query source check → skip session_memory/compact queries (avoid deadlock)
2. Feature flags → REACTIVE_COMPACT suppresses proactive compaction
                   CONTEXT_COLLAPSE suppresses it (would compete with collapse)
3. Threshold check → tokenCount >= effectiveWindow - 13,000
4. Try SessionMemory compaction first
   └── Failure → fall back to full compaction
```

**Circuit breaker**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`; after 3 consecutive failures, stop retrying.

> **Design rationale**: The source comment explains the origin of the circuit breaker—"was causing ~250K wasted API calls/day". In unrecoverable oversized sessions, 1,279 sessions produced 50+ failures.

**Environment variable overrides**:

| Variable | Purpose |
|------|------|
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Override effective context window |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Trigger by percentage |
| `DISABLE_COMPACT` | Disable all compaction |
| `DISABLE_AUTO_COMPACT` | Disable auto compaction but keep manual `/compact` |

### 7.4.8 Tool Result Persistence

Source: `utils/toolResultStorage.ts`

Outside compaction, large tool results are managed by a separate mechanism:

**Three-layer size management**:

| Layer | Mechanism | Threshold |
|------|------|------|
| Per-result persistence | Results above the threshold are saved to disk; the model receives a path + 2KB preview | `min(tool.maxResultSizeChars, 50,000)` characters |
| Aggregate message budget | Total budget for parallel tool results in one user message | 200,000 characters (`MAX_TOOL_RESULTS_PER_MESSAGE_CHARS`) |
| Micro-compaction clearing | Replace old tool results in place with `[Old tool result content cleared]` | Triggered by time/cache expiry |

**Cache stability**: Once a tool result has been "seen", it is "frozen" (either replaced or kept as-is). The same decision is reapplied every turn to preserve the prompt-cache prefix.

### 7.4.9 Message Grouping

Source: `services/compact/grouping.ts` (63 LOC)

Messages are grouped by **API-round boundaries** (new `assistant.message.id`), ensuring compaction does not split `tool_use`/`tool_result` pairs.

## 7.5 CLAUDE.md Memory System

### 7.5.1 Six-Layer Discovery Hierarchy

Source: `utils/claudemd.ts` (1,479 LOC)

| Layer | Name | Path | Scope | Excludable |
|------|------|------|--------|--------|
| 1 | Managed | `/etc/claude-code/CLAUDE.md` | Sysadmin policy (all users) | ❌ |
| 2 | User | `~/.claude/CLAUDE.md` + `~/.claude/rules/*.md` | Personal global instructions | ✅ |
| 3 | Project | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` | Team-shared (committed to Git) | ✅ |
| 4 | Local | `CLAUDE.local.md` (project root) | Personal project-level (gitignored) | ✅ |
| 5 | AutoMem | `~/.claude/projects/<slug>/memory/MEMORY.md` | Auto-generated | ❌ |
| 6 | TeamMem | `<autoMemPath>/team/MEMORY.md` | Team-shared (feature flag) | ❌ |

**Loading priority**: **Later-loaded content has higher priority**—the model pays more attention to content that appears later. Within the Project/Local layers, files closer to the CWD are loaded later (higher priority). Directory traversal proceeds from root to CWD.

> **Design rationale**: The source comment says: "Files loaded LATER have HIGHER priority because the model pays more attention to later content." This is counterintuitive (usually earlier-loaded means more important), but matches LLM attention characteristics.

> **Layer 3 vs Layer 4 note**: Project and Local are loaded during the same directory traversal, but they are independent layers—they use different `MemoryType` markers (`'Project'` vs `'Local'`), are gated by different `isSettingSourceEnabled` sources (`projectSettings` vs `localSettings`), and Local is not affected by worktree deduplication.

### 7.5.2 `@include` Directive System

Source: `utils/claudemd.ts`

**Syntax**: `@path`, `@./relative/path`, `~/home/path`, `@/absolute/path`

**Constraints**:
- Works only in leaf text nodes (not inside code blocks)
- Supports recursion, with maximum depth `MAX_INCLUDE_DEPTH = 5`
- Uses the `processedPaths` set to prevent circular references
- Supports 70+ text-file extensions (allowlist `TEXT_FILE_EXTENSIONS`)
- HTML comments are stripped; frontmatter parsing is used for glob-based conditional rules

**Settings-source gating**: Each layer is controlled by its settings source—`userSettings`, `projectSettings`, and `localSettings` control their respective layers. `claudeMdExcludes` patterns can skip specific User/Project/Local files (Managed/AutoMem/TeamMem are never excluded).

### 7.5.3 `getMemoryFiles()` Function

Source: `utils/claudemd.ts:790` (memoized)

Processing order:
1. Managed → User → Project/Local (root→CWD traversal) → `--add-dir` directories → AutoMem → TeamMem
2. Each step calls `processMemoryFile()` to process `@include` recursion
3. `getClaudeMds()` formats everything into a unified prompt string

**Header statement**: `"Codebase and user instructions are shown below. These instructions OVERRIDE any default behavior."`

Each file includes a descriptive label (for example, "project instructions, checked into the codebase").

### 7.5.4 AutoMem Path Resolution

Source: `memdir/paths.ts` (278 LOC)

`getAutoMemPath()` priority chain:
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` environment variable (SDK/Cowork full-path override)
2. `autoMemoryDirectory` in `settings.json` (trusted sources only: policy/flag/local/user; **projectSettings excluded** to prevent injection)
3. Default: `<memoryBase>/projects/<sanitized-git-root>/memory/`

**Path safety** (`validateMemoryPath()`): rejects relative paths, root paths (length < 3), Windows drive roots, UNC paths, null bytes, and bare `~` expansion.

### 7.5.5 Memory Content Types

Source: `memdir/memoryTypes.ts` (271 LOC)

| Type | Scope | Content |
|------|------|------|
| `user` | Always private | User role, goals, preferences, knowledge level |
| `feedback` | Private by default; shareable at team level | Corrections and confirmed methods |
| `project` | Team-oriented | Ongoing work, goals, deadlines |
| `reference` | Usually team-level | Pointers to external systems (Linear projects, Grafana dashboards) |

**Content exclusion rules** (`WHAT_NOT_TO_SAVE_SECTION`):
- Do not save code patterns/architecture/file paths (derivable from project state)
- Do not save git history (use `git log`/`git blame`)
- Do not save debugging solutions (the fix is already in code)
- Do not save content already in CLAUDE.md
- Do not save temporary task details
- These exclusion rules still apply even when saving explicitly

**Truncation limit**: MEMORY.md is capped at 200 lines **and** 25KB.

## 7.6 Automatic Memory Extraction

### 7.6.1 ExtractMemories (Background Agent at Turn End)

Source: `services/extractMemories/extractMemories.ts` (616 LOC)

Runs at the end of each complete query loop (when the model produces a final response with no tool calls), triggered through `handleStopHooks`.

**Characteristics**:
- Feature gates: `tengu_passport_quail` (extraction mode) + `tengu_slate_thimble` (non-interactive sessions)
- Uses `runForkedAgent()`—perfectly copies the main conversation and shares prompt cache
- **Mutually exclusive with the main agent**: if the main agent wrote memory in the current turn (`hasMemoryWritesSince()` detection), the background agent skips that range
- Allowed tools: FileEdit, FileWrite, FileRead, Glob, Grep, Bash (read-only), REPL
- The `createAutoMemCanUseTool()` factory constrains tools to operate only inside the automatic memory directory

### 7.6.2 SessionMemory (In-Session Memory Cache)

Source: `services/SessionMemory/` (3 files, 1,026 LOC total: `sessionMemory.ts` 495 + `prompts.ts` 324 + `sessionMemoryUtils.ts` 207)

Lightweight in-session memory, periodically extracted through a forked subagent.

**Characteristics**:
- Feature gate: `tengu_session_memory` GrowthBook flag
- **Initialization threshold**: triggered when token count exceeds a configurable threshold
- **Update threshold**: requires minimum token growth + minimum tool-call count (since the last extraction)
- Uses `runForkedAgent()` to share prompt cache
- Writes a session-scoped memory file (`getSessionMemoryPath()`)
- Customizable: `~/.claude/session-memory/config/template.md` and `prompt.md` override default prompts

### 7.6.3 autoDream (Background Memory Consolidation)

Source: `services/autoDream/autoDream.ts` (325 LOC)

When both time gating and session-count gating pass, runs the `/dream` prompt through a forked subagent.

**Dual trigger gates + execution lock** (starting from the cheapest):

| Stage | Type | Condition | Default |
|------|------|------|--------|
| **Time gate** | Trigger condition | Time since last consolidation >= minHours | 24h (GrowthBook `tengu_onyx_plover`) |
| **Session gate** | Trigger condition | Number of sessions with mtime > lastConsolidatedAt >= minSessions | 5 |
| **Lock gate** | Execution protection | No other process is consolidating | File lock `tryAcquireConsolidationLock()` |

**Additional constraints**:
- Scan throttling: session directory scans at least 10 minutes apart (`SESSION_SCAN_INTERVAL_MS`)
- Excludes the current session
- KAIROS mode (long-lived assistant session): appends timestamped entries to a dated log file and consolidates with nightly `/dream`
- On success, records `tengu_auto_dream_completed` telemetry (including cache hit/creation metrics)
- On failure, rolls back lock mtime so the time gate passes again

## 7.7 Team Memory Synchronization

### 7.7.1 Architecture

Source: `services/teamMemorySync/index.ts` (1,257 LOC)

Team memory synchronizes between the local filesystem and Anthropic server APIs, scoped by repository (identified through a git remote hash) and shared by all authenticated members in the organization.

### 7.7.2 API Contract

| Operation | Method | Description |
|------|------|------|
| Pull | `GET /api/claude_code/team_memory?repo={owner/repo}` | Supports ETag/304 |
| Checksum | `GET ...&view=hashes` | Fetches only SHA-256 for conflict detection |
| Push | `PUT /api/claude_code/team_memory?repo={owner/repo}` | Upsert semantics |

### 7.7.3 Synchronization Semantics

| Operation | Behavior |
|------|------|
| **Pull** | Server wins (local files are overwritten) |
| **Push** | Incremental upload—only entries with different SHA-256 content hashes are pushed |
| **Deletes do not propagate** | Deleting a local file does not delete it from the server; the next pull restores it |

> ⚠️ **Inference**: The non-propagating delete design may be intended to prevent one member's mistake from affecting the entire team, but it increases coordination cost.

### 7.7.4 Security Measures

Source: `services/teamMemorySync/secretScanner.ts` (324 LOC)

| Measure | Implementation |
|------|------|
| **Secret scanning** | Run `scanForSecrets()` before Push to prevent credential leaks |
| **Path validation** | `validateTeamMemKey()` prevents path traversal |
| **Settings-source trust** | Only policy/flag/local/user can configure team memory paths (**projectSettings excluded**) |
| **Authentication** | Requires first-party OAuth (`CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`) |

**Key constants**:

| Constant | Value |
|------|---|
| `MAX_FILE_SIZE_BYTES` | 250,000 / entry |
| `MAX_PUT_BODY_BYTES` | 200,000 (gateway limit; large batches split into sequential PUTs) |
| `TEAM_MEMORY_SYNC_TIMEOUT_MS` | 30,000 |
| `MAX_RETRIES` | 3 |
| `MAX_CONFLICT_RETRIES` | 2 |

## 7.8 Worktree Isolation

### 7.8.1 Overview

Source: `utils/worktree.ts` (1,519 LOC) + `tools/EnterWorktreeTool/` (177 LOC) + `tools/ExitWorktreeTool/` (386 LOC)

Git worktree allows multiple Claude Code instances to work on different branches in parallel without interfering with each other.

### 7.8.2 Worktree Creation

```bash
# CLI method
claude --worktree [name]
claude --worktree --tmux    # Run in a tmux session

# Tool method (dynamic switching within a session)
EnterWorktree: { name?: string }
ExitWorktree:  { action: 'keep' | 'remove', discard_changes?: boolean }
```

**Creation flow** (`getOrCreateWorktree`):

1. **Fast recovery path**: directly reads the `.git` pointer file (no subprocess, saves ~15ms)
2. **New worktree path**:
   - Gets the base branch (skips fetch if `origin/<branch>` already exists locally, saving 6-8s)
   - `git worktree add -B`
   - Supports `--no-checkout` + `sparse-checkout set --cone` for large repositories
3. **Post-creation setup** (`performPostCreationSetup`):
   - Copies `settings.local.json`
   - Configures `core.hooksPath` (husky compatibility)
   - Symlinks configuration directories (such as `node_modules`, to avoid disk bloat)
   - Copies `.worktreeinclude` files
   - Installs commit attribution hook (`installPrepareCommitMsgHook`, feature flag `COMMIT_ATTRIBUTION`)

**Worktree storage location**: `.claude/worktrees/<flattened-slug>`

**Slug validation** (`validateWorktreeSlug`):
- Rejects path traversal (`..`, absolute paths)
- Validates each segment: `/^[a-zA-Z0-9._-]+$/`
- Maximum 64 characters
- `flattenSlug()` replaces `/` with `+` to avoid git ref D/F conflicts

### 7.8.3 Worktree Cleanup

| Operation | Behavior |
|------|------|
| `keep` | Keep the worktree + branch on disk |
| `remove` | `git worktree remove --force` + `git branch -D` (wait 100ms for lock release) |

**Stale cleanup** (`cleanupStaleAgentWorktrees`):
- Cleans only slugs matching 6 temporary patterns (`EPHEMERAL_WORKTREE_PATTERNS`)
- **Fail-closed**: skips if `git status` fails or tracked changes exist; also skips commits not reachable from remote
- `hasWorktreeChanges()` checks dirty worktree + new commits

### 7.8.4 ExitWorktree Safety Validation

Source: `tools/ExitWorktreeTool/ExitWorktreeTool.ts`

Safety gates in `validateInput()`:
1. Rejects cases with no active session
2. When `action: 'remove'` and `discard_changes: true` is not set:
   - If `countWorktreeChanges` returns null (cannot verify status) → fail
   - If there are uncommitted files or new commits → fail (lists exact counts)

This is the only safety gate that prevents data loss.

## 7.9 File Checkpoints and Rollback

### 7.9.1 Overview

Source: `utils/fileHistory.ts` (1,116 LOC)

Claude Code automatically creates file snapshots before every tool call, supporting rollback at precise file granularity.

### 7.9.2 Storage Format

**Backup path**: `~/.claude/file-history/<sessionId>/<hash>@v<N>`

- `hash`: first 16 hexadecimal characters of `sha256(filePath)`
- Version number: monotonically increases per file
- `null` backupFileName = the file did not exist at that version
- Paths are stored relative to `originalCwd` (saves space)

### 7.9.3 Core Data Structures

```typescript
type FileHistoryBackup = {
  backupFileName: string | null  // null = file does not exist
  version: number
  backupTime: Date
}

type FileHistorySnapshot = {
  messageId: UUID
  trackedFileBackups: Record<string, FileHistoryBackup>
  timestamp: Date
}

type FileHistoryState = {
  snapshots: FileHistorySnapshot[]
  trackedFiles: Set<string>
  snapshotSequence: number  // Monotonic counter, never reset
}
```

**Snapshot limit**: `MAX_SNAPSHOTS = 100`

### 7.9.4 Three-Phase Pattern

All operations use a three-phase **synchronous capture → asynchronous I/O → synchronous commit** pattern:

**`fileHistoryTrackEdit`** (called before file writes):
1. Phase 1 (sync): check whether the file is already tracked in the latest snapshot
2. Phase 2 (async): `createBackup()` creates the backup
3. Phase 3 (sync): commit to state (recheck race conditions)

**`fileHistoryMakeSnapshot`** (called once per turn):
1. Phase 1 (sync): capture state through a no-op updater
2. Phase 2 (async): stat each tracked file + compare with latest backup (mtime optimization skips unmodified files)
3. Phase 3 (sync): append snapshot and evict down to MAX_SNAPSHOTS

### 7.9.5 Rollback Mechanism

**`fileHistoryRewind(state, messageId)`**: restores all tracked files to their state at the snapshot corresponding to the specified `messageId`.

`applySnapshot()` behavior:
- Deletes files that did not exist in the target version
- Restores files through `restoreBackup()` (copy + restore permissions)
- Writes only files that differ from current state

**Helper functions**:
- `fileHistoryGetDiffStats()`: dry-run rollback and return `{ filesChanged, insertions, deletions }`
- `fileHistoryHasAnyChanges()`: lightweight boolean check; returns on the first changed file

### 7.9.6 Recovery Support

Source: `copyFileHistoryForResume(log)` — migrates backups from the old session to the new session through hard links (`fs.link`, falling back to copy). Only snapshots for which all backup migrations succeeded are recorded.

### 7.9.7 Enablement Conditions

```typescript
fileHistoryEnabled():
  // Interactive mode: enabled by default
  //   Disabled when: fileCheckpointingEnabled === false or CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING

fileHistoryEnabledSdk():
  // SDK/non-interactive mode: disabled by default
  //   Must be explicitly enabled: CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING=true
  //   And must not be overridden by CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING
```

> **Design rationale**: SDK mode is disabled by default because automation scripts usually run in environments without user interaction, where the disk overhead and I/O latency of file snapshots may be unacceptable.

## 7.10 Concurrent Session Management

### 7.10.1 PID File System

Source: `utils/concurrentSessions.ts` (204 LOC)

**PID directory**: `~/.claude/sessions/`
**PID file**: `{process.pid}.json`

```json
{
  "pid": "<process.pid>",
  "sessionId": "<UUID>",
  "cwd": "<original working dir>",
  "startedAt": "<timestamp>",
  "kind": "<interactive|bg|daemon|daemon-worker>",
  "status": "<busy|idle|waiting>",
  "name": "<session name>",
  "bridgeSessionId": "<remote session ID or null>"
}
```

**Session types** (`SessionKind`):

| Type | Description |
|------|------|
| `interactive` | Interactive CLI session |
| `bg` | Background tmux session (`claude --bg`) |
| `daemon` | Daemon session |
| `daemon-worker` | Daemon worker thread |

**Session status** (`SessionStatus`):

| Status | Description |
|------|------|
| `busy` | Processing |
| `idle` | Waiting for input |
| `waiting` | Blocked while waiting for a specific condition |

**Stale cleanup**: `countConcurrentSessions()` automatically deletes PID files for processes that no longer exist (skipped on WSL because WSL cannot detect Windows host PIDs through `/proc`, preventing accidental deletion of running host processes). Strict filename validation `/^\d+\.json$/` prevents accidental deletion.

## 7.11 MCP Integration

### 7.11.1 Transports

| Protocol | Description | Applicable scenarios |
|------|------|----------|
| **stdio** | Standard input/output | Local processes, most common |
| **sse** | Server-Sent Events | Remote servers |
| **streamable-http** | Streamable HTTP | Cloud deployments |

### 7.11.2 Configuration

**CLI method**: `--mcp-config <configs...>` loads MCP server configurations (JSON files or strings)

**Project-level** (`.claude/settings.json`):

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_xxx" }
    },
    "remote-server": {
      "type": "sse",
      "url": "https://mcp.example.com/sse"
    }
  }
}
```

### 7.11.3 Tool Naming and Limits

MCP tools are registered in the `mcp__serverName__toolName` format (double underscores as separators).

**Key constants**:

| Constant | Value | Purpose |
|------|---|------|
| `MAX_MCP_OUTPUT_TOKENS` | 25,000 | MCP output token limit (overridable via environment variable) |
| MCP truncation multiplier | 0.5 | Truncate output to 50% when it exceeds the limit |

**MCP-related tools** (see section 4.8 of 04-tools.md):
- `McpAuthTool`: OAuth flow for unauthenticated MCP servers
- `ReadMcpResourceTool`: read MCP resources
- `ListMcpResourcesTool`: list MCP resources

**SDK control messages** (for MCP management):
- `mcp_status`: query MCP server status
- `mcp_message`: send MCP message
- `mcp_set_servers`: dynamically set MCP servers
- `mcp_reconnect`: reconnect MCP servers
- `mcp_toggle`: enable/disable MCP servers

## 7.12 Implementer Checklist

For Code Agent developers designing session and memory systems:

### Session Persistence
- [ ] Use append-only JSONL format (avoid in-place update concurrency issues)
- [ ] Rewrite metadata at the tail (fast metadata reads without full-file scans)
- [ ] Track running sessions with PID files (support `ps` command and concurrency limits)
- [ ] Use a `parentUuid` tree for message chains (support forks and recovery)
- [ ] Lazily create session files (avoid junk files from empty sessions)
- [ ] SIGINT/SIGTERM signal handling: force-drain the `pendingEntries` queue on graceful exit to ensure JSONL integrity

### Context Compaction
- [ ] Implement a multi-layer incremental compaction strategy (try layers from low cost to high cost)
- [ ] Prefer micro-compaction to clear expired cached content (zero extra cost)
- [ ] Try SessionMemory compaction before full compaction (preserve more information)
- [ ] Use a forked-agent for full compaction to reuse prompt cache (reduce cost)
- [ ] Use a circuit breaker to prevent consecutive compaction failures from wasting API calls
- [ ] On prompt-too-long, discard the oldest API-round group and retry

### Memory System
- [ ] Multi-layer discovery (global/project/directory-level) covers different use cases
- [ ] Loading order and priority are consistent (later loaded = higher priority)
- [ ] `@include` supports recursive references with circular-reference detection and depth limits
- [ ] Memory content exclusion rules prevent saving derivable information
- [ ] Background extraction is mutually exclusive with the main agent (avoid duplicate writes for the same range)
- [ ] Team memory secret scanning prevents credential leakage

### Worktree Isolation
- [ ] VCS-agnostic hook interface (not tied to a specific VCS)
- [ ] Fail-closed cleanup strategy (refuse deletion when uncommitted changes exist)
- [ ] Symlink large directories (`node_modules`) to avoid disk bloat
- [ ] Stale cleanup matches only temporary patterns (prevent accidental deletion of user worktrees)

### File Checkpoints
- [ ] Three-phase operation pattern prevents React re-render storms
- [ ] Hard-link backup migration (saves disk during cross-session recovery)
- [ ] mtime fast path skips unmodified files
- [ ] Version-name deduplication (files with identical content share the `{hash}@v1` backup name)

## 7.13 Design Philosophy and Architectural Tradeoffs

| Decision | Choice | Alternative | Tradeoff |
|------|------|----------|------|
| Session storage format | Append-only JSONL | SQLite / LevelDB | Zero write-concurrency conflicts and simple crash recovery; queries require full scans |
| 5-layer incremental compaction | Escalate L1→L2→L3→L4→L5 | Single-layer intelligent compaction | Progressive degradation preserves availability; high implementation complexity |
| CLAUDE.md priority | Later loaded = higher priority | Earlier loaded = higher priority | Matches LLM attention behavior (models focus more on the end of the prompt) |
| Team memory deletion | Deletes do not propagate | Bidirectional delete synchronization | Prevents mistakes from affecting the whole team; local deletion affects only local state |
| File checkpoint pattern | sync→async→sync | Pure async | Prevents React re-render storms; small latency overhead |
| autoDream trigger | Dual gates + file lock | Scheduled task | Resource-efficient (runs only when criteria are met); consolidation may be delayed |
| File backup naming | `hash@vN` | Timestamp | Deduplication (identical content shares backups) + deterministic verification |
