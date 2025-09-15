# [运行适用于分布式 .NET 应用程序的独立 Aspire 数据看板](https://www.milanjovanovic.tech/blog/standalone-aspire-dashboard-setup-for-distributed-dotnet-applications)

你已经构建了一个分布式.NET 应用程序，其中包含多个服务、数据库和消息队列。现在某个地方运行缓慢，你需要找出原因。

Aspire 数据看板可以作为一个独立容器完美运行，无需完整的编排框架，即可为你提供分布式追踪、结构化日志和实时指标。

虽然 Aspire 的编排功能对于管理分布式应用非常强大，但有时你可能仅仅需要可观测性部分。也许你已经在使用 Docker Compose 或 Kubernetes，或者正在调试一个现有系统。独立的数据看板只需极少的配置，就能为你提供宝贵的遥测可视化信息。

让我们在不到 5 分钟内让它运行起来吧。

## 为什么选择单独运行 Aspire 数据看板？

大多数团队都已经想好了部署方案：Docker Compose、Kubernetes，或者某些特定于平台的编排工具。你并不想仅仅为了实现可观测性而重写所有内容。

独立的 Aspire 数据看板 为开发提供了一个理想的选择：

- **即插即用的可观测性**：只需将一个容器添加到你现有的环境中即可
- **全面支持 OpenTelemetry**：可与任何兼容 OTLP 的应用程序配合使用
- **对开发者友好**：专为本地开发和调试而设计
- **即时价值**：几分钟内即可查看跟踪记录、日志和指标

一个注意事项：它仅限于开发或者测试时使用。非常适合开发和调试，但不适合生产环境。对于生产环境，你可能需要像 Jaeger、Prometheus 或商业 APM 解决方案这样的工具。

## 步骤 1：添加数据看板容器

将这个放入你的 docker-compose.yml 中：

```yaml
aspire-dashboard:
  container_name: aspire-dashboard
  image: mcr.microsoft.com/dotnet/aspire-dashboard:9.0
  ports:
    - 18888:18888
```

简单几句，数据看板正在运行。导航到 <http://localhost:18888> ，然后……你需要一个令牌。

然后，你可以查看容器日志以获取登录链接。数据看板在启动时会生成一个唯一的身份验证令牌：

![身份验证令牌](https://www.milanjovanovic.tech/blogs/mnw_157/aspire_dashboard_login_link.png?imwidth=3840)

点击那个链接你就能进入访问数据看板了。此时数据看板是空的，但很快就会填充数据。

## 步骤 2：连接你的 .NET 服务

你的服务需要知道将遥测数据发送到哪里。请将以下环境变量添加到你的 API 容器中：

```yaml
users.api:
  image: ${DOCKER_REGISTRY-}usersapi
  build:
    context: .
    dockerfile: Users.Api/Dockerfile
  ports:
    - 5100:5100
    - 5101:5101
  environment:
    - OTEL_EXPORTER_OTLP_ENDPOINT=http://aspire-dashboard:18889
    - OTEL_EXPORTER_OTLP_PROTOCOL=grpc
  depends_on:
    - users.database
```

细心的你肯定留意到了，怎么是端口 18889 ？那是 OTLP 数据摄取端点。数据看板在 18888 上监听 UI， 18889 上监听遥测数据。

## 步骤 3：在你的代码中配置 OpenTelemetry

安装必要的 OpenTelemetry 软件包：

```xml
<PackageReference Include="Npgsql.OpenTelemetry" Version="9.0.3" />
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.12.0" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.12.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.12.0" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.12.0" />
```

然后，在你的 Program.cs 中配置 OpenTelemetry：

```csharp
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService(builder.Environment.ApplicationName))
    .WithTracing(tracing => tracing
        .AddHttpClientInstrumentation()
        .AddAspNetCoreInstrumentation()
        .AddNpgsql())
    .WithMetrics(metrics => metrics
        .AddHttpClientInstrumentation()
        .AddAspNetCoreInstrumentation());

builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeScopes = true;
    options.IncludeFormattedMessage = true;
});

builder.Services.AddOpenTelemetry().UseOtlpExporter();
```

以上代码配置了：

- 跟踪 HTTP 调用、ASP.NET Core 请求和数据库查询
- 收集请求时长、响应代码和吞吐量的指标
- 带有完整上下文和格式化消息的结构化日志记录
- 通过 OTLP 将所有内容导出到 Aspire 数据看板

**UseOtlpExporter()** 方法会自动获取你之前配置的 **OTEL_EXPORTER_OTLP_ENDPOINT** 环境变量。

## 你将获得的内容

启动你的应用程序并发出几项请求。数据看板随即亮起，显示数据。

### 结构化日志

每条日志条目都包含完整上下文：跟踪 ID、请求路径和用户身份。点击任意日志即可查看完整的结构化数据。

![结构化日志](https://www.milanjovanovic.tech/blogs/mnw_157/structured_logs.png?imwidth=3840)

### 分布式追踪

查看跨所有服务的完整请求流程。是哪个数据库查询速度慢？是哪次 HTTP 调用失败了？追踪视图会准确地向你展示时间都花在了哪里。

![分布式追踪](https://www.milanjovanovic.tech/blogs/mnw_157/distributed_traces.png?imwidth=3840)

你可以点击进入一条追踪记录，以查看其中的各个跨度及其相关的元数据。

![分布式追踪详情](https://www.milanjovanovic.tech/blogs/mnw_157/distributed_trace_details.png?imwidth=3840)

### 实时指标

响应时间、错误率、吞吐量，全部实时更新。非常适合进行负载测试或了解流量模式。

![实时指标](https://www.milanjovanovic.tech/blogs/mnw_157/metrics.png?imwidth=3840)

## 总结

独立的 Aspire 数据看板非常适合本地开发和调试。启动你的技术栈，发出请求，即可立即查看所有服务中的实时动态。在追踪视图中查找瓶颈，将日志与请求关联起来，并实时观察指标的更新。

请注意：这仅适用于开发环境，因为数据存储在内存中，重启后会消失。根据 Aspire 的路线图，不久的将来可能会很快得到解决。如果用于生产环境，建议使用更合适的解决方案，例如使用 Jaeger 进行追踪、Prometheus 收集指标，或者采用 Application Insights 等商业 APM 工具。

但如果你在开发过程中立刻遇到“我的代码到底在做什么？”这样的问题呢？只需不到 5 分钟，你就能实现专业的可观测性。

只需添加容器，配置 OpenTelemetry，就能像专业人士一样开始调试。

今天就到这里。期待与你再次相见。
