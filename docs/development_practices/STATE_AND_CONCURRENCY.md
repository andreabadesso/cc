# State Management & Concurrency Patterns

Claude Code manages state across three challenging dimensions: a React terminal UI, a streaming LLM API, and multiple concurrent agents sharing a filesystem. The patterns chosen reflect these constraints.

---

## Central State Store

### Architecture (`src/state/store.ts`)

The state store follows a minimal Pub/Sub pattern (not Zustand, but Zustand-inspired):

```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

Key behaviors:
- `setState` uses `Object.is()` equality check to skip no-op updates
- Optional `onChange` callback fires on mutations (used for side effects like persisting to disk)
- No computed selectors or derivations built in -- React's `useSyncExternalStore` handles this

### AppState Shape (`src/state/AppStateStore.ts`)

The AppState is a large type (~453 properties) wrapped in `DeepImmutable<T>`:

```typescript
type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  toolPermissionContext: ToolPermissionContext
  mcp: { clients, commands, resources }
  tasks: { [taskId: string]: TaskState }
  plugins: { enabled, disabled }
  teamContext: TeamContext
  inbox: TeammateMessage[]
  // ... ~50 fields total
}>
```

`DeepImmutable<T>` enforces immutability at the **type level** -- you get compile errors if you try to mutate state directly. Exceptions exist for fields containing function types (like `AbortController` in `TaskState`), which can't be made deeply immutable.

### React Integration (`src/state/AppState.tsx`)

State flows into React/Ink components via `useSyncExternalStore`:

```typescript
function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState()),
  )
}
```

The selector pattern means components only re-render when their selected slice changes:

```typescript
const verbose = useAppState(s => s.verbose)  // Re-renders only when verbose changes
const model = useAppState(s => s.mainLoopModel)  // Independent subscription
```

This is important for a terminal UI where unnecessary re-renders cause visible flicker.

---

## Multi-Agent Coordination

### File-Based Architecture

Teams are coordinated through the filesystem:

```
~/.claude/teams/{team-name}/
  ├── config.json            # Team metadata + member registry
  ├── tasks/
  │   ├── 1.json ... N.json  # Individual task files
  │   └── .highwatermark     # Monotonic ID counter
  ├── inboxes/
  │   ├── team-lead.json     # Per-agent message queues
  │   └── researcher.json
  └── worktrees/             # Git worktrees (if used)
```

**Why files instead of IPC?** Files survive process crashes, work across tmux panes, and don't require a coordination server. If an agent dies, its mailbox and task assignments persist for the next agent.

### Mailbox System (`src/utils/teammateMailbox.ts`)

Each agent has an inbox file containing an ordered array of messages:

```typescript
type TeammateMessage = {
  from: string
  text: string
  timestamp: string
  read: boolean
  color?: string
  summary?: string
}
```

Operations use exclusive file locking:

```typescript
let release = await lockfile.lock(inboxPath, {
  lockfilePath: `${inboxPath}.lock`,
  retries: { retries: 10, minTimeout: 5, maxTimeout: 100 },
})
try {
  // Re-read current messages UNDER lock (don't trust pre-lock read)
  const messages = await readMailbox(recipientName, teamName)
  messages.push(newMessage)
  await writeFile(inboxPath, JSON.stringify(messages, null, 2))
} finally {
  await release()
}
```

The critical pattern: **always re-read under the lock**. Between your initial read and lock acquisition, another agent may have written to the same inbox.

### Task System with High-Water Mark IDs

Task IDs are allocated from a monotonically increasing counter:

```
.highwatermark file: stores maximum ID ever assigned
Next ID = max(current_watermark, max_existing_file_id) + 1
```

On task list reset:
1. Acquire exclusive lock
2. Read highest existing task ID
3. Update `.highwatermark` if needed
4. Delete all task JSON files
5. Next task gets watermark + 1

This guarantees **no ID reuse across reset cycles**, which matters because chat messages and logs reference task IDs.

---

## Two Agent Execution Models

### In-Process Teammates

Multiple agents run as concurrent async tasks within the same Node.js process:

| Aspect | Details |
|--------|---------|
| Isolation | `AsyncLocalStorage<TeammateContext>` |
| Communication | File-based mailbox (same as pane-based) |
| Resource sharing | API client, MCP connections, AppState |
| Termination | `AbortController` |
| Memory cap | UI shows last 50 messages per agent |

### Pane-Based Teammates (tmux/iTerm2)

Each agent runs in a separate OS process:

| Aspect | Details |
|--------|---------|
| Isolation | Separate process, separate memory |
| Communication | File-based mailbox (same as in-process) |
| Resource sharing | None -- each process has own state |
| Termination | Kill pane / shutdown signal |
| Identity | Via environment variables: `CLAUDE_CODE_AGENT_ID`, etc. |

The mailbox system is identical in both models. This means agents can switch between in-process and pane-based execution without changing their communication code.

---

## AsyncLocalStorage for Context Isolation

### Problem

When multiple agents run in the same process, each needs its own identity for analytics, logging, and state management. Global variables would cross-contaminate.

### Solution (`src/utils/agentContext.ts`, `src/utils/teammateContext.ts`)

Two separate `AsyncLocalStorage` instances:

**AgentContext** (for subagents via Agent tool):
```typescript
type SubagentContext = {
  agentId: string
  parentSessionId?: string
  subagentName?: string
  isBuiltIn?: boolean
  invokingRequestId?: string
}

