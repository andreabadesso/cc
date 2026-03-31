# Skills System Deep Dive

This document covers the full skills system: the SkillTool, bundled skills, disk-based skills, MCP skills, the command type hierarchy, and how skills interact with permissions.

---

## Architecture Overview

Skills are reusable composable workflows that invoke the Claude model with a prompt and optional tool restrictions. They can be invoked inline (within the current conversation) or forked (in a sub-agent).

```
User/Model references skill name
  ↓
SkillTool.call({skillName, args})
  ↓
getAllCommands()                           [SkillTool.ts:81]
  - getCommands(cwd)                      [commands.ts:476]
    - bundledSkills                        [bundledSkills.ts]
    - builtinPluginSkills
    - skillDirCommands                     [loadSkillsDir.ts]
    - workflowCommands
    - pluginCommands
    - COMMANDS()
  - appState.mcp.commands (loadedFrom === 'mcp')
  ↓
findCommand(name, commands)
  ↓
Load prompt + parse frontmatter
  ↓
┌─────────────────────────────────────┐
│ context === 'fork'?                 │
│   YES → runAgent() (sub-agent)      │
│   NO  → inject prompt as messages   │
└─────────────────────────────────────┘
  ↓
Render results
```

---

## 1. SkillTool Implementation

### Core Files

| File | Purpose |
|---|---|
| `src/tools/SkillTool/SkillTool.ts` | Full tool implementation |
| `src/tools/SkillTool/prompt.ts` | System prompt, budget calculation |
| `src/tools/SkillTool/UI.tsx` | All render functions |
| `src/tools/SkillTool/constants.ts` | `SKILL_TOOL_NAME = 'Skill'` |

### Invocation Flow

The `SkillTool` is built with `buildTool()` (line 331). Three phases:

#### Phase 1: `validateInput()` (lines 354–430)

1. Strips leading `/` if present (compatibility with slash-command syntax)
2. For `EXPERIMENTAL_SKILL_SEARCH`: checks for `_canonical_` prefix → skip local lookup
3. Calls `getAllCommands(context)` to get local + MCP skills
4. Calls `findCommand(normalizedCommandName, commands)`
5. Checks `foundCommand.disableModelInvocation` → errorCode 4
6. Checks `foundCommand.type !== 'prompt'` → errorCode 5

#### Phase 2: `checkPermissions()` (lines 432–578)

