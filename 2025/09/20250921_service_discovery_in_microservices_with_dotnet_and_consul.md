# [使用 .NET 和 Consul 实现微服务中的服务发现](https://www.milanjovanovic.tech/blog/service-discovery-in-microservices-with-net-and-consul)

微服务彻底改变了我们构建和扩展应用程序的方式。通过将大型系统分解为更小、独立的服务，我们获得了灵活性、敏捷性以及快速适应不断变化需求的能力。然而，微服务系统也是非常动态的。服务可以随时出现和消失，扩展或缩减，甚至可以在你的基础设施内移动。

这种动态性带来了一个重大挑战。你的服务如何可靠地找到彼此并进行通信？

硬编码 IP 地址和端口是一种脆弱的做法。如果一个服务实例改变了位置或者启动了一个新实例，你的整个系统可能会陷入停顿。

服务发现充当微服务的中央目录。它提供了一种机制，让服务能够注册自己并发现其他服务的位置。

今天，我们将了解如何在 .NET 微服务中使用 Consul 实现服务发现。

## 什么是服务发现？

服务发现是一种允许开发人员使用逻辑名称而非物理 IP 地址和端口来引用外部服务的模式。它提供了一个集中的位置供服务注册自身。客户端可以查询服务注册表以找出服务的物理地址。这是大规模分布式系统中的常见模式，如 Netflix 和 Amazon。

以下是服务发现流程的样子：

- 该服务将在服务注册表中注册自己
- 客户端必须查询服务注册表以获取物理地址
- 客户端使用解析到的物理地址向服务发送请求

![服务注册](https://www.milanjovanovic.tech/blogs/mnw_097/service_discovery_flow.png?imwidth=3840)

当我们想要调用多个服务时，同样的概念也适用。每个服务都会在服务注册表中注册自己。客户端使用逻辑名称来引用服务，并从服务注册表中解析物理地址。

![服务发现](https://www.milanjovanovic.tech/blogs/mnw_097/service_discovery_microservices.png?imwidth=3840)

服务发现最流行的解决方案是 Netflix Eureka 和 HashiCorp Consul。

微软在 Microsoft.Extensions.ServiceDiscovery 库中也提供了一个轻量级解决方案。它使用应用程序设置来解析服务的物理地址，因此仍需要一些手动工作。不过，你可以将服务位置存储在 Azure 应用配置中，以实现集中式服务注册表。我将在未来的文章中探讨这个服务发现库。

但现在我想向你展示如何将 Consul 与 .NET 应用程序集成。

## 设置 Consul 服务器

在本地运行 Consul 服务器的最简单方法是使用 Docker 容器。你可以创建一个 hashicorp/consul 镜像的容器实例。

以下是在 docker-compose 文件中配置 Consul 服务的示例：

```yaml
consul:
  image: hashicorp/consul:latest
  container_name: Consul
  ports:
    - '8500:8500'
```

如果你访问 localhost:8500 ，将会看到 Consul 数据看板。

![Consul 数据看板](https://www.milanjovanovic.tech/blogs/mnw_097/consul_dashboard.png?imwidth=3840)

现在，让我们看看如何将我们的服务注册到 Consul。

## 使用 Consul 在 .NET 中进行服务注册

我们将使用 Steeltoe Discovery 库来实现与 Consul 的服务发现。Consul 客户端实现让你的应用程序能够向 Consul 服务器注册服务，并发现其他应用程序注册的服务。

让我们安装 Steeltoe.Discovery.Consul 库：

```bash
Install-Package Steeltoe.Discovery.Consul
```

我们必须通过调用 AddServiceDiscovery 并显式配置 Consul 服务发现客户端来配置一些服务。另一种方法是调用 AddDiscoveryClient ，它在运行时使用反射来确定可用的服务注册表。

```csharp
using Steeltoe.Discovery.Client;
using Steeltoe.Discovery.Consul;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddServiceDiscovery(o => o.UseConsul());

var app = builder.Build();

app.Run();
```

最后，我们的服务可以通过应用程序设置配置逻辑服务名称来注册到 Consul。当应用程序启动时， reporting-service 逻辑名称将被添加到 Consul 服务注册表中。Consul 将存储该服务的相应物理地址。

```json
{
  "Consul": {
    "Host": "localhost",
    "Port": 8500,
    "Discovery": {
      "ServiceName": "reporting-service",
      "Hostname": "reporting-api",
      "Port": 8080
    }
  }
}
```

当我们启动应用程序并打开 Consul 数据看板时，我们应该能够看到 reporting-service 及其相应的物理地址。

![Consul 数据看板](https://www.milanjovanovic.tech/blogs/mnw_097/consul_dashboard_with_service.png?imwidth=3840)

## 使用服务发现

我们可以在使用 HttpClient 进行 HTTP 调用时使用服务发现。服务发现允许我们使用逻辑名称来表示想要调用的服务。发送网络请求时，服务发现客户端会将逻辑名称替换为正确的物理地址。

在此示例中，我们将 ReportingServiceClient 类型化客户端的基础地址配置为 <http://reporting-service> ，并通过调用 AddServiceDiscovery 来添加服务发现功能。

负载均衡是一个可选步骤，我们可以通过调用 AddRoundRobinLoadBalancer 或 AddRandomLoadBalancer 来配置它。你还可以通过提供 ILoadBalancer 实现来配置自定义负载均衡策略。

```csharp
builder.Services
    .AddHttpClient<ReportingServiceClient>(client =>
    {
        client.BaseAddress = new Uri("http://reporting-service");
    })
    .AddServiceDiscovery()
    .AddRoundRobinLoadBalancer();
```

我们可以像使用常规 HttpClient 一样使用 ReportingServiceClient 类型化客户端来发送请求。服务发现客户端将请求发送到外部服务的 IP 地址。

```csharp
app.MapGet("articles/{id}/report",
    async (Guid id, ReportingServiceClient client) =>
    {
        var response = await client
            .GetFromJsonAsync<Response>($"api/reports/article/{id}");

        return response;
    });
```

## 总结

服务发现通过自动化服务注册和发现简化了微服务的管理。这消除了手动配置更新的需要，降低了出错的风险。

服务可以按需发现彼此的位置，确保即使服务环境不断变化，通信渠道也能保持畅通。通过使服务能够在发生中断或故障时发现替代的服务实例，服务发现增强了微服务系统的整体弹性。

掌握服务发现将为你构建现代分布式应用程序提供强大工具。

今天就到这里。期待与你再次相见。
