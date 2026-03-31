# Application Architecture

This document covers the application layer above the TUI engine — entry points, the query engine, tool system, permission model, state management, and configuration.

## Boot Sequence

### Entry Point (`entrypoints/cli.tsx`)

The CLI entry handles special flags before full initialization:
- Fast-paths for `--version`, `--dump-system-prompt`, and specialized servers (MCP, Chrome native host)
- Uses dynamic imports to minimize module evaluation time
- Delegates to `init.ts` for full system bootstrap

### Initialization (`entrypoints/init.ts`)

Memoized `init()` — runs once per session:

1. Config system validation (`enableConfigs()`)
2. Safe environment variables (pre-trust dialog)
3. Graceful shutdown handlers
4. TLS/mTLS configuration
5. Upstream proxy (for remote/CCR mode)
6. Scratchpad directory
7. Team cleanup registration
8. Telemetry initialization deferred until trust is established

### Startup Optimization

Startup parallelizes independent work for fast boot:

```typescript
// Fired as side-effects before other imports
startMdmRawRead()       // Mobile Device Management settings
startKeychainPrefetch() // Auth token prefetch
// Also: API preconnect, GrowthBook init
```

Heavy modules are deferred:
- OpenTelemetry (~400KB) — loaded on first telemetry event
- gRPC (~700KB) — loaded on first gRPC call

## Query Engine

### QueryEngine (`QueryEngine.ts`)

The core LLM orchestrator. Coordinates:
- System prompt assembly (git context, user context, CLAUDE.md files, memory)
- Message normalization (internal format → API format)
- Streaming requests to the Anthropic API
- Tool execution within the streaming loop
- Context compaction when approaching limits
- Special modes: coordinator, plan, fast-mode
- SDK compatibility layer

### Query Loop (`query.ts`)

The main execution loop:

```
User input
  |
Parse slash commands / check message queue
  |
Fetch system context (git status, repo info)
  |
Fetch user context (CLAUDE.md files, memory)
  |
Prepare attachment messages
  |
QueryEngine.query() — stream tokens from Claude API
  |
Buffer tool uses, execute tools (with permission checks)
  |
Collect tool results, feed back to LLM
  |
Apply auto-compact if needed (multiple strategies)
  |
Yield response to UI, update state, re-render
```

### API Integration (`services/api/claude.ts`)

Handles:
- Message formatting and content block normalization
- Tool schema conversion to API format
- Streaming and non-streaming requests
- Cache scope management (prompt caching)
- Beta features (extended thinking, tool use output)
- Model-specific max token selection
- Usage tracking and cost calculation
- Error handling and retries

## Tool System

### Tool Interface (`Tool.ts`)

Every tool is a self-contained module:

```typescript
interface Tool {
  name: string
  aliases?: string[]
  inputSchema: ZodSchema          // Zod validation
  outputSchema?: ZodSchema
  isEnabled(): boolean
  call(input, context): Promise<Result>
  validateInput(input): ValidationResult
  prompt(context): string         // Description with context
  // Permission requirements
}
```

### ToolUseContext

Rich context passed to every tool execution:

```typescript
type ToolUseContext = {
  getAppState / setAppState       // App state accessors
  fileStateCache                  // LRU cache for file reads
  messages                        // Conversation history
  permissionContext               // Permission rules and mode
  abortController                 // Cancellation
  toolProgress                    // Progress callbacks for UI
  mcpClients                      // MCP server connections
  agentDefinitions                // Available agent types
  thinkingConfig                  // Extended thinking settings
}
```

### Tool Registry (`tools.ts`)

~50 tools organized by category:

| Category | Tools |
|----------|-------|
| File Operations | FileRead, FileEdit, FileWrite, NotebookEdit |
| Execution | Bash, PowerShell, REPL |
| Search | Glob, Grep, ToolSearch, WebSearch, WebFetch |
| Code Intelligence | LSP |
| Multi-Agent | Agent, SendMessage, TeamCreate, TeamDelete |
| Tasks | TaskCreate, TaskUpdate, TaskGet, TaskList |
| Interactive | AskUserQuestion, ExitPlanMode, EnterWorktree |
| Scheduling | CronCreate, CronDelete, CronList |
| Advanced | SkillTool, MCPTool, RemoteTrigger, SyntheticOutput |

