# [.NET 中的现代可观测性：OpenTelemetry & Grafana](https://blog.devops.dev/modern-observability-in-net-opentelemetry-grafana-4c08b9d74240)

> 从基础到高级可观测性：OpenTelemetry & Grafana

可观测性是通过收集和关联三种数据类型——指标、追踪和日志——并在 Grafana 等工具中可视化它们，来了解应用程序内部情况的过程。本文将引导你完成以下内容：

- 使用内置 .NET 工具进行快速诊断
- 使用 OpenTelemetry SDK 进行检测
- 在 Prometheus、Grafana、Jaeger/Tempo 和 Loki/Seq 中导出和可视化数据
- 在代码中编写自定义跨度
- 构建服务地图和告警规则

无需预先具备深厚的专业知识 — 每个代码片段前都有简单的解释，让你了解其存在的原因和作用。

## 什么是可观测性？指标、追踪和日志解析

在编写任何代码之前，让我们先明确三大支柱：

- 指标是随时间采样的数值测量，如 CPU 使用率、请求率或错误计数。它们回答"现在正在发生什么？"
- 跟踪记录单个事务在组件间的路径。跟踪由跨度组成，每个跨度代表一个操作（例如"调用 SQL"、"验证用户"）。跟踪回答"为什么请求花了这么长时间？"
- 日志是带时间戳的文本条目（纯文本或结构化），记录离散事件，如异常、调试消息或业务事件。它们回答"这一点之前和之后发生了什么？"

可观测系统将这三者关联起来，为你提供全面的可见性，并能够快速诊断生产问题。

## 快速诊断：dotnet-counters 和 dotnet-trace

### dotnet-counters：实时指标

dotnet-counters 是一个命令行工具，用于显示 .NET 进程的运行时指标——无需更改代码。

```bash
# 展示可用计数器 (e.g., CPU, GC, ThreadPool)
dotnet-counters list --process-id 1234

# 实时监控 CPU、GC 暂停、HTTP 请求计数
dotnet-counters monitor --process-id 1234 System.Runtime,System.Net.Http
```

使用场景：发现 CPU 高峰、线程池耗尽或请求率的突然激增。

### dotnet-trace：临时跟踪

dotnet-trace 收集 EventPipe 跟踪，这些跟踪捕获方法级别的事件和持续时间。

```bash
# 记录 20 秒的跟踪到 trace.nettrace
dotnet-trace collect --process-id 1234 --duration 20

# 转换为 Web 视图器的格式
dotnet-trace convert --format speedscope trace.nettrace
```

用例：当你怀疑存在性能热点时，无需修改代码即可快速深入方法级别的计时情况。

## OpenTelemetry (.NET) 简介

OpenTelemetry (OTel) 是一个 CNCF 标准，它统一了你收集指标、跟踪和日志的方式。.NET SDK 提供：

- 对 ASP.NET Core、HttpClient、SQL 客户端等的自动检测。
- 用于创建自定义指标和跨度的单一 API。
- 多种导出器（Prometheus、OTLP、Jaeger 等）。

### 安装软件包

在你的 .csproj 中：

```xml
<ItemGroup>
  <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.*" />
  <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.*" />
  <PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.*" />
  <PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.*" />
  <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.*" />
</ItemGroup>
```

## 将指标导出到 Prometheus & Grafana

### 在 Program.cs 中的基本设置

配置 OpenTelemetry 以收集服务器和 HTTP 客户端指标，并在 /metrics 上暴露这些指标，供 Prometheus 抓取。

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics
          .AddAspNetCoreInstrumentation()    // HTTP 服务器指标
          .AddHttpClientInstrumentation()    // 外部 HTTP 调用
          .AddPrometheusExporter();          // 暴露端点 /metrics
    });
var app = builder.Build();
// Prometheus在这里抓取指标
app.MapPrometheusScrapingEndpoint();
app.MapGet("/", () => "Hello, Observability!");
app.Run();
```

### Grafana 数据看板

1. 运行 Prometheus（Docker 或 Kubernetes），指向 <http://your-api:PORT/metrics> 。
2. 在 Grafana 中，添加 Prometheus 作为数据源。
3. 导入或构建显示 .NET Core > CPU 、 .NET Core > GC Heap Size 、 HTTP > Request Rate 、 HTTP > Error Rate 的数据看板。

你现在有关键性能指标的实时图表。

## 使用 Jaeger 和 Grafana Tempo 进行分布式追踪

### 为什么需要分布式追踪？

追踪让你能够看到请求在各个服务间的生命周期：

> User → API Gateway → Orders Service → Inventory Service → Database

每个部分是一个跨度，它们共同构成一个追踪。

### 将跨度发送到 Jaeger

添加 Jaeger 导出器，使跨度自动流向 Jaeger 代理进行可视化。

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
          .AddAspNetCoreInstrumentation()
          .AddHttpClientInstrumentation()
          .AddJaegerExporter(o =>
          {
              o.AgentHost = "localhost";
              o.AgentPort = 6831;
          });
    });
```

