# 为Agent重新设计云：Vibe Coding平台背后的基础设施

- **来源：** 微信公众号（腾讯云开发者）
- **作者：** 未标注
- **日期：** 2026-06-03
- **原文链接：** https://mp.weixin.qq.com/s/xML30_LcVYXTNUfmQK2syA

---

### 01 Vibe Coding 平台正在重塑软件的生产方式

在过去 18 个月里，有一类全新的产品形态从边缘走向主流：Vibe Coding 平台。

用户不再写代码、不再配置环境，甚至不再面对 IDE，只需要在一个对话框里说一句话，比如「帮我做一个待办清单」、「做一个能让朋友给我留言的网站」，几分钟后就能拿到一个可访问、有数据库、能登录的在线应用。

这个赛道在过去半年完成了从「玩具」到「严肃生意」的跃迁，几个标志性数据：

- Lovable：ARR 在 2026 年 2 月突破 4 亿美元，仅 146 人团队、单月新增 1 亿美元收入；估值从 2025 年夏天的 18 亿美元一路涨到 66 亿美元，被外媒称为「欧洲历史上增速最快的初创公司」；
- Vercel v0：作为 Vercel AI Cloud 的核心产品线，推动 Vercel 整体 ARR 达到 3.4 亿美元、YoY 增长 84%，并在 2026 年 3 月完成 3 亿美元 F 轮、估值 93 亿美元；CEO Guillermo Rauch 已经开始公开释放 IPO 信号；
- Replit：靠 Replit Agent 把营收从 1000 万美元做到 1 亿美元只用了 9 个月，2026 年 3 月完成 4 亿美元 D 轮、估值 90 亿美元，目标年底冲 10 亿 ARR；
- Loopit：国内由百川智能联创团队孵化的 AI 原生应用，主打「自然语言生成可交互小游戏 / 互动内容」，2026 年 4 月完成 5000 万美元新一轮融资，由 Garena 领投、累计融资约 1 亿美元，产品因被马斯克转发一度登顶榜单；

如果说 Cursor、Claude Code、CodeBuddy 这类 IDE 内的 Coding Agent 改变的是专业开发者的工作流，那么 Vibe Coding 平台正在做的是更激进的事情：把「拥有一个软件」这件事的门槛拉到普通人也能触达。

这背后是一场关于软件生产关系的重排，软件的供给方第一次有可能在数量级上超过职业开发者。而每一个 Vibe Coding 平台的背后，都需要一套全新的基础设施来支撑这种新生产方式。

### 02 当前 Vibe Coding 平台的工程难题：LLM 写代码已经不是瓶颈

Vibe Coding 平台表面是 AI 产品，底层其实是 Agent Runtime、云沙箱、应用后端、多租户隔离和按量计费的组合工程。对开发者来说，真正的挑战不是让模型写出代码，而是如何把这些工程能力稳定、安全、低成本地组织成一套可复用的基础设施底座。

从产品形态看，Vibe Coding 平台只是「自然语言进、应用出」的一个黑盒：自然语言 → Vibe Coding 平台 → 应用

但从工程视角看，它是几种复杂系统的叠加体：Agent Loop × 多租户云沙箱 × 完整的应用后端 × 严格的权限与计费体系

任何一个想落地 Vibe Coding 平台的团队，都必然需要解决下面的问题：

**2.1 Agent 怎么运行，并持续工作？**

用户「做一个 Todo 应用」的提示词，背后不是一次简单的模型调用，而是一个持续运转的 Agent Loop：组装上下文、调用 LLM、运行 Bash、读写文件、回填上下文、继续推理。这个循环可能要跑几十轮，跨越几分钟到几十分钟。

真正难的地方在于：它既要有地方执行 LLM 生成的不可信代码，又要能记录完整会话事件，还要能在模型出错、工具失败、容器崩溃或用户隔天回来时继续工作。换句话说，Agent Runtime、隔离沙箱和会话持久化，本质上是同一个问题的三个侧面：如何让 Agent 安全、稳定、可恢复地持续运行。

**2.2 生成出来的应用怎么真的「能用」？**

用户要的不是一个静态页面，而是一个有数据库、能登录、能存文件、能被其他人访问的真应用。这意味着平台必须在用户毫无感知的情况下，为他/她准备好一整套后端，包括数据库、身份认证、对象存储、API 网关、域名、HTTPS 证书。

**2.3 怎么把这些都做成多租户？**

