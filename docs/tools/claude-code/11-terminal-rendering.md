# 11. Terminal Rendering and Flicker Prevention — Developer Reference

> DEC 2026 synchronized output, differential rendering, double buffering, hardware scrolling, cache pooling, 60fps throttling, and 13 other flicker-prevention mechanisms. Claude Code implements these low-level optimizations through its own Ink fork.
>
> **Qwen Code comparison**: Qwen Code uses standard Ink and suffers from flicker with large outputs. Gemini CLI has already implemented partial solutions upstream, including SlicingMaxSizedBox and hard limits (see [Tool Output Height Limiting](../../comparison/tool-output-height-limiting-deep-dive.md)). This article's DEC synchronized output and differential rendering are lower-level solutions.

## Why a Custom Rendering Engine Is Needed

### Core Problem

Standard Ink uses a "clear screen → full redraw" rendering strategy: every React state update writes the entire screen contents. This is fine for simple CLI tools, but a Code Agent TUI faces unique challenges:

| Scenario | Update Frequency | Standard Ink | Claude Code Ink fork |
|------|---------|---------|---------------------|
| Streaming LLM output | 10-30 times per second | Full-screen redraw every time → flicker | Differential rendering updates only changed cells |
| Spinner animation | 4-8 times per second | Full-screen redraw just to rotate one character | Damage tracking down to individual cells |
| 500 lines of shell output | Triggered per line | Layout + render 500 lines | Hardware scrolling + render only newly added lines |
| Terminal resize | Occasional | Full relayout → tearing | Atomic wrapping with BSU/ESU |

### Design Decision: Why Not Replace This with Application-Level Height Limits

Gemini CLI and Qwen Code chose an application-level approach: `SlicingMaxSizedBox` clips data to 15 lines before rendering, avoiding Ink layout over large amounts of content. This is effective but incomplete:

| Approach | Layer | Problems Solved | Problems Not Solved |
|------|------|-----------|--------------|
| Application-level height limiting (Gemini/Qwen) | React component | Large output → clipped to 15 lines | Spinner, streaming output, and resize still cause full-screen redraws |
| Custom rendering engine (Claude Code) | Ink fork | Flicker in all scenarios | High maintenance cost; cannot merge upstream Ink updates |

**The best solution is to combine both**: application-level height limits reduce the amount of data (already done in Gemini CLI), while rendering-engine-level optimization reduces redraw cost (already done in Claude Code; still pending in Qwen Code).

### Rendering Approach Comparison with Competitors

| Agent | Rendering Engine | Flicker-Prevention Mechanisms | Result |
|-------|---------|-----------|------|
| **Claude Code** | Custom Ink fork (~6,800 lines) | DEC synchronized output + differential rendering + hardware scrolling + damage tracking | Best |
| **Gemini CLI** | `@jrichman/ink@6.6.7` (custom fork) | SlicingMaxSizedBox + 15-line hard limit + VirtualizedList | Good |
| **Qwen Code** | Standard `ink@6.2.3` | MaxSizedBox visual clipping (after rendering) | Flickers on large output |
| **Cursor** | VS Code Webview | Browser DOM + GPU compositing | No flicker (web technology) |
| **Copilot CLI** | Custom Ink fork | Similar to Claude Code | Good |

## Problem Background

A terminal UI is different from the browser DOM: there is no hardware compositing layer, no `requestAnimationFrame`, and no GPU acceleration. Every output is a byte stream of ANSI escape sequences written to stdout, and the terminal emulator parses and renders it character by character in real time. If one render involves two steps, "clear screen → redraw," the terminal first shows a blank screen and then the new content, producing visible flicker.

Claude Code builds its TUI (Terminal UI) with React + Ink and must maintain visual stability in the following scenarios:

- **Streaming text output**: assistant responses arrive in chunks and update multiple times per second
- **Spinner / progress indication**: high-frequency rotation animation
- **Scrolling**: scrolling up and down through conversation history
- **Layout changes**: expanding/collapsing tool calls and opening permission panels
- **Window resizing**: full relayout after terminal resize

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│  React component tree (Ink)                                    │
│  JSX → Yoga layout → virtual screen (Screen Buffer)            │
└────────────────────┬───────────────────────────────────────────┘
                     │ renderNodeToOutput()
                     ▼
┌────────────────────────────────────────────────────────────────┐
│  Screen Buffer (two-dimensional cell array)                    │
│  output.ts: damage tracking + CharCache + blit optimization    │
└────────────────────┬───────────────────────────────────────────┘
                     │ logUpdate()
                     ▼
