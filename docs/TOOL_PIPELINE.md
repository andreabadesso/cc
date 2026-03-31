# Tool Assembly Pipeline & Permission System Deep Dive

This document covers the tool type system, assembly pipeline, execution flow, permission system, and deferred tool loading.

---

## 1. Tool Type System

### The `Tool<Input, Output, P>` Interface

**File:** `src/Tool.ts` (lines 362–695)

Three type parameters:
- `Input extends AnyObject` — Zod schema type for input
- `Output` — return type from `call()`
- `P extends ToolProgressData` — progress event type

#### Key Fields

| Field | Purpose |
|---|---|
| `name: string` | Primary identifier. Used for lookups, permission rules, API calls |
| `aliases?: string[]` | Backwards compatibility (e.g., `KillShell` → `TaskStop`) |
| `searchHint?: string` | 3–10 word capability phrase for `ToolSearch` keyword matching |
| `inputSchema: Input` | Zod schema — parsed before every `call()` |
| `inputJSONSchema?: ToolInputJSONSchema` | For MCP tools providing raw JSON Schema |
| `maxResultSizeChars: number` | Exceeded → result persisted to disk, model gets reference. `Infinity` for self-bounding tools |
| `shouldDefer?: boolean` | Built-in tool deferred behind `ToolSearch` |
| `alwaysLoad?: boolean` | Overrides `isMcp` deferral. Set via `_meta['anthropic/alwaysLoad']` |
| `isMcp?: boolean` | MCP-derived (always deferred unless `alwaysLoad`) |
| `strict?: boolean` | Strict mode for Anthropic API |
| `mcpInfo?: { serverName, toolName }` | Raw unnormalized names for permission matching |

#### Permission Methods

```typescript
// Tool-specific permission logic (default: defer to general system)
checkPermissions(input, context): Promise<PermissionResult>

// Expensive pattern parsing done once per hook-input pair
preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
```

### `ToolResult<T>` (lines 321–336)

```typescript
type ToolResult<T> = {
  data: T                    // Actual output
  newMessages?: (...)[]      // Additional messages to inject
  contextModifier?: (ctx: ToolUseContext) => ToolUseContext  // Mutates context
  mcpMeta?: { _meta?, structuredContent? }  // MCP passthrough
}
```

### `ToolUseContext` (lines 158–300)

Runtime context for every tool call:
- `options.tools: Tools` — complete tool pool the model sees
- `options.mcpClients: MCPServerConnection[]` — live MCP connections
- `abortController: AbortController` — shared cancellation signal
- `getAppState() / setAppState()` — global app state
- `localDenialTracking?: DenialTrackingState` — for async subagents with no-op `setAppState`
- `toolDecisions?: Map<string, {source, decision, timestamp}>` — audit log per turn
- `contentReplacementState?: ContentReplacementState` — budget for tool result compression

### `buildTool()` and `TOOL_DEFAULTS` (lines 757–792)

All tools go through `buildTool()` with fail-closed defaults:

| Default | Value |
|---|---|
| `isEnabled` | `true` |
| `isConcurrencySafe` | `false` (conservative) |
| `isReadOnly` | `false` (assume writes) |
| `isDestructive` | `false` |
| `checkPermissions` | `{ behavior: 'allow', updatedInput: input }` (defer to general system) |
| `toAutoClassifierInput` | `''` (skip classifier) |
| `userFacingName` | `name` |

---

## 2. Tool Assembly

**File:** `src/tools.ts`

### `getAllBaseTools()` (lines 193–251)

Source of truth for ALL tools. Conditional inclusions based on:

- **Embedded search tools** — omits `GlobTool`/`GrepTool` in ant-native builds with `bfs`/`ugrep`
- **`USER_TYPE === 'ant'`** — enables `ConfigTool`, `TungstenTool`, `REPLTool`, `SuggestBackgroundPRTool`
- **Feature flags** — `PROACTIVE`/`KAIROS` → `SleepTool`, `AGENT_TRIGGERS` → cron tools, `WEB_BROWSER_TOOL` → `WebBrowserTool`, `COORDINATOR_MODE` → coordinator tools, etc.
- **Environment variables** — `CLAUDE_CODE_VERIFY_PLAN`, `ENABLE_LSP_TOOL`, `NODE_ENV === 'test'`
- **Utility gates** — `isTodoV2Enabled()`, `isWorktreeModeEnabled()`, `isAgentSwarmsEnabled()`, etc.
- **Lazy `require()`** — `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool` loaded via `require()` to break circular imports

