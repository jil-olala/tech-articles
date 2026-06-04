# Part III - 架构对比与设计启示

- **来源：** 腾讯技术工程（微信公众号）
- **作者：** rianli
- **日期：** 2026-05-29
- **原文链接：** https://mp.weixin.qq.com/s/49dxdMXEUoWIYlIh8fFqMQ
- **系列：** OpenClaw与Hermes源码复盘（Part III）

---

Part I 和 Part II 分别拆开了两个框架的源码细节。Part III 把它们放在同一个标尺下做对比——为什么走了不同的路、每个维度差在哪、架构师能从中学到什么。
### 21. 架构对比：OpenClaw vs Hermes
### 21.1 为什么走了两条不同的路
把 10 张对比表全部摆出来之前，先讲清一个根本问题——OpenClaw 和 Hermes 不是同一个目标的两种实现，而是两个不同目标的成熟解。理解这一点，后面所有维度的差异才能讲通。
哲学一：OpenClaw 选微内核——因为它要做"长期演进的平台"
OpenClaw 的所有架构选择都指向同一个目标：让多个团队、多种语言、多个迭代周期的代码能共存而不互相破坏。
- Plugin SDK 强契约 —— 让第三方插件可以独立演进（百余 extensions 是结果不是原因）
- Channel 25+ Adapter —— 让"加新通道"不需要懂 Gateway 内核
- Auth Profile 双轴 —— 让"换 Provider"不需要改路由逻辑
- Context Engine 可插拔 —— 让"换记忆策略"不需要改 Runtime
- CLI Backend 双路径 —— 让"换 LLM 调用方式"不需要改业务逻辑
代价是上手陡——新人要先理解 SDK 才能写第一行代码。回报是可演进性——核心 src/ 几千行，能力靠插件叠出来；upstream/main 一年能合上千个 PR 而核心架构基本不动。这是平台级框架的宿命。
哲学二：Hermes 选单体——因为它要做"个人开发者闭环"
Hermes 的所有架构选择都指向另一个目标：让一个开发者从安装到改源码到自己造工具的链路最短。
- AIAgent 单体类 —— 一个文件看完就能理解全部核心逻辑
- ToolRegistry 自注册 —— 加新工具就是加一个文件 + 一行 @register
- 技能自创建闭环 —— Agent 自己根据经验创建新技能（连"加工具"都不用你做）
- MEMORY.md, USER.md 直接编辑 —— 不需要懂数据库或向量索引
- Smart Approval LLM 辅助安全评估 —— 不需要写规则也能跑得稳
代价是难演进——改核心行为（压缩策略、记忆机制、凭证池）要动 1 万行的 AIAgent 类，profile 串扰是结构性问题。回报是个人开发者的"全栈掌控感"——一个人在本地就能看完代码、改核心、加工具、造技能，反馈闭环极短。这是工具级框架的宿命。
哲学三：两个框架都把记忆当成长期投入的主战场，而不是可选功能
两个框架的共同点：记忆不是附加在对话历史上的小组件，而是有独立生命周期、独立存储、独立检索链路、并持续迭代的一级模块——各自投入的设计复杂度（Dreaming 三阶段、MemoryManager + 8 插件 + Nudge + Session Search）远超大多数 Agent 框架。
- OpenClaw：启动时 8 个工作区文件注入 System Prompt（SOUL/USER/MEMORY/AGENTS/TOOLS/IDENTITY/HEARTBEAT/BOOTSTRAP）+ MEMORY.md 与 memory/ 子目录 .md 文件构成持久记忆层 + SQLite-vec 向量 + FTS5 全文的混合召回（向量权重 0.7 / 文本权重 0.3，支持 MMR 去重和时间衰减）+ Dreaming 三阶段自动整理（Light 整理候选物料 → REM 提取跨日主题产出强化信号 → Deep 消费所有信号做加权评分后写入持久记忆，是唯一写入路径）
- Hermes：MemoryManager + 8 插件提供者（Honcho 辩证建模, Mem0, Hindsight 等）+ Memory Nudge 周期性反思 + Session Search 跨会话搜索
两个框架的共同判断：工具和通道可以靠生态补齐（加个适配器就能扩），但"Agent 对用户的长期了解"必须框架自己深入做——这是长期使用场景里 Agent 能否持续变好用、而不是越用越无聊的关键变量。
用一张表看哲学差异
维度OpenClawHermes Agent目标受众平台团队 / 多人协作 / 长期演进个人开发者 / 重度 dogfood / 快速迭代设计哲学边界 vs 实现分离（核心做契约，能力靠插件）一体化丰满（核心做完整能力，扩展靠改源码）演进策略微内核 + Plugin SDK 强契约 + 百余插件生态单体 + ToolRegistry 自注册 + 技能自创建协作成本低（插件之间不互相破坏）高（多人改单体类容易冲突）可观测性显式（FailoverError 13 闭合枚举 / Bootstrap 截断告警注入 LLM）隐式（错误启发式分类 / 直接 stdout 日志）适合场景多通道 Bot 平台、多租户 SaaS、需要长期维护的 Agent 服务个人助手、研究原型、单团队工具人核心总结：OpenClaw 是给平台架构师的范本，Hermes 是给个人开发者的瑞士军刀——两者的差异不是"谁更好"，是目标受众不同。后面 10 张维度对比表，本质都是这一句话的具体展开。
### 21.2 Agent 执行引擎对比
维度OpenClawHermes Agent主循环runEmbeddedPiAgent（双路径：embedded vs CLI backend）AIAgent.run_conversation（单一路径）迭代上限Auth profile 数量驱动的 MAX_RUN_LOOP_ITERATIONS硬编码 父 90 / 子 50凭据抽象Auth Profile（api_key + token + oauth，带生命周期）Credential Pool（API Key 数组）凭据持久化磁盘 store，重启保留冷却状态进程内内存外部凭据同步从 claude-cli, codex-cli 自动导入账号无错误分类FailoverError 闭合枚举（13 种 reason）错误类型 + 计数启发式降级触发Profile 耗尽 → 模型 fallback，冷却 profile 支持 probe错误类型直接触发，固定黑名单时长压缩算法三级（pre-request / timeout-triggered / context-overflow）四步（工具裁剪 → 边界 → 摘要 → 组装）Context 管理Context Engine 契约（可插拔，支持检索型）内置 context_compressorCLI 兼容可以把 Claude Code, Codex CLI 当 backend 用仅支持 Copilot ACP 作为 chat backend子 AgentSubagent spawn（registry + control + role + depth 可配）delegate_tool（阻止列表 + 默认深度 1，可配置）并发调度5 车道 CommandLane（防 cron 自递归死锁）全局单一命令队列Prompt 缓存Cache Trace 全链路追踪 + 依赖 Provider 侧主动注入 cache_control 断点 + 冻结快照小结：执行引擎差异最大的地方不是"代码多少行"，而是 "错误契约的闭合性"——OpenClaw 用 13 种 FailoverError 闭合枚举把外部世界的不确定性显式吃掉，Hermes 用启发式分类应对。前者更工程化但维护成本高，后者更敏捷但黑盒风险大。这是平台 vs 工具最深刻的工程差异。
### 21.3 记忆系统对比
层级OpenClawHermes Agent静态层SOUL.md, USER.md, MEMORY.md / AGENTS.md → 每次 buildPrompt 注入MEMORY.md + USER.md → 首轮构建后冻结快照向量层memory-core（SQLite + FTS5 + sqlite-vec；可切 QMD sidecar）或 memory-lancedb插件化记忆提供者（Honcho 辩证建模 / Mem0 / ...）搜索BM25 + Vector 混合，MMR 重排，时间衰减依赖选定记忆提供者后台整理Dreaming 三阶段（Light → REM → Deep）❌ 无等价机制主动召回Active Recall 插件（pre-reply sub-agent）Memory Nudge（每 10 轮触发后台 review）跨会话搜索❌ 无内置Session Search（SQLite FTS5 + Gemini Flash 摘要）记忆安全插件安装时静态扫描每次写入实时检测（12 种威胁模式 + 不可见 Unicode）小结：记忆系统其实分两层——Session 层（原始对话日志，短期记忆）和 Memory 层（从 Session 沉淀出的结构化记忆，长期记忆），两者是"原始日记"和"读书笔记"的互补关系，不是二选一。两个框架各补了一层的缺口：OpenClaw 把 Memory 层做得重（Dreaming 三阶段加权晋升 + 向量/FTS 混合召回），但 Session 层只是 JSONL append-only，无跨会话搜索；Hermes 反过来把 Session 层做得重（SQLite FTS5 + Gemini Flash 摘要做跨会话搜索），但 Memory 层缺自动沉淀机制。理想形态是两层都做好——Session 层保证"找得到原始出处"，Memory 层保证"不用每次都重读原始"。演进方向（程序性记忆、千人千面、遗忘机制）详见"写在最后"第 2 点。
### 21.4 插件/工具系统对比
维度OpenClawHermes Agent扩展机制npm 包 + Plugin SDK 合约 + definePluginEntryPython 模块 + 导入时自注册到全局 Registry添加工具创建独立插件包，实现 register(api)创建工具文件 + registry.register() + 修改 toolsets工具分发npm 发布 + openclaw plugins installpip 安装整个包类型安全TypeScript 编译时类型检查Python 运行时检查MCP 支持通过插件内置 MCP 客户端（stdio + HTTP）技能系统目录式 Markdown 指令（人工编写）目录式 + Agent 自主创建/编辑/patchToolsetsPlugin 粒度的 enable/disable工具集编排（20+ 预定义 toolset，支持递归组合）小结：根本差异是 "扩展路径的耦合度"——OpenClaw 的扩展走独立仓库（独立 npm 包 + 版本化 SDK 契约 + 强类型检查），扩展者不需要动核心代码；Hermes 的扩展走同一仓库（新建 Python 文件 + 导入时自注册），扩展者需要直接在主代码树里加东西。前者为扩展与核心的解耦付出了设计成本（契约/版本/兼容性），后者省下这部分成本换改动直接、反馈快。不存在哪个更好，只有哪个更匹配你的扩展场景——要做"插件化"时，先盘清扩展频率、是否需要多方并行扩展、是否要做版本兼容，再选路径。
### 21.5 安全模型对比
安全层OpenClawHermes Agent网络层TLS 强制 + 证书 Pinning + SSRF 防护IPv4 偏好 + 代理支持认证层Device Identity + Ed25519 签名 + RBACDM 配对验证码命令执行Allowlist + 交互式审批35 条 DANGEROUS_PATTERNS + Smart Approval内容安全插件安装静态扫描Tirith 扫描 + 上下文注入检测 + 记忆扫描 + MCP 扫描技能安全路径遍历 + 文件权限4 级信任 + 6 类威胁静态分析沙箱Docker / SSHLocal/Docker/SSH/Modal/Daytona/Singularity供应链npm 签名验证Tirith cosign + SHA-256 供应链证明小结：两个框架的安全重心落在不同层——OpenClaw 偏底层（网络层 TLS Pinning + SSRF 防护、身份层 Ed25519 签名 + RBAC），Hermes 偏应用层（Tirith Rust 扫描 + cosign 供应链验证 + Smart Approval 三态评估 + 8 种沙箱后端含 Modal/Singularity）。从覆盖广度看 Hermes 的应用层防御更密集，从底层防护看 OpenClaw 做得更扎实——对沙箱隔离/内容扫描要求高的场景倾向 Hermes，对跨机器身份认证/RPC 安全要求高的场景倾向 OpenClaw。
### 21.6 子 Agent / 委派对比
维度OpenClawHermes Agent机制多 Agent 路由 + Session 隔离delegate_tool 子 Agent 并行并发Session 级并行（无进程内并发限制）ThreadPoolExecutor（最大 3 并发）嵌套无限制（Session 隔离）默认深度 1（可配置 max_spawn_depth）工具隔离插件级隔离阻止列表（禁止 delegate_task/memory/send_message）记忆共享通过记忆插件子 Agent skip_memory，共享 session DB小结：对应两种**"任务分解哲学"——OpenClaw 用 Multi-Agent 路由实现"职责分解"（不同 Agent 服务不同用户/角色，物理隔离），Hermes 用 delegate_tool 实现"任务分解"（同一 Agent 把复杂任务拆给子 Agent 并行）。前者解决"多人协作"，后者解决"单任务复杂度"。背后共通的工程模式是"状态隔离：串行 + 多实例"**——只要问一句"并发修改后下次读取会不一致吗？"，答案是"会"就该走这条路，把并发问题转化成实例数量问题。跨架构编排讨论见"写在最后"第 4 点。
### 21.7 Prompt 缓存对比
维度OpenClawHermes Agent策略依赖 Provider 侧缓存主动 system_and_3（4 个 cache_control 断点）系统提示每次 buildPrompt() 动态构建首轮构建后冻结（跨轮次稳定）记忆注入时机每次 Prompt 构建时仅新会话开始时（当前会话冻结）成本节省取决于 Provider~75% 输入 token 成本（Anthropic）动态性高（记忆实时反映）低（记忆延迟一个会话）小结：最有意思的是 "动态性 vs 命中率"的取舍——OpenClaw 每次 buildPrompt 动态构建（记忆实时反映但缓存命中率低），Hermes 首轮冻结快照（命中率 75% 但记忆延迟一个会话）。这是个没有"正确答案"的工程取舍：成本敏感选 Hermes，记忆驱动场景选 OpenClaw。
### 21.8 记忆+检索方案
业界目前对 Agent 记忆+检索方案的讨论，主要围绕三种范式展开——Static RAG、Agentic RAG、LCM（Lossless Context Management），核心区别在于检索策略和数据建模方式：
维度Static RAGAgentic RAGLCM一句话一次检索 + 一次生成多次检索 + 反思循环DAG 建模 + 按需下钻检索次数1 次（固定）多次（LLM 自主决定）按需沿因果链下钻数据建模扁平索引（向量/BM25）扁平索引（向量/BM25）DAG（保留因果/时序关系）信息保留有损有损无损能否反思❌✅✅典型实现LangChain RetrievalQASelf-RAG, Multi-Hop, Adaptivelossless-claw（lcm_grep / lcm_expand）三者不是递进替代关系，而是解决不同层面的问题：Static RAG 和 Agentic RAG 关注**"怎么检索"（一次 vs 多次+反思，作用对象可以是外部文档、对话历史或记忆），LCM 关注"怎么建模上下文"**（用 DAG 替代线性流水，确保信息无损）。一个成熟的 Agent 可能同时需要 Agentic RAG 做多轮检索 + LCM 管理对话历史。
LCM 的核心思想：传统 Agent 的对话历史是线性流水——消息按时间追加，超出窗口就压缩或丢弃。LCM 把对话历史建模为有向无环图（DAG）：原始消息永久存储，逐层生成摘要节点（叶摘要 → 浓缩摘要），Agent 日常只看摘要 + 最近原始消息，需要历史细节时沿 DAG 逐级展开回溯。类似 Git 的 commit graph——可以 checkout 到任意历史节点看当时的完整上下文，信息永远不丢。
两个框架在三种范式中的位置：
- Static RAG：两框架都已具备——OpenClaw 的 Memory Search 单次召回，Hermes 的 MemoryManager + 8 插件提供者
- Agentic RAG：两框架都有潜力但尚未成体系——OpenClaw 可通过 hooks 串联多次召回但无内置 Self-RAG，Hermes 的 Memory Nudge 是周期性反思而非严格的检索-反思循环
- LCM：两框架核心都未覆盖——真正实现 LCM 的是 OpenClaw 第三方插件 **lossless-claw**（DAG + SQLite 持久化 + lcm_expand_query 子 Agent 下钻）
- 自动记忆整理：OpenClaw 独有 Dreaming 三阶段（Light/REM/Deep 加权晋升），Hermes 无等价物
lossless-claw 存在本身，就是"微内核 + 插件"长期红利的具体体现：核心不够的能力，生态可以补。