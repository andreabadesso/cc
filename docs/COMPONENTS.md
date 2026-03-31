# UI Components & Hooks

The Ink layer exposes React components and hooks for building the terminal UI. Components handle layout, text, interaction, and fullscreen rendering. Hooks manage input, animation, focus, selection, and terminal integration.

## Components

### App (`components/App.tsx`)

Root component managing the entire Ink application lifecycle.

- Parses raw stdin into keyboard/mouse events via a stateful escape sequence parser
- Provides context to the entire tree: `AppContext`, `StdinContext`, `TerminalSizeContext`, `TerminalFocusProvider`, `ClockProvider`, `CursorDeclarationContext`
- Multi-click tracking: 500ms window, 1-cell position tolerance for double/triple-click
- Raw mode ref-counting: multiple `useInput` hooks can independently enable raw mode
- Detects tmux detach/attach and SSH reconnect via a 5-second stdin resume gap

### Box (`components/Box.tsx`)

Fundamental layout container — the `<div>` of the terminal.

```typescript
type Props = {
  ref?: Ref<DOMElement>
  tabIndex?: number        // ≥0 = tab cycle, -1 = programmatic only
  autoFocus?: boolean
  onClick?: (event: ClickEvent) => void
  onFocus?: (event: FocusEvent) => void
  onBlur?: (event: FocusEvent) => void
  onKeyDown?: (event: KeyboardEvent) => void
  onKeyDownCapture?: (event: KeyboardEvent) => void
  onMouseEnter?: () => void
  onMouseLeave?: () => void
  // + all Styles props (flexGrow, padding, margin, gap, etc.)
}
```

Defaults: `flexDirection: 'row'`, `flexGrow: 0`, `flexShrink: 1`, `flexWrap: 'nowrap'`, `overflow: 'visible'`.

### Text (`components/Text.tsx`)

Text rendering with ANSI styling.

```typescript
type Props = {
  color?: Color                // ANSI name, hex, or rgb
  backgroundColor?: Color
  bold?: boolean               // Mutually exclusive with dim
  dim?: boolean
  italic?: boolean
  underline?: boolean
  strikethrough?: boolean
  inverse?: boolean
  wrap?: 'wrap' | 'wrap-trim' | 'end' | 'middle' |
         'truncate' | 'truncate-end' | 'truncate-middle' | 'truncate-start'
  children?: ReactNode
}
```

### Button (`components/Button.tsx`)

Interactive button with keyboard, mouse, and focus handling.

```typescript
type Props = {
  onAction: () => void         // Fires on Enter, Space, or click
  tabIndex?: number            // Default: 0
  autoFocus?: boolean
  children: ((state: ButtonState) => ReactNode) | ReactNode
}

type ButtonState = {
  focused: boolean
  hovered: boolean
  active: boolean              // 150ms visual feedback after activation
}
```

Supports render-prop pattern: `children` can be a function receiving `ButtonState` for conditional styling.

### ScrollBox (`components/ScrollBox.tsx`)

Constrained container with imperative scroll API and viewport culling.

```typescript
type ScrollBoxHandle = {
  scrollTo(y: number): void
  scrollBy(dy: number): void
  scrollToElement(el: DOMElement, offset?: number): void
  scrollToBottom(): void
  getScrollTop(): number
  getScrollHeight(): number        // Cached (up to 16ms stale)
  getFreshScrollHeight(): number   // Direct Yoga call
  getViewportHeight(): number
  isSticky(): boolean
  subscribe(listener: () => void): () => void
  setClampBounds(min?, max?): void // Virtual scroll bounds
}
```

Key behaviors:
- Scroll mutations bypass React entirely: mutate `scrollTop` on the DOM node, mark dirty, defer render via microtask
- `scrollBy()` accumulates in `pendingScrollDelta`; the renderer drains at a capped rate per frame
- Viewport culling: only children intersecting `[scrollTop, scrollTop+height]` are rendered
- `stickyScroll`: when enabled, auto-follows new content at the bottom

### AlternateScreen (`components/AlternateScreen.tsx`)

Fullscreen mode with optional mouse tracking.

```typescript
type Props = {
  mouseTracking?: boolean   // Default: true
  children: ReactNode
}
```

On mount:
1. Enters alt screen (DEC 1049)
2. Clears screen (`ESC[2J` + `ESC[H`)
3. Enables mouse tracking if requested (SGR mode 1000+1002+1003+1006)
4. Constrains height to terminal rows
5. Notifies Ink instance

Uses `useInsertionEffect` (not `useLayoutEffect`) so `ENTER_ALT_SCREEN` reaches the terminal before the first frame renders to the main screen.

### Link (`components/Link.tsx`)

Hyperlink with terminal hyperlink protocol detection.

```typescript
type Props = {
  url: string
  children?: ReactNode    // Default: display URL
  fallback?: ReactNode    // Shown if hyperlinks unsupported
}
```

If `supportsHyperlinks()`: wraps content in `<ink-link href={url}>`. Otherwise: renders fallback or URL as plain text.

### RawAnsi (`components/RawAnsi.tsx`)

Bypass React/Yoga for pre-rendered ANSI content.

```typescript
type Props = {
  lines: string[]   // Pre-wrapped, ANSI-escaped lines
  width: number     // Fixed leaf width for Yoga
}
```

Single Yoga leaf with O(1) measure function. Avoids the `<Text> → React tree → Yoga → squash → re-serialize` roundtrip. Ideal for syntax-highlighted diffs and external renderer output.

### NoSelect (`components/NoSelect.tsx`)

Marks content as non-selectable in alt-screen text selection. Used for line numbers, diff sigils, list bullets — so click-drag yields clean copyable text.

### Newline / Spacer

