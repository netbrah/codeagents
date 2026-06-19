# 14. MCP Integration — Developer Reference

> MCP (Model Context Protocol) is the general protocol for connecting Code Agents to external tools and data sources. Claude Code's MCP implementation is the most complete among the three major Agents: 23 files, ~12,000 lines, 6 transport types, OAuth + XAA authentication, and Channel message push.
>
> **Qwen Code comparison**: Qwen Code already has a basic MCP implementation (~4,300 lines), including OAuth token storage abstraction and Google authentication. The main gaps are Channel message push, resource subscriptions, and reconnection strategy.
>
> **v2.1.82 → v2.1.132 delta** (see [§23 §8](./23-recent-updates.md) for details):
> - **Per-tool MCP result-size override** (Week 14): can set a single tool output limit up to **500K**, overriding the default limit
> - **Plugin executables on Bash `PATH`** (Week 14): plugins can inject binaries into the shell environment
> - **MCP server OAuth improvements** (v2.1.126): manually paste the OAuth code when the browser callback is unreachable
> - **MCP retry logic improvements** (v2.1.132): clearer status for connection failures
> - **stdio MCP memory leak fix** (v2.1.132): fixed unbounded RSS growth (10GB+) in long-running scenarios — a critical fix for long-running daemons

## 1. Why MCP Integration Is Critical Agent Infrastructure

### Problem Definition

A Code Agent's built-in tools (Read/Write/Edit/Bash) cover core programming operations, but developer workflows go far beyond that:

| Need | Can built-in tools do it? | Need MCP? |
|------|-------------|-----------|
| Read/write local files | ✓ | — |
| Execute shell commands | ✓ | — |
| Query Jira/Linear tasks | — | ✓ |
| Send Slack messages to notify the team | — | ✓ |
| Operate Docker containers | — | ✓ |
| Query databases | — | ✓ |
| Access GitHub API | — | ✓ |

MCP extends an Agent's capabilities from the "local file system" to "any external system" without requiring a built-in tool for every system.

### Architectural Core: Tool Namespace

MCP tools are registered in the `mcp__<serverName>__<toolName>` format and share the same tool registry as built-in tools:

```
Tools available to the Agent:
├─ Read           (built-in)
├─ Write          (built-in)
├─ Bash           (built-in)
├─ mcp__github__get_issue       (MCP: GitHub server)
├─ mcp__github__create_pr       (MCP: GitHub server)
├─ mcp__slack__send_message     (MCP: Slack server)
└─ mcp__postgres__run_query     (MCP: PostgreSQL server)
```

To the model, MCP tools and built-in tools are the same: callable functions.

## 2. MCP Implementation Comparison Across Three Agents

| Dimension | Claude Code | Gemini CLI | Qwen Code |
|------|-------------|-----------|-----------|
| **Files / LOC** | 23 files / ~12,000 lines | 4 files / ~8,400 lines | 21 files / ~4,300 lines |
| **Transport types** | 6 types (stdio/sse/http/ws/sdk/sse-ide) | 3 types (stdio/sse/streamableHttp) | 3 types (stdio/sse/streamableHttp) |
| **OAuth** | ✓ + XAA (cross-application authentication) | ✓ + Google authentication | ✓ + Google authentication + encrypted file storage |
| **Resources** | ✓ supports subscriptions | ✓ supports listChanged | — |
| **Prompt (MCP Prompts)** | ✓ supports listChanged | ✓ supports listChanged | — |
| **Progress tracking** | ✓ MCPProgress | ✓ McpProgressReporter | ✓ onprogress callback |
| **Channel messages** | ✓ complete implementation (7-layer gating) | — | — |
| **Reconnection** | ✓ cache clearing + session recovery | — | — |
| **Configuration format** | `.mcp.json` (7 scopes) | mcp-config.json | mcp configuration |
| **Special transports** | SdkControlTransport (in-process) | XcodeMcpBridgeFixTransport | SdkControlClientTransport |

### Capabilities Unique to Claude Code

**1. Channel message push**: MCP servers can push messages into Agent sessions through the `notifications/claude/channel` method (for example, new Slack messages or GitHub PR comments). Seven layers of gating ensure safety:

```
Channel enablement conditions (all must be satisfied):
1. Server declares capabilities.experimental['claude/channel']
2. Runtime isChannelsEnabled()
3. Claude.ai OAuth authentication (API key users are not supported)
4. Organization policy channelsEnabled: true
5. --channels startup argument includes this server
6. Marketplace source verification
7. Approval allowlist
```

