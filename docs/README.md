# Documentation

Source analysis and architecture documentation for Claude Code's internals.

## General

| Document | Description |
|----------|-------------|
| [OVERVIEW.md](OVERVIEW.md) | High-level overview: tech stack, directory structure, core architecture layers, key flows |
| [APP_ARCHITECTURE.md](APP_ARCHITECTURE.md) | Application layer: boot sequence, query engine, tool system, permissions, state management, configuration |
| [TOOL_PIPELINE.md](TOOL_PIPELINE.md) | Tool type system, assembly pipeline, execution flow, permission integration, deferred tool loading |
| [SKILLS_SYSTEM.md](SKILLS_SYSTEM.md) | Skill system: bundled, disk-based, and MCP-sourced reusable workflows with tool restrictions |
| [MCP_INTEGRATION.md](MCP_INTEGRATION.md) | Model Context Protocol: configuration, connections, OAuth, tool/command/resource wrapping |

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
