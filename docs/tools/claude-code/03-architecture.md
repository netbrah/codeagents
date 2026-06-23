# 3. Claude Code Technical Architecture — Developer Reference

> This article analyzes Claude Code's core architecture: the Bootstrap startup chain, QueryEngine inference loop, 22 Feature Flag DCE gates, Prompt Cache partitioning, five-layer compaction strategy, retry backoff, and more. Most of these patterns are model-agnostic and can be reproduced in agents such as Qwen Code.
>
> **Qwen Code benchmarks**: startup optimization (TCP preconnect), core loop (Mid-Turn Queue Drain, PR#2854 ✓ merged), Prompt Cache partitioning, API retry strategy
>
> **v2.1.82 → v2.1.132 delta** (see [§23](./23-recent-updates.md) for details):
> - Native binaries replaced Bun-packaged JS (Week 16) — startup speed improved further
> - Default model for Max/Team Premium **Opus 4.6 → Opus 4.7** (Week 16), with a new `xhigh` effort level
> - Interactive `/effort` slider replaces command-line args
> - Tool Search / Lazy Loading stabilized in v2.1.76 (lower startup overhead)
>
> The model/version descriptions below are based on the v2.1.81 baseline and should be understood together with the deltas above.

## Why architectural design is more instructive than model capability

Claude Code's model capabilities (Claude Opus/Sonnet) are unique to Anthropic and cannot be replicated. But its **engineering architecture**—startup chain, inference loop, caching strategy, and retry backoff—is a general pattern applicable to Code Agents built on any model.

### Key architectural differences

| Architecture area | Claude Code | Qwen Code | Impact of gap |
|---------|-------------|-----------|---------|
| Startup time | TCP preconnect + keyboard capture (sub-second) | No preconnection optimization | First-input latency |
| Core loop | StreamingToolExecutor + Mid-Turn Drain | Sequential tool execution | Tool execution latency |
| Prompt Cache | Static/dynamic partitioning + tool schema locking | Basic cache | Cache hit rate |
| API retry | 10 attempts + exponential backoff + 529 downgrade + model fallback | Separate retry budget | Reliability |
| Feature management | 22 Feature Flags + build-time DCE | No Feature Flags | Experimental feature management |

### Competitor architecture comparison

| Component | Claude Code | Gemini CLI | Qwen Code |
|------|-------------|-----------|-----------|
| Runtime | Bun → native Rust | Node.js + esbuild | Node.js |
| UI framework | Ink (in-house fork) | Ink (`@jrichman/ink` fork) | Ink (standard) |
| Tool execution | StreamingToolExecutor | Sequential / wave-based parallel (scheduler) | CoreToolScheduler |
| Cache | Prompt Cache static/dynamic partitioning | No Prompt Cache management | Basic cache |
| Startup | TCP preconnect + keyboard capture | Startup Profiler | No optimization |

---

## 1. Runtime and binary format

| Item | Details |
|------|------|
| **Binary format** | ELF 64-bit LSB executable, x86-64, dynamically linked |
| **Size** | ~227 MB (single-file executable) |
| **Packaging method** | Bun-compiled single-file executable (`bun build --compile`), embedding `Bun.embeddedFiles` |
| **Build tool** | Bun bundler (`bun:bundle` compile-time feature gating + runtime APIs such as `Bun.spawn`/`Bun.hash`) |
| **UI framework** | Ink (custom fork embedded in the codebase, not an external dependency) + React 19 + Yoga layout engine |
| **React compiler** | Uses `react/compiler-runtime` (`_c` cache function), not handwritten memoization |
| **Distribution method** | npm `@anthropic-ai/claude-code` or `curl` download to `~/.local/share/claude/versions/` |

### Bun runtime detection

```typescript
// Source: utils/bundledMode.ts#L7-L22
export function isRunningWithBun(): boolean {
  return process.versions.bun !== undefined  // Bun runtime detection
}

export function isInBundledMode(): boolean {  // post-compilation single-file detection
  return typeof Bun !== 'undefined'
    && Array.isArray(Bun.embeddedFiles)
    && Bun.embeddedFiles.length > 0
}
```

Confirmed Bun runtime APIs in use: `Bun.spawn()` (process creation), `Bun.hash()` (fast hashing), `Bun.which()` (command lookup), `Bun.embeddedFiles` (embedded files), `Bun.stringWidth()` (terminal width calculation), and `Bun.serve()` (WebSocket service).

### Embedded native modules

| Module | Purpose |
|------|------|
| `tree-sitter-bash.node` | Bash AST parsing (command security validation) |
| `tree-sitter-typescript.node` | TypeScript AST parsing |
| `tree-sitter-json.node` / `tree-sitter-yaml.node` | JSON/YAML parsing |
| `tree-sitter-kotlin.node` | Kotlin parsing |
| `sharp.node` / `image-processor.node` | Image processing (Sharp library) |
| `audio-capture.node` | Audio capture (voice mode) |
| `file-index.node` | File indexing (code search) |
| `color-diff.node` | Color diff display |
| `resvg.wasm` | SVG rendering (WebAssembly) |

### Build-time feature gating (Dead Code Elimination)

Bun bundler's `feature()` function is evaluated at compile time, and disabled feature branches are removed entirely:

```typescript
// Source: entrypoints/cli.tsx#L1
import { feature } from 'bun:bundle'

// Compile-time condition: in external builds, feature('DUMP_SYSTEM_PROMPT') is false and the entire branch is eliminated by DCE
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') { ... }

// Conditional import: when not matched, require is not executed and the module does not enter the bundle
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

Known feature gates:

| Feature Flag | Controlled content |
|-------------|----------|
| `DUMP_SYSTEM_PROMPT` | `--dump-system-prompt` debug command (ant-only) |
| `DAEMON` | Daemon worker mode |
| `BRIDGE_MODE` | Remote Control bridge mode |
| `VOICE_MODE` | `/voice` voice interaction |
| `COORDINATOR_MODE` | multi-agent Coordinator scheduling |
| `PROACTIVE` / `KAIROS` | Assistant mode (Kairos) |
| `AGENT_TRIGGERS` / `AGENT_TRIGGERS_REMOTE` | Cron scheduler triggers |
| `BASH_CLASSIFIER` | Bash command AI classifier (ant-only) |
| `ABLATION_BASELINE` | Harness ablation experiment baseline |
| `TERMINAL_PANEL` | terminal panel capture tool |
| `WEB_BROWSER_TOOL` | web browser tool |
| `WORKFLOW_SCRIPTS` | workflow scripts |
| `HISTORY_SNIP` | history clipping tool |
| `UDS_INBOX` | Unix Domain Socket inbox |
| `CONTEXT_COLLAPSE` | context collapse |
| `MCP_SKILLS` | MCP skill registration |
| `CHICAGO_MCP` | Computer Use MCP |
| `OVERFLOW_TEST_TOOL` | overflow test tool |
| `MONITOR_TOOL` | monitoring tool |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub Webhook subscriptions |
| `KAIROS_PUSH_NOTIFICATION` | push notifications |

> **Note**: `feature()` is a compile-time macro, not a runtime check. In external builds, large amounts of ant-only code are stripped completely.

---

## 2. Bootstrap startup chain

The startup chain has three levels, each lazily loaded to reduce startup latency:

```
cli.tsx (entrypoint)
  │
  ├─ Fast path (zero import)
  │  --version → console.log(MACRO.VERSION) → exit
  │
  ├─ Dedicated path (minimal import)
  │  --dump-system-prompt  → output system prompt → exit
  │  --claude-in-chrome-mcp → Chrome extension MCP → exit
  │  --chrome-native-host  → Chrome native host → exit
  │  --computer-use-mcp   → Computer Use MCP → exit
  │  --daemon-worker=<kind> → Daemon Worker → exit
  │  remote-control / rc   → Bridge mode → exit
  │
  └─ Main path → import main.tsx
        │
        ├─ Parallel prefetch (module top-level side effect)
        │  ├── profileCheckpoint('main_tsx_entry')  // startup timing
        │  ├── startMdmRawRead()           // MDM policy subprocess
        │  └── startKeychainPrefetch()        // macOS Keychain OAuth + API Key
        │
        ├─ Commander.js command registration
        │  ├── Global options: --model, --print, --continue, --resume, --permission-mode, --mcp-config, --agent ...
        │  └── Subcommands: remote-control, config, mcp, session, ...
        │
        ├─ init() (entrypoints/init.ts, memoized runs only once)
        │  ├── enableConfigs()            // enable configuration system
        │  ├── applySafeConfigEnvironmentVariables() // safe environment variables (before Trust)
        │  ├── applyExtraCACertsFromConfig()     // CA certificates (before Bun TLS cache)
        │  ├── setupGracefulShutdown()        // SIGINT/SIGTERM handling
        │  ├── initialize1PEventLogging()      // first-party event logging (async)
        │  ├── configureGlobalMTLS() + configureGlobalAgents() // mTLS + proxy
        │  ├── preconnectAnthropicApi()       // TCP+TLS preconnect (~100-200ms)
        │  └── initializePolicyLimitsLoadingPromise() // policy limits Promise initialization
        │
        ├─ main.tsx command handler
        │  ├── initializeGrowthBook()        // Feature Flag service
        │  ├── loadPolicyLimits()          // policy limits loading
        │  ├── loadRemoteManagedSettings()      // remote managed settings
        │  └── initializeTelemetryAfterTrust()    // telemetry initialization (delayed until after Trust Dialog)
        │
        └─ launchRepl() → Ink render → REPL loop
```

> Source: `entrypoints/cli.tsx#L33-L106`, `main.tsx#L1-L80`

### Special handling for remote environments

```typescript
// Source: entrypoints/cli.tsx#L9-L14
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  process.env.NODE_OPTIONS = existing
    ? `${existing} --max-old-space-size=8192`
    : '--max-old-space-size=8192'  // container environment has 16GB; subprocess limited to 8GB
}
```

### Ablation experiment baseline

```typescript
// Source: entrypoints/cli.tsx#L21-L26
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  // Disable thinking, compact, auto memory, background tasks, and more
  // Used for harness-science L0 ablation experiments
  for (const k of ['CLAUDE_CODE_SIMPLE', 'CLAUDE_CODE_DISABLE_THINKING', ...]) {
    process.env[k] ??= '1'
  }
}
```

---

## 3. Module architecture

```
claude-code/ # Source directory structure
├── entrypoints/       # entrypoints
│  ├── cli.tsx        # Bootstrap entrypoint (fast-path dispatch)
│  ├── init.ts        # Initialization (config/telemetry/security)
│  └── mcp.ts        # MCP Server entrypoint
├── main.tsx         # main CLI entrypoint (4,683 lines, Commander registration)
├── QueryEngine.ts      # core conversation engine (1,295 lines)
├── Tool.ts          # tool interface definitions & ToolPermissionContext
├── tools.ts         # tool registry (getAllBaseTools / getTools)
├── commands.ts        # command registry (~88 commands, including ant-only)
├── cost-tracker.ts      # cost tracker
│
├── state/          # state management
│  ├── store.ts       # generic Store (35-line pub/sub)
│  ├── AppStateStore.ts   # AppState type definitions
│  └── AppState.tsx     # React Provider + subscription
│
├── services/         # backend services
│  ├── api/
│  │  ├── client.ts     # Anthropic SDK client factory (390 lines)
│  │  ├── claude.ts     # core API calls + streaming (3,419 lines)
│  │  ├── withRetry.ts   # retry and backoff (823 lines)
│  │  ├── errors.ts     # error classification
│  │  ├── logging.ts    # usage tracking
│  │  ├── bootstrap.ts   # Bootstrap data prefetch
│  │  └── promptCacheBreakDetection.ts # cache break detection
│  ├── analytics/
│  │  ├── index.ts     # logEvent() central telemetry
│  │  ├── growthbook.ts   # Feature Flag service
│  │  ├── datadog.ts    # Datadog APM
│  │  └── firstPartyEventLogger.ts # first-party events
│  ├── mcp/         # MCP client/server
│  ├── compact/       # session compaction
│  ├── PromptSuggestion/   # Prompt suggestions + speculation
│  ├── policyLimits/     # policy limits
│  └── remoteManagedSettings/ # remote managed settings
│
├── tools/          # 39 tool implementations
│  ├── BashTool/       # Bash execution + security validation (23 checks)
│  ├── FileEditTool/     # file editing (diff replacement)
│  ├── FileReadTool/     # file reading
│  ├── FileWriteTool/    # file writing
│  ├── GlobTool/       # file search
│  ├── GrepTool/       # content search
│  ├── AgentTool/      # subagent spawning
│  ├── SkillTool/      # Skill invocation
│  ├── WebSearchTool/    # Web search
│  ├── WebFetchTool/     # Web fetching
│  ├── TaskCreateTool/ ...  # task management toolset
│  └── ...
│
├── commands/         # ~88 slash commands (including ant-only)
├── components/        # React/Ink UI components
├── hooks/          # React Hooks
├── skills/          # Skill loading and registration
├── coordinator/       # multi-agent Coordinator
├── bridge/          # Remote Control bridge
├── assistant/        # Assistant (Kairos) mode
├── daemon/          # Daemon Worker
├── memdir/          # Auto Memory system
├── ink/           # Ink rendering engine (diff rendering / flicker prevention)
├── cli/           # non-interactive mode
│  ├── print.ts       # Headless mode (5,594 lines)
│  ├── structuredIO.ts    # SDK structured I/O
│  └── remoteIO.ts      # remote I/O
│
├── utils/          # utility functions
│  ├── permissions/     # permission system
│  ├── sandbox/       # sandbox management
│  ├── hooks/        # Hook event system
│  ├── model/        # model selection and routing
│  ├── settings/       # settings loading
│  ├── bash/         # Bash parsing (shell-quote, tree-sitter)
│  ├── systemPrompt.ts    # system prompt assembly
│  ├── context.ts      # context window management
│  ├── config.ts       # configuration file loading
│  ├── fastMode.ts      # Fast Mode management
│  ├── fileHistory.ts    # file history tracking
│  ├── gracefulShutdown.ts  # graceful shutdown
│  └── startupProfiler.ts  # startup performance profiling
│
└── constants/        # compile-time constants
    ├── prompts.ts      # system prompt templates
    ├── betas.ts       # Beta headers
    ├── oauth.js       # OAuth configuration
    └── tools.js       # tool constants
```

---

## 4. Core runtime loop

### 4.1 QueryEngine — conversation engine

`QueryEngine` is the core class. It implements the `AsyncGenerator` pattern to drive multi-turn conversations:

```
User input → processUserInput() → QueryEngine.submitMessage()
  │
  ├─ Assemble system prompt (fetchSystemPromptParts)
  ├─ Load Memory Prompt (loadMemoryPrompt)
  ├─ Serialize conversation history (normalizeMessagesForAPI)
  ├─ Construct tool list (getTools)
  │
  ├─ API call (query → queryModelWithStreaming)
  │  └─ for await (const part of stream) ← SSE event stream
  │    ├─ message_start   → initialize usage
  │    ├─ content_block_start → create content block (text/tool_use/thinking)
  │    ├─ content_block_delta → accumulate partial content
  │    ├─ content_block_stop → complete content block
  │    ├─ message_delta    → update stop_reason
  │    └─ message_stop    → end stream
  │
  ├─ Tool call handling
  │  ├─ permission check (canUseTool)
  │  ├─ Hook trigger (PreToolUse / PostToolUse)
  │  ├─ Tool execution → collect results
  │  └─ Inject results into conversation history → continue to next turn
  │
  └─ Termination conditions
      ├─ stop_reason === 'end_turn'  → normal completion
      ├─ stop_reason === 'max_tokens' → output truncated
      ├─ user interrupt (AbortController)  → abort
      └─ Token budget exhausted        → trigger compact
```

> Source: `QueryEngine.ts#L1-L100`

### 4.2 Two runtime modes

| Mode | Entrypoint | Purpose |
|------|------|------|
| **Interactive mode** (REPL) | `components/REPL.tsx` → Ink render | terminal interaction, React-driven UI |
| **Headless mode** | `cli/print.ts:runHeadless()` | `--print` pipeline mode, SDK integration, CI |

Interactive mode uses Ink's React render loop, triggering a re-render on every state change. Headless mode has no UI dependency and directly outputs JSON/text streams.

### 4.3 Process model

Claude Code is a **single-process event loop**. Subprocesses are used for:

| Scenario | Method |
|------|------|
| Agent tool calls | `AgentTool` spawn child Claude Code processes |
| MDM policy reading | `plutil` (macOS) / `reg query` (Windows) subprocess |
| Keychain prefetch | macOS Keychain readsubprocess |
| Bash command execution | `child_process.spawn` execute user commands |
| Daemon Worker | `--daemon-worker=<kind>` lightweight worker process |

---

## 5. State management

### 5.1 Store pattern

A minimal pub/sub Store in 35 lines of code:

```typescript
// Source: state/store.ts#L10-L34
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return // skip when references are equal
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 5.2 AppState type

`AppState` is the single source of truth for global state, distributed through React Context:

```typescript
// Source: state/AppStateStore.ts#L89-L158 (abridged)
type AppState = DeepImmutable<{
  settings: SettingsJson          // user settings
  verbose: boolean             // verbose mode
  mainLoopModel: ModelSetting        // current model
  mainLoopModelForSession: ModelSetting   // session-level model
  statusLineText: string | undefined    // status line text
  isBriefOnly: boolean           // Brief mode
  toolPermissionContext: ToolPermissionContext // permission context
  agent: string | undefined         // agent specified by --agent
  kairosEnabled: boolean          // Assistant mode
  // Speculation state
  speculation: SpeculationState       // idle | active
  // Remote Control bridge state
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeConnectUrl: string | undefined
  replBridgeSessionUrl: string | undefined
  // ...
}> & {
  tasks: { [taskId: string]: TaskState }  // background tasks
  agentNameRegistry: Map<string, AgentId>  // agent name registry
  mcp: { clients, tools, commands, resources } // MCP state
  plugins: { enabled, disabled, commands, errors } // plugin state
  agentDefinitions: AgentDefinitionsResult // .claude/agents/ definitions
  fileHistory: FileHistoryState       // file history snapshot
  attribution: AttributionState       // commit attribution tracking
}
```

### 5.3 Speculation state

`SpeculationState` manages speculative execution for Prompt Suggestions:

```typescript
// Source: state/AppStateStore.ts#L58-L77
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string             // speculation ID
      abort: () => void          // cancel function
      startTime: number
      messagesRef: { current: Message[] } // mutable reference — avoids spreading every message
      writtenPathsRef: { current: Set<string> } // overlay write paths
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean        // pipeline mode
    }