┌────────────────────────────────────────────────────────────────┐
│  Diff engine (log-update.ts, 773 lines)                        │
│  Previous/current frame diff → Diff patch array                │
│  DECSTBM hardware scrolling / per-cell incremental writes      │
└────────────────────┬───────────────────────────────────────────┘
                     │ writeDiffToTerminal()
                     ▼
┌────────────────────────────────────────────────────────────────┐
│  Terminal output (terminal.ts, 248 lines)                      │
│  BSU/ESU wrapping → single stdout.write()                      │
└────────────────────────────────────────────────────────────────┘
```

## Core Rendering Files

| File | LOC | Responsibility |
|------|-----|------|
| `ink/ink.tsx` | 1,722 | Ink main loop: render scheduling, double buffering, pool management, alt-screen |
| `ink/log-update.ts` | 773 | Diff engine: per-cell diff, DECSTBM scrolling, full-reset fallback detection |
| `ink/output.ts` | 797 | Screen drawing: damage tracking, CharCache, blit optimization |
| `ink/screen.ts` | 1,486 | Screen buffer: StylePool, CharPool, HyperlinkPool |
| `ink/render-node-to-output.ts` | 1,462 | React node → Screen buffer rendering |
| `ink/terminal.ts` | 248 | Terminal capability detection, BSU/ESU wrapping, single write |
| `ink/renderer.ts` | 178 | Cursor management and alt-screen switching |
| `utils/bufferedWriter.ts` | 100 | Batched writes for streaming text |

## Mechanism 1: DEC 2026 Synchronized Output (Atomic Updates)

Source: `ink/terminal.ts#L66-118`, `ink/terminal.ts#L190-248`

**The most critical flicker-prevention mechanism**. All terminal output is wrapped with BSU/ESU (Begin/End Synchronized Update), so the terminal does not render any intermediate state before receiving ESU. Here, "DEC 2026" is the number of the DEC private mode DECSET/DECRST `?2026` (synchronized output), not a year:

```
CSI ?2026h   ← BSU: begin buffering
...clear screen, move cursor, write text...
CSI ?2026l   ← ESU: flush to the screen atomically
```

### Terminal Detection

`isSynchronizedOutputSupported()` is computed once when the module loads (capabilities do not change midway). Except for VTE, DEC 2026 detection **does not perform version checks**: VTE requires `VTE_VERSION >= 6800` (0.68+), while other terminals are matched only by name/environment variable. Note: version checking also exists in the separate `isProgressReportingAvailable()` function (for OSC 9;4 progress reporting); these are different features:

| Terminal | Detection Method |
|------|----------|
| iTerm2 | `TERM_PROGRAM === 'iTerm.app'` |
| WezTerm | `TERM_PROGRAM === 'WezTerm'` |
| Ghostty | `TERM_PROGRAM === 'ghostty'` or `TERM === 'xterm-ghostty'` |
| Kitty | `TERM` contains `kitty` or `KITTY_WINDOW_ID` exists |
| Alacritty | `TERM_PROGRAM === 'alacritty'` or `TERM` contains `alacritty` |
| foot | `TERM` starts with `foot` |
| Windows Terminal | `WT_SESSION` exists |
| VS Code | `TERM_PROGRAM === 'vscode'` |
| Warp | `TERM_PROGRAM === 'WarpTerminal'` |
| Contour | `TERM_PROGRAM === 'contour'` |
| Zed | `ZED_TERM` exists |
| VTE family (GNOME Terminal, Tilix, etc.) | `VTE_VERSION >= 6800` (VTE 0.68+) |
| **tmux** | **Skipped**: tmux does not implement DEC 2026; BSU/ESU pass through to the outer terminal, but tmux has already broken atomicity |

> **SSH scenario**: `TERM_PROGRAM` is not forwarded by SSH by default. On startup, Claude Code asynchronously detects the terminal name through an XTVERSION query (`CSI > 0 q → DCS > | name ST`), filling gaps in env-based detection (Source: `terminal.ts#L120-147`).

### Output Pipeline

`writeDiffToTerminal()` (Source: `terminal.ts#L190-248`) concatenates all diff patches into a single string, wraps it with BSU/ESU, and sends it with one `stdout.write()`:

```typescript
let buffer = useSync ? BSU : ''
for (const patch of diff) {
  switch (patch.type) {
    case 'stdout':        buffer += patch.content; break
    case 'clear':         buffer += eraseLines(patch.count); break
    case 'clearTerminal': buffer += getClearTerminalSequence(); break
    case 'cursorMove':    buffer += cursorMove(patch.x, patch.y); break
    case 'styleStr':      buffer += patch.str; break
    // ...
  }
}
if (useSync) buffer += ESU
terminal.stdout.write(buffer)  // single write
```

## Mechanism 2: Differential Rendering Engine

Source: `ink/log-update.ts` (773 lines)

Instead of a full-screen redraw, it compares the previous and current frames cell by cell and outputs only the cells that changed.

### Diff Algorithm

1. **Damage-region check**: scan only cells within the `screen.damage` rectangle (Source: `log-update.ts#L268-305`)
2. **Line-by-line comparison**: for each cell on each line, compare the charId, styleId, and hyperlinkId tuple
3. **Skip rules**:
   - Empty cells do not overwrite existing content (avoids terminal line wrapping caused by trailing spaces)
   - Wide-character placeholders (`SpacerTail`/`SpacerHead`) are skipped
   - Cells identical between previous and current frames are skipped (most cells)
4. **Cursor management**: track the virtual cursor position and use relative movement (CR+LF, CUD) rather than absolute positioning

### Full-Reset Fallback Detection

In some scenarios, diff cannot handle the update and a full-screen redraw is required (Source: `log-update.ts#L142-147`):

| Trigger Condition | Reason |
|----------|------|
| `viewport.height` shrinks | Terminal height changes cannot be handled incrementally |
| `viewport.width` changes | Text wrapping changes completely |
| Content exceeds the viewport | The scrollback buffer needs to be cleared |

The full redraw is executed through `fullResetSequence_CAUSES_FLICKER()`; the function name itself is a warning to developers.

### Post-Diff Optimization

Before the patch array produced by diff is written to the terminal, it goes through `optimize()` post-processing (Source: `ink.tsx#L621`), which merges adjacent cursor movements, removes redundant style sequences, and concatenates multiple small patches into fewer write units, further reducing output bytes.

## Mechanism 3: DECSTBM Hardware Scrolling

Source: `ink/log-update.ts#L149-185`

When a ScrollBox's `scrollTop` changes, terminal hardware scrolling instructions are used instead of rewriting line by line:

```
CSI top;bottom r    ← DECSTBM: set scroll region
CSI n S             ← SU: scroll up n lines
CSI r               ← reset scroll region
CSI H               ← return cursor to origin
```

Key optimization: the same `shiftRows()` is simulated on `prev.screen`, so the diff loop naturally detects only the new rows that have scrolled into view.

**Safety conditions** (Source: `log-update.ts#L158-164`):
- Must be in alt-screen mode
- Must have DEC 2026 support (`decstbmSafe` parameter)
- Without BSU/ESU, fall back to line-by-line rewriting: more output bytes, but no intermediate-state flicker

> **Original source comment**: *"Without atomicity the outer terminal renders the intermediate state — region scrolled, edge rows not yet painted — a visible vertical jump on every frame where scrollTop moves."*

## Mechanism 4: Double Buffering

Source: `ink/ink.tsx#L99-100, #L593-595`

```typescript
private frontFrame: Frame;   // current visible frame (diff baseline)
private backFrame: Frame;    // previous frame (reused as render target)
```

After each frame diff, they are swapped: `backFrame = frontFrame; frontFrame = frame`.

**`prevFrameContaminated` flag** (Source: `ink.tsx#L739-743`): when a selection overlay or search highlight modifies cell styleIds in place on the screen buffer, the previous frame is "contaminated" (`selActive || hlActive`). The next frame must force a full diff to avoid blitting stale cells with inverse video/highlight styles.

## Mechanism 5: Damage Tracking

Source: `ink/output.ts#L268-305, #L522-528`

The damage rectangle records which areas have changed, and the diff engine scans only dirty regions:

- **Steady-state frames** (spinner rotation, text append): narrow damage → O(changed cells) diff
- **Layout changes** (`layoutShifted`): full damage → prevents ghosting at sibling component boundaries
- **blit optimization**: parent nodes recursively blit unchanged subtrees to avoid O(children) redraws

```typescript
// Source: render-node-to-output.ts#L28-42
// layoutShifted: set when any yoga node's position/size changes
// → triggers full-damage fallback (PR #20120 fixes sibling resize boundary artifacts)
```

## Mechanism 6: Cache Pooling

Source: `ink/output.ts`, `ink/screen.ts`

### CharCache (output.ts#L178, #L198-205)

