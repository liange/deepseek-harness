# API、客户端与平台 — 实现原理

[English](api-client-platform.md) | 中文

阐述 Typert 类型图、JSON-RPC SDK、ACP 服务器、API 网关、Web 服务器、客户端模块系统、UI 槽位组合和基础设施工具的内部实现。类型定义参见 [subsystems/typert.md](../subsystems/typert.md)、[subsystems/web-server.md](../subsystems/web-server.md)、[subsystems/client-modules.md](../subsystems/client-modules.md) 和 [subsystems/invariants.md](../subsystems/invariants.md)。

## Typert 类型图 (`dsh-typert-generator`, `dsh-typert-loader`, `dsh-typert-registry`, `dsh-typert-protocol`)

Typert 系统有四层。`dsh-typert-protocol` 定义编译器无关协议：`@Remote`/`@RemoteScope` 类方法装饰器通过 `addInitializer` 在 `WeakMap` 中记录标记。`InvocationDescriptor` 携带服务、命名空间、方法、编解码器（严格 Zod 或 SRC-JSON）和可选取消信号。`dsh-typert-generator` 是 TypeScript 编译器：`WorkspaceAnalyzer` 解析每个面的聚合 tsconfig，构建 `ts.Program`，并将 Cordis 服务、事件和标记的 Typert 根提取到 `FaceModel` 中。`FaceModelEmitter.emit(package)` 生成 `typert.host` JS/d.ts 工件，带有 Zod v4 模式。`typertPlugin`（tsdown 插件）通过 `ts.transpileModule` 降低装饰器，并在 `writeBundle` 中写入工件。`dsh-typert-loader` 自动注册包的 `./typert` 主机面工件：它增量扫描 Loader 条目，通过微任务刷新协调脏名称，并在 `ctx.typert.register()` 之前验证每个清单（包所有权、面、zod 实例、调用描述符）。`dsh-typert-registry` 是运行时注册表：`DescriptorStore` 按端点和 ID 保存条目；`LocalStore` 托管活跃组合的定义；`RemoteStore` 在 Cordis 效果中拥有注册；`LookupStore` 将提供者注册与解析器配置分离；`ContextStore` 具有 `registerHost`/`configureHost` 和 `registerClient`。

## JSON-RPC SDK (`dsh-sdk-protocol`, `dsh-sdk-jsonrpc-server`, `dsh-sdk-client`)

`dsh-sdk-protocol` 定义线协议：通过 stdio 的换行分隔 JSON-RPC 2.0。`JsonRpcLineTransport` 通过 `StringDecoder` 累积 UTF-8，按换行分割，并通过 `pending` Map 关联请求。线词汇：`InitializeParams/Result`、`SessionPromptParams/Result` 和四个服务器→客户端通知（`session.event`、`session.status`、`subagent.started`、`subagent.finished`）。`dsh-sdk-jsonrpc-server` 实现 `HarnessSdkJsonRpcServer`：构造函数订阅 Cordis 事件流并将其作为通知转发。`initialize` 验证 `maxTokens`，记录 cwd/provider/model，并挂载 DeepSeek 提供者纤维。`prompt` 惰性创建 Agent+Session 对并 `followup` 用户消息。`shutdown` 是记忆化的异步拆卸。`dsh-sdk-client` 是低级客户端：`HarnessClient` 直接生成运行时，通过 stdio 使用 ndjson JSON-RPC 通信，并实现 `initialize`/`prompt`/`shutdown`。通知扇出到带有队列和等待器的 `NotificationSubscription`。拆卸是私有阶梯：stdin EOF → SIGTERM → SIGKILL，每个阶段与无引用的定时器竞争退出。

## ACP 服务器 (`dsh-acp`)

`dsh-acp` 是通过 JSON-RPC stdio 的仅自动化 Agent Client Protocol 服务器。它使用 `@agentclientprotocol/sdk` 的 `AgentSideConnection`。按会话的 `SessionRecord` 持有拥有的 Agent、其确切的回收器、用于有序助手投递的 `outputTail` promise 和一个飞行中结算记录。处理程序：`initialize`、`session/new`、`prompt`、`cancel`。富内容准入根据四种类型的光栅 MIME 白名单和规范 RFC 4648 base64 验证内联图像；失败为 `AcpContentError`，带有 `invalid`|`internal` 类别。`turnEndToStopReason` 映射 completed→`end_turn`，max-tokens→`max_tokens`，interrupted→`cancelled`，其他→`end_turn`。