Note: Must stay in sync with Statsig dynamic config for prompt cache stability.

### `getTools(permissionContext)` (lines 271–327)

Filters `getAllBaseTools()` for what the model should see:

1. **`CLAUDE_CODE_SIMPLE` mode** — minimal set: `BashTool`, `FileReadTool`, `FileEditTool`
2. **Special tools exclusion** — removes `ListMcpResourcesTool`, `ReadMcpResourceTool`, `SYNTHETIC_OUTPUT_TOOL_NAME`
3. **`filterToolsByDenyRules()`** — strips blanket-denied tools before model sees them
4. **REPL mode filtering** — hides `REPL_ONLY_TOOLS` when `REPLTool` present
5. **`isEnabled()` check** — each tool's predicate evaluated

### `assembleToolPool(permissionContext, mcpTools)` (lines 345–367)

**Single source of truth** for combining built-in + MCP tools:

```typescript
const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

Design decisions:
- **Two-partition sort** — built-ins sorted among themselves, MCP tools among themselves, groups kept as contiguous prefix + suffix. Preserves server-side prompt cache breakpoint.
- **`uniqBy('name')`** — built-ins first → built-in always beats MCP tool with same name
- **MCP deny-rule filtering** — `filterToolsByDenyRules(mcpTools, permCtx)` applied before merge

### `getMergedTools(permissionContext, mcpTools)` (lines 383–389)

Simpler variant: `[...builtInTools, ...mcpTools]` without sorting or dedup. Used for token counting and `isToolSearchEnabled` threshold checks. **Not for serving to the model.**

### `filterToolsByDenyRules()` (lines 262–269)

Calls `getDenyRuleForTool(permCtx, tool)` for each tool. Applied to both built-ins (in `getTools()`) and MCP tools (in `assembleToolPool()`). Blanket deny rules like `mcp__server1` strip all tools from that server **before the model sees them**.

---

## 3. Tool Execution Pipeline

### Concurrency Model

**File:** `src/services/tools/toolOrchestration.ts`

Entry point: `runTools()` (line 19). Partitions tool calls into batches via `partitionToolCalls()`:

```
[read, read, bash, read, read]
 └── batch 1: [read, read] → concurrent
 └── batch 2: [bash]       → serial
 └── batch 3: [read, read] → concurrent
```

`isConcurrencySafe` from `tool.isConcurrencySafe(parsedInput.data)`. Default: `false`.

Concurrent batches run via `all()` (bounded parallel generator) with `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (default: 10). Serial batches run sequentially with context mutation applied after each.

Context mutations (`contextModifier` from `ToolResult`) only applied for serial tools — concurrent tools cannot safely mutate shared context.

### `runToolUse()` (lines 337–490)

1. Looks up tool in `toolUseContext.options.tools` by name
2. Falls back to `getAllBaseTools()` for deprecated alias lookups
3. If not found: emits `tool_result` error
4. Checks `abortController.signal.aborted` → `CANCEL_MESSAGE`
5. Delegates to `streamedCheckPermissionsAndCallTool()`

### `checkPermissionsAndCallTool()` (lines 599–end)

The core execution function. Steps in order:

**Step 1 — Input validation:**
- `tool.inputSchema.safeParse(input)` — Zod validation
- On failure: `buildSchemaNotSentHint()` detects if model tried calling deferred tool without loading schema (tells model to call `ToolSearch` first)
- `tool.validateInput?.(parsedInput.data, context)` — business logic validation

**Step 2 — Input backfilling:**
- `tool.backfillObservableInput?.(clone)` — shallow clone mutated for hooks/canUseTool. Original `callInput` preserved for `tool.call()` to avoid altering transcript hashes.

**Step 3 — Speculative classifier start (Bash only):**
- `startSpeculativeClassifierCheck()` — kicks off allow-classifier in parallel before permission prompt, reducing latency

**Step 4 — Pre-tool hooks:**
- `runPreToolUseHooks()` — async generator yielding:
  - `'message'` — hook output messages
  - `'hookPermissionResult'` — hook made permission decision
  - `'hookUpdatedInput'` — hook mutated input
  - `'preventContinuation'` / `'stopReason'` — hook stopped tool
  - `'stop'` — hard stop, emits `ToolResultStop`

