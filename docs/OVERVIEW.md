# Claude Code - Source Analysis Overview

This is the leaked/extracted source code of **Claude Code**, Anthropic's official CLI tool for interacting with Claude. It is a production-grade, AI-powered terminal application that orchestrates tool execution, multi-agent workflows, and IDE integration.

## Tech Stack

- **Runtime**: Bun
- **Language**: TypeScript (strict mode)
- **Terminal UI**: React + Ink (React for CLIs)
- **CLI Parsing**: Commander.js
- **Schema Validation**: Zod v4
- **Code Search**: ripgrep (native binary)
- **State Management**: Zustand-like custom store
- **Protocols**: MCP SDK, LSP, OAuth 2.0
- **API**: Anthropic SDK
- **Telemetry**: OpenTelemetry + gRPC
- **Analytics**: GrowthBook (feature flags)
- **Auth**: OAuth 2.0, JWT, macOS Keychain
- **Scale**: ~1,900 TypeScript/TSX files, 512K+ lines of code

## Core Architecture

| Layer | Key Files | Purpose |
|-------|-----------|---------|
| **Entry** | `main.tsx` | CLI parsing, React/Ink init, auth bootstrap |
| **Query Engine** | `QueryEngine.ts`, `query.ts` | Streams LLM calls, manages tool-use loops |
| **Tools** | `src/tools/` (44 tools) | File ops, bash, search, web, agents, tasks |
| **Commands** | `src/commands/` (~88) | Slash commands (`/commit`, `/review`, etc.) |
| **UI** | `src/components/` (~140) | Terminal UI components (messages, inputs, dialogs) |
| **Services** | `src/services/` | API client, MCP, OAuth, LSP, analytics |
| **State** | `src/state/` | Zustand-like store for app state |
| **Multi-agent** | `src/coordinator/` | Sub-agent spawning, teams, task distribution |
| **IDE Bridge** | `src/bridge/` | VS Code & JetBrains integration |
| **Plugins** | `src/plugins/` | Plugin loader, versioning, managed plugins |
| **Skills** | `src/skills/` | Reusable composable skill workflows |
| **Memory** | `src/memdir/` | Persistent file-based memory system |

## Directory Structure

```
src/
  main.tsx                 # Main entry point
  QueryEngine.ts           # Core LLM query engine (~46K lines)
  Tool.ts                  # Tool type system (~29K lines)
  query.ts                 # Query pipeline (~68K lines)
  commands.ts              # Command registry (~25K lines)
  tools.ts                 # Tool registry (~17K lines)
  context.ts               # System/user context collection
  cost-tracker.ts          # Token usage tracking

  commands/                # ~88 command subdirectories
  tools/                   # ~44 tool implementations
  components/              # ~140 Ink/React UI components
  hooks/                   # ~80+ custom React hooks
  services/                # External integrations (API, MCP, OAuth, LSP, etc.)
  entrypoints/             # Initialization and SDK entry points

  state/                   # State management (AppState, store, observers)
  context/                 # React context providers
  coordinator/             # Multi-agent orchestration
  bridge/                  # IDE integration (VS Code, JetBrains)
  plugins/                 # Plugin system (bundled + user)
  skills/                  # Reusable skill workflows
  tasks/                   # Task management system

  utils/                   # 100+ utility modules
  constants/               # Constants and enums
  types/                   # TypeScript type definitions
  schemas/                 # Zod validation schemas
  migrations/              # Config migrations

  memdir/                  # Persistent memory system
  keybindings/             # Vim/modal keybindings
  vim/                     # Vim mode implementation
  voice/                   # Voice input integration
  screens/                 # Full-screen UIs (REPL, Doctor, Resume)
  remote/                  # Remote session handling
  server/                  # Server mode implementation
  ink/                     # Ink renderer wrapper
```

## Key Flow

