# [使用 A2A .NET SDK 构建 AI 智能体](https://devblogs.microsoft.com/foundry/building-ai-agents-a2a-dotnet-sdk/)

人工智能开发领域正在快速发展。自主智能体正变得越来越复杂、专业化，并能够跨不同领域处理复杂任务。从客服机器人到数据分析智能体，从内容创作助手到工作流编排器——AI 智能体正以前所未有的速度激增。

但是，当这些智能体需要协同工作时会发生什么呢？专门的智能体如何发现彼此、有效沟通，并协作完成需要多种能力的复杂任务？

很幸运， A2A .NET SDK 发布了——这是 Agent2Agent (A2A) 协议的一种实现，它使 .NET 开发者能够构建 A2A 服务器和客户端。借助此 SDK，你现在可以创建能够与生态系统中的其他智能体无缝通信和协作的智能体，无论这些智能体是使用 .NET 构建的，还是使用任何其他支持 A2A 协议的技术构建的。

SDK 加强了更广泛的现有 AI 生态系统，特别是对于使用 Azure AI Foundry 和 Semantic Kernel 的开发者而言。Semantic Kernel 已通过其 Python 和 .NET 实现包含了 A2A 智能体功能，而这个由社区驱动的 .NET SDK 使 Semantic Kernel 团队能够在保持框架企业级稳定性的同时，更快地采用最新的 A2A 协议功能。

## 什么是 Agent2Agent（A2A）协议？

Agent2Agent（A2A）协议是一个旨在解决 AI 生态系统中一个基本挑战的开放标准：独立的 AI 智能体如何发现、通信并相互协作？

在当今世界，AI 智能体通常是孤立的系统，由不同的供应商使用不同的框架构建，并具备不同的功能。A2A 提供了一种通用语言，允许这些智能体：

- 发现彼此的能力
- 协商交互模式（文本、表单、媒体）
- 安全地协作执行长时间运行的任务
- 在不暴露其内部状态、内存或工具的情况下运行

## A2A .NET SDK 的主要特性

### 智能体能力发现

该 SDK 包含 A2ACardResolver 类，用于检索在标准化智能体卡中描述的智能体能力。这些卡片提供了关于智能体能力（流式传输、推送通知、扩展）、智能体主机 URL、图像、支持的模态以及其他重要信息。

```csharp
A2ACardResolver cardResolver = new A2ACardResolver(new Uri("http://localhost:5100/"));
AgentCard agentCard = await cardResolver.GetAgentCardAsync();
```

### 灵活的通信模式

基于消息的通信：直接、同步的消息传递，用于即时响应

```csharp
A2AClient client = new A2AClient(new Uri(agentCard.Url));

// 发送一条消息，智能体可以立即处理并回复一条消息。
Message response = (Message)await client.SendMessageAsync(new MessageSendParams
{
    Message = new Message
    {
        Role = MessageRole.User,
        Parts = [new TextPart { Text = "What is the current weather in Settle" }]
    }
});

Console.WriteLine($"Received: {((TextPart)response.Parts[0]).Text}");
```

基于任务的通信：持久的、长时间运行的智能体任务，用于异步工作流

```csharp
A2AClient client = new A2AClient(new Uri(agentCard.Url));

// 发送一条消息，智能体需要异步处理它，智能体将回答一个任务。
AgentTask agentTask = (AgentTask)await client.SendMessageAsync(new MessageSendParams
{
    Message = new Message
    {
        Role = MessageRole.User,
        Parts = [new TextPart { Text = "Generate an image of a sunset over the mountains" }]
    }
});

Console.WriteLine("Received task details:");
Console.WriteLine($" ID: {agentTask.Id}");
Console.WriteLine($" Status: {agentTask.Status.State}");
Console.WriteLine($" Artifact: {(agentTask.Artifacts?[0].Parts?[0] as TextPart)?.Text}");

// 拉取任务状态
agentTask = await client.GetTaskAsync(agentTask.Id);
switch (agentTask.Status.State)
{
    case TaskState.Running:
        // 处理运行中的任务
        break;
    case TaskState.Completed:
        // 处理已完成的任务
        break;
    case TaskState.Failed:
        // 处理任务失败
        break;
    default:
        Console.WriteLine($"Task status: {agentTask.Status.State}");
        break;
}
```

