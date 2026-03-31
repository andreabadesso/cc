# Conversation Compaction System

The compaction system manages context window pressure in long-running agent sessions. When a conversation grows too large for the model's context window, compaction summarizes older messages, prunes tool results, or edits the server-side cache to free space while preserving the information the model needs to continue working.

All code lives under `src/services/compact/`.

---

## Strategy Hierarchy

Compaction is not a single mechanism. There are six distinct strategies arranged in a priority/trigger hierarchy:

| Strategy | Trigger | Mechanism | Lossy? | Preserves Cache? |
|---|---|---|---|---|
| **Microcompact (time-based)** | Gap since last assistant message > threshold (default 60min) | Content-clears old tool results in-place | Partially | No (cache is cold anyway) |
| **Microcompact (cached / cache editing)** | Count-based: tool results exceed trigger threshold | Sends `cache_edits` API blocks to delete tool results server-side | Partially | Yes |
| **Auto-compact (proactive)** | Token count exceeds `effectiveContextWindow - 13,000` | Full LLM-generated summary replaces conversation | Yes | No |
| **Session memory compact** | Same as auto-compact, tried first | Uses pre-extracted session memory file as summary (no LLM call) | Yes | No |
| **Reactive compact** | API returns `prompt_too_long` (413) error | Peels API-round groups from tail, retries until it fits | Yes | No |
| **Manual `/compact`** | User types `/compact [instructions]` | Same as auto-compact but user-initiated, supports custom instructions | Yes | No |
| **Partial compact** | User selects a message boundary in the UI | Summarizes one side of the pivot, keeps the other verbatim | Yes | Depends on direction |

### Execution order in the query loop

1. **Microcompact** runs first (in `microcompactMessages`), before every API call. It checks time-based trigger, then cached MC.
2. **Auto-compact** (`autoCompactIfNeeded`) runs after each turn if token usage exceeds the threshold. It tries session-memory compact first, then falls back to full LLM compaction.
3. **Reactive compact** (`tryReactiveCompact`) fires when the API itself returns a prompt-too-long error, as a last-resort recovery.
4. **Manual `/compact`** is user-initiated and follows the same path: session-memory first, then reactive (if in reactive-only mode), then traditional.

---

## The Compaction Prompts

### System prompt for the summarizer

The compaction call uses a dedicated system prompt:

```
You are a helpful AI assistant tasked with summarizing conversations.
```

Thinking is disabled (`thinkingConfig: { type: 'disabled' }`). Tool use is denied.

### No-tools preamble (prepended to all compact prompts)

The model is aggressively instructed not to call tools:

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn -- you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

A trailer is also appended:

```
REMINDER: Do NOT call any tools. Respond with plain text only --
an <analysis> block followed by a <summary> block.
Tool calls will be rejected and you will fail the task.
```

### Full compaction prompt (BASE_COMPACT_PROMPT)

Used for full `/compact` and auto-compact:

```
Your task is to create a detailed summary of the conversation so far,
paying close attention to the user's explicit requests and your previous actions.
This summary should be thorough in capturing technical details, code patterns,
and architectural decisions that would be essential for continuing development
work without losing context.
```

The model is asked to produce an `<analysis>` scratchpad block (stripped before the summary enters context) followed by a `<summary>` block with these 9 sections:

1. **Primary Request and Intent** -- all user requests and intents
2. **Key Technical Concepts** -- technologies, frameworks discussed
3. **Files and Code Sections** -- files examined/modified/created with full code snippets and rationale
4. **Errors and fixes** -- every error and how it was fixed, with user feedback
5. **Problem Solving** -- solved problems and ongoing troubleshooting
6. **All user messages** -- every non-tool-result user message (critical for understanding intent)
7. **Pending Tasks** -- explicitly requested outstanding work
8. **Current Work** -- precise description of what was being worked on immediately before compaction
9. **Optional Next Step** -- next step, only if directly aligned with the user's most recent request; includes verbatim quotes to prevent task drift

The prompt also tells the model to honor any custom summarization instructions found in included context (e.g., from `CLAUDE.md` files):

```
There may be additional summarization instructions provided in the included context.
If so, remember to follow these instructions when creating the above summary.
Examples of instructions include:
## Compact Instructions
When summarizing the conversation focus on typescript code changes...
```