平台上同时有成千上万个用户在生成应用，每个应用都是一个独立的「项目」：数据要隔离、计算要隔离、计费要隔离、出问题不能互相影响。一个用户写了死循环把 CPU 打满，不能拖慢其它用户的编程任务；一个用户的应用突然爆火产生大量请求，也不应该把其它用户的在线应用也打挂。

**2.4 在保证以上 3 点的同时，平台的运营成本如何尽量得低？**

Vibe Coding 平台是一个典型的「长尾应用」业务：用户随手生成的应用里，绝大多数访问量稀疏、生命周期短暂，但平台又必须为每一个应用持续提供数据库、对象存储、域名和 HTTPS。如果按传统云的思路，给每个应用都常驻一台 VM 或一个容器，算力大部分时间都在空转，而账单却按小时实打实计费，用户越多，亏得越快。

成本压力还来自 Agent 本身。一次「做一个 Todo」可能要跑几十轮 LLM 调用、启动多个 Sandbox、产生大量临时文件和日志。如果 Sandbox 长期保活、会话和运行产物存储没有冷热分层，一次生成任务的运行成本很容易失控。

因此，Vibe Coding 平台需要的基础设施必须在几个维度上做到「按需付费」：

- 计算按需：Sandbox 和应用运行时都要能秒级冷启动、空闲时归零，而不是 7×24 占着资源；
- 存储分层：高频的会话、记忆数据放在线数据库，低频的运行产物数据放对象存储；
- 生命周期管理：会话日志、Sandbox 快照、构建产物这类临时数据要能自动过期或归档，否则存量会随着用户量线性膨胀，最终被存储和带宽费用反噬。

把这四件事拆开看，每一件都已经是复杂的基础设施问题；而 Vibe Coding 平台要做的，是把它们打包成一个「一句话就有应用」的体验。这四件事很少有团队能从零自建，它们对应的是云厂商打磨了十几年的底层能力。对 Vibe Coding 平台来说，更现实的路径是站在一套成熟的云基础设施之上，把精力集中在 Agent 编排和产品体验上。

### 03 CloudBase Vibe Coding 平台解决方案

云开发 CloudBase 是腾讯云推出的 AI-Native 的全栈应用开发平台，专为 AI 驱动的开发流程和终端应用而设计。提供应用开发所需的完整 Serverless 资源，包括身份认证、数据库、云函数与容器、文件存储、Web 应用托管等能力。

从去年开始我们陆续接到很多 Vibe Coding 平台团队的咨询：Sandbox 怎么持久化、用户环境怎么隔离、后端能力怎么开放、集成 CloudBase 和按量计费怎么做。腾讯云开发 CloudBase 的思路，是把已有的 Serverless、BaaS、托管、多租户和权限能力，重新组织成一套面向 Vibe Coding 平台的基础设施方案，分别回应上一节的四个难题。

**3.1 让 Agent 安全、稳定、可恢复地持续运行**

在腾讯云 CloudBase 的 Vibe Coding 平台方案中，把 Agent Loop、Sandbox、Session 和受控边界拆开：Agent Loop 负责编排模型和工具，Sandbox 执行不可信代码，Session 记录会话事件和状态，敏感操作通过 MCP Proxy、管理面 API 和临时凭证完成。这样可以避免把模型推理、代码执行和平台凭证揉在同一个运行环境里。

详细设计见 Agent 运行：Brain / Hands 分离与受控信任边界：https://docs.cloudbase.net/solutions/vibe-coding-platform/agent-runtime

**3.2 让 Agent 生成的应用真正可用**

Vibe Coding 产物不应该只停留在静态页面，而要自然获得数据库、身份认证、云存储、云函数、静态托管、容器托管和 HTTPS 访问能力。CloudBase 把这些后端资源封装成 Agent 可以通过 MCP 调用的工具，让 Agent 用声明式方式完成资源创建和配置，减少 SSH、进程管理、依赖安装、日志排查这类运维动作。

更多能力说明见 应用后端与托管：https://docs.cloudbase.net/solutions/vibe-coding-platform/app-backend

**3.3 让 Agent 跑得更快、更省、更安全**

把 Agent 当成一种新型「开发者」来看，它的工作效率受三件事直接影响：每一步要不要 SSH / shell 这类多轮试错的低带宽操作、调用云资源时要不要查文档凑参数、运行环境本身有没有把不可信代码挡在外面。

CloudBase 在这四点上都按「Agent 视角」做了重新设计：

