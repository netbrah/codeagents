# 9. Multi-Agent System — Developer Reference

> Leader-Worker collaboration, three Swarm backends (InProcess/tmux/iTerm2), file-mailbox IPC, task management, and Kairos autonomous mode. The most complex implementation of multi-agent orchestration among Code Agents (~20,500 lines).
>
> **Qwen Code comparison**: Agent Team (PR#2886) and Fork Subagent (PR#2936) are implementing similar capabilities. The InProcess isolation (AsyncLocalStorage), mailbox communication, and task-topology management in this document are key references.

## Why a Multi-Agent System Is Needed

### Problem Definition: The Ceiling of a Single Agent

A single Agent is limited by **serial execution** and a **single context**:

| Scenario | Single Agent | Multi-Agent |
|------|---------|---------|
| "Refactor the auth module + update tests + modify docs" | Serial: refactor → tests → docs, 30 minutes | 3 Agents in parallel, 10 minutes |
| "Apply the same migration across 5 files" | Processes each file serially | 5 worktree Agents in parallel |
| "Continuously monitor CI + fix failures" | Manual user loop | Kairos autonomous mode schedules automatically |
| "Code review needs multiple perspectives" | Review with a single prompt | 4 dimension-specific Agents in parallel + verification Agent |

### Design Evolution

Claude Code's multi-Agent system has gone through three stages:

| Stage | Time | Capability | Architecture |
|------|------|------|------|
| 1. Basic Subagent | 2025 | Agent tool spawns child Agents | Execution in a single process |
| 2. Swarm System | Late 2025 | Leader-Worker + 3 backend types | tmux/iTerm2/InProcess |
| 3. Kairos Autonomy | 2026 | Cron scheduling + proactive behavior | Always-On daemon mode |

### Multi-Agent Comparison with Competitors

| Agent | Multi-Agent Mode | Isolation Mechanism | IPC | Autonomous Scheduling |
|-------|-------------|---------|-----|---------|
| **Claude Code** | Coordinator/Swarm + Fork + Kairos | AsyncLocalStorage + worktree | File mailbox | ✓ Cron + Proactive Tick |
| **Gemini CLI** | A2A Protocol + Subagent | Process isolation | gRPC/REST/JsonRpc | — |
| **Qwen Code** | Arena + Agent Team + Fork (PR#2936) | CoreToolScheduler parallelism | File IPC | — |
| **Copilot CLI** | Plugin Agent + Background Agent | GitHub Actions runner | — | ✓ via GitHub Actions |
| **Cursor** | Background Agent | Cloud sandbox | — | — |
>
> **Counting rule**: Source line counts are based on `wc -l` statistics for TypeScript files.

## 9.1 Architecture Overview

### 9.1.1 Subsystem Panorama

| Subsystem | Total Directory Size | Core File (single-file LOC) | Responsibility |
|--------|-----------|----------------------|------|
| **Swarm Core** | 7,548 LOC | `utils/swarm/inProcessRunner.ts` (1,552) | In-process teammate execution engine |
| **Swarm Backends** | (included in Swarm Core) | `utils/swarm/backends/TmuxBackend.ts` (764) | tmux pane management |
| **Swarm Permissions** | (included in Swarm Core) | `utils/swarm/permissionSync.ts` (928) | File-level permission delegation |
| **Swarm Teams** | (included in Swarm Core) | `utils/swarm/teamHelpers.ts` (683) | Team file management |
| **Agent Tool** | 6,782 LOC | `tools/AgentTool/AgentTool.tsx` (1,397) | Agent entry point (3 execution paths) |
| **Agent Execution** | (included in Agent Tool) | `tools/AgentTool/runAgent.ts` (973) | Subagent execution engine |
| **Agent UI** | (included in Agent Tool) | `tools/AgentTool/UI.tsx` (871) | React progress/status rendering |
| **Agent Definitions** | (included in Agent Tool) | `tools/AgentTool/loadAgentsDir.ts` (755) | Agent definition loading |
| **Mailbox Communication** | 1,183 LOC | `utils/teammateMailbox.ts` (1,183) | File-mailbox message bus |
| **Task Management** | 862 LOC | `utils/tasks.ts` (862) | File-level task list and atomic claiming |
| **Send Message** | 997 LOC | `tools/SendMessageTool/SendMessageTool.ts` (917) + prompt (49) + UI (30) + constants (1) | Cross-agent message routing |
| **Teleport** | 2,020 LOC | `utils/teleport.tsx` (1,225) + 5 UI components (795) | Local ↔ remote session migration |
| **Coordinator Mode** | 369 LOC | `coordinator/coordinatorMode.ts` (369) | Pure-coordinator system prompt |
| **Team Spawning** | 1,093 LOC | `tools/shared/spawnMultiAgent.ts` (1,093) | Teammate spawning across 3 backends |
| **Agent ID** | 99 LOC | `utils/agentId.ts` (99) | Deterministic agent ID format |

### 9.1.2 Leader-Worker Model

```
                    ┌────────────────────────────────────────────┐
                    │           Leader (team-lead)               │
                    │                                            │
                    │  Agent Tool ──▶ spawnTeammate()           │
                    │  SendMessage ──▶ mailbox broadcast         │
                    │  TaskCreate/Update ──▶ task assignment     │
                    │  Coordinator Mode (optional pure coordinator)│
                    └──────┬──────────┬──────────┬──────────────┘
                           │          │          │
                    ┌──────▼──┐ ┌─────▼───┐ ┌───▼──────┐
                    │ Worker 1 │ │ Worker 2 │ │ Worker N  │
                    │         │ │         │ │          │
                    │ Mailbox │ │ Mailbox │ │ Mailbox  │
                    │ Tasks   │ │ Tasks   │ │ Tasks    │
                    │ Worktree│ │ Worktree│ │ (shared) │
                    └─────────┘ └─────────┘ └──────────┘
```

**Three backends**:

| Backend | Suitable Environment | Isolation Level | Communication Method |
|------|---------|---------|---------|
| **In-process** | No terminal panes, or teammate-mode configuration | `AsyncLocalStorage` context isolation | Memory + mailbox files |
| **Split-pane** | tmux / iTerm2 | Process isolation (different terminal panes) | Mailbox files |
| **Separate-window** | tmux | Process isolation (different tmux windows) | Mailbox files |

### 9.1.3 Multi-Agent Lifecycle

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Team Creation│     │ Worker Spawn │     │ Task Execution│    │ Cleanup/     │
│              │     │              │     │              │     │ Disbanding   │
│ TeamCreate   │────▶│ Agent Tool   │────▶│ Parallel     │────▶│ TeamDelete   │
│ config.json  │     │ 3 backends   │     │ execution    │     │ Worktree     │
│ Task dir init│     │ auto-detected│     │ Mailbox comm │     │ cleanup      │
│              │     │ Worktree     │     │ Task claim/  │     │ PID cleanup  │
│              │     │ assignment   │     │ completion   │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

## 9.2 Agent Definition System

### 9.2.1 AgentDefinition Type Hierarchy

Source: `tools/AgentTool/loadAgentsDir.ts` (755 LOC)

```typescript
type AgentDefinition =
  | BuiltInAgentDefinition
  | CustomAgentDefinition
  | PluginAgentDefinition

interface BaseAgentDefinition {
  agentType: string
  whenToUse: string
  tools?: string[]
  disallowedTools?: string[]
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit'
  effort?: string
  permissionMode?: string
  maxTurns?: number
  mcpServers?: Record<string, MCPServerConfig>
  hooks?: Record<string, HookConfig[]>
  skills?: SkillDefinition[]
  color?: string
  isolation?: 'worktree' | 'remote'
  memory?: { scope: 'user' | 'project' | 'local' }
}

interface BuiltInAgentDefinition extends BaseAgentDefinition {
  getSystemPrompt(): string
}

interface CustomAgentDefinition extends BaseAgentDefinition {
  source: SettingSource
  getSystemPrompt(): string
}

interface PluginAgentDefinition extends BaseAgentDefinition {
  source: 'plugin'
  pluginMetadata: PluginMeta
}
```

### 9.2.2 Built-in Agents

Source: `tools/AgentTool/builtInAgents.ts` (72 LOC)

| Agent | Type | Tool Set | Model | Enablement Condition |
|-------|------|--------|------|---------|
| **general-purpose** | General-purpose | All | Default | Always enabled |
| **statusline-setup** | Configuration | Read, Edit | Sonnet | Always enabled |
| **explore** | Search | Glob/Grep/Read/Bash (read-only) | Haiku (external) / Inherit (internal) | `BUILTIN_EXPLORE_PLAN_AGENTS` compile flag + `tengu_amber_stoat` GrowthBook (on by default) |
| **plan** | Planning | Same as explore | Inherit | Same as explore |
| **claude-code-guide** | Guide | Read/WebFetch/WebSearch + Glob/Grep (Ant-native variant uses Bash instead of Glob/Grep) | Haiku | Non-SDK entry points (excludes sdk-ts/sdk-py/sdk-cli) |
| **verification** | Verification | All (editing/writing disallowed) | Inherit | `VERIFICATION_AGENT` compile flag + `tengu_hive_evidence` GrowthBook (off by default) |

**Coordinator agents**: When `COORDINATOR_MODE` is enabled, `getBuiltInAgents()` is replaced with `getCoordinatorAgents()` (dynamic `require('../../coordinator/workerAgent.js')`; the source contains only the compiled artifact path, and the `.ts` source file is not included in the leaked repository). The 6 Agents above are not loaded.

**SDK override**: `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` can disable all built-in Agents in non-interactive mode.

**Feature flag types**:

| Flag | Type | Default | Impact Scope |
|------|------|--------|---------|
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Compile time (`bun:bundle`) | Depends on build | explore / plan |
| `VERIFICATION_AGENT` | Compile time (`bun:bundle`) | Depends on build | verification |
| `COORDINATOR_MODE` | Compile time (`bun:bundle`) | Depends on build | full coordinator set |
| `tengu_amber_stoat` | Runtime (GrowthBook) | `true` | explore / plan |
| `tengu_hive_evidence` | Runtime (GrowthBook) | `false` | verification |

### 9.2.3 Custom Agent Loading

Source: `tools/AgentTool/loadAgentsDir.ts` (755 LOC) + `utils/markdownConfigLoader.ts`

Agent definitions load `.md` files from 4 filesystem locations plus 1 CLI injection path (priority from low to high):

| Source | Path | SettingSource | Description |
|------|------|---------------|------|
| **Built-in** | `builtInAgents.ts` | `builtIn` | Hard-coded, lowest priority |
| **Plugin** | Plugin directory | `plugin` | Third-party extensions |
| **User** | `~/.claude/agents/*.md` | `userSettings` | Personal global |
| **Project** | `.claude/agents/*.md` | `projectSettings` | Team-shared (committed to Git) |
| **CLI injection** | `--agents` argument (JSON) | `flagSettings` | Programmatic injection |
| **Policy** | Administrator configuration directory | `policySettings` | Enforced enterprise policy (highest priority) |

**File loading mechanism**: `loadMarkdownFilesForSubdir('agents', cwd)` scans `.md` files in the directories above. Higher-priority entries with the same `agentType` name override lower-priority entries (`Map.set()`).

**CLI injection path**: `parseAgentsFromJson()` parses JSON-format Agent definitions (the `--agents` argument); fields mirror frontmatter.

Each `.md` file defines an Agent using frontmatter format:

```markdown
---
name: my-custom-agent
description: When you need to perform custom analysis
tools: [Glob, Grep, Read, Bash]
model: sonnet
maxTurns: 10
memory: project
---

You are a custom analysis agent. Focus on...
```

**Required fields**: `name` (becomes `agentType`) and `description` (becomes `whenToUse`). The body content after frontmatter becomes the system prompt. Files missing the `name` field are silently skipped (treated as reference documents).

**Optional fields**: `tools`, `disallowedTools`, `model`, `effort`, `permissionMode`, `maxTurns`, `color`, `background`, `memory`, `isolation`, `mcpServers`, `hooks`, `skills`, `initialPrompt`.

## 9.3 Agent Tool (3 Execution Paths)

### 9.3.1 Input Parameters

Source: `tools/AgentTool/AgentTool.tsx` (1,397 LOC)

```typescript
interface AgentToolInput {
  description: string       // Required, 3-5 word task summary
  prompt: string            // Required, complete task description
  subagent_type?: string    // Agent definition name (omitted = fork path)
  model?: 'sonnet' | 'opus' | 'haiku'  // Model override
  run_in_background?: boolean  // Start asynchronously
  name?: string             // Teammate name (+ team_name = swarm path)
  team_name?: string        // Team name
  mode?: string             // Permission mode
  isolation?: 'worktree' | 'remote'  // Isolation method
  cwd?: string              // Working-directory override
}
```

### 9.3.2 Three-Path Routing

```typescript
// Core routing logic in AgentTool.call() (pseudocode)
if (input.name && input.team_name) {
  // Path 1: Swarm/Teammate spawning
  return spawnTeammate(config, context)
} else if (!input.subagent_type && isForkSubagentEnabled()) {
  // Path 2: Fork subagent (selectedAgent = FORK_AGENT, calls runAgent())
  return runAgent(forkContext)
} else {
  // Path 3: Standard subagent (finds effectiveType, calls runAgent())
  return runAgent(standardContext)
}
```

### 9.3.3 Path Comparison

| Feature | Swarm/Teammate | Fork Subagent | Standard Subagent |
|------|---------------|------------|-----------|
| **Trigger condition** | `name` + `team_name` | `subagent_type` omitted + feature gate | Explicit `subagent_type` |
| **Context** | Fully independent session | Inherits parent conversation (cache-identical prefix) | New independent session |
| **Execution** | Asynchronous, background | Asynchronous, background | Synchronous or asynchronous |
| **Communication** | Mailbox + SendMessage | None (one-shot) | None (one-shot) |
| **Lifecycle** | Persistent (supports multiple interactions) | One-shot | One-shot |
| **Isolation** | Optional worktree/remote | None | None |
| **Output** | `teammate_spawned` | `async_launched` | `completed` / `async_launched` |

### 9.3.4 Fork Subagent

Source: `tools/AgentTool/forkSubagent.ts` (210 LOC)

Fork subagents use `buildForkedMessages()` to construct a **prompt-cache-identical** message prefix:

```typescript
const FORK_AGENT: BuiltInAgentDefinition = {
  agentType: 'fork',
  // Strict rules:
  // - No meta-commentary ("I'm a sub-agent")
  // - Stay within the specified scope
  // - Respond in 500 words or fewer
}
```

**Key design**: fork reuses the parent API message prefix to maximize prompt-cache hit rate and reduce token cost.

### 9.3.5 Agent Memory

Source: `tools/AgentTool/agentMemory.ts` (177 LOC) + `agentMemorySnapshot.ts` (197 LOC)

Subagents can declare persistent memory through the `memory` field, enabling knowledge accumulation across sessions. Agent Memory is independent of the global Memory system described in 07-session.md; it is **local private memory** for each Agent definition.

**Scopes**:

| Scope | Storage Path | Cross-Project | VCS |
|-------|---------|--------|-----|
| `user` | `~/.claude/agent-memory/{agentType}/` | ✅ | ❌ |
| `project` | `{cwd}/.claude/agent-memory/{agentType}/` | ❌ | ✅ (committed to Git) |
| `local` | `{cwd}/.claude/agent-memory-local/{agentType}/` | ❌ | ❌ |

**Definition method**: declare it in the Agent definition frontmatter:

```markdown
---
agentType: security-reviewer
memory: project
---

You are a security review agent. Remember vulnerability patterns...
```

**Prompt injection**: `loadAgentMemoryPrompt()` injects memory instructions into the Agent system prompt:
- **user scope**: keep recommendations general because they are shared across projects
- **project scope**: encourage recording project-specific knowledge and sharing it with the team through Git
- **local scope**: record machine-specific configuration and keep it out of VCS

The memory file is `MEMORY.md` and is stored under the corresponding scope directory. The Agent can read and write this file in each session to accumulate experience.

**Snapshot system**:

Source: `tools/AgentTool/agentMemorySnapshot.ts` (197 LOC)

Project maintainers can provide seed memory through `.claude/agent-memory-snapshots/{agentType}/`:

| Action | Trigger Condition | Behavior |
|------|---------|------|
| `initialize` | No local memory file | Copy `.md` files from snapshot |
| `prompt-update` | Snapshot update time > synced time | Set the `pendingSnapshotUpdate` flag |
| `none` | Already synced or no snapshot | No operation |

**Feature gate**: `feature('AGENT_MEMORY_SNAPSHOT') && isAutoMemoryEnabled()`

**Integration points**:
- Permission system (`utils/permissions/filesystem.ts`): `isAgentMemoryPath()` allows Agent memory paths
- Memory detection (`utils/memoryFileDetection.ts`): classifies Agent memory files for folding/badge display
- Attachment system (`utils/attachments.ts`): searches the corresponding Agent memory directory for `@agent-` mentions
- Tool injection: when `isAutoMemoryEnabled()` is true, file read/write/edit tools are automatically added to the Agent's allowed tools

## 9.4 Swarm Architecture

### 9.4.1 Spawning Engine

Source: `tools/shared/spawnMultiAgent.ts` (1,093 LOC)

**Core flow of `spawnTeammate()`**:

```
spawnTeammate(config, context)
  ├── 1. Generate deterministic agentId: agentName@teamName
  ├── 2. Resolve model: inherit → leader model, undefined → default
  ├── 3. Deduplicate name: append -2, -3 suffixes when already present
  ├── 4. Backend selection:
  │   ├── In-process (AsyncLocalStorage) ─── teammate-mode or no backend available
  │   ├── Split-pane (tmux 30/70 split) ─── tmux or iTerm2 available
  │   └── Separate-window (independent tmux window) ─── legacy mode
  ├── 5. Generate Worktree (if isolation: 'worktree')
  ├── 6. Assign color (round-robin from AGENT_COLORS)
  ├── 7. Register in TeamFile config.json
  ├── 8. Propagate CLI flags (model, settings, permissions)
  └── 9. Send initial prompt to Mailbox
```

**Model resolution chain**: `resolveTeammateModel(inputModel, leaderModel)`
- `'inherit'` → use the leader's model (`leaderModel ?? getDefaultTeammateModel()`)
- `undefined` → `getDefaultTeammateModel(leaderModel)` (provider-aware default that considers the leader model)
- Explicitly specified → use the specified model

**CLI flag propagation**: `buildInheritedCliFlags()` propagates `--dangerously-skip-permissions`, `--permission-mode`, `--model`, `--settings`, `--plugin-dir`, `--chrome`/`--no-chrome`, and `--teammate-mode` from leader to teammate.

### 9.4.2 Backend Registration and Detection

Source: `utils/swarm/backends/registry.ts` (464 LOC)

**Backend detection priority**:

```
detectAndGetBackend():  // Pane backend detection (tmux/iTerm2)
  ├── isInsideTmux() → nested inside tmux → TmuxBackend
  ├── isInITerm2() && isIt2CliAvailable() → native iTerm2 → ITermBackend
  ├── isTmuxAvailable() → external tmux → TmuxBackend
  └── no available backend → throw Error (installation guidance)

isInProcessEnabled():  // Independent in-process determination
  ├── teammate-mode configuration → in-process
  ├── CLAUDE_CODE_TEAMMATE_COMMAND environment variable → in-process
  └── when detectAndGetBackend() throws → in-process fallback
```

### 9.4.3 Tmux Backend

Source: `utils/swarm/backends/TmuxBackend.ts` (764 LOC)

**Layout strategy**: Leader occupies the left 30%; teammates occupy the right 70%. Each teammate is split within the right-side region.

| Operation | tmux Command |
|------|----------|
| Create pane | `split-window` (horizontal/vertical) |
| Hide pane | `break-pane` → `join-pane` (move to hidden window) |
| Show pane | `join-pane` back to main window |
| Color border | `select-pane -P 'bg=...'` |
| Set title | `select-pane -T "name"` |

### 9.4.4 In-Process Backend

Source: `utils/swarm/inProcessRunner.ts` (1,552 LOC) + `spawnInProcess.ts` (328 LOC)

In-process teammates use `AsyncLocalStorage` for context isolation:

```typescript
// Each teammate has an independent context
const context = {
  abortController: new AbortController(),    // Independent interrupt
  taskState: new InProcessTeammateTaskState(), // Independent state
  // Shared: API client, MCP connections
}
```

The **execution engine** `startInProcessTeammate()` includes:
- **Progress tracking**: reports progress to the leader UI
- **Idle notification**: automatically notifies the leader when a turn completes
- **Plan mode approval**: bridges approval through the leader UI
- **Permission handling**: displays permission requests in the leader UI through `leaderPermissionBridge`
- **Mailbox polling**: periodically checks for messages and shutdown requests
- **Auto-compact**: automatically compacts context when it exceeds the threshold
- **Perfetto tracing**: performance tracing

### 9.4.5 Permission Delegation

Source: `utils/swarm/permissionSync.ts` (928 LOC)

File-level protocol for Worker permission requests → Leader approval:

```
Worker                                    Leader
  │                                         │
  │─── PermissionRequest ──────────────────▶│ (write to leader mailbox)
  │                                         │ (show UI confirmation)
  │◀── PermissionResponse ─────────────────│ (write to worker mailbox)
  │                                         │
```

**Supported types**:
- Standard permission requests (file reads/writes, command execution)
- Sandbox permission requests (network access)

**File-lock protection**: all request/response files are protected with `proper-lockfile` to prevent concurrent write conflicts.

### 9.4.6 Team File Management

Source: `utils/swarm/teamHelpers.ts` (683 LOC)

**TeamFile structure**:

```typescript
interface TeamFile {
  leadAgentId: string       // "team-lead@{teamName}"
  leadSessionId: string
  hiddenPaneIds: string[]
  teamAllowedPaths: string[]
  members: TeamMember[]     // Member list
}

interface TeamMember {
  agentId: string           // "agentName@teamName"
  name: string
  model: string
  tmuxPaneId?: string
  backendType: BackendType  // 'tmux' | 'iterm2' | 'in-process'
  isActive: boolean
  mode?: string             // Permission mode
  subscriptions?: string[]  // PR subscriptions
  worktreePath?: string     // Isolated worktree path
}
```

**Storage location**: `~/.claude/teams/{team-name}/config.json`

### 9.4.7 Color Assignment

Source: `utils/swarm/teammateLayoutManager.ts` (107 LOC)

**Palette** `AGENT_COLORS`: red, blue, green, yellow, purple, orange, pink, cyan (8 colors)

Assigned round-robin; the leader always uses the default color. Colors are used for:
- Tmux pane border coloring
- Agent identity in the UI
- Color markers embedded in messages

## 9.5 Mailbox Communication System

### 9.5.1 Architecture

Source: `utils/teammateMailbox.ts` (1,183 LOC)

Each agent has a JSON-format inbox file:

```
~/.claude/teams/{team}/inboxes/{agent}.json
```

**Message structure**:

```typescript
interface TeammateMessage {
  from: string        // Sender agent name
  text: string        // Message content
  timestamp: number   // Timestamp
  read: boolean       // Whether it has been read
  color?: string      // Sender color
  summary?: string    // Message summary
}
```

### 9.5.2 Structured Message Types

The mailbox supports **14 structured protocol message types**:

| Message Type | Direction | Purpose |
|---------|------|------|
| `TeammateMessage` | Bidirectional | Plain-text message |
| `IdleNotificationMessage` | Worker → Leader | Idle notification |
| `PermissionRequestMessage` | Worker → Leader | Permission request |
| `PermissionResponseMessage` | Leader → Worker | Permission response |
| `SandboxPermissionRequestMessage` | Worker → Leader | Sandbox network permission request |
| `SandboxPermissionResponseMessage` | Leader → Worker | Sandbox network permission response |
| `PlanApprovalRequestMessage` | Worker → Leader | Plan approval request |
| `PlanApprovalResponseMessage` | Leader → Worker | Approval result |
| `ShutdownRequestMessage` | Leader → Worker | Shutdown request |
| `ShutdownApprovedMessage` | Worker → Leader | Accept shutdown |
| `ShutdownRejectedMessage` | Worker → Leader | Reject shutdown |
| `TaskAssignmentMessage` | Leader → Worker | Task assignment notification |
| `TeamPermissionUpdateMessage` | Leader → broadcast | Permission-rule change |
| `ModeSetRequestMessage` | Leader → Worker | Switch permission mode |

### 9.5.3 Concurrency Protection

All mailbox operations are protected with `proper-lockfile`:

| Operation | Retry Count | Backoff Strategy |
|------|---------|---------|
| Read | No lock | — |
| Write | 10 attempts | Exponential backoff |
| Mark as read | 10 attempts | Exponential backoff |
| Clear | 10 attempts | Exponential backoff |

### 9.5.4 SendMessage Tool

Source: `tools/SendMessageTool/SendMessageTool.ts` (917 LOC)

**Input**: `{ to: string, summary?: string, message: string | StructuredMessage }`

**Special routing**:

| Target Format | Routing Method |
|---------|---------|
| `agentName` | Direct mailbox delivery |
| `*` | Broadcast to all teammates (skipping self) |
| `uds:/path/to.sock` | Unix domain socket (cross-session) |
| `bridge:session_...` | Remote Control bridge (cross-machine) |

**Automatic recovery**: when sending a message to a stopped agent, execution is automatically resumed through `resumeAgentBackground`.

**Permission model**: Bridge messages require explicit user consent (security check; cannot be auto-approved).

## 9.6 Task Management System

### 9.6.1 Task Data Structure

Source: `utils/tasks.ts` (862 LOC)

```typescript
interface Task {
  id: number               // Monotonically increasing integer
  subject: string          // Task title
  description?: string     // Detailed description
  activeForm?: string      // In-progress description (UI display)
  owner?: string           // Claimant agentId
  status: 'pending' | 'in_progress' | 'completed'
  blocks: string[]         // List of blocked task IDs
  blockedBy: string[]      // Tasks that block this task
  metadata?: Record<string, unknown>
}
```

**Storage location**: `~/.claude/tasks/{team-name}/{id}.json`

### 9.6.2 Atomic Claiming

```typescript
claimTaskWithBusyCheck(taskListId, taskId, claimantId):
  // 1. Acquire the task-list-level file lock
  // 2. Check whether claimant already has an in_progress task
  // 3. Check whether the task is blocked by unfinished tasks in blockedBy
  // 4. Atomically update status → in_progress, owner → claimantId
  // 5. Release the lock
```

**TOCTOU protection**: a task-list-level lock ensures atomicity for "check + claim", preventing multiple agents from claiming the same task simultaneously.

### 9.6.3 Dependency Graph

Tasks support bidirectional dependencies:

```
blockTask(taskListId, fromId, toId):
  // fromId blocks toId
  // fromId.blocks.push(toId)
  // toId.blockedBy.push(fromId)
```

**Pre-claim check**: `claimTaskWithBusyCheck` verifies that all `blockedBy` tasks are completed before claiming.

### 9.6.4 Task List ID Resolution

```typescript
getTaskListId() priority:
  1. CLAUDE_CODE_TASK_LIST_ID environment variable
  2. teamName in teammate context
  3. leader's teamName
  4. sessionId (fallback outside team mode)
```

### 9.6.5 Task Tool Set

| Tool | LOC | Function |
|------|-----|------|
| `TaskCreateTool` | 195 | Create tasks (supports hook interception) |
| `TaskGetTool` | — | Query a single task |
| `TaskListTool` | — | List all tasks |
| `TaskUpdateTool` | — | Update task status/description |

**Feature gate**: `isTodoV2Enabled()` (available in non-interactive mode or when V2 is enabled).

## 9.7 Coordinator Mode

### 9.7.1 Overview

Source: `coordinator/coordinatorMode.ts` (369 LOC)

**Enablement condition**: `feature('COORDINATOR_MODE')` **and** `process.env.CLAUDE_CODE_COORDINATOR_MODE=1`

In coordinator mode, the Leader **never edits code directly** and acts only as a pure orchestrator:
- Assign tasks to Workers
- Synthesize research findings
- Decide whether to continue a Worker conversation or spawn a new Worker

### 9.7.2 Coordinator Tool Set

| Tool | Purpose |
|------|------|
| `Agent` | Spawn Worker (`subagent_type: "worker"`) |
| `SendMessage` | Continue an in-progress Worker conversation |
| `TaskStop` | Stop Worker |
| `subscribe_pr_activity` / `unsubscribe_pr_activity` | PR activity monitoring |

### 9.7.3 Four-Stage Workflow

> **Note**: The following four stages are the recommended workflow in the coordinator system prompt, not a formal code-level architecture pattern. The related descriptions in the source are only informal mentions in prompt comments (such as "research, implementation, or verification").

```
┌───────────────────┐     ┌───────────────────┐
│ 1. Research Stage │     │ 2. Synthesis Stage│
│                   │     │                   │
│ Multiple Workers  │────▶│ Coordinator       │
│ in parallel       │     │ summarizes findings│
│ Search/analyze/   │     │ Identify key files│
│ understand        │     │ Create implementation plan│
└───────────────────┘     └─────────┬─────────┘
                                    │
                          ┌─────────▼─────────┐
                          │ 3. Implementation│
                          │ Stage             │
                          │ Worker performs   │
                          │ modifications     │
                          │ Coordinator       │
                          │ assigns tasks     │
                          └─────────┬─────────┘
                                    │
                          ┌─────────▼─────────┐
                          │ 4. Verification   │
                          │ Stage             │
                          │ Worker runs tests │
                          │ Check types/build │
                          └───────────────────┘
```

**Core principle** (source prompt text): "Parallelism is a superpower" — launching multiple parallel Workers during the research stage can greatly improve efficiency.

### 9.7.4 Continue vs Spawn Decision Matrix

Source: `coordinator/coordinatorMode.ts` — Section 5 "Choose continue vs. spawn by context overlap"

The coordinator system prompt includes a decision table for 6 scenarios:

| Scenario (source text) | Decision | Rationale |
|------|------|------|
| Research explored exactly the files that need editing | **Continue** | Worker already has the files in context AND now gets a clear plan |
| Research was broad but implementation is narrow | **Spawn fresh** | Avoid dragging along exploration noise; focused context is cleaner |
| Correcting a failure or extending recent work | **Continue** | Worker has the error context and knows what it just tried |
| Verifying code a different worker just wrote | **Spawn fresh** | Verifier should see the code with fresh eyes, not carry implementation assumptions |
| First implementation attempt used the wrong approach entirely | **Spawn fresh** | Wrong-approach context pollutes the retry; clean slate avoids anchoring |
| Completely unrelated task | **Spawn fresh** | No useful context to reuse |

**Core principle** (source prompt text): "There is no universal default. Think about how much of the worker's context overlaps with the next task. High overlap → continue. Low overlap → spawn fresh."

**Key rule**: Workers cannot see the coordinator's conversation. Every prompt must be **self-contained** — do not say "based on your previous findings"; include all necessary context instead.

## 9.8 Teleport

### 9.8.1 Overview

Source: `utils/teleport.tsx` (1,225 LOC)

Teleport implements session migration between local and remote Claude Code Runtime (CCR), supporting the Agent tool's `isolation: "remote"` path.

### 9.8.2 Teleport Flow

**Upload (local → remote)**:

```
teleportToRemote():
  1. Haiku generates title and branch name
  2. git bundle creates a code snapshot
  3. Upload to Anthropic API
  4. Remote CCR environment starts
  5. Return remote session ID
```

**Download (remote → local)**:

```
resumeFromTeleport():
  1. Fetch remote session logs
  2. Reconstruct conversation history
  3. checkout remote branch locally
  4. Restore session state
```

### 9.8.3 Teleport UI Components

| Component | LOC | Function |
|------|-----|------|
| `TeleportError.tsx` | 188 | Precondition checks (login, stash) |
| `TeleportResumeWrapper.tsx` | 166 | Resume-loading orchestration |
| `TeleportProgress.tsx` | 139 | 5-step progress indicator |
| `TeleportStash.tsx` | 115 | Git stash dialog |
| `TeleportRepoMismatchDialog.tsx` | 103 | Repository mismatch dialog |
| `useTeleportResume.tsx` | 84 | Resume state-management Hook |

**Progress steps**: `validating` → `fetching_logs` → `fetching_branch` → `checking_out` → `done`

### 9.8.4 Authentication

Uses OAuth for authentication and supports Anthropic account authorization.

## 9.9 Agent ID System

Source: `utils/agentId.ts` (99 LOC)

### 9.9.1 ID Format

| ID Type | Format | Example |
|---------|------|------|
| Agent ID | `{agentName}@{teamName}` | `researcher@my-project` |
| Request ID | `{type}-{timestamp}@{agentId}` | `shutdown-1702500000000@researcher@my-project` |

**Determinism**: the same `agentName` + `teamName` always produces the same `agentId`.

**Constraint**: `@` is a reserved separator — agent names are sanitized to remove the `@` character.

### 9.9.2 ID Purposes

- **Session recovery**: deterministic IDs reconnect teammates after crashes/restarts
- **Mailbox routing**: `~/.claude/teams/{team}/inboxes/{agentName}.json`
- **Task ownership**: the `Task.owner` field uses agentId
- **Debug tracing**: agent identity in Perfetto traces

## 9.10 Integration Points

### 9.10.1 With the Session System

Source: `utils/swarm/reconnection.ts` (119 LOC) + `teammateInit.ts` (129 LOC)

- `computeInitialTeamContext()` injects team context into the initial React state in `main.tsx`
- During teammate initialization, registers a Stop hook and sends an idle notification when the turn ends
- `parentSessionId` associates each teammate with the leader's session UUID

### 9.10.2 With the Tool System

- `AgentTool` is the main Swarm entry point (described in 07-session.md)
- `SendMessage` handles all cross-agent communication
- `TeamCreateTool` / `TeamDeleteTool` manage the team lifecycle
- `TaskCreateTool` / `TaskUpdateTool` manage task lists
- `TaskStopTool` stops running agents
- In-process teammate permission requests are routed to the leader UI through `leaderPermissionBridge`

### 9.10.3 With Remote Control

- Teleport enables the `isolation: "remote"` path
- `RemoteAgentTask` manages the remote agent lifecycle
- The `CLAUDE_CODE_REMOTE` environment variable propagates to pane-based teammates
- Bridge messages are routed through the Remote Control infrastructure

### 9.10.4 With Worktree Isolation

- `isolation: 'worktree'` creates an independent git worktree for the teammate
- Worktree cleanup is triggered automatically when the team is disbanded
- Fail-closed strategy: deletion is refused when uncommitted changes are present

## 9.11 Constants Reference

### 9.11.1 Swarm Constants

Source: `utils/swarm/constants.ts` (33 LOC)

| Constant | Value | Purpose |
|------|---|------|
| `TEAM_LEAD_NAME` | `"team-lead"` | Leader's default name |
| `SWARM_SESSION_NAME` | `"claude-swarm"` | tmux session name |
| `CLAUDE_CODE_TEAMMATE_COMMAND` | Environment variable | Custom teammate launch command |
| `CLAUDE_CODE_AGENT_COLOR` | Environment variable | Custom agent color |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Environment variable | Force plan mode |

### 9.11.2 Mailbox Constants

| Constant | Value | Purpose |
|------|---|------|
| Lock retry count | 10-30 | `proper-lockfile` backoff |
| Backoff range | 5-100ms | Exponential backoff interval |

### 9.11.3 Tmux Layout

| Parameter | Value | Description |
|------|---|------|
| Leader pane ratio | 30% | Left side |
| Teammates pane ratio | 70% | Right side |

### 9.11.4 Agent Color Palette

| Color | Use Case |
|------|---------|
| red, blue, green, yellow, purple, orange, pink, cyan | Round-robin teammate assignment |

## 9.12 Implementer Checklist

### Agent Definitions
- [ ] Typed AgentDefinition hierarchy (built-in/custom/plugin)
- [ ] Support multi-location loading (global/project/plugin/policy)
- [ ] Markdown definition files in frontmatter format
- [ ] Feature gates control built-in agent enablement
- [ ] Agent Memory local persistence (3 scopes + snapshot seeds)

### Multi-Agent Spawning
- [ ] Three-backend architecture (in-process / split-pane / separate-window)
- [ ] Automatic environment detection and fallback
- [ ] Deterministic agent ID (supports crash recovery)
- [ ] CLI flag propagation (model, settings, permissions)
- [ ] Name deduplication (append suffixes)

### Communication System
- [ ] File mailbox (JSON format, lock-protected)
- [ ] 10+ structured message types (covering all protocol needs)
- [ ] Broadcast, direct messages, and cross-session routing
- [ ] Automatic recovery of stopped agents

### Task Management
- [ ] Monotonically increasing integer IDs (high-water mark + file lock)
- [ ] Bidirectional dependency graph (blocks / blockedBy)
- [ ] Atomic claiming (TOCTOU protection)
- [ ] Busy check (prevents executing multiple tasks simultaneously)

### Coordinator Mode
- [ ] Pure orchestrator (never edits code directly)
- [ ] Continue vs Spawn decision matrix
- [ ] Self-contained prompts (Workers cannot see the coordinator conversation)
- [ ] Four-stage workflow (research → synthesis → implementation → verification)

### Isolation and Security
- [ ] Worktree isolation (independent git workspace)
- [ ] Remote isolation (CCR environment teleport)
- [ ] Permission delegation (Worker → Leader UI bridge)
- [ ] Bridge messages require explicit user consent

### Cleanup and Recovery
- [ ] Clean up worktree, PID, and mailbox when the team is disbanded
- [ ] Fail-closed strategy (refuse disbanding when active members exist)
- [ ] Rebuild team context during session recovery

## 9.13 Design Philosophy and Architecture Trade-offs

### 9.13.1 Core Trade-off Matrix

| Decision | Choice | Alternative | Trade-off |
|------|------|----------|------|
| Communication method | File mailbox | Message queue / shared memory | No extra dependencies and cross-process safe; slightly higher latency |
| Spawning backend | Three backends + auto-detection | Single backend | Maximum compatibility; high implementation complexity |
| Coordinator mode | Pure orchestrator | Editable semi-coordinator | Avoids blurred responsibilities between coordinator and Worker; increases Worker spawn cost |
| Agent ID | Deterministic string | UUID | Readable and supports crash recovery; name conflicts require deduplication |
| Task system | File-level + high-water mark | Database | No extra dependencies; concurrency depends on file-lock performance |
| Fork subagent | Shared prompt-cache prefix | Fully independent session | Reduces token cost; context may be too heavy |
| Permission delegation | Leader UI bridge | Auto-approval | High security; increases user interaction |
| Agent Memory | 3-scope file storage | Global KV storage | Clear code-level isolation; file granularity is relatively coarse |
| Context isolation | AsyncLocalStorage | Independent process | Shares API/MCP resources; complex implementation |

### 9.13.2 File Mailbox vs Message Queue

Source: `utils/teammateMailbox.ts`

Claude Code chooses a file-based mailbox instead of IPC message queues (such as Unix sockets or gRPC):

- **Advantages**: zero extra dependencies; tmux/iTerm2 backends naturally cannot share memory; JSON format is easy to debug (directly `cat` the mailbox file); messages are not lost after process crashes
- **Costs**: file locking (`proper-lockfile`) introduces concurrency overhead with 10-30 retries; write latency is affected by the filesystem (usually <1ms); performance bottlenecks in high-frequency messaging scenarios

### 9.13.3 Three-Backend Compatibility Strategy

Source: `utils/swarm/backends/registry.ts` (464 LOC)

The backend detection priority (tmux nested > native iTerm2 > external tmux > in-process fallback) reflects the principle of "maximum compatibility":

- **In-process**: lowest barrier, but shares process resources
- **Split-pane**: best experience (visible multi-pane UI), but depends on tmux/iTerm2
- **Separate-window**: legacy mode, retained for backward compatibility

The cost of implementation complexity: `spawnMultiAgent.ts` (1,093 LOC) is the second-largest file in the multi-agent subsystem, mainly because it must handle differentiated logic across all 3 backends.

### 9.13.4 Prompt Cache Optimization for Fork Subagents

Source: `tools/AgentTool/forkSubagent.ts` (210 LOC)

Fork subagents are the most refined cost-optimization design in Claude Code's multi-agent system:

- **Principle**: `buildForkedMessages()` constructs a message prefix exactly identical to the parent, maximizing the Anthropic API prompt-cache hit rate
- **Constraint**: the Fork Agent system prompt requires "responses in 500 words or fewer" to prevent overly long output from offsetting cache savings
- **⚠️ Inference**: this design implies that Anthropic API prompt-cache billing uses **prefix matching**; the longer the prefix, the greater the cache savings

### 9.13.5 Crash-Recovery Semantics of Deterministic IDs

Source: `utils/agentId.ts` (99 LOC)

The `agentName@teamName` format is not arbitrary — it allows session recovery without persisting an ID mapping:

- **Recovery flow**: after a process crash, the restarted teammate regenerates the agentId from the same `agentName + teamName`, automatically matching unread messages in `~/.claude/teams/{team}/inboxes/{agentName}.json`
- **Cost**: name conflicts require deduplication logic (append -2, -3 suffixes), but this is a one-time cost

### 9.13.6 Pure Coordinator vs Editable Semi-Coordinator

Source: `coordinator/coordinatorMode.ts` (369 LOC)

Coordinator mode chooses "never edit code" instead of "can intervene with edits":

- **Advantages**: avoids blurred responsibilities between the coordinator and Workers; the coordinator focuses on orchestration, and every code modification can be traced to a specific Worker
- **Cost**: even simple tasks (such as changing one configuration value) require spawning a Worker, increasing token cost and latency
- **Design signal**: 4 of the 6 scenarios in the decision matrix choose Spawn fresh, reflecting the preference for "clean context > context reuse"
