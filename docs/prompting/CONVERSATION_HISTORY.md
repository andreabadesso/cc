# Conversation History: How It Gets Into Context

A complete trace of how Claude Code collects, manages, and injects conversation history into the API context — from user keystroke to the final `messages` array sent to the Claude API.

---

## The Full Picture at a Glance

Every API call Claude Code makes looks like this:

```
┌─────────────────────────────────────────────────────────────┐
│  API Request                                                │
├─────────────────────────────────────────────────────────────┤
│  system: SystemPromptBlock[]                                │
│    ├── Static sections (cached up to 1h)        ~4-8K tok   │
│    ├── SYSTEM_PROMPT_DYNAMIC_BOUNDARY                       │
│    └── Dynamic sections (per-turn)              ~2-6K tok   │
│                                                             │
│  messages: Message[]                                        │
│    ├── [0] User context (<system-reminder>)      ~1-3K tok  │
│    ├── [1..N] Conversation history               variable   │
│    │    ├── User messages                                   │
│    │    ├── Assistant messages (text + tool_use)             │
│    │    ├── Tool results                                    │
│    │    ├── Attachment messages (memory, skills)             │
│    │    └── System messages (hooks, reminders)              │
│    └── [N+1] Current user message                           │
│                                                             │
│  tools: ToolSchema[]                            ~8-15K tok  │
│  max_tokens: number                             8K default  │
│  thinking: { type: 'adaptive' }                             │
└─────────────────────────────────────────────────────────────┘
```

**Key insight: ALL conversation history is sent on every API call.** There is no sliding window that drops old messages. The entire conversation — from the first message to the latest — is included in the `messages` array. The only thing that shrinks it is compaction (summarization).

---

## Message Lifecycle

### 1. User Types a Message

The user's input becomes a `UserMessage`:

```typescript
{
  type: 'user',
  role: 'user',
  id: 'uuid-...',
  content: [{ type: 'text', text: 'Fix the login bug' }]
}
```

### 2. System Enrichment

Before the API call, several things are prepended/appended:

- **User context message** (message `[0]`): Wraps CLAUDE.md content, current date, and other context in `<system-reminder>` tags
- **Attachment messages**: Memory files, invoked skills, command permissions — injected as additional user messages with `<system-reminder>` wrappers
- **Hook results**: If pre-tool hooks ran, their output is injected

### 3. Message Normalization (`normalizeMessagesForAPI`)

Before messages reach the API, they pass through extensive cleanup:

| Rule | What happens |
|------|--------------|
| Virtual messages (`isVirtual=true`) | Removed — display-only REPL artifacts |
| Progress messages | Removed |
| Non-local system messages | Removed |
| Whitespace-only assistant messages | Removed |
| Orphaned thinking-only messages | Removed — failed streaming retries |
| Trailing thinking on last assistant | Stripped — confuses model |
| Consecutive user messages | Merged into one (Bedrock requirement) |
| Consecutive same-response assistant messages | Merged (parallel tool calls share `message.id`) |
| `<system-reminder>` text blocks | Folded into adjacent `tool_result` blocks (when enabled) |

### 4. Microcompact (Runs Before Every API Call)

Before sending, `microcompactMessages()` selectively prunes old tool results:

**Time-based microcompact** (when cache is cold):
- Triggers when gap since last assistant message > 60 minutes
- Keeps the 5 most recent tool results
- Replaces older ones with `[Old tool result content cleared]`

**Cache-editing microcompact** (preserves cache):
- Uses `cache_edits` API to delete tool results server-side
- Does NOT modify local messages — the server handles it
- Only runs on the main thread (sub-agents excluded)

**Eligible tools for clearing:** Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write

### 5. Cache Markers

One `cache_control` marker is placed per request (default: on the last message). Tool result blocks before this marker get `cache_reference` tags, enabling server-side deletion without re-sending.

### 6. The API Call

The final request:

```typescript
{
  model: 'claude-opus-4-6',
  system: [/* static + dynamic blocks */],
  messages: [/* normalized conversation */],
  tools: [/* tool schemas */],
  max_tokens: 8000,
  thinking: { type: 'adaptive' },
  context_management: { edits: [/* microcompact edits */] }
}
```

### 7. Response → New Messages

The API response becomes an `AssistantMessage` with content blocks:
- `{ type: 'text', text: '...' }` — prose output
- `{ type: 'tool_use', id: '...', name: '...', input: {...} }` — tool calls

Tool results are collected and appended as `tool_result` blocks. The cycle repeats.

---

## What Consumes the Context Window

### Token Budget Breakdown

For a 200K-token context window (Opus 4.6):

| Component | Typical Size | Notes |
|-----------|-------------|-------|
| System prompt (static) | 4,000–8,000 | Cached after first call |
| System prompt (dynamic) | 2,000–6,000 | Memory, env, MCP instructions |
| Tool schemas | 8,000–15,000 | Depends on MCP servers loaded |
| User context (CLAUDE.md) | 500–3,000 | Depends on CLAUDE.md size |
| Conversation history | **Everything else** | Grows until compaction |
| Auto-compact buffer | 13,000 | Reserved — triggers compaction |
| Output reservation | 8,000 | Default `max_tokens` |