## API 网关 (`dsh-api-gateway`, `dsh-api-remotes`)

`TypertGatewayService` 是主机端调度器，将 Typert Remote 端点转换为通过 Connection RPC 载体的活跃 Cordis 服务方法调用。`claimsEndpoint` 接受存在于 `ctx.typert.local` 或通过 SRC 标记声明的端点。`invoke` 解析 `InvocationDescriptor`，断言确切的线参数，解码参数，解析查找参数和 Context 接收器，通过 `Reflect.apply` 调用，并验证结果。`dsh-api-remotes` 是主机 BFF 策略层：`API_REMOTE_FORWARDED_EVENTS` 列出转发的事件（编译时 `satisfies` 门禁）。`createApiRemoteAgentResolver` 复用活跃 Agent，每个身份恢复一次冷会话，并配置 Typert 查找和上下文。客户端面挂载选定的 Host Remote 贡献。

## Web 服务器 (`dsh-host-webserver`, `dsh-host-frontend-static`, `dsh-host-apiproxy`)

`WebServer` 是路由注册插件：`node:http` 服务器，带有精确、前缀和升级映射、索引点击器和一个回退席位。`register` 在重复时抛出；`registerFallback` 仅允许一个所有者。匹配是精确优先，然后最长前缀优先。`dsh-host-frontend-static` 占用回退席位：非 GET/HEAD 回答 405；否则 `serveStatic` 解析目标，拒绝超出 distRoot 的遍历返回 403，并在任何未命中时提供 index.html 用于 SPA 路由。每个索引渲染是 `await ctx.webServer.applyIndexTaps(readFile(distIndex))`——启动清单注入点。`dsh-host-apiproxy` 提供传输无关的 API 网关：`createApiProxy` 返回每个领域的纯闭包。`toFetchHandler` 编译带有 zod 验证的调用闭包的 `UNARY_ROUTES`。业务错误携带 200 + ServerResponse；HTTP 状态仅表示载体故障。

## 客户端模块系统 (`dsh-client-connection`, `dsh-client-modules`, `dsh-client-runtime`, `dsh-client-web`, `dsh-client-web-react`)

`dsh-client-connection` 是浏览器载体：主机 HTTP/WebSocket 桥接，带有 DNS 重绑定信任围栏。`PRIVILEGED_METHODS` 在可信主机部署上固定到环回。客户端 `ConnectionController` 以拉取模式打开两个 SSE 流，等待 `onOpen` 和 `describe`，使用抖动指数退避重新连接。`dsh-client-modules` 是双面模块系统：主机扫描 Loader 条目中的 `dsh.client` 包，并提供浏览器插件图。`parseDshClient` 验证 platform、inject 和 immediately 字段。组合图是同时向注入的 `window.__DSH_BOOT__` 启动清单和提供的图提供数据的单一线形状。客户端：外壳内核在 Cordis 存在之前构建带有静态模块的 `ClientModuleSystem`。

`dsh-client-runtime` 是持有所有业务状态的无 React 对象层。`apply` 挂载 `SlotRegistry`，构建 `SessionRuntime` 和 `WorkspaceRuntime`，注册 Agent Typert Context 客户端绑定器，然后启动连接。快照存储引擎（`createSnapshotStore`、zustand/immer）生成无钩子的裸可观察源。`ConversationNodeDefinition` 注册表 + 键控的 `conversation.chat.node` 渲染器按日志 seq 驱动确定性重放。`dsh-client-web` 是外壳库：`AppWebEntry.run()` 两阶段——模块面（解析 `window.__DSH_BOOT__`，构建 `ClientModuleSystem`）然后是插件面（渲染加载页面，预取 immediately 层，挂载 Loader，为每个插件视图行创建一个条目，`loader.await()` + 完整纤维扫描）。`dsh-client-web-react` 提供渲染机制：`bindSnapshotSelector` 是唯一的钩子构造函数（`useSyncExternalStoreWithSelector`）。`createSlotRenderer` 接收运行时 SlotRegistry 主机面，并安装缓存在 `WeakMap` 中的每个条目 renderSlot 绑定。

