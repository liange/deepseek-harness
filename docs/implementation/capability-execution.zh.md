# 能力执行 — 实现原理

[English](capability-execution.md) | 中文

阐述 Shell、子进程、终端、文件系统、LSP、代码运行时、沙箱、E2B 和工作区的内部实现。类型定义参见 [subsystems/shell.md](../subsystems/shell.md)、[subsystems/subprocess.md](../subsystems/subprocess.md)、[subsystems/terminal.md](../subsystems/terminal.md)、[subsystems/filesystem.md](../subsystems/filesystem.md)、[subsystems/lsp.md](../subsystems/lsp.md)、[subsystems/code-runtime.md](../subsystems/code-runtime.md)、[subsystems/sandbox.md](../subsystems/sandbox.md) 和 [subsystems/workspace.md](../subsystems/workspace.md)。

## Shell 执行 (`dsh-shell`)

`ShellExecutor` 抽象服务定义了 `ctx.shell` 能力缝。`resolve(request)` 填充默认值并生成 `ShellExecSpec`；`run` 仅在基础设施故障时拒绝（非零退出、超时和中止都以结果形式解决）；`start` 返回一个 `ShellProcess`，其 `readOutput()` 是增量且消耗性的，`done` 永不拒绝，`kill()` 是幂等的。`timedOut`/`aborted` 是互斥的（首因分类）。`dsh-shell-env` 拥有 `ctx.shellEnv` 注册表，管理注入到每个模型 Shell 调用中的可信 `DSH_*` 变量。环境执行器在快照最后合并之前丢弃主机 `DSH_*`，因此陈旧的主机值永远不会泄漏。

`dsh-bash-local` 通过 `ctx.subprocess` 以 `bash -c` 运行命令。`resolve()` 限制 `timeoutMs`（默认 120s，上限 600s），默认 `workdir`，并携带 stdout 字节预算。`spawnSpec()` 按 `ENV_OVERRIDES → spec.env → spec.dshEnv` 分层设置环境。`dsh-bash-sandbox` 扩展了它：通过 `ctx.sandbox.confine()` 包装 argv，在运行器故障分类规则之前分类运行器生成失败（`SandboxUnavailableError`），在拒绝签名之前分类，并维护按进程的 `processFacts` 映射。`dsh-pwsh-local` 镜像 bash-local 的 PowerShell 版本：每个命令是 `-Command` 之后的一个 argv 元素，`ENCODING_PREAMBLE` 设置 UTF-8 输出，`resolvePwshPath()` 探测 PowerShell 7 然后 Windows PowerShell 5.1。`dsh-pwsh-sandbox` 与 bash-sandbox 完全镜像。

`dsh-tool-bash` 是面向模型的消费者：前台调用同步运行；`run_in_background` 注册一个 `ctx.jobs` 任务驱动 `ShellProcess` 句柄。沙箱提权在执行前通过 `ctx.approval` 审批。`dsh-tool-bash-persistent` 使用 PTY 缝：每个 Agent 一个长期存活的 Shell，命令包装为 `printf start; eval -- <command>; printf end:$status` 放在一行，使用基于滚动缓冲的结束标记检测循环。`dsh-tool-pwsh` 镜像 tool-bash 的 PowerShell 方言契约。

## 子进程执行 (`dsh-subprocess`, `dsh-subprocess-local`)

`SubprocessRuntime` 是抽象的服务定义：`spawn` 不应用任何默认值，每个配置都是显式的。`scrubbedParentEnv()` 是共享的清洗函数：父环境减去匹配 `SENSITIVE_ENV_PATTERN` 的名称，再减去 `DSH_*`。`dsh-subprocess-local` 是主机本地后端。`spawnSubprocess()` 使用 POSIX `detached: true`（独立进程组）。`OutputCollector` 保留内存尾部，上限为 `maxBytes`；在首次溢出时，它打开一个私有 `0700` 目录的溢出文件，以 `'wx'` 打开并重放之前的块。`treeAlive()` 轮询 `kill(-pid, 0)` 加上 Linux `/proc` 成员探测。`terminate()` 启动单一的 `observeTreeExit` 观察者，发送 SIGTERM，设置一个有引用的 `graceMs` SIGKILL 定时器，并等待整个进程树退出。如果后代持有管道，`done` 在相同的宽限期内完成。

## 终端 (`dsh-terminal`, `dsh-terminal-bash`)