- MCP 工具替代 SSH + Bash 操作链：登录配置、建表、写权限规则、绑定域名等操作都被封装成结构化 MCP 工具，整个 Todo 应用 Agent 只用 5 次 MCP 调用就完成了原本需要几十次 SSH 的基础设施搭建；
- 结构化返回值替代 raw stdout/stderr：MCP 工具返回的是带错误码和下一步建议的 JSON，而不是 apt 安装进度条、SSH banner 和系统日志——这直接消除了 VM 路径上 140 万+ 个无效 input token 的污染源；
- BaaS 让 Agent「调用」而不是「搭建」后端：Auth、云数据库、云存储、云函数等能力开箱即用，会话管理、数据隔离都通过声明式安全规则一次性配置好，Agent 不需要写 session 中间件，也不会漏写鉴权校验；
- 平台级安全边界：仅 HTTPS API 出网、无 SSH 暴露、平台默认 DDoS 防护，把不可信代码和平台凭证彻底隔离开。

这种「为 Agent 而设计」的差异，最终会直接反映在生成同一个应用所需的时间、Token 和攻击面上。我们用同一个 Todo 应用、同一个 Agent、同一组提示词，在「传统 VM 部署」和「CloudBase」两条路径上做了等价对照实验：

核心结论：在功能等价（匿名会话 + Todo CRUD + 数据隔离）的前提下，CloudBase AI 开发路径比传统 VM 部署快 3.8 倍，Token 消耗降低 52%，且彻底消除了 SSH 暴露带来的安全风险。

这些数字差异最直接的来源是 Agent 的「注意力分配」：在 VM 路径上，Agent 把 76% 的工具调用花在了 SSH、文件传输、进程管理这类运维操作上；而在 CloudBase 路径上，这个比例被压缩到 14%，剩下的 78% 全部回归到「读懂需求、写出代码」这件 Agent 真正擅长的事情上。当 Vibe Coding 平台的成本结构主要由 Token 用量和 Agent 运行时长决定时，这种倍率级的差距会直接转化为单用户的获客成本与毛利。

实测对比见 VM vs CloudBase：同一 Todo 应用的实测对比：https://docs.cloudbase.net/solutions/vibe-coding-platform/vm-vs-cloudbase-comparison

**3.4 把隔离、成本和易用做成统一的多租户模型**

CloudBase 采用「N+1 架构」：1 个平台环境承载 Agent Loop、Sandbox 和模型推理，N 个用户环境承载每个用户生成的应用。每个用户环境拥有独立的数据、计算、存储和访问边界，同时支持按量计费和平台集成，平台方不需要在应用层手写复杂的多租户隔离逻辑。

完整方案见 多租户架构：https://docs.cloudbase.net/solutions/vibe-coding-platform/demo

**3.5 OpenVibeCoding：基于 CloudBase 实现的完全开源的 Vibe Coding 平台**

为了进一步降低开发者的接入门槛，我们基于 CloudBase 实现了一个完全开源的 OpenVibeCoding，可以作为团队验证架构和二次开发的起点。

项目链接：https://github.com/TencentCloudBase/OpenVibeCoding

- 开箱即用：深度集成 CloudBase，AI 能直接操作数据库、云函数和存储；
- Brain/Hands 分离：采用先进的 Brain/Hands 分离的 Harness 架构，提升开发任务完成度；
- 支持多租户：支持多用户/任务隔离，环境秒级创建，能直接给企业内部或客户部署，当 SaaS 用；
- 生产级体验：支持沙箱预览、HMR 热更新，集成小程序工具链，支持小程序发布生成预览二维码。

### 04 CloudBase Vibe Coding 平台解决方案

**4.1 腾讯吐司 APP**

产品链接：https://tusi.qq.com/

吐司 APP 是应用宝团队推出的"应用生成及灵感共创平台"。在吐司中，用户可以输入任何提示词给 AI，由 AI 完成 APP 的开发和部署上线，并生成可直接下载安装的 App。

吐司 APP 采用了 CloudBase 作为应用主要的后端服务平台：

