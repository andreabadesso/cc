# MCP Integration Deep Dive

This document covers the full Model Context Protocol (MCP) integration: configuration, connections, authentication, tool wrapping, output handling, lifecycle management, and supporting utilities.

---

## Architecture Overview

MCP servers provide dynamic tools, commands (skills), and resources discovered at connection time and merged into the tool pool. The integration spans ~22 files under `src/services/mcp/`.

```
Config Load (.mcp.json, settings, plugins, claude.ai)
  ↓
getAllMcpConfigs()                         [config.ts]
  ↓
getMcpToolsCommandsAndResources()         [client.ts:2226]
  ↓
connectToServer(name, config)             [client.ts:595]
  ↓
fetchToolsForClient()   [TOOLS]           [client.ts:1743]
fetchCommandsForClient() [SKILLS]         [client.ts:2033]
fetchResourcesForClient() [RESOURCES]     [client.ts:2000]
  ↓
onConnectionAttempt({client, tools[], commands[], resources})
  ↓
AppState.mcp = {clients[], tools[], commands[], resources{}}
  ↓
assembleToolPool(permissionContext, appState.mcp.tools)   [tools.ts:345]
  ↓
Model sees combined tool list
```

---

## 1. Configuration

### Sources and Priority

**File:** `src/services/mcp/config.ts`

`getClaudeCodeMcpConfigs()` (line 1071) merges in this order (last wins):

| Priority | Source | Location |
|---|---|---|
| Lowest | Plugin servers | Plugin manifests / directories |
| | User servers | `~/.claude/settings.json` → `mcpServers` |
| | Project servers | `.mcp.json` files (closer to CWD wins) |
| Highest | Local servers | Per-project local config |

**Enterprise override:** If `managed-mcp.json` exists, ALL other sources are ignored. Only enterprise servers are returned (after policy filtering).

**Plugin-only lock:** `isRestrictedToPluginOnly('mcp')` suppresses user/project/local servers but keeps plugin servers.

The merge is a simple `Object.assign({}, dedupedPluginServers, userServers, approvedProjectServers, localServers)` (line 1232).

### Project Config Traversal

`getMcpConfigsByScope('project')` (line 908) walks from filesystem root down to CWD, collecting `.mcp.json` at each directory. Processed in reverse (root-first) so **closer files win** via `Object.assign`. This enables per-directory `.mcp.json` overrides.

### Transport Types

**File:** `src/services/mcp/types.ts`

| Transport | Schema | Key Fields |
|---|---|---|
| `stdio` | `McpStdioServerConfigSchema` (line 28) | `command`, `args`, optional `env` |
| `sse` | `McpSSEServerConfigSchema` (line 58) | `url`, optional `headers`, optional `oauth` |
| `http` | `McpHTTPServerConfigSchema` (line 89) | `url`, optional `headers`, optional `oauth` |
| `ws` | `McpWebSocketServerConfigSchema` (line 99) | `url`, optional `headers` |
| `sdk` | `McpSdkServerConfigSchema` (line 108) | `name` |
| `sse-ide` / `ws-ide` | IDE extension transport (internal) | |
| `claudeai-proxy` | `McpClaudeAIProxyServerConfigSchema` (line 116) | `url`, `id` |

### Claude.ai Server Deduplication

`dedupClaudeAiMcpServers()` (line 281) computes `getMcpServerSignature()` for each enabled manual server. Claude.ai connectors whose signature matches a manual server are suppressed. Only **enabled** manual servers count — a disabled manual server must not suppress its connector twin.

`getMcpServerSignature()` (line 202) returns:
- `stdio:${JSON.stringify([cmd, ...args])}` for stdio
- `url:${unwrapCcrProxyUrl(url)}` for remote

`unwrapCcrProxyUrl()` (line 182) strips CCR proxy wrapping by extracting the original `mcp_url` query parameter — lets a plugin's raw vendor URL match a connector's CCR-proxied URL.

### Plugin Server Deduplication

`dedupPluginMcpServers()` (line 223): Plugin server keys are namespaced `plugin:{pluginName}:{serverName}` so they never key-collide with manual servers. Signature-based dedup catches duplicate processes/connections. Manual wins over plugin; between plugins, first-loaded wins. Disabled/policy-blocked plugin servers are split out before dedup to avoid winning the first-plugin-wins race.

### Policy Filtering

`isMcpServerAllowedByPolicy()` (line 417): Combined allowlist/denylist. Denylist has absolute precedence. Allowlist is type-aware:
- If any `serverCommand` entries exist, stdio servers **must** match one
- If any `serverUrl` entries exist, remote servers **must** match one
- Otherwise: name-based matching

