# 21. Why is Claude Code Faster and More Stable? What Should Qwen Code Improve?

> Root causes of the performance gap and actionable improvement recommendations based on source-code analysis

## 1. Where Claude Code Is Faster

### 1. Native binary vs JavaScript runtime

**This is the most fundamental gap.**

| Dimension | Claude Code | Qwen Code |
|------|------------|-----------|
| **Distribution** | Precompiled native binary (Rust) | Node.js + npm dependencies |
| **Cold start** (estimate) | Sub-second | Several seconds |
| **Memory** (estimate) | Lower | Higher (Node.js + React overhead) |
| **Dependency size** | Single file | Hundreds of MB of node_modules |

Claude Code installs a **compiled native executable** via `curl install.sh | bash`; the npm installation method has been marked deprecated. This means:
- No Node.js startup overhead (V8 engine initialization ~100ms)
- No module-resolution overhead (thousands of files under node_modules)
- No JIT warm-up
- Rust zero-cost abstractions and memory safety

### 2. Aggressive parallelization

Several optimizations are recorded in the Claude Code CHANGELOG:

```
✓ macOS startup faster by ~60ms: keychain reads parallelized with module loading
✓ Startup memory reduced by ~80MB in large repositories (250k-file repos)
✓ Startup memory reduced by ~18MB across all scenarios
✓ Git operations parallelized: read refs directly and skip redundant git fetch
✓ Session resume 45% faster, peak memory reduced by 100-150MB
```

**Core strategy**: all I/O operations (credential reads, Git operations, file scanning) run in parallel without blocking the main thread.

### 3. Extreme lazy loading

- **Tools loaded on demand**: `ToolSearch` mechanism; tool schemas are fetched lazily and loaded only when the model requests them
- **Plugins discovered on demand**: commands/agents/skills Markdown is parsed only when needed
- **MCP connections on demand**: not every MCP server connects at startup
- **`--bare` mode**: skips hooks, LSP, plugin sync, and skill-directory scanning for script invocations

### 4. Fine-grained memory management

- **Streaming response buffers**: released immediately after the generator terminates
- **Session compression**: automatic compression + circuit breaker (stop retrying after 3 consecutive failures)
- **File operation optimization**: check file existence without reading full contents
- **Large-session handling**: sessions >5MB automatically extract memories and save the transcript

### 5. Timeouts and circuit breakers

- **Non-streaming API fallback**: 2-minute timeout per attempt
- **Automatic compression circuit breaker**: stop after 3 consecutive failures
- **Parallel tool isolation**: Bash errors do not cascade to read-only tools such as Read/Glob
- **MCP reconnect**: automatic reconnect + spinner cleanup

### 6. Filesystem-level sandbox

- **Filesystem isolation**: `sandbox.filesystem.allowWrite`/`allowRead` control read/write permissions
- **Network isolation**: `sandbox.network.allowedDomains` controls accessible domains
- **Weaker macOS network isolation**: `enableWeakerNetworkIsolation` option (CHANGELOG line 385)
- The exact isolation mechanism is closed-source; the CHANGELOG does not mention kernel-level technologies such as iptables/ipset/seccomp

---

## 2. Where Qwen Code Is Slow (Source-Level Issues)

### Issue 1: The startup path is heavily blocked

**File**: `packages/cli/src/gemini.tsx` (lines 215-432, about 8-9 serial awaits on the interactive path)

```
Typical interactive startup path (non-sandbox case, about 8-9 serial awaits):
  setupUnhandledRejectionHandler()  // line 213 (synchronous, correctly first)
  cleanupCheckpoints()              // line 215 ← can be parallelized
  → parseArguments()                // line 217 ← can be parallelized (independent of the above)
  → loadSandboxConfig(argv)         // line 251 ← depends on argv (sandbox path relaunches and exits)
  → loadCliConfig(full)             // line 356 ← includes extension loading
  → initializeApp()                 // line 408 (contains i18n → auth → IDE connection, all serial)
  → getStartupWarnings()            // line 417 ← can be parallelized
  → getUserStartupWarnings()        // line 418 ← can be parallelized (independent of the above)
  → kittyProtocolDetection          // line 431
  → startInteractiveUI()            // line 432

Note: the sandbox path (lines 259-327) relaunches the process and exits, so it is not counted in normal startup.
The file contains about 22 awaits in total, but because of branch conditions the interactive path actually executes about 8-9.
```

