# Part I - OpenClaw TypeScript 微内核架构

- **来源：** 腾讯技术工程（微信公众号）
- **作者：** rianli
- **日期：** 2026-05-29
- **原文链接：** https://mp.weixin.qq.com/s/49dxdMXEUoWIYlIh8fFqMQ
- **系列：** OpenClaw与Hermes源码复盘（Part I）

---

Part I 深入剖析 OpenClaw 的设计原理、Gateway 核心、插件系统、Agent 执行引擎、记忆系统、安全机制等完整架构，并以 QQ Bot 插件为实战案例。
版本说明：本文已基于 OpenClaw v2026.5.6更新。
### 1. 设计原理
### 1.1 OpenClaw 解决什么问题
传统 AI 助手存在三个核心痛点：
痛点传统方案OpenClaw 的解法平台锁定每个通道需要独立开发 Bot一个 Agent 实例，通过 Channel Plugin 接入 20+ 通道能力割裂能调工具但缺乏安全管控Agent 执行 shell、读写文件、调用工具，同时有五层纵深防御 + 审批机制兜底隐私失控数据流经第三方服务控制平面和状态数据留在本地设备，仅 LLM 推理请求出站（本地模型则完全离线）
### 1.2 核心设计理念
本地优先（Local-First）：OpenClaw 不是云服务，而是运行在用户设备上的 Gateway 进程。所有会话数据、配置、媒体文件都存储在 ~/.openclaw/ 目录下。Gateway 是控制平面，Agent 是产品本身。
万物皆插件（Everything is a Plugin）：核心代码只负责编排——消息路由、会话管理、安全网关。所有具体能力（Discord 通道、Anthropic 模型、浏览器工具）都以插件形式实现，统一通过 Plugin SDK 注册。
安全纵深（Defense in Depth）：不是简单的"开或关"，而是五层递进防御——从网络层 TLS 到认证层 Device Identity，从命令执行审批到插件安装扫描，再到沙箱隔离。执行策略默认为 deny，所有 shell 命令需要通过白名单或人工审批。插件安装时进行静态代码扫描，发现危险模式直接阻断：
"Security in OpenClaw is a deliberate tradeoff: strong defaults without killing capability."
记忆驱动（Memory-Driven）：Agent 不仅有静态的工作区文件（SOUL.md, USER.md, MEMORY.md）定义人格与记忆，还有向量记忆引擎实现混合搜索、Dreaming 后台整合和 Active Recall 主动召回。需要注意的是，记忆按 Agent 维度隔离——同一个 Agent 下所有用户共享记忆（因为 OpenClaw 定位为个人 AI Agent）。多用户场景下，可通过多 Agent 路由绑定（第 4.3 章）为不同用户分配独立 Agent，从而实现记忆隔离。
配置驱动（Config-Driven）：一个 JSON 文件（~/.openclaw/openclaw.json）定义所有行为——Agent 配置、Channel 凭证、模型选择、安全策略、定时任务。支持运行时热重载，改配置不需要重启。
### 1.3 架构全景
从宏观视角看，OpenClaw 的架构可以分为五层：
- 触达层：用户通过各种消息平台与 Agent 交互，每个平台对应一个 Channel Plugin
- 编排层：Gateway 负责消息路由、Agent 调度、安全控制和配置管理
- 能力层：所有具体功能以插件形式提供，通过 Plugin SDK 与核心交互
- 记忆层：向量记忆引擎、Dreaming 后台整合、Active Recall 主动召回（第 7 章详述）
- 模型层：支持 9 种 LLM API 协议，多模型降级链
### 2. 整体架构详解
OpenClaw 是一个以 Gateway 为中心 的 AI Agent 平台，采用 TypeScript（ESM）构建。通过插件化架构连接消息通道、LLM 提供商和工具扩展，实现「一个 Agent，多端触达」。
核心数据流
### 3. Gateway — 系统心脏
Gateway 是整个系统的中枢，默认监听 :18789。它的职责远不止消息收发——聊天、会话、配置热加载、模型目录、执行审批、定时任务、远程节点、语音唤醒等几乎所有功能域都通过它的 RPC 方法暴露和调度，是名副其实的微内核中枢。
### 3.1 启动流程
### 3.2 连接认证流程
Gateway 采用 Challenge-Response + Device Identity 认证：
先厘清：这里的 "Client" 指什么
Client 指**一切独立于 Gateway 进程、通过 WebSocket 主动连入 Gateway 的"操作端"**。
同机 Client 默认走 ws://127.0.0.1（loopback 明文），跨机 Client 强制 wss:// + TLS 指纹 Pinning。下图中的 Client 指 TUI, Control UI, Mobile App, Node-Host 等操作端，不包括 Channel 插件（Channel 是进程内模块，不走 WebSocket 握手）。握手流程对所有 Client 一致：
容易混淆的一组概念：Client vs Channel
简单记：Client 是"谁在操作 Agent"，Channel 是"Agent 通过哪条线路收发消息"——两者正交。
- Client 是 Gateway 外部的连接方——TUI 、Control UI、原生OpenClaw App 、Web 聊天页面），也可以是程序。所有 Client 都通过 WebSocket 连入 Gateway，走 Ed25519 认证。
- Channel 是 Gateway 内部加载的插件模块，负责对接一个具体的 IM 平台。它跟 Gateway 之间是函数调用（不需要 WS、不需要鉴权），但它自己会向外连接对应平台的接口——QQ Bot 通过 WebSocket 接收事件 + HTTP 调用 OpenAPI，飞书走 HTTP + Event 订阅，Telegram 走 long poll 或 webhook。
两者通过 SessionKey 交汇：同一个用户可以在手机 OpenClaw App（Client）上看到 QQ Channel 产生的对话，也能在 TUI（Client）里继续回复。SessionKey 把"谁在操作"和"哪条线路"绑在一起（格式 agent:{agentId}:{channelId}:...，详见 §4.1）。
安全约束：
- 非 loopback 地址强制 TLS（拒绝明文 ws://，CWE-319）
- TLS SHA-256 证书指纹 Pinning
- 控制平面写操作限流（consumeControlPlaneWriteBudget）
- RBAC Scope 最小权限校验
### 3.3 RPC 方法体系
上述职责在源码中通过 server-methods.ts（39 个直接注册）+ server-aux-handlers.ts（3 个懒加载）共计 42 个 RPC handler 模块落地，下图按功能域归纳为十余类：
### 3.4 方法授权流程
### 3.5 Gateway 的 5 大角色与"边界 vs 实现"哲学
把 Gateway 定位为"操作系统内核"——它不是一个普通的消息网关，而是 OpenClaw 区别于 Hermes, Claude Code 等单体 Agent 框架的根本架构选择。
Gateway 同时承担 5 大角色：
角色 1：唯一长驻进程（Single Source of Truth）
"A single long-lived Gateway owns all messaging surfaces" "One Gateway per host; it is the only place that opens a WhatsApp session."
避免多进程下的 "WhatsApp 二次扫码、Telegram session 冲突" 等致命问题——channel session 天然是状态强相关的，不能多进程并发持有。
角色 2：消息总线（一切流量必经之路）
所有 channel, client, node 流量都走Gateway或由Gateway分发（默认 127.0.0.1:18789）：
- 用户聊天（req:agent, event:agent 流式）
- 控制操作（health, status 、send）
- 节点能力（canvas.*, camera.* 、screen.record, location.get）
- 心跳事件（event:tick）+ 状态广播（event:presence）
设计哲学：不分协议入口 —— HTTP, SSE、私有 RPC 全部统一到 WS Schema。
角色 3：多 Agent 路由的物理边界
通过 Multi-Agent Router 做 Agent 隔离：
- 来自 Telegram@user1 的消息 → 路由到 Agent A
- 来自 Discord@user2 的消息 → 路由到 Agent B
- 不同 Agent 物理隔离（独立 workspace, SOUL, MEMORY, sessions）
这是 OpenClaw 最关键的差异化能力 —— 解决了单 Agent 的三个瓶颈：
- 上下文污染：不同任务（商业文案 vs 写代码）语气切换困难 → 各 Agent 各自的 SOUL.md
- 工具链冲突：工具过多时 LLM 注意力分散 → 各 Agent 只挂自己需要的工具
- 渠道风格差异：飞书要严谨、Telegram 可随意 → 按渠道绑定不同 Agent 人格
对比 Hermes：
- ❌ Hermes：一份 USER.md 多用户共享 → 串扰
- ✅ OpenClaw：多 Agent 物理隔离 → 不串扰
没 Gateway 这个上层路由，就做不到。
角色 4：认证 + 信任边界
场景认证方式同主机 loopback自动信任（auto-approve）Tailnet, LAN必须 connect.challenge 签名 + 配对审批Tailscale Serve、反向代理通过 header 注入身份私网 ingress可配 gateway.auth.mode: "none"公网 ingress强制 shared-secret + idempotency key关键设计：
- Pairing v3 协议：connect.challenge 包含 platform + deviceFamily，变更必须重配对
- idempotency key 必填：send, agent 等副作用操作可安全重试（分布式标准做法，多数 Agent 框架没做）
- Token-based device identity：首次配对后用 device token 长期连接
意义：一个 Gateway 同时承担"消息路由 + 认证 + 信任根" —— 不需要再叠 nginx、网关。
角色 5：嵌入式 HTTP Host（不只是 WS）
Gateway HTTP Host（同端口 18789）：
- /__openclaw__/canvas/    ← Agent 可编辑的 HTML/CSS/JS
- /__openclaw__/a2ui/      ← A2UI 主机界面
意义：Agent 可以主动构造 UI（canvas）让用户在浏览器看，不需要单独起 web server。
"边界 vs 实现"哲学 —— 微内核保持几千行的根本原因
OpenClaw 架构里，Gateway 是边界，不是实现：
事谁做协议定义（WS schema）Gateway路由 + 认证GatewayWhatsApp 会话生命周期GatewayAgent 推理Embedded Pi Runtime（Gateway 内嵌）Channel 消息收发Channel PluginsMemory 整理memory-core 插件工具执行Plugins, SkillsGateway 自己只做"协议 + 路由 + 信任"，其他全是插件 —— 这才能保证微内核保持几千行核心代码。
一些关键工程细节
- 默认 127.0.0.1:18789 不对外 —— 安全默认值（secure by default），主动配置才暴露
- First frame 必须是 connect —— 握手原子化，握手失败立刻断连，无半连接
- hello-ok.features.methods/events 动态发现 —— 客户端不需预知服务端能力，连上后服务端告诉你"我支持哪些方法/事件"
- 写操作（chat.send, agent 等会改变状态的方法）必须带 idempotency key —— 分布式系统标准做法，但多数 Agent 框架没做
### 4. 消息路由 — Session Key 机制
OpenClaw 通过 Session Key 实现消息到 Agent 的精确路由。
### 4.1 Session Key 格式
agent:{agentId}:{scope}
场景Session Key 示例默认主会话agent:main:mainQQ 私聊agent:main:qqbot:default:direct:207A5B83...Discord 群组agent:support:discord:acc1:group:123456789Telegram 线程agent:main:telegram:bot1:direct:user456:thread:msg789DM 隔离策略：通过 session.dmScope 配置控制私聊会话的隔离粒度。默认 per-channel-peer（同一用户同一 Channel 共享会话）；多账号场景可设为 per-account-channel-peer（同一用户通过不同 Bot 账号分别独立会话）。
### 4.2 多 Agent 路由绑定
OpenClaw 支持在同一 Gateway 下运行多个 Agent，通过 agents.bindings 配置将不同来源的消息路由到不同 Agent。每个 Agent 拥有独立的工作区（人格/记忆/Dreaming）。
配置示例（openclaw.json）：
{
  "agents": {
    "list": {
      "support": { "model": "anthropic/claude-opus-4-6", "identity": "客服助手" },
      "dev": { "model": "openai/gpt-4o", "identity": "技术顾问" }
    },
    "bindings": [
      { "match": { "channel": "qqbot", "peer": { "kind": "direct", "id": "207A5B83..." } }, "agentId": "support" },
      { "match": { "channel": "qqbot", "peer": { "kind": "group", "id": "GROUP_123" } }, "agentId": "dev" },
      { "match": { "channel": "discord", "guildId": "987654321" }, "agentId": "dev" }
    ]
  }
}
路由匹配按优先级逐级尝试（源码 resolve-route.ts）：
优先级匹配维度说明示例1binding.peer精确用户QQ 用户 A → support Agent2binding.peer.parent线程父级Telegram 线程继承父会话绑定3binding.peer.wildcard同类型通配所有 QQ 私聊 → 同一 Agent4binding.guild+roles服务器 + 角色Discord 管理员 → dev Agent5binding.guild服务器/组织Discord 服务器 987654321 → dev6binding.team团队MS Teams team → ops Agent7binding.accountBot 账号qqbot:bot2 的消息 → bot2 Agent8binding.channel整个通道所有 Discord 消息 → dev9default未匹配兜底到 main Agent各 Agent 的工作区目录隔离：
~/.openclaw/
├── workspace/                  ← main Agent 的工作区
│   ├── SOUL.md, USER.md       ← 人格与用户画像
│   ├── MEMORY.md               ← 持久记忆
│   └── memory/                 ← 每日记忆文件（YYYY-MM-DD-slug.md）
├── workspace-support/          ← support Agent 的工作区（结构同上）
├── workspace-dev/              ← dev Agent 的工作区（结构同上）
└── agents/                     ← 运行时状态（与 workspace 平级但关联）
    ├── main/
    │   ├── agent/              ← Agent 运行时元数据
    │   └── sessions/           ← 会话转录（UUID.jsonl）
    ├── support/sessions/
    └── dev/sessions/
