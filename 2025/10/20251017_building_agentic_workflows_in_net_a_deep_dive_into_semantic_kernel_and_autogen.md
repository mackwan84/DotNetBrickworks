# [在 .NET 中构建智能体工作流：深入探索 Semantic Kernel 和 AutoGen](https://sairam6381.medium.com/building-agentic-workflows-in-net-a-deep-dive-into-semantic-kernel-and-autogen-cfeacd5bee00)

我们都曾为单次调用大型语言模型（LLM）所能实现的功能而着迷。从生成代码片段到起草电子邮件，其力量不可否认。但是，当任务过于复杂，无法一次性生成时，会发生什么？如果你需要一个 AI 编写代码，另一个审查代码，第三个编写测试，第四个记录最终结果，该怎么办？

这不是对未来的一瞥；这是由智能体工作流驱动的 AI 开发新前沿。最棒的是什么？你可以在强大而熟悉的 .NET 世界中直接构建这些复杂、协作的 AI 系统。

在本文中，我们将探讨如何结合两个强大的框架 —— Microsoft 的 Semantic Kernel 和 AutoGen——来构建你的第一个“AI 团队”。我们将揭开这些概念的神秘面纱，理解它们如何协同工作，并通过一个实际的例子，演示如何构建一个协作的编码员和评论员智能体组合。

## 首先，AI 智能体究竟是什么？

在构建团队之前，让我们先了解单个成员。在 LLMs 的背景下，AI 智能体不仅仅是一个聊天机器人。可以将其视为一个为实现特定目标而设计的自主系统。它包含四个关键组成部分：

1. 大脑（核心逻辑）：这是一个强大的 LLM（如 GPT-4），提供推理、语言理解和决策能力。
2. 目标（指令）：智能体有明确的执行目标。这可以是任何任务，从“总结这份文档”到“编写一个功能、创建拉取请求，并在测试成功后合并它”。
3. 工具（能力）：这正是将智能体提升到超越简单 LLM 调用的关键。智能体可以访问各种工具——如 API、文件系统、数据库或网络搜索——使其能够与外部世界交互、收集信息并采取行动。
4. 记忆（上下文）：智能体可以同时拥有短期记忆（当前对话的历史记录）和长期记忆（一个用于从过往交互中检索相关信息的向量数据库）。

简而言之，AI 智能体就是一个被赋予了目标、工具和记忆能力的 LLM，使其能够自主执行复杂的多步骤任务。

![一个简单的示意图，展示了一个 AI Agent 及其四个组成部分：大脑（LLM）、目标、工具和记忆。](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*DgFyITeE3EcwW3aNdbCzkQ.png)

## 从单一智能体到协作团队

一个智能体固然很棒，但现实世界的问题通常需要多样化的技能和视角。单个软件开发人员通常不会一次性完成编码、测试和部署，而是在没有任何反馈的情况下完成所有工作。他们会进行协作。

这就是智能体工作流的用武之地。你不再创建一个试图包揽一切的单一庞大智能体，而是创建一个由专业智能体组成的团队，通过协作来解决问题。

- PlannerAgent 可能会将用户的请求分解为一系列步骤。
- CoderAgent 会接收一个步骤并编写必要的代码。
- CriticAgent 会审查代码的错误、风格和正确性。
- ExecutorAgent 随后可以运行代码或测试。

这种方法更加稳健、可扩展，并且反映了人类团队的工作方式。但是协调这种对话可能会很复杂。你如何管理谁在何时发言？你如何知道目标何时达成？这就是我们的.NET 梦之队发挥作用的地方。

## .NET 梦之队：Semantic Kernel 和 AutoGen

要在 C# 中构建这些工作流，我们可以结合两个令人难以置信的开源框架。它们不是竞争关系，而是完美地互补彼此。

### 1. Semantic Kernel：智能体的大脑和工具包

可以将 Semantic Kernel (SK) 视为构建高能力独立智能体的 SDK。它提供了基础架构，用于为智能体配备技能和智能。

- 内核：处理请求的核心编排器。
- 插件（以前称为技能）：这些是智能体的工具。插件是函数的集合。可以是“语义函数”（自然语言提示）或“原生函数”（你的 C# 代码）。通过这种方式，你可以让智能体具备写入文件、调用 API 或搜索数据库的能力。
- 规划器：SK 内置了规划器，能够接收高级目标，并使用可用插件自动生成分步计划。

本质上，你使用 Semantic Kernel 来构建一个单一的、强大的、工具增强的智能体。

### 2. AutoGen：协作框架