**Note**: `loadCliConfig()` is called **twice** during startup:
- **Parent process** (line 259): used only for sandbox-path authentication, then exits via `relaunchOnExitCode()`
- **Child process** (line 356): loads the full configuration including extensions

Although these are different processes, from the user's perspective they are **serial** (the child starts only after the parent completes), so total time includes two loadCliConfig calls. A source TODO acknowledges the redundancy:
```typescript
// TODO(jacobr): refactor loadCliConfig so there is a minimal version
// that only initializes enough config to enable refreshAuth
```

**Supplement**: in non-sandbox cases, the parent process also exits after `relaunchAppInChildProcess()` (line 325), and the child process calls loadCliConfig only once (line 356). The double-call issue **exists only when sandboxing is enabled**.

**Specific issues**:

| Operation | File | Issue |
|------|------|------|
| I18N initialization | `initializer.ts:42` | `await initializeI18n()` blocks inside `initializeApp()` and must finish before UI rendering |
| **Serial work inside initializeApp** | `initializer.ts:42-47` | `initializeI18n()` (file I/O + dynamic import, 20-100ms) and `performInitialAuth()` (network request, 100-3000ms) are **fully independent** but executed serially; they should use `Promise.all()` |
| I18N file checks | `i18n/index.ts:99-104` | Multiple synchronous `fs.existsSync()` calls block the event loop |
| Double loadCliConfig call | `gemini.tsx:259, 356` | Same function called twice (source TODO acknowledges redundancy) |
| Startup Warnings | `gemini.tsx:417-418` | Two independent awaits run serially and can use `Promise.all()`, but the benefit is extremely small (see round-eight verification) |
| ~~FileDiscoveryService~~ | `config.ts:1710` | ✅ Actually a lazy-initialized getter; constructor only parses gitignore rules and does not scan the filesystem |
| ~~LSP blocks startup~~ | `config.ts:635` | ✅ Getter/setter only; does not block startup |

### Issue 2: String-concatenation overhead in streaming tool-call parsing

**File**: `packages/core/src/core/openaiContentGenerator/streamingToolCallParser.ts`

```typescript
// Line 155: string concatenation on every incoming chunk
const newBuffer = currentBuffer + chunk;
this.buffers.set(actualIndex, newBuffer);  // store in Map
```

When each chunk arrives, `currentBuffer + chunk` creates a new string. **But V8 optimizes this with ConsString (rope)**: `a + b` does not immediately allocate a full copy; it creates a reference chain and materializes it only when flattening is required (for example `JSON.parse` or `for..of` over the full string). The code's `for (const char of chunk)` (L163) iterates over **chunk**, not newBuffer, so ConsString is not flattened on every call; it is materialized only once when `depth === 0` and `JSON.parse(newBuffer)` runs. Therefore the **actual GC pressure is far lower than expected**. Switching to array collection + `join()` would not bring meaningful improvement and may even be slower because of array-object overhead.

> ⚠️ **Round-eight verification correction**: this issue was downgraded from "medium benefit" to "extremely low benefit / not recommended". V8 ConsString optimization already covers this scenario.

### Issue 3: Token-counting method

**File**: `packages/core/src/core/openaiContentGenerator/openaiContentGenerator.ts` (lines 82-107)

```typescript
// countTokens() method (ContentGenerator interface method):
async countTokens(request: CountTokensParameters): Promise<CountTokensResponse> {
  try {
    const estimator = new RequestTokenEstimator();
    const result = await estimator.calculateTokens(request);
    return { totalTokens: result.totalTokens };
  } catch (error) {
    // Fallback: rough estimate of 4 characters ≈ 1 token
    const content = JSON.stringify(request.contents);
    const totalTokens = Math.ceil(content.length / 4);
    return { totalTokens };
  }
}
```

**Round-six verification found**: `countTokens()` is **not explicitly called in the agent main loop** (neither client.ts nor geminiChat.ts calls it). Actual token counts come from **usage metadata in API responses** (`uiTelemetryService.getLastPromptTokenCount()`, chatCompressionService.ts:109).

Therefore this issue has **less impact than expected**: `countTokens()` is not a hot path executed every turn; it is called on demand (for example inside SDK internals or specific features). The JSON.stringify fallback path is triggered relatively infrequently.

### Issue 4: Session history grows without bound

**File**: `geminiChat.ts` (lines 246, 295, 520)

```typescript
// Deep clone every time history is read:
getHistory() { return structuredClone(history); }  // full deep clone of multi-MB history

// History only grows:
history.push(message);  // no automatic pruning
```

