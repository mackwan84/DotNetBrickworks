# [Semantic Kernel 到 Agent Framework 迁移指南](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-semantic-kernel/?pivots=programming-language-csharp)

## Microsoft Agent Framework 相比 Semantic Kernel Agent Framework 的优势

- 简化 API：降低复杂性和样板代码
- 更好的性能：优化对象创建和内存使用
- 统一接口：跨不同 AI 提供商的一致模式
- 增强的开发者体验：更直观且易于发现的 API

## 主要差异

以下是 Semantic Kernel Agent Framework 和 Microsoft Agent Framework 之间的主要差异摘要，以帮助您迁移代码。

### 1. 命名空间更新

Semantic Kernel

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
```

Agent Framework

Agent Framework 命名空间位于 Microsoft.Agents.AI 下。Agent Framework 使用来自 Microsoft.Extensions.AI 的核心 AI 消息和内容类型，用于组件之间的通信。

```csharp
using Microsoft.Extensions.AI;
using Microsoft.Agents.AI;
```

### 2. 简化的智能体创建

Semantic Kernel

在 Semantic Kernel 中，每个智能体都依赖于一个 Kernel 实例，如果未提供，则会有一个空的 Kernel 。

```csharp
Kernel kernel = Kernel
    .AddOpenAIChatClient(modelId, apiKey)
    .Build();

 ChatCompletionAgent agent = new() { Instructions = ParrotInstructions, Kernel = kernel };
 ```

 Azure AI Foundry 需要先在云中创建一个智能体资源，然后才能创建使用该资源的本地智能体类。

```csharp
PersistentAgentsClient azureAgentClient = AzureAIAgent.CreateAgentsClient(azureEndpoint, new AzureCliCredential());

PersistentAgent definition = await azureAgentClient.Administration.CreateAgentAsync(
    deploymentName,
    instructions: ParrotInstructions);

AzureAIAgent agent = new(definition, azureAgentClient);
```

Agent Framework

Agent Framework 中的智能体创建通过所有主要提供商提供的扩展得到了简化。

```csharp
AIAgent openAIAgent = chatClient.CreateAIAgent(instructions: ParrotInstructions);
AIAgent azureFoundryAgent = await persistentAgentsClient.CreateAIAgentAsync(instructions: ParrotInstructions);
AIAgent openAIAssistantAgent = await assistantClient.CreateAIAgentAsync(instructions: ParrotInstructions);
```

此外，对于托管的智能体提供商，您还可以使用 GetAIAgent 从现有的托管智能体中检索智能体。

```csharp
AIAgent azureFoundryAgent = await persistentAgentsClient.GetAIAgentAsync(agentId);
```

### 3. 智能体线程创建

Semantic Kernel

调用者必须知道线程类型并手动创建它。

```csharp
// 创建用于智能体对话的线程。
AgentThread thread = new OpenAIAssistantAgentThread(this.AssistantClient);
AgentThread thread = new AzureAIAgentThread(this.Client);
AgentThread thread = new OpenAIResponseAgentThread(this.Client);
```

Agent Framework

智能体负责创建线程。

```csharp
AgentThread thread = agent.GetNewThread();
```

### 4. 托管智能体线程清理

本案例仅适用于少数仍提供托管线程的 AI 提供商。

Semantic Kernel

线程有一个 self 删除方法，即：OpenAI Assistants 提供程序

```csharp
await thread.DeleteAsync();
```

Agent Framework

Agent Framework 在 AgentThread 类型中没有线程删除 API，因为并非所有提供程序都支持托管线程或线程删除，而且随着更多提供程序转向基于响应的架构，这种情况将变得更加普遍。

如果您需要删除线程且提供程序允许这样做，调用方应跟踪已创建的线程，并在必要时通过提供程序的 SDK 稍后删除它们。

```csharp
await assistantClient.DeleteThreadAsync(thread.ConversationId);
```

### 5. 工具注册

Semantic Kernel

在 Semantic Kernel 中，要将函数公开为工具，您必须：

1. 使用 [KernelFunction] 属性装饰该函数。
2. 拥有一个 Plugin 类或使用 KernelPluginFactory 来包装该函数。
3. 拥有一个 Kernel 来添加你的插件。
4. 将 Kernel 传递给智能体。

```csharp
KernelFunction function = KernelFunctionFactory.CreateFromMethod(GetWeather);
KernelPlugin plugin = KernelPluginFactory.CreateFromFunctions("KernelPluginName", [function]);
Kernel kernel = ... // 创建并配置内核
kernel.Plugins.Add(plugin);

