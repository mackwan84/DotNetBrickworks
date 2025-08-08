# [使用 IMeterFactory 测量 .NET 应用程序性能](https://code-maze.com/dotnet-imeterfactory-application-performance/)

性能监控对于确保我们的应用程序高效可靠运行至关重要。.NET 提供了一套工具来帮助实现这一目标，这些工具可通过 IMeterFactory 访问。在本文中，我们将学习如何使用这些工具来检查应用程序的健康状况、测量性能以及收集用于优化的数据。

那么我们开始吧。

## .NET 指标工具是什么？

在.NET 中，我们有各种可用的工具来捕获应用程序的性能数据，例如：

- **Counter\<T\>**：跟踪递增计数，例如总请求数或点击次数
- **Gauge\<T\>**：测量波动的非累积值，例如当前内存消耗
- **UpDownCounter\<T\>**：捕获可以增加和减少的值，例如队列大小
- **Histogram\<T\>**：可视化数据如何在不同值范围内分布

除此之外，还有可观测的仪器，如 **ObservableCounter\<T\>**、**ObservableGauge\<T\>** 和 **ObservableUpDownCounter\<T\>**，它们在被观测时会报告其值。

这些仪器经过精心设计，以满足不同的监控需求，从而实现准确且有意义的性能跟踪。

## 在 ASP.NET Core Web API 中配置 IMeterFactory

让我们创建一个 ASP.NET Core Web API 项目并配置它以使用 **IMeterFactory**。在创建和使用上述提到的度量工具之前，我们需要先完成这一步。

**IMeterFactory** 是 System.Diagnostics.Metrics NuGet 包的一部分，该包在.NET 8+中默认包含。这意味着我们可以直接将 **IMeterFactory** 注入到我们的类中。现在让我们通过创建一个 MetricsService 类来实现：

```csharp
public class MetricsService
{
    public MetricsService(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("Metrics.Service");     
    }
}
```

我们定义一个 MetricsService 类并向其中注入 **IMeterFactory** 来初始化一个 Meter 实例。现在，我们可以使用这个 Meter 实例来定义和捕获指标。

## 定义 IMeterFactory 工具

接下来，让我们看看如何捕获各种指标。

让我们声明一个用于保存用户点击次数的计数器，一个用于报告响应时间的直方图，以及几个用于存储请求和内存消耗的变量：

```csharp
public class MetricsService
{
    private readonly Counter<int> _userClicks;
    private readonly Histogram<double> _responseTime;

    private int _requests;
    private double _memoryConsumption;

    public MetricsService(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("Metrics.Service");

        _userClicks = meter.CreateCounter<int>("metrics.service.user_clicks");

        _responseTime = meter.CreateHistogram<double>("metrics.service.response_time");

        meter.CreateObservableCounter("metrics.service.requests", () => _requests);

        meter.CreateObservableGauge("metrics.service.memory_consumption", 
            _memoryConsumption);
    }
}
```

首先，我们使用 CreateCounter() 方法初始化计数器。接下来，我们使用 CreateHistogram() 方法初始化直方图指标。

之后，我们使用 CreateObservableCounter() 方法设置了一个可观测计数器，它通过回调函数返回 _requests 的值。同样，我们使用 CreateObservableGauge() 方法设置了一个可观测仪表，通过其自己的回调函数返回_memoryConsumption 值。

由于 Counter 和 UpDownCounter 之间的唯一区别是前者只能增加数值，而后者可以增加和减少，因此我们在这里不讨论 UpDownCounter 。

## 捕获指标

现在，让我们在接口和类中添加几个用于记录这些指标值的方法。

首先，让我们创建一个 IMetricsService 接口并添加一些方法约定：

```csharp
public interface IMetricsService
{
    void RecordUserClick();
    void RecordResponseTime(double value);
    void RecordRequest();
    void RecordMemoryConsumption(double value);
}
```

然后，让我们在 MetricsService 类中实现这个新接口：

