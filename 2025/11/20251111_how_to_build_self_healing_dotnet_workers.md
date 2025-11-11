# [如何构建自愈式 .NET 工作服务](https://juliocasal.com/blog/how-to-build-self-healing-dotnet-workers)

你的后台工作进程可能看起来很健康，但你确定它们不是几小时前就停止工作了吗？

你的监控显示它们正在运行。CPU 使用率看起来正常。但是你的队列每小时都在增长数千条消息，因为每个工作进程都在卡住重试同一个失败的数据库连接。

我曾经给大家介绍过 .NET API 的健康检查，但后台工作进程需要一种不同的方法。

API 会快速且明显地失败。而工作进程则会缓慢且悄无声息地失败——在永远不会成功的操作上消耗资源。

解决方案出奇地简单：将内置的健康检查中间件与一个实时监控工作进程依赖项的组件连接起来。

今天，我将向你展示如何创建一个具有健康感知能力的后台工作进程。

让我们开始吧。

## 我们的工作服务

这是我们开始使用的工作服务。它负责实际的后台工作，如处理数据库记录或轮询队列，这些工作将在 StartProcessingAsync 中进行：

```csharp
public class Worker(ILogger<Worker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Simple worker started");

        await StartProcessingAsync(stoppingToken);

        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    private async Task StartProcessingAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("START doing work");

        // 添加实际的工作
        // 例如：开始处理数据库记录，轮询队列等。

        await Task.CompletedTask;
    }
}
```

目前，这个工作服务完全不知道其任何依赖项的健康状态，包括 PostgreSQL 数据库和 Azure Service Bus 队列，正如我们在 Program.cs 中看到的：

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.AddNpgsqlDbContext<HealthyWorkerContext>("todosdb");
builder.AddAzureServiceBusClient("serviceBus");

builder.Services.AddHostedService<Worker>();

var host = builder.Build();

await host.MigrateDbAsync();

host.Run();
```

如果 PostgreSQL 或 Service Bus 开始出现任何问题（在云环境中很常见），我们的工作服务将开始失败，记录大量错误，并可能向我们的死信队列发送数百条消息。

为防止这种情况，让我们首先跟踪工作器及其依赖项的健康状况。

## 跟踪健康状态

让我们首先添加一个新类，该类将存储我们当前的整体健康状态，并且还可以引发一个简单事件来报告它：

```csharp
public class HealthStatusTracker
{
    private volatile bool _isHealthy = false;

    public bool IsHealthy => _isHealthy;

    public event Action<bool>? HealthStatusChanged;

    public void UpdateHealthStatus(bool isHealthy)
    {
        var previousStatus = _isHealthy;
        _isHealthy = isHealthy;

        if (previousStatus != isHealthy)
        {
            HealthStatusChanged?.Invoke(isHealthy);
        }
    }
}
```

现在，让我们引入一个新的后台服务，它将定期检查当前的健康状态，并通过我们的新跟踪器进行报告：

```csharp
public class HealthMonitorWorker(
    HealthCheckService healthCheckService,
    HealthStatusTracker healthStatusTracker,
    ILogger<HealthMonitorWorker> logger) : BackgroundService
{
    private readonly TimeSpan _healthCheckInterval = TimeSpan.FromSeconds(10);

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Health monitor worker started");

        while (!stoppingToken.IsCancellationRequested)
        {
            var healthReport = await healthCheckService.CheckHealthAsync(stoppingToken);
            var isHealthy = healthReport.Status == HealthStatus.Healthy;

            healthStatusTracker.UpdateHealthStatus(isHealthy);

            logger.LogInformation(
                "Health check completed. Status: {Status}",
                healthReport.Status);

            await Task.Delay(_healthCheckInterval, stoppingToken);
        }

        logger.LogInformation("Health monitor worker stopped");
    }
}
```

这里的一个关键见解是使用内置的 HealthCheckService，它是健康检查中间件的一部分，可以报告所有向其提供健康信息的依赖项的健康状态。

现在，让我们确保将 HealthStatusTracker 和 HealthMonitorWorker 都注册到服务容器中，这样它们就能与我们的主工作器一起启动：

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.AddNpgsqlDbContext<HealthyWorkerContext>("todosdb");
builder.AddAzureServiceBusClient("serviceBus");

builder.Services.AddSingleton<HealthStatusTracker>();
builder.Services.AddHostedService<HealthMonitorWorker>();

builder.Services.AddHostedService<Worker>();

var host = builder.Build();

await host.MigrateDbAsync();

host.Run();
```

最后，我们还应该注册健康检查中间件，否则这一切都无法工作。

通过调用 builder.Services.AddHealthChecks() 可以很容易地实现这一点，或者，如果你正在使用 .NET Aspire，可以通过调用 AddServiceDefaults() 来实现：

```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.AddServiceDefaults();

builder.AddNpgsqlDbContext<HealthyWorkerContext>("todosdb");
builder.AddAzureServiceBusClient("serviceBus");

builder.Services.AddSingleton<HealthStatusTracker>();
builder.Services.AddHostedService<HealthMonitorWorker>();

builder.Services.AddHostedService<Worker>();

var host = builder.Build();

await host.MigrateDbAsync();

host.Run();
```