Cache from text strings to grapheme clusters (`ClusteredChar[]`, including width, styleId, and hyperlinkId). Most lines are unchanged across frames, so the hit rate is extremely high. The limit is 16k entries; the cache is cleared when it exceeds that.

### StylePool (screen.ts#L112-163)

Resident pool for ANSI style sequences:
- Interns by stylecode string key
- Caches style transitions: `transition(fromId, toId)` → pre-serialized ANSI string
- Zero allocations after warm-up: style transitions become Map lookup + string concatenation
- Bit 0 of styleId encodes whether a space is visible (background color, inverse video), used to optimize skipping space cells

### CharPool / HyperlinkPool

String → ID mapping. IDs are copied directly during blit (O(1) comparison), with no need to re-intern. Session-level lifetime.

### Pool Reset (ink.tsx#L597-603)

Reset once every 5 minutes to prevent memory growth in long sessions. The O(cells) migration cost is negligible at 5-minute intervals.

## Mechanism 7: Render Throttling (60fps Limit)

Source: `ink/ink.tsx#L205-216`

```typescript
const FRAME_INTERVAL_MS = 16  // 60fps
const deferredRender = (): void => queueMicrotask(this.onRender)
this.scheduleRender = throttle(deferredRender, FRAME_INTERVAL_MS, {
  leading: true,
  trailing: true
})
```

- **Microtask delay**: `queueMicrotask` ensures React `useLayoutEffect` calls (such as `useDeclaredCursor`) commit first, so the cursor position does not lag by one frame
- **Same event loop**: does not affect throughput
- **ScrollBox drain timer**: after hardware scrolling, backlog frames are drained quickly at intervals of `FRAME_INTERVAL_MS >> 2` (4ms) (Source: `ink.tsx#L756-758`), rather than rendering continuously at a high frame rate

## Mechanism 8: Cursor Hiding and Positioning

Source: `ink/renderer.ts#L170-175`, `ink/ink.tsx#L653-734`

- **Initial cursor state**: hidden in non-TTY mode or when screen height is 0 (Source: `renderer.ts#L173`)
- **Hidden during rendering**: in alt-screen TTY mode, the rendering process is wrapped with `HIDE_CURSOR`/`SHOW_CURSOR` inside BSU to prevent the cursor from flickering during frame updates
- `useDeclaredCursor()` parks the terminal cursor at the input box position and supports inline rendering for IME pre-edit text
- At the beginning of each frame, `CSI H` anchors the cursor at (0,0), self-healing cursor drift caused by tmux status bars, pane refreshes, and similar effects
- At the end of each frame, the cursor is parked on the prompt line to prevent the iTerm2 cursor guide from jumping frame by frame with cursor position

> **Original source comment**: *"BSU/ESU protects content atomicity but iTerm2's guide tracks cursor position independently. Parking at bottom (not 0,0) keeps the guide where the user's attention is."* (Source: `ink.tsx#L631-634`)

## Mechanism 9: Wide-Character Compensation

Source: `ink/log-update.ts#L638-750`

Terminals may judge the width of emoji/CJK characters differently from the Unicode standard:

| Issue Type | Description |
|----------|------|
| Unicode 12.0+ emoji | Newer emoji display as narrow characters in older terminals |
| Text-default emoji with VS16 (U+FE0F) | Whether they render as wide depends on the terminal implementation |

When a width mismatch is detected, CHA (Cursor Horizontal Absolute) is sent to skip the compensation columns. On correct terminals, the emoji glyph naturally overwrites the area; on older terminals, the gap is filled.

## Mechanism 10: Batched Writes (BufferedWriter)

Source: `utils/bufferedWriter.ts` (100 lines)

`BufferedWriter` is used for batched writes of error logs (`errorLogSink.ts`), asciicast recordings (`asciicast.ts`), and debug logs (`debug.ts`), avoiding disk I/O blocking caused by frequent small writes:

- Buffer limit: 100 entries
- Timed flush: 1000ms interval
- Overflow handling: `setImmediate` deferral (does not block key input)
- Ordered writes: order is preserved even on overflow

> **Note**: Streaming rendering of assistant responses does not go through `BufferedWriter`; it is completed through the React → Ink → screen buffer → diff → `writeDiffToTerminal()` rendering pipeline. After each chunk is written into the screen buffer, damage marks the changed lines, and the next frame's diff updates only newly added/changed lines.

## Mechanism 11: Alt-Screen Specialization

Source: `ink/ink.tsx#L568-651`

