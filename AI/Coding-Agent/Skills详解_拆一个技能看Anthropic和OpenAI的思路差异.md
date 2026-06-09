# Skills 详解：拆一个技能，看 Anthropic 和 OpenAI 的思路差异

- **来源：** 微信公众号
- **作者：** 架构师（JiaGouX）
- **日期：** 2026-03-04
- **原文链接：** https://mp.weixin.qq.com/s/31CPyi0FB5tfXOzL7cIOtQ

---

我们之前写过不少关于 Skills 的文章——从《代码就是一切：Anthropic 为什么从 Agent 转向 Skills》到《Skill 到底是什么》再到《Anthropic 官方 33 页指南拆解: 构建Skills的最佳实践》，概念和机制基本讲透了。
但有一个问题一直没展开：**一个真实的 Skill 内部到底长什么样？它为什么可以这么高效？**
今天换个思路: 拿 Anthropic 官方开源仓库 anthropics/skills 里的一个真实技能 frontend-design 做剖面，逐行拆它的设计选择。然后再把 Anthropic 和 OpenAI 两家围绕 Skill 搭建的体系拉出来对照——格式一样，路线不同。
先补一个很多人不知道的背景: Agent Skills 这个格式最早由 Anthropic 发起，作为开放标准发布在 agentskills.io。目前已经被 OpenAI Codex、Cursor、Amp、JetBrains Junie、Google Gemini CLI、GitHub 等 30 多个工具采纳。也就是说，这不是某家的专属功能，而是一个跨平台的通用格式。
![image](images/skills-anthropic-openai/img-01.png)
### 先快速回顾：Skill 解决什么问题
如果你读过我们之前的《Claude Skills 入门：把“会用 AI”变成“可复制的工程能力”》，应该已经知道：MCP 解决的是"AI 能连什么工具"，Skills 解决的是"连上之后怎么把事办成"。
更直白地说，Skill 解决的是**提示词漂移**。
你应该经历过：某个提示词一开始能用，过两周出了问题加一条规则，同事复制到另一个项目里稍微改了改，再过一个月同一个流程在三个地方长出三套变体，没人说得清哪个是"最新版"。
Skill 的核心思路是**把提示词从对话框搬到文件系统**：
一个文件夹，一个 SKILL.md，可以附带参考文档和脚本。
搬到文件系统之后，它就可以被版本控制、被 PR review、被跨项目复用。
在《2026 做 Agent 的正确姿势：单 Agent 起步，Skills 沉淀方法论，MCP 负责连接》里我们聊过，Anthropic 的策略是"单 Agent 起步，Skills 沉淀方法论，MCP 负责连接"。Skills 在这个体系里的角色是**知识层**——不是让模型更聪明，而是让"怎么用好模型"这件事变得可持续。
Skill 能跑得稳，核心靠三个机制：渐进式披露（按需加载，不撑爆上下文）、目录结构即语义（模型擅长沿着文件系统探索）、可附带资源（参考文档、脚本、模板让流程端到端跑起来）。这些在《Skill 到底是什么：从第一性原理深入剖析 Claude Agent Skills》里有详细拆解，这里不再重复。
今天的重点是：打开一个真实的 Skill，看看这些原则落到实处是什么样的。
### 打开 frontend-design：一个 42 行的 SKILL.md
Anthropic 官方维护了一个开源技能仓库（github.com/anthropics/skills），里面有 17 个示例技能，涵盖创意设计、文档生成、MCP 构建等方向。其中一些技能（docx、pdf、pptx、xlsx）直接驱动着 Claude.ai 的文档创建能力——你在 Claude.ai 里生成 PDF 或 PPT 时，背后跑的就是这些 Skill。
我们拿 frontend-design 来展开。它的作用是让 AI 生成高质量的前端界面，避免"一眼看出是 AI 做的"那种模板感。
目录结构极简：
frontend-design/
  SKILL.md
  LICENSE.txt
