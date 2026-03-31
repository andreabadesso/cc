# Feature Flag System

Claude Code uses a two-layer feature flag architecture: **compile-time flags** via Bun's bundler and **runtime flags** via GrowthBook. A third, lighter layer uses **environment variable flags** for deployment-specific behavior. Together they enable safe gradual rollouts, dead code elimination for external builds, and runtime experimentation without redeployment.

---

## Two-Layer Architecture

```
                   Build time                         Runtime
                  ┌──────────────────┐       ┌──────────────────────┐
  Source code --> │  bun:bundle      │ ----> │  GrowthBook          │
                  │  feature('FLAG') │       │  tengu_* flags       │
                  │                  │       │                      │
                  │  Dead code       │       │  Remote eval         │
                  │  elimination     │       │  Disk cache          │
                  │  (tree shaking)  │       │  Periodic refresh    │
                  └──────────────────┘       └──────────────────────┘
                          │                           │
                  Strips entire code          Toggles behavior at
                  paths from binary           runtime per user/org
```

### Layer 1: Compile-Time (Bun)

Flags are imported from `bun:bundle` and evaluated at build time:

```typescript
import { feature } from 'bun:bundle'

// Entire branch is eliminated from external builds if VOICE_MODE is disabled
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice.js').default
  : null
```

Bun's bundler replaces `feature('FLAG')` with a boolean literal. When `false`, the dead code eliminator strips the unreachable branch -- including all transitively imported modules. This is critical for keeping external builds small and preventing internal-only features from leaking.

**Pattern constraint**: `feature()` only works in `if`/ternary conditions. The positive ternary pattern is preferred:

```typescript
// CORRECT: Positive ternary -- string literals eliminated from external builds
return feature('BRIDGE_MODE')
  ? getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
  : false

// INCORRECT: Negative guard -- inline string literals survive in external builds
if (!feature('BRIDGE_MODE')) return false
return getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
```

### Layer 2: Runtime (GrowthBook)

GrowthBook provides server-evaluated feature flags with user targeting. The client is initialized in `src/services/analytics/growthbook.ts`.

```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'

// Returns cached value immediately (non-blocking)
const isEnabled = getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
```

### How the Two Layers Interact

Compile-time flags gate whether runtime flag references even exist in the binary. This prevents GrowthBook flag name strings from appearing in external builds:

```typescript
// voiceModeEnabled.ts
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

When `VOICE_MODE` is disabled at build time, the entire GrowthBook call (including the `tengu_amber_quartz_disabled` string) is stripped. When enabled, the runtime flag provides a kill switch.

### Layer 3: Environment Variables

Environment variables act as deployment-specific flags, checked via `isEnvTruthy()`:

```typescript
import { isEnvTruthy } from '../utils/envUtils.js'