AddServiceDefaults() 来自 Aspire 的 ServiceDefaults 项目，它不仅会为你处理健康检查，还会完成许多必备的工作。如果你是 .NET Aspire 的新手，可以在这里查看我的初学者教程。

现在，让我们更新我们的工作器，使其能够感知健康状态。

## 使用 HealthStatusTracker

我们现在需要做的就是将 HealthStatusTracker 注入到我们的 Worker 中，并对其健康状态通知做出反应：

```csharp
public class Worker(
    HealthStatusTracker healthStatusTracker,
    ILogger<Worker> logger) : BackgroundService
{
    private bool isProcessing = false;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 订阅健康状态变化
        healthStatusTracker.HealthStatusChanged += OnHealthStatusChanged;

        logger.LogInformation("Simple worker started");

        // 初始检查 - 如果健康则开始处理
        if (healthStatusTracker.IsHealthy)
        {
            await StartProcessingAsync(stoppingToken);
        }

        await Task.Delay(Timeout.Infinite, stoppingToken);
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        // 取消订阅健康状态变化
        healthStatusTracker.HealthStatusChanged -= OnHealthStatusChanged;
        await StopProcessingAsync(cancellationToken);

        logger.LogInformation("Simple worker stopped");

        await base.StopAsync(cancellationToken);
    }

    private async void OnHealthStatusChanged(bool isHealthy)
    {
        if (isHealthy)
        {
            await StartProcessingAsync(CancellationToken.None);
        }
        else
        {
            await StopProcessingAsync(CancellationToken.None);
        }
    }

    private async Task StartProcessingAsync(CancellationToken cancellationToken)
    {
        if (!isProcessing)
        {
            isProcessing = true;
            logger.LogInformation("START doing work - Application is healthy");

            // 在这里添加实际的工作
            // 例如：开始处理数据库记录，轮询队列等。

            await Task.CompletedTask;
        }
    }

    private async Task StopProcessingAsync(CancellationToken cancellationToken)
    {
        if (isProcessing)
        {
            isProcessing = false;
            logger.LogWarning("STOP doing work - Application is unhealthy");

            // 在这里添加清理工作
            // 例如：停止处理数据库记录，停止轮询队列等。

            await Task.CompletedTask;
        }
    }
}
```

正如你所见，每当我们的 HealthStatusTracker 报告应用程序（及其依赖项）的健康状态发生变化时，我们都会处理 HealthStatusChanged 事件并开始或停止处理工作。

我们对不健康情况的反应速度取决于你在 HealthMonitorWorker 中配置的间隔时间，但这应该能显著减少所有这些错误和堆积的消息。

现在，让我们来试试看。

## 对健康状态变化的反应

由于我在我的代码库中添加了 .NET Aspire，我可以轻松地启动我的工作进程及其所有依赖项，并在仪表板中查看所有内容的初始状态：

![仪表板](https://juliocasal.com/assets/images/2025-10-11/4ghDFAZYvbFtvU3CTR72ZN-iAiJXUcLUsVsXPRMKrq4ek.jpeg)

现在，如果我们深入查看 Worker 的日志，应该会看到其健康状态每 10 秒报告一次：

![控制台日志](https://juliocasal.com/assets/images/2025-10-11/4ghDFAZYvbFtvU3CTR72ZN-nBVUjcD22tN4mSifWzoGtQ.jpeg)

好的。现在，让我们通过在仪表板中停止 PostgresSQL 服务器来模拟它完全宕机的情况：

![停止 PostgresSQL](https://juliocasal.com/assets/images/2025-10-11/4ghDFAZYvbFtvU3CTR72ZN-sPeF6jaUqtrao2TvK5sXtr.jpeg)

让我们再次查看那些工作日志，看看它是否注意到了这个问题：

![控制台日志](https://juliocasal.com/assets/images/2025-10-11/4ghDFAZYvbFtvU3CTR72ZN-o3Zgc84eqN98CfQSWiAsep.jpeg)

很好！工作进程停止了工作，尽管它仍在检查健康状态变化以了解何时重新启动。

让我们通过在仪表板中重新启动资源来模拟 PostgreSQL 的恢复：

![重新启动 PostgresSQL](https://juliocasal.com/assets/images/2025-10-11/4ghDFAZYvbFtvU3CTR72ZN-85REeoEAf9EdSMSvizDiLG.jpeg)

让我们再次查看这些日志，看看我们是否已恢复到健康状态：

![控制台日志](https://juliocasal.com/assets/images/2025-10-11/4ghDFAZYvbFtvU3CTR72ZN-rN3kzZ4UzKg4R6ii4uynWq.jpeg)

工作进程确实注意到 PostgreSQL 已恢复，因此恢复了正常运行。

大功告成！

## 总结

后台工作进程很容易被遗忘，直到它们悄无声息地失败。

通过连接健康检查、HealthStatusTracker 和监控工作器，你确保了后台作业在系统不健康时不会盲目地继续运行。

这就是你为能够自我修复的应用程序实现可靠性可视化的方式。

不是通过充满红灯的仪表板，而是通过能够感知何时出现问题并自动做出反应的代码。

今天就到这里。希望这些内容对你有帮助。