- `Newline({ count = 1 })` — inserts newline characters
- `Spacer` — equivalent to `<Box flexGrow={1} />`

## Hooks

### useInput(handler, options)

Capture keyboard input with automatic raw mode management.

```typescript
type Handler = (input: string, key: Key, event: InputEvent) => void
type Options = { isActive?: boolean }
```

- `useLayoutEffect`: enables raw mode synchronously during React commit phase (before render returns)
- `isActive=false`: deactivates capture without removing the listener slot
- Blocks Ctrl+C if `exitOnCtrlC=false`

### useAnimationFrame(intervalMs | null)

Synchronized animations that pause when offscreen.

```typescript
type Return = [ref: (el: DOMElement | null) => void, time: number]
```

- Returns a ref callback + elapsed time in milliseconds
- Clock only ticks while the element is in viewport (`keepAlive: true`)
- All instances share the global animation clock (stay in sync)
- Pass `null` to pause
- Automatically slows when terminal is blurred

### useInterval(callback, intervalMs | null)

Timer backed by the shared animation clock (not `setInterval`).

- All timers consolidate into one wake-up cycle
- Pass `null` to pause; resume by passing a number again
- Clock only runs when at least one `keepAlive` subscriber exists

### useTerminalViewport()

Detect if a component is within the terminal viewport.

```typescript
type Return = [ref: (el: DOMElement | null) => void, entry: { isVisible: boolean }]
```

- Updates during layout phase (`useLayoutEffect`) — callers read fresh values
- Visibility changes do NOT trigger re-renders (avoids infinite loops)
- Walks DOM ancestor chain, accounting for scroll containers and `scrollTop`

### useTerminalFocus()

Returns `boolean` — `true` if the terminal window has focus (or if focus state is unknown). Uses DEC 1004 focus reporting, automatically handled by Ink.

### useSelection()

Text selection operations (alt-screen only). Returns no-ops outside fullscreen mode.

```typescript
type Return = {
  copySelection(): string
  copySelectionNoClear(): string
  clearSelection(): void
  hasSelection(): boolean
  subscribe(cb: () => void): () => void
  moveFocus(move: FocusMove): void        // Keyboard selection
  captureScrolledRows(first, last, side): void  // Rows scrolling out of view
  setSelectionBgColor(color: string): void
}
```

### useHasSelection()

Reactive boolean — re-renders when selection is created/cleared. Uses `useSyncExternalStore`.

### useDeclaredCursor(line, column, active)

Position terminal cursor for IME composition and accessibility. Returns a ref callback. When `active=true`, declares cursor position relative to the parent Box.

### useSearchHighlight()

Query-based text search highlighting on the screen buffer.

```typescript
type Return = {
  setQuery(query: string): void
  scanElement(el: DOMElement): MatchPosition[]
  setPositions(state: { positions, rowOffset, currentIdx } | null): void
}
```

Matches rendered text (not source). All matches get SGR 7 (inverse); current match gets yellow background + bold + underline.

### useTerminalTitle(title: string | null)

Set terminal tab/window title. Strips ANSI sequences. Uses `process.title` on Windows, OSC 0 on Unix.

### useTabStatus(kind: 'idle' | 'busy' | 'waiting' | null)

Set tab-status indicator via OSC 21337. Presets: green dot (idle), orange dot (busy), blue dot (waiting). Safe on unsupported terminals.

### useApp()

Returns `{ exit: (error?: Error) => void }` — app lifecycle control.

### useStdin()

Returns stdin stream, raw mode control, event emitter, and terminal querier.

## Event System

### Event Types

| Event | Properties | Behavior |
|-------|-----------|----------|
| `ClickEvent` | `col`, `row`, `localCol`, `localRow`, `cellIsBlank` | Left-click, bubbles up DOM tree. `localCol/Row` recomputed per handler |
| `KeyboardEvent` | `key`, `ctrl`, `shift`, `meta`, `superKey`, `fn` | Capture + bubble phases. Printable: `key.length === 1` |
| `FocusEvent` | `relatedTarget` | Fires on focus/blur, bubbles. Non-cancelable |
| `InputEvent` | `keypress`, `key`, `input` | Legacy: fired to EventEmitter before DOM dispatch |

All events support `stopImmediatePropagation()`.

### Focus Management (`focus.ts`)

DOM-like focus manager:
- `activeElement`: currently focused node
- `focusStack`: MRU stack (max 32, deduplicated) for tab cycling
- `focus(node)`: dispatches blur + focus events, maintains stack
- `handleNodeRemoved(node, root)`: clears node and descendants from stack, restores previous focus
- `handleClickFocus(node)`: focuses on click if `tabIndex` is set

### Text Selection (`selection.ts`)

Mutable selection state for alt-screen mode:
- Line-based selection (not rectangular): anchor + focus, normalized on render
- Supports char/word/line modes (single/double/triple click)
- `scrolledOffAbove/Below`: captures text that scrolled out of the viewport during selection
- `captureScrolledRows()`: preserves selected text as it leaves the visible area

### Input Parsing (`parse-keypress.ts`)

Stateful parser that handles multi-chunk sequences:

1. **Tokenizer**: splits input into `text` + `sequence` tokens
2. **Terminal response detection**: DECRPM, DA1, DA2, Kitty keyboard, cursor position, XTVERSION
3. **Keypress pattern matching**: CSI u (Kitty), xterm modifyOtherKeys, application keypad, function keys, mouse events (SGR)
4. **Timeout handling**: 50ms for normal escape sequences, 500ms for bracketed paste

```typescript
type ParsedKey = {
  kind: 'key'
  name: string           // 'return', 'escape', 'down', 'a', etc.
  sequence: string       // Raw escape sequence
  ctrl, shift, meta, option, super, fn: boolean
  isPasted?: boolean     // Inside bracketed paste
}
```
