---
title: SKILL
date: 2026-01-30 17:06:00
categories: 
- AI
- SKILL
tags:
- SKILL
---



### 3.1 SKILL 概念

Skill是一组打包了指令的文件夹：文件夹会包含Skill.md 的核心指令文件，以及Scripts,Reference和assets的可选资源文件。它教会AI如何处理制定的任务和流程。当需要处理重复的工作流，比如根据规范生成前端设计，创建符合团队风格指南的文档，多步骤流程编排等任务时，Skill可以让你无需每次对话重新解释你的偏好，流程，领域知识，而是通过Skill一次教会AI，之后每次都能受用。相比传统的提示词，它采用渐进式披露，动态加载提示词，更节约Token.



### 3.2 SKILL目录结构

从图中我们可以清晰的理解概念所说，一个SKILL就是一个文件夹。其中包含以下内容：

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-68.png)

1.SKILL.md：必须，包含了 Yaml头和Body。Yaml是md文件中的头，告诉AI是否加载这个SKILL，AI必定会加载。而body

是正文，是渐进式披露的，只有AI读取了YAML头，认为这个SKILL应该被加载，才会加载这个正文部分。

2.Scripts: 可选。包含着在AI执行过程中需要使用的脚本文件。

3.Refernces: 可选。存放执行SKILL相关的文档，说明，内容以MD为主。

4.Assets：可选。存放执行SKILL是需要的模板，静态资源，比如图片等。



2,3,4其实就是在SKILL执行过程中，所需要的资源。Anthropics 官方明确建议一个SKILL不应该超过500行，所以SKILL需要引用其它资源的时候，应该将这些东西放在SKILL.MD之外，而在SKILL.MD中显示指向它。



### 3.3 SKILL的常见应用场景：

1.生成一致的，高质量的内容，比如文档，PPT，app, design, 代码等。实例：[fronted-design skill](https://github.com/anthropics/skills/tree/main/skills/frontend-design)

2.编排多步流程，让这些步骤遵循特定的顺序，示例：[Skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)

3.规范/增强MCP使用，将MCP转换成可靠的工作流。实例：[Sentry-code-review skill](https://github.com/getsentry/sentry-for-claude/tree/main/skills)

（根据Anthropics官方指南，规范MCP的使用指的是用了Skill, 可以让用户/AI更清楚知道某个MCP Server 如何使用，遇到问题如何解决，取得稳定的结果。通常的最佳实践是在MCP Server的文档中附上SKILL的链接。用户在使用的多个MCP Server的时候，也可以通过SKILL来编排多个MCP的使用流。从这一点可以看出SKILL和MCP是相辅相成的，而不是替代关系。这一点是Anthropics官方指南，我觉得这一点比较新颖，也是一些视频没有提到的，面试时候说出一点SKILL和MCP的补充关系和最佳实践，可以加分）



### 3.4 SKILL测试与评估

通常可以从以下几个角度来测试Skill的质量：

1. 触发测试：保证SKILL会在正确的时机被加载。在不该触发的情况不会被误触发，在该触发的情况也不会不触发。

2. 功能测试：测试SKILL产生正确的结果：输出是符合预期的，API正确调用，错误被处理，边缘情况被覆盖。

3) 性能测试：对比使用SKILL和不适用SKILL时，用户的介入次数是否减少，API调用失败次数是否减少，Token消耗是否降低。



### &#x20;  3.5 SKILL典型范式

1. 顺序工作流编排： 当有多个步骤，并且需要按照指定的顺序执行时使用。

核心要点：说明步骤顺序，步骤间的依赖，每一步如何验证，失败时如何回滚。

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-69.png)

2. 多MCP 协作： 任务流包含多个MCP Server时使用。

核心要点：说明多MCP步骤间如何传递数据，错误异常处理，清晰的步骤界限；

![image-20260707012932733](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/image-20260707012932733.png)

3. 迭代优化： 需要通过多次迭代提升输出质量时：

核心要点：清晰的质量标准， 迭代提升，验证脚本，知道何时停止迭代。

![image-20260707012944792](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/image-20260707012944792.png)

4. 上下文感知工具选择： 当需要根据上下文场景，来动态选择使用不同的工具时使用

核心要点：清晰的选择策略，兜底选择，选择理由透明

![image-20260707012953427](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/image-20260707012953427.png)

* 领域专业智能：Skill需要使用特定领域的知识来完成任务时使用。