- `structuredClone()` is very expensive on large histories
- There are 8 non-test call sites, typically 1-2 per turn and up to 3-4 at peak:
  - `geminiChat.ts:296` — before API request (**called every turn**, once)
  - `nextSpeakerChecker.ts:55,63` — next-speaker detection (2 calls within one function, but runs **only when there are no pending tool calls**, guarded by `client.ts:736`)
  - `client.ts:568` — IDE context check (conditional)
  - `client.ts:681` — Stop Hook (conditional, only when no pending tools)
  - `turn.ts:351` — error reporting (exception path)
  - `chatCompressionService.ts:88` — compression (70% threshold)
- There is no automatic pruning mechanism, so history can reach tens of MB after 100 turns

### Issue 5: JSON serialization when chat compression triggers

**File**: `chatCompressionService.ts` (lines 45, 80-126)

```typescript
// findCompressSplitPoint() (line 45) — called only when compression triggers:
const charCounts = contents.map((content) => JSON.stringify(content).length);
```

**Trigger conditions** (`compress()` method, lines 80-126):
- Runs only when token count exceeds 70% of the context window (`COMPRESSION_TOKEN_THRESHOLD = 0.7`)
- **Does not run every turn**; in typical use it may first trigger after ~70 turns
- But once triggered, it runs `JSON.stringify()` over every historical Content item, which is still significant overhead in large sessions

### Issue 6: Heavy dependency loading

**File**: `packages/core/package.json` (lines 30-38), `packages/cli/src/gemini.tsx`

| Dependency | Package count/size | Import method | Actual load timing |
|------|----------|-----------|-------------|
| ~~OpenTelemetry~~ | 9 packages | Top-level import in `telemetry/sdk.ts` | ✅ **Conditional loading**: loaded only when `telemetry.enabled=true` (default false), so it does not affect normal startup |
| web-tree-sitter | ~380 KB (WASM 187KB) | Top-level import in `shellAstParser.ts:17` | Loaded when the module is imported, but Parser is a lazy singleton via `initParser()` |
| React 19 + Ink | ~300 KB | Top-level imports in `gemini.tsx:15,20` | **Loaded in both interactive and non-interactive modes** (top-level import), although `nonInteractiveCli.ts` itself does not import React |

**Key findings**:
- ~~OpenTelemetry is the biggest performance issue~~ → **Actually conditional loading**. `config.ts:746-748` checks `telemetrySettings.enabled` and calls `initializeTelemetry()` only when enabled; default is off
- React/Ink are top-level imported in `gemini.tsx` via `import { render } from 'ink'` and `import React from 'react'` (lines 15, 20), so they are already parsed even if execution enters the non-interactive path. `nonInteractiveCli.ts` does not reference React, but the entry file `gemini.tsx` has already loaded it

### Issue 7: MCP timeout is too long

**File**: `packages/core/src/tools/mcp-client.ts`

```typescript
// Line 59: default timeout is 10 minutes
export const MCP_DEFAULT_TIMEOUT_MSEC = 10 * 60 * 1000;

// Lines 153-155: connection uses configured timeout
await this.client.connect(this.transport, {
  timeout: this.serverConfig.timeout,  // default 10 minutes
});
```

MCP connections **do** have timeout protection (10 minutes), but 10 minutes is too long for the startup path. An unresponsive MCP server can make users wait a long time before seeing a timeout error. Recommendation: use a shorter timeout during startup (for example 5-10 seconds), then use the longer timeout for subsequent operations.

### ~~Issue 8: Anthropic client has no connection reuse~~ ✅ Verified not to exist

**File**: `packages/core/src/core/anthropicContentGenerator/anthropicContentGenerator.ts`

```typescript
// Lines 58-88: the client is created once in the constructor and reused afterward
export class AnthropicContentGenerator implements ContentGenerator {
  private client: Anthropic;  // instance field, reused

  constructor(config, cliConfig) {
    this.client = new Anthropic({
      apiKey: config.apiKey,
      timeout: config.timeout || DEFAULT_TIMEOUT,
      maxRetries: config.maxRetries,
      ...runtimeOptions,
    });
  }
}
```

**Correction**: the Anthropic client is created once in the constructor and reused. It is not recreated for every request. This issue does not exist.

---

## 3. Qwen Code Improvement Roadmap

### First priority: startup speed (the user's first impression)

#### 1.1 Parallelize the startup flow

