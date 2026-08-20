# LLM 与模型 — 实现原理

[English](llm-and-models.md) | 中文

阐述 LLM 服务、适配器、流式传输和 Token 计量器的内部实现。`Message`/`ContentBlock` 词汇参见 [subsystems/llm-streaming.md](../subsystems/llm-streaming.md)，Token 计量器类型参见 [subsystems/token-meter.md](../subsystems/token-meter.md)。

## LLM 运行时 (`dsh-llm`)

`LlmRuntime` 是核心服务（`ctx.llm`）。`registerAdapter(providers, adapter)` 是一个全有或全无的效果：它准备路由（按路由捕获 `providerInfo` 和 `providerRetryPolicy`），原子提交，并发出 `llm/adapters-updated`。`stream(options)` 运行 `ctx.waterfall('llm/stream', …, next → adapterStream)`。`adapterStream` 实现将选择和迭代失败规范化为终端 `finish` 块，在提前返回时关闭迭代器，并剥离历史提供者属于另一个适配器的重放状态。`prepareCall` 在一次性的深度冻结句柄中捕获注册和已解析的配置；分发仅允许一次，配置必须匹配。循环构建的请求携带 `markAgentLoopRequest` 并以深度冻结形式到达。`BlockAssembler` 在 `max-tokens` 截断下丢弃工具调用，并在步骤中修剪重放信封——块和重放元数据从一次保留/丢弃的过程中推导出来。Token 计数是不相交的（缓存字段是分开的）。

## DeepSeek 适配器 (`dsh-llm-deepseek`)

`DeepSeekAdapter` 是直接 fetch 的 OpenAI 兼容适配器。连接事实按请求通过一个 thunk 解析，按设置快照记忆化，在错误的活跃节上回退到最后一次成功的值。API 密钥按请求通过可选的 `ctx.credentials` 或环境解析，由 `assertUsableApiKey` 验证。凭据和端点作为一个快照传输，因此过期的密钥永远不能与新的 URL 配对。`installSettingsSection` 交换当前的设置节，`onChange` 仅在重试策略更改时重新注册（通过原子的 `registration.replace`）。一个稳定的 `AbortSignal` 跨越 fetch 和正文读取；调用方中止 → `ABORTED`，空闲看门狗 → `TIMEOUT`。适配器将 SSE 块转换为 `StreamChunk` 事件，从缓存字段中减去 `prompt_tokens`。

## pi-ai 适配器 (`dsh-llm-pi-ai`)

`PiAiAdapter` 是基于 `@earendil-works/pi-ai` 的通用库适配器。配置文件从按原始快照记忆化的 `providers` 字典解析；每个操作捕获一个不可变快照，因为 `Models.streamSimple()` 延迟解析提供者。凭据由 harness 解析并作为 `apiKey` 传入。路由集、displayName 和重试策略构成 `registrationFacts`；更改会原子重新注册。引用缺失凭据的配置抛出 `MISSING_CREDENTIAL`（永远不会回退到环境 pi-ai 密钥）。当前构建无法使用的重放状态会降级为带有警告的提供者无关内容。

## 重试策略 (`dsh-llm-retry`)

此插件监听 `agent/request-error` 瀑布流。当没有策略、代码不可重试或会话日志重放显示重试次数 ≥ `maxRetries` 时，它委托给 `next()`。幸存的重试通过扫描匹配回合、步骤、提供者和 policyKey 的先前 `llm/retry` 事件来计数——会话日志是计数器，而非内存。每个计划的重试在可取消的抖动延迟之前追加 `llm/retry`，然后追加 `llm/retry-started`。等待之前的持久化：延迟中的崩溃永远不会重新运行一个从未报告的回合。`always` 模式首先委托给下游恢复。延迟尊重提供者的 `providerRetryAfterMs`（普通模式拒绝超过最大值，always 模式回退到局部延迟）。

## Token 计量器 (`dsh-token-meter`)

`TokenMeter` 是一个可重放感知的服务，测量请求压力和模型可见的表面。按会话的 `ReplayState` 通过折叠 `session.events` 赶上进度：跟踪 `request/header`、`step/start`/`step/end` 嵌套和表面 Token 增量。锚点建立在每个 `assistant/message` 处：仅当当前规范头等于锚点的头且提供者总数 ≥ 完整启发式定价的锚点时，才复用提供者使用量；否则用估计值锚定。`_estimateProviderAssistant` 使用 `BlockAssembler` 从引用的 `assistant/chunk` seq 重新组装提供者输出，以避免对截断后的持久表面进行定价。启发式方法是固定的 4 字符/Token——对 CJK/JSON 故意低估，但锚定使其远离占用计算。不相交的使用量桶确保推理已经在 `outputTokens` 中。该服务注册 `tokenUsage`、`contextPressure` 和 `contextBreakdown` 投影单元。