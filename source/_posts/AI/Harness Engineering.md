---
title: Harness Engineering
date: 2026-05-01 12:06:00
categories: 
- AI
tags:
- Harness
---



### 1 概念定义

**一句话定义**

Harness Engineering 是围绕 AI 模型构建"运行环境与管控基础设施"的工程实践，目的是让 AI Agent 在长时、复杂、多步骤的真实任务中稳定、可靠、高效地运行。



**核心思想：从"马具"理解 Harness**

Harness 这个词的本意其实是马具——套在马身上用来控制马的那些装备，比如缰绳、头套等。马本身非常强大，但人类必须借助马具的力量才能驾驭它、让它为我所用。



把这个类比迁移到 AI 领域：

* 大模型 = 一匹脱缰的马：能力极强，但如果任由它自己跑，就会发散思维、产生幻觉，无法稳定给出我们想要的结果。

* Harness = 那套马具：用来控制和驾驭大模型的系统。



由此我们可以推导出 Harness 的一个常见公式：

Harness=Agent−Model&#x20;

换句话说，一个完整的 Agent 减去里面的大模型，剩下的所有东西都是 Harness。需要注意，Harness Engineering 是非常新的概念，业界尚未形成严格定义，这个公式是目前比较被认可的一种说法，并非学术定义。



举个具体例子：在 Claude Code 里面，所有不属于 Claude 模型的部分都是 Harness——比如写在 `CLAUDE.md` 里那些大模型要遵循的规则、Claude Code 可以使用的工具列表、它的定时调度机制等等。



### **2 演进脉络**

![image-20260707011349375](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/image-20260707011349375.png)

三者是叠加关系：Harness 内部的每个 session 仍然需要 Context Engineering，每次发给模型的 prompt 仍然需要 Prompt Engineering。



### **3 Harness 工程实践要点**

* 外部持久记忆——用文件系统（JSON、Markdown、git log）代替模型内部记忆，解决"失忆"问题。

* 确定性验证轨道——Linter、类型检查、单元测试等刚性门禁，AI 的输出必须通过测试才能被接受（对应解决"幻觉完成"——不让模型自己说"我做完了"）。

* 原子任务 + 上下文刷新——每个 Agent 只做一个小任务就销毁，新 Agent 从干净状态启动，彻底杜绝 Context Rot。

* 子 Agent 集群（Swarm）——用大量便宜快速的小模型并行工作，而非一个大模型顺序处理。

* 技能文件（Skills）——按需加载的专用指南，Agent 只看到与当前任务相关的工具和说明（建议不超过 30 个工具）。

* Guard Rails + Checkpoints——在关键节点自动运行检查，防止 Agent 偏离轨道。

* Handoffs——session 之间通过进度文件、git commit 等传递状态，实现跨 session 连续性。

* Human-in-the-loop——在关键阶段设置人工审核断点，弥补 AI 自验证的不足。

* 架构约束——通过 ArchUnit 等框架强制要求 Agent 生成的代码遵循特定模式。

* 垃圾回收 Agent——定期扫描修复代码库和文档中的不一致性，对抗系统熵增。



**&#x20;   关键工程原则**

* **轻量化，为删除而建**：不要过度编码人类知识，模块要能快速移除替换（Manus 6个月5次重构、Vercel 删80%手工工具反而更好）

* **模型无关性**：Harness 不绑定特定模型，新模型出来能快速替换

* **将 Harness 视为数据集**：每次 Agent 失败、漂移、异常都是珍贵的训练素材，用于迭代优化



### **4 Harness 思想的运用**

#### 4.1 Retrival（检索）

##### **4.1.1 背景 / 基础知识**

我们需要根据自己的任务场景，来提供给 AI 检索的工具。虽然 Claude Code 里其实内置了 Grep 工具，但 Claude Code 是通用的，而我们的任务是定制的。这是一个实用的建议——不管你是使用 AI 还是构建 AI，都可以想一下：什么样的检索工具对我的场景是合适的？

举个例子：如果你的项目是一个超大型项目（比如 100GB 代码库），直接让 Claude Code 用 grep 线性扫盘会很慢；这时候你可能要给它建立一个 BM25 倒排索引，再以 Skill 的形式装上去让 CC 调用这个搜索工具。

检索工具从"词法"到"语义"是一条谱系，不是非此即彼：

![](/Users/vic/Downloads/大模型应用!算法学习路线+八股+面试实战_3/images/大模型应用!算法学习路线+八股+面试实战_3-image-54.png)

视频里 Raj 跑的实验是"**Agentic + 纯 BM25**"反超了"**One-shot + embedding 切块**"，但这是"embedding 从必选变可选"，不是"BM25 取代 embedding"。

##### **4.1.2  最佳实践建议**

\- **默认 grep 起手**：90% 的代码搜索靠 grep 就够，不要一上来就上向量库。