if (isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) { ... }
```

`isEnvTruthy` normalizes `'1'`, `'true'`, `'yes'`, `'on'` (case-insensitive) to `true`.

---

## Complete Catalog of Compile-Time Flags

There are **90 unique compile-time flags** across the codebase (828 total usages in 199 files). The flags span categories from core features to internal tooling.

### Core Agent Capabilities

| Flag | Purpose |
|------|---------|
| `PROACTIVE` | Proactive agent mode (autonomous task suggestions) |
| `KAIROS` | Full Kairos assistant mode (sessions, channels, briefing) |
| `KAIROS_BRIEF` | Briefing feature subset (without full Kairos) |
| `KAIROS_CHANNELS` | MCP channel support for Kairos |
| `KAIROS_DREAM` | Dream/background processing for Kairos |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook subscriptions for Kairos |
| `KAIROS_PUSH_NOTIFICATION` | Push notification settings for Kairos |
| `COORDINATOR_MODE` | Multi-agent coordinator mode |
| `FORK_SUBAGENT` | Fork subagent spawning via `/branch` |
| `UDS_INBOX` | Unix domain socket inbox for peer messaging |
| `ULTRAPLAN` | Ultra-planning mode with extended reasoning |

### Context Management

| Flag | Purpose |
|------|---------|
| `CONTEXT_COLLAPSE` | Aggressive context window management |
| `HISTORY_SNIP` | Conversation history truncation/snipping |
| `REACTIVE_COMPACT` | Reactive compaction triggered by context pressure |
| `CACHED_MICROCOMPACT` | Cache-aware micro-compaction of tool results |
| `PROMPT_CACHE_BREAK_DETECTION` | Detect and log prompt cache breaks |
| `COMPACTION_REMINDERS` | Add reminders during compaction |
| `TOKEN_BUDGET` | User-specified token budget tracking |
| `CONNECTOR_TEXT` | Summarize connector text beta header |
| `BREAK_CACHE_COMMAND` | Command to break prompt cache |

### Tool Ecosystem

| Flag | Purpose |
|------|---------|
| `MONITOR_TOOL` | Background task monitoring tool |
| `WORKFLOW_SCRIPTS` | Workflow/automation script commands |
| `WEB_BROWSER_TOOL` | Web browser tool integration |
| `EXPERIMENTAL_SKILL_SEARCH` | Skill discovery and search tool |
| `MCP_SKILLS` | MCP-based skill integration |
| `OVERFLOW_TEST_TOOL` | Overflow test tool (testing) |
| `TERMINAL_PANEL` | Terminal panel tool |
| `TORCH` | Torch command |
| `REVIEW_ARTIFACT` | Review artifact skill |
| `CHICAGO_MCP` | Chicago MCP server integration |

### Bridge / Remote Control

| Flag | Purpose |
|------|---------|
| `BRIDGE_MODE` | Remote control bridge mode |
| `CCR_AUTO_CONNECT` | Auto-connect to CCR |
| `CCR_MIRROR` | CCR mirror mode |
| `CCR_REMOTE_SETUP` | Remote setup web command |
| `DAEMON` | Daemon mode (background bridge process) |
| `DIRECT_CONNECT` | Direct connection mode |
| `SSH_REMOTE` | SSH remote support |
| `SELF_HOSTED_RUNNER` | Self-hosted environment runner |
| `BYOC_ENVIRONMENT_RUNNER` | Bring-your-own-compute runner |

### Permissions and Safety

| Flag | Purpose |
|------|---------|
| `TRANSCRIPT_CLASSIFIER` | Auto-mode transcript classification |
| `BASH_CLASSIFIER` | Bash command safety classifier |
| `POWERSHELL_AUTO_MODE` | PowerShell auto-mode permissions |
| `TREE_SITTER_BASH` | Tree-sitter bash parsing |
| `TREE_SITTER_BASH_SHADOW` | Shadow-mode tree-sitter bash (comparison testing) |
| `VERIFICATION_AGENT` | Verification agent for plan validation |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Built-in explore/plan agents |
| `UNATTENDED_RETRY` | Automatic retry in unattended mode |

### Memory and Learning

| Flag | Purpose |
|------|---------|
| `EXTRACT_MEMORIES` | Automatic memory extraction |
| `TEAMMEM` | Team memory sync across users |
| `MEMORY_SHAPE_TELEMETRY` | Memory file shape telemetry |
| `AGENT_MEMORY_SNAPSHOT` | Agent memory snapshot for subagents |
| `LODESTONE` | Lodestone memory/navigation system |

### UI and Experience

| Flag | Purpose |
|------|---------|
| `BUDDY` | Companion sprite (animated buddy) |
| `VOICE_MODE` | Voice input/output mode |
| `STREAMLINED_OUTPUT` | Streamlined output formatting |
| `COMMIT_ATTRIBUTION` | Commit attribution tracking |
| `HOOK_PROMPTS` | User hook prompts |
| `MESSAGE_ACTIONS` | Message action buttons |
| `QUICK_SEARCH` | Quick search keybinding |
| `HISTORY_PICKER` | History picker UI |
| `AUTO_THEME` | Automatic theme switching |
| `SHOT_STATS` | Shot/turn statistics display |
| `MCP_RICH_OUTPUT` | Rich MCP tool output |
| `NATIVE_CLIPBOARD_IMAGE` | Native clipboard image paste |

### Background Sessions and Tasks

| Flag | Purpose |
|------|---------|
| `BG_SESSIONS` | Background session support |
| `AGENT_TRIGGERS` | Scheduled agent triggers (cron) |
| `AGENT_TRIGGERS_REMOTE` | Remote agent triggers |
| `AWAY_SUMMARY` | Away summary generation |

### Internal / Build

| Flag | Purpose |
|------|---------|
| `DUMP_SYSTEM_PROMPT` | Dump system prompt to stdout |
| `HARD_FAIL` | Hard fail on errors (development) |
| `SLOW_OPERATION_LOGGING` | Log slow operations |
| `ABLATION_BASELINE` | Ablation baseline mode (testing) |
| `ALLOW_TEST_VERSIONS` | Allow test version downloads |
| `ANTI_DISTILLATION_CC` | Anti-distillation protections |
| `NATIVE_CLIENT_ATTESTATION` | Native client attestation header |
| `ENHANCED_TELEMETRY_BETA` | Enhanced telemetry beta |
| `PERFETTO_TRACING` | Perfetto performance tracing |
| `COWORKER_TYPE_TELEMETRY` | Coworker type telemetry |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | C library detection flags |

### Settings and Configuration

| Flag | Purpose |
|------|---------|
| `DOWNLOAD_USER_SETTINGS` | Download user settings sync |
| `UPLOAD_USER_SETTINGS` | Upload user settings sync |
| `TEMPLATES` | Template/job classification |
| `SKILL_IMPROVEMENT` | Skill improvement survey |
| `NEW_INIT` | New initialization flow |
| `FILE_PERSISTENCE` | File-based persistence |
| `BUILDING_CLAUDE_APPS` | Building Claude apps skill |
| `RUN_SKILL_GENERATOR` | Skill generator |
| `ULTRATHINK` | Extended thinking mode |

---

## Major GrowthBook Flags (tengu_* Patterns)

GrowthBook flags follow a `tengu_` prefix convention. Some are boolean gates; others carry structured JSON configs.

### Feature Gates (Boolean)

| Flag | Purpose |
|------|---------|
| `tengu_ccr_bridge` | Enables Remote Control bridge functionality |
| `tengu_ccr_bridge_multi_session` | Multiple sessions per host:dir in bridge |
| `tengu_harbor` | Enables MCP channels system |
| `tengu_harbor_permissions` | Channel-level permission controls |
| `tengu_scratch` | Coordinator mode eligibility |
| `tengu_amber_quartz_disabled` | Voice mode kill switch (true = voice OFF) |
| `tengu_cobalt_harbor` | Auto-connect all sessions to CCR |
| `tengu_cobalt_lantern` | Remote agent scheduling gate |
| `tengu_surreal_dali` | Advanced remote agent features |
| `tengu_bridge_system_init` | Bridge system initialization behavior |
| `tengu_terminal_panel` | Terminal panel feature gate |
| `tengu_slate_thimble` | Non-interactive memory extraction |
| `tengu_slate_prism` | Controls prompt cache strategy |
| `tengu_tool_pear` | Strict mode for tool schemas |
| `tengu_lodestone_enabled` | Lodestone navigation system |
| `tengu_ccr_mirror` | CCR mirror mode |
| `tengu_ccr_bundle_seed_enabled` | Bundle seeding in CCR |
| `tengu_chrome_auto_enable` | Auto-enable Chrome integration |
| `tengu_desktop_upsell` | Desktop upsell messaging |
| `tengu_disable_bypass_permissions_mode` | Disable bypass permissions mode |
| `tengu_destructive_command_warning` | Warn on destructive git commands |
| `tengu_thinkback` | Thinking feedback feature |
| `tengu_kairos_brief` | Brief mode for Kairos |
| `tengu_willow_mode` | Willow mode behavior |
| `tengu_marble_fox` | Marble fox feature |
| `tengu_hive_evidence` | Evidence collection for hive |

### Dynamic Configs (Structured Values)

| Flag | Purpose |
|------|---------|
| `tengu_bridge_poll_interval_config` | Bridge polling interval configuration |
| `tengu_auto_mode_config` | Auto-mode configuration (enabled/opt-in/disabled) |
| `tengu_ant_model_override` | Model override for internal users |
| `tengu_max_version_config` | Maximum version kill switch |
| `tengu_otk_slot_v1` | Output token escalation configuration |
| `tengu_harbor_ledger` | Channel allowlist configuration |
| `tengu_1p_event_batch_config` | First-party event batching config |
| `tengu_ultraplan_model` | Model selection for ultraplan mode |
| `tengu_cicada_nap_ms` | Timing configuration |

---

## Environment Variable Flags

### CLAUDE_CODE_* Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_COORDINATOR_MODE` | Enable coordinator mode |
| `CLAUDE_CODE_SIMPLE` | Bare/simple mode (reduced features) |
| `CLAUDE_CODE_REMOTE` | Running in remote/CCR environment |
| `CLAUDE_CODE_EAGER_FLUSH` | Eager stream flushing |
| `CLAUDE_CODE_IS_COWORK` | Running in cowork mode |
| `CLAUDE_CODE_USE_CCR_V2` | Use CCR v2 transport (SSE) |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | Use POST for session ingress v2 |
| `CLAUDE_CODE_STREAMLINED_OUTPUT` | Streamlined output mode |
| `CLAUDE_CODE_PROACTIVE` | Enable proactive mode via env |
| `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` | Synchronous plugin installation |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable automatic memory |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Disable CLAUDE.md files |
| `CLAUDE_CODE_DISABLE_POLICY_SKILLS` | Disable policy skills |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Override memory directory path |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | Remote session identifier |
| `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` | Exit after first render (testing) |
| `CLAUDE_CODE_BUBBLEWRAP` | Enable bubblewrap sandboxing |
| `CLAUDE_CODE_VERIFY_PLAN` | Verify plan mode |
| `CLAUDE_CODE_ENTRYPOINT` | Entrypoint identifier for attribution |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | Attribution header control |
| `CLAUDE_CODE_OVERRIDE_DATE` | Override current date |
| `CLAUDE_CODE_SYNTAX_HIGHLIGHT` | Syntax highlighting theme |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Maximum output tokens override |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | Custom OAuth endpoint URL |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | OAuth client ID override |
| `CLAUDE_CODE_WORKER_EPOCH` | Worker epoch for CCR |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session-scoped access token |
| `CLAUDE_CODE_ENVIRONMENT_RUNNER_VERSION` | Environment runner version |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Environment kind (e.g., `bridge`) |
| `CLAUDE_CODE_GB_BASE_URL` | GrowthBook API base URL override |
| `CLAUDE_CODE_PLUGIN_SEED_DIR` | Plugin seed directory for CCR |
| `ENABLE_SESSION_PERSISTENCE` | Enable session persistence |

