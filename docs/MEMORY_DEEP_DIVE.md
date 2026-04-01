# Memory System: Deep Dive

A complete trace of how Claude Code creates, stores, fetches, and consolidates auto-memories from source code in `src/`.

---

## Directory Map of Memory-Related Files

```
src/
  memdir/
    memdir.ts               — Core prompt builder + loadMemoryPrompt()
    memoryTypes.ts          — Four memory type definitions + all prompt text constants
    memoryScan.ts           — scanMemoryFiles() + formatMemoryManifest()
    findRelevantMemories.ts — Async Sonnet-powered relevance ranking
    memoryAge.ts            — Freshness warnings injected at recall time
    paths.ts                — isAutoMemoryEnabled(), getAutoMemPath(), path resolution
    teamMemPaths.ts         — Team memory path validation + symlink safety
    teamMemPrompts.ts       — Combined (auto + team) prompt builder

  services/
    extractMemories/
      extractMemories.ts    — Background forked-agent extraction engine
      prompts.ts            — Extraction agent's system prompt template
    autoDream/
      autoDream.ts          — Nightly consolidation (KAIROS mode)
      config.ts             — isAutoDreamEnabled()
      consolidationLock.ts  — Mutex via mtime + session-counting
      consolidationPrompt.ts — The /dream agent's prompt
    teamMemorySync/
      index.ts              — Server pull/push (delta upload, ETag)
      watcher.ts            — File-watcher that triggers sync
      secretScanner.ts      — Blocks API keys before upload
      types.ts              — Zod schemas for API responses

  utils/
    backgroundHousekeeping.ts — initExtractMemories() + initAutoDream() at startup
    attachments.ts            — startRelevantMemoryPrefetch(), readMemoriesForSurfacing()
    memory/
      types.ts                — UI-layer MemoryType enum
      versions.ts             — projectIsInGitRepo() helper

  skills/bundled/
    remember.ts             — The /remember skill prompt

  commands/
    memory/index.ts         — Entry point for /memory command

  query/
    stopHooks.ts            — handleStopHooks() — triggers extraction fire-and-forget

  query.ts                  — Main query loop; startRelevantMemoryPrefetch() called here
  constants/prompts.ts      — System prompt assembly; calls loadMemoryPrompt()
```

---

## 1. Full Data Flow

```
Application startup
  └─ startBackgroundHousekeeping()         [src/utils/backgroundHousekeeping.ts:34]
       ├─ initExtractMemories()
       └─ initAutoDream()

Session start
  └─ getSystemPromptForMode()              [src/constants/prompts.ts]
       └─ systemPromptSection('memory', () => loadMemoryPrompt())  [line 495]
            └─ loadMemoryPrompt()          [src/memdir/memdir.ts:419]
                 ├─ isAutoMemoryEnabled()
                 ├─ ensureMemoryDirExists()
                 └─ buildMemoryLines() → reads MEMORY.md synchronously
                      → injected into system prompt (cached)

User submits query
  └─ query()                               [src/query.ts]
       └─ startRelevantMemoryPrefetch()    [line 301]
            └─ async getRelevantMemoryAttachments()
                 ├─ scanMemoryFiles()      [reads frontmatter of all .md files]
                 └─ selectRelevantMemories()  [sideQuery to Sonnet, JSON response]
                      → returns up to 5 RelevantMemory {path, mtimeMs}

       [model streams response + executes tools]

Post-tool iteration consume point
  └─ pendingMemoryPrefetch.settledAt !== null    [src/query.ts:1599]
       └─ filterDuplicateMemoryAttachments() + readMemoriesForSurfacing()
            → yield AttachmentMessage {type: 'relevant_memories'}
            → injected as <system-reminder> into the conversation

Model produces final response (no pending tool calls)
  └─ handleStopHooks()                     [src/query/stopHooks.ts:65]
       ├─ void executeExtractMemories(context, appendSystemMessage)  [line 149]
       │    └─ executeExtractMemoriesImpl()
       │         ├─ gate checks (agentId, feature flag, enabled, remote, inProgress, throttle)
       │         └─ runExtraction()
       │              ├─ hasMemoryWritesSince()  → skip if main agent already wrote this turn
       │              ├─ scanMemoryFiles()       → pre-inject manifest for fork
       │              ├─ buildExtractAutoOnlyPrompt(newMessageCount, existingMemories)
       │              └─ runForkedAgent({maxTurns: 5, skipTranscript: true})
       │                   [fork: read files → write topic file → update MEMORY.md]
       │                   └─ extractWrittenPaths() → createMemorySavedMessage()
       │                        → appendSystemMessage("Saved N memories")
       └─ void executeAutoDream(context, appendSystemMessage)  [line 155]
            └─ time gate + session gate + lock → runForkedAgent()
```

