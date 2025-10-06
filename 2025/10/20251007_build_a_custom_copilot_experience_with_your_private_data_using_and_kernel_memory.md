# [使用 Kernel Memory 构建基于私有数据的 Copilot](https://www.developerscantina.com/p/kernel-memory/)

在构建由 AI 驱动的应用程序时，你需要支持的最常见场景之一是使 LLM 能够处理私有数据。模型是使用大量公共数据训练的，这意味着它们可以很好地处理诸如告诉你著名书籍的情节或为你提供历史事实概述等任务。但是，如果你想使用 LLM 来帮助你规划工作日呢？或者回答有关公司内部政策的问题？这些都是 LLM 无法处理的任务，因为这些是私人信息，因此它们不属于用于训练模型的数据集

有多种技术可以实现这一场景。其中最常用的一种技术被称为检索增强生成（或 RAG）。这种技术基于将 LLM 与搜索引擎相结合的理念。LLM 用于生成答案，而搜索引擎用于从私有数据集中检索最相关的信息。从高层次来看，过程如下：

- 用户通过编写提示向 LLM 提问。
- 用户提示用于生成用户意图，这是一句描述用户试图实现什么的句子。例如，如果用户问"公司关于居家工作的政策是什么？"，用户意图可能是"公司远程工作政策"。
- 用户意图用于查询持有私有数据的数据源。查询结果是一组与解决用户意图相关的文档。
- 用户提示与从数据源检索到的文档相结合，然后提交给 LLM。
- LLM 使用提示中提供的信息生成答案，并将响应发送回用户。

通过阅读这些步骤，你可能已经意识到 RAG 的核心组件是搜索体验。我们都知道 LLM 在自然语言生成内容方面的强大能力，因此我们相信它可以生成一个好的答案。然而，答案的可靠性完全取决于我们与用户提示一起发送的文档。如果这些文档与解决用户意图相关，LLM 将生成一个好的答案。否则，响应将是不可靠的。

## 介绍向量数据库

因此，在过去几个月里，另一项技术作为 RAG 的补充开始兴起：向量数据库或向量索引。深入探讨向量数据库背后的理论超出了本文的范围，但主要思想是它们可以用来存储文档，并根据文档与查询的相似性来检索它们。事实上，标准搜索的一个限制是它通常基于关键词：我们搜索那些标题或正文中包含特定词语的文档。而向量数据库则支持语义搜索的概念：它们能够检索与查询在语义上相似的文档，即使这些文档不包含相同的词语。让我们重用之前的例子，即"公司关于在家工作的政策是什么？"这个问题。在这种情况下，搜索体验应该能够不仅返回提及"在家工作"的文档，还应该涵盖"远程工作"、"智能工作"、"灵活性"或"工作与生活平衡"等主题的文档。 向量数据库能够支持这种场景，它通过将文档存储为向量，并根据它们与查询向量的相似性来检索这些文档，这是一种数学函数。

事实上，你可以将向量数据库想象成一个多维空间，其中每个文档都是一个点。两点之间的距离是它们相似性的度量。它们越接近，就越相似。它们越远，就越不同。下图展示了这个概念：