```

---

## 6. API communication layer

### 6.1 Multi-provider client factory

`getAnthropicClient()` creates SDK clients for different providers based on configuration:

```
┌─────────────────────────────────────────────────┐
│       getAnthropicClient()        │
│    Source: services/api/client.ts#L88     │
├─────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌────────────────┐│
│ │ Direct  │ │ Bedrock │ │ Vertex AI   ││
│ │ API   │ │ (AWS)  │ │ (GCP)     ││
│ ├──────────┤ ├──────────┤ ├────────────────┤│
│ │Anthropic │ │Anthropic │ │AnthropicVertex ││
│ │SDK direct │ │Bedrock  │ │+ GoogleAuth  ││
│ └──────────┘ └──────────┘ └────────────────┘│
│ ┌──────────────────────────────────────────────┤│
│ │ Azure Foundry                ││
│ │ AnthropicFoundry + DefaultAzureCredential  ││
│ └──────────────────────────────────────────────┘│
└─────────────────────────────────────────────────┘
```

| Provider | Authentication method | Environment variables |
|----------|----------|----------|
| **Direct API** | `ANTHROPIC_API_KEY` or OAuth Token | `ANTHROPIC_API_KEY` |
| **AWS Bedrock** | AWS SDK default credential chain | `AWS_REGION`, `AWS_DEFAULT_REGION` |
| **Google Vertex** | `google-auth-library` | `ANTHROPIC_VERTEX_PROJECT_ID`, `CLOUD_ML_REGION` |
| **Azure Foundry** | API Key or Azure AD | `ANTHROPIC_FOUNDRY_RESOURCE`, `ANTHROPIC_FOUNDRY_API_KEY` |

Client configuration details:

```typescript
// Source: services/api/client.ts#L100-L150
const defaultHeaders = {
  'x-app': 'cli',                  // app identifier
  'User-Agent': getUserAgent(),            // user agent
  'X-Claude-Code-Session-Id': getSessionId(),     // session ID
  ...(containerId ? { 'x-claude-remote-container-id': containerId } : {}),
  ...(clientApp ? { 'x-client-app': clientApp } : {}),
}
// Timeout: default 600 seconds, override with API_TIMEOUT_MS
timeout: parseInt(process.env.API_TIMEOUT_MS || String(600 * 1000), 10)
```

### 6.2 Streaming processing

SSE (Server-Sent Events) event handling loop:

```typescript
// Source: services/api/claude.ts#L1930-L2020 (abridged)
for await (const part of stream) {
  switch (part.type) {
    case 'message_start':    // → initialize partialMessage, update usage
    case 'content_block_start': // → create text/tool_use/thinking/server_tool_use block
    case 'content_block_delta': // → accumulate text_delta / input_json_delta / thinking_delta
    case 'content_block_stop': // → complete content block
    case 'message_delta':    // → update stop_reason, usage
    case 'message_stop':    // → finish
  }
}
```

**Stall detection** (30-second threshold):

```typescript
// Source: services/api/claude.ts#L1934-L1965
const STALL_THRESHOLD_MS = 30_000
if (lastEventTime !== null && timeSinceLastEvent > STALL_THRESHOLD_MS) {
  logEvent('tengu_streaming_stall', { stall_duration_ms, stall_count, ... })
}
```

### 6.3 API endpoints

| Endpoint | Purpose |
|------|------|
| `api.anthropic.com/v1/messages` | core LLM API (Claude model calls) |
| `claude.ai/api/oauth/authorize` | OAuth authentication |
| `claude.ai/api/claude_code/settings` | remote settings retrieval |
| `claude.ai/api/claude_code/policy_limits` | policy limit query |
| `claude.ai/api/claude_code/team_memory` | team memory (per repository) |
| `claude.ai/api/claude_code/metrics` | telemetry reporting |
| `claude.ai/api/ws/speech_to_text/voice_stream` | speech-to-text (WebSocket) |
| `claude.ai/api/claude_cli_feedback` | feedback submission |

---

## 7. Model selection and routing

### 7.1 Model resolution priority

```
1. /model command override (session runtime)
2. --model CLI argument (startup)
3. ANTHROPIC_MODEL environment variable
4. configuration saved in user settings
5. subscription-type default
```

> Source: `utils/model/model.ts#L80-L98`

