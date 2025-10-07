# [Semantic Kernel：使用记忆创建智能 AI 智能体](https://jamiemaguire.net/index.php/2024/10/06/semantic-kernel-using-memories-to-create-intelligent-ai-agents/)

智能体 AI 仍在快速发展。状态管理是大多数应用中的重要方面，对于开发智能体来说也是如此。

Semantic Kernel 具有记忆的概念。

记忆让你能够存储和回忆过去交互中的相关信息，从而使其能够做出明智的决策并提供上下文感知的响应。记忆帮助智能体从经验中学习，提高其对用户偏好的理解，并随时间调整其行为。

这让你能够创建更加个性化、连贯且有效的交互，从而增强智能体的实用性，并让你更容易创造更好的用户体验。

今天，我们探讨智能体记忆。我们了解如何在 Semantic Kernel 中使用记忆。

其他主题包括：

- 什么是记忆
- 记忆存储选项
- 记忆存储机制
- 语义记忆
- Kernel Memory
- 记忆在现有 Semantic Kernel 解决方案中的位置
- 在 Semantic Kernel 记忆和 Kernel Memory 之间做选择

## 什么是记忆

记忆是一种允许 AI 智能体存储、检索和利用过去互动或经验中信息的机制。记忆可以保存各种类型的数据，如对话历史、事实或用户偏好。

这让智能体能够创建具有上下文感知和自适应性的行为。

通过利用记忆，智能体可以在未来的交互过程中回忆相关细节，提高其生成连贯响应的能力，个性化其行为，并提供更智能、且随时间推移保持一致的用户体验。

## 记忆存储选项

在语义内核中，有两种记忆存储机制：易失性记忆和非易失性记忆。以下是每种机制的描述。

### 易失性记忆

这种记忆是临时且短期的。它在会话或交互期间保存数据，但在会话结束后不会保留。易失性记忆对于处理即时上下文非常有用，例如对话流程或不需要后续回忆的任务相关数据。

在创建原型或一次性交互时使用易失性记忆。

### 非易失性记忆

非易失性记忆是持久性的，意味着它可以长期存储，并且可以在不同的会话中访问。这种记忆对于维护用户偏好、重要事实或其他需要保留并在未来交互中调用的数据非常有用。

当你需要创建能够处理多种用例的功能丰富的智能体时，请使用非易失性记忆。

上述每种记忆存储机制都使智能体能够管理短期和长期信息，以提供更丰富、更智能的用户交互。

## 记忆存储机制

现在有两种存储智能体记忆的方法。

- 语义记忆
- Kernel Memory

以下是每种方法的概述。

### 语义记忆

语义记忆指的是智能体可以参考和使用的知识和概念的结构化存储。它通常涉及以允许智能体获取见解、在事实之间建立联系并在交互中上下文地使用这些知识的方式存储数据。

在 Semantic Kernel 的内容中，语义记忆是 Semantic Kernel SDK 中的一个库。它在你的应用程序和数据库之间创建了一个包装器，从而使使用嵌入存储和检索文本变得更加容易。

### Kernel Memory

Kernel Memory 是一个可以在后台运行的服务。还记得旧的 ASP.NET State Server 吗？你可以把它想象成类似的东西。Kernel Memory 通过一组 REST API 端点暴露出来，让你能够轻松地为智能体 AI 解决方案构建 RAG 编排层。

Kernel Memory 提供了更全面的记忆管理层。它结合了多种类型的记忆（短期、长期等）来管理应用程序中的上下文和状态。它为集成不同类型的记忆提供了统一的 API 和可扩展框架。

### 语义记忆 vs. Kernel Memory

从高层次来看，语义记忆和 Kernel Memory 之间的根本区别在于，KM 是一个可以部署和使用的服务，而语义记忆并不是一个真正可部署的服务。SM 构成了你的语义内核解决方案的一部分。

下表包含语义记忆和 Kernel Memory 之间的功能比较。

