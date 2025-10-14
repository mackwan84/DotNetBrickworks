# [TickerQ：超越 Quartz 和 Hangfire 的现代.NET 作业调度器](https://antondevtips.com/blog/tickerq-the-modern-dotnet-job-scheduler-that-beats-quartz-and-hangfire)

## 介绍

调度后台作业是许多 .NET 应用程序的基本组成部分——无论是发送邮件、处理报告还是同步数据。

多年来，开发者们一直依赖像 Quartz.NET 和 Hangfire 这样的工具。

虽然它们都是很棒的工具，但它们有几个缺点。

Quartz.NET 的缺点：

- 基于反射的作业注册
- 复杂的 API
- 繁琐的配置
- 向作业传递参数的方式笨拙
- 没有内置的 EF Core 集成（只能通过第三方包实现）
- 没有用于作业监控的内置数据看板（仅通过第三方软件包）
- 没有内置重试功能

Hangfire 的缺点：

- 基于反射的作业注册
- 有限的依赖注入支持
- 有限的异步支持
- 一些内置存储提供商需要商业许可证

TickerQ 是对 .NET 中作业调度的新诠释——提供现代、简洁的 API、内置持久化、EF Core 集成以及用于作业监控的精美数据看板。

它消除了我们习惯的大量样板代码和繁琐仪式，转而提供了对开发者友好、代码优先的体验。

今天，我们将探讨：

- 如何在.NET 项目中设置 TickerQ
- 将 TickerQ 与 EF Core 集成
- 创建和调度作业（周期性和一次性）
- 动态注册作业
- 使用 TickerQ 数据看板监控作业执行

闲话少说，让我们开始吧！

## 开始使用 TickerQ

TickerQ 是一个针对 .NET Standard 2.1 的 NuGet 包，与任何 .NET Core 3.1+ 应用程序兼容。它与 ASP.NET Core 开箱即用，不需要任何额外的配置文件——一切都是代码优先的，并且支持依赖注入。

按照以下步骤开始使用 TickerQ：

### 步骤 1：安装 NuGet 包

```bash
dotnet add package TickerQ
```

这会添加核心的 TickerQ 库。你可以选择安装其他包，如 TickerQ.EntityFrameworkCore 和 TickerQ.Dashboard （我们将在下一节中介绍）。

### 步骤 2：在 Program.cs 中注册 TickerQ

TickerQ 直接插入到你的应用程序启动管道中。在你的 Program.cs 中添加以下行：

```csharp
builder.Services.AddTickerQ();

var app = builder.Build();
app.UseTickerQ();

app.Run();
```

AddTickerQ 方法注册了 TickerQ 所需的所有服务。 UseTickerQ 方法添加了处理传入请求的中间件。

就是这样。无需管理后台服务。无需额外的托管服务。TickerQ 在底层为你处理了这一切。

### 步骤 3：定义一个作业

TickerQ 作业只是普通的类方法，类不需要继承任何接口或基类。只需在任何类的任何方法上添加 TickerFunction 属性即可。

以下是一个创建报告的作业示例：

```csharp
public class CreateReportJob
{
    private readonly ReportDbContext _dbContext;

    public CreateReportJob(ReportDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    [TickerFunction(functionName: "Send Notifications", cronExpression: "0 * * * *")]
    public async Task CreateReport(TickerFunctionContext tickerContext,
        CancellationToken cancellationToken)
    {
        var report = new Report
        {
            Title = $"Scheduled Report - {DateTime.UtcNow:yyyy-MM-dd HH:mm}",
            Content = $"This is an automatically generated report created at {DateTime.UtcNow:yyyy-MM-dd HH:mm:ss}",
            CreatedAt = DateTime.UtcNow
        };

        _dbContext.Reports.Add(report);
        await _dbContext.SaveChangesAsync(cancellationToken);
    }
}
```

在这个例子中，TickerQ 每小时会调用 CreateReport 方法。

cronExpression 参数是可选的，可用于基于 cron 表达式调度周期性作业。如果省略它，你可以在 DI 中注册 TickerQ 时配置调度，或者动态指定它。

依赖开箱即用地支持依赖注入，你可以在单个类中指定任意数量的作业方法。

在这里， TickerFunctionContext 用于访问有关作业执行的附加信息。

你也可以使用 context 来取消或删除作业：

```csharp
tickerContext.CancelTicker();

await tickerContext.DeleteAsync();
```

你也可以使用 %Section:Key% 语法来引用来自 appsettings.json 的 cron 表达式：

```csharp
[TickerFunction(FunctionName: "ExampleMethod", CronExpression: "%CronTicker:EveryMinute%")]
public void ExampleMethod() { }
```

```json
{
  "CronTicker": {
    "EveryMinute": "* * * * *"
  }
}
```

### 步骤 4：处理异常（可选）

