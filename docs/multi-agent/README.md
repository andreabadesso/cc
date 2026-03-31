# Multi-Agent Orchestration System

Claude Code implements a production-grade multi-agent coordination framework. Agents can be spawned as lightweight in-process coroutines or as full separate processes in tmux/iTerm2 panes. They coordinate through a file-based mailbox system and a shared task list with file-level locking for concurrent safety.

## Architecture Overview

```mermaid
graph TB
    User["User Input"]
    Leader["Team Leader Agent"]

    subgraph SpawnLayer["Spawn Layer"]
        AgentTool["AgentTool<br/>(entry point for all spawning)"]
        SpawnDecision{"Has name +<br/>team_name?"}
        SubagentPath["Subagent Path<br/>(sync/async query)"]
        TeammatePath["Teammate Path<br/>(persistent agent)"]
    end

    subgraph Backends["Execution Backends"]
        InProc["In-Process Backend<br/>(AsyncLocalStorage isolation)"]
        TmuxB["Tmux Backend<br/>(new Bun process in pane)"]
        ITermB["iTerm2 Backend<br/>(new Bun process in pane)"]
    end

    subgraph SharedInfra["Shared Coordination Infrastructure"]
        Mailbox["File-Based Mailbox<br/>(~/.claude/teams/{name}/inboxes/)"]
        TaskSystem["Task System<br/>(~/.claude/tasks/{id}/)"]
        TeamConfig["Team Config<br/>(~/.claude/teams/{name}/config.json)"]
        Permissions["Permission Sync<br/>(file + mailbox-based)"]
    end

    subgraph Workers["Active Teammates"]
        W1["Teammate A<br/>(researcher)"]
        W2["Teammate B<br/>(coder)"]
        W3["Teammate C<br/>(tester)"]
    end

    User --> Leader
    Leader --> AgentTool
    AgentTool --> SpawnDecision
    SpawnDecision -->|No| SubagentPath
    SpawnDecision -->|Yes| TeammatePath
    TeammatePath --> InProc
    TeammatePath --> TmuxB
    TeammatePath --> ITermB
    InProc --> W1
    TmuxB --> W2
    ITermB --> W3
    Workers <-->|"read/write messages"| Mailbox
    Workers <-->|"claim/update tasks"| TaskSystem
    Leader <-->|"read/write messages"| Mailbox
    Leader <-->|"create/assign tasks"| TaskSystem
    Leader -->|"register/remove members"| TeamConfig
    Workers <-->|"request/grant permissions"| Permissions
```

## Subsystem Documentation

| Document | Covers |
|----------|--------|
| [AgentTool](multi-agent/AGENT_TOOL.md) | The unified entry point for all agent spawning — schema, execution paths, fork experiment, result formatting, error handling, feature flags |
| [Spawning & Backends](multi-agent/SPAWNING.md) | How teammates are actually created — backend detection, in-process vs pane-based spawning, CLI flag/env propagation, process lifecycle |
| [Teams & Coordinator](multi-agent/TEAMS.md) | Team creation/deletion, team file structure, AppState integration, coordinator mode system prompt, color assignment, session cleanup |
| [Communication](multi-agent/COMMUNICATION.md) | File-based mailbox, SendMessage routing, structured message protocols, inbox polling, inter-agent message delivery |
| [Task System](multi-agent/TASK_SYSTEM.md) | Task data model, ID generation, file locking, claiming, dependencies, hooks, UI display, task list resolution |
| [In-Process Runner](multi-agent/IN_PROCESS_RUNNER.md) | The teammate execution loop — permission bridging, mailbox polling, idle detection, auto-compaction, system prompt assembly, abort propagation |
| [Worktree Isolation](multi-agent/WORKTREES.md) | Git worktree creation/cleanup, sparse checkout, symlinks, post-creation setup, stale worktree cleanup |

## Complete Team Lifecycle

```mermaid
sequenceDiagram
    participant U as User
    participant L as Leader
    participant TC as TeamCreateTool
    participant AT as AgentTool
    participant FS as Filesystem
    participant T1 as Teammate 1
    participant T2 as Teammate 2
    participant TD as TeamDeleteTool

    U->>L: "Build a feature with tests"
    L->>TC: Create team "feature-dev"
    TC->>FS: Write config.json (leadAgentId: team-lead@feature-dev)
    TC->>FS: Create task dir + inbox dir
    TC-->>L: Team ready

    L->>FS: TaskCreate #1: "Implement auth module"
    L->>FS: TaskCreate #2: "Write auth tests" (blockedBy: [1])

    L->>AT: Spawn "coder" (name + team_name)
    AT->>FS: Detect backend → in-process/tmux
    AT->>T1: Start teammate
    AT->>FS: Update config.json (add member)

    L->>AT: Spawn "tester" (name + team_name)
    AT->>T2: Start teammate
    AT->>FS: Update config.json (add member)

    T1->>FS: TaskList → sees #1 available
    T1->>FS: claimTask(#1) with file lock
    T1->>T1: Implement auth module
    T1->>FS: TaskUpdate #1 → completed

    T2->>FS: TaskList → #2 now unblocked
    T2->>FS: claimTask(#2) with file lock
    T2->>T2: Write tests

    T1->>FS: Idle notification → mailbox
    T2->>FS: TaskUpdate #2 → completed
    T2->>FS: Idle notification → mailbox

    L->>FS: SendMessage(shutdown_request) to all
    T1->>FS: shutdown_response(approved)
    T2->>FS: shutdown_response(approved)

    L->>TD: Delete team
    TD->>FS: Remove team dir, task dir, worktrees
    TD-->>L: Cleanup complete
```

## Key Design Decisions

1. **File-based coordination over IPC**: All inter-agent communication uses JSON files with `proper-lockfile` for concurrent safety. This works across both in-process and separate-process backends without protocol differences.

2. **AsyncLocalStorage for in-process isolation**: Multiple teammates run in the same Node.js event loop but with fully isolated contexts via `AsyncLocalStorage`. No shared mutable state leaks between agents.

3. **High-water-mark task IDs**: Task IDs are never reused, even after deletion or team reset. A `.highwatermark` file persists the maximum ID ever assigned.

4. **Prompt cache optimization (fork path)**: When forking subagents, the system constructs byte-identical message prefixes across all children to maximize API prompt cache hits. Only the per-child directive differs.

5. **Graceful shutdown negotiation**: Teammates aren't force-killed. The leader sends a `shutdown_request` message, and the teammate's LLM decides whether to approve or reject based on its current state.

6. **Permission bridging**: In-process teammates route permission prompts through the leader's UI dialog (with colored worker badges). If the UI is unavailable, they fall back to mailbox-based permission requests.
