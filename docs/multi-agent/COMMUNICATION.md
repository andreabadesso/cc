# Communication — Mailbox, SendMessage & Inbox Polling

**Sources**: `src/utils/teammateMailbox.ts`, `src/tools/SendMessageTool/`, `src/hooks/useInboxPoller.ts`

All inter-agent communication flows through a file-based mailbox system. Messages are JSON arrays stored at `~/.claude/teams/{team-name}/inboxes/{agent-name}.json`, protected by `proper-lockfile` for concurrent access.

## Mailbox Architecture

```mermaid
graph TB
    subgraph Leader["Leader Process"]
        LWriter["writeToMailbox()"]
        LReader["readUnreadMessages()"]
        LPoller["useInboxPoller()<br/>(1000ms interval)"]
    end

    subgraph FS["Filesystem"]
        subgraph Inboxes["~/.claude/teams/{team}/inboxes/"]
            LeaderInbox["team-lead.json"]
            LeaderLock["team-lead.json.lock"]
            T1Inbox["researcher.json"]
            T1Lock["researcher.json.lock"]
            T2Inbox["coder.json"]
            T2Lock["coder.json.lock"]
        end
    end

    subgraph T1["Teammate 1 (In-Process)"]
        T1Writer["writeToMailbox()"]
        T1Reader["waitForNextPromptOrShutdown()<br/>(500ms polling)"]
    end

    subgraph T2["Teammate 2 (Tmux Process)"]
        T2Writer["writeToMailbox()"]
        T2Reader["useInboxPoller()<br/>(1000ms interval)"]
    end

    LWriter -->|lock + append| T1Inbox
    LWriter -->|lock + append| T2Inbox
    LReader -->|lock + read| LeaderInbox

    T1Writer -->|lock + append| LeaderInbox
    T1Reader -->|lock + read| T1Inbox

    T2Writer -->|lock + append| LeaderInbox
    T2Reader -->|lock + read| T2Inbox

    LPoller --> LReader
```

## TeammateMessage Type

```typescript
interface TeammateMessage {
  from: string       // Sender's agent name (e.g., "researcher", "team-lead")
  text: string       // Message body — plain text or JSON-encoded structured message
  timestamp: string  // ISO 8601
  read: boolean      // Read status
  color?: string     // Sender's UI color
  summary?: string   // 5-10 word preview (used by leader for overview)
}
```

## File Locking

All mailbox operations use `proper-lockfile` with retry:

```typescript
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,    // 5ms initial backoff
    maxTimeout: 100,  // 100ms max backoff
  }
}
```

**Lock file**: `{inboxPath}.lock` (e.g., `researcher.json.lock`)

### Write Operation

```mermaid
sequenceDiagram
    participant W as Writer
    participant L as Lock File
    participant I as Inbox File

    W->>I: Ensure inbox dir exists (mkdir -p)
    W->>I: Create inbox file if needed (flag: 'wx')
    W->>L: lockfile.lock(inboxPath, LOCK_OPTIONS)

    Note over W,L: Lock acquired (retries if contended)

    W->>I: Read current messages (JSON parse)
    W->>I: Append new message (read: false)
    W->>I: Write back atomically (JSON stringify)

    W->>L: release()
```

### Read Operation

```mermaid
sequenceDiagram
    participant R as Reader
    participant I as Inbox File

    R->>I: readFile(inboxPath)
    Note over R: No lock needed for reads<br/>(eventual consistency OK)
    R->>R: JSON.parse → filter(msg => !msg.read)
    R-->>R: Return unread messages
```

### Mark as Read

```mermaid
sequenceDiagram
    participant R as Reader
    participant L as Lock File
    participant I as Inbox File

    R->>L: lockfile.lock(inboxPath)
    R->>I: Read messages
    R->>I: Set msg[index].read = true
    R->>I: Write back
    R->>L: release()
```

## Core Mailbox Functions

