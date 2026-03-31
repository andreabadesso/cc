# Development Practices & Feature Flag Architecture

An analysis of Claude Code's development methodology, feature flag system, and release engineering practices, inferred from source code exploration.

---

## Trunk-Based Development

Claude Code follows **trunk-based development**. All evidence points to a single `main` branch with no long-lived feature branches, no gitflow, and no release branches. The canonical indicators are all present:

| Indicator | Evidence |
|-----------|----------|
| Single main branch | All commits land on `main` directly |
| No release branches | Versions managed at build time via `MACRO.VERSION` |
| Feature flags everywhere | GrowthBook (runtime) + Bun bundle flags (compile-time) |
| Gradual rollouts | User attribute targeting, percentage-based bucketing |
| Automated migrations | 11 idempotent migration files in `src/migrations/` |
| Instant rollback | Feature flags can be toggled server-side without a deploy |

The key enabler is a **two-layer feature flag system** that decouples deployment from release. Code ships to `main` continuously; whether users see a feature is controlled independently through flags.

---

## Feature Flag System

### Layer 1: Compile-Time Flags (Bun Bundle)

Bun's `bun:bundle` feature system provides **dead code elimination** at build time. Inactive code paths are completely stripped from the binary.

```typescript
import { feature } from 'bun:bundle'

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [CronCreateTool, CronDeleteTool, CronListTool]
  : []
```

**Known compile-time flags:**

| Flag | Purpose |
|------|---------|
| `PROACTIVE` | Proactive agent mode |
| `KAIROS` | High-context multi-turn support |
| `BRIDGE_MODE` | IDE integration (VS Code, JetBrains) |
| `DAEMON` | Background daemon mode |
| `VOICE_MODE` | Voice input |
| `AGENT_TRIGGERS` | Scheduled/cron agent execution |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook integration |
| `WORKFLOW_SCRIPTS` | User-defined workflows |
| `FORK_SUBAGENT` | Fork-based subagent spawning |
| `UDS_INBOX` | Unix domain socket peer messaging |
| `MONITOR_TOOL` | System monitoring capabilities |
| `TERMINAL_PANEL` | Terminal capture tool |
| `WEB_BROWSER_TOOL` | Browser automation |
| `COORDINATOR_MODE` | Multi-agent coordinator |
| `HISTORY_SNIP` | Conversation history snipping |
| `TRANSCRIPT_CLASSIFIER` | Auto-mode ML classifier |
| `BASH_CLASSIFIER` | Bash permission ML classifier |

These flags create **different build variants**. The internal (`ant`) build likely enables all flags, while external builds enable a subset.

### Layer 2: Runtime Flags (GrowthBook)

GrowthBook provides **server-evaluated, remotely configurable** feature flags. This is the primary mechanism for gradual rollouts, A/B testing, and killswitches.

**Core implementation** lives in `src/services/analytics/growthbook.ts` (~1,155 lines).

#### API Functions

| Function | Behavior | Use Case |
|----------|----------|----------|
| `getFeatureValue_CACHED_MAY_BE_STALE(key, default)` | Non-blocking, disk-cached | Startup-critical paths |
| `getDynamicConfig_CACHED_MAY_BE_STALE(key, default)` | Returns object configs | Complex feature configuration |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gate)` | Legacy compatibility | Statsig-to-GrowthBook migration |
| `getFeatureValue_DEPRECATED(key, default)` | Async, blocks on init | Deprecated; avoid |

The `_CACHED_MAY_BE_STALE` suffix is a deliberate API design choice: it forces callers to acknowledge that the value might be from a previous session's cache, encouraging correct handling of stale data.

#### Naming Convention

All runtime feature flags use the `tengu_` prefix:

| Feature Key | Purpose |
|------------|---------|
| `tengu_amber_flint` | Agent teams/swarms killswitch |
| `tengu_malort_pedway` | Computer Use configuration |
| `tengu_scratch` | Scratchpad mode gate |
| `tengu_frond_boric` | Analytics sink killswitch |
| `tengu_auto_mode_config` | Auto-mode model allowlist |
| `tengu_tool_pear` | Structured outputs / strict tools |
| `tengu_amber_json_tools` | Token-efficient JSON tool format |
| `tengu_slate_prism` | Connector text summarization |
| `tengu_bridge_poll_interval_config` | IDE bridge keepalive tuning |
| `tengu_log_datadog_events` | Datadog analytics control |

#### Caching Architecture

GrowthBook values flow through a multi-level cache:

```
1. ENV OVERRIDES (highest priority, ant-only)
   CLAUDE_INTERNAL_FC_OVERRIDES='{"feature": true}'

