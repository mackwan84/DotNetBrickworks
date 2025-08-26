# [.NET 中的 EF Core 简单日志记录](https://www.nikolatech.net/blogs/ef-core-simple-logging-in-dotnet)

日志记录是每个应用程序的重要组成部分，它帮助我们了解幕后发生的情况，追踪错误，监控性能并调试问题。

在数据访问方面，日志记录可能特别有价值。

你可以看到正在执行的查询、它们需要多长时间以及它们是否按预期运行。

如果你正在使用 Entity Framework Core，你可以通过一个名为简单日志记录的强大功能轻松记录和检查所有内容。

## 简单日志记录

简单日志是一种轻量级方式，可以查看你的 LINQ 查询是如何转换为原始 SQL 的。

它不需要任何高级配置或第三方包，只需几行设置即可。

简单日志的用例：

- 检查由 EF Core 生成的 SQL 查询
- 在本地监控查询性能
- 调试错误或低效查询的问题

它是开发过程中用于学习、调试和优化的绝佳工具。

注意：EF Core 还与 Microsoft.Extensions.Logging 集成，这需要更多配置，但通常更适合在生产应用程序中进行日志记录。

## 简单日志配置

在 EF Core 中启用简单日志记录非常直接。你只需要在设置 DbContext 时配置日志记录即可。

EF Core 日志可通过在配置 DbContext 实例时使用 LogTo 方法从任何类型的应用程序中访问。根据你的偏好和需求，你可以记录到：

- 控制台
- 调试窗口
- 文件

LogTo 方法需要一个 Action 委托，EF Core 会为每个生成的日志消息调用此委托。该委托负责处理消息，无论是将其写入控制台还是文件。

## 记录到控制台

直接记录到控制台是我个人最喜欢的方式之一。只需将 Console.WriteLine 传递给 LogTo 方法，EF Core 就会将每条日志消息写入控制台。

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder.LogTo(Console.WriteLine);
```

## 记录到调试窗口

就像控制台日志记录一样，你可以使用 Debug.WriteLine 将 EF Core 日志消息定向到你 IDE 中的调试窗口。

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder.LogTo(message => Debug.WriteLine(message));
```

在这种情况下必须使用 Lambda 语法，因为 Debug 类在发布版本构建中会被编译排除。

## 记录到文件

记录到文件需要更多设置，因为你需要创建一个 StreamWriter 来将日志写入文件。

一旦流设置完成，你就可以将其 WriteLine 方法传递给 LogTo，就像前面的示例中那样：

```csharp
private readonly StreamWriter _logStream = new StreamWriter("mylog.txt", append: true);

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder.LogTo(_logStream.WriteLine);

public override void Dispose()
{
    base.Dispose();
    _logStream.Dispose();
}

public override async ValueTask DisposeAsync()
{
    await base.DisposeAsync();
    await _logStream.DisposeAsync();
}
```

注意：在释放 DbContext 时，正确释放 StreamWriter 非常重要。

## 敏感数据

默认情况下，EF Core 从消息和日志中排除数据值，如参数和键。

这是一个出于安全考虑的选择，因为此类信息可能是敏感的，不应被暴露。

然而，在调试过程中，访问这些值可能非常有用。要在日志中包含这些信息，你可以使用 EnableSensitiveDataLogging() 显式启用它：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder
        .LogTo(Console.WriteLine)
        .EnableSensitiveDataLogging();
```

注意：仅在调试环境中启用敏感数据日志记录。

## 详细的查询异常

EF Core 默认不会将从数据库读取的每个值包装在 try-catch 块中。

为了改善此类情况下的错误信息，你可以启用 EnableDetailedErrors()。这会指示 EF Core 将各个读取操作包装在 try-catch 块中，并在异常发生时提供更多上下文信息：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder
        .LogTo(Console.WriteLine)
        .EnableDetailedErrors();
```

## 筛选

每个 EF Core 日志消息都与 LogLevel 枚举中的一个严重性级别相关联。

默认情况下，EF Core 会记录 Debug 级别及以上的所有消息，但是，你可以通过指定更高的最低日志级别来减少噪音。

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);
```

若需更精细的控制，LogTo 还支持自定义筛选函数。

这允许你根据 LogLevel 和 EventId 精确指定应该记录哪些消息：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder) =>
    optionsBuilder
        .LogTo(
            Console.WriteLine,
            (eventId, logLevel) => logLevel >= LogLevel.Information
                                   || eventId == RelationalEventId.ConnectionOpened
                                   || eventId == RelationalEventId.ConnectionClosed);
```

## 总结

EF Core 的简单日志记录是一个功能强大且轻量级的特性，它能让你深入了解应用程序如何与数据库交互。

无论你是检查生成的 SQL、调试查询问题，还是优化性能，它都能以最少的配置提供宝贵的透明度。

通过 EnableSensitiveDataLogging、EnableDetailedErrors 和灵活的日志过滤等功能，你可以微调日志记录体验以满足开发需求。

今天就到这里。期待与你再次相见。
