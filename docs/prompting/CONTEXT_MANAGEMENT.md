# Context Management

How Claude Code manages the context window — compaction, tool result budgeting, cache optimization, thinking mode, and message normalization.

## Context Window Budget

```
┌─────────────────────────────────────────────────────┐
│              Full Context Window (model-dependent)   │
├─────────────────────────────────────────────────────┤
│  System Prompt (static + dynamic)                   │
│  ────────────────────────────────────────            │
│  Conversation Messages                              │
│  ────────────────────────────────────────            │
│  Tool Results (budgeted)                            │
│  ────────────────────────────────────────            │
│  ┌─────────────────────────────────────┐            │
│  │  Auto-Compact Buffer: 13,000 tokens │ ← trigger  │
│  ├─────────────────────────────────────┤            │
│  │  Warning Buffer: 20,000 tokens      │ ← notify   │
│  ├─────────────────────────────────────┤            │
│  │  Summary Reserve: 20,000 tokens     │ ← for      │
│  │                                     │   compact   │
│  │                                     │   output    │
│  └─────────────────────────────────────┘            │
└─────────────────────────────────────────────────────┘
```

### Key Thresholds

| Threshold | Buffer | Action |
|-----------|--------|--------|
| Auto-Compact | contextWindow - 13,000 | Trigger automatic compaction |
| Warning | contextWindow - 20,000 | Notify user context is nearly full |
| Error | contextWindow - 20,000 | Block new tool use unless auto-compact |
| Manual Block | contextWindow - 3,000 | Prevent messages until `/compact` |

**Source:** `src/services/compact/autoCompact.ts`

## Auto-Compaction

### Trigger

When `tokenCountWithEstimation()` exceeds the auto-compact threshold:

```typescript
tokenCountWithEstimation(messages):
  // Last API response's token count (input + output + cache)
  // + rough estimate of NEW messages added since
  // Handles parallel tool calls by walking back to FIRST sibling
```

### Execution

1. Call `compactConversation()` — generates a detailed multi-section summary
2. Insert `createCompactBoundaryMessage()` — marker with metadata:
   - `trigger`: how it was initiated (auto, manual, context-overflow)
   - `preTokens`: context size before compaction
   - `preservedSegment`: optional UUID range for partial compaction
3. All messages before the boundary are replaced with the summary

### Partial Compaction

Only old messages are summarized. Recent context (between `headUuid` and `tailUuid`) is preserved intact. This means the model keeps recent tool results and conversation turns while old history is compressed.

### Circuit Breaker