```csharp
public class MetricsService : IMetricsService
{
    // 私有字段和构造函数省略

    public void RecordUserClick()
    {
        _userClicks.Add(1);
    }

    public void RecordResponseTime(double value)
    {
        _responseTime.Record(value);
    }

    public void RecordRequest()
    {
        Interlocked.Increment(ref _requests);
    }

    public void RecordMemoryConsumption(double value)
    {
        _memoryConsumption = value;
    }
}
```

在这里， RecordUserClick() 方法将 _userClicks 计数器增加一，以跟踪用户点击次数。 RecordResponseTime() 方法使用直方图指标记录所提供应用程序的响应时间。 RecordRequest() 方法在多线程环境中每次调用时安全地将_requests 计数器增加一，而 RecordMemoryConsumption() 方法则用提供的值更新 _memoryConsumption 字段。

之后，让我们创建一个控制器类并向其中注入 IMetricsService 。在控制器中，让我们添加一个 GET 方法来生成一些指标数据，并使用 MetricsService 方法记录这些数据：

```csharp
[Route("api/[controller]")]
[ApiController]
public class MetricsController(IMetricsService metricsService) : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        var random = Random.Shared;

        metricsService.RecordUserClick();

        for (int i = 0; i < 100; i++)
        {
            metricsService.RecordResponseTime(random.NextDouble());
        }

        metricsService.RecordRequest();

        metricsService.RecordMemoryConsumption(GC.GetAllocatedBytesForCurrentThread() / (1024 * 1024));

        return Ok();
    }
    
}
```

我们首先通过调用 RecordUserClick() 方法记录用户点击事件。接下来，我们在循环中为响应时间生成随机值，并通过调用 RecordResponseTime() 方法捕获这些值。之后，我们通过调用 RecordRequest() 方法记录请求事件。最后，我们通过调用 RecordMemoryConsumption() 方法记录当前线程的内存使用量（以兆字节为单位）。

在这里，控制器的 Get() 方法通过调用 MetricsService 中的各种方法并传入随机值来模拟指标数据收集，然后返回一个 HTTP 200 OK 响应。

最后，让我们在依赖注入容器中注册 MetricsService ：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddSingleton<IMetricsService, MetricsService>();

var app = builder.Build();
//...
app.Run();
```

这会将 MetricsService 注册为 IMetricsService 接口的单例服务，在整个应用程序的生命周期内有效。

另外，请确保添加 Swashbuckle.AspNetCore NuGet 包并在 Program 类中配置 Swagger UI：

```csharp
var builder = WebApplication.CreateBuilder(args);
//...
builder.Services.AddSwaggerGen();
builder.Services.AddSingleton<IMetricsService, MetricsService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
//...
app.Run();
```

这将启用 Swagger UI，使可视化和运行 API 端点变得更加容易。

## 可视化指标

让我们运行 API 应用程序，它应该会显示 Swagger UI。要查看指标，我们将使用 dotnet-counters 工具。

首先，我们需要使用 dotnet tool update 命令安装 dotnet-counters 工具：

```bash
dotnet tool update -g dotnet-counters
```

工具安装完成后，我们会收到一条成功消息：

```bash
> dotnet tool update -g dotnet-counters

You can invoke the tool using the following command: dotnet-counters
Tool 'dotnet-counters' (version '9.0.553101') was successfully installed.
```

在应用程序仍在运行时，让我们使用 dotnet-counters 来监控我们应用程序中的所有指标：

```bash
dotnet-counters monitor -n MetricsAPI --counters Metrics.Service
```

在这里，我们指定 dotnet-counters 工具来监控来自 MetricsService 仪表的 MetricsAPI 应用程序中的所有指标。请记住，仪表名称区分大小写。

这将打开指标屏幕，由于我们尚未运行端点来生成这些指标，因此屏幕将是空的：

```bash
Press p to pause, r to resume, q to quit.
    Status: Waiting for initial payload...

Name                                                                                   Current Value
```

现在，让我们从 Swagger UI 调用 \metrics 端点来创建指标并观察输出：

```bash
Press p to pause, r to resume, q to quit.
    Status: Running

Name                                                                                       Current Value
[Metrics.Service]
    metrics.service.memory_consumption                                                             0.003
    metrics.service.requests (Count)                                                               1      
    metrics.service.response_time
    Percentile
        50                                                                                         0.567
        95                                                                                         0.938
        99                                                                                         0.988
    metrics.service.user_clicks (Count)                                                            1    