---

## 2. How Memories Are Extracted

### Trigger

Every time the model finishes a turn (no pending tool calls), `handleStopHooks()` fires in `src/query/stopHooks.ts` line 148:

```typescript
void extractMemoriesModule!.executeExtractMemories(
  stopHookContext,
  toolUseContext.appendSystemMessage,
)
```

This is fire-and-forget (`void`). The conversation is not blocked.

### Gate Checks

`executeExtractMemoriesImpl()` in `src/services/extractMemories/extractMemories.ts` performs these checks, cheapest first:

1. `context.toolUseContext.agentId` must be absent — main agent only, not subagents (line 532)
2. Feature gate `tengu_passport_quail` must be active (line 536)
3. `isAutoMemoryEnabled()` must be true (line 545)
4. `getIsRemoteMode()` must be false (line 550)
5. If `inProgress`, stash context in `pendingContext` and return — only the latest context is kept (line 557)
6. Throttle check via feature value `tengu_bramble_lintel` (default 1) — skip if fewer than N eligible turns have passed (line 378)
7. `hasMemoryWritesSince()` — if the main agent already wrote to the auto-mem path this turn, skip the fork and advance the cursor (line 348)

### The Forked Agent

`runForkedAgent()` is called with:

```typescript
{
  maxTurns: 5,          // hard cap prevents rabbit holes
  skipTranscript: true, // no transcript pollution
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  cacheSafeParams,      // shares parent's prompt cache prefix → cache hits
}
```

The fork receives the same message history as the main conversation (efficient via prompt cache sharing). The extraction prompt is injected as the last user message.

The extraction prompt (`src/services/extractMemories/prompts.ts`, `buildExtractAutoOnlyPrompt()`) contains:
- The four memory type definitions
- The `WHAT_NOT_TO_SAVE_SECTION` (hard exclusions)
- Two-step save instructions (write topic file → update `MEMORY.md`)
- The manifest of existing memory files (to prevent duplicate creation)
- Efficiency strategy: "Turn 1 — issue all Read calls in parallel; Turn 2 — issue all Write/Edit calls in parallel. Do not interleave reads and writes."

### Cursor Tracking and Stashing

`lastMemoryMessageUuid` (closure-scoped) marks the last processed message. `countModelVisibleMessagesSince()` counts only `user` and `assistant` messages from that cursor — ignoring progress/system/attachment messages.

If a new turn arrives while extraction is `inProgress`, the context is saved to `pendingContext` (overwriting prior stash). In `runExtraction()`'s `finally` block (lines 506–521), after completion, if `pendingContext` is set it fires a trailing run.

### Tool Permissions for the Fork

`createAutoMemCanUseTool()` restricts the fork to:

| Tool | Restriction |
|------|-------------|
| `FileReadTool`, `GrepTool`, `GlobTool` | Unrestricted |
| `BashTool` | Read-only commands only (ls, find, cat, stat, wc, head, tail) |
| `FileEditTool` / `FileWriteTool` | Only if path is inside `isAutoMemPath()` |
| All other tools | Denied |

---

## 3. How Memories Are Stored

### File Format

Each memory is a Markdown file with YAML frontmatter:

```markdown
---
name: database testing policy
description: Integration tests must use real database, not mocks
type: feedback
---

Integration tests must hit a real database, not mocks.

**Why:** Prior incident where mock/prod divergence masked a broken migration.
**How to apply:** When writing or reviewing test code in this repo.
```

Valid types: `user`, `feedback`, `project`, `reference` (defined as a const tuple in `src/memdir/memoryTypes.ts` line 14). Invalid `type:` fields degrade gracefully via `parseMemoryType()`.

### Directory Structure

`getAutoMemPath()` in `src/memdir/paths.ts` resolves in order:

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var (Cowork/multi-tenant spaces)
2. `autoMemoryDirectory` from settings.json (policy/flag/local/user sources; project settings excluded for security)
3. Default: `{CLAUDE_CONFIG_DIR}/projects/{sanitized-git-canonical-root}/memory/`

`{CLAUDE_CONFIG_DIR}` defaults to `~/.claude`. The canonical git root is used (via `findCanonicalGitRoot`) so all worktrees of the same repo share one memory directory — no per-worktree silos.

Team memories live at: `{auto_mem_path}/team/`

### Two-Step Write Process

**Step 1:** Fork writes the topic file (e.g., `feedback_testing.md`) via `FileWriteTool` or `FileEditTool`. The file is only writable because `isAutoMemPath(filePath)` returns true — this is a security carve-out in `filesystem.ts`.

**Step 2:** Fork updates `MEMORY.md` with a one-line index entry:

```markdown
- [Database testing policy](feedback_testing.md) — integration tests must use real DB
```

