# 19. Reference Quick Lookup—Data Structures, Glossary, Entity Relationships

> The most common confusion developers encounter when reading about the Claude Code architecture is not that there are too many features, but that it is **hard to tell which concepts belong to which layer**. This article provides three quick-reference tables: core data structures, a glossary, and an entity relationship diagram.
>
> **Qwen Code comparison**: These conceptual models also apply to Qwen Code—both share the basic architectural paradigm of a tool-calling Agent.
>
> **Acknowledgments**: The conceptual framework in this article references the data-structure and terminology summaries from the [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) project.

## 1. Core Data Structures Quick Reference

### Design Principles

1. **Separate content state from control state**: `messages` and `tool_result` are content state; `turn_count` and `transition` are control state
2. **Separate persistent state from runtime state**: tasks and memory are persistent; permission decisions and MCP connections are runtime state

### Queries and Conversations

| Data structure | Responsibility | Layer |
|---------|------|--------|
| **Message** | Conversation and tool round-trip history | `messages` array |
| **NormalizedMessage** | Normalized messages (adapted for the model API) | API request construction |
| **QueryParams** | External input for starting a query | Query engine entry point |
| **QueryState** | Mutable state that changes across turns | Inside the query engine |
| **TransitionReason** | Explains why the next turn is entered | Query boundary |
| **CompactSummary** | Compressed summary context | Context management |

### Prompts and Input

| Data structure | Responsibility | Layer |
|---------|------|--------|
| **SystemPromptBlock** | A stable prompt fragment | System prompt construction |
| **PromptParts** | Separated prompt fragments (before assembly) | Prompt pipeline |
| **ReminderMessage** | Temporary single-turn/single-mode injection | Message pipeline |

### Tools and Control Plane

| Data structure | Responsibility | Layer |
|---------|------|--------|
| **ToolSpec** | The model's understanding of a tool (name + schema) | Tool registry |
| **ToolDispatchMap** | Routing table from names to handlers | Tool dispatch |
| **ToolUseContext** | Shared environment during tool execution | Tool runtime |
| **ToolResultEnvelope** | Standardized tool return result | Main loop |
| **PermissionRule** | Permission policy (allow/deny/ask) | Permission layer |
| **PermissionDecision** | Structured output from permission gating | Permission layer |
| **HookEvent** | Lifecycle event emitted around the loop | Hook system |

### Persistent Work State

| Data structure | Responsibility | Layer |
|---------|------|--------|
| **TaskRecord** | Persistent work-graph node (goal + state + dependencies) | Task panel |
| **ScheduleRecord** | Rule describing when work should trigger | Cron scheduling |
| **MemoryEntry** | Knowledge retained across sessions | Memory system |

### Runtime Execution State

| Data structure | Responsibility | Layer |
|---------|------|--------|
| **RuntimeTaskState** | Real-time execution slot for background/long-running work | Runtime manager |
| **Notification** | Bridges runtime results back to the main loop | Notification system |
| **RecoveryState** | State used for coherent recovery after failure | Error recovery |

### Team and Platform

| Data structure | Responsibility | Layer |
|---------|------|--------|
| **TeamMember** | Persistent teammate identity | Team configuration |
| **MessageEnvelope** | Structured messages between teammates | Mailbox system |
| **RequestRecord** | Protocol workflows such as approval/close/handoff | Request tracker |
| **WorktreeRecord** | Record of an isolated execution channel | Worktree index |
| **MCPServerConfig** | Configuration for an external capability provider | MCP configuration |

## 2. Glossary

### Core Loop

| Term | Meaning |
|------|------|
| **Query** | One complete "user input → Agent processing → output" process, possibly spanning multiple turns |
| **Turn** | One round within a Query: model response + tool execution |
| **Transition** | Reason for moving from the current turn to the next turn (tool completion/token truncation/compaction/retry/Hook interception) |
| **End Turn** | The model decides to stop (`stop_reason: end_turn`) |
| **Mid-Turn Drain** | Checking whether the user has new input between tool batches |
| **Continuation** | The Query is still alive and should continue advancing (for various reasons) |

### Tool System

| Term | Meaning |
|------|------|
| **Tool Spec** | A tool's schema definition (the JSON Schema sent to the model) |
| **Tool Dispatch** | Routing the `tool_use` returned by the model to the actual handler |
| **Tool Control Plane** | Tool registration/discovery/filtering/permissions—not execution itself |
| **Tool Execution Runtime** | Scheduling/concurrency/progress/merge rules during actual tool execution |
| **ToolSearch** | Lazy loading: the model discovers rarely used tools through search |
| **Streaming Tool Execution** | Parsing and preparing tool execution while the API streams tool calls |

### Context Management

| Term | Meaning |
|------|------|
| **Compact** | Context compaction (trimming old tool outputs + generating a summary) |
| **Cache Edits** | The lightest compaction—only trims old edits inside the cached prefix |
| **Prompt Cache** | API-side cache for matching prefix tokens, reducing cost |
| **Static/Dynamic Boundary** | Boundary between immutable and frequently changing parts of the system prompt |
| **System Reminder** | Temporarily injected `<system-reminder>` tag context |

### Multi-Agent

| Term | Meaning |
|------|------|
| **Subagent** | A child Agent derived from the main Agent |
| **Fork** | A Subagent that inherits the parent Agent's full context |
| **Coordinator** | Coordinator in the Leader-Worker pattern |
| **Swarm** | Multi-Agent collaboration system (InProcess/tmux/iTerm2 backends) |
| **Teammate** | One Agent instance in a Swarm |
| **Mailbox** | File-IPC mailbox system (communication between Teammates) |
| **Kairos** | Always-On autonomous Agent mode |

### Hook System

| Term | Meaning |
|------|------|
| **PreToolUse** | Hook event triggered before tool execution |
| **Prompt Hook** | Hook type that uses an LLM to make decisions (unique to Claude Code) |
| **Agent Hook** | Hook type that creates a temporary Agent for deep validation |
| **hookify** | Mechanism that automatically generates Hook rules from conversations |

## 3. Entity Relationship Quick Reference

```
┌─────────────────────────────────────────────────────────┐
│                    User Layer                           │
│  UserInput → QueryParams → ProcessSlashCommand         │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                 Query Engine Layer                      │
│  QueryState ←→ TransitionReason                         │
│       │                                                 │
│       ├─→ SystemPromptBlock[] → PromptParts             │
│       ├─→ NormalizedMessage[] → API Request             │
│       └─→ ReminderMessage (temporary injection)          │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│               Tool Control Plane                        │
│  ToolSpec → ToolDispatchMap → ToolUseContext             │
│       │                                                 │
│       ├─→ PermissionRule → PermissionDecision            │
│       ├─→ HookEvent (PreToolUse/PostToolUse)            │
│       └─→ ToolResultEnvelope → back to QueryState       │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Persistent State Layer                     │
│  TaskRecord (work graph) ←→ RuntimeTaskState (execution slot) │
│  MemoryEntry (cross-session) ←→ CompactSummary (compaction)   │
│  ScheduleRecord (Cron) ←→ Notification (result bridge)        │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│               Team/Platform Layer                       │
│  TeamMember → MessageEnvelope → Mailbox IPC              │
│  WorktreeRecord → isolated execution channel             │
│  MCPServerConfig → CapabilityRoute                       │
└─────────────────────────────────────────────────────────┘
```
