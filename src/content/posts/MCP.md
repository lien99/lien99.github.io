---
title: MCP的一些理解
published: 2025-04-20
draft: false
---

## 什么是MCP

英文全称：Model Context Protocol （模型上下文协议）。

MCP 是一种开放式**协议**，它规范了应用程序向 LLM提供上下文的方式。

简单来说，MCP就是一个标准，让不同的应用程序和AI模型可以更容易地交流和共享信息，而不需要为每个信息来源单独设计一套复杂的连接方式。

这样，AI模型就能更高效地工作，提供更好的服务。

![image-20250418155907968](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250418155907968.png)



### MCP Sever & Client

MCP server 是一个程序，暴露特定的功能或数据源，例如访问文件、数据库或 API，供 AI 模型使用。

MCP client 则也是一个程序，代表 AI 模型连接到这些服务器，允许模型请求和接收数据或执行操作。

![image-20250418160108992](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/image-20250418160108992.png)

**实际应用** 例如：

- 一个MCP server 可以提供天气信息，MCP client 则帮助 AI 模型通过该服务器获取天气预报，而无需自己处理数据获取逻辑。
- 另一个例子是，MCP server 可能允许 AI 访问用户电脑上的文件，MCP client 确保连接和权限管理，保护用户数据安全。



## 为什么是MCP

function calling、AI Agent、MCP 这三者之间有什么区别？

### Function Calling

- Function Calling 指的是 AI 模型根据上下文自动执行函数的机制。
- Function Calling 充当了 AI 模型与外部系统之间的桥梁，不同的模型有不同的 Function Calling 实现，代码集成的方式也不一样。由不同的 AI 模型平台来定义和实现。



如果我们使用 Function Calling，那么需要通过代码给 LLM 提供一组 functions，并且提供清晰的函数描述、函数输入和输出，那么 LLM 就可以根据清晰的结构化数据进行推理，执行函数。

Function Calling 的缺点在于处理不好多轮对话和复杂需求，适合边界清晰、描述明确的任务。如果需要处理很多的任务，那么 Function Calling 的代码比较难维护。

### MCP

- MCP 是一个标准协议，如同电子设备的 Type C 协议(可以充电也可以传输数据)，使 AI 模型能够与不同的 API 和数据源无缝交互。
- MCP 旨在替换碎片化的 Agent 代码集成，从而使 AI 系统更可靠，更有效。通过建立通用标准，服务商可以基于协议来推出它们自己服务的 AI 能力，从而支持开发者更快的构建更强大的 AI 应用。开发者也不需要重复造轮子，通过开源项目可以建立强大的 AI Agent 生态。
- MCP 可以在不同的应用/服务之间保持上下文，从而增强整体自主执行任务的能力。



可以理解为 MCP 是将不同任务进行分层处理，每一层都提供特定的能力、描述和限制。而 MCP Client 端根据不同的任务判断，选择是否需要调用某个能力，然后通过每层的输入和输出，构建一个可以处理复杂、多步对话和统一上下文的 Agent。

### AI Agent

- AI Agent 是一个智能系统，它可以自主运行以实现特定目标。传统的 AI 聊天仅提供建议或者需要手动执行任务，AI Agent 则可以分析具体情况，做出决策，并自行采取行动。
- AI Agent 可以利用 MCP 提供的功能描述来理解更多的上下文，并在各种平台/服务自动执行任务。



## MCP如何工作

![MCP 架构图](https://typora-1305283193.cos.ap-guangzhou.myqcloud.com/typora/MCP.png)

总共分为了下面五个部分：

- MCP Hosts: Hosts 是指 LLM 启动连接的应用程序，像 Cursor, Claude Desktop、[Cline](https://github.com/cline/cline) 这样的应用程序。
- MCP Clients: 客户端是用来在 Hosts 应用程序内维护与 Server 之间 1:1 连接。
- MCP Servers: 通过标准化的协议，为 Client 端提供上下文、工具和提示。
- Local Data Sources: 本地的文件、数据库和 API。
- Remote Services: 外部的文件、数据库和 API。

Server 如何理解呢？先从Cursor的发展历程理解：从 Chat 到 Composer 再进化到完整的 AI Agent

AI Chat 只是提供建议，如何将 AI 的 response 转化为行为和最终的结果，全部依靠人类，例如手动复制粘贴，或者进行某些修改。

AI Composer 是可以自动修改代码，但是需要人类参与和确认，并且无法做到除了修改代码之外的其它操作。

AI Agent 是一个完全的自动化程序，未来完全可以做到自动读取 Figma 的图片，自动生产代码，自动读取日志，自动调试代码，自动 push 代码到 GitHub。

而 MCP Server 就是为了实现 AI Agent 的自动化而存在的，它是一个中间层，告诉 AI Agent 目前存在哪些服务，哪些 API，哪些数据源，AI Agent 可以根据 Server 提供的信息来决定是否调用某个服务，然后通过 Function Calling 来执行函数。



## 如何使用MCP

首先推荐的是官方组织的一些 Server：[官方的 MCP Server 列表](https://github.com/modelcontextprotocol/servers)。

目前社区的 MCP Server 还是比较混乱，有很多缺少教程和文档，很多的代码功能也有问题，我们可以自行尝试一下 [Cursor Directory](https://cursor.directory/) 的一些例子。





Reference：

https://xiangyangqiaomu.feishu.cn/wiki/PXAKwEgLQir9rkkV1zjcGnMHntg?fromScene=spaceOverview

https://guangzhengli.com/blog/zh/model-context-protocol





