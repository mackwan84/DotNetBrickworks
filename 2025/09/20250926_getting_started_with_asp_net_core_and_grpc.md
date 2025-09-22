# [ASP.NET Core 和 gRPC 入门指南](https://blog.jetbrains.com/dotnet/2021/07/19/getting-started-with-asp-net-core-and-grpc/)

作为开发者，我们正处在 API 设计理念的复兴时期。有 REST、HTTP API、GraphQL、SOAP，而且这个列表还在不断增长。面对如此多的选择，我们不难理解，很难决定应该使用哪一种。

今天，我们想向你介绍 gRPC，这是.NET 开发人员在设计 API 时可以采用的较新方法之一。该方法因其高效性、基于契约的设计、跨技术堆栈的互操作性以及代码生成工具而日益受到欢迎。

在阅读完本文后，你将对 gRPC 是什么、它的工作原理以及如何实现 ASP.NET Core 客户端/服务器演示有一个了解。届时，你将拥有必要的工具来判断这种方法是否适合你。

闲话少说，让我们开始吧！

## 什么是 gRPC？

gRPC 代表 google Remote Procedure Calls（谷歌远程过程调用），是一种架构服务模式，帮助开发人员以调用进程内对象方法的相同方式构建和使用分布式服务。gRPC 的目标是通过预定义的接口集合，使分布式应用程序和服务对客户端和服务器实现都更易于管理。

> gRPC 是一个现代开源的高性能远程过程调用（RPC）框架，可在任何环境中运行。它能够高效地连接数据中心内部和跨数据中心的服务，并提供可插拔的负载均衡、跟踪、健康检查和身份验证支持。它也适用于分布式计算的最后一公里，将设备、移动应用程序和浏览器连接到后端服务。
> — gRPC About Page

gRPC 与技术栈无关，支持 Java、Go、Python、Ruby、C#和 F#等多种客户端和服务器语言。总共有十种 gRPC 客户端库语言实现。这种方法允许构建多样化的解决方案系统，利用每个生态系统的优势来提供整体价值。它的互操作性来自于采用 HTTP/2、Protocol Buffer（Protobuf）和其他格式等普遍技术。

gRPC 的优势包括以下几点：

- 非常适合低延迟、高可扩展性的分布式系统。
- 服务和消息的设计与实现是分离的。
- 分层设计允许实现可扩展性。
- 支持流式传输、同步、异步、取消和超时等高级概念。

gRPC 已在众多组织和现有产品中得到广泛应用，如 Netflix、Cloudflare、Google 和 Docker。许多人采用 gRPC 是因为它能够以一致且符合语言习惯的方式提供基于 HTTP/2 的双向流式传输。通过利用 HTTP/2，许多 gRPC 采用者还能获得基于 TLS 的安全性，这为分布式系统增加了一层保护。使用 HTTP/2 这样的协议也使 gRPC 采用者能够在其托管平台（无论是本地还是云环境）中添加遥测和诊断工具。

总之，gRPC 是一种快速构建分布式系统的方法，专注于互操作性和性能。虽然第三方可以访问这些服务，但 gRPC 系统通常是为内部分布式系统构建的。

## 什么是 Protocol Buffers

虽然 gRPC 确实支持多种序列化格式，但默认且最常用的是 Protocol Buffers。Protocol Buffers 是一种开源的序列化结构，强调效率。我们可以使用协议缓冲区语言定义消息，该语言目前有两个公开版本 proto2 和 proto3 。

该语言本身包含消息、标量值类型、可选和默认值、枚举等构造。我们通常可以在 .proto 文件中找到 Protocol Buffers 的定义。

让我们从 SearchRequest 的基本 message 定义开始。

```proto
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}
```

我们可以看到，在消息定义中有三个属性： query 、 page_number 和 result_per_page 。这些属性在调用方必须指定的要求上也有所不同。最后，我们注意到每行末尾的字段编号分配。这些编号定义了数据在编码过程中的顺序，这被称为消息二进制格式。由于知道了数据的顺序，Protobuf 减少了对字段名额外标记的需求，而在 JSON 或 XML 等其他格式中可以找到这些标记。这也意味着一旦消息格式投入使用，字段的顺序就不能更改。

Protobuf 允许我们定义消息、数据和服务。正如我们将在下一节中看到的，C# 工具可以利用 .proto 文件来生成在其他 API 方法中构建服务所需的大部分样板代码。

## gRPC 和 .NET 示例

.NET 团队的成员们一直在努力工作，为 .NET 社区带来了一流的 gRPC 支持。在这篇博客文章的其余部分，我们将创建一个独立的 gRPC 演示，其中包含服务器和客户端。我建议安装前面提到的 Protobuf 插件。

我们的第一步是使用空模板创建一个新的 ASP.NET Core 应用程序。