```typescript
// Current (gemini.tsx): all awaits are serial
await cleanupCheckpoints();                    // line 215
let argv = await parseArguments();             // line 217
// ...everything afterward depends on argv and cannot start earlier

// Improvement 1: parallelize independent operations
const [_, argv] = await Promise.all([
  cleanupCheckpoints(),   // independent of argument parsing
  parseArguments(),
]);

// Improvement 2: parallelize inside initializeApp() (currently serial in initializer.ts:42-47)
// initializeI18n (file I/O) and performInitialAuth (network request) are fully independent
const [_, authError] = await Promise.all([
  initializeI18n(languageSetting),   // 20-100ms (file I/O + dynamic import)
  performInitialAuth(config, authType),  // 100-3000ms (network token refresh)
]);

// Improvement 3: parallelize startup warnings (currently serial at lines 417-418)
// Note: benefit is extremely small; getStartupWarnings() only reads a temp file (<2ms),
// and getUserStartupWarnings() already uses Promise.all() internally; bottleneck is the ripgrep check
const [warnings, userWarnings] = await Promise.all([
  getStartupWarnings(),
  getUserStartupWarnings({...}),
]);

// Improvement 4: eliminate double loadCliConfig call (currently lines 259 and 356)
// The source TODO already acknowledges redundancy; refactor into minimal config + full config separation
```

**Expected benefit** (estimate, not measured):
- Parallelizing inside `initializeApp`: **saves 20-100ms** (i18n cost hidden behind auth network latency) — **the most valuable startup-stage parallelization**
- `cleanupCheckpoints` ‖ `parseArguments`: **saves <5ms** (both are very lightweight)
- Startup warnings parallelization: **saves <2ms** (`getStartupWarnings` only reads a temp file; `getUserStartupWarnings` is already parallel internally)
- Optimizing the double loadCliConfig call: benefits only sandbox scenarios

#### 1.2 Replace synchronous I18N loading with embedded default languages

```typescript
// Current: language pack is loaded asynchronously on every startup
await import(languagePackUrl);

// Improvement: embed English/Chinese and lazy-load other languages
import defaultLocale from './locales/en.json';  // embedded at compile time
```

#### 1.3 Separate non-interactive entry point (avoid loading React/Ink)

```typescript
// Current: gemini.tsx imports React and Ink at the top level
import { render } from 'ink';        // line 15
import React from 'react';           // line 20
// Even when taking the non-interactive path (lines 457-510), these modules have already been parsed at the entry point

// Improvement plan A: use an independent entry for non-interactive mode
// bin/qwen-noninteractive → directly import nonInteractiveCli.ts (without going through gemini.tsx)

// Improvement plan B: change React/Ink in gemini.tsx to dynamic imports
// Import only inside startInteractiveUI()
async function startInteractiveUI(...) {
  const { render } = await import('ink');
  const React = await import('react');
  // ...
}
```

> Note: ~~eager loading of 9 OTel packages~~ was verified in round three to actually be **conditional loading** (`config.ts:746`, default `enabled: false`), so it does not affect normal startup and does not need changes.

### Second priority: runtime performance

#### ~~2.1 Use arrays for streaming parsing~~ ❌ Not recommended (round-eight verification)

> **Round-eight verification correction**: V8 ConsString (rope) optimization already covers the `currentBuffer + chunk` scenario. Concatenation does not immediately allocate a full copy; the code's `for (const char of chunk)` (L163) iterates over chunk rather than newBuffer, so ConsString is not flattened on each call. It is materialized only once when `depth === 0` and `JSON.parse(newBuffer)` runs. Switching to array collection + `join()` would not bring meaningful improvement.

#### ~~2.2 Token-count caching + incremental counting~~ → Low priority

Round-six verification found that `countTokens()` is not called in the agent main loop; token counts come from API-response usage metadata. This optimization is valuable only for SDK internals or specific features that call it on demand, not for the runtime hot path.

#### 2.3 Replace structuredClone with readonly references

```typescript
// Current (geminiChat.ts:514-520): getHistory() deep-clones on every call
getHistory(curated?: boolean): Content[] {
  return structuredClone(history);  // expensive! At least once per turn; up to 3-4 times when tool calls finish
}

// Improvement: return a readonly reference (after verifying all callers only read and do not mutate)
getHistory(curated?: boolean): readonly Content[] {
  const history = curated ? extractCuratedHistory(this.history) : this.history;
  return history;  // return reference; TypeScript readonly prevents mutation
}
```

**Safety verification**:

> ⚠️ **Major round-eight correction**: the round-four conclusion that "all callers only read and do not mutate" was **wrong**. `nextSpeakerChecker.ts:92` contains a **real deep mutation**:
> ```typescript
> // nextSpeakerChecker.ts:86-92
> if (lastComprehensiveMessage && lastComprehensiveMessage.role === 'model' &&
>     lastComprehensiveMessage.parts && lastComprehensiveMessage.parts.length === 0) {
>   lastComprehensiveMessage.parts.push({ text: '' });  // ← directly mutates the parts array!
> }
> ```
> If `structuredClone` is removed, this `push` will pollute the internal state of `this.history`.
>
> `.filter(...).pop()` in `copyCommand.ts:24` is safe: `.filter()` creates a new array, so `.pop()` mutates only the temporary array.
>
> **Recommended plan**: do not directly remove `structuredClone` and return a raw reference. The correct approach is:
> 1. **Fix `nextSpeakerChecker.ts:92`**: use a local copy (for example `{ ...lastMsg, parts: [{ text: '' }] }`) instead of mutating the input
> 2. After that fix, `getHistory()` can safely remove `structuredClone` and return a readonly reference
> 3. Or provide two methods: `getHistoryRef()` (no copy, readonly contract) and `getHistoryClone()` (deep copy), letting callers choose as needed

#### 2.4 Change compression service to estimate character length

```typescript
// Current: calculate character count with JSON.stringify
const len = JSON.stringify(content).length;

// Improvement: direct estimate (accurate enough for threshold decisions)
function estimateSize(content: Content): number {
  if (typeof content === 'string') return content.length;
  if (content.text) return content.text.length;
  return 1000; // default estimate for non-text
}
```

### Third priority: stability

#### 3.1 Shorten MCP timeout during startup

```typescript
// Current: MCP connection default timeout is 10 minutes (mcp-client.ts:59)
// On the startup path, an unresponsive MCP server can make users wait 10 minutes

// Improvement: use a short timeout during startup, restore long timeout afterward
const STARTUP_TIMEOUT = 5_000;  // 5 seconds
await this.client.connect(this.transport, {
  timeout: isStartup ? STARTUP_TIMEOUT : this.serverConfig.timeout,
});
```

#### ~~3.2 HTTP connection reuse~~ ✅ Verified unnecessary

The Anthropic client is already created once and reused in the constructor (`anthropicContentGenerator.ts:58-88`), so this improvement is unnecessary.

#### 3.3 Error-recovery circuit breaker

```typescript
// Reference Claude Code's circuit-breaker pattern
class CircuitBreaker {
  private failures = 0;
  private readonly threshold = 3;

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.failures >= this.threshold) {
      throw new Error('Circuit breaker open');
    }
    try {
      const result = await fn();
      this.failures = 0;
      return result;
    } catch (e) {
      this.failures++;
      throw e;
    }
  }
}
```

#### ~~3.4 Capture unhandled exceptions earlier~~ ✅ Verified unnecessary

`setupUnhandledRejectionHandler()` is already called on the first line of `main()` (line 213), before all async operations. This improvement is unnecessary.

### Fourth priority: long-term architecture

#### 4.1 Consider rewriting core modules in Rust

Rewrite the hottest paths as Rust N-API modules:
- File search (grep/glob, currently implemented in Node.js)
- ~~Streaming parsing~~ V8 ConsString optimization already covers this, so Rustification has limited benefit

```
Estimated performance benefit (estimate):
  File search: 5-10x faster (ripgrep level)
```

> Note: token counting was verified in R6 not to be a hot path (data comes from API-response usage), so Rustification has limited benefit.

#### 4.2 Introduce `--bare` mode

Following Claude Code, provide minimal startup for scripts/CI:

```bash
qwen --bare -p "fix this bug"  # skip hooks/LSP/plugins/skill scanning
```

#### 4.3 Move session storage to SQLite

```
Current: JSONL append-only writes (no indexes; pagination relies on file reads)
Improvement: SQLite (WAL mode, indexed queries, incremental writes)
Benefit: 10x+ faster large-session loading and support for complex queries
```

---

## 4. Improvement Priority Matrix