2. LOCAL CONFIG OVERRIDES (ant-only, via /config Gates tab)
   ~/.claude/settings.json → growthBookOverrides

3. IN-MEMORY CACHE
   remoteEvalFeatureValues Map (refreshed periodically)

4. DISK CACHE (survives restarts)
   ~/.claude/settings.json → cachedGrowthBookFeatures

5. DEFAULT VALUES (code-level fallbacks)
```

The refresh cycle:
- **Init**: Remote eval with 5s timeout
- **Periodic**: Auto-refresh every 20 min / 6 hours
- **Auth change**: Client recreated on login (fresh targeting attributes)
- **Every refresh**: Disk cache synced

#### User Targeting Attributes

Features are evaluated against a rich set of user attributes:

```typescript
type GrowthBookUserAttributes = {
  id: string                    // Device ID
  sessionId: string             // Current session
  deviceID: string              // Device identifier
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string       // Enterprise proxy detection
  organizationUUID?: string     // Organization
  accountUUID?: string          // Account
  userType?: string             // 'ant' for internal
  subscriptionType?: string     // 'max', 'pro', 'team', 'enterprise'
  rateLimitTier?: string        // Rate limit bucket
  firstTokenTime?: number       // User tenure
  email?: string                // Email
  appVersion?: string           // Client version
  github?: GitHubActionsMetadata
}
```

This enables targeting by platform, subscription tier, organization, user tenure, and more.

#### Experiment Tracking

GrowthBook supports A/B testing with exposure deduplication:

```typescript
// Each feature can be backed by an experiment
{
  experimentId: string      // Experiment tracking key
  variationId: number       // 0 = control, 1+ = variants
  inExperiment?: boolean
  hashAttribute?: string    // Bucketing attribute
  hashValue?: string        // Bucketing value
}
```

Exposures are logged once per session per feature (deduplication via `loggedExposures` Set), enabling experiment analysis without double-counting.

---

## Feature Gating Patterns

### Pattern 1: Internal-First with Killswitch

Features are often enabled unconditionally for internal users (`ant`), with a GrowthBook killswitch controlling external access:

```typescript
// src/utils/agentSwarmsEnabled.ts
export function isAgentSwarmsEnabled(): boolean {
  if (process.env.USER_TYPE === 'ant') return true

  if (!isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS)
      && !isAgentTeamsFlagSet()) {
    return false
  }

  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_flint', true)) {
    return false
  }
  return true
}
```

### Pattern 2: Dynamic Configuration Objects

Complex features use object-valued flags for runtime-tunable configuration:

```typescript
// src/utils/computerUse/gates.ts
function readConfig(): ChicagoConfig {
  return {
    ...DEFAULTS,
    ...getDynamicConfig_CACHED_MAY_BE_STALE<Partial<ChicagoConfig>>(
      'tengu_malort_pedway', DEFAULTS
    ),
  }
}
// Config includes: enabled, pixelValidation, clipboardPasteMultiline, etc.
```

### Pattern 3: Killswitch Maps

A single flag can control multiple subsystems:

```typescript
// src/services/analytics/sinkKillswitch.ts
export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<SinkName, boolean>>
  >('tengu_frond_boric', {})
  return config?.[sink] === true
}
```

### Pattern 4: Subscription-Tier Gating

Features can require specific subscription levels:

```typescript
function hasRequiredSubscription(): boolean {
  if (process.env.USER_TYPE === 'ant') return true
  const tier = getSubscriptionType()
  return tier === 'max' || tier === 'pro'
}
```

---

## Migration System

The `src/migrations/` directory contains 11 idempotent migration files that run on startup. These handle transitions that would otherwise be breaking changes in trunk-based development:

- `migrateOpusToOpus1m` - Model name transitions (gated by feature flag)
- `migrateSonnet45ToSonnet46` - Model aliasing updates
- `migrateFennecToOpus` - Model deprecation handling
- `migrateAutoUpdatesToSettings` - Setting location changes
- `resetAutoModeOptInForDefaultOffer` - Default behavior changes
- `migrateChangelogFromConfig` - Storage format migration

Each migration follows the same pattern:
1. **Conditional** - Gated by eligibility check or feature flag
2. **Idempotent** - Safe to run repeatedly (checks current state before acting)
3. **Logged** - Emits analytics events for tracking
4. **Non-blocking** - Failures are caught silently

This is a textbook trunk-based development pattern: instead of requiring users to upgrade through specific versions, migrations run on every startup and only apply when relevant.

---

## Configuration Hierarchy

Settings are resolved through a layered system (highest priority first):

```
1. Managed settings (enterprise policy)
     managed-settings.json + managed-settings.d/*.json (drop-in, sorted alphabetically)
2. Remote managed settings (GrowthBook-managed)
3. Project settings (.claude/settings.json)
4. User settings (~/.config/claude/settings.json)
5. MDM settings (Mobile Device Management, enterprise)
6. Defaults
```

This design supports:
- Enterprise policy enforcement (layers 1, 5 override user choices)
- Remote configuration without deploys (layer 2)
- Per-project customization (layer 3)
- User preferences (layer 4)

---

## Release Engineering

### Versioning

Versions are stamped at build time via `MACRO.VERSION`. There are no release branches; the changelog is a markdown file fetched from GitHub:

```
~/.claude/cache/changelog.md
Format: ## X.X.X - YYYY-MM-DD
        - Bullet point items
```

Release notes are shown to users when their version changes, with:
- Background fetch (500ms timeout, non-blocking)
- Disk-cached fallback
- Maximum 5 recent versions shown
- Internal builds use bundled `MACRO.VERSION_CHANGELOG`

### User Type Differentiation

Two primary build variants:

| Build | `USER_TYPE` | Extra Features |
|-------|------------|----------------|
| Internal | `ant` | All compile-time flags, REPLTool, config overrides, GrowthBook Gates UI |
| External | (unset) | Subset of flags, feature-gated access |

Internal users get all features immediately. External users get features through gradual GrowthBook rollouts. This "dogfooding" approach means internal users serve as the first line of validation before broader rollout.

---

## Non-Blocking Initialization

Every subsystem is designed to fail gracefully, a necessity for trunk-based development where the remote configuration state might change at any time:

```typescript
// Parallelized at startup
void Promise.all([
  import('../services/analytics/firstPartyEventLogger.js'),
  import('../services/analytics/growthbook.js'),
]).then(([fp, gb]) => {
  fp.initialize1PEventLogging()
  gb.onGrowthBookRefresh(() => {
    void fp.reinitialize1PEventLoggingIfConfigChanged()
  })
})
```

Heavy modules (OpenTelemetry ~400KB, gRPC ~700KB) are lazy-loaded. Network calls have timeouts and fallbacks. Disk caches provide last-known-good state. The application always starts, even if every remote service is unreachable.

---

## Key Takeaways

1. **Deployment != Release**: Code deploys continuously to `main`. Feature visibility is controlled independently through flags. This is the defining characteristic of trunk-based development.

2. **Two-layer flags are intentional**: Compile-time flags reduce binary size and eliminate dead code. Runtime flags enable gradual rollouts without rebuilding. Each layer serves a distinct purpose.

3. **The `_CACHED_MAY_BE_STALE` naming is brilliant API design**: It forces every caller to acknowledge staleness, preventing bugs where code assumes fresh values. The cache hierarchy (env > local override > memory > disk > default) ensures the application always has a value.

4. **Migrations replace version-gated upgrades**: Instead of "if version < X, do Y", migrations run idempotently on every startup. This eliminates the need to track which version a user upgraded from.

5. **Internal-first rollout reduces risk**: Every feature is validated by `ant` users before reaching external users. This built-in dogfooding stage catches issues before they reach customers.

6. **Everything fails open**: Remote services, feature flags, analytics - all have timeouts, fallbacks, and cached defaults. The application always works, even offline. This resilience is essential when feature flags are fetched remotely.
