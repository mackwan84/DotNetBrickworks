# [.NET 调度的新选择 TickerQ](https://dev.to/markjackmilian/from-zero-to-scheduled-tickerq-for-net-48lg)

> 作为一名尝试过市面上几乎所有作业调度器的 .NET 开发者，我想分享一下为什么 TickerQ 迅速成为了我的首选工具。它快速、简洁，并且是为开发者而设计的。下面快速介绍一下它提供的功能以及如何开始使用。

## 介绍 TickerQ – 一款现代化的 .NET 调度器

TickerQ 是一个高性能、无反射的 .NET 后台任务调度器。它利用源生成器，提供 cron 和基于时间的执行功能，支持 EF Core 持久化，并包含实时仪表板 UI——所有这些都只需最小的开销。

主要特点包括：

- 源代码生成器发现（无反射）
- TimeTickers（一次性）和 CronTickers（周期性）
- 用于持久化作业、状态和历史记录的 EF Core 集成
- 使用 SignalR 和 Tailwind（基于 Vue.js）构建的仪表板 UI
- 重试策略、限流、冷却期
- 分布式协调、优先级、DI 支持

## 安装与配置

使用以下 NuGet 包。虽然只需要核心 TickerQ 包，但在本示例中我们还将设置用于 EF Core 持久化和实时仪表板的可选包：

```bash
dotnet add package TickerQ
dotnet add package TickerQ.EntityFrameworkCore
dotnet add package TickerQ.Dashboard
```

在 Program.cs

```csharp
builder.Services.AddTickerQ(options =>
{
    options.SetMaxConcurrency(4);
    options.SetExceptionHandler<MyExceptionHandler>();
    options.AddOperationalStore<MyDbContext>(efOpt =>
    {
        efOpt.UseModelCustomizerForMigrations();
        efOpt.CancelMissedTickersOnApplicationRestart();
    });
    options.AddDashboard(basePath: "/tickerq-dashboard");
    options.AddDashboardBasicAuth();
});

app.UseTickerQ();
```

## 定义和调度作业

TickerQ 使用 [TickerFunction] 属性和编译时源生成。你可以定义多个具有不同触发器的计划函数：

```csharp
[TickerFunction("CleanupLogs")]
public static async Task CleanupLogs(TickerRequest req, CancellationToken ct)
{
    // 执行日志清理逻辑
}

[TickerFunction("CleanupTempFiles","0 */6 * * *")]
public static Task CleanupTempFiles(TickerRequest req, CancellationToken ct)
{
    Console.WriteLine("Cleaning temporary files...");
    return Task.CompletedTask;
}
```

### TimeTicker 示例（一次性或延迟）

```csharp
await timeTickerManager.AddAsync(new TimeTicker
{
  Function = "SendWelcomeEmail",
  Request = new { UserId = 123 },
  RunAt = DateTime.UtcNow.AddMinutes(5),
  Retries = 3,
  RetryIntervals = new[] { 60, 120, 300 }
});
```

### CronTicker 示例（周期性）

```csharp
await cronTickerManager.AddAsync(new CronTicker
{
  Function = "CleanupLogs",
  CronExpression = "0 */6 * * *",
  Retries = 2,
  RetryIntervals = new[] { 60, 300 }
});
```

## 使用 EF Core 实现持久化

要启用持久化，你需要使用 EF Core 迁移创建必要的表。

TickerQ 的 EF Core 提供程序将计时器、执行历史和作业状态持久化到你的 DbContext .

有两种集成方法：

1. 使用 UseModelCustomizerForMigrations() ：最适合那些希望避免在 DbContext 类中添加额外配置的项目。这种方法仅在 EF Core 设计时操作（例如 dotnet ef migrations add ）期间自动连接架构。在运行时，它不会干扰你的模型。

2. 手动配置：如果你想要完全控制或需要在运行时模型中查看架构，则需要进行手动配置。这需要在你的 OnModelCreating 方法中直接应用必要的配置。

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);
    builder.ApplyConfiguration(new TimeTickerConfigurations());
    builder.ApplyConfiguration(new CronTickerConfigurations());
    builder.ApplyConfiguration(new CronTickerOccurrenceConfigurations());
}
```

应用迁移：

```bash
dotnet ef migrations add AddTickerQTables
dotnet ef database update
```

## 仪表板 UI 和实时监控

![仪表盘](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2F0qsus1ywabj1pbse6xum.jpeg)

TickerQ 仪表盘是一个基于 Vue.js 的现代 Web 界面，由 SignalR 和 Tailwind CSS 驱动。它提供实时更新以及对作业调度和监控的全面控制。

### 设置

```csharp
builder.Services.AddTickerQ(opt =>
{
    opt.AddOperationalStore<MyDbContext>();
    opt.SetInstanceIdentifier("MyAppTickerQ");
    opt.AddDashboard(basePath: "/tickerq-dashboard");
    opt.AddDashboardBasicAuth();
});

