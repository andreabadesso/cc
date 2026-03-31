# Task System

**Sources**: `src/utils/tasks.ts`, `src/tools/Task{Create,Update,Get,List,Stop}Tool/`, `src/hooks/useTasksV2.ts`, `src/hooks/useTaskListWatcher.ts`

Tasks are the primary coordination primitive for multi-agent work. They live as individual JSON files on disk with file-level locking for concurrent safety.

## Task Data Model

```typescript
const TASK_STATUSES = ['pending', 'in_progress', 'completed'] as const
type TaskStatus = 'pending' | 'in_progress' | 'completed'

interface Task {
  id: string                          // Numeric string ("1", "2", "3")
  subject: string                     // Brief actionable title
  description: string                 // Detailed requirements
  activeForm?: string                 // Present continuous for spinner ("Running tests")
  owner?: string                      // Agent name or ID that claimed this task
  status: TaskStatus                  // pending | in_progress | completed
  blocks: string[]                    // Task IDs this task blocks (downstream)
  blockedBy: string[]                 // Task IDs that block this task (upstream)
  metadata?: Record<string, unknown>  // Arbitrary metadata
}
```

**Reserved metadata keys:**
- `_internal`: When truthy, task is hidden from `TaskListTool` output (used for system tasks)

## Storage Layout

```
~/.claude/tasks/{taskListId}/
├── .lock                   # Task list level lock (for ID generation, busy checks)
├── .highwatermark          # Maximum task ID ever assigned (prevents reuse)
├── 1.json                  # Task file (pretty-printed JSON, 2-space indent)
├── 2.json
├── 3.json
└── ...
```

Each task is an individual JSON file. No transaction log — atomicity comes from file locking.

## Task List Resolution

The `taskListId` determines which directory tasks are stored in. It follows a priority chain:

```mermaid
flowchart TD
    Start["getTaskListId()"]
    EnvVar{"CLAUDE_CODE_TASK_LIST_ID<br/>env var set?"}
    UseEnv["Use env var value"]
    InProc{"In-process teammate?<br/>(getTeammateContext())"}
    UseTeamName1["Use teammateCtx.teamName"]
    ExtTeam{"Process teammate?<br/>(getTeamName())"}
    UseTeamName2["Use CLAUDE_CODE_TEAM_NAME"]
    Leader{"Leader has team?<br/>(leaderTeamName)"}
    UseLeader["Use leaderTeamName"]
    Fallback["Use getSessionId()"]

    Start --> EnvVar
    EnvVar -->|Yes| UseEnv
    EnvVar -->|No| InProc
    InProc -->|Yes| UseTeamName1
    InProc -->|No| ExtTeam
    ExtTeam -->|Yes| UseTeamName2
    ExtTeam -->|No| Leader
    Leader -->|Yes| UseLeader
    Leader -->|No| Fallback
```

This means all teammates in a team share the same task directory (keyed by team name), enabling shared coordination.

## High-Water-Mark ID Generation

Task IDs are never reused, even after deletion or team reset. This prevents confusion when agents reference tasks by ID.

```mermaid
sequenceDiagram
    participant C as createTask()
    participant L as Lock File
    participant D as Task Directory
    participant H as .highwatermark

    C->>L: Acquire task-list-level lock
    C->>D: Scan directory for highest numeric file ID
    C->>H: Read high-water-mark value
    C->>C: nextId = Math.max(fromFiles, fromMark) + 1
    C->>D: Write {nextId}.json
    C->>L: Release lock
    C->>C: notifyTasksUpdated()
```

### Why High-Water-Mark?

When a team is reset (`resetTaskList()`), all task JSON files are deleted but the `.highwatermark` is updated first:

```typescript
async function resetTaskList(taskListId: string): Promise<void> {
  // Acquire lock
  const currentHighest = await findHighestTaskIdFromFiles(taskListId)
  if (currentHighest > 0) {
    const existingMark = await readHighWaterMark(taskListId)
    if (currentHighest > existingMark) {
      await writeHighWaterMark(taskListId, currentHighest)
    }
  }
  // Delete all .json files (keep .lock, .highwatermark)
  // Release lock, notify
}
```

After reset, the next team's tasks start from `previousMax + 1`.

## File Locking

```typescript
const LOCK_OPTIONS = {
  retries: {
    retries: 30,        // Supports ~10+ concurrent agents
    minTimeout: 5,      // 5ms initial backoff
    maxTimeout: 100,    // 100ms max backoff
  }
  // Total budget: ~2.6 seconds worst case
}
```

### Two Lock Scopes

