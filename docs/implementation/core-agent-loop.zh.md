# 核心 Agent 循环 — 实现原理

[English](core-agent-loop.md) | 中文

阐述 Agent 驱动引擎、会话日志、工具注册表和提示词装配的内部实现。类型定义参见 [subsystems/core.md](../subsystems/core.md) 和 [subsystems/session.md](../subsystems/session.md)。

## Agent 循环驱动 (`dsh-agent-loop`)

驱动引擎是一个带有 `wakeRequested` 锁存标志的相位机（`idle | maintenance | running`）。在空闲时发送的唤醒始终打开一个回合；在队列排空时发送的锁存唤醒会被抑制。`turn()` 循环打开 `turn/start`，从 Agent 收件箱取下一个排队输入，然后执行 `step()`：通过 `ctx.systemPrompt.assemble()` 组装系统提示词和工具模式，运行 `agent/pre-step` 瀑布流（监听器可以拒绝或重写已取出的消息），根据折叠的日志头构建冻结的模型请求，通过 `agent/request` 瀑布流分发到 `ctx.llm.stream()`，将块缓冲为 `assistant/chunk` 事件，追加 `assistant/message`，然后通过并行/排他调度器分发工具调用。调度器采用先提交后重新分类的策略：工具调用被分为并行组和排他组，排他调用形成屏障，已中止的调用记录合成的 `TOOL_ABORTED_BEFORE_DISPATCH` 结果以保证重放的有效性。`max-tokens` 在每个回合内是粘性的。当模型没有结果且没有下一步输入可用时，`agent/turn-stopping` 串行运行（无 `next()`）。拆卸顺序为：取消 → `whenIdle()` → `scope.dispose()` → 分离 Agent → 分离会话。拆卸在发布前注册，因此创建失败时可以回滚。

关键源码：`packages/core/agent-loop/src/agent.ts:246-330`（回合循环），`packages/core/agent-loop/src/tool-calls.ts:121-246`（调度器）。

## 会话事件日志 (`dsh-session`)

`Session` 是一个仅追加的事件溯源日志。`Session.append` 深度冻结并验证事件数据，分配连续的 `seq` 编号，推入日志数组，然后通过存储拥有的钩子发布。重入由 `appending` 标志保护。表面层（`SurfaceManager`）投影模型可见的消息：产生消息的事件携带 `surfaceOp`（追加或替换），`deriveMessages()` 从表面节点增量投影冻结的消息，在 `replaceGeneration` 变更时重建。`requestHeader()` 折叠从已记录的 `request/header` 事件中增量推导出规范请求头。会话分叉需要在一个开放回合之外的边界，并将日志前缀复制到子会话中。

关键源码：`packages/core/session/src/index.ts:604-655`（追加边界），`packages/core/session/src/surface.ts:398-460`（SurfaceManager）。

## 工具注册表和执行流水线 (`dsh-tools`)

工具运行时是一个三阶段流水线：`prepare`（物化并冻结参数，运行 `tools/pre-execute` 瀑布流 → 允许/拒绝/询问，执行单调的 `guard()` 检查，取消检查），`dispatch`（在 `tools/execute` 瀑布流中包装执行，将调用方信号与包装器替换融合），`finalize`（运行 `tools/post-execute` 瀑布流，物化内容，发送 `tools/result`）。每次执行的状态存储在 WeakMap 中（`deferredContexts`、`cancellationStates`、`contentFinalizers`）。工具可见性是有作用域限制的：`view()` 将链式限制应用于继承的表面，豁免作用域自身的层，并仅对非原生展示模式追加保留的 `run_code` 传输。`run_code` 名称不能被注册或限制。代码模式执行在策略检查之前折叠非 `run_code` 的模型直接调用名称。

关键源码：`packages/core/tools/src/index.ts:1459-1507`（准备门禁），`packages/core/tools/src/index.ts:1532-1560`（融合信号分发）。

## 系统提示词装配 (`dsh-system-prompt`)

基于 Cordis `ScopedLayers` 的作用域 `PromptLayer` 持有节、上下文提供者和变量提供者。`assemble()` 合并全局和作用域链层（最近者优先），通过提供者解析模板变量，通过 `systemPrompt.tools(...)` 提供者收集工具模式（带有分离的参数），应用可配置的工具排序，然后运行 `system-prompt/assemble` 瀑布流。标记为 `complete: true` 的节在瀑布流之后恢复为唯一节。严格的 `{{variable}}` 插值：未知、未定义或格式错误的引用会抛出异常。节按 `order` 升序连接（身份在 -100，角色在 0，工具指南在 100-199）。

关键源码：`packages/core/system-prompt/src/index.ts:467-542`（装配），`packages/core/system-prompt/src/index.ts:258-295`（插值）。

## 作用域上下文 (`dsh-scope`)

作用域是一个不透明的 `object` 键。`scopeParents` WeakMap 存储带有循环检测的一次性绑定嵌套。`scopeTarget(base, key)` 构建一个 Cordis 事件载体，其过滤器允许未标记的监听器以及标记了该键或任何祖先的监听器——事件沿链向上流动，从不向下。`createScope` 创建一个 `ctx.plugin(noop)` 纤维并扩展一个 `kScope` 标签。`ScopedLayers` 拥有一个全局层以及按需创建的精确作用域覆盖层；注册通过 `effect()` 从调用上下文推导可见性，`merge`/`chainLayers` 实现遮蔽（最远祖先优先，最近者胜出）。空的作用域层会被回收。