`MEMORY.md` has no frontmatter and contains no memory content — it is an index only.

### Post-Write Notification

After the fork completes, `extractWrittenPaths()` collects written paths. `MEMORY.md` is filtered out. The remaining paths go to `createMemorySavedMessage()` → `appendSystemMessage?.()`, displaying "Saved N memories" in the UI.

---

## 4. How Memories Are Fetched

Two retrieval mechanisms run independently.

### A. Session-Start: MEMORY.md Injected into System Prompt

At session start, `loadMemoryPrompt()` is called from `src/constants/prompts.ts` line 495. Call chain:

```
getSystemPromptForMode()
  └─ systemPromptSection('memory', () => loadMemoryPrompt())   [cached — not re-read on streaming]
       └─ loadMemoryPrompt()    [src/memdir/memdir.ts:419]
            ├─ isAutoMemoryEnabled()
            ├─ ensureMemoryDirExists()
            └─ buildMemoryLines() → fs.readFileSync(MEMORY.md) synchronously
                 → appended under "## MEMORY.md" in the system prompt
```

`MEMORY.md` is subject to:
- Truncation at **200 lines** (`MAX_ENTRYPOINT_LINES`)
- Truncation at **25KB** (`MAX_ENTRYPOINT_BYTES`) at the last newline before the cap
- A warning appended when truncation fires

`systemPromptSection` caches the result so re-renders during streaming don't re-read the file.

### B. Async Prefetch: Top-5 Relevant Topic Files Mid-Turn

Gated on feature flag `tengu_moth_copse`. Starts at `src/query.ts` line 301:

```typescript
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
)
```

`startRelevantMemoryPrefetch()` in `src/utils/attachments.ts` line 2361:
1. Checks `isAutoMemoryEnabled()` and `tengu_moth_copse`
2. Extracts the last real user message (skips `isMeta` messages)
3. Rejects single-word prompts (insufficient context for ranking)
4. Checks session-total byte budget (`RELEVANT_MEMORIES_CONFIG.MAX_SESSION_BYTES`) — if exceeded, no more surfacing this session
5. Fires `getRelevantMemoryAttachments()` as a non-blocking async promise

`getRelevantMemoryAttachments()` calls `findRelevantMemories()` in `src/memdir/findRelevantMemories.ts`:

1. `scanMemoryFiles()` reads all `.md` files recursively (excluding `MEMORY.md`), parses frontmatter (first 30 lines only), returns `MemoryHeader[]` sorted newest-first, capped at 200 files
2. Filters out `alreadySurfaced` paths — dedup across conversation turns
3. Issues a `sideQuery` to Sonnet with `SELECT_MEMORIES_SYSTEM_PROMPT` and a JSON schema response: `{selected_memories: string[]}` — max 256 tokens, max 5 files
4. The selector also receives `recentTools` (successfully used tools since last turn) so it doesn't re-surface reference docs for tools already being used correctly

**Consume point** (`src/query.ts` lines 1599–1613): After each tool-use iteration, `pendingMemoryPrefetch.settledAt !== null` is polled. If settled, results are wrapped via `filterDuplicateMemoryAttachments()` and `readMemoriesForSurfacing()` (reads files with truncation), then yielded as `AttachmentMessage {type: 'relevant_memories'}` and injected as `<system-reminder>` into the message stream.

### Freshness Warnings

`src/memdir/memoryAge.ts` — `memoryFreshnessText()` generates plain text for memories older than 1 day:

> "This memory is 47 days old. Memories are point-in-time observations..."

The staleness text is prepended to the attachment content header via `memoryHeader()`.

---

## 5. Memory Types in Detail

Defined in `src/memdir/memoryTypes.ts` line 14:

```typescript
export const MEMORY_TYPES = ['user', 'feedback', 'project', 'reference'] as const
```

| Type | Scope (team mode) | Description |
|------|-------------------|-------------|
| `user` | Always private | Personal profile: role, preferences, expertise |
| `feedback` | Private by default; team if project-wide convention | How to approach work — corrections and validated choices |
| `project` | Strongly bias toward team | Ongoing initiatives, decisions, deadlines |
| `reference` | Usually team | Pointers to external systems (Linear, Grafana, etc.) |

Two prompt variants exist:
- `TYPES_SECTION_INDIVIDUAL` — single memory directory, no scope guidance
- `TYPES_SECTION_COMBINED` — private + team directories, with `<scope>` XML tags per type

`WHAT_NOT_TO_SAVE_SECTION` and `TRUSTING_RECALL_SECTION` are injected into the **main agent's** system prompt (not the extraction fork). They govern recall behavior: verify file/function claims before asserting them, treat architecture snapshots as frozen in time.

---

## 6. Auto-Dream: Nightly Consolidation (KAIROS Mode)

