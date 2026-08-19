# 会话与存储 — 实现原理

[English](session-storage.md) | 中文

阐述会话持久化、投影、遥测、标题、会话查询和领域 KV 存储的内部实现。类型定义参见 [subsystems/session.md](../subsystems/session.md)、[subsystems/persistence.md](../subsystems/persistence.md)、[subsystems/session-projection.md](../subsystems/session-projection.md)、[subsystems/session-telemetry.md](../subsystems/session-telemetry.md)、[subsystems/session-title.md](../subsystems/session-title.md)、[subsystems/session-query.md](../subsystems/session-query.md) 和 [subsystems/storage.md](../subsystems/storage.md)。

## 会话持久化 (`dsh-session-persistence`)

`PersistenceCoordinator` 是共享的写入路径编排。每个会话 ID 有一个 Promise 链序列化器，因此写入永远不会交错。`SessionWriteBehind` 以固定的 `maxDelayMs` 截止定时器缓冲事件；失败的批次被重新前置并暂停自动排空。`SessionPreparations` 管理每个 ID 的条目，具有 `loading | ready | committing | reserved` 阶段，共享的飞行中读取和 LRU 上限的就绪条目。崩溃恢复：`prepareCore` 读取后端的 `StoredPrefix`，为完整的中断回合合成 `interruptedTurnClosers`，并仅丢弃撕裂的尾部。`commitPrepared` 使修复持久化，然后强制重新加载。修订稳定性重试循环回退冷读取。遗留迁移拒绝已退役的事件类型（`request/header-delta`、`mode/set`、`fallback`）并升级预身份的消息事件。

**JSONL 后端**（`dsh-session-persistence-jsonl`）：每个会话一个仅追加的工件，位于 `root/<project>/<encoded-id>/session.jsonl[.zstd]`。`encodeSegment` 将每个不安全的 UTF-16 单元映射到 `~XXXX`（单射，防遍历）。追加 = 打开 `a` → stat → writeFile → fsync；部分失败截断回写入前的大小。物化（POSIX）执行 mkdir + dir-fsync 链，fsync 后的临时文件，通过 `link+unlink` 发布（EEXIST 使第二个进程失败），fsync 父目录。每个追加批次是一个可独立解码的校验和 zstd 帧。崩溃恢复：`scanZstdFrames` 找到完整帧加上撕裂起始；撕裂的最后一帧被前缀解码以挽救完整记录。`commitRepair` = 截断+fsync 然后追加恢复+关闭器。

**SQLite 后端**（`dsh-session-persistence-sqlite`）：模式 `SCHEMA_VERSION = 15`（单调递增；非当前版本拒绝，无迁移），由 `application_id 0x44534850` 守卫。STRICT 表：`persistence_state`（单例存储 UUID），`sessions`（头部 + `incarnation` UUID + `revision` 计数器），`events`（主键 session_id+seq；`data` JSON 文本；可空的 `source_event_seqs` 和 `surface_op` JSON）。`appendBatch` = 一个 BEGIN/COMMIT：插入或替换会话行，插入事件，`revision+1`。`commitRepair` = 一个事务 DELETE `seq>=tornMarker` + 插入关闭器。撕裂尾部语义与 JSONL 匹配：`scanRows` 容忍最后一个 `turn/end` 之后的 seq 间隙，但在已提交前缀内的间隙抛出异常。

## 会话投影 (`dsh-session-projection`, `dsh-session-projection-cache`)

`SessionProjectionMap` 是服务定义。投影单元是 `{key, schema (zod), init(), apply(state, event), view(state), stateVersion}`——纯、同步、纯 JSON。注册表一次性订阅 `session/event`，并对每个事件 `drive()` 每个单元。相同引用返回产生零下游工作。单元是按会话的 WeakMap 条目，从内存日志惰性折叠。`register()` 是一个引用计数的效果。`dsh-session-projection-cache` 在 `session_projcache` 存储领域（v3）上存储持久化检查点。每个活跃会话的写入延迟按事件计数；触发条件包括计数 ≥ `writeEveryEvents`、间隔定时器和两个强制点（`turn/end`、`session/disposed`）。缓存始终滞后，永不领先于日志。冷启动阶梯：`cachedSnapshot` 查看与会话生命周期身份匹配的行；身份不匹配、收缩或版本不匹配 → 从 seq 0 进行一次完整重新读取。

`dsh-session-stats` 注册 `sessionStats` 投影单元：全日志的回合/步骤计数和墙钟时间，不受分页和压缩影响。`step/end`（每个进入的步骤在 `finally` 中发出一次）是被计数的步骤。`tool/result` 通过 `callId` 匹配，带有 `Object.hasOwn` 守卫。

## 会话遥测 (`dsh-session-telemetry`, `dsh-session-telemetry-otel`)

