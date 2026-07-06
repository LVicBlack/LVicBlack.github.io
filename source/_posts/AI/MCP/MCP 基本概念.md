---
title: MCP 基本概念
date: 2025-06-06 20:06:00
categories: 
- AI
- MCP
tags:
- MCP
---

### 1 MCP 和 Function calling 的区别

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-19.png)

### &#x20;2 MCP 本地Server和远端Server 的对比

![](https://cdn.jsdelivr.net/gh/LVicBlack/IMG/root/2026/7/%E5%A4%A7%E6%A8%A1%E5%9E%8B%E5%BA%94%E7%94%A8!%E7%AE%97%E6%B3%95%E5%AD%A6%E4%B9%A0%E8%B7%AF%E7%BA%BF+%E5%85%AB%E8%82%A1+%E9%9D%A2%E8%AF%95%E5%AE%9E%E6%88%98_3-image-20.png)

### 3 MCP的传输协议

(最开始我复习的时候就是弄清楚了两种协议，本地:Stdio 远端：SSE，这也是有一些视频这么讲的。但我在面试的时候，其实因为这个答案被挑战过，因为现在SSE在MCP里已经被替代了，所以我有了以下答案，相对来说这个答案是比较完整的，这题也是面试会考的题）

MCP的传输协议针对于**本地服务器**和**远端服务器**有明显区别。

对于远端MCP Server, 协议在 2025 年 3 月进行了重大更新：原先采用 **有状态的 HTTP 长连接**，并强制使用 **Server-Sent Events (SSE)** 来实现服务器实时推送，这种方式复杂且对网络稳定性要求高。新版本改为 **无状态协议**，主要通过 **标准的 GET 和 POST HTTP 请求**完成交互，SSE 变为可选增强功能，只有在需要实时推送时才启用。这一改动降低了实现复杂度，提高了兼容性和灵活性，使协议更接近 REST 风格，易于集成和扩展。

对于 **本地 MCP Server**，通常运行在开发者机器上，直接通过本地接口或 STIDO（标准 I/O）进行通信，不涉及网络传输，因此不需要复杂的认证或长连接管理。



### 4 MCP Server Tool设计原则

> **资料来源**：MCP 官方规范 \[modelcontextprotocol.io/specification/2025-06-18/server/tools]\(<u>https://modelcontextprotocol.io/specification/2025-06-18/server/tools</u>)

**第一，入参设计——参数最少化、类型严格、约束内联。** LLM 是根据 inputSchema 来构造参数的，schema 本身就是"给 LLM 看的说明书"。原则是：必填参数尽量少，能给默认值就给默认值，用枚举收敛可选项而不是让 LLM 自由发挥，类型要明确。约束直接写在 schema 里，比如最小值、最大值、默认值，这样 Server 端校验方便，LLM 传参准确率也更高。我们项目里查询工具的返回条数参数就限定了范围和默认值，LLM 不传就用默认值，传了超出范围直接截断。

**第二，出参设计——结构化返回，LLM 友好。** MCP 规范支持 outputSchema 声明返回结构，让 Client 和 LLM 提前知道数据长什么样，做类型校验和自动解析。同时规范要求向后兼容——结构化返回的同时要附带一个纯文本版本，保证老 Client 也能读。我们项目的做法是统一格式化输出，把检索结果转成带标题、来源引用、置信分数的可读文本，而不是丢裸 JSON 给 LLM——因为 LLM 读自然语言比读嵌套 JSON 效率高得多。

**第三，错误处理——双层错误机制，错误信息对 LLM 友好。** MCP 规范定义了两层错误：**协议级错误**用标准 JSON-RPC error code 处理参数非法、tool 找不到等问题；**业务级错误**通过 \`isError: true\` 加 error message 报告。关键原则是：**错误信息要告诉 LLM 下一步该怎么办，而不是丢一堆堆栈。** 比如返回"集合不存在，可用的集合有 A、B、C"，而不是返回 Python 异常堆栈。前者 LLM 能自主修正重试，后者只能把堆栈原样输出给用户看。

**第四，Tool Annotations——声明副作用，让 Client 做安全决策。** MCP 规范定义了 annotations 字段，包含四个关键属性：是否只读、是否有破坏性、是否幂等、是否与外部世界交互。这些标注不影响执行逻辑，但 **Client 会据此决定是否需要弹确认框**。比如只读的查询工具可以自动执行，但有破坏性的删除工具就应该让用户二次确认。我们项目的三个 tool 全部是只读查询，都应该标注为只读。

**第五，粒度控制——单一职责，工具间可组合。** MCP 的设计是 model-controlled——LLM 自己决定调用顺序和组合方式。所以工具拆分的粒度很关键：太粗 LLM 无法灵活组合，太细调用链路过长容易出错。我们项目拆了三个 tool：列出集合 → 查看文档详情 → 查询知识库，形成"先发现 → 再了解 → 最后查询"的自然工作流，上游的输出可以直接作为下游的输入参数。

**第六，安全与校验——输入校验和输出脱敏。** MCP 规范明确要求 Server 必须校验所有输入、实现访问控制、限流、清洗输出。即使 LLM 传了符合 schema 的参数，Server 端仍然要做业务层面的校验——比如路径穿越检查、注入防护、敏感信息脱敏。同时规范强调"始终要有人类在回路中"，高危操作必须有确认机制。

**总结来说**，MCP tool 的编写是"六位一体"的：

- description 决定 LLM 能不能找到你
- inputSchema 决定传参准不准
- outputSchema 决定结果可不可解析
- error handling 决定失败后能不能自愈
- annotations 决定 Client 信不信任你
- 粒度和安全则决定整个系统是否健壮可控。



### 5 MCP的参考资料

https://www.bilibili.com/video/BV1uronYREWR/

https://www.bilibili.com/video/BV1Y854zmEg9/

https://www.bilibili.com/video/BV1v9V5zSEHA/