Also fired from `handleStopHooks()` line 155, fire-and-forget.

Gate conditions in `src/services/autoDream/autoDream.ts`:
- Non-KAIROS / non-remote mode only
- 24+ hours since `lastConsolidatedAt` (lock file mtime check)
- 5+ sessions touched since last consolidation
- 10-minute cooldown on the session scan (avoids repeated `stat` calls)
- `tryAcquireConsolidationLock()` prevents multiple processes from dreaming simultaneously

Runs a forked agent with the same `createAutoMemCanUseTool()` restrictions as extraction.

On success: appends `createMemorySavedMessage()` with `verb: 'Improved'`.
On failure: `rollbackConsolidationLock()` rewinds the mtime so the time-gate passes again next turn.

In KAIROS mode, memories are appended to daily log files at `logs/YYYY/MM/YYYY-MM-DD.md` instead of MEMORY.md. The path is expressed as a pattern in the prompt (not today's literal date) to preserve the prompt cache across midnight rollovers.

---

## 7. Team Memory Sync

`src/services/teamMemorySync/watcher.ts` monitors the team memory directory via filesystem events and triggers sync on change.

**Pull:** GET to `/api/claude_code/team_memory?repo={owner/repo}`. Server wins — local files are overwritten. Uses ETags (`lastKnownChecksum`) for conditional GETs.

**Push:** Delta upload — only keys whose local `sha256:` hash differs from `serverChecksums` are uploaded. Batched into sequential PUTs when body exceeds `MAX_PUT_BODY_BYTES` (200KB).

**Security** in `validateTeamMemKey()` and `validateTeamMemWritePath()`:
- Null bytes, URL-encoded traversals, Unicode NFKC normalization attacks, backslashes, absolute paths — all rejected
- `realpathDeepestExisting()` resolves symlinks on the deepest existing ancestor
- Dangling symlinks explicitly detected via `lstat()` after `realpath()` ENOENT

**Secret scanning** (`secretScanner.ts`) runs before every push — blocks files containing API keys or credentials from reaching the server.

---

## 8. Implementation Details and Edge Cases

### Prompt Cache Sharing

The forked extraction agent shares the parent's `CacheSafeParams` (system prompt + message prefix). It gets prompt cache hits — its marginal cost is only the new messages analyzed plus its output, not a full context re-tokenization.

### `tengu_moth_copse` (Skip-Index Mode)

When this flag is enabled, the two-step write process is simplified — the fork only writes the topic file. `MEMORY.md` injection in the system prompt is replaced by the async prefetch mechanism, making `MEMORY.md` optional.

### Compact Boundary Interaction

`collectSurfacedMemories()` scans messages for `relevant_memories` attachments. After a compact, those attachments are gone from the compacted transcript, naturally resetting the `alreadySurfaced` dedup set and the session byte budget — memories can be re-surfaced after compaction.

### Cowork (SDK / Multi-Tenant)

`CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` redirects memory to a space-scoped mount. `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` injects additional policy text into all memory prompts. Memory mechanics still apply when a custom system prompt is in use, provided `hasAutoMemPathOverride()` is true.

### Git Worktree Handling

`findCanonicalGitRoot()` returns the common git directory so all worktrees of the same repo use the same memory path.

---

## Key Files Reference

| File | Role |
|------|------|
| `src/memdir/memdir.ts` | `loadMemoryPrompt()`, `buildMemoryLines()`, `truncateEntrypointContent()` |
| `src/memdir/memoryTypes.ts` | All four type definitions, prompt text, `WHAT_NOT_TO_SAVE_SECTION`, `TRUSTING_RECALL_SECTION` |
| `src/services/extractMemories/extractMemories.ts` | Full extraction engine: gate checks, forked agent, cursor tracking, stashing |
| `src/services/extractMemories/prompts.ts` | Extraction agent's prompt |
| `src/memdir/memoryScan.ts` | `scanMemoryFiles()` and `formatMemoryManifest()` — shared by extraction and recall |
| `src/memdir/findRelevantMemories.ts` | Sonnet-powered relevance ranking via `sideQuery` |
| `src/utils/attachments.ts` | `startRelevantMemoryPrefetch()`, `readMemoriesForSurfacing()`, `collectSurfacedMemories()` |
| `src/query/stopHooks.ts` | Trigger point — `void executeExtractMemories(...)` line 149 |
| `src/memdir/paths.ts` | `isAutoMemoryEnabled()`, `getAutoMemPath()`, `isAutoMemPath()` |
| `src/memdir/memoryAge.ts` | Freshness warnings and staleness caveat |
| `src/services/autoDream/autoDream.ts` | Nightly consolidation for KAIROS mode |
| `src/utils/backgroundHousekeeping.ts` | Startup initialization |
