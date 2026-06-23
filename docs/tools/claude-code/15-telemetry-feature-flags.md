# 15. Telemetry and Feature Flags — Developer Reference

> Claude Code's telemetry system has 891+ events, a dual-sink architecture (Datadog + 1P), and GrowthBook remote feature flags. Feature Flags control not only gradual feature rollout, but also telemetry sampling rates and sink circuit breakers, forming an **observability-driven feature release system**.
>
> **Qwen Code comparison**: Qwen Code has basic OpenTelemetry telemetry (~40 events) and no remote Feature Flags. Claude Code's GrowthBook integration (remote rollout, A/B testing, production circuit breakers) and privacy-protection patterns (_PROTO_ key tiers, user bucketing) are the main references.

## 1. Why Code Agents Need Telemetry and Feature Flags

### Problem Definition

Code Agents are much more complex than ordinary CLI tools: their behavior depends on combinations of model behavior, user input, and project state, making exhaustive pre-release testing difficult:

| Scenario | Without Telemetry | With Telemetry |
|------|-------|--------|
| Crash rate rises after a new feature launches | Known only after user complaints | Real-time alerts, discovered within minutes |
| Prompt Cache hit rate drops | API costs rise mysteriously | See a spike in `tengu_compact_cache_sharing_fallback` events |
| A specific MCP server frequently times out | Users think the Agent is stuck | `tengu_mcp_server_connection_failed` pinpoints the server |
| New Shell security check has false positives | Users are blocked from normal operations | `tengu_tool_use_rejected_in_prompt` helps analyze false-positive patterns |

Feature Flags solve **release risk**: new features are no longer "roll out to everyone or roll back for everyone"; they can be rolled out by percentage, grouped by user type, and circuit-broken within seconds when problems occur.

### Telemetry Comparison with Competitors

| Agent | Event Count | Infrastructure | Remote Feature Flag | Privacy Protection |
|-------|--------|---------|-------------------|---------|
| **Claude Code** | 891+ | Datadog + 1P BigQuery | ✓ GrowthBook (6h/20m refresh) | _PROTO_ tiers, user bucketing, MCP tool-name redaction |
| **Gemini CLI** | ~50 | Google Cloud (Trace/Monitoring/Logging) | — | User opt-out |
| **Qwen Code** | ~40 | OpenTelemetry + Alibaba Cloud/local | — | `logPrompts` switch |
| **Copilot CLI** | Not public | GitHub internal | ✓ through GitHub platform | Enterprise-grade privacy |

## 2. GrowthBook Feature Flag System

### 2.1 Architecture

```
┌─────────────────────────────────────────────────┐
│  GrowthBook server (Anthropic-hosted)           │
│  Manages Feature Flag values + A/B experiment   │
│  configuration                                  │
└────────────────────┬────────────────────────────┘
                     │ HTTPS polling
                     │ External users: every 6 hours
                     │ Internal users: every 20 minutes
                     ▼
┌─────────────────────────────────────────────────┐
│  Claude Code local GrowthBook Client            │
│  ├─ In-memory cache (runtime)                   │
│  ├─ Disk cache (~/.claude/config.json)          │
│  └─ Environment variable overrides (dev/debug)  │
└────────────────────┬────────────────────────────┘
                     │ feature('FLAG_NAME')
                     ▼
┌─────────────────────────────────────────────────┐
│  Feature code                                   │
│  if (getGrowthBookFeatureValue('tengu_kairos')) │
│    → Enable Kairos autonomous mode              │
└─────────────────────────────────────────────────┘
```

### 2.2 Two Types of Feature Flags

| Type | Evaluation Time | Mechanism | Purpose |
|------|---------|------|------|
| **Compile-time Flag** | `bun:bundle` build | `feature('KAIROS')` → Dead Code Elimination | Internal features do not exist at all in external builds |
| **Runtime Flag** | GrowthBook remote evaluation | `getGrowthBookFeatureValue('tengu_kairos')` | Gradual rollout, A/B testing, production circuit breakers |

**Compile-time Flag list** (22):
`KAIROS`, `KAIROS_BRIEF`, `KAIROS_CHANNELS`, `KAIROS_DREAM`, `KAIROS_GITHUB_WEBHOOKS`, `KAIROS_PUSH_NOTIFICATION`, `PROACTIVE`, `CACHED_MICROCOMPACT`, `EXPERIMENTAL_SKILL_SEARCH`, `VERIFICATION_AGENT`, `TOKEN_BUDGET`, `COORDINATOR_MODE`, `BUILDING_CLAUDE_APPS`, `RUN_SKILL_GENERATOR`, `AGENT_TRIGGERS`, `AGENT_TRIGGERS_REMOTE`, `REVIEW_ARTIFACT`, `VOICE_MODE`, `CHICAGO_MCP`, `COWORKER_TYPE_TELEMETRY`, `TRANSCRIPT_CLASSIFIER`, and others

**Runtime Flag examples**:
- `tengu_kairos`: user-level switch for Kairos autonomous mode
- `tengu_kairos_cron`: Cron scheduler circuit breaker (checked every tick)
- `tengu_kairos_cron_config`: jitter tuning parameters
- `tengu_log_datadog_events`: Datadog event routing allowlist
- `tengu_1p_event_batch_config`: 1P event batching configuration
- `tengu_event_sampling_config`: event sampling rate
- `tengu_frond_boric`: telemetry sink circuit-breaker switch