| Improvement | Difficulty | Benefit | Priority |
|------|------|------|--------|
| **Parallelize inside initializeApp** (i18n ‖ auth, `initializer.ts:42-47`) | Low | **High** (saves 20-100ms; auth is the slowest single startup operation) | **P0** |
| structuredClone optimization (first fix the `nextSpeakerChecker.ts:92` mutation, then remove deep cloning) | Low | **High** (large-session performance) | **P0** |
| Shorten MCP startup timeout (10min→5-10s) | Low | **High** (stability) | P0 |
| Eliminate double loadCliConfig call (sandbox path; source TODO acknowledges it) | Medium | Medium (sandbox only) | P1 |
| Replace compression-service JSON.stringify with estimates | Low | Medium | P1 |
| Make i18n fs.existsSync async + embed default language | Low | Medium | P1 |
| Parallelize startup awaits (cleanupCheckpoints ‖ parseArguments) | Low | Extremely low (<5ms) | P2 |
| Parallelize startup warnings | Low | Extremely low (<2ms; already Promise.all internally) | P2 |
| Separate non-interactive entry in gemini.tsx (avoid loading React/Ink) | Medium | Medium | P2 |
| `--bare` mode | Medium | Medium | P2 |
| Circuit-breaker pattern | Medium | Medium | P2 |
| ~~Change streaming parsing string concatenation to array collection~~ | — | ~~Extremely low (V8 ConsString already optimized)~~ | ~~Not recommended~~ |
| ~~Incremental token-count cache~~ | Medium | Low (`countTokens` is not a hot path) | P3 |
| Rust N-API for hot paths (file search) | High | **High** | P3 |
| Move session storage to SQLite | High | High | P3 |

---

## 5. One-Sentence Summary

**Why Claude Code is fast**: native Rust binary + aggressive parallelization + extreme lazy loading + filesystem-level sandbox + timeout circuit breakers.

**Why Qwen Code is slow**: Node.js runtime overhead + serial i18n/auth inside startup-path `initializeApp()` (auth network request 100-3000ms) + `structuredClone` deep cloning 1-4 times per turn + React/Ink loaded even in non-interactive mode + 10-minute MCP startup timeout.

> Note: ~~eager loading of 9 OTel packages~~ was verified in round three to actually be conditional loading (default off), so it does not affect normal startup.
> Note: ~~streaming parser string concatenation~~ was verified in round eight to be covered by V8 ConsString optimization; changes are not recommended.
> Note: ~~directly removing structuredClone~~ was verified in round eight to be unsafe because `nextSpeakerChecker.ts:92` mutates data; fix that first, then remove it.

**Three highest-impact low-cost changes**:
1. Parallelize `initializeI18n()` and `performInitialAuth()` inside `initializeApp()` (`Promise.all()`, a few lines of code, saves 20-100ms)
2. After fixing the mutation in `nextSpeakerChecker.ts:92`, remove `structuredClone` from `getHistory()` (deep clone 1-4 times per turn)
3. Shorten MCP startup timeout from 10 minutes to 5-10 seconds

---

*Analysis is based on local Qwen Code source and the Claude Code plugin repository + CHANGELOG, as of March 2026. Round-eight verification was supplemented by an external audit.*

---

## Appendix: Source Verification Log (Eight Rounds)

### Round 1 Verification

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| Serial startup | `gemini.tsx` | 215-432 | ✅ Confirmed |
| i18n blocking | `initializer.ts` → `i18n/index.ts` | 42, 253-257 | ✅ Confirmed |
| FileDiscoveryService | `config.ts` | 37, 534, 778, 1710 | ✅ Exists, lazy initialization |
| ~~LSP blocks startup~~ | `config.ts` | 635-636 | ❌ Getter/setter only; not blocking |
| Streaming string concatenation | `streamingToolCallParser.ts` | 155 | ⚠️ O(n), not O(n²), but GC pressure high |
| structuredClone | `geminiChat.ts` | 520 | ✅ Confirmed |
| JSON.stringify compression | `chatCompressionService.ts` | 45 | ✅ Confirmed |
| MCP timeout | `mcp-client.ts` | 59, 153-155 | ⚠️ Has 10-minute timeout, but too long |
| ~~Anthropic created per request~~ | `anthropicContentGenerator.ts` | 58-88 | ❌ Created once in constructor and reused |
| OpenTelemetry package count | `package.json` | 30-38 | ⚠️ Actually 9 packages |
| Token counting | `openaiContentGenerator.ts` | 82-107 | ✅ Confirmed |

