# Implementation Principles

English | [中文](README.zh.md)

Per-domain reference pages explaining how the DeepSeek Harness modules work internally: data structures, state machines, algorithms, and cross-package wiring. These pages complement [architecture.md](../architecture.md) (the ordered map of behavior) and [subsystems/](../subsystems/README.md) (type definitions and semantics). For per-package contracts and config, see the owning package README.

| Page | Covers |
|---|---|
| [core-agent-loop.md](core-agent-loop.md) | The agent driver, session event log, tool registry, tool execution pipeline, system-prompt assembly, scoped-context primitive, plan mode, todo, compaction, and human interaction (commands, approval, user-questions, permission presets) |
| [capability-execution.md](capability-execution.md) | Shell, subprocess, terminal, filesystem, LSP, code-runtime, sandbox, E2B remote sandbox, and workspace registry |
| [llm-and-models.md](llm-and-models.md) | LLM adapter registry, streaming, block assembly, DeepSeek and pi-ai providers, retry policy, token-meter |
| [tools-and-services.md](tools-and-services.md) | Skill provider registry, subagent delegation, workflow engine, background jobs, goal tracking, schedule, web search/fetch, attachment storage, message feedback, and spill policy |
| [session-storage.md](session-storage.md) | Session persistence (JSONL and SQLite backends), projection engine, projection cache, telemetry, session titles, session query with FTS5, and the domain-KV storage seam |
| [composition-runtime.md](composition-runtime.md) | Boot process, profiles and bundles, agent presets, dynamic plugin extensions, layered settings, credential references, hook bridges, and MCP client |
| [api-client-platform.md](api-client-platform.md) | Typert type-graph system, JSON-RPC SDK, ACP server, API gateway, web server, client module system, UI slot composition, and the infrastructure utilities |