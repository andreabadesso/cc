# Architecture & Type System Patterns

Claude Code's architecture balances extensibility with type safety. The tool system, command system, plugin system, and MCP integration all follow consistent patterns that make the codebase predictable and safe to extend.

---

## Layered Architecture

```
┌────────────────────────────────────────────────────────────┐
│  UI Layer (main.tsx, screens/, components/)                │
│  Ink/React terminal components, permission dialogs         │
├────────────────────────────────────────────────────────────┤
│  Command/Skill/Tool Dispatch (commands.ts, tools.ts)       │
│  Command registry, tool assembly, MCP discovery            │
├────────────────────────────────────────────────────────────┤
│  Query Pipeline (query.ts, QueryEngine.ts, query/*)        │
│  LLM interaction, message normalization, tool execution    │
├────────────────────────────────────────────────────────────┤
│  Tool/Service Layer (tools/*, services/*)                  │
│  Tool implementations, MCP client, analytics               │
├────────────────────────────────────────────────────────────┤
│  Cross-Cutting Concerns (utils/)                           │
│  Permissions, file ops, git, shells, formatting            │
└────────────────────────────────────────────────────────────┘
```

Each layer only depends on layers below it. The permission system is cross-cutting -- it intercepts at the dispatch layer but is implemented in utils.

---

## Tool System

### Tool Interface (`src/Tool.ts`)

Tools are the primary extensibility point. The interface uses generics for type-safe input/output:

```typescript
type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  inputSchema: Input                    // Zod schema
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive(input): boolean
  checkPermissions(input, context): Promise<PermissionResult>
  // ... ~30 optional methods for rendering, validation, classification
}
```

### Builder Pattern

`buildTool()` fills in safe defaults for the ~30 optional methods:

```typescript
const grepTool = buildTool({
  name: 'Grep',
  inputSchema: lazySchema(() => z.strictObject({ pattern: z.string(), ... })),
  call: async (args, context) => { ... },
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

Adding a new tool requires only the mandatory fields. Defaults ensure safe behavior:
- `isEnabled()` → `true`
- `isConcurrencySafe()` → `false` (conservative)
- `isDestructive()` → `false`
- `checkPermissions()` → `{ behavior: 'allow' }`

### Tool Registration (`src/tools.ts`)

Tools are assembled into a pool through several mechanisms:

```typescript
// Static imports (always-enabled tools)
import { BashTool } from './tools/BashTool/BashTool.js'
import { FileEditTool } from './tools/FileEditTool/FileEditTool.js'

// Feature-gated imports (build-time DCE)
const SleepTool = feature('PROACTIVE') ? require('./tools/SleepTool').SleepTool : null

// User-type gated imports
const REPLTool = process.env.USER_TYPE === 'ant' ? require('./tools/REPLTool').REPLTool : null

// Lazy getters (circular dependency breaking)
const getTeamCreateTool = () => require('./tools/TeamCreateTool').TeamCreateTool

// Dynamic tools (MCP servers register at runtime)
```

---

## Command System

### Command Types (`src/types/command.ts`)

Commands use a discriminated union:

```typescript
type Command = PromptCommand | LocalCommand | LocalJSXCommand

type PromptCommand = {
  type: 'prompt'
  name: string
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
  // ... model, allowedTools, hooks, context ('inline'|'fork'), effort
}

type LocalCommand = {
  type: 'local'
  load(): Promise<LocalCommandModule>  // Lazy-loaded
}

type LocalJSXCommand = {
  type: 'local-jsx'
  load(): Promise<LocalJSXCommandModule>  // Lazy-loaded React component
}
```

- `PromptCommand`: Expands to an LLM prompt, executed via `SkillTool`
- `LocalCommand`: Runs JavaScript directly (e.g., `/compact`, `/cost`)
- `LocalJSXCommand`: Renders an interactive React UI (e.g., `/config`, `/doctor`)

### Dynamic Discovery

Commands come from multiple sources:
1. Built-in (hardcoded in `commands.ts`)
2. `~/.claude/skills/` (user-defined)
3. `.claude/skills/` (project-defined)
4. `src/skills/bundled/` (shipped with the app)
5. Plugins (`getPluginCommands()`)
6. MCP servers (dynamic discovery)

The command registry uses a memoized factory `COMMANDS()` to defer config reads to first access.

---

## Skill System

### Markdown + YAML Frontmatter

Skills are markdown files with structured metadata:

```yaml
---
description: "Do something"
whenToUse: "When you need to..."
context: "inline" | "fork"
agent: "general-purpose"
model: "claude-opus-4"
hooks:
  pre_tool_use:
    - matcher: "Bash(git *)"
      hooks:
        - type: command
          command: "pre-commit install"