```

正如预期，我们可以看到包含所有收集指标的输出。当我们多次运行端点时，注意 user_clicks 和 request 的值如何递增 1。另一方面， response_time 显示了大量随机样本的百分位。虽然 dotnet-counters 工具将直方图仪器渲染为三个百分位统计（50th、95th 和 99th），但其他工具可能会以不同方式总结分布或提供更多配置选项。同样， memory_consumption 每次都显示不同的值，因为它代表一个测量仪。

## 通过单位和描述增加清晰度

当我们定义仪表时，可以指定一个可选的单位与描述。这些详细信息不会改变任何计算，但它们可以帮助我们在收集工具的界面中理解数据。目前，dotnet-counters 工具不显示描述文本，但如果提供了单位，它会显示该单位。

让我们修改 MetricsService 构造函数，在创建用于捕获响应时间的直方图时指定单位为秒并添加描述：

```csharp
_responseTime = meter.CreateHistogram<double>(name: "metrics.service.response_time",
    unit: "Seconds",
    description: "This metric measures the time taken for the application to respond to user requests.");
```

该选项在可观测仪器上同样可用。

让我们进一步修改构造函数，在创建内存消耗的可观测仪表指标时指定单位为兆字节并添加描述：

```csharp
meter.CreateObservableGauge(name: "metrics.service.memory_consumption",
    () => _memoryConsumption,
    unit: "Megabytes",
    description: "This metric measures the amount of memory used by the application.");
```

现在，当我们运行应用程序，访问端点并观察指标时，我们可以看到响应时间以秒为单位显示，而内存消耗则以兆字节为单位显示：

```bash
Press p to pause, r to resume, q to quit.
    Status: Running
Name                                                                                        Current Value
[Metrics.Service]
    metrics.service.memory_consumption (Megabytes)                                                 0.003
    metrics.service.requests (Count)                                                               1
    metrics.service.response_time (Seconds)
        Percentile
        50                                                                                         0.532
        95                                                                                         0.952
        99                                                                                         0.976
    metrics.service.user_clicks (Count)                                                            1    
```

## 定义多维指标

测量可以带有标签，将它们链接到键值对，这有助于组织数据进行分析。我们可以在重载的 Add() 和 Record() 方法中使用特定标签来标记 Counter 和 Histogram 测量，这些方法接受一个或多个 KeyValuePair 参数。对于 ObservableCounter 和 ObservableGauge ，我们可以在提供给构造函数的回调中添加带标签的测量。

例如，为了通过添加用户所在地区和点击的功能等详细信息来改进用户点击指标，我们可以在 MetricsService 中创建一个名为 RecordUserClickDetailed() 的方法。该方法允许我们将这些额外详细信息发送到重载的 Counter.Add() 方法：

同样，让我们创建一个多维仪表，用于报告详细的资源消耗情况，如 CPU、内存和线程数。首先，让我们将这些额外字段添加到我们的 MetricService 类中：

```csharp
private double _cpu;
private double _memory;
private double _threadCount;
```

接下来，让我们创建一个返回 IEnumerable<Measurement<int>> 的 GetResourceConsumption() 方法：

```csharp
private IEnumerable<Measurement<double>> GetResourceConsumption()
{
    return
    [
        new Measurement<double>(_cpu, new KeyValuePair<string,object?>
            ("resource_usage", "cpu")),
        new Measuremcent<double>(_memory, new KeyValuePair<string,object?>
            ("resource_usage", "memory")),
        new Measurement<double>(_threadCount, new KeyValuePair<string,object?>
            ("resource_usage", "thread_count")),
    ];
}
```

接下来，在我们的构造函数中，我们需要更新可观察 Gauge 创建中的回调，以调用我们的新 GetResourceConsumption() 方法：

```csharp
public MetricsService(IMeterFactory meterFactory)
{
    // 此处代码省略

    meter.CreateObservableGauge(name: "metrics.service.resource_consumption",
        () => GetResourceConsumption());
}
```

此外，让我们创建一个 RecordResourceUsage() 方法来捕获资源使用情况：

```csharp
public void RecordResourceUsage(double currentCpuUsage, double currentMemoryUsage, double currentThreadCount)
{
    _cpu = currentCpuUsage;
    _memory = currentMemoryUsage;
    _threadCount = currentThreadCount;
}
```

另外，让我们创建一个工具类和方法来计算 CPU 使用率：

```csharp
public static class Utilities
{
    public static double GetCpuUsagePercentage()
    {
        var process = Process.GetCurrentProcess();

        var startTime = DateTime.UtcNow;
        var initialCpuTime = process.TotalProcessorTime;

        Thread.Sleep(1000);

        var endTime = DateTime.UtcNow;
        var finalCpuTime = process.TotalProcessorTime;

        var totalCpuTimeUsed = (finalCpuTime - initialCpuTime).TotalMilliseconds;
        var totalTimeElapsed = (endTime - startTime).TotalMilliseconds;

        var cpuUsage = (totalCpuTimeUsed / (Environment.ProcessorCount * totalTimeElapsed)) * 100;

        return cpuUsage;
    }
}
```

此方法通过捕获一秒内使用的 CPU 时间并计算 CPU 利用率百分比来测量当前进程的 CPU 使用率。

最后，我们需要确保使用我们的两个新记录方法更新 IMetricsService 接口：

```csharp
void RecordUserClickDetailed(string region, string feature);
void RecordResourceUsage(double currentCpuUsage, double currentMemoryUsage, double currentThreadCount);
```

现在，让我们通过将这些方法添加到控制器中 GET 端点的末尾来调用它们，就在最后的`return Ok()`行之前：

```csharp
metricsService.RecordUserClickDetailed("US", "checkout");

