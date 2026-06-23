# Claude Code Source Analysis (for Code Agent Developers)

> This documentation series is based on Claude Code source analysis (~1800 files) and binary reverse engineering. It extracts architecture designs and implementation patterns that are useful for developers of Code Agents such as Qwen Code and Gemini CLI.
>
> **Audience**: Engineers developing or improving CLI Code Agents
>
> **Not**: A Claude Code user manual or feature introduction

> **Baseline version note**:
> - The main content is based on **v2.1.81 binary reverse engineering** (2026-03-25)
> - **v2.1.82 → v2.1.132 incremental updates** are summarized in [§23-recent-updates.md](./23-recent-updates.md) (5 new cloud features / default Opus 4.6 → 4.7 / new slash commands / new env vars / hooks conditional / MCP per-tool size override / 1 breaking change). Minor updates inside chapters are marked in place as `(v2.1.X+)` or `(after 2026-04)`.

## Document Index

| Document | Developer Focus | Qwen Code Comparison |
|------|------------|----------------|
| [01-Overview and Comparison](./01-overview.md) | Core capability matrix and architecture differences at a glance | Feature gap overview |
| [02-Command System](./02-commands.md) | Registration/loading/permission design for 79 commands | Slash command architecture reference |
| [03-Technical Architecture](./03-architecture.md) | Bootstrap chain, QueryEngine loop, Feature Flag DCE, Provider communication | Core loop + startup optimization |
| [04-Tool System](./04-tools.md) | Zod Schema, permission model, lazy loading, and security validation for 42 tools | Tool registration + ToolSearch |
| [05-Skill System](./05-skills.md) | Skill definition format, loading priority, built-in Skills | Skill architecture |
| [06-Settings and Security](./06-settings.md) | 5-layer settings, sandbox isolation, 24 Hook events, permission model | Security hardening path |
| [07-Session and Memory](./07-session.md) | 5-layer context compression, CLAUDE.md memory, team memory, MCP, file checkpoints | Context management + memory system |
| [08-Remote Control](./08-remote-control.md) | WebSocket/SSE bridge, session lifecycle, defense in depth | Remote control architecture |
| [09-Multi-Agent System](./09-multi-agent.md) | Leader-Worker, three Swarm backends, mailbox communication, task management, Kairos | Multi-Agent orchestration |
| [10-Prompt Suggestions](./10-prompt-suggestions.md) | Predictive generation, 12 filtering rules, Speculation execution | Intelligent completion |
| [11-Terminal Rendering](./11-terminal-rendering.md) | DEC 2026 synchronized output, differential rendering, cache pooling | Anti-flicker + rendering performance |
| [12-Hook System](./12-hooks.md) | 27 events, 6 handler types (including LLM reasoning decisions), hookify automatic rules | Hook event coverage + prompt/agent Hooks |
| [13-System Prompt](./13-system-prompt.md) | Dynamic assembly, static/dynamic partitioning, Prompt Cache optimization, `<system-reminder>` injection | Prompt Cache partitioning + QWEN.md injection approach |
| [14-MCP Integration](./14-mcp.md) | 6 transports, OAuth + XAA, Channel messages, reconnect, resource subscriptions | MCP resource/Prompt support + reconnect strategy |
| [15-Telemetry and Feature Flag](./15-telemetry-feature-flags.md) | 891+ events, GrowthBook remote rollout, dual Sink architecture, privacy protection | Remote Feature Flag + telemetry extension |
| [16-Auto Dream](./16-auto-dream.md) | Four-stage memory consolidation, three-gate trigger, mutual exclusion with Kairos | Automatic memory cleanup |
| [17-LSP Client](./17-lsp.md) | Compiler-level code understanding, automatic diagnostic injection | LSP enabled by default + diagnostic integration |
| [18-File Index](./18-file-index.md) | Rust NAPI fzf fuzzy search, async incremental indexing | File search performance optimization |
| [19-Reference Quick Sheet](./19-reference.md) | Data structures, glossary, entity relationship diagrams | Concept quick reference |
| [20-Query State Transitions](./20-query-transitions.md) | 6 transition reasons, state-machine model | TransitionReason enum |
| [21-Tool Execution Runtime](./21-tool-execution-runtime.md) | Concurrency classification, wave scheduling, progress messages | Wave-based parallelism |
| [22-Message Pipeline](./22-message-pipeline.md) | Message normalization, system-reminder injection, Cache partitioning | QWEN.md injection approach |
| [23-Recent Updates](./23-recent-updates.md) | v2.1.82 → v2.1.132 increments (Computer Use / Auto Mode / Ultraplan / Ultrareview / Routines / Opus 4.7 / new commands / hooks conditional / MCP size override) | Cloud modes + multi-agent fleet reference |
| [EVIDENCE.md](./EVIDENCE.md) | Raw evidence from binary analysis | — |

## How to Use This Series

1. **Locate gaps quickly**: Start with the capability matrix in [01-Overview](./01-overview.md) to find the area you care about
2. **Dive into architecture**: Open the corresponding chapter to study Claude Code implementation details
3. **Compare and improve**: Each chapter's "Developer Reference" section provides Qwen Code comparison analysis and implementation suggestions
4. **Review evidence**: All technical claims are supported by source/binary analysis in [EVIDENCE.md](./EVIDENCE.md)

## Related Documents

- [Qwen Code Improvement Recommendations Report (Claude Code Comparison)](../../../docs/comparison/qwen-code-improvement-report.md)
- [Qwen Code Upstream Backport Report (Gemini CLI Comparison)](../../../docs/comparison/qwen-code-gemini-upstream-report.md)
- [/review Feature Improvement Recommendations](../../../docs/comparison/qwen-code-review-improvements.md)

## Analysis Methodology

- **Source analysis**: Based on public decompilation/source map analysis results
- **Binary analysis**: `strings` / `readelf` / decompilation of ELF x86-64 binaries
- **Official documentation**: [code.claude.com/docs](https://code.claude.com/docs/en)
- **Web research**: GitHub Blog, community analysis, and third-party teardowns