effort: "light" | "moderate" | "challenging"
paths: ["*.ts", "*.tsx"]
---

# Skill content (prompt text with $ARGUMENTS substitution)
```

### Execution Modes

| Mode | `context` | Behavior |
|------|-----------|----------|
| Inline | `"inline"` | Prompt injected into current conversation |
| Fork | `"fork"` | Sub-agent spawned with isolated context and token budget |

Fork execution uses the `AgentTool` internally, with the skill's `agent` field determining the sub-agent type.

### Hook Registration

Skills can register hooks that are active only during their execution:
- Hooks validated against schema
- Timeout enforcement on hook commands
- Hooks cleaned up after skill completes

---

## Plugin System

### Plugin Manifest

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string              // e.g., "myplugin@github:user/repo"
  enabled?: boolean
  isBuiltin?: boolean
  commandsPaths?: string[]
  skillsPaths?: string[]
  agentsPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
}
```

### Discovery Order
1. Built-in plugins (enable/disable in user settings)
2. Marketplace plugins (`~/.claude/plugins/`)
3. Session plugins (CLI `--plugin-dir`)

### Sandboxing
- **Filesystem**: Plugin paths validated with `validatePathWithinBase()`
- **Network**: MCP servers require explicit config per scope
- **Hooks**: Shell commands run in spawned processes with timeouts
- **LSP/MCP**: Separate server processes with transport isolation

### Error Handling

Plugin errors use a discriminated union with ~14 variants:
```typescript
type PluginError =
  | { type: 'manifest_not_found'; path: string }
  | { type: 'manifest_parse_error'; error: ZodError }
  | { type: 'duplicate_plugin'; existing: string }
  | ...
```

This allows exhaustive handling and clear error messages for each failure mode.

---

## MCP Integration

### Transport Polymorphism (`src/services/mcp/types.ts`)

```typescript
type McpServerConfig =
  | { type: 'stdio'; command: string; args?: string[] }
  | { type: 'sse'; url: string; headers?: Record<string, string> }
  | { type: 'http'; url: string; oauth?: McpOAuthConfig }
  | { type: 'ws'; url: string }
  | { type: 'sdk'; name: string }
  | { type: 'sse-ide'; url: string; ideName: string }
```

### Connection State Machine

```
Pending → Connected   (with capabilities)
Pending → Failed      (with error)
Pending → NeedsAuth   (awaiting OAuth)
Connected → Pending   (auto-reconnect on disconnect)
Any → Disabled        (user action)
```

Each state is a discriminated union variant:
```typescript
type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

### Tool Wrapping (Adapter Pattern)

MCP tools (JSON Schema) are wrapped into the internal `Tool` interface:

```typescript
const tool: Tool = {
  name: mcpTool.name,
  inputSchema: jsonSchemaToZod(mcpTool.inputJSONSchema),  // JSON Schema → Zod
  call: (args) => client.callTool(tool.name, args),
  // ... permission checks, rendering, etc.
}
```

---

## TypeScript Patterns

### Lazy Schema Factory (`src/utils/lazySchema.ts`)

```typescript
function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

Used everywhere Zod schemas reference other schemas. Without it, circular module initialization would fail because schemas are evaluated eagerly during import.

### Discriminated Unions

Pervasive throughout the codebase:

```typescript
// Hooks
type HookCommand =
  | { type: 'command'; command: string }
  | { type: 'prompt'; prompt: string }
  | { type: 'agent'; prompt: string }
  | { type: 'http'; url: string }

// Messages
type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | ToolUseSummaryMessage
  | TombstoneMessage

// MCP connections
type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | PendingMCPServer
  | ...
```

The `type` discriminant enables exhaustive `switch` statements with compile-time completeness checking.

### DeepImmutable

```typescript
type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  // ...
}>
```

Recursive type that makes all properties `readonly` and all arrays `ReadonlyArray`. Catches mutation bugs at compile time.

### Semantic Coercion

```typescript
const semanticBoolean = (schema) => schema.pipe(z.coerce.boolean())
const semanticNumber = (schema) => schema.pipe(z.coerce.number())
```