### Override Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_INTERNAL_FC_OVERRIDES` | JSON map of GrowthBook feature overrides (ant-only) |
| `USER_TYPE` | User type (`ant` for internal, else external) |
| `ANTHROPIC_BASE_URL` | API base URL (used as GrowthBook targeting attribute) |

---

## GrowthBook Initialization and Targeting

### Initialization Flow

1. `getGrowthBookClient()` creates a `GrowthBook` instance with `remoteEval: true`
2. User attributes are sent to the server for server-side evaluation
3. The server returns pre-evaluated feature values
4. Values are cached in memory (`remoteEvalFeatureValues` Map) and synced to disk (`~/.claude.json` under `cachedGrowthBookFeatures`)
5. Periodic refresh runs every **20 minutes** (internal) or **6 hours** (external)

### Targeting Attributes

Defined in `GrowthBookUserAttributes`:

```typescript
type GrowthBookUserAttributes = {
  id: string                    // Device ID (stable across sessions)
  sessionId: string             // Current session ID
  deviceID: string              // Device ID (duplicate for GB conventions)
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string       // Non-default API proxy hostname
  organizationUUID?: string     // Organization identifier
  accountUUID?: string          // User account identifier
  userType?: string             // 'ant' (internal) or undefined
  subscriptionType?: string     // Subscription tier
  rateLimitTier?: string        // Rate limit tier
  firstTokenTime?: number       // First API token timestamp
  email?: string                // User email (for internal targeting)
  appVersion?: string           // App version
  github?: GitHubActionsMetadata // CI metadata for GitHub Actions
}
```