\- **代码库 > 5GB 或文件数 > 10w，加 BM25**：典型做法是包一个 SKILL（封装 \`ripgrep --index\` 或开源 BM25 库），让模型像调工具一样用。

\- **跨语言/跨同义词的查找才用 embedding**：比如查"用户鉴权相关代码"，关键词散在 \`auth/login/session/jwt\`——这种 case 上 semantic search。日常代码定位别用。

\- **配好模型（Opus/GPT-5 级别）就上 Agentic RAG**：让模型自己重写 query、按文件级别迭代，比预先 chunk 切片更准；前提是你愿意多花 2–5x token。底层用 BM25 / embedding / 混合都可以。

\- **文件级 > chunk 级**：能不切块就不切块，整文件交给模型，避免上下文撕裂。

\- **什么时候迁数据库**：当你需要"按 author 过滤 + 按 commit 时间排序 + 按路径匹配"这种 **metadata join** 时，才考虑结构化索引/数据库；纯文本搜索别上 DB。

***

#### **4.2 Context & Memory（上下文与记忆）**

##### **4.1 背景**

虽然主流模型号称 1M token 上下文，但实测**填越多越退化**——通常超过一半就开始丢信息。所以"有 1M 窗口"≠"能用 1M 窗口"，真正能可靠使用的往往只有前 30–50%。这就引出一个问题：信息越来越多（长对话、错误日志、跨会话的项目知识），但能放进窗口的就这么多，**该把什么放进来、什么放出去、什么留到下次**？

视频博主把这件事拆成三层记忆来处理：**Active Context Window、Working State、Durable Memory**。下面就只讲清楚这三层各自负责什么。

> 关于记忆的分类有很多说法（短期/长期、工作记忆/情景记忆等），笔记里讲开源架构（CC、OpenHands、Claude Code）时也各有各的叫法。本质都是一回事，只是切分粒度不同。本节更重要的还是使用这些记忆背后的最佳实践和蕴含的思想。



##### **4.2 基础知识：三层记忆分别干什么**

**Layer 1：Active Context Window（当前上下文）**

就是这一轮 API 调用真正发给模型的那段 token。它的职责是**管好"现在"**——决定这次推理时模型眼前能看到什么。因为窗口会退化，所以这一层的核心工作是"瘦身"：用 sliding window 只保留最近几轮、用 compaction 把老对话压成摘要、用 rewind 把走错的分支直接砍掉，避免错误内容继续占着窗口。



**Layer 2：Working State（临时工作文件）**

> 和跨会话记忆相同的地方都是用外部文件存储。不同的地方在于这个是给当前对话用的，当前对话用完就丢。而长期记忆的文件是跨会话的。这里其实对我们做Agent有一个启示，就是不要什么东西都往上下文塞，我们可以把一些结果保存在磁盘，运行的时候动态查找。因为上下文很宝贵。

任务进行中、但塞不进窗口的中间状态，落到**文件系统**里。它的职责是**管好"这次任务"**——给 agent 一个比 context window 大得多的"草稿纸"。最典型的就是 \`plan.md\` / \`todo.md\`：让 agent 边做边勾选，目标不会在长循环中走丢。再激进一点是 Recursive Language Models 范式：把所有历史都落盘，用 Python REPL 按需检索，相当于"无限上下文"。任务结束这层就可以扔掉。



**Layer 3：Durable Memory（跨会话长期记忆）**

**跨会话**留下来的知识，下次开新会话还能用。它的职责是**管好"这个项目/这个 agent"**——把经验沉淀下来不要每次重学。两种主流形态：

\- **AGENTS.md**：项目级事实（构建命令、目录约定、坑），主流 harness 都会自动注入 system prompt。

\- **Skills**：博主把SKILL定义成了新的一种Agent的持久记忆的表现形式。

三层是**自下而上递进**的：窗口装不下 → 落到 working files；任务结束还想留 → 升级成 durable memory。



##### **4.3 最佳实践建议**

按三层记忆分别给出操作建议，**自下而上**对应"管现在 / 管这次任务 / 管这个项目"：

**Layer 1：Active Context Window —— 管好"现在"**

\- **主动管理上下文**：

1. 主动压缩上下文，不要等满了再压缩，因为模型能力会随着窗口增大而退化。

2. 不要过于依赖模型的compaction, 因为很多compaction是黑盒，你不知道它发生了什么。另外主动压缩，指定它保留哪些信息，是好的工程实践（主动使用claude code的/compaction，后面加上参数，指定compaction的时候保留哪些信息。

3. 只保留关键的信息在上下文，避免挤爆上下文。比如长 stack trace、20000 行日志先在 Agent 层截断/抽取关键帧再喂回去。使用Claudecode的rewind功能，直接把走错的分支砍掉，不要让错误信息继续占着窗口。



**Layer 2：Working State —— 管好"这次任务"**

\- **长任务先写一个 todo markdown**：把计划落盘成 \`plan.md\` / \`todo.md\`，让 agent 每完成一步勾选；避免任务在长循环中丢目标。（Langchain的Deep Agent就是这么做的）

\- **能落盘就别塞上下文**：中间产物（搜索结果、抓取的网页、生成的代码片段）写文件，让 agent 按需读，不要全堆进窗口。上下文很贵，磁盘很便宜。（博主举了一个例子，让Agent 所有历史落盘，用 Python REPL 按需检索，相当于给 agent 一张"无限大草稿纸"  相比不使用这个技巧，效果更好）

\- **任务结束就清理**：working files 是一次性的，任务完成后归档或删除，别让它污染下一次任务。



**Layer 3：Durable Memory —— 管好"这个项目 / 这个 agent"**

\- **AGENTS.md 铁律**：

1. 手写不自动生成；

2. 只写"项目特有的事实"（构建命令、目录约定、坑），不写通用编程知识；

3) Boris(CC的开发者）建议每次大模型升级或每季度复盘一次，删掉过时条目。（也许这个有写夸张了，不过定期检查Agent.md是一个很好的习惯）

4) Agentd.md写废话，反而会误导模型。 具体这里可以参考笔记论文部分。

\- **写 Skill 的判定标准**：

1. 同一类操作做过 3 次以上 + 步骤超过 5 步 + 通用模型容易做错 —— 才值得沉淀成 Skill。

2\. **Skill 建议配上 eval**：如果你的SKILL写了很多模型已经知道的废话，反而会降低模型性能。所以可以给每个 Skill 准备一个小评测集，定期跑 **with skill vs without skill** 两组对照；模型升级后重跑，发现负贡献立刻下线（旧 Skill 可能被新模型的原生能力 superseded）。

***

#### **4.3 Loops, Tool Use, and Feedback（循环、工具与反馈）**

##### **4.3.1 背景**

Agent 之所以比单次 LLM 调用强，核心是**循环 + 反馈**——给它时间思考、跑工具、看结果、再调整。从 模型o1 开始，模型就不再是"一锤子买卖"，而是默认要在 harness 里循环跑、反复 refine。



但"循环跑起来"只是第一步。真正决定循环跑得好不好的，是**循环周围的那一圈配套**：

* 循环本身要够聪明（不能每次啥也没学会，就盲目重试）；

* 循环里跑的工具不能反过来污染循环（一条 20000 行的日志就能撑爆窗口，后面循环全废，参考杠杆 2）；

* 循环不能放任 agent 写出越来越大的文件、越来越乱的结构；

* 循环出错时不能炸掉真机（API key、`~/.ssh` 直接暴露）。

所以这一节真正讲的是一件事的四个层面：**让循环本身更聪明 + 让工具 I/O 不污染循环 + 给循环加结构性约束 + 给循环加安全边界**。下面就按这四块展开。



> SWE-bench 论文留下的三条设计原则贯穿全节：actions 简单紧凑、environmental feedback 信息量足且简洁、guardrails 抑制错误扩散且便于恢复。



##### **4.3.2 基础知识**

**1. 循环范式（Loop Strategies）—— 让循环本身更聪明**

循环不是一种东西，而是一条从"暴力重试"到"假设-验证"的谱系，越靠后越省 token、越快收敛：

1\. **Ralph Wiggum loop**（最朴素）：失败就清空重来、什么也不学 → 极度耗 token，但能 work。视频里举了用这种方式硬怼出一个 C compiler 的例子。（但这种方式很蠢啊，感觉也会陷入死循环，感觉谁会用这种方式写Agent啊。。）

2\. **Plan-then-Execute**：先规划再执行，中途守住计划。大多数现代 harness 默认就这么做。

3\. **Hypothesis → Verification loop**（Karpathy 的 auto-research 范式）：假设 → 跑实验 → 看结果 → 反馈进下一轮 → 螺旋上升。这是当前性价比最高的循环形态。

4\. **Test-Driven Development for Agents**：把测试当作循环的成功判据。Factory 的提醒——**务必先写测试再写代码**，反过来会把已有 bug 也"橡皮图章"进测试。



** 工具 I/O 卫生（Tool Hygiene）—— 别让工具污染循环**

循环每跑一轮，工具的输出都会回灌进上下文。如果不管 I/O，工具一次吐 20000 行日志，几轮下来窗口就废了（参考杠杆 2 的"窗口退化"）。所以 harness 必须替 agent 做这件事：

\- **截断**：harness 自动只保留尾部 2000 行而不是整段 20000 行。看似简单，但能直接救活后续 N 轮循环。

\- **抽帧**：长 stack trace 只保留报错那几帧 + 失败的测试名，而不是整段 traceback。

\- **落盘**：超大输出（>5MB）直接写文件，agent 按需读，不要全堆进窗口。



**3. 结构性约束（Structural Constraints）—— 替循环挡掉错误方向**

这部分是 OpenAI 反复强调的点："开发者正在变成 AI 系统的管理者"——写代码已经不是瓶颈，**给系统设边界**才是。具体做法是把约束写死进 instructions / system prompt，让 agent 在循环里被动遵守，比如可以规定以下规则：

* 单文件不超过 200 行 → 强迫 agent 分解，而不是堆一个巨大的 god file。

* 函数 / 模块大小限制 → 同理。

* 新增依赖必须先问 → 防止循环里悄悄引入一堆库。

这些约束的本质是：**用 harness 替你做"代码 review"的硬性那一半**，让模型不会沿着错误方向越跑越远。



**4. 安全护栏（Safety & Guardrails）—— 让循环出错不致命**

视频里 Raj 反复警告：**不要在 laptop 上直接跑陌生 agent**。宿主机上有 API key、\`\~/.ssh\`、浏览器 cookie，循环一旦失控代价巨大。所以 harness 这一层要提供两道防线：

\- **隔离（Sandbox）**：容器 / VM / devcontainer，给 agent 一个干净的、可丢弃的环境。

\- **审批关卡（Guardrails）**：三档放行

* ✅ 自动允许：读文件、跑测试、`git diff`

* ⚠️ 需要确认：写文件、安装依赖

* 🛑 强制人工：`rm -rf`、`git push --force`、删分支、改共享配置



##### **4.3.3 最佳实践建议**

按上面四个层面对应组织：

**1. 循环范式**

\- **能用"假设-验证"loop 就别用 Ralph Wiggum**：明确让模型先说"我猜问题在 X，我要跑 Y 来验证"，再跑——比"失败重试 10 次"省 5–10x token。

\- **任务 ≥ 3 步先要计划**：复杂任务用"先 plan、再 execute、每步对照 plan"；简单一步任务不要强加 plan，反而拖慢。

\- **TDD 严格"先测试，后代码"**：让 agent 先写测试并跑红，再写实现到绿；反过来会让测试为已有 bug 背书。



** 工具 I/O 卫生**

\- **所有工具输出加截断**：默认保留尾部 2000 行 / 50KB；长 stack trace 只留报错块；命令输出 > 5MB 强制落盘成文件再让 agent 按需读。

\- **错误反馈要"短而准"**：与其把整段 traceback 喂回去，不如让 harness 抽取 error type + 出错文件:行号 + 最近一帧 + 失败的测试名。



**3. 结构性约束**

\- **写死结构约束进 system prompt**：例如"单文件 ≤ 200 行""函数 ≤ 50 行""新增依赖必须先问"——强制 agent 分解和确认。

* 这条对应 OpenAI 那句"developer → manager"——你不再亲自写每一行，而是给系统定规则。

**4. 安全护栏**

\- **永远在 sandbox 跑陌生 agent**：容器 / VM / devcontainer 三选一，宿主机上的 \`\~/.aws\`、\`\~/.ssh\`、浏览器 cookie 别暴露给 agent。

\- **三档权限白名单**：✅ 读文件 / 跑测试 / git diff → 自动；⚠️ 写文件 / 安装依赖 → 确认；🛑 \`rm -rf\` / \`git push --force\` / 删分支 / 改共享配置 → 必须人工。

***

#### **4.4 Orchestration（单 Agent vs 多 Agent）**

##### **4.4.1 背景 / 基础知识**

直觉是："上下文窗口不够大，那就拆给多个 agent 并行做。"听起来很对。但视频里 Raj 给的**反直觉真相**是：多 Agent 不是默认更好——多 benchmark 平均下来，**Single Agent 表现优于 Multi-Agent**。

原因是**协调税（coordination tax）**：agent 之间要互相同步状态、互相理解输出、互相处理冲突，这一层开销往往把并行收益吃光。社交媒体上常见的"我同时开 100 个 agent 改 100 个 bug"在真实工程里几乎一定塌掉。

多 Agent 真正有效的场景是结构化的：

\- **Orchestrator-Worker**：一个主 agent 派任务、若干 worker 干活、最后汇总。

\- **明确可并行的子任务**：比如对 50 个独立文件做格式化，子任务之间零依赖。

\- **Reflection / Critic**：一个 agent 写代码、另一个 agent 专门挑毛病。视频里 Raj 自己的工作流——**Opus 写代码 + GPT 系列反思批评**——就是这个模式；OpenHands 等 harness 已经内建。

Factory 的实战结论也印证了这点：**串行 Orchestrator → Worker → Validator** 比"并行 swarm 一百个 agent"靠谱得多。



##### **4.4.2 最佳实践建议**

\- **默认就用 Single Agent**：除非你能说清"我为什么需要并行"或"我需要一个独立 critic"，否则不要拆。

\- **要并行先验证"子任务零依赖"**：能不能用一句 \`for f in files: agent(f)\` 描述清楚？不能就意味着有隐式依赖，并行会出乱子。

\- **Orchestrator-Worker 模式的硬要求**：worker 之间不能直接通信，只通过 orchestrator 中转；每个 worker 的输入输出必须是可序列化的明确契约（JSON / 文件路径），不要靠"共享内存"。

\- **强烈推荐 Reflection 模式**：主写 + 副审 是 ROI 最高的多 agent 用法；建议主副用**不同厂家的模型**（如 Opus 写 + GPT 审），让盲区互补。

\- **限制 agent 数量**：一个 orchestration 内活跃 agent ≤ 5；超过这个数协调成本会指数上升。

\- **不要让多 agent 完全 event-driven**：自由互相调用 = 死锁/无限循环高发；坚持**有明确层级**的调度。

\- **Reflection 的输出要可执行**：让 critic 直接产出 diff 或 patch，而不是写"建议你考虑改进 X"——后者 main agent 不一定听得懂。

***

### **5 Anthropic Harness架构详解**

Anthropic 的 Harness 是目前行业内最被广泛引用的长时任务管控架构范例。我们来对它进行拆解，来看一下Anthropic公司是如何使用Harness架构来完成一个长时，复杂的Vibecoding项目的。（其实我们这个笔记中的RAG项目也是从一个文档为起点，完成整个代码的。但是当时我们的方法比较朴素，用一个agent就做到底。学了本章，我们可以试一下，使用Harness 工程，来完成这个项目，看看效果会不会更好）

#### 4.1 设计哲学

Anthropic 的 Harness 核心设计理念是：与其让一个天才持续工作直到崩溃，不如让无数个"新手"接力完成，每人只干一件事。传统做法是把一个强大的模型（如 Claude Opus 4.6）扔进去从头干到尾——但无论模型多强，单一上下文窗口最终都会被垃圾数据淹没，造成**上下文腐化**，导致**失忆和幻觉**。

Anthropic 的方案彻底绕开了这个问题：**每个 Agent 只活一次，干完就死**。

#### 4.1 总体结构

**1. App Spec / PRD：项目起点**

&#x20;Anthropic 的 Harness 通常从一份由人提供的**需求说明开始**。这份需求说明不是直接交给模型去连续编码，而是作为整个系统初始化的输入。它的作用是定义产品目标、功能范围和预期结果，为后续的任务拆解和状态组织提供基础。也就是说，项目最开始进入系统的不是"代码任务"，而是一份高层次的目标描述。

** Initializer：初始化规划 Agent**

Initializer 的职责不是直接产出业务代码，而是把需求文档转化为**后续开发可以执行的工程结构**。它会**读取需求，拆解出特性清单，建立任务边界，定义每个功能点的验证标准，并完成项目骨架和代码仓库的初始化。**&#x8FD9;个阶段的意义在于，先把一个模糊的产品目标整理成一个可以持续推进、可以被后续 agent 反复读取和执行的蓝图。

**3. Coding Loop：编码循环 Agent**

系统进入正式执行阶段后，**并不是由同一个 agent 一直写下去，而是不断启动新的 coding agent，让它们逐轮接力**。每一轮 agent 都会先读取当前的任务状态和项目进度，然后只选择一个明确的小任务去实现。完成之后，它不会长期保留在系统中，而是会在交接完成后退出。下一轮再由一个全新的 agent 接手。Anthropic 正是**通过这种循环式接力，而不是持续式长对话，来避免上下文不断膨胀的问题**。

**4. External Memory：外部记忆系统**

**&#x20;Anthropic Harness 的一个核心思想，是将项目状态从模型内部记忆中剥离出来，放入外部可持久化的介质中**，比如特性清单、进度文件和 Git 历史。这样一来，系统的连续性不依赖某个 agent 的上下文窗口，而依赖外部状态记录。每个新的 agent 启动时，只需要重新读取这些状态，就能接手上一个 agent 已经完成的工作，从而让整个系统具备持续推进的能力。

**5. Deterministic Validation：确定性验证轨道**

&#x20;Anthropic 的架构**并不把“是否完成”这件事交给模型自己判断，而是通过一套外部的、确定性的验证机制来裁定结果是否成立**。也就是说，agent 生成代码之后，系统会通过测试、检查和验证来决定这项工作能否被接受。只有满足这些外部标准，系统才会更新状态并进入下一轮。这种设计的意义在于，用工程规则替代模型主观判断，把“感觉完成了”变成“验证通过了”。

**6. Sandbox / Isolation：隔离执行环境**

每个 agent 的运行通常都处于**一个受控环境中**，**这样既能限制它可访问的工具范围**，**也能避免错误或脏上下文污染整个系统**。隔离环境让每一轮执行都尽量在明确边界内进行，也让失败后的回滚、重试和重新启动变得可控。Anthropic Harness 依赖这种隔离性，把 agent 的自由度压缩在一个足够安全、足够清晰的工作空间里。

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-73.png)

**总结来说**：Anthropic 公司的 Harness 架构核心，可以概括为一种“**初始化拆解 + 外部状态持久化 + 单任务循环执行 + 确定性验证门控”的长时任务运行体系**。 它的关键不在于让单个大模型持续工作，而在于先由 Initializer 将需求拆解为可执行的特性清单，再让一个个全新的 Coding Agent 逐轮读取状态文件与代码库，只完成一个原子任务，并通过测试、提交与交接文件更新系统状态后立即退出。这样做的本质，是把**项目记忆从模型内部迁移到外部系统**，**把任务完成判断从模型主观输出转移到确定性验证机制，从而显著降低上下文腐化、幻觉完成和多步任务失控的风险。**

#### 4.3 局限性

相比于单个Agent直接完成整个项目，Anthropic 公司也承认整个工程的局限性（但我觉得Harness Engineering还处于早期发展阶段(2026年3月写），Anthropic 公司给大家打了个样，后续大家会继续在此基础上改进和发展，仍然有非常大的借鉴作用）

**Initializer 是单点高杠杆环节**

如果初始拆分错了，后面全部都会跟着错。也就是说，蓝图质量决定了整个系统上限。

**Handoff 质量并不稳定**

进度文件有时候总结得好，有时候会漏掉关键问题。一旦漏掉“为什么失败、怎么修复”，后面的 agent 可能反复踩同一个坑。

**多步可靠性仍然会衰减**

即使单步成功率很高，步骤一多，整体可靠性还是会下降。所以它不是彻底解决了可靠性问题，而是显著缓解。

**仍然需要 Human in the Loop**

从实战角度看，Anthropic Harness 更像：高度自动化，但不是完全放手，还是在关键节点引入人工检查。

#### 4.4 参考资料

https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

### **6 ClawCode Harness工程详解**

#### 6.1 整体方法论：五阶段流水线

Claw Code 的核心思路是：**先把"什么需要被实现"变成机器可查的清单，再边实现边用 mock harness 自动验证行为对等性。**

整个流程分五个阶段：

1\~4阶段是准备。5阶段才是真正的coding。接下来我们逐一讲解。

#### 6.2 五个阶段详解

##### 6.1 归档 + 快照原始代码表面

第一步不是写代码，而是**把要重写的东西完整记录下来，形成机器可查的"参考清单"**。

Claude Code 的代码组织非常规整——所有命令集中注册在 `src/commands.ts`，所有工具集中注册在 `src/tools.ts`，启动流程定义在 `src/entrypoints/cli.tsx`。每个命令/工具都是一行标准的 `import` 语句：

因为格式统一，用**逐行字符串匹配**就能提取出所有名字和路径——，不需要 AI 介入。

具体做了三件事：

1. **获取原始 TypeScript 源码快照。** 将 Claude Code 的原始 TS 源码存放在 `archive/claude_code_ts_snapshot/src`（在 `.gitignore` 里忽略，不公开上传），作为本地参考基准。

2. **compat-harness 自动提取清单。** 仓库中的 `rust/crates/compat-harness` 是一个 Rust crate，专门解析上游 TS 源码：扫描 `commands.ts` 逐行匹配 `import { xxx } from "./commands/..."`，提取出 207 个斜杠命令；扫描 `tools.ts` 提取 184 个工具模块；扫描 `cli.tsx` 搜索关键词提取启动流程的 7 个阶段。这是一个"活的"提取器——当上游 TS 代码更新时可重新运行，自动刷新清单。

3. **生成结构化快照 JSON。** 提取结果固化为四组 JSON 文件，存放在 `src/reference_data/` 目录：

   * `archive_surface_snapshot.json`：原始仓库的整体结构（18 个根文件、35 个子目录、1,902 个 TS 文件）

   * `commands_snapshot.json`：所有命令的 name + source\_hint + responsibility（207 条）

   * `tools_snapshot.json`：所有工具模块的 name + source\_hint + responsibility（184 条）

   * `subsystems/*.json`（29 个文件）：各子系统的模块数和样本文件名（assistant、bridge、utils 等）

这些清单的用途：**量化 scope**（"一共 207 个命令 + 184 个工具"→ 可估算工作量）、**定位参考代码**（AI 收到工单时知道去哪个文件读原始逻辑）、**发现复杂度分布**（`BashTool` 有 18 个子模块是最复杂的）、**覆盖率审计**（随时对比已实现多少）。局限性：这种字符串匹配**只适用于代码结构规整的项目**，如果用动态注册或插件扫描就行不通。



##### 6.2  Python 镜像工作区

不直接从 TS 翻 Rust——直接翻译等于盲写。原始 TS 有 1,902 个文件、51 万行代码，AI agent 拿到工单后如果要自己在里面翻找架构信息，效率极低且容易遗漏。Claw Code 先建了一个 **Python 中间层**——一个只有 67 个文件的轻量"沙盘模型"，让 AI agent 在后续 Rust 重写时可以先查这个中间层快速定位，而不需要大海捞针。

这个 **Python 中间层包含四样东西**：

**1:1 结构映射表。** `src/parity_audit.py` 定义了 TS→Python 的完整映射关系，原始的 18 个根文件和 35 个子目录全部有对应的 Python 占位物。这张表的作用是让任何人（或 agent）一眼看清"原始系统有哪些组成部分，每个部分对应到 Python 层的哪个文件"。



**占位包（29 个子目录的结构索引）。** 29 个子目录（`src/assistant/`、`src/bridge/`、`src/utils/` 等）各有一个 `init.py`，但不实现任何业务逻辑。每个占位包只做一件事——从 JSON 加载该子系统的元数据，导出 `MODULE_COUNT`（有多少模块）和 `SAMPLE_FILES`（包含哪些文件）。当 AI agent 收到"实现 assistant 模块"的工单时，读一下占位包就知道原始子系统有 1 个模块、文件叫 `sessionHistory.ts`，然后直接去对应的 TS 源码中精准阅读，而不需要在 1,902 个文件里盲目搜索。



**关键架构模型（5 个 Python 文件复现核心决策逻辑）。** 光知道"有哪些模块"不够，还需要知道"系统怎么运转"。Python 层用 5 个文件把原始系统最关键的几条逻辑管线提取出来，编码为可运行的 Python 代码：

* `bootstrap_graph.py`：启动流程的 7 个阶段——prefetch → 环境检查 → CLI 解析 → 命令加载 → 延迟初始化 → 模式路由 → 查询主循环

* `command_graph.py`：把 207 个命令分类为 builtins（185）、plugin-like（20）、skill-like（2）

* `query_engine.py`：复现查询引擎的 turn-loop 行为——轮次限制（max turns）、token 预算执行（max budget）、消息压缩触发（compaction）。这里有真实的状态管理逻辑，不是 stub

* `setup.py`：建模 workspace 环境发现和启动 preflight 流程

* `runtime.py`：建模运行时路由——把用户的 prompt 分词、和所有命令/工具的名字做匹配打分、返回 top-N 结果。这也是真实的匹配算法，给一个 prompt 会返回实际的路由结果



**可运行的审核 CLI。** Python 层提供了 `python -m src.main`（40+ 个子命令），供人类和 AI agent 随时验证中间层的自洽性：`summary` 输出工作区摘要，`parity-audit` 对比覆盖率（确保没有遗漏的模块），`route <prompt>` 验证路由逻辑是否和预期一致，`bootstrap <prompt>` 跑通完整的启动→路由→执行→历史记录流程。这个 CLI 同时也是 Rust 重写的**行为基准**：Rust 版的路由结果应该和 Python 版匹配，不匹配就说明实现有偏差。



##### 6.3  量化重写进度

核心思想：把"重写进度"从感觉变成**机器可检查的量化指标**。具体是通过三样东西实现的：**一个自动审计脚本、两份 PARITY 清单文档、以及一个 9-lane 并行开发模型。**



**自动审计脚本。** `parity_audit.py` 中的 `run_parity_audit()` 自动扫描当前代码库，计算四个维度的覆盖率：根文件覆盖率（18 个中实现了多少）、目录覆盖率（35 个中覆盖了多少）、命令覆盖率（207 个中实现了多少）、工具覆盖率（184 个中实现了多少），并列出所有缺失的目标。测试中有断言确保覆盖率只升不降——写了一个新模块覆盖率上去了，后面不允许退化回来。



**PARITY.md 双层清单。** 项目维护两份 PARITY 文档，分别追踪不同粒度的进度：

* **顶层 `PARITY.md`**：记录 9 条开发 lane 的宏观状态——每条 lane 的 feature commit hash、merge commit hash、diff 行数统计，做到全程可追溯

* **`rust/PARITY.md`**：更细粒度，对 40 个工具逐个标注实现深度，分四档：strong parity（行为完全对齐）、good parity（核心路径对齐）、moderate parity（基本可用）、stub（仅占位）



**9-lane 并行开发模型。** 人类根据前两个阶段产出的清单，把重写工作切分为 9 条相互独立的 lane，每条一个 feature branch，可以由不同的 AI agent 并行推进，互不阻塞：

* **Lane 1 — Bash validation**：9 个验证子模块（+1,005 行）

* **Lane 2 — CI fix**：sandbox 探测修复（+22/-1 行）

* **Lane 3 — File-tool**：边界检查——二进制检测、大小限制、路径穿越防护（+195/-1 行）

* **Lane 4 — TaskRegistry**：内存任务生命周期管理（+336 行）

* **Lane 5 — Task wiring**：6 个 task 工具接入 registry（+79/-35 行）

* **Lane 6 — Team+Cron**：团队和定时任务注册表（+441/-37 行）

* **Lane 7 — MCP lifecycle**：MCP 服务器连接、资源、工具分发桥接（+491/-24 行）

* **Lane 8 — LSP client**：LSP 诊断、hover、定义、引用、补全（+461/-9 行）

* **Lane 9 — Permission enforcement**：工具门禁、文件写入边界、bash 只读检查（+357 行）

每条 lane 独立 merge，互不阻塞。PARITY.md 中记录了每条 lane 的 feature commit hash 和 merge commit hash，做到**全程可追溯**。



##### 6.4 确定性行为验证

重写不能靠"编译通过就算对"，但也不能每次测试都调真实 Anthropic API（贵、慢、结果不确定）。解法是建一套**确定性的端到端测试体系**，不调真实 API 也能验证 Rust 重写的行为是否和原始系统一致。这套体系由三样东西组成：一个 mock 服务、一组场景脚本、以及一个双向验证工具。

**Mock Anthropic 服务。** `rust/crates/mock-anthropic-service` 是一个独立 Rust crate，实现了 Anthropic 兼容的 `/v1/messages` 端点。它不调真实模型，而是返回**预编排的** SSE 流式响应，可以脚本化多轮对话，每次运行结果完全相同。CLI 通过设置 `ANTHROPIC_BASE_URL` 指向这个本地服务，就可以在完全离线、零成本的环境下跑端到端测试。



> 这个场景脚本其实就是一些测试用例，进度文档需要根据这些测试用例来判断是否将某些模块标记为完成。

**10 个场景脚本。** 在 `mock_parity_scenarios.json` 中定义了 10 个测试场景，覆盖了核心工具链路、权限拒绝、多工具同轮、插件路径等关键行为。每个场景都通过 `parity_refs` 字段指向 PARITY.md 中的具体条目，确保测试和文档一一对应：

* `streaming_text`（baseline）：纯文本流式响应，无工具调用

* `read_file_roundtrip`（file-tools）：文件读取往返

* `grep_chunk_assembly`（file-tools）：grep 分块 JSON 组装

* `write_file_allowed`（file-tools）：workspace-write 模式下写入成功

* `write_file_denied`（permissions）：read-only 模式下写入被拒绝

* `multi_tool_turn_roundtrip`（multi-tool）：同一 turn 内执行多个工具

* `bash_stdout_roundtrip`（bash）：bash 执行和 stdout 往返

* `bash_permission_prompt_approved`（permissions）：bash 权限提示 → 批准

* `bash_permission_prompt_denied`（permissions）：bash 权限提示 → 拒绝

* `plugin_tool_roundtrip`（plugin）：加载外部插件工具并执行



**双向验证工具（防止文档和代码漂移）。** `run_mock_parity_diff.py` 做两件事：先遍历每个场景脚本的 `parity_refs`，确认每条引用在 PARITY.md 中**确实存在**（找不到就报错）；再运行 `cargo test` 输出每个场景的通过状态。这意味着 PARITY.md 不只是给人看的文档——它是场景清单的**锚点**，文档改了但测试没更新、或者测试加了场景但文档没跟进，都会被这个工具检测到。



##### 6.5 多Agent协作执行

如 `PHILOSOPHY.md` 所述，这个项目的核心哲学是：**人类提供方向，claws（AI agent）执行劳动**。**前四个阶段建好了清单、中间层、进度条和测试题，第五阶段就是让多个 AI agent 并行干活，人类只在关键节点介入。**

为了实现这一点，项目作者编写了三个专用编排工具（不是通用框架，而是为这套 Vibe Coding 工作流定制的）：

* **OmX**（oh-my-codex）——**任务拆解器**：把一句话指令转化为结构化工单，定义子任务、执行模式和验证标准（仓库地址：https://github.com/Yeachan-Heo/oh-my-codex）

* **clawhip** ——**后台监控员**：监听 git commits、CI 结果、PR 状态、agent 生命周期事件，在 agent 上下文窗口之外独立运行，只推送通知不干扰执行（仓库地址：https://github.com/Yeachan-Heo/clawhip）

* **OmO**（oh-my-openagent）——**角色协调员**：管理 Architect/Executor/Reviewer 之间的分工和分歧解决，确保多 agent 循环收敛而不是无限争吵。（仓库地址：https://github.com/code-yeongyu/oh-my-openagent）



有了这三个工具，加上前四个阶段建好的清单和测试，就能实现"人类发指令、agent 自主干活"的工作流。**完整流程如下：**

1. **人类输入一句高层指令**（通过 Vibe Coding 工具），例如："实现 Lane 3：给 file tool 加边界检查"

2. **OmX 自动拆解**成子任务（二进制检测、大小限制、路径穿越防护），确定执行模式（并行/串行）和验证标准（cargo test + mock parity 通过）

3. **OmO 分配角色并启动 agent**：Architect 规划方案、Executor 写代码、Reviewer 审查质量。多组 agent 在服务器上并行工作

4. **Agent 自主循环**：写代码 → 跑测试 → 失败 → 读错误 → 改代码 → 再跑测试。Reviewer 不满意就打回重做，OmO 协调分歧直到收敛。**全程不需要人类参与**

5. **clawhip 在后台监控**所有事件（`lane.started`、`lane.commit.created`、`lane.red`、`lane.green`、`lane.pr.opened`、`lane.finished` 等），向人类推送状态通知，但**不打扰正在干活的 agent**——agent 的上下文窗口是宝贵资源，不能被监控噪音占据

6. **人类只在两种情况介入**：agent 自己恢复不了的失败（决定重试还是换方案），或 lane 完成后判断能否 merge



#### 6.3 设计原则

这套方法论的核心是四条设计原则：

1. **人定标准，AI 执行。** 所有判断标准由人预先制定——覆盖率数字、parity 标签体系（strong/good/moderate/stub）、mock 场景清单。AI agent 不需要自己判断"够不够好"，只需对照人制定的验收标准逐项打勾。

2. **建中间层，降低 AI 认知成本。** 51 万行 TS 太大，AI 无法每次从头阅读。用 JSON 索引 + Python 骨架建一个轻量参考层，让 agent 查一下就能定位和理解，而不需要大海捞针。

3. **用工具编排多 agent，而非人工协调。** Python 脚本生成"做什么"（清单），OmX/clawhip/OmO 三件套负责"谁来做、怎么分发、怎么验收"，人类不需要坐在终端前逐个分配任务。

4. **人只做方向、分解、判断。** 人基于架构理解把工作拆成 9 条独立 lane，定义 lane 边界和合入标准。292 个 commit 中，人的贡献集中在：重写什么（方向）、怎么拆（分解）、何时 merge（判断）。剩下的全是 agent 的事。