After 3 consecutive compaction failures (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`), auto-compact stops retrying. Prevents infinite compaction loops.

### Suppression

Auto-compact is disabled for:
- `DISABLE_COMPACT` or `DISABLE_AUTO_COMPACT` env vars
- `autoCompactEnabled=false` in user config
- `session_memory` and `compact` query sources (deadlock prevention)
- Context-collapse agents (`marble_origami`)

## Tool Result Budgeting

Tool outputs are aggressively managed to prevent context blowout:

### Per-Tool Limits

```
Default per-tool cap:        50,000 chars
System-wide absolute cap:   400,000 bytes  (100K tokens × 4 bytes)
Per-message aggregate:      200,000 chars  (all tool results in one turn)
```

### Persistence Strategy

When a tool result exceeds its threshold:

1. **Persist to disk:** `{sessionId}/tool-results/{tool_use_id}.{json|txt}`
2. **Return preview:** First ~2,000 bytes + file path reference
3. **Wrap in tags:** `<persisted-output>...</persisted-output>`
4. **Empty guard:** Inject `(toolName completed with no output)` for truly empty results

### Aggregate Budget Enforcement

`enforceToolResultBudget()` tracks cumulative size per message. When the aggregate exceeds 200K chars, the largest individual results are persisted to disk (largest first) until the total is under budget.

State is frozen per `tool_use_id` for prompt cache stability — the same persistence decisions are made across turns.

### Preview Generation

Truncation respects newline boundaries within 2,000 bytes when possible, appending `...` when truncated. This ensures previews are readable, not cut mid-line.

## Prompt Caching

### Cache TTL

| User Type | TTL | Condition |
|-----------|-----|-----------|
| Internal (ant) | 1 hour | Always |
| Subscriber (not in overage) | 1 hour | If allowlisted via GrowthBook |
| Bedrock | 1 hour | If `ENABLE_PROMPT_CACHING_1H_BEDROCK` set |
| Others | 5 minutes | Default ephemeral |

**State latching:** Eligibility is cached in bootstrap state to prevent mid-session TTL flips from busting a 20K-token cache.

### Cache Breakpoint Placement

One `cache_control` marker per request. Default position: last message. For fire-and-forget forks: second-to-last message (avoids orphaning KV cache entries).

### Cache Reference Injection

Tool result blocks before the last cache marker get `cache_reference: block.tool_use_id` — enabling server-side cache microcompact (the server can delete old tool results without re-sending them).

### System Prompt Caching

The system prompt is split into blocks via `splitSysPromptPrefix()`. Each block conditionally gets `cache_control` based on scope (`global` vs ephemeral). The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` separates cacheable static content from per-turn dynamic content.

### Fork Cache Optimization

When forking subagents, all tool_use results are replaced with identical placeholder text. This makes the entire conversation prefix byte-identical across all forks, maximizing prompt cache reuse. Spawning 5 parallel forks costs tokens for 1 full context + 5 small directive suffixes.

## Message Normalization

Before messages reach the API, they go through extensive normalization in `normalizeMessagesForAPI()`:

### Filtering

| Rule | Action |
|------|--------|
| `isVirtual=true` | Remove (display-only REPL artifacts) |
| Progress messages | Remove |
| Non-local system messages | Remove |
| Whitespace-only assistant messages | Remove |
| Orphaned thinking-only messages | Remove (failed streaming retries) |
| Trailing thinking on last assistant | Strip (confuses model) |

### Merging

- Consecutive user messages merged into one (Bedrock requirement)
- Consecutive assistant messages from same API response merged (parallel tool calls share `message.id`)

### Error-Driven Block Stripping

Maps error messages to content block types:
- PDF errors → strip document blocks
- Image errors → strip image blocks
- Targets nearest preceding `isMeta` user message

### Tool Reference Handling

When tool search is disabled: strip ALL `tool_reference` blocks. When enabled: strip only references to unavailable tools. Inject `"Tool loaded."` text sibling to prevent two-consecutive-human-turns pattern.

### System Reminder Smooshing

When enabled (via GrowthBook flag), `<system-reminder>`-prefixed text blocks are folded into adjacent `tool_result` blocks. This reduces message count and improves prompt cache stability.

### Message ID Tags

When `HISTORY_SNIP` feature is enabled, non-meta user messages get `[id:xxxxx]` tags appended — enabling the snip tool to selectively remove messages from conversation.

## Thinking Mode

### Detection

```typescript
modelSupportsThinking(model)         // Claude 4+ all models
modelSupportsAdaptiveThinking(model)  // Sonnet 4.6+ and Opus 4.6+ only
```

### Configuration

```typescript
if (modelSupportsAdaptiveThinking) {
  thinking = { type: 'adaptive' }           // No budget, model decides
} else {
  thinking = { type: 'enabled',
               budget_tokens: maxThinking }  // Capped at maxOutput - 1
}
```

### Triggers

- `shouldEnableThinkingByDefault()` — `MAX_THINKING_TOKENS` env var or settings
- `hasUltrathinkKeyword(text)` — parses "ultrathink" keyword in user messages
- `CLAUDE_CODE_DISABLE_THINKING` env var — global disable
- `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` env var — force budget mode

### Context Management for Thinking

```typescript
getAPIContextManagement({
  hasThinking,
  isRedactThinkingActive,
  clearAllThinking  // true when >1h idle (cache miss)
})

// When thinking is active and not redacted:
//   If clearAllThinking: keep only last thinking turn
//   Otherwise: keep all thinking
```

### Token Accounting

Thinking tokens are counted separately:
- `input_tokens`: prompt consumption
- `cache_creation_input_tokens`: ephemeral cache write (1h TTL)
- `cache_read_input_tokens`: cache hit (10% of normal rate)
- `output_tokens`: model generation (includes thinking)

Context window = input + output (cache excluded from window calculation).

## Max Output Tokens

| Mode | Value | Source |
|------|-------|--------|
| Slot reservation (default) | 8,000 | `CAPPED_DEFAULT_MAX_TOKENS` |
| Override | Custom | `CLAUDE_CODE_MAX_OUTPUT_TOKENS` env var |
| Model default | Model-dependent | Opus: 4K, Sonnet: 4K, Haiku: 1K |