### 4.3 Agent 间通信——不只是各干各的
多 Agent 不只是"路由隔离"——它们之间可以互相调用。OpenClaw 通过 agentToAgent 工具实现 Agent 间通信：
{
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allow": ["main", "coder", "writer"]
    }
  }
}
4 种协作模式（通过 SOUL.md 中的 prompt 工程实现，不是框架内置开关）：
模式底层工具做法适用场景Supervisorsessions_send主 Agent 调度，收到编程需求 → 传给 @coder，写作需求 → 传给 @writer，最后汇总中央统筹Routersessions_send主 Agent 只做路由分发，不参与执行分诊台Pipelinesessions_sendA 的输出是 B 的输入，串行传递翻译 → 润色 → 排版Parallelsessions_spawn主 Agent spawn 多个子代理并行执行，全部完成后汇总（详见 6.11）同时翻译 3 篇文章两套机制的区别：
- sessions_send（Agent 间通信）= 向已有的另一个 Agent 发消息，两个 Agent 各自独立存在
- sessions_spawn（Subagent 委派）= 创建一个临时子 session 执行任务，干完即走
框架通过 maxPingPongTurns（最大 5 轮）防止 Agent 间 sessions_send 无限来回。
4.4 同一 Agent 下的多用户隔离
同一 Agent 下多用户并发使用时，会话隔离但记忆共享：
会话隔离：每个用户的 SessionKey 不同，对话历史存储在独立的 .jsonl 文件中（文件名 = {sessionId}.jsonl，sessionId 为 UUID）：
sessions.json 索引（SessionKey → sessionId 映射）：
  agent:main:qqbot:direct:207A5B83... → c7cbdbf1-...-b303
  agent:main:qqbot:direct:9F3E2C71... → ff4b5290-...-0edc
  agent:main:discord:acc1:direct:123456789 → 69a8d0ce-...-b5d8
对应磁盘文件：
  ~/.openclaw/agents/main/sessions/
      c7cbdbf1-2ef0-4dc3-8e0f-8471e4a2b303.jsonl  ← 用户 A 的对话
      ff4b5290-6aea-48ec-8076-53a5581d0edc.jsonl  ← 用户 B 的对话
      69a8d0ce-6011-49ef-a651-b046d3f6b5d8.jsonl  ← 用户 C 的对话
记忆共享：所有用户的记忆都写入同一个 ~/.openclaw/workspace/memory/ 目录——文件名为 YYYY-MM-DD-slug.md，不含用户标识。LanceDB 向量库也是单一表，无用户分区。
为什么同一 Agent 多用户不是推荐用法？ OpenClaw 定位为个人 AI Agent——设计上假设一个 Agent 服务一个人（或一个角色）。同一 Agent 下多用户共享记忆会导致偏好串扰和敏感信息跨用户可见。如果你的场景是"一个 Bot 对外服务多个用户"（如 QQ Bot 公共助手），正确的做法是为不同用户/用户组配置多 Agent 路由绑定（第 4.2 章），让每个 Agent 拥有独立的 workspace 和记忆。简单说：一个 Agent = 一份记忆 = 一个服务对象，这是 OpenClaw 的记忆隔离模型。
### 5. 插件系统 — 万物皆插件
### 5.1 插件分类
分类代表插件说明ChannelDiscord, Telegram, QQ Bot, Slack, 飞书, WhatsApp, Signal, MS Teams消息通道接入，每个 Channel 一个插件ProviderAnthropic, OpenAI, Google, DeepSeek, Ollama, Groq, BedrockLLM 模型提供商，统一 API 抽象ToolBrowser, Exa, Firecrawl, Tavily, SearXNG, Brave, DuckDuckGoAgent 可调用的外部工具（搜索、浏览等）MediaElevenLabs, Deepgram, MLX Talk, Voice-Call语音合成（TTS）、语音识别（STT）、本地推理MemoryMemory-LanceDB, Memory-Wiki, Active Recall, Dreaming记忆存储、知识库、主动召回、睡眠整理基础设施Diagnostics-OTEL, Device-Pair, Thread-Ownership, Compaction监控、设备配对、会话压缩等内部能力
### 5.2 Channel Plugin 适配器架构
每个 Channel Plugin 由一组可选适配器组成，按需实现：
Channel 完整契约：25+ Adapter
OpenClaw 的 ChannelPlugin 不是简单的"消息适配器"——它同时承担协议适配、身份配对、安全审批、命令路由、配置生命周期、Gateway 协议绑定等角色，是一个完整的 IM 域协作单元。
接口源码：
type ChannelPlugin = {
// ━━━ 必选 4 项 ━━━
  id: ChannelId;                  // 唯一标识（telegram, discord / ...）
  meta: ChannelMeta;              // 元数据（图标、名称、类型）
  capabilities: ChannelCapabilities; // 能力声明
  config: ChannelConfigAdapter;   // 配置加载、校验、解析
// ━━━ Setup 三件套 ━━━
  setupWizard?: ChannelSetupWizard;
  setup?: ChannelSetupAdapter;
  configSchema?: ChannelConfigSchema;
// ━━━ Auth + Security 7 项 ━━━
  auth?: ChannelAuthAdapter;
  pairing?: ChannelPairingAdapter;
  security?: ChannelSecurityAdapter;
  approvalCapability?: ChannelApprovalCapability;
  elevated?: ChannelElevatedAdapter;
  secrets?: ChannelSecretsAdapter;
  allowlist?: ChannelAllowlistAdapter;
// ━━━ Messaging 7 项 ━━━
  messaging?: ChannelMessagingAdapter;
  message?: ChannelMessageAdapterShape;
  outbound?: ChannelOutboundAdapter;
  streaming?: ChannelStreamingAdapter;     // ⭐ per-channel 流式协议
  threading?: ChannelThreadingAdapter;
  mentions?: ChannelMentionAdapter;
  agentPrompt?: ChannelAgentPromptAdapter;
// ━━━ 协作能力 7 项 ━━━
  commands?: ChannelCommandAdapter;
  groups?: ChannelGroupAdapter;
  directory?: ChannelDirectoryAdapter;
  resolver?: ChannelResolverAdapter;
  bindings?: ChannelConfiguredBindingProvider;
  conversationBindings?: ChannelConversationBindingSupport;
  actions?: ChannelMessageActionAdapter;
// ━━━ Gateway + 运维 6 项 ━━━
  gateway?: ChannelGatewayAdapter;          // ⭐ Gateway 协议绑定（核心）
  gatewayMethods?: string[];                // 暴露给 Gateway 的方法列表
  lifecycle?: ChannelLifecycleAdapter;
  status?: ChannelStatusAdapter;
  heartbeat?: ChannelHeartbeatAdapter;
  doctor?: ChannelDoctorAdapter;
  reload?: { configPrefixes: string[] };    // 精细化热重载
// ━━━ 反向工具 ━━━
  agentTools?: ChannelAgentToolFactory;     // ⭐ Channel 给 LLM 提供工具
};
所有槽位都是可选的 —— Telegram/Discord 实现了 30+ 个，简单内部 webhook channel 只需实现 4 个必选 + 5 个可选。
Channel ↔ Gateway 的 5 种交互模式
模式方向说明入站消息Channel → Gateway → AgentChannel inbound 归一化 → Gateway 路由到 Agent出站回复Agent → Gateway → ChannelAgent 出 turn → Gateway 派单 → Channel outbound客户端控制Client → Gateway → ChannelWS method: "telegram.send" → ChannelGatewayAdapter.handle反向工具Channel → AgentagentTools 注册到 Agent tool registry（Telegram 提供查群成员、Discord 提供加 reaction 等）反向通知Channel → Gateway → Clientevent:presence / event:tick 推到所有连 Gateway 的客户端所有 5 种模式都走同一个 WS Schema。
Per-channel Streaming Adapter 是 Channel 的核心价值——LLM 流式输出的"语义"在每个 IM 协议里完全不同（Telegram 用 editMessageText 反复编辑同一条消息、Discord 用 interaction.followUp, iMessage 不支持流式退化为分段发送），Channel 把这些差异封装掉。
Channel Docking 是 OpenClaw 的独门能力——跨 Channel 会话迁移。用户 Alice 在 Telegram 发起会话后想切到 Discord 继续，发 /dock_discord，Gateway 验证 identityLinks 确认两个账号属于同一用户后，保留 session 上下文不变，只换投递地址。不重建 session——相当于"AI 会话的呼叫转移"。
精细化热重载：每个 Channel 声明自己关心哪些 config prefix（如 telegram.bot.*），Gateway 只在对应配置变更时重启该 Channel，不重启整个进程。
Channel 的核心价值就在这里 —— 把"流式 LLM 输出"翻译成每个 IM 协议的最佳呈现。
反向能力：Channel → LLM 工具
agentTools?: ChannelAgentToolFactory —— Channel 可以反向给 LLM 提供工具：
- Telegram Channel 提供 telegram_get_chat_members 工具 → LLM 可以查群成员
- Discord Channel 提供 discord_react 工具 → LLM 可以加 reaction
- Slack Channel 提供 slack_pin_message 工具 → LLM 可以钉消息
Channel 不只是消息通道，还是 LLM 的能力扩展源。
Channel Docking — 跨 Channel 会话迁移（独门能力）
源码 docs/concepts/channel-docking.md：
用户 Alice 同时在 Telegram 和 Discord 用 OpenClaw
    ↓
identityLinks: { alice: ["telegram:123", "discord:456"] }
    ↓
Alice 在 Telegram 发起会话 → active session 路由到 telegram:123
    ↓
Alice 想切到 Discord 继续 → 在 Telegram 发 "/dock_discord"
    ↓
OpenClaw 验证：telegram:123 和 discord:456 都属于 alice？是 → 允许
    ↓
保留 session 上下文不变 → 改路由 → 后续回复发到 discord:456
实现机制：
- Gateway 自动为每个 Channel 插件生成 /dock-{channel} 和 /dock_{channel} 命令（auto-reply/commands-registry.data.ts:22）
- Session 层有 identityLinks 配置
- 不重建 session —— 只换"投递地址"
**这是 Hermes, Claude Code 等单 channel 框架做不到的"call forwarding for AI session"**。
精细化热重载
reload: { configPrefixes: ["telegram.bot."] }
这告诉 Gateway：当用户修改 telegram.bot.* 任何配置时，只重启 telegram channel，不重启整个 Gateway —— 精细化热重载。
对照：OpenClaw Channel vs Hermes Channel
维度Hermes ChannelOpenClaw Channel抽象层级函数式 send/recv25+ 个可选槽位的完整契约Setup 流程改源码、手填配置SetupWizard + Schema + UI 引导认证API Key 写文件Auth + Pairing + Security + Approval + Elevated 5 层Streaming单一实现Per-channel Streaming AdapterDocking❌✅ Cross-channel session forwardingDoctor❌✅ 自诊断热重载重启✅ 精细化 reload prefix反向工具❌✅ Channel 给 LLM 提供工具设计取舍：Hermes 把 Channel 当消息收发管道——轻量、容易加新平台；OpenClaw 把 Channel 当需要长期维护的平台集成点——重、但加上之后不用再操心认证/重载/诊断。
### 5.3 插件注册模式
### 5.4 插件发现与加载
安全检查：
- 路径遍历防护（拒绝 source 逃逸 rootDir）
- 文件权限检查（拒绝 world-writable）
- 所有权校验（uid 匹配）
- 安装时静态代码扫描
### 5.5 插件安装安全扫描
### 6. Agent 执行引擎
OpenClaw 的 Agent Runtime 本质是一个**"调度 + 容错 + 预算"的编排核——它不直接承担"如何思考"，而是通过 hook 和插件把具体能力外包出去，自己专注于三件事：决定调谁（调度）、失败了怎么办（容错）、花多少资源（预算）**。这让 runtime 核心保持在几千行代码量级，却支撑起了完整的多用户、多通道、多模型的生产级能力。
** 底层引擎：@mariozechner/pi-agent-core（ReAct 循环的工程级实现）**
OpenClaw 的 Agent 执行循环建立在一个独立的底层包 @mariozechner/pi-agent-core之上——由 OpenClaw 创始人 Mario Zechner 维护。这个包实现的就是经典的 ReAct（Reason + Act）模式：
agentLoop(prompts, context, config):
  while (未结束):
    convertToLlm(context.messages)     → 准备 LLM 可理解的 Message[]
    streamFn(messages, model, tools)   → 调 LLM，流式返回（Reason）
    解析 assistant response:
      ├─ 有 toolCall → beforeToolCall → 执行工具 → afterToolCall → 结果加入 context → 继续（Act）
      └─ 无 toolCall → 结束
