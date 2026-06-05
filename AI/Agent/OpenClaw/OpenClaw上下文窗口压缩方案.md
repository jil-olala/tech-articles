# 深入解析OpenClaw上下文窗口压缩方案：一切都是为了效果与省钱

- **来源：** 微信公众号
- **作者：** 腾讯云开发者
- **日期：** 2026-03-04
- **原文链接：** https://mp.weixin.qq.com/s/CwnJ9jK4_g9Sd8jxaOddQA

---

关注腾讯云开发者，一手技术干货提前解锁👇
最近很火的 OpenClaw 的出镜率是越来越高了，内外网的技术文章，新产品的问世，Mac Mini 的涨价，自媒体的宣传层出不穷。作者是国外一个叫 Peter Steinberger（现已经被奥特曼高薪挖到 OpenAI ），说是花了一个周末就烹饪完成的“小龙虾”。 随着司内 OpenClaw 也支持了，我也在企微中增加了对应的机器人，给它取名「靓靓蒸虾🦞」，协助处理一些需求管理上的事情。我一直也在做上下文相关的事情，现在我们就拨开虾外壳，看看它内部详细是如何调味的（进行上下文窗口管理的）。
![image](images/openclaw-context-compression/img-01.pngby7w5ZabXeyBXsqTJzMzibMZdn78wqyMBYiahTQ4UHPcHibEml8AgM/640?wx_fmt=png&from=appmsg)
仓库: github.com/openclaw/openclaw 本文基于该仓库源码进行分析。
在 AI Agent 的长会话场景中，上下文窗口溢出是一个绕不开的问题。当对话越来越长、工具调用结果越来越多，LLM 的上下文窗口终将被填满。OpenClaw 为此设计了一套多层防御系统，从"尽量不溢出"到"溢出了也能恢复"，覆盖了整个上下文生命周期。本文将从架构到实现细节，完整剖析这套方案。
## 01
## 整体架构
OpenClaw 的上下文管理分为三个阶段，形成递进的防御纵深：
```
┌────────────────────────┐  ┌─────────────────────┐  ┌──────────────────────────┐
│   LLM 调用前（预防）     │  │  LLM 调用中（自动）   │  │   LLM 调用后（溢出恢复）   │
│                        │  │                     │  │                          │
│ 1. History Turn Limit  │  │ 3. SDK 自动          │  │ 5. Overflow 错误检测      │
│ 2. Context Pruning     │  │    Compaction        │  │ 6. 显式 Compaction       │
│    (Tool Result 裁剪)  │  │                     │  │ 7. 超大 Tool Result 截断  │
│                        │  │                     │  │ 8. 重试 / 放弃           │
└────────────────────────┘  └─────────────────────┘  └──────────────────────────┘
```
一个关键的设计原则是渐进式降级：先做轻量级裁剪（只丢弃冗余数据），再尝试 LLM 摘要（有损但保留语义），最后才是暴力截断（只保留头部内容）。每一层只在前一层不够用时才会介入。
## 02
第一层：预防性裁剪（发送 LLM 之前）
这一层的目标是：在消息发送给 LLM 之前，尽可能裁剪掉不再需要的冗余内容，避免触发上下文溢出。
   2.1 会话历史轮次限制（History Turn Limit）
文件：src/agents/pi-embedded-runner/history.ts
这是最简单、最粗粒度的保护——直接限制保留的用户对话轮次数。
工作原理
limitHistoryTurns() 函数从消息列表的末尾向前遍历，计数 role === "user" 的消息。当计数超过 limit 时，丢弃更早的所有消息：
```
export function limitHistoryTurns(
  messages: AgentMessage[],
  limit: number | undefined,
): AgentMessage[] {
  if (!limit || limit = 0; i--) {
    if (messages[i].role === "user") {
      userCount++;
      if (userCount > limit) {
        return messages.slice(lastUserIndex);
      }
      lastUserIndex = i;
    }
  }
  return messages;
}
```
注意这里的截断边界是 lastUserIndex——即被计数的最后一个 user 消息的位置。这意味着截断点始终在一个完整的 user 轮次边界上，不会把一个 user-assistant-toolResult 的三元组截断成碎片。
配置解析
限制数值通过 getHistoryLimitFromSessionKey() 从配置中解析，支持多级覆盖：
```
per-DM 用户级别覆盖
  → channels.*.dms[userId].historyLimit
provider 级别 DM 默认
  → channels.*.dmHistoryLimit
provider 级别群组/频道默认
  → channels.*.historyLimit
```
这个分层设计使得运营者可以为特定的高频用户设置更严格的限制，同时保持其他用户的默认行为。session key 的 kind 字段（dm/direct/channel/group）决定走哪条解析路径。
   2.2 Context Pruning 扩展（渐进式裁剪旧 Tool Results）
