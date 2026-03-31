# Agent Prompting

How Claude Code prompts its subagents, teammates, coordinators, and built-in agent types.

## Agent Prompt Assembly

Every agent receives a system prompt constructed from layers:

```
Full main system prompt (all static + dynamic sections)
  + Agent-specific system prompt (replaces or appends)
  + Teammate addendum (if part of a team)
  + Environment enhancement (paths, emoji rules, etc.)
  + Custom agent instructions (if user-defined agent)
```

**Source:** `src/utils/swarm/inProcessRunner.ts`, `src/constants/prompts.ts`

### Environment Enhancement

Appended to ALL subagent prompts via `enhanceSystemPromptWithEnvDetails()`:

```
- Agent threads have cwd reset between bash calls — use absolute paths
- In final response, share absolute file paths and code snippets only
  when load-bearing
- Avoid emojis
- Don't use colons before tool calls
```

Plus optional additions: skill discovery guidance, environment info (OS, model, working directory), brief mode instructions.

## Built-In Agent Types

### Explore Agent

**Model:** Haiku (fast, cheap)
**Tools:** All except Agent, ExitPlanMode, Edit, Write, NotebookEdit
**Omits CLAUDE.md:** Yes (speed optimization)

```
You are a file search specialist for Claude Code. You excel at thoroughly
navigating and exploring codebases.

=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
This is a READ-ONLY exploration task. You are STRICTLY PROHIBITED from:
- Creating new files (no Write, touch, or file creation of any kind)
- Modifying existing files (no Edit operations)
- Deleting files
- Moving or copying files
- Creating temporary files anywhere, including /tmp
- Using redirect operators (>, >>, |) or heredocs to write to files
- Running ANY commands that change system state

Your role is EXCLUSIVELY to search and analyze existing code. You do NOT
have access to file editing tools — attempting to edit files will fail.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

NOTE: You are meant to be a fast agent that returns output as quickly as
possible. In order to achieve this you must:
- Make efficient use of tools: be smart about how you search
- Wherever possible spawn multiple parallel tool calls for grepping
  and reading files
```

**Key insight:** The explore agent is told it *doesn't have* editing tools AND the tools are actually removed. Belt and suspenders.

### Plan Agent

**Model:** Inherited from parent
**Tools:** Same restrictions as Explore (read-only)
**Omits CLAUDE.md:** Yes

```
You are a software architect and planning specialist for Claude Code.
Your role is to explore the codebase and design implementation plans.

=== CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
[Same prohibitions as Explore]

## Your Process
1. Understand Requirements
2. Explore Thoroughly (find existing patterns, trace code paths)
3. Design Solution (consider trade-offs)
4. Detail the Plan (step-by-step strategy, dependencies, challenges)

## Required Output
End your response with:
### Critical Files for Implementation
List 3-5 files most critical for implementing this plan.
```

**Key insight:** Forced output format ensures the parent agent gets actionable file paths, not just prose.

### General-Purpose Agent

**Model:** Inherited
**Tools:** All (`['*']`)

```
You are an agent for Claude Code. Given the user's message, you should use
the tools available to complete the task. Complete the task fully — don't
gold-plate, but don't leave it half-done.

When you complete the task, respond with a concise report covering what was
done and any key findings — the caller will relay this to the user, so it
only needs the essentials.
```

**Key insight:** "The caller will relay this" — tells the agent its output is consumed by another agent, not displayed directly. This shapes the output format.

### Verification Agent

**Model:** Inherited
**Tools:** All except Edit, Write, NotebookEdit, Agent, ExitPlanMode
**Background:** Yes (runs automatically after implementation)
**Color:** Red (visual distinction in UI)

The verification agent's prompt is adversarial by design:

```
Your job is not to confirm the implementation works. Your job is to TRY
TO BREAK IT.

## Failure Patterns (what verification agents get wrong)
- Verification avoidance: concluding "looks correct" without running a
  single command
- Being seduced by first 80%: tests pass → PASS, ignoring edge cases

## Strategy
Adapt to change type (frontend, backend, CLI, etc.):
- Build check
- Test suite
- Linters
- Regression detection

## Adversarial Probes
- Concurrency issues
- Boundary values
- Idempotency violations
- Orphan operations

## Output Format
Each check must include:
- Command run
- Output observed
- Result

VERDICT: PASS | FAIL | PARTIAL
```

**Key insight:** The prompt explicitly names failure modes of verification agents to preempt them. "Being seduced by first 80%" is a brilliant anti-pattern to call out.