These attributes enable targeting by organization, subscription tier, platform, user type, and more -- supporting percentage-based rollouts bucketed by `deviceID` or `accountUUID`.

### Reader Functions (Priority Order)

| Function | Behavior | Use Case |
|----------|----------|----------|
| `getFeatureValue_CACHED_MAY_BE_STALE()` | Non-blocking. Reads in-memory, falls back to disk cache | Startup-critical paths, sync code |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | Same as above, with Statsig cache fallback | Migration from Statsig |
| `checkGate_CACHED_OR_BLOCKING()` | Returns cached `true` immediately; awaits init if `false`/missing | Entitlement gates (e.g., bridge access) |
| `checkSecurityRestrictionGate()` | Waits for re-init if in progress | Security-critical gates |
| `getFeatureValue_DEPRECATED()` | Blocks on GrowthBook init | Legacy code (being removed) |
| `getDynamicConfig_BLOCKS_ON_INIT()` | Blocks on init, returns structured config | Long-lived config objects |

### Override Precedence

1. **Env var overrides** (`CLAUDE_INTERNAL_FC_OVERRIDES`) -- highest priority, ant-only
2. **Config overrides** (`/config` Gates tab) -- ant-only, runtime adjustable
3. **In-memory payload** (from last GrowthBook refresh)
4. **Disk cache** (`~/.claude.json` `cachedGrowthBookFeatures`)
5. **Default value** (passed by caller)

