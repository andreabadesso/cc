# Startup Optimization & Performance Patterns

Claude Code is an interactive CLI tool where startup latency directly impacts user experience. The codebase demonstrates systematic attention to performance through lazy loading, parallel initialization, and strategic caching.

---

## Startup Sequence

### Fast Paths (`src/entrypoints/cli.tsx`)

Before loading the main application, the entry point checks for commands that don't need it:

```typescript
// Zero module loading -- exits before any imports
if (args.includes('--version') || args.includes('-v')) {
  console.log(MACRO.VERSION)
  process.exit(0)
}

// Minimal imports for specific subcommands
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
  const { enableConfigs } = await import('../utils/config.js')
  // ... minimal work, then exit
}
```

Each subcommand (`--computer-use-mcp`, `--daemon-worker`, `remote-control`, `ps`, `logs`, `new`) has its own fast path that loads only what's needed.

### Initialization Order (`src/entrypoints/init.ts`)

The init function is **memoized** -- calling it multiple times returns the same Promise:

```typescript
export const init = memoize(async (): Promise<void> => { ... })
```

The initialization sequence prioritizes getting the user to an interactive prompt as fast as possible:

```
1. enableConfigs()                       # Validate settings.json
2. applySafeConfigEnvironmentVariables() # Load safe env vars (before TLS)
3. applyExtraCACertsFromConfig()         # CA certs before first TLS (Bun caches cert store at boot)
4. setupGracefulShutdown()               # Register cleanup handlers
5. [ASYNC] 1P Event Logging             # Fire-and-forget
6. [ASYNC] OAuth, JetBrains detection   # Fire-and-forget
7. [ASYNC] GitHub repo detection        # Fire-and-forget
8. [ASYNC] Remote settings init         # Non-blocking with timeout
9. mTLS configuration                   # Before any network calls
10. Proxy configuration                  # Before any network calls
11. preconnectAnthropicApi()            # Fire-and-forget HEAD request
12. [ASYNC] Upstream proxy              # Fire-and-forget, fail-open
```

Steps 5-8 and 11-12 run in parallel. The user sees the prompt after step 10 completes; everything else continues in the background.

### CA Certificate Ordering

A subtle but important detail: `applyExtraCACertsFromConfig()` runs before any TLS connection because Bun's BoringSSL caches the certificate store at boot. If custom CA certs are added after the first TLS handshake, they won't be used.

---

## Lazy Loading Strategies

### Heavy Module Deferral

| Module | Size | When Loaded |
|--------|------|-------------|
| OpenTelemetry | ~400KB | When telemetry is enabled |
| gRPC exporter | ~700KB | When gRPC protocol is selected |
| AWS SDK (Bedrock) | ~279KB | When Bedrock token counting is needed |
| `proper-lockfile` | ~8ms init | When first file lock is needed |

### Protocol-Based Selective Import

The telemetry system only loads the exporter for the configured protocol (`src/utils/telemetry/instrumentation.ts`):

```typescript
switch (protocol) {
  case 'grpc':
    const { OTLPMetricExporter } = await import('@opentelemetry/exporter-metrics-otlp-grpc')
    break
  case 'http/json':
    const { OTLPMetricExporter } = await import('@opentelemetry/exporter-metrics-otlp-http')
    break
  case 'http/protobuf':
    const { OTLPMetricExporter } = await import('@opentelemetry/exporter-metrics-otlp-proto')
    break
}
```

Static imports would load all three (~1.2MB). Dynamic imports load only one.

### Lazy Lockfile (`src/utils/lockfile.ts`)

```typescript
let _lockfile: Lockfile | undefined

function getLockfile(): Lockfile {
  if (!_lockfile) {
    _lockfile = require('proper-lockfile')  // ~8ms, monkey-patches fs
  }
  return _lockfile
}
```

`proper-lockfile` patches `fs` via `graceful-fs` on require. A static import would add 8ms to startup even for `--help`. Lazy loading defers this to the first actual lock operation.

### Circular Dependency Breaking

Tool modules that import the tool registry (which imports them) use lazy getters:

```typescript
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool

const getSendMessageTool = () =>
  require('./tools/SendMessageTool/SendMessageTool.js').SendMessageTool
```

The `require()` is deferred to first call, breaking the cycle `tools.ts → TeamCreateTool → tools.ts`.

---

## API Preconnect (`src/utils/apiPreconnect.ts`)

TCP+TLS handshake takes 100-200ms. The preconnect fires a `HEAD` request to the API endpoint while the application finishes initializing:

```typescript
export function preconnectAnthropicApi(): void {
  if (fired) return  // Idempotent
  fired = true

  // Skip when connection can't be reused (proxy, mTLS, unix socket)
  if (process.env.HTTPS_PROXY || process.env.ANTHROPIC_UNIX_SOCKET) return

  void fetch(baseUrl, {
    method: 'HEAD',
    signal: AbortSignal.timeout(10_000),
  }).catch(() => {})  // Failure is fine -- first real request will connect
}
```

Bun's `fetch` shares a global keep-alive pool, so the connection established by the preconnect is reused for the first actual API call.

The preconnect fires **after** mTLS and proxy configuration (step 10-11), ensuring the connection uses the correct transport settings.

---

## Dead Code Elimination

### Build-Time Feature Flags

`bun:bundle`'s `feature()` function enables compile-time dead code elimination:

```typescript
import { feature } from 'bun:bundle'

// This entire block is stripped from builds where PROACTIVE is disabled
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

### User Type Gating

```typescript
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null
```

External builds set `USER_TYPE` at build time, allowing Bun to eliminate internal-only code paths.

### Impact

The combination of feature flags and user type gating means the external build is significantly smaller than the internal build. Each disabled feature eliminates not just the tool code, but all its transitive dependencies.

---

## Token Budget Management

### Output Token Capping (`src/utils/context.ts`)

```typescript
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000    // Default slot reservation
export const ESCALATED_MAX_TOKENS = 64_000         // Recovery retry
```

Analysis of production data showed that p99 output length is 4,911 tokens. The default 32K/64K `max_tokens` over-reserves 8-16x the actual slot capacity. Capping to 8K by default means:
- Less than 1% of requests hit the limit
- Failed requests are retried with 64K (escalation)
- Infrastructure costs are lower (slot reservation is proportional to `max_tokens`)

### Diminishing Returns Detection (`src/query/tokenBudget.ts`)

```typescript
const COMPLETION_THRESHOLD = 0.9  // Stop at 90% budget usage
const DIMINISHING_THRESHOLD = 500  // <500 tokens/iteration = diminishing

// Stop if: 3+ iterations AND <500 tokens produced in each of the last 2
const isDiminishing =
  tracker.continuationCount >= 3 &&
  deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
  tracker.lastDeltaTokens < DIMINISHING_THRESHOLD
```

This prevents runaway loops where the model produces tiny amounts of output per iteration, wasting API calls.

---

## Memory Prefetch (`src/utils/attachments.ts`)

Memory search (finding relevant past conversations) is fired speculatively while the query loop runs:

```typescript
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(messages, toolUseContext)

// Later: consume if settled, skip if still pending
if (pendingMemoryPrefetch?.settledAt !== null) {
  const memory = await pendingMemoryPrefetch.promise
  // Inject into message flow
}
```

The `using` keyword (TC39 Explicit Resource Management) ensures cleanup happens on all exit paths:
- If consumed: metrics logged with iteration number
- If not consumed: abort controller cancels the search
- On error: caught silently, returns empty array

---

## Caching Strategies

### Feature Value Cache (GrowthBook)

Three-level cache for feature flags:
1. **In-memory**: `Map<string, unknown>` -- fastest, populated by remote eval
2. **Disk**: `~/.claude/settings.json → cachedGrowthBookFeatures` -- survives restarts
3. **Default**: Code-level fallback values

Exposure event deduplication:
```typescript
const loggedExposures = new Set<string>()
// Prevents duplicate events in hot paths (e.g., isAutoMemoryEnabled in render loops)
```

### Session Cost Persistence (`src/cost-tracker.ts`)

Cost state is cached per-project:
- Saved to disk when session ends
- Restored on session resume (validated by session ID)
- Derived fields (context window, max tokens) recomputed on restore to reflect current model capabilities

### Per-Project Disk Cache (`src/utils/cachePaths.ts`)

Cache paths are derived from the working directory:
```typescript
const sanitized = cwd.replace(/[^a-zA-Z0-9]/g, '-')
// Long paths get hashed for stability
if (sanitized.length > 200) {
  return `${sanitized.slice(0, 200)}-${djb2Hash(cwd).toString(36)}`
}
```

DJB2 hashing ensures stable cache paths across upgrades.

---

## Startup Profiling (`src/utils/startupProfiler.ts`)

### Two Profiling Modes

| Mode | Sampling Rate | Output |
|------|--------------|--------|
| Sampled logging | 100% internal, 0.5% external | Phase durations to analytics |
| Detailed profiling | `CLAUDE_CODE_PROFILE_STARTUP=1` | Full report with memory snapshots |

### Key Checkpoints

```
cli_entry
  → cli_before_main_import
  → cli_after_main_import
  → init_function_start
  → init_configs_enabled
  → init_safe_env_vars_applied
  → init_after_graceful_shutdown
  → init_after_1p_event_logging
  → init_network_configured
  → init_function_end
  → main_after_run
```

Phase durations computed from checkpoint pairs:
- `import_time`: `cli_entry` → `main_tsx_imports_loaded`
- `init_time`: `init_function_start` → `init_function_end`
- `settings_time`: `eagerLoadSettings_start` → `eagerLoadSettings_end`
- `total_time`: `cli_entry` → `main_after_run`

The 0.5% external sampling rate provides real-world performance data without impacting most users.

---

## Immutable Config Snapshot

At query loop entry, configuration is snapshotted once:

```typescript
const config = buildQueryConfig()  // Snapshot once

type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

This prevents per-iteration feature gate lookups during the query loop, where `getFeatureValue_CACHED_MAY_BE_STALE` would otherwise be called on every token.

---

## Key Design Principles

1. **Measure before optimizing**: Startup profiling runs on real users (sampled) and provides data-driven performance insights.

2. **Parallel by default**: Independent initialization tasks run concurrently. Sequential only when there's a genuine dependency (e.g., mTLS before preconnect).

3. **Lazy by default, eager when measured**: Modules load on first use unless profiling data shows the eager path is faster.

4. **Preconnect overlaps with computation**: The TCP+TLS handshake happens while the application finishes initializing. By the time the first API call fires, the connection is warm.

5. **Cap defaults, escalate on failure**: Output tokens default to 8K (covering 99% of cases) rather than 64K. The 1% that need more get it via escalation retry.

6. **Snapshot hot-path config**: Configuration used in tight loops is read once and frozen, avoiding repeated function calls.