URL matching supports wildcards via `urlPatternToRegex()` (line 320): `*` → `.*` after escaping regex metacharacters.

### Environment Variable Expansion

**File:** `src/services/mcp/envExpansion.ts`

`expandEnvVarsInString()` expands `${VAR}` and `$VAR` syntax in server config strings. Used for commands, args, URLs, and headers.

---

## 2. Connection Establishment

### `connectToServer()` (client.ts:595)

Core entry point. **Memoized via lodash `memoize`** with cache key `"${name}-${jsonStringify(serverRef)}"` (line 585). First call does real work; subsequent calls return the cached promise. Cache is cleared in `client.onclose`, so reconnections start fresh.

Handles 9 transport types in a large `if/else if` chain:

| Transport | Behavior |
|---|---|
| `sse` | `SSEClientTransport` with `ClaudeAuthProvider` + step-up detection wrapper |
| `sse-ide` | `SSEClientTransport`, no auth, proxy-only |
| `ws-ide` | WebSocket with `X-Claude-Code-Ide-Authorization` header |
| `ws` | WebSocket with optional session-ingress JWT |
| `http` | `StreamableHTTPClientTransport` with `ClaudeAuthProvider` + step-up detection |
| `sdk` | Throws immediately; handled separately in `print.ts` |
| `claudeai-proxy` | `StreamableHTTPClientTransport` using `createClaudeAiProxyFetch` |
| `stdio` (in-process) | `InProcessTransport` via `createLinkedTransportPair()` (Chrome MCP, Computer Use) |
| `stdio` (default) | `StdioClientTransport`, uses `subprocessEnv()` + server-specific env overrides |

### Timeouts

| Timeout | Source | Default |
|---|---|---|
| Connection | `MCP_TIMEOUT` env var | 30 seconds (line 457) |
| Per-request fetch | `wrapFetchWithTimeout` | 60 seconds (line 492) |
| Tool call | `MCP_TOOL_TIMEOUT` env var | ~27.8 hours (line 211) |

Connection timeout uses `Promise.race([connectPromise, timeoutPromise])`. GET requests are excluded from fetch timeout because they are long-lived SSE streams. The implementation uses `setTimeout` + `clearTimeout` (not `AbortSignal.timeout()`) explicitly to avoid a Bun memory leak of ~2.4KB per request.

### Error Handling

`isTerminalConnectionError()` (line 1249) checks for: `ECONNRESET`, `ETIMEDOUT`, `EPIPE`, `EHOSTUNREACH`, `ECONNREFUSED`, `Body Timeout Error`, `terminated`, `SSE stream disconnected`, `Failed to reconnect SSE stream`.

Three consecutive terminal errors trigger `closeTransportAndRejectPending` (via `consecutiveConnectionErrors` counter with `MAX_ERRORS_BEFORE_RECONNECT = 3`, line 1228).

Session expiry: `isMcpSessionExpiredError()` (line 193) requires HTTP 404 **AND** JSON body `"code":-32001`. Both conditions required to avoid false positives from generic 404s.

**Stdio stderr:** Capped at 64MB to prevent unbounded memory growth (line 973).

**Stdio process cleanup:** 3-stage signal escalation: SIGINT (wait 100ms) → SIGTERM (wait 400ms) → SIGKILL, with 600ms absolute failsafe. Entire cleanup capped at 500ms.

---

## 3. Authentication

### OAuth Flow

**File:** `src/services/mcp/auth.ts`

`ClaudeAuthProvider` implements the MCP SDK's `OAuthClientProvider`. On 401, the PKCE flow runs.

**Discovery order** (line 256):
1. User-configured `authServerMetadataUrl` (must be HTTPS)
2. RFC 9728 → RFC 8414 via `discoverOAuthServerInfo`
3. Fallback: path-aware RFC 8414 probe against the MCP server URL

### Token Storage

Tokens stored in OS secure storage (keychain on macOS), keyed by `sha256(jsonStringify({type, url, headers})).slice(0,16)` prefixed with `serverName` (line 326). Prevents credential reuse across differently-configured servers with the same name.

### Cross-App Access (XAA)

Enterprise SSO path. When `serverConfig.oauth?.xaa` is set, `performCrossAppAccess` is called instead of standard PKCE. IDP discovery via `discoverOidc` and `acquireIdpIdToken`.

### Needs-Auth Caching

