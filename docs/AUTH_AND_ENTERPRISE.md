# Authentication, Authorization, and Enterprise System

This document covers how Claude Code handles authentication (proving who you are), authorization (what you can do), credential storage, workspace trust, enterprise MDM enforcement, remote managed settings, policy limits, and environment variable security.

---

## Table of Contents

1. [OAuth 2.0 Flow (PKCE)](#oauth-20-flow-pkce)
2. [API Key Management](#api-key-management)
3. [Keychain / Secure Storage](#keychain--secure-storage)
4. [Trust Model (Workspace Trust)](#trust-model-workspace-trust)
5. [Enterprise MDM Settings](#enterprise-mdm-settings)
6. [Remote Managed Settings](#remote-managed-settings)
7. [Policy Limits Enforcement](#policy-limits-enforcement)
8. [Environment Variable Security](#environment-variable-security)
9. [Settings Merge Pipeline](#settings-merge-pipeline)
10. [Lessons for Building Auth in Your Own Agent Product](#lessons-for-building-auth-in-your-own-agent-product)

---

## OAuth 2.0 Flow (PKCE)

**Source:** `src/services/oauth/`

Claude Code implements a standard OAuth 2.0 Authorization Code flow with PKCE (Proof Key for Code Exchange). There are two parallel flows -- automatic (browser redirect) and manual (copy-paste) -- running simultaneously.

### How the Flow Works

1. **Code verifier generation** (`src/services/oauth/crypto.ts`): A 32-byte random `codeVerifier` is generated, then SHA-256 hashed to produce the `codeChallenge`. A random `state` parameter is also generated for CSRF protection.

2. **Local HTTP server** (`src/services/oauth/auth-code-listener.ts`): A temporary `AuthCodeListener` starts an HTTP server on `localhost` with an OS-assigned port. This server listens at `/callback` for the OAuth redirect.

3. **Dual-flow initiation** (`src/services/oauth/index.ts` -- `startOAuthFlow`):
   - A **manual flow URL** is shown to the user (redirects to a platform-hosted callback page where the user copies a code).
   - An **automatic flow URL** is opened in the browser (redirects back to `http://localhost:{port}/callback`).
   - Whichever completes first wins.

4. **Token exchange** (`src/services/oauth/client.ts` -- `exchangeCodeForTokens`): The authorization code, state, and PKCE code verifier are POSTed to the token endpoint to receive `access_token`, `refresh_token`, and `expires_in`.

5. **Profile fetch**: After tokens are obtained, the user's profile is fetched from `/api/oauth/profile` to determine subscription type (`pro`, `max`, `team`, `enterprise`), rate limit tier, and billing type.

### Multiple Providers

The OAuth system supports two authorization endpoints selected at login time:

| Provider | Authorize URL | Purpose |
|----------|--------------|---------|
| Console (Anthropic Platform) | `platform.claude.com/oauth/authorize` | API key creation for Console users |
| Claude.ai | `claude.com/cai/oauth/authorize` | Direct inference for Claude.ai subscribers |

The `loginWithClaudeAi` option selects between them. A custom `CLAUDE_CODE_CUSTOM_OAUTH_URL` is also supported, but only for an allowlist of approved FedStart/PubSec deployments.

### OAuth Scopes

Defined in `src/constants/oauth.ts`:

| Scope | Purpose |
|-------|---------|
| `user:inference` | Claude.ai inference access |
| `user:profile` | Read user profile data |
| `org:create_api_key` | Create API keys via Console |
| `user:sessions:claude_code` | Claude Code session management |
| `user:mcp_servers` | MCP server access |
| `user:file_upload` | File upload capability |

### Token Refresh

`refreshOAuthToken` in `src/services/oauth/client.ts`:

- Tokens are refreshed when they are within 5 minutes of expiration (`isOAuthTokenExpired` uses a 5-minute buffer).
- On refresh, the backend allows scope expansion beyond the initial grant (via `ALLOWED_SCOPE_EXPANSIONS`).
- A performance optimization skips the extra `/api/oauth/profile` round-trip when the profile is already cached in both global config and secure storage. This reportedly cuts ~7M requests/day fleet-wide.
- Refresh failures are logged with analytics events (`tengu_oauth_token_refresh_failure`).

### SDK / Non-Browser Flows

The `skipBrowserOpen` option hands both URLs (manual and automatic) to the caller instead of calling `openBrowser()`. This is used by the SDK control protocol (`claude_authenticate`) where the SDK client owns the display.

---

## API Key Management

**Source:** `src/utils/auth.ts`

Claude Code supports multiple ways to provide an API key, checked in a specific priority order:

### Key Resolution Priority

```
1. ANTHROPIC_API_KEY environment variable (if approved in config)
2. File descriptor (CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR)
3. apiKeyHelper (shell command from settings, executed with caching)
4. macOS Keychain / config file (/login-managed key)
```

In `--bare` mode, only `ANTHROPIC_API_KEY` and `apiKeyHelper` (from `--settings` flag) are consulted. Keychain, config files, and approval lists are skipped.

### apiKeyHelper

The `apiKeyHelper` setting specifies a shell command that outputs an API key. It has:

- **Stale-while-revalidate caching**: Returns the cached key immediately, refreshes in the background.
- **Configurable TTL**: `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` env var (default 5 minutes).
- **Trust gating**: If the apiKeyHelper comes from project settings (not user settings), it requires trust dialog acceptance before execution. This prevents malicious repos from running arbitrary commands.
- **Epoch-based cache invalidation**: Cache clears on settings changes or 401 retries, with epoch tracking to prevent stale concurrent executions from clobbering fresh cache.

### OAuth Token Sources

Beyond API keys, auth tokens can come from:

| Source | How set |
|--------|---------|
| `ANTHROPIC_AUTH_TOKEN` | Environment variable |
| `CLAUDE_CODE_OAUTH_TOKEN` | Environment variable (used by Claude Desktop, SSH) |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | Pipe-based FD (Claude Code Remote) |
| Claude.ai OAuth (keychain) | Via `/login` flow |

### Managed OAuth Contexts

When running under Claude Code Remote (`CLAUDE_CODE_REMOTE`) or Claude Desktop (`CLAUDE_CODE_ENTRYPOINT=claude-desktop`), the user's `~/.claude/settings.json` API key configs are ignored. This prevents a terminal-configured API key from interfering with the managed session's OAuth flow.

---

## Keychain / Secure Storage

**Source:** `src/utils/secureStorage/`

### Platform-Specific Backends

| Platform | Primary | Fallback |
|----------|---------|----------|
| macOS | macOS Keychain (`security` CLI) | Plain-text JSON file |
| Linux | Plain-text JSON file | -- |
| Windows | Plain-text JSON file | -- |

### macOS Keychain Integration

`src/utils/secureStorage/macOsKeychainStorage.ts`:

- **Read**: Calls `security find-generic-password -a "{username}" -w -s "{service}"`.
- **Write**: Uses `security -i` (stdin) to pipe `add-generic-password` commands, avoiding process monitors (like CrowdStrike) from seeing the payload. Data is hex-encoded to prevent escaping issues.
- **Overflow protection**: The `security -i` stdin has a 4096-byte `fgets()` buffer. If the hex payload exceeds this (minus 64 bytes headroom), it falls back to argv arguments.
- **Stale-while-error**: If a keychain read fails but a previous value exists, the stale value is served rather than caching `null`. This prevents transient `security` spawn failures from surfacing as "Not logged in."
- **Locked keychain detection**: `isMacOsKeychainLocked()` checks `security show-keychain-info` (exit code 36 = locked). This is cached for the process lifetime since keychain lock state doesn't change during a CLI session.

### Fallback Storage Strategy

`src/utils/secureStorage/fallbackStorage.ts`:

The `createFallbackStorage` wrapper tries the primary backend first, then falls back:

- **Read**: Primary first; if null, try secondary.
- **Write**: If primary write fails and secondary succeeds, the stale primary entry is deleted to prevent it from shadowing the fresh data.
- **Migration**: When migrating from secondary to primary for the first time, the secondary data is cleaned up.

---

## Trust Model (Workspace Trust)

**Source:** `src/utils/config.ts`, `src/components/TrustDialog/`

### Concept

When Claude Code opens a project directory for the first time, it shows a **Trust Dialog**. This is a security boundary: project-scoped settings (`.claude/settings.json`, environment variables, apiKeyHelper from project config) can contain dangerous configurations. Trust must be established before these are applied.

### How Trust Works

- Trust is stored per-directory in `~/.claude.json` under `projects[path].hasTrustDialogAccepted`.
- Trust is **inherited**: if a parent directory is trusted, all subdirectories are trusted.
- Trust only transitions `false -> true` during a session (never the reverse). Once true, the result is latched and not re-checked.
- `checkHasTrustDialogAccepted()` walks from the current working directory up to the filesystem root, checking each path.

### What Trust Gates

Before trust is established:
- Only **safe** environment variables from project settings are applied (see [Environment Variable Security](#environment-variable-security)).
- Shell-executing settings like `apiKeyHelper` from project settings are blocked.
- User settings (`~/.claude/settings.json`) and policy settings are always applied regardless of trust.

After trust:
- All environment variables from project settings are applied (including dangerous ones like `LD_PRELOAD`, `PATH`).
- apiKeyHelper from project settings can execute.

### Non-Interactive Mode

In non-interactive mode (CI, SDK), trust checks are skipped -- the assumption is that the caller controls the environment.

---

## Enterprise MDM Settings

**Source:** `src/utils/settings/mdm/`

### Overview

Claude Code reads enterprise settings from OS-level MDM (Mobile Device Management) configuration. These are admin-controlled settings that individual users cannot override.

### Platform-Specific Sources

| Platform | Admin Source | User Source |
|----------|-------------|-------------|
| macOS | `/Library/Managed Preferences/[username]/com.anthropic.claudecode.plist` and `/Library/Managed Preferences/com.anthropic.claudecode.plist` | `~/Library/Preferences/com.anthropic.claudecode.plist` (ant builds only) |
| Windows | `HKLM\SOFTWARE\Policies\ClaudeCode` (admin-only) | `HKCU\SOFTWARE\Policies\ClaudeCode` (user-writable, lowest priority) |
| Linux | No MDM equivalent | Uses `/etc/claude-code/managed-settings.json` instead |

### How MDM Settings Are Read

`src/utils/settings/mdm/rawRead.ts`:

1. **Startup**: `startMdmRawRead()` fires at `main.tsx` module evaluation time, spawning subprocess reads in parallel with module loading.
2. **macOS**: Spawns `/usr/bin/plutil -convert json -o - --` for each plist path. Uses `existsSync` as a fast-path to skip the subprocess if the file doesn't exist (saves ~5ms per non-existent path).
3. **Windows**: Spawns `reg query HKLM\SOFTWARE\Policies\ClaudeCode /v Settings` (and HKCU in parallel).
4. **Timeout**: All subprocesses have a 5-second timeout (`MDM_SUBPROCESS_TIMEOUT_MS`).

### First-Source-Wins Policy

`src/utils/settings/mdm/settings.ts`:

Policy settings use "first source wins" -- the highest-priority source that exists provides all policy settings:

```
remote (API) > HKLM/plist (MDM) > managed-settings.json (file) > HKCU (Windows user)
```

### What Enterprises Can Control

MDM settings use the same `SettingsJson` schema as all other settings sources. Enterprises can control:

- **Permissions**: `allow`, `deny`, `ask` rules; `defaultMode` (e.g., force `plan` mode)
- **Environment variables**: Set `ANTHROPIC_BASE_URL` to route through corporate proxy, configure OTEL endpoints
- **Shell settings**: `apiKeyHelper`, `awsAuthRefresh`, `gcpAuthRefresh`, `otelHeadersHelper`
- **Hooks**: Pre/post-execution hooks for bash commands, prompts, etc.
- **Tool restrictions**: Disable specific tools
- **Permission mode**: `disableBypassPermissionsMode` to prevent users from running in unrestricted mode

### File-Based Managed Settings

For Linux (and as a cross-platform alternative), enterprises can deploy:

- `/etc/claude-code/managed-settings.json` -- base managed settings file
- `/etc/claude-code/managed-settings.d/*.json` -- drop-in files, sorted alphabetically, merged on top (systemd-style)

---

## Remote Managed Settings

**Source:** `src/services/remoteManagedSettings/`

### Overview

Enterprise customers (Team and Enterprise tiers) can have settings delivered from the Anthropic API at `{BASE_API_URL}/api/claude_code/settings`. This enables centralized policy management without requiring MDM profile deployment.

### Eligibility

`src/services/remoteManagedSettings/syncCache.ts`:

| User Type | Eligible? |
|-----------|-----------|
| Console users (API key) | Yes (all) |
| OAuth Enterprise/Team | Yes |
| OAuth with null subscriptionType (externally-injected tokens) | Yes (API decides) |
| OAuth Pro/Max | No |
| Third-party providers (Bedrock/Vertex/Foundry) | No |
| Custom base URL | No |
| Cowork (`local-agent` entrypoint) | No |

### Fetch and Caching Strategy

1. **Cache-first**: On startup, if a cached `~/.claude/remote-settings.json` exists, it's applied immediately and waiters are unblocked.
2. **ETag-based HTTP caching**: The client computes a SHA-256 checksum of the cached settings (matching the server's Python `json.dumps(sort_keys=True, separators=(",", ":"))`) and sends it as `If-None-Match`. A 304 means the cache is valid.
3. **Retry with backoff**: Up to 5 retries with exponential backoff on transient failures.
4. **Fail-open**: If the fetch fails and no cache exists, the CLI continues without remote settings. If a stale cache exists, it's used (graceful degradation).
5. **Background polling**: After initial load, polls every 1 hour for changes. Changes trigger a hot-reload via `settingsChangeDetector.notifyChange('policySettings')`.

### Security Check for Dangerous Settings

`src/services/remoteManagedSettings/securityCheck.tsx`:

When remote settings contain "dangerous" settings (shell-executing commands, unsafe env vars, hooks), a blocking security dialog is shown:

- **Dangerous shell settings**: `apiKeyHelper`, `awsAuthRefresh`, `awsCredentialExport`, `gcpAuthRefresh`, `otelHeadersHelper`, `statusLine`
- **Dangerous env vars**: Anything NOT in the `SAFE_ENV_VARS` allowlist (e.g., `ANTHROPIC_BASE_URL`, `HTTP_PROXY`, `LD_PRELOAD`, `NODE_TLS_REJECT_UNAUTHORIZED`)
- **Hooks**: Any hooks configuration

The dialog shows the setting names (not values) and requires explicit user approval. If rejected, `gracefulShutdownSync(1)` exits the process.

The check is skipped:
- If dangerous settings haven't changed from the cached version
- In non-interactive mode

### Auth for Remote Settings

`src/services/remoteManagedSettings/index.ts`:

```
API key users:  x-api-key header
OAuth users:    Authorization: Bearer {access_token} + anthropic-beta header
```

Auth headers are obtained directly (not through `getSettings()`) to avoid circular dependencies during settings loading.

---

## Policy Limits Enforcement

**Source:** `src/services/policyLimits/`

### Overview

Separate from managed settings, policy limits are organization-level restrictions that disable specific CLI features. The API endpoint is `{BASE_API_URL}/api/claude_code/policy_limits`.

### How It Works

The response schema is:

```json
{
  "restrictions": {
    "allow_remote_sessions": { "allowed": false },
    "allow_product_feedback": { "allowed": false }
  }
}
```

- **Absent policy = allowed**: If a policy key is not in the restrictions object, it's implicitly allowed.
- **Fail-open**: If the fetch fails, all policies are considered allowed (no restrictions).
- **HIPAA exception**: Policies in `ESSENTIAL_TRAFFIC_DENY_ON_MISS` (currently `allow_product_feedback`) fail **closed** when essential-traffic-only mode is active and the cache is unavailable. This prevents a cache miss from re-enabling features for regulated orgs.

### Eligibility

Same as remote managed settings for API key users. For OAuth users, stricter: only Team and Enterprise with `user:inference` scope.

### Architecture

Mirrors remote managed settings exactly:
- File cache at `~/.claude/policy-limits.json`
- SHA-256 checksums for ETag-based HTTP caching
- 5 retries with exponential backoff
- 1-hour background polling
- Fail-open with stale cache fallback

---

## Environment Variable Security

**Source:** `src/utils/managedEnv.ts`, `src/utils/managedEnvConstants.ts`

### The Two-Phase Application Model

Environment variables from settings are applied in two phases:

**Phase 1 -- Before trust dialog** (`applySafeConfigEnvironmentVariables`):

1. Global config (`~/.claude.json` env) -- always applied.
2. **Trusted sources** (`userSettings`, `flagSettings`, `policySettings`) -- ALL env vars applied.
3. Remote managed settings eligibility is computed (reads `CLAUDE_CODE_USE_BEDROCK`, `ANTHROPIC_BASE_URL`).
4. Policy settings env vars applied last (highest priority).
5. From the fully-merged settings (including project sources), only **safe** env vars are applied.

**Phase 2 -- After trust dialog** (`applyConfigEnvironmentVariables`):

All env vars from all settings sources are applied, including dangerous ones.

### Safe vs Dangerous Environment Variables

`SAFE_ENV_VARS` is an allowlist of ~80 environment variables that are safe to apply before trust. These are Claude Code-specific settings that don't pose security risks.

**Categories of dangerous env vars (NOT in the safe list):**

| Category | Examples |
|----------|----------|
| Redirect to attacker server | `ANTHROPIC_BASE_URL`, `HTTP_PROXY`, `HTTPS_PROXY`, `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Trust attacker server | `NODE_TLS_REJECT_UNAUTHORIZED`, `NODE_EXTRA_CA_CERTS` |
| Switch to attacker project | `ANTHROPIC_FOUNDRY_RESOURCE`, `ANTHROPIC_API_KEY` |
| Arbitrary code execution | `LD_PRELOAD`, `PATH` |

### Provider-Managed Env Vars

When `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` is set (e.g., by Claude Desktop), provider-routing env vars are stripped from all settings-sourced env. This prevents a user's `~/.claude/settings.json` (configured for terminal CLI with Bedrock) from overriding the host's first-party auth setup.

### SSH Tunnel Protections

When `ANTHROPIC_UNIX_SOCKET` is set (SSH remote mode), auth-related env vars (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_BASE_URL`) are stripped from settings-sourced env. The SSH tunnel proxy manages auth injection.

### CCD Spawn-Env Protections

When running under Claude Desktop, env vars that were present in the original spawn environment are snapshotted on first call. Settings-sourced env vars cannot override these, preventing issues like `OTEL_LOGS_EXPORTER=console` corrupting the stdio JSON-RPC transport.

---

## Settings Merge Pipeline

All settings come together in a layered merge with increasing priority:

```
userSettings          (~/.claude/settings.json)
  |
projectSettings       (.claude/settings.json in project)
  |
localSettings         (.claude/settings.local.json, gitignored)
  |
flagSettings          (--settings CLI flag or SDK inline)
  |
policySettings        (first-source-wins: remote API > MDM > managed file > HKCU)
```

Policy settings have the highest priority and cannot be overridden by any other source. Within policy settings, the first-source-wins rule applies:

```
Remote API settings > MDM plist/HKLM > managed-settings.json + drop-ins > HKCU
```

---

## Lessons for Building Auth in Your Own Agent Product

These patterns from Claude Code's auth system are broadly applicable to CLI-based AI agents:

### 1. Dual-Flow OAuth

Run automatic (browser redirect to localhost) and manual (copy-paste code) flows simultaneously. The first to complete wins. This handles environments where browsers can't redirect to localhost (remote servers, containers, SSH).

### 2. PKCE is Non-Negotiable

CLI apps are public clients (no client secret). PKCE with S256 challenge method is the only secure option. Generate a fresh code verifier per flow.

### 3. Layer Your Credential Sources

Support multiple credential sources with clear priority: env vars > helpers > stored credentials. The `apiKeyHelper` pattern (run a shell command to get credentials) is powerful for corporate environments with custom auth providers.

### 4. Trust Before Execution

Never execute project-provided shell commands (`apiKeyHelper`, hooks) before the user has explicitly trusted the workspace. A malicious `.claude/settings.json` in a cloned repo could otherwise run arbitrary code on `git clone && claude`.

### 5. Separate Safe from Dangerous Settings

Maintain an explicit allowlist of safe environment variables. Everything else is dangerous until proven otherwise. Apply safe variables early; gate dangerous ones on trust.

### 6. Fail Open for Enterprise Features

Remote managed settings and policy limits should fail open (no restrictions on fetch failure). The alternative -- locking users out when the API is down -- is worse. Exception: regulated environments (HIPAA) where specific policies must fail closed.

### 7. Stale-While-Revalidate Everywhere

Cache credentials, settings, and policy data. Serve stale data on transient failures rather than failing entirely. This applies to keychain reads, API key helpers, remote settings, and policy limits.

### 8. Platform-Specific Secure Storage with Fallbacks

Use the OS keychain (macOS) where available, but always have a fallback. Handle keychain lock detection (SSH sessions), overflow limits, and process monitor evasion (hex encoding, stdin piping).

### 9. ETag-Based Caching for Remote Config

Compute checksums locally and use `If-None-Match` for efficient polling. For hourly polls across a large fleet, this avoids unnecessary data transfer and server load.

### 10. MDM for Enterprise Control

Support OS-level MDM (macOS plist, Windows registry) for enterprises that already manage their fleet this way. Use "first source wins" so remote API > local MDM > file-based config, giving admins multiple deployment options without conflicts.
