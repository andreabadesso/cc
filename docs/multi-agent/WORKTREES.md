# Worktree Isolation

**Source**: `src/utils/worktree.ts`

Git worktrees provide filesystem isolation for agents. Each worktree is a separate working directory backed by the same git repository (shared `.git/objects`), allowing agents to make changes without interfering with the leader or each other.

## Architecture

```mermaid
graph TB
    subgraph MainRepo["Main Repository"]
        Git[".git/<br/>(objects, refs, config)"]
        WD["Working Directory<br/>(leader's files)"]
        NM["node_modules/"]
    end

    subgraph WT1["Worktree: agent-a1b2c3d4"]
        WD1["Working Directory<br/>(agent A's files)"]
        Sym1["node_modules/ →<br/>symlink to main"]
        GitRef1[".git file →<br/>points to main .git"]
    end

    subgraph WT2["Worktree: agent-e5f6g7h8"]
        WD2["Working Directory<br/>(agent B's files)"]
        Sym2["node_modules/ →<br/>symlink to main"]
        GitRef2[".git file →<br/>points to main .git"]
    end

    Git ---|"shared objects"| WT1
    Git ---|"shared objects"| WT2
    NM -.->|"symlink"| Sym1
    NM -.->|"symlink"| Sym2
```

## Worktree Session State

```typescript
interface WorktreeSession {
  originalCwd: string           // Where the leader started
  worktreePath: string          // Full path to worktree
  worktreeName: string          // Slug used for creation
  worktreeBranch?: string       // Git branch name
  originalBranch?: string       // Leader's branch at creation time
  originalHeadCommit?: string   // Base commit SHA
  sessionId: string             // Claude session UUID
  tmuxSessionName?: string      // Optional tmux session name
  hookBased?: boolean           // Worktree created via VCS hook
  creationDurationMs?: number   // How long creation took
  usedSparsePaths?: boolean     // Whether sparse checkout was applied
}
```

## Creation Flow

```mermaid
flowchart TD
    Start["getOrCreateWorktree(repoRoot, slug, options?)"]

    FastResume{"readWorktreeHeadSha()<br/>Worktree already<br/>exists?"}
    ResumeReturn["Return immediately<br/>{ existed: true }"]

    CreateDir["Create .claude/worktrees/ directory"]

    FetchBase["Fetch base branch / PR"]
    ResolveRef{"resolveRef()<br/>Ref exists locally?"}
    SkipFetch["Skip fetch (already local)"]
    GitFetch["git fetch origin {ref}<br/>GIT_TERMINAL_PROMPT=0<br/>GIT_ASKPASS=''<br/>stdin: 'ignore'"]

    GetSHA["git rev-parse {base} → base SHA"]

    AddWorktree["git worktree add -B {branch} {path} {base}<br/>-B flag: reset orphan, avoid D/F conflicts"]

    SparseCheck{"Sparse paths<br/>configured?"}
    SetSparse["git sparse-checkout set --cone -- {paths}<br/>git checkout HEAD"]
    SparseError["On error: remove worktree immediately<br/>before propagating error"]

    PostSetup["performPostCreationSetup()"]
    Return["Return { worktreePath, branch, existed: false }"]

    Start --> FastResume
    FastResume -->|Yes| ResumeReturn
    FastResume -->|No| CreateDir --> FetchBase
    FetchBase --> ResolveRef
    ResolveRef -->|Yes| SkipFetch --> GetSHA
    ResolveRef -->|No| GitFetch --> GetSHA
    GetSHA --> AddWorktree --> SparseCheck
    SparseCheck -->|Yes| SetSparse
    SparseCheck -->|No| PostSetup
    SetSparse -->|Success| PostSetup
    SetSparse -->|Error| SparseError
    PostSetup --> Return
```

### Fast Resume Path

Before spawning a subprocess to check worktree existence, the system reads the worktree's `.git` pointer file directly from the filesystem:

```typescript
function readWorktreeHeadSha(worktreePath: string): string | null {
  // Read .git file (not directory) → points to main repo's worktree entry
  // Read HEAD from the worktree's gitdir
  // No subprocess needed (~0ms vs ~15ms for git rev-parse)
}
```

### Git Fetch Safety

All git fetch operations use safety measures to prevent hanging:
- `GIT_TERMINAL_PROMPT=0` — prevent password prompts
- `GIT_ASKPASS=''` — prevent SSH askpass
- `stdin: 'ignore'` — prevent stdin blocking

### Slug Validation