ChatCompletionAgent agent = new() { Kernel = kernel, ... };
```

Agent Framework

在 Agent Framework 中，您可以在单次调用中直接在智能体创建过程中注册工具。

```csharp
AIAgent agent = chatClient.CreateAIAgent(tools: [AIFunctionFactory.Create(GetWeather)]);
```

### 6. 智能体非流式调用

从 Invoke 到 Run 的方法名称、返回类型和参数 AgentRunOptions 中可以看到关键差异。

Semantic Kernel

非流式方法使用流式模式 IAsyncEnumerable\<AgentResponseItem\<ChatMessageContent\>\> 来返回多个智能体消息。

```csharp
await foreach (AgentResponseItem<ChatMessageContent> result in agent.InvokeAsync(userInput, thread, agentOptions))
{
    Console.WriteLine(result.Message);
}
```

Agent Framework

非流式返回一个包含智能体响应的 AgentRunResponse ，该响应可以包含多条消息。运行的文本结果可在 AgentRunResponse.Text 或 AgentRunResponse.ToString() 中获取。作为响应一部分创建的所有消息都在 AgentRunResponse.Messages 列表中返回。这可能包括工具调用消息、函数结果、推理更新和最终结果。

```csharp
AgentRunResponse agentResponse = await agent.RunAsync(userInput, thread);
```

### 7. 智能体流式调用

从 Invoke 到 Run 的方法名称、返回类型和参数 AgentRunOptions 的主要差异。

Semantic Kernel

```csharp
await foreach (StreamingChatMessageContent update in agent.InvokeStreamingAsync(userInput, thread))
{
    Console.Write(update);
}
```

Agent Framework

类似的流式 API 模式，主要区别在于它返回 AgentRunResponseUpdate 对象，每次更新包含更多智能体相关信息。

AIAgent 底层任何服务产生的所有更新都会被返回。智能体的文本结果可通过连接 AgentRunResponse.Text 值来获取。

```csharp
await foreach (AgentRunResponseUpdate update in agent.RunStreamingAsync(userInput, thread))
{
    Console.Write(update);
}
```

### 8. 工具函数签名

问题：SK 插件方法需要 [KernelFunction] 属性

```csharp
public class MenuPlugin
{
    [KernelFunction] // SK 必须
    public static MenuItem[] GetMenu() => ...;
}
```

解决方案：AF 可以直接使用方法而无需属性

```csharp
public class MenuTools
{
    [Description("Get menu items")] // 可选的工具描述
    public static MenuItem[] GetMenu() => ...;
}
```

### 9. 选项配置

问题：SK 中的复杂选项设置

```csharp
OpenAIPromptExecutionSettings settings = new() { MaxTokens = 1000 };
AgentInvokeOptions options = new() { KernelArguments = new(settings) };
```

解决方案：AF 中的简化选项

```csharp
ChatClientAgentRunOptions options = new(new() { MaxOutputTokens = 1000 });
```

### 10. 依赖注入

Semantic Kernel

需要在服务容器中注册 Kernel 才能创建智能体，因为每个智能体抽象都需要使用 Kernel 属性进行初始化。

Semantic Kernel 使用 Agent 类型作为智能体的基础抽象类。

```csharp
services.AddKernel().AddProvider(...);
serviceContainer.AddKeyedSingleton<SemanticKernel.Agents.Agent>(
    TutorName,
    (sp, key) =>
        new ChatCompletionAgent()
        {
            Kernel = sp.GetRequiredService<Kernel>(),
        });
```

Agent Framework

Agent Framework 提供了 AIAgent 类型作为基础抽象类。

```csharp
services.AddKeyedSingleton<AIAgent>(() => client.CreateAIAgent(...));
```

### 11. 智能体类型整合

Semantic Kernel

Semantic Kernel 为各种服务提供了特定的智能体类，例如

- ChatCompletionAgent 用于基于聊天补全的推理服务。
- OpenAIAssistantAgent 用于 OpenAI Assistants 服务。
- AzureAIAgent 用于 Azure AI Foundry Agents 服务。

Agent Framework

Agent Framework 通过单一的智能体类型 ChatClientAgent 支持上述所有服务。

ChatClientAgent 可用于使用任何底层服务构建智能体，这些服务提供实现 Microsoft.Extensions.AI.IChatClient 接口的 SDK。