## Permission System

### Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Prompt for dangerous tools, allow safe ones |
| `plan` | Require plan approval before execution |
| `acceptEdits` | Auto-allow file edits |
| `bypassPermissions` | Skip all permission checks |
| `dontAsk` | Never prompt (deny if not pre-approved) |
| `auto` | AI classifier decides safe vs. dangerous |

### Permission Flow

```
Tool invocation
  |
Pre-request permission hooks
  |
Check allow/deny rules (settings.json per tool/pattern)
  |
If auto mode: classifier decision
  |
Denial tracking (3 denials → fallback to prompting)
  |
If not resolved: show approval dialog
  |
Post-decision hooks
  |
Execute or deny
```

### Filesystem Permissions

- Working directory restrictions (tools can only access project files)
- Additional allowed directories (configurable)
- Scratchpad directory for cross-worker durable storage
- CLAUDE.md auto-discovery with memory injection

## State Management

### AppState (`state/AppStateStore.ts`)

Central immutable state tree:

- Settings and configuration
- Message history (conversation)
- Background tasks and remote agents
- Permission context and mode
- Tool usage tracking
- Plugin and MCP server state
- Speculation state (for parallelism)
- File history and attribution tracking

### Store (`state/store.ts`)

Zustand-like pattern:
- `setState()` with updater functions
- `subscribe()` for external watchers
- Shallow comparison for performance
- Centralized change handler persists to disk and triggers side effects

## Configuration

### File Locations

| File | Scope | Purpose |
|------|-------|---------|
| `~/.config/claude/settings.json` | User | Global settings |
| `.claude/config.json` | Project | Project-local settings |
| `~/.claude/` | User | Global state, plugins, skills |
| `.claude/` | Project | Session memory, tasks, teams |

### Config Contents

- Permission rules (allow/deny/ask patterns per tool)
- MCP server configurations
- Model preferences
- Theme and UI settings
- Keybindings and hooks
- Memory file locations
- Speed/effort settings

### Remote Managed Settings

Loaded from Anthropic backend after trust dialog. Overrides local settings for policy compliance. Eligibility determined by account type.

### Feature Flags

Dead code elimination via Bun's `bun:bundle`:

```typescript
import { feature } from 'bun:bundle'

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Inactive code is completely stripped at build time. Notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`, `FORK_SUBAGENT`, `COORDINATOR_MODE`.

## Commands & Skills

### Command System (`commands.ts`)

~100 slash commands registered dynamically. Examples: `/commit`, `/review`, `/init`, `/login`, `/mcp`, `/tasks`, `/agents`. Lazy-loaded based on feature gates.

### Skill System (`skills/`)

Reusable composable workflows:

```typescript
type BundledSkillDefinition = {
  name: string
  allowedTools: string[]
  modelOverride?: string
  hooks?: HookConfig[]
  // File extraction on first invocation (for reference docs)
}
```

Skills can be user-invocable (via `/skill-name`) or internal. Users can add custom skills via `.CLAUDE_SKILL` directories.

## Service Layer (`services/`)

| Service | Purpose |
|---------|---------|
| `api/` | Anthropic API client, file API, bootstrap |
| `mcp/` | Model Context Protocol server management |
| `oauth/` | OAuth 2.0 authentication flow |
| `lsp/` | Language Server Protocol integration |
| `analytics/` | GrowthBook feature flags + usage analytics |
| `compact/` | Conversation context compression |
| `policyLimits/` | Organization policy limits enforcement |
| `remoteManagedSettings/` | Remote config from Anthropic backend |
| `extractMemories/` | Automatic memory extraction from conversations |
| `tokenEstimation.ts` | Token count estimation |
| `teamMemorySync/` | Team memory synchronization |

## IDE Bridge (`bridge/`)

Bidirectional communication layer connecting IDE extensions with the CLI:

- `bridgeMain.ts` — bridge main loop
- `bridgeMessaging.ts` — message protocol
- `bridgePermissionCallbacks.ts` — permission callbacks
- `replBridge.ts` — REPL session bridge
- `jwtUtils.ts` — JWT-based authentication
- `sessionRunner.ts` — session execution management

Supports VS Code and JetBrains IDEs.