| Function | Purpose |
|---|---|
| `readMailbox(agentName, teamName)` | Read all messages (no lock) |
| `readUnreadMessages(agentName, teamName)` | Filter to unread only |
| `writeToMailbox(recipientName, message, teamName)` | Lock → append → unlock |
| `markMessageAsReadByIndex(agentName, teamName, index)` | Lock → mark → unlock |
| `markMessagesAsRead(agentName, teamName)` | Lock → mark all → unlock |
| `clearMailbox(agentName, teamName)` | Delete all messages |
| `formatTeammateMessages(messages)` | Convert to `<teammate-message>` XML for LLM |

## SendMessageTool — Routing Logic

**Source**: `src/tools/SendMessageTool/SendMessageTool.ts`

### Input Schema

```typescript
{
  to: string        // Recipient: name, "*" (broadcast), "uds:/path", "bridge:session_id"
  summary?: string  // 5-10 word preview (required for plain text)
  message: string | StructuredMessage
}
```

### Structured Message Types

```typescript
type StructuredMessage =
  | { type: 'shutdown_request'; reason?: string }
  | { type: 'shutdown_response'; request_id: string; approve: boolean; reason?: string }
  | { type: 'plan_approval_response'; request_id: string; approve: boolean; feedback?: string }
```

### Complete Routing Decision Tree

```mermaid
flowchart TD
    Entry["SendMessageTool.call(to, message)"]

    IsCross{"to starts with<br/>'bridge:' or 'uds:'?"}
    CrossSession["Route via peer protocol<br/>(Remote Control / UDS socket)"]

    IsStructured{"message is<br/>structured?"}

    IsShutdownReq{"type === 'shutdown_request'?"}
    HandleShutdownReq["Generate request_id<br/>Write to target's mailbox<br/>Return { request_id, target }"]

    IsShutdownResp{"type === 'shutdown_response'?"}
    IsApproved{"approve === true?"}
    HandleApprove["Write approval to mailbox<br/>In-process: abortController.abort()<br/>External: gracefulShutdown()<br/>Remove from team file"]
    HandleReject["Write rejection to mailbox"]

    IsPlanApproval{"type === 'plan_approval_response'?"}
    CheckLead{"Caller is<br/>team lead?"}
    HandlePlan["Write response to mailbox<br/>Include permissionMode"]
    ErrorNotLead["Error: only team lead<br/>can approve plans"]

    IsPlainText["Plain text message"]
    IsBroadcast{"to === '*'?"}
    HandleBroadcast["Write to ALL teammate inboxes<br/>(excluding sender)"]

    CheckInProc{"Recipient in<br/>agentNameRegistry?"}
    IsRunning{"Agent task<br/>status === 'running'?"}
    QueueMsg["queuePendingMessage()<br/>(in-memory delivery)"]
    ResumeAgent["resumeAgentBackground()<br/>with message"]
    WriteMailbox["writeToMailbox(recipientName, message)"]

    Entry --> IsCross
    IsCross -->|Yes| CrossSession
    IsCross -->|No| IsStructured
    IsStructured -->|Yes| IsShutdownReq
    IsStructured -->|No| IsPlainText

    IsShutdownReq -->|Yes| HandleShutdownReq
    IsShutdownReq -->|No| IsShutdownResp
    IsShutdownResp -->|Yes| IsApproved
    IsApproved -->|Yes| HandleApprove
    IsApproved -->|No| HandleReject
    IsShutdownResp -->|No| IsPlanApproval
    IsPlanApproval -->|Yes| CheckLead
    CheckLead -->|Yes| HandlePlan
    CheckLead -->|No| ErrorNotLead

    IsPlainText --> IsBroadcast
    IsBroadcast -->|Yes| HandleBroadcast
    IsBroadcast -->|No| CheckInProc
    CheckInProc -->|Yes| IsRunning
    CheckInProc -->|No| WriteMailbox
    IsRunning -->|Yes| QueueMsg
    IsRunning -->|No| ResumeAgent
```