Used in tool input schemas to accept lenient input from the LLM (strings like `"true"`, `"42"`) while providing typed values to the implementation.

### Cycle Breaking via Type Extraction

When two modules need each other's types:

```typescript
// src/types/permissions.ts (ONLY types, zero runtime dependencies)
export type PermissionResult = { behavior: 'allow' | 'deny' | 'ask' }

// src/Tool.ts (imports type only)
import type { PermissionResult } from './types/permissions.js'

// src/utils/permissions/permissions.ts (imports Tool, implements permissions)
import type { ToolUseContext } from '../Tool.js'
```

The `types/` directory contains only type definitions -- no runtime code. This breaks cycles because TypeScript's `import type` is erased at runtime.

---

## Dependency Injection via Context

### ToolUseContext

Rather than global singletons, tools receive their dependencies through a context object:

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    agentDefinitions: AgentDefinitionsResult
  }
  getAppState(): AppState
  setAppState(f: (prev) => AppState): void
  readFileState: FileStateCache
  abortController: AbortController
  // ... ~30 fields
}
```

Benefits:
- Tools don't import global state modules (no hidden dependencies)
- Testing: inject mock context
- Multi-agent: each agent gets its own context with its own abort controller

### Service Registration Pattern

For cases where lazy initialization is needed across module boundaries:

```typescript
// src/skills/mcpSkillBuilders.ts
let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) throw new Error('Not registered yet')
  return builders
}
```

This breaks the cycle: `client.ts → mcpSkills.ts → loadSkillsDir.ts → ... → client.ts`. Registration happens at module init time.

---

## Configuration Validation

### Multi-Source Resolution

```
Priority (highest to lowest):
1. Managed settings (enterprise policy)
   managed-settings.json + managed-settings.d/*.json (drop-in, sorted alphabetically)
2. Remote settings (GrowthBook-managed)
3. Project settings (.claude/settings.json)
4. User settings (~/.config/claude/settings.json)
5. MDM settings
6. Defaults
```

The drop-in directory pattern (`managed-settings.d/*.json`) follows the systemd/sudoers convention: files are sorted alphabetically, last wins. This allows multiple enterprise teams to manage independent policy files without merge conflicts.

### Zod for Runtime + Static Safety

```typescript
const schema = z.object({ name: z.string(), age: z.number() })
type User = z.infer<typeof schema>  // { name: string; age: number }

// Runtime validation
const parsed = schema.parse(data)        // Throws on invalid
const safe = schema.safeParse(data)      // Returns {success, data, error}
```

A single Zod schema serves as both runtime validator and TypeScript type source. No risk of type definitions drifting from validation logic.

---

## Design Patterns Summary

| Pattern | Where Used | Purpose |
|---------|-----------|---------|
| **Builder** | `buildTool()` | Safe defaults for 30+ optional tool methods |
| **Factory** | Feature-gated `require()` | Dead code elimination at build time |
| **Strategy** | Tool classification methods | Per-invocation security classification |
| **Discriminated Union** | Commands, hooks, messages, MCP connections | Type-safe exhaustive handling |
| **Observer** | Hook system, Signal pattern | Extensible event notification |
| **Adapter** | MCP tool wrapping | JSON Schema → Zod, external → internal interface |
| **Chain of Responsibility** | Permission checks | Hook → classifier → rules → user prompt |
| **State Machine** | MCP connection lifecycle | Explicit transitions between connection states |
| **Context Object** | `ToolUseContext` | Dependency injection without globals |
| **Service Locator** | `registerMCPSkillBuilders` | Lazy cross-module initialization |

---

## Key Design Principles

1. **Type safety at boundaries**: Zod validates all external input (user settings, API responses, MCP messages). Internal code trusts the types.

2. **Lazy by default**: Schemas, modules, and services load on first use. This keeps startup fast and memory low.

3. **Discriminated unions over inheritance**: The codebase uses almost no class hierarchies. Discriminated unions with exhaustive switches provide the same polymorphism with better type safety.

4. **Context over globals**: Dependencies flow through `ToolUseContext`, not global state. This enables multi-agent execution and testability.

5. **Types in separate files**: `src/types/` contains only type definitions. This breaks circular dependencies because `import type` has no runtime effect.

6. **One schema, two purposes**: Zod schemas generate both runtime validation and TypeScript types from a single source of truth.