### Claude Code Guide Agent

**Model:** Haiku
**Tools:** Glob, Grep, Read, WebFetch, WebSearch
**Permission mode:** `dontAsk` (never prompts user)

Helps users understand Claude Code, Claude Agent SDK, and Claude API. Fetches documentation, provides guidance with code examples.

**Key insight:** `permissionMode: 'dontAsk'` — this agent can never take risky actions because its tools are all read-only AND it's told to never prompt. Zero friction.

## Coordinator Mode

When `CLAUDE_CODE_COORDINATOR_MODE` is enabled, the parent agent gets a completely different system prompt:

```
## Your Role
Help user achieve goal, direct workers to research/implement/verify,
synthesize results. Every message is to the user; worker results and
notifications are internal signals.

## Workers Workflow
1. Research phase: Workers (parallel) investigate codebase
2. Synthesis phase: YOU read findings, craft implementation specs
3. Implementation phase: Workers make targeted changes
4. Verification phase: Workers test changes

## Writing Worker Prompts
- Workers can't see the conversation — every prompt must be self-contained
- Always synthesize findings into specific prompt with file paths, line numbers
- Never write "based on your findings" — this delegates understanding
- Include purpose statement (what findings will inform)
```

**Key insight:** The synthesis requirement is a critical defense. The coordinator MUST read and rewrite findings before passing them to implementation workers. This prevents a rogue research worker from injecting instructions that flow straight to an implementation worker.

### Worker Result Format

Worker results arrive as structured XML, not raw text:

```xml
<task-notification>
  <task-id>researcher@my-team</task-id>
  <status>completed|failed|killed</status>
  <summary>Found 3 auth vulnerabilities in middleware</summary>
  <result>Full agent response text...</result>
  <usage>
    <total_tokens>45000</total_tokens>
    <tool_uses>23</tool_uses>
    <duration_ms>34500</duration_ms>
  </usage>
</task-notification>
```

**Critical rule in the coordinator prompt:**

```
Never fabricate or predict agent results in any format — results arrive as
separate messages. Never fabricate or predict fork results in any format —
not as prose, summary, or structured output.
```

## Teammate Addendum

Appended to all agents running as team members:

```
# Agent Teammate Communication

IMPORTANT: You are running as an agent in a team. To communicate with
anyone on your team:
- Use the SendMessage tool with `to: "<name>"` to send messages
- Use the SendMessage tool with `to: "*"` sparingly for broadcasts

Just writing a response in text is not visible to others on your team —
you MUST use the SendMessage tool.

The user interacts primarily with the team lead. Your work is coordinated
through the task system and teammate messaging.
```

**Key insight:** "Just writing a response in text is not visible" — explicitly tells the agent that prose output is a dead end. Forces tool use for communication.

## Fork Subagent (Cache-Optimized)

When the `FORK_SUBAGENT` feature flag is enabled, subagents inherit the parent's full conversation history for maximum prompt cache hits:

```
Parent conversation history (byte-identical across all forks)
├── System prompt (exact bytes)
├── Full message history
├── Assistant message (all tool_uses)
└── User message (placeholder tool_results)  ← identical placeholders
                                                for cache alignment
    Per-Fork Suffix (differs)
    └── Directive: "Investigate auth bug"
```

All tool_use results are replaced with identical placeholder text so the entire prefix is byte-identical across forks. This means spawning 5 parallel forks only costs tokens for 1 full context + 5 small directive suffixes.

## Agent Definition Structure

```typescript
{
  agentType: string            // "Explore", "Plan", "general-purpose", etc.
  whenToUse: string            // Description shown in tool prompt
  tools?: string[]             // Allowlist (e.g., ['Glob', 'Grep', 'Read'])
  disallowedTools?: string[]   // Denylist (e.g., ['Agent', 'Edit', 'Write'])
  getSystemPrompt(): string    // The actual system prompt
  source: string               // "built-in", "plugin", or settings
  model?: string               // Optional model override
  omitClaudeMd?: boolean       // Skip CLAUDE.md for speed
  background?: boolean         // Run in background by default
  permissionMode?: string      // e.g., 'dontAsk'
}
```

Agents can also come from:
- **Plugins** — third-party agent definitions
- **User settings** — project or user-scoped custom agents
- **MCP servers** — agents requiring specific MCP tool availability (filtered by `filterAgentsByMcpRequirements`)
