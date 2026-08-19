# Core Agent Loop — Implementation

English | [中文](core-agent-loop.zh.md)

How the agent driver, session log, tool registry, and prompt assembly work internally. See [subsystems/core.md](../subsystems/core.md) for the `Agent` and `AgentLoop` type definitions, and [subsystems/session.md](../subsystems/session.md) for the `SessionEventMap` vocabulary.

## Agent loop driver (`dsh-agent-loop`)

The driver is a phase machine (`idle | maintenance | running`) with a latched `wakeRequested` flag. A wake sent while idle always opens a turn; a latched wake sent while the queue is draining is suppressed. The `turn()` loop opens `turn/start`, claims the next queued input from the agent inbox, then runs a `step()`: assemble the system prompt and tool schemas through `ctx.systemPrompt.assemble()`, run the `agent/pre-step` waterfall (listeners may reject or rewrite the claimed messages), build a frozen model request from the folded log header, dispatch through `agent/request` waterfall into `ctx.llm.stream()`, buffer chunks as `assistant/chunk` events, append `assistant/message`, then dispatch tool calls through the parallel/exclusive scheduler. The scheduler commits-before-reclassify: tool calls are classified into parallel and exclusive groups, exclusive calls form barriers, and aborted calls record synthetic `TOOL_ABORTED_BEFORE_DISPATCH` results so replay stays valid. `max-tokens` is sticky per turn. When the model owes nothing and no next-step input is available, `agent/turn-stopping` runs serially (no `next()`). Teardown is reverse-order: cancel → `whenIdle()` → `scope.dispose()` → detach session → detach agent. Teardown is registered before publication so creation failure rolls back.

Key source: `packages/core/agent-loop/src/agent.ts:246-330` (turn loop), `packages/core/agent-loop/src/tool-calls.ts:121-246` (scheduler).

## Session event log (`dsh-session`)

A `Session` is an append-only, event-sourced log. `Session.append` deep-freezes and validates the event data, assigns a contiguous `seq` number, pushes to the log array, then publishes through store-owned hooks. Reentrancy is guarded by an `appending` flag. The surface layer (`SurfaceManager`) projects model-visible messages: message-producing events carry a `surfaceOp` (append or replace), and `deriveMessages()` incrementally projects frozen messages from surface nodes, rebuilding on a `replaceGeneration` bump. A `requestHeader()` fold incrementally derives the canonical request header from logged `request/header` events. Session fork requires a boundary outside an open turn and copies the log prefix into a child session.

Key source: `packages/core/session/src/index.ts:604-655` (append boundary), `packages/core/session/src/surface.ts:398-460` (SurfaceManager).

## Tool registry and execution pipeline (`dsh-tools`)

The tool runtime is a three-stage pipeline: `prepare` (materialize and freeze arguments, run `tools/pre-execute` waterfall → allow/deny/ask, apply monotonic `guard()` checks, cancellation checks), `dispatch` (wrap execution in `tools/execute` waterfall, fuse caller signal with wrapper replacements), and `finalize` (run `tools/post-execute` waterfall, materialize content, emit `tools/result`). Per-execution state lives in WeakMaps (`deferredContexts`, `cancellationStates`, `contentFinalizers`). Tool visibility is scoped: `view()` applies chain restrictions to the inherited surface, exempts the scope's own layer, and appends the reserved `run_code` transport only for non-native presentation modes. The `run_code` name cannot be registered or restricted. Code-mode execution collapses non-`run_code` model-direct names before policy checks.

Key source: `packages/core/tools/src/index.ts:1459-1507` (prepare gate), `packages/core/tools/src/index.ts:1532-1560` (fused signal dispatch).

## System prompt assembly (`dsh-system-prompt`)

Scoped `PromptLayer`s over Cordis `ScopedLayers` hold sections, context providers, and variable providers. `assemble()` merges global and scope-chain layers (nearest wins), resolves template variables via providers, collects tool schemas through `systemPrompt.tools(...)` providers with detached parameters, applies configurable tool ordering, then runs the `system-prompt/assemble` waterfall. A section marked `complete: true` is restored after the waterfall as the sole section. Strict `{{variable}}` interpolation: unknown, undefined, or malformed references throw. Sections are concatenated by ascending `order` (identity at -100, persona at 0, tool guidance at 100-199).

Key source: `packages/core/system-prompt/src/index.ts:467-542` (assemble), `packages/core/system-prompt/src/index.ts:258-295` (interpolate).

## Scoped context (`dsh-scope`)