关键要点：将专业知识嵌入逻辑，合规性保证，详尽的领域文档，

![image-20260707013004380](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/image-20260707013004380.png)

### 3.6 SKILL  vs MCP

借用Anthropic 官网文档原话，Skills 和 MCP的核心差别是：MCP 提供给模型数据，而Skills教会模型如何使用这些数据（"MCP connects Calude to data, Skills teach Claude what to do with data)。二者的区别为：

1. MCP侧重于提供工具调用，而Skills则侧重于提示词。MCP更像是一个标准的工具箱，提供规范，可复用的工具。而Skills则是一个带目录的说明书。

2. MCP 的本质是一个独立运行的程序，而Skills 本质是一个文档。他们本质不同，决定了他们性质也不同，Skills更适合处理轻量级的，简单的逻辑，安全性和稳定性都不及MCP。

3. Skills 采用渐进式披露提示词的机制，Tokens消耗低。而MCP则是一次性塞入所有提示词，Token消耗高。

4. Skills的开发难度相比于MCP，开发难度低很多。

总结来说，SKILL和MCP二者不是相互替代，而是相互补充的。 在开发MCP SERVER的时候，可以在MCP Server的文档中添加一个链接，指向一个SKILL的链接，这个SKILL让用户/AI更清楚知道该MCP 如何使用，遇到问题如何解决，让模型调用MCP时取得更稳定的结果。在同时使用了多个MCP Server的时候，也可以用SKILL对多个编排多个MCP Server的工作流



### 3.7 SKILL vs Prompts

Skills 相比于Prompts的核心差别在于他们的**形态**和**生命周期**。

Prompts 是一次性对话内容，你需要自己保存，复制和粘贴。模型不会知道什么时候用哪一个Prompt，而且你需要把Prompts **全部塞到上下文里面**。这会大量占用你的上下文空间。

Skills 则是将提示词以一种**更加工程化的包装方式打包**，他是一个**文件夹的形式**，里面包含skills.md 和相关的资源文件。他会动**态加载/渐近式披露提示词，**&#x66F4;加节省提示&#x8BCD;**。**&#x5BB9;易复用和版本管理，易于协作和共享。

总结来说，Skills不是一个更复杂的提示词，而是把复杂的提示词，变成了一个**有结构，可复用，可以自动调用**的模块。

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-66.png)



### 3.8 SKILL的运行模式

1. **用户发起任务**
   &#x20;用户对 Claude Code 提需求，Claude Code 把这个请求连同当前可用的 Skills列表结合元数据一起交给 AI，让 AI 先基于需求判断该用哪类能力。

2. **AI 选择要用的 Skill**
   &#x20;AI 在可用的Skills里做匹配，然后返回指定的Skills

3. **按需加载 Skill 的指令层**

   一旦选定 skill，Claude Code 才把该 skill 的具体定义交给 AI, 这一步体现了Skills**指令层 按需加载**：只有用到这个 skill 时，才把更详细的指令灌给模型。

4. **按需加载 Skill 的资源层→ 读范文/材料 → 生成最终回答**

   如果需要使用到资源层，接下来 Claude Code 会执行读资源的动作，把 **references/范文内容**喂给 AI；AI 基于 用户材料 + skill 指令 + references 完成写作并输出 最终回答。



![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-67.png)





### 3.9 SKILL的设计理念

&#x20;   发展到现在（2026年初），现在的Agent工程的痛点不再是能不能做，而是能不能交付。目前的行业痛点是：

1.虽然模型能力很强，但是常识很弱，能解决通用问题，却在具体业务问题上频繁踩雷。经常做的是文本补全，而不是业务理解。这通常导致造出来的Agent需要质检，返工，人力成本高。

2.现在经常导致的一个问题是，公司重复造轮子。给财务做一个Agent，法务做一个Agent, 运营做一个Agent，每个Agent都有自己权限，工具，prompt等。就会变成同一个业务规则，在5个Agent里各写一遍，这就导致重复造轮子很严重，难以维护。

3.现在的Agent通常是一个会有复杂的prompts，这导致token太大，模型不能精准执行，另外成本上升。当你用复制粘贴扩展能力，系统就会用维护地狱来还债。

所以行业发展到现在，需要从堆砌提示词，迭代，调试，优化做一个Agent的过程变成做一个 可治理，可迭代，可复用的产品。通过Skills， 行业开发Agent的模式变成去写一个可复用，可管理的技能资产（Reusable skills), 结合通用内核，来解决这些痛点。