**Effective space for conversation:** ~150K–170K tokens before auto-compact triggers.

### What's Expensive in Conversation History

| Content Type | Why It's Expensive |
|-------------|-------------------|
| **Tool results** | File reads, grep output, bash output can be massive |
| **Thinking blocks** | Extended thinking adds output tokens (but is auto-managed) |
| **Images/documents** | ~2,000 tokens each (flat estimate) |
| **Repeated file reads** | Reading the same file multiple times = multiple full copies |
| **Large code edits** | Edit tool results contain old + new content |

### Tool Result Budgets (Hard Caps)

```
Per-tool result cap:        50,000 chars
Per-message aggregate cap: 200,000 chars  (all tool results in one turn)
System-wide absolute cap:  400,000 bytes  (100K tokens × 4 bytes)
```

When exceeded, results are persisted to disk with a ~2KB preview left in context.

---

## How Compaction Manages History

### The Trigger

Auto-compact fires when:

```
tokenCountEstimate > contextWindow - maxOutputTokens - 13,000
```

For Opus 4.6 (200K context, 8K output): triggers at ~179K tokens.

### What Happens

1. **Session memory compact** is tried first (free — uses pre-extracted summary file, no LLM call)
2. If unavailable, **full LLM compaction** runs: a summarizer model reads the entire conversation and produces a 9-section summary
3. The summary replaces ALL messages before a boundary point
4. Recent messages (10K–40K tokens, minimum 5 text-block messages) are preserved verbatim
5. The 5 most recently read files are re-injected as attachments (5K tokens each, 50K total budget)
6. Plans, skills, and tool discovery state are restored

### The Summary Structure

The compaction prompt asks for 9 sections:

1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections (with full snippets)
4. Errors and Fixes
5. Problem Solving
6. All User Messages (every non-tool-result user message)
7. Pending Tasks
8. Current Work
9. Optional Next Step (with verbatim quotes to prevent drift)

### After Compaction, the Messages Array Looks Like

```
[CompactBoundaryMessage]     ← metadata: trigger, preTokens, etc.
[Summary as UserMessage]     ← "This session is being continued..."
[Preserved recent messages]  ← last 10K-40K tokens kept verbatim
[File re-read attachments]   ← 5 most recent files
[Plan attachments]
[Skill attachments]
[Current user message]
```

---

## Practical Tips to Reduce Token Usage

### 1. Use `/compact` Proactively with Custom Instructions

Don't wait for auto-compact. Run `/compact` with instructions to focus the summary:

```
/compact Focus on the authentication refactoring. Drop all the debugging we did for the CSS issue.
```

This gives you control over what's preserved vs. discarded.

### 2. Add `## Compact Instructions` to CLAUDE.md

The compaction prompt explicitly looks for summarization instructions in your context:

```markdown
## Compact Instructions
When summarizing, focus on TypeScript code changes and API contract decisions.
Discard debugging sessions and exploratory file reads.
```

### 3. Avoid Repeated File Reads

Every `Read` tool result adds the full file content to the conversation. If you're reading a 1,000-line file five times, that's five copies in history (~20K tokens).

**Instead:** Read once, then reference by line numbers. Or read specific line ranges:

```
Read file.ts lines 100-150
```

### 4. Be Specific with Grep and Glob

Broad searches return massive results that stay in context forever:

```
# Bad: returns hundreds of matches, all stored in history
Grep for "import" in src/

# Good: targeted search, small result
Grep for "import.*AuthService" in src/services/
```

### 5. Use Agents for Exploratory Work

Sub-agents get their own context. Their full exploration doesn't pollute your main conversation — only the final summary comes back:

```
Use an agent to investigate how the payment system handles refunds.
Report back with file paths and key functions.
```

The agent might read 50 files and run 20 greps, but your main context only grows by the summary paragraph.

### 6. Minimize Bash Output

Long bash outputs (test suites, build logs, `ls -la` on large directories) consume tokens:

```
# Bad: full npm output in context
npm install

# Better: suppress output
npm install --silent

# Best: pipe to file, read selectively
npm test 2>&1 | tail -20
```

### 7. Keep CLAUDE.md Lean

Your CLAUDE.md content is injected into EVERY API call as the first message. A 5KB CLAUDE.md costs ~1,250 tokens per turn. Over a 50-turn conversation, that's 62,500 tokens just for CLAUDE.md (though cached after first call at 10% cost).

### 8. Use Partial Compaction for Pivot Points

When switching tasks mid-conversation, use partial compaction to summarize the old task while keeping recent context:

```
/compact up_to here
```

This summarizes everything before the boundary while keeping your recent work intact.

### 9. Understand Cache Economics

Not all tokens cost the same:

| Token Type | Cost vs Full Rate |
|-----------|-------------------|
| Cache read (prompt cache hit) | **10%** |
| Cache write (first time) | 25% |
| Cache miss (no caching) | 100% |
| Output tokens | 100% |