**pi-agent-core 只负责"循环本身"**——它不懂预算、不懂容错、不懂通道路由。OpenClaw 在它之上叠加了所有生产级能力：
层谁负责做什么循环层pi-agent-coreReAct 循环、工具调用（parallel/sequential）、流式输出、上下文转换编排层OpenClaw pi-embedded-runner/预算控制、Auth Profile failover, Compaction, Lane 分车道、Bootstrap 注入拦截层OpenClaw hooksbeforeToolCall（审批/安全扫描）、afterToolCall（截断/日志）、transformContext（Compaction）能力层OpenClaw plugins循环本身不提供的外部能力——记忆检索、消息出站、模型适配等，按需调用、可插拔关键设计点（来自源码 pi-agent-core/dist/types.d.ts）：
- AgentMessage ≠ LLM Message：内部用自定义的 AgentMessage（支持 compactionSummary, notification, steering 等非 LLM 消息类型），只在调 LLM 边界才通过 convertToLlm 转成标准 Message[]。这让 OpenClaw 可以在历史里插入 Compaction 标记、Bootstrap 截断告警等"Agent 自己看的消息"而不污染 LLM 输入
- StreamFn 可替换：默认用 pi-ai 的 streamSimple 调 LLM API，但可以换成自定义函数——OpenClaw 的 CLI Backend 就是用这个把 Claude Code 的 stdio 流当作"LLM 响应"
- beforeToolCall, afterToolCall：工具执行前后的拦截点——OpenClaw 用 beforeToolCall 实现 Exec Approval（危险命令审批），用 afterToolCall 实现 tool result truncation（超 16K 字符截断）
- transformContext：每次调 LLM 前的上下文变换钩子——OpenClaw 用它实现 Compaction（压缩中段历史释放 token 空间）
和 Hermes 的对比：Hermes 的 AIAgent.run_conversation() 也是 ReAct 循环，但循环和编排耦合在同一个万行类里——没有独立的"循环层"。OpenClaw 把循环抽成独立包的好处是：升级 ReAct 策略（如从 sequential 改 parallel tool call）不需要动编排逻辑，反之亦然。
### 6.1 分层执行架构
OpenClaw 的 Agent 入站有三条路径，最终都汇聚到同一条执行链路上：
关于入站层的两点澄清：
- ACP Server 是"经 Gateway 入站"的——openclaw acp 启动一个 ACP 前端（供 Zed, Copilot CLI 等 IDE 连接），但它收到 session/prompt 后会通过 gateway.request("chat.send", ...) 把请求转发到 Gateway（见 src/acp/translator.ts），和 QQ Bot、飞书走同一条入站路径。所以 ACP 不是"另一个入口平行于 Gateway"，而是"Gateway 的一个前端协议适配器"——这就是第 22 章说的 ACP Bridge 模式。
- CLI 有两种模式——openclaw tui 默认通过 WS 连接 Gateway（和 Control UI / Mobile App 走同一条路径），而 openclaw chat（tui 的别名）默认启用 --local，在进程内启动嵌入式 Agent 运行时（EmbeddedTuiBackend），不经过 Gateway RPC。Local 模式让开发调试不需要先拉起 Gateway，但代价是不受 Gateway 上的审批/速率限制策略管控（local 模式走本地 TUI 审批）。
关于 Provider 层的三路分叉：注意这里的 DECIDE 不是"会话类型"而是"provider 类型"——Hermes 的 ACP 客户端（copilot_acp_client.py）和 OpenClaw 的 acpManager.runTurn 做的是同一件事：把 ACP 反过来当 LLM provider 用。当你想用 GitHub Copilot 订阅额度跑 Agent 时，就会走这条分支（不是 ACP server 入站）。
三层错误边界：
- 内层（runEmbeddedAttempt）：LLM 调用 + 工具执行的一次尝试，失败时抛 FailoverError
- 中层（runEmbeddedPiAgent）：接住 FailoverError，决定是换 Auth Profile 重试还是向上抛
- 外层（runWithModelFallback）：最终接住不可恢复的 FailoverError，遍历 model.fallbacks[] 切换模型
这是 OpenClaw 容错设计的核心——可恢复错误的处理是静态可证明的，不靠 LLM 猜。
runEmbeddedPiAgent 主循环深入剖析
runEmbeddedPiAgent（src/agents/pi-embedded-runner/run.ts，约 1000 行）是 6.1 架构图中核心层的中央——非 CLI/ACP provider 的所有请求最终都汇聚到这里。名字里的 Embedded 表示"直接调 Provider SDK、不 spawn 子进程"，Pi 表示构建在 @mariozechner/pi-agent-core 之上——OpenClaw 没有自己写 Agent 核心循环，而是包装 pi-agent-core 并在外面套多 provider 适配 + 容错降级 + Hook 触发 + 缓存追踪。
三段结构
export asyncfunction runEmbeddedPiAgent(params): Promise<EmbeddedPiRunResult> {
// ─── 阶段 1: 一次性初始化（循环外，高成本 IO 只做一次）───
const sessionLane = resolveSessionLane(params.sessionKey);
const globalLane = resolveGlobalLane(params.lane);
const authController = createEmbeddedRunAuthController({ ... });
await authController.initializeAuthProfile();
const contextEngine = await resolveContextEngine(params.config);  // 跨重试复用
// ─── 阶段 2: 预算常量与计数器 ───
const MAX_RUN_LOOP_ITERATIONS = resolveMaxRunRetryIterations(profileCandidates.length);
let runLoopIterations = 0;
// ... 其他计数器：overflowCompactionAttempts, timeoutCompactionAttempts ...
// ─── 阶段 3: 主循环（真正的重试-降级-恢复）───
while (true) {
    if (runLoopIterations >= MAX_RUN_LOOP_ITERATIONS) return retryLimitExceededResult();
    runLoopIterations += 1;
    const attempt = await runEmbeddedAttempt({ ... });
    // 七类分支按优先级判断（顺序不能乱）
    if (attempt.aborted) return abortedResult();
    if (consumeLiveSessionModelSwitch(...)) thrownew LiveSessionModelSwitchError(...);
    if (timedOut && tokenUsedRatio > 0.65) { /* Timeout Compaction */continue; }
    if (contextOverflowError) { /* 三级降级：compact → truncate → 抛错 */continue; }
    if (assistantErrorText) { /* 分类：auth 刷新 / overloaded backoff / 轮换 profile / 抛 FailoverError */continue; }
    await markAuthProfileGood({ profileId: lastProfileId });
    return successResult({ payloads: attempt.payloads, ... });
  }
}
关键设计
1. 双 Lane 排队：同时持有 globalLane（调用类型：Default/Nested/Subagent/Cron）和 sessionLane（sessionKey 哈希）。双锁意义——一个 Cron 任务和用户对话即使打到同一会话也会被 sessionLane 强制串行，不同会话的 Cron 之间互不阻塞。
2. 七类分支顺序决定正确性：
优先级分支不能放后面的原因1aborted用户中断必须立即响应2live model switch要在产生任何副作用前重启3timed out + 高 token预防性主动压缩，避免下次又被 timeout kill4context overflow三级降级（compact → truncate → 抛错）逐级恶化5assistant error分类后选 profile 轮换、token 刷新、overloaded backoff6success path成功路径7兜底迭代上限 → 抛错两个关键细节：timeout compaction 必须在 overflow 之前——timed out + 65% context 是预防性信号（LLM 还没报错，但延迟暗示 prefill 慢），先走 overflow 会等到下次明确报错，但那时可能直接被 timeout kill；truncate 是 overflow 的最后手段——截断会永久删除 tool 输出并写回 session.json，只在 compaction 都失败后才用，整个 run 只用一次。
3. runEmbeddedAttempt 是"跑一次完整 Agent 轮次"：外层 runEmbeddedPiAgent永远不直接调 stream，所有 LLM 交互在 runEmbeddedAttempt（run/attempt.ts，2000+ 行）内完成——Bootstrap 上下文加载 → buildSystemPrompt → 创建 pi-agent-core session → session.run()（内部是 pi-agent-core 的多轮 LLM↔Tool 循环）。关注点分离：attempt 做"跑一轮"，外层做"重试到成功或策略耗尽"。
4. Auth Controller 封装凭证决策：createEmbeddedRunAuthController 对外只暴露 initializeAuthProfile, advanceAuthProfile, maybeRefreshRuntimeAuthForAuthError, stopRuntimeAuthRefreshTimer 4 个方法。外层主循环看不到"profile cooldown / token refresh / probe slot"这些复杂度，全被收进 auth-controller.ts。
5. FailoverError 是与外层的唯一契约：主循环里所有"可恢复错误"最终都 throw new FailoverError(reason, ...)，调用者 runWithModelFallback 只接 FailoverError（instanceof 匹配就换模型，否则直接抛）。**runEmbeddedPiAgent 是"FailoverError 工厂"，runWithModelFallback 是"消费者"——两者只通过这一个错误类型交流**（详见 §6.3）。
6. Live Model Switch 的幂等条件：只有完全干净的 attempt（没发消息、没执行工具、没产生 assistant 文本、没审批提示、没工具错误）才允许实时切模型。一旦对外产生过影响，切模型重来就会导致重复发送或不可撤销操作——这些条件一旦触发就锁死 live switch 路径。
换句话说，runEmbeddedPiAgent 本身不调 LLM、不执行工具、不构建 prompt（这些都委托给 runEmbeddedAttempt），它只做一件事：反复尝试，直到成功或把错误以 FailoverError 抛给上层。
### 6.2 Auth Profile——不只是"API Key 数组"
先看一个真实场景：你有 3 个 Anthropic 账号——个人 Pro 订阅、公司 Max 订阅、一个 AWS Bedrock 账号。你想让 OpenClaw 自动管理这 3 个账号：Pro 额度用完了自动切 Max，Max 被限频了自动切 Bedrock，任何一个恢复了自动切回来。
Hermes 做不到——它的 Credential Pool 只是 API Key 数组，按顺序试，失败了不知道为什么失败，也不记得"上次哪个 key 挂了"。
OpenClaw 的 Auth Profile 把每个账号建模为带健康状态的对象：
你的 3 个 Profile：
Profile A: "个人Pro"
  ├─ 类型: OAuth（可自动刷新 token）
  ├─ 状态: ⚠️ 冷却中（billing 错误，5min 后重试）
  └─ 冷却原因: 当日额度用完
Profile B: "公司Max"
  ├─ 类型: API Key
  ├─ 状态: ✅ 可用（上次用于 30s 前）
  └─ 冷却原因: 无
Profile C: "AWS Bedrock"
  ├─ 类型: Token（带过期时间）
  ├─ 状态: ✅ 可用（token 2h 后过期）
  └─ 冷却原因: 无
当请求失败时的行为差异：
场景：用 Profile A 调 Claude，返回 429 rate_limit
Hermes 的做法：
  retry → retry → retry → 超时报错（不知道该切 key）
OpenClaw 的做法：
  ① 识别错误类型 = rate_limit（凭证类）
  ② Profile A 标记冷却 30s
  ③ 50ms 内切到 Profile B
  ④ 用户无感知，对话继续
  ⑤ 30s 后 Profile A 自动"探针重试"——如果恢复了加回可用队列
数据结构（auth-profiles/types.ts）：
type AuthProfileCredential =
  | ApiKeyCredential       // { key, provider }          ← 最简单
  | TokenCredential        // { token, expiresAt }       ← 会过期，到期自动换下一个
  | OAuthCredential;       // { clientId, refreshToken } ← 能自动刷新，最持久
type ProfileUsageStats = {
  lastUsed: number;
  cooldownUntil: number;          // 临时退避：30s → 1min → 5min（指数退避）
  cooldownReason: "rate_limit" | "overloaded" | "billing" | ...;
  disabledUntil: number;          // 永久型错误（key 被吊销）
  failureCounts: Record<FailureReason, number>;
};
选取策略（auth-profiles/order.ts）——决定"下一个用哪个 Profile"：
- 类型偏好：oauth > token > api_key（能自动刷新的优先——活得更久）
- 均衡轮转：同类型按 lastUsed 升序（不让一个 key 被打爆）
- 冷却探针：冷却中的 profile 排末尾，到期后自动试一次——恢复了就回主队列
- 用户锁定：显式指定的 preferredProfile 永远优先（调试/测试用）
和 Hermes Credential Pool 的关键差异：
维度Hermes Credential PoolOpenClaw Auth Profile抽象层级API Key 数组凭据 + 健康状态 + 来源凭据类型只支持 api_keyapi_key, token（带过期）/ oauth（可刷新）持久化进程内内存（重启丢失）磁盘 store（~/.openclaw/auth-profiles/），重启保留冷却状态外部同步无通过 external-cli-sync.ts 自动发现本地 claude-cli / codex 已登录的账号选取逻辑线性尝试（从头到尾试）round-robin + 冷却队列 + 类型偏好 + 用户锁定冷却策略统一计数按 FailoverReason 分级退避（rate_limit 30s / billing 5min / auth_permanent 永久禁用）生产体验差异举例：
- Hermes 重启后 → 不记得哪个 key 上次挂了 → 又去撞已知欠费的 key → 白等 30s
- OpenClaw 重启后 → 磁盘 store 里 Profile A 还标着"billing 冷却到 14:30" → 直接跳过 → 0ms 恢复
- 你在终端里跑过 claude-cli login → OpenClaw 自动发现并同步为一个 OAuth Profile → 不用手动配 key
### 6.3 FailoverError——把错误分类做成结构化契约
多数框架遇到错误"抓异常重试"，OpenClaw 把错误的 reason 做成闭合枚举：
// agents/pi-embedded-helpers.ts
type FailoverReason =
  | "billing"          // 402
  | "rate_limit"       // 429
  | "overloaded"       // 503
  | "auth"             // 401（可刷新）
  | "auth_permanent"   // 403（禁用）
  | "timeout"          // 408 / ETIMEDOUT / ECONNRESET...
  | "format"           // 400（payload 问题）
  | "model_not_found"  // 404
  | "session_expired"; // 410