metricsService.RecordResourceUsage(
    Utilities.GetCpuUsagePercentage(),
    GC.GetTotalAllocatedBytes() / (1024 * 1024),
    Process.GetCurrentProcess().Threads.Count);
```

在调用 RecordUserClickDetailed() 方法时，我们传递了区域和功能。

调用该方法时，我们传递 CPU 使用率、总内存和线程计数的值。我们使用 GetCpuUsagePercentage() 实用方法获取 CPU 使用率，而 GC.GetTotalAllocatedBytes() 方法可以提供分配给当前进程的总内存， Process.GetCurrentProcess().Threads.Count 则报告该进程中运行的线程数量。

现在，当我们再次运行应用程序，调用端点并观察指标时，可以看到它将这些详细信息显示为多维指标：

```bash
Press p to pause, r to resume, q to quit.
    Status: Running

Name                                                                                       Current Value
[Metrics.Service]
    metrics.service.memory_consumption (Megabytes)                                                 0.016
    metrics.service.requests (Count)                                                               1    
    metrics.service.resource_consumption
        resource_usage
        cpu                                                                                        0.482
        memory                                                                                     6
        thread_count                                                                              50
    metrics.service.response_time (Seconds)
        Percentile
        50                                                                                         0.419
        95                                                                                         0.958
        99                                                                                         0.983
    metrics.service.user_clicks (Count)                                                            1
        user.feature user.region
        checkout     US                                                                            1    
