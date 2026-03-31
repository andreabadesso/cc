# Permission System & Security Architecture

Claude Code implements defense-in-depth security through multiple layers: permission modes, rule matching, ML classifiers, hooks, sandboxing, enterprise policy enforcement, and environment variable filtering.

---

## Permission Modes

Six permission modes define the security posture (`src/types/permissions.ts`):

| Mode | Behavior |
|------|----------|
| `default` | Prompts user for each tool use |
| `plan` | Auto-allows file edits; other operations prompt |
| `acceptEdits` | Auto-allows edits; denies everything else |
| `bypassPermissions` | Skips all checks (explicit user opt-in only) |
| `dontAsk` | Converts all "ask" decisions to "deny" (headless safe mode) |
| `auto` | ML classifier auto-approves/denies based on transcript analysis |
| `bubble` | Internal mode for subcommand result aggregation |

The `auto` mode is the most interesting -- it uses a classifier model to evaluate whether each tool invocation is safe given the conversation context.

---

## Permission Decision Hierarchy

When a tool is invoked, the system walks a strict decision hierarchy (`src/utils/permissions/permissions.ts`):

```
1. Pre-tool-use hooks (can approve/block/modify input)
   ↓
2. Deny rules (hard block, no override)
   ↓
3. Allow rules (auto-approve matching patterns)
   ↓
4. Mode-specific logic:
   - bypassPermissions → allow
   - dontAsk → deny
   - auto → ML classifier evaluation
   - default/plan → prompt user
   ↓
5. Denial tracking (after N consecutive denials in auto mode, fall back to prompting)
```

### Denial Tracking

Prevents the ML classifier from getting stuck in denial loops:

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 3 denials in a row → fall back to prompting
  maxTotal: 20,        // 20 total denials in session → fall back to prompting
}
```

This is a pragmatic safeguard: if the classifier is unsure about an operation, it's better to ask the user than to keep denying and stalling progress.

---

## Tool Permission Declarations

Each tool declares its security profile through the `Tool` interface:

```typescript
type Tool = {
  isReadOnly(input): boolean       // Can this invocation modify state?
  isDestructive(input): boolean    // Could this cause irreversible damage?
  isConcurrencySafe(input): boolean // Safe to run in parallel with other tools?
  checkPermissions(input, context): Promise<PermissionResult>
  preparePermissionMatcher(input): PermissionMatcher  // For rule matching
}
```

The key insight is that these are **per-invocation**, not per-tool. A `BashTool` running `ls` is read-only; the same tool running `rm -rf /` is destructive. The input determines the classification.

---

## Dangerous Operation Classification

### Bash Commands (`src/utils/permissions/dangerousPatterns.ts`)

Commands classified as dangerous (stripped from auto-mode allow rules):

```typescript
export const DANGEROUS_BASH_PATTERNS = [
  // Interpreters (arbitrary code execution)
  'python', 'python3', 'node', 'deno', 'ruby', 'perl', 'php',
  // Package runners
  'npx', 'bunx', 'npm run', 'yarn run',
  // Shells
  'bash', 'sh', 'zsh',
  // Remote execution
  'ssh',
  // Code execution
  'eval', 'exec', 'env', 'xargs', 'sudo',
]
```

### Protected Files and Directories (`src/utils/permissions/filesystem.ts`)

```typescript
export const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules', '.bashrc', '.bash_profile',
  '.zshrc', '.zprofile', '.profile', '.ripgreprc',
  '.mcp.json', '.claude.json'
]