### Round 2 Verification (Deep Validation)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| Total startup await count | `gemini.tsx` | 215-510 | ⚠️ Correction: actually **22** awaits (not 12+) |
| Double loadCliConfig call | `gemini.tsx` | 259, 356 | ✅ Confirmed: called twice; source has TODO acknowledging redundancy |
| Await pairs that can be parallelized | `gemini.tsx` | 215/217, 417/418 | ✅ Confirmed: at least 2 pairs can use Promise.all() |
| ~~FileDiscoveryService scans filesystem~~ | `fileDiscoveryService.ts` | 25-36 | ❌ Correction: constructor only parses gitignore rules; no scan |
| ~~Exception handler registered late~~ | `gemini.tsx` | 213 | ❌ Correction: first line in main(), before all async operations |
| i18n fs.existsSync | `i18n/index.ts` | 99-104, 212 | ✅ Confirmed: multiple synchronous fs.existsSync() calls |
| OpenTelemetry import method | `telemetry/sdk.ts` | 7-30 | ✅ Confirmed: all 9 packages are top-level static imports |
| web-tree-sitter loading | `shellAstParser.ts` | 17 | ⚠️ Correction: module is static import, but Parser initialization is a lazy singleton via `initParser()` |
| Token fallback scope | `openaiContentGenerator.ts` | 100 | ⚠️ Correction: serializes `request.contents` (not the whole request) |
| initializeApp internals | `initializer.ts` | 33-74 | ✅ Confirmed: i18n → auth → IDE connection, all serial |
| Non-interactive path | `gemini.tsx` | 457-510 | ✅ Confirmed: also has serial awaits that can be optimized |

### Round 3 Verification (Cross-Validation)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| ~~OTel eager loading~~ | `config.ts` + `telemetry/sdk.ts` | 746-748, 7-30 | ❌ **Major correction**: conditional loading (`telemetry.enabled` default false), does not affect normal startup |
| structuredClone call frequency | `geminiChat.ts`, `client.ts`, `turn.ts` | 296, 568, 681, 351 | ✅ Confirmed: at least once per turn, more on error/compression/Hook paths |
| React loaded in non-interactive mode | `gemini.tsx` | 15, 20 | ✅ Confirmed: top-level import, parsed in non-interactive mode too; `nonInteractiveCli.ts` itself does not import React |
| Compression service granularity | `chatCompressionService.ts` | 45 | ✅ Confirmed: `contents.map(c => JSON.stringify(c).length)` serializes each array item |
| readStdin condition | `gemini.tsx` | 463-468 | ✅ Confirmed: condition `!process.stdin.isTTY`, dedicated to non-interactive path |
| CHANGELOG: keychain 60ms | `CHANGELOG.md` | 136 | ✅ Confirmed: original text "Faster startup on macOS (~60ms) by reading keychain credentials in parallel" |
| CHANGELOG: large repos 80MB | `CHANGELOG.md` | 50 | ✅ Confirmed: original text "Reduced memory usage on startup in large repositories (~80 MB saved on 250k-file repos)" |
| CHANGELOG: resume 45% | `CHANGELOG.md` | 137 | ✅ Confirmed: original text "up to 45% faster loading and ~100-150MB less peak memory" |

### Round 4 Verification (Precision Validation)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| ~~22 serial awaits~~ | `gemini.tsx` | 215-432 | ⚠️ Correction: the file contains 22 awaits, but the interactive path actually executes **8-9** (sandbox/non-interactive branches are not taken) |
| structuredClone can be safely removed | `client.ts` | 568, 681 | ✅ Confirmed: all callers only use `.filter()`/property access and do not mutate the returned array |
| ~~Compression JSON.stringify every turn~~ | `chatCompressionService.ts` | 80-126 | ⚠️ Correction: triggers only when token > 70% of context, not every turn (first trigger around ~70 turns) |
| ~~Kernel-level sandbox iptables/ipset~~ | `CHANGELOG.md` | Full-text search | ❌ **Correction**: CHANGELOG contains no mention of iptables/ipset/seccomp/BPF; actual isolation is filesystem + network-domain isolation |
| ~~web-tree-sitter ~2.5MB~~ | `node_modules/web-tree-sitter/` | Measured | ❌ **Correction**: WASM is 187KB, total package ~380KB (original description overstated by 6-7x) |
| Duplicate header in priority matrix | Document itself | 449-452 | ✅ Fixed formatting error |

