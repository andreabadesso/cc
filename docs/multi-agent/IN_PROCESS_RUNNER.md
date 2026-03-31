# In-Process Teammate Runner

**Source**: `src/utils/swarm/inProcessRunner.ts` (~1550 lines)

This is the execution engine for in-process teammates. It runs the full agent loop — prompt assembly, LLM queries, tool execution, mailbox polling, task claiming, and idle management — all within the leader's Node.js process, isolated via `AsyncLocalStorage`.

## Execution Overview

```mermaid
flowchart TD
    Start["startInProcessTeammate()<br/>[fire-and-forget]"]
    Setup["Setup Phase"]
    Loop["Main Loop"]
    Cleanup["Cleanup Phase"]

    Start --> Setup --> Loop --> Cleanup

    subgraph Setup["Setup Phase"]
        BuildPrompt["Build system prompt:<br/>default + TEAMMATE_ADDENDUM + agent prompt"]
        ResolveModel["Resolve model (inherit → leader → default)"]
        InitContext["Initialize agent context<br/>(agentType: 'teammate')"]
    end

    subgraph Loop["Main Loop (while !abort && !shouldExit)"]
        direction TB
        PrepareTurn["Prepare turn"]
        RunAgent["Run agent (LLM query + tools)"]
        GoIdle["Transition to idle"]
        WaitWork["Wait for next work"]
        RouteResult["Route wait result"]
    end
```

## Main Loop Detail

```mermaid
flowchart TD
    LoopStart["Loop iteration start"]

    subgraph Prepare["1. Prepare Turn"]
        SetRunning["Set status: running, isIdle: false"]
        CreateAbort["Create per-turn currentWorkAbortController<br/>(Escape stops turn, not teammate)"]
        CheckCompact{"Token count ><br/>autoCompactThreshold?"}
        Compact["compactConversation()<br/>suppressFollowUpQuestions: true<br/>Reset microcompact + contentReplacement state"]
        BuildMsgs["Build prompt messages from allMessages"]
    end

    subgraph Run["2. Run Agent"]
        CallRunAgent["runAgent() async generator<br/>with createInProcessCanUseTool()"]
        Accumulate["Accumulate messages into allMessages"]
        UpdateState["Update task state:<br/>progress, inProgressToolUseIDs,<br/>messages (capped at 50)"]
    end

    subgraph Idle["3. Transition to Idle"]
        SetIdle["Set isIdle: true"]
        WasIdle{"Was already<br/>idle?"}
        SkipNotify["Skip notification<br/>(already notified)"]
        CallIdleCallbacks["Call onIdleCallbacks[]<br/>(unblocks waiters)"]
        SendIdleNotif["Send idle notification to mailbox<br/>Include last peer DM summary"]
    end

    subgraph Wait["4. Wait for Next Work"]
        WaitCall["waitForNextPromptOrShutdown()"]
        PollPending["Check pendingUserMessages<br/>(injected from UI)"]
        PollMailbox["Read mailbox (500ms interval)"]
        PollTasks["tryClaimNextTask()"]
        CheckAbort["Check abort signal"]
    end

    subgraph Route["5. Route Result"]
        ResultType{"Result type?"}
        NewMsg["new_message:<br/>Wrap in XML, add to allMessages,<br/>continue loop"]
        Shutdown["shutdown_request:<br/>Pass to LLM with approveShutdown/<br/>rejectShutdown tools"]
        Aborted["aborted:<br/>Set shouldExit = true"]
    end

    LoopStart --> Prepare
    Prepare --> SetRunning --> CreateAbort --> CheckCompact
    CheckCompact -->|Yes| Compact --> BuildMsgs
    CheckCompact -->|No| BuildMsgs

    BuildMsgs --> Run
    Run --> CallRunAgent --> Accumulate --> UpdateState

    UpdateState --> Idle
    Idle --> SetIdle --> WasIdle
    WasIdle -->|Yes| SkipNotify
    WasIdle -->|No| CallIdleCallbacks --> SendIdleNotif

    SkipNotify --> Wait
    SendIdleNotif --> Wait
    Wait --> WaitCall --> PollPending --> PollMailbox --> PollTasks --> CheckAbort

    CheckAbort --> Route
    Route --> ResultType
    ResultType -->|new_message| NewMsg --> LoopStart
    ResultType -->|shutdown_request| Shutdown --> LoopStart
    ResultType -->|aborted| Aborted
```