### 2.3 User Attributes (A/B Grouping Basis)

GrowthBook groups experiments using the following attributes:

```typescript
{
  id: userId,
  sessionId,
  deviceID,
  platform,                // darwin/linux/win32
  organizationUUID,
  accountUUID,
  email,
  subscriptionType,        // pro/max/team/enterprise
  rateLimitTier,
  firstTokenTime           // First-use time
}
```

### 2.4 Developer Takeaways

For Qwen Code, GrowthBook's core value is **production circuit breaking**:

```
New feature launch → 5% rollout → monitor telemetry → no anomalies → expand to 50%
                                   → problem found → disable Flag in seconds → 0% impact
```

No new release and no user upgrade are required — flipping a Feature Flag remotely is enough. Qwen Code currently lacks this capability; all features are "full rollout on release."

## 3. Telemetry Architecture

### 3.1 Dual-Sink Architecture

```
logEvent('tengu_tool_use_success', metadata)
  │
  ├─ Event Queue (buffer before startup)
  │
  ├─ Sink Router
  │     ├─ Check circuit breaker (tengu_frond_boric)
  │     ├─ Check sampling rate (tengu_event_sampling_config)
  │     └─ Route to sink
  │
  ├─ Sink 1: Datadog
  │     ├─ Allowlist filtering (64 events)
  │     ├─ _PROTO_ key stripping (privacy protection)
  │     ├─ MCP tool-name redaction
  │     └─ 15-second batch send (up to 100 records)
  │
  └─ Sink 2: 1P BigQuery
        ├─ Full metadata (including _PROTO_ fields)
        ├─ OpenTelemetry BatchLogRecordProcessor
        └─ 5-second batch send (up to 200 records, 8 exponential-backoff retries)
```

### 3.2 Privacy-Protection Mechanisms

| Mechanism | Approach | What It Protects |
|------|------|---------|
| `_PROTO_*` key tiers | Datadog strips them automatically; only 1P BigQuery (privileged columns) can see them | PII fields do not leak into general logs |
| MCP tool-name redaction | `mcp__github__get_issue` → `"mcp_tool"` | Names of MCP servers used by users do not leak |
| User bucketing | SHA256(userId) → 30 buckets | Count unique users without tracking individuals |
| File-extension truncation | Extensions longer than 10 characters become `"other"` | Prevent hash-like filenames from leaking |
| Tool-input truncation | Strings capped at 512 characters; JSON capped at 4KB | Code content does not enter telemetry |
| Type-system enforcement | `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | Compile-time prevention of code/paths entering metadata |

### 3.3 Datadog Integration

- **Endpoint**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **Allowlist**: 64 events (API/OAuth/tool/session/voice/team-memory related)
- **Tag fields**: 16 fields including arch, model, platform, subscriptionType, toolName, userType, version
- **Feature Gate**: `tengu_log_datadog_events` controls routing

## 4. 891+ Telemetry Event Categories

| Category | Event Count | Examples |
|------|--------|------|
| API & queries | 30+ | `tengu_api_error`, `tengu_api_retry`, `tengu_api_529_background_dropped` |
| OAuth & authentication | 30+ | `tengu_oauth_token_refresh_success`, `tengu_oauth_flow_start` |
| Tool execution | 50+ | `tengu_tool_use_success`, `tengu_bash_tool_command_executed` |
| MCP | 50+ | `tengu_mcp_server_connection_failed`, `tengu_mcp_tool_call_auth_error` |
| Session & startup | 20+ | `tengu_init`, `tengu_started`, `tengu_exit`, `tengu_startup_perf` |
| Memory & context | 40+ | `tengu_memdir_loaded`, `tengu_compact`, `tengu_compact_failed` |
| Compaction | 30+ | `tengu_compact_cache_sharing_success`, `tengu_partial_compact` |
| Bridge & remote | 80+ | `tengu_bridge_started`, `tengu_bridge_session_done` |
| Skill & plugins | 50+ | `tengu_skill_loaded`, `tengu_plugin_installed` |
| Permissions | 30+ | `tengu_auto_mode_decision`, `tengu_auto_mode_denial_limit_exceeded` |
| File operations | 30+ | `tengu_file_operation`, `tengu_binary_content_persisted` |
| Model & configuration | 40+ | `tengu_config_model_changed`, `tengu_fast_mode_toggled` |

## 5. Qwen Code Improvement Suggestions

### P1: Remote Feature Flags

Introduce a Feature Flag service (GrowthBook is open source; LaunchDarkly or a self-built service are also options). Core benefits:

1. **Gradual rollout**: release new features progressively at 5% → 20% → 50% → 100%
2. **Production circuit breaker**: disable a Flag within seconds when problems occur, without releasing a new version
3. **A/B testing**: compare the effectiveness of different prompts/strategies

### P2: Expand Telemetry Events

The current ~40 events cover basic operations. Recommended additions:
- API cache-hit-rate events (optimize Prompt Cache)
- Tool execution latency events (identify slow tools)
- Permission-decision events (analyze permission-mode usage patterns)
- Crash/exception events (proactively discover problems)

### P3: Strengthen Privacy Protection

Use Claude Code's type-system enforcement (`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`) as a reference: compile-time checks ensure code content does not enter telemetry data.
