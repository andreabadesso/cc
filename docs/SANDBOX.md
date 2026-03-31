# Sandbox Architecture

How Claude Code sandboxes command execution at the OS level — filesystem restrictions, network isolation, and platform-specific mechanisms.

## Architecture

Three-tier design:

```
Integration Points (BashTool, Shell.ts)
  └─ shouldUseSandbox() decision
       └─ Adapter Layer (sandbox-adapter.ts)
            └─ convertToSandboxRuntimeConfig()
                 └─ External Runtime (@anthropic-ai/sandbox-runtime)
                      └─ OS-Level Enforcement
                           ├─ macOS: Seatbelt (native sandbox)
                           └─ Linux/WSL: bubblewrap + seccomp BPF
```

## Sandbox Decision Flow

```typescript
function shouldUseSandbox(input: { command?, dangerouslyDisableSandbox? }): boolean {
  if (!SandboxManager.isSandboxingEnabled()) return false
  if (input.dangerouslyDisableSandbox &&
      SandboxManager.areUnsandboxedCommandsAllowed()) return false
  if (!input.command) return false
  if (containsExcludedCommand(input.command)) return false
  return true
}
```

**Source:** `src/tools/BashTool/shouldUseSandbox.ts`

## Filesystem Restrictions

### Always Blocked (Write)

- `.claude/settings.json` and `.claude/settings.local.json` (both original and current cwd)
- `.claude/skills`, `.claude/commands`, `.claude/agents`
- Bare git repo files: `HEAD`, `objects`, `refs`, `hooks`, `config` (prevents git escape attacks)

### Always Allowed (Write)

- Current working directory (`.`)
- Claude temp directory (`getClaudeTempDir()`)
- Git worktree main repo paths

### Path Resolution

```
Permission rules (Tool-specific):
  "/path"   → relative to settings file directory
  "//path"  → absolute from filesystem root
  "~/path"  → expanded to home

sandbox.filesystem.* (standard path semantics):
  "/path"   → absolute path
  "~/path"  → expanded to home
  "./path"  → relative to settings file
```

### Git Bare Repo Escape Prevention

```typescript
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.
```

Defence: Planted files are either deny-listed at config time or scrubbed post-command via `scrubBareGitRepoFiles()`.

## Network Restrictions

### Configuration

```json
{
  "sandbox": {
    "network": {
      "allowedDomains": ["api.anthropic.com", "*.github.com"],
      "deniedDomains": ["evil.com"],
      "allowUnixSockets": ["/var/run/docker.sock"],
      "allowAllUnixSockets": false
    }
  }
}
```

`WebFetch(domain:...)` permission rules are automatically parsed and merged into allowed/denied domains.

### Policy Override

`allowManagedDomainsOnly`: When set in policy settings, only policy-approved domains are used. User/project/local settings are ignored (denied domains still respected from all sources).

### Proxy Support

- `httpProxyPort` and `socksProxyPort` for proxy routing
- `enableWeakerNetworkIsolation`: Allows `com.apple.trustd.agent` access on macOS for Go CLI tools with custom CAs — opens data exfiltration vector (documented trade-off)

## Platform-Specific Mechanisms

### macOS: Seatbelt

- Native macOS sandbox mechanism, built-in (no dependencies beyond ripgrep)
- Profile-based enforcement via system APIs
- Unix socket filtering via path-based rules
- Full glob pattern support in path rules
- Log monitoring enabled by default

### Linux/WSL: bubblewrap + seccomp

- **bubblewrap (bwrap):** User-namespace containerization with ro-bind/rw-bind filesystem mounting
- **seccomp BPF filters:** Syscall filtering, required for Unix domain socket blocking
- **socat:** Network proxy implementation
- **Limitation:** No glob pattern support — must use exact paths
- **Limitation:** seccomp cannot filter by socket path (only by operation type)
- Creates 0-byte mount-point files for denied non-existent paths
- Post-command cleanup needed on Linux for planted files

### Comparison

| Aspect | macOS | Linux/WSL |
|--------|-------|-----------|
| Mechanism | Seatbelt (native) | bubblewrap + seccomp |
| Dependencies | ripgrep | bubblewrap, socat, seccomp, ripgrep |
| Socket filtering | Path-based rules | Syscall-based only |
| Glob support | Full | Exact paths only |
| Profile type | Seatbelt DSL | BPF bytecode |

## dangerouslyDisableSandbox

Two gates protect this escape hatch:

1. **Setting-level:** `allowUnsandboxedCommands: boolean` (default: `true`). When `false`, the parameter is completely ignored.
2. **Decision logic:** Only effective when both `dangerouslyDisableSandbox=true` AND `areUnsandboxedCommandsAllowed()=true`.

## Command Execution Flow

```
BashTool.tsx → exec(command, ...)
  └─ Shell.ts exec()
       ├─ shouldUseSandbox({ command, dangerouslyDisableSandbox })
       ├─ if sandboxed:
       │    commandString = SandboxManager.wrapWithSandbox(command, shell)
       ├─ spawn(binary, args, env)
       ├─ Monitor stdout/stderr
       ├─ Extract sandbox violations from stderr
       └─ SandboxManager.cleanupAfterCommand()
            └─ scrubBareGitRepoFiles()
```

### Symlink Protection

Output files opened with `O_NOFOLLOW`:
```typescript
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
outputHandle = await open(path,
  O_WRONLY | O_CREAT | O_APPEND | O_NOFOLLOW)
```

Prevents sandbox-spawned commands from following symlinks to write output to unintended locations.

### Violation Tracking

Sandbox violations are embedded in stderr as `<sandbox_violations>...</sandbox_violations>` tags, collected in `SandboxViolationStore`, and displayed in the UI via `SandboxViolationExpandedView`.

## Dynamic Configuration Updates

Sandbox config subscribes to settings changes:

```typescript
settingsChangeDetector.subscribe(() => {
  const newConfig = convertToSandboxRuntimeConfig(getSettings())
  BaseSandboxManager.updateConfig(newConfig)
})
```

No restart required — settings changes apply to the next command.

## Relationship to Permission System

Two complementary layers:

```
User tries: cat /etc/passwd
  ↓
1. Permission check: Read rule denied? → User prompt
2. Sandbox check: denyRead includes /etc? → OS blocks + violation event
```

Both layers can independently prevent access. The permission system is app-level (prompts user). The sandbox is OS-level (kernel enforcement).

`autoAllowBashIfSandboxed`: When enabled, Bash commands are automatically approved by the permission system if sandboxing is active (the sandbox provides the safety net).
