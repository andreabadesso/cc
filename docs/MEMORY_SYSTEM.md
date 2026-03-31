# Auto-Memory System

How Claude Code automatically extracts, stores, retrieves, and synchronizes memories across conversations.

## Architecture Overview

```
Session Start
  └─ loadMemoryPrompt() → Read MEMORY.md → Inject into system prompt

User Query
  └─ startRelevantMemoryPrefetch() → Async ranking of topic files
       └─ Attach top 5 relevant memories to conversation

End of Turn (Stop Hooks)
  └─ executeExtractMemories() → Fork agent to extract new memories
       └─ Write topic files + update MEMORY.md index

Team Sync (Background)
  └─ Pull from server → Push deltas → Merge team memories
```

## Memory Types

| Type | Scope | When to Save | Example |
|------|-------|-------------|---------|
| `user` | Always private | Learn about user's role, preferences, expertise | "Deep Go expertise, new to React" |
| `feedback` | Private (or team for conventions) | User corrections AND confirmations | "Don't mock DBs — burned by prod divergence" |
| `project` | Private or team (bias team) | Goals, deadlines, incidents, decisions | "Merge freeze 2026-03-05 for mobile release" |
| `reference` | Usually team | Pointers to external systems | "Bugs tracked in Linear project INGEST" |

**Source:** `src/memdir/memoryTypes.ts`

## File Format

Individual memory files use YAML frontmatter:

```markdown
---
name: database testing policy
description: Integration tests must use real database, not mocks
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.

**How to apply:** When writing or reviewing test code that touches the database layer, always use the test database, never mock the connection.
```

## MEMORY.md Index

**Location:** `~/.claude/projects/<sanitized-git-root>/memory/MEMORY.md`

The index is always loaded into the system prompt (first 200 lines, ~25KB cap). Each entry is one line:

```markdown
- [Database testing policy](feedback_testing.md) — integration tests must use real DB, not mocks
- [User role](user_role.md) — senior backend engineer, new to React
- [Pipeline bugs](reference_linear.md) — tracked in Linear project INGEST
```

**MEMORY.md is an INDEX, not storage.** Content lives in individual topic files. Two-step save: write topic file → add pointer to MEMORY.md.

## CLAUDE.md Hierarchy

Memories complement the CLAUDE.md hierarchy:

| Priority | Path | Scope |
|----------|------|-------|
| 1 | `/etc/claude-code/CLAUDE.md` | Managed (enterprise MDM) |
| 2 | `~/.claude/CLAUDE.md` | User (private, global) |
| 3 | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` | Project (checked in) |
| 4 | `CLAUDE.local.md` | Local (private, project-specific) |
| 5 | `~/.claude/projects/.../memory/` | Auto-memory (cross-session) |

## Automatic Memory Extraction

### Trigger

Runs at end-of-turn via stop hooks. Fire-and-forget (errors don't block the conversation).

### Gate Checks (Cheapest First)

1. Feature gate enabled (`tengu_passport_quail`)
2. `isAutoMemoryEnabled()` check
3. Not in remote mode
4. Main agent only (not subagents)
5. Throttle: every N eligible turns (configurable)
6. Mutual exclusion: skip if main agent already wrote to memory this turn

### Forked Agent Execution

- Runs as a **perfect fork** of the main conversation (shares prompt cache)
- Reads `~N` new messages since last extraction
- **Tool restrictions:** Read/Grep/Glob (unlimited), Bash (read-only), Edit/Write (memory dir only)
- **Max 5 turns** (hard cap to prevent rabbit holes)
- Pre-injected with memory manifest (existing files, types, timestamps) to prevent duplicates

### What Gets Extracted

- User corrections: "no not that", "stop doing X"
- Validated approaches: "yes exactly", "perfect, keep doing that"
- Facts learned: who's doing what, why, by when
- External pointers: dashboards, Linear projects, Slack channels
- User role/expertise signals

### What Does NOT Get Extracted

- Code patterns, conventions, file paths (derive from code)
- Git history (use `git log`)
- Debugging solutions (the fix is in the code)
- Anything already in CLAUDE.md
- Ephemeral task details, temporary state

### Stashing

If extraction is already running when new context arrives, the new context is stashed. After the current extraction completes, a trailing run processes the stashed context.

## Memory Retrieval

### Always-On: MEMORY.md Index

Loaded synchronously at session start. Injected into system prompt. The model always sees the full index (up to 200 lines).

### Async Prefetch: Relevant Topic Files

1. Feature gate: `tengu_moth_copse`
2. Runs in parallel with user's query (during streaming)
3. `findRelevantMemories()` uses Sonnet to rank topic files by relevance
4. Up to 5 topic files attached as `relevant_memories` messages
5. Filters: skip recently-surfaced memories, tool reference docs

### Memory Freshness

Memories > 1 day old get staleness warnings:
```
This memory is 47 days old — verify against current code
```

The model is instructed:
```
Memory records can become stale. Before answering based solely on memory,
verify that the memory is still correct by reading current files. If a
recalled memory conflicts with current information, trust what you observe
now — and update or remove the stale memory.
```

## Team Memory Sync

**Source:** `src/services/teamMemorySync/index.ts`

### Infrastructure

- Subdirectory: `~/.claude/projects/.../memory/team/`
- Requires OAuth + first-party Anthropic auth
- Repo-scoped via git remote hash

### Sync Semantics

| Operation | Behavior |
|-----------|----------|
| Pull | Server wins (overwrites local) |
| Push | Delta upload (only changed content hashes) |
| Delete | Don't propagate (server preserves, next pull restores) |

### Security

- Path validation with `realpath()` symlink resolution
- Rejects `..`, null bytes, URL-encoded traversals
- Max file size: 250KB per entry
- Max PUT body: 200KB (batches split into sequential PUTs)

## Promotion Workflow

Memories can graduate through the hierarchy via the `/remember` skill:

```
auto-memory → CLAUDE.local.md → CLAUDE.md → team memory
```

The `/remember` skill:
1. Gathers all memory layers
2. Classifies each entry by appropriate layer
3. Identifies cleanup opportunities (duplicates, outdated, conflicts)
4. Presents proposals grouped by action type
5. Waits for explicit user approval before making changes

## Auto-Dream (Nightly Consolidation)

**Source:** `src/services/autoDream/`

For assistant mode (KAIROS feature):
1. Appends to daily log: `logs/YYYY/MM/YYYY-MM-DD.md`
2. Nightly `/dream` skill consolidates into topic files
3. MEMORY.md rebuilt from consolidated topics
4. Triggered after 24+ hours since last dream and 5+ sessions

## What NOT to Save (Enforced in Prompt)

The extraction agent prompt explicitly excludes:

```
- Code patterns, architecture, file paths → derived by reading the code
- Git history, who-changed-what → git log / git blame are authoritative
- Debugging solutions → the fix is in the code; commit message has context
- Anything already documented in CLAUDE.md files
- Ephemeral task details: in-progress work, temporary state
```

Even when the user explicitly asks to save these, the system pushes back: "ask what was *surprising* or *non-obvious* — that is the part worth keeping."
