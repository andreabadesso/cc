# Prompting

Deep analysis of Claude Code's prompting architecture — how it instructs the model, defends against injection, constrains rogue agents, manages context, and optimizes for cache hits.

## Documents

| Document | Focus |
|----------|-------|
| [SYSTEM_PROMPT.md](SYSTEM_PROMPT.md) | System prompt construction, section hierarchy, dynamic boundaries, CLAUDE.md injection, and the full assembly pipeline |
| [AGENT_PROMPTING.md](AGENT_PROMPTING.md) | How subagents, teammates, coordinators, and built-in agent types are prompted — with exact prompt text |
| [TOOL_PROMPTING.md](TOOL_PROMPTING.md) | Tool description generation, conditional prompting, safety protocols, and the patterns tools use to steer model behavior |
| [PROMPT_DEFENSE.md](PROMPT_DEFENSE.md) | Prompt injection defenses, rogue agent containment, trust boundaries, content sandboxing, and the layered security model |
| [CONTEXT_MANAGEMENT.md](CONTEXT_MANAGEMENT.md) | Context window management, auto-compaction, tool result budgeting, cache breakpoints, thinking mode, and message normalization |
| [PATTERNS.md](PATTERNS.md) | Reusable prompting patterns and hidden gems — the techniques worth stealing for your own agent systems |