The system prompt and early conversation turns typically get cached. Your actual per-turn cost is mostly: output tokens + the new message tokens since the last cache breakpoint.

**Subscribers get 1-hour cache TTL.** This means during active work, you're mostly paying 10% rate for the cached prefix. If you leave for over an hour, the cache goes cold and the next turn is a full-price cache write.

### 10. Monitor with `/cost`

Run `/cost` to see your actual token usage breakdown:

```
/cost
```

This shows input/output/cache-read/cache-write token counts per model. Look for:
- High `input_tokens` (uncached) → your cache is busting frequently
- High `output_tokens` → model is being verbose (try "be concise" in CLAUDE.md)
- Low `cache_read_input_tokens` → you're not getting cache hits

### 11. Disable Extended Thinking for Simple Tasks

Extended thinking generates additional output tokens. For straightforward tasks:

```
CLAUDE_CODE_DISABLE_THINKING=1
```

Or configure in settings. Adaptive thinking (default for Opus/Sonnet 4.6) lets the model decide, which is usually efficient.

### 12. Deferred Tool Loading Saves System Prompt Tokens

Claude Code defers tool schemas until requested. With 40+ built-in tools plus MCP servers, tool schemas alone can be 15K+ tokens. Only the tools you actually use get their full schemas loaded via `ToolSearch`.

---

## Token Flow Diagram

```
Turn 1:
  → [System Prompt: 10K] + [CLAUDE.md: 2K] + [User msg: 100] + [Tools: 12K]
  ← [Assistant response: 500] + [Tool calls: 200]
  → [Tool results: 3K]
  ← [Assistant response: 800]
  
  Total context at end of turn 1: ~28.6K tokens

Turn 10:
  → [System: 10K] + [CLAUDE.md: 2K] + [History turns 1-9: 80K] + [User msg: 150] + [Tools: 12K]
  ← Response
  
  Total context: ~104K tokens
  Cache hit on system + early history: paying 10% on ~90K = 9K effective tokens

Turn 30 (auto-compact triggers at ~179K):
  → Compaction runs, summarizes turns 1-25 into ~5K summary
  → [System: 10K] + [Summary: 5K] + [History turns 26-30: 30K] + [Tools: 12K] + [File re-reads: 20K]
  
  Post-compact context: ~77K tokens (from 179K)
  
Turn 31+:
  → Grows again from 77K until next compaction
```

---

## Key Constants Reference

| Constant | Value | What It Controls |
|----------|-------|-----------------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Buffer before auto-compact triggers |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | When user sees "context nearly full" |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Forces user to run `/compact` |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Reserved for compaction output |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Files re-read after compaction |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Total budget for post-compact file attachments |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | Per-file cap in attachments |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | Default output reservation |
| Per-tool result cap | 50,000 chars | Max size of any single tool result |
| Per-message aggregate cap | 200,000 chars | Max total tool results in one turn |
| Max MEMORY.md lines | 200 | Memory index truncation |
| Max MEMORY.md size | 25,000 bytes | Memory index size cap |
| Session memory `minTokens` | 10,000 | Min tokens preserved after session-memory compact |
| Session memory `maxTokens` | 40,000 | Max tokens preserved after session-memory compact |
| Session memory `minTextBlockMessages` | 5 | Min messages kept after session-memory compact |
| Microcompact `gapThresholdMinutes` | 60 | Time-based MC trigger (matches cache TTL) |
| Microcompact `keepRecent` | 5 | Tool results kept by time-based MC |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker for compaction loops |

---

## Message Types in the Conversation Array

```typescript
type Message =
  | UserMessage                    // User-submitted text
  | AssistantMessage               // Model response (text + tool_use blocks)
  | SystemMessage                  // System-injected (hooks, reminders)
  | SystemCompactBoundaryMessage   // Compaction marker
  | ToolUseSummaryMessage          // Summary replacing tool results
  | AttachmentMessage              // Memory, skills, permissions
  | ProgressMessage                // Tool execution status (filtered before API)
  | TombstoneMessage               // Placeholder for deleted messages
```

Only `UserMessage`, `AssistantMessage`, and their embedded `tool_use`/`tool_result` blocks make it to the API after normalization. Everything else is either filtered out or folded into these base types.

---

## Related Documentation

- [CONTEXT_MANAGEMENT.md](CONTEXT_MANAGEMENT.md) — Auto-compaction thresholds, tool result budgets, cache optimization, thinking mode
- [SYSTEM_PROMPT.md](SYSTEM_PROMPT.md) — System prompt assembly pipeline and CLAUDE.md injection
- [../COMPACTION.md](../COMPACTION.md) — Full compaction system: 6 strategies, prompts, boundary management, microcompact
- [../COST_TRACKING.md](../COST_TRACKING.md) — Pricing tiers, cache economics, token estimation, rate limits
- [PATTERNS.md](PATTERNS.md) — Fork cache alignment and other cost-saving patterns