```

这是展示具有多个维度的指标的绝佳方式。

## 使用 MetricCollector 测试 IMeterFactory 指标

我们可以使用 MetricCollector类来测试添加到应用程序中的任何自定义 IMeterFactory 指标。该类简化了从特定仪器记录测量值的过程，并帮助我们验证其准确性。让我们看看如何做到这一点。

首先，我们需要添加 Microsoft.Extensions.DependencyInjection 和 Microsoft.Extensions.Diagnostics.Testing NuGet 包。接下来，我们需要定义一个 CreateServiceProvider() 以在我们的测试方法中使用：

```csharp
private static ServiceProvider CreateServiceProvider()
{
    var serviceCollection = new ServiceCollection();
    serviceCollection.AddMetrics();
    serviceCollection.AddSingleton<MetricsService>();

    return serviceCollection.BuildServiceProvider();
}
```

CreateServiceProvider() 方法设置了一个依赖注入容器。它创建一个新的 ServiceCollection ，添加指标服务和一个 MetricsService 的单例实例，然后构建并返回一个可用于解析这些服务的服务提供程序。

让我们使用 MetricCollector\<int\> 编写一个用户点击指标的测试：

```csharp
public void GivenMetricsConfigured_WhenUserClickRecorded_ThenCounterCaptured()
{
    // 分配
    using var services = CreateServiceProvider();
    var metrics = services.GetRequiredService<MetricsService>();
    var meterFactory = services.GetRequiredService<IMeterFactory>();
    var collector = new MetricCollector<int>(meterFactory, "Metrics.Service", "metrics.service.user_clicks");

    // 操作
    metrics.RecordUserClick();

    // 断言
    var measurements = collector.GetMeasurementSnapshot();
    Assert.Single(measurements);
    Assert.Equal(1, measurements[0].Value);
}
```

此测试验证 MetricsService 是否准确记录用户的点击。它设置所需的服务和度量收集器，然后调用 MetricsService 中的 RecordUserClick() 方法。之后，它检查度量收集器是否恰好捕获了一次用户点击。在这里，度量收集器将收集指定的度量并返回所收集度量的快照。

同样，让我们为使用 ObservableCounter 的请求指标编写一个测试：

```csharp
public void GivenMetricsConfigured_WhenRequestRecorded_ThenObservableCounterCaptured()
{
    // 分配
    using var services = CreateServiceProvider();
    var metrics = services.GetRequiredService<MetricsService>();
    var meterFactory = services.GetRequiredService<IMeterFactory>();
    var collector = new MetricCollector<int>(meterFactory, "Metrics.Service", "metrics.service.requests");

    // 操作
    metrics.RecordRequest();

    // 断言
    collector.RecordObservableInstruments();
    var measurements = collector.GetMeasurementSnapshot();
    Assert.Single(measurements);
    Assert.Equal(1, measurements[0].Value);
}
```

此测试验证 MetricsService 是否准确记录请求。它设置了必要的组件，包括 MetricsService 和 MetricCollector ，以捕获指标。测试随后调用 MetricsService 上的 RecordRequest() 方法，并检查请求的可观察计数器是否增加了一。

在收集可观测指标（ ObservableCounter 、 ObservableGauge 等）时，我们需要在 MetricsCollector 上调用 RecordObservableInstruments() 方法来扫描所有可观测指标。

MetricCollector 简化了为我们应用程序中的各种指标集合编写测试的过程。

## IMeterFactory 最佳实践

让我们探讨选择和实现 IMeterFactory 工具的最佳实践。对于支持依赖注入的库，应避免使用静态变量，而选择依赖注入(DI)。

创建 Meter 时，选择一个唯一的名称很重要。如前所述，遵循 OpenTelemetry 命名指南，使用小写、点分层次结构和下划线来分隔单词，为所有构造命名。确保仪器名称在整个系统中是唯一的，通常需要包含程序集或命名空间名称。

我们应该始终根据需求选择合适的工具；但请记住，在性能密集型场景中，例如当每个线程每秒有超过一百万次调用时，Observable 等效版本可能会表现更好。

如果我们需要了解分布的尾部情况，例如第 90、95 和 99 百分位数，而不仅仅是平均值，那么使用直方图来测量事件时间。对于测量缓存、队列和文件大小，根据与现有代码集成的难易程度，选择 UpDownCounter 或 ObservableUpDownCounter，可以通过 API 调用来进行增减操作，也可以通过回调从维护的变量中获取当前值。

.NET API 允许将任何字符串作为单位，但建议使用 UCUM（单位名称的国际标准）。对于多维指标，API 接受任何对象作为标签值。然而，收集工具通常要求数值类型和字符串，因此提供这些格式至关重要。此外，建议遵循标签名称的命名指南。

## 总结

在本文中，我们讨论了如何在 ASP.NET Core Web API 中设置 IMeterFactory 以有效跟踪各种指标。我们探讨了如何使用 dotnet-counters 工具显示这些指标，并分享了选择和使用不同指标的技巧。最后，我们总结了最佳实践以及在代码中测试指标收集的讨论。

今天就到这里。期待与你再次相见。
