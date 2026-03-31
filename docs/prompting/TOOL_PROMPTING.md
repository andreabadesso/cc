# Tool Prompting

Every tool generates a `prompt()` that becomes the `description` field in the API's tool definition. These prompts shape how the model uses each tool — and they're full of carefully crafted steering techniques.

## How Tool Prompts Reach the Model

```
Tool definition
  └─ prompt({ agents, tools, permissionContext, ... })
       └─ Returns description string
            └─ Becomes API tool schema:
                 {
                   name: "Bash",
                   description: "...",        ← the prompt
                   input_schema: { ... },     ← Zod → JSON Schema
                   cache_control?: { ... },
                 }
```

Tool schemas are cached per session via `toolSchemaCache` to prevent mid-session description drift from feature flags or GrowthBook changes.

**Source:** `src/utils/api.ts` — `toolToAPISchema()`

## Conditional Prompting

Tool prompts are not static. They adapt based on:

| Condition | Example |
|-----------|---------|
| User type | Internal users get shorter prompts with skill references |
| Feature flags | Fork guidance only when `FORK_SUBAGENT` enabled |
| Sandbox config | Bash prompt changes based on sandbox restrictions |
| MCP servers | AgentTool filters agents by available MCP servers |
| Permission rules | AgentTool filters denied agent types from listing |
| Current date | WebSearchTool injects current month/year |
| Subscription level | AgentTool adjusts concurrency guidance |
| Agent swarms | TaskCreateTool adds teammate tips when swarms enabled |
| File limits | FileReadTool adjusts offset guidance based on configured limits |

## Tool-by-Tool Analysis

### BashTool — The Most Complex Tool Prompt

The Bash prompt is the largest and most heavily conditional. Key sections:

**Tool Avoidance (steering toward dedicated tools):**
```
IMPORTANT: Avoid using this tool to run `find`, `grep`, `cat`, `head`,
`tail`, `sed`, `awk`, or `echo` commands, unless explicitly instructed.
Instead, use the appropriate dedicated tool:

- File search: Use Glob (NOT find or ls)
- Content search: Use Grep (NOT grep or rg)
- Read files: Use Read (NOT cat/head/tail)
- Edit files: Use Edit (NOT sed/awk)
- Write files: Use Write (NOT echo >/cat <<EOF)
```

**Git Safety Protocol (most heavily guarded section):**
```
# Git Safety Protocol
- NEVER update the git config
- NEVER run destructive git commands (push --force, reset --hard,
  checkout ., restore ., clean -f, branch -D) unless explicitly requested
- NEVER skip hooks (--no-verify, --no-gpg-sign) unless explicitly requested
- NEVER run force push to main/master, warn the user
- CRITICAL: Always create NEW commits rather than amending. When a
  pre-commit hook fails, the commit did NOT happen — so --amend would
  modify the PREVIOUS commit, which may destroy work
- NEVER commit changes unless the user explicitly asks
- Never use git commands with the -i flag (interactive input not supported)
- ALWAYS pass commit messages via HEREDOC
```

The `--amend` warning is particularly notable — it addresses a subtle failure mode where the model would amend after a hook failure, destroying the previous commit.

**Sandbox Configuration:**
Dynamically includes filesystem/network restriction details and override guidance (`dangerouslyDisableSandbox`).

### FileEditTool — Pre-Read Enforcement

```
You must use your Read tool at least once in the conversation before
editing. This tool will error if you attempt an edit without reading
the file.

ALWAYS prefer editing existing files. NEVER write new files unless
explicitly required.

The edit will FAIL if old_string is not unique in the file. Either
provide a larger string with more surrounding context or use replace_all.
```

**Key insight:** The pre-read requirement is enforced at both the prompt level AND the implementation level. The tool literally errors if you haven't read first. Prompt + code enforcement together.

### FileWriteTool — Anti-Documentation Bias

```
NEVER create documentation files (*.md) or README files unless
explicitly requested by the User.

If this is an existing file, you MUST use the Read tool first.
This tool will fail if you did not read the file first.

Prefer the Edit tool for modifying existing files — it only sends
the diff.
```

**Key insight:** The anti-documentation bias prevents the model from proactively creating README/docs files, which is a common LLM behavior that clutters repositories.

### GrepTool — Tool Exclusivity

```
ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a
Bash command. The Grep tool has been optimized for correct permissions
and access.
```

