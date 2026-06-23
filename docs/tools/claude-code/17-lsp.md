# 17. LSP Client: Developer Reference

> Claude Code has a built-in LSP (Language Server Protocol) client that connects to a project's language server to obtain type information, diagnostics, and symbol definitions. This gives the Agent **compiler-level code understanding** beyond plain-text search.
>
> **Qwen Code comparison**: Qwen Code has experimental LSP support (the `--experimental-lsp` startup flag), but it is disabled by default. Claude Code's LSP integration is more mature.

## 1. Why Code Agents Need LSP

### Problem Definition

An Agent's `grep_search` tool is text-level search: it does not understand code semantics.

| Task | grep_search | LSP |
|------|------------|-----|
| "Find the definition of `getUserById`" | Searches for the string "getUserById" and may match comments, strings, or imports | `textDocument/definition` → jumps precisely to the definition |
| "What is this function's parameter type?" | Cannot obtain it | `textDocument/hover` → full type signature |
| "Does this file have compilation errors?" | Cannot obtain it | `textDocument/publishDiagnostics` → error list |
| "Rename all references to this variable" | grep may miss dynamic references or aliases | `textDocument/references` → complete semantic references |

### Practical Value

The greatest value of LSP is not making the Agent "smarter" (the LLM already understands code), but providing **deterministic compiler diagnostics**:

- TypeScript type errors
- Python import errors
- Java compilation errors
- Unused variables and unresolved references

If this information can be automatically injected into the Agent's context, it is equivalent to the "Agent having a built-in compiler": type errors can be found without manually running `tsc --noEmit`.

## 2. Claude Code's LSP Implementation

Source: `services/lsp/` (7 files)

### Core Capabilities

| Capability | LSP Method | How the Agent Uses It |
|------|---------|---------------|
| Diagnostics (errors/warnings) | `textDocument/publishDiagnostics` | Automatically obtains compilation errors after edits |
| Go to definition | `textDocument/definition` | Understands where functions/classes are implemented |
| Find references | `textDocument/references` | Determines impact scope |
| Hover information | `textDocument/hover` | Obtains type signatures |
| Symbol search | `workspace/symbol` | Searches project symbols by name |

### LSP Comparison with Competitors

| Agent | LSP Support | Enabled by Default | Implementation |
|-------|---------|---------|---------|
| **Claude Code** | ✓ | Experimental | Built-in LSP client |
| **Gemini CLI** | — | — | — |
| **Qwen Code** | ✓ `--experimental-lsp` | Disabled by default | Built-in LSP client |
| **Cursor** | ✓ | ✓ | Inherits VS Code's full LSP support |
| **Copilot CLI** | — | — | — |

## 3. Qwen Code Improvement Suggestions

The core value of LSP is **automatic diagnostic injection**: after editing a file, automatically obtain compilation errors and inject them into context. Recommendations:

1. Enable LSP by default (at least for TypeScript/Python projects)
2. Automatically append `publishDiagnostics` results to the tool output of `PostToolUse(Edit)`
3. Long term: combine LSP with `/review` and use LSP diagnostics instead of LLMs to detect type errors