当我们的开发环境初始化完成后，我们需要添加几个 NuGet 包。这些包将帮助生成用于实现消息和服务的 gRPC 基类。要快速添加所有必需的包，请将以下 ItemGroup 粘贴到你的 .csproj 文件中。

```xml
<ItemGroup>
    <PackageReference Include="Google.Protobuf" Version="3.15.5" />
    <PackageReference Include="Grpc.AspNetCore" Version="2.35.0" />
    <PackageReference Include="Grpc.Net.Client" Version="2.35.0" />
    <PackageReference Include="Grpc.Tools" Version="2.36.1">
        <PrivateAssets>all</PrivateAssets>
        <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
</ItemGroup>
```

你也可以使用 NuGet 工具安装以下软件包：

- Google.Protobuf
- Grpc.AspNetCore
- Grpc.Net.Client
- Grpc.Tools

我们的下一步是在项目下添加一个新的 Protos 文件夹，并创建一个名为 greet.proto 的空文件。

我们将使用 Protobuf 定义语言来规划我们的 gRPC 服务、请求和响应。

```proto
syntax = "proto3";

option csharp_namespace = "GrpcDemo";

// 服务定义
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply);
}

// 请求消息定义
message HelloRequest {
  string name = 1;
}

// 响应消息定义
message HelloReply {
  string message = 1;
}
```

Protobuf 还有一个 option 关键字，允许定义额外的元数据，我们的生成器可以使用这些元数据来创建惯用代码。在这种情况下，gRPC 工具将在 GrpcDemo 命名空间中生成我们的基类。

最后一步是在我们的构建过程中利用生成器。幸运的是，这就像在我们的 .csproj 文件中添加一个元素一样简单。

```xml
<ItemGroup>
    <Protobuf Include="Protos\greet.proto" GrpcServices="Server,Client" />
</ItemGroup>
```

我们应该注意到 GrpcServices 属性。它告诉工具我们当前应用程序将在哪一侧，服务器或客户端。在我们的例子中，我们将生成两者来创建我们的演示。

## gRPC 服务

让我们开始实现我们的 gRPC 服务。查看我们之前的定义，我们可以看到一个单一的 SayHello 方法。在我们的项目中，我们可以添加一个新的 C#类。我们将把这个服务托管在 ASP.NET Core 中，这意味着我们将能够使用 ASP.NET Core 提供的所有功能，例如依赖注入和服务。

在一个新的 C# 文件中，我们创建一个派生自 Greeter.GreeterBase 的 Service 类。如果这个类不可用，请确保先构建你的解决方案，它将在我们第一次编译后可用。

```csharp
namespace GrpcDemo
{
    public class Service : Greeter.GreeterBase
    {

    }
}
```

在类中输入 override ，我们将看到在 greet.proto 文件中找到的方法定义。

让我们实现 Service 及其 SayHello 方法。

```csharp
using System.Threading.Tasks;
using Grpc.Core;
using Microsoft.Extensions.Logging;

namespace GrpcDemo
{
    public class Service : Greeter.GreeterBase
    {
        private readonly ILogger<Service> _logger;
        public Service(ILogger<Service> logger)
        {
            _logger = logger;
        }

        public override Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
        {
            return Task.FromResult(new HelloReply
            {
                Message = "Hello " + request.Name
            });
        }
    }
}
```

## gRPC 客户端

我们将利用 ASP.NET Core 的后台服务来托管一个与 gRPC 服务本地的客户端。虽然我们在同一进程中托管客户端和服务器，但它们仍将通过 HTTP/2 协议进行通信。

我们将首先创建一个名为 Client 的新 C#类，它也将派生自 BackgroundService 类，使我们能够在 ASP.NET Core 的托管服务基础设施中托管它。

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Grpc.Net.Client;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace GrpcDemo
{
    public class Client : BackgroundService
    {
        private readonly ILogger<Client> _logger;
        private readonly string _url;

        public Client(ILogger<Client> logger, IConfiguration configuration)
        {
            _logger = logger;
            _url = configuration["Kestrel:Endpoints:gRPC:Url"];
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            using var channel = GrpcChannel.ForAddress(_url);
            var client = new Greeter.GreeterClient(channel);

            while (!stoppingToken.IsCancellationRequested)
            {
                var reply = await client.SayHelloAsync(new HelloRequest
                {
                    Name = "Worker"
                });

                _logger.LogInformation("Greeting: {reply.Message} -- {DateTime.Now}");
                 await Task.Delay(1000, stoppingToken);
             }
         }
     }
 }
 ```

 我们可以看到，基本的客户端元素发生在我们工作服务的 ExecuteAsync 方法中。

 1. 我们使用 URL 创建一个新的 gRPC 通道。
 2. 我们使用生成的客户端定义并传入通道。
 3. 我们使用 HelloRequest 合约调用 SayHelloAsync 方法。

 现在我们已经实现了服务器和客户端，让我们在 ASP.NET Core 中将它们连接起来。

## ASP.NET Core 设置

 我们有两个地方需要添加客户端和服务器的实现。让我们先从服务器开始，因为这很可能是大多数人开始使用 gRPC 的地方。

 在我们的 Web 项目的 Startup 文件中，我们需要在服务集合中注册 gRPC。在 ConfigureServices 方法中，我们需要调用 AddGrpc 扩展方法。

 ```csharp
 public void ConfigureServices(IServiceCollection services)
{
    services.AddGrpc();
}
```

在我们的 Configure 方法中，我们需要为 Service 实现注册端点。我们可以使用在 IEndpointRouteBuilder 接口上使用的 MapGrpcService 扩展方法来注册 gRPC 服务。

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapGet("/", async context => { await context.Response.WriteAsync("Hello World!"); });
    endpoints.MapGrpcService<Service>();
});
```