```typescript
function validateWorktreeSlug(slug: string): void {
  // Max 64 characters
  // Segments only [a-zA-Z0-9._-]
  // Rejects '.', '..', leading/trailing '/'
  // Allows nesting with '/' (e.g., 'user/feature')
}
```

For agent worktrees, the slug is `agent-{8-char-random-hex}`.

## Post-Creation Setup

**Function**: `performPostCreationSetup(repoRoot, worktreePath)`

```mermaid
flowchart TD
    Start["performPostCreationSetup()"]
    CopySettings["Copy settings.local.json<br/>(propagate local secrets)"]
    ConfigHooks["Set core.hooksPath<br/>→ main repo's hooks dir"]
    Symlink["Symlink directories<br/>(from settings.worktree.symlinkDirectories)"]
    CopyInclude["Copy .worktreeinclude files"]
    Attribution["Install attribution hook<br/>(async, if feature enabled)"]

    Start --> CopySettings --> ConfigHooks --> Symlink --> CopyInclude --> Attribution
```

### Settings Propagation
- Copies `settings.local.json` from the main repo to the worktree
- This propagates local secrets and project-specific overrides

### Git Hooks Configuration
- Sets `core.hooksPath` in the worktree's git config to point to the main repo's hooks directory
- Caches the existing value to skip the subprocess on subsequent worktree creates

### Symlinked Directories
- Configured via `settings.worktree.symlinkDirectories`
- Typically includes `node_modules/` to avoid duplicating hundreds of MB
- Symlinks point back to the main repo's directories

### .worktreeinclude Files
- Uses `git ls-files --directory` to find gitignored files that need to be in the worktree
- Filters via the `ignore` library (respects `.worktreeinclude` patterns)
- Copies matching files to the worktree

## Worktree Cleanup

### Session Worktree Cleanup

```mermaid
sequenceDiagram
    participant U as User / Exit Handler
    participant W as cleanupWorktree()
    participant Git as Git

    U->>W: cleanupWorktree()
    W->>W: Change back to originalCwd

    alt Hook-based worktree
        W->>W: executeWorktreeRemoveHook()
    else Git-based worktree
        W->>Git: git worktree remove --force {path}
        W->>W: Wait 100ms (git lock release)
        W->>Git: git branch -D {branch}
    end

    W->>W: Update project config
```

### Agent Worktree Cleanup

```typescript
async function removeAgentWorktree(
  path: string,
  branch?: string,
  gitRoot?: string,
  hookBased?: boolean
): Promise<boolean>
```

Lightweight version called from agent context with explicit `gitRoot`. Same logic: `git worktree remove --force` + `git branch -D`.

### Stale Worktree Cleanup

```mermaid
flowchart TD
    Start["cleanupStaleAgentWorktrees(cutoffDate)"]
    Scan["Scan .claude/worktrees/ for ephemeral patterns:<br/>agent-[0-9a-f]{7}, wf_*, bridge-*, job-*"]
    ForEach["For each matching worktree:"]
    CheckAge{"mtime > cutoffDate?"}
    Skip["Skip (too recent)"]
    CheckClean{"Status clean?"}
    SkipDirty["Skip (has changes)"]
    CheckReachable{"Commits reachable<br/>from remote?"}
    SkipUnreachable["Skip (unique commits)"]
    Remove["git worktree remove --force"]
    Count["Increment cleanup count"]
    Prune["git worktree prune<br/>(clean git internal state)"]

    Start --> Scan --> ForEach --> CheckAge
    CheckAge -->|No| Skip
    CheckAge -->|Yes| CheckClean
    CheckClean -->|No| SkipDirty
    CheckClean -->|Yes| CheckReachable
    CheckReachable -->|No| SkipUnreachable
    CheckReachable -->|Yes| Remove --> Count
    Count --> ForEach
    ForEach -->|Done| Prune
```

## Tmux Integration

**Function**: `execIntoTmuxWorktree(args)`

For the `--worktree --tmux` CLI flag combination, the system:

1. Validate worktree slug
2. Create/resume worktree via `getOrCreateWorktree()`
3. Generate tmux session name: `{repoName}_{branch}` (with `/` → `_`)
4. Set environment variables:
   - `CLAUDE_CODE_TMUX_SESSION`
   - `CLAUDE_CODE_TMUX_PREFIX`
   - `CLAUDE_CODE_TMUX_PREFIX_CONFLICTS`
5. Spawn tmux:
   - `tmux new-session -A -s {name} -c {dir} -- {executable} {args}`
   - `stdio: 'inherit'` for interactive use
   - If already inside tmux: `switch-client` instead of nested attach