## Permission Bridging

**Function**: `createInProcessCanUseTool()` (lines 127-451)

In-process teammates don't have their own terminal UI. Permission prompts are routed through the leader's `ToolUseConfirm` dialog.

```mermaid
flowchart TD
    ToolUse["Teammate invokes tool"]
    BaseCheck["hasPermissionsToUseTool()"]
    Decision{"Base decision?"}
    Allow["Return: allow"]
    Deny["Return: deny"]
    Ask["Need user approval"]

    IsBash{"Is bash tool +<br/>BASH_CLASSIFIER<br/>enabled?"}
    Classifier["Await classifier auto-approval"]
    ClassOk{"Approved?"}

    LeaderUI{"Leader's ToolUseConfirm<br/>queue available?"}
    QueueInUI["Add to leader's UI queue<br/>(with workerBadge: color)"]
    UserDecides["User sees dialog with<br/>colored worker badge"]
    PersistPerm["Persist permission if allowed"]

    MailboxPath["Fallback: mailbox permission flow"]
    SendReq["Send permission_request<br/>to leader's mailbox"]
    PollResp["Poll own mailbox every 500ms<br/>for permission_response"]
    GotResp["Process response"]

    ToolUse --> BaseCheck --> Decision
    Decision -->|allow| Allow
    Decision -->|deny| Deny
    Decision -->|ask| Ask

    Ask --> IsBash
    IsBash -->|Yes| Classifier
    Classifier --> ClassOk
    ClassOk -->|Yes| Allow
    ClassOk -->|No| LeaderUI
    IsBash -->|No| LeaderUI

    LeaderUI -->|Yes| QueueInUI --> UserDecides --> PersistPerm
    LeaderUI -->|No| MailboxPath
    MailboxPath --> SendReq --> PollResp --> GotResp
```

### UI Path (Primary)

1. Get leader's `ToolUseConfirmQueue` via `getLeaderToolUseConfirmQueue()`
2. Add entry with `workerBadge: identity.color` for visual identification
3. Return a `Promise<PermissionDecision>` that resolves when the leader responds
4. On allow: persist permission updates, write back to leader's context
5. On abort: clean up queue entry

### Mailbox Path (Fallback)

1. Create `permission_request` via `createPermissionRequest()`
2. Send to leader's mailbox
3. Register callback via `registerPermissionCallback()`
4. Poll own mailbox every **500ms** for `permission_response`
5. On response: invoke callback with decision
6. Clean up interval and callback on abort or response

### Abort Handling

If the teammate's `AbortController` fires during a permission wait, the function returns `{ behavior: 'ask' }` — effectively blocking the tool execution and allowing the runner to continue to cleanup.

## Mailbox Polling (waitForNextPromptOrShutdown)

When idle, the teammate polls for work from multiple sources:

