# [Semantic Kernel 集成 Kernel Memory](https://www.developerscantina.com/p/semantic-kernel-memory/)

昨天，我们了解到微软的开源服务内核内存如何极大地简化检索增强生成(RAG)体验的实现，这种体验使得能够将 LLMs 的力量与私有数据（如组织文档）结合使用。我们在上一篇文章中看到的功能非常强大，但也有一定的局限性。事实上，我们只能提出与文档相关的直接问题，比如"什么是 Contoso Electronics？"，但如果你需要使用这些文档执行更复杂的任务，比如实现连续聊天智能体或使用存储在文档中的信息执行其他活动，该怎么办呢？这正是语义内核的用武之地，它通过使用插件、函数调用和规划器，允许构建复杂的 AI 工作流。

今天，我们将了解如何通过使用专用插件来结合 Semantic Kernel 和 Kernel Memory。我们将构建两种不同的场景：

- 一个控制台应用程序，它将结合我们已经看到的两种场景：使用提示函数将文本转换为商务邮件和使用 Kernel Memory 将虚构公司的员工手册存储为嵌入向量。在这种情况下，我们不仅仅是想回答关于手册的问题，而是想将答案转换为商务邮件。
- 一个可以连续聊天的智能体，我们将在此体验中询问多个关于手册的问题，同时保留对话的上下文。

要运行这些示例，我将假设你已经使用 Blazor 应用程序上传了员工手册并将其转换为嵌入向量，这些向量已存储在 Azure AI Search 实例中。如果你尚未完成此操作，请按照昨天的说明进行操作。

无论哪种场景，我们设置插件的方式都是相同的。就让我们开始吧！

## 设置插件

让我们首先设置项目。第一步是初始化 Semantic Kernel，方式与我们在所有关于这个库的其他文章中所做的相同：

```csharp
string apiKey = "AzureOpenAI:ApiKey";
string deploymentChatName = "AzureOpenAI:DeploymentChatName";
string endpoint = "AzureOpenAI:Endpoint";

var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(deploymentChatName, endpoint, apiKey)
    .Build();
```

现在我们需要导入 Kernel Memory 插件，因此第一步是安装名为 Microsoft.KernelMemory.SemanticKernelPlugin 的 NuGet 包。现在我们可以使用这个库提供的 MemoryPlugin 类，但是这需要初始化我们要使用的 KernelMemory 实例。在上一篇文章中，我们使用了无服务器模式的 Kernel Memory（这意味着服务由应用程序本身托管），因此在这篇文章中我们将继续使用相同的方法。但是，请记住，如果你需要更具可扩展性的解决方案，你可以自由使用专用服务。因此，我们使用 KernelMemoryBuilder 类以与我们在上一篇文章中构建的 Blazor 应用程序相同的方式初始化 KernelMemory 对象：

```csharp
string apiKey = "AzureOpenAI:ApiKey";
string deploymentChatName = "AzureOpenAI:DeploymentChatName";
string deploymentEmbeddingName = "AzureOpenAI:DeploymentEmbeddingName";
string endpoint = "AzureOpenAI:Endpoint";

string searchApiKey = "AzureSearch:ApiKey";
string searchEndpoint = "AzureSearch:Endpoint";

var embeddingConfig = new AzureOpenAIConfig
{
    APIKey = apiKey,
    Deployment = deploymentEmbeddingName,
    Endpoint = endpoint,
    APIType = AzureOpenAIConfig.APITypes.EmbeddingGeneration,
    Auth = AzureOpenAIConfig.AuthTypes.APIKey
};

var chatConfig = new AzureOpenAIConfig
{
    APIKey = apiKey,
    Deployment = deploymentChatName,
    Endpoint = endpoint,
    APIType = AzureOpenAIConfig.APITypes.ChatCompletion,
    Auth = AzureOpenAIConfig.AuthTypes.APIKey
};

var kernelMemory = new KernelMemoryBuilder()
    .WithAzureOpenAITextGeneration(chatConfig)
    .WithAzureOpenAITextEmbeddingGeneration(embeddingConfig)
    .WithAzureAISearchMemoryDb(searchEndpoint, searchApiKey)
    .Build<MemoryServerless>();
```

Kernel Memory 需要一个文本生成模型（用于根据问题生成答案）和一个嵌入模型（用于将文档转换为向量，以便存储在向量数据库中）。因此，我们使用 AzureOpenAIConfig 类为这两个模型提供配置（如果你使用的是 OpenAI API，可以切换到 OpenAIConfig 类）。在本示例中，我们还将使用与上一篇文章相同的 Azure AI Search 实例，因此我们也通过提供相同的端点和 API 密钥，使用 WithAzureAISearchMemoryDb() 方法对其进行初始化。最后，我们通过调用 Build\<MemoryServerless\>() 方法生成一个 KernelMemory 对象，该方法将以无服务器模式初始化服务。

