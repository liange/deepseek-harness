# 组合与运行时 — 实现原理

[English](composition-runtime.md) | 中文

阐述启动流程、配置文件与包、Agent 预设、动态扩展、设置、凭据、钩子和 MCP 客户端的内部实现。类型定义参见 [subsystems/settings.md](../subsystems/settings.md)、[subsystems/credentials.md](../subsystems/credentials.md) 和 [subsystems/extensions.md](../subsystems/extensions.md)。

## 启动流程 (`dsh-app-boot`, `dsh-cmdline`)

`boot()` 创建一个 `Context`，将 `baseUrl` 设置为配置目录，提供 `dshHomePath`，等待 `ctx.plugin(Loader)`，挂载一个固定 ID 的 `'include'` 根条目，携带所有补丁层作为一个扁平列表通过 `applyEntryPatches`，然后结算并审计每个条目纤维。`loadLayeredEnv` 使用 `parseEnv` 解析项目和 home `.env` 层，拒绝仅引导用的名称，在不替换继承值的情况下应用，并快照来源。`renderConfigDump` 对前缀快照重新运行补丁算法，以计算每行的来源 YAML 注释。`dsh-cmdline` 提供启动器→应用命令行契约：`provideCmdline` 冻结 argv 快照，并提供 `ctx.cmdlineArgs` 和 `ctx.appExit`。`parseCmdline` 通过全局存储读取它们，将 `exitOverride()` 和 `configureOutput` 应用于每个子命令，并将帮助、版本、解析和 `program.error` 拒绝通过 `ctx.appExit` 路由（永不使用 `process.exit`）。

## 配置文件与包组合 (`dsh-base`, `dsh-headless`, `dsh-web-app`)

配置文件是存储在 harness 家目录中的命名组合。包是 Cordis 配置行的分发格式。`dsh-base` 是每个配置文件的第一层：一个 `cordis.patch.yml` 插入 78 行按 ID 寻址的配置（定时器、hmr、llm、session、typert、session-title、agent、jobs、llm-retry、settings、credentials、sandbox + policy、bash/pwsh 对、审批预设、tools、skills、commands、goal、plan-mode、compaction、subagent、workflow、todos、web-search、fs-sandbox、llm-deepseek）。补丁会替换整行配置。`dsh-headless` 覆盖基础行以支持一次性任务模式：运行器等待 Loader 结算，使用新的 `SessionId` 创建 `Agent`，发送用户消息，等待 `whenIdle()`，打印最终助手文本，并通过 `appExit` 退出。`dsh-web-app` 覆盖基础行以支持浏览器界面：插入存储三元组、webserver、api-gateway、cordis-host-runner、api-remotes、客户端模块系统和 ui-* 花名册。粘合插件在绑定到所有接口时采样一次 LAN IPv4，挂载 `FrontendStatic`，注册 `app:web-surface` 提示词节，并仅在 Loader 结算后打印 URL 行。

## Agent 预设 (`dsh-agent-presets`)

`AgentPresets` 从配置目录和 home `~/.agent-presets` 解析根。`ensureStanding` 单次飞行一个 `Map<presetId, Promise>`：它在挂载前统计组合（mtimeMs+size 戳），从无追踪上下文创建 `dsh-scope` 作用域，并 `mountPreset` 一个 `PresetTree` Include 到其中。`mount.ts` 覆盖 `import` 以从记录的 harness `baseUrl` 解析裸露说明符，并覆盖 `write()` 为空操作。它审计 `inactiveRows`（待处理的注入）和 `leakedServices`（根领域发布），并修剪死挂载。Agent 通过 `bindScopeParent` 加入；`composeFrom` 同步将子节点加入其父节点的常设挂载。`dsh-persona` 是一个仅作用域的行，用于在单个 Agent 预设中替换部署角色。

## 动态扩展 (`dsh-cordis-host-runner`, `dsh-cordis-client-runner`)

`DynamicCordisRunner` 是主机端插件系统：不可变的 Packages（主机+客户端源码）、按会话的 Plugins、一个活跃的 Run 和人工批准的 Client 激活。`run()` 解析模态规则（`run` vs `update` 针对 `currentPackageId`/`nextPackageId`），单次飞行每个插件的激活。`startFresh` 撤回先前的运行，在 `node:vm` 沙箱中评估主机端（标记的 console、`harness.handle/defineTool/registerTool`、编码原语、将 `require`/timers/`fetch` 重定向到服务的陷阱），并将插件作为子纤维挂载在 `cordis-dynamic` 组下。`dsh-cordis-client-runner` 是浏览器端：每个包的源码作为异步函数体运行，其参数是符号表面。结果被包装并作为工厂安置在模块表中，创建真正的 Loader 条目。`dsh-tool-cordis` 提供 `cordis_inspect_*`/`cordis_define`/`cordis_run`/`cordis_stop`/`cordis_undefine` 工具，以及 `@pluginId` 上下文注入。`dsh-client-ui-cordis` 提供浏览器清单面板、审批流程和运行卡片工具。