### 7.2 Default models

| Subscription type | Default model |
|----------|----------|
| Max / Team Premium | Opus 4.6 (1M context if enabled) |
| Pro / other | Sonnet 4.6 |
| 3P Provider (Bedrock/Vertex) | Sonnet 4.5 |
| Ant internal | determined by Feature Flag configuration |

### 7.3 Fast Mode

Fast Mode uses the same model with a higher-throughput inference configuration:

| Mode | Opus 4.6 pricing (per Mtok) |
|------|--------------------------|
| Standard | $5 / $25 (input / output) |
| **Fast Mode** | **$30 / $150** |

Fast Mode integrates with retry:

- Short `retry-after` (<20s): retry in Fast Mode (preserve cache)
- Long `retry-after`: enter cooldown and switch to Standard
- Overage rejection: permanently disable Fast Mode

> Source: `services/api/withRetry.ts#L261-L314`

### 7.4 Advisor (server-side review model)

Advisor is a **server-side tool** (`server_tool_use`) that lets the backend review the full conversation history with a stronger review model:

| Item | Details |
|------|------|
| Trigger method | Autonomously invoked by the model; no user parameters required |
| Backend model | Configured by Feature Flag `tengu_sage_compass` (for example, Opus 4.6 reviewing Sonnet output) |
| Configuration command | `/advisor <model>` (user switches review model) |
| Result type | `advisor_tool_result` (supports encryption) |
| Cost tracking | Independently tracks advisor token usage and recursively adds it to total session cost |