就两个文件，SKILL.md 全文 42 行。先看元数据：
---
name: frontend-design
description: Create distinctive, production-grade frontend interfaces
  with high design quality. Use this skill when the user asks to build
  web components, pages, artifacts, posters, or applications (examples
  include websites, landing pages, dashboards, React components,
  HTML/CSS layouts, or when styling/beautifying any web UI). Generates
  creative, polished code and UI design that avoids generic AI aesthetics.
license: Complete terms in LICENSE.txt
---
### description 为什么写这么长
前面说过，渐进式披露的机制下，智能体在决定"要不要加载这个技能"时，第一步看的就是 description。它相当于这个技能的"路由规则"。
frontend-design 的 description 做了几件事：
- **划定触发范围**：build web components, pages, dashboards, React components, HTML/CSS layouts...
- **列出具体任务类型**：用括号给出了 examples，消除歧义。
- **写明产出标准**：production-grade, avoids generic AI aesthetics。
如果写成"用于提升前端 UI 质量"——这等于告诉模型"随时可能跟我有关"，结果是该用的时候没用，不该用的时候乱入。好的 description 读起来更像路由规则：精确匹配，不多不少。
在我们之前拆的 Anthropic 官方 33 页指南里也强调过：description 的变更需要重点 review，因为改一个词就可能让技能在错误时机被加载。
### 正文指令做了什么
再看正文部分。它对应的是"AI 生成 UI"最常见的几类失败模式，每一条设计都有针对性。
**第一，强制模型先选一个审美立场，再开始写代码。**
正文里有一个 Design Thinking 环节，要求模型在动手之前先回答四个问题：
- **Purpose**：这个界面解决什么问题？给谁用？
- **Tone**：选一个明确的审美方向——brutally minimal、maximalist chaos、retro-futuristic、organic/natural、luxury/refined、playful/toy-like……
- **Constraints**：技术约束（框架、性能、可访问性）。
- **Differentiation**：什么让这个界面令人难忘？
这些风格选项不是随便列的。UI 生成最常见的失败不是"代码跑不起来"，而是产出像模板站——什么功能都有，但没有任何辨识度。强迫模型先建立一个清晰的创意方向，然后全程保持一致，相当于给模型装了一个"创意方向锁"。
**第二，负面约束写得比正面要求还狠。**
正文里有一整段 NEVER 清单：
NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character.
还有一条很有意思的细节：
NEVER converge on common choices (Space Grotesk, for example) across generations.
这是在禁止模型在多次生成中"塌缩"到同一个安全选择上。每次用这个技能，它都得选一个不同的方向，不能反复用同一套"看起来还行"的模板。
为什么负面约束特别有效？在《别再把 System Prompt 写成巨石》里我们聊过，模型对正向目标的理解往往太宽——"做出好看的 UI"有无数种解读。但"不要用 Inter 字体""不要用紫色渐变"非常具体，边界清晰，模型很容易遵守。这个思路适用于所有类型的 Skill：先想清楚"最常见的失败模式是什么"，然后把它们写成禁区。
**第三，复杂度和目标要匹配。**
它没有要求所有设计都做满特效，而是提了一条很克制的原则：
Match implementation complexity to the aesthetic vision. Maximalist designs need elaborate code with extensive animations and effects. Minimalist or refined designs need restraint, precision, and careful attention to spacing, typography, and subtle details.
这条约束对应的失败模式是"华而不实"——极简风格堆了一堆动效，或者极繁风格细节粗糙。实现复杂度要和审美方向对齐。
**最后一句收尾也值得注意：**
Remember: Claude is capable of extraordinary creative work. Don't hold back, show what can truly be created when thinking outside the box and committing fully to a distinctive vision.
很多时候模型输出平庸，不是因为做不到，是因为它默认在"安全区"里运行。这句话在告诉模型：别保守。
### 小结：42 行背后的设计逻辑
- **description 写得像路由规则**，精确控制加载时机。
- **Design Thinking 环节锁定创意方向**，防止模型滑向模板化。
- **负面约束比正面要求更具体**，用禁区稳定质量。
- **实现复杂度和审美目标对齐**，避免形式和内容脱节。
- **全文只有 42 行**，没有冗余。短，恰恰是它最强的地方。
### 同一个标准，两家搭了不同的体系
Agent Skills 的格式是统一的，但 Anthropic 和 OpenAI 围绕 Skill 搭建的上层体系各有侧重。
### Anthropic（Claude Code）：Skill 是扩展体系的一个模块
在 Claude Code 的设计里，Skill 不是唯一的扩展手段。在《Claude Agent Teams 架构与实战拆解》和《CC核心开发者分享 | 构建 Claude Code 的经验教训：像 Agent 一样看世界》里我们已经聊过 Claude Code 的分层架构，这里做一个完整的梳理。
Claude Code 的扩展层包括：
组件
做什么
加载方式
上下文成本
**CLAUDE.md**
常驻规则，每次会话自动加载
启动时全量加载
每次请求
**Skills**
可复用的知识和工作流
按需，渐进式
低（用到时才加载）
**Hooks**
确定性脚本，在特定事件时自动执行
事件驱动
零
**MCP**
连接外部服务
启动时加载工具定义
每次请求
**Subagents**
隔离上下文的独立工作者
按需启动
与主会话隔离
**Agent teams**
多个独立会话协作
各自独立
各自独立
**Plugins**
打包以上所有东西为可安装单元
安装时展开
取决于内容
这个分层背后有一个很明确的原则。Claude Code 文档里有一句话能概括：
"Hooks run outside the loop entirely as deterministic scripts."
Hooks 在智能体的循环之外运行，是确定性脚本——不经过模型判断，不占上下文，事件触发就执行。比如每次编辑文件后自动跑 lint、每次提交前自动跑类型检查。这些事不需要模型"自觉"去做，hooks 保证 100% 执行。
文档里也给出了明确的选择标准：
- 如果 Claude 每次都应该知道它——放 **CLAUDE.md**。
- 如果 Claude 有时候需要——放 **Skill**。
- 如果必须 100% 执行，不能靠模型判断——放 **hooks**。
- 如果需要外部系统的数据或操作——用 **MCP**。
Anthropic 的思路总结起来就是：**Skill 负责"教模型怎么思考"，hooks 负责"必须做的事"，MCP 负责"连接外部的事"，subagents 负责"上下文快溢出时怎么办"。每种需求都有最合适的机制去承担。**
### OpenAI（Codex）：Skill 本身就是完整的工作流包
Codex 的技能体系围绕另一个思路：**把工作流打包成可分发的独立单元**。
在 Agent Skills 标准的基础上，Codex 增加了一个 agents/openai.yaml 配置文件：
interface:
  display_name: "Optional user-facing name"
  short_description: "Optional user-facing description"
  icon_small: "./assets/small-logo.svg"
  brand_color: "#3B82F6"
  default_prompt: "Optional surrounding prompt"
