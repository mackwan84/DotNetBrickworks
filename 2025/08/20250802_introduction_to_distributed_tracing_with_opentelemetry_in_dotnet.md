# [.NET 中使用 OpenTelemetry 进行分布式追踪](https://www.milanjovanovic.tech/blog/introduction-to-distributed-tracing-with-opentelemetry-in-dotnet)

如果你正在构建或维护分布式.NET 应用程序，了解它们的行为是确保可靠性和性能的关键。

分布式系统提供了灵活性，但也引入了复杂性，使得故障排除变得令人头疼。了解请求如何通过你的系统流动对于调试和性能优化至关重要。

**OpenTelemetry** 是一个开源的可观测性框架，使这一切成为可能。

在这篇文章中，我们将深入了解 OpenTelemetry 是什么，如何在你的 .NET 项目中使用它，以及它提供的强大洞察力。

## OpenTelemetry 简介

OpenTelemetry（OTel）是一个供应商中立的、开源的标准，用于对应用程序进行仪器化以生成遥测数据。OpenTelemetry 包含 API、SDK、工具和集成，用于创建和管理这些遥测数据（跟踪、指标和日志）。

遥测数据包括：

- **跟踪**：表示请求在分布式系统中的流动，展示服务之间的时序和关系。
- **指标**：随时间对系统行为的数值测量（例如，请求计数、错误率、内存使用）。
- **日志**：带有丰富上下文信息的文本事件记录。结构化日志。

OpenTelemetry 提供了一种统一的方式来收集这些数据，使得理解和监控复杂分布式应用的性能和健康状况变得更加容易。

我们可以将收集的遥测数据导出到一个能够处理并提供分析界面的服务中。

我们将配置 OpenTelemetry，以直接将跟踪信息导出到 Jaeger。

## 将 OpenTelemetry 添加到 .NET 应用程序中

OpenTelemetry 提供库和 SDK，以便向你的 .NET 应用程序中添加代码（仪器化）。这些仪器化自动捕获我们感兴趣的跟踪、指标和日志。

我们将安装以下 NuGet 包：

```bash
# 自动化追踪、度量
Install-Package OpenTelemetry.Extensions.Hosting

# Telemetry 数据导出
Install-Package OpenTelemetry.Exporter.OpenTelemetryProtocol

# Telemetry 测量器
Install-Package OpenTelemetry.Instrumentation.Http
Install-Package OpenTelemetry.Instrumentation.AspNetCore
Install-Package OpenTelemetry.Instrumentation.EntityFrameworkCore
Install-Package OpenTelemetry.Instrumentation.StackExchangeRedis
Install-Package Npgsql.OpenTelemetry
```

一旦我们安装了这些 NuGet 包，就该配置一些服务了。

```csharp
services
    .AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService(serviceName))
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddRedisInstrumentation()
            .AddNpgsql();

        tracing.AddOtlpExporter();
    });
```

- **AddAspNetCoreInstrumentation** - 这启用了 ASP.NET Core 框架的测量。
- **AddHttpClientInstrumentation** - 这启用了 HttpClient 的测量。
- **AddEntityFrameworkCoreInstrumentation** - 这启用了 EF Core 的测量。
- **AddRedisInstrumentation** - 这启用了 Redis 的测量。
- **AddNpgsql** - 这启用了 Npgsql 的测量。

配置了所有这些测量器后，我们的应用程序将在运行时开始收集大量有价值的跟踪信息。

我们还需要为添加了 **AddOtlpExporter** 的导出器配置一个环境变量，以确保其正确工作。我们可以通过应用程序设置来设置 **OTEL_EXPORTER_OTLP_ENDPOINT** 。这里指定的地址将指向一个本地的 Jaeger 实例。

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

## 在本地运行 Jaeger

Jaeger 是一个开源的分布式追踪平台。Jaeger 能够映射请求和数据在分布式系统中的流动过程。这些请求可能涉及到对多个服务的调用，而 Jaeger 知道如何将这些信息整合在一起。

在 Docker 容器中运行 Jaeger 的方法如下：

```bash
docker run -d -p 4317:4317 -p 16686:16686 jaegertracing/all-in-one:latest
```

我们使用 jaegertracing/all-in-one:latest 镜像，并暴露 4317 以接收遥测数据。Jaeger 用户界面将在 16686 端口上暴露。

## 分布式追踪

在安装了 OpenTelemetry 库并在我们的应用程序中配置了跟踪之后，我们可以发送一些请求以生成遥测数据。然后我们可以访问 Jaeger，开始分析我们的分布式跟踪。

### 注册新用户

以下是注册新用户到系统的示例。我们正在访问 API 网关（ Evently.Gateway ）服务，该服务将请求代理到 Evently.Api 服务。你可以看到 Evently.Api 服务在进行数据库中持久化新记录之前，会发出几个 HTTP 请求。

![分布式追踪](https://www.milanjovanovic.tech/blogs/mnw_086/trace_1.png?imwidth=1920)

### 使用 MassTransit 发布消息

这里还有另一个分布式追踪示例，我们通过消息总线发布 UserRegisteredIntegrationEvent 。你可以看到它被两个不同的服务消费，这些服务将一些数据写入数据库。

![分布式追踪](https://www.milanjovanovic.tech/blogs/mnw_086/trace_2.png?imwidth=1920)

### 检查额外的跟踪信息

分布式跟踪可以包含一些有用的上下文信息。以下是一个表示数据库命令的示例跟踪。这来自 PostgreSQL 的检测工具，我们可以看到正在执行的 SQL 查询。

![分布式追踪](https://www.milanjovanovic.tech/blogs/mnw_086/trace_3.png?imwidth=1920)

### 复杂的分布式追踪

这是一个更复杂的分布式追踪，它包括：

- 三个 .NET 应用程序
- PostgreSQL 数据库
- Redis 缓存

我们正在发送一个请求以获取客户的购物车信息。该请求首先会到达 API 网关，然后由网关代理到拥有数据的 Evently.Ticketing.Api 服务。然而， Evently.Ticketing.Api 服务需要联系 Evently.Api 服务以获取授权信息。所有这些操作最终形成了你下面可以看到的分布式追踪。

![分布式追踪](https://www.milanjovanovic.tech/blogs/mnw_086/trace_4.png?imwidth=1920)

## 总结

理解现代应用程序，尤其是分布式应用程序，可能会让人感到非常困惑。OpenTelemetry 就像是为你的系统配备了 X 光透视能力。

虽然添加 OpenTelemetry 需要一些前期工作，但请将其视为一项投资。当问题出现时，这项投资会带来巨大的回报。与其进行慌乱的猜测，不如拥有精确的数据，快速定位问题。

OpenTelemetry 是解决你所有问题的灵丹妙药吗？不是。

但它是一个极好的工具，可以添加到你的故障排除工具库中，尤其是随着你的.NET 应用程序不断增长和变得更加复杂时。

今天就到这里。期待与你再次相见。