如果说 SK 构建了参与者，那么 AutoGen 则构建了会议室并设定了对话规则。AutoGen 是一个用于简化多个智能体之间对话编排的框架。

- 可对话智能体：它为能够发送和接收消息的智能体提供了高级抽象。
- 群聊自动化：你可以定义一组智能体及其交互规则（例如，轮询方式）。这些智能体相互交流，直到满足终止条件（比如最终代码获得批准）。

AutoGen 管理对话中的"谁"和"何时"，使你的专业智能体能够有效协作。

### 为何它们结合更佳 这种协同作用非常美妙

你使用 Semantic Kernel 构建每个智能体的功能。你使用 AutoGen 定义它们如何交互。想象一下，在 SK 中创建一个带有访问 GitHub 和运行 `dotnet build` 插件的 `DevOpsAgent`。然后，你使用 AutoGen 将其与 `ProductManagerAgent` 放入对话中，从而自动化你的整个发布周期。

## 让我们开始构建！用 C# 实现编码员与评论员组合

理论固然重要，但代码更有说服力。让我们通过你提供的代码来逐步构建我们的 AI 团队。我们将创建一个编写 C# 代码的`CoderAgent`和一个审查代码的`CriticAgent`，循环往复直到代码获得批准。

目标：用户将请求一段 C# 代码。`CoderAgent`将编写初稿。`CriticAgent`将对其进行审查，提供反馈，直到代码令人满意为止。

我们的团队：

1. CoderAgent：其工作是根据请求编写 C# 代码。它由 Semantic Kernel 驱动，并拥有生成 C# 代码的工具。
2. CriticAgent：其工作是审查代码。它会检查代码的正确性、潜在错误以及对最佳实践的遵循情况。

先决条件：

确保已安装必要的 NuGet 包：

```bash
dotnet add package AutoGen.SemanticKernel 
dotnet add package Microsoft.SemanticKernel 
dotnet add package Microsoft.SemanticKernel.Connectors.AzureOpenAI
```

### 步骤 1：使用 Semantic Kernel 打造编码器的核心工具

首先，我们需要为我们的编码器智能体提供其主要技能：编写 C# 代码的能力。我们不只是想发送一个通用提示；我们想为这个特定任务创建一个可靠、可重用的工具（在 SK 术语中是`KernelFunction`）。我们首先定义一个详细的提示，告诉 LLM 确切的行为方式。

```csharp
// 初始化 Semantic Kernel
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion("Model_Id", "End_Point", "Api_Key")
    .Build();

// 定义提示，指定任务
var prompt = """
You are an expert C# developer. Your role is to write clean, efficient, and correct C# code based on the user's request.
ONLY output the raw C# code inside a ```csharp ... ``` block. Do not include any surrounding text, explanations, or other markdown formatting.

Request: {{ $input }}
""";

// 转换提示为可重用函数，并将其打包在插件中
var generateCodeFunction = kernel.CreateFunctionFromPrompt(prompt, functionName: "CSharp_Code_Generator");
var codePlugin = kernel.CreatePluginFromFunctions("CodeWriterPlugin", "Plugins to generate program code", [generateCodeFunction]);

// 将插件添加到内核的工具包中
kernel.Plugins.Add(codePlugin);
```

我们刚刚做了什么？

- 我们为代码生成任务定义了一套清晰的指令。`{{$input }}` 是实际请求的占位符。
- 我们使用 `CreateFunctionFromPrompt` 将该文本转换为 Semantic Kernel 中可调用的函数。
- 然后，我们将该函数包装在一个名为 CodeWriterPlugin 的 KernelPlugin 中。这至关重要，因为它使工具可以通过名称被发现和调用，我们将在智能体的指令中使用这一点。

### 步骤 2：定义智能体角色

现在我们创建我们的智能体。我们将使用 AutoGen 库中的`SemanticKernelAgent`，它在两个框架之间充当完美的桥梁。最重要的部分是`systemMessage`，它为每个智能体提供了其在对话中的个性、规则和目标。

#### CoderAgent

```csharp
var coderAgent = new SemanticKernelAgent(
    kernel: kernel,
    name: "Coder Agent",
    systemMessage: """
    You are a C# Coder Agent. Your task is to generate C# code using the CodeWriterPlugin.CSharp_Code_Generator function based on the user's request.
    - Output ONLY the generated C# code block.
    - Do NOT describe your actions or narrate your process.
    - If the Critic Agent replies with feedback, revise the code accordingly and resubmit ONLY the updated C# code block.
    """,
    new PromptExecutionSettings()
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.Required(),
    }
).RegisterMessageConnector().RegisterPrintMessage();
```