要处理异常，你可以实现 ITickerExceptionHandler 接口并在依赖注入中注册它：

```csharp
public class TickerExceptionHandler : ITickerExceptionHandler
{
    public async Task HandleExceptionAsync(Exception exception,
        Guid tickerId, TickerType tickerType)
    {
        // 你的逻辑...
    }

    public async Task HandleCanceledExceptionAsync(Exception exception,
        Guid tickerId, TickerType tickerType)
    {
        // 你的逻辑...
    }
}
```

然后在 Program.cs 中注册它：

```csharp
services.AddTicker(opt =>
{
  opt.SetExceptionHandler<TickerExceptionHandler>(); 
});
```

现在让我们来探索如何将 TickerQ 与 EF Core 集成。

## 将 TickerQ 与 EF Core 集成

为了使 TickerQ 投入生产，特别是在你的应用程序可能会重启或跨实例扩展的情况下，你将需要持久化功能。

EF Core 为 TickerQ 提供了一个关系型存储，用于作业状态、历史记录、锁定和恢复。

首先，为 TickerQ 安装 EF Core 扩展：

```bash
dotnet add package TickerQ.EntityFrameworkCore
```

在服务注册中，调用 AddTickerQ(...) 时启用 EF Core 功能。例如：

```csharp
services.AddTickerQ(options =>
{
    // 设置一个备用超时来检查错过的作业并执行它们。
    options.UpdateMissedJobCheckDelay(TimeSpan.FromSeconds(10));

    options.SetInstanceIdentifier("TickerQ");

    // 配置 TickerQ 元数据、锁和状态的 EF Core 支持的操作存储。
    options.AddOperationalStore<ReportDbContext>(efCoreOptions =>
    {
        // 应用自定义模型配置仅在 EF Core 迁移期间（设计时）进行。 运行时模型保持不变。
        // efCoreOptions.UseModelCustomizerForMigrations();

        // 当程序启动时，取消处于 Expired 或 InProgress（终止）状态的作业，
        // 以防止在崩溃或突然关闭后重复执行。
        efCoreOptions.CancelMissedTickersOnAppStart();

        // 定义的基于 cron 的函数默认会自动填充到数据库中。
        // 例如: [TickerFunction(..., "*/5 * * * *")]
        // 使用此选项可以忽略它们并保持种子仅在运行时可用。
        efCoreOptions.IgnoreSeedMemoryCronTickers();
    });
});
```

TickerQ 使用 EF Core 来定义其自己的内部表/实体。

你有两个选择：

### 1. 使用内置的模型自定义功能进行迁移

如果你调用 UseModelCustomizerForMigrations() ，TickerQ 的实体配置将在迁移过程中自动应用。这使代码保持整洁。

### 2. 在 DbContext 中手动配置

如果你不使用模型自定义器，则需要在 OnModelCreating 中显式应用实体配置，例如：

```csharp
public class ReportDbContext : DbContext
{
    public ReportDbContext(DbContextOptions<ReportDbContext> options)
        : base(options) { }

    public DbSet<Report> Reports { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.HasDefaultSchema(DbConsts.SchemaName);

        // 添加 TickerQ 的实体配置
        modelBuilder.ApplyConfiguration(new TimeTickerConfigurations(schema: DbConsts.SchemaName));
        modelBuilder.ApplyConfiguration(new CronTickerConfigurations(schema: DbConsts.SchemaName));
        modelBuilder.ApplyConfiguration(new CronTickerOccurrenceConfigurations(schema: DbConsts.SchemaName));

        // ...
    }
}
```

设置完成后，像往常一样使用 EF Core CLI 命令创建你的 EF 迁移：

```bash
dotnet ef migrations add InitialMigration -c ReportDbContext
```

以下是使用 TickerQ 表时的数据库架构：