功能 | Kernel Memory | 语义记忆
--- | --- | ---
数据格式 | 网页、PDF、图像、Word、PowerPoint、Excel、Markdown、文本、JSON、HTML | 仅文本
搜索 | 余弦相似度、带过滤器的混合搜索（AND/OR 条件） | 余弦相似度
语言支持 | 任何语言、命令行工具、浏览器扩展、低代码/无代码应用、聊天机器人、助手等 | C#、Python、Java
存储引擎 | Azure AI Search、Elasticsearch、MongoDB Atlas、Postgres+pgvector、Qdrant、Redis、SQL Server、内存 KNN、磁盘 KNN。 | Azure AI Search、Chroma、DuckDB、Kusto、Milvus、MongoDB、Pinecone、Postgres、Qdrant、Redis、SQLite、Weaviate
文件存储 | 磁盘、Azure Blob、AWS S3、MongoDB Atlas、内存（易失性） | –
RAG | 是的，带来源查找 | 是的，需要额外的库
摘要 | 是的 | –
OCR | 是的，通过 Azure 文档智能 | –
安全筛选器 | 是 | –
大型文档摄取 | 是的，包括使用队列的异步处理（Azure 队列、RabbitMQ、基于文件的队列或内存队列） | –
文档存储 | 是 | –
自定义存储架构 | 一些数据库 | –
具有内部嵌入的向量数据库 | 是的 | –
并发写入多个向量数据库 | 是 | –
LLMs | Azure OpenAI、OpenAI、Anthropic、Ollama、LLamaSharp、LM Studio、Semantic Kernel 连接器 | Azure OpenAI、OpenAI、Gemini、Hugging Face、ONNX、自定义。
具有专用分词功能的 LLMs | 是 | 无
云部署 | 是 | 是的，但是定制的。本地 SLM/LLM 等。
OpenAPI | 是 | –

## 我应该使用语义记忆还是 Kernel Memory？

一如既往，这取决于你的具体需求。比较表将帮助你决定使用哪种技术。

如果你希望快速入门并构建一个快速原型，我建议使用语义记忆。建立你的基础知识，创建一个简单的智能体，然后在此基础上进行迭代。

如果你正在寻求构建企业级解决方案，并实现一个能够摄取、解析和处理用于智能体式 AI 应用内容的编排层/管道，或者需要数据格式、额外的存储选项/搜索功能，那么 Kernel Memory 可能是一个更适合你的选择。

## 记忆在现有 Semantic Kernel 解决方案中的位置

语义记忆的一个简单实现可以用大约 100 行代码编写。

主要步骤包括：

- 创建一个新的记忆系统。
- 选择一个文本嵌入模型
- 选择一个记忆存储机制（比如，内存/易失性存储适合用于测试）

你可以在这里看到：

```csharp
var memoryWithCustomDb = new MemoryBuilder()
      .WithOpenAITextEmbeddingGeneration("text-embedding-ada-002", apiKey)
      .WithMemoryStore(new VolatileMemoryStore())
      .Build();
```

接下来，你必须向记忆存储中添加内容。你可以在这里看到这一过程。

```csharp
private async Task StoreMemoryAsync(ISemanticTextMemory memory)
 {
     Dictionary d = new Dictionary<string, string>
            {
                ["somePage.html"]
                    = “someDesc””,               
            };

     var counter = 0;

     foreach (var c in d)
     {
         await memory.SaveReferenceAsync(
             collection: “YourMemoryCollectionName”,
             externalSourceName: "SourceName",
             externalId: entry.Key,
             description: entry.Value,
             text: entry.Value);

         Console.Write($" #{++i} saved.");
     }
}
```

注意：有两种方法可以将内容添加到记忆中。

- SaveReferenceAsync
- SaveInformationAsync

内容添加到记忆后，现在可以进行搜索。以下代码展示了如何实现这一功能：

```csharp
private async Task SearchMemoryAsync(ISemanticTextMemory memory, string query)
 {

     var memoryResults = memory.SearchAsync(MemoryCollectionName, query, limit: 2, minRelevanceScore: 0.5);

     int i = 0;

     await foreach (MemoryQueryResult memoryResult in memoryResults)
     {
         Console.WriteLine($"Result {++i}:");
         Console.WriteLine("  URL:     : " + memoryResult.Metadata.Id);
         Console.WriteLine("  Title    : " + memoryResult.Metadata.Description);
         Console.WriteLine("  Relevance: " + memoryResult.Relevance);
         Console.WriteLine();
     }
}
```

好了，就是这么简单。

## 总结

今天我们探讨了 Semantic Kernel 中智能体记忆的概念。

我们检查了记忆的内容以及你可能选择存储的数据类型。

我们还介绍了目前在 Semantic Kernel 中创建、搜索和集成智能体记忆的可用选项。

未来，我们将看到更多关于语义记忆和 Kernel Memory 的实际应用示例。

今天就到这里。期待与你再次相见。