`TerminalSessionService` 拥有会话 ID、发布、授权和清理。`spawn(owner, {type, name?, cwd?})` 生成 ID `pty-N`，保留所有者本地的名称，并使用 `AbortController` 跟踪 `PendingSpawn`。发布是全有或全无的：失败时未发布的会话被关闭（回滚）。所有操作都需要确切的所有者（`expectOwned`）。`dsh-terminal-bash` 是 PTY 后端：生成带有 `PS1='dsh> '` 和 `PROMPT_COMMAND` 的受控 bash 提示词，该命令在每个提示词前发出私有 OSC 标记。输出通过 `TerminalSanitizer`（去除 CSI/OSC，跟踪拥有的标记）流入行+字节限制的滚动缓冲。就绪判定：在 `terminal.write` 之后，`pollReadiness` 循环在空闲窗口后看到拥有的提示词加 shell 前台 pgid 时以 `stdin_read` 完成，或在 `idleSilenceMs` 后以 `inferred_idle` 完成。`dsh-tool-terminal` 暴露六个面向模型的工具（`terminal_open/send/read/signal/close/list`），后台发送注册为 `ctx.jobs` 任务。

## 文件系统 (`dsh-fs`, `dsh-fs-local`, `dsh-fs-sandbox`)

`FileSystem` 是抽象的服务定义：`resolve` 返回带有别名不变 `targetKey` 的稳定 `FsTarget`；`versionOf` = `dev:ino:size:mtimeNs:ctimeNs`。`dsh-fs-local` 通过 realpath 解析目标：对于缺失的目标，它向上遍历 realpath 到最近存在的祖先，然后重新附加缺失的后缀。变更按 `targetKey` 在 FIFO Promise 链中序列化（`withLock`）。`writeText` 使用 `writeFileAtomic`：同级 `.name.pid.uuid.tmpdir` 目录 chmod `0700`，临时文件以 `'wx'`/`0600` 打开，写入内容并 fsync，然后 `rename` 覆盖目标。`editText` 将 CRLF 规范化为 LF，检测主导行尾，应用 `applyLiteralEdit`（计算出现次数：0 → `FS_EDIT_NOT_FOUND`，>1 且无 replaceAll → `FS_AMBIGUOUS_EDIT`），并在回写时恢复行尾。`dsh-fs-sandbox` 扩展 `LocalFileSystem`，在变更操作上添加按调用的沙箱防护：`read-only` 抛出 `FS_SANDBOX_DENIED`；`workspace-write` 重新解析目标，并要求在 `writableRoots`（工作区根 + `/tmp` + 平台临时目录）下进行包含检查。`dsh-fs-observation-policy` 记录每个会话的每个权威观察，并推导变更守卫：`present` → `replaceIfVersion`，`absent` → `createIfAbsent`。

`dsh-tool-fs` 提供 `read`/`write`/`edit`/`read_image` 工具。`read` 进行一次 stat 用于类型路由和大小判断，然后流式传输或窗口化。`write`/`edit` 解析沙箱策略，占用单槽 `fs/write-intent` 或 `fs/edit-intent` 瀑布流，然后调用提供者。`dsh-tool-fs-search` 通过 `ctx.subprocess.spawn` 运行打包的 ripgrep 二进制文件——从不使用 `ctx.shell`。`dsh-tool-str-replace-editor` 通过 `ctx.fs` 提供 `view`/`create`/`str_replace`/`insert` 命令。

## LSP (`dsh-lsp`, `dsh-lsp-stdio`)

`Lsp` 是服务定义，包含四个只读操作（`goToDefinition`、`findReferences`、`goToImplementation`、`hover`）。`registerProvider` 在变更前验证所有内容（全有或全无）：非空 ID、扩展名映射、语言 ID、提供者内部和跨提供者冲突检查。`dsh-lsp-stdio` 在发布任何提供者之前通过 `ctx.subprocess.resolveExecutable` 解析每个可执行文件。`LocalLspProvider.query` 通过 `ctx.fs` 规范化工作区，然后按工作区键排队完整的读取→（获取或创建实例）→查询生命周期。`LspInstance.initialize` 协商位置编码（仅 utf-16）。每个查询是 `didOpen → request → didClose`。传输失败会销毁实例并在替换实例上重试一次。`LspConnection` 通过 Content-Length 编解码器进行帧封装（头部 ≤64KiB，正文 ≤`maxMessageBytes`）。`dsh-tool-lsp` 将以 1 为基的 UTF-16 光标坐标转换为缝的以 0 为基的位置。