分类器（resolveFailoverReasonFromError）是递归的——逐级走 HTTP status → 符号码（RESOURCE_EXHAUSTED / THROTTLING_EXCEPTION）→ errno → cause 链 → timeout heuristics，容忍不同 Provider SDK 的错误表达差异。
分类结果驱动不同策略（failover-policy.ts）：
策略适用 reason效果shouldAllowCooldownProbeForReasonrate_limit, overloaded, billing允许探针式重试冷却中的 profileshouldUseTransientCooldownProbeSlot瞬时错误（rate_limit, overloaded）走临时 slot，不占用主 profileshouldPreserveTransientCooldownProbeSlot永久错误（auth_permanent, session_expired）保留 slot 供后续复用甚至连不是 API 调用的错误也会被翻译成 FailoverError——context_length_exceeded, session_expired, model_not_found 全走同一条路。这样 runWithModelFallback 能用统一契约处理所有可恢复错误，代价是错误分类器要维护大量启发式规则（值得，因为这部分的边界条件是"外部世界决定的"，不是业务复杂度）。
### 6.4 双路径执行——把 Claude Code, Codex CLI 当 Backend
runAgentAttempt 在 command/attempt-execution.ts 里做一次关键分叉：
isCliProvider?
├─ true  → runCliAgent
│          • 调用 claude-cli, codex-cli 子进程
│          • 通过 cli-session.ts 管理子进程生命周期
│          • 共享同一套 workspace, memory, session 结构
│
└─ false → runEmbeddedPiAgent
           • 通用 pi-agent 引擎（基于 @mariozechner/pi-agent-core）
           • 直接调用 Provider SDK（openai / anthropic / google / ...）
这是微内核架构的真正红利——OpenClaw 不把 Claude Code, Codex CLI 当"竞品"，而是把它们当可替换的执行 backend：同一个 Gateway 管理、同一个 Agent 人格、同一套记忆系统、同一个会话转录格式，只是底层 LLM 调用换了个 Runner。
实际使用中的典型配置：同一个 Gateway 下多个 Agent 各用不同 backend——
- Agent A 用 claude-cli backend → 复用 Claude Code 的文件编辑/终端/浏览器等内置工具链，适合重度编程任务
- Agent B 用 embedded backend + DeepSeek → 自有 API Key 直连，token 成本低，适合日常问答
- Agent C 用 codex-cli backend → 走 ChatGPT Plus 订阅额度，不额外花钱
CLI Backend 的协议适配
把"Claude Code, Codex CLI 当 backend 用"听起来像是接入一个标准协议——但实际上没有这样的协议。这一节讲 OpenClaw 是怎么解决这个问题的。
各家 CLI 的输出协议互不相同
Backend命令输入方式输出协议claude-cliclaude -p --output-format stream-json --verbose --permission-mode bypassPermissionsprompt 作为命令行参数（超长走 stdin）Claude Code 自定义的 stream-json（一行一个 JSON 对象，含 message / tool_use / tool_result 事件）codex-clicodex exec --json --color never --sandbox workspace-write命令行参数Codex 的 JSONL 事件流（fresh）/ 纯文本（resume，因 codex quirk）gemini-cligemini --prompt --output-format json命令行参数单一 JSON 对象输出关键事实：这三种格式互不相同——不是 ACP、不是 MCP、不是 OpenAI Chat Completions、不是 Anthropic Messages，是各家 CLI 各自定义的 stdout 协议。Claude CLI, Codex CLI, Gemini CLI 的输出格式都是为各自 IDE 集成（VSCode 插件等）设计的私有协议，早于 ACP 标准出现。
CliBackendConfig ——配置驱动的适配层
由于没有标准协议，OpenClaw 用一个配置对象来抽象差异（src/config/types.agent-defaults.ts:47）：
export type CliBackendConfig = {
  command: string;            // 可执行文件名
  args?: string[];            // 默认参数
  output?: "json" | "text" | "jsonl";    // 输出解析模式
  resumeOutput?: "json" | "text" | "jsonl"; // resume 模式可独立设置
  input?: "arg" | "stdin";    // prompt 怎么喂
  maxPromptArgChars?: number; // arg 超长就转 stdin
  modelArg?: string;          // 怎么传模型 ID（如 --model）
  modelAliases?: Record<string, string>;  // OpenClaw model id → CLI model id
  sessionArg?: string;        // 怎么传 session id
  sessionMode?: "always" | "existing" | "none";
  sessionIdFields?: string[]; // 从输出哪个字段读 session id
  systemPromptArg?: string;   // 怎么传 system prompt
  systemPromptMode?: "append" | "replace";
  imageArg?: string;          // 怎么传图片
  imageMode?: "repeat" | "list";
  clearEnv?: string[];        // 启动前清的 env vars
  serialize?: boolean;
  reliability?: { watchdog: { fresh: {...}, resume: {...} } };
};
**整个适配器层做的事就是把"OpenClaw 抽象的请求"翻译成"目标 CLI 能听懂的命令行 + stdin"**：
OpenClaw runtime 请求
   ├─ prompt: "Help me debug..."
   ├─ model: "claude-cli/claude-sonnet-4-6"
   ├─ systemPrompt: "You are..."
   └─ sessionId: "abc-123"
                ↓
        buildCliArgs (cli-runner/helpers.ts)
                ↓
        spawn("claude", [
          "-p", "--output-format", "stream-json", "--verbose",
          "--permission-mode", "bypassPermissions",
          "--model", "sonnet",                              ← 经 modelAliases 映射
          "--session-id", "abc-123",                        ← sessionArg 决定
          "--append-system-prompt", "You are...",           ← systemPromptArg 决定
          "Help me debug..."                                 ← input=arg
        ])
                ↓
        子进程 stdout 输出 stream-json
                ↓
        cli-runner/execute.ts 按 output="jsonl" 行解析
                ↓
        翻译回 OpenClaw 抽象的 AgentMessage 流
反向 MCP 注入——隐藏的协议
这是 CLI Backend 设计里最巧妙的一环。Claude CLI 自己有原生工具集（read, write/bash 等），但 OpenClaw 还想让 CLI 能用自己的扩展工具（比如 send_message, subagents 等）。怎么做？
// extensions/anthropic/cli-backend.ts:17
{
  id: CLAUDE_CLI_BACKEND_ID,
  bundleMcp: true,        // ← 关键
  config: { command: "claude", args: [...] },
}
bundleMcp: true 让 OpenClaw 在启动 CLI 时，通过 Claude CLI 的 MCP 配置注入一个本地 MCP 服务器——这个 MCP 服务器就是 OpenClaw 自己跑起来的（src/mcp/channel-server.ts）。Claude CLI 通过 MCP 协议反过来调 OpenClaw 提供的工具：
                    OpenClaw runtime
                         ↓ spawn
              ┌──────────────────────┐
              │  Claude CLI 子进程   │
              │                       │
              │  → Anthropic API      │ ◄── LLM 调用是 CLI 自己做的
              │                       │      （用 CLI 已登录的 OAuth）
              │  → MCP Client         │ ◄── 调 OpenClaw 工具
              └─────────┬─────────────┘
                        │
                  stdio MCP
                        │
                        ▼
              ┌────────────────────┐
              │ OpenClaw MCP Server │ ◄── 提供 send_message /
              │  (channel-server)   │      subagents / 等扩展工具
              └────────────────────┘
所以 CLI ↔ OpenClaw 之间实际是混合协议：LLM 输出走各家 CLI 的私有 stdout 协议，工具调用走 MCP。这是文章第 22 章讲到的"反向 MCP"在 CLI 路径上的第二处应用——不只是面向第三方 IDE 暴露，也面向自己 spawn 的 CLI 子进程暴露。
与 ACP 路径的对比
文章 6.1 图里有三种 provider 类型：embedded, CLI provider, ACP provider。它们的协议是这样的：
Provider 类型协议适用embedded各 LLM 厂商的 HTTP API（Anthropic Messages / OpenAI Responses / Google generateContent / ...）走自己的 API KeyCLI provider每家 CLI 各自的 stdout 流格式（stream-json / jsonl / json）+ 反向 MCP 注入工具复用本地 CLI 的登录态（Claude OAuth, ChatGPT 订阅、Gemini 账号）ACP providerACP 协议（JSON-RPC over stdio，由 @agentclientprotocol/sdk 定义）把别的 ACP agent（GitHub Copilot ACP 等）当 LLM backendACP 路径不是直接调"gemini"或"copilot"的同一个二进制——extensions/acpx/ 是 OpenClaw 的 ACP 客户端代理，它启动的是专门的 ACP wrapper 脚本（如 codex-acp-wrapper.mjs, claude-agent-acp-wrapper.mjs），走标准的 ACP 协议（JSON-RPC over stdio）。支持的 ACP harness（注：harness 是 OpenClaw 内部用语，不是 ACP 协议规范术语，指"被 OpenClaw 通过 ACP 协议驱动的外部 Agent 运行时"，见 extensions/acpx/skills/acp-router/SKILL.md 和 docs/tools/acp-agents.md）包括：codex, claude, gemini, droid, opencode 等。同一个"codex"既可以作为 CLI provider（走 JSONL stdout 流），也可以作为 ACP harness（走 JSON-RPC 协议）——取决于配置里选哪条路径。
设计哲学：CLI Backend 走的是"已存在生态优先"路线——不要求 CLI 厂商支持 ACP，而是用 CliBackendConfig 写适配器吃掉各家差异。
双向连接——OpenClaw 也是别人的 Backend
6.4 和 6.4.1 讲了"OpenClaw 把 CLI 当 backend"的方向。但这个故事还有另一半——OpenClaw 自己也被设计为别人的 backend。这一节讲反方向。
先解决一个疑问：私有协议为什么能"当 backend 用"？
理解反方向之前，要先回答 6.4.1 留下的隐含问题——既然 stream-json, codex jsonl, gemini json 都是私有协议，OpenClaw 为什么能稳定接入？
三个原因：
- 私有协议是"事实开放"的——CLI 厂商为了支持自家 IDE 集成（VSCode 插件, Cursor, Zed），必须让协议对外可解析且稳定。一旦改动会破坏所有下游集成，所以厂商有强烈动机保持向后兼容。OpenClaw 把自己当成"另一个下游集成"——和写一个 VSCode 插件没有本质区别。可以类比 gh CLI 的 --json 输出：没有 RFC 标准，但全世界写脚本的人都在用。
- 协议适配做成可配置层——CliBackendConfig 是外部可注册的（api.registerCliBackend(...)）。任何人都能加一个新 CLI backend，不改 OpenClaw 核心代码。所以"协议私有"不是问题，**问题是"协议是否稳定 + 是否可解析"**——这两条满足了，谁定的协议都不重要。
- OpenClaw 不需要懂 LLM 协议本身——这是最关键的一点：
embedded 路径：
  OpenClaw 必须自己实现 anthropic-messages, openai-responses、
  google-generateContent 等多套 HTTP 协议适配
CLI 路径：
  OpenClaw 只需要 spawn CLI、解析 stdout
  LLM 协议适配的复杂度被 CLI 吃掉了
**复杂度从"N 套 HTTP 协议"降到"N 套 stdout 格式"**——后者天然更简单（行 JSON 比 SSE + tool call schema + thinking blocks 简单太多）。CLI backend 实质是把"LLM 协议适配"委托给 CLI 厂商，OpenClaw 只解决"如何驱动 CLI"这个更窄的问题。
反方向：OpenClaw 提供三种暴露面
那 Claude Code, Codex CLI, Cursor, Zed 这些工具能反过来调用 OpenClaw 吗？能——而且 OpenClaw 主动设计了三种粒度的暴露面：
                外部工具想要什么粒度的访问？
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
         工具粒度     Agent 粒度    系统粒度
              │           │           │
              ▼           ▼           ▼
        MCP Server    ACP Server   HTTP API
路径 1：MCP Server——工具粒度
openclaw mcp serve 把 OpenClaw 暴露为一个 MCP server。任何支持 MCP 的客户端都可以连接：
Claude Code (用户主动启动的)
       ↓ stdio MCP 协议
       ↓
OpenClaw MCP Server (channel-server.ts)
       ↓ 暴露 9 个具体工具：
       │
       ├─ conversations_list      （列出聊天会话）
       ├─ conversation_get        （读会话详情）
       ├─ messages_read           （读 QQ Bot、飞书、Discord 历史消息）
       ├─ messages_send           （从 Claude Code 反向发消息到 QQ Bot）
       ├─ attachments_fetch       （拉取附件）
       ├─ events_poll / events_wait（监听新消息）
       ├─ permissions_list_open   （看待审批列表）
       └─ permissions_respond     （审批/拒绝 Agent 的执行请求）
使用场景：Codex / Claude Code 等 MCP 客户端需要访问 OpenClaw 管理的 IM 会话时（如在 IDE 里直接测试 QQ Bot 消息收发、CI 完成后通过 OpenClaw 向飞书群发通知等），在客户端配置里加一行 openclaw mcp serve，就能通过 conversations_list（列出会话）/ messages_read（读历史）/ events_wait（等新消息）/ messages_send（回复）等标准 MCP 工具操作 OpenClaw 的所有 Channel 会话——coding agent 把 OpenClaw 当成"统一通讯能力扩展"，一个 MCP server 覆盖所有通道。
路径 2：ACP Server——Agent 粒度
openclaw acp 启动 ACP 协议前端。支持 ACP 的 IDE（Zed, Copilot CLI 等）可以把 OpenClaw 当 Agent 用：
Zed IDE
       ↓ ACP 协议（JSON-RPC over stdio）
       ↓
