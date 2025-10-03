
# [Semantic Kernel 简介：定义以及它为何重要？](https://blog.nashtechglobal.com/introduction-to-semantic-kernels-what-they-are-and-why-they-matter/)

## 什么是 Semantic Kernel？

Semantic Kernel（语义内核，简称 SK）是微软开发的一个 SDK，它弥合了大型语言模型（LLMs）（如 OpenAI 的 ChatGPT）与我们用来开发与这种"语义硬件"通信软件的各种环境之间的差距。

SK 通过提供一种明确定义的方法来帮助管理对 LLMs 的访问，这种方法用于创建封装可由 LLMs 执行能力的插件。 Semantic Kernel 还可以在可用插件中进行选择以完成所需任务，这得益于规划器（Planner）—— Semantic Kernel 的一个关键组件。它提供了记忆功能，允许规划器从向量数据库等来源检索信息，并基于这些数据，提炼出使用特定插件执行的步骤计划。这使我们能够构建高度灵活、自主的应用程序。

![Semantic Kernel 架构图](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/semantic7-1024x491.png)

## Semantic Kernel 的关键组件

![Semantic Kernel 组件](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/semantic5-1024x436.png)

在使用 Semantic Kernel 开发解决方案时，有几个可用组件可以增强我们应用程序中的用户体验。虽然并非所有组件都是必需的，但建议熟悉它们。构成 Semantic Kernel 的组件如下：

### Memories 记忆

该组件通过回忆过去的对话，使我们的插件能够为用户问题提供上下文。实现记忆有三种方式：

- 键值对：这些存储方式与环境变量类似，传统搜索需要在键和用户输入之间存在直接的 1 对 1 关系。
- 本地存储：信息保存在本地文件系统中的文件里。当键值存储变得过大时，建议转换为这种类型的存储。
- 语义内存搜索：信息被表示为数值向量，即嵌入向量，从而实现更复杂的搜索功能。

### Planners 规划器

规划器是一个处理用户提示并生成执行计划以完成请求的函数。SDK 包含多个可供选择的预定义规划器：

- SequentialPlanner：构建一个包含多个相互连接函数的计划，每个函数都有自己的输入和输出变量。
- BasicPlanner：SequentialPlanner 的简化版本，将函数按顺序链接在一起。
- ActionPlanner：生成一个由单个函数组成的计划。
- StepwisePlanner：逐步执行每个步骤，在进入下一步之前评估结果。

### Connectors 连接器

连接器在 Semantic Kernel 中至关重要，作为各个组件之间的桥梁，促进信息交换。它们可以被设计为与外部系统交互，例如在我们的开发中集成 HuggingFace 模型或利用 SQLite 数据库进行记忆存储。

### Plugins 插件

插件由一组函数组成，这些函数可以是原生函数或语义函数，它们被提供给 AI 服务和应用程序使用。编写这些函数的代码仅仅是个开始；我们还需要提供关于它们行为、输入输出参数以及任何潜在副作用的语义细节。

- 语义函数：这些函数解释用户请求并以自然语言回应。它们依靠连接器来有效执行其任务。
- 原生函数：用 C#、Python 和 Java（目前处于实验阶段）编写，这些函数管理 AI 模型不适合处理的任务，例如：
  - 执行数学计算。
  - 从内存中读取和保存数据。
  - 访问 REST API。

## Semantic Kernel 中的编排

驱动 Semantic Kernel 编排的主要单元是规划器。 Semantic Kernel 包含内置规划器，如 SequentialPlanner，你也可以创建自定义规划器，如 teams-ai 包中所示。

规划器作为一个提示运行，它使用语义函数或原生函数的描述来确定要执行哪些函数以及执行顺序。因此，函数及其参数的描述必须简洁而信息丰富，这一点至关重要。根据应用中可用的函数，这些描述必须进行定制，以确保其独特性，并与应用需要执行的特定任务相关。