| Scope | Lock File | Used For |
|---|---|---|
| **Task-level** | `{taskId}.json` (the task file itself) | Single task mutations (update, claim without busy check) |
| **Task-list-level** | `.lock` | Atomic multi-task operations (ID generation, busy check claims) |

```mermaid
flowchart LR
    subgraph TaskLevel["Task-Level Lock"]
        Update["updateTask(id, updates)"]
        SimpleClaim["claimTask(id, agent)"]
    end

    subgraph ListLevel["Task-List-Level Lock"]
        Create["createTask(data)"]
        BusyClaim["claimTask(id, agent, checkBusy)"]
        Reset["resetTaskList()"]
    end
```

## Task Lifecycle

```mermaid
stateDiagram-v2
    [*] --> pending: TaskCreateTool

    pending --> in_progress: claimTask() / TaskUpdateTool<br/>(auto-sets owner)
    pending --> [*]: TaskUpdateTool(status: deleted)<br/>→ file deleted + cascade cleanup

    in_progress --> completed: TaskUpdateTool(status: completed)<br/>→ executes TaskCompleted hooks
    in_progress --> pending: TaskUpdateTool(owner: undefined)<br/>→ unclaim / reassign
    in_progress --> [*]: TaskUpdateTool(status: deleted)

    completed --> [*]: Eventually cleaned up by resetTaskList()

    note right of pending
        Claiming auto-sets owner to
        getAgentName() if not explicit
    end note

    note right of completed
        Completed tasks are hidden from
        blockedBy in TaskListTool output
    end note
```

## Task Claiming

### Basic Claim (Task-Level Lock)

```mermaid
flowchart TD
    Start["claimTask(taskListId, taskId, agentId)"]
    PreCheck["Read task (no lock) — fast reject"]
    NotFound1{"Task<br/>exists?"}
    Fail1["Return: task_not_found"]
    BusyOpt{"checkAgentBusy<br/>option?"}
    BusyPath["claimTaskWithBusyCheck()<br/>(task-list-level lock)"]

    Lock["Lock task file"]
    ReadTask["Read task again (under lock)"]
    NotFound2{"Task<br/>exists?"}
    Fail2["Return: task_not_found"]
    Claimed{"Already owned by<br/>different agent?"}
    Fail3["Return: already_claimed"]
    Resolved{"Status ===<br/>completed?"}
    Fail4["Return: already_resolved"]
    Blocked{"Has unresolved<br/>blockers?"}
    Fail5["Return: blocked<br/>+ blockedByTasks"]
    Claim["Set owner = agentId<br/>Write back"]
    Success["Return: success + updated task"]

    Start --> PreCheck --> NotFound1
    NotFound1 -->|No| Fail1
    NotFound1 -->|Yes| BusyOpt
    BusyOpt -->|Yes| BusyPath
    BusyOpt -->|No| Lock
    Lock --> ReadTask --> NotFound2
    NotFound2 -->|No| Fail2
    NotFound2 -->|Yes| Claimed
    Claimed -->|Yes| Fail3
    Claimed -->|No| Resolved
    Resolved -->|Yes| Fail4
    Resolved -->|No| Blocked
    Blocked -->|Yes| Fail5
    Blocked -->|No| Claim --> Success
```

### Claim with Busy Check (Task-List-Level Lock)

When `checkAgentBusy: true`, the system acquires a **list-level lock** and checks whether the agent already owns other open tasks:

```typescript
// Inside claimTaskWithBusyCheck() — under task-list-level lock
const allTasks = await listTasks(taskListId)

const agentOpenTasks = allTasks.filter(t =>
  t.status !== 'completed' &&
  t.owner === claimantAgentId &&
  t.id !== taskId
)

if (agentOpenTasks.length > 0) {
  return { success: false, reason: 'agent_busy', busyWithTasks: agentOpenTasks.map(t => t.id) }
}
```

### Claim Result Type

```typescript
interface ClaimTaskResult {
  success: boolean
  reason?: 'task_not_found' | 'already_claimed' | 'already_resolved' | 'blocked' | 'agent_busy'
  task?: Task
  busyWithTasks?: string[]    // When reason is 'agent_busy'
  blockedByTasks?: string[]   // When reason is 'blocked'
}
```

## Dependency Management

```mermaid
graph LR
    T1["Task #1<br/>Design schema"]
    T2["Task #2<br/>Implement ORM"]
    T3["Task #3<br/>Write tests"]

    T1 -->|"blocks"| T2
    T1 -->|"blocks"| T3
    T2 -->|"blocks"| T3
```

### Establishing Dependencies