15-minute file-backed cache at `~/.claude/mcp-needs-auth-cache.json` (line 261–316). Avoids re-probing servers that returned 401 on every startup. Writes serialized through a `writeChain` promise to prevent concurrent read-modify-write races.

### Claude.ai Proxy Auth

`createClaudeAiProxyFetch()` (line 372): Wraps fetch to attach Claude.ai OAuth bearer token. On 401, calls `handleOAuth401Error(sentToken)` which force-refreshes only if keychain has a newer token. The `sentToken` parameter (passed by value before the request) prevents the ELOCKED race where another connector refreshed concurrently.

### Step-Up Detection

`wrapFetchWithStepUpDetection()`: Wraps innermost fetch so 403 responses are seen before the SDK's auth handler fires.

### Slack Non-Standard Error Normalization

`normalizeOAuthErrorBody()` (line 157): Some OAuth servers return HTTP 200 with `{"error":"invalid_grant"}`. Rewritten to HTTP 400 so the SDK's error mapping fires. Slack-specific codes (`invalid_refresh_token`, `expired_refresh_token`, `token_expired`) normalized to `invalid_grant`.

---

## 4. Elicitation Mechanism

**File:** `src/services/mcp/elicitationHandler.ts`

Elicitation allows MCP servers to request information from the user during a tool call.

### Form Elicitation (`mode: 'form'`)

Server sends `elicitation/create` with JSON Schema (`requestedSchema`) and a message. Client shows a form. Returns `{action: 'accept', content: {...}}`, `{action: 'decline'}`, or `{action: 'cancel'}`.

### URL Elicitation (`mode: 'url'`)

Server needs user to visit a URL (e.g., OAuth consent page). Two sub-flows:

**Normal URL Elicitation** (via `registerElicitationHandler`, line 68): Request arrives via JSON-RPC during connection. Handler queues `ElicitationRequestEvent` into `AppState.elicitation.queue`. UI shows dialog with URL button. Server sends `notifications/elicitation/complete` with `elicitationId` to signal completion.

**Error-Based URL Elicitation** (via `callMCPToolWithUrlElicitationRetry`, line 2813): Tool call returns error code `-32042` (`UrlElicitationRequired`) with `{elicitations: [{mode: 'url', url, elicitationId, message}]}`. Two-phase UI flow:
1. **Consent phase:** User sees "Open URL" button. `respond()` with `{action: 'accept'}` is a no-op.
2. **Waiting phase:** After URL opened, `onWaitingDismiss('retry')` resolves with `{action: 'accept'}` to retry the tool call.
3. Up to 3 retries (`MAX_URL_ELICITATION_RETRIES = 3`).

### Hook Integration

Both request and result can be intercepted by hooks (`runElicitationHooks`, `runElicitationResultHooks`). Hooks can programmatically resolve elicitations without user interaction, or override/block user responses.

Default handler: a cancel-returning fallback registered at connection time (line 1191), replaced by `registerElicitationHandler` in `onConnectionAttempt`.

---

## 5. Tool Discovery & Wrapping

### `getMcpToolsCommandsAndResources()` (client.ts:2226)

1. Loads all configs via `getAllMcpConfigs()`
2. Partitions into disabled and active servers
3. For each active server: `connectToServer(name, config)`
4. On success, parallel fetches: tools, commands, skills (feature-gated), resources
5. Calls `onConnectionAttempt()` to update AppState

### Tool Wrapping (`fetchToolsForClient()`, line 1743)

Each MCP tool is converted to internal `Tool` format (lines 1766–1832):

```typescript
return toolsToProcess.map((tool): Tool => {
  const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
  return {
    ...MCPTool,
    name: skipPrefix ? tool.name : fullyQualifiedName,
    mcpInfo: { serverName: client.name, toolName: tool.name },
    isMcp: true,
    searchHint: tool._meta?.['anthropic/searchHint'],
    alwaysLoad: tool._meta?.['anthropic/alwaysLoad'],
    description: async () => tool.description ?? '',
    isConcurrencySafe: () => tool.annotations?.readOnlyHint ?? false,
    isReadOnly: () => tool.annotations?.readOnlyHint ?? false,
    isDestructive: () => tool.annotations?.destructiveHint ?? false,
    isOpenWorld: () => tool.annotations?.openWorldHint ?? false,
    inputJSONSchema: tool.inputSchema,
    async call(args, context, _canUseTool, parentMessage, onProgress) {
      // Delegates to callMCPToolWithUrlElicitationRetry()
    },
  }
})
```

**Tool naming:** `mcp__<normalizedServerName>__<toolName>` via `buildMcpToolName()` (`src/services/mcp/mcpStringUtils.ts`). SDK servers can override via `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` env var.