See [Permission Handling](#permission-handling) below.

#### Phase 3: `call()` (lines 580–869)

Three execution paths:

**Path A — Remote Canonical Skill** (`_canonical_` prefix, ant-only)
- Fetches SKILL.md from AKI/GCS with local cache
- Injects directly as user message, no `$ARGUMENTS` interpolation

**Path B — Forked Skill** (`command.context === 'fork'`)
```typescript
// SkillTool.ts:622
if (command?.type === 'prompt' && command.context === 'fork') {
  return executeForkedSkill(...)
}
```

Inside `executeForkedSkill()` (line 122–288):
1. Creates `agentId` via `createAgentId()`
2. `prepareForkedCommandContext()` sets up sub-agent: clones AppState, runs `getPromptForCommand`
3. Merges `command.effort` if defined
4. Runs `runAgent({ agentDefinition, promptMessages, ... })` as async loop
5. Emits progress via `onProgress`
6. `extractResultText(agentMessages)` pulls final result
7. `clearInvokedSkillsForAgent(agentId)` releases state
8. Returns `{ data: { success, commandName, status: 'forked', agentId, result } }`

**Path C — Inline Skill** (default)
```typescript
// SkillTool.ts:635
const { processPromptSlashCommand } = await import(
  'src/utils/processUserInput/processSlashCommand.js'
)
```

After `processPromptSlashCommand` returns:
1. Extracts `allowedTools`, `model`, `effort`
2. Gets `toolUseID` from parent `AssistantMessage`
3. Filters out `<command-message>` tagged messages
4. `tagMessagesWithToolUseID(newMessages, toolUseID)` marks messages as transient
5. Returns `{ data: { success, commandName, allowedTools, model }, newMessages, contextModifier }`

The `contextModifier` function modifies `ToolUseContext` for subsequent turns:
- Non-empty `allowedTools` → merged into `appState.toolPermissionContext.alwaysAllowRules.command` (line 794)
- `model` set → `resolveSkillModelOverride()` preserves context window suffix (e.g., `opus` on `opus[1m]` becomes `opus[1m]`)
- `effort` set → updates `appState.effortValue`

### Forked vs Inline Comparison

| Aspect | Inline | Fork |
|---|---|---|
| Trigger | `context` undefined or `'inline'` | `context === 'fork'` |
| Execution | Prompt injected as messages into main conversation | Sub-agent with isolated context window |
| User visibility | Messages appear inline | Only final result returned; sub-agent steps shown as progress |
| allowedTools | Applied via `contextModifier` to ongoing conversation | Applied only within sub-agent's `modifiedGetAppState` |
| Result | `{ success, commandName, allowedTools, model }` | `{ success, commandName, status: 'forked', agentId, result }` |

### Prompt Injection

Inside `getMessagesForPromptSlashCommand()` (processSlashCommand.tsx, lines 900–916), the message sequence:

1. **Metadata message** — visible user message with `<command-message>`, `<command-name>`, `<skill-format>` XML tags
2. **Skill content message** — actual prompt from `getPromptForCommand(args, context)`, marked `isMeta: true` (model-visible but hidden in UI)
3. **Attachment messages** — `@`-mentions or MCP resources referenced in SKILL.md body
4. **Permissions attachment** — `type: 'command_permissions'` with `additionalAllowedTools`

Before building messages, two side effects:
- `registerSkillHooks()` — registers any hooks from frontmatter
- `addInvokedSkill(name, path, content, agentId)` — records skill in session state for compaction preservation

### Rendering (UI.tsx)

| Function | When | What |
|---|---|---|
| `renderToolUseMessage` (line 47) | "Claude is deciding" phase | Skill name with optional `/<name>` prefix |
| `renderToolUseProgressMessage` (line 62) | During forked execution | Last 3 progress messages (all in verbose mode) |
| `renderToolResultMessage` (line 20) | After completion | `Done` for forked; `Successfully loaded skill` + tool/model count for inline |
| `renderToolUseRejectedMessage` (line 94) | On rejection | Progress + fallback rejection message |
| `renderToolUseErrorMessage` (line 111) | On error | Progress + fallback error message |

---

## 2. Bundled Skills

### Registration

**File:** `src/skills/bundledSkills.ts`

#### `BundledSkillDefinition` Type (lines 15–41)

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>
  getPromptForCommand: (args: string, context: ToolUseContext)
    => Promise<ContentBlockParam[]>
}
```

Key fields:
- `files` — reference files extracted to disk on first invocation (TOCTOU-safe: `O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW`, mode `0o600`)
- `isEnabled` — lazy per-invocation callback (e.g., checking feature flags at call time)
- `disableModelInvocation: true` — user-only, invisible to model

#### `registerBundledSkill()` Internals (lines 53–100)

1. If `files` defined: wraps `getPromptForCommand` in lazy-extraction closure. Extraction memoized at process level using a promise (concurrent callers await same write). `extractBundledSkillFiles` writes with `safeWriteFile`, catches errors gracefully.
2. Constructs `Command` with `type: 'prompt'`, `source: 'bundled'`, `loadedFrom: 'bundled'`, `progressMessage: 'running'`, `isHidden: !(userInvocable ?? true)`, `contentLength: 0`
3. Pushes to module-level `bundledSkills: Command[]` array

#### `initBundledSkills()` (bundled/index.ts:24–79)

Called at startup. Registers unconditionally: `update-config`, `keybindings-help`, `verify`, `debug`, `lorem-ipsum`, `skillify`, `remember`, `simplify`, `batch`, `stuck`.

Feature-gated (lazy `require`): `dream` (KAIROS), `hunter` (REVIEW_ARTIFACT), `loop` (AGENT_TRIGGERS), `schedule-remote-agents` (AGENT_TRIGGERS_REMOTE), `claude-api` (BUILDING_CLAUDE_APPS), `claude-in-chrome` (conditional), `run-skill-generator` (RUN_SKILL_GENERATOR).

### Bundled Skill Reference

| Skill | File | Description |
|---|---|---|
| `update-config` | `updateConfig.ts` | Configure settings.json. `allowedTools: ['Read']`. Embeds full `SettingsSchema` JSON at invocation time |
| `keybindings-help` | `keybindings.ts` | Keyboard shortcut help. `userInvocable: false` (model-only). Generates reference tables dynamically |
| `simplify` | `simplify.ts` | Launches 3 parallel agents (Code Reuse, Code Quality, Efficiency). Available to all users |
| `batch` | `batch.ts` | `disableModelInvocation: true`. Requires git repo. Orchestrates 5–30 parallel worktree agents. 3-phase (research, spawn workers, track progress) |
| `debug` | `debug.ts` | `disableModelInvocation: true`. Enables debug logging, tail-reads last 64KB of log |
| `loop` | `loop.ts` | `AGENT_TRIGGERS` flag. Schedules recurring prompts via cron. Parses `[interval] <prompt>` |
| `skillify` | `skillify.ts` | `disableModelInvocation: true`. Captures session into SKILL.md. Uses `AskUserQuestion` for interactive design |
| `remember` | `remember.ts` | Reviews memory layers (auto-memory, CLAUDE.md, CLAUDE.local.md). Proposes promotions without applying |
| `verify` | `verify.ts` | Uses `files` (SKILL_FILES). Parses frontmatter from embedded SKILL_MD |
| `stuck` | `stuck.ts` | Diagnoses frozen sessions. Uses `ps` to detect high-CPU/D-state/zombie processes |

---

## 3. Disk-based Skills

### Directory Walking

**File:** `src/skills/loadSkillsDir.ts`

`getSkillDirCommands()` (memoized by cwd, line 638–803) loads from these sources in parallel:

| Priority | Source | Notes |
|---|---|---|
| Highest | Managed | `{managedFilePath}/.claude/skills/`. Blocked by `CLAUDE_CODE_DISABLE_POLICY_SKILLS` env |
| | User | `~/.claude/skills/`. Blocked if `isSettingSourceEnabled('userSettings')` false |
| | Project | `getProjectDirsUpToHome('skills', cwd)` — walks up from cwd to home |
| | Additional | `--add-dir` paths via `getAdditionalDirectoriesForClaudeMd()` |
| Lowest | Legacy | `loadSkillsFromCommandsDir(cwd)` — old `.claude/commands/` format |

In `--bare` mode, only `--add-dir` paths loaded.

### File Identity / Deduplication (lines 118–124, 725–763)

```typescript
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)
  } catch {
    return null
  }
}
```

Uses `realpath` (symlink resolution) rather than inode numbers. Handles:
- Duplicate files via symlinks
- Virtual/container/NFS filesystems with inode=0
- ExFAT precision loss

All identities computed in parallel. Dedup is synchronous, order-preserving (first-wins): managed > user > project > additional > legacy.

### Frontmatter Schema

**File:** `loadSkillsDir.ts`, `parseSkillFrontmatterFields()` (lines 185–265)

Only the `skillName/SKILL.md` directory format is supported in `/skills/`; single `.md` files are silently skipped.

| Field | Type | Notes |
|---|---|---|
| `description` | string | Falls back to first line of markdown |
| `name` | string | Display name override |
| `user-invocable` | boolean | Default `true`, via `parseBooleanFrontmatter` |
| `model` | string | `'inherit'` → `undefined`; else via `parseUserSpecifiedModel` |
| `effort` | string/int | Via `parseEffortValue`; logs if invalid |
| `allowed-tools` | string/string[] | Via `parseSlashCommandToolsFromFrontmatter` |
| `argument-hint` | string | Display hint in typeahead |
| `arguments` | string/string[] | Via `parseArgumentNames` |
| `when_to_use` | string | Trigger description for model |
| `version` | string | Skill version |
| `disable-model-invocation` | boolean | Default `false` |
| `hooks` | object | Validated against `HooksSchema()`; skipped if invalid |
| `context` | `'fork'` | Only `'fork'` is meaningful; anything else → `undefined` |
| `agent` | string | Agent type for forked execution |
| `shell` | object | Via `parseShellFrontmatter` |
| `paths` | string/string[] | Glob patterns; `**` = unconditional |

### Conditional Skills (Path-filtered)

Skills with `paths` frontmatter stored in `conditionalSkills` map (line 822–830). Not returned from `getSkillDirCommands` until corresponding files touched. The `activatedConditionalSkillNames` set persists across cache clears within a session.

`discoverSkillDirsForPaths()` (line 861) walks up from file paths to cwd to find nested `.claude/skills/` directories during file operations (Read/Write/Edit) — enables dynamic skill discovery as the model explores.

### `createSkillCommand()` Factory (lines 270–401)

Builds `Command` with `type: 'prompt'`. The `getPromptForCommand` closure:

1. Prepends `Base directory for this skill: ${baseDir}\n\n` if set
2. `substituteArguments(content, args, true, argumentNames)` — replaces `$arg_name` or positional `$1`, `$2`
3. Replaces `${CLAUDE_SKILL_DIR}` with normalized skill directory path
4. Replaces `${CLAUDE_SESSION_ID}` with current session ID
5. For non-MCP skills: `executeShellCommandsInPrompt()` processes inline `!command` or `` ```! ``` `` shell blocks. **MCP skills explicitly excluded** (line 374: `loadedFrom !== 'mcp'`) — untrusted content cannot execute shell commands

---

## 4. MCP Skills

### The Builder Pattern

**File:** `src/skills/mcpSkillBuilders.ts`

A dependency-graph leaf that imports nothing but types. Breaks a circular import cycle: `client.ts → mcpSkills.ts → loadSkillsDir.ts → commands.ts → ... → client.ts`.

1. `loadSkillsDir.ts` at module init calls `registerMCPSkillBuilders({ createSkillCommand, parseSkillFrontmatterFields })` (lines 1083–1086)
2. `mcpSkills.ts` (loaded lazily at runtime) calls `getMCPSkillBuilders()` to get functions without importing `loadSkillsDir.ts`

### How MCP Resources Become Skills

In `client.ts:2344–2356`:
```typescript
const [tools, mcpCommands, mcpSkills, resources] = await Promise.all([
  fetchToolsForClient(client),
  fetchCommandsForClient(client),
  feature('MCP_SKILLS') && supportsResources
    ? fetchMcpSkillsForClient!(client)
    : Promise.resolve([]),
  ...
])
const commands = [...mcpCommands, ...mcpSkills]
```

`fetchMcpSkillsForClient` (gated on `MCP_SKILLS`):
1. Enumerates resources looking for `skill://` URIs
2. Reads content (the SKILL.md text) for each
3. `getMCPSkillBuilders().parseSkillFrontmatterFields()` parses frontmatter
4. `getMCPSkillBuilders().createSkillCommand({ loadedFrom: 'mcp', source: 'mcp' })` builds command

