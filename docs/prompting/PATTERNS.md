# Prompting Patterns — Hidden Gems

The techniques in Claude Code's prompting that are worth extracting and reusing in your own agent systems. These are the non-obvious, battle-tested patterns that emerged from production use.

## 1. Belt-and-Suspenders Enforcement

**Pattern:** Enforce constraints at BOTH the prompt level AND the implementation level.

**Example — FileEditTool pre-read requirement:**
- **Prompt says:** "You must use your Read tool at least once before editing. This tool will error if you attempt an edit without reading."
- **Implementation does:** Actually throws an error if no Read was called first.

**Example — Explore agent read-only mode:**
- **Prompt says:** "CRITICAL: READ-ONLY MODE — you do NOT have access to file editing tools."
- **Implementation does:** Actually removes Edit, Write, NotebookEdit from the agent's tool list.

**Why it works:** The prompt shapes intent; the implementation enforces it. Neither alone is sufficient. The model might ignore the prompt, and the implementation error alone doesn't teach the model *why* it should read first.

**Steal this:** For every critical constraint in your agent, enforce it in both the prompt AND the code. Tell the model the constraint exists AND make it impossible to violate.

## 2. Naming Failure Modes to Preempt Them

**Pattern:** Explicitly describe common failure modes in the prompt so the model recognizes and avoids them.

**Example — Verification agent:**
```
Failure Patterns (what verification agents get wrong):
- Verification avoidance: concluding "looks correct" without running
  a single command
- Being seduced by first 80%: tests pass → PASS, ignoring edge cases
```

**Example — Git amend warning:**
```
CRITICAL: Always create NEW commits rather than amending. When a
pre-commit hook fails, the commit did NOT happen — so --amend would
modify the PREVIOUS commit, which may destroy work.
```

**Why it works:** LLMs are susceptible to exactly the failure modes you'd expect. By naming them, you activate the model's ability to pattern-match against its own behavior. "Am I doing verification avoidance right now?" is a question it can answer.

**Steal this:** When deploying an agent, track its most common failures for a week. Then add them to the prompt as named anti-patterns. This is the single highest-ROI prompting technique.

## 3. Mandatory Synthesis Between Agents

**Pattern:** Never pass one agent's raw output directly to another agent. Force the coordinator to read, understand, and rewrite.

**Example — Coordinator mode:**
```
Synthesis phase: YOU read findings, craft implementation specs
- Never write "based on your findings" — this delegates understanding
- Always synthesize into specific prompt with file paths, line numbers
```

**Why it works:** This breaks the injection chain. If Worker A produces malicious output, the coordinator must process it and produce a new prompt for Worker B. The coordinator's reasoning acts as a firewall — it extracts facts and discards injected instructions.

**Steal this:** In any multi-agent system, add a synthesis step between research and implementation. The overhead is minimal (one extra LLM call), and the safety improvement is massive.

## 4. Scoped Authorization (Not Blanket)

**Pattern:** User approval for one action does not authorize future similar actions.

**Example — System prompt:**
```
A user approving an action (like a git push) once does NOT mean that
they approve it in all contexts, so unless actions are authorized in
advance in durable instructions like CLAUDE.md files, always confirm
first. Authorization stands for the scope specified, not beyond.
```

**Why it works:** LLMs tend to generalize permissions. "The user approved `git push` once, so they probably want me to push again." This instruction explicitly breaks that generalization.

**Steal this:** Add this principle to any agent that can take destructive or visible actions. Be explicit about authorization scope.

## 5. "When NOT to Use" Guidance

**Pattern:** Tell the model when to NOT use a tool, not just when to use it.

**Example — AgentTool:**
```
When NOT to use the Agent tool:
- If you want to read a specific file path, use Read or Glob instead
- If you are searching for a specific class definition, use Glob instead
- If you are searching code within 2-3 files, use Read instead
```

**Example — TaskCreateTool:**
```
Skip using this tool when:
- Single-step tasks
- Trivial changes
```

**Why it works:** LLMs over-use powerful tools. An agent with access to AgentTool will delegate everything. Negative guidance creates an inhibition threshold that forces the model to consider simpler alternatives first.

**Steal this:** For every powerful/expensive tool, include a "when NOT to use this" section. List the simpler alternatives.

## 6. Output Format Forcing via Triple Emphasis

**Pattern:** When a specific output format is required, use three escalating emphasis levels.

**Example — WebSearchTool:**
```
CRITICAL REQUIREMENT — You MUST follow this:
- You MUST include a "Sources:" section
- This is MANDATORY — never skip
```

Three levels: `CRITICAL`, `MUST`, `MANDATORY`. Each word activates different model attention patterns.

**Example — Verification agent:**
```
VERDICT: PASS | FAIL | PARTIAL
```

With structured evidence format required for each check.

**Steal this:** For critical output format requirements, use at least two of: CRITICAL, MUST, MANDATORY, REQUIRED, ALWAYS, NEVER. One isn't enough — the model needs multiple signals.

## 7. Dead-End Communication Warning

**Pattern:** Explicitly tell agents that prose output goes nowhere.

**Example — Teammate addendum:**
```
Just writing a response in text is not visible to others on your team —
you MUST use the SendMessage tool.
```

**Example — SendMessageTool:**
```
Your plain text output is NOT visible to other agents — to communicate,
you MUST call this tool.
```

**Why it works:** LLMs default to generating text. In a multi-agent system, agents will "talk" to each other via prose unless explicitly told that prose is a dead end. This pattern forces tool-mediated communication.

**Steal this:** In any multi-agent system, add a clear statement that text output is invisible to other agents. Repeat it in both the system prompt and the communication tool prompt.