---

## Flag Latching

Latching prevents mid-session instability by "sticking" a flag value once it first becomes active. The system uses four latches stored in `src/bootstrap/state.ts`:

```typescript
afkModeHeaderLatched: boolean | null   // AFK (auto) mode beta header
fastModeHeaderLatched: boolean | null   // Fast mode beta header
cacheEditingHeaderLatched: boolean | null // Cache editing beta header
thinkingClearLatched: boolean | null    // Thinking clear optimization
```

### How Latching Works

1. Each latch starts as `null` (not yet triggered)
2. When the feature is first activated during a session, the latch flips to `true`
3. Once latched, the value stays `true` for the remainder of the session
4. Latches are **reset on `/clear` and `/compact`** so a fresh conversation context can pick up updated flag values

```typescript
export function clearBetaHeaderLatches(): void {
  STATE.afkModeHeaderLatched = null
  STATE.fastModeHeaderLatched = null
  STATE.cacheEditingHeaderLatched = null
  STATE.thinkingClearLatched = null
}
```

### Why Latching Matters

Without latching, a flag flip mid-session could:
- Change the beta headers sent to the API, invalidating the prompt cache
- Toggle thinking behavior inconsistently within a single reasoning chain
- Switch auto-mode state between tool invocations

The `sticky-on` pattern ensures the API sees consistent headers throughout a conversation turn sequence.

---

## Gradual Rollout and Bucketing

GrowthBook's `remoteEval: true` mode means all targeting logic runs server-side. The client sends user attributes and receives pre-evaluated results.

### Rollout Mechanisms

- **Percentage-based**: GrowthBook hashes a targeting attribute (typically `deviceID` or `accountUUID`) to assign users to buckets. The server controls the percentage.
- **Organization-based**: `organizationUUID` enables per-org rollouts for enterprise customers.
- **User-type gating**: `userType: 'ant'` targets internal users first.
- **Platform-based**: `platform` attribute enables OS-specific rollouts.
- **Version-based**: `appVersion` enables rollouts to specific CLI versions.
- **Subscription-based**: `subscriptionType` gates features by subscription tier.