### Partial compaction prompts

There are two partial variants depending on direction:

**`PARTIAL_COMPACT_PROMPT` (direction: `from`)** -- summarizes recent messages only:
```
Your task is to create a detailed summary of the RECENT portion of the conversation --
the messages that follow earlier retained context. The earlier messages are being kept
intact and do NOT need to be summarized.
```

**`PARTIAL_COMPACT_UP_TO_PROMPT` (direction: `up_to`)** -- summarizes everything before the pivot:
```
Your task is to create a detailed summary of this conversation. This summary will be
placed at the start of a continuing session; newer messages that build on this context
will follow after your summary.
```

The `up_to` variant replaces section 8/9 with:
- **Work Completed** -- what was accomplished
- **Context for Continuing Work** -- decisions/state needed for subsequent messages

### Analysis instruction

Both full and partial prompts include a detailed analysis instruction that asks the model to work through the conversation chronologically in `<analysis>` tags:

```
Before providing your final summary, wrap your analysis in <analysis> tags...
1. Chronologically analyze each message and section of the conversation. For each section:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like: file names, full code snippets, function signatures, file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback
2. Double-check for technical accuracy and completeness
```

---

## Summary Format and Post-Processing

The raw model output contains `<analysis>...</analysis>` and `<summary>...</summary>` XML tags.

`formatCompactSummary()` strips the analysis section entirely (it is a scratchpad that improves quality but has no value once written) and replaces `<summary>` tags with a `Summary:` header.

The formatted summary is wrapped in a user message:

```
This session is being continued from a previous conversation that ran out of context.
The summary below covers the earlier portion of the conversation.

Summary:
1. Primary Request and Intent:
   ...
```

If a transcript file exists, it appends:

```
If you need specific details from before compaction (like exact code snippets,
error messages, or content you generated), read the full transcript at: <path>
```

For auto-compact (where `suppressFollowUpQuestions` is true), it appends:

```
Continue the conversation from where it left off without asking the user any further
questions. Resume directly -- do not acknowledge the summary, do not recap what was
happening, do not preface with "I'll continue" or similar. Pick up the last task as
if the break never happened.
```

---

## Compaction Boundary Management

Every compaction creates a **compact boundary message** (`SystemCompactBoundaryMessage`) that serves as a marker in the message history. It records:

- `trigger`: `'manual'` or `'auto'`
- `preTokens`: token count before compaction
- `userContext`: optional custom instructions from the user
- `messagesSummarized`: count of messages that were summarized
- `preCompactDiscoveredTools`: tool names discovered before compaction (preserved across boundary)
- `preservedSegment`: metadata for messages kept verbatim (`headUuid`, `anchorUuid`, `tailUuid`)

After compaction, messages are ordered: `[boundary, summary, messagesToKeep, attachments, hookResults]`.

### What gets preserved across boundaries

1. **Recent messages** (in session-memory compact): calculated by `calculateMessagesToKeepIndex`, which expands backwards from the last summarized message until meeting minimums of 10,000 tokens and 5 text-block messages, with a hard cap of 40,000 tokens.
2. **File state**: the 5 most recently read files are re-read and injected as attachments (up to 5,000 tokens per file, 50,000 total budget).
3. **Plans**: plan file content is re-attached.
4. **Skills**: invoked skill content is re-attached (5,000 tokens per skill, 25,000 total budget).
5. **Deferred tools and MCP instructions**: re-announced via delta attachments.
6. **Discovered tool names**: stored in `compactMetadata.preCompactDiscoveredTools` so the tool search system continues to include them.
7. **Session metadata**: re-appended so `--resume` display still shows custom session titles.

### What gets discarded

- The `<analysis>` scratchpad
- Image/document blocks (replaced with `[image]`/`[document]` markers before summarization)
- Skill discovery/listing attachments (re-injected fresh)
- Progress messages

---

## Microcompact: Selective Pruning

Microcompact does not summarize the conversation. Instead, it selectively clears old tool results to reduce token count without any LLM call.

### Compactable tools

Only results from these tools are eligible for clearing:

- File read (`Read`)
- Shell/Bash tools
- Grep, Glob
- Web search, Web fetch
- File edit, File write

### Time-based microcompact

Configured via `TimeBasedMCConfig`:

```typescript
{
  enabled: boolean           // default: false (feature-flagged)
  gapThresholdMinutes: number // default: 60 (server cache TTL)
  keepRecent: number         // default: 5
}
```

When the gap since the last assistant message exceeds `gapThresholdMinutes`, the server-side prompt cache has expired. Since the full prefix will be rewritten anyway, this is the optimal time to content-clear old tool results.

It keeps the most recent N compactable tool results and replaces older ones with `[Old tool result content cleared]`.

### Cached microcompact (cache editing)

Instead of modifying message content locally, cached microcompact uses the API's `cache_edits` mechanism to delete tool results server-side without invalidating the cached prefix.

Key properties:
- Does NOT modify local message content
- Uses count-based trigger/keep thresholds from remote config
- Tracks tool results by ID and queues `cache_edits` blocks for the API layer
- Pinned edits are re-sent at their original positions in subsequent calls for cache hits
- Only runs on the main thread (sub-agents are excluded)

The flow:
1. Register tool results as they appear in messages
2. When trigger threshold is exceeded, select oldest tools for deletion
3. Create a `CacheEditsBlock` and queue it as `pendingCacheEdits`
4. API layer consumes pending edits via `consumePendingCacheEdits()` and inserts them
5. After API response, compute `cache_deleted_input_tokens` delta from the baseline

### API-based context management (`apiMicrocompact.ts`)

A separate server-side approach using the `context_management` API parameter:

```typescript
{
  edits: [
    {
      type: 'clear_tool_uses_20250919',
      trigger: { type: 'input_tokens', value: 180_000 },
      clear_at_least: { type: 'input_tokens', value: 140_000 },
      clear_tool_inputs: [/* tool names */]
    },
    {
      type: 'clear_thinking_20251015',
      keep: { type: 'thinking_turns', value: 1 } | 'all'
    }
  ]
}
```

This delegates context management entirely to the API server. It can clear tool results (by tool name), clear tool use inputs (for edit/write tools), and manage thinking block retention.

---

## Session Memory Compact

An alternative to LLM-based summarization that uses a pre-extracted session memory file. The session memory system continuously extracts key information during the conversation, so when compaction triggers, it can use that file as the summary without an additional LLM call.

Configuration (from `SessionMemoryCompactConfig`):

| Parameter | Default | Purpose |
|---|---|---|
| `minTokens` | 10,000 | Minimum tokens to preserve after compaction |
| `minTextBlockMessages` | 5 | Minimum messages with text blocks to keep |
| `maxTokens` | 40,000 | Hard cap on preserved tokens |

The system calculates which messages to keep by:
1. Starting from `lastSummarizedMessageId` (the last message the session memory has processed)
2. Expanding backwards to meet both `minTokens` and `minTextBlockMessages`
3. Stopping if `maxTokens` is reached
4. Adjusting to not split `tool_use`/`tool_result` pairs or thinking blocks sharing the same `message.id`

If the post-compact token count would still exceed the auto-compact threshold, session memory compact is skipped and falls back to traditional LLM compaction.

---

## Metrics and Monitoring

### Events logged

| Event | When |
|---|---|
| `tengu_compact` | After successful full compaction |
| `tengu_partial_compact` | After successful partial compaction |
| `tengu_compact_failed` | Compaction failure (with reason: `prompt_too_long`, `no_summary`, `api_error`, `no_streaming_response`) |
| `tengu_compact_ptl_retry` | When compact request itself hits prompt-too-long and retries |
| `tengu_compact_cache_sharing_success` | Forked-agent cache-sharing path succeeded |
| `tengu_compact_cache_sharing_fallback` | Forked-agent path failed, falling back to streaming |
| `tengu_compact_streaming_retry` | Streaming compact retried after failure |
| `tengu_cached_microcompact` | Cached MC deleted tool results |
| `tengu_time_based_microcompact` | Time-based MC cleared tool results |
| `tengu_sm_compact_*` | Session memory compact events (no session memory, empty template, threshold exceeded, error, etc.) |

### Metrics tracked per compaction