```mermaid
flowchart TD
    Start["waitForNextPromptOrShutdown()"]

    subgraph PollLoop["Poll Loop (500ms interval)"]
        CheckPending["Check pendingUserMessages<br/>(in-memory, from transcript view)"]
        HasPending{"Has pending<br/>messages?"}
        ReturnPending["Return as new_message"]

        CheckAbort{"AbortController<br/>aborted?"}
        ReturnAbort["Return: aborted"]

        ReadMail["readMailbox()"]
        ScanShutdown["Scan ALL unread for<br/>shutdown_request"]
        HasShutdown{"Found<br/>shutdown?"}
        ReturnShutdown["Mark as read<br/>Return: shutdown_request"]

        PrioritizeMsgs["Prioritize: team-lead > peers"]
        HasMsg{"Has unread<br/>message?"}
        ReturnMsg["Mark as read<br/>Return: new_message"]

        TryClaim["tryClaimNextTask()"]
        HasTask{"Claimed<br/>task?"}
        ReturnTask["Format task as prompt<br/>Return: new_message"]

        Sleep["Sleep 500ms"]
    end

    Start --> CheckPending
    CheckPending --> HasPending
    HasPending -->|Yes| ReturnPending
    HasPending -->|No| CheckAbort
    CheckAbort -->|Yes| ReturnAbort
    CheckAbort -->|No| ReadMail
    ReadMail --> ScanShutdown
    ScanShutdown --> HasShutdown
    HasShutdown -->|Yes| ReturnShutdown
    HasShutdown -->|No| PrioritizeMsgs
    PrioritizeMsgs --> HasMsg
    HasMsg -->|Yes| ReturnMsg
    HasMsg -->|No| TryClaim
    TryClaim --> HasTask
    HasTask -->|Yes| ReturnTask
    HasTask -->|No| Sleep
    Sleep --> CheckPending
```

**Priority order:**
1. In-memory pending messages (from UI injection)
2. Abort signal
3. Shutdown requests (highest priority mailbox message — scans ALL unread)
4. Team lead messages (prioritized over peer messages)
5. Other teammate messages
6. Available tasks from the task list

## Idle Detection & Notification

```mermaid
stateDiagram-v2
    [*] --> Working: Agent starts / receives new work
    Working --> Idle: runAgent() completes
    Idle --> Working: New message / task claimed / shutdown to evaluate

    state Idle {
        [*] --> CheckWasIdle
        CheckWasIdle --> SendNotification: First idle transition
        CheckWasIdle --> SkipNotification: Already was idle
        SendNotification --> WaitForWork
        SkipNotification --> WaitForWork
    }
```

- `isIdle: true` is set after each `runAgent()` completion
- `onIdleCallbacks[]` are called to unblock `waitForTeammatesToBecomeIdle()`
- Idle notification is sent via mailbox **only on the first idle transition** (checked via `wasAlreadyIdle`)
- Notification includes `getLastPeerDmSummary()` — the summary from the last `SendMessage` call

## Auto-Compaction

Before each turn, the runner checks if the conversation exceeds the token threshold:

```mermaid
sequenceDiagram
    participant R as Runner
    participant TC as Token Counter
    participant C as Compactor

    R->>TC: tokenCountWithEstimation(allMessages)
    TC-->>R: tokenCount

    alt tokenCount > autoCompactThreshold
        R->>C: compactConversation()
        Note over C: suppressFollowUpQuestions: true<br/>Isolated ToolUseContext (no cache pollution)
        C-->>R: Compacted messages

        R->>R: Rebuild allMessages from compacted output
        R->>R: Reset microcompactState (old tool IDs invalid)
        R->>R: Create fresh contentReplacementState
        R->>R: Update task.messages mirror
    end

    R->>R: Continue with turn
```

After compaction, the conversation is typically 10-20% of the prior token count. The full transcript remains on disk.

## System Prompt Assembly

```mermaid
flowchart TD
    Start["Build system prompt"]
    Mode{"systemPromptMode?"}

    Replace["Use provided systemPrompt directly"]

    Default["getSystemPrompt() — full default system prompt"]
    AddTeammate["Append TEAMMATE_SYSTEM_PROMPT_ADDENDUM<br/>(swarm-specific instructions)"]
    HasAgent{"Custom agent<br/>definition?"}
    AddAgent["Append agent.getSystemPrompt()"]
    HasOverride{"systemPrompt<br/>provided?"}
    Append["Append systemPrompt"]
    Done["Final system prompt"]

    Start --> Mode
    Mode -->|"replace"| Replace --> Done
    Mode -->|"default/append"| Default --> AddTeammate --> HasAgent
    HasAgent -->|Yes| AddAgent --> HasOverride
    HasAgent -->|No| HasOverride
    HasOverride -->|Yes| Append --> Done
    HasOverride -->|No| Done
```