### In-Process Message Fast Path

When sending a plain text message to a named in-process agent:

1. Look up in `appState.agentNameRegistry` → get agentId
2. Find the `LocalAgentTask` by agentId
3. **If running**: Call `queuePendingMessage()` for immediate in-memory delivery (no mailbox roundtrip)
4. **If stopped**: Call `resumeAgentBackground()` which re-starts the agent with the message injected

If the agent is not in the registry, fall back to file-based mailbox.

### Broadcast

Broadcast (`to: "*"`) writes the message to every teammate's inbox in the team file, **excluding the sender**. Returns a `BroadcastOutput` with the list of recipients.

## Structured Message Protocols

### Shutdown Negotiation

```mermaid
sequenceDiagram
    participant L as Leader
    participant MB as Mailbox
    participant T as Teammate

    L->>MB: SendMessage(shutdown_request)<br/>requestId = "shutdown-{timestamp}-{random}"
    MB->>T: Teammate reads from inbox

    Note over T: LLM evaluates shutdown request<br/>Has special tools: approveShutdown / rejectShutdown

    alt Teammate approves
        T->>MB: SendMessage(shutdown_response, approve: true)
        MB->>L: Leader reads response
        L->>L: In-process: abortController.abort()<br/>External: gracefulShutdown(0, 'other')
        L->>L: Remove member from team file
        L->>L: Unassign member's tasks (→ pending)
    else Teammate rejects
        T->>MB: SendMessage(shutdown_response, approve: false, reason: "still working")
        MB->>L: Leader reads rejection
        Note over L: Leader can retry or wait
    end
```

### Plan Approval

```mermaid
sequenceDiagram
    participant T as Teammate (plan mode)
    participant MB as Mailbox
    participant L as Leader

    Note over T: Teammate in plan mode creates a plan
    T->>MB: plan_approval_request to leader
    MB->>L: Leader reads request

    alt Leader approves
        L->>MB: SendMessage(plan_approval_response, approve: true)
        Note over MB: Includes permissionMode<br/>(leaderMode === 'plan' ? 'default' : leaderMode)
        MB->>T: Teammate reads approval
        T->>T: Exit plan mode<br/>Apply permission mode<br/>Begin implementation
    else Leader rejects
        L->>MB: SendMessage(plan_approval_response, approve: false, feedback: "...")
        MB->>T: Teammate reads rejection
        T->>T: Revise plan based on feedback
    end
```

### Permission Request/Response

```mermaid
sequenceDiagram
    participant T as Teammate
    participant MB as Mailbox
    participant L as Leader

    T->>MB: permission_request<br/>{request_id, agent_id, tool_name, input}
    MB->>L: Leader reads request

    L->>L: Show in ToolUseConfirm dialog<br/>(with worker badge + color)
    L->>MB: permission_response<br/>{request_id, subtype: 'success'|'error', updated_input}

    MB->>T: Teammate reads response
    T->>T: Apply permission decision
```

## Inbox Polling

### Leader / Pane-Based Teammates

**Source**: `src/hooks/useInboxPoller.ts`

Polls every **1000ms** via `useInterval()`.

