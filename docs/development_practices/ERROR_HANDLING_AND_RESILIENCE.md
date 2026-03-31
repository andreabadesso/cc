# Error Handling & Resilience Patterns

Claude Code demonstrates a carefully designed approach to error handling where every subsystem has an explicit failure mode: either fail-open (continue with degraded functionality) or fail-closed (block and notify). The choice depends on whether the subsystem is essential to the user's task.

---

## Retry Strategy

### API Call Retries (`src/services/api/withRetry.ts`)

The core retry engine handles transient API failures:

```
Attempts:        10 (configurable)
Base delay:      500ms
Backoff:         Exponential, 2^(n-1) × base
Jitter:          ±25% (prevents thundering herd)
Retry-After:     Respected when present in response headers
```

### 529 Overload Handling

The system tracks consecutive 529 (overloaded) responses:

```
3 consecutive 529s → trigger model fallback (Opus → Sonnet)
```

This is a graceful degradation: rather than failing entirely, the system switches to a less loaded model. The user gets a response, even if from a different model than requested.

### Fast Mode Circuit Breaker

Fast mode (lower latency variant) has its own circuit breaker:
- Short retries preserve the response cache
- Long retries trigger a 30-minute cooldown
- After cooldown, fast mode is automatically re-enabled

---

## Context Overflow Recovery

When the conversation approaches or exceeds the context window, three recovery strategies are attempted in sequence (`src/query.ts`):

```
1. Context Collapse → drain pending tool results, compact conversation
   ↓ (if still over limit)
2. Reactive Compact → aggressive compaction with image stripping
   ↓ (if still over limit)
3. Max Output Escalation → increase max_output_tokens from 8K → 64K
   ↓ (if still fails)
4. Surface error to user
```

Each strategy is attempted exactly once per conversation. The `hasAttemptedReactiveCompact` and `maxOutputTokensRecoveryCount` state fields prevent infinite recovery loops.

---

## Auto-Compact Circuit Breaker

The auto-compact system (`src/services/compact/autoCompact.ts`) has its own circuit breaker:

```
3 consecutive compaction failures → stop attempting compaction
```

