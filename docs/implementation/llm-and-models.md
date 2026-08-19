# LLM and Models — Implementation

English | [中文](llm-and-models.zh.md)

How the LLM service, adapters, streaming, and token-meter work internally. See [subsystems/llm-streaming.md](../subsystems/llm-streaming.md) for the `Message`/`ContentBlock` vocabulary and [subsystems/token-meter.md](../subsystems/token-meter.md) for token-meter types.

## LLM runtime (`dsh-llm`)

`LlmRuntime` is the central service (`ctx.llm`). `registerAdapter(providers, adapter)` is an all-or-nothing effect: it prepares routes (capturing `providerInfo` and `providerRetryPolicy` per route), commits atomically, and emits `llm/adapters-updated`. `stream(options)` runs `ctx.waterfall('llm/stream', …, next → adapterStream)`. The `adapterStream` implementation normalizes selection and iteration failures to a terminal `finish` chunk, closes the iterator on early return, and strips replay state whose historical provider is owned by another adapter. `prepareCall` captures registration plus resolved config in a one-shot deep-frozen handle; dispatch is allowed once, and config must match. Loop-built requests carry `markAgentLoopRequest` and arrive deep-frozen. `BlockAssembler` drops tool-calls under `max-tokens` truncation and prunes the replay envelope in step — blocks and replay metadata derive from one keep/drop pass. Token counts are disjoint (cache fields are separate).

## DeepSeek adapter (`dsh-llm-deepseek`)

`DeepSeekAdapter` is a direct-fetch OpenAI-compatible adapter. Connection facts resolve per request through a thunk, memoized per settings snapshot with last-good fallback on a bad live section. The API key is resolved per request through optional `ctx.credentials` or environment, validated by `assertUsableApiKey`. Credentials and endpoint travel as one snapshot so a stale key can never pair a new URL. `installSettingsSection` swaps the current settings section, and `onChange` re-registers only when the retry policy changes (via atomic `registration.replace`). One stable `AbortSignal` spans fetch and body reads; caller abort → `ABORTED`, idle watchdog → `TIMEOUT`. The adapter translates SSE chunks into `StreamChunk` events, subtracting `prompt_tokens` from cache fields.

## pi-ai adapter (`dsh-llm-pi-ai`)

`PiAiAdapter` is a generic library-backed adapter over `@earendil-works/pi-ai`. Profiles resolve from a `providers` dict memoized per raw snapshot; each operation captures one immutable snapshot because `Models.streamSimple()` resolves the provider lazily. Credentials are resolved by the harness and passed as `apiKey`. Route set, displayName, and retry policy form `registrationFacts`; changes re-register atomically. A profile that names a missing credential throws `MISSING_CREDENTIAL` (never falls back to ambient pi-ai keys). Replay state unusable by the current build degrades to provider-neutral content with a warning.

## Retry policy (`dsh-llm-retry`)

This plugin listens on the `agent/request-error` waterfall. It defers to `next()` when no policy exists, the code is not retryable, or the session-log replay shows retry count ≥ `maxRetries`. Surviving retries are counted by scanning prior `llm/retry` events matching turn, step, provider, and policyKey — the session log is the counter, not memory. Each scheduled retry appends `llm/retry` before the cancellable jittered delay, then `llm/retry-started`. Durability before wait: a crash mid-delay can never re-run a turn it never reported. `always` mode delegates to downstream recovery first. Delays respect the provider's `providerRetryAfterMs` (normal mode refuses over-max, always mode falls back to local delay).

## Token meter (`dsh-token-meter`)

`TokenMeter` is a replay-aware service measuring request pressure and model-visible surface. Per-session `ReplayState` catches up through `session.events` by fold: tracks `request/header`, `step/start`/`step/end` nesting, and surface token deltas. The anchor is established at each `assistant/message`: provider usage is reused only when the current canonical header equals the anchor's and provider total ≥ the full heuristic priced anchor; otherwise the estimate anchors. `_estimateProviderAssistant` re-assembles provider output from cited `assistant/chunk` seqs with `BlockAssembler` to avoid pricing the post-truncation durable surface. The heuristic is fixed 4 chars/token — deliberately underpriced for CJK/JSON, but anchoring keeps it out of occupancy calculations. Disjoint usage buckets ensure reasoning is already inside `outputTokens`. The service registers `tokenUsage`, `contextPressure`, and `contextBreakdown` projection units.