### 实时流支持

对于需要实时更新的场景，SDK 使用服务器发送事件 (Server-Sent Events) 提供了全面的流式支持：

```csharp
A2AClient client = new A2AClient(new Uri(agentCard.Url));

await foreach (SseItem<A2AEvent> sseItem in client.SendMessageStreamAsync(new MessageSendParams { Message = userMessage }))
{
    Message agentResponse = (Message)sseItem.Data;
    // 处理流响应块
    Console.WriteLine($"Received: {((TextPart)agentResponse.Parts[0]).Text}");
}
```

### ASP.NET Core 集成

使用包含的 ASP.NET Core 扩展构建兼容 A2A 的智能体非常简单：

```csharp
WebApplicationBuilder builder = WebApplication.CreateBuilder(args);
WebApplication app = builder.Build();

TaskManager taskManager = new TaskManager();
EchoAgent agent = new EchoAgent();
agent.Attach(taskManager);

// 简单一行代码通过 A2A 协议公开你的智能体
app.MapA2A(taskManager, "/agent");
app.Run();
```

## A2A Agent 入门

为了了解 A2A 协议的实际运作方式，我们将构建一个简单的回显智能体来演示核心概念。该智能体将接收消息并通过回显消息进行响应，从而清晰地展示 A2A 通信的流程。

### 先决条件

在开始构建我们的 A2A 智能体之前，请确保你的开发机器上已安装以下组件：

- .NET 8.0 SDK 或更高版本
- 代码编辑器 – Visual Studio、Visual Studio Code 或你偏好的 .NET IDE
- 具备 C# 和 ASP.NET Core 基础知识 – 我们将创建 Web 应用程序并使用 async/await 模式

你可以通过运行以下命令来验证 .NET 安装：

```bash
dotnet --version
```

### 设置我们的 Agent 项目

我们首先创建一个新的 Web 应用程序来承载我们的 Agent。这个承载应用为我们的 Agent 提供了运行基础，使其能够与外部世界进行通信：

```bash
dotnet new webapp -n A2AAgent -f net9.0
cd A2AAgent
```

创建项目后，我们需要添加 A2A 包，这些包将为我们处理协议通信：

```bash
dotnet add package A2A --prerelease              # 核心 A2A 协议支持
dotnet add package A2A.AspNetCore --prerelease   # 与 ASP.NET Core 集成 A2A 协议
dotnet add package Microsoft.Extensions.Hosting  # 托管服务
```

### 构建我们的回声智能体

现在到了有趣的部分——创建我们的实际智能体。我们将构建一个简单的回声智能体，它会接收任何收到的消息并立即将其发送回去。虽然这看起来可能很基础，但它展示了你构建更复杂智能体所需的所有基本概念。

让我们创建一个名为 EchoAgent.cs 的新文件，用于实现我们的智能体：

```csharp
using A2A;

namespace A2AAgent;

public class EchoAgent
{
    public void Attach(ITaskManager taskManager)
    {
        taskManager.OnMessageReceived = ProcessMessageAsync;
        taskManager.OnAgentCardQuery = GetAgentCardAsync;
    }

    private async Task<Message> ProcessMessageAsync(MessageSendParams messageSendParams, CancellationToken ct)
    {
        // 获取接收到的消息
        string request = messageSendParams.Message.Parts.OfType<TextPart>().First().Text;

        // 创建并返回一条回应
        return new Message()
        {
            Role = MessageRole.Agent,
            MessageId = Guid.NewGuid().ToString(),
            ContextId = messageSendParams.Message.ContextId,
            Parts = [new TextPart() { Text = $"Echo: {request}" }]
        };
    }

    private async Task<AgentCard> GetAgentCardAsync(string agentUrl, CancellationToken cancellationToken)
    {
        return new AgentCard()
        {
            Name = "Echo Agent",
            Description = "An agent that will echo every message it receives.",
            Url = agentUrl,
            Version = "1.0.0",
            DefaultInputModes = ["text"],
            DefaultOutputModes = ["text"],
            Capabilities = new AgentCapabilities() { Streaming = true },
            Skills = [],
        };
    }
}
```