![向量数据库表示](https://www.developerscantina.com/p/kernel-memory/vector-database.png)

在图中，你可以看到两个突出显示的词组彼此接近，因为它们与同一主题相关：食物。然而，有些词比其他词更接近，因为它们的相似性更强。单词 muffin（松饼）比 steak（牛排）或 lobster（龙虾）更接近 donuts（甜甜圈）或 coffee（咖啡）。通过进行语义搜索，我们能够更轻松、更快速地检索到与我们正在寻找的主题相似的所有文档。

但这些向量是从哪里来的呢？生成它们的最常见方法是使用一种特殊的 AI 模型，称为嵌入模型。嵌入模型能够将给定的输入（如文本）转换为向量，以便你可以将其存储在向量数据库中。例如，OpenAI 提供了一个名为 text-embedding-ada-002 的模型，可用于此任务。

但这还不是全部。对于简单场景，你可以直接将整个文本作为向量存储到向量数据库中。但是，对于需要在可能很长的文档中进行搜索的实际场景，这种方法并不合适。事实上，LLMs 的一个限制是它们无法处理无限的提示。每个 LLM 都有一个最大窗口大小，必须考虑输入提示和响应的长度。这个长度以令牌（tokens）来衡量：你可以（非常粗略地）将它们视为单词或文本块。LLM 的最大窗口以令牌来衡量。例如，OpenAI 的基础 GPT-4 模型支持最大 8000 个令牌的窗口。

这意味着在提示中发送文档的全部内容可能会轻易消耗所有可用的令牌，使 LLM 无法处理响应。让我们举一个具体的例子。假设你已将一份包含公司政策的 50 页文档作为单个文本存储到向量数据库中。这意味着，如果你问"公司关于在家工作的政策是什么？"这个问题，搜索必须检索整个文档并将其发送给 LLM，即使它可能包含许多与在家工作政策无关的信息。然后 LLM 会尝试生成响应，但可能会失败，因为文档太长，会消耗所有可用的令牌。

解决这个问题的方法是将文档分割成多个小块，每个小块都足够小以便 LLM 处理。这被称为分块。当你使用这种方法时，在向量数据库中你不会将整个文档作为一个整体存储，而是将其分割成小块，并将每个小块作为单独的文档存储。当你执行搜索时，你会检索最相关的小块，然后将它们发送给 LLM。市场上有许多工具和服务（如 Azure AI Document Intelligence）可以用来执行此任务。

如果你一直跟着我的思路，你可能已经意识到，构建一个支持我们刚刚描述的 RAG 实现的应用程序需要一定程度的复杂性：

- 它需要将用户提示转化为用户意图。
- 它需要使用嵌入模型将我们的私有数据转换为向量，以便将它们存储到向量数据库中。我们还需要应用分块技术，以提高 LLM 处理长文档的能力。
- 它必须支持根据用户意图对向量数据库执行语义搜索。
- 它必须将用户提示与从向量数据库检索到的文档结合起来，并将它们提交给 LLM 以生成响应。

今天，我们将学习微软创建的一项名为 Kernel Memory 的服务，它可以大大简化这一实现，并且可以通过插件与 Semantic Kernel 集成。

## 介绍 Kernel Memory

Kernel Memory 构建在与 Semantic Kernel 相同的原则之上：它抽象了 RAG 背后的许多概念，并且得益于多种扩展方法，你可以轻松将其连接到常见的 AI 服务（如 OpenAI 和 Azure OpenAI）和向量数据库（如 Azure AI Search 或 Qdrant）。正如我们将在本文中看到的，从一个服务切换到另一个服务并不意味着需要更改应用程序的代码。

此外，它提供内置支持，可将多种类型的内容转换为嵌入向量，如 PDF、Word 文档、PowerPoint 演示文稿、网站等。它还允许你跳过"用户提示 -> 用户意图 -> 向量数据库搜索 -> 提示增强 -> LLM 响应"的流程，通过提供一个执行所有这些步骤的单一方法。最后，它提供内置的分块支持，可以在将长文档存储到向量数据库之前自动将其分割成较小的块。

Kernel Memory 可以通过两种方式运行：

- Serverless ：整个服务由运行应用程序直接托管，该应用程序负责生成嵌入、存储嵌入以及执行搜索。这是最简单的场景，但它不适合大型数据集或复杂应用程序，因为它难以扩展。
- As a service : 该服务作为独立实例托管，主应用程序可以通过 REST API（由专用 C# 客户端封装）使用。对于复杂应用程序，这是推荐的方法，因为它允许你独立于主应用程序扩展服务。例如，你可以在 Azure 上使用应用服务托管它，并根据工作负载轻松扩展到多个实例。

无论你选择哪种模型，执行主要任务（如生成向量）的 API 都是相同的。在本文中，我们将使用一个基于 Blazor 的示例 Web 应用程序，该应用程序以无服务器模式使用 Kernel Memory 来实现 RAG。该应用程序将提供两个主要功能：

- 内容摄取：用户可以上传文档，或者简单地编写文本，应用程序将为其生成向量并将其存储在向量数据库中。
- 问答：用户可以就上传的内容提问，应用程序将使用 Kernel Memory 检索最相关的文档，并使用 LLM 生成响应。

## 创建 Blazor 应用程序

对于本示例，我将使用 Blazor，这是一个用于构建客户端运行的网络应用程序的框架，使用 C# 和 .NET 而非 JavaScript。我将使用服务器端模型，这意味着应用程序将在服务器上运行，代码将通过 SignalR 在客户端上渲染。这是最简单的模型，因为它不需要任何额外的配置。在 Visual Studio 2022 中，选择"创建新项目"并选择 Blazor Web App 作为模板。保留所有默认设置，这将生成一个基于 .NET 8.0 的 Blazor 服务器端应用程序：

![创建新的 Blazor Web 应用程序的向导](https://www.developerscantina.com/p/kernel-memory/new-blazor-app.png)

首先，你必须安装名为 Microsoft.KernelMemory.Core 的 Kernel Memory NuGet 包。

现在我们将构建一个服务，该服务将为我们封装所有与 Kernel Memory 的操作。创建一个名为 Services 的新文件夹，并添加一个名为 MemoryService 的新类。在实现任何功能之前，我们需要初始化 Kernel Memory。我们可以在服务的构造函数中完成这一操作。让我们先看完整代码，然后再进行讨论：

```csharp
public MemoryService()
{
    string apiKey = "AzureOpenAI:ApiKey";
    string deploymentChatName = "AzureOpenAI:DeploymentChatName";
    string deploymentEmbeddingName = "AzureOpenAI:DeploymentEmbeddingName";
    string endpoint = configuration"AzureOpenAI:Endpoint";

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

    kernelMemory = new KernelMemoryBuilder()
        .WithAzureOpenAITextGeneration(chatConfig)
        .WithAzureOpenAITextEmbeddingGeneration(embeddingConfig)
        .WithSimpleVectorDb()
        .Build<MemoryServerless>();
}
```

与我们初始化 Semantic Kernel 的方式相比，我们使用单个模型执行所有操作，而在使用内核内存时，我们需要使用两个不同的模型：

- 一个用于文本生成，LLM 使用它来生成响应。
- 一个用于嵌入生成，它用于将文档转换为向量。

与 Semantic Kernel 类似，我们有一个 KernelMemoryBuilder 对象，可以用来创建一个内核，该内核支持多种扩展方法来注册不同的 AI 服务。在前面的示例中，我们将使用由 Azure OpenAI 托管的模型，因此我们使用 WithAzureOpenAITextGeneration() 和 WithAzureOpenAITextEmbeddingGeneration() 等方法，但你也可以使用 WithOpenAITextGeneration() 和 WithOpenAITextEmbeddingGeneration() 变体来直接使用 OpenAI 服务。

由于我使用的是 Azure OpenAI，我必须向这两种方法提供一个 AzureOpenAIConfig 对象，该对象用于配置与 Azure OpenAI 的连接。在我的情况下，我使用的是单个 Azure OpenAI 实例，在其中部署了我需要的两种模型。因此，像 APIKey 和 Endpoint 这样的属性是相同的。唯一的区别是 Deployment 属性的值，这个值确实不同，因为文本生成需要传统的 GPT 模型（如 gpt-4 ），而嵌入生成则需要一个嵌入模型（如 text-embedding-ada-002 ）。

![一个已部署多个模型的 Azure OpenAI 实例](https://www.developerscantina.com/p/kernel-memory/azure-openai.png)

一旦我们定义了模型，就必须指定要使用哪个向量数据库。Kernel Memory 支持多种数据库，但在我们的演示中，我们将保持简单。我们将使用 SimpleVectorDb ，这是一个可以完全在 RAM 或磁盘上运行的解决方案。它不适合生产场景，但对于演示来说非常完美，因为它不需要我们设置任何外部服务。目前，我们只是将向量数据库保存在 RAM 中，因此我们不需要指定任何参数。最后，我们调用 Build() 来创建内核，传递类型 MemoryServerless ，因为我们将在应用程序中直接托管该服务。

现在我们准备实现应用程序的两个主要功能。

## 实现内容摄取

我们要实现的第一个场景是内容摄取：我们需要将文档转换为嵌入向量，并将它们存储到向量数据库中。Kernel Memory 通过提供多种基于我们想要摄取的内容类型的方法，使这一过程变得非常简单。让我们为最常见的场景添加一种方法，即文件摄取：

```csharp
public async Task<bool> StoreFile(string path, string filename)
{
    try
    {
        await kernelMemory.ImportDocumentAsync(path, filename);
        return true;
    }
    catch
    {
        return false;
    }
}
```

我们调用 ImportDocumentAsync() 方法，将要导入的文件路径及其文件名作为参数传递。它可以是项目仓库中指定的支持的文件类型之一。 kernelMemory 对象提供了许多其他方法来导入其他类型的内容。例如，我们可以使用 ImportTextAsync() 方法导入文本，或使用 ImportWebPageAsync() 方法导入网站。

## 实现问答功能

Kernel Memory 大大简化了问答体验，因为我们不必手动在向量数据库中搜索相关内容，然后将其包含在发送给 LLM 的提示中。实际上，整个操作可以通过一个名为 AskAsync() 的方法来执行。让我们通过向 MemoryService 类添加一个新方法来看看如何使用它：

```csharp
public async Task<string> AskQuestion(string question)
{
    var answer = await kernelMemory.AskAsync(question);
            
    return answer.Result;
}
```

我们只需调用 AskAsync() 方法，将我们想提问的问题作为参数传递。该方法将执行以下步骤：

- 它将使用文本生成模型将问题转换为用户意图。
- 它将使用用户意图在向量数据库上执行语义搜索。
- 它将组合从向量数据库检索到的文档，并将它们添加到用户提示中。
- 它将把提示提交给 LLM，然后 LLM 会返回响应。

只需一行代码即可完成众多步骤！

在测试代码之前，让我们添加一个接口来描述我们刚刚创建的服务，这样我们就可以在 Blazor 应用程序中通过依赖注入轻松使用它：

```csharp
public interface IMemoryService
{
    Task<bool> StoreFile(string content, string filename);

    Task<KernelResponse> AskQuestion(string question);
}
```

现在我们可以转到项目的 Program.cs 类中，在调用 builder.Build() 方法之前，让我们注册我们的服务：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();

builder.Services.AddSingleton<IMemoryService, MemoryService>();

var app = builder.Build();
```

现在我们可以在页面中使用该服务了。

## 创建 Blazor UI

现在让我们添加所需的 Blazor 组件来摄取内容和提问。我们从内容摄取开始，在 Pages 文件夹下添加一个名为 AddContent.razor 的新组件。第一步，我们需要使用 @page 指令创建路由，将组件映射到页面；然后，必须使用 @inject 指令获取刚刚创建的服务实例：

```razor
@page "/addcontent"
@inject IMemoryService MemoryService
```

现在让我们添加支持文件摄取的 UI：

```razor
<h3>Store file</h3>
<div>
    <InputFile OnChange="@LoadFiles" />
</div>
<div>
    @Output
</div>
```

我们添加了一个 InputFile 组件，并订阅了 OnChange 事件，该事件在用户从磁盘选择文件时触发。我们还添加了一个 Output div，它将用于显示操作结果。现在让我们添加处理 OnChange 事件的代码：

```razor
@code {
    private string Output;

    private async Task LoadFiles(InputFileChangeEventArgs e)
    {
        Output = "Saving file in progress...";
        var stream = e.File.OpenReadStream();
        var directory = Path.GetDirectoryName(Environment.ProcessPath);
        var documentsPath = Path.Combine(directory, "Documents");
        if (!Directory.Exists(documentsPath))
        {
            Directory.CreateDirectory(documentsPath);
        }
        var path = Path.Combine(directory, "Documents", e.File.Name);
            
        using (var fileStream = new FileStream(path, FileMode.Create, FileAccess.Write))
        {
            await stream.CopyToAsync(fileStream);
        }

        await MemoryService.StoreFile(path, e.File.Name);
        Output = "File stored succesfully";
    }
}
```

LoadFiles() 处理程序接收 OnChange 事件的事件参数作为输入，其中包括一个 File 属性，用于访问所选文件的内容。我们使用 Name 属性来检索文件的完整名称，并用它来定义我们想要存储文件的完整路径。在我们的例子中，这是一个名为 Documents 的文件夹，它是应用程序运行位置的子文件夹。然后，我们使用 OpenReadStream() 方法获取文件内容的流，并使用 FileStream 类和 CopyToAsync() 方法将其复制到一个新文件中。最后，我们调用 MemoryService 类的 StoreFile() 方法，将文件的路径和名称作为参数传递，这将用于生成向量并将其存储在向量数据库中。

现在让我们添加 UI 来支持问答体验。在 Pages 文件夹下创建一个名为 AskQuestion.razor 的新组件。同样，我们需要使用 @page 指令将其添加到页面，并使用 @inject 指令注入 MemoryService 对象。然后，添加以下代码：

```razor
@page "/askquestion"
@inject IMemoryService MemoryService
```

页面的代码非常简单：

```razor
<p>
    Type your question:
</p>
<p>
    <input @bind="question" style="width: 100%;" />
</p>
<p>
    <button class="btn btn-primary" @onclick="Ask">Ask</button>
</p>
<p>
    <strong>The answer is:</strong> @answer
</p>

@code {
    private string question;
    private string answer;

    private async Task Ask()
    {
        answer = await MemoryService.AskQuestion(question);

    }
}
```

我们有一个简单的输入字段，它绑定到问题属性。当用户点击"询问"按钮时，会调用 Ask() 方法，该方法会调用 MemoryService 类的 AskQuestion() 方法。该方法将在向量数据库中启动语义搜索，获取最相关的文档，将它们发送给 LLM 并生成响应，然后我们将向用户显示该响应。

## 测试应用程序

让我们测试应用程序！在 Visual Studio 中按 F5，然后当 Blazor 网站在浏览器中加载后，前往 /addcontent 页面。对于这次测试，我们将使用以下 PDF 文件，这是一个虚构公司的员工手册。点击"选择文件"并选择该 PDF。几秒钟后，处理将完成，你将看到成功消息。

现在前往 /askquestion 页面，让我们尝试询问一些关于文档内容的问题。例如，"什么是 Contoso Electronics？"。

![什么是 Contoso Electronics？](https://www.developerscantina.com/p/kernel-memory/question1.png)

答案是正确的！让我们再试一个："Contoso Electronics 的价值观是什么？"。

![Contoso Electronics 的价值观是什么？](https://www.developerscantina.com/p/kernel-memory/question2.png)

再次，答案是正确的！然而，这些问题并没有展示语义搜索的全部潜力，因为可能生成回答所需的信息也可以通过更传统的关键词搜索来检索。让我们尝试一个更复杂的问题："Contoso Electronics 与飞行物之间有什么联系吗？"。

![Contoso Electronics 与飞行物之间有什么联系吗？](https://www.developerscantina.com/p/kernel-memory/question3.png)

正如你所见，尽管我们上传的 PDF 文件中并未包含"会飞的东西"这些具体词语，但语义搜索仍能检索到文档中讨论 Contoso Electronics 公司使命的段落，即制造飞机零部件。这是语义搜索能力的一个绝佳例子！

但我们如何确保信息确实来自 PDF 文档？我们可以利用 Kernel Memory 的另一个强大功能，即在响应中包含引用的能力。我们在 KernelMemory 类中创建的 AskQuestion() 方法目前只返回 Answer 属性的值，但响应对象还包含其他信息，比如引用。让我们在 Models 文件夹下创建一个名为 KernelResponse 的新类，它将以更完整的方式存储 AskQuestion() 方法的响应：

```csharp
using Microsoft.KernelMemory;

namespace AzureRag.Models
{
    public class KernelResponse
    {
        public string Answer { get; set; }
        public List<Citation> Citations { get; set; }
    }
}
```

现在让我们更新 AskQuestion() 类的 MemoryService 方法以返回这个类的实例：

```csharp
public async Task<KernelResponse> AskQuestion(string question)
{
    var answer = await kernelMemory.AskAsync(question);
            
    var response = new KernelResponse
    {
        Answer = answer.Result,
        Citations = answer.RelevantSources
    };

    return response;
}
```

如你所见，响应还包含一个名为 RelevantSources 的属性，它是用于生成响应的所有来源的集合。

最后，让我们更新 AskQuestion.razor 页面的 UI 和代码以显示引用：

```razor
@page "/askquestion"
@using System.Text.RegularExpressions
@using System.Web
@using AzureRag.Models
@inject IMemoryService AIService
@rendermode InteractiveServer

<h1>Ask question</h1>

<p>
    Type your question:
</p>
<p>
    <input @bind="question" style="width: 100%;" />
</p>
<p>
    <button class="btn btn-primary" @onclick="Ask">Ask</button>
</p>

<p>
    @if (answer != null)
    {
        <strong>The answer is:</strong> @answer.Answer
        @foreach (var citation in answer.Citations)
        {
            <ul>

                <li><strong>File name:</strong> @citation.SourceName</li>
                <li><strong>File type:</strong>@citation.SourceContentType</li>
            </ul>
        }
    }

</p>

@code {
    private string question;
    private KernelResponse answer;

    private async Task Ask()
    {
        answer = await AIService.AskQuestion(question);

    }
}
```

在答案下方，现在我们将使用 foreach 语句来显示存储在 Citation 属性中的引用列表，其中包括文件名（ SourceName ）和文件类型（ SourceContentType ）。在 @code 部分，我们只是将 answer 属性的类型从 string 更改为 KernelResponse 。

现在按 F5 并再次尝试提问，比如"什么是 Contoso Electronics？"。你将看到，在答案下方，你会找到一系列引用，其中包括对我们之前上传的 employee_handbook.pdf 文件的引用：

![引用示例](https://www.developerscantina.com/p/kernel-memory/citations.png)

## 将向量数据库存储在磁盘上

如果你停止并重新启动应用程序，然后再次询问相同的问题，体验将会完全不同：

![未找到信息](https://www.developerscantina.com/p/kernel-memory/info-not-found.png)

这是因为向量数据库存储在内存中，这意味着当应用程序停止时，它会丢失。为了避免这个问题，我们可以将向量数据库存储在磁盘上。要做到这一点，我们需要修改 MemoryService 类的代码，通过添加一个新方法来初始化向量数据库：

```csharp
var directory = Path.GetDirectoryName(Environment.ProcessPath);
var path = Path.Combine(directory, "Memory");
if (!Directory.Exists(path))
{
    Directory.CreateDirectory(path);
}

kernelMemory = new KernelMemoryBuilder()
    .WithAzureOpenAITextGeneration(chatConfig)
    .WithAzureOpenAITextEmbeddingGeneration(embeddingConfig)
    .WithSimpleVectorDb(path)
    .Build<MemoryServerless>();
```

我们向 WithSimpleVectorDb() 方法添加了一个新参数，即我们想要存储向量数据库的路径。在本例中，我们将把它存储在一个名为 Memory 的文件夹中，该文件夹是应用程序运行位置的子文件夹。现在再次启动应用程序，并再次使用 AddContent 页面上传 PDF。这一次，你将在项目中找到一个名为 Memory 的新文件夹，其中包含一系列文件：

![存储在磁盘上的向量数据库](https://www.developerscantina.com/p/kernel-memory/vector-database-disk.png)

现在，如果你停止并重新启动应用程序，向量数据库将从磁盘加载，你将能够提问而无需再次上传 PDF 文件。

那么，如果你想将应用程序投入生产环境并使用更结构化的向量数据库，比如 Azure AI Search，该怎么办？得益于对多个提供程序的内置支持，你无需更改代码。你只需要使用 KernelMemoryBuilder 提供的不同扩展方法，如下例所示：

```csharp
string searchApiKey = "AzureSearch:ApiKey";
string searchEndpoint = "AzureSearch:Endpoint";

kernelMemory = new KernelMemoryBuilder()
    .WithAzureOpenAITextGeneration(chatConfig)
    .WithAzureOpenAITextEmbeddingGeneration(embeddingConfig)
    .WithAzureAISearchMemoryDb(searchEndpoint, searchApiKey)
    .Build<MemoryServerless>();
```

你只需使用 WithAzureAISearchMemoryDb() 方法并传递 Azure 门户提供的端点和 API 密钥作为参数来初始化 Azure AI 搜索。现在，如果你尝试再次使用 AddContent 页面处理 PDF 文件，你将在 Azure 门户中看到已创建了一个名为 default 的新索引。如果你探索内容，你会看到文档已被分割成多个块，并且每个块都被转换为一个向量：

![Azure AI Search](https://www.developerscantina.com/p/kernel-memory/azure-ai-search.png)

## 总结

这是一篇相当长的文章！我们已经了解了如何使用 Kernel Memory 来简化 RAG 的实现，这使得应用程序能够使用 LLM 基于私有数据生成答案。我们已经了解了如何使用 Kernel Memory 将文档转换为向量并存储到向量数据库中，以及如何使用它进行语义搜索并使用 LLM 生成响应。我们还了解了如何将向量数据库存储在磁盘上，以及如何使用更结构化的向量数据库，如 Azure AI Search。

在下一篇文章中，我们将了解如何通过特定插件在 Semantic Kernel 中利用 Kernel Memory。敬请期待！

与此同时，你可以点击**阅读原文**并在其中找到本文中使用的代码，项目名为 KernelMemory。
