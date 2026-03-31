# Development Practices

An analysis of the engineering practices, patterns, and design decisions in Claude Code's source code. These documents are written from the perspective of learning from a production-grade, well-architected system.

## Documents

| Document | Focus |
|----------|-------|
| [OVERVIEW.md](OVERVIEW.md) | Trunk-based development, feature flag system (GrowthBook + Bun bundle), migration system, configuration hierarchy |
| [PERMISSION_AND_SECURITY.md](PERMISSION_AND_SECURITY.md) | Permission modes, ML-based auto-approval, sandbox isolation, enterprise MDM enforcement, environment variable security |
| [ERROR_HANDLING_AND_RESILIENCE.md](ERROR_HANDLING_AND_RESILIENCE.md) | Retry strategies, circuit breakers, context overflow recovery, fails-open vs fails-closed philosophy |
| [STATE_AND_CONCURRENCY.md](STATE_AND_CONCURRENCY.md) | Custom state store, file-based multi-agent coordination, AsyncLocalStorage isolation, streaming pipeline |
| [STARTUP_AND_PERFORMANCE.md](STARTUP_AND_PERFORMANCE.md) | Lazy loading, parallel initialization, API preconnect, dead code elimination, token budget management |
| [ARCHITECTURE_AND_TYPE_SYSTEM.md](ARCHITECTURE_AND_TYPE_SYSTEM.md) | Tool/command/plugin/skill extensibility, Zod patterns, discriminated unions, dependency injection via context |

## Cross-Cutting Themes

Several principles appear consistently across all subsystems:

1. **Explicit over implicit** -- Permission modes, failure modes, concurrency safety: everything is declared, not inferred.
2. **Lazy by default** -- Modules, schemas, services, and connections load on first use. Eager loading requires justification from profiling data.
3. **Files as coordination primitive** -- Multi-agent communication, task management, and configuration all use the filesystem. No IPC, no shared memory, no coordination server.
4. **Type safety at boundaries, trust internally** -- Zod validates all external input. Internal code trusts the types.
5. **Remote control without deploys** -- Feature flags (GrowthBook), managed settings (MDM), and analytics sink killswitches allow behavior changes without shipping code.
6. **Measure, then optimize** -- Startup profiling (0.5% sampling), token budget tracking, and cost persistence provide data-driven optimization insights.