**Step 5 — Permission resolution:**
- `resolveHookPermissionDecision()` — resolves hook result, then calls `canUseTool` if needed
- `canUseTool` is the `hasPermissionsToUseTool` function

**Step 6 — If denied:**
- Emits error `tool_result` with `is_error: true`
- Runs `PermissionDenied` hooks for auto-mode classifier denials

**Step 7 — Tool execution:**
- `processedInput` vs `callInput` reconciliation
- `tool.call(callInput, context, canUseTool, assistantMessage, onProgress)`

**Step 8 — Post-tool hooks:**
- `runPostToolUseHooks()` — can modify MCP tool output (`updatedMCPToolOutput`)
- For non-MCP tools: `addToolResult()` called **before** hooks
- For MCP tools: `addToolResult()` called **after** hooks (allows reshaping output)

**Step 9 — Result storage:**
- `processToolResultBlock()` checks `tool.maxResultSizeChars`. If exceeded: persisted to temp file, replaced with reference.

---

## 4. Permission System

### Permission Modes

**Files:** `src/utils/permissions/PermissionMode.ts`, `src/types/permissions.ts`

```
ExternalPermissionMode: 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'
InternalPermissionMode: ExternalPermissionMode | 'auto' | 'bubble'
```

| Mode | Behavior |
|---|---|
| `default` | Ask user for any non-whitelisted action |
| `acceptEdits` | Auto-allow file edits in working directory; ask for Bash/network |
| `plan` | Tool calls not executed, planning only |
| `bypassPermissions` | Skip all checks (except `requiresUserInteraction` and `safetyCheck`) |
| `dontAsk` | Any `'ask'` → `'deny'` |
| `auto` | AI classifier auto-allows/denies (ant-only, `TRANSCRIPT_CLASSIFIER` flag) |
| `bubble` | Propagates up through subagents |

### Permission Rules

```typescript
type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string      // e.g. 'Bash', 'mcp__server1', 'mcp__server1__tool1'
    ruleContent?: string  // e.g. 'npm publish:*' for Bash(npm publish:*)
  }
}
```

Sources: `'userSettings'`, `'projectSettings'`, `'localSettings'`, `'flagSettings'`, `'policySettings'`, `'cliArg'`, `'command'`, `'session'`.

Rules with no `ruleContent` match the entire tool. Rules with `ruleContent` are tool-specific (BashTool matches command prefixes; AgentTool matches agent types).

### `ToolPermissionContext`

Central permission context (`DeepImmutable`):
- `mode: PermissionMode`
- `additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>`
- `alwaysAllowRules: ToolPermissionRulesBySource`
- `alwaysDenyRules: ToolPermissionRulesBySource`
- `alwaysAskRules: ToolPermissionRulesBySource`
- `isBypassPermissionsModeAvailable: boolean`
- `shouldAvoidPermissionPrompts?: boolean` — headless/background agents
- `strippedDangerousRules?: ToolPermissionRulesBySource`

### The Permission Decision Ladder

**File:** `src/utils/permissions/permissions.ts`

`hasPermissionsToUseToolInner()` (lines 1158–1319). Steps evaluated in order, early-return on first decisive result:

| Step | Check | Bypass-immune? |
|---|---|---|
| 1a | Deny rule for entire tool | Yes |
| 1b | Ask rule for entire tool | Yes (except Bash + sandbox) |
| 1c | Tool's own `checkPermissions()` | — |
| 1d | Tool denied by `checkPermissions` | Yes |
| 1e | `requiresUserInteraction()` | Yes |
| 1f | Content-specific ask rules from `checkPermissions` (type=rule, behavior=ask) | Yes |
| 1g | Safety checks (`.claude/`, `.git/`, shell configs) | Yes |
| 2a | `bypassPermissions` mode (or plan + bypass available) | — |
| 2b | Always-allow rule | — |
| 3 | `passthrough` → `ask` | — |

After `hasPermissionsToUseToolInner()`, the outer `hasPermissionsToUseTool` (`CanUseToolFn`) applies mode-level transforms:
- `dontAsk`: `'ask'` → `'deny'`
- `auto` mode: acceptEdits fast-path → safe-tool allowlist → `classifyYoloAction()` API call
- `shouldAvoidPermissionPrompts`: headless agents auto-deny after `PermissionRequest` hooks