### 将跨度发送到 Grafana Tempo (OTLP)

使用 OTLP 导出器将跨度直接发送到 Grafana Tempo 的 gRPC 端点。

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
          .AddAspNetCoreInstrumentation()
          .AddHttpClientInstrumentation()
          .AddOtlpExporter(o =>
          {
              o.Endpoint = new Uri("http://tempo:4317");
          });
    });
```

在 Grafana 中，将 Jaeger 或 Tempo 配置为跟踪数据源。现在你可以通过跟踪 ID、服务名称或操作来搜索跟踪。

## 使用 Loki 和 Seq 实现集中式日志

### 什么是日志聚合器？

它们从许多服务中收集日志，让你可以集中查询这些日志，通常还能与追踪和指标进行关联。

### 使用 Loki 和 Seq 进行集中式日志记录

拥有指标和追踪固然很好——但日志能为你提供详细信息。像 Loki 和 Seq 这样的聚合器提供了集中式查询和关联功能。

### 通过 OTLP 将日志推送到 Loki

使用带有 OpenTelemetry 接收器的 Serilog 将结构化日志转发到 Loki。

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.OpenTelemetry(options =>
    {
        options.Protocol = OpenTelemetry.Logs.Protocol.OtlpGrpc;
        options.Endpoint = new Uri("http://loki:4317");
    })
    .CreateLogger();
```

### 将日志推送到 Seq

或者，你可以直接将 Microsoft.Extensions.Logging 与 Seq 集成。

```csharp
builder.Logging.AddSeq("http://seq:5341");
```

通过在日志中包含跟踪和跨度 ID，你能够实现从 Grafana 跟踪无缝切换到 Loki 或 Seq 中的详细日志。

## 代码检测：使用 ActivitySource 创建自定义跨度

自动检测涵盖了框架和库，但你通常需要测量关键业务操作。 ActivitySource 允许你在代码中创建有意义的跨度。

## 围绕业务逻辑创建跨度

此代码片段将订单验证和出站 HTTP 调用包装在带有标记和事件的命名跨度中。

```csharp
private static readonly ActivitySource MySource =
    new("MyApp.Service", "1.0.0");

public async Task ProcessOrderAsync(Order order)
{
    using var span = MySource.StartActivity("ValidateOrder", ActivityKind.Internal);
    span?.SetTag("order.id", order.Id);
    Validate(order);  // 业务逻辑
    span?.AddEvent(new ActivityEvent("OrderValidated"));
    // 外部调用会有自己的子跨度
    await httpClient.PostAsJsonAsync("/ship", order);
}
```

跨度放置技巧：

- 包装高延迟操作（数据库调用、HTTP 调用）。
- 包含关键业务步骤（验证、支付处理）。
- 避免为微小操作创建过于细粒度的跨度。

## 高级：服务地图与告警

当你在 Grafana 中拥有指标、跟踪和日志时，你可以构建：

### 服务地图

根据跨度的 service.name 和 peer.service 属性自动推导拓扑图。在 Grafana 的 Explore → Traces 中，查看基于跨度的 service.name 和 peer.service 属性的自动拓扑。立即看到哪些服务与哪些服务通信。

### 告警

- 针对指标的 Prometheus 告警（例如，1 分钟内 http_requests_total{status="500"} > 5 ）。
- 针对追踪的 Grafana 告警（例如，p99 延迟 > 1 秒）。
- Loki/Seq 对日志模式的警报（例如，10 分钟内出现 5 次异常）。

与 Slack、PagerDuty 或电子邮件集成，以便在出现问题时立即通知你的团队。

## 后续步骤

- 搭建一个新的 .NET Web API 项目。
- 添加 OpenTelemetry 包并复制代码片段。
- 部署本地堆栈（Prometheus、Grafana、Jaeger/Tempo、Loki）。
- 探索数据看板并调整检测。

有了这些构建块，你的 .NET 服务将不仅具有可观测性，而且具有弹性。当你能够在几分钟而不是几小时内 pinpoint 并修复生产问题时，你的团队（和你的用户）会感谢你。祝你检测顺利！

## 总结

现代 .NET 可观测性结合了 dotnet-counters 和 dotnet-trace 的即时洞察力，OpenTelemetry SDK 的结构化指标和跟踪的丰富性，以及 Grafana、Prometheus、Jaeger/Tempo 和 Loki/Seq 的端到端可视化灵活性。通过遵循本指南——从基本的计数器转储到高级自定义 span 和警报——你将把你的 .NET 应用转变为完全可观测、可靠的服务，你的团队和用户都会信任这些服务。

准备好为你的下一个 .NET 项目添加检测了吗？搭建一个示例 API，放入上面的代码片段，今天就开始在 Grafana 中探索你的指标、跟踪和日志吧！