```mermaid
flowchart TD
    Poll["useInboxPoller tick (1000ms)"]
    WhoAmI{"Agent type?"}
    InProc["In-process teammate:<br/>SKIP (uses waitForNextPromptOrShutdown)"]
    External["External teammate:<br/>Use CLAUDE_CODE_AGENT_NAME"]
    Lead["Team lead:<br/>Use lead name from teamContext"]
    ReadMail["readUnreadMessages(name, teamName)"]
    HasMsg{"Has unread<br/>messages?"}
    NoOp["No-op, continue polling"]

    Classify{"Message type?"}
    PlanApproval["Plan approval response:<br/>Apply mode + exit plan mode"]
    PermReq["Permission request:<br/>Route to leader UI"]
    PermResp["Permission response:<br/>Process as permission update"]
    ShutdownReq["Shutdown request:<br/>Queue for LLM evaluation"]
    ShutdownApproval["Shutdown approval:<br/>Remove member, unassign tasks, kill pane"]
    RegularMsg["Regular message:<br/>Submit as turn / queue if busy"]

    Poll --> WhoAmI
    WhoAmI -->|In-process| InProc
    WhoAmI -->|External| External
    WhoAmI -->|Leader| Lead
    External --> ReadMail
    Lead --> ReadMail
    ReadMail --> HasMsg
    HasMsg -->|No| NoOp
    HasMsg -->|Yes| Classify
    Classify --> PlanApproval
    Classify --> PermReq
    Classify --> PermResp
    Classify --> ShutdownReq
    Classify --> ShutdownApproval
    Classify --> RegularMsg
```

### In-Process Teammates

In-process teammates use a different polling mechanism inside their execution loop (`waitForNextPromptOrShutdown()` in `inProcessRunner.ts`). See [In-Process Runner](IN_PROCESS_RUNNER.md) for details.

### Message Delivery Strategy

- **Not loading** (idle, awaiting input): Submit immediately as a new conversation turn
- **Loading** (currently processing): Queue in `AppState.inbox.messages` with `status: 'pending'`, deliver when turn ends

## Structured Protocol Messages (Routing Table)

These message types are detected by `isPermissionRequest()`, `isShutdownRequest()`, etc. and routed by `useInboxPoller` instead of being passed to the LLM as raw text:

| Type | Direction | Purpose |
|---|---|---|
| `permission_request` | Worker → Leader | Request tool permission approval |
| `permission_response` | Leader → Worker | Grant/deny permission |
| `sandbox_permission_request` | Worker → Leader | Request sandbox network access |
| `sandbox_permission_response` | Leader → Worker | Grant/deny sandbox access |
| `shutdown_request` | Leader → Worker | Request graceful shutdown |
| `shutdown_approved` | Worker → Leader | Approve shutdown |
| `shutdown_rejected` | Worker → Leader | Reject shutdown with reason |
| `team_permission_update` | Leader → All | Broadcast permission update |
| `mode_set_request` | Leader → Worker | Set permission mode |
| `plan_approval_request` | Worker → Leader | Request plan approval |
| `plan_approval_response` | Leader → Worker | Approve/reject plan |
| `idle_notification` | Worker → Leader | Worker finished current task |

## Message Creator/Parser Functions

The mailbox provides symmetric creator and parser functions for each structured message type:

| Creator | Parser |
|---|---|
| `createIdleNotification(agentId, options)` | `isIdleNotification(text)` |
| `createShutdownRequestMessage(params)` | `isShutdownRequest(text)` |
| `createShutdownApprovedMessage(params)` | `isShutdownApproved(text)` |
| `createShutdownRejectedMessage(params)` | `isShutdownRejected(text)` |
| `createPermissionRequestMessage(params)` | `isPermissionRequest(text)` |
| `createPermissionResponseMessage(params)` | `isPermissionResponse(text)` |

Each parser does a `JSON.parse()` + `type` field check, returning the parsed object or `null`.

## Idle Notification

Sent when a teammate finishes its current task and goes idle:

```typescript
{
  type: 'idle_notification'
  from: string
  timestamp: string
  idleReason?: 'available' | 'interrupted' | 'failed'
  summary?: string             // "[to {name}] {summary}" of last SendMessage
  completedTaskId?: string     // Task that was just completed
  completedStatus?: 'resolved' | 'blocked' | 'failed'
  failureReason?: string       // If completedStatus is 'failed'
}
```

The summary includes the last peer DM summary from the teammate's conversation (extracted via `getLastPeerDmSummary()`), giving the leader context without reading the full transcript.