Alt-screen (the alternate screen buffer) has dedicated optimizations:

- At the beginning of each frame, `CSI H` anchors the cursor at (0,0), self-healing external cursor drift
- `cursor.y` is clamped to the viewport range to prevent unexpected scrolling caused by LF
- `ERASE_SCREEN` after resize is placed **inside** BSU/ESU: old content remains visible until the new frame is ready

```typescript
// Source: ink.tsx#L644-646
if (this.needsEraseBeforePaint) {
  this.needsEraseBeforePaint = false
  optimized.unshift(ERASE_THEN_HOME_PATCH)  // erase + redraw within the same BSU block
}
```

> **Comparison**: If `ERASE_SCREEN` is written synchronously in `handleResize`, the screen stays blank for ~80ms (the time spent in render()), causing visible flicker.

## Mechanism 12: Flicker-Cause Tracking (for Debugging)

Source: `ink/ink.tsx#L604-617`

When a full redraw is unavoidable, the system records flicker metadata for debugging:

```typescript
if (isDebugRepaintsEnabled() && patch.debug) {
  const chain = dom.findOwnerChainAtRow(this.rootNode, patch.debug.triggerY)
  logForDebugging(
    `[REPAINT] full reset · ${patch.reason} · row ${patch.debug.triggerY}\n` +
    `  prev: "${patch.debug.prevLine}"\n` +
    `  next: "${patch.debug.nextLine}"\n` +
    `  culprit: ${chain.join(' < ')}`
  )
}
```

Recorded fields: flicker cause (resize/offscreen/clear), trigger row number, content differences between previous and current frames, and the React component chain responsible for the redraw.

## Mechanism 13: Special Handling for Windows/WSL

Source: `ink/terminal.ts#L171-179`

Windows conhost's `SetConsoleCursorPosition` pulls the viewport back into the scrollback buffer during streaming output (the viewport yank bug). It is detected through `process.platform === 'win32'` or `WT_SESSION`.

In addition, according to the official CHANGELOG (external source: [github.com/anthropics/claude-code/CHANGELOG.md](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md), v2.1.81 entry), v2.1.81 disabled line-by-line streaming rendering on Windows (including WSL in Windows Terminal), because rendering issues caused visual anomalies.

## Design Trade-offs

| Decision | Trade-off | Rationale |
|------|------|------|
| Single `stdout.write()` instead of multiple small writes | Memory concatenation vs I/O count | A single write prevents the terminal from rendering intermediate states between writes |
| Differential rendering instead of full-screen redraw | Diff complexity vs output bytes | Steady-state frames write only a small number of cells, greatly reducing bandwidth/latency |
| Hardware scrolling + BSU/ESU dependency | Requires terminal support | When unsupported, falls back to line-by-line rewriting: more bytes but no flicker |
| Pool reset every 5 minutes | Occasional O(cells) migration vs memory growth | Without resets, long sessions (several hours) would cause unbounded Map growth |
| `queueMicrotask` delayed rendering | Tiny delay vs cursor synchronization | Ensures layout effects commit first, so IME/cursor do not lag |
| Erase inside alt-screen instead of synchronous erase | Requires BSU/ESU | Avoids a ~80ms blank screen during resize |
| `fullResetSequence_CAUSES_FLICKER` naming | — | The function name itself is a "flicker warning" for developers |

## Source File Index

| File | LOC | Key Functions/Classes |
|------|-----|-------------|
| `ink/ink.tsx` | 1,722 | `Ink` class, `onRender()`, `scheduleRender`, double-buffer swap |
| `ink/log-update.ts` | 773 | `logUpdate()`, `fullResetSequence_CAUSES_FLICKER()`, DECSTBM scrolling |
| `ink/output.ts` | 797 | `renderNodeToOutput()`, CharCache, damage tracking |
| `ink/screen.ts` | 1,486 | `Screen`, `StylePool`, `CharPool`, `HyperlinkPool` |
| `ink/render-node-to-output.ts` | 1,462 | `renderNodeToOutput()`, `layoutShifted`, blit optimization |
| `ink/terminal.ts` | 248 | `isSynchronizedOutputSupported()`, `writeDiffToTerminal()` |
| `ink/optimizer.ts` | 93 | `optimize()` patch optimization: merge cursor moves, remove redundant styles |
| `ink/renderer.ts` | 178 | `createRenderer()`, cursor visibility management |
| `utils/bufferedWriter.ts` | 100 | `BufferedWriter` class, timed flush |
