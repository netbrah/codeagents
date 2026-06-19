# 18. File Index and Fuzzy Search: Developer Reference

> Claude Code uses a native Rust/NAPI module to implement fzf-style fuzzy file search with asynchronous incremental indexing. This is the key for an Agent to quickly locate target files in large codebases (100k+ files).
>
> **Qwen Code comparison**: Qwen Code relies on a `glob` + `rg` combination to search files. Claude Code's native file index (Rust NAPI) is 10-100x faster in large repositories.

## 1. Why a Dedicated File Index Is Needed

### Problem Definition

| Scenario | glob/rg | Dedicated File Index |
|------|---------|------------|
| "Find files related to auth" | `glob("**/auth*")` — exact matching, cannot find `authentication.ts` | Fuzzy search "auth" → matches `auth.ts`, `authentication.ts`, `AuthProvider.tsx` |
| A monorepo with 100k files | Each search scans the file system for 2-5 seconds | <10ms after indexing |
| Real-time completion while the user types | Each keystroke triggers a new search → latency | Incremental indexing + in-memory query → instant |

### Claude Code's Implementation

Source: `native-ts/file-index/` (Rust NAPI module)

```
At startup
  │
  ├─ Asynchronously scan the project file tree
  │   └─ Exclude .gitignore, node_modules, etc.
  │
  ├─ Build an in-memory index (implemented in Rust)
  │
  └─ Provide query APIs
        ├─ Fuzzy matching (fzf algorithm)
        ├─ Path weighting (files under src/ take priority)
        └─ Incremental updates (when files change)
```

### Competitor Comparison

| Agent | File Search | Index | Fuzzy Matching | Performance |
|-------|---------|------|---------|------|
| **Claude Code** | Rust NAPI module | ✓ async incremental | ✓ fzf-style | <10ms (100k files) |
| **Gemini CLI** | glob + rg | — | — | 100ms-2s |
| **Qwen Code** | glob + rg | — | — | 100ms-2s |
| **Cursor** | VS Code file search | ✓ | ✓ | Fast (VS Code optimized) |

## 2. Relationship with ToolSearch

The file index and ToolSearch are two different layers of "search":

| Dimension | File Index | ToolSearch |
|------|---------|-----------|
| Search Target | File paths in the project | Claude Code's lazily loaded tools |
| User | Agent Read/Edit/Write tools | When the Agent needs to activate uncommon tools |
| Implementation | Rust NAPI module | TypeScript string matching |

## 3. Qwen Code Improvement Suggestions

### P2: File Index Optimization

Currently, Qwen Code scans the file system for every `glob`. Recommendations:
1. Build a file-list cache asynchronously at startup
2. Use `fs.watch` to watch file changes and update incrementally
3. Fuzzy matching can use a pure JS library such as `fzf-for-js`; Rust NAPI is not required

### P3: Path Completion Optimization

Qwen Code PR#2879 (path completion) has already implemented an LRU cache. A file index can be integrated on top of it, upgrading from "directory scanning + cache" to "full index + incremental updates".