**Security:** Shell execution explicitly disabled for MCP skills (`loadedFrom !== 'mcp'` guard in `createSkillCommand`). MCP skill markdown treated as untrusted content.

### Merging in SkillTool

```typescript
// SkillTool.ts:81-94
const mcpSkills = context.getAppState().mcp.commands.filter(
  cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp',
)
if (mcpSkills.length === 0) return getCommands(getProjectRoot())
const localCommands = await getCommands(getProjectRoot())
return uniqBy([...localCommands, ...mcpSkills], 'name')
```

Local commands win on name collision. Only `loadedFrom === 'mcp'` commands included (not all MCP prompts).

---

## 5. Command Type Hierarchy

**File:** `src/types/command.ts`

```
Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

| Type | `type` | Description | SkillTool? |
|---|---|---|---|
| `PromptCommand` | `'prompt'` | Skills/slash commands that expand to model prompts. Has `getPromptForCommand()` | Yes |
| `LocalCommand` | `'local'` | Runs local TypeScript code, returns text. Example: `/compact` | No |
| `LocalJSXCommand` | `'local-jsx'` | Renders Ink UI components. Example: `/theme` | No |

### `getAllCommands()` vs `getCommands()`

`getAllCommands()` in SkillTool (line 81) is distinct from `getCommands()` in commands.ts. It:
1. Filters `appState.mcp.commands` to `loadedFrom === 'mcp'` prompt commands
2. If none: returns `getCommands(getProjectRoot())` directly
3. Otherwise: `uniqBy([...localCommands, ...mcpSkills], 'name')` — local wins

`getCommands(cwd)` in commands.ts:476 flows through `loadAllCommands(cwd)` (memoized, line 449):
```
bundledSkills → builtinPluginSkills → skillDirCommands
  → workflowCommands → pluginCommands → COMMANDS()