OpenClaw ACP Server (src/acp/server.ts)
       ↓ gateway.request("chat.send", ...)
       ↓
Gateway → agentCommand → runEmbeddedPiAgent
       ↓
LLM 调用 + 工具执行
       ↓ ACP 协议事件流
返回到 Zed
典型场景：用户在 Zed 里有一个聊天面板，里面接的是 OpenClaw 的某个 Agent。用户在 Zed 里发"帮我把今天 QQ Bot 的对话整理成日报"——OpenClaw 用自己的 Agent 跑这个任务，结果通过 ACP 协议流回 Zed 显示。
与 MCP 的区别：MCP 是"暴露几个工具"，ACP 是"暴露整个 Agent"——粒度完全不同。
路径 3：Gateway HTTP API——系统粒度
OpenClaw 的 Gateway 本身有完整的 HTTP API（src/gateway/server-methods/），包括 chat.send, session.list, config.get 等几十个方法。任何能发 HTTP 请求的程序都能调它——curl, Python 脚本, Bot, CI/CD 流水线。
ACP Server 和 MCP Server **本质都是这个 HTTP API 的"协议前端适配器"**——把 ACP/MCP 请求翻译成 Gateway HTTP 调用。
三种路径对比
路径协议谁会用暴露什么MCP ServerMCP（stdio JSON-RPC）Claude Code, Cursor, Zed、自定义 Agent9 个具体工具（消息收发、审批管理）ACP ServerACP（stdio JSON-RPC）Zed, Copilot CLI 等 ACP-aware IDE整个 Agent 能力（一个聊天 session）Gateway APIHTTP REST/RPC任何 HTTP 客户端所有 Gateway 方法（最底层）完整的协议反转图
把"OpenClaw 调 CLI"和"CLI 调 OpenClaw"放在一起看：
方向 A 和方向 B 都用到了 MCP，但角色完全相反——
- A 方向：OpenClaw 是 MCP Server（被自己 spawn 的 Claude CLI 调）
- B 方向：OpenClaw 还是 MCP Server（被用户启动的 Claude Code 调）
差别只在触发者——MCP server 不知道也不关心是谁连过来的，它就是个工具暴露面。
"双向连接"的设计意图
OpenClaw 的选择是不试图取代任何现有工具，而是和它们互联：
场景谁是主，谁是辅用户主要在聊天平台（QQ Bot、飞书）和 Agent 对话OpenClaw 是主，Claude CLI 是 LLM backend用户主要在 Claude Code 里写代码Claude Code 是主，OpenClaw 是消息通道扩展用户在 Zed 里需要一个聊天 AgentZed 是主，OpenClaw 是 Agent 实现外部脚本通过 HTTP 触发任务调用方是主，OpenClaw Gateway API 是执行入口Hermes 的对比：Hermes 也能处于"被调"位置——v0.13 已有 hermes acp 命令可作为 ACP server 供 VS Code, Zed, JetBrains 调用。但 OpenClaw 的"双向"更彻底——同一个进程同时暴露 MCP server + ACP server + HTTP API + WebSocket，且和 CLI Backend 模式并存，让它能在主/辅/中间层任意位置运转。
协议私有不是障碍——只要厂商对外稳定就能适配；而 OpenClaw 不仅消费别人的私有协议，自己还提供 MCP, ACP, HTTP 三层标准化暴露面，让别人也能消费它。这种"既能当主、也能当辅"的双向连接能力，是微内核架构 + Plugin SDK 抽象的红利在协议层的最直接体现。
### 6.5 三级 Compaction 策略
OpenClaw 的上下文压缩不是单一触发点，而是分三级响应：
级别触发条件位置动作L1: Pre-request会话历史 > 阈值run.ts 主循环开头主动调 compaction.ts 生成摘要L2: Timeout-triggeredLLM 首 token 超时 且 prompt > 65% contextrun.ts 主循环紧急压缩，用缩短的 prompt 立即重试L3: Context overflowAPI 返回 context_length_exceededrun.ts 主循环三级降级：auto-compact → 截断 oversized tool result → 抛错L2 是容易忽视但很巧妙的一层——LLM 首 token 慢不一定是服务端慢，很可能是 context 太大导致 prefill 耗时过长。用"超时 + 大 context"双条件判断主动压缩后重试，比盲目切换 profile 更经济（不浪费冷却配额）。
Compaction 本身的实现也比 Hermes 精细一层：
- **identifier-policy**：压缩时保留 symbol identifiers（函数名/文件路径）的出现频次
- **identifier-preservation**：压缩后验证关键 identifier 没丢
- **tool-result-details**：专门处理 tool 调用输出的摘要格式
- **retry**：压缩本身的重试（压缩调用失败时回退到原始 messages，而不是让整个 turn 挂掉）
### 6.6 Context Engine 契约——把"上下文管理"做成插件
src/context-engine/types.ts 定义了一个抽象，把"上下文怎么管"从 runtime 剥离：
interface ContextEngine {
  bootstrap?(ctx)          // 会话初始化
  ingest(msg)              // 吸收一条 message
  ingestBatch?(batch)      // 吸收一轮 turn
  afterTurn?(ctx)          // 一轮结束后做后处理
  assemble(budget)         // 按 tokenBudget 组装 prompt
  compact()                // 压缩
  maintain?()              // 分支重写
  prepareSubagentSpawn?()  // 子 agent 派生前
  onSubagentEnded?()       // 子 agent 结束后
}
为什么这是亮点：它不把"上下文"当成静态的 message 列表，而是让 engine 自己决定：
- assemble 时返回哪些 message（支持检索型引擎——只召回相关的历史，不是全部回放）
- maintain 时可以"分支重写"（修历史消息的 payload 但保留 id）
- afterTurn 时 engine 可以主动触发压缩决策
这为未来接入 RAG-style 上下文管理（GraphRAG, LightRAG 之类）留好了接口——默认实现走"线性 + compaction"，高级场景可以换成"图谱召回"。
### 6.7 Lanes——并发执行的分车道管控
// src/process/lanes.ts
export const enum CommandLane {
  Default = "default",
  Nested = "nested",      // 内嵌 agent（如 dreaming review）
  Subagent = "subagent",  // sessions-spawn 子 agent
  Cron = "cron",          // 定时任务
}
Lanes 不是简单的"线程池"，而是命令调度的隔离通道：
- 不同 lane 的命令队列独立，互不阻塞——cron 任务堆积不会拖垮用户交互
- Nested agent 不继承 cron lane——cron 触发的内嵌 review 不应该再占用 cron 执行槽，否则会形成自我递归的死锁
- Subagent lane 有独立的并发预算，不与用户命令竞争
- Lane 还是权限边界——cron lane 的命令看不到交互审批 UI（没有用户在线），走的是预授权路径
这种"按调用来源分车道"的设计在同类 agent 框架里不多见。Hermes 是单一全局队列，所有命令排同一条线。
### 6.8 Bootstrap Budget——启动上下文的精细预算
Agent 启动时加载 SOUL.md / USER.md / MEMORY.md / AGENTS.md 等工作区文件时，不是"全部读进来"，而是按预算裁剪：
- maxChars, totalMaxChars：单文件与总上限
- BootstrapContextMode：full 或 lightweight（节省 prompt token）
- BootstrapContextRunKind：
- default：正常加载所有 bootstrap 文件
- heartbeat：只加载 HEARTBEAT.md（定时心跳不需要完整人格）
- cron：默认不加载任何 bootstrap 文件（cron 任务通常只需要特定上下文）
- bootstrap-hooks.ts 允许 hook 动态修改 bootstrap 文件列表（按时间段注入不同上下文）
- bootstrap-cache.ts 以 sessionKey 为 key 缓存，session 切换自动失效
这是典型的"把简单的事做对"——大多数框架把"启动加载哪些文件"写死在代码里，OpenClaw 把它做成可配置、可 hook、可缓存的三层结构。
### 6.9 Shell 命令执行与审批
核心问题：Agent 想执行 bash 命令时，命令到底在哪台机器上跑？用户怎么审批"是否允许执行"？
两种执行 Host
同一个 bash 工具有两种执行路径，取决于 Agent 的运行模式：
Host何时使用怎么执行exec-host-nodeopenclaw chat（本地 CLI 模式）Node.js spawn，在 agent 进程内直接执行exec-host-gateway通过 Gateway 运行（TUI, Control UI, Channel）命令经 RPC 送到 Gateway，审批通过后才执行为什么要分：Gateway host 可以跨机器代理执行（src/node-host/ 的 node exec）——Agent 跑在 macOS，shell 命令在远端 Linux 节点上执行。本地 CLI 模式不需要这层间接。
审批路径
两种 host 的审批方式不同，但共享同一套安全基线（evaluateShellAllowlist + detectCommandObfuscation + resolveApprovalAuditCandidatePath）：
- Gateway host：审批走聊天通道——用户在 QQ Bot、飞书、Telegram 里看到"是否允许执行 rm -rf ...？"并点按钮
- Node host：审批走本地终端 prompt（TUI 弹窗确认）
PTY 与脚本预检
- PTY 分支（bash-tools.exec.ts）：需要 TTY 的命令（终端 UI、嵌套 coding agent）走 usePty=true 分支，带 pty-keys.ts, pty-dsr.ts 支持原生键盘事件和光标响应——让 Agent 能真正操控 htop, vim, claude-cli 这样的交互式程序
- Script preflight：执行脚本前做静态检查——检测"脚本内含 shell variable injection"、"JS 文件以 NODE 开头"等常见错误，提前拦截
### 6.10 Cache Trace——全链路可观测性
src/agents/cache-trace.ts 把每次 LLM 调用的上下文变化分 7 个阶段（session loaded → sanitized → limited → prompt before → images → stream context → session after）落盘到 ~/.openclaw/state/cache-trace/，可以精确定位 prompt cache miss 的原因。这是 production agent 才会关心的运维能力——Hermes 靠"冻结快照"兜底，OpenClaw 靠"全链路可追溯"定位。
### 6.11 Subagent Spawn——深度可控的子 Agent 委派
与 Hermes delegate_tool 的"阻止列表 + 默认深度 1"不同，OpenClaw 的子 Agent 机制更"系统化"，被拆成十几个文件：
- **subagent-announce-***：子 Agent 在父会话里的状态广播（announce-delivery, announce-dispatch, announce-queue, announce-idempotency），父 Agent 可以看到子进度
- **subagent-registry-***：子 Agent 生命周期注册表（completion, cleanup, queries, helpers）
- **subagent-capabilities.ts**：子 Agent 可以做什么（哪些工具、能不能访问父的工作区）
- subagent-control.ts：父 agent 对子 agent 的控制权限——subagentControlScope: "children" | "none" 决定父能不能 abort 或 steer 子
- **subagentRole: "orchestrator" | "leaf"**：orchestrator 可以再 spawn 子子 agent，leaf 不可以（防 fork bomb）
- **spawnDepth**：跟踪嵌套深度（0 = main, 1 = sub, 2 = sub-sub），硬上限可配置
**深度可控 vs Hermes 的默认 MAX_DEPTH=1**：OpenClaw 把"递归策略"做成 per-session 的元数据——研究场景允许深度 3，客服场景强制 leaf，不需要改代码。
### 6.12 预算贯穿 Runtime——把稀缺资源显式量化
6.1 开头把 runtime 的三件事概括为**"调度 + 容错 + 预算"**。前两件事已经在 6.2–6.11 展开——预算这件事散落在多处，这里集中讲一次。
OpenClaw 对"预算"的理解不是"token 限额"那么窄，而是把 runtime 里的每一种稀缺资源都显式量化，并配一条超限后的降级路径。这和 Hermes 用单一 IterationBudget 计数器的风格形成鲜明对比：
稀缺资源预算形式超预算后的降级上下文窗口contextTokenBudget（按模型解析，Context Engine assemble/compact/afterTurn 都传入）三级 Compaction 降级（L1 pre-request → L2 timeout-triggered → L3 overflow）单次工具输出MAX_TOOL_RESULT_CONTEXT_SHARE = 0.3（30% 软限）+ HARD_MAX_TOOL_RESULT_CHARS = DEFAULT_MAX_LIVE_TOOL_RESULT_CHARS = 16_000（16K 字符硬限，源码 tool-result-truncation.ts:40）head + tail 截断，最小保留 2000 字符，错误关键词（error, exception, traceback）优先保留 tail启动上下文bootstrap-budget.ts 的 maxChars + totalMaxChars + nearLimitRatio = 0.85按优先级裁剪单文件 + 发 prompt warning 告诉 Agent "有文件被截了"循环迭代次数MAX_RUN_LOOP_ITERATIONS = resolveMaxRunRetryIterations(profileCount)（按 profile 数动态算，不硬编码）抛 FailoverError，交给外层 runWithModelFallback 切模型Overflow 压缩尝试MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3放弃压缩，截断最大的 tool result；仍不行则报错Timeout 压缩尝试MAX_TIMEOUT_COMPACTION_ATTEMPTS（独立计数）放弃压缩尝试，走 auth profile 轮换凭证可用性Cooldown 时间预算（rate_limit: 30s → 1min → 5min 分级递增）轮换到下一个 profile；冷却结束后允许 probe 探针式重试子 Agent 递归spawnDepth + subagentRole: orchestrator/leaf达到深度后强制 leaf（禁止再 spawn）Lane 并发Cron / Subagent / Nested / Default 四车道独立队列超过 lane 并发上限进队列等待，不挤占其他 laneSteer 速率STEER_RATE_LIMIT_MS（父对子发送 steer 消息的节流）超过间隔内的 steer 消息被丢弃为什么这种做法重要：
- 防雪崩：一个持续超窗口的 turn 如果没有压缩次数预算，会无限触发"压缩→压缩不动→再触发"的死循环；Overflow budget = 3 是这条路径的 circuit breaker
- 让降级路径可证明：每种预算都对应一条明确的降级路径，所以 FailoverError 一旦抛出，调用链上任何一层都能判断"我还能做什么补救"，不用靠 LLM 重新思考
- 可观测性的必然结果：因为每种预算都是显式变量（而不是隐式的"让进程自然崩溃"），Cache Trace（6.10）才能把每个阶段的资源消耗落盘，事后能复盘"这次失败是预算耗尽的哪一环"
这是 OpenClaw 风格和 Hermes 风格的分水岭——Hermes 靠单一的 90 迭代上限兜底（到上限就硬停），OpenClaw 把每种资源拆成独立预算并配对应的降级。前者简单可预测但粒度粗，后者精细但实现复杂度更高。
Agent Runtime 预算 5 层防御
OpenClaw 的预算不是单一值，而是**"防爆 + 留余 + 自适应" 的 5 层防御组合** —— 4 种预算 × 4 路超预算决策 × 单工具双层硬限 × Bootstrap 双层 char 预算 × 截断告警自感知。
** 4 种预算类型 **
预算类型控制什么关键常量单位1. Context Token Budget整个 LLM context 窗口大小contextTokenBudget（动态，从模型 contextWindow 解析）token2. Prompt Budget留给 prompt 的部分（context - reserve）promptBudgetBeforeReservetoken3. Reserve Tokens留给 output 的余量DEFAULT_PI_COMPACTION_RESERVE_TOKENS_FLOOR = 20_000token4. Bootstrap Char Budget静态层 8 文件 push 注入预算bootstrapMaxChars=20_000 / bootstrapTotalMaxChars=150_000字符注意 Token 和 Char 是两套预算——Bootstrap 用 Char（push 阶段还没进 LLM，只能按字符估算），其他用 Token。
核心约束公式（源码 preemptive-compaction.ts）
// 1. context 必须正整数
const contextTokenBudget = Math.max(1, Math.floor(params.contextTokenBudget));
// 2. reserve 必须非负
const requestedReserveTokens = Math.max(0, Math.floor(params.reserveTokens));
// 3. 计算最小 prompt budget — 取两者较小值
const minPromptBudget = Math.min(
  MIN_PROMPT_BUDGET_TOKENS,                            // 8K 绝对下限
Math.max(1, Math.floor(contextTokenBudget * 0.5)),   // 50% 上下文
);
// 4. 实际 reserve 被截断 — 不能挤占 minPromptBudget
const effectiveReserveTokens = Math.min(
  requestedReserveTokens,
Math.max(0, contextTokenBudget - minPromptBudget),
);
// 5. 真正给 prompt 的预算
const promptBudgetBeforeReserve = Math.max(1, contextTokenBudget - effectiveReserveTokens);
如果 reserve 想吞太多 → 自动让步给 prompt minPromptBudget**——保证 prompt 至少有 min(8K, 50% context) 可用。
反例：用户配置 reserveTokens=100K 但 contextTokenBudget=20K：
- minPromptBudget = min(8K, 10K) = 8K
- effectiveReserveTokens = min(100K, 20K - 8K) = 12K（被截！）
- promptBudgetBeforeReserve = 20K - 12K = 8K
这个 clamp 防止配置错误导致 prompt 完全没空间。
SAFETY_MARGIN = 1.2（源码 compaction.ts:22）
export const SAFETY_MARGIN = 1.2; // 20% buffer for estimateTokens() inaccuracy
所有 token 估算都乘以 1.2：
const estimated = estimateMessagesTokens(messages) + ...;
return Math.max(0, Math.ceil(estimated * SAFETY_MARGIN));
为什么乘 1.2：
- estimateTokens() 是估算，不是精确 tokenizer（精确的太慢）
- 实际 token 数可能比估算多 5-15%
- 乘 1.2 留 20% 安全边距 = 防止"估算说够，实际溢出"
很多 Agent 框架直接用 estimate 不留 margin，会随机触发 context overflow。OpenClaw 的 1.2 倍安全边际把"估算不准"这个已知风险直接消化掉。
超预算的 4 路决策
检查到 overflow 后不是只有一个动作，而是分情境路由：
let route: PreemptiveCompactionRoute = "fits";
if (overflowTokens > 0) {
  if (toolResultReducibleChars <= 0) {
    route = "compact_only";                     // 没工具结果可裁 → 只能 compact
  } else if (toolResultReducibleChars >= truncateOnlyThresholdChars) {
    route = "truncate_tool_results_only";       // 工具结果够裁 → 只裁工具
  } else {
    route = "compact_then_truncate";            // 都做：先 compact，再裁工具
  }
}
场景工具结果可裁量选择路径为什么装得下—fits不动装不下，没工具结果≤ 0compact_only只能压缩历史装不下，工具结果够多≥ overflow × 1.5 + 512 tokentruncate_tool_results_only工具截断更便宜装不下，工具结果不够之间compact_then_truncate双管齐下设计取舍：优先裁工具结果——工具结果通常可重新获取（再跑一次工具就有了），对话历史压缩了就丢了语义。
单工具结果双层硬限（源码 tool-result-truncation.ts）
const MAX_TOOL_RESULT_CONTEXT_SHARE = 0.3;             // 软限：占 context 不超过 30%
const DEFAULT_MAX_LIVE_TOOL_RESULT_CHARS = 16_000;     // 硬限：单次最多 16K 字符
const HARD_MAX_TOOL_RESULT_CHARS = DEFAULT_MAX_LIVE_TOOL_RESULT_CHARS;
双层防御逻辑：
工具返回 100K 字符
    ↓