> Source: `utils/advisor.ts`, `commands/advisor.ts`

### 7.5 Extended Thinking

The thinking system is gated at both compile time and runtime:

```
Compile time: feature('ULTRATHINK')   → whether the code is included in the bundle
Runtime: tengu_turtle_carbon    → GrowthBook controls whether it is enabled
Model:  modelSupportsAdaptiveThinking() → Claude 4.6+ only (Opus/Sonnet)
```

Three modes:

| Mode | Behavior |
|------|------|
| `adaptive` | model decides whether to use thinking (default) |
| `enabled` + `budgetTokens` | force enabled with a specified token budget |
| `disabled` | completely disabled |

Configuration: `MAX_THINKING_TOKENS` environment variable or the `alwaysThinkingEnabled` setting.

> Source: `utils/thinking.ts`

---

## 8. Context window management

### 8.1 Context Window resolution

```
1. CLAUDE_CODE_MAX_CONTEXT_TOKENS environment variable (ant-only)
2. model name [1m] suffix → 1,000,000
3. ModelCapability database query
4. Beta Header (CONTEXT_1M_BETA_HEADER)
5. Experimental Treatment (Sonnet 1M experiment)
6. Ant-only codename configuration
7. default: 200,000
```

> Source: `utils/context.ts#L51-L98`

### 8.2 Key constants

| Constant | Value | Purpose |
|------|------|------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200,000 | default context window |
| `COMPACT_MAX_OUTPUT_TOKENS` | 20,000 | maximum output for Compact operations |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | default max_output_tokens cap |
| `ESCALATED_MAX_TOKENS` | 64,000 | escalation value for truncation retries |
| `MAX_OUTPUT_TOKENS_DEFAULT` | 32,000 | default maximum output |
| `MAX_OUTPUT_TOKENS_UPPER_LIMIT` | 64,000 | maximum output limit |

### 8.3 Output token slot optimization

The default `max_output_tokens` is capped at 8,000 (BQ p99 output is only 4,911 tokens), avoiding 8-16× slot over-reservation. Fewer than 1% of requests hit the limit; those requests are retried once with 64,000:

```
First request: max_output_tokens = 8,000 (CAPPED_DEFAULT_MAX_TOKENS)
         ↓ stop_reason === 'max_tokens'
Retry:   max_output_tokens = 64,000 (ESCALATED_MAX_TOKENS)
```

> Source: `utils/context.ts#L18-L25`

### 8.4 1M Context support

Models supporting 1M (compile-time list):

```typescript
// Source: utils/context.ts#L43-L49
export function modelSupports1M(model: string): boolean {
  const canonical = getCanonicalName(model)
  return canonical.includes('claude-sonnet-4') || canonical.includes('opus-4-6')
}
```

Can be disabled with `CLAUDE_CODE_DISABLE_1M_CONTEXT=true` (for HIPAA compliance).

---

## 9. Prompt Caching

### 9.1 Cache scopes

| Scope | Applicable providers | TTL | Purpose |
|------|-------------|-----|------|
| `global` | First-Party only | — | static system prompt shared across all users |
| `org` | all providers | `5m` / `1h` | organization-level temporary cache |

### 9.2 System prompt caching strategy

The system prompt is split into multiple blocks, each with independent cache control:

| Block | Caching strategy |
|----|----------|
| Attribution header | No cache (changes every time) |
| CLI prefix (static instructions) | Org-scoped (sticky across turns) |
| Static content (tool descriptions, etc.) | Global (First-Party only) |
| Dynamic content (git status, etc.) | No cache (updated every turn) |

> Source: `utils/api.ts#L321-L435`

### 9.3 Cache break detection

```typescript
// Source: services/api/promptCacheBreakDetection.ts
// Hash the following content to detect whether the cache has broken:
// - System prompt content
// - Tool schema
// - Model, fast mode, beta headers, effort level
// - Detect changes in each tool description individually
// Minimum miss threshold: 2,000 tokens (no alert below this)
```

---

## 10. Retry and backoff

### 10.1 Retry parameters

| Parameter | Value |
|------|------|
| Default maximum retries | 10 attempts (`DEFAULT_MAX_RETRIES`) |
| Base delay | 500ms (`BASE_DELAY_MS`) |
| Backoff formula | `500ms × 2^(attempt-1)` + 25% jitter |
| Honor `retry-after` | Yes |

> Source: `services/api/withRetry.ts#L52-L55`

### 10.2 Foreground vs background queries

Only **foreground queries** are retried on 529 (overload). Background queries fail immediately to avoid cascading amplification:

```typescript
// Source: services/api/withRetry.ts#L62-L82
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread',     // main conversation loop
  'sdk',            // SDK call
  'agent:custom', 'agent:default', 'agent:builtin', // Agent call
  'compact',          // compaction operation
  'auto_mode',         // safety classifier
  'verification_agent',    // verification agent
  // ...
])
```

### 10.3 529 Opus downgrade

After three consecutive 529 errors, Opus automatically downgrades to Sonnet:

```
Opus request → 529 (×1) → retry → 529 (×2) → retry → 529 (×3)
  → downgrade to Sonnet (prevent capacity cascade)
```

> Source: `services/api/withRetry.ts#L54`, `MAX_529_RETRIES = 3`

### 10.4 Connection error recovery

| Error type | Recovery strategy |
|----------|----------|
| 401/403 (authentication) | clear credential cache and refresh OAuth Token |
| ECONNRESET / EPIPE | disable Keep-Alive connection pool |
| AWS credentials invalid | clear AWS credential cache and refresh |
| GCP credentials invalid | clear GCP credential cache and refresh |

### 10.5 Persistent retry mode (Unattended)

The ant-only `CLAUDE_CODE_UNATTENDED_RETRY` mode is used for unattended sessions:

```typescript
// Source: services/api/withRetry.ts#L96-L98
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000    // maximum backoff 5 minutes
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000  // reset after 6 hours
const HEARTBEAT_INTERVAL_MS = 30_000          // 30-second heartbeat
```

---

## 11. Cost tracking

### 11.1 Pricing table

| Model | Input / Mtok | Output / Mtok | Cache Write / Mtok | Cache Read / Mtok |
|------|:----:|:-----:|:-----:|:-----:|
| Haiku 3.5 | $0.80 | $4 | $1 | $0.08 |
| Haiku 4.5 | $1 | $5 | $1.25 | $0.10 |
| Sonnet 4 / 4.5 / 4.6 | $3 | $15 | $3.75 | $0.30 |
| Opus 4 / 4.1 | $15 | $75 | $18.75 | $1.50 |
| Opus 4.5 / 4.6 (Standard) | $5 | $25 | $6.25 | $0.50 |
| Opus 4.6 (Fast Mode) | $30 | $150 | $37.50 | $3.00 |

> Source: `utils/modelCost.ts#L36-L126`

Web Search: $0.01/request ($10/1,000 requests).

### 11.2 Cost calculation formula

```typescript
// Source: utils/modelCost.ts#L131-L142
cost = (input_tokens / 1M) × inputRate
     + (output_tokens / 1M) × outputRate
     + (cache_read_tokens / 1M) × cacheReadRate
     + (cache_write_tokens / 1M) × cacheWriteRate
     + web_search_requests × $0.01
```

### 11.3 Session cost persistence

Costs are stored in project configuration by session ID and can be accumulated when resuming with `--resume`:

```
project/.claude/config.json → lastSessionId, lastCost, lastAPIDuration, lastSessionMetrics
```

---

## 12. System prompt construction

### 12.1 System prompt sections

The system prompt is dynamically assembled from eight modular sections:

| Section | Core content |
|------|----------|
| `# System` | Runtime behavior: tool execution, permission modes, tag system, context compaction instructions |
| `# Doing tasks` | Software engineering focus, over-engineering warnings, secure coding, backward compatibility avoidance |
| `# Using your tools` | Tool priority (Read>cat, Edit>sed), parallel calls, subagent guidance |
| `# Tone and style` | No emoji, concise, file_path:line_number format |
| `# Output efficiency` | "Get to the point; give the answer before reasoning; skip filler words" |
| `# Executing actions with care` | Reversibility/blast-radius framework, confirmation for risky operations |
| `# Committing changes` | Git safety protocol: never force push, never amend |
| `# auto memory` | Four memory types (user/feedback/project/reference), MEMORY.md index |

### 12.2 Assembly priority

```
Source: utils/systemPrompt.ts#L1-L124
1. --system-prompt override     → complete replacement
2. Coordinator system prompt      → multi-agent scheduling mode
3. Agent system prompt         → specified by --agent
4. Custom system prompt         → user configuration
5. Default system prompt          → standard Claude Code instructions
6. Appended system prompt       → additional context
```

### 12.3 Cache partitioning

The system prompt is split by the `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker into static and dynamic parts. Static parts are cached across users, while dynamic parts (git status, current date, etc.) are updated every turn.

### 12.4 Key safety instructions

```
"Carefully consider the reversibility and blast radius of actions."

"A user approving an action (like a git push) once does NOT mean that they
approve it in all contexts"

"NEVER update the git config"
"NEVER run destructive git commands (push --force, reset --hard, checkout .,
 restore ., clean -f, branch -D) unless the user explicitly requests"
"CRITICAL: Always create NEW commits rather than amending"
```

---

## 13. Message types

### 13.1 Message type hierarchy

| Type | Role | Description |
|------|------|------|
| `UserMessage` | user | user input (text/attachments) |
| `AssistantMessage` | assistant | model response (including tool calls) |
| `SystemMessage` | — | internal system event |
| `ProgressMessage` | — | streaming tool progress |
| `ToolUseSummaryMessage` | — | tool execution summary |
| `TombstoneMessage` | — | placeholder after Compact |

### 13.2 System message subtypes

| Subtype | Purpose |
|--------|------|
| `SystemLocalCommandMessage` | Bash/Shell output |
| `SystemPermissionRetryMessage` | permission retry |
| `SystemCompactBoundaryMessage` | compaction boundary |
| `SystemAPIErrorMessage` | API error |
| `SystemBridgeStatusMessage` | Bridge connection status |
| `SystemScheduledTaskFireMessage` | Cron task trigger |

### 13.3 Content Block types

| Type | Description |
|------|------|
| `text` | text content |
| `thinking` / `redacted_thinking` | thinking process / redacted thinking (content is masked) |
| `tool_use` | client-side tool call |
| `server_tool_use` | server-side tool (such as `advisor` or `web_search`) |
| `tool_result` | tool execution result |
| `mcp_tool_use` / `mcp_tool_result` | MCP tool call and result |

### 13.4 Message flow

```
User input → UserMessage
  ↓
Serialize to ContentBlockParam[] → API
  ↓
Streaming response → accumulate as AssistantMessage
  ↓
Extract tool_use → permission check → execute
  ↓
Tool results → ToolResultBlockParam[] → inject into history
  ↓
Loop until stop_reason === 'end_turn' or budget is exhausted
  ↓
Final response → ToolUseSummaryMessage
```

---

## 14. Tool system

### 14.1 Tool registry

`getAllBaseTools()` returns all available tools, and `getTools()` filters them based on context:

```typescript
// Source: tools.ts#L193-L200
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    // Skip Glob/Grep when embedded search tools are available
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, WebSearchTool,
    // ... conditional tools
  ]
}
```

### 14.2 Core tool inventory

| Category | Tools | Description |
|------|------|------|
| **Files** | FileReadTool, FileEditTool, FileWriteTool | read/edit/write |
| **Search** | GlobTool, GrepTool | file search / content search |
| **Execution** | BashTool, PowerShellTool | Shell command execution |
| **Agents** | AgentTool, SendMessageTool | subagent spawning / message passing |
| **Team** | TeamCreateTool, TeamDeleteTool | team creation/deletion |
| **Tasks** | TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool, TaskOutputTool, TaskStopTool | background task management |
| **Web** | WebSearchTool, WebFetchTool | Web search / fetching |
| **Skill** | SkillTool | user skill invocation |
| **Editors** | NotebookEditTool | Jupyter editing |
| **MCP** | ListMcpResourcesTool, ReadMcpResourceTool | MCP resources |
| **Interaction** | AskUserQuestionTool | ask the user a question |
| **Planning** | EnterPlanModeTool, ExitPlanModeV2Tool | planning mode |
| **Worktree** | EnterWorktreeTool, ExitWorktreeTool | Git Worktree |
| **Scheduling** | CronCreateTool, CronDeleteTool, CronListTool, RemoteTriggerTool | Cron scheduling |
| **Search** | ToolSearchTool | lazy tool search |
| **Configuration** | ConfigTool | runtime configuration modification |
| **LSP** | LSPTool | Language Server Protocol |
| **Internal** | BriefTool, TungstenTool, REPLTool, SleepTool | special-purpose |

### 14.3 Conditional tool loading

Tools are conditionally loaded through three mechanisms:

```typescript
// 1. feature() compile-time gating — disabled tools are removed from the bundle
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool : null

// 2. USER_TYPE runtime gating — ant vs external
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool : null

// 3. isEnabled() method — each tool decides whether it is available
const tools = getAllBaseTools().filter(tool => tool.isEnabled())
```

### 14.4 Tool execution pipeline

```
Model outputs tool_use block
  ↓
ToolPermissionContext.canUseTool() — permission check
  ├─ allow → continue
  ├─ deny → return PermissionDenial
  └─ ask  → show confirmation dialog
  ↓
PreToolUse Hook trigger
  ↓
tool execute() method
  ↓
PostToolUse / PostToolUseFailure Hook trigger
  ↓
Serialize result as tool_result → inject into conversation history
```

---

## 15. Permission and security model

### 15.1 Permission modes

| Mode | Behavior |
|------|------|
| `default` | ask the user for dangerous operations |
| `acceptEdits` | automatically accept file edits; ask for the rest |
| `bypassPermissions` | skip all permission checks |
| `plan` | read-only mode; modifications forbidden |
| `dontAsk` | do not ask; directly deny unauthorized operations |
| `auto` | AI classifier decides automatically (ant-only) |

### 15.2 Permission rule sources

Rule sources are ordered from lowest to highest priority (later sources override earlier ones):

```typescript
// Source: utils/settings/constants.ts#L7-L22 + utils/permissions/permissions.ts#L109-L114
const PERMISSION_RULE_SOURCES = [
  'userSettings',   // 1. user global settings (~/.claude/settings.json) — lowest
  'projectSettings',  // 2. project shared settings (.claude/settings.json)
  'localSettings',   // 3. local settings (.claude/config.local.json, gitignore)
  'flagSettings',   // 4. --settings CLI flag
  'policySettings',  // 5. enterprise policy (distributed by MDM)
  'cliArg',      // 6. CLI arguments (--permission-mode, etc.)
  'command',      // 7. slash command frontmatter
  'session',      // 8. session runtime — highest
]
```

**Decision priority**: within the same source, **Deny > Ask > Allow** (deny rules always take precedence over allow).

> Source: `utils/permissions/permissions.ts#L1071-L1156`

### 15.3 Bash security validation (23 checks)

`bashSecurity.ts` performs 23 security checks on each Bash command:

| ID | Check | Risk |
|:--:|--------|------|
| 1 | `INCOMPLETE_COMMANDS` | incomplete commands |
| 2 | `JQ_SYSTEM_FUNCTION` | jq system function injection |
| 3 | `JQ_FILE_ARGUMENTS` | jq file arguments |
| 4 | `OBFUSCATED_FLAGS` | obfuscated command flags |
| 5 | `SHELL_METACHARACTERS` | Shell metacharacters |
| 6 | `DANGEROUS_VARIABLES` | dangerous environment variables |
| 7 | `NEWLINES` | newlines in commands |
| 8 | `COMMAND_SUBSTITUTION` | `$()`, `${}` and 12 other command substitution patterns |
| 9 | `INPUT_REDIRECTION` | input redirection |
| 10 | `OUTPUT_REDIRECTION` | output redirection |
| 11 | `IFS_INJECTION` | IFS variable injection |
| 12 | `GIT_COMMIT_SUBSTITUTION` | Git commit substitution |
| 13 | `PROC_ENVIRON_ACCESS` | /proc/environ access |
| 14 | `MALFORMED_TOKEN_INJECTION` | malformed token injection |
| 15 | `BACKSLASH_ESCAPED_WHITESPACE` | backslash-escaped whitespace |
| 16 | `BRACE_EXPANSION` | brace expansion |
| 17 | `CONTROL_CHARACTERS` | control characters |
| 18 | `UNICODE_WHITESPACE` | Unicode whitespace characters |
| 19 | `MID_WORD_HASH` | mid-word # character |
| 20 | `ZSH_DANGEROUS_COMMANDS` | 18 Zsh dangerous commands |
| 21 | `BACKSLASH_ESCAPED_OPERATORS` | backslash-escaped operators |
| 22 | `COMMENT_QUOTE_DESYNC` | comment/quote desynchronization |
| 23 | `QUOTED_NEWLINE` | quoted newline |

> Source: `tools/BashTool/bashSecurity.ts#L77-L101`

### 15.4 Zsh-specific protections

Blocks 18 Zsh-specific dangerous commands:

```typescript
// Source: tools/BashTool/bashSecurity.ts#L45-L74
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',  // module loading — entry point to mapfile/sysopen/ztcp
  'emulate',  // -c flag is equivalent to eval
  'sysopen', 'sysread', 'syswrite', 'sysseek', // zsh/system file operations
  'zpty',    // pseudo-terminal command execution
  'ztcp',    // TCP network connection
  'zsocket',  // Unix/TCP socket
  'mapfile',  // zsh/mapfile associative array (invisible file I/O)
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', 'zf_chown', 'zf_mkdir', 'zf_rmdir', 'zf_chgrp',
  // ↑ zsh/files builtins — bypass binary checks
])
```

### 15.5 Command substitution pattern detection

All 12 command substitution patterns are blocked (`<()` and `>()` are detected separately):

| Pattern | Description |
|------|------|
| `<()` / `>()` | process substitution |
| `=()` | Zsh process substitution |
| `=cmd` (word-initial) | Zsh EQUALS expansion (`=curl` → `/usr/bin/curl`) |
| `$()` | command substitution |
| `${}` | parameter substitution |
| `$[]` | legacy arithmetic expansion |
| `~[]` | Zsh parameter expansion |
| `(e:` | Zsh glob qualifier |
| `(+` | Zsh glob command execution |
| `} always {` | Zsh try/always block |
| `<#` | PowerShell comment (defense in depth) |

> Source: `tools/BashTool/bashSecurity.ts#L16-L41`

---

## 16. Command system

### 16.1 Command types

```typescript
// Source: types/command.ts
PromptCommand  // Skill/prompt expanded into a conversation (model-callable)
LocalCommand   // local asynchronous execution (with loader)
LocalJSXCommand // terminal UI command (React-rendered)
```

### 16.2 Command dispatch

```typescript
// Source: utils/handlePromptSubmit.ts#L120-L145
handlePromptSubmit(input)
  → identify /command prefix
  → match command registry
  → processUserInput() execute
```

### 16.3 Partial command list

The source registers about 88 slash commands (including ant-only and feature-gated commands); about 79 are user-visible in external builds:

| Command | Type | Description |
|------|------|------|
| `/help` | Local | help information |
| `/clear` | Local | clear conversation |
| `/compact` | Local | compact conversation |
| `/commit` | Prompt | create Git commit |
| `/review` | Prompt | code review |
| `/model` | Local | switch model |
| `/cost` | Local | show cost |
| `/memory` | Local | manage memory |
| `/config` | Local | configuration management |
| `/mcp` | Local | MCP server management |
| `/resume` | Local | resume session |
| `/remote-control` | Local | remote control |
| `/voice` | LocalJSX | voice mode |
| `/web-setup` | Local | Chrome extension configuration |
| `/init` | Local | initialize CLAUDE.md |
| `/tasks` | Local | task management |

---

## 17. Hook event system

### 17.1 Hook event types

24 Hook events (confirmed in source); users can register Shell command responses in `settings.json`:

| Event | Trigger timing | Matcher |
|------|----------|---------|
| `PreToolUse` | before tool execution | tool_name |
| `PostToolUse` | after tool execution | tool_name |
| `PostToolUseFailure` | after tool execution fails | tool_name |
| `UserPromptSubmit` | user submits input | — |
| `SessionStart` | session initialization | source |
| `SubagentStart` | Agent tool spawns subagent | — |
| `FileChanged` | file change monitoring | filenames |
| `PermissionRequest` | permission dialog display | — |
| `Setup` | repository initialization/maintenance | trigger (init/maintenance) |
| `Elicitation` | MCP interaction request | — |
| `ConfigChange` | configuration change | — |
| `WorktreeCreate` | Worktree creation | — |
| `CwdChanged` | working directory change | — |

> Source: `utils/hooks/hooksConfigManager.ts#L26-L267`

### 17.2 Hook response schema

```typescript
// Source: types/hooks.ts#L50-L166
syncHookResponseSchema = {
  decision?: 'approve' | 'deny' | 'block'  // decision
  reason?: string               // reason
  additionalContext?: string         // additional context
  suppressOutput?: boolean          // suppress output
  hookSpecificOutput?: Record<string, unknown> // specific output
}
```

### 17.3 Event dispatch mechanism

```typescript
// Source: utils/hooks/hookEvents.ts#L61-L81
registerHookEventHandler()  // register global handler (singleton)
emit()            // emit event (queued when no handler is present, max 100)
shouldEmit()         // gating check
emitHookStarted()      // execution start
emitHookProgress()      // execution progress
emitHookResponse()      // completion result (success/error/cancelled)
```

---

## 18. Plugin system

Claude Code supports plugin extensions. Plugins can provide hooks, commands, agents, output styles, and MCP integrations:

### 18.1 Plugin scopes

| Scope | Location | Description |
|--------|------|------|
| `global` | user-level (`~/.claude/plugins/`) | applies to all projects |
| `project` | project-level (`.claude/plugins/`) | current repository only |
| `flag` | session-level (`--plugin-dir` argument) | current session only |

### 18.2 Plugin lifecycle

```
settings.json → installed_plugins.json → loadPluginHooks()
  ├─ loadPluginCommands()  → register slash commands
  ├─ loadPluginAgents()   → register agent definitions
  ├─ loadPluginHooks()    → register Hook callbacks
  └─ file watcher → needsRefresh → /reload-plugins or automatic refresh
```

### 18.3 Marketplace

- Official Anthropic marketplace: auto-updates by default
- Third-party marketplace: no automatic updates
- Installation state is tracked independently: `installationStatus` contains the `pending → installing → installed / failed` state machine

> Source: `utils/plugins/` (15+ files)

---

## 19. Auto-update

| Item | Details |
|------|------|
| **npm channel** | `@anthropic-ai/claude-code` (JavaScript installation) |
| **Native channel** | GCS bucket download of compiled binaries (mainly for ant) |
| **Release channel** | `stable` (default) / `latest` (cutting edge) |
| **Concurrency lock** | 5-minute timeout + stale lock reclamation (TOCTOU-safe) |
| **Version constraints** | `tengu_version_config` minimum version / `tengu_max_version_config` maximum version (Kill Switch) |
| **Disable method** | `isAutoUpdaterDisabled()` configuration check |

> Source: `utils/autoUpdater.ts`

---

## 20. Telemetry system

### 20.1 Telemetry endpoints

| Endpoint | Purpose |
|------|------|
| `api.anthropic.com/api/claude_code/metrics` | primary telemetry reporting |
| Datadog (`http-intake.logs.us5.datadoghq.com`) | APM logs |
| Segment (`api.segment.io`) | user analytics |

### 20.2 Telemetry initialization strategy

Telemetry initialization is delayed until after the Trust Dialog to avoid unauthorized data collection:

```
Trust dialog → remote managed settings loading → initializeTelemetryAfterTrust()
  → OpenTelemetry SDK lazy loading (~400KB)
```

### 20.3 Feature Flag service (GrowthBook)

```typescript
// Source: services/analytics/growthbook.ts
initializeGrowthBook()          // initialize
getFeatureValue_CACHED_MAY_BE_STALE()   // disk cache (may be stale)
getFeatureValue_CACHED_WITH_REFRESH()   // use after asynchronous refresh
```

### 20.4 Telemetry event scale

- **782 unique `tengu_*` event types**
- Main categories:`tengu_agent_*`, `tengu_api_*`, `tengu_auto_mode_*`, `tengu_session_*`, `tengu_tool_*`

### 20.5 Machine ID collection

| Platform | Method |
|------|------|
| macOS | `IOPlatformUUID` (`ioreg -rd1 -c IOPlatformExpertDevice`) |
| Linux | `/etc/machine-id` → `/var/lib/dbus/machine-id` |
| Windows | `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` |
| FreeBSD | `/etc/hostid` → `kenv -q smbios.system.uuid` |

> MAC addresses are **not collected**.

### 20.6 tengu Feature Flags

| Flag | Purpose |
|------|------|
| `tengu_defer_all_bn4` | deferred tool loading |
| `tengu_defer_caveat_m9k` | deferred tool usage warning |
| `tengu_turtle_carbon` | Ultrathink mode |
| `tengu_marble_anvil` | Thinking Edits (clear thinking) |
| `tengu_hawthorn_steeple` | content deduplication |
| `tengu_hawthorn_window` | deduplication window size |
| `tengu_chomp_inflection` | Prompt Suggestion |

### 20.7 Privacy controls

| Variable | Effect |
|------|------|
| `DISABLE_TELEMETRY=true` | completely disable telemetry |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=true` | disable nonessential network traffic |
| `CLAUDE_CODE_ENABLE_TELEMETRY=false` | disable telemetry |

### 20.8 MDM enterprise management

| Platform | Policy path |
|------|----------|
| macOS | plist managed settings (read with `plutil`) |
| Windows | `HKLM\SOFTWARE\Policies\ClaudeCode` / `HKCU\SOFTWARE\Policies\ClaudeCode` |

### 20.9 Total environment variables

**161 `CLAUDE_CODE_*` environment variables**, including 19 `DISABLE_*` switches.

### 20.10 Startup performance profiling

```typescript
// Source: utils/startupProfiler.ts
profileCheckpoint('main_tsx_entry')    // module evaluation start
profileCheckpoint('init_function_start') // init() start
profileCheckpoint('init_configs_enabled') // configuration enabled
profileReport()              // output timing report
```

---

## 21. Key dependencies

| Category | Dependency | Purpose |
|------|------|------|
| **AI SDK** | `@anthropic-ai/sdk` | Anthropic API client |
| **UI** | `ink`, `react` | terminal React rendering |
| **CLI** | `@commander-js/extra-typings` | type-safe command-line parsing |
| **MCP** | `@modelcontextprotocol/sdk` | MCP protocol client |
| **Validation** | `zod` | runtime Schema validation |
| **Parsing** | `@babel/parser` | JS/TS AST parsing |
| **Utilities** | `lodash-es` | utility functions |
| **Styling** | `chalk`, `strip-ansi` | terminal coloring / ANSI stripping |
| **Telemetry** | `@opentelemetry/*`(lazy loading) | OpenTelemetry SDK |
| **gRPC** | `@grpc/grpc-js`(lazy loading) | gRPC transport |
| **Cloud authentication** | `@azure/identity`, `google-auth-library` | Azure / GCP authentication |
| **YAML** | `js-yaml` | YAML parsing |
| **Serialization** | `protobuf` | Protocol Buffer |

---

## 22. Internal codenames

| Codename | Purpose |
|------|------|
| **tengu** | telemetry system (event prefix) |
| **kairos** | Assistant mode(long-running agent) |
| **grove** | internal feature identifier |
| **penguin** | Linux sandbox mode |
| **chicago** | Computer Use MCP |
| **tungsten** | internal debugging tool |
| **bagel** | UI Footer item |
| **companion** / **buddy** | companion animation system |

> **Disclaimer**: The analysis above is based on decompiled source from Q1 2026. Later versions may have changed.