### Tool Call Execution (`callMCPTool`, line 3029)

1. Starts progress log interval every 30 seconds
2. Creates timeout promise with `getMcpToolTimeoutMs()`
3. Calls `client.callTool({name, arguments, _meta}, CallToolResultSchema, {signal, timeout, onprogress})`
4. `_meta` carries `{'claudecode/toolUseId': toolUseId}` for correlation
5. On `isError: true`: throws `McpToolCallError`
6. On 401/`UnauthorizedError`: throws `McpAuthError` → client state → `needs-auth`
7. On session expiry (404/-32001 or -32000): clears cache, throws `McpSessionExpiredError`. Outer retry (`MAX_SESSION_RETRIES = 1`) retries with fresh client
8. On abort (`AbortError`): returns `{content: undefined}` silently (line 3237)

---

## 6. Output Handling

### Token Limit

**File:** `src/utils/mcpValidation.ts`

`getMaxMcpOutputTokens()` (line 26) precedence:
1. `MAX_MCP_OUTPUT_TOKENS` env var
2. GrowthBook flag `tengu_satin_quoll.mcp_tool`
3. Default: 25,000 tokens

### Truncation Decision

`mcpContentNeedsTruncation()` (line 151):
1. `getContentSizeEstimate()` — fast heuristic (char-based). If below 50% of limit (`MCP_TOKEN_COUNT_THRESHOLD_FACTOR = 0.5`), skip API call.
2. For borderline content: `countMessagesTokensWithAPI()` — actual token-counting API.

**Image handling:** `IMAGE_TOKEN_ESTIMATE = 1600` tokens (line 18). When image exceeds remaining budget, compression attempted via `compressImageBlock` with `remaining_chars × 0.75` for base64 overhead. If compression fails, image skipped.

### Large Output Persistence (`processMCPResult`, line 2720)