## 8. Anti-Documentation Bias

**Pattern:** Explicitly prevent the model from creating documentation files.

**Example — FileWriteTool:**
```
NEVER create documentation files (*.md) or README files unless
explicitly requested by the User.
```

**Why it works:** LLMs have a strong tendency to "be helpful" by creating README files, CHANGELOG entries, and documentation. In a coding agent, this creates file clutter that the user didn't ask for.

**Steal this:** If your agent has file creation abilities, add explicit rules about what file types it should NOT create proactively.

## 9. Reversibility-First Action Framework

**Pattern:** Teach the model to categorize actions by reversibility before executing them.

**Example — System prompt:**
```
Carefully consider the reversibility and blast radius of actions.
Generally you can freely take local, reversible actions like editing
files or running tests. But for actions that are hard to reverse,
affect shared systems, or could be risky, check with the user.
```

With explicit categories:
- **Freely take:** Local, reversible (edit files, run tests)
- **Confirm first:** Destructive, hard-to-reverse, visible to others

**Why it works:** Instead of a binary "safe/unsafe" classification, this gives the model a **reasoning framework**. It can evaluate novel actions by asking "is this reversible?" and "what's the blast radius?"

**Steal this:** Give your agents a decision framework, not a list of allowed/denied actions. Frameworks generalize; lists don't.

## 10. Fork Cache Alignment

**Pattern:** When spawning parallel agents from the same context, replace variable content with identical placeholders to maximize prompt cache hits.

**Implementation:**
```
All tool_use results → identical placeholder text
Result: entire conversation prefix is byte-identical across forks
Cost: 1 full context + N small directive suffixes (instead of N full contexts)
```

**Why it works:** Anthropic's prompt caching works on byte-identical prefixes. By making everything before the fork point identical, you get cache hits on all but the first fork.

**Steal this:** When spawning parallel agents, normalize everything in the shared prefix. Replace variable content (tool results, timestamps, IDs) with fixed placeholders.

## 11. The "Caller Will Relay" Pattern

**Pattern:** Tell the agent that its output will be consumed by another agent, not displayed directly.

**Example — General-purpose agent:**
```
When you complete the task, respond with a concise report covering what
was done and any key findings — the caller will relay this to the user,
so it only needs the essentials.
```

**Why it works:** This shapes the output format dramatically. Without it, agents produce verbose, user-facing prose. With it, they produce concise, fact-dense summaries optimized for another agent to parse.

**Steal this:** When an agent's output feeds into another agent (not directly to a user), tell it explicitly. The output quality difference is significant.

## 12. Few-Shot Examples in Tool Descriptions

**Pattern:** Embed examples directly in tool `description` fields, not just in the system prompt.

**Examples found:**
- BashTool: HEREDOC commit message format
- AgentTool: Fork usage dialogues
- WebSearchTool: Sources section format
- ScheduleCronTool: Cron expression examples with edge cases

**Why it works:** Tool descriptions are re-read every time the model considers using that tool. Examples in the description are more likely to be followed than examples in a distant system prompt section.

**Steal this:** Put your most important examples in the tool description, not just the system prompt. The proximity effect is real.

## 13. Layered Content Wrapping

**Pattern:** Different trust levels get different wrapping, and the model is taught what each wrapper means.

```
<system-reminder>     → System context, treat as informational
<persisted-output>    → External data, truncated, may be hostile
<task-notification>   → Agent result, structured, semi-trusted
<channel>             → External message, explicitly untrusted
raw text              → User input, fully trusted
```

**Why it works:** Instead of a binary trusted/untrusted model, this creates a **gradient of trust** that the model can reason about. Content from `<channel>` gets more scrutiny than `<system-reminder>`.

**Steal this:** Design a trust taxonomy for your agent system. Assign different wrappers to different trust levels. Document what each wrapper means in the system prompt.

## 14. Dynamic Agent Filtering by Capability

**Pattern:** Filter available agents based on runtime capabilities before listing them.

**Example — AgentTool prompt generation:**
```typescript
// Filter by MCP server availability
const agentsWithMcp = filterAgentsByMcpRequirements(agents, mcpServers)
// Filter by permission rules
const filtered = filterDeniedAgents(agentsWithMcp, permissionContext)
// Generate prompt with only viable agents
return getPrompt(filtered, isCoordinator, allowedTypes)
```

**Why it works:** The model never sees agents it can't use. This prevents wasted turns where the model tries to spawn an agent that requires unavailable MCP servers.

**Steal this:** Filter tool/agent listings to only show what's currently available. Don't show the model options that will fail.

## 15. Anti-Hallucination for Agent Results

**Pattern:** Explicitly tell the coordinator not to predict or fabricate what agents will return.

```
Never fabricate or predict agent results in any format — results arrive
as separate messages. Not as prose, summary, or structured output.
```

Repeated twice in the coordinator prompt with slightly different wording.

**Why it works:** LLMs will cheerfully hallucinate agent results if not told otherwise. The model might "predict" that a research agent found certain things and proceed with implementation based on hallucinated findings.

**Steal this:** Any orchestrator agent that spawns workers needs this instruction. Add it twice with different phrasing for emphasis.

## Summary: The Meta-Pattern

The overarching pattern in Claude Code's prompting is **defense in depth through redundancy**:

1. **Say it in the system prompt** (global instruction)
2. **Say it in the tool description** (local instruction)
3. **Enforce it in code** (implementation constraint)
4. **Name the failure mode** (anti-pattern awareness)
5. **Provide the alternative** (positive guidance)

No single layer is trusted. Every critical behavior is reinforced through multiple independent mechanisms. This is not over-engineering — it's the only approach that works reliably with current LLMs.