A scope is an opaque `object` key. The `scopeParents` WeakMap stores nesting with cycle-checked one-shot binding. `scopeTarget(base, key)` builds a Cordis event carrier whose filter admits untagged listeners plus listeners tagged with the key or any ancestor — events flow up the chain, never down. `createScope` mints a `ctx.plugin(noop)` fiber and extends it with a `kScope` tag. `ScopedLayers` owns a global layer plus lazily-created exact-scope overlays; registrations derive visibility from the calling context via `effect()`, and `merge`/`chainLayers` implement shadowing (farthest-ancestor-first, nearest wins). Empty scoped layers are reclaimed.

Key source: `packages/core/scope/src/index.ts:137-147` (createScope), `packages/core/scope/src/store.ts:226-266` (ScopedLayers.effect).

## Agent registry (`dsh-agent`)

`AgentRegistry` stores a `Map<SessionId, AgentEntry>`. Initiator attribution flows through two nested `AsyncLocalStorage` stores with ref-counted promise drains. The `enter`/`announce` lifecycle implements deferred-detach creation: synchronous `agent/created` throw vetoes publication, and a detach requested during `announcing` defers until dispatch unwinds. `Inbox` is a replay-once projection: the constructor replays `agent/inbox/spliced` events from the seed length, and every mutation commits one normalized splice durably before mutating live state. `agentEvents` fuses the agent subject into the event payload and scope carrier.

Key source: `packages/core/agent/src/index.ts:450-509` (enter/announce lifecycle), `packages/core/agent/src/inbox.ts:139-193` (splice-commit-before-mutate).

## Guard plugins

`dsh-guard-timeout-policy` wraps `tools/execute`: reads `ctx.tools.get(name, agent)?.timeoutMs`, arms a `deadline(exec.signal, timeoutMs, TOOL_TIMEOUT)`, swaps the signal, delegates `next()`, then restores the upstream signal in `finally`. Only if its own timer fired replaces the tool's abort-result with a structured `TOOL_TIMEOUT` result. `dsh-guard-repeat-tool-reminder` counts consecutive repeat calls per tool via canonicalized argument keys in a per-agent `WeakMap`, and prepends reminder messages onto `additionalContexts` of post-execute decisions when a threshold is hit.

## Context plugins

`dsh-agent-instructions` loads AGENTS.md-compatible workspace instructions. Baseline enters via a `user/message` with `source.kind: 'agent-instructions'`. Live state is folded from visible instruction `changes` over surface nodes. Reconciliation probes candidate scopes (user-global, ancestors from project to cwd, touched-path descendants) using a per-session version cache to skip unchanged files. File touches queue serialized per-agent projections; touches during an open step are buffered until `step/end`. `dsh-time-context` injects a clock reading at accepted `agent/pre-step` with a configurable refresh interval. `dsh-tmux-context` runs one `tmux display-message` command through `ctx.shell` per turn, proving the process lives in `$TMUX_PANE` by matching `#{pane_tty}` against the controlling terminal.

## Plan mode and todo (`dsh-plan-mode`, `dsh-tool-todo`)

Plan state is folded from the session log: the last `plan/mode` event wins. Pending intents are parked in a `WeakMap<Session, {active, narrate}>` and appended from an accepted in-turn `agent/pre-step`. The `exit_plan_mode` tool requires active mode plus a plan heading and uses a `userQuestions` channel for review. `todo_write` validates trimmed unique content and enforces the `in_progress` discipline (at most one, or >=1 depending on config). Execution appends `todo/write` to the owning agent's session. Replay is last-write-wins; the projection is cleared by `turn/start`.

## Compaction (`dsh-compaction`, `dsh-compaction-basic`)

The compaction seam defines an abstract `CompactionEngine` that replaces a surface span with one summary node. `dsh-compaction-basic` is the replay-aware backend: `selectCompactableRange` walks the priced tail backwards to retain `retainTokens`, then extends until the cut is tool-pairing balanced. `compactSurfaceRegion` runs one transaction: read-only validation, `compaction/start` (lock), stability check after summarization, then commit with `compaction/summary` plus the replacement `user/message` carrying `surfaceOp: {op:'replace', start, end}` and `sourceEventSeqs`, then `compaction/end`. The summarizer replays the last routed request's system prompt, tools, and message prefix so the one-shot call reuses the provider KV cache. `dsh-compaction-tool-result-pruner` replaces oversized tool results with head/tail snippets in the durable surface.

## Human interaction

`dsh-commands` registers human slash-commands with scoped-layer merging (global + agent-scoped shadows). `execute` mints a monotonically increasing `CommandId`, appends `command/run` before the handler, and `command/done` after settlement. `dsh-user-approval` appends audit pairs `approval/asked`/`approval/decided` to the requesting session. The `never` policy is decided by the service before any dispatch so a prepend-registered gate listener cannot outrun it. `dsh-user-questions` validates the caller at the ask boundary: the agent must be the exact live instance and a registry root (child agents cannot ask humans). `dsh-permission-presets` derives the effective preset from folded knob state (sandbox mode + approval policy), writing changed knobs only through their canonical setters.