runWithAgentContext(context, async () => {
  // All async operations inside see this context
  // getAgentContext() returns it, even across await boundaries
  await runAgent(...)
})
```

**TeammateContext** (for in-process team members):
```typescript
type TeammateContext = {
  agentId: string        // "researcher@my-team"
  agentName: string
  teamName: string
  planModeRequired: boolean
  abortController: AbortController
}
```

AsyncLocalStorage propagates across `await` boundaries automatically. No explicit context passing needed.

---

## Streaming State Management

### Token Flow Architecture

Tokens flow from the API to the UI through a streaming pipeline:

```
API streaming response
  → Token accumulated into ContentBlock
  → Tool use block detected → StreamingToolExecutor queue
  → Tool execution (concurrent or exclusive)
  → Results buffered in receive order
  → Yielded to UI via async generator
```

### StreamingToolExecutor (`src/services/tools/StreamingToolExecutor.ts`)

```typescript
type TrackedTool = {
  id: string
  status: 'queued' | 'executing' | 'completed' | 'yielded'
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  pendingProgress: Message[]
}
```

Concurrency rules:
- **Concurrent-safe tools** (Read, Glob, Grep) run in parallel with each other
- **Non-safe tools** (Bash, FileEdit) run exclusively -- no other tools execute alongside
- **Progress messages** are yielded immediately (don't wait for tool completion)

### Streaming Fallback Recovery

If the stream breaks mid-tool:
```typescript
try {
  for await (const event of response.stream()) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
} catch (fallbackError) {
  streamingFallbackOccured = true
  streamingToolExecutor.discard()  // Abandon pending tools
  streamingToolExecutor = new StreamingToolExecutor(...)  // Fresh executor
  // Retry query with full message history
}
```

The `discard()` method ensures queued-but-not-started tools never execute after a fallback.

---

## Signal Pattern (`src/utils/signal.ts`)

A minimal pub/sub primitive used ~15 times across the codebase:

```typescript
type Signal<Args extends unknown[] = []> = {
  subscribe: (listener: (...args: Args) => void) => () => void
  emit: (...args: Args) => void
  clear: () => void
}
```

Used for:
- Task update notifications (re-fetch task list on change)
- Mailbox notifications (re-render on new message)
- Settings change notifications (re-apply settings)
- GrowthBook refresh notifications (re-evaluate feature flags)

No EventEmitter class -- just listener sets + callbacks. This avoids the common pitfalls of EventEmitter (string-typed events, no type safety, memory leaks from forgotten listeners). The `subscribe` function returns an unsubscribe function, making cleanup explicit.

---

## Conversation State Management

### Message History

Messages exist in three places:

| Location | Purpose | Lifetime |
|----------|---------|----------|
| In-memory `allMessages` array | Active query loop | Current query |
| Disk transcript `~/.claude/sessions/{id}/transcript.json` | Persistence | Across sessions |
| Compacted summary | Token budget management | After compaction |

### Message Types (discriminated union)

```typescript
type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ToolUseSummaryMessage    // After compaction replaces tool results
  | AttachmentMessage
  | ProgressMessage
  | TombstoneMessage         // Placeholder for deleted messages
```

### In-Process Teammate Memory Cap

For in-process teammates, the UI only shows the last 50 messages:

```typescript
const TEAMMATE_MESSAGES_UI_CAP = 50
```

This was added after observing sessions with 292 concurrent agents that consumed 36GB+ RAM storing full message histories in the UI state. The full history remains on disk; the cap only applies to the in-memory UI representation.

---

## Key Design Principles

1. **Files as coordination primitive**: No IPC, no shared memory, no coordination server. Files survive crashes, work across process boundaries, and are debuggable with standard tools.

2. **Type-level immutability**: `DeepImmutable<T>` catches mutation bugs at compile time. Exceptions are explicitly documented where they're necessary.

3. **Selector-based subscriptions**: Components subscribe to state slices, not the entire store. This prevents cascade re-renders in the terminal UI.

4. **AsyncLocalStorage over globals**: Multiple agents in one process each see their own context without explicit passing. This is crucial for correct analytics attribution.

5. **Monotonic IDs**: The high-water mark pattern prevents ID reuse, which matters for any system where external references (logs, chat history) point to resources by ID.

6. **Explicit concurrency safety**: Each tool declares whether it's safe to run concurrently. The streaming executor respects this declaration. There's no implicit assumption about safety.