### MCP vs Built-in Permission Differences

MCP tools go through the same ladder, but:

1. **`tool.checkPermissions()`** — MCP tools use the default (always allow). No custom logic.
2. **Permission rule matching** — `getToolNameForPermissionCheck(tool)` returns full `mcp__server__tool` name even in no-prefix mode
3. **Server-level rules** — `toolMatchesRule()` recognizes `mcp__server1` (no tool part) as matching all tools from that server. `mcp__server1__*` also supported.
4. **Assembly-time filtering** — `filterToolsByDenyRules()` strips denied MCP tools before model sees them
5. **Auth errors** — `McpAuthError` triggers elicitation flows, not permission dialog

---

## 5. Deferred Tools / ToolSearch

### Why Deferred Tools Exist

With large MCP tool sets, sending every tool's full JSON Schema on every API call is token-expensive and may overflow context. Deferred tools send only names; the model calls `ToolSearch` to load full schemas.

### `isDeferredTool(tool)` (prompt.ts:62–108)

Decision order:
1. `alwaysLoad === true` → never defer (MCP opt-out via `_meta['anthropic/alwaysLoad']`)
2. `isMcp === true` → always defer
3. `name === TOOL_SEARCH_TOOL_NAME` → never defer
4. `FORK_SUBAGENT` + `AgentTool` → not deferred (needed turn 1)
5. `KAIROS`/`KAIROS_BRIEF` + `BriefTool` → not deferred
6. `KAIROS` + `SendUserFileTool` + REPL bridge → not deferred
7. `shouldDefer === true` → deferred

### Search Algorithm (`searchToolsWithKeywords()`, lines 186–302)

Scored keyword search over deferred tools.

**Pre-processing:**
- `+term` prefix → required (must appear in name/description)
- Undecorated → optional
- `compileTermPatterns()` pre-compiles word-boundary regexes

**Fast paths:**
1. Exact name match → immediate return
2. `mcp__` prefix match → all tools with that prefix

**Required-term pre-filter:**
- Must match ALL required terms in name parts, description, or `searchHint`

**Scoring:**

| Signal | Score |
|---|---|
| Exact part match (MCP) | +12 |
| Exact part match (built-in) | +10 |
| Partial part match (MCP) | +6 |
| Partial part match (built-in) | +5 |
| Full-name fallback | +3 |
| `searchHint` word-boundary match | +4 |
| Description word-boundary match | +2 |

MCP tools score higher because models often search by server name.

### Result Format

`mapToolResultToToolResultBlockParam()` (lines 444–470):
```json
{
  "type": "tool_result",
  "content": [
    { "type": "tool_reference", "tool_name": "mcp__slack__send_message" },
    { "type": "tool_reference", "tool_name": "mcp__slack__list_channels" }
  ]
}
```

The `tool_reference` type causes the API to inject full JSON Schema, making tools callable.

### Query Forms

- `select:Read,Edit,Grep` — direct name lookup, falls back to full set
- `+slack send` — require "slack" in name, rank by "send"
- `notebook jupyter` — keyword search

### Cache

Description lookups memoized by tool name via `getToolDescriptionMemoized`. Invalidated when deferred tool name set changes (tracked via sorted join string `cachedDeferredToolNames`).

---

## Architecture Notes

### Design Strengths

1. **`buildTool()` / `TOOL_DEFAULTS`** — fail-closed defaults with explicit overrides. `BuiltTool<D>` mapped type preserves precise arity at compile time.
2. **Two-partition sort in `assembleToolPool()`** — built-in tools form a stable prefix the server can cache.
3. **Permission ladder step numbering** — bypass-immune steps (1e, 1f, 1g) prevent escalation.
4. **`localDenialTracking`** — elegant solution for subagents with no-op `setAppState`.

### Design Tensions

1. **`getMergedTools()` vs `assembleToolPool()`** — different purposes but easy to confuse. `getMergedTools()` does NOT sort/dedup.
2. **`backfilledClone` / `callInput` / `processedInput` triple** — complex but necessary: hooks, permissions, and transcript need different input views.
3. **MCP tool result timing** — hooks run before `addToolResult` for MCP, after for built-ins. Different ordering due to hook ability to reshape MCP output.
