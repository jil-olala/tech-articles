# 基于钉钉机器人的 Qoder CLI / Claude Code 双引擎 AI 助手实践

- **来源：** 微信公众号（阿里技术）
- **作者：** 久梦
- **日期：** 2026-06-02
- **原文链接：** https://mp.weixin.qq.com/s/UdQ7xhM25Er6Eyk0xs577w

![image](images/dingtalk-dual-engine/img-01.jpg)
## 阿里妹导读
文章内容基于作者个人技术实践与独立思考，旨在分享经验，仅代表个人观点。
## 一、背景与问题
在闪购搜索团队的日常工作中，我们需要频繁地进行搜索问题排查、性能分析、实验管理等操作。这些操作分散在多个平台（SLS日志、TPP实验平台、代码仓库等），效率低下。
我们的目标是：在钉钉群里直接对话一个AI助手，它能代替人去查日志、看实验、分析性能、甚至部署代码。
然而面临几个核心挑战：
挑战
具体描述
内网部署限制
服务部署在内网，无法暴露公网回调地址，传统Webhook方案不可行
实时性要求
AI推理耗时较长(30s-120s)，用户无法接受"发消息→等2分钟→一次性返回"
安全性要求
需要权限隔离，非管理员不能执行写操作(部署代码、修改配置等)
工具集成需求
需要访问代码仓库、日志系统、实验平台等多种外部工具
## 二、方案概览
**2.1 整体架构**
我们最终采用 "钉钉 Stream + CLI 代理" 的方案：
```
┌────────────────────────────────────────────────────────────────┐
│                    钉钉群 / 单聊                               │
├────────────────────────────────────────────────────────────────┤
│  钉钉 Stream WebSocket 长连接（内网直通，无需公网回调）        │
├────────────────────────────────────────────────────────────────┤
│                  Java 服务（alsc-intervene）                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  DingTalkStreamService                                    │  │
│  │  • 权限校验（管理员 vs 只读用户）                        │  │
│  │  • 上下文管理（LRU + TTL + 滑动窗口）                    │  │
│  │  • 并发控制（线程池 10-15）                              │  │
│  │  • AI卡片投放与流式更新                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                    │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  CLI 代理层（ProcessBuilder）                             │  │
│  │  • Qoder CLI / Claude Code（可切换）                     │  │
│  │  • stdbuf -oL 行缓冲优化                                │  │
│  │  • 120s 超时保护 + 异常杀进程                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                    │
│                           ▼                                    │
│              MCP Server（工具调用层）                          │
└────────────────────────────────────────────────────────────────┘
```
**2.2 关键技术选型**
组件
方案
选择理由
消息通道
钉钉 Stream（WebSocket）
内网无需公网回调地址，完全规避DNS/防火墙限制
AI引擎
CLI 代理模式
轻量无框架依赖，复用成熟 CLI 生态
流式展示
钉钉 AI 卡片
原生支持打字机效果，用户体验好
进程管理
Java ProcessBuilder
进程级隔离，一个请求一个进程，互不影响
工具扩展
MCP（Model Context Protocol）
标准化协议，可插拔配置驱动
上下文存储
内存 LinkedHashMap
自动 LRU 淘汰，无需外部存储
## 三、引擎选型：从 Qoder CLI 到 Claude Code