### Round 5 Verification (Consistency and Precision)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| ~~Residual "kernel-level sandbox"~~ | Document summary line 468 | — | ❌ Corrected: inconsistent with section 6; changed to "filesystem-level sandbox" |
| ~~Double loadCliConfig call in same process~~ | `gemini.tsx` | 247-327 | ⚠️ Correction: line 259 is in the parent process (sandbox auth), line 356 in the child process; non-sandbox cases call it only once |
| Exact count of structuredClone call sites | Global grep in `core/src/` | 8 non-test call sites | ✅ Confirmed: at least 3-4 times per turn (API + nextSpeaker×2 + IDE check) |
| ~~Residual 3.1 LSP timeout suggestion~~ | Section 3.1 of document | — | ❌ Corrected: R1/R2 confirmed LSP does not block startup; removed LSP timeout suggestion |
| React top-level import lines | `gemini.tsx` | 15, 20 | ✅ Confirmed: `import { render } from 'ink'` and `import React from 'react'` |

### Round 6 Verification (Final Validation)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| ~~Token counting runs every turn~~ | Global search in `client.ts`, `geminiChat.ts` | No matches | ⚠️ **Major correction**: `countTokens()` is not called in the main loop; token counts come from API-response usage metadata |
| CHANGELOG: 18MB | `CHANGELOG.md` | 66 | ✅ Confirmed: original text "Improved startup memory usage by ~18MB across all scenarios" |
| CHANGELOG: Git parallelization | `CHANGELOG.md` | 178 | ✅ Confirmed: original text "reading git refs directly and skipping redundant `git fetch`" |
| structuredClone comment "3-6" | Document itself | 349 | ⚠️ Corrected to "3-4" (consistent with exact R5 count) |
| Expected-benefit numbers | No source basis | — | ⚠️ Corrected: marked as estimates, not measured |
| Appendix title | Document itself | 495 | ⚠️ Corrected from "four rounds" to "six rounds" |

### Round 7 Verification (Final Consistency)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| Does nextSpeakerChecker run every turn? | `client.ts` | 736-737 | ⚠️ Correction: it has a conditional guard (`!pendingToolCalls && !skipNextSpeakerCheck`); not guaranteed every turn |
| 2.2 Token-cache recommendation conflicts with R6 | Document section 2.2 | — | ⚠️ Corrected: marked low priority, consistent with R6 findings |
| 4.1 Rustification of token counting conflicts with R6 | Document section 4.1 | — | ⚠️ Corrected: removed recommendation to Rustify token counting |
| structuredClone frequency | Multiple files | — | ⚠️ Precise: called once per turn unconditionally (API request), plus 0-3 extra conditional calls |

### Round 8 Verification (External Audit)

| Issue | Verified file | Actual lines | Conclusion |
|------|---------|---------|------|
| ~~Safe to remove structuredClone~~ | `nextSpeakerChecker.ts` | 86-92 | ❌ **Major correction**: `lastComprehensiveMessage.parts.push({ text: '' })` directly mutates the input parts array, so structuredClone cannot be removed directly. Fix this mutation first |
| ~~Streaming parser string concatenation has high GC pressure~~ | `streamingToolCallParser.ts` | 155, 163, 186 | ❌ **Correction**: V8 ConsString (rope) optimization means `a + b` does not immediately allocate new memory; `for..of` iterates chunk rather than newBuffer; only `JSON.parse` when `depth===0` materializes once. Actual GC pressure is extremely low, so changes are not recommended |
| ~~Startup await parallelization has high benefit~~ | `gemini.tsx`, `cleanup.ts`, `startupWarnings.ts`, `userStartupWarnings.ts` | 215-217, 417-418 | ⚠️ **Downgraded**: `cleanupCheckpoints()` is only one `fs.rm()` (<1ms); `getStartupWarnings()` only reads a temp file (<2ms); `getUserStartupWarnings()` already uses `Promise.all()` internally. Total parallelization benefit <5ms |
| **Missing: serial work inside initializeApp** | `initializer.ts` | 42-47 | ✅ **Added**: `initializeI18n()` (file I/O + dynamic import, 20-100ms) and `performInitialAuth()` (network token refresh, 100-3000ms) are fully independent but serial; this is the **most valuable parallelization point** during startup |
| `copyCommand.ts` mutation? | `copyCommand.ts` | 23-24 | ✅ Confirmed safe: `.filter()` creates a new array in `.filter(...).pop()`, and `.pop()` mutates only the temporary array, not original history |
| `parseArguments()` cost | `config.ts`, `package.ts` | 181-660, 25-37 | ✅ Confirmed: contains `getCliVersion()` → `readPackageUp()` file search, but the result is cached; first call 5-20ms |
| Priority matrix ordering | Document itself | — | ⚠️ Corrected: initializeApp parallelization promoted to P0; startup await parallelization downgraded to P2; streaming parser marked not recommended |