查看这段代码，你可以看到我们的智能体通过三个关键部分变得活跃。 Attach() 方法是我们将智能体连接到 A2A 框架的地方——可以把它想象为将我们的智能体插入通信网络。当消息到达时，它们会流经 ProcessMessageAsync() 方法，这是我们智能体的核心，所有的思考都在这里发生。在我们的例子中，我们只是简单地将消息回显回去，但这里是你集成 LLM 或其他 AI 处理逻辑的地方。 GetAgentCardAsync() 方法充当我们智能体的名片，告诉其他智能体我们是谁以及我们能做什么。

请注意，我们的智能体声明了流式传输能力，这意味着其他智能体可以选择与我们进行实时通信以获得即时响应。

### 上线我们的智能体

有了我们的智能体实现后，我们需要将其连接到网络，以便其他智能体能够找到并与它通信。这正是 A2A .NET SDK 的真正闪光之处——我们只需要几行代码就能使我们的智能体可被发现。

让我们更新我们的 Program.cs 来托管我们的智能体：

```csharp
using A2A;
using A2A.AspNetCore;
using A2AAgent;
using Microsoft.AspNetCore.Builder;

WebApplicationBuilder builder = WebApplication.CreateBuilder(args);
WebApplication app = builder.Build();

// 创建并附加 EchoAgent 到 TaskManager
EchoAgent agent = new EchoAgent();
TaskManager taskManager = new TaskManager();
agent.Attach(taskManager);

// 通过 A2A 协议公开智能体
app.MapA2A(taskManager, "/agent");
await app.RunAsync();
```

这个简单的设置创建了我们的 Web 应用程序，将 EchoAgent 连接到一个 TaskManager（负责处理 A2A 协议的细节），并通过“/agent”端点公开所有内容。 MapA2A 扩展方法完成了所有繁重的工作，自动处理智能体发现、消息路由和协议合规性。

一旦我们使用 dotnet run 运行我们的智能体，它就会在 <http://localhost:5000> 上可用，并准备好与任何其他兼容 A2A 的智能体或客户端进行通信。

### 测试我们的智能体

既然我们的智能体已经运行，我们需要一种方式与它通信。让我们创建一个简单的客户端应用程序来演示智能体之间如何相互通信。该客户端将发现我们的智能体，了解其功能，然后向其发送一些消息以观察回显的实际效果。

我们首先创建一个新的控制台应用程序：

```bash
dotnet new console -n A2AClient -f net9.0
cd A2AClient
dotnet add package A2A --prerelease
dotnet add package System.Net.ServerSentEvents --prerelease
```

客户端项目准备就绪后，我们可以创建一个程序来连接到我们的回显智能体，并演示常规通信和流式通信：