- 云数据库：每一个用户生成的 App 如果涉及后端，都将自动创建一份独立的云数据库实例，数据在租户维度天然隔离，平台方无需在应用层再写一遍多租户逻辑。Agent 在生成阶段可以通过 MCP 直接声明数据模型并写入测试数据，App 上线后真实用户的业务数据也落到同一份数据库里——从「生成」到「运行」走的是同一条数据链路，不存在「生成时一套库、上线后再迁一次」的接缝。
- 身份认证：CloudBase 内置了微信、手机号、邮箱、匿名登录等多种鉴权方式，吐司生成的 App 默认就具备「能登录、能识别用户、能区分数据归属」的能力。用户不需要在提示词里专门要求「帮我做一套登录系统」，Agent 也不需要现写一遍 OAuth / Session 管理；身份是平台的默认能力，App 从第一行代码起就自带账户体系，可以直接做收藏、个人空间、社交分享这类强账户特性。
- MCP / Skills：CloudBase 把数据库、云函数、云存储、静态托管、容器托管等后端资源全部包装成标准 MCP 工具，并沉淀了一批面向常见场景（用户体系、内容存储、AI 调用、支付等）的可复用 Skills。吐司的 Agent 在生成 App 时不需要写 DDL、不需要配 Nginx、不需要管 HTTPS 证书，而是用声明式的方式调用 MCP，由 CloudBase 负责资源创建、配置发布和运行维护。这让 Agent Loop 显著变短、出错面大幅收敛，也让生成出来的 App 一上线就具备生产级的工程质量。

**4.2 GenieAI（CodeBuddy）**

产品链接：https://genie.codebuddy.ai/

GenieAI 是腾讯云 CodeBuddy 推出的在线 AI 编程工作台，覆盖从需求理解、界面设计、代码生成到数据库配置、一键部署上线的全流程，可以基于一句话需求生成网页应用、小程序、AI 应用、小游戏等多种形态的产品。

与吐司面向 C 端「让普通人做出自己的 App」不同，GenieAI 更偏 Pro-sumer：服务的是产品经理、创业者、独立开发者，生成出来的项目不是 Demo，而是包含前后端和数据库的完整全栈应用，并且可以直接交付给团队继续在 CodeBuddy IDE 里做二次开发。

GenieAI 选择 CloudBase 作为后端基础设施，用到的能力包括：

- PostgreSQL 数据库：GenieAI 面向的是「真正会拿来用的应用」，因此后端默认采用 CloudBase 提供的 PostgreSQL 数据库，原生支持事务、外键、复杂 SQL、向量检索等关系型数据库特性，足以承载从内部工具、CRM、AI 应用到中小型 SaaS 的真实业务负载。每一个用户生成的项目都自动获得一份独立的 PG 实例，实例级别的租户隔离避免了「一个项目慢查询拖垮全平台」的风险，同时 Agent 在生成阶段就可以直接基于标准 SQL 建表、写约束、加索引，不需要去适配某种私有的简化数据模型。
- 身份认证（Google 登录）：GenieAI 服务的是全球用户，因此身份方案天然要面向海外。CloudBase 身份认证支持 Google、Apple、邮箱、手机号等多种登录方式，GenieAI 生成的应用一键即可获得「Sign in with Google」能力，由 CloudBase 负责 OAuth 流程、Token 管理、会话续期和跨端同步——Agent 不需要在每个项目里再写一遍 OAuth 客户端，也不需要为不同应用分别申请和维护 Google OAuth Client，开箱即用且符合海外用户预期。
- 一键部署与白标域名：GenieAI 生成的应用通过 CloudBase 静态托管和云函数完成部署，每一个项目都能在生成完成后秒级获得一条可访问的 HTTPS 链接，并支持绑定用户自有域名。整个发布流程被 MCP 化，Agent 通过声明式调用即可完成构建、上传、发布和回滚，不需要触碰服务器、Nginx 或证书。

### 05 Agent 是新时代的开发者，云也需要为它重新设计

把镜头拉远一点看，吐司、GenieAI、Lovable、v0 这些 Vibe Coding 平台真正在改变的，是软件的生产者：未来在云上下达指令、调用资源、把代码部署到生产环境的，将不再主要是人类工程师，而是大量并发运行、永不疲倦的 AI Agent。

如果接受这一点，那随之而来的结论几乎是必然的：面向人类开发者设计的云，并不是面向 Agent 设计的云。

Agent 友好云的诉求 vs CloudBase 的做法：

- 后端能力要能被 Agent 直接调用，而不是藏在控制台 UI 背后 → 数据库、云函数、存储、托管、身份等全部能力都以 MCP 工具暴露，并配套一批沉淀好的 Skills，Agent 用一行声明就能创建资源
- 文档要能塞进上下文窗口，而不是依赖工程师在 Stack Overflow 上摸索 → 提供面向 LLM 优化的精简文档与 llms.txt，Agent 可以一次性把所需上下文加载进 prompt
- 资源要能秒级冷启动、用完即走，匹配 Agent 短任务、高并发、强爆发的工作模式 → 数据库、云函数、容器托管全栈 Serverless，支持缩容到零和毫秒级弹起，按实际用量计费
- 需要安全沙箱来运行 Agent 生成的不可信代码 → 提供 CloudBase Sandbox，为每个会话提供隔离的执行环境，支持快照与断点恢复，避免不可信代码污染平台
- 模型推理也要像调用云资源一样简单 → 统一内置 AI 模型调用网关，对接主流大模型，统一鉴权、计费和限流，应用侧无需逐家集成 SDK

这张表里的每一项单独看都不算颠覆，但放在一起，回答的其实是一个更大的问题：当软件的生产者从人换成 Agent，云本身应该被重新设计成什么样？而这个问题，并不是云计算行业第一次面对。

AI is the new electricity. 电力刚出现时，许多工厂只是把蒸汽机换成了电动机，但仍然保留了中央驱动轴、皮带传动和老式厂房布局，生产力提升相当有限。

蒸汽时代的工厂之所以效率低，是因为所有机器都被一根中央驱动轴绑死：必须挤在轴的周围、必须同时开停、坏一台得停全厂，厂房的形状、流程的顺序、车间的扩张能力全部被这根轴限制住。

真正的爆发，发生在工程师意识到电力可以「分布式供给」的那一刻：小型电动机被直接装到每一台机床上，机器之间彼此独立、按需启停，厂房不再被驱动轴绑架，可以按生产流程而不是动力源来重新布局。于是流水线诞生了，工厂可以横向扩张、可以局部停机检修、可以为不同产品重排车间。

电力之所以带来数量级的生产力提升，不是因为它「比蒸汽更强」，而是因为它解除了原有架构对生产组织的约束。

云之于 AI Agent 也正在经历同样的过程。把传统云硬塞给 Agent，相当于给电动机配上中央驱动轴，虽然能用，但浪费了这场技术变革本身的杠杆。真正的机会，是按 Agent 的工作方式重新设计云：Brain / Hands 分离的运行时、声明式 MCP 工具、一租户一环境的多租户模型、为长尾应用而生的 Serverless 计费。这些不是对老云的修修补补，而是「为 Agent 重新画一遍图纸」。

### 06 结语

回顾一下这篇文章想说的事：

- Vibe Coding 平台是 AI 编程下半场的主战场，它要面对的不再是「让 AI 写代码」，而是「让 AI 自主生产可用的软件」；
- 支撑这种新形态需要一套完整的基础设施，Agent Runtime、安全沙箱、应用后端、多租户隔离，缺一不可；
- 这套基础设施有自己的设计原则：Brain / Hands 分离、受控信任边界、一租户一环境、按量计费、MCP 标准化；
- 腾讯云 CloudBase 提供了完整的解决方案，并以 OpenVibeCoding 作为开源参考实现。

如果你正在做一个 Vibe Coding 平台，或者在思考怎么把 AI Agent 真正落地到生产环境，承载海量用户的使用。欢迎读一下我们的方案文档，也许会对你有所启发。

### 参考链接

- Lovable：https://lovable.dev/
- Lovable ARR 突破 4 亿美元报道：https://techcrunch.com/2026/03/11/lovable-says-it-added-100m-in-revenue-last-month-alone-with-just-146-employees/
- Vercel v0：https://v0.app/
- Vercel ARR 数据：https://sacra.com/c/vercel/
- Vercel F 轮融资报道：https://www.linkedin.com/pulse/v0-ai-platform-statistics-2026-users-revenue-adoption-pantodev-bvzfc
- Replit：https://replit.com/
- Replit D 轮融资报道：https://blog.replit.com/replit-raises-400-million-dollars
- Loopit：https://loopit.ai/
- Loopit 融资报道：https://finance.sina.com.cn/wm/2026-04-23/doc-inhvntfa5409311.shtml
- 腾讯云开发 CloudBase：https://cloud.tencent.com/product/tcb
- CloudBase Vibe Coding 平台解决方案文档：https://docs.cloudbase.net/solutions/vibe-coding-platform/overview
- Agent 运行：Brain / Hands 分离与受控信任边界：https://docs.cloudbase.net/solutions/vibe-coding-platform/agent-runtime
- 应用后端与托管：https://docs.cloudbase.net/solutions/vibe-coding-platform/app-backend
- VM vs CloudBase：同一 Todo 应用的实测对比：https://docs.cloudbase.net/solutions/vibe-coding-platform/vm-vs-cloudbase-comparison
- 多租户架构：https://docs.cloudbase.net/solutions/vibe-coding-platform/multi-tenancy
- OpenVibeCoding 开源项目：https://github.com/TencentCloudBase/OpenVibeCoding
- GenieAI：https://genie.codebuddy.ai/
- CloudBase llms.txt：https://docs.cloudbase.net/llms.txt