- `preCompactTokenCount` -- tokens before compaction
- `postCompactTokenCount` -- total tokens from the compaction API call (input tokens, kept for backwards compatibility)
- `truePostCompactTokenCount` -- estimated token count of the resulting post-compact context
- `willRetriggerNextTurn` -- whether post-compact tokens still exceed the auto-compact threshold
- `compactionInputTokens`, `compactionOutputTokens` -- API usage breakdown
- `compactionCacheReadTokens`, `compactionCacheCreationTokens` -- cache efficiency of the compaction call itself
- Context analysis breakdown (via `analyzeContext`)

### Circuit breaker

Auto-compact has a circuit breaker: after 3 consecutive failures (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`), it stops retrying for the rest of the session. This was added because 1,279 sessions had 50+ consecutive failures (up to 3,272 each), wasting approximately 250K API calls per day.

---

## Thresholds and Constants

| Constant | Value | Purpose |
|---|---|---|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Buffer between effective context window and auto-compact trigger |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | Buffer for showing context warning to user |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Buffer for blocking limit (forces user to compact) |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Reserved output tokens during compaction (based on p99.99 = 17,387) |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Max files re-read after compaction |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Total token budget for file attachments |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | Per-file token limit |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | Per-skill token limit |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | Total skill attachment budget |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | Streaming retry attempts |
| `MAX_PTL_RETRIES` | 3 | Prompt-too-long truncation retries |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker threshold |

The effective auto-compact threshold is: `contextWindowSize - maxOutputTokens - 13,000`.

---

## Prompt-Too-Long Recovery

When the compaction API call itself hits prompt-too-long (the conversation is so large that even the summarization request exceeds the context window), the system:

1. Groups messages by API round using `groupMessagesByApiRound()` (boundaries at new assistant message IDs)
2. Calculates how many groups to drop based on the token gap from the error, or drops 20% as fallback
3. Prepends a synthetic marker `[earlier conversation truncated for compaction retry]` if needed
4. Retries up to 3 times

---

## Cache Sharing for Compaction

The compaction call can reuse the main conversation's prompt cache by running as a "forked agent" that shares the same system prompt, tools, and message prefix. This avoids paying for cache creation on the compaction call.

The forked-agent path:
- Sends the compact prompt as a single user message appended to the shared prefix
- Uses `maxTurns: 1` and `skipCacheWrite: true`
- Falls back to regular streaming if it fails

This is enabled by default (confirmed via Jan 2026 experiment: the non-cache-sharing path had 98% cache miss rate).

---

## Lessons for Building Compaction in Your Own Agent System

1. **Layer your strategies.** Microcompact (cheap, no LLM call) runs on every turn. Full compaction (expensive, one LLM call) runs only when needed. Reactive compaction (emergency) runs only on API errors. Each layer catches what the previous missed.

2. **The analysis scratchpad pattern works.** Having the model write `<analysis>` before `<summary>` improves summary quality. Strip the analysis from the final output.

3. **Preserve API invariants.** When keeping a suffix of messages, you must ensure `tool_use`/`tool_result` pairs are not split. Thinking blocks sharing the same `message.id` must stay together. The `adjustIndexToPreserveAPIInvariants` function handles this.

4. **Re-inject critical context.** After compaction, re-read recently accessed files, re-attach plans, skills, and tool schemas. The model loses these when old messages are replaced with a summary.

5. **Track pre/post token counts.** Knowing whether compaction actually freed enough space to avoid re-triggering on the next turn (`willRetriggerNextTurn`) is essential for debugging compaction loops.

6. **Circuit breakers prevent runaway costs.** If compaction fails 3 times in a row, stop trying. Without this, a single stuck session can make thousands of futile API calls.

7. **Aggressively prevent tool use in the summarizer.** Even with `maxTurns: 1`, models on newer versions attempt tool calls during summarization. The NO_TOOLS_PREAMBLE and NO_TOOLS_TRAILER sandwich, plus a `canUseTool` function that always returns deny, are all needed.

8. **Use session memory as a cheap compaction path.** If you maintain a running summary/memory file during the conversation, you can use it directly instead of making an expensive summarization call. This is the session-memory compact path.

9. **Respect the prompt cache.** Microcompact (cache editing) is specifically designed to free tokens without breaking the cached prefix. Time-based microcompact only fires when the cache is already cold. These are cache-aware strategies.

10. **Handle the recursive failure case.** When the compaction request itself is too long for the context window, truncate the oldest message groups and retry rather than leaving the user stuck.