```typescript
async function blockTask(taskListId, fromTaskId, toTaskId): Promise<boolean> {
  // fromTask.blocks.push(toTaskId)  — bidirectional update
  // toTask.blockedBy.push(fromTaskId)
}
```

Called by `TaskUpdateTool` when `addBlocks` or `addBlockedBy` parameters are provided.

### Dependency Resolution

When listing tasks (`TaskListTool`), `blockedBy` is filtered to **only show unresolved blockers**:

```typescript
// Completed task IDs
const completedIds = new Set(tasks.filter(t => t.status === 'completed').map(t => t.id))

// For each task, filter blockedBy to unresolved only
task.blockedBy.filter(id => !completedIds.has(id))
```

When claiming a task, unresolved blockers prevent the claim:

```typescript
const unresolvedTaskIds = new Set(allTasks.filter(t => t.status !== 'completed').map(t => t.id))
const blockedByTasks = task.blockedBy.filter(id => unresolvedTaskIds.has(id))
if (blockedByTasks.length > 0) return { success: false, reason: 'blocked' }
```

### Cascade Cleanup on Delete

When a task is deleted, its references are removed from all other tasks:

```typescript
async function deleteTask(taskListId, taskId): Promise<boolean> {
  // Update high-water mark BEFORE deletion
  // Delete the file
  // Cascade: remove taskId from all other tasks' blocks/blockedBy arrays
}
```

## Task Tools

### TaskCreateTool

```typescript
// Input
{ subject: string, description: string, activeForm?: string, metadata?: object }

// Process
1. createTask(getTaskListId(), { ...input, status: 'pending', owner: undefined, blocks: [], blockedBy: [] })
2. executeTaskCreatedHooks(taskId, subject, description, agentName, teamName)
3. If hook returns blockingError → deleteTask() (rollback) + throw Error
4. Auto-expand task list in UI

// Output
{ task: { id: string, subject: string } }
```

### TaskUpdateTool

```typescript
// Input
{
  taskId: string,
  subject?: string, description?: string, activeForm?: string,
  status?: 'pending' | 'in_progress' | 'completed' | 'deleted',
  owner?: string,
  addBlocks?: string[], addBlockedBy?: string[],
  metadata?: object
}

// Special behaviors
- status: 'deleted' → permanently deletes task file
- status: 'completed' → executes TaskCompleted hooks BEFORE updating
  - If hook returns blockingError → return error (don't complete)
- status: 'in_progress' → auto-sets owner to getAgentName() if not explicitly set
- owner change → sends task_assignment notification to new owner via mailbox
- After completion → teammate gets reminder:
  "Call TaskList now to find your next available task"
```

### TaskGetTool

```typescript
// Input:  { taskId: string }
// Output: { task: Task | null }
// Read-only, returns full task details
```

### TaskListTool

```typescript
// Input: {} (no parameters)
// Output: { tasks: Task[] }
// Filtering:
//   - Excludes tasks with metadata._internal
//   - Filters blockedBy to only unresolved dependencies
```

### TaskStopTool

Stops **background execution tasks** (LocalShellTask, LocalAgentTask), NOT work tasks. Validates the task exists and is in `'running'` state.

## Task Hooks

**Source**: `src/utils/hooks.ts`

### TaskCreated Hook

Executed after task creation, before returning to caller:

```typescript
interface TaskCreatedHookInput {
  hook_event_name: 'TaskCreated'
  task_id: string
  task_subject: string
  task_description?: string
  teammate_name?: string
  team_name?: string
}
```

If a hook script exits with code 2, it returns a `blockingError`. The task is then rolled back (deleted) and an error thrown.

### TaskCompleted Hook

Executed before marking a task as completed:

```typescript
interface TaskCompletedHookInput {
  hook_event_name: 'TaskCompleted'
  task_id: string
  task_subject: string
  task_description?: string
  teammate_name?: string
  team_name?: string
}
```

If a hook returns a blocking error, the completion is prevented and the error returned to the LLM.

## Task Notification System

In-process subscribers are notified of mutations via a signal pattern:

```typescript
const tasksUpdated = createSignal()

// Subscribe
const unsubscribe = tasksUpdated.subscribe(() => { /* re-fetch */ })

// Emit (called after every mutation)
function notifyTasksUpdated(): void {
  tasksUpdated.emit()
}
```

Notifications fire after: `createTask`, `updateTask`, `deleteTask`, `blockTask`, `setLeaderTeamName`, `clearLeaderTeamName`, `resetTaskList`.

## UI Display

### useTasksV2 Hook

