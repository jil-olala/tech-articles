---
title: 大家都在讲Harness，但它到底该怎么理解
source: 微信公众号（架构师 JiaGouX）
author: 若飞（整理）
date: 2026-03-29
original_link: https://mp.weixin.qq.com/s/tTjhHOrCslCW7vG14WFn3A
tags: [AI, Agent, Harness Engineering, OpenAI, Anthropic, 架构]
---

# 大家都在讲 Harness，但它到底该怎么理解

## 核心观点

如果一个概念什么都能装，它通常也就什么都没讲清。本文把 Harness 从热词拆回一个有边界、有层次的工程对象。

结合 OpenAI 和 Anthropic 两篇文章，拼出了 Harness 的不同切面。

## 一组对比数据说明一切

Anthropic 实验：同一句 Prompt"做一个 2D 复古游戏编辑器"，同一个模型跑两次。

| | 单 Agent | 完整 Harness |
|---|---|---|
| 时间 | 20 分钟 | 6 小时 |
| 成本 | $9 | $200 |
| 结果 | 界面有了，核心功能全坏 | 精灵动画、行为模板、音效、AI关卡生成、导出分享全部可用 |

**模型没换，Prompt 没换。变的只是它跑在什么系统里。**

## Harness 三层结构

### 第一层：知识层

解决：模型该读什么、团队经验放哪里、哪些规则必须显式可见。

OpenAI 的经验：
- AGENTS.md 不是百科全书，是内容目录（约100行）
- 真正知识库放在结构化 docs/ 目录
- 代码仓库本身是记录系统
- **对 Agent 来说，看不见的知识就等于不存在**
- 跑"doc-gardening"智能体扫描过时文档，自动发起修复 PR

### 第二层：约束与流程层

解决：任务怎么拆、哪些步骤必须先发生、哪个角色负责什么。

Anthropic 的做法：
- **Planner / Generator / Evaluator 三角色分离**
- Planner 只聚焦产品上下文和高层设计，不规定技术细节（避免错误级联）
- 每个 Sprint 开始前，Generator 和 Evaluator 先谈好"契约"（27条验收标准）
- **不是为并行更快，而是为了别一边生成一边自我表扬**
- context reset：定期清空上下文，通过结构化工件交接状态

OpenAI 的做法：
- 把约束编码成机械规则
- 业务域依赖方向固定为：Types → Config → Repo → Service → Runtime → UI
- **对人类可能迂腐，对 Agent 是倍增器**

### 第三层：反馈与运行时层

解决：模型做完之后，谁来告诉它对不对。

Anthropic：
- Evaluator 不是读代码打分，是用 Playwright 真正跑页面、点按钮
- 开箱即用的 Claude 是很差的 QA Agent——发现问题后自己说服自己不是大事，通过验收
- 花了好几轮迭代才把评估质量调到合理水平

OpenAI：
- 应用按 git worktree 启动，每次改动独立验证
- Chrome DevTools 协议接入 Agent 运行时
- 本地可观测性堆栈，Agent 直接查日志和指标
- 单次 Codex 运行持续超过6小时（通常在人类睡觉时）

## Harness 是长出来的，不是设计出来的

Mitchell Hashimoto（HashiCorp联创）：每次 Agent 犯一个错，就把修复方式写进配置文件。Ghostty 终端模拟器的 Harness 就是这么长出来的。

清华+哈工大论文提出 NLAH（Natural-Language Agent Harness）：把控制逻辑抽成接近自然语言但仍可执行的文件，配套 IHR（Intelligent Harness Runtime）。

**Harness 的起点不是"画架构图"，而是"从第一个犯错开始补"。**

## Harness 会随模型变强而变轻，但不会消失

Anthropic 提醒：Harness 每个组件都编码了对模型局限性的假设。模型变强后，有些组件不再承重。

但评估器去不掉——"自己评价自己"的能力提升速度，远跟不上生成能力的提升。

**一边拆掉不再承重的组件，一边在新边界处补新的约束和反馈。**

## Prompt、Context、Harness：包含关系，不是替代关系

| 层 | 核心问题 | 关注点 |
|---|---|---|
| Prompt | 你怎么说 | 措辞、格式、few-shot、CoT |
| Context | 模型看到什么 | RAG、长上下文、工具定义、动态注入 |
| Harness | 系统怎么运行 | 知识挂载、约束生效、反馈接回、经验沉淀 |

**不是让模型更聪明，而是让系统把"对错"变成机器可以执行的判断标准。**

## 参考材料

- OpenAI Codex 团队《Harness Engineering》
  https://openai.com/zh-Hans-CN/index/harness-engineering/
- Anthropic《Harness design for long-running application development》
  https://www.anthropic.com/engineering/harness-design-long-running-apps
- Mitchell Hashimoto《My AI Adoption Journey》
- 清华+哈工大《Natural-Language Agent Harnesses》