关键源码：`packages/core/scope/src/index.ts:137-147`（createScope），`packages/core/scope/src/store.ts:226-266`（ScopedLayers.effect）。

## Agent 注册表 (`dsh-agent`)

`AgentRegistry` 存储一个 `Map<SessionId, AgentEntry>`。发起者归属通过两个嵌套的 `AsyncLocalStorage` 存储流转，带有引用计数的 Promise 排空。`enter`/`announce` 生命周期实现延迟分离创建：同步的 `agent/created` 抛出会否决发布，在 `announcing` 期间请求的分离会延迟到分发完成。`Inbox` 是一个一次性重放的投影：构造函数从种子长度重放 `agent/inbox/spliced` 事件，每次变更在修改活跃状态之前持久地提交一个规范化的剪接。`agentEvents` 将 Agent 主体融入事件负载和作用域载体。

关键源码：`packages/core/agent/src/index.ts:450-509`（enter/announce 生命周期），`packages/core/agent/src/inbox.ts:139-193`（先提交剪接后修改）。

## 守卫插件

`dsh-guard-timeout-policy` 包装 `tools/execute`：读取 `ctx.tools.get(name, agent)?.timeoutMs`，设置一个 `deadline(exec.signal, timeoutMs, TOOL_TIMEOUT)`，交换信号，委托 `next()`，然后在 `finally` 中恢复上游信号。仅当它自己的定时器触发时，才将工具的中止结果替换为结构化的 `TOOL_TIMEOUT` 结果。`dsh-guard-repeat-tool-reminder` 通过规范化的参数键在按 Agent 的 `WeakMap` 中计算每个工具的连续重复调用次数，并在达到阈值时将提醒消息添加到后执行决策的 `additionalContexts` 中。

## 上下文插件

`dsh-agent-instructions` 加载兼容 AGENTS.md 的工作区指令。基线通过带有 `source.kind: 'agent-instructions'` 的 `user/message` 进入。活跃状态从表面节点上的可见指令 `changes` 折叠而来。协调过程探测候选作用域（用户全局、从项目到 cwd 的祖先、被触碰路径的后代），使用按会话的版本缓存跳过未更改的文件。文件触碰按 Agent 排队序列化投影；在开放步骤期间的触碰被缓冲到 `step/end`。`dsh-time-context` 在接受的 `agent/pre-step` 中注入时钟读数，带有可配置的刷新间隔。`dsh-tmux-context` 每个回合通过 `ctx.shell` 运行一个 `tmux display-message` 命令，通过将 `#{pane_tty}` 与控制终端匹配来证明进程确实存在于 `$TMUX_PANE` 中。

## 计划模式和待办事项 (`dsh-plan-mode`, `dsh-tool-todo`)

计划状态从会话日志折叠而来：最后一个 `plan/mode` 事件胜出。待处理的意图暂存在 `WeakMap<Session, {active, narrate}>` 中，并在接受的回合内 `agent/pre-step` 中追加。`exit_plan_mode` 工具要求活跃模式加上计划标题，并使用 `userQuestions` 通道进行审查。`todo_write` 验证修剪后的唯一内容，并强制执行 `in_progress` 规则（最多一个，或根据配置至少一个）。执行将 `todo/write` 追加到所属 Agent 的会话中。重放是最后写入胜出；投影在 `turn/start` 时清除。

## 压缩 (`dsh-compaction`, `dsh-compaction-basic`)

压缩缝定义了一个抽象的 `CompactionEngine`，将表面区间替换为一个摘要节点。`dsh-compaction-basic` 是可重放感知的后端：`selectCompactableRange` 从尾部向后遍历定价区间以保留 `retainTokens`，然后扩展直到切割点达到工具配对平衡。`compactSurfaceRegion` 运行一个事务：只读验证，`compaction/start`（锁定），摘要化后的稳定性检查，然后提交 `compaction/summary` 加上带有 `surfaceOp: {op:'replace', start, end}` 和 `sourceEventSeqs` 的替换 `user/message`，最后 `compaction/end`。摘要器重放最后一次路由请求的系统提示词、工具和消息前缀，以便一次性调用复用提供者的 KV 缓存。`dsh-compaction-tool-result-pruner` 将过大的工具结果替换为持久表面中的头/尾摘要。

## 人类交互

`dsh-commands` 注册人类斜杠命令，采用作用域层合并（全局 + Agent 作用域遮蔽）。`execute` 生成单调递增的 `CommandId`，在处理程序之前追加 `command/run`，在完成后追加 `command/done`。`dsh-user-approval` 将审计对 `approval/asked`/`approval/decided` 追加到请求会话。`never` 策略在服务中的任何分发之前就被决定，因此前置注册的门禁监听器无法超越它。`dsh-user-questions` 在询问边界验证调用方：Agent 必须是确切的活跃实例和一个注册表根（子 Agent 不能向人类提问）。`dsh-permission-presets` 从折叠的开关状态（沙箱模式 + 审批策略）推导出有效预设，仅通过其规范设置器写入更改的开关。