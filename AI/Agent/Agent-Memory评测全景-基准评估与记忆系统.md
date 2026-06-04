# Agent-Memory 评测全景：基准、评估与记忆系统（理论篇）

- **来源：** 微信公众号
- **作者：** 阿元（淘天集团 - 场景智能技术团队）
- **日期：** 2026-06-03
- **原文链接：** https://mp.weixin.qq.com/s/JZhN6auXKOzEh3OHgkjrdw

---

## 引言

随着大语言模型（LLM）在对话系统与智能代理中的应用加深，长期记忆能力正成为影响真实效能的关键因素。尽管LLM擅长短上下文生成，但在多轮、跨会话甚至多模态交互中仍常出现遗忘、推理断裂与一致性缺失。如何构建、更新与检索长期记忆，使模型能持续保留关键信息并适应变化，已成为重要挑战。

近年研究从三条主线推进：一是提出更贴近真实交互的基准与数据集（如MUSE、LOCOMO），二是建立更系统的评估框架（如MemoryAgentBench、LONGMEMEVAL、MemBench），三是探索更有效的记忆方法与系统（如THEANINE、RMM、Mem0，以及面向多模态场景的M3-Agent）。

## 技术概况

### Memory Benchmark

| 来源 | 发表（统计截止2026年2月） |
|------|--------------------------|
| MUSE - Northeastern University | ACL 2025，被引次数5 |
| LOCOMO - University of North Carolina | ACL 2024，被引次数274 |

### Memory Evaluation

| 来源 | 发表（统计截止2026年2月） |
|------|--------------------------|
| MemoryAgentBench - UC San Diego | arxiv，被引次数43 |
| LONGMEMEVAL - UCLA, Tencent | arxiv，被引次数141 |
| MemBench - Huawei | ACL 2025，被引次数23 |

### Memory System

| 来源 | 发表（统计截止2026年2月） |
|------|--------------------------|
| THEANINE&TeaFarm - Yonsei University | NAACL 2025，被引次数23 |
| RMM - Google | ACL 2025，被引次数35 |
| M3-Agent - ByteDance-Seed | ICLR 2026，被引次数29 |
| Mem0 - mem0ai | ECAI 2026，被引次数222 |

---

## Memory Benchmark

### MUSE

《MUSE: A Multimodal Conversational Recommendation Dataset with Scenario-Grounded User Profiles》

**基本信息：**
- 特点：大模型生成对话，基于真实场景和VLM生成的用户画像生成
- 数量：7k个case，8.3w个对话
- 场景：对话推荐数据集，服装领域

**数据集构建：**
- 用户画像生成器：收集多样的真实场景，生成用户画像
- 模拟对话生成器：利用用户画像进行角色扮演，模拟用户与推荐助手之间的对话
- 对话优化器：通过重写和审查机制提升对话的多样性和质量

**数据集质量评估：** 将MUSE与四个数据集（MMCONV、Redial、Inspired和PEARL）比较。采用LLM评估，随机抽取200个对话评估，涵盖对话自然性、逻辑连贯性、信息丰富性、产品上下文相关性和图像文本一致性等五个维度。

### LOCOMO

《Evaluating Very Long-Term Conversational Memory of LLM Agents》

**基本信息：**
- 特点：大模型生成对话，基于个性化角色和时间事件图来构建对话
- 数量：50个对话，每个对话平均300轮、9000个标记
- 场景：评估LLM处理长对话的记忆能力：问题回答、事件总结和多模态对话生成

**数据集构建：**
- 人物设定与时间事件图：获取初始人物设定，利用LLM扩展设定；每个agent构建的时间事件图包含多个事件，通过因果关系相互连接
- 反思与回应机制：每次会话结束，生成总结，存储为短期记忆；每次对话的单个回合，作为观察内容，存储为长期记忆
- 人工验证与编辑：人工矫正长期不一致性、移除不相关图像，并确保对话内容与事件图对齐

**对模型能力的评估：**
- 问答任务：gpt-4-turbo表现最佳（32.4），但仍显著低于人类基准（87.9）
- 事件总结：gpt-3.5-turbo召回率和F1分数最高。五类主要错误包括信息缺失、幻觉、对话线索误解、说话者归属错误以及不重要对话被错误视为重要事件
- 多模态对话生成：包含上下文的训练提升了生成性能
- 总体：LLMs在理解长时间叙述和提取时间及因果关系方面存在困难

---

## Memory Evaluation

### MemoryAgentBench

《Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions》

四个核心能力：
1. **准确检索 (AR)：** 从长对话历史中识别并检索重要信息
2. **测试时学习 (TTL)：** 动态获取新技能，无需额外训练
3. **长程理解 (LRU)：** 在长对话中形成抽象的高层次理解
4. **冲突解决 (CR)：** 面对新旧信息冲突时检测并解决矛盾

评估了三种记忆代理：长上下文代理、RAG代理、代理记忆代理。

**评估结论：**
- RAG方法在准确检索任务中表现优越
- 长上下文模型在TTL和LRU任务中表现最佳
- 所有现有方法在冲突解决（CR）任务上均表现不佳，多跳场景准确率最高仅为6%

### LONGMEMEVAL

《LONGMEMEVAL: BENCHMARKING CHAT ASSISTANTS ON LONG-TERM INTERACTIVE MEMORY》

统一框架优化记忆设计：
- **会话分解：** 将会话分解为多个轮次，提取摘要、关键短语或用户事实
- **事实增强的键扩展：** 通过提取value中的摘要、关键短语、用户事实和时间戳事件来增强键
- **时间感知的查询扩展：** 从文本中提取时间戳事件，从查询中推断时间范围

500个问题，两种配置：LONGMEMEVAL-S（约115k tokens）和LONGMEMEVAL-M（500个会话，约150万tokens）。

**评估结论：** 现有商业聊天助手和长上下文LLM准确率下降30%至60%。

### MemBench

《MemBench: Towards More Comprehensive Evaluation on the Memory of LLM-based Agents》

创新点：
- 包含**事实记忆**和**反思记忆**
- 引入**参与场景**和**观察场景**两种互动场景
- 多指标评估：准确率、召回率、容量和效率

---

## Memory System

### THEANINE & TeaFarm

《Towards Lifelong Dialogue Agents via Timeline-based Memory Management》

- 构建基于时间和因果关系的记忆图，保留重要的上下文信息
- TeaFarm：基于反事实的评估基准，通过设计反事实问题测试代理的记忆引用能力

### RMM

《In Prospect and Retrospect: Reflective Memory Management for Long-term Personalized Dialogue Agents》

- **前瞻性反思：** 将对话历史动态总结为主题基础的记忆表示，优化未来检索
- **回顾性反思：** 利用在线强化学习方法，基于LLMs生成的引用证据迭代精炼检索过程

### M3-Agent

《Seeing, Listening, Remembering, and Reasoning: A Multimodal Agent with Long-Term Memory》
- GitHub: https://github.com/ByteDance-Seed/m3-agent

- 具备长期记忆的多模态智能体框架，实时处理视觉和听觉输入
- 记忆以图形结构存储：情节记忆记录具体事件，语义记忆提取一般知识
- 通过强化学习优化，自主决定调用哪种搜索功能
- M3-Bench：100个真实世界视频 + 929个网络视频
- 实验结果：M3-Agent在M3-Bench-robot、M3-Bench-web和VideoMME-long上准确率分别提高6.7%、7.7%和5.3%

### Mem0

《Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory》
- GitHub: https://github.com/mem0ai/mem0

**Mem0记忆架构：**
- 动态捕获、组织和检索持续对话中的显著信息
- 提取阶段：接收消息对，利用对话摘要和最近消息建立上下文，用LLM提取重点记忆
- 更新阶段：评估候选事实与现有记忆的一致性，进行添加、更新、删除或无操作

**Mem0g：**
- 在Mem0基础上引入图形记忆表示
- 节点代表实体，边表示实体之间的关系
- 实体提取模块 + 关系生成模块

**实验结果：**
- Mem0在单跳和多跳推理任务中表现出色
- Mem0g在时间推理和开放域任务中表现出色
- 在响应延迟和计算效率方面显著优于全上下文方法

---

## 总结与讨论

当Agent从单轮对话走向长程任务与跨会话交互，Memory从"加分项"变成决定体验与能力上限的关键组件。

现有评测的共性问题：
1. 增益难归因（记忆、长上下文、RAG常叠加）
2. 口径不统一，易"命中但无用"，指标与端到端收益脱钩
3. 动态更新与遗忘覆盖不足，缺少长期压力测试
4. 成本与约束缺位（时延、token/调用、存储、隐私合规）

面向真实应用的评测应同时覆盖四个维度：检索正确性、使用有效性、时间维度（跨会话/变化/遗忘）、成本维度（延迟/费用/存储/合规）。
