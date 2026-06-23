# 8. Remote Control — Developer Reference

> Remotely control a local terminal Agent from a phone or browser. The session always executes locally; the remote side is only an interaction window. This document analyzes the WebSocket/SSE Bridge architecture and defense-in-depth security design.
>
> **Qwen Code comparison**: Qwen Code currently has no remote-control capability. The Bridge architecture here can serve as an implementation reference, but its priority is relatively low (P2-P3).

## Why Remote Control Is Needed

### Problem Definition

Code Agents run in a developer's terminal, but developers are not always sitting at that computer:

| Scenario | Pain Point | Remote Control Solution |
|------|------|----------------------|
| An Agent is running a long task and the developer goes to a meeting | Cannot check progress or approve actions from a phone | Remote control from a mobile browser |
| Running an Agent on a remote server over SSH | Terminal disconnect = session lost | The Bridge keeps the session alive and any device can reconnect |
| Remote pair-programming demo | Cannot share the terminal with colleagues | Real-time sharing through a link |

### Design Decision: Local Execution + Remote UI

Key architecture decision: **the session always executes locally**; the remote side is only an interaction window. This means:
- Code does not leave the local machine (security)
- Network disconnection does not affect execution (robustness)
- The remote side is a "view", not a "controller" (controllable permissions)

### Competitor Remote Capability Comparison

| Agent | Remote Capability | Architecture | Security Model |
|-------|---------|------|---------|
| **Claude Code** | Remote Control + Claude Code on the Web | WebSocket/SSE Bridge, local execution | Bearer token + session key |
| **Gemini CLI** | No standalone remote control | — | — |
| **Qwen Code** | None | — | — |
| **Copilot CLI** | Background Agent (GitHub Actions) | Cloud execution | GitHub authentication |
| **Cursor** | No standalone remote capability (inherits VS Code Remote) | VS Code SSH/Tunnel | VS Code authentication |

## 8.1 Overview

Remote Control is Claude Code's cross-device session bridging feature. Among 19 mainstream AI programming Agents, it is **unique to Claude Code** ([feature matrix](../../comparison/features.md)). It solves a core developer pain point: after starting a long-running terminal agent task, the developer may need to leave the workstation while continuing to monitor or intervene.

Core capabilities:

- **Local execution, remote control**: The session runs on the local machine, with the full file system, MCP servers, toolchain, and project configuration available
- **Real-time bidirectional synchronization**: Conversations stay synchronized across all connected devices, and messages can be sent from any device
- **Automatic reconnection**: The connection automatically recovers after laptop sleep or brief network interruption
- **Zero inbound ports**: All communication uses outbound HTTPS; no firewall ports need to be opened

## 8.2 Prerequisites

| Requirement | Details |
|------|------|
| **Version** | Claude Code v2.1.51+ (check with `claude --version`) |
| **Subscription** | Pro / Max / Team / Enterprise plans. **API Key is not supported** |
| **Authentication method** | Must sign in through claude.ai OAuth (`/login`); API Key and `claude setup-token` are not supported |
| **Workspace trust** | Run `claude` at least once in the project directory to accept the workspace trust dialog |
| **Team/Enterprise** | An administrator must enable the Remote Control toggle in `claude.ai/admin-settings/claude-code` |

## 8.3 Three Startup Methods

### 8.3.1 Method 1: Server Mode (Dedicated Service)

```bash
claude remote-control
claude remote-control --name "My Project"
```

Runs a dedicated Remote Control server and waits for remote connections. The terminal displays the session URL; press the **Space key** to show the QR code.

**Available options:**

| Option | Description |
|------|------|
| `--name <name>` | Custom session title, visible in the session list |
| `--spawn <mode>` | `same-dir` (default): all sessions share the CWD; `worktree`: each session gets an independent Git worktree |
| `--capacity <N>` | Maximum number of concurrent sessions (default: 32) |
| `--verbose` | Detailed connection/session logs |
| `--sandbox` / `--no-sandbox` | Enable/disable file-system and network sandboxing (disabled by default) |

### 8.3.2 Method 2: Interactive Session + Remote Control

```bash
claude --remote-control              # or --rc
claude --remote-control "My Project" # with a name
```

Starts a full interactive session in the terminal while also allowing control from remote devices. Local input is still available, and remote clients can connect at the same time.

### 8.3.3 Method 3: Enable from an Existing Session

```
/remote-control          # or /rc
/remote-control My Project
```

Enables Remote Control in the current conversation. This method does not support the `--verbose`, `--sandbox`, or `--no-sandbox` options.

> **Command type**: The `/remote-control` slash command type is `local-jsx` ([command details](./02-commands.md)); it renders the remote-control configuration UI and starts the connection to claude.ai/code.

## 8.4 Connecting from Other Devices

Three connection methods:

| Method | Operation |
|------|------|
| **Session URL** | Open it in any browser; it redirects to `claude.ai/code` |
| **QR code** | Scan it with the Claude App (in Server mode, press the Space key to toggle display) |
| **Session list** | Find it by name in `claude.ai/code` or the Claude App; online Remote Control sessions show a **computer icon with a green dot** |

**Session title priority** (descending):
1. Name specified by the `--name` / `--remote-control` / `/remote-control` argument
2. Title set through `/rename`
3. Content of the last meaningful message in the conversation history
4. The first prompt sent by the user

## 8.5 Global Default Enablement

Run `/config` in Claude Code and set **"Enable Remote Control for all sessions"** to `true`. After that, every interactive process automatically registers a remote session. If multiple concurrent sessions are needed within one process, use **Server mode** with `--spawn`.

## 8.6 Technical Architecture