## 设置 (`dsh-settings`, `dsh-settings-file`)

`SettingsProvider` 是服务定义：`register(ns, schema, {base, applies, validate})` 返回 `SettingsScope`。解析 = `schema(mergeLayers(base, section))`（纯对象递归合并；数组/标量整体替换），深度冻结。写入：`cloneJsonShaped` 拒绝非 JSON 安全的值；每个命名空间有一个序列化的写入链；`expectedRevision` 检查在队列前端运行 → `SettingsConflictError`；持久化 → 提交。提交仅在 `deepEqualJson` 检测到更改时触发：每个观察者序列化的调用链，然后 `settings/updated` 扇出。`dsh-settings-file` 是基于文件的后端：`$DSH_HOME/settings.yaml` 下的一个 YAML/JSON 文档。`persistSection`：mkdir `0700` → `withFileLock` → `reconcileFromDisk`（合并未观察到的外部编辑）→ 渲染 → `writeFileAtomic` `0600`。YAML 渲染重新解析缓存的文本，并应用 `patchNode` 叶级差异，以便注释、锚点和格式保留。chokidar 监视器热重载；无效重载保留最后良好版本。

## 凭据 (`dsh-credentials`, `dsh-credentials-local`)

`CredentialProvider` 是抽象缝：`resolve` 按调用（消费者不得跨操作缓存），`describe`（来源/可写性，不含值），`set`/`unset`（空存储值在各处均为不存在）。`dsh-credentials-local` 是基于文件的后端，位于 `$DSH_HOME/.credentials.yaml`。解析顺序：继承的进程环境（优先，只读）> 管理文件 > 项目 `.env` > home `.env`。`set`/`unset` 首先运行 `assertUnshadowed`（当继承的环境提供该引用时拒绝），然后序列化操作：`withFileLock` → `reconcileFromDisk` → 保留注释的单键编辑 → `writeFileAtomic` `0600` → `notifyUpdated`。`assertOwnerOnly` 拒绝组/其他权限位（POSIX）。chokidar 监视器热重载；无效重载保留最后快照。

## 钩子桥接 (`dsh-hook-protocol`, `dsh-hooks-claude-code`, `dsh-hooks-codex`)

`dsh-hook-protocol` 是共享的非插件库：`runHook` 通过 `ctx.shell.run()` 执行，带有 JSON stdin、每个钩子的 `timeoutSec` 和所属操作信号。`parseHookOutput`：exit 2 = 阻止并返回 stderr 原因；exit 0 时，以 `{` 为前缀的 stdout 被解析为 `continue`/`stopReason`/`systemMessage`/`decision` 加上 `hookSpecificOutput`。`mergeHookOutputs` 折叠 `deny > ask > allow`。`dsh-hooks-claude-code` 在加载时一次性解析 `hooks.json`，并将点映射到瀑布流扩展点：`agent/session-start`、`agent/pre-step`（deny → reject）、`tools/pre-execute`（deny/ask）、`tools/post-execute`（deny → block）、`agent/turn-stopping`（deny → steer）、`subagent/start|end`。`dsh-hooks-codex` 镜像 CC 桥接，但遵循 Codex 的契约差异：五个点，仅正则匹配器，`permission_mode: 'default'`，以及 SessionStart 和 UserPromptSubmit 上的 `plainStdoutAsContext: true`。

## MCP 客户端 (`dsh-mcp-client`)

每个实例在根键控的 WeakMap 集合中保留其 `serverName`。监督器拥有客户端和传输代际：每个代际是一个全新的 MCP SDK `Client` 加上传输（stdio 生成使用 `scrubbedParentEnv()`；可流式 HTTP 接受 URL+headers）。初始连接 + 工具同步完成 `ready` 以激活插件。`ToolListChangedNotification` 触发在一个 `syncChain` 上序列化的重新同步。重新连接使用每次中断的加倍退避；正常运行时间 ≥ `maxDelayMs` 重置预算；耗尽则取消注册工具并停止。`dispose` 关闭活跃客户端，等待 `settling` + `syncChain`，并取消注册所有内容。注册的工具前缀为 `mcp__<serverName>__<rawName>`。