Short, absolute, no exceptions. This pattern appears for multiple tools — establishing that the dedicated tool is the ONLY acceptable way.

### WebSearchTool — Mandatory Output Format

```
CRITICAL REQUIREMENT — You MUST follow this:
- After answering the question, you MUST include a "Sources:" section
- In the Sources section, list all relevant URLs as markdown hyperlinks
- This is MANDATORY — never skip including sources in your response
```

**Key insight:** Uses three escalating words — "CRITICAL", "MUST", "MANDATORY" — to force a specific output format. The model almost never skips sources with this level of emphasis.

### WebFetchTool — MCP Preference + Auth Warning

```
IMPORTANT: WebFetch WILL FAIL for authenticated or private URLs. Before
using this tool, check if the URL points to an authenticated service
(e.g. Google Docs, Confluence, Jira, GitHub). If so, look for a
specialized MCP tool that provides authenticated access.
```

Also includes a secondary model prompt for content processing: strict 125-character quote limits for non-pre-approved domains (copyright protection).

### AgentTool — Dynamic Agent Listing

The most dynamically constructed prompt:

1. Filters agents by MCP requirements (checks if required servers have tools)
2. Filters agents by permission rules (removes denied agent types)
3. Lists available agents with `whenToUse` descriptions
4. Optionally moves agent listing to an attachment message for cache stability

```
When NOT to use the Agent tool:
- If you want to read a specific file path, use Read or Glob instead
- If you are searching for a specific class definition, use Glob instead
- If you are searching code within 2-3 files, use Read instead
```

**Key insight:** The "when NOT to use" section prevents over-delegation. Without it, the model tends to spawn agents for trivial tasks.

### SkillTool — Blocking Requirement

```
When a skill matches the user's request, this is a BLOCKING REQUIREMENT:
invoke the relevant Skill tool BEFORE generating any other response
about the task.

NEVER mention a skill without actually calling this tool.

If you see a <command-name> tag in the current conversation turn, the
skill has ALREADY been loaded — follow the instructions directly
instead of calling this tool again.
```

**Key insight:** "BLOCKING REQUIREMENT" + "BEFORE generating any other response" prevents the model from describing what it would do instead of actually doing it.

### SendMessageTool — Communication Enforcement

```
Your plain text output is NOT visible to other agents — to communicate,
you MUST call this tool.

Messages from teammates are delivered automatically; you don't check
an inbox.

Refer to teammates by name, never by UUID.
```

### TaskCreateTool — Conditional Teammate Context

```
Use this tool proactively in these scenarios:
- 3+ steps needed
- Non-trivial tasks
- Multi-file changes

Skip using this tool when:
- Single-step tasks
- Trivial changes
```

When agent swarms are enabled, adds teammate-specific tips about task dependencies, claiming, and blocking.

## Common Prompting Patterns Across Tools

### 1. Pre-Read Requirements
FileEditTool and FileWriteTool both require reading before writing — enforced in prompt AND implementation.

### 2. Tool Avoidance / Tool Preference
Multiple tools tell the model to prefer the dedicated tool over Bash equivalents. This is repeated in:
- The system prompt's "Using Your Tools" section
- BashTool's own prompt
- GrepTool's prompt ("NEVER invoke grep as Bash command")

### 3. "NEVER / ALWAYS / CRITICAL" Escalation
The strongest behavioral constraints use escalating emphasis:
- `IMPORTANT:` — general guidance
- `CRITICAL:` — must-follow rules
- `NEVER` / `ALWAYS` — absolute constraints
- `BLOCKING REQUIREMENT` — must happen before anything else

### 4. Negative Examples
Several tools include "when NOT to use" guidance, which is more effective than positive-only instruction:
- AgentTool: "when NOT to use the Agent tool"
- BashTool: "avoid using this tool to run find, grep, cat..."
- TaskCreateTool: "skip using this tool when..."

### 5. Output Format Enforcement
When a specific output format is required, the prompt uses multiple reinforcement:
- WebSearchTool: "CRITICAL REQUIREMENT" + "MUST" + "MANDATORY" + example format
- VerificationAgent: "VERDICT: PASS | FAIL | PARTIAL" with required evidence fields
- PlanAgent: "End your response with: ### Critical Files for Implementation"

### 6. Few-Shot Examples
Embedded directly in tool prompts:
- BashTool: HEREDOC commit message example
- AgentTool: fork usage examples with user/assistant dialogue
- WebSearchTool: Sources section format example