现在我们可以创建一个 MemoryPlugin 对象并将其加载到 Semantic Kernel 中：

```csharp
var plugin = new MemoryPlugin(kernelMemory, waitForIngestionToComplete: true);
kernel.ImportPluginFromObject(plugin, "memory");
```

这种方法与我们之前看到的方法类似：我们创建 MemoryPlugin 类的新实例，然后通过提供插件名称，使用 ImportPluginFromObject() 方法将其导入 Semantic Kernel。

由于我们想要根据从员工手册中获得的答案生成商务邮件，别忘了导入我们的 MailPlugin ，其中包含 WriteBusinessMail 提示函数（有关插件的详细信息请参考这篇文章）：

```csharp
var pluginsDirectory = Path.Combine(Directory.GetCurrentDirectory(), "Plugins", "MailPlugin");
kernel.ImportPluginFromPromptDirectory(pluginsDirectory, "MailPlugin");
```

现在让我们使用 Semantic Kernel 的函数调用功能来定义我们的请求，并让框架自动选择我们加载的两个插件： MemoryPlugin ，用于查询 Azure AI Search，以及 MailPlugin ，用于将答案转换为商务邮件。然而，要做到这一点，我们需要稍微改变定义提示的方式。让我们看一下完整的代码：

```csharp
OpenAIPromptExecutionSettings settings = new()
{
    ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions,
};


var prompt = @"
            Question to Kernel Memory: {{$input}}

            Kernel Memory Answer: {{memory.ask}}

            If the answer is empty say 'I don't know', otherwise reply with a business mail to share the answer.
            ";


KernelArguments arguments = new KernelArguments(settings)
{
    { "input", "What is Contoso Electronics?" },
};

var response = await kernel.InvokePromptAsync(prompt, arguments);

Console.WriteLine(response.GetValue<string>());
Console.ReadLine();
```

首先，我们将 OpenAIPromptExecutionSettings 的 ToolCallBehavior 属性设置为 AutoInvokeKernelFunctions ，这意味着语义内核不仅会识别用于满足请求的插件，还会自动调用插件定义的函数。然后，我们定义提示，这与我们目前所见略有不同。第一部分应该很熟悉：我们正在使用提示函数的模板功能定义一个名为 input 的占位符，稍后我们将用用户的真实问题替换它。现在让我们看看一些新内容：在模板中，我们可以要求语义内核显式调用一个函数。在这种情况下，我们要求调用 MemoryPlugin 提供的 ask 函数，该函数将 LLM 功能与向量数据库（在本例中为 Azure AI Search）上的语义搜索相结合。隐式地，我们提供的相同 input 占位符将用作该函数的参数。

总之，提示词将以这种方式执行：

- 用户提出一个问题，例如"什么是 Contoso Electronics？"。这个问题被嵌入到提示中。
- 我们使用这个问题作为 input 来调用 ask 函数。生成的答案也将被嵌入到提示中。
- 最后，我们要求将答案转为商务邮件。如果 Kernel Memory 无法找到答案，我们会指示 LLM 直接说"我不知道"。

最后，我们使用 InvokePromptAsync() 方法通过提供提示和参数（在这种情况下， input 占位符的值包含我们想要提问的问题）来调用提示。结果将会是这样的：

```markdown
Subject: Introduction to Contoso Electronics - Your Partner in Advanced Aerospace Solutions

Dear [Recipient's Name],

I hope this message finds you well.

I am writing to introduce you to Contoso Electronics, a distinguished provider of advanced electronic components within the aerospace industry. 
Our expertise lies in delivering both commercial and military aircraft systems that embody the pinnacle of innovation, reliability, and efficiency.

At Contoso Electronics, we take immense pride in our steadfast commitment to quality. Our primary focus is to ensure the delivery of superior aircraft components, 
with an unwavering attention to safety and the pursuit of excellence. We are honored to have established a credible reputation 
in the aerospace sector and are relentlessly dedicated to the ongoing enhancement of our product offerings and customer service.

Our success is driven by a team of skilled engineers and technicians, all of whom are devoted to furnishing our clientele with exceptional products and services. 
The values that define us-hard work, innovation, collaboration, quality, integrity, teamwork, respect, excellence, accountability, and community engagement-are at the core of everything we do.

Should you have any questions or wish to explore how Contoso Electronics can support your business needs, please feel free to reach out. 
We would be delighted to engage in further discussion regarding our solutions and how we can contribute to the success of your projects.

Thank you for considering an affiliation with Contoso Electronics. We look forward to the possibility of a fruitful collaboration.

Warm regards,

AI Assistant
```

成功了！如你所见，只需一次执行，我们就能够从私有数据（我们的员工手册）生成响应，并将其转换为商务邮件。让我们通过实现聊天智能体来看另一个例子！