文件：src/agents/pi-extensions/context-pruning/
如果说 History Turn Limit 是"剪头"，Context Pruning 就是"瘦身"。它是一个运行时扩展（extension），注册在 context 事件上，在每次构造 LLM 请求时拦截消息列表。
触发机制
扩展采用 cache-ttl 模式运行，默认 TTL 为 5 分钟。这意味着：
- 上一次 pruning 之后 5 分钟内，不会再次触发（避免频繁裁剪）
- 超过 TTL 后的第一次 context 事件会触发裁剪评估
```
// extension.ts
if (runtime.settings.mode === "cache-ttl") {
  const ttlMs = runtime.settings.ttlMs;
  const lastTouch = runtime.lastCacheTouchAt ?? null;
  if (!lastTouch || ttlMs  0 && Date.now() - lastTouch
两级裁剪策略
核心逻辑在 pruneContextMessages() 中，使用字符占比作为触发条件：
阶段
触发条件
行为
效果
**Soft Trim**totalChars / charWindow > 0.3
对超过 4000 字符的旧 tool result，保留首 1500 + 尾 1500 字符，中间用 ... 替代
保留关键信息（开头通常是命令/文件头，结尾通常是结论/错误），丢弃中间冗余
**Hard Clear**totalChars / charWindow > 0.5
将整个 tool result 替换为 "[Old tool result content cleared]"
彻底释放空间，仅保留"这里曾经有一个工具调用"的标记
其中 charWindow = contextWindowTokens × 4（使用 1 token ≈ 4 字符的粗略估算）。
Soft Trim 的截断实现很精巧——它不是简单地 slice，而是在文本块（text content block）的层面操作，分别从头部和尾部取字符，保持换行符的完整性：
```
function softTrimToolResultMessage(params) {
  const parts = collectTextSegments(msg.content);
  const rawLen = estimateJoinedTextLength(parts);
  if (rawLen
裁剪范围与保护规则
并非所有消息都会被裁剪，有几条硬性保护规则：
- 第一条 user 消息之前的内容永远不裁剪。因为会话开头通常包含身份文件的读取（如 SOUL.md、USER.md 等），这些 tool result 是 AI 理解用户身份的基础。
- 最近 N 条 assistant 消息关联的 tool result 不裁剪（默认 N=3）。通过 findAssistantCutoffIndex() 从末尾向前数 3 个 assistant 消息，这些消息之后的 tool result 被标记为受保护区域。
- 含图片的 tool result 不裁剪。图片无法被部分截断且通常直接与用户需求相关。
- 可配置的工具白/黑名单。通过 tools.allow / tools.deny 配置，可以精确控制哪些工具的结果可以被裁剪。
Hard Clear 还有一个额外的安全阈值：只有当可裁剪的工具结果总量超过 minPrunableToolChars（默认 50,000 字符）时才会执行，避免对少量工具结果做无意义的清理。
默认参数一览
```
// settings.ts
{
  mode: "cache-ttl",
  ttlMs: 5 * 60 * 1000,          // 5 分钟缓存 TTL
  keepLastAssistants: 3,          // 保护最近 3 条 assistant 消息的 tool results
  softTrimRatio: 0.3,             // 总字符占比 > 30% 时触发 soft trim
  hardClearRatio: 0.5,            // 总字符占比 > 50% 时触发 hard clear
  minPrunableToolChars: 50_000,   // hard clear 前需要可裁剪 tool result ≥ 50K 字符
  softTrim: {
    maxChars: 4_000,              // tool result 超过 4K 字符才触发 soft trim
    headChars: 1_500,             // 保留首 1500 字符
    tailChars: 1_500,             // 保留尾 1500 字符
  },
  hardClear: {
    enabled: true,
    placeholder: "[Old tool result content cleared]",
  },
}
```
   2.3 单条 Tool Result 截断
文件：src/agents/pi-embedded-runner/tool-result-truncation.ts
Context Pruning 处理的是"很多旧的 tool result 累加起来太大"的情况，而单条截断处理的是另一种场景——单条工具返回了巨量内容（比如读取了一个大文件或执行了一个产生大量输出的命令）。
大小限制
```
MAX_TOOL_RESULT_CONTEXT_SHARE = 0.3   // 单条 tool result 最多占上下文窗口的 30%
HARD_MAX_TOOL_RESULT_CHARS = 400_000  // 绝对上限 400K 字符（约 100K tokens）
MIN_KEEP_CHARS = 2_000                // 截断时至少保留前 2000 字符
```
最终限制取两者中的较小值：
```
export function calculateMaxToolResultChars(contextWindowTokens: number): number {
  const maxTokens = Math.floor(contextWindowTokens * MAX_TOOL_RESULT_CONTEXT_SHARE);
  const maxChars = maxTokens * 4;  // 1 token ≈ 4 chars 估算
  return Math.min(maxChars, HARD_MAX_TOOL_RESULT_CHARS);
}
```
对于一个 200K token 上下文窗口的模型，单条 tool result 最大约为 200K × 0.3 × 4 = 240K 字符。对于 2M 上下文的模型，理论值 2.4M 会被 HARD_MAX_TOOL_RESULT_CHARS 限制到 400K。
截断策略
截断时保留内容的头部（开头通常包含最重要的信息），并尽量在换行符处断开以避免切断行中间：
```
export function truncateToolResultText(text, maxChars, options) {
  const keepChars = Math.max(minKeepChars, maxChars - suffix.length);
  let cutPoint = keepChars;
  // 尝试在 80% 范围内找到最近的换行符
  const lastNewline = text.lastIndexOf("\n", keepChars);
  if (lastNewline > keepChars * 0.8) {
    cutPoint = lastNewline;
  }
  return text.slice(0, cutPoint) + suffix;
}
```
截断后追加一段提示信息，引导模型通过 offset/limit 等参数获取更多内容：
```
⚠️ [Content truncated — original was too large for the model's context window.
The content above is a partial view. If you need more, request specific sections
or use offset/limit parameters to read smaller chunks.]
```
多文本块的比例分配
一个 tool result 可能包含多个 text content block。此时截断预算会按各 block 的原始长度比例分配：
```
// truncateToolResultMessage() 中
const blockShare = textBlock.text.length / totalTextChars;
const blockBudget = Math.max(
  minKeepChars + suffix.length,
  Math.floor(maxChars * blockShare),
);
```
两种截断模式
模式
函数
场景
持久化
内存级
truncateOversizedToolResultsInMessages()
LLM 调用前的预防性守卫
不修改 session 文件
持久级
truncateOversizedToolResultsInSession()
溢出恢复时的最后手段
修改 session 文件（通过分支重写）
持久级截断的实现很有意思——它通过 Session Manager 的分支机制（branching）来修改历史：
- 找到第一个超大 tool result 的 entry
- 从该 entry 的父节点处创建新分支
- 从该位置开始，依次重新追加所有后续 entry，对超大 tool result 做截断处理
- 新分支成为当前活动分支
这种方式避免了直接修改已有的 entry，保持了 session 文件的 append-only 语义。
## 03
第二层：Compaction（基于 LLM 的主动压缩）
这是 OpenClaw 最核心的上下文压缩机制——用另一次 LLM 调用来生成对话历史的摘要，用摘要替代原始消息。
核心文件：
- src/agents/compaction.ts — 摘要算法
- src/agents/pi-extensions/compaction-safeguard.ts — compaction 协调扩展
- src/agents/pi-embedded-runner/compact.ts — compaction 入口
   3.1 触发时机
Compaction 在两种场景下触发：
- SDK 自动触发：当 pi-coding-agent SDK 检测到上下文接近窗口上限时，自动触发 session_before_compact 事件
- 溢出后显式触发：LLM 返回上下文溢出错误后，以 trigger: "overflow" 强制执行
   3.2 Compaction Safeguard 协调流程
compaction-safeguard 扩展监听 session_before_compact 事件，协调整个 compaction 流程。它不只是简单地做摘要，而是一个完整的信息保留+压缩 pipeline：
```
session_before_compact 事件触发
    │
    ├── 1. 前置检查
    │     ├── 解析 model（ctx.model → fallback runtime.model）
    │     └── 获取 API key → 任一缺失则 cancel，保留原始历史
    │
    ├── 2. 收集元数据
    │     ├── 文件操作记录：read/edited/written → readFiles + modifiedFiles
    │     └── 工具失败记录：最多 8 条，每条 ≤ 240 字符
    │
    ├── 3. 历史预裁剪（可选）
    │     └── 当新内容 > maxHistoryShare(50%) 的上下文窗口时
    │           ├── 丢弃最老的 chunk
    │           ├── 修复 tool_use/tool_result 配对
    │           └── 对被丢弃消息单独做摘要 → droppedSummary
    │
    ├── 4. 分段摘要（summarizeInStages）
    │     └── messagesToSummarize → chunks → per-chunk 摘要 → 合并
    │
    ├── 5. Split Turn 处理（可选）
    │     └── 如果是分裂轮次，额外摘要 turnPrefixMessages
    │
    └── 6. 组装最终 summary
          ├── 对话摘要文本
          ├── Tool Failures 列表
          ├──  /  文件操作
          └──  AGENTS.md 关键规则
```
摘要失败保护
整个流程用 try-catch 包裹，任何异常都会导致 { cancel: true }——取消 compaction，保留原始历史。这是一个重要的设计决策：宁可让上下文溢出（进入溢出恢复流程），也不要因为摘要失败而丢失历史。
   3.3 摘要生成算法详解
分段摘要（summarizeInStages）
当消息量较大时，不能一次性把所有消息送给 LLM 做摘要（摘要请求自身也有上下文窗口限制）。summarizeInStages 采用分而治之的策略：
```
消息列表
  → splitMessagesByTokenShare (按 token 均分为 N 个 chunk)
  → 逐 chunk 调用 summarizeWithFallback 生成摘要
  → 如果产生多个 partial summaries
      → 将各摘要作为 user 消息送入 LLM 做合并摘要
      → 使用 MERGE_SUMMARIES_INSTRUCTIONS 指导合并
```
关键参数：
```
const DEFAULT_PARTS = 2;                     // 默认分 2 段
const BASE_CHUNK_RATIO = 0.4;               // 每个 chunk 最多占上下文窗口的 40%
const MIN_CHUNK_RATIO = 0.15;               // 最小 15%
const SAFETY_MARGIN = 1.2;                  // 20% 安全裕量
const SUMMARIZATION_OVERHEAD_TOKENS = 4096;  // 预留 4096 tokens 给摘要 prompt 本身
```
splitMessagesByTokenShare 的分割算法按 token 总量均分，确保每个 chunk 的 token 数接近 totalTokens / parts。分割点在消息边界上，不会切断单条消息。
自适应 chunk 大小
如果消息平均体积很大（比如用户频繁读取大文件），固定的 chunk 比例可能仍然导致单个 chunk 溢出摘要模型。computeAdaptiveChunkRatio 会动态缩小 chunk 比例：
```
export function computeAdaptiveChunkRatio(messages, contextWindow) {
  const avgTokens = totalTokens / messages.length;
  const safeAvgTokens = avgTokens * SAFETY_MARGIN;  // 乘以 1.2 安全系数
  const avgRatio = safeAvgTokens / contextWindow;
  // 当平均消息大小 > 上下文窗口的 10% 时，开始缩小
  if (avgRatio > 0.1) {
    const reduction = Math.min(avgRatio * 2, BASE_CHUNK_RATIO - MIN_CHUNK_RATIO);
    return Math.max(MIN_CHUNK_RATIO, BASE_CHUNK_RATIO - reduction);
  }
  return BASE_CHUNK_RATIO;  // 默认 0.4
}
```
举例：如果平均消息占上下文窗口的 20%（avgRatio = 0.2），那么 reduction = 0.4，chunk ratio 被缩小到 MIN_CHUNK_RATIO = 0.15。每个 chunk 只会放 15% 上下文窗口大小的内容。
超大消息的三级降级（summarizeWithFallback）
对于包含极大单条消息的情况，摘要可能直接失败。summarizeWithFallback 实现了三级降级：
```
Level 1: 尝试全量摘要
  ↓ 失败（超时/溢出/API 错误）
Level 2: 剔除超大消息（单条 > 50% 上下文窗口），只摘要剩余小消息
         + 追加 "[Large assistant (~150K tokens) omitted from summary]" 标注
  ↓ 仍然失败
Level 3: 返回兜底文本
         "Context contained N messages (M oversized). Summary unavailable due to size limits."
```
每一级都能产出一个结果，不会让 compaction 因为单条超大消息而完全中断。
摘要调用的容错
每个 chunk 的摘要调用通过 retryAsync 封装，具有内建重试机制：
```
summary = await retryAsync(
  () => generateSummary(chunk, model, reserveTokens, apiKey, signal, customInstructions, summary),
  {
    attempts: 3,
    minDelayMs: 500,
    maxDelayMs: 5000,
    jitter: 0.2,
    label: "compaction/generateSummary",
    shouldRetry: (err) => !(err instanceof Error && err.name === "AbortError"),
  },
);
```
最多重试 3 次，退避延迟从 500ms 到 5000ms，附加 20% 的抖动避免雷同重试。只有 AbortError（用户主动取消）不重试。
   3.4 历史裁剪预处理（pruneHistoryForContextShare）
在开始摘要之前，如果待摘要的消息总量太大，需要先做一轮预裁剪。这发生在新内容（摘要后需要保留的部分）已经占用了超过 maxHistoryShare（默认 50%）的上下文窗口时。
裁剪算法：
```
1. 计算 budgetTokens = contextWindowTokens × maxHistoryShare
2. while (消息总 token > budgetTokens):
     a. 将消息按 token 均分为 N 段（默认 2）
     b. 丢弃第一个（最老的）chunk
     c. 调用 repairToolUseResultPairing 修复孤立的 tool_result
     d. 统计丢弃量
3. 对所有被丢弃的消息做单独摘要 → droppedSummary
4. droppedSummary 作为 previousSummary 传递给后续的主摘要流程
```
这里的 repairToolUseResultPairing 至关重要——丢弃消息后，可能出现 tool_result 的对应 tool_use（在 assistant 消息中）已被丢弃的情况。Anthropic 的 API 会严格检查配对关系，孤立的 tool_result 会导致 "unexpected tool_use_id" 错误。修复函数会：
- 将匹配的 tool_result 移动到紧跟其对应 tool_use 之后
- 丢弃孤立的 tool_result（对应 tool_use 不存在）
- 为缺失结果的 tool_use 插入合成的错误 tool_result
- 去除重复的 tool_result
   3.5 Compaction Summary 的结构化输出
最终生成的 summary 不仅仅是对话摘要。OpenClaw 在摘要文本后附加了结构化信息，确保 compaction 后 AI 仍然知道"自己做过什么"：
```
[对话摘要文本]
## Tool Failures                          ← 工具失败记录（最多 8 条）
- bash (exitCode=1): command not found...
- read_file (status=error): file too large
                              ← 已读文件列表
src/foo.ts
src/bar.ts
                          ← 已修改文件列表
src/baz.ts
                ← AGENTS.md 中的关键规则
...Session Startup / Red Lines 内容...     （限制 ≤ 2000 字符）
```
这些附加信息的意义：
- Tool Failures：让 AI 知道之前哪些工具调用失败了，避免重蹈覆辙
- File Lists：让 AI 知道自己读过和修改过哪些文件，维持工作上下文的连续性
- Workspace Rules：将 AGENTS.md 中的关键规则注入 summary，确保 compaction 不会导致 AI "忘记"项目的核心约束
   3.6 安全保护机制
Compaction 涉及将对话历史送入 LLM 处理，有专门的安全考虑：
保护机制
说明
stripToolResultDetails()
永远不将 toolResult.details 送入 LLM 做摘要。details 可能包含不可信的 payload（如外部 API 返回的原始数据），防止 prompt injection
repairToolUseResultPairing()
丢弃消息后修复孤立/重复的 tool_result，防止 Anthropic API 拒绝请求
EMBEDDED_COMPACTION_TIMEOUT_MS
Compaction 超时保护，防止摘要调用无限挂起
会话写锁
Compaction 期间通过 acquireSessionWriteLock() 获取写锁，防止并发写入导致 session 文件损坏
摘要失败 → 取消
API key 缺失或摘要过程抛异常时，返回 { cancel: true } 保留原始历史
## 04
第三层：溢出后恢复
文件：src/agents/pi-embedded-runner/run.ts
即使有了预防性裁剪和主动压缩，仍然可能出现上下文溢出——比如模型的实际 token 计数与估算的 chars/4 启发式有较大偏差，或者 SDK 自动 compaction 后上下文仍然超限。
溢出检测
通过 isLikelyContextOverflowError() 检测 LLM 返回的错误是否为上下文溢出。检测逻辑覆盖两个来源：
- promptError：prompt 提交阶段就被拒绝（provider 的 API 直接返回 413 或类似错误）
- assistantError：LLM 开始生成但在过程中报告上下文溢出（stopReason === "error"）
恢复决策树
```
检测到 context overflow 错误
    │
    ├── 分支 A: 本次 attempt 内 SDK 已自动 compaction？
    │     └── 是 → 增加 overflowCompactionAttempts 计数
    │           └── 直接重试 prompt（不再额外 compact，避免重复压缩）
    │
    ├── 分支 B: 本次 attempt 内无 auto-compact？
    │     └── overflowCompactionAttempts
恢复约束
- Compaction 最多 3 次尝试（MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3）。每次 compaction 都涉及额外的 LLM 调用，需要避免无限循环。
- Tool Result 截断只尝试一次（toolResultTruncationAttempted 标志位）。因为持久级截断修改了 session 文件，重复执行没有意义。
- 全局迭代上限: 整个 run loop 有 32-160 次的迭代上限（取决于 auth profile 数量），防止各种重试机制叠加导致无限循环。
- Compaction 失败检测: 如果错误本身就是 compactionFailureError（说明 compaction 自身因溢出而失败），则直接跳过再次 compaction，进入 fallback。
## 05
Token 估算策略
OpenClaw 使用 chars / 4 的启发式方法估算 token 数量（即 1 token ≈ 4 字符），这是一个有意为之的简化：
- 不依赖具体 tokenizer: 适用于所有 LLM provider（Anthropic、OpenAI、Google 等）
- 已知偏差: 对多字节字符（中文、日文等）会低估，对代码 token 也可能有偏差
- 补偿机制: 通过 SAFETY_MARGIN = 1.2 乘以 20% 的安全系数来弥补
在 chunkMessagesByMaxTokens 中：
```
const effectiveMax = Math.max(1, Math.floor(maxTokens / SAFETY_MARGIN));
```
在 compaction-safeguard 中，计算历史裁剪阈值时也会应用安全系数：
```
const maxHistoryTokens = Math.floor(contextWindowTokens * maxHistoryShare * SAFETY_MARGIN);
```
此外，stripToolResultDetails() 在估算前移除 toolResult.details，避免不可信的大体积附加数据干扰估算和摘要。
## 06
配置项汇总
配置路径
说明
默认值
agents.defaults.contextTokens
上下文窗口上限覆盖
模型默认值
agents.defaults.compaction.reserveTokens
compaction 后为新回复保留的 token
20,000
agents.defaults.compaction.reserveTokensFloor
reserveTokens 下限
20,000
agents.defaults.compaction.keepRecentTokens
保留最近消息的 token 数
SDK 默认
agents.defaults.contextPruning.mode
pruning 模式
"cache-ttl"agents.defaults.contextPruning.ttl
缓存 TTL
"5m"agents.defaults.contextPruning.keepLastAssistants
保护最近 N 条 assistant 消息
3
agents.defaults.contextPruning.softTrimRatio
soft trim 触发阈值
0.3
agents.defaults.contextPruning.hardClearRatio
hard clear 触发阈值
0.5
channels.*.dmHistoryLimit
DM 会话历史轮次限制
无限制
channels.*.historyLimit
群组会话历史轮次限制
无限制
## 07
全景流程图
```
                    ┌─────────────────────────────────┐
                    │   用户消息进入会话                 │
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │  History Turn Limit              │  ← 最简粗粒度截断
                    │  (只保留最近 N 轮用户对话)         │
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │  Context Pruning (soft/hard)     │  ← 渐进式裁剪旧 tool results
                    │  - Soft: 保留 head+tail          │     5 分钟 TTL 节流
                    │  - Hard: 替换为占位符             │     ratio 阈值 0.3/0.5
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │  单条 Tool Result 截断            │  ← 内存级预防守卫
                    │  (单条 ≤ 30% 上下文窗口)          │     ≤ 400K 字符硬顶
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │  发送给 LLM                      │
                    └────────────┬────────────────────┘
                                 │
                         成功 ◄──┤──► 溢出错误
                                 │
                    ┌────────────▼────────────────────┐
                    │  Compaction（LLM 生成摘要）       │
                    │  1. 自适应 chunk 大小             │
                    │  2. 分段摘要 + 摘要合并           │
                    │  3. 超大消息三级降级              │
                    │  4. 附加文件操作 + 工具失败信息    │
                    │  5. 附加工作区关键规则             │
                    │  6. 修复 tool_use/result 配对    │
                    └────────────┬────────────────────┘
                                 │
                         成功 ◄──┤──► 仍然溢出？
                                 │
                    ┌────────────▼────────────────────┐
                    │  截断超大 Tool Results            │  ← 最后手段
                    │  (持久级修改 session 文件)         │     通过 branching 重写
                    └────────────┬────────────────────┘
                                 │
                          成功 ◄─┤─► 放弃（提示用户 /reset）
```
## 08
核心设计思路
- 渐进式降级: 从轻量裁剪 → LLM 摘要 → 暴力截断 → 放弃，逐级升级。每一层都是对前一层不够用时的兜底，避免"一刀切"导致信息过度损失。
- 保护关键信息: 在每一个环节都有明确的保护规则——最近的对话不动、身份文件不动、文件操作记录注入摘要、工具失败信息随 summary 传递、workspace 规则注入 summary。即使经过 compaction，AI 仍然知道"我是谁、我做了什么、有哪些规则要遵守"。
- 自适应: chunk 大小根据消息平均体积动态调整，token 估算乘以安全系数，pruning 阈值基于比例而非绝对值。这使得同一套逻辑能适应从 8K 到 2M 各种大小的上下文窗口。
- 安全优先: 不可信数据（toolResult.details）永远不进入摘要 prompt，防止 prompt injection；tool_use/result 配对修复防止 API 报错；摘要失败时取消而不是丢弃历史。
- 可恢复性: 溢出不是终点——自动重试、显式 compaction、tool result 截断，多种手段递进尝试。只有在所有手段都用尽后，才给用户一个明确的恢复建议（/reset）。
- 最小侵入: session 文件修改通过 branching 实现（append-only），内存级操作不持久化，写操作有排他锁保护。整个系统对 session 数据的修改是谨慎且可追溯的。
## 09
附录：上下文管理对 Provider KV Cache 的影响分析
上下文管理方案不可避免地会改变发送给 LLM 的消息序列。而主流 LLM Provider（Anthropic、OpenAI、Google 等）都提供了 Prompt Caching 机制——如果新请求的 prompt 前缀与前一次请求相同，Provider 可以复用已有的 KV Cache，大幅降低延迟和计费。
以 Anthropic 为例：cache read 价格仅为普通 input 的 10%，cache write 则为 125%。一次 cache miss 可能导致成本翻倍。
   9.1 OpenClaw 对 Provider Cache 的感知
OpenClaw 明确知晓并利用了 Provider 的 Prompt Caching 能力：
- Cache Retention 配置: 通过 cacheRetention 参数（"short" = 5 min / "long" = 1 hour）向 Anthropic 声明缓存保留策略。直接 Anthropic 调用默认为 "short"。
- System Prompt Cache Control: 对 OpenRouter 的 Anthropic 模型，通过 createOpenRouterSystemCacheWrapper 主动在 system message 上注入 cache_control: { type: "ephemeral" }，确保系统提示被缓存。
- Usage 追踪: UsageAccumulator 明确追踪 cacheRead cacheWrite lastCacheRead / lastCacheWrite，可以观察每次调用的 cache 命中情况。
- Cache TTL 感知的 Pruning: Context Pruning 扩展通过 isCacheTtlEligibleProvider() 检查 Provider 是否支持 cache TTL，仅对支持的 Provider（Anthropic、Moonshot、ZAI 等）启用 pruning。
   9.2 各层操作对 KV Cache 的影响
1. History Turn Limit — 对 cache 无直接影响
这个操作只会在首次构建消息列表时截断最老的轮次。由于每次 LLM 调用的消息列表都是从 session 文件重建的，截断行为在每次请求间是一致的。只要 limit 不变，每次调用的 prompt 前缀是稳定的，不会导致 cache miss。
但如果 limit 触发了截断（消息数超过限制），被截断的那一次请求的 prompt 前缀会与前一次完全不同——这一次一定是 cache miss。不过这通常只发生在长时间运行的会话中。
2. Context Pruning — 会导致 cache 失效，但有刻意的缓解设计
这是 cache 影响最大的操作。Soft Trim 和 Hard Clear 会修改旧 tool result 的内容，改变 prompt 中间的文本。由于 KV Cache 是严格前缀匹配的，一旦修改了 prompt 中间的某条消息内容，从修改点到末尾的所有 token 都会 cache miss。
OpenClaw 的缓解设计——Cache TTL 对齐:
Context Pruning 的 5 分钟 TTL 不是随意选择的。它与 Anthropic 的 "short" cache retention（也是 5 分钟）精确对齐：
```
// Context Pruning 默认 TTL
ttlMs: 5 * 60 * 1000  // 5 分钟
// Anthropic cache retention 默认
cacheRetention: "short"  // 5 分钟
```
设计意图是：
- 在 cache 存活期间，不进行 pruning。因为 Provider 的 cache 还在，任何修改都会浪费已缓存的 KV。
- 当 cache 自然过期后，再做 pruning。此时 cache 已经失效，pruning 不会造成额外的 cache miss 成本。
- Pruning 完成后更新 lastCacheTouchAt，重新开始 TTL 计时，确保下一次 pruning 至少间隔一个 cache 周期。
此外，isCacheTtlEligibleProvider() 确保只有支持 cache 的 Provider 才启用这个基于 TTL 的 pruning 模式。不支持 cache 的 Provider（如 OpenAI、DeepSeek）不会使用 cache-ttl 模式，因此不受 TTL 约束。
Cache 失效时的成本影响：
当 pruning 确实执行时：
- Soft Trim 修改了 prompt 中间的内容 → 从修改点往后全部 cache miss
- Hard Clear 更彻底——将整条工具结果替换为占位符
但由于 pruning 只在 cache 已过期时执行，这次 miss 的额外成本仅是"cache write"（比普通 input 贵 25%），而不是"本可以 cache read 却 miss 了"（cache read 便宜 90%）。
3. 单条 Tool Result 截断 — 内存级无影响，持久级会 cache miss
- 内存级截断（truncateOversizedToolResultsInMessages）：只在内存中修改，不影响 session 文件。但由于修改了发送给 LLM 的 prompt，会导致一次 cache miss。不过这通常发生在超大 tool result 刚产生时（第一次发送），cache 尚未建立。
- 持久级截断（truncateOversizedToolResultsInSession）：通过 branching 重写 session 文件，会永久改变后续所有请求的 prompt 序列。必然导致一次完整的 cache miss。但这是溢出恢复的最后手段，能正常工作比 cache 命中更重要。
4. Compaction — 完全重建 prompt，cache 完全失效
Compaction 是最极端的操作——用一段摘要替代了大量历史消息。compaction 后的 prompt 与之前完全不同，KV Cache 必然 100% miss。
这是一个有意接受的 trade-off：
- Compaction 本身就是"不得不做"的操作（上下文已经快溢出了）
- Compaction 后 prompt 通常显著缩短，后续请求的 input token 总量大幅减少
- Cache write 成本是一次性的，后续请求立即开始构建新的 cache
   9.3 成本影响量化估算
以 Anthropic Claude 的定价为例（claude-opus-4-6）：
场景
input 单价
cache read 单价
cache write 单价
正常（cache hit）
$5/M
$0.5/M
—
Cache miss（新内容）
$5/M
—
$6.25/M
假设一次 100K token 的请求：
操作
Cache Hit 率
成本估算
正常对话（无 pruning/compaction）
~90%+
~$0.10（90K cache read + 10K input）
Pruning 执行后的第一次请求
0%（全量 write）
~$0.63（100K cache write）
Compaction 后（prompt 缩至 30K）
0%（全量 write）
~$0.19（30K cache write）
关键观察：Compaction 虽然导致 cache 完全失效，但由于 prompt 大幅缩短，即使全量 cache write 的成本也远低于溢出前每次请求的成本。
   9.4 总结
操作
Cache 影响
缓解措施
成本评估
History Turn Limit
低（一致性截断）
无需特殊处理
几乎无额外成本
Context Pruning
**中等**
（修改中间内容）
TTL 对齐 Provider cache 周期
仅在 cache 过期后触发
Tool Result 截断
中等（单次 miss）
只在溢出恢复时使用
一次性成本，可接受
Compaction
**高**
（完全重建）
prompt 大幅缩短补偿
长期看反而降低总成本
OpenClaw 的设计在 cache 效率 和 上下文管理 之间取得了合理的平衡。最关键的缓解机制是 Context Pruning 的 TTL 与 Provider Cache 周期对齐——这确保了最频繁的上下文修改操作不会浪费有效的 cache。而 Compaction 虽然代价最大，但它本身就是一个"救命"操作，执行后 prompt 缩短带来的长期成本节约远超一次 cache miss 的损失。