### Experiment Exposure Logging

GrowthBook experiments are tracked with deduplication:

```typescript
const loggedExposures = new Set<string>()  // Session-level dedup

function logExposureForFeature(feature: string): void {
  if (loggedExposures.has(feature)) return
  // ... log to first-party event system
}
```

Features accessed before GrowthBook init are queued in `pendingExposures` and logged once init completes.

---

## Conditional Prompting

Feature flags directly control which sections appear in the system prompt.

### System Prompt Sections

The system prompt is built from a registry of sections (`src/constants/prompts.ts`), each wrapped in `systemPromptSection()` or `DANGEROUS_uncachedSystemPromptSection()`:

```typescript
// Cached once per session (computed on first use, survives across turns)
systemPromptSection('token_budget', () =>
  'When the user specifies a token target...'
)

// Recomputed every turn (cache-breaking, use sparingly)
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',
)
```

### Compile-Time Gating of Prompt Sections

```typescript
// TOKEN_BUDGET section only exists in builds where TOKEN_BUDGET is enabled
...(feature('TOKEN_BUDGET')
  ? [systemPromptSection('token_budget', () => '...')]
  : []),

// Brief section gated behind KAIROS flags
...(feature('KAIROS') || feature('KAIROS_BRIEF')
  ? [systemPromptSection('brief', () => getBriefSection())]
  : []),
```

### Runtime Gating in Prompt Computation

Inside compute functions, GrowthBook values adjust prompt content:

```typescript
function getAntModelOverrideSection(): string | null {
  if (process.env.USER_TYPE !== 'ant') return null
  return getAntModelOverrideConfig()?.defaultSystemPromptSuffix || null
}
```

### Tool Availability via Flags

Tools themselves are conditionally included based on compile-time flags:

```typescript
// commands.ts -- entire commands gated
const voiceCommand = feature('VOICE_MODE')
const bridge = feature('BRIDGE_MODE')
const forceSnip = feature('HISTORY_SNIP')
const workflowsCmd = feature('WORKFLOW_SCRIPTS')
const forkCmd = feature('FORK_SUBAGENT')
```

Runtime flags further control tool behavior. For example, `tengu_tool_pear` enables strict mode on tool schemas, changing how the API validates tool input.

---

## Architecture Lessons

### 1. Compile-Time Elimination is Essential for Agent Products

With 90+ feature flags, including every code path in every build would bloat the binary and leak internal features. Bun's `feature()` from `bun:bundle` provides zero-cost dead code elimination -- disabled features add literally zero bytes to the binary.

### 2. Non-Blocking Flag Reads Prevent Startup Latency

The `_CACHED_MAY_BE_STALE` pattern reads from disk or memory without awaiting network. GrowthBook init happens in the background. This means the CLI starts instantly, using the last-known flag values, and refreshes asynchronously.

### 3. The Two Layers Serve Different Needs

- **Compile-time**: "Should this code exist in this build?" (binary size, IP protection)
- **Runtime**: "Should this user see this behavior right now?" (rollout, experiments, kill switches)

A runtime flag alone cannot prevent code from being bundled. A compile-time flag alone cannot respond to server-side changes.

### 4. Latching Stabilizes Mid-Session Behavior

Agent sessions are long-lived. A flag flip mid-session could change API parameters (beta headers), break prompt caches, or alter tool availability. Latching ensures consistency within a session while still picking up changes on `/clear` or `/compact`.

### 5. Disk Cache Provides Offline Resilience

GrowthBook values are synced to `~/.claude.json` on every successful refresh. New process starts read from disk immediately, so flags work even when the network is unavailable. The cache is wholesale-replaced (not merged) to ensure deleted flags don't persist as ghost entries.

### 6. Override Hierarchy Supports Development and Testing

Three override layers (env vars > config > server) let developers test specific flag configurations locally without affecting the fleet, while the server remains the source of truth for production.