**2. XAA (cross-application authentication)**: enterprise authentication based on the IETF draft Identity Assertion Authorization Grant, supporting the IdP → JWT → Token Exchange flow.

**3. Six transports**: in addition to the standard three, it includes WebSocket (ws), in-process SDK (sdk), and IDE-specific SSE (sse-ide).

### Qwen Code's Unique Advantage

**Token storage abstraction**: Qwen Code has the most complete token persistence layer: FileTokenStorage (AES-256-GCM encryption), KeychainTokenStorage, and HybridTokenStorage. Claude Code and Gemini CLI have comparatively simpler token storage.

## 3. MCP Configuration Format

### Claude Code's `.mcp.json`

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "ghp_..." }
    },
    "slack": {
      "type": "sse",
      "url": "https://mcp.slack.com/sse",
      "oauth": {
        "clientId": "...",
        "callbackPort": 8080,
        "authServerMetadataUrl": "https://slack.com/.well-known/openid-configuration"
      }
    },
    "database": {
      "type": "http",
      "url": "https://db-mcp.internal/api",
      "headers": { "Authorization": "******" }
    }
  }
}
```

**Seven configuration scopes**: local → user → project → dynamic → enterprise → claudeai → managed

### Configuration Differences from Qwen Code/Gemini CLI

| Feature | Claude Code | Gemini CLI | Qwen Code |
|------|-------------|-----------|-----------|
| Configuration file | `.mcp.json` | `mcp-config.json` or settings | in settings.json |
| Scopes | 7 types | 3 types (user/project/extension) | 2 types (user/project) |
| Environment variables | `$VAR` syntax | env field | env field |
| Inline OAuth | ✓ | ✓ | ✓ |
| Enterprise control | ✓ managed-mcp.json | ✓ admin policy | — |

## 4. Tool Registration and Execution

### Registration Flow

```
MCP server connection
  │
  ├─ transport.connect()        ← Establish communication channel
  │
  ├─ client.initialize()        ← Negotiate capabilities
  │
  ├─ tools/list                 ← Get tool list
  │     └─ Each tool → registered as mcp__server__tool
  │
  ├─ prompts/list              ← Get prompt list (optional)
  │
  ├─ resources/list            ← Get resource list (optional)
  │
  └─ Listen for listChanged notifications     ← Dynamic tool/resource updates
```

### Execution Flow

```
Model calls mcp__github__get_issue({"number": 42})
  │
  ├─ Validate input (JSON Schema / AJV)
  │
  ├─ client.callTool("get_issue", {"number": 42})
  │     ├─ OAuth 401 → Trigger authentication flow → retry
  │     ├─ Session expired (404 -32001) → reconnect → retry
  │     └─ Success → return result
  │
  ├─ Convert result (text/image/PDF)
  │
  ├─ Truncation check (>8KB → truncate + hint)
  │
  └─ Return to model
```

## 5. Reconnection Strategy

Claude Code's reconnection strategy:

| Error Type | Detection Method | Recovery Action |
|---------|---------|---------|
| Session expired | HTTP 404 + JSON-RPC -32001 | Clear memoization cache + automatically reconnect on next operation |
| SSE connection interrupted | SDK default: 2 retries | Exponential-backoff reconnect |
| Consecutive failures | Failure counter | Close transport + trigger explicit reconnect |
| OAuth token expired | 401 response | Automatic refresh → retry original request |

**Difference from Gemini CLI**: Gemini CLI has `MCP Auto-Reconnect` (automatically reconnects after 3 consecutive errors), while Claude Code's reconnection is more granular (distinguishes session expiration vs connection interruption vs authentication expiration).

## 6. Qwen Code MCP Improvement Suggestions

### P1: Resources Support

MCP resources allow servers to expose structured data (such as database schemas and API documentation), which Agents can read on demand. Qwen Code currently supports only tool calls, not resource discovery and reading.

### P1: Reconnection

Currently, MCP servers must be restarted manually after disconnection. Consider Claude Code's memoization cache clearing + automatic reconnection pattern.

### P2: MCP Prompt Support

MCP servers can expose prompt templates (for example, "summarize this PR") and register them as slash commands. Qwen Code currently does not support MCP prompts.

### P2: Channel Message Push

Allow MCP servers to push messages into Agent sessions. This is the foundation for Slack/Discord integrations. However, implementation complexity is high (7-layer gating), so it is recommended as a medium- to long-term goal.

### P3: WebSocket Transport

Currently only stdio/sse/http are supported. WebSocket is more efficient for scenarios requiring bidirectional real-time communication, such as real-time collaboration tools.