软限检查：100K > context * 0.3？是 → 裁
    ↓
硬限检查：100K > 16K？是 → 裁到 16K
    ↓
取两者较严的限制 → 16K
最小保留 + error 关键词 tail 保留：
- 不论怎么裁，至少保留前 2K 字符（保 LLM 知道工具是干啥的）
- error, exception, traceback 关键词在尾部时优先保留尾部（错误信息往往在末尾）
Bootstrap 截断告警注入 LLM（自感知机制）
源码 bootstrap-budget.ts 的 appendBootstrapPromptWarning：
Bootstrap 文件被截断
    ↓
计算 truncation signature（哪些文件被截、各自被截百分比）
    ↓
检查 warning mode（off, once, always）
    ↓
once 模式 + 之前看过这个 signature → 不再警告
once 模式 + 新 signature → 注入警告到 prompt
    ↓
告警内容：
  "[Bootstrap truncation warning]
   Some workspace bootstrap files were truncated before injection.
   - AGENTS.md: 25000 raw -> 20000 injected (~20% removed; max/file).
   - SOUL.md:   18000 raw -> 18000 injected (...; max/total)."
    ↓
LLM 看到警告 → 知道"我看到的 context 可能不全，必要时主动 read_file"
告警是给 LLM 看的，不只是给开发者看——LLM 知道自己被截断后，会主动用工具补读完整文件，而不是基于不完整信息瞎猜。
### 6.13 两个"反直觉"的设计选择
- Agent 是单进程串行的——同一个 Agent 同一时刻只跑一个 turn，多用户并发通过多 Agent 路由（不同 agent_id）实现。原因：workspace, memory, sessions 是状态化共享资源，并发会互相污染。
- 核心功能通过插件 API 注入——Memory Search 是插件注册的 Tool + Capability，Compaction 通过 hook 暴露扩展点。Runtime 本质是编排壳，调度、容错、预算是它的，具体能力由插件填充。
### 6.14 模型选择与降级
支持 9 种 LLM API 协议（OpenAI, Anthropic, Gemini, Bedrock, Ollama, GitHub Copilot, Azure 等），降级顺序是 Auth Profile 优先于 Model Fallback——同模型的所有 profile 都耗尽后，才切换到 model.fallbacks[] 中的下一个模型。冷却中的 profile 还允许探针式重试（shouldAllowCooldownProbeForReason），成功则回收继续使用。
### 6.15 工作区文件 — Agent 的人格与记忆
OpenClaw 为每个 Agent 维护两个目录（目录结构详见 §4.2）：**workspace/**（Agent 的"大脑"——人格、记忆、文件等内容）和 **agents/{id}/**（Agent 的"档案"——会话转录和运行时元数据）。§4.2 未提及的 AGENTS.md 也在 workspace 下，用于存放项目级指令。
memory/ 和 sessions/ 容易混淆：sessions/ 是对话的完整录像（JSONL 格式，每轮问答自动记录），memory/ 是从对话中提炼的笔记（Markdown 格式，由 hook 或 Dreaming 有选择地写入）。前者用于维护对话上下文，后者注入 System Prompt 影响 Agent 的长期行为。两者在用户维度的隔离也不同：sessions/按用户隔离（每个 SessionKey 对应独立的 .jsonl 文件），memory/所有用户共享（同一 Agent 下写入同一目录，文件名不含用户标识）。
默认 Agent（main）的工作区在 ~/.openclaw/workspace/，非默认 Agent 在 ~/.openclaw/workspace-{id}/（源码 resolveAgentWorkspaceDir()）。工作区下的 Markdown 文件构成 Agent 的个性化上下文：
各文件的作用已在 §2 和 §4.2 介绍，这里补充典型内容示例：
文件示例内容SOUL.md"有观点、先尝试再提问、对外部操作谨慎"USER.md姓名、时区、关注的项目、编码习惯MEMORY.md由 session-memory hook 自动维护的跨会话关键结论AGENTS.md编码风格、工具使用规则、特定领域知识这些文件在每次 Agent 启动时被注入到 System Prompt 中，使 Agent 具备跨会话的记忆和个性化的交互风格。用户可以直接编辑这些 Markdown 文件来调整 Agent 行为，无需修改代码或配置。
工作区文件是记忆系统的静态层。OpenClaw 还提供了完整的向量记忆引擎和 Dreaming 后台整合机制，详见第 7 章。
Bootstrap 截断策略与子 Agent allowlist
工作区 8 个 Markdown 文件并不会无条件全量注入 —— OpenClaw 有两道精细的过滤层：
第一道：截断策略 — head 70% + tail 20%（不是前缀截断）
源码 bootstrap.ts —— 当某个文件超过 bootstrapMaxChars=20_000 字符时：
原始文件                    截断后
┌────────┐                ┌────────┐
│ HEAD   │ 70% 保留 ──→  │ HEAD   │
│        │                │ ...    │
│ MIDDLE │ ✗ 砍中间       │ TAIL   │
│        │                └────────┘
│ TAIL   │ 20% 保留
└────────┘
截断策略是头尾保留，砍中间——不是简单的前缀或后缀截断：
- **保 head 70%**：通常是文档的"使命宣言、核心约定、全局原则" —— 不能丢
- **保 tail 20%**：通常是"近期更新、最新约定、例外说明" —— 也很重要
- 砍 middle：通常是"中间累积的事例、历史细节" —— 损失最小
对比常见错误印象：
- ❌ "截断 = 前缀截断" → 砍掉最近的（错）
- ❌ "截断 = 后缀截断" → 砍掉最远的（也错）
- ✅ OpenClaw = 头尾保留，砍中间 —— 兼顾"全局约定"和"最新更新"
第二道：子 Agent allowlist（5 文件保留）
源码 src/agents/workspace.ts：
const MINIMAL_BOOTSTRAP_ALLOWLIST = new Set([
  DEFAULT_AGENTS_FILENAME,    // AGENTS.md       ✅
  DEFAULT_TOOLS_FILENAME,     // TOOLS.md        ✅
  DEFAULT_SOUL_FILENAME,      // SOUL.md         ✅
  DEFAULT_IDENTITY_FILENAME,  // IDENTITY.md     ✅
  DEFAULT_USER_FILENAME,      // USER.md         ✅
]);
// filterBootstrapFilesForSession:
if (isSubagentSessionKey(sessionKey) || isCronSessionKey(sessionKey)) {
  return files.filter((file) => MINIMAL_BOOTSTRAP_ALLOWLIST.has(file.name));
}
子 Agent, Cron session 只注入这 5 个文件，其他 workspace 文件被剥离：
文件注入主 Agent注入子 Agent/Cron为什么AGENTS.md✅✅项目说明必须TOOLS.md✅✅工具指南必须SOUL.md✅✅人设必须保持一致USER.md✅✅用户画像必须一致IDENTITY.md✅✅Agent 自我认同统一HEARTBEAT.md✅❌ 剥离心跳任务子 Agent 不需要BOOTSTRAP.md✅❌ 剥离仅新 workspace 才注入MEMORY.md✅❌ 剥离避免污染子任务的独立上下文设计哲学：保留人格连续性，剥离状态性数据
子 Agent 必须和主 Agent "同一个人格"（否则用户感受会崩——感觉是另一个陌生助手），但跑独立子任务不携带历史包袱（避免主线对话污染子任务判断）：
类别文件为什么保留/剥离人格连续性SOUL / USER / IDENTITY子 Agent 不能变路人协作上下文AGENTS / TOOLS项目和工具说明必须状态性数据HEARTBEAT / BOOTSTRAP / MEMORY子 Agent 是独立子任务，不要污染
### 7. 记忆系统 — 从静态文件到智能召回
记忆系统是 Agent 执行引擎（第 6 章）的关键上游——Agent Engine 在每次 buildPrompt() 时，会从记忆系统获取相关上下文注入 System Prompt，使 Agent 的回复具备历史感知。两者的关系是：Agent Engine 负责"思考和行动"，记忆系统负责"记住和回忆"。
隔离粒度：记忆按 Agent 维度隔离，同一个 Agent 下所有会话（包括不同用户、不同 Channel）共享同一份记忆。这是因为 OpenClaw 定位为个人 AI Agent——默认场景是一个人使用，Agent 的记忆就是"这个 Agent 的全部记忆"。多用户场景下，通过多 Agent 路由绑定（第 4.3 章）为不同用户分配独立 Agent，即可实现记忆隔离：
# 默认：所有用户共享 main Agent 的记忆
~/.openclaw/workspace/memory/
 2026-04-09-api-design.md        ← 来自用户 A
 2026-04-09-qqbot-debug.md       ← 来自用户 B
# 多 Agent 绑定后：每个用户独立 workspace 和记忆
~/.openclaw/workspace/memory/              ← main Agent（默认）
~/.openclaw/workspace-support/memory/      ← support Agent（用户 A 专属）
~/.openclaw/workspace-dev/memory/          ← dev Agent（用户 B 专属）
除了 MEMORY.md 这样的静态工作区文件，OpenClaw 还提供了完整的向量记忆引擎，实现了从记忆捕获、索引、搜索到主动召回的完整链路：
引擎后端存储方案特点memory-core（默认）builtin（内置）每 Agent 一个 SQLite（~/.openclaw/memory/<id>.sqlite）：FTS5 全文 + 可选 sqlite-vec 向量；CJK trigram 分词零依赖，开箱即用memory-coreqmd（可选）外挂 QMD sidecar（Bun + node-llama-cpp 独立二进制，底层也是 SQLite）额外提供 reranking、查询扩展、索引工作区外路径和会话转录；不可用时自动降级到 builtinmemory-lancedb（第三方插件）—LanceDB 嵌入式向量数据库向量检索性能更强，支持 auto-capture + auto-recall 生命周期钩子QMD 不是 OpenClaw 代码库的一部分，而是 Shopify CTO 开源的一个通用本地搜索工具。OpenClaw 通过子进程调用 qmd update, qmd embed, qmd query 来驱动它，并管理其生命周期（boot 时 + 每 5 分钟周期性 embed，带 15 分钟分布式锁防并发）。
三者都支持混合搜索（BM25 + 向量相似度），通过 plugins.slots.memory 和 memory.backend 两层配置切换：
{
  "plugins": { "slots": { "memory": "memory-core" } },
  "memory": { "backend": "qmd" }
}
plugins.slots.memory 设为 "none" 可完全禁用记忆插件。默认不配置时使用 memory-core + builtin backend。
### 7.1 记忆捕获的三条路径
路径触发时机实现Session Memory Hook用户执行 /new 或 /reset读取最近 15 条消息 → LLM 生成摘要 → 写入 memory/YYYY-MM-DD-slug.mdMemory FlushCompaction（上下文压缩）前当 totalTokens >= contextWindow - reserve - softThreshold 时自动触发，将即将被压缩掉的上下文保存到记忆文件Auto Capture对话中实时检测基于正则规则匹配关键信息（偏好、联系方式、决策等），自动捕获并分类Auto Capture 的触发规则（源码 extensions/memory-lancedb/index.ts 的 MEMORY_TRIGGERS）：
"remember" / "记住" / "记下"     → 用户明确要求记忆
"prefer" / "like" / "hate"       → 情感偏好
"+8613800000xxxx"                → 电话号码（10位以上数字）
"user@example.com"               → 邮箱
"my X is" / "is my"              → 所有权声明
"I like / prefer / hate / want"  → 偏好动词
"always / never / important"     → 强调词
"我喜欢 / 我偏好 / 决定 / 重要"  → 中文触发词
触发后还有安全过滤（shouldCapture）：跳过 prompt injection 载荷、跳过 Agent 自己生成的内容（含 markdown/emoji 特征的）、长度限制（10 字符 < 内容 < maxChars）。
捕获后自动分类（detectCategory）为：preference | decision | entity | fact | other
### 7.2 混合搜索算法
检索时采用 BM25 + 向量相似度 的混合搜索，经过三层后处理：
查询 ──→ BM25 关键词搜索（权重 0.3）
     └─→ 向量相似度搜索（权重 0.7）
                ↓
          加权融合 mergeHybridResults()
                ↓
          时间衰减：score × e^(-λ × age)
          λ = ln(2) / halfLifeDays，默认半衰期 30 天
          "常青"文件（MEMORY.md 等）豁免衰减
                ↓
          MMR 多样性重排：避免返回高度相似的结果
          MMR = λ × relevance − (1−λ) × max_similarity
                ↓
          Top-K 结果返回
### 7.3 Dreaming 概览 — 后台记忆整合系统
OpenClaw 最新引入的 Dreaming 机制在上述基础设施之上，增加了 Agent 在后台自主整理和晋升记忆的能力。核心设计借鉴人类睡眠的记忆整合过程，分为三个协作阶段（源码 extensions/memory-core/src/dreaming-phases.ts）。
⚠️ 重要前提：Dreaming 默认 opt-in 关闭（dreaming.enabled: false）——这是关键工程取舍：
- 成本：每次 sweep 跑 LLM 评分 + 写入
- 副作用：错误晋升会污染所有后续对话（MEMORY.md 每轮都会注入 LLM）
- 复杂度：cron + timezone + 阶段调度
启用后 cron 默认每天 03:00 触发；也可手动跑 openclaw memory promote --apply。
三阶段职责与本质：
阶段做什么是否写 MEMORY.md本质Light Sleep读取近期短期召回、每日记忆和脱敏会话，去重后整理候选条目❌ 仅整理候选物料准备REM Sleep提取反复主题 + 选候选"潜在真理" + 反馈 Deep 排名权重❌ 仅提取信号抽象思考Deep Sleep加权评分 + 阈值门控 + 回源验证后写入持久记忆✅ 唯一写入路径固化决策Dream Diary（DREAMS.md）：每次 Dreaming 运行后，系统调用一个后台 Subagent 生成诗意的"梦境日记"追加写入，文风被定义为：
"You are a curious, gentle, slightly whimsical mind reflecting on the day. Write like a poet who happens to be a programmer — sensory, warm, occasionally funny. Mix the technical and the tender: code and constellations, APIs and afternoon light."
Active Memory Recall 插件（extensions/active-memory/）：独立插件，在每次对话前运行一个阻塞式记忆子 Agent，15 秒超时：
- 读取当前对话上下文（支持 message, recent, full 三种查询模式）
- 搜索记忆库找到相关记忆
- 生成 ≤220 字符的摘要，以隐藏 Prompt Prefix 形式注入（召回结果作为 hidden prefix 注入用户消息前方，对用户不可见但 LLM 可读）
- 结果缓存 15 秒，避免重复查询
支持 6 种 prompt 风格：balanced | strict | contextual | recall-heavy | precision-heavy | preference-only
### 7.4 Dreaming 算法深度解析
源码extensions/memory-core/src/dreaming.ts (788 行) + dreaming-phases.ts (1741 行) + short-term-promotion.ts (1957 行) + docs/concepts/dreaming.md —— 共 14 文件、4486 行核心代码。
Deep 阶段：6 信号加权评分
源码 DEFAULT_PROMOTION_WEIGHTS（short-term-promotion.ts）：
score = 0.24 × frequency        命中次数
      + 0.30 × relevance        召回质量（权重最大）
      + 0.15 × diversity        query/天 多样性
      + 0.15 × recency          时间衰减新鲜度（半衰期 14 天）
      + 0.10 × consolidation    多天复现强度
      + 0.06 × conceptual       概念标签密度
      + Light boost (≤ +0.06)   浅睡命中加成
      + REM   boost (≤ +0.09)   REM 命中加成
Deep 阶段：3 重门禁（必须全部通过才晋升）
DEFAULT_PROMOTION_MIN_SCORE          = 0.75   总分 ≥ 0.75
DEFAULT_PROMOTION_MIN_RECALL_COUNT   = 3      命中 ≥ 3 次
DEFAULT_PROMOTION_MIN_UNIQUE_QUERIES = 2      不同 query ≥ 2 个
为什么 3 重门禁缺一不可（防过拟合）：
- 单纯总分高 → 可能一次召回特别准
- 单纯命中多 → 可能同一 query 反复命中
- 单纯 query 多 → 可能召回质量差
- 三个都通过 = 真正"重要" ✅
防 stale 设计：晋升前重新读 daily 文件 hydrate —— 你删了 daily 笔记 = 自动撤回候选。
REM 阶段深度解析（最容易被忽略的"抽象思考层"）
源码 dreaming-phases.ts，REM 的 3 个核心工作：
- buildRemReflections —— 统计 concept tags 频率，提取跨条记忆的高强度主题
- selectRemCandidateTruths —— 选"潜在真理"（4 信号置信度，REM 最精华）
- recordDreamingPhaseSignals —— 写 phase-signals.json 给 Deep 加成
REM 置信度公式（calculateCandidateTruthConfidence）：
confidence = averageScore   × 0.45   // 召回质量（权重最大！）
           + recallStrength × 0.25   // log1p(recallCount) / log1p(6) 次线性饱和
           + consolidation  × 0.20   // min(1, recallDays.length / 3) 3 天饱和
           + conceptual     × 0.10   // min(1, conceptTags.length / 6) 6 标签饱和
// 过滤：confidence ≥ 0.45（远宽松于 Deep 的 0.75）
// 去重：相似度 0.88 阈值
// 排序：confidence desc + snippet asc
// 截断：top 3
REM vs Deep 公式对比（架构师必懂）
信号REM truth 置信度Deep 晋升评分Relevance0.450.30Frequency0.250.24Consolidation0.200.10Conceptual0.100.06Diversity无0.15Recency无0.15阈值≥ 0.45（宽松）≥ 0.75（严格）深刻差异：
- REM 不看 diversity 和 recency —— 只关心"内在质量"
- REM 更偏重相关性（0.45 vs 0.30）—— 找的是"内容本身过硬"
- REM 更偏重 consolidation（0.20 vs 0.10）—— 跨天出现 = 不是偶然
- Deep 有 diversity + recency —— 因为 Deep 是"永久固化"，不能让陈旧/单一视角永久驻场
结论：REM 找"稳固事实"，Deep 找"值得晋升到每轮可见的稳固事实" —— 两个目标不同。
为什么 REM boost (+0.09) > Light boost (+0.06)
const PHASE_SIGNAL_LIGHT_BOOST_MAX = 0.06;
const PHASE_SIGNAL_REM_BOOST_MAX   = 0.09;
- Light = "这条东西被看到过" —— 弱信号
- REM = "这条东西被 Agent 主动识别为可能是真理" —— 强信号
人脑类比：
- Light = 睡觉前快速过一遍今天的事（还没思考）
- REM = 做梦时大脑抽象推演（"哦，这几件事有共同模式"）
- 被 REM 选中 = 下次 Deep 固化时优先考虑
控制入口
# Slash 命令
/dreaming status / on / off / help
# CLI（即使没启 cron 也能手动）
openclaw memory promote               # 预览候选
openclaw memory promote --apply       # 应用晋升
openclaw memory promote-explain "xxx" # 解释为什么会/不会晋升
openclaw memory status --deep
### 7.5 Memory 双层流转 — 每天 memory ↔ 全局 MEMORY.md
Dreaming 只是晋升机制，背后真正的记忆架构是双层存储：memory/YYYY-MM-DD.md（每天召回层）和 MEMORY.md（全局静态层），两者通过 Dreaming 晋升串联。
文件物理结构
<workspace>/
├─ MEMORY.md                          ← 🎯 全局（静态层 push，每轮注入 LLM）
├─ DREAMS.md                          ← 梦境日记（人类阅读）
└─ memory/                            ← 召回层（pull）
   ├─ 2026-05-07.md                   ← 📅 每天文件（daily memory）
   ├─ 2026-05-07-vendor-pitch.md      ← /new 触发的会话归档
   ├─ .dreams/                        ← 🔧 Dreaming 内部状态
   │  ├─ short-term-recall.json       （短期召回追踪 + 6 维度）
   │  ├─ phase-signals.json           （Light/REM 信号）
   │  └─ short-term-promotion.lock    （并发锁）
   └─ dreaming/                       ← 🌙 Dreaming 阶段报告（人类阅读）
      ├─ light/YYYY-MM-DD.md
      ├─ deep/YYYY-MM-DD.md
      └─ rem/YYYY-MM-DD.md
** 4 路径流入 + 1 路径晋升 **
        【对话流（运行中）】
              │
   ┌──────────┼──────────┬──────────┐
   ▼          ▼          ▼          ▼
路径1       路径2      路径3      路径4
/new      Compaction  LLM 主动   Dreaming 摄入
触发      Pre-Flush   Write     脱敏 session
(用户)    (自动)      (LLM 自决)  (Dreaming)
   │          │          │          │
   └──────────┴──────┬───┴──────────┘
                    ▼
          【memory/YYYY-MM-DD.md】（召回层）
                    │
                    ▼ 路径 5：Dreaming 晋升 ⭐⭐⭐
                    │   Light → REM → Deep（算法详见 §7.4）
                    ▼
          【MEMORY.md】（静态层）
                    │
                    ▼
              每轮注入 LLM context
每天 vs 全局 — 核心差异
维度每天 memory/YYYY-MM-DD.md全局 MEMORY.md属于哪层召回层（pull）静态层（push）是否注入 LLM context❌ 不自动注入✅ 每轮注入进入 LLM 的方式LLM 调 memory_search / memory_get 才能看Bootstrap 机制每轮 push写入触发多种来源（compaction flush, /new, LLM 主动）只有 Dreaming Deep 阶段类比海马体的短期记忆皮层的长期记忆所以双层架构的本质：召回层是"原始日记"（全量保存但不默认注入），静态层是"读书笔记"（精炼内容每轮可见），Dreaming 是两层之间的"晋升管道"。用户显式启用 Dreaming 前，每天 memory 永远不会自动晋升——双层之间的管道默认是关闭的。
### 8. 安全机制 — 多层纵深防御
### 8.1 Exec Approval 交互流程
当 Agent 尝试执行危险命令时，系统会要求人工审批：
审批决策类型：
决策含义allow-once仅允许本次执行allow-always将命令模式加入白名单，后续自动通过deny拒绝执行混淆检测规则（部分）：
规则 ID检测模式curl-pipe-shellcurl/wget ... | sh/bashbase64-pipe-execbase64 -d | basheval-decodeeval ... base64/decodepipe-to-shell... | sh/bash/zshpython-exec-encodedpython -c ... exec/eval
### 9. 配置系统 — 单文件掌控全局
所有行为由 ~/.openclaw/openclaw.json 驱动，支持运行时热重载。
配置变更触发热重载：
### 10. Hooks & Skills
### 10.1 Hooks（事件钩子）
Hook 加载优先级：bundled → managed → workspace
示例：session-memory hook
当用户发送 /new 或 /reset 命令开启新会话时，session-memory hook 自动将当前会话的关键上下文保存到 ~/.openclaw/memory/ 目录，供后续会话参考：
用户: /new
  → 触发 command:new 事件
  → session-memory hook 执行：
    1. 提取当前会话的摘要（用户偏好、关键结论等）
    2. 写入 <workspace>/memory/YYYY-MM-DD-slug.md
    3. 新会话启动时，Agent 可引用历史记忆
示例：command-logger hook
监听所有斜杠命令事件（/new, /reset, /stop, /help 等），记录为 JSONL 审计日志，用于调试和安全回溯：
{"event":"command:new","sessionKey":"agent:main:qqbot:...","timestamp":"2026-04-01T09:00:00Z"}
{"event":"command:reset","sessionKey":"agent:main:main","timestamp":"2026-04-01T09:05:00Z"}
{"event":"command:stop","sessionKey":"agent:main:telegram:...","timestamp":"2026-04-01T09:10:00Z"}
### 10.2 Skills（技能系统）
技能注入策略：
- 全格式：名称 + 描述 + 路径（默认）
- 紧凑格式：仅名称 + 路径（超预算时自动降级）
- 预算限制：maxSkillsInPrompt=150，maxSkillsPromptChars=30000
### 11. 出站消息管道
Agent 生成回复后，需要经过标准化、分块、适配三个阶段才能送达用户。这个管道对所有 Channel 统一，具体的发送方法由各 Channel 的 Outbound 适配器实现。
关键机制说明：
- ReplyPayload：Agent 输出的标准化负载，包含 text（文本）、mediaUrl（媒体链接）、interactive（交互元素，如投票）、audioAsVoice（音频作为语音气泡发送）等字段
- 智能分块：长文本按 Channel 的字符限制自动分块（如 QQ Bot 5000 字符、Telegram 4000 字符），优先在换行处切分以保持 Markdown 格式完整
- Outbound 适配器：每个 Channel 实现自己的 sendText 和 sendMedia，处理平台特定的 API 调用、速率限制、格式转换等差异
- message:sent Hook：消息发送后触发钩子，用于审计日志、镜像投递等扩展场景
### 12. 媒体处理管道
资源与安全限制：
- 单文件大小：媒体 store 默认 5MB（MEDIA_MAX_BYTES，src/media/store.ts:21）；同时也是沙箱暂存上限（STAGED_MEDIA_MAX_BYTES）。chat.send RPC 的解析上限更宽（默认 20MB），所以 5–20MB 的非图片文件会在 RPC 通过、沙箱暂存阶段被拒，源码里 chat.ts:920-934 显式提前拦截以避免不必要的 5xx 重试。
- 路径遍历防护（拒绝 ../, \0、符号链接）
- SSRF 防护（私有网络检测）
- TTL 过期清理（默认 2 分钟）
### 13. 实战案例：QQ Bot 插件架构
前面 12 章介绍了 OpenClaw 的框架全貌。接下来，我们以 QQ Bot 插件为例，展示一个完整的 Channel Plugin 如何从零开始、利用上述架构机制实现端到端的消息通道接入，并最终合入 OpenClaw 主仓成为内置扩展。
### 13.1 插件在架构中的位置
### 13.2 消息处理全链路
从 QQ 用户发送消息到收到 AI 回复的完整链路：
### 13.3 模块架构
QQ Bot 插件的模块分为六层：
### 13.4 三种消息场景
QQ Bot 支持三种消息场景，对应不同的 API 和路由策略：
多机器人账号与多 Agent 的关系
QQ Bot 插件支持在 channels.qqbot.accounts 下配置多个机器人账号（每个账号对应一个 appId + clientSecret），每个账号独立建立 WebSocket 连接。这与 OpenClaw 的多 Agent 机制是两个独立维度，可以灵活组合：
维度配置位置控制什么多账号channels.qqbot.accounts多个机器人同时在线，各自独立的 appId/凭证/WebSocket 连接多 Agentagents.bindings不同消息路由到不同 Agent，各自独立的人格/记忆/工作区会话隔离session.dmScope同一账号不同用户的会话隔离粒度（per-channel-peer 或 per-account-channel-peer）典型场景：两个机器人 + 两个 Agent——default 机器人的消息走 main Agent（通用助手），bot2 机器人的消息走 vip Agent（VIP 服务，独立记忆）。也可以多个机器人共享同一个 Agent，或一个机器人按用户绑定不同 Agent。
### 13.5 WebSocket 网关生命周期
QQ Bot 的 WebSocket 连接管理是插件比较复杂的部分：
### 13.6 独立插件 vs Bundled Extension
QQ Bot 同时存在两个版本，分别服务不同场景：
维度独立插件Bundled Extension包名@tencent-connect/openclaw-qqbot随 openclaw 主仓安装方式openclaw plugins install内置，无需安装注册方式api.registerChannel()defineChannelPluginEntry()流式消息✅ C2C 打字机效果✅ 最新已支持流式配置消息去抖✅❌大文件上传✅ 100MB 分片❌免登录热更新✅ bash/npx 一键升级❌ 跟随主仓版本迭代节奏独立快速发布跟随主仓版本这种「独立插件 + 框架内置」双轨模式在同类项目中并不常见。大多数 AI Agent 框架采用纯平台托管或纯独立包模式。双轨模式兼顾了独立插件的快速迭代和 bundled 分发的开箱即用。
演进模式：独立插件功能先行、快速迭代 → Bundled 版本精简整合、框架对齐。
### 13.7 里程碑：从独立项目到框架内置
2026 年 3 月 31 日，经过社区贡献者与团队的共同努力，QQ Bot 插件正式合入 OpenClaw 主仓，成为框架内置的 Channel Extension。这背后离不开 QQ 开放平台持续开放 Bot API 能力——从消息收发、富媒体上传到群组管理，平台侧的能力开放为插件的功能演进提供了基础。
合入后的变化：
- 开箱即用：用户安装 OpenClaw 后直接可用 QQ Bot 通道，无需额外安装插件
- 零门槛接入：从早期需要手动配置 WebSocket, Token、权限等多个步骤，到提供一键安装脚本和 CLI onboarding 向导，将 QQ Bot 的接入成本降为零
- 版本同步：插件与框架同步发布，兼容性有保障
- 双轨并行：独立插件 @tencent-connect/openclaw-qqbot 继续作为功能先行版迭代，成熟的特性回流到 bundled 版本
这个过程也验证了 OpenClaw「万物皆插件」架构的开放性——任何开发者都可以从一个独立插件起步，逐步融入框架生态。
### 13.8 QQ Bot Channel 与 QQ CLI——"在 QQ 里用 AI" vs "让 AI 用 QQ"
为什么需要 Channel + CLI 双形态
IM 平台接入 AI Agent 有两个方向，解决的是不同问题：
方向谁发起目的**"在 IM 里用 AI"**（Channel）用户在 IM 中 @BotAgent 作为 IM 内的对话助手**"让 AI 用 IM"**（CLI）外部 AI Agent / 脚本 / CIAgent 把 IM 当可操作资源（发消息/查文档/管日程）两者是覆盖范围的互补：Channel 封装高频操作（精而不全），CLI 覆盖全平台 API（全而不精）。从飞书生态的实践中可以提取 4 条设计原则：
- Channel 负责消息通道 + 高频工具，CLI 负责长尾能力 + 用户身份操作——Agent 按需选择走哪条路径
- CLI 面向所有 AI Agent，不锁定单一框架——飞书 CLI 支持 Claude Code, Cursor, Codex, Copilot, Windsurf 等 6 种 Agent
- 认证模型分层——Channel 消息层走 Bot 身份（7×24），工具层 per-call 切换身份（渐进式授权）；CLI 全局配置身份 + 严格模式锁死（安全默认）
- 仓库独立、授权独立、演进独立——Channel 和 CLI 各自存储 token（物理隔离），各自发版迭代
飞书的参考实现（源码调研）
基于 larksuite/openclaw-lark（Channel，TypeScript）和 larksuite/cli（CLI，Go）的 GitHub 源码。
                OpenClaw Agent
                     │
        ┌────────────┼────────────┐
        ▼                         ▼
飞书 Channel Plugin           飞书 CLI Skills
(openclaw-lark)              (larksuite/cli)
38 个 AI 工具                24 个 Skills
     │                            │
     │ 负责：                      │ 负责：
     │ ① 消息收发（入站/出站）     │ ① Channel 没覆盖的业务域
     │ ② 流式卡片回复             │   （邮箱/审批/考勤/OKR/
     │ ③ 交互式确认按钮           │    妙记/幻灯片/白板...）
     │ ④ 群管理/访问控制          │ ② 以用户身份代操作
     │ ⑤ 常用文档/表格/日历操作   │   （Channel 只能 Bot 身份）
     │                            │ ③ 全量 API 覆盖
     │                            │   （Channel 只有精选）
     ▼                            ▼
飞书 IM 内                    Agent 内部工具调用
（用户在飞书里和 Agent 对话）    （Agent 操作飞书资源）
配合模式：Channel 能力范围内的请求直接内嵌工具完成（延迟最低）；超出覆盖的请求（邮件/审批/OKR 等）Agent 调 CLI 的对应 Skill 完成；需要用户身份的请求（查私人日历/代发邮件）走 CLI 的 OAuth 认证。
身份切换的精华：Channel 的身份切换是"工具粒度"的——消息层永远是 Bot 身份，但工具层可以 per-call 选择 { as: "user" | "tenant" }。同一个对话中，Agent 收消息用 Bot 身份，帮用户查私人日历时自动切到 User 身份（渐进式授权，auto-auth.ts 自动触发 OAuth 卡片）。CLI 则是全局配置身份（--identity bot-only | user-default），默认 bot-only（安全默认值）。
QQ 生态的映射：qqbot + qqcli
维度qqbot（Channel Plugin）qqcli（对标飞书 CLI）方向在 QQ 里用 AI——用户 @Bot → Agent 响应让 AI 用 QQ——外部 Agent 操作 QQ宿主OpenClaw Agent（Gateway 内）任何 AI Agent（Claude Code / Cursor / Codex / OpenClaw）能力范围消息收发 + 群管理 + 审批 + 提醒 + 语音（精而不全）QQ 全量 API 覆盖（全而不精）认证AppID + ClientSecret（Bot 身份）可扩展：Bot + OAuth 用户身份用户在 QQ 里                    外部 AI Agent（Cursor, Claude Code, OpenClaw）
     │                                    │
     │ @Bot 触发                           │ Skills / CLI 调用
     ▼                                    ▼
┌────────────┐                    ┌────────────┐
│  qqbot     │                    │  qqcli     │
│  (Channel) │                    │  (CLI)     │
│ "在QQ里    │                    │ "让AI      │
│  用AI"     │                    │  用QQ"     │
└─────┬──────┘                    └──────┬─────┘
      │                                  │
      └──────────┐          ┌────────────┘
                 ▼          ▼
            ┌─────────────────┐
            │   QQ Open API   │
            │  (统一底层能力)   │
            └─────────────────┘
两种融合形态（不互斥，可叠加）
**形态 A：qqcli 工具内嵌——"一个 turn 内闭环"**（demo）
用户在 QQ 里跟 Agent 说"帮我发条 QZone，配上昨天的九宫格"→ qqbot Channel 接收 → Agent 推理 → 调 qqcli 的 qq_post_qzone 工具（内嵌 MCP），自动选图、排版、发布 → 通过 qqbot 回复"已发布，9 张图"。Agent 在同一个推理 turn 内闭环，用户感知是"和一个能帮我发 QZone 的 Bot 对话"。