```csharp
using A2A;
using System.Net.ServerSentEvents;

// 1. 获取智能体卡片
A2ACardResolver cardResolver = new(new Uri("http://localhost:5000/"));
AgentCard echoAgentCard = await cardResolver.GetAgentCardAsync();

Console.WriteLine($"Connected to agent: {echoAgentCard.Name}");
Console.WriteLine($"Description: {echoAgentCard.Description}");
Console.WriteLine($"Streaming support: {echoAgentCard.Capabilities?.Streaming}");

// 2. 创建一个 A2A 客户端来使用智能体卡片中的 URL 与智能体通信
A2AClient agentClient = new(new Uri(echoAgentCard.Url));

// 3. 创建一条要发送给智能体的消息
Message userMessage = new()
{
    Role = MessageRole.User,
    MessageId = Guid.NewGuid().ToString(),
    Parts = [new TextPart { Text = "Hello from the A2A client!" }]
};

// 4. 使用非流式 API 发送消息
Console.WriteLine("\n=== Non-Streaming Communication ===");
Message agentResponse = (Message)await agentClient.SendMessageAsync(new MessageSendParams { Message = userMessage });
Console.WriteLine($"Received response: {((TextPart)agentResponse.Parts[0]).Text}");

// 5. 使用流式 API 发送消息
Console.WriteLine("\n=== Streaming Communication ===");
await foreach (SseItem<A2AEvent> sseItem in agentClient.SendMessageStreamAsync(new MessageSendParams { Message = userMessage }))
{
    Message streamingResponse = (Message)sseItem.Data;
    Console.WriteLine($"Received streaming chunk: {((TextPart)streamingResponse.Parts[0]).Text}");
}
```

此客户端演示了完整的 A2A 通信流程。首先，我们通过获取智能体卡来发现我们的智能体，智能体卡告诉我们该智能体能做什么以及如何与其通信。然后我们使用智能体的 URL 创建一个 A2A 客户端，并向其发送一条友好的消息。A2A 的妙处在于它同时支持流式和非流式通信模式，使我们能够选择最适合我们需求的方法。

当我们使用 dotnet run 运行客户端时，我们可以看到我们的回显智能体正在工作：

![A2A 客户端输出](https://devblogs.microsoft.com/foundry/wp-content/uploads/sites/89/2025/07/a2a-client-output.png)

### 进一步探索

既然我们的回显智能体已经可以运行，你可能想通过 A2A Inspector 探索它的功能。这个基于 Web 的工具可以连接到任何 A2A 智能体，让你检查其功能、发送测试消息并查看响应。该检查器对于调试也极具价值，因为它会显示正在交换的原始请求和响应消息，帮助你准确了解底层发生的情况。只需将检查器指向运行中的智能体 <http://localhost:5000> ，即可开始实验。

你可以在 A2A Inspector 仓库中找到 A2A Inspector 及其安装说明。

我们在此构建的仅仅是一个开始。我们简单的回显智能体演示了核心概念，但你可以轻松地对其进行扩展，例如将回显逻辑替换为对 AI 模型甚至其他 AI 智能体的调用，从而创建真正智能且协作的系统。

对于已经在使用 Semantic Kernel 的开发者来说，这个社区驱动的 A2A .NET SDK 提供了一条跟上不断发展的 A2A 协议的途径。Semantic Kernel 现有的 A2A 智能体实现可以利用此 SDK 的改进和功能，确保你的多智能体编排与最新的协议发展保持同步，同时保持 Semantic Kernel 提供的生产就绪稳定性。

## 你的智能体之旅从此继续

现在你已经创建了第一个 A2A 智能体，可以开始探索更多内容了。

- 探索示例——查看真实世界的示例和不同的智能体模式
- 浏览 GitHub 仓库 – 深入源代码并与其他开发者建立联系
- 掌握 A2A 协议 – 了解我们构建背后的完整规范
- 探索 AI Foundry – 构建和部署具有企业级安全性和合规性的智能体解决方案
- 加入 AI Foundry 开发者论坛 – 与使用 Azure AI Foundry 的志同道合者建立联系
- 参与 A2A 讨论并提出问题 – 与社区建立联系，获取 A2A 开发方面的帮助
- 了解 Semantic Kernel – 探索支持 A2A 的 Microsoft 企业级 AI 编排框架
- 在 Semantic Kernel 中查看 A2A – 查看使用 Semantic Kernel 的 A2A 智能体实现示例

今天就到这里。期待与你再次相见。
