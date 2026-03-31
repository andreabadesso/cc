# TUI Engine - Deep Dive

Claude Code's terminal UI is built on a **heavily customized fork of [Ink](https://github.com/vadimdemedes/ink)** — React for the terminal — extended with a game-loop style rendering pipeline that runs at ~60 FPS with double-buffered screen output and diff-patched ANSI serialization.

This document covers the full rendering pipeline, from React reconciliation to terminal output.

## Architecture Overview

```
React Component Tree
       |
[React 19 Reconciler]
       |
Virtual DOM (ink-box, ink-text, ink-link, ...)
       |
[Yoga Layout Engine (WASM)]
       |
DOMElement tree with computed positions
       |
[Renderer: renderNodeToOutput()]
       |
Output operations (write, blit, shift, clear, clip)
       |
[Output.get() → Screen Buffer]
       |
Packed Int32Array cell grid
       |
[LogUpdate.render() — Frame Diff]
       |
Patch[] (cursor moves, style transitions, text writes)
       |
[writeDiffToTerminal()]
       |
Single ANSI string → stdout
```

## The Game Loop

The engine uses a classic game-loop pattern, throttled at ~60 FPS.

### Clock System (`components/ClockContext.tsx`)

A synchronized animation clock drives all frame-based animations:

- `setInterval(tick, 16ms)` — runs at `FRAME_INTERVAL_MS` (16ms ≈ 60fps)
- All animation hooks subscribe to the same clock, keeping them in sync
- Clock slows to 32ms (`2 × FRAME_INTERVAL_MS`) when the terminal is blurred (detected via DEC focus events)
- Clock only runs when at least one `keepAlive` subscriber exists (no idle CPU)

### Render Scheduling (`ink.tsx`)

Renders are not triggered on every React state change. Instead:

```
React commit (setState, etc.)
       |
reconciler.resetAfterCommit()
       |
Yoga.calculateLayout()   +   scheduleRender()
                               |
                         throttle(16ms)
                               |
                         queueMicrotask(onRender)
```

- `scheduleRender` is wrapped in `lodash.throttle(16ms, { leading: true, trailing: true })`
- The actual render is deferred via `queueMicrotask` to ensure `useLayoutEffect` runs first (for IME cursor positioning)
- This means: multiple state changes within a 16ms window collapse into one render

### Frame Timing Metrics

Every frame emits a `FrameEvent` with phase-level timing:

| Phase | What it measures |
|-------|-----------------|
| `commit` | React reconciliation duration |
| `yoga` | Yoga `calculateLayout()` duration |
| `renderer` | DOM → Screen buffer conversion |
| `diff` | Screen diff algorithm |
| `optimize` | Patch merging/deduplication |
| `write` | Serialization + `stdout.write()` |
| `yogaVisited` | Total layout node visits |
| `yogaMeasured` | Text measure function calls |
| `yogaCacheHits` | Layout cache hits |
| `yogaLive` | Total Yoga nodes alive |

## Virtual DOM (`dom.ts`)

The virtual DOM mirrors HTML DOM semantics but for terminal elements.

### Element Types

| Node Name | Has Yoga Node | Purpose |
|-----------|:---:|---------|
| `ink-root` | Yes | Root container; holds `focusManager`, `onRender`, `onComputeLayout` |
| `ink-box` | Yes | Layout container (flexbox) |
| `ink-text` | Yes | Text with Yoga measure function |
| `ink-virtual-text` | No | Text-inside-text (no layout node) |
| `ink-link` | No | Hyperlink (OSC 8) |
| `ink-progress` | No | Progress bar |
| `ink-raw-ansi` | Yes | Pre-rendered ANSI strings with fixed dimensions |

### Key Node Fields

```typescript
type DOMElement = {
  nodeName: ElementNames
  childNodes: DOMNode[]
  yogaNode?: LayoutNode          // Yoga WASM layout node
  style: Styles                  // Flexbox + visual styles
  dirty: boolean                 // Needs re-rendering
  isHidden?: boolean             // display: none

  // Scroll state (overflow: scroll)
  scrollTop?: number
  pendingScrollDelta?: number    // Accumulated, drained per-frame
  stickyScroll?: boolean         // Auto-pin to bottom

  // Lifecycle hooks
  onComputeLayout?: () => void
  onRender?: () => void

  // Debug
  debugOwnerChain?: string[]     // React component stack (CLAUDE_CODE_DEBUG_REPAINTS)
}
```

### Dirty Propagation

