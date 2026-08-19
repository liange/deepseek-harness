# 实现原理

[English](README.md) | 中文

按领域划分的参考页面，阐述 DeepSeek Harness 各模块的内部实现：数据结构、状态机、算法和跨包连接。这些页面是对 [architecture.md](../architecture.md)（行为的有序地图）和 [subsystems/](../subsystems/README.md)（类型定义和语义）的补充。各包契约和配置请查阅所属包的 README。

| 页面 | 覆盖范围 |
|---|---|
| [core-agent-loop.md](core-agent-loop.md) | Agent 驱动引擎、会话事件日志、工具注册表、工具执行流水线、系统提示词装配、作用域上下文原语、计划模式、待办事项、压缩和人类交互（命令、审批、用户提问、权限预设） |
| [capability-execution.md](capability-execution.md) | Shell、子进程、终端、文件系统、LSP、代码运行时、沙箱、E2B 远程沙箱和工作区注册表 |
| [llm-and-models.md](llm-and-models.md) | LLM 适配器注册表、流式传输、块组装、DeepSeek 与 pi-ai 适配器、重试策略、Token 计量器 |
| [tools-and-services.md](tools-and-services.md) | Skill 能力提供者注册表、子代理委托、工作流引擎、后台任务、目标追踪、定时提醒、Web 搜索/抓取、附件存储、消息反馈和溢出策略 |
| [session-storage.md](session-storage.md) | 会话持久化（JSONL 和 SQLite 后端）、投影引擎、投影缓存、遥测、会话标题、基于 FTS5 的会话查询和领域 KV 存储 |
| [composition-runtime.md](composition-runtime.md) | 启动流程、配置文件和包组合、代理预设、动态插件扩展、分层设置、凭据引用、钩子桥接和 MCP 客户端 |
| [api-client-platform.md](api-client-platform.md) | Typert 类型图系统、JSON-RPC SDK、ACP 服务器、API 网关、Web 服务器、客户端模块系统、UI 槽位组合和基础设施工具 |