The `TEAMMATE_SYSTEM_PROMPT_ADDENDUM` adds swarm-specific instructions — how to use `SendMessage`, `TaskList`, `TaskUpdate`, when to go idle, and how to handle shutdown requests.

## Abort / Cancellation

Three levels of abort exist:

```mermaid
flowchart TD
    subgraph Level1["Level 1: Lifecycle Abort (kills teammate)"]
        KillButton["Kill button / killInProcessTeammate()"]
        LifecycleAbort["abortController.abort()"]
        LoopExit["Main loop exits with status: killed"]
    end

    subgraph Level2["Level 2: Work Abort (stops current turn)"]
        EscapeKey["User presses Escape"]
        WorkAbort["currentWorkAbortController.abort()"]
        TurnStops["Current turn stops"]
        AddAbortMsg["Add ERROR_MESSAGE_USER_ABORT to transcript"]
        GoIdle["Teammate goes idle with reason: interrupted"]
        ContinueLoop["Continue main loop"]
    end

    subgraph Level3["Level 3: Permission Abort"]
        PermWait["Waiting for permission"]
        AbortDuringPerm["AbortController fires during wait"]
        ReturnAsk["Return behavior: ask (blocks tool)"]
        ToolBlocked["Tool execution blocked"]
    end

    KillButton --> LifecycleAbort --> LoopExit
    EscapeKey --> WorkAbort --> TurnStops --> AddAbortMsg --> GoIdle --> ContinueLoop
    AbortDuringPerm --> ReturnAsk --> ToolBlocked
```

## AsyncLocalStorage Context

**Source**: `src/utils/teammateContext.ts`, `src/utils/agentContext.ts`

### Why AsyncLocalStorage?

When agents are backgrounded (ctrl+b), multiple agents run concurrently in the same Node.js process. AppState is a single shared mutable object that would be overwritten by concurrent agents. AsyncLocalStorage isolates each async execution chain automatically across `await`/`Promise` chains.

### TeammateContext

```typescript
interface TeammateContext {
  agentId: string              // "researcher@my-team"
  agentName: string            // "researcher"
  teamName: string             // "my-team"
  color?: string               // "blue"
  planModeRequired: boolean
  parentSessionId: string      // Leader's session UUID
  isInProcess: true            // Discriminator
  abortController: AbortController
}
```

### SubagentContext (AgentContext)

```typescript
interface SubagentContext {
  agentId: string
  parentSessionId: string
  subagentName?: string
  isBuiltIn: boolean
  invokingRequestId: string
  invocationKind: 'spawn' | 'continue'
  invocationEmitted: boolean
}
```

### Nesting

The runner wraps execution in both contexts:

```typescript
runWithTeammateContext(teammateCtx, () =>
  runWithAgentContext(agentCtx, () =>
    runAgent(/* ... */)
  )
)
```

Any code anywhere in the call stack can call `getTeammateContext()` or `getAgentContext()` to discover identity without passing it through function parameters.

## State Machine

```mermaid
stateDiagram-v2
    [*] --> Spawned: spawnInProcessTeammate()

    Spawned --> Running: startInProcessTeammate()

    state Running {
        [*] --> Executing
        Executing --> Idle: runAgent() completes
        Idle --> Executing: New work received
    }

    Running --> Completed: Normal exit (no more work)
    Running --> Failed: Unhandled exception
    Running --> Killed: abortController.abort()

    Completed --> [*]: Evict from AppState after display timeout
    Failed --> [*]: Evict from AppState after display timeout
    Killed --> [*]: Evict from AppState after display timeout

    note right of Killed
        Set notified: true (prevent duplicate SDK event)
        Emit SDK termination bookend
    end note
```

## Cleanup

After the main loop exits (for any reason):

1. Set final status (`completed` or `failed`) unless already terminal (`killed`)
2. Set `notified: true` (prevents duplicate SDK notification event)
3. Clear messages to last one only (memory optimization)
4. Emit SDK termination bookend via `emitTaskTerminatedSdk()`
5. Evict task from AppState after `STOPPED_DISPLAY_MS` delay (graceful UI removal)