When a node changes, `markDirty()` walks ancestors marking each dirty. For `ink-text` nodes, this also invalidates Yoga's cached text measurement. `scheduleRenderFrom(node)` triggers a render by climbing to the root's `onRender` callback.

## React Reconciler (`reconciler.ts`)

Uses React 19's `react-reconciler` to bridge React components to the virtual DOM.

### Critical Reconciler Hooks

| Hook | When | What |
|------|------|------|
| `createInstance(type, props)` | Element creation | Creates DOM node with Yoga node; validates nesting (Box can't nest in Text) |
| `commitUpdate(node, old, new)` | Props changed | Applies style/attribute diffs, marks dirty |
| `appendChild/insertBefore` | Tree mutation | Updates both DOM and Yoga child indices |
| `removeChild` | Tree mutation | Removes node, collects rects for clearing absolute-positioned nodes |
| `resetAfterCommit(root)` | After React commit | Triggers Yoga layout + render scheduling |
| `hideInstance/unhideInstance` | Suspense | Toggles `display: none` |

### Debug: Owner Chain Tracking

When `CLAUDE_CODE_DEBUG_REPAINTS` is set, each node stores its React Fiber owner chain (e.g., `['ToolUseLoader', 'Messages', 'REPL']`). The function `findOwnerChainAtRow(root, y)` does a DFS to attribute screen flicker to the responsible component.

## Renderer (`renderer.ts`)

Converts the Yoga-laid-out DOM tree into a screen buffer.

### Pipeline

1. **Validate Yoga**: Check for `NaN`, `Infinity`, or negative dimensions
2. **Clamp height**: In alt-screen mode, enforce `height ≤ terminalRows`
3. **Create/reset Output**: Initialize the operation accumulator
4. **Render node tree**: `renderNodeToOutput(root, output, { prevScreen })` traverses the DOM, emitting operations
5. **Drain scroll**: If a `ScrollBox` has pending scroll, mark dirty for next frame
6. **Build Frame**: Return `Frame { screen, viewport, cursor }`

### Cursor Positioning

- **Alt-screen**: cursor hidden, clamped to `min(screen.height, terminalRows) - 1`
- **Main-screen**: cursor placed at `screen.height` (past content, for continued terminal writing)
- **IME support**: `useDeclaredCursor()` hook positions cursor relative to input box for composition

## Output Buffer (`output.ts`)

Accumulates render operations then applies them to the screen buffer in two passes.

### Operation Types

| Operation | Purpose |
|-----------|---------|
| `write(x, y, text, softWrap[])` | Write text at position |
| `blit(src, x, y, w, h)` | Copy cells from previous frame |
| `shift(top, bottom, delta)` | Scroll rows (for DECSTBM optimization) |
| `clear(region)` | Zero cells in a region |
| `noSelect(region)` | Mark cells non-selectable (gutters, line numbers) |
| `clip(bounds)` / `unclip()` | Push/pop clipping rectangle stack |

### Two-Pass Application

1. **Damage expansion**: Compute bounding box of all clears; collect absolute-node removals
2. **Operation processing**: Apply clip/unclip, blit, shift, write, noSelect in order

### Character Cache

The `charCache` (Map from line hash to `ClusteredChar[]`) persists across frames. Text tokenization + grapheme clustering is expensive, so unchanged lines reuse their cached cluster arrays.

```typescript
type ClusteredChar = {
  value: string       // Grapheme cluster (may be multi-codepoint)
  width: number       // Terminal display width (0, 1, or 2)
  styleId: number     // Interned style ID from StylePool
  hyperlink?: string  // OSC 8 URI
}
```

## Screen Buffer (`screen.ts`)

A packed Int32Array-based cell grid. Two 32-bit integers per cell.

### Cell Layout

```
word 0 (Int32): charId
  └─ Index into CharPool (interned character strings)

word 1 (Int32): styleId[31:17] | hyperlinkId[16:2] | width[1:0]
  ├─ styleId:     15-bit interned AnsiCode[] from StylePool
  ├─ hyperlinkId: 15-bit interned OSC 8 URI from HyperlinkPool
  └─ width:       2-bit CellWidth enum
```

### CellWidth Enum

| Value | Name | Meaning |
|:---:|-------|---------|
| 0 | Narrow | Width 1 (ASCII, most characters) |
| 1 | Wide | Width 2 (CJK, emoji) |
| 2 | SpacerTail | 2nd column of wide char (skip during render) |
| 3 | SpacerHead | Placeholder at EOL when wide char wraps |

### Interning Pools

**CharPool**: Interns character strings. ASCII fast-path via direct `Int32Array[charCode]` lookup. Unicode falls back to a `Map`. Index 0 = space, Index 1 = empty (for spacers).

**StylePool**: Interns `AnsiCode[]` arrays. Encodes visibility-on-space in bit 0 (odd IDs = styles with backgrounds/inverse/underline that are visible on space characters). Pre-computes `transition(fromId, toId)` as a cached ANSI diff string. Supports overlay operations:
- `withInverse(id)` — SGR 7 for text selection/search highlighting
- `withCurrentMatch(id)` — yellow background + bold + underline for current search match
- `withSelectionBg(id)` — solid selection color (preserves foreground)

**HyperlinkPool**: Interns OSC 8 URIs. Index 0 = no hyperlink.

### Key Operations

- `resetScreen()` — Zero all cells using `BigInt64Array.fill()` (bulk 8-byte fill)
- `blitRegion()` — Bulk copy cells + noSelect bitmap between screens
- `shiftRows()` — Row-level shift (for DECSTBM scroll optimization)
- `diffEach(prev, next, callback)` — Iterate changed cells, skip spacers and empties

## Frame Diff Engine (`log-update.ts`)

Computes the minimal set of terminal patches to transition from the previous frame to the current one.

### Patch Types

```typescript
type Patch =
  | { type: 'stdout'; content: string }     // Direct text output
  | { type: 'clear'; count: number }        // Erase N lines
  | { type: 'clearTerminal'; reason: ... }  // Full screen clear
  | { type: 'cursorMove'; x: number; y: number }  // Relative cursor move
  | { type: 'cursorTo'; col: number }       // Absolute column (CHA)
  | { type: 'cursorHide' | 'cursorShow' }
  | { type: 'hyperlink'; uri: string }      // OSC 8
  | { type: 'styleStr'; str: string }       // Pre-serialized SGR
  | { type: 'carriageReturn' }
```

### Diff Algorithm

1. **Full-reset triggers**: If viewport shrank horizontally/vertically, or if scrollback is unreachable — clear everything and repaint

2. **DECSTBM scroll optimization** (alt-screen only): If a `ScrollBox.scrollTop` changed, emit DECSTBM commands (`CSI r` to set scroll region, `CSI n S/T` to scroll). Then simulate the shift on `prevScreen` so the subsequent diff naturally discovers only the new/changed rows

3. **Row-by-row diff**: For each row in the damage region:
   - Skip rows in scrollback (unreachable with relative cursor moves)
   - Skip SpacerTail cells (terminal auto-advances for wide chars)
   - Skip empty cells (no need to overwrite)
   - For changed cells: emit `cursorMove` + `styleStr` + `stdout`
   - For removed cells (screen shrank): clear with space + reset style

4. **Growth handling**: New rows at the bottom emit directly (terminal scrolls naturally with newlines)

5. **Cursor restore**: Alt-screen skips (next frame resets with `CSI H`). Main-screen emits CRs + LFs to create rows if cursor is past content.

### Emoji Compensation

Some terminals have old wcwidth tables that don't recognize newer emoji as width-2. The diff engine compensates:
1. Write a space at `x+1`
2. `CHA` back to column `x`
3. Write the emoji

This ensures proper cell alignment even on terminals with stale Unicode width tables.

### VirtualScreen

An internal cursor tracker during patch generation:

```typescript
class VirtualScreen {
  cursor: Point
  diff: Diff[]
  txn(fn: (prev) => [patches, delta]): void  // Atomic patch + cursor update
}
```

## Terminal Abstraction (`terminal.ts`)

### Capability Detection

| Function | What it detects |
|----------|----------------|
| `isSynchronizedOutputSupported()` | DEC 2026 (atomic redraws) — iTerm, WezTerm, ghostty, kitty, etc. Skips tmux |
| `isProgressReportingAvailable()` | OSC 9;4 — ConEmu, Ghostty 1.2+, iTerm2 3.6.6+ |
| `supportsExtendedKeys()` | Kitty keyboard protocol + xterm modifyOtherKeys |
| `hasCursorUpViewportYankBug()` | Windows bug where cursor-up yanks scrollback |
| `isXtermJs()` | xterm.js-based terminal (VS Code, Cursor, Windsurf) |

### Output Serialization

`writeDiffToTerminal()` buffers all patches into a single string, wraps with BSU (Begin Synchronized Update) + ESU (End Synchronized Update) if DEC 2026 is supported, then calls `stdout.write()` once. Patch dispatch:

| Patch type | ANSI output |
|------------|-------------|
| `stdout` | Direct content |
| `clear` | `CSI K` (erase in line) |
| `clearTerminal` | Full screen clear |
| `cursorMove` | CUP/CUD relative |
| `cursorTo` | CHA (cursor horizontal absolute) |
| `styleStr` | Pre-serialized SGR codes |
| `hyperlink` | OSC 8 URI |

## Terminal I/O Primitives (`termio/`)

### CSI (Control Sequence Introducer)

Standard ANSI terminal control sequences — cursor movement, erasing, scrolling, mode setting, and SGR (Select Graphic Rendition).

### DEC Private Modes

```
DEC 25    — cursor visible
DEC 47    — alt screen
DEC 1049  — alt screen + clear
DEC 1000  — mouse normal tracking
DEC 1002  — mouse button tracking
DEC 1003  — mouse any-event tracking
DEC 1006  — mouse SGR format
DEC 1004  — focus events
DEC 2004  — bracketed paste
DEC 2026  — synchronized update
```

### OSC (Operating System Command)

Terminal title setting, clipboard access, hyperlinks (OSC 8), tab status, and progress reporting. Includes `wrapForMultiplexer()` for tmux/screen passthrough.

### SGR Parser

Parses Select Graphic Rendition parameters into structured style objects:

```typescript
type TextStyle = {
  bold, dim, italic, strikethrough, overline, inverse, hidden, blink: boolean
  underline: 'none' | 'single' | 'double' | 'curly' | 'dotted' | 'dashed'
  fg, bg, underlineColor: Color
}

type Color =
  | { type: 'named'; name: NamedColor }    // 16-color palette
  | { type: 'indexed'; index: number }     // 256-color
  | { type: 'rgb'; r, g, b: number }       // Truecolor
  | { type: 'default' }
```

### Tokenizer

Splits raw terminal input into `text` and `sequence` tokens. Tracks parser state across calls (handles multi-chunk sequences). States: `ground`, `escape`, `escapeIntermediate`, `csi`, `ss3`, `osc`, `dcs`, `apc`.

## Performance Optimizations

### Interning & Pooling
- **StylePool**: caches ANSI diffs by `(fromId, toId)` pair — zero allocations after warmup
- **CharPool**: ASCII fast-path via Int32Array lookup, Map fallback for Unicode
- **HyperlinkPool**: interns OSC 8 URIs

### Blit Optimization
When a clean subtree hasn't changed between frames, `blit()` copies entire screen regions from the previous frame's buffer instead of re-rendering. Disabled when:
- An absolute-positioned node was removed (`consumeAbsoluteRemovedFlag()`)
- The previous frame was mutated post-render (e.g., selection overlay — `prevFrameContaminated`)

### Damage Tracking
Only the `screen.damage` bounding box is diffed — blitted regions are skipped entirely.

### Scroll Draining
`ScrollBox.pendingScrollDelta` accumulates scroll changes. The renderer drains up to `SCROLL_MAX_PER_FRAME` rows per frame for smooth animations. If pending scroll remains, `scrollDrainPending` triggers an additional frame.

### Synchronized Output (DEC 2026)
All patches are wrapped in BSU/ESU markers so intermediate rendering state is never visible to the user. Eliminates flicker on supported terminals.

### Packed Cell Array
Two Int32 values per cell (8 bytes) — halves memory accesses compared to object-per-cell. `BigInt64Array` view enables bulk fill for `resetScreen()`.

## Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `ink.tsx` | ~1700 | Main Ink class: render loop, lifecycle, cursor management |
| `reconciler.ts` | ~500 | React 19 reconciler bridge to virtual DOM |
| `dom.ts` | ~485 | Virtual DOM node types, tree mutation, dirty propagation |
| `renderer.ts` | ~180 | Yoga layout → Screen buffer conversion |
| `output.ts` | ~800 | Operation accumulator (write, blit, shift, clear, clip) |
| `screen.ts` | ~1500 | Packed cell array, pools, diff algorithm |
| `log-update.ts` | ~775 | Frame diff → Patch[] generation |
| `terminal.ts` | ~250 | Capability detection, ANSI serialization |
| `frame.ts` | ~125 | Frame and FrameEvent type definitions |
| `constants.ts` | 3 | `FRAME_INTERVAL_MS = 16` |
