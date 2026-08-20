# 工具与服务 — 实现原理

[English](tools-and-services.md) | 中文

阐述 Skill、子代理、工作流、后台任务、目标追踪、定时提醒、Web 访问、附件存储、消息反馈和溢出策略的内部实现。类型定义参见 [subsystems/skills.md](../subsystems/skills.md)、[subsystems/subagent.md](../subsystems/subagent.md)、[subsystems/workflow.md](../subsystems/workflow.md)、[subsystems/jobs.md](../subsystems/jobs.md)、[subsystems/goal.md](../subsystems/goal.md)、[subsystems/schedule.md](../subsystems/schedule.md)、[subsystems/web.md](../subsystems/web.md)、[subsystems/attachment.md](../subsystems/attachment.md)、[subsystems/feedback.md](../subsystems/feedback.md) 和 [subsystems/spill.md](../subsystems/spill.md)。

## Skill 注册表 (`dsh-skill`, `dsh-skill-filesystem`)

`SkillRegistry` 将注册存储在调用上下文的作用域层（`ScopedLayers`）中，因此 Agent 预设的挂载绑定到该作用域。`collect` 合并全局和作用域链层，使用修订键缓存（有界 LRU，最多 2 次重试以应对并发修订变更）。候选排序：较低的 `rank` 然后注册顺序；最近的层直接遮蔽。`registerProvider` 返回确切的效果回收器。`dsh-skill-filesystem` 是本地文件系统提供者：根目录按项目 `.dsh/skills` 与 `.agents/skills`、自定义、用户 `~/.dsh/skills` 与 `~/.agents/skills` 和捆绑包解析，rank 100-600。读取在可用时通过 `ctx.fs` 进行。`WatchManager` 保持引用计数的根状态，使用 chokidar 的 `awaitWriteFinish` 稳定性和微任务聚合的失效。`edit`/`write` 工具的 `fs/observed` 变更在路径可能为 Skill 时触发失效。`dsh-tool-skill` 发布持久会话 Skill 目录，并提供面向模型的 `skill` 加载器工具。目录瀑布流将条目的 SHA-256 摘要与可见的持久目录历史进行比较；变更时发布 `<available_skills>` 替换。直接用户文本中的 `/name` 手势标记触发 `user-invocable` Skill 的确定性指令注入。

## 子代理委托 (`dsh-subagent`)

`SubagentRuntime` 是服务定义：`start()` 验证能力，解析持久描述符，调用提供者，并将发布包装在生命周期观察者中。`SubagentContinuationManager` 保留子 ID，物化激活（子 Agent + 报告工具贡献），并使用 Agent 收件箱作为唯一的回合队列（FIFO）。可持续提供者仅贡献一个分离的创建规范（`prepareContinuable`）。`dsh-subagent-in-process-driver` 是共享驱动：`startInProcessRun` 解析子深度、选项和元数据，在第一个 await 之前捕获委托的策略覆盖，并运行 `parent.ctx.agents.create()`，其设置应用 `appendDelegatedPolicyOverrides`、`applyChildComposition` 和结构化运行时。`dsh-subagent-fork-in-process` 用父会话的已完成回合前缀（通过最后一个 `turn/end` 切片）作为种子。`dsh-subagent-spawn-in-process` 启动一个无种子的全新子会话。`dsh-subagent-acp` 通过 `ctx.subprocess.spawn` 在 ACP 线上生成子进程；拆卸顺序为 stdin EOF → `disposeEofGraceMs` → `terminate()` 并等待整个进程树退出证明。`dsh-subagent-claude-code` 通过官方 Agent SDK 生成 Claude Code，投影到子进程缝上。`dsh-subagent-codex` 生成一个 `codex app-server --stdio` 进程。`dsh-subagent-dsh-sdk` 通过 stdio JSON-RPC 运行完整的 dsh 运行时作为子进程。`dsh-tool-subagent` 是面向模型的委托工具；`dsh-tool-subagent-control` 提供 `send_message`/`interrupt_agent`；`dsh-tool-subagent-report` 安装子作用域的 `report` 工具。

## 工作流引擎 (`dsh-workflow`, `dsh-workflow-worker-thread`)

`WorkflowEngine` 是抽象的服务定义。`WorkflowStopReason` 是一个封闭联合类型（`completed | cancelled | error`）；`WorkflowError.fatal` 驱动组合器规则：`parallel`/`pipeline` 重新抛出致命错误并清空每个项目的失败。`dsh-workflow-worker-thread` 是默认后端：一个全新的 Worker 在可逃逸的 vm 中运行模型编写的 JS。主机端 `WorkerRun.cancel` 发布 Cancel，中止规范的子信号，并设置一个无引用的宽限期。消息准入在第一个死亡信号时关闭。`AgentStart`/`AgentEnd` 门禁确保每个启动的 Agent 恰好有一个结束，为遗留的 Agent 合成主机端 `cancelled` 结束。Worker 端 `WorkflowExecution` 提供 FIFO 并发槽、Agent 总数上限、每个项目的上限，以及钩子（`agent`、`parallel`、`pipeline`、`phase`、`log`）；取消使每个钩子在下一个边界抛出 `CANCELLED`。`dsh-tool-workflow` 是面向模型的工具；`dsh-tool-ralph` 是一个带有有界轮次交接的固定 Ralph 循环工具。

## 后台任务 (`dsh-jobs`, `dsh-jobs-local`)

