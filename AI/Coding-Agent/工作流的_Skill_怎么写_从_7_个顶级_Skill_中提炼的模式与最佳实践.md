---
title:      1|工作流的 Skill 怎么写？从 7 个顶级 Skill 中提炼的模式与最佳实践
source: https://mp.weixin.qq.com/s/aoNwyY5ZkCRMkZirn1rElQ
archived: 2026-06-16
---

#      1|工作流的 Skill 怎么写？从 7 个顶级 Skill 中提炼的模式与最佳实践

> 原文链接：https://mp.weixin.qq.com/s/aoNwyY5ZkCRMkZirn1rElQ

     4|一、Skill 是什么
     5|
     6|Skill 是一个文件夹，核心是 
     7|
     8|SKILL.md
     9|
    10| 文件，使用 
    11|
    12|YAML frontmatter + Markdown
    13|
    14| 正文
    15|
    16| 的格式。当 LLM 判断需要某个 
    17|
    18|Skill
    19|
    20| 时，会调用 
    21|
    22|skill
    23|
    24| 工具加载它，SKILL.md 的全部内容会作为 tool-result 注入到对话上下文中，LLM 读到后自主决定怎么执行。
    25|
    26|my
    27|-skill/
    28|
    29|├── SKILL.md              
    30|# 主文件（必须）
    31|
    32|├── scripts/              
    33|# 可执行脚本（可选）
    34|
    35|├── references/           
    36|# 详细参考文档（可选，按需加载）
    37|
    38|├── resources/            
    39|# 模板、清单等资源（可选）
    40|
    41|└── examples/             
    42|# 示例（可选）
    43|
    44|关键机制
    45|
    46|：
    47|
    48|Skill 本质是"知识注入"——它不会动态生成新工具，而是把指令文本注入到 LLM 的上下文中，LLM 用已有的工具（bash、read、edit 等）来执行这些指令。
    49|
    50|二、Frontmatter：
    51|
    52|决定 Skill 是否被加载的"门面"
    53|
    54|2.1 必填字段
    55|
    56|字段
    57|
    58|作用
    59|
    60|示例
    61|
    62|name
    63|
    64|唯一标识符，小写连字符
    65|
    66|test-driven-development
    67|
    68|description
    69|
    70|最关键
    71|
    72|——LLM 通过它决定是否加载
    73|
    74|见下方对比
    75|
    76|2.2 Description 的写法决定加载率
    77|
    78|# ✅ 好的 description — 包含触发短语和关键词
    79|
    80|description: Deploy applications 
    81|and
    82| websites to Vercel. Use when the user 
    83|
    84|  requests deployment actions like 
    85|"deploy my app"
    86|, 
    87|"push this live"
    88|, 
    89|
    90|  
    91|or
    92| 
    93|"create a preview deployment"
    94|.
    95|
    96|# ✅ 好的 description — 定义时序位置
    97|
    98|description: Use when implementing 
    99|any
   100| feature 
   101|or
   102| bugfix, before writing 
   103|
   104|  implementation code
   105|
   106|# ❌ 差的 description — 太模糊
   107|
   108|description: Helps 
   109|with
   110| deployment stuff
   111|
   112|核心原则
   113|
   114|：
   115|
   116|列举触发短语
   117|
   118|：
   119|
   120|把用户可能说的话写进去（"deploy my app"、"push this live"）
   121|
   122|定义时序位置
   123|
   124|：
   125|
   126|说明"在什么之前/之后"使用（"before writing implementation code"）
   127|
   128|包含产品关键词
   129|
   130|：
   131|
   132|如果覆盖大平台，把所有产品名列出来
   133|
   134|2.3 可选扩展字段
   135|
   136|从 7 个 Skill 中观察到的扩展字段：
   137|
   138|字段
   139|
   140|来源
   141|
   142|作用
   143|
   144|references
   145|
   146|OpenCode cloudflare
   147|
   148|声明最重要的参考文档
   149|
   150|allowed-tools
   151|
   152|Google Labs stitch-loop
   153|
   154|声明需要的工具权限
   155|
   156|type
   157|
   158|Dean Peters discovery-process
   159|
   160|声明 Skill 类型（workflow/component）
   161|
   162|best_for
   163|
   164|Dean Peters discovery-process
   165|
   166|最适合的场景列表
   167|
   168|scenarios
   169|
   170|Dean Peters discovery-process
   171|
   172|具体的触发场景示例
   173|
   174|estimated_time
   175|
   176|Dean Peters discovery-process
   177|
   178|预估执行时间
   179|
   180|三、5 种核心设计模式
   181|
   182|模式 1：线性流程
   183|
   184|适用场景
   185|
   186|：
   187|
   188|部署、安装、迁移等有明确步骤的操作。
   189|
   190|代表
   191|
   192|：
   193|
   194|openai/skills — vercel-deploy
   195|
   196|（77 行）
   197|
   198|结构
   199|
   200|：
   201|
   202|#
   203| 标题
   204|
   205|#
   206|
   207|# Prerequisites（前置条件）
   208|
   209|#
   210|
   211|# Quick Start（主流程：Step 1 → 2 → 3）
   212|
   213|#
   214|
   215|# Fallback（降级方案）
   216|
   217|#
   218|
   219|# Troubleshooting（故障排除）
   220|
   221|关键技巧
   222|
   223|：
   224|
   225|技巧
   226|
   227|示例
   228|
   229|为什么有效
   230|
   231|安全默认值
   232|
   233|"Always deploy as preview, not production"
   234|
   235|防止 LLM 做出危险操作
   236|
   237|具体命令
   238|
   239|每步给出可直接执行的 bash 命令
   240|
   241|LLM 不需要猜测
   242|
   243|超时提示
   244|
   245|"Use a 10 minute (600000ms) timeout"
   246|
   247|防止 LLM 因超时中断
   248|
   249|降级方案
   250|
   251|CLI 失败有 Fallback 脚本
   252|
   253|提供 B 计划
   254|
   255|负面指令
   256|
   257|"Do not curl the deployed URL to verify"
   258|
   259|明确禁止不该做的事
   260|
   261|适用判断
   262|
   263|：
   264|
   265|如果你的 Skill 可以用"先做 A，再做 B，最后做 C"描述，就用线性模式。
   266|
   267|模式 2：决策树 + 按需加载
   268|
   269|适用场景
   270|
   271|：
   272|
   273|大型平台选型、产品导航、问题诊断。
   274|
   275|代表
   276|
   277|：
   278|
   279|openai/skills — cloudflare-deploy
   280|
   281|（224 行）
   282|
   283|结构
   284|
   285|：
   286|
   287|#
   288| 标题
   289|
   290|#
   291|
   292|# Authentication（认证前置）
   293|
   294|#
   295|
   296|# Quick Decision Trees（决策树）
   297|
   298|  #
   299|
   300|## "I need to run code"（按用户意图分类）
   301|
   302|  #
   303|
   304|## "I need to store data"
   305|
   306|  #
   307|
   308|## "I need AI/ML"
   309|
   310|#
   311|
   312|# Product Index（产品索引表）
   313|
   314|关键技巧
   315|
   316|：
   317|
   318|技巧
   319|
   320|示例
   321|
   322|为什么有效
   323|
   324|用户意图分类
   325|
   326|"I need to run code" 而非 "Compute products"
   327|
   328|用用户语言而非技术术语
   329|
   330|树形导航
   331|
   332|├─ 边缘无服务器函数 → workers/
   333|
   334|LLM 快速定位正确产品
   335|
   336|渐进式披露
   337|
   338|主文件 7KB，references/ 按需展开到几十万字
   339|
   340|不浪费上下文窗口
   341|
   342|产品索引表
   343|
   344|Product → Reference 的映射表
   345|
   346|结构化的快速查找
   347|
   348|适用判断
   349|
   350|：
   351|
   352|如果你的 Skill 覆盖的知识域有 10+ 个分支，且每个分支都有大量详细文档，就用决策树模式。
   353|
   354|进阶
   355|
   356|：
   357|
   358|同一个知识域可以拆成两个 Skill——
   359|
   360|导航型
   361|
   362|（cloudflare）：只做选型，不涉及操作
   363|
   364|操作型
   365|
   366|（cloudflare-deploy）：包含认证、命令、故障排除
   367|
   368|模式 3：循环迭代
   369|
   370|适用场景
   371|
   372|：
   373|
   374|TDD、代码审查、设计评审等需要反复执行的流程。
   375|
   376|代表
   377|
   378|：
   379|
   380|obra/superpowers — test-driven-development
   381|
   382|（371 行）
   383|
   384|结构
   385|
   386|：
   387|
   388|#
   389| 标题
   390|
   391|#
   392|
   393|# The Iron Law（铁律——不可违反的核心原则）
   394|
   395|#
   396|
   397|# Red-Green-Refactor（循环体）
   398|
   399|  #
   400|
   401|## RED — 写失败的测试
   402|
   403|  #
   404|
   405|## Verify RED — 验证确实失败
   406|
   407|  #
   408|
   409|## GREEN — 写最少的代码
   410|
   411|  #
   412|
   413|## Verify GREEN — 验证确实通过
   414|
   415|  #
   416|
   417|## REFACTOR — 清理
   418|
   419|  #
   420|
   421|## Repeat（回到 RED）
   422|
   423|#
   424|
   425|# Common Rationalizations（借口反驳表）
   426|
   427|#
   428|
   429|# Verification Checklist（退出条件）
   430|
   431|关键技巧
   432|
   433|：
   434|
   435|技巧
   436|
   437|示例
   438|
   439|为什么有效
   440|
   441|强硬语气
   442|
   443|"Delete it. Start over."
   444|
   445|LLM 倾向于"灵活变通"，强硬语气提高遵从率
   446|
   447|Good/Bad 对比
   448|
   449|用 
   450|
   451|<Good>
   452|
   453| 和 
   454|
   455|<Bad>
   456|
   457| 标签包裹代码示例
   458|
   459|对比教学效果最好
   460|
   461|借口反驳表
   462|
   463|预判 LLM 可能的 12 种偷懒借口并逐一反驳
   464|
   465|堵死所有逃避路径
   466|
   467|验证清单
   468|
   469|8 项 checklist 作为循环退出条件
   470|
   471|确保质量达标才能结束
   472|
   473|人类兜底
   474|
   475|"ask your human partner"
   476|
   477|不确定时交给人
   478|
   479|适用判断
   480|
   481|：
   482|
   483|如果你的 Skill 需要 LLM 反复执行"做→验证→改进"的循环，就用迭代模式。
   484|
   485|模式 4：接力棒循环（跨 Session 持久化）
   486|
   487|适用场景
   488|
   489|：
   490|
   491|多次迭代的长期项目，需要跨多个 session 持续工作。
   492|
   493|代表
   494|
   495|：
   496|
   497|google-labs-code/stitch-skills — stitch-loop
   498|
   499|（203 行）
   500|
   501|