`SessionTelemetry` 定义了逻辑记录词汇和后端（接收器）契约。协调器监听 `session/created`（采用 = 从移交光标通过投影重放）、`session/event` 和 `session/disposed`。`handoffCursor` 是一个模块作用域的 WeakMap，以 Session 为键——光标仅在 `emit` 接受后前进。编辑是 `session-telemetry/record` 瀑布流。`dsh-session-telemetry-otel` 组合 OTel JS SDK，使用 `LoggerProvider` + `BatchLogRecordProcessor` + OTLP/HTTP 导出器。模式：`FULL | FEEDBACK_ONLY | DISABLED`（默认 DISABLED）。DISABLED 不构建 SDK 状态。FULL 组合活跃协调器；FEEDBACK_ONLY 组合按需协调器加上监护检查（`session.events[event.seq] === event`）。`shutdown()` 与截止时间竞争提供者关闭。

## 会话标题 (`dsh-session-title`, `dsh-session-title-llm`)

标题是仅日志的 `session/title` 事件，携带 `{title, messageSeqs, source}`。`get()` = `foldSessionTitle`（最后胜出）。`rename()` 规范化（去除控制字符，折叠空白，UTF-8 字节预算），拒绝空字符串，并固定标题同时取代飞行中的生成。自动节奏使用按会话的 `SessionTitleWorkState`：符合条件的 `user/message` 调度生成；工作仅在确切的主要请求路由被记录后开始。每个会话一个活跃的提供者调用：`supersede` 中止过时的控制器，`assertCurrent` 拒绝来自死修订的完成。`dsh-session-title-llm` 是共享的框架库：`generateSessionTitleWithLlm` 将消息帧化为一个 JSON 文档，按 `maxInputBytes` 检查字节，解析路由，应用截止时间，追加仅日志的 `session/title-llm-request` 事件，并通过 `ctx.llm.stream` 流式传输。`dsh-session-title-first-prompt-llm` 和 `dsh-session-title-all-prompts-llm` 是轻量提供者插件，仅在消息选择节奏上有所不同。

## 会话查询 (`dsh-session-query`, `dsh-session-query-sqlite`)

`SessionQuery` 是服务定义，提供组合的会话历史读取、追踪、过滤和全文搜索。`SessionCorpus` 可选地绑定持久化；`listSessions` 合并持久化的头部与活跃会话（活跃优先）。`load()` 偏好分离的活跃快照并重新检查附加竞争。`tracing.analyzeEventLog` 对每个日志运行一次规范的 `foldSurface`：事件分类为 `current | shadowed | log-only`，通过 `sourceEventSeqs` 遍历替换链。`traceSession` 带有循环检测地遍历 `parentSession` 祖先。

`dsh-session-query-sqlite` 是基于活跃优先语料库的 FTS5 具体后端。FTS 模式 v8 / application id `0x44534851`；版本不匹配则 DROP 所有表并在原地重置。每次搜索是序列化的，然后协调：两次尝试的稳定观察（`listSnapshots` 之前和之后以及活跃 ID 集合必须匹配），然后在一个 `BEGIN IMMEDIATE` 事务中计算索引行的差异。查询 CTE 联合持久化文档与活跃文档。FTS5 `highlight()` 使用私有区域标记；排名：match_count 降序，document_length 升序，时间降序，seq 降序。光标是 base64url `{version, instance, scope, fingerprint, generation, offset}`。查询是双引号转义的单短语——FTS 语法惰性。`dsh-tool-session-query` 提供五个工作区授权的工具；授权在每个观察到的头部上强制执行，而非每个请求。

## 领域 KV 存储 (`dsh-storage`, `dsh-storage-domain`)

`Storage` 是中心枢纽：命名的后端注册表加上挂载的数据表单设施。`dsh-storage-domain` 提供 `DomainFacility`（`ctx.storage.domain`）：模式验证的、发出变更的 KV 领域。`defineDomain` 验证名称和版本。`DomainImpl` 持有权威的内存映射，具有一个写入链（`enqueue`）：每次写入首先等待后端持久化，然后修改内存，然后发出 `domain/changed`，因此被拒绝的写入不会修改内存。`close()`：拒绝新写入，排空链，关闭单元，释放名称。

`dsh-storage-json` 在根目录下每个单元存储一个格式化文件。`writeAtomic`：临时文件（`wx`，`0600`）→ 写入 → fsync → `rename` 覆盖目标。`dsh-storage-sqlite` 存储一个数据库文件，带有共享表：`units(name PK, version)`、`unit_globals` 和每个单元的记录表 `u_<unit>_<table>(key TEXT PK, value TEXT NOT NULL) STRICT`。每个原语是单个 SQL 语句；调用方拥有写入顺序。`loadAll` 构建空原型记录对象，因此 `__proto__` 键作为自有属性存在。