- 角色：它的`systemMessage`告诉它如何在聊天中表现。关键是，它被指示使用我们刚刚创建的`CSharp_Code_Generator`工具。
- 行为：`FunctionChoiceBehavior.Required`是一个强大的设置，它强制智能体使用其工具，而不是尝试从其一般知识中编写代码。这使其更加可预测和可靠。

#### CriticAgent

```csharp
var criticAgent = new SemanticKernelAgent(
    kernel: kernel,
    name: "Critic Agent",
    systemMessage: """
    You are a C# code critic agent. Your role is to review the code provided by the Coder Agent.
    - If the code is correct, well-written, and meets the request, reply with ONLY the final C# code block. Do not add any other text, comments, or markdown(```csharp).
    - If the code has errors, is incomplete, does not meet the request, or if the original instructions are missing, ambiguous, or unclear, you MUST reply with the prefix 'FeedBack:' followed by specific, constructive feedback.
    """,
    new PromptExecutionSettings()
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.None(),
    }
).RegisterMessageConnector().RegisterPrintMessage();
```

- 角色设定：它的规则是我们工作流的引擎。通过回复 `”FeedBack:”` 或仅回复代码，它清晰地表明了任务的状态
- 行为：我们将其 `FunctionChoiceBehavior` 设置为 `None`，因为它的职责是推理和回复，而不是调用外部工具。

### 步骤 3：使用 AutoGen 编排协作式聊天

定义好我们的智能体后，我们只需要将它们放入一个"房间"并给它们一个任务。这正是 AutoGen 的专长。

```csharp
// 创建一个群聊来主持对话
var groupChat = new GroupChat(members: [coderAgent, criticAgent]);

// 定义初始任务，启动对话
var initialTask = new TextMessage(
    Role.User, 
    from: "Critic Agent", // 第一条消息
    content: "Please write a C# function to calculate the factorial of a number."
);

string finalResult = string.Empty;

// 开启对话并循环处理消息
await foreach (var message in groupChat.SendAsync([initialTask]).ConfigureAwait(false))
{
    // 这是我们的终止条件！
    // 如果评论者回复且其消息不包含“FeedBack”，则表示代码已获批准。
    if ((message?.From == "Critic Agent") && !message.GetContent()!.Contains("FeedBack", StringComparison.OrdinalIgnoreCase))
    {
        finalResult = message.GetContent()!;
        break; // Exit the loop
    }
}
Console.WriteLine($"Final Approved Code:\n{finalResult}");
```

这最后一部分就是编排器。

- `GroupChat`是智能体们进行交互的"会议室"。
- 我们通过最初请求一个阶乘函数来启动工作流程。
- `await foreach` 循环是神奇之处所在。AutoGen 根据系统消息中的规则，管理 Coder 和 Critic 之间的来回交互。
- `if` 语句是我们的退出条件。循环会一直持续，直到 Critic 智能体通过不带“FeedBack:”前缀的响应给出最终批准。这就是我们判断任务已成功完成的方式。

通过将 Semantic Kernel 强大的工具构建能力与 AutoGen 优雅的对话管理相结合，我们在 .NET 内部创建了一个动态的、自我修正的工作流。

## 从开发者到导演的旅程

我们在这里构建的不仅仅是一个巧妙的演示；它代表了作为开发者的我们角色的根本转变。我们正从代码的唯一作者转变为 AI 劳动力的指挥者。你的新核心技能将是设计角色、定义清晰的通信协议，以及为你的智能体构建强大的工具。

那么，你应该从哪里开始呢？

1. 从一个优秀的智能体开始：在编排团队之前，先用 Semantic Kernel 掌握构建一个高效智能体的方法。给它一个自定义的 C# 工具——也许是一个与 HttpClient 交互以调用 API 的插件，或者一个使用 System.IO 读取项目结构的插件。一个强大的智能体是强大团队的基础。
2. 像管理者一样思考："系统提示"是你为智能体制定的工作描述。花时间完善它。目标是否清晰？互动规则是否明确？你将如何衡量成功？
3. 设计工作流：AutoGen 的魔力在于对话。思考一下流程。谁需要和谁交流？什么退出条件表示任务已完成？像我们的编码员和评论员这样的简单循环是一个很好的开始，但你可以设计更复杂的工作流。

Semantic Kernel 和 AutoGen 的结合为 .NET 开发者提供了独特的优势——能够将强大的静态类型原生代码与 LLMs 的流畅推理相结合。下一个伟大的应用程序不仅仅是被编写出来的，它将被精心编排。去构建你的第一个智能体吧。

今天就到这里。期待与你再次相见。