我们的服务已准备好开始接收 gRPC 客户端请求。现在让我们设置客户端。在我们的 Program 文件中，我们需要将 Client 添加为托管服务。让我们修改 CreateHostBuilder 方法来实现这一点。

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace GrpcDemo
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                })
                .ConfigureServices(svc =>
                {
                    svc.AddHostedService<Client>();
                });
    }
}
```

下一步根据我们的开发环境是可选的。.NET gRPC 实现在 macOS 上不支持 TLS，以及在 Linux 设备上的 ASP.NET Core 开发证书信任级别方面存在一些开发注意事项。.NET 团队将在.NET 的未来版本中解决这些问题。请修改 appsettings.json 文件，以便在所有操作系统平台上都能正常演示。

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "Kestrel": {
    "Endpoints" : {
      "Http" : {
        "Url" : "https://localhost:5001",
        "Protocols": "Http1AndHttp2"
      },
      "gRPC": {
        "Url": "http://localhost:5000",
        "Protocols": "Http2"
      }
    }
  }
}
```

我们正在配置 ASP.NET Core 的服务器 Kestrel，使其仅通过 HTTP 监听 gRPC 调用，并使用 HTTP/2 协议。

好的，我们准备运行演示了！应用程序应该会启动，我们的后台服务将开始调用我们的 gRPC 服务。我们将在控制台输出窗口中看到结果。

恭喜！我们成功了。我们刚刚成功构建了一个客户端/服务器的 gRPC 演示。

## 端点调试器

眼尖的 ASP.NET Core 开发者可能已经注意到，我们在 ASP.NET Core 中使用端点来托管 gRPC 服务。为了确认这一点，我们可以修改根路径端点，使其返回所有已注册端点的结果集。将现有的"Hello, World"代码替换为以下实现。你可能需要添加对 Microsoft.AspNetCore.Routing 命名空间的引用。

```csharp
endpoints.MapGet("/", async context =>
{
    var endpointDataSource = context
        .RequestServices.GetRequiredService<EndpointDataSource>();

    await context.Response.WriteAsJsonAsync(new
    {
        results = endpointDataSource
            .Endpoints
            .OfType<RouteEndpoint>()
            .Where(e => e.DisplayName?.StartsWith("gRPC") == true)
            .Select(e => new
            {
                name = e.DisplayName, 
                pattern = e.RoutePattern.RawText,
                order = e.Order
            })
            .ToList()
    });
});
```

重新运行我们的应用程序，我们将看到所有已注册到 ASP.NET Core 的 gRPC 端点。

```json
// 20210310105244
// https://localhost:5001/

{
  "results": [
    {
      "name": "gRPC - /Greeter/SayHello",
      "pattern": "/Greeter/SayHello",
      "order": 0
    },
    {
      "name": "gRPC - Unimplemented service",
      "pattern": "{unimplementedService}/{unimplementedMethod}",
      "order": 0
    },
    {
      "name": "gRPC - Unimplemented method for Greeter",
      "pattern": "Greeter/{unimplementedMethod}",
      "order": 0
    }
  ]
}
```

你会注意到 gRPC 端点的路径模式是一个简单的文字字符串。gRPC 方法使用这种技术来避免解析复杂的 URL 路径和查询字符串，从而绕过解析复杂 URI 的开销。

## 总结

gRPC 的诞生源于构建分布式服务系统互操作性的需求，它不是重新发明轮子，而是拥抱现有技术。其优势来自于高效和标准化的内部服务，以及类似于进程内对象调用的编程模型。gRPC 及其二进制序列化非常适合在网络间传输大量数据。同时，该技术采用 HTTP/2 协议，提供了自然的扩展点和现有工具，帮助理解 gRPC 系统的功能。

采用微服务方法的多语言开发团队将会发现，在服务边界间标准化通信具有诸多好处。由于我们热爱.NET，开发人员会发现使用优秀的生成工具来构建和使用 gRPC 定义是一种愉快且熟悉的体验。最后，与 ASP.NET Core 的集成使开发团队能够轻松采用 gRPC，并按照自己的节奏应用它。

今天就到这里。期待与你再次相见。