app.UseTickerQDashboard();
```

在 appsettings.json 中添加凭据：

```json
{
  "TickerQBasicAuth": {
    "Username": "admin",
    "Password": "admin"
  }
}
```

### 功能

- 系统概述：吞吐量、活跃节点、并发性。
- 作业状态分解：已完成、到期完成、失败、待处理。
- CronTickers 视图：时间线、手动触发、复制、删除。
- TimeTickers 视图：执行历史，实时重试/取消/编辑/删除选项。
- 添加/编辑作业：直观的用户界面用于定义函数、负载和重试策略。

## 通过 EF Core 实现分布式锁

要在多个节点上安全运行 TickerQ 而不重叠执行作业，TickerQ 使用基于 EF Core 的分布式锁机制。

### 工作原理

- 一个节点接取任务并在数据库中将自身标记为 OwnerNode 。
- 行级锁定（例如， SELECT ... FOR UPDATE ）确保互斥。
- 失败的任务会自动释放锁，使其他任务能够重试。
- 冷却和过期机制为防止僵尸任务提供了安全保障。

```csharp
builder.Services.AddTickerQ(options =>
{
  options.AddOperationalStore<MyDbContext>(efOpt =>
  {
    efOpt.CancelMissedTickersOnApplicationRestart();
  });
});
```

### 你将获得什么

| 功能 | 描述 |
| --- | --- |
| 单一所有权 | 一次只能有一个节点处理作业 |
| 故障转移安全性 | 锁在重启或超时时被释放 |
| 协调的简单性 | 无需外部锁存储 |
| EF Core 友好 | 利用你现有的数据库和事务逻辑 |

> 如果你进行水平扩展并希望实现安全的数据库驱动协调，请使用此方案。对于极高吞吐量或跨集群工作负载，请考虑使用 Redis 或 Zookeeper。

### TickerQ 的对比分析

与像 Hangfire 或 Quartz.NET 这样的经典调度器相比：

| 功能 | TickerQ | Hangfire | Quartz.NET |
| --- | --- | --- | --- |
| 无反射 | ✅ 源代码生成 | ❌ 反射 | ❌ 反射 |
| 一次性定时任务 与 周期性循环任务 | ✅ 两者 | ✅ 支持周期性循环任务，有限制的延迟任务 | ✅ 两者 |
| EF Core 持久化 | ✅ 内置 | ✅ | ✅ |
| 仪表板界面 |  ✅ 自带实时界面 | ⚠️ 基础界面 |  ⚠️ 第三方界面 |
| 重试与冷却 |  ✅ 高级 |  ⚠️ 基础 |  ⚠️ 需要手动配置 |
| 分布式锁 |  ✅ 通过 EF Core |  ⚠️ 部分 |  ✅ |
| DI 支持 |  ✅ 原生 |  ⚠️ 基于类型的限制 |  ⚠️ 手动配置 |
| 性能 | ✅ 低开销 | ⚠️ 存储限制 |  ⚠️ 线程阻塞 |

## 何时选择 TickerQ

如果你符合以下情况，请选择 TickerQ：

- 优先选择编译时安全性而非运行时反射
- 需要一个轻量级、快速的调度器
- 想要支持重试/冷却的定时任务和时间作业
- 使用 EF Core 或需要分布式协调
- 重视自带实时仪表盘界面

## 需要考虑的限制因素

- 尚不支持 Redis 或分布式缓存持久化（目前）
- 无特定节点路由或基于标签的作业定向
- 节流和任务链功能正在开发中

## 总结

TickerQ 提供了一个现代化、高效且对开发者友好的后台调度器，其特点包括：

- 通过源代码生成实现无反射架构
- 支持一次性定时与周期性循环的任务调度
- 强大的 EF Core 持久化
- 实时仪表板，支持实时更新

它特别适合那些注重性能、可见性和可维护性的应用程序。如需深入了解，请访问官方文档。

今天就到这里。期待与你再次相见。