## 实现可以连续聊天的智能体

接下来，我们将使用相同的场景：我们想询问有关员工手册的问题。然而，这次我们要实现一个聊天智能体，这样就可以提出多个问题并保留对话的上下文，这是仅用 Kernel Memory 无法实现的。初始化代码与之前看到的相同，但我们将改变使用 Semantic Kernel 的方式：

```csharp
var chatHistory = new ChatHistory();
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();

while (true)
{
   var message = Console.ReadLine();

   var prompt = $@"
           Question to Kernel Memory: {message}

           Kernel Memory Answer: {{memory.ask}}

           If the answer is empty say 'I don't know', otherwise reply with the answer.
           ";


   chatHistory.AddMessage(AuthorRole.User, prompt);
   var result = await chatCompletionService.GetChatMessageContentAsync(chatHistory, settings, kernel);
   Console.WriteLine(result.Content);
   chatHistory.AddMessage(AuthorRole.Assistant, result.Content);
}
```

要实现使用 Semantic Kernel 的聊天智能体，我们可以使用 ChatCompletionService 对象，这是我们在介绍函数调用时已经了解到的。当我们在 KernelBuilder 中注册聊天完成模型时，Semantic Kernel 会自动注册该服务，因此我们可以通过调用 GetRequiredService\<IChatCompletionService\>() 使用依赖注入来检索它。聊天实现发生在一个无限循环内，这样我们可以持续提问，直到关闭应用程序。我们使用 Console.ReadLine() 读取用户的消息，然后将其注入到本文第一个示例中看到的相同提示中。然而，在这种情况下，我们是使用 C# 字符串操作功能来注入消息，而不是使用 Semantic Kernel 模板引擎。这是因为，当我们使用 ChatCompletionService 时，我们不能使用提示模板。不过，提示的工作方式是相同的：我们使用插件的 ask 函数来提供问题的答案，然后将其分享给用户。然而，这次我们只是直接返回答案，而不将其转换为商务邮件。

在通过调用 GetChatMessageContentAsync() 执行提示之前，我们将其添加到 ChatHistory 集合中，并将角色设置为 User ，因为这是用户编写的消息。一旦获得结果，我们将其展示给用户，并将其添加到 ChatHistory 集合中，这次将角色设置为 Assistant ，因为这是由 LLM 生成的。得益于这种方法，每次我们调用 GetChatMessageContentAsync() 方法时，都会向 LLM 提供完整的聊天历史记录，这将使我们能够保留对话的上下文。让我们来看一个例子。运行应用程序，让我们询问"什么是 Contoso Electronics?"。答案将类似于这样：

```markdown
Contoso Electronics is a leader in the aerospace industry, known for providing advanced electronic components for both commercial and military aircraft. 
The company focuses on creating innovative and reliable systems that offer efficiency and quality. 
With a commitment to safety and excellence, Contoso Electronics has built a reputable stature in the aerospace market.
Its mission involves delivering top-quality aircraft components to its customers.
```

现在让我们问另一个后续问题，比如"它的价值观是什么？"。这是我们应该得到的回答：

```markdown
The values of Contoso Electronics are:

1. **Quality**: Ensuring the delivery of the highest quality products and services to customers.
2. **Integrity**: Upholding honesty, respect, and trustworthiness in all interactions.
3. **Innovation**: Fostering creativity and supporting new ideas and approaches to business.
4. **Teamwork**: Emphasizing the belief that collaboration leads to greater success.
5. **Respect**: Maintaining respect and dignity towards all employees, customers, and partners.
6. **Excellence**: Aiming to exceed expectations and provide exceptional service.
7. **Accountability**: Taking responsibility for one's actions and holding individuals accountable for their performance.
8. **Community**: Committing to creating a positive impact in the communities where they operate.
```

正如你所看到的，我们不必再次指定完整的上下文。我们没有问"Contoso Electronics 的价值观是什么？"，而是只问了"它的价值观是什么？"，但 LLM 能够理解对话的上下文并提供正确的答案，这要归功于 ChatHistory 集合的使用。

## 总结

Kernel Memory 是一个强大的记忆服务，用于在你的应用程序中实现 RAG 体验，并使 LLMs 能够处理私有数据。虽然 Kernel Memory 单独使用在简单的问答场景中表现良好，但当它与 Semantic Kernel 结合使用时才能真正发挥其优势，因为你可以实现更复杂的 AI 工作流。在本文中，我们看到了两个示例：

- 将向量数据库中存储的私有知识与其他插件和功能相结合的能力。
- 聊天智能体的实现，为用户提供了更自然、强大的问答体验。

同样，你可以点击**阅读原文**并在其中找到本文中使用的代码。

今天就到这里。期待与你再次相见。
