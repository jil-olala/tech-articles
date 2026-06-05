# 和 Hermes Agent 聊了一下午，我把这个 AI Agent 框架摸透了

> 最近在研究 AI Agent 框架，花了一个下午和 Hermes Agent 深度对话，从配置到架构到核心概念，算是把这套系统摸了个透。这篇文章整理一下整个过程，帮你快速理解 AI Agent 的关键技术细节。

## 一、Hermes Agent 是什么？

Hermes Agent 是 Nous Research 开源的 AI Agent 框架，和 Claude Code、OpenAI Codex 同类——都是一个能用工具、能执行任务的 AI 助手。

**核心特点：**

- **多平台**：Telegram、Discord、Slack、微信、钉钉、飞书……10+ 平台都能接入
- **Provider 自由切换**：OpenRouter、Anthropic、OpenAI、智谱 GLM、DeepSeek、本地模型……随便换
- **持久记忆**：跨会话记住你是谁、你的偏好、你的项目约定
- **Skill 系统**：Agent 解决复杂问题后，可以把经验沉淀成可复用的 Skill
- **Sub-agent 子代理**：自动拆分任务，多个 Agent 并行干活

## 二、从一个 404 错误说起

接入微信后，每条消息后面都跟了一个错误：

```
⚠ Auxiliary title generation failed: HTTP 404: 
Error code: 404 - path: '/v4/v1/messages'
```

原因很简单：Hermes 默认会调用一个**辅助模型**给每次会话生成标题。但我用的是智谱 GLM 的自定义 endpoint，辅助模型找不到对应的 API 路径，就 404 了。

**解决方案：直接关掉。**

```bash
hermes config set auxiliary.title.enabled false
hermes gateway restart
```

这也反映了一个设计理念：AI Agent 系统里，**不是每个功能都需要**。坏了就关掉，不要过度工程化。

## 三、Provider 路由：想让模型走指定供应商怎么办？

用 OpenRouter 的同学可能知道，同一个模型（比如 `deepseek/deepseek-v4-pro`）背后可能有多个供应商在提供算力：DeepSeek 官方、百度云、Azure……

**问题来了：能不能指定走某个供应商？**

答案是可以。Hermes 通过 `provider_routing` 配置实现：

```yaml
# config.yaml
provider_routing:
  order:                    # 优先但不强制
    - baidu/fp8
```

这里有三种策略：

| 策略 | 配置 | 效果 |
|------|------|------|
| **强制** | `only: ['baidu/fp8']` | 只走这个供应商，不支持就报错 |
| **优先** | `order: ['baidu/fp8']` | 优先走它，不行自动回退 |
| **忽略** | `ignore: ['xxx']` | 排除某个供应商 |

**踩坑记录：** 我一开始用了 `only`（强制），结果 `baidu/fp8` 不支持 tool use（工具调用），直接报 404。改成 `order`（优先）后，不支持的功能会自动回退到其他供应商。

这个设计很实用——在成本和功能之间做权衡时，`order` 是最灵活的选择。

## 四、微信公众号文章自动归档

我给 Hermes 定了一个日常工作流：发一个微信文章链接，它自动抓取内容、按类别归档到本地。

**实际操作：**

```
我：https://mp.weixin.qq.com/s/RCEGqAvfJxbzaPJIJDOKwA
```

Hermes 会：
1. 用 curl 抓取文章 HTML（模拟微信 UA 绕过验证码）
2. 提取标题、作者、日期、正文
3. 按类别创建文件夹，保存为 Markdown

最终目录结构：

```
~/articles/
├── AI/
│   ├── Agent/
│   │   └── AI_Agent核心概念_Model_Tool_Skill_Harness.md
│   └── 助手实践/
│       └── 基于钉钉机器人的_Qoder_CLI_Claude_Code_双引擎_AI_助手实践.md
```

**技术细节：** 微信公众号的验证码（滑块拼图）会拦截浏览器自动化。解决方案是直接用 curl + 微信 UA 抓取，绕过前端验证。文章内容在 `id="js_content"` 的 div 里，用正则提取即可。

## 五、记忆系统：让 Agent 记住你是谁

Hermes 支持多种记忆后端，我选了 **mem0**：

```bash
hermes config set memory.provider mem0
# 在 .env 中配置 MEM0_API_KEY
```

**记忆系统做了什么？**

- **用户画像**：自动记录你的身份、偏好、工作领域
- **跨会话记忆**：上次聊的内容，下次还能记得
- **语义搜索**：不只是关键词匹配，能理解意思来检索

比如我现在，Hermes 记住了：

> 用户是大模型算法全栈工程师，管理外包团队。工作涉及架构设计、大模型微调（SFT/RLHF/DPO/GRPO）、外包团队管理、技术写作。

这样下次对话，它就能基于这些上下文给更精准的回答。

## 六、SOUL.md：给 Agent 一个灵魂

这是我觉得最有意思的设计。

**SOUL.md** 是一个放在 `~/.hermes/SOUL.md` 的文件，定义 Agent 的"人格"。改完立刻生效，不用重启。

我给 Agent 取了个名字叫**旺财**，定义了它的工作风格：

