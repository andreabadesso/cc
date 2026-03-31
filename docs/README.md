# Documentation

Source analysis and architecture documentation for Claude Code's internals.

## General

| Document | Description |
|----------|-------------|
| [OVERVIEW.md](OVERVIEW.md) | High-level overview: tech stack, directory structure, core architecture layers, key flows |
| [APP_ARCHITECTURE.md](APP_ARCHITECTURE.md) | Application layer: boot sequence, query engine, tool system, permissions, state management, configuration |
| [STREAMING_STATE_MACHINE.md](STREAMING_STATE_MACHINE.md) | The core streaming + tool execution loop: state transitions, concurrent execution, error recovery, abort handling |
| [TOOL_PIPELINE.md](TOOL_PIPELINE.md) | Tool type system, assembly pipeline, execution flow, permission integration, deferred tool loading |
| [SKILLS_SYSTEM.md](SKILLS_SYSTEM.md) | Skill system: bundled, disk-based, and MCP-sourced reusable workflows with tool restrictions |
| [MCP_INTEGRATION.md](MCP_INTEGRATION.md) | Model Context Protocol: configuration, connections, OAuth, tool/command/resource wrapping |

## Infrastructure

| Document | Description |
|----------|-------------|
| [SANDBOX.md](SANDBOX.md) | OS-level command sandboxing: Seatbelt (macOS), bubblewrap+seccomp (Linux), filesystem/network restrictions |
| [HOOKS.md](HOOKS.md) | 26 hook events, 6 hook types, permission integration, policy enforcement, async execution |
| [MEMORY_SYSTEM.md](MEMORY_SYSTEM.md) | Auto-memory extraction, 4 memory types, MEMORY.md index, relevance prefetch, team sync |
| [COST_TRACKING.md](COST_TRACKING.md) | Pricing tiers, cache economics, token estimation, rate limits, overage detection, per-session persistence |
| [IDE_BRIDGE.md](IDE_BRIDGE.md) | VS Code/JetBrains bridge: V1 (WebSocket) and V2 (SSE) transports, JWT auth, session management |
| [COMPACTION.md](COMPACTION.md) | 6 compaction strategies, actual summarization prompts, boundary management, microcompact, cache editing |
| [AUTH_AND_ENTERPRISE.md](AUTH_AND_ENTERPRISE.md) | OAuth 2.0 PKCE flow, keychain integration, workspace trust, enterprise MDM, policy limits, env var security |
| [FEATURE_FLAGS.md](FEATURE_FLAGS.md) | Two-layer system: 90 compile-time Bun flags + 65+ runtime GrowthBook flags, latching, conditional prompting |
| [PLUGIN_EXTENSIBILITY.md](PLUGIN_EXTENSIBILITY.md) | Plugin manifest, custom skills/agents/commands, hook registration, security boundaries, discovery pipeline |
| [BATCH_WORKFLOWS.md](BATCH_WORKFLOWS.md) | /batch 3-phase orchestration, /simplify 3-agent review, fork cache optimization, progress tracking |

## Terminal UI

| Document | Description |
|----------|-------------|
| [TUI_ENGINE.md](TUI_ENGINE.md) | Rendering engine: 60 FPS game loop, Yoga layout, double-buffered screen, frame diff, ANSI serialization |
| [COMPONENTS.md](COMPONENTS.md) | Ink components (Box, Text, ScrollBox, etc.), hooks (useInput, useAnimationFrame, etc.), and event system |

## Multi-Agent Orchestration

| Document | Description |
|----------|-------------|
| [multi-agent/](multi-agent/README.md) | Overview: spawn layer, execution backends, coordination infrastructure |
| [multi-agent/AGENT_TOOL.md](multi-agent/AGENT_TOOL.md) | AgentTool: four execution paths (subagent, teammate, isolation, background) |
| [multi-agent/SPAWNING.md](multi-agent/SPAWNING.md) | Backend detection and fallback: Tmux, iTerm2, in-process |
| [multi-agent/TEAMS.md](multi-agent/TEAMS.md) | Team lifecycle, config, member management |
| [multi-agent/COMMUNICATION.md](multi-agent/COMMUNICATION.md) | File-based mailbox system with lockfile concurrency |
| [multi-agent/TASK_SYSTEM.md](multi-agent/TASK_SYSTEM.md) | Task coordination: JSON storage, locking, dependency graphs, ownership |
| [multi-agent/IN_PROCESS_RUNNER.md](multi-agent/IN_PROCESS_RUNNER.md) | In-process teammate execution loop with AsyncLocalStorage isolation |
| [multi-agent/WORKTREES.md](multi-agent/WORKTREES.md) | Git worktree isolation for agents |

## Prompting

| Document | Description |
|----------|-------------|
| [prompting/](prompting/README.md) | Index and overview of the prompting architecture |
| [prompting/SYSTEM_PROMPT.md](prompting/SYSTEM_PROMPT.md) | System prompt construction: section hierarchy, dynamic boundaries, CLAUDE.md injection, assembly pipeline |
| [prompting/AGENT_PROMPTING.md](prompting/AGENT_PROMPTING.md) | How subagents, teammates, coordinators, and built-in agent types are prompted |
| [prompting/TOOL_PROMPTING.md](prompting/TOOL_PROMPTING.md) | Tool description generation, conditional prompting, safety protocols, behavioral steering |
| [prompting/PROMPT_DEFENSE.md](prompting/PROMPT_DEFENSE.md) | 8-layer defense model: injection defenses, rogue agent containment, trust boundaries, content sandboxing |
| [prompting/CONTEXT_MANAGEMENT.md](prompting/CONTEXT_MANAGEMENT.md) | Context window management, auto-compaction, tool result budgeting, cache optimization, thinking mode |
| [prompting/PATTERNS.md](prompting/PATTERNS.md) | 15 reusable prompting patterns and hidden gems worth stealing for your own agent systems |

## Development Practices

| Document | Description |
|----------|-------------|
| [development_practices/](development_practices/README.md) | Index and cross-cutting themes |
| [development_practices/OVERVIEW.md](development_practices/OVERVIEW.md) | Trunk-based development, feature flags (GrowthBook + Bun bundle), migrations |
| [development_practices/ARCHITECTURE_AND_TYPE_SYSTEM.md](development_practices/ARCHITECTURE_AND_TYPE_SYSTEM.md) | Layered architecture, Zod patterns, discriminated unions, dependency injection |
| [development_practices/PERMISSION_AND_SECURITY.md](development_practices/PERMISSION_AND_SECURITY.md) | Six permission modes, ML-based auto-approval, sandbox isolation, MDM |
| [development_practices/ERROR_HANDLING_AND_RESILIENCE.md](development_practices/ERROR_HANDLING_AND_RESILIENCE.md) | Retry strategies, circuit breakers, fail-open vs fail-closed |
| [development_practices/STATE_AND_CONCURRENCY.md](development_practices/STATE_AND_CONCURRENCY.md) | Custom state store, file-based coordination, AsyncLocalStorage, streaming |
| [development_practices/STARTUP_AND_PERFORMANCE.md](development_practices/STARTUP_AND_PERFORMANCE.md) | Lazy loading, parallel init, API preconnect, token budget management |