## 代码运行时 (`dsh-code-runtime`, `dsh-code-runtime-worker-thread`)

`CodeRuntime` 定义了抽象缝：一个模型编写的程序针对主机异步绑定运行。`dsh-code-runtime-worker-thread` 是 Worker 线程后端。`run()` 通过 `node:module.stripTypeScriptTypes` 在 `async function __dsh_program__() {}` 包装器中对程序进行类型剥离。生成一个带有 `env:{}`、`execArgv:[]` 和堆 `resourceLimits` 的新 Worker。Worker 内部：`runWorkerMain` 构建空原型命名空间（通过 MessagePort 的异步桥接），五个方法的 console 填充，并将 `process.stdout/stderr.write` 补丁到有序的 `LogBuffer` 中。预算：eventLoopUtilization 忙碌时间轮询（25ms 节奏）→ `timeout`，加上墙钟后备。主机端 `parseWorkerMessage` 逐字段重新验证每个入站帧（敌对节点）。`OutputLedger` 强制执行组合的 JSON 字节上限。恰好一个结果胜出；每个路径都会排空管道、终止并等待 Worker。

## 沙箱 (`dsh-sandbox`, `dsh-sandbox-local`, `dsh-sandbox-policy`)

`SandboxProvider` 是抽象缝：`SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'`。策略按调用携带（两个消费者可以同时以不同的策略进行限制）。`dsh-sandbox-policy` 拥有 `ctx.sandboxPolicy`：`resolve(request)` 优先级为显式批准的模式 > 会话最后的 `sandbox/mode` 事件 > 部署默认值。按会话的覆盖就是会话日志：`setSandboxMode` 追加一个 `sandbox/mode` 事件；`effectiveSandboxMode` 是纯向后折叠。`dsh-sandbox-local` 选择平台运行器链：Linux bwrap→Landlock，macOS Seatbelt，Windows ACL 受限令牌运行器。多级链通过缓存一次的功能性探测来仲裁。`confine()` 发出 `[runner, ...profileArgs(policy), '--', ...argv]`。在 Windows ACL 级别，`AclSandbox` 物化一个持久的工作区写入 SID 授权（从 sha256 确定性派生）和一个按会话的私有临时 SID，创建带有能力 SID 的受限令牌，并在令牌 DEFAULT DACL 中合并一个完全访问 ACE，以便孙进程管道能够存活。

## E2B 远程沙箱 (`dsh-e2b`, `dsh-fs-e2b`, `dsh-subprocess-e2b`)

`dsh-e2b` 拥有共享的 E2B SDK 沙箱句柄。`open()` 创建沙箱，准备 `cwd` 和私有 `runtimeRoot`，并验证它是一个真实目录。`dsh-fs-e2b` 在远程沙箱上实现 `ctx.fs`：`resolve` 通过远程 `realpath -mz | base64` 命令计算 `targetKey`；变更按键序列化，并通过私有暂存目录使用 SDK `rename` 原子发布。`dsh-subprocess-e2b` 实现 `ctx.subprocess`：每个句柄通过 `sandbox.commands.run` 以 `background: true` 运行一个生成的控制脚本。命令文本通过 `command -v` 解析引导工具，将 pgid 和退出代码写入状态文件，并通过 `tee > head -c spillCap > file | node -e <base64 encoder>` 分流每个输出流。输出通过基于换行符分隔的 base64 帧传输，带有终端 `!dsh-e2b-output-complete!` 帧。

## 工作区 (`dsh-workspace`)

`WorkspaceRegistry` 在 `dsh-storage-domain` 领域（版本 2）中存储工作区实体。`[Service.init]` 打开领域，恢复待处理的变更标记，验证存储的状态（唯路径、每会话一个所有者、顺序↔表双向映射），然后运行一次性引导：按规范化的 `header.cwd` 分组持久化的会话头，最新优先，将未归属的成员分配到现有工作区。所有变更在一条操作链上运行。`createCanonical` 是待处理标记的双写操作，失败时回滚。`WorkspaceEntity` 持有一个在每次持久 `mutate` 后交换的私有记录快照；其 `sessionIds` 获取器会过滤出规范 cwd 与工作区路径不同的成员。