This was added after observing sessions with 3,272 consecutive compaction failures, each wasting an API call. The circuit breaker prevents runaway API costs when compaction is fundamentally broken (e.g., the conversation has a single enormous message that can't be meaningfully compressed).

---

## Error Classification

### API Error Types (`src/services/api/errors.ts`)

Errors are classified into actionable categories:

| Category | Response |
|----------|----------|
| Transient (429, 529, 5xx) | Retry with backoff |
| Prompt too long | Context recovery sequence |
| Authentication failure | Block query, prompt re-auth |
| Media processing error | Strip media, retry |
| Model not available | Fall back to alternative |
| Rate limit | Respect Retry-After header |

### Error Watermarking

The QueryEngine uses error watermarking to avoid surfacing stale errors:

```
1. Before each API call, set a watermark
2. If an error occurs, compare against watermark
3. Only surface errors newer than the watermark
```

This prevents confusing UX where an error from a previous attempt gets displayed after a successful retry.

---

## Fails-Open vs Fails-Closed Philosophy

### Fails Open (non-essential subsystems)

| Subsystem | Failure Behavior |
|-----------|-----------------|
| Upstream proxy (CCR) | Log warning, continue without proxy |
| Telemetry/analytics | Best-effort, non-blocking |
| MCP server connections | Individual failures don't block other servers |
| API preconnect | Fire-and-forget; if it fails, first real request does the handshake |
| GrowthBook (feature flags) | Return cached/default values |
| Remote managed settings | Return empty settings for ineligible users |
| Memory prefetch | Abort and continue without memory context |
| Release notes fetch | Return cached version or skip |

The common pattern: wrap in `void promise.catch(() => {})` and ensure a fallback path exists.

### Fails Closed (essential subsystems)

| Subsystem | Failure Behavior |
|-----------|-----------------|
| Configuration parsing | Show dialog (interactive) or exit (non-interactive) |
| Authentication | Block query until re-authenticated |
| Permission checks | Default to "ask" (never silently allow) |
| Prompt too long after all recovery | Surface error to user |
| Sandbox configuration | Block operations that exceed policy |

The principle: if failure could lead to data loss, security bypass, or incorrect behavior, stop and notify.

---

## File Operation Resilience

### Multi-Agent File Locking (`src/utils/teammateMailbox.ts`)

File-based coordination uses `proper-lockfile` with exponential backoff:

```typescript
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,    // ms
    maxTimeout: 100,  // ms
  },
}
```

The pattern for read-modify-write operations:

```
1. Acquire exclusive lock on file.lock
2. Re-read file contents under lock (don't trust pre-lock read)
3. Modify in memory
4. Write atomically
5. Release lock in finally block
```

Step 2 is critical: the file may have changed between the initial read and lock acquisition. Always re-read under the lock.

### High-Water Mark for Task IDs

Task IDs use a high-water mark file (`.highwatermark`) that is never decremented:

```
Reset cycle:  Task IDs 1..15 created
              Reset: save watermark=15, delete all task files
              Next task: ID 16 (not 1)
```

This prevents ID reuse, which would cause stale references in chat history or logs to point to unrelated tasks.

---

## Error Logging Architecture (`src/utils/log.ts`)

### In-Memory Ring Buffer

Errors are logged to a 100-entry ring buffer:
- Oldest entries are evicted when the buffer is full
- Buffer can be dumped to disk on crash or user request
- No filesystem I/O on the hot path

### Pluggable Sink Pattern

The logging system uses a sink pattern where different backends can consume log entries:
- Console (development)
- Disk (crash reports)
- Telemetry (analytics)
- First-party event logger (Anthropic backend)

Each sink independently decides which log levels to accept.

---

## MCP Connection Resilience (`src/services/mcp/client.ts`)

### Terminal Error Detection

MCP connections distinguish between recoverable and terminal errors:
- **Recoverable**: Network timeout, temporary disconnection → auto-reconnect with backoff
- **Terminal**: Auth failure, server gone → mark as failed, surface to user

### SSE Reconnection

Server-Sent Events connections have built-in reconnection:
- Reconnect on disconnect with exponential backoff
- Resume from last event ID when possible
- Fall back to full re-initialization if resume fails

### Graceful Server Shutdown

When shutting down, MCP servers receive a clean shutdown signal:
- Pending requests are drained
- Resources are released
- Connection state is persisted for next session

---

## Auto Mode Circuit Breaker

The auto mode (ML-based permission classification) has its own circuit breaker gated by GrowthBook:

```typescript
kickOutOfAutoIfNeeded()
// Re-checks feature gate at state-set time
// Catches mid-await mode shifts (e.g., if GrowthBook refreshes during a long tool execution)
```

This means Anthropic can remotely disable auto mode for all users (or a subset) if the classifier is misbehaving, without requiring a client update.

---

## Key Design Principles

1. **Explicit failure modes**: Every subsystem documents whether it fails open or closed. There are no "undefined" failure paths.

2. **Circuit breakers prevent cascading waste**: When something fails repeatedly (compaction, fast mode, auto mode), the system stops trying rather than burning resources.

3. **Recovery is sequential, not parallel**: Context overflow recovery tries one strategy at a time, with state tracking to prevent loops.

4. **Re-read under lock**: File operations never trust data read before the lock was acquired. This eliminates TOCTOU races in multi-agent scenarios.

5. **Error watermarking**: Users only see errors from the most recent attempt, preventing confusion from stale error messages.

6. **Remote kill switches**: Critical subsystems (auto mode, analytics sinks, feature rollouts) can be disabled remotely via GrowthBook without deploying code.
