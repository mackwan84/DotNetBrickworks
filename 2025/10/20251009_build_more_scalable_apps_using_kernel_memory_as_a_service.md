# [使用 Kernel Memory 服务构建更具可扩展性的应用](https://www.developerscantina.com/p/kernel-memory-service/)

前几天，我们介绍了 Kernel Memory，这是由微软创建的一个非常强大的开源服务，你可以利用它来大大简化应用程序中检索增强生成（RAG）模式的实现，从而更轻松地构建由你的数据驱动的 AI 体验。在那篇文章中，我们介绍了 Kernel Memory 在无服务器模式下的使用，这意味着 Kernel Memory 实例与使用它的应用程序在同一进程中运行。这种方法使实现和部署更加简单，但效率不高，因为 Kernel Memory 实例与应用程序实例绑定，无法在同一个应用程序的多个实例之间共享。此外，无论你的应用程序是前端、后端还是 API 等，其性能都会受到 Kernel Memory 工作的影响，因为它在同一进程中运行。

而今天，我们将介绍如何将 Kernel Memory 设置为服务使用。作为一个独立的应用程序，你可以将其部署在与主应用程序不同的环境中，从而允许你设置不同的扩展规则。

## 将 Kernel Memory 配置为服务

作为服务的 Kernel Memory 简单地以 REST API 形式暴露，你的应用程序可以调用该 API 来执行我们在上一篇文章中学习的各种操作，例如上传新文档或询问有关你数据的问题。这意味着无论你使用哪种技术栈，只要你可以发送 HTTP 请求，就可以轻松使用 Kernel Memory。但是，如果你使用 C# 和 .NET，则可以使用 NuGet 包而不是手动调用 REST API，该包实现了与无服务器模式下使用的类相同的基础接口。这意味着你可以轻松地从在无服务器模式下使用 Kernel Memory 切换到将其作为服务使用，而无需更改用于摄取和交互内容的代码。

作为服务的 Kernel Memory 有两种可用方式：

- 作为 .NET Web API，它具有灵活性，可以在支持 .NET 的任何环境中执行，无论是在本地还是在云端。
- 作为 Docker 容器，这样你可以轻松利用 Azure Kubernetes Services 或 Azure App Containers 等云平台来部署和扩展它。

首先，我们需要从官方 GitHub 仓库克隆或下载一份 Kernel Memory。

无论你喜欢使用哪种方法，第一步都是配置服务。当我们使用无服务器模式时，我们通过代码来完成此操作，就像我们在专门的文章中看到的那样：

```csharp
kernelMemory = new KernelMemoryBuilder()
    .WithAzureOpenAITextGeneration(chatConfig)
    .WithAzureOpenAITextEmbeddingGeneration(embeddingConfig)
    .WithAzureAISearchMemoryDb(searchEndpoint, searchApiKey)
    .Build<MemoryServerless>();
```

当我们将其作为服务使用时，我们需要使用存储在应用程序配置文件中的设置，即名为 appsettings.json 的文件。你可以通过打开存储库中 service\Service 文件夹下的 appsettings.Development.json 文件来查看其定义。你会注意到该文件相当复杂，因为 Kernel Memory 支持多种设置和服务，每种都需要自己的配置。为了简化配置文件的设置，Kernel Memory 提供了一个向导，可以为你生成配置文件。在 service\Service 文件夹中打开终端并运行以下命令：

```bash
dotnet run setup
```

向导会询问你一系列问题，以了解你想要启用哪些服务以及如何配置它们。让我们逐步看看：

