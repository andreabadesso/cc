# Cost Tracking & Token Economics

How Claude Code tracks costs, estimates tokens, manages cache economics, and enforces spending limits.

## Pricing Model

Pricing is defined as tiers per million tokens (Mtok):

| Tier | Input | Output | Cache Read | Cache Write | Models |
|------|-------|--------|-----------|-------------|--------|
| $3/$15 | $3 | $15 | $0.30 (10%) | $0.75 (25%) | Sonnet 4.6 |
| $5/$25 | $5 | $25 | $0.50 | $1.25 | Opus 4.5, 4.6 |
| $15/$75 | $15 | $75 | $1.50 | $3.75 | Opus 4/4.1 |
| $30/$150 | $30 | $150 | $3.00 | $7.50 | Opus 4.6 fast mode |
| Haiku 3.5 | $0.80 | $4 | $0.08 | $0.20 | Haiku 3.5 |
| Haiku 4.5 | $1 | $5 | $0.10 | $0.25 | Haiku 4.5 |

Web search: flat $0.01 per request.

**Source:** `src/utils/modelCost.ts`

### Cost Calculation Formula

```typescript
cost = (input_tokens / 1M) × inputPrice
     + (output_tokens / 1M) × outputPrice
     + (cache_read_tokens / 1M) × cacheReadPrice     // 10% of input
     + (cache_creation_tokens / 1M) × cacheWritePrice // 25% of input
     + web_search_requests × $0.01
```

### Fast Mode

When fast mode is active AND `usage.speed === 'fast'`, Opus 4.6 costs jump to the $30/$150 tier (6x input, 6x output vs standard).

## Token Accounting

The API returns three distinct token counts:

| Field | Meaning | Cost |
|-------|---------|------|
| `input_tokens` | Standard input, not cached | Full input rate |
| `cache_read_input_tokens` | Cache hit (previously cached) | 10% of input rate |
| `cache_creation_input_tokens` | Cache write (new cache entry) | 25% of input rate |

All three are tracked separately and summed in cost calculation.

## Per-Session Persistence

Costs are persisted per session in project config:

```typescript
{
  lastSessionId: string
  lastCost: number                        // USD
  lastAPIDuration: number                 // ms
  lastToolDuration: number                // ms
  lastTotalInputTokens: number
  lastTotalOutputTokens: number
  lastTotalCacheCreationInputTokens: number
  lastTotalCacheReadInputTokens: number
  lastTotalWebSearchRequests: number
  lastModelUsage: Record<string, {        // Per-model breakdown
    inputTokens, outputTokens,
    cacheReadInputTokens, cacheCreationInputTokens,
    webSearchRequests, costUSD
  }>
}
```

Restored on session resume if `lastSessionId` matches.

## Subagent Cost Rollup

Advisor/subagent costs are extracted from the API response's `usage.iterations` array and recursively added to the parent session total:

```typescript
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

## Token Estimation

Three strategies, from most to least accurate:

### 1. API-Based (Most Accurate)

```typescript
anthropic.beta.messages.countTokens({
  model, system, messages, tools, thinking
})
```

### 2. Haiku Fallback

For compaction and secondary queries. Creates minimal request (`max_tokens=1`) via Haiku 4.5. Falls back to Sonnet on platforms where Haiku doesn't support thinking blocks.

### 3. Rough Estimation

When API is unavailable (primarily Bedrock):

| Content Type | Estimate |
|-------------|----------|
| JSON/JSONL/JSONC | 2 bytes/token (dense, many single-char tokens) |
| Other text | 4 bytes/token |
| Images/documents | 2,000 tokens flat |
| tool_use blocks | roughTokenCount(name + JSON input) |

## Rate Limits & Overage

### Limit Types

| Type | Window | Scope |
|------|--------|-------|
| `five_hour` | 5h rolling | Per-session |
| `seven_day` | 7d rolling | Account-wide |
| `seven_day_opus` | 7d rolling | Opus-specific |
| `seven_day_sonnet` | 7d rolling | Sonnet-specific |
| `overage` | Pay-as-you-go | Extra usage |

### Status Tracking

```typescript
{
  status: 'allowed' | 'allowed_warning' | 'rejected'
  utilization: number            // 0-1 fraction
  resetsAt: number               // Unix epoch seconds
  isUsingOverage: boolean
  overageDisabledReason?: string  // Why overage isn't available
}
```

### Early Warning Thresholds

**5-hour window:** Warn at 90% utilization if only 72% of time elapsed.

**7-day window:** Progressive warnings:
- 75% used with only 60% of time elapsed
- 50% used with only 35% elapsed
- 25% used with only 15% elapsed

### Overage Disabled Reasons

`overage_not_provisioned`, `org_level_disabled`, `org_level_disabled_until`, `out_of_credits`, `seat_tier_zero_credit_limit`, `member_zero_credit_limit`, `no_limits_configured`

## Cost Optimizations

### Prompt Caching

| Operation | Cost vs Full Input |
|-----------|-------------------|
| Cache write | 25% of input rate |
| Cache read | 10% of input rate |
| Cache miss | Full input rate |

**Sticky latching:** Once cache strategy headers are sent, they're locked for the session to avoid busting mid-session cache entries. Latched values: `promptCache1hEligible`, `afkModeHeaderLatched`, `fastModeHeaderLatched`, `cacheEditingHeaderLatched`, `thinkingClearLatched`.

### Compaction

Reduces context to ~25-40% of original. Cost = input + output tokens for the summarization call. Tracked via `preCompactTokenCount` vs `postCompactTokenCount`.

### Microcompact (Cache Editing)

Selective pruning of old tool results without re-sending the full conversation. Uses `cache_deleted_input_tokens` (cache editing beta) to track savings.

### Fork Cache Alignment

Parallel subagents share an identical conversation prefix (all tool results replaced with placeholders). Cost: 1 full context + N small directive suffixes instead of N full contexts.

## /cost Command

```
Subscribers:  Shows limit status and overage info
Non-subscribers: Shows formatted cost breakdown:

Total cost:            $X.XXXX
Total duration (API):  XXs
Total duration (wall): XXs
Total code changes:    X lines added, Y lines removed
Usage by model:
  opus-4-6:  1,234,567 input, 987,654 output, 456,789 cache read ($Y.YY)
  haiku-4-5: ...
```

## Telemetry

Each API call logs `tengu_api_success` with: model, token counts (input, output, cached, uncached), cost USD, duration, cache strategy, request ID, stop reason, query source.

Advisor calls log `tengu_advisor_tool_token_usage` with per-advisor model and token breakdowns at micros precision.