```
User input
  |
  v
Parse slash commands / check message queue
  |
  v
Fetch system context (git status, repo info)
  |
  v
Fetch user context (CLAUDE.md files, memory)
  |
  v
Prepare attachment messages
  |
  v
QueryEngine.query() -- stream tokens from Claude API
  |
  v
Buffer tool uses, execute tools (with permission checks)
  |
  v
Collect tool results, feed back to LLM
  |
  v
Apply auto-compact if needed
  |
  v
Yield response to UI, update state, re-render
```

## Tool System

Every tool is a self-contained module with an input schema (Zod), permission model, execution logic, and progress state types.

**Key tools (44 total):**

- **File Operations**: `FileReadTool`, `FileWriteTool`, `FileEditTool`, `NotebookEditTool`
- **Code Search**: `GlobTool`, `GrepTool` (ripgrep-based)
- **Execution**: `BashTool`, `PowerShellTool`, `REPLTool`
- **Web**: `WebFetchTool`, `WebSearchTool`
- **AI/Agent**: `AgentTool` (sub-agent spawning), `SkillTool`, `MCPTool`, `LSPTool`
- **Coordination**: `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool`
- **Scheduling**: `CronCreateTool`, `CronDeleteTool`, `CronListTool`
- **Tasks**: `TaskCreateTool`, `TaskUpdateTool`, `TaskGetTool`, `TaskListTool`
- **IDE**: `EnterPlanModeTool`, `ExitPlanModeTool`, `EnterWorktreeTool`, `ExitWorktreeTool`
- **Advanced**: `ToolSearchTool`, `MCPAuthTool`, `RemoteTriggerTool`, `SyntheticOutputTool`

## Command System

~88 user-facing slash commands including:

- **Core**: `/commit`, `/review`, `/compact`, `/config`, `/doctor`
- **Auth**: `/login`, `/logout`
- **Workflow**: `/memory`, `/skills`, `/tasks`, `/vim`, `/help`
- **Integration**: `/mcp` (Model Context Protocol), `/ide`, `/bridge`
- **Tools**: `/cost`, `/diff`, `/context`, `/theme`
- **Advanced**: `/pr_comments`, `/resume`, `/share`, `/teleport`
- **Experimental**: `/proactive`, `/brief`, `/assistant`, `/workflows`

## Permission Model

Tools declare required permissions, and every invocation goes through a permission check:

1. Check permission mode (`default`, `plan`, `auto`, `bypassPermissions`)
2. In `auto` mode: classify tool as safe or dangerous
3. If not pre-approved: show user approval dialog
4. Track execution in analytics

## Multi-Agent Orchestration

```
TeamCreateTool creates a team (shared task list)
  |
  v
Spawn teammates via AgentTool
  |
  v
Assign tasks via TaskCreateTool / TaskUpdateTool
  |
  v
Teammates execute tasks in parallel
  |
  v
SendMessageTool for inter-agent communication
  |
  v
TeamDeleteTool cleanup
```

## Notable Design Patterns

- **Dead code elimination**: Feature flags via `bun:bundle` strip inactive code at build time (flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`, etc.)
- **Lazy module loading**: Heavy modules (OpenTelemetry ~400KB, gRPC ~700KB) deferred via dynamic `import()` until needed
- **Parallel prefetch**: Startup parallelizes MDM settings reads, keychain prefetch, API preconnect, GrowthBook init
- **Memoization & caching**: System context, user context, tool permission context, and settings are cached and invalidated on change
- **Streaming & incremental updates**: Token-by-token streaming from API with incremental tool result collection and render updates on each stream event
- **Auto-compact**: Compresses conversation history when approaching context window limits (multiple strategies: auto, reactive, snip)

## Configuration & State

- `~/.config/claude/settings.json` -- user settings
- `.claude/` -- session memory, tasks, teams, memories (per-project)
- `~/.claude/` -- global state, plugins, skills
- Environment variables with safe/unsafe filtering
- MDM (Mobile Device Management) integration for enterprise
- Remote managed settings (organization-level)
- Policy limits enforcement