```markdown
# 旺财 - AI 技术助理

你叫旺财，是用户的 AI 技术助理。用户是一位大模型算法全栈工程师，同时管理外包团队。

**沟通风格：**
- 用中文回复，称呼自己为"旺财"
- 简洁专业，少废话，直接给方案或结论
- 技术讨论时精准使用术语，不解释基础知识（除非被问到）
- 代码和架构方案优先，先给可执行的答案，再补充解释

**旺财应该主动做的：**
- 发现架构问题或潜在风险时主动提醒
- 给出方案时附带优缺点对比
- 技术选型时给出推荐和理由
- 外包任务拆解时注意粒度合理、验收标准清晰、交付物明确
```

**设计哲学很清晰：**

| 文件 | 管什么 | 作用域 |
|------|--------|--------|
| **SOUL.md** | Agent 是谁、怎么说话 | 全局 |
| **AGENTS.md** | 项目代码规范、开发约定 | 项目级 |
| **config.yaml** | 模型、工具、平台等技术配置 | 全局 |

SOUL.md 管"灵魂"，AGENTS.md 管"规矩"，config.yaml 管"能力"。三层分离，互不干扰。

## 七、Sub-agent：自动拆分任务，并行干活

Sub-agent（子代理）是 Hermes 的杀手级特性。

**和 Tool、Skill 的区别：**

| 层级 | 是什么 | 举例 |
|------|--------|------|
| **Tool** | 一个动作 | 调 API、读文件 |
| **Skill** | 一套方法 | "排查 bug 的标准流程" |
| **Sub-agent** | 独立的小 Agent | 能自己思考、调工具、完成子任务 |

**工作流：**

```
用户："帮我调研 vLLM 和 TGI 的性能对比"

主 Agent
  ├── Sub-agent A → 搜索 vLLM 相关资料
  ├── Sub-agent B → 搜索 TGI 相关资料
  └── 汇总结果 → 返回给用户
```

关键配置：

```yaml
delegation:
  max_concurrent_children: 3    # 最多 3 个 sub-agent 并行
  max_spawn_depth: 1            # 不允许子代理再派子代理
  child_timeout_seconds: 600    # 10 分钟超时
  default_toolsets:
    - terminal
    - file
    - web
```

**不需要手动创建。** 你只管说需求，Agent 自己判断要不要拆分、怎么拆分。

## 八、Kanban 看板：多 Agent 任务协作

如果你的项目需要多个 Agent 分工协作，Kanban 就是任务调度中心。

```
你（PM）→ kanban create "设计用户表结构"
              kanban create "用户注册 API"
              kanban create "登录页面 UI"
                    ↓
        Dispatcher（自动调度）
              ↓          ↓
        Backend Agent   Frontend Agent
        （做后端任务）    （做前端任务）
```

**核心能力：**
- 任务依赖（前端等 API 定义好了再开工）
- 自动重试（Agent 挂了会回收任务重新分配）
- 代码隔离（每个 Agent 用独立 worktree）

**特别适合外包团队管理的场景** — 你用自然语言描述需求，AI 帮你拆任务、建看板、分配执行。

## 九、Gateway 多租户：一个实例服务多人

最后一个值得说的设计：Gateway 天然支持多用户。

不需要为每个用户创建独立的 Profile。Gateway 自动按 `user_id` 隔离：

```
用户A（微信）→ session_a（独立会话+记忆）
用户B（微信）→ session_b（独立会话+记忆）
用户C（钉钉群）→ session_c（独立会话+记忆）
```

- **Session 隔离**：每个用户独立对话历史
- **Memory 隔离**：mem0 按 user_id 存储，A 看不到 B 的记忆
- **权限隔离**：可配置不同用户的不同工具权限

**Profile 的正确用法是"同一用户多身份"**（工作模式 vs 个人模式），而不是"多用户"。

## 十、总结：AI Agent 的核心是系统，不是模型

回顾一下这些概念，其实可以用一张图概括：

```
                    ┌─────────────────┐
                    │   SOUL.md       │ → 我是谁
                    │   (人格/灵魂)    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Agent Loop    │ → 怎么想
                    │   (推理+决策)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───┐  ┌──────▼──────┐  ┌───▼────────┐
     │   Tools    │  │   Skills    │  │ Sub-agents │ → 怎么干
     │   (动作)   │  │   (套路)    │  │  (分工)    │
     └────────────┘  └─────────────┘  └────────────┘
              │              │              │
     ┌────────▼──────────────▼──────────────▼────────┐
     │              AGENTS.md / 项目约定              │ → 规矩
     └──────────────────────┬────────────────────────┘
                            │
     ┌──────────────────────▼────────────────────────┐
     │           config.yaml / Provider              │ → 能力
     └───────────────────────────────────────────────┘
```

**一句话总结：** AI Agent 不是一个大模型，而是一整套围绕模型搭建的系统。模型负责理解和决策，工具负责行动，执行系统负责把任务一轮轮推进下去。

把这些概念分清之后，再去看各种 Agent 产品、框架和论文，就不会那么容易混乱了。

---

*本文基于与 Hermes Agent 的实际对话整理。Hermes Agent 是开源项目，GitHub: https://github.com/NousResearch/hermes-agent*