![TickerQ 数据库表](https://antondevtips.com/media/code_screenshots/aspnetcore/tickerq/img_1.png)

以下是你可以初始化初始定时器（基于时间和基于 cron）的方法：

```csharp
efCoreOptions.UseTickerSeeder(
    async timeTicker =>
    {
        await timeTicker.AddAsync(new TimeTicker
        {
            Id = Guid.NewGuid(),
            Function = "Create Report",
            ExecutionTime = DateTime.UtcNow.AddSeconds(5)
        });
    },
    async cronTicker =>
    {
        await cronTicker.AddAsync(new CronTicker
        {
            Id = Guid.NewGuid(),
            Expression = "0 0 * * *", // 每天 00:00 UTC
            Function = "Create Report"
        });
    }
);
```

你可以使用 TimeTicker 来安排一个在特定时间执行的任务。

## 动态注册作业

TickerQ 支持一次性/延迟作业（称为 TimeTickers）以及周期性或基于 cron 的作业（称为 CronTickers）。

你可以通过 API 端点或在应用程序启动时动态调度作业，支持重试策略、自定义间隔等功能。

假设你有一个 API 端点，用户可以在其中请求在未来的日期/时间发送通知。

你可以使用 ITimeTickerManager\<TimeTicker\> 来调度这样的作业：

```csharp
public record NotificationJobContext(string Title, string Content);

app.MapPost("/api/schedule‑notification", async (NotificationRequest request, 
    ITimeTickerManager<TimeTicker> timeTickerManager) =>
{
    if (request.ScheduledTime <= DateTime.Now)
    {
        return Results.BadRequest("Scheduled time must be in the future");
    }

    // 创建类型化的作业数据
    var jobData = new NotificationJobContext(request.Title, request.Content);

    // 调度作业
    var result = await timeTickerManager.AddAsync(new TimeTicker
    {
        Function = "Send Notifications",
        ExecutionTime = request.ScheduledTime.ToUniversalTime(),
        Request = TickerHelper.CreateTickerRequest(jobData),
        Retries = 3,
        RetryIntervals = new[] { 30, 60, 120 } // 30秒，60秒，然后2分钟后重试
    });

    return Results.Ok(new
    {
        JobId = result.Result.Id,
        Message = $"Notification '{request.Title}' scheduled for {request.ScheduledTime}"
    });
});
```

我在这里将 NotificationJobContext 传递给作业，然后可以在作业方法中提取它：

```csharp
public class NotificationJob
{
    private readonly ILogger<NotificationJob> _logger;
    public const string TitleKey = "Title";
    public const string ContentKey = "Content";

    public NotificationJob(ILogger<NotificationJob> logger)
    {
        _logger = logger;
    }

    [TickerFunction(functionName: "Send Notifications", cronExpression: "0 0 * * *" )]
    public Task Execute(TickerFunctionContext<NotificationJobContext> tickerContext,
        CancellationToken cancellationToken)
    {
        var title = tickerContext.Request.Title;
        var content = tickerContext.Request.Content;

        // ...

        return Task.CompletedTask;
    }
}
```

或者，你可以使用 ICronTickerManager\<CronTicker\> 来调度一个周期性作业：

```csharp
var result = await cronTickerManager.AddAsync(new CronTicker
{
    Function = "Send Notifications",
    Request = TickerHelper.CreateTickerRequest(jobData),
    Expression = "* * * * *",
    Retries = 3,
    RetryIntervals = new[] { 30, 60, 120 } // 30秒，60秒，然后2分钟后重试
});
```

以下是更新和删除 CronTicker 的方法：

```csharp
// 通过 ID 更新 CronTicker
// 你在调度作业时会获得 ID (result.Result.Id)
await cronTickerManager.UpdateAsync(tickerId, ticker =>
{
    ticker.Description = "Updated cron description";
    ticker.Expression = "*/10 * * * *"; // every 10 minutes
});

// 通过 ID 删除 CronTicker
await cronTickerManager.DeleteAsync(tickerId);
```

## 使用 TickerQ 数据看板

TickerQ 拥有一个内置的官方数据看板 UI，让你能够实时查看和控制所有计划任务（包括基于时间和基于 cron 的任务）。

它对于调试、维护和管理生产工作负载非常有用。

将 TickerQ.Dashboard NuGet 包添加到你的项目中。

```bash
dotnet add package TickerQ.Dashboard
```

在依赖注入中配置 TickerQ 时注册数据看板支持：

```csharp
builder.Services.AddTickerQ(options =>
{
    // ...
    
    options.AddOperationalStore<ReportDbContext>(efCoreOptions =>
    {
        // ...
    });
    
    options.AddDashboard(x =>
    {
        x.BasePath = "/tickerq-dashboard";
        x.EnableBasicAuth = true;
    });
});
```

你也可以为数据看板 UI 配置身份验证；更多详情请参阅文档。

TickerQ 为数据看板 UI 提供开箱即用的基本身份验证支持。你可以在 appsettings.json 中配置凭据：

```json
{
  "TickerQBasicAuth": {
  "Username": "admin",
  "Password": "admin"
}
}
```

要访问数据看板，请在浏览器中导航到 /tickerq-dashboard

输入 admin/admin 登录：

![TickerQ 数据看板登录](https://antondevtips.com/media/code_screenshots/aspnetcore/tickerq/img_2.png)

![TickerQ 数据看板主界面](https://antondevtips.com/media/code_screenshots/aspnetcore/tickerq/img_3.png)

## 总结

TickerQ 是一个现代、轻量级且可用于生产环境的 .NET 应用程序调度库——为 Hangfire 和 Quartz.NET 提供了一个令人耳目一新的替代方案。

以下是 TickerQ 与 Quartz 与 Hangfire 的完整对比表：

![TickerQ 与 Quartz 与 Hangfire 的对比](https://antondevtips.com/media/code_screenshots/aspnetcore/tickerq/img_4.png)

今天就到这里。期待与你再次相见。
