# [Microsoft Agent Framework：让每位开发者都能轻松构建 AI 智能体](https://devblogs.microsoft.com/dotnet/introducing-microsoft-agent-framework-preview/)

构建 AI 智能体不应该是火箭科学。然而，许多开发者发现自己正在与复杂的编排逻辑作斗争，努力连接多个 AI 模型，或者花费数周时间构建托管基础设施，只是为了将一个简单的智能体投入生产。

如果构建 AI 智能体能像创建 Web API 或控制台应用程序一样简单，那会怎样？

## 智能体和工作流

在我们深入探讨之前，让我们定义智能体系统的两个核心构建块：智能体和工作流。

### 智能体

在网络上，你会发现许多关于智能体的定义。有些相互冲突，有些相互重叠。

在这里，我们将定义简单化：**智能体是能够实现目标的系统。**

![智能体](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2025/09/agent.png)

当具备以下条件时，智能体的能力会更强：

- 推理和决策能力：由 AI 模型（LLMs）、搜索算法或规划和决策系统提供支持。
- 工具使用：访问模型上下文协议（MCP）服务器、代码执行和外部 API。
- 上下文感知能力：通过聊天历史、线程、向量存储、企业数据或知识图谱获取信息。
- 这些能力使智能体能够更自主、更自适应、更智能地运行。

### 工作流

随着目标变得越来越复杂，需要将其分解为可管理的步骤。这就是工作流的用武之地。

**工作流定义了实现目标所需的步骤序列。**

想象一下，你正在业务网站上推出一个新功能。如果只是简单的更新，你可能会在几小时内从想法到上线。但对于更复杂的计划，过程可能包括：

- 需求收集
- 设计与架构
- 实现
- 测试
- 部署

![工作流](https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2025/09/workflow.png)

几个重要观察：

- 每个步骤可能包含子任务。
- 不同的专家可能负责不同的阶段。
- 进展并不总是线性的。测试过程中发现的错误可能会让你回到实现阶段。
- 成功取决于规划、协调以及与利益相关者之间的沟通。

### 智能体 + 工作流

工作流不需要智能体，但智能体可以为其注入强大动力。

当智能体具备推理、工具和上下文能力时，它们可以优化工作流程。

这是多智能体系统的基础，智能体在工作流程中协作以实现复杂目标。

## 认识 Microsoft Agent Framework

Microsoft Agent Framework 是一套全面的 .NET 库，可降低智能体开发的复杂性。无论你是在构建简单的聊天机器人，还是在复杂工作流程中协调多个 AI 智能体，Microsoft Agent Framework 都能为你提供所需的工具：

- 使用最少的样板代码构建智能体
- 轻松编排多智能体工作流
- 使用熟悉的 .NET 模式托管和部署智能体
- 在生产环境中监控和观察智能体行为

## 建立在经过验证的基础之上

Microsoft Agent Framework 利用成熟的技术为 .NET 开发人员简化智能体开发：

- Semantic Kernel – 提供强大的编排功能
- AutoGen – 实现先进的多智能体协作和前沿研究驱动技术
- Microsoft.Extensions.AI – 为 .NET 提供标准化的 AI 构建块。