export const DANGEROUS_DIRECTORIES = [
  '.git', '.vscode', '.idea', '.claude'
]
```

These files could be used for persistence attacks (modifying shell startup files) or to compromise the tool itself (modifying `.claude` config).

---

## Bash Command Parsing: AST over Regex

A critical design decision: bash commands are parsed using **tree-sitter AST analysis**, not regex (`src/tools/BashTool/bashPermissions.ts`).

Why this matters:
- Regex can be fooled by quoting, escaping, variable expansion
- AST parsing understands the actual command structure
- `parseForSecurityFromAst()` validates command shape semantically
- Compound commands are split and analyzed individually
- Cap at `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50` to prevent ReDoS

Command prefix validation only accepts clean patterns:
```typescript
// Only accepts: lowercase letters, digits, hyphens
// Rejects: flags, filenames, paths, numbers as "subcommands"
getSimpleCommandPrefix() → validates against ^[a-z][a-z0-9]*(-[a-z0-9]+)*$
```

---

## Hook System

Hooks provide extensibility points for permission decisions (`src/types/hooks.ts`):

### Pre-Decision Hooks

| Hook | When | Can Do |
|------|------|--------|
| `PreToolUse` | Before permission check | Modify input, return permission decision |
| `PermissionRequest` | Before user prompt (headless) | Allow/deny/interrupt session |

### Post-Decision Hooks

| Hook | When | Can Do |
|------|------|--------|
| `PostToolUse` | After tool executes | Modify output (MCP tools) |
| `PostToolUseFailure` | On tool failure | Handle error |
| `PermissionDenied` | On denial | Request retry |

Hook response format:
```typescript
{
  decision: 'approve' | 'block'
  reason: string
  permissionDecision?: 'allow' | 'deny' | 'ask'
  updatedInput?: Record<string, unknown>  // Can modify tool input
}
```

Hooks are themselves gated by policy:
- `shouldAllowManagedHooksOnly()` → Only enterprise-configured hooks run
- `shouldDisableAllHooksIncludingManaged()` → Nuclear option, no hooks at all

---

## Sandbox Architecture

Multi-layer isolation defined in `src/entrypoints/sandboxTypes.ts`:

### Network Isolation
```typescript
{
  allowedDomains: string[]              // Allowlist
  allowManagedDomainsOnly?: boolean     // Enterprise lockdown
  allowUnixSockets?: string[]
  allowLocalBinding?: boolean
  httpProxyPort?: number
  socksProxyPort?: number
}
```

### Filesystem Isolation
```typescript
{
  allowWrite: string[]
  denyWrite: string[]
  denyRead: string[]
  allowRead: string[]                    // Override denyRead
  allowManagedReadPathsOnly?: boolean    // Enterprise lockdown
}
```

### Always-Denied Writes
Regardless of configuration, writes are always denied to:
- `settings.json`, `.claude/settings.*` (prevents self-modification)
- `.claude/skills` (same privilege level as commands/agents)
- Bare repo files (`HEAD`, `objects`, `refs`, `hooks`, `config`) to prevent git escape attacks

---

## Enterprise Policy Enforcement (MDM)

### Platform-Specific Reading

**macOS**: `/Library/Managed Preferences/{username}/com.anthropic.claudecode.plist` (per-user) or `/Library/Managed Preferences/com.anthropic.claudecode.plist` (device-level)

**Windows**: Registry keys at `HKLM\SOFTWARE\Policies\ClaudeCode\Settings` (machine) and `HKCU\SOFTWARE\Policies\ClaudeCode\Settings` (user)

MDM reads use isolated subprocess calls (`plutil` on macOS, `reg query` on Windows) with 5-second timeouts, non-blocking.

### Policy Enforcement Points

| Setting | Effect |
|---------|--------|
| `shouldAllowManagedPermissionRulesOnly` | Only enterprise-defined permission rules apply |
| `shouldAllowManagedSandboxDomainsOnly` | Only enterprise-defined network domains allowed |
| `shouldAllowManagedReadPathsOnly` | Only enterprise-defined read paths allowed |
| `shouldAllowManagedHooksOnly` | User/project hooks disabled; only enterprise hooks run |
| `shouldDisableAllHooksIncludingManaged` | All hooks disabled |

These controls allow enterprise security teams to enforce strict policies without modifying the application code.

---

## Environment Variable Security

### Safe vs Unsafe Classification (`src/utils/managedEnvConstants.ts`)

~80 variables are classified as "safe" for project-scoped settings:
- Model config: `ANTHROPIC_SMALL_FAST_MODEL`, region overrides
- Telemetry: OTEL headers, protocols, export intervals
- Sandbox: `BASH_MAX_TIMEOUT_MS`, `MCP_TIMEOUT`
- Feature flags: `DISABLE_TELEMETRY`, `MAX_THINKING_TOKENS`

**Explicitly NOT safe** (can redirect traffic or bypass TLS):
- `ANTHROPIC_BASE_URL`, `HTTP_PROXY`
- `NODE_TLS_REJECT_UNAUTHORIZED`, `NODE_EXTRA_CA_CERTS`

### Provider-Managed Variable Stripping

When `CLAUDE_CODE_PROVIDER_MANAGED_BY_HOST` is set (desktop app, IDE extension):
- All provider selection vars stripped from user settings
- All endpoint vars stripped
- All auth vars stripped
- Prevents user settings from redirecting API calls to attacker endpoints

### SSH Tunnel Protection

When `ANTHROPIC_UNIX_SOCKET` is set (remote SSH scenario):
- Auth-related vars stripped from local settings
- Prevents local `~/.claude/settings.json` from overriding the remote proxy configuration

### Dangerous Shell Settings

Settings that execute shell commands require special handling:
```typescript
export const DANGEROUS_SHELL_SETTINGS = [
  'apiKeyHelper',        // Runs shell command to refresh API key
  'awsAuthRefresh',      // AWS auth refresh
  'gcpAuthRefresh',      // GCP auth refresh
  'otelHeadersHelper',   // Telemetry header refresh
  'statusLine',          // Status line rendering
]
```

---

## OAuth Architecture

### Endpoints (`src/constants/oauth.ts`)

```
Authorization: platform.claude.com/oauth/authorize
Token:         platform.claude.com/v1/oauth/token
API Key:       api.anthropic.com/api/oauth/claude_cli/create_api_key
```

### Scopes
- `user:inference` — Claude.ai inference
- `user:profile` — User profile access
- `org:create_api_key` — Console API key creation
- `user:sessions:claude_code` — Session management
- `user:mcp_servers` — MCP server management

### Enterprise OAuth
Custom OAuth endpoints are **allowlisted**, not open:
- Only FedStart/PubSec endpoints accepted
- Blocks arbitrary `CLAUDE_CODE_CUSTOM_OAUTH_URL` values
- Prevents credential phishing via malicious OAuth servers

---

## Key Design Principles

1. **Defense in depth**: No single layer is trusted alone. Permission modes, rules, classifiers, hooks, and sandboxing all contribute independently.

2. **Per-invocation classification**: A tool's safety depends on its *input*, not just its name. `Bash(ls)` and `Bash(rm -rf /)` get different treatment.

3. **AST over regex**: Security-critical parsing uses tree-sitter, not pattern matching. This resists evasion through quoting, escaping, and variable expansion.

4. **Fail-safe defaults**: Unknown operations default to "ask". Auto mode has denial limits that force fallback to user prompting.

5. **Enterprise controls override everything**: MDM policies can restrict permissions, network access, file access, and hooks regardless of user preferences.

6. **Variable stripping at boundaries**: Environment variables are classified and stripped based on trust context (project settings, SSH tunnels, hosted providers). This prevents settings from one trust domain from affecting another.
