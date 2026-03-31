# Plugin and Extensibility Architecture

This document describes the plugin system, custom skills, custom agents, custom commands, and the overall extensibility architecture of the Claude Code CLI. All information is derived from the source code.

---

## Table of Contents

1. [Extensibility Overview](#extensibility-overview)
2. [Plugin Architecture](#plugin-architecture)
3. [Custom Skills](#custom-skills)
4. [Custom Agents](#custom-agents)
5. [Custom Commands (Legacy)](#custom-commands-legacy)
6. [Bundled Skills](#bundled-skills)
7. [Built-in Plugins](#built-in-plugins)
8. [Hook Registration](#hook-registration)
9. [Security Boundaries](#security-boundaries)
10. [Discovery Pipeline](#discovery-pipeline)
11. [Lessons for Building Extensibility](#lessons-for-building-extensibility)

---

## Extensibility Overview

The system has five distinct extensibility layers, each with different trust levels and discovery mechanisms:

| Layer | Location | Trust Level | Format |
|---|---|---|---|
| **Built-in agents** | `src/tools/AgentTool/built-in/` | Highest (compiled in) | TypeScript modules |
| **Bundled skills** | `src/skills/bundled/` | High (compiled in) | TypeScript modules calling `registerBundledSkill()` |
| **Built-in plugins** | `src/plugins/bundled/` | High (compiled in, user-toggleable) | TypeScript calling `registerBuiltinPlugin()` |
| **Marketplace plugins** | `~/.claude/plugins/cache/` | Medium (user-installed) | Git repos with `plugin.json` manifest |
| **Custom skills/agents/commands** | `.claude/skills/`, `.claude/agents/`, `.claude/commands/` | User-controlled | Markdown files with YAML frontmatter |

Source precedence for agents (later overrides earlier): built-in, plugin, userSettings, projectSettings, flagSettings, policySettings. This is implemented in `getActiveAgentsFromList()` in `src/tools/AgentTool/loadAgentsDir.ts`.

---

## Plugin Architecture

### Plugin Manifest (`plugin.json`)

A plugin is a directory with an optional `plugin.json` manifest. The manifest schema (`PluginManifestSchema` in `src/utils/plugins/schemas.ts`) accepts these top-level fields:

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "What the plugin does",
  "author": { "name": "Author Name", "email": "...", "url": "..." },
  "homepage": "https://...",
  "repository": "https://...",
  "license": "MIT",
  "keywords": ["tag1", "tag2"],
  "dependencies": ["other-plugin@marketplace"],

  "commands": "./extra-commands.md",
  "agents": ["./agents/custom.md"],
  "skills": ["./skills/my-skill"],
  "outputStyles": "./styles/",
  "hooks": { "hooks": { "PreToolUse": [...] } },
  "mcpServers": { "server-name": { "command": "...", "args": [...] } },
  "lspServers": { "server-name": { "command": "...", "extensionToLanguage": { ".ts": "typescript" } } },
  "channels": [{ "server": "server-name", "userConfig": {...} }],
  "userConfig": {
    "api_key": { "type": "string", "title": "API Key", "sensitive": true, "required": true }
  },
  "settings": { "agent": {...} }
}
```

### Plugin Directory Structure

```
my-plugin/
├── plugin.json          # Optional manifest with metadata
├── commands/            # Slash commands (*.md files)
│   ├── build.md
│   └── deploy/
│       └── SKILL.md     # Directory-format skill
├── skills/              # Skills (directory format only)
│   └── my-skill/
│       └── SKILL.md
├── agents/              # Custom AI agents (*.md files)
│   └── test-runner.md
├── output-styles/       # Output style customization
├── hooks/               # Hook configurations
│   └── hooks.json       # Hook definitions
└── .mcp.json            # MCP server configuration
```

Paths in the manifest (`commands`, `agents`, `skills`, `hooks`, `mcpServers`, `lspServers`) accept multiple formats:
- A single relative path string (`"./path"`)
- An array of relative paths (`["./path1", "./path2"]`)
- For commands: an object mapping names to metadata (`{ "about": { "source": "./README.md", "description": "..." } }`)
- For hooks/mcpServers: inline definitions or paths to JSON files

### Plugin Loading and Caching

Plugins are loaded by `pluginLoader.ts` (`src/utils/plugins/pluginLoader.ts`). The discovery flow:

1. **Marketplace resolution**: Plugins are referenced as `plugin-name@marketplace-name` in user settings. The marketplace is a Git repository containing a `marketplace.json` index.
2. **Version computation**: `calculatePluginVersion()` computes a version from the marketplace entry (semver, git SHA, or hash).
3. **Cache path**: Plugins are cached at `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`.
4. **Seed directories**: Pre-populated plugin caches can be provided via seed directories (for offline/enterprise installs). Seeds are probed before network fetch.
5. **ZIP cache mode**: An optional mode (`isPluginZipCacheEnabled()`) stores plugins as `.zip` files instead of directories to reduce inode usage.
6. **Session-only plugins**: `--plugin-dir` CLI flag or SDK plugins option loads plugins for the current session only, without marketplace resolution.
7. **Simple mode**: `CLAUDE_CODE_SIMPLE=true` skips all custom/plugin loading, returning only built-in agents.

### Plugin Versioning

From `src/utils/plugins/pluginVersioning.ts` and `pluginLoader.ts`:

- Versioned path format: `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`
- Legacy (unversioned) path: `~/.claude/plugins/cache/{plugin-name}/` (backward compatible)
- Version strings are sanitized to prevent path traversal: `version.replace(/[^a-zA-Z0-9\-_.]/g, '-')`
- Auto-update is enabled by default for official Anthropic marketplaces (except those in `NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES`)

### Plugin Enable/Disable State

```typescript
// From user settings
settings.enabledPlugins: Record<string, boolean>
// e.g. { "my-plugin@my-marketplace": true }
```

Policy (managed settings) can force-disable plugins via `isPluginBlockedByPolicy()` which checks `policySettings.enabledPlugins[pluginId] === false`.

### Plugin Load Result

```typescript
type PluginLoadResult = {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  errors: PluginError[]
}
```

The `LoadedPlugin` type carries all resolved paths and config:

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string              // Filesystem path to plugin root
  source: string            // "name@marketplace" identifier
  repository: string
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string              // Git commit SHA for version pinning
  commandsPath?: string     // Resolved path to commands/ directory
  commandsPaths?: string[]  // Additional command paths from manifest
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

---

## Custom Skills

### Skill Directory Format

Skills use a directory-based format. Each skill is a directory containing a `SKILL.md` file:

```
.claude/skills/
└── my-skill/
    ├── SKILL.md           # Required: skill definition
    ├── schemas/           # Optional: reference files
    └── scripts/           # Optional: bundled scripts
```

The skill name is taken from the directory name. Single `.md` files are NOT supported in the `/skills/` directory (only directory format).

### Skill Frontmatter Format

`SKILL.md` uses YAML frontmatter. Parsed by `parseSkillFrontmatterFields()` in `src/skills/loadSkillsDir.ts`:

```yaml
---
name: my-skill                    # Display name (optional, defaults to directory name)
description: "What this skill does"
when_to_use: "When the user asks to..."   # Guidance for automatic invocation by the model
allowed-tools: Bash, Read, Write          # Tools the skill can use (comma-separated)
argument-hint: "[file] [options]"         # Hint shown to users
arguments: name, file                     # Named arguments ($name, $file in content)
user-invocable: true                      # Whether user can type /skill-name (default: true)
disable-model-invocation: false           # Block model from auto-invoking (default: false)
model: claude-sonnet-4-20250514           # Model override (or "inherit" for parent model)
effort: medium                            # Effort level: low, medium, high, or integer
version: "1.0.0"                          # Skill version
context: fork                             # "fork" runs in a sub-agent, "inline" (default) runs in main context
agent: my-agent                           # Delegate to named agent
paths: src/**, tests/**                   # File path patterns to scope skill visibility
shell: bash                               # Shell for inline commands (bash, zsh, sh, etc.)
hooks:                                    # Hooks that activate when skill runs
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "echo pre-hook"
---

Your skill prompt content goes here.

Use $1 or $name for argument substitution.
Use ${CLAUDE_SKILL_DIR} for the skill's own directory path.
Use ${CLAUDE_SESSION_ID} for the current session ID.

Execute shell commands inline:
!echo "This runs at skill load time"
```

### Skill Discovery Locations

Skills are loaded from multiple directories in parallel by `getSkillDirCommands()`:

| Source | Path | Setting Source |
|---|---|---|
| Managed (enterprise policy) | `{managed_path}/.claude/skills/` | `policySettings` |
| User-level | `~/.claude/skills/` | `userSettings` |
| Project-level (walks up to $HOME) | `.claude/skills/` in each ancestor dir | `projectSettings` |
| Additional directories | `{--add-dir}/.claude/skills/` | `projectSettings` |
| Legacy commands | `.claude/commands/` (all levels) | varies |
| Plugin skills | from `LoadedPlugin.skillsPath` | `plugin` |
| Bundled skills | compiled into CLI | `bundled` |

Deduplication uses `realpath()` to resolve symlinks. First-seen wins.

### Skill Prompt Processing Pipeline

When a skill is invoked (`getPromptForCommand` in `createSkillCommand()`):

1. Prepend `Base directory for this skill: {baseDir}` if the skill has a base directory
2. `substituteArguments(content, args, true, argumentNames)` -- replaces `$arg_name` or positional `$1`, `$2`
3. Replace `${CLAUDE_SKILL_DIR}` with the normalized skill directory path
4. Replace `${CLAUDE_SESSION_ID}` with the current session ID
5. For non-MCP skills: `executeShellCommandsInPrompt()` processes inline `!command` or `` ```! ``` `` shell blocks. MCP skills are explicitly excluded (security: untrusted remote content cannot execute shell commands)

### Plugin Skills Additional Processing

Plugin skills go through additional processing in `loadPluginCommands.ts`:

- `${CLAUDE_PLUGIN_ROOT}` is substituted with the plugin's root directory path (distinct from `${CLAUDE_SKILL_DIR}` which points to the individual skill subdirectory)
- `${user_config.KEY}` references are substituted with user-configured values (non-sensitive only for content; sensitive values resolve to a placeholder)

---

## Custom Agents

### Agent Definition Format

Agents are defined as Markdown files with YAML frontmatter. Loaded by `parseAgentFromMarkdown()` in `src/tools/AgentTool/loadAgentsDir.ts`:

```yaml
---
name: my-agent
description: "When to use this agent (shown to the model for delegation decisions)"
tools: Bash, Read, Write, Grep, Glob     # Allowed tools (comma-separated or YAML list)
disallowedTools: FileWrite                # Tools to block
skills: my-skill, other-skill            # Skills to preload (comma-separated)
model: claude-sonnet-4-20250514          # Model override (or "inherit")
effort: high                              # low, medium, high, or integer
color: blue                               # Terminal color: blue, green, yellow, magenta, cyan
permissionMode: bypassPermissions         # Permission mode for the agent
maxTurns: 20                              # Max agentic turns before stopping
background: true                          # Always run as background task
initialPrompt: "/setup-env"              # Prepended to first turn (slash commands work)
memory: user                              # Persistent memory scope: user, project, local
isolation: worktree                       # Run in isolated git worktree
mcpServers:                               # MCP servers for this agent
  - "slack"                               # Reference existing server by name
  - my-server:                            # Inline definition
      command: "node"
      args: ["server.js"]
hooks:                                    # Session-scoped hooks
  PreToolUse:
    - matcher: ".*"
      hooks:
        - type: command
          command: "echo hook"
---

You are a specialized agent for...
(system prompt goes here)
```

### Agent Discovery Locations

Agents are loaded by `getAgentDefinitionsWithOverrides()`:

1. **Built-in agents**: `getBuiltInAgents()` returns compiled-in agents (Explore, Plan, General Purpose, Verification, etc.). Can be disabled with `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=true`.
2. **Plugin agents**: Loaded from enabled plugins via `loadPluginAgents()`.
3. **Custom agents**: Loaded from `.claude/agents/` directories via `loadMarkdownFilesForSubdir('agents', cwd)`. Searches managed, user, and project directories.

### Agent Type Hierarchy

```typescript
type AgentDefinition =
  | BuiltInAgentDefinition   // source: 'built-in', dynamic getSystemPrompt()
  | CustomAgentDefinition    // source: SettingSource, from markdown files
  | PluginAgentDefinition    // source: 'plugin', from plugin directories
```

Agent override precedence (later wins for same `agentType` name):
```
built-in < plugin < userSettings < projectSettings < flagSettings < policySettings
```

### JSON Agent Format

Agents can also be defined in JSON (used by flag/policy settings). Parsed by `parseAgentFromJson()`:

```json
{
  "my-agent": {
    "description": "When to use this agent",
    "prompt": "You are a specialized agent...",
    "tools": ["Bash", "Read"],
    "model": "claude-sonnet-4-20250514",
    "effort": "high",
    "maxTurns": 20,
    "skills": ["my-skill"],
    "memory": "user",
    "isolation": "worktree",
    "background": true
  }
}
```

### Plugin Agent Security Restrictions

Plugin agents (source `'plugin'`) have intentional restrictions compared to custom agents (defined in `.claude/agents/`). From `loadPluginAgents.ts`:

> `permissionMode`, `hooks`, and `mcpServers` are intentionally NOT parsed for plugin agents. Plugins are third-party marketplace code; these fields escalate what the agent can do beyond what the user approved at install time. For this level of control, define the agent in `.claude/agents/` where the user explicitly wrote the frontmatter.

Plugins can still ship hooks and MCP servers at the manifest level -- that is the install-time trust boundary. Per-agent declarations would let a single agent file silently add them.

---

## Custom Commands (Legacy)

The `.claude/commands/` directory is the legacy location for skills. It supports both formats:

1. **Single file**: `.claude/commands/my-command.md` -- the filename (minus `.md`) becomes the command name
2. **Directory format**: `.claude/commands/my-skill/SKILL.md` -- the directory name becomes the command name

Commands from `/commands/` default to `user-invocable: true` and use `loadedFrom: 'commands_DEPRECATED'`.

Namespacing uses `:` separator based on directory structure:
- `.claude/commands/deploy/staging.md` becomes `/deploy:staging`
- `.claude/commands/deploy/SKILL.md` becomes `/deploy`

---

## Bundled Skills

Bundled skills are compiled into the CLI binary. They are registered at startup via `registerBundledSkill()` in `src/skills/bundledSkills.ts`.

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
  files?: Record<string, string>  // Reference files extracted to disk on first invocation
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

The `files` field enables bundled skills to ship reference files. On first invocation, files are extracted to a deterministic directory under `getBundledSkillsRoot()` with security hardening:
- Per-process nonce in the path prevents pre-created symlink attacks
- `O_NOFOLLOW | O_EXCL` flags on file creation (no symlink following, exclusive create)
- `0o700` directory permissions, `0o600` file permissions
- Path traversal validation: relative paths are normalized and checked for `..` components

Current bundled skills include: `simplify`, `loop`, `remember`, `debug`, `verify`, `claude-api`, `update-config`, `keybindings`, `schedule`, `batch`, `skillify`, and others (see `src/skills/bundled/index.ts`).

---

## Built-in Plugins

Built-in plugins are a hybrid between bundled skills and marketplace plugins. They:
- Ship with the CLI (compiled in)
- Appear in the `/plugin` UI under a "Built-in" section
- Can be enabled/disabled by users (persisted to settings as `{name}@builtin`)
- Can provide skills, hooks, and MCP servers

Defined by `BuiltinPluginDefinition` in `src/types/plugin.ts`:

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean      // Hide if system lacks capability
  defaultEnabled?: boolean         // Default before user sets preference (default: true)
}
```

Registration happens in `src/plugins/bundled/index.ts` via `initBuiltinPlugins()`. As of this writing, the scaffolding exists but no built-in plugins are registered yet -- it serves as the migration path for bundled skills that should be user-toggleable.

Enable/disable state resolution:
```
user preference (settings.enabledPlugins[pluginId]) > plugin defaultEnabled > true
```

---

## Hook Registration

### Hook Events

Plugins can register hooks for these lifecycle events (from `loadPluginHooks.ts`):

| Event | Description |
|---|---|
| `PreToolUse` | Before a tool is executed |
| `PostToolUse` | After a tool executes successfully |
| `PostToolUseFailure` | After a tool execution fails |
| `PermissionDenied` | When a permission request is denied |
| `PermissionRequest` | When a permission is requested |
| `Notification` | On notification events |
| `UserPromptSubmit` | When user submits a prompt |
| `SessionStart` | At session startup |
| `SessionEnd` | At session end |
| `Stop` | When agent stops |
| `SubagentStart` / `SubagentStop` | Sub-agent lifecycle |
| `PreCompact` / `PostCompact` | Context window compaction |
| `Setup` | Initial setup |
| `TaskCreated` / `TaskCompleted` | Task lifecycle |
| `Elicitation` / `ElicitationResult` | Model elicitation |
| `ConfigChange` | Settings changes |
| `WorktreeCreate` / `WorktreeRemove` | Git worktree lifecycle |
| `InstructionsLoaded` | CLAUDE.md loading |
| `CwdChanged` | Working directory changes |
| `FileChanged` | File system changes |

### Hook Configuration Format

Hooks use a matcher-based pattern. Each hook event contains an array of matchers:

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "echo 'pre-tool hook fired'"
        }
      ]
    }
  ]
}
```

### Plugin Hook Loading

Plugin hooks are loaded by `loadPluginHooks()` (memoized). The process:

1. Load all enabled plugins via `loadAllPluginsCacheOnly()`
2. For each plugin with `hooksConfig`, convert to `PluginHookMatcher` objects (adding `pluginRoot`, `pluginName`, `pluginId` context)
3. Atomic clear-then-register: `clearRegisteredPluginHooks()` then `registerHookCallbacks()`

The atomic swap is critical -- previous implementation had a bug (gh-29767) where `clearAllCaches()` wiped hooks from state, and they were never re-registered for certain events (like `Stop`).

### Hot Reload

Plugin hooks support hot reload via `settingsChangeDetector`. When `enabledPlugins` changes in settings, hooks are pruned/reloaded. `pruneRemovedPluginHooks()` removes hooks from disabled/uninstalled plugins without adding new ones (consistency with how commands/agents/MCP behave).

---

## Security Boundaries

### Trust Hierarchy

```
Compiled code (built-in agents, bundled skills)
    └── Can do anything, full API access
Built-in plugins (user-toggleable)
    └── Same trust as bundled, but user can disable
Marketplace plugins (user-installed)
    └── Restricted: no per-agent permissionMode/hooks/mcpServers
    └── Hooks and MCP servers only at manifest level (install-time approval)
    └── ${user_config.KEY} sensitive values resolve to placeholder in content
Custom skills/agents/commands (user-written)
    └── Full frontmatter support including permissionMode, hooks, mcpServers
    └── User explicitly wrote the file, so full trust
MCP skills (remote)
    └── Most restricted: no shell execution in markdown body
    └── ${CLAUDE_SKILL_DIR} is meaningless
```

### Plugin Security Measures

1. **Marketplace impersonation prevention**: Reserved names (`claude-code-marketplace`, `anthropic-plugins`, etc.) can only come from the official `anthropics` GitHub org. Homograph attack prevention blocks non-ASCII characters in marketplace names.

2. **Path traversal prevention**: All plugin paths are sanitized. Version strings, marketplace names, and plugin names are sanitized with `replace(/[^a-zA-Z0-9\-_]/g, '-')`. Relative paths must start with `./` and cannot contain `..`.

3. **Bundled skill file extraction security**: Per-process nonce in extraction directory, `O_NOFOLLOW | O_EXCL` flags, `0o700`/`0o600` permissions.

4. **Plugin agent restrictions**: `permissionMode`, `hooks`, and `mcpServers` frontmatter fields are silently ignored for plugin agents. This prevents a single buried `.md` file in a plugin from escalating permissions.

5. **MCP skill shell execution block**: Skills loaded from MCP sources (`loadedFrom === 'mcp'`) cannot execute inline shell commands (`!command` syntax). This prevents untrusted remote content from running arbitrary code.

6. **Policy enforcement**: Enterprise managed settings can force-disable plugins via `isPluginBlockedByPolicy()`. The `isRestrictedToPluginOnly('skills')` check can lock down skill loading to only plugin sources.

7. **Sensitive user config**: Plugin `userConfig` fields marked `sensitive: true` are stored in secure storage (macOS keychain or `.credentials.json`) rather than `settings.json`. In content substitution, sensitive values resolve to a placeholder.

8. **Plugin dependency verification**: `verifyAndDemote()` checks that declared dependencies are available and enabled before plugin activation.

---

## Discovery Pipeline

### Startup Flow

```
CLI startup
├── initBuiltinPlugins()                    # Register built-in plugins
├── registerBundledSkill() (per skill)      # Register compiled skills
│
├── getSkillDirCommands(cwd)                # Load file-based skills
│   ├── loadSkillsFromSkillsDir(managed)    #   from policySettings
│   ├── loadSkillsFromSkillsDir(user)       #   from ~/.claude/skills/
│   ├── loadSkillsFromSkillsDir(project[])  #   from .claude/skills/ (walk up)
│   ├── loadSkillsFromSkillsDir(addDirs[])  #   from --add-dir paths
│   └── loadSkillsFromCommandsDir(cwd)      #   legacy .claude/commands/
│
├── getAgentDefinitionsWithOverrides(cwd)   # Load agents
│   ├── getBuiltInAgents()                  #   compiled-in agents
│   ├── loadPluginAgents()                  #   from enabled plugins
│   └── loadMarkdownFilesForSubdir('agents')#   .claude/agents/ (walk up)
│
├── loadAllPluginsCacheOnly()               # Load plugin metadata
│   ├── getBuiltinPlugins()                 #   built-in plugins
│   ├── marketplace resolution              #   plugin@marketplace lookup
│   └── resolvePluginPath()                 #   versioned cache resolution
│
├── loadPluginHooks()                       # Register plugin hooks
│   └── convertPluginHooksToMatchers()      #   with plugin context
│
├── loadPluginCommands()                    # Load plugin skills/commands
│   └── walkPluginMarkdown()                #   walk plugin dirs
│
└── MCP/LSP integration                     # Start plugin servers
    ├── mcpPluginIntegration                #   MCP servers from plugins
    └── lspPluginIntegration                #   LSP servers from plugins
```

### Memoization

All loader functions are memoized (`lodash-es/memoize`):
- `getSkillDirCommands(cwd)` -- keyed by cwd
- `getAgentDefinitionsWithOverrides(cwd)` -- keyed by cwd
- `loadAllPluginsCacheOnly()` -- no args
- `loadPluginAgents()` -- no args
- `loadPluginHooks()` -- no args

Each has a `clear*Cache()` function for invalidation (e.g., after `/plugins` UI changes or `/reload-plugins`).

---

## Lessons for Building Extensibility

### 1. Use a layered trust model

The system has five trust tiers from "compiled in" to "untrusted remote." Each layer has specific restrictions. Plugin agents cannot set `permissionMode` or `hooks` -- those fields are silently ignored with a debug log. MCP skills cannot run shell commands. This defense-in-depth prevents privilege escalation through the extension system.

### 2. Make the manifest schema additive

The `plugin.json` manifest uses Zod with `.partial()` for all component schemas. Unknown top-level fields are silently stripped (future-proofing), but nested config objects remain strict (catch typos). This balance allows plugins to evolve without breaking the loader.

### 3. Separate discovery from execution

Skill/agent/command discovery is heavily memoized and runs at startup. The actual prompt execution (argument substitution, shell commands, variable replacement) happens lazily on invocation. This keeps startup fast while maintaining full flexibility at runtime.

### 4. Use frontmatter for declarative configuration

Rather than requiring plugins to write TypeScript or register hooks programmatically, the system uses YAML frontmatter in Markdown files. This makes the extension format accessible to non-developers and enables static analysis/validation without execution.

### 5. Version and cache aggressively

Plugins are cached at versioned paths (`{marketplace}/{plugin}/{version}/`). Seed directories enable offline/enterprise deployment. ZIP cache mode reduces filesystem overhead. The `.git` directory is stripped from cache copies.

### 6. Make hooks atomic

The hook registration uses an atomic clear-then-register pattern. A previous approach where clearing and registering were separate operations led to a bug where hooks were wiped and never re-registered (gh-29767). The lesson: state mutations in extension systems should be transactional.

### 7. Deduplicate via canonical paths

Skills loaded from overlapping directories (symlinks, parent traversal) are deduplicated using `realpath()`. This avoids double-loading without requiring unique names across all sources.

### 8. Provide multiple extension surfaces

Plugins can contribute commands, skills, agents, hooks, MCP servers, LSP servers, output styles, and settings -- all from a single `plugin.json`. Users writing local extensions use the simpler Markdown-with-frontmatter format. This tiered complexity serves both simple and advanced use cases.