![编排](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/semantic6.png)

## 示例代码

首先，我们需要从 nuget 包管理器安装 Microsoft.SemanticKernel。

在 Semantic Kernel 中，所有内容都通过 IKernel 接口流动，它代表了与 Semantic Kernel 的主要交互点。

要创建一个 IKernel，你需要使用 KernelBuilder，它遵循了在 .NET 其他地方（如 ConfigurationBuilder 和 Iconfiguration）中常用的构建器模式。

以下是一个创建 IKernel 并向其添加一些基本服务的示例：

![创建 IKernel](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/semantic2.png)

此内核与一个或多个语义函数或自定义插件协同工作，并使用它们生成响应。这个特定的内核使用 ConsoleLogger 进行日志记录，并连接到 Azure OpenAI 模型部署来生成响应。

### 语义函数

语义函数是一种模板化函数，为大型语言模型提供生成响应的框架。在一个大型项目中，你可能需要各种语义函数来支持应用程序需要处理的不同类型的任务。

下面是一个语义函数的示例，它训练机器人采用蝙蝠侠的角色并作出相应的回应：

![语义函数](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/semantic3.png)

### 调用语义函数

你可以通过向内核提供函数和文本来调用语义函数。

在这里，我们将提示用户输入一个字符串，然后将其发送到内核：

![调用语义函数](https://blog.nashtechglobal.com/wp-content/uploads/2024/06/semantic4.png)

这里会允许用户的文本被添加到提示模板中，整个字符串随后被发送到 LLM 以生成响应。

## 为什么使用 Semantic Kernel ？

- 与大型语言模型（LLMs）的集成：Microsoft Semantic Kernel SDK 提供了一种结构化方法，用于将 LLMs（如 OpenAI 的 GPT）集成到你的应用程序中。这使你能够利用这些模型强大的自然语言处理能力，而无需处理集成的复杂性。
- 插件架构：SK 提供了一种插件架构，允许你将特定功能封装到插件中，从而更轻松地管理和扩展应用程序的功能。这种模块化方法促进了代码重用和可维护性。
- 任务规划与编排：通过 SK 的规划器组件，你可以定义任务并使用可用插件自动生成完成这些任务的计划。这使得能够创建动态和自适应的应用程序，以响应不断变化的需求和环境。
- 内存管理：SK 包含内存管理功能，使你的应用程序能够存储和检索上下文信息。这支持上下文感知的交互和决策，增强了应用程序的智能性和自主性。
- 灵活性和可扩展性：通过提供管理 LLMs 访问和编排任务的明确定义框架，SK 在开发 AI 驱动应用程序方面提供了灵活性和可扩展性。无论你是构建简单的聊天机器人还是复杂的虚拟助手，SK 都能适应你的需求并随你的项目扩展。
- 微软支持与生态系统：作为微软开发的产品，SK 受益于这家领先技术公司的支持与专业知识。此外，它与微软其他技术和服务（如 Azure、Visual Studio 和 Azure 认知服务）无缝集成，增强了开发体验以及与现有解决方案的互操作性。
- 开源与社区驱动：SK 是开源的，允许开发者社区进行透明、协作和贡献。这促进了创新，并确保 SDK 能够跟上人工智能和自然语言处理的最新进展。

## 总结

Microsoft Semantic Kernel SDK 为开发人员提供了一个强大的框架，用于将大型语言模型（LLMs）集成到他们的应用程序中。凭借其插件架构、任务规划能力和内存管理功能，该 SDK 使开发人员能够创建智能的、具有上下文感知能力的应用程序。得益于 Microsoft 的专业知识和生态系统支持，它提供了灵活性、可扩展性以及与其他 Microsoft 技术的无缝集成。作为一个开源项目，它鼓励协作和创新，使其成为构建能够有效理解和处理人类语言的 AI 驱动应用程序的宝贵工具。

今天就到这里。期待与你再次相见。