Decision tree when content exceeds token limit:
1. IDE server → skip (IDE tools don't go to model)
2. `ENABLE_MCP_LARGE_OUTPUT_FILES` explicitly falsy → in-place truncation
3. Contains image blocks → in-place truncation (JSON-serialized images non-viewable)
4. Otherwise → `persistToolResult()` writes to `~/.claude/tool-results/mcp-{server}-{tool}-{timestamp}`

Returns `getLargeOutputInstructions(filepath, contentLength, formatDescription)` telling the model to use offset/limit to read 100% of the content.

### Result Type Discrimination (`transformMCPResult`, line 2662)

Three paths:
- `toolResult`: String coercion
- `structuredContent`: JSON serialization with `inferCompactSchema` (depth=2, 10 keys/object cap)
- `content` array: Each item dispatched through `transformResultContent` (text/image/audio/resource/resource_link)

**Binary content** (audio, blobs): `persistBlobToTextBlock` → `persistBinaryContent` writes with mime-derived extension (pdf, csv, docx) and replaces with file-path text block.

---

## 7. Caching & Memoization

Three LRU-backed caches (`memoizeWithLRU` from `src/utils/memoize.ts`), each capped at 20 entries (`MCP_FETCH_CACHE_SIZE = 20`), keyed by server name:

| Cache | Invalidated By |
|---|---|
| `fetchToolsForClient` (line 1743) | `ToolListChangedNotification`, `onclose` |
| `fetchResourcesForClient` (line 2000) | `ResourceListChangedNotification`, `onclose` |
| `fetchCommandsForClient` (line 2033) | `PromptListChangedNotification`, `onclose` |

`connectToServer` uses lodash `memoize` keyed by `getServerCacheKey(name, serverRef)`. Cleared in `client.onclose` (line 1384–1396).

`memoizeWithLRU` uses `lru-cache` underneath with `peek()` for cache reads to avoid updating recency during observation.

---

## 8. Lifecycle Management

**File:** `src/services/mcp/useManageMCPConnections.ts`

### Reconnection Logic (line 88)

| Constant | Value |
|---|---|
| `MAX_RECONNECT_ATTEMPTS` | 5 |
| `INITIAL_BACKOFF_MS` | 1,000 |
| `MAX_BACKOFF_MS` | 30,000 |

Backoff formula: `Math.min(1000 × 2^(attempt-1), 30000)`

Only **remote transports** (SSE, HTTP, WebSocket, claudeai-proxy) get automatic reconnection. Stdio and SDK types immediately go to `type: 'failed'`.

Reconnect timers stored in `reconnectTimersRef.current` (a Map). Allows cancellation when server is disabled mid-retry or new connection triggered. On each retry, state updated to `type: 'pending'` with `reconnectAttempt` and `maxReconnectAttempts` for UI display.

On disconnect: `clearKeychainCache()` called before `clearServerCache` to force fresh credential reads (critical for VS Code extension scenarios where another process may have modified stored tokens).

### Notification Subscriptions (lines 618–751)

Registered in `onConnectionAttempt` when server declares capability:

| Capability | Notification | Action |
|---|---|---|
| `capabilities.tools.listChanged` | `ToolListChangedNotificationSchema` | Invalidate tool cache, re-fetch, update server |
| `capabilities.prompts.listChanged` | `PromptListChangedNotificationSchema` | Invalidate command cache, re-fetch |
| `capabilities.resources.listChanged` | `ResourceListChangedNotificationSchema` | Invalidate all three resource caches simultaneously |

For `MCP_SKILLS` feature: resource list changes also refresh skills and invalidate skill-search index (`clearSkillIndexCache()`).

### Two-Phase Loading (lines 858–1024)

**Phase 1:** `getClaudeCodeMcpConfigs(dynamicMcpConfig, claudeaiPromise)` (fast, file-only + cached plugin data). Kicks off `getMcpToolsCommandsAndResources` without awaiting.

**Phase 2:** Awaits the same `claudeaiPromise` (already in flight, memoized — no second network call). Deduplicates claude.ai connectors against manual servers, starts second `getMcpToolsCommandsAndResources` for claude.ai servers only.

### Batched State Updates (line 207)

16ms flush window (`MCP_BATCH_FLUSH_MS`). Multiple `updateServer` calls coalesced into single `setAppState`. Updates accumulated in `pendingUpdatesRef.current` and flushed together via `setTimeout`.

### Channel Permissions

**File:** `src/services/mcp/channelPermissions.ts`

For MCP channel servers (Telegram, iMessage, Discord) — `notifications/claude/channel/permission` capability allows relaying permission approvals from users.

Short 5-letter ID generated: FNV-1a hash → base-25 with 25-char alphabet (excluding `l`), with blocklist for offensive substrings. Server sends user's `"yes tbxkq"` reply parsed into `{request_id, behavior}`. `ChannelPermissionCallbacks.resolve()` matches to pending Map entry and fires resolver.

Gated by `feature('KAIROS') || feature('KAIROS_CHANNELS')` AND `isChannelPermissionRelayEnabled()`.

---

## 9. AppState

**File:** `src/state/AppStateStore.ts` (lines 173–184)

```typescript
mcp: {
  clients: MCPServerConnection[]
  tools: Tool[]
  commands: Command[]
  resources: Record<string, ServerResource[]>
  pluginReconnectKey: number
}
```

### Connection States (`MCPServerConnection`)

**File:** `src/services/mcp/types.ts` (lines 179–215)

| State | Meaning |
|---|---|
| `'connected'` | Active with Client object and capabilities |
| `'needs-auth'` | OAuth token required (shows auth tool) |
| `'failed'` | Connection error |
| `'pending'` | Attempting connection |
| `'disabled'` | Explicitly disabled in config |

---

## 10. Supporting Files

| File | Purpose |
|---|---|
| `src/services/mcp/normalization.ts` | `normalizeNameForMCP` — safe identifiers for tool name prefixes |
| `src/services/mcp/mcpStringUtils.ts` | `buildMcpToolName`, `getMcpPrefix` |
| `src/services/mcp/channelNotification.ts` | `gateChannelServer` — registration gating (feature flag, session auth, policy, marketplace/channel allowlist) |
| `src/services/mcp/envExpansion.ts` | `expandEnvVarsInString` — `${VAR}` and `$VAR` expansion |
| `src/services/mcp/headersHelper.ts` | `getMcpServerHeaders` — merge static + dynamic headers |
| `src/utils/mcpWebSocketTransport.ts` | Custom WebSocket transport bridging `ws` → MCP SDK Transport |
| `src/services/mcp/InProcessTransport.ts` | `createLinkedTransportPair()` — in-process MCP servers |
| `src/services/mcp/SdkControlTransport.ts` | Routes MCP JSON-RPC through SDK control channel |
| `src/utils/mcpOutputStorage.ts` | Binary persistence, format descriptions, large-output instructions |
| `src/utils/mcpValidation.ts` | Token limit, truncation decision, image compression |
| `src/utils/plugins/mcpPluginIntegration.ts` | Plugin MCP loading, namespace keys, policy filtering |
