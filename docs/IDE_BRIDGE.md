# IDE Bridge Protocol

How VS Code and JetBrains extensions communicate with the Claude Code CLI — transport layers, authentication, session management, and message protocol.

## Architecture

The bridge is **protocol-agnostic** — the same protocol supports any client that can send HTTP/WebSocket requests. There's no VS Code-specific or JetBrains-specific code.

Two implementations:

| Mode | Transport Read | Transport Write | Auth |
|------|---------------|----------------|------|
| **V1 (Hybrid)** | WebSocket | HTTP POST | OAuth token |
| **V2 (CCR)** | Server-Sent Events (SSE) | HTTP POST | JWT (worker role) |

**Source:** `src/bridge/`, `src/cli/transports/`

## Connection Flow

### Environment-Based (Primary)

```
1. POST /v1/environments/bridge
   → Register: { machine_name, directory, branch, git_repo_url }
   ← { environment_id, environment_secret }

2. GET /v1/environments/{id}/work/poll
   → Poll for work items (long-poll)
   ← { id, type, data: { type: 'session', id }, secret }

3. POST /v1/environments/{id}/work/{workId}/acknowledge
   → Acknowledge work, get session JWT

4. Connect transport (WS or SSE)
   → Begin streaming messages

5. PUT /v1/environments/{id}/work/{workId}/heartbeat
   → Extend work lease (JWT-based, no DB hit)
```

### Environment-Less (REPL-specific, V2 only)

```
1. POST /v1/code/sessions
   → Create session (OAuth-authed)
   ← { session.id }

2. POST /v1/code/sessions/{id}/bridge
   → Register worker (each call bumps worker_epoch)
   ← { worker_jwt, expires_in, api_base_url }

3. Connect SSE + CCRClient
   → Direct ingress, no polling layer
```

## Message Protocol

### Inbound (Server → CLI)

**User/Assistant messages:**
```json
{
  "type": "user",
  "message": { "content": "..." },
  "uuid": "unique-id"
}
```

**Control requests** (must respond within ~10-14s):
```json
{
  "type": "control_request",
  "request_id": "req_123",
  "request": {
    "subtype": "initialize | set_model | interrupt | set_permission_mode | can_use_tool",
    "...fields based on subtype..."
  }
}
```

### Outbound (CLI → Server)

**Control responses:**
```json
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "req_123",
    "response": { "...fields..." }
  },
  "session_id": "cse_..."
}
```

**Stream events (V2 only):**
```json
{
  "type": "stream_event",
  "event": { "type": "content_block_delta", "delta": { "type": "text_delta", "text": "..." } }
}
```

**Results (session archival):**
```json
{
  "type": "result",
  "subtype": "success",
  "duration_ms": 1234,
  "usage": { "..." },
  "session_id": "cse_..."
}
```

### Control Request Subtypes

| Subtype | Handler | Response |
|---------|---------|----------|
| `initialize` | Return commands, models, output styles, account info, PID | Config object |
| `set_model` | Switch active model | Success |
| `set_permission_mode` | Change permission mode (with policy check) | Success or policy verdict |
| `set_max_thinking_tokens` | Adjust thinking budget | Success |
| `interrupt` | Abort current generation | Success |

## JWT Authentication

### Token Types

**session_ingress_token** (V1): Embedded in work secret. Claims include session_id, api_base_url, auth config. Prefix: `sk-ant-si-`.

**worker_jwt** (V2): From `/bridge` endpoint. Role: `worker`. Refreshed before expiry.

### Token Refresh

```
Proactive refresh: 5 minutes before expiry
V1: onRefresh callback → deliver to child process stdin
V2: Call /bridge endpoint → get fresh worker_jwt
Fallback: If expiry unparseable, retry every 30 minutes
Max consecutive failures: 3 → give up
```

## Session Management

### Spawn Modes

| Mode | Behavior | Isolation |
|------|----------|-----------|
| `single-session` | One child process per environment, tears down on session end | None |
| `worktree` | Each session in isolated git worktree | Filesystem isolation |
| `same-dir` | All sessions share same cwd | None (risk of concurrent edits) |

### Multiple IDE Instances

- `reuseEnvironmentId`: Resume with existing environment
- Concurrent sessions: Same environment polls for multiple work items
- Bridge ID: Client-generated UUID identifying bridge instance

### Session Persistence

Bridge pointer file (`~/.claude/bridge-{dir-hash}.json`):
```json
{
  "sessionId": "cse_...",
  "environmentId": "env_...",
  "source": "repl"
}
```

Used by `claude remote-control --continue` to resume after crash/kill.

## Transport Details

### V1: HybridTransport

- **Read:** WebSocket, auto-reconnect (1s → 30s backoff, 10min give-up)
- **Write:** HTTP POST via SerialBatchEventUploader
  - Max 500 events per POST
  - Max 100K events queued
  - Retry: 500ms → 8s with 1s jitter
  - Stream event batching: 100ms delay buffer to coalesce deltas
  - Grace period on close: 3s

### V2: SSETransport + CCRClient

- **Read:** SSE stream with sequence numbers
  - Same reconnect policy as V1
  - Liveness timeout: 45s after last frame
  - Server keepalives every 15s
- **Write:** CCRClient HTTP POST
  - Heartbeat: 20s intervals, extends worker lease
  - Epoch handling: 409 → close transport for poll-loop recovery
- **Sequence carry-over:** Passed between transport swaps to avoid replaying history

### File Synchronization

- **Upload:** `POST /api/{org}/upload` → `file_uuid` reference in messages
- **Download:** `GET /api/oauth/files/{uuid}/content` → written to `~/.claude/uploads/{sessionId}/`
- **Attachment:** Prepended as `@"..."` quoted paths to last text block
- **Best-effort:** Network/disk failures skip attachment, message still flows

## Permission Flow Across Bridge

```
CLI detects tool needing approval
  ↓
CLI sends control_request { subtype: 'can_use_tool', tool_name, tool_input }
  ↓
Server forwards to IDE
  ↓
IDE user approves/denies
  ↓
Server sends control_response { behavior: 'allow' | 'deny', updatedInput?, message? }
  ↓
CLI applies decision, executes or blocks tool
```

## Disconnection & Recovery

### Automatic (Transport Level)

- **Transient closes** (1006, 1011): Auto-retry for 10 minutes
- **Permanent closes** (1002, 4001, 4003): Immediate fail
- **Sleep detection:** If reconnect gap > 60s, assume sleep/wake, reset budget

### Manual (Bridge Level)

- **Poll failure (404):** Environment dead → attempt reconnect via `/reconnect` endpoint (max 3 attempts)
- **Token expiry (401):** Call `/reconnect` → re-queue session → poll picks up fresh work item
- **Perpetual mode:** Bridge pointer survives clean shutdown, next CLI invocation auto-reconnects

## REPL Bridge vs Main Bridge

| Aspect | REPL Bridge | Main Bridge |
|--------|------------|-------------|
| Process model | In-process | Spawns child CLI processes |
| Transport | Direct (Hybrid or V2) | Child gets `--sdk-url` |
| Token management | Shared scheduler | Per-session via child stdin |
| Concurrency | Single process, multiple sessions | Multiple children, one per session |
| Initial messages | Flushed via FlushGate | `--replay-user-messages` flag |
| Permissions | Callbacks wired at init | Forwarded via stdout |
| Special features | Session title derivation, perpetual mode | Work dispatch loop, child stdio monitoring |