`JobRegistry` 是抽象的服务定义：`<kind>-N` ID，按所有者会话 ID 隔离的访问控制，首次胜出的结算，以及提交后受控的 `onJobDone` 监听器。`dsh-jobs-local` 是内存后端：`TrackedTask` 记录在 Map 中加上类型计数器；基于 `ScopedLayers` 的作用域分层控制器、监听器和变更通知。`settle` 是首次胜出；`disposeAll` 关闭监听器，取消，强制失败孤儿取消，等待已结算的尾部，然后移除效果。`dsh-tool-jobs` 提供 `job_output`/`job_list`/`job_kill` 工具。`onJobDone` 投递未报告的通知：唤醒 → 空闲时 `owner.followup` 在 `maxConsecutiveWakes` 范围内，否则 `owner.inject`；唤醒在用户来源的收件箱取回时重置。

## 目标追踪 (`dsh-goal`, `dsh-goal-round-driver`)

`GoalService` 是事件溯源的：按会话的 `GoalCache` 增量折叠 `goal/change` 事件。变更构建带有单调修订的完整快照，通过 `session.append` 加上 `sync` 在 `pendingActivation` seq 守卫下提交，然后发出 `goal/changed`。激活（`armed`/`disarmed`）是进程本地的，在未匹配挂起激活 seq 时重新构建为 disarmed。`dsh-goal-round-driver` 驱动自动同会话继续：`agent/status idle`、`goal/changed` 和收件箱事件触发 `drive`，计算下一个回合，并在达到轮次上限或队列失败时调用 `ctx.goals.block`。`dsh-tool-goal` 提供 `get_goal`/`create_goal`/`update_goal` 工具，带有权限策略（直接人类或目标回合）；`blocked` 仅在 `blockedAfterConsecutiveRounds` 之后成功。

## 定时提醒 (`dsh-schedule`)

Agent 作用域的持久化一次性和固定频率提醒，基于会话日志。折叠从 `schedule/change` 事件推导出 `FoldedSchedules`（分叉仅从 `seedLength` 折叠）。`driveOnce` 在 `runScheduleTransaction` 内运行；每个持久决策重置无引用的有界定时器。`dueDecision` 推导最新发生次数而不枚举积压。固定频率下限为 `MIN_EVERY_INTERVAL_SECONDS = 300`。分发使用 `followup` 然后在同步入队后追加 `schedule/change` 分发事件。运行时仅附加到未来的根 Agent，而非现有的或已持久化的。

## Web 访问 (`dsh-web`, `dsh-web-fetch-http`, `dsh-web-search-*`)

`WebRuntime` 是服务定义，具有独立的搜索和抓取提供者注册表。选择规则在执行时从 `available()` 解析（从不按注册顺序）。`dsh-web-fetch-http` 是匿名安全的 HTTP 抓取提供者：每跳手动同源重定向验证，带有跳数预算，content-length 预拒绝 vs 流式字节上限，TextDecoder 在读取前解析。`dsh-web-search-deepseek` 通过 DeepSeek API 使用原生 `web_search_20250305` 服务器工具每次搜索分发一次模型推理。`dsh-web-search-exa` 使用 Exa `/search` API。`dsh-web-search-perplexity` 使用 Perplexity chat-completions 端点。`dsh-tool-web` 提供 `web_search` 和 `web_fetch` 工具；抓取工具通过 Turndown 将 HTML 转换为 Markdown，带有 `MAX_CONVERSION_DEPTH = 512` 防护以应对超线性 DOM 遍历。

## 附件存储 (`dsh-attachment`, `dsh-attachment-local`)

`AttachmentStore` 是抽象缝：`saveImages` 强制执行计数、总字节和媒体类型检查，在首次写入前验证每个输入（完整解码），然后按批次顺序保存。`dsh-attachment-local` 在 `DSH_HOME/attachments/v1` 下存储。保存流程：从完全解码的字节探测尺寸，通过 2 级前缀的 sha256 内容寻址，暂存加 `link` 去重。在引用可见之前，直到 FS 根的每个祖先目录都被 fsync。默认限制：5 MB/图片，20 张图片，100 MB 批次，40 MP。

## 消息反馈 (`dsh-message-feedback`, `dsh-command-feedback`)

`MessageFeedbackService` 在 `message-feedback` 存储领域中为已完成的助手消息存储持久的带版本侧写反馈（评分 + 备注）。`put` 运行完整的读取/比较/写入变更，按会话通过 `operationTails` 序列化，将 `ifVersion` 与当前行比较。会话所有权由持久化身份匹配守卫。`dsh-command-feedback` 提供面向人类的 `/feedback` 命令，追加 `feedback/record` 仅日志事件。

## 溢出策略 (`dsh-spill`, `dsh-spill-local`, `dsh-spill-policy`)

`SpillStore` 是抽象缝：`saveText` 逐字持久化完整内容。`dsh-spill-local` 在私有 `0700` 按进程临时目录下存储文件。`encodeSegment` 是一个无损单射方案（`~XXXX` 转义），适用于每个 UTF-16 字符串。`saveTextFile` 以 `'wx'` `0600` 打开，带有随机 6 字节前缀以防御符号链接投毒。`dsh-spill-policy` 是一个 `tools/post-execute` 转换器：当最终结果的 UTF-8 大小超过 `maxInlineBytes` 时，它将完整文本保存到溢出工件，并将面向模型的结果替换为有界的头/尾预览加上定位器和检索指南。面向模型的手臂和 `tools/code-dispatch-log` 手臂共享一个 `spillReplacement` 实现。失败语义为尽力而为：无会话所有者、无后端或 `saveText` 失败 → 警告并返回原始内联结果。