## UI 槽位组合

每个 ui-* 包是一个功能，带有 `src/client/` 浏览器端和存根节点端。组合只有一个 API：`ctx.slots.register({ name, children?, store?, inject?, locale? }, Component)`。`children` 对象同时是组件渲染哪些槽位的声明和授权——渲染未声明的槽位在加载时失败。组件属性是四个派生的份额：`PropsRuntime<K>`（标准钩子）、`PropsRenderSlots<S>`（子键）、`PropsStore<H>`（存储工厂），加上注入面。五个常设钩子席位：`useSession`、`useSessions`、`useWorkspaces`、`useStore`、`renderSlot`。`dsh-client-ui-slots` 提供框架核心：`SlotCore` 注册表，具有排他所有权强制执行。`dsh-client-ui-primitives` 提供通过 `--dsw-*` 令牌样式的无 Cordis React 原语。`dsh-client-ui-conversation` 是聊天视图领域：对话节点渲染、编写器机器和视图选项卡。`dsh-client-ui-layout` 提供带有四个子槽位的 AppFrame。`dsh-client-locale` 提供双语区域设置，带有类型化的 `LocaleNamespaceMap` 和修订键控绑定。

## 基础设施工具

`dsh-atomic-write` 提供 `writeFileAtomic`（随机后缀同级文件，独占 `wx` 创建，重命名）和 `withFileLock`（跨进程锁，指数退避）。`dsh-timeout` 提供 `clampTimeout`、`Deadline`（一次性 `using` 回收）、针对异步迭代器需求的 `IdleWatchdog` 和 `MAX_TIMER_DELAY_MS`。`dsh-output-retention` 提供 `ItemRetainer`（有序单元，仅头部）和 `TextRetainer`（面向字节的头部/尾部/头尾，UTF-8 边界安全）。`dsh-launch-environment` 提供不可变的启动时环境快照，分层为 `process > project-env > user-env`，带有来源。`dsh-home-paths` 提供 `DSH_HOME` 解析和 `expandHomePath`。`dsh-brand` 是仅类型的 `Branded<B>` 名义类型原语。`dsh-native-command` 是一个无 shell 的 `execFile` 库，带有 AbortSignal 终止。`dsh-anonymous-user-id` 推导出作用域限定在 harness 家目录的随机 UUID。

## 运行时不变式 (`dsh-invariants`)

`InvariantRegistry` 是包拥有的运行时不变式贡献的可配置注册表。模式通过允许列表然后阻止列表选择编译。包调用 `ctx.invariants.register(packageName, { install, inject? })`；安装器在注册拥有的子纤维中运行。`verify-package-invariants` 门禁审计每个配套文件：每个必须检查事件/数据关系或声明一个合理的"无运行时不变式"原因。

## 测试支持

`dsh-acp-snapshot` 提供快照测试工具：子进程启动器、脚本化场景工具、纯规范化器和套件工厂。`dsh-agent-loop-testkit` 按顺序挂载五个前提服务（LlmRuntime、SessionStore、SystemPrompt、ToolRuntime、AgentRegistry）。`dsh-client-runtime`（测试支持）提供 jsdom 槽位测试运行时，带有真实 Cordis Context + 生产 SlotRegistry + web-react 渲染器，围绕测试双倍。`dsh-llm-mock-server` 是一个可脚本化的 OpenAI 兼容 HTTP/SSE 服务器，具有 24 种行为名称。`dsh-llm-replay` 是一个无密钥快照测试 LLM 适配器：`deriveReplayScript` 从 `assistant/chunk` 事件构建模型调用脚本。`dsh-loader-smoke` 提供无密钥示例冒烟测试的子进程工具，带有 tsx 源码启动和隔离的 `DSH_HOME`。