> The following combines [Anthropic official documentation](https://docs.anthropic.com/en/docs/claude-code/remote-control), source analysis (`bridge/` directory: 12,613 lines + `remote/` directory: 1,127 lines + about 21,000 relevant lines in `utils/` and `entrypoints/`), GitHub Issues community feedback, and Anthropic engineering blog posts.

### 8.6.1 Three-Party Relay Architecture

Remote Control uses a **Three-Party Relay** architecture, with the Anthropic API acting as the message broker:

```
┌──────────────┐                              ┌───────────────────────┐
│ Local        │   1. Register session        │    Anthropic API      │
│ Terminal     │      (POST /v1/...)          │    (message relay/    │
│ (Claude Code)│ ──────────────────────────→   │     broker)           │
│              │   Get JWT + session creds    │                       │
│              │                                │  claude.ai/code       │
│              │   2. WebSocket (v1) / SSE (v2)│  session registration │
│              │ ←────────────────────────────→ │  + message routing    │
│              │   Bidirectional message flow  │                       │
└──────────────┘                                └───────────┬───────────┘
                                                            │
                                                 3. WebSocket/HTTPS │
                                                            │
                                                ┌───────────▼───────────┐
                                                │ Browser / Mobile App   │
                                                │ (claude.ai/code)       │
                                                └───────────────────────┘
```

**Two-generation transport architecture** (confirmed by source):

The source reveals that Claude Code internally implements two generations of transport protocols, with the server selecting one dynamically per session:

| Dimension | v1 (HybridTransport) | v2 (SSETransport + CCRClient) |
|------|----------------------|-------------------------------|
| **Read channel** | WebSocket receives messages | SSE (Server-Sent Events) receives messages |
| **Write channel** | POST to the Session-Ingress endpoint | POST to CCR `/worker/*` endpoints |
| **Authentication** | OAuth token | JWT (with `session_id` + `worker` role) |
| **Selection condition** | Default | Server switches via the `secret.use_code_sessions` flag; the `CLAUDE_BRIDGE_USE_CCR_V2` environment variable can force enablement |
| **Source** | `bridge/replBridge.ts` | `bridge/remoteBridgeCore.ts` |

**Data flow**:

| Phase | v1 Flow | v2 Flow | Confidence |
|------|---------|---------|--------|
| **Registration** | `registerBridgeEnvironment()` -> `environment_id` + `environment_secret` | `createCodeSession()` -> `sessionId` -> `fetchRemoteCredentials()` -> JWT | Confirmed by source |
| **Connection** | Long-lived WebSocket + POST writes | Long-lived SSE + POST writes | Confirmed by source |
| **Authentication refresh** | Injected through the `refreshHeaders` callback after OAuth token refresh | Refreshed through the `/bridge` endpoint 5 minutes before JWT expiration, with epoch bump | Confirmed by source |
| **Disconnection/reconnection** | Exponential backoff (2s -> 60s, give up after 15 minutes), reconnect-in-place or create a new session | SSE 401 triggers OAuth refresh + credential refetch | Confirmed by source |
| **Heartbeat** | `keep_alive` message (default 120-second interval) | `heartbeatIntervalMs` + `heartbeatJitterFraction` | Confirmed by source |

| Aspect | Details | Confidence |
|------|------|--------|
| **Local -> server** | Outbound WebSocket/SSE; no local inbound port is opened | Confirmed by source |
| **Remote client -> server** | Browser/App connects to claude.ai infrastructure (WebSocket) | Confirmed by source |
| **Message routing** | Anthropic servers relay messages bidirectionally between the remote client and local session | Officially confirmed |
| **Transport security** | TLS encryption throughout, same as normal Claude Code sessions | Officially confirmed |
| **Credential system** | Multiple short-lived credentials (JWT + OAuth), each limited to a single purpose and expiring independently | Confirmed by source |

#### 8.6.1.1 Three-Layer Architecture Split: Control Plane / Data Plane / Local State Plane

From an implementer's perspective, Remote Control can be split into three subsystems with clear responsibilities (corresponding in source to the unified `ReplBridgeHandle` interface and `BridgeCoreParams` dependency injection):

**Control Plane** — session registration, eligibility checks, credential management, and policy enforcement:

| Component | Responsibility | Evidence Source |
|------|------|----------|
| Session registration | v1: `registerBridgeEnvironment()` -> `environment_id` + `environment_secret`; v2: `createCodeSession()` -> JWT | Source: `bridge/replBridge.ts` |
| Eligibility check | The `admin_requests/eligibility` endpoint determines whether the user can use RC (affected by subscription type, administrator policy, and organization policy) | Source: `services/api/adminRequests.ts` + `bridge/bridgeEnabled.ts` |
| Credential refresh | v1: OAuth refresh -> `refreshHeaders` callback; v2: call the `/bridge` endpoint 5 minutes before JWT expiration and bump epoch to prevent double refresh | Source: `bridge/remoteBridgeCore.ts` |
| Policy enforcement | Query the `policy_limits` endpoint for the organization-level RC toggle; Team/Enterprise administrator gate | EVIDENCE.md |
| Configuration delivery | GrowthBook feature gates (`tengu_bridge_poll_interval_config`, `tengu_bridge_initial_history_cap`, etc.) dynamically adjust runtime parameters | Source: `bridge/bridgeMain.ts` |
| PID file management | `~/.claude/sessions/{pid}.json` tracks concurrent sessions, including `kind` (interactive/bg/daemon/daemon-worker) and `status` (busy/idle/waiting) | Source: `utils/concurrentSessions.ts` |

**Data Plane** — message transport, bidirectional synchronization, and streaming responses:

| Component | Responsibility | Evidence Source |
|------|------|----------|
| v1 transport | WebSocket reads + POST writes to Session-Ingress; exponential-backoff reconnection (2s->60s, give up after 15 minutes) | Source: `bridge/replBridge.ts` |
| v2 transport | SSE reads + POST writes to CCR `/worker/*`; JWT authentication + automatic refresh | Source: `bridge/remoteBridgeCore.ts` |
| Message protocol | JSON-lines format, 21 `control_request` subtypes + standard SDK messages | Source: `entrypoints/sdk/controlSchemas.ts` |
| History refresh | `initialHistoryCap` (default 200 messages, GrowthBook-adjustable) limits initial history push; FlushGate prevents interleaving history messages with real-time messages | Source: `bridge/replBridge.ts` |
| Echo deduplication | Two-layer UUID protection: `initialMessageUUIDs` + `recentPostedUUIDs` (2,000-entry ring buffer) to prevent message echoes | Source: `bridge/replBridge.ts` |
| Conversation persistence | Session-Ingress API + optimistic concurrent writes (`Last-Uuid` header + automatic recovery from 409 Conflict) | Source: `services/api/sessionIngress.ts` |
| WebSocket heartbeat | `keep_alive` message, default 120-second interval (`session_keepalive_interval_v2_ms`) | Source: `bridge/replBridge.ts` |

**Local State Plane** — Redux state machine, environment variables, and runtime state:

| Component | Responsibility | Evidence Source |
|------|------|----------|
| Redux state machine | 13 `replBridge*` fields manage the bridge lifecycle (enabled/connected/active/reconnecting, etc.) | Source: `bridge/replBridge.ts` |
| Client type detection | 3-layer token priority chain: env var -> FD -> well-known file | Source: `utils/sessionIngressAuth.ts` |
| Environment-variable configuration | 14+ environment variables control RC behavior (authentication mode, network proxy, sandbox, remote environment, etc.) | Official documentation + source |
| initReplBridge | Core bridge layer, with 7 callback functions injected through `BridgeCoreParams` | Source: `bridge/initReplBridge.ts` |
| Crash recovery | Bridge pointer file (`{sessionId, environmentId, source}`), allowing recovery on the next startup after a process crash | Source: `bridge/replBridge.ts` |
| Sleep detection | `setTimeout` timeout threshold of 60+ seconds -> system sleep detection -> reset error budget | Source: `bridge/replBridge.ts` |
| Session activity tracking | Refcount heartbeat timer (30-second interval), distinguishing `api_call` / `tool_exec` activity | Source: `utils/sessionActivity.ts` |

> **Design insight**: The three-layer separation means **control-plane changes do not affect message transport** (for example, modifying transport policy does not require changing the message format), and the **local state plane is independent of the network** (after a process crash, part of the state can be rebuilt from the pointer file and conversation history). In source, `BridgeCoreParams` uses dependency injection; core logic does not directly import `bootstrap/state` or `sessionStorage`, achieving zero coupling between modules.

### 8.6.2 Session Lifecycle

```
┌─────────┐     ┌───────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│Register │ ──→ │ Waiting   │ ──→ │ Active   │ ──→ │ Idle/     │ ──→ │ Expired/ │
│         │     │           │     │          │     │Disconnected│     │ Exit     │
└─────────┘     └───────────┘     └──────────┘     └───────────┘     └──────────┘
  - OAuth auth    - Show URL/QR     - Bidirectional     - Automatic        - Process exits
  - API register  - Wait for client   message sync        reconnect attempts - Clean session files
                  - Server mode can - Tool calls can be  - WS/SSE disconnect - Archive session
                    accept clients    approved remotely - Sleep detection
                                     - keep_alive         recovery
                                       heartbeat         - Reconnect-in-place or new session
```

**Session states in source** (`server/types.ts`): `'starting' | 'running' | 'detached' | 'stopping' | 'stopped'`

**Phase details**:

| Phase | Trigger | Behavior |
|------|------|------|
| **Registration** | Start `claude remote-control` or `/rc` | Uses a full-scope OAuth token to register the session with the Anthropic API and obtain `SESSION_ACCESS_TOKEN` |
| **Waiting for connection** | After successful registration | The terminal shows the session URL and QR code. The local process waits for a remote client connection (v1 waits for work assignment through polling; v2 waits through SSE) |
| **Active** | Remote client connects | Bidirectional message sync: remote instructions are routed to local execution, and local output is pushed to the remote side in real time. Permission requests are bridged to remote approval through `control_request` (`can_use_tool` subtype) | Confirmed by source |
| **Idle/disconnected** | Network interruption or laptop sleep | Automatically attempts reconnection (exponential backoff, 2s->60s). **Sleep detection**: a `setTimeout` timeout of 60+ seconds indicates system sleep and resets the error budget. v1 strategy: reconnect-in-place (`reuseEnvironmentId`) or create a new session; v2 strategy: SSE 401 triggers OAuth refresh + credential refetch | Confirmed by source |
| **Expired/exit** | Timeout or process termination | v1: `stopWork()` + `archiveSession()` + cleanup of PID files; v2: transport close + archive. In Perpetual mode, no result is sent, allowing the backend TTL (300s) to expire and requeue | Confirmed by source |

### 8.6.3 Session Files and Local Storage

| Path | Contents | Plane | Lifecycle | Recoverability | Source |
|------|------|--------|----------|----------|------|
| `~/.claude/sessions/{pid}.json` | Session metadata: `pid`, `sessionId`, `cwd`, `startedAt`, `kind` (interactive/bg/daemon/daemon-worker), `entrypoint`, `name`, `status` (busy/idle/waiting), `logPath`, `agent`, `messagingSocketPath`, `bridgeSessionId` | Control plane + local state plane | One file per interactive process; remains after process exit but is meaningless | The file remains after process exit. `countConcurrentSessions()` cleans up stale PID files (except on WSL). **Process restart does not automatically recover it** | Source: `utils/concurrentSessions.ts` |
| `/tmp/cc-socks/*.sock` | Unix domain socket for local inter-process messaging (such as UI bridge and multi-client reuse) | Data plane (local IPC) | Created/destroyed with the process; invalid once the process exits | Not recoverable: Unix socket files become invalid when the process exits; reconnecting requires establishing a new socket | GitHub Issues |
| `~/.claude/projects/<project-hash>/` | Session conversation history (`.jsonl` format), containing the complete conversation stream; automatically cleaned after `cleanupPeriodDays` (default 30 days) | Data plane (persistence) | Independent of the Remote Control lifecycle; shared with normal sessions | Recoverable: conversation history persists on disk and can be used by `/continue` or `/teleport` to restore context | [EVIDENCE.md](./EVIDENCE.md) |

> **Implementer notes**:
> - PID file naming (`{pid}.json`) means **one remote session per OS process**, not one state file per bridge session. Child processes created by Server mode `--spawn` each have their own PID file
> - The `messagingSocketPath` field is stored in the PID file, indicating that the Unix socket path is a **server/client negotiation result**, not hard-coded
> - `bridgeSessionId` and `sessionId` are different concepts: `sessionId` is the local session ID, while `bridgeSessionId` is the bridge ID assigned by the relay server
> - In source, `registerCleanup` ensures the PID file is unlinked when the process exits; the filename is strictly validated with `/^\d+\.json$/` to avoid deleting non-PID files by mistake
> - The **Bridge pointer file** (`bridge/bridgeMain.ts`) is independent of the PID file and is used for crash recovery: it contains `{sessionId, environmentId, source}` and refreshes mtime every hour

### 8.6.4 `--spawn` Multi-Session Architecture

Server mode supports managing multiple concurrent remote sessions through the `--spawn` option:

| Mode | Behavior | Use Case |
|------|------|----------|
| `same-dir` (default) | All sessions share the current working directory | Multiple people remotely control different tasks in the same project |
| `worktree` | Each on-demand session gets an independent Git worktree | Parallel development tasks that need file isolation |

**Runtime switching**: In Server mode, press `w` to dynamically switch the spawn mode.

**Capacity control**: `--capacity <N>` limits the maximum number of concurrent sessions (default 32), preventing resource exhaustion. The `capacityWake` signal interrupts at-capacity sleep when a session completes, accepting new work immediately.

**Session tracking** (Source: `bridge/bridgeMain.ts`): The runtime maintains 9 Map/Set data structures:

| Data Structure | Purpose |
|----------|------|
| `activeSessions: Map<string, SessionHandle>` | Active session handles |
| `sessionStartTimes: Map<string, number>` | Session start times |
| `sessionWorkIds: Map<string, string>` | Session-to-work-item mapping |
| `sessionIngressTokens: Map<string, string>` | Session-to-Ingress-token mapping |
| `sessionTimers: Map<string, Timeout>` | Session timeout timers |
| `completedWorkIds: Set<string>` | Completed work items (deduplication) |
| `sessionWorktrees: Map<string, WorktreeInfo>` | Worktree isolation information |
| `timedOutSessions: Set<string>` | Timed-out sessions |
| `v2Sessions: Set<string>` | v2 transport sessions |

> **Difference from CCR (Claude Code Remote)**: `--spawn` creates **local multi-sessions** (isolated by worktree), while the `RemoteTrigger` tool used by `/schedule` creates **cloud-isolated sessions** (CCR) that run independently on Anthropic infrastructure ([command details](./02-commands.md)).

### 8.6.5 Defense-in-Depth Security Model

Remote Control uses a multilayer security architecture:

| Layer | Mechanism | Description |
|------|------|------|
| **1. Authentication gate** | claude.ai OAuth full-scope token | API Key, `setup-token`, Bedrock/Vertex/Foundry are all rejected |
| **2. Administrator gate** | `claude.ai/admin-settings/claude-code` toggle | Disabled by default for Team/Enterprise; compliance configuration can block enablement |
| **3. Credential isolation** | v1: OAuth + environment_secret; v2: JWT (`session_id` + `worker` role), multiple short-lived credentials with single-purpose scopes | Source: `bridge/replBridge.ts`, `bridge/remoteBridgeCore.ts` |
| **4. Network isolation** | Outbound-only WebSocket/SSE + POST, zero inbound ports | Significantly reduces the network exposure surface |
| **5. Transport encryption** | TLS throughout | Same as normal Claude Code sessions |
| **6. Optional sandbox** | `--sandbox` enables file-system + network isolation | Disabled by default; available in Server mode |
| **7. Permission bridge** | `can_use_tool` control_request is relayed to the remote client for approval, and the response returns through `control_response` | Source: `bridge/bridgeMessaging.ts` |
| **8. Session Ingress authentication** | 3-layer token priority chain: env var -> FD -> well-known file; session key uses Cookie, JWT uses Bearer token | Source: `utils/sessionIngressAuth.ts` |
| **9. Echo deduplication** | Two-layer UUID protection (`initialMessageUUIDs` + 2,000-entry ring buffer) prevents message echoes | Source: `bridge/replBridge.ts` |
| **10. Safety classifier** | Auto mode uses two-layer defense (server-side probe + client-side classifier) | [Engineering blog](https://anthropic.com/engineering/claude-code-auto-mode), powered by Sonnet 4.6 |

**Telemetry coupling phenomenon**: Setting `DISABLE_TELEMETRY=1` causes Remote Control registration to fail ([GitHub #41189](https://github.com/anthropics/claude-code/issues/41189)), appearing as a failed eligibility check. Current evidence is insufficient to confirm that the root cause is "eligibility checks use the telemetry channel"; it is marked as **suspected implementation coupling**.

### 8.6.6 Related API Endpoints

| Endpoint | Purpose | Source |
|------|------|------|
| `claude.ai/api/claude_code/settings` | Fetch remote settings | [EVIDENCE.md](./EVIDENCE.md) |
| `claude.ai/api/claude_code/policy_limits` | Query policy limits (RC administrator-gate check) | [EVIDENCE.md](./EVIDENCE.md) |
| `claude.ai/api/oauth/authorize` | OAuth authentication (RC registration requires a full-scope token) | [EVIDENCE.md](./EVIDENCE.md) |
| `api.anthropic.com/api/claude_code/metrics` | Telemetry reporting (eligibility check depends on this channel) | [EVIDENCE.md](./EVIDENCE.md) |
| `claude.ai/api/ws/speech_to_text/voice_stream` | Speech-to-text (shared WebSocket infrastructure) | [EVIDENCE.md](./EVIDENCE.md) |
| `POST /v1/sessions` | CCR v2 session creation (with `anthropic-beta: ccr-byoc-2025-07-29` header) | Source: `utils/teleport.tsx` |
| `POST /v1/session_ingress/session/{id}` | Append conversation logs (`Last-Uuid` optimistic concurrency control) | Source: `services/api/sessionIngress.ts` |
| `GET /v1/session_ingress/session/{id}` | Read conversation logs | Source: `services/api/sessionIngress.ts` |
| `GET /v1/sessions/{id}/events` | CCR v2 event stream (cursor pagination, 1,000 items/page, up to 100 pages) | Source: `utils/teleport.tsx` |
| `POST /v1/sessions/{id}/archive` | Archive a remote session (409 = already archived, treated as success) | Source: `utils/teleport.tsx` |
| `GET /v1/sessions/{id}/teleport-events` | Teleport event stream (Spanner v2 / threadstore fallback) | Source: `services/api/sessionIngress.ts` |
| `api.anthropic.com/admin_requests/eligibility` | Eligibility-check endpoint (eligibility decision before RC enablement) | Source: `services/api/adminRequests.ts` |
| `api.anthropic.com/api/claude_code_grove` | Grove endpoint (purpose pending confirmation) | EVIDENCE.md |
| `api.anthropic.com/api/claude_code_penguin_mode` | Fast-mode endpoint | EVIDENCE.md |
| `api.anthropic.com/api/claude_code_shared_session_transcripts` | Shared session transcripts | EVIDENCE.md |
| `api.anthropic.com/api/claude_code/team_memory` | Team memory | EVIDENCE.md |

> **Note**: Remote Control-specific session registration and message relay endpoint URLs are not exposed as plaintext in the source (they may be constructed by concatenation or dynamically obtained from the server). The endpoints above are infrastructure endpoints confirmed in source and EVIDENCE.md.

### 8.6.7 Related Environment Variables

The first 7 items come from [Anthropic official documentation](https://docs.anthropic.com/en/docs/claude-code/remote-control); the rest come from source analysis (`bridge/`, `utils/`, and related directories).

| Variable | Effect | Source |
|------|------|------|
| `ANTHROPIC_API_KEY` | Blocks Remote Control; must be cleared and OAuth login used | Official documentation |
| `CLAUDE_CODE_OAUTH_TOKEN` | Provides a limited-scope token; incompatible with Remote Control | Official documentation |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | May break eligibility checks | Official documentation |
| `DISABLE_TELEMETRY` | Blocks Remote Control registration (suspected implementation coupling, [GitHub #41189](https://github.com/anthropics/claude-code/issues/41189)) | Official documentation + community observation |
| `CLAUDE_CODE_USE_BEDROCK` | Incompatible: Remote Control requires claude.ai authentication | Official documentation |
| `CLAUDE_CODE_USE_VERTEX` | Incompatible: Remote Control requires claude.ai authentication | Official documentation |
| `CLAUDE_CODE_USE_FOUNDRY` | Incompatible: Remote Control requires claude.ai authentication | Official documentation |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session access credential; when present, client type is classified as "remote" | Source: `utils/sessionIngressAuth.ts` |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | WebSocket authentication (file-descriptor passing); when present, client type is classified as "remote" | Source: `utils/sessionIngressAuth.ts` |
| `CLAUDE_CODE_ENTRYPOINT` | When set to `"remote"`, marks a remote entrypoint and changes client-type behavior | Source: `utils/concurrentSessions.ts` |
| `CLAUDE_CODE_REMOTE` | Affects auto-memory behavior when present; passed to teammate spawn environments | Source: `bridge/replBridge.ts` |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Identifies a bridge child process when set to `"bridge"` | Source: `bridge/replBridge.ts` |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | Enables the V2 session ingress protocol when set to `"1"` | Source: `bridge/remoteBridgeCore.ts` |
| `SSE_PORT` | Local SSE port (extracted from source; may be used by Remote Control or MCP SSE transport) | EVIDENCE.md |

### 8.6.8 Implementer Checklist: Design Decision Table

> The following checklist is distilled from source analysis and official documentation. Each item corresponds to a design decision that **must be made** when implementing a Remote Control-like capability; Claude Code's choice is included as a reference.

| # | Design Decision | Claude Code's Choice | Implementation Considerations |
|---|----------|-------------------|----------|
| **1** | **Who holds session state (source of truth), local or cloud?** | **The cloud is the control-plane source of truth** (eligibility checks, policy enforcement, and configuration delivery all happen server-side); local holds data-plane state (conversation history `.jsonl`, PID files) | Cloud control plane allows runtime adjustments (such as poll interval) without client upgrades, but network availability is required to start |
| **2** | **Should eligibility checks reuse the telemetry channel?** | **Suspected coupling**: `DISABLE_TELEMETRY=1` blocks RC registration ([#41189](https://github.com/anthropics/claude-code/issues/41189)); root cause unconfirmed | Decoupling is safer: telemetry switches should not affect feature availability; shared channels can simplify implementation |
| **3** | **Should multi-client authentication use a shared token or scoped tokens?** | **Scoped tokens**: `SESSION_ACCESS_TOKEN` (session access), `WEBSOCKET_AUTH_FILE_DESCRIPTOR` (WebSocket authentication), each expiring independently | Multiple tokens increase management complexity but reduce the blast radius of credential leakage |
| **4** | **Is the poll interval hard-coded or server-adjustable?** | **Delivered by server and validated by Zod schema**, cached with a 5-minute TTL | Server adjustability allows dynamic tuning based on load (for example, switching from 2s to 10min at full capacity), but configuration-service availability must be considered |
| **5** | **Are network/proxy environments first-class supported?** | **Partial support**: `HOST_HTTP_PROXY_PORT` and `HOST_SOCKS_PROXY_PORT` exist in the binary, but connection issues under proxy environments are still reported ([#41324](https://github.com/anthropics/claude-code/issues/41324)) | Enterprise proxies are a common obstacle; outbound HTTPS must correctly handle CONNECT, certificate chains, and authenticated proxies |
| **6** | **Can state recover after a process crash?** | **Partially recoverable**: conversation history `.jsonl` can be restored with `/continue`; bridge state (`replBridge*`) is in-memory only and lost on process exit; PID files remain, but the server treats them as abandoned after 5s | Distinguish "conversation context recovery" (easy) from "bridge session recovery" (requires cloud cooperation) |
| **7** | **How are concurrent sessions isolated?** | `--spawn same-dir` (shared CWD) or `--spawn worktree` (independent Git worktree), with a `--capacity` limit of 32 | File isolation is a basic requirement; worktrees allow parallel edits on different branches but increase disk usage |
| **8** | **Where do diagnostic logs go?** | The `--debug-file <path>` option specifies debug output; `--verbose` controls connection/session log verbosity | Production environments need switchable detailed logs for diagnosing connection loops, credential-refresh failures, and similar issues |

## 8.7 Remote Control vs Claude Code on the Web

They are often confused, but they are fundamentally different:

| | Remote Control | Claude Code on the Web |
|---|---|---|
| **Execution location** | Your local machine | Anthropic cloud infrastructure |
| **Local tools/MCP** | Available (file system, MCP servers, etc.) | Not available |
| **Installation requirement** | Requires a local Claude Code process | No local installation required |
| **Best for** | Continuing in-progress work from another device | Starting new tasks when no local environment exists, parallel tasks |
| **Startup method** | `claude remote-control` / `--rc` / `/rc` | `claude --remote "task description"` |
| **Reverse operation** | — | `claude --teleport` (pull a Web session back into the terminal) |

### 8.7.1 Cross-Device Workflow Overview

Claude Code provides multiple cross-device workflows, each with a different focus:

| Method | Trigger | Run Location | Use Case |
|------|----------|----------|----------|
| **Dispatch** | Send a message from Claude Mobile App | Local machine (Desktop) | Delegate tasks while away from the workstation |
| **Remote Control** | Control a running session from browser/Mobile | Local machine (CLI/VS Code) | Remotely control work in progress |
| **Channels** | Push events from Telegram/Discord | Local machine (CLI) | Respond to external events |
| **Slack** | `@Claude` mention in a team channel | Anthropic cloud | Handle PRs/reviews from team chat |
| **Scheduled Tasks** | Set a schedule | CLI / Desktop / cloud | Periodic automation |
| **`--remote`** | CLI pushes task to Web | Anthropic cloud | Start a Web session |
| **`/teleport`** | Pull a long-running Web task into the terminal | Local machine | Pull a cloud session back locally to continue (CLI equivalent: `claude --teleport`) |

## 8.8 Limitations

| Limitation | Description |
|------|------|
| **One remote session per process** | In interactive mode (non-Server mode), each process has only one remote session. Use `--spawn` when multiple sessions are needed |
| **Terminal must stay open** | Closing the terminal or terminating the process ends the session |
| **Network timeout** | If the machine is online but the network remains unreachable for about 10 minutes, the session times out and the process exits |
| **API Key unsupported** | Must use claude.ai OAuth authentication |
| **Third-party providers unsupported** | Bedrock / Vertex / Foundry users cannot use it |

## 8.9 Known Issues (Community Feedback)

The following issues come from GitHub Issues and are community-observed phenomena; **root causes have not been officially confirmed**:

| Issue | Observed Phenomenon / Suspected Cause | Impact | Source |
|------|------|------|------|
| **Pidfile race** | Non-atomic read-modify-write in `updatePidFile()` in `concurrentSessions.ts` (missing tmp+rename) | JSON file corruption during concurrent sessions; Bun `fallocate` may produce null-byte truncation | [#41195](https://github.com/anthropics/claude-code/issues/41195) |
| **Telemetry coupling** | RC registration fails after setting `DISABLE_TELEMETRY=1` (suspected shared code path between eligibility check and telemetry) | RC fails but the error message misleadingly says "not enabled" | [#41189](https://github.com/anthropics/claude-code/issues/41189) |
| **Zombie process** | Client process does not exit after the server closes the connection (observed `CLOSE_WAIT` TCP state; suspected missing TCP read timeout or `CLOSE_WAIT` detection) | Client still consumes 1+ GB memory after server closure, with no automatic exit | [#41024](https://github.com/anthropics/claude-code/issues/41024) |
| **Connection loop** | Connecting/Disconnected loop, possibly related to credential refresh or network proxies | Remote client cannot connect stably | [#41324](https://github.com/anthropics/claude-code/issues/41324) |
| **Mobile stale connection** | Mobile App reuses expired WebSocket/session token | Idle sessions cannot be restored on mobile, while the CLI side works normally | [#41128](https://github.com/anthropics/claude-code/issues/41128) |
| **VS Code configuration gap** | Extension does not read the `remoteControlAtStartup` setting | Global enablement through `/config` does not take effect in the VS Code extension | [#41036](https://github.com/anthropics/claude-code/issues/41036) |
| **Windows MCP compatibility** | Cloud MCP + RC fails to load on Windows | Windows users cannot use MCP and Remote Control together | [#41044](https://github.com/anthropics/claude-code/issues/41044) |
| **Message loss during active turn** | stdin messages may be lost while the Agent is executing a turn | Instructions sent remotely may not be processed while the agent is busy | [#41230](https://github.com/anthropics/claude-code/issues/41230) |

## 8.10 Troubleshooting

| Error Message | Cause and Fix |
|----------|-----------|
| *"Requires a claude.ai subscription"* | Not authenticated through claude.ai. Run `claude auth login` and select claude.ai. If `ANTHROPIC_API_KEY` is set, clear it first |
| *"Requires a full-scope login token"* | A limited token from `claude setup-token` or `CLAUDE_CODE_OAUTH_TOKEN` is being used. Run `claude auth login` to obtain a full session token |
| *"Unable to determine your organization"* | Cached account information is expired. Run `claude auth login` to refresh it |
| *"Not yet enabled for your account"* | Check whether `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DISABLE_TELEMETRY`, `CLAUDE_CODE_USE_BEDROCK`, etc. are set; clear them. Otherwise `/logout` and then `/login` again |
| *"Disabled by your organization's policy"* | Three possible causes: (1) API Key is used -> switch to claude.ai OAuth; (2) the Team/Enterprise administrator has not enabled the `claude.ai/admin-settings/claude-code` toggle; (3) the administrator toggle is grayed out -> data retention/compliance configuration blocks it; contact Anthropic support |
| *"Remote credentials fetch failed"* | Use `--verbose` for details. Common causes: not logged in, firewall/proxy blocks outbound HTTPS port 443, inactive subscription |

## 8.11 Relationship with the `/session` Command

`/session` (alias `/remote`) is another remote-related command, but it differs from Remote Control:

| Command | Function |
|------|------|
| `/remote-control` `/rc` | Enables bidirectional remote control: real-time control of a terminal session from browser/Mobile |
| `/session` `/remote` | Shows the remote session URL and QR code for viewing/connecting to the session from other devices |
| `/remote-env` | Configures remote environment settings (Claude Code instances on remote servers) |
| `/desktop` `/app` | Hands the current session off to the Claude Desktop app to continue |
| `/mobile` `/ios` `/android` | Shows a QR code for downloading the Claude Mobile app |

## 8.12 Industry Comparison

Among the 19 compared AI programming Agents, Remote Control is **unique to Claude Code**:

| Agent | Remote-Control Capability |
|-------|-------------|
| **Claude Code** | Yes: `/remote-control` + Server mode + `--spawn` multi-session |
| **Copilot CLI** | No (has VS Code integration but no terminal remote control) |
| **Codex CLI** | No |
| **Gemini CLI** | No |
| **Qwen Code** | No ([feature gap analysis](../../comparison/qwen-code-feature-gaps.md); must be built from scratch) |
| **Kimi CLI** | No (has the Wire protocol but has not implemented remote control) |
| **Other Agents** | No |

## 8.13 Implementation Reference for Code Agent Developers

> The following data comes from source analysis (`bridge/`, `remote/`, `utils/`, `entrypoints/`, and related directories; about 35,000 lines of TypeScript). Source filenames and line numbers are directly traceable.
>
> **Applicable scenario**: When developers of other Code Agents implement a similar "remote control" feature, this section can be used as an architecture reference. Claude Code's implementation is production-validated, but it is not the only feasible path.

### 8.13.1 Internal State Machine (Redux Store)

Remote Control maintains 13 bridge state fields in Redux AppState (Source: `bridge/replBridge.ts`):

```javascript
// Source: bridge/replBridge.ts — initial AppState
replBridgeEnabled: false,          // Whether bridging is enabled (legacy field, migrated to remoteControlAtStartup)
replBridgeExplicit: false,         // Whether the user explicitly enabled it (vs automatic enablement)
replBridgeOutboundOnly: false,     // Outbound-only mode: can push messages but does not accept remote-control commands
replBridgeConnected: false,        // Connection state with the relay server
replBridgeSessionActive: false,    // Whether an active remote-client session exists
replBridgeReconnecting: false,     // Whether reconnection is in progress
replBridgeConnectUrl: undefined,   // Remote-client connection URL (for QR codes/links)
replBridgeSessionUrl: undefined,   // Session management URL
replBridgeEnvironmentId: undefined,// Runtime environment identifier (for spawn multi-session routing)
replBridgeSessionId: undefined,    // Current bridge session ID
replBridgeError: undefined,        // Most recent error message
replBridgeInitialName: undefined,  // Initial session name (--name argument value)
showRemoteCallout: false           // UI flag: whether to show the remote-control prompt
```

**State migration logic** (Source: `migrations/migrateReplBridgeEnabledToRemoteControlAtStartup.ts`):

```javascript
// Source: migrations/migrateReplBridgeEnabledToRemoteControlAtStartup.ts
function migrateBridgeConfig(state) {
  if (state.replBridgeEnabled === undefined) return state;
  if (state.remoteControlAtStartup !== undefined) return state;
  let next = {...state, remoteControlAtStartup: Boolean(state.replBridgeEnabled)};
  delete next.replBridgeEnabled;
  return next;
}
```

**Implementation notes**:
- `replBridgeEnabled` is the legacy field name; current versions use `remoteControlAtStartup` (options: `"true"` / `"false"` / `"default"`)
- When `replBridgeOutboundOnly` is `true`, the UI displays "This session is outbound-only. Enable Remote Control locally to allow inbound control."
- `replBridgeReconnecting` is used by the UI to display reconnection state (Connecting/Disconnected loop issues are related to this state)

### 8.13.2 Polling Configuration Parameters (Server-Adjustable / Dynamically Delivered by GrowthBook)

The server delivers polling configuration, and the client validates it with a Zod schema before use. The following are default values and validation rules in source (Source: `bridge/pollConfigDefaults.ts` + `bridge/pollConfig.ts`):

```javascript
// Source: bridge/pollConfigDefaults.ts — default polling configuration
const DEFAULT_POLL_CONFIG = {
  poll_interval_ms_not_at_capacity: 2000,           // Not at capacity: poll every 2 seconds
  poll_interval_ms_at_capacity: 600000,              // At capacity: 10-minute heartbeat (or 0 = disabled)
  non_exclusive_heartbeat_interval_ms: 0,             // Non-exclusive heartbeat: disabled by default
  multisession_poll_interval_ms_not_at_capacity: 2000, // Multi-session not at capacity: 2 seconds
  multisession_poll_interval_ms_partial_capacity: 2000, // Multi-session partially at capacity: 2 seconds
  multisession_poll_interval_ms_at_capacity: 600000,    // Multi-session at capacity: 10 minutes
  reclaim_older_than_ms: 5000,                        // Reclaim threshold: reclaim abandoned sessions after 5 seconds
  session_keepalive_interval_v2_ms: 120000            // WebSocket ping/pong: 2 minutes
};
```

**Zod validation schema** (`hM9`):

| Parameter | Type | Constraint | Default |
|------|------|------|--------|
| `poll_interval_ms_not_at_capacity` | `int` | `>= 100` | `2000` |
| `poll_interval_ms_at_capacity` | `int` | `= 0 or >= 100` | `600000` |
| `non_exclusive_heartbeat_interval_ms` | `int` | `>= 0` | `0` |
| `reclaim_older_than_ms` | `int` | `>= 1` | `5000` |
| `session_keepalive_interval_v2_ms` | `int` | `>= 0` | `120000` |

**Configuration loading mechanism**:

```javascript
// Load through the remote configuration service, TTL 5 minutes
loadConfig("tengu_bridge_poll_interval_config", DEFAULT_POLL_CONFIG, 300000);
```

**Implementation notes**:
- Poll intervals are **server-adjustable**; clients should not hard-code them. Deliver them dynamically through a remote configuration service
- At capacity (`at_capacity`), the poll interval switches from 2s to 10min, effectively entering a "heartbeat keepalive" mode
- `reclaim_older_than_ms: 5000` means the server can mark a session abandoned after 5 seconds without a response; implementers should consider network jitter scenarios
- `session_keepalive_interval_v2_ms` is the WebSocket-layer ping/pong and complements HTTP-layer polling

### 8.13.3 Message Protocol (Wire Format)

Remote Control uses JSON-lines for message transport. The following message types come from source (`entrypoints/sdk/controlSchemas.ts`, about 510 lines of Zod v4 schema definitions):

**Message envelope**:

| Schema | `type` Field | Purpose |
|--------|------------|------|
| `SDKControlRequestSchema` | `control_request` | Control request from the remote side, containing `request_id` + `request` with an inner `subtype` |
| `SDKControlResponseSchema` | `control_response` | Response to a control request, containing `subtype` (`success`/`error`) + corresponding data |
| `SDKControlCancelRequestSchema` | `control_cancel_request` | Cancels a pending control request |
| `SDKKeepAliveMessageSchema` | `keep_alive` | Heartbeat keepalive |
| `SDKUpdateEnvironmentVariablesMessageSchema` | `update_environment_variables` | Parent process injects environment-variable updates into child process |

**Control Request subtypes** (21 types, complete list from source):

| Subtype | Purpose | Source |
|--------|------|------|
| `initialize` | Session initialization (hooks, MCP servers, agents, system prompt) | `SDKControlInitializeRequestSchema` |
| `interrupt` | Interrupt the current turn | `SDKControlInterruptRequestSchema` |
| `can_use_tool` | Tool-permission approval request (with `tool_name`, `input`, `tool_use_id`) | `SDKControlPermissionRequestSchema` |
| `set_permission_mode` | Set permission mode | `SDKControlSetPermissionModeRequestSchema` |
| `set_model` | Switch model | `SDKControlSetModelRequestSchema` |
| `set_max_thinking_tokens` | Set thinking-token limit | `SDKControlSetMaxThinkingTokensRequestSchema` |
| `mcp_status` | Query MCP server status | `SDKControlMcpStatusRequestSchema` |
| `get_context_usage` | Get context-window analysis (including categories, totals, percentages, and grid visualization data) | `SDKControlGetContextUsageRequestSchema` |
| `rewind_files` | Revert file changes to a specified user message | `SDKControlRewindFilesRequestSchema` |
| `cancel_async_message` | Discard a pending asynchronous user message | `SDKControlCancelAsyncMessageRequestSchema` |
| `seed_read_state` | Prepopulate the readFileState cache | `SDKControlSeedReadStateRequestSchema` |
| `hook_callback` | Deliver a hook callback | `SDKHookCallbackRequestSchema` |
| `mcp_message` | Send JSON-RPC to an MCP server | `SDKControlMcpMessageRequestSchema` |
| `mcp_set_servers` | Replace dynamic MCP servers | `SDKControlMcpSetServersRequestSchema` |
| `reload_plugins` | Reload plugins from disk | `SDKControlReloadPluginsRequestSchema` |
| `mcp_reconnect` | Reconnect failed MCP servers | `SDKControlMcpReconnectRequestSchema` |
| `mcp_toggle` | Enable/disable an MCP server | `SDKControlMcpToggleRequestSchema` |
| `stop_task` | Stop a running task | `SDKControlStopTaskRequestSchema` |
| `apply_flag_settings` | Merge flag settings | `SDKControlApplyFlagSettingsRequestSchema` |
| `get_settings` | Get effective settings (including per-source layering: user/project/local/flag/policy) | `SDKControlGetSettingsRequestSchema` |
| `elicitation` | MCP elicitation (user input request) | `SDKControlElicitationRequestSchema` |

> **iOS compatibility layer**: Older iOS App versions send camelCase `requestId` (because Swift CodingKeys are missing). Source uses `normalizeControlMessageKeys()` to convert `requestId` -> `request_id` for compatibility. snake_case has higher priority than camelCase.

### 8.13.4 Token Refresh System

The source reveals two independent generations of token refresh strategies:

**v1 (OAuth refresh)** — `bridge/replBridge.ts`:
- `clearOAuthTokenCache()` + `checkAndRefreshOAuthTokenIfNeeded()` refresh the OAuth token
- New token is injected into the WebSocket connection through the `refreshHeaders` callback
- Child processes update `CLAUDE_CODE_SESSION_ACCESS_TOKEN` through an `update_environment_variables` stdin message

**v2 (JWT refresh)** — `bridge/remoteBridgeCore.ts`:
- `createTokenRefreshScheduler` triggers 5 minutes before JWT expiration
- Calls the `/bridge` endpoint to refresh credentials and bumps epoch on each call (double-refresh prevention: the `authRecoveryInFlight` flag serializes concurrent requests)
- SSE 401 triggers emergency refresh: `onAuth401` -> fetch OAuth again -> refetch credentials -> rebuild transport
- `initialFlushDone` is reset to false to ensure history messages are resent

| Parameter | v1 Value | v2 Value | Description |
|------|-------|-------|------|
| Refresh lead time | — | 5 minutes before expiration | Driven by JWT expiry |
| Maximum retries | 3 | — | Give up after refresh failure |
| Retry delay | 60 seconds | Exponential backoff | — |
| Sleep recovery | Budget reset | epoch bump | After laptop wakes from sleep |

**Refresh strategy**: Scheduled based on JWT expiry; refresh is triggered 5 minutes before token expiration and gives up after at most 3 failures.

### 8.13.5 Bridge Initialization Interface (initReplBridge)

The extracted `initReplBridge` callback interface from source (`bridge/initReplBridge.ts`, 569 lines). This is the core Remote Control bridge layer and injects all external dependencies through `BridgeCoreParams`:

**`BridgeCoreParams` (injected parameters)**:
- `createSession` — create a new session
- `archiveSession` — archive a session
- `toSDKMessages` — convert internal messages to SDK messages
- `onAuth401` — 401 authentication-failure callback
- `getPollIntervalConfig` — get GrowthBook-delivered polling parameters
- `onSetPermissionMode` — permission-mode change callback
- `onEnvironmentLost` — environment-lost callback

**`ReplBridgeHandle` (returned unified handle)**:
- Read-only properties: `bridgeSessionId`, `environmentId`, `sessionIngressUrl`
- `writeMessages(Message[])` — write raw messages
- `writeSdkMessages(SDKMessage[])` — write SDK-format messages
- `sendControlRequest` / `sendControlResponse` / `sendControlCancelRequest` — permission bridge
- `sendResult` — send session-end signal
- `teardown()` — clean up resources

**Key design**: In source, `BridgeCoreParams` does not directly import `bootstrap/state` or `sessionStorage`; all external dependencies are passed by injection, achieving zero coupling between core bridge logic and the UI/storage layer.

### 8.13.6 Bridge Child-Process Environment Variables

Child processes created by `--spawn` in Server mode inherit the following environment variables (Source: `bridge/replBridge.ts`):

```javascript
// Environment-variable settings for spawned child processes
{
  CLAUDE_CODE_OAUTH_TOKEN: undefined,              // Explicitly clear it to prevent child processes from re-registering with the OAuth token
  CLAUDE_CODE_ENVIRONMENT_KIND: "bridge",          // Identify as a bridge child process
  CLAUDE_CODE_SESSION_ACCESS_TOKEN: accessToken,   // Pass the session access credential
  CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2: "1",    // Enable the V2 session ingress protocol
  CLAUDE_CODE_USE_CCR_V2: "1",                     // Use CCR V2 (if useCcrV2=true)
  CLAUDE_CODE_WORKER_EPOCH: String(workerEpoch),    // Worker epoch (if useCcrV2=true)
  CLAUDE_CODE_FORCE_SANDBOX: "1"                    // Force sandboxing (if --sandbox)
}
```

**Implementation notes**:
- `CLAUDE_CODE_OAUTH_TOKEN` is explicitly set to `undefined`; child processes should not use the parent process's OAuth credentials to register again, and should instead take over the session through `SESSION_ACCESS_TOKEN`
- `CLAUDE_CODE_ENVIRONMENT_KIND: "bridge"` lets the child process know it is running in bridge mode and adjust behavior (for example, not starting its own WebSocket/SSE connection)
- `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2: "1"` enables the newer session ingress protocol

### 8.13.7 Concurrent Session Management (PID File)

Extracted session registration file format and update logic from source (`utils/concurrentSessions.ts`, 205 lines):

**PID file path**: `$CONFIG_DIR/sessions/{process.pid}.json`

**Written during registration**:

```javascript
// Session registration: write PID file
await fs.writeFile(pidFilePath, JSON.stringify({
  pid: process.pid,
  sessionId: getSessionId(),
  cwd: getCurrentWorkingDir(),
  startedAt: Date.now(),
  kind: sessionKind,                              // "interactive" | "server" | ...
  entrypoint: process.env.CLAUDE_CODE_ENTRYPOINT,  // "cli" | "remote" | ...
  name: sessionName                                // optional
}));
```

**Update logic** (`gv7` function):

```javascript
// Non-atomic update: read -> merge -> write (has a race condition)
async function updatePidFile(updates) {
  const path = join(configDir(), `${process.pid}.json`);
  try {
    const existing = JSON.parse(await fs.readFile(path, "utf8"));
    await fs.writeFile(path, JSON.stringify({...existing, ...updates}));
  } catch (err) {
    log(`[concurrentSessions] updatePidFile failed: ${formatError(err)}`);
  }
}
```

**Known issue**: This read-modify-write operation is **non-atomic** (missing tmp+rename) and can corrupt JSON files under concurrency ([GitHub #41195](https://github.com/anthropics/claude-code/issues/41195)). Implementers should use an atomic write pattern such as `writeFileSync(tmp, data)` + `renameSync(tmp, path)`.

### 8.13.8 Bridge Session Registration Schema

Extracted Zod schema from source (Source: `bridge/createSession.ts`) for registering bridge sessions:

```javascript
// Session registration request-body schema
const bridgeSessionSchema = z.object({
  session_id: z.string(),         // Unique session identifier
  ws_url: z.string(),             // WebSocket connection URL
  work_dir: z.string().optional() // Working directory (optional)
});
```

### 8.13.9 Complete CLI Options (`remote-control` Subcommand)

Complete `claude remote-control` option list extracted from source (Source: `bridge/bridgeMain.ts`):

```
claude remote-control [options]
  --spawn <mode>                      Spawn mode: same-dir | worktree | session
  --capacity <N>                      Maximum number of concurrent sessions
  --create-session-in-dir <path>      Create a session in the specified directory
  --session-id <id>                   Resume the specified session ID
  --continue                          Continue the previous session
  --permission-mode <mode>            Permission mode
  --name <name>                       Session name
  --verbose                           Verbose logs
  --sandbox                           Enable sandbox
  --debug-file <path>                 Debug output file
  --session-timeout-ms <ms>           Session timeout (milliseconds)
```

**New findings** (compared with official documentation):

| Option | Official Documentation | Source Finding |
|------|----------|-----------|
| `--spawn session` | Not mentioned | Third spawn mode |
| `--create-session-in-dir` | Not mentioned | Creates a session in the specified directory |
| `--session-id` | Not mentioned | Resumes a specific session ID |
| `--session-timeout-ms` | Not mentioned | Precisely controls session timeout |
| `--debug-file` | Not mentioned | Writes debug output to a file |

### 8.13.10 Telemetry Events

Remote Control-related telemetry event prefixes are `tengu_bridge_*`; the following event names were extracted from source (Source: `bridge/bridgeMain.ts`):

| Event Name | Purpose |
|--------|------|
| `tengu_bridge_token_refreshed` | Tracks token refresh success/failure |
| `tengu_bridge_multi_session_denied` | Multi-session access denied (capacity full) |
| `tengu_bridge_poll_interval_config` | Tracks polling configuration loading |
| `tengu_concurrent_sessions` | Tracks concurrent session state |

### 8.13.11 Client Type Detection

Client type detection logic extracted from source (Source: `utils/sessionIngressAuth.ts` + `bridge/replBridge.ts`):

```javascript
// Client type detection
function detectClientType() {
  const hasSessionToken = process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN
                       || process.env.CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR;
  if (process.env.CLAUDE_CODE_ENTRYPOINT === "remote" || hasSessionToken) {
    return "remote";  // Remote Control bridge client
  }
  // ... Other type checks
}
```

**Key environment variables**:

| Variable | Description | Source |
|------|------|------|
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Marks the client type as "remote" when present | Source: `utils/sessionIngressAuth.ts` |
| `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR` | Marks the client type as "remote" when present (file-descriptor passing for WebSocket authentication) | Source: `utils/sessionIngressAuth.ts` |
| `CLAUDE_CODE_ENTRYPOINT` | Marks a remote entrypoint when set to `"remote"` | Source: `utils/concurrentSessions.ts` |
| `CLAUDE_CODE_REMOTE` | Affects auto-memory behavior when present; passed to teammate spawn environments | Source: `bridge/replBridge.ts` |

### 8.13.12 Implementation Recommendations: Architecture Pattern Summary

Based on source analysis (`bridge/` directory: 12,613 lines + `remote/` directory: 1,127 lines), the recommended architecture pattern for implementing a Remote Control-like capability is:

```
┌──────────────────────────────────────────────────────────────────┐
│                  Recommended Architecture Pattern                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐    1. Register      ┌──────────────┐               │
│  │  Local   │ ──────────────────→ │              │               │
│  │  Agent   │ ←────────────────── │   Relay      │               │
│  │  Process │    2. Token + URL   │   Server     │               │
│  │          │                     │              │               │
│  │          │    3. HTTPS Poll    │  - Message   │               │
│  │          │ ──────────────────→ │    routing   │               │
│  │          │ ←────────────────── │  - Credential│               │
│  │          │    4. Stream Msgs   │    management│               │
│  │          │                     │  - Session   │               │
│  │          │                     │    tracking  │               │
│  │          │                     │  - Config    │               │
│  │          │                     │    delivery  │               │
│  └─────────┘                     └──────┬───────┘               │
│                                         │                        │
│                                    5. WebSocket/HTTPS            │
│                                         │                        │
│                                  ┌──────▼───────┐               │
│                                  │   Remote      │               │
│                                  │   Client      │               │
│                                  │  (Web/Mobile) │               │
│                                  └──────────────┘               │
│                                                                  │
│  Key design decisions:                                           │
│  1. Local outbound-only HTTPS -> enterprise firewall traversal    │
│  2. Server-delivered poll config -> runtime adjustability         │
│  3. JWT + short-lived credentials -> scheduled refresh (5min before expiry) │
│  4. JSON-lines message protocol -> simple and extensible          │
│  5. PID file concurrent registration -> requires atomic write (tmp+rename) │
│  6. Child process explicitly clears OAuth token -> prevents duplicate registration │
│  7. keep_alive no-op -> reduces useless processing overhead       │
│  8. Client type detection -> distinguishes local/remote/bridge behavior │
└──────────────────────────────────────────────────────────────────┘
```

### 8.13.13 Comparison Reference with Gemini CLI

Qwen Code is forked from Gemini CLI ([Qwen Code EVIDENCE](../qwen-code/EVIDENCE.md)). Gemini CLI uses Google Cloud relay to implement the `--remote` feature. The implementation paths differ as follows:

| Dimension | Claude Code | Gemini CLI / Qwen Code |
|------|-------------|----------------------|
| **Relay architecture** | Anthropic API relay | Google Cloud relay |
| **Local transport** | WebSocket (v1) / SSE (v2), server-adjustable | SSE (Server-Sent Events) |
| **Authentication** | claude.ai OAuth full-scope | Google OAuth |
| **Message format** | JSON-lines (8+ message types) | SSE stream |
| **State management** | Redux 13-field state machine | — |
| **Token refresh** | JWT expiry - refresh 5min early | — |
| **Multi-session** | `--spawn` + `--capacity` + PID file | — |

> **Note**: Gemini CLI's `--remote` implementation details are not public source. Comparison data comes from CLI help output and behavioral observation; items marked `⚠️` are inferred.

## 8.14 Evaluation and Pros/Cons Analysis

### 8.14.1 Core Strengths

**1. The only Agent that implements "remote control of terminal sessions"**

Claude Code's Remote Control stands apart among all 19 Agents. It is not a simple Web UI or API exposure; it bidirectionally bridges a **running terminal interactive session** to a browser/phone. This means the remote side can fully use the local file system, MCP servers, and project configuration, enabling true "leave the workstation without interrupting work" behavior.

**2. Careful security architecture design**

- Outbound-only HTTPS and zero inbound ports: enterprise-firewall friendly
- Mandatory claude.ai OAuth full-scope token: excludes API Key / third-party providers
- Team/Enterprise administrator gate: organization-level control
- Multiple short-lived isolated credentials: prevents lateral movement after credential leakage

**3. Flexible multi-mode adaptation**

Three startup methods (Server mode / interactive / in-session) cover different scenarios: Server mode for long unattended tasks, interactive mode for daily development, and in-session enablement for temporary needs. `--spawn worktree` supports file isolation across multiple sessions, and `--capacity` prevents resource exhaustion.

**4. Complete cross-device ecosystem**

Remote Control is not an isolated feature; it is part of Claude Code's cross-device matrix. Together with `--remote` (push to Web), `/teleport` (pull back to terminal), Dispatch (mobile delegation), Channels (Telegram/Discord push), and `/schedule` (scheduled tasks), it forms complete workflow coverage from "real-time remote control" to "asynchronous event-driven automation".

### 8.14.2 Core Weaknesses

**1. Strong binding to the claude.ai ecosystem**

- Does not support API Key, Bedrock, Vertex, or Foundry, excluding enterprise private-deployment scenarios
- `DISABLE_TELEMETRY` unexpectedly blocks registration, forcing privacy-sensitive users to choose between telemetry and remote control
- Disabled by default for Team/Enterprise, and some compliance configurations cannot be overridden

**2. Stability issues**

As of v2.1.81, the community has reported 8 known issues (see above), several of which affect real-world use:
- Zombie process: client does not exit after server closure, and memory is not released
- Connection loop: repeated Connecting/Disconnected state, preventing stable operation
- Mobile stale connection: cannot recover after WebSocket/session token expiration
- VS Code extension ignores the `remoteControlAtStartup` configuration

**3. Strong dependency on the local process**

- Terminal must remain open; closing it disconnects the session
- Times out and exits after about 10 minutes of network disconnection
- Unlike `/schedule` (CCR cloud execution), it cannot "keep running after shutdown"

**4. Closed source with limited evidence**

Communication protocol details (two-generation WebSocket/SSE transport, message format, endpoint URLs) are not publicly documented, but have been fully revealed through source analysis. Security auditing can already be independently verified from source.

### 8.14.3 Design Trade-Off Summary

| Design Decision | Benefit | Cost |
|----------|------|------|
| Local execution + remote control | Full local environment is available | Terminal must be online |
| WebSocket/SSE relay | Zero inbound ports, enterprise-friendly | Two-generation transport increases maintenance complexity |
| Strong binding to claude.ai OAuth | Unified authentication and administrator control | Excludes API Key / third-party users |
| Multiple short-lived credentials | Small blast radius for credential leakage | More registration failure points |
| Telemetry coupled with eligibility checks | — | Can unexpectedly block privacy-sensitive users |

## 8.15 Competitor Comparison: Remote Access Capability Overview

Claude Code Remote Control is unique in the dimension of "cross-device remote control of terminal sessions", but "remote access" itself appears in different forms in other Agents:

### 8.15.1 Feature Comparison Matrix

> **Dimension definitions**: This table compares capabilities horizontally by whether each capability exists. The capabilities are defined as follows:
> - **Terminal session remote control**: Real-time control of a running CLI session from another device
> - **Web/browser UI**: A graphical interface accessible through a browser
> - **Multiple clients connected simultaneously**: The same session can be connected by multiple client types (TUI/Web/Desktop/Mobile) at the same time
> - **Native mobile App**: Native iOS/Android application (not mobile-browser access to a Web UI)
> - **Zero inbound ports**: No local listening port needs to be opened

| Capability | Claude Code | Kimi CLI | OpenCode | Goose | Codex CLI | Copilot CLI | Aider |
|------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **Terminal session remote control** | Yes | No | No | No | No | No | No |
| **Web/browser UI** | No | Yes, FastAPI+React | Yes, SolidJS (experimental) | No (Desktop App only; see [note](../goose.md)) | No | No | No |
| **Multiple clients connected simultaneously** | Yes, TUI+browser+Mobile | Yes, Wire four clients | Yes, TUI+Web+Desktop | Yes, CLI+Desktop | No | No | No |
| **Native mobile App** | Yes, iOS/Android | No (mobile browser can access Web UI) | No | No | No | No | No |
| **Zero inbound ports** | Yes (outbound-only relay) | No (opens a port) | No (opens a port) | No (opens a port) | No | N/A | N/A |
| **Remote IDE integration** | Yes, VS Code | Yes, ACP | No | No | Yes, app-server | Yes, native | No |
| **Protocol** | WebSocket/SSE + relay | Wire v1.6 (WS) | Hono HTTP+WS+SSE | REST (Axum) | JSON-RPC (WS) | CLI only | CLI only |

### 8.15.2 Key Competitor Differences

**Kimi CLI** (most complete Web UI):
- `kimi web` starts a FastAPI + React Web UI (default `localhost:5494`) with multi-session management, real-time diff preview, and approval dialogs
  (Source: [Kimi CLI architecture documentation](../kimi-cli/03-architecture.md))
- The Wire v1.6 protocol unifies four clients: TUI / Web / IDE / custom UI
  (Source: [Kimi CLI architecture documentation](../kimi-cli/03-architecture.md))
- Supports three network modes: `--network`, `--lan-only`, and `--public`, with token authentication
- **Weakness**: Requires opening a local port (unlike Claude Code's outbound-only model) and has no native mobile App

**OpenCode** (multi-client architecture):
- TUI + Web Console (SolidJS) + Desktop (Tauri v2 / Electron) share a Hono HTTP backend
  (Source: [OpenCode architecture documentation](../opencode/03-architecture.md))
- MDNS local-network device discovery
- **Weakness**: Remote workspace remains experimental, and stability still needs validation

**Goose** (REST API-driven):
- `goose-server` (Axum HTTP) provides a REST API, and the Electron Desktop App acts as a GUI client
  (Source: [Goose architecture documentation](../goose/03-architecture.md), [EVIDENCE](../goose/EVIDENCE.md))
- Repository documentation claims support for CLI/Web/Desktop clients ([Goose overview](../goose.md)), but the actual source contains only two concrete client implementations: CLI (`goose-cli`) and Desktop (`ui/desktop/`). The "Web" label means the `goose-server` HTTP API can be connected to by a Web client, but there is no standalone browser frontend
- **Weakness**: No standalone browser Web UI (only HTTP API) and no mobile client

**Codex CLI** (IDE-integration oriented):
- `codex app-server` provides JSON-RPC 2.0 over stdio/WebSocket, and `--remote` connects to a remote instance
  (Source: [Codex CLI command documentation](../codex-cli/02-commands.md), [EVIDENCE](../codex-cli/EVIDENCE.md))
- **Weakness**: Designed for IDE plugins, not general-purpose remote access

### 8.15.3 Why Has No Other Agent Copied Remote Control? (Author Analysis, Not a Source-Verified Conclusion)

> The following is analytical speculation based on public information and has not been verified by source or official statements.

| Possible Reason | Description |
|----------|------|
| **Requires cloud infrastructure** (inferred) | Anthropic's relay servers and claude.ai/code platform are prerequisites for Remote Control. Open-source Agents lack cloud relay infrastructure at equivalent scale |
| **Strong binding to authentication system** (inferred) | Mandatory OAuth + short-lived credentials + administrator gate depend on a centralized identity system, which self-hosted Agents are hard-pressed to replicate |
| **Different product positioning** (inferred) | Claude Code is positioned as an "enterprise AI programming platform" (including Web/Desktop/Mobile clients), while most Agents are positioned as "local developer tools" |
| **Lower-cost alternatives** (inferred) | Kimi CLI/OpenCode and similar tools provide basic remote access through local Web UI + opened ports, which is sufficient for many scenarios |

> **Disclaimer**: The data above is based on Q1 2026 source analysis and official documentation and may become outdated.