```

After loading: `meetsAvailabilityRequirement` and `isCommandEnabled` checks applied.

### Filtering Functions

**`getSkillToolCommands()`** (commands.ts:563) — what SkillTool lists for the model:
- Must be `type: 'prompt'`
- Must NOT have `disableModelInvocation`
- Must NOT have `source: 'builtin'`
- Must have: `loadedFrom` in `bundled|skills|commands_DEPRECATED`, OR `hasUserSpecifiedDescription`, OR `whenToUse`

**`getSlashCommandToolSkills()`** (commands.ts:586) — the "true skill" subset:
- Must be `type: 'prompt'`
- Must NOT have `source: 'builtin'`
- Must have `hasUserSpecifiedDescription` OR `whenToUse`
- Must have `loadedFrom` in `skills|plugin|bundled`, OR `disableModelInvocation === true`

Key difference: `getSkillToolCommands` includes legacy `commands_DEPRECATED` entries. `getSlashCommandToolSkills` is tighter, used by `getSkillInfo` to count "true skills".

---

## 6. Permission Handling

### The `SAFE_SKILL_PROPERTIES` Allowlist (SkillTool.ts:875–908)

Central concept for auto-allow vs permission-required decisions. Includes all standard `CommandBase` fields and benign `PromptCommand` fields.

Properties **NOT** in the set that trigger permission:
- `hooks` (non-null)
- `allowedTools` (non-empty array)
- `paths` (non-null)

Properties in the safe set: `type`, `progressMessage`, `contentLength`, `argNames`, `model`, `effort`, `source`, `pluginInfo`, `disableNonInteractive`, `skillRoot`, `context`, `agent`, `getPromptForCommand`, `frontmatterKeys`, plus all `CommandBase` fields.

New properties default to requiring permission unless explicitly added to the allowlist.

### `checkPermissions()` Priority (lines 432–578)

1. **Deny rules** — `getRuleByContentsForTool(permCtx, SkillTool, 'deny')`. Supports exact match (`review-pr`) and prefix wildcard (`review:*`). Leading `/` stripped.
2. **Remote canonical skills** — `_canonical_` prefix, auto-granted after deny check
3. **Allow rules** — same rule-matching logic
4. **Safe properties auto-allow** — `skillHasOnlySafeProperties(commandObj)` iterates all keys, checks against `SAFE_SKILL_PROPERTIES`
5. **Default: ask** — returns `{ behavior: 'ask', message, suggestions: [exact, prefix-wildcard] }`

### How `allowedTools` Flow Into Permissions

When a skill declares `allowedTools: ['Bash(npm:*)']`:

1. `processPromptSlashCommand` extracts via `parseToolListFromCLI(command.allowedTools ?? [])`
2. `AttachmentMessage` of type `command_permissions` created
3. **Inline:** `contextModifier` adds to `appState.toolPermissionContext.alwaysAllowRules.command` (line 794–802), using `new Set()` to dedup
4. **Fork:** `prepareForkedCommandContext` creates `modifiedGetAppState` closure injecting tools only within sub-agent

Skill-granted `allowedTools` are valid for conversation duration (inline) or scoped to sub-agent lifetime (fork).
