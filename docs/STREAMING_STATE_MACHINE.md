# Streaming & Tool Execution State Machine

The core loop that handles: model streaming → tool call detected mid-stream → execute tool → inject result → resume generation. This is the hardest part of building an agent product.

## Architecture Overview

The system is a **generator-based state machine** implemented as an infinite `while(true)` loop with explicit state transitions via `continue` and terminal returns. It yields incremental events to the React UI in real time.

```
AsyncGenerator<StreamEvent | Message | TombstoneMessage, Terminal>
```

**Source:** `src/query.ts` (~1,729 lines)

## State Definition

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined   // Why previous iteration continued
}
```

## State Transitions

### Loop Transitions (continue → next iteration)

| Reason | Cause | What changes |
|--------|-------|-------------|
| `next_turn` | Tool results ready, feed back to model | messages += assistantMsgs + toolResults, turnCount++ |
| `collapse_drain_retry` | Prompt-too-long recovery via context collapse | Drained context, guard prevents re-attempt |
| `reactive_compact_retry` | Prompt-too-long recovery via full summarization | Compacted messages, hasAttemptedReactiveCompact=true |
| `max_output_tokens_escalate` | Retry with higher token limit (8K → 64K) | maxOutputTokensOverride set |
| `max_output_tokens_recovery` | Truncate old turns after OTK exhaustion | maxOutputTokensRecoveryCount++ (max 3) |
| `stop_hook_blocking` | Stop hook injected blocking errors | messages += hookErrors, stopHookActive=true |
| `token_budget_continuation` | Token budget nudge → model should continue | messages += nudge message |

### Terminal Returns (exit the loop)

| Reason | Cause |
|--------|-------|
| `completed` | Model finished, no more tools needed |
| `aborted_streaming` | User abort mid-stream |
| `aborted_tools` | User abort during tool execution |
| `prompt_too_long` | Unrecoverable context overflow |
| `image_error` | Image size error |
| `model_error` | API error |
| `max_turns` | Max turns exceeded |
| `stop_hook_prevented` | Stop hook prevented continuation |
| `hook_stopped` | Hook stopped mid-tools |
| `blocking_limit` | Agent/permission limits hit |

## Main Loop Flow

```
while (true):
  ┌─ Prefetch memories and skills (async, parallel with streaming)
  │
  ├─ Stream Model Response
  │   ├─ For each streaming event → yield to UI (incremental text)
  │   ├─ Detect tool_use blocks mid-stream
  │   ├─ Add to StreamingToolExecutor (starts executing DURING stream)
  │   ├─ Yield completed tool results as they finish
  │   └─ Withhold recoverable errors (PTL, media, OTK)
  │
  ├─ NO_TOOLS_USED:
  │   ├─ Attempt error recovery (PTL → compact, OTK → escalate)
  │   ├─ Run stop hooks
  │   ├─ Check token budget
  │   └─ RETURN 'completed'
  │
  └─ TOOLS_USED:
      ├─ Collect remaining tool results
      ├─ Check for abort
      ├─ Run stop hooks
      ├─ Check maxTurns
      └─ state = { messages + toolResults, turnCount++ }
          → continue (next iteration)
```

## Two Execution Modes

### Streaming Execution (Primary)

Tools execute **during** model streaming, not after. The `StreamingToolExecutor` starts running tools as soon as their input is fully received, even while the model is still generating.

```
Model streaming: [text...] [tool_use A] [text...] [tool_use B] [text...]
                            ↓                      ↓
Tool execution:   ─────────[A executing]──────────[B executing]──────
                            ↓ result ready          ↓
UI receives:      [text] [A progress] [text] [A result] [B progress] [B result]
```

### Batch Execution (Fallback)

After all tools are detected, execute them in partitioned batches.

## StreamingToolExecutor

**Source:** `src/services/tools/StreamingToolExecutor.ts` (~531 lines)

### Tool Lifecycle

```
queued → executing → completed → yielded
```

### Concurrency Rules

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = this.tools.filter(t => t.status === 'executing')
  return (
    executing.length === 0 ||
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  )
}
```

- **Concurrency-safe tools** (Glob, Grep, Read, etc.): Execute in parallel
- **Non-safe tools** (Bash, Write, Edit): Execute alone (exclusive access)
- Queue processing stops at first blocked non-safe tool

### Error Cascade

Only Bash errors cascade to sibling tools:

```typescript
if (isErrorResult && tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')  // Cancel siblings
}
```

Read/WebFetch/Grep errors are independent — they don't cancel other tools.

### Result Ordering

Results are yielded in **registration order**, not completion order. If tool B finishes before tool A, B's results are buffered until A completes. This ensures deterministic output.

Progress messages (spinners, status updates) are yielded immediately regardless of ordering.

## Batch Tool Execution

**Source:** `src/services/tools/toolOrchestration.ts`

### Partitioning

Tools are grouped into batches:

```
[Safe, Safe, Safe] → Concurrent batch (parallel, max 10)
[Non-safe]         → Sequential batch (single tool)
[Safe, Safe]       → Concurrent batch
```

### Bounded Parallelism

```typescript
yield* all(
  toolUseMessages.map(async function* (toolUse) {
    yield* runToolUse(toolUse, ..., toolUseContext)
  }),
  getMaxToolUseConcurrency()  // Default: 10
)
```

## Error Recovery

### Prompt-Too-Long (PTL) — Two-Stage Recovery

1. **Collapse Drain**: Drain staged context-collapses (cheap, granular). Guarded — only attempts once.
2. **Reactive Compact**: Full history summarization. Guarded by `hasAttemptedReactiveCompact`.
3. **Terminal**: If both fail, surface error and return.

### Max Output Tokens (OTK) — Three-Stage Escalation

1. **Escalation**: If default 8K cap was used, retry at 64K. One-shot.
2. **Truncation**: Remove old turns from context. Up to 3 attempts.
3. **Terminal**: After 3 truncations, return `completed`.

### Streaming Fallback

When a `FallbackTriggeredError` occurs (e.g., model switch):
1. Yield tombstones for all orphaned partial messages (UI removes them)
2. Clear assistant messages, tool results, tool use blocks
3. Discard StreamingToolExecutor, create fresh one
4. Retry with fallback model

## Abort Handling

### Abort Controller Hierarchy

```
toolUseContext.abortController (main query)
  ├─ siblingAbortController (per-executor, Bash error cascade)
  │   └─ toolAbortController (per-tool)
  └─ permission rejection → query abort
```

### Abort Reasons

- `'interrupt'` — user typed new message or pressed Escape
- `'sibling_error'` — Bash tool errored, cancel siblings
- `undefined` — permission rejection

### On Abort

1. Consume remaining results from StreamingToolExecutor (generates synthetic tool_results for aborted tools)
2. Yield missing tool result blocks for any unresolved tool_use blocks
3. Create user interruption message
4. Return terminal state

## Speculative Prefetch

Three async operations start at loop entry and resolve during model streaming:

| Prefetch | When | Resolves |
|----------|------|----------|
| Memory Prefetch | Every iteration | During streaming (5-30s window) |
| Skill Discovery | If feature enabled | During streaming |
| Tool Use Summary | After tool execution | Next iteration (yielded at top of loop) |

## Interaction with React UI

### Yield Types

| Type | UI Effect |
|------|-----------|
| `stream_request_start` | Show spinner |
| `StreamEvent` (text delta) | Append text incrementally |
| `AssistantMessage` | Display complete message |
| `ProgressMessage` | Show tool execution progress |
| `TombstoneMessage` | Remove orphaned partial message |
| `ToolUseSummaryMessage` | Show summary of tool execution |

### Real-Time Flow

```
1. yield { type: 'stream_request_start' }  → UI shows spinner
2. for each streaming event:
     yield message                          → UI appends text chunk
3. StreamingToolExecutor yields results:
     yield toolResult                       → UI shows tool output
4. yield { type: 'tombstone', message }     → UI removes message
```

## Stop Hooks

After the model finishes (or tools complete), stop hooks run:

```typescript
type StopHookResult = {
  blockingErrors: Message[]      // Errors that should be fed back
  preventContinuation: boolean   // Hard stop
}
```

- `preventContinuation: true` → Terminal return
- `blockingErrors.length > 0` → Feed errors back to model, retry same turn
- `stopHookActive` guard prevents infinite hook loops

## Token Budget Continuation

When users specify token budgets (`+500k`, `use 2M tokens`):

1. Track turn token usage vs budget
2. At turn end, check if budget exhausted
3. If not: inject nudge message, continue loop
4. Stop at 90% or on diminishing returns (<500 delta for 3+ continuations)