![Agent Framework 基础](https://devblogs.microsoft.com/dotnet/introducing-microsoft-agent-framework-preview/agent-framework-foundations.png)

通过结合这些技术，Agent Framework 提供了可靠性、灵活性和对开发者友好的 API。这使你能够快速高效地构建和部署强大的 AI 智能体。

## 几分钟内构建你的第一个智能体

开始使用 Microsoft Agent Framework 非常简单。在以下示例中，你将构建一个创意写作智能体，生成引人入胜的短篇故事。

### 步骤 0：配置先决条件

要开始使用，你需要以下内容：

- .NET 9 SDK 或更高版本
- 具有 models 作用域的 GitHub 个人访问令牌(PAT)。你可以在 GitHub 设置中创建一个。

此项目可以使用 GitHub 托管的模型，因此你需要使用 GITHUB_TOKEN 环境变量向应用程序提供 GitHub 个人访问令牌 (PAT)。

## 步骤 1：设置你的项目

创建一个新的 C#控制台应用程序并安装 Agent Framework 包：

```bash
dotnet new console -o HelloWorldAgents
cd HelloWorldAgents
dotnet add package Microsoft.Agents.AI --prerelease
```

你还需要安装以下软件包才能使用 GitHub 模型中的模型。

```bash
dotnet add package OpenAI
dotnet add package Microsoft.Extensions.AI.OpenAI --prerelease
dotnet add package Microsoft.Extensions.AI
```

### 步骤 2：编写你的智能体

将此代码添加到你的 Program.cs 文件中，以创建一个故事编写智能体：

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Agents.AI;
using OpenAI;
using OpenAI.Chat;
using System.ClientModel;

IChatClient chatClient =
    new ChatClient(
            "gpt-4o-mini",
            new ApiKeyCredential(Environment.GetEnvironmentVariable("GITHUB_TOKEN")!),
            new OpenAIClientOptions { Endpoint = new Uri("https://models.github.ai/inference") })
        .AsIChatClient();

AIAgent writer = new ChatClientAgent(
    chatClient,
    new ChatClientAgentOptions
    {
        Name = "Writer",
        Instructions = "Write stories that are engaging and creative."
    });

AgentRunResponse response = await writer.RunAsync("Write a short story about a haunted house.");

Console.WriteLine(response.Text);
```

运行你的应用程序

就是这样！只需几行代码，你就有了一个功能齐全的 AI 智能体。

### 抽象的力量

Microsoft Agent Framework 是围绕强大的抽象概念设计的，这些抽象概念简化了智能体开发。

其核心是 AIAgent 抽象，它为构建智能体提供了统一接口。能够使用任何兼容的 AI 模型提供商的灵活性来自于 Microsoft.Extensions.AI，它通过 IChatClient 接口标准化了模型访问。

ChatClientAgent 的实现接受任何 IChatClient ，让你可以轻松选择以下提供商：

- OpenAI
- Azure OpenAI
- Foundry Local
- Ollama
- GitHub Models
- 还有其他

这意味着你可以更换提供商或集成新的提供商，而无需更改你的智能体代码。相同的 AIAgent 接口可与以下内容无缝协作：

- Azure Foundry Agents
- OpenAI Assistants
- Copilot Studio
- 还有其他

如果你曾使用不同的 SDK 或平台构建过智能体，使用 Microsoft Agent Framework 将非常简单。你将获得一致的开发体验，并可以利用编排、托管和监控功能。

## 扩展规模：编排多个智能体

单个智能体功能强大，但现实场景通常需要多个专业智能体协同工作。也许你的写作智能体能创作出优秀内容，但你还需要一个编辑来润色它，或一个事实核查员来验证细节。

智能体框架使多智能体编排像连接积木一样简单。

### 添加专业智能体

让我们通过添加一个编辑智能体来审查和改进作者的输出，从而增强我们的故事示例：

```csharp
// 创建一个专业的编辑智能体
AIAgent editor = new ChatClientAgent(
    chatClient,
    new ChatClientAgentOptions
    {
        Name = "Editor",
        Instructions = "Make the story more engaging, fix grammar, and enhance the plot."
    });
```

### 构建工作流

现在到了神奇的部分——在工作流中连接这些智能体。

安装 Microsoft.Agents.Workflows NuGet 包：

```bash
dotnet add package Microsoft.Agents.AI.Workflows --prerelease
```

创建一个工作流来协调你的智能体：

```csharp
// 创建一个将作者连接到编辑的工作流
Workflow workflow =
    AgentWorkflowBuilder
        .BuildSequential(writer, editor);

AIAgent workflowAgent = await workflow.AsAgentAsync();

AgentRunResponse workflowResponse =
    await workflowAgent.RunAsync("Write a short story about a haunted house.");

Console.WriteLine(workflowResponse.Text);
```

### 结果

现在当你运行应用程序时，编写器会创建初始故事，编辑器会自动审查并改进它。整个工作流程在外界看来就像一个单一但功能更强大的智能体。

![工作流](https://devblogs.microsoft.com/dotnet/introducing-microsoft-agent-framework-preview/writer-editor-workflow.png)

这种模式可扩展至任何复杂程度：

- 研究工作流：研究员 → 事实核查员 → 摘要撰写员
- 内容管道：撰稿人 → 编辑 → SEO 优化师 → 发布者
- 客户服务：意图分类 → 客户专员智能体 → 质量审核员

这里的关键要点是，复杂的智能体系统是由简单、专注的智能体组合而成的。

### 所有类型的工作流

上面的示例使用了顺序工作流，其中智能体依次处理任务，每个智能体都在前一个智能体的输出基础上构建。然而，Microsoft Agent Framework 支持多种工作流模式以适应不同需求：

- 顺序：智能体按顺序执行，沿着链传递结果。
- 并发：多个智能体并行工作，同时处理任务的不同方面。
- 交接：责任根据上下文或结果在智能体之间转移。
- 群聊：智能体在共享的实时对话空间中进行协作。

这些灵活的工作流类型能够实现从简单管道到动态多智能体协作的各种编排。

## 为智能体配备工具

Microsoft Agent Framework 让你可以轻松地为智能体提供对外部函数、API 和服务的访问权限，从而使你的智能体能够采取行动。

### 创建智能体工具

让我们通过一些有用的工具来增强我们的写作智能体。在我们的故事示例中，我们可能希望智能体能够执行以下操作：

```csharp
[Description("Gets the author of the story.")]
string GetAuthor() => "Jack Torrance";

[Description("Formats the story for display.")]
string FormatStory(string title, string author, string story) =>
    $"Title: {title}\nAuthor: {author}\n\n{story}";
```

### 将工具连接到智能体

向你的智能体添加这些工具非常简单。以下是我们的写入智能体的修改版本，该版本已配置为使用前面定义的工具。

```csharp
AIAgent writer = new ChatClientAgent(
    chatClient,
    new ChatClientAgentOptions
    {
        Name = "Writer",
        Instructions = "Write stories that are engaging and creative.",
        ChatOptions = new ChatOptions
        {
            Tools = [
                AIFunctionFactory.Create(GetAuthor),
                AIFunctionFactory.Create(FormatStory)
            ],
        }
    });
```

再次运行应用程序将根据你在 FormatStory 中提供的模板生成一个格式化的故事。

```markdown
**Title: The Haunting of Blackwood Manor**
**Author: Jack Torrance**

On the outskirts of a quaint village, a grand but crumbling mansion, known as Blackwood Manor, loomed against the twilight sky. Locals spoke in hushed tones about the house, claiming it was haunted by the spirits of its former inhabitants, who had mysteriously vanished decades ago. Tales of flickering lanterns, echoing whispers, and the ghostly figure of a woman in white gliding through the halls filled the village’s atmosphere with a sense of dread
//...
```

### 超越简单功能

由于 Microsoft Agent Framework 构建于 Microsoft.Extensions.AI 之上，你的智能体可以使用更强大的工具，包括：

- 模型上下文协议 (MCP) 服务器 – 连接到外部服务，如数据库、API 和第三方工具
- 托管工具 – 访问服务器端工具，如代码解释器、Bing 基础服务等众多工具

例如，你可以连接到提供数据库访问、网络搜索甚至硬件控制的 MCP 服务器。在我们的 MCP 客户端快速入门中了解有关构建 MCP 集成的更多信息。

## 自信地部署：让托管变得简单

将智能体投入生产环境不应意味着学习新的部署模型。Microsoft Agent Framework 与你已经使用的 .NET 托管模式无缝集成。

### 集成 Minimal Web API

在 REST API 中使用你的智能体只需几行代码。

在 ASP.NET Minimal Web API 中，首先注册你的 IChatClient ：

```csharp
builder.AddOpenAIClient("chat")
    .AddChatClient(Environment.GetEnvironmentVariable("MODEL_NAME")!);
```

使用 Microsoft.Agents.AI.Hosting NuGet 包注册你的智能体：

```csharp
builder.AddAIAgent("Writer", (sp, key) =>
{
    var chatClient = sp.GetRequiredService<IChatClient>();

    return new ChatClientAgent(
        chatClient,
        name: key,
        instructions:
            """
            You are a creative writing assistant who crafts vivid, 
            well-structured stories with compelling characters based on user prompts, 
            and formats them after writing.
            """,
        tools: [
            AIFunctionFactory.Create(GetAuthor),
            AIFunctionFactory.Create(FormatStory)
        ]
    );
});

builder.AddAIAgent(
    name: "Editor",
    instructions:
        """
        You are an editor who improves a writer’s draft by providing 4–8 concise recommendations and 
        a fully revised Markdown document, focusing on clarity, coherence, accuracy, and alignment.
        """);
```

一旦注册，你的智能体就可以在应用程序的任何位置使用：

```csharp
app.MapGet("/agent/chat", async (
    [FromKeyedServices("Writer")] AIAgent writer,
    [FromKeyedServices("Editor")] AIAgent editor,
    HttpContext context,
    string prompt) =>
{
    Workflow workflow =
        AgentWorkflowBuilder
            .CreateGroupChatBuilderWith(agents =>
                new AgentWorkflowBuilder.RoundRobinGroupChatManager(agents)
                {
                    MaximumIterationCount = 2
                })
            .AddParticipants(writer, editor)
            .Build();

    AIAgent workflowAgent = await workflow.AsAgentAsync();

    AgentRunResponse response = await workflowAgent.RunAsync(prompt);
    return Results.Ok(response);
});
```

### 即插即用的功能

Microsoft Agent Framework 托管包含了你生产环境所需的一切：

- 配置 – 通过标准 .NET 配置管理智能体设置
- 依赖注入 – 与你现有的 DI 容器和实践集成
- 中间件支持 – 添加身份验证、速率限制或自定义逻辑

### 部署

Microsoft Agent Framework 不会重新发明部署方式。如果你知道如何发布 .NET 应用程序，那么你已经知道如何发布智能体。

无需新工具。无需特殊托管模型。只需添加智能体，即可在任何运行 .NET 的地方部署。

## 观察与改进：内置监控

生产环境中的智能体需要可观测性。Microsoft Agent Framework 提供全面的监控功能，可与你现有的可观测性堆栈集成。

### OpenTelemetry 集成

只需一行代码即可启用详细的遥测功能：

```csharp
// 让所有智能体开启 OpenTelemetry
writer.WithOpenTelemetry();
editor.WithOpenTelemetry();
```

这包括：

- 对话流程 – 可视化消息在智能体间的流动方式
- 模型使用情况 – 追踪令牌消耗、模型选择和成本
- 性能指标 – 监控响应时间和吞吐量
- 错误跟踪 – 快速识别和调试问题

### 丰富的仪表盘

当连接到你现有的可观测性平台时，例如：

- Aspire
- Azure Monitor
- Grafana
- 还有其他

你可深入了解智能体行为，从而优化性能并提前发现问题，避免影响用户体验。

### Aspire 中的 OpenTelemetry

要将智能体遥测数据发送到 Aspire 仪表板，请启用 OpenTelemetry 并允许敏感数据以获取更丰富的洞察。

在配置客户端时，将 EnableSensitiveTelemetryData 设置为 true：

```csharp
builder
    .AddAzureChatCompletionsClient("chat", settings =>
    {
        settings.EnableSensitiveTelemetryData = true;
    })
    .AddChatClient(Environment.GetEnvironmentVariable("MODEL_NAME")!);
```

然后，配置 Aspire 以识别遥测源：

```csharp
public static TBuilder ConfigureOpenTelemetry<TBuilder>(this TBuilder builder) where TBuilder : IHostApplicationBuilder
{
    builder.Logging.AddOpenTelemetry(logging =>
    {
        //...
    })
    .AddTraceSource("Experimental.Microsoft.Extensions.AI.*");

    builder.Services.AddOpenTelemetry()
        .WithMetrics(metrics =>
        {
            metrics.AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddRuntimeInstrumentation()
                .AddMeter("Experimental.Microsoft.Extensions.AI.*");
        })
        .WithTracing(tracing =>
        {
            tracing.AddSource(builder.Environment.ApplicationName)
                .AddSource("Experimental.Microsoft.Extensions.AI.*")
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation();
        });
        //...
}
```

通过此设置，Aspire 仪表板会显示详细的遥测数据，包括对话流程、模型使用情况、性能指标和错误跟踪。

![调用链路](https://devblogs.microsoft.com/dotnet/introducing-microsoft-agent-framework-preview/aspire-telemetry-dashboard.png)

![详细对话](https://devblogs.microsoft.com/dotnet/introducing-microsoft-agent-framework-preview/aspire-telemetry-dashboard-detail.png)

### 确保质量：评估与测试

对 AI 系统的信任来自于严格的评估。Microsoft Agent Framework 可以轻松集成 Microsoft.Extensions.AI.Evaluations，帮助你构建可靠、可信赖的智能体系统。

这使得：

- 自动化测试 – 将评估套件作为 CI/CD 流水线的一部分运行
- 质量指标 – 衡量相关性、连贯性和安全性
- 回归检测 – 在部署前捕获质量下降问题
- A/B 测试 – 比较不同配置

## 立即开始构建智能体

Microsoft Agent Framework 将智能体开发从一项复杂、专业的技能转变为每位 .NET 开发者都能掌握的技术。无论你是在构建聊天机器人，还是在复杂工作流中协调多个 AI 智能体，Microsoft Agent Framework 都提供了清晰的前进道路。

### 关键要点

- 设计简约：仅需几行代码即可开始。几分钟内创建你的首个智能体，而非数天。
- 随你扩展：从单个智能体开始，然后在需求增长时轻松添加工作流、工具、托管和监控功能。
- 基于成熟技术构建：Microsoft Agent Framework 汇集了 AutoGen 和 Semantic Kernel 的精华。它建立在 Microsoft.Extensions.AI 之上，这是现代 AI 开发的统一基础，为 .NET 开发人员提供强大而一致的体验。
- 即插即用：使用熟悉的 .NET 模式进行部署，具备内置的可观测性、评估和托管功能。

### 下一步是什么？

准备好开始构建了吗？

**阅读原文**下载并运行 Hello World 智能体示例。

然后，请前往 Microsoft Agent Framework 文档继续学习。

软件开发的未来将把人工智能智能体作为现代软件开发中的核心组件。微软智能体框架确保每位 .NET 开发者都能轻松实现这一愿景。

今天就到这里。期待与你再次相见。
