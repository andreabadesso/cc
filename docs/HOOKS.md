# Hook System

User-configurable shell commands that execute in response to agent events. Hooks enable automation ("run lint after every edit"), policy enforcement ("block commits to main"), and observability without writing plugins.

## Configuration

Hooks are defined in `settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "eslint --fix $FILE",
            "timeout": 30,
            "if": "Write(*.ts)"
          }
        ]
      }
    ]
  }
}
```

**Source:** `src/schemas/hooks.ts`

## Hook Types

| Type | How it runs | Use case |
|------|------------|----------|
| `command` | Shell command (Bash/PowerShell) | Linting, formatting, custom validation |
| `prompt` | LLM evaluation via small fast model | Content policy checks |
| `agent` | Multi-turn agent verification | Complex validation requiring reasoning |
| `http` | HTTP POST (JSON payload) | External service integration |
| `callback` | TypeScript function (internal only) | Built-in analytics, attribution |
| `function` | JavaScript callback (session-only) | Runtime validation in agent hooks |

## Hook Events (26 Total)

### Tool Lifecycle

| Event | When | Can Block? |
|-------|------|-----------|
| `PreToolUse` | Before tool execution | Yes (approve/deny/ask + modify input) |
| `PostToolUse` | After tool succeeds | Yes (can update MCP tool output) |
| `PostToolUseFailure` | After tool fails | No (fire-and-forget) |

### User Input

| Event | When | Can Block? |
|-------|------|-----------|
| `UserPromptSubmit` | User submits prompt | Yes (can filter/modify) |

### Session Lifecycle

| Event | When | Can Block? |
|-------|------|-----------|
| `SessionStart` | Session init (startup, resume, clear, compact) | No |
| `SessionEnd` | Session termination | No |
| `Stop` | Before Claude finishes response | Yes (can inject context) |
| `StopFailure` | Turn ends due to API error | No (fire-and-forget) |

### Agent/Team

| Event | When | Can Block? |
|-------|------|-----------|
| `SubagentStart` | Agent tool call starts | No |
| `SubagentStop` | Agent about to conclude | No |
| `TeammateIdle` | Teammate going idle | Yes |
| `TaskCreated` | Task creation | Yes |
| `TaskCompleted` | Task marked complete | Yes |

### File/Directory

| Event | When | Can Block? |
|-------|------|-----------|
| `CwdChanged` | Working directory changes | No (can set env via `$CLAUDE_ENV_FILE`) |
| `FileChanged` | Watched file changes | No |
| `WorktreeCreate` | Worktree created | No |
| `WorktreeRemove` | Worktree removed | No |

### Other

`PermissionRequest`, `PermissionDenied`, `Notification`, `Setup`, `ConfigChange`, `InstructionsLoaded`, `Elicitation`, `ElicitationResult`, `PreCompact`, `PostCompact`

## Hook Input Format

All hooks receive JSON on stdin:

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": { "file_path": "/src/index.ts", "content": "..." },
  "tool_use_id": "toolu_abc123",
  "permissions_mode": "default",
  "current_working_directory": "/home/user/project",
  "claude_code_version": "1.2.3"
}
```

**PostToolUse** adds `tool_response`. **PostToolUseFailure** adds `error`, `error_type`, `is_interrupt`, `is_timeout`. **UserPromptSubmit** has `prompt_text`.

## Hook Output Format

Hooks output JSON on stdout (first line):

```json
{
  "continue": true,
  "decision": "approve",
  "reason": "Lint passed",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": { "file_path": "/src/index.ts", "content": "// fixed\n..." },
    "additionalContext": "Note: auto-fixed 2 lint issues"
  }
}
```

### Exit Code Semantics

| Code | Meaning | Behavior |
|:---:|---------|----------|
| 0 | Success | Continue normally |
| 2 | Blocking error | STOP execution, show stderr to model |
| Other | Non-blocking error | Show stderr to user, continue |

## Permission Decision Flow (PreToolUse)

```
Tool invocation
  ↓
PreToolUse hooks execute (parallel)
  ├─ Command hook → JSON output with permissionDecision
  ├─ Prompt hook → LLM evaluation → {ok: true/false}
  └─ Agent hook → Multi-turn verification → {ok: true/false}
  ↓
Aggregate results (precedence: deny > ask > allow > passthrough)
  ↓
Apply permission decision:
  ├─ "allow" → Skip user prompt (but deny/ask rules still apply)
  ├─ "ask"   → Show dialog with hook's message
  ├─ "deny"  → Block immediately
  └─ updatedInput → Modify tool arguments before execution
```

**Key:** Hook `allow` does NOT bypass permission rules in settings.json. Rules always take precedence.

## Configuration Sources (Precedence)

1. Built-in hooks (lowest priority)
2. Plugin hooks
3. User `~/.claude/` hooks
4. Project `.claude/` hooks
5. Local `.claude/` hooks
6. Managed settings hooks (highest, if `allowManagedHooksOnly` set)

**Deduplication:** By `{type, command/prompt/url, if condition}`. Context namespacing via pluginRoot prevents cross-plugin collisions.

## Policy Restrictions

- `allowManagedHooksOnly`: Only managed hooks run; user/project/local/plugin hooks ignored
- `allowedHttpHookUrls`: URL whitelist for HTTP hooks (supports `*` wildcard)
- `httpHookAllowedEnvVars`: Allowlist for environment variable interpolation in HTTP headers
- **SSRF guard:** Blocks private IP ranges in HTTP hook URLs (unless proxy in use)
- **Header injection prevention:** CRLF/NUL sanitization on interpolated env vars
- **Workspace trust gate:** All hooks blocked if trust not accepted

## Async Hooks

Hooks can run in the background:

```json
{ "async": true, "asyncTimeout": 5000 }
```

- `async: true`: Fire-and-forget, don't block the tool
- `asyncRewake: true`: Background execution, but exit code 2 triggers wake-up

## Built-In Hook Examples

| Hook | Event | Purpose |
|------|-------|---------|
| Session file access analytics | PostToolUse | Records file reads for context analysis |
| Attribution hooks | PostToolUse | Tracks commit authorship context |
| Structured output enforcement | Stop | Ensures agent returns JSON via SyntheticOutput |
| Skill improvement hooks | Various | From skill definition frontmatter |
| Frontmatter hooks | Various | From agent/skill CLAUDE.md files |

## Lifecycle Example

```
User: "Write hello.ts"
  ↓
PreToolUse fires for Write tool:
  ├─ Matcher "Write|Edit" matches ✓
  ├─ "if" condition "Write(*.ts)" matches ✓
  ├─ Hook spawned: eslint --fix $FILE
  ├─ Stdin: JSON { tool_name: "Write", tool_input: {...} }
  ├─ Output: { "permissionDecision": "allow", "updatedInput": {...} }
  └─ Exit code: 0
  ↓
Permission: allow (skip dialog, check rules still apply)
  ↓
Write tool executes (with possibly modified input)
  ↓
PostToolUse fires:
  ├─ Input: { tool_name: "Write", tool_response: {...} }
  └─ Exit code: 0
  ↓
Model receives tool result + any additionalContext from hooks
```
