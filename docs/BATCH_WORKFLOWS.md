# Batch & Parallel Agent Workflows

This document covers the multi-agent parallel execution patterns in Claude Code: the `/batch` skill for large-scale parallel changes, the `/simplify` multi-agent review pattern, the coordinator mode orchestration layer, and the fork subagent system for cache-optimized parallel spawning.

---

## Architecture Overview

Claude Code supports three distinct parallel agent patterns, each suited to different use cases:

| Pattern | Entry Point | Workers | Isolation | Use Case |
|---------|-------------|---------|-----------|----------|
| `/batch` | `src/skills/bundled/batch.ts` | 5-30 background agents | git worktrees | Large-scale parallel refactors, migrations |
| `/simplify` | `src/skills/bundled/simplify.ts` | 3 parallel agents | in-process | Multi-angle code review of recent changes |
| Coordinator mode | `src/coordinator/coordinatorMode.ts` | N background workers | configurable | General-purpose task orchestration |
| Fork subagent | `src/tools/AgentTool/forkSubagent.ts` | N forks | cache-shared context | Research fanout, implementation delegation |

---

## 1. The `/batch` Skill: 3-Phase Architecture

**Source**: `src/skills/bundled/batch.ts`

`/batch` orchestrates a large parallelizable change across a codebase in three distinct phases with a human approval gate between planning and execution.

```
  Phase 1: Research & Plan          Phase 2: Spawn Workers         Phase 3: Track Progress
 ┌─────────────────────────┐     ┌──────────────────────────┐    ┌─────────────────────────┐
 │ EnterPlanMode            │     │ Spawn N background agents│    │ Render status table      │
 │  ├─ Launch research      │     │ (all in one message)     │    │  ├─ Parse PR: <url>      │
 │  │  subagents (fg)       │     │  ├─ isolation: "worktree"│    │  │  from each result      │
 │  ├─ Decompose into       │     │  ├─ run_in_background    │    │  ├─ Update: done/failed   │
 │  │  5-30 units           │     │  └─ Self-contained prompt│    │  └─ Final summary table   │
 │  ├─ Find e2e test recipe │     └──────────────────────────┘    │     "22/24 units landed"  │
 │  ├─ Write plan file      │                                     └─────────────────────────┘
 │  └─ ExitPlanMode         │
 │     (user approval gate) │
 └─────────────────────────┘
```

### Phase 1: Research and Plan (Plan Mode)

The batch skill begins by injecting a prompt that instructs Claude to enter plan mode via the `EnterPlanMode` tool. In plan mode, the model:

1. **Research** -- Launches foreground subagents to deeply explore what the instruction touches. These run synchronously because their results are needed to build the plan.

2. **Decompose** -- Breaks the work into 5-30 independent units (constants `MIN_AGENTS = 5`, `MAX_AGENTS = 30`). Each unit must be:
   - Independently implementable in an isolated git worktree
   - Mergeable on its own without depending on sibling PRs
   - Roughly uniform in size

   The prompt specifies a scaling heuristic: few files means closer to 5 units, hundreds of files means closer to 30. It prefers per-directory or per-module slicing over arbitrary file lists.

3. **E2E test recipe** -- The model must figure out how a worker can verify its change end-to-end. It looks for browser automation skills, tmux CLI verifiers, dev-server + curl patterns, or existing integration test suites. If no concrete e2e path exists, it must ask the user (via `AskUserQuestion`) before proceeding -- workers cannot ask the user themselves.

4. **Write the plan** -- The plan file includes: research summary, numbered work units (title, files, description), the e2e recipe, and worker instructions.

5. **ExitPlanMode** -- Presents the plan to the user for approval.

### The Plan/Approval Gate

The `EnterPlanMode` / `ExitPlanMode` tool pair creates a hard gate between research and execution. From `src/tools/ExitPlanModeTool/prompt.ts`:

- The model writes its plan to a plan file
- `ExitPlanMode` signals readiness and surfaces the plan content for user review
- The user must approve before execution proceeds
- This prevents wasted compute on a misguided decomposition

This gate is critical because spawning 5-30 worktree agents is expensive. The user sees the full plan -- every work unit, file list, and the verification strategy -- before any code is written.

### Phase 2: Spawn Workers

Once approved, the batch skill spawns one background agent per work unit. The prompt requires:

- **All agents in a single message block** -- so they launch in parallel
- **`isolation: "worktree"`** -- each agent gets its own git worktree
- **`run_in_background: true`** -- non-blocking execution

Each worker prompt is fully self-contained and includes:

1. The overall goal (user's instruction)
2. This unit's specific task (title, file list, change description -- copied verbatim from plan)
3. Codebase conventions discovered during research
4. The e2e test recipe
5. The worker instructions (a fixed template)

### Worker Instructions Template

The `WORKER_INSTRUCTIONS` constant defines what every worker does after implementing its change:

```
After you finish implementing the change:
1. Simplify -- Invoke the Skill tool with skill: "simplify" to review and clean up changes.
2. Run unit tests -- Run the project's test suite. If tests fail, fix them.
3. Test end-to-end -- Follow the e2e test recipe from the coordinator's prompt.
4. Commit and push -- Commit with a clear message, push the branch, create a PR with gh pr create.
5. Report -- End with a single line: PR: <url> so the coordinator can track it.
```

Each worker self-reviews via `/simplify`, runs tests, verifies e2e, commits, pushes, creates a PR, and reports the URL. If no PR was created, it reports `PR: none -- <reason>`.

### Phase 3: Progress Tracking

After launching all workers, the coordinator renders an initial status table:

```
| # | Unit         | Status  | PR  |
|---|--------------|---------|-----|
| 1 | auth-module  | running | --  |
| 2 | api-routes   | running | --  |
| 3 | tests        | running | --  |
```

As `<task-notification>` messages arrive from background agents, the coordinator:
1. Parses the `PR: <url>` line from each agent's result
2. Re-renders the table with updated status (`done` / `failed`) and PR links
3. Keeps a brief failure note for any agent that did not produce a PR
4. When all agents report, renders the final table and a summary like "22/24 units landed as PRs"

---

## 2. The `/simplify` Skill: 3-Agent Parallel Review

**Source**: `src/skills/bundled/simplify.ts`

`/simplify` runs a parallel 3-agent code review, then fixes the issues found. It is also invoked by `/batch` workers as a self-review step.

```
  Phase 1              Phase 2                    Phase 3
 ┌──────────┐    ┌──────────────────────┐    ┌──────────────┐
 │ git diff  │    │ Launch 3 agents      │    │ Aggregate    │
 │ to find   │───>│ in one message       │───>│ findings     │
 │ changes   │    │ (parallel)           │    │ Fix issues   │
 └──────────┘    │  ├─ Code Reuse Review │    │ Skip false   │
                 │  ├─ Code Quality      │    │ positives    │
                 │  └─ Efficiency Review │    └──────────────┘
                 └──────────────────────┘
```

### The Three Review Agents

Each agent receives the full diff and reviews from a different angle:

**Agent 1: Code Reuse Review**
- Searches for existing utilities that could replace newly written code
- Flags functions that duplicate existing functionality
- Identifies inline logic that could use existing utilities (string manipulation, path handling, type guards, etc.)

**Agent 2: Code Quality Review**
- Redundant state, parameter sprawl, copy-paste with variation
- Leaky abstractions, stringly-typed code
- Unnecessary JSX nesting, unnecessary comments

**Agent 3: Efficiency Review**
- Redundant computations, N+1 patterns, missed concurrency
- Hot-path bloat, recurring no-op updates
- Unnecessary existence checks (TOCTOU), memory leaks, overly broad operations

### Fix Phase

After all three agents complete, the coordinator aggregates their findings and fixes each issue directly. False positives are noted and skipped without argument.

---

## 3. Coordinator Mode

**Source**: `src/coordinator/coordinatorMode.ts`

Coordinator mode is the general-purpose orchestration layer. When enabled (via `CLAUDE_CODE_COORDINATOR_MODE=1`), the system prompt switches to a coordinator persona that delegates to workers via the `Agent` tool and communicates with them via `SendMessage`.

### System Prompt Structure

The coordinator system prompt (`getCoordinatorSystemPrompt()`) defines:

| Section | Content |
|---------|---------|
| Role | Coordinator: direct workers, synthesize results, communicate with user |
| Tools | Agent (spawn), SendMessage (continue), TaskStop (abort), subscribe_pr_activity |
| Task Workflow | Research -> Synthesis -> Implementation -> Verification phases |
| Concurrency | "Parallelism is your superpower" -- launch independent workers concurrently |
| Worker Prompts | Detailed guidance on prompt construction (Section 5, see below) |

### Phase-Based Workflow

```
| Phase          | Who              | Purpose                                     |
|----------------|------------------|---------------------------------------------|
| Research       | Workers (parallel) | Investigate codebase, find files, understand |
| Synthesis      | Coordinator      | Read findings, craft implementation specs    |
| Implementation | Workers          | Make targeted changes per spec, commit       |
| Verification   | Workers          | Test changes work                            |
```

### Concurrency Rules

- **Read-only tasks** (research) -- run in parallel freely
- **Write-heavy tasks** (implementation) -- one at a time per set of files
- **Verification** -- can sometimes run alongside implementation on different file areas

### Worker Context Injection

`getCoordinatorUserContext()` injects runtime information:
- Available tools for workers (from `ASYNC_AGENT_ALLOWED_TOOLS`, excluding internal-only tools like `TeamCreate`, `TeamDelete`, `SendMessage`, `SyntheticOutput`)
- Connected MCP server names
- Scratchpad directory for cross-worker durable knowledge sharing (when the `tengu_scratch` feature gate is enabled)

---

## 4. Worker Prompt Construction Patterns

The coordinator system prompt contains extensive guidance on writing effective worker prompts. These patterns apply to both `/batch` workers and general coordinator workers.

### The Synthesis Rule

The coordinator's "most important job" is synthesis. When workers report research findings:

```
// Anti-pattern -- lazy delegation
Agent({ prompt: "Based on your findings, fix the auth bug" })

// Good -- synthesized spec
Agent({ prompt: "Fix the null pointer in src/auth/validate.ts:42.
  The user field on Session (src/auth/types.ts:15) is undefined when
  sessions expire but the token remains cached. Add a null check before
  user.id access -- if null, return 401 with 'Session expired'.
  Commit and report the hash." })
```

The phrases "based on your findings" and "based on the research" are explicitly called out as anti-patterns -- they delegate understanding to the worker instead of proving comprehension.

### Self-Contained Prompts

Workers start with zero context. Every prompt must include:
- What you are trying to accomplish and why
- What you have already learned or ruled out
- Enough surrounding context for judgment calls
- Specific file paths, line numbers, error messages
- A clear definition of "done"

### Purpose Statement

Include a brief purpose so workers calibrate depth:
- "This research will inform a PR description -- focus on user-facing changes."
- "I need this to plan an implementation -- report file paths, line numbers, and type signatures."
- "This is a quick check before we merge -- just verify the happy path."

### Continue vs. Spawn Decision

| Situation | Mechanism | Why |
|-----------|-----------|-----|
| Research explored exactly the files that need editing | Continue (SendMessage) | Worker already has files in context |
| Research was broad but implementation is narrow | Spawn fresh (Agent) | Avoid exploration noise in context |
| Correcting a failure | Continue | Worker has error context |
| Verifying another worker's code | Spawn fresh | Fresh eyes, no implementation assumptions |
| Wrong approach entirely | Spawn fresh | Avoid anchoring on failed path |

---

## 5. Work Decomposition Strategies

### `/batch` Decomposition

The batch prompt specifies these constraints for work units:

1. **Independence** -- Each unit implementable in an isolated git worktree with no shared state with siblings
2. **Mergeability** -- Each unit mergeable on its own without depending on another unit's PR
3. **Uniformity** -- Roughly uniform in size; split large units, merge trivial ones
4. **Scaling** -- Count scales with actual work: few files = ~5 units, hundreds of files = ~30 units
5. **Slicing preference** -- Per-directory or per-module slicing over arbitrary file lists

### Coordinator Decomposition

The coordinator uses a more flexible approach:
- Research phase fans out with independent exploration agents
- Synthesis phase determines the right decomposition
- Implementation workers get specific file-level assignments
- Verification runs independently per implementation

---

## 6. Fork Subagent: Cache-Optimized Parallel Spawning

**Source**: `src/tools/AgentTool/forkSubagent.ts`, `src/utils/forkedAgent.ts`

The fork subagent pattern is a cache-optimized alternative to spawning fresh agents. When enabled (feature gate `FORK_SUBAGENT`), omitting `subagent_type` from an `Agent` call creates a fork that inherits the parent's full conversation context.

### Cache Sharing Mechanism

The key insight is that forked agents share the parent's prompt cache. The Anthropic API cache key is composed of: system prompt, tools, model, messages (prefix), and thinking config. `CacheSafeParams` carries these values identically between parent and fork:

```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt        // Must match parent
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext     // Contains tools, model
  forkContextMessages: Message[]    // Parent's message prefix
}
```

### Building Cache-Identical Prefixes

`buildForkedMessages()` constructs child messages to maximize cache hits across parallel forks:

```
Parent messages:  [...history, assistant(tool_use_1, tool_use_2, ...)]
                                      │
Fork child A:     [...history, assistant(all_tool_uses),
                   user(placeholder_result_1, placeholder_result_2, ..., directive_A)]
                                                                         ▲ differs
Fork child B:     [...history, assistant(all_tool_uses),
                   user(placeholder_result_1, placeholder_result_2, ..., directive_B)]
                                                                         ▲ differs
```

All fork children produce byte-identical API request prefixes. Only the final text block (the per-child directive) differs. The placeholder text for all tool results is the constant string `"Fork started -- processing in background"`.

### Fork Child Rules

The `buildChildMessage()` function injects strict operating rules into each fork:

```
You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. USE your tools directly: Bash, Read, Write, etc.
4. If you modify files, commit your changes before reporting.
5. Do NOT emit text between tool calls. Report once at the end.
6. Stay strictly within your directive's scope.
7. Keep your report under 500 words.
8. Your response MUST begin with "Scope:".
```

Output format enforced:
```
Scope: <echo back assigned scope>
Result: <answer or key findings>
Key files: <relevant file paths>
Files changed: <list with commit hash>
Issues: <list if any>
```

### Worktree Notice for Forks

When a fork runs in an isolated worktree, `buildWorktreeNotice()` is injected:

```
You've inherited the conversation context above from a parent agent
working in /original/path. You are operating in an isolated git worktree
at /worktree/path -- same repository, same relative file structure,
separate working copy. Paths in the inherited context refer to the
parent's working directory; translate them to your worktree root.
Re-read files before editing if the parent may have modified them.
```

### Anti-Recursion Guard

`isInForkChild()` prevents recursive forking by scanning conversation history for the `<fork-boilerplate>` tag. Fork children keep the Agent tool in their pool (for cache-identical tool definitions) but are blocked from forking at call time.

### When to Fork vs. Spawn Fresh

From the Agent tool prompt (`src/tools/AgentTool/prompt.ts`):

- **Fork** (omit `subagent_type`): when the intermediate tool output is not worth keeping in your context. Forks inherit context and share the parent's cache.
- **Spawn fresh** (specify `subagent_type`): when the worker needs independent assessment without parent context, or a specialized agent type.

Key fork guidance:
- "Forks are cheap because they share your prompt cache"
- "Don't set `model` on a fork -- a different model can't reuse the parent's cache"
- "Don't peek" -- do not Read the fork's output file mid-flight; trust the completion notification
- "Don't race" -- never fabricate or predict fork results

---

## 7. Progress Tracking Details

### Background Agent Notifications

Worker results arrive as `<task-notification>` XML in user-role messages:

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

### Live Progress Updates

`runAsyncAgentLifecycle()` in `src/tools/AgentTool/agentToolUtils.ts` drives background agents with real-time progress:

1. Creates a `ProgressTracker` for each agent
2. On each message, calls `updateProgressFromMessage()` to track tool use counts, activity descriptions, and token counts
3. Emits `emitTaskProgress` events for SDK consumers
4. On completion: `completeAsyncAgent()` -> `enqueueAgentNotification()`
5. On abort: `killAsyncAgent()` with partial result extraction
6. On error: `failAsyncAgent()` with error message

### Batch Status Table

The `/batch` prompt instructs the coordinator to maintain a markdown table:

```
| # | Unit         | Status  | PR                              |
|---|--------------|---------|----------------------------------|
| 1 | auth-module  | done    | https://github.com/org/repo/123 |
| 2 | api-routes   | done    | https://github.com/org/repo/124 |
| 3 | tests        | failed  | none -- test suite not found     |
```

The coordinator parses the `PR: <url>` line from each agent's result text and re-renders the table after each notification.

---

## 8. Lessons for Building Parallel Agent Workflows

These patterns from the Claude Code source suggest general principles for multi-agent orchestration:

### 1. Gate expensive operations with human approval

`/batch` never spawns 30 worktree agents without a plan approval gate. The `EnterPlanMode` / `ExitPlanMode` pair ensures the user sees every work unit, the decomposition rationale, and the verification strategy before compute begins.

### 2. Make worker prompts fully self-contained

Workers cannot see the parent conversation. Every prompt must include the full context: file paths, conventions, what "done" looks like, and how to verify. The coordinator system prompt explicitly bans phrases like "based on your findings" -- the coordinator must synthesize, not delegate understanding.

### 3. Use isolation for write operations

`/batch` requires `isolation: "worktree"` for every worker. This prevents file conflicts between parallel agents. Read-only research agents can share the filesystem; write-heavy agents need isolation.

### 4. Exploit prompt cache for parallel forks

The fork subagent pattern achieves cost efficiency by ensuring all parallel children share byte-identical API request prefixes. The key techniques:
- Identical placeholder text for all tool results
- Per-child variation only in the final text block
- Inheriting the parent's system prompt bytes (not reconstructing them)
- Cloning `contentReplacementState` so fork children make identical replacement decisions for parent message content

### 5. Separate review concerns into parallel agents

`/simplify` demonstrates that specialized reviewers (reuse, quality, efficiency) find more issues than a single general review. Each agent has a focused checklist, and findings are aggregated afterward.

### 6. Embed verification in worker instructions

`/batch` workers do not just implement and report -- they run `/simplify`, run unit tests, run e2e tests, commit, push, and create a PR. This multi-step verification is baked into the worker instructions template, not left to the coordinator.

### 7. Track progress with structured output conventions

The `PR: <url>` reporting convention and the status table pattern give the coordinator a parseable signal from each worker. This avoids unstructured text parsing and enables automated tracking.

### 8. Scale worker count to the actual work

The batch prompt explicitly says: "Scale the count to the actual work: few files -> closer to 5; hundreds of files -> closer to 30." Over-decomposition creates coordination overhead; under-decomposition misses parallelism opportunities.