![配置 Kernel Memory 服务的向导](https://www.developerscantina.com/p/kernel-memory-service/service-wizard.png)

**Kernel Memory 服务应如何运行并处理内存和文档摄取？** 你可以选择将 Kernel Memory 用作 Web 服务还是仅用于摄取内容。在第二种情况下，你将需要提供自己的机制来排队摄取作业。在标准场景中，首选的选择是 **"Web Service with Asynchronous Ingestion Handlers"**。

那么， **我们需要使用 API 密钥保护 Web 服务？** 如果你计划将服务暴露到互联网，强烈建议选择此选项。这样，在调用服务时你将需要提供 API 密钥，否则请求将会失败。如果不开启此选项，任何知道你终端节点 URL 的人都能够使用你的服务实例。如果你选择"是"，系统将要求你提供两个 API 密钥。

接着， **我们需要启用 OpenAPI swagger 文档吗？** Kernel Memory 服务内置支持 OpenAPI，这是一种描述 API 工作原理的开放格式。通过将此选项设置为"是"，你将在 /swagger/index.html 端点公开 Swagger 文档，这将使你能够轻松理解 API 的工作原理并对其进行测试。

然后，**需要使用哪种队列服务吗？** Kernel Memory 支持使用队列来更好地扩展文档的摄取。为了测试，你只需选择 SimpleQueues，它会将数据存储在内存中。

**服务应该在哪里存储文件？** Kernel Memory 将在此位置存储将被摄取的文件。同样，为了测试目的，你可以选择 SimpleFileStorage，它只会使用本地文件系统。

**应该使用哪种服务从图像中提取文本？** Kernel Memory 支持将图像作为文档类型进行摄取。但是，要启用此功能，你需要将 Kernel Memory 连接到 Azure AI Document Intelligence，这是一项能够从图像中提取文本的 Azure 服务。如果你选择启用它，你将需要提供可以从 Azure 门户检索的终端点和 API 密钥。

**在导入数据时，是生成嵌入向量，还是让内存数据库类来处理？** 一些向量数据库提供了在摄取过程中自动生成嵌入向量的选项。在我的案例中，我将使用 Azure AI Search 作为向量数据库，因此我会选择"是"，让 Kernel Memory 为我生成嵌入向量。

**在搜索文本和/或答案时，应该使用哪种嵌入生成器进行向量搜索？** 嵌入模型是用于将文本转换为向量的模型，以便后续可以存储在向量数据库中。你可以在 Azure OpenAI、OpenAI 或自定义选项之间选择。最常见的选择是 Azure OpenAI 和 OpenAI，在这两种情况下，你都需要提供凭据以向你的服务进行身份验证。

**在搜索答案时，哪个内存数据库服务包含要搜索的记录？** 在这一步中，你可以选择要使用哪种支持的向量数据库来存储嵌入。在我的例子中，我选择了 Azure AI Search。如果你只是在尝试 Kernel Memory，也可以选择 SimpleVectorDb，它不需要任何依赖，但它也不适合生产环境，因为嵌入将存储在内存中（服务重启时会丢失）或磁盘上（这会导致性能不佳）。根据你选择的向量数据库类型，你需要提供访问的端点和凭据。

**在生成答案和合成数据时，应该使用哪个 LLM 文本生成器？** 这是将用于生成你向 Kernel Memory 提出的数据问题的答案的 LLM。你可以在 Azure OpenAI、OpenAI 或 LLama 之间进行选择。

**日志级别？** 最后一步允许你定义要启用的日志记录级别，从跟踪（记录所有内容）到严重（仅记录严重错误）。

好了，我们的配置文件已经准备好了，现在你已经准备好使用这个服务了。

## 使用 Kernel Memory 服务

Kernel Memory 服务只是一个使用 .NET 构建的 Web API，因此你可以在本地机器上、Azure App Service 或任何可以部署 .NET 应用程序的环境中托管它。如果你想在本地测试该服务，可以直接从命令行使用标准的 .NET 命令：

```bash
dotnet run
```

然而，服务很可能第一次无法启动，因为它期望找到一个名为 ASPNETCORE_ENVIRONMENT 的环境变量。这个变量被 ASP.NET Core 用来理解要使用哪个配置文件，并且通常在托管平台（如 Azure App Service）中已经设置。在你的本地计算机上，该变量很可能尚未设置，因此你可以通过在 PowerShell 终端中运行以下命令来手动设置它：

```bash
$env:ASPNETCORE_ENVIRONMENT = 'Development'
```

你需要将其设置为 Development ，因为我们刚刚完成的向导已经创建了一个 appSettings.Development.json 文件，因此在开发环境中它会被自动加载。现在如果你再次尝试 dotnet run ，你将看到熟悉的日志，显示服务正在运行并在 <http://localhost:9001> 上监听。你可以通过打开浏览器并导航到 <http://localhost:9001/swagger/index.html> 来测试服务确实在正常工作。你将看到我们在向导中启用的 Swagger 文档。

![Kernel Memory 的 Swagger 文档](https://www.developerscantina.com/p/kernel-memory-service/swagger.png)

如你所见，该服务提供了各种端点，这些端点与我们可以使用 Kernel Memory 执行的各种操作相匹配，比如上传内容或询问有关数据的问题。然而，在我们的案例中，我们不会手动使用 REST API，而是将在应用程序中利用专门的 NuGet 包。

## 在我们的 C# 应用程序中使用 Kernel Memory 服务

让我们以我们在前一篇文章中构建的示例 Blazor 应用程序为例，并将其更改为使用 Kernel Memory 作为服务。第一步是安装一个名为 Microsoft.KernelMemory.WebClient 的 NuGet 包。正如我们已经看到的，在原始应用程序中，我们使用以下代码来初始化 Kernel Memory：

```csharp
public MemoryService()
{
    kernelMemory = new KernelMemoryBuilder()
        .WithAzureOpenAITextGeneration(chatConfig)
        .WithAzureOpenAITextEmbeddingGeneration(embeddingConfig)
        .WithSimpleVectorDb()
        .Build<MemoryServerless>();
}
```

现在我们只需要用以下行替换它：

```csharp
kernelMemory = new MemoryWebClient("http://localhost:9001", kernelMemoryApiKey);
```

我们创建一个 MemoryWebClient 对象，将服务的 URL 作为参数传递，如果我们在配置向导中选择了身份验证，则可选择性地传递 API 密钥。

最秒的是， MemoryWebClient 类实现了 IKernelMemory 接口，这与我们之前使用的 MemoryServerless 类实现的接口相同。这意味着我们可以在两种实现之间轻松切换，而无需更改其余代码。事实上，我们可以继续使用我们在上一篇文章中看到的相同方法（如 ImportDocumentsAsync() 和 AskAsync() ）来摄取内容并询问有关数据的问题，这些方法在我们示例项目的 MemoryService 类中实现。

现在你可以再次启动 Blazor 应用程序，并尝试执行我们在原始文章中进行的相同操作，例如摄取 Contoso 员工手册并询问相关问题。结果将与原始应用程序完全相同。然而，在运行 Kernel Memory 服务的控制台中，你将从日志中看到所有摄取和问答操作都是由服务执行的，而不是由 Blazor 应用程序执行的。

作为一个使用 .NET 开发的 Web API，现在你可以将其部署在支持扩展和副本等功能的云平台上，这样你就可以管理更繁重的工作负载。在 Microsoft 生态系统中，Azure App Service 是这种场景下的绝佳平台，因为它原生支持 .NET 应用程序。一旦部署完成，你只需要在 MemoryWebClient 构造函数中更改 URL，使其指向云中服务的 URL 即可。

## 将 Kernel Memory 用作 Docker 容器

如果你更喜欢将 Kernel Memory 用作 Docker 容器，你可以利用团队在 Docker Hub 上发布的镜像。

要拉取该镜像，请确保你的机器上已安装并运行 Docker Desktop，然后执行以下命令：

```bash
docker pull kernelmemory/service
```

然而，一旦镜像被拉取，你还不能做太多操作，因为你需要为各种服务提供配置文件。最简单的方法是挂载一个包含你之前创建的配置文件的卷。你需要从 Kernel Memory 存储库的 service/Service 文件夹中执行以下命令：

```bash
docker run --volume .\appsettings.Development.json:/app/appsettings.Production.json -it --rm -p 9001:9001 kernelmemory/service
```

此命令将获取 service/Service 文件夹中的 appsettings.Development.json 文件，并将其作为 appsettings.Production.json 挂载到容器内部。这种重命名是必需的，因为默认情况下，镜像配置的 ASPNETCORE_ENVIRONMENT 值设置为 Production ，因此它会查找 appsettings.Production.json 文件。如果你一切都操作正确，你应该在 Docker Desktop 应用程序中看到 Kernel Memory 服务的日志记录：

![运行 Kernel Memory 服务的 Docker Desktop](https://www.developerscantina.com/p/kernel-memory-service/docker-desktop.png)

从应用程序的角度来看，在本地机器上直接运行 Kernel Memory 或作为 Docker 容器运行并不会改变任何东西。服务将在 URL <http://localhost:9001/> 上暴露，你可以在 <http://localhost:9001/swagger/index.html> 阅读 Swagger 文档。而且，最重要的是，你可以使用我们之前见过的同一个 MemoryWebClient 类来与服务交互。

## 总结

今天，我们了解了如何将 Kernel Memory 作为服务使用，从而构建更具可扩展性的应用程序。通过将 Kernel Memory 作为独立应用程序运行，你可以将其部署在与主应用程序不同的环境中，从而能够设置不同的扩展规则。此外，你可以利用之前在无服务器模式下与 Kernel Memory 交互的相同代码，因为 MemoryWebClient 类实现了与 MemoryServerless 类相同的接口。这意味着你可以轻松地在两种实现之间切换，而无需更改其余代码。

同样，你可以点击**阅读原文**并在其中找到本文中使用的代码。

今天就到这里。期待与你再次相见。