**Source**: `src/hooks/useTasksV2.ts`

Singleton task list store with file watching:

```mermaid
flowchart TD
    subgraph Store["TasksV2Store (singleton)"]
        Watcher["File watcher on<br/>~/.claude/tasks/{taskListId}/"]
        Signal["onTasksUpdated() signal<br/>(in-process mutations)"]
        Debounce["Debounce 50ms"]
        Fetch["Fetch task list from disk"]
        State["Current tasks state"]
    end

    subgraph Timers["Lifecycle Timers"]
        Poll["Fallback poll: 5000ms<br/>(when tasks incomplete)"]
        Hide["Hide timer: 5000ms<br/>(after all tasks completed)"]
    end

    Watcher --> Debounce
    Signal --> Debounce
    Debounce --> Fetch --> State
    Poll --> Fetch
    State -->|"all completed"| Hide
    Hide -->|"timeout"| State

    style Store fill:#f0f0f0
```

- **Debounce**: 50ms after file change or signal
- **Fallback poll**: 5000ms when any tasks are incomplete (in case file watcher misses events)
- **Hide delay**: 5000ms after all tasks complete → tasks disappear from UI
- **Reset**: When hidden, calls `resetTaskList()` to clean up files

### useTaskListWatcher Hook

**Source**: `src/hooks/useTaskListWatcher.ts`

Used in "tasks mode" where Claude watches a task list and auto-claims work:

```mermaid
flowchart TD
    Watch["Watch task directory (1000ms debounce)"]
    Scan["Scan for available tasks"]
    Available{"Available task<br/>found?"}
    NoOp["Continue watching"]
    Claim["claimTask(taskId, agentId)"]
    ClaimOk{"Claim<br/>success?"}
    Format["formatTaskAsPrompt(task)"]
    Submit["onSubmitTask(prompt)"]
    SubmitOk{"Submitted?"}
    Release["Release claim (clear owner)"]

    Watch --> Scan --> Available
    Available -->|No| NoOp
    Available -->|Yes| Claim
    Claim -->|No| NoOp
    Claim -->|Yes| Format --> Submit
    Submit -->|No| Release
    Submit -->|Yes| NoOp

    NoOp --> Watch
    Release --> NoOp
```

Available task criteria:
- `status: 'pending'`
- `owner: undefined`
- All `blockedBy` tasks are completed

## Task Unassignment on Teammate Exit

When a teammate exits (shutdown or killed), their tasks are released:

```typescript
async function unassignTeammateTasks(teamName, teammateId, teammateName, reason) {
  // Find tasks where:
  //   status !== 'completed' AND (owner === teammateId OR owner === teammateName)
  // For each: updateTask({ owner: undefined, status: 'pending' })
  // Return notification message with count and list
}
```

## Agent Status Tracking

```typescript
interface AgentStatus {
  agentId: string
  name: string
  agentType?: string
  status: 'idle' | 'busy'      // idle if no open tasks, busy if owns any
  currentTasks: string[]        // Task IDs agent owns
}

async function getAgentStatuses(teamName: string): Promise<AgentStatus[] | null> {
  // Read team file → get all members
  // For each member: find tasks with owner matching
  // Return idle/busy status
}
```

## Complete Task Coordination Example

```mermaid
sequenceDiagram
    participant L as Leader
    participant FS as Task Files
    participant A as Agent A
    participant B as Agent B

    L->>FS: TaskCreate #1: "Design schema" (pending)
    L->>FS: TaskCreate #2: "Implement ORM" (pending, blockedBy: [1])
    L->>FS: TaskCreate #3: "Write tests" (pending)
    L->>FS: TaskCreate #4: "Update docs" (pending)

    par Concurrent claiming
        A->>FS: claimTask(#1) → success (owner: A)
        B->>FS: claimTask(#3) → success (owner: B)
    end

    A->>A: Works on schema design
    B->>B: Works on tests

    B->>FS: TaskUpdate #3 → completed
    B->>FS: claimTask(#4) → success (owner: B)
    Note over B: Picks next available task

    A->>FS: TaskUpdate #1 → completed
    Note over FS: Task #2 now unblocked<br/>(#1 no longer in unresolved set)

    A->>FS: claimTask(#2) → success (owner: A)
    Note over A: Was blocked, now available

    B->>FS: TaskUpdate #4 → completed
    B->>FS: Send idle notification
    A->>FS: TaskUpdate #2 → completed
    A->>FS: Send idle notification

    Note over FS: All tasks complete<br/>→ useTasksV2 hides after 5s<br/>→ resetTaskList() cleans up
```