policy:
  allow_implicit_invocation: false
dependencies:
  tools:
    - type: "mcp"
      value: "openaiDeveloperDocs"
      description: "OpenAI Docs MCP server"
      transport: "streamable_http"
      url: "https://developers.openai.com/mcp"
这个文件做三件事：
1. **UI 元数据**：display_name、图标、品牌色、默认提示词——让技能在 Codex App 界面里有自己的"门面"。
2. **调用策略**：allow_implicit_invocation: false 禁止模型隐式触发，只允许用户手动调用。
3. **依赖声明**：声明这个技能运行需要哪些外部工具。技能可以自描述"我需要什么才能跑起来"。
Codex 还在分发层做了更多设计：
- **多级作用域**：仓库级（.agents/skills）、用户级（$HOME/.agents/skills）、管理员级（/etc/codex/skills）、系统级（内置）四个层级。
- **技能安装器**：$skill-installer 可以从其他仓库安装技能。
- **技能创建器**：$skill-creator 通过对话引导创建新技能。
整个体系在朝"技能生态"的方向走——不只是"我自己用"，而是"我做的技能别人能直接装上用"。
### 差异对照
维度
Claude Code（Anthropic）
OpenAI Codex
**标准关系**
发起了 Agent Skills 开放标准
采纳并扩展了该标准
**Skill 的定位**
扩展体系中的一个模块
可移植的工作流包
**触发控制**disable-model-invocation: trueallow_implicit_invocation: false**确定性动作**
分离到 hooks，不经过模型判断
偏向写进 Skill 的 scripts/
**上下文隔离**
subagents + agent teams
渐进式披露为主
**元数据与依赖**
SKILL.md frontmatter
额外的 agents/openai.yaml
**分发与生态**
plugins + marketplaces
多级作用域 + skill-installer
这两种路线解决的是不同的核心矛盾：
- **Anthropic** 更关心"模型应该管什么、不应该管什么"——把判断留给模型，把确定性动作和上下文隔离交给专门的机制。
- **OpenAI** 更关心"技能怎么流转起来"——让技能可以自描述依赖、跨仓库安装、在 UI 上有门面。
### 有副作用的技能：两家的共识
虽然体系不同，但两家在一个问题上高度一致：**当技能能跑脚本、能写文件、能发消息时，它不再是"写作模板"，它是一个自动化工具。这时候不能让模型隐式触发它。**
Claude Code 的做法是在 SKILL.md 的 frontmatter 里设 disable-model-invocation: true——模型完全看不到这个技能，只有用户手动用 /<name> 调用时才加载。
Codex 的做法是在 agents/openai.yaml 里设 allow_implicit_invocation: false——效果相同。
Claude Code 还走了一步更远：把确定性动作整体分离到了 hooks 层。技能里可以描述"应该跑 lint"，但实际执行由 hook 在事件触发时自动完成——不占上下文，不经过模型判断。
### 回到 frontend-design
回头看这个 42 行的 SKILL.md，它其实是 Agent Skills 设计哲学的一个缩影：
- **渐进式披露**——description 够具体，所以只在正确时机被加载。
- **结构即语义**——虽然它只有单文件，但 YAML + 正文的两层结构已经足够传达元信息和执行指令。
- **短而有立场**——它不试图覆盖所有情况，而是在一个明确的方向上做到极致。
Skill 不会让模型变得更聪明。它解决的是另一个问题：让"怎么用好模型"这件事本身变得可复用、可审查、可持续。
**参考资料**
- Agent Skills 开放标准（Anthropic 发起）：https://agentskills.io
- Anthropic 官方技能仓库（本文案例来源）：https://github.com/anthropics/skills
- Claude Code 扩展体系文档：https://code.claude.com/docs/en/features-overview
- OpenAI Codex Skills 文档：https://developers.openai.com/codex/skills
### 延伸阅读
Anthropic 发布 2026 Agentic Coding 趋势报告：八大趋势与 4 个优先级深度解读
### 深度拆解 Clawdbot（OpenClaw）架构与实现
OpenClaw 背后的秘密武器：极简智能体框架 Pi
NanoBot 架构拆解：4000 行代码实现OpenClaw能力?
深度拆解 OpenClaw 系统提示词：如何更像人?
你可信? OpenClaw+Skills，让 AI 开始在 Moltbook 自主注册、开趴、开会
OpenClaw 是怎么工作的？一条消息的旅程讲清楚
OpenClaw 是怎么工作的（2）：控制面两阶段协议与 runId
OpenClaw 是怎么工作的（3）：会话键与队列策略，怎么把并发关进笼子