**3.1 最初选择 Qoder CLI**
项目最初选择 Qoder CLI 作为 AI 引擎，主要考虑：
- 内部产品，接入方便，有现成的 Skills 和 MCP 生态
- CLI 模式天然适合服务端 spawn 调用
- 支持 stream-json 流式输出
**3.2 遇到的问题**
实际使用中发现，Qoder CLI 在复杂问题排查场景下的表现不如预期：
- 对于需要多步推理的复杂排查（如跨系统问题定位），回答质量有时不够准确
- 与 Qoder IDE 桌面端相比，CLI 模式的能力有一定差距
**3.3 引入 Claude Code**
为了提升复杂场景下的回答质量，我们后续引入了 Claude Code 作为替代引擎。实际效果验证：
- 复杂问题排查能力显著更强：对于涉及多系统、多步骤的排查场景，Claude Code 的推理深度和准确性明显优于之前的方案
- MCP 工具调用更稳定：工具调用的成功率和参数构造准确度更高
目前两个引擎并行部署，通过不同的入口调用：
- Qoder CLI：钉钉 Stream 入口（群聊场景）
- Claude Code：HTTP SSE 入口 + 独立钉钉机器人
## 四、Docker 部署方案
**4.1 部署架构**
两个引擎部署在同一个 Docker 容器内，共享工作目录和 MCP 配置：
```
Docker 容器
├── /home/admin/qoder-workspace/     # 共享工作目录
│   ├── .mcp.json                    # MCP服务配置
│   ├── AGENTS.md                    # Qoder CLI 知识入口
│   └── CLAUDE.md                    # Claude Code 指令文件
├── /home/admin/.qoder/              # Qoder CLI 专属配置
│   ├── settings.json                # MCP server列表
│   ├── mcp-oauth-tokens.json        # OAuth认证token
│   └── skills/                      # 安装的Skills
├── /home/admin/.claude/             # Claude Code 专属配置
│   └── settings.json                # 环境变量 + 权限
└── /opt/mcp-auth/                   # Claude MCP认证文件
```
**4.2 Dockerfile 核心设计**
```
FROM reg.docker.alibaba-inc.com/alibase/alios7u2-min
# 基础运行时
RUN yum install -y ajdk11-11.0.27.26 nodejs git
# 安装双引擎
RUN npm install -g @qoder-ai/qodercli \
        --registry=https://registry.npm.alibaba-inc.com && \
    npm install -g @ali/claude-code@2.1.144 \
        --registry=https://registry.npm.alibaba-inc.com
# 部署工作区
COPY qoder-workspace/ /home/admin/qoder-workspace/
# Qoder CLI 认证
COPY qoder-home/.qoder/ /home/admin/.qoder/
RUN chmod 600 /home/admin/.qoder/mcp-oauth-tokens.json && \
    chmod 444 /home/admin/.qoder/mcp-oauth-clients.json
# Claude Code 认证
COPY claude-home/.claude/ /home/admin/.claude/
COPY claude-mcp-auth/ /opt/mcp-auth/
RUN ln -sf /opt/mcp-auth /home/admin/.mcp-auth
```
**4.3 Java 层调用方式**
两个引擎的调用方式几乎相同，都是通过 ProcessBuilder spawn 子进程：
```
// Qoder CLI
ProcessBuilder pb = new ProcessBuilder(
    "stdbuf", "-oL", "qodercli",
    "-p", prompt,
    "--output-format", "stream-json",
    "--yolo", "--max-turns", "10",
    "-w", "/home/admin/qoder-workspace"
);
pb.environment().put("QODER_PERSONAL_ACCESS_TOKEN", qoderToken);
// Claude Code
ProcessBuilder pb = new ProcessBuilder(
    claudeCliPath, "-p", prompt,
    "--output-format", "stream-json",
    "--verbose", "--max-turns", "5"
);
pb.environment().put("ANTHROPIC_AUTH_TOKEN", anthropicToken);
```
两者输出格式统一为 stream-json，Java 层的流式解析、AI卡片更新逻辑完全复用。
五、MCP 工具集成与 OAuth 认证跳过方案
**5.1 MCP 是什么**
MCP（Model Context Protocol）是 AI 调用外部工具的标准化协议。通过 MCP，AI 可以像人一样去调用各种平台的 API——查代码、看日志、管实验，实现"问一句话，帮你查完所有系统"的效果。
我们从本地常用的 MCP 服务中选择了一部分部署到远端，覆盖代码仓库、日志查询、实验管理等核心场景。
**5.2 MCP 认证问题**
MCP 服务默认采用 OAuth2 认证流程：
- 客户端向 MCP 网关发起请求
- 网关返回 401 + OAuth discovery URL
- 客户端走 OAuth 授权码流程获取 access_token
- 后续请求携带 Bearer token
这个流程在本地开发时没问题（浏览器打开授权页面点击授权），但在无头服务器（Docker容器）上完全行不通——没有浏览器，无法完成交互式授权。
**5.3 跳过 OAuth 的方案：
静态 Bearer Token**
我们的解决方案是预先获取 token，以静态方式注入配置文件，跳过运行时的 OAuth 流程：
```
// .mcp.json - 直接在 headers 中携带 Bearer token
{
  "mcpServers": {
    "code": {
      "type": "streamable-http",
      "url": "https://mcp.alibaba-inc.com/code/mcp",
      "headers": {
        "Authorization": "Bearer mcpa_xxxxxxxxxxxxxxxx"
      }
    },
    "sls-mcp": {
      "type": "streamable-http",
      "url": "https://mcp.alibaba-inc.com/sls-mcp/mcp",
      "headers": {
        "Authorization": "Bearer mcpa_yyyyyyyyyyyyyyyy"
      }
    }
  }
}
```
工作原理：
- 本地预获取：在本地开发环境通过正常 OAuth 流程获取 access_token（mcpa_ 前缀的长期token）
- 静态注入：将 token 写入 .mcp.json 的 headers.Authorization 字段
- 直接认证：CLI 发起 MCP 请求时直接携带此 header，MCP 网关验证 token 有效即放行，完全跳过 OAuth 握手
- Docker 构建时打包：.mcp.json 在 Dockerfile 中 COPY 进容器，启动即可用
Token 管理要点：
- Token 有有效期，过期后需要重新获取并更新配置
- .mcp.json 文件权限设为 600，防止泄露
- mcp-oauth-clients.json 设为 444 只读，防止运行时被 DCR（Dynamic Client Registration）覆盖
**5.4 Qoder CLI 的 OAuth Token 方式**
Qoder CLI 还支持另一种认证方式——通过 mcp-oauth-tokens.json 文件存储 OAuth token：
```
[
  {
    "serverName": "code",
    "token": {
      "accessToken": "mcpa_xxxxx",
      "refreshToken": "mcpr_xxxxx",
      "expiresAt": 1782186490063,
      "tokenType": "Bearer"
    },
    "clientId": "mcp-client-xxxx",
    "tokenUrl": "https://mcp.alibaba-inc.com/oauth/token",
    "mcpServerUrl": "https://mcp.alibaba-inc.com/code/mcp"
  }
]
```
理论上 refreshToken 可以自动续期，但实测发现远端环境中 token 刷新不够可靠，因此 Claude Code 端我们统一使用 .mcp.json 静态 headers 方案，更加稳定。
六、钉钉 Stream 集成核心实现
**6.1 为什么选择 Stream 模式**
传统钉钉机器人采用 HTTP 回调，要求服务有公网可达地址。Stream 模式通过 WebSocket 长连接解决了这个问题——服务主动连接钉钉服务器，无需暴露任何端口。
```
@PostConstruct
public void init() {
    if (!streamEnabled) return;
    OpenDingTalkClient client = OpenDingTalkStreamClientBuilder.custom()
        .credential(new AuthCredential() {{
            setClientId(appKey);
            setClientSecret(appSecret);
        }})
        .registerCallbackListener("/v1.0/im/bot/messages/get", this)
        .build();
    client.start();
}
```
**6.2 进程管理与流式输出**
```
public void chatStream(String prompt, String workspace,
                       Consumer lineConsumer, Consumer processConsumer) {
    List cmd = Arrays.asList(
        "stdbuf", "-oL",           // 强制行缓冲
        cliPath, "-p", prompt,
        "--output-format", "stream-json",
        "--yolo", "--max-turns", "10",
        "-w", workspace
    );
    ProcessBuilder pb = new ProcessBuilder(cmd);
    pb.environment().put("QODER_PERSONAL_ACCESS_TOKEN", token);
    pb.redirectErrorStream(true);
    Process process = pb.start();
    processConsumer.accept(process);
    try (BufferedReader reader = new BufferedReader(
            new InputStreamReader(process.getInputStream()), 256)) {
        String line;
        while ((line = reader.readLine()) != null) {
            lineConsumer.accept(line);
        }
    }
    if (!process.waitFor(120, TimeUnit.SECONDS)) {
        process.destroyForcibly();
    }
}
```
关键设计点：
- stdbuf -oL：强制行缓冲，避免 Node.js 4KB 全缓冲导致的延迟
- 256字节 BufferedReader：Java 侧小缓冲确保及时输出
- 异常即杀进程：consumer 异常立即 destroyForcibly，停止消耗推理额度
- 进程引用暴露：支持用户发"停止"命令时主动中断
**6.3 AI卡片流式更新**
```
// 频率控制：累计超50字符才更新一次
if (fullContent.length() - lastUpdateLen > 50) {
    String content = fullContent.length() > 3000
        ? fullContent.substring(0, 3000)
        : fullContent.toString();
    cardService.streamUpdate(trackId, content, false, false);
    lastUpdateLen = fullContent.length();
}
```
**6.4 用户上下文管理（三重防护）**
```
private final LinkedHashMap contextMap =
    new LinkedHashMap<>(16, 0.75f, true) {
        @Override
        protected boolean removeEldestEntry(Entry eldest) {
            return size() > 500;
        }
    };
```
保护层
机制
触发条件
效果
第一层
TTL过期
48小时无活动
清空上下文
第二层
滑动窗口
单用户超200KB
FIFO删除最早对话
第三层
LRU淘汰
全局超500用户
淘汰最久未使用
**6.5 权限隔离**
- 管理员：完全权限（读写+命令执行+部署）
- 普通用户：只读模式（系统指令最高优先级强制约束）
**6.6 并发控制**
```
private final ExecutorService qoderExecutor = new ThreadPoolExecutor(
    10, 15, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(30),
    new ThreadPoolExecutor.AbortPolicy()
);
```
七、知识自进化机制
五级知识沉淀模型，让 AI 越用越聪明：
```
L0  git history（自动追踪代码变更）
 ↓
L1  .qoder/context/（每次任务的过程报告）
 ↓
L2  .qoder/memory_recent.md（最近5次会话摘要）
 ↓
L3  .qoder/rules/candidates/（候选规则，经验沉淀）
 ↓
L4  AGENTS.md + P0-constraints.md（正式规则）
```
候选规则触发 ≥3 次且成功率 ≥80%，自动提议晋升为正式规则。
八、钉钉机器人配置与申请指南
**8.1 创建机器人应用**
- 登录钉钉开放平台[1]
- 创建企业内部应用 → 选择"机器人"类型，选群聊机器人即可（只需要主管审批）
- 获取 appKey 和 appSecret
**8.2 开通必要权限**
目前看下来内部申请的机器人默认都开通了下面三项重要的权限，如果有遗漏，可能得单独申请一下
| 权限名称 | 用途 |
|----------|------|
| 企业内机器人发送消息 | 主动回复用户消息 |
| 互动卡片实例写权限 | 创建和投放AI卡片 |
| AI卡片流式更新权限 | 实现打字机效果 |
**8.3 启用 Stream 模式**
- 机器人配置页 → "消息接收模式" → Stream 模式
- 无需填写回调URL
- 发布到组织内
**8.4 AI卡片模板配置**
- 进入卡片平台[2]
- 新建卡片模板 → 场景选"消息卡片 + AI卡片"
- 开启"流式组件"开关
- 记录模板ID配置到 application.properties
**8.5 关键配置项**
```
dingtalk.stream.enabled=true           # Stream开关（预发/线上设false）
dingtalk.app.key=${DINGTALK_APP_KEY}   # antx注入
dingtalk.app.secret=${DINGTALK_APP_SECRET}
dingtalk.robot.code=${DINGTALK_ROBOT_CODE}
```
**8.6 注意事项**
- 多实例部署需通过环境开关控制，只在日常环境开启 Stream，否则消息会重复处理
- 凭证必须通过 antx/diamond 注入，禁止硬编码
- AI卡片需选择"AI卡片"场景并开启流式组件
九、运行效果展示
**9.1 性能追踪**
![image](images/dingtalk-dual-engine/img-02.png)
**9.2 问题场景**
![image](images/dingtalk-dual-engine/img-03.png)
![image](images/dingtalk-dual-engine/img-04.png)
![image](images/dingtalk-dual-engine/img-05.png)
**9.3 需求跟进**
（还需要继续验证下模型能力的上限，目前用qoder cli试过不太稳，但也能处理一些基本问题，复杂问题回答可能会跟本地的qoder有偏差，claude试下来效果更好~）
![image](images/dingtalk-dual-engine/img-06.png)
![image](images/dingtalk-dual-engine/img-07.png)
十、踩坑经验
**10.1 stdbuf 行缓冲是必须的**
Node.js 进程在非 TTY 环境下默认全缓冲（4KB），不加 stdbuf -oL 用户会看到"卡住→突然一大段"的糟糕体验。
**10.2 MCP OAuth Token 格式要求**
mcp-oauth-tokens.json 必须是 JSON 数组格式 [{...}]，对象格式会导致 crash。
**10.3 Stream 模式多实例冲突**
多容器实例同时开启 Stream 监听会导致消息重复处理，需用环境开关精确控制。
**10.4 AI 卡片权限**
流式更新需单独申请"AI卡片流式更新权限"，否则返回 403。
**10.5 Claude Code 权限透传**
Claude Code 的 -p 模式不会自动应用 settings.json 中的 permissions，需通过 --allowedTools 参数显式传递工具权限。
十一、总结
本方案通过 钉钉 Stream + CLI 代理 的架构，实现了：
- 完全内网部署：WebSocket 长连接规避公网回调
- 实时流式回复：stdbuf + AI卡片打字机效果
- 安全权限隔离：管理员/只读双模式
- MCP 工具开放：静态 Bearer token 跳过 OAuth，实现无头环境下的工具调用
- 引擎可切换：从 Qoder CLI 到 Claude Code，复杂排查能力显著提升
- 生产级稳定：线程池+超时+LRU 三层保护
以最低的侵入度、最轻的工程成本，实现了企业级 AI 助手从零到一的落地。由于代码中涉及了很多私人的配置项，不直接公开了，感兴趣的同学可以私我拿代码。
参考链接:
[1] https://open.dingtalk.com/?spm=ata.21736010.0.0.67567536U8RSTo
[2] https://card.dingtalk.com/?spm=ata.21736010.0.0.67567536U8RSTo
                        预览时标签不可点
          微信扫一扫
关注该公众号
  继续滑动看下一个
              轻触阅读原文
![image](images/dingtalk-dual-engine/img-08.png)
                        阿里云开发者
        向上滑动看下一个
      知道了
          微信扫一扫
使用小程序
          ****
          取消
          允许
      ****
      取消
      允许
      ****
      